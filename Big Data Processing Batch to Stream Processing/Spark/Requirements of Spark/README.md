# Requirements of Spark
## Functional requirements
The functional requirements of Spark are listed below:

- Data processing: The system needs to process a large working dataset efficiently and also be able to do it repeatedly for iterative or interactive queries.

- Latency and throughput: Our system should achieve low latency and high throughput for the tasks, like iterative data processing (where we use the same data repeatedly) and performing ad hoc queries on the same dataset. For example, we expect our system to query many terabytes of data in a few seconds. Usually, the first run is slower than the others because our system needs to load the data from the disks that involve IO operations.

[Functional requirements of Spark]

## Non-functional requirements
Following are the non-functional requirements of Spark.

- Fault tolerance: If a data partition is lost, it should be recovered effectively.
```
Data partition: For parallel processing data is divided into partitions called data partitions.
```

- Data locality: The system should do computations on the worker where the data resides.

- Persistent in-memory data: The data that needs processing is only loaded once and should persist in the memory for fast access in the future. Our system should enable the programmer to explicitly tell the system which data should be kept in memory.

- Memory management: There should be a fallback strategy to manage workloads bigger than the available cluster memory, where the working set can slowly change by swapping out unused data to disks so that new data can be loaded.

[Non-functional requirements of Spark]

For example, an evaluation of Spark to perform an iterative machine learning problem known as the K-means algorithm was conducted. The K-means algorithm was performed for iterations on 100 GB of data using 100 machines. Iterations of K-means are compute-intensive because they find the distance of each point in a cluster with its centroid, compute new centroids, and then partition the clusters repeatedly. Spark’s performance was compared with Hadoop, which employs MapReduce on K-means. Since this algorithm can take several iterations to converge, results have been presented for the first iterations and then all the subsequent iterations.
```
Evaluation: Taken from www.usenix.org/legacy/event/hotcloud10/tech/full_papers/Zaharia.pdf
```

In the first iteration, both Hadoop and Spark read data from Hadoop Distributed File System (HDFS), so there isn’t much difference. Spark is slightly better due to some specific optimizations.

In the following iterations, Spark achieves about two times better performance than Hadoop. If we had an application that was I/O intensive, Spark could give 20 to 40 times better performance than Hadoop (because Spark will do the operations from memory instead of expensive IO).

[Approximated iteration times of K-means algorithm]

## Estimations
In our design, we’ll have to specify the number of workers required to achieve a task based on the input size. There can be a lot of latency and throughput variances in the tasks performed by Spark. The reasons for such variance can be heterogeneous commodity servers with varying computational capacity, the load of the servers, potentially failing memories, disks, network interfaces, and other unforeseen failures during the processing. For coarse-grained estimates, we can ignore such sources of variance. We’ll make the following assumptions:

### Assumptions
- The input size is 25.6 TB (a made-up number to simplify our math).
- Data will be distributed into multiple partitions of 256 MBs each.
- Let’s suppose we have to perform three functions filter, Map, and Reduce.
- Every operation gets its own set of workers, and no worker performs more than one operation.
- All the workers are identical in their memory and computation resources.
- The usable memory of each worker will be 25.6 GB (an assumed number to simplify our calculations).
- A single machine failure happens for every 100 machines working in parallel.

### Tasks estimation
- We can estimate the number of partitions on which a filter operation  will be applied using the following formula:

Tf = file size in MBs / 256 MB

No. of filter tasks come out to be: 25600000 MB / 256 MB  =100000

- Let’s assume that each filter after filtering 256 MB of data gives an output of 128 MB, so the size of partitions from here on will be 128 MB.

- Map is a one-to-one function, so it will have the same amount of tasks.

- Reduce operation will be performed on every single machine holding the results from the Map operation, so it will have the same number of tasks as filter and Map tasks.


### Server estimation
- For filter tasks, we can find the number of partitions a filter worker (P_F)  will be able to process with the following formula:

P_F = RAM size MBs / Partition size

In this case, it will be equal to be 25600 MBs / 256 MBs =100

- The formula for the number of workers (W_F) needed for filter tasks will be: 

W_F = Number of filter tasks / Number of partitions in a filter worker

In this case, they will be equal to 100000 / 100 = 1000
- Because the size of partitions is halved, the Map and Reduce workers will be able to store twice the partitions compared to filter workers, so we will need half the workers that are 1000 / 2 =500.

The total number of workers needed for these operations will be 1000 + 500 + 500 = 2000.

One worker fails for every 100 workers, so the percentage of total workers is 0.1%. The total number of workers, including the replaced workers, will be 2000 + 2000 / 100 =2020

```
   A                                              B
1 Input file size (MB)                    	25600000
2 Tasks estimation	                        =B1/256 = 100000
3 RAM size (MB)	                              25600
4 Partition estimation in a filter worker  	 =B3/256 = 100
5 Workers estimation                      	 =B2/B4+2*B2/(B4*2)+(B2/B4+2*B2/(B4*2))/100 = 2020
```
Now that we have an idea of the requirements and estimations of Spark, we can start to ponder on how to design it.
