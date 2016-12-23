---
layout: post
category : notes
tagline: ""
tags : [research]
---

{% include JB/setup %}

VL2: 可扩展的灵活的数据中心网络架构_SIGCOMM_2009

*****

`Introduction`

* VL2使用了(1)**flat addressing**，允许service搭建在网络中任何位置; (2)**Valiant Load Balancing**，traffic均匀分布在网络中; (3)**基于end-system的address resolution**，构建更大规模的server pool并且不为network control plane引入complexity

* 现有的网络架构存在三方面的问题(1)existing architectures do not provide enough **capacity** between the servers they interconnect; (2)the network does little to prevent a traffic flood in one service from affecting the **other services** around it: (3)the routing design in conventional networks achieves scale by assigning servers **topologically significant IP addresses** and dividing servers among VLANs

* 为了解决上述问题，VL2 give each service the illusion that all the servers assigned to it, and only those servers, are connected by a single non-interfering Ethernet switch—*a Virtual Layer 2*— and maintain this illusion even as the size of each service varies from 1 server to 100,000

* 实现上述解决方案(vision)需要构建满足下列三个目标的网络:

>
  ***Uniform high capacity***: The maximum rate of a server-to-server traffic flow should be limited only by the available capacity on the network-interface cards of the sending and receiving servers, and assigning servers to a service should be independent of network topology.
>
  ***Performance isolation***: Traffic of one service should not be affected by the traffic of any other service, just as if each service was connected by a separate physical switch.
>
  ***Layer-2 semantics***: Just as if the servers were on a LAN—where any IP address can be connected to any port of an Ethernet switch due to flat addressing—data-center management software should be able to easily assign any server to any service and configure that server with whatever IP address the service expects. Virtual machines should be able to migrate to any server while keeping the same IP address, and the network configuration of each server should be identical to what it would be if connected via a LAN. Finally, features like link-local broadcast, on which many legacy applications depend, should work.
>

* The switches that make up the network operate as layer-3 routers with routing tables calculated by **OSPF**, thereby enabling the use of **multiple paths** (unlike Spanning Tree Protocol) while using a well-trusted protocol

* The IP addresses used by services running in the data center cannot be tied to **particular switches** in the network, or the **agility** to reassign servers between services would be lost

* VL2 assigns servers IP addresses that act as **names** alone, with **no topological significance**. When a server sends a packet, the **shim-layer** on the server invokes a directory system to learn the **actual location of the destination** and then tunnels the original packet there. Binding between **addresses and location**

* VL2使用了**[Clos topology][1]** that provides extensive path diversity between servers, and adopt **Valiant Load
Balancing (VLB)** to spread traffic across all available paths without any centralized coordination or traffic engineering

* VLB by causing the traffic between any pair of servers to bounce off a **randomly chosen switch** in the **top level** of the Clos topology and leverage the features of layer-3 routers, such as **EqualCost MultiPath (ECMP)**, to spread the traffic along multiple subpaths for these two path segments

* 主要工作总结为以下四个方面: 

>
  Make a first of its kind study of the ***traffic patterns*** in a production data center, and find that there is tremendous volatility in the traffic, cycling among 50-60 different patterns during a day and spending less than 100s in each pattern at the 60th percentile.
>
  Design, build, and deploy every component of VL2 in an 80-server cluster. Using the cluster, we experimentally validate that VL2 has the properties set out as objectives, such as uniform capacity and performance isolation. Also demonstrate the speed of the network, such as its ability to shuffle 2.7TB of data among 75 servers in 395s.
>
  Apply ***Valiant Load Balancing*** in a new context, the interswitch fabric of a data center, and show that flow-level traffic splitting achieves almost identical split ratios (within 1% of optimal fairness index) on realistic data center traffic, and it smoothes utilization while eliminating persistent congestion.
>
  Justify the design trade-offs made in VL2 by comparing the ***cost*** of a VL2 network with that of an equivalent network based on existing designs
>

*****

`Inspiration`

>
>![]({{ site.img_url }}/2014-8-24/dominant.JPG)
>

* 如上图所示的Cisco传统数据中心网络架构，在今天仍然占据主导地位。To **limit overheads** (e.g., packet flooding and ARP broadcasts) and to **isolate different services or logical server groups** (e.g., email, search, web front ends, web back ends), servers are partitioned into **virtual LANs (VLANs)**.

* 传统网络架构设计受以下三个方面的限制: 

>
 ***Limited server-to-server capacity***: As we go up the hierarchy, we are confronted with steep technical and financial barriers in sustaining high bandwidth. Thus, as traffic moves up through the layers of switches and routers, the over-subscription ratio increases rapidly. For example, servers typically have 1:1 over-subscription to other servers in the same rack — that is, they can communicate at the full rate of their interfaces (e.g., 1 Gbps). We found that up-links from ToRs are typically 1:5 to 1:20 oversubscribed (i.e., 1 to 4 Gbps of up-link for 20 servers), and paths through the highest layer of the tree can be 1:240 oversubscribed. This large over-subscription factor fragments the server pool by preventing idle servers from being assigned to overloaded services, and it severely limits the entire data-center’s performance.
>
 ***Fragmentation of resources***: As the cost and performance of communication depends on distance in the hierarchy, the conventional design encourages service planners to cluster servers nearby in the hierarchy. Moreover, spreading a service outside a single layer-2 domain frequently requires reconfiguring IP addresses and VLAN trunks, since the IP addresses used by servers are topologically determined by the access routers above them. The result is a high turnaround time for such reconfiguration. Today’s designs avoid this reconfiguration lag by wasting resources; the plentiful spare capacity throughout the data center is ofen effectively reserved by individual services (and not shared), so that each service can scale out to nearby servers to respond rapidly to demand spikes or to failures. Despite this, we have observed instances when the growing resource needs of one service have forced data center operations to evict other services from nearby servers, incurring significant cost and disruption.
>
 ***Poor reliability and utilization***: Above the ToR, the basic resilience model is 1:1, i.e., the network is provisioned such that if an aggregation switch or access router fails, there must be sufficient remaining idle capacity on a counterpart device to carry the load. This forces each device and link to be run up to at most 50% of its maximum utilization. Further, multiple paths either do not exist or aren’t effectively utilized. Within a layer-2 domain, the Spanning Tree Protocol causes only a single path to be used even when multiple paths between switches exist. In the layer-3 portion, Equal Cost Multipath(ECMP) when turned on, can use multiple paths to a destination if paths of the same cost are available. However, the conventional topology offers at most two paths.
>

*****

`Analysis`

* Data-Center Traffic Analysis

* Flow Distribution Analysis

* Traffic Matrix Analysis

* Failure Characteristics
 
  With no obvious way to eliminate all failures from the top of the hierarchy, VL2’s approach is to broaden the topmost levels of the network so that the impact of failures is muted and performance degrades gracefully, moving from 1:1 redundancy to n:m redundancy.

*****

`Separating names from locators`

* The data center network support for hosting any service on any server, for rapid growing and shrinking of server pools, and for rapid virtual machine migration. In turn, this calls for separating names from locations

* VL2’s addressing scheme separates server names, termed **application-specific addresses (AAs)**, from their locations, termed **location-specific addresses (LAs)**

* VL2 uses a scalable, reliable **directory system** to maintain the mappings between names and locators. A shim layer running in the network stack on every server, called the **VL2 agent**, invokes the directory system’s resolution service.

* The VL2 agent enables fine-grained path control by adjusting the randomization used in **VLB**. The agent also replaces **Ethernet’s ARP functionality** with queries to the VL2 **directory system**

*****

`Scale-out Topologies`

* Conventional hierarchical data-center topologies have poor bisection bandwidth and are also susceptible to major disruptions due to device failures at the highest levels

>
>![]({{ site.img_url }}/2014-8-24/scaleout.JPG)
>

* 如上图所示，An example folded Clos network between Aggregation and Intermediate switches provides a richly-connected backbone wellsuited for VLB. e network is built with two separate address families — topologically significant Locator Addresses (LAs) and flat Application Addresses (AAs)

* Use *DA*-port Aggregation and *DI*-port Intermediate switches, and connect these switches such that the capacity between each layer is *DI***DA*/2 times the link capacity

* Switch-to-switch links are typically faster than server-to-switch links. By leveraging this gap, reduce the number of cables required to implement the Clos (as compared with a fat-tree), and simplify the task of spreading load over the links

*****

`Address resolution and packet forwarding`

>
>![]({{ site.img_url }}/2014-8-24/address.JPG)
>

* 如上图所示，VL2 uses two different IP-address families. All switches and interfaces are assigned LAs, and
switches run an IP-based (layer-3) link-state routing protocol that disseminates only these LAs. This allows switches to obtain the complete switch-level topology, as well as forward packets encapsulated with LAs along shortest paths

* On the other hand, applications use application-specific IP addresses (AAs), which remain unaltered no matter how servers’ locations change due to virtual-machine migration or re-provisioning. Each AA (server) is associated with an LA, the identifier of the ToR switch to which the server is connected

* The VL2 directory system stores the mapping of AAs to LAs, and this mapping is created when application servers are provisioned to a service and assigned AA addresses

* The crux of offering layer-2 semantics is having servers believe they share a single large IP subnet (i.e., the entire AA space) with other servers in the same service, while eliminating the ARP and DHCP scaling bottlenecks that plague large Ethernets

* **Packet forwarding:**  To route traffic between servers, which use AA addresses, on an underlying network that knows routes for LA addresses, 如上图所示 the VL2 agent at each server traps packets from the host and **encapsulates** the packet with the LA address of the ToR of the destination. Once the packet arrives at the LA (the destination ToR), the switch **decapsulates** the packet and delivers it to the destination AA carried in the inner header.

* **Address resolution:**  Servers in each service are configured to believe that they all belong to the same IP subnet. Hence, when an application sends a packet to an AA for the first time,the networking stack on the host generates a broadcast ARP request for the destination AA. e VL2 agent running on the host intercepts this ARP request and converts it to a unicast query to the VL2 directory system. The directory system answers the query with the LA of the ToR to
which packets should be tunneled. The VL2 agent caches this mapping from AA to LA addresses, similar to a host’s ARP cache, such that subsequent communication need not entail a directory lookup.

* **Access control via the directory service:**  A server cannot send packets to an AA if the directory service refuses to provide it with an LA through which it can route its packets. is means that the directory service can enforce access-control policies. An advantage of VL2 is that, when inter-service communication is allowed, packets
flow directly from a source to a destination, without being detoured to an **IP gateway** as is required to connect two VLANs in the conventional architecture.

* These addressing and forwarding mechanisms were chosen for two reasons. **First**, they make it possible to use low-cost switches, which ofen have small routing tables (typically just 16K entries) that can hold only LA routes, without concern for the huge number of AAs. **Second**, they reduce overhead in the network control plane by preventing it from seeing the churn in host state, tasking it to the more scalable directory system instead.

*****

`Random traffic spreading over multiple paths`

* To offer hot-spot-free performance for arbitrary traffic matrices, VL2 uses two related mechanisms: **VLB** and **ECMP**. The goals of both are similar — VLB distributes traffic across a set of intermediate nodes and ECMP distributes across equal-cost paths — but each is needed to overcome limitations in the other. VL2 uses flows,
rather than packets, as the basic unit of traffic spreading and thus avoids out-of-order delivery.

* 如上图所示，VL2 agent uses encapsulation to implement VLB by sending traffic through a randomly-chosen Intermediate switch. The packet is first delivered to one of the Intermediate switches, decapsulated by the switch, delivered to the ToR’s LA, decapsulated again, and finally sent to the destination.

* While encapsulating packets to a specific, but randomly chosen, Intermediate switch correctly realizes VLB, it would require updating a potentially huge number of VL2 agents whenever an Intermediate switch’s availability changes due to switch/link failures. Instead, we assign the **same LA address** to all Intermediate switches, and the directory system returns this anycast address to agents upon lookup. Since all Intermediate switches are exactly three hops away
from a source host, ECMP takes care of delivering packets encapsulated with the anycast address to any one of the active Intermediate switches. Upon switch or link failures, ECMP will react, eliminating the need to notify agents and ensuring scalability.

* In practice, however, the use of ECMP leads to two problems. **First**, VL2 defines **several anycast addresses**, each associated with only as many Intermediate switches as ECMP can accommodate. When an Intermediate switch fails, VL2 reassigns the anycast addresses from that switch to other Intermediate switches so that all anycast addresses remain live, and servers can remain unaware of the network churn. **Second**, some inexpensive switches cannot correctly retrieve the five-tuple values (e.g., the TCP ports) when a packet is encapsulated with multiple IP headers. us, the agent at the source **computes a hash of the five-tuple values** and **writes that value into the source IP address field**, which all switches do use in making ECMP forwarding decisions.

* The greatest concern with both ECMP and VLB is that if “**elephant flows**” are present, then the random placement of flows could lead to **persistent congestion** on some links while others are **underutilized**. Should it occur, initial results show the VL2 agent can detect and deal with such situations with simple mechanisms, such as **re-hashing** to change the path of large flows when TCP detects a severe congestion event (e.g., a full window loss).

* VL2 completely eliminates the most common sources of broadcast: ARP and DHCP. **ARP** is replaced by the **directory system**, and **DHCP messages** are intercepted at the ToR using **conventional DHCP relay agents** and **unicast forwarded to DHCP servers**. To handle other general layer-2 broadcast traffic, **every service** is assigned an IP **multicast address**, and all broadcast traffic in that service is handled via IP multicast using the **service-specific multicast address**. The VL2 agent rate-limits broadcast traffic to prevent storms.

*****

`The VL2 Directory System`

* The VL2 directory provides three key functions: (1) lookups and (2) updates for AA-to-LA mappings; and (3) a reactive cache update mechanism so that latency-sensitive updates (e.g., updating the AA to LA mapping for a virtual machine undergoing live migration) happen quickly.

>
>![]({{ site.img_url }}/2014-8-24/dsa.JPG)
>

* 如上图所示，the design of a two-tiered directory system consists of (1) a modest number (50-100 servers for 100K servers) of **read-optimized, replicated directory servers** that cache AA-to-LA mappings and handle queries from VL2 agents, and (2) a small number (5-10 servers) of **write-optimized, asynchronous replicated state machine (RSM) servers** that offer a strongly consistent, reliable store of AA-to-LA mappings. 

* Updating caches reactively: when a stale mapping is used, some packets arrive at a stale LA—a ToR which **does not host the destination server anymore**. The ToR may forward a sample of such **non-deliverable packets** to a directory server, triggering the directory server to gratuitously correct the stale mapping in the source’s cache via **unicast**.


[1]: http://en.wikipedia.org/wiki/Clos_network   "clos network"
