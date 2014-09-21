---
layout: post
category : notes
tagline: ""
tags : [research]
---

{% include JB/setup %}

Pathlet Tracer: Enabling Layer 2 Pathlet Tracing through Context Encoding in Software-Defined Networking_HotSDN_2014

*****

##What's the question

In this paper, we introduce a `Layer 2 path tracing` utility named `PathletTracer`. PathletTracer offers an interface for users to specify multiple Layer 2 paths to inspect. Based on the Layer 2 paths of `interests`, PathletTracer then accounts paths with `identifiable IDs`, and installs a set of flow table entries into switches to `imprint path IDs` on the packets going through.

*****

##Why study

`Troubleshooting` Software-Defined Networks requires a `structured approach` to detect mistranslations between `high-level intent` (policy) and `low-level forwarding behavior`, and a flexible on-demand `packet tracing tool` is highly desirable on the `data plane`.

*****

##Innovativeness

PathletTracer re-uses some fields in packet headers such as the `ToS octet` for recording `path IDs`. To efficiently carry imprints using `limited bits`, PathletTracer uses an `encoding algorithm` motivated by the `calling context encoding scheme` in the `software engineering domain`. With `k bits` for encoding, PathletTracer is able to trace more than `2^k` paths simultaneously.

The contributions of the paper are as follows:

* We introduce pathlet tracing, a new and flexible mechanism for packet tracing on the SDN data plane.
* We describe a novel Layer 2 path encoding mechanism using unused octets in a packet header by adopting the
software engineering approach to encode program calling context. For example, as we will show in Section 4, PathletTracer requires only `2 bits` to trace the `7 paths` in the example network shown in Figure shown below.


*****

##Introduction

For correct troubleshooting of SDN operations, it is necessary to verify whether the `low-level actions` performed by `network devices` conform to the `highlevel policies` deployed by operators. In this paper we focus on `path tracing`, a specific operation for SDN troubleshooting that determines the `Layer 2 path` taken by a given packet.
<br>
Consider the network in Figure shown below, where host A can reach host B via multiple paths.

>
>![]({{ site.img_url }}/2014-9-15/f1.JPG)
>

Path tracing allows us to verify whether a packet from host A has taken the `desired route`, such as 1->2->3->5->6, to reach host B. Path tracing can help network operators to `improve network performance` (e.g., by `comparing various routing options` in `load balancing`) to `validate routing decisions` (e.g., by ensuring that a `routing algorithm performs correctly`), and to `allocate resource optimally` (e.g., by identifying `hot and cold spots` in networks).

We introduce PathletTracer, a Layer 2 path tracing utility that is both `correct` and `scalable`. PathletTracer offers an `interface` for users to specify multiple Layer 2 paths to trace, and get the information on which of the traced path a later packet has taken, or none of them. For correctness, PathletTracer records forwarding information in the `data plane` as a packet traverses a switch, rather than infer it from `switch configuration`. Specifically, PathletTracer associates each path to be traced with an `ID` and uses `unused bits` (e.g., `ToS octets`) in a packet’s header to carry the ID of the `path` the packet is traversing. `Specialized forwarding rules` installed by the `centralized controller` ensure that each switch in the network imprints path IDs to packets in transit.

Once a packet has arrived at the destination, PathletTracer `decodes` the ID to determine the path traversed. The scalability of PathletTracer depends on how, where, and when the centralized network controller enforces the recording of the path information by the switches.

The `scalability` of PathletTracer depends on `how`, `where`, and `when` the centralized network `controller` enforces the `recording` of the path information by the switches. We preserve scalability by making several design decisions about how we specify and encode paths.

**Pathlet tracing**. PathletTracer borrows the pathlet idea from Pathlet Routing. The underlying intuition is that the number of `path segments` (i.e., pathlets) is much less than the number of `source-destination combinations`, especially in a `virtualized environment` where we consider `VM-to-VM paths`. This helps `reducing the number of IDs` that we encode in the header of traced packets.

**Calling context encoding**. We use `precise calling context encoding` (PCCE) to ensure that we can encode each path within the `limited space` available in the `packet header`. PCCE is a software engineering technique to represent the `calling context`, a path of function calls from the `root` (e.g., main) function, of a program in concise values at runtime. PCCE analyzes the structure of a program and inserts arithmetic operation (e.g., addition) code to update concise runtime status such that it can be uniquely decoded relative to a calling context.

*****

##ARCHITECTURE

The main components of PathletTracer as shown below.

>
>![]({{ site.img_url }}/2014-9-15/f2.JPG)
>

* The user interface (UI) can take the following inputs: the interested paths to be `traced and encoded`, and the path ID from a received packet for `decoding`.

* The encoder generates the `codebook` and the corresponding set of `forwarding rules` for SDN switches based on the network topology and paths of interests to `stamp` the traversing packets with `compact IDs` for the paths.

* The decoder will use the codebook from the `encoder` to translate the path ID into a `hop-by-hop` path.

An overview of the PathletTracer workflow is shown below. Given a set of pathlets (valid Layer 2 path segments), PathletTracer compiles it with a `directed acyclic graph model` and applies a `path encoding algorithm`. This leads to a set of control messages to add `flow table entries` on switches for encoding packets on-the-fly. This process also generates a `codebook` to translate a packet ID to a Layer 2 path which the packet has taken.

>
>![]({{ site.img_url }}/2014-9-15/f3.JPG)
>

*****

##OFFLINE ENCODING

we describe how we encode network paths using concepts from the `calling context encoding`. First, the encoder builds a `forest of directed acyclic graphs` (DAGs) by composing the valid paths to be traced. On each DAG, the encoder creates a `virtual root node`, and adds a link from it to all the nodes with `0 indegree`. two DAGs by composing the seven paths in Figure shown above. G1 contains only the path 6 → 5 → 3 → 2 → 1 as it does not share any link with the other 6 paths; G2 contains the rest of the 6 paths as they share some links between each other.

>
>![]({{ site.img_url }}/2014-9-15/f4.JPG)
>

On each DAG, the encoder generates the IDs of the included paths. The algorithm starts by traversing the nodes in a topological order, and computes the path number (*PN*) for each edge. With respect to each node n, the *PN* value
for the first incoming edge is 0; for each of the remaining edges e : (p → n) the *PN* value is the sum of the possible paths from the root node to p. The ID of a path with respect to a node n is the sum of *PN*s of all edges
on the path from the virtual root node to n.

Algorithm 1 describes the steps. For example, G2 in Figure shown below has all 0 *PN* edges except e : (8 → 4) and e : (4 → 5). The ID of Path 8 → 4 → 5 → 6 is therefore (1 + 2 + 0) = 3 (11 in binary).

>
>![]({{ site.img_url }}/2014-9-15/f5.JPG)
>

After the encoding, the encoder produces a codebook for the traced paths. The codebook includes four fields: the `time period T` when the path IDs are valid; `path identifiers (ID)`, the `end switch on the path (Site)`, and the `encoded path (Path)`. Figure shown below shows an example of the code book for the 7 paths traced in Figure shown above.

>
>![]({{ site.img_url }}/2014-9-15/f6.JPG)
>

The `decoder` uses the `codebook` to serve path queries. When a user sends a `query` on a `path ID i` encoded in a packet received at `host x` at `time t`, the decoder first uses the network `topology information` to `resolve x` to the `switch (site) s` where x is attached. The decoder then looks up the codebook with `(t, i, s)` and returns the full path information matching the 3-tuple value.

*****

##ONLINE ENCODING

After computing the path IDs, the encoder generates `control messages` for all switches in the `DAGs` to enable `online path tracing`. These messages create rules in switches to imprint the `encoded IDs` to `packets in transit`. While the original `PCCE` work used arithmetic addition operations to record this information (e.g., ID = ID + 4), similar operations on packet header fields are not supported in existing OpenFlow table action sets; thus we need to make per-site additions in calling context encoding OpenFlow-compatible.

In general, there are three types of ingress ports on a switch in the DAGs for path tracing regarding how a switch is connected to its neighbors : (a) a port that `no traced path traverses`, (b) a port traversed by `traced paths sharing the same ID value`, and (c) a port traversed by `traced paths with different IDs`.

For example, switch i in Figure shown below has the link ending at its ingress port b in path X, and the link ending at port c in path Y and Z which have different encoding IDs. Therefore, the ingress port a of switch i is type (a), port b is type (b), and port c is type (c). For each type of port, PathletTracer adds a different set of flow table entries to enable switch i to imprint the `path ID` into the ToS bits of packets. Other fields can be in the matching rules if the traced paths are associated with specific flows.

>
>![]({{ site.img_url }}/2014-9-15/f7.JPG)
>

As shown below, the 4 flow table entries with ingress ports of `type (b)` for tracing 7 paths.

>
>![]({{ site.img_url }}/2014-9-15/f8.JPG)
>

As shown below, An example of `type (c)` ingress ports and the corresponding flow table entries.

>
>![]({{ site.img_url }}/2014-9-15/f9.JPG)
>


