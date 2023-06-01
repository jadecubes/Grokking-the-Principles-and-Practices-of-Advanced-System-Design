# System Design: MapReduce
## Parallel data processing—the domain of a Jedi programmer!
Parallel computing is known to be difficult, intense, and full of potential minefields in terms of Heisenbugs. Combined with the rise of many-core servers and distributed computing, parallel computing is useful and can’t be ignored by regular programmers to speed up their applications, or leave it to Jedi programmers.

```
Heisenbugs: A software bug that disappears or changes its behavior when one attempts to investigate or resolve it.
```
Google introduced a new programming model (MapReduce) that enabled programmers (of any expertise level) to express their data processing needs as if they were writing sequential code. The MapReduce runtime automatically takes care of the messy details of distributing data and running the jobs in parallel on multiple servers, even under many fault conditions. The widespread use of the MapReduce model proves its applicability to a broad range of data processing problems.

In this chapter, we will study the design of the MapReduce system in detail and the application of its programming model.

## Motivation
Information extracted from different kinds of data sources plays an increasingly integral part in our society. Some examples include collecting, summarizing, and indexing various data from the World Wide Web for efficient searches and detecting anomalies in outbound data from some organizations.

```
In 2022, the Google Search index contained approximately 48 billion web pages, and is well over 100,000,000 gigabytes in size.
```
As the volume of one data source increases, there is also a growing variety of new data sources available. Recent trends show that the data increase rate far exceeds the available computational resources needed to transform raw data into actionable information.

The graph below shows the trendline of processing power, data growth, and storage cost since 2000.

[A graph representing the trend of processing power, data growth, and storage cost since 2000]

Although the trendline for storage cost shows support for storing this enormous amount of data generation, it’s important to note that the trendline representing the processing power is not increasing at the rate according to Moore’s law. To keep up with the era of big data, we need to develop systems capable of processing the data in a distributable and parallel manner.

In addition to having systems capable of processing this massive data, we also need efficient tools that allow programmers to concentrate on their business logic and analytics while hiding the complexities of parallel and distributed computing from the end user. The MapReduce system was (and remains) a seminal effort to achieve this goal. In this chapter, we will design the MapReduce system.


## MapReduce
MapReduce is a restricted programming model to process large datasets (big data) effectively and efficiently, structured or unstructured alike, on a cluster of machines. One of the model’s restrictions is that the input and output of the processing code should be in key-value pairs. It takes input data from key-value pairs and aggregates user-defined processed output as different key-value pairs. The input is a large and complex dataset, and the computations get distributed across numerous machines to finish the processing task in a reasonable amount of time.

The following illustration depicts how the process works.
```
restricted programming model: A programming model where the programmer is expected to adhere to a specific way of programming but, in return, gets some benefits
```

[An overview of the MapReduce system]

```
Note: We’ll design this MapReduce system in detail in the following lessons.
```
It basically gets its motivation from LISP’s map and reduce operations and requires two user-defined functions, Map and Reduce. It automatically provides an abstraction for all the internal data distribution mechanisms, parallelization, load balancing, and fault tolerance.
```
LISP: A programming language developed in 1958.
```

```
Note: The guiding rule here is that many hands make light work. To process huge amounts of data quickly, we can divide the massive input data into smaller pieces, give these pieces to multiple worker machines for processing, and later combine the results.
```
### Advantages
The key advantages of using MapReduce for processing large datasets are listed below.

- Automatic parallelization: One of the main advantages of MapReduce is its inherent parallelization abilities for batch processing big data. It distributes the task among multiple workers and processes. The dataset is split to produce faster results.

- Code simplicity: Besides parallelization, the system also automatically handles the implementation logic for data distribution and result accumulation, which eases the operational complexities of processing vast amounts of data. Otherwise, all this implementation demands great insight into programming knowledge related to computer architecture, operating systems, computer networks, and distributed computing.

- Good fit for computation and graph problems: MapReduce performs exceptionally well for processing large amounts of complex raw data. It processes data such as crawled documents, web request logs, etc., to derive valuable insights, such as graph representations of crawled documents, inverted indices, specific request frequency over a period of time, etc.
Massive data generation (from social networks, tweets, messages, and many more) has given rise to interesting and complex graph problems that are impractical to be solved using conventional analysis techniques. MapReduce provides a decent solution for partitioning and analyzing such graph problems.


### Disadvantages
MapReduce has its inherent limitations for some use cases. A few of them are as follows:

- Not a good fit for streaming data: MapReduce does not perform well for streaming data systems, where data keeps coming in over time. Additionally, workloads that iteratively reuse the same data might perform poorly with MapReduce due to its fault model that each iteration of a MapReduce is independent of the next.
- Impractical for non-parallel tasks: It’s also impractical to use MapReduce for problems that are inherently serial data processing basedor problems that do do not lend themselves to data distribution.


### Requirements
Let’s look at the functional and non-functional requirements for the system.

#### Functional requirements
The functional requirement for our system is to provide us with an adaptable programming model to process big data, supported by the following features:

- Data partitioning: The system should be able to distribute the input data into standardized splits, as defined by the Google File System (GFS). GFS is a distributed file system that we will use to read input and write the output. A standard GFS chunk is 64 megabyte, and to achieve good locality, we should split our input into 64 megabyte chunks. We’ll discuss the exact details of this partitioning later in the chapter.

- Parallelization: Our system should be able to distribute and process multiple input splits simultaneously on various workers. It should also be able to manage them through a central machine, referred to as a master.

- Dynamic load balancing: Our system should distribute the tasks among workers based on their performance analysis for faster results. That means faster servers get more tasks and vice versa.

- Fault tolerance: We can expect components or network failures in a commodity cluster. When unchecked, such situations can produce partial results for our MapReduce operation. Since we can’t afford to have partial results, our system should automatically handle component, server, or network failures to ensure complete results for each iteration.
```
Note: Usually, fault tolerance is a characteristic that we classify as a non-functional requirement. We have kept it in the functional requirements here because for MapReduce to properly work on commodity clusters, fault tolerance is critical.
```
[The functional requirements for an adaptable programming model of MapReduce]

#### Non-functional requirements
- Throughput: Our system should have a high throughput ensuring maximum data processing in a given time.
- Latency: Our system should ensure that the data processing happens with low latency.
- Scalability: Our system should be horizontally scalable; for instance, adding more servers can speed up the processing.
- Reliability: Our system should be reliable in performing the assigned tasks. Although we’ve defined fault tolerance as a functional requirement, dealing with non-MapReduce-specific faults in our system can be added as an additional non-functional requirement.
- Availability: Our system should ensure high availability. This non-functional requirement is a by-product of two functional requirements mentioned above: fault tolerance and dynamic load balancing. We can say that if our system is fault-tolerant, it’ll automatically ensure high availability.

[The non-functional requirements for MapReduce]

## Bird’s eye view
The following concept map summarizes this chapter. We will now dive deep into designing and evaluating the MapReduce system in the upcoming lessons.

[The mind map for MapReduce design problem](./overview.png)

Now that we have introduced the MapReduce system and defined its requirements, let’s discuss its design in the next lesson.
