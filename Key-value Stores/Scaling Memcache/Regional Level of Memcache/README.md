# Regional Level of Memcache
## Introduction to the regional level
We call a data center a region, and a region is a collection of multiple clusters. At the cluster level, the dominant concern is sharding the key space (using consistent hashing) and grouping the keys into appropriate buckets (for example, viral keys vs. dormant keys and high-churn keys vs. low-churn keys). At the regional level, our main concern will be the replication of keys to meet the overall load.

Consistency concerns come with replication. At the regional level, we must maintain consistency between Memcached and storage clusters (we will provide read-your-writes consistency at the regional level). How can we invalidate stale cached data that has been updated in the storage cluster? These cross-cluster problems are going to be discussed in this lesson.


## Overview of design problems at the regional level
To manage the high workload, we add multiple front-end clusters that use the same storage cluster, but to do this, we need to manage replication and data consistency.

- If we scale a single cluster naively, our networks start to face incast congestion. Rather, we can replicate clusters when the load becomes too high.

- Replication and consistency: when we have multiple Memcached servers caching the data from the same storage cluster, how can we ensure that all the Memcached servers are up to date?

  - Should we send invalidations through web servers?

  - Should we send invalidations from the storage clusters to the Memcached servers for items that are no longer up to date or need to be updated?

  - Do batching invalidations help to reduce the packet rates?

- Sometimes we require a substantially high hit rate on a set of keys, but having a tight coupling between memory and throughput (such as packing the webserver and Memcached in a front-end server) can be memory-in-efficient.

- A new cluster doesn't have any key-value items cached to it; how can we bring it online without causing a decrease in the hit rate?

We will discuss the above concerns in detail in the following lesson.

## Invalidation from web servers
The following reasons explain why we want to split Memcache into front-end and storage clusters and then disproportionately increase the front-end clusters that serve the storage clusters.

1. We attain a system with smaller failure domains.

2. Easier to control and manage network configurations.

3. Reduction of incast congestion.

However, when dealing with a distributed cache with replicas, we need to efficiently flag items that are not up to date. When dealing with cache invalidations, let's first consider broadcasting them from the web servers. Even though this might seem like the obvious solution, it comes with two problems:

1. The packet overhead is larger due to it being a broadcast.

2. If web servers face systemic invalidations (for example, a misroute of a delete operation because of a configuration error), broadcasting them might exacerbate the problem. A rolling restart might be required for the whole system in such cases, which affects system availability.


## Invalidations from storage servers
The second option is to use the storage layer to inform the Memcache about any updates. For this to work, we can look for specific key changes in the SQL statements.

When a web server updates data on a storage cluster, it sends an invalidation to its own Memcached clusters. This way, Memcached servers have to re-request updated items from the database to maintain read-after-write semantics. However, when dealing with updates from the replica clusters, we can use an invalidation daemon at the database that checks SQL statements for invalidations to then broadcast those invalidations from the storage cluster to other Memcached clusters. SQL statements that modify the primary SQL database are modified to contain the Memcached key that needs to be invalidated.

[Invalidations]


However, this method of sending invalidations to all Memcached servers causes another problem. When many storage clusters communicate with many Memcached servers, it can cause unmanageably high packet rates (many-to-many communication patterns). To reduce packet rates, our invalidation daemons can batch deletes into fewer packets and send them to Mcrouter instances, which then unpack the deletes and send them to the correct Memcached server. As mentioned in this study, when batching our deletes, we see an 18-fold increase in the median number of deletes per packet.
```
Source: Rajesh Nishtala, Hans Fugal, Steven Grimm, Marc Kwiatkowski, Herman Lee, Harry C. Li, Ryan McElroy, Mike Paleczny, Daniel Peek, Paul Saab, David Stafford, Tony Tung, and Venkateshwaran Venkataramani. 2013. Scaling Memcache at Facebook. In Proceedings of the 10th USENIX conference on Networked Systems Design and Implementation (nsdi’13). USENIX Association, USA, 385–398.
```


## Controlling the degree of replication via regional pools
The important question to consider is, in how many places should keys be replicated. On one end of the spectrum, we can choose that each cluster independently caches and replicates within-cluster as per the client's requests, which are carefully routed to specific clusters. The second end of the spectrum is that we distribute the clients' requests uniformly and randomly to any Memcached cluster. Doing so will replicate keys at many clusters, and hence, taking out one (or a few) of the clusters will not adversely affect the hit rate. Such a high degree of replication will use memory inefficiently because not every key has the same access rate. We can borrow the idea of pools from the cluster level and use different clusters for keys with different access characteristics.

[Regional pools]

A regional pool is created when multiple front-end clusters share the same set of Memcached servers. If a cluster stores large items that are accessed rarely, then instead of having multiple replica Memcached servers, we can have fewer Memcached servers serving to a larger number of front-end clusters. This reduces unnecessary key replications. The decision to move certain keys to certain pools is made using heuristics based on the total data set size, the required access rates for those set of items, and the number of unique users accessing those items.

## Bringing new clusters online
Due to scaling needs and failures, new clusters are added to a Memcache system. However, if such an addition is done without care, it can incur a substantial load on the storage layer to fill the empty cache.

[New clusters]

When a new cluster is added, it will have very poor hit rates since it won't have any cached data. This means that there will be a significant load on the databases. To insulate our storage clusters, we can gradually bring them online. To "warm up" a "cold" cluster, we can route a cold cluster's cache misses to other already active clusters that have normal hit rates instead of our storage clusters.

One problem with implementing this is the race condition, where an update occurs in a database through a cold cluster, but the Memcached server gets the stale value from another Memcached server that hasn't yet received the invalidation. To avoid the race condition, we can hold-off the delete requests to the cold cluster for two seconds. When the cold cluster misses, the client gets the key from the warm cluster and adds it to the cold cluster. Due to the hold off period, the add fails and the client can then go to the storage cluster to get the latest version of the key.

## Summary
In this lesson, we learned what kind of problems we could face when dealing with replication at the regional level. In the next lesson, we will learn how scaling these clusters to a global level will affect the consistency of our system.

