# Data Model of Bigtable
A data model and an associated API are the cornerstones of any database. In this lesson, we will learn how Bigtable uses key-value stores to provide an abstraction of a table and associated table and data manipulation operations.

Bigtable is a sparse, distributed, persistent multi-dimensional sorted map. In traditional databases, we have two-dimensional layouts. Each cell is determined by a row ID and column name. On the other hand, Bigtable has the following four dimensions.
```
Sparse: It does not need to store an entry in every cell (row/column intersection).
Distributed: The table will be on many physical servers via sharding.
Persistent: Data we store will persist even after our session ends.
Multi-dimensional sorted map: A map essentially means a key/value store. The multidimensional sorted map is like a sorted map of maps.
```

1. Row key: It uniquely determines the row.
2. Column family: This depicts a group of columns.
3. Column name: It uniquely determines the column.
4. Timestamp: The columns can have different versions of a value uniquely determined by timestamps.

[Dimensions of Bigtable]

A row key, column key, and timestamp are used to index the map. All map’s values are uninterpreted arrays of bytes. Bigtable treats all data as raw byte strings.
```
(row: string, column: string, time: int64) → string
```

```
Google designed Bigtable to store large amounts of data efficiently. For instance, if we are storing the web page, then the row key will be a URL e.g., educative.io. The column key says that we’re going to store the contents of the webpage at that URL. The timestamp is just possibly the time at which we crawled the web and fetched that webpage. The value is the content of the web page. Now, this is a three-dimensional table. If we’ve crawled the web at previous times, we’ll just leave the older versions in the table.
```

## Rows
All rows in Bigtable have an associated row key, an arbitrary string up to 64 KB in size (most of the keys are much smaller than 64 KB). Each write and read of the data in a distinct row key is atomic regardless of the columns being read to avoid inconsistencies.
```
An atomic operation is an indivisible and irreducible series of database operations such that either all occurs, or nothing occurs. (Source: Wikipedia)
```
Bigtable keeps the data in lexicographic order by row key e.g., 5 > 10 lexicographically, yet 10 > 05. Adding a leading zero to the 5 guarantees that the numbers are sorted numerically. (Bigtable keeps the rows sorted lexicographically so that keys that are the same or near to each other can be stored physically nearby on storage for better query results.) The row range of a table is dynamically sharded into blocks referred to as a tablet, which is the distribution and load balancing unit. As a result, reads of small row ranges are fast and can usually be served from a single server.


## Columns
The following syntax is used to define a column key: family:qualifier. In the example below, we’ve used theUserdetails:Name column key, where Userdetails is the column family and Name is the qualifier. Column family names must be printable (so that the Bigtable system can use them, for example for the locality, by keeping members of a column family nearby), but qualifiers could be any string. Access control, disk, and memory accounting are handled at the column-family level.

[Data model]

## Column families
The column keys are organized into sets called column families. These column families constitute the primary unit of access control. Normally, columns that are related to each otherare combined into a column family for fast reading and writing.

Typically, all the data recorded in a column family belongs to the same type. The column family must be constructed before inserting the data under any column key in that family. Any column key inside the family can be utilized after it has been generated. A table should have a limited number of different column families (no more than a few hundred), and families should rarely change while the table is being used. However, there can be an unlimited number of columns in a table.
```
Bigtable tries to keep the columns of a column family nearby because they are likely to be used together in queries. Such nearness enables faster reads/writes compared to not using nearby optimizations. Additionally, the constraints on the number of column families or columns in a family usually arise from the fact that there are so many things that can be put nearby.
```
## Timestamps
Different versions of the data can be stored in each column cell. Storing numerous cells in a column allows us to keep track of how the data in that row and column has evolved over time. As we saw in the last example, we may have many timestamped variants of a user’s email. Each version is identified by a 64-bit timestamp, which reflects “real-time” in microseconds or a custom number supplied by the client. When no timestamp is specified while reading, Bigtable returns the latest version.
```
In a multi-dimensional indexed system, often writing a new value (against a new timestamp) is faster than overwriting a previous value (for example, no locking might be needed to write a new value while readers are utilizing older values). Additionally, such versioned writes can also help when flash-based disks are in use because of the typical characteristics of flash drives (limited number of write cycles and fairly involved over-write process where a large block needs updates).
```
If we allow any number of versions of a cell to exist, data might get unwieldy. Hence, garbage collection is necessary over time. Bigtable has two per-column-family options for automatically garbage-collecting cell versions. First, the user can only request the most recent “N” versions of a cell be retained. The second option is to keep versions that are new enough, such as when only the versions that are less than five days old are kept.

[Timestamps]

## High-level design
As mentioned earlier, we are designing a database that is literally a Bigtable containing huge datasets. So, how can we take the data in such a massive table, divide all the work, and distribute it over thousands of machines? Firstly, we will introduce an abstraction known as the tablet. The tablet is a dynamic partition of the row space. We’ll break up the row space dynamically. We’ll allocate one part of the row space to one server and another part of the row space to the other. This provides us with parallelism. It also provides a mechanism for clients to control locality, allowing them to select row keys in such a way that their data receives spatial locality based on the access patterns they expect.
```
Spatial locality (also termed data locality) refers to the use of data elements within relatively close storage locations. (Source: Wikipedia)
```
- Three main parts make up the Bigtable implementation: a library that is linked to each client, one master server, and several tablet servers.

- Client library: A library component that is shared by all clients. This library enables clients to communicate with Bigtable.
- One master server: The master server is in charge of allocating tablets to tablet servers, identifying tablet server addition and expiration, regulating tablet-server traffic, and garbage collection of files in GFS (a distributed file system). It also supports schema changes like table and column family formation.
Tablet servers: All tablet servers are in charge of a certain group of tablets generally around 10 to 1000 tablets. Each Tablet server provides reads and writes of the data to the tablets to whom it is allocated. Servers can be added or removed in a Bigtable cluster as per need. New tablets can be made and assigned, old ones can be merged, and they can be reassigned from one server to the other to accommodate changes in demand.
```
One master server can have its issues (like the risk of being a single point of failure), though such centralized control drastically reduces the complexity of the overall system compared to a design where we could have multiple masters. Complexity can make the dollar cost of designing and operating a system quite high. The designs of a system should reflect the current (and near-future) needs and evolve from there.
```

[High-level architecture]

One of the system’s characteristics is that one tablet is owned by one server. In contrast, one server often owns several tablets, so the server is responsible for handling reads and writes to a large number of tablets in the system. For example, we might have tens of thousands of tablets in total, and each server could handle a couple of hundred tablets.

Load balancing is expressed in terms of tablets. So, if one server has too much load in terms of data or reads/writes requests, we migrate tablet ownership from one server to another. The tablets could become imbalanced (in terms of the amount of data or number of reads and writes coming for them) in comparison to other tablets. The data is actually stored as files in an underlying file system, which is the Google File System. This means that when we move tablets around, we don’t have to move the data, we just move the ownership of the data. The following picture depicts how a typical table can be sharded as tablets.

[Tablets]

The first tablet may contain our application’s home page, as well as the page’s contents. We also may have collected some data, such as the language. The first tablet is located on server one. We might have another tablet on server two that is overloading, either because the tablet itself is too huge or because it has too much load, so we might need to split it apart. The system will automatically split and downsize tablets, and we will be able to move one of the new tablets from server two to server three. This is how our system will handle load balancing.

Bigtable is assembled on many other Google infrastructure components, including:

- GFS (Google File System): To keep log files and the data.
- SSTable: Internally, Bigtable data is stored in the Google SSTable file format.
- Chubby: It is a highly available and persistent distributed lock service to coordinate actions between multiple entities.
- Cluster Scheduling System: Bigtable uses a cluster management system. This system schedules the jobs, manages resources on shared machines, deals with failures of machines, and monitors the status of the machine.

### API design

The Bigtable API has operations for adding and removing tables and column families. Additionally, it offers tools for modifying information related to clusters, tables, and column families, including access control rights.

Clients have the ability to add, modify, or delete values from Bigtable. Additionally, clients can iterate over a subset of the information in a table or look up values from specific rows. Clients can traverse several column families, and there are various techniques for restricting the rows, columns, and timestamps generated by a scan. The following table describes some of the Bigtable characteristics regarding reads/writes.


```
                         Bigtable characteristics

Single row transactions  Bigtable enables single row transactions, allowing users to execute atomic read-modify-write operations on data that is stored in a single row key.

Client interface         Bigtable has a client interface for batch writing over row keys but it does not allow transactions across row keys.

Integer counters         Cells can be used as integer counters in Bigtable.

MapReduce jobs           Bigtable can be used in MapReduce jobs as both an input source and an output target due to a set of wrappers.

Writing scripts          Clients can also write Sawzall scripts (a language created by Google) to guide server-side data processing.

Here are some commonly used APIs:

- Set(): For writing cells in a row.
- Delete(): For deleting a specific cell in a row or all cells in a row.
- lookup(): To lookup values over particular rows or traverse over a subset of data in the table.
This was the high-level part of Bigtable. Let’s get into further detail about Bigtable and find out how different components work together in the next lesson.
