# MapReduce: Detailed Design

Let’s discuss the MapReduce design in detail, including estimations and the detailed role of all the components—their internal mechanisms and the execution flow.

## Estimations
In our design, we’ll have to specify the number of workers required to achieve the task based on the input file size. Let’s make some assumptions before we formulate the workers’ calculations.
#### Assumptions
- The input file size is 12.8 TB (we assume this data size to simplify our calculations, as we will see shortly.)
- We will distribute the input data into splits of roughly 16 MB to 64 MB (we use multiples of 16 MB because the underlying storage of GFS’s chunk size is 64 MB, and we want to maximize data locality benefits).
- We can distinguish the tasks into two categories based on their executing functions— Map and Reduce.
- All workers are identical in their memory and computation resources.
- There are no malfunctioning workers.
```
Note: In this chapter, we’ll refer to workers as machines. Even though our design automatically handles the differences in workers’ memories and computation resources and the cases of faultiness, these calculations remain independent of these points.
```


### Estimating the Map tasks (M)
### Workers estimation
### Estimating the Reduce tasks (R)
## Components
## Execution flow
## Locality
## Task granularity
## Backup tasks
