# Detailed Design of a Many-core System
Now, we will look into the detailed design of the solution to the problems we have while implementing a key-value store on a many-core system. We'll be discussing the solution to these three problems:

1. Memory limitation of TilePro64

2. Limitations of a multi-threaded Memcached

3. Allocation of tasks to cores


## Solution: Limitations of memory and Memcached
Our first challenge is that TilePro64 has a 32-bit instruction set, and we cannot assign more than 4GB of virtual address space to a single process. The second challenge identified how the use of global locks hurt the performance of our key-value store. Both problems can be solved by implementing a version of Memcached that supports multiple processes:

1. Use multiple processes to access the key-value store in their own address space and overcome the memory limitation.

2. Use data sharding (a direct consequence of using multiple processes) to parallelize data access and stop the use of locks within a shard.
```
Data shardings: Breaking data into smaller chunks.
```

### Use multiple processes
When using domain-specific processors, we run into unique problems like the TilePro64 having only a 32-bit virtual address space. So, rather than using a single process that uses multiple threads, we will use those threads to communicate with independent processes that contain the key-value data shard in their own dedicated address space. This will allow us to overcome the limitation of the 32-bit virtual address space, and each process will run its operations in serial while also allowing us to avoid inter-thread synchronizations.


### Modifying Memcached
We have a multithreaded key-value store which unfortunately needs locks. Locks limit parallel processing in our system. As we allow each process to have its own dedicated address space to overcome the 4GB memory limit, it also allows us to stop using any locking protection. This is because all requests directed to a hashtable are handled by a single process, which is already serialized.


### Flow of requests
After implementing the new multi-process version of Memcached, the following is the new flow of requests:

1. A network core receives a request and passes it to a worker thread.

2. The worker thread uses modulus arithmetic to identify the relevant shard process.

3. The worker thread writes the request's information into a shared memory, which is both accessible by the thread and the process. Shared memory is used to store request data until its process becomes available to process it.

4. Next, the worker thread notifies the shard process to STORE or GET request.

5. The shard process can then directly send the reply back to the client.

[Flow]

```
Question
In our multi-process design, do we still need a lock or synchronization between the request thread and the process holding the shard?

Answer
Yes, we need some synchronization between the request thread and the shard process because they will use a shared producer-consumer buffer for requests/data. The lock will not be contended (we have just two participants), and lock-taking and releasing is a fast operation (only a few instructions). Hence, it will not create a bottleneck.

Producer-Consumer Buffer: Since a producer-consumer buffer is used to work with shared memory.
```
## Solution: Core task allocation
The third challenge was optimally allocating each task to each core. Let's define the metrics we need to meet.



### Metrics
When calibrating a processor for a specific task, we first need to set some metrics that we can use to judge the processor's capability. Here are some of the metrics that we use to judge the processor's capability:

1. Responsiveness as median response time.

2. Throughput (capacity) is the highest offered load at which the mean response time remains under some constant value.

3. Power consumption of the system.

### Evaluating parametric space
We need to determine what kind of scenario we should use to simulate what our system might go through in reality. So, for example, key-value stores usually lean heavily towards more read requests than write requests.

Let’s consider whether having more write requests affects the read rates.

To test this, we can fix low, medium, and high write request rates and continuously vary read requests from low to high. We can then measure the read request latency to see if the higher write requests affect them. Usually, reading rates should not be affected as long as the write requests are normal (under 50k writes/second).

```
50k writes/second: M. Berezecki, E. Frachtenberg, M. Paleczny and K. Steele, "Many-core key-value store," 2011 International Green Computing Conference and Workshops, 2011, pp. 1-8, doi: 10.1109/IGCC.2011.6008565.
```

```
Writes/second            Reads/second x1000         Median latency for reads (Millisecond)

Low (5k)                   75k                         0.175

Low (5k)                   200k                         0.2

Medium (50k)               75k                          0.175

Medium (50k)               200                          0.2

High (200k)                75k                          > 1.2
```
Next, we can check how packet size affects latency when using our system. We can fix the read rate and vary the packet size from a few bytes to a few hundred bytes. In real-life scenarios, the packet size is usually less than 100 bytes.

[After a certain point the latency increases many folds]


The graph above shows a spike around the 1200 byte packet size—at a read rate of 100,000 transactions per second (TPS) means a bit rate of 960Mbps—which reaches the theoretical limit of the 1Gbps channel. Let's move on to how network protocols can affect our latency.

Let's consider whether changing protocols affects the speed at which our system reads.

We can test this by first using one protocol, then the other, while increasing the read rate from low to high. Usually, TCP connections take longer than UDP because of their transaction-based protocol. Therefore, we'll be using UDP capacity as our main metric for optimization in the next step. We usually use TCP for writes and UDP for reads. UDP does not need any initial connection setup or final tear down phases, making it low-latency option. For read-heavy workload, this choice benefits us (an example of making the common case fast design pattern.)

Moving on, we'll need to allocate roles to the cores present in our processor. Suppose we have 64 cores. We'll need four cores for Linux. That leaves us with 60 cores and the following roles:

- Network workers: Route requests to the demultiplexing thread on the network layer.

- Hashtable processes: Contain the key-value stores.

- TCP cores: Run threads that only respond to TCP requests like SET.

- UDP cores: Run threads that only respond to UDP requests like GET.

We will need to evaluate different core allocations to optimize performance systematically. In the following slides, UDP capacity means UDP transactions per second.

[Evaluating]

```
The parametric search space is large where we need to find out the optimal core allocation (64 of them) to five different functions. At times, domain-specific heuristics help us reduce the search space. Other times, we might not need the absolute best configuration—good enough might work. At times, adding more cores does not further improve the performance and performance plateaus (we will see that trend with network workers in our case). Such cases provide us an upper bound for core allocation for a specific resource, and once that number of resources has been assigned, we can remove them from further consideration. (There are more complicated scenarios where allocation changes to one function interfere with the others. For those cases, we might not be able to exclude some functions from further allocation decisions.)
```

Here's a summary of trends that were noticed while statically allocating the tasks:

```
Using a parametric space search, we find a reasonable core allocation to the functions. We do so offline. Once done, we use that allocation for our system without any further change, and we allocate cores exclusively to the respective tasks.
```

```
Role                Performance Trend

Network worker      Adding more network cores will stop performance improvements after a point

Hashtable           If we want a 24GB key-value store we'll use six cores. Adding any more cores does not improve performance because each process handles their shard independently. We cannot have more than one process per shard, as that would reintroduce global locks and inter-thread synchronization.

                    However, if we get hot spots on a shard, reducing the size of the shard and giving new piece(s) to a different process might help.

TCP                 More TCP cores do affect TCP reads. A few of the cores should be dedicated to the occasional writes.

                    Any core given to TCP is a potential loss in throughput that we could have gotten using UDP.

UDP                 UDP has a direct effect on read speeds, so allocating all available cores would be ideal.


```
## Summary

When responding to GET requests, the absolute latency of x86 is about 20% less than the TilePro64, but because both responses are under the 1-millisecond threshold, this difference is not qualitatively substantial for the gains made in power efficiency. Our Tile-based solution gives us about 4 times better TPS per second than the x86 system. The per-watt capacity offered by the domain-specific processor is far superior to the x86 processors.
```
1-million threshold: Which we set almost arbitrarily, as it is below the human perception level and fast enough for in cluster communication.
```
