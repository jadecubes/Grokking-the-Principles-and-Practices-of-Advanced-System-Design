# Detailed Design of Chubby: Part III
This lesson describes some more aspects of Chubby's design. The table below summarizes the goals of this lesson.
```
     Lesson Summary
Section        Purpose

Caching        This helps Chubby to deal with excessive read traffic.

Session       This explains the relationship between the client and the master and how a client can stay connected to the master.
```
## Caching
What if we have too many read requests? Chubby reduces the read traffic by letting Chubby clients cache locked file data and metadata of client-requested nodes in a consistent and write-through cache in memory.
```
The data is updated both in the client's cache and main memory of master.
```
[Caching]

### Consistency
The master keeps a list of data that clients are caching and sends invalidations to the client to keep the cache consistent. The protocol ensures that a client can only get a consistent Chubby state or an error.

### Maintenance
What if the data or metadata is to be changed, and a client is reading from the cache? The following process occurs.

- The master blocks the modifications and sends invalidations to clients with the relevant data cached.

- When a client receives an invalidation, it flushes the invalidated state and acknowledges it by making a KeepAlive call (KeepAlives are discussed in detail in the next section).

- The master is notified that the client has flushed its cache from the client.

- After that, the modification is started on the data or metadata.

While the invalidation is in progress, two approaches can be opted for read operations that are happening afterwards.

Approach 1: If cache invalidations remain unacknowledged by the client, the master keeps the node uncacheable, so a round of invalidations is necessary. This approach lets the reads to have uncached access without delay. It is favorable because read operations exceed the write operations considerably.

Approach 2: It involves blocking the read calls while the invalidation is in progress. This would prevent the clients from bombarding the master with uncached access. However, it will cause some delay.

If both approaches are bad options individually, one can opt for a hybrid solution by switching approaches if an overload of requests is detected.


### Cache protocol
The protocol of caching is simple: if a change happens in cached data, the client invalidates it (clients can never change it). Using an update-only protocol can also be simple, but it can be inefficient. A client that accesses a file might get an unrestricted number of unnecessary updates.

Caching protocol also allows clients to cache locks (reminder that clients can cache locks, handles, meta-data and data). If a lock is cached, it can be held longer than necessary so that the client can use it again. These locks can be released when an event notifies the lock holder that another client wants to hold this lock.

In addition to caching data, metadata, and locks, clients can also cache open handles. A handle is created by a client's Open() call to the master and then cached on the client. Hence, when a client tries to open a previously opened handle, it does not lead to an RPC call to the master; only the first Open() call does. Caching handles is slightly restricted just to ensure that it does not affect the semantics observed by the client.

- If an application has closed a handle on ephemeral files, it cannot be held open.

- Handles that allow locking can be reused. However, they cannot be used by multiple application handles simultaneously. The restriction is in place to prevent the side effects of poison() or Close() called by another client because these calls cancel remaining Acquire() calls to the master.
```
Question
Do all the clients stay connected to a cell all the time?

Answer
No, Chubby provides a session and a lease timeout, both of which define the connection between a client and a cell and the time period for which they would be connected.
```

## Sessions
What is the relationship between a client and a cell, and how long can they stay connected to each other? These are the questions that are answered by a Chubby session. A relationship between a client and a cell is called a session.

This relationship exists for a certain amount of time, which is called a lease.

It is maintained by a series of periodical handshakes that are called KeepAlives.

If a session remains valid and a client does not inform the master that its handle, locks, and cached data are invalid, they remain valid. However, maintenance protocol may require the client to acknowledge a cache validation to maintain a session. A client that is in session with the master of the cell requests a new session. However, it ends only when it terminates, or the session has no open handles, and no calls occur for over a minute.


### Lease
A session comes with a lease. It is defined as a time period during which the master commits with the client that it will not terminate the connection unilaterally. The master can increase this time period but cannot decrease it. The expiration of this time period is called session lease timeout.

Master increases the time in a session under the following situations:

- When a session is created

- When a master failover occurs

- When it responds to a KeepAlive RPC


### KeepAlives
KeepAlives are a series of periodic handshakes between the client and the master to literally keep the session alive. When a KeepAlive is sent to the master, it blocks the RPC and only allows it to return when the session’s time is close to being over. The returning RPC notifies the client of a new lease time-out. After receiving this new time-out, the client initiates another KeepAlive just to ensure that the master almost always has a KeepAlive call blocked from the client and the session has no chance of expiring. The default extension value of a lease time-out is 12s. However, if a master is to be overloaded with constant KeepAlives, it can increase this value so that it has to process fewer KeepAlives.

[Session of a client and master]

Along with the client’s extended lease, two other components, events and cache invalidations, are also returned with the KeepAlive reply.

As we already know, the master returns the KeepAlive reply when the lease is close to expiring. However, if a master has any cache invalidation or events to report, it returns the reply earlier. This extra time that the client receives before sending another KeepAlive call and master sending information in KeepAlive replies has some benefits.

- The client cannot maintain a session without acknowledging any cache invalidations.

- Chubby RPCs flow only from the client to the master, which results in the simplification of the client.

- Permits Chubby protocol to operate via firewalls that allow one-way connections.

Let's say that a lease has timed out, the master hasn't sent back any replies, or it may have failed. How will the client know about this?

### Local lease
The client also keeps a local lease time-out, which approximates the master’s lease time-out so that if the master does not send back any replies, the client knows something is wrong. This local lease is a strict approximation of the master’s lease time-out. The strict approximation is dependent on two things:

- The time that KeepAlive calls spend away from clients

- The rate at which the master's clock is ticking

The master’s clock should only be a known constant factor faster than the client’s local clock.

If a client’s local lease times out, it becomes unsure. It could be that the master has terminated the session, or it may have failed. Such situations are known as jeopardy. When in jeopardy, the client takes the following steps:

- Empties and disables the cache

- Waits for a period called a grace period, which is 45 seconds by default, and keeps trying KeepAlives

- If the client successfully exchanges a KeepAlive reply within the grace period, it enables its cache

- If the client does not receive it within the grace period, it assumes that the session has been terminated

This allows the ChubbyAPI calls not to be stuck indefinitely if the cell is inaccessible and returns an error upon the end of the grace period.

There are three types of events during which the Chubby library notifies the client about the situation of the session. The brief details of those events are as follows:

- Jeopardy event: This signifies that the grace period has begun.

- Safe event: This signifies that the session has survived the communication.

- Expired event: This signifies that the session has timed out.

In this lesson, we learned how Chubby deals with a lot of read traffic by caching and how clients remain connected with a master. In the next lesson, we will explore how the session's components contribute to Chubby's failovers.
