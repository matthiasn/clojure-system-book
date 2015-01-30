### Wordcount Statistics

The ````birdwatch.wordcount```` **[namespace](https://github.com/matthiasn/BirdWatch/blob/8a469ff1493816dbedc6cbca9b51fe915aec487f/Clojure-Websockets/MainApp/src/cljs/birdwatch/stats/wordcount.cljs)** is responsible for running wordcount statistics over tweets received from the server for both live tweets and chunks of previous tweets.

~~~
(ns birdwatch.wordcount
  (:require [clojure.string :as s]
            [birdwatch.util :as util]))
            
;; shortened to fit the page better, see link for full set
(def stop-words
  #{"use" "good" "want" "amp" "just" "now" "like" "til" "new" "get" "one" "i" 
    "us" "our" "ours" "ourselves" "you" "your" "yours" "yourself" "yourselves"
    "she" "her" "hers" "herself" "it" "its" "itself" "they" "them" "their" 
    "which" "who" "whom" "whose" "this" "that" "these" "those" "am" "is" 
    "being" "have" "has" "had" "having" "do" "does" "did" "doing" "will"})

(defn get-words
  "get vector of maps with word as :key and count as :value"
  [state n]
  (vec (map
        (fn [w] (let [[k v] w] {:key k :value v}))
        (take n (:words-sorted-by-count state)))))

(defn get-words2
  "get vector of maps with word as :key and count as :value"
  [state n]
  (vec (take n (:words-sorted-by-count state))))

(defn words-in-tweet
  "process tweet: split, filter, lower case, replace punctuation, add word"
  [text]
  (->> (s/split text #"[\s—\u3031-\u3035\u0027\u309b\u309c\u30a0\u30fc\uff70]+")
       (filter #(not (re-find #"(@|https?:)" %)) ,)
       (filter #(> (count %) 3) ,)
       (filter #(< (count %) 25) ,)
       (map s/lower-case ,)
       (map #(s/replace % #"[;:,/‘’…~\-!?\[\]\"<>()\"@.]+" "" ) ,)
       (filter (fn [item] (not (contains? stop-words item))) ,)))
~~~

First, we define a set named ````stop-words````. It actually has many more entries but is shortened here so that it fits the page format better. These words are not counted towards the result as they aren't very interesting to look at in the word cloud and the bar chart.

Next, there is the ````get-words```` function which formats the wordcount data from the application state snapshot as required by the word cloud library by using a mapping function.

For the wordcount bar chart, which I was lucky enough to implement myself, I don't need to reformat the data; here ````get-words2```` retrieves the data from the application state as is.

Finally, the ````words-in-tweet```` function takes the text of a tweet, splits it, removes words that are too short or too long, converts them to lowercase, replaces a few character, filters out words contained in the ````stop-words```` set and finally returns a sequence of the relevant words it found.