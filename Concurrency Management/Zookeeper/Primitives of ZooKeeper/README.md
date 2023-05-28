# Primitives of ZooKeeper
ZooKeeper helps clients to implement a set of primitives using the client API. ZooKeeper servers do not know about these powerful primitives because there isn’t any function for each of them. For the servers, these are just a set of functions from the client API sent by the client. Some of the common primitives are wait-free, which are group membership and configuration management. These primitives are either updated once or very often, and are mostly used for reading which makes it wait-free. Others, like rendezvous and locks, are based on waiting unless an event, such as a watch, are not triggered.

## Configuration management
ZooKeeper allows the client to implement dynamic configuration by using the following steps:

1. Create a znode Zc with 8090 as the port value.
```
newZnode = create("/config/port", 8090, EPHEMERAL)
```
2. Each process is called with the full path of its Zc and gets its configuration from Zc, whose watch flag is set to true, using the command below.
```
getData("/config/port", true)
```
The configurations are not updated too often, but when they are, using the command below, the process is notified to use the updated configuration that sets the watch flag to true.
```
setData("/config/port", value, version)
```
Watches help ensure that the process which has set the watch flag to true gets updated information. Any process that has set the watch flag to true on the Zc will be notified so that it can use the updated value before it tries to read Zc. The process will be notified only once for multiple changes in the Zc before the process reads Zc. The aim is to notify the process that the information it has about Zc is stale and needs to get updated information from the server.
 

## Rendezvous
Due to dynamic configuration in distributed systems, we mostly have no idea about the system’s final configuration. Let’s take the master-worker architecture as an example where the client wishes to initiate multiple worker processes along with a master process. The client uses a scheduler to schedule these master and worker processes. The client itself doesn’t know when the master process will be initiated. If a couple of worker processes are initiated before the master, we cannot connect with the master, since we don’t have the details (the IP address and port) of the master yet. What should we do in such a scenario? ZooKeeper provides a solution to this problem. We call it the rendezvous znode, Zr.

Our solution consists of the following steps:

1. Create a new rendezvous znode Zr, normally a regular znode. Since Zr is an ephemeral znode, it will not be deallocated as long as the master and worker processes will be active in the memory.
```
newZnode = create("/rendezvous/candidateOne", "", EPHEMERAL)
```
The client can put all worker processes that are started before the master process to sleep by putting a watch on the newly formed Zr.
```
getData("/rendezvous/candidateOne", true)
```
When the master process will be initiated, the master process will store its details in the created Zr. After updating Zr, it will notify all the worker processes initiated before the master because the watch flag is set to true.
```
setData("/rendezvous/candidateOne", {"10.0.0.1", 8080})
```
All the worker processes will get the details from Zr and start communicating with the master.
 
## Group membership
We have ensured that multiple clients can perform operations on the shared resources in Zookeeper. We have allowed clients to create groups so similar types of clients can share the workspace. We call this, group membership. For simplicity, we have used ephemeral znodes to implement group membership by using the following steps:

1. Create a parent znode Zg that represents a group, such as member0001, as shown below.
```
newZnode = create("/GroupMembership/member0001", EHPEMERAL)
```
2. Each process belonging to that group will create a child node of its own under the created Zg.

    i. Each process stores its information (the IP and port to whom it is listening).

    ii. Each process should give the child node under Zg a unique name or can use the SEQUENTIAL flag to generate a unique name, such as processOne, as shown below.
```
newZnode = create("/GroupMembership/member0001/processOne", EPHEMERAL,SEQUENTIAL)
```
We can get the information of the group by simply listing all the children of that Zg, using the getChildren(path, watch) method. We can set the watch flag to true to monitor any changes in the group.
```
childrenList = getChildren("/GroupMembership/member0001", true)
```
## Locks
While Zookeeper does not provide explicit locking mechanisms, clients can use its API to implement a locking service. Many applications use ZooKeeper for synchronization according to their requirements. Keeping the aim (simple implementation) in mind, ZooKeeper allows its clients to implement locks simply.

The client process uses the following steps to implement the locks:

1. Create a znode, Zl, with the EPHEMERAL flag.
```
newZnode = create("/locks/lock-1", EPHEMERAL)
```
2. If Zl is not created, the client process puts a watch on the locks.
```
exists("/locks", true)
```
3. If Zl is created successfully, it means the lock has been acquired.

4. The client process releases the lock either when the client is disconnected, or Zl is deleted.
```
delete(newZnode)
```
Once Zl is deleted, all the clients’ processes waiting for the lock to be released will be notified because of the Step 2 and they will now try to acquire it.
Since simplicity comes with some tradeoffs, our proposed solution has the following issues:

It causes a herd effect. As the name suggests, once the lock is released in Step 5, multiple clients want to acquire the lock.
It only implements exclusive locks.
Let’s discuss the solution to the issues mentioned above.


### Simple locks without the herd effect
### Read/write locks
## Double barrier
