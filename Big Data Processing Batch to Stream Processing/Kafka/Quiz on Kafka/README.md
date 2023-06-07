# Quiz on Kafka
```
Question 1
If consumers need to pull data from the brokers, and assuming no data is available for consumption, a consumer might have to poll the broker periodically. How can a consumer avoid such polling?

Answer
Kafka can block an incoming consumer call if no data is available. As soon as at least one message is available, the blocked consumer will be unblocked and handed out the data. This way, we get two benefits:

Consumers don’t need to poll. This doesn’t waste resources when no data is available, but it also forces the consumer to come up with an appropriate value of wait time (before the next polling). Using a large wait value can add to the latency for the client, and a small value might mean many useless polling round-trips.

A broker can readily send the data to the consumer as soon as it is available. It reduces client-visible latency, which is important for Kafka to be suitable for real-time use cases.
```

```
Question 2
Producers send the messages to different brokers, and brokers give the messages to consumers. Why do we need broker intermediaries? Why can’t producers directly provide messages to consumers?

Answer
There are many benefits to having Kafka as the intermediary between producers and consumers:

- It is usually hard to precisely predict data production (for example, a sale event triggering a sudden traffic spike). Similarly, it might be the case that different consumers are running on nodes with varying computational capabilities, and different kinds of messages might take different times to process. By decoupling producers and consumers, we solve both of the above-mentioned problems.

- Producers and consumers can fail (and restart) independently without impacting each other directly.

- By offloading many aspects of securely saving and efficiently delivering messages, it is possible to keep producer and consumer code simpler. (Though now we need a rock-solid intermediary service, Kafka.)
```

```
Question 3
What is one of the aspects of Kafka that enables scalability in terms of higher production and consumption rate?

Answer
The sharding of topics is the primary enabler to scale production and consumption throughput. Each shard of a topic can be deployed on a different node, and by carefully distributing producers and consumers to these portions, we can get high system throughput.
```

```
Question 4
What happens if a node holding a topic partition dies?

Answer
The partition data on the dead node will be unavailable for consumption. This data might be lost if the node does not come back online. However, if the partition was replicated on other nodes, there will be no data loss, and consumption and production to this specific partition will continue.
```

```
Question 5
Kafka producers, and consumers rely on the ZooKeeper service for different aspects of the system. What happens if the ZooKeeper service dies?

Answer
If the ZooKeeper service is not available, it will cause an outage for the Kafka service as well. The services we depend on need to have higher availability.

We might want to use two independent instances of the ZooKeeper to minimize downtime for our application. There are many details to ensure that the instances of ZooKeeper are consistent and work in harmony with each other. We highly encourage you to figure out the details of such a solution.
```
