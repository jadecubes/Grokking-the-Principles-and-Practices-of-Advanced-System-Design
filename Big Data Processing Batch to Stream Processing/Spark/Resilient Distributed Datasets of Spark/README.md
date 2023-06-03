# Resilient Distributed Datasets of Spark
RDDs provide a restricted form or an abstraction of shared memory based on coarse-grained transformations rather than ﬁne-grained transformations. Simply put, RDDs are distributed data on a collection of worker nodes' memories based on coarse-grained transformations in a cluster.
```
coarse-grained transformations: A transformation applied with the help of a function like Map and Reduce on a bulk of data.
 ﬁne-grained transformations: A transformation applied to an entity of a database.
```

# Creation of RDDs
RDDs are an object in the language they are being made. We can build an RDD in the following ways.

## From a file
An RDD can be built from a file in a distributed file system (DFS). It would create an RDD in which each block of data in DFS will be a partition in the RDD, and each record in a partition would represent a line in that file.
```python
val RDD = spark.textFile("dfs://file")
```
## From a collection
An RDD can be built by parallelizing a collection that is inherently sequential. It could be an array or a list. The user can decide the number of partitions to make out of a collection.
```python
val RDD2 = (1 to 9999).toList.par
```

## From another RDD
RDDs can be made from other RDDs in two ways.

- An RDD can be built by transforming an already built RDD––from the lines that contain NULL in them.
```python
val RDD3 = RDD.filter(_.contains("NULL"))
```
- It can also be built by altering the persistence of an existing RDD. RDDs, by default, are lazy (created on demand when used in parallel operations) and ephemeral (eliminated from the memory afterward). To change this behavior, Spark provides two actions:

    - Cache: The cache action makes Spark keep data in memory for further use after its creation. However, if there is no memory to cache the data, it recomputes it whenever used again. This process is opted to ensure that the system keeps running even if a node fails or there is a memory shortage.
    ```python
    val RDD4 = RDD3.cache()
    ```
    - Save: The save action evaluates and saves the data to a distributed file system to perform operations on it in future.
    ```python
    val RDD4 = RDD3.saveAsTextFile("dfs://saved_data")
    ```
    [another RDD]
# Representation of RDDs
## Controlling partitions
# RDDs vs. DSMs
