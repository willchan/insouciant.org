---
title: Creating a self-hosted website is hard
author: willchan
layout: post
categories:
  - tech
tags:
  - ssl
  - wordpress
comments: true
---
I didn’t realize it’d be so hard. It’s a good experience though. Getting SSL set up correctly was a bit of a pain, and I still don’t know if I’ve got it done right :) Hopefully my coworkers can point out my silly mistakes. Apparently StartSSL won’t give me a cert until insouciant.org has been around for a few days, so I’ll just have to use a self-signed cert for now. Getting all the various WordPress tweaks to make HTTPS work, and enable caching headers (which are still sadly off, according to redbot.org, but close enough), and employ minification and all those other techniques was fun but time-consuming. Shouldn’t this all just work out of box?

I’m also kinda confused what people do with the WordPress admin page before they set up SSL (assuming they even set up SSL). Isn’t that just some auth token in the cookie that is sent in the clear, so it could easily be stolen? I made sure to set up SSL quickly, but given that I’m using a self-signed cert, I obviously could get MITM’d easily, but at least that’d require an active attacker.

Anyway, I learned a lot, which I guess was the point of this exercise :P
