## matthiasn.systems-toolbox-ui.charts.histogram

The most interesting stuff for rendering the histograms happens in the **[matthiasn.systems-toolbox-ui.charts.histogram](https://github.com/matthiasn/systems-toolbox-ui/blob/master/src/cljs/matthiasn/systems_toolbox_ui/charts/histogram.cljs)** namespace. Feel free to skip it if you don't particularly care about constructing **[Scalable Vector Graphics](https://en.wikipedia.org/wiki/Scalable_Vector_Graphics)**. Also, this is not an introduction to SVG, as that's not the focus of this book, so I will only describe the construction of SVGs with **Reagent**, not what an **SVG** is. If you've never worked with Scalable Vector Graphics,
maybe this **[tutorial](https://developer.mozilla.org/en-US/docs/Web/SVG/Tutorial/Introduction)** will give you a gentler introduction. 

Okay, with that being said, let's dive into the code:

~~~
(ns matthiasn.systems-toolbox-ui.charts.histogram
  "Functions for building a histogram, rendered as SVG using Reagent and React."
  (:require [matthiasn.systems-toolbox-ui.charts.math :as m]))

(def text-default {:stroke "none" :fill "black" :style {:font-size 12}})
(def text-bold (merge text-default {:style {:font-weight :bold :font-size 12}}))
(def x-axis-label (merge text-default {:text-anchor :middle}))
(def y-axis-label (merge text-default {:text-anchor :end}))

(defn path
  "Renders path with the given path description attribute."
  [d]
  [:path {:fill         :black
          :stroke       :black
          :stroke-width 1
          :d            d}])

(defn histogram-y-axis
  "Draws y-axis for histogram."
  [x y h mx y-label]
  (let [incr (m/default-increment-fn mx)
        rng (range 0 (inc (m/round-up mx incr)) incr)
        scale (/ h (dec (count rng)))]
    [:g
     [path (str "M" x " " y "l 0 " (* h -1) " z")]
     (for [n rng]
       ^{:key (str "yt" n)}
       [path (str "M" x " " (- y (* (/ n incr) scale)) "l -" 6 " 0")])
     (for [n rng]
       ^{:key (str "yl" n)}
       [:text (merge y-axis-label {:x (- x 10)
                                   :y (- y (* (/ n incr) scale) -4)}) n])
     [:text (let [x-coord (- x 45)
                  y-coord (- y (/ h 3))
                  rotate (str "rotate(270 " x-coord " " y-coord ")")]
              (merge x-axis-label text-bold {:x         x-coord
                                             :y         y-coord
                                             :transform rotate})) y-label]]))

(defn histogram-x-axis
  "Draws x-axis for histogram."
  [x y mn mx w scale increment x-label]
  (let [rng (range mn (inc mx) increment)]
    [:g
     [path (str "M" x " " y "l" w " 0 z")]
     (for [n rng]
       ^{:key (str "xt" n)}
       [path (str "M" (+ x (* (- n mn) scale)) " " y "l 0 " 6)])
     (for [n rng]
       ^{:key (str "xl" n)}
       [:text (merge x-axis-label {:x (+ x (* (- n mn) scale))
                                   :y (+ y 20)}) n])
     [:text (merge x-axis-label text-bold {:x (+ x (/ w 2))
                                           :y (+ y 48)}) x-label]]))

(defn insufficient-data
  "Renders warning when data insufficient."
  [x y w text]
  [:text {:x           (+ x (/ w 2))
          :y           (- y 50)
          :stroke      "none"
          :fill        "#DDD"
          :text-anchor :middle
          :style       {:font-weight :bold :font-size 24}} text])

(defn histogram-view-fn
  "Renders a histogram."
  [{:keys [data x y w h x-label y-label color bin-cf min-bins max-bins increment-fn warning]}]
  (let [mx (apply max data)
        mn (apply min data)
        rng (- mx mn)
        increment-fn (or increment-fn m/default-increment-fn)
        increment (increment-fn rng)
        mx2 (m/round-up (or mx 10) increment)
        mn2 (m/round-down (or mn 0) increment)
        x-scale (/ w (- mx2 mn2))
        bin-size (max (/ rng max-bins) (* (m/freedman-diaconis-rule data) bin-cf))
        binned-freq (frequencies (map (fn [n] (Math/floor (/ (- n mn) bin-size))) data))
        binned-freq-mx (apply max (map (fn [[_ f]] f) binned-freq))
        bins (inc (apply max (map (fn [[v _]] v) binned-freq)))
        bar-width (/ (* rng x-scale) bins)
        y-scale (/ (- h 20) binned-freq-mx)]
    [:g
     (if (>= bins min-bins)
       (for [[v f] binned-freq]
         ^{:key (str "bf" x "-" y "-" v "-" f)}
         [:rect {:x      (+ x (* (- mn mn2) x-scale) (* v bar-width))
                 :y      (- y (* f y-scale))
                 :fill   color :stroke "black"
                 :width  bar-width
                 :height (* f y-scale)}])
       [insufficient-data x y w warning])
     [histogram-x-axis x (+ y 7) mn2 mx2 w x-scale increment x-label]
     [histogram-y-axis (- x 7) y h (or binned-freq-mx 5) y-label]]))

(defn histogram-view
  "Renders an individual histogram for the given data, dimension, label and color, with a
   reasonable size inside a viewBox, which will then scale smoothly into any div you put it in."
  [data label color]
  [:svg {:width   "100%"
         :viewBox "0 0 400 250"}
   (histogram-view-fn {:data     data
                       :x        80
                       :y        180
                       :w        300
                       :h        160
                       :x-label  label
                       :y-label  "Frequencies"
                       :warning  "insufficient data"
                       :color    color
                       :bin-cf   0.8
                       :min-bins 4
                       :max-bins 25})])
~~~

Okay, that was a bit involved. But hey, in order to use a histogram in your project, all you need is a import this namespace, and then use a one-liner to plot your histogram (and more chart types to come - feel free to contribute).

After skim reading the namespace, are you still interested in constructing charts? Good, then let's go through function by function:

~~~
(defn histogram-view
  "Renders an individual histogram for the given data, dimension, label and color, with a
   reasonable size inside a viewBox, which will then scale smoothly into any div you put it in."
  [data label color]
  [:svg {:width   "100%"
         :viewBox "0 0 400 250"}
   (histogram-view-fn {:data     data
                       :x        80
                       :y        180
                       :w        300
                       :h        160
                       :x-label  label
                       :y-label  "Frequencies"
                       :warning  "insufficient data"
                       :color    color
                       :bin-cf   0.8
                       :min-bins 5
                       :max-bins 25})])
~~~

The `histogram-view` function simply creates a container `:svg` element, which scales into its parent element through the `:width "100%"` setting. Also note the `:viewBox "0 0 400 250"`, which allows us to work with a useful coordinate system that's independent from the size of the rendered element. Finally, the we pass some data to the `histogram-view-fn`, which we'll look into next.

~~~
(defn histogram-view-fn
  "Renders a histogram."
  [{:keys [data x y w h x-label y-label color bin-cf min-bins max-bins increment-fn warning]}]
  (let [mx (apply max data)
        mn (apply min data)
        rng (- mx mn)
        increment-fn (or increment-fn m/default-increment-fn)
        increment (increment-fn rng)
        mx2 (m/round-up (or mx 10) increment)
        mn2 (m/round-down (or mn 0) increment)
        x-scale (/ w (- mx2 mn2))
        bin-size (max (/ rng max-bins) (* (m/freedman-diaconis-rule data) bin-cf))
        binned-freq (frequencies (map (fn [n] (Math/floor (/ (- n mn) bin-size))) data))
        binned-freq-mx (apply max (map (fn [[_ f]] f) binned-freq))
        bins (inc (apply max (map (fn [[v _]] v) binned-freq)))
        bar-width (/ (* rng x-scale) bins)
        y-scale (/ (- h 20) binned-freq-mx)]
    [:g
     (if (>= bins min-bins)
       (for [[v f] binned-freq]
         ^{:key (str "bf" x "-" y "-" v "-" f)}
         [:rect {:x      (+ x (* (- mn mn2) x-scale) (* v bar-width))
                 :y      (- y (* f y-scale))
                 :fill   color :stroke "black"
                 :width  bar-width
                 :height (* f y-scale)}])
       [insufficient-data x y w warning])
     [histogram-x-axis x (+ y 7) mn2 mx2 w x-scale increment x-label]
     [histogram-y-axis (- x 7) y h (or binned-freq-mx 5) y-label]]))
~~~

Above, we render an **[SVG g element](https://developer.mozilla.org/en/docs/Web/SVG/Element/g)**, which contains the histogram. Before doing so, we need to calculate a few things from the provided data, which happens in the `let` binding, starting with the maximum value `mx`, the minimum value `mn` and the range contained in the data, `rng`. Next, we calculate the increments between the intervals on the histogram's x-axis, either by calling a provided `increment-fn`, or, if that doesn't exist, the `default-increment-fn`. We'll look into that when discussing the `charts.math` namespace in the next section. For now, it is enough to know that it'll give us a reasonable increment to use for the intervals, such as `10`, `25`, or also `500`, depending on the range in the provided `data`. Next, we calculate `mn2` and `mx2`, which are the next possible intervals given the chosen increments. Then, we calculate the `x-scale`, which will be used to translate positions into the given coordinate system.

Next, we calculate the size of the bins using the `freedman-diaconis-rule` function. We'll look into that function in the next section. For now, it's enough to know that the following four lines give us a reasonable number of bins for the provided data. A bin then translates into a bar in the histogram.

~~~
        bin-size (max (/ rng max-bins) (* (m/freedman-diaconis-rule data) bin-cf))
        binned-freq (frequencies (map (fn [n] (Math/floor (/ (- n mn) bin-size))) data))
        binned-freq-mx (apply max (map (fn [[_ f]] f) binned-freq))
        bins (inc (apply max (map (fn [[v _]] v) binned-freq)))
~~~

Finally, we calculate the width of each bar in the histogram, plus the `y-scale`, which is like the `x-scale`, only for the **y-axis**.

With the calculations done, we can render the histogram into a `:g` element. Here, the bars are only displayed if there are enough bins, otherwise we display `"insufficient data"`. The number of bins is configured in the `:min-bins` key of the argument map. When called from the `histogram-view`, I've chosen a minimum of five bins. This value is entirely arbitrary, but seems to work fairly well. Less than five bins look stupid, and don't provide much meaningful information either.

If the data is deemed sufficient, we render a vertical bar as a `:rect` for each bin. This happens in a **[for-comprehension](https://clojuredocs.org/clojure.core/for)**, as you've already seen in the previous chapter. Of importance here is the `:key` on each elements' metadata. While we would get by without, **React** needs this key to work more efficiently by reusing elements in the next render cycle. Without assigning the keys, the browser needs to do more work, and React prints long and ugly warnings in the browser's console.

Then, we render the **x-axis** by calling `histogram-x-axis`, and the **y-axis** in `histogram-y-axis`.

The fucntions for rendering the axes are fairly straightforward. Here's the `histogram-x-axis` function:

~~~
(defn histogram-x-axis
  "Draws x-axis for histogram."
  [x y mn mx w scale increment x-label]
  (let [rng (range mn (inc mx) increment)]
    [:g
     [path (str "M" x " " y "l" w " 0 z")]
     (for [n rng]
       ^{:key (str "xt" n)}
       [path (str "M" (+ x (* (- n mn) scale)) " " y "l 0 " 6)])
     (for [n rng]
       ^{:key (str "xl" n)}
       [:text (merge x-axis-label {:x (+ x (* (- n mn) scale))
                                   :y (+ y 20)}) n])
     [:text (merge x-axis-label text-bold {:x (+ x (/ w 2))
                                           :y (+ y 48)}) x-label]]))
~~~

The mechanism here will probably look fairly familiar by now:

* there's a group inside the `:g` element
* next, there's the axis itself, rendered by the `path` function
* there's a `for`-comprehension for the ticks on the axis, which also use the `path` function
* there's another `for`-comprehension for the axis labels (the numbers associated with a tick)
* finally, there's a label, which in our example application here would for example be `"Roundtrip t/ms"`

Both `for`-comprehensions make use of the range `rng`, which is a sequence from `mn` to one larger than `mx`, with the step size `increment`. All these values depend on the data and are computed individually, as we will see in the next section.

Here's the aforementioned `path` function, which is really only a thin wrapper over `:path`, with a few defaults so we save some typing later on:

~~~
(defn path
  "Renders path with the given path description attribute."
  [d]
  [:path {:fill         :black
          :stroke       :black
          :stroke-width 1
          :d            d}])
~~~


The `histogram-y-axis` is somewhat similar, only that here we can calculate more in the function, as we don't need `scale` or `rng` in the calculation of the bins:


~~~
(defn histogram-y-axis
  "Draws y-axis for histogram."
  [x y h mx y-label]
  (let [incr (m/default-increment-fn mx)
        rng (range 0 (inc (m/round-up mx incr)) incr)
        scale (/ h (dec (count rng)))]
    [:g
     [path (str "M" x " " y "l 0 " (* h -1) " z")]
     (for [n rng]
       ^{:key (str "yt" n)}
       [path (str "M" x " " (- y (* (/ n incr) scale)) "l -" 6 " 0")])
     (for [n rng]
       ^{:key (str "yl" n)}
       [:text (merge y-axis-label {:x (- x 10)
                                   :y (- y (* (/ n incr) scale) -4)}) n])
     [:text (let [x-coord (- x 45)
                  y-coord (- y (/ h 3))
                  rotate (str "rotate(270 " x-coord " " y-coord ")")]
              (merge x-axis-label text-bold {:x         x-coord
                                             :y         y-coord
                                             :transform rotate})) y-label]]))
~~~


If you want to use a histogram in your application and are happy with the defaults, you can simply call the `histogram-view` function. Or, if you want more fine-grained control, you can copy this function and use the values you desire. Or, you can of course use this whole thing as an inspiration and come up with your own chart. In that case, please consider submitting a PR, others might benefit from that, too.


## matthiasn.systems-toolbox-ui.charts.math

In the `matthiasn.systems-toolbox-ui.charts.histogram` namespace, we used a handful of mathematical helper functions. These live in the **[matthiasn.systems-toolbox-ui.charts.math](https://github.com/matthiasn/systems-toolbox-ui/blob/master/src/cljc/matthiasn/systems_toolbox_ui/charts/math.cljc)** namespace. This is actually a **[cljc](https://github.com/clojure/clojurescript/wiki/Using-cljc)** file, which allows us to target both **Clojure** and **ClojureScript**, which in this case is useful for **testing** the functions on the **JVM**. Here's the entire namespace:

~~~
(ns matthiasn.systems-toolbox-ui.charts.math)

(defn interquartile-range
  "Determines the interquartile range of values in a collection of numbers."
  [sample]
  (let [sorted (sort sample)
        n (count sorted)
        q1 (nth sorted (Math/floor (/ n 4)))
        q3 (nth sorted (Math/floor (* (/ n 4) 3)))
        iqr (- q3 q1)]
    iqr))

(defn percentile-range
  "Returns only the values within the given percentile range."
  [sample percentile]
  (let [sorted (sort sample)
        n (count sorted)
        keep-n (Math/ceil (* n (/ percentile 100)))]
    (take keep-n sorted)))

(defn freedman-diaconis-rule
  "Implements approximation of Freedman-Diaconis rule for determing bin size in histograms:
  bin size = 2 IQR(x) n^-1/3 where IQR(x) is the interquartile range of the data and n is the
  number of observations in the sample x. Argument coll is expected to be a collection of numbers."
  [sample]
  (let [n (count sample)]
    (when (pos? n)
      (* 2 (interquartile-range sample) (Math/pow n (/ -1 3))))))

(defn round-up [n increment] (* (Math/ceil (/ n increment)) increment))
(defn round-down [n increment] (* (Math/floor (/ n increment)) increment))

(defn abs
  "(abs n) is the absolute value of n.
  Borrowed from: https://github.com/clojure/math.numeric-tower"
  [n]
  (cond
    (not (number? n)) (throw #?(:clj   (IllegalArgumentException. "abs requires a number")
                                :cljs (js/Error. "abs requires a number")))
    (neg? n) (* n -1)
    :else n))

(defn exp [x n]
  #?(:clj (Math/pow x n)
     :cljs (.pow js/Math x n)))

(defn increment-fn
  "Takes a seq of increments, a desired number of intervals in histogram axis,
   and the range of the values in the histogram. Sorts the values in increments
   by dividing the range by each to determine number of intervals with this value,
   subtracting the desired number of intervals, and then returning the increment
   with the smallest delta."
  [increments desired-n rng]
  (first (sort-by #(abs (- (/ rng %) desired-n)) increments)))

(defn default-increment-fn
  "Determines the increment between intervals in histogram axis.
   Defaults to increments in a range between 1 and 5,000,000."
  [rng]
  (if rng
    (let [multipliers (map #(exp 10 %) (range 0 6))
          increments (flatten (map (fn [i] (map #(* i %) multipliers)) [1 2.5 5]))
          best-increment (increment-fn increments 5 rng)]
      (if (zero? (mod best-increment 1))
        (int best-increment)
        best-increment))
    1))
~~~

