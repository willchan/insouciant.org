---
layout: post
title: "HTTP/2 Considerations and Tradeoffs"
date: 2014-01-10 18:26
comments: true
categories: 
  - tech
tags:
  - IETF
  - SPDY
  - HTTP/2
---
_\[Disclaimer: The views expressed in this post are my own, and do not represent those of my employer.\]_

_\[Disclaimer: I'm a Chromium developer working on HTTP/2 amongst other things.\]_

There are many things to love about HTTP/2, but there are also many things to hate about it. It improves over HTTP/1.X in a number of ways, yet it makes things worse in a number of ways as well. When proposing something new, it's important to try to comprehensively examine the tradeoffs made, to evaluate if it makes sense overall. I'm going to try (and probably fail) to provide a reasonably comprehensive description of the important considerations & tradeoffs involved with HTTP/2. Warning: this is going to be a long post, but it's required in order to try to present a more complete analysis of HTTP/2's tradeoffs.

Oh, and if you don't know what HTTP/2 is, it's the in-progress next major version of HTTP (the latest version is 1.1), based on Google's [SPDY](http://en.wikipedia.org/wiki/SPDY) protocol. It keeps the same HTTP message semantics, but uses a new connection management layer. For a basic introduction to SPDY, I recommend reading the [whitepaper](http://www.chromium.org/spdy/spdy-whitepaper) or Ilya's [post](http://www.igvita.com/2011/04/07/life-beyond-http-11-googles-spdy/). But at a high level, it uses:

* Muliplexed streams (each one roughly corresponding to a HTTP request/response pair)
* Stream prioritization (to advise the peer of the stream's priority relative to other streams in the connection)
* Stateful (HTTP) header compression
* Secure transport (TLS) in all browsers that support SPDY (it's not required by spec though, just by current browser implementations)

I'm not going to bother explaining anymore, since there are plenty of great descriptions of HTTP/2 out there, so if you want to know more, use your favorite search engine. Let's dive into the considerations.

## <a name="Considerations">[Major Considerations:](#Considerations)</a>

* [Network performance](#NetworkPerformance)
* [Scalability & DoS](#ScalabilityDoS)
* [Implementation complexity](#ImplementationComplexity)
* [Text vs Binary](#TextBinary)
* [Deployability](#Deployability)
* [TLS & Privacy](#TLSPrivacy)

### <a name="NetworkPerformance">[Network Performance](#NetworkPerformance)</a>

HTTP/1.X is very inefficient with its network usage. Excepting [pipelining](http://en.wikipedia.org/wiki/HTTP_pipelining), [which has its own issues](/tech/status-of-http-pipelining-in-chromium/), HTTP/1.X only allows one transaction per connection at any point in time. This causes major head of line blocking issues that costs expensive roundtrips, which is the [dominant factor (in terms of networking) of page load performance](http://www.belshe.com/2010/05/24/more-bandwidth-doesnt-matter-much/).

{% img /images/2014/01/PLT_RTT.png %}

{% img /images/2014/01/PLT_BW.png %}

If page load performance matters, then in terms of networking protocol design, reducing roundtrips is the way to go, since reducing the actual latency of a roundtrip is very hard (the speed of light does not show signs of increasing, so the only remaining option is to pay for more servers closer to the clients and solving the associated distributed systems problems). There are some HTTP/1.X workarounds ([hostname sharding](http://www.stevesouders.com/blog/2009/05/12/sharding-dominant-domains/), [resource inlining](https://developers.google.com/speed/pagespeed/service/InlineSmallResources), [CSS sprites](http://alistapart.com/article/sprites), etc), but they are all worse than supporting prioritized multiplexing in HTTP/2. Some reasons why include:

* Hostname sharding incurs DNS lookups (more roundtrips to DNS servers) for each hostname shard
* Hostname sharding ends up opening more connections which:
	* [Increases contention](/tech/throttling-subresources-before-first-paint/) since there's no inter-connection prioritization mechanism
	* [Can lead to network congestion](/tech/network-congestion-and-web-browsing/) due to multiplying the effective cwnd
	* Incurs more per-connection overhead in intermediaries, servers, etc.
	* Requires waiting for each connection to open, rather than for just one connection to open (and multiplex all the requests on that one connection)
* Resource inlining, CSS sprites, etc. are all forms of resource concatenation which:
	* Prevents fine grained caching of resources, and may even outright prevent caching (if you're inlining into uncacheable content like most HTML documents)
	* Bloats resources, which delays their overall download time. Many resources must be downloaded in full before the browser can begin processing them. Inlining images as data URIs in CSS can hurt performance because documents can't render before they download all external stylesheets in <head>.
	* Can interfere with resource prioritization. Theoretically you want lower priority resources (like images) downloaded later rather than be inlined in the middle of a high priority resource (like HTML).
* These techniques all require extra work/maintenance for the web developer, so only websites with developers who know these techniques and are willing to put up with the extra maintenance burden will actually employ them. HTTP/2 makes the simple, natural way of authoring web content just work fast, so the benefits accrue to the entire web platform, not just the websites with engineers who know how to hack around issues.

Another important aspect of network performance is optimizing for the mobile network, where roundtrips are even more costly, and the uplink bandwidth is even more constrained. Header compression makes the per-request overhead cheap, since lots of the overhead manifests in the form of large HTTP headers like cookies. Indeed, HTTP's [per-request overhead](https://developers.google.com/speed/docs/best-practices/request) is costly enough that web performance advocates recommend using [fewer requests](http://developer.yahoo.com/blogs/ydn/high-performance-sites-rule-1-fewer-http-requests-7163.html). Where this really can kill you is in [the roundtrips required to grow the client-side TCP congestion window](http://lists.w3.org/Archives/Public/ietf-http-wg/2012JulSep/1096.html) as Patrick McManus from Firefox explains. Indeed, he goes as far as to say that it's effectively [necessary in order to reach sufficient parallelization levels](http://lists.w3.org/Archives/Public/ietf-http-wg/2013JulSep/0268.html):

> Header compression on the upstream path is more or less required to enable
> effective  prioritized mux of multiple transactions due to interactions
> with TCP congestion control. If you don't have it you cannot effectively
> achieve the parallelism needed to leverage the most important HTTP/2
> feature with out an RTT penalty. and RTT's are the enemy.
> 
> Without compression you can "pipeline" between 1 and 10 requests depending
> on your cookie size. Probably closer to 3 or 4. With compression, the sky
> is more or less the limit.

For more details about HTTP/2 networking performance concepts, check out my colleague Ilya's wonderful [talk](https://www.youtube.com/watch?v=SWQdSEferz8) and [slidedeck](http://www.igvita.com/slides/2012/http2-spdy-devconf.pdf).

On the flip side, browsers that currently deploy SPDY only do so over TLS for various reasons (see the later [deployability](#Deployability) and [TLS](#TLSPrivacy) sections), and that TLS handshake will generally incur at least an extra 1-2 roundtrips. Moreover, to the degree that webpages load resources from different domains, HTTP/2 will be unable to multiplex requests for those resources over the same connection, and the browser will instead have to open up separate connections.

Now, that's the theory of course behind most of the theoretical networking performance improvements offered by HTTP/2. But there's also some debate over how well it performs in practice. That topic is beyond of the scope of this post, and indeed, many posts have been written about this topic. Guy wrote a [pretty good post identifying issues with SPDY / HTTP/2 performance on unmodified websites](http://www.guypo.com/technical/not-as-spdy-as-you-thought/). When reading that, it's also important to read [Mike's counterpost](http://www.belshe.com/2012/06/24/followup-to-not-as-spdy-as-you-thought/) where he critiques Guy's first party domain classifier. TL;DR: Guy points out that if web sites don't change, they won't see much benefit since resources are loaded across too many different hostnames, but Mike points out that lots of the hostnames are first party (belonging to the website owner), so in a real deployment, they would be shared over the same HTTP/2 connections.

And then it's important to note that Google, [Twitter](http://lists.w3.org/Archives/Public/ietf-http-wg/2012JulSep/0250.html), and [Facebook](http://lists.w3.org/Archives/Public/ietf-http-wg/2012JulSep/0251.html) (notably all sites that are already primarily HTTPS, so they aren't paying any additional TLS penalty for switching) all deployed SPDY because of its wins. If the website is already using TLS, then deploying to SPDY is a clear win from a page load time improvement perspective. One particularly exciting result though that Google [announced in a blog post](http://googledevelopers.blogspot.com/2012/01/making-web-speedier-and-safer-with-spdy.html) is that when they switched from non-SSL search to SSL search, SPDY-capable browsers actually loaded the search results page even faster. Another result from Google+ is that [leveraging SPDY prioritization dramatically sped up their page loads](https://plus.google.com/104404461680012191607/posts/Uw87yxQFCfY).

Putting aside the page load performance aspect of network performance for now, let's consider the other way in which HTTP/2 may affect network performance: real-time networking latency. For a long time now, folks have been raising a big fuss over [bufferbloat](http://www.bufferbloat.net/projects/bloat/wiki/Introduction), and with good reason! These bloated buffers are leading to huge queueing delays for everything going through these queues (located in routers, switches, NICs, etc). This is why when someone is uploading or downloading a whole lot of data, like video download/upload, it fills these huge buffers, which massively increases the queueing delays and kills the interactivity of real-time applications like Skype and videoconferencing which depend on low latency. [Web browsing tends to be another one of these bad contributors to network congestion](/tech/network-congestion-and-web-browsing/) due to websites using so many HTTP/1.X connections in the page load. As [Patrick McManus observes](http://bitsup.blogspot.com/2011/12/spdy-bufferbloat-http-and-real-time.html), HTTP/2's multiplexing will lead to a reduction of connections a browser will have to use to load a page. Fewer connections will both decrease the overall effective cwnd to something more reasonable and increase the likelihood that congestion related packet loss signals will affect the transmission rate, leading to less queue buildup. HTTP/2 is a key piece in the overall incentives so that web developers don't have to increase the connection count in order to get sufficient parallelization.

I've primarily discussed networking performance from a browser page load performance here, but the same principles apply to [non-browser use cases too](http://lists.w3.org/Archives/Public/ietf-http-wg/2013OctDec/1825.html). As Martin Thomson (HTTP/2 editor) says:

> I don't think that I'm alone in this, but the bulk of my day job at
> Skype was building those sorts of systems with what you call "RESTful
> APIs".  Having HTTP/2.0 was identified as a being hugely important to
> the long term viability of those systems.  The primary feature there
> was multiplexing (reducing HOL blocking is a big win), but we did also
> identify compression as important (and potentially massively so).  We
> were also speculatively interested in push for a couple of use cases.

Indeed, Twitter even [provides us with performance data](https://blog.twitter.com/2013/cocoaspdy-spdy-for-ios-os-x) for its API users who experienced significant performance improvements when using SPDY:


{% img https://g.twimg.com/blog/blog/image/chart-low.png %}

{% img https://g.twimg.com/blog/blog/image/chart-high.png %}

### <a name="ScalabilityDoS">[Scalability & DoS](#ScalabilityDoS)</a>

Another major HTTP concern relates to scalability and DoS. The working group is very sensitive to these issues, especially because a large chunk of the active participants of the httpbis working group represent intermediaries / large services (e.g. Akamai, Twitter, Google, HAProxy, Varnish, Squid, Apache Traffic Server, etc). HTTP/2 has a number of different features/considerations that influence scalability:

* Header compression - very controversial
* Multiplexing - not controversial at all from a scalability standpoint, widely considered a Good Thing (TM)
* Binary framing - not controversial at all from a scalability standpoint, widely considered a Good Thing (TM)
* TLS - controversial

To get an understanding of scalability & DoS concerns, it's useful to see what some of the high scalability server folks have to say here. [Willy Tarreau](http://1wt.eu/#wami) of [HAProxy](http://haproxy.1wt.eu/) has written / talked extensively about these issues. From Willy's [IETF83 httpbis presentation slides](http://www.ietf.org/proceedings/83/slides/slides-83-httpbis-0.pdf):

[Slide 2](http://www.ietf.org/proceedings/83/slides/slides-83-httpbis-0.pdf#page=2):
> Intermediaries have a complex role :
>
> * must support unusual but compliant message formating (eg: variable case in header names, single LF, line folding, variable number of spaces between colon and field value)
> * fix what ought to be fixed before forwarding (eg: multiple content-length and folding), adapt a few headers (eg: Connection)
> * must not affect end-to-end behaviour even if applications rely on improper assumptions (effects of rechunking or multiplexing)
> * need to maintain per-connection context as small as possible in order to support very large amounts of concurrent connections
> * need to maintain per-request processing time as short as possible in order to support very high request rates
> * front line during DDoS, need to take decisions very quickly

[Slide 4](http://www.ietf.org/proceedings/83/slides/slides-83-httpbis-0.pdf#page=4):
> Intermediaries would benefit from :

> * Reduced connection/requests ratio (more requests per connection)
> 	* drop of connection rate
> 	* drop of memory footprint (by limiting concurrent conns)
> * Reduced per-request processing cost and factorize it per-connection
> 	* higher average request rate
> 	* connection setup cost is already "high" anyway
> * Reduced network packet rate by use of pipelining/multiplexing
> 	* reduces infrastructure costs
> 	* significantly reduces RTT impacts on the client side

As Willy discusses in his [HTTP/2 expression of interest for HAProxy](http://lists.w3.org/Archives/Public/ietf-http-wg/2012JulSep/0179.html), he's overall favorably inclined towards the SPDY proposal, which became the starting point for HTTP/2. Prefixing header name/values with their size makes header parsing simpler and more efficient. And the big win really is multiplexing, since you reduce the number of concurrent connections, and lots of the memory footprint is per connection. Also, as can be seen in [HAProxy's homepage](http://haproxy.1wt.eu/#perf), a large majority of the time is spent in the kernel. By multiplexing multiple transactions onto the same connection, you reduce the number of expensive syscalls you have to execute, since you can do more work per syscall. Header compression is a key enabler of high levels of parallelism while multiplexing, so it helps enable doing more work per syscall. On the flip side, header compression is also the major issue Willy has with the original SPDY proposal:

> For haproxy, I'm much concerned about the compression overhead. It requires
> memory copies and cache-inefficient lookups, and still maintains expensive
> header validation costs. For haproxy, checking a Host header or a cookie
> value in a compressed stream requires important additional work which
> significantly reduces performance. Adding a Set-Cookie header into a
> response or remapping a URI and Location header will require even more
> processing. And dealing with DDoSes is embarrassing with compressed
> traffic, as it improves the client/server strength ratio by one or two
> orders of magnitude, which is critical in DDoS fighting.

That said, that's in reference to SPDY header compression which used zlib. The new header compression proposal ([HPACK](http://tools.ietf.org/html/draft-ietf-httpbis-header-compression-03)) is different in that it is [CRIME](http://en.wikipedia.org/wiki/CRIME_(security_exploit\))-resistant, by not being stream-based, and instead relying on delta encoding of header key-value pairs. It is notably still stateful, but allows the memory requirement to be bounded (and even set to 0, which effectively disables compression). This latter facility is key, because if header compression does pose significant enough scalability or DDoS concerns, it can definitely be disabled. There is a long, complicated, somewhat dated [thread](http://lists.w3.org/Archives/Public/ietf-http-wg/2013JulSep/0246.html) discussing it, which is highly educational. I think it's safe to say that the feature is still fairly controversial, although there's definitely a lot of momentum behind it.

There have been alternate proposals for header encoding, mostly based around [binary/typed coding](http://datatracker.ietf.org/doc/draft-snell-httpbis-bohe/). Many people hope that this is enough, and it removes the requirement for stateful compression, but many fear that it is not enough due to huge, opaque cookie blobs. I do not mention it further here since while these proposals have merit, they don't seem to generate sufficient interest/discussion for the working group at large to want to pursue them. Although recently someone expressed interest again.

And [Varnish](https://www.varnish-cache.org/) maintainer [Poul-Henning Kamp](http://en.wikipedia.org/wiki/Poul-Henning_Kamp) has been especially [critical of HTTP/2](https://www.varnish-cache.org/docs/trunk/phk/http20.html) on the scalability / DDoS prevention front. Indeed, one of his slides from his RAMP presentation ["HTTP Performance is a solved problem"](http://www.infoq.com/presentations/HTTP-Performance) says it best:

> HTTPng HTTP/2.0 –

> * ”Solves” non-problems (bandwidth)
> * Ignores actual problems (Privacy, DoS, speed)
> * Net benefit: At best, very marginal
> * -> Adoption: Why bother ?

PHK is well known for being a bit hyperbolic, so rather than address the actual words he writes here, I'll interpret the general meaning being that we aren't doing _enough_ to make HTTP more scalable (computationally performant, reduced memory consumption, and DDoS resistant). He puts forth a number of concrete proposals for things we could do better here:

> What does a better protocol look like ?

> * Flow-label routing
> * Only encryption where it matters
> * Non-encrypted routable envelope
> * Fixed-size, fixed order fields
> * No dynamic compression tables

Notably HPACK runs against a number of these. You can't simply do flow-label (subset of headers represents a flow that always gets routed to backend X) routing since HPACK requires "processing" all headers, and the order is not fixed. HPACK has a number of advantages, but PHK's primary counterpoint is that most of those advantages come in a world where cookies comprise the largest portion of HTTP headers, and cookies are simply evil. PHK would love to see us kill cookies in HTTP/2 and replace them with tiny session IDs. Lots of his rationale here comes from political/legal opinions about cookies and user tracking / privacy, which I'm not going to dwell on. But he makes an interesting point that perhaps the main thing HPACK fixes is redundant sending of large cookies, and he'd like us to kill them off and replace them with session ids. As [Willy points out](http://lists.w3.org/Archives/Public/ietf-http-wg/2013JulSep/0287.html), this is actually contrary to overall scalability due to requiring distributed server-side session management syncing across machines/datacenters. This is obviously controversial because if we want to encourage servers to adopt HTTP/2, we need to provide incentives to do so, and breaking backwards compatibility with cookies and requiring deploying distributed data stores is likely to make most companies question the wisdom of switching to HTTP/2.

Now, the other major controversial issue from a scalability standpoint is TLS, and the reasons are fairly obvious. It incurs extra buffer copies for symmetric encryption/decryption, expensive computation for the asymmetric crypto used in the handshake, and extra session state. Going into detail on the exact cost here is beyond the scope of this post, but you can read up about it on the internets: [Adam Langley on SSL performance at Google](https://www.imperialviolet.org/2010/06/25/overclocking-ssl.html) & Vincent Bernat's [two](http://vincent.bernat.im/en/blog/2011-ssl-benchmark.html) [posts](http://vincent.bernat.im/en/blog/2011-ssl-benchmark-round2.html) on the performance of common SSL terminators.

Another obvious scalability issue with encryption is sometimes intermediaries want to improve scalability (and latency) by doing things like caching or content modification like [image&video transcoding/downsampling/etc](http://www.macworld.com/article/1157635/verizon_throttle_iphone_data.html)). However, if you can't inspect the payload, you can't cache, which means caching intermediaries are not viable unless they are able to decrypt the traffic. This clearly has some amount of internet scalability / latency concerns, especially in the developing world and other places further away from the web servers, like Africa and Australia.

Lastly, TLS faces some scalability issues due to IPv4 address space scarcity. With HTTP, webservers can reduce their IP address utilitization by using [virtual hosting](http://en.wikipedia.org/wiki/Virtual_hosting#Name-based). That same virtual hosting method doesn't [work for HTTPS without SNI support](http://en.wikipedia.org/wiki/Server_Name_Indication#Background_of_the_problem), meaning that as long as a [significant user base doesn't support SNI](http://en.wikipedia.org/wiki/Server_Name_Indication#Browsers_with_support_for_TLS_server_name_indication.5B6.5D) (e.g. IE users on XP and Android browser pre-Honeycomb), servers that don't want to break those users will need to use separate VIPs per hostname. Given IPv4 address scarcity, and the insufficient deployment of IPv6, TLS can present [additional scalability issues in terms of address space availability](http://bugs.python.org/issue5639#msg192233).

Multiplexing is a big win for scalability, but stuff like header compression is more ambiguous, and TLS clearly imposes scalability costs. There hasn't been much public release of numbers here, except for this [one article from Neotys](http://www.neotys.com/blog/performance-of-spdy-enabled-web-servers/) which concludes with:

> It’s no surprise that SPDY improves response times on the client side. That’s what it was designed to do. It turns out that SPDY also has advantages on the server side:
>
> * Compared to HTTPS, SPDY requests consume less resources (CPU and memory) on the server.
> * Compared to HTTP, SPDY requests consume less memory but a bit more CPU. This may be good, bad, or irrelevant depending on which resource (if either) is currently limiting your server.
> * Compared to HTTP/S, SPDY requires fewer Apache worker threads, which increases server capacity. As a result, the server may attract more SPDY traffic.

### <a name="ImplementationComplexity">[Implementation Complexity](#ImplementationComplexity)</a>

A lot has been said about SPDY and HTTP/2's implementation complexity. I think it's pretty useful to see what actual implementers have to say about the matter, since they have real experience:

* Patrick McManus discusses the binary framing decision [here](http://blog.jgc.org/2012/12/speeding-up-http-with-minimal-protocol.html#c5703739431744738432), where he explains why binary is so much simpler and more efficient than ASCII for SPDY / HTTP/2.
* On the flip side, Jamie Hall, who had worked on SPDY for his undergraduate dissertation, discusses his SPDY/3 implementation [here](https://groups.google.com/forum/#!msg/spdy-dev/aA69aSj8Fmc/7PrPrBhq5G0J). Notably, he says that most things were fine, but that header compression and flow control were a little complicated.
* [Jesse Wilson](https://twitter.com/jessewilson) of Square also wrote up [his feedback on HTTP/2 header compression](http://lists.w3.org/Archives/Public/ietf-http-wg/2013JulSep/1004.html). He has a lot of substantive "nitpicks" about the "rough edges", but says that "Overall I’m very happy with the direction of HPACK and HTTP/2.0."
* [Adrian Cole](https://twitter.com/adrianfcole) of Square also wrote his [thoughts on HPACK](http://lists.w3.org/Archives/Public/ietf-http-wg/2014JanMar/0067.html), saying: "I found that implementing this protocol, while not trivial, can be done very efficiently." and "All in, HPACK draft 5 has been quite enjoyable to develop. Thanks for the good work."
* [Stephen Ludin of Akamai also noted that](http://lists.w3.org/Archives/Public/ietf-http-wg/2013JulSep/0292.html):

> I just implemented the header compression spec and it felt incredibly
> complex so I am definitely inclined to figure out a scheme for
> simplification.  My concern is similar to Mike's: we will get buggy
> implementations out there that will cause us to to avoid its use in the
> long run.  The beauty of the rest of the HTTP/2.0 spec is its simplicity
> of implementation.  I like to think we can get there with header
> compression as well.

* [James Snell](https://twitter.com/jasnell) from IBM has participated heavily in the working group and has written up a number of good, [detailed](http://www.chmod777self.com/2013/07/http2-status-update.html) [blog](http://www.chmod777self.com/2013/07/http2-wait-theres-more.html) [posts](http://www.chmod777self.com/2013/07/http2-wait-theres-more.html) highlighting complexity in the HTTP/2 draft specs. He spends the vast majority of the time criticizing the complexity, so people might think that he's overall negative on HTTP/2, but they miss the point that he's soundly in favor of the core parts of HTTP/2 - binary framing and multiplexing. As he says:

> Many people who have responded to my initial post have commented to the effect that switching to a binary protocol is a bad thing. I disagree with that sentiment. While there are definitely things I don't like in HTTP/2 currently, I think the binary framing model is fantastic, and the ability to multiplex multiple requests is a huge advantage. I just think it needs to be significantly less complicated.

Overall, I think it goes without saying that everyone agrees there is definitely more complexity in HTTP/2, especially around header compression and state management. Some people have incorrectly viewed the binary framing as a source of implementation complexity. This is simply false as [Patrick McManus goes to great detail to demonstrate](http://blog.jgc.org/2012/12/speeding-up-http-with-minimal-protocol.html#c5703739431744738432). Indeed, if you go look at the [actual implementations out there](https://github.com/http2/http2-spec/wiki/Implementations), the binary framing is some of the simplest, most straightforward code. But there are plenty of areas that are clearly more complicated, with multiplexing, header compression, and server push clearly among them. For a discussion of the complexities of header compression, one has only to look at Jesse Wilson's [email to httpbis](http://lists.w3.org/Archives/Public/ietf-http-wg/2013JulSep/1004.html) and James Snell's [blog post on header compression](http://www.chmod777self.com/2013/08/http2-header-compression.html) to get an idea of the complexities involved.

Header compression used to be significantly less complicated, practically speaking, when [SPDY used zlib for header compression](http://dev.chromium.org/spdy/spdy-protocol/spdy-protocol-draft3#TOC-2.6.10.1-Compression). When you don't need a [whole, new, separate spec for header compression](https://github.com/http2/compression-spec), and can instead rely on [an existing standard](http://www.ietf.org/rfc/rfc1950.txt) that has widely deployed [open source implementations](http://www.zlib.net/), header compression adds significantly less implementation complexity. That said, [CRIME](http://en.wikipedia.org/wiki/CRIME_(security_exploit\)) rendered zlib insecure for use in HTTP header compression, leaving us with the burden of added complexity for implementing a new header compression algorithm if we still want header compression.

Putting header compression aside, another big chunk of the complexity in HTTP/2 is due to stream multiplexing. Now, multiplexing sounds simple, and conceptually it is. You tag frames with a stream id, big deal. And now each stream is its own, _mostly_ independent unit with its own moderately sized [state machine](http://http2.github.io/http2-spec/#StreamStates), which definitely isn't trivial, but is quite understandable. But the simple act of introducing multiplexing streams leads to other issues. Many of them have to do with handling races and keeping connection state synchronized. None of these are hard per se, but they increase the number of edge cases that implementers need to be aware of.

One of the bigger implications of multiplexing is the necessity of HTTP/2 level flow control. Previously, with HTTP/1.X, there was only ever one stream of data per transport connection. This means that the transport's flow control mechanisms were by themselves sufficient. If you ever ran out of memory for buffers, you just stopped reading from the transport connection socket, and the transport flow control mechanism kicked in. However, with multiplexed streams, [things got more complicated](https://docs.google.com/document/d/15LMnvVWSY-kF-ME4RZIFHN0FukViapjiefQMmQCnr14/pub). TL;DR: Fundamentally, the issue is that not reading from the socket will block _all_ streams, even if only one stream has buffering issues. Flow control has non-trivial complexity, and naive implementations could definitely kill their performance by keeping the windows too small. Indeed, the spec actually [recommends disabling flow control](http://http2.github.io/http2-spec/#DisableFlowControl) on the receiver end unless you absolutely need the control provided by flow control.

Another controversial part of HTTP/2, primarily due to its complexity, is server push. HTTP is traditional a request/response protocol, so having the server push responses clearly complicates the stream state model, not to mention introduces the possibility for races which the protocol would have to address. There is a lot more to say about server push, but it clearly adds additional complexity to HTTP/2 and for that reason amongst others, its status within the draft spec has always been rather shaky. The primary counterpoint to it is that servers are already using [existing, but suboptimal techniques](/tech/server-push-already-exists-but-spdy-server-push-is-better/) (most notably [inlining small resources](https://developers.google.com/speed/pagespeed/service/InlineSmallResources)) to "push" responses, so server push would just provide a better, protocol-based solution.

I could go on about the other complexities of HTTP/2, but I think it's fair to say that it's clearly nontrivially more complicated than HTTP/1.X. It's not rocket science though, and this complexity is clearly quite tractable, given the wide availability of both SPDY and [HTTP/2 implementations](https://github.com/http2/http2-spec/wiki/Implementations).

### <a name="TextBinary">[Text vs Binary](#TextBinary)</a>

One of the most noticeable changes with HTTP/2 is that it changed from a text based protocol to a binary protocol. There are a lot of tradeoffs between textual and binary protocols, and a lot of them have been written up before. Stack Overflow has a great [thread](http://stackoverflow.com/questions/2525188/are-binary-protocols-dead) on some of the tradeoffs, and ESR also [wrote about it](http://www.catb.org/esr/writings/taoup/html/ch05s01.html) in The Art of Unix Programming.

In the specific HTTP case, there are lots of wins to binary. I've already covered them earlier in the scalability and implementation complexity sections. Binary framing with length indicators makes parsing way more efficient and easy. The interesting thing to discuss here is the downsides, first and foremost of which is ease of use/understanding.

Binary sucks in terms of ease of use and understanding. In order to handle the binary, you'll most likely have to pipe stuff through a library or tool to process it. No more straight up telnet. No more human readable output by just sniffing the bytes off the network. This obviously sucks. One of the best things about the web has been its low barrier of entry - the ease of understanding by just looking at what's actually getting sent over the wire, and testing things out with telnet.

That said, it's interesting to think about how things will change in a HTTP/2 world, especially one that's sitting atop TLS. How are people poking around today? Are they running tcpdump and examining the raw bytes? Are they using telnet and netcat and what not? My gut instinct, which could be very wrong, is that most people are using the browser web developer tools. HTTP/2 may introduce wire level changes, but the HTTP semantics are unchanged, and you'll still see the same old HTTP request/response pairs in the developer tools, emitted to and parsed from the wire in HTTP/2 binary format for you by the browser. Server-side, the webservers that support HTTP/2 will likewise handle all the parsing and generating of HTTP/2 frames into HTTP messages, and will likely emit the same logs for debugging purposes. For people who actually examine the network traffic, they're probably using Wireshark, which [already has basic support for HTTP/2 draft versions](http://lists.w3.org/Archives/Public/ietf-http-wg/2013JulSep/1100.html) and can [decrypt SSL sessions](http://security.stackexchange.com/questions/35639/decrypting-tls-in-wireshark-when-using-dhe-rsa-ciphersuites/42350#42350) when the pre-master secret is provided.

It's difficult for me to say how much binary will impact ease of use and understanding. It's not clear to me how often people care about the actual wire format, rather than the HTTP request/response messages, which will still be good ol text. Who's to say?

Moreover, depending on whether or not you're sold on HTTP/2 being over TLS everywhere, then a good part of the binary vs text discussion may already be decided. If you're a HTTPS/TLS all the things! kinda guy, then yeah, the wire format is going to be a bunch of binary gibberish anyway.

### <a name="Deployability">[Deployability](#Deployability)</a>

One of the major considerations with SPDY and HTTP/2 has been how to deploy it widely and safely. If you don't care about wide, public interop, then there's not a huge need to standardize a protocol and you can go ahead and use the whatever private protocol you want. But otherwise, this is a major factor to keep in mind. SPDY sidesteps many issues by (1) not changing HTTP semantics, thereby avoiding compatilibity issues and (2) using TLS in all major deployments, which avoids issues with intermediaries. Let's review some of the various other options for deployment that people have discussed:

#### <a name="Pipelining"/>[Don't bother with HTTP/2, just use HTTP Pipelining](#Pipelining)</a>

Many have questioned the need for HTTP/2 if [pipelining](http://en.wikipedia.org/wiki/HTTP_pipelining) can already provide many of the benefits. Putting aside the [ambiguous performance benefits of pipelining](http://www.guypo.com/technical/http-pipelining-not-so-fast-nor-slow/), the main reason that pipelining isn't adopted by browsers is because of the compatibility issues. As Patrick [explains](http://bitsup.blogspot.com/2012/11/a-brief-note-on-pipelines-for-firefox.html) for Firefox:

> This is a painful note for me to write, because I'm a fan of HTTP pipelines.
> 
> But, despite some obvious wins with pipelining it remains disabled as a high risk / high maintenance item. I use it and test with it every day with success, but much of the risk in tied up in a user's specific topology - intermediaries (including virus checkers - an oft ignored but very common intermediary) are generally the problem with 100% interop. I encourage readers of planet moz to test it too - but that's a world apart turning it on by default- the test matrix of topologies is just very different.

I explain the same reasoning in more detail for [why Chromium won't enable pipelining by default either](/tech/status-of-http-pipelining-in-chromium/):

> For example, the test tries to pipeline 6 requests, and we’ve seen that only around 65%-75% of users can successfully pipeline all of them. That said, 90%-98% of users can pipeline up to 3 requests, suggesting that 3 might be a magic pipeline depth constant in intermediaries. It’s unclear what these intermediaries are...transparent proxies, or virus scanners, or what not, but in any case, they clearly are interfering with pipelining. Even 98% is frankly way too low a percentage for us to enable pipelining by default without detecting broken intermediaries using a known origin server, which has its aforementioned downsides. Also, it’s unclear if the battery of tests we run would provide sufficient coverage.

You'll note that Patrick refers to 100% interop, and I say that 98% interop is way too low. Breakage has to be absolutely tiny, because users have low tolerance for breakages. If a user can't load their favorite kitten website, they'll give up and switch browsers very quickly.

#### <a name="Upgrade"/>[HTTP Upgrade in the clear over port 80](#Upgrade)</a>

[HTTP Upgrade](http://tools.ietf.org/html/draft-ietf-httpbis-p1-messaging-24#page-56) is definitely an option, and it's currently in the [HTTP/2 draft spec](http://http2.github.io/http2-spec/#discover-http), primarily to serve as the cleartext option for HTTP/2. One of the major issues with it, from a deployment standpoint, is that it [has a relatively high failure rate over the internet](http://www.ietf.org/mail-archive/web/tls/current/msg05593.html). People around the web are [independently discovering this themselves](https://groups.google.com/forum/#!topic/sockjs/wAgVZoN5iC4) and are [recommending](https://speakerdeck.com/3rdeden/websuckets?slide=42) that you [only deploy WebSockets over TLS](http://lucumr.pocoo.org/2012/9/24/websockets-101/#when-to-use-them). There's some hope that if HTTP/2 gets standardized with the Upgrade method, that HTTP intermediaries will eventually (years, decades, who knows) get updated to accept upgrading to HTTP/2. But internet application developers, especially those who have real customers such that any loss of customer connectivity affects the bottom line, simply have very little incentive to use HTTP Upgrade to HTTP/2 as their deployment option. On the other hand, within private networks like corporate intranets, there shouldn't be as many troublesome uncontrolled HTTP intermediaries, so HTTP Upgrade might be much more successful in that scenario.

#### <a name="DifferentProtocol"/>[Use a different transport protocol](#DifferentProtocol)

In IETF 87 in Berlin, there was a joint tsvwg (transport area folks) and httpbis (HTTP folks) meeting. One of the things that came out of this meeting was that the transport area folk wanted a list of features that application (HTTP) folks wanted, which my colleague [Roberto provided](http://www.ietf.org/mail-archive/web/tsvwg/current/msg12184.html). This led to a series of responses asking why [HTTP/2 was reinventing the wheel](http://www.ietf.org/mail-archive/web/tsvwg/current/msg12183.html) and [why not use other protocols like SCTP/IP](http://lists.w3.org/Archives/Public/ietf-http-wg/2013JulSep/0984.html) and what not. Almost all of these basically come down to deployability. These transport features are not available on the host OSes that our applications run on top of, nor do they traverse NATs particularly well. That makes them not deployable for a large number of users on the internet. SCTP/UDP/IP is much more interesting to consider, although as [noted by Mike](http://lists.w3.org/Archives/Public/ietf-http-wg/2013JulSep/0646.html), it has its own issues like extra roundtrips for the handshake.

#### <a name="NewURLScheme"/>[Use a new URL scheme and ports](#NewURLScheme)

As [James Snell blogged about previously](http://www.chmod777self.com/2013/07/http2-wait-theres-more.html):

> Why is it this complicated? The only reason is because it was decided that HTTP/2 absolutely must use the same default ports as HTTP/1.1.... which, honestly, does not make any real sense to me. What would be easier? (1) Defining new default TCP/IP ports for HTTP/2 and HTTP/2 over TLS. (2) Creating a new URL scheme http2 and https2 and (3) Using DNS records to aid discovery. It shouldn't be any more complicated than that.

Going back to the earlier numbers that Adam Langley [shared with the TLS working group on the WebSocket experiment](http://www.ietf.org/mail-archive/web/tls/current/msg05593.html), using a new TCP port has a much lower connectivity success rate than SSL, most likely due to firewalls that whitelist only TCP ports 80 and 443. Granted, the success rate was higher than upgrading over port 80, but whereas HTTP Upgrade theoretically gracefully falls back to HTTP/1.1 when the upgrade fails (at least we hope so!), failure to connect to a new port is just a failure. And it might not even be a TCP RST or something, it might just hang (some firewalls send RSTs, some just drop packets), and force clients to employ timer based fallback solutions which are absolutely terrible for interactivity (like browsing).

The incentives simply aren't favorable for this deployment strategy to succeed. Server operators and content owners aren't terribly inclined to support this new protocol if it means they both have to update URLs in their content to use the new scheme and also tolerate a loss in customer connectivity. And client (e.g. browser) vendors aren't terribly incentivized to support the new protocol if using it results in connectivity failures, because the first thing a user does when a page loads in browser X is try browser Y out, and if it works there, then switch to it. And switching to a new scheme here breaks the shareability of URLs. Let's say I see a cute kitten photo at http2://this.website.has.cute.kittens.com/ in my favorite browser that supports the http2 scheme. I send this URL to my friend who is on IE8. It fails to load. This is a terrible user experience. We've seen this [happen before with WebP](http://arstechnica.com/information-technology/2013/04/chicken-meets-egg-with-facebook-chrome-webp-support/) (when not using content negotiation via Accept):

> It turns out that Facebook users routinely do things like copy image URLs and save images locally. The use of WebP substantially defeated both of these use cases, because apart from Chrome and Opera, there's very little software that's in regular day-to-day use that supports WebP. As a result, users were left with unusable URLs and files; files they couldn't open locally, and URLs that didn't work in Internet Explorer, Firefox, or Safari.

While this proposal simplifies the negotiation as James points out, it suffers from the downsides of terrible user experience, deployability difficulties, and requiring updating all URLs in all content.

#### <a name="ForgetCompat"/>[Forget backwards compatibility, fix issues from HTTP/1.1](#ForgetCompat)

PHK has written and talked extensively about his issues with the current HTTP/2 proposal. One of the major things he asserts is that the HTTP/2 proposal [simply layers more complexity on top of the existing layers, without tackling the deep architectural issues with HTTP/1.1](https://www.varnish-cache.org/docs/trunk/phk/http20.html#beating-http-1-1). For one, he suggests killing off cookies and replacing them with session IDs. Cookies are hated for many reasons (privacy, network bloat, etc), and the user agent definitely has some incentive to replace them with something better. However, as [Willy notes](http://lists.w3.org/Archives/Public/ietf-http-wg/2013JulSep/0287.html) and [Stephen affirms](http://lists.w3.org/Archives/Public/ietf-http-wg/2013JulSep/0292.html), servers don't really have much incentive to switch to this replacement. PHK [claims that](https://www.varnish-cache.org/docs/trunk/phk/http20.html#beating-http-1-1):

> In my view, HTTP/2.0 should kill Cookies as a concept, and replace it with a session/identity facility, which makes it easier to do things right with HTTP/2.0 than with HTTP/1.1.
>
> Being able to be "automatically in compliance" by using HTTP/2.0 no matter how big dick-heads your advertisers are or how incompetent your web-developers are, would be a big selling point for HTTP/2.0 over HTTP/1.1.

It's unclear to me how effective this "automatically in compliance" carrot is compared to the technical and financial costs of updating web servers/content to replace client-side cookies with server-side distributed, synchronized data stores. As PHK himself says, this raises the question for me, [what if they made a new protocol, and nobody used it](https://www.varnish-cache.org/docs/trunk/phk/http20.html#what-if-they-made-a-new-protocol-and-nobody-used-it)?

### <a name="TLSPrivacy">[TLS & Privacy](#TLSPrivacy)</a>

Ah yes, privacy. Well, that's a good thing, right? If communications aren't kept confidential, then it's difficult to maintain one's privacy. So why not try to preserve confidentiality in all communications, by doing stuff like mandating running HTTP/2 over a secure transport such as TLS?

#### <a name="PrivacyPolitics">[Political / Legal / Corporate restrictions on Privacy](#PrivacyPolitics)</a>

Well for one, some people think that increasing the use of TLS on the internet is overall bad for privacy due to political, legal, and economic reasons. PHK covers a long list of reasons in [his ACM Queue article on encryption](http://queue.acm.org/detail.cfm?id=2508864), but [his email to httpbis](http://lists.w3.org/Archives/Public/ietf-http-wg/2013JulSep/0933.html) summarizes it pretty well:

> Correct, but if you make encrypt mandatory, they will have to break
> _all_ encryption, that's what the law tells them to.
> 
> As long as encryption only affects a minority of traffic and they can
> easier go around (ie: FaceBook, Google etc. delivering the goods)
> they don't need to render _all_ encryption transparent.

PHK seems to fundamentally believe that it's futile to preserve confidentiality via technical means like encryption. If more people use encrypted communications, then that will just incentivize governments to break that encryption. The only solution here is political, and the IETF has no business trying to mandate use of encryption. On the other hand, some other folks believe that various parties already have plenty of incentive to try to break encryption, not to mention they are more likely to use other attack vectors rather than directly breaking the cryptosystems since the other vectors are typically easier to attack.

Putting aside the economics of breaking encryption, the legal aspect of mandatory encryption is very useful to note. As PHK's ACM Queue article details, mandatory encryption clashes with legal mandates in certain nation-states like the UK. Adrien de Croy (author of [WinGate](http://en.wikipedia.org/wiki/WinGate)) also [explains these issues](http://lists.w3.org/Archives/Public/ietf-http-wg/2012JulSep/0468.html):

> Proponents of mandatory crypto 
> are making an assumption that privacy is ALWAYS desirable.
> 
> It is not ALWAYS desirable. 
> 
> In many cases privacy is undesirable or even illegal.
> 
> 1. Many states have outlawed use of crypto.  So will this be HTTP only 
> for the "free" societies?  Or is the IETF trying to achieve political 
> change in "repressed" countries?  I've dealt with our equivalent of the 
> NSA on this matter.  Have you?  I know the IETF has a neutral position 
> on enabling wiretapping.  Mandating SSL is not a neutral / apolitical 
> stance.  Steering HTTP into a collision course with governments just 
> doesn't seem like much of a smart idea.
> 
> 2. Most prisons do not allow inmates to have privacy when it comes to 
> communications.  Would you deny [\*] all prisoners access to the web?  
> There are other scenarios where privacy is not expected, permitted or 
> desirable.

As Adrien and PHK have both pointed out, there are many situations where arguably privacy is undesirable. Employees sometimes don't get privacy because their companies want to scan all traffic to detect malware or loss of corporate secrets. Schools often must monitor students' computer use for porn. Going further, [Adrien blames large websites](http://lists.w3.org/Archives/Public/ietf-http-wg/2013OctDec/0688.html) whose use of SSL has encouraged companies and schools and other organizations to deploy MITM proxies (that rely on locally installed root certificates) to break MITM SSL connections to fulfill their corporate needs or legal obligations:

> We added MITM in WinGate mostly because Google and FB went to https.  
> Google and FB you may take a bow.
> 
> Does this improve security of the web overall?  IMO no.  People can now 
> snaffle banking passwords with a filter plugin.
> 
> You really want to scale this out?  How will that make it any better?

The counter to this argument seems fairly obvious to me. Just because some subset of users in specific situations (at work, at school, in a police-state, etc) may not be "allowed" to have privacy (due to whatever reason, be it legal, corporate policy, school policy, etc), we shouldn't give up trying to provide confidentiality of communication for everyone else the rest of the time. [I responded accordingly](http://lists.w3.org/Archives/Public/ietf-http-wg/2013OctDec/0698.html) and [Patrick McManus followed up saying](http://lists.w3.org/Archives/Public/ietf-http-wg/2013OctDec/0703.html):

<blockquote>
<p>
On Wed, Nov 13, 2013 at 7:09 PM, William Chan (陈智昌)<br>
<willchan@chromium.org> replied to Wily:<br>

><br>
> Just to be clear, the MITM works because the enterprises are adding new<br>
> SSL root certificates to the system cert store, right? I agree that that is<br>
> terrible. I wouldn't use that computer :) I hope we increase awareness of<br>
> this issue.<br>
</p>
<p>
This is a super important point. If someone can install a root cert onto
your computer then you are already owned - there is no end to the other
things they can do too. Call it a virus, call it an enterprise, but call it
a day - you're owned and there is no in-charter policy this working group
can enact to change the security level of that user for good or for bad..

The good news is not everyone is already owned and SSL helps those people
today.
</p>
</blockquote>

#### <a name="TLSPKICost">[The Cost of TLS & PKI](#TLSPKICost)</a>

Another common argument against trying to increase TLS usage is that it's costly. This cost takes a number of different forms:

* Computational / scalability costs - As discussed [earlier](#ScalabilityDoS), TLS incurs some amount of computational costs (more buffer copies, symmetric encryption/decryption, asymmetric crypto for key exchange, preventing caching at intermediaries, etc). As before, I won't delve into these costs in detail, there is already plenty of information out on the interwebs about them. The costs are real and exist.
* [PKI](http://en.wikipedia.org/wiki/Public-key_infrastructure) cost - Setting up certificates can incur some amount of cost. Note that StartSSL provides free certificates, and indeed that's what I myself use. That said, there are definitely costs to acquiring a certificate.
* Operational costs - Properly managing the keys (key rotation, etc) and certificates is difficult. Debugging is more difficult since now you have to decrypt the traffic. Setting up HTTPS requires a certain amount of knowledge, and it's not obvious when you do it incorrectly.

Cost has to be put into perspective. What portion of the total cost of operation does TLS increase? Is it significant? Here's a discussion thread between Mike Belshe and Yoav Nir on the [topic](http://lists.w3.org/Archives/Public/ietf-http-wg/2013OctDec/1375.html):

<blockquote>
<p>
> It's not contentious, it's just false. Go to the pricing page for Amazon <br>
> cloundfront CDN <http://aws.amazon.com/cloudfront/#pricing> (I would have <br>
> picked Akamai, but they don't put pricing on their website), and you pay <br>
> 33% more plus a special fee for the certificate for using HTTPS. That's <br>
> pretty much in line with the 40% figure. That's real cost that everybody <br>
> has to bear. And you will get similar numbers if you host your site on your <br>
> own servers.<br>
>
</p>
<p>
I think you're thinking like an engineer.  You're right, they do charge
more (and I'm right those prices will continue to come down).  But those
prices are already TINY.  I know 33% sounds like a lot, but this is not the
primary cost of operating a business.  So if you want to do a price
comparison, do an all-in price comparison.  And you'll find that the cost
of TLS is less than a fraction of a percent difference in operating cost
for most businesses.

And if you're not talking about businesses, but consumers, CDNs aren't
really relevant.  As an example, I run my home site at Amazon for zero
extra cost, but I did buy a 5yr $50 cert.
</p>
</blockquote>

Another consideration that has come up a few times is that [there are a number of other HTTP users that aren't browsers](http://lists.w3.org/Archives/Public/ietf-http-wg/2013JulSep/0511.html), such as printers (and other electronic devices). These users may want the new capabilities of HTTP/2, or they may simply not want to be stuck with an older/unsupported version of HTTP, yet they may not want to bear the cost of TLS. I mean, do people that own printers really want to have to set up a certificate for it? How important is it really to secure the communications to your printer or router? What about when washers, refrigerators, toasters, etc. are all networked?

#### <a name="DoWeWantToEncryptEverything">[Do We Want To Encrypt Everything?](#DoWeWantToEncryptEverything)</a>

In addition to many already well-known issues like passive monitoring & [attacks on unencrypted wifi networks](http://en.wikipedia.org/wiki/Session_hijacking#Exploits), it's become clear after the Snowden revelations that there's large scale state-sponsored passive pervasive monitoring occurring. In light of this information, how much traffic do we want to secure? It's costly, so is it OK if we don't encrypt communications to a printer? What about "unimportant" traffic? What if people don't think their traffic is important enough to encrypt?

Many people believe that if they have [nothing to hide](http://en.wikipedia.org/wiki/Nothing_to_hide_argument), then surveillance does not impact them. There have been [numerous](https://www.google.com/search?q=privacy+nothing+to+hide) [articles](https://chronicle.com/article/Why-Privacy-Matters-Even-if/127461/) [arguing](http://www.wired.com/politics/security/commentary/securitymatters/2006/05/70886) that this is a [flawed belief](http://www.thoughtcrime.org/blog/we-should-all-have-something-to-hide/). And as time goes on, there's more and more evidence showing how our governments are indiscriminately gathering all sorts of [information](http://www.theatlantic.com/politics/archive/2013/11/the-nsas-porn-surveillance-program-not-safe-for-democracy/281914/) on people for later possible use.

Given all the known (and potential) threats to privacy (and security), many people feel that it's of utmost importance to secure as much of people's communications as possible. Part of the reason is that, as [Stephen Farrell (IETF Security Area Director) says](http://lists.w3.org/Archives/Public/ietf-http-wg/2013OctDec/1377.html):

> Here, Mike is entirely correct. The network stack cannot know when
> the payload or meta-data are sensitive so the only approach that
> makes sense is to encrypt the lot to the extent that that is practical.
> Snowdonia should be evidence enough for that approach even for those
> who previously doubted pervasive monitoring.
>
> HTTP is used for lots of sensitive data all the time in places that
> don't use https:// URIs today. Sensitive data doesn't require any
> life or death argument, it can be nicely mundane, e.g. a doctor
> visit being the example Alissa used in the plenary in Vancouver.
>
> We now can, and just should, fix that. There's no hyperbole needed
> to make that argument compelling.

It's difficult to know what is "sensitive" and what isn't. It's reasonable to assume that most users browsing the web expect/want privacy in their browsing habits, yet don't know exactly what communications need to be secured. Making communications secure by default would address some of the privacy issues here.

It's of course, unclear how much it helps, and indeed some argue that doing this may provide a false sense of security. There are [well-known](https://bugzilla.mozilla.org/show_bug.cgi?id=647959) [issues](https://www.eff.org/deeplinks/2011/09/post-mortem-iranian-diginotar-attack) with the [CA system](http://en.wikipedia.org/wiki/Certificate_authority), some suspicions about whether or not cryptographic algorithms / algorithmic parameters have been [compromised](http://arstechnica.com/security/2013/09/new-york-times-provides-new-details-about-nsa-backdoor-in-crypto-spec/), and does transport level security even matter if [governments can simply persuade the endpoints to hand over data](http://en.wikipedia.org/wiki/PRISM_(surveillance_program))? On the flip side, many feel that despite any issues it has, encryption does work. At the [IETF 88 Technical Plenary](http://www.ietf.org/live/ietf88/text.html), Bruce Schneier drove [this point](http://www.youtube.com/watch?v=oV71hhEpQ20#t=31m29s) home:

> So we have a choice, we have a choice of an Internet that is vulnerable to all attackers or an Internet that is secure for all users.
> 
> All right? We have made surveillance too cheap, and we need to make it more expensive.
> 
> Now, there's good news-bad news. All right? Edward Snowden said in the first interview after his documents became public, he said, "Encryption works. Properly implemented, strong cryptosystems are one of the few things that you can rely on."
> 
> All right? We know this is true. This is the lesson from the NSA's attempts to break Tor. They can't do it, and it pisses them off.
> 
> This is the lessons of the NSA's attempt to collect contact lists from the Internet backbone. They got about ten times as much information from Yahoo! users than from Google users, even though I'm sure the ratio of users is the reverse. The reason? Google use SSL by default; Yahoo! does not.
> 
> This is the lessons from MUSCULAR. You look at the slides. They deliberately targeted the data where SSL wasn't protecting it. Encryption works.

On the other hand, some folks are worried that if we encrypt too much traffic, then it might make finding hostile traffic emanating from one's device hard to find, and thereby lower overall security. Bruce Perens [chimed in](http://lists.w3.org/Archives/Public/ietf-http-wg/2013OctDec/att-0779/00-part), originally to point out that encryption is illegal for ham radio, but also to point out that encrypting normal traffic will lower security since the hostile encrypted traffic will be hard to find:

> Let's make this more clear and ignore the Amateur Radio issue for now. I don't wish to be forced into concealment in my normal operations on the Internet.
> 
> Nor do I wish to have traffic over my personal network which I can not supervise. Unfortunately, there are a lot of operating systems and applications that I have not written which use that network. When I can't see the contents of their network traffic, it is more likely that traffic is being used to eavesdrop upon me. Surrounding that traffic with chaff by requiring encryption of _all_ HTTP traffic means that this hostile encrypted traffic will be impossible to find.
> 
> Thus, my security is reduced.

##### <a name="OpportunisticEncryption">[Opportunistic Encryption](#OpportunisticEncryption)</a>

Let's assume for the sake of discussion that securing more traffic is a good thing. How would one do so? There are a number of barriers to increased HTTPS adoption, which is why it's very slow going. But what about trying to secure http:// URIs too? That's the fundamental idea behind opportunistic encryption - to opportunistically encrypt http:// URIs when the server advertises support for it. Mark Nottingham (httpbis chair) put together a [draft for this](http://tools.ietf.org/html/draft-nottingham-http2-encryption-02). It's key to note that from the web platform perspective, http:// URIs remain http:// URIs, so the origins aren't changing, nor would the browser SSL indicator UI change.

There used to be some discussion of whether or not opportunistic encryption should require authentication. Requiring authentication would be a big barrier to adoption, since acquiring certificates is a major blocker for some folks. It's definitely an interesting middle ground, but I won't bother discussing it further since it's mostly died out for now.

The appeal of unauthenticated encryption should be fairly evident. It doesn't require CA-signed certificates, which means that it becomes perhaps feasible to achieve wide deployment of encryption by adding support for this into a few common webservers (perhaps enabled by default) and the major browsers.

Now, unauthenticated encryption obviously has some issues. If you do not authenticate the peer, then it's easy to do an active MITM attack. Authentication is key to preventing this, so unauthenticated encryption can only thwart passive attackers. However, is that good enough? Active attacks leave traces (and thus are detectable) and cost more, so if it doesn't have much cost, then it will raise the bar at least, which is a good thing. That said, it's an open question how much it raises the bar. It's easily defeated by [cheap downgrade attacks](http://tools.ietf.org/html/draft-nottingham-http2-encryption-02#section-3.1), although there's some hope that some form of pinning (maybe Trust on First Use?) could mitigate that.

It's key here to note that opportunistic encryption (at least without pinning or some mechanism to defeat active attackers) provides very marginal security benefit here, but might have some privacy wins if it can impose enough costs to make large-scale pervasive surveillance too costly. It's easy to see that any active attacker can downgrade/MITM you, so no one should have any illusions about the actual security benefit here. As for raising the costs for organizations to do pervasive surveillance, I don't think anyone would argue against that as long as it has no downsides. That said, it might be unwise to underestimate the resources that governments are willing to pour into pervasive surveillance.

As to whether or not it has tradeoffs, it definitely has some, although it's hard to quantify. By providing a middle ground between cleartext HTTP and secured HTTPS, it may lead to some folks switching to opportunistic encryption and not going all the way to HTTPS. [Some folks](http://it.slashdot.org/story/08/06/24/2345223/when-is-a-self-signed-ssl-certificate-acceptable) think that [encryption is sufficient](http://robert.accettura.com/blog/2008/07/19/unobstructed-https/), and skeptical of the value of authentication. Thus, the fear that some people who want fully authenticated HTTPS everywhere have is that offering the middle ground of opportunistic value may prevent some folks from biting the bullet and going all the way to HTTPS.

As [Tim notes](http://lists.w3.org/Archives/Public/ietf-http-wg/2013OctDec/0677.html), it's hard to weigh these benefits and costs here since there's no hard data:

<blockquote>
<p>
> As for downsides, will people read too much into the marginal security<br>
> benefit and thus think that it's OK not to switch to HTTPS? If so, that<br>
> would be terrible. It's hard to assess how large this risk is though. Do<br>
> you guys have thoughts here?
</p>
<p>
I agree that’s a risk, but we’re all kind of talking out of our asses here
because we don’t have any data.  My intuition is that people who actually
understand the issues will understand the shortcomings of opportunistic and
not use it where inappropriate, and people who don’t get why they should
encrypt at all will get some encryption happening anyhow.  But intuition is
a lousy substitute for data.
</p>
</blockquote>

Due to the difficulty of this cost/benefit analysis, in addition to the questionable (by some) value of encrypting as much traffic as possible, opportunistic encryption remains a hotly debated topic in httpbis. It'll be very interesting to see how it turns out.

#### <a name="Proxies">[Proxies](#Proxies)</a>

All this talk about switching more traffic to being encrypted end to end obviously raises a big question about what that means for proxies. [Interception (otherwise known as "transparent") proxies](http://trac.tools.ietf.org/wg/httpbis/trac/ticket/210) are quite ubiquitous in computer networks, especially in corporate networks, ISPs, mobile operators, etc. If use of encryption does increase on the web, then what happens to all these interception proxies? In the current world, there are only two options: (1) Do nothing. The interception proxies will simply see less traffic. (2) Turn the interception proxies into active SSL MITM proxies, by installing additional root certs on devices, for use by the proxy.

There are many issues with option (2), some of which are:

* MITM proxies are not detectable by users nor servers.
* When additional root certs are installed on devices, then the user agent is no longer sure it's authenticating the real server, so enhanced security mechanisms such as [public key pinning](http://tools.ietf.org/html/draft-ietf-websec-key-pinning-09) must be disabled.
* Likewise, SSL client authentication cannot work, since the MITM proxy does not (at least one should hope not!) have the client's private key.
* In order to achieve their goals, MITM proxies have to fully break the TLS connection. This means complete loss of confidentiality, integrity, and authentication guarantees, even when the proxy operators may only want to break confidentiality (e.g. for malware scanning) or just metadata confidentiality (e.g. examining HTTP headers in order to perhaps serve a cached copy).

For these various reasons, many in the httpbis working group are looking into ["explicit" proxies](https://github.com/http2/http2-spec/issues/316), where the "explicit" is mostly in contrast to the transparency of interception proxies. And many of these proposals call for making the proxy "trusted", for various definitions of trusted (give up confidentiality? integrity? everything?). Note that today's HTTP proxies (both configured and transparent) are essentially "trusted", since they can MITM HTTP transactions and do whatever they want. The question is whether or not we want to "weaken" HTTPS in any explicit manner. There are a [variety](http://tools.ietf.org/html/draft-rpeon-httpbis-exproxy-00) of [proposals](http://lists.w3.org/Archives/Public/ietf-http-wg/2013OctDec/1415.html) on the [table](http://lists.w3.org/Archives/Public/ietf-http-wg/2013OctDec/1431.html) here, almost all of which explicitly give up some of HTTPS' guarantees in some way. Some people go as far as to suggest that the client should fully trust the proxy. Some propose server, protocol (TLS or HTTP/2), or content modifications in order to provide more specific loss of guarantees, perhaps by switching to object level integrity instead of end to end TLS channel integrity.

#### <a name="WhatCanShouldTheIETFDo">[What Can/Should the IETF Do?](#WhatCanShouldTheIETFDo)</a>

There are some big questions here about what is reasonable and feasible for the IETF to do. A number of folks feel like the IETF has no business requiring certain behavior, but merely should specify mechanisms and leave it up to individual actors to decide what to adopt. For example, Adrien had [this](http://lists.w3.org/Archives/Public/ietf-http-wg/2013OctDec/1789.html) to say about it:
> Maybe the problem is us.
> 
> e.g. that we think the level of https adoption is a problem to be 
> solved.
> 
> personally I do not.
> 
> What if it simply reflects the desires of the people who own and run the 
> sites. Exercising their choice.
> 
> We are proposing (yet again) taking that choice away which I have a 
> major problem with.  It's a philosophical problem, I don't believe any 
> of us have the right to make those choices for everyone else, especially 
> considering (which few seem to be) the ENORMOUS cost.

Moreover, some people think that mandating encryption in HTTP/2 [is security theater](http://lists.w3.org/Archives/Public/ietf-http-wg/2013JulSep/0909.html). As Eliot Lear said:

> There simply is no magic bullet.  The economics are
> clear.  The means to encrypt has existed nearly two decades.   Mandating
> encryption from the IETF has been tried before – specifically with IPv6
> and IPsec.  If anything, that mandate may have acted as an inhibitor to
> IPv6 implementations and deployment and served as a point of ridicule.

On the flip side, there's a strong movement in the IETF to explicitly [treat pervasive surveillance as an attack](http://tools.ietf.org/html/draft-farrell-perpass-attack-03) that IETF protocols should defend against. This goes a step further beyond the IETF's previous consensus stance here, documented in [RFC 2804](http://www.ietf.org/rfc/rfc2804.txt). [Brian Carpenter explained](http://www.ietf.org/mail-archive/web/perpass/current/msg01342.html) further that:

> My understanding of the debate in Vancouver was that we intend
> to go one step beyond the RAVEN consensus (RFC 2804). Then, we
> agreed not to consider wiretapping requirements as part of the
> standards development process. This time, we agreed to treat
> pervasive surveillance as an attack, and therefore to try to
> make protocols resistant to it.
> 
> Which is completely disjoint from whether operators deploy
> anti-surveillance measures; that is a matter of national law
> and not our department.

While I still consider it very much under debate as to what the IETF _should_ be doing here, I think the current sentiment is definitely leaning towards adopting Stephen's draft to treat pervasive surveillance as an attack. In terms of how this applies to HTTP/2, httpbis chair Mark Nottingham [answered that for me](http://lists.w3.org/Archives/Public/ietf-http-wg/2013OctDec/1456.html):
<blockquote>
<p>
> What do "adequately address pervasive monitoring in HTTP/2.0"
</p>
<p>
Well, that's the fun part. Since there isn't specific guidance in this draft, we'll need to come up with the details ourselves. 

So far, our discussion has encompassed mandatory HTTPS (which has been controversial, but also seems likely to be in some of the first implementations of HTTP/2.0) and opportunistic encryption (which seems to have decent support in principle, but there also seems to be some reluctance to implement, if I read the tea leaves correctly). Either of those would probably "adequately address" if we wrote them into HTTP/2.0.

Alternatively, it may be that we don't address pervasive monitoring in the core HTTP/2.0 document itself, since HTTP is used in a such a wide variety of ways, but instead "adequately address" in a companion document. One proposal that might have merit is shipping a "HTTP/2.0 for Web Browsing" document and addressing pervasive monitoring there.

My biggest concern at this point is the schedule; we don't have the luxury of a drawn-out two year debate on how to do this.
</p>
<p>
> and "we'll very likely get knocked back for it" mean?
</p>
<p>
It means the IESG would send the documents back to us for further work when we go to Last Call.
</p>
</blockquote>

This puts the httpbis working group in an interesting situation of perhaps being required to do something to address pervasive surveillance in HTTP/2. Of course, whether or not there's any consensus at all to do something here remains to be seen. Most of the players involved seem to be sticking to their various positions, which makes me a little skeptical that mandating any sort of behavior will reach "rough consensus".

What will happen if the IETF can't get any consensus on anything TLS-related? At that point, it's likely that the market will decide. So it's interesting to see what the current landscape of vendor opinion is. On the browser front, all major browsers currently only support SPDY over TLS. Chromium and Firefox representatives have insisted on only supporting HTTP/2 over TLS, whereas Microsoft insists that HTTP/2 must provide a cleartext option:

[Patrick McManus (Firefox)](http://lists.w3.org/Archives/Public/ietf-http-wg/2013OctDec/0987.html):
> I will not deploy another cleartext protocol. Especially another one where
> the choice of encryption is solely made by the server. It doesn't serve my
> user base, or imo the web.

[Me (Chromium)](http://lists.w3.org/Archives/Public/ietf-http-wg/2013OctDec/0676.html):
> Well, it should be no surprise that the Chromium project is still planning
> on supporting HTTP/2 only over a secure channel (aka TLS unless something
> better comes along...).

[Rob Trace (WinInet, in other words, IE)](http://lists.w3.org/Archives/Public/ietf-http-wg/2013OctDec/0662.html):
<blockquote>
<p>
We are one browser vendor who is in support of HTTP 2.0 for HTTP:// URIs.  The same is true for our web server.  I also believe that we should strongly encourage the use of TLS with HTTP, but not at the expense of creating a standard that is as broadly applicable as HTTP 1.1.
</p>
<p>
I think this statement correctly captures the proposal:
</p>
<p>
> To be clear - we will still define how to use HTTP/2.0 with http://<br>
> URIs, because in some use cases, an implementer may make an<br>
> informed choice to use the protocol without encryption.
</p>
</blockquote>

If these vendors maintain these positions, and all signs point towards that being the case, then it'll be interesting to see how those market forces, combined with the unreliability of deploying new non-HTTP/1.X cleartext protocols over port 80, will affect how HTTP/2 gets deployed on the internet.
