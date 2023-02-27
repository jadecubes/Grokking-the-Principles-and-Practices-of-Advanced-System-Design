# Evaluation of Bigtable
To meet our functional requirements, we employed a variety of strategies. However, we must determine if we have met the non-functional requirements. We’ll go through some of the techniques we’ve used to meet non-functional requirements.

## Scalability
Bigtable divides a table into tablets, each of which holds related data with a continuous region of rows. At first, each table has only one tablet. As the table expands, it is divided into several tablets. The Bigtable master is in charge of allocating tablets to tablet servers. Tablet servers are in charge of managing read and write requests to the tablets to which they have been allocated, as well as separating tablets that have become too big. It is really simple to add new tablet servers to a Bigtable cluster. The master server allocates the tablets to any newly installed tablet server from the pool of unallocated tablets or from other tablet servers.


The master checks the load (read/write requests) on the tablet servers periodically. If there is too much load on a tablet server for a specific tablet, the master splits those busier tablets and merges tablets with lesser loads, and redistributes them between different tablet servers to balance the load.

```
Note: Since Bigtable is based on the GFS, it is also meant to be used within a data center. For across-data center databases, we will study the Spanner system later on.
```
## High number of read/writes operations
Bigtable uses an SSTable file format, which enables a high number of reads and writes. Bigtable provides strong consistency for writes and reads. Since SSTables are immutable, reads are highly consistent. For static indexes, it is a great solution: reading the index, and we can read the file with one disk seek. So, Random reads are fast and easy.

Unless the required data is in memory, random writes are substantially more difficult and costly. We want to keep the quick read access provided by SSTables, but we also want to provide fast random writes. Random writes can be fast when we are writing in memory. Bigtable solves this problem of fast read/write access for huge datasets with SSTables in GFS with the help of memtable. Memtable is in the memory, where the newly committed writes are stored, and once it gets full, these writes are permanently stored in GFS.

## Availability and fault tolerance
Bigtable’s design is highly available and fault tolerant. It ensures the availability in the following ways:

- In master: The master obtains a lock in a Chubby file and keeps a lease. If the master’s lease ends at any point, it eliminates itself. When Google’s Cluster Management System discovers that there is no current master, it initiates the creation of a new one. Before functioning as a master, the new master must first get the lock on the Chubby file.

- In tablet servers: The master of Bigtable is in charge of handling the tablet servers. The master does this by verifying the state of the Chubby lock for each tablet server on a regular basis. Whenever the master discovers that a tablet server has failed, it reassigns the faltering tablet server’s tablets.

- In Chubby and GFS: Chubby and GFS are two distinct systems used by Bigtable. Each of these systems uses a replication method to increase fault tolerance and availability. A Chubby system, for example, is often composed of five servers, with one serving as the master and the rest four serving as replicas. If the master fails, one of the replicas is chosen to be the leader, reducing Chubby’s downtime. Similarly, GFS replicates data over several Chunk servers.

## Durability
Bigtable uses Google File System which is highly durable. Once data is stored in GFS, it cannot be lost unless a user deletes the data. GFS files are divided into fixed-size units known as chunks. Chunks are kept on data servers known as chunkservers. The metadata is managed by the GFS master. SSTables are broken into fixed-size blocks, which are then stored on chunkservers. For durability, each chunk in GFS is duplicated over many chunkservers.
## Conclusion
Bigtable is an example of a distributed storage solution that can handle enormous amounts of structured data, expanding up to petabytes and dividing over hundreds of commodity servers. Many Google applications, including web crawling, Google Maps, and Google Finance, use Bigtable to keep records. All such applications have extremely diverse demands for Bigtable in terms of latency and data size. But besides these varying requirements, Bigtable has delivered a flexible, high-performance solution for each of these Google apps.

Bigtable provides a different implementation from the traditional relational databases to achieve much higher data scalability with a good read-and-write performance. However, doing so sacrifices many features (such as general transactions) that database programmers expect from the system. The end programmers of Bigtable need to have a reasonable understanding of how it differs from a traditional relational store to get maximum benefits from it.

## System design wisdom in Bigtable
- Bigtable uses other services (Chubby and GFS) as the building blocks. Doing so helps to manage the design complexity by offloading data durability to GFS and locking issues to Chubby.

- Bigtable uses important locality hints from the users for many of its optimizations. Applications often know the best about their data, and our systems should provide an easy way to get such hints from the users.

- The decision to keep one log file per tablet server is an example of designing for the common case where one log has significant performance benefits during normal operations (a common case) while making a recovery operation more complicated.

- Bigtable is yet another example of a simple design that achieves its goals. By intelligently using a single master, Bigtable has a centralized control plane. By using Chubby service, and client-side caching of tablet server locations, the single master does not become a single point of failure (many features of the Bigtable keep working even if a master is momentarily not available).

The list above is not exhaustive. We can observe that some of the above points also appeared in our other systems (for example, in GFS).
