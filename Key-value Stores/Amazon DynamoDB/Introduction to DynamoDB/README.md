# Introduction to DynamoDB

Before the cloud, every company was responsible for managing its database services. This required hiring a wide range of specialists. Now, companies can pay for managed databases that delegate the necessary operations for setting up, maintaining, and scaling to the cloud service.

How do such databases work? As of September 2022, the global tech industry is worth 5.2 trillion USD. If the aim is to provide a database service to such a huge market, how can we design a system that can achieve this?

Before we design such a system, we need to understand what we require from the system.
```
Source: https://www.zippia.com/advice/tech-industry-statistics/
```

## Design requirements
Keeping in mind that we need a database that can provide services to the entire tech industry, we will require the following from our design.

### Functional requirements
Let's start with listing our functional requirements.


#### Managed service over the cloud
We will deploy our service on the cloud. We will assume that we have high-tech cloud data centers that professionals manage. This frees our users from the hassle of managing any of the following: bug fixing, hardware management, and managing distributed services. Our service will handle resource provisioning, failure management, software upgrades, and backups.

We’ll only provide our users with an API. Users can create tables and read and write data without considering the location of their tables or who manages the service and how they manage it.

#### Multi-tenant architecture
In multi-tenant architecture, different customers will share resources, for example, sharing a single node. We will not have a single node dedicated for a single customer as we see in single-tenant architecture. This does not mean customers sharing a node will only use that node. A single customer can use multiple nodes that other customers can also use.

Since we are managing a database, our system comprises data in tables. Later ,we will see that our design splits tables into partitions, and stores these partitions on different hosts. As a result, a single host stores partitions of several tables.

The multi-tenant architecture explained above ensures high utilization of resources. We’ll also see that the multi-tenant approach provides flexibility for resource provisioning—we can allocate or deallocate resources on the partition level. It also allows us to pass the costs we saved (by efficient resource provisioning) to our users.

### Non-functional requirements
Now, let’s list our non-functional requirements.


#### Easy horizontal scaling
We'll provide our users with easy horizontal scaling according to their needs. Our design will not assume any predetermined limit to the volume of data a table can store. Tables will be able to scale elastically to meet the demands of our users' applications.

The multi-tenant architecture will help us achieve this elasticity. Since we'll spread tables across multiple hosts, their maximum size is not dependent on a single host's resources. A table's partition will take up as much of a host's resources as it requires, and the table will have as many partitions across hosts as it requires resulting in high elastic scalability.

#### Predictable performance
To design a reliable service that many consumers can adopt, we need to offer a widely accepted level of performance. In most cases, databases are the base of an application (the backend). Any latency in the database adds to the latency of the entire application. Ideally, our design should respond to requests within single-digit milliseconds. We will ensure that our design returns the API's get and put requests with consistently low latency, granted that the data is present in the same region as the application.

We will design our service to exhibit predictable performance to achieve the above. Predictable performance means the size of a table or database or varying traffic patterns will not affect the service's performance (latency). We will ensure this through dynamic throughput provisioning.

```
dynamic throughput provisioning: Our model will adjust provisioned throughput for a table or its partition with changing requirements automatically in real time
```

```
Note: Traditionally, services had reported median latencies but tail latencies are also very important. We aim for single-digit latency even at the tail so that applications building on this system can provide good SLAs.
```

```
Tail latency represents the latency of requests that take the most time, out of a set of requests handled by a server. For example, P95 tail latency is the latency of requests that take longer than 95 percent of the requests handled by a server.
```

```
SLA: Service Level Agreements
```

#### High availability
Modern cloud services are available in different zones across the globe. We need to ensure that our service can replicate data to be available in all Availability Zones. Within zones, our service needs to replicate tables on different nodes in case of a node failure. We will aim for an SLA of five nines—99.999—for global tables (replicated across zones) and four nines—99.99—for local tables (replicated within zones).



#### Flexible use cases

Catering to a wide market requires our service to be adaptable to an equally wide range of requirements. We will not force our users into a data model. Our service doesn’t have a fixed schema. Instead, our data can have varying types. It can also have attributes with more than one value. The data model our tables will use can be any NoSQL data model—documents or key-value. Our customers will have the option to select strong consistency for their tables. If not selected, they can choose to have eventual consistency.

## Bird's eye view

The following concept map summarizes our work in this chapter. In the next lesson, we will start designing our system, which is called DynamoDB. We will explore the high-level design of our service and list its characteristics. We will explain these characteristics from a system design perspective.

[Overview](./overview.png)
