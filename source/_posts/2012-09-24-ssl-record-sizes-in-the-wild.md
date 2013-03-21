---
title: SSL record sizes in the wild
author: willchan
layout: post
categories:
  - tech
tags:
  - performance
  - ssl
---
I decided to set up this website as a learning experience, since I don’t have any firsthand experience with how the world outside Google deploys sites. I thought it’d be fun to set it up as an https site so I could learn more about SSL deployments. Remembering one of Mike’s rants about [the need to tune SSL record sizes][1], I wanted to make sure I configured nginx to do this properly. However, a configuration option doesn’t seem to exist! I shot Igor an email to ask him if this was true, and he confirmed it. He also added his thoughts on the appropriate record size:

 [1]: http://www.belshe.com/2010/12/17/performance-and-the-tls-record-size/

“16K is certiainly large. However, I think that 4K is more suitable than 2K.  
I believe all SSL libraries will eventually enable TLS random padding  
(AFAIK currently only GnuTLS does it). The 0-255 bytes padding size plus  
about 30 bytes SSL header HMAC add 1-14% overhead per 2K data record.  
For 4K data record the overhead is 1-6%.

4K of data and up to 224 of SSL overhead can be sent in 3 typical 1440-bytes  
ethernet packets.”

Since at least nginx doesn’t provide a configuration option, I was curious what websites out in the wild do. Here are some packet traces from some random websites I examined.

UPDATE (Oct 24, 2012): andrew@nginx.com tells me you can update NGX\_SSL\_BUFSIZE in the code.

    nginx-1.3.7 $ grep -n NGX_SSL_BUFSIZE src/event/ngx_event_openssl.h
    96:#define NGX_SSL_BUFSIZE  16384

Facebook is pretty good:[![][3]][3]

 []: http://insouciant.org/wp-content/uploads/2012/09/tls-trace-facebook.com_.png

My site sucks since I have no way to control this in nginx:[![][4]][4]

 []: http://insouciant.org/wp-content/uploads/2012/09/tls-trace-insouciant.org_.png

I happened to be reading an article on DigitalOcean (dunno who they are), so I grabbed a trace that turned out to be remarkably bad:[![][5]][5]

 []: http://insouciant.org/wp-content/uploads/2012/09/tls-trace-www.digitalocean.com_.png

wordpress.com also has large record sizes:[![][6]][6]

 []: http://insouciant.org/wp-content/uploads/2012/09/tls-trace-wordpress.com_.png

I’m kinda worried that most widely available server software that terminates SSL probably does not allow configuration of SSL record sizes, nor provide reasonable defaults (from a web performance perspective). Yet another reason why Mike always says SSL is the unoptimized frontier.
