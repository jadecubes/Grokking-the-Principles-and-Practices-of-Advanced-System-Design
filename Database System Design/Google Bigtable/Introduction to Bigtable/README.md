# Introduction to Bigtable
## From the dark ages to a renaissance for the databases
With the advent of hyperscale services such as worldwide search, online shopping, messaging, and so on, the deficiencies of the traditional databases (based on the relational data model) became apparent. These deficiencies can be grouped into two classes—scalability challenges and performance challenges.

Traditional databases are optimized for read-heavy workload, where data schema is known at the time of writing and does not change too frequently. Additionally, most implementations of relation DB engines were either based on a single beefy server or a group of servers physically nearby. Such a setup is needed to rely on vertical scaling for improvements, though there are limits to such scaling. The workloads for applications were approaching the limits in terms of raw data size and available IOPS (input/output operations per second) with good throughput and latency from the database systems.

These deficiencies pushed organizations on a multi-decade quest to research and develop custom database systems. Primarily, the guiding rule was that for some specific applications, we might not need the full feature set of a relational model, and inventing a new, simpler model would enable us to get highly scalable and highly performant database systems. In this chapter, we will focus on one such system designed by Google, known as Bigtable.
```
Vertical scaling, also known as “scaling up,” refers to scaling by providing additional capabilities (for example, additional CPUs or RAM) to an existing device.
```

## The need for Bigtable
While traditional relational databases apply to many data problems, they are not suitable for important use cases concerning data-size scalability and read/write performance. Some of those use cases are:

- Fraud detection: It relies on rules for data detection algorithms, transaction information, customer information, time of day, location, etc., all of which are instantly applied on a big scale. For a common case, most of the data might not be read too frequently, but when needed, we might have to read most of it in near real-time. Such workloads are not suitable for traditional databases.

- Time-series data: This concerns data such as cumulative CPU and memory usage across several thousand servers of a data center.

- Marketing data: This concerns data such as client preferences and order history. The sheer number of customers and the quest to record fine-grained activity (like clicks) generates an enormous amount of data with a widely variable structure of data.

- Financial data: This concerns data such as transaction records, stock markets, and exchange rate changes. On the one hand, the volume of data is enormous, and it can vary (for example, when a stock exchange opens vs. when it is closed). On the other hand, such data might need to be read a few times in the future. Again, such workload patterns do not fit well with the relational model.

- Internet of Things (IoT) data: This concerns data such as statistics from household appliances and energy devices. The nature of IoT dictates the structure of the data schema. In a traditional table, we might need a different number of columns for different devices, and some columns might not be present for all devices. At times, such entities are called sparse tables. A conventional database will need to have every possible column for every row that impacts space utilization and query latencies.

Let’s elaborate how traditional database systems are deficient in terms of scalability and performance for the above use cases.


### Scalability
Scalability and flexibility are major challenges for relational databases. Relational databases were created during a time when data could be kept tiny, tidy, and organized. Relational databases are designed to operate on a single server to ensure the authenticity of the table mappings and to prevent the issues associated with distributed computing. Users must purchase larger, more sophisticated, and costlier proprietary machinery with additional processing capacity, memory, and space if a system has to scale under this design. Upgrades are particularly difficult since the organization must undergo a lengthy procurement procedure before taking the system offline to carry out the upgrade. All of this is happening as the number of clients keeps growing, putting greater strain and risk on the underprovisioned resources. It is tough to manage tables across several servers in relational databases.

```
The CAP theorem tells us why it is challenging to have a strongly consistent and highly available system under common faults such as network partitioning.
```

[Vertical scalability in a relational database]
### Performance
Relational databases are known to be slower due to the following reasons:

- The number of tables influences the performance of a relational database. The response time to queries will be slower as the quantity of tables increases (such as in multi-table joins).

- Furthermore, an increase in data slows down the system and makes it more difficult to collect the required information. Especially when data quantity becomes huge, the associated indices data structure can grow too large to provide consistently good performance.

- Relational databases frequently struggle with tables that have many columns or even fields with large volumes of data. Adding new columns in the table (and hence modifying the database schema) can be time-consuming. Sparse tables and variable-sized data (in a column) pose challenges for traditional databases in terms of efficient storage use and query times.

- Joins are constructed at query time in relational databases by matching primary and foreign keys of the numerous (possibly indexed) rows of the tables that are joining. These processes are computationally intense and memory heavy, with an exponential cost as the number of requests increases. For a sparse table (that is common in many real-world applications), a traditional database will first need to fetch all rows, and then it will do filtering to leave out the empty (null) values. Touching on a lot of data can drastically slow the read query times.

- Relational databases, like phone books that record contact numbers and addresses, are organized and have predetermined schemas. There is a procedure in relational databases that verifies that the added data matches the schema of the table, which takes time (different integrity checks). Non-relational databases, like file directories that store anything from a person’s constant information to their shopping preferences, are unstructured, scattered, and feature a dynamic schema.

[Relational databases vs. non-relational databases]

Here are some of the differences between relational and non-relational databases: (Source: MongoDB)

```
               Relational Databases vs. Non-relational Databases
Features                   Relational                       Non-relational

Availability                High	                           High

Scalability                  Low                             High

Performance                 Medium                           High

Flexibility                  Low	                           High	

Reliability                  High	                           Medium
```

Google wanted a database store for projects like web indexing, Google Earth, and Google Finance that could handle large amounts of data with high performance. These applications have significantly diverse needs in terms of data size and latency. Think about the challenges we would face if we were to create Google’s whole web search system using a MySQL database. Issues such as table and index size, redundancy, massively parallel simultaneous access may arise. Relational databases perform well until a certain point, but scaling out and sharding data to several computers becomes a problem.

Google created Bigtable, its own database store, for all these reasons. It also has a notion of tables, but they are not like relational databases. This is built to enable quick lookups and manage massive amounts of data. It is also intended as a wide-column store. This implies that each row in a table can hold a large number of columns, with the particular column names varying from row to row.
```
Bigtable is an example of a wide-column database that, at times, is also called a column family database. Read and writes are faster in wide-column databases due to indexing that is similar to a hashtable, which helps to reach a particular cell quickly and gives us the ability to write a newer version of a cell instead of changing the older one (which might have needed an exclusive lock). We should not confuse it with columnar databases, which primarily lay out data based on columns instead of rows to speed up a certain workload access pattern.
```

## Requirements
Let’s list the requirements of designing a distributed storage system for managing structured and semi-structured data.
```
If data schema is known at the time of data writing, it is well-structured data. If data schema is only known (or could be deciphered) at the time of reading, it is non-structured data.
```


### Functional requirements
The functional requirements of such a system are as follows:

- Wide applicability: We want to design a general-purpose data center database system that is applicable to many applications such as web indexing, online maps, and online financial applications. However, our system will not provide a full relational data model.

- High performance: We expect our system to provide high throughput and low latency as per the needs of the application. Due to the inherent tradeoff between high throughput and lower latency, it might not be possible to get the best of both for a single application. However, applications should be able to indicate if they prefer one over the other, and the system should honor it.

- User-guided locality: We want to design a system that can effectively take data locality into consideration because when data will be sharded, such data locality will increase the performance of certain queries. The system should enable the users to dictate which columns should be kept together for better locality in the schema.

- Continuous updates: Asynchronous processes should be able to update different pieces of data continuously.

  - Access most current data at any time.
  - Examine varying data, e.g., many web page crawls.
- Atomic rows: An operation either impacts the full row or none of the rows. Other than that, the system will not provide general transactions.

### Non-functional requirements
The non-functional requirements are as follows:

- Scalability: The system should scale up to terabytes of in-memory data and petabytes of disk data across thousands of commodity servers.

- High number of read/writes operations: The system shouldhave very high read/write rates (millions of operations per second). The latency of each read and write should be low compared to relational DBs.

- Availability: The system should be highly available. The system should operate uninterrupted despite server or component failures.

- Durability: The system should be highly durable. The data should be stored in stable storage so that even if all the power goes out, it can be restored.
## Bird’s eye view
Here’s a summary of this chapter. We will now dive into the design and evaluation of the Bigtable system.

[Overview](./overview.jpg)
