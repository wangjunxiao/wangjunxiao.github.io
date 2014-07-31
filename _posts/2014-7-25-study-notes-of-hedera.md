---
layout: post
category : notes
tagline: ""
tags : [research]
---
{% include JB/setup %}

Hedera: 数据中心网络动态流调度_NSDI_2010

*****

####`Motivation`

>
>![]({{ site.img_url }}/2014-7-25/topo.JPG)
>

* ECMP冲突导致网络中的[对剖带宽][1]减少

>
>![]({{ site.img_url }}/2014-7-25/loss.JPG)
>

* ECMP导致[fat-tree][2](k=48)网络中每Host随流数量变化对应的对剖带宽损失

*****

####`Implementation`

>
>![]({{ site.img_url }}/2014-7-25/struc.JPG)
>

* 数据中心内部网络交换机运行OF协议，控制网络为简单的星型结构，控制网络与内部网络的连接通过48端口的中心交换机

* 当有流经过边缘交换机时，若不匹配交换机TCAM和SRAM中的流表，交换机会基于ECMP添加新的流表项，将其转发到合适端口，使得该流随后到来的数据包匹配流表项实现线速转发

* 交换机运行OF协议，维护对每个流、每个端口的统计信息，例如总字节数、流生命周期等。Flow Scheduler通过OpenFlow Controller轮询边缘交换机的流统计信息，实现对大象流的侦测（该流的流量的增长超过设定的阀值）

* 对于大象流，其当前带宽需求不能准确反映其实际带宽需求，因此Flow Scheduler会周期性地对侦测的大象流执行带宽需求估计

>![]({{ site.img_url }}/2014-7-25/est1.JPG)
>
>![]({{ site.img_url }}/2014-7-25/est2.JPG)
>
>![]({{ site.img_url }}/2014-7-25/est3.JPG)

* 带宽需求估计算法的伪代码

*****

* 需求估计中使用N×N的矩阵M，N为host数量，M中第i行第j列元素由三个值组成: (1) 从host[i]到host[j]的流的数量; (2) 从host[i]到host[j]的每个流的估计带宽需求; (3) 标志流是否已收敛的布尔变量

* 举例，考虑4个host（host0，host1，host2，host3）的情况。host0各发送1个流到host1、host2和host3; host1发送2个流到host0，1个流到host2; host2各发送1个流到host0和host3; host3发送2个流到host1

>
>![]({{ site.img_url }}/2014-7-25/est4.JPG)
>

* 带宽需求估计的伪代码演示

* 带宽需求估计的算法实现使用了并行计算技术（有利于扩展），稀疏矩阵的数据结构（周期内某host只会与其余远端host的一个小子集产生大象流）

*****

* 利用估计带宽需求，执行调度算法，修改大象流原匹配流表项，将流重定向到其他选定路径，提升内部网络的对剖带宽。这里提出两种调度算法: (1) Global First Fit; (2) [Simulated Annealing][3]

>
>![]({{ site.img_url }}/2014-7-25/glo.JPG)
>

* Global First Fit调度算法的伪代码

>
>![]({{ site.img_url }}/2014-7-25/sim.JPG)
>

* Simulated Annealing调度算法的伪代码

* State s: 一组目的host与核心交换机间的映射，每个pod中的每个host都对应指定一台核心交换机(并不是为每个流分别指定，因此极大地减少了算法的搜索空间)，该host只接受来自指定核心交换机的流量

* INIT-STATE(): 本次调度算法的初始state被设置为上次调度的结果state（best state）

* NEIGHBOR(s): 将一个pod中的host分配不同的核心交换机，等概率地随机选择三种实现方法中的一种来产生一个new state，避免迭代陷入局部最优解

* E(s): 对于新energy的计算，由于迭代过程中一次仅为一对host交换核心交换机，只影响以这两台host为目的的流。因此计算新energy时只需重新计算涉及到这两台host的link（不需要重新计算所有link），并基于当前状态下的energy，更新energy的值

*****

* 在容错方面，Hedera增加实现PortLand路由和容错协议。因此通过基本的Portland机制，Hedera能够感知故障，并将映射到故障组件的流重路由

 [1]: http://en.wikipedia.org/wiki/Bisection_bandwidth    "bisection bandwidth "
 [2]: http://en.wikipedia.org/wiki/Fat_tree      "fat tree"
 [3]: http://en.wikipedia.org/wiki/Simulated_annealing     "simulated annealing"