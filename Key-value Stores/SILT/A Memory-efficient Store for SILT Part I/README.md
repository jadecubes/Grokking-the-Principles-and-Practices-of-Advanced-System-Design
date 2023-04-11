# A Memory-efficient Store for SILT: Part I

## Preamble
Memory efficiency is one of the core design goals for our key-value store. In previous lessons, we’ve enabled clients to write keys quickly (using our write-friendly store) by keeping the per-key memory consumption of our key-value store low. Our previous efforts were on stores we used to achieve other goals—fast sequential writing in the write-friendly store and accumulating intermediary stores for bulk merge to a memory-efficient store. In this lesson, we will make our most significant contribution to keeping memory-consumption low in the last-level store. Our goal will be to store the largest proportion of entries (80%) in our most memory-efficient store to ensure our design uses, on average, low memory per key.

We need our last-level store to be a memory-efficient store. Here’s a list of requirements for our last-level store:

1. Read-friendliness: This is essential because the last-level store will hold a large proportion of keys in our design. Chances are that most of the lookups will happen in this store.

  - O(log n) time lookups: Our representation will allow us to achieve logarithmic time complexity in the average case.

2. Low memory consumption per key: This will help us store most entries in our store and will keep our design's overall per-key memory consumption low.

  - Compact representation: Our physical representation of keys in memory will be crucial in keeping per-key memory consumption low.

3. Easy creation: Over time, new data (from the intermediary store) will arrive in last-level store for incorporation. Such data incorporation should be efficient because this operation will be fairly frequent. We will soon see that having the intermediary store sorted has already helped us towards efficient incorporation.

  - Immutability: We do not require this store to change in real time—we have the write-friendly store for this—hence, we will give up the need to do so to make this store's creation easier.

## Immutability and sorting data in storage
Our memory-efficient store will sort keys by key order in storage. For example, for 3-bit keys, 000 will be stored first, then 001, 010, and so on. This will help us achieve the following:

- Efficient bulk merging of hash-sorted intermediary stores into the key-sorted memory-efficient store

- Compact representation of keys in memory

A key will only be stored if it exists in the store. In the example above, if key 001 does not exist in the store, then 000 will be followed by 010. We do not need to keep space for key 001 since this store will not change in real time. Doing so, we are able to achieve the following:

- Our in-memory index for this store will have to keep track of fewer keys, resulting in lower per-key memory consumption.

- It allows us to keep our memory-efficient store immutable. Immutability is more of a correlation than a causality–one cannot exist without the other in our situation.

Keeping our store immutable might initially seem like a disadvantage, however, it's not. We'll soon see that we can configure our compact index only when the entire keys that we have to store are known. To keep per-key memory consumption low, we must ensure that our in-memory index only tracks necessary keys—stored keys.

If we were to update this store in real time, we’ve got two options:

1. We’ll reserve storage for all possible keys to avoid recreating our compact index. This results in higher per-key memory consumption since although our in-memory index is compact, however, has to keep track of more keys than it has to.

2. We can only store keys that are present in the store, and reconfigure the in-memory index every time a new key is added. Even if recreating the index was efficient, rearranging entries in storage every time a new key is added would have resulted in very high write amplification.

Since we only store necessary keys and do not require changing the store, we will keep our store immutable. We will only configure our compact in-memory index once, at the time of the creation of this store. Therefore, we can store keys compactly on storage. The multi-store approach has given us the freedom only to keep one store changing in real time while we can lazily update our other stores. So we will keep the last-level store immutable and only recreate it when there is a significant amount of keys to be updated (represented by the number of intermediary stores accumulated), thereby keeping write amplification low.

## Indexing using tries
Our memory-efficient store will store most of our entries. It is essential to design it to use both storage and memory efficiently. We will index values in our memory-efficient store using tries.

A trie, also called a prefix tree, stores keys such that a path from the root to a leaf node is unique. An internal node represents the longest common prefix—represented by the path from the root to the internal node—that all its descendant nodes have.

[The illustration represents an example of a trie. The term "trie" comes from the word retrieve.]

To sort keys on storage, we will distinguish them using a trie of their shortest unique prefix as the index. The shortest unique prefix of a key is its shortest prefix, which is different from all other keys. Looking for the key in the trie only requires reading up to its shortest unique prefix, at which point we will find a leaf node pointing to the key's index in storage.
```
Note: We are assuming fixed-length entries. Furthermore, we will also store entries sequentially in storage. So the only information required to access a storage entry is an index that lets us know the number of entries to skip to reach our desired entry.
```
Let's look at an example. Here’s how our memory-efficient store will look.

[An example of our memory-efficient store's memory and storage use]

In the illustration above:

1. MSBs stands for the most significant bits

2. The order of MSBs is most to least significant from top to bottom

3. The dark-shaded purple part is the part of the key not shown in the trie. It is ignored for indexing.

4. The grey-shaded indexes are only shown for understanding. These are not stored in storage.
```
Note: To reach a particular index, we skip multiples of the fixed length of key-value entries. For example, to reach index 3 in storage for 10byte key-value entries, we will skip 3(10byte) = 30byte.
```
Our keys are sorted as shown in the storage section. Now for this set of keys, we can have a trie like the one shown in the memory section. We can see that using the shortest unique prefix approach allows us to ignore the maximum possible number of bits of a key for indexing. This is because the suffix part will not affect the key's location.

Let's look at the key on index 4. A lookup of the key 01000110 takes us to index 4 because there are four leaf nodes before the key's leaf node. However, it is important to note that we only need the first two bits (01) to reach the relevant leaf node. Now, we only need to skip four fixed length entries in storage to reach index 4.

[Indexing]

Another important thing to note is that all keys starting with 01 will take us to this leaf node (at index 4). Therefore, this index is not a filter. We will always look for a key in storage for all GET requests in this store. We will match the key from storage with the lookup. We will return the value only if we get an exact match. Otherwise, we will return that the lookup key is not stored in this store.

How does ignoring bits lead to reduced per-key memory consumption? Ignoring a single bit enables skipping one internal node in the prefix tree. Continuing with the example of key 01000110, if we choose to represent this key with three MSBs, we may need to add an internal node to represent the extra bit. The current leaf node will become an internal node, and a new leaf node will store the index 4. The only benefit this yields is that we will ignore requests for keys starting with 011 since there is no representation for such keys in the tree. It results in a lower read amplification. We are already looking up in storage for keys that do not exist since we have prepared our layout in storage to work efficiently in that case—keys are sorted in key order. Hence, we will choose lower memory consumption over marginally lower read amplification.

Although we have kept memory consumption low by ignoring the maximum number of bits in keys in our index, a general tree implementation with pointers is still too expensive. We will improve our design in the next lesson to represent a prefix trie compactly.

