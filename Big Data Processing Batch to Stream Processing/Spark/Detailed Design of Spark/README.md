# Detailed Design of Spark
Spark can read input data from any Hadoop data source. Every Spark application has driver and worker nodes. An important consideration is how Spark would know which applications need how many workers to be executed. This is the job of the cluster manager.
```
Hadoop: It is a framework for distributed storage and big data processing that uses the programming model of MapReduce.
```

## Cluster manager
There can be multiple applications running in Spark. If a user starts its own application while some other applications are already running on a cluster of machines, they would need resources to allocate to their tasks. This is where the cluster manager comes in. The driver uses cluster manager (an external service) to allocate a cluster of machines to the application. The cluster manager manages the cluster by keeping an eye on the failed workers and replacing them with another, greatly reducing the programming complexity we had to add to Spark.

The cluster managers that Spark can use include Mesos, YARN, and Spark’s standalone. The option that is available on all cluster managers is static partitioning of resources, meaning that each application gets maximum resources and holds on to them for the duration of its execution. However, the following resource allocations can be controlled:

- The number of executors an application gets

- The number of cores per executor

- The executor memory

[Detailed architecture of Spark](./1.png)
```
Note: While a Spark cluster is capable of running multiple Spark jobs, often relatively smaller clusters are configured to utilize all the resources for one job. It usually reduce latency due to reduced contention for node-specific resources such as memory.
```
## Workflow of Spark
The driver gets the user program, on top of which it builds the RDD lineage graph by parsing the user program. Spark creates multiple tasks to process each data partition whenever an action is invoked. Now, it needs a certain number of workers to execute its tasks. It gets these workers from the cluster manager and then sends these tasks directly to worker nodes for execution and gets results from them.

[Workflow](./2.png)

As is evident from the illustration above, Spark keeps a map of RDDs and does not execute them until an action is called. When an action is called, the execution of the Spark job is started.

### Direct acyclic graph (DAG)
Spark builds a direct acyclic graph (DAG), an execution graph that contains stages of execution of several transformations. It groups all the consecutive narrow transformations to one stage. It defines a new stage whenever a wide transformation occurs, showing a shuffle operation in the RDD lineage graph. A stage can compute transformations in parallel if they are independent. But why does the scheduler need it? To group faster transformations and help mitigate the complications in fault tolerance.

### Job
A spark job is the highest level of execution in Spark. Each job corresponds to an action, which is called by the driver. A job includes an execution graph of transformations. No further transformations can be added once an action is called. Since action is something that returns non-RDD values, so it cannot have any children. Therefore, the actions form a leaf in the DAG of the job.

### Stage
A job can include one or several transformations. A stage can also include one or several transformations, but the difference is that the stage only contains a group of narrow transformations. A wide transformation marks the end of a stage if there are more transformations to be executed or the end of the job if an action is called. A stage can be thought of as a set of tasks that can be computed at a single worker without communicating with other workers or with the driver. However, a wide dependency that acts as a boundary line between two stages demands a shuffle operation and cannot be executed at one worker without communication with other workers or the driver.

### Task
A stage comprises multiple tasks, which is the smallest unit of execution in Spark. All the tasks execute the same code but on different partitions of RDD, so the number of tasks in a stage are equal to the number of partitions in the output RDD of the stage. One task cannot be executed on more than one worker. However, a worker can allocate memory slots to multiple tasks simultaneously.

[Spark’s execution hierarchy]

```
Question
How will Spark schedule the parallel operations?

Answer
To manage running tasks on multiple worker nodes, we need to schedule them. The driver has a scheduler for this purpose.
```

## Scheduler
The Spark scheduler is built to perform the following functions:

- The scheduler starts working as soon as an action operation is called. It starts the execution of the job by getting the RDD lineage graph from the driver.

- After examining the RDD lineage graph, it builds a DAG.

- Spark executes the stages according to the sequence defined in DAG.

- The scheduler keeps a tab on whether the partitions of persistent RDDs are available in the memory or not.

- For missing partitions, the scheduler launches tasks to compute them at each stage.

- It assigns workers tasks using delay scheduling based on optimal data locality.

- The scheduler sends a task that needs to process a certain partition to the worker node, which has that partition cached in its memory.

- Suppose the partition is not cached in any worker node, and the task is processing a partition for which an RDD provides preferred locations. In that case, the scheduler seeks that RDD to provide its preferred location (for example, an HDFS file assuming that each partition of an RDD is made from a block of the file in HDFS) and sends the task to that location.

[Scheduler]

The illustration above shows a lineage graph on top of which the scheduler makes the DAG. This can be seen on slide 2. In slide 3, we see two persistent RDDs, and also see that RDD4 is missing a partition. The scheduler, as we know, keeps track of the partitions in the persistent RDDs and launches tasks to recompute them. So, it launches tasks on each partition of RDD3 to recompute the missing partition of RDD4.


### Implications for performance
Narrow transformations are faster to execute because they do not require data movement across multiple partitions. They can be combined and executed in one pass of the data. Therefore, all the narrow transformations are combined in the same stage. A series of narrow transformations can be applied on any partition of an RDD.

A wide transformation needs data to be shuffled across multiple partitions. Computing a single partition at the child RDD needs data from several partitions in wide dependencies, since they are slower than narrow transformations, and are used to separate stages.


### Implications for fault tolerance
Computing missing partitions through a narrow dependency is much faster because only one partition in the parent RDD needs to be recomputed. However, if we have to compute the missing partitions through a wide dependency, multiple partitions in the parent node will need to be accessed and transformed.

So, the cost of computing partition in case of a failure with wide dependency is much higher than that of narrow dependency.

### An idea for fault tolerance
If the cost of computing an RDD partition is high, we can take advantage of the stages being separated at wide dependencies by checkpointing (keeping data on disks) or persisting data (Spark persistence is primarily in the RAM) between stages.

- We can persist or checkpoint it by keeping intermediate records to avoid complexities in fault tolerance.

- We can keep intermediate records on the worker nodes with parent partitions to simplify fault recovery between wide dependencies, just like the intermediate files that store Map outputs in MapReduce. How long we should keep this intermediate data around is an important decision (to be discussed in the next lesson).

- There are no mechanisms to recover from any driver failures in Spark. However, we can checkpoint the driver's state to a reliable data storage to recover from driver failures by extracting and using the checkpointed data in a new driver.

[Intermediate records between stages]

The scheduler helps Spark execute the tasks in stages efficiently. However, Spark has no mechanism in place for any kind of scheduler failures. Though, a copy of the RDD lineage graph can be kept for starting the process again.
