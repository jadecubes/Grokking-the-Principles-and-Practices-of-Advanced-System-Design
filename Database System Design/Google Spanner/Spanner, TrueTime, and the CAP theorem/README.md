# Spanner, TrueTime, and the CAP Theorem
## CAP theorem
Spanner is a global, highly available SQL database that easily handles massive amounts of data and transaction volume due to its replication capabilities. All data inserted into the database is given a timestamp that is accurate in the present, and clients can perform global consistent reads without requiring locking.

According to the CAP theorem, only two of the following three features can coexist in a given system:

C: Consistency

A: Availability

P: Tolerance to network partitions

Depending on which letter we omit, we get three distinct systems: CA, CP, and AP. For distributed systems, partitions are often unavoidable. Therefore, any distributed system has to give up either consistency (AP) or availability (CP), which is a tradeoff no one wants to undertake.

[The CAP Theorem](./cap.png)

The CAP theorem was developed to make designers think about this trade-off. However, there are two crucial caveats:

1. We only need to forfeit something during an actual partition.

2. The actual theorem is about 100% availability.

## Does Spanner violate the CAP theorem?

Spanner claims it is consistent and available even if it's a globally distributed system. Although it seems that Spanner has disregarded the CAP theorem, it is a CP system, because when partitions occur, Spanner opts for C and gives up A.

Standard RDBMS implementations are usually CA, but they cease to be CA systems in a distributed environment because of the network partition. For Spanner to claim to be a CA system, it should be in the following states of relative probabilities:

1. High availability of the system in real time so users are not impacted, or the impact is minimal

2. Low fraction of partition outages

Spanner does not guarantee 100% uptime. It gives five-nines of availability which means there are five minutes of downtime a year. Most relational databases require ten minutes or more a month for maintenance and so on. To upgrade the schema, the databases require downtime too. Therefore, this is how Spanner backs up its "essentially CA" claim by providing high availability so users can tolerate outages.

The following illustration shows the outages of the Spanner due to unexpected events (the data is taken from Brewer, Eric. "Spanner, TrueTime and the CAP theorem." (2017)). Most events are caused by the Users themselves, whether through overload or incorrect configuration, but the other categories may affect several users. Events in a cluster indicate a larger issue with the underlying infrastructure, such as servers or power. Spanner will automatically switch to a different replica to avoid downtime. Misconfigurations and Site Reliability Engineers (SREs) can lead to mishaps known as operator events. A bug is an error in the software that causes a malfunction, which may or may not affect the entire system. Both of the most serious disruptions occur because of software flaws that simultaneously strike every database replica. The other category includes a wide range of issues, most of which occur only once.

[The percentage of Spanner outages due to unexpected events](./issues.png)

Partitions and networking configuration issues fall within the Network category, which account for less than 8% of all issues. Google Spanner faced network issues in some data centers or regions where they lost connectivity with the rest of the network. They also experienced brief periods of poor latency due to hardware failures and misconfigurations with underprovisioned bandwidth. In one case, a failure in one traffic channel led to an unusual partition that could only be fixed by shutting down several nodes. There have been no major disruptions due to networking issues so far. Hence, Spanner has a low fraction of network outages.

Spanner is not completely a CA system but can act like one due to high availability and low network outages.

### Network in Spanner
It seems like Spanner circumvents CAP by utilizing the TrueTime API. TrueTime allows for time zones to be kept in sync with one other on a global level. While it is incredible, TrueTime does not fundamentally aid in achieving CA. Rather, it is the network. Due to Google's extensive network and years of operational advancements, network partitions are extremely rare.

Google has a private worldwide network. Every Spanner packet (except edge links to remote clients) only travels across Google-managed routers and networks. Every pair of data centers also benefits from path diversity thanks to the fact that each data center has at least three separate fibers linking it to the private global network. There are also backup hardware and connectivity routes within a data center. This means that disruptions in service, such as those caused by severe events like cutting fiber lines, are avoided.

This means that a software or configuration update that breaks numerous paths at once poses a greater threat to a partition than a cut path ever could. When a poor update is eventually pushed, the usual aim is to restrict that update's impact or blast radius so that just a subset of pathways or replicas are lost. This is done before attempting any other sorts of alterations.

[The bad update gets replicated to other nodes](./bad_update.png)

Good network engineering and operations can drastically reduce the possibilities of partitions. Across continents, the minimum round trip time for consistent operations can be in the tens of milliseconds. Due to the need for a balance between latency and disaster tolerance, Google defines "regions" as having a 2ms round-trip time. The global timestamps and the option to utilize a local replica combine to make reads extremely fast and reliable.

```
Tens of milliseconds: A thousand miles is nearly five million feet, thus at half a foot each nanosecond, the bare minimum would be ten milliseconds.
```

Potentially faster update latencies could be achieved in a model with reduced consistency. A disaster could wipe out the local site and delete all copies before the data is replicated in another region. Without the long round trip, it would also have a window of lower durability.

### Network partitioning
Spanner maintains isolation and high consistency using the two-phase commit (2PC) and strict two-phase locking, just like other ACID databases. 2PC is considered an anti-availability protocol since all members must participate for 2PC to function. Spanner increases the availability and performance of 2PC by using leaders of the respective Paxos groups. Now, even if a 2PC participant dies, it will be replaced by a leader election. Since the state of 2PC was written to the respective Paxos log, the new leader can resume 2PC where the old one left off. Groups of data serve as the fundamental building block for placement and replication.

Spanner opts for C over A during a partition due to the following:

- To reach a consensus on an update, Paxos groups are used. If a partition prevents the leader from maintaining a quorum, updates are halted, and the system is unavailable. At some point, a new leader may take over, but that calls for a vote of confidence from the majority.

- A partition of the members can block commitments if the two-phase commit is used for cross-group transactions.

Mostly, the outcome of the partition is that one side has a quorum and will continue electing new leaders. Therefore, the service keeps running, but a few users face accessibility issues. However, this is an instance where differences in availability are considerable. Those users almost certainly also have other serious issues, such as a lack of connectivity or are down. As a result, multi-region services developed on top of Spanner continue to function reasonably effectively even after a partition has occurred. The likelihood of some groups being unavailable is low, but it is a possibility.

All groups that are affected during a Spanner transaction must have a quorum-elected leader and be on the same side of the partition for the transaction to be successful. That's why sometimes a transaction will go through smoothly, while other times it will time out, yet both scenarios are consistent. Regardless of whether or not the transaction is rolled back, if Spanner returns any reads, they will be consistent for any reason, including time-outs.


## TrueTime's role
In a distributed system, communication can typically be avoided using synchronized clocks. TrueTime is a limited non-zero error, globally synchronized clock. It delivers a time interval that is guaranteed to include the clock's actual time for some time during the call's execution. This means that calls were certainly ordered in real time if no two intervals overlap. We cannot determine an accurate order if the intervals are overlapping.

```
communication can typically be avoided using synchronized clocks: See Barbara Liskov's paper for details: Liskov, Barbara. "Practical uses of synchronized clocks in distributed systems." (1991)
```

Spanner achieves serializability via locks, while TrueTime ensures external consistency (linearizability). For any two transactions, T1 and T2 (even if on different continents), Spanner's external consistency states that if T2 begins to commit after T1 completes committing, then T2's timestamp is greater than T1's timestamp. Moreover, Spanner uses TrueTime as a clock that allows it to ensure invariants holds, especially while committing a transaction. During the commit, the replicas wait until they are sure to commit. The wait time is generally not longer. Waiting out the uncertainty is a common pattern, and monotonically increasing timestamps are required for external consistency in general.

```
Serializability: In concurrency control of databases, transaction processing (transaction management), and various transactional applications (for example, transactional memory) and software transactional memory, both centralized and distributed, a transaction schedule is serializable if its outcome (for example, the resulting database state) is equal to the outcome of its transactions executed serially, that is without overlapping in time. It is considered the highest level of isolation between transactions and plays an essential role in concurrency control. Strong strict two-phase locking (SS2PL) is a popular serializability mechanism utilized in most the database systems. [source: Wikipedia]
```

A consistent snapshot for a timestamp t allows us to read the version before that timestamp, so we do not need to block any write transactions. This shows the worth of TrueTime as it makes reading from a pasttime easy. It is possible because Spanner versions all changes, and each change is given a TrueTime timestamp. This could have been problematic if snapshots aren’t transactionally consistent, since this could reveal a partially applied transaction that breaks an invariant or integrity constraint. Inconsistency makes it difficult to restore from backups. For example, Bigtable stores past versions, but the concept of time is jagged across data nodes. It produces inconsistent and unpredictable results. However, with Spanner, we can get consistent results against a timestamp.
 
The partitioning can slow down TrueTime. GPS receivers and atomic clocks keep precise time with extremely small drift and serve as the underlying time source. With time masters in each data center, precise time is likely to be maintained on both sides of a partition. However, without communication with the masters, the time at any given node will drift. Because of these constraints on the rate of local clock drift, their intervals gradually expand over the course of a partition. This causes delays in TrueTime-reliant operations like Paxos leader elections and transaction commits, but ultimately the operation succeeds.

In conclusion, Spanner is essentially a CA system despite its distributed nature due to its reliability and high availability, which is more than five-nines. Serializability in Spanner is accomplished by a two-phase commit, whereas TrueTime provides consistent reads without locking, external consistency, and consistent snapshots.


[Consistent snapshots ease storage backups]

```
Note: This lesson primarily relies on the content taken from Brewer, Eric. "Spanner, TrueTime and the CAP theorem." (2017)
```
