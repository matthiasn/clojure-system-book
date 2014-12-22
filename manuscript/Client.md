# Client-side Architecture

Currently, the client-side is not nearly as well organized as the server-side code. However, that should change until all the parts of the client side are described.

Generally, as a constraint, there should be as few external JavaScript dependencies as possible. Since earlier versions, the following libraries have already been removed and replaced by ClojureScript code:

* **[Rickshaw](http://code.shutterstock.com/rickshaw/)** - time series chart library, replaced by directly manipulating SVG from ClojureScript
* **[Regression.js](https://github.com/Tom-Alexander/regression-js)** - library for regression analysis, replaced by translating a Common Lisp library to Clojure.
* Wordcount barchart - previous implementation was realized with React.js in JavaScript, now implemented in ClojureScript

The following libraries are still required, should also be replaced though where possible, depending on the time available:

* **[Moment.js](http://momentjs.com)** - required for Date formatting and parsing
* **[D3.js](http://d3js.org)** - required for Wordcloud
* **[d3.layout.cloud.js](https://github.com/jasondavies/d3-cloud)**
* **[wordcloud.js](https://github.com/matthiasn/BirdWatch/blob/43a9c09493257b9c9b5e9e5644df5f67085feb84/Clojure-Websockets/MainApp/resources/public/js/wordcloud.js)** - own JavaScript implementation for interacting with d3.layout.cloud.js

{pagebreak}
