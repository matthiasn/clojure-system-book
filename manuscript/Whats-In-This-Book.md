# What's in this book?

This book is a book about Clojure and it is also a book about systems. What is a system? Let's ask an expert in the discipline of **systems thinking**:

> A system isn't just any old collection of things. A system is an interconnected set of
elements that is coherently organized in a way that achieves something. If you look at that definition closely for a minute, you can see that a system must consist of three kinds of things: elements, interconnections, and a function or purpose. 
> _Donatella Meadows (2008)_[^Meadows]

In this book, we will discuss multiple systems that together form other systems or that somehow interact with each other. For example, the BirdWatch application that we will discuss is a system as a whole. This system is then made up of other (sub-)systems that communicate with each other. Twitter is another system, where the purpose of BirdWatch is to observe the system that is Twitter, and that manipulates observed events in some (potentially useful) way. In order to obtain a better knowledge of the BirdWatch system, we will have to observe it, and we will have to stress test it; otherwise, our knowledge of this system will remain shallow. For that, we will create yet another systems for monitoring and also for load testing, each again with observable properties.

The discussion of systems in the context of software is likely relevant no matter what language you are using. The actual code in this book will be in Clojure, but please do not be discouraged if you don't know Clojure just yet. Clojure is a beautifully simple language, and I have been fascinated from the first moment I laid eyes on it. However, what remained a mystery to me for most of the time was how to build more complex systems. I started researching the options that would allow me to structure an arbitrarily complex application in a way that is **easy to understand and maintain**. Here is what I found.

In this book, we will build a system in Clojure that processes streaming data and visualizes different insights into the data in a browser. All visualizations of data display information in (near) real-time.

The sample implementation that we will use is the **[BirdWatch](https://github.com/matthiasn/BirdWatch)** application -- a project I have started in Scala with AngularJS on the client side, but that I then switched over to an all-Clojure/ClojureScript implementation.

![Screenshot](images/screenshot.png)

This application subscribes to the **[Twitter Streaming API](https://dev.twitter.com/docs/streaming-apis)** for all tweets that contain one or more terms out of a set of terms and makes the tweets searchable through storing them in ElasticSearch. All search results are updated in the browser when new matching tweets come in through the streaming data source.

A live version of the Clojure version of this application is available **[here](http://birdwatch2.matthiasnehlsen.com/#*)**.

Over the course of this book, we will follow the data and explore parts of the system in the same order that the streaming data flows through the system.

There has been a **[series of articles](http://matthiasnehlsen.com/blog/2014/09/24/Building-Systems-in-Clojure-1/)** that attempted to explain different aspects of the system, but I think that a book is a much better artifact to work on when trying to explain this application than a series of blog articles.

Please join me on this journey as I explore the problem further and at the same time write a book about it. The book will begin with a description of the current architecture, both on the server and the client side. Later on, we will put the design to the test, come up with a load testing strategy and look at ways of improving performance. Most likely, we will also go through some refactoring of the current architecture and add some functionality.

I am looking forward to a fun project and a lively discussion about all aspects of building a system in Clojure.

Cheers,
Matthias

[^Meadows]: Meadows, Donatella H. (2008) _Thinking in Systems: A Primer_ - Page 11
