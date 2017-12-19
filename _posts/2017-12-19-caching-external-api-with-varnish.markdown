---
layout: post
title:  "Caching an external HTTP API with Varnish"
date:   2017-12-19 00:00:00 +0000
category: Programming
tags: [HTTP, caching]
---

I recently participated in a connect four AI challenge. The goal was to write
a bot in Python that played connect four against other bots. Since connect four
is a strongly solved game, the optimal move can be bruteforced. I looked online
for an API that did this, and found [this online connect four game](http://connect4.gamesolver.org/?pos=).
When I tried to implement this in my own Python script, it sometimes took more
than a second, and got killed by the competition server. I then decided to cache
the answers the API gives on a local server with Varnish.

# Config

```
vcl 4.0;

# Default backend definition. Set this to point to your content server.
backend default {
    .host = "50.19.250.53";
    .port = "80";
}

sub vcl_backend_response {
  set beresp.ttl = 5000h;
}

sub vcl_recv {
    # Happens before we check if we have this in cache already.
    #
    # Typically you clean up the request here, removing cookies you don't need,
    # rewriting the request, etc.
    set req.http.host = "connect4.gamesolver.org";
}
```

The IP address is the IP address of the server that we are caching.
