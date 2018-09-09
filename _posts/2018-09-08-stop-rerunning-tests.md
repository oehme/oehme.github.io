---
layout: post
title: "Stop rerunning your tests"
image: images/me.png
---

Tests are usually the longest running operation in your development process.
Running them unnecessarily is the ultimate time waster.
Gradle helps you avoid this cost with its [build cache](https://docs.gradle.org/current/userguide/build_cache.html) 
and [incremental build](https://docs.gradle.org/current/userguide/more_about_tasks.html#sec:up_to_date_checks) features.
It knows when any of your test inputs, like your code, your dependencies
or system properties, have changed. If everything stays the same, Gradle will
skip the test run, saving you a lot of time.

So you can imagine my desparation when I see snippets like this on StackOverflow:

```(groovy)
tasks.withType(Test) {
    outputs.upToDateWhen { false }
}
```

Let's talk about what this means and why it is **always** wrong.

## This doesn't actually force reruns

What the author of this snippet probably wanted to say is "Always rerun my tests".
That's not what this snippet does though. 
It will only mark the task out-of-date, forcing Gradle to *recreate* the output.
But here's the thing, if the build cache is enabled, Gradle doesn't need to run the task to recreate the output. 
It will find an entry in the cache and unpack the result into the test's output directory.

The same is true for this snippet:

```(groovy)
test.dependsOn cleanTest
```

Gradle will unpack the test results from the build cache after the output has been cleaned, so nothing will be rerun.
In short, these snippets are creating a very expensive no-op.

If you're now thinking "Okay, I'll deactivate the cache too", let me tell you why you shouldn't.

## Forcing reruns is insane

> "Insanity is doing the same thing over and over and expecting different results"
>  
> \- Not Albert Einstein

The vast majority of your tests should be deterministic, i.e. given the same inputs they should produce the same result.
If this is not the case, your project is in serious trouble. Stop reading this post and start fixing your code!

Rerunning deterministic tests is a waste of your team's time.

## Non-deterministic tests

There are a few reasons why you might want to rerun *some* tests in *some* cases even though none of the code changed.
In these cases, you should model the additional input properly. 
Tell Gradle what exactly makes your tests non-deterministic.

### Randomized tests

Some tests use randomization to improve the quality of your software.

- [Random testing](https://en.wikipedia.org/wiki/Random_testing) can be used to ensure that the production code can handle all kinds of inputs,
not just the ones that the developer came up with.
- [Mutation testing](https://en.wikipedia.org/wiki/Mutation_testing) changes the production code in subtle way (e.g. introducing off-by-one errors)
and checks whether your test suite catches these mistakes.

Make this randomization explicit by making the random seed an input to your task:

```(groovy)
task randomizedTest(type: Test) {
    systemProperty "random.testing.seed", new Random().nextInt()
}
```

This will force Gradle to always rerun that test, because it will always have a different seed.
Even better, you could make the seed user-configurable to locally reproduce bugs that were found on your build server.

### System integration tests

System integration tests verify your application against realistic versions of other applications that are controlled by other teams.
They may fail even if you didn't change anything, e.g. because another team broke an API.
They also tend to be among the slowest tests in your code base, so you don't want to rerun them just because you changed some documentation.
A good compromise may be to check integration at least once a day, even if nothing on your side has changed.

Make this interval part of your test inputs:

```(groovy)
task systemIntegrationTest(type: Test) {
    inputs.property "integration.date", LocalDate.now()
}
```

You can then set up an automated build that runs this test in the morning before everyone starts working and let it push the result to a shared build cache.
When your team comes to work, the test result will be downloaded from the cache and the tests won't have to rerun for that day.
They can then use the saved time on more productive tasks like fixing bugs or developing new features.

### Flaky tests

Sometimes you encounter a bug that only makes a test fail 1 out of 10 times.
In order to analyze the situation, you want Gradle to rerun the test even if it was successful before.
In this case it is reasonable to use the random input approach I showed above.
However, I find it more productive to wrap the flaky test in an endless loop.
This way I can keep the debugger in the IDE running and even make small changes on the fly without having to restart.

## Model your tests properly

Properly modeling your requirements is just as important for your build logic as it is for your production code.
Doing it right will make your builds faster and your team more productive.

The `java` plugin's built-in [`test`](https://docs.gradle.org/current/userguide/java_plugin.html#sec:java_test) task is meant for deterministic and fast unit tests.
It may seem convenient to put all your tests into this one task and rerun them on every build, because a few of them are non-deterministic.
This lazy approach will save you a few minutes of thinking and coding, but the long term costs are huge, slowing everyone on your team down every day.

Instead, create additional [`Test`](https://docs.gradle.org/current/dsl/org.gradle.api.tasks.testing.Test.html) tasks for different types of tests like functional, performance, or randomized tests. For each one of them, consider when it needs to rerun and model that as an input to the task.

Don't waste your time.