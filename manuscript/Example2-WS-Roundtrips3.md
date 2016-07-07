## matthiasn.systems-toolbox-ui.charts.math

In the `matthiasn.systems-toolbox-ui.charts.histogram` namespace, we used a handful of mathematical helper functions. These live in the **[matthiasn.systems-toolbox-ui.charts.math](https://github.com/matthiasn/systems-toolbox-ui/blob/master/src/cljc/matthiasn/systems_toolbox_ui/charts/math.cljc)** namespace. This is actually a **[cljc](https://github.com/clojure/clojurescript/wiki/Using-cljc)** file, which allows us to target both **Clojure** and **ClojureScript**. This is very useful for **testing** the functions on the **JVM**, which I much prefer over testing in the browser. Here's the entire namespace:

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
  "Implements approximation of the Freedman-Diaconis rule for determing bin size in histograms:
   bin size = 2 IQR(x) n^-1/3 where IQR(x) is the interquartile range of the data and n is the
   number of observations in sample x. Argument is expected to be a sequence of numbers."
  [sample]
  (let [n (count sample)]
    (when (pos? n)
      (* 2 (interquartile-range sample) (Math/pow n (/ -1 3))))))

(defn round-up [n increment] (* (Math/ceil (/ n increment)) increment))
(defn round-down [n increment] (* (Math/floor (/ n increment)) increment))

(defn increment-fn
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
          best-increment (increment-fn increments 5 rng)]
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

