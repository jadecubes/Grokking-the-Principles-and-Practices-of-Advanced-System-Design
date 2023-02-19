# ZippyDB Design
In Tectonic design, metadata management is critical in achieving our goals of scalability, availability, and durability. A special-purpose key-value store (ZippyDB) is the cornerstone of our Metadata Store. In this lesson, we’ll focus on ZippyDB’s design. Following is the overall architecture of the Tectonic.

[The architecture of Tectonic](./tectonic.jpg)

## ZippyDB–the key-value store
ZippyDB is a general-purpose, consistent, and geographically distributed key-value store created by Facebook on top of RocksDB. ZippyDB is an end-to-end managed service that provides usual key-value operations (get, put, del, etc.) because of the underlying RocksDB storage engine, ZippyDB enables the system to perform a large amount of write operations efficiently while providing good performance for the read workload.

```
ZippyDB: https://www.youtube.com/watch?v=ZRP7z0HnClc
```

```
RocksDB is a consistent key—value store–a fork of Google’s LevelDB. It uses log-structured database engines for maximum performance. It is optimized for fast and low-latency storage devices. It also provides basic and advanced database operations. Source: RocksDB’s official website.
```

```
Before ZippyDB, Facebook’s different teams were using RocksDB. All these teams faced the same challenges, such as consistency, failure recovery, fault tolerance, replication, and many more, at the individual level. It was time-consuming, redundant, and a waste of resource for each team to solve the same issues with their own solutions. Therefore, Facebook had to build ZippyDB on top of RocksDB to provide a unified solution for each team. Instead of using RocksDB, all the teams interact with ZippyDB to manage their data. In that sense, the goal of making one unified key-value store, that could be configured for different use cases, matches that of Tectonic itself.
```

```
Note: RocksDB is primarily an efficient storage engine that can be embedded in other applications as a library. It frees the programmers from the messy details of efficient storage and lets them concentrate on their specific problems. RocksDB is optimized for write-heavy workload by using log-structured storage. The read performance is good because of the use of Bloom filters.
```

```
Bloom Filters: This is is a probabilistic data structure that saves space and can be used to determine whether an element is a part of a set or not.
```

ZippyDB uses basic key-value operations, snapshot reads, and partial read and write operations. ZippyDB uses partial read and writes for low latency and durability (we’ll learn about it more in an upcoming lesson). To avoid complex operations, we have used basic key-value operations such as get, put, and delete in our system. Our system gets multiple read requests on the same data, and we have used snapshot reads to handle that.
```
Snapshot reads: The process of creating a copy of the content which is going to be read.
```

```
Partial reads and writes: The process of reading or writing partial data from/to a storage node. The partial append, if not handled carefully, can leave different sizes of replica chunks which can cause inconsistency.
```

A snapshot read is an extremely cheap operation that uses snapshot handlers to perform multiple reads that are consistent with each other. When read or write operations occur simultaneously on the same file, a snapshot of that file is generated, and the client with a read request is propagated to the snapshot, as shown in the slides below. (In some sense, snapshots make a replica of data as needed and enable non-blocking reads when writes are in progress.) A new client will also go to the block, but when it will realize that the block is locked, the snapshot handler will redirect that client to the snapshot so that it can start reading.

[ZippyDB](./zippydb)

Reading or writing partial data can leave data inconsistent. We implement read-modify-write on the server side to avoid inconsistency in partial write operations. We can perform partial read operations by combining data of different blocks and trying to get most of the data from the same node.

ZippyDB has the following components:

1. Store: This is a storage node for storing data.
2. Shard management service: This is used to manage, relocate and replicate shards.
3. Data shuttle: This is used to send the shards on the data replication pipeline to another data shuttle so that the replica of the shard can be generated on another store.
4. Shard management client: This communicates with shard management services and data shuttle for shard replication.
5. Directory service: This contains all the information regarding the shard. This service helps with the easy discovery of shards.
6. Request handler: This checks where the request should go. For a read request, it is moved toward the store to read requested keys and values, and for a write request, it forwards it to the data shuttle.
7. Zippy-clients: This uses the ZippyDB-client library to generate read or write requests.

[The architecture of ZippyDB](./zippydbarch.jpg)

### Architecture
In ZippyDB, shards are the units on a tier containing data regarding a use case. On the server side, the shard is the basic unit of management, and each shard contains multiple replicas of data placed geographically for fault tolerance. Depending on the configuration, we can either use Paxos or async replication.
```
The units in which ZippyDB is deployed are known as a tier. It contains both the storage and the compute resources that are geographically placed for fault tolerance.
```

```
Paxos: An algorithm that solves the consensus problem for majority quorum acknowledgment from followers to perform synchronous replication.
```

```
Async replication: In this process, the data is written on the primary node and acknowledges the client about the successful write when secondary nodes haven’t acknowledged the successful write operation.
```

A subset of replicas within a shard is synchronously replicated between nodes. These replicas are a part of the Paxos quorum group. Most of the Paxos replica logs provide persistence of data in write operations and also write the data to RocksDB on the primary node. Once the write operation is completed, the client gets notified about the write operation, and it provides highly durable writes.

[ZippyDB](./internal)

Some of the shard replicas (possibly a remote one) that were left out from the synchronous replication are managed via asynchronous replication.

Additional replicas can be generated to manage read load. Such replicas are called followers, and they don’t participate in write operations. They just observe the current state of a shard and replicate it locally. We can use this strategy to bring in updates inside a data center, and then others can sync up locally from the followers without generating any traffic to original shard Paxos groups. Although followers can be behind the primary replica, these replicas can handle low-latency readings.

Clients can use an appropriate combination of these synchronous and asynchronous replication schemes.They can also configure between the number of follower replicas and quorum size, where the quorum size tells us the number of followers that have to acknowledge the leader before the acknowledgement is sent to the client. It enables users to choose their ideal combination of consistency, durability, read performance, and write performance.

[Async](./async)

## Data consistency model
ZippyDB divides time into small epochs. For each epoch, ZippyDB assigns a known primary replica for a shard. This primary replica is the Paxos leader for writing. All the writes come to the primary, which is then synchronously written to the majority quorum and asynchronously to some remote replicas. Such a mechanism provides strongly consistent write operations. Clients might opt for a looser consistency model if they need to.

For reads, clients have three options. For consistent reading, they can go to the current primary of the shard. Clients can also go to some secondary replicas for reading if they can tolerate somewhat stale information. Such replicas eventually become consistent. ZippyDB actually provides stronger guarantees that a secondary replica will have a bounded staleness. Once that bound is reached for a secondary, it stops serving reads. Clients can also get read-your-writes consistency by using a special tag that was returned in the previous write. Now, when the client sends the read request with that tag/token to the secondary, it should get the updated data. If that secondary has not received those updates, the secondary will not return any data.



## ZippyDB’s working
We’ll now examine ZippyDB’s components in detail.



### Data shuttle
The data shuttle is the replication layer connected with the replication pipeline to improve data dependency. A single data shuttle instance manages the replication of all the shards present on that server. The abstraction of the data shuttle and replication is powered by Paxos. We have the following three roles:

1. Primary
2. Secondary
3. Follower
These roles are assigned on a per-shard basis, and it is dynamic since it constantly changes because the machine goes online and offline. We can lease a primary in case of a single writer and can also configure n number of secondaries for more durable writes and more performant reads.
 
### Shard management service
Shard management not only manages the shard but also manages replication and performs load balancing. This system provides the optimal shard placement and management in addition to managing roles. It not only tells the data shuttle that it owns the shard but also tells the role of the shard. If a server goes offline, it has to move all the primary roles to another server. An automated load balancing technique ensures that workload is even on the entire tier.


### Shard management client
It talks to the shard management services and pushes all the replication configurations to the data shuttle.



### Directory service
All the information regarding role assignment, shard placement, and many more are stored so that it can be accessed publicly.

### Store
The store is built on top of RocksDB. It is a compact representation of write requests streaming into the data shuttle so that we can provide a read facility as well. It is plugged into the data shuttle, and the data shuttle orders the number of the write request and pushes the data to the store, so the reader will read the same way the data was written. The data shuttle can plug in multiple stores using the store API.

### Request handler
The request handler sends read requests directly to the store and sends write requests to the data shuttle, which does its replication. After writing the data, it returns the acknowledgment of request completion.

### Zippy-clients
Zippy-Clients are application services embedded with the ZippyDB client, a local API, which talks to the directory service to get the shard information. After retrieving the shard information, it forwards their request to the request handler.

The following illustration summarizes the main steps of the shard management, replication, read, and write workflow:

[The skeleton of the ZippyDB server](./zippydbclients)

## How Tectonic uses ZippyDB
ZippyDB hosts the file system's metadata, and its durability and integrity are critical for Tectonic. Tectonic uses a separate instance of ZippyDB for security and performance-isolation reasons. This instance uses strongly consistent writes and reads. ZippyDB, an end-to-end managed service, can be considered a BlackBox that provides durable, scalable, and highly performant key-value services to Tectonic.

In the next lesson, we’ll continue with Tectonic design.
