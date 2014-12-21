# Client-side Architecture

Currently, the client-side is not nearly as well organized as the server-side code. However, that should change until all the parts of the client side are described.

Generally, as a constraint, there should be as few external JavaScript dependencies as possible. Since earlier versions, the following libraries have already been removed and replaced by ClojureScript code:

* Rickshaw - time series chart library, replaced by directly manipulating SVG from ClojureScript
* Regression.js - library for regression analysis, replaced by translating a Common Lisp library to Clojure.
* Wordcount barchart - previous implementation was realized with React.js in JavaScript, now implemented in ClojureScript

The following libraries are still required, should also be replaced though:

* Moment.js - required for Date formatting and parsing
* D3.js - required for Wordcloud
* d3.layout.cloud.js 
* wordcloud.js - own JavaScript implementation for interacting with d3.layout.cloud.js

{pagebreak}
