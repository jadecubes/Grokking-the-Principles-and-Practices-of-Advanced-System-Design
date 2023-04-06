# Cross-regional Level of Memcache

## Introduction to the cross-regional level
At the regional level, latency was not a huge problem as the latencies inside a data center are around one millisecond, but as soon as we go to the cross-regional level, the latencies might go to around a 100 milliseconds. Due to this, and unlike in previous layers the CAP theorem comes in full effect and we have to choose between availability and consistency.

Cross-regional replication brings many benefits to our system:

- Firstly, it reduces latency by allowing clients to communicate with local Memcached and database servers.

- Secondly, it can mitigate the effects of natural disasters like earthquakes or hurricanes.

- Thirdly, having vastly different geographical locations can have economic incentives like cheaper electricity or land.

Replication on this level poses a challenge in maintaining the consistency between the primary and secondary regions.


## Overview of design problems at the cross-regional level
The storage layer is fully replicated across data centers. We use a primary-secondary set up to replicate data at the storage layer. One might think that once data is available in a data center, the Memcache layer can trivially work, though more care is needed to deal with a few subtle data consistency issues.

When data centers are available worldwide, we must manage the lag between them for data replication. Two problems occur when replication is in progress:

- Writes from a primary region: One problem that happens when a replication is occurring is how an invalidation from a primary region arrives before the data has been completely replicated to that region. Such a scenario can happen when the storage layer replication is lagging behind invalidation traffic from the front-end clusters.

- Writes from a secondary region: The other problem that occurs is that of updating data when it is in a secondary region. The user can update the data in a local replica–but due to a cache refill might show stale data because all writes and updates need to go to the primary storage and will be relayed back to the secondaries. Such a scenario can be confusing for the end users (for example, clients might see an item that they just deleted!).

The source of both problems stems from the storage layer lag.

[Overview](./overview)

One region holds the primary databases, while the other regions contain the read-only replicas. This replication is implemented using MySQL's replication mechanism. Writes, cache misses, and invalidations are sent to the primary region.

## Writes from a primary region
Using daemons to invalidate data significantly benefits writing from the primary region. A multi-region architecture can face race conditions where an invalidation from a primary region can arrive at a secondary region before the replication has been completed. Using invalidations from the database allows us to avoid this race condition. This is because MySQL inherently maintains the sequence of its transactions.


## Writes from a secondary region
When a user updates their data in the secondary region, the write is routed to the primary region, but if the user reads the same data and the replication lag is too large, then the user might get a stale version of their data. This means that the read-after-write semantics can't be upheld.

To fix this, we can set a mechanism that will:

1. Set a remote marker rk in the local region for a specific key.

2. Perform writes that are appended with the key k and the remote marker rk.

3. Delete the k in the local cluster.

When a web server requests the key k, it won't be able to find the cached data in the database, so it will check if rk exists. If rk does exist, the read request will be routed to the primary region. This way, we can trade latency for higher consistency.

Note that read-your-write consistency will be maintained as long the specific client interacts with the same data center. If, after a delete, a client is moved to a different data center (for example because user turned on a VPN), where changes haven't been reflected yet, the client can still see state values. Overall the consistency model provided by the system at the global level is eventual consistency.


## Summary
This lesson teaches the mechanisms that maintain the read-after-write semantics at a cross-regional level. In the next lesson, we will learn how our distributed key-value store performs as a whole.
