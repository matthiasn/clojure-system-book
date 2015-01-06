## WebSocket Communication 

The ````birdwatch.communicator```` **[namespace](https://github.com/matthiasn/BirdWatch/blob/574d2178be6f399086ad2a5ec35c200d252bf887/Clojure-Websockets/MainApp/src/cljs/birdwatch/communicator.cljs)** handles the interaction with the server-side application through the use of a WebSocket connection provided by the **[sente](https://github.com/ptaoussanis/sente)** library. Conceptually, the bi-directional WebSocket connection is similar to two **core.async** channels, one for sending and one for receiving. Since there are no different channels for different message types, all messages on the WebSocket connection need to be marked with their type. This is done by wrapping the payload in a vector where the message type is represented by a **[namespaced keyword](https://clojuredocs.org/clojure.core/keyword)** in the first position and the payload in the second position. With this convention it is really easy to pattern match using **[core.match](https://github.com/clojure/core.match)** as we will see below. It is just as easy to add new message types.

~~~
(ns birdwatch.communicator
  (:require-macros [cljs.core.match.macros :refer (match)]
                   [cljs.core.async.macros :refer [go-loop go]])
  (:require [birdwatch.channels :as c]
            [birdwatch.util :as util]
            [birdwatch.state :as state]
            [cljs.core.match]
            [taoensso.sente  :as sente  :refer (cb-success?)]
            [taoensso.sente.packers.transit :as sente-transit]
            [cljs.core.async :as async :refer [<! >! chan put! alts! timeout]]))

(enable-console-print!)

(def packer
  "Defines our packing (serialization) format for client<->server comms."
  (sente-transit/get-flexi-packer :json))

(let [{:keys [chsk ch-recv send-fn state]} (sente/make-channel-socket! "/chsk" {:packer packer :type :auto})]
  (def chsk       chsk)
  (def ch-chsk    ch-recv) ; ChannelSocket's receive channel
  (def chsk-send! send-fn) ; ChannelSocket's send API fn
  (def chsk-state state))  ; Watchable, read-only atom

(defn query-string
  "format and modify query string"
  []
  {:query_string {:default_field "text"
                  :default_operator "AND"
                  :query (str "(" (:search @state/app) ") AND lang:en")}})

(defn start-percolator
  "trigger starting of percolation matching of new tweets"
  []
  (chsk-send! [:cmd/percolate {:query (query-string)
                               :uid (:uid @chsk-state)}]))

(def prev-chunks-loaded (atom 0))

(defn load-prev
  "load previous tweets matching the current search"
  []
  (let [chunks-to-load 10
        chunk-size 500]
    (when (< @prev-chunks-loaded chunks-to-load)
      (chsk-send! [:cmd/query {:query (query-string)
                               :n chunk-size
                               :uid (:uid @chsk-state)
                               :from (* chunk-size @prev-chunks-loaded)}])
      (swap! prev-chunks-loaded inc))))

(defn start-search
  "initiate new search by starting SSE stream"
  []
  (let [search (:search-text @state/app)
        s (if (= search "") "*" search)]
    (reset! state/app (state/initial-state))
    (reset! prev-chunks-loaded 0)
    (swap! state/app assoc :search-text search)
    (swap! state/app assoc :search s)
    (aset js/window "location" "hash" (js/encodeURIComponent s))
    (start-percolator)
    (dotimes [n 4] (load-prev))))

(defn- event-handler [{:keys [event]}]
  (match event
         [:chsk/state {:first-open? true}] (do (print "Socket established!") (start-search))
         [:chsk/state new-state]           (print "Chsk state change:" new-state)
         [:chsk/recv  payload]
         (let [[msg-type msg] payload]
           (match [msg-type msg]
                  [:tweet/new             tweet] (put! c/tweets-chan tweet)
                  [:tweet/missing-tweet   tweet] (put! c/missing-tweet-found-chan tweet)
                  [:tweet/prev-chunk prev-chunk] (do (put! c/prev-chunks-chan prev-chunk)(load-prev))
                  [:stats/users-count        uc] (put! c/user-count-chan uc)
                  [:stats/total-tweet-count ttc] (put! c/total-tweets-count-chan ttc)))
         :else (print "Unmatched event: %s" event)))

(defonce chsk-router (sente/start-chsk-router! ch-chsk event-handler))

; loop for sending messages about missing tweet to server
(go-loop [] (let [tid (<! c/tweet-missing-chan)]
              (chsk-send! [:cmd/missing {:id_str tid :uid (:uid @chsk-state)}])
              (recur)))
~~~

Let's go through this function by function.

~~~
(def packer
  "Defines our packing (serialization) format for client<->server comms."
  (sente-transit/get-flexi-packer :json))
~~~

This defines the packer for sente, we're using **[transit](http://blog.cognitect.com/blog/2014/7/22/transit)** here. Next, we ````def```` a few functions that we get in the returned map from calling ````sente/make-channel-socket! ````:

~~~
(let [{:keys [chsk ch-recv send-fn state]} (sente/make-channel-socket! "/chsk" {:packer packer :type :auto})]
  (def chsk       chsk)
  (def ch-chsk    ch-recv) ; ChannelSocket's receive channel
  (def chsk-send! send-fn) ; ChannelSocket's send API fn
  (def chsk-state state))  ; Watchable, read-only atom
~~~

The ````query-string```` function is a simple helper to format a query before sending it on the wire:

~~~
(defn query-string
  "format and modify query string"
  []
  {:query_string {:default_field "text"
                  :default_operator "AND"
                  :query (str "(" (:search @state/app) ") AND lang:en")}})
~~~

The ````start-percolator```` function tell the server to start the percolation process for the web client with the current search:
~~~
(defn start-percolator
  "trigger starting of percolation matching of new tweets"
  []
  (chsk-send! [:cmd/percolate {:query (query-string)
                               :uid (:uid @chsk-state)}]))
~~~

Then, there's the ````load-prev```` function:

~~~
(def prev-chunks-loaded (atom 0))

(defn load-prev
  "load previous tweets matching the current search"
  []
  (let [chunks-to-load 10
        chunk-size 500]
    (when (< @prev-chunks-loaded chunks-to-load)
      (chsk-send! [:cmd/query {:query (query-string)
                               :n chunk-size
                               :uid (:uid @chsk-state)
                               :from (* chunk-size @prev-chunks-loaded)}])
      (swap! prev-chunks-loaded inc))))
~~~

This function takes care of loading chunks of previous tweets matching the search. The ````prev-chunks-loaded```` atom keeps track of how many have been loaded already, this is incremented each time a ````:cmd/query```` is sent to the server. If all are completed, this function won't do anything. It is initially triggered by the ````start-search```` function:

~~~
(defn start-search
  "initiate new search by starting SSE stream"
  []
  (let [search (:search-text @state/app)
        s (if (= search "") "*" search)]
    (reset! state/app (state/initial-state))
    (reset! prev-chunks-loaded 0)
    (swap! state/app assoc :search-text search)
    (swap! state/app assoc :search s)
    (aset js/window "location" "hash" (js/encodeURIComponent s))
    (start-percolator)
    (dotimes [n 4] (load-prev))))
~~~

This function resets the application state ````state/app```` to ````(state/initial-state)````, the ````prev-chunks-loaded````, then sets the search on the new application state, sets the address in the browser and finally starts the percolator and also triggers ````(load-prev)````, with four requests running in parallel. Every time one of them comes back completed from the server, ````(load-prev)```` will be called again, as we can see in the ````event-handler```` below:

~~~
(defn- event-handler [{:keys [event]}]
  (match event
         [:chsk/state {:first-open? true}] (do (print "Socket established!") (start-search))
         [:chsk/state new-state]           (print "Chsk state change:" new-state)
         [:chsk/recv  payload]
         (let [[msg-type msg] payload]
           (match [msg-type msg]
                  [:tweet/new             tweet] (put! c/tweets-chan tweet)
                  [:tweet/missing-tweet   tweet] (put! c/missing-tweet-found-chan tweet)
                  [:tweet/prev-chunk prev-chunk] (do (put! c/prev-chunks-chan prev-chunk)(load-prev))
                  [:stats/users-count        uc] (put! c/user-count-chan uc)
                  [:stats/total-tweet-count ttc] (put! c/total-tweets-count-chan ttc)))
         :else (print "Unmatched event: %s" event)))

(defonce chsk-router (sente/start-chsk-router! ch-chsk event-handler))
~~~

The ````event-handler```` function handles incoming messages from the server. Mostly, it just puts received payloads on the appropriate channels. Only in the case of a completed ````prev-chunk````, it also calls ````(load-prev)```` again, which only actually does something when there are more chunks to retrieve.

Then finally in this namespace, there's a ````go-loop```` which takes requests off the ````c/tweet-missing-chan```` channel and forwards them to the server:
~~~
; loop for sending messages about missing tweet to server
(go-loop [] (let [tid (<! c/tweet-missing-chan)]
              (chsk-send! [:cmd/missing {:id_str tid :uid (:uid @chsk-state)}])
              (recur)))
~~~
