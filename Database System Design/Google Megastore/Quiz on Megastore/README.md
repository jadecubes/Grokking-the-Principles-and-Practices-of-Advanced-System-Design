# Quiz on Megastore

```
Question 1
What is the consistency model of Google Megastore?

Answer
Megastore provides different consistency models for different use cases. For example, transactions inside an entity group are possible with strong consistency guarantees and ACID semantics. However, the system does not provide transactions with ACID semantics across entity groups. Therefore, it prioritizes availability over consistency overall.
```

```
Question 2
How does Megastore ensure ACID transactions?

Answer
Within an entity group, ACID transactions are processed in a single phase, while a two-phase commit is used for atomic modifications between entity groups.
```

```
Question 3
Is Megastore a relational database?

Answer
No, Megastore offers a semi-relational data model. It employs Bigtable as its storage subsystem and combines the abstract tuples of an RDBMS with the column storage of NoSQL.

Tables in a schema hold entities, and those entities have their properties. Values in a property have both a name and a type. The entity’s primary key is an ordered set of properties. Entities that will be read together are grouped under the same key.
```

```
Question 4
Can the applications using Megastore control latency and throughput?

Answer
Bigtable is Megastore’s go-to scalable datastore. Applications can manage latency and throughput by selecting Bigtable instances and specifying the locality inside an instance. Moreover, applications typically store original data near users and keep replicas close to one another to reduce waiting times.
```

```
Question 5
How large (or small) should a partition be in Megastore?

Answer
The appropriate size of an entity group depends on a specific use case. We would want to keep all those rows inside the boundary of an entity group on which we need to run transactions frequently.

If we make a large entity group, availability can suffer due to Paxos’ liveness issues. If we make an entity group too small, we will need across-partition transactions, and Megastore does not provide strong guarantees there. Therefore, the right partition size needs to be thought through and evaluated experimentally. Over time shard rebalancing will be required as well.

It is a good exercise to think about what will happen if we put all the rows in just one partition (we know in reality that will not be possible because each node has an upper bound on the amount of data it can hold, but let’s assume it is possible), or if we put just one row in one entity group?

To drive the idea home, we will encourage you to think about some of your use cases and how you will make the entity groups for them for good performance, fault tolerance, and availability. You should have good arguments to support your decision about the entity groups. If you don’t have any use case of yours, we have a question that will help you think along these lines.

How will you construct entity groups for a large online retailer that sells different products to customers who put them in a shopping cart and then pay the balance due? First, what should be in the entity groups, and how should we shard the data? Secondly, how big should each shard be?
```
