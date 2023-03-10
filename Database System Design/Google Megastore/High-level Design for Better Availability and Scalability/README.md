# High-level Design for Better Availability and Scalability
Let’s see Megastore’s high-level design before discussing how Megastore improves availability and scalability.

## High-level design
The components involved in the high-level design of our system are as follows:

- Application server: It is used to deploy Megastore along with the Megastore library. There is a defined local replica on each application server. By sending transactions directly to the local Bigtable, the Megastore library makes Paxos operations on that replica persistent.
- Megastore library: Applications connect to the Megastore library, which implements Paxos and other algorithms, like picking a replica for reading, catching up with a replica that is falling behind, and so on.
- Replication server: Periodically, replication servers look for unfinished writes and provide Paxos no-op values to finish them.
- Coordinator: A coordinator server keeps track of a set of entity groups for which its replica has seen every Paxos write.
- Bigtable: For scalable fault-tolerant storage within a particular data center, we utilize Google’s Bigtable, which allows us to handle arbitrary read and write throughput by distributing operations across several rows.
The following illustration shows the high-level design of Megastore components within a data center.

[The high-level design of Megastore components within a data center](./arch.jpg)

In this design problem, our primary concern is to provide better transactions with stronger consistency, availability, and scalability. Megastore uses the following two approaches to provide high availability and scalability:

- To improve availability, Megastore uses a fault-tolerant, synchronous log replicator.
- To improve scalability, Megastore divides the data into numerous mini databases. A replicated log in a NoSQL datastore is implemented for each of them.
```
Consistency model: Megastore provides full ACID semantics within a partition, but it loses strong consistency across the partitions. Therefore, overall it prioritizes availability over consistency.
```
## Replication for high availability
Data replication between servers inside the same data center enhances availability by overcoming host-specific failures. However, the data center itself is vulnerable to regional catastrophes. The data should be replicated over multiple geographically distributed data centers to overcome data center-specific failures and regional catastrophes.

Some common replication strategies are as follows:

- Asynchronous primary/secondary: Write-ahead log entries are replicated to at least one secondary node by the primary node. Log appends are recognized and serialized at the primary node while being sent to secondary nodes. The primary node can handle quick ACID transactions, but there is a danger of downtime when the primary node fails and a secondary node becomes the primary node. Loss of data can also occur during this period.

- Synchronous primary/secondary: Prior to actually acknowledging updates, a primary node waits for them to be replicated to secondary nodes, enabling failover without data loss. External systems must identify primary and secondary node failures in a timely manner.

- Optimistic replication: This approach has no primary node. Any node can accept the changes and propagate them asynchronously to other nodes. This approach provides high availability and better latency. However, the transactions are not possible since the global mutations ordering are not known at the time of the commit.

```
In computer science, write-ahead logging (WAL) is a family of techniques for providing atomicity and durability (two of the ACID properties) in database systems. A write ahead log is an append-only auxiliary disk-resident structure used for crash and transaction recovery. The changes are first recorded in the log, which must be written to stable storage, before the changes are written to the database. [source: Wikipedia]
```

```
Question
What is the difference between synchronous and asynchronous replication?

Answer
In synchronous replication, the primary node waits for acknowledgments from secondary nodes about updating the data. After receiving acknowledgment from all secondary nodes, the primary node reports success to the client. Whereas, in asynchronous replication, the primary node doesn’t wait for acknowledgment from the secondary nodes and reports success to the client after updating itself. 
```

### Paxos
Megastore doesn’t use the three replication strategies above since all of them have some disadvantages, like data loss. Megastore uses the Paxos algorithm for replication which has the following characteristics:

- It is a fault-tolerant consensus algorithm.
- There is no primary node but a group of similar peer nodes.
- A write-ahead log can be replicated to all the nodes.
- Any node can start the read or write operation.
- Every log adds the changes only if the majority of the nodes acknowledge these changes.
- The remaining nodes that didn’t acknowledge will eventually acknowledge the changes.
- There is no notable failed state due to the algorithm’s built-in fault tolerance.



#### Issues with Paxos
There are also some issues with using Paxos:

- If we have a replicated log over a wide area, it might suffer high latencies due to additional communication restricting the throughput.
- The progress is hindered if no replica is updated or the majority of the replicas don’t acknowledge the changes.
To overcome the issues above and to improve the availability and throughput, we introduce numerous replicated logs, each controlling its own partition of the dataset (instances of having one instance of Paxos for the whole dataset).
```
Note: The original Paxos algorithm is always safe (meaning the Paxos algorithm works correctly under all conditions) but might not be always live (meaning the consensus progress can stop under some failures).
```

```
original Paxos algorithm: Lamport, Leslie. “Paxos made simple.” ACM SIGACT News (Distributed Computing Column) 32, 4 (Whole Number 121, December 2001) (2001): 51-58.
```

## Partitioning and locality for scalability
To upgrade our replication strategy and improve the overall performance of the underlying datastore, we offer apps fine-grained control over the partitioning and locality of their data.

### Entity groups
To increase throughput and minimize failures, the data is partitioned into entity sets in Megastore. These sets are replicated over a vast geographic area both independently and synchronously. Within every data center, the partitioned data is kept in a NoSQL datastore, which is scalable. The illustration below shows the replication in those entity groups.

[Scalable replication](./scalable_rep.jpg)

In an entity group, entities are changed using single-phase ACID transactions (Paxos is used to replicate the commit record). While Megastore’s efficient asynchronous messaging is often used for inter-entity group operations, more costly two-phase commits can also be used. A transaction in a transmitting entity group queues one or more messages, and transactions in recipient entity groups read those messages atomically and execute the resulting modifications.

Notably, instead of using asynchronous communication between physically separated replicas, Megastore uses asynchronous messaging between groups of logically separate entities. Synchronous and consistent replicated operations generate all communication across data centers.

Indexes inside an entity group follow ACID semantics, but those between different entity groups have weaker consistency. The illustration below shows the different actions within and between entity groupings.

```
Single-phase ACID transactions: The transactions where all rows involved in the operations of a transaction exist in a single shard of a distributed database. (Source: https://www.yugabyte.com/tech/distributed-acid-transactions/ )
```

[Operations between entity groups](./op.jpg)

### How to select entity group boundaries
Entity groups are used to establish the a priori data grouping that will facilitate swift operations. The boundaries should not be too fine-grained since they may need much more cross-group operations, but putting excessive unrelated data in a single group serializes unrelated writes and reduces performance.

### Physical layout
Google’s Bigtable provides us with fault-tolerant, scalable storage inside a datacenter. It distributes the operations over numerous rows to support random read/write throughput.

We reduce latency and increase throughput by allowing applications to manage data placement which is done by selecting Bigtable instances and specifying locality in an instance, for example, by using Bigtable’s column-families feature.

To reduce latency, apps aim to keep data close to the customers and replicas close to one another. Every entity group is assigned to the location from where it is accessed frequently. Within that area, data centers with discrete failure domains are assigned a triplet or quintuplet of replicas.

Bigtable rows store adjacent ranges of an entity group’s data for low latency, cache efficiency, and throughput. This helps fulfill a range of queries from within a shard.

Let’s dig deep into the data model of the Megastore in the next lesson.
