# Client-side Architecture
On the client side, we have a single-page web application that responds to new tweets matching a current real-time query with an immediately updated user interface, including statistics. It is a ClojureScript application in which different parts of the system are separated by interacting exclusively via immutable messages put on channels. All state is local to each component, protecting the state from accidentally mutilating it from outside the "component" that is the owner of the state. The UI is rendered using **[Reagent](http://reagent-project.github.io)** on top of **[React.js](http://facebook.github.io/react/)**.

## React.js
**[React.js](http://facebook.github.io/react/)** is a JavaScript UI rendering library from Facebook that by design works well with immutable data structures because it doesn't depend on watching for changes inside mutable objects. Instead, the entire state is rendered into a virtual DOM every time new state is passed to it. While this may seem like a waste of CPU resources at first, it is actually not because we are not talking about the actual and slow DOM. Instead, React compares the newly rendered virtual DOM representation with the previous one and only puts the actual changes it detects into effect. Now this diffing algorithm is very fast, resulting in React.js typically beating the more full-fledged UI frameworks in terms of performance.

This mechanism with passing new state [^props] now works particularly well with immutable data coming from ClojureScript. After running away from **AngularJS** soon after trying it out with Clojurescript, this came as more than welcome and pleasant surprise back in the last quarter of 2013. David Nolen deserves all the praise for discovering this sweet match between React.js and ClojureScript. [^nolen-mvc]

[^props]: State passed from the outside and conceptually treated is immutable data is called called **Props** in React's terminology. 

[^nolen-mvc]: David's article **[The Future of JavaScript MVC Frameworks](http://swannodette.github.io/2013/12/17/the-future-of-javascript-mvcs/)** hit me while I was working on a book about AngularJS. I realized, oh darn, he's right, what I'm doing here doesn't make the amount of sense that I would want it to make, so I quit working on the book. While slightly discomforting back then, I am glad I followed my intuition. Immutable data in the browser is pure bliss.

## Reagent
**[Reagent](http://reagent-project.github.io)** provides a thin layer on top React.js in which we can use **Hiccup** syntax to write user interfaces. I find this to be wonderfully terse.

However, when you study it's documentation, you will notice that UI components directly interact with application state held in atoms. While that may be fine for small applications, I do not like this approach when the application becomes more complex and we want to handle state alterations in a specialized part of the application. In that case, when I wear the hat of the developer who works on the "business logic" of the application, I would not want the UI developer to be able to change something so that all of a sudden my artifact appears to be broken, even if some other part of the application is to blame. When you think about an atom, guess what happens when you get a hold of it and run something like this:

~~~
(reset! app-atom {})
~~~

Oops. Now this would be easy to detect, but the effect could be much more subtle by only altering or removing a key somewhere inside the application state. Long story short, as the business logic guy, I would sleep better. This does not change at all when the UI guy happens to be me. When my mind works in the context of UI, it shouldn't also have to be carefuly about not accidentally tripping on some wire in the business logic. It's a major selling point for immutable data is that it's safe to pass around.

## Ownership of application state
Ownership of the application state did not seem like a big deal at first when I started working on the application. But when I realized that encapsulation also entails limiting write access to application state, I rewrote the client-side code base. Now, there's a state namespace that is responsible for modifying and guarding application state. There are actually a few namespaces around the state namespace but they only contain pure functions. All application state is encapsulated in an atom inside a let binding in the function that initializes the application state.

All changes to the application state are then broadcast using a core.async Pub/Sub, but only as the dereferenced application state as it was at the time of dereferencing it. This broadcast application state is immutable so no subscriber can accidentally mutate or mutilate it in any way. You can think of it like a TV station, broadcasting moving images over the air. An arbitrary number of viewers can then tune in and watch. What you'll watch is usually immutable, except for some kind of backchannel like the telephone, email or an actual letter. But then it is completely up to the TV station how to deal with these messages. No ordinary viewer can just mess the experience up for all the other viewer [^tv]. 

I want the same kind of conceptual protection for my application's state. This is handled nicely by encapsulating it inside a function and only interact via channels, where the control on how to deal with incoming messages lies entirely with the state's owner. Here's a simple drawing for that:




Image sources:

* Receiving antenna (**[wikimedia](http://commons.wikimedia.org/wiki/File:Gizmotchy.svg)**)




[^tv]: Unless, of course, you're the owner of the channel and make stupid decisions on which TV shows and films to buy.

## Constraints
Generally, as a constraint, there should be as few external JavaScript dependencies as possible. The following libraries have recently been removed and replaced by ClojureScript code:

* **[Rickshaw](http://code.shutterstock.com/rickshaw/)** - timeseries chart library, replaced by directly manipulating SVG from ClojureScript
* **[Regression.js](https://github.com/Tom-Alexander/regression-js)** - library for regression analysis, replaced by translating a Common Lisp library to Clojure.
* Wordcount bar chart - previous implementation was realized with React.js in JavaScript, now implemented in ClojureScript

The following libraries are still required, however should also be replaced where possible, depending on the time available:

* **[Moment.js](http://momentjs.com)** - required for date formatting and parsing
* **[D3.js](http://d3js.org)** - required for the word cloud
* **[d3.layout.cloud.js](https://github.com/jasondavies/d3-cloud)**
* **[wordcloud.js](https://github.com/matthiasn/BirdWatch/blob/43a9c09493257b9c9b5e9e5644df5f67085feb84/Clojure-Websockets/MainApp/resources/public/js/wordcloud.js)** - own JavaScript implementation for interacting with d3.layout.cloud.js

## Code Walkthrough
In the following chapter about the client side, I will walk you through the client-side code base. While the restructuring I mentioned above is complete in **[master](https://github.com/matthiasn/BirdWatch)**, the following chapters still reflect the previous version. I am actively working on the rewrite.
