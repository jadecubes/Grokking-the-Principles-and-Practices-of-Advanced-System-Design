# Evaluation of Chubby
## Availability
One of the main requirements of Chubby’s design was ensuring high availability. Even though it’s of prime importance, we should always consider failure probability and avoid approaching services like Chubby as if they were always available.

The global Chubby cell is usually always online since it is uncommon for more than two geographically distant data centers to be down simultaneously. However, a client’s observed availability is frequently different from the global availability for the following two reasons.

- The local cell is usually not partitioned from the client.
- The local cell can be temporarily down, which can happen due to maintenance, which directly impacts the client. Therefore, the client doesn’t notice Chubby’s unavailability.
We can employ three methods to be realistic about Chubby’s availability achievement, particularly that of the global cell.

1. The way we end up utilizing Chubby in an application has a significant impact on its availability. It’s necessary to keep the applications’ availability somewhat independent of Chubby’s availability to gain better availability out of the system.
2. We can rely on additional libraries executing various high-level activities, isolating application developers from Chubby’s downtime.
3. We can analyze each Chubby malfunction to find ways to prevent similar issues in the future and make applications less reliant on Chubby, ultimately leading to an increase in the overall system’s availability.

[Methods to achieve realistic availability for Chubby]
Let’s also evaluate how Chubby handles different failure scenarios to ensure availability.

### Master failure
If a master fails, other replica servers will just wait till the expiration of the master lease and then elect a new master from among themselves using the consensus protocol. The master lease is kept short, just a few seconds, to ensure that replica servers don’t have to wait too long to elect a new master in case it fails.

### Replica failure
If a failed replica is not recovering for a few hours, the following steps occur sequentially:

1. A replacement system is used to replace the failed replica with a new machine from a pool of free machines.
2. It initiates the lock server binary on the new replica.
3. The DNS tables are updated by replacing the IP address of the failed replica with a new one.
4. The current master also has to have this information, for which it keeps polling and eventually gets the information of this change.
5. It updates the cell members in the cell database.
6. All replica servers then replicate this list by using the replication protocol.
7. The new replica server also recieves a copy of the most recent version of the database saved as a backup on a file server, which will be discussed later.
8. It also gets the recent updates from the active replica servers.
9. It is permitted to vote in an election only after it processes a write request that the master sent.

## Reliability
Chubby’s design places a lot of emphasis on replication. This ensures consistent, reliable services. The replication of each server within a cell ensures that no data is lost, and we have proper reliable data backups in case of any failures.

[Evaluation of non-functional requirements of Chubby]

## Easy-to-understand semantics
Many features in Chubby were inspired by already existing systems, ensuring higher developer familiarity and smoother onboarding. One of the main factors ensuring this non-functional requirement is Chubby’s relationship with distributed file systems. Let’s analyze this in general, followed by a specific one.

### Chubby and distributed file systems
Chubby has a straightforward relationship with distributed file systems. Let’s analyze some similarities and differences between the two.

Similarities: Chubby borrows its design ideas from various well-established systems, especially the distributed file system. Chubby gets its cache design, API implementation, and file-naming system from a distributed file system.

Differences: Chubby varies from distributed file systems regarding performance and storage goals:

- Performance: Clients generally do not require high performance unless their data is being cached. They care more about consistency, availability, and dependability. These characteristics are easier to provide when performance is not a significant concern.

- Storage: Because Chubby’s database is practically tiny, we can keep many copies online (typically five replica servers and a few backups). We do complete backups numerous times per day and compare replicas with one another every few hours using checksums of the database state.

Since Chubby’s database is relatively small, we can maintain multiple copies (usually five replicas and a few backups) online. We perform full backups frequently throughout the day and compare the replicas with checksums of the database state every few hours. Due to the lower file system performance and storage requirements, a single Chubby master can handle serving tens of thousands of clients.

### Chubby and Boxwood
## Throughput
## Conclusion
### System design wisdom in Chubby
