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

[The new master recreates an approximation of the in-memory state of the old master]

```
Note: All sessions, held locks, and ephemeral files are recorded in the database.
```

### Newly elected master proceedings
The proceedings of the newly elected master are shown in the following slides.

[new master]

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

## Backup
## Mirroring
## Scalability
### Proxies
### Partitioning
