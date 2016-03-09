# The systems-toolbox-ui library


## Introduction

Pretty much any application out there has some a user facing aspect. Even those that don't have one would usually greatly benefit from having a user interface that tells you at a glance if your services are healthy. Also, I just generally find it more rewarding to work on something that has some presentation layer, no matter how sparse. So let's make it easy to add a presentation layer to an arbitrary piece of backend code, shall we?


## React

**[React](https://facebook.github.io/react/)** is a UI renderer by Facebook. It **only** does the rendering and is not a Framework for JavaScript applications. However, it does the rendering a) quite well and b) in a way that is particularly useful when coming from a functional programming approach.

Let me explain. What happens is that we define a function that turns some data into a DOM (sub)-tree. There is no notion of manipulating that tree, as you'd tediously have to do with the likes of jQuery. Rather, every time the data changes, the entire thing is rendered. So far, that sounds conceptually pure but at the same time very expensive, right? Well, it would be, only that this rendering does not happen in the actual, heavy-weight DOM but into a lightweight JavaScript data structure called virtual DOM. Then, there's a diffing process that compares the previous version of that virtual DOM (which corresponds to what is on the page) and the latest version, determines the changes, and only then puts those changes into effect on the page. This mechanism happens to be fast. 

Why should you care? In Clojure and ClojureScript, we strongly favor working with immutable data structures. Thus, it conceptually fits very well to think of DOM tree as something immutable, instead of something that you have to modify in place. If there are state changes, we just pass the new state to that function that creates our UI, and then let it figure out under the hood how to get there. For all we need to care about, this UI renderer recreates our UI on every single change. 


## Redux

**[Redux](https://github.com/reactjs/redux)** is one possible approach to creating a larger application that makes use of React as its renderer. Redux is not tied to React but in reality, those libraries are probably most often used together. The general idea here is that there is some protected state container. Mutations of that managed application state then cannot happen from random places in your program. Instead, you can only dispatch actions to that managed state container, which will act on that action (or not, if no such action is defined). This model defines clear borders of where state mutation can happen, and thus makes the application easier to maintain.

I have not worked with Redux extensively, but the approach is the same with the **[systems-toolbox-ui](https://github.com/matthiasn/systems-toolbox-ui)** library, and it works well there. Let's have a look at a simple example next.


## Counter example





