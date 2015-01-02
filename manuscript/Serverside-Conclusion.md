# Server-side Conclusion

In the previous chapters I have outlined an architecture which, at the time of writing the code, I believed to be appropriate to meet the requirements. Now, some time later, I'm naturally skeptical and want to put this design to the test.

I will have to describe the client side first, though, so that we get a better understanding of the entire application. Expect further work on the client-side chapters in January 2015.

Once that's done, I will try to come up with load testing strategies so that we'll have a much better idea of what this architecture can do. How many simultaneous clients can it handle, for example? I have no idea but I can't wait to find out. All I know is that I've done some tests with 50 concurrent connections while subscribing to a term that maxed out the Streaming API connection and the application wasn't very busy at all. So it appears the application could have handled a lot more. Exactly how many more is what I'd like to find out in subsequent chapters.

