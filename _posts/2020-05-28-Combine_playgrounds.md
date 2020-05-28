---
layout: post
title:  "Combine playgrounds"
subhead: "Learning combine with a hands on experience"
date:   2020-05-28 9:35:32 +0100
categories: ["combine"]
---

At WWDC 2019 Apple introduced *Combine*, a new framework for reactive programming. This programming paradigm is becomming more and more popular over the recent years, and it's clear that Apple intends to move forward with it.

Reactive programming is a declarative programming paradigm concerned with data streams and the propagation of change. In *Combine* these streams process values and finish with a completion or an error.

It is difficult to change our mindset from imperative coding, where programs describe how the data is treated, to declarative coding, where programs describe the chain of logic operator which data is going to flow through.
I figured playing around with these concepts in a playground makes it easier to understand. If you are a beginner with *Combine* I suggest you take a look at the playground I put together on [my github](https://github.com/denisPoifol/CombinePlaygrounds/tree/CombineInDepth).
It will give you the opportunity to tip toe around the framework and slowly grasp the concepts of reactive programming with *Combine*. Instead of going through the mind shift of reactive programming on your own, while discovering an unknown framework. And let's face it, 2019 was not the year of well documented frameworks at Apple.

If you are more advanced with *Combine* there are still some interesting things to dig out from this playground. Most of the time people want to create their custom publishers, but in order to do it properly there are a few concepts you need to understand that are often overlooked.

The first one is memory management. It is always important to understand how the memory is handled. Especially if you dont want to find yourself with memory leaks all over the place. Granted *Combine* works in mysterious ways regarding memory management, and it seems like everything sorts itself out magically in the end. But this is because the creation of a publisher or a subscriber needs to follow certain rules, and in the end it all comes down to knowing how to conform to `Cancellable`.

The second one is back pressure. Back pressure is the idea that a `Subscriber` is the one in control of the data flow in the stream. It will receive values from its publisher only if it requested it. The idea is pretty simple by itself, but it actually means a publisher needs to keep track of the `Demand` from its subscribers.
But its not the only danger with back pressure. If we consider a `TimerPublisher` that will emit a `Date` every second. And its `Subscriber` that only request a value once every two seconds, it means our publisher will want to publish values when its subscriber did not requeste any. The way the publisher handles this case (if it conforms to the *Combine* standard) is to drop values and pretend it never occured. But droping values is not always ideal, and in order to avoid it, *Combine* provides back pressure operators (such as `Collect`). These operators do not actually apply back pressure, instead they define strategies to salvage dropped values.

Unfortunatelly these concepts are way to complex to explore in a single blog post so you might have to take a look at [the playground](https://github.com/denisPoifol/CombinePlaygrounds/tree/CombineInDepth), and dive into the depths of *Combine*.
