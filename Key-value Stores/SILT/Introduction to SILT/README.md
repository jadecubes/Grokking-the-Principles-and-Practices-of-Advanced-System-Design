# Introduction to SILT
## Optimized elements make a better system
A distributed key-value store uses multiple storage and service nodes connected to a high-speed network to serve its clients. Optimizing the constituent parts (the nodes) helps us make a better key-value store. A store that saves several keys per node while minimizing indexing overheads, as well as raw disk searches and reads is highly desirable.

In this lesson, we will explain the need to optimize the individual nodes and formalize our requirements and metrics to measure our progress in achieving a better key-value store.


## The importance of key-value stores
In recent years, applications' reliance on data usage has increased. Modern-day applications are data-intensive, requiring prompt and efficient access. Furthermore, most applications need to be scaled for millions of users; as a result, they need to support a significant amount of queries in real time.

Relational databases are used to store and retrieve data. However, many applications might not require all functionalities of an RDBMS. Key-value stores offer a simple alternative to traditional databases. These are fundamental building blocks for many real-world applications, such as e-commerce platforms (like Amazon), data deduplication on flash storage, web object caching, and many more. Improving the key-value stores indirectly improves many applications that use the key-value stores as a building block.

```
RDBMS: Relational Database Management System
```

The costs (for example, performance penalties regarding disk seeks and dollar cost) affiliated with data storage and retrieval are proportional to the designated design's use of memory, computation, and storage. For a better key-value store, we would like to minimize such costs. While the per-unit cost of these resources has decreased over time, application requirements have increased significantly. Modern applications require highly efficient solutions to keep costs at a minimum.

Key-value stores are distributed hash tables (DHTs) that map a key to a value in a short time. We can engineer key-value stores to meet our application's requirements with acceptable costs.

[A key-value store is a distributed hash table. Only three nodes are shown in the illustration above. In an actual system, there can be thousands]

Like the illustration above, for a key-value store, nodes in a distributed network (servers) contain values for disjoint keys. For example, the first key, 001, is mapped to a URL, the second key, 002, is mapped to an image, and so on.

## Design goals
There are multiple types of key-value stores. Some are in-memory only and are usually used for serving values quickly, others use persistent stores including rotating disks and flash disks. In this chapter, we’ll focus on persistent data primarily on flash disks. Additionally, we’ll use memory to speed up different aspects of our design.

Let’s categorize and declare the following design goals.

1. Efficient use of resources

    I. Minimize the use of memory (RAM) per key (for quick access, we want to keep key indices and at least some of the data in the RAM)

    II. Computation efficient indexing (bytes needed per key for indexing a key should be low)

2. High performance with low latency

    I. Quick lookups and writes

    II. Low read and write amplifications

It might be tempting to keep all the indices and data (keys and values) in RAM. However, the data volume is often many orders of magnitude bigger than the available memory. Additionally, DRAM-based memory is eight times more expensive (in dollar terms) and uses 25 times more power than flash-based storage. Therefore, we must use both RAM and disk storage to optimize the overall performance.

```
Note: Depending on the workload access patterns, data is often divided into hot (actively accessed), warm (less frequently accessed), and cold (rarely accessed). We can keep hot data in RAM while pushing warm and cold data to slower disks. A good key-value store should provide its customer with such customization to pick a trade-off between the performance and the dollar cost of resources.
```


## Solution sketch
### Reduced per-key memory consumption
### Fast lookup
### Low read amplification
### Write optimization
### Controllable write amplification
## Bird's eye view
