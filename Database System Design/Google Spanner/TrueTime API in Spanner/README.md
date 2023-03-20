# TrueTime API in Spanner
TrueTime is a highly available distributed clock service. With TrueTime, applications can produce timestamps that increase monotonically. For example, if the generation of the timestamp T ends before that of the timestamp T' begins, then the resultant timestamp T' will be bigger than the previous timestamp 
T (with in an error bound). This is guaranteed for all servers and at any given time.

Spanner assigns timestamps to each transaction using the TrueTime API. The timestamp is the exact time Spanner considered the transaction to commit. Spanner's multi-version concurrency control guarantees the ordering of transactions on timestamps. It allows clients to execute consistent reads over the database, even when spread across different cloud regions, without blocking writes.

Let's dive into the details of the TrueTime API.

## TrueTime API methods
As per Spanner's published paper, the methods that the TrueTime API allows are as follows:

```
TrueTime API Methods
Method                   Returns

TT.now()          TTinterval: [earliest, latest]

TT.after(t)       True, if t has definitely passed

TT.before(t)      True, if t has definitely not arrived
```
The argument t is of the TTstamp type. TrueTime reports an interval of time, TTinterval, instead of an exact timestamp. The interval with bounded time uncertainty is represented directly in TrueTime. All endpoints of TTinterval must be of the TTstamp type. Upon calling TT.now(), we'll get back a TTinterval (earliest and latest values) that includes the precise moment that we want. The values are the earliest possible and latest possible timestamps. As per Spanner's published paper, the time epoch is analogous to the UNIX time epoch with leap-second smearing. ϵ represents instantaneous error bound, and ϵ' represents average error bound. ϵ is half of the interval's width.

```
Spanner paper: Corbett, James C., Jeffrey Dean, Michael Epstein, Andrew Fikes, Christopher Frost, Jeffrey John Furman, Sanjay Ghemawat et al. "Spanner: Google’s globally distributed database." ACM Transactions on Computer Systems (TOCS) 31, no. 3 (2013): 1-22.
```

For an event, e, consider the absolute time t_abs (e). The TrueTime API will guarantee the following for an invocation:

```
tt=TT.now(),tt.earliest<=t_abs(e_now)<=tt.latest where e_now is the invocation event.
```
where e_now is the invocation event.

**Question**

Why is the error bound half of the interval’s width?

**Answer**

Consider s to be a timestamp returned by the TrueTime API. When we request TT.now() at s, we get an interval of the earliest and latest timestamps. Ideally, the difference between s and earliest or s and latest is ϵ. It means the total interval is twice the ϵ.
[TrueTime]


## Real-time dependency in TrueTime
The TrueTime API is based on physical clocks that explicitly capture uncertainty in the timestamps. Consider that a replica A wants to commit transaction T1, so it requests a timestamp from TrueTime. The API responds with TTinterval that has a range of two timestamps, the earliest possible and the latest possible.

There can be no perfect synchronization of clocks in the systems, since we are not sure about the current real physical timestamp. However, we can track all the errors and uncertainties in the system.

We can determine TTinterval by correctly accounting for the uncertainty. Therefore, this means we have to track the following:

- The round trip time to the clock server
- The clock drift
- The time taken to synchronize with other clocks

All these potential causes of error and factor add into a single uncertainty interval. We know the real timestamp is between earliest and latest. The transaction will wait, and the time it’s going to wait is exactly the difference between the two timestamps, which is 2∗ϵ. The transaction waits for that time and holds all the locks until the wait time is over and then commits the transaction and releases all the locks. This extra wait time is the key since it provides serialization.

Let’s assume that replica, B wants to execute transaction, T2. It requests the TrueTime API, and after receiving TTinterval, it waits for 2∗ϵ times and then commits the transaction.

```
Note: The average value of ϵ is the same for all replicas. However, it can vary since ϵ is calculated based on the local clock drift and other uncertainties a replica faces while fetching the value of TT.now() from the TrueTime API. To differentiate between the ϵ calculated for each transaction, we will use ϵ1 for transaction T1 and ϵ2 for transaction T2.
```

Now, we have a real-time dependency. T2 that started after T1 ended. This waiting ensures that if we have these two timestamps where there’s a real-time dependency, then their uncertainty intervals from TrueTime will not overlap. Since uncertainty TrueTime intervals do not overlap, the commit timestamp for T1 will be a lower timestamp than the commit timestamp for T2. By having these uncertainties with non-overlapping periods, we have eliminated the possibility of the timestamps getting reordered or being inconsistent with causality.

[TrueTime]

The key here is waiting to commit. In a real system, we want to wait for the minimum time. So we now need to do two things:

1. We need to quantify the uncertainty so we know how long we must wait.

2. We have to keep the uncertainty as small as possible so that our waiting time is as short as possible.


## Controlling uncertainty in TrueTime
TrueTime relies on atomic clocks and Global Positioning System (GPS) coordinates to keep accurate time. Due to their varied failure mechanisms, TrueTime employs two-time reference sources. It is possible for the integrity of the GPS reference source to be compromised by a number of factors, including antenna and receiver failures, local radio interference, correlated failures, and GPS outages. Due to frequency inaccuracies, atomic clocks can drift dramatically over extended periods of time. The failure modes of atomic clocks can be independent of the GPS and one another.

```
Correlated failures: For example, design errors like erroneous leap-second handling and spoofing.
```

Each data center has its own set of time master machines, and each machine has its copy of the time follower daemon. Most masters use GPS receivers with dedicated antennas and keep their systems physically apart to lessen the impact of antenna failures, radio interference, and spoofing. Armageddon masters are the surviving masters that have atomic clocks. The price of an atomic clock is comparable to that of a Global Positioning System (GPS) master. The various masters' time standards are often checked against one another. Every master compares the speed at which its reference advances time to that of its local clock. If the two times are significantly out of sync, then the master removes itself. Using a conservative estimate of the worst-case scenario for clock drift, Armageddon masters advocate for an ever-increasing interval between synchronizations. GPS masters announce uncertainty that is approximately zero.

```
Question
Why do we use two sources (GPS and atomic) for clocks?

Answer
When compared to competing protocols, Spanner’s ability to keep servers’ clocks in sync is superior because it relies on GPS and atomic clocks as its underlying time reference.

Both have different failure characteristics. However collectively, they provide a more fault-tolerant system as compared to using just one of them.
```

Each daemon will poll multiple masters to lessen its dependence on any one master and, therefore, its susceptibility to failure. Some are handpicked GPS masters from close-by data centers, while others come from further afield, along with a few Armageddon masters. The clocks of the local machine and non-liars are synchronized. Daemons use Marzullo's algorithm's modified version to identify and remove the liars. Computers with frequency excursions larger than the worst-case bound are booted to prevent faulty local clocks.

```
Frequence Excursion: They are determined using specifications of component and the operating environment.
```

```
Marzullo’s algorithm is an agreement method used to choose reliable time sources from a set of potentially noisy ones.

Marzullo’s algorithm is a time-efficient approach for selecting the best interval from a collection of estimates accompanied by the intervals. The real value may lie outside the confidence interval for some sources. Accordingly, the optimal estimate is the narrowest interval that accounts for the maximum number of sources.

For more details, you can refer to Marzullo’s paper, Marzullo, Keith, and Susan Owicki. “Maintaining the time in a distributed system.” In Proceedings of the second annual ACM symposium on Principles of distributed computing, pp. 295-305. 1983.
```

The following slides explain how TrueTime’s time master servers work with the GPS and atomic clocks in multiple data centers. In every data center, we have time handlers. GPS timemasters have GPS receivers attached, and few have atomic clocks. When a client needs to call the TrueTime API, the client communicates to the local daemon. The daemon (a background service that runs on all nodes in the data center) mostly contacts GPS time masters and sometimes contacts atomic clock time masters to get the redundancy of different time references. Marzullo's algorithm is run, which intersects time intervals to determine a time reference. The API gives an interval from the earliest to the latest.

[TrueTime]

## Minimize uncertainty in TrueTime
A daemon alerts of increasing time uncertainty between synchronizations. The value comes from using the worst-case scenario for possible local clock drift. It is also influenced by the unpredictability of the time taken by masters and the lag in receiving information from them. This will result in a sawtooth function of time. Throughout every poll period, the function ranges between 1 ms to 7 ms. On average, it is 4 ms. The sawtooth boundaries of 0 to 6 ms result from the present settings for the poll interval (30 seconds) and the applied drift rate (200 microseconds/second). One millisecond is lost because of slow messages reaching the time masters. In the presence of setbacks, deviations from this sawtooth are possible. For instance, when the time master is unavailable on rare occasions, it can result in data center-wide increases in ϵ. Similarly, occasionally isolated ϵ spikes can result from overburdened machines or network connectivity.

The following slides show the progress of epsilon on a node over time that forms a sawtooth each time the client requests to the TrueTime API.

[TrueTime]

## Summary

There are two key insights in this lesson:

1. While it is true that perfectly synchronizing clocks in a distributed system is very hard, we can control the amount of uncertainty in clock readings. Using special hardware, we can reduce such uncertainty.

2. The TrueTime API seems innocuous, but it is very powerful. It enables us to do what a vector clock does under certain conditions (having timestamps with causality in a large system where the difference between the two timestamps is large than clock uncertainty). Vector clocks are expensive in space use, but TrueTime does the same work using a simple timestamp. We will see in later lessons that TrueTime provides more than causality because its timestamp is related to the wall clock time.

We learned how the TrueTime API works. It seems that TrueTime allows Spanner to overrule the CAP theorem by providing high availability and consistency simultaneously. In the next lesson, we will discuss the CAP theorem and Spanner.

