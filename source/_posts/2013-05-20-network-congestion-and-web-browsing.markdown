---
layout: post
title: "Network Congestion and Web Browsing"
date: 2013-05-20 12:17
comments: true
categories: 
   - tech
tags:
   - SPDY
   - HTTP/2
   - TCP
---
Recently, there was [discussion on the ietf-http-wg mailing list](http://lists.w3.org/Archives/Public/ietf-http-wg/2013AprJun/0003.html) about the current HTTP/2.0 draft's inclusion of a SPDY feature that allows the server to send its TCP [congestion window](http://en.wikipedia.org/wiki/Congestion_window) (often abbreviated as cwnd) to the client, so it can echo it back to the server when opening a new connection. It's a pretty interesting conversation so I figured I'd share the backstory of how we got to where we are today.

## <a id="TCP_Slow_Start">[TCP Slow Start](#TCP_Slow_Start)</a>

A long time ago in a galaxy far, far away (the 1980s), [the series of tubes got clogged](http://en.wikipedia.org/wiki/Congestive_collapse#Congestive_collapse). [Many personal Internets got delayed](http://en.wikipedia.org/wiki/Series_of_tubes#Partial_text_of_Stevens.27s_comments) and the world was sad. Van Jacobson was pretty annoyed that his own personal Internets were getting delayed too due to all this congestion, so he proposed that we all agree to [_avoid congestion_](http://ee.lbl.gov/papers/congavoid.pdf) rather than cause it.

{% img /images/2013/05/amitheonlyone_congestion.jpg %}

A key part of that proposal was [Slow Start](http://en.wikipedia.org/wiki/Slow-start), which called for using a congestion window and exponentially growing it until a loss event happens. This congestion window specified a limit on the number of TCP segments a sender could have outstanding at any point. The TCP implementation would initialize the connection's congestion window to a predefined "initial window" (often abbreviated as initcwnd or IW) and increase the window by 1 upon receipt of an ACK. If you got an ACK for each TCP segment you sent out, then in each roundtrip, the congestion window would double, thus leading to exponential growth. Originally, the initial congestion window was set to 1 segment, and eventually [RFC 3390](http://tools.ietf.org/html/rfc3390) increased it to 2-4. The congestion window would continue to grow until the connection encountered a loss event, which it assumes is due to congestion. In this way, TCP starts with low utilization and increases utilization (exponentially each roundtrip) until a loss event occurs.

<a href="http://www.mathcs.emory.edu/~cheung/Courses/455/Syllabus/A1-congestion/FIGS/slow-start.gif">{% img /images/2013/05/slow-start.gif %}</a>

## <a id="TCP_HTTP_and_the_Web">[TCP, HTTP, and the Web](#TCP_HTTP_and_the_Web)</a>

Way back when (the 1990s), [Tim Berners-Lee felt a burning need to see kitten photos](http://www.quickmeme.com/meme/3sg13r/), so he invented the World Wide Web, where web browsers would access kitten photo websites via HTTP over TCP. For a long time, the [recommendation was to only use two connections per host](http://www.w3.org/Protocols/rfc2616/rfc2616-sec8.html#sec8.1.4), in order to prevent kitten photo congestion in the tubes:

> Clients that use persistent connections SHOULD limit the number of
> simultaneous connections that they maintain to a given server. A
> single-user client SHOULD NOT maintain more than 2 connections with
> any server or proxy. A proxy SHOULD use up to 2\*N connections to
> another server or proxy, where N is the number of simultaneously
> active users. These guidelines are intended to improve HTTP response
> times and avoid congestion.

However, over time, people noticed that their kitten photos weren't loading as fast as they wanted them to. Due to this connection per host limit of 2, browsers were unable to send enough HTTP requests to receive enough responses to saturate the available downstream bandwidth. But users wanted more kitten photos faster and [incentivized browser vendors](https://bugzilla.mozilla.org/show_bug.cgi?id=423377) to [increase their connection per host limits](http://www.stevesouders.com/blog/2008/03/20/roundup-on-parallel-connections/) so they could download more kitten photos in parallel. Indeed, the [httpbis working group](http://tools.ietf.org/wg/httpbis/) recognized that this limit was way too low and [got rid of it](http://trac.tools.ietf.org/wg/httpbis/trac/ticket/131). Yet even this wasn't fast enough, and many users were still using older browsers, so website authors started [domain sharding](http://www.stevesouders.com/blog/2009/05/12/sharding-dominant-domains/) in order to send even more kitten photos in parallel to users.

{% img /images/2013/05/spiderman_connections.jpg %}

## <a id="Multiple_TCP_Connections_and_Slow_Start">[Multiple TCP Connections and Slow Start](#Multiple_TCP_Connections_and_Slow_Start)</a>

Now that we see that browsers have raised their parallelization limits, and websites sometimes use domain sharding to get even more parallelization, it's interesting to see how that interacts with TCP Slow Start. As previously noted, TCP Slow Start begins the congestion window with a conservative initial value and rapidly increases it until a loss event occurs. A key part to avoiding congestion is that the initial congestion window starts off below the utilization level that would lead to congestion. Note however, that if N connections simultaneously enter Slow Start, then the effective initial congestion window across those N connections is N\*initcwnd. Obviously for values of N and initcwnd high enough, congestion will occur immediately on network paths which cannot handle such high utilization, which may lead to poor [goodput](http://en.wikipedia.org/wiki/Goodput) (the "useful" throughput) due to wasting throughput on retransmissions. Of course, this begs the question how many connections and what initcwnd values we see in practice.

{% img /images/2013/05/family_guy_congestion.jpg %}

## <a id="Number_of_Connections_in_a_Page_Load">[Number of Connections in a Page Load](#Number_of_Connections_in_a_Page_Load)</a>

The Web has changed a lot since Tim Berners-Lee put up the first kitten website. Websites have way more images now, and may even have tons of third party scripts and stylesheets, not to mention third party ads. Fortunately, [Steve Souders](http://stevesouders.com/about.php) maintains the [HTTP Archive](http://httparchive.org/) which can help us figure out, for today's Web, [_roughly_ how many parallel connections may be required to load them](http://httparchive.org/trends.php#numDomains&maxDomainReqs).

<figure>
{% img /images/2013/05/http_archive_domains_requests.png %}
<figcaption>
As we can see, loading a modern webpage requires connecting to many domains and issuing many requests.
</figcaption>
</figure>

Looking at this diagram, we see that the average number of domains involved in a page load today is around 16. Depending on how the webpage is structured, we can probably expect to open upwards of tens of connections during the page load. This is of course only a very rough estimate, given this available data.

## <a id="Typical_initcwnd_values">[Typical Initial Congestion Window Values](#Typical_initcwnd_values)</a>

In their continuing efforts to [make the web fast](https://developers.google.com/speed/), Google researchers have figured out that removing throttles will make things go faster, so [they proposed raising the initial congestion window to 10](https://developers.google.com/speed/protocols/tcpm-IW10), and [it's on its way to becoming an RFC](http://www.ietf.org/id/draft-ietf-tcpm-initcwnd-08.txt). It's key to note why this makes a big difference to web browsing. Slow Start requires multiple roundtrips to increase the congestion window, but web browsing is interactive and bursty, so those roundtrips directly impact user perceived latency, and by the time TCP roughly converges on an appropriate congestion window for the connection, the page load most likely has completed.

{% img /images/2013/05/slow_start_aint_nobody_got_time_for_that.gif %}

Linux kernel developers found Google's argument compelling and have already [switched the kernel to using IW10 by default](http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=356f039822b8d802138f7121c80d2a9286976dbd). CDN developers were likewise pretty quick to figure out that if they wanted to compete in the make kitten photos go fast space, they needed to [raise their initcwnd values](http://www.cdnplanet.com/blog/initcwnd-settings-major-cdn-providers/) too to keep up.

{% img /images/2013/05/ishouldincreaseinitcwnd.jpg %}

## <a id="But_What_About_Slow_Start">[But What About Slow Start?](#But_What_About_Slow_Start)</a>

<figure>
{% img /images/2013/05/davem_tcp_cwnd.png %}
<figcaption>
As David Miller says, the <a href="http://vger.kernel.org/~davem/davem_ibm2010.pdf">TCP initial congestion window is a myth</a>.
</figcaption>
</figure>

As David Miller points out, any app _can_ increase the initial congestion window by just opening more connections, and obviously websites _can_ make the browser open more connections by domain sharding. So websites can effectively get as high an initial congestion window as they want.

{% img /images/2013/05/asian_father_iw_infinity.jpg %}

But the question is - does this happen in practice and what happens in that case? To answer this, I used a highly scientific methodology to identify representative websites on the Web, loaded them up in WebPageTest with a fairly "slow" (DSL) network setup, and looked to see what happened:

### <a id="Google_Kitten_Search">[Google Kitten Search](#Google_Kitten_Search)</a>

Here I examine my daily ["kittens" image search](https://www.google.com/search?q=kittens&tbm=isch) in [WebPageTest](http://www.webpagetest.org) and also [use CloudShark to do basic TCP analysis](http://cloudshark.org/captures/4f184b12288e?filter=tcp.analysis.retransmission).

<figure>
<a href="http://www.webpagetest.org/result/130417_4W_172N/">{% img /images/2013/05/Google_Image_Search_Kittens_Waterfall.png %}</a>
<figcaption>
Waterfall of the Google Kitten Search page load. Of note is the portion where the browser opens 6 connections in parallel to each of the sharded encrypted-tbn\[0-3\].gstatic.com hostnames, leading to severe congestion related slowdown during the SSL handshake (due to transmitting the SSL certificates) and the image downloads. In particular, the waterfall makes clear the increased latency in completing the SSL handshake, which directly adds to latency until image downloads begin, which increases the time until the user sees the images he/she is searching for.
</figcaption>
</figure>

<figure>
<a href="http://cloudshark.org/captures/4f184b12288e/graphs/new?filters=%21%28tcp.analysis.retransmission%29+%7Bgoodput%7D%2Ctcp.analysis.retransmission+%7Bretransmission%7D%2Ctcp.analysis.spurious_retransmission+%7Bspurious_retransmission%7D">{% img /images/2013/05/Google_Image_Search_Kittens_CloudShark.png %}</a>
<figcaption>
CloudShark analysis graph of goodput (which I've defined here as bytes that weren't retransmissions, using the !tcp.analysis.retransmission filter) vs retransmissions (tcp.analysis.retransmission) vs "spurious" retransmissions (tcp.analysis.spurious_retransmission - subset of retransmissions where the bytes had already previously been received). Retransmissions means the sender is having to waste bytes on repeated transmissions, but from the receiver's perspective, the bytes aren't necessarily wasted. Spurious retransmissions on the other hand are clearly wasteful from the receiver's perspective and increase the time until the document is completely loaded.
</figcaption>
</figure>

### <a id="Etsy">[Etsy](#Etsy)</a>

<figure>
<a href="http://www.webpagetest.org/result/130419_0Y_7P2/">{% img /images/2013/05/Etsy_Kittens_Connection_View.png %}</a>
<figcaption>
Etsy shards its image asset hostnames 4 ways (img[0-3].etsystatic.com), in addition to its other hostnames, opening up around 30~ connections in parallel.
</figcaption>
</figure>

<figure>
<a href="http://cloudshark.org/captures/988f7d27361a/graphs/new?filters=%21%28tcp.analysis.retransmission%29+%7Bgoodput%7D%2Ctcp.analysis.retransmission+%7Bretransmission%7D%2Ctcp.analysis.spurious_retransmission+%7Bspurious_retransmission%7D">{% img /images/2013/05/Etsy_Kittens_CloudShark.png %}</a>
<figcaption>
Etsy's sharding causes so much congestion related spurious retransmissions that it _dramatically_ impacts page load time.
</figcaption>
</figure>

## <a id="SPDY_initcwnd">[SPDY - Initial Congestion Window Performance Bottleneck](#SPDY_initcwnd)</a>

As shown above, plenty of major websites are using a significant amount of domain sharding, which can be highly detrimental for their user experience when the user has insufficient bandwidth to handle the burst from the servers. With so many connections opening up in parallel, the effective initial congestion window across all the connections is huge, and Slow Start becomes ineffective at avoiding congestion.

The advent of SPDY changes this since it helps fix HTTP/1.X bottlenecks and thus obviates the need for workaround hacks like domain sharding that increase parallelism by opening up more connections. However, after fixing the application layer bottlenecks in HTTP/1.X, SPDY runs into [transport layer bottlenecks, such as the initial congestion window](http://www.slideshare.net/mbelshe/spdy-tcp-and-the-single-connection-throttle/11), due to web browsing's bursty nature. SPDY is at a distinct disadvantage here compared to HTTP, since browsers will open up upwards of 6 connections per host for HTTP, whereas they'll only open a single connection for SPDY, therefore HTTP often gets 6 times the initial congestion window that SPDY does.

{% img /images/2013/05/spdy_http_cwnd.png %}

Given this knowledge, what should we be doing? Well, there's some argument to be made that since SPDY tries to be a good citizen and use fewer connections than HTTP/1.X, perhaps individual SPDY connections should get a higher initial congestion window than individual HTTP/1.X connections. One way do so is for the kernel to provide a socket option for the application to control the initial congestion window. Indeed, Google TCP developers [proposed this Linux patch](http://comments.gmane.org/gmane.linux.network/162103), but David Miller [shut the proposal down pretty firmly](http://permalink.gmane.org/gmane.linux.network/162107):

> Stop pretending a network path characteristic can be made into
> an application level one, else I'll stop reading your patches.
> 
> You can try to use smoke and mirrors to make your justification by
> saying that an application can circumvent things right now by
> openning up multiple connections.  But guess what?  If that act
> overflows a network queue, we'll pull the CWND back on all of those
> connections while their CWNDs are still small and therefore way
> before things get out of hand.
> 
> Whereas if you set the initial window high, the CWND is wildly out
> of control before we are even started.
> 
> And even after your patch the "abuse" ability is still there.  So
> since your patch doesn't prevent the "abuse", you really don't care
> about CWND abuse.  Instead, you simply want to pimp your feature.

{% img /images/2013/05/its_not_going_to_happen_app_cwnd.jpg %}

## <a id="SPDY_caching_cwnd">[SPDY - Caching the Congestion Window](#SPDY_caching_cwnd)</a>

Despite the rejection of this patch, Google's servers have been leveraging this kernel modification to experiment with increasing the initial congestion window for SPDY connections. Beyond just experimenting with statically setting the initial congestion window, Google has also experimented with using [SPDY level cookies](http://dev.chromium.org/spdy/spdy-protocol/spdy-protocol-draft2#TOC-SETTINGS) (see SETTINGS_CURRENT_CWND) to cache the server's congestion window at the browser for reuse later on when the browser re-establishes a SPDY connection to the server. This is precisely the functionality recently under debate in the httpbis working group.

It's easy to see why this would be controversial. It definitely does raise a number of valid concerns:

* Violates layering. Why is a TCP internal state variable being stored by the application layer?
* Allows the client to request the server use a specific congestion window value.
* Attempts to reuse an old, very possibly inaccurate congestion window value.
* Possibly increases the likelihood of overshooting the appropriate congestion window.

So why are SPDY developers even experimenting at all with this? Well, there are some more and less reasonable explanations:

* While it's true that this is clearly a layering violation, this is the only way to experiment with caching the congestion window for web traffic at reasonable scale. This could be conceivably done by the TCP layer itself with a TCP option, but given the rate of users updating to new operating system versions, that would take ages. Experimenting at the application layer, while ugly, is much more easily deployable in the short term.
* As with any variable controllable by the client, servers must be careful about abuse, treat this information as only advisory, and ignore ridiculous values.
* Yes, while it's true that there's definitely reason to be skeptical of an old congestion window value, a reasonable counterpoint question is, as [Patrick McManus mentions](http://lists.w3.org/Archives/Public/ietf-http-wg/2013AprJun/0107.html), is there a reason to believe that an uninformed static guess like IW10 is any better than an informed guess based on an old congestion window?
* Is reusing an old congestion window more likely to cause overshooting the appropriate congestion window? Well, given that in a HTTP/1.X world, browsers are often opening 6 connections per host, and thus combined with IW10 often have an effective initial congestion window per host of 60, it's interesting to note what values the SPDY CWND cookie caches in practice. Chromium (and Firefox too, according to Patrick McManus), sees median values around 30~, so in many ways, reusing the old congestion window is more conservative than what happens today with multiple HTTP/1.X connections.

It's great though to see this get discussed in IETF, and especially great to see tcpm and transport area folks get involved in the HTTP related discussions. And it looks like there's increasing [interest from these parties](http://lists.w3.org/Archives/Public/ietf-http-wg/2013AprJun/0475.html) in collaborating more closely in the future, so I'm hopeful that we'll see more research in this area and come up with some better solutions here.

## <a id="Chromium_Strikes_Back">[Chromium Strikes Back](#Chromium_Strikes_Back)</a>

Until SPDY and HTTP/2 are widely deployed enough that web developers don't need to rely on domain sharding, and we're still years out from this, we have to make things work well in today's web. And that means dealing with domain sharding. Even though domain sharding may make sense for older browsers with low connection per host limits, when domain sharding is combined with higher limits, then opening so many connections in parallel can actually be harmful to both performance and network congestion as previously demonstrated.

{% img /images/2013/05/browser_connections_lumbergh.jpg %}

As I've [previously discussed](/tech/connection-management-in-chromium/#how_much_parallelism_is_good), Chromium historically has had poor control over connection parallelism. That said, now we have our fancy new ResourceScheduler, courtesy of my colleague James Simonsen. With that, we now have page level visibility into all resource requests. With that capability, we can start placing some limits on request parallelism on a per-page basis. Indeed, we've experimented with doing so and found [compelling data to support limiting the number of concurrent image requests to 10](https://chromiumcodereview.appspot.com/12874003), which should roll out to stable channel in the upcoming Chrome 27 release and dramatically mitigate congestion related issues. For further details, [look at the waterfalls here](http://www.webpagetest.org/video/compare.php?tests=130517_CN_5bb5fc609a54e9e08aa8050843bd0d69-r:1-c:0,130517_WM_08e57463a955d1713bc7e4134bd6ff37-r:1-c:0) and watch the [video comparing the page loads](http://www.webpagetest.org/video/view.php?id=130517_946b0aadbccb2349e262f7b468c7ce90d8bbe88c).

<figure>
<a href="http://cloudshark.org/captures/a0a5a7105940/graphs/new?filters=%21%28tcp.analysis.retransmission%29+%7Bgoodput%7D%2Ctcp.analysis.retransmission+%7Bretransmission%7D%2Ctcp.analysis.spurious_retransmission+%7Bspurious_retransmission%7D">{% img /images/2013/05/Chrome_26_Etsy_Kittens_CloudShark.png %}</a>
<figcaption>
Chrome 26 demonstrates a fair amount of congestion related retransmissions.
</figcaption>
</figure>

<figure>
<a href="http://cloudshark.org/captures/1649861da844/graphs/new?filters=%21%28tcp.analysis.retransmission%29+%7Bgoodput%7D%2Ctcp.analysis.retransmission+%7Bretransmission%7D%2Ctcp.analysis.spurious_retransmission+%7Bspurious_retransmission%7D">{% img /images/2013/05/Chrome_29_Etsy_Kittens_CloudShark.png %}</a>
<figcaption>
Chrome 29 demonstrates that the image parallelism change dramatically improves the situation here.
</figcaption>
</figure>

{% img /images/2013/05/vader_limit.jpg %}

While this limit would obviously be good for users with slower internet connections, I was minorly concerned that it would lead to degraded performance for users with faster internet connections due to not being able to fully saturate the connection's available bandwidth. That said, it appears that it is still [high enough for at least cable modem type bandwidth](http://www.webpagetest.org/video/compare.php?tests=130518_XC_fd76b453d1f9373cc21741fc4e1ef558,130518_QF_7afffba181911e1f71ae12482a15240e-r:1-c:0).

## <a id="Conclusion">[Conclusion](#Conclusion)</a>

###In summary:

* Congestion is bad for the internet and for user perceived latency
* Extra roundtrips are bad for user perceived latency
* Slow Start helps avoid congestion
* Slow Start can incur painful roundtrips
* Low initial congestion windows can be bottlenecks for user perceived latency
* But high initial congestion windows can lead to immediate congestion, which can also hurt user perceived latency
* There's an interesting discussion in IETF on whether or not we can improve upon Slow Start by picking better initial congestion windows, possibly dynamically by using old information.
* Web developers using high amounts of domain sharding to work around low connection per host limits in old browser should reconsider their number of shards for newer browsers. Anything over 2 is probably too much, unless most of your user base is using older browsers. Better yet, stop hacking around HTTP/1.X deficiencies and use SPDY or HTTP/2 instead.

###Food for thought (and possibly future posts):

* Impact on [bufferbloat](http://en.wikipedia.org/wiki/Bufferbloat)
* How does TCP behave exactly when it encounters such high initial congestion like some of the demonstrated packet traces show? How does this change with the advent of newer TCP algorithms like [Proportional Rate Reduction](http://research.google.com/pubs/pub37486.html) and [Tail Loss Probe](http://tools.ietf.org/html/draft-dukkipati-tcpm-tcp-loss-probe-01) that are available in newer Linux kernel versions?
* Limiting requests will reduce attempted bandwidth utilization, which reduces congestion risk (and the risk for precipitous drops in goodput under excessive congestion). How else does it improve performance? For hints, see my some of my old posts like [this](/tech/resource-prioritization-in-chromium/) and [this](/tech/throttling-subresources-before-first-paint/).
