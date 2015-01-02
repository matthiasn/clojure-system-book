## Core Namespace

This namespace initializes the application on the client side. Here's the entire **[code](https://github.com/matthiasn/BirdWatch/blob/117689dee3559506cad747a01a5bba02502eb817/Clojure-Websockets/MainApp/src/cljs/birdwatch/core.cljs)**, with the explanation below.

~~~
(ns birdwatch.core
  (:require [birdwatch.util :as util]
            [birdwatch.timeseries :as ts]
            [birdwatch.communicator :as comm]
            [birdwatch.charts.wordcount-chart :as wc-c]
            [birdwatch.charts.cloud-chart :as cloud]
            [birdwatch.ui.elements :as ui]
            [birdwatch.state :as state]
            [birdwatch.wordcount :as wc]))

;;;; Main file of the BirdWatch application written in ClojureScript

;;; The application state lives in a single atom in birdwatch.state and
;;; will be initialized with the map returned by state/initial-state.
(reset! state/app (state/initial-state))

;;; Reagent components for the application are initialized here.
(ui/init-views)

; update the expensive word cloud periodically
(util/update-loop cloud/redraw 5000)

; update the cheap charts every second
(util/update-loop #(ts/update-ts state/app) 1000)
(util/update-loop #(wc-c/update-words (wc/get-words2 state/app 25)) 1000)

;;; The app starts with the search string encoded in the URI location hash.
(swap! state/app assoc :search-text (util/search-hash))
~~~

The first thing that happens is that the initial application state is set in the respective atom. There is a function named ````initial-state````in the ````state```` namespace that returns a "blank slate" application state, it's result is used to populate the ````state/app```` atom.

Next, ````ui/init-views```` is called, a function that initializes all UI components. we will look at it later when covering the ````ui```` namespace.

Then, ````update-loop````s are started for updating the charts in regular intervals. The charts themselves are initialized in their respective namespaces. We will look at that when discussing the charts.
