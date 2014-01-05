---
layout: post
title: "Prioritization Only Works When There's Pending Data to Prioritize"
date: 2014-01-07 19:42
comments: true
categories: 
  - tech
tags:
  - chromium
  - SPDY
---
_\[ TL;DR: It's possible to write data to sockets, even when they will not immediately be sent. Committing data too early can result in suboptimal scheduling decisions if higher priority data arrives later. If multiplexing different priority data onto the same TCP connection (e.g. SPDY, HTTP/2), consider using the [TCP_NOTSENT_LOWAT socket option / sysctl](http://lwn.net/Articles/560082/) on relevant platforms to reduce the amount of unsent, buffered data in the kernel, keeping it queued up in user space instead so the application can apply prioritization policies as desired. \]_

## <a id="The Problem">[The Problem](#The_Problem)</a>

A large part of my job is helping different teams at Google understand their frontend networking performance. Last year, one of the teams that asked for debugging assistance was the Google Maps team, since they were launching the new version of Google Maps. Amongst the resources loaded on the page were the minimaps used in parts of the UI and the actual tiles for the map, which is the main content they care about. So what they would like is for the tiles to have a higher priority than the minimaps.

Recall that browsers will assign [priorities to resources by their resource type](/tech/resource-prioritization-in-chromium/). The new Google Maps will load these minimaps as image resources, whereas they'll fetch the tile data using XHRs. Chromium, if not all modern browsers, will give image resources a lower priority than XHRs. Moreover, Google Maps uses HTTPS by default, so SPDY-capable browsers, like Chromium, should see higher priority responses preempt lower priority responses, assuming response data for a higher priority stream is available at the server. Yet, Michael Davidson (Maps engineer) claimed that this wasn't happening. Better yet, he had [Chromium net-internals logfiles](http://dev.chromium.org/for-testers/providing-network-details) to prove it. He had one with a fast connection (probably on the Google corp network) and one with a simulated 3G connection. Let's look at what they said for the fast connection: "The requests that we really care about are stream\_id 45 and 47. These are requested via XHR, and are the tiles for the site. Note that Chrome is servicing 33-43 before we get a response for 45 and 47." Here are the relevant snippets of the net-internals logfile, with the unimportant fluff elided. Note that "[st]" refers to start time in milliseconds since the net-internals log started.

    [st=1996]  SPDY_SESSION_SYN_REPLY
               --> fin = false
               --> :status: 200 OK
                   content-length: 12354
                   content-type: image/jpeg
               --> stream_id = 33
    [st=1996]  SPDY_SESSION_RECV_DATA
               --> fin = false
               --> size = 1155
               --> stream_id = 33
    // ...
    [st=1998]  SPDY_SESSION_RECV_DATA
               --> fin = true
               --> size = 0
               --> stream_id = 33
    // ... (I've cut out the rest of the minimap image responses for brevity)
    [st=2004]  SPDY_SESSION_RECV_DATA
               --> fin = true
               --> size = 0
               --> stream_id = 43
    [st=2004]  SPDY_SESSION_SYN_REPLY
               --> fin = false
               --> :status:0 OK
                   content-length: 14027
                   content-type: image/jpeg
               --> stream_id = 41
    [st=2004]  SPDY_SESSION_RECV_DATA
               --> fin = false
               --> size = 4644
               --> stream_id = 41
    // ...
    [st=2004]  SPDY_SESSION_RECV_DATA
               --> fin = true
               --> size = 0
               --> stream_id = 41

    // At this point, the minimap image responses are done.
    // 77~ms of network idle time later, we start receiving the tile responses.

    [st=2081]  SPDY_SESSION_SYN_REPLY
               --> fin = false
               --> :status: 200 OK
                   content-type: application/vnd.google.octet-stream-compressible; charset=x-user-defined
               --> stream_id = 45
    [st=2085]  SPDY_SESSION_RECV_DATA
               --> fin = false
               --> size = 8192
               --> stream_id = 45
    // ... The data is rather large and keeps streaming in over 100ms. At which point the other tile
    //     response becomes available and starts interleaving since they're the same priority.

    [st=2187]  SPDY_SESSION_RECV_DATA
               --> fin = false
               --> size = 3170
               --> stream_id = 45
    [st=2188]  SPDY_SESSION_SYN_REPLY
               --> fin = false
               --> :status: 200 OK
                   content-type: application/vnd.google.octet-stream-compressible; charset=x-user-defined
               --> stream_id = 47
    [st=2188]  SPDY_SESSION_RECV_DATA
               --> fin = false
               --> size = 141
               --> stream_id = 47
    ...
    [st=2194]  SPDY_SESSION_RECV_DATA
               --> fin = false
               --> size = 1155
               --> stream_id = 45
    
    [st=2247]  SPDY_SESSION_RECV_DATA
               --> fin = true
               --> size = 0
               --> stream_id = 47
    
    [st=2263]  SPDY_SESSION_RECV_DATA
               --> fin = true
               --> size = 0
               --> stream_id = 45

Indeed, Michael is quite correct that the response data for streams 45 and 47 are arriving after the other streams. Now, the question is whether or not this is due to incorrect prioritization. It's interesting to note that there's a 70+ms gap from st=2004 to st=2081 where stream 41 has finished and the network goes idle for 70+ms before starting to send the response for stream 45. This makes the incorrect prioritization hypothesis somewhat suspect. At this point, it'll be useful to describe how a common SPDY deployment works. Let me borrow the NSA's useful diagram of Google's serving infrastructure:

<figure>
{% img /images/2014/01/nsa-smiley-face-580.jpg %}
<figcaption>
Both SSL and SPDY are added and removed at GFE. GFE will demux incoming requests on the client SPDY connection to backends (application frontends, like Maps) and mux the backend responses onto the client SPDY connection.
</figcaption>
</figure>

           Client                          Google
    
                                |               +---------+
                                |               |         |
                                |          +--->| Backend |
                                |          |+--+|         |
                 Image reqs     |          +v   +---------+
    +---------+  Tile reqs      |     +-----+
    |         |+----------------|---->|     |+------>.
    | Browser |                 |     | GFE |        . More backends
    |         |<----------------|----+|     |<------+.
    +---------+  Image resps    |     +-----+
                 Tile resps     |          ^+   +---------+
                                |          |+-->|         |
                                |          +---+| Backend |
                                |               |         |
                                |               +---------+
    
    
                         Backend Response Queues
                           (X represents data)
    
    +--------------+                                +-----------+
    |              | Queue                          |           |
    |              | XXXXXXXXXX       X      X    X | Backend 1 |
    |              |<------------------------------+|           |
    |              |                                +-----------+
    |              |
    |              |                                +-----------+
    |              | Queue                          |           |
    |              | XXXXX        X        X        | Backend 2 |
    |              |<------------------------------+|           |
    |      GFE     |                                +-----------+
    |              |
    |              |
    |              |                                     ...
    |              |
    |              |
    |              |                                +-----------+
    |              | Queue                          |           |
    |              |                                | Backend X |
    |              |<------------------------------+|           |
    +--------------+                                +-----------+

Hopefully these diagrams highlight why the 70+ms delay between the minimap image responses and the tile responses makes the hypothesis that prioritization is broken seem less likely. Request prioritization will only work if there is data from multiple responses to choose from. If there's no higher priority data to choose to prioritize, then prioritization cannot have any effect. For example, if the image responses were coming from backends 1 & 2 in the above diagram, but the higher priority tile responses were coming from backend X, then given the backend response queues in the diagram, the GFE would only be able to respond with data from backends 1 and 2, and has no data from backend X to stream instead. That's why it's key to note the 70+ms delay between the image responses and the tile responses. That implies that backend responses for the tiles only arrived at the GFE _after_ the image responses had already been forwarded onward to the browser. There are other possible explanations like TCP delays and what not, but they're less likely for various reasons I won't bother explaining here. Theoretically, if the GFE wanted to strictly enforce prioritization, it could hold up the image responses until the tile responses arrived, but that's generally a pretty silly thing to do, as it implies wasting available bandwidth, since it's likely the peer can still process lower priority responses while waiting for higher priority responses to arrive.

The hypothesis that prioritization does not happen because the higher priority responses only arrive later after the lower priority responses have already been drained is a reasonable hypothesis given the network characteristics here (the corporate network has low latency and high bandwidth to Google datacenters). That of course raises the question of what would happen were we to use a slower network connection, which might conceivably allow backend responses to build up in GFE queues. Luckily, Michael also had a net-internals log for that case, using a simulated 3G connection. Michael helpfully identifies the noteworthy streams in the log for us: "Stream ID 51 is the tiles we care about. 49 is an image that we don't care about. We don't get any data for 51 until 49 completes. In the network tab, this appears as 'waiting' time. We waited 1.3 seconds for 51, and 3.8 seconds for 53!"

    [st= 4085]  SPDY_SESSION_SYN_REPLY
                --> fin = false
                --> :status: 200 OK
                    content-length: 13885
                    content-type: image/jpeg
                --> stream_id = 49
    [st= 4099]  SPDY_SESSION_RECV_DATA
                --> fin = false
                --> size = 1315
                --> stream_id = 49
    ...
    [st= 4231]  SPDY_SESSION_RECV_DATA
                --> fin = true
                --> size = 0
                --> stream_id = 49
    [st= 4232]  SPDY_SESSION_SYN_REPLY
                --> fin = false
                --> :status: 200 OK
                    content-type: application/vnd.google.octet-stream-compressible; charset=x-user-defined
                --> stream_id = 51

As we can see here, the responses are clearly back to back. Due to the low bandwidth and high RTT of the simulated 3G connection, it takes over 100ms to stream the response for the minimap image in stream 49. A millisecond after that stream finishes, the server immediately streams the higher priority response in stream 51. So either GFE's SPDY prioritization is broken (which would be quite a significant performance regression) or it still doesn't have the chance to take effect, even with this slow network. Since I've done a fair amount of performance debugging of Chromium-GFE SPDY prioritization before, I did not believe it was at fault here. Rather,  I suspected that we discovered our first concrete instance of a problem that Roberto and I had previously only theorized could occur - the TCP send socket buffer was buffering too much unsent data.

                                                   GFE
                    +---------------------------------------------------------------+
                    |                                                               |
                    |                                                               |XXXXX
                    |                                                               |<------------
                    |                                                               |  Backend 1
                    |                                                               |
                    |                                                               |XXXXXXXX
                    |                                                               |<------------
                    |                                                               |  Backend 2
                    |                                                               |
                    |       One of the GFE kernel socket buffers                    |XXX
                    |+------------------------------------------------+             |<------------
                    ||                                                |             |  Backend 3
                    ||                      limit of sendable data due|             |
                    ||                      to stuff like rwnd & cwnd |             |
                    ||                                   +            |             |
                    ||                                   |            |             |    ...
                    ||+----------------------------------v-----------+|             |
    Client<---------|||XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX|UUUUUUUUUUU||             |
                    ||+----------------------------------|-----------+|             |
                    ||              sent data             unsent data |             |<------------
                    || (kept for possible retransmission)             |             |  Backend X
                    |+------------------------------------------------+             |
                    +---------------------------------------------------------------+

Once a proxy receives data from the previous hop, it will likely try to forward it onward to the next hop if the outgoing socket is writable. Otherwise, it will have to queue up the data and possibly assert flow control on the previous hop to prevent it from having to queue up too much data (which would eventually lead to OOM). If the outgoing TCP connection is a SPDY connection, and thus supports prioritized multiplexing, the proxy will probably mux the data onto the same SPDY connection from the incoming queues in more or less priority order.

Of course, this raises the question of when the TCP socket is writable. That's a fairly complicated question, as it is based on variables like the peer's receive window, the estimated [BDP](http://en.wikipedia.org/wiki/Bandwidth-delay_product), and other settings like the socket send buffer size. On the one hand, if the sender wants to fully saturate the bandwidth, it needs to buffer at least a BDP's worth of data in the kernel socket buffer, since it needs to keep it around to retransmit in case of loss. On the other hand, it doesn't want to use an excessive amount of memory for the socket buffers. Yet, it may also want to minimize the computational costs of extra kernel-user context switches to copy data from user space to the kernel for transmission.

At the end of the day, depending on the various variables, a TCP socket may become writable _even if the TCP stack won't immediately write the data out to the network_. The data will simply sit in the kernel socket buffer until the TCP stack decides to send it (possibly when already sent data has been ACK'd). The problem here should be evident: given the current BSD interface for stream sockets (TCP sockets are stream sockets), there's no way to re-order the data (e.g. data for a higher priority stream just became available) once it has been written to the socket. And even if there were, that wouldn't help the typical SPDY case, since it runs over TLS, which uses the sequence number in its ciphers, so it can't reorder data that has already been processed by the TLS stack. Both the TLS stack API and the BSD stream socket API operate similarly to high latency FIFO queues (inserted at the sender end, and consumed at the network receiver's end). Therefore, in order to better prioritize data, it's desirable to delay committing the data to the TLS/TCP stacks until one knows that the data will be immediately sent out over the network. Unfortunately, the only signal a user space application has related to this is the socket becoming writable (aka [POLLOUT](http://pubs.opengroup.org/onlinepubs/009695399/functions/poll.html)), meaning the kernel is willing to accept new data into its buffers, although it may not want to or be able to send the data out immediately. Furthermore, POLLOUT does not indicate to the user space application how much data it can write. This poses a difficulty for the application developer who does not know how much data to feed into the TLS/TCP stack. An obvious solution is to simply write smaller chunks of data to the socket, but this of course requires more syscalls.

## <a id="The Solution">[The Solution](#The_Solution)</a>

Anyhow, at this point, I voiced my suspicions and my wonderful colleagues Hasan and Roberto (GFE devs) investigated the hypothesis and confirmed that it was indeed the case. Great! But now what do we do? Well, as I stated earlier, what we want is for the kernel to only mark the socket as writable if the data will be promptly written out to the network. Good thing our resident Linux networking ninja Eric [added this feature to the Linux kernel](http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=c9bee3b7fdecb0c1d070c7b54113b3bdfb9a3d36). This feature allows controlling the limit of unsent bytes in the kernel socket buffer. Setting it as low as possible will enable a SPDY server to make the best possible scheduling decision (based on stream priorities), but will also incur extra computational costs from extra syscalls and context switches. As my colleague Jana [explains in detail](https://code.google.com/p/chromium/issues/detail?id=310927#c5):

> A bit of background: this socket buffer is used for two reasons -- for unacked data and for unsent data. TCP on a high Bandwidth-Delay Product (BDP) path will have a large window and will have a large amount of unacked data; setting this buffer too small limits TCP's ability to grow its sending window. TCP buffer autotuning is often used to increase the buffersize to match a higher BDP than the default (most OSes do autotuning now, and I don't think that they generally ever  *decrease* the buffer size.) As Will points out, this increasing buffer to match the BDP also ends up opening up more room for unsent data to sit in. Generally autotuning errs on the side of more buffersize so as to not limit the TCP connection, and this is at odds with an app that wants to retain scheduling decisions (such as SPDY with prioritized sending). TCP_NOTSENT_LOWAT is that knob that allows the app to decide how much unsent data ought to live in the socket buffer.
> 
> The simplest use of this sockopt is to set it to the amount of buffer that we want in the kernel (per our latency tolerance). But there's another tradeoff to be watchful of -- if the watermark is set to be too low, the TCP sender will not be able to keep the connection saturated and will lose throughput.  If an ack arrives and the socket buffer has no data in it, TCP will lose throughput. Ideally, you want to set the watermark high enough to exactly match the TCP drain rate ( the ack-clock tick, ~ cwnd/rtt) to the process's schedule rate (which is f(context switch time, load)).
> 
> Practically, you want the buffer to be small in low-bw environments (cellular) and larger in high-bw environments (wifi). The right metric to be considering in general is delay -- how long did a data segment spend in the socket buffer -- and we can use this to tweak the watermark. (An app can measure this by measuring time between two POLLOUTs, given that we set the low watermark, and send data into the socket above the low watermark).

So far in this discussion, I've focused almost entirely on the HTTP response path from SPDY origin server. That's simply because this is the case that typically matters most for this issue. This case can also happen on the upstream path from a client to a server. For example, let's say you have a client that is uploading a large file (maybe sync'ing data on Google Drive perhaps?) up to a server over a SPDY connection. That's a bulk data transfer and theoretically should be assigned a low priority. It cares about throughput so it will try to hand off data to the kernel ASAP so that the kernel buffers never drain and it maximizes the network throughput. However, if the client tries to send interactive requests over the same SPDY connection, then those requests may sit in the client kernel send socket buffer behind the bulk upload data, which would be unfortunate. Clients usually have an advantage here over servers, in that the extra kernel-user context switch and syscall costs are generally not significant issues, at least in comparison to highly scalable servers.

More generally speaking, this is very analogous to [bufferbloat](http://www.bufferbloat.net/projects/bloat/wiki/Introduction). In the typical bufferbloat scenario, multiple network flows of varying priority levels share a network queue. The way TCP works, it will try to maximize throughput (and thereby fill the buffers/queues) until a congestion event occurs (packet loss being the primary signal here), which leads to queueing delays for everyone else. The queueing delays especially hurts applications with higher priority network traffic, like videoconferencing or gaming, where high-latency/jitter can seriously impact the user experience. In the SPDY prioritization case above, the shared network queue is actually a single TCP connection, including not just the inflight data, but the data sitting in the kernel send and receive socket buffers, and the multiple flows are the different SPDY streams that are being multiplexed on the same TCP connection. And applications, much like TCP flows, will greedily fill the buffers if given the chance. Unlike how network devices treat packets, it's usually unacceptable for the kernel to discard unack'd application data that it received via the stream socket API, as the application expects it to be reliable.

Great, so now we understand both the problem and the solution, right? More or less that's true. The appropriate/available solution mechanism is the TCP_NOTSENT_LOWAT socket option / sysctl. AFAICT, it's available only on Linux and OS X / iOS based operating systems. The bigger question is _how_ to use this. As mentioned previously, there are tradeoffs. Lowering TCP_NOTSENT_LOWAT will keep more of the data buffered in user space instead, allowing the application to make better scheduling decisions. On the flip side, it may incur extra computation from increased syscalls and kernel-user context switches and, if set too low, may also underutilize available bandwidth (if the kernel socket buffer drains all unsent data before the application can respond to POLLOUT and write more data to the socket buffer). Since these are all workload dependent tradeoffs, it's hard to specific a single value that will solve everyone's problems. So, the answer is to measure and tune your application performance using this knob.
