# DRAFT: WebSocket Latency Visualization Example

Some time ago, I wrote an article about this sample application for the **[systems-toolbox](https://github.com/matthiasn/systems-toolbox)**, the one for visualizing **[WebSocket](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API)** round trip times. 

[Screenshot]

A live demo can be found **[here](http://systems-toolbox.matthiasnehlsen.com/)** and the code is **[here](https://github.com/matthiasn/systems-toolbox/tree/master/examples/trailing-mouse-pointer)**.

Now that I'm back at actively developing the systems-toolbox, it was a good time to update this application to be fully validated by **[clojure.spec](https://clojure.org/about/spec)**, and write about it. By the way, clojure.spec came at a crucial time for me. This kind of validation was really missing in the systems-toolbox, and made me question the whole approach after fighting with annoying bugs in my latest application, **[iWasWhere](https://github.com/matthiasn/iWasWhere)** (which I'll introduce in a subsequent chapter). But now with clojure.spec, these annoying bugs are gone for good. In a matter of a little over a week I upgraded all my applications, and the systems-toolbox libraries, to use clojure.spec and I'm now more convinced about the approach than ever. You really **can** build applications this way, and **stay sane** at the same time.

We'll look into validation in this chapter, too. But let's get started with the application itself. Here, we have a couple of different components:

On the client:

* there's a component that shows the position of the mouse, both locally and for the message coming back from the server
* there's the store, which holds the client-side state
* there's a UI component for visualizing the round trip times as histograms
* in addition, there are components for visualizing message flow, and for showing some JVM stats

On the server

* there's the store, which keeps a counter, and returns the message with the mouse pos to the client where it originated
* then, there's also a component for gathering some stats about the JVM

On both sides, there are **[Sente](https://github.com/ptaoussanis/sente)** components for establishing bi-directional communication between client and server. These ready-to-use components are provided by the **[systems-toolbox-sente](https://github.com/matthiasn/systems-toolbox-sente)** library.

The store component on the client is observed by both the histogram and the mouse move components; these two render something based on what's in the state of the store component.

The communication between these components is comparable to what was introduced in the previous chapter. What's new here is the `sente-cmp`. Let's have a brief look what **[WebSockets](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API)** are. They give us a way for establising a very low latency **bi-directional** connection between client and server. It's not HTTP but instead its own protocol. They are well supported from IE 10 on, and in all other recent browsers. Some critics may say that they may be problematic because firewalls and reverse proxies might have to reconfigured. Well, that might indeed problematic if (and only if) your OPS guys are incompetent. But most likely they won't be, so it's only a matter of communication (and some upfront planning) to get this out of the way.

Other than that potential issue, there appears to be no downside to using WebSockets, and plenty of upsides. You absolutely need to be able to send messages from server to client at any time if you want to build a modern, responsive UI. Sure, you could also use SSE for that route, and send messages from client to server the way you'd normally do: via REST calls, for each message. That may work; it may also be too expensive if you want to send messages often. For example here, with the mouse positions, the user experience would likely be quite poor.

WebSockets are also nice because, using them via the systems-toolbox, gives you an ordering guarantee, which would be much harder with REST calls.

Anyway, let's look at some code, starting where the messages originate in our example: the mouse move component:

```
(ns example.ui-mouse-moves
  (:require [reagent.core :as rc]
            [matthiasn.systems-toolbox-ui.reagent :as r]
            [matthiasn.systems-toolbox-ui.helpers :refer [by-id]]))

(def circle-defaults {:fill "rgba(255,0,0,0.1)" :stroke "black" :stroke-width 2 :r 15})
(def text-default {:stroke "none" :fill "black" :style {:font-size 12}})
(def text-bold (merge text-default {:style {:font-weight :bold :font-size 12}}))

(defn trailing-circles
  "Displays two transparent circles where one is drawn directly on the client and the other is drawn after a roundtrip.
  This makes it easier to experience any delays."
  [state local-state]
  (let [pos (:pos local-state)
        from-server (:from-server state)]
    [:g
     [:circle (merge circle-defaults {:cx (:x pos) :cy (:y pos)})]
     [:circle (merge circle-defaults {:cx (:x from-server) :cy (:y from-server) :fill "rgba(0,0,255,0.1)"})]]))

(defn text-view
  "Renders SVG with an area in which mouse moves are detected. They are then sent to the server and the round-trip
  time is measured."
  [state pos mean mn mx last-rt]
  [:g
   [:text (merge text-bold {:x 30 :y 20}) "Mouse Moves Processed:"]
   [:text (merge text-default {:x 183 :y 20}) (:count state)]
   [:text (merge text-bold {:x 30 :y 40}) "Processed since Startup:"]
   [:text (merge text-default {:x 183 :y 40}) (:count (:from-server state))]
   [:text (merge text-bold {:x 30 :y 60}) "Current Position:"]
   (when pos [:text (merge text-default {:x 137 :y 60}) (str "x: " (:x pos) " y: " (:y pos))])
   [:text (merge text-bold {:x 30 :y 80}) "Latency (ms):"]
   (when last-rt
     [:text (merge text-default {:x 115 :y 80}) (str mean " mean / " mn " min / " mx " max / " last-rt " last")])])

(defn mouse-touch-ev-handler
  "Handler function for mouse move and touch move events. Determines position from event and sends
  coordinates as :cmd/mouse-pos messages. Also sets position in the local atom."
  [local put-fn curr-cmp touch?]
  (fn [ev]
    (let [rect (-> curr-cmp rc/dom-node .getBoundingClientRect)
          ev (if touch? (aget (.-targetTouches ev) 0) ev)
          pos {:x (.toFixed (- (.-clientX ev) (.-left rect))) :y (.toFixed (- (.-clientY ev) (.-top rect)) 0)}]
      (swap! local assoc :pos pos)
      (put-fn [:cmd/mouse-pos pos])
      (.stopPropagation ev))))

(defn mouse-view
  "Renders SVG with an area in which mouse moves are detected. They are then sent to the server and the round-trip
  time is measured."
  [{:keys [observed local put-fn]}]
  (let [state-snapshot @observed
        mouse-div (by-id "mouse")
        last-rt (:rt-time (:from-server state-snapshot))
        rtt-times (:rtt-times state-snapshot)
        mx (apply max rtt-times)
        mn (apply min rtt-times)
        mean (/ (apply + rtt-times) (count rtt-times))
        update-width #(swap! local assoc :width (- (.-offsetWidth mouse-div) 2))]
    (update-width)
    (aset js/window "onresize" update-width)
    [:div {:style {:border-color :darkgray :border-width "1px" :border-style :solid}}
     [:svg {:width         (:width @local)
            :height        (:width @local)
            :style         {:background-color :white}
            :on-mouse-move (mouse-move-ev-handler local put-fn (rc/current-component))
            :on-touch-move (touch-move-ev-handler local put-fn (rc/current-component))}
      (text-view state-snapshot (:pos @local) (.toFixed mean 0) mn mx last-rt)
      (trailing-circles state-snapshot @local)]]))

(defn cmp-map
  [cmp-id]
  (r/cmp-map {:cmp-id  cmp-id
              :view-fn mouse-view
              :dom-id  "mouse"
              :cfg     {:msgs-on-firehose true}}))
```

Here, we have a UI component which emits messages whenever the user moves her mouse on top of it's div. Then, the rendering is actually done via SVG, which is trivial when you combine it with Reagent. You can see that we just create a group with two circle, each with a distinct position based on the last known message. fast movements will reveal the latency, as you'll see how the messages from the server lag behind.

Let's go through this function by function, starting from the bottom of the namespace:

```
(defn cmp-map
  [cmp-id]
  (r/cmp-map {:cmp-id  cmp-id
              :view-fn mouse-view
              :dom-id  "mouse"
              :cfg     {:msgs-on-firehose true}}))          
```

This function creates the component map, which is like a blueprint that tells the switchboard how to fire up the component. The UI part is done by calling `r/cmp-map`, which is the main function in the **systems-toolbox-ui** library. Once the returned map is sent to the switchboard, a component will be initialized that renders the `mouse-view` function into the **DOM element** with the `"mouse"` ID.

Next, let's have a look at the `mouse-view` function, which is responsible for rendering the UI component:

```
(defn mouse-view
  "Renders SVG with an area in which mouse moves are detected. They are then sent to the server and the round-trip
  time is measured."
  [{:keys [observed local put-fn]}]
  (let [state-snapshot @observed
        mouse-div (by-id "mouse")
        last-rt (:rt-time (:from-server state-snapshot))
        rtt-times (:rtt-times state-snapshot)
        mx (apply max rtt-times)
        mn (apply min rtt-times)
        mean (/ (apply + rtt-times) (count rtt-times))
        update-width #(swap! local assoc :width (- (.-offsetWidth mouse-div) 2))]
    (update-width)
    (aset js/window "onresize" update-width)
    [:div {:style {:border-color :darkgray :border-width "1px" :border-style :solid}}
     [:svg {:width         (:width @local)
            :height        (:width @local)
            :style         {:background-color :white}
            :on-mouse-move (mouse-move-ev-handler local put-fn (rc/current-component))
            :on-touch-move (touch-move-ev-handler local put-fn (rc/current-component))}
      (text-view state-snapshot (:pos @local) (.toFixed mean 0) mn mx last-rt)
      (trailing-circles state-snapshot @local)]]))
```

Note that this component gets passed a map with the `observed`, `local`, and `put-fn` keys. The `observed` key is an atom which holds the state of an observed component. Here, this is always the latest snapshot of the `store-cmp`. The `local` atom contains some local state, such as the width of the SVG for resizing.

[hmm, maybe the mouse-pos should not be recorded in local atom]

Finally, the `put-fn` is used to emit a message when an event, either `:on-mouse-move` or `:on-touch-move`, is detected. Note that we're detecting the width on every call to the function, and also in the `onresize` callback of `js/window`. This ensures that the square mouse div fills the parent element, while working with the correct pixel coordinate system. One could instead also work with a viewBox, like this: `{:width "100%" :viewBox "0 0 1000 1000"}`. However, that would not work correctly as the mouse position would not be aligned with the circles here.

Next, let's look at the `mouse-touch-ev-handler` function:

```
(defn mouse-touch-ev-handler
  "Handler function for mouse move and touch move events. Determines position from event and sends
  coordinates as :cmd/mouse-pos messages. Also sets position in the local atom."
  [local put-fn curr-cmp touch?]
  (fn [ev]
    (let [rect (-> curr-cmp rc/dom-node .getBoundingClientRect)
          ev (if touch? (aget (.-targetTouches ev) 0) ev)
          pos {:x (.toFixed (- (.-clientX ev) (.-left rect))) :y (.toFixed (- (.-clientY ev) (.-top rect)) 0)}]
      (swap! local assoc :pos pos)
      (put-fn [:cmd/mouse-pos pos])
      (.stopPropagation ev))))
```

This higher-order function returns an event handling function that is called for both `:on-mouse-move` and `:on-touch-move` events. These are almost the same, except that we want the first item in the **[TouchList](https://developer.mozilla.org/en-US/docs/Web/API/TouchList)** that we can get a hold off inside the **[targetTouches](https://developer.mozilla.org/en-US/docs/Web/API/TouchEvent/targetTouches)** property of the event. Because of the similarity, we can use the same code and only change `ev` in the let-binding, like this `(if touch? (aget (.-targetTouches ev) 0) ev)`.

Next, there's the `text-view` function:

```
(defn text-view
  "Renders SVG with an area in which mouse moves are detected. They are then sent to the server and the round-trip
  time is measured."
  [state pos mean mn mx last-rt]
  [:g
   [:text (merge text-bold {:x 30 :y 20}) "Mouse Moves Processed:"]
   [:text (merge text-default {:x 183 :y 20}) (:count state)]
   [:text (merge text-bold {:x 30 :y 40}) "Processed since Startup:"]
   [:text (merge text-default {:x 183 :y 40}) (:count (:from-server state))]
   [:text (merge text-bold {:x 30 :y 60}) "Current Position:"]
   (when pos [:text (merge text-default {:x 137 :y 60}) (str "x: " (:x pos) " y: " (:y pos))])
   [:text (merge text-bold {:x 30 :y 80}) "Latency (ms):"]
   (when last-rt
     [:text (merge text-default {:x 115 :y 80}) (str mean " mean / " mn " min / " mx " max / " last-rt " last")])])
```

This one is very straight-forward. There's an **[SVG group](https://developer.mozilla.org/en/docs/Web/SVG/Element/g)** with some text at different positions inside.

Finally, we have the `trailing-circles` function:

```
(defn trailing-circles
  "Displays two transparent circles where one is drawn directly on the client and the other is drawn after a roundtrip.
  This makes it easier to experience any delays."
  [state local-state]
  (let [pos (:pos local-state)
        from-server (:from-server state)]
    [:g
     [:circle (merge circle-defaults {:cx (:x pos) :cy (:y pos)})]
     [:circle (merge circle-defaults {:cx (:x from-server) :cy (:y from-server) :fill "rgba(0,0,255,0.1)"})]]))
```

This one renders an SVG group with the two circles inside. Then finally, there are some defaults fro the different elements, which can be merged with more specific maps as desired:

```
(def circle-defaults {:fill "rgba(255,0,0,0.1)" :stroke "black" :stroke-width 2 :r 15})
(def text-default {:stroke "none" :fill "black" :style {:font-size 12}})
(def text-bold (merge text-default {:style {:font-weight :bold :font-size 12}}))
```

That's it for the rendering of the mouse element. Next, let's discuss the server side, before looking into the wiring of the components. It's really short, this is the entire namespace:

```
(ns example.pointer
  "This component receives messages, keeps a counter, decorates them with the state of the counter, and sends
  them back. Here, this provides a way to measure roundtrip time from the UI, as timestamps are recorded as
  the message flows through the system.")

(defn process-mouse-pos
  "Handler function for received mouse positions, increments counter and returns mouse position to sender."
  [{:keys [current-state msg-meta msg-payload]}]
  (let [new-state (update-in current-state [:count] inc)]
    {:new-state new-state
     :emit-msg (with-meta [:cmd/mouse-pos (assoc msg-payload :count (:count new-state))] msg-meta)}))

(defn cmp-map
  [cmp-id]
  {:cmp-id      cmp-id
   :state-fn    (fn [_] {:state (atom {:count 0})})
   :handler-map {:cmd/mouse-pos process-mouse-pos}})
```

At the bottom, you see the `cmp-map`, which again is the map specifiying the component that the switchboard will then instantiate. Inside, there's the `:state-fn`, which does nothing but create the initial state inside an atom. Then, there's the `:handler-map`, which here only handles a single message type `:cmd/mouse-pos`.

The `process-mouse-pos` handler function then gets the `current-state`, the `msg-payload`, and the `msg-meta` inside the map it gets passed as a single argument, and returns both the `:new-state` and a message to emit, which is the same message it received, only now enriched by the `:count` from this component's state. Note that we are reusing the `msg-meta` from the original message, as this metadata also contains the `:sente-id` of the originating client, which is required to route the message back to the correct client. There's more information on the metadata, we'll get to that later.

Next, the messages need to get from the UI component to the server, and back to the client. Here's how that looks like:

[message flow drawing]

For establishing these connections, let's have a look at the `core` namespaces on both server and client, starting with the client:

```
(ns example.core
  (:require [example.spec]
            [example.store :as store]
            [example.ui-histograms :as hist]
            [example.ui-mouse-moves :as mouse]
            [example.conf :as conf]
            [matthiasn.systems-toolbox-ui.charts.observer :as obs]
            [matthiasn.systems-toolbox.switchboard :as sb]
            [matthiasn.systems-toolbox-sente.client :as sente]
            [matthiasn.systems-toolbox-metrics.jvmstats :as jvmstats]))

(enable-console-print!)

(defonce switchboard (sb/component :client/switchboard))

(defn init! []
  (sb/send-mult-cmd
    switchboard
    [[:cmd/init-comp
      #{(sente/cmp-map :client/ws-cmp ;  WebSocket communication component
                       {:relay-types #{:cmd/mouse-pos} :msgs-on-firehose true})
        (mouse/cmp-map :client/mouse-cmp) ; UI component for capturing mouse moves
        (store/cmp-map :client/store-cmp)                           ; Data store component
        (hist/cmp-map :client/histogram-cmp)                        ; histograms component
        (jvmstats/cmp-map :client/jvmstats-cmp "jvm-stats-frame")   ;  UI component: JVM stats
        (obs/cmp-map :client/observer-cmp conf/observer-cfg-map)}]  ; UI component for observing system
     ;; Then, messages of a given type are wired from one component to another
     [:cmd/route {:from :client/mouse-cmp :to :client/ws-cmp}]
     [:cmd/route {:from :client/ws-cmp :to #{:client/store-cmp :client/jvmstats-cmp}}]
     [:cmd/observe-state {:from :client/store-cmp :to #{:client/mouse-cmp :client/histogram-cmp}}]

     ;; Finally, wire firehose with all messages into the observer component
     [:cmd/attach-to-firehose :client/observer-cmp]]))

(init!)
```

First, as usual, we create a `switchboard`. Then, we send messages to the switchboard, with the blueprints for the components we need initialized. For the core functionality discussed so far, only three of them are important: `:client/ws-cmp`, `:client/mouse-cmp` and `:client/store-cmp`.

