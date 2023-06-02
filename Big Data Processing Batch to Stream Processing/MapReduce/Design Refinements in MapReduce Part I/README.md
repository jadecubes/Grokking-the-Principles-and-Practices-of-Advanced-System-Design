# Design Refinements in MapReduce: Part I
Real-world systems are rarely designed in one go—it often takes many iterations to improve the design. As initial versions of our system are deployed in production, we get usage data and possibly new insights. In this and the next lesson, we will improve many aspects of MapReduce design.

Ordering our refinements goes along with the execution flow of the system.

## Input and output types
Let’s analyze the supported input and output types by the MapReduce library.

### Input types
By default, the MapReduce library supports reading a limited set of various input data types. Each input type implementation automatically handles the data splitting into meaningful ranges for further processing by the Map tasks.

#### Examples
As we know, the data gets partitioned into key-value pairs before it is processed by the Map tasks. The “text” mode input processes each line as a key-value pair, such that:

- The key is an offset in the input file.
- The value is the content of that line.
This mode ensures that the partitioning happens only at the line boundaries.

[Data processing in the text mode](./textmode.png)

#### Support for new input types
Based on the desired functionality, the users can also define a new reader interface to add functionality for a new input type. For example, we can define a reader to read data from a database or a memory-mapped data structure.

### Output types
The MapReduce library also supports various output types by default, and similar to the input types, it also gives the functionality to define new output types.

Using custom types for data is a powerful extension that enables end programmers to read and write data from many different sources and sinks.


## Partitioning function
The distribution of the intermediate data to each of the user-defined R partitions is handled by the partitioning function. By default, the MapReduce library provides a hash(key) mod R partitioning function.
 
[The default partitioning function](./partitioning.png)

### Customization of the partitioning function
The partitioning for many computations can be reasonably uneven in the real world, resulting in poor speed-up because only a few workers do most of the work (because the hash function sends most of the data to a few buckets). To achieve linear speed-up, it is critical that each partition gets roughly the same amount of data so that all the available servers can be employed to do the work in parallel.

Instead of using the default partitioning function of hash(key), such use cases demand partitioning the data by another different function of the output key.
```
Note: The user can define a customized function to partition the data across R partitions of the output file.
``` 

#### Examples
An example scenario can be where the output keys are URLs, and we want to partition the URLs from a single host to a separate partition of the output file. If we use the default partitioning function of hash(URL), we wouldn’t be able to generate separate files per hostname. To achieve the desired output, we can modify the partitioning function to hash(Hostname(URLkey)).

Let’s assume we have the following URL to a course on the Educative platform: https://www.educative.io/courses/grokking-modern-system-design-interview-for-engineers-managers. Instead of taking the hash of this URL, we first resolve its hostname, which will be www.educative.io in this case, and then take the hash of that hostname. This way, all the Educative URLs will map to just one Reduce partition instead of going to various partitions if we had taken the hashes of URLs.

[Examples](./example)

## The Combiner function
As established earlier, a reducer gets input from all the Map tasks, with each mapper incarnation contributing a portion (a bucket out of R buckets). When received on the reducer, this input might contain a significant repetition having multiple batches of similar output keys. All these repetitions waste the network bandwidth and should be dealt with before sending them out on the network. The user can define a customized combiner function to merge the similar output keys’ data before sending it out to the Reduce function, noticeably saving the network bandwidth and speeding up certain MapReduce operations at the reducer.
```
Note: The Reduce function is commutative and associative, so adding the combiner function before it doesn’t affect its operation.

commutative: A function is commutative if changing the order of the operands does not change the result. (Source: W. E. Weihl. “The impact of recovery on concurrency control”, Proceedings of the eighth ACM SIGACT-SIGMOD-SIGART symposium on Principles of database systems - PODS '89, 1989)
associative: A function is associative if rearranging the parentheses in its input expression doesn’t change the result.
```
#### Examples
Let’s take the example of the word count problem. The Map function produces millions of records of the form  <the,1>. Instead of sending these records individually to the Reduce function, we can partially merge them by doing a local sum and sending that result instead. It will avoid the unnecessary network bandwidth being burnt and save a lot of time by the Reduce function, which would have to merge these records individually otherwise.

[Merging <key, value> pairs with the same keys using the combiner function]

### Comparison of the Combiner function and the Reduce function
Functionality wise, there is no difference between the Combiner function and the Reduce functions. Typically, we use the same code for both these functions. The only difference is their implementation location and the handling of their outputs by the MapReduce library.

- The MapReduce library writes the output of the Combiner function to the intermediate file destined to be sent to a reducer.
- The MapReduce library writes the output of the Reduce function to the final output file.


## Guaranteed ordering
The system needs to distribute the data among workers effectively and logically to engage as many workers as possible in the cluster. This implementation also has underlying by-products that ensure that the final output is sorted and can be analyzed accordingly.

The MapReduce library ensures that the intermediate <key,value> pairs are processed, within a partition, in increasing key order, ensuring sorting during all operations. It makes it feasible for reducers to write outputs in a sorted manner, facilitating quicker random access lookups on keys in the final output file.

A sorted order of intermediate keys might help to simplify the Reduce function. For example, for word counting, when the keys starting with the finish, the Reduce function knows that it will not get the key the after that and can emit its output. If intermediate data was not sorted on the keys, the Reduce function would have to keep the partial sums in memory until it processes all of its data.

[Sorting the intermediate <key, value> pairs in increasing order in a partition](./sorting.png)

## Side effects
In addition to having only the program-generated output files for Map or Reduce tasks, the user can also produce additional auxiliary output files (side-effects) for both these functions (possibly for debugging purposes).

### Restrictions on the side-effects
By default, in such an added functionality, the MapReduce library outputs to a temporary file and eventually renames it when it has entirely created it. The sole responsibility of making these side-effects atomic and idempotent lies on the application writers (users).


Let’s see the process of an auxiliary file generation using the slide deck below:

[Restrictions](./restrictions)

Sometimes, the users wish to produce multiple files from a single task. The MapReduce library does not support atomic two-phase commits in this case. Therefore, the tasks producing several output files while ensuring consistency should be deterministic.

```
two-phase commits: In transaction processing, databases, and computer networking, the two-phase commit protocol (2PC) is a type of atomic commitment protocol (ACP). It is a distributed algorithm that coordinates all the processes that participate in a distributed atomic transaction on whether to commit or abort (roll back) the transaction. (Source: Wikipedia)
```

The MapReduce library expects the Map and Reduce functions code (provided by the programmers) to be deterministic and idempotent so that potentially repeated executions on the same data produce the same results. If the Map or Reduce code is not idempotent or has side effects that are not idempotent, then the programmer should be aware that the final results might not be valid.

```
Idempotent: Producing the same results on repetition.
```

```
Note: Even with MapReduce’s restricted programming model and further expectations of the idempotent Map and Reduce code, this library is applicable to a wide range of applications.
```
