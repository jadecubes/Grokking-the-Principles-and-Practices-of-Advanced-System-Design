# Quiz on Spark

```
Question 1
How is Spark better than MapReduce?

Answer
Spark goes beyond MapReduce by providing low-latency processing for iterative and interactive workloads. MapReduce was not a good choice because of the excessive use of slow I/O operations on the local or networked disks. Spark utilizes the RAM of the cluster to avoid most of the slow I/O operations.

Using persistent disks enabled MapReduce to provide a simple fault-tolerant mechanism. Spark uses lineage-based fault tolerance where multiple concurrent workers can recover lost data.

Sparks’s primary goal is to provide low latency processing, while MapReduce is optimized for high throughput.
```

```
Question 2
How does Spark keep track of the data distributed in the memories of different worker nodes?

Answer
Spark uses resilient distributed datasets (RDD)—an abstraction of data kept on memories of different worker nodes by Spark. RDDs keep track of the distributed data by keeping the following:

A list of partition objects.
A list of RDDs it depends on.
A compute() function that can calculate any partition given the iterator at parent partitions.
To keep track of the partitioning technique used to make specific shards.
To keep preferred locations referring to worker nodes at which partitions are saved (to benefit from data locality).
```

```
Question 3
How does Spark eliminate setup overhead?

Hide Answer
When a variable like a machine learning model is needed for executing tasks on the partitions of each worker, the normal way of sending it to the workers is through the tasks. However, sometimes the tasks need recomputation, and it might be possible that multiple Spark jobs will need the variable. This process of sending a variable with each task creates a setup overhead.

To eliminate this setup overhead, Spark provides two types of shared variables. These variables can be shared only once with the worker nodes instead of sending them with each task. There are two types of shared variables in Spark—broadcast variables and accumulators.

- Broadcast variables are created at the driver and shared with all the worker nodes.
- Accumulators are used to apply add operation on the data at the worker nodes and can only be read at the driver node.
```

```
Question 4
How does Spark make RDDs and extract results from them?

Answer
Spark provides two kinds of parallel operations—transformations and actions.

Transformations are used to make RDDs from the disk or another RDD. There are two types of transformations: narrow transformations and wide transformations.
Narrow transformations are used to create RDDs with narrow dependency, in which each child partition is made from a single partition of the parent RDD.
Wide transformations are used to make RDDs with wide dependency, in which each partition of child RDD is dependent on multiple partitions of parent RDD.
Actions are used to get non-RDD values from an RDD.
```

```
Question 5
How does Spark recover lost data resulting from different faults (such as node failure)?

Answer
Spark keeps a lineage graph through which it knows which RDD was created from which RDDs.

When data is kept in shards on different worker nodes, it becomes highly unlikely that all of it will go missing at once. However, there is a chance that some partitions might go missing. To identify any such incidences, Spark’s scheduler keeps track of the partitions of persistent RDDs. When a partition of a persistent RDD is missing, it launches tasks at its parent RDDs and recomputes the missing partition.
```

```
Question 6
When and how does Spark compute the transformations? Does it keep all the RDDs in the memory all the time?

Answer
Spark’s scheduler keeps creating an execution plan by making a direct acyclic graph that builds the execution stages of a Spark job. A stage comprises RDDs that result from narrow transformations consecutively and separate at wide transformation because they cause shuffle operation. Thesecan’t be computed at a single worker node without communicating with the driver or other worker nodes. Spark only starts execution when an action is called.

Hence, Spark only materializes the RDDs whenever an action is called. Spark intelligently keeps those RDDs in the memory cache it needs per the execution plan DAG. Caching is either explicitly done by the programmer or via a history-based mechanism (least recently used list).
```
