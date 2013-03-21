---
title: Prioritization Is Critical To SPDY
author: willchan
layout: post
permalink: http://insouciant.org/tech/prioritization-is-critical-to-spdy/
force_ssl:
  - 1
categories:
  - Tech
tags:
  - performance
  - spdy
---
# 

I get the sense that when people discuss SPDY performance features, they pay attention to features like multiplexing and header compression, but very few note the importance of prioritization. I think it’s a shame, because [resource prioritization][1] is [critical][2]. SPDY prioritization enables browsers to advise servers on appropriate priority levels for resources, without having to resort to hacks like [not requesting a low priority resource until higher priority resources have completed][3], which make it difficult for browsers to fully utilize the link. Theoretically, SPDY will let you have your cake and eat it too.

 [1]: https://insouciant.org/tech/resource-prioritization-in-chromium/
 [2]: http://lists.w3.org/Archives/Public/ietf-http-wg/2012AprJun/0523.html
 [3]: https://insouciant.org/tech/throttling-subresources-before-first-paint/

In order to demonstrate the effect of SPDY prioritization, my plan was to roll out SPDY on my own server and show the performance improvement we get from [disabling WebKit’s ResourceLoadScheduler][4] (and thus send all requests immediately to Chromium’s network stack, rather than throttling them) and thus rely on SPDY prioritization instead. However, to my dismay, disabling WebKit’s ResourceLoadScheduler actually **slowed down** my personal website. See this [video][5] of the page load (Chrome 23 stable on left, Chrome 25 canary on right. Stable still throttles subresources before first paint) where most of the page completes seconds faster on Chrome 23 in comparison to Chrome 25.

 [4]: http://trac.webkit.org/changeset/129070
 [5]: http://www.webpagetest.org/video/view.php?id=121222_e5ad6f227bd85a3f09b7962d7916004f66ec6ab4



Ugh, what’s wrong? Does SPDY prioritization not work as advertised? Check out the [Chrome 23 stable release load of my website][6] vs the [Chrome 25 canary load of my website][7] to see the difference in behavior.

 [6]: http://www.webpagetest.org/result/121222_3N_acfc2f0884d67b9fba7d39e337343916/1/details/
 [7]: http://www.webpagetest.org/result/121222_SZ_0cd27c8ef13e08d6ba1c124493e62821/1/details/

[![Chrome 23 Stable load of https://insouciant.org][9]][9]
Chrome 23 Stable load of https://insouciant.org

[![Chrome 25 Canary load of https://insouciant.org][10]][10]
Chrome 25 Canary load of https://insouciant.org

The key thing to notice here is that in Chrome Canary, the critical JS and CSS resources are delayed due to contention. Why is there contention? Shouldn’t SPDY prioritization solve this issue? I looked at the [nginx SPDY patch][10] to find out how they were doing prioritization, and couldn’t figure out how it worked since it didn’t even seem to be present, so I shot Valentin (the nginx dev who authored the SPDY patch) an email asking about SPDY prioritization not working, and he responded with:

 []: http://insouciant.org/tech/prioritization-is-critical-to-spdy/attachment/insouciant_stable/
 []: http://insouciant.org/tech/prioritization-is-critical-to-spdy/attachment/insouciant_canary/
 [10]: http://nginx.org/patches/spdy/patch.spdy.txt

> Yes, it is known. I’m currently working on an implementation that will respect priorities as much as possible.  
> That’s one of the reasons of why we do not push current patch into nginx source.

OK! That makes sense. Nginx’s SPDY implementation is still in beta, so it “works” but does not respect prioritization yet. That’s fair, because they’re still working on it. The problem is that people are deploying real websites using nginx’s SPDY support, even though prioritization doesn’t work at all. For example, check out the [Chrome 23 stable][11] vs [Chrome 25 canary][12] load time waterfalls and [video][13] for :

 [11]: http://www.webpagetest.org/result/121224_GW_5d074747d79ba4f51b79b169662a7447/1/details/
 [12]: http://www.webpagetest.org/result/121224_9H_3a0c207b85c38f3b36f8c84f2b987e69/1/details/
 [13]: http://www.webpagetest.org/video/view.php?id=121224_62a84fe14d026ce05f784f7fb352b9a82f2afbe4



Looking at the waterfalls I’ve linked above, you can see that Chrome 23 stable achieves a faster first paint and page load time because it reduces contention on the stylesheets and script, which both lets it reach first paint faster and also DOMContentLoaded, which fires off some more requests, so the overall page load also completes sooner, despite Chrome 25 canary’s better early link utilization.

If you’ve heard the SPDY team talk about optional features, you’ll know that we hate them (see Mike’s comment (d) in his [blog post][14]). However, despite prioritization’s importance, we sadly aren’t able to make servers incompatible if they don’t respect them properly. That’s because the prioritization has to be advisory, since it’s quite reasonable for a low priority resource to be immediately available for transmission, if say it’s cached, whereas a higher priority resource may need more server side processing. It would be wasteful not to allow the server to transmit lower priority resources while higher priority resources aren’t yet available, and since it’s impossible for the client to distinguish between this case and a lack of prioritization, it’s impossible to enforce server support from the client side.

 [14]: http://www.belshe.com/2012/03/29/comments-on-microsofts-spdy-proposal/

While I’m excited that it shows people are excited about SPDY, I think it’s a bit unfortunate that nginx’s incomplete SPDY implementation has prematurely seen relatively wide adoption. It would be terrible if we reached a state where a nontrivial chunk of SPDY server deployments did not respect prioritization and there was little chance of change, since that would pressure browsers to give up on SPDY prioritization and fall back to throttling low priority resources before first paint. That said, I’m not too worried yet. SPDY is still in its early stages of deployment, and old SPDY versions will get disabled when we finish the SPDY/4 spec and start deploying that, not to mention HTTP/2. But once we get to the HTTP/2 deployment stage, we’ll have to make sure to push hard on server implementations to get prioritization right so clients can depend on its support server-side.