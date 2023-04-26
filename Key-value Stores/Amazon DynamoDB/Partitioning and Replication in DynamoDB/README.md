# Partitioning and Replication in DynamoDB
The reason why we chose a NoSQL schema is that it allows for easy partitioning of tables. Let's quickly refresh our understanding of partition and learn how we will partition our tables. We will conclude this lesson by understanding how our design replicates partitions.

## Partitioning
As the name suggests, partitioning means dividing the database or a table and storing it in multiple nodes. This concept is also known as sharding. It is important to note that partitioning can be of two types, vertical and horizontal. We will briefly revisit both types in this lesson. Then, we will discuss how we will partition our design.

The purpose of partitioning is to distribute the load of read and write requests on several nodes. There are two ways to achieve this.


### Vertical
Vertical partitioning is the splitting of a table by columns. The illustration below demonstrates vertical sharding.


[Vertical partitioning splits tables by columns](./vertical.png)

In the example above, we have partitioned a table into two tables. Note how both tables have the same primary key.


### Horizontal
Horizontal partitioning is the partitioning of rows in a table. This is useful for large tables since it allows us to partition a table with many rows; the partitions of the table will have fewer rows. Different from vertical partitioning, all partitions will have the same number of rows. There are better ways to partition when the number of rows is expected to be large that is partitioning horizontally. Read-write access to a large table stored on a single server is limited to the throughput capabilities of that server. If we split the entries in the table equally and store them on two different servers, the same table will have higher availability. The illustration below demonstrates horizontal partitioning.

[Horizontal partitioning splits tables by rows](./horizontal.png)

Here, we can see that the resultant tables have the same schema. We've only split the entries in the original table into two tables with the same schema.

We will use horizontal sharding because our data will not have a fixed schema. Furthermore, another reason to choose horizontal over vertical partitioning is that the former is better for our design. We are expecting a vast number of rows in our tables, and we wish to distribute the throughput of our nodes across those entries. Usually, automating horizontal sharding is much easier as compared to vertical sharding—achieving a fair distribution of throughput among partitions with vertical sharding requires knowing how frequently columns are accessed.

Note: For a detailed explanation of partitioning, visit Data Partitioning lesson in Grokking Modern System Design Interview for Engineers and Managers.


### Our partitioning
We will horizontally divide every table in our database into partitions. This will help us cater to the different throughput and storage requirements of partitions. We can think of a partition as an allocation of storage that is backed by SSDs. Every partition will host a disjoint part of the table's key range. Our design will increase the number of partitions of a table in the following scenarios:

1. If the user changes a table's provisioned throughput to make it more than what can be accommodated with the existing number of partitions.

2. If an existing partition fills up or requires higher throughput or resources.

Our design will carry out partition management, creating a hassle-free experience for the user. We will discuss this in detail in the next lesson.

### Accessing partitions
As mentioned before, the primary key for a table can have one of two schemas: only a partition key or a partition key with a sort key. Whenever we wish to determine the location of an item, whether it is for inserting, updating, looking up, or deleting an item, we will first input the partition key into the internal hash function. The output of the internal hash function is a partition. The process to determine the partition for an item is the same for both schemas: entering the partition key into the internal hash function.
```
Note: For both schemas, the primary key is unique. The primary key is used to identify items uniquely.
```

#### Partition key (only) schema
For the first schema, the internal hash function provides us with the partition in which we will store the item for inserts. Within the partition, we will store items ordered by partition key—the sorting order of items can be ascending or descending depending on the user's request. The internal hash function provides the partition where the item is present (if stored) for updates, deletes, or lookups. We can use the partition key to look for the item in a partition since we have sorted items by partition key inside the partition. To summarize, for the first schema, the hash value of the partition key determines the item's partition, and we can use the partition key to search for the item in the partition.
```
first schema: partition key only
```

```
In the schema below, items cannot have the same partition key.
```

[The partition key in this example is "ClothesType". It is used to determine the partition that the item belongs to using the hash function.](./partitionkey.png)

#### Composite key schema: partition key and sort key
For the second schema, it is permitted for a primary key to identify more than one item. Our design stores such items in the same partition in ascending or descending order by sort key and physically close to each other. Such items must have different sort keys.

For inserts, we will store the item in the partition provided by the internal hash function and order them by partition key; we will store items with the same partition key ordered by sort key.

We will look inside the partition given to us by the internal hash function for updates, deletes, or lookups. Inside the partition, we will first look for the item by partition key since we have sorted items using the partition key. We will use the sort key to decide between items with the same partition key. The operation will happen on the item where both the partition key and sort key values match.

Let’s summarize the second schema as follows:

1. The partition key determines the partition where the item is stored or will be stored.

2. The partition key gives us the position of the item inside the partition.

3. The sort key helps us find the item from a set of items with the same partition key.
```
Note: For updates, lookups, and deletes, in a table with the second schema, both the partition key and sort key from the function arguments must match the values in the item for the operation to be carried out on the item.
```

[The partition key in this example is "ClothesType," and the sort key is "Size." Only the partition key is used to determine the partition the item belongs to; the sort key is not used yet. The sort key is used to determine the position of the item among other items with the same partition key.](./composite.png)

## Replication
Modern cloud service providers have data centers dispersed around the world. These have been clumped together in zones since they provide similar latency to the same user. Our design needs replicas in each availability zone to take advantage of these zones for high availability and durability. Replication allows to cater to the different throughput and storage requirements of tables, even partitions. Furthermore, we require replicas for partitions that require frequent read access. A single replica in a zone might be overwhelmed by the requests from that zone.
```
Note: Replicating based on partitions is more useful than replicating full tables since it allows control over having different resources allocated to different partitions of the same table. Replication based on tables will result in replicating all partitions of a table, potentially underutilizing resources since all table partitions might not require the extra resources.
```
We will replicate based on partitions. Every partition will have multiple replicas spread throughout the global network. The replicas of a single partition are called a replication group. Our replication groups will maintain consistent replicas worldwide using an algorithm called Multi-Paxos. However, before we explain the algorithm, let's understand the problem it solves.

### Consistency
In distributed systems, especially with data replication, maintaining consistency and consensus is a difficult problem: getting all nodes to agree on a specific thing. In our case, this is the state of our database's partition. We require replication not only across availability zones but even inside an availability zone. We want that all replicas of a partition have the same copy.

The problem above is a consensus problem. Let's formally define the consensus problem. We will also contextualize the consensus problem with our situation to better understand the problem we are facing here.


#### Consensus problem
A consensus problem is as follows:

1. We have a set of k nodes: n1, n2,...,nk. These are servers in a distributed network that store a partition. Collectively, we can call them a replication group.

2. Each of these nodes can propose a value vi to other nodes. In our case, this is the state of our database partition.

3. Every node must respond to a proposed value, either agree that it is the correct value, or return a different value proposed by another node or the node itself. Our servers in a replication group must do the same for the state of the partition. This property of a consensus is called termination.

4. By the end of the algorithm, all nodes must have agreed to the same value v, the consensus. We must end up with one partition state that is accepted by all servers. This property of a consensus is called agreement.

5. A node in the set must have proposed the agreed value. To reach an agreement, every server in the network must agree to a partition proposed by a replication group member. This property of a consensus is called validity.


#### Multi-Paxos
Multi-Paxos is one of the most popular algorithms for solving a consensus problem. We will skip the original Multi-Paxos algorithm and explain the salient features of our implementation instead. We will use a leader-election implementation of Multi-Paxos, where all our nodes (members of the replication group) carry out all three roles in the algorithm—proposer, acceptor, and learner.

[Multi-Paxos](./mp)

In the example above, nodes play different roles: leader, acceptor, and learner. We have done this for simplicity. In reality, a node can play all three roles. At any one point, a replication group has one leader. All the rest of the nodes are both acceptors and learners.

#### Election
Any replica (a node member of the replication group) can trigger an election to elect a leader. The leader replica serves to write requests and strongly-consistent read requests. The leader replica will remain the leader if it keeps renewing its leadership lease. A non-leader replica will trigger an election if it considers the leader replica faulty, unhealthy, or unavailable. The new leader will start serving writes and consistent reads after the previous leader's lease expires.


#### Write requests
Here's how a write request is processed:

1. The leader replica receives the request.

2. It generates a write-ahead log record and sends it to other replicas in the replication group.

3. When a quorum of replicas accepts the log record to their local write-ahead logs, our design acknowledges that the write request is processed.

#### Consistent reads
Only the leader replica can serve instant consistent reads. Other replicas of the replication group will eventually be able to serve consistent reads.


## What's next?
In the next lesson, we will understand how our replication model enables smooth adaptation to customer traffic patterns when provisioning throughput to a table or partition.
