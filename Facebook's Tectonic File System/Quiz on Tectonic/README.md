# Quiz on Tectonic
```
Question 1
What is one major benefit of multitenancy in a file system, and what is one challenge that arises from it?
```

```
Question 2
How can Reed-Solomon-based data codes be beneficial compared to full data replication?
```

```
Question 3
Why are some of the blob fully replicated and some RS-encoded?
```

```
Question 4
How much space can be saved by using RS(15,9) codes compared to 3-way full replication?
```

```
Question 5
Why is the single-writer semantic per file adopted by Tectonic? Why did they not allow multiple writers per file?
```

Answers
```
Question 1
What is one major benefit of multitenancy in a file system, and what is one challenge that arises from it?

Answer
Multi-tenancy enables us to increase resource utilization across disparate storage customers. However, such sharing needs to be done carefully such that:

- The data of different tenants is isolated from each other and appropriate data access controls are in place.

- There should be performance isolation between tenants meaning one misbehaving tenant should not impact others.

- The applications are able to pick and choose different optimizations they need for their use cases so that our shared system’s performance approaches a dedicated system’s performance.
```

```
Question 2
How can Reed-Solomon-based data codes be beneficial compared to full data replication?

Answer
Typically, data is three-way replicated, while Reed-Solomon (RS) codes generally use lesser storage (it varies based on the specific parameters used for Reed-Solomon codes). The catch is that RS codes consume CPU for encoding (on writing) and decoding (or reading data back and any reconstruction to recover from errors). Storage savings are substantial for an exabyte-scale file system.
```


```
Question 3
Why are some of the blob fully replicated and some RS-encoded?

Answer
The hot blobs (most demanding by the clients) need to be replicated so that clients can easily access them, and encoding such data will affect the latency (for encoding on writes and decoding on each read). In contrast, warm blobs (not very demanding by clients) are RS-encoded to save storage space.

There is a trade-off between latency and storage space when we decide about RS-encoding or full replication.
```

```
Question 4
How much space can be saved by using RS(15,9) codes compared to 3-way full replication?

Answer
RS(15,9) takes 9 units of data, adds parity to them, and makes them 15 units. This means 9 units of data bloats to 15 units after encoding. On the other hand, for full 3-way replication, 9 units of data will become 27 units.

Comparing 15 to 27 units, RS(15,9) provides about 50% storage reduction as compared to full 3-way replication.
```

```
Question 5
Why is the single-writer semantic per file adopted by Tectonic? Why did they not allow multiple writers per file?

Answer
Allowing multiple writers per file will cause the race condition and handling that will become complicated. Since the main goal was to keep things simple, they decided to go with the single-writer semantic.

Some other file systems, such as the Google File System, allow multiple concurrent writers in the same file. However, allowing multiple writers makes the data consistency model more challenging to work with.

Can Tectonic satisfy those use cases that GFS’s multiple concurrent writers were able to do, such as using a file as a shared queue between multiple concurrent producers and consumers? Thinking about this might provide you with more insight into Tectonic.
```
