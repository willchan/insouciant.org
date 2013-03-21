---
title: SSL Performance Case Study
author: willchan
layout: post
categories:
  - tech
tags:
  - ocsp
  - performance
  - ssl
---
Back when [Mike Belshe][1] was still at Google, he used to keep saying that SSL was the unoptimized frontier. Unfortunately, even years later, it still is. There’s low hanging fruit everywhere, and most folks, [myself included][2], don’t know what they’re doing. Tons of people are making basic mistakes. Anyone deploying a website served over HTTPS really ought to read [Adam Langley’s][3] [post on overclocking SSL][4]. There is a lot of useful information in his post. I’m going to call out a few of them really quickly as they pertain to latency:

 [1]: https://twitter.com/mikebelshe
 [2]: /tech/configuring-ssl-i-have-no-idea-what-im-doing/
 [3]: https://plus.google.com/118082204714636759510/posts
 [4]: http://www.imperialviolet.org/2010/06/25/overclocking-ssl.html

*   Send all necessary certificates in the chain – save the DNS TCP HTTP fetch for missing ones.
*   Try to keep the certificate chain short – initcwnd is often small and takes a while to ramp up due to [TCP slow start][5].
*   Don’t waste bytes sending the root certificate – it should already be in the browser certificate store, otherwise it won’t pass certificate verification anyway.
*   [Use reasonable SSL record sizes][6] - user agents can only process each record as a whole, so don’t spread them across too many TCP packets or you’ll incur unnecessary delay (potentially roundtrips).

 [5]: http://www.igvita.com/2011/10/20/faster-web-vs-tcp-slow-start/
 [6]: http://www.belshe.com/2010/12/17/performance-and-the-tls-record-size/

## Case Study – CloudFlare

I decided to analyze a website ([CloudFlare][7]) to demonstrate how these rules can make a difference in SSL performance. I figured CloudFlare would be a good example, because with their [new OCSP stapling support][8], I could demonstrate how that saved a serial OCSP request in exchange for increased cwnd pressure. Remember though, as [Patrick McManus points out][9], [missed][10] [optimizations][11] [are][12] [everywhere][13], so definitely don’t view this case as the exception.

First step, I [loaded it up in WebPageTest][14] (thanks [Pat][15] for a great product!). and looked at the SSL connection for the main document:

{% img /images/2012/11/cloudflaresslconnect.png %}

Here we go, we see that it takes around 800ms to finish the SSL handshake to https://www.cloudflare.com, including 2 OCSP requests to verify the certificates in the certificate chain. Let’s dive into the SSL handshake to see what’s taking all the time. To do so, I take the [tcpdump from WebPageTest][17] and [feed it into CloudShark][18] (learned about this webapp recently, ain’t it cool?). The TCP stream index for the main document is 4, so I use [`tcp.stream eq 4` in CloudShark][19] to follow that TCP connection. Let’s check out the handshake messages.

{% img /images/2012/11/cloudflaresslcertmsg.png %}

Unfortunately, it appears that the Certificate Certificate Status ([OCSP stapling][8]) messages for CloudFlare are too large and thus overflows [initcwnd][21], thus we see frame 79 arrive a full RTT after frame 62. To see why this is the case, we dive into the actual certificate chain.

{% img /images/2012/11/cloudflaresslcertmsgdetails.png %}

Yeah, so we see that the Certificate chain is 7123 bytes and consists of 5 certificates. Unfortunately, viewing this certificate chain in CloudShark makes it difficult to see what each certificate is signing. To see it more clearly, I turn to the openssl command line utility and run a command like `openssl s_client -connect www.cloudflare.com:443`, which gives me output that includes this blurb:

 [7]: https://www.cloudflare.com
 [8]: http://blog.cloudflare.com/ocsp-stapling-how-cloudflare-just-made-ssl-30
 [9]: https://plus.google.com/100166083286297802191/posts/PLPDyKCNrF5
 [10]: http://www.webpagetest.org/result/121111_VN_D0V/1/details/
 [11]: http://www.cloudshark.org/captures/fcd7d9c0d36b?filter=tcp.stream eq 3
 [12]: http://www.webpagetest.org/result/121111_N9_D15/1/details/
 [13]: http://www.cloudshark.org/captures/9515d102095d?filter=tcp.stream eq 4
 [14]: http://www.webpagetest.org/result/121103_34_DGS/
 [15]: https://twitter.com/patmeenan
 [17]: http://www.webpagetest.org/result/121103_34_DGS/1.cap
 [18]: http://www.cloudshark.org/captures/addc3cc7e6fc
 [19]: http://www.cloudshark.org/captures/addc3cc7e6fc?filter=tcp.stream eq 4
 [21]: http://www.cdnplanet.com/blog/tune-tcp-initcwnd-for-optimum-performance/

    Certificate chain
    0 s:/1.3.6.1.4.1.311.60.2.1.3=US/1.3.6.1.4.1.311.60.2.1.2=Delaware/businessCategory=Private Organization/serialNumber=4710875/C=US/ST=California/L=Palo Alto/O=CloudFlare, Inc./OU=Internet Security and Acceleration/OU=Terms of use at www.verisign.com/rpa (c)05/CN=www.cloudflare.com
      i:/C=US/O=VeriSign, Inc./OU=VeriSign Trust Network/OU=Terms of use at https://www.verisign.com/rpa (c)06/CN=VeriSign Class 3 Extended Validation SSL CA
    1 s:/C=US/O=VeriSign, Inc./OU=VeriSign Trust Network/OU=Terms of use at https://www.verisign.com/rpa (c)06/CN=VeriSign Class 3 Extended Validation SSL CA
      i:/C=US/O=VeriSign, Inc./OU=VeriSign Trust Network/OU=(c) 2006 VeriSign, Inc. - For authorized use only/CN=VeriSign Class 3 Public Primary Certification Authority - G5
    2 s:/C=US/O=VeriSign, Inc./OU=VeriSign Trust Network/OU=(c) 2006 VeriSign, Inc. - For authorized use only/CN=VeriSign Class 3 Public Primary Certification Authority - G5
      i:/C=US/O=VeriSign, Inc./OU=Class 3 Public Primary Certification Authority
    3 s:/C=US/O=VeriSign, Inc./OU=VeriSign Trust Network/OU=Terms of use at https://www.verisign.com/rpa (c)06/CN=VeriSign Class 3 Extended Validation SSL CA
      i:/C=US/O=VeriSign, Inc./OU=VeriSign Trust Network/OU=(c) 2006 VeriSign, Inc. - For authorized use only/CN=VeriSign Class 3 Public Primary Certification Authority - G5
    4 s:/C=US/O=VeriSign, Inc./OU=VeriSign Trust Network/OU=(c) 2006 VeriSign, Inc. - For authorized use only/CN=VeriSign Class 3 Public Primary Certification Authority - G5
      i:/C=US/O=VeriSign, Inc./OU=VeriSign Trust Network/OU=(c) 2006 VeriSign, Inc. - For authorized use only/CN=VeriSign Class 3 Public Primary Certification Authority - G5

I took a look at this before and was very confused about what was going on. My colleague [Sir Ryan of Sleevi][23], who is always educating me on the grotty corner cases of SSL (including several chunks which appear in this post), helped me grok this by providing this helpful snippet:

    [0] www.cloudflare.com
      [1] - Class 3 Extended Validation SSL CA
        [2] - Class 3 Public Primary CA - G5 [cross-signed version]
          [omitted] - Class 3 Public Primary CA - [legacy root]
      [3] - Class 3 Extended Validation SSL CA
        [4] - Class 3 Public Primary CA - G5 [self-signed version]

 [23]: https://plus.google.com/105761279104103278252/posts

Really fascinating! \[3\] is actually a duplicate of \[1\], and \[4\] is a root certificate, which user agents should really have in their certificate store, so it’s superfluous and also violates the [TLS 1.0 spec][24], section 7.4.2, which says:

    This is a sequence (chain) of X.509v3 certificates. The sender's
    certificate must come first in the list. Each following
    certificate must directly certify the one preceding it. Because
    certificate validation requires that root keys be distributed
    independently, the self-signed certificate which specifies the
    root certificate authority may optionally be omitted from the
    chain, under the assumption that the remote end must already
    possess it in order to validate it in any case.

 [24]: http://tools.ietf.org/html/rfc2246

Yeah. So CloudFlare’s setup is weird, violates the spec (\[3\] signs \[0\] instead of \[2\]), and leads to a bloated certificate chain that contributes to overflowing their initcwnd. I reached out to them and they confirmed that this was a configuration error that they would be fixing. Great!

Sadly though, it’s the OCSP stapled response that apparently pushes them over the edge. The CertificateStatus message is 1995 bytes, spilling over both frames 76 and 79. The ServerKeyExchange and ServerHelloDone messages on the other hand are only 331 and 4 bytes respectively, and without the CertificateStatus message, they would have fit in frame 76.

{% img /images/2012/11/cloudflaresslserverhellodone.png %}

It’s rather unfortunate that this is the case, but at least the OCSP stapled response is saving them a OCSP request (DNS TCP HTTP). You can already see how much (500 ms?) the 2 OCSP requests that Windows’ cert verifier makes in requests 2 and 3 stall the SSL handshake for request 1 (the root document) in the [waterfall][26], so as this example demonstrates, OCSP stapling is totally better than an OCSP request. Also, once CloudFlare fixes their broken certificate chain, they probably won’t overflow initcwnd anymore. But man, wouldn’t it be great if we didn’t do these requests in the first place? I mean, [how much value do they really add][27]? Well, one option is to use [CRLSets][28] (which don’t apply in this case for EV certs, not to mention on WebPageTest the CRLSets haven’t been downloaded yet, as they are downloaded separately after first run).

Anyway, back to [TCP stream 4][19]. Now that we’ve finished the SSL handshake, we’re good to go, right? Well, there are other things to watch out for. For one, we want to make sure that the SSL record sizes are small enough so we don’t unnecessarily delay the user agent from being able to process the HTTP response body. Unfortunately, we see that this is indeed happening:

{% img /images/2012/11/cloudflaresslrecordsize.png %}

9k records split across 7 packets, leading to a 45~ms delay for the first packet of data with 622 bytes. I haven’t looked more thoroughly, but it’s likely that this kind of pattern repeats itself during the lifetime of this connection. Hopefully it won’t lead to roundtrips, especially once the congestion window ramps up a bit more.

Looking at the [next SSL connection for ajax.cloudflare.com][30], we see some interesting SSL performance benefits/losses:

{% img /images/2012/11/cloudflareajaxsslconnection.png %}

In contrast to the SSL connection for www.cloudflare.com, the certificate chain for ajax.cloudflare.com (rooted at GlobalSign) is relatively small (2495 bytes), and fits nicely in the initial congestion window. CloudFlare also staples the OCSP response there too, saving a request which is great, except that serving up the OCSP stapled response seems to take a shocking 280ms. I’m confused at what’s happening here, since when you examine the CertificateStatus message, it’s supposed to be 1507 bytes long, and frames 186 and 189 contain 330 and 1176 bytes respectively, which adds up to 1506 bytes…which is rather unfortunate. It doesn’t appear to be an initcwnd issue since the 280ms delay is way above the RTT. It’s suspiciously near a RTT after the delayed ACK in frame 228. Indeed, this pattern seems to repeat itself [multiple][32] [times][33], so it’s not a one time glitch. It smells of some sort of implementation detail. I reached out to CloudFlare earlier and they’re looking into it. In any case, on the upside, Chrome manages to save a roundtrip here because CloudFlare uses both forward secrecy and [NPN][34] (not to mention [SPDY][35]!), so Chrome [can use][36] [False Start][37]. Nice!

Next, we examine the [SSL connection for ssl.google-analytics.com][38]. It’s rather boring because everything just works great. The Certificate message is 1756 bytes long, there’s no online revocation check, and Google advertise NPN and uses forward secrecy, so Chrome can use False Start and save a roundtrip. And Google uses SSL records capped at 1345 bytes, which generally prevents records from spanning more than one packet. All in all, it looks pretty optimized to me.

Lastly, there’s the [SSL connection for cdn01.smartling.com][39]. Now this one highlights another common misconfiguration problem: not including the intermediate certificates. You can see from the waterfall how costly this is:

{% img /images/2012/11/smartlingsslintermediatecert.png %}

Despite the obvious cost here, there’s an argument to be made that not including the [intermediate certificates][41] will keep the cert chain short, and if the OS is smart it’ll cache the intermediates, thus preventing the painful lookup. Obviously, some intermediate certificates are more likely to be cached than others. Also, to my knowledge, so far only Windows caches intermediate certificates. Given the tradeoffs, I’d still recommend including the intermediate certificates in general.

Interestingly enough, I doublechecked today, and Smartling seems to have [fixed their missing intermediate chain issue][42]. That’s awesome, except now they seem to be including too many certificates in their certificate chain and [exceed their tiny initcwnd][43] :( They should definitely remove the ValiCert self-signed cert, and possibly the Go Daddy Class 2 Certification Authority one (it’s included in my Mac’s root certificate store, but I’m not sure if it’s on all relevant devices). Actually, now that I look closer, Smartling didn’t actually change, but rather has two EC2 instances running, each with a different configuration (ec2-50-18-112-254.us-west-1.compute.amazonaws.com and ec2-23-21-68-21.compute-1.amazonaws.com). You can see their respective chains here:

    ---
    Certificate chain
     0 s:/O=*.smartling.com/OU=Domain Control Validated/CN=*.smartling.com
       i:/C=US/ST=Arizona/L=Scottsdale/O=GoDaddy.com, Inc./OU=http://certificates.godaddy.com/repository/CN=Go Daddy Secure Certification Authority/serialNumber=07969287
    ---

 [26]: http://www.webpagetest.org/result/121103_34_DGS/1/details/
 [27]: http://www.imperialviolet.org/2011/03/18/revocation.html
 [28]: http://www.imperialviolet.org/2012/02/05/crlsets.html
 [30]: http://www.cloudshark.org/captures/addc3cc7e6fc?filter=tcp.stream eq 8
 [32]: http://www.cloudshark.org/captures/a9df447dcc2a?filter=tcp.stream eq 6
 [33]: http://www.cloudshark.org/captures/53004cb01a2b?filter=tcp.stream eq 7
 [34]: https://technotes.googlecode.com/git/nextprotoneg.html
 [35]: http://blog.cloudflare.com/introducing-spdy
 [36]: http://src.chromium.org/viewvc/chrome?view=rev&revision=133255
 [37]: http://tools.ietf.org/html/draft-bmoeller-tls-falsestart-00
 [38]: http://www.cloudshark.org/captures/addc3cc7e6fc?filter=tcp.stream eq 9
 [39]: http://www.cloudshark.org/captures/addc3cc7e6fc?filter=tcp.stream eq 10
 [41]: https://www.wormly.com/help/ssl-tests/intermediate-cert-chain
 [42]: http://www.webpagetest.org/result/121111_W1_CCW/1/details/
 [43]: http://www.cloudshark.org/captures/67bfc6131271?filter=tcp.stream eq 2

    ---
    Certificate chain
     0 s:/O=*.smartling.com/OU=Domain Control Validated/CN=*.smartling.com
       i:/C=US/ST=Arizona/L=Scottsdale/O=GoDaddy.com, Inc./OU=http://certificates.godaddy.com/repository/CN=Go Daddy Secure Certification Authority/serialNumber=07969287
     1 s:/C=US/ST=Arizona/L=Scottsdale/O=GoDaddy.com, Inc./OU=http://certificates.godaddy.com/repository/CN=Go Daddy Secure Certification Authority/serialNumber=07969287
       i:/C=US/O=The Go Daddy Group, Inc./OU=Go Daddy Class 2 Certification Authority
     2 s:/C=US/O=The Go Daddy Group, Inc./OU=Go Daddy Class 2 Certification Authority
       i:/L=ValiCert Validation Network/O=ValiCert, Inc./OU=ValiCert Class 2 Policy Validation Authority/CN=http://www.valicert.com//emailAddress=info@valicert.com
     3 s:/L=ValiCert Validation Network/O=ValiCert, Inc./OU=ValiCert Class 2 Policy Validation Authority/CN=http://www.valicert.com//emailAddress=info@valicert.com
       i:/L=ValiCert Validation Network/O=ValiCert, Inc./OU=ValiCert Class 2 Policy Validation Authority/CN=http://www.valicert.com//emailAddress=info@valicert.com
    ---

## Conclusion

In this single case study, we see that HTTPS pages are full of missed optimizations. All the rules I listed in the beginning of the post are pretty basic, but nobody knows to look for them. I highly recommend examining your website’s SSL packet traces to look for these common mistakes. At least try taking advantage of existing [automated][44] [tools][45] to do some basic verification of your SSL configuration.

Thanks to my colleague [Ryan Sleevi][46] for reading through a draft of this post to make sure I didn’t have any glaring inaccuracies.

 [44]: https://www.ssllabs.com/ssltest/analyze.html?d=cdn01.smartling.com
 [45]: https://www.wormly.com/test_ssl/h/cdn01.smartling.com/i/204.236.224.156/p/443
 [46]: https://plus.google.com/105761279104103278252
