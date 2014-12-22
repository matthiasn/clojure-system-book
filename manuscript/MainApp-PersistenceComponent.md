## MainApp - Persistence Component

This component takes care of retrieving tweets from an index in ElasticSearch so that they can be delivered to the connected web client.

https://github.com/matthiasn/BirdWatch/blob/d104db4a7ac7a745593e34398751f81a50d167d0/Clojure-Websockets/MainApp/src/clj/birdwatch/persistence/component.clj

~~~
(ns birdwatch.persistence.component
  (:gen-class)
  (:require
   [birdwatch.persistence.elastic :as es]
   [clojure.tools.logging :as log]
   [clojure.pprint :as pp]
   [clojurewerkz.elastisch.rest :as esr]
   [com.stuartsierra.component :as component]
   [clojure.core.async :as async :refer [<! chan go-loop tap pipeline-blocking]]))

(defrecord Persistence [conf channels]
  component/Lifecycle
  (start [component]
         (log/info "Starting Persistence Component")
         (let [conn (esr/connect (:es-address conf))
               q-chan (:query channels)
               q-res-chan (:query-results channels)
               mt-chan (:tweet-missing channels)
               mt-res-chan (:missing-tweet-found channels)]
           (pipeline-blocking 4 q-res-chan (es/query-xf conf conn) q-chan)
           (pipeline-blocking 2 mt-res-chan (es/tweet-query-xf conf conn) mt-chan)
           (es/run-tweet-count-loop (:tweet-count channels) conf conn)
           (assoc component :conn conn)))
  (stop [component] ;; TODO: proper teardown of resources
        (log/info "Stopping Persistence Component")
        (assoc component :conn nil)))

(defn new-persistence [conf] (map->Persistence {:conf conf}))

(defrecord Persistence-Channels []
  component/Lifecycle
  (start [component] (log/info "Starting Persistence Channels Component")
         (assoc component
           :query (chan)
           :query-results (chan)
           :tweet-missing (chan)
           :missing-tweet-found (chan)
           :tweet-count (chan)))
  (stop [component] (log/info "Stop Persistence Channels Component")
        (assoc component :query nil :query-results nil :tweet-missing nil :missing-tweet-found nil
                         :tweet-count nil)))

(defn new-persistence-channels [] (map->Persistence-Channels {}))
~~~


https://github.com/matthiasn/BirdWatch/blob/d104db4a7ac7a745593e34398751f81a50d167d0/Clojure-Websockets/MainApp/src/clj/birdwatch/persistence/elastic.clj

~~~
(ns birdwatch.persistence.elastic
  (:gen-class)
  (:require
   [birdwatch.persistence.tools :as pt]
   [com.matthiasnehlsen.inspect :as inspect :refer [inspect]]
   [clojure.tools.logging :as log]
   [clojure.pprint :as pp]
   [clojurewerkz.elastisch.rest :as esr]
   [clojurewerkz.elastisch.rest.document :as esd]
   [clojurewerkz.elastisch.query :as q]
   [clojurewerkz.elastisch.rest.response :as esrsp]
   [clojure.core.async :as async :refer [<! <!! chan put! timeout go-loop thread onto-chan]]))

(defn query
  "run a query on previous matching tweets"
  [{:keys [query n from]} conf conn]
  (let [search (esd/search conn (:es-index conf) "tweet" :query query :size n :from from :sort {:id :desc})
        hits (esrsp/hits-from search)
        source (pt/get-source hits)
        res (vec source)]
    (inspect :elastic/query-res {:query query
                                 :total-hits (esrsp/total-hits search)
                                 :retrieved (count hits)
                                 :first-hit (first hits)
                                 :chars (count (str res))})
    res))

(defn query-xf
  "create transducer for answering queries"
  [conf conn]
  (map (fn [q]
         (inspect :elastic/query q)
         {:uid (:uid q) :result (query q conf conn)})))

(defn tweet-query-xf
  "create transducer for finding missing tweets"
  [conf conn]
  (map (fn [req]
         (let [res (esd/get conn (:es-index conf) "tweet" (:id_str req))]
           (inspect :elastic/missing {:req req :res res})
           (when-not res (log/debug "birdwatch.persistence missing" (:id_str req) res))
           {:tweet (pt/strip-source res) :uid (:uid req)}))))

(defn run-tweet-count-loop
  "run loop for sending stats about total number of indexed tweets"
  [tweet-count-chan conf conn]
  (go-loop [] (<! (timeout 10000))
           (let [cnt (esd/count conn (:es-index conf) "tweet")]
             (inspect :elastic/tweet-count cnt)
             (put! tweet-count-chan (format "%,15d" (:count cnt))))
           (recur)))
~~~



https://github.com/matthiasn/BirdWatch/blob/d104db4a7ac7a745593e34398751f81a50d167d0/Clojure-Websockets/MainApp/src/clj/birdwatch/persistence/tools.clj

~~~
(ns birdwatch.persistence.tools
  (:gen-class))

(defn- strip-tweet
  "take only actually needed fields from tweet"
  [t]
  (let [u (:user t)]
    {:id_str (:id_str t)
     :id (:id t)
     :text (:text t)
     :created_at (:created_at t)
     :retweet_count (:retweet_count t)
     :favorite_count (:favorite_count t)
     :entities (:entities t)
     :user {:followers_count (:followers_count u)
            :name (:name u)
            :profile_image_url (:profile_image_url u)
            :screen_name (:screen_name u)}}))

(defn strip-source
  "get tweet stripped down to necessary fields"
  [val]
  (let [s (:_source val)
        t (strip-tweet s)
        rt (:retweeted_status s)]
    (if rt
      (assoc t :retweeted_status (strip-tweet rt))
      t)))

(defn get-source
  "get vector with :_source of each ElasticSearch result"
  [coll]
  (map strip-source coll))
~~~

