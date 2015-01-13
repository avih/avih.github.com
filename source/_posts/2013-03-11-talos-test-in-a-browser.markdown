---
layout: post
title: "Talos - test in a browser, noise detection"
date: 2013-03-11 11:31
comments: true
categories: Mozilla, talos
---

[Talos](https://wiki.mozilla.org/Buildbot/Talos) is a performance tests framework used by Mozilla. It's invoked on every checkin to the main code repository for early detection of performance regressions. You can find many interesting talos notes on [Joel Maher's blog](https://elvis314.wordpress.com/).

Talos tests are pretty simple - they perform some task within a browser, then report a completion duration (or a comma-separated list of those) via the `tpRecordTime` javascript talos call. Talos then logs and processes those values, tries to ignore the noisy bits, and comes up with an average and StdDev values (it also deploys a similar process over several runs of the same test) - which can then be compared to other runs of the same test on different versions/platforms of Firefox.

###Run and develop talos tests - without talos

While [updating tscroll](https://bugzilla.mozilla.org/show_bug.cgi?id=845943) talos test, I found that running the test in a browser outside of the talos framework could be useful. The page could be reloaded to re-run the test, allowing quicker iterations of the code. While this isn't as sterile as the talos run environment, it's still useful during development. This required substituting `tpRecordTime` with my own, and also displaying various statistics on the collected data.

I'm not the not the only one who deployed such tactics, and one can find commented out `alert`s and statistics calculation (which are not reported to talos) on various talos tests.
<!-- more -->

So we had the idea of separating this debug stuff from tscroll, such that one can include this `talos-debug.js` file from any test, and then simply load the page in a browser and get a report of the recorded values together with some statistics.

This file adds a `window.talosDebug` object which provides some statistical functions (`sum`, `average`, `median` and `stddev`) and a `tpRecordTime` method which also displays the stats. It can be controlled by few properties (such as `displayData` - if set to true, will also display the raw collected data). It also provides automatic detection of the "warmup period" for the data set (see next).

If a global tpRecordTime doesn't exist, then it maps it to window.talosDebug.tpRecordTime. It's safe to include this file in any talos test, which will allow running the test in a browser, while it shouldn't affect the results when running within the talos framework.

[Bug 849558](https://bugzilla.mozilla.org/show_bug.cgi?id=849558) is about adding `talos-debug.js` to the talos repository. If you touch the talos tests, consider using it instead of rolling out your own. Suggestions are welcome.

###Automated tests and noise removal

The talos tscroll test records average frame interval of different real-world-like content types scroll. It showed some regression when I tried to remove a timer compensation mechanism which was actually [hurting timimg](https://bugzilla.mozilla.org/show_bug.cgi?id=590422#c21) on most cases these days. When I looked at the tscroll source code, I thought that this isn't a real regression, and instead, we might be able to improve tscroll.

While modifying tscroll to be less susceptible to timing changes (and also report all the raw intervals instead of the average only), I noticed that typically the first few frame intervals during scroll are more noisy than the rest.

While this isn't surprising - the browser, layers, etc need to build-up and settle-down, hopefully quickly - it did bring up the question of how to handle this.

Since this is obviously a transitional effect, we probably want to exclude this warm-up period when measuring scroll *smoothness*. I manually estimated the number of first intervals we should ignore by observing the values. It appeared that ignoring the first 5 values would do the trick for most of the tsctoll sub-tests, and that's what we agreed to do.

For fun, I also wrote some auto-warmup-detection code, and incorporated it into [talos-debug.js](https://bugzilla.mozilla.org/show_bug.cgi?id=849558). It actually works surprisingly consistently and accurately for the noise patterns (is that an oxymoron??) of scroll intervals, coming up, for instance, with results such as this (notice how stddev goes from 10.8% over all the values, down to 2.8% after ignoring the first 2 out of 42 intervals):

    Count: 44
    Average: 16.38
    Median: 16.88
    StdDev: 1.77 (10.83% of average)

    Warmup auto-detected: 2
    After ignoring first 2 items:
    Count: 42
    Average: 16.70
    Median: 16.89
    StdDev: 0.47 (2.79% of average)

    Recorded:
    5.68, 13.59, [warmed-up:], 17.26, 16.93, 15.95, 17.05, 16.98, 16.08, 16.94, 16.91, 16.39, 16.72, 17.03, 16.03, 16.96, 17.72, 16.03, 17.04, 17.07, 15.92, 16.99, 17.09, 15.94, 16.99, 16.99, 16.01, 16.99, 17.08, 15.96, 16.84, 17.21, 16.11, 16.86, 16.92, 16.21, 16.86, 16.88, 16.19, 16.88, 16.97, 16.21, 16.74, 17.11, 16.21

    Stddev from item NN to last:
    1.77, 0.66, [warmed-up:], 0.47, 0.46, 0.47, 0.46, 0.46, 0.46, 0.46, 0.46, 0.47, 0.47, 0.48, 0.49, 0.48, 0.49, 0.45, 0.44, 0.44, 0.45, 0.43, 0.43, 0.43, 0.41, 0.42, 0.42, 0.40, 0.41, 0.41, 0.38, 0.39, 0.37, 0.35, 0.37, 0.37, 0.37, 0.38, 0.40, 0.39, 0.42, 0.44, 0.45, 0.64, 0.00


Looking at this issue with a wider perspective, the subject of warmup period (and noise in general) is common to automated testing. It definitely affects stability of the results, and mostly hurts StdDev - the results appear noisier than they actually are after everything settles down.

Talos uses a [similar strategy](https://elvis314.wordpress.com/2012/03/12/reducing-the-noise-in-talos/) (warning - year old post). It runs each test few times, then removes the noisiest result, which is typically the first one.

However, as was noted at some of the comments on that post, the warmup period does carry its own weight and probably shouldn't be ignored completely.

Maybe if we deploy an automatic warmup detection system similar to the one described above (after it's tuned and tested with a much wider types of results than just the frame intervals during scroll), we could have a double win - reduce the noise level of the results, AND record the warmup period required for each test. We could then watch this value over time, and if - for a specific test - the warmup period goes up, then it's probably a regression, and vice verse.




