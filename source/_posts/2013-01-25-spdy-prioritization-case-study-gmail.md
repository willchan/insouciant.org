---
title: 'SPDY Prioritization Case Study &#8211; Gmail'
author: willchan
layout: post
permalink: http://insouciant.org/tech/spdy-prioritization-case-study-gmail/
categories:
  - Tech
tags:
  - chromium
  - performance
  - spdy
---
# 

Awhile back, Gmail team asked some Chromium devs why Google Chrome downloaded CSS so much more slowly than JS. This sounded strange to me, since I knew the code, and Chromium [clearly prioritizes script and stylesheets at the same level][1] (look for DetermineRequestPriority). I was initially skeptical of their claim, but then they said they had data that proved otherwise. Specifically, they told me:

 [1]: https://src.chromium.org/viewvc/chrome/trunk/src/content/browser/loader/resource_dispatcher_host_impl.cc?revision=178529&view=markup

    I start downloading 400KB gzip'ed JS
    JS  x--------;
    Short after that I start downloading a JSON file containing our CSS, 60KB gzip'ed.
    JS  x-------->;
    CS   x---->;
    I would expect to see this, in 99% of the cases:
    JS x---------------------------------------| done
    CS   x--------| done
    But what we see often is this pattern:
    JS x---------------------------------------| done
    CS   x-----------------------------------------------------------------------------| done

At this point I was pretty intrigued. There’s nothing like a mystery to pique my interest.  First, to understand Gmail’s loading infrastructure, check out [Gmail’s webperf slides][2] to see how they bootstrap the initial main page and then load the main script and stylesheet. So, I dove into the debugger and saw that, not only is the CSS downloaded as JSON, it’s downloaded using an XHR (not very surprising in retrospect). On the other hand, the javascript resource isn’t a true javascript resource, it’s actually an iframe! I dive into Chrome DevTools to see what’s going on:

 [2]: http://www.w3.org/2012/11/webperf-slides-hundt.pdf

[![Viewing Gmail's main JS and CSS loading in DevTools][4]][4]
Viewing Gmail’s main JS and CSS loading in DevTools

As can be seen, the CSS is smaller, and it starts after the JS load started, but it generally finishes afterward. Indeed, in practice, this matches what we see in the wild. The question is, why? Well, the answer of course is that this is expected behavior with SPDY.

 []: https://insouciant.org/wp-content/uploads/2013/01/GmailDevTools.png
 [4]: https://insouciant.org/wp-content/uploads/2013/01/GmailDevTools.png

To figure out why, we need to see what the resources look like from the browser’s perspective. As previously mentioned, the javascript resource is actually an iframe where the html is full of inline script blocks. As the code I linked to earlier shows, an iframe is requested with the highest priority. And the CSS resource is actually JSON that is requested via an XHR. Again, as the code I linked to earlier shows, an XHR is requested with the lowest priority. If we look at the [SPDY3 spec][5], we see that The sender and recipient SHOULD use best-effort to process streams in the order of highest priority to lowest priority. As specced, that means that Gmail’s iframe should pre-empt the XHR served over the same SPDY session. That’s of course why, from the Gmail team’s perspective, the CSS download seems to download more slowly than JS. They’re measuring from Javascript using techniques that do not have visibility into what’s happening in the SPDY session, so they can see that the CSS JSON is taking a long time to download, but cannot tell that it’s because it’s being downloaded at a lower priority than other resources and thus is getting pre-empted. Anyhow, mystery solved!

 [5]: http://dev.chromium.org/spdy/spdy-protocol/spdy-protocol-draft3#TOC-2.3.3-Stream-priority

But we should step back here and ask, is SPDY doing the right thing? Should a resource (or rather, a SPDY stream) at a higher priority starve a resource at a lower priority? It’s very reasonable how it made Gmail team wonder why CSS looked so slow to download. But if you think about it, SPDY prioritization isn’t changing the actual throughput. It’s simply affecting the order in which the resources’ bytes are being sent by the server, thus the time to download all resources should be the same in absence of prioritization. However, outside of exceptional, adversarial cases, it should generally result in individual resources completing sooner. Now, the question is whether or not it’s better to have individual resources complete sooner, or to get more interleaving amongst resources. It’s useful to note that for many resources such as script, the web rendering engine is not able to progressively process them. The entire script is delivered as a whole to the script engine (V8 in Chromium’s case). On the other hand, resources like HTML are able to be incrementally processed. That said, would it be better to completely deliver one HTML resource and then another HTML resource or interleave them? That’s completely unclear to the browser. For images, it’s likewise unclear whether or not it’s better to deliver them serially or interleaved, although it’s generally more likely that images earlier on in the document will be in the viewport. But some images are progressive, and interleaving them will let the user see lower quality versions of more images sooner, which is arguably a better user experience. And since documents don’t always have the layout markup for images, it may be better to interleave the images in order to get the dimensions for images sooner so the rendering engine can layout the page correctly sooner. Current SPDY prioritization allows for conveying fixed priority levels, but it only has very rough semantics for describing where the browser wants strict ordering of resources versus equivalency (such that a server could interleave if it’s better). Indeed, SPDY3 has 3 bits for priorities, which isn’t necessarily enough since there are generally far more than 8 resources per page. This, amongst other reasons, is why we’re revamping SPDY prioritization in SPDY4, and I will [present many of these use cases to the httpbis working group next week in Tokyo][6] so we can design for them for HTTP/2.

Digging in even deeper, it’s curious to ask *why* Gmail is downloading JS using an iframe with inline script, and *why* it’s downloading CSS as JSON using an XHR. Chromium gives scripts and stylesheets the same priority levels, so if Gmail didn’t use these techniques, Google would probably have interleaved the two resources. As far as the JS goes, there are multiple reasons to use the [script in iframe][7] technique, and I won’t cover them all here. For one, it is an effective cross browser technique for loading resources in parallel. Also, Gmail is segmenting the script into multiple script blocks which allows the browser to incrementally feed script chunks to the javascript engine for parsing and execution in parallel with the download of more script blocks in the iframe. Moreover, this technique also allows Gmail to render the progress bar as each script block executes. Amongst other downsides though, it defeats the browser’s attempt to recognize the correct resource type and accurately prioritize the resource. In this case, it introduces contention with the initial main page, since both are considered to be documents and thus are at the same priority level, which may or may not be a good thing. So I asked Dr. Barth to help me figure out a replacement for the script in iframe technique, and he [proposed supporting multipart/mixed responses for script elements][8].

 [6]: https://docs.google.com/presentation/d/1OfgPJsW6P7pky5PiyEzBNZnf5dWXq-y19ReSSg6BIeM/pub?start=false&loop=false&delayms=3000
 [7]: http://stevesouders.com/cuzillion/?ex=10012&title=Script in Iframe
 [8]: http://lists.w3.org/Archives/Public/public-whatwg-archive/2012Dec/0016.html

As far as why Gmail downloads CSS as JSON using an XHR, it’s actually for a number of reasons, not all of which I’m going to dive into here. But one major reason is to avoid blocking first paint, since rendering engines will block first paint until relevant stylesheets come in, in order to prevent [FOUC][9]. Gmail doesn’t want to block first paint on downloading the CSS for the main part of the web app, since it wants to render the progress bar in the meanwhile. There are other reasons CSS as JSON is useful for them, but if we ignore those, then what Gmail needs is a way in the web platform to declaratively (so the speculative parser can discover the resource sooner) asynchronously load stylesheets in a manner that doesn’t block first paint, fires a load event, and is properly recognized by the web platform as a stylesheet download (so it can be appropriately prioritized) rather than an opaque blob. There are many loading techniques, like creating a link element and appending from script, that meet most of these goals, but not all of them.

 [9]: https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=2&ved=0CD4QFjAB&url=http://en.wikipedia.org/wiki/Flash_of_unstyled_content&ei=yrMCUZ3dLKeViQKPpYDIBQ&usg=AFQjCNGg8uCVCVVjdE0Lb3h8gXE0b5kxRg&bvm=bv.41524429,d.cGE

Gmail’s a fascinating case to study, since they’ve done so many web performance optimizations that had to work across a large number of browsers, many of them very old. These web performance techniques were the best options for older browsers, but they interact in interesting, potentially suboptimal ways with modern browsers.
