# Introduction to Chubby
## Problem statement
Achieving consensus among several participants has been a challenge in distributed systems. Many attempts have been made to achieve a reliable solution to this consensus problem. Chubby, mainly a locking service, attempts to solve this consensus problem by extending an already existing consensus algorithm, Paxos, which is discussed later in this lesson. We’ll take the example of the election of a primary among peers (to be discussed later) as the core problem to which it provides a solution. We’ll also discuss how this locking service helps in the leader election in later chapters.
```
Note: We will discuss how this locking service helps in the leader election in later chapters.
```
It is a distributed consensus problem, and we need a solution based on asynchronous communication that describes the behavior of most real networks, like the Internet.

Such asynchronous communication allows packets to be lost, delayed, and reordered. Hence, our system needs a solution to achieve synchronization, coarse-grained (for longer durations) rather than fine-grained (for shorter and focused durations). It will help us solve the problem of selecting a leader from a set of otherwise equivalent servers precisely, which is the main problem statement here.


## Paxos—an existing solution
The Paxos protocol handles asynchronous consensus in a distributed system. Paxos is at the heart of most working protocols for asynchronous consensus that were present at the time of Chubby’s introduction. To use Paxos to achieve this functionality, we can simply build a Paxos library and ask all the applications to use that library for consensus. However, there are some limitations to implementing the Paxos library for consensus. These are listed below.
```
Paxos is a family of protocols for solving consensus in a network of unreliable or fallible processors. (Source: Wikipedia)
```
- We must write our applications as state machines.
- There are potential performance problems in case of state failure, as one state failure leads to the overall application failure for a particular consensus. This is because of the Fishcer-Lynch-Patterson (FLP) impossibility result.
```
FLP: In an asynchronous message-passing distributed system, an agreement is impossible if even one crash failure is permitted.
```
- There are potential correctness issues, specifically liveness problems, and the only possible solution is to fix them manually.
```
correctness: We can define the correctness of a system in terms of its compliance to safety and liveness properties.
liveliness: A liveness property defines something that must eventually happen in a correct system.
```

[Problems with the Paxos library implementation]

Since Paxos only maintains safety without timing assumptions (asynchronous consensus), we can’t deploy it because of the visible limitations related to asynchronization discussed above.
```
A safety property defines something that must never happen in a correct system.
```

One of the ways to overcome this issue is to introduce clocks (time stamps) and synchronization to ensure liveness. This is where Chubby is useful—it uses Paxos with lease timers to achieve synchronization.

## Chubby–a proposed solution
Google has a lot of distributed systems, and they all need to perform a primary selection (consensus) for their operations. Before the deployment of Chubby, most distributed systems at Google relied on the following two primary methods:

1. Ad hoc methods: Using ad hoc methods enables electing or changing a leader by allowing the distributed system to duplicate its work or progress without serious harm.
2. Human intervention: This was the last option when the distributed system required manual correctness and operator input.
Chubby handles two essential requirements of a distributed system more sophisticatedly. These two requirements map directly from the points mentioned above:

1. Primary election: Chubby reduces the computing power required to perform an ad hoc operation.
2. Availability: Chubby significantly improves the system’s fault-handling capabilities and ensures availability by eliminating human intervention in failures.

[Chubby service's main responsibilities]
## Applications of Chubby
Google has a lot of internal services that need to divide work (at a coarse grain) among multiple servers. These services—Google File System (GFS) and Bigtable—use Chubby as a core part. Let’s explore Chubby’s role in these two services.

- GFS: It uses the Chubby lock to appoint a GFS master server among all the equivalent servers. It also uses Chubby to store and access small amounts of metadata.
- Bigtable: It uses Chubby in more ways than GFS, mainly:
    - Electing the master
    - Enabling the master to discover its controlled servers
    - Enabling clients to locate the master
    - Using it as low-volume storage for the metadata.
```
Note: Chubby is the root of GFS and Bigtable’s distributed data structures.
```

## Infrastructure overview of Chubby
Chubby is a lock service designed for usage in a loosely coupled distributed system comprised of many commodity machines linked by a high-speed network. A Chubby instance, often referred to as a Chubby cell, may support thousands of multi-processor machines linked by a high-speed network.
```
Note: Most Chubby cells are confined within a single data centre. However, to adhere to our system’s replication policy, we must run at least one Chubby cell with replicas thousands of kilometers apart.
```
The client interface of Chubby is similar to a simple file system with the following commonalities:

- Whole-file reads and writes: This is a single API call that reads or writes a full file.
- Advisory locks: These are application-defined arbitrary locks that hold no intrinsic meaning by themselves.
- Notification of events: These include file modifications.

## Requirements
Chubby’s primary purpose is to achieve synchronization consensus among its clients. Let’s define some functional and non-functional requirements of this consensus service.

### Functional requirements
If we closely examine the purpose of Chubby, such a service must have the following functional requirements:

- Coarse-grained locking service: One of the functional requirements of our service is to provide locking service to clients for a longer duration of time and with minimal client interaction with the locking manager.
- Reliable low-volume storage: Our service must also offer storage to clients of a loosely coupled distributed system. The storage capacity provided by the service does not have to be significant; it can work with low volumes because most clients will use this facility to store only metadata.


### Non-functional requirements
In addition to the functional requirements, our service has some non-functional requirements.

- Availability: Our service must have high availability, effectively handling failures. We stated that a Chubby cell serves thousands of machines, and failing one could immensely affect our system’s performance. So, - handling failures is of utmost priority to our service.
- Reliability: Our service needs to be reliable and consistently provide locking and storage services under all circumstances.
- Easy-to-understand semantics: Achieving synchronization is a daunting task in the distributed system world. Hence our service should overcome this limitation and provide an easy-to-understand interface to the developers.
- Throughput: Even though this isn’t a primary goal of our service, it should ensure high throughput for overall better performance.

[Non-functional requirements of Chubby]

## High-level design
The illustration below depicts the high-level design composed of two main components in Chubby that are briefly explained.

### Chubby cell
The Chubby cell is composed of multiple servers (usually five), all of which are replicas of each other. One of these servers is a master, which the clients must communicate with.

#### Server
Each server has a namespace that is composed of directories and files which contain data that is relevant to different applications. In addition to this namespace, the server contains an ACL files directory to have access control lists of all the files and directories within the namespace.
```
Access Control List: A list that tells which processes or people can be given what kind of access to computational resources.
```


### The Chubby library
Communication between clients and servers in a Chubby cell is mediated by the Chubby library. It takes a request from a client who wants to use Chubby service and then finds the relevant cell, directs the request to that cell via remote procedure calls (RPCs), and then reports any changes made in the namespace, data, or metadata (which can also be known as events) back to the client.

[High-level design of Chubby]

```
Note: It’s important to consider the following two things in Chubby’s high-level design:

A node refers to a file or a directory in the namespace tree.

The "ls" used in the paths stands for lock service, it is a prefix that is common to all the names of Chubby nodes.
```

## Bird’s eye view
In the next few lessons, we will design and evaluate Chubby. The following concept map is a quick summary of the problems Chubby solves and its novelties.

[Overview](./overview.png)
