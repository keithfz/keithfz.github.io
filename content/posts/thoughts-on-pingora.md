+++
title = "Thoughts around Pingora"
date = "2025-07-07"
author = "keithfz"
description = "Or is Envoy the jenkins of cloud-native networking"
+++

# Envoy is middle-aged 

I will preface this by saying that I love Envoy. It is truly a revolutionary design that I use as an engineer, and consumer of the interent, daily.

However, Envoy Proxy was initially developed in 2015. Kubernetes had just had it's first commit the year prior in June 2014. Rust 1.0 was release in 2015 as well. What's the point? Idk but it feels like the landscape is different now than it was in 2015. I know that's kind of obvious and a silly statement to make. Matt Klein said it himself, "Would I choose C++ today? No. However, today is not early 2015, eons ago in the technology world."

Sometimes, Envoy's age shows up. C++ wouldn't have been the first choice today. A fresh build without a Bazel cache takes an eternity. But most annoying of all, it's configurations are cumbersome. We're abstracted away from most of it nowadays, with Envoy Gateway or Istio being used as a control plane for fleets of envoy proxies. But for example, here is what the official documentation describes as "a minimal fully static bootstrap config":

```yaml
admin:
  address:
    socket_address: { address: 127.0.0.1, port_value: 9901 }

static_resources:
  listeners:
  - name: listener_0
    address:
      socket_address: { address: 127.0.0.1, port_value: 10000 }
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: ingress_http
          codec_type: AUTO
          route_config:
            name: local_route
            virtual_hosts:
            - name: local_service
              domains: ["*"]
              routes:
              - match: { prefix: "/" }
                route: { cluster: some_service }
          http_filters:
          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
  clusters:
  - name: some_service
    connect_timeout: 0.25s
    type: STATIC
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: some_service
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: 127.0.0.1
                port_value: 1234

```

I get it, I can read it and it makes sense, I understand why it is the way it is, but that doesn't make it any less ugly. Another example of bloat -- Why on earth does there need to be a DynamoDB filter in the official Envoy codebase? (Yes I know it's under `/contrib`, but it's still funny to me)

I was thinking about this because I was poking around with Pingora, a library for building networking services that was open-sourced by Cloudflare. It provides a library that can be used to build bespoke networking tools rather than a one-size fits all.

Would it make sense for networked services in today's topology to go back and embody more of the Unix philosophy instead of doing it all -- handling north-south, east-west, service mesh, load balancing, circuit breaking, observability, and so on? I don't have any answers, someone smarter probably does, but it seems under-explored and something that deserves a nudge. And I'm not totally alone in thinking this way. Istio's new Ambient mesh was built from the ground up, tailored specifically to be a node-level proxy in a service mesh, and intentionally has a minimalist philosophy. This is an evolution out of sidecar mode, to resolve many of the pain points around running Envoy as a sidecar, such as lifecycle management, resource utilization, etc.

A lot of the tooling is there -- We can orchestrate easier with Kubernetes as our control plane, and now we have a pretty neat little library that can act as little lego blocks for building something that can receive some packets, do a bunch of custom stuff with them, then shove them somewhere else. So we could at least try that approach and see what happens. Alternatively, it could just end in a web of half-baked microservices that are held together by duct tape and toothpicks. The truth is probably somewhere in the middle there. In any case, it's mainly a fun excuse for me to program in Rust a little bit more.

