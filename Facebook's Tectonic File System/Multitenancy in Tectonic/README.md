# Multitenancy in Tectonic
## Preamble
Remember that one of the primary goals of Tectonic is to efficiently utilize storage resources for many use cases of multiple groups of clients. We call these groups, tenants. Multitenancy allows multiple tenants to share underlying resources, though the challenge is to do so in a way that there is performance isolation between the tenants’ workloads.

[Multitienancy]
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

[TrafficGroup and TrafficClasses within a cluster]

##### Sharing resources globally
#### Sharing resources locally
### Multitenant access control
