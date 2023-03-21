# Database Operations in Spanner
In this lesson, we will learn about read-write, read-only, and schema-change transactions utilizing the timestamping mechanism.

## Read-write transactions
A transaction's writes are buffered on the client side until the commit. Therefore, the results of a transaction's writes are not visible to subsequent reads inside the same transaction. This architecture is particularly well-suited to Spanner since uncommitted writes do not have timestamps assigned yet, and the timestamps of any data read are returned by a read transaction.

The following slides explain the read and write transactions.

[Transactions](./read_write)

Spanner uses the wound-wait approach to prevent deadlocks during reads within read-write transactions. Whenever a client requests up-to-date information, it sends the request to the group’s designated leader replica, acquiring the necessary read locks and retrieving the data. To avoid having its transaction timed out by the participant leaders, a client periodically sends keepalive messages while a transaction is still open. Then, the client finishes all reads and writes data to its write buffer.

[Transactions](./read_write2)

### Two-phase commit in Spanner
Spanner uses a two-phase commit (2PC) to guarantee isolation and strong consistency. The 2PC begins once a client has finished all the reads and has written data to its write buffer.

If participants in a 2PC are physically nearby, the latency for data propagation will be lower. Spanner ensures serializability by running 2PC and two-phase locking on the Paxos leaders. The client selects a 2PC coordinator that communicates with the other non-coordinator leaders of the Paxos group. The 2PC coordinator is the leader of that group too. The rest of the Paxos leaders are participants, and the client notifies the group leaders of the coordinator's identity. It also tells the participants the number of buffered writes via a commit message.

If the coordinator crashes, 2PC fails. To cater to it and ensure fault tolerance of the system, all states of the 2PC for both the coordinator and participant are stored in the Paxos state machine. If one of them were to go down in the middle of a 2PC round, the new leader would have all the necessary information to complete the commit.

[The leader in the partition 2 is the 2PC coordinator that communicates with the other non-coordinator leaders of the Paxos group](./2pc.png)

#### Non-coordinator role
A leader who isn't the coordinator gets access to write locks. To guarantee monotonicity, it chooses a prepare timestamp after any timestamps assigned to prior transactions, and the prepared record is logged via Paxos. After that, all participants communicate their prep time to the leader.

#### Coordinator role
The coordinator leader bypasses the prepare step and gets locks for the write. After receiving input from all the group's leaders, it selects a single timestamp for the entire transaction. Let's denote the commit transaction as s and it should be as follows:

1. Greater than or equal to all prepare timestamps to satisfy the invariants of read-write transactions

2. Greater than TT.now().latest (latest value is fetched when the client sends a commit message to the coordinator)

3. Greater than the timestamps of all the transactions that the leader coordinator has assigned previously

All of the above help maintain invariants like monotonicity and constraints of read-write transactions.

Another constraint is commit wait. Therefore, the leader coordinator will wait till TT.after(s) before sending the commit message to other replicas. Based on commit wait and the latest timestamp fetched, the expected wait is 2∗ ϵ_average where ϵ_average is the average uncertainty bound on the time clock. The coordinator then communicates the commit time to the client and all other leader participants after the commit wait period has elapsed. The transaction's outcome is recorded in Paxos by each participant leader. Before releasing locks, each participant applies the same timestamp.

The following slides explain how Spanner applies the two-phase commit:

[Two-phase commit](./coord)


## Read-only transactions
Whenever a timestamp is being assigned, all participating Paxos groups must be involved for consensus. Therefore, for every read-only transaction, Spanner needs a scope expression. The scope includes all the keys that the entire transaction will read.

### Read within one Paxos group
If only one Paxos group can serve the values in the scope, the client sends the read-only transaction to that group's leader. A timestamp for a read-only transaction is selected only at the Paxos leader. The leader executes the read after assigning s_read. For a single region, Spanner performs better in latency than TT.now().latest.

As mentioned in the Spanner paper, consider LastTS() as the timestamp of the last committed write in the Paxos group. For no prepared transactions, the assignment of s_read =LastTS() will satisfy the property of external consistency since the transaction will see the outcome of the latest write and will be ordered accordingly. If any prepared transaction is there, then s_read =TT.now().latest will suffice.
```
Corbett, James C., Jeffrey Dean, Michael Epstein, Andrew Fikes, Christopher Frost, Jeffrey John Furman, Sanjay Ghemawat et al. "Spanner: Google’s globally distributed database." ACM Transactions on Computer Systems (TOCS) 31, no. 3 (2013): 1-22.
```

```
Note: Using s_read =LastTS() is a special case that we use when we only have one Paxos group involved in the transaction and there are no prepared statements. Doing so usually gives us a better timestamp as compared to the more conservative s_read =TT.now().latest that might also need to wait for a safe time and to comply with the disjointness condition.
``` 


### Read within multiple Paxos groups

If multiple Paxos groups are serving the scope's values, then we have the following options:

- A complicated option is to determine the s_read value based on LastTS() by doing consensus with leaders of all participants Paxos groups.

- A simpler option is to avoid the consensus round. The client can have its read executed at s_read =TT.now().latest. Here, the client has to wait for a safe time to proceed.

We can route all read transactions to up-to-date replicas.
## Schema-change transactions
TrueTime allows Spanner to accommodate incremental updates to the schema. Using a conventional transaction is infeasible since there can be millions of participants in the groups in a database. For example, Bigtable allows for atomic schema updates inside a single data center. However, these updates halt all other processing.

Changing the schema in a Spanner database does not block any other transaction. The process of the schema-change transaction is as follows:

1. A future timestamp is determined in the prepare phase. This allows schema updates to be implemented across thousands of servers with minimum impact on ongoing operations.

2. Consider that t is the timestamp of a registered schema-change transaction. All read and write transactions that are dependent on the schema, will sync with t. If the transction timestamps of these are before t, they can continue. However, timestamps that are later than t will be blocked until the schema change transaction completes.

It would be pointless to define schema-change to occur at t without TrueTime, since we cannot determine if it's safe to execute the schema-change request.

So far, we have covered how the read and write transactions are executed in Spanner. The next lesson will cover how Spanner satisfies our non-functional requirements.


