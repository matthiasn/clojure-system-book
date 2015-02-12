# Appendix

## Introduction to Clojure
This section first appeared in a [blog article](http://matthiasnehlsen.com/blog/2014/07/24/birdwatch-cljs-om/)** of mine.

First of all, you will need to understand a few very basic things about Clojure being a **[Lisp](http://en.wikipedia.org/wiki/Lisp_(programming_language\))**. My aim is for you to be able to follow along even if you've never tried Clojure or a Lisp before. So the basic idea in a Lisp is the List (no wonder, as Lisp stands for List Programming), a **[singly linked list](http://en.wikipedia.org/wiki/Singly_linked_list#Singly_linked_lists)**, to be precise. This list can hold both code and data. Let's see how that looks like. You can try these examples out using the **[REPL](http://en.wikipedia.org/wiki/Read–eval–print_loop)** in **[Leiningen](http://leiningen.org)** by running ````lein repl```` from your command line.

This is an empty list: ````()```` It evaluates to itself.

When the list is not empty, the first item in the list will be evaluated as a function: ````(some-function "a" "b")```` 
Here, *some-function* will be called with the two arguments "a" and "b". Example ````(print "Hello World!")```` Sweet, that is all there is to **[Hello World](http://en.wikipedia.org/wiki/Hello_world_program)**.

The first item in a list has to implement the *IFn* interface meaning it must be possible to call the item as a function. Try this: ````("a" "b")````. Not surprisingly, the string "a" is not a function, causing this to fail. You can however **quote** the list to prevent evaluation, like this: ````'("a" "b")````. Now we can use the list to store items without the first one being evaluated.

Conveniently, Clojure also has a **vector** which is comparable to an array. You can use it in place of a quoted list, and in fact it is idiomatic to do so when we do not want the first item to be evaluated. Example ````[1 2 3]````

When you want to name something, you have different options available. The first one is **def**; you can use this to name stuff in the top level of a namespace, for example ````(def foo [1 2 3])```` This will create a vector named *foo* which you can then refer to from elsewhere. After typing in the previous example, you will see that now you can just type ````foo```` in the REPL and get the vector we have defined previously.

Or you can use the **let-binding** to name things locally, for example inside a function body, like this: ````(let [bar [1 2 3]])```` Here, you can only refer to *bar* inside the let form, meaning inside the pair of braces that enclose the let form. Let's use *bar*: ````(let [bar [1 2 3]] (print bar))```` You should see the vector being printed in your REPL.

Functions can be defined as follows: ````(fn [a] (+ a 1))```` with this, we have defined a function that adds 1 to the argument provided .
You can use the above as an anonymous function like this: ````((fn [a] (+ a 1)) 2)````. Remember that the first item in a list will be evaluated. This is what happens to be the anonymous function we have just defined. However, this can be a little clumsy. We can also store the function in a def: ````(def add-one (fn [a] (+ a 1)))````; now we can call the function like this: ````(add-one 2)````.

However, this can even be simpler if we use the **defn macro**: ````(defn add-one [a] (+ a 1))````

Sometimes, you may want to create a function in place using the anonymous function literal: ````(#(+ % 1) 2)````. This does the same as the anonymous function in the first position of the list above, except that it is shorter. During compilation the ````#(+ % 1)```` expands into ````(fn [a] (+ a 1))````, where the percent sign denotes the first argument. If there are multiple arguments, you use *%1*, *%2* and so on (1-based).

No language would be complete if there wasn't a way to make decisions and branch off accordingly. Of course, Clojure has constructs for flow control as well, most notably the **[if special form](http://clojure.org/special_forms#Special%20Forms--(if%20test%20then%20else?\))**. It is really quite simple: ````(if test then else?)````. The test will be evaluated first. Then based on the result, either **then** or **else** are yielded. Other constructs derive from it, such as the **[when macro](http://clojure.github.io/clojure/clojure.core-api.html#clojure.core/when)**.

With these basic constructs you may already find yourself in the position to follow the subsequent source code. Clojure is not hard. Please let me know if you have problems following along. If so, it's most likely not you but rather the fact that my intro wasn't good enough.

Of course, there's a lot more to the language and plenty of stuff to learn when you are ready to delve deeper. As a next step, I suggest **[Learn Clojure in Y Minutes](http://learnxinyminutes.com/docs/clojure/)**. There, you simply have more examples to follow along and play around with in your REPL.

As another online resource, I have also found **[Clojure for the Brave and True](http://www.braveclojure.com)** to be fun and helpful. Check it out. And if you like it, why not support the author and buy his ebook? You can then enjoy the book on your favorite ebook reader as well, and even if you read it on the web, you will ensure that the author can keep up the good work. Great feeling, I did the same.

Another great resource is **[Joy of Clojure, 2nd edition](http://r.matthiasnehlsen.com/joyclojure/link)**. Fun read and I learned a lot. I will revisit it and share some thoughts about it in the **[reviews section](/reviews)** soon.

## Chapter Status History
**Server Side Chapters before February 12th:** I think the architecture is reasonably clean, with dependencies between different parts of the application reduced as much as possible. I am not completely sold on the necessity of the **Component library** though. While the advantages of reloading the application seem compelling, I have not made much use of that yet and haven't in a while. On the client side, I am dealing with the same issues around trying to avoid hairballs of interdependencies and I think I have succeeded, even without the added complexity required by Component. On the other hand, using Component adds good structure. Maybe I'd remove the channel components and have the channels provided by each component directly. I'm curious about your thoughts on this subject.

**Client Side Chapters before Jan 31st:** The client-side application has been rewritten to reflect the changes mentioned in the new introduction. They are mostly complete, except for not all proofreading being completed yet. Also, the client summary is still missing.

**Client Chapters before Jan 30th:** The client-side application is rewritten to reflect the changes mentioned in the new introduction. The chapters about the ````core````, ````communicator````, and ````state.*```` namespaces have been rewritten. Approach the remaining chapters with care, they are about to change a lot.

**Client Side (Jan 27th, 2015):** The client-side application is rewritten to reflect the changes mentioned in the new introduction. The chapters about the ````core```` and ````communicator```` namespaces have been rewritten. Approach the remaining chapters with care, they are about to change a lot.

**Client Side (Jan 21, 2015):** The client-side application is rewritten to reflect the changes mentioned in the new introduction. The new introduction will now undergo proofreading before landing in the book. For now, read it on  **[GitHub](https://github.com/matthiasn/clojure-system-book/blob/master/manuscript/Client-Architecture.md) instead. Approach the existing chapters about the client side with care, they are about to change a lot.

**before Jan 21st, 2015:**
Feel free to use this as a bad example. There are way too many dependencies between namespaces. Also, the application state lives in an atom is not properly protected. I do not feel comfortable with freely passing it around because anyone (including myself) using it from anywhere could accidentally mutilate it, causing the application to fail. Instead, I want to protect it and only hand out immutable data for rendering or whatever other purpose. The required redesign is done in code, it's in **[master](https://github.com/matthiasn/BirdWatch)** on GitHub. Chapter rewrites and archtectural drawings due very soon.
