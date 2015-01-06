## SVG Manipulation with ClojureScript

Two of the three charts in the application are constructed using ClojureScript alone with no need for JavaScript interop. Only the word cloud chart uses D3.js as an external library plus some interop code.

The timeseries chart and the wordcount trend bar chart are constructed as SVG by using the **[reagent](https://github.com/reagent-project/reagent)** library, just like for displaying tweets. The only difference here is that we are not constructing HTML but **SVG** instead.

### Timeseries Chart

We've already seen the timeseries chart in the last chapter to get an idea what the data represents. Let's have a look at the **[code](https://github.com/matthiasn/BirdWatch/blob/574d2178be6f399086ad2a5ec35c200d252bf887/Clojure-Websockets/MainApp/src/cljs/birdwatch/charts/ts_chart.cljs)**:

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

First, we have a couple of ````def````s:

~~~
(def bars (atom []))
(def label (atom {}))

(def ts-elem (util/by-id "timeseries1"))
(def ts-w (aget ts-elem "offsetWidth"))
(def ts-h 100)
~~~

In ````bars````, as the name suggests, we simply store the data for the bars. Remember, this was is reset from the ````update-ts```` function in the ````birdwatch.timeseries```` namespace: ````(reset! tsc/bars (vec (ts-data app))````. Then, there's the ````label```` atom. If something is in here, a label is rendered, as we will see below. ````ts-elem```` is the DOM element where the chart is rendered. ````ts-w```` and ````ts-h```` are simply width and height of the chart. The width is dependent on the responsive layout managed by the CSS.

Next, we have the ````bar```` function:

~~~
(defn bar [x y h w idx]
  [:rect {:x x :y (- y h) :fill "steelblue" :width w :height h
          :on-mouse-enter #(reset! label {:idx idx})
          :on-mouse-leave #(reset! label {})}])
~~~

This renders a single bar of height ````h```` and width ````w```` at the coordinates ````x```` and ````y````. Note the ````:y (- y h)````. This is because in SVG's coordinate system, x=0 and y=0 is in the upper left corner, which is not that useful for charts. Then, we also have the ````idx````, which is the index of each bar. This is used for rendering a label. When the mouse enter the bar, the label is shown by setting the ````label```` atom: ````:on-mouse-enter #(reset! label {:idx idx})````, which is cleared again when the mouse leaves ````:on-mouse-leave #(reset! label {})````. We will look at the label below.

Next, we have the ````barchart```` function:

~~~
(defn barchart [indexed mx cnt w]
  (let [gap (/ (/ ts-w 20) cnt)]
    [:svg {:width ts-w :height ts-h}
     [:g
      (for [[idx [k v]] indexed]
        ^{:key k} [bar (* idx w) ts-h (* (/ v mx) ts-h) (- w gap) idx])]]))
~~~

This first of all determines the ````gap```` between the bars. It then renders the ````:svg```` with all bars inside a group ````:g````. Once again, we are setting the key so React can reuse elements and be more efficient.



![](images/ts-example-label.png)

~~~
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
~~~



Finally in this namespace, we have another component named ````ts-chart```` and render it into ````ts-elem````:

~~~
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

This component creates the ````:div```` that holds both the ````barchart```` SVG and ````labels````. Note that I've removed the Rickshaw library from the project, but for now I'm still using some of its CSS, e.g. the ````rickshaw_graph```` class. 