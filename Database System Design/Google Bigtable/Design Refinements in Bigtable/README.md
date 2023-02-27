# Design Refinements in Bigtable
To fulfill our users’ high performance, availability, and reliability requirements, the system described in the preceding lessons needs to be refined in several ways. This lesson goes into further depth about specific implementation components to emphasize these improvements.

## Locality groups
Another abstraction in our data model is the locality group, which is a partition of the columns. Recall that tables are partitioned horizontally where each shard is called a tablet. A locality group can be viewed as a vertical partition and what it does is combine the data in a specific set of columns (column families). Columns that are in different locality groups are stored separately from one another.

Regarding the implementation, different locality groups mean different files on GFS. Bigtable generates separate SSTables for each locality group, which gives clients the ability to control where their data is stored.

It also provides various other knobs, and a particular one that many clients have found helpful is that clients can state whether or not a locality group should be memory-mapped. That is, whether or not the server should serve the data directly out of memory rather than reading from the underlying Google File System.

[Group of columns forming a locality group](./locality.jpg)

Here’s a picture of what this means for the model. Bigtable takes the tablets and cuts them in the column dimension as well. It might, for example, store half the content in one locality group because we only access that relatively infrequently, and it might store the other half in another. We can say we will serve the right-hand locality group out of memory to make access much faster in the above illustration.

## Compression
To conserve space, clients could also opt to compress the SSTable for a locality group. Bigtable clients can select compression algorithms based on their application needs. When many versions of the same data are saved, the compression ratio improves even further. Each SSTable block is compressed individually.
```
There is a range of compression algorithms available. Thereis also a tradeoff between compression and decompression time (both are not necessarily equal always), and space reduction versus time to compress (usually for high compression ratios, we need to burn a lot of CPU cycles.)
```

## Caching
Tablet servers increase read performance by using two levels of caching.

- Scan cache: The scan cache is a higher-level cache that stores the key-value pairs given to the tablet server by the SSTable interface. The scan cache is particularly beneficial for applications that often read the same data.
- Block cache: To read blocks of SSTables for GFS, we can use a block cache, which is a relatively low-level cache for it. The block cache helps applications that read data that is in close proximity to the data they just read, such as sequential or random reads of various columns belonging to the same locality group within a regularly visited row.
```
The scan cache is for temporal locality, while the block cache helps with spatial locality.
```

[Scan vs. block cache](./vs.jpg)

## Bloom filters
Every read operation must read from all of the SSTables that comprise a tablet. If certain SSTables are not present in memory, the read operation may result in several disk accesses. Bigtable employs Bloom filters to decrease the number of disk accesses.

Bloom filters are developed for SSTables specifically for the locality groups. They aid in reducing disk seeks by anticipating if an SSTable potentially contains data corresponding to a specific row/column pair. Bloom filters consume little memory yet significantly enhance read performance.

```
A Bloom filter is a data structure designed to determine if an element exists in a set quickly and effectively.
```

## Commit log implementation
If we retained each tablet’s commit log in a distinct log file, a substantial number of files would be written simultaneously in GFS. These writes may result in a huge number of disk accesses to write to the numerous physical log files. Furthermore, keeping distinct log files for each tablet lowers the efficiency of group commit (that is, a batched commit operation) optimization because groups would be smaller if we had a distinct file for each tablet. To address these difficulties, we append mutations to a unified commit log for each tablet server, combining changes from several tablets within the same physical log file. This improves the write performance.

One downside of keeping a single log file is that it makes tablet recovery more challenging. When a tablet server fails, all of its tablets are transferred to other tablet servers, where each server generally takes a small number of the initial server’s tablets. The changes from the commit log that were recorded by the previous tablet server must be repeated by the new tablet server in order to restore a tablet’s state. However, as we are using a single log file for all tablets, the changes for all tablets will be mixed in that log file.

One possibility for every new tablet server is to scan the whole commit log file and enforce only the information needed for the tablets it is required to restore. But there is a drawback with this technique, i.e., the log file would be viewed by every server assigned to a new tablet from a failing tablet server. For example, if there are 100 servers allocated to a single tablet, the total number of times the log file would be viewed would be 100.

Bigtable eliminates duplicate log reads by first sorting commit log entries by the keys <table, row name, log sequence number>. All changes for a certain tablet are continuous in this sorted output and can thus be read rapidly with one disk seek followed by a sequential read. To parallelize this sorting procedure, Bigtable divides the log file into 64 MB chunks and sorts each segment in parallel on various tablet servers. This sorting procedure is controlled by the master and starts as soon as a tablet server signals that it needs to retrieve mutations from a commit log file.

```
Question
What are the reasons for performance issues while writing commit logs to GFS, and how do tablet servers cope with these latency spikes in GFS?

Answer
There can be several reasons for facing performance issues while writing commit logs to GFS, such as when a GFS server participating in the writes fails or when the network paths used to get to the specific group of three GFS servers are congested or excessively loaded.

Every tablet server runs two threads for log writing, each of which is writing to a distinct log file, to defend against GFS latency spikes. At any one moment, only one of the threads is functional. If any threads fail to function well (for example, network congestion), the writing is transferred to the other thread. Log entries include sequence numbers to aid with retrieval.

When we use different GFS files for logging, hopefully, the files will be allocated to different sets of chunk servers (underlying servers to hold part of file data) with the different current loads.
```

## Speeding up tablet recovery
As previously stated, one of the most challenging and time-consuming jobs while loading tablets is ensuring that the tablet server gets all entries from the commit log. Once the master transfers a tablet from one tablet server to another, the source tablet server performs minor compactions on the commit log so that the target tablet server does not need to read it. Here are the steps in which a tablet is transferred from one tablet server to the other.

- The origin tablet server conducts minor compaction in the first phase. This compaction shortens the time to recover by decreasing the amount of uncompacted state in the tablet server’s commit log. When this compaction is complete, the tablet server terminates serving the tablet.
- Then, the source server conducts another (typically extremely rapid) minor compaction to reflect any new log entries that came while the first minor compaction was being executed.
- After the second minor compaction is finished, the tablet can be loaded onto another tablet server without the requirement for recovery.

[Speed up](./speedup)
