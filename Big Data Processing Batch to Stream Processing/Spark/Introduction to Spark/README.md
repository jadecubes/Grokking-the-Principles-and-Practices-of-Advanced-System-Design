# Introduction to Spark
## Problem statement
The original MapReduce system set the stage to process a large volume of data efficiently by adding more worker machines to get faster results. The runtime of MapReduce automatically managed the cumbersome details of distributed and parallel computing. Moreover, the programming interface was also very simple to use for the programmers. Even though MapReduce is primarily a batch-oriented data processing system, all the data to be processed should be available at the start of the process. Many of our data processing needs are not met by the MapReduce model. Two prominent examples are as follows:

1。Iterative data process where we use the same data repeatedly until we converge on some goal

2。Interactive, ad hoc queries on the same set of data

While one can argue that for both of the scenarios above, we can still use the MapReduce framework where we repeatedly use Map and Reduce tasks. Though the problem is that the latency to get the result will be non-real time and fairly high because each new iteration of the MapReduce job reads input data from the replicated persistent store (that it just wrote as output in the last cycle). We need a new processing framework without the inefficiencies of the MapReduce model.

[Iterative MapReduce]

```
Note: One might argue that use of a replicated, persistent store for the final output of the Reduce phase is wasteful. However, doing so simplifies many tricky aspects of data loss and integrity. We can also see it as a trade-off between design simplicity and performance (latency). It was a plausible design point because in a typical MapReduce job, the time spent in the Map and Reduce processing is much larger than the time spent on writing. Additionally, writing is highly parallelized when a sufficient number of Reduce workers write to many different servers of GFS like file system.
```
Now, we will see in detail the two use cases where MapReduce is not a good fit and then formalize the requirements of our new processing system.

## Iterative data processing
There are a lot of iterative machine learning and graph algorithms that reuse a dataset multiple times in parallel operations to get some results. An example machine learning algorithm is K-means on a big dataset for multiple iterations.


### K-means algorithm
The K-means algorithm is used to find k non-overlapping subclusters in a big cluster. It follows the pattern listed below:

1. The user specifies the k number of subclusters to be found.

2. The algorithm chooses k centroids in the feature space for finding k different clusters.

3. It assigns the points relatively nearer to each centroid point to a group to form a unique cluster.

4. It computes the centroid of the computed clusters again, relative to the points assigned to each cluster.

5. It repeat the 3rd and 4th steps until no more changes occur in the centroid's position.
```
Feature space: All possible values for features of a dataset.
```
[k-means]

We will read data from the disk, perform the first iteration of computing subclusters, and then write the results into the output file. For the second iteration, we will have to re-read the data of output file from the disk and then write computations into another output file in the disk again.


## Ad hoc queries on a dataset
Suppose we have a database containing data from a shopping mall, and we have to perform a list of queries on it: how many items came in, how much we sold, how much profit we made, etc.

[Performing ad hoc queries on a dataset repeatedly]

The system will have to read data from the disk every time a query is performed on the dataset.

I/O operations on disk are very slow, and if a disk is a replicated store, writing also takes extra time. We need to develop a technique capable of efficiently handling the reuse of a dataset in applications with inherent scalability, fault tolerance, and scheduling properties.
```
Scehduling: The process that decides which, when, and in case of parallel processing, where to perform a task.
```

## Problems with MapReduce
MapReduce is very efficient in parallel processing big data problems on commodity clusters, but those applications mostly have an acyclic data flow model. If we try to perform the MapReduce algorithm on an application that reuses a dataset multiple times, we can face the following sequence of issues:

Need for cascading MapReduce jobs: If we perform the K-means algorithm, we will need to cascade multiple MapReduce jobs to perform the iterations of the algorithm. We cannot sequentially apply more than two operations in a single go.

Inter job data persistence: MapReduce does not keep any data in memory for future operations. This means that a MapReduce job has to read data from the disk every time it’s initiated.

IO-induced latency: Every time a Map task is completed, it has to write on the local disk, and then a reduce worker has to read from that local disk. When a Reduce task is completed, it has to write results in a file again, which increases the latency due to inherent slowness in IO operations.

[Disk I/Os in MapReduce]

Considering the issues above with MapReduce, we need a system that keeps the data and its computations in memory for fast processing.

```
Question
What are the key characteristics that will enable a framework to perform iterative algorithms and ad hoc queries on a dataset with low latency?

Answer
They key characteristic is keeping data in memory.
```

```
Note: The key insight is that we want to keep data in the cluster’s RAM so that we don’t incur the cost of slow IO multiple times. However, the challenge is that often the volume of data is much more than the available collective RAM in the cluster.

The second insight comes from the operating systems. A typical process does not access every part of its address space at the same time. It is the concept of a working set where only a small subset of the full data is accessed in some period of time. By keeping the working set in faster caches, and replacing part of it well before it is needed (for example, via prefetching), we make a process run fast. Similarly for data processing in the cluster, we can keep the working set in RAM.
```

## Desired processing framework
In the early 2000s, we couldn’t load a large volume of data into memories because of its small size , so MapReduce was our best option. However, a decade later, we have memories that can contain data in excess of a few hundred GBs (servers with 128 GB to 512 GB are common in 2022). Hence, there was a need for a new framework that could keep a large volume of data in memory and use it repeatedly with inherent properties of MapReduce like fault tolerance, scalability, and scheduling.

[Desired cluster computing framework]

The desired framework should have the following key characteristics:

Distributed memory abstraction: It should be able to do an abstraction of data that performs in-memory operations on chunks of data in a cluster of machines. Just like the MapReduce framework, the programming model should be simple.

Multiple operations: It should be able to apply multiple operations sequentially without ending the job or having to spill it to the disks.

Low latency: It should keep the data in memory to achieve high-speed data processing.

Fault tolerance: Distributed memory abstraction should save information about where the chunks are computed within the data and how they were computed. So, if a machine where a specific chunk of data was being processed has failed, we can compute that chunk again from the dataset on another machine. We will need a new fault-tolerance model because when a server fails, the part of data in that server’s memory is lost (and failures are a norm in a large system).

Explicit ability to choose the working set: It should let users decide to store the results of specific computations they want to keep in the memory. Usually, a well-behaved program only touches a small subset of the whole data, known as the working dataset. We get good performance by bringing the working dataset into cluster memory and keeping it there.
```
Disclaimer: We will primarily focus on the Spark system as it was originally described in the Hot Cloud 2010 paper (Spark: Cluster Computing with Working Sets) and the NSDI 2012 paper (Resilient Distributed Datasets: A Fault-Tolerant Abstraction for In-Memory Cluster Computing). After that, Spark has been continuously improving under Apache Spark and other derivative commercial products. We will largely ignore those because our goal here is to learn important design aspects, and not necessarily to learn about specific products.
```

## Bird’s eye view
The following concept map summarises our work in this chapter. In the next few lessons, we will design and evaluate our system.
[A comprehensive guide to a fast in-memory processing system]
