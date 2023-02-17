# What is the weighted round-robin load balancing technique?

## Overview
The purpose of load balancers is to improve the performance of applications and decrease the burden by "efficiently" distributing the incoming traffic across a group of servers. For user-facing applications, this will result in improved response times.
```
Note: Here, we will mainly talk about application load balancers.
```

```
Efficiency depends on server selection.
```
Now, let’s jump right into the details of the weighted round-robin load balancing technique.

## About the technique
This technique is similar to the round-robin load balancer. But in the weighted round-robin load balancer, the network administrator assigns a numeric weight to all of the servers behind the load balancer. The weights can be assigned based on factors such as the server’s processing power or total bandwidth.

- A server, say ServerA, with the most processing power will be assigned the maximum weight. It will also receive the maximum proportion of incoming requests from the load balancer.

- A server, say ServerB, with half the processing capacity compared to ServerA will be assigned a weight that is half of the actual weight of ServerA. Additionally, it will receive the proportion of incoming requests from the load balancer accordingly.

- A server, say ServerC, with the lowest specifications will be assigned the lowest weight, and it will receive the minimum proportion of incoming requests from the load balancer.
```
Note: The weighted round-robin load balancer is a static load balancer, as it does not modify the state of the servers while distributing incoming traffic.
```

## Example
Let’s understand this with the help of an example:

Suppose we have three servers —ServerA, ServerB, ServerC— with weights (5, 2, 1) that are waiting to serve incoming requests behind the load balancer.

The load balancer will forward the first five requests to ServerA, the next two requests to ServerB, and then one request to ServerC.

If any of the other incoming requests arrive, the load balancer will forward those requests back to ServerA again for the next five incoming requests, then ServerB will get its turn, and after that the requests will be forwarded to ServerC. The cycle will continue on this way.

This cycle is shown in the illustration below:

[Weighted round-robin load balancing technique](./wrr.jpg)

```
In the above scenario, we talked about an approach that is called CWRR, Classical Weighted Round Robin.

Interleaved weighted round robin
There is another approach that is called IWRR, Interleaved Weighted Round Robin.

It will first calculate the maximum weight by taking maximum of all of the numeric weights.

w max =max{w1 , w2 , ... wn }

where n is the total number of numeric weights assigned to n servers.

In our case, the maximum weight is 5. So, each cycle will split into 5 rounds. We can assign one request at round r to each server only if r ≤ wi

if the weight w of the ith server is less than or equal to the value of round r only then the request will be assigned to this server.

Example
Let’s take the same example, we have three Servers (ServerA, ServerB, ServerC) with weights in (5, 2, 1) waiting to serve incoming traffic behind the load balancer.

- In the first round r = 1, ServerA, ServerB, and SeverC will receive one request sequentially.
- In the second round r = 2, ServerA and ServerB will receive one request sequentially, but not ServerC because it doesn’t satisfy the desired condition.
- In the third, fourth, and fifth round, r = 3 or r = 4 or r = 5, only ServerA will receive one request, rest of the servers don’t satisfy the desired condition.

Final result
Incoming requests:{1, 2, 3, 4, 5, 6, 7, 8}

ServerA:{1, 4, 6, 7, 8}

ServerB:{2, 5}

ServerC:{3}
```

## Algorithmic explanation
- i: This is the request number that is processed. It is initially set to zero.
- Weights: This is an array that holds the weights for all the servers.
- T: This is the total number of servers that are available.

## Algorithm
1. Begin.

2. Assign a specific number of requests to i mod Tth server, according to its numeric weight Weights[i].

3. Increment the value of i by value of 1.

4. Repeat steps 2 and 3, until there are no more requests.

5. End.

## Advantages
This load balancing technique prevents a server from receiving a large number of incoming requests.

It utilizes fewer resources.

The request assignment is deterministic and it is easy to determine the assigned server.

The network administrator can set the numeric weight for the server. This is helpful for when we want some of our server’s resources to be available for other tasks.

## Limitations
The weighted round-robin load balancing technique is not a suitable choice for when we have incoming requests with extensive service time or for when the service time of each request is different.

Real time load balancers
- Nginx
- Varnish
- HAProxy
LVS
