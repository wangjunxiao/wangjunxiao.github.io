---
layout: post
category : notes
tagline: ""
tags : [research]
---

{% include JB/setup %}

PortLand: 可扩展的二层容错数据中心网络架构_SIGCOMM_2009

*****

`Introduction`

* 理想情况下，DCN架构和管理员将对交换机使用“[即插即用][1]”的部署方案。这样一个场景相应的需求: 

1. 任意VM可以迁移到任意物理机中。迁移VM时不需改变其IP地址（改变的话会破坏其预先存在的TCP连接以及应用层状态）
2. 部署之前管理员不需对任何交换机进行配置
3. 任意host可通过任意可用物理链路与其他host通信
4. 转发无环路
5. 故障检测应迅速、有效。只要底层物理连接允许，现有的单播和组播会话就不受影响

******

`Fabric Manager`

* PortLand部署了一个逻辑集中的fabric manager，维护网络配置信息例如拓扑

* fabric manager作为用户进程运行在一台专用计算机或单独的控制网络上，协助ARP解析、容错、组播

* PortLand不需要管理员对fabric manager进行例如交换机数量、位置和标识的配置

* fabric manager的主要异步更新状态存在一到多个备份，但副本间无需保证严格的一致性

*****

`Positional Pseudo MAC Addresses`

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

`Proxy-based ARP`

* 以太网默认将ARP请求广播给同个二层域中的所有host。PortLand利用fabric manager来减轻这一广播负载

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

`Distributed Location Discovery`

* PortLand交换机利用它们在全局拓扑中的位置执行有效地转发和路由。因此在确定位置前交换机不进行包转发，并且不需要管理员的人为设置和干预。为了完全实现PortLand的“即插即用”，提出location discovery protocol（LDP）

* PortLand交换机周期性向所有端口发送Location Discovery Message（LDM），用于确定位置并检测连通性

* LDM包含以下信息:

1. *Switch identifier（switch_id）*: 每个交换机的全局唯一标识。例如，本地所有端口的最低位MAC地址
2. *Pod number（pod）*: 同个pod内的所有交换机使用一个pod号。核心交换机无pod号 
3. *Position（pos）*: 每个pod内的边缘交换机有唯一的位置号
4. *Tree level（level）*: 取值0、1和2分别代表边缘交换机、汇聚交换机和核心交换机
5. *Up/down（dir）*: 布尔值，代表交换机端口在多根树中是向上或向下

* PortLand初始时，除了switch_id和port number外其他值是未知的。假定所有交换机的端口处在三个状态: 断开连接; 与host直连:与交换机直连

* 通过LDP确定交换机level的关键就是边缘交换机只能收到来自汇聚交换机的LDM（另外一半端口与不发送LDM的host直连）

* 边缘交换机当发现自己部分端口没有收到LDM（与host直连），确定自己的level为0

* 汇聚交换机当发现自己部分端口收到level为0的LDM（来自边缘交换机），确定自己的level为1

* 核心交换机当发现自己所有端口收到level为1的LDM（来自汇聚交换机），确定自己的level为2

>
>![]({{ site.img_url }}/2014-8-14/a1.JPG)
>
>![]({{ site.img_url }}/2014-8-14/a2.JPG)
>

* 如上所示算法展示了每个交换机收到LDM后执行的操作

* 考虑到DCN中可能存在的配线错误，例如将边缘交换机的两个端口直接用以太网电缆连接起来，这样就会引入环路并影响PortLand的转发机制。在这种情况下，对于未初始化结束的交换机，收到来自边缘交换机的LDM时有两种可能: 一、这个交换机是汇聚交换机; 二、这个交换机是边缘交换机并且发生了配线错误。这时，如果这个交换机收到了来自汇聚交换机的LDM或发现大部分端口与host直连，就可以断定边缘交换机发生了配线错误

*****

`Provably Loop Free Forwarding`

* 一旦交换机利用LDP确定了位置，就会从邻居交换机获取信息并填充自身的转发表。例如，核心交换机从直连的汇聚交换机获取pod号。转发数据包时，核心交换机简单地检查数据包PMAC对应的pod号就能确定适当的输出端口。同样地，汇聚交换机从直连的边缘交换机获取position号。汇聚交换机必须通过检查数据包的PMAC确定目的host是否位于本pod中，如果目的host位于本pod中，那么汇聚交换机就将数据包转发到PMAC指定position所对应的边缘交换; 如果目的host位于其他pod中，（无故障时）汇聚交换机可以将数据包转发到任意一个直连的核心交换机。在考虑负载均衡的情况下，可以借助相关技术选择合适的输出端口。fabric manager将会下发一系列流表项重写默认的单个流转发行为

* 一种基础技术——flow hashing in ECMP: PortLand通过确定的哈希函数将multicast group映射到一个核心交换机，交换机将所有的multicast数据包转发到这个核心交换机，也就是利用流哈希在可用路径之中进行挑选。（无故障时，编码在SRAM中的故障除外）通过简单的硬件支持就可以实现该哈希函数。没有硬件支持的话，每个multicast group对应一个流表项。边缘交换机利用涉及到host的PMAC向fabric manager发出IGMP请求。fabric manager将在所有的核心交换机和汇聚交换机中安装转发状态，务必确保multicast数据包传输到目的host子网所在的边缘交换机

* PortLand避免transient loop与broadcast storm的关键是一个数据包一旦在拓扑中向下传输了，就不可能返方向向上传输。在罕见的故障情况下，数据包会绕回到汇聚交换机再发往能够到达目的host的核心交换机。在安全方面，PortLand宁愿失去这些故障连接也要避免出现loop

* PortLand无环证明: fat-tree网络拓扑中存在许多物理loop。然而，DCN中物理loop的存在是有益的，例如增大了网络对剖带宽和容错能力。传统以太网在降低对剖带宽和容错能力的前提下利用minimum spanning tree以防止转发loop。相比较，PortLand中防止转发loop的措施不需要利用最小生成树，只需要一个简单、无状态、本地化、全局统一的约束

* **Constraint 1**. *A switch must never forward a packet out along an upward-facing port when the ingress port for
that packet is also an upward-facing port.*

* **Theorem 1**. *When all switches satisfy Constraint 1 (C1), a fat tree will contain no forwarding loops.*

* **Proof**. C1 prevents traffic from changing direction more than once. It imposes the logical notion of up-packets and down-packets. Up-packets may travel only upward through the tree, whereas down-packets may travel only downward. C1 effectively allows a switch to perform a one-time conversion of an up-packet to a down-packet. There is no provision
for converting a down-packet to an up-packet. In order for a switch to receive the same packet from the same ingress port more than once, this packet should change its direction at least twice while routed through the tree topology. However this is not possible since there is no mechanism for converting a down-packet to an up-packet, something that would be required for at least one of these changes in direction.

*****

`Fault Tolerant Routing`

* LDP除了确定交换机位置外的一个重要作用就是liveness monitoring session 

>
>![]({{ site.img_url }}/2014-8-14/fault1.JPG)
>

* 如上图所示的**unicast fault detection**，如果*step.1*交换机在一段时间内没有收到LDM，就会认定链路发生了故障

* *step.2*检测到故障的交换机就会向fabric manager汇报该故障

* *step.3*fabric manager负责维护并更新一个表征每条链路连接状态的故障矩阵，k-port的fat-tree网络中交换机间链路数量(链路连接状态)为k^3/2

* *step.4*fabric manager将故障通知到受影响的交换机，交换机基于故障拓扑计算转发表

* 传统路由协议使用all-to-all的通信模式，因此在n个交换机的情形下，故障避免带来O(n^2)级别的网络通信负载。而PortLand是由故障交换机发送一条信息到fabric manager，最糟糕时由fabric manager发送n条信息到n个受影响的交换机，因此故障避免带来O(n)级别的网络通信负载

>
>![]({{ site.img_url }}/2014-8-14/fault2.JPG)
>

* 如上图所示的**multicast fault detection**，一个multicast group映射到最左边的核心交换机。三个接受host分布在pod0和pod1，一个发送host先将数据包发送到核心交换机，反过来核心交换机再将数据包发送到接受host

* *step.1*两条链路同时故障

* *step.2*故障所在的汇聚交换机检测到故障，并通知fabric manager

* *step.3*fabric manager更新故障矩阵

* *step.4*fabric manager为受影响的multicast group计算转发项（贪婪）

* *step.5*fabric manager将所需的转发状态插入受影响的交换机

>
>![]({{ site.img_url }}/2014-8-14/fault.JPG)
>

* 如上图所示的**multicast fault recovery**，multicast发送host所在子网的边缘交换机将数据包复制并发往两个不同的核心交换机，由他们转发给接受host

*****

`System Architecture`

>
>![]({{ site.img_url }}/2014-8-14/architecture.JPG)
>

* 边缘交换机将ARP request和IGMP join request拦截下来并转发给local switch module，local switch module通过OpenFlow与fabric manager交互，负责解决ARP请求和管理multicast session的转发表

* 新流的前几个数据包因为缺少匹配的流表项，所以被转发到local switch module。local switch module利用ECMP的hashing在可用链路间选择，并且插入一条新的流表项来匹配这个流

* 交换机接收来自fabric manager的故障以及恢复的相关通知，更新全局链路状态，并对受影响流的转发表进行修改

* fabric manager利用OpenFlow将control update传输给每个交换机的local switch module

>
>![]({{ site.img_url }}/2014-8-14/t.JPG)
>

* the state maintained locally at each switch as well as the fabric manager. Here:

  *k = Number of ports on the switches,*

  *m = Number of local multicast groups,*

  *p = Number of multicast groups active in the system.*

[1]: http://en.wikipedia.org/wiki/Plug_and_play    "plug and play"
