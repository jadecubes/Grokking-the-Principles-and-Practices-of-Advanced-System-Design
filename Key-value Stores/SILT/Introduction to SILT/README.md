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

[A key-value store is a distributed hash table. Only three nodes are shown in the illustration above. In an actual system, there can be thousands](./brief.png)

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
DRAM: Dynamic Random Access Memory—Volatile memory that stores bits of data in transistors, loses its state on turning off.
flash: Non-volatile memory that has high-performance storage
```


```
Note: Depending on the workload access patterns, data is often divided into hot (actively accessed), warm (less frequently accessed), and cold (rarely accessed). We can keep hot data in RAM while pushing warm and cold data to slower disks. A good key-value store should provide its customer with such customization to pick a trade-off between the performance and the dollar cost of resources.
```

```
Component	            GB/USD                      	Percentage Increase
DDR SDRAM	          0.20 (2012) to 0.37 (2022)	       17%
Portable Flash	      1.42 (2012) to 4.67 (2017)	       325%
Flash-based SSD       1.39 (2013) to 13.70 (2022)	       1231%
Disk	              16.67 (2012) to 62.50 (2022)	        275%
```


```
Source: https://jcmit.net/memoryprice.htm
Source: https://jcmit.net/flashprice.htm
Source: https://jcmit.net/diskprice.htm
```


The table above shows that the per-dollar capacity for both flash-based and disk storage has increased more significantly than the per-dollar amount of DRAM memory. Therefore, to design our key-value store, we’ll focus on reducing memory consumption more than the storage consumption. This means optimizing RAM use gives us more pay-offs than storage.

We’ve already declared that resources are costly, hence, our key-value store should be geared to use minimum resources. We’ll focus on two important aspects of key-value stores:

- They store a set of key-value entries.

- They are hash tables, which means they require indexing.

The key indices in the key-value entries are where we can focus on reducing memory consumption since we’ll store the values in storage. We can focus on reducing the per-key impact, significantly reducing overall memory use.

Nowadays, data centers are adopting alternatives to disk storage that support a much higher number of input and output operations per second (IOPS), such as flash-based SSDs, which is capable of hundreds of thousands of IOPS. Our key-value store will require efficient indexing to utilize I/O capability and avoid bottlenecks.

We have also discussed that our key-value store needs to meet the requirements of modern data-intensive applications with scalability—it should have the ability to scale to billions of keys with acceptable costs. While the first design goal somewhat covers scalability (by reducing the amount of RAM for indices and such memory can be used for actual data storage), we will ensure that our key-value store offers more scalability.

Since the main problem we are trying to solve is data storage and retrieval, we can focus on read and write operations. Read operations involve looking up values in data. Ideally, we would want these to be as fast as possible. One metric to gauge the efficiency of read and write operations is amplification. We will use the terms read amplification and write amplification.

Read amplification means when a single read operation requires multiple storage reads or seeks. We can measure it using the following formula.
```
ReadAmplification=  data read from storage / data read by application
```
 
Write amplification is similar to read amplification. It occurs when the size of total data written to storage is more than the size of data in the write operation. We can measure it using the following formula.

```
WriteAmplification = data written to storage /  data written by application 
```
 
Here, the application initiates the request (read or write). The application only sees the result of a read request. It doesn’t know the total volume of data read from the storage to get this result. Similarly, the application is only aware of the size of data it has requested to write and not the total size of the writes that will ensue to complete its request.

We’ll aim for a near 1 read amplification. We will see later that our design cannot have write amplification lower than 2. This is because we are trading this higher write amplification for lazy writes and deletes in our pursuit of efficiency.

[For key-value stores, reliability, availability, and fault tolerance are assumed as goals. We will call these general goals. However, our main focus is efficiency, speed, and scalability.](./req.png)

## Solution sketch
While our design objectives give us a good idea about the characteristics of the solution we seek, we need to be more specific when designing a system. In this section, we will list specific requirements of our key-value store.


### Reduced per-key memory consumption
Memory is expensive, and we can minimize its use by ensuring that our key-value indices take up less space in memory. We will employ memory-efficient data structures to achieve this.

### Fast lookup
Lookups need to be prompt to achieve scaling for many users. We will maintain dedicated data structures in storage with a sorted key order for lookups that point directly to the value when searching our key.

### Low read amplification
The fast lookup requirement is useful in keeping read amplification low. We will go further by keeping most of our data in data structures optimized for read operations.

We’ll use more than one data structure–each prepped for a specific purpose. Furthermore, we will use hash filters to probabilistically determine if a key is stored before allowing a read request to memory, thus reducing the number of memory reads. We will aim for a near 1 read amplification.

[Diagram](./amp.png)

The diagram above shows the design space. Our desirable solution should have low memory overhead and should need a low number of reads in storage per lookup.

### Write optimization
We will use a write-optimized data structure for write operations. Over time, our solution will move data to memory-efficient data structures.

### Controllable write amplification
Using multiple data structures means we will have to write a new entry more than once, which may result in higher write amplification. However, we will seek to have some control over our write amplification.

We will use an intermediary data structure to move our new entries and delete requests from our write-friendly data structure and move them to our memory-efficient data structure. This is to move our data in a way that helps our solution meet our design goals.

## Bird's eye view
The following concept map summarizes our work in this chapter. In the next lesson, we will start designing our system that we call the SILT (small index large table) system.

[Overview](./birdsview.png)
