# No Fixed Schema in DynamoDB
Providing services to a wide range of customers that need database services will require a database design that can work for an equally wide range of use cases with easy integration via APIs. Providing services to a large number of customers requires scalability. Operating at a large scale requires a high-performance solution, especially because modern applications often have high throughput requirements. Also, to build a reliable service, high availability is crucial. Let's discuss how we can achieve these goals with our schema model.

For our design, we have chosen a NoSQL database because of its flexibility with highly functional APIs, easy scalability, performance at scale, and high availability. First, we will explain why we have chosen this system over the more traditional RDBMS (Relational Database Management System). Then, we will explore the table structure of our database.

## NoSQL instead of RDBMS

[RDBMS needs schema to be known a priori at the writing time so it can generate indexes. Traditional RDBMS fits the pattern of WORM (write-once read-many), and indexes make reading efficient. For NoSQL, schema can wait until the read time. Hence there is more flexibility for NoSQL solutions.](./sql.png)
In an RDBMS, data is stored in tables that are related to one another. NoSQL (not only SQL) describes any database with no relational structure of RDBMS. Our choice of NoSQL over RDBMS is motivated by the following reasons:

- Flexibility: The NoSQL model allows unstructured and semi-structured data, something RDBMS does not allow. Including these in our design allows for various use cases. Compared to an RDBMS, we can store data from multiple tables (of a normalized RDBMS) in a single document. This simplicity makes writing our API's codes easier, resulting in more functional APIs. For this reason, NoSQL allows flexibility.

- Scalability: As mentioned, data is stored in documents, not multiple tables, like an RDBMS. Scaling an RDBMS is harder. For example, scaling for a single entity in an entity relationship diagram would require scaling multiple tables. As for NoSQL, scaling would only require increasing the capacity to store more documents (much simpler than scaling multiple tables). Furthermore, in an RDBMS, performance is highly dependent on the underlying storage hardware, whereas NoSQL's performance depends less on the storage hardware (primarily because of the use of vertical scaling for RDBMS as compared to horizontal scaling for NoSQL solutions). NoSQL is preferred in distributed systems because of its ability to run databases on a large cluster. Using NoSQL based database allows us to easily scale our design by using more computation resources of a distributed network.

- Performance: The underlying data models in NoSQL (key-value, document, column, and graph) are optimized for performance. This allows us to maintain high performance at scale. The performance of a NoSQL database generally depends on the hardware cluster size.

- Availability: The simpler NoSQL model allows node replacement without downtime, resulting in higher availability. We are not claiming that RDBMS does not allow this, just that it is easier to achieve with NoSQL. NoSQL databases are easy to partition instead of an RDBMS. We will also maintain replicas of partitions of our tables. Since NoSQL data is easier to replicate and maintain, it allows for less downtime if a node fails because we can re-route the requests to a shard replica.

Later, we will see that partitioning our data is key in provisioning the throughput of our network to different tables.

```
Note: For a detailed comparison of RDBMS and NoSQL, visit Types of Databases lesson in Grokking Modern System Design Interview for Engineers and Managers.
```



## Table structure
In our NoSQL design, we will store data in tables, similar to other database systems where the table abstraction is built on top of the underlying key-value store. Every table is a collection of data. Our tables have data stored as items.

[Partition and sort key in action to locate required data row](./structure.png)


### Single item
Our tables can be empty, or have one or more items. A single item is a set of attributes that we can identify uniquely. We will identify an item using a primary key.

### Attributes
Every item comprises one or more attributes. An attribute is a unit of information that does not require further simplification. We can think of attributes as fundamental data types, such as a string or a number.

### Primary key
The primary key uniquely identifies items in tables. Its schema is configured at the time of table creation. The primary key can be one of the following:

1. A partition key

2. A partition key and a sort key

In both cases, inputting the partition key into a hash function returns the location where the item is stored. For the second case, there are two important things to keep in mind:

1. The sort key, along with the output of the hash function, is used to determine the item's location.

2. It is permited for a primary key to identify more than one item, however, such items cannot have the same sort key.
```
Note: We will provide more detail about how the primary key is used in the next lesson.
```
### Secondary indexes
Our design allows for querying a table using an alternative key in addition to querying with the primary key. This provides developers with programmable options to query a single table.


## API
Now that we have an understanding of our database system, let's look at the functions provided by our API.

- PutItem: Inserts an item when no item exists for the input key and replaces the item otherwise.

- UpdateItem: Updates an existing item if it exists for the input key. It adds a new item otherwise.

- DeleteItem: Given a primary key, this deletes the relevant item from the table.

- GetItem: Given a primary key, this returns the set of attributes for the relevant item.

## What's next?
Now that we know how our database system is structured, we will discuss partitioning and spreading it across nodes in a distributed network in the next lesson.
