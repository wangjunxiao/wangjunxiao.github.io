---
layout: post
category : notes
tagline: ""
tags : [research]
---

{% include JB/setup %}

ECMP: Analysis of an Equal-Cost Multi-Path Algorithm_RFC2992_2000

*****

####`Introduction`

* Equal-cost multi-path (ECMP) is a routing technique for routing packets along **multiple paths of equal cost**. The forwarding engine identifies paths by **next-hop**. 

* **Hash-Threshold**: One method for determining which next-hop to use when routing with ECMP can be called hash-threshold. The router first selects a **key** by performing a **hash** (e.g., [CRC16][1]) over the **packet header fields** that identify a **flow**. The router uses the key to determine which region and thus which next-hop to use.

* Upon receiving a packet the router performs a CRC16 on the packet’s header fields that define the flow (e.g., the source and destination fields of the packet), this is the key. Say for this destination there are 4 next-hops to choose from. Each next-hop is assigned a region in 16 bit space (**the key space**). For equal usage the router may have chosen to **divide it up evenly** so each region is 65536/4 or 16k large. The next-hop is chosen by determining **which region contains the key** (i.e., the CRC result).

*****

####`Analysis`

* There are a few concerns when choosing an algorithm for deciding which next-hop to use. One is **performance**, the computational requirements to run the algorithm. Another is **disruption** (i.e., the changing of which path a flow uses). **Balancing** is a third concern; however, since the algorithm’s balancing characteristics are directly related to the **chosen hash function** this analysis does not treat this concern in depth.

* The **performance** of the hash-threshold algorithm can be broken down into three parts: **selection of regions for the next-hops**, **obtaining the key** and **comparing the key to the regions to decide which next-hopto use**.

* To choose the next-hop we must determine which region contains the key. Because the regions are of equal size determining which region contains the key is a simple division operation.

  *regionsize* = *keyspace*.*size* / #{*nexthops*}

  *region* = *key* / *regionsize*;

* Thus the time required to find the next-hop is dependent on the way the next-hops are **organized in memory**. The obvious use of an **array indexed by region** yields O(1).

*****

* Protocols such as **TCP** perform better if the path they flow along does **not change** while the stream is connected. **Disruption** is the measurement of how many flows have their paths changed due to some change in the router.

* Some algorithms such as **round-robin** (i.e., upon receiving a packet the least recently used next-hop is chosen) are **disruptive regardless** of any change in the router. Clearly this is not the case with **hash-threshold**. As long as the region boundaries remain unchanged the same next-hop will be chosen for a given flow. 

* Because we have required regions to be equal in size the only reason for a change in region boundaries is the addition or removal of a next-hop. In this case the regions must all grow or shrink to fill the key space.

>
>![]({{ site.img_url }}/2014-9-5/region.JPG)
>

* 如上图所示，**region 3** has been deleted. The remaining regions grow equally and shift to compensate. In this case 1/4 of region 2 is now in region 1, 1/2 (2/4) of region 3 is in region 2, 1/2 of region 3 is in region 4 and 1/4 of region 4 is in region 5. Since each of the original regions represent 1/5 of the flows, **the total disruption is 1/5*(1/4 + 1/2 + 1/2 + 1/4) or 3/10**.

* Note that the disruption to flows when **adding a region** is equivalent to that of **removing a region**. That is, we are considering the fraction of total flows that changes regions when moving from N to N-1 regions, and that same fraction of flows will change when moving from N-1 to N regions.

>
>![]({{ site.img_url }}/2014-9-5/region2.JPG)
>

* 如上图所示，**region 4** has been deleted. Again the remaining regions grow equally and shift to compensate. 1/4 of region 2 is now in region 1, 1/2 of region 3 is in region 2, 3/4 of region 4 is in region 3 and 1/4 of region 4 is in region 5. Since each of the original regions represent 1/5 of the flows the, **total disruption is 7/20**.

* To generalize, upon removing a region K the remaining N-1 regions grow to fill the 1/N space. This growth is evenly divided between the N-1 regions and so the change in size for each region is 1/N/(N-1) or 1/(N(N-1)). This change in size causes non-end regions to move. The first region grows and so the second region is shifted towards K by the change in size of the first region. **1/(N(N-1)) of the flows from region 2 are subsumed by the change in region 1’s size. 2/(N(N-1)) of the flows in region 3 are subsumed by region 2**.

* **This is because region 2 has shifted by 1/(N(N-1)) and grown by 1/(N(N-1))**. This continues from both ends until you reach the regions that bordered K. The calculation for the number of flows subsumed from the Kth region into the bordering regions accounts for the removal of the Kth region.

>
>![]({{ site.img_url }}/2014-9-5/ca1.JPG) ![]({{ site.img_url }}/2014-9-5/ca2.JPG) 
>![]({{ site.img_url }}/2014-9-5/ca3.JPG) ![]({{ site.img_url }}/2014-9-5/ca4.JPG) ![]({{ site.img_url }}/2014-9-5/ca5.JPG)
>

* 如上图所示，为disruption的计算公式. Since we are **minimizing for K** the right side **(N+1)/2(N-1) is constant** as is the denominator (N)(N-1) so we can drop them. To minimize we take the derivative.

>
>![]({{ site.img_url }}/2014-9-5/ca6.JPG) ![]({{ site.img_url }}/2014-9-5/ca7.JPG)
>

* 如上图所示，Which is zero **when K is (N+1)/2**.

* The last thing to consider is that **K must be an integer**. When N is odd (N+1)/2 will yield an integer. however when N is even (N+1)/2 yields an integer + 1/2, in the case, because of symmetry, we get the least disruption when K is N/2 or N/2 + 1. If K is 1 or N the formula reduces to 1/2. The minimum possible disruption is obtained by letting K=(N+1)/2. In this case the formula reduces to 1/4 + 1/(4*N). So the range of possible disruption is (1/4, 1/2]. **To minimize disruption we recommend adding new regions to the center rather than the ends**.

*****

####`Comparison to other algorithms`

* **Modulo-N** is a "simpler" form of hash-threshold. Given N next-hops the packet header fields which describe the flow are run through a hash function. A final modulo-N is applied to the output of the hash. This result then directly maps to one of the next-hops. Modulo-N is the **most disruptive** of the algorithms; if a next-hop is added or removed the disruption is **(N-1)/N**. The performance of Modulo-N is equivalent to hash-threshold.

* **Highest random weight (HRW)** is a comparative method similar in some ways to hash-threshold with non-fixed sized regions. For each nexthop, the router seeds a pseudo-random number generator with the packet header fields which describe the flow and the next-hop to obtain a weight. The next-hop which receives the highest weight is selected. The advantage with using HRW is that it has **minimal disruption** (i.e., disruption due to adding or removing a next-hop is
always **1/N**.) The disadvantage with HRW is that the next-hop selection is more expensive than hash-threshold.

* Since each of modulo-N, hash-threshold and HRW require **a hash on the packet header fields which define a flow**, we can factor the performance of the hash out of the comparison. If the hash can not be done inexpensively (e.g., in hardware) it too must be considered when using any of the above methods. 

* The **lookup performance** for hash-threshold, like modulo-N is an optimal **O(1)**. HRW’s lookup performance is **O(N)**.

* **Disruptive behavior** is the opposite of performance. HRW is best with **1/N**. Hash-threshold is **between 1/4 and 1/2**. Finally Modulo-N is **(N-1)/N**.

*****

####`Security Considerations`

* The implementation of ECMP routing decision does not directly affect the security of the Internet Infrastructure.


[1]: http://en.wikipedia.org/wiki/Cyclic_redundancy_check   "cyclic redundancy check"