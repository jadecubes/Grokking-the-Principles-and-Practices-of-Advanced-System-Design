# Parallel Operations in Spark
The main focus of this lesson is to know how we would perform operations on an RDD's workers in parallel to transform it into another RDD, and how we would extract information from these distributed datasets. Spark provides parallel operations solely for this purpose. The users don't have to extract or transform data from each worker separately. The Spark system applies each function simultaneously across all the workers in an RDD. Parallel operations can transform RDDs to get new RDDs. There are generally two types of operations we can perform on RDDs––transformations and actions.

## Transformations
These are the operations applied on an RDD to get a new RDD. Transformations are lazy operations, i.e., they get executed only when an action is called. Instead of modifying the data immediately, Spark waits until action is called and builds an execution plan to make all the transformations run efficiently whenever they are executed, possibly pipelining many transformations. Since RDDs are immutable, the input RDD remains the same. Spark supports many transformations, such as map(), ﬂatMap(), mapValues(), ﬁlter(), groupByKey(), reduceByKey(), union(), join(), cogroup(), crossProduct(), sample(), partitionBy(), and sort(). Making another RDD from an RDD and then applying a transformation on it again to get another RDD makes a transformation chain or pipeline. Spark provides a graph-based representation for RDDs called a lineage graph to track the lineage of transformations.

The lineage graph shown below contains a series of transformations in MMA fights. First, UFC fights are filtered out from the data, then the winners of each fight are mapped with an integer 1. Finally, all the wins of each fighter are reduced to give out the total wins of each fighter.

[Lineage graph of RDD transformations]

When a transformation is applied to an RDD, it gets applied to all the partitions of the RDD. A partition in the parent RDD can be used to create one or more RDD partitions in the child RDD. Based on these facts, Spark provides narrow transformations and wide transformations.

### Narrow transformations
An RDD transformation that results in each partition contributing to build only one partition in the child RDD is called a narrow transformation. Two or more partitions can also lead to the formation of only one partition in the child RDD, provided all the parent RDD partitions are in different RDDs. Some of the narrow transformations are explained below.

#### The map() function
This operation is a one-to-one mapping. It can be used to transform each RDD element into an element of a new RDD. The following code takes names in rdd and maps it with the value 1 to get a newRDD.
```python
val newRDD = rdd.map(name => (name,1))
```

#### The filter() function
This operation is used to filter out specific elements of an RDD partition. The following code filters david from the names in RDD and makes a new RDD of names that only include david.
```python
val newRDD = rdd.filter(name => name.contains("david"))
```

#### The join() function
This operation is only available at RDDs with key-value pairs. It is used to combine two RDDs' elements based on their keys. If two co-partitioned RDDs (those RDDs with the same number of partitions are made with the same technique) are joined, it will not cause any data shuffling. Therefore, the resulting transformation would be a narrow transformation. The join type is specified by an argument in the command, which in this case, is inner. Let's say we have two co-partitioned RDDs with data in the form of key/value. In the following code, an inner join is applied, returning only the pairs with common keys in both RDDs.
```
inner join: The resultant RDD contains only the rows that are present in both input RDDs.
```
```python
val rdd = spark.sparkContext.parallelize(Seq(("Human",2),("Lion",3),("Cat",1)))
val otherRDD = spark.sparkContext.parallelize(Seq(("Human","Omnivore"),("Lion","Carnivore"),("Deer","Herbivore"),("Cat","Omnivore")))
val newRDD = rdd.join(otherRDD)
```
The output of this operation is as follows:
```
(Human,2,Omnivore)
(Lion,3,Carnivore)
(Cat,1,Omnivore)
```
#### The union() function
This operation is used to create an RDD by combining two RDDs.
```python
val newRDD = rdd.union(otherRDD)
```

[Union operation]

### Wide transformations
An RDD transformation that results in each partition contributing to building multiple partitions in the child RDD is called a wide transformation.

#### The reduceByKey() function
This operation performs the aggregation of data. It receives key/value pairs and aggregates the values of elements with the same keys with the help of a function. The following code reduces the data in rdd by applying + operator on their values.

```python
val rdd = spark.sparkContext.parallelize(Seq(("Human",1),("Lion",1),("Human",1),("Deer",1),("Cat",1)))
val newRDD = rdd.reduceByKey( _ + _ )
```
The reduced results in the newRDD would look like this:
```
(Human,2)
(Lion,1)
(Deer,1)
(Cat,1)
```


#### The join() function
This operation is similar to the join operation we used in narrow transformations. However, the only difference is that two non-copartitioned RDDs, meaning that RDDs with a different number of partitions or RDDs made by different partitioning techniques, are joined. This kind of join would cause a shuffling of data that will result in multiple partitions of an RDD, contributing to make a single partition in the child partition. So, the resulting transformation would be a wide transformation.
```python
val rdd = spark.sparkContext.parallelize(Seq(("Human",2),("Lion",3),("Cat",1)))
val otherRDD = spark.sparkContext.parallelize(Seq(("Human","Omnivore"),("Lion","Carnivore"),("Deer","Herbivore"),("Cat","Omnivore")))
val newRDD = rdd.join(otherRDD)
```
The output of this operation is as follows:
```
(Human,2,Omnivore)
(Lion,3,Carnivore)
(Cat,1,Omnivore)
```
#### The groupByKey() function
This operation receives key/value pairs, shuffles, and groups the key/value pairs based on their keys.The following code groups the data in rdd with respect to their keys.
```python
val rdd = spark.sparkContext.parallelize(Seq(("Human",2),("Lion",1),("Human",1),("Lion",3),("Cat",1)))
val newRDD = rdd.groupByKey()
```
The grouped results in the newRDD will look like this:
```
(Human,(2,1))
(Lion,(1,3))
(Cat,1)
```

[GroupByKey operation]

## Dependencies
When an RDD is created, its relationship with the parent data can be classified into two types depending on the type of transformation used to create it. Dependencies are important in Spark because they help define the program's execution stages in the scheduler section of Spark (for example, the Spark scheduler might be able to combine many narrow transformations into one).

- Narrow dependency: When a narrow transformation is used to create an RDD, it results in narrow dependency.

- Wide dependency: When a wide transformation is used to create an RDD, it results in wide dependency.

```
                                           Comparison of Dependencies
Narrow dependency                                                                                 Wide dependency

At most, one partition of a child RDD is dependent on each partition of a parent RDD.           Each partition of a parent RDD is used to compute multiple partitions of a child RDD.

Narrow dependencies allow pipelined execution to compute all parent and child partitions        Wide dependencies shuffle the data across the cluster and needs data from all the partitions to be 
on a group of machines in a single cluster. Users can apply any operations on a parent RDD      present to compute the child partitions.
on an element-by-element basis.                                                    

It also allows easy partition recovery, since the lost child partition needs to recompute       It results in relatively complex partition recovery because the loss of a single partition will 
data from only one partition of the parent RDD, in parallel, on a separate node.                cause computation at multiple partitions of the parent RDD, and if they are not present, those                                                                                                     parent partitions will also need recomputation.



Map, filter and union operations result in narrow dependencies.                                  Both groupByKey and join operations whose inputs are not co-partitioned result in wide                                                                                                            dependencies.

```

[Dependencies]

## Actions
The transformations let Spark build up a logical plan of execution. Actions are operations that trigger the execution of that logical plan. These are the operations that return non-RDD values. Actions are performed when we want to extract information from the data. Spark supports actions such as count(), reduce(), collect(), and lookup(). Some of the actions are explained below.

### The count() function
This operation is used to get the count of elements in the dataset.
```python
val no_of_elements = rdd.count()
```
### The reduce() function
This operation is used to aggregate the dataset with the help of a function.
```python
val reduced = rdd.reduce((x,y) => x*y)
```
### The collect() function
This operation returns all the data in an RDD as a list.
```python
val data = RDD.collect()
```
### The lookup() function
This operation looks up a certain key in data inside an RDD and returns a grouped list of values against that key.
```python
val data = rdd.lookup("Human")
```

All of these actions, when executed in Spark, initiate parallel processing on the RDD level. Each transformation is applied one by one on their respective unique RDD's workers in parallel, unless multiple transformations are being applied to a single RDD. Those can be executed simultaneously.

In this lesson, we learned about the parallel operations in Spark, how they could help create new RDDs, and extract meaningful information from them, and how partitioning of RDDs can effect the dependencies of the resulting RDDs.
