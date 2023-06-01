# Quiz on MapReduce
```
Question 1
In MapReduce, we primarily focused on batch processing. Why is MapReduce not suitable for streaming data?

Answer
The MapReduce system processes a batch of data—when all the data we need is available, we do the processing. Many streaming workloads are different—data keeps coming in at variable rates and there is no end to this stream.

One way to use MapReduce on streaming data is to collect a mini-batch of data, run a MapReduce job on it, collect the next mini-batch, process it (by possibly utilizing earlier results as well), and so on. This is useful for two reasons. First, waiting adds latency, and for many streaming use cases, low latency is required. Second, utilizing results from a previous iteration becomes increasingly difficult in the simple abstractions provided by MapReduce. Therefore, we need a processing engine suitable for streaming data.
```

```
Question 2
We established that the MapReduce system provides a restricted programming model. Why do we call it restricted, and why is it useful?

Answer
The MapReduce system requires the programmer to do the data processing in two phases: the Map and Reduce phases. An additional restriction is that the input and output of these phases should be in the form of key-value pairs.

These restrictions enable the MapReduce system to automatically manage intricate details of parallel and distributed computing, fault management, and load balancing for efficient data processing using many servers. The programmer does not need to worry about them.
```

```
Question 3
During execution, the input data is divided into splits of 64MB. Why is this number significant, and how does it align with our other design choices?

Answer
The number 64 aligns with the underlying storage choice of GFS, which has a file chunk size of 64MB.

Our design supports data splits from 16MB–64MB (multiples of 16MB). In simpler words, we can store 1–4 input data splits in one GFS file chunk. When the master schedules work for the mappers, one deciding factor is the locality of the GFS input data on that server. By restricting data size within a chunk, we attempt to maximize the benefits of data locality and can avoid network use for reading data.
```

```
Question 4
In the Detailed Design lesson, we estimated the number of Map and Reduce tasks. Are these numbers dependent on one another?

Answer
The number of Map tasks depends on input data and split size (16MB–64MB), whereas the users define the number of Reduce tasks per the output files requirement. For better parallelism and faster fault recovery, it is recommended to use a larger number of Map and Reduce tasks as compared to available workers.

Even though the system and user define these two numbers, they are independent of each other (only the numbers are independent, the data produced by one is used by the others, and they are dependent in the sense that each M task will need to use R buckets for its output and each R tasks will need to fetch its bucket from all M tasks).
```

```
Question 5
To refine our design, we added a Combiner function before the Reduce function. Which properties of the Reduce function make it possible to use a combiner without affecting the final output?

Answer
Remember that the Combiner function will be run on all the M task’s output in a MapReduce job. A usual impact of a Combiner function is that it condenses the amount of intermediate data (how it does that is application specific). Due to the commutative and associative nature of the Reduce function, a Reduce task is not impacted (in terms of correctness) due to the application of the Combiner function.
```

```
Question 6
In the performance analysis section, we analyzed that disengaging some workers during their operation did not have a substantial negative impact on the system’s performance. How can we explain this using our design choices?

Answer
In the performance analysis section, we mimicked machine failures by disengaging machines. Our analysis showed us that it didn’t have a detrimental effect on the performance (just a 5% increase in the overall time) because of the worker failure mechanism we installed in our design. The master aggressively monitors task statuses and uses speculative re-executions to get the work done faster.
```

```
Question 7
To tackle stragglers, we introduced the reassignment of tasks to other idle workers. What if the reassigned tasks also become stragglers? How do you think our system will handle it?

Answer
Our system handles stragglers independently of whether they are first-layer stragglers or not. If our system marks a worker as a straggler, it will reassign its tasks to other idle workers, regardless of the straggler running a reassigned task itself. The system will keep reassigning the task unless it marks it as done.

If most jobs finish very slowly, a master might not run a lot of re-executions. The master detects a straggler by comparing the timespan of a job with the average finish time of already-completed jobs. If the average finish time is very high, the master might not be able to detect them until their running time goes above the average.

Nevertheless, having many stragglers might indicate a more severe issue, such as a failing network switch, and might need human intervention to recover. If there are a lot of stragglers on a specific server, the master can stop assigning that specific server any future tasks.
```
