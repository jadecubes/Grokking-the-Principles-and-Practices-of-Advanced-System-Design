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
## Group membership
## Locks
### Simple locks without the herd effect
### Read/write locks
## Double barrier
