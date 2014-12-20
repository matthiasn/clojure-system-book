# Operational Insight

Before we can start optimizing for just about anything, we will need operational insight. We will need a dashboard that shows (near-)realtime data about how all processes participating in the system are running. Luckily, the basic infrastructure we would need for such an effort is already in place in the main application. We are already dealing with a web interface that receives live data over websockets and that draws charts. On the server side, we have Redis, which could just as well collect metrics from all JVMs in the system and distribute the data to a service that processes this data for consumption by a dashboard web application.

My idea is to build this dashboard without relying on any external services such as StatsD/Graphite, Riemann and such. This is not to say that these tools wouldn't help, but it is an interesting problem that I believe can be solved without having to learn about additional tools.
