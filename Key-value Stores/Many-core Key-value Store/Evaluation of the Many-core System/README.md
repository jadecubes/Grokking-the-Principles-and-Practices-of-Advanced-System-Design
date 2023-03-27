# Evaluation of the Many-core System

## Power efficiency
We wanted power efficiency to be able to decrease the cost of running a key-value store while increasing its throughput. We also wanted to be more power efficient than the key-value stores running on the x86 architecture processors.

With the TilePro64, our system can handle roughly three times the transactions per second per watt compared to x86 processors because of the higher throughput and lower power consumption. To make a more power-efficient system, we've increased its throughput and decreased its power consumption using many but slower and hence less power-hungry cores.

### Higher throughput
To increase the power efficiency, we had to improve the performance of our system. The performance metric we chose was the system's throughput (and latency, which we will discuss a bit later in the lesson).

We used a processor with a higher number of low-powered cores to achieve higher throughput despite having slower cores. Then, we used an optimized version of Memcached to take advantage of the higher number of cores, which reduced the reliance on locks. We then statically assigned tasks to cores to attain the highest throughput and gave a 67% higher throughput than a lower core x86 system.

### Less power consumption
While increasing the performance, we also had to decrease the power consumption of our system for a net increase in power efficiency.

We were able to achieve this since TilePro64 uses a shorter instruction pipeline depth to reduce power usage. (Recall that longer instruction pipelines are power-hungry, and more work is wasted in case of missed branch predictions when the pipeline might need to be flushed.) The x86 processors completed individual GET requests roughly 20% faster. However, they still consumed a lot of energy, and since they had a centralized table, that had a lower saturation point and significantly reduced throughput.

```
Centralized table: A hashtable which is not set up in a distributed manner.
Saturation point: The amount of traffic before a network reaches a latency wall.
```

## Latency
We wanted to maintain the latency of less than 1 millisecond. The x86 processors were approximately 20% faster, but TilePro64 can complete the GET requests under the 1-millisecond threshold.

TilePro64 also uses an 8x8 mesh network to allow for direct reading and writing of the packets. Due to the multi-process design, we could also stop the usage of global locks, which further reduces the latency of our system. So even with a significantly lower clock rate, the processor could still keep up with the performance of the x86 processors.

```
                Meeting Functional Requirements
Requirement                                 Techniques           

Data parallelization                   We split the key-value store into multiple shards for parallel access.
                                       Uses independent processes to hold data shards.
                                       
Core–task mapping                      We identified trends by systematically testing different configurations.
                                       Using those trends, we were able to finalize a configuration that maximizes throughput.
```

## Distinct features
This system design problem provides two distinct features:

1. Using a domain-specific processor, TilePro64 achieves excellent power efficiency by giving us more parallel processing to work with while sacrificing individual core performance.

2. Modifying the Memcached key-value store to take advantage of the new processor by removing its reliance on global locks.

## Design limitations
1. We assumed that read and write workload would be reasonably distributed over the available shards. If that is not the case, and a single shard becomes a hotspot, the performance of our design will suffer substantially. We might try to make the shard size smaller (and use more cores) to reduce the contention. Dealing with hotspots often requires insights into the workload, and generic solution can be less effective.

2. The paper this chapter was based on didn't optimize Memcached for x86 multicores. It would have been a fairer comparison with a Tile-based system. The paper specifies that the key-value workload will be bottlenecked on the network stack of x86 (not on cores). The core-optimized Memcached of x86 cannot compete with a Tile-based system that has a special purpose network mechanism for delivering data quickly.

3. The paper only concentrates on median latency, while for many workloads, the 95th and 99th percentile latencies are critical because the key-value store can be a foundational service for others. Median latency guarantees are usually too weak for such scenarios.

## Conclusion
This chapter examined how to use and test a domain-specific processor to solve a complex problem (i.e., optimizing key-value store reading and writing). Using a domain-specific processor means, we will face unique issues; for example, we had to circumvent the 4GB virtual address space limitation of TilePro64 to implement our key-value store. Domain-specific processors are becoming a norm for solving complex machine learning, power efficiency, and artificial intelligence problems. Using TilePro64 and a modified version of Memcached, we increased the throughput of our key-value store by 67% and transactions per watt by 4X as compared to x86 systems.

This design activity teaches us that while on the one hand, domain-specific systems are getting commonplace and can potentially provide better performance per watt (or performance per dollar), optimizing software for such systems needs careful thinking. That might not be the case with general-purpose systems, where one software might work well for many systems and workloads. As a designer, it is our job to evaluate when to use a general-purpose system, a domain-specific system, or both together.
