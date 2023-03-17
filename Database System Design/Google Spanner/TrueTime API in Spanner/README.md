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

For an event, e, consider the absolute time t_ abs (e). The TrueTime API will guarantee the following for an invocation:

tt=TT.now(),tt.earliest<=t_abs(e_now)<=tt.latest where e_now is the invocation event.


Question
Why is the error bound half of the interval’s width?

Answer
Consider s to be a timestamp returned by the TrueTime API. When we request TT.now() at s, we get an interval of the earliest and latest timestamps. Ideally, the difference between s and earliest or s and latest is ϵ. It means the total interval is twice the ϵ.
[TrueTime]


## Real-time dependency in TrueTime
## Controlling uncertainty in TrueTime
## Minimize uncertainty in TrueTime
## Summary
