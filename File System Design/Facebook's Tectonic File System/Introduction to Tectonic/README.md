# Introduction to Tectonic
## Storage systems—specialization vs. generality
Over the years, organizations have built large distributed storage systems to meet their evolving needs. Such systems are often optimized for specific use cases and might not be a good fit for a general storage workload. The operational complexity of evolving and maintaining many storage systems takes its toll in terms of monetary cost and potential duplicate work. As operational experience with specialized systems builds, system designers often get new insights on how they could use a single generalized system to meet the needs of many use cases.

Note: In system design, we often start with a specialized system that is optimized for a specific use case. Over time it might be possible to consolidate many such specialized systems into one general system, until we get some new use case that a general system is not able to meet. The design activity acts like a pendulum between specialized and general systems over here.

[Systems often swing between specialized and generalized systems over time](./swing.jpg)

The Facebook service is a canonical example where data needs are diverse in terms of workload, and overall data size is huge and increasing. In the following lesson, we’ll discuss Facebook’s storage systems to better understand specialization versus generalization, in the context of storage systems.

[The global data growth rate in one decade is about 50 folds (Source: IDC’s Digital Universe Study, sponsored by EMC, December 2012)](./growth.jpg)

## Facebook: From a constellation of storage systems to Tectonic
There are numerous different tenants and hundreds of use cases/applications per tenant, for a variety of storage needs. Blob storage and data warehousing are two major storage applications with different workload characteristics and storage needs.

```
A tenant can be considered an organizational division or group with well-defined and coherent business and technical objectives.
```

For a blob store, data access patterns change over time. Some proportion of data is heavily accessed and such a workload needs a substantial number of input-output per second (IOPS) to serve the clients well. Over time, as new hot data comes in, while the older data starts becoming cool off as fewer read/write requests come in for such data. Such data has much lower needs in terms of IOPS but an always growing need for data storage.

Facebook had a specialized storage (Haystack) to store hot data while another system (f4) to move less frequently accessed data to it. To meet the evolving needs of hot data in Haystack, a high number of storage nodes/disks were commissioned to meet the IOPS requirements. There are a limited number of IOPS available per disk, so the required overall count of disks was always increasing. However, these disks were not fully utilized in terms of storage and had a lot of excess capacity.

```
Haystack: Beaver, Doug, Sanjeev Kumar, Harry C. Li, Jason Sobel, and Peter Vajgel. “Finding a needle in haystack: Facebook’s photo storage.” In 9th USENIX Symposium on Operating Systems Design and Implementation (OSDI 10). 2010.
```

```
f4: Muralidhar, Subramanian, Wyatt Lloyd, Sabyasachi Roy, Cory Hill, Ernest Lin, Weiwen Liu, Satadru Pan et al. “f4: Facebook’s warm {BLOB} storage system.” In 11th USENIX Symposium on Operating Systems Design and Implementation (OSDI 14), pp. 383-398. 2014.
```

On the other hand, the f4 system was bottlenecked on storage capacity, while the IOPS needs were nominal. One might wish that Haystack could utilize the excess IOPS of the f4 system and that f4 could utilize the excess storage capacity of Haystack. However, since these are independent systems, there was no provision for such resource sharing, and costly resources were being stranded.
```
The disks’ storage capacities grew steadily over time while the IOPS per disk essentially stayed the same. This means that the IOPS per terabyte has declined over time. This trend concerns applications that are IOPS-bound (like a blob store).
```

[Storage](./fb)

As a second example application, data warehousing not only needs an enormous amount of data capacity but also the ability to crunch this data to extract business intelligence. Facebook was using multiple clusters of HDFS in a federated fashion. A single HDFS cluster can scale from many Terabytes to a few Petabytes. However, this is not enough for the warehousing application, and multiple HDFS clusters were in use where data was divided between HDFS clusters. Clients were required to keep track of their data to know the HDFS clusters on which the data resides. Going forward, warehouse application data needs are approaching upto multiple Exabytes, and the federated strategy is not only operationally complex but also hard to scale.
```
HDFS: Borthakur, Dhruba. “The hadoop distributed file system: Architecture and design.” Hadoop Project Website 11, no. 2007 (2007): 21.
```


```
Note: Carefully selecting the multiple HDFS clusters so that our needs are met with an efficient use of the clusters’ capacity and available throughput is an instance of a two-dimensional bin-packing problem (which is an NP-hard problem).
```
The two examples above highlight the problems that arise in specialized storage systems. Facebook’s answer to these challenges was a new, general storage system, Tectonic, that could provide a common storage layer where resources are well utilized. However, applications are still performance-isolated from each other and could meet Facebook’s needs.


## Our needs
Our system is based on the following functional and non-functional requirements.

### Functional requirements
Following are our primary functional requirements:

- Tectonic should be able to provide multiple Exabytes of storage capacity to its tenants, and this storage should be horizontally scalable.

```
A system that has the tendency to increase or decrease storage based on the requirements is known as horizontally scalable.
```

- Tectonic should be able to utilize the storage resources well by sharing them with all the tenants.

- Tectonic should provide configuration knobs to the applications so specific applications could pick and choose specific aspects of the storage system for specific optimizations. An example of such a knob is the ability to choose either full data replication or Reed Solomon-based codes for fault tolerance.
```
Reed Solomon: This is a technique for error correction on blocks. It first encodes the data with the redundant bit and calls it parity. In case of any errors, the decoder then fixes the error block by block using parity.
```

### Non-functional requirements
Following are our non-functional requirements:

- The availability of many thousands of IOPS and the ability to horizontally scale IOPS over time.

- Tectonics should ensure performance isolation between applications so that sharing resources does not negatively impact the applications.

- Tectonics should be highly available because many applications will rely on it for storage needs.

- Tectonic should provide other usual desirable properties from such a large distributed system, such as fault tolerance, maintainability, etc.


## High-level design
Tectonic will primarily be within a data center file system running on a cluster of servers. A typical cluster can span from hundreds to thousands of servers. A tectonic system consists of three major types of components—a Metadata Store, many Chunk Stores, and some stateless background services. The high-level architecture is shown in the illustration below (we’ll discuss every component of the architecture in detail in the coming lessons).

[The architecture of Tectonic](./arch.png)

- The client application uses a Client Library through which the end users perform the file and data operations.

- The Metadata Store consists of stateless metadata services and a scalable key-value store and builds the file system logic on top of the key-value store.

- The stateless background services provide services such as garbage collection, rebalancer, disk inventory, memory utilization, the maintenance of nodes in the cluster, and many more to improve performance.

- The Chunk Store is a collection of nodes for storage that maps data onto chunks and places them on the hard disk. The data can be accessed in the form of chunks.

```
Chunks: Logical partition of data that is physically stored on the disks.
```

- The Client Library requests the Metadata Store for the metadata information, such as the location of chunks of the requested file. The Metadata Store looks into its metadata and responds to the client with the location of the requested chunks in the Chunk Store. The client then asks the Chunk Store for the data operations.
```
Note: A single Tectonic cluster can store multiple exabytes of data efficiently and allow hundreds of clients to access it concurrently. One exabyte is 10^18 bytes or 1000 petabytes.
```

## Bird’s eye view
In the next lessons, we’ll design and evaluate Tectonic. The following concept map is a quick summary of the problem Tectonic solves and its novelties.
[Overview](./birds.jpg)
