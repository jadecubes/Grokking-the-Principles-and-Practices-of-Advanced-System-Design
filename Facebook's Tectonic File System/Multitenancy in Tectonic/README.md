# Multitenancy in Tectonic
## Preamble
Remember that one of the primary goals of Tectonic is to efficiently utilize storage resources for many use cases of multiple groups of clients. We call these groups, tenants. Multitenancy allows multiple tenants to share underlying resources, though the challenge is to do so in a way that there is performance isolation between the tenants’ workloads.

[Multitienancy](./tenants.jpg)

Multiple tenants were using their dedicated, special-purpose storage systems. When they will migrate to our system, they want the following:

- A sufficient amount of resources as per their needs.

- Performance similar to what before, via their custom storage systems.

On the other hand, we want our system resources to be shared such that:

- Sharing is high where resources are fully utilized (not stranded).

- Sharing is fair in the sense that each tenant gets what it wants but cannot overstep others.

- There is performance isolation between the tenants.

This lesson discusses the issues above and their solutions in detail. The high-level summary is as follows:

- Static vs. dynamic resource allocation: Tectonic system uses quotas for some resource types (for example, storage capacity), which are adjusted manually since needs change. Additionally, such resources are isolated from others, for example, on different storage nodes.

- Application priorities within a tenant: Since each tenant can have many applications, Tectonic enables a tenant to mark each application in one of the available traffic groups. It is similar to the nice values associated with processes in the Linux operating system. They convey relative priority to our system.
```
We use nice values in the Linux operating system to set the priority of the process.
```
- Across tenant priorities: Tectonic then takes all the traffic group markings from all the tenants and then sorts them into one of the three service buckets. (It is analogous to the multi-level feedback-based queues in the Linux OS, where each queue gets a different service from the OS.) A form of admission control is used where applications tell their needs, and the system ensures that the needs match the tenant/application profile.

- Rules enforcement: Tectonic then uses different policing mechanisms to ensure that every application gets its fair share and that there is performance isolation.

Let's now dive into the details.


## Challenges
It can be difficult to give tenants similar performance, since they transition from separate, specialized storage systems to a consolidated filesystem. This is because they must share resources while still offering each tenant a fair share or at least the same resources as they would have in a single-tenant system. The following are two major challenges for implementing multitenancy:

1. In resource sharing, the challenge is to decide how each tenant will share its resources with other tenants in the same cluster, keeping in mind that tenant x must not consume all of the resources of tenant y while the tenants y wait for tenant x to complete its task. Which of the resources can be shared and why?
2. In access control, the challenge is to decide which of the primitives we should have for performing read and write operations to avoid data corruption.

## Solution
The challenges, as mentioned earlier, have the following two solutions:

1. Efficient resource sharing: This is used to fairly distribute spare resources within a tenant cluster-wide without affecting the tenant’s performance.
2. Multitenant access control: This is used for deciding which client will have access to which data for read and write operations.
Let’s discuss them in detail.

### Efficient resource sharing
In a cluster, efficient resource sharing across the tenants is crucial. A weighted fair sharing (more on this algorithm later) of resources is required among tenants for the performance isolation between tenants. Tectonic also shifts resources smoothly among various applications to achieve high resource utilization. Distinguishing the requests with latency sensitivity is required to prevent these requests from being blocked by large requests. We have the following two types of resources:

1. Ephemeral resources: These can be shared among tenants with the same cluster, such as IOPS and metadata query capacity. These requests for frequent changes and allocation of resources in real-time provide isolation between tenants with high resource utilization in the same cluster.

2. Non-ephemeral resources: These are non-sharable resources that are once allocated to a tenant and will not be allocated to another, such as storage. Changes that occur in them are slow and predictable. There is no automatic elasticity in the tenants’ quotas, and disc space is statically divided among them. Operator intervention is necessary to modify a tenant’s quota, and tenants are in charge of distributing their own quota among their users. To avoid system downtime in an emergency, the quota can be modified by an operator while the system is still operational.

```
Note: We’ll discuss more about IOPS (ephemeral resource) rather than storage (non-ephemeral resources) in the upcoming sections.
```
#### Distributing ephemeral resources among and within tenants
Enabling sharing between ephemeral resources is not an easy task. We have to devise a technique so that we can efficiently share the ephemeral resources within and among other tenants. We can encounter the following two challenges:

1. Tenants’ diversity
2. Tenants need to serve different applications with different performance and traffic needs.
For example, blob storage includes both traffic of production and garbage collection from Facebook. Production traffic is more important than garbage collection traffic.

Even though we have two challenges, we have come up with a single solution for that, which is TrafficGroups. The ephemeral resource sharing among tenants is decided by application groups called TrafficGroups. Another way to explain TrafficGroups is that it is a collection of a tenant’s clients with the same latency and IOPS requirements. There are two major benefits of TrafficGroups:

1. It reduces resource-sharing complexity by grouping similar applications into one traffic group.
2. It reduces the overhead of managing multitenancy and also provides performance isolation.
There are 50 TrafficGroups per cluster (it is a configurable number). All the tenants do not have the same number of TrafficGroups because TrafficGroups depend on the group of applications each tenant serves. Each tenant chooses the number of TrafficGroups. Based on the sensitivity of latency and performance, TrafficGroups are allocated, and resources are shared accordingly.

For Tectonic, 50 TrafficGroups are way too many to manage. We need to further classify them into a small number of classes across tenants. We use three classes here. Each TrafficGroup is assigned one of the following TrafficClasses:

- The gold class is for high-latency requests.
- The silver class is for normal-latency requests.
- The bronze class is for background services.

[TrafficGroup and TrafficClasses within a cluster](./traffic.jpg)

The fair share of ephemeral resources is achieved using the following three steps:

1. Each tenant allocates the required ephemeral resources based on TrafficGroup and TrafficClasses.
2. All the spare ephemeral resources in each tenant are made available to be used by tenants cluster-wide, according to the TrafficClass.
3. All the unused ephemeral resources (allocated or unallocated) within a tenant are shared with the TrafficGroup within the tenant of a high TrafficClass.
This gives every tenant an equal or fair opportunity to utilize ephemeral resources within a cluster.

Any TrafficGroup that has completed their majority work has spare ephemeral resource. There are two ways to share these resources with lower TrafficClass’s TrafficGroup:

1. Within the tenant (high priority)
2. To another tenant
In this way, the same set of ephemeral resources that a single TrafficGroup once used can serve other TrafficGroups to meet their requirements.

Let’s learn more about how we manage to share ephemeral resources globally (at the cluster level and mostly across tenants) and locally (at the storage nodes level and mostly within a tenant).



##### Sharing resources globally
We got a general idea of how to achieve a fair share of ephemeral resources within and among tenants. To do this, we use the rate limiter, which implements the modified leaky bucket algorithm. The modified leaky bucket algorithm tracks the demand for each tracked resource in each tenant and TrafficGroup over the most recent brief period of time using high-performance, near-realtime distributed counters.

All this is done using the following steps:

1. Whenever a request comes, it requests for the incremented bucket counter.
2. Before entertaining the request, the client library has to look for spare capacity.
  - First, it will look for the spare capacity within the same TrafficGroup.
  - Secondly, it will look for the spare capacity in a different TrafficGroup within the same tenant.
  - Lastly, it will look for the spare capacity in the different tenants with the same TrafficClass priority.
3. If the spare capacity is found, the request will be sent to the backend. Otherwise, it can be either delayed or rejected based on its timeout.
This rate limiter avoids potentially wasted requests since failed requests put back pressure on the client.

#### Sharing resources locally
Global fair sharing of resources was not the only issue. We also have to avoid the local hotspots for storage and metadata nodes. The monitoring of the resources quota is done by using a weighted round-robin (WRR) scheduler for skipping specific TrafficGroup to avoid excessive use of a resource quota. In a cluster, the storage nodes must take precautionary measures to abstain from high-latency requests, such as data warehouse requests, and from using all the resources when the small IO request, such as blob storage requests, are in progress. If Gold TrafficClass requests on storage nodes are obstructed by lower-priority requests, Gold TrafficClass requests are allowed to have delays in their latency targets.

The following three optimizations are used by storage nodes to ensure low latency for the requests of the Gold TrafficClass:

1. In WRR, we have used greedy optimization allowing low-latency requests to give up their turn for the high TrafficClass if the request can be completed after the completion of the high TrafficClass. We do this so that they won’t be stuck behind a low-priority request.
2. We can apply a limit for every disk to n number of non-Gold requests to be active. WRR will block all the incoming non-Gold requests from scheduling unless any of the Gold request or active non-Gold request is completed. This allows the blob storage requests to be entertained simultaneously with warehouse requests.
3. We have given disks enough liberty to rearrange the IO requests in hand. For example, our system may serve an upcoming non-Gold request earlier than an existing Gold request. If the Gold request has been pending for a specific time period, we stop scheduling the non-Gold requests to a disk.
By combining these three techniques, we can effectively manage the latency profile of a relatively large number of blob storage requests compared to warehouse requests.

### Multitenant access control
Secure access to resources only by authorized clients is necessary. Each tenant is given specific access control, and they are not allowed to violate this access control, whether between different tenants or within a tenant. The Client Library interacts with each layer of the system directly, so we must enforce access control at every layer for every read and write operation. These access controls should be lightweight.

We have introduced token-based authentication in our system specifying the access of a resource through a token. For high-level client requests, such as opening a file, each layer of the system will generate a token for the next layer. The payload of the token contains the details of the resource for which the request has been granted by giving full access control. Once the token is granted, it is verified for the specified resource in the payload, and this verification is done in tens of microseconds. To minimize the access control overhead, we use piggybacking token-passing technique on our existing protocols.

```
Piggybacking: The process of delaying the acknowledgment until the next dataframe is available so that it can attach the acknowledgment of the received data along with the data to send.
```

In this lesson, we studied the challenges and their solutions to provide a secure and efficient storage-sharing substrate to multiple tenants. In the next lesson, we’ll understand how different tenants can get specific optimizations for their workloads.

