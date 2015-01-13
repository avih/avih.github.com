---
layout: post
title: "Tabstrip animation progress #2: vsync, newtab page rendering, Lightweight theme"
date: 2013-04-15 18:23
comments: true
categories: 
---
Tabstrip animation - Progress #2
--------------------------------

The [previous post](/blog/2013/04/14/tabstrip-project-intro/) introduced the Tabstrip animation project, and some work which has been done so far. This post reviews some more recent related progress.

###Vsync
During the Paris [snappy work week](http://taras.glek.net/blog/2013/03/26/snappy-number-54-snappy-discussion-in-paris/), Bas and I discussed animation smoothness, and specifically the default 60hz refresh rate which Firefox currently uses. We've conducted a very unscientific poll among some attendees at the meeting, and found out that 4 out of 9 laptops had 50Hz monitor refresh rate. The rest had 60. For those with 60Hz, the current refresh system is acceptable, even if not perfect (actually  ... [Often, developers choose 60 Hz as the refresh rate, not knowing that the enumerated refresh rate from the monitor is approximately 60,000 / 1,001 Hz](http://msdn.microsoft.com/en-us/library/windows/desktop/ee417025%28v=vs.85%29.aspx) ...), but those with 50Hz monitors will always get a very noticeable jitter during any animation.

So Bas wrote a patch to expose some vblank timing info on windows via widget, and I updated the refresh driver to use this info. The result is decent vsync synchronization on monitors with all refresh rates (you can test the slightly outdated [Windows try build](https://bugzilla.mozilla.org/show_bug.cgi?id=856427#c5), which hasn't landed).

However, by using the vblank timing info and targeting a timer just past it, we're forced to "lose" some 2-3ms in order to always hit the correct vsync phase - by targeting a bit later than the actual future vblank signal, to make sure we account for the inherent inaccuracies of our timers system (integer ms interval, callback delays/too-early, etc).

Vladimir didn't like this approach, and so now we're trying a different one with a thread which blocks on actual vblank signal, and then posts an event when we've hit it (no patch yet).

However, this approach is also not perfect. As bas [noted](https://bugzilla.mozilla.org/show_bug.cgi?id=856427#c4), the earlier approach allows easier association with different windows/monitors, possibly easier to integrate with fallback to the current code path, and also probably allows easier disabling of vsync on cases where we can't animate fast enough, or even if gamers want to reduce input/output lag to the minimum possible.

The verdict is not out on this yet, but you can follow the [windows vsync](https://bugzilla.mozilla.org/show_bug.cgi?id=856427) implementation and the global [vsync tracking](https://bugzilla.mozilla.org/show_bug.cgi?id=689418) bug.

###New tab page
Tim taubert has been working on [improving new tab](https://bugzilla.mozilla.org/show_bug.cgi?id=843853) page rendering, which often competes with tab open animation on resources, and may degrade it, especially on slow systems.

While at it, we [got distracted](https://bugzilla.mozilla.org/show_bug.cgi?id=787698) by a patch which regressed animation smoothness after changing its trigger from asynchronous (setTimeout 0) to synchronous. The main arguments were that setTimeout delays the animation start more than we should, and therefore synchronous is the way to go. On the other hand, setTimeout causes a relatively small delay, and could result in considerably smoother animation, since its callback is typically (but not always) handled after the entire current event queue is handled.

Eventually, we settled on using requestAnimationFrame which is asynchronous and will not delay the first animation frame, but may still be handled earlier than setTimeout, especially on slow systems. The numbers approve this theory (15 animation frames using setTimeout, 7 frames after changing to synchronous, and 12 frames after switching to rAF).

Looking back at this, however, this *was* a distraction and somewhat premature discussion. The reason being that these measurements were taken with about:blank page, and not our typical newtab page (which is still too slow to actually measure improved animation when using it, on slow systems).

Note to self: Identify distractions and premature discussions before they consume too much time.

Back to new tab rendering. Trying different semi-random approaches didn't get us too far:

E.g. replacing XUL flexbox with CSS3 - which gave some improvement but added complexity, we've considered delaying the tab animation until after newtab rendering settles, or delaying newtab rendering until after the animation is done (or even disabling tab animation on [slow systems](https://bugzilla.mozilla.org/show_bug.cgi?id=787698), especially now that we have tab animation smoothness measuring infrastructure).

We should probably get back to basics and profile the interaction between tab animation and newtab page rendering (which is not so easy, since a lot of stuff is happening when a tab is animating concurrently with new tab page rendering, but we should still do that), get some real real numbers such that we could point some real fingers ;)

Yet another note: While the newtab rendering is apparently high priority for both the performance team and the fx-team, in a hindsight, it's got too little progress. Maybe we have too many high priority items for the resources we can afford, and we need to get back to re-prioritizing the high priority items?

###Lightweight Themes (LWT, previously: Personas)
MattN and mconley are implementing [Lightweight themes support for Australis](https://bugzilla.mozilla.org/show_bug.cgi?id=813786).

As part of their ongoing effort to keep performance in check, they've performed animation measurements, comparing performance on various systems and getting some real numbers. They're summurizing their results on [this spreadsheet](https://docs.google.com/spreadsheet/ccc?key=0Av64yYvszN34dDdZX0FYWEd3MVpEVzdjYXlFcGpUQmc#gid=0).

Their current results with LTW is that tab animation is on par with Australis (at the above spreadsheet, columns AL-AP, rows 46-58), which itself is almost on par with current theme, which is a great achievement, considering Australis is much more complex.

Keep up the great work! :)
