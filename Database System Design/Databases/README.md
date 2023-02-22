# Introduction to Distributed Databases
## Motivation
Databases are a critical component of most real-world systems. Systems moved from traditional relational databases to NoSQL solutions to meet the evolving scalability and availability needs. They often paid the price in terms of relinquishing strong consistency or lower performance. It has been the goal of the database community to have the best of both worlds (strong consistency, high performance of relational databases, and scalability and availability of NoSQL databases). While we have ways to go towards that end, research and development over the last decades have brought us closer to that goal.

## What we will learn
We’ve selected the following three papers to discuss in the next few chapters:

- [Bigtable] Fay Chang, Jeffrey Dean, Sanjay Ghemawat, Wilson C. Hsieh, Deborah A. Wallach, Mike Burrows, Tushar Chandra, Andrew Fikes and Robert E. Gruberng. 2003. Bigtable: A Distributed Storage System for Structured Data. In Proceedings of the 7th {USENIX} Symposium on Operating Systems Design and Implementation (OSDI 2006). Conference proceeding pages 205-218.

- [Megastore] Jason Baker, Chris Bond, James C. Corbett, JJ Furman, Andrey Khorlin, James Larson, Jean-Michel Leon, Yawei Li, Alexander Lloyd, and Vadim Yushprakh. Megastore: Providing Scalable, Highly Available Storage for Interactive Services Proceedings of the Conference on Innovative Data system Research (CIDR) (2011), pp. 223-234

- [Spanner] James C. Corbett, Jeffrey Dean, Michael Epstein, Andrew Fikes, Christopher Frost, J. J. Furman, Sanjay Ghemawat, Andrey Gubarev, Christopher Heiser, Peter Hochschild, Wilson Hsieh, Sebastian Kanthak, Eugene Kogan, Hongyi Li, Alexander Lloyd, Sergey Melnik, David Mwaura, David Nagle, Sean Quinlan, Rajesh Rao, Lindsay Rolig, Yasushi Saito, Michal Szymaniak, Christopher Taylor, Ruth Wang, and Dale Woodford. 2012. Spanner: Google’s Globally-Distributed Database In Proceedings of the 10th USENIX conference on Operating Systems Design and Implementation (OSDI’12). USENIX Association, USA, 251–264.

## Why did we choose these systems?
The three systems we’ve selected are not only seminal work in themselves, but they also help us understand the trade-offs for distributed databases.

### The Bigtable system
Bigtable is a key-value store at its heart but with some exciting functionalities, which are as follows:

- The keys have some elaborate structure to them that helps construct a table-like abstraction on top of it.

- The data can be multi-versioned, which helps speed up concurrent reads and writes without locking for some use cases.

- A few table columns are physically kept together to benefit from the locality.

Bigtable is a column-family database that introduced many innovations like SSTables. This system shows us how a key-value store can be a high-performance database. Bigtable does not provide strong data consistency across table rows (the guarantees are only at the key level). Additionally, the data model of Bigtable is not relational, which implies that users of Bigtable have a learning curve.
### The Megastore system
While Bigtable was suitable for many use cases, developers of online transactional processing (OLTP) applications were challenged to build applications without a strong schema, cross row, cross-shard transactions, and the familiar SQL query language. Megastore was the response to that need. Megastore was built on top of Bigtable and provided stronger consistency within a shard of a table. However, the application code needed a lot of work in its code to do more complicated transactions. Additionally, the performance of such applications was often low.

### The Spanner system
Spanner provided the ability of transactions across shards with external consistency, and many lock-free operations like read snapshots. Surprisingly, the innovations of Spanner were due to a special kind of timing mechanism (called TrueTime), where Google could control the clock's skew and construct linearizability guarantees on top of that. Additionally, Google's private wide-area network between data centers with redundant paths made the network partitions less often, and therefore resulted in high availability. Additionally, Spanner provided an SQL language to interface with the system.

We hope our selection of distributed database systems teaches us many important lessons in system design. Let's dive in!


[Evolution of distributed databases](./timeline.jpg)
