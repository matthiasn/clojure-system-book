## Channels

Different parts of the client communicate via **core.async** channels.

https://github.com/matthiasn/BirdWatch/blob/574d2178be6f399086ad2a5ec35c200d252bf887/Clojure-Websockets/MainApp/src/cljs/birdwatch/channels.cljs

~~~
(ns birdwatch.channels
  (:require [cljs.core.async :as async :refer [<! >! chan put! alts! timeout]]))

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
