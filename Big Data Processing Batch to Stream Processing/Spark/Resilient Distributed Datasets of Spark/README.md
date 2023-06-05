# Resilient Distributed Datasets of Spark
RDDs provide a restricted form or an abstraction of shared memory based on coarse-grained transformations rather than ﬁne-grained transformations. Simply put, RDDs are distributed data on a collection of worker nodes' memories based on coarse-grained transformations in a cluster.
```
coarse-grained transformations: A transformation applied with the help of a function like Map and Reduce on a bulk of data.
 ﬁne-grained transformations: A transformation applied to an entity of a database.
```

# Creation of RDDs
RDDs are an object in the language they are being made. We can build an RDD in the following ways.

## From a file
An RDD can be built from a file in a distributed file system (DFS). It would create an RDD in which each block of data in DFS will be a partition in the RDD, and each record in a partition would represent a line in that file.
```python
val RDD = spark.textFile("dfs://file")
```
## From a collection
An RDD can be built by parallelizing a collection that is inherently sequential. It could be an array or a list. The user can decide the number of partitions to make out of a collection.
```python
val RDD2 = (1 to 9999).toList.par
```

## From another RDD
RDDs can be made from other RDDs in two ways.

- An RDD can be built by transforming an already built RDD––from the lines that contain NULL in them.
```python
val RDD3 = RDD.filter(_.contains("NULL"))
```
- It can also be built by altering the persistence of an existing RDD. RDDs, by default, are lazy (created on demand when used in parallel operations) and ephemeral (eliminated from the memory afterward). To change this behavior, Spark provides two actions:

    - Cache: The cache action makes Spark keep data in memory for further use after its creation. However, if there is no memory to cache the data, it recomputes it whenever used again. This process is opted to ensure that the system keeps running even if a node fails or there is a memory shortage.
    ```python
    val RDD4 = RDD3.cache()
    ```
    - Save: The save action evaluates and saves the data to a distributed file system to perform operations on it in future.
    ```python
    val RDD4 = RDD3.saveAsTextFile("dfs://saved_data")
    ```
    [another RDD](./rdd)
# Representation of RDDs
Internally, each RDD can represent five pieces of information, two of which are optional. Let's assume we have two RDDs, i.e., RDD1 and RDD2, with two partitions each.
[RDDs](./rdds.png)
RDD2 will have the following information.

- It gives a list of partition objects, i.e., C1 or C2.

- It lists other RDDs it depends on, i.e., RDD 1.

- It has a compute() function that computes the elements of a specific partition, i.e., C1 or C2, given the iterator for its parent partitions.

- Optionally, it gives metadata specifying the technique used for partitioning the distributed memory abstraction. In this case, we could say that RDD2 is hash partitioned or range partitioned.

- Optionally, it gives a sequence of strings with some information about the nodes where a specific partition p can be quickly accessed. In an RDD made from an HDFS file, each string represents - the Hadoop name of the worker from where the partition can be accessed (for the initial partitions, each partition is based on one HDFS chunk so that we can benefit from the locality of data). In this case, C1 can be accessed at Worker 1, and C2 can be accessed at Worker 2.

```
hash partitioned: A technique which is used to partition the keys that have the same hash value and distribute them in partitions.
range partitioned: A technique which is used to partition keys based on their order and distribute them by a specific range to different partitions.
```


## Controlling partitions
Spark automatically partitions the data in RDDs and distributes it over the cluster of memories of available nodes. However, it also provides control to the user on how to partition the data residing on a cluster of machines. So, the user can specify a certain kind of partitioning for RDDs. When the data is in the form of key values, partitioning becomes necessary to decrease data shuffling between different nodes in an RDD while performing parallel operations.

- Repartition: We can apply the repartition operation on an RDD to increase or decrease the number of partitions it already has. This operation causes the shuffling of data across worker nodes.

- Repartition and sort: We use repartitionAndSortWithinPartitions to repartition an RDD and also specify the order in which they will be kept. The user can provide both the number of partitions and technique used to sort data.

- Coalesce: We use the coalesce operation to decrease the number of partitions in a worker node. This operation does not shuffle data across nodes because it decreases the number of partitions on a worker level.

- Hash partition and range partition: When the data is in the form of key values, hash and range partitioners come in very handy. We use HashPartitioner to distribute data evenly across different nodes depending on the keys. It takes the number of partitions and a hash code to partition the dataset. When an RDD has keys that follow a certain ordering, RangePartition is used to group tuples with keys that lie in the same range in the same machine.


# RDDs vs. DSMs
Next, we will compare RDDs and DSMs to understand why the designers did not just use the traditional distributed shared memory system distributed shared memory system instead of RDDs.
```
 distributed shared memory system: In a distributed shared memory system, we utilize the RAM of all participating nodes such that the system gives the illusion of a long, random-access array to its users. The system takes logical addresses from the processes and then translates it to a specific node, and then forwards the read or write request there.
 ```


Read/Write: In DSMs, read and write operations are done on specified locations in the global address space, but in RDDs, both coarse and fine-grained transformations can be used on the dataset for reading, and only coarse-grained writes are permissible under RDDs.

So while the RDDs are restrictive in terms of DSM, sacrificing some generality allows us to get simpler fault tolerance.

Fault recovery: DSMs have to use checkpointing to keep a version of the whole dataset at a particular time frame, which is very costly in terms of spatial restrictions for storing data, meaning that even if we want to checkpoint data only once, we will need double the amount of original memory to store it.

However, RDDs do not need to bear the expensive spatial overheads provided by checkpointing because we can recover RDD partitions using a lineage graph. Furthermore, in RDDs, only the lost RDD partitions can be recomputed in parallel on a different worker node without having to apply all the transformations again on the full data or go back to a certain state where checkpointing had been done.

Therefore, RDDs provide data recovery that is much better in terms of spatial overheads. It is a big deal because our system will utilize many commodity-class nodes in a data center, where nodes can fail or memories can get corrupted.

Immutable nature: RDDs are immutable. If an operation is applied to an RDD, a new RDD is returned instead of the modifications in the original RDD. This nature of RDDs lets the system run backup tasks in parallel to mitigate stragglers, just as in MapReduce. However, it is very hard in DSM to run multiple copies of a task because they will access the same memory locations and interfere with each other's updates.

Hence, RDDs are a feasible option when it comes to running multiple tasks in parallel on different machines as compared to DSMs.

Data locality: When a bulk operation is applied on an RDD, the system can schedule tasks based on data locality to achieve better performance. However, this is not the case for DSMs.

Hence, RDDs are better than DMMs in scheduling tasks based on data locality.

Storage on disk: When there is not enough memory to store RDD partitions, they are saved on the disk. DSMs have no mechanism for memory shortages, and they might resort to operating system-initiated swapping between the RAM and disk partition.

Therefore, the RDDs can handle memory shortages better as opposed to DSMs as long as we use the concept of a working set or linear data scan. Otherwise, for the general case, both systems will provide poor performance due to excessive IO.

```
Question
Why are RDDs kept immutable in Spark?

Answer
They are kept immutable to keep the lineage graphs as simple as they could be. Otherwise, Spark would have had to keep versioned RDDs and keep track of versions in the lineage graph.
```

```
Note: Spark’s RDD abstraction resembles the distributed shared memory (DSM) in that it also utilizes memory of many participating nodes for data. However, RDD does provide a restricted programming model as compared to DSM. For example, Spark does not allow random writes, rather it provides bulk writes. By sacrificing byte-oriented writes, Spark gains a simple and efficient fault-tolerance mechanism. This is important because in a cluster of commodity nodes, failures are a norm than exception.
```

RDDs are an integral part of the Spark framework. They help Spark to keep the data distributed in the memories of different worker nodes and allow it to be easy to use, scalable, and fault tolerant. We can consider each piece of the RDD as a shard of the original data.

Now that we know how the data will be kept in Spark, we can discuss how to perform operations on data.
