# Workflow of Create and Read File Operations in GFS
Since the GFS is a system that stores and retrieves data from files at a large scale, we must know how it performs different file operations. We listed the file operations that it supports in lesson of GFS File Operations. This lesson focuses on the create and read file operations. The rest of the operations are discussed in the upcoming lessons.


## Create a file
To create a file, the client needs to connect to the master node. The master creates a file in the directory specified by the client and updates the namespace. Many other operations can be performed by different clients on the same directory concurrently. Master needs to manage such concurrent requests. The master acquires two types of locks to guarantee the atomicity and correctness of file creation operation.

```
The GFS namespace is a lookup table that contains the full pathnames of files and directories and the associated metadata. Each pathname is stored in a separate table row, and the mapping to the metadata is recorded in the same row in a different column.
```

A read lock is acquired on the directory name (full path to the directory) to make sure that the directory in which the file is being created is not deleted or renamed by another client during the file creation operation.

A write lock is acquired on the file name (full path to the file) to serialize the creation of two or more files with the same name.

The master releases the locks acquired by an operation when it is done with that operation and has responded to the client. The following illustration shows the workflow for file creating operations.

[Create](./create)

GFS allows multiple metadata operations to be performed at the same time by using fine-grained locks. For example, the master doesn't lock the whole namespace to perform an operation. Instead, it just acquires locks on a small region of the namespace where the operation has to be performed. Other operations that have nothing to do with that region will not be delayed. They will acquire locks on their regions and be executed at the same time. The master uses two types of locks: a read lock, and a write lock. We can acquire a read lock on a region that is already read-locked but we can't acquire a write lock on a region that is already write-locked. Operations that just need read lock can run concurrently. One operation doesn't need to wait for the other operation to release the lock. Operations that require a write lock on the same region (conflicting locks) are serialized properly.
```
Note: Locking resources can decimate the performance if not done carefully (for example coarse-grained locks, or heavy contention on locks by concurrent clients). Fine-grained locking often provides better performance and scales well with increasing requests (the hope is that after fine-grained locks, clients requests will be spread out to many different locks).

```
```
Question
What happens if the master dies after locking something?

Answer
Since the information about the locks is stored in the master’s memory, the locks’ state is lost if a master fails. The master writes metadata mutations in persistent (local disk and remote storage) storage and then performs the operation. If a mutation is in the log, the new master will act on it by taking the required locks anew. If the master fails before writing to the log, clients need to retry the mutation later. Client connections to the failed master can time-out, in which case clients might not be sure if the mutation happened or not. For most such cases, a retry from the client will be required.
```
Deadlocks are a concern whenever locks are involved. Deadlocks are avoided by acquiring locks on the directories path and file name path in order, from left to right of the full pathname, as shown in the following illustration. Therefore, the master always takes locks for all clients in a well-specified left-to-right order.


[Locks are acquired in an order to ensure that both aren’t waiting on each other and no deadlock is produced](./lock.jpg)


In operating systems, there are four necessary conditions if a deadlock happens:

1. Bounded resources: Only a finite number of client requests can access a resource concurrently.

2. No preemption: Once a lock is acquired, its ownership can only be changed by the thread that acquired the lock.

3. Waiting while holding locks: When a client needs multiple locks, it acquires one, then waits for the next ones to be acquired by keeping the previously locked resources.

4. Circular waiting: There is a circular wait between different threads.

The four conditions listed above are necessary for a deadlock. This means that if we don't let any one of them happen, we can prevent the deadlock. GFS primarily targets circular waiting. With its well-defined locking order, it never lets a circular waiting happen and avoids deadlocks.
## Read file
To read data from a file, the user request is first processed by the GFS client which identifies the chunk index in order to reach a specific chunk that contains the required data. As we know, the file is divided into fixed-size chunks; the chunk index can easily be computed using the offset provided by the user's application. Let's take an example of a video file of size 200 MBs. On GFS, it is stored in multiple chunks, each consisting of 64 MBs. The last chunk may have some bytes that are not filled when the file size is not a multiple of 64. The following illustration shows the metadata that the master stores for each file.

[Metadata stored at the master node (file to chunk mapping, and chunk metadata consisting of chunk IDs and their locations)](./master.jpg)

Let's say the user wants to read the data starting from the 64^{th} megabyte, as shown in the following illustration. The GFS client can easily find that it is the very initial part of chunk 2 (64–127 MB).
```
Note: The large streaming read operations at GFS read about 1 MB of data per read operation.
```

[File1: The red line shows the data that is already read, and the black line shows the data that is requested](./file1.jpg)

In the illustration above, the 64^{th} megabyte is the starting point of the data the client wants to read. The total data to be read is one MB. The chunk index is calculated by dividing the starting byte value by the chunk size and adding 1 to it, which turns out to be chunk 2 in this example. The GFS client also calculates the byte range for the data to be read from chunk 2. When an end-user performs a read operation, it considers the file stored as a whole, so the starting byte the user wants to read data onward is based on the complete file size. For example, 64^{th} megabyte of the file_1 is the 0^{th} byte of chunk 2 and if we have to read 1 MB of data, the byte range will be(0–1) megabyte on chunk 2.

The following illustration shows the workflow for a read operation. For simplicity, we use the variables W, X, Y, and Z instead of the actual IP addresses for the chunkservers.

[Read](./read)

```
The metadata is cached at the client side to perform further read operations on the same chunk without master interaction that help lessen burden on the master. The large chunk size also helps in reducing the load on the master.
```
For processing large datasets, a large number of streaming read operations are performed that require the file system to serve a large amount of data at the same time. With the architecture designed above, the read requests for multiple files and multiple chunks are distributed among multiple chunkservers, which together serve large amounts of data at the same time, improving the overall throughput of the system.

For streaming reads, the master also sends the metadata for some chunks contiguous to the one that is actually requested. This helps clients read the data faster as the client already has the metadata for the next chunk.

Performance concerns arise from small random reads. Applications that prioritize performance frequently batch and sort their small reads in order to progress continuously through the file as opposed to back and forth (and possibly pre-fetching metadata information ahead of the time the user needs the data).
