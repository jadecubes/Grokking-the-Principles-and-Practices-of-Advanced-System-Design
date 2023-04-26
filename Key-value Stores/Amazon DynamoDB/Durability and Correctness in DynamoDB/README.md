# Durability and Correctness in DynamoDB
A large scale database service can run into issues. Faulty code and hardware failures, among other causes, can result in corrupted data. We need to ensure that our system is resilient to these faults. This often requires regular checks and balances that can help us better take care of customers' data. We will aim that data is never lost after it has been committed.

## Hardware failures and backups
Modern data centers use commodity servers. Over the years, the reliability of these nodes has substantially improved. Though, they still fail in one way or the other. We will incorporate measures for situations in which data loss can occur due to hardware failures.

```
A commodity computer is easily available and at an attractive price point for an organization
```
A write-ahead log is very important in our design. It is a log-based (sequential writing in storage with index in memory) data structure that allows us to store large incoming data. The log stored in the storage is essentially the complete data received from incoming requests. The in-memory index helps to access that data in constant time. Even if we lose the in-memory index due to memory failure, we can still use the write-ahead logs to recreate the index.

[The in-memory index can be recreated using the write-ahead log from storage. By regularly backing up the write-ahead log, we can minimize data loss and recreate the write-ahead log in case the storage of a node fails.](./d.png)

We will regularly archive our write-ahead logs. So, in the event of a node failure, we can recreate the partition using the archived write-ahead log.

However, in case of a replica node failure, the repair process can take a while since replacing a replica involves copying the write-ahead log and the memory index. One solution is to introduce log replicas in partition groups. A log replica only stores the write-ahead log. Log replicas are like acceptors in Paxos.

As soon as a leader replica of a replication group detects an unhealthy node, it backs up its write-ahead log in a node and adds it to the replication group as a log replica. This maintains the same level of durability as the replication group would in the absence of the unhealthy replica. This process is quick since only the unarchived write-ahead log needs to be backed up.

## Silent data errors
Hardware failures can also cause incorrect data writes. These are hard to track because they can occur anywhere in the network without indicating hardware failure. The faulty server is working fine because it is serving acknowledgments and participating in replication.

We can add checksums to avoid faulty data writes. The frequency of verifying data using checksums can vary based on the historical performance of the hardware. The hardware, known to fail more frequently, might require more frequent checksums. We can have checksums for a single log entry or the entire log. Using checksums prevents incorrect data from spreading throughout the replication group.

We also need to introduce checksums before archiving data. We should only archive verified data. We can do this by maintaining information about the unarchived write-ahead log—table name, partition ID, key range, and entries at the start and end. We can use this information to generate checksums. We can also have other checks, like verifying that every entry in the log belongs to the correct table and partition.

We can use archived data to verify write-ahead logs. If a node realizes its write-ahead log is already archived, we can use archived data to verify that the replica has the correct write-ahead log. If not, this means that the node could be faulty.

```
Note: After decades of relentless chasing to extract more performance from silicon-based processors, we've reached a point where it is becoming increasingly challenging to guard against the possible data and calculation malfunctions. Hyperscalers like Google and Facebook have reported a new category of silent errors due to silicon level defects. See Detection and Prevention of Silent Data Corruption in an Exabyte-scale Database System and Silent Data Corruptions at Scale for more details.

https://storage.googleapis.com/pub-tools-public-publication-data/pdf/cdecb8fc0a5f5705af0688eeff72dbd6a353f274.pdf
https://arxiv.org/pdf/2102.11245.pdf
```

## Continuous verification
Our design should carry out data verification of data at rest. These can be scheduled, and their frequency can vary. Such verification can help identify silent errors like bit rot, which is a non-functioning bit.

## Software bugs
Developing a large complex system requires a lot of code. While the size and complexity of the code depend on our implementation and tools, it is still subject to human error. We can reduce code errors by thoroughly planning our code using formal methods and extensive testing of our protocols. Testing is also important while deploying upgrades as it can reduce the chances of deploying buggy code.

Failure injection and stress tests can help us better understand the resilience of our code.

## What’s next?
In the next lesson, we will discuss some design features that can help maintain high availability in our design.

