# The Rationale Behind Chubby’s Design
Chubby is a locking service that provides coordination between a large number of clients trying to do some work, for example, accessing a database. Let's explore some arguments as to why a lock service is added to the system.


## Why use a lock service?
It could be argued that a library built around Paxos should have been used which only depends on a name service, instead of a library that accesses a central lock service. The reason is that a Paxos library would provide a standard framework to the programmers if their services can be implemented as state machines. Chubby does provide such a client library which is independent of it. Having said that, adding a lock service to a system does have advantages over using only a client library (that would need all the code for consensus and locking amongst independent parties).

### Availability
Developers initially tend not to go for a highly available system. They start with prototypes that run with little load and low availability. While coding, they usually don’t put much thought into structuring their code so that it can be used with a consensus protocol.
However, when the service gains clients, it needs availability guarantees to improve. The following aspects are added to the existing design to increase availability:

- Replication

- Primary elections

Availability can be provided by a library that provides distributed consensus. However, an advantage of a lock server is that it makes management of the current program structure and communication patterns a lot simpler.

#### Example
If a system has to elect a master which starts writing to an existing file server, we only need to add two new statements and an RPC parameter to the program.

- Statement 1: Acquire a lock to become the master.

- RPC parameter: Pass a lock acquisition count as an integer with the write RPC.

- Statement 2: Reject the write requests with an acquisition count less than the currently acknowledged value with an if-statement to the file server. It helps guard against delayed packets.

This technique is much simpler than operating existing servers in a consensus protocol, particularly if we have to maintain compatibility during a transition period (when an application is being interfaced with the Chubby system).

### The mechanism for advertising results
Most of the services that either perform elections for a primary server or divide the data between components of the service need a process through which they can advertise the results. For that, clients should be allowed to perform operations on small files. This process can be accomplished through the name service.

[Name service]

However, the lock service itself is well-suited for using namespace because of two reasons:

It reduces the number of servers on which a client depends.

It provides a protocol with shared consistency features that everyone can use (clients can either see a consistent view of the Chubby state or an error).

Chubby has been successful as a name server mostly because it uses consistent client caching instead of time-based caching. Developers do not like to use a cache time-out (i.e., DNS time-to-live value) since it can lead to a high DNS load or long client failover times.

### Reduction in the number of servers
The distributed-consensus algorithms need a certain number of replicas to be available to make decisions. Therefore, they need those replicas to be available to achieve high availability.

#### Example
Chubby typically has five replicas, and at least three must be available for the cell to be in working condition. However, if a client system deploys a lock service, it allows even a single client to acquire a lock and continue. In other words, a client can depend on a reliable locking service making client code much simpler.

So, one advantage of a lock service is that it reduces the number of servers needed for reliable client systems.

A consensus service can also allow clients to progress even with only one active client process by using multiple servers as acceptors in the Paxos protocol. However, it is not specifically used for locking because it would provide no solution for the problems described above. While a locking service will need a consensus to reliably replicate the state, it needs much more than just consensus.

## Design decisions
The arguments above lead to two key design decisions:

- Use a lock service instead of a library or a consensus service.

- Serve small files that allow advertisement of elected primaries and their parameters instead of building and maintaining another service.

The expected use and environment lead to some more decisions.

- Permit a huge number of clients to access a Chubby file without using many servers because a service's primary being advertised through a file may have a huge number of clients.

- Use an event notification mechanism to avoid polling because clients and replicas of a replicated service may want to know when the service’s primary changes.

- Cache the data because one of the downsides of supporting many developers is that even if they do not need to poll files periodically, they will. Caching helps increase the scalability of the system in terms of the number of concurrent requests that can be serviced.

- Use consistent caching because developers get confused by non-intuitive caching semantics.

- Provide security mechanisms, including access control, to avoid any financial losses.


## What kind of locking does Chubby need?
Chubby needs coarse-grained locking (which can hold for long durations) instead of fine-grained locking. The reason is that the applications will use a lock to elect a primary that would manage access to that data long term. It could be hours or days.

These two kinds of locking have different sets of features.

### Coarse-grained locking
Coarse-grained locks will need much less load on the lock server. They are rarely acquired (the lock-acquisition rate is weakly related to the client applications’ transaction rate), so temporary lock server unavailability rarely delays any clients.

Transferring a lock from one client to another needs expensive recovery procedures. Therefore, the loss of lock by a lock server during a lock failover is not desirable. Hence, surviving lock server failures is important, even with the overhead of doing so.

Such reliable locks allow many clients to be efficiently served by a few lock servers with lower availability because clients rarely need to acquire them.

### Fine-grained locking
Fine-grained locks are frequently accessed. Even brief unavailability of the lock server can delay many clients. The performance of a system and the ability to add new servers into the system is challenging because the transaction rate of the lock service is directly proportional to the combined transaction rate of clients.

Fine-grained locking can be advantageous due to the following reasons:

- Overhead of locking can be reduced by not maintaining locks across lock server failures.

- The locks are held only for short periods, so the time penalty for dropping locks is not severe. (Clients should be prepared to lose locks during network partitioning. So that introduction of new recovery paths can be prevented in case of loss of lock during a lock failover.)

### Locking in Chubby
Chubby provides only coarse-grained locking. However, clients can implement their fine-grained locks according to the needs of their application. Chubby's coarse-grained locking can allocate locks to application-specific lock servers by dividing the locks into different groups.

#### Benefits
The benefits of Chubby's locking are as follows:

- A little state is needed to maintain the fine-grained locks.

- The servers only need to keep a non-volatile, monotonically increasing acquisition counter that is seldom updated.

- Clients can learn about the lost locks at unlock time.

- The protocol can be simple and efficient if a lease of fixed length is used.

- The client developers are responsible for providing a specific number of servers depending on their load, which frees them of implementing a complex consensus service.

In this lesson, we learned why we added a locking mechanism in Chubby and the rationale for the type of locking Chubby uses.
