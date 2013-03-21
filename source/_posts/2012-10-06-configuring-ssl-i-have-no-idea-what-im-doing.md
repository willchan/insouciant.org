---
title: 'Configuring SSL &#8211; I have no idea what I&#8217;m doing'
author: willchan
layout: post
categories:
  - tech
---
{% img /images/2012/10/sslihavenoideawhatimdoing.png %}

When I first set up this server, I went to StartSSL to get a certificate. Not having done this ever before, I made a number of errors. First, I had StartSSL generate my private key for me. Probably a bad idea, I hope they don’t record that :P Second, I had them generate a 4096 bit key rather than a 2048 bit key. I had figured, the bigger the better, right? Well, in load testing this wimpy micro EC2 server, I found that the majority of the CPU usage is in nginx, and I have to imagine that it’s in the SSL handshake. Oops. I should have read [this Stack Overflow thread][1] first I guess. I went to revoke my certificate today, but apparently they charge $25 per revocation. Oh well, I don’t expect much traffic anyway, and I’ll just change it when my certificate expires in a year. Lesson learned.

 [1]: http://stackoverflow.com/questions/589834/what-rsa-key-length-should-i-use-for-my-ssl-certificates
