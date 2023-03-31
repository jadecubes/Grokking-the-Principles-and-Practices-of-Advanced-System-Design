# Cluster Level of Memcache

## Introduction to the cluster level
At the single server level, we didn't have to worry about routing or replication. Once we start to deal with thousands of servers, we need to understand the problems that arise with them. The Memcache server clusters' key load is managed by consistent hashing, but there are still challenges to tackle. Network congestion, too many repeating cache misses, dynamic workloads, and cluster failures are all problems that we face at the cluster level and not at the single server level.

Clusters are manageable units of a data center. The number of nodes inside a cluster is configurable. Nodes inside a cluster can communicate with each other with low latency and high throughput (because they are often near each other). After scaling our key-value store on single nodes, the next level is to utilize multiple key-value stores in a cluster.



## Overview of design problems at the cluster level
At the cluster level, we attempt to solve a read-heavy workload and a wide fan-out. This wide fan-out occurs because a single web request will get routed to a few clusters and then will further trigger multiple Memcached requests.

```
Fan-out: New content needs to be notified to more number of users, whereas the creation of new content is more rare.
```

Network congestion: Why do Memcache clusters face network congestion in the first place? We can explain this by giving the example of loading a user feed of posts. One web request can trigger tens, if not hundreds, of Memcached requests that are used to construct the feed.

Too many repeating cache misses: If we can't respond to a request quickly enough, the front-end servers consider it as a cache miss and the data will be fetched from a more costly path. So, we need mechanisms that reduce the rate of cache misses.

Diverse application needs: Different applications have different requirements for their caches, so there might be some set of key-value items which are very expensive to compute again, like all the birthdays of a user's friends. On the other hand, another set of key-value items, such as viral images that are recommended at random, needs to be replicated quickly, and it's okay if some of the servers miss it as users aren't following the page/person that shared it.

Cluster failures: Machine failure is inevitable in any distributed system. So what do we do when a request comes in and is routed to a cluster that has failed? There is a slight chance that the load might cause other clusters to fail, resulting in "cascading failures."


## The overall design of Memcache
The individual layers of our cluster-level design are as follows:

Stateless client layer: A stateless client to the Memcached is responsible for optimal retrieval of items from Memcached servers.

Mcrouter layer: Mcrouters are used to route requests to multiple Memcached servers using consistent hashing. To the client layer, the Mcrouter layer is the same as the Memcached layer.

Memcached server layer: Stores and serves the actual key-value items.

[Design]

Let's discuss cluster-level optimizations we can make to improve our caching system.



## Network efficiency problem 1: Data dependencies
Problem: How do we reduce the number of network round trips? When a lot of requests need to be handled together, we can usually batch them. In this case, we need to be careful that there might be data dependencies between the requests.


Solution: We'll use a Directed Acyclic Graph (DAG) to maximize the concurrent calls that can be made to Memcache. Rather than making individual updates serially, we can use a DAG to identify which operations are independent of the others and run them in parallel.

Using data dependencies, the web application constructs a DAG to send it to the web server. The web server uses the DAG to execute as many operations concurrently as possible to reduce network round trips for a given request.



## Network efficiency problem 2: Comparing UDP and TCP
Problem: When getting data, which protocol should we use? Remember that we need to reduce latency.

UDP is much faster due to it having a connectionless protocol and a smaller header size. However, TCP is more reliable in getting data to the destination than UDP. Does using UDP instead of TCP bring more performance benefits even with the packet loss rate increase?
```
Packet loss rate: Ratio of lost packets to the packets successfully transmitted.
```
Solution: Compare the latency between the two protocols while fetching data.

To keep the system simple, we'll maintain stateless Memcache clients where possible. This allows us to deploy new versions easily. This also allows us to handle most of the complexity on the client side. We then use a Mcrouter to route data from the Memcache client side to the server. Memcache clients also have a map of all the servers they can access. This map is updated using an auxiliary service. According to a study, there is roughly a 20% advantage to using UDP for get requests over TCP. We can also decrease network load by directly allowing the webserver to access the Memcache servers while fetching data.

```
Source: Rajesh Nishtala, Hans Fugal, Steven Grimm, Marc Kwiatkowski, Herman Lee, Harry C. Li, Ryan McElroy, Mike Paleczny, Daniel Peek, Paul Saab, David Stafford, Tony Tung, and Venkateshwaran Venkataramani. 2013. Scaling Memcache at Facebook. In Proceedings of the 10th USENIX conference on Networked Systems Design and Implementation (nsdi’13). USENIX Association, USA, 385–398.
```



## Network efficiency problem 3: Incast congestion
Problem: Incast congestion occurs when a large number of responses to requests overload a data center's cluster and rack switches. This is a problem that comes when there is an "all to all" pattern in communication, which comes when web servers need to communicate with many Memcached servers to complete a user request.

Solution: Use a sliding window mechanism—similar to what TCP uses to implicitly manage congestion.

## The repeating cache miss problem
### Causes of high load
### Stale sets
### Thundering herds
### Using leases to reduce load
## Diverse application needs problem
### Replication within pools
## Handling failures
### Smaller outages
## Summary
