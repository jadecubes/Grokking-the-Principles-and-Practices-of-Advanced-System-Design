# Detailed Design of Bigtable: Part II
Many tables are kept in a Bigtable cluster. A table in Bigtable is made up of tablets, each of which stores all the data related to a specific range of rows. Each table starts out with only one tablet. As a table expands, it is automatically divided into many tablets, each of which has a standard size of between 100 and 200 MB. Let’s look at how we can locate and assign tablets and how read/write works in Bigtable.


## How to locate the tablets
As tablets can migrate from server to server due to load balancing, tablet server failures, and so on, how do we figure out the right tablet server given a row? To find the answer to this question, we must locate the tablet whose row range includes the target row. To save tablet location data, Bigtable uses a three-level structure similar to that of a B+ tree.

- In those three levels, the root tablet’s location is stored in a Chubby file at the first tier.

- The second level contains all Metadata tablets.

- The third level contains all user tablets.

- The root tablet has a unique metadata table that records the position of all other tablets.

- The first tablet in the Metadata table is the root tablet, which is treated differently from other tablets. To ensure that the tablet location hierarchy does not surpass the three tiers, the root tablet is never divided.

- The Metadata table stores the information in the following way:

  - The position of a tablet is stored in the Metadata table under a row key that encodes the tablet’s table identity and end row (the end row helps in identifying the start of the next tablet’s information). Every Metadata row holds about one KB of information in memory. The three-level hierarchy method can handle 2^34 tablets with a reasonable restriction of 128 MB Metadata tablets (This simply means that a tablet can contain information for 128000 tablets in memory. If the root tablet contains information for 128000 Metadata tablets and each of these tablets also contains information for 128000 user tablets, the total becomes around 2^34).
  - The Metadata table also stores secondary information, such as a log of all events related to each tablet; for instance, this could be the time a server begins servicing a tablet. This data is useful for troubleshooting and analyzing performance.

[Tablet location](./location.png)

- Tablet positions are cached by the client library. In the event that the client cannot determine the position of a tablet, or if the cached location information turns out to be inaccurate, the client will iteratively climb up the tablet location hierarchy. If the client does not have any stored information in its cache, the locating algorithm will require three round trips within the network, one of which is to Chubby. Since outdated cache entries are only found after a miss, the locating technique might take more than three round trips if the client’s cache is outdated.
- Even though tablet locations are maintained in memory, eliminating the need for GFS visits, we lower this cost further in the usual scenario by making the client library prefetch tablet locations. When reading the Metadata table, the client library retrieves the metadata for multiple tablets at the same time.

[Empty cache](./locate)


## How to assign the tablets
At any single moment, a tablet is allocated to just one tablet server. The master is responsible for keeping track of which tablet servers are currently up and which tablets are currently assigned to those servers. The master also maintains a record of unassigned tablets and allocates them to tablet servers that have enough space.

Upon starting, a tablet server creates a file (which has a unique name) in Chubby’s “servers” directory and immediately obtains a distinct lock on that file. This technique informs the master that the tablet server is operational. If a tablet server ends up losing its exclusive lock, such as during a network partition that causes it to terminate its Chubby session, it will no longer serve its tablets. (Chubby offers a fast approach that allows a tablet server to verify whether the lock is still in place without consuming network bandwidth.) As long as the file remains, a tablet server tries to recover the exclusive lock on it. Since the tablet server can’t continue operating if the file has disappeared, it terminates its own process. When a tablet server stops, it tries to free its lock so that the master can redistribute its tablets more rapidly. The tablet server can stop, for example, whenever the tablet server’s machine is removed by the cluster management server.

The master is in charge of determining whether a tablet server seems to be no longer serving its tablets and reassigning them as quickly as technically possible. The master regularly requests each tablet server for the condition of its lock on the file in Chubby to determine when a tablet server is no longer servicing its tablets. If a tablet server indicates that it has failed its lock, or if the master has not been able to access a server in the past few tries, the master tries to obtain an exclusive lock on the server’s file. If the master is successful in obtaining the lock, this indicates that Chubby is still functioning and the tablet server is either inactive or experiencing difficulty connecting to Chubby. In this case, the master deletes the server file of the tablet server to make sure that it doesn’t serve anymore. Once it removes the server’s file, the master can shift all the tablets, which were once allocated to a server, into the group of unassigned tablets.

[How](./assign)

When a master’s Chubby session runs out, it terminates itself to protect the Bigtable cluster from networking problems between Chubby and the master. Nonetheless, as previously stated, the distribution of tablets across tablet servers is unaffected by the loss of a master.

When the cluster management system starts a master, it must first figure out the existing tablet assignments before it can alter them. At startup, the master performs the following processes.

1. To avoid repeated master instantiations, the master acquires a distinct master lock in Chubby.
2. The master searches Chubby’s “servers” directory for active tablet servers.
3. The master interacts with each active tablet server to determine which tablets are allocated to each server.
4. To understand the whole collection of tablets, the master analyses the Metadata table. When this search detects a tablet that has not yet been assigned, the master inserts it into the list of unassigned tablets. Likewise, the master creates a pool of unassigned tablet servers that are eligible for tablet assignment. This information is used by the master to allocate unassigned tablets to relevant tablet servers.

## Tablet serving
GFS saves a tablet’s persistent state. Updates are saved in a commit log, which also saves redo data. The most recent changes are saved in Memtable, a sorted buffer in memory, while previous modifications are kept in a series of SSTables. A tablet server gets its metadata from the Metadata table to recover a tablet. The list of SSTables that make up a tablet is included in this metadata information, along with a collection of redo points. These serve as pointers to any commit logs that could provide the tablet-related data that it is recovering. The server loads the SSTables’ indexes into memory and recreates the memtable by incorporating all modifications that have been committed since the redo point.

### Write operation
When a write request is received by a tablet server, it is checked to ensure that the request is properly formatted and that the writer has permission to make the change. To determine who is authorized to write, we can scan the list of authorized editors in the Chubby file. The list can usually be found in the Chubby client cache. A legitimate mutation is recorded in the GFS commit log. We employ a group commit to increase the throughput of several minor alterations. After the write has been committed in the commit log, its contents are loaded into the memtable, which is in memory.

[Write](./write)

### Read operation
A read operation is also evaluated for well-formedness and sufficient authorization when it reaches a tablet server. A legitimate read operation is accomplished on a combined view of the series of SSTable and the memtable. The combined view can be created effectively because the Memtable and SSTables are sorted.

[Read](./read)

While tablets are separated and merged, incoming read and write operations can carry on with the help of compactions.
## Compactions
Bigtable rearranges SSTables on a regular basis to permanently erase deleted entries and rearrange data to make reads and writes more efficient. This is referred to as compaction. Here are the following types of compactions.

### Minor compaction
The size of the memtable grows when write operations are completed. When the size of a memtable hits a certain limit, it is frozen, a new memtable is produced, and the frozen memtable is transformed into an SSTable and written to GFS. This process is known as minor compaction. Minor compaction creates a new SSTable with the following objectives.

It decreases the tablet server’s memory use.
In the event that this server fails, it decreases the amount of information that must be read during recovery from the commit log. Incoming read/write operations can proceed without interruption as compactions occur.

[Compaction](./minor)

### Merging compaction
Every minor compaction results in the creation of a new SSTable. If this behavior was allowed to continue unchecked, read operations may be required to integrate updates from a random number of SSTables. Merging compaction is conducted to minimize the amount of SSTables, which examines the data of a few SSTables and the memtable and writes out a new SSTable. When the compaction is complete, the input SSTables and memtables can be disposed of as they are already copied to the new SSTable.

[Compaction](./merging)

### Major compaction
Major compaction is a merging compaction that merges all SSTables into a single SSTable. SSTables produced by major compactions do not include any deletion information or deleted data, however, SSTables produced by non-major compactions might include deleted entries. Bigtable performs major compactions to free up resources that were being utilized by deleted data and also promptly removes the deleted data from the system. This is especially important for services that handle sensitive information, such as users’ financial information.

[Compaction](./major)
