# Detailed Design of GFS
## Assumptions
GFS was designed to match the needs of Google’s own applications. It is based on the following observations and assumptions of its applications:

- The system is made up of low-cost commodity machines that frequently fail. Our system must continuously monitor itself to detect, tolerate, and recover from frequent component failures.

- Multi-GB files are more common, so the system must be optimized to manage huge files more efficiently than small files. There would be a few million files in total, each of which is 100 MBs or larger.

- Large streaming reads and sequential writes are the more common cases for targeted applications. A large streaming read reads one MB or more; this is also the case for sequential writes. The system should be efficient in managing these operations. Small random reads (a few KBs) and writes should also be supported, although the system doesn’t need to be optimized for these operations.

- Hundreds of clients can concurrently append data to the same file. The system should provide well-defined semantics for concurrent appends.

- The majority of target applications necessitate bulk data processing at a high rate, which requires high throughput rather than low latency. A few applications, however, have low latency requirements for each read and write operation.

## Design pattern
In the first lesson, we discussed the GFS architecture at a high level with the following illustration. Let's discuss it in detail considering the assumptions above, and we will see why the GFS was designed in this way.


In the first lesson, we discussed the GFS architecture at a high level with the following illustration. Let's discuss it in detail considering the assumptions above, and we will see why the GFS was designed in this way.

[Architecture of GFS](./architecture.jpg)

A single storage server, even with a large amount of storage space, wouldn't work to cope with the following requirements:

- Storing millions of files read and written by hundreds of clients simultaneously

- High throughput for applications processing huge datasets

We have already seen the limitations of single server-based systems in the first lesson. As per our assumption, we have to build our system with commodity machines, so we can't set up storage attached networks (SANs) that require specialized hardware and are very expensive. In GFS, we are using a large number of commodity machines to share the request and the data load. Each machine runs a Linux file system to manage the storage attached to it.

To manage large files efficiently, the files are split into multiple chunks of the same size. Each chunk is 64 MB in size and is identified with a unique ID called a chunk handle. The machines that store these chunks are called chunkservers. A single file's chunks are distributed and replicated among multiple chunkservers. GFS stores three replicas for each chunk by default for reliability. All of the chunkservers that hold chunks of different files need a manager. The manager will track which chunk belongs to which file and is placed on which chunk server. GFS calls this manager a master.

A GFS cluster comprises many chunkservers and a single master, as shown in the illustration below. The master stores namespaces, access control information, file-to-chunk mapping, and current chunk placement. Other system-wide tasks, such as choosing the primary replica to perform write operations, garbage collection for unreferenced chunks, and migrating chunks among chunkservers, are also managed by the master.

[Single master managing multiple chunk servers](./master.jpg)


The master regularly needs to communicate with the chunkservers to determine the chunks they store, the storage space they are left with to manage chunk placement, and whether they are operational. All this communication between the master and the chunkservers is done through heartbeat messages.
```
Note: Heartbeats are periodic messages that one server can send to the other to determine its state, such as server liveness.
```
Each application's GFS client code implements the GFS API, through which it reads and writes data by communicating with the master and the chunkservers. GFS separates the data and the control flow. For metadata operations, clients communicate with the master, whereas all data-related communication takes place directly with chunkservers. If data is to pass through the single master, we wouldn't be able to serve huge amounts of data against many client requests at once due to the limited resources of a single server. Separating data and the control flow is crucial for scalability as it allows us to distribute data requests among multiple chunkservers.

The master and all the chunkservers are running on commodity machines that can fail at any time, so we need a monitoring system that can monitor the failures and take the required action. GFS uses a monitoring system that runs outside the cluster to monitor the machines inside the cluster.
## Design choices
Let’s investigate the rationale for the following GFS design choices and identify the benefits.


#### Single master
The reason for having a single master is to simplify the management tasks such as chunk placement and replication. The single master can make better placement and replication decisions for chunks based on a global understanding of the states of all chunkservers. However, we need to minimize master involvement in the client operations so that the master doesn’t become a bottleneck.




The separate control and data flow lessens the burden on the master. Clients interact with the master for the chunk locations and then contact that chunk server to read or write data. The chunk locations are also cached at the client side for some time, which enables the client to perform subsequent data requests on that chunk without interacting with the master.

#### Large chunk size (64 MB)
Most file systems have block sizes of a few kilobytes. The chunk size in GFS is substantially greater. The large chunk size is favorable for most of Google's application workloads that read/write large files sequentially. One reason for this design choice is to lessen master involvement. To perform multiple subsequent operations on the same chunk, the client only needs to interact with the master once.

Since the clients are most likely to perform sequential reads/writes on the same chunk, the large chunk size helps reduce the network overhead by maintaining a persistent connection with the chunkserver until we are done with reading/writing that chunk completely or the connection duration expires.

There is some metadata per chunk, so we keep the cost of that metadata in check by making the chunk large enough. It will help reduce metadata the single master has to manage and store.

The large chunk size has its benefits in our case, where we assume that most of the files are very large. If files are smaller than the chunk size, it causes internal fragmentation. For example, a file of size one MB will consist of only one chunk containing one MB of data, and the rest of the 63 MB of space allocated to that chunk will be wasted.
```
Question
How to avoid a waste of space due to internal fragmentation?

Answer
The large chunk size is not good for small files consisting of a few chunks or just one chunk, even with lazy space allocation. This is because the chunkserver holding a chunk of that file will become a hotspot if many clients access that file, which is not the case for large files where the requests are distributed to multiple chunks. However, the small files are less often read or written by our targeted applications. But to support small files, we increase the replication factor of small files. Another solution to this problem is to have a client read data from the peer client instead of the chunk server.
```
We might wonder why the chunk size is 64 MB and not more than this. The very small chunk size, a few KBs, leads to an increase in the metadata that the single master has to manage (because some of the master's metadata is per chunk). Whereas the very big chunk size, like 512 MB, results in fewer file splits (chunks), limiting the load distribution and hence the ability to serve many client requests for the same file. The 64 MB chunk size was suitable for the workload for which GFS was designed. However, the later file systems based on GFS, such as Hadoop Distributed File System (HDFS), changed the chunk size to 128 MB according to their needs.

#### In-memory data structure for metadata
The amount of metadata reduction with a large chunk size as our design choice helps us maintain the metadata in the master’s memory. The metadata that the master stores include namespaces, access control information, file-to-chunk mapping, and chunk locations. The in-memory data structure for the metadata helps the master perform metadata operations quickly.




The in-memory data structure also helps the master perform periodic background operations such as garbage collection, re-replication, and relocating the chunk to balance the load and disk space usage across chunkservers which would otherwise require repeatedly loading the metadata from the disk.

Despite the benefits of in-memory metadata storage, this approach may be a bottleneck on the master's memory because of the increasing number of chunks, and thus the metadata. This may be a problem for large file systems with time. The scale for which the GFS system was built didn't encounter any practical limitations with this approach. The metadata size per file and chunk was very small; less than 64 bytes are required to store a chunk's metadata and the same for the file name and other file-specific information. The chunk metadata includes chunk ID, chunk's current version, and chunk's replica locations while the file metadata includes the file name (stored in a prefix-compressed form), file ownership and permission information, and file-to-chunk mapping. A rough estimation of bytes required to store metadata per chunk is shown in the following table.

```
                               Per Chunk Metadata Size Estimation
Chunk ID                  Chunk version           Chunkserver's IP address x No. of replicas         Timestamp to carry out garbage collection, etc.        Total bytes

64 bit = 8 bytes            8 bytes                   32 bit x 3 = 12 bytes                                       13 bytes                                      41 bytes
```
Additionally, the cost of adding extra RAM to the master is a small price to pay for the performance, reliability, flexibility, and simplicity achieved by keeping metadata in memory.

##### Operation log
The in-memory metadata might be lost in case of the master’s failure. A persistent metadata record is kept on the master’s hard disk by logging the metadata modifications in an operation log. The operation log is a persistent record of metadata and the logical timeline that specifies the order of concurrent operations. Each file, chunk, and their versions are uniquely identified by their logical creation times.




The master does not store the current location of chunks persistently. Rather it rebuilds that information on restart from the chunkservers via heartbeat messages.
```
Question
Why is the logical time required on a single master node?

Answer
The failure of the master, which leads us to create a new master, may be one of the reasons. The new master can have different time (had we used physical clocks). Let’s name the original failed master master_{old} and the newly initiated master the master_{new}. Assume that two concurrent operations were being performed on master_{old}. The physical time at the master_{old} was t_{old}. The master_{old} logged one operation to the operation log with timestamp t_{old}, but it failed before logging the other operation. The master_{new} takes the place of the master_{old} and logs the left operation in the operation log with timestamp t_{new}, the physical time at master_{new}. We are not sure that the t_{new} is greater than t_{old}. If t_{new} is less than t_{old}, the operation logged later will have a timestamp prior to the operation logged earlier. The operation logged earlier should have a timestamp prior to the one logged later. The different physical times on different instances of the master node, though one active at a time, produce incorrect order for the operations. We need a logical time to have the same understanding of the time among different machines. For logical clocks, often the cluster manager provides the servers with a reliable mechanism to get them. Lamport or Vector clocks are the examples of logical clocks.

Also, we have shadow masters that replicate the operations in their log files. So, we should have a global view of time for all the machines (that we can’t have with physical clocks due to skew and should rely on logical clocks).
```
#### To not store chunk locations persistently
If we maintain the chunk locations persistently on the master's hard disk, we must update this information regularly due to the failures of the chunkservers. However, in commodity servers, failures are a norm.

The choice to not keep chunk locations persistently helps keep the master and chunkservers in sync because chunkservers fail, so they leave and rejoin the cluster, change names, and more. The chunkserver has the right information about what chunks it holds and what not. The master keeps itself up to date by asking the chunkservers about the chunks they hold. This is done via regular heartbeat messages between the master and the chunkservers.

```
Question
After a master’s failure, a new master is initiated that builds its state by replaying the operation log replicated on a remote machine and collecting the list of chunks from all chunkservers. What happens if the location of the chunk present on a chunkserver is unknown to the new master, and the client requests that data?

Answer
A new master does not start serving clients until it feels that it has reasonably rebuilt the state. It is possible that a client asks for a chunk whose chunk location is yet missing at master (but that will happen when all three replicas of that chunk are unavailable and haven’t shared the list of chunks with the master, and that is a low-probability event). At that time the master will return some failure to the client until that chunk is available. If it is a permanent loss, admin/human intervention is needed. However, the client might read other available chunks meanwhile.

```

In this lesson, we discussed the GFS design in detail. Next, we’ll see how each file operation is performed with this GFS architecture. The next few lessons will cover file operations, and then we will discuss the consistency model of GFS. At the end of the chapter, we will evaluate the GFS design.
