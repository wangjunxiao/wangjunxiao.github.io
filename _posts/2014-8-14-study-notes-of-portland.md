---
layout: post
category : notes
tagline: ""
tags : [research]
---

{% include JB/setup %}

PortLand: 可扩展的二层容错数据中心网络架构_SIGCOMM_2009

*****

####`Introduction`

* 理想情况下，DCN架构和管理员将对交换机使用“[即插即用][1]”的部署方案。这样一个场景相应的需求: 

1. 任意VM可以迁移到任意物理机中。迁移VM时不需改变其IP地址（改变的话会破坏其预先存在的TCP连接以及应用层状态）
2. 部署之前管理员不需对任何交换机进行配置
3. 任意host可通过任意可用物理链路与其他host通信
4. 转发无环路
5. 故障检测应迅速、有效。只要底层物理连接允许，现有的单播和组播会话就不受影响

******

####`Fabric Manager`

* PortLand部署了一个逻辑集中的fabric manager，维护网络配置信息例如拓扑

* fabric manager作为用户进程运行在一台专用计算机或单独的控制网络上，协助ARP解析、容错、组播

* PortLand不需要管理员对fabric manager进行例如交换机数量、位置和标识的配置

* fabric manager的主要异步更新状态存在一到多个备份，但副本间无需保证严格的一致性

*****

####`Positional Pseudo MAC Addresses`

* 实现有效的转发、路由以及VM迁移的基础就是分层Pseudo MAC（PMAC）地址。PortLand为每个host分配一个唯一的PMAC，PMAC包含host所在位置的编码。例如，同一个pod内的所有host的PMAC前缀是相同的

* host保持不变，维护自身的actual MAC（AMAC）地址。host进行ARP请求，接收目的host的PMAC。所有的数据包转发都是基于PMAC

* 出口边缘交换机负责将目的PMAC改写为目的host的AMAC，让目的host产生转发是基于AMAC的错觉

* PortLand中每个边缘交换机有唯一的pod号以及pod中的位置号，这些值由Location Discovery Protocol分配

* 所有与边缘交换机直连的host，边缘交换机会为其分配形如*pod.position.port.vmid*的48位PMAC

1. **pod**: 16位，代表边缘交换机的pod号
2. **position**: 8位，代表边缘交换机在pod中的位置
3. **port**: 8位，代表边缘交换机与host直连的port号
4. **vmid**: 16位单调递增，代表物理host中的VM号，超时可重用

>
>![]({{ site.img_url }}/2014-8-14/figure2.JPG)
>

* 如上图所示，当入口边缘交换机收到一个源MAC为之前没有见过的数据包时，首先为其分配一个PMAC，并在本地PMAC表中加入一个表项，将这个host的AMAC和IP映射到其PMAC上，然后将该映射发送给fabric manager，fabric manager利用该映射响应ARP请求

* 本质上，这以一种对host透明并与现有商品交换机兼容的方式，将host对应的位置与标志分离。更重要的是，没有在现有的协议的基础上增加额外的字段。从底层硬件的角度，需要流表项实现由交换机软件定义的PMAC←→AMAC改写。交换机转发表的填充是基于对目的PMAC的最长前缀匹配。OpenFlow为PortLand在商品交换机中提供操作和本地硬件方面的支持

*****

####`Proxy-based ARP`

* 以太网默认将ARP请求广播给同个二层域中的所有host。PortLand利用fabric manager来减轻这一广播负载。

>
>![]({{ site.img_url }}/2014-8-14/figure3.JPG)
>

* 如上图所示: 在*step.1*，某个边缘交换机拦截了一个ARP请求，并在*step.2*将该请求转发给中的fabric manager

* fabric manager查询其PMAC表，寻找ARP请求IP对应的表项。如果找到，那么在*step.3*返回给边缘交换机对应的PMAC

* 边缘交换机在*step.4*发送APR回复给host

* 如果fabric manager在其PMAC表中找不到ARP请求IP对应的表项，（无故障时）fabric manager会将ARP请求发送到任意核心交换机，然后由该核心交换机广播给所有的host

>
>* 如上图所示，PortLand中一次TCP流传输的全过程（假设初始fabric manager的ARP表、边缘交换机的PMAC表为空）:
>
1. 10.5.1.2发出ARP请求想要获取10.2.4.5的MAC
2. ARP请求数据包首先到达10.5.1.2子网所在的边缘交换机，该边缘交换机发现10.5.1.2的AMAC是其PMAC表中没有的
3. 该边缘交换机为10.5.1.2分配一个PMAC，并将ARP请求数据包的源MAC字段改写为PMAC
4. 该边缘交换机在其PMAC表中生成一条表项，将10.5.1.2的IP和AMAC映射到生成的PMAC，然后将PMAC与IP的映射发送给fabric manager
5. 该边缘交换机询问fabric manager是否知道10.2.4.5的PMAC，fabric manager回复不知道（初始fabric manager的ARP表为空），并广播ARP请求到所有的host
6. 经过一系列广播，10.2.4.5子网所在的边缘交换机收到了10.5.1.2的ARP请求，并将其广播给10.2.4.5
7. 10.2.4.5收到并回复ARP请求，该边缘交换机收到ARP回复数据包，并发现10.2.4.5的AMAC是其PMAC表中没有的
8. 该边缘交换机为10.2.4.5分配一个PMAC，并将ARP回复数据包的源MAC字段改写为PMAC
9. 该边缘交换机在其PMAC表中生成一条表项，将10.2.4.5的IP和AMAC映射到生成的PMAC，然后将PMAC与IP的映射发送给fabric manager
10. 经过一系列转发，10.5.1.2子网所在的边缘交换机收到了10.2.4.5的ARP回复，并将数据包的目的MAC字段改写为10.5.1.2的AMAC
11. 10.5.1.2收到10.2.4.5的ARP回复（成功获取了10.2.4.5的PMAC），经过三次握手建立连接后开始传输TCP流
>

* PortLand支持VM迁移。当VM完成从一台物理机迁移到另一台时，VM发送gratuitous ARP并被分配新的PMAC，同时更新fabric manager中的PMAC表

* 默认情况VM迁移后，其他与VM通信的host的ARP cache中仍存放着VM迁移之前的已失效PMAC，故只能等待ARP cache超时。为解决这个限制，fabric manager会向VM迁移之前所在子网边缘交换机发送invalidation message。该边缘交换机根据message建立流表项捕获那些目的PMAC已失效的数据包，并向这些数据包的发送者发送gratuitous ARP，更新他们的ARP cache，这些发送者检测到数据包丢失，会将数据包发送到迁移之后的VM

*****

####`Distributed Location Discovery`

>
>![]({{ site.img_url }}/2014-8-14/a1.JPG)
>
>![]({{ site.img_url }}/2014-8-14/a2.JPG)
>




[1]: http://en.wikipedia.org/wiki/Plug_and_play    "plug and play"