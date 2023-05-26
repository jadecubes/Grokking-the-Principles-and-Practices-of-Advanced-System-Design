# Detailed Design of Chubby: Part IV
## Failovers
Nodes can fail, and we need to have an excellent strategy to minimize downtime. One way to reduce downtime is a fast failover. Let’s discuss the failover scenario and how our system handles such cases.

A master discards its state about sessions, handles, and locks when it fails or loses leadership. This must be followed by the election of a new master with the following two possible options:

1. If a master gets elected rapidly, clients can connect with the new master before their own approximation of the master lease’s (local lease) duration runs out.
2. If the election extends for a longer duration, clients discover the new master by emptying their caches and waiting for the grace period. As a result, our system can keep sessions running during failovers that go past the typical lease time-out, thanks to the grace period.
Once a client has made contact with the new master, the client library and the master give this impression to the application that nothing went wrong by working together. The new master must recreate the in-memory state of the master that it replaced. It accomplishes this in part by:

- Reading data that has been duplicated via the standard database replication technique and kept safely on disk
- Acquiring state from clients
- Making cautious assumptions

[The new master recreates an approximation of the in-memory state of the old master](./newmaster.png)

```
Note: All sessions, held locks, and ephemeral files are recorded in the database.
```

### Newly elected master proceedings
The proceedings of the newly elected master are shown in the following slides.

[new master](./newlyelected.png)

```
Question 1
Corresponding to slide 8 of the slide deck above, how does the new master prevent the recreation of an already closed handle?

Answer
To prevent a delayed or duplicated network packet from unintentionally recreating a closed handle, the master stores any closed handles in memory so that the new master does not recreate them in this epoch.

A faulty client can regenerate a closed handle in a subsequent epoch, but this is harmless since the client is faulty in the first place.
```

```
Question 2
Why does the client library struggle to keep the old session going? Why not simply make a new session when we can contact the new master?

Answer
A new session comes with many pre-steps, and we want to finish a session without a failover. The system tries to mimic this effort and keep the old session going by relying on the already-established connection and stored information. In case of a switch, we’ll have to duplicate most of the previous work, and practically we want to keep this to a minimum.
```

### Example scenario
The following illustration depicts the progression of an extended master failover event where the client must use its grace time to keep the session alive. The time grows from top to bottom, but we won’t scale it since it’s not linear.
```
Note: Arrows from left to right show KeepAlive requests, and the ones from right to left show their replies.
```

[Example](./example)

```
Note: During the grace period, the client is uncertain as to whether its lease at the master has ended at this time. It does not end its session, but it does stop all API requests from applications to stop them from seeing erroneous data.
```

```
Question
How did Google implement Chubby’s database?

Answer
The database for Chubby’s initial iteration was a replicated copy of Berkeley DB. In the successive iterations, Google did some modifications as well to make it best for their use case. Google’s implementation is mapped to Berkeley’s DB design based on the following points, with some modifications:

- Berkeley DB offers B-trees that translate arbitrary byte-string values to byte-string keys. Based on this, Google implemented a function that compares the keys to sort the path names based on the number of components in each path. This ensured that the siblings are maintained adjacent in the sort order while allowing nodes to be keyed by their path names

Note: Chubby does not employ path-based permissions. Therefore, each file access only requires a single database query.

- Berkeley DB replicates its database logs over several servers using a distributed consensus approach. This made it similar to Chubby’s concept after Chubby introduced master leases, which simplified implementation.

- The replication code for Berkeley DB was just recently introduced at the time of Chubby’s release, although the B-tree code was well-established and in use. Google created a straightforward database utilizing write-ahead logging and snapshotting because the exposed replication code was more prone to risks.

As discussed previously, a distributed consensus protocol was used to disseminate the database log across the replicas. Chubby only utilized a small portion of Berkeley DB’s functionality. Therefore, Google’s rewriting allowed for a significant system-wide simplification. For instance, Google required atomic operations but did not require general transactions.
```

## Backup
Each Chubby cell’s master takes a snapshot of its database every few hours and uploads it to a GFS file server located in a separate building. Using a different site guarantees that the backup in GFS will endure any damages on a single site and that the backups do not incur cyclic dependencies into the system; otherwise, a GFS cell deployed at a single site may depend on the Chubby cell to choose its master.

[Chubby's backup strategy](./backup.png)

Backups offer both disaster recovery and a way to set up the database of a replacement replica without putting additional stress on active replica servers.

## Mirroring
Chubby enables mirroring a group of files from one cell to another. The fact that the files are short and the event system promptly notifies the mirrored code whenever a file is created, removed, or updated, and makes mirroring quick. Changes are mirrored in dozens of mirrors worldwide in under a second, assuming no network issues. A mirror that cannot be accessed remains unaltered until communication is reestablished. Then, updated files are located by contrasting their checksums.
```
Mirroring: A technique that allows a system to automatically maintain multiple copies.
```
[Mirroring a Chubby cell across five different locations](./mirror.png)

Most frequently, configuration files are copied to numerous computing clusters dispersed worldwide via mirroring. A unique cell called global has a subtree called /ls/global/master that is mirrored to every other Chubby cell’s subtree called /ls/cell/shadow. The global cell is unique because it can nearly always be reached by the organization owing to its five replica servers spread across five different geographical locations.

Some of the files mirrored from the global cell are listed below:

- Chubby’s ACLs
- The files that Chubby cells and other systems use to let the monitoring services know of their presence
- Pointers to large datasets like Bigtable cells (these pointers help clients find these datasets)
- Configuration files for other systems.

## Scalability
Google observed that 90,000 clients communicate directly with a Chubby master, which is significantly greater than the number of computers involved. Since Chubby’s clients are independent processes, Chubby must manage more clients than one might anticipate (because each node can have many processes communicating with Chubby). There is only one master per cell and its machine is the same as the clients. This allows clients to vastly outnumber the master. Therefore, communication with the master is significantly decreased using the best scaling strategies.

We can utilize several mechanisms to scale Chubby. A few of them are as follows:

1. Minimizing round-trip time (RTT): Any number of Chubby cells can be created; clients usually use a local cell (located with DNS) to avoid relying on distant computers. For a data center with several thousand computers, our standard setup requires one Chubby cell.
2. Minimizing KeepAlive loads: When under a substantial load, the master may raise lease times from the standard 12 seconds to around 60 seconds, requiring it to process fewer KeepAlive RPCs (KeepAlives are by far the most common request, and their inability to be processed promptly is the typical failure mode of a server that is overloaded–clients are often oblivious to variations in latency in other calls.).
3. Optimizing caching: To minimize the number of calls Chubby clients make to the server, they cache file data, metadata, the absence of files, and currently open handles.
4. Protocol conversions: We use protocol-conversion servers to convert the sophisticated Chubby protocol into other less complicated protocols like DNS. We’ll review proxies and partitioning as two such scaling mechanisms below.

[Scaling mechanisms for Chubby](./scaling.png)

```
Question
Are there any associated drawbacks of using proxies?

Answer
For writes and first reads, proxies add an extra RPC. For these specific requests, each proxied client depends on two servers now, namely, its proxy and the Chubby master. Since both of them have a chance of malfunctioning, one can anticipate that by using proxies, the client temporarily feels the failed service at least twice as frequently as before.
```

### Proxies
Trusted processes that relay requests from other clients to a Chubby cell can proxy (use the same protocol on both sides) Chubby’s protocol. While a proxy can handle both KeepAlive and read requests to lessen server load, it cannot lessen write traffic since it goes via the proxy’s cache. Proxy servers enable a considerable increase in the number of clients because write traffic is significantly less than 1% of Chubby’s typical workload, even with aggressive client caching.

Proxies can positively impact different requests in the following ways:

- KeepAlive traffic is decreased by a factor of N_proxy if a proxy manages N_proxy clients, which might be 10,000 or more per cell.
- At best, a proxy cache can reduce read traffic by a factor of 10 or by the average amount of read-sharing.
However, as reads only make up around 10% of Chubby’s current load, the reduction in KeepAlive traffic is far more significant (in one of Chubby’s installations at Google, they observed 1% write traffic, 10% read traffic, and the rest for the other operations related to sessions, etc.).

[Scaling Chubby using proxies](./proxy.png)

### Partitioning
We can shard Chubby’s namespace for scalability. Doing so, a Chubby cell would consist of N partitions, each including a set of replicas and a master.

Let’s assume that we have a directory D and there are one or many child leaf nodes C with the namespace of D/C. If we enable partitioning, the partition P(D/C)=hash(D) mod N would contain all of the nodes D/C in the directory D. Since the metadata of a directory D may be on its parent node— let’s call it D′— then a separate partition P(D)=hash(D′) mod N may store the metadata for D.

Partitioning in Chubby was done to facilitate large Chubby cells with minimal communication between the partitions. However, that’s not always the case: despite the absence of hard links, directory modification times, and cross-directory rename operations in Chubby, several operations still call for cross-partition communication. Here are a few sample scenarios:

Most clients read publically available files that do not require an ACL. ACLs are files themselves and are easily cached; however, one partition may utilize another to verify permissions, especially for Open() and Delete() calls, as they require an ACL check.
When a directory is removed, a cross-partition call may be required to verify that the directory is empty.
```
Note: Since most calls are handled individually by each system partition, we anticipate that this communication will only have a minor effect on performance or availability.
```
One would anticipate that each client will contact most of the partitions unless N is a very big number. Consequently, partitioning decreases read and write traffic on every given partition by a factor of N, but partitioning does not always reduce KeepAlive traffic because the ongoing sessions might span across partitions.
```
Note: We can use a combination of proxies and partitioning should Chubby need to handle more clients.
```
This marks the end of our discussion on design. The next lesson will discuss the rationale behind various design decisions.
