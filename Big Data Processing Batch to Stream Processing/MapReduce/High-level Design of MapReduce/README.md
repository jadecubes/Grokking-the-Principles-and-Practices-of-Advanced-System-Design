# High-level Design of MapReduce
Let’s start designing our system on a high level. This lesson will overview the components and their combined implementation to achieve the functional and non-functional requirements mentioned in the previous lesson.

## High-level components
Let’s list the main components we need to design the MapReduce system.

- Distributed file system: We’re using the Google File System (GFS) as our distributed file system for storing the input data. We’ll explain the detailed functionality of this distributed system, concerning our system, in the detailed design lesson of this chapter.
- Cluster: We need a cluster of machines to process the data in parallel.
- User program: We need the user program, mainly the Map and Reduce functions, to run on all the workers for data processing.
```
workers: A worker is one commodity machine inside a cluster capable of achieving the system’s functionality independently. It gets its share of the work through a master.
```
- Scheduler: Before starting the MapReduce operation, the user program gets installed on all the workers with which they can perform the dedicated task assigned to them. We need a scheduler to manage the job assignment for various workers. It mainly optimizes the workers’ usage in the cluster.

## High-level implementation
Our system works on the following scheme:

1. We divide the input data into a specific number of splits, processed individually on workers using a Map function, producing their own intermediate key-value pairs.
2. We distribute these intermediate key-value pairs into the next type of workers (performing the Reduce operation), using the following hash function:
```
hash(key) mod number of workers implementing the Reduce function
```
Each worker writes their output to an output file using key-value pairs produced as results of a Reduce function.
```
Note: The cornerstone of our programming interface is that the programmer only needs to provide us with the two functions, Map and Reduce. The system handles the rest of the details automatically.
```

[High-level implementation](./hld.png)

## Programming model
As we outlined earlier, MapReduce is a restricted programming model that requires less effort from its users. Hence, internal implementations of this model automatically ensure the following requirements:

1. Parallelization
2. Data distribution
3. Dynamic load balancing
4. Fault tolerance
The user doesn’t need to worry about handling internal operations and only needs to provide the input dataset, along with two functions. The Map function processes the splits of the dataset, while the Reduce function combines the processed intermediate results to produce the final result-. Let’s formally define the Map and Reduce functions for better understanding.


### The Map function
The Map function takes a split, in the shape of a <input_key, input_value> pair, as input and produces intermediate <output_key, intermediate_value> pairs.
```
map(input_key, input_value) ->  list(output_key, intermediate_value)
```

```
Note: The MapReduce library consolidates all the intermediate  values associated with an output key and passes them to the Reduce function.
```
[The Map() function](./mapfunc.png)

### The Reduce function
The Reduce function accepts an input pair of the form <output_key, list(intermediate_value)> and returns an output pair of the form <output_key, list(output_value)>.
```
reduce(output_key, list(intermediate_value)) -> list(output_value)
```

[The Reduce() function](./reduce.png)

```
Question
How do we handle a situation where the count of intermediate values exceeds the memory bounds?

Answer
We might encounter situations where the number of values is too large to fit in the memory. To counter such an issue, we pass the intermediate values to the Reduce function using an iterator, allowing us to keep track of them. We feed the iterator to the reducer incrementally from local disks or the network.
```

```
Note: It is important to note that the domain of the Map input data keys and values typically differ from that of the Reduce output data keys and values. Moreover, the intermediate keys and values – the output of Map and input to Reduce – have the same domain as the Reduce output data keys and values.
```

### Word count example
Let’s understand the above functions with the help of a simple word counting problem with the following problem statement: “Count the number of occurrences of each unique word in a large set of documents.”

The user needs to define two functions, Map and Reduce, corresponding to the two main stages of the MapReduce operation. Let’s look at the pseudocode for both of them.

```python
map(String key, String value): 
// key: document name
// value: document contents 
for each word w in value:
  EmitIntermediate(w, "1");
  
              The pseudocode for the Map() function
```

The Map function emits the count of occurrences associated with each word. The associated count is simply 1 for each word in this example.

```python
reduce(String key, Iterator values): 
// key: a word
// values: a list of counts
int result = 0;
    for each v in values:
      result += ParseInt(v);
    Emit(AsString(result));
    
       The pseudocode for the Reduce() function
 ```
The Reduce function sums up a particular word (an intermediate key) count and emits it as an output.
```
Note: One Reduce call typically produces zero or more output values for an intermediate key (depending on the specific use case). For in the word count example, Reduce generates one output for each word, for example, <the 100> output from Reduce will mean that there were 100 instances of the word the in the text.
```

### The MapReduce specification object
In addition to the Map and Reduce functions, the user needs to provide the code for a MapReduce specification object with the names (URLs to the files in GFS) of the input and output files and the additional tuning parameters. The MapReduce library passes this specification object to the MapReduce function at invocation and is responsible for linking all the segments of the user code.

