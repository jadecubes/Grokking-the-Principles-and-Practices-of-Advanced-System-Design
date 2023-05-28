# Evaluation of ZooKeeper
## Throughput
In this lesson, we’ll evaluate the design of our Zookeeper. To keep it simple, we’ll discuss the throughput of read/write requests, load management, atomic broadcast, and system failure. However, before moving forward, let’s discuss the system’s requirements used for the evaluation.
```
Note: The experiment and results are taken from Hunt, Patrick, Mahadev Konar, Flavio P. Junqueira, and Benjamin Reed. “ZooKeeper: Wait-free Coordination for Internet-scale Systems.” In 2010 USENIX Annual Technical Conference (USENIX ATC 10). 2010.
```
### System specification
For the evaluation of the system, the number of servers was changed, but the clients were always 250. Even though we have implemented the clients in both Java and C, we have used Java servers and asynchronous Java clients in the experiment. Each client has a minimum of 100 requests for the server, where each request of either read or write will be done on the data of 1 KB. Since the performance of all the requests, such as create(), setData(), getData(), and many more, excluding sync() is approximately the same, we won’t explicitly discuss these functions. To keep the session active, the client sends the count of completed operations after every 
300 ms, and we have recorded the status after every 6s. To ensure that the server doesn’t get overwhelmed, we have added a throttle for requests which are concurrent 
2K requests, as shown in the table below.
 ```
    System Specification for Testing
Attributes                              Values

Number of clients                         250

Language                                  Java

Requests per client                     100 at least

Request on data size                      1 KB

Maximum concurrent request executions    2K at most
```
### Read/write requests
As discussed above, the system was tested with a different number of servers by changing the read-to-write ratio. As compared to write throughput, read throughput is higher because there is no atomic broadcasting in read operations. For example, the write throughput of 3 servers is 21k, less than the read throughput, which is 87k, as shown in the table below.
```
Source:Hunt, Patrick, Mahadev Konar, Flavio P. Junqueira, and Benjamin Reed. “ZooKeeper: Wait-free Coordination for Internet-scale Systems.” In 2010 USENIX Annual Technical Conference (USENIX ATC 10). 2010
```
 
```
          Throughput Performance at Extremes of System

Servers                  100% Reads                100% Writes

13                          460k                      8k	

9                           296k                      12k

7                           257k                      14k

5                           165k                      18k

3                            87k                      21k
```

The following are the two reasons why the write request takes more time than the read request:

Broadcasting overhead for the write requests, which the read request doesn’t have to do
Logging the transaction to disk before notifying the leader
The atomic broadcasting seems excessive, but the goal is to achieve reliability so we can trade performance with it. However, increasing the number of servers affects the performance of broadcasting, and helps the system be fault tolerant. For example, the write throughput of 3 servers (21k) is greater than the write throughput of 13 servers (8k).

Load distribution helps ZooKeeper achieve high throughput. The relaxed consistencies allowed us to distribute the load, unlike Chubby, where every client is connected with the leader (master). The graph below on the right shows the behavior of ZooKeeper acting as Chubby, with all the clients connecting with the leader (master). However, in ZooKeeper, the follower doesn’t forward any client to the leader or request the leader perform some operation for the follower. Therefore, the throughput of read-dominant workloads is much lower. As the follower only sends the write request to the leader after performing it locally, the write-dominant workload also has lower throughput.

[The throughput of ZooKeeper (Source:Hunt, Patrick, Mahadev Konar, Flavio P. Junqueira, and Benjamin Reed. "{ZooKeeper}: Wait-free Coordination for Internet-scale Systems." In 2010 USENIX Annual Technical Conference (USENIX ATC 10). 2010.)]

[The throughput of system where all read/write requests are handled by the leader(master) only (Source: Hunt, Patrick, Mahadev Konar, Flavio P. Junqueira, and Benjamin Reed. "{ZooKeeper}: Wait-free Coordination for Internet-scale Systems." In 2010 USENIX Annual Technical Conference (USENIX ATC 10). 2010.)]

If we compare both graphs, we can see that moving all the requests to the leader highly affects our performance, as the graph on the left has the behavior of ZooKeeper when the load is distributed to all the servers. However, in the graph on the right, all requests are sent to the leader. The starting and ending values of the graph on the right are taken from the table above. For 3 servers, the starting value of the graph (at 100% Write) is 21k, and its maximum value (at 100% Reads) is 87k and the same for other servers.
```
Note that the ending point of servers > 3 is not shown in the graph, as these values are too big.
```

### Atomic broadcast
The atomic broadcast is a vital component of the servers but affects their performance. The graph below shows the write throughput of the atomic broadcast since it decreases with the increase in the number of servers. Since the leader is broadcasting the write request to all the followers, clients can directly connect with the leader. However, for this experiment, the clients have been configured in such a way that they can only connect with the followers, not with the leader, so the leader won’t have to respond to the clients. The atomic broadcast hits the CPU limit at maximum throughput. Theoretically, the performance of the graph below should be equal to ZooKeeper’s performance with 100% writes.

[The average throughput of the atomic broadcast component in isolation (Source: Hunt, Patrick, Mahadev Konar, Flavio P. Junqueira, and Benjamin Reed. "{ZooKeeper}: Wait-free Coordination for Internet-scale Systems." In 2010 USENIX Annual Technical Conference (USENIX ATC 10). 2010.)]

However, the CPU is needed for other multiple tasks (such as the conversion of requests to transactions, access control list (ACL), which significantly decreases the throughput of the system. However, this is less than the effect on throughput by the atomic broadcast component alone. Robustness and correctness have always been the top priority while its development as ZooKeeper is a crucial production component. By removing things such as unnecessary replicas, numerous serializations of the same item, implementing highly effective data structures internally, and many more, we can considerably improve speed.
```
ACL is the list that stores the access information regarding who can access which znodes, checks, interacting with clients, and much more.
```

### Fault tolerance
System failures are the expected phenomena for which we have studied the ZooKeeper throughput. In experimentation, we kept the write percentage fixed at 30% and a lesser workload than expected. With time, we forcefully terminated some of the server processes. We have forcefully terminated the following processes, and these events are also shown in the graph below with the same numbers:

1. Failure of a follower and its recovery

2. Failure of another follower and its recovery

3. Failure of the leader

4. Failure of two followers

    a. Failure of one follower

    b. Failure of another follower

    c. Recovery of both of the followers

5. Failure of the newly elected leader

6. Recovery of the leader

[The throughput upon failure (Source: Hunt, Patrick, Mahadev Konar, Flavio P. Junqueira, and Benjamin Reed. "{ZooKeeper}: Wait-free Coordination for Internet-scale Systems." In 2010 USENIX Annual Technical Conference (USENIX ATC 10). 2010.)]

The following two observations are very important:

1. When the follower failure occurrs, the recovery of the follower can either be quick or delayed. This will affect the throughput by sharing that follower’s read requests with other followers.

    a. For quick recovery (where the failed node comes back in a short time), ZooKeeper will be able to maintain the throughput even in the case of failure. Quick recovery helps because the throughput is slightly reduced.

    b. For delayed recovery (where the failed node might take a long to come back), ZooKeeper will increase the throughput by distributing the traffic of the follower to other active servers. After the events, 1, 2, and 4, ZooKeeper is unable to get its full strength back, and the client only connects to another follower if the connection is lost from the previous one. The client connected to the leader doesn’t look for another server unless the leader doesn’t fail at events 3 and 5.

2. When leader failure occurs, the recovery of the leader should be quick to prevent throughput from dropping significantly. Within 200 ms, a new leader is elected by ZooKeeper. In that fraction of time, servers stop responding to the requests and get back to responding as soon as a leader is elected.

ZooKeeper handles such imbalances and does not affect the client’s view (the data it gets from the servers).


## Latency of requests
For the evaluation of the latency of requests, our benchmark is modeled after the benchmark of Chubby. In the benchmark, we performed the following two steps repeatedly:

1. We created a worker process and wait for it to finish.
2. We deleted the created worker process asynchronously.
In the testing, we created a different number of worker processes, but each worker process creates 50,000 nodes using the create() method. We computed the throughput being the average latency by using the formula below:
 ```
throughput= number of create requests completed / total time for all the workers to complete
```
 
```
      Create Requests Per Second
Workers             Number of Servers
                    3        5	      7	     9

1                   776	     748	    758     711	

10                  2074     1832     1572    1540

20                  2740     2336     1934    1890
```
 
The create() request with 1 KB data matches our anticipated utilization better, unlike Chubby, which has 5 bytes of data. Compared to Chubby, ZooKeeper has a throughput of 3 times higher. To compute the average request latency for 3 servers with a single worker, we have 
776 requests per second, so 1/776=0.0012 s =0.0012×1000 ms = 1.2ms. Similarly, for 9 servers with a single worker, we have 711 requests per second, so 1/711=0.0014 s =0.0014×1000 ms=1.4 ms=1.4 ms.

## Performance of barriers
To check the performance of primitives we discussed in the previous lesson, Primitives of Zookeepers, we conducted an experiment by executing n number of barriers sequentially. As we have discussed earlier, the double-barrier is where clients have to wait for b (barrier threshold) number of clients to enter the barrier before the execution starts in enter() and the same for leave().

The results of this experiment are given in the table below. We have successfully entered multiple clients such as 50, 100, and 200 in n number of barriers such that n ∈ {200, 400, 800, 1600}. ZooKeeper clients can be in thousands, but since they are frequently classified according to the characteristics of the application, only a considerably smaller fraction of clients actually participates in each coordinating activity.

```
        Barrier Experiment with Time in Seconds
Number of barriers          Number of clients	
                            50	    100	   200	

200	                       9.4	    19.8	   41.0	

400	                      16.4	    34.1	   62.0

800	                       28.9	   55.9	   112.1	

1600	                      54.0	   102.7	   234.4
```

We have taken the following two observations from the experiment:

The amount of time required to process all barriers increases fairly linearly with the number of barriers, demonstrating that unanticipated delays were not caused by concurrent access to the same area of the data tree.
Latency is directly proportional to the number of clients.
In reality, even when clients move forward in lock-step, we see that the throughput of barrier operations, enter() and leave() is always between 1,950 and 3,100 operations per second. If ZooKeeper is used, this translates to the throughput rates ranging between 10,700 and 17,000 operations per second. Each barrier-level construct used ZooKeeper’s primitives to implement it. The implementation has a read-to-write ratio of 4:1 (80% of read operations), which is significantly lower than the actual throughput (without read-to-write ratio) that ZooKeeper can achieve, which is 40,000 according to the graph on the left side under read/write requests. This happens because customers have to wait on other customers.

## Conclusion
ZooKeeper provides wait-free coordination by using a simple interface for the developers. This simple interface allows ZooKeeper developers to create multiple primitives using the client API. Even though ZooKeeper is not a locking service because of the wait-free feature, it can be used to build one.

### System design wisdom in ZooKeeper
ZooKeeper solves similar problems that Chubby solves but in a different way. This shows us that the same problem has many viable solutions with different tradeoffs. ZooKeeper’s consistency model enables its clients to choose a tradeoff between strong consistency, latency, and throughput as per need, while Chubby enforces a strong consistency model that impacted its throughput scalability.
