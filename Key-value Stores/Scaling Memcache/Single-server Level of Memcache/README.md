# Single-server Level of Memcache

## Introduction to the single server level
We'll improve certain aspects of Memcached by modifying its internal mechanisms for a single server. Memcached uses four main data structures to form a key-value store:

1. A Hash table

2. A Cache item data structure

3. A Slab allocator

4. A Least Recently Used (LRU) list

### Hashtable with separate chaining
The hashtable uses hash functions to look up items using the key quickly. As illustrated below, the hashtable is an array of buckets, where each bucket is a linked list.

[Hashtable with separate chaining]

### Memcached items
A Memcached item is an object that holds data for the key-value pair. This item is what we store in a single chunk. The item stores metadata like the max key length, time till expiration and more.


### Slab allocator
The slab allocator is a memory management mechanism used to decrease memory fragmentation.


[Using the slab allocator in Memcached]

#### Slabs
Slabs are lists of memory sections containing objects (pages) categorized by the memory size range they fall into. By default, the memory is split into 42 slabs. The first slab is for items less than 96 bytes, the second is for items from 96 to 120 bytes, and so on until the 42nd slab. The motivation to use these slabs is to reduce memory fragmentation as items of similar sizes are in the same slab, which leads to more efficient memory usage.


#### Pages
Slabs contain multiple pages, where each page is the size of 1MB. Pages contain multiple chunks, where each chunk takes up the maximum size for that slab.


#### Chunks
A chunk contains data of size less than the slab's maximum size range. Chunks contain the item for a specific key in memory.

### The least recently used (LRU) list
Memcached uses an LRU list to evict the oldest and least used items. Whenever an item is requested from the LRU list, the item is removed from its position and added at the head of the list.

[A LRU Least Recently Used list–a doubly linked list]

## Requirements
The following are the server-level requirements:

1. Memory efficiency: We want to minimize the wastage of memory when storing items.

2. Flexible design: Have optimal performance for varying workloads.

## Overview of design problems
At the single-server level, we need to understand and improve the limitations faced by Memcached. At this level, we need to analyze the workloads and the inefficiencies caused by those workloads. For example, we can recognize items with high eviction rates or certain item sizes being a majority. Being flexible to different types of workloads allows us to leverage our systems to handle them more efficiently.


## Detailed design
When a frontend server receives a request, it can contact many Memcached servers to fulfill the request. Thus, in a short amount of time, all frontend servers communicate with every Memcached server. This all-to-all pattern means a single server can be a bottleneck for a whole cluster.

### General performance optimizations
Fine-grained locks have higher hit rates while also having a better performance for misses. Using fine-grained locks triples the peak get performance from 600k to 1.8M items per second while almost improving 1.5 times the performance of those replies, which results in misses from 2.7M to 4.5M items per second (meaning we can now process the missed requests faster). As misses require a single static response for signaling an end, they have a higher performance because they are less expensive than hits. Hits, on the other hand, have to be constructed and then transmitted.

Moving on to our choice of protocols for Memcached, UDP and TCP have different advantages and disadvantages. To test the effect of different protocols, we can develop Memcached using both. According to this study the tests show that the UDP implementation of Memcached is 13% faster for single get operations and 10% for multi-get operations than the TCP version.

```
Source: Rajesh Nishtala, Hans Fugal, Steven Grimm, Marc Kwiatkowski, Herman Lee, Harry C. Li, Ryan McElroy, Mike Paleczny, Daniel Peek, Paul Saab, David Stafford, Tony Tung, and Venkateshwaran Venkataramani. 2013. Scaling Memcache at Facebook. In Proceedings of the 10th USENIX conference on Networked Systems Design and Implementation (nsdi’13). USENIX Association, USA, 385–398.
```

#### More minor changes
Assign a UDP port to each thread to reduce contention when sending responses. The hashtable can slowly drift towards the O(n) time complexity due to its reliance on handling collisions using separate chaining. We can automatically expand the hash table when we start approaching O(n) loop-up times to avoid this problem.
We can make the Memcached server multi-threaded using a global lock to take advantage of the multi-threaded architecture.
Assign a UDP port to each thread to reduce contention when sending responses.

### Adaptive slab allocator
When we have an application with a different workload, the original (fixed) memory allocated to each slab class may not be optimal, yielding poorer hit rates. To solve this, we need to periodically rebalance slab assignments using an adaptive slab allocator.

A slab needs more memory:

1. If it is evicting a lot of items.

2. If an item that is evicted was being used 20% more recently than the average of the least recently used item of other slab classes.

Once a needy slab has been identified, we can free up the least recently used slab and transfer it to the needy class.


### The transient item cache–increasing memory efficiency

[buffer]

Items that receive a burst of usage are not memory efficient because they will remain in the cache until they reach the end of the LRU list. To solve this, we can proactively remove short-lived keys. We can place short-lived items in a circular buffer of linked-list based buckets. The buffer is indexed by the seconds till expiration. Every second, the bucket at the head gets evicted.

According to a study, using this technique can reduce the percentage of these items from 6% to 0.3% without affecting the hit rate.

```
Rajesh Nishtala, Hans Fugal, Steven Grimm, Marc Kwiatkowski, Herman Lee, Harry C. Li, Ryan McElroy, Mike Paleczny, Daniel Peek, Paul Saab, David Stafford, Tony Tung, and Venkateshwaran Venkataramani. 2013. Scaling Memcache at Facebook. In Proceedings of the 10th USENIX conference on Networked Systems Design and Implementation (nsdi’13). USENIX Association, USA, 385–398.
```
### Software upgrades

When deployed, the software on our Memcached servers can require upgrades, performance testing, or bug fixes. Memcached servers require a long time to upgrade because the load on the database can become too much if not managed appropriately. We need to route keys slowly to other servers and then take them offline for an upgrade.

To avoid this, we can store critical data structures on shared memory regions so that data can stay in memory while the processes accessing the data can be changed or upgraded. This reduces the need to re-cache all the data from the database.

These single server strategies allow us to avoid server bottlenecks, which contribute to the performance of our distributed key-value store.

```
A shared memory function allows multiple processes to access the same region of memory.
```
