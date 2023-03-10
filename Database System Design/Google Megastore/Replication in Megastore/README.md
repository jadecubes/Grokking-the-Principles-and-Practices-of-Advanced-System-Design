# Replication in Megastore
The replication of Megastore gives a unified picture of the data kept in its dependent copies. No matter which replica a client accesses to begin an operation, read or write operations can be started from any replica, and ACID semantics will be retained. Replication is performed per entity group by synchronously replicating the transaction log of the group to a quorum of replicas. Usually, writes need one round of communication between data centers, while reads, in a healthy case, run locally. Megastore guarantees the following for reads:

- The read operation will always be based on the write that was acknowledged at last.
- Following the completion of a write operation, all future read operations will reflect the modifications made by the write.


## Paxos
The Paxos algorithm is a method for reaching an agreement on a single value between a set of replicas. It handles messages that are delayed or reordered, as well as replicas that fail by simply stopping. For the algorithm to proceed, most replicas must be active and reachable, i.e., it can tolerate up to F failures with 2F + 1 replicas. Once the majority has decided on a value, all further attempts to access or modify the value will provide the same outcome.

Paxos is commonly used by databases to replicate a transaction log, with a distinct Paxos instance utilized for each point in the log. Here are the advantages and disadvantages of Paxos:

- Advantage: It handles messages that are delayed or reordered, as well as replicas that fail by simply stopping.

- Disadvantage: It results in high latency since it demands multiple rounds of communication.

Due to the drawback mentioned above, Paxos-based real-world systems lower the number of round trips necessary to make the algorithm feasible. First, we will discuss how master-based systems employ Paxos and then examine how Megastore enhances the efficiency of Paxos in these systems.


### Master-based approaches
Many systems utilize a designated master through which all reads and writes are routed to reduce latency. The master’s state is constantly updated because it is involved in all writes. It does not need to communicate with the network to offer reads of the current consensus state.

- Disadvantage: Dependence on a master restricts reading and writing flexibility. it causes a bottleneck. The election of a new master also slows down the process.


## Megastore’s approach
Let’s discuss the improvements and developments in Paxos that made it suitable for Megastore.


### Fast reads
Megastore allowed local reads from anywhere. These local reads improve utilization, provide low latencies in all areas, allow for fine-grained read failover, and simplify programming.

Megastore created a service called the Coordinator, which was hosted on servers in each replica’s data center. The coordinator server keeps track of a group of entities for which its replica has observed all the updates made using the Paxos algorithm. The replica contains enough state to serve local reads for entity groups in that set.

### Fast writes
Megastore adopts the pre-preparing optimization employed by master-based techniques to enable quick single-round trip writes. It uses leaders instead of a dedicated master.

For each log position, Megastore executes a separate Paxos algorithm instance. (It is multi-Paxos. Multi-Paxos can be optimized to skip the preparation phase and directly enter the acceptance phase when the same proposer makes continuous proposals. Source https://www.alibabacloud.com/blog/paxos-raft-epaxos-how-has-distributed-consensus-technology-evolved_597127) The leader for every log position is a distinguished replica chosen together with the consensus value of the previous log position. The leader decides which values are allowed to use proposal zero. The writer who submits the leader with a value first has the power to request each replica to adopt that value as proposal 0. All other writers are forced to rely on two-phase Paxos.

We reduce the delay in communication between the writer and leader as the writer syncs with the leader and then sends the value to replicas. We based our method for determining the next write’s leader on the fact that most apps consistently send writes from the same location. As a result, a simple yet useful heuristic emerges: use the closest replica.

```
Note: Paxos algorithm’s implementation has proved challenging. See the following paper for more details: Chandra, Tushar D., Robert Griesemer, and Joshua Redstone. Paxos made live: an engineering perspective. In Proceedings of the twenty-sixth annual ACM symposium on Principles of distributed computing, pp. 398-407. 2007.
```
### Types of replicas
The replicas discussed up to this point have been full replicas. These replicas include data of all entities and indexes and can support current reads. Megastore also supports two other types of replicas:

#### Witness replicas
- They can keep the write-ahead log and also vote in Paxos rounds
- They have reduced storage costs because they do not write logs or keep entity data.
- They are able to avoid the need for an additional round trip when they are unable to acknowledge a write since they lack a coordinator.

#### Read-only replicas
- They cannot vote.
- They contain snapshots of the data.
- Read-only replicas can distribute data over a large geographic area without affecting the speed at which data is written, which is useful for reads that don’t require the most up-to-date information.

## Architecture
The major elements of Megastore for a scenario with two complete replicas and one witness replica are depicted in the illustration below:


[Example of a Megastore architecture](./arch.jpg)

A client library and auxiliary servers are used to deploy Megastore. The client library, which includes the implementation of Paxos and other algorithms, is connected to various applications. These algorithms assist in tasks such as choosing a replica for read operations, synchronizing lagging replicas, and more.

There is a defined local replica on each application server. By sending transactions directly to the local Bigtable, Paxos operations on that replica are made durable by the client library. The library sends Paxos operations to intermediary replication servers that do not retain state and communicate with their own local Bigtables. This helps to minimize the number of wide-area round trips.

A write may not be successfully completed if there are issues with the client, the network, or Bigtable. Periodically, replication servers look for unfinished writes and provide Paxos no-op values to finish them.

## Data structures and algorithms
Let’s discuss the data structures and algorithms necessary to get from consensus on a single value to a working replicated log.


### Replicated logs
Each replica keeps track of changes and metadata about the log entries for the group. We allow replicas to accept out-of-order proposals to ensure that they may participate in a write quorum while recovering from past disruptions. In Bigtable, log entries are stored as independent cells.

When a log replica has an incomplete log prefix, we refer to it as having “holes.” The illustration below shows this scenario containing several typical log replicas for a Megastore entity group (Source). Since each replica has been informed that the other replicas will never seek a copy, log positions 0-99 have been entirely scavenged, and position 100 is partially scavenged. All replicas recognize log position 101. Log position 102 discovered a low quorum in A and C. Position 103 is notable because it is accepted by A and C, leaving B with a hole at 103. A contradictory write attempt at position 104 on replicas A and B has blocked consensus.

[The write ahead log](./writeahead.jpg)

### Reads
Before a read or a write operation can be initiated, at least one replica must be updated to reflect all the changes that have been committed to the log. This involves copying and applying all previous mutations to the replica. This is referred to as the catchup process. The algorithm for a current read is as follows:

1. Query local: This checks if the entity group is up-to-date locally by contacting the coordinator of the local replica.
2. Find position: This determines and picks a replica that has applied through the highest possibly-committed log position.
  I. Local read: The highest approved log position and timestamp should be read from the local replica if step 1 shows that it is consistent.
  II. Majority read: The highest log position that any replica has seen should be used to choose a replica to read from if the local replica is out-of-date (or if steps 1 or 2(I) fail). We choose the replica that is the most responsive or updated and not necessarily the local replica.
3. Catchup: Once a replica has been chosen, it should be caught up to the highest known log position as follows:
  I. Read the value from a different replica for each log position for which the selected replica is unaware of the consensus value. Request a no-op write from Paxos for all log positions that do not have a known-committed value available. Paxos will force the majority of replicas to agree on the no-op or the previously suggested write.
  II. To progress the replica’s position to the distributed consensus state, apply the consensus value of all pending log positions sequentially.
4. Validate: This sends a validation message to the coordinator stating that the pair of the entity group and replica now accurately represent all committed writes if the local replica was chosen and was not already up-to-date. There is no need to wait for a response because if the request does not succeed, the next read will try again.
5. Query data: Using the timestamp of the given log location, read the selected replica. If the chosen replica gets offline, choose another, conduct catchup, and read from it instead.

[Reads](./reads)

### Writes
Megastore monitors the next available log position, the previous write’s time, and the subsequent leader replica after executing the read algorithm. When a commit is made, all the changes that are waiting to be made to the state are grouped together, along with a timestamp and the name of the next leader candidate, and presented as the proposed consensus value for the next log slot. If this value is chosen by the distributed consensus, it is copied to all full replicas. Otherwise, the whole transaction is terminated, and the read phase must be restarted from scratch.

Coordinators maintain a record of which entity groups are up-to-date in their replicas. The replica’s coordinator removes the key of the entity group if the write is rejected on that replica. This procedure is known as invalidation. All full replicas for that entity group must either approve or have their coordinator invalidated till a write is deemed committed and ready to be implemented. Here’s the write algorithm:

1. Accept leader: This requests that the value is accepted as proposal number 0 by the leader. Proceed directly to step 3 if it was successful.
2. Prepare: This runs the Paxos prepare phase on any replica that has a proposal number greater than the one that has been seen thus far at this log position. If several new proposals are found, substitute the value being recorded with the one with the greatest number.
3. Accept: This urges the rest of the replicas to accept the value. If most of the replicas don’t accept, go back to step 2 after performing a randomized backoff.
4. Invalidate: For any full replicas that did not accept the value, the coordinator should be invalidated.
5. Apply: The modifications in values will be applied to the maximum feasible number of replicas.

[Writes](./writes)

## Coordinator availability
Coordinator processes are run within every data center and solely maintain track of their local replica. It might seem that the failure of a full replica, including both the coordinator and Bigtable, will lead to inaccessibility in the write algorithm described above, since each full replica must either accept the write or have its coordinator invalidated.

In practice, this is not a frequent issue. The coordinator is often more reliable than a Bigtable since it is a straightforward procedure with no permanent storage and no external dependencies. However, the coordinator might still be inaccessible due to network or host issues.

### Failure detection
To determine whether other coordinators are available, functioning properly, and can be reached to handle network partitions, the coordinators employ the out-of-band protocol.

Megastore uses the Chubby lock service. At initialization, the coordinators get particular Chubby locks in remote data centers. A coordinator must keep the maximum number of locks to handle requests. In the event of a crash or network partition that causes the coordinator to lose a significant number of locks, it will return to a safe default state in which it will treat all entity groups under its control as being out of date.


Writers check whether a coordinator has lost their locks. This is how they are protected against failures caused by coordinators. If this is the case, the writer is aware that the coordinator will deem itself invalidated once it regains control of the locks.

This algorithm runs the risk of a short write outage of around tens of seconds if a data center hosting live coordinators is unexpectedly unavailable. This is why before any writes can finish, all writers must stand by until the coordinator's Chubby locks expire. In contrast to the situation that occurs after a master failover, writes and reads may continue normally even while the state of the coordinator is being rebuilt. This risk of a short and infrequent outage is worth it because it allows for fast, local reads most of the time.


### Validation races
Protocols for accessing (read/write) the coordinator must not only address issues of availability but also handle various race conditions. Validate messages must be treated carefully, while invalidate messages are always secure. Higher-numbered invalidates always take precedence over lower-numbered validates. There are additional races that occur as a result of a crash between an invalidate by a writer at point n and a validate at some point m<n
. We use a different epoch number for every iteration of the coordinator to identify crashes, and validates are only permitted to alter the coordinator state if the epoch hasn’t changed since the coordinator was last read.

The following points minimize the majority of the challenges associated with running the coordinator:

- Coordinators are less complex operations than Bigtable, have less dependencies, and are therefore more available.
- Coordinator’s uniform workload makes them inexpensive and easy to use.
- The minimal network traffic of coordinators enables the use of a high network QoS with stable connectivity.
- Operators have the ability to deactivate coordinators for maintenance or when they are not functioning properly. For some monitoring signals, this is done automatically.
- Usually, network partitions and node unavailability are detected by a quorum of Chubby locks.


## Write throughput
The Paxos Megastore implementation involves intriguing trade-offs in system behavior. For a given per-entity-group commit rate, the higher latency imposed by synchronous replication increases the risk of conflicts.


- If the rate of writes to a particular entity group is limited to a few per second, the likelihood of conflicts occurring will be low. This limitation is usually not a problem for applications whose entities are only being changed by a few users at once. To increase write throughput, many of Megastore’s target clients use carefully done sharding of entity groups or place replicas in the same region, which reduces both latency and the likelihood of conflicts.
- Another typical batching strategy is the bulk processing of Megastore queue messages, which reduces conflict rates while boosting aggregate throughput.
- Applications can utilize the fine-grained advisory locks provided by coordinator servers for application groups that must consistently surpass a few writes per second.


## Operational issues
Megastore’s performance may suffer if a specific full replica becomes unreliable or loses connectivity. We have several options for dealing with these errors, which include the following:

- In the event of an outage, the first priority should be to deactivate client requests at the affected replica and redirect these requests to application servers close to other replicas.
- Rerouting traffic is ineffective if unhealthy coordinator servers retain their Chubby locks. The next step is to deactivate the coordinators of the replicas. This will make sure that the problem has as little effect as possible on write latency.
- Disabling a replica fully is a drastic measure that is seldom utilized since it prevents clients and replication servers from attempting to establish a connection with the replica.
It marks the end of the replication design of Megastore. Let’s evaluate our Megastore design and check whether or not it fulfills the non-functional requirements in the next lesson.

