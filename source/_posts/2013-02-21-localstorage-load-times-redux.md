---
title: LocalStorage Load Times Redux
author: willchan
layout: post
permalink: http://insouciant.org/tech/localstorage-load-times-redux/
categories:
  - Tech
tags:
  - localstorage
  - performance
---
# 

I [previously discussed LocalStorage load times][1], and had concluded that it wasn’t too bad. However, when I got back from my new year’s holiday in Patagonia, I started digging through my email backlog and found an interesting line of questioning from GMail engineers, asking why Chrome’s first LocalStorage access was so slow in their use case, and providing data showing it. I, of course, knew why it *should* be slow, because it’s loading the *entire* DB from disk on the first access. But I previously had data that showed it wasn’t so bad. What I realized then was that I was not accounting for the size of the DB. So I added [histograms to record LocalStorage DB sizes and load times by size][2]. Note that LocalStorage is cached both in the renderer and browser processes. Furthermore, note that I’m only recording a single sample for each time LocalStorage is loaded into the process’s in-memory cache. This obviously means that long-lived renderer processes (roughly speaking, tabs) will only get one sample recorded, even if they heavily use LocalStorage.

 [1]: https://insouciant.org/tech/time-to-load-localstorage-into-memory/
 [2]: http://src.chromium.org/viewvc/chrome?view=rev&revision=181855

[![Shows the size of the browser process and renderer process LocalStorage in-memory cache at load time (first access).][4]][4]
Shows the size of the browser process and renderer process LocalStorage in-memory cache at load time (first access). There’s an interesting little jump at 1MB, which I’m unable to explain.

As can be seen, the vast majority of LocalStorage DBs are rather small in size (only on the order of a few KBs). That means that in my previous post where I recorded the LocalStorage load times, the histograms primarily consisted of samples for small LocalStorage DBs. In this iteration, I separated out the load times for DBs into three buckets: size 