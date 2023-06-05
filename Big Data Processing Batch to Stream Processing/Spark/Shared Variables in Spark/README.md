# Shared Variables in Spark
In addition to RDDs, Spark's second abstraction is distributed shared variables. We might want to send static data to all the workers (driver-to-worker information flow) or might want to collect some state from all the workers (workers-to-driver information flow). Spark's shared variable abstraction helps with both of these scenarios.

## Shared variables
Setup work is required for some operations, like creating a random number from a specific distribution, for each partition. The user will have to create and send it to the worker with specific partitions every time a task is run on it. Shared variables are used to help cater to the setup overhead. Shared variables can be used to add together data from all tasks or save a large value on all worker nodes and reuse it across many Spark jobs without resending them to the whole cluster. Spark offers two types of shared variables––broadcast variables and accumulators.

### Broadcast variables
Normally, a variable used in a driver node’s tasks is simply referenced in a closure (function). This process can be very inefficient in the following cases.

```
closures can refer to variables in the scope where they are created
```


- If we have large variables like a machine learning model or a lookup table because they have to be deserialized on a worker node every time they are sent with a task

- If a variable is used in multiple jobs

Instead of only sending it once, it needs to be delivered with each job. This creates the need for broadcast variables.

[Workflow of broadcast variables](./1.png)

#### Implementation of broadcast variables
The implementation of broadcast variables is given as follows:

- Broadcast variables are immutable.

- They are initialized on the driver.

- They are shared as read-only copies on all the worker nodes.

- They are lazily replicated on all the worker nodes only once an action is called.

- A use case example could be to share a huge dictionary, potentially of gigabytes in size, that contains words with some information attached to them.

```python
//Supplementing a list of words with numbers
val data = Map("Python" -> 100, "C++" -> 150,      
               "Scala" -> -1500, "Perl" -> 1000)
//Broadcasting data to all workers
val dataBroadcast = spark.sparkContext.broadcast(data)  
                    
              Broadcasting a variable
```

### Accumulators
Accumulators are used for updating a variable in transformations and then propagating it to the driver. A possible way of doing this would be to create variables at the worker nodes and then update them in transformations and send them over to the driver. However, this can be an inefficient way of propagating them to the driver because the tasks or stages can be re-executed (tasks can be re-executed in case if a machine fails and has to recompute the partitions it had stored in it.), updating the variables more than once. This is where accumulators come in. Spark ensures that a restarted task cannot update an accumulator, and they are only updated once when a task is initially executed.

[Workflow of accumulators](./2.png)

#### Implementation of accumulators
The implementation of accumulators is given as follows:

- Accumulators are created at the worker nodes.

- They are mutable variables. However, only worker nodes can update them.

- They efficiently support aggregation in parallel because they can only be added, and addition is both an associative and commutative operation, i.e., the addition order wouldn't matter.

- They are also updated lazily, only once an action is called. Therefore, if they are being updated in an RDD, they are not updated unless the RDD is computed.

- An accumulator can be used to add all the elements of a big array.

```python
// Creating accumulator with initial value 0
val accArray = sc.accumulator(0)
// Adding up all elements of array in accArray
sc.parallelize(array(10, 12, 13, 40)).foreach(x => accArray += x)

                      Accumulator example
```

This lesson taught us how useful shared variables are in Spark. They help reduce setup overhead and can be very used for debugging or optimization tasks.


