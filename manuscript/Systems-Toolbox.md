# Update - April 9th, 2015

**TL;DR: Where are the book updates?** I've been developing a library for building systems, and it has already made the codebase of the BirdWatch application much simpler. I'll be back to updating the book using the new code and concepts soon. Thanks for buying the book!

---

Hi there, and thank you for becoming a reader of **[Building a System in Clojure (and ClojureScript)](https://leanpub.com/building-a-system-in-clojure)** early on[^1]. You will probably have wondered by now when I was planning to release new content. That's a valid question -- and you deserve to know.

Writing a book while people are reading it is an interesting endeavor as that substantially raises the hurdle for changing something. I never thought about that before, yet it is very real. Without readers, you can just shuffle stuff around, throw away or completely rewrite parts without having to answer to anyone. With readers, the author should at least have a good reason to make radical changes. I'm not saying that's a bad thing, it's just different.

Anyhow, I believe I have good reasons to make substantial changes to what exists already. In the chapters you have probably read so far, I describe building a system, both on client and server, for consuming and processing some real-time information source. This source happens to be one of the **[Twitter Streaming APIs](https://dev.twitter.com/streaming/overview)**. Personally, I don't think it's the most interesting data source of such kind out there, but it happens to be interesting enough while also being easily accessible -- and constantly running.

While editing the existing content of the **[book](https://leanpub.com/building-a-system-in-clojure)**, I increasingly noticed that I was portraying building a house on a fairly low level of abstraction. It was like I'm describing the plumbing in a house by providing a detailed account of every body movement performed by the plumber. If I were the recipient of such account, on the second or third time of explaining how to hold the pressing tool in order to connect two copper pipes with a valve in between, I'd probably yawn. And mention that I understood the process the first time -- please just continue with locations and purposes of the valves as I have no problem imagining how the valves got there in the first place.

The code inside the BirdWatch application felt the same way. Where I only wanted to focus on what makes individual components different, I was forced to write even the stuff that's the same over and again. That's boilerplate.

So instead, I wanted to pull common functionality out into a separate library. Then, a new consulting gig came along with **[Aviso](www.aviso.io)** thanks to the recommendation of **[Ryan Neufeld](http://homegrown.io)**. That could have been a problem in terms of finding the time for working on the library. But luckily, the opposite was true. The library could well be helpful in the project I'm working on, so they generously let me explore the problem and write the library.

I call this library  **[systems-toolbox](https://github.com/matthiasn/systems-toolbox)**. If you happen to follow me on GitHub, you may have seen the **[79 commits](https://github.com/matthiasn/systems-toolbox/commits/master)** I pushed there over the last five weeks, and now I can say that it is shaping up nicely. The documentation is not up to speed yet, but the codebase of **[BirdWatch](https://github.com/matthiasn/BirdWatch)** provides a good point of reference. I've already fully migrated BirdWatch to use the new approach, both on the client- and the server-side, and it's working well.

I've been very interested in Systems Thinking lately. If you're not familiar with this kind of thinking, it is based on approaching and understanding the world by determining observable systems.

One such system is your house and how it's dealing with changing weather conditions outside the thermal envelope. The system is trying to stabilize the inside conditions like temperature and humidity all while either energy flows into the building on a hot summer day or out on a freezing winter night. Through observing the system, we can start interacting with it through thermostat settings or, in modern times, through some smart home control center.

Now my realization was that a piece of software is not all that different. We have some input, such as user requests, a desired output, and a set of constraints. Now by observing the system, we can start utilizing resources better and get closer to the desired behavior. So far, so old. But what if the observability was built right into the system, instead of being tugged on with some monitoring tool? When I have components emit some logging information, and I do roughly the same in every component, then that's boilerplate, too. Instead, the library should provide the same kind of logging for every component, such as the input and output messages, processing times, wait times, plus whatever we might want to log about a particular component's behavior.

In the library I mentioned I above, so far I have mostly focused on the mechanics of the plumbing between components and how to embed functionality into these components, and that part is working well. In the **BirdWatch** application, I have thrown out the **[Component library](https://github.com/stuartsierra/component)** in **[master on GitHub](https://github.com/matthiasn/BirdWatch)**, and I find using the **[systems-toolbox](https://github.com/matthiasn/systems-toolbox)** much more straight-forward.

But please don't take my word for it and have a look for yourself. The best place to start is the **BirdWatch** code, specifically in these namespaces:

**[Server-side SwitchBoard in Clojure ](https://github.com/matthiasn/BirdWatch/blob/master/Clojure-Websockets/MainApp/src/clj/birdwatch/main.clj)**

**[Client-side SwitchBoard in ClojureScript ](https://github.com/matthiasn/BirdWatch/blob/master/Clojure-Websockets/MainApp/src/cljs/birdwatch/core.cljs)**

The new library is used throughout, both on the server and on the **[ClojureScript](https://github.com/clojure/clojurescript)** client. Observability and metrics are not as developed as I want them to be, but that's an area of focus now and will be addressed soon.

Okay, this is all for today. The **[systems-toolbox](https://github.com/matthiasn/systems-toolbox)** library is stabilizing, and I will soon get around to writing about it in the book. For now, your best bet is to explore the code mentioned above. I've tried to make message flow as obvious as possible there. Please don't forget to let me know what you think. _Right now_ is probably the best time for feedback.

Cheers,
Matthias

[^1]: If you are not a reader of the book yet, no worries, you can get the book while supply lasts on **[Leanpub](https://leanpub.com/building-a-system-in-clojure)**.
