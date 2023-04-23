# Quiz on DynamoDB

```
** Question 1 *8
How does our design take advantage of the multi-tenant architecture?

Answer
The multi-tenant architecture allows us to use our resources efficiently. Dedicating one node for one table could result in underutilization if the table contains a limited amount of data. To scale the table, we might need to dedicate another node if the first node cannot store all the table’s data. One node dedicated to the table is utilized well, while the other could be underutilized. The multi-tenant architecture allows us to spread our tables across our network as partitions, thus spreading the potential effect of underutilizing dedicated resources.

Multi-tenant architecture also enables easy scaling since we can allocate and deallocate resources at the partition level.
```
