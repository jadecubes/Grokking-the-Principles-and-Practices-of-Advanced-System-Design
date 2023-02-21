# Empirical Evaluation of Tectonic's Functional Requirements

We previously discussed how we can use both of our resources (ephemeral and non-ephemeral) optimally. In this lesson, we’ll evaluate how well Tectonic performs on real data. Moreover, we’ll quantify the benefits of storage consolidation and how metadata hotspots are managed. The specification of the Tectonic instance that was used in this study is shown in the table below.
```
This evaluation is primarily based on section 6 (Tectonic in Production) of the paper: Pan, Satadru, Theano Stavrinos, Yunqiao Zhang, Atul Sikaria, Pavel Zakharov, Abhinav Sharma, Mike Shuey, et al. Facebook’s tectonic filesystem: Efficiency from exascale. In 19th USENIX Conference on File and Storage Technologies (FAST 21), pp. 217-231. 2021.
```

```
       Statistics of a Multitenant Tectonic Production Cluster
Capacity      Used Bytes	    Files	      Blocks	     Storage Nodes

1590 PB	      1250 PB	        10.7 B	     15 B	         4208
```

## Multitenant exabyte-scale clusters
The capacity of a single cluster in the production Tectonic is at an exabyte scale. The details of the multitenant cluster are in the table above. We had the 1250 PB (around 70%) of storage capacity used out of 1590 PB at the time of snapshot operation, having 10.7 billion files mapped on 15 billion blocks. By utilizing just 4K storage nodes, Tectonic’s design can reach an exabyte scale.

## Efficiency from storage consolidation
We had two tenants in the cluster mentioned in the table above—blob storage and data warehouse. The space used by the blob storage is around 49%, and the space used by the data warehouse is around 51%. We applied a regular load of very large jobs and large workloads to the data warehouse. The traffic of blob storage was predictable and smooth compared to the data warehouse workload.
### Sharing surplus IOPS capacity
An increase in storage demand from the data warehouse is handled by the cluster utilizing the additional capacity of IOPS by blob storage with its consolidation. The queries of blob storage are IOPS limited and frequently smaller, whereas the requests of the data warehouse are bandwidth limited and frequently larger. Therefore, neither bandwidth nor IOPS accurately depict disc IO use. The resource that causes storage operations to lag is called disc time, which measures how frequently a certain disc is in use. Tectonic needs adequate free disc time to accommodate an increased storage load. For example, if a disc did 10 IOs in a second, each taking 50 milliseconds, it would be busy for 500 out of 1000 milliseconds (for search and retrieve operations). Disk time is utilized to account for usage by different types of requests accurately.
```
             Normalized Disk Time Demand vs. Supply

           Warehouse	        Blob Storage	     Combined	      
Supply	     0.51	              0.49	             1.00	

Peak 1	     0.60	              0.12	             0.72	

Peak 2       0.54	              0.14	             0.68	

Peak 3	     0.57	              0.11	             0.68


```

The table above shows the normalized disc time demand of the production cluster for three days of daily peak of blob storage and data warehouse, along with the supply of disc time by running each on their own cluster. To account for cluster space that has been used, we normalize by total disc time. The two graphs (aggregated cluster IOPS and bandwidth) below represent the same three days of traffic by the daily peaks. On each of the three days, there is a bigger demand than there is supply for data warehouses, necessitating disc overprovisioning to manage it independently.

[Aggregate cluster IOPS](./1.jpg)

[Aggregate cluster bandwidth](./2.jpg)




## Metadata hotspots

Whenever the client needs to access a file, it has to first go to the Name layer of the Metadata Store with the filename to get the file’s metadata. Hash-partitioning (uniform distribution of workload) of all the layers of the Metadata Store is not much useful for the Name layer since it gets a lot of requests. Neither the File nor the Block layer gets as hot as the single directory of the Name layer does. As the demand for the Metadata Store fluctuates, hotspots in the metadata shards may develop.

[Metadata hotspots](./hotspots)

The resource limiting the provision of metadata activities is the number of queries per second (QPS). For the Metadata Store to be able to withstand load spikes, each shard must match the QPS criterion. A maximum of 
10K QPS can be handled by each production shard. This limitation is placed on the metadata nodes’ resources by the current isolation approach. The cluster’s metadata shards’ QPS for the three layers (Name, Block, and File) are shown in the graph below. All shards are beneath this limit at the File and Block levels.

[Peak metadata load](./peek.jpg)


As a result of holding extremely hot directories over the span of these 3 days, almost 1% of Name layer shards exceed the QPS limit. A backoff is followed by a new attempt to process the small portion of unprocessed metadata queries. As a result of the backoff, the metadata nodes are able to handle retried requests effectively and remove the majority of the first surge. This method allows Tectonic to handle the significant increases in metadata demand from the data warehouse successfully, and all other shards operate below their limit.

The Name, File, and Block layers have different load distributions among shards. Due to the fact that each higher layer co-locates more of a tenant’s activities, it has a wider distribution of QPS per shard. For example, all the lookups of directory-to-file for a specific directory are managed by a single shard. The data warehouse frequently read directories with similar names and also read many files within a single directory, which increases the workload of the Name layer. The following two techniques can be employed to solve the metadata hotspots:

1. Hash-partitioning: It reduces the placement of files and directories in the same shard. In addition, fewer nodes are required to manage spikes in the metadata load, which would have been increased by using range partitioning. Range-partitioning the File layer means that the directory and its files will be stored on the same shard, increasing the load on the File layer since there are many more file operations rather than directory operations.

2. Data warehouse collaboration with the Metadata Store: We can maintain the list of files’ IDs (along with their names) in the data warehouse, which is once accessed by the data warehouse. Maintaining this list helps the worker threads of the data warehouse to get the file ID within the data warehouse instead of asking the Metadata Store to avoid metadata hotspots.

In summary, the empirical study shows that Tectonic can easily scale to the exabytes level for multiple tenants. The sharing mechanisms work correctly so that collective needs are met. Additionally, the Metadata Store performs well under the expected workload. In the future, storage and metadata services can scale horizontally by adding more nodes.


