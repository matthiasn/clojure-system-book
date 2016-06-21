# Introduction

Hamburg, June 2016

A year and a half ago I started working on this book. The initial chapters just poured out of me, and then I realized that the kind of application that I was trying to build would greatly benefit from a library that makes composing a system out of individual components or subsystems that communicate via message passing much simpler. Then I had this gig that allowed me to explore the problem space further, and build a commercial application on top of it.

For a long time, I had wanted to get back to the book. But two things held me back:

1) I was waiting for inspiration to strike me once again, and then a few weeks later regain full conscience, noticing to my pleasant surprise that the book had completed itself in the meantime. Imagine my disappointment when I noticed that it doesn't work that way.

2) I was dreading having to write the documentation of the library without a mechanism in place that validates its usage, the messages passed around, and alterations to the managed component state. I had looked into **[schema](https://github.com/plumatic/schema)** a few times, but somehow did not feel compelled to adopt it. No validation wasn't viable either, as then everything depends on the wording of the documentation, and that's a source of ambiguity that's best avoided.

Then, along came **[clojure.spec](http://clojure.org/about/spec)**, and I was stunned how well it fit the bill. Within days, I had adapted the **[systems-](https://github.com/matthiasn/systems-toolbox)** libraries and all of the sample applications, and I'm very excited how much more sense the entire approach makes after THE source of errors I had dealt with just fall by the wayside. The biggest problem had always been maps not structured as expected, and me as the developer having to keep those expectations in my head, rather than have the program do it for me. The capacity of my brain is way too small to keep such stuff in working memory.

In particular, I had those issues while working on my latest application, **[iWasWhere](https://github.com/matthiasn/iWasWhere)**, which is an application for helping me get more done while being happier, in a geolocation-aware context. No worries, I'll get to that as it'll be one of the sample applications of this book. Among other things, **iWasWhere** is supposed to help me reach longer-term goals, such as finishing this very book, and I write it using the **systems-toolbox** libraries. Thus, I expect some cross-pollination between the book and the project. Anyway, building this application luckily put me back in the position of a user of the libraries, and that's a welcome change of perspective for moving it forward. And now with **clojure.spec**, this has become much more pleasant.

Now, this is going to be a **reboot** of the book project. For now, I will move all the existing chapters into the appendix, for reference. However, your time is probably better spent reading the new material, and then joining a discussion about it. I want this book to **help you** approach this beautiful language named Clojure, and I can do that best when you **let me know** what you found unclear, or where you would like additional explanations. To provide **feedback**, you can either contact me directly under <matthias.nehlsen@gmail.com> or create an issue in this book's **[GitHub project](https://github.com/matthiasn/clojure-system-book)**, ideally with file and line number. 

I imagine the **systems-toolbox** libraries to be helpful when you want to build applications like the ones described in this book. For anything that you find missing or where you want something improved, please also consider opening an issue on **[GitHub](https://github.com/matthiasn/systems-toolbox)**.

Now have fun playing around with the sample applications. I'm confident you will **learn the most** when checking out the code, changing stuff here and there, and build something different out of these applications. All of them here are compatible with **[FigWheel](https://github.com/bhauman/lein-figwheel)**, by the way. It's instant feedback make coding all the more gratifying.

Okay, that's all, let me know how I can help!

Matthias

**P.S.** I'm curious about what you will build on top of these libraries. If you have a project you would like me to look at, shoot me an email. If it's an open source project, I'll be happy to do a code review for free. All else can be discussed.
