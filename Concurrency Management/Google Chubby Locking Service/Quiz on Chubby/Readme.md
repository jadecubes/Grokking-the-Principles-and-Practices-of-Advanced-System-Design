# Quiz on Chubby
```
Question 1
What is a practical scenario of using Chubby’s locking service for leader election?

Answer
Chubby is an excellent service for leader election because of its fault-tolerant locking. Let’s assume there is a group of processes running in an application. If they wish to choose a leader, each process in the group might ask for a lock on the same Chubby file. The leader is whoever obtains the lock (for the lease period of the lock).

Once the lease expires, for leader re-election, the application can repeat this same process of acquiring a lock on a single Chubby file and electing the one which gets the lock.

To make the whole process more robust, the lock file can be ephemeral so that if the holder of the lock dies, others are notified proactively (instead of waiting for the lease to expire).
```


```
Question 2
What purpose do master and replica servers serve in Chubby?

Answer
It is possible to use only one server to serve clients, but what if that node fails? We will lose all the data on the failed node, and the whole system will become useless. For this purpose, multiple replica servers are introduced in Chubby, whose databases are copies of each other, which can be used if the server serving a client fails.

Now, if multiple replica servers can serve clients, why do we need a master? We need a master server that can entertain all the read and write operations in Chubby and replica servers that keep copies of the master’s database. The master will take all client requests and update the replica servers on any changes. If we start using multiple replica servers to serve clients instead of them just copying the master’s database, it would be hard to keep track of which edits are being made at which server. If any write operation happens on a particular file in any of the servers, and that same file is being read by a client on a different server, this could cause consistency issues. Therefore, we use a single master, responsible for entertaining all read and write operations, thereby keeping the data up-to-date for read operations to not get any outdated data, and keeping all replica servers in sync with its database.

Reading and writing through a single master is the primary mechanism Chubby uses to provide strong data consistency.
```

```
Question 3
Why are write locks not held in a shared mode (like shared read locks)?

Answer
Write locks can only be held in an exclusive mode, which means that while they are being held by a client they cannot be held by any other client unless the lease expires. This is to ensure that if two clients are making edits to the same file, they don’t contradict each other’s writes and corrupt the data leading to a bunch of bad read operations.

Similarly, taking a write lock is not allowed when read locks are active so that data can not be used from under their feet and to provide a consistent data view.
```


```
Question 4
How does a client stay connected with a master?

Answer
When a Chubby client first contacts a master, a session is started with a period called a lease. The session is terminated once the lease period at the master is completed. At the end of the lease period another client can start a session with the master. However, a client can extend its lease period by sending a KeepAlive RPC to the master. When the master receives a KeepAlive RPC request, it temporarily blocks all other RPC calls and only allows it to be completed when the client’s previous lease interval is almost finished. The returning RPC sends cache invalidations and events back to the client.

While doing so is convenient for the client, it also helps to keep the traffic patterns predictable at the master node. Such predictability often translates into better performance (for example, controlling the client traffic with predictable patterns brings the system into a steady state where the master’s state changes slowly, and local processor and master OS caches are used more efficiently).
```

```
Question 5
Using proxies in the middle layer (between clients and the Chubby service), we can scale the Chubby service. Since we’re adding an extra layer, what type of requests get negatively affected by this?

Answer
We introduced proxies as a scaling mechanism for the Chubby service. The addition of a middle layer negatively affects two particular types of requests.

Write requests take longer because they must go to the proxy before getting routed to the Chubby cell.
First-time reads are not cached in proxies and must be forwarded to the Chubby cell after landing on the middle layer.
In short, all requests that the middle layer can’t fulfill and sends to the Chubby cell suffer because of the proxies’ introductions.

Proxies were one of the solutions to reduce the load on the current master node. As often is the case in system design, a problem usually has many viable solutions. We encourage you to brainstorm an alternative method of reducing the load on the master and scale it without the use of proxies. How does your solution compare to a proxy-based solution?
```
