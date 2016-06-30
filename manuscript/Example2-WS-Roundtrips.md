# WebSocket Latency Visualization Example

Communication between backend and web applications via **[WebSockets](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API)** is an integral part of delivering a rich user experience. With that addition, it is much easier to push new information to the user at any given time, without having to resort to constant polling. 

But how fast IS that communication? Let's find out. The next sample application for the **[systems-toolbox](https://github.com/matthiasn/systems-toolbox)** deals with just that, visualizing the latency for messages sent from client to server and back.

![Screenshot](images/ws-lat-screenshot.png "Screenshot")

Check out the live demo **[here](http://systems-toolbox.matthiasnehlsen.com/)**; it'll give you some additional information about the application. There was a previous version of this example, but with **[clojure.spec](https://clojure.org/about/spec)** released, it was a good time to revisit this application and have it fully validated.

By the way, **[clojure.spec](https://clojure.org/about/spec)** came at a **crucial time** for me. This kind of validation was missing in the systems-toolbox (and more broadly in Clojure, too), and that made me question the whole approach, especially after fighting with annoying bugs in my latest application, **[iWasWhere](https://github.com/matthiasn/iWasWhere)** (which I'll introduce in a subsequent chapter). But now with clojure.spec, the entire class of those annoying bugs is gone for good. In a matter of a little over a week, I upgraded all my applications plus the systems-toolbox libraries to use clojure.spec, and I'm now more convinced about the approach than ever. You **can** build applications this way, and **stay sane** at the same time.

We'll look into validation in this chapter, too. But let's get started with the application itself. Here, we have a couple of different components:

On the client:

* there's the `:client/mouse-cmp` component that shows the position of the mouse, both locally and for the message coming back from the server
* there's the `:client/store-cmp`, which holds the client-side state
* there's the `:client/histogram-cmp` UI component for visualizing the round trip times as histograms
* there's `:client/info-cmp` UI component that shows some information about the app
* also, there are components for visualizing message flow, and for showing some JVM stats: `:client/observer-cmp` and `:client/jvmstats-cmp`

On the server:

* there's the `:server/ptr-cmp` component, which keeps a counter of all messages passed through since application startup, and returns each mouse position message to the client where it originated, plus, upon request, a history of mouse positions from all connected clients
* then, there's also the `:server/metrics-cmp` component for gathering some stats about the JVM, which get broadcast to all connected clients

On both sides, there are **[Sente](https://github.com/ptaoussanis/sente)** components for establishing **bi-directional communication** between client and server. These ready-to-use components are provided by the **[systems-toolbox-sente](https://github.com/matthiasn/systems-toolbox-sente)** library, and you can use them in your projects, too, with a simple import and no more than a handful of lines of code.

The store component on the client, which holds the client-side state, is then observed by the histogram, the mouse moves, and the info components; these three render something based on what's in the state that they observe.

The communication between these components is comparable to what was introduced in the previous chapter. What's new here is the `sente-cmp`. Let's have a brief look what **[WebSockets](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API)** are. They give us a way to establish a very low latency **bi-directional** connection between client and server. It's not HTTP but instead its own protocol. WebSockets are well supported from IE 10 on, and in all other recent browsers. Some critics say that they may be problematic because firewalls and reverse proxies might have to be reconfigured. Well, that might indeed problematic if (and only if) your OPS people are incompetent. But most likely they are not, so it's only a matter of communication (and some upfront planning) to get this potential hurdle out of the way.

Other than that potential issue with your firewall, there appears to be no downside to using **WebSockets**, and plenty of upsides. You absolutely need to be able to send messages from server to client at any time if you want to build a modern, responsive UI. Sure, you could also use **[Server-sent Events (SSE)](https://en.wikipedia.org/wiki/Server-sent_events)** for the server -> client direction, and send messages from client to server the way you'd normally do: via REST calls, for each message. That may work; it may also be too expensive if you want to send messages often. For the use case in this very example, with the mouse positions, the user experience would likely be quite poor.

WebSockets are also nice because you get an ordering guarantee, which would be much harder with REST calls. Another aspect not to underestimate is that with REST calls, you need to think about authentication on every single request, where you do it once for a WebSockets connection.


## :client/mouse-cmp

Anyway, let's look at some code, starting with where the messages originate in our example, the `:client/mouse-cmp` component, and its respective **[namespace](https://github.com/matthiasn/systems-toolbox/blob/master/examples/trailing-mouse-pointer/src/cljs/example/ui_mouse_moves.cljs)**:

~~~
(ns example.ui-mouse-moves
  (:require [matthiasn.systems-toolbox-ui.reagent :as r]
            [matthiasn.systems-toolbox-ui.helpers :refer [by-id]]))

;; some SVG defaults
(def circle-defaults {:fill "rgba(255,0,0,0.1)
" :stroke "rgba(0,0,0,0.5)"
                      :stroke-width 2 :r 15})
(def text-default {:stroke "none" :fill "black" :style {:font-size 12}})
(def text-bold (merge text-default {:style {:font-weight :bold :font-size 12}}))

(defn mouse-hist-view
  "Render SVG group with filled circles from a vector of mouse positions in state."
  [state state-key stroke fill]
  (let [positions (map-indexed vector (state-key state))]
    (when (seq positions)
      [:g {:opacity 0.5}
       (for [[idx pos] positions]
         ^{:key (str "circle" state-key idx)}
         [:circle {:stroke       stroke
                   :stroke-width 2
                   :r            15
                   :cx           (:x pos)
                   :cy           (:y pos)
                   :fill         fill}])])))

(defn trailing-circles
  "Displays two transparent circles. The position of the circles comes from the most recent
  messages, one sent locally and the other with a roundtrip to the server in between.  This
  makes it easier to visually detect any delays."
  [state]
  (let [local-pos (:local state)
        from-server (:from-server state)]
    [:g
     [:circle (merge circle-defaults {:cx (:x local-pos)
                                      :cy (:y local-pos)})]
     [:circle (merge circle-defaults {:cx (:x from-server)
                                      :cy (:y from-server)
                                      :fill "rgba(0,0,255,0.1)"})]]))

(defn mouse-view
  "Renders SVG with both local mouse position and the last one returned from the server,
  in an area that covers the entire visible page."
  [{:keys [observed local]}]
  (let [state-snapshot @observed
        mouse-div (by-id "mouse")
        update-dim #(do (swap! local assoc :width (- (.-offsetWidth mouse-div) 2))
                        (swap! local assoc :height (aget js/document "body" "scrollHeight")))]
    (update-dim)
    (aset js/window "onresize" update-dim)
    [:div
     [:svg {:width  (:width @local)
            :height (:height @local)}
      (trailing-circles state-snapshot)
      (when (-> state-snapshot :show-all :local)
        [mouse-hist-view state-snapshot :local-hist "rgba(0,0,0,0.06)" "rgba(0,255,0,0.05)"])
      (when (-> state-snapshot :show-all :server)
        [mouse-hist-view state-snapshot :server-hist "rgba(0,0,0,0.06)" "rgba(0,0,128,0.05)"])]]))

(defn init-fn
  "Listen to onmousemove events for entire page, emit message when fired.
  These events are then sent to the server for measuring the round-trip time,
  and also recorded in the local application state for showing the local mouse
  position."
  [{:keys [put-fn]}]
  (aset js/window "onmousemove"
        #(put-fn [:mouse/pos {:x (.-pageX %) :y (.-pageY %)}]))
  (aset js/window "ontouchmove"
        (fn [ev]
          (let [t (aget (.-targetTouches ev) 0)]
            (put-fn [:mouse/pos {:x (.-pageX t) :y (.-pageY t)}])
            #_(.preventDefault ev)))))

(defn cmp-map
  "Configuration map for systems-toolbox-ui component."
  [cmp-id]
  (r/cmp-map {:cmp-id  cmp-id
              :view-fn mouse-view
              :dom-id  "mouse"
              :init-fn init-fn
              :cfg     {:msgs-on-firehose true}}))
~~~

Here, we have a UI component that covers the entire page. This is facilitated by the following **CSS**:

~~~
#mouse {
    position: absolute;
    top: 0;
    width: 100%;
    pointer-events: none;
    z-index: 10;
    margin-left: -12.5%;
}
~~~

Note that we want this transparent element on top, covering the rest of the page, which is what the `z-index` does. Also, we want `pointer-events` to reach the elements below, for example for clicking links or buttons, so we set them to `none`.

Then, in the `init-fn`, we set `ontouchmove` and `ontouchmove` event handlers, which get called when these events are fired anywhere on the page. We could also more specifically handle these events in the component's div, but then the `pointer-events` would not be available for elements below the `mouse-view` element, such as for clicking a button. Then, whenever an event is fired, a messaged is sent with the mouse position. This message will be received by the client side store directly, and also via the server side, where it'll be enriched with some additional data.

Then, the rendering of the **[SVG](https://www.w3.org/Graphics/SVG/)** covering the entire page is done in the `mouse-view` function, which adapts the size of the element when `onresize` element fires. Here, the `trailing-circles` function is called, which renders the two circles. This SVG rendering is trivial to achieve with Reagent. You can see that we just create a group with two circles, each with a distinct position based on the last known message. Fast movements will then reveal latency, as you'll see how the messages coming back from the server are lagging behind. Then, there are two calls to the `mouse-hist-view` function, which renders either a local history or the last moves of all clients, as you hopefully have seen when playing around with the live demo of the application. If not, here's what that looks like:

![Screenshot](images/ws-lat-screenshot2.png "Screenshot")

In the screenshot above, you can see green circles for the mouse moves captured locally, and charcoal ones for those from all clients.

Let's go through the **[namespace](https://github.com/matthiasn/systems-toolbox/blob/master/examples/trailing-mouse-pointer/src/cljs/example/ui_mouse_moves.cljs)**, function by function, starting from the bottom:

~~~
(defn cmp-map
  "Configuration map for systems-toolbox-ui component."
  [cmp-id]
  (r/cmp-map {:cmp-id  cmp-id
              :view-fn mouse-view
              :dom-id  "mouse"
              :init-fn init-fn
              :cfg     {:msgs-on-firehose true}}))      
~~~

The `cmp-map` function creates the component map, which is like a blueprint that tells the switchboard how to fire up the component. The **UI** part is done by calling `r/cmp-map`, which is the main function in the **systems-toolbox-ui** library. Once the returned map is sent to the switchboard, a component will be initialized that renders the `mouse-view` function into the **DOM element** with the `"mouse"` ID.

Then, there's the `init-fn`:

~~~
(defn init-fn
  "Listen to onmousemove events for entire page, emit message when fired.
  These events are then sent to the server for measuring the round-trip time,
  and also recorded in the local application state for showing the local mouse
  position."
  [{:keys [put-fn]}]
  (aset js/window "onmousemove"
        #(put-fn [:mouse/pos {:x (.-pageX %) :y (.-pageY %)}]))
  (aset js/window "ontouchmove"
        (fn [ev]
          (let [t (aget (.-targetTouches ev) 0)]
            (put-fn [:mouse/pos {:x (.-pageX t) :y (.-pageY t)}])
            #_(.preventDefault ev)))))
~~~

This function takes care of registering handler functions for all mouse movements (and also touch movement, for that matter) for the entire window. By doing that here, for the entire window, we can get away with the `mouse-view` element not getting any mouse movement events, which is required for still reacting to clicks in elements that are in fact covered by it, since it spans the entire page. When such an event is encountered, a `:mouse/pos` message is sent, which then happens to be received by both the `:client/store-cmp` and the `:server/pos-cmp`. Not that this component needs to be concerned with that in any way, though - there's proper decoupling between them.

You can see how those messages are supposed to look like in the respective **specs**:

~~~
(s/def :ex/x pos-int?)
(s/def :ex/y pos-int?)

(s/def :mouse/pos
  (s/keys :req-un [:ex/x :ex/y]))
~~~

If you still haven't heard Rich Hickey talk about **[clojure.spec](http://clojure.org/about/spec)** on the **[Cognicast](http://blog.cognitect.com/cognicast/103)**, you seriously need to do that now. **clojure.spec** has many useful properties. Among them is that you'll immediately know if you've broken your application with some recent change, as the system would throw an error immediately, rather than drag that problem along and blow up in your face somewhere else, where you'll have a hard time figuring out where it originated. What's also very useful is that when you come back to some code you wrote some time ago and wanted to know what a message is supposed to look like, you don't have to print it out and infer what the rules may be. No, instead you just look at the piece of code that's run when validating the message, it'll tell you all nitty-gritty details of what the expectations are. Much nicer.

Next, let's have a look at the `mouse-view` function, which is responsible for rendering the UI component:

~~~
(defn mouse-view
  "Renders SVG with both local mouse position and the last one returned from the server,
  in an area that covers the entire visible page."
  [{:keys [observed local]}]
  (let [state-snapshot @observed
        mouse-div (by-id "mouse")
        update-dim #(do (swap! local assoc :width (- (.-offsetWidth mouse-div) 2))
                        (swap! local assoc :height (aget js/document "body" "scrollHeight")))]
    (update-dim)
    (aset js/window "onresize" update-dim)
    [:div
     [:svg {:width  (:width @local)
            :height (:height @local)}
      (trailing-circles state-snapshot)
      (when (-> state-snapshot :show-all :local)
        [mouse-hist-view state-snapshot :local-hist "rgba(0,0,0,0.06)" "rgba(0,255,0,0.05)"])
      (when (-> state-snapshot :show-all :server)
        [mouse-hist-view state-snapshot :server-hist "rgba(0,0,0,0.06)" "rgba(0,0,128,0.05)"])]]))
~~~

Note that this component gets passed a map with the `observed` and `local` keys. The `observed` key is an atom which holds the state of the component it observes. Here, this is always the latest snapshot of the `store-cmp`. The `local` atom contains some local state, such as the width of the SVG for resizing. Note that we're detecting the width on every call to the function, and also in the `onresize` callback of `js/window`. This ensures that the mouse div fills the entire page, while working with the correct pixel coordinate system. One could instead also use a viewBox, like this: `{:width "100%" :viewBox "0 0 1000 1000"}`. However, that would not work correctly in this case as the mouse position would not be aligned with the circles here.

Next, we have the `trailing-circles` function:

~~~
(defn trailing-circles
  "Displays two transparent circles. The position of the circles comes from the most recent
  messages, one sent locally and the other with a roundtrip to the server in between.  This
  makes it easier to visually detect any delays."
  [state]
  (let [local-pos (:local state)
        from-server (:from-server state)]
    [:g
     [:circle (merge circle-defaults {:cx (:x local-pos)
                                      :cy (:y local-pos)})]
     [:circle (merge circle-defaults {:cx (:x from-server)
                                      :cy (:y from-server)
                                      :fill "rgba(0,0,255,0.1)"})]]))
~~~

This one renders an SVG group with the two circles inside. Then, there are some defaults for the different elements, which can be merged with more specific maps as desired:

~~~
(def circle-defaults {:fill "rgba(255,0,0,0.1)" :stroke "black" :stroke-width 2 :r 15})
(def text-default {:stroke "none" :fill "black" :style {:font-size 12}})
(def text-bold (merge text-default {:style {:font-weight :bold :font-size 12}}))
~~~

Finally, there's the `mouse-hist-view` function:

~~~
(defn mouse-hist-view
  "Render SVG group with filled circles from a vector of mouse positions in state."
  [state state-key stroke fill]
  (let [positions (map-indexed vector (state-key state))]
    (when (seq positions)
      [:g {:opacity 0.5}
       (for [[idx pos] positions]
         ^{:key (str "circle" state-key idx)}
         [:circle {:stroke       stroke
                   :stroke-width 2
                   :r            15
                   :cx           (:x pos)
                   :cy           (:y pos)
                   :fill         fill}])])))
~~~

Here, the history of mouse movements is rendered, either for your local mouse movements, or the last 1000 from all users. You've seen how that looks like in the screenshot above.


## :server/ptr-cmp

That's it for the rendering of the mouse element. The messages emitted there then get sent both to the client-side and the server-side store components. Let's discuss the server side first, before looking into the wiring of the components. It's really short; this is the entire **[example.pointer](https://github.com/matthiasn/systems-toolbox/blob/master/examples/trailing-mouse-pointer/src/cljc/example/pointer.cljc)** namespace:

~~~
(ns example.pointer
  "This component receives messages, keeps a counter, decorates them with the state of the
  counter, and sends them back. Here, this provides a way to measure roundtrip time from the UI,
  as timestamps are recorded as the message flows through the system.
  Also records a recent history of mouse positions for all clients, which the component provides
  to clients upon request.")

(defn process-mouse-pos
  "Handler function for received mouse positions, increments counter and returns mouse position
  to sender."
  [{:keys [current-state msg-meta msg-payload]}]
  (let [new-state (-> current-state
                      (update-in [:count] inc)
                      (update-in [:mouse-moves] #(vec (take-last 1000 (conj % msg-payload)))))]
    {:new-state new-state
     :emit-msg (with-meta [:mouse/pos (assoc msg-payload :count (:count new-state))] msg-meta)}))

(defn get-mouse-hist
  "Gets the recent mouse position history from server."
  [{:keys [current-state msg-meta]}]
  {:emit-msg (with-meta [:mouse/hist (:mouse-moves current-state)] msg-meta)})

(defn cmp-map
  [cmp-id]
  {:cmp-id      cmp-id
   :state-fn    (fn [_] {:state (atom {:count 0 :mouse-moves []})})
   :handler-map {:mouse/pos      process-mouse-pos
                 :mouse/get-hist get-mouse-hist}})
~~~

At the bottom, you see the `cmp-map`, which again is the map specifying the component that the switchboard will then instantiate. Inside, there's the `:state-fn`, which does nothing but create the initial state inside an atom. Then, there's the `:handler-map`, which here handles the two message types `:cmd/mouse-pos` and `:mouse/get-hist`.

The `process-mouse-pos` handler function then gets the `current-state`, the `msg-payload`, and the `msg-meta` inside the map it gets passed as a single argument, and returns both the `:new-state` and a message to emit, which is the same message it received, only now enriched by the `:count` from this component's state. Note that we are reusing the `msg-meta` from the original message, as this metadata also contains the `:sente-uid` of the client, which is required to route the message back to where it originated. There's more information on the metadata; we'll get to that later. Also, this function maintains the last 1001 positions from all connected client by taking the last 1000 and conjoining the received position.

The `get-mouse-hist` handler function returns the history of mouse moves that's maintained in the `:server/ptr-cmp` back to the client. Once again, the `:sente-uid` on the metadata contains the requester's ID, so we pass on the `msg-meta` in the response.

Next, the messages need to get from the UI component to the server, and back to the client. Here's how that looks like:

[message flow drawing]


## example.core on client side

For establishing these connections, let's have a look at the `core` namespaces on both server and client, starting with the **[client](https://github.com/matthiasn/systems-toolbox/blob/master/examples/trailing-mouse-pointer/src/cljs/example/core.cljs)**:

~~~
(ns example.core
  (:require [example.spec]
            [example.store :as store]
            [example.ui-histograms :as hist]
            [example.ui-mouse-moves :as mouse]
            [example.ui-info :as info]
            [example.metrics :as metrics]
            [example.observer :as observer]
            [matthiasn.systems-toolbox.switchboard :as sb]
            [matthiasn.systems-toolbox-sente.client :as sente]))

(enable-console-print!)

(defonce switchboard (sb/component :client/switchboard))

(defn init! []
  (sb/send-mult-cmd
    switchboard
    [[:cmd/init-comp
      #{(sente/cmp-map :client/ws-cmp {:relay-types #{:mouse/pos :mouse/get-hist}
                                       :msgs-on-firehose true})
        (mouse/cmp-map :client/mouse-cmp)
        (info/cmp-map  :client/info-cmp)
        (store/cmp-map :client/store-cmp)
        (hist/cmp-map  :client/histogram-cmp)}]
     [:cmd/route {:from :client/mouse-cmp :to #{:client/store-cmp :client/ws-cmp}}]
     [:cmd/route {:from :client/ws-cmp :to :client/store-cmp}]
     [:cmd/route {:from :client/info-cmp :to #{:client/store-cmp :client/ws-cmp}}]
     [:cmd/observe-state {:from :client/store-cmp
                          :to #{:client/mouse-cmp :client/histogram-cmp :client/info-cmp}}]]))

(init!)

(metrics/init! switchboard)
(observer/init! switchboard)
~~~


First, as usual, we create a switchboard. Then, we send messages to the switchboard, with the blueprints for the components we want the switchboard to initialize. For the core functionality discussed so far, only three of them are important: `:client/ws-cmp`, `:client/mouse-cmp`, and `:client/store-cmp`. We'll look at the other components later.

Note that the switchboard is kept in a `defonce`, which means that it can't be redefined later on. This is necessary for working with **[Figwheel](https://github.com/bhauman/lein-figwheel)**, as it allows the switchboard to shut down existing components and fire them up again after reload, while retaining the previous component state. Otherwise, without the `defonce`, the old state of each component would be lost as there would be an entirely new switchboard.

Then, inside the component init block, the `:client/ws-cmp` is fired up first. This is the WebSockets component provided by the **[systems-toolbox-sente](https://github.com/matthiasn/systems-toolbox-sente)** library. Here, we specify that only messages of the types `:mouse/pos` and `:mouse/get-hist` should be relayed to the server.

Next, we wire the components together:

* messages from `:client/mouse-cmp` are sent to both `:client/store-cmp` and `:client/ws-cmp`
* messages from `:client/ws-cmp` are sent to both `:client/store-cmp` and `:client/jvmstats-cmp`
* messages from  `:client/info-cmp` are sent to both `:client/store-cmp` and `:client/ws-cmp`
* `:client/mouse-cmp`, `:client/histogram-cmp` and `:client/info-cmp` all observe the state of the `:client/store-cmp`
* finally, the `:client/observer-cmp` is attached to the firehose, but more about that later when we look at `:client/observer-cmp`.

At the bottom of the namespace, we also fire up the observer and metrics components. We'll look at that when covering the respective components. 


## example.core on server side

With the client-side wiring in place, let's look at the server-side wiring in **[core.clj](https://github.com/matthiasn/systems-toolbox/blob/master/examples/trailing-mouse-pointer/src/clj/example/core.clj)**:

~~~
(ns example.core
  (:require [example.spec]
            [matthiasn.systems-toolbox.switchboard :as sb]
            [matthiasn.systems-toolbox-sente.server :as sente]
            [example.metrics :as metrics]
            [example.index :as index]
            [clojure.tools.logging :as log]
            [clj-pid.core :as pid]
            [example.pointer :as ptr]))

(defonce switchboard (sb/component :server/switchboard))

(defn start!
  "Starts or restarts system by asking switchboard to fire up the provided ws-cmp and the ptr
  component, which handles and counts messages about mouse moves."
  []
  (sb/send-mult-cmd
    switchboard
    [[:cmd/init-comp (sente/cmp-map :server/ws-cmp index/sente-map)]
     [:cmd/init-comp (ptr/cmp-map   :server/ptr-cmp)]
     [:cmd/route {:from :server/ptr-cmp :to :server/ws-cmp}]
     [:cmd/route {:from :server/ws-cmp  :to :server/ptr-cmp}]]))

(defn -main
  "Starts the application from command line, saves and logs process ID. The system that is fired up
  when restart! is called proceeds in core.async's thread pool. Since we don't want the application
  to exit when just because the current thread is out of work, we just put it to sleep."
  [& args]
  (pid/save "example.pid")
  (pid/delete-on-shutdown! "example.pid")
  (log/info "Application started, PID" (pid/current))
  (start!)
  (metrics/start! switchboard)
  (Thread/sleep Long/MAX_VALUE))
~~~

Here, just like on the client side, a switchboard is kept in a `defonce`. Then, we ask the switchboard to instantiate two components for us, the `:server/ws-cmp` and the `:server/ptr-cmp`, and then wire a simple message flow together.

We've already discussed the `:server/ptr-cmp` above. The `:server/ws-cmp` is the server side of the Sente-WebSockets component, and it takes a configuration map, which you can find in the **[example.index](https://github.com/matthiasn/systems-toolbox/blob/master/examples/trailing-mouse-pointer/src/clj/example/index.clj)** namespace:

~~~
(def sente-map
  "Configuration map for sente-cmp."
  {:index-page-fn index-page
   :relay-types   #{:mouse/pos :stats/jvm :mouse/hist}})
~~~

In this configuration map, we tell the component to relay three message types, `:mouse/pos`, `:stats/jvm`, and `:mouse/hist`. Also, we provide a function that renders the static HTML that is served to the clients. Have a look at the namespace to learn more. In particular, watch out for elements with an ID, such as `[:div#mouse]`, `[:figure#histograms.fullwidth]`, `[:div#info]`, or `[:div#observer]`. The client-side application will render dynamic content into these DOM elements.

Then, also in the server-side `example.core` namespace, there is the `-main` function, which is the entry point into the application. Here, we save a PID file, which will contain the process ID, also log the PID, and `start!` the application. We also start the server-side portion of the metrics gathering and display, but more about that later.

Finally, we let the main thread sleep until roughly the end of time, or until the application gets killed, whatever happens first. Well, `Long/MAX_VALUE` in milliseconds is only until roughly 292 million years from now, but hey, that should be enough.

Okay, now we have the message flow from capturing the mouse events to the server and back. Next, let's look at what happens to those events when they are back at the client.


## :client/store-cmp

Processing the returned data happens in the **[example.store namespace](https://github.com/matthiasn/systems-toolbox/blob/master/examples/trailing-mouse-pointer/src/cljs/example/store.cljs)**:

~~~
(ns example.store)

(defn mouse-pos-handler
  "Handler function for mouse position messages. When message from server:
    - determine the round trip time (RTT) by subtracting the message creation timestamp
      from the timestamp when the message is finally received by the store component.
    - determine server side processing time is determined. For this, we can use the timestamps
      from when the ws-cmp on the server side emits a message coming from the client and when the
      processed message is received back for delivery to the client.
    - update component state with the new mouse location under :from-server.
   When message received locally, only update position in :local."
  [{:keys [current-state msg-payload msg-meta]}]
  (let [new-state
        (if (:count msg-payload)
          (let [mouse-out-ts (:out-ts (:client/mouse-cmp msg-meta))
                store-in-ts (:in-ts (:client/store-cmp msg-meta))
                rt-time (- store-in-ts mouse-out-ts)
                srv-ws-meta (:server/ws-cmp msg-meta)
                srv-proc-time (- (:in-ts srv-ws-meta) (:out-ts srv-ws-meta))]
            (-> current-state
                (assoc-in [:from-server] (assoc msg-payload :rt-time rt-time))
                (update-in [:count] inc)
                (update-in [:rtt-times] conj rt-time)
                (update-in [:server-proc-times] conj srv-proc-time)
                (update-in [:network-times] conj (- rt-time srv-proc-time))))
          (-> current-state
              (assoc-in [:local] msg-payload)
              (update-in [:local-hist] conj msg-payload)))]
    {:new-state new-state}))

(defn show-all-handler
  "Toggles boolean value in component state for provided key."
  [{:keys [current-state msg-payload]}]
  {:new-state (update-in current-state [:show-all msg-payload] not)})

(defn mouse-hist-handler
  "Saves the received vector with mouse positions in component state."
  [{:keys [current-state msg-payload]}]
  {:new-state (assoc-in current-state [:server-hist] msg-payload)})

(defn state-fn
  "Return clean initial component state atom."
  [_put-fn]
  {:state (atom {:count             0
                 :rtt-times         []
                 :network-times     []
                 :server-proc-times []
                 :local             {:x 0 :y 0}
                 :show-all          {:local  false
                                     :remote false}})})

(defn cmp-map
  "Configuration map that specifies how to instantiate component."
  [cmp-id]
  {:cmp-id      cmp-id
   :state-fn    state-fn
   :handler-map {:mouse/pos    mouse-pos-handler
                 :cmd/show-all show-all-handler
                 :mouse/hist   mouse-hist-handler}
   :opts        {:msgs-on-firehose      true
                 :snapshots-on-firehose true}})
~~~

The `cmp-map` function once again generates the blueprint for how to instantiate this component. We specify that the initial component state is generated by calling the `state-fn`, which is a map with some keys as you can see above. Then, there are handler functions for three message types `:mouse/pos`, `:cmd/show-all`, and `:mouse/hist`, which we'll look at in detail. Finally, there is some configuration in `:opts`, which specifies that both messages and state snapshots should go on the firehose. We'll discuss the firehose when looking into the `:client/observer` component.

The most important handler function in this application is the `mouse-pos-handler` function. This function receives all `:mouse/pos` messages, which in this application can come either directly from the `:client/mouse-cmp` or from the `:server/ptr-cmp`. Where an individual message comes from is determined by the predicate `(:count msg-payload)` in the if statement. If that key exists, the message comes from the server, otherwise it's directly from `:client/mouse-cmp`.

In case the message is local, we do return new-state altered like this:

~~~
(-> current-state
    (assoc-in [:local] msg-payload)
    (update-in [:local-hist] conj msg-payload))
~~~

First, we set the `:local` key to contain the latest mouse position; then we add it to the local history.

The branch when the message comes from the server is slightly more involved:

~~~
(let [mouse-out-ts (:out-ts (:client/mouse-cmp msg-meta))
      store-in-ts (:in-ts (:client/store-cmp msg-meta))
      rt-time (- store-in-ts mouse-out-ts)
      srv-ws-meta (:server/ws-cmp msg-meta)
      srv-proc-time (- (:in-ts srv-ws-meta) (:out-ts srv-ws-meta))]
  (-> current-state
      (assoc-in [:from-server] (assoc msg-payload :rt-time rt-time))
      (update-in [:count] inc)
      (update-in [:rtt-times] conj rt-time)
      (update-in [:server-proc-times] conj srv-proc-time)
      (update-in [:network-times] conj (- rt-time srv-proc-time))))
~~~

Here, we calculate a few durations, the `rt-time`, which is the entire roundtrip time, and the `srv-proc-time`, which the duration between the `:server/ws-cmp` passing the message from the client on, and the same component encountering the response. For fully understanding this, you need to know that the **systems-toolbox** automatically timestamps messages when they are received or sent by any component, and saves that on the message metadata. 

Here's how the metadata looks like when the `:client/store-cmp` receives a `:mouse/pos` message from the server:

~~~
{:server/ws-cmp    {:out-ts 1467046063466
                    :in-ts  1467046063467}
 :sente-uid        "25450474-0887-4612-b5ad-07d1ca1f4885"
 :server/ptr-cmp   {:in-ts  1467046063467
                    :out-ts 1467046063467}
 :cmp-seq          [:client/mouse-cmp
                    :client/ws-cmp
                    :server/ptr-cmp
                    :server/ws-cmp
                    :client/store-cmp]
 :client/mouse-cmp {:out-ts 1467046063454}
 :client/store-cmp {:in-ts 1467046063506}
 :client/ws-cmp    {:in-ts  1467046063465
                    :out-ts 1467046063488}
 :tag              "61f2f357-3d12-40ff-9827-8a481cf36f75"
 :corr-id          "a31f12e7-33fb-48a8-833b-3d764c2c14bc"}
~~~

In contrast, this is how it looks like when the message comes directly from `:client/mouse-cmp`:

~~~
{:cmp-seq          [:client/mouse-cmp :client/store-cmp]
 :client/mouse-cmp {:out-ts 1467046063476}
 :corr-id          "2d32de55-cf1e-4646-8709-0c02c66d260f"
 :tag              "a7ebdac0-ce78-4e47-adbc-0b955efef5b4"
 :client/store-cmp {:in-ts 1467046063478}}
~~~

Of course, we could have also looked for the existence of the `:server/ptr-cmp` key on the metadata, rather than looking for the `:count` key on the payload in the branching logic when determining if a message comes from the server, it does not matter.

Okay, back to the `:client/store-cmp`. We do a little bit more there:

~~~
(update-in [:network-times] conj (- rt-time srv-proc-time)
~~~

Here, the RTT times are collected in a sequence so we can use the individual values as input to the histograms.

Next, there's the `show-all-handler` function to look at:

~~~
(defn show-all-handler
  "Toggles boolean value in component state for provided key."
  [{:keys [current-state msg-payload]}]
  {:new-state (update-in current-state [:show-all msg-payload] not)})
~~~

This handler toggles the value in the view configuration for showing either `:local` or the `:remote` history of mouse positions. These are then used as switches in the `:client/mouse-cmp`, as we've seen above. Finally, there's the `mouse-hist-handler` function:

~~~
(defn mouse-hist-handler
  "Saves the received vector with mouse positions in component state."
  [{:keys [current-state msg-payload]}]
  {:new-state (assoc-in current-state [:server-hist] msg-payload)})
~~~

This handler takes care of a sequence of mouse positions received from the server and stores them in the component state, which is returned under the `:new-state` key in the returned map. If these are shown is then dependent on the `:remote` key in the `:show-all` map inside the component state. Typically, when the `:mouse/hist` is received, this switch will be set to true, as the request for these values and switching this key on will have been sent by the `:client/info-cmp` at the same time. The beauty of the UI component watching the state of another component which holds the application state is that we don't have to do anything else. Once the data is back from the server, the mouse component will just know that it needs to re-render itself, now with the new data available. This was all to the `:client/store-cmp`, so let's look into the next component, where the histograms are rendered. But actually, now might be a good time to take a break and go for a walk.


## :client/histogram-cmp

Okay, ready? Let's move on. We've got some ground to cover. The `:client/histogram-cmp` in the **[example.ui-histograms namespace](https://github.com/matthiasn/systems-toolbox/blob/master/examples/trailing-mouse-pointer/src/cljs/example/ui_histograms.cljs)** makes use of the data we just collected:

~~~
(ns example.ui-histograms
  (:require [matthiasn.systems-toolbox-ui.reagent :as r]
            [matthiasn.systems-toolbox-ui.charts.histogram :as h]))

(defn histograms-view
  "Renders histograms with different data sets, labels and colors."
  [{:keys [observed]}]
  (let [state @observed
        rtt-times (:rtt-times state)
        server-proc-times (:server-proc-times state)
        network-times (:network-times state)]
    [:div
     [:div
      [h/histogram-view rtt-times "Roundtrip t/ms" "#D94B61"]
      [h/histogram-view
       (h/percentile-range rtt-times 99) "Roundtrip t/ms (within 99th percentile)" "#D94B61"]
      [h/histogram-view
       (h/percentile-range rtt-times 95) "Roundtrip t/ms (within 95th percentile)" "#D94B61"]]
     [:div
      [h/histogram-view network-times "Network time t/ms (within 99th percentile)" "#66A9A5"]
      [h/histogram-view (h/percentile-range network-times 95)
       "Network time t/ms (within 95th percentile)" "#66A9A5"]
      [h/histogram-view server-proc-times "Server processing time t/ms" "#F1684D"]]]))

(defn cmp-map
  [cmp-id]
  (r/cmp-map {:cmp-id  cmp-id
              :view-fn histograms-view
              :dom-id  "histograms"
              :cfg     {:throttle-ms           100
                        :msgs-on-firehose      true
                        :snapshots-on-firehose true}}))
~~~

The most exciting stuff here happens in the histogram namespace of the **systems-toolbox-ui** library, but we'll get there. There are some things of interest here anyway. Did you notice the `:throttle-ms` key in the `:cfg` of the `cmp-map`? This tells the systems-toolbox to deliver new state snapshots only every 100 milliseconds. This throttling is done because it is expensive enough to calculate the histograms for us not to want to do it on every frame. Ten times a second appears to be a good compromise between feeling alive and saving some CPU cycles.

The rest of this namespace is probably not terribly surprising by now. The `histograms-view` function, which is the `:view-fn` of this **systems-toolbox-ui** component, renders a `:div` with six different `histogram-view`s, which each renders into an SVG with the chart itself. In some cases, we do some data manipulation first, such as the `hist/percentile-range` from the library namespace. Notice that there are two `:div`s inside the parent, each with three elements inside? That's for the **[Flexible Box](https://www.w3.org/TR/2016/CR-css-flexbox-1-20160526/)** layout, also known as **flexbox**. The rest of the layout is then done in **[CSS](https://github.com/matthiasn/systems-toolbox/blob/master/examples/trailing-mouse-pointer/resources/public/css/example.css)**:

~~~
#histograms {
    margin-bottom: 1em;
}

#histograms div div{
    display: flex;
    flex-flow: row;
}
~~~

So what happens here is that we have two `flex` elements, each with `flex-flow: row;` so that each triplet will cover a row inside the available space.

Okay, that's it in this namespace.


## matthiasn.systems-toolbox-ui.charts.histogram

The most interesting stuff for rendering the histograms happens in the **[matthiasn.systems-toolbox-ui.charts.histogram](https://github.com/matthiasn/systems-toolbox-ui/blob/master/src/cljs/matthiasn/systems_toolbox_ui/charts/histogram.cljs)** namespace:

~~~
(ns matthiasn.systems-toolbox-ui.charts.histogram)

(def text-default {:stroke "none" :fill "black" :style {:font-size 12}})
(def text-bold (merge text-default {:style {:font-weight :bold :font-size 12}}))
(def x-axis-label (merge text-default {:text-anchor :middle}))
(def path-defaults {:fill :black :stroke :black :stroke-width 1})

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

(defn histogram-y-axis
  "Draws y-axis of a chart."
  [x y h mx]
  (let [increment (cond (> mx 6000) 1000
                        (> mx 3000) 500
                        (> mx 1000) 250
                        (> mx 250) 100
                        (> mx 100) 50
                        (> mx 50) 20
                        (> mx 25) 10 :else 5)
        rng (range 0 (inc (round-up mx increment)) increment)
        scale (/ h (dec (count rng)))]
    [:g
     [:path (merge path-defaults {:d (str "M" x " " y "l 0 " (* h -1) " z")})]
     (for [n rng]
       ^{:key (str "yt" n)}
       [:path (merge path-defaults
                     {:d (str "M" x " " (- y (* (/ n increment) scale)) "l -" 6 " 0")})])
     (for [n rng]
       ^{:key (str "yl" n)}
       [:text (merge text-default
                     {:x (- x 10) :y (- y (* (/ n increment) scale) -4) :text-anchor :end}) n])]))

(defn histogram-x-axis
  "Draws x-axis for histrogram."
  [x y mn mx w scale increment]
  (let [rng (range mn (inc mx) increment)]
    [:g
     [:path (merge path-defaults {:d (str "M" x " " y "l" w " 0 z")})]
     (for [n rng]
       ^{:key (str "xt" n)}
       [:path (merge path-defaults {:d (str "M" (+ x (* (- n mn) scale)) " " y "l 0 " 6)})])
     (for [n rng]
       ^{:key (str "xl" n)}
       [:text (merge x-axis-label {:x (+ x (* (- n mn) scale)) :y (+ y 20)}) n])]))

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

(defn histogram-view-fn
  "Renders a histogram for roundtrip times."
  [{:keys [rtt-times x y w h x-label color bin-cf max-bins increment-fn]}]
  (let [mx (apply max rtt-times)
        mn (apply min rtt-times)
        rng (- mx mn)
        increment-fn (or increment-fn default-increment-fn)
        increment (increment-fn rng)
        mx2 (round-up (or mx 100) increment)
        mn2 (round-down (or mn 0) increment)
        rng2 (- mx2 mn2)
        x-scale (/ w rng2)
        bin-size (max (/ rng max-bins) (* (freedman-diaconis-rule rtt-times) bin-cf))
        binned-freq (frequencies (map (fn [n] (Math/floor (/ (- n mn) bin-size))) rtt-times))
        binned-freq-mx (apply max (map (fn [[_ f]] f) binned-freq))
        bins (inc (apply max (map (fn [[v _]] v) binned-freq)))
        bar-width (/ (* rng x-scale) bins)
        y-scale (/ (- h 20) binned-freq-mx)]
    [:g
     (if (> bins 4)
       (for [[v f] binned-freq]
         ^{:key (str "bf" x "-" y "-" v "-" f)}
         [:rect {:x      (+ x (* (- mn mn2) x-scale) (* v bar-width))
                 :y      (- y (* f y-scale))
                 :fill   color :stroke "black"
                 :width  bar-width
                 :height (* f y-scale)}])
       [:text {:x           (+ x (/ w 2))
               :y           (- y 50)
               :stroke      "none"
               :fill        "#DDD"
               :text-anchor :middle
               :style       {:font-weight :bold :font-size 24}}
        "insufficient data"])
     (histogram-x-axis x (+ y 7) mn2 mx2 w x-scale increment)
     [:text (merge x-axis-label text-bold {:x           (+ x (/ w 2))
                                           :y           (+ y 48)
                                           :text-anchor :middle})
      x-label]
     [:text (let [x-coord (- x 45)
                  y-coord (- y (/ h 3))
                  rotate (str "rotate(270 " x-coord " " y-coord ")")]
              (merge x-axis-label text-bold {:x         x-coord
                                             :y         y-coord
                                             :transform rotate}))
      "Frequencies"]
     (histogram-y-axis (- x 7) y h (or binned-freq-mx 10))]))

(defn histogram-view
  "Renders an individual histogram for the given data, dimension, label and color,
  with a reasonable size inside a viewBox, which will then scale smoothly into any
  div you put it in."
  [data label color]
  [:svg {:width   "100%"
         :viewBox "0 0 400 250"}
   (histogram-view-fn {:rtt-times data
                       :x         80
                       :y         180
                       :w         300
                       :h         160
                       :x-label   label
                       :color     color
                       :bin-cf    0.8
                       :max-bins  25})])
~~~



Okay, that was a bit involved. But hey, in order to use a histogram in your project, all you need is a import this namespace, and then use a one-liner to plot your histogram (and more chart types to come).

