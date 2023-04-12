# Intermediary Store(s) in SILT

A write-friendly store will not add more entries beyond a certain point when the memory bound is reached or the store cannot find an empty bucket in the allocated number of displacements. Now, we will convert this store into a more memory-efficient immutable store. We will call this our intermediary store.

## Why we need an extra store
Keeping data around in our completed write-friendly stores incurs high memory overhead due to the indices. On the other hand, if we merge a completed write-friendly log to the final (our most memory-efficient store, which uses sorting for compaction of keys), it might require an excessive movement of keys for merging and re-sorting. The intermediate store helps us find a reasonable middle ground between the above two bad options (high memory overhead if we keep a write-friendly log and possible movement of many keys on each log merging). In the following lesson, we will explain this situation in detail.

We will see later that our memory-efficient store will also be our largest store and keep our key-value pairs sorted. Due to the memory-efficient store's compact representation (one of the reasons why it is memory-efficient), it will be immutable (unchangeable). Frequently sorting our small write-friendly store's entries into our memory-efficient store will require a lot of rewriting and results in a higher write amplification.

Keeping many write-friendly stores before merging their entries will reduce write amplification. However, this increases memory use because of the write-friendly store's in-memory hash table.

Our multi-store approach allows us to solve this problem by introducing a new store between our write-friendly and memory-efficient stores. We will convert our write-friendly store to an intermediary store as soon as it stops accepting new entries. During the conversion, a newly initialized write-friendly store will serve PUT and DELETE requests.

This intermediary store will be immutable. As a result, we will have one intermediary store for every instance of a write-friendly store. This store will consume less memory than the write-friendly store for the same number of entries, ensuring lower memory use for accumulating entries before merging with the memory-efficient store, which is next in line.


## Storing entries in hash order for reduced memory consumption
Transforming a write-friendly store to an intermediary store requires reordering the key-value entries in storage. Previously, in the write-friendly store, entries were stored in the order we received them (insertion order). Our conversion algorithm will rearrange the entries in the new intermediate store as per the order of their hash values—the order of the keys' hashes in the write-friendly hash table—and alleviate the need to store their offsets in memory.

Here is the conversion from the write-friendly store to the intermediary store in storage.

[The storage log before (above) and after (below) conversion from a write-friendly store to an intermediary store. Both data structures are the same and store key-value pairs sequentially in storage.](./reduced.png)

There are two things to notice here:

- The intermediary store only has the latest values for keys.

- The order of entries has changed in the intermediary store.

Now, we can locate entries in storage using only the first column of the in-memory hash table since the entries on storage are now in the same order as the in-memory hash table. We do not require the offset column since the position of the hash in the table can act as the offset itself. Furthermore, we do not require empty buckets, that is, buckets with no entries stored. Previously, we had dedicated this space because the insertion algorithm of the write-friendly store might have to use this. Since we will not insert any new entries to this store, we can drop these.

In memory, we drop the empty buckets and the offset column.

[The memory hash table before (left) and after (right) conversion from a write-friendly store to an intermediary store. Both data structures have the same hash functions, which we can see from the identical mapping of keys to buckets.](./trans.png)

Notice how a key's hash's position represents its offset in storage. To clarify, h1(K1) is less than h2(K2), which is lesser than h2(K9), and so on.
```
Note: We can remove the offsets of keys (on the disk) in our intermediate-level store because each record on the log is of equal size in bytes.
```
## Memory-efficient creation process

The in-memory index for an intermediary is created by simply copying from its write-friendly store instance. We cloned the write-friendly store's memory index to convert it to the intermediary store's index since we need the original index to serve GET requests until the conversion to the intermediary store is complete. Here's our creation algorithm.

We will traverse the hash table from the start to the end. For every entry in the hash table:

1. Check if the bucket is empty

    I. If the bucket is empty, skip this bucket and make no changes in storage.

    II. If the bucket is not empty:

        i. Locate the entry in the write-friendly store's storage log using the offset.

        ii. Append the entry found in the write-friendly store's storage log to the intermediary store's log.

2. When the above process has been completed for the entire hash table:

    I. Drop the offset column in the hash table.

    II. Drop the rows with empty buckets.

    III. Drop the entire write-friendly store's storage log and hash table.

The slide deck below demonstrates the conversion from a write-friendly store to an intermediary one.

[Process](./memefficient)

```
Note: We lazily update or delete keys' values. Remember that in our write-friendly store, we sequentially append all operations, even if an already present key's value was updated or deleted. We didn't try to update (or remove keys) right away because it would have required random reading and writing (a slow operation), so we traded disk space for writing speed.

When we transformed a completed write-friendly log store to an intermediate representation, we only appended the latest value (or operation in case of deletion). We keep the DELETE operation in our intermediate log so that the next store in line can remove the keys. This way, we lazily removed redundant value changes of the same key. However, we will need random reading during this conversion since this operation is offline (not on the end-client's critical path). Hence, latency that has been added due to random reads from the write-friendly log will not impact the client.
```

## In-memory filter for reduced read amplification
Similar to our write-friendly store, this intermediary uses the in-memory tags copied from its corresponding write-friendly store instance to filter out requests for keys that are not stored.


### GET requests
We will process GET requests for the intermediary store in the same manner as we do for the write-friendly store. The only difference is that whenever a tag matches the computed hash, we will not check for the entry on the offset position but instead the position of the hash in the hash table. We will only proceed with the lookup for a GET request if its key's computed tag matches a tag in the in-memory filter. So, when the intermediary store receives a GET request for key Kg:

- It computes the candidate buckets h1(Kg) and h2(Kg) and looks inside these buckets.

    - Inside h1(Kg), it looks for the tag h2(Kg). If the computed tag h2(Kg) matches the tag inside bucket h1(Kg), then it looks up the entry in the storage log stored on the position where h2(Kg) is stored in the hash table.

    - Inside h2(Kg), it looks for the tag h1(Kg). If the computed tag h1(Kg) matches the tag inside bucket h2(Kg), then it looks up the entry in the storage log stored on the position where h1(Kg) is stored in the hash table.

- Upon a successful match in either of the above two cases, it looks up the key-value pair (in the storage log) at the position where the computed tag is stored in the hash table: (Ks, Vs). It then matches the key Ks from the key-value pair returned from storage with Kg to confirm that we have looked up the correct key.

```
Successful match: computed tag matches tag in bucket
```

    - If the keys match (Kg equals to Ks), we check the value Vs in the key-value pair returned from storage.

        - If Vs is not the special DELETE indicator (Vs is not equal to DELETE), the intermediary store signals that the GET request should terminate and return that Vs is the value stored against Kg.

        - If Vs is the special DELETE indicator (Vs equals to DELETE), the intermediary store signals that the GET request should terminate and return that Kg is not stored in the key-value store (the entire key-value store and not just this intermediary store).

    - If the keys do not match (Kg is not equal to Ks), the intermediary store signals that Kg is not present inside this intermediary store, and the GET request should continue in the next intermediary store.

- Upon an unsuccessful match in both candidate buckets, the intermediary store signals that Kg is not present inside this intermediary store, and the GET request should continue in the next intermediary store. Like we saw with the write-friendly store, this is the case where our hash table acts as an in-memory filter. We have returned that Kg is not stored inside this intermediary store without checking the storage log.

```
Unsuccessful match: computed tag does not match tag in bucket
```

- Suppose this intermediary store is the last intermediary store, and the GET request needs to continue in the next intermediary store. In such a case, the GET request will continue in the memory-efficient store.
```
Note: The GET requests do not happen in parallel. Instead, we check intermediary stores from newest to oldest. While an argument for parallelization can be made, it is important to understand that the newer intermediary store will always have the latest value for a key compared with older intermediary stores.
```

## Bulk merge to memory-efficient store for efficiency
We've already mentioned that keeping many stores before merging them to one big sorted memory-efficient store will provide less write amplification than a frequent merging of only one store. We will accumulate a configurable number of intermediary stores before bulk merging into our memory-efficient store. All intermediary stores will continue to serve GET requests until the merge is complete.

If we had decided to accumulate write-friendly stores instead, that would have increased our memory use significantly since the write-friendly store uses more memory per key. Furthermore, our intermediary store keeps entries in hash order, allowing decent read speed. Accumulating and merging a bulk of our intermediary store is much more efficient than doing the same for our write-friendly store.

Next, we will explain the benefits we get from the intermediary store when we do a bulk merge to the memory-efficient store.
