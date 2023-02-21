# Evaluation of GFS
## Scalability
The GFS architecture supports storing large files by splitting them into small chunks, which can be stored on multiple storage servers. To end users, it is just like the file is stored as a single unit.

More chunkservers can easily be added to the cluster to store an increasing amount of data. Unlike traditional systems, we don’t need to replace the disks with a larger capacity, which also involves copying data to the newer disk with the larger capacity. With the GFS architecture, admins can add more chunkservers, and the GFS master can start utilizing them.


A single GFS cluster has the capability to store hundreds of terabytes of data. If a single cluster doesn't suffice for storing all the tenants data, multiple specialized instances of the GFS cluster can be created for multiple tenants or for multiple applications of the same tenant.
```
Note: GFS provides horizontal scalability by adding more chunkservers as needed. Though such scalability is not infinite, primarily due to a single manager (the master node) in the system.
```
## Availability
GFS stores three copies of each chunk on different chunkservers by default. If one fails, the other can serve. Moreover, the replica placement strategy of GFS also supports availability. The chunkservers are placed into different machine racks. We may spread all the replicas across different chunkservers placed in the same rack. This increases the system’s availability in the event of a chunkserver failure. Still, we risk losing all replicas if the entire rack is damaged or goes offline due to a shared resource, such as power or a network switch.


The GFS master re-replicates the chunks if one or more of its replicas are lost permanently. This is how GFS makes the data available to the clients. The GFS master prioritizes the re-replication in the case when two out of three chunk replicas are gone bad. Let's see how the master makes the metadata available so that client operation could continue at all times.

The GFS master holds all the metadata that might be needed or can change due to client operations. The single master could be the single point of failure, and therefore unavailable to the clients in case of failure. GFS checkpoints the metadata state and logs the metadata change operations into an operation log that is placed on the master's hard disk and remote replicated storage. If the master undergoes a temporary failure, it restarts in seconds by loading the checkpoint (from the backup storage) and applying the log operations afterward. If just the master process fails and the master restarts on the same server, it can use the backup from the local disk.

For the system to be available in the event of a long-term or permanent failure of the master, the checkpoints and the operation logs are replicated on remote machines. The monitoring infrastructure outside GFS detects the master failure and quickly starts a new instance of the master on a different machine. The failed master, if recovered, can be used as a primary master after applying all the mutations it missed, or it can simply be discarded. However, there will only be one primary master at a time.

There are also shadow masters that serve only the read operations, thus increasing the availability. Shadow masters may lag the master by fractions of a second. In this case, the clients may read the stale data. GFS uses the shadow servers for the applications that don't mind receiving slightly stale data as well as files that are not mutated often.
```
Note: GFS can have a small window of unavailability when a master fails and a new one is being spawned. During this window, any operation that needs mutation in the metadata will not be possible. All other operations can carry on.
```
## Fault tolerance
We’ve seen how the system remains available in case of chunkserver and master failures. Both the chunkserver and the master restore their state and start in seconds if there were a temporary failure like a shutdown. In case of permanent chunkserver failures, the chunk replicas on that server are cloned from valid replicas placed on other chunkservers so that the number of replicas doesn’t go beyond a specific limit, which can cause the data to be unavailable.

In case of the permanent failure of the master, the monitoring service initiates a new instance of the master on a new machine.

Some failures can corrupt the data. In that case, the system may remain available, but the data becomes incorrect. GFS detects the corrupted data using checksums and restores the valid copy of the chunk from another replica. On reading, chunk/data checksums are validated. For files no one has read in a while, the master asks respective chunkservers to validate data integrity by checksumming.

## Durability
Once the system has acknowledged to the user that its data has been stored, that data is not lost except if the user itself deletes that data. GFS, by default, keeps three replicas of the user’s data (chunks). We’ve seen the replica placement strategy in the availability section above. The replicas are distributed across several chunkservers to save the data in the event of a chunkserver failure, and across various racks to save the data in the event of a shared resource failure resulting in the failure of an entire rack.
```
A rack consists of a limited number of chunk servers sharing resources like a network switch or power circuit.
```

When the number of available replicas for a chunk falls below a predetermined threshold, the master re-replicates that chunk. Priority is given to chunks that have lost two replicas over chunks that have lost only one replica. In this way, GFS makes sure that it doesn't lose all of the replicas of a chunk.
```
Note: GFS is designed to work inside a datacenter. If the whole datacenter goes offline (due to extreme events such as flooding, fires, an earthquake etc.), GFS will go down with it (possibly losing all data). We need more advanced techniques to replicate across geographically dispersed data centers. We need to be careful that the protocols involved are not very chatty because each round trip from the remote datacenter will add ~100 ms latency.
```
## Easy management
Having a single master with a global knowledge of all the files metadata simplifies the management. The fixed chunk size also helps manage the file's metadata simply. Larger files can be easily handled by splitting them into small chunks and storing them independently on multiple chunkservers which have the space available to accommodate that chunk.
```
Note: Operational simplicity is important in practice. It is only a matter of time that something unexpected happens, and if the systems involved are complex, it can make a bad situation worse under time pressure to fix things quickly.
```
## Throughput
GFS separates the data and the metadata flow. The data flow doesn’t need the master’s involvement. It is directly transferred between clients and the chunkservers. For data requests, GFS achieves high throughput by:

- Serving requested data chunks from multiple chunkservers
- Serving multiple requests for different chunks from multiple chunkservers

Since each chunkserver serves data for part of the total requests coming to the system, it increases its overall throughput.

For metadata, the master is the single point of contact, and it could have been a bottleneck. The GFS architecture reduces the master’s involvement in many common operations. For example, the large chunk size is the reason for lessening communication between the client and the master. The chunk lease mechanism is another design decision that helps to reduce the overhead on the master to manage/order multiple mutation requests on multiple replicas. Lazy deletion also helps reduce the master workload on delete operations and lets the master perform other priority tasks first. Also, the client caches a few chunk’s metadata that helps reduce the number of metadata requests at the master.

By avoiding the need to take locks during the writing and providing fast append operations, GFS can service many clients concurrently.
```
Note: If thousands of clients start reading and writing to the same chunk, the chunkservers responsible can become overloaded. More advanced architecture is required to cater to such hotspots. Though applications can avoid such cases by intelligently distributing their reading and writing to different parts of the file or different files.
```
## Consistency
Based on the assumption that most files are mutated by appending data to them and are read sequentially, GFS provides a relaxed consistency model that is optimized for append operations. We have seen that, for both sequential and concurrent append operations, GFS guarantees defined regions intermingled with inconsistent regions that can be identified and removed later. For random writes, applications need to serialize the conflicting operations in order to have the defined regions or to come up with some application-level strategy not to over-step each other's writes. The best strategy here will likely be to use GFS's append operation as much as possible and to avoid random writes.
```
Note: GFS's data consistency model is probably the hardest part to understand for end-programmers (especially those who are not familiar with the peculiarities of the distributed systems). It is advised to use the append operation as much as possible and to avoid random writing either altogether or to do so without overlapping regions.
```
GFS files might get many undefined and padding-filled regions in a large file. While such parts are automatically managed by the GFS-client code, they still take up space.

Usually, the proportion of undefined and padding-filled regions compared to the complete application data is small. Still, an application can recover some of the space lost to those unwanted regions, especially for large files that will stay in GFS for a long time. To avoid (or reduce) undefined and padding-filled regions in a large file, an application might use a single reader which scans the whole file from start to end and copies the data into a new temporary file. Because of the absence of any concurrent writers, the new file will not have undefined regions. After the successful copy, the original file can be deleted.

For metadata consistency, GFS synchronously replicates the checkpoints and the operation logs. Note that metadata changes are atomic.

## Conclusion
GFS supports workloads for handling huge amounts of data on cost-effective hardware. Because of the commodity hardware used by GFS, component failures are treated as a norm. GFS has a proper mechanism to deal with these failures. It does so by constant monitoring, data, and metadata replication, checksumming, and fast recovery of master and chunkservers. The separate data and control flow in the GFS design provides high aggregate throughput to the clients for reading and writing data. In addition to achieving some basic file system objectives, GFS is optimized for large files in which data is typically appended and read sequentially.

GFS is a triumph in system design because it is fairly simple and yet can scale well and can meet its other requirements. Let’s highlight some of the key design points.

### System design wisdom in GFS
We should never dismiss a design because it apparently has a single point of failure. The master of GFS is likely to have a single point of failure. GFS manages its shortcomings by carefully deciding the master's role and what can (and cannot) be done if it is unavailable. The master’s availability isn’t an issue in operations; however, its upper limit on scalability is an issue. (We will see in another chapter how Google managed that with GFS's descendent called Colossus.)

GFS is an example of system co-designing where GFS is optimized for a specific workload and specific kind of commodity hardware. That means GFS might not be a good choice for all the use cases, but it is a good choice for many Google applications.

Data consistency is hard when there are a lot of possible mutations, and different failures can happen. GFS uses a design pattern called “make the fast case common,” where we have old-styled rotation disks that can write fast and sequentially (as compared to random writes). GFS provides the fast append operation without the need for locking for the clients.

There is another important design principle called end-to-end arguments, which says that the core of the system should be kept simpler, and the ends ( in our case, GFS-client code) should optimize for specific cases. This principle is at play in a lot of places, for example, the architecture of the global Internet follows it where the core of the Internet is simple (and does not try to recover from reordered, or undelivered packets) and lets the end clients manage it (for example, via TCP protocol).
```
https://web.mit.edu/Saltzer/www/publications/endtoend/endtoend.pdf
```

The list above is far from exhaustive, and there are more points to take away from this design. This is the hallmark of a seminal system's work–even if you reread it after a while, you get new insights.
