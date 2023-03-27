# Introduction to Key-value Stores
## Motivation
A key-value store is one of the fundamental building blocks for many distributed services. For some use cases, a key-value store is used as a caching layer between the front-end services and the back-end persistent stores to speed up reads and writes. For other use cases, a key-value store provides a foundation to build NoSQL databases as they are also a type of non-relational data store (Google’s Bigtable is one example).

## What we will learn
We’ve selected the following four papers on key-value systems to discuss in the following chapters:

- [MC] M. Berezecki, E. Frachtenberg, M. Paleczny, and K. Steele. 2011. Many-core key-value store. In Proceedings of the 2011 International Green Computing Conference and Workshops (IGCC '11). IEEE Computer Society, USA, 1–8.

- [SILT] Hyeontaek Lim, Bin Fan, David G. Andersen, and Michael Kaminsky. 2011. SILT: a memory-efficient, high-performance key-value store. In Proceedings of the Twenty-Third ACM Symposium on Operating Systems Principles (SOSP '11). Association for Computing Machinery, New York, NY, USA, 1–13.

- [Memcache] Rajesh Nishtala, Hans Fugal, Steven Grimm, Marc Kwiatkowski, Herman Lee, Harry C. Li, Ryan McElroy, Mike Paleczny, Daniel Peek, Paul Saab, David Stafford, Tony Tung, and Venkateshwaran Venkataramani. 2013. Scaling Memcache at Facebook. In Proceedings of the 10th USENIX conference on Networked Systems Design and Implementation (nsdi’13). USENIX Association, USA, 385–398.

- [DynamoDB] Perianayagam, Somasundaram, Akshat Vig, Doug Terry, Swami Sivasubramanian, James Christopher Sorenson III, Akhilesh Mritunjai, Joseph Idziorek et al. Amazon {DynamoDB}: A Scalable, Predictably Performant, and Fully Managed {NoSQL} Database Service. In 2022 USENIX Annual Technical Conference (USENIX ATC 22), pp. 1037-1048. 2022.


## Why did we choose these systems?
The original Dynamo key-value system by Amazon was a seminal work that taught us how to build an industrial-grade key-value store for large-scale commercial use. Dynamo took many novelties from the peer-to-peer research era (such as complete decentralization without any central entity) and avoided many of the pitfalls of peer-to-peer systems, such as security concerns arising from the lack of a central entity.
```
Note: Dynamo was built and operated by one organization. Therefore, the operational environment was assumed to be trusted and non-malicious.
```

```
DeCandia, Giuseppe, Deniz Hastorun, Madan Jampani, Gunavardhan Kakulapati, Avinash Lakshman, Alex Pilchin, Swaminathan Sivasubramanian, Peter Vosshall, and Werner Vogels. Dynamo: Amazon’s highly available key-value store. ACM SIGOPS operating systems review 41, no. 6 (2007): 205-220.
```
#### The era of multiple cores
2004 onwards was the dawn of the multicore era. The established norm for decades, where the processor will get faster every two years without needing any major restructuring for the software, abruptly ended. Now, to utilize multiple processing cores, the software is required to explicitly make use of such speed-up opportunities.

Our first system (MC) shows an effort where Memcached was restructured to use multiple cores. At the same time, we stress the need not just to measure performance in isolation but rather as performance per watt or dollar to capture the critical aspect of the cost (for modern data centers, one of the high operational costs is electricity use).


#### The rise of flash drives
Our next key-value store (SILT) focuses on individual servers (in a cluster of Dynamo-like key-value stores) to efficiently use memory to pack as many keys as possible with lower metadata overhead. Instead of slow I/O, it utilizes flash drives judiciously as they have limited write cycles that must be used carefully to extract a good value. We learn how we can use multiple key-values stores collectively to optimize for both memory and flash storage.

#### Zooming out: Across-datacenter key-value stores
Our next system (Memcache) shows how to combine many local instances of a key-value store to build a globally consistent, highly scalable key-value store. In some respects, this system resembles the Dynamo system.


#### A generalized managed key-value store
Our final system (DynamoDB) shows how to build a key-value store that performs well for various applications. We see how to design a system with low latency for all calls.

We hope our selection of key-value systems teaches us many important lessons in system design. Let’s dive in!

[Timeline of the evolution of key-value stores](./timeline.png)
