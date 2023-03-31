# Introduction to Scaling Memcache
## Key-value stores and big data
When designing large-scale social media websites, we need to understand that there are huge volumes of data and that a number of requests are being made. The large scale of data on such platforms entails very intense load and spontaneous I/O requests, which might not be optimized using prefetching.

```
Prefetching loads data into the cache before the user requests it.
```

This is where caching in memory can be used to allow for linear scaling. A key-value store is one way to cache data. Key-value stores can run on network-enabled systems that share information with other machines without requiring specialized hardware. This allows architects to scale their key-value stores by adding more front-end servers. One such key-value store is Memcached.
```
Note: We'll call the distributed key-value store as Memcache and the source code or the running binary as Memcached.
```

## Why scale Memcache?
Memcache allows developers to add caching to their applications while having simple operations like set, get, and delete. Although it comes with a lot of features for single-server use, there are major hurdles in scaling it to a global level since it doesn't have built-in features (like replication, or clustering) that allow it to scale out for large-scale applications.

Scaling Memcache requires us to solve novel problems like:

1. Managing hot spots that arise due to highly popular content.

2. Batching requests to reduce network load.

3. Managing stale sets and thundering herds to reduce load.


## Requirements
To keep up with the cache demands of a large-scale service, we need to consider the following requirements.


### Functional requirements
In key-value stores, we have the usual requirements for the set, get and delete functions. Additionally, we have the following needs:

1. Accessing and updating highly viewed content: Sometimes, many clients start to access the same key-value pairs in Memcached servers. This can bring an enormous load on the Memcached servers themselves, and can also cause too many cache misses, which can in turn may bring the load to the storage layer. Our system should be able to manage such workloads efficiently.

2. Handling different workloads: Different key-value pairs have different characteristics. Some have very large keys while others are very popular for just a few hours. We need efficient methods to manage these differences.

3. Maintaining hit rate: We want to maintain the hit rate while the service is live.

4. Reducing network load: At the cluster level, one web request can make hundreds of Memcache calls within a cluster, which could lead to an extensive load on the network. Our system should have an intelligent placement of Memcache components and good request routing to appropriate components.

5. Insulating storage layer: Too many cache misses can overload the storage layer, so we want to reduce the frequency of the key-value store fetches to the database.


### Non-functional requirements
1. Near real-time communication: Latency should be low throughout the system, but we'll have to make a trade-off to maintain consistency.

2. Real-time feed generation from multiple sources: Front-end servers should be able to communicate with hundreds of Memcached servers quickly to respond to a request.

3. Memory efficient: The caching system from the cluster level to the cross-regional level should be memory efficient.

4. Replication: To cater to high-demand popular content, replication becomes necessary to process millions of user requests per second. Replication occurs on the regional and cross-regional levels.

5. Eventual consistency: All replica regions across the globe should eventually become consistent.

6. Read-after-write consistency: In a local context, clients ought to be able to update an item and then view the changes made to an item.

7. Handling failures: We need to safely handle a cluster failure to avoid a cascading failure of clusters.


## Problem statement
When we have multiple key-value (Memcached) servers throughout the world, how can we replicate data and maintain its consistency while balancing the tradeoffs (for example, between consistency and latency)?


[We have four levels where we can focus our attention to improve our system]

When providing service worldwide, considerations must be made on different levels as each level has different requirements. Because a component of latency is directly related to the distance (propagation delay), we make different tradeoffs as the distance between Memcache nodes changes.

The system is similar to an onion, where we want to ensure that each layer is healthy so that it can support the layer above it. For example, if there is a problem with the server-level implementation, all the layers above will be impacted, but at a much larger scale.

We can analyze and optimize at four different levels.

1. Single server

2. Cluster level (multiple servers)

3. Region level (multiple clusters, also known as a data center)

4. Cross Regional Level (multiple data centers)

The image below compares the architecture of a single-node Memcached system with the architecture used to scale Memcached into a distributed key-value store.


[Scaling Memcached to manage a worldwide network]

At different levels, we encounter different tradeoffs, as mentioned below.

```
               Differentiating the Layers
Levels                           Problems

Single server level              Single-server bottleneck
                                 Non-disruptive software upgrades
                                 Better memory efficiency

Cluster level                    Replication within frontend clusters to accommodate user traffic
                                 Reducing the number of network round trips
                                 Reducing incast congestion
                                 Replication within pools of clusters
                                 Handling stale sets and thundering herds
                                 Varying workloads
                                 Read-after-write consistency

Region level                     Sending invalidations to frontend clusters (where replication occurs) from storage clusters 
                                 Handling incast congestion due to too many Memcached servers
                                 Bringing new cold clusters online

Cross-regional level             Making replications and writes over vast distances, with eventual-consistency
```


```
                                          Different requirements of each level


                      Single Server Level                                    Cluster Level                                                                       Region Level                                                                              Cross Regional Level

Replication          No replication is required                        We require replication within cluster pools to efficiently serve read-heavy workload      We want replica clusters to route around failures to provide high availability            We want cross regional level replication to reduce latency caused by vast distances
     

Latency              Negligible latency                                We want to reduce latency as much as possible for highly popular data                    Synchronous replication within a data center is usually fast                               Asynchronous replication is one way to reduce latency

Consistency          Read-your-write consistency at the server level    Read-your-writes consistency                                                             Read-your-writes consistency                                                              We provide eventual consistency across data centers
```

## Bird's eye view of Scaling Memcache
In the next few lessons, we will dive into scaling the Memcache key-value store. The following concept map is a quick summary of this chapter.
