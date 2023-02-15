# Design and Evaluation of Colossus
## Colossus control plane and scalability
We’ve seen the architecture of Colossus and its components in the previous lesson. The most significant and interesting part of Colossus is the control plane that manages the underlying storage/file servers and replaces the GFS single master. A GFS cluster consists of a single master and multiple chunkservers, with a possibility of multiple GFS cluster instances per data center. If an application's data exceeds the limits that a single GFS cluster can handle, it uses multiple GFS cluster instances. If an application is storing its data on multiple clusters, it means there is some partitioning logic that should be implemented to tell which data should be stored in cluster 1, which data should go to cluster 2, and so on.

With GFS clusters, applications themselves are responsible for partitioning data across different GFS cluster instances. The need for multiple GFS clusters arises because a single cluster is not scalable enough to meet the growing data demands of the application. As we’ve discussed in the previous lesson, the reason for these scalability issues was the single master. Colossus comes with a control plane that replaces the single master in GFS, and makes the Colossus cluster scalable to exabytes of data and hundreds of thousands of machines.

[A typical Colossus cluster]

The control plane is the foundation of the Colossus file system. As shown in the illustration above, there are many clients that are sharing the same storage pool (D file servers): the Youtube servers storing and retrieving their data, Google Cloud storage attached to Google Compute Engine VMs, and Ads MapReduce nodes. All of these clients are able to share the same storage pool with the help of the Colossus control plane that is managing the underlying storage pool. The control plane is providing an illusion to the clients that they have isolated file systems. Let's see how the control plane manages all this at such a large scale.
```
Note: Resource sharing among different workloads helps utilize resources efficiently and lower the costs for clients. Since there are different requirements from workloads, we can prioritize them. For example, the low-latency workloads at the peak time can be executed first and batch-analytics workloads can be executed during the idle time.
```
### Curators (control operation servers)
To handle a large number of control/metadata operations for a large number of clients, there are many curators to which the client requests are distributed. The curators are horizontally scalable. Since there’s a large number of clients, and a large volume of data, the metadata for managing all this is also very large. All the curators store the file system metadata in Bigtable, which is Google's highly scalable database. Let's look at Bigtable.

### Bigtable (metadata database)
Colossus was primarily developed to address metadata scaling issues that were encountered in the Google File System (GFS). GFS was designed with the assumption that the files to be stored in the file system were very large, so the metadata-to-data ratio was tiny compared to the data size. In a search system where there are many small documents/files, the size of the metadata almost reaches the size of the data. For such systems, the GFS metadata service becomes the bottleneck. Colossus scaled over the largest GFS clusters a hundred times by storing file metadata in Bigtable.

Bigtable is Google’s sparsely filled table, scalable to billions of rows and thousands of columns. This allows us to store terabytes or even petabytes of data. With low latency, it offers a high read and write throughput. Each row in Bigtable consists of a single key and multiple values, as shown in the illustration below. Multiple columns can be grouped into a column family. Each cell holds a unique version of the data, identified by a timestamp.

[The structure of Bigtable]

Bigtable is partitioned horizontally. Each partition is known as a tablet in Bigtable. Tablets are distributed among multiple nodes (also called tablet servers) in the Bigtable cluster. Each node is responsible to serve requests for the tablets it contains. It is important to note that these nodes don't store tablets, but they just store pointers to the set of tablets. The actual tablets are stored on GFS/Colossus in SSTable format files. A Bigtable cluster is all about partitioning Bigtable data into tablets, balancing and rebalancing the load on the tablet servers (this happens quickly because we just have to update the pointers and not move the tablets) by splitting or merging the tablets. A high-level architecture of the Bigtable cluster is shown in the illustration below.

[The architecture of Bigtable]

A cluster's maximum throughput and the number of concurrent requests it can handle can both increase with the addition of nodes. Cluster replication is used to manage cluster failover.

Colossus uses Bigtable to store the file system's metadata. (It may raise a concern that Colossus and Bigtable are both dependent on each other because Bigtable uses Colossus to store SSTables. This has been addressed in quiz question three.) Since Bigtable is scalable, available, and durable, we can store an increasing volume of metadata which we can access at low latency and we are sure that metadata once written is not lost. The details on the Bigtable design can be found in the Bigtable chapter.


## Evaluation of Colossus
Let's see how the Colossus design meets its requirements.

- Scalability: Colossus scales because its metadata service also scales. To handle a large number of client requests, Colossus has multiple curators which scale horizontally. The metadata is stored in a highly scalable database, Bigtable. To increase the data storage space, we can add storage servers to the storage pool.

- Availability: There is no single point of failure for metadata in Colossus. At the frontend of the metadata service, there are many curators, if one fails, the requests can be sent to the others, and we can add curators to the cluster. At the backend, Colossus uses Bigtable to store all metadata. The Bigtable serves large reads and writes with low latency. The data holding servers (the D servers) are either replicated or use Reed-Solomon encoding to recover from disk or data node failures.

- Storage efficiency: Colossus maximizes storage efficiency by sharing resources among different workloads. The storage consists of different spinning disks and flash drives of different sizes and types. Flash drives are fast, so Colossus puts hot data on flash drives for fast retrieval and low latency.

## Conclusion
We’ve seen all of Google's applications share the same file system, so it should be scalable and available. A single Colossus cluster provides scalability to exabytes of data, and the backbone of it is:

- a scalable metadata service built on top of Google's Bigtable database

- a shared storage pool to add more and more storage

### System design wisdom in Colossus
- The needs of the real world are evolving, and the systems that respond to them are also changing. As designers, we need to stay ahead of the needs in a cost-effective manner. The progression of the distributed file system from GFS to Colossus is an example of such progression. Colossus focuses on those parts of GFS that were becoming bottlenecks to scalability (a single master) while keeping many of the other design traits of GFS design.

- Disaggregating different parts of the system helps to scale them independently. For example, if Colossus gets a lot of small files (that will be a lot of metadata but less data), the metadata layer can scale independently of the D servers. If Colossus gets huge data in a few files, D servers can horizontally scale without impacting other parts of the system. Horizontal scalability enables operations to scale up or down the resource as per (predicted) need.

In conclusion, Colossus is a natural design progression to the original GFS system.
