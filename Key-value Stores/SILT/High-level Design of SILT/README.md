# High-level Design of SILT
## Design approach
There are a number of different approaches to making key-value stores, each of which can result in a different system.

For example, some are solely in the RAM and not backed by persistent storage. While these can provide top-notch performance, they are costly and primarily support fast reads only. The key-value cache will need to be rebuilt on any server failure from the source store.

On the other hand, for some key-value stores, everything is stored in a persistent disk; this is economical but low in performance due to inherently slow IO operations. However, there have been suggestions to improve performance in such stores. Ideally, a key-value store should use memory and storage to provide acceptable dollar cost and performance. However, flexibility in using these two major resources depends on our approach to designing a key-value store.

Our experience with key-value store designs shows that it is challenging to get the best of both worlds (economic and fast reads and writes) using a single key-value store. In our design, we will use a neat idea where we collectively use many kinds of stores, each optimized for a specific goal, and providing a good trade-off point. This approach is called a multi-store approach. In the following section, we will elaborate on why single-store approaches limit us from achieving our goals and then show how multi-store design copes with these limits.


## Single-store approach
The single-store approach uses only one key-value store with features to improve performance. Generally, a single-store key-value pair has three main components:

A filter, stored in memory, reduces disk reads by checking if a key is stored or not. Read requests that pass the filter result in disk reads.

An in-memory index to provide the address of a key in storage.

A storage data layout that contains all of our key-value entries.

A single-store approach makes it hard to simultaneously achieve all our design goals (high-performance with efficient use of resources). Let's look at some single-store key-value stores.

```
This filters requests and only allows those requests to pass through for which keys are stored in the key-value store
```

### HashCache
HashCache is a single-store key-value store. We should consider the following two features of HashCache:

It has no in-memory index. Instead, it has a hash table in storage. This reduces the use of expensive memory.

HashCache keeps track of its cached objects via a special in-memory table. It shifts the workload of confirming cache misses to memory and is optimized to use low memory.
```
Note: The in-memory table differs from a standard in-memory index. This table is only for cache, used for storing frequently accessed items. The size of this table does not increase proportionally with the number of entries stored as it would with the in-memory index.
```
HashCache is efficient with memory since it only uses memory for cache and stores all its keys in a hash table in storage. However, it is hard to scale.

HashCache stores all its entries in an in-storage hash table. Scaling it would require a large hash table. A large hash table will take up a significant portion of storage. This will increase the chances of keys in consequent requests pointing to values far from each other on storage. For these requests, the storage hardware will first find the address of the value for the first key, then the second key. Such requests are called random requests. Random reads require searching for the address of the value to return. Similarly, random writes require searching for the address to write the value. We need to note that for storage, regardless of the hardware used (disk, SSD, or flash), sequential reading and writing are faster than random reading and writing.

### Log-based key-value stores
We can solve the problem above by ensuring our system writes sequentially to storage—appending new entries to a log. Some single-store key-value stores that apply this technique are FAWN-DS, FlashStore, and SkimpyStash.

FAWN-DS uses an in-memory hash table based on a partial key to locate the key on the in-storage log. It appends a new entry to the log and stores its offset on the hash table. Both FlashStore and SkimpyStash maintain a buffer in memory, write to storage after the buffer is full, and update their in-memory hashtable, which resolves to a location in storage. Such buffered writing is efficient in terms of throughput, latency, and using write-cycles of flash disks.

Another example of a log-based KV store is the Microsoft FASTER.
```
partial key: a part of the key
```

## The problem with a single-store approach
Writing sequentially to storage and keeping addresses in memory solves the problem of degraded performance with random requests. However, such a system is hard to scale. As the size of the key-value store increases, so does its per-key memory consumption because of the in-memory hash table.

While a single-store approach keeps things simple, it limits our ability to achieve a better design.

[A single-store approach generally produces a trade-off between high performance and memory efficiency]

Here’s how this trade-off exists. Let’s start with HashCache–a simple design with no in-memory index. HashCache does not work well with random writes since its main hash table is in storage. Here, we are compromising on a higher performance for memory efficiency. For our system to be fast, it needs to write sequentially to storage—since writing sequentially to storage is faster. Using a log-based approach—a sequential write to storage and keeping the offset in memory with hashing—improves our performance. However, as the number of keys grows, so does memory consumption and our cost. This trade-off makes it hard to achieve all our design goals simultaneously.

## The big idea: A multi-store approach
Imagine if we could have specialized key-value stores, all geared for a particular purpose. We could write new entries to a store that writes sequentially to storage and eventually move our entries to another store that uses memory sparingly. We can do this using the multi-store approach.

While designing a multi-store, consider the following:

1. All individual stores must be efficient on their own, especially for their purpose.

2. Stores must work together. For example, if store A collects new entries and eventually moves them to store B, this process should be efficient.

3. The overall design must query all stores for lookups in an efficient manner.


## Our model
We chose multi-store over single-store for its efficiency. Now, let's build a high-level understanding of our multi-store.

### API
Our system will support three operations: PUT, DELETE, and GET.

1. The PUT operation inserts a new key-value entry or updates an already existing entry.

2. The DELETE operation removes a key-value entry.

3. The GET operation fetches the latest value for a key in our stores.

### Design overview
At this point, we have a fair idea of what we want from our key-value store. Now we can talk about our design in a little more detail. The characteristics below declare how we will design our store.

1. We will insert keys into a write-optimized store. This store will be responsible for all PUT and DELETE operations. Over time, we will move new and updated entries to more memory-efficient stores.

2. We will use the multi-store approach to ensure that transfer of new entries to other stores is efficient.

3. We will store the majority of key-value pairs in the most memory-efficient store.

4. We will optimize our system for worst-case performance–lookups in the last and largest store.

#### Insert keys into a write-optimized store
The point of using a multi-store approach is to have specialized stores for each purpose. Our first store will efficiently collect new and updated entries and pass them onto the system for storing in memory-efficient stores over time. The flow of DELETE requests will be the same.

There is one important thing to consider about DELETE requests in this design. Deleting a key from all our stores as soon as we encounter a DELETE request is not the best way because it requires a lot of random reading. We might have to DELETE a single key from multiple stores. So instead, we will treat DELETE requests like PUT requests. The only difference will be that DELETE requests will store a special DELETE indicator instead of a value. If a GET request reads a key-value entry with DELETE stored in it as the value, it will return that the key does not exist in the store.

#### Efficient transition from write-optimized to memory-efficient store
It may not be easy to transfer entries directly from our first store to our most memory-efficient store. Our write-friendly store keeps entries in insertion order (the order in which they arrive). By contrast, our memory-efficient store will keep key-value pairs sorted—we need this to enable fast reads since this will store the majority of the key values (described below) and receive the bulk of GET requests. It is also important to note that our memory-efficient store will be immutable since it will store keys with a complicated encoding that requires computation work. It will also store keys compactly.

For example, if keys 003 and 005 exist in the memory-efficient store, and key 004 does not exist, it will store keys 003 and 005 one after the other without any space for key 004. We need to make two considerations here. The first is that frequently merging the small write-friendly store (containing approximately 5% of the entries to keep per-key memory consumption low) is not a good idea since it will increase per-key computational work and write amplification. The second is that the task of merging can be made efficient if the entries from the write-friendly store are sorted.

Our multi-store approach allows us to design stores between our write-friendly and memory-efficiency stores. These intermediary stores will efficiently receive entries from our first store, are relatively memory-efficient, and pass entries to the next store in the sequence. We can add any number of stores between our write-friendly and memory-efficient stores. We will use these intermediate stores to alleviate the problems mentioned in the previous paragraph. We will lazily convert a write-friendly store to a sorted intermediary store to not interrupt fast writing. We will also accumulate a decent number of keys into these intermediary stores before merging to reduce the per-key computational cost of constructing the memory-efficient store. The distribution of entries stored in each store should result in optimal performance.

#### Store most entries in the memory-efficient store
We have to attain low costs by minimizing memory use. While the other store(s) in our system might use more memory, the overall per-key memory cost will remain low since we will store most keys in this memory-efficient store. The GET requests will look for keys in the following sequence: write-friendly store, intermediary stores (latest to oldest), memory efficient store.


#### Optimize system for lookups in the largest (memory-efficient) store
The general flow of new entries will be from write-friendly to memory-efficient stores. A GET request will read stores in the same sequence a new entry travels to reach our memory-efficient store eventually. In this situation, a worst case would be that a key is not found in stores before the search reaches the last store in the sequence, our most memory-efficient store. Since we aim to store the majority of our entries in the most memory-efficient store, we should design our system for high performance in this case because most of the searches will reach the worst case.
```
Note: The worst case does not mean anything negative like slow reads or degraded performance. Here, worst case means that the search did not find the key in the stores that are not memory-efficient. The probability of that happening is already very high since we will store the largest proportion of keys in the memory-efficient store—we will design our system to behave well in the worst case. And therefore, we need to optimize its performance in the worst case.
```

[Multiple individual stores work collectively to achieve our goals of memory efficiency and read/write performance]

The illustration above shows our model. A few key things to notice are:

1. As the memory efficiency of our stores increases, so does the proportion of the entries they store.

2. Both PUT and DELETE requests are managed by the first store. We haven’t yet explained how we will update the entries relevant to these requests. For now, it is important to note that the system handles these in the first store. Later, we will see that the effects of these requests are lazy. Values are updated or deleted at a later time.

3. The second point does not mean that the design will return old values since our GET requests are processed such that,

  I. The system searches for the key in the store that receives DELETE and PUT requests first (write-friendly store), intermediary stores (newest to oldest), and the memory-efficient store.

  II. If the store currently being searched has the key at any point, the request will end, returning the found value. This way, our system ensures that it returns the latest value.

In this lesson, we gained a general understanding of our design. The following lessons will explore the design in detail.
