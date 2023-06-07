# Efficiency of Kafka
Kafka has a few features that make it efficient, like simple storage, efficient transfer, and a stateless broker.

## Simple storage
Kafka has a simple layout for storage. Key parts of its features are discussed below.

### Implementation of a partition
A partition can be implemented as a large file. However, Kafka does not keep the data forever. It has to clean older data from the disk to make way for new data. If partitions are implemented as a large file, it is hard to find and clean data that is no longer needed. Therefore, a partition in a topic is implemented like a logical log that comprises a set of segment files approximately of the same size. This way, we can append new messages in a segment file by deleting messages from the oldest updated segment file without having to find and delete a part of a large file and then append data to it.

The broker appends each message to the last used segment file or an active segment whenever a producer publishes it to a partition. These segment files are flushed to the disk. However, to achieve better performance, the system waits till a segment file has gathered a certain amount of data or if a certain amount of time has passed before writing it to the disk, whichever happens first. A segment file usually contains either 1 GB or a week’s data. Consumers can only read a message after it has been flushed to the disk.
```
Question 1
What if a broker fails before flushing the segment files to the disk?

Answer
If a machine fails before flushing a segment file to the disk, all the segment files and partitions saved in it would be inaccessible to the consumer.

The cluster controller notices this failure, and it also knows that all the partitions that had the leader in the failed broker now need a new leader. It accesses the replica list that is kept on ZooKeeper, which has the information regarding the leader and replicas of each partition, to determine a new leader partition. It selects the next replica in the replica list as the leader partition and sends a request to all the brokers (that either have a new leader or replicas that lost their leader), containing information regarding the new leaders and the followers for the partitions.
```

```
Question 2
How does Kafka clear up space to make space for more data?

Answer
Two configurable parameters can be set to delete messages from the brokers i.e., the quantity of data that has been gathered (1 GB by default) or the time it has been retained (7 days by default). These are treated as limits to retain data in the brokers. If data exceeds 1 GB in a topic, or a topic’s data is retained for a week, Kafka starts deleting messages in the oldest segment file of that topic. Moreover, Kafka cannot delete messages from an active segment.
```
### Message IDs or offsets
Kafka stores each segment in a single file that comprises messages and their offsets. In other messaging systems, every message has its ID. However, Kafka addresses each message by a logical offset. This feature eliminates the overhead of maintaining random access data structures that map an ID to locations in storage where messages are saved. The offset of messages in a partition is in increasing order, but it’s not a constant increase for each message because each message can be of different lengths. To compute the ID of a new message being appended to a partition, we must add its length to the offset of the last message.

### Consumption of messages
A consumer sends pull requests to the broker, which keeps a data buffer ready for consumption. It consumes messages from a specific partition in a sequential manner. A consumer having a specific message offset indicates that all the messages with lesser offsets have already been consumed.

#### Pull request
A pull request is composed of an offset and a number of bytes. The offset corresponds to the message from where the consumption will start, and the number of bytes corresponds to an acceptable number of bytes that must be fetched. A broker has a list of sorted offsets that includes offsets of the first messages of all the segment files. Whenever a broker receives a pull request from a consumer for a batch of messages, it searches the offset list for the potential segment file that could have those messages and then returns the messages to the consumer. After receiving the messages, the consumer computes the offset of the next message and sends another pull request.
[List of sorted offsets in the broker]

```
Question
What offset will the consumer use for the first pull request on a partition?

Answer
A partition’s offset starts with 0. So, the 0 offset will be used by the consumer in the first pull request.
```
##### Number of bytes in a pull request
Users can set a limit on the number of bytes that a consumer can get from a broker, for example, 1 MB. If a consumer tries to get a message larger than 1 MB from the broker, it will get an error. If the number of bytes specified in the pull request is smaller than 1 MB, it will get an error on messages larger than the specified number of bytes. However, if the specified number of bytes is larger than 1 MB, it will return messages whose size stacks up to be equal to or less than the specified limit.

## Efficient transfer
Following are enhancements of Kafka related to the handling of data transfer.

### Data flow in batches
Kafka is very efficient in transferring data in and out of it.

- A producer can dispatch multiple messages (batch) in each send request.

- Although a consumer API goes through every message individually, it also gets a batch of messages in a single pull request. It gets messages up to a certain size, usually around 100s of KBs.

[Data flow in batches]
```
Note: Batching messages helps reduce the latency for producers and consumers. This is especially useful when producers, brokers, and consumers are physically far away from each other.
```
### Caching
A common approach for caching is to keep data in the RAM for speedy access, reads, and writes. This is because one of the goals of Kafka is to safely store incoming messages on storage devices, Kafka does not employ explicit caching of data. Though the writing and reading of messages happen sequentially, the file system’s page cache acts as an implicit caching (both for writing and reading). This implies that the write-through policy (both the page cache and storage are updated simultaneously) will help because, first, data is persistently written on the disk, and second, a consumer (usually slightly behind producers) will read the data from the cache.

#### Benefits
The benefits that page cache and caching heuristics provide are as follows:

- We avoid double buffering by avoiding writing messages explicitly into the memory of a user-land Kafka process. (It is called double buffering because the data is buffered firsts by Kafka application and second by the implicit cache managed by the file system's page cache.)

- When messages are buffered in the page cache by using write-through or read-ahead heuristics, a warm cache is retained even when a broker process is restarted, making the broker avoid reading from the disk every time the consumer asks for messages. It is possible because the Kafka process has not explicitly kept data for the cache, and the file system cache is usually not flushed right away.

- Kafka can be implemented efficiently into a VM-based language (for example, Java) because of the absence of explicit caching. VM-based languages can lag when it comes to garbage collection. Since Kafka is not caching any messages in the memory, it results in little to no garbage collection overhead in memory, making way for its efficient implementation in VM-based language feasible.

### Optimized network access
Kafka has optimized network access when it comes to the subscription of a message by multiple consumers. Typically, the following steps would be required for sending data from a local file to a remote socket:

1. Pull data from storage into the page cache in an operating system.

2. Send a copy of the data in the page cache to an application buffer.

3. Send a copy of the data in the application buffer to a kernel buffer.

4. Send the data in the kernel buffer to the remote socket.

[Methods for transferring data from local files to remote socket]

The whole process takes four data copying and two system calls. However, Kafka uses a sendfile API that exists in Unix and Linux operating systems. The API can directly transfer bytes of a file to a remote socket. A sendfile API can reduce a system call from steps 2 and 3 and reduces the copy operations from four to two. Kafka takes advantage of it, delivers bytes of segment files from a broker to a consumer, and avoids overhead copy and system calls.

## Stateless broker
The features that Kafka deploys on a broker are discussed as follows.

### Information at the broker
In Kafka, information about the amount of consumption done by the consumer is not maintained by the broker. It is maintained by the consumers themselves on ZooKeeper, which also helps in case of consumer failures, which are discussed in the next lesson. This feature decreases the overhead and complexity of the broker.

### Deletion of messages
Maintaining consumption details by consumers creates a void of information at the broker. It makes it difficult to delete a message at the broker. The reason is that it does not know if all the consumers that were subscribed to that message’s topic have consumed it. For this sole purpose, Kafka employs a time-based service level agreement (SLA) to retain messages. Kafka saves 1 GB of data or the data of a week, whichever is small in the log segment files, and after a segment of the file has become seven days old or has 1 GB of data in it (whichever condition is met first), it is deleted and replaced by a new segment, if available.

[Deleting a message]

Because each consumer takes at most a day to consume messages, it can do so in a matter of hours or even in real time. The solution that Kafka employs for retaining messages works well in practice without a degradation in performance, making itself a long retention feasible framework.

The rationale behind message expiry is that it is necessary to absorb any traffic spikes (where consumers might lag behind the producers). However, on average, the rate of production and rate of consumption should be equal over some time period. If that is not the case, it means data might keep piling up in Kafka indefinitely, which is not a plausible situation for the system stakeholders.

### Side benefit
The design that we have discussed so far also has another benefit. A consumer can violate the queue principle and consume messages it already has consumed. It can rewind to an offset of a previous message and consume it. It is a very crucial feature when it comes to fault tolerance.

- When an error occurs in the code of a consumer, the application can rewind the offset and consume certain messages after the error is dealt with. This is important, especially when loading data into offline consumers like Hadoop or a data warehouse.

- When consumed data is being flushed to a persistent store, and a consumer fails, we no longer have a track of the unflushed data. It can be catered to by checkpointing offsets after certain gaps and start flushing the messages from the last checkpointed offset when the consumer is restarted.

We will get some duplicated flushing of messages. However, the number of duplicated flushes can be decreased if we make the checkpointing more frequent. Moreover, this method is very convenient for pull methods instead of push methods because a consumer has control over the messages that it will be getting.

In this lesson, we learned how keeping simple storage, transferring the data in batches in an optimized network, and keeping as less information on the broker and clearing it with time to make way for newer data can help make Kafka an efficient framework.

