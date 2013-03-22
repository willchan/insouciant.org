---
title: Connection Management in Chromium
author: willchan
layout: post
categories:
  - tech
tags:
  - chromium
  - http
  - performance
  - spdy
  - ssl
---
Connection latency and parallelism are significant factors in the networking component of web performance. As such, Chromium engineers have spent a significant amount of time studying how best to manage our connections – how many to keep open, how long to keep them open, etc. Here, I present the various considerations that have motivated our current design and implementation.

## <a id="#Connection_latency">[Connection latency](#Connection_latency)</a>

It’s expensive to set up a new connection to a host. First <a id="return-note-conn-man-1"></a>[<sup>1</sup>](#note-conn-man-1), Chromium has to resolve the hostname, which is already an expensive process as [I’ve previously discussed][20]. Then it has to do various handshakes. This means doing the TCP handshake, and then potentially doing an SSL handshake. As seen below, both of these are expensive operations. Note that the samples gathered here represent the TCP connect()s and SSL handshakes as gathered over a single day in February 2013 from our users who opt-in to making Google Chrome better. And since these are gathered at the network stack level, it includes more than just HTTP transactions, and more than just web browsing requests.

 [20]: https://plus.google.com/103382935642834907366/posts/FKot8mghkok

<a id="tcp_connection_latency_chart"></a>
<figure>
{% img /images/2013/02/tcp_connection_latency.png %}
<figcaption>
TCP connect() latency. Note the spikes towards the long tail, indicating the platform-specific TCP retransmission timeouts.
</figcaption>
</figure>

<figure>
{% img /images/2013/02/tcp_connection_latency_cdf.png %}
<figcaption>CDF of TCP connect() latency.</figcaption>
</figure>

<figure>
{% img /images/2013/02/ssl_connection_latency.png %}
<figcaption>
SSL handshake latency. Includes the cert verification time (which may include an online certificate revocation check).
</figcaption>
</figure>

<figure>
{% img /images/2013/02/ssl_connection_latency_cdf.png %}
<figcaption>
CDF of SSL handshake latency.
SSL handshake latency. Includes the cert verification time (which may include an online certificate revocation check).
</figcaption>
</figure>

As can be seen, new TCP connections, much less full SSL connections, generally cost anywhere from tens to hundreds of milliseconds, so hiding this latency as much as possible by better connection management is important for reducing user perceived latency.

## <a id="socket_late_binding">[Socket “late binding”](#socket_late_binding)</a>

When I first began working on Chromium in 2009, the team had recently finished the cross platform network stack rewrite, and finally had time to begin optimizing it. Our HTTP network transaction class originally did the simple thing, maintaining the full state machine internally and proceeding serially through it – proxy resolution, host resolution, TCP connection, SSL handshake, etc. We called this original state “early binding”, where the HTTP transaction was bound early on to the target socket. This was obviously suboptimal, because while the HTTP transaction was bound to a socket and waiting for the TCP (and possibly the SSL) handshake to complete, a new socket might become available (might be a newly connected socket or newly idle persistent HTTP connection). Furthermore, one HTTP transaction may be higher priority than another, but due to early binding and network unreliability (packet loss and reordering and what not), the higher priority transaction may get a connected socket later than the lower priority socket. My first starter project on the team was to replace this with socket “late binding”, introducing the concepts of socket requests and connect jobs. HTTP transactions would issue requests for sockets, and sit in priority queues, waiting for connect jobs or newly released persistent HTTP connections to fulfill them. When we first launched this feature for TCP sockets, we saw a [6.5% reduction in time to first byte on each HTTP transaction][29], due to the improved connection reuse.

 [29]: http://src.chromium.org/viewvc/chrome?view=rev&revision=36230

<figure>
{%img /images/2013/02/socket_late_binding.png %}
<figcaption>
Example of socket late binding. Connection 1 is used for bar.html, and then reused for a.png. Connections 2 and 3 get kicked off, but 3 connects first, and binds to b.png. Before 2 finishes connecting, a.png is done on 1, so connection 1 is reused again for c.png. In this example, only connections 1 and 3 are used, and connection 2 is unused and idle.
</figcaption>
</figure>

## <a id="socket_late_binding_optimizations">[Leveraging socket late binding to enable more optimizations](#socket_late_binding_optimizations)</a> #

Delaying the binding of a HTTP transaction to an actual socket gave us important flexibility in deciding which connect job would fulfill a socket request. This flexibility would enable us to improve the median and long tail cases in important ways.

### <a id="tcp_backup_connect_jobs"></a>[TCP “backup” connect jobs](tcp_backup_connect_jobs)

Examining the TCP connection latency chart [above](#tcp_connection_latency_chart), you can see that the Windows connection latency has a few spikes. The spikes at the 3s mark and later correspond to the Windows TCP SYN retransmission timeout. Now, 3 seconds may make sense for some applications, but it’s horrible for an interactive application where the user is staring at the screen waiting for the page to load. Our data indicates that around 1% of Windows TCP connect() times fall into the 3~ second bucket. Our solution here was to introduce a “backup” connect job, set to start 250ms <a id="return-note-conn-man-2"></a>[<sup>2</sup>](#note-conn-man-2) after the first socket connect() to an origin. This [very hacky solution][33] attempts to workaround the unacceptably long TCP SYN retransmission by retrying sooner. Note that we only ever have one “backup” connect job active per destination host. After implementing this, our socket request histograms showed that the spike at 3s due to the TCP level SYN transmission timer went down significantly. As someone who feels very passionately about long-tail latency and reducing jank, this change makes me feel very good, and I was excited when Firefox [followed suit][34].

 [33]: http://src.chromium.org/viewvc/chrome?view=rev&revision=41543
 [34]: http://bitsup.blogspot.com/2010/12/accelerated-connection-retry-for-http.html

<figure>
{% img /images/2013/02/tcp_backup_connect_job.png %}
<figcaption>
Here you see the DNS TCP times of each socket connection vs the request time for a connected socket (filtered by only newly connected TCP sockets). It’s useful to note where the backup job helps…only with the TCP SYN retransmission timeouts, but not with the DNS retransmission timeouts. This is because we don’t call getaddrinfo() multiple times per host, only once. With our upcoming new DNS stub resolver, we can control DNS retransmission timeouts ourselves. The backup connect job is fairly effective at reducing the spikes at the TCP retransmission timeouts.
</figcaption>
</figure>

Some people have asked why we employ this approach instead of the [IE9 style approach][37] of opening 2 sockets in parallel <a id="return-note-conn-man-3"></a>[<sup>3</sup>](#note-conn-man-3). That idea is indeed intriguing and has its upsides as they point out. But in terms of compensating for SYN packet loss, it’s probably an inferior technique, if you assume a bursty packet loss model where bursts overflow routing buffers. The overall idea definitely has merits which we need to consider, but there’s less benefit for Chromium since Chromium also implements socket preconnect.

 [37]: http://blogs.msdn.com/b/ie/archive/2011/03/17/internet-explorer-9-network-performance-improvements.aspx

### <a id="socket_preconnect"></a>[Socket preconnect](#socket_preconnect)

Once the HTTP transactions were detached from a specific socket and instead just used the first available one, we were able to speculatively preconnect sockets if we thought they would be needed. Basically, we simply leveraged our existing Predictor infrastructure that we used to power DNS prefetching, and hooked it up to [preconnect sockets][39] when we had extremely high confidence. Therefore, by the time WebKit decides it wants to initiate a resource request, the socket connect has often already started or even completed. As Mike demonstrates, this [results in huge (7-9%) improvements in PLT in his synthetic tests][40]. Later on, when we refactored the [SSL connections to leverage the late binding socket pool infrastructure][41], we were able to expose a generic preconnect interface that allowed us to prewarm all types of connections (do TCP handshakes and [HTTP CONNECTs][42] and SSL handshakes) as well as prewarm multiple connections at the same time. Our Predictor infrastructure lets us predict based on past browsing, for a given network topology, what is the maximum number of concurrent transactions we’d want, and takes advantage of the genericized “socket” <a id="return-note-conn-man-4"></a>[<sup>4</sup>](#note-conn-man-4) pool infrastructure to fully “connect” a socket.

 [39]: http://src.chromium.org/viewvc/chrome?view=rev&revision=47479
 [40]: http://www.belshe.com/2011/02/10/the-era-of-browser-preconnect/
 [41]: http://src.chromium.org/viewvc/chrome?view=rev&revision=52275
 [42]: http://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html#sec9.9

## <a id="improving_connection_reuse"></a>[Improving connection reuse](#improving_connection_reuse)

While working on improving the code so we could better take advantage of available connections, we also spent some time studying how to increase the reusability of connections and optimize the choice of connection to reuse.

### <a id="socket_lifetime"></a>[Determining how long to keep sockets open](#socket_lifetime)

The primary mechanism we studied for increasing connection reusability was adjusting our connection idle times. This is a sensitive area to play around with. Too long, and it’s likely that some middlebox or the origin server has closed the connection, and may or may not have send a TCP FIN (or it might be on its way), leading to a wasted attempt to use the connection only to pay a roundtrip to receive a TCP RST. Too short, and we shrink the window for potential reuse. It also got more complicated in a socket late binding world, since sometimes we would get resource request cancellations or preconnects that would lead to sockets potentially sitting idle in the socket pool prior to being used. Previously, a socket would only be idle in between reuse, but now a socket might sit idle before first use. It turns out that middleboxes and origin servers have different idle timeouts for unused vs previously used connections. Moreover, we have to be much more conservative with unused sockets, since if we get a TCP RST on first use, it may be indicative of server/network issues, rather than idle connection timeouts. Therefore, we only retry our HTTP request on previously used persistent HTTP(S) connections. With that in mind, we gathered histograms on the type of socket usage, and the time sockets sat idle before use/reuse, in order to guide our choice of appropriate socket timeouts.

<figure>
{% img /images/2013/02/HTTPSocketIdleTimeBeforeUse.png %}
<figcaption>
Shows the amount of time we wait before using or reusing a socket. Note that Chromium currently times out unused sockets after 10-20s, and used sockets after 6min~.
</figcaption>
</figure>

<figure>
{% img /images/2013/02/HTTPSocketUseType.png %}
<figcaption>
Shows how often we are waiting for a new UNUSED socket, how often we immediately grab an UNUSED_IDLE socket, and how often we get to reuse a previously used idle socket.
</figcaption>
</figure>

One thing to note about the first graph is that the reason we’re fuzzy about the absolute idle socket timeout is because we run a timer every 10s that expires a socket if it exceeded the timeout, so when the timeout is 10s (as it is for unused, idle sockets), the true timeout is 10-20s. What the above charts show, is that there are significantly diminished returns to extending the idle socket times for previously used sockets, but that there is definitely potential for reuse by extending the socket timeout for unused, idle sockets (that’s why the slope for their CDFs caps out so abruptly at 100%). Before people jump to concluding that we should indeed increase the socket timeout for unused, idle sockets, keep in mind that every time we get this wrong, we may show a user-visible error page, which is a terribly confusing user experience. All they get is that Chromium failed to load their requested resource (which is terrible if it’s a toplevel frame). Moreover, looking at the [HTTP Socket Use Types chart][48], you can see that the percentage of times that we actually utilize unused, idle sockets is extremely low. Obviously, if we increased their idle socket timeout, the reuse percentage would increase somewhat, and we should also note that the use of a UNUSED or UNUSED_IDLE socket is extremely important, since it’s probably directly contributing to user perceived latency during page loads. There’s a tradeoff here, where we could gain more connection reuse, at the cost of more user visible errors.

 [48]: #HTTPSocketUseType

### <a id="total_socket_limit"></a>[Determining how many total sockets to keep open](#total_socket_limit)

Beyond trying to optimize reuse of individual sockets, we also looked into our overall socket limits to see if we could increase the number of sockets we keep available for potential reuse. Originally we had no limits, like IE, but eventually we encountered issues with not having limits due to [broken third party software][49], so we had to fix our accounting and lower our max socket limit. We knew this probably impacted performance due to lower likelihood of being able to reuse sockets, but since certain users were broken, we didn’t know what to do. Eventually, due to many bug reports where I saw us reach the max socket limit (not necessarily all in active use, but it meant we were closing idle sockets before their timeout, thereby limiting potential reuse), we reversed our policy and just decided that the third party should fix their code (some of our users helpfully reached out to the vendor) and bumped our limit up to 256, which hopefully should be enough for the foreseeable future.

 [49]: https://code.google.com/p/chromium/issues/detail?id=32817

### <a id="idle_socket_reuse"></a>[Determining which idle socket to use](#idle_socket_reuse)

One thing to keep in mind about reusing idle sockets is that not all sockets are equal. If sockets have been idle longer, there’s a greater likelihood that the middleboxes/servers have terminated the connection or that the kernel has timed out the congestion window <a id="return-note-conn-man-5"></a>[<sup>5</sup>](#note-conn-man-5) If some sockets have read more data, then their [congestion windows][51] are probably larger. The problem with greedily optimizing at the HTTP stack level is it’s unclear if it’s ultimately beneficial for user perceived latency. We can more or less optimize for known HTTP transactions within the HTTP stack, but a millisecond later, WebKit might make a request for a much higher priority resource. We may hand out the “hottest” socket for the highest priority currently known HTTP transaction, but if a higher priority HTTP transaction comes in immediately afterward, that behavior may be *worse* off for overall performance. Since we don’t have crystal balls lying around, it makes it difficult for us to determine how best to allocate idle sockets to HTTP transactions. So, we decided to run [live user experiments][52] <a id="return-note-conn-man-6"></a>[<sup>6</sup>](#note-conn-man-6) to see what worked best in practice. Unfortunately, the results were inconclusive: “Skimming over the results, the primary determinant of whether there’s any difference in the four metrics I looked at, and if so, which group came out on top (Time to first byte, time from request send to last byte, PLT, and net error rates) was which day I was looking at.  No consistent differences in either the long tail or everyone else.” Since we didn’t see any clearly better approach, we optimized for simplicity and kept our behavior of preferring the most recently used idle socket.

 [51]: http://en.wikipedia.org/wiki/Congestion_window
 [52]: http://src.chromium.org/viewvc/chrome?view=rev&revision=97991

## <a id="how_much_parallelism_is_good"></a>[How much parallelism is good?](#how_much_parallelism_is_good)

The simple, naive reasoning is that more parallelism is good, so why don’t we parallelize our HTTP transaction as much as possible? The short answer is contention. There are different sorts of contention, but the primary concern is bandwidth. More parallelization helps achieve better link utilization, which as [I’ve discussed previously][54], often introduces contention in the web browsing scenario and thus may actually *increase* page load time. It’s difficult for the browser to know what level of parallelization is optimal for a given website, since that’s also predicated on network characteristics (RTT, bandwidth, etc) which we often are not aware of (we can potentially estimate this on desktop, but it’s very difficult to do so on mobile given the high variance). So, in the end, for HTTP over TCP, at least for now, we’re basically stuck with choosing static parallelization values for all our users. Since Chromium’s HTTP stack is agnostic of web browsing (it has multiple consumers like captive portal detection, WebKit, omnibox suggestions, etc), it’s been historically difficult to control parallelization beyond the connections per host limit, but now that we’re developing our own Chromium-side [ResourceScheduler][55] that handles resource scheduling across all renderer processes, we may be able to be smarter about this. So our live experimentation in this area has primarily focused on [studying connection per host limits][56].

 [54]: /tech/resource-prioritization-in-chromium/
 [55]: https://code.google.com/p/chromium/issues/detail?id=163963
 [56]: http://src.chromium.org/viewvc/chrome?view=rev&revision=49057

### <a id="connection_per_host_limit"></a>[Experimenting with the connection per host limit](#connection_per_host_limit)

In order to identify the optimal connection per host limit in real websites, we ran [live experiments][57] on [Google Chrome’s dev channel][58] with a variety of connection per host limits (4, 5, 6, 7, 8, 9, 16) back in summer 2010. We primarily analyzed two metrics: Time to First Byte (TTFB) and Page Load Time (PLT). 4 and 16 were immediately thrown out since their performance was very clearly worse, and we focused our experiment population on 5-9. Out of those options, 5 and 9 performed the worst, and 6-8 performed fairly similarly. 7 and 8 performed very very similarly in both TTFB and PLT. 8 connections had anywhere from 0-2% TTFB improvement over 6 connections. In terms of actual PLT improvement though, 8 connections had around a 0.2%-0.8% improvement over 6 connections up to the 90th percentile. Beyond the 90th percentile, 8 connections had a PLT **regression **in comparison to 6 connections of around 0.7-1.3%. The [summary of our discussion][59] was: “6 is actually a decent value to be using. 7 connections seemed to perform slightly better in some circumstances, but probably not enough to make us want to switch to it.” There are many qualifications about this experiment.

 [57]: http://code.google.com/p/chromium/issues/detail?id=44491
 [58]: http://www.chromium.org/getting-involved/dev-channel
 [59]: http://code.google.com/p/chromium/issues/detail?id=44491#c7

For example, these [metrics are sort of flawed][60], but they were the best we had at the time. Today, there are other interesting metrics, like [WebPageTest’s Speed Index][61], which would be cool to analyze. Also, this experiment data only shows us in aggregate across our population what the ideal connection per host limit might be, but as previously noted, there’s every reason to believe that the optimal connection per host limit would be different based on certain variables, like network type (mobile, modem, broadband), geographic location (roundtrip times are very high in certain regions), user agent (web content often varies greatly for mobile and desktop user agents), etc. As noted before, it’s difficult to do accurate network characteristics estimation, especially on mobile, but we might try to tweak the limit based on whether or not you’re on a mobile connection.

 [60]: http://calendar.perfplanet.com/2012/moving-beyond-window-onload/
 [61]: https://sites.google.com/a/webpagetest.org/docs/using-webpagetest/metrics/speed-index

### <a id="too_many_connections"></a>[Opening too many TCP connections can cause congestion](#too_many_connections)

Another issue that we’ve seen with increasing parallelism, aside from contention, is bypassing TCP’s congestion avoidance. TCP will normally try to avoid congestion by starting [slow start][62] with a conservative initial congestion window (initcwnd), and then ramp up from there. Note, however, that the more connections a client opens concurrently, the higher the effective initcwnd is. Also recall that [newer Linux kernel versions have increased the TCP initcwnd to 10][63]. What this means is that when the client opens too many connections to servers, it’s very possible to cause congestion and packet loss immediately, thus negatively impacting page load time. Indeed, as I’ve [previously mentioned][64], this definitely happens in practice with some major websites. Really, as we’ve said time and time again, the solution is to deploy a protocol that supports prioritized multiplexing over a single connection, e.g. [SPDY][65] ([HTTP/2][66]), so web sites don’t need to open more TCP connections to achieve more parallelism.

 [62]: http://en.wikipedia.org/wiki/Slow-start
 [63]: http://kernelnewbies.org/Linux_2_6_39#head-c2acd2f0463943210471a42bf6f5b469a6999e7b
 [64]: /tech/some-more-reasons-hacking-around-http-bottlenecks-sucks-and-we-should-fix-http/
 [65]: http://www.chromium.org/spdy/spdy-whitepaper
 [66]: https://github.com/http2/http2-spec

## <a id="conn_man_spdy"></a>[Making Chromium’s connection management SP\[ee\]DY](#conn_man_spdy)

As our SPDY implementation got under way, we realized that our existing connection management design held back the SPDY implementation from being optimal. What we realized was that our HTTP transaction didn’t really need to bind to a socket per se, but rather to a HTTP “stream” over which it could send a HTTP request over and from which receive a HTTP response. The normal HttpBasicStream would of course be directly on top of a socket, but this abstraction would let us also create a SpdyHttpStream that would translate the HTTP requests to SPDY [SYN_STREAMs][67], and [SYN_REPLYs][68] to HTTP responses. It also provided us with a good shim point in which to play around with a HTTP pipelining implementation that would create HttpPipelinedStreams. Now, with this new layer of indirection, we had a new place where we wanted to once again delay our HTTP transaction binding to a HttpStream as long as possible, which meant pulling more states out of our serial HTTP transaction state machine and moving them into HttpStreamFactory jobs. When a new SPDY session gets established over a socket, each HttpStreamFactory job for that origin can create SpdyHttpStreams from that single SPDY session. So, even if we don’t know that the origin server supports SPDY, and we may initiate multiple connections to the server, once one connection completes and indicates SPDY support, we are able to immediately execute all HTTP transactions for that origin over that first SPDY connection, without waiting for the other connections to complete. Moreover, in certain circumstances, a single SPDY session might even service HttpStreamFactory jobs for [different origins][69].

 [67]: http://dev.chromium.org/spdy/spdy-protocol/spdy-protocol-draft3#TOC-2.6.1-SYN_STREAM
 [68]: http://dev.chromium.org/spdy/spdy-protocol/spdy-protocol-draft3#TOC-2.6.2-SYN_REPLY
 [69]: https://code.google.com/p/chromium/issues/detail?id=42669

<figure>
{%img /images/2013/02/http_stream_late_binding_spdy.png %}
<figcaption>
HttpStream late binding, enabling a single SPDY session to create HttpStreams simultaneously.
</figcaption>
</figure>

Once we had the HttpStream abstraction, we could switch from preconnecting sockets to preconnecting HTTP streams. When we visit SPDY-capable origins, [we record that it supports SPDY][72], so we know on repeat visits only to preconnect one actual socket, even though the Predictor requested X HTTP streams.

 [72]: http://src.chromium.org/viewvc/chrome?view=rev&revision=104666

## <a id="mobile_conn_man"></a>[Mobile connection management](#mobile_conn_man)

Ever since the Android browser started embedding Chromium’s network stack, the core network team has assisted our mobile teams in tuning networking for our mobile browsers.

### <a id="blackholed_tcp"></a>[Detecting blackholed TCP connections with SPDY](#blackholed_tcp)

One of the very first changes we had to make was related to how we kept alive our idle sockets. Historically, we’ve found that [TCP middleboxes may time out TCP connections (probably actually NAT mappings), but instead of notifying the sender with a TCP RST, they simply drop packets, leading to painful TCP retransmission timeout, causing requests to hang for minutes until the TCP connection finally timed out][73]. This happened for HTTP connections before, but since we often opened multiple HTTP connections to a server, page loads would generally be pretty resilient, and at worst the user could recover by reloading the page (the problematic connection would still be hung and thus not used for the new HTTP requests). However, for SPDY, where we try to send all requests over the same connection, a hung TCP connection would completely hang the page load.  We solved this problem at first by making sure TCP middleboxes never timed out our connections, by [using TCP keepalives][74] at 45s intervals. However, this obviously wakes up the mobile radio (the radio is very power hungry) every 45s, which drains a ton of batteries, so the Android browser team turned it off. And then they found that SPDY connections started hanging. So we sat around wondering what to do, and realized that what we wanted wasn’t to keep the connections alive, but rather to detect the dead connections (rather than waiting for lengthy TCP timeouts). So, we decided to use the [SPDY PING frame][75] as our [liveness detection mechanism][76], since it was specced to require the peer to respond asap. This means that we could detect dead SPDY connections far sooner than dead HTTP/1.X connections, and close the SPDY connection and retry the requests over a new connection. We still have to be a bit conservative with the retry timeout, since if it is less than RTT, we’ll always fail the PING. Really, what we want to do in the future is not actually close the connection down on timeout, but [race building a new SPDY connection in parallel and only use that connection if it completes before the PING response returns][77].

 [73]: https://code.google.com/p/chromium/issues/detail?id=27400
 [74]: http://src.chromium.org/viewvc/chrome?view=rev&revision=71482
 [75]: http://dev.chromium.org/spdy/spdy-protocol/spdy-protocol-draft2#TOC-PING
 [76]: http://src.chromium.org/viewvc/chrome?view=rev&revision=105723
 [77]: https://code.google.com/p/chromium/issues/detail?id=127812

### <a id="radio_usage"></a>[Reducing unnecessary radio usage](#radio_usage)

While examining this issue for Android browser, they also informed me that they were under a lot of pressure to try to save even more power by closing all HTTP connections after single use. The reasoning is as follows:

*   After a server-specific timeout, the server will close the socket, sending a TCP FIN to the Android device which wakes up the radio just to close the socket.
*   After Chromium’s idle socket timeouts, we close the sockets, which sends a FIN from the Android device to the server, again waking up the radio just to send a FIN.
*   As [Souders explains in detail][78], waking up the radio is expensive and the radio only powers back down based off timers that move the connection through a state machine.

 [78]: http://www.stevesouders.com/blog/2011/09/21/making-a-mobile-connection/

The problem with this proposed solution is that closing down each connection after a single HTTP transaction prevents reuse, which slows down page load, which may ultimately keep the radio alive longer. It’s unclear to me what the right solution is, but I sort of suspect that closing down the HTTP connections after a single transaction is ultimately worse for power, and clearly worse from the performance perspective. One thing we can do to mitigate the problem is [not to ever wake up the radio just to send a FIN][79]. Indeed, this is what we [now do][80] for most platforms. However, that only addresses the client side of the problem, but not the server side. This is another one of those cases where SPDY helps. By reducing the number of connections, SPDY reduces the number of server FINs that will ultimately be received by the client, thus reducing the total number of times server FINs wake up / keep alive the client radio.

 [79]: https://plus.google.com/103382935642834907366/posts/JqxAbV2AWgw
 [80]: https://code.google.com/p/chromium/issues/detail?id=101820

## <a id="conn_man_conclusion"></a>[Conclusion](#conn_man_conclusion)

Connection management is one of the parts of Chromium’s networking stack that has a large impact on performance, so we have spent a lot of time optimizing it. Much of the work has been refactoring the original HTTP stack which previously kept all HTTP transactions fairly independent of each other, with a serial state machine where the transaction would only be blocked on a single network event, into a much more intertwined HTTP stack, where networking events initiated by one HTTP transaction may create opportunities for a different HTTP transaction to make progress. All HTTP transactions share the same overall context, with different layers initiating jobs and servicing requests in priority order. We’ve also spent a lot of time experimenting with different constants in the HTTP stack, trying to figure out what static compile-time constants are best for the majority of our users. Yet we know that any single constant we pick will be suboptimal for a large number of users/networks. This is one reason why we advocate SPDY so strongly – it reduces our reliance on these constants by providing much better protocol semantics. Rather than having to make a tradeoff between link utilization/parallelism and contention, not to mention having to worry about bypassing TCP’s congestion avoidance with too many connections, we can have our cake and eat it too by leveraging SPDY’s prioritized multiplexing over a single connection. We’ve spent a lot of work in this area of the Chromium network stack, but there’s still a lot more work to be done. We really have only started scratching the surface of optimizing for mobile. And as Mike likes to say, SSL is the unoptimized frontier. Moreover, new technologies like TCP Fast Open are coming out, and we’re wrapping our heads around how to properly [take advantage of a zero roundtrip connect][81] (well, you only get to fit in a single packet’s worth of data, but that’s plenty for say…a SSL CLIENT_HELLO). So much to do, so little time.

 [81]: https://code.google.com/p/chromium/issues/detail?id=175623

Notes:

1.  <a id="note-conn-man-1"></a>Well, not quite first. There are other steps for certain configurations, like proxy resolution. [↩][82]
2.  <a id="note-conn-man-2"></a>This constant is obviously suboptimal in lots of cases, and we’re looking into fixing it by implementing it more dynamically. [↩][83]
3.  <a id="note-conn-man-3"></a>It appears to have some interesting side effects. It looks like they issue [double GETs][84], and cancel them, hoping that Windows will reuse the socket for other GETs, but sometimes it [doesn’t work][85] and they just burn a socket. They probably have to do it this way since they rely on [WinINet][86], rather than implementing the HTTP stack themselves. But I’m just speculating, I don’t really know. [edit: [Eric Lawrence][87] chimed in with a much more [authoritative explanation][88]]. [↩][89]
4.  <a id="note-conn-man-4"></a>Chromium abuses the term socket to generically refer to a “connected” byte stream [↩][90]
5.  <a id="note-conn-man-5"></a>Linux (for HTTP, we generally care more about the server congestion window) by default times out the congestion window after a RTO, controllable by the [*tcp\_slow\_start\_after\_idle*][91] setting. [↩][92]
6.  <a id="note-conn-man-6"></a>“warmest socket” uses the socket which has read the most bytes. ”last accessed socket” uses the socket that has been idle for the least amount of time. ”warm socket” uses “read bytes / (idle time)^0.25 [↩][93]

 [82]: #return-note-conn-man-1
 [83]: #return-note-conn-man-2
 [84]: http://www.webpagetest.org/result/130218_Q4_VQH/1/details/
 [85]: http://cloudshark.org/captures/4b7e565ee34a?filter=tcp.stream eq 4
 [86]: http://msdn.microsoft.com/en-us/library/windows/desktop/aa383630(v=vs.85).aspx
 [87]: https://twitter.com/ericlaw
 [88]: #comment-67
 [89]: #return-note-conn-man-3
 [90]: #return-note-conn-man-4
 [91]: http://linux.die.net/man/7/tcp
 [92]: #return-note-conn-man-5
 [93]: #return-note-conn-man-6
