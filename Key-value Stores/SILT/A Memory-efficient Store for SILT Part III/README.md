# A Memory-efficient Store for SILT: Part III
## Bulk merge of intermediary stores with memory-efficient store
There needs to be a configurable number of intermediary stores accumulated. Once we have the required number then we can bulk merge the intermediary stores and the current instance of the memory-efficient store.

The intermediary store maintains values in hash order and maintains sequential read access. Merging these with our sorted memory-efficient store is efficient since all values are sorted. The merge process occurs in two steps listed below:

1. Sorting entries in intermediary stores

2. Sequentially merging sorted intermediary stores with the memory-efficient store

The first step in our bulk merge is sorting all intermediary stores that need merging. We do this by enumerating every entry in the intermediary stores and storing them in sorted key order. In the end we have all the entries from our intermediary stores sorted and stored together.

Next, we will choose the newest valid entries between our sorted intermediary store entries and our memory-efficient store entries. This is a sequential merge. It means that we will read sequentially from our merged and sorted intermediary store entries, and our memory-efficient store entries. We will have two pointers for this. At the start of our merge, one of these will point to the first entry of the memory-efficient store entries and the second one to the intermediary store entries in storage. At any point, we will refer to the key pointed to by the first pointer as Km (a key in the memory-efficient entries). We will refer to the key pointed to by the second pointer as Ki (a key in the intermediary store entries). We will construct a new entry log in storage based on the following steps:

1. If Km is lesser than Ki, we will copy the Km entry to our new log and move the pointer in the memory-efficient store entries by one entry. It points to another Km.

2. If Km is greater than Ki, and the Ki entry is not a DELETE request, we will copy the Ki entry to our new log and move the pointer in the intermediary store entries by one entry. It points to another Ki.

3. If Km is greater than Ki, and the Ki entry is a DELETE request, we will move the pointer in the intermediary store entries by one entry. It points to another Ki.

4. If Km equals Ki, and the Ki entry is not a DELETE request, we will copy the Ki entry to our new log and move the pointers in both memory-efficient and intermediary store entries by one entry. These point to another Km and Ki.

5. If Km equals Ki, and the Ki entry is a DELETE request, move the pointers in both memory-efficient and intermediary store entries by one entry. These point to another Km and Ki.

If the intermediary store pointer reaches the end of its log before the memory-efficient store pointer, we will copy the remaining entries in the memory-efficient store to the new log until its pointer reaches the end of the log. If the memory-efficient store pointer reaches the end of its log before the intermediary store pointer, then we will copy all remaining requests excluding DELETE requests to the new log until the pointer reaches the end of the log.
```
Note: A key's entry is a DELETE request if it contains the special DELETE character.
```
Here’s a visual representation of our bulk merge.

[Merge]

The new log represents our new memory-efficient store. It is ready for the construction of its compact prefix tree representation for the in-memory index. When this is complete, we can route new requests to this store and drop the intermediary stores used in its creation and the older version of the memory-efficient store.

### Lazy deletes for sequential write performance
If there were DELETE requests in our intermediary stores, then the total number of entries in the intermediary stores and the old memory-efficient store will be more than in the new memory-efficient store. This is because we do not include the DELETE requests in the memory-efficient store, and if the old memory-efficient store had an entry for the key with a DELETE request, we would also drop that entry.

We reclaim the space for DELETE requests when we drop intermediary stores and the old memory-efficient store (our new memory-efficient store has fewer entries in storage). These are the lazy deletes.

We have done this to ensure that our design always writes sequentially. We also needed to keep DELETE requests because we had to let our memory-efficient store know to drop keys with DELETE requests.


## What is next?
Our design is complete. Our lessons at this point have presented a complete design to meet our design goals. In the next lesson, we will discuss how each type of request (PUT, GET, and DELETE) is processed by our stores.
