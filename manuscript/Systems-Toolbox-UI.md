# The systems-toolbox-ui library


## Introduction

Pretty much any application out there has some a user facing aspect. Even those that don't have one would usually greatly benefit from having a user interface that tells you at a glance if your services are healthy. Also, I just generally find it more rewarding to work on something that has some presentation layer, no matter how sparse. So let's make it easy to add a presentation layer to an arbitrary piece of backend code, shall we?


## React

**[React](https://facebook.github.io/react/)** is a UI renderer by Facebook. It **only** does the rendering and is not a Framework for JavaScript applications. However, it does the rendering a) quite well and b) in a way that is particularly useful when coming from a functional programming approach.

Let me explain. What happens is that we define a function that turns some data into a DOM (sub)-tree. There is no notion of manipulating that tree, as you'd tediously have to do with the likes of jQuery. Rather, every time the data changes, the entire thing is rendered. So far, that sounds conceptually pure but at the same time very expensive, right? Well, it would be, only that this rendering does not happen in the actual, heavy-weight DOM but into a lightweight JavaScript data structure called virtual DOM. Then, there's a diffing process that compares the previous version of that virtual DOM (which corresponds to what is on the page) and the latest version, determines the changes, and only then puts those changes into effect on the page. This mechanism happens to be fast. 

Why should you care? In Clojure and ClojureScript, we strongly favor working with immutable data structures. Thus, it conceptually fits very well to think of DOM tree as something immutable, instead of something that you have to modify in place. If there are state changes, we just pass the new state to that function that creates our UI, and then let it figure out under the hood how to get there. For all we need to care about, this UI renderer recreates our UI on every single change. 


## Redux

**[Redux](https://github.com/reactjs/redux)** is one possible approach to creating a larger application that makes use of React as its renderer. Redux is not tied to React but in reality, those libraries are probably most often used together. The general idea here is that there is some protected state container. Mutations of that managed application state then cannot happen from random places in your program. Instead, you can only dispatch actions to that managed state container, which will act on that action (or not, if no such action is defined). This model defines clear borders of where state mutation can happen, and thus makes the application easier to maintain.

I have not worked with Redux extensively, but the approach is the same with the **[systems-toolbox-ui](https://github.com/matthiasn/systems-toolbox-ui)** library, and it works well there. Let's have a look at a simple example next.


## Counter example

When you go through the examples for Redux, you'll see one with some counters. Let's do the same thing with **systems-toolbox-ui** library. You can also find the code this example **[here](https://github.com/matthiasn/systems-toolbox-ui/tree/master/examples/redux-counter01)**. This is how it is going to look like:

![](images/counter.png)

As you can see in the screenshot above, we have a very simple UI that first of all shows us the application state, and then initially three counters, each of which with a button to increment and decrement the counter, plus buttons to add and remove another counter at the end.

Now remember from Redux that we can only interact with the state using actions. Let's have a look at the namespace implementing the store first.

~~~
(ns example.store)

(defn inc-handler
  "Handler for incrementing specific counter"
  [{:keys [current-state msg-payload]}]
    {:new-state (update-in current-state [:counters msg-payload] inc)})

(defn dec-handler
  "Handler for decrementing specific counter"
  [{:keys [current-state msg-payload]}]
    {:new-state (update-in current-state [:counters msg-payload] dec)})

(defn remove-handler
  "Handler for removing last counter"
  [{:keys [current-state]}]
  {:new-state (update-in current-state [:counters] #(into [] (butlast %)))})

(defn add-handler
  "Handler for adding counter at the end"
  [{:keys [current-state]}]
    {:new-state (update-in current-state [:counters] conj 0)})

(defn state-fn
  "Returns clean initial component state atom"
  [_put-fn]
  {:state (atom {:counters [2 0 1]})})

(defn cmp-map
  [cmp-id]
  {:cmp-id      cmp-id
   :state-fn    state-fn
   :handler-map {:cnt/inc inc-handler
                 :cnt/dec dec-handler
                 :cnt/remove remove-handler
                 :cnt/add add-handler}})
~~~
 
Okay, let's go through this namespace from the bottom up. The `cmp-map` generates a map which the `systems-toolbox` library needs to create/instantiate a component. The `cmp-id` is straight-forward, this is required when later wiring the component with other components. Then, there is the `:state-fn` key. Here, a function is expected that takes the `put-fn`, which is the function that allows a component to emit messages at any time, irrespective of incoming messages, and that returns a map with the initial component state as an atom.

Then, there's the handler map. It's a map with the different message types that the component handles as keys, and the respective handler functions as values.

Then, there are the handler functions. Handler functions take a single argument, a map, which among other things contains the `current-state` and `msg-payload`. We'll look at messages later. For now, it's only important that the message payloads here need to be integers that correspond to existing counter entries. Then, the handler functions can return the new component state after processing the message in the `new-state` key on the return map. Handlers can also emit messages themselves, but we'll look at that later.







