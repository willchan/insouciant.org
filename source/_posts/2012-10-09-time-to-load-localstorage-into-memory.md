---
title: Time to load localStorage into memory
author: willchan
layout: post
categories:
  - tech
tags:
  - chromium
  - localstorage
  - performance
comments: true
---
For all the reasons that Taras Glek lists in [https://blog.mozilla.org/tglek/2012/02/22/psa-dom-local-storage-considered-harmful/][1], I’ve been very skeptical of using localStorage. Synchronous APIs that do I/O are the suck as far as I’m concerned. I was curious about how much of an effect this had in practice, so [I added Chromium histograms][2]  
[ for it][2]. This tracks the time that it takes to load localStorage from the persistent disk store, which Chromium does on the first access, after which subsequent accesses \*should\* be a simple memory access or IPC roundtrip memory access. There’s potentially room for optimization here, where we could speculatively prime the in-memory data structure before the first access. I’m skeptical that’d have much impact though.

 [1]: https://blog.mozilla.org/tglek/2012/02/22/psa-dom-local-storage-considered-harmful/
 [2]: http://src.chromium.org/viewvc/chrome?view=rev&revision=159400

I only have data from a day’s worth of Google Chrome dev channel weekend traffic on desktop Chrome. Mobile may well be a different beast, and I’ll be curious to see the data when I get a hold of it. In any case, here’s what I see:

Time in ms to prime localStorage from disk (Win/Mac/Linux) by percentile:  
50th: 0/0/0  
75th: 2/0/0  
90th: 40/17/17  
95th: 160/57/160  
99th: 1200/890/1200

This data is very much subject to interpretation. My read of it is, it’s actually not so bad. It’d be interesting to do more slicing and dicing of data (distribution per user, distribution based on localStorage size, yada yada). I used to diss localStorage a lot before, but after seeing this data, I’m less concerned about its effect on performance, at least on desktop. I still think it’s probably a bad idea on mobile, but I’ll reserve judgment until I get data for Chrome on Android and iOS.
