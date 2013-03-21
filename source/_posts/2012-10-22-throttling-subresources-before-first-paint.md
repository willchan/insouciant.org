---
title: Throttling Subresources Before First Paint
author: willchan
layout: post
categories:
  - tech
tags:
  - chromium
  - performance
  - webkit
---
In my last [post about resource prioritization][1], I mentioned that WebKit actually [holds back from issuing subresources that can’t block the parser before first paint][2]. Well, that’s true, except for the Chromium port, because recently we decided to [disable that][3]. You may be wondering, why would we do that?

 [1]: /tech/resource-prioritization-in-chromium/
 [2]: http://trac.webkit.org/browser/trunk/Source/WebCore/loader/cache/CachedResourceLoader.cpp?rev=129070#L743
 [3]: http://trac.webkit.org/changeset/129070

Well, we’re doing it because the rendering engine is not the best place to be making resource scheduling decisions. The browser has access to more information than the renderer does. For example, if the origin server supports SPDY, then it’s better to simply issue all the requests. Or maybe the tab is in the background, so we should be de-prioritizing or outright suppressing the tab’s resource requests in order to prevent contention. Or maybe given the amount of contention, we should start the hostname resolution or preconnect a socket, rather than doing a full resource requests.

Therefore, we’re disabling WebKit’s various resource loading throttles in order to get all the resource requests to the browser and have it handle resource scheduling. We haven’t re-implemented WebKit’s mechanisms yet browser side, so currently we have a situation where we aren’t throttling subresources before first paint. So if you’re running a Chromium build after [r157740][4] (currently, only Google Chrome dev and canary channels), then your browser isn’t throttling subresources before first paint. It’s interesting to see what effects this has on page load time.

 [4]: http://src.chromium.org/viewvc/chrome?view=rev&revision=157740

For example, for www.gap.com, we can [compare][5] page loads on [Chrome 22][6] (stable) vs [Chrome 24][7] (dev) using [WebPageTest][8]. In this case, we see that the onload and [speed index][9] are **both** worse in Chrome 24. The speed index is clearly worse because the first paint occurs way earlier for Chrome 22 than for Chrome 24. Looking a bit more closely at the waterfalls, that seems to be due to the globalOptimized.js taking longer on Chrome 24 than on Chrome 22. Poking into the code, we see the document is synchronously loading that script, thereby blocking parsing and slowing down both the first paint and the DOMContentLoaded event. The DOMContentLoaded event is key here, since it appears to trigger a number of other resource requests, so slowing down the DOMContentLoaded event also slows down onload. Examining the connection view, we see that the reason that globalOptimized.js (264.4 KB) takes longer to download is due to increased bandwidth contention from image resources.

 [5]: http://www.webpagetest.org/video/compare.php?tests=121020_83_BF0,121020_1C_BF5
 [6]: http://www.webpagetest.org/result/121020_83_BF0/
 [7]: http://www.webpagetest.org/result/121020_1C_BF5/
 [8]: http://webpagetest.org
 [9]: https://sites.google.com/a/webpagetest.org/docs/using-webpagetest/metrics/speed-index

<figure>
{% img /images/2012/10/gapchrome22.png 'gap.com waterfall, connection, bandwidth for chrome 22' 'gap.com waterfall, connection, bandwidth for chrome 22' %}
<figcaption>gap.com waterfall</figcaption>
</figure>
<figure>
{% img /images/2012/10/gapchrome24.png 'gap.com waterfall, connection, bandwidth on chrome 24' 'gap.com waterfall, connection, bandwidth on chrome 24' %}
<figcaption>gap.com waterfall</figcaption>
</figure>

Examining another [example][10] (cvs.com), [Chrome 22][11] again has better first paint / speed index scores, but [Chrome 24][12] has a shorter onload time. Diving into the waterfalls again, we can see that the reason Chrome 22 again has shorter first paint times is because it gets the stylesheets sooner, and [stylesheets block][13] [first paint][14] in order to prevent [FOUC][15]. What’s interesting about this case is that there’s no real bandwidth contention this time, since the last 3 stylesheets only add up to around 7KB. The contention is actually on the connections per host (limit is 6), since the images are requested first, and until some complete, the stylesheets (higher priority resources) requests cannot begin. However, unlike the gap.com case, there isn’t a period of low bandwidth utilization due to waiting for resources to be requested during the DOMContentLoaded event, so issuing the subresource requests earlier results in overall better bandwidth utilization, and thus reaching onload sooner.

 [10]: http://www.webpagetest.org/video/compare.php?tests=121021_W3_19D,121021_EA_19E
 [11]: http://www.webpagetest.org/result/121021_W3_19D/
 [12]: http://www.webpagetest.org/result/121021_EA_19E/
 [13]: https://code.google.com/searchframe#OAMlx_jo-ck/src/third_party/WebKit/Source/WebCore/rendering/RenderBlock.cpp&exact_package=chromium&ct=rc&cd=2&q=fouc&l=2921
 [14]: https://code.google.com/searchframe#OAMlx_jo-ck/src/third_party/WebKit/Source/WebCore/rendering/RenderLayer.cpp&exact_package=chromium&ct=rc&cd=3&q=fouc&l=3005
 [15]: http://en.wikipedia.org/wiki/Flash_of_unstyled_content

<figure>
{% img /images/2012/10/cvschrome22.png 'cvs.com waterfall, connection, bandwidth for chrome 22' 'cvs.com waterfall, connection, bandwidth for chrome 22' %}
<figcaption>cvs.com waterfall</figcaption>
</figure>
<figure>
{% img /images/2012/10/cvschrome24.png 'cvs.com waterfall, connection, bandwidth on chrome 24' 'cvs.com waterfall, connection, bandwidth on chrome 24' %}
<figcaption>cvs.com waterfall</figcaption>
</figure>

Yeah, there are cases where this change worsens the experience, and cases where it actually improves the user experience. In the real world though, what does it usually do? [Pat Meenan][16] ran a test of Chrome stable vs Chrome canary for us awhile back to check it out on a bunch of websites, and in aggregate, we saw a minor improvement in onload times with the new behavior, but a hit on the speed index and a significant hit on first paint time. Therefore, we’re calling this a [regression][17] and will either fix or revert the change by the time we hit code complete for Chrome 24.

 [16]: https://twitter.com/patmeenan
 [17]: https://code.google.com/p/chromium/issues/detail?id=157763

Update: Thanks to [Steve][18] for editorial suggestions and teaching me how to compare tests on WebPageTest :)

 [18]: https://twitter.com/souders
