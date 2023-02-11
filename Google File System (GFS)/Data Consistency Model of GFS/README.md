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


## Consistency assurance by GFS
### Random write
### Record append
## Summary
