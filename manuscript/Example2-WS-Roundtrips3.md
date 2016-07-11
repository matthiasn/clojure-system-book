## matthiasn.systems-toolbox-ui.charts.math

In the `matthiasn.systems-toolbox-ui.charts.histogram` namespace, we used a handful of mathematical helper functions. These live in the **[matthiasn.systems-toolbox-ui.charts.math](https://github.com/matthiasn/systems-toolbox-ui/blob/master/src/cljc/matthiasn/systems_toolbox_ui/charts/math.cljc)** namespace. This is actually a **[cljc](https://github.com/clojure/clojurescript/wiki/Using-cljc)** file, which allows us to target both **Clojure** and **ClojureScript**. This is very useful for **testing** the functions on the **JVM**, which I much prefer over testing in the browser. Here's the entire namespace:

~~~
(ns matthiasn.systems-toolbox-ui.charts.math)

(defn mean
  "From: https://github.com/clojure-cookbook/"
  [coll]
  (let [sum (apply + coll)
        count (count coll)]
    (if (pos? count)
      (/ sum count)
      0)))

(defn median
  "Modified from: https://github.com/clojure-cookbook/
   Adapted to return nil when collection empty."
  [coll]
  (let [sorted (sort coll)
        cnt (count sorted)
        halfway (quot cnt 2)]
    (if (empty? coll)
      nil
      (if (odd? cnt)
        (nth sorted halfway)
        (let [bottom (dec halfway)
              bottom-val (nth sorted bottom)
              top-val (nth sorted halfway)]
          (mean [bottom-val top-val]))))))

(defn interquartile-range
  "Determines the interquartile range of values in a sequence of numbers. Returns nil
   when sequence empty or only contains a single entry."
  [sample]
  (let [sorted (sort sample)
        cnt (count sorted)
        half-cnt (quot cnt 2)
        q1 (median (take half-cnt sorted))
        q3 (median (take-last half-cnt sorted))]
    (when (and q3 q1) (- q3 q1))))

(defn percentile-range
  "Returns only the values within the given percentile range."
  [sample percentile]
  (let [sorted (sort sample)
        cnt (count sorted)
        keep-n (Math/ceil (* n (/ percentile 100)))]
    (take keep-n sorted)))

(defn freedman-diaconis-rule
  "Implements approximation of the Freedman-Diaconis rule for determing bin size in histograms:
   bin size = 2 IQR(x) n^-1/3 where IQR(x) is the interquartile range of the data and n is the
   number of observations in sample x. Argument is expected to be a sequence of numbers."
  [sample]
  (let [n (count sample)]
    (when (pos? n)
      (* 2 (interquartile-range sample) (Math/pow n (/ -1 3))))))

(defn round-up [n increment] (* (Math/ceil (/ n increment)) increment))
(defn round-down [n increment] (* (Math/floor (/ n increment)) increment))

(defn best-increment-fn
  "Takes a seq of increments, a desired number of intervals in histogram axis,
   and the range of the values in the histogram. Sorts the values in increments
   by dividing the range by each to determine number of intervals with this value,
   subtracting the desired number of intervals, and then returning the increment
   with the smallest delta."
  [increments desired-n rng]
  (first (sort-by #(Math/abs (- (/ rng %) desired-n)) increments)))

(defn default-increment-fn
  "Determines the increment between intervals in histogram axis.
   Defaults to increments in a range between 1 and 5,000,000."
  [rng]
  (if rng
    (let [multipliers (map #(Math/pow 10 %) (range 0 6))
          increments (flatten (map (fn [i] (map #(* i %) multipliers)) [1 2.5 5]))
          best-increment (best-increment-fn increments 5 rng)]
      (if (zero? (mod best-increment 1))
        (int best-increment)
        best-increment))
    1))

(defn histogram-calc
  "Calculations for histogram."
  [{:keys [data bin-cf max-bins increment-fn]}]
  (let [mx (apply max data)
        mn (apply min data)
        rng (- mx mn)
        increment-fn (or increment-fn default-increment-fn)
        increment (increment-fn rng)
        bin-size (max (/ rng max-bins) (* (freedman-diaconis-rule data) bin-cf))
        binned-freq (frequencies (map (fn [n] (Math/floor (/ (- n mn) bin-size))) data))]
    {:mn             mn
     :mn2            (round-down (or mn 0) increment)
     :mx2            (round-up (or mx 10) increment)
     :rng            rng
     :increment      increment
     :binned-freq    binned-freq
     :binned-freq-mx (apply max (map (fn [[_ f]] f) binned-freq))
     :bins           (inc (apply max (map (fn [[v _]] v) binned-freq)))}))
~~~

The first two functions here, `mean` and `median`, are borrowed from the **[Clojure Cookbook](https://github.com/clojure-cookbook/")**. I've adapted `median` to return `nil` when the collection is empty, as that's useful further on. I've taken those two functions because they are useful in my implementation of the **[interquartile range](https://en.wikipedia.org/wiki/Interquartile_range)**:

~~~
(defn interquartile-range
  "Determines the interquartile range of values in a sequence of numbers. Returns nil
   when sequence empty or only contains a single entry."
  [sample]
  (let [sorted (sort sample)
        cnt (count sorted)
        half-cnt (quot cnt 2)
        q1 (median (take half-cnt sorted))
        q3 (median (take-last half-cnt sorted))]
    (when (and q3 q1) (- q3 q1))))
~~~

The `interquartile-range` function takes a sample, which is a sequence of numbers. If this sequence is empty, `nil` is returned. Otherwise, we `sort` the sequence, `count` it, and then take the `half-cnt`, which is the `floor` of dividing the `cnt` by two. This is the number of items in half the data, minus the halfway point if there is one (when `cnt` is odd). Then the values `q1` and `q3` are defined as the `median` of the first or last half of the values, respectively. Finally, the IQR is returned, which is the difference between `q1` and `q3`, and thus the range of half the data. The interquartile range is something we need to determine when computing the bin size via the **[Freedman-Diaconis Rule](https://en.wikipedia.org/wiki/Freedman%E2%80%93Diaconis_rule)**:

~~~
(defn freedman-diaconis-rule
  "Implements approximation of the Freedman-Diaconis rule for determing bin size in histograms:
   bin size = 2 IQR(x) n^-1/3 where IQR(x) is the interquartile range of the data and n is the
   number of observations in sample x. Argument is expected to be a sequence of numbers."
  [sample]
  (let [n (count sample)]
    (when (pos? n)
      (* 2 (interquartile-range sample) (Math/pow n (/ -1 3))))))
~~~

The freedman-diaconis-rule is fairly simply, once we have implemented the IQR:

* determine the IQR
* multiply it by 2
* multiply it by the cube root of n, the count of items

This gives us a suggested size of the bins in a histogram, which can then be used to determine the number of bins and thus the number of bars to display in our histogram.

Then, there's also the `percentile-range` function:

~~~
(defn percentile-range
  "Returns only the values within the given percentile range."
  [sample percentile]
  (let [sorted (sort sample)
        cnt (count sorted)
        keep-n (Math/ceil (* cnt (/ percentile 100)))]
    (take keep-n sorted)))
~~~

This function helps when trying to get rid of outliers. This may or may not be helpful in your data. Here, it sometimes helps, for example when all values are in the low hundreds and there's a single outlier in the thousands, as that outlier would otherwise lead to bins that are too large, with many empty bins. As with all visualization, it depends on the data and requires some experimentation.

Next, there are helpers for determining the intervals at which to put the ticks in the histogram axes:

~~~
(defn best-increment-fn
  "Takes a seq of increments, a desired number of intervals in histogram axis,
   and the range of the values in the histogram. Sorts the values in increments
   by dividing the range by each to determine number of intervals with this value,
   subtracting the desired number of intervals, and then returning the increment
   with the smallest delta."
  [increments desired-n rng]
  (first (sort-by #(Math/abs (- (/ rng %) desired-n)) increments)))

(defn default-increment-fn
  "Determines the increment between intervals in histogram axis.
   Defaults to increments in a range between 1 and 5,000,000."
  [rng]
  (if rng
    (let [multipliers (map #(Math/pow 10 %) (range 0 6))
          increments (flatten (map (fn [i] (map #(* i %) multipliers)) [1 2.5 5]))
          best-increment (best-increment-fn increments 5 rng)]
      (if (zero? (mod best-increment 1))
        (int best-increment)
        best-increment))
    1))
~~~

This is an interesting problem. Of course we could hardwire the increments between the ticks, but then the histogram would hardly be reusable. My initial approach was something this:

~~~
(defn default-increment-fn
  [rng]
  (cond (> rng 20000) 5000
        (> rng 8000) 2000
        (> rng 3000) 1000
        (> rng 1500) 500
        (> rng 900) 200
        (> rng 400) 100
        (> rng 200) 50
        (> rng 90) 20
        :else 10))
~~~ 

Depending on the range `rng`, I would select different spacing between the ticks on an axis. But that's not really general enough. So what I came up with instead is this:

* generate some multipliers, such as `(1.0 10.0 100.0 1000.0 10000.0 100000.0)`
* multiply each with `1`, `2.5` and `5`, then flatten those result vectors
* then, with this sequence of possible and a target value of five ticks (which look good in a histogram IMHO), call `best-increment` function
* there, the candidate increments are sorted by the delta between the desired number of ticks and the number of ticks we'd get with the respective increment
* the first of these sorted values is returned, which is the one with the smallest delta

This approach is much more generic and seems to work well.

D> I'm always amazed that we can do all these calculations whenever there's a change in the data. The browser really has become a powerful environment these days.

