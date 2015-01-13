---
layout: post
title: "Help wanted - Slow touchpad scroll in firefox?"
date: 2013-02-28 19:33
comments: true
categories: 
---
[Bug 829952](https://bugzilla.mozilla.org/show_bug.cgi?id=829952) says that scrolling using the touchpad feels slow on some laptops.

Turns out that it's quite hard to provide a consistent scroll experience on the various platforms which Firefox supports, due to many laptop and touchpad manufacturers, different drivers, OS configurations etc. We could have provided some (or a lot of) configuration UI, but the best solution would be to get it as good as possible out of the box.

And to do that, we need data - and you might be able to help.
<!-- more -->

On my system (Asus laptop, Elantech touchpad, Windows 7), the touchpad does produce pixel-accurate scroll in Firefox, but doesn't send its events as true pixel data (WM_GESTURES windows event). Instead, it sends mouse wheel events, but very rapidly and with fractional line values (it's a legitimate approach with [WM_MOUSEWHEEL](http://msdn.microsoft.com/en-us/library/windows/desktop/ms645617%28v=vs.85%29.aspx)). This results in Firefox processing it through the "mouse wheel path", even when the frequency and value of the events allow pixel-accurate positioning of the content.

It was interesting to compare how this kind of event is handled on other browsers, so I wrote a cross-browser testcase which measures [just that](/testcases/mozwheel-test-v4.html). The best way to use it is to scroll (e.g. with two fingers) consistently from top to bottom of the touch pad. After each such scroll, the page displays related events values, and the overall scrolled distance (pageYOffset). Reset and repeat this few times, until you can get consistent results. Then, try it also with other browsers.

On my system, if Firefox scroll x pixels for one top-to-bottom two-fingers swipe, then, for the same swipe:  
- Chrome scrolls 1.7x  
- IE scrolls 6x  
- Opera scrolls 13x

Quite a difference! With the numbers (consistently) all over the place. For me, Opera scrolls way too fast (does it treat every fractional mouse wheel event as a full line request?), Firefox scrolls slower than I'd like. Chrome could also be faster, and IE feels a bit too fast for my taste. Obviously, there's no real right or wrong here, as it's a very subjective matter.

How do the browsers compare on your system? (Linux? OS X? Windows?) Is Firefox considerably slower than the rest? Does it feel too slow to you? If you have a bugzilla account, you can post directly at the [bug](https://bugzilla.mozilla.org/show_bug.cgi?id=829952), otherwise, you're welcome to leave a comment here.
