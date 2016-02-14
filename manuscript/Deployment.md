# Deployment

In this chapter, we will have a look at potential deployment strategies, for example by running the individual parts in separate Docker containers on one or more host machines, or by deploying through an application server.

## Docker Containers

My initial excitement for Docker has somewhat cooled down -- it is certainly not the cure for all as I had erroneously assumed some time ago, and it has caused me some anger in the past few months. But Docker still has some positive sides, and there are ways to make your experience when running your application inside a Docker container smoother.

**If you think this chapter should be prioritized, send me an eMail.**


## Wildfly / Deploying a WAR file

I've recently experimented with running the BirdWatch application in a WildFly application server, with great success. There's something quite nice about using functionality from such an application server, such as content compression or TLS, with no more than simply adding some configuration, rather than having to implement all that yourself.

**If you think this chapter should be prioritized, send me an eMail.**