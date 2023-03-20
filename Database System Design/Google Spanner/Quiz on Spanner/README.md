# Quiz on Spanner

```
Question 1
How does Spanner provide serializability to transactions?

Answer
Spanner relies on two-phase locking (2PL) to provide serializability across concurrent transactions.

(Remember that serializability is an isolation property of transactions.)
```

```
Question 2
How does Spanner provide external consistency to transactions?

Answer
Spanner relies on TrueTime and other invariants to assign commit times to a transaction for external consistency.
```

```
Question 3
For what purpose does Spanner use a two-phase commit (2PC)?

Answer
Spanner uses 2PC to provide atomicity (all or nothing) to transactions.
```

```
Question 4
Isn’t the CAP theorem applicable to the Spanner system?

Answer
The CAP theorem applies to the Spanner system as well. Spanner chooses C over A when network partitions happen to provide a strongly consistent distributed database.

Due to Google’s use of its private (well-engineered with three or more redundant paths and well-operated) wide-area network, partitions are rare, and we get the illusion that the Spanner system is always available.

There is a lot of nuance around the CAP theorem. We covered it in the Spanner, TrueTime, and CAP Theorem lesson.
```

```
Question 5
What will happen if the uncertainty increases in TrueTime?

Answer
Different transactions might need to wait longer to hold different timing invariants. Therefore, a higher value of clock uncertainty will translate into lower performance and no correctness problems.

NTP-based clocks can have uncertainty anywhere from 100 ms to 250 ms. The Spanner system’s uncertainty is mostly kept around 7 ms. Therefore, we can build a Spanner-like system using NTP-based clocks, but its performance will be much worse than Google’s Spanner because Spanner will need to wait for less (actually two orders of magnitude less).
```

```
Question 6
Can the Spanner system tolerate Byzantine faults?

A Byzantine fault is a condition of a computer system, particularly distributed computing systems, where components may fail, and there is imperfect information on whether a component has failed. In a Byzantine fault, a component such as a server can inconsistently appear as both failed and functioning to failure-detection systems, presenting different symptoms to different observers. It is difficult for the other components to declare it failed and shut it out of the network because they need to first reach a consensus regarding which component has failed in the first place. Byzantine fault tolerance (BFT) is the resiliency of a fault-tolerant computer system to such conditions. [source: Wikipedia]

Hide Answer
No. Spanner assumes a non-Byzantine fault environment. As an example, nodes whose local clock drift is exceeding the maximum set by the operators of Spanner can self-extract themselves from the system. Now, if a node was Byzantine, it can do all kinds of malicious activities with time. For example, artificially exceeding uncertainty bound on the local node or not self-exiting on such an event.

Spanner being deployed on Google’s data centers and using their private network is safe with its assumption of a non-Byzantine fault model.
```


```
Question 7
We saw how TrueTime helps Spanner provide serializable and linearizable transactions. Can we use the TrueTime API for other things in a distributed system? For example, can we timestamp an event in a distributed system using TrueTime, and can we compare two arbitrary timestamps in the system for causality?

Answer
Let’s assume we have two TrueTime-based timestamps, T1 and T2, associated with two events on two nodes in the distributed system. If we know that our TrueTime uncertainty was always under the max (say 7 ms) and if T1 and T2 are apart by more than 7ms, then T1 would have happened before T2 if the value of T1 is less than the value of T2. So under such conditions, TrueTime can provide the functionality of a vector clock.

Another way to enforce vector clock-like timing characteristics for a few events could be to wrap those events around synthetic read abstractions of Spanner. Doing so will consistently assign timestamps. Though such a solution will incur the cost of read transactions.

Though we should remember that real-world cause-and-effect chains might not be inferred with TrueTime timestamps. For example, if two timestamps of two unrelated events in two different systems have timestamps where one was smaller than the other, it does not mean that the first event caused the second event.

We encourage you to think further about how the availability of the TrueTime API with small uncertainty can help perform different functions in a distributed system.
```
