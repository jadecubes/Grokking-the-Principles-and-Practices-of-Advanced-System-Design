# Delivery Guarantees of Kafka
Reliability in data delivery is as important as the performance of the system. The guarantees that Kafka provides are explained below.

## At-least-once delivery
Kafka, by default, provides a guarantee of at-least-once delivery.

[At-least-once delivery](./delievery.png)

Kafka delivers each message exactly once to each consumer group. However, if a consumer breaks down, the rebalancing occurs, and another consumer takes over the partitions owned by the failed consumer. The new consumer fetches the last offset that was successfully saved in the offset registry. Let’s assume that the offset of the last message consumed by the failed consumer is greater than the last offset that is saved in the offset registry. This causes the new consumer to consume messages that were already consumed by the failed consumer, causing duplication in consumption.

[Reconsumption of messages](./reconsumption.png)

In the illustration above, we can see that the last offset saved in the offset registry by the failed consumer was for the third message. However, it failed while calculating the new offset to start the consumption of messages 6 to 9, which are ahead of the saved offset, meaning that they have already consumed messages 3 to 5. The consumer that will replace this consumer checks the offset registry. It also finds the offset of the third message and starts consuming messages from that point, which have already been consumed. This causes replication in consumed messages, which creates the possibility of a more-than-once delivery of a message. If this whole process is avoided, the more-than-once delivery of a message can be avoided.

Another scenario in which a consumer may consume a record multiple times is that it read messages 3 to 6 and then failed to write just the fifth message in the consumer. In this scenario, the consumer cannot update the offset to the sixth message because that will result in the fifth message never being consumed. In this scenario, the consumer updates the offset to the last successfully written message before the failed message and sends the next pull request with that offset, and ends up consuming the sixth record again.

## Exactly once delivery
If duplicates cause problems for an application, it is possible to achieve an exactly-once delivery through some deduplication logic on the consumers. The following techniques can be deployed at the consumers to make an exactly-once delivery possible in Kafka:

- We can use unique keys in the messages and write the results to a system that supports these keys. This is usually done by writing the unique IDs of consumed records as keys and records themselves as their values in a system. If a message whose key already exists in the saved results is consumed again, it can be discarded, or the existing message can be overwritten. These kinds of write operations are known as idempotent writes.

- For transactional systems, there is another option. The record and its offset can be written in the same transaction so that the saved offset is saved with the consumption of each message and does not lag behind the consumption process (this technique will only work in cases where offsets could not be updated due to failed consumers).

## In-order delivery
Kafka guarantees that messages delivered through a partition in a specific order will be received at the consumer in that specific order. This means that if the producer sends messages in order, and the broker writes them all in a single partition, the consumers will get those messages in the same order the producer sent them in. However, if the messages are written randomly to multiple partitions, then the order of the delivery of messages to a consumer is not guaranteed.

## Fault tolerance
Let’s go through some processes of Kafka that help make sure it achieves its promised delivery guarantees.

### Avoid log corruption
Kafka keeps a cyclic redundancy check (CRC) for all the messages in the created log (to keep important updates and segment information) to help cater to log corruption. These CRCs help with two types of errors:

In case of any I/O errors on the broker, Kafka handles them by removing the messages with inconsistent CRCs.

Saving a CRC for each message allows us to check any network errors that happen during the production and consumption of messages.

In case of any broker failures, all the messages saved on it are no longer available, be they consumed or unconsumed, which adversely affects the delivery guarantees. Kafka employs a replication mechanism to avoid loss of messages in case of broker failures.

```
A cyclic redundancy check is a short check value for each piece of data, used for detecting error at the receiving end.
```

### Replication
Partitions are the most basic unit of parallelism in Kafka. So, partitions need to be replicated to cater to data loss, and that’s exactly what Kafka does. It can replicate partitions multiple times and designate one of them as a leader. The producers producing messages and the consumers consuming messages of certain partitions, which are replicas of each other, need to do production and consumption from the leader partition. All other partitions need to be in sync with the leader replica, so that whenever a broker with a leader partition is lost, another partition that was in sync with the leader partition can be made the leader replica.

```
Question
What are in-sync replicas?

Hide Answer
The replicas that do not lag in the updates that are happening in the leader partition are called in-sync replicas. They can be identified by the following two characteristics:

- Replica partitions that sent a heartbeat to the ZooKeeper in the last 6 seconds (can be configured by the user).

- Replica partitions that fetched messages from the leader in the last 10 seconds (can be configured by the user).

In contrast, the replicas that do not have the most recent messages that the leader partition has are called out-of-sync replicas. Usually, we use synchronous replication to keep all the replicas consistent.
```

The in-sync replicas that are lagging a bit slow down the production and consumption of messages. This is because producers and consumers wait for all in-sync replicas to get the recent messages for a configurable amount of time after which the replica is just considered as out-of-sync and they do not wait for out-of-sync replicas to get in sync. Therefore, they do not let them slow down their consumption. The amount of in-sync replicas is critical because the fewer they are, the greater the risk of data loss. If the producers and consumers do not wait for the in-sync replicas for a certain amount of time and keep producing and consuming messages, they will become out-of-sync and if all replicas become out-of-sync and we lose the broker with the leader partition. Kafka will have to choose an in-sync replica for a leader but such a replica will not be there, and hence loss of data will occur. The out-of-sync replicas can get in sync whenever they get all the recent messages.

In this lesson, we learned how Kafka guarantees provides different message delivery guarantees and how different kinds of failures can complicate the process.
