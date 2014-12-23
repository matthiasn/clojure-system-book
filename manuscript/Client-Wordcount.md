## Wordcount Statistics

https://github.com/matthiasn/BirdWatch/blob/574d2178be6f399086ad2a5ec35c200d252bf887/Clojure-Websockets/MainApp/src/cljs/birdwatch/wordcount.cljs

~~~
(ns birdwatch.wordcount
  (:require [clojure.string :as s]
            [birdwatch.util :as util]))
            
;; shortened to fit the page better, see link for full set
(def stop-words #{"use" "good" "want" "amp" "just" "now" "like" "til" "new" "get" "one" "i" 
                  "us" "our" "ours" "ourselves" "you" "your" "yours" "yourself" "yourselves"
                  "she" "her" "hers" "herself" "it" "its" "itself" "they" "them" "their" 
                  "which" "who" "whom" "whose" "this" "that" "these" "those" "am" "is" 
                  "being" "have" "has" "had" "having" "do" "does" "did" "doing" "will"})

(defn get-words
  "get vector of maps with word as :key and count as :value"
  [app n]
  (vec (map (fn [w] (let [[k v] w] {:key k :value v})) (take n (:words-sorted-by-count @app)))))

(defn get-words2
  "get vector of maps with word as :key and count as :value"
  [app n]
  (vec (take n (:words-sorted-by-count @app))))

(defn add-word
  "add word to the words map and the sorted set with the counts (while discarding old entry)"
  [app word]
  (util/swap-pmap app :words-sorted-by-count word (inc (get (:words-sorted-by-count @app) word 0))))

(defn process-tweet
  "process tweet: split, filter, lower case, replace punctuation, add word"
  [app text]
  (doall ;; initially lazy, needs realization
   (->> (s/split text #"[\s—\u3031-\u3035\u0027\u309b\u309c\u30a0\u30fc\uff70]+")
        (filter #(not (re-find #"(@|https?:)" %)) ,)
        (filter #(> (count %) 3) ,)
        (filter #(< (count %) 25) ,)
        (map s/lower-case ,)
        (map #(s/replace % #"[;:,/‘’…~\-!?\[\]\"<>()\"@.]+" "" ) ,)
        (filter (fn [item] (not (contains? stop-words item))) ,)
        (map #(add-word app %) ,))))
~~~