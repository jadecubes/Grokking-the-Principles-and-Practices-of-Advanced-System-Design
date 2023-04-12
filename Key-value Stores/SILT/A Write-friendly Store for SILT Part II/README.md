# A Write-friendly Store for SILT: Part II
## Indexing for reduced per-key memory consumption
Since this store contains the least amount of keys, the overall impact of using the hash table on per-key memory consumption will be low. However, since reduced per-key memory consumption is our goal, we should find ways to reduce the impact of the above hash table. We can reduce per-key memory used by the write-friendly store by:

1. Ensuring high occupancy of the hash table

2. Using a partial key instead of the full key

### Partial-key cuckoo hashing for higher occupancy
The technique we will use for hashing is partial-key cuckoo hashing. The partial-key means we will use a part of the key.

#### Hash functions
Cuckoo hashing requires two hash functions, h1, and h2. Both map a key K to two candidate buckets, h1(K) and h2(K), on the same table.

#### Insertion algorithm
To insert a new key, K1, we will compute its candidate buckets h1(K1) and h2(K1) and check if one, both, or none of the two buckets are empty.

1. If both buckets are empty, store K1 in one of the buckets.

2. If one bucket is empty, check if K1 is stored in the occupied bucket.

    I. If K1 is already stored, update the offset.

    II. Otherwise, store K1 in the empty bucket.

3. If neither bucket is empty, check if K1 is stored in one of the occupied buckets.

    I. If K1 is already stored, update the offset.

    II. Otherwise, K1 displaces K2 in one of the K1 candidate buckets. We will insert K2 into its other candidate bucket.

        i. If the other candidate bucket for K2 is empty, it is stored there.

        ii. If the other candidate bucket is not empty, K2 displaces K3, which resides in its other candidate bucket.

        iii. We will repeat the process in 3(II) for K3 until we find an empty bucket for a limited number of displacements. If we reach our allowed number of displacements and cannot find an empty bucket, we will initialize a new write-friendly store and store the displaced key in one of its candidate buckets in the new store.

[Algorithm](./partialkey)


We will repeat the process of displacing keys for a limited number of attempts. When that number is reached, we will initialize a new write-friendly store and move all entries in the current store to the next key-value store in the pipeline. From this point onwards, the new write-friendly store will cater to arriving PUT and DELETE requests. The configured limit of the number of keys displaced is our way of determining the level of occupancy we would want for our hash table. The number of displacements required to insert a key increases as the occupancy of the hash table increases and becomes very high in the late range (90%+). Based on our hardware and other configurations, we can determine an acceptable number of attempts before we declare the current write-friendly store too occupied to allow for an acceptable cost of inserts.

Let's look at how limiting the number of displacements works. In the example below, the number of allowed displacements to insert a key is 3.

[Algorithm](./partialkey2)

#### Partial-key hashing
We will save memory by not storing the entire key in the hash table. Instead, we will use a tag for each key in the hash table. We will mark a key's entry in the hash table with its tag by storing the tag along with its offset (position in the storage log). Every key-value entry in the write-friendly store will have the following:

1. Complete key and value in the storage log

2. Hash table entry containing:

    I. The key's tag

    II. The key's offset

We will compute each key's tag using a hash function and two non-overlapping key fragments from the lower-order bits of the key. Our two hash functions for cuckoo hashing essentially use only one hash function at the core but apply it to the two key fragments of a key.

To formalize, for a key K,

- h1 applies the hash function to the first key fragment to compute the index of the first candidate bucket of that key. We will denote this index as h1(K) for the key K.

- h2 does the same to the second key fragment to compute the index of the second candidate bucket. We will denote this index as h2(K) for the key K.

While the technique above reduces per-key memory consumption, it raises a problem. For displacing a key and storing it in its alternate bucket, we require computing the index for the alternate bucket. We cannot do this without looking up the complete key stored in storage. If we look up a key for every displacement, this can result in multiple disk seeks/reads for a single PUT or DELETE request and, consequently, a higher read amplification.

We can overcome the above problem by storing the tag for the alternate bucket instead of the current bucket in which the key is stored. For example, we insert the key K1 into its candidate bucket h1(K1), then we will mark it with h2(K1). If we need to move K1 to its alternate bucket in the future, we do not need to compute h2(K1). Now, if we need to store another key, say K2 in h1(K1) since it is the second candidate bucket for K2; more specifically, h2(K2) equals h1(K1). Since we mark entries by their alternate bucket, we do not require to look up either key to compute the alternate bucket. We can locate h2(K1) because we have marked an entry for K1 with this tag. We will store K2 in h2(K2). Now we need to mark the K1 entry with the tag h1(K1):

- If K2 is for a new request, meaning it is not itself being displaced by another key, then we will have values for both its candidate buckets h1(K2) and h2(K2) from the initial computation of candidate buckets. In this case, we will use the value from the initial computation of candidate buckets for new requests and mark K1 with h2(K2) (which is also equal to h1(K1)) and mark K2 with h1(K2).

- If K2 is not a new request, meaning another key is displacing it, then it is marked with h2(K2) (equal to h1(K1)), and we will use this value to mark K1. We will mark K2 with the tag on the key displacing it.

Let’s look at how this will look using our first displacement example.

[Algorithm](./partialkey3)



#### Associativity
We can improve occupancy by increasing the associativity of the cuckoo hash table, allowing us to store more than one entry in a single bucket.
