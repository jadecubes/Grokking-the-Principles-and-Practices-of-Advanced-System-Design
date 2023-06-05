# Refinements in Spark
The problems that Spark faces include worker failures and limited memory issues. It can also have driver failures for which Spark does not provide any tolerance.

## Managing limited memory
A Least Recently Used (LRU) eviction policy is used at the RDD level to manage limited memory. Whenever there is insufficient memory to cache a newly computed RDD partition, Spark removes an RDD partition that belongs to the least recently used RDD. However, we cannot do this if the newly computed RDD partition belongs to the same RDD that was least recently used. In that case, Spark will keep all the partitions to avoid cycling data in and out of an RDD.

Not removing a partition of an RDD whose partition has just been computed makes sense because, in Spark, an operation is usually performed simultaneously on all the partitions of an RDD. We cannot apply an operation on all of them if some of them have been removed, and the partitions of an RDD that are already in the memory are more likely to be used in future operations. Spark also gives users more control with persistence priority for RDDs. Priority decides which part of data in the memory will be moved to the disk first.

### Persistent RDDs
Spark provides the following three kinds of storage for persistent RDDs:

- Disk storage: It is useful when we have large partitions of RDD, which are very costly to keep in memory and expensive to be recomputed. This might not be the best strategy if there are a lot of repeated computations.

- In-memory storage as deserialized Java objects: It is the quickest persistence Spark offers because the Java VM can access RDD elements natively.

- In-memory storage as serialized data: It provides an alternative representation of the Java object graph, which is more memory-efficient, when the space in the memory is limited at the expense of performance.
```
Java object graph: A set of objects which will be automatically serialized, if the object which contains reference to them is serialized.
```

[Managing limited memory](./limited.png)

## Problem with the lineage graphs
Lineage graphs can always be used to recover lost partitions of an RDD, but if the lineage graph becomes too long, it becomes very time-consuming to compute or recompute partitions.

### Checkpointing in Spark
Unlike the cache() operation, saving a copy of an RDD in the disk is the process of checkpointing. The main difference between the persist() and checkpoint operations is that persist keeps the lineage of the RDD, and checkpoint forgets the lineage of the RDD. It becomes helpful to have support for checkpointing when there is a long lineage graph with wide dependencies.

- In the case of a long lineage graph with many wide dependencies, a lost partition may result in a temporary loss of some data from multiple partitions of parent RDDs.

- In the case of a long lineage graph with narrow dependencies, checkpointing may never be necessary. Lost partitions can be computed on different worker nodes in parallel with a fraction of the time consumed compared to computing a whole RDD.

[checkpointing](./checkpointing)

Spark provides an API for checkpointing, i.e., checkpoint(). Users can choose which data needs to be checkpointed. Finally, checkpointing RDDs is very simple because of their read-only nature. The user does not have to worry about the changes made to an RDD because there are none.

In this lesson, we learned how Spark manages limited memory and long chains of lineage graphs. In the next lesson, we will evaluate how Spark meets its requirements.
