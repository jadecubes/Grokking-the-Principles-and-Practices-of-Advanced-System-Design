# Evaluation of Spark
Spark can be used efficiently for many data processing use cases. Spark does data processing in memory. Hence, it should provide low latency. Other functionalities that Spark provides include fault tolerance, data locality, persistent in-memory data, and memory management. Let’s discuss how well Spark provides these functionalities with the following arguments.
```
Note: All the computational results and time spent on them that is stated in the text below are gathered from the paper Resilient Distributed Datasets: A Fault-Tolerant Abstraction for In-Memory Cluster Computing. Experiments are done on a data of 100 GB with approximately 25 to 100 machines (for different experiments) with 4 cores and 15 GB RAM each.
```
## Latency
When we use Spark to perform an algorithm that requires more than one iteration, for example, the K-means algorithm or logistic regression, we get to know the speed-up achieved by Spark. Suppose we perform such a task with Hadoop (an open-source implementation of the MapReduce framework). In that case, it will run slower even if we use HadoopBinMem (HadoopBM), which converts the data into a binary format and stores it in a replicated instance of in-memory HDFS for the following reasons.

Overheads: The first overhead that makes Hadoop slower than Spark is the signaling overhead due to Hadoop's heartbeat protocol between master and worker nodes. Running no-operation tasks on Hadoop gives a minimum overhead of 25s because of job setups, starting tasks, and cleaning up, as well asan overhead of HDFS while serving data to each block.

```
Hadoop's heartbeat protocol: Driver keeps sending a signal to the workers to check if they are working fine or not, if there is no response from a worker, that means the worker has failed.
```

Deserialization cost: Hadoop also takes time to process text and convert binary records to Java objects usable in memory. This overhead occurs even in all cases, whether the data lies in the in-memory HDFS of a local machine or in an in-memory file.

Spark stores RDD elements as Java objects directly in memory to avoid all these overheads.

In the first iteration of the K-means algorithm performed for 10 iterations on 100 machines, Spark completes the first iteration in 82 seconds. Hadoop is a bit slower than Spark because of its heartbeat protocol. HadoopBinMem completes its first iteration in 182 seconds and Hadoop in 115 seconds. HadoopBinMem is the slowest because it has to perform an additional MapReduce job to convert data into binary format and write it in an instance of in-memory HDFS.

[First iteration times on different frameworks in K-means algorithm](./firstiteration.png)

In subsequent iterations, Hadoop completes each iteration in 106 seconds, HadoopBinMem in 87 seconds, and Spark in 33 seconds. Surprisingly, Spark is still a lot better than HadoopBinMem even when it performs no extra jobs, and that is because HDFS makes multiple copies in memory and a checksum to serve each block.

[Subsequent iteration times on different frameworks in K-means algorithm](./subsequent.png)

## Fault tolerance
Spark recomputes RDD partitions of a failed node using lineage graphs. For example, if we are executing an algorithm performed in iterations like the K-means algorithm on 75 machines and a worker node fails in, let’s say, the sixth iteration of the algorithm, this results in a loss of tasks and data partitions on it.

Spark reruns these tasks on another worker node in parallel, which computes the lost RDD partitions from the relevant input data via lineage. This process consumes 22 extra seconds as opposed to the time normally taken by the iterations experiencing a 38% percent increase in time, which is about 58 seconds. However, it does not affect the execution of further iterations.

[Difference in iteration times of K-means algorithm due to failure of a worker node](./diff.png)

Suppose the recovery is wholly based on checkpointing. In that case, Spark might have to perform several iterations again to recover the lost partitions depending on the checkpointing frequency. It would have to keep tracking back in the lineage graph until it finds a checkpointed RDD.

However, lineage graphs of RDDs only consume a negligible of memory.


## Data locality
The Spark scheduler assigns tasks based on data locality. Let’s suppose a task has to process an RDD that resides in memory on several specific nodes. In that case, the scheduler directly sends it to those nodes, and if a certain partition is missing, it will be computed at the preferred locations provided by the RDD. In the illustration below, the scheduler sends the task that has to be applied on RDD1 exactly on the machines where RDD1 is kept.

[Driver sending task to Worker 1 and 2](./locality.png)

## Memory management
RDDs are an abstraction of memories spread across a set of worker nodes. If, under any circumstances, the machines run out of memory, Spark removes a partition of the least recently used RDD to fit in the newly computed partition. However, some tweaks can be used to avoid memory shortages.

- Users can persist or cache only those partitions that can be used repeatedly. Users can specify which partitions they will reuse and choose how they will persist—in memory or in storage. There are two further ways to store data in memory: deserialized Java objects or serialized data.

- To simplify fault tolerance, the users can also cache or persist only those RDDs formed due to a wide transformation.

- If it's not feasible to do both of the above because of memory shortage, RDDs can also be checkpointed in the disk.

- Persisting RDDs found due to join transformations can also be avoided if we avoid wide transformations. For that, users can specify the same partitioning technique for both RDDs that are supposed to be joined.

Let’s study some experiments that include the execution of logistic regression under different circumstances.

- If we keep all the 100 GB data inside the memories of 25 machines, Spark completes an iteration in 11.5 seconds. However, if we do not keep any data in the memory, it takes 68.8 seconds to complete an iteration.

[Spark’s performance with different volumes of data in memory](./1.png)

- If we use a different number of machines for executing the logistic regression algorithm, we’ll have a different amount of memory available. If the execution is done on 25 machines, Spark takes 15 seconds for each iteration after the first iteration, and if it is done on 100 machines, Spark takes 3 seconds for each iteration.

[The Spark technique’s performance with different number in memory](./2.png)

## Conclusion
In this chapter, we learned about a framework that can store and perform operations on a large dataset in memory. Spark provides two simple data abstractions RDDs and shared variables. These abstractions are significantly better at expressing a lot of applications that the previous cluster computing frameworks were unable to. Unlike existing cluster computing frameworks that need to replicate data to provide fault tolerance, Spark provides a lineage graph-based recovery method for data, and that, too, with coarse-grained transformations. Spark also outperforms the existing frameworks in iterative applications and applications that need to query a large dataset.


### System design wisdom in Spark
At a high level, the core contribution of Spark is its RDD abstraction and lineage-based fault tolerance, where we can keep data in the RAM for fast reading and writing after parallel processing and can quickly recover from lost data. There was a lot of engineering around this idea to turn it into a working system. This is common for many problems. A key contribution (keeping data in memory instead of disk) with a lot of supporting engineering is required to solve them.

Spark is a restricted programming model (just like MapReduce was) where it does not allow fine-grained writing to distributed memory. However, by trading off this generality, Spark provides an efficient fault-tolerance model that can recover lost RDD partitions quickly.

In conclusion, Spark is a major step forward from MapReduce to efficiently process iterative and interactive workloads.
