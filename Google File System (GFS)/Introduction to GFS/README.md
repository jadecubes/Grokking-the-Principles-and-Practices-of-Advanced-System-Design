# Introduction to GFS
## Problems with traditional file systems
Before Google File System (GFS), there were single-node file systems, network-attached storage (NAS) systems, and storage area networks (SAN). These file systems served well at the time they were designed. However, with the growing demands of data storage and processing needs of large distributed data-intensive applications, these systems had limitations, some of which have been discussed below.

```
Throughput measures how much data is being sent through your network at any given moment. It is frequently measured in bits per second. [source: https://networkshardware.com/throughput-vs-latency/]
```

```
Latency refers to the amount of time it takes for your data to be sent to its intended destination. It’s also referred to as the delay in moving data between two clients. It is usually measured in milliseconds (ms). The lower it is, the better. [source: https://networkshardware.com/throughput-vs-latency/]
```

A single-node file system is a system that runs on a single computer and manages the storage attached to it. A single server has limited resources like storage space, processing power, as well as I/O operations that can be performed on a storage disk per second. We can attach substantial storage capacity to a single server, increase the RAM, and upgrade the processor, but there are limits to this type of vertical scaling. A single server also has limitations regarding the number of data reads and writes, and how quickly data is stored and accessed. These limitations restrict the system's ability to process large datasets and serve a large number of clients simultaneously. We also, can't ignore the fact that a single-node system is a single point of failure where the system becomes unavailable to the users. The focus should be on high throughput rather than low latency for applications requiring large datasets processing.

The network-attached storage (NAS) system consists of a file-level storage server with multiple clients connected to it over a network running the network file system (NFS) protocol. Clients can store and access the files on this remote storage server like the local files. The NAS system has the same limitations as a single-node file system. Setting up and managing a NAS system is easy but expensive to scale. This system can also suffer from throughput issues while accessing large files from a single server.




[The traditional networked file systems](./traditional.jpg)

The storage area network (SAN) system consists of a cluster of commodity storage devices connected to each other, providing block-level data storage to the clients over the network. SAN systems are easy to scale by adding more storage devices. However, these systems are difficult to manage because of the complexity of the second network — the Fiber Channel (FC). To set up the Fiber Channel, we need dedicated host bus adapters (HBAs) to be deployed on each host server, switches, and specialized cabling. It is difficult to monitor where failure has occurred in this complex system. Data inconsistency issues among replicas may appear in this case. Rebalancing the load on the storage devices might also be difficult to handle with this architecture.

Note: SAN deployments are special-purpose networks apart from the usual Ethernet networks. This duplicate network, while good for segregating storage traffic, is expensive in terms of dollar cost.

Given the issues above with the traditional networked file systems, Google developed a file system called GFS that mitigates the posed limitations while also providing the benefits that these traditional systems possess. It supports the increasing workload of its applications and data processing needs on commodity hardware.



## GFS
Google File System (GFS) is a distributed file system that stores and processes large amounts of distributed data-intensive applications data on a storage cluster of commodity servers. The functional and non-function requirements of GFS are listed below.

### Functional requirements
Functional requirements include the following:

- Data storage: The system should allow users to store their data on GFS.
- Data retrieval: The system should be able to give data back to users when they request it.

### Non functional requirements
Non functional requirements include the following:

- Scalability: The system should be able to store an increasing amount of data (hundreds of terabytes and beyond), and handle a large number of clients concurrently.

- Availability: A file system is one of the main building blocks of many large systems used for data storage and retrieval. The unavailability of such a system disrupts the operation of the systems that rely on it. Therefore, the file system should be able to respond to the client’s requests all the time.

- Fault tolerance: The system’s availability and data integrity shouldn’t be compromised by component failures that are common in large clusters consisting of commodity hardware.

- Durability: Once the system has acknowledged to the user that its data has been stored, the data shouldn’t be lost unless the user deletes the data themselves.

- Easy operational management: The system should easily be able to store multi-GB files and beyond. It should be easy for the system to handle data replication, re-replication, garbage collection, taking snapshots, and other system-wide management activities. If some data becomes stale, there should be an easy mechanism to detect and fix it. The system should allow multiple independent tenants to use GFS for safe data storage and retrieval.

- Performance optimization: The focus should be on high throughput rather than low latency for applications that require processing for large datasets. Additionally, Google’s applications, for which the system was being built, most often append data to the files instead of overwriting the existing data. So, the system should be optimized for append operations. For example, a logging application appends log files with new log entries. Instead of overwriting existing copies of the crawled data within a file, a web crawler appends new web crawl data to a crawl file. All MapReduce outputs write a file from beginning to end by appending key/value pairs to the file(s).

- Relaxed consistency model: GFS does not need to comply with POSIX standards because of the unique characteristics of the use cases/applications that it targets to serve. A file system must implement a strong consistency model in order to be POSIX compatible. In POSIX, random write is one of the fundamental operations. In GFS, there are more append operations and very few random writes. That’s why GFS doesn’t comply with POSIX and provides a relaxed consistency model that is optimized for append operations. Data consistency in a distributed setting is hard, and GFS carefully opts for a custom-consistency model for better scalability and performance.
```
The Portable Operating System Interface (POSIX) is a set of standards set out by the IEEE Computer Society to support compatibility between operating systems. It defines system and user-level application programming interfaces (API), command line shells, and utility interfaces for software portability among operating system variants. Both application and system developers are encouraged to use POSIX. [Wikipedia]
```

Let’s explore GFS’s architecture, which enables it to fulfill the requirements mentioned above.

## Architecture
A GFS cluster consists of two major types of components– a master node and a large number of chunkservers. It stores a file’s data in the form of chunks. The architecture is shown in the following illustration.

```
A chunk is a unit of data storage in GFS.
```


[Architecture of GFS](./gfs.jpg)


- The client is a GFS application program interface through which the end users perform the directory or the file operations.

- Each file is split into fixed-size chunks. The master assigns each chunk a 64-bit globally unique ID and assigns chunkservers where the chunk is stored and replicated. A master is like an administrator that manages the file system metadata, including namespaces, file-to-chunk mapping, and chunk location. The metadata is stored in the master's memory for good performance. For a persistent record of the metadata, the master logs the metadata changes in an operation log placed on the master's hard disk so that it can recover its state after the restart by replaying the operation log. Besides managing metadata, the master also handles the following tasks:

  - Data replication and rebalancing

  - Operational locks to ensure data consistency

  - Garbage collection of the deleted data

- Chunkservers are commodity storage servers that store chunks as plain Linux files.

The client requests the master node for metadata information, such as the location of the requested chunks. The master looks into its metadata and responds to the client with the location of the requested chunks. The client then asks the chunkservers for the data operations. It is important to note that the data doesn't flow through the master but directly between the client and the chunkserver.
```
Note: The largest GFS cluster can store up to tens of petabytes of data and can be accessed by hundreds of clients concurrently.
```


```
Question 1
How does a single master suffice to handle requests from hundreds of clients?

Answer
According to the architecture of GFS, the master appears to have a tremendous amount of work to do and this could act as a bottleneck. For the simplicity a single master offers, making it lighter weight is worthwhile instead of switching to a distributed master. GFS optimizes the single master by:

Minimizing master involvement in read/write operations. First, it separates the data and the control flow, so the data doesn’t need to pass through the master. The client has to interact with the master only for the metadata. Secondly, the chunk size is kept large to reduce the metadata requests on the master.
Keeping the metadata in memory and serving the clients from there.
Caching the metadata on the client side
```

```
Question 2
Does the in-memory approach for storing metadata put a limitation on the amount of data we can store? The more data, the more the metadata would be.

Answer
Practically, it’s not a big issue as the space required to store metadata per file is small. Less than 64 bytes are needed to store file-specific information such as the file name, and less than 64 bytes are required to keep each chunk’s metadata. Moreover, the cost of adding additional memory to a single master is negligible compared to the gains in performance and simplicity a single master architecture provides.
```

```
Question 3
How can we handle the single master failure?

Answer
To handle the master failure, we store the metadata changes (mutations) on persistent storage on the master’s disk as well as on a remote machine to recover in case of disk failures. This persistent record of metadata changes is known as the operation log. The failed master can recover its state by replaying the operation log placed on the master’s hard disk.

In case the original master faces a long-lasting hardware failure, we start a new instance of the master on another machine and redirect the clients to it.
```

```
Question 4
How can we reduce the amount of time required to replay the growing operation log to rebuild the master state?

Answer
Checkpoint the master state when the log grows beyond a certain size. Load the latest checkpoint and replay the logs that are recorded after the last saved checkpoint.
```


## Bird’s eye view
In the coming lessons, we will design and evaluate GFS. The following concept map is a quick summary of the problem GFS solves and its novelties.

