## Channels

Different parts of the client communicate via **core.async** channels. All of the channels are located in a single namespace, which makes it easy to import the channels namespace as the communication backbone from all other namespaces without the risk of circular dependencies. Here's the simple *[code](https://github.com/matthiasn/BirdWatch/blob/19e6265b562ec5e08ec6bb2bbe4aaadd72bd8970/Clojure-Websockets/MainApp/src/cljs/birdwatch/channels.cljs)**:

~~~
(ns birdwatch.channels
  (:require [cljs.core.async :as async :refer [chan]]))

;;; Channels for handling information flow in the application.
(def tweets-chan (chan 1))
(def tweet-missing-chan (chan))
(def missing-tweet-found-chan (chan))
(def prev-tweets-chan (chan 10000))
(def user-count-chan (chan))
(def total-tweets-count-chan (chan))
(def prev-chunks-chan (chan))
(def missing-tweets-chan (chan))
~~~

A few channels are created here, which are then used as their names indicate.