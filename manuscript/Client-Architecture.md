# Client-side Architecture

Currently, the client side is not nearly as well organized as the server-side code. However, that should change by the time all the parts of the client side have been described.

Generally, as a constraint, there should be as few external JavaScript dependencies as possible. The following libraries have already been removed and replaced by ClojureScript code:

* **[Rickshaw](http://code.shutterstock.com/rickshaw/)** - time series chart library, replaced by directly manipulating SVG from ClojureScript
* **[Regression.js](https://github.com/Tom-Alexander/regression-js)** - library for regression analysis, replaced by translating a Common Lisp library to Clojure.
* Wordcount bar chart - previous implementation was realized with React.js in JavaScript, now implemented in ClojureScript

The following libraries are still required, however should also be replaced where possible, depending on the time available:

* **[Moment.js](http://momentjs.com)** - required for date formatting and parsing
* **[D3.js](http://d3js.org)** - required for the word cloud
* **[d3.layout.cloud.js](https://github.com/jasondavies/d3-cloud)**
* **[wordcloud.js](https://github.com/matthiasn/BirdWatch/blob/43a9c09493257b9c9b5e9e5644df5f67085feb84/Clojure-Websockets/MainApp/resources/public/js/wordcloud.js)** - own JavaScript implementation for interacting with d3.layout.cloud.js

In the following chapter about the client side, I will walk you through the code as it currently exists. I am not completely happy with how the code is structured at the moment, particularly in comparison to the server side, but we can look at ways of refactoring the codebase later on. I'd love to get some feedback from you, the readers, and then work on the architecture some more with your feedback in mind.