# Analysis and Evaluation of Two-Phase Locking (2PL)
## Handling of deadlocks
A situation might arise where transaction A is waiting for a long time for transaction B to finish, since we use a lot of locks simultaneously. In general, we refer to the situation where a cycle of transactions waits for one another to release locks as a deadlock.

[A deadlock]
The database automatically identifies deadlocks among the transactions. To resolve them, it relies on two approaches—detection and prevention.

### Deadlocks detection
The database management system (DBMS) produces a waits-for graph, where transactions will form the nodes. Consequently, an edge will form between two nodes, Ti and Tj, if Ti is waiting for the release of a lock held by Tj. The system will frequently inspect the waits-for graph for cycles before determining how to break them. The system will have a deadlock only if there is a cycle in the waits-for graph.
```
wats-for graph: A directed graph used for deadlock detection.
```
[Detection]

- The DBMS will choose a victim transaction to roll back when it notices a deadlock to break the loop.
- Depending on how the application invoked the transaction, the victim transaction will either restart or abort.
- When choosing a victim, there are many transaction properties to consider. There isn’t a decision that is superior to another. All 2PL DBMSs carry out several tasks in the following ways:
    1. By age (newest or oldest timestamp)
    2. In order of query execution progress (least/most)
    3. Based on the number of locked objects
    4. Based on the number of transactions, we need to roll back along with it
    5. The number of times a transaction has previously been restarted

### Deadlock prevention
## Shortcomings
### Poor throughput and high query response time
### Unstable latencies
### Frequent deadlocks
## Conclusion
