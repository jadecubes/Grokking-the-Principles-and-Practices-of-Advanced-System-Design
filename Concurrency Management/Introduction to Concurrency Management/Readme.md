# Introduction to Concurrency Management
## Motivation
Concurrent activities are common in a distributed system where thousands of client requests might be coming into service and clients might be expecting sub-second reply latency. Concurrency also comes into play when we need to parallelize different aspects of a request (for example, fetching relevant ads while data is being searched on a search engine). However, concurrent programming has a reputation for being difficult, where bugs can be subtle and might exhibit in strange ways at the most inconvenient time.

Different concurrency control mechanisms help us reason for the correctness of the concurrent activity.


## What we will learn
- We have selected the following three papers to discuss in the following few chapters:

- [2PL] Two-phase locking

- [Chubby] Burrows, Mike. The Chubby lock service for loosely-coupled distributed systems. In Proceedings of the 7th symposium on Operating systems design and implementation, pp. 335-350. 2006.

- [ZooKeeper] Hunt, Patrick, Mahadev Konar, Flavio P. Junqueira, and Benjamin Reed. ZooKeeper: Wait-free Coordination for Internet-scale Systems. In 2010 USENIX Annual Technical Conference (USENIX ATC 10). 2010.

## Why did we choose these systems?
The concurrency control systems can be divided into two broad categories—optimistic concurrency control and pessimistic concurrency control. The philosophy of optimistic concurrency control is that, if we know beforehand that contention on some concurrent data object will be low, the operation will succeed in the common case. For a few cases when concurrent activity actually happens, we roll back and retry. However, for highly contended data objects, optimistic concurrency control incurs a lot of overhead (due to repeated aborts and retries). For such scenarios, it is better to first acquire locks on required objects for exclusive access and then do the operations—pessimistic concurrency protocol.

Two of our systems are based on pessimistic concurrency control, while the third enables the clients to implement optimistic concurrency control.

### The 2PL system
This is a classic concurrency control scheme, originally invented and used in IBM’s system R database. Most current databases use some variant of 2PL, and it makes it necessary to learn it. To deal with the tricky corner cases of deadlocks, there are intriguing 2PL rules and associated deadlock prevention and correction measures.

### The Chubby system
Chubby is Google’s distributed locking service that can provide concurrency control to thousands of clients. Chubby is at the heart of many other Google services, such as Bigtable. Chubby exports its services using an API that resembles the Unix file system API. All clients see a consistent state because all reads and writes are done at a leader (master) that has Paxos-based replication.


### The ZooKeeper system
The ZooKeeper system takes a different approach. Instead of directly providing locks, it provides some primitives on which more complicated abstractions can be built. To speed up operations, clients have the choice to read from the replicas if clients can live with a somewhat stale state. In other words, this system provides an option for the clients to speed up operations at the expense of looser consistency guarantees or get strong consistency at a higher delay. It has been a recurrent theme for many data stores, but it is interesting to see it again in the context of concurrency control systems.

We hope our selection of concurrency control systems teaches us many important lessons in system design. Let’s dive in!

[Timeline of the evolution of concurrency management systems](./2pl.png)
