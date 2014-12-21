## Application Initialization

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
