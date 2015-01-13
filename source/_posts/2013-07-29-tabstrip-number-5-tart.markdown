---
layout: post
title: "Tabstrip #5 - TART, talos stress, smooth scrolling"
date: 2013-07-29 15:14
comments: true
categories: TART smooth scroll talos svgc scrollx
---

###Talos stress
Talos `tsvg` and `tscroll` are about to be replaced with `tsvgx` and `tscrollx`, respectively ([bug 897054](https://bugzilla.mozilla.org/show_bug.cgi?id=897054)). The main difference is that the "x" tests stress Firefox by iterating animations as fast as possible, AKA "ASAP mode".

The old tests were performing some animation and then report overall (or average per frame) duration. However, they were using intervals which were not stressing Firefox at all, making the results almost meaningless WRT svg/scroll performance, and rather mostly sensitive to timing changes.

Stressing Firefox exposed some issues such as paint starvation ([bug 880036](https://bugzilla.mozilla.org/show_bug.cgi?id=880036)).

Hopefully the new tests will have better correlation to our performance. Joel Maher did (and still does) a lot of work on the Talos side of things with these new tests (a  l-o-t!)

There's a more technical explanation on this [dev.platform post](https://groups.google.com/forum/#!topic/mozilla.dev.platform/RICw5SJhNMo).

###TART - Tab Animation Regression Test

After previous work which went into improving tab animations in Firefox, It's time to put it into Talos. TART is implemented as an addon which works either stand alone or from within Talos. It measures frames intervals during different animation cases, and it works equally well on mozilla-central and on the UX branch.

TART uses ASAP mode to iterate animation with unlimited frame rate, thus able to expose differences even if under normal conditions it would have been limited to 60Hz.

Joel Maher did most of the talos side of things, and currently the addon awaits review.

- TART tests animation on 3 main cases:
  - "simple" - open/close of about:blank.
  - "icon" - open/close of a blank html page with favicon and a long title.
  - "newtab" - open with and without newtab preload.

"simple" is tested at DPI scaling of 1.0, "icon" at scaling of 1.0 and 2.0, and "newtab" at default scaling.

- Each animation produces 3 result values:
  - "all" - Overall average interval.
  - "half" - Average interval over 50% of the designated duration - from the end of the animation backwards.
  - "error" - The difference between the designated duration to the actual one.

Typically, "half" is the value which corresponds to the stable part of the animation once it's hopefully settled. Most tab animations have first frame (or few) which are longer than the rest, so the "all" value, while not meaningless, doesn't tell the whole story.

- Some issues which had to be resolved while working on TART:
  - Inactive tabs are throttled, so to get accurate timing (intervals, end event), the test is executed from the Chrome window (rather than from the tab itself).
  - At DPI scaling of 2.0 at the talos window size, Australis can't open a second tab without shrinking the first (thus measures animation of more than one tab). Solution: keep the TART tab pinned.
  - On OS X, "ASAP mode" doesn't work with OMTC. Solution for now: disable OMTC for TART.
  - On ASAP mode, mostly on OSX, paints can be starved ([bug 880036](https://bugzilla.mozilla.org/show_bug.cgi?id=880036)). Solution: use a temporary pref to prevent this ([bug 884955](https://bugzilla.mozilla.org/show_bug.cgi?id=884955)).
  - At DPI scaling of 2.0, Australis can only have 3 tabs before it starts overflowing the tabstrip (e.g. [bug 891450](https://bugzilla.mozilla.org/show_bug.cgi?id=891450)). This made it very hard to test multi-tab shrink/expand in a comparable fashion on m-c and Australis, and eventually this test was dropped for TART v1.

You can watch TART at [bug 848358](https://bugzilla.mozilla.org/show_bug.cgi?id=848358). Other than a review, it also awaits [bug 888899](https://bugzilla.mozilla.org/show_bug.cgi?id=888899) which should enable "ASAP" (stress) mode on OS X as well.


### Absolute scroll smoothness on windows

I was asked by [Taras](http://taras.glek.net/) to look into scroll smoothness on windows. Apparently, even after vlad's great [timing improvement](https://bugzilla.mozilla.org/show_bug.cgi?id=731974), the timers [filter removal](https://bugzilla.mozilla.org/show_bug.cgi?id=590422) and making vsync on windows [work](https://bugzilla.mozilla.org/show_bug.cgi?id=856427), Firefox still almost never scrolls 100% smoothly on windows.

I did some extensive research into scroll performance of Firefox and other browsers on various platforms, and posted the results at [bug 894128](https://bugzilla.mozilla.org/show_bug.cgi?id=894128). You'll also find there a a bookmarklet to test scroll performance on any browser.

One interesting observation is that it's very hard to programatically detect if the scroll is absolutely smooth. Frame intervals apparently don't tell the whole story (though they are not meaningless), and neither their standard deviation. To judge if the scroll is 100% smooth, it has to be assessed visually.

- Some interesting observations from this research:
  - On OS X, even with 50% perf per core (and half the cores as well) compared to a fast Windows system, Firefox scrolls much smoother than on Windows, even with OMTC off.
  - Firefox may occasionally degrade scroll for several frames for no apparent reason (the content is homogenous). GC/CC?
  - OMTC typically doubles the throughput on OS X. On windows OMTC is not yet mature enough to test this.
  - Very low intervals variance is _not_ required for 100% smooth scroll (though IE and Chrome mostly manage very low variance compared to Firefox).
  - Looks like Chrome and IE use something like APZC during scroll. IE is especially exceptional and is able to practically never drop a frame even on extremely low-end HW (Atom z2760), and also almost never show blanks during scroll.
  
Jet has already shown interest in looking into it. Hopefully some day Firefox will scroll as smooth as it can get...