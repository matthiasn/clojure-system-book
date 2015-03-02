# Monitoring
There is only one way to get to know a system: observation. We will want to observe how a system responds under different circumstances. For that, we will need data from instrumenting the system and then process the data and make it more digestible for human consumption. For example, what kind of load can my system handle until the response times become unacceptable? Will the system recover after short bursts or will it just die, e.g. with an **Out of Memory Error**? Is there a network bottle neck? Is the processing inside the application IO-bound or CPU-bound? 

All these insights will make us more comfortable when the system is deployed. The monitoring should be in place all the time, both in production and when doing load tests. Then, we could for example detect unhealthy patterns and proactively fire up addition instances on **AWS** or whatever before things get ugly. Or we could monitor the behavior of the JVM (e.g. garbage collection pauses) and optimize the settings to make the system hosted on it run as smoothly as possible. The JVM is really only another system that interacts with the systems running on it and that can be observed and optimized.

I recently discovered the **[metrics library](https://github.com/dropwizard/metrics)** from **[@coda](https://twitter.com/coda)** by watching his **[talk](http://pivotallabs.com/139-metrics-metrics-everywhere/)** about the library. It contains a lot of the functionality that we could want for observing a system, such as **Meters**, **Gauges**, **Counters** and **Histograms**. We will look at all these in detail and see how we can possibly use them in the context of our architecture that relies heavily on **core.async** for separating different parts of the system.

Let's get started. 

## Inspect
