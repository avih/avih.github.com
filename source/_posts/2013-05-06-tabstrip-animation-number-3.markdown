---
layout: post
title: "Tabstrip animation progress #3"
date: 2013-05-06 11:36
comments: true
categories: Tabstrip Mozilla vsync newtab
---
Quick update on recent progress.

### Vsync from a thread
Following the first experimental [vsync implementation](https://bugzilla.mozilla.org/show_bug.cgi?id=856427) on Windows (using GetVBlankInfo which is non-blocking and synchronous), I've been experimenting with another implementation which vlad suggested: Run a thread which uses WaitForVBlank (blocking until next vblank) and post the timing event to the main thread.

The conclusions are [here](https://bugzilla.mozilla.org/show_bug.cgi?id=856427#c38). In a nutshell: WaitForVBlank works nicely (when it works), but also blocks the main thread, and while some circumvention is possible (using sleep), it's still far from ideal. Thanks goes (yet again) to Bas for helpful tips on Windows APIs and integration into the Layer manager.

Since OMTC might integrate WaitForVBlank into its pipe more naturally, it was decided to land the original approach - which is very lite and reasonably working, especially when using the main/single monitor - and reconsider the other approach with OMTC.

Expect vsync on Windows nightlies soon.

### New tab previews hurts tab animation
That's [bug 843853](https://bugzilla.mozilla.org/show_bug.cgi?id=843853). I decided to put it on hold for now, as it appears that not enough resources can be directed towards it. It would have been nice, however, to see a more practical list of priorities - which more carefully takes resources into account. Hopefully, we'll have such list soon.

### Timer filter removal regressions
[Bug 590422](https://bugzilla.mozilla.org/show_bug.cgi?id=590422) (remove averaging filter for timers - which probably hurts timing more than it helps) landed some time ago, but still rears its head. The most recent is [lost frames](https://bugzilla.mozilla.org/show_bug.cgi?id=866561) while decoding video.

Apparently what happens is that the filter removal also hurts average timer intervals when high resolution timers are not enabled, and video decoding uses timers without making sure they're in "high-res mode". The fix was quick everyone's happy.

This also brought a new discussion about automatically enabling high-res timers according to some heuristics on the requested timeouts (and possibly also the request source). While it sounds useful, the main argument against it is that high-res timers have slightly higher power draw, and that it could be abused by content pages which use setTimeout(0) while what they actually want is "soon enough", and therefore get much more frequent callbacks than required.

This discussion is [still ongoing](https://bugzilla.mozilla.org/show_bug.cgi?id=590422#c60), you're welcome to contribute more factors to consider.

### Tabstrip animation project goals
There has been some discussion on refining the project goals, to make them both useful from a user perspective (rather than mostly synthetic bar) and also reasonably achievable. We feel comfortable with [this definition](https://wiki.mozilla.org/Performance#Smooth_Tab_Animation), which, in a nutshell, is:

- Have Smooth enough (50FPS or more) animation during the last part of the animation.
- Make sure this happens on some concrete cases we care about (such that newtab open, close a tab and get into a gmail tab, etc).

### What's next?
- Land Windows vsync.

- Get a clear priorities list for newtab slowdown.

- Implement a [regression test](https://bugzilla.mozilla.org/show_bug.cgi?id=848358) which could measure our discrete tab animation goals ([tab animation telemetry](https://bugzilla.mozilla.org/show_bug.cgi?id=828097) is useful, but has lower resolution than we'd like), and give each a score. Still not sure if as a talos test, but right now talos appears to be the proper place for it.

