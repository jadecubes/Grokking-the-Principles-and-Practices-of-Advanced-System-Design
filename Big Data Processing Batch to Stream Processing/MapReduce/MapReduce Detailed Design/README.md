# MapReduce: Detailed Design

Let’s discuss the MapReduce design in detail, including estimations and the detailed role of all the components—their internal mechanisms and the execution flow.

## Estimations
In our design, we’ll have to specify the number of workers required to achieve the task based on the input file size. Let’s make some assumptions before we formulate the workers’ calculations.
#### Assumptions
- The input file size is 12.8 TB (we assume this data size to simplify our calculations, as we will see shortly.)
- We will distribute the input data into splits of roughly 16 MB to 64 MB (we use multiples of 16 MB because the underlying storage of GFS’s chunk size is 64 MB, and we want to maximize data locality benefits).
- We can distinguish the tasks into two categories based on their executing functions— Map and Reduce.
- All workers are identical in their memory and computation resources.
- There are no malfunctioning workers.
```
Note: In this chapter, we’ll refer to workers as machines. Even though our design automatically handles the differences in workers’ memories and computation resources and the cases of faultiness, these calculations remain independent of these points.
```

### Estimating the Map tasks (M)
We can estimate the number of Map tasks required using the following formula:
Number of map tasks (M)= File size in MBs / 64 MB

Using the file size under the assumptions section, we can estimate the number of Map tasks: 

M = 12800000 MB / 64 MB =200000
 
### Workers estimation
Based on the number of Map tasks (M), we can estimate the number of workers (W) as well, making each worker perform 100 Map tasks.

Number of workers (W)= Number of map tasks / 100

Using the number from the previous calculation for Map tasks, this estimation is as follows: 
W= 200000/100 =2000


### Estimating the Reduce tasks (R)
Mainly, the number of Reduce tasks (R) are user-defined as each task produces its separate output file. We usually estimate this number as a small multiple of the available workers in the cluster, 2.5 as an example.

Number of reduce tasks (R)=Number of workers available∗2.5

Based on the number of workers above, we can estimate R as follows:

R=2000∗2.5=5000

You may change the input file size in the following calculator widget to see how it impacts M, worker count, and the R values.

```
                        MapReduce Estimations Calculator
          A                       B
1	Input file size (MB)	    12800000
2	Map tasks estimation—M	  =B1/64=200000
3	Workers estimation—W	    =B2/100=2000
4	Reduce tasks estimation—R	=B3*2.5=5000
````
Now that we have a basic estimation of the number of various worker types, let’s formally define the system’s components.

## Components
We can implement this MapReduce system in a distributed and parallel manner on a cluster of commodity machines with basic hardware capabilities. This execution will follow the master-worker pattern with the help of a MapReduce library that handles the internal workings automatically. Here is a detailed design describing all the components of such a system:

1. Cluster: To achieve our target of parallelization, we use a cluster of commodity machines. We divide the cluster members into two types based on their functions, along with a master that controls them.

    - Master: It controls all the subordinate workers, assigns them tasks, and acts as a relay for intra-worker communications. It stores the identity (a unique identifier for each worker) and state (idle, in-progress, or finished) of each worker to check each worker’s progress.

    - Mapper: These workers operate on the input data splits to produce intermediate key-value pairs using a user-defined Map function. The master handles the number of Map tasks required, M, to perform a particular job. Typically, one Map worker will run many Map tasks in a MapReduce job.

    - Reducer: These workers process the intermediate key-value pairs to produce the output file using a Reduce function. The number of Reduce tasks inside the system, R, is user-defined depending on the number of output files a user wishes to produce. Typically, one reducer worker will run many Reduce tasks in a MapReduce job.
```
The number of workers assigned for a Map or Reduce task will change over the life cycle of a MapReduce job. Initially, most of the workers will perform the Map tasks. When we have sufficient Map tasks completed, workers can be moved to do the Reduce tasks in parallel.
```

```
Question 1
Is there a need to run all Map workers simultaneously?

Answer
Since one of the functional requirements of our system is parallelization, running all available mappers simultaneously, at least in the first iteration of Map tasks, is the only logical approach. Moreover, one mapper will perform more than one Map task, so the master assigns the next iteration of the Map tasks based on availability.

Note: The Map workers are the actual servers that run one or more Map tasks over time.
```

```
Question 2
Can reducer workers run before mappers?

Answer
In our design, we first complete all the Map tasks and then perform the Reduce tasks. So, reducers can’t run before mappers.

But practically, we perform several pre-reduce tasks (discussed later in the chapter) in parallel with the Map tasks to minimize the overhead of the overall MapReduce job during Reduce tasks.
```

```
Question 3
Are there any practical bounds on the size of M and R?

Hide Answer
We estimated the M and R earlier in this lesson. The practicality of their exact number depends on the number of map-reduce states we have to store at the master for the whole operation.

We need approximately one byte of data to save the state of each map-reduce job pair inside the master. So, the total space complexity required for this at the master is O(M∗R). Hence, based on the RAM availability at the master, we can have practical bounds on the size of M and R.
```

[High-level cluster design]

2. The MapReduce library: This library handles the abstractions for parallelization, data splitting, dynamic load balancing, and fault tolerance. To achieve the processing tasks, this library inquires two user-specified functions, Map and Reduce. Both are discussed in detail later in this lesson.

It also allows users to fine-tune the parameters and tweak the processing. We’ll describe some of these refinements later in the chapter.

3. Network: We need network bandwidth for communication between the master and workers and to read input and write output operations. Moreover, the system has an intense phase that requires considerable network bandwidth. It involves communication among the workers since a piece of each Map task goes to all the reducers (such communication patterns are also called all-to-all communication patterns). Despite all these daunting network requirements, even commodity network infrastructure serves the purpose well for our system.

Ideally, we don’t need network bandwidth for the mappers, as the input data splits are stored on them locally using GFS. They process those splits and store the intermediate key-value pairs locally. In the worst-case scenario, if the master can not schedule a Map function on a server where its input split is residing, the Map function’s instance will need to fetch the data over the network.

In the Reduce phase, since the reducers have to do a lot of writing to the output disks, we need a considerable network bandwidth. A commodity network will suffice for our system to achieve these writing tasks.

4. File storage system: To store the input data splits and the output of Reduce tasks, we use GFS, which stores multiple copies of the same split on multiple servers, primarily three. We’ll explain the purpose of having multiple copies in the Locality section of this lesson.

After processing the input data splits, the mappers store the results of their intermediate key-value pairs in local disks (and not on GFS). The master then reads the locations of these intermediate key-value pairs from the local disks of mappers and passes them on to reducers.

[GFS creating multiple copies of each split on different workers]

```
Question
Why don’t we store intermediate data (the output of Map tasks) in GFS?

Answer
We store intermediate data on local disks because it’s faster and conserves network bandwidth. We don’t necessarily need multiple copies of intermediate data since we can quickly regenerate it in case of disk failures or data loss.
```

5. Scheduler: To perform Map and Reduce tasks on multiple machines in parallel, we need a scheduler inside the master to schedule tasks for the workers. It keeps checking the engaged and available workers and assigns a pending task to an available worker.

## Execution flow
1. The MapReduce library slices the input data in M splits of 16 MB to 64 MB (aligned with the GFS’s chunk size of 64 MB and to support locality in our system) and starts copies of the user program on multiple cluster machines.
2. One of the program copies is declared as a master. It’s responsible for assigning M Map tasks and R Reduce tasks to the rest of the workers based on their availability.
3. On one split, the master assigns a Map task to an idle mapper. The mapper parses the input split for <k1,v1> pairs and generates intermediate <k2, v2> pairs using the user-defined Map function and writes them to its local disk.
4. The MapReduce library’s partitioning function is responsible for splitting each mapper’s output, based on the output keys, into R partitions and storing them in R different files on the mapper’s local disk. The number of partitions equals R, which is the number of Reduce tasks. The program maps those intermediate <k2,v2> pairs to their respective R regions using a hash function, and sends their locations to the master. The master stores all the ready locations and is responsible for forwarding them to the available reducers.
5. The master notifies an available reducer about a ready-to-be-processed location, fetching the stored intermediate key-value pairs from the mapper’s local disk. The reducer sorts data by intermediate keys to group the entirety of instances of a key before processing it.

```
Question 1
For some data, it might be the case that the output of the Map tasks goes to a handful of intermediate keys. It is an example of imbalance, and if not taken care of, will substantially reduce parallelism because only a few Reduce tasks will get a huge amount of data.

How can our system handle such cases of imbalanced partitions?

Answer
Imbalanced data partitions can cause imbalanced load distribution among Reduce tasks. To counter such a scenario, we can use a pre-pass MapReduce job to compute the split points of the data.

It involves a pre-scan of the input data and taking a sample of keys from it. We then compute a baseline distribution of this sample and use it for split points calculation. Doing so helps us pick a suitable hash function to reduce or remove the imbalance issue.
```

```
Question 2
Why do we need a Reduce worker to sort the intermediate data?

Answer
Since we use a hash function to map the intermediate keys to their respective regions, each Reduce task can fetch the intermediate key-value pairs associated with several keys. The reducer needs to sort them before it can efficiently process them. It also facilitates the guaranteed ordering of the output keys, a feature we’ll elaborate on further in the chapter.

Since it has optimization benefits, we can even resort to external sorting if the intermediate data is large enough that the reducer cannot process it locally.
```

```
Question 3
When do reducers start their operations?

Answer
The output of map-jobs gets shuffled, sorted, and merged before passing it to the reduce-jobs (more on this in the Refinements lesson). Once this phase is over, the reducers start their operations.

To reduce the latency of the job, some riders can start early on, though they might need to wait for all the data to be available.
```

6. The reducer accumulates the values against each unique intermediate key, say k2, from the sorted intermediate data. It passes the unique intermediate key and the associated values to the Reduce function, which applies the required function to those values, generating <k2, f(v2)> pairs as its output.
7. The reducer writes its output to the respective output file on GFS.
8. Once all data splits are processed using the Map and Reduce functions, the master returns to the user code’s next instruction by waking up the user program.

[Execution]

```
Question
What are the space complexities for master operations?

Hint: The master needs to schedule M Map and R Reduce tasks and check their relational states.

Answer
The master mainly performs the following two functions:

1. It schedules the space complexity of O(M+R) since the total work is just the sum of the individual Map and Reduce tasks.
2. It checks the state with a space complexity of O(M∗R) since the total connections between mappers and reducers are the product of their numbers.
 ```

## Locality
Our system deals with tons of data, and one of the requirements is to achieve parallelization using commodity machines with minimum bandwidth requirements. We achieve this using the Google File System (GFS) to store our files locally on workers.

GFS creates copies of each split on multiple workers. The master initiates the Map task on a mapper containing the replica of the concerned input data split. In case of failure, it attempts to initiate the same task on another mapper containing the replica, ideally in the same network switch, to conserve network bandwidth.

Since the approach is to perform all the MapReduce operations utilizing a fraction of network bandwidth in the cluster, we can perform all the input data reading operations locally.

[Locality]

## Task granularity
Not all the machines in a cluster behave in the same manner, and that performance difference plays an integral role in the system’s overall performance. While creating splits for the input and intermediate data, we keep the values of M and R significantly larger than the number of workers, making each worker perform several tasks.
```
Note: As we mentioned before, the number R is specified by the user as it defines the final number of output files.
```
It facilitates in the following two ways:

1. Dynamic load balancing: While assigning tasks to workers, the master considers the previous performance of each worker and assigns more tasks to those which perform well. So, instead of uniformly balancing the load, it relies on dynamic load balancing backed by the performance analysis.
2. Fault tolerance: In case of failures, we have idle workers ready to take up the pending tasks at our disposal, complementing the lesser time complexity.
```
Note: To achieve this granularity, the master needs to complete O(M+R) scheduling decisions and maintain O(M*R) states in its memory, where each one of the states consumes a memory of one byte.
```

## Backup tasks
Some machines, commonly referred to as stragglers, take unnecessarily long to complete some of the few remaining Map or Reduce tasks, lengthening the overall time taken for a MapReduce job. There are multiple reasons for a machine to behave as a straggler.

- Faulty disk: A machine may experience a slow reading performance because of frequent correctable errors in a bad disk.
- Machine overload: Besides running MapReduce operations, a cluster manager also assigns other tasks to the servers. The cluster manager may have assigned other processing-power-hungry tasks to a worker, occupying its CPU, RAM, local disk, or network bandwidth, resulting in the slow execution of the MapReduce program.
- Code bugs: The bugs in machine initialization code may cause errors slowing down the computation of affected workers by a noticeable factor. A relatable example is a bug in code that results in disabled processor caches. Consequently, this slows down the computations on affected machines to a hundredth of what they were originally.

To overcome this problem of stragglers, the master, near the end of MapReduce completion, assigns the tasks (that straggler in-progress tasks are currently doing) to multiple other workers. It marks the pending tasks as completed as soon as it gets a confirmation message from the original or backup workers and ends the remaining tasks.

[Backup]

```
Note: We have to adjust this mechanism such that it doesn’t increase the computational resources by more than a few percent while ensuring a considerable decrease in the overall time completion.
```
Next, we’ll explore some refinements to the design, which are tweaks to achieve better performance.


