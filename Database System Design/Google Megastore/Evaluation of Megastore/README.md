# Evaluation of Megastore
To meet our functional requirements, we employed a variety of strategies. However, we must determine if we have met the non-functional requirements. We’ll go through some techniques we’ve used to meet non-functional requirements.


## Scalability
Megastore was one of the first storage systems to use replication via Paxos to ensure data consistency between data centers. It also meets the demands for scalability and performance for cloud-based web applications. Partitioning the database into entity group sub-databases provides conventional transactional characteristics for the majority of operations while permitting storage and throughput scalability.

Megastore divides the data into numerous mini databases. A replicated log in a NoSQL datastore is implemented for each of them. To increase the overall speed of the underlying datastore and to upscale our replication strategy, Megastore provides applications with fine-grained control over data-partitioning and locality. In an entity group, entities are changed using single-phase ACID transactions (Paxos is used to replicate the commit record).

For scalable fault-tolerant storage within a particular data center, Megastore utilizes Google’s Bigtable, which allows handling arbitrary read and write throughput by distributing operations across several rows. Bigtable divides a table into tablets, each of which holds related data with a continuous region of rows.


## Availability
Megastore uses the Paxos algorithm for replication which has the following characteristics:

- It is a fault-tolerant consensus algorithm.
- There is no primary node but a group of similar peer nodes.
- A write-ahead log can be replicated to all the nodes.
- Any node can start the read or write operation.
- Every log adds the changes only if the majority of the nodes acknowledge these changes.

Megastore improved availability and throughput using numerous replicated logs. Each replicated log controls its dataset partition.

The data is stored on Bigtable, which uses the GFS file system, a replicated datastore. Doing so makes Megastore’s data and metadata fault-tolerant within a data center as well.


## Consistency
Megastore uses synchronous replication for a consistent view of the data. Indexes inside an entity group follow full ACID semantics, but those between different entity groups have weaker consistency. To get strong consistency across entity groups, the end programmers might need to rely on their own code (Such as two-phase locking and two-phase commits).

## Conclusion
Megastore was developed to fulfill the storage needs of modern interactive web services. Megastore adopts the middle road in the RDBMS vs. NoSQL technology field. It partitions the datastore and replicates each partition individually, offering complete ACID semantics inside partitions but only limited consistency guarantees between them. It includes standard database functionality such as secondary indexes. Megastore only offers certain features that can be efficiently handled by the partitioning system within acceptable latency for users and those that align with the system’s capabilities.

Paxos is used for synchronous wide-area replication, allowing for a lightweight and quick failover of individual activities. Bigtable is used as the scalable datastore, with additional primitives such as ACID transactions, indexes, and queues added.

Megastore performs well in production. For several years, Google has utilized it internally on more than 100 commercial apps to handle more than three billion writes and 20 billion read operations daily and store a petabyte of data across multiple worldwide data centers.

### System design wisdom in Megastore system
Megastore uses what Bigtable provided but goes beyond that. Bigtable only has per key (or per row) atomic transactions, while Megastore can have transactions on a range of keys (rows). Two important lessons are in play here. First, we evolve the system by building on top of what already went before (instead of reinventing the wheel). Second, the incremental improvement is not only manageable (in terms of dollar cost and effort) but also has a better probability of success. Megastore relies on Bigtable but provides strong consistency on transactions that remains within a partition. Transactions among any rows anywhere would have been a difficult task to achieve in one go (we will see another chapter on how Google took that leap from Megastore to Spanner). Such leapfrogging brings controlled progress and innovation.

According to the CAP theorem, a large, geographically dispersed distributed system cannot simultaneously guarantee strong consistency, availability, and partition tolerance to all nodes. However, Megastore demonstrates that, by making wise decisions (for example, differentiating transactions within and across entity groups, introducing a coordinator per data center, etc.), we can have different trade-offs for different use cases. When partition happens, Megastore prefers strong consistency over availability within a Paxos group amongst a partition. However, across entity groups, it gives more weight to availability than strong consistency.
