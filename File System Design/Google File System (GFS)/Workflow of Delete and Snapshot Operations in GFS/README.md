# Workflow of Delete and Snapshot Operations in GFS
In the previous lessons, we’ve discussed the GFS design approach and the workflow of the create, read, and write operations. This lesson will cover the workflow of two remaining operations—delete file, and snapshot file/directory.

## Delete file
The files on GFS are huge, implying that a file will have several chunks spread across the multiple chunkservers. Moreover, each chunk is replicated on multiple chunkservers for availability. Deleting so many chunks from multiple chunkservers while holding the client's delete request will add substantial latency to the client. If any of the replicas are temporarily down, we have to wait for them to recover to delete the chunk. This will produce an unnecessary wait on the client side. So, the file system implements a service called garbage collection. This service deletes the chunks but responds to the client immediately after marking the deleted file in the namespace tree maintained at the master. This is shown in the following illustration.

[Delete Files](./delete)

The garbage collection service regularly scans the namespace on the master to find out the files that have been marked deleted and deletes the metadata for such files. The chunkservers share the chunk handles for the chunks that they hold with the master through heartbeat messages regularly. If the master doesn't find a chunk handle in the metadata, it informs the chunkserver about it, and the chunkservers delete such chunks.

## Snapshot file/directory
A snapshot operation creates a copy of a file or a directory tree almost immediately and at a low cost. A copy of the file means we have a copy of all of its data chunks. Consider if someone performed a write operation on a file while a snapshot was also being taken then the snapshot will result in an incorrect copy of the data. The concurrent snapshot operations might result in inconsistent data among the copies of the same file. We need to stop all the write operations for the file/directory to perform the snapshot operation. We also have to ensure that the source and the destination directories are not deleted while the snapshot operation is in progress.

To be careful with the above problems, we need to temporarily disallow some operations on the directories or the files involved in the snapshot operation (via locks). For example, if we are taking a snapshot of a file with the full path /dir_src/file_1 and we are saving the snapshot at path /dir_des/file_1, we have to acquire the read locks on source and destination directories and write locks on the file to perform the snapshot operation both at the source and the destination. This is illustrated below.

[Snapshot](./delete)

The read lock on the, /dir_src and /dir_des directories stops others from deleting or renaming them. The master node in the GFS makes sure that these directories are not deleted or renamed in the namespace tree if a snapshot operation on these directories is in progress.

The write lock on the file /dir_src/file_1 file prevents any mutations on the file if the snapshot operation is in progress. The master node in the GFS does this by revoking the leases for the chunks that belong to the file undertaking the snapshot operation. The replicas holding the lease complete the mutation in progress and release the lease. For the next mutations, the client needs to contact the master for the chunk lease, and the master will hold the client until the snapshot operation is complete. Write the lock on the file name /dir_des/file_1 stops the creation of the similar file name in the destination directory.
```
Note: Clients can have multiple read locks, while write locks are exclusive. Write locks can’t be acquired if there is already a read or write lock on the resource.
```

```
Question
Do we need to instantaneously duplicate the data for all chunks for the file that is undertaking a snapshot operation?

Answer
We don’t need to duplicate the data unless it has changed. The master in the GFS duplicates the metadata for the snapshotted file. In this case, a chunk handle in the metadata will be pointing to two different files (more than two if more snapshots of the same file are created).

Suppose a client requests a write operation on the chunk that is pointing to more than one file in the metadata. In that case, the master first generates a new chunk handle, duplicates the chunk data for it, and then performs the write operation on the chunk associated with the requested operation in the metadata.

The mechanism above is called copy-on-write (COW). This idea is borrowed from Linux’s fork system call.
```
