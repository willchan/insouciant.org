---
title: Some more reasons hacking around HTTP bottlenecks sucks and we should fix HTTP
author: willchan
layout: post
categories:
  - tech
tags:
  - css sprites
  - hostname sharding
  - http
  - performance
  - spdy
---
{% img /images/2012/11/HateHTTP.png 'It&#39;s not you I hate Cardassian...' 'It&#39;s not you I hate, HTTP. I hate the hacks people write because of you' %}

Web developers keep devising more and more clever hacks that help work around HTTP level issues like overhead from roundtrips and lack of parallelism. The problem is that a lot of these techniques are not magic bullets, but have some important tradeoffs. Here I go through two such hacks and how they are suboptimal and hopefully will be unnecessary if a prioritized multiplexing protocol that looks like SPDY gets standardized as HTTP/2.0. With prioritized multiplexing, the natural way of authoring content will hopefully also be the faster way, and then these hacks can hopefully fade away into irrelevance. This isn’t an exhaustive list of hacks and why they suck. They’re simply two that I have run into as of late that I haven’t seen discussed elsewhere (but could be wrong).

# <a id="css_sprites">[CSS Sprites](#css_sprites)</a> #

[CSS Sprites][2] are pretty neat, because they help reduce roundtrips by combining multiple image into a single image, thereby reducing the number of image requests ([at some cost][3]). CloudFlare does an awesome job spriting their website. For example, their CDN features page utilizes CSS sprites to great effect, helping reduce the number of requests. But it’s useful to examine the [waterfall][4] to see how referencing images in the stylesheet instead of the main document can slow things down.

 [2]: https://developers.google.com/speed/docs/best-practices/rtt?hl=en#SpriteImages
 [3]: https://www.google.com/search?q=problems with css sprites
 [4]: http://www.webpagetest.org/result/121124_WV_DHK/1/details/

{% img /images/2012/11/cloudflarespritedownload.png 'CloudFlare Sprite Download Waterfall' %}

As you can see, referencing the images in an external stylesheet requires downloading the stylesheet before the image gets requested. One might reasonably ask why we [don’t speculatively download image resources][6] in the stylesheet as the stylesheet comes in, but many stylesheets include a bunch of resources that are never used. For example, here’s the [stylesheet][7] included on my Google search for [flowers] when I’m signed in. Chrome tells me there are 96 instances of “background:url(” in that stylesheet, and you can bet that we’re not using all those resources on the Google search page. Note that [waiting for the external stylesheet download to complete to issue image requests happens anyways][8] in Chrome stable and IE.

 [6]: https://code.google.com/p/webkit-mirror/source/browse/Source/WebCore/html/parser/CSSPreloadScanner.cpp?r=2e3b7d95ca518663e911b42429ed1c563a7d667c
 [7]: https://plus.google.com/_/apps-static/_/ss/sbw/ver=17b50ogit0knz/am=!uxU9uUXNl_eJBf3iMaRoAqSv6P37yJZlfKGQ22rkk0T2/bf=BA/r=O/rs=AItRSTO79WXIfuvH8yFOtZzYeKhsNUENiw
 [8]: /tech/throttling-subresources-before-first-paint/

So we have the situation where, in order to reduce the number of image requests, many content authors switch from using normal  tags to using CSS sprites. This prevents the browser from discovering the  resources during speculative parsing (e.g. [WebKit’s PreloadScanner][9]) and instead we have to wait for the stylesheet to complete downloading and then for the rendering engine to match the CSS selector for the parsed element to decide to request the appropriate background image. It’s particularly frustrating because CSS sprites are a commonly accepted “best practice” by web developers, because they do indeed help workaround deficiencies in HTTP, but it also **prevents** the browser from loading resources as early as possible.

 [9]: http://gent.ilcore.com/2011/01/webkit-preloadscanner.html

Note that CloudFlare’s making the right tradeoffs here in an HTTP world. But they could be even faster if they undid that optimization and just declared the image resources normally in the document when serving over a prioritized multiplexing protocol like SPDY (and hopefully HTTP/2.0 in the future!). CloudFlare’s been pretty quick about iterating on new technologies, so I have no doubt that they’ll pick up on this missed optimization.

# <a id="hostname_sharding">[Hostname Sharding](#hostname_sharding)</a> #

[Hostname sharding][10] is another commonly accepted “[best practice][11]” designed to work around HTTP’s lack of parallelism. It’s true that it does indeed make browsing faster in general, but it has numerous downsides. It increases the number of connections, thereby increasing resource consumption at TCP endpoints and middleboxes, increases DNS traffic and entries (and the TCP connection is blocked on the DNS lookup), and splits congestion control information across multiple connections. Moreover, more parallelism doesn’t necessarily make things faster. It might result in more contention and more interleaving. For more details, see [Patrick’s post][12] on the matter.

 [10]: https://developers.google.com/speed/docs/best-practices/rtt#ParallelizeDownloads
 [11]: http://www.stevesouders.com/blog/2009/05/12/sharding-dominant-domains/
 [12]: http://bitsup.blogspot.com/2011/02/http-parallel-connections-firefox.html

The benefits have been written about numerous times already, so I won’t bother repeating them. But I will warn about the dangers of hostname sharding, in particular because [Patrick McManus][13] showed me that it was being abused. Some sites seem to be using a scary number of shards, and it’s pretty bad for the internet and for their performance. Check out the [waterfall for 163.com][14]. It’s pretty absurd. Here we have one of the most popular sites on the web and it’s opening up an astonishing 210 connections to serve its homepage! 72 of those connections are sharded across img[1-6].cache.netease.com, some shards across img[1-3].126.net, and a lot of connections on g.163.com. Glancing at the connection view, it seems that around 100 connections are utilized at the same time. This is obviously absurd and totally bypasses TCP slow start, since all of these connections will start with a server-side initcwnd of at least 3, and very possibly more, meaning an initial congestion window of 300 ! That’s just silly and totally busted, and you can see from [CloudShark][15] that the page load exhibits a [number][16] of self-inflicted [congestion][17] [wounds][18]:

{% img /images/2012/11/163comcongestion.png %}

 [13]: https://plus.google.com/100166083286297802191
 [14]: http://www.webpagetest.org/result/121124_V6_ART/1/details/
 [15]: http://www.cloudshark.org/captures/d3d390236f5d?filter=tcp.analysis.retransmission or tcp.analysis.duplicate_ack or tcp.analysis.lost_segment
 [16]: http://www.cloudshark.org/captures/d3d390236f5d?filter=tcp.analysis.retransmission
 [17]: http://www.cloudshark.org/captures/d3d390236f5d?filter=tcp.analysis.duplicate_ack
 [18]: http://www.cloudshark.org/captures/d3d390236f5d?filter=tcp.analysis.lost_segment

Hostname sharding is a hack to work around HTTP’s lack of parallelism, but here the number of connections is simply too much, and it looks like it probably causes the page to load more slowly than it could. Site owners need to be careful with the number of hostname shards they use, as the increased parallelism may not always improve performance. The potential for congestion when using multiple connections is also higher nowadays due to the increased adoption of [IW10][20]. The real solution of course is to switch to a prioritized multiplexed protocol like SPDY and send all resources over as few connections as possible.

 [20]: https://developers.google.com/speed/protocols/tcpm-IW10
