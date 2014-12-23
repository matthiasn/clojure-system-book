## WebSocket Communication Component



https://github.com/matthiasn/BirdWatch/blob/3c793a8ded198ba9aa2360f1efb538dd548383b2/Clojure-Websockets/MainApp/src/clj/birdwatch/communicator/component.clj

~~~
(ns birdwatch.communicator.component
  (:gen-class)
  (:require
   [clojure.pprint :as pp]
   [clojure.tools.logging :as log]
   [birdwatch.communicator.websockets :as ws]
   [taoensso.sente :as sente]
   [taoensso.sente.packers.transit :as sente-transit]
   [com.stuartsierra.component :as component]
   [clojure.core.async :as async :refer [chan]]))

(def packer (sente-transit/get-flexi-packer :json)) ;; serialization format for client<->server comm

(defrecord Communicator [channels chsk-router]
  component/Lifecycle
  (start [component] (log/info "Starting Communicator Component")
         (let [{:keys [ch-recv send-fn ajax-post-fn ajax-get-or-ws-handshake-fn connected-uids]}
               (sente/make-channel-socket! {:packer packer :user-id-fn ws/user-id-fn})
               event-handler (ws/make-handler (:query channels) (:tweet-missing channels) (:register-perc channels))
               chsk-router (sente/start-chsk-router! ch-recv event-handler)]
           (ws/run-users-count-loop send-fn connected-uids)
           (ws/send-loop (:perc-matches channels) (ws/perc-matches connected-uids send-fn))
           (ws/send-loop (:tweet-count channels) (ws/tweet-stats connected-uids send-fn))
           (ws/send-loop (:query-results channels) (ws/relay-msg :tweet/prev-chunk :result send-fn))
           (ws/send-loop (:missing-tweet-found channels) (ws/relay-msg :tweet/missing-tweet :tweet send-fn))
           (assoc component :ajax-post-fn ajax-post-fn
                            :ajax-get-or-ws-handshake-fn ajax-get-or-ws-handshake-fn
                            :chsk-router chsk-router)))
  (stop [component] (log/info "Stopping Communicator Component")
        (chsk-router) ;; stops router loop
        (assoc component :chsk-router nil)))

(defn new-communicator [] (map->Communicator {}))

(defrecord Communicator-Channels []
  component/Lifecycle
  (start [component] (log/info "Starting Communicator Channels Component")
         (assoc component
           :query (chan)
           :query-results (chan)
           :tweet-missing (chan)
           :missing-tweet-found (chan)
           :persistence (chan)
           :rt-persistence (chan)
           :tweet-count (chan)
           :register-perc (chan)
           :perc-matches (chan)))
  (stop [component] (log/info "Stop Communicator Channels Component")
        (assoc component :query nil :query-results nil :tweet-missing nil :missing-tweet-found nil
                         :persistence nil :rt-persistence nil :tweet-count nil)))

(defn new-communicator-channels [] (map->Communicator-Channels {}))
~~~


https://github.com/matthiasn/BirdWatch/blob/3c793a8ded198ba9aa2360f1efb538dd548383b2/Clojure-Websockets/MainApp/src/clj/birdwatch/communicator/websockets.clj

~~~
(ns birdwatch.communicator.websockets
  (:gen-class)
  (:require
   [clojure.core.match :as match :refer (match)]
   [clojure.pprint :as pp]
   [clojure.tools.logging :as log]
   [taoensso.sente :as sente]
   [com.matthiasnehlsen.inspect :as inspect :refer [inspect]]
   [clojure.core.async :as async :refer [<! >! put! timeout go-loop]]))

(defn user-id-fn
  "generates unique ID for request"
  [req]
  (let [uid (str (java.util.UUID/randomUUID))]
    (log/info "Connected:" (:remote-addr req) uid)
    uid))

(defn make-handler
  "create event handler function for the websocket connection"
  [query-chan tweet-missing-chan register-percolation-chan]
  (fn [{event :event}]
    (match event
           [:cmd/percolate params] (put! register-percolation-chan params)
           [:cmd/query params]     (put! query-chan params)
           [:cmd/missing params]   (put! tweet-missing-chan params)
           [:chsk/ws-ping]         () ; currently just do nothing with ping (no logging either)
           :else                   (log/debug "Unmatched event:" (pp/pprint event)))))

(defn send-loop
  "run loop, call f with message on channel"
  [channel f]
  (go-loop [] (let [msg (<! channel)] (f msg)) (recur)))

(defn tweet-stats
  "send stats about number of indexed tweets to all connected clients"
  [uids chsk-send!]
  (fn [msg] (doseq [uid (:any @uids)]
              (chsk-send! uid [:stats/total-tweet-count msg]))))

(defn perc-matches
  "deliver percolation matches to interested clients"
  [uids chsk-send!]
  (fn [msg]
    (inspect :comm/perc-matches msg)
    (let [[t matches subscriptions] msg]
      (doseq [uid (:any @uids)]
        (when (contains? matches (get subscriptions uid))
          (chsk-send! uid [:tweet/new t]))))))

(defn relay-msg
  "send query result chunks back to client"
  [msg-type msg-key chsk-send!]
  (fn [msg] (chsk-send! (:uid msg) [msg-type (msg-key msg)])))

(defn run-users-count-loop
  "runs loop for sending stats about number of connected users to all connected clients"
  [chsk-send! connected-uids]
  (go-loop [] (<! (timeout 2000))
           (let [uids (:any @connected-uids)]
             (inspect :comm/connected-uids uids)
             (doseq [uid uids] (chsk-send! uid [:stats/users-count (count uids)])))
           (recur)))
~~~


