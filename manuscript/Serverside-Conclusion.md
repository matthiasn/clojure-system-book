# Server-side Conclusion

In the previous chapters I have outlined an architecture which, at the time of writing the code, I believed to be appropriate to meet the requirements. Now, some time later, I'm naturally skeptical and want to put this design to the test. Also, there's too much repetition in the code, which is why I started writing the **[systems-toolbox](https://github.com/matthiasn/systems-toolbox)** library, which will be described in subsequent chapters.

First of all, I will have to describe the client side though, so that we get a better understanding of the entire application.

Once that's done, I will try to come up with load testing strategies so that we'll have a much better idea of what this architecture can do. How many simultaneous clients can it handle, for example? I have no idea but I can't wait to find out. All I know is that I've done some tests with 50 concurrent connections while subscribing to a term that maxed out the Streaming API connection and the application wasn't very busy at all. So it appears the application could have handled a lot more. Exactly how many more is what I'd like to find out in subsequent chapters.
