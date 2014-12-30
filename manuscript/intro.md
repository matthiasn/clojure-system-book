# Introduction

Clojure is a beautiful language and I have been fascinated from the first moment I laid eyes on it in 2013. However, what remained a mystery to me for most of the time was how to build more complex systems. I started researching the options that would allow me to structure an arbitrarily complex application in a way that is **easy to understand and maintain**. Here is what I found.

In this book, we will build a system in Clojure that processes streaming data and visualizes different insights into the data in a browser. All visualizations of data display information in (near) real-time.

The sample implementation that we will use is the **[BirdWatch](https://github.com/matthiasn/BirdWatch)** application, a project I have started in Scala with AngularJS on the client side but that I have then switched over to an all-Clojure/ClojureScript implementation. This application subscribes to the **[Twitter Streaming API](https://dev.twitter.com/docs/streaming-apis)** for all tweets that contain one or more terms out of a set of terms and makes the tweets searchable through storing them in ElasticSearch. All search results are updated in the browser when new matching tweets come in through the streaming data source. Here's how that looks like:

![Screenshot](images/screenshot.png)

A live version of the Clojure version of this application is available **[here](http://birdwatch2.matthiasnehlsen.com/#*)**.

Over the course of this book, we will follow the data and explore parts of the system in the same order that the streaming data flows through the system.

There has been a **[series of articles](http://matthiasnehlsen.com/blog/2014/09/24/Building-Systems-in-Clojure-1/)** that attempted to explain different aspects of the system, but I think that a book is a much better artifact to work on when trying to explain this application than a series of blog articles.

Please join me on this journey as I explore the problem further and at the same time attempt to write a book about it.

During the first couple of days, I made this content freely available. If you received this book for free and you realize that you like the content, please consider paying the suggested price. I guarantee you that any additional revenue will allow me to finish this book earlier.

I am looking forward to a fun project and a lively discussion about all aspects of building a system in Clojure.

Cheers,
Matthias