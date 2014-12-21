# Load Testing and Optimization

In this part of the book, we shall determine performance baselines, optimize obvious performance bottlenecks and implement a load testing strategy so that we obtain a good understanding about how the system performs under load. Once we are able to generate load, we can also have a look at optimizing the environment, for example for reasonable garbage collection behavior in which the system does not become unnecessarily unresponsive.


## Performance and Load Characteristics

Of course, it would be interesting to have actual user load. But with or without actual load, we want to find a way of how to generate / simulate load and then observe the system, identify the bottlenecks and remove them. For example, the clients could be simulated by connecting a load generator via Redis and deliver matches back to that application and check if they are as expected (correct, complete, timely). The Twitter stream could also be simulated, for example by connecting to a load generator that either replays recorded tweets, with full control over the rate, or with artificial test cases, for which we could exactly specify the expectations on the output side.


## Optimization and Environment
