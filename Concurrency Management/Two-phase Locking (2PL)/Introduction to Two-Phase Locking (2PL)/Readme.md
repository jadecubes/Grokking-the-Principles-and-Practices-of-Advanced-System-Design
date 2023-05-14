# Introduction to Two-Phase Locking (2PL)

Concurrency is everywhere in reading and writing to a data store. It poses challenges to prove that certain concurrent code is safe to execute in parallel with others, potentially interleaving arbitrarily. The problem of concurrent data access was thoroughly studied in the context of databases and transactions in 1974. The transaction is an abstraction to safely deal with concurrency (and many other data issues). We will discuss concurrency control in the context of transactions, though the concepts are more widely applicable.

This chapter will elaborate on potential problems of data transactions, and a specific strategy databases employ to prevent them, i.e., Two-Phase Locking (2PL). We will focus mainly on:

- Concurrency management
- Addressing numerous race conditions due to concurrency insurance
- Implementations of databases’ isolation levels like serializability using 2PL

## Race conditions
Multiple clients typically access databases at once. That is not a problem if they read and write to different database entries. We might still encounter concurrency issues if they access the exact database records. These are called race conditions.

**Example**

Imagine that two clients are incrementing a database-stored counter at the same time. Each client must read the current value, add 1, and then write back the new value, assuming the database doesn’t have a built-in increment operation. Due to the race condition, the counter only gets to 13 when it should have gone from 12 to 14 due to two increments.
## Concurrency problems
The harsh reality of data systems presents several potential concurrency problems:

- Multiple clients may simultaneously write to the database, overwriting one another’s modifications.
- Data that has only been partially updated may be seen by a client and can be confusing.
- Race conditions among clients can result in unexpected bugs.
A system must handle these errors and make sure they don’t result in the catastrophic breakdown of the entire system to be dependable. However, it takes a lot of labor to implement fault-tolerant methods. To make sure the solution truly works, it takes a lot of meticulous consideration of all the potential problems.

Before understanding the 2PL mechanism, let’s first understand what transactions are and why they’re so integral in solving concurrency issues.
## Transactions
Transactions have been the preferred method for resolving these problems for many years. A transaction is a means for an application to logically unite multiple reads and writes. In theory, a transaction’s reads and writes are combined into one operation, either succeeding (commits) or failing (abort, rollback). The transaction is safe to retry if it fails. An application can handle errors considerably more quickly with transactions since partial failure—when some operations succeed, and others fail—is not a concern.
```
Note: The application can overlook potential error scenarios and concurrency problems by utilizing transactions because the database handles them instead. We call these safety guarantees.
```
Given the case of race conditions and concurrency issues in transactions, the practical approach is achieving isolation using serializability (serializable isolation). Let’s introduce this concept quickly before utilizing it in our further discussion.

## Serializable isolation
We use isolation, to separate concurrently running transactions from one another, making it impossible for them to be interdependent. The ability of each transaction to act as though it is the only one currently active throughout the whole database is known as serializability. This process is called serializable isolation.
```
isolation: Transactions that are active simultaneously shouldn’t conflict with one another. For instance, if one transaction does several writes, another transaction should only see all or none of them, but not a partial set.
```
Although the transactions may have been running concurrently, the database ensures that their commits produce an outcome similar to as if they were running serially, i.e., (one after the other.

There are three ways to achieve serializability in databases:

1. Actual Serial Execution: This is the actual serial execution of transactions.
2. Two-Phase Locking (2PL): We will discuss 2PL in this chapter.
3. Serializable Snapshot Isolation (SSI): This is an optimistic concurrency control technique and beyond the scope of this course.

## Two-Phase Locking (2PL)
Two-Phase Locking (2PL) makes decisions about a transaction’s ability to access a database item in real time based on a pessimistic concurrency control protocol, i.e., if anything may possibly go wrong (as specified by another transaction already holding a lock), it’s advisable to wait before acting until everything is secure again.
```
Note: For its implementation, the protocol is not required to be aware, in advance, of every query a transaction will run.
```

### Rules
Following are the three rules that govern the basic functioning of 2PL.

- Rule 1: The lock manager checks for conflicts in locks acquiring, against the incoming operation request and the already set locks by the running operations. It proceeds or waits with the incoming requests based on its conflict findings. The waiting period continues till the conflict resolves. The previous operations release their locks. The lock manager also sends the approved operations to the database manager for its executions and saves the record of the acquired locks for further conflict lookups till the operation completes.

This rule avoids conflicts between two transactions acquiring locks on the same data. It ensures that the conflicting transactions get scheduled in the same way as their locks-acquiring schedule.


- Rule 2: The lock manager can’t release any lock for any operation until the database manager has marked that operation completed (abort or committed) and notified the lock manager.

This rule ensures that the database manager performs operations on a data item in the same order that the scheduler sends them. Consider the case where T1 acquires a readlock[a] and releases it before the database manager verifies that read[a] has been processed. Then, T2 may acquire a conflicting lock on a, writelock[a], and the lock manager sends write[a] operations to the database manager. Even though the lock manager sends the read[a] before write[a], there is no guarantee that the database manager will receive and process these operations in the same order in the absence of this rule.

- Rule 3: The lock manager will not allow any transaction to acquire any more locks for any data item once it has released even a single lock. This rule is called the two-phase rule, hence the name “Two-Phase Locking”.
This refers to the following two phases of its operation.

Phase 1: Growing/Expansion: Here, the transaction acquires all the locks and releases no locks.

    - Each transaction asks the DBMS’s lock manager for the locks it requires, and the lock manager either allows or denies requests for locks.
Phase 2: Shrinking/Contraction: Here, the transaction releases all the locks and acquires none.

    - This phase is the transaction’s first step after releasing its first lock.
    - Only locks that the transaction previously acquired may be released. During this phase, it cannot acquire new locks.

[Rules](./violation)

```
Question 1
What goes wrong if a lock manager does not follow 2PL rules?

Answer
The 2PL rules, especially the third rule is interesting, since it is not obvious how rule three helps. Any violation of 2PL rules might mean that we can not ensure a serializable schedule of commands and cannot get isolation between transactions.

We will use an example to answer this question. Let’s assume that a lock manager is handling the following two transactions:

T1: read[a]→write[b]→commit

T2: write[a]→write[b]→commit

Let’s assume that the execution of the two transactions mentioned above doesn’t follow 2PL rules, and we execute this schedule (the subscripts of 1 indicate that the specific command is from T1 and so on): 

readlock1[a], read1[a], readunlock1[a], writelock2[a], write2[a], writelock2[b], write2[b], writeunlock2[a], writeunlock2[b], commit2, writelock1[b], write1[b], writeunlock1[b], commit1.

These are some violations of 2PL in the schedule above, mainly the two-phase rule. T1 released a lock on object a and subsequently acquired a lock on object b, violating the no-lock acquiring rule once a lock gets released. This scenario created a whole other transaction, T2, executed between these two operations of T1. This in-between execution may cause dirty writes because T2 is following T1 in terms of object a but leading it concerning object b.

If the overall schedule of the two transactions’ execution had followed 2PL rules, it would have been like this:

1. There are no locks by any transaction.
2. T1 acquires the readlock1[a].
3. T2 requests writelock2[a], but the lock manager places it in a queue as the database manager hasn’t yet confirmed T1’s completion.
4. T1 requests and acquires the writelock1[b].
5. T1 sends the commit1 operation to the lock manager, which subsequently forwards it to the database manager for processing. The lock manager releases all locks held by T1, marking this transaction’s completion. It complies with all the rules of 2PL.
6. The lock manager initiates the T2’s writelock2[a] and sends the operation command to the database manager for processing.
7. T2 requests the writelock2[b] and acquires it. The lock manager forwards the operation to the database manager.
8. Once the lock manager has received the commit2 and the database manager processes it, the lock manager releases all the locks held by T2.
The schedule above can be shown like this: readlock1[a], read 1[a], writelock1[b], write1[b], commit1, readunlock1[a], writeunlock1[b], writelock2[a], write2[a], writelock2[b], write2[b], commit2, writeunlock2[a], writeunlock2[b].
 ```
 
 ```
Question 2
How will it impact if one client does not follow 2PL’s third rule while the others do?

Answer
Since the lock manager has all the records for current locks, it ensures that no client acquires/releases a lock in violation of the basic rules of 2PL. It is solely the lock manager’s responsibility to ensure that the clients follow these rules.

Hypothetically speaking, if the lock manager misses a client avoiding the 2PL rules, it can cause correctness issues for the client who is not following rules, while others might be safe. However, it will indirectly impact the remaining clients if one client makes inconsistent changes to some data.
```
### Requirements
Before listing down the requirements for 2PL, let’s first understand that in a database, we can block the reading and writing of transactions by introducing two modes of locks— shared or exclusive.

- Shared lock: This lock is termed as “shared” because it can be shared among multiple transactions reading data from an object in the database. It is also known as a read-only lock, since transactions can only read the object by acquiring this lock and can’t update it.

[Shared lock](./shared_lock)

- Exclusive lock: An exclusive lock allows a single transaction to read and write from an object in the database. No two transactions are allowed to interact with the same object simultaneously.

[Exclusive lock allowing just one transaction to access object x](./exclusive.png)

Using the aforementioned lock modes, the 2PL must have the following requirements:

1. Shared lock for reading: A transaction must acquire the lock in the shared mode to read an object from the database. Since it’s a shared lock, many other transactions might be reading the same object simultaneously, which is acceptable.

The only limitation is if there is already an exclusive lock placed on the object, the transactions trying to access that object must wait for the completion (commit or abort) of the previous transaction.

[Shared lock for reading](./shared_lock_reading)

2. Exclusive lock for writing: A transaction must acquire the lock in the exclusive mode to write to an object in the database. If the object already has a lock placed on it, either shared or exclusive, the transaction must wait until the previous transaction finishes, since this phase doesn’t allow more than one transaction to hold the lock for the same object.

[Exclusive lock for writing](./exclusive_lock_for_writing)

3. Lock upgrade: A transaction needs to upgrade its lock from shared to exclusive if it reads and then writes an object. This shift is similar to acquiring an exclusive lock altogether. The transaction has to wait for another transaction that has already placed the lock on the object.

```
Question
In which phase would the lock upgrade happen?

Answer
Since we set the condition that the lock upgrade is similar to acquiring a new exclusive lock, it must happen in the expansion phase of the two phases described earlier.
```

### Architecture
To implement 2PL, we essentially need two main components:

1. A database that supports transactions: Currently, most of the relational and a few non-relational databases support transactions, following the style of the first relational database—IBM’s System R. Others like MySQL, PostgreSQL, Oracle, SQL Server, etc., have close similarities with System R.

2. A lock manager: This is the central managing component for all the locking and unlocking approvals. Transactions will request the lock manager to acquire locks and inform it while releasing them, so the lock manager must have a global state of each active lock in the database. Once a transaction requests a lock, the lock manager must be able to look into its table of existing locks for a match. If found, it must also queue the new requests to entertain them afterward, in the designated way.

The lock manager implementation varies from database to database and from application to application. We can either have a central lock manager or a distributed lock manager, each with its pros and cons.
```
Most of our chapters have a bird’s eye view section to summarize the chapter and also as a milestone in our learning journey. Due to the shorter size of this chapter, we haven’t included that bird’s eye view section here.
```
In the next lesson, we’ll discuss the scenarios of deadlocks in acquiring locks and analyze the limitations of 2PL.

