# Testing

In this chapter, I will discuss testing strategies for both the server and the client parts of a system. Let me start with a confession. I have to admit that I am not an adamant disciple of **TDD**. In theory, that all sounds great, but I just haven't seen many real world examples of convincing test suites in the last couple of years. 

If you do have a setup that gives you confidence in making changes, that is awesome. I don't buy that the confidence comes from the test suite alone, though. Rather, trust in changing an existing codebase comes from the feeling that you understand the codebase. Sure, green tests assure you that you did not break anything **for which there are tests**. Depending on how well thought-through a test suite is, that may mean a lot or absolutely nothing. But if your codebase is an unintelligible mess, no amount of tests will give me any confidence in anything. Proper reasoning and having an overall strategy of what you want to achieve is far more important IMHO. Let's have a look what Rich Hickey has to say about testing in **[Simple Made Easy](https://github.com/matthiasn/talk-transcripts/blob/master/Hickey_Rich/SimpleMadeEasy.md)**:

> And I like to ask this question: What's true of every bug found in the field?

[Audience reply: Someone wrote it?] [Audience reply: It got written.]

> It got written. Yes. What's a more interesting fact about it? It passed the type checker.

[Audience laughter]

> What else did it do?

[Audience reply: (Indiscernible)]

> It passed all the tests. Okay. So now what do you do? Right? I think we're in this world I'd like to call guardrail programming. Right? It's really sad. We're like: I can make change because I have tests. Who does that? Who drives their car around banging against the guardrail saying, "Whoa! I'm glad I've got these guardrails because I'd never make it to the show on time."

[Audience laughter]

> Right? And - and do the guardrails help you get to where you want to go? Like, do guardrails guide you places? No. There are guardrails everywhere. They don't point your car in any particular direction. So again, we're going to need to be able to think about our program. It's going to be critical. All of our guardrails will have failed us. We're going to have this problem. We're going to need to be able to reason about our program. Say, "Well, you know what? I think," because maybe if it's not too complex, I'll be able to say, "I know, through ordinary logic, it couldn't be in this part of the program. It must be in that part, and let me go look there first," things like that.

> -- <cite>Rich Hickey, 2011</cite>

If you haven't seen this talk, please do that now. Or read the transcript or both. It's worth it.

Of course, tests aren't useless either. There can be value in them, but you do need to think about testing the right stuff. Just having 2,000 (poorly conceived) tests means little for the amount of confidence your software deserves.

## Unit Tests

In general, I find unit tests overused. How deeply do we want to test implementation details? I find this particularly bothersome when testability guides how you write the code in the first place. I have just seen it too often that a particular part of a codebase became way too rigid and difficult to change for no reason other than testing some rather unimportant implementation detail. YMMV.


## Integration Tests

Integration tests, on the other hand, can be very helpful, particularly when working on the same codebase with multiple teams. I found that to be the case even more so with Clojure. I don't miss a type system generally, but there is something nice about, say, defining a class for an individual exchange of information in your program. Sure, a map is more flexible, but with flexibility comes great responsibility. With a class, you, at least, know what's supposed to be in it. When what you pass is a map, you should honor some conventions and not arbitrarily change stuff just because it's a full moon, and when the moon is full, you prefer date format y over date format x or whatnot. Even if you change the tests for your web service over to your idiotic date format y, you still break downstream consumers of your API. By the way, your adapted unit tests will not have helped this at all.

However, this is where the integration tests for the downstream users come in handy. You should not be able to make such a change that breaks the API users AND merge it to a common branch. Rather, the consumer should have a set of high-level integration or acceptance tests that need to pass before you can merge any change back to the master branch.

Such integration tests need to meet a few criteria:

1) The integration tests need to be reliable so we can trust them. That means that if they are not, your team has to stop everything else they are doing until the tests are trustworthy again, even if management finds other matters more important. Ultimately, we are responsible for a broken environment, not management.

2) All integration tests need to run all the time. There needs to be a continuous integration environment that runs tests on all commits in all branches. Also, it should be feasible to run the test suite on your development laptop, without blocking it for longer than a handful of minutes, maximum.

3) For the above to be feasible, the test suite needs to be quick to execute. Tests that take an hour to run and are therefore never executed are utterly useless. There's also little justification for that kind of situation. Some people believe that every test should start with a clean slate, but that's particularly harmful in Clojure. Yes, the time required to start anything Clojure on the JVM sucks. But why then would anyone want to start a fresh instance of your backend for each test??? That's probably okay when you start something written in C that fires up in a fraction of a second but starting a JVM with a Clojure service can easily take ten seconds or more. How can this be a good idea to do hundreds of times? Also, my expectation against a backend is that it can deal with all conceivable testing scenarios in any order. Rather than fire up some service over and again, I'm strongly for firing up the artifact that will be deployed just once and observe if it does what it is expected to do.

4) Merges to the master branch that break tests must be forbidden, even when it's the test suite of another team that happens to depend on your code. For that, it's best to have CI vote against your change when not all tests pass.


## Continuous Integration

One important part of building a software system is building the environment that allows us to be confident that it works. For this, continuous integration is essential. There are many options out there. When you really want to host the CI environment yourself, I have good experiences with **[Jenkins](https://jenkins-ci.org/)**, which is open source and free. If you use **[Jira](https://www.atlassian.com/software/jira)**, you may want to have a look into **[Bamboo](https://www.atlassian.com/software/bamboo)**, which integrates nicely. I'm just not so sure I like Jira, but that's an entirely different story. 


## Hosted CI

After wasting an incredible amount of precious lifetime on the internally hosted CI environment in my last consulting gig, I thought I should have a look at some hosted options for continuous integration. Luckily, the better ones host open source projects for free, so I set up two with GitHub integration for the systems-toolbox library. So far, both seem to do the job, and both support open source projects for free.

Unless you're in the business of building CI servers, I can only recommend you spend your time and resources on the problems you want to solve, rather than wasting time on tweaking your own, poor CI environment. Otherwise, you're not only blocking the guys trying to set up your CI nodes, but also everyone else who's waiting for them.

**DISCLAIMER**: I don't receive money from either, nor do I know anyone in either company.


### TravisCI

![](images/testing/travis-ci.png)

**[TravisCI](https://travis-ci.org/matthiasn/systems-toolbox)** quite easy to use. Once you've signed up, all you need to do is add a YAML file named `.travis.yml` to your repository. In the case of the systems-toolbox, it looks as follows:

~~~
language: clojure
lein: lein2
script: lein2 test
jdk:
  - oraclejdk8
~~~

This file defines that the project is written in Clojure, that we want Leiningen 2 to run the test, and that we want the test to run on an Oracle JDK 8. Once this file exists, TravisCI will happily test your Clojure repository on every commit ever after. Oddly, as of this writing, surprisingly there was no `openjdk8` available.

Note that it will take longer for the tests to run on a hosted CI server because first, Leiningen will have to resolve the dependencies by talking to Clojars. Currently, running the tests locally take about 10 seconds whereas running them on TravisCI take a little under a minute.

I'm missing an easy way to have JUnit reports available in TravisCI. The `lein test2junit` task can create JUnit XML files, which we can then turn into an HTML report using the `ant` command from the root of the project. However, with TravisCI you need to set up Amazon's S3 to upload artifacts, as outlined **[here](https://docs.travis-ci.com/user/uploading-artifacts/)**.


### CircleCI 

![](images/testing/circle-ci.png)

**[CircleCI](https://circleci.com/gh/matthiasn/systems-toolbox/tree/master)** feels quite similar to TravisCI. It is also configured with a YAML file in the root of your project, only that here it is called `circle.yml`. For the systems-toolbox, it looks as follows:

~~~~
test:
  override:
    - lein test2junit
  post:
    - ant
~~~~

Here, I'm using the `lein test2junit` task to generate JUnit-style test reports. Then, once the test has completed, I run `ant` to create an HTML report, which will then be available with the artifacts of the particular build. For that, a small modification of the `project.clj` file is also required:

~~~
  :test2junit-output-dir ~(or (System/getenv "CIRCLE_TEST_REPORTS") "target/test2junit")
~~~

With these modifications, we can now keep the JUnit reports without having to set up S3.

![](images/testing/circle-artifacts.png)

![](images/testing/circle-junit.png)


### Conclusion

If you want JUnit reports, you probably want CircleCI. If not, TravisCI seems to be an equally fine option and the choice is up to your taste. However, and that is the most important takeaway here, if you enjoy writing Clojure, then don't waste your time on maintaining a flaky CI environment yourself. For open source projects, this is a complete no-brainer, but even if you have to pay for one of the plans because you're working on a closed-source solution, the engineers on your team cost money, too. Plus the hardware, electricity and all.


## Testing the systems-toolbox library

Okay, I admit that I'm not necessarily doing things in the right order here. I started writing this library a little over a year ago and this didn't happen in a **TDD** way. I don't even necessarily feel too bad about it, as there is a fairly comprehensive test suite for the first "real" application using it and that ever growing test suite has been run thousands of times. Thus, I am fairly confident that the library does what it suggests it does. However, that test suite just takes too long, with a lot of browser-based tests, which makes me crave an independent test suite that gives me confidences in changes in less time than it takes to go to the kitchen and grab a cup of black coffee. On the other hand, the good thing about not having all the tests in place is that I can write about it.

As of this writing, there are some tests for the component and the scheduler namespaces. Before completing the functionality, I'd like to take a step back and look at what I'm testing. Here we have a library that is written entirely in **cljc**, yet the tests so far are written in **clj** and thus only run on the JVM. **That's not right.** I only want to invest the effort in more comprehensive testing because I expect to reap the benefit of restoring confidence after a potentially breaking change. But, and this I find very important when writing any of your valuable code in **cljc** so that you can use it on either platform, you absolutely must test it on both platforms. Everything else is half-baked and probably more harmful than anything. Or can you imagine the disappointment when you get handed some logic in `.cljc` that is "tested" so you'd expect it to work in the browser, too, only to find out that that promise has been complete baloney, and rather than just use it to meet your deadline, you start hating your colleague who only after your discovery tells you, "well, we've not actually tested it in the browser". Yeah seriously, that's not cool.

So, before adding tests that potentially only run on the JVM, I'd rather fix the situation by rewriting the existing tests to run on either platform, and only then make the test suite more comprehensive.


### Porting existing tests to cljc, running tests with doo

I've established above why I want to test code written in **cljc** on both the **JVM** and at least one **JavaScript Engine**, and probably all the one's that are relevant for my use case. Then I looked around and luckily found **[doo](https://github.com/bensu/doo)**, which makes testing on a JS engine much easier than last time I checked (and shied away).

Adding doo is easy, you add it to the plugins section in `project.clj`:

~~~
  :plugins [[lein-codox "0.9.4"]
            [test2junit "1.2.1"]
            [lein-doo "0.1.6"]
            [lein-cljsbuild "1.1.2"]]
~~~

And then you add a build config for it:

~~~
  :cljsbuild {:builds [{:id           "cljs-test"
                        :source-paths ["src" "test/cljs"]
                        :compiler     {:output-to     "out/testable.js"
                                       :main          matthiasn.systems-toolbox.runner
                                       :optimizations :whitespace}}]}
~~~

Here, I've added an initial test in the `test/cljs` path. Then, there's a runner namespace, in which we define the tests to call:

~~~
(ns matthiasn.systems-toolbox.runner
  (:require [doo.runner :refer-macros [doo-tests]]
            [matthiasn.systems-toolbox.test]))

(doo-tests 'matthiasn.systems-toolbox.test)
~~~

Before converting tests, let's try something simple.

~~~
(ns matthiasn.systems-toolbox.test
  (:require [cljs.test :refer-macros [deftest is]]))

(deftest do-i-work
  (is (= 2 2)))
~~~

With these namespaces in place, we can now call test tests, for example `$ lein doo firefox cljs-test once`. Oops, I'm writing this on a machine that didn't have the **karma** test runner installed. If you can't run `$ karma -v`, you want want to install it with `$ npm install -g karma`, plus check the further error output that tells you clearly which additional **npm** modules you want to have installed. Or, obviously, if you don't have **npm** available, you want to get it from **[Node.js](https://nodejs.org/en/)** first. With the dependencies met, my initial test runs fine.

Now I should just be able to rename my existing tests to `.cljc` and be off to the pub, right? Not so fast. While the library is written in `.cljc` pretty much from the get-go, that means nothing for the existing tests. And, there we have have it, the tests as of the **[current commit](https://github.com/matthiasn/systems-toolbox/blob/994ff8d698d1fa3f4b1d32d706f63de72bb283a4/test/matthiasn/systems_toolbox/scheduler_test.clj)** at the time of writing use **promises**. Such a shame those are platform-specific and only exist on the JVM. Hmm, let's see, can we replace them **core.async**? Probably.

