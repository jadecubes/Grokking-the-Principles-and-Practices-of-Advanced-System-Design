# Detailed Design of Spanner
Universe is the term for a Spanner deployment. Since Spanner handles data on a global scale, only a select few universes will be active at any given time. The components of Spanner are as follows:

1. Optimized Spanner server organization: This performs automatic resharding based on the data size and load and facilitates the client's request of read and write.

2. TrueTime API: This is an API that provides the time within well-defined error bounds. It can be used to ensure strong external consistency and global serialization.

3. Strong network infrastructure: We should have a redundant and highly available network that provides global connectivity to make high performance for Spanner possible.

[Components of Spanner]

First, we will understand how the components work together in Spanner's deployment. In the next lessons, we will learn about the TrueTime API, which makes Spanner unique. Moreover, we will also learn how a strong network helps Spanner to be a strongly consistent database.

## Components of the Spanner server organization
The components involved in the server organization of Spanner are as follows:

- Zone: A zone is a unit for administrative deployment. Both sets—the possible replication sites and the zones—are the same. As new data centers are brought online and older ones are shut down, zones can be added to or withdrawn from an active system. A zone is also a unit of physical isolation. One or more zones may exist within a data center, for instance, if data from various applications needs to be segregated across several groups of servers within the same data center. A zone consists of a single zonemaster and is between a 100 to a few thousand servers.

- Zonemaster: It manages a single zone and serves data to the clients.

- Location proxy: The clients use the per-zone location proxies to find the spanservers designated to serve their data. There is more than one location proxy per zone to avoid the bottleneck of requests directed to only one proxy.

- Universe master: This is to facilitate interactive debugging. The master functions as a console shows the current state of all zones.

- Placement driver: It is responsible for the automated, minute-by-minute data transfer between zones. From time to time, it will communicate with the spanservers to determine if any data needs to be relocated to accommodate changed replication constraints or to distribute the workload better.

- Spanserver: It is responsible for serving data to the client. It consists of tablets. A Spanner tablet is a data structure like Bigtable’s tablet. Each server has between a 100 and 1000 instances of tablets.

The implementation of the universe master and placement driver are both singletons. Only one instance will be actively handling the responsibilities of both. A shadow replica will take over if the universe master or placement driver fails. Moreover, the failure of these components is not a single point of failure. It can slow down the transfer of data between zones.

The following illustration shows the placement of servers in the Spanner universe:

[The Spanner server organization]

```
Question 1
What would happen if two zones are in the same data center and that data center is taken down due to a natural disaster?

Answer
To avoid data loss during an outage of the data center, the users should replicate the data in multiple regions of different data centers. Spanner’s regional configurations allow replicating the data between multiple regional zones. However, with multiple regions, the data is saved at physically different places. Moreover, the availability of the data in the chosen region also improves.
```

```
Question 2
The implementation of the universe master and placement driver are both singletons. What happens if they fail?

Answer
The singleton service will export only one instance of itself. However, that does not mean that failover will not happen in case of a failure.

On the other hand, the jobs of both the universe master and placement driver are such that a short outage, of a few seconds, for example, will not adversely impact the operations of the rest of the system.
```

## The spanserver software stack
We'll learn about the spanserver design to show how replication and distributed transactions are associated with Bigtable's design.

Each zone will function like a cluster of Bigtable servers. Each spanserver maintains several tablet instances ranging from one hundred to one thousand. Like the tablet abstraction of Bigtable implements a bag of mappings, so does a Spanner's tablet, as shown below:
```
(key:string, timestamp:int64) → string
```

Spanner and Bigtable offer internal routing and sharding as part of their fully managed service. Spanner is an extensible service for managing relational databases, including transactional consistency, SQL query support, and secondary indexes. While Bigtable is a NoSQL database that can scale horizontally. Spanner resembles a multi-version database more than a key-value store, and we want our data to be consistent and available. Since time is relative and the time of the servers cannot be trusted because of the Network Time Protocol (NTP), we will use the TrueTime API to provide us with a globally consistent timestamp. Moreover, we also need to store our write-ahead log. We will utilize a distributed file system called Colossus to store our data.


```
The Network Time Protocol (NTP) is a networking protocol for clock synchronization between computer systems over packet-switched, variable-latency data networks. [source:Wikipedia]
```
### Using Paxos
The Paxos consensus can ensure consistency. Each server implements a tablet with a single Paxos state machine, allowing for replication to take place. The metadata and logs of every state machine are stored in its tablet. The implementation of Paxos in Spanner supports time-based leader leases that enable long-lived leaders. The default lease time is 10 seconds. Spanner stores two copies of every Paxos write, one in the corresponding tablet and the other in Paxos logs. While Paxos applies writes in order of their receipt, Spanner's implementation of Paxos is pipelined to increase the database's throughput when WAN latencies are present.
The following slides illustrate how Spanner uses Paxos:

[The use]

To create a bag of mappings that are replicated in the same way over and over, the Paxos state machines are employed. Each duplicate has its corresponding tablet where the key-value mapping state is kept. To perform a write, the Paxos protocol must be initiated at the leader, while readers can use any sufficiently up-to-date replica to get the state directly from the underlying tablet. This collection of replicas forms a Paxos group.

An elected leader replica handles all incoming write requests from the client. The leader replicates the write requests to the replicas of the group through a Paxos round. The leader replicates the write requests to the Paxos group's replicas through a Paxos round. Followers are the remaining replicas, and they can serve the read requests. The following illustration demonstrates this:

[The spanserver software stack]

### Concurrency control via lock table
A lock table is implemented on each server to enforce the concurrency control mechanism at leader replicas. The two-phase locking state is kept in the lock table, which associates key ranges with the states of the lock. A long-lived Paxos leader is essential for the smooth operation of the lock table. Long-lived transactions such as report generation, which can take minutes, are challenging in distributed systems as the write and read operations are on hold until such transactions are completed. Other operations skip the lock table while transactional reads and synchronous operations acquire locks from it.
```
Question
A long-lived Paxos leader is essential for the smooth operation of the lock table. What happens to the lock state if the leader server dies?

Answer
Since the leader has taken a lease for a certain amount of time, the participants will wait until that lease ends, and they’ll elect a new leader after that time.

The old leader can also save the lock states in the underlying Paxos log. That means the new participant can easily rebuild the lock table state and effectively resume where the older leader left off.
```

### Transaction manager

Each spanserver will run a transaction manager to facilitate distributed transactions at every replica that is a leader. A Paxos group has a participant leader that is implemented using the transaction manager, and the other replicas are called the participant followers. Since Paxos and the lock table collectively ensure transactionality, the involvement of the transaction manager can be skipped if a transaction only involves a single Paxos group. Leaders of all participating Paxos groups collaborate to execute a two-phase commit if the transaction involves more than one group. The coordinator is the head of one of the participant groups, and other nodes in that group are known as coordinator followers. Each transaction manager's state persists in the underlying Paxos cluster and is replicated.

In the next lesson, we will learn more about the Spanner server organization by exploring the directories and placement of the data.
