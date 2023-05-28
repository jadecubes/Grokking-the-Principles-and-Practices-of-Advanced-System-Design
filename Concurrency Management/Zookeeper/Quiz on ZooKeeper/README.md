# Quiz on ZooKeeper

```
Question 1
How does the ZooKeeper server notify the clients when the watch is triggered?

Answer
A connection object is stored on the server when a client is connected to the server. Whenever a client sets a watch on a znode, a list of connection objects (of the client) requesting the update notification of that znode is maintained on the server. Once the znode is updated, all the connection objects in the list are notified about the change. Therefore they can use the updated value whenever they have to use that value.
```

```
Question 2
How is it that transactions are idempotent but requests are not, in ZooKeeper?

Answer
Multiple clients update the same data on different servers, and they send these requests to the leader (non-read request), and the leader creates the transactions using the Request Processor and forwards it to other followers. The client can send the same request twice, but transactions generated for the same data will not update the data twice.
```

```
Question 3
Why does the leader only use the request processor?

Answer
Since it is only the job of the leader to broadcast write requests atomically, that is why only the leader uses a request processor to convert the write request into a transaction and broadcast it to all the followers.
```

```
Question 4
ZooKeeper claims to be a wait-free coordination system, but we saw in locks and barrier implementations that end clients had to wait. How can we reconcile this apparent contradiction?

Answer
Firstly, ZooKeeper provides low-level primitives on which application-level primitives such as locks and barriers are built. If the semantics of a lock are such that only a limited number of clients can concurrently execute an operation, then the remaining one would have to wait. However, such a wait has nothing to do with ZooKeeper.

Secondly, ZooKeeper provides callbacks in the form of watches so that clients can be informed of interesting events just in time, and they don’t need to wait. An example of this is in a tight polling loop.

Therefore, there is no contradiction here.
```

```
Question 5
What is the consistency model provided by ZooKeeper for writes and reads?

Hide Answer
ZooKeeper does not provide strong consistency, primarily to achieve high performance and higher availability.

While all writes must go through the current leader, the dissemination lag might mean that different clients can read stale or different values at some point. The state eventually becomes consistent if new writes are not pending.

Clients that need stronger consistency might do reads only at the leader. However, if many clients start doing so, it will impact the performance for everyone.

Clients can use the sync() method to enforce updates until a point in time. Calling the sync() method synchronizes all the write operations on all the servers.

If all replicas of ZooKeeper are physically nearby, e.g, in one data center where latency between any two nodes is ~ 1 ms or less, clients might not see many consistency discrepancies. However, if some replicas are far away, e.g., in a different data center on the other coast, latency will be higher and consistency issues will be more visible.
 ```
 
 ```
Question 6
What fault model is assumed by ZooKeeper?

Answer
The original ZooKeeper assumed a non-Byzantine fault model because doing so matches the computational environment where nodes in a data center are in complete control of one entity.

Others have extended ZooKeeper protocols to work under a Byzantine environment, but we haven’t discussed that in this chapter.

The original ZooKeeper assumed a non-Byzantine fault model because doing so matches the computational environment where nodes in a data center are in complete control of one entity.
```

```
Question 7
ZooKeeper provides high read throughput because we can add more servers in the system, and clients can be distributed to them. However, write throughput is lower because writes need to go through a single leader.

How can we improve the write throughput?

Answer
There are two approaches to solve this problem.

Firstly, we can rely on the vertical scaling of the servers. One of the reasons for low write-throughput is that the leader node’s CPU maxed out. By beefing up the node, a leader can extract more throughput. However, because leadership can rotate amongst servers, such vertical scaling will be required for all the servers, which makes this solution a costly one.

Moreover, we might use multiple, independent instances of ZooKeeper and somehow divide the client load amongst them. The load division can be based on the isolation in the znode space so that dividing based on the operation type, such as locks, barriers, etc. However, doing so will require us to add some logic on the client library to first know if ZooKeeper is in this special mode and then to find out which instance to go to.

A more sophisticated approach is to shard the znode space among many servers, where for each shard, we have a separate leader. This is similar to what Chubby suggested.
```
