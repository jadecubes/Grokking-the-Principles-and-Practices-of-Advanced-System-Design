# Database Buckets and Data Model of Spanner
## Buckets and placement
Spanner provides an additional layer of abstraction over the bag of key-value mappings in the form of a directory or bucket—a group of adjacent keys that all begin with the same prefix. Applications that support buckets manage the data locality by carefully selecting keys.

The basic organizational structure for data is a bucket. All the bucket's data share the same replication settings. Consider the illustration below. The data is transferred between Paxos groups bucket by bucket. To minimize the load on a Paxos group, we can relocate the frequently accessed buckets into the same Paxos group or place a bucket geographically closer to its accessors. Changing a bucket's location doesn't have to interrupt service for the client. Normally, copying 50 MB of data to a new bucket would take a few seconds.

[The bucket2 is being moved from Paxos group 1 to 2]

Given that a Paxos group may have several buckets, the tablet in Spanner and Bigtable differs in a way that the Spanner tablet does not need to be a single and lexicographically contiguous partition of the row space. A Spanner tablet is an enclosure containing many row-ranges. It allows co-locating numerous frequently used buckets together.


## Relocating buckets
Spanner uses the movedir function to relocate buckets across Paxos groups. The movedir also adds or removes replicas from Paxos groups. We don't implement movedir as a single transaction to prevent a large data move from stalling ongoing reads and writes. Instead, it keeps track of when it has started moving and moves the data in the background. After all data (except a small amount of data) has been moved, the remaining small quantity will be moved in one atomic operation while the metadata for the two Paxos groups is updated.

A bucket is the smallest unit for which an application can specify its replicas' geographical replication attributes or placement. Administrators have command over two dimensions:

1. The total number and type of replicas

2. The geographical placement of replicas

### Types of replicas
In a multi-region setup, replicas can be of different types (more on that later), whereas single-region instances only use read-write replicas. For a write transaction, Spanner requires majority voting replicas to agree on the commit before it is made. The voting replicas must do consensus if data is updated in a Spanner database. To keep the latency of consensus communication to a minimum, it is better to use lesser voting replicas and place them physically close to control the latency due to propagation delay. Therefore, there are always three read-write replicas in their availability zones in regional deployments. These replicas have a complete copy of the client's data and can cast a vote. Because all three copies are located in the same location, network latency is kept to a minimum, and data may still be written to the database if one replica goes down.

The multi-region configurations have more replicas spread across multiple data centers. It allows clients to access their data promptly from multiple locations. If all the replicas are read-write replicas, then it is not desirable to use a multi-region configuration because increasing the number of read-write replicas will also increase the write quorum size. This can lead to longer network latency if the replicas are spread out across different physical locations and they also use more storage since read-write replicas have a complete copy of the data. Multi-region configurations have two more sorts of replicas in addition to read-write replicas with fewer responsibilities.

- Read-only replica: They cannot vote for electing leaders when the write transactions are committed. Therefore, these replicas allow scaling the read without increasing the quorum size required for writes. The write transaction is only handled by leaders. Whenever the data is changed, the leader updates the replicas.

- Witness: They vote for the leader and commit the write transactions. The witness replicas don't keep a copy of the complete data, so they can't serve reads or take over as leaders. While the read-write replicas need to keep a complete data copy to serve reads. Witness simplifies quorum attainment for writes and does not necessitate the resources (storage and computation) needed by read-write replicas.

To summarize, replicas in Spanner can be of three types: read-only, read-write, and witness replica. However, multi-region instance setups may employ any mix of the three types of replicas, while single-region instances only use read-write replicas.

The following table highlights the various Spanner replica kinds and their properties. This information has been taken from https://cloud.google.com/spanner/docs/replication.

```
Replica Type         Can Vote       Can Become a Leader       Can Serve Reads

Read-write            Yes             Yes                        Yes

Read-only             No               No                        Yes

Witness               Yes              No                         No

```
Using these, a list of possibilities in these two dimensions can be generated with meaningful names. For example, Europe can have one witness and replicate five ways.

By assigning different values to each database and/or bucket, an application can manage the rate and pattern of data replication. For instance, the data of User A could have three replicas in North America. In contrast, the data of User B could have five replicas in Europe because the application stores each user's data in its bucket.
```
Note: Spanner provides placement-specification language to decouple the management of replication configurations from the placement itself.
```
If a bucket becomes too big, we will shard it into several fragments. Several Paxos groups or different servers may be used to serve a fragment. The movedir moves fragments between groups rather than whole buckets.

## Data model
The following are data characteristics that Spanner provides to its applications:

- A data model with schematized semi-relational tables. The tables are organized hierarchically so relevant data can be kept together in the same buckets and easily replicated, locked, and maintained across Spanner's spanservers.

- A query language

- Generic transactions

Several considerations are pushed for the inclusion of support for these characteristics. Megastore's success proves the importance of schematized, semi-relational tables and synchronous replication. Megastore's data schema is easier to administer than Bigtable's and supports synchronous replication across data centers, which is why around 300 Google applications use Megastore despite its poor performance. Bigtable only allows for replication that is eventually consistent across data centers. Popular services of Google, such as Gmail, Picasa, Calendar, Android Market, and AppEngine, were all powered by Megastore when the Google Spanner paper was released.

Bigtable does not support cross-row transactions. The two-phase commit can solve this, but the performance and availability issues make it too costly to sustain. Spanner allows application developers to address bottlenecks caused by excessive transaction use when they happen rather than always coding around the absence of transactions. The availability issues can be reduced by using a two-phase commit over Paxos.

The application data model is used to extend the directory-bucketed key-value mappings. A universe may contain one or more databases created by an application. There is no limit to the number of schematized tables that can be stored in a database. The tables are similar to those in a relational database, with rows, columns, and versioned values.

Spanner's data model is not strictly relational to ensure scalability. It will require naming each row. A specific column of the table is used for this purpose. Specifically, one or more primary key columns must exist in every table. As a result of this need, Spanner still resembles a key-value store: row names are formed from primary keys, and mappings between primary-key and non-primary-key fields are defined for each table. A row exists only if the keys of the row have been assigned values, even if those values are NULL. By imposing this structure, applications can manage data locality via key selection.

## Creating schema in Spanner
Let’s consider the illustration below that shows an example of Spanner's schema that is used to store user's photo metadata on an album basis. As the clients need to partition their Spanner database into one or more tables hierarchies, the schema language of Spanner resembles Megastore's.

Consider an example where we create an interleaved table. In this table, rows are physically stored alongside their corresponding parent rows. The primary key from the parent table must be the first part of the composite primary key for the child table. First, we will create a parent table named "Users." Then, we will create a child table, "Albums." We will specify it as a child of the “Users” table using INTERLEAVE IN. We will insert data in both tables. Each parent table row will be associated with the relevant child data and will be a part of the bucket together.

```
The following slides are based on Corbett, James C., Jeffrey Dean, Michael Epstein, Andrew Fikes, Christopher Frost, Jeffrey John Furman, Sanjay Ghemawat et al. "Spanner: Google’s globally distributed database." ACM Transactions on Computer Systems (TOCS) 31, no. 3 (2013): 1-22.
```

[Creating]

The INTERLEAVE IN declarations in database schemas are how client applications communicate the underlying hierarchies. A bucket table is the parent table in a tree structure. One can construct a bucket by joining the rows of all descendent tables that begin with the key K in lexicographic order. If you delete a row in the bucket table and use the ON DELETE CASCADE clause, all dependent entries will also be removed.

The illustration above also depicts the database's interleaved layout. For example, the Albums table's row depicts user id 2 and album id 1 is labeled Albums(2,1). Clients may express the locality relations between different tables because this bucket-based interleaving of tables is required for optimal performance in a partitioned and distributed database. Otherwise, the crucial location connections would not be known by Spanner.

In this lesson, we covered how buckets and their placement allow applications to manage the data locality and Spanner's data model. In the next lesson, we will learn about the TrueTime API.

