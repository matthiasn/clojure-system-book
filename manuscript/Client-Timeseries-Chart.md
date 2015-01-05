## SVG Manipulation with ClojureScript

Two of the three charts in the application are constructed using ClojureScript alone with no need for JavaScript interop. Only the word cloud chart uses D3.js as an external library plus some interop code.

The timeseries chart and the wordcount trend bar chart are constructed as SVG by using the **[reagent](https://github.com/reagent-project/reagent)** library, just like for displaying tweets. The only difference here is that we are not constructing HTML but **SVG** instead.

### Timeseries Chart

https://github.com/matthiasn/BirdWatch/blob/574d2178be6f399086ad2a5ec35c200d252bf887/Clojure-Websockets/MainApp/src/cljs/birdwatch/charts/ts_chart.cljs

~~~
(ns birdwatch.charts.ts-chart
  (:require [birdwatch.util :as util]
            [reagent.core :as r :refer [atom]]))

(enable-console-print!)

(def bars (atom []))
(def label (atom {}))

(def ts-elem (util/by-id "timeseries1"))
(def ts-w (aget ts-elem "offsetWidth"))
(def ts-h 100)

(defn bar [x y h w idx]
  [:rect {:x x :y (- y h) :fill "steelblue" :width w :height h
          :on-mouse-enter #(reset! label {:idx idx})
          :on-mouse-leave #(reset! label {})}])

(defn barchart [indexed mx cnt w]
  (let [gap (/ (/ ts-w 20) cnt)]
    [:svg {:width ts-w :height ts-h}
     [:g
      (for [[idx [k v]] indexed]
        ^{:key k} [bar (* idx w) ts-h (* (/ v mx) ts-h) (- w gap) idx])]]))

(defn labels [bars mx cnt w]
  (when-not (empty? @label)
    (let [idx (:idx @label)
          [k v] (get bars idx)
          top (- ts-h (* (/ v mx) ts-h))
          lr (if (< (/ idx cnt) 0.6) "left" "right")]
      [:div.detail {:style {:left (* idx w)}}
       [:div.x_label {:class lr} (.toString (.unix js/moment k))]
       [:div.item.active {:class lr :style {:top top}} "Tweets: " v]
       [:div.dot.active {:style {:top top :border-color "steelblue"}}]])))

(defn ts-chart []
  (let [bars @bars
        indexed (vec (map-indexed vector bars))
        mx (apply max (map (fn [[k v]] v) bars))
        cnt (count bars)
        w (/ ts-w cnt)]
    [:div.rickshaw_graph
     [barchart indexed mx cnt w]
     [labels bars mx cnt w]]))

(r/render-component [ts-chart] ts-elem)
~~~
