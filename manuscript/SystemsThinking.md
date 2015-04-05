# Systems Thinking

When I started writing this book in December 2014, I did not have a clear idea about what the overarching idea would be, or why I had even mentioned the word **System** in the title. But at some point it dawned on me. Maybe four years ago or so, I had read **[Thinking in Systems by Donatella Meadows](http://www.amazon.com/Thinking-Systems-Donella-H-Meadows/dp/1603580557/ref=sr_1_1?ie=UTF8&qid=1424720251&sr=8-1&keywords=thinking+in+systems)** [^ThinkingInSystemsDraft] and remembered that I found the book enjoyable to read and the ideas presented useful. So I re-read it, which surprised me in terms of just how applicable **systems thinking** is to building a software system.

But don't take my word for it. Instead, I have assembled a few quotes so you can see for yourself. First, let's see how Donatella Meadows defines a system:

> "A system isn't just any old collection of things. A system is an interconnected set of elements that is coherently organized in a way that achieves something. If you look at that definition closely for a minute, you can see that a system must consist of three kinds of things: elements, interconnections, and a function or purpose."
>
> Meadows, Donatella H. (2008) Thinking in Systems: A Primer - Page 11

What can I add? The definition applies not only to an entire application, but also to individual parts or components of a system. But just the same way, the definition applies to separate applications that interact with each other. No matter at which level, we always deal with systems and these systems, no matter at which level, share certain properties.

Also quite important is the notion that a system expresses behavior and that that behavior changes over time:

> "The behavior of a system is its performance over time--its growth, stagnation, decline, oscillation, randomness, or evolution."
>
> Meadows, Donatella H. (2008) Thinking in Systems: A Primer - Page  88

What good is any software if it's not run? The code that we write is a blueprint for a system, not a system itself. Code becomes a system when we breathe life into it running it. Only then will we be able to observe its performance. Code that can't be observed while running is meaningless.

Now that we've established that a system becomes valuable only when it runs, we should be able to see that keeping the system alive is a valuable thing and that we should put thought into the ability of the system to stay up and alive. This is resilience:

> "Awareness of resilience enables one to see many ways to preserve or enhance a system's own restorative powers."
>
> Meadows, Donatella H. (2008) Thinking in Systems: A Primer - Page 78

An interesting part of resilient systems is the existence of feedback loops that allow the system to adapt to the circumstances:

> "The structure of a system is its interlocking stocks, flows, and feedback loops."
>
> Meadows, Donatella H. (2008) Thinking in Systems: A Primer - Page 88

Then, there are those system that don't behave nicely:

> "But some systems are more than surprising. They are perverse. These are the systems that are structured in ways that produce truly problematic behavior; they cause us great trouble."
>
> Meadows, Donatella H. (2008) Thinking in Systems: A Primer - Page 111

Does that remind you of one or more projects you've worked on in the past? I definitely had that feeling when I read this. This could be said about multiple systems I've had the dubious pleasure to work on in the past. How can these problems be dealt with?

> "Understanding archetypal problem-generating structures is not enough. Putting up with them is impossible. They need to be changed. The destruction they cause is often blamed on particular actors or events, although it is actually a consequence of system structure. Blaming, disciplining, firing, twisting policy levers harder, hoping for a more favorable sequence of driving events, tinkering at the margins-these standard responses will not fix structural problems. That is why I call these archetypes 'traps'."
>
> "But system traps can be escaped-by recognizing them in advance and not getting caught in them, or by altering the structure-by reformulating goals, by weakening, strenghtening, or altering feedback loops, by adding new feedback loops. That is why I call these archetypes not just traps, but opportunities."
>
> Meadows, Donatella H. (2008) Thinking in Systems: A Primer - Page  111

One interesting thing I learned is that apparently in the Netherlands, some houses have visible electricity meters and some don't. Who knew that this actually makes a difference?

> "With no other differences in the houses, electricity consumption was 30 percent lower in the houses where the meter was in the highly visible location in the front hall."
>
> Meadows, Donatella H. (2008) Thinking in Systems: A Primer - Page 156

I think this also applies to developing software in teams. _Hey, why is your code hogging resources like there's no tomorrow?_ Do you really want to be the guy who has a hunch that the code that someone elses wrote burns substantially more CPU cycles than would be reasonable and then have to prove that? I didn't think so. How about if the **Definition of Done** included the implementation of metrics so that instead of guessing, we could see right away where the application spends its time.

How do we get better at dealing with systems?

> "Magical leverage points are not easily accessible, even if we know where they are and which direction to push on them. There are no cheap tickets to mastery. You have to work hard at it, whether that means rigorously analyzing a system or rigorously casting off your own paradigms and throwing yourself into the humility of not-knowing. In the end, it seems that mastery has less to do with pushing leverage points than it does with strategically, profoundly, madly, letting go and dancing with the system."
>
> Meadows, Donatella H. (2008) Thinking in Systems: A Primer - Page 165

Lovely, really. I like that picture of dancing with the system. Have you ever felt that when making changes for the better in a piece of software? I certainly have.

Another valuable insight is where control lies. We really can't control everything:

> "Systems can't be controlled, but they can be designed and redesigned. We can't surge forward with certainty into a world of no surprises, but we can expect surprises and learn from them and even profit from them. We can't impose our will on a system. We can listen to what the system tells us, and discover how its properties and our values can work together to bring forth something much better than could ever be produced by our will alone."
>
> Meadows, Donatella H. (2008) Thinking in Systems: A Primer - Page 169

I think that the quote above is a rather strong argument against type systems. This may sound far-fetched, but let me explain. I've experienced this over and over and over again when working with a strongly typed language that the initial assumptions on, for example, how best to structure an application turned out to be just plain wrong. Hooray, at least we have strong types... And then, you find yourself refactoring ten different places and realize that you've really dug yourself in a hole because you've added a massive extra layer of complexity that really only gets in the way when you realize that your initial assumptions were less than genius.

When I work with Clojure, I find that I dread changes much less than in a language like Scala. Change happens and it always will. Let's embrace that instead of trying to cement a structure that is hard to code yourself out of.

How do we find out that we need to change stuff? One way is by properly observing the system. I really like this quote here:

> "Before you disturb the system in any way, watch how it behaves."
>
> Meadows, Donatella H. (2008) Thinking in Systems: A Primer - Page 170

I couldn't justify monitoring the system any better than this. The code is just code, it is not the system. We will never be able to fully understand a system by reading code alone; we will need to understand how it behaves under different circumstances. And then, we will need to be open to let go when we find that something does not behave as intended:

> "Getting models out into the light of day, making them as rigorous as possible, testing them against the evidence, and being willing to scuttle them if they are no longer supported is nothing more than practicing the scientific method-something that is done too seldom even in science, and is done hardly at all in social science or management or government or everyday life."
>
> Meadows, Donatella H. (2008) Thinking in Systems: A Primer - Page 172

In Clojure, the process above feels a lot more natural when working with a few built-in abstractions for data such as maps, vectors, lists, and sets than when you have to engineer classes for every possible data represention you may encounter in your system. Once again, I think that type systems inevitably slow you down and hamper your ability to adapt to change. At least that's my experience from working in Scala for two years before deciding to go with Clojure instead.

> "What's appropriate when you're learning is small steps, constant monitoring, and a willingness to change course as you find out more about where it's leading."
>
> Meadows, Donatella H. (2008) Thinking in Systems: A Primer - Page 180 

Fully agreed. Have you ever worked on a system over a period of months where all the initial assumptions have proven correct? The only way that I could imagine this to happen is if the system never goes live.

Obviously, systems thinking is not a panacea:

> "Systems thinking can only tell us to do that. It can't do it. We're back to the gap between understanding and implementation. Systems thinking by itself cannot bridge that gap, but it can lead us to the edge of what analysis can do and then point beyond-to what can and must be done by the human spirit."
>
> Meadows, Donatella H. (2008) Thinking in Systems: A Primer - Page 185

So far for the systems thinking book. I would recommend reading it. While it's not at all about software, a lot of things still apply because at the end of the day, software is also about systems on different levels.

## Intervention points

What's the most interesting thing to monitor for? In a given system where the individual processing units can become a bottleneck and wait, I want to know how long each processing step takes. Wait times should be listed individually. In the case of core.async, I want to know the overall amount of time it takes to iterate through a go-loop and also, if blocking puts are used and how long the wait time is (indicating congestion down the line).

Could parallelity be used as an intervention point? Yeah, quite likely. If an individual step takes too long, we could fire up more parallel instances of the processing loop while reducing the parallelity in other parts of the system. This way, the system becomes adaptive within reasonable boundaries.

Buffer sizes are equally important. We could just sunset a go-loop with an attached buffer by letting it run out while at the same time creating a new one for further processing with a different buffer size and have that new one process incoming messages henceforth. Or block until the old one has run out in order to avoid compromises in potential ordering guarantees we may rely on. Hopefully, we don't rely on them, though, as they would preclude parallelity, which is pretty incompatible with such guarantees.

In the systems thinking book, buffers aren't high on the list but mostly to them being inflexible in the real world, such as a dam. In a dynamic software system, they consist merely of thought, so they can be changed at will.

When automating, introduce delays. No fast decision in order to avoid oscillating behavior. Maybe even introduce an amount of randomness in the strategies involved within reasonable boundaries. That should make oscillating behavior even less likely. Or vary the duration between decision points about corrective action. That should at least dampen any tendency of the system to oscillate between extremes.

According to the definition in the systems thinking book these parts of the system adapting to change balance feedback loops.

How about associating costs to parallelity factors and buffer sizes? These are costly to the system as a whole (more threads or memory bound). Each agent can then earn cash by providing system services and burn it for requesting resouces. And then have itself optimize for profitability, while trying to meet SLAs as closely as possible.

[^ThinkingInSystemsDraft]: I've also found a draft of the book as a **[pdf](http://www.natcapsolutions.org/Presidio/Articles/WholeSystems/ThinkingInSystems_MEADOWS_TiS%20v13.2_DRAFT.pdf)** but I don't know if it's actually legal to download it from there.
