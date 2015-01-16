# Progress
This section of the book will allow you to keep track of what I'm working on at any moment, what's going well (or not) and what to expect next. I will keep it updated frequently.


## Journal

**Jan 9 to present:** Redesign of the client-side architecture according to **[this blog post](http://matthiasnehlsen.com/blog/2015/01/09/Hairball-Removal/)**. Going well, I have so far removed pretty much all dependencies between namespaces. Only the state namespace is still larger than I'd want it to be. That'll be something to work on this weekend.


## Chapter Status

**Server Side:** I think the architecture is reasonably clean, with dependencies between different parts of the application reduced as much as possible. I am not completely sold on the necessity of the **Component library** though. While the advantages of reloading the application seem compelling, I have not made much use of that yet and haven't in a while. On the client side, I am dealing with the same issues around trying to avoid hairballs of interdependencies and I think I have succeeded, even without the added complexity required by Component. The new solution is not in writing yet though, it just exists in **[master](https://github.com/matthiasn/BirdWatch)** on GitHub as of now since it's not completely finished. 

**Client Side:** Feel free to use this as a bad example. There are way too many dependencies between namespaces. Also, the application state lives in an atom is not properly protected. I do not feel comfortable with freely passing it around because anyone (including myself) using it from anywhere could accidentally mutilate it, causing the application to fail. Instead, I want to protect it and only hand out immutable data for rendering or whatever other purpose. The required redesign is well under way and mostly done in code. Reflecting the changes in the chapters is under way and expected to arrive soon.