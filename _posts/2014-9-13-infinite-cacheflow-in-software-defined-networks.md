---
layout: post
category : notes
tagline: ""
tags : [research]
---

{% include JB/setup %}

Infinite CacheFlow in Software-Defined Networks_HotSDN_2014

*****

##What's the question

we define a `hardware-software hybrid` switch design that relies on `rule caching` to provide `large rule tables` at low cost. Unlike traditional caching solutions. we neither cache `individual rules` (to respect rule dependencies) nor `compress rules` (to preserve the per-rule traffic counts). Instead we `“splice”` long dependency chains to cache smaller groups of rules while preserving the semantics of the `network policy`.

*****

##Why study

Ternary Content Addressable Memory (TCAM) enables OpenFlow switches to process packets at high speed based on multiple header fields, today’s commodity switches support just thousands to tens of thousands of rules. To realize the potential of SDN on this hardware, we need efficient ways to support the abstraction of a switch with arbitrarily large rule tables.

*****

##Innovativeness

Our design satisfies four core criteria: (1) `elasticity` (combining the best of hardware and software switches), (2) `transparency`(faithfully supporting native OpenFlow semantics, including traffic counters), (3) `fine-grained rule caching` (placing popular rules in the TCAM, despite dependencies on less-popular rules), and (4) `adaptability` (to enable incremental changes to the rule caching as the policy changes).

*****

##Computing Rule Dependencies

As shown below, an example with six rules that match a ternary format. If the TCAM can store four rules (k=4), we cannot select the four rules with highest weight (ie., R3, R4, R5, and R6), because packets that should match R1 (with pattern 0000) would match R3 (with pattern 000\*); similarly, some packets (with pattern 11\*\*) that should match R2 would match R4 (with pattern 1\*1\*). That is, rules R3 and R4 `depend` on rules R1 and R2, respectively.

>
>![]({{ site.img_url }}/2014-9-13/f1.JPG)
>

When a rule R is cached in the TCAM, the corresponding dependent rules should also move to the TCAM. However, checking for intersecting patterns does not capture all of the rule dependcies. R6 also depends on R2 (even though the matches of R2 and R6 do not intersect), because the match 11\*\* overlaps with that of R4 (1\*1\*). If the TCAM stored only R4 and R6, packets that should match R2 would inadvertently match R4. 

The algorithm shown below captures all such dependencies.

>
>![]({{ site.img_url }}/2014-9-13/f2.JPG)
>

Our algorithm constructs a single dependency graph, as shown in Figure (a).

>
>![]({{ site.img_url }}/2014-9-13/f3.JPG)
>

*****

##Caching Groups of Dependent Rules

We present a new algorithm that avoids catching low-weight rules. Each rule is assigned a `"cost"` corresponding to the number of rules that must be installed together and a `"weight"` corresponding to the number of packets expected to hit that rule. For example, R5 depends on R1 and R3, leading to a cost of 3, as shown in Figure (a). In this situation, R5 and R6 hold the majority of the weight, but cannot be simultaneously on the switch, as installing either has a cost of 3 and together they do not fit. The best we can do is to install rules R1, R2, R4 and R6. This maximizes total weight, subject to respecting all dependencies. 

The current problem of maximizing the total weight can be formulated as a linear integer programming problem, where each rule has a variable indicating whether the rule is installed in the cache. The objective is to maximize the sum of the weights of the installed rules, while installing at most k rules; if rule Rj depends on rule Ri, rule Rj cannot be installed unless Ri is also installed. The problem can be solved with an O(n^k) brute-force algorithm that is expensive for large k.

We use a heuristic that is modeled on a greedy [PTAS(polynomial time approximation scheme)][1] for the [Budgeted Maximum Coverage][2] problem, which is a relaxed version of the `all-neighbors knapsack` problem. In our greedy heuristic, at each stage, the algorithm chooses a set of rules that maximizes the ratio of combined rule weight to combined rule cost (△W/△C), until the total cost reaches k. This algorithm runs in O(nk) time.

On the example, this greedy algorithm selects R6 first (and its dependent-set {R2, R4}), and then R1 which brings the total cost to 4. Thus the set of rules in the TCAM are R1, R2, R4, and R6. We refer to this algorithm as the `dependent-set algorithm`. 

*****

##Splicing Long Chains of Dependent Rules

For Example, consider a firewall that has a single low-priority "accept" rule that depends on many high-priority "deny" rules that match relatively little traffic. Catching the one "accept" rule would require catching many "deny" rules. In particular, we "splice" the dependency chain by creating a small number of new rules that cover many low-weight rules and send the affected packets to the `software switch`. as shown in Figure (b).

>
>![]({{ site.img_url }}/2014-9-13/f4.JPG)
>

To take a more general case, the old cost for the red rule in Figure (c) was the entire set of ancestors (in light red), but the new cost (in Figure (d)) is  defined just by the immediate ancestors (in light red).

>
>![]({{ site.img_url }}/2014-9-13/f5.JPG) ![]({{ site.img_url }}/2014-9-13/f6.JPG) 
>

**Combining the best of the two techniques:** We consider a metric that chooses the `best` of the two sets i.e., max(△**W**dep/△**C**dep, △**W**cover/△**C**cover). We refer to this version as the `mixed-set algorithm`.

*****

##Updating the rules incrementally

When a new rule is inserted or an old rule is removed from the dependency graph, the graph is still maintained without incurring the complexity of the static algorithm shown above.

The key intuition is to store `edge-related metadata` so that we can incrementally maintain the graph while manipulating only the relevant edges.

*****

##CACHEFLOW SYSTEM DESIGN

CacheFlow consists of a `CacheMaster` module that receives `OpenFlow commands` from the `controller`, and uses the OpenFlow protocol to distribute `rules` to the `underlying switches`, as shown below.

>
>![]({{ site.img_url }}/2014-9-13/f7.JPG)
>

Having multiple software switches allows CacheFlow to "shard" the `cache-miss` traffic over multiple software switches (by assigning different `"cover"` rules to different output ports), for higher throughput and rule capacity. Packets stay in the `"fast path"` in the data plane, reducing the `performance penalty` for a cache miss.

**Preserving inports/outports:** CacheFlow installs three kinds of rules in the hardware switch: (i) fine-grained rules that apply part of the policy (a "hit"), (ii) coarse-grained rules that forward packets to a software switch (a "miss"), and (iii) coarse-grained rules that direct return traffic from the software switches to the right output port(s), In addition to matching on packet-header fields, an OpenFlow policy may match on the inport where the packet arrives. Therefore, the hardware switch `tags` cache-miss packets with the input port (e.g., using a VLAN tag) so the software switches can apply rules that depend on the inport.

The rules in the software switches apply any "drop" or "modify" actions, tag the packets for proper forwarding at the `hardware switch`, and direct the packet `back to` the hardware switch. Upon receiving the return packet, the hardware switch simply matches on the `tag`, `pops` the tag, and forwards to the designated output port(s). If a cache-miss rule has an action that sends the packet to the `controller`, the `CacheMaster` transforms the `packet_in` message from the `software switch` by (i) copying the `inport` from the tag into the inport of the packet_in message and (ii) stripping the tag from the packet before sending the message to the controller.

**Traffic counts, barrier messages, and rule timeouts:** CacheFlow preserves the semantics of `OpenFlow 1.0` constructs like `queries on traffic statistics`, `barrier messages`, and `rule timeouts` by emulating all of these features in the `CacheMaster`. i.e, the behavior of the switches maintained by CacheFlow is no different from that of a single OpenFlow switch with infinite rule space.

CacheMaster maintains packet and byte counts for each rule installed by the controller, updating its `local information` each time a rule moves to a `different part` of the `cache hierarchy`.



[1]: http://en.wikipedia.org/wiki/Polynomial-time_approximation_scheme    "polynomial time approximation scheme"
[2]: http://en.wikipedia.org/wiki/Maximum_coverage_problem#Budgeted_maximum_coverage    "Budgeted Maximum Coverage"