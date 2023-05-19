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

## Applications of Chubby
## Infrastructure overview of Chubby
## Requirements
### Functional requirements
### Non-functional requirements
## High-level design
### Chubby cell
#### Server
### The Chubby library
## Bird’s eye view
