# Data Consistency Model of GFS
## Data consistency
In a distributed file system, we keep multiple copies/replicas of file data for availability purposes. If such systems allow data mutations, then data inconsistencies among replicas tend to appear for various reasons. For example, a node failure causes a mutation to fail on one of the replicas, making the data inconsistent among replicas; one replica will contain stale data while others have been updated. Clients reading data from multiple replicas will get different data, which can have consequences depending on the specific use case. Therefore, the file system that allows mutations should ensure data consistency.

In GFS, data inconsistency starts from a write or an append operation that changes the file’s content. Each file has three replicas by default. The mutation should be applied to all the replicas, otherwise an inconsistency issue arises. Let’s assume that we have a file that consists of multiple chunks, as shown in the illustration below. We’ll call the part of the file where the mutation is applied a file region.

If we have data that is consistent among all replicas, we can still face another problem due to concurrent writes on the same region. Since the file system allows multiple writers to write on the same region, the file region may mix data from multiple writes. Therefore, a file system should handle all these cases and provide its users with some guarantees so that the users don't see unexpected results. We can think of implementing a strong consistency model here to make the data consistent among all replicas. However, our system has to serve many requests at a time, and a system that provides strong consistency might compromise the system's performance. We need to provide consistency with good scalability and good performance. GFS's data consistency model is one of the most involved parts of the system. This increases its difficulty level. In this lesson, we will see what data consistency guarantees GFS offers its users and how it meets all their requirements. Before this, we need to know about the possible states of a file region after data mutation. Let's define these states first.


## States of a file region after data mutation
The state of a file region can be consistent or inconsistent, and defined or undefined after a data mutation.

- Consistent: A file region is consistent if a client sees the same data on all replicas after a mutation. In the illustration below, the left part shows that all the replicas have the same data. Thus, the clients reading from any of the replicas will read the same data.

- Inconsistent: A file region is inconsistent if a client sees different data on multiple replicas. In the illustration below, the right part shows that one of the replicas has different data than the other two. A client reading the same file region from three replicas will find different data.

[Consistency vs. inconsistency]

- Defined: After mutations to a file region (possibly concurrent), if the region has properly changed data such that applications can parse and read it as per application-level record format, that region is defined. This is shown in the following illustration.

- Undefined: After mutations are made to a file region (possibly concurrent), if the region hasn’t properly changed data such that applications can parse and read it as per application-level record format, that region is undefined. This is shown in the following illustration.

```
Note: By definition, a defined region is a stronger condition than a consistent one. If a region is defined, it means that it is also consistent. Due to local contrapositive, if a region is inconsistent, then it is undefined as well. A consistent region doesn't imply that it is defined; it can be undefined as well.

In symbols:

defined⇒consistent
inconsistent⇒undefined
consistent⇏defined
```

## Consistency assurance by GFS
The specific state of a file region depends on the type of mutation (write/append). It also depends on how the system carries out the mutations (serial/concurrent) on that region, and whether the mutations result–in success or failure. Success ensures that the data is consistent among all replicas, while failure (except the ack failure where the mutation is actually done on all replicas but the reply from any of the replicas is lost) makes the file region inconsistent among replicas. A successful mutation includes retries by the client in case of failure on any replicas. The defined or undefined regions come out as a result of serial or concurrent executions of mutations.

The following table shows the state of the file region against different types of mutations and how each type of mutation was carried out by the system.

```
                        Random Write                    Record Append

1. Serial success       Defined                         Defined interspersed with inconsistent

2. Concurrent success   Consistent but undefined        Defined interspersed with inconsistent

3. Failure              Inconsistent                    Inconsistent

```
1. Serial success means that the mutations were carried out in a sequence, one after the other, and successfully completed on all replicas.

2. Concurrent success means that the mutations were carried out concurrently (more than one mutation is being executed at the same time on one specific region) and successfully completed on all replicas.

3. Failure means that the mutation failed to execute either on the primary replica or the secondary replicas, even after retries. If it fails to execute at the primary and is not therefore forwarded to the secondary replicas, then the file region will remain consistent. If the mutation has succeeded at the primary replica but failed at any of the secondary replicas, it will leave the file region inconsistent.
Let's analyze how the table above holds true.

### Random write
1. Serial success: The serial success of random write operations ensures that the region is defined and, therefore, consistent. A write on the same region actually overwrites the data in that region. If these operations are executed in a sequence on each replica, each write operation overwrites the data written by the previous write operation, and at the end, the region contains the complete data written by the last write operation. The successful serial execution of random writes will leave the region defined and, by implication, consistent. It is important to note that, in case of a failure on any of the replicas, the successful execution of an operation involves a limited number of retries by the GFS client. Retrying the operation will apply the mutation on all replicas, including those that had already applied the write on them. Since a write operation on a region overwrites what was already written, the retry won’t cause any inconsistency among replicas.

2. Concurrent success: The concurrent success of random write operations on a region results in a consistent but potentially undefined region. This is because one write operation can interrupt the other, resulting in mingled fragments from multiple writes. An example is shown in the section above, states of a file region after data mutation, where we described an undefined region.

3. Failure: The failure of serial or concurrent random write operations results in inconsistent data among multiple replicas.
### Record append
Both the serial or concurrent success of record append operations result in defined regions interspersed with inconsistent regions.

1. Serial success: One might think about how the serial execution of record append operations produces inconsistent regions in between the defined regions and why the regions are not just defined.

The record append operation writes data at the system’s chosen offset, which should be higher than the end of the file data. To report success for an append operation, data must be appended at the same offset on all replicas. If the operation fails on any of the replicas, the client retries the operation on all replicas. It produces record duplicates at the replicas that have already performed this operation successfully and empty spaces/padding at the replica that previously failed to perform the operation, as shown in the illustration below. This is why the serial success of the append operations produces inconsistent regions in between the defined regions.

2. Concurrent success: One might also think about how concurrent execution of record append operations results in defined regions that are intermixed with the inconsistent regions. Why don’t they result in totally undefined regions, as in the case of random writes?

In the case of random writes, the offset at which the data is to be written is provided by the clients, while in the record appends, the client just pushes the data, and the data is written at the offset chosen by the system. The data from current appends will be written at the different offsets, so there is no chance of data overlapping; thus, it results in defined regions.

3. Failure: The failure of serial or concurrent record append operations results in inconsistent data among multiple replicas.
## Summary
In this lesson, we’ve covered how GFS guarantees data consistency. GFS doesn't provide a strong consistency model to achieve good performance and scalability for client operations. It provides the clients with a relaxed consistency model. The guarantees are provided in terms of defined, undefined, consistent, and inconsistent regions based on the type of operation performed on a file region. These guarantees show that GFS is more suitable for append operations. So, GFS is a good solution for applications that append data to files, like logging systems, web crawlers, MapReduce, etc.

Since GFS provides different consistency guarantees for different operations, it depends on the applications to use GFS and deal with inconsistencies and undefined regions if they choose to use the operations that cause these issues. Some inconsistencies have to be handled by the GFS itself and the rest is on the applications that use it. We’ll look into how GFS or the applications deal with data inconsistency issues in the next lesson.
