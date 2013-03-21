---
title: Resource prioritization in Chromium
author: willchan
layout: post
permalink: http://insouciant.org/tech/resource-prioritization-in-chromium/
categories:
  - Tech
tags:
  - chromium
  - performance
  - spdy
---
# 

Resource prioritization is a difficult problem for browsers, because they don’t fully understand the nature of the resources in the page, so they have to rely on heuristics. It seems like the main document is probably the most important resource, so it’d be good to prioritize that highly. Scripts block parsing, and stylesheets and block rendering. Both can lead to discovering other resources, so it’s probably a good idea to prioritize them reasonably highly too. Then you have other resources like media and async XHRs and images. For the exact algorithm Chromium uses, you can refer to the [code][1]. It’s noticeably [suboptimal with regards to sync vs async resources][2], and there is probably more room for improvement. Indeed, we’re kicking around some ideas we hope to play around with in the near future.

 [1]: https://code.google.com/p/chromium/source/search?q=DetermineRequestPriority&origq=DetermineRequestPriority&btnG=Search Trunk
 [2]: http://code.google.com/p/chromium/issues/detail?id=154596

Note that it’s already difficult for browsers to characterize resource priority appropriately, and the techniques that web developers have to employ to get good cross browser performance make the situation worse. For example, Steve and Guy have great recommendations on various techniques to load [scripts][3] or [stylesheets][4] without blocking. The problem, from a resource prioritization perspective, is these techniques defeat the attempts of the browser to understand the resources and prioritize them accordingly. The XHR based techniques will all get a lower priority. The JS based techniques for adding a script tag defeat browser attempts to [speculatively parse when blocked][5]. The script in iframe technique gives the script the priority of a subframe (high). Really, what web devs probably want to do is declare the script/stylesheet resource as async.

 [3]: http://www.stevesouders.com/blog/2009/04/27/loading-scripts-without-blocking/.
 [4]: http://www.guypo.com/technical/eliminating-the-css-bottleneck/
 [5]: http://gent.ilcore.com/2011/01/webkit-preloadscanner.html

Now, how important is resource prioritization? Well, it depends, primarily on the number of domains content is hosted on and the number of resources. That’s because the main places we use resource prioritization are in our host resolution and connection pool priority queues. Chromium unfortunately [caps the concurrent DNS lookups to 6][6] due to the prevalence of [crappy home routers][7]. Our connection pools cap concurrent connections per host to 6, per proxy to 32, and total to 256. Once you’ve got a TCP connection though, application level prioritization no longer plays a role, and the existing TCP connections will interleave data “fairly”. What this also implies is that for truly low priority resources, you may actually want to starve them. You want to [throttle them to reduce network contention with higher priority resources][8][,][9] although when doing so, you run the risk of underutilizing the pipe.

 [6]: https://code.google.com/p/chromium/issues/detail?id=122566
 [7]: http://www.amazon.com/2701HG-B-2Wire-Wireless-Gateway-Router/dp/B001W9ASMS
 [8]: http://trac.webkit.org/browser/trunk/Source/WebCore/loader/cache/CachedResourceLoader.cpp?rev=129070#L743
 [9]: http://trac.webkit.org/browser/trunk/Source/WebCore/loader/cache/CachedResourceLoader.cpp?rev=129070#L743),

[![][11]][11]

 []: {% img /images/2012/10/contention_vs_underutilization.gif %}

It’s sort of silly that the browser has to throttle itself to prevent network contention. What you really want is to send all the requests to the server tagged with priorities and let the server respond in the appropriately prioritized order. Turns out that [some][11] [browsers][12] [already][13] support [this][14]. Now, what happens if the browser and website both support SPDY? Well, then resource prioritization actually has significant effects, and higher priority resources may crowd out lower priority resources on the pipe (which probably is generally, but not always, good for page load time), depending on the server implementation. So if you’re utilizing one of the aforementioned loading techniques to asynchronously load resources, then you may incur performance hits due to suboptimal prioritization or lack of ability for the rendering engine to speculatively issue a request for said resource (when parsing is blocked). Thus, if [HTTP/2.0][15] adopts SPDY features like multiplexing & prioritization, then it’ll become more important for web content to use appropriate markup to allow browsers to figure out appropriate resource prioritization.

 [11]: https://hacks.mozilla.org/2012/03/firefox-aurora-13-is-out-spdy-on-by-default-and-a-list-of-other-improvements/
 [12]: https://www.google.com/chrome
 [13]: http://dev.opera.com/articles/view/opera-spdy-build/
 [14]: http://dev.chromium.org/spdy/
 [15]: http://en.wikipedia.org/wiki/HTTP_2.0