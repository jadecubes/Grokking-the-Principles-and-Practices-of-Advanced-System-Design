# A Write-friendly Store for SILT: Part I
## Preamble
In the last lesson, we decided to use multiple stores in our design, each optimized for a particular purpose. These stores include a write-friendly store, intermediary stores, and a memory-efficient store. Collectively, these stores will help our design achieve good performance and low memory overhead per key. Let's dive into our first store: the write-friendly store.

Our write-friendly store will handle all PUT and DELETE requests. The PUT requests represent new entries or update old entries. The DELETE requests are special operations that request the removal of a key-value entry. For the GET request, we'll later use a higher-level wrapper to find a key in all the stores. Doing so is necessary because, over time, keys trickle from a write-friendly store to a memory-efficient store.

## Appending to storage log for fast write
Memory provides fast random access. However, storing all the key-value data in memory will need a lot of RAM (which is fairly expensive and might not be enough to hold all the data). Writing to storage is slow, and storage generally does not perform well with random writes. However, sequential writing to storage is faster than random writing.

Before moving on, let's refresh the difference in random and sequential writing performance for the most commonly used storage hardware.

### Random vs. sequential writing
We will not dive into the details of random vs. sequential access. We will compare the differences in random and sequential performances of some modern hardware to emphasize the importance of the considerations made in our design.
```
Note: We prefer writing and reading traditional rotating disks and newer flash-based disks sequentially for different reasons. Sequential writing is faster on the rotating disks because doing so avoids unnecessary head-seek latencies, and for flash-based disks we can better utilize the storage cells to increase its overall lifespan. Additionally, sequential operations enable us to benefit from on-disk caches that can prefetch data next in line.
```
The following table lists all the hardware we selected for this analysis. Except for the first disk, all other disks are different kinds of SSDs. For the following table, all conversions from IOPS to MB/s assumes 4KB blocks.


```
Name (Capacity)	                                  Application	                Sequential Write Performance	              Random Write Performance
Western Digital Blue (1 TB)	                       PCs	                        Up to 130 MB/s ~= 800 IOPS	              Up to 53 MB/s ~= 450 IOPS
Samsung 980 (1 TB)	                               PCs                        	Up to 3,000 MB/s ~= 0.73M IOPS          	Up to 2,000 MB/s ~= 0.48M IOPS
Samsung 990 PRO (1 TB)	                           PCs and Game Consoles	      Up to 6,900 MB/s ~= 1.68M IOPS	          Up to 6,300 MB/s ~= 1.55M IOPS
Intel® Optane™ SSD DC P4800X Series (750 GB)	     Data Centers	                Up to 2,200 MB/s ~= 0.54M IOPS	          Up to 2,300 MB/s ~= 0.55M IOPS
Intel® Optane™ SSD DC P5800X Series (800 GB)	     Data Centers	                Up to 6,100 MB/s ~= 1.49M IOPS	          Up to 5,500 MB/s ~= 1.35M IOPS
```

```
Source: https://www.storagereview.com/review/western-digital-caviar-blue-1tb-review-wd10ealx
Source: https://semiconductor.samsung.com/consumer-storage/internal-ssd/980
Source: https://semiconductor.samsung.com/consumer-storage/internal-ssd/990-pro-with-heatsink/
Source: https://www.intel.com/content/www/us/en/products/sku/97154/intel-optane-ssd-dc-p4800x-series-750gb-2-5in-pcie-x4-3d-xpoint/specifications.html
Source: https://www.intel.com/content/www/us/en/products/sku/201860/intel-optane-ssd-dc-p5800x-series-800gb-2-5in-pcie-x4-3d-xpoint/specifications.html
```

```
On the official websites of the above-mentioned storage products, their manufacturers have listed sequential write performance in MB/s and random write performance in IOPS. Why is that?

This is because sequential writing almost only requires writing operations. The given rating is for when the address to write is known, and the only work left is to write the data onto the drive. Random writing is different. In the worse case, every write operation will require at least one read operation. Therefore, the rating for random write performance is listed in IOPS rather than MB/s. In the best case, an entire random write operation is essentially sequential–the addresses of a sequence of random writes are one after the other.

Based on the information above, we can think of the number of IOPS in the sequential write performance column as write operations for 4KB blocks per second. The number of MB/s in the random write performance column is the data that can be simultaneously read and written for 4KB blocks for random writing.
```
### Writing sequentially
We’ll write sequentially to storage. Our write-friendly store will append all PUT and DELETE requests to an in-storage log making the writing process fast.

[Sequential writing to the in-storage log]

## In-memory hash table
The process above presents a problem. Accessing a key would require O(n) time, where n is the number of key-value entries stored in the key-value store. We’ll have to search for the key in the unsorted log for its entry. The following reasons require us to have faster access to specific keys.

- We may need to search for keys for GET requests.

- We will need to access keys later to move our entries to memory-efficient stores.

A major consideration we need to make is that this store will contain its keys' latest (most updated) values. For GET requests, we should look for the key in this store first to avoid returning an incorrect or outdated value (from the subsequent stores).

We cannot afford a linear search in this store for all GET requests and we need to be able to access our keys in constant time. Furthermore, since this store will contain the least proportion of overall stored key-value entries (because we would have moved keys to a more memory-efficient store), this store might not have the required key stored most of the time. So, in addition to returning stored values faster, we also need to reject requests in constant time too in case the key is not stored.

To solve the above problem, we will maintain an in-memory hash table that will map keys to their offset in the log; an entry in the hash table is added right after a successful entry in the in-storage log. Now we can access all our keys in the store in O(1) time.

This hash table also serves as an in-memory filter. The hash function will give us the hash value for the key we are looking for, and if that hash value does not exist in the hash table, the key is not present in this store. This eliminates any storage seeks required for keys that do not exist in the write-friendly store, resulting in lower read-amplification. We will explore this further in the GET request lesson in A Memory-efficient Store for SILT: Part III.

[Log in storage with an in-memory hash table to store offset.]
