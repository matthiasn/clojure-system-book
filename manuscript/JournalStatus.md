# Progress
This section of the book will allow you to keep track of what I'm working on at any moment, what's going well (or not) and what to expect next. I will keep it updated frequently.

## Journal
**Feb 14th onwards**: introducing the concept of systems thinking.

**Feb 10th onwards**: basic monitoring added within codebase on GitHub.

**Feb 1st onwards**: preparations for Docker: setting up Linux development environment. Also: conception of load tests.

**Jan 31st**: Client chapters ready for review and feedback.

**Jan 28th to Jan 30th**: Rewrite of the UI chapters on the client side.

**Jan 27th**: Rewrite of the State chapter.

**Jan 26th:** Rewrite of the Communicator chapter.

**Jan 22nd to Jan 25th:** Architectural drawing for client-side architecture and rewrite of core namespace article.

**Jan 21th:** Proofreading of client intro.

**Jan 20th:** New introduction for the client-side chapter, including introduction to managing and protecting application state. The text is not proofread yet, so it hasn't made its way into the book yet, but if you're interested, you can already read it on **[GitHub](https://github.com/matthiasn/clojure-system-book/blob/master/manuscript/Client-Architecture.md).

**Jan 16th to Jan 19th:** Refactoring of the *state* namespace. It is now split up to have each involved namespace only handle a distinct aspect of state management. The subscribers to state updates, such as UI elements, do not have access to the atom that keeps the application state any longer, instead they will only be provided with the dereferenced state, which is immutable. Thus, the application state cannot be erroneously mutilated from parts of the system that do not have anything to do with managing application state. Drawings and a rewritten client side chapter are expected soon. Only read the current version when you want to get to know the not-so-great client code first in order to compare it with the better version later.

**Jan 9th to Jan 15th:** Redesign of the client-side architecture according to **[this blog post](http://matthiasnehlsen.com/blog/2015/01/09/Hairball-Removal/)**. Going well, I have so far removed pretty much all dependencies between namespaces. Only the state namespace is still larger than I'd want it to be. That'll be something to work on this weekend.

## Chapter Status
**Server Side:** I think the architecture is reasonably clean, with dependencies between different parts of the application reduced as much as possible. **Ready for feedback.**

**Client Side:** The client-side application has been rewritten to reflect the changes mentioned in the new introduction. **Ready for feedback.**
