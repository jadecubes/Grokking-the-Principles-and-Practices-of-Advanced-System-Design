# Ensuring High Availability in DynamoDB

Let's discuss some ways in which we can keep our database highly available.

## Write availability
Our database follows a model where tables are partitioned, and partitions are replicated. A set of replicas of the same partition is called a replication group. Partitions use a leader election based on Multi-Paxos to elect a leader replica. This leader replica is responsible for serving to write requests—it adds the new entry to the write-ahead log and indexes in its memory. Later, this leader shares its write-ahead log and tree index with other partitions in the replication group. This is possible if enough healthy replicas (two-thirds) are available to form a write quorum. Write consistency in our distributed database depends on the ability of the network to maintain healthy leader partition replicas and enough healthy replicas in the replication group to form a write quorum. A minimum quorum is also required to elect a leader.

One way to maintain write availability is for the leader replica to add another member to the replication group as a log replica as soon as it finds that some replica is unresponsive or faulty. This is the fastest way to maintain the minimum number of replicas required to form a quorum.

```
A log replica only stores the write-ahead log. Log replicas are like acceptors in Paxos.
```
[Write availability](./tx)

In the example above, we still have two-thirds of the nodes available when the node in replication group four becomes faulty. Does that mean the replication group has enough healthy nodes to maintain quorum? No. One of the nodes in the replication group is the leader, and the replicas that can form a quorum is just one. So, if one of them is down, the quorum is lost; half of the nodes are available. If a node in replication group one becomes faulty, the replication group can still form a quorum since two-thirds of the nodes that can form a quorum are available.

## Consistent read availability
Consistent read availability means we can always count on our service to return the value of the latest write to a read request.
```
Note: To learn more about consistency, visit Spectrum of Consistency Models lesson in Grokking System Design Interview for Engineers and Managers.
```
Our replication system provides eventual read consistency. Our system does this because, eventually, all replicas maintain the same entries as all members in their replication group. However, it is important to notice that only the leader replica provides instant consistent reads.


## Failure detection
A lot depends on the leader replica in a replication group. If a leader replica fails and another leader is elected, it cannot start accepting write requests until the old leader's lease expires (because it might be the case the old leader is not reachable to a majority but is accessible to a minority and we don't want two leaders in the system at any time).

Our system must ensure that a failed or faulty leader replica node is detected instantly. If the leader replica loses connection with all replication group members, then its failure is easy to detect. Gray network failures make it hard to detect a failure of the leader replica. A gray failure is defined as a failure in a component of a cloud network whose manifestations are fairly subtle, making it harder to detect. (Source: Gray Failure)

In our design, a gray failure can occur due to the following reasons:

1. There can be communication problems between the leader and the rest of the replication group.

2. There are issues with the communication (in and out) of a node.

3. The front-end routers or nodes may face communication issues with the leader while the leader's communication with the rest of the replication group is working fine.

In all these cases, there is a higher chance of a false positive in detecting the failure.

One solution for gray failures is to enable communication between replicas in the replication group such that they can confirm that the leader replica can communicate with the rest of the replication group. The follower replica can confirm from enough (a simple majority) of its fellow members of the replication group that communication with the leader is happening before deciding to start an election. If it cannot get confirmation from enough member replicas that they can communicate with the leader replica, then it will trigger a leader election.

[Failure detection](./detection)

## Tracking availability
To better track the availability of our designed system, we can have nodes outside of the replication network continuously measuring the availability at service and table levels. As soon as a table's or service's availability falls below the SLA issued to the customer, they trigger alarms that there could be a problem. We can have automated checks to generate a report as soon as an alarm is triggered. Such alarms are called customer-facing alarms (CFAs).

```
SLA: Service Level Agreement
```

We can set these alarms on the client side to measure the user's perceived availability.



## Deployments
Traditional means of deployment cause downtime. Our design needs to handle deployments without downtime. Deployments can also disrupt the service because they can also be faulty. While one way to reduce the impact of such deployments is to test the deployment thoroughly, we can also reduce their impact through other means.

We can deploy on a limited number of nodes before deploying on the entire network. In addition to CFAs, we can set other alarms for when performance is being degraded due to a deployment. These alarms can work with rollback procedures to ensure that a bad deployment never affects the system, at least a significant portion of our nodes. If a leader fails due to a faulty deployment, it can be programmed to relinquish its leadership to another candidate before its lease expires.



## Conclusion
In this chapter, we learned how an extensive cloud database is designed and operated to provide consistently good performance under dynamic load conditions. It was an exercise to see how the usual building blocks of system design can be used together to get the required results. It also shows us that for many real-world system design problems (and for interviews), we might not need to invent a new design—identifying the right building blocks and incremental tweaking should do the work.

### System design wisdom in DynamoDB
For many system design problems, the primary skill required of a designer is reducing problems at hand to well-known building blocks or other designs. Such reduction is similar to the reduction we have seen for algorithms. For example, if an algorithmic problem can be reduced to sorting, we know we have efficient algorithms for sorting, and we can then use them to solve our problem at hand. It saves us effort and dollar cost to reinvent the wheel and to build on top of what has already been accomplished.
