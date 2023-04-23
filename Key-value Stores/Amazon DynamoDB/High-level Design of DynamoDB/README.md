# High-level Design of DynamoDB
There are numerous components in the system, but we will focus on some salient ones. In this chapter, our objective in this chapter is to understand different design trade-offs while meeting the needs we stated earlier. Before we get into the details, we must develop a high-level understanding of our design. This will help us understand how our design meets our goals before explaining the specifics.
```
Note: DynamoDB is a feature-rich database and it is not our goal to cover all of its details here. We will discuss some of the most important design aspects.
```
[High-level architecture of the system]

The illustration above gives some details about our design. We will explain it in the coming lessons. Our design will have the following key characteristics.

## No fixed schema
We aim to provide a service that caters to all use cases with easy horizontal scaling. We will allow our database to function without a set schema. This gives users easy access to data and they do not need to declare a schema before using the database. It also keeps our design simple. We'll be using a NoSQL-based database system.

[No fixed schema required]

## Partitioning tables across nodes
We have decided to use a multi-tenant architecture to better utilize resources in a distributed network. To do this, we will partition our database based on tables so multiple nodes can host them. Later, we will see that this helps us achieve horizontal scalability and high availability. We will discuss the partitioning of our tables and the replication of partitions across our distributed network in a later lesson.

[A multi-tenant architecture]

## Automated adaption to traffic patterns
We will discuss means to avoid underutilization and shortage of resources for tables. Our design will adapt to customer needs by making decisions to allocate and deallocate resources (that also affect throughput). The design will make these decisions by observing usage statistics. This will help to keep our service highly available and maintain predictable performance.

[A table partition experiencing high load (left figure) is split and distributed to different nodes (right figure) to meet throughput demands]

## Durability and correctness
We will also ensure that our design is fault tolerant such that committed data is verified and backed up. This is essential to build a reputable service. We will discuss several methods on how to achieve this. These will include dealing with hardware failures and silent data errors, continuous verification of data, reducing the impact and occurrence of software bugs, and regularly backing up our data.

[Avoiding data loss and inconsistencies]

## High availability
We will primarily rely on data replication for higher availability. Tables (or their shards) can be replicated either within the same data center or at many different data centers. While using dispersed data centers provides better availability, it can incur higher latency and higher dollar costs. We will also discuss keeping our service available during maintenance events.

[High availability is crucial to build a reliable service]

## What's next?
Next, we will explore the characteristics above to develop a deeper understanding of our design.
