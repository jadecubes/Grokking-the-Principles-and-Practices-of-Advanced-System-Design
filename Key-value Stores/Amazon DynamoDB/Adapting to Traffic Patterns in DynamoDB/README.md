# Adapting to Traffic Patterns in DynamoDB
We will allow users to set a provisioned (allocated) throughput for their tables. The initial partitioning of a table divides a table's provisioned throughput equally across all partitions, especially if the range of keys contained in each partition is the same. However, applications might access some keys more frequently. This results in underutilizing dedicated throughput for partitions accessed that are less frequently, and overloading and downtime of partitions that are accessed more frequently. Our partitioning must adapt to customer traffic patterns to solve the problem above.
```
Note: We can think of throughput as the ability of our service to complete fixed-size requests (read or write) per second.
```

## Key considerations
Let's understand the problem in detail. We will start by modeling our problem.

We define read capacity unit (RCU) as the ability of the system to complete one read request of an item of arbitrary size, say x KB. We define write capacity unit (WCU) as the ability of the system to complete one write request of an item of the same arbitrary size. We will use RCU and WCU when talking about throughput. For example, if a table has provisioned throughput of 10,000 RCUs and 5,000 WCUs, then for items of size x KB, at maximum throughput, it cannot read more than 10,000 items and cannot write more than 5,000 items per second.

Continuing this example, let's consider dividing our table into ten partitions with equal key ranges (the disjoint ranges of keys that identify items hosted by a partition). Our design works such that it assumes that all keys have an equal chance of being accessed. As a result, it divides throughput equally among all partitions. So after dividing a table into ten partitions, a single partition will have a throughput of 1,000 RCUs and 500 WCUs. However, what happens when we wish to add or remove partitions from the table or if the table's provisioned throughput changes? One solution is to continue with our assumption that keys are accessed uniformly. Therefore, we will allocate all partitions' throughput based on the table's provisioned throughput.

- When adding or removing partitions, we will distribute a table's provisioned throughput equally among the new number of partitions. So if we added ten more partitions in the table from our example, the total number of partitions would be 20. Each partition will have a throughput of 500 RCUs and 250 WCUs. Moreover, if we removed five partitions from the original number of partitions (ten), the resulting five would have a throughput of 2000 RCUs and 1000 WCUs.

- If the table has provisioned throughput changes, the new throughput would be equally divided amongst the table's partitions. For example, doubling the table's provisioned throughput would double the throughput of each partition.

[In the illustration above, we've added another partition to a table. We did this because there was za request to insert an item with the key ("ClothesType") and the value "Shorts." This key is in the key range covered by the partition (old partition), and the partition was full. Now, the table has two partitions, and its throughput is equally divided among its partitions. Half of the old partition's key range is covered by new partition 1 and the other half by new partition 2.]

In practice, customers or applications disproportionately access keys—some keys are accessed more frequently than others. Suppose we allocate throughput to partitions uniformly, in that case, applications might access some partitions more frequently than others—since some keys are accessed more frequently than others. There are two key things to notice here.

- This non-uniform access to keys leads to non-uniform access to key ranges and, eventually, partitions.

- Access patterns can also change over time. For example, if we have three partitions, A, B, and C. At some point, applications access partition A most frequently, followed by B and C. This can change. Later it could be C that is accessed more frequently than B and A.

The following situations can arise where partitions end up with less available throughput than they require.

One case is when a table's provisioned throughput is divided equally among partitions, and one or more partitions have frequently accessed keys that require more throughput than what they have. We call such partitions hot partitions.

Another case is when a partition requires more space than is available and needs to be split into more partitions. After the split, the table's provisioned throughput will be equally divided in the resultant number of partitions. The partition that was originally split has less throughput than it did before. This is called throughput dilation.

When a partition's allocated throughput does not meet its requirement in real time—that is, the keys in that partition are accessed more frequently than its throughput can handle—it might experience downtime, meaning some of its read and writes will get rejected. This phenomenon is called throttling. Throttling is when a partition's read or write requests get rejected because it is experiencing more traffic than it can handle, either because the traffic requires more throughput than the partition is allocated or because there aren't enough resources. Throttling is necessary to provide consistently good performance to clients. Had we not throttled, the excess load can bog down specific nodes with excessive load.

Throttling makes a partition and our design, in general, less available because it results in periods of unavailability. The first reaction of any user would be to increase the provisioned throughput of their table, but this could very well result in over-provisioning of throughput for partitions that might not need it; only the partition that is experiencing throttling needs more throughput.

We need a solution that allows us to control the throughput of a partition without affecting the throughput of other partitions of its parent table.

## Bursting
It is important to notice that just like partitions sometimes require more throughput than they are allocated, they might be allocated more throughput than they require. We can use this to our advantage when working with co-resident partitions. Bursting a partition means temporarily tapping into the unused throughput of its co-resident partitions.

```
co-resident partitions: partitions stored in the same node
```
We can allow a partition to burst for an arbitrarily short period, say 300 seconds. We cannot allow bursting to permanently use the unused throughput of a co-resident partition since the co-resident partition belongs to another customer and that user has provisioned that throughput for their partition. We also cannot reserve throughput for bursting since that results in underutilizing the node's resources.

We can allow a partition to burst for an arbitrarily short period, say 300 seconds. We cannot allow bursting to permanently use the unused throughput of a co-resident partition since the co-resident partition belongs to another customer and that user has provisioned that throughput for their partition. We also cannot reserve throughput for bursting since that results in underutilizing the node's resources.

[Without bursting, when the number of incoming requests per second (black line) exceeds the number of requests a partition can handle with its provisioned throughput (red line), the difference of requests is rejected because the partition cannot handle such requests. With bursting, we can acquire extra throughput for short periods of increased traffic.]

Bursting is a good solution to absorb temporary spikes in traffic for a partition on a best-effort basis. Our design must be careful only to allow a partition to burst if unused throughput is available. This is a form of workload isolation. Workload isolation between co-resident partitions means that the workload of one partition does not interfere with the workload of any of its co-resident partitions.

By workload, we mean the specific characteristics of read and write traffic. By interfering, we mean any hindrance in these day-to-day workings. For example, one partition cannot complete its write request since the node's disk's I/O is occupied by one or more co-resident partitions.

We can maintain a token bucket system at the node level to enforce workload isolation. Here's how it works. We will have two token buckets for each partition—allocated throughput bucket (or allocation bucket) and an available burst throughput bucket (burst bucket).

1. When a node receives a request (read or write), a control system in the node will check for available tokens in the allocation bucket.

    I. If tokens are available in the allocation bucket, the control system deducts tokens from the allocation bucket, and the request is completed.

    II. If tokens are unavailable in the allocation bucket—the partition has exhausted all the provisioned tokens, i.e., its provisioned throughput—the control system checks token availability in the burst bucket.

        i. The partition is allowed to burst if tokens are available in the burst bucket.

        ii. Otherwise, the request is rejected.
        
[Tokens in the allocation bucket indicate that a partition has provisioned throughput available. The allocation bucket is empty when a bucket's provisioned throughput is being used. Tokens in the burst bucket of a partition indicate if the partition can burst while providing workload isolation. If it is empty, then the partition's node does not have burst capacity: no available throughput without sacrificing workload isolation.]        

The system above executes read requests if tokens are available. For write requests, the system needs to check for token availability of buckets in nodes of the entire replication group. This information can be stored and maintained by the leader replica of the replication group. By doing so, we are prioritizing writes over reads, which benefits a write-heavy workload. However, if traffic is read-heavy, we can allow reads to use any free token from the entire replication group.


## Adaptive capacity
The problem with bursting is that it only works well for short-lived spikes of increased traffic. What if a partition experiences longer periods of throttling?

The only way to handle such situations is to increase the throughput capacity of a partition. This is achieved in two steps:

1. Relocating the partition to a node that has spare throughput available. This is only done if the current node of the partition does not have the required throughput available.

2. Increasing the allocated throughput of the partition that has been experiencing throttling for a long period (ideally longer than the time we allow a partition to burst).

We will call this solution adaptive capacity. Bursting and adaptive collectively can eliminate the problem of non-uniform access to partitions leading to throttling. However, both these solutions have their limitations. Bursting is temporary and only happens when the node has unused throughput. Adaptive capacity is reactive and only happens after throttling is observed. If our service is to guarantee a five nines SLA, then we need to do better.


## Global admission control
We need a solution that achieves more than bursting and adaptive capacity. Bursting does not guarantee that a partition will adapt to throttling since the partition's node might not have unused throughput. Adaptive capacity allows a brief period of inactivity before increasing a partition's throughput.

One solution is to always allow the partition to burst while providing workload isolation. We can build on our token bucket system by moving the tracking of tokens to a more global level. We will call this system global admission control (GAC). We can design a GAC by designing it around the following properties:

- Our GAC will track the consumption of a table using tokens. A GAC is a control center node.

- Request routers maintain local token buckets to allow requests to pass through.

- Request routers communicate to the GAC service to replenish tokens. These requests are made in a matter of seconds.

- All GAC servers tracked by the GAC are part of an independent hash ring.

- Request routers manage tokens and keep deducting tokens as they accept requests. When out of tokens, request routers request the GAC for more tokens.

- The GAC estimates global token consumption using the information provided by the client.

- The GAC allocates tokens that are available to the user's overall tokens.


[GAC replenishes tokens in request router buckets. Requests routers allow requests to pass through to partitions if they have tokens available.]

GAC ensures that workloads that send requests only for some items, or partitions, can execute at maximum partition throughput.

We can keep the partition-level token buckets from our bursting solution and set an upper limit to them to ensure that one application cannot consume all or a significant share of the throughput of storage nodes.


## Splitting for consumption
While GAC enables adjusting to throughput requirements, throttling can still occur when some items are hot for longer periods. It would be better to have some permanent scaling out in such cases.

One solution is to split the partition based on key distribution if the consumed throughput crosses a certain threshold. If we wish to keep keys continuous, the split point should be where the throughput requirement is equally split between the resulting child partitions.


## On-demand provisioning
Our users should not have to choose the right provisioned throughput for applications with highly irregular loads. In such cases, we should regulate throughput based on recent traffic partitions. As soon as we observe a peak, we should either increase the resources available to a partition or split the partition.

## What’s next?
Next, we will learn how to maintain a healthy database. We will explore dealing with hardware failures, silent data errors, and other problems that can arise in a database.
