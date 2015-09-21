---
layout: post
category : notes
tagline: ""
tags : [research]
---

{% include JB/setup %}

Identifying SDN State Inconsistency in OpenStack_SOSR_2015

*****

##`MOTIVATION`

Openstack中的Edge-based SDN实现机制可概括为Server-DB-Agent-Client模式，可能会产生State Inconsistency，原因可分为三方面：

####Unreliable State Dissemination：
一方面，Server（SDN Controller）向Client发送的网络配置，可能因为网络连接的不稳定，导致out of sync（Agent执行了配置任务，但是Server没有收到Agent回复）；另一方面，这些网络配置（对Iptables和OVS的配置）可能与Client上的其他软件或权限冲突（Server收到了Agent回复，但配置没有生效）。

####Human Errors：
网络管理员对Client执行的reboot和patch操作，执行过程中产生错误，但是Server没有及时感知到。

####Software Bugs：
Edge-based SDN自有的bug使得Server（SDN Controller）发送的网络配置，不能最终映射为底层实现，并且很难检测到。

####Inconsistency Example：

>
>![]({{ site.img_url }}/2015-9-21/1.JPG)
>

如上图所示，Nova-Network的实现机制为，不同子网间的通信需要经过计算节点内部的vRouter转发。例如，vm1想和vm4通信必须先将流量经由vRouter转发给vm2，之后经由L2转发给vm4，实现通信。但是，如果此时vm2未创建成功（或发生故障），而Server（SDN Controller）又没有感知到。就会产生State Inconsistency，影响正常通信。

*****

##`IMPLEMENTATION`

>
>![]({{ site.img_url }}/2015-9-21/2.JPG)
>

如上图所示，State Verification System的实现基于的是Neutron L3 Agent架构，包括三个子系统：Data Collection Subsystem，分为两部分，Agent，运行在Client（Network Node和Compute Node），收集网络状态，Data Collector，运行在Verification Server，接收收集到的网络状态，并从DB中查询网络配置；Data Processing Subsystem（State Parser），读取Data Collector中的网络状态，并以特定表示方法将收集到的网络状态格式化（形成L2，L3，L4状态表示）；State Verification and Reporting Subsystem，分析格式化后的L2，L3，L4状态，并生成Inconsistency Alerts。

>
>![]({{ site.img_url }}/2015-9-21/3.JPG)
>

*****

##`NETWORK STATE ABSTRACTION`

> L2状态：
>![]({{ site.img_url }}/2015-9-21/4.JPG)
>

i表示vm的mac address，j表示vm所在的L2网络，可能是vlanId（vlan），或者是VNI（gre和vxlan）。该Map=1，说明vm的L2可达。

*****

> L3状态：
>![]({{ site.img_url }}/2015-9-21/5.JPG)
>

IP-MAC表示vm的ip and mac address，r=1，说明vm间L3可达。

>
>![]({{ site.img_url }}/2015-9-21/6.JPG)
>

特别地，当vm绑定了Floating Ip时，该map=1，说明vm可NAT到external。

*****

> L4状态：
>![]({{ site.img_url }}/2015-9-21/7.JPG)
>

主要考虑Security Group中定义的L4 rules（底层映射为Compute Node中的Iptables rules），这里将每条rule的匹配域定义为一个包含5元组的bitmap，src-ip（32 bits），dst-ip（32 bits），protocol（2 bits），src-port（16 bits），dst-port（16 bits），共98 bits，并且将这个bitmap以[BDD][1]的数据结构表示。如上图所示，每个BDD有两个端点，0表示reject，1表示accept，Bn表示第n bit需要检查。（a）表示128.0.0.0/28（前4 bit分别是1000）的BDD，（b）表示192.0.0.0/28（前4 bit分别是1100）的BDD。去掉（a）（b）中的B2，那么就有了（c），由此可见，BDD这一数据结构在Iptables rule匹配中的灵活优势，使用BDD，可以简化对rules中匹配域的并，交，差运算。

*****

##`STATE PARSING`

L2：检查是否vm的internal vlanId被映射为同一个external L2 Id（vlan type：vlanId，gre/vxlan type：VNI），若符合，则L2可达。

L3：检查是否vm在L2可达，若可达，说明在一个L3 Subnet下，故L3可达，否则，观察vm所在Subnet间，在Network Node上是否有vRouter，若有，则需检查vRouter的路由表和Iptables，路由表中需有Subnet间的路由表项，Iptables中需有Floating IP的NAT表项，若符合，则L3可达。

L4：分析Iptables Chain（FORWARD），需对整个Forward Chain进行顺序遍历，在此基础上，如果考虑SNAT以及DNAT，还需遍历PREROUTING以及POSTROUTING。

>
>![]({{ site.img_url }}/2015-9-21/8.JPG)
>

其中A/D表示该链中最终accept/drop的包集合，R表示该链中最终return的包集合，C表示该链中call其他链的包集合，M为匹配当前rule的包集合。
<br>
另外，由于rules的匹配域与BDD之间的转换比较耗时，这里选择在转换后对BDD，以匹配域的字符串作为索引的形式，进行缓存以提高效率。

*****

##`STATE VERIFICATION`

L2：直接匹配Map。

L3：分为两种情况，public L3直接同L2一样，匹配Map；internal L3匹配Matrix。

L4：可以选择匹配BDD（A，D，R），但这种方式开销过大。

>
>![]({{ site.img_url }}/2015-9-21/9.JPG)
>

改进：vm-shared chains（来源于Security Group Template）和vm-specified chains区别对待，可对vm-shared chains进行String level匹配，对vm-specified chains进行BDD level匹配，减小开销。

>
>![]({{ site.img_url }}/2015-9-21/10.JPG)
>

特殊情况：Compute Node内的rule（vm-shared chains）与vm内的rule产生冲突（由于Controller Node同Compute Node间通信失败，software bugs），这时需对发生mismatch的vm进行BDD level匹配。
Compute Node不执行vm-shared chains（由于网络管理员的操作），这时需对Compute Node内的所有rule进行BDD level匹配。


[1]: http://ieeexplore.ieee.org/xpl/articleDetails.jsp?arnumber=1676819    "Graph-Based Algorithms for Boolean Function Manipulation"