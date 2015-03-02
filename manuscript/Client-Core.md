## Core Namespace

This namespace initializes the application on the client side. Here's the entire **[code](https://github.com/matthiasn/BirdWatch/blob/d88d191ccad1c921a7fa318ef23e10508d725182/Clojure-Websockets/MainApp/src/cljs/birdwatch/core.cljs)**, with the explanation below.

~~~
(ns birdwatch.core
  (:require [birdwatch.util :as util]
            [birdwatch.charts.ts-chart :as ts-c]
            [birdwatch.communicator :as comm]
            [birdwatch.charts.wordcount-chart :as wc-c]
            [birdwatch.charts.cloud-chart :as cloud]
            [birdwatch.ui.tweets :as tw]
            [birdwatch.ui.elements :as ui]
            [birdwatch.state.data :as state]
            [birdwatch.stats.timeseries :as ts]
            [birdwatch.stats.wordcount :as wc]
            [cljs.core.async :as async :refer [chan pub]]))

;;;; Main file of the BirdWatch client-side application.

;;; Channels for handling information flow in the application.
(def stats-chan (chan)) ; Stats from server, e.g. total tweets, connected users.
(def data-chan  (chan)) ; Data from server, e.g. new tweets and previous chunks.
(def qry-chan   (chan)) ; Queries that will be forwarded to the server.
(def cmd-chan   (chan)) ; Web-client internal command messages (e.g. state modification).
(def state-pub-chan (chan)) ; Publication of state changes.
(def state-pub (pub state-pub-chan first)) ; Pub for subscribing to

;;; Initialize application state (atom in state namespace) and wire channels.
(state/init-state data-chan qry-chan stats-chan cmd-chan state-pub-chan)

;;; Initialization of WebSocket communication.
(comm/start-communicator cmd-chan data-chan stats-chan qry-chan)

;;; Initialize Reagent components and inject channels.
(ui/init-views         state-pub cmd-chan)
(tw/mount-tweets       state-pub cmd-chan)
(wc-c/mount-wc-chart   state-pub cmd-chan {:bars 25 :every-ms 1000})
(cloud/mount-wordcloud state-pub cmd-chan {:n 250 :every-ms 5000})
(ts-c/mount-ts-chart   state-pub {:every-ms 1000})
~~~

The namespace starts with a couple of imports, mostly of other namespaces from this application. Other than that, only ````chan```` and ````pub```` from ````core.async```` are needed.

Next, we create the channels that provide the communication backbone of this application. All communication between the major building blocks of this application takes place by putting messages on conveyor belts, as Rich Hickey likes to call channels. These channels provide for a great abstraction. Just like conveyor belts, we can put data on them, without having to worry where the conveyor belts will take them. In fact, even if a component in the system wants to know where the channel leads to, it has no way of finding it out.

In addition to a couple of regular channels, there also is a ````pub````, which is a ````core.async```` construct for implementing the **[Publish-subscribe pattern](http://en.wikipedia.org/wiki/Publish–subscribe_pattern)**:

~~~
(def state-pub-chan (chan)) ; Publication of state changes.
(def state-pub (pub state-pub-chan first)) ; Pub for subscribing to
~~~

Here, the ````state-pub```` is a publisher that takes all items off the ````state-pub-chan```` and that uses the function ````first```` to determine the topic of a message. In order for that to work, it expects a data structure where the topic is the first element in a sequence or a vector. In this application, we will use vectors for that type of messages. The ````pub```` will then apply the function ````first```` to each element, so whatever is in the first position of that vector will be the topic of the message. **Namespaced keywords** are particularly useful for this purpose.

One might wonder if sending the entire dereferenced application state on a channel wasn't too expensive. But it is not expensive at all since we are dealing with an immutable data structure that exists already and which can be passed around cheaply because it doesn't need to be copied.

Then, with the channels and the pub created, we instantiate different components of the system and pass the channels to the components as needed. 

The **State** component, as you've seen in the previous architectural drawing, requires the ````data-chan```` and the ````stats-chan```` for incoming data from the server, the ````qry-chan```` for outgoing messages to the server, the ````state-pub-chan```` for publishing updated application state and the ````cmd-chan```` for receiving command messages from the UI. All these are injected by calling the ````state-init```` function:

~~~
;;; Initialize application state (atom in state namespace) and wire channels.
(state/init-state data-chan qry-chan stats-chan cmd-chan state-pub-chan)
~~~

Next, we initialize the **Communicator** for the WebSockets interaction with the server:

~~~
;;; Initialization of WebSocket communication.
(comm/start-communicator cmd-chan data-chan stats-chan qry-chan)
~~~

Here, the ````cmd-chan```` is required for triggering an initial search once the WebSocket connection with the server is established. ````data-chan```` and ````stats-chan```` are the channels onto which the component puts the respective data items coming from the server. Finally, ````qry-chan```` is the channel from which the component takes queries to forward them to the server.

Note that the Communicator and State components know nothing of each other. They only know the channels with which the communication is facilitated. This is a great feature especially when applications grow larger. We then want as little coupling as possible.

Finally, we have a couple of UI components and charts. They all follow the same pattern:

~~~
;;; Initialize Reagent components and inject channels.
(ui/init-views         state-pub cmd-chan)
(tw/mount-tweets       state-pub cmd-chan)
(wc-c/mount-wc-chart   state-pub cmd-chan {:bars 25 :every-ms 1000})
(cloud/mount-wordcloud state-pub cmd-chan {:n 250 :every-ms 5000})
(ts-c/mount-ts-chart   state-pub {:every-ms 1000})
~~~

All of them render the application state, which they receive by subscribing to the ````state-pub````. Then, when they require the sending of messages back to the application logic inside the State component, they are also provided with the ````cmd-chan````. The charts then also receive a map with the options. Besides chart-specific options like the number of words to render in the wordcloud or the number of bars to render in the bar chart, they also contain the ````:every-ms```` key. This key specifies how often the component will re-render, no matter how often the state changes. We will see later how this is implemented when discussing the charts. For now, all we need to know is that we have control over how often potentially expensive operations are triggered.

