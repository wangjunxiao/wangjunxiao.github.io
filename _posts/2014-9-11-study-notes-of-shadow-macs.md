---
layout: post
category : notes
tagline: ""
tags : [research]
---

{% include JB/setup %}

Shadow MACs: Scalable Label-switching for Commodity Ethernet_HotSDN_2014

*****

#What's the question

we demonstrate that using `destination MAC addresses` as opaque `forwarding labels` allows an SDN controller to leverage large MAC (L2) forwarding tables to manage a plethora of `fine-grained` paths.

*****

##Why study

While SDN promises `fine-grained`, dynamic control of the network, in practice limited switch TCAM rule space restricts most forwarding to be `coarse-grained`. Current switches have only a few thousand flexible rules, making it hard to provide even `host-pair granularity` for forwarding, so forwarding is typically based on `aggregates` 

*****

##Innovativeness

In this shadow MAC model, the SDN controller can install `MAC rewrite rules` at the network edge to guide traffic on to intelligently selected paths to balance traffic, avoid failed links, or route flows through middleboxes. 
<br>
Further, by decoupling the network `edge` from the `core`, we address many other problems with SDN, including consistent network updates, fast rerouting, and multipathing with end-to-end control.
<br>
In this paper, we make the following contributions:

* We demonstrate how to implement label switching with `shadow MAC addresses` as forwarding labels, which exploits large L2 forwarding tables available in commodity Ethernet hardware.
* We demonstrate that label switching solves many current `SDN problems` by enabling consistent network updates, scalable fine-grained forwarding, fast rerouting, and multipathing with end-to-end control.

*****

##Introduction

Shadow MACs, as forwarding labels. Unlike `traditional (physical) MAC addresses`, shadow MACs are not associated with a particular `physical endpoint` in the network. Rather, they are `opaque values` akin to `MPLS labels`. `Shadow MAC forwarding rules` can be installed in the `L2 forwarding tables`, which are typically the largest tables available with 100,000+ entries.

*****

##Design

***1. Control Plane***

The control plane of our label-based forwarding mechanism is implemented via `extensions` to a `centralized SDN controller`. We modify the controller to export an `install route API` to install `a shadow-MAC-based label-routed path` to a destination. The controller then programs `core` and `egress` switches with the appropriate `forwarding rules`. We can activate the installed route immediately by configuring the `ingress` switches. This is achieved by assigning a `unique identifier` to each of the `installed routes`.  SDN applications can activate one of the pre-installed routes for a flow by making an `API call` to the `select route interface` and specifying the`source and flow identifier` along with the `route identifier` for ingress switch match.

***2. Core Forwarding***

The SDN controller allocates a unique shadow MAC address for each path in the network. It then installs rules that match on the shadow MAC address in the `L2 forwarding table` of each switch along the path. Switches in the network core are unaware that `shadow MAC addresses` do not correspond to `physical endpoint MAC addresses`, and simply forward packets based the `installed MAC forwarding rules`. 

***3. Edge Forwarding***

The source needs to select the appropriate `shadow MAC` based on `flow information` and (re)write it in each packet’s `destination MAC field`. The destination needs to rewrite the `destination MAC address` from the `shadow MAC` to the destination’s `real MAC address`. We have implemented two schemes to accomplish these goals, `MAC address rewriting` and `ARP spoofing`.

**MAC Address Rewriting:** The MAC address rewriting scheme leverages the fact that OpenFlow-compatible switches can `rewrite addresses` in the data plane at line rate. To steer a packet to a particular path, we install a rule in the `ingress` switch that matches `flow-specific fields` and rewrites the `destination MAC address` to the `shadow MAC address` for the `desired path`. At the `egress` switch, we install a rule that rewrites the `destination MAC` to the destination host’s `real MAC address`. If end hosts are `virtualized`, the rewrite rules are installed in the corresponding `hypervisor vSwitches`. Otherwise they are installed in the `physical ingress and egress switches` using one `TCAM rule` per `active shadow MAC`.

Of note, this scheme allows `distinct shadow MACs`, and thus `distinct paths` through the network, to be assigned to `different flows` between the `same source-destination host pair`. As shown below.

>
>![]({{ site.img_url }}/2014-9-11/f1.JPG)
>

we use shadow MACs to configure two different routes between a source (A) and destination (B). The first route uses shadow MAC B1 to carry traffic destined to TCP port 80; the second route uses shadow MAC B2 to carry the remainder of traffic between A and B.

**ARP Spoofing:** In this scheme, the `SDN controller` acts as an `ARP proxy` and handles all `ARP requests` from hosts When a path is activated between source and destination, the SDN controller sends a `gratuitous ARP response` to the `source` identifying the `shadow MAC` as the `MAC address` corresponding to the `destination`. The source host will insert the `shadow MAC address` in all packets intended for the `destination`. We then configure the destination host to accept packets addressed to the corresponding shadow MAC address either by doing `rewrites` or by putting the `destination NIC` in `promiscuous mode` and modifying the `Linux kernel` to remove `MAC filtering`. As shown below.

>
>![]({{ site.img_url }}/2014-9-11/f2.JPG)
>

*****

##KEY BENEFITS

**Minimal TCAM Usage:** we use `no` TCAM rules in the network `core` for forwarding and `at most one` TCAM entry per `edge` switch per active shadow MAC sourced or drained by a directlyconnected host. In virtualized environments, MAC rewriting can be performed in the hypervisor vSwitches, reducing the physical switch TCAM requirement to `zero`. With ARP spoofing, `no` TCAM entries are used at the edge, except possibly at the egress if host stacks are `unmodified`.

**Consistent Updates:** When the route for an ongoing flow needs to be `updated`, the SDN controller assigns the flow a `fresh shadow MAC` and installs `new rules` in the core and egress switches along the `new path`, all while ongoing traffic continues to be forwarded along the `original path`. When the new path is `fully installed`, the SDN controller updates the route `atomically` either by installing a `new rewrite rule` in the ingress switch or by sending a `new gratuitous ARP`. Once the SDN controller is confident that all packets using the old path have `drained` from the network, it can `tear down` the old path.

**End-to-End Multipathing:** Many data-center topologies, e.g., `fat-trees` and `Jellyfish`, provide `multiple distinct paths` between host pairs. On such topologies, it is straightforward to implement flow-based multipathing using shadow MACs. First, the SDN controller allocates `multiple distinct shadow MACs` and associated `paths` through the network for `each host pair`. Then, each ingress switch assigns each flow to one of the shadow MACs associated with the flow’s destination.

**Fast Switch-over:** Fast switch-over to alternate routes is important for `reacting to failures` or `rerouting flows around congestion`. Our API allows SDN applications to `pre-install multiple paths` for a given flow, each of which uses a `distinct shadow MAC address`. After installation, only one is `activated` (by installing the appropriate ingress rewrite rule or performing the gratuitous ARP), while the remainder `lie dormant until needed`.
