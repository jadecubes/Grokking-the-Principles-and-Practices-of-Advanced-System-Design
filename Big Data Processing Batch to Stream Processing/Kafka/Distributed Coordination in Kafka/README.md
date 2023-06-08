# Distributed Coordination in Kafka
There are three ways a producer can publish messages to partitions. It can either publish a message to a particular partition, a random partition, or by selecting a partition by applying a partition function on a message key. In this lesson, we’ll see how the consumer will interact with multiple brokers in parallel.

## Coordination in consumer groups
Certain features are there in Kafka to help achieve scalability goals. Those features are explained below.

***Goals of coordination***

1. Evenly distribute the messages (stored in the brokers) across multiple consumers.

2. Concede the lowest possible coordination overhead.
### Consumption of messages
Partitions are made to be the smallest unit of parallelism in Kafka. Each smallest unit of parallelism needs to be consumed only by a single consumer. This means only one consumer consumes all the messages in a specific partition of a topic.

If multiple consumers were allowed to consume a partition, they would have had to communicate and decide which messages would be consumed by whom, incurring a locking and state maintenance overhead.

[Communication between consumers while consumption of messages](./comm.png)

In Kafka, consumers need to coordinate only when a consumer fails, and all the other consumers need to rebalance the load. To balance the load with fewer consumers, one thing that can be done is over-partitioning a topic and assigning multiple partitions to each consumer.

### No master node
When there is a master node in a system, it must have a mechanism employed that can handle master failures. To avoid these circumstances, Kafka employs no master node. It has decentralized decision-making, which lets consumers coordinate with each other. For the coordination of consumers, it uses a service called ZooKeeper.

## ZooKeeper
ZooKeeper is a highly dependable service that helps manage the coordination of machines in a large cluster. It is a highly reliable source that facilitates coordination between machines in a large cluster. It has a very simple programming interface. ZooKeeper has data registers that are referred to as nodes.

Its API allows the following functionalities:

- Create a node.

- Delete a node.

- Set the value of the node.

- Get the value of the node.

- Get the children of the node.

- Create ephemeral nodes that get removed when their creating client is gone.

- Replicate data across multiple servers, making it highly reliable and available.

- Register a watcher on a node notifying whenever children or the value of a path has changed.

## ZooKeeper in Kafka
The ZooKeeper performs the following tasks in Kafka:

- Detect the addition or removal of consumers and brokers.

- Register consumers and brokers by creating ephemeral nodes.

    - All the brokers try to create an ephemeral node called a controller, but only the first broker that starts in a cluster succeeds. Others receive a "node already exists" exception.

    - When a broker fails, its node is removed. If the controller is lost, other brokers are notified, and they attempt to create a controller node for themselves. The first one to do so is the new controller.

- Initiate a load rebalance process in case brokers or consumers are added or removed.

- Maintain the consumption from brokers and track the offset of the last consumed messages of a partition.

[ZooKeeper in Kafka](./zookeeper.png)

```
Note: Recently, there have been efforts to make Kafka independent of ZooKeeper.
```

### Registries
ZooKeeper has some registries where it keeps different types of information. Brokers and consumers create nodes to save a meaningful piece of information as their value.

Consumer registry: A consumer saves its information in the consumer registry whenever it starts. The information includes the consumer group it belongs to and its subscribed topics.

Broker registry: A broker saves its information in the broker registry whenever it starts. The information includes the hostname, the port of the broker, and the set of partitions stored in it.

Ownership registry: It has the information related to each subscribed partition in a broker. The information includes the consumer ID that is consuming messages from a partition. So, it tells which partition is being consumed by which consumer.

Offset registry: The offset registry also stores information related to each subscribed partition. The information includes just the offset of the last message consumed in a partition.

No offsets are available for the newly created consumer groups. The consumers have to start from the smallest or the largest offset of the messages in the partition.

### Persistence of nodes
There are two types of nodes—ephemeral and persistent. The nodes in the broker, consumer, and ownership registry are ephemeral, and the nodes in the offset registry are persistent. Ephemeral nodes get removed whenever their creator is down.

- In case of a broker failure, all of its partitions saved in it are lost, resulting in loss of information from the broker registry.

- In case of a consumer failure, the information nodes it had in the consumer registry is removed as well as the information regarding the partitions it owned in the ownership registry.

### ZooKeeper watcher
A broker and consumer registry can also have a watcher registered on them by each consumer or by each broker. The watcher notifies about the changes that occur in the broker and consumer registry.

- A change in the broker registry occurs when a topic that a consumer is consuming is modified.

- A change in the consumer registry occurs whenever a consumer fails, or a new consumer is added.

The watcher can also trigger a load rebalance whenever a new consumer is initialized or when a change occurs in the broker or consumer.


#### Rebalance process
The process of changing the ownership of a partition is called a rebalance. The rebalance process determines the partitions a consumer will be consuming. It is a crucial process because it allows the addition and removal of consumers ensuring high availability and scalability at consumer groups. However, it is not a desirable operation. It should not occur unnecessarily because it stops the consumption of messages for a short period and also causes the consumers—who are redefining their ownerships—to refresh their caches. This slows down the consumption even more.

[Process](./rebalance)

The rebalance process shown in the slides above is described as follows:

- Whenever a new consumer is introduced or an existing consumer is notified about any changes in the broker or consumer registry by the ZooKeeper watcher, it (ZooKeeper) initiates the process of rebalancing the consumption of partitions.

- Each consumer reads the ownership registry and removes the partitions owned by it.

- Each consumer reads the consumer and the broker registry and gets the set of partitions (P) in subscribed topic T available for consumption by set consumers C.

- Each consumer then range partitions P into C number of chunks.

- Each consumer deterministically picks a chunk of partitions for consumption.

- Now that the consumer is subscribed to a chunk of partitions, it updates the ownership registry by adding itself as the owner of these partitions.

- Each consumer gets the offset of the last consumed messages in those partitions from the offset registry.

- Each consumer starts consuming messages from those partitions starting from the message after the last consumed message.

##### Complexities in rebalancing
A consumer group is often composed of more than one consumer. When a change occurs in the broker or consumer registry, the watcher notifies each consumer in the consumer group. However, the timing at which they are notified can differ. So, let's say consumer A gets the notification first and initiates the rebalance process first, and takes ownership of a specific new set of partitions, let's say P1 and P2. Furthermore, while this process was going on, consumer B got the notification and started the rebalancing process by checking the ownership registry, and until that point, P1 and P2 had no owner. At the end of this process, the watcher ends up taking P2’s ownership. However, P2 already has an owner at that point, which is Consumer A.

```
Question
How does Kafka make sure that only one consumer owns a certain partition?

Answer
Kafka relies on ZooKeeper to track consumers per partition. Whenever a Kafka consumer is added to a consumer group, it triggers a rebalance process explained above.

Whenever a consumer gets a specific set of partitions from the rebalancing process, which are already owned by another consumer, it is made to drop all the partitions it tried to own and try the rebalance process again after waiting for some time. It is a complicated process, but the rebalancing stabilizes after a few tries.
```

In this lesson, we learned how Kafka keeps track of changes that are happening in the brokers and consumers by using ZooKeeper and how it helps in case of broker and consumer failures.
