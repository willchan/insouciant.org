---
title: '&#8220;Server Push&#8221; Already Exists, But SPDY Server Push Is Better'
author: willchan
layout: post
categories:
  - tech
tags:
  - spdy
comments: true
---
SPDY server push is one of the most poorly understood parts of SPDY. When people hear that the protocol supports the server pushing resources to the client, some of them are excited by the possibilities, but many are scared that SPDY will allow the server to push undesired content. What many people don’t realize is that “server push”, where the server is able to push content down to the client that it may not have explicitly requested [yet], already exists. It’s called [resource inlining][1]. Servers already sometimes take an external resource (e.g. scripts, stylesheets, and images) and directly inline it into the document (via inline  or  blocks, or data URIs). SPDY server push is superior to this approach in a number of ways. Here are a few:

 [1]: https://developers.google.com/speed/docs/pss/InlineSmallResources

*   Inlined resources cannot be cached separately from the document by the client. They are directly inlined into the document, which is usually kept uncacheable. People often complain that SPDY server push may push content that is already in the browser cache, but if servers instead inline resources, then the resource will never be cached. At least for SPDY server push, the resource may be in the cache and the client can use a RST_STREAM frame to race to cancel the pushed stream.
*   Inlined resources cannot be shared across documents. If your website has multiple references to a resource, it’s desirable to reference them via the same URL for caching purposes. This reduces unnecessary redundancy across the pages on the website.
*   Inlined resources may be poorly prioritized. Generally speaking, browsers will try to prioritize the document over external resources like stylesheets, scripts, and images, since the document will usually allow discovering and downloading more resources sooner, thus speeding up resource loading. However, when originally external resources are inlined directly into the document, then their content will preempt the rest of the content in the document. SPDY server push gives the server the ability to advertise that it will push some of the externally referenced resources, but rather than immediately pushing them before the rest of the document completes, as would happen with resource inlining, it can wait until the document finishes before pushing resources.
*   For clients that truly do not want to be pushed content, they can disable it by setting [SETTINGS\_MAX\_CONCURRENT_STREAMS][2] to 0. On the other hand, clients are not able to disable resource inlining. Clients should be careful about disabling SPDY server push, since if SPDY server push becomes too unreliable, then servers may instead go back to resource inlining despite its downsides.

 [2]: http://dev.chromium.org/spdy/spdy-protocol/spdy-protocol-draft3#TOC-2.6.4-SETTINGS

Fundamentally, once a client makes a request to a server, the server can send *anything* back in its response. For the servers that want to reduce roundtrips by pushing content instead, SPDY server push is a superior mechanism to resource inlining.

PS: I’m pleased to note that, despite earlier signs that SPDY server push may be removed from HTTP/2, at the interim httpbis meeting in Tokyo, everyone agreed that server push should stay in the spec.
