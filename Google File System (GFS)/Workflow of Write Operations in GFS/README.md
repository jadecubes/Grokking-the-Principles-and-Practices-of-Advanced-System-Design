# Workflow of Write Operations in GFS
We’ve discussed how GFS carries out file operations, and we have seen the two of them (create and read files) in the previous lesson. In this lesson, we will look into what kind of write operations GFS supports and the workflow of these operations.

## Writing data to the file
Data can be written at a random position in the file or appended to the file in case of sequential writes. GFS offers two operations–a random write and an append operation. In a random write operation, the client provides the offset at which the data should be written. In an append operation, the data is written at the end of the file at the GFS chosen offset.

For write operations, the GFS client needs to know which chunk they will be writing the data to. We’ve already seen that the master keeps three replicas for each chunk by default. Among these three replicas, the master gives one the lease. The replica that has the current lease acts as the primary node, while others act as secondary replicas.




All write operations from the clients are carried out by the replica that has the current lease, called the primary replica. The primary replica then forwards the request to secondary replicas and replies to the client once the request has been processed. Since it is the master who decides which replica will take the lease, therefore, replicas don't need any election algorithms to choose a primary between them.
```
A lease is a contract signed by one of the replicas at a time to act as the primary node for a given time duration (60 seconds and can be extended if some operation is in progress).
```

```
Note: Leases are a fault-tolerance mechanism that enable the master to provide good availability under different kinds of failures. Lease value should not be either too large (in which case if a server takes a lease and dies, master might need to wait out for the lease to expire before it can give it to some other server) or too small (in which case master might get excessive traffic for lease renews). A good lease value usually depends on specific use cases and evolves with the operational experience with the system.
```
```
Question
How are leases easing the availability of the system?

Answer
Since the master has the authority to choose the primary replica for the clients, it grants a lease to a healthy replica, not a failed replica. Still, it is possible that everything was okay when the master handed out the lease, and a client was about to contact the primary (the leaseholder) when the primary died. The client will go back to master because the connection/request will fail/time-out.
```
Lease values are usually small, like a few seconds to a few minutes. If the master doesn’t do anything proactively, the lease will expire, a different replica will be given the lease, and a new node will be found to reach the replication factor of 3. This helps GFS to be available for the client requests with some small windows of unavailability (during lease reassignment and finding a new node to have enough replication factor) for writing to a specific chunk (not reading because reading can happen from other replicas).

In the next section, we will discuss what can go wrong if one of the replicas was not designated as a primary replica.

## Problems with not designating a primary replica

Suppose the master allows the clients to perform a write operation on any random replica. In that case, multiple write operations on the same chunk will be performed by different replicas simultaneously. Each replica that receives the write request from the client also has to propagate the write to the other replicas. These replicas can get the propagated write operations out of order. Each replica executes the same set of operations in a different order. Thus, the replicas will contain different data. However, the replicas should actually be identical. The following illustration depicts this issue through an example.



Serialization is required to cope with the data inconsistency issue among the replicas shown above. It is challenging to serialize write operations across multiple replicas. The master has to do a lot of work to manage all this. So, the master uses the lease mechanism to reduce this management overhead. With a lease, all the write operations are carried out on a single replica at a time, which makes it easy to serialize operations on a single chunkserver. Let's see how it works by looking into the workflow of a write operation.

## Workflow of write operations
The workflow for both the random write and the append operation is almost the same and is illustrated below. The only difference is the chunk to which the data is written.

- In random writes, the clients provide the offset at which the data is to be written. The client can map the offset to a chunk index if each chunk contains all 64 MB (the maximum data a chunk holds/the size of the chunk) and there is no padding. This is internal to GFS. It is also possible that it uses the byte range instead of an index to deal with the padding. The chunk is found using the client-provided offset.

- In append operations, the client doesn't need to provide the offset. The data is pushed to the last chunk.

Now, all of the write operations from multiple clients will be directed to the replica that has the current lease for the chunk. The replica holding the lease is the primary replica. The primary replica will serialize all of the write operations that are to be performed on the chunk. It then performs the operation in the order of serial numbers assigned to each operation. All of the secondary replicas execute the write operations in the same order.

The successful serialized execution of write operations on the replicas makes sure that all the replicas contain the same data. If any of the replicas fail to perform the write operation on it, then that replica will lag behind. If the clients read data from that replica, they won’t get the updated data. We will see how GFS copes with this issue, as well as others, in the data consistency model lesson.
## The edge cases
The append operation writes data to the end of the file. To perform the operation, the client asks the master for the last chunk's metadata (chunk handle, and chunkservers). The master node looks into the metadata for the last chunk of the file and responds to the GFS client with the required metadata. The client then sends data to the chunkserver holding the last chunk. There are three different scenarios the client has to deal with depending on the available space on the last chunk and the total size of the data to be written. Let's look at each scenario.

1。Suppose the last chunk has available space for appending new data. In that case, the chunkservers write that data in the specific chunk returned by the master and respond to the client with a success message, as shown in the following illustration.

```
Since each chunk is replicated on multiple chunkservers for availability, the primary replica has to ensure that the write operation is performed on all replicas.
```

2。If the last chunk already has 64 MB of data, then the chunkserver responds to the client that it is already full. The client then asks the master to create a new chunk. The master generates a new chunk ID, allocates the chunkservers where this chunk's data would be placed, and responds to the client with the new chunk's metadata. The client will then write the data to the new chunk. An example is shown in the following illustration.

```
One approach is for the chunkserves to inform the master about the full chunk in the last heartbeat message to save the client a round-trip to the master later on. On the other hand, delaying such a declaration to the master might reduce the immediate load on the master.
```

3。It is possible that the last chunk is not full, but doesn't have the capacity to accommodate all data bytes in the append request. In this case, the chunkserver holding the last chunk will respond to the client with a message that the available space in that chunk is less than the size of the write. The chunkserver likely adds information about how much available space it has on the last chunk, based on which the client will split the writing data into two. The first part of the split will be written to the end of the last chunk, and for the second part, the client asks the master to generate a new chunk. The remaining data will be written at the start of the newly created chunk.

## Decoupling the control and data flow
Decoupling the control and data flow helps efficiently use the network. We’ve seen in the workflow of a write operation that the control flows between the client and the master, from the client to the primary replica, and from the primary replica to the secondary replicas. On the other hand, data flows linearly from the client to the chunkservers. Each machine forwards data to the nearest machine on the network that has not yet received it. It does this to maximize the usage of each machine's network bandwidth.
```
Note: The network distance depends on a specific network topology (the way servers are connected in a network). Near might mean two servers in the same rack or minimizing the number of switches that must be crossed to reach each other. Usually, when we need to traverse more switches to reach the destination, the bandwidth might become less, and latency might increase.
```
The following illustration shows the control versus data flow.

[Control flow vs. data flow]

The linear data flow through a chain of chunkservers over a TCP connection avoids going through network bottlenecks and high-latency links while minimizing the latency to push all data.

## Summary
In this lesson, we learned how GFS handles write operations. We have also seen some edge cases that come up when writing and how those are handled. In the end, we highlighted the virtues of separating control and data flow. The next lesson will cover two further operations—the deletion, and the snapshot.
