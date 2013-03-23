---
title: LocalStorage Load Times Redux
author: willchan
layout: post
categories:
  - tech
tags:
  - localstorage
  - performance
comments: true
---
I [previously discussed LocalStorage load times][1], and had concluded that it wasn’t too bad. However, when I got back from my new year’s holiday in Patagonia, I started digging through my email backlog and found an interesting line of questioning from GMail engineers, asking why Chrome’s first LocalStorage access was so slow in their use case, and providing data showing it. I, of course, knew why it *should* be slow, because it’s loading the *entire* DB from disk on the first access. But I previously had data that showed it wasn’t so bad. What I realized then was that I was not accounting for the size of the DB. So I added [histograms to record LocalStorage DB sizes and load times by size][2]. Note that LocalStorage is cached both in the renderer and browser processes. Furthermore, note that I’m only recording a single sample for each time LocalStorage is loaded into the process’s in-memory cache. This obviously means that long-lived renderer processes (roughly speaking, tabs) will only get one sample recorded, even if they heavily use LocalStorage.

 [1]: /tech/time-to-load-localstorage-into-memory/
 [2]: http://src.chromium.org/viewvc/chrome?view=rev&revision=181855

<figure>
{% img /images/2013/02/chrome_26_dev_local_storage_db_sizes.png %}
<figcaption>
Shows the size of the browser process and renderer process LocalStorage in-memory cache at load time (first access). There’s an interesting little jump at 1MB, which I’m unable to explain.
</figcaption>
</figure>

As can be seen, the vast majority of LocalStorage DBs are rather small in size (only on the order of a few KBs). That means that in my previous post where I recorded the LocalStorage load times, the histograms primarily consisted of samples for small LocalStorage DBs. In this iteration, I separated out the load times for DBs into three buckets: size < 100KB, 100KB < size < 1MB, and 1MB < size < 5MB, and got the following results.

<figure>
{% img /images/2013/02/chrome_26_dev_local_storage_prime_time_windows.png %}
<figcaption>Shows the load times of LocalStorage caches by size and process. Windows only.</figcaption>
</figure>

<figure>
{% img /images/2013/02/chrome_26_dev_local_storage_prime_time.png %}
<figcaption>Shows the load times of LocalStorage caches by size and process.</figcaption>
</figure>

The short of it is that the long tail is terribly slow for large DBs. This definitely has implications for people considering [using LocalStorage as an application resource cache for performance reasons][3], since caching resources will probably noticeably increase the size of the LocalStorage DB, and also has some [security implications][4]. And just in case you forgot, the API is synchronous, so the renderer main thread is blocked while the browser loads LocalStorage into memory, at most likely the worst time for performance – initial page load. And note the jump at the end of the chart…that’s because my max bucket was capped at 10s, since I didn’t think we’d have many samples that exceeded that. Unfortunately, I was wrong :(

 [3]: http://www.stevesouders.com/blog/2011/03/28/storager-case-study-bing-google/
 [4]: http://lists.w3.org/Archives/Public/public-webcrypto-comments/2012Aug/0076.html

In the end, as with all web performance techniques, you really should measure the impact of the technique to make sure that it is actually a performance win in your use case.

PS: [I’ve published the full CDFs from the histogram data][5]. Note that this consists of data gathered from Google Chrome 26 opted-in dev channel users in February over a space of 5-6 days. The results should definitely change somewhat for stable channel users (probably for the worse...dev channel users tend to have more advanced machines). Take the Mac and especially the Linux data with extra grains of salt, since their sample sizes are significantly lower.

 [5]: https://docs.google.com/spreadsheet/pub?key=0AufvXHY7HPw0dFZPTkk2NVVHZzBDbXBpeGI2b3dpelE&output=html
