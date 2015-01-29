### The birdwatch.ui.util namespace
In the code for the Reagent components, we have used a couple of helpers from the ````birdwatch.ui.util```` **[namespace](https://github.com/matthiasn/BirdWatch/blob/aae1a7313211f26946ae55278fb8d3c484060c70/Clojure-Websockets/MainApp/src/cljs/birdwatch/ui/util.cljs)**, all of which are pure functions:

~~~
(ns birdwatch.ui.util
  (:require [clojure.string :as s]))

(defn by-id
  "Helper function, gets DOM element by ID."
  [id]
  (.getElementById js/document id))

(defn number-format
  "Formats a number for display, e.g. 1.7K, 122K or 1.5M followers."
  [number]
  (cond
   (< number 1000) (str number)
   (< number 100000) (str (/ (.round js/Math (/ number 100)) 10) "K")
   (< number 1000000) (str (.round js/Math (/ number 1000)) "K")
   :default (str (/ (.round js/Math (/ number 100000)) 10) "M")))

(defn from-now
  "Formats a date using the external moment.js library."
  [date]
  (let [time-string (. (js/moment. date) (fromNow true))]
    (if (= time-string "a few seconds") "just now" time-string)))

(def twitter-url "https://twitter.com/")

(defn a-blank
  "Creates HTML string for a link that opens in a new tab in the browser."
  [url text]
  (str "<a href='" url "' target='_blank'>" text "</a>"))

(defn- url-replacer
  "Replace URL occurences in tweet texts with HTML (including links)."
  [acc entity]
  (s/replace acc (:url entity) (a-blank (:url entity) (:display_url entity))))

(defn- hashtags-replacer
  "Replaces hashtags in tweet text with HTML (including links)."
  [acc entity]
  (let [hashtag (:text entity)
        f-hashtag (str "#" hashtag)]
    (s/replace acc f-hashtag (a-blank (str twitter-url "search?q=%23" hashtag) f-hashtag))))

(defn- mentions-replacer
  "Replaces user mentions in tweet text with HTML (including links)."
  [acc entity]
  (let [screen-name (:screen_name entity)
        f-screen-name (str "@" screen-name)]
    (s/replace acc f-screen-name (a-blank (str twitter-url screen-name) f-screen-name))))

(defn- reducer
  "Generic reducer, allows calling specified function for each item in provided collection."
  [text coll fun]
  (reduce fun text coll))

(defn format-tweet
  "Formats tweet text for display by running multiple reducers."
  [tweet]
  (let [{:keys [urls media user_mentions hashtags]} (:entities tweet)]
    (assoc tweet :html-text
      (-> (:text tweet)
          (reducer , urls url-replacer)
          (reducer , media url-replacer)
          (reducer , user_mentions mentions-replacer)
          (reducer , hashtags hashtags-replacer)
          (s/replace , "RT " "<strong>RT </strong>")))))

(defn entity-count
  "Gets count of specified entity from either tweet, or, when exists, original (retweeted) tweet."
  [tweet state k s]
  (let [rt-id (if (contains? tweet :retweeted_status)
                (:id_str (:retweeted_status tweet))
                (:id_str tweet))
        count (k ((keyword rt-id) (:tweets-map state)))]
    (if count (str (number-format count) s) "")))

(defn rt-count
  "Gets the formatted string for the :retweet_count if exists, otherwise yields empty string."
  [tweet state]
  (entity-count tweet state :retweet_count " RT | "))

(defn fav-count
  "Gets the formatted string for the :favorite_count if exists, otherwise yields empty string."
  [tweet state]
  (entity-count tweet state :favorite_count " fav"))

(defn rt-count-since-startup
  "Gets RT count since startup for tweet, if exists returns formatted string."
  [tweet state]
  (let [t (if (contains? tweet :retweeted_status)
            (:retweeted_status tweet)
            tweet)
        cnt ((keyword (:id_str t)) (:by-rt-since-startup state))
        reach ((keyword (:id_str t)) (:by-reach state))]
    (when (> cnt 0)
      (str "analyzed: " (number-format cnt) " retweets, reach " (number-format reach)))))

(defn tweets-by-order
  "Finds top n tweets by specified order."
  [order state n skip]
  (map (fn [[k v]] (get (:tweets-map state) k {:id_str (name k)}))
       (->> (order state)
            (drop (* n skip) ,)
            (take n ,))))
~~~

I don't know, for my taste, this namespace is almost too long in terms of the number of lines. What do you think? Anyway, let's go through the code function by function. First, we have the ````by-id```` function:

~~~
(defn by-id
  "Helper function, gets DOM element by ID."
  [id]
  (.getElementById js/document id))
~~~

This function is very straightforward, despite the interaction with the JavaScript host platform. All it does it get a DOM element by ID. Next, we have the ````number-format```` function:

~~~
(defn number-format
  "Formats a number for display, e.g. 1.7K, 122K or 1.5M followers."
  [number]
  (cond
   (< number 1000) (str number)
   (< number 100000) (str (/ (.round js/Math (/ number 100)) 10) "K")
   (< number 1000000) (str (.round js/Math (/ number 1000)) "K")
   :default (str (/ (.round js/Math (/ number 100000)) 10) "M")))
~~~

You know what this does if you've ever used Twitter. It reduces large numbers to fewer significant figures. Take Justin Bieber for example. There's simply not enough space to display his number of followers, which was **59,776,559** at the time of writing this. It is so much easier to squeeze in **59.8M** instead.

~~~
(defn from-now
  "Formats a date using the external moment.js library."
  [date]
  (let [time-string (. (js/moment. date) (fromNow true))]
    (if (= time-string "a few seconds") "just now" time-string)))
~~~

The result of the ````from-now```` function should be equally familiar. It shows "just now" when a timestamp was, well, just now. Or "30 min" or "12 hours" or whatever. You get the idea. If you want to know more about the behavior, the **[moment.js](http://momentjs.com)** documentation has you covered. This function is just a thin wrapper over it.

~~~
(def twitter-url "https://twitter.com/")

(defn a-blank
  "Creates HTML string for a link that opens in a new tab in the browser."
  [url text]
  (str "<a href='" url "' target='_blank'>" text "</a>"))
~~~

The ````a-blank```` returns an HTML string for a link that opens in a new tab. The tweet text is constructed as an HTML string as a whole and passed into the ````tweet-text```` component's div. Remember from the last chapter? Here's a reminder, it involved dangerously setting something:

~~~
[:div {:dangerouslySetInnerHTML #js {:__html (:html-text tweet)}}]
~~~

One could likely implement this in pure Reagent, without this "dangerous" behavior. I don't mind it as injections of malicious code don't seem to be an issue in this application. However, it would be more elegant. Anyone interested in solving this? Maybe that could be a task to solve during a meetup or so. If you're interested, knock yourself out.

The ````a-blank```` function we've seen a moment ago is then used by the ````url-replacer````, ````hashtags-replacer````, and ````mentions-replacer```` functions:

~~~
(defn- url-replacer
  "Replace URL occurences in tweet texts with HTML (including links)."
  [acc entity]
  (s/replace acc (:url entity) (a-blank (:url entity) (:display_url entity))))
~~~

~~~
(defn- hashtags-replacer
  "Replaces hashtags in tweet text with HTML (including links)."
  [acc entity]
  (let [hashtag (:text entity)
        f-hashtag (str "#" hashtag)]
    (s/replace acc f-hashtag (a-blank (str twitter-url "search?q=%23" hashtag) f-hashtag))))
~~~

~~~
(defn- mentions-replacer
  "Replaces user mentions in tweet text with HTML (including links)."
  [acc entity]
  (let [screen-name (:screen_name entity)
        f-screen-name (str "@" screen-name)]
    (s/replace acc f-screen-name (a-blank (str twitter-url screen-name) f-screen-name))))
~~~

The three _replacer_ functions we just looked at obviously need to be called from somewhere. I wanted to use the **[thread-first macro](http://clojuredocs.org/clojure.core/-%3E)** and pass the result from one ````reduce```` into the next ````reduce```` until all replacements have been performed. However, the signature of ````reduce```` did not match. Not to worry, I can just write a function for that:

~~~
(defn- reducer
  "Generic reducer, allows calling specified function for each item in provided collection."
  [text coll fun]
  (reduce fun text coll))
~~~

In the ````reducer```` function above, reduce is called internally, while providing the signature I need to work with either the ````thread-first```` or the ````thread-last```` macro. This ````reducer```` can now be called from inside the function that formats the entire tweet text HTML string:

~~~
(defn format-tweet
  "Formats tweet text for display by running multiple reducers."
  [tweet]
  (let [{:keys [urls media user_mentions hashtags]} (:entities tweet)]
    (assoc tweet :html-text
      (-> (:text tweet)
          (reducer , urls url-replacer)
          (reducer , media url-replacer)
          (reducer , user_mentions mentions-replacer)
          (reducer , hashtags hashtags-replacer)
          (s/replace , "RT " "<strong>RT </strong>")))))
~~~

So much for the tweet text. But we have more to discuss. We need to get the counts of multiple entities, such as the _followers_, _retweet_, _favorites_, or also the _retweets within the loaded tweets_:

~~~
(defn entity-count
  "Gets count of specified entity from either tweet, or, when exists, original (retweeted) tweet."
  [tweet state k s]
  (let [rt-id (if (contains? tweet :retweeted_status)
                (:id_str (:retweeted_status tweet))
                (:id_str tweet))
        count (k ((keyword rt-id) (:tweets-map state)))]
    (if count (str (number-format count) s) "")))
~~~

This ````entity-count```` function takes a ````tweet````, the ````state```` snapshot, a keyword ````k```` and a string ````s````. It then checks if the ````tweet```` is a retweet (contains a ````:retweeted-status```` ). If so, it uses the ID of the retweet, otherwise it takes the ID of ````tweet```` as ````rt-id````. Then, ````count```` is determined by looking up the ````rt-id```` within the ````state```` snapshot, using ````k```` as the lookup function. Finally, if ````count```` exists, it concatenates the result of calling the ````number-format```` function with ````count```` and the string ````s````. This ````entity-count```` function can now be used more specifically:

~~~
(defn rt-count
  "Gets the formatted string for the :retweet_count if exists, otherwise yields empty string."
  [tweet state]
  (entity-count tweet state :retweet_count " RT | "))

(defn fav-count
  "Gets the formatted string for the :favorite_count if exists, otherwise yields empty string."
  [tweet state]
  (entity-count tweet state :favorite_count " fav"))
~~~

Both ````rt-count```` and ````fav-count```` simply call ````entity-count```` with the respective keywords and strings. The ````rt-count-since-startup```` is a little more involved:

~~~
(defn rt-count-since-startup
  "Gets RT count since startup for tweet, if exists returns formatted string."
  [tweet state]
  (let [t (if (contains? tweet :retweeted_status)
            (:retweeted_status tweet)
            tweet)
        cnt ((keyword (:id_str t)) (:by-rt-since-startup state))
        reach ((keyword (:id_str t)) (:by-reach state))]
    (when (> cnt 0)
      (str "analyzed: " (number-format cnt) " retweets, reach " (number-format reach)))))
~~~

Once again, we look if the tweet is a retweet. If so, further reasoning is done on the retweet, if not, the tweet itself is used. Then, we can look up the number of retweets within the loaded tweets and the reach within the same dataset by looking up the the count within the respective sort orders which we have derived and updated when ingesting tweets.

Finally, there is a function that returns a list of tweets for display in the UI, paginated, with the correct number of items and in the right sort order:

~~~
(defn tweets-by-order
  "Finds top n tweets by specified order."
  [order state n skip]
  (map (fn [[k v]] (get (:tweets-map state) k {:id_str (name k)}))
       (->> (order state)
            (drop (* n skip) ,)
            (take n ,))))
~~~

