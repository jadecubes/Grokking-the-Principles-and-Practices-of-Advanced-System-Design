# Quiz on 2PL

```
Question 1
Consider a transaction with the following operations: read[a]→write[b]→commit, follows this schedule for its execution: readlock[a], read[a], readunlock[a], writelock[b], write[b], writeunlock[b].

Based on your understanding of the basic rules of 2PL, does the schedule above follow those rules?

Answer
The schedule above violates the basic rule of 2PL that a transaction can acquire no lock once it has released even one lock. Since the schedule above states that the transaction readunlocks[a] and then the transaction acquires a writelock[b], it doesn’t follow 2PL. A correct execution schedule can be as follows: readlock[a], read[a], writelock[b], write[b], readunlock[a], writeunlock[b].
```


```
Question 2
Suppose there are three transactions, T1, T2, and T3, with timestamps 2, 4, and 6, respectively. How will the deadlock prevention techniques handle the following two cases?

T1 requesting an object held by T2
T3 requesting an object held by T2
Answer
The two techniques will handle the requests in the following way.

1. Wait-die:

If T1 requests an object held by T2, then T1 will wait because old waits.
If T3 requests an object held by T2, then T3 will be terminated (die).

2. Wound-wait:

If T1 requests an object held by T2, then T2 will be aborted (wounded).
If T3 requests an object held by T2, then T3 will wait.
```

```
Question 1
Consider a transaction with the following operations: read[a]→write[b]→commit, follows this schedule for its execution: readlock[a], read[a], readunlock[a], writelock[b], write[b], writeunlock[b].

Based on your understanding of the basic rules of 2PL, does the schedule above follow those rules?

Answer
The schedule above violates the basic rule of 2PL that a transaction can acquire no lock once it has released even one lock. Since the schedule above states that the transaction readunlocks[a] and then the transaction acquires a writelock[b], it doesn’t follow 2PL. A correct execution schedule can be as follows: readlock[a], read[a], writelock[b], write[b], readunlock[a], writeunlock[b].
```
