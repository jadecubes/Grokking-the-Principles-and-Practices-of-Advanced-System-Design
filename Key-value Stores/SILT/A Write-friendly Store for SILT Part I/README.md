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
### Writing sequentially
## In-memory hash table
