# Tenant-specific Optimization in Tectonic
In the previous lesson, we discussed how we used multitenancy to fairly and efficiently share IOPS and storage capacity. Earlier tenants were using different strategies to store data reliably. Some were using full data replication for speedy writes and reads, while others were using the Reed-Solomon encoding to reduce storage needs, but at the cost of added latency (that will be needed to encode data while writing and decoding data while reading).

We allowed multiple tenants with various workload characteristics and performance requirements to work on the same shared storage. We’ll enable tenants to request their required storage mechanism via the Client Library that we discussed earlier in our design.

## Overview
Now, we consider two tenants, data warehousing and blob store, as our examples to explain the specific storage consumption or latency-related optimizations. The following are the two ways to perform tenant-specific optimization for low latency and storage efficiency.

1. Optimizing writes on data warehouse: We need to optimize how large-scale data can be stored using full-block operations. Since the data is large, we can’t use partial block operations because, in such use cases, partial blocks will increase the latency and decrease storage efficiency.

2. Optimizing blob storage: We need to optimally store the small-scale data (blobs) as well, where blob storage comes in. Since the data is not at a large scale, we’ll perform partial-block operations on both hot and warm blobs.

The following illustration shows the summary of the optimizations for both tenants.

## Optimizing writes on data warehouse
Writing data once and reading it many times later is a dominant pattern in workloads for data warehouses. The file is only accessible to readers for certain workloads after the file is closed by its creator. As a result, the file becomes immutable forever. Since the data can only be read once the file creator is done writing, we prioritize write requests with low latency over the low latency append requests.

Since we will perform write operations on a large amount of data, these write operations will be on the full block. To optimize such write operations, we have used the following two ways:

1. Reed Solomon(RS)-encoded asynchronous writes: This uses write-once-read-many for optimal network, storage, and IO performance.

2. Hedged quorum writes: This is for generating reservation requests to decrease latency.

### RS-encoded asynchronous writes
The write-once-read-many design pattern is used by Tectonic to decrease the overall file write time while increasing IO and network performance. Applications can buffer writes up to the block size because this approach doesn’t involve partial file reads. The blocks are subsequently RS-encoded by applications, and then the storage nodes store the chunks of data in them, as shown below.

[Asnc]

The data that will be lived longer is normally encoded using RS(15,9), while the data that will be lived for a shorter period of time is usually encoded with RS(6,3). In RS(n,k), n is the total number of bits after encoding, whereas k is the original data bits, and its decoder can perform error correction up to t symbols where t= (n-k)/2. In RS(15,9), where n=15 and k=9, t will be (15−9)/2 =3. This means that the encoding for long-lived data with RS(15,9) will perform error correction up to 3 symbols anywhere in its basic data unit. In RS(6,3), where n=6 and k=3, t will be (6−3)/2 which is 1.5. This means that the encoding for short-lived data with RS(6,3) will be able to correct about 1.5 corrupted symbols per basic data unit.

[The Reed-Solomon packet containing data and parity bits]

```
Note: It is a well-known bound from the coding theory that for RS(n, k) decoding is only possible if (n−k)≥2t, where k was the original message size. n is the code words size after encoding and 2t are the parity bits. Some notations use RS′ (10,4) (where 10 is the size of the original data, and 4 represents 2t parity bits) to explicitly tell about 2t. We will use RS(14,10) in our notation. Usually, it is clear from the context which notation is being used but we should be careful.
```

Storage space, network bandwidth, and disc IO are all reduced when writing entire blocks that have been RS-encoded. Since less data is written overall on the ChunkStore, the storage, bandwidth, and disc IO are reduced. For a tiny write request, whose quantity of data to be transferred is substantially less than the data replication, more IOPS are required to write chunks to 15 discs in RS(15, 9). By employing larger block sizes, the bottleneck for full-block writes is now disc bandwidth rather than IOPS, leading to more effective disc IO.

Applications can write a file’s blocks concurrently and asynchronously using the write-once-read-many technique, greatly reducing file write latency. After the file’s blocks have been written, the file’s metadata is completely updated. Since a file is only accessible when it has been entirely written, there is no chance of inconsistent behavior using this approach.



### Hedged quorum writes
Hedged quorum writes are used to improve tail latency. With no additional IO required, Tectonic’s full-block writes use a type of quorum for write operations that decrease tail latency. Rather than sending the chunk to write further on the storage nodes, Tectonic only sends reservation requests. By doing so, the nodes which accept the request of reservation first will perform that write operation by storing the chunks. Such behavior of reservation is closer to hedging. However, the reservation prevents sending data to such nodes, which will reject the write request due to a shortage of resources or because the requester has used more resources on that node than it was allocated.

For example, if the Client Library needs to write to three storage nodes, it can send the request to five nodes and can pick the first three that respond first.

```
Hedged quorum write: When a specific number of storage nodes send the acknowledgment of a successful write operation, this process is known as a quorum.
```

```
Tail latency: It represents a high response time (more than 98%) of requests from all the requests that are being handled by the application or service, which is a high percentile latency.
```

```
Reservation requests: A smaller request just as a ping from the system to a storage node to confirm whether the node is available for the write operation.
```

[Hedged quorum writes]

With a highly loaded cluster, the hedging method becomes more effective. In an empirical study, after applying hedged quorum in a test cluster with 80% of throughput utilization, they experienced approximately 20
% improvement in the 72MB full-block write and 99th percentile latency for RS(15,9)-encoding.

```
The study is Taken from the paper: Pan, Satadru, Theano Stavrinos, Yunqiao Zhang, Atul Sikaria, Pavel Zakharov, Abhinav Sharma, Mike Shuey, et al. Facebook’s tectonic filesystem: Efficiency from exascale. In 19th USENIX Conference on File and Storage Technologies (FAST 21), pp. 217-231. 2021.
```


## Optimizing blob storage
With the increase of objects, it becomes challenging to index all the objects in blob storage. Log-structured files are used to store the blobs by appending them to the file. Blobs are located using a map that links the blobID to the blob’s position in the file. Since the blobs are smaller in size, less than block size S, we’ll perform partial-block operations for blobs rather than full-block operations.

To optimize blob storage, we need to optimize how hot (blobs that are actively read) and warm (blobs on which read activity has substantially decreased) blobs are being stored. We can do so in the following two ways:

1. Consistent appends on partial blocks: They help decrease the latency of hot blobs since they are needed to be accessed more frequently. Since this data is being more frequently accessed, it is better to get it fully replicated consistently.
2. Re-encoding blocks: They help with the compression of warm blobs, which will rarely be accessed. Since this data is rarely accessed, it is better to get it re-encoded and rewritten using RS to achieve maximum compression.

### Consistent appends on partial blocks
Blob storage also entertains multiple user requests, so it is better to have low latency. The size of a blob B is smaller than the block size S, so one block can store multiple blobs. Whenever these small blobs come for write, they are written on the partial block, and we perform replication on that partial block append for the low latency. To make our blobs available right after the write operation, we make the partial block append read-after-write consistent. This partial block replication takes more space on the disk than earlier discussed RS-encoding of full-block data.

Partial block quorum appends provide us durability, low-latency, and consistent blob writes. Two scenarios in which the temporary decrease of durability is accepted by the quorum write are as follows:

1. The block will be re-encoded soon.
2. Another data center keeps the second copy sent by blob storage.
Even partial block quorum appends can cause issues such as incomplete or missed append operations that can cause different variations of the chunk replicas of different sizes if not handled carefully. For example, we perform 3× replication of the append operation performed by 3 different clients A, B, and C. Client A commits 512 bytes of the append operation, but unfortunately, the second replica fails. Similarly, Client B also commits 512 bytes of the append operation, but the first replica fails this time. Client C also commits 512 bytes of the append operation, but the third replica failed this time, as shown below.

[Consistent appends on partial blocks]

Now, all the chunks are of the same size – 1MB. However, each copy is different than the other one. The actual size of the append, which is 1.5MB, is different from all the stored chunks. When the reader comes to read the file, expectations would be to read a 1.5MB chunk based on the information provided by the metadata, but none of these chunks have a size of 1.5MB. Giving any chunk to the reader will contain inaccurate and incomplete data.

To avoid this, we have to control who is allowed to append the block and when those appends will be visible by using the following four steps:

1. We only allow the writer that has created the block to append that block.
2. We commit the updated block size S and checksum to block metadata when the append operation is completed before acknowledging the partial block quorum append. After the commit, the replica of the block up to size S is stored on a minimum of 2 storage nodes. The clients with the read request can read the contents of the block up to being the offset) when the acknowledgment is generated by the system and will be updated in block metadata. 

A single appender with this ordering of operations provides consistency.

[The blob storage write latency]

[The blob storage write latency]

```
Pan, Satadru, Theano Stavrinos, Yunqiao Zhang, Atul Sikaria, Pavel Zakharov, Abhinav Sharma, Mike Shuey, et al. Facebook’s tectonic filesystem: Efficiency from exascale. In 19th USENIX Conference on File and Storage Technologies (FAST 21), pp. 217-231. 2021.
```

The fact that Haystack and Tectonic’s blob storage have similar read and write latencies (as shown in the graphs above) confirms that Tectonic’s generality does not significantly affect performance.



### Re-encoding blocks
As we already discussed, applying RS-encoding on full blocks is IO efficient, but small partial block appends are inefficient. Therefore, instead of RS-encoding after each small append, we use three-way replication until a full block of data is written. When a block is full, it is sealed for any further mutation. At that point, our system reads the block from the replicated storage and RS-encodes it for storage efficiency. The number of IOPS used this way is fewer as compared to IOPS that would have been needed had we RS-encoded after each append.

[Re-encoded]

In this lesson, we learned about the optimization of storage (non-ephemeral resources). In the upcoming lesson, we’ll learn about how Tectonic performed during production and the tradeoffs between simplicity and performance.

