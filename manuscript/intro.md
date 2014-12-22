# Introduction

Clojure is a beautiful language and I have been fascinated from the first moment I laid eyes on it in 2013. However, what remained a mystery to me for most of the time was how to build more complex systems. I started researching the options that would allow me to structure an arbitrarily complex application in a way that is **easy to understand and maintain**. Here is what I found.

In this book, we will build a system in Clojure that processes streaming data and visualizes different insights into the data in a browser. All visualizations of data display information in (near) real-time.

The sample implementation that we will use is the **[BirdWatch](https://github.com/matthiasn/BirdWatch)** application, a project I have started in Scala with AngularJS on the client side but that I have then switched over to an all-Clojure/ClojureScript implementation. This application subscribes to the **[Twitter Streaming API](https://dev.twitter.com/docs/streaming-apis)** for all tweets that contain one or more terms out of a set of terms and makes the tweets searchable through storing them in ElasticSearch. All search results are updated in the browser when new matching tweets come in through the streaming data source.

A live version of the Clojure version of this application is available **[here](http://birdwatch2.matthiasnehlsen.com/#*)**.

Over the course of this book, we will follow the data and explore parts of the system in the same order that the streaming data flows through the system.

There has been a **[series of articles](http://matthiasnehlsen.com/blog/2014/09/24/Building-Systems-in-Clojure-1/)** that attempted to explain different aspects of the system, but I think that a book is a much better artifact to work on than a series of blog articles. Please note that part of the content in this book is still more or less directly taken from the previous blog posts.

Please join me on this journey as I explore the problem further and at the same time attempt to write a book about it. Iinitially, I will make this content freely available. Please check out if you find the content at all useful and discuss where you see better solutions. Only if you a) find the content helpful and b) you can afford it, I'd ask you to pay the recommended price (or whatever other amount you find appropriate). I am looking forward to a fun project and a lively discussion about all aspects of building a system in Clojure.

Cheers,
Matthias