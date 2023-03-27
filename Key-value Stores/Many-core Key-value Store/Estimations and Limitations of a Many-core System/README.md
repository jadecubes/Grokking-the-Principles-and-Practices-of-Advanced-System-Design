# Estimations and Limitations of a Many-core System
In this lesson, we will estimate how using a key-value server with an efficient processor impacts the system. To do this, we will be comparing the power consumption of two systems:

1. TilePro64: This system will use a lower-clocked, power-efficient 64-core processor with the Tile 32-bit architecture and has an optimized version of Memcached for higher parallel data access.

2. Opteron: This system will use a higher-clocked, power-hungry, octa-core processor with an x86 64-bit architecture and has a standard multi-threaded version of Memcached.

Once we understand the relative benefits of the two systems, we can start figuring out the higher-level design.


## Estimations
Let's assume that Facebook receives 2 billion daily active users (DAU). Now, the worst case would be that all of the DAUs have sent requests simultaneously. So, we need a Memcached system that can handle this peak load of 2 billion requests per second.

Power_watt = Power / Transaction × Total Transactions
The following calculator widgets help us change different values to see their effect. The default values (for example, power per transaction per second (power/TPS)) are taken from this paper, M. Berezecki, E. Frachtenberg, M. Paleczny and K. Steele, "Many-core key-value store," 2011 International Green Computing Conference and Workshops, 2011, pp. 1-8, doi: 10.1109/IGCC.2011.6008565.

```
                           TilePro64 vs. Opteron: Power Consumption
      A                            B                                 C                          D                           
1  Architecture	       Transactions Per Second (TPS)	           Power / TPS              	Power (Watt)
2  TilePro64	                   2000000000                      	0.00002	                  40000 =B2 * C2
3   Opteron	                    2000000000                     	0.0001	                  200000=B3* C3
                                            
```                                            
Based on the calculations above, Opteron-based servers would require 160,000 more Watts to handle 2 billion requests.

Given that we know the capacity of a single node from the same paper, we can calculate how many nodes we will need at a minimum to manage the load.
```
Capacity: The maximum load (measured in transactions per second) a server can manage while the average response time is below one millisecond.

Paper: M. Berezecki, E. Frachtenberg, M. Paleczny and K. Steele, "Many-core key-value store," 2011 International Green Computing Conference and Workshops, 2011, pp. 1-8, doi: 10.1109/IGCC.2011.6008565.
```
So how many nodes would Facebook need to respond to this many requests?


```
                        TilePro64 vs. Opteron: Nodes Required
       A	                     B                            	   C	           D                        
1	Architecture	    Transactions Per Second (TPS)        	Capacity / Node   	Nodes
2	TilePro64	               2000000000                  	335000	            5971= ceil(B2 / C2)
3	Opteron	                2000000000	                  165000	            12122 = ceil(B3/C3)                        

```

So, Facebook will require roughly 5971 nodes while using the TilePro64's architecture compared to 12122 nodes using the Opteron's architecture.

In conclusion, when using the TilePro64, we'll need about 2x fewer nodes (less administrative costs) and less power (less electricity cost) to manage the same workload compared to Opteron.


## Design challenges
The calculations above tell us that Tilera is a better choice for our key-value store in terms of our goals (power efficiency, low node count, etc.). We now see three challenges that we must overcome in our design to run Memcached (a memory resident key-value store) on the Tile processor.


### Challenge 1: Memory limitation of the TilePro64
Since TilePro64 has a 32-bit virtual address space, a single process is limited to only 232 bytes (4GB). In contrast, a process from a 64-bit Opteron processor can theoretically access 264 bytes (16EB). We should note that the physical memory limit of the Opteron processor (that was used in our study) is 128 GB. Given that the TilePro64 has a physical memory limit of 64 GB, we need to modify Memcached to take advantage of this memory. To do this, we'll shift from the standard multiple-threaded version of Memcached to a new multiple-process version of Memcached. We can use multiple processes to access more than 4GB of memory in aggregate.
```
TilePro64 had a virtual address space of 32 bits while physical address space of 36 bits. We needed a process-based design (instead of thread-based) to overcome this limitation. This is an example of peculiar characteristics that a special-purpose or custom processor might have. Using different kinds of systems together in a system can pose challenges for a designer. Balancing the costs and benefits of such design is necessary to find an appropriate solution to the problem.
```


### Challenge 2: Limitations of a multi-threaded Memcached
A multi-threaded implementation of a key-value store means that locks are needed to avoid race conditions and to ensure data atomicity. We need to lock the item even while performing GET operations because if an item gets modified while being fetched, it can corrupt the data being sent. Excessive locking and lock contentions can substantially lower parallelism and reduce the system throughput. Global locks also slow down the removal of items from the Least Recently Used (LRU) buffers. Hence, even though Memcached is achieving task parallelism, no data parallelism occurs due to the implementation of locks, i.e., we are accessing only one data item at any given moment.
```
The version of Memcached that was used in this study is 1.2.3h. It is primarily from the era when multi-core processors were getting commonplace but most software was yet not optimized for them. However, using special switches between processor cores and many high-speed network interfaces was not the only challenge. Moreover, to take advantage of this special hardware, we need to tweak our key-value store software. As designers, we need to weigh the costs and benefits of using domain-specific hardware. Here, Tile can give us what we want (power efficiency, low absolute number of nodes, high throughput, low latency) but we need to aggressively tweak our key-value store to efficiently use what Tile system has to offer.
```

**Question**
What are the differences between data parallelism and task parallelism?
**Answer**
Data parallelism vs. task parallelism
Data parallelism can occur when many data items can be independently worked on at the same time.

Task parallelism can occur when tasks are created that can be executed independently of one another. For example, when each incoming client request to get/put a key is handled by a different thread.

[Comparison]




In the image below, we show a multi-threaded version of Memcached where every operation requires a global lock to safely access the shared hash table. Only a single thread can access the key-value store at any one time.

[Multi-threaded architecture for a standard version of Memcached]


```
Question
Redis is a well-known RAM-based store. It is single-threaded and very fast (> 100000 queries per second). Why?

Answer
Being single-threaded means that clients will have to wait for a slow request, but it also allows for avoiding unnecessary context switching and expensive synchronization between threads. Furthermore, normal commands like GET and SET already take a small amount of (constant) time to execute; adding multi-threading to this would only add complexity. It is unlikely that the CPU ever becomes a bottleneck for such workload. In most cases, performance is bounded by the network or memory storage and throughput, and newer versions of Redis actually use threads to execute slow I/O operations in the background.

To utilize multiple cores, one approach could be to shard the key space and use a different, independent instance of Redis for different shards and for parallelism (a solution that we will use for our Tile-based design as well).
```


### Challenge 3: Task allocation for many-core processors
When we have a lot of cores available, our challenge is to find out how many cores should be assigned to different functions (hash workers, workers for TCP and UDP processing, etc.). There is no one correct answer to this problem, and with the solution can change with changing workload or problem domain. In our design, we will empirically find a reasonable allocation of cores to tasks (that give us the maximum performance).

Next, we will tackle these challenges in our design.

