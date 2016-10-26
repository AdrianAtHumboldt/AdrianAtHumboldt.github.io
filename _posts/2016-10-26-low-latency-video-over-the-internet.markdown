---
published: true
title: Low Latency Video over the Internet
layout: post
tags: [video]
---
Flash is dying, and I'm one of the few people who'll miss it. While Flash was and is a terrible security risk, it solved one problem almost completely by accident: sending video to many people with a low delay. Working with online auctions, delay means a lot to me. If the viewers are actually bidding in a real-time auction, a delay of a few seconds is a problem. If they're bidding against people who are actually at the auction, it's impossible. By the time the remote buyer hears a bid from the auction hall, buyers who are physically there can bid again several times.

The RTMP protocol used by Flash requires a direct TCP connection from the browser to the video server: this allows an end to end delay of less than a second, while making it impossible to set up a shared cache infrastructure or CDN. Modern streaming protocols such as HLS and DASH are CDN friendly, by breaking the video into segments of around 1 to 10 seconds. This segmentation forces an end-to-end delay of at least one segment duration.  

It's actually a little worse than that, because each segment should be an independent video file, and not depend on any previous file. So as the file duration gets shorter, the efficiency gets less. At the limit, if each segment was a single frame of video, it would be equivalent to sending a sequence of JPEG files.

The modern alternative is WebRTC, which provides browser APIs to set up an [RTP session](https://webrtc.org/architecture/).
 Unfortunately [support is currently limited to Firefox, Chrome, and Opera](http://caniuse.com/#search=webrtc).  Which means that the near future looks like a mixed world, with RTMP streaming to the Flash plugin in IE and Edge with WebRTC to Chrome and Firefox.

And iOS? WebRTC is marked as ["in development" for WebKit](https://webkit.org/status/#specification-webrtc), but we don't know when, or whether this will be available on iOS Safari. Until then Apple doesn't support any low-latency protocol in their web browser, so there's no alternative to an app for that.