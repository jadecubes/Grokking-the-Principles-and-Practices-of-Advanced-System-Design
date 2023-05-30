# Introduction to Big Data Processing Systems

## Motivation
It might not be an understatement to say that data runs our world. From calculating accurate travel times for a map allocation by taking dynamic traffic information into account to personalized recommendations for pretty much all the services, such as shopping, list of songs, etc., it is data that needs to be harnessed to get the right information.

## What we will learn
We have selected three big data processing papers to discuss in the following few chapters:

[MapReduce] Dean, Jeffrey, and Sanjay Ghemawat. MapReduce: simplified data processing on large clusters. OSDI’04: Sixth Symposium on Operating System Design and Implementation (2008): pp. 137-150.

[Spark-I] Matei Zaharia, N. M. Mosharaf Chowdhury, Michael Franklin, Scott Shenker, and Ion Stoica. Spark: Cluster Computing with Working Sets. In 2nd USENIX Workshop on Hot Topics in Cloud Computing (HotCloud 10), 2010. (Initial version with brief details)

[Spark-II] Matei Zaharia, Mosharaf Chowdhury, Tathagata Das, Ankur Dave, Justin Ma, Murphy McCauly, Michael J. Franklin, Scott Shenker, and Ion Stoica. Resilient distributed datasets: A Fault-Tolerant abstraction for In-Memory cluster computing.. In 9th USENIX Symposium on Networked Systems Design and Implementation (NSDI 12), pp. 15-28. 2012. (Includes detailed explanation of resilient distributed dataset)

[Kafka] Kreps, Jay, Neha Narkhede, and Jun Rao. Kafka: A distributed messaging system for log processing.. In Proceedings of the NetDB, vol. 11, pp. 1-7. 2011.

## Why did we choose these systems?
There are hundreds of data engines out there and choosing a few was indeed hard. We picked some of the seminal papers that have stood the test of time.

### Start of big data processing era
MapReduce showed us how commodity servers can collectively process gigantic amounts of data. Another important aspect of MapReduce was a simple programming model for the end programmers. Traditionally, it has been challenging to use parallel and distributed computing to speed up the processing. MapReduce asks the programmers to write two functions (namely, Map and Reduce), and the system will take care of running them on the data even under different kinds of failures. The success of MapReduce was first witnessed by Google for its WWW crawling and indexing system. The Hadoop project (an open-source implementation of MapReduce and related technologies) spun an era where anyone could process large datasets economically and efficiently.

### Focusing on latency
The original focus of MapReduce was on increasing throughput, though there are many use cases where lower latency is also important. The Spark system reduced this latency by keeping and processing the dataset in the cluster RAM. While datasets are often too big to fit into the cluster RAM, Spark uses the working set principle, where only a subset of all data is being actively processed by the applications at any given time. This working set should fit in the cluster RAM. Over time, this working set changes, where new data becomes active and the old data becomes inactive (and therefore, can be pushed to the persistent store).

### Enabling real-time processing
While Spark’s processing engine can enable low latency processing, the latency in getting results also comes while collecting data, possibly from many geographically dispersed sources around the world. Kafka was specifically designed to quickly gather and disseminate data between producers and consumers. Spark processing engine can use Kafka as data source or sink.

We hope our selection of big data systems teaches us many important lessons in system design. Let’s dive in!

[Timeline of the evolution of big data processing systems]
