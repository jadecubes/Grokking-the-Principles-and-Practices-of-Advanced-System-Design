# Quiz on Memcache

```
Question 1
What advantage do we attain from using a key-value store cache instead of directly reading and writing to a SQL database?

Answer
Using cache we can provide low latency for the clients. Reading and writing in RAM is much faster than a disk-based database. Such a benefit amplifies because typical user-level content (for example, a web page) touches many keys.
```

```
Question 2
Why is it advantageous to have replicas when scaling?

Answer
A replica allows us to divide the user requests by the number of replicas we have.
In case of a failure, user requests are routed to a replica without a loss in hit rate.
A replica can also allow for lower latency due to being geographically closer.
```

```
Question 3
What problems do we face when we have multiple replicas?

Answer
Whenever we replicate data, we need to provide a data consistency model. Providing strict consistency models becomes harder when replicas are physically far away.
```


```
Question 4
What are the problems of an all-to-all communication pattern?

Answer
Incast congestion can occur when a lot of responses need to be sent back.
Network congestion might occur when many packets need to be delivered through the network.
```

```
Question 5
How is users’ data shared across the Memcache global system in the system we described?

Answer
Users can go to any of the available data centers to get service. Usually, we use DNS and related mechanisms to route user requests to a nearby data center to get low latency.

We assume that the storage layer is fully replicated across all data centers. That means any user’s data is available at all the data centers. Though, for a typical user session, some set of Memcached nodes will be caching all the keys a user has accessed.

Within a data center, key space is sharded using consistent hashing.

As data is increasing, it is becoming increasingly untenable to fully replicate all users’ data to all the data centers. How will you change this system to work under this new setting? You can think over this scenario as an exercise.
```
