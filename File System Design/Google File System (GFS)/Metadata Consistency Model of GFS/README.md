# Metadata Consistency Model of GFS
## Metadata
The master node stores all of the metadata in memory and serves user requests from there for good performance. Some part of the metadata is stored persistently on the hard disk of the master and also replicated on remote machines, while some metadata is not, as shown in the following illustration. (The part of the metadata that is not persistently stored can be rebuilt if needed. At times, such data is called a soft state.)

[Metadata stored in the master's memory vs. metadata stored persistently on the hard disk](./vs.jpg)

```
Note: In February 2017, AWS's S3 storage system suffered an outage. As part of the recovery, some subsystems needed a full restart, which took many hours. One of the reasons for this delay was because the soft state was being rebuilt. Since such restarts are uncommon for services like S3 (that often haven't been restarted in years), the designer might forget how large their systems have become over the years and how it will impact restart time. As designers, we need to be mindful about the total size of the system state, and how long it could take to rebuild it.

https://aws.amazon.com/message/41926/
```
The location of each chunk's replicas doesn't need to be stored permanently on the master because the master can determine this information from chunkservers through heartbeat messages. The chunkservers share their state and the chunks they hold with the master.

Metadata such as namespaces and file-to-chunk mappings only occur at the master level. We can't determine this metadata from anywhere else if it has been lost as a result of the failure and master's restart. The master checkpoints such metadata state and logs the changes to this metadata in an operation log that is stored on the master's hard disk. On the restart, the master loads the checkpoint and replays the operation log to have the up-to-date namespaces and file-to-chunk mappings in its memory again.


## Shadow masters
To recover from long-lasting failures or permanent failures at the master node, the checkpoints and the operation log are replicated on remote machines, and also maintain the shadow masters. Such shadow masters are only used by clients when they cannot reach the original (primary) master. Shadow masters can facilitate those client queries that don't need any change in the metadata, i.e., the read requests. Additionally, the shadow masters might be a bit behind the primary. A shadow master periodically reads the primary master's operation log from remote persistent storage and applies the changes to the metadata. During this window, clients will get stale metadata if they use any shadow master. The primary master synchronously replicates its checkpoint data and operation log on the remote storage.

We know that when we have replicas of something, there is always a chance of inconsistency among them. In the case of a single master, we might lose some important metadata changes if the master fails before propagating metadata changes to the replicas. GFS ensures there won’t be inconsistencies in the metadata by implementing synchronous replication of the metadata changes. Let's see how synchronous replication works.

```
Question
How do clients find out where a new master is after a possible failure, and how can a client reach the shadow masters?

Answer
We can use the following DNS records:

Primary master 10.1.1.1
Shadow master1 10.1.1.2
Shadow master2 10.1.1.3
When a new server is designated as a primary master, we can update the record in the DNS. Clients can find current primary and shadow masters using the DNS.
```

## Synchronous replication
In synchronous replication, we respond to a client's request only when we are done logging the metadata changes to the operation log placed on the master's hard disk and on all the replicas. The client can see the changes thereafter. The illustration below shows how synchronous replication takes place.

[Synchronous replication](./sync_replication)

Failures can occur on the master. Let’s see how the master handles the client’s operations when it fails while performing a metadata change operation.

- If the master fails before writing a metadata change operation into the operation log, the operations should be retried by the client. The master will be able to perform the operation when it is recovered back. We can add this logic in the GFS client so that the master failure errors are not shown to end users.
- If the master fails after writing the metadata change operation into the log but hasn’t applied the changes, the operation will be there in the operation log. It will be replayed on the master’s recovery. After reconnecting with the master, the client will see that the metadata changes were successfully made, so the client doesn’t need to retry that operation.
- If the master fails after writing the metadata change operation to log, executing the metadata operation, but before returning success to the client. The client can see that the changes it requested are there when it reconnects with the master; it should be smart enough not to request the same operation again (nevertheless, most of the API calls that a client can do returns an appropriate error if redone. For example, making a file twice with the same name, at the same location).


## Summary
For metadata, the GFS provides a strong consistency model so that the system works normally in case of a single master's failure. We might rightfully think that the synchronous replication of each log record will affect the system's throughput and add latency. It is a tradeoff in performance that we must pay to achieve strong consistency in the face of failures.

To reduce the impact of flushing and replication of each individual log record on the overall system's throughput, the GFS batches several log records together. While batching benefits the throughput side, all of the clients in that batch will get the error if the master fails before it has a chance to push the batch out. All of the requests in the batch need to be retried by the client(s) if a failure or timeout happens. This is an example of a tradeoff between throughput and latency. Because IO operations are slow, batching many requests together reduces the trips we take to remote storage, and increases the throughput of storing/synching metadata.

