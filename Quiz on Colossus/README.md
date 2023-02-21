# Quiz on Colossus

**Question 1**
What is the difference between GFS and Colossus?

Answer
- A GFS cluster is scalable to hundreds of terabytes of data, while a Colossus cluster is scalable to exabytes.
- GFS is not good for storing small files, while Colossus’s design makes it suitable for small files as well.
- GFS was designed for batch-oriented workload; thus, it provides high throughput, but latency could be large in GFS. Colossus design supports high read/write throughput with minimal latency.



**Question 2**
What makes Colossus scalable to exabytes of data?

Answer
The control plane consisting of horizontally scalable metadata servers (called curators) and a scalable metadata database makes Colossus scalable to exabytes of data.


**Question 3**
Bigtable says that it uses Colossus to store its data persistently. On the other hand, Colossus uses Bigtable to achieve scalability. Is this codependence logical?

Answer
Well, it seems so. Colossus documentation doesn’t have a clear explanation of it. For most codependence instances, we often bootstrap the process by special means. We have speculatively described a possible solution in the following. Though we encourage you to think of alternative ways

Solution sketch: Bigtable uses a hierarchy to locate data shards. The first level is stored in another service called Chubby to bootstrap that lookup process. Usually, whenever someone writes into a Colossus, the associated metadata goes to a Bigtable instance. But when this specific Bigtable instance puts its data into Colossus, it especially puts the location metadata using a lean hierarchy (probably just one level) of shards. The main Colossus metadata can use many instances of such Bigtable to scale.

Details:

Let’s re-examine the progression from GFS to Colossus.

The GFS logically represents metadata and namespaces as a lookup table. The lookup table maps full pathnames to the metadata. With an increased volume of data requirements and small files, the metadata size became huge.

A single master is not able to store such a large lookup table.
Searching from such a large table takes time and adds to the latency.
We can’t serve an increasing number of metadata requests simultaneously from the single master.
Colossus needed to use multiple masters instead of a single master to manage and serve metadata with low latency and high throughput to many clients. So, given that multiple masters (metadata nodes) are present, Colossus needs some partitioning logic to split the large metadata table across various masters. (They might be using some variant of consistent hashing, and the client can locate which master to contact). Let’s call the group of masters (metadata nodes) a metadata cluster. As the nodes/masters leave or join the cluster, we need to rebalance the load among the nodes in the cluster. Rebalancing includes splitting or merging the metadata tables.

To have this partitioning logic for a large lookup table, splitting or merging tables to balance the load among multiple masters inside Colossus, Google exploits Bigtable, which implements all this. Maybe Colossus uses Bigtable as a black box, or maybe it uses one that is specialized for Colossus, but the central idea is the same.

[Architecture](./arch.jpg)

In the illustration above, we call the nodes in the Bigtable cluster masters. Each master has pointers for a set of metadata table partitions. For the sake of simplicity, we have shown two partitions only, one is called tablet 2, and the other 24. Master 1 has pointer 2, which means it will serve the requests for tablet 2, and the master 2 is responsible for serving requests for tablet 24. Each master is searching from a small table so that the search is quick, and metadata operations can be performed quickly.
The masters in the metadata cluster (or the nodes in the Bigtable cluster) store tablets on the Colossus file system. The actual metadata on the tablets is stored in the same storage pool where Colossus stores data. The metadata for the tablet locations can again be stored in the metadata cluster (the Bigtable). Alternatively, the masters store the location of the tablets along with the pointer to the tablets.
