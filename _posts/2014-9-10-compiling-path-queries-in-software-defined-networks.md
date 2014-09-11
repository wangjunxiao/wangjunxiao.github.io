---
layout: post
category : notes
tagline: ""
tags : [research]
---

{% include JB/setup %}

Compiling Path Queries in Software-Defined Networks_HotSDN_2014

*****

##What's the question

Monitoring the flow of traffic along network paths.

*****

##Why study

It's essential for SDN programming and troubleshooting. For example, traffic engineering requires measuring the ingress-egress traffic matrix; debugging a congested link requires determining the set of sources sending traffic through that link; and locating a faulty device might involve detecting how far along a path the traffic makes progress.

Tools such as NetFlow, sFlow, and SNMP are effective at monitoring flows, packets, or aggregate statistics at `individual links`. However, operators often need to reason about the flow of traffic along `paths` through the network.

*****

##Innovativeness

Past path-based monitoring systems operate by diverting packets to collectors that perform `“after-the-fact”` analysis, at the expense of large data-collection overhead.
<br> 
In this paper, we show how to do more efficient `“during-the-fact”` analysis.
<br>

* We introduce `a query language` that allows each SDN application to specify queries independently of the forwarding state or the queries of other applications.
* The queries use a `regular-expression` based path language that includes `SQL-like` “groupby” constructs for count aggregation.
* We track the packet trajectory directly on the data plane by converting the regular expressions into an automaton, and tagging the automaton state (i.e., the path prefix) in each packet as it progresses through the network.

In summary, our contributions are:
<br>

* `a query interface` that allows compositional specification of path queries, through a regular expression-based language with grouping constructs.
* `a runtime system` that directly observes packet paths on the data plane, and works with unmodified end hosts and OpenFlow 1.0 switches.
* preliminary results showing that our in-network “Neat Freak” strategy reduces `data-collection overhead` over typical “Hoarder” strategies.

*****

##Introduction

In the context of SDN, NetSight(NSDI_2014) is an example towards the `Hoarder`(record as much information as possible, so users can specify a wide range of more refined queries after the fact) end of the spectrum: It creates a `postcard` for each packet at each hop in its journey. These postcards, which contain the `packet header`, the `matching flow entry`, and the `switch` and `output port`, are sent to servers that store them for later analyses, such as backtrace construction.

In contrast, in this paper, we illustrate how to develop a `“tunable” Neat Freak`(specify queries ahead of time, allowing them to collect and store much less data, but support a narrower range of pre-determined queries) for path queries in SDNs.

Our run-time system implements these queries by generating `OpenFlow rules` that analyze packets as they flow through the network’s `data plane`, to avoid directing every packet (or postcard) to `collectors` for analysis.

To achieve this, we record packets’ past trajectories onto bits on the packets themselves (i.e., tags).

******

##PATH-QUERY LANGUAGE

As shown below，the basic syntax of path expressions.

>
>![]({{ site.img_url }}/2014-9-10/syn.JPG)
>

*****

***Basic Atoms***. The simplest path query is one consisting of just `one point` in the network. Moreover, every atom contains a `predicate`, which helps identify the specific network location of interest and the set of packets flowing through that location.

For example, the atom

>in_atom(match(srcip=H1) & match(switch=1))

identifies the set of packets with source IP H1 that flow `in` to switch 1. In contrast,

>out_atom(match(srcip=H1) & match(switch=1))

identifies the set of packets with source IP H1 that flow `out` of switch 1.
<br>
Two other useful atoms are `in_atom(ingress())`, which matches all packets `entering` the network, and `out_atom(egress())`, which matches all packets `leaving` the network.

*****

***Grouping Atoms***. A basic atom identifies just `one set` of packets. A `grouping atom`, on the other hand, identifies `many closely-related sets` of packets. For example, the following atom matches all packets entering the network (via the ingress predicate) and then divides that set into subsets based on the `switch` at which they arrive.

>in_group(ingress(), [’switch’])

More generally, `in_group(pred, [h1, h2, ..., hn])` selects packets coming into a switch that satisfy the predicate pred and then divides that set into subsets where each packet in the subset shares `common values` for the headers h1, h2, ..., hn. Hence the grouping atom

>in_group(ingress(), [’srcip’,’dstip’])

would divide the incoming packets into groups based on source IP-destination IP pairs. The `out_group atom` is similar to the `in_group atom`, except it identifies packets on their way out of a switch (like `out_atom`).

*****

***Regular Path Combinators***. Given path queries p1, p2, we can build more complex paths using the usual `concatenation (^)`, `alternation/choice (|)` and `Kleene star/iteration (*)`.

For instance, the path `p1 ^ p2` requires that a packet first traverse the path `p1` and then traverse the path `p2`. If p1 and p2 are `simple atoms`, this means that the packet must satisfy the `predicate` designated by `p1`, and then satisfy the predicate designated by `p2`. As shown below, several additional example queries.

>
>![]({{ site.img_url }}/2014-9-10/t1.JPG)
>

*****

***Using Path Queries to Inspect or Count Packets***. A programmer uses a path query by directing the packets matching the query to a `bucket`. Intuitively, a bucket is an abstract “location” in the network that collects packets. There are two kinds of buckets: `packet buckets` and `counting buckets`. 

* `Packet buckets` are currently implemented as `queues` on the controller that store a set of packets.
* Collecting aggregate traffic statistics requires a different sort of bucket—`the counting bucket`. Rather than directing individual packets to the controller, a counting bucket simply returns the `total byte` and `packet counts` across all packets matching the query.

*****

##PATH QUERY COMPILATION

we achieve this goal by compiling the `queries` into `OpenFlow rules` that preserve the `underlying forwarding policy`, but add a few bits of `state` (i.e., a tag) to packets as they traverse the network.

*****

we first convert our `path queries` into `standard regular expressions` over `strings`. We use off-the-shelf libraries to convert `regular expressions` into `DFAs`.

***Converting path queries to regular expressions***. To compile them, we convert them into regular expressions over `strings of characters`.The key step here is assigning specific `characters` to `predicates`.

Here, consider a packet with srcip=H1 and dst=10.0.0.3 entering switch 1. This packet satisfies `two predicates`: match(switch=1, srcip=’H1’) from p1 and match(switch=1) from p2. Consequently, we partition the complex predicates we find in path queries into `non-overlapping subparts` (i.e., an orthogonal basis for the complete set of predicates) and assign one character to each such non-overlapping subpart. The representations for the predicates and the paths p1 and p2 is shown below.

>
>![]({{ site.img_url }}/2014-9-10/pre.JPG)
>

The DFA for p1 and p2 together is shown below. (a) DFA only for path p1. (b) DFA for both p1 and p2.

>
>![]({{ site.img_url }}/2014-9-10/dfa.JPG)
>

*****

The next step is to translate the `DFA` into `primitive state transitioning` and `accepting policies` in `Pyretic`. These policies will read and write the relevant `DFA state` into each `packet` as it moves from switch to switch. 

***State Transition Policies***. We implement these states using Pyretic’s `virtual header field` mechanism. Packets just entering the network are stamped with the `“starting” state` of the DFA, and transitions thereafter are implemented on each switch as policies that check the current state, and write in a new state according to the transition rules.

For example, consider a transition from `DFA state s` to `DFA state t` on seeing `character c`. Assume also that a function `pred_of` converts `c` to the associated `predicate p`. In such a case we generate the following clause.

>match(state=s & pred_of(c)) >> modify(state=t)

The tagging policy for an automaton is the parallel composition of all such transition policies:

>tagging = clause_1 + ... + clause_n

For example, the tagging policy for the DFA in Fig. (a) is implemented as follows:

>
>![]({{ site.img_url }}/2014-9-10/p1.JPG)
>

***Accepting policies***. In addition to encoding the DFA transitions, we must encode when a packet is `accepted` by the automaton. The `accepting actions` are encoded as `Pyretic policies` that recognize the `transition` into an `accepting DFA state`, and direct those packets to the `bucket` associated with the accepting query.

For example, we generate the following counting policy for the accepting state in Fig. (a)

>match(state=s) & pred_of(c) >> p.bucket()

*****

The final step is merging the policies generated from the DFA with the global packet forwarding policy generated by `other SDN applications`. Our compiler assembles the following components:

* (1) The tagging policy, which implements DFA transitions on packets that undergo a state change
* (2) An unaffected policy, which implements the identity function on those packets that do not undergo a state change
* (3) The fwding policy, which implements the forwarding policies of other applications
* (4) The counting policy, which identifies the accepted packets. We put them together as follows: 

> ((tagging + unaffected) >> fwding) + counting

We show a sample of the final OpenFlow 1.0 rules generated as a result of compiling the path queries for switch
1 in Fig. (b), as table shown below.

>
>![]({{ site.img_url }}/2014-9-10/of.JPG)
>

However, the compilation has carefully teased apart `overlapping predicates` and `actions` among the `queries` and the `forwarding rules`, to generate data plane rules that work for both forwarding as well as queries.









