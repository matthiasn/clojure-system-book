### Wordcount Trends Chart (with Linear Regression)

The word count trends chart shows the top 25 words within the body of tweets loaded in the application, as determined in the ````birdwatch.wordcount```` namespace we have discussed previously. The ranking is in descending order, with a bar and a count number allowing for comparison between different words. In addition, there are trend arrows that show how a word has developed recently and over time. Each bar is also clickable, whic adds the word to the search in the input field below the menu bar. Here's how it looks like:

![](images/wordcount-chart.png)

Let's start with the simplest part, the ````birdwatch.charts.shapes```` **[namespace](https://github.com/matthiasn/BirdWatch/blob/574d2178be6f399086ad2a5ec35c200d252bf887/Clojure-Websockets/MainApp/src/cljs/birdwatch/charts/shapes.cljs)**:

~~~
(ns birdwatch.charts.shapes)

(def arrows
  {:RIGHT      ["#428bca" "-600,100 200,100 -200,500 100,500 600,0 100,-500 -200,-500 200,-100 -600,-100 "]
   :UP         ["#45cc40" "100,600 100,-200 500,200 500,-100 0,-600 -500,-100 -500,200 -100,-200 -100,600"]
   :DOWN       ["#dc322f" "100,-600 100,200 500,-200 500,100 0,600 -500,100 -500,-200 -100,200 -100,-600"]
   :RIGHT-UP   ["#45cc40" "400,-400 -200,-400 -350,-250 125,-250 -400,275 -275,400 250,-125 250,350 400,200"]
   :RIGHT-DOWN ["#dc322f" "400,400 -200,400 -350,250 125,250 -400,-275 -275,-400 250,125 250,-350 400,-200"]})

(defn arrow [x y dir]
  (let [[color points] (dir arrows)
        arrowTrans (str "translate(" x ", " (+ y 7) ") scale(0.01) ")]
    [:polygon {:transform arrowTrans :stroke "none" :fill color :points points}]))
~~~

Above, we first have a map named ````arrows````, which contains a vector with the color and the points for each arrow polygon. Then, we have a Reagent component that takes ````x```` and ````y```` coordinates and the direction ````dir```` and that returns a ````:polygon```` in the matching shape and color for the specified orientation of the arrow, positioned at the specified coordinates.

The ````arrow```` component is used in the ````birdwatch.charts.wordcount-chart```` **[namespace](https://github.com/matthiasn/BirdWatch/blob/574d2178be6f399086ad2a5ec35c200d252bf887/Clojure-Websockets/MainApp/src/cljs/birdwatch/charts/wordcount_chart.cljs)**, which we'll look at next:

~~~
(ns birdwatch.charts.wordcount-chart
  (:require [birdwatch.util :as util]
            [birdwatch.stats.regression :as reg]
            [birdwatch.charts.shapes :as s]
            [birdwatch.state :as state]
            [reagent.core :as r :refer [atom]]))

(enable-console-print!)

(def items (atom []))
(def pos-trends (atom {}))
(def pos-items (atom {}))
(def ratio-trends (atom {}))
(def ratio-items (atom {}))
(def ts-elem (util/by-id "wordcount-barchart"))
(def ts-w (aget ts-elem "offsetWidth"))
(def text-defaults {:stroke "none" :fill "#DDD" :fontWeight 500 :fontSize "0.8em" :dy ".35em" :textAnchor "end"})
(def opts [[10 "10 tweets"][100 "100 tweets"][500 "500 tweets"][1000 "1000 tweets"]])

(defn bar [text cnt y h w idx]
  (let [pos-slope (get @pos-trends text)
        ratio-slope (get @ratio-trends text)]
    [:g {:on-click #(state/append-search-text text)}
     [:text {:y (+ y 8) :x 138 :stroke "none" :fill "black" :dy ".35em" :textAnchor "end"} text]
     [s/arrow 146 y (cond (pos? pos-slope)   :UP       (neg? pos-slope )   :DOWN       :else :RIGHT)]
     [s/arrow 160 y (cond (pos? ratio-slope) :RIGHT-UP (neg? ratio-slope ) :RIGHT-DOWN :else :RIGHT)]
     [:rect {:y y :x 168 :height 15 :width w :stroke "white" :fill "#428bca"}]
     (if (> w 50)
       [:text (merge text-defaults {:y (+ y 8) :x (+ w 160)}) cnt]
       [:text (merge text-defaults {:y (+ y 8) :x (+ w 171) :fill "#666" :textAnchor "start"}) cnt])]))

(defn wordcount-barchart []
  (let [indexed @items
        mx (apply max (map (fn [[idx [k v]]] v) indexed))
        cnt (count indexed)]
    [:div
     [:svg {:width ts-w :height (+ (* cnt 15) 5)}
      [:g
       (for [[idx [text cnt]] indexed]
         ^{:key text} [bar text cnt (* idx 15) 15 (* (- ts-w 190) (/ cnt mx)) idx])
       [:line {:transform "translate(168, 0)" :y 0 :y2 (* cnt 15) :stroke "black"}]]]
     [:p.legend [:strong "1st trend indicator:"]
      " recent position changes"]
     [:p.legend [:strong "2nd trend indicator:"]
      " ratio change termCount / totalTermsCounted over last "
      [:select {:defaultValue 100}
       (for [[v t] opts] ^{:key v} [:option {:value v} t])]]]))

(r/render-component [wordcount-barchart] ts-elem)

(defn update-words
  "update wordcount chart"
  [words]
  (reset! items (vec (map-indexed vector words)))
  (let [items @items
        total-cnt (apply + (map (fn [[_[_ cnt]]] cnt) items))]
    (doseq [[idx [text cnt]] items]
      (swap! pos-items update-in [text] conj idx)
      (swap! ratio-items update-in [text] conj (/ total-cnt cnt))
      (swap! pos-trends assoc-in [text]
             (get (reg/linear-regression (take 3 (get @pos-items text))) 1))
      (swap! ratio-trends assoc-in [text]
             (get (reg/linear-regression (take 1000 (get @ratio-items text))) 1)))))
~~~

Okay, this is quite a bit to go through, but the code also takes care of the linear regression for the trend arrow. Before we go through the code function by function, let's have a look at the ````birdwatch.stats.regression```` **[namespace](https://github.com/matthiasn/BirdWatch/blob/574d2178be6f399086ad2a5ec35c200d252bf887/Clojure-Websockets/MainApp/src/cljs/birdwatch/stats/regression.cljs)** first:

~~~
(ns birdwatch.stats.regression)

(enable-console-print!)

(defn square [x] (* x x))
(defn mean [xs]
  (let [cnt (count xs)]
    (when (pos? cnt) (/ (apply + xs) cnt))))

; adapted from from http://compbio.ucdenver.edu/Hunter_lab/Hunter/cl-statistics.lisp
(defn linear-regression [ys]
  (let [n (count ys)]
    (when (pos? n)
      (let [xs (range n)
            x-bar (mean xs)
            y-bar (mean ys)
            Lxx (reduce + (map (fn [xi] (square (- xi x-bar))) xs))
            Lyy (reduce + (map (fn [yi] (square (- yi y-bar))) ys))
            Lxy (reduce + (map (fn [xi yi] (* (- xi x-bar) (- yi y-bar))) xs ys))
            slope (/ Lxy Lxx)
            intercept (- y-bar (* slope x-bar))
            reg-ss (* slope Lxy)
            res-ms (/ (- Lyy reg-ss) (- n 2))]
        [intercept slope]))))
~~~

I've adapted the code from **[Statistical functions in Common Lisp. Version 1.04](http://compbio.ucdenver.edu/Hunter_lab/Hunter/cl-statistics.lisp)** and just removed the stuff I didn't need plus also created the functions ````square```` and ````mean```` for use in the ````linear-regression```` function. 

![](images/linear-regression.png)
