---
layout: post
category : notes
tagline: ""
tags : [research]
---

{% include JB/setup %}

Flow Caching for High Entropy Packet Fields_HotSDN_2014

*****

##What's the question

In this paper, we consider the problem of `flow caching` and more specifically, how to cache forwarding decisions that depend on `packet fields` with `high entropy` (and therefore, change often). In this paper, we consider tackling exactly this problem: how to compute `megaflow cache entries` such that we don’t converge to exact match connection caching even if the `slow path` operates over `L4 headers`.

*****

##Why study

Network virtualization and network function virtualization value general purpose CPUs exactly for their flexibility: in such systems, a single x86 forwarding element does not implement a single, static classification step but a sequence of dynamically reconfigurable and potentially complex forwarding operations. This leaves a software developer looking for maximal packet forwarding throughput with few options besides flow caching. 

*****

##Innovativeness

we arrive at algorithms that allow us to efficiently compute near optimal `flow cache entries` spanning several transport connections, even if forwarding decisions depend on transport protocol headers. 

*****

##Introduction

Most definitions emphasize the role of the network `edge` in providing the complex `packet processing` functions: `forwarding` elements in the network `core` are left with high speed, low latency, and power efficient transportation of packets from a network edge to another. The edge provides the complex packet operations implementing network services and policies relevant for end hosts.

Packet processing built on generic CPUs allows a single server to execute an `arbitrary sequence` of complex `packet processing` operations for a packet. Without leaving the server, a packet may traverse a simple `chain of services` (in the case of `NFV`) or a more complex `logical topology` of switches, routers, and services (in the case of `network virtualization`). The `lack of TCAMs` necessitates algorithmic classification which remains difficult to optimize for `cache hierarchies` of general CPUs.

In this paper, we consider the problem of computing flow cache entries for forwarding decisions that depend on `high entropy` packet fields. We loosely define high entropy packet fields as those which are likely to have `differing values` from packet to packet flowing through a switch. For example, all traffic originating from a particular host will likely have the same source and destination `MAC fields`, but the source and destination `L4 port fields` are likely to change from connection to connection. Thus, we say the L4 port fields are `higher entropy` than the L2 address fields.Our primary focus here is on `TCP/IP` and hence on the relatively high entropy transport port fields, but the proposed algorithms are more general and applicable to future protocols.

*****

##Open vSwitch Flow Caching

Many modern software forwarding products rely on `flow caching`. For instance, `Open vSwitch` recognized early the need for a `slow path` for `complex` forwarding logic (in `userspace`) and a `fast path` for `simple` forwarding logic merely `replicating decisions` of the slow path (in `kernel`). Originally, the Open vSwitch flow cache was `exact match`. Upon entering the kernel, a packet’s headers would be looked up in a simple `hash` table using all the `packet fields` as a key. If `present`, the `cache` returned `instructions` for forwarding the packet. If `not present`, the packet would be sent to a `slow path` in `userspace` which executed the `full OpenFlow pipeline`, possibly consisting of several `packet classifications` (in the form of multiple `OpenFlow tables` traversed). The slow path would cache the decision in the kernel’s fast path for `subsequent packets` of the transport connection.

The current Open vSwitch architecture is depicted in Figure below. Just as before, packets entering the system are first checked against an `exact match cache`. Packets which miss the exact match cache are punted up to a `megaflow cache`. The `megaflow cache` provides the same `simplified semantics` as the exact match cache, with the additional ability to support `arbitrary bitwise` matching on any packet header field. Packets which `miss` the megaflow cache are sent up
once more to the OpenFlow pipeline provided by an `SDN controller`. The `OpenFlow pipeline` supports a wide range of `complicated operations` defined by the `OpenFlow standard` and `various OVS extensions`. These features, such as arbitrary packet metadata, multiple packet classification tables, and arbitrary recursion, are significantly `more complicated` and `computationally expensive` than the basic packet classification provided by the megaflow and exact
match caches.

>
>![]({{ site.img_url }}/2014-9-14/f1.JPG)
>

Given the packet forwarding performance difference between the megaflow cache and OpenFlow pipeline, population of the megaflow cache with `“good” entries` (that cover large traffic aggregates) can have a significant impact on the `overall performance` of the switch. Initially Open vSwitch implemented a native algorithm to populate the `megaflow cache`. Any bit `not matched` by the OpenFlow pipeline would be marked as `wildcarded` in the final megaflow cache entry.

This algorithm works `fine` if the slow path exclusively matches `low-entropy` packet fields, such as `L2/L3 addresses`. `Not wildcarding` these fields has a `low probability` of creating excessive `cache misses` (since a single IP address probably originates several transport connections). However, if the `slow path` matches `high entropy fields` like port numbers, the `megaflow cache` may `reduce` to an `exact match connection cache`.

*****

##Conclusion

In this paper, we considered the problem of `computing flow cache entries` for `a slow path operating` over `high entropy packet fields`, such as transport protocol port numbers. After ruling out the ideal, `proactive header space` based algorithms as too expensive, we developed `heuristic`, `reactive` algorithms that provide `near optimal` results with `limited overhead` in the slow path. The `decision tree algorithm` produces a flow cache with a small number of `unique flow masks`, despite being suboptimal compared to the `common match algorithm` in identifying the most general flow cache entries. However, in `Open vSwitch`, the decision tree algorithm is preferred because the implemented `tuple space search packet classification algorithm` is O(n) in the number of masks, and software can handle the resulting larger cache size. In a memoryconstrained environment, or with a limited number of rules on high entropy fields, the common match algorithm would be preferred. We have contributed our results to `mainstream Open vSwitch` and its `megaflow` implementation now supports flow caching of forwarding decisions including `L4 headers`, without requiring `every new` forwarded transport connection to be handled by the slow path.