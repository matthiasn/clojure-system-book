# Building a System in #Clojure 1 - Overview

Blog post originally published on **[matthiasnehlsen.com](http://matthiasnehlsen.com/blog/2014/09/24/Building-Systems-in-Clojure-1/)**. The content will be adapted to fit the book format better.


This is the first of **n** articles about building **systems** in **[Clojure](http://clojure.org/)**. Clojure is a beautiful language and I have been fascinated from the first moment I laid eyes on it last summer. However, what remained a mystery to me for most of the time was how to build more complex systems. I started researching the options that would allow me to structure an arbitrarily complex application in a way that is **easy to understand and maintain**. Here is what I found.

As an example for such a system, we will be looking at the Clojure rewrite of an application I wrote last year: **[BirdWatch](https://github.com/matthiasn/BirdWatch)**. This application subscribes to the **[Twitter Streaming API](https://dev.twitter.com/docs/streaming-apis)** for all tweets that contain one or more terms out of a set of terms and makes the tweets searchable through storing them in ElasticSearch. A live version of the Clojure version of this application is available here: **[http://birdwatch2.matthiasnehlsen.com](http://birdwatch2.matthiasnehlsen.com/#*)**.

In this first installment we will be looking at the basic architecture of the server side. Let's start with an animation [^1] to demonstrate how components in the system get wired up when the application initializes before we go into details.

![channels overview](images/bw-channels.png)

The architecture above is a huge improvement over the first version and was only possible thanks to Stuart Sierra's **[component library](https://github.com/stuartsierra/component)**. This new version has cleanly separated components with no dependencies between namespace at all (except, of course, in the main namespace that wires everything together). But the individual components don't know anything about each other except for where the components in the animation touch each other. And even there, it is mostly just plain **[core.async](https://github.com/clojure/core.async)** channels.

In the initial version that I wrote, where everything depended on everything, things were very different. Some people would call that "spaghetti code", but I think that is not doing justice to spaghetti. Unlike bad code, I don't mind touching spaghetti. I would rather liken bad code to hairballs, of the worst kind that is. Have you ever experienced the following: you are standing in the shower and the water doesn't drain. You notice something in the sink, so you squat down to pull it out only to start screaming, "Oh my god, it's a dead rat" a second later. I am referring to that kind of entangled hairball mess, nothing less. On top, you may even hit your head when you jump up in disgust. 

This is where dependency injection comes in. Can we agree that we don't like hairballs? Good. Usually, what we are trying to achieve is a so-called inversion of control, in which a component of the application knows that it will be injected something which implements a known interface at runtime. Then, no matter what the actual implementation is, it knows what methods it can call on that something because of the implemented interface.

Here, unlike in object-oriented dependency injection, things are a little different because we don't really have objects. The components play the role of objects, but as a further way of decoupling, I wanted them to only communicate via **core.async** channels. Channels are a great abstraction. Rich Hickey likens them to conveyor belts onto which you put something without having to know at all what happens on the other side. We will have a more detailed look at the channels in the next article. For now, as an abstraction, we can think about the channel components (the flat ones connecting the components with the switchboard) as **wiring harnesses**, like the one that connects the electronics of your car to your engine. The only way to interface with a modern engine (that doesn't have separate mechanical controls) is by connecting to this wiring harness and either send or receive information, depending on the channel / cable that you interface with.

Let's have a look at how the initialization of the application we have already seen in the animation looks in code:

https://github.com/matthiasn/BirdWatch/blob/a26c201d2cc2c89f4b3d2ecb8e6adb403e6f89c7/Clojure-Websockets/src/clj/birdwatch/main.clj main.clj

{line-numbers=off,lang=clojure}
~~~
(ns birdwatch.main
  (:gen-class)
  (:require
   [birdwatch.twitter-client :as tc]
   [birdwatch.communicator :as comm]
   [birdwatch.persistence :as p]
   [birdwatch.percolator :as perc]
   [birdwatch.http :as http]
   [birdwatch.switchboard :as sw]
   [clojure.edn :as edn]
   [clojure.tools.logging :as log]
   [clj-pid.core :as pid]
   [com.stuartsierra.component :as component]))

(def conf (edn/read-string (slurp "twitterconf.edn")))

(defn get-system [conf]
  "Create system by wiring individual components so that component/start
  will bring up the individual components in the correct order."
  (component/system-map
   :communicator-channels (comm/new-communicator-channels)
   :communicator  (component/using (comm/new-communicator) {:channels :communicator-channels})
   :twitterclient-channels (tc/new-twitterclient-channels)
   :twitterclient (component/using (tc/new-twitterclient conf) {:channels :twitterclient-channels})
   :persistence-channels (p/new-persistence-channels)
   :persistence   (component/using (p/new-persistence conf) {:channels :persistence-channels})
   :percolation-channels (perc/new-percolation-channels)
   :percolator    (component/using (perc/new-percolator conf) {:channels :percolation-channels})
   :http          (component/using (http/new-http-server conf) {:communicator :communicator})
   :switchboard   (component/using (sw/new-switchboard) {:comm-chans :communicator-channels
                                                         :tc-chans :twitterclient-channels
                                                         :pers-chans :persistence-channels
                                                         :perc-chans :percolation-channels})))
(def system (get-system conf))

(defn -main [& args]
  (pid/save (:pidfile-name conf))
  (pid/delete-on-shutdown! (:pidfile-name conf))
  (log/info "Application started, PID" (pid/current))
  (alter-var-root #'system component/start))
~~~

I personally think this **reads really well**, even if you have never seen Clojure before in your life. Roughly the first half is concerned with imports and reading the configuration file. Next, we have the ````get-system```` function which declares, what components depend on what other components. The system is finally started in the ````-main```` function (plus the process ID logged and saved to a file). This is all there is to know about the application entry point. 

Now, when we start the application, all the dependencies will be started in an order that the component library determines so that all dependencies are met. Here's the output of that startup process:

{line-numbers=off,lang=text}
~~~
mn:Clojure-Websockets mn$ lein run
16:46:30.925 [main] INFO  birdwatch.main - Application started, PID 6682
16:46:30.937 [main] INFO  birdwatch.twitter-client - Starting Twitterclient Channels Component
16:46:30.939 [main] INFO  birdwatch.twitter-client - Starting Twitterclient Component
16:46:30.940 [main] INFO  birdwatch.twitter-client - Starting Twitter client.
16:46:31.323 [main] INFO  birdwatch.persistence - Starting Persistence Channels Component
16:46:31.324 [main] INFO  birdwatch.persistence - Starting Persistence Component
16:46:31.415 [main] INFO  org.elasticsearch.plugins - [Chameleon] loaded [], sites []
16:46:32.339 [main] INFO  birdwatch.communicator - Starting Communicator Channels Component
16:46:32.340 [main] INFO  birdwatch.communicator - Starting Communicator Component
16:46:32.355 [main] INFO  birdwatch.http - Starting HTTP Component
16:46:32.375 [main] INFO  birdwatch.http - Http-kit server is running at http://localhost:8888/
16:46:32.376 [main] INFO  birdwatch.percolator - Starting Percolation Channels Component
16:46:32.377 [main] INFO  birdwatch.percolator - Starting Percolator Component
16:46:32.380 [main] INFO  birdwatch.switchboard - Starting Switchboard Component
~~~

Next week, we will look at how these components wire a channel grid and how information flows through this grid. You do not have to wait to see more code though, everything is on **[GitHub](https://github.com/matthiasn/BirdWatch)** already.

Once we have discussed the architecture in detail over the next couple of weeks, we can start observing the system under load. Of course, it would be interesting to have actual user load. But with or without actual load, we want to find a way of how to generate / simulate load and then observe the system, identify the bottlenecks and remove them. For example, the clients could be simulated by connecting a load generator via ZeroMQ or the like and deliver matches back to that application and check if they are as expected (correct, complete, timely). The Twitter stream could also be simulated, for example by connecting to a load generator that either replays recorded tweets, with full control over the rate, or with artificial test cases, for which we could exactly specify the expectations on the output side.

That's it for now. Would you like to be informed when the next article is out? Just **sign up** for the <a href="http://eepurl.com/y0HWv" target="_blank"><strong>mailing list</strong></a> and I will let you know. Also, if you are interested in Clojure, you may want to check out my curated list of **[useful Clojure-related resources on GitHub](https://github.com/matthiasn/Clojure-Resources)**.

Cheers,
Matthias