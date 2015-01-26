## Application State - outdated

In the last chapter, we have seen how different parts of the application are wired together. Since they don't depend on each other but only on the channels, it doesn't really matter in which order we discuss them, so why .

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
