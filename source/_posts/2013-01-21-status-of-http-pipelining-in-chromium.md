---
title: Status of HTTP pipelining in Chromium
author: willchan
layout: post
categories:
  - tech
tags:
  - chromium
  - http pipelining
  - performance
comments: true
---
OK, people have asked me this enough times, so it’s time to write down what’s up with pipelining in Chromium. In short, Chromium has a very naive pipelining implementation that is off by default, and it’s unclear if we’ll ever enable it by default. The primary reasons we will not enable it for at least the foreseeable future are:

*   Interoperability concerns
*   Unclear performance benefits

# <a id="interop_concerns">[Interoperability Concerns](#interop_concerns)</a> #

This is probably the most important reason for not enabling HTTP pipelining by default. First, it’s important to recognize how critical interoperability is. If a browser feature breaks web compatibility, irrespective of which entity (browser/intermediary/server/etc) is at fault, the typical user response is to simply switch browsers. If we’re lucky, the user may reach out to our support forums or bug tracker. Indeed, interoperability concerns are the primary reason we [disabled False Start][1], despite [its clear performance benefits][2].

 [1]: http://www.imperialviolet.org/2012/04/11/falsestart.html
 [2]: http://www.belshe.com/2011/05/19/ssl-falsestart-performance-results/

So, when we discuss the interoperability of HTTP pipelining, what are we worried about? Well, for one, we’re concerned about the failure modes of pipelining. What happens when a server or intermediary doesn’t support HTTP pipelining? Does it close the connection? Does it hang? [Does it corrupt responses][3]? If the failure mode is clearly detectable, we can probably retry without pipelining, albeit at the cost of a roundtrip to detect the failure. But what would you do if it hangs? Retry without pipelining after some sort of fixed timeout? If it actually simply corrupts responses, that’s super scary.

 [3]: https://bugzilla.mozilla.org/show_bug.cgi?id=716840

Also, where are the failures happening? Is it primarily origin servers? Intermediaries? In [mnot][4]‘s[ internet draft discussing how to improve pipelining in the open web][5], he proposes some strategies here. For origin servers, he proposes maintaining a blacklist of broken origins. It does seem conceivable that if we could reliably detect pipelining incompatibility, we could maintain a blacklist. It’s not obvious to me that this is reliable though. Well, for what it’s worth, it seems [Firefox has a server blacklist][6] and it seems to maybe work for them. Even assuming we could somehow detect all the pipelining incompatible origin servers, we’d still have to detect broken intermediaries. Indeed, this is a huge part of the problem, and [Patrick McManus identifies this as the primary reason desktop Firefox does not enable pipelining by default][7]. Again, mnot’s I-D proposes a solution: sending pipelined requests to a known pipeline-compatible origin server in order to detect problematic intermediaries. Now, this is problematic for many reasons. For one, it requires “phoning home” to a known server, which is always concerning from a privacy perspective. Moreover, it requires repeating this test on when switching network topologies, which, given the increased mobility of today’s computers (phones, laptops, etc), impacts its utility, not to mention wasting bandwidth which the user may be paying for (e.g. mobile data). Note that these downsides do not rule out the approach, but they must indeed factor into any decision to rely on such a pipelining compatibility test.

 [4]: https://twitter.com/mnot
 [5]: http://tools.ietf.org/html/draft-nottingham-http-pipeline-01
 [6]: http://hg.mozilla.org/mozilla-central/file/1d122eaa9070/netwerk/protocol/http/nsHttpConnection.cpp#l666
 [7]: http://bitsup.blogspot.com/2012/11/a-brief-note-on-pipelines-for-firefox.html

That said, we [implemented **some** of these basic pipelining tests in Chromium][8] and [enabled it for, at its peak, 100% of our Google Chrome dev channel users][9]. As always, it’s important to caveat that the different Google Chrome release channels have different populations, and indeed we did see some differences between our canary and dev channel pipelining tests, so it’s important not to put too much faith in the numbers being exactly representative of all users. That said it still offers some cool insights. For example, the test tries to pipeline 6 requests, and we’ve seen that only around 65%-75% of users can successfully pipeline all of them. That said, 90%-98% of users can pipeline up to 3 requests, suggesting that 3 might be a magic pipeline depth constant in intermediaries. It’s unclear what these intermediaries are…transparent proxies, or virus scanners, or what not, but in any case, they clearly are interfering with pipelining. Even 98% is frankly way too low a percentage for us to enable pipelining by default without detecting broken intermediaries using a known origin server, which has its aforementioned downsides. Also, it’s unclear if the battery of tests we run would provide sufficient coverage.

 [8]: https://code.google.com/p/chromium/issues/detail?id=110794
 [9]: http://src.chromium.org/viewvc/chrome/trunk/src/chrome/browser/net/http_pipelining_compatibility_client.cc?revision=134439&view=markup

There are some potential modifications we could do to address some of these problems. For example, we could give up on true pipelining and do pseudo pipelining. In other words, only start pipelining the next request once the previous request’s response has started coming back. The assumption here is that some broken intermediaries / servers are not expecting a recv() to include multiple responses, so this helps ensure that doesn’t happen. It obviously loses most of the potential latency reduction of pipelining, but it might have much better interoperability, and is less vulnerable to head of line blocking issues. Another idea is only to use pipelining with HTTPS. That helps eliminate the vast majority of interoperability problems with intermediaries (barring SSL MITM proxies and their ilk), [as people trying to ][10][deploy WebSockets have found][11].

 [10]: https://speakerdeck.com/3rdeden/realtimeconf-dot-oct-dot-2012?slide=35
 [11]: http://www.ietf.org/mail-archive/web/tls/current/msg05593.html

I’ve given numbers about problems with intermediaries that our desktop users have seen, but what about mobile? People have been saying how pipelining totally works with mobile, and how it’s important for Chrome on Android to use it. The short answer is I don’t know. Maybe it’d be awesome, or maybe it’s sucky, I really don’t know. I’ve been waiting for us to get better data gathering infrastructure on mobile Chromium before making any claims here. I’ve definitely got some pet theories though. While I do think it’s possible that mobile carriers have fewer pipelining incompatible intermediaries, as far as I know mobile browsers aren’t turning pipelining on/off based on connection type (e.g. WiFi vs 3G), so I don’t think that can explain why mobile browsers seem to have fewer incompatibility issues with pipelining. I can think of two possible explanations. Either the truly problematic intermediaries are client-side (virus scanners and what not), and those are far less prevalent on these mobile devices, or people just have higher tolerance for mysterious hangs and what not on their mobile devices and press reload more often. It’s not like they’ve historically had other browsers to try out on their mobile devices when they get frustrated when the default browser hangs on loading a page :) Anyway, I’m just speculating here and have no real data about mobile pipelining compatibility.

# <a id="unclear_perf_benefits">[Unclear performance benefits](#unclear_perf_benefits)</a> #

[Pipelining can definitely significantly reduce latency][12]. That said, an optimal pipelining implementation is fairly complicated. Darin notes some of the complexities in a somewhat outdated [FAQ on pipelining][13]. Most of the issues revolve around the dreaded head of line blocking issue. On the one hand, pipelining deeper may reduce queueing delay, but if the request gets stuck behind a slow request, then it may have been better to schedule the request on a different pipeline / connection. In order to minimize the likelihood of being stuck behind slow requests, it may be better to have shallower pipelines. This also lets the browser do re-prioritization of HTTP requests before they get assigned to an available pipeline, since if the HTML parser encounters a new high priority resource like an iframe, it will obviously sit behind any other resource requests on the pipeline it is assigned to, despite perhaps being higher priority. It is certainly possible for pipelining to actually [worsen page load time if requests end up waiting for a long time behind slower requests][14]. There are mitigation strategies, primarily based on guessing at request latency based on heuristics like resource type, and also re-requesting head of line blocked requests on a different pipeline / connection, perhaps based on a timer (again, possibly wasting bandwidth, which the user may have to pay for). In the end, they’re just heuristics though, and heuristics can be wrong. Moreover, another problem with pipelines is that if we encounter a transport error on that pipeline, we have to retry all the requests on another pipeline / connection, so it might again be slower.

 [12]: http://bitsup.blogspot.com/2011/02/apex-of-pipelines.html
 [13]: http://www-archive.mozilla.org/projects/netlib/http/pipelining-faq.html
 [14]: https://code.google.com/p/chromium/issues/detail?id=119287

All in all, tuning a pipelining implementation is fairly complicated, and Chromium’s implementation is nowhere near optimal, as we’ve primarily focused on detecting broken intermediaries in our compatibility tests. That said, even if we could tune it, how much of a difference would it make? Guypo’s done some analysis here and believes that [pipelining doesn’t make much of a difference web performance wise][15]. Looking at his study, I agree with most of his conclusions, which makes me even more lukewarm about enabling pipelining by default, at least for desktop.

 [15]: http://www.guypo.com/technical/http-pipelining-not-so-fast-nor-slow/

# Conclusion

Currently the majority of the Chromium developers discussing pipelining aren’t very excited about its prospects, at least for desktop. SPDY and HTTP/2 have always been our long-term plan for the future, but some of us had been hopeful that we could improve performance for users sooner by getting pipelining to work. For the foreseeable future, the pipelining code will probably stay in its zombie state while we work on other performance initiatives, unless we feel like killing it off or we get/gather new data showing interoperability concerns aren’t a big deal anymore or there are dramatic performance improvements to be had. There’s perhaps more reason to be optimistic about mobile, but I’ll wait until someone shows me the data. For now, I’m focusing my attention on HTTP/2. I’m looking forward to next week’s httpbis interim meeting in Tokyo!
