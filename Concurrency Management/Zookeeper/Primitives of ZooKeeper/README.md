# Primitives of ZooKeeper
ZooKeeper helps clients to implement a set of primitives using the client API. ZooKeeper servers do not know about these powerful primitives because there isn’t any function for each of them. For the servers, these are just a set of functions from the client API sent by the client. Some of the common primitives are wait-free, which are group membership and configuration management. These primitives are either updated once or very often, and are mostly used for reading which makes it wait-free. Others, like rendezvous and locks, are based on waiting unless an event, such as a watch, are not triggered.

## Configuration management
ZooKeeper allows the client to implement dynamic configuration by using the following steps:

1. Create a znode Zc with 8090 as the port value.
```python
newZnode = create("/config/port", 8090, EPHEMERAL)
```
2. Each process is called with the full path of its Zc and gets its configuration from Zc, whose watch flag is set to true, using the command below.
```python
getData("/config/port", true)
```
The configurations are not updated too often, but when they are, using the command below, the process is notified to use the updated configuration that sets the watch flag to true.
```python
setData("/config/port", value, version)
```
Watches help ensure that the process which has set the watch flag to true gets updated information. Any process that has set the watch flag to true on the Zc will be notified so that it can use the updated value before it tries to read Zc. The process will be notified only once for multiple changes in the Zc before the process reads Zc. The aim is to notify the process that the information it has about Zc is stale and needs to get updated information from the server.
 

## Rendezvous
Due to dynamic configuration in distributed systems, we mostly have no idea about the system’s final configuration. Let’s take the master-worker architecture as an example where the client wishes to initiate multiple worker processes along with a master process. The client uses a scheduler to schedule these master and worker processes. The client itself doesn’t know when the master process will be initiated. If a couple of worker processes are initiated before the master, we cannot connect with the master, since we don’t have the details (the IP address and port) of the master yet. What should we do in such a scenario? ZooKeeper provides a solution to this problem. We call it the rendezvous znode, Zr.

Our solution consists of the following steps:

1. Create a new rendezvous znode Zr, normally a regular znode. Since Zr is an ephemeral znode, it will not be deallocated as long as the master and worker processes will be active in the memory.
```python
newZnode = create("/rendezvous/candidateOne", "", EPHEMERAL)
```
The client can put all worker processes that are started before the master process to sleep by putting a watch on the newly formed Zr.
```python
getData("/rendezvous/candidateOne", true)
```
When the master process will be initiated, the master process will store its details in the created Zr. After updating Zr, it will notify all the worker processes initiated before the master because the watch flag is set to true.
```python
setData("/rendezvous/candidateOne", {"10.0.0.1", 8080})
```
All the worker processes will get the details from Zr and start communicating with the master.
 
## Group membership
We have ensured that multiple clients can perform operations on the shared resources in Zookeeper. We have allowed clients to create groups so similar types of clients can share the workspace. We call this, group membership. For simplicity, we have used ephemeral znodes to implement group membership by using the following steps:

1. Create a parent znode Zg that represents a group, such as member0001, as shown below.
```python
newZnode = create("/GroupMembership/member0001", EHPEMERAL)
```
2. Each process belonging to that group will create a child node of its own under the created Zg.

    i. Each process stores its information (the IP and port to whom it is listening).

    ii. Each process should give the child node under Zg a unique name or can use the SEQUENTIAL flag to generate a unique name, such as processOne, as shown below.
```python
newZnode = create("/GroupMembership/member0001/processOne", EPHEMERAL,SEQUENTIAL)
```
We can get the information of the group by simply listing all the children of that Zg, using the getChildren(path, watch) method. We can set the watch flag to true to monitor any changes in the group.
```python
childrenList = getChildren("/GroupMembership/member0001", true)
```
## Locks
While Zookeeper does not provide explicit locking mechanisms, clients can use its API to implement a locking service. Many applications use ZooKeeper for synchronization according to their requirements. Keeping the aim (simple implementation) in mind, ZooKeeper allows its clients to implement locks simply.

The client process uses the following steps to implement the locks:

1. Create a znode, Zl, with the EPHEMERAL flag.
```python
newZnode = create("/locks/lock-1", EPHEMERAL)
```
2. If Zl is not created, the client process puts a watch on the locks.
```python
exists("/locks", true)
```
3. If Zl is created successfully, it means the lock has been acquired.

4. The client process releases the lock either when the client is disconnected, or Zl is deleted.
```python
delete(newZnode)
```
5. Once Zl is deleted, all the clients’ processes waiting for the lock to be released will be notified because of the Step 2 and they will now try to acquire it.
Since simplicity comes with some tradeoffs, our proposed solution has the following issues:

1. It causes a herd effect. As the name suggests, once the lock is released in Step 5, multiple clients want to acquire the lock.
2. It only implements exclusive locks.
```
Exclusive locks: These are locks that can be acquired only by one process at a time. For example, the lock for the write operation can be acquired only by one writer at a time.
```
Let’s discuss the solution to the issues mentioned above.

### Simple locks without the herd effect
We have made the following changes to improve locking:

1. When creating the znode, Zl, we have used the SEQUENTIAL flag along with EPHEMERAL, which will help us order each cleint’s attempt to acquire the lock, as shown in line 1 of the Lock procedure.

2. Once the clients are numbered by their attempts, the client with the lowest sequence number will hold the lock, as shown in line 4 of the Lock procedure. The clients who don’t hold the lock have to wait for the lock to be released.

3. Instead of notifying all the clients, it is better to inform the client who is next in line and check whether or not it is holding the lock, as shown in lines 6 to 8 of the Lock procedure.

```python
newZnode = create(path + "/lock-", EPHEMERAL, SEQUENTIAL)
do
    childrenList = getChildren(path, false)
    if newZnode is lowest znode in children 
        exit
    newPath = znode in childrenList ordered just before newZnode
    if exists(newPath, true) 
        wait for event 
while true

                     lock() without herd effect
```

```python
delete(newZnode)

                     unlock() without herd effect
```

When it comes to releasing the lock, it is as simple as deleting Zl as shown in line 1 of the Unlock procedure. Since we don’t want any locks to be acquired when the process is crashed, we have created Zl using the EPHEMERAL flag.
### Read/write locks
Once we have understood the locks without the herd effect, read/write locks are its successor with slight modifications. The write lock is the same as the lock we have learned in the previous section, but the lock name is changed from /lock- to /write-. We have to ensure that other writers won’t be able to write once a writer is writing, which we have done in line 4 of the WriteLock procedure. This lock doesn’t only look for writers but also checks if there aren’t any readers reading the same data. If any readers have acquired the lock, the writer has to wait for them to finish reading.

```python
newZnode = create(path + "/write-", EPHEMERAL, SEQUENTIAL)
do
    childrenList = getChildren(path, false)
    if newZnode is lowest znode in children 
        exit
    newPath = znode in childrenList ordered just before newZnode
    if exists(newPath, true) 
        wait for event 
while true

              writeLock() used by the writer processes
```

```python
newZnode = create(path + "/read-", EPHEMERAL, SEQUENTIAL)
childrenList = getChildren(path, false)
do
    if no write znodes lower than newZnode in childrenList
        exit
    newPath = write znode in childrenList ordered just before newZnode
    if exists(newPath, true)
        wait for event
while true

               readLock() used by the read processes
```
We have some slightly more modifications in read locks as much we have done in write locks. Since writers have high priority, we will check whether all the writers are done writing, as shown in line 4 of the ReadLock procedure. If there is any writer writing, the reader must wait, as shown in lines 6 to 8 of the ReadLock procedure. As you have noticed, we have just checked for the writers only in the read lock, and that is because multiple readers can read the same data at the same time.

## Double barrier
The synchronization of the computation of multiple processes from its start to end is done by using the double barrier in ZooKeeper. A threshold can be defined for the barrier, and all the processes in that barrier have to wait for the computation to start until the total number of processes in the barrier is not the same as the threshold. There are two barriers, one before entering and the second before leaving. That is why it is known as the double barrier. The barrier starts the computations using the following steps:

1. Create a new znode, Zb, with the process name as shown in line 1 of the Enter procedure.
2. Set the watch to check whether the barrier is ready for computation, as shown in line 2 of the Enter procedure.
3. Create a child znode under the created Zb as shown in line 3 of the Enter procedure.
4. Wait until the number of processes is not equal to the threshold of the barrier by checking the number of children Zb has, as shown in lines 4 to 6 of the Enter procedure.
5. When the number of processes is equal to the threshold, a new znode is created with the name, ready, appended to it, as shown in line 8 of the Enter procedure.

```python
newZnode = create(path + "/" + processName)
exists(path + "/ready", true)
newZnodeChild = create(newZnode, EPHEMERAL)
childrenList = getChildren(path, false)
if fewer children in childrenList than barrierThreshold
  wait for watch event
else 
  create(path + "/ready", REGULAR)
        
        enter() to enter in the barrier
```

```python
do
  childrenList = getChildren(path, false)
  if no children
    exit
  if processOne is only process node in childrenList
    delete(newZnode) 
    exit
  if process is the lowest process node in childrenList
    wait on highest process node in childrenList
  else 
    delete(newZnodeChild) # if still exists and wait on lowest process node in childrenList
while true

                         leave() to leave in the barrier
```

Leaving is easy, and for that, we’ll perform the following steps:

1. We’ll check if the process that is leaving is the last process. If yes, we will delete Zb as shown in lines 5 to 7 of the Leave procedure.
2. If the process that wishes to leave is the process that has other processes as siblings under the path, it will have to wait for the other processes to be ready to leave, as shown in lines 8 and 9 of the Leave procedure.

[Double barrier](./barrier)
The behavior of the double barrier, as shown in the illustration above, shows that we have achieved linearizability.

[The namespace of primitives](./namespace.png)

In this lesson, we learned that, for each primitive, we have a znode created under it which can be used to solve our specific issues, as shown in the illustration above. In the next lesson, we’ll evaluate ZooKeeper and see how it solves the problems we mentioned in the introduction of ZooKeeper.
