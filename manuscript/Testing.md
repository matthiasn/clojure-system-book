# Testing

In this chapter, I will discuss testing strategies for both the server and the client parts of a system. Let me start with a confession. I have to admit that I am not an adamant disciple of **TDD**. In theory, that all sounds great, but I just haven't seen many real world examples of convincing test suites in the last couple of years. 

If you do have a setup that gives you confidence in making changes, that is awesome. I don't buy that the confidence comes from the test suite alone, though. Rather, trust in changing an existing codebase comes from the feeling that you understand the codebase. Sure, green tests assure you that you did not break anything **for which there are tests**. Depending on how well thought-through a test suite is, that may mean a lot or absolutely nothing. But if your codebase is an unintelligible mess, then no amount of tests will give me any confidence in anything, especially since bad coding practices that make code hard to reason about usually extend to the tests as well. I have just been in the situation while there were tests for some area of the codebase, the tests made little sense and did not keep anyone from introducing breaking changes that only later came back as bugs to fix. Such inadequate tests are more harmful than anything, as they give you false confidence.

Proper reasoning and having an overall strategy of what you want to achieve is far more important IMHO. Let's have a look what Rich Hickey has to say about testing in **[Simple Made Easy](https://github.com/matthiasn/talk-transcripts/blob/master/Hickey_Rich/SimpleMadeEasy.md)**:

> And I like to ask this question: What's true of every bug found in the field?

[Audience reply: Someone wrote it?] [Audience reply: It got written.]

> It got written. Yes. What's a more interesting fact about it? It passed the type checker.

[Audience laughter]

> What else did it do?

[Audience reply: (Indiscernible)]

> It passed all the tests. Okay. So now what do you do? Right? I think we're in this world I'd like to call guardrail programming. Right? It's really sad. We're like: I can make change because I have tests. Who does that? Who drives their car around banging against the guardrail saying, "Whoa! I'm glad I've got these guardrails because I'd never make it to the show on time."

[Audience laughter]

> Right? And - and do the guardrails help you get to where you want to go? Like, do guardrails guide you places? No. There are guardrails everywhere. They don't point your car in any particular direction. So again, we're going to need to be able to think about our program. It's going to be critical. All of our guardrails will have failed us. We're going to have this problem. We're going to need to be able to reason about our program. Say, "Well, you know what? I think," because maybe if it's not too complex, I'll be able to say, "I know, through ordinary logic, it couldn't be in this part of the program. It must be in that part, and let me go look there first," things like that.

> -- *Rich Hickey, 2011*

If you haven't seen this talk, please do that now. Or read the transcript or both. It's worth it.

Of course, tests aren't useless either. There can be value in them, but you do need to think about testing the right stuff. Just having 2,000 (poorly conceived) tests means little for the amount of confidence your software deserves.


## Unit Tests

In general, I find unit tests to be overused. How deeply do we want to test implementation details? I find this particularly bothersome when testability guides how you write the code in the first place. I have just seen it too often that a particular part of a codebase became way too rigid and difficult to change for no reason other than testing some rather unimportant implementation detail. YMMV.


## Integration Tests

Integration tests, on the other hand, can be very helpful, particularly when working on the same codebase with multiple teams. I found that to be the case even more so with Clojure. I don't miss a type system generally, but there is something nice about, say, defining a class for an individual exchange of information in your program. Sure, a map is more flexible, but with flexibility comes greater responsibility. With a class, you, at least, know what's supposed to be in it. When what you pass is a map, you should honor some conventions and not arbitrarily change stuff just because it's a full moon, and when the moon is full, you prefer date format y over date format x or whatnot. Even if you change the tests for your web service over to your idiotic date format y, you still break downstream consumers of your API. By the way, having modified your unit tests with the new date format would not have helped this at all.

However, this is where the integration tests for the downstream users come in handy. You should not be able to make such a change that breaks the API users **and** merge it to a common branch. Rather, there should be a set of high-level integration or acceptance tests that need to pass before you can merge any change back to the master branch.

Such integration tests need to meet a few criteria:

1) The integration tests need to be reliable so we can trust them. That means that if they are not, your team has to **stop everything** else they are doing until the tests are fixed--even if management finds other matters more important. Ultimately, us developers are responsible for a broken environment, not management.

2) All integration tests need to run all the time. There needs to be a continuous integration environment that runs tests on all commits in all branches. In situations where tests aren't reliable, I have found it helpful to set up the CI environment to run all tests every 15 or 30 minutes and then have a look which tests fail most often. By running all tests like an extra hundred times a day will very soon give you the data you need to figure out which of those tests are the most unreliable. Then, you can start fixing them in that order. Sure, it'll increase your electricity bill for a while, but fixing the broken tests will also increase your level of sanity.

3) For the above to be feasible, the test suite needs to be quick to execute. Tests that take an hour to run and are therefore never executed are utterly useless. There's also little justification for that kind of situation. Some people believe that every test should start with a clean slate, but that's particularly harmful in Clojure. Yes, the time required to start anything Clojure on the JVM sucks. But why then would anyone want to start a fresh instance of your backend for each test??? That's probably okay when you start something written in C that fires up in a fraction of a second, but starting a JVM with a Clojure service can easily take ten seconds or more. How can it be a good idea to do this over and again, hundreds of times? Also, my expectation against a backend is that it can deal with all conceivable testing scenarios in any order. Rather than fire up some service over and again, I'm strongly for firing up the artifact that will be deployed just once and observe if it does what it is expected to do. Also, make sure you test the artifact that is supposed to be deployed. If it's anything other than that, then you can't have confidence in the deployable artifact.

4) Merges to the master branch that break tests must be forbidden, even when it's the test suite of another team that happens to depend on your code. For that, it's best to have your CI server automatically vote against your change when not all tests pass.


## Continuous Integration

One important part of building a software system is creating the environment that allows us to be confident that it works. For this, continuous integration is essential. There are many options out there. When you want to host the CI environment yourself, I have good experiences with **[Jenkins](https://jenkins-ci.org/)**, which is open source and free. If you use **[Jira](https://www.atlassian.com/software/jira)**, you may want to have a look into **[Bamboo](https://www.atlassian.com/software/bamboo)**, which integrates nicely. I'm just not so sure I like Jira, but that's an entirely different story. 


## Hosted CI

After wasting an incredible amount of precious lifetime on the internally hosted CI environment in my last consulting gig, I thought I should have a look at some hosted options for continuous integration. Luckily, the better ones host open source projects for free, so I set up two with GitHub integration for the systems-toolbox library. So far, both seem to do the job, and both support open source projects for free.

Unless you're in the business of building CI servers, I can only recommend you spend your time and resources on the problems you want to solve, rather than wasting time on tweaking your own, poor CI environment. Otherwise, you're not only blocking the guys trying to set up your CI nodes, but also everyone else who's waiting for them.

**DISCLAIMER**: I don't receive money from either, nor do I know anyone in either company.


### TravisCI

![](images/testing/travis-ci.png)

**[TravisCI](https://travis-ci.org/matthiasn/systems-toolbox)** is quite easy to use. Once you've signed up, all you need to do is add a YAML file named `.travis.yml` to your repository. In the case of the systems-toolbox, it looks as follows:

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

![](images/testing/circle-ci-artifacts.png)

![](images/testing/circle-ci-junit.png)


### Conclusion

If you want JUnit reports, you probably want CircleCI. If not, TravisCI seems to be an equally fine option, and the choice is up to your taste. However, and that is the most important takeaway here, if you enjoy writing Clojure, then don't waste your time on maintaining a flaky CI environment yourself. For open source projects, this is a complete no-brainer, but even if you have to pay for one of the plans because you're working on a closed-source solution, the engineers on your team cost money, too. Plus the hardware, electricity and all.


## Testing the systems-toolbox library

Okay, I admit that I'm not necessarily doing things in the right order here. I started writing this library a little over a year ago, and this didn't happen in a **TDD** way. I don't even necessarily feel too bad about it, as there is a fairly comprehensive test suite for the first "real" application using it and that ever growing test suite has been run thousands of times. Thus, I am fairly confident that the library does what it suggests it does. However, that test suite just takes too long, with a lot of browser-based tests, which makes me crave an independent test suite that gives me confidences in changes in less time than it takes to go to the kitchen and grab a cup of black coffee. On the other hand, the good thing about not having all the tests in place is that I can write about it.

As of this writing, there are some tests for the component and the scheduler namespaces. Before completing the functionality, I'd like to take a step back and look at what I'm testing. Here we have a library that is written entirely in **cljc**, yet the tests so far are written in **clj** and thus only run on the JVM. **That's not right.** I only want to invest the effort in more comprehensive testing because I expect to reap the benefit of restoring confidence after a potentially breaking change. But, and this I find crucial when writing any of your valuable code in **cljc** so that you can use it on either platform, you absolutely must test it on both platforms. Everything else is half-baked and probably more harmful than anything. Or can you imagine the disappointment when you get handed some logic in `.cljc` that is "tested" so you'd expect it to work in the browser, too, only to find out that that promise has been complete balone? Rather than being able to use the code and meet your deadline, you start hating your colleague who now, only after your discovery, tells you, "well, we've not actually tested it in the browser". Yeah seriously, that's not cool.

So, before adding tests that potentially only run on the JVM, I'd rather fix the situation by rewriting the existing tests to run on either platform and only then make the test suite more comprehensive.


### Porting existing tests to cljc, running tests with doo

I've established above why I want to test code written in **cljc** on both the **JVM** and, at least, one **JavaScript Engine**, and probably all the ones that are relevant to my use case. Then I looked around and luckily found **[doo](https://github.com/bensu/doo)**, which makes testing on a JS engine much easier than last time I checked (and shied away).

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

With these namespaces in place, we can now call test tests, for example `$ lein doo firefox cljs-test once`. Oops, I'm writing this on a machine that didn't have the **karma** test runner installed. If you can't run `$ karma --version`, you want to install it with `$ npm install -g karma`, plus check the further error output that tells you clearly which additional **npm** modules you want to have installed. Or, obviously, if you don't have **npm** available, you want to get it from **[Node.js](https://nodejs.org/en/)** first. With the dependencies met, my initial test runs fine.

Now I should just be able to rename my existing tests to `.cljc` and be off to the pub, right? Not so fast. While the library is written in `.cljc` pretty much from the get-go, that means nothing for the existing tests. And, there we have it, the tests as of the **[current commit](https://github.com/matthiasn/systems-toolbox/blob/994ff8d698d1fa3f4b1d32d706f63de72bb283a4/test/matthiasn/systems_toolbox/scheduler_test.clj)** at the time of writing use **promises**. Such a shame those are platform-specific and only exist on the JVM. Hmm, let's see, can we replace them **core.async**? Probably.


### Promises for testing in ClojureScript?

Using **promises** for determining when the assertions should be made was not a bad idea, were it not for the lack of them on the ClojureScript side. But, **core.async** to the rescue, we can model the behavior of promises ourselves. In core.async, there's a **promise-chan**, which can only be delivered on once. 'put!' then gives us the `deliver` functionality for promises. For finally waiting for either a result or a timeout, which is done by `deref` with promises, we can use `alts!`. Let's look at a super simple **[example](https://github.com/matthiasn/systems-toolbox/blob/af1cb5368628d158141808dfbe2f409effd13511/test/matthiasn/systems_toolbox/component_test.cljc#L15)** first:

~~~
(deftest cmp-all-msgs-handler
  "Tests that a very simple component that only has a handler for all messages regardless of type receives all
  messages sent to the component. State management of the component is not used here, instead we keep track
  of the messages in an atom that's external to the component and that the handler function has access to.
  A promise is used here which is delivered on when the message count received matches those sent. This does
  not tell us anything about the order yet, but it is still very useful when waiting for all messages to be
  delivered. In the subsequent assertion, we then check if the received messages are complete and in the
  expected order."
  (let [msgs-recvd (atom [])
        cnt 1000
        msgs-to-send (vec (range cnt))
        all-recvd (promise-chan)
        cmp (component/make-component {:all-msgs-handler (fn [{:keys [msg-payload]}]
                                                           (swap! msgs-recvd conj msg-payload)
                                                           (when (= cnt (count @msgs-recvd))
                                                             (put! all-recvd true)))})]

    (component/send-msgs cmp (map (fn [m] [:some/type m]) msgs-to-send))

    (tp/w-timeout 5000 (go
                         (testing "all messages received"
                           (is (true? (<! all-recvd))))
                         (testing "sent messages equal received messages"
                           (is (= msgs-to-send @msgs-recvd)))))))
~~~

As you can see above, there's the **promise-chan** `all-recvd`, onto which we `put!` a message (`true` in this case) when done. Then, in the `go` block inside the call to `tp/w-timeout`, we can wait for the promise-chan to be delivered on first, before proceding with other assertions that only make sense when the promise is delivered.

This `w-timeout` function implements behavior differently, depending on the target platform. Let's have a look at the whole **[namespace](https://github.com/matthiasn/systems-toolbox/blob/9286c070f8be8684cf69676adc9bbaa393832201/test/matthiasn/systems_toolbox/test_promise.cljc)**. However, this is optional, it requires some understanding of `go` blocks and channels, which the systems-toolbox tries to hide from you. So feel free to skip the next code block and only use this promise-like behavior as a recipe, if you so desire.

~~~
(ns matthiasn.systems-toolbox.test-promise

  "Provide a promise-like experience for testing."

  #?(:cljs (:require-macros [cljs.core.async.macros :refer [go]]))
  (:require
   #?(:clj  [clojure.test :refer [is]]
      :cljs [cljs.test :refer-macros [async is]])
   #?(:clj  [clojure.core.async :refer [go alts! <!! timeout]]
      :cljs [cljs.core.async :refer [alts! take! timeout]])))

(defn test-async
  "Asynchronous test awaiting ch to produce a value or close. Makes use of cljs.test's facility
  for async testing.
  Borrowed from http://stackoverflow.com/questions/30766215/how-do-i-unit-test-clojure-core-async-go-macros"
  [ch]
  #?(:clj (<!! ch)
     :cljs (async done (take! ch (fn [_] (done))))))

(defn test-within
  "Asserts that ch does not close or produce a value within ms. Returns a channel from which the value
  can be taken. Also borrowed from stackoverflow comment above."
  [ms ch]
  (go (let [t (timeout ms)
            [v ch] (alts! [ch t])]
        (is (not= ch t)
            (str "Test should have finished within " ms "ms."))
        v)))

(defn w-timeout
  "Combines tes-async and test-within to provide the deref functionality we expect from a promise.
  The first argument is the timeout in milliseconds, the second argument should be a go-block (which
  returns a channel with the return value of the block once completed). Then, in that go block, we can
  await the promise-chan to be delivered first before making any further assertions."
  [ms ch]
  (test-async
    (test-within ms ch)))
~~~

First, the `test-async` function implements the wait differently. On the JVM, we have the blocking `<!!` which simply blocks until there's value on the channel. On the ClojureScript side, we can make use of `async` to achieve the same thing. Note that the channel here, when composed in `w-timeout`, is the `go` block inside `test-within`. As mentioned, `go` blocks return a channel, onto which their return value will be put on completion.

Then, inside `test-within`, `alts!` is used, which will return either the value on the promise-chan or the timeout, whatever happens first. Then, there is an assertion that the channel which returned first was not the timeout. This gives us a nice way to wait for as long as necessary, up to the timeout, with having to use dumb waiters like `Thread/sleep` that always wait for the entirety of its duration and thus hold up test runs. Also, `Thread/sleep` does not work in the browser, while this mechanism presented here does.

In `w-timeout`, these two functions are then combined into one that takes both the timeout and a go-block, in which we should wait for the promise-chan.


### Running tests in the browser / PhantomJS



### Performance considerations

The other day, I wanted to know how many messages the systems-toolbox could process per second, for a single component. So I wrote an initial test for that and got like 70K messages per second. Now, this is probably not terrible, especially in the browser where I currently cannot think of any application that would need anything near this number. Here's the **[test](https://github.com/matthiasn/systems-toolbox/blob/cbeeb951d34f65da60e8772f194c7e609b71eae1/test/matthiasn/systems_toolbox/component_test.cljc#L40)**:

~~~

(defn cmp-all-msgs-handler-cmp-state-fn
  []
  "Like cmp-all-msgs-handler test, except that the handler function here acts on the component state provided
  in the map that the :all-msgs-handler function is called with. [...]"
  (let [cnt (* 100 1000)
        state (atom 0)
        vals-to-send (vec (range cnt))
        msgs-to-send (map (fn [m] [:some/type m]) vals-to-send)
        all-recvd (promise-chan)
        res (reduce + (range cnt))
        cmp (component/make-component {:state-fn         (fn [_put-fn] {:state state})
                                       :all-msgs-handler (fn [{:keys [msg-payload cmp-state]}]
                                                           (let [new-state (+ @cmp-state msg-payload)]
                                                             (reset! state new-state)
                                                             (when (= res new-state)
                                                               (put! all-recvd true))))})
        start-ts (component/now)]

    (component/send-msgs cmp msgs-to-send)

    (tp/w-timeout cnt (go
                        (testing "all messages received"
                          (is (true? (<! all-recvd))))
                        (testing "processes more than 1K messages per second"
                          (let [msgs-per-sec (int (* (/ 1000 (- (component/now) start-ts)) cnt))]
                            (log/debug "Msgs/s:" msgs-per-sec)
                            (is (> msgs-per-sec 1000))))
                        (testing "sent messages equal received messages"
                          (is (= res @state)))))))

(deftest cmp-all-msgs-handler-cmp-state1
  (cmp-all-msgs-handler-cmp-state-fn))

(deftest cmp-all-msgs-handler-cmp-state2
  (cmp-all-msgs-handler-cmp-state-fn))

(deftest cmp-all-msgs-handler-cmp-state3
  (cmp-all-msgs-handler-cmp-state-fn))

(deftest cmp-all-msgs-handler-cmp-state4
  (cmp-all-msgs-handler-cmp-state-fn))

(deftest cmp-all-msgs-handler-cmp-state5
  (cmp-all-msgs-handler-cmp-state-fn))

(deftest cmp-all-msgs-handler-cmp-state6
  (cmp-all-msgs-handler-cmp-state-fn))
~~~

What happens here is that I take an atom with the number zero in it, and the range from zero to 100,000 (exclusive) and send each number in that range to the component `cmp`, which adds each to its component state. Then, in the assertions section, I calculate how many messages per second the processing time corresponds to, print that value for reference, assert that it's large enough, and finally check that all numbers were processed by comparing the number in the component state with the result of `(reduce + (range cnt)`. I also make an assertion about the number of messages per second being high enough here, but that's probably not terribly useful as this number has to be pretty conservative anyway to take into account less powerful nodes, such as the ones used by TravisCI or CircleCI. Also note that the test code is in a function that I then call multiple times to see how much of an effect JIT compilation has on the result. Apparently, some optimizations do kick in on subsequent runs:

~~~
lein test matthiasn.systems-toolbox.component-test
DEBUG m.systems-toolbox.component-test - Msgs/s: 62814
DEBUG m.systems-toolbox.component-test - Msgs/s: 71479
DEBUG m.systems-toolbox.component-test - Msgs/s: 91575
DEBUG m.systems-toolbox.component-test - Msgs/s: 88731
DEBUG m.systems-toolbox.component-test - Msgs/s: 78616
DEBUG m.systems-toolbox.component-test - Msgs/s: 89928
~~~

Okay, roughly 60K messages per second on the first run and then towards 90K messages per second on subsequent runs. Doesn't sound that terrible.

Then on the next day, I went for breakfast with my friend Peter. I told him what I was working on and about the numbers I achieved. Then he mentioned that he was working on an implementation of the **[actor model](https://en.wikipedia.org/wiki/Actor_model)** in **[Rust](https://www.rust-lang.org/)** and that there, his latest benchmark gave him numbers well north of 2 million messages per second, without much optimization effort yet. That was a bit of a downer. Sure, I wouldn't be too bothered to learn that Rust was faster than Clojure, that's kind of to be expected, but well above an order of magnitude for the simple task of delivering a message to some entity is more than I'd be willing to accept, especially since those numbers for his Rust project were pre-optimization.

That made me wonder where time was spent in my library, so I set out to check what the JVM is capable of. I started with looking at atoms and how often I can reset or swap them per second. Here's the test for resetting an atom:

~~~
(defn reset-atom-repeatedly-fn
  []
  "This test aims at getting some perspective how expensive resetting an atom is in Clojure/ClojureScript.
  Answer: not terribly expensive. On the JVM, this can be performed around 90 million times per second,
  whereas in ClojureScript, this can be done 15 million times per second on PhantomJS and over 60 million
  times per second in Firefox (2015 Retina MacBook)."
  (let [start-ts (component/now)
        cnt (* 1000 1000)
        state (atom 0)]
    (dotimes [n cnt] (reset! state n))
    (let [ops-per-sec (int (* (/ 1000 (- (component/now) start-ts)) cnt))]
      (log/debug "Atom resets/s:" ops-per-sec)
      (is (> ops-per-sec 1000)))))

(deftest reset-atom-repeatedly
  (dotimes [_ test-runs]
    (reset-atom-repeatedly-fn)))
~~~

Okay, so even when I run this test alone in a cold JVM with `$ lein test :only matthiasn.systems-toolbox.runtime-perf-test/reset-atom-repeatedly` and only a single run, I get anywhere between 29 and 43 **million** ops/sec. When I set `test-runs` to 10, I even get up to **90 million** ops/sec on the JVM. On phantom, there's no noticeable effect of JIT on subsequent runs, by the way. But there I also don't know how to isolate test runs, so likely the JIT optimizations will already have kicked in by the time the tests run. Still, I get a solid 15 million ops/sec in phantom at the time of writing. On Chrome, I got 38 million ops/sec and on Firefox, I got a whopping **66 million** ops/sec. Interesting, last time I checked, Chrome was faster than any other browser, but that does not seem to be the case any longer. Anyway, in either case, resetting an atom is quite obviously not a bottleneck on any of those platforms.

Hmm, maybe the atom watching, which leads to publishing a new state snapshot when a change is detected, could be the culprit? Let's check:

~~~
(defn swap-watched-atom-repeatedly-fn
  []
  "This test aims at getting some perspective how expensive swapping an atom is in Clojure/ClojureScript.
  Answer: not terribly expensive. On the JVM, this can be performed around 70 million times per second,
  whereas in ClojureScript, this can be done 15 million times per second (2015 Retina MacBook)."
  (let [start-ts (component/now)
        cnt (* 1000 1000)
        state (atom 0)]
    (add-watch state :watcher (fn [_ _ _ _new-state] #()))
    (dotimes [_ cnt] (swap! state inc))
    (let [ops-per-sec (int (* (/ 1000 (- (component/now) start-ts)) cnt))]
      (log/debug "Watched atom swaps/s:" ops-per-sec)
      (is (> ops-per-sec 1000)))))

(deftest swap-watched-atom-repeatedly
  (dotimes [_ test-runs]
    (swap-watched-atom-repeatedly-fn)))
~~~

Nope. Almost 70 million ops/sec on Firefox and 40 million ops/sec on the JVM (on repeated runs) suggest differently. Hmm, could it be core.async? Let's see:

~~~
(defn put-on-chan-repeatedly-fn
  "Channel with attached mult and no other channels tapping into mult: messages silently dropped."
  []
  (let [start-ts (component/now)
        cnt (* 100 1000)
        ch (chan)
        m (mult ch)
        done (promise-chan)]
    (go
      (dotimes [n cnt] (>! ch n))
      (put! done true))

    (tp/w-timeout cnt (go
                        (testing "all messages received"
                          (is (true? (<! done))))
                        (let [ops-per-sec (int (* (/ 1000 (- (component/now) start-ts)) cnt))]
                          (log/debug "Channel puts/s:" ops-per-sec)
                          (is (> ops-per-sec 1000)))))))

(deftest put-on-chan-repeatedly1
  (put-on-chan-repeatedly-fn))
(deftest put-on-chan-repeatedly2
  (put-on-chan-repeatedly-fn))
(deftest put-on-chan-repeatedly3
  (put-on-chan-repeatedly-fn))
(deftest put-on-chan-repeatedly4
  (put-on-chan-repeatedly-fn))
(deftest put-on-chan-repeatedly5
  (put-on-chan-repeatedly-fn))
(deftest put-on-chan-repeatedly6
  (put-on-chan-repeatedly-fn))
~~~

Okay, around 1 million ops/sec on Firefox and a little under 250K ops/sec on the JVM (on repeated runs). Now this is substantially slower than the atom operations. We might be onto something here, since the library does multiple core.async operations for each message. Let's emulate the (basic) behavior of the library using core.async directly:

~~~

(defn put-consume-mult-w-pub-repeatedly-fn
  "Channel with attached go-loop, simple calculation using messages from channel, publication of state change. This
  imitates the basic use case of the systems-toolbox: there's a go-loop, some processing and publication of component
  state. Running this test gives some perspective of the amount of overhead that the systems-toolbox introduces,
  such as adding metadata to messages."
  []
  (let [start-ts (component/now)
        cnt (* 100 1000)
        ch (chan (buffer 1))
        m (mult ch)
        ch2 (chan)
        state-pub-chan (chan (sliding-buffer 1))
        state-mult (mult state-pub-chan)
        state (atom 0)
        done (promise-chan)]

    (go-loop []
      (let [n (<! ch2)
            res (+ @state n)]
        (reset! state res)
        (>! state-pub-chan res)
        (when (= (dec cnt) n)
          (put! done true)))
      (recur))

    (tap m ch2)
    (go (dotimes [n cnt] (>! ch n)))

    (tp/w-timeout cnt (go
                        (testing "promise delivered"
                          (is (true? (<! done))))
                        (let [ops-per-sec (int (* (/ 1000 (- (component/now) start-ts)) cnt))]
                          (log/debug "Channel puts and consume from mult/s (w/pub):" ops-per-sec)
                          (is (> ops-per-sec 1000)))
                        (testing "all messages received (sum of all number sent matches)"
                          (is (= @state (reduce + (range cnt)))))
                        :done))))

(deftest put-consume-mult-w-pub-repeatedly
  (put-consume-mult-w-pub-repeatedly-fn))
(deftest put-consume-mult-w-pub-repeatedly2
  (put-consume-mult-w-pub-repeatedly-fn))
(deftest put-consume-mult-w-pub-repeatedly3
  (put-consume-mult-w-pub-repeatedly-fn))
(deftest put-consume-mult-w-pub-repeatedly4
  (put-consume-mult-w-pub-repeatedly-fn))
(deftest put-consume-mult-w-pub-repeatedly5
  (put-consume-mult-w-pub-repeatedly-fn))
(deftest put-consume-mult-w-pub-repeatedly6
  (put-consume-mult-w-pub-repeatedly-fn))
~~~

Here, we do roughly the same a component in the systems-toolbox does, which receives messages on a channel, process them in a `go-loop` (which is hidden from the user in the case of the systems-toolbox), and publish state changes onto the `state-pub-chan`. Et voilà, the results are pretty much the same as we see when processing messages with the systems-toolbox, with around 90K msgs/sec on the JVM.

Okay, so I'm fully aware of my tendency to shave yaks when it comes to looking at performance. However, I think this little excursion was useful, at least for me, as it provides some context where time is spent and where future optimizations could go. For example, you've probably heard that atoms are slower than their `volatile!` counterpart. However, when looking at the data that surfaced here, interacting with atoms is not where substantial amounts time are wasted. Thus, looking at replacing atoms with `volatile!` is likely not going to help much. If anything, it might be worth looking into core.async, which does seem to add a considerable amount of overhead. Considering that we are talking about in-process conveyance here and that Kafka is capable of handling **[millions of messages](https://engineering.linkedin.com/kafka/benchmarking-apache-kafka-2-million-writes-second-three-cheap-machines)** a second, the numbers here are a little lame, especially since in the case of Kafka, this involves round trips to the filesystem and network. But then again, this is not a real problem until it is. So far, the applications I have written with the systems-toolbox have not hit a brick wall when it comes to performance. 90K msgs/sec is still plenty and probably more than you could expect to get from REST-based microservices.

However, the results confirm my hunch that it's probably a good idea to make the message conveyance in the systems-toolbox pluggable, and then offer a choice to handle it with either **core.async** or **Kafka** on the JVM, as that would offer attractive properties for observability out of the box. But that's a different story and not within the scope of this chapter. In the browser, I cannot think of a use case right now where tens of thousands of messages per second would not suffice.
