# Server-side Architecture

## Architectural Overview

Let's start with the basic architecture of the server side. Here's an overview:

![channels overview](images/bw-channels.png)

You can see an animated version of this drawing in the **[original blog post](http://matthiasnehlsen.com/blog/2014/09/24/Building-Systems-in-Clojure-1/)** that demonstrates how components in the system get wired up when the application initializes.

In the initial version that I wrote, where everything depended on everything, things were very different. Some people would call that "spaghetti code", but I think that is not doing justice to spaghetti. Unlike bad code, I don't mind touching spaghetti. I would rather liken bad code to hairballs, of the worst kind that is. Have you ever experienced the following: you are standing in the shower and the water doesn't drain. You notice something in the sink, so you squat down to pull it out only to start screaming, "Oh my god, it's a dead rat" a second later. I am referring to that kind of entangled hairball mess, nothing less. On top, you may even hit your head when you jump up in disgust. 

This is where dependency injection comes in. Can we agree that we don't like hairballs? Good. Usually, what we are trying to achieve is a so-called inversion of control, in which a component of the application knows that it will be injected something which implements a known interface at runtime. Then, no matter what the actual implementation is, it knows what methods it can call on that something because of the implemented interface.

Here, unlike in object-oriented dependency injection, things are a little different because we don't really have objects. The components play the role of objects, but as a further way of decoupling, I wanted them to only communicate via **core.async** channels. Channels are a great abstraction. Rich Hickey likens them to conveyor belts onto which you put something without having to know at all what happens on the other side. We will have a more detailed look at the channels in the next article. For now, as an abstraction, we can think about the channel components (the flat ones connecting the components with the switchboard) as **wiring harnesses**, like the one that connects the electronics of your car to your engine. The only way to interface with a modern engine (that doesn't have separate mechanical controls) is by connecting to this wiring harness and either send or receive information, depending on the channel / cable that you interface with.

