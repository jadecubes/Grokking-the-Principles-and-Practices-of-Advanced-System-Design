# Introduction to Kafka
# Problem statement
Most big companies gather data, analyze it, and make decisions depending on the results. Delivering data from a source to a destination in real time in streams is crucial for many Internet companies. This task is called event streaming. It involves collecting data from different sources like user activity and click information of a page, log messages, credit card transactions, smart appliances, and software applications and storing data efficiently for retrieval by consumer applications later.

For many use cases, it is critical to collect and process individual events in near real time. One such example is to alert a credit-card user about a suspicious transaction and to take the user's consent to proceed. Not delivering such events quickly to the user or a user unable to pay because the system hasn’t yet processed the consent provided on the mobile phone can cause a range of issues, such as potential financial losses and consumer annoyance.

[Event streaming](./streaming.png)

Traditional messaging systems and more recent log aggregators have been used for event streaming, but they have shortcomings. So, there is a clear need for a streaming system that combines the advantages of both the existing log aggregators and messaging systems by being able to collect data from multiple sources, store it, and be able to supply it to any system that subscribes to it and does not have any of the shortcomings that these systems have.

Let’s explore the shortcomings of the currently available systems.

## Shortcomings in traditional messaging systems
A messaging system is a framework that allows different applications to exchange messages.

[Traditional messaging system](./traditional.png)

Traditional messaging systems like ActiveMQ, IBM Websphere MQ, Oracle Enterprise Messaging Service, and TIBCO Enterprise Message Service have been used as event buses in processing asynchronous data flows. However, they have had some shortcomings when it comes to log processing.

### Low scalability due to complex delivery semantics
Often, the abstractions and delivery guarantees provided by the traditional systems are complex and make their implementations hard to scale with increasing data. An example is IBM Websphere MQ, which provides transactional support that enables data producers to insert messages into multiple queues atomically. Another example is Java Message Service (JMS), which acknowledges the consumption of each message.

### Low performance
Throughput is not given due importance as a primary feature in these systems. For example, in JMS, each message requires a complete round trip of TCP/IP because it does not allow batching multiple messages in a single request, considerably affecting the throughput.

### No distribution support
Traditional systems offer little to no support for distributed processing of messages. They do not provide an easy way to divide and store messages on multiple machines in parallel.

### Unrealistic assumptions
Traditional messaging systems assume that messages will be consumed immediately, so the queue of unconsumed messages is small. Their performance considerably degrades if a bulk of messages are allowed to accumulate. The same happens with offline consumers like data warehousing applications that process huge datasets periodically instead of consuming them continuously. So, online consumption gets affected in case too many messages get accumulated, and offline consumption gets affected because small queues don’t have a bulk of messages that need consumption.

## Shortcomings in distributed log aggregators
A log aggregator is a system responsible for collecting log data from different sources and loading it into a data warehouse or Hadoop system for analysis. Some examples of recently employed log aggregators are Facebook's Scribe, Yahoo's Data Highway, and Cloudera's Flume. Most of these systems support distributed operations but have other shortcomings.

[Log aggregation system](./agg.png)

```
Some of the recently employed log aggregators are given below:

Scribe

- It is employed by Facebook.
- Each front-end machine sends log data to a group of machines over a socket.
- Each machine aggregates log data and dumps it in HDFS or NFS periodically.

Data highway
- The data highway project is employed by Yahoo.
- It has a similar data flow as that of Scribe.
- Each client sends event logs to a group of machines.
- Each machine aggregates these event logs and dumps “minute files” into HDFS.

Flume
- It was developed by Cloudera.
- It deploys “pipes” and “sinks.”
- It makes streaming log data very flexible and provides distributed support.
```

### No online processing support
We need online processing to provide low-latency, near real-time collection and data processing. Most log collectors provide offline services, where data is first collected in a data store and then processed.


### Expose unnecessary implementation details to end consumer
Some log aggregators (for example, Data Highway) make a log file per minute. They usually give consumers access to implementation details ("minute files").

### The push model
Most aggregators use the push model, which sends data to the consumer without knowing its processing capability. What we need is a pull model where consumers can get the data at their speed.

We present the Kafka system to overcome all of the shortcomings mentioned above.

# Requirements
Let's go through the functional and non-functional requirements of the desired messaging system.

## Functional requirements
Data streaming: The system should be able to collect and deliver data efficiently.

Low latency and high throughput: It should be able to deliver high volumes of data in real time.

Batch processing: Instead of just receiving one message per request, it should be able to receive and process a batch of several messages.

[Functional requirements](./functional.png)

## Non-functional requirements
Distribution support: It should provide integrated distributed support and be able to partition and store data in multiple nodes.

Scalability: It should be able to collect and deliver events efficiently even if the data increases over time through the horizontal scalability of its relevant components.

Data retention: It should be able to retain data durably so that the data is not lost even if consumers are not extracting any data.

[Non-functional requirements](./nonfunctional.png)

# Bird’s eye view
In the coming lessons, we’ll design and evaluate Kafka. The following concept map is a quick summary of the problem Kafka solves and its novelties.
[Overview](./overview.png)
