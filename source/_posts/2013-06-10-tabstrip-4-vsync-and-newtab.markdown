---
layout: post
title: "Tabstrip #4 - Vsync, newtab, talos, paint starvation"
date: 2013-06-10 21:38
comments: true
categories: vsync newtab talos tsvg tscroll paint starvation
---
I've been slightly behind with my blog updates and there has been some great progress recently, so this post covers a bit more than usual.

###Vsync

Vsync has finally [landed](https://bugzilla.mozilla.org/show_bug.cgi?id=856427) on Windows Nightlies not long ago. This means that Firefox will now synchronize its paints with the actual refresh rate of the monitor (if available), which is essential for properly smooth animations. This will work (and is enabled by default) on Windows Vista and later with DWM enabled ("Aero" themes). This also works with WebGL content such as [Epic Citadel](https://blog.mozilla.org/futurereleases/2013/05/02/epic-citadel-demo-shows-the-power-of-the-web-as-a-platform-for-gaming/) ([demo](http://www.unrealengine.com/html5/)).

Without vsync, Firefox uses 60hz refresh rate by default, which works relatively decently with 60hz displays, but fails badly at displaying properly smooth animations on monitors with other refresh rates (50hz on quite a few laptop displays, 100hz monitors, etc).

However, the current implementation synchronizes with the main display only, so if using a multi-monitor setup, Firefox windows on secondary monitors might not gain from this.

You can control the refresh rate manually by modifying `layout.frame_rate` at `about:config` (and restart Firefox). The default value is `-1` which means "Use vsync if available or 60hz otherwise". Any other integer value > 0 will force that specific refresh rate.

Naturally, if Firefox can't animate specific content quickly enough to keep up with the designated rate, then the actual rate will be lower.

###about:newtab

`about:newtab` is the default page when opening a new empty tab - with the thumbnails of recently visited pages. It was [affecting tab animations badly](https://bugzilla.mozilla.org/show_bug.cgi?id=752839#c11), to the point of displaying as little as a 3 frames slideshow on low-end systems while animating the tabstrip after pressing the `+` button to add a tab.

*Tim Taubert* from the Firefox team has done some extensive work [recently](https://bugzilla.mozilla.org/show_bug.cgi?id=843853) on several bugs which fall into two categories:

- Enable newtab [preload by default](https://bugzilla.mozilla.org/show_bug.cgi?id=791670). Newtab preload means that Firefox prepares the newtab page in the background, such that when opening a new tab it's already rendered and therefore could be quickly displayed with very little processing. It has been working for a while but disabled by default due to accessibility issues. *Tim* removed this blocker, as well as simplified the preload mechanism and made sure that background preload [doesn't happen during animation](https://bugzilla.mozilla.org/show_bug.cgi?id=875509). This landed about a week ago and newtab preload is now enabled by default on nightlies.

- Prevent unnecessary reflows of the newtab page. Reflow is the process of re-calculating the layout of the page, e.g. after window resize, and it's relatively CPU intensive. Tim discovered that newtab causes [unnecessary reflows](https://bugzilla.mozilla.org/show_bug.cgi?id=876374) on several cases, some of them [related to preload](https://bugzilla.mozilla.org/show_bug.cgi?id=875257). He fixed them and also added an automatic test to detect if new reflows are accidentally introduced over time.

The result of all this work is that animating a new tab with the newtab page is now almost as smooth as opening a blank tab, with few further potential improvements to be worked on soon. Well done!

###Talos

[Bug 590422](https://bugzilla.mozilla.org/show_bug.cgi?id=590422) (remove timers averaging filter) landed more than two months ago and should greatly improve timers accuracy, but it caused few regressions which are still being worked on. Talos [tscroll](https://bugzilla.mozilla.org/show_bug.cgi?id=845943) was already modified as a result - which landed about 2 weeks ago, and recently [tsvg regressions](https://bugzilla.mozilla.org/show_bug.cgi?id=854746) from this bug were examined as well.

As it turned out, tsvg was rendering SVG animations at 100-200ms intervals, and measuring/reporting the overall runtime as representative of SVG performance. At 4 of the 7 animation tests within the tsvg suite, more than 95% of the time was spent inside `setTimeout` with the browser completely idle. This resulted in tests which in practice represent the accuracy of timers more than anything else.

To make sure the overall runtime represents actual SVG performance, we should iterate the animation frames as fast as possible. We could do this with `setTimeout(loop, 0)`, but since Firefox renders to screen at 60hz, and it's usually smart enough to delay reflow calculations until it's absolutely necessary, we could end up fully processing only one in few frames when the SVG iterations are quicker than 60Hz, and at tsvg they're much quicker than that.

To make sure each frame is fully processed on each iteration, we should therefore use `requestAnimationFrame` (fondly nicked `rAF`) - which makes sure that once we're back at the callback, whatever layout changes which we introduced were already fully processed and flushed by Firefox. However, `rAF` iterates at 60hz by default (or vsync rate), which is usually not fast enough to stress our SVG tests. The solution is to set `layout.frame_rate` to a high enough value (during these tests) which causes Firefox to iterate and fully-process each frame as fast as possible. *Joel Maher* helps a lot with testing and deploying the new tsvg tests, and we hope to land it soon.

Making tsvg iterate ASAP also brought us back to tscroll. The recent change of tscroll was to use `rAF` instead of `setTimeout` for scroll iterations. However, while it represents scroll smoothness under normal conditions, it doesn't detect small regressions - since Firefox isn't actually being stressed and the scroll doesn't limit our iteration frequency. But stressing the browser with ASAP iteration will lose the real-world simulation of normal refresh rates. So we don't have a single approach to satisfy both of these needs.

We're still [considering this](https://bugzilla.mozilla.org/show_bug.cgi?id=771326), but it seems talos will eventually run tscroll twice: once @60hz to make sure we can keep up at real-world scenarios, and once with ASAP iterations to detect performance regressions at much higher resolutions.

###Paint starvation

While working on tsvg with ASAP iterations, we noticed that Firefox sometimes appears frozen for the entire animation (typically less than 1 second) - it only shows the last animation frame. While this happened when `rAF` was set to iterate ASAP, we discovered that there are scenarios under perfectly normal conditions which would exhibit exactly the same symptom.

This is now [bug 880036](https://bugzilla.mozilla.org/show_bug.cgi?id=880036). It shows is that if, for some reason, Firefox can't iterate frames as fast as it wants to (60hz by default), then it may spend too much time on these iteration and prevent other event types from being handled on time. The most noticeable one is the paint event, since it makes Firefox appear frozen, but possibly other events are starved as well.

A patch which seems to fix the issue was already posted, but since this freeze appears to 'fix' itself after about 2s, we want to first understand this unfreeze mechanism, and possibly fix it instead of posting a patch to cover up for another bug.