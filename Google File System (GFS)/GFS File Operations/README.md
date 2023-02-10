# GFS File Operations
We saw the high-level architecture of GFS in the previous lesson, along with its system-level goals. Since it stores files, we should also know about the API it provides to its clients for storing, retrieving, or deleting their data. In this lesson, we will look into the GFS client API. Next, we will explain how GFS handles each of the client operations and how it meets its system-level goals.

The illustration below shows all of the operations that GFS clients can perform on files or directories.

[GFS API]

## Directory operations
GFS clients can perform the following simple directory-level operations. The semantics for these operations are the same as those of the Unix file system.

- Create directory: Users can create multiple directories to organize their files.

- Delete directory: Users should be able to delete a directory. The system should ensure this operation doesn't leave dangling data on the chunkservers. It should also ensure that all the files in the directory are deleted before deleting the directory itself. If the directory contains files, the system should ask the client to delete those files first.

- List directory: The users should be able to list what's inside a directory. In GFS, there is no inode-like data structure for representing namespaces. GFS represents a file with its full pathname. A lookup table is maintained that maps the full pathname of a file to its metadata, as shown in the following table. This table logically represents the namespace. The last component in the path can be a file or a directory. The rest of the path is all about directories. In the table below, listing a directory with path /dir1/dir2 should list the path names that are one name longer than it, for example, /dir1/dir2/fileA and /dir1/dir2/dir4 and so on.

```
The inode (index node) is a data structure in a Unix-style file system that defines file system object (FSO) model like files or directories. Each inode stores the attributes and disk block locations of the object's data. Attributes of a file or a directory object may include metadata like times of last change, access, and modification, with owner and permission data. A directory is a list of inodes with their assigned names. The list includes an entry for itself, its parent, and each of its children. [source: Wikipedia]
```

[Lookup table]

## File operations
GFS client API allows the users to perform the following file operations.



### Create a file
Multiple users can concurrently create files in the same directory. During this operation, it must be ensured that the directory where one user creates the file is not deleted by another user who has access to that directory.

The create file operation also involves handling multiple files being created with the same name. If there are N concurrent requests to create a file with the same name in the same directory, only one of the request is successful, while other N-1 clients will receive an error. Also, the operation is performed atomically. We’ll see how GFS does all this in the next lesson.

### Open file
GFS allows users to open the file containing the latest version of the data in order to perform read and write operations. When a client opens the file, the metadata is fetched from the master and cached on the client side. While the client is reading the data in that file, some of the data might become stale if another client is performing a write operation on the file in the meantime.

The reopening of the file by the client should fetch the latest metadata from the master instead of using the cached metadata.

### Read file
GFS workloads often comprise two kinds of reads– small random reads, and large streaming reads. Let’s look at the difference between the two below:

- Small random read: This kind of read accesses a part of the file data at a random position/offset. Some of the examples of random read operations include jumping/scrolling to a random page in a text document consisting of multiple pages, skipping some part of a video and starting watching from an arbitrary frame, and so on.

A few kilobytes of data are read in a small random read. These operations involve repeatedly going back and forth in the same file and possibly interacting with the master again and again for the chunk locations. Since the application doesn’t have any idea of which chunk the user is about to request, it can’t cache that chunk’s location. That’s why small random read operations may experience a large response time.

- Large streaming read: In streaming reads, the contiguous data of a file is accessed by consecutive read requests from the same client. An example of streaming reads on the client side is when they watch a video without interruption. Each such request reads one MB or more data. Large streaming reads are efficient as the client most probably has the chunk location in its cache.

```
Remember that the chunk size is 64 MB. If the client reads the first MB of a chunk, the chunk location is fetched from the master and cached on the client side. The next successive 59 requests of one MB for the contiguous data can be performed without interacting with the master for the chunk location. Large streaming reads require the system to serve large amounts of data at once. Therefore, a high sustained bandwidth is very important.
```
### Write file
GFS was designed for larger sequential writes (record append) than random writes. The data, once written by these applications, is rarely modified. Knowing this, GFS handles random write and record append separately to optimize the system performance for the later operation.




- Random write: This operation writes data on a random position in the file specified by the client. It can be data insertion or data overwriting.

- Record append: It is another name for large sequential writes. It appends data at the end of the file. If there are multiple appends to the same file, GFS puts all of them at offsets of its choice.

File mutations pose data consistency challenges as data is replicated for availability. It also involves handling concurrency and atomicity. We’ll discuss how GFS manages it in the next lesson.
```
Note: Concurrent append with a custom consistency model is a novelty of GFS (it is not present in standard Unix-like file systems). Additionally, it is an example of a design pattern: to make the common case fast, as mentioned in this book. Append operations are fast on most kinds of disks, especially for traditional rotating disks where continuous writes don’t have to incur seek latency on each write. By making a common operation (appends) faster, GFS makes a large proportion of the overall operations fast.
```

### Close file
Users should be able to close a file if they are done with the read/write operations.



### Delete file
Users can delete the files that they don’t need anymore. Since each file is stored in small chunks on multiple chunkservers and replicated, it will take some time to delete all chunks that belong to the deleted file and the associated metadata. GFS needs to have a mechanism that can control the waiting time for the users against the delete operation. We’ll see how GFS reduces delete latency in the upcoming lessons.

## Snapshot file/directory
Additionally, GFS allows users to duplicate files or directories quickly. While the file is being duplicated, the other users might make changes to it. The system should handle such scenarios carefully to make sure that the copy has the right data.

Let’s take an example to understand the right data for a snapshot. A client requested a snapshot operation at time t. The data at time t was x. While the snapshot operation was being performed, someone performed the write operation on the data being snapshotted, and the data changed to x′. The snapshot operation was completed successfully but might point to x' or x or an arbitrary combination of the two.


In this case, the snapshot does not contain the right data because the snapshot was taken for x. However, it might contain something else. The system has to stop all mutations at the time of the snapshot so that we have the right data in the copy.

```
Question
How are GFS operations different from Unix file systems?

Answer
GFS provides the standard file system operations with two additional operations unique to GFS: the snapshot and record append operation.

A snapshot generates a cost-efficient copy of a file or directory tree, enabling users to instantly create copies of large datasets. With the help of a snapshot operation, users are able to create a checkpoint of the system’s current state, prior to committing any changes so that they can easily revert back if needed.

The record append operation enables multiple clients to append their data concurrently in the same file while also guaranteeing the atomicity of each append operation. It facilitates the implementation of multi-way merge results and producer-consumer queues in which multiple clients are allowed to append without requiring any additional locks.
```
