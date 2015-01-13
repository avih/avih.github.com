---
layout: post
title: "Tabstrip animation Project #1 - Introduction"
date: 2013-04-14 18:37
comments: true
categories: tabstrip animation vsync
---

I've had few slow weeks during which I attended some personal matters. Hopefully I'm back in full steam ahead again.

Here's a summary of recent events.

Tabstrip animation project
--------------------------

As part of the recent Performance team changes ([1](http://taras.glek.net/blog/2013/03/27/snappy-number-55-snappy-evolution/), [2](http://lawrencemandel.com/2013/03/21/no-more-snappy-meetings-and-other-changes-from-the-snappy-team/)), I'm now the tech lead for the tabstrip animation project. It'll require some additional coordinations and regular blog posts, but otherwise, work continues as usual.

The goal of the tabstrip animation project:

*Make tabstrip animations as smooth and as snappy as possible, both visually and perceptually, by identifying and removing bottlenecks, deficiencies, and perception issues. The ultimate goal here is 60 FPS on a recent Atom CPU on all platforms, with minimal delays*.

So far, this work touched several different subjects:
<!--more-->

- [Infrastructure](https://bugzilla.mozilla.org/show_bug.cgi?id=826383) and [telemetry](https://bugzilla.mozilla.org/show_bug.cgi?id=828097) for tabstrip animation smoothness measurements. This infrastructure already proved very useful with the various subjects below. In general, good measurements allow much more concrete goals, feedback and regression detections. We should strive to use measurements wherever possible, while making sure the measurements are a good representation for the kind of performance which interests us.

- *Timing related improvements*. Timing is the essence of animation. It's important for tabstrip animations and for Firefox in general to be able to present each frame at exactly the right time, or else the animation won't be as smooth as it could be. This is an ongoing effort within Mozilla.

Following Vladimir's very useful [timers change](https://bugzilla.mozilla.org/show_bug.cgi?id=731974) which recently arrived to the release channel, there were few more issues remaining, mainly [timers averaging delay removal](https://bugzilla.mozilla.org/show_bug.cgi?id=590422#c21) which was a long standing bug which also affected tab animation and finally resolved recently, [vsync](https://bugzilla.mozilla.org/show_bug.cgi?id=689418) synchronization (present frames in sync with the monitor refresh rate) which greatly affects perception of smoothness and is a sore issue since Firefox is now lagging behind Chrome and IE on this (preliminary [Windows vsync build](https://bugzilla.mozilla.org/show_bug.cgi?id=856427#c5)/patch), and finally, [OMTC](https://wiki.mozilla.org/Platform/GFX/OffMainThreadCompositing) which aims at offloading some rendering/animation related tasks off the main thread, and which needs to integrate well with other timing related work.

- *Identifying and removal of relevant graphic bottlenecks*. Tabstrip animation (and animation in general) is limited by rendering speed. Any improvements here translate to possibly faster and smoother animation, especially on slow computers. We've already improved D2D accelerated tab animation by about 25% by [improving the gradients cache system](https://bugzilla.mozilla.org/show_bug.cgi?id=838758), and we still have similar potential improvements with [SVG raster caching](https://bugzilla.mozilla.org/show_bug.cgi?id=764299), and possibly other graphic bottlenecks.

- *Theme optimizations*. During the development of [Australis](https://bugzilla.mozilla.org/show_bug.cgi?id=732583) theme, we've noticed that its tabs animate slower than the current theme. This was followed up by great [ongoing](https://bugzilla.mozilla.org/show_bug.cgi?id=837885) effort and attention from Mike Conley ([good post](http://mikeconley.ca/blog/2013/03/01/australis-curvy-tabs-more-progress/)) and Matthew Noorenberghe to measure and improve tab animation speed with Australis. One of the results of this effort was replacing SVG usage with images, but we'd still like to get back to SVG, as it's more elegant and it scales better for different sizes/DPI.

- *Browser issues* which affect tab animation. Even if the timing is great, there are no major GFX bottlenecks, and the theme is light to render, tab animation is still affected by implementations within the browser. E.g. the way it's triggered ([sync or async](https://bugzilla.mozilla.org/show_bug.cgi?id=850163), delayed, etc), pages which render concurrently with animation and compete with it on resources (e.g. the [new tab](https://bugzilla.mozilla.org/show_bug.cgi?id=843853) page) and possibly other similar issues. Tim Taubert from the Firefox team has been working on these issues.

- *Regression detection*. We've already identified an [unoptimal talos scroll test](https://bugzilla.mozilla.org/show_bug.cgi?id=845943) during the timer filter removal process, and posted an improvement. We should also add a similar [tab animation regression test](https://bugzilla.mozilla.org/show_bug.cgi?id=848358). In general, it's important that talos tests are useful, accurate, and are good representation of actual regressions which we care about. [Nathan](https://blog.mozilla.org/nfroyd/2013/03/25/perf-workweek-talos-discussion/) is the performance team representative for talos issues, and I'll help as much as I can.


I'll be posting regular progress reports on this subject.
