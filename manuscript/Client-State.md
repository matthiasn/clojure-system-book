## Application State - outdated

![](images/client-state.png)

The application state of the application is held inside the _let-binding_ of the ````init-state```` function within the ````birdwatch.state.data```` **[namespace](https://github.com/matthiasn/BirdWatch/blob/c10fd4ecf7e2d763a5f6476fb4be6605d51123e7/Clojure-Websockets/MainApp/src/cljs/birdwatch/state/data.cljs)**: 

~~~
(ns birdwatch.state.data
  (:require [birdwatch.state.initial :as i]
            [birdwatch.state.comm :as c]))

(defn init-state
  "Init app state and wire all channels required in the state namespace. The app
   atom is held inside the let binding of this function and thus protected from
   outside access / alteration. The only way to interact with it is by sending
   messages on channels, such the provided data channel for adding new data or
   sending commands on the cmd-chan."
  [data-chan qry-chan stats-chan cmd-chan state-pub-chan]
  (let [app (atom {})]
    (i/init app)
    (c/stats-loop stats-chan app)
    (c/data-loop data-chan qry-chan app)
    (c/cmd-loop cmd-chan state-pub-chan qry-chan app)
    (c/broadcast-state state-pub-chan app)))
~~~

This function first of all keeps an atom named ````app```` in a let-binding. Before instantiating the "business logic", the state needs to be initialized. This is done by calling the ````init```` function inside the ````birdwatch.state.initial```` **[namespace](https://github.com/matthiasn/BirdWatch/blob/c10fd4ecf7e2d763a5f6476fb4be6605d51123e7/Clojure-Websockets/MainApp/src/cljs/birdwatch/state/initial.cljs)**:

~~~
(ns birdwatch.state.initial
  (:require [birdwatch.util :as util]
            [tailrecursion.priority-map :refer [priority-map-by]]))

(defn initial-state
  "Returns fresh application state."
  []
  {:count 0
   :n 10
   :prev-chunks-loaded 0
   :tweets-map {}
   :search-text ""
   :page 1
   :search "*"
   :users-count 0
   :total-tweet-count 0
   :sorted :by-id
   :live true
   :by-followers (priority-map-by >)
   :by-retweets (priority-map-by >)
   :by-favorites (priority-map-by >)
   :by-rt-since-startup (priority-map-by >)
   :by-reach (priority-map-by >)
   :by-id (priority-map-by >)
   :words-sorted-by-count (priority-map-by >)})

(defn init
  "Initializes application start when application starts by providing fresh state
   and setting the :search-text from the URI location hash."
  [app]
  (reset! app (initial-state))
  (swap! app assoc :search-text (util/search-hash)))
~~~

This ````init```` function only takes the ````app```` atom passed as its only paramater and resets it with the result of a call to the ````initial-state```` function. Next, it also swaps the ````:search-text```` key of the application state with a call to ````util/search-hash````. This function gets the URL hash, which is also set whenever a new search is started. This allows bookmarking of specific searches. Next time a bookmark is open, the same search will load as last time.

With the application state properly initialized, we can now fire up the "business logic" of the application which is realized as a number of functions in the ````birdwatch.state.comm```` namespace. These functions start up behavior such as taking messages off channels and reacting according to the received message and adding a listener to state changes that are then put on a channel for broadcasting on a ````pub````.

Let's have a look at the ````birdwatch.state.comm```` **[namespace](https://github.com/matthiasn/BirdWatch/blob/4f654a0ce326bd85b331390bdcc205ca19fd82fd/Clojure-Websockets/MainApp/src/cljs/birdwatch/state/comm.cljs)** in detail in order to see what gets initialized in the body of the ````init-state```` function:

~~~
(ns birdwatch.state.comm
  (:require-macros [cljs.core.async.macros :refer [go-loop]])
  (:require [birdwatch.state.search :as s]
            [birdwatch.state.initial :as i]
            [birdwatch.state.proc :as p]
            [cljs.core.async :as async :refer [<! put! pipe timeout chan sliding-buffer]]
            [cljs.core.match :refer-macros [match]]))

;;;; Channels processing namespace. Here, messages are taken from channels and processed.

(defn- stats-loop
  "Process messages from the stats channel and update application state accordingly."
  [stats-chan app]
  (go-loop []
           (let [msg (<! stats-chan)]
             (match msg
                    [:stats/users-count       n] (swap! app assoc :users-count n)
                    [:stats/total-tweet-count n] (swap! app assoc :total-tweet-count n)
                    :else (prn "unknown msg in stats-loop" msg))
             (recur))))

(defn- prev-chunks-loop
  "Take messages (vectors of tweets) from prev-chunks-chan, add each tweet to application
   state, then pause to give the event loop back to the application (otherwise, UI becomes
   unresponsive for a short while)."
  [prev-chunks-chan app]
  (go-loop []
           (let [chunk (<! prev-chunks-chan)]
             (doseq [t chunk] (p/add-tweet! t app))
             (<! (timeout 50))
             (recur))))

(defn- data-loop
  "Process messages from the data channel and process / add to application state.
   In the case of :tweet/prev-chunk messages: put! on separate channel individual items
   are handled with a lower priority."
  [data-chan qry-chan app]
  (let [prev-chunks-chan (chan)]
    (prev-chunks-loop prev-chunks-chan app)
    (go-loop []
             (let [msg (<! data-chan)]
               (match msg
                      [:tweet/new             tweet] (p/add-tweet! tweet app)
                      [:tweet/missing-tweet   tweet] (p/add-to-tweets-map! app :tweets-map tweet)
                      [:tweet/prev-chunk prev-chunk] (do
                                                       (put! prev-chunks-chan prev-chunk)
                                                       (s/load-prev app qry-chan))
                      :else (prn "unknown msg in data-loop" msg))
               (recur)))))

(defn- cmd-loop
  "Process command messages, e.g. those that alter application state."
  [cmd-chan pub-chan qry-chan app]
  (go-loop []
           (let [msg (<! cmd-chan)]
             (match msg
                    [:toggle-live            ] (swap! app update :live not)
                    [:set-search-text    text] (swap! app assoc :search-text text)
                    [:set-current-page   page] (swap! app assoc :page page)
                    [:set-page-size         n] (swap! app assoc :n n)
                    [:start-search           ] (s/start-search app (i/initial-state) qry-chan)
                    [:set-sort-order by-order] (swap! app assoc :sorted by-order)
                    [:retrieve-missing id-str] (put! qry-chan [:cmd/missing {:id_str id-str}])
                    [:append-search-text text] (s/append-search-text text app)
                    :else (prn "unknown msg in cmd-loop" msg))
             (recur))))

(defn- broadcast-state
  "Broadcast state changes on the specified channel. Internally uses a sliding
   buffer of size one in order to not overwhelm the rest of the system with too
   frequent updates. The only one that matters next is the latest state anyway.
   It doesn't harm to drop older ones on the channel."
  [pub-channel app]
  (let [sliding-chan (chan (sliding-buffer 1))]
    (pipe sliding-chan pub-channel)
    (add-watch app :watcher
               (fn [_ _ _ new-state]
                 (put! sliding-chan [:app-state new-state])))))
~~~

Let's go through this namespace function by function. First, we have the ````stats-loop````:

~~~
(defn- stats-loop
  "Process messages from the stats channel and update application state accordingly."
  [stats-chan app]
  (go-loop []
           (let [msg (<! stats-chan)]
             (match msg
                    [:stats/users-count       n] (swap! app assoc :users-count n)
                    [:stats/total-tweet-count n] (swap! app assoc :total-tweet-count n)
                    :else (prn "unknown msg in stats-loop" msg))
             (recur))))
~~~

This function starts up an infinitely running ````go-loop```` that takes messages off the ````stats-chan````, and then matches the messages against the two following patterns using **[core.match](https://github.com/clojure/core.match)**. When the message matches one of the two patterns, the application state is updated to reflect the data coming from the server. If the message doesn't match, a warning is printed to the browser console.

Next, let's have a look at the ````data-loop```` function:

~~~
(defn- data-loop
  "Process messages from the data channel and process / add to application state.
   In the case of :tweet/prev-chunk messages: put! on separate channel individual items
   are handled with a lower priority."
  [data-chan qry-chan app]
  (let [prev-chunks-chan (chan)]
    (prev-chunks-loop prev-chunks-chan app)
    (go-loop []
             (let [msg (<! data-chan)]
               (match msg
                      [:tweet/new             tweet] (p/add-tweet! tweet app)
                      [:tweet/missing-tweet   tweet] (p/add-to-tweets-map! app :tweets-map tweet)
                      [:tweet/prev-chunk prev-chunk] (do
                                                       (put! prev-chunks-chan prev-chunk)
                                                       (s/load-prev app qry-chan))
                      :else (prn "unknown msg in data-loop" msg))
               (recur)))))
~~~

This function follows the same pattern we have already seen with the ````stats-loop```` function, only that there are more patterns to match on. Also, the messages do not contain stats but tweet data. In the case that a new tweet is received, which is detected by the ````:tweet/new```` keyword in the first position of the message vector, the ````add-tweet!```` function from the ````birdwatch.state.proc```` namespace is called with the payload. We will look at the mechanisms in that namespace later. When a missing tweet is encountered, the ````add-to-tweets-map!```` function from the same namespace is called. Finally, when a ````:tweet/prev-chunk```` message is encountered, two functions are called. First of all, the ````prev-chunk```` is put on a channel for processing these chunks. We'll look at that next. Then, also the ````load-prev```` function from the ````birdwatch.state.search```` namespace is called. We'll have a look at that in detail later. As a short description for now, a number of previous chunks are loaded, currently with 500 tweets each, and in order to not flood the server with too many queries at the same time, subsequent queries are only fired when another chunk has been retrieved. Next, let's have a look at the ````prev-chunks-loop```` function which processes chunks of previous tweets as mentioned above:

~~~
(defn- prev-chunks-loop
  "Take messages (vectors of tweets) from prev-chunks-chan, add each tweet to application
   state, then pause to give the event loop back to the application (otherwise, UI becomes
   unresponsive for a short while)."
  [prev-chunks-chan app]
  (go-loop []
           (let [chunk (<! prev-chunks-chan)]
             (doseq [t chunk] (p/add-tweet! t app))
             (<! (timeout 50))
             (recur))))
~~~

Here in this ````go-loop````, chunks are taken off the ````prev-chunks-chan```` and then every tweet in this chunk is added to the application state, in a similar fashion to what we've seen previously for messages of type ````:tweet/new```` by calling the ````add-tweet!```` function in the ````birdwatch.state.proc```` namespace. Then, after each chunk, ````(<! (timeout 50))```` is used. This is done to give control back to the JavaScript event loop instead of blocking until the ````prev-chunks-chan```` is empty. Without this, the UI became unresponsive until all previous tweets were loaded.

Next, we have the ````cmd-loop```` function, its purpose is to take command messages off the ````cmd-chan```` and process them as required:
~~~
(defn- cmd-loop
  "Process command messages, e.g. those that alter application state."
  [cmd-chan pub-chan qry-chan app]
  (go-loop []
           (let [msg (<! cmd-chan)]
             (match msg
                    [:toggle-live            ] (swap! app update :live not)
                    [:set-search-text    text] (swap! app assoc :search-text text)
                    [:set-current-page   page] (swap! app assoc :page page)
                    [:set-page-size         n] (swap! app assoc :n n)
                    [:start-search           ] (s/start-search app (i/initial-state) qry-chan)
                    [:set-sort-order by-order] (swap! app assoc :sorted by-order)
                    [:retrieve-missing id-str] (put! qry-chan [:cmd/missing {:id_str id-str}])
                    [:append-search-text text] (s/append-search-text text app)
                    :else (prn "unknown msg in cmd-loop" msg))
             (recur))))
~~~

The mechanism at play in the ````cmd-loop```` function above should be familiar to you by now. 


~~~
(defn- broadcast-state
  "Broadcast state changes on the specified channel. Internally uses a sliding
   buffer of size one in order to not overwhelm the rest of the system with too
   frequent updates. The only one that matters next is the latest state anyway.
   It doesn't harm to drop older ones on the channel."
  [pub-channel app]
  (let [sliding-chan (chan (sliding-buffer 1))]
    (pipe sliding-chan pub-channel)
    (add-watch app :watcher
               (fn [_ _ _ new-state]
                 (put! sliding-chan [:app-state new-state])))))
~~~



**From here on: Outdated**
The entire application state is held in atoms, mostly in one large map, and stored in one namespace that all other namespaces in the application can import. Here's the **[code](https://github.com/matthiasn/BirdWatch/blob/574d2178be6f399086ad2a5ec35c200d252bf887/Clojure-Websockets/MainApp/src/cljs/birdwatch/state.cljs)**:

~~~
(ns birdwatch.state
  (:require-macros [cljs.core.async.macros :refer [go-loop go alt!]])
  (:require [birdwatch.channels :as c]
            [tailrecursion.priority-map :refer [priority-map-by]]
            [cljs.core.async :as async :refer [<!]]
            [reagent.core :as r :refer [atom]]))

;;; Application state in a single atom
;;; Will be initialized with the map returned by util/initial-state.
;;; Reset to a new clean slate when a new search is started.
(def app (atom {}))

(defn initial-state
  "function returning fresh application state"
  []
  {:count 0
   :n 10
   :tweets-map {}
   :search-text ""
   :page 1
   :search "*"
   :users-count 0
   :total-tweet-count 0
   :sorted :by-rt-since-startup
   :by-followers (priority-map-by >)
   :by-retweets (priority-map-by >)
   :by-favorites (priority-map-by >)
   :by-rt-since-startup (priority-map-by >)
   :by-reach (priority-map-by >)
   :by-id (priority-map-by >)
   :words-sorted-by-count (priority-map-by >)})

(go-loop []
         (let [uc (<! c/user-count-chan)]
           (swap! app assoc :users-count uc)
           (recur)))

(go-loop []
         (let [ttc (<! c/total-tweets-count-chan)]
           (swap! app assoc :total-tweet-count ttc)
           (recur)))

(defn append-search-text [s]
  (swap! app assoc :search-text (str (:search-text @app) " " s)))
~~~

The ````app```` atom holds the application state. It is initially empty. On initialization of the application the empty map is replaced by the "clean slate" application state returned by the ````initial-state```` function. This function is also used when the application state is reset when starting a new search without reloading the page. 

Then, there are two ````go-loops```` - one takes values off of ````c/user-count-chan```` and updates the ````:users-count```` key in the application state and the other does the same for ````c/total-tweets-count-chan```` and updates the ````:total-tweet-count```` key.

Finally, the ````append-search-text```` function appends strings to the ````:search-text```` key, used for example when clicking on entries in the word cloud in order to add words to the search input field.
