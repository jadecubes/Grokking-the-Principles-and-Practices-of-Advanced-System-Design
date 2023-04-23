# Quiz on DynamoDB

```
Question 1
How does our design take advantage of the multi-tenant architecture?

Answer
The multi-tenant architecture allows us to use our resources efficiently. Dedicating one node for one table could result in underutilization if the table contains a limited amount of data. To scale the table, we might need to dedicate another node if the first node cannot store all the table’s data. One node dedicated to the table is utilized well, while the other could be underutilized. The multi-tenant architecture allows us to spread our tables across our network as partitions, thus spreading the potential effect of underutilizing dedicated resources.

Multi-tenant architecture also enables easy scaling since we can allocate and deallocate resources at the partition level.
```

```
Question 2
Why have we chosen a NoSQL database over a relational database?

Answer
If an application can utilize a NoSQL solution well, it is more efficient and available than traditional databases (which are harder to scale over geographically dispersed sites).

NoSQL allows unstructured and semi-structured data, allowing a wider range of use cases.
We can store data from multiple tables of an RDBMS in a single NoSQL document.
NoSQL is easier to scale than an RDBMS.
NoSQL data models are optimized for performance, especially when replicas are far away.
The NoSQL model allows node replacement without downtime.
```

```
Question 3
What problem occurs during the uniform distribution of provisioned throughput for a table across partitions?

Answer
In practice, keys are not accessed uniformly. This could result in some partitions having underutilized throughput, while some may face a shortage of throughput.
```

```
Question 4
What is an important factor to consider when bursting a partition?

Answer
Bursting should only take place when there is workload isolation between co-resident partitions. This means that allowing the bursting partition to utilize extra throughput should not impact other tenants (who might be residing on the same physical node).
```

```
Question 5
Assume that a tenant workload is fluctuating heavily across many keys. This means that while throughput provision is enough overall, it is not enough for specific partitions on specific nodes.

How can we manage such a case in our system?

Answer
First, we need to detect such pathological cases where spikes on a table partition are moving around very fast, and there is no fixed pattern of access.

We might put multiple partitions of this specific tenant on the same nodes so that as spikes move around, the same local node could serve it (instead of moving the partition around). Additionally, doing so isolates other tenants from such access patterns.

This might be a case where our system alone can’t solve the problem, and we need cooperation from the client’s application. Altering the application’s access pattern might be one strategy. If nothing works, the client might need over-provisioning of resources.

We encourage you to think about how such a use case can be managed (remember that there are always multiple ways to solve the same problem, and as designers, we should be able to evaluate them).
```
