# Concurrency Control in Spanner
In this lesson, we’ll learn how TrueTime guarantees correctness properties around concurrency control and how those properties ensure the following:

```
Concurrency control: In information technology and computer science, especially in the fields of computer programming, operating systems, multiprocessors, and databases, the concurrency control ensures that correct results for concurrent operations are generated, while getting those results as quickly as possible. [source: Wikipedia]
```

1. Externally consistent transactions
```
External consistency is a property of transaction-processing systems, where clients dynamically synthesize transactions that contain multiple read and write operations on arbitrary objects. [source: Google Cloud]
```

2. Reads from the past that do not block any ongoing transaction

3. Read-only transactions that do not require any lock

These features make it possible to ensure that at any time, t, the full database audit read will see the exact results of all transactions that are committed as of t.

We will use Paxos write and Spanner clients write to differentiate between both writes.

## Timestamp management
The implementation of Spanner supports reads from a snapshot and read-only and read-write transactions. The system implements the read-write transactions as standalone writes, whereas the read-only transactions are implemented as non-snapshot standalone reads. The following table shows the operation types Spanner supports and their respective concurrency control.

```
  Operation Types and Their Concurrency Control
Operation                                Concurrency Control

Read-Write Transaction                        Pessimistic 

Read-Only Transaction                          Lock-free

Snapshot Read, client-provided timestamp       Lock-free

Snapshot Read, client-provided bound           Lock-free
```

### Read-only transactions
The advantages of snapshot isolation are utilized by the read-only transaction. A read-only transaction requires pre-declaration of being read-only and not having any writes, since it is not a read-write transaction without any writes. Spanner executes reads within a read-only transaction without locking at a timestamp t
 which is determined by the system. Executing the read at t avoids blocking the incoming writes. This is an example of any suitably up-to-date replica that may handle the reads in a read-only transaction.

A snapshot read is where we read from the past without acquiring a lock. When performing a snapshot read, a client can either provide a timestamp themselves, or the system can determine the timestamp on the basis of an upper bound on the timestamp’s expiration provided by the client. A snapshot read can be executed on any up-to-date replica in either scenario.


[Snapshot reads do not require locks](./db.png)

Once a timestamp has been chosen, the commit is unavoidable for both snapshot reads and read-only transactions. The commit does not happen if the data has been garbage collected at that timestamp. Therefore, clients can escape a retry loop's buffering of results by receiving them immediately. Clients can continue their queries on other servers if a server goes down by sending the same timestamp and reading location.

## Paxos leader leases
Timed leases are used in our Paxos implementation for sustainable leadership, that is, by default, 10 seconds. A node sends the request for timed lease votes to other participants to become a leader. When that node has a quorum of lease votes from other participants, it means it has the lease to become a leader.

For leader election, Paxos uses the Bully algorithm. If the proposer is not proposing to be a leader, any node can aim to be a leader. For a particular node to be accepted by the cluster, the node sends its server IDs to all peers within the cluster. The peers respond with their server ID. If all responses are smaller than the node's server ID, that node becomes the leader.

[Leader leases](./leader_leases1)

If a peer receives the ID of a node that wants to be a leader, and its ID is greater than the received ID, then instead of returning its ID, it starts an election to become the leader. The bully algorithm avoids livelock issues.

[Leader leases](./leader_leases2)

After a successful write, a replica's lease vote is automatically extended, and the leader will ask for a lease vote extension if it is about to expire. The lease interval of the leader starts when it receives the necessary number of votes to constitute a quorum, and the lease interval will end when the leader no longer has a quorum of votes. Within a Paxos group, a Paxos leader can change over time for different reasons. Spanner enforces a disjointness invariant, which implies that lease intervals of leaders are disjointed across all leaders of the shard over time.

Spanner implementation allows an abdication of a Paxos leader by freeing the lease votes of the participants. Spanner limits the conditions under which abdication is allowed to ensure that the disjointness invariant is maintained. The conditions are as follows:

- Consider that the Paxos leader defines the maximum timestamp as Smax.

- Before abdication, the Paxos leader should wait for TT.after(s_max).

The abdication of the Paxos leader is explained in the following slides:

[Abdication](./leader_leases3)

To work, for each Paxos group, we require a disjointness invariant to be true, which means that the lease intervals of each Paxos group leader should be disjointed from other Paxos group leaders.


## Read and write transactions
The read and write transactions use two-phase locking. The timestamp can be assigned to a transaction after acquiring all locks and before releasing them. Spanner assigns a timestamp for any given transaction. Spanner enforces the following invariants:

1. The disjointness invariant

2. The external consistency invariant

Spanner assigns timestamps to Paxos writes in increasing order in each Paxos group. The assignment is in monotonically ordered across leaders too. The assignment of timestamps in monotonically ascending order is a simple task for a single leader replica. Using the disjointness invariant, all leaders must comply with this rule: a leader can assign timestamps only during their leader lease. It means when a timestamp s is assigned to Paxos leader, s_max is incremented to maintain disjointness.

In the external consistency invariant, if the T2 transaction is started after T1's commit, we can infer from it that T2's commit timestamp should be greater than T1's commit timestamp. Consider the following for the transaction Ti:

- Start of Ti: e_i_start
- Commit of Ti: e_i_commit
- Commit timestamp of Ti: s_i

The t_abs is the absolute of time, then the invariant becomes: if t_abs (e_1_commit )<t_abs(e_2_start) then s1 < s2.
```
Absolute of time is the actual wall clock time, a hypothetically calculated ideal time with precision and no drift. Spanner ensures that its TT.now()'s absolute time is always within the closed interval [earliest, latest] that Spanner tells with each invocation to the TrueTime API.
```

[Transactions](./read_write)

The following protocols are followed to assign timestamps to transactions. These protocols ensure the invariant is followed. The commit request's arrival event at the coordinator leader is e_i_server for the write transaction Ti.

1. Start: The coordinator leader assigns a commit timestamp s_i to T_i. The timestamp assigned is not less than the TT.now().latest, computed after e_i_server (remember that TT.now() provides an interval [earliest,latest]).

2. Commit wait: The coordinator leader waits until TT.after(s_i) or TT.now().earliest>s_i is true. After the wait, the client can see the updated data. The commit wait also makes sure that for Ti, s_i < t_abs (e_i_commit).

[Transactions](./eps)

This is the monotonicity invariant that allows Spanner to reliably decide if a replica's state is up-to-date enough to fulfill a read. While traditional databases employ strict two-phase locking and single-version storage to guarantee external consistency. When traditional databases perform a "strong read," requesting the latest data, Spanner gains a read lock on the data, preventing any updates on the data that is part of the read transaction.


## Serving reads at a timestamp
Each replica in Spanner keeps track of the maximum timestamp, t_safe, at which it was up-to-date. A read transaction at timestamp t is considered successful if t <= t_safe at any replica. Spanner defines t_safe as follows:

```
t_safe =min(t_safe_Paxos,t_safe_TM)
```
where, 
- t_safe_Paxos is the safe time of each Paxos state machine or the latest Paxos write's timestamp. With respect to Paxos, incoming writes will no longer be committed at or below t_safe_Paxos since timestamps are incremented monotonically, and write transactions are also committed in order.

- t_safe_TM is the safe time of each transaction manager. For zero-prepared transactions, the value of t_safe_TM is infinity. For these transactions, the state to be affected is indeterminate, and even the participant replica isn’t sure if these transactions will commit or not.
 
```
Zero-prepared transactions: Such transactions that are not committed or are between the two phases (pre commit and post decision) of two-phase commit.
```

```
Note: For a participant, t_safe_TM refers to the replica leader's transaction manager. The state of the transaction manager can be inferred by the participant via metadata transmitted with Paxos writes.
```

The commit protocol guarantees that all nodes know the lower bound of a prepared transaction's timestamp. As a result of the commit protocol, all participants involved in a transaction will be aware of the lower bound of the timestamp of prepared transactions.

As per the Spanner paper, for a group g and a transaction Ti, every participant leader assigns a prepare timestamp, s_i,g_prepare to its prepare record. The coordinator leader ensures that the commit timestamp of the transaction is greater than the prepare record for all participant groups g. This is for every replica in a group g, over all transactions Ti, prepared at g, t_safe_TM =min_i(s_i,g_prepare )−1 over all transactions prepared at g.

```
Question
We calculated t_safe_TM for the transaction manager as t_safe_TM =min_i(s_i,g_prepare)−1. Why is 1 subtracted from the minimum prepared timestamp?

Answer
An explanation can be that we treat a read transaction as a snapshot read—a read from the past. So, −1 helps us in achieving that because if we do not subtract 1, we might use a time where a transaction is still in progress (more specifically in a prepare phase).
```


## Assigning timestamps to read-only transactions
A snapshot read is a read from the past. The client gives a timestamp, and the state of the object or row on that timestamp is returned. Read-only transactions are non-snapshot standalone read. They are executed in the same way as a snapshot read but since timestamp is not given, we calculate s_read and fetch the data on basis of s_read.

There are two stages of execution for a read-only transaction as per Spanner paper:

```
Corbett, James C., Jeffrey Dean, Michael Epstein, Andrew Fikes, Christopher Frost, Jeffrey John Furman, Sanjay Ghemawat et al. "Spanner: Google’s globally distributed database." ACM Transactions on Computer Systems (TOCS) 31, no. 3 (2013): 1-22.
```

- Assign a timestamp s_read
- Execute the transaction's reads as snapshot reads at s_read

We can perform snapshot reads on those replicas that are up-to-date. TT.now() returns TTinterval that contains the earliest and latest timestamp, which we will assign s_read=TT.now().latest at any time after the start of a transaction. However, this timestamp may block data reads at s_read if t_safe hasn't advanced enough. To ensure disjointness, it is important to remember that picking a value for s_read can also increase s_max. Because of this, Spanner should choose the oldest timestamp that maintains external consistency to minimize the possibility of blocking. In the next lesson, we will learn how to choose such a timestamp.

In this lesson, we learned how Spanner does concurrency control over read and write operations. Moreover, we learned how the read operations execute without locking at a timestamp chosen by the system and can only be executed on up-to-date replicas.

