---
layout: post
category : notes
tagline: ""
tags : [research]
---
{% include JB/setup %}

DCN Architecture Design Notes of Fat-Tree

*****

####`Introduction`

* fat-tree网络架构由Charles E. Leiserson在1985年提出。[ “Fat-Trees: Universal Networks for Hardware-Efficient Supercomputing” ]

>
>![]({{ site.img_url }}/2014-8-5/fat1985.JPG)
>

* 上图所示的设计中交换机的端口数随交换机所处层次不同而不同。然而，企业网中商品交换机的端口数是固定的。因此，另外一种拓扑被提出，用于有效利用端口数固定的交换机

>
>![]({{ site.img_url }}/2014-8-5/fat2.JPG)
>

* fat-tree最明显的特点是对于任何一个交换机，向下通往子女节点的链路数量与向上通往双亲节点的链路数量相等。并且沿树根方向，链路变得越来越fat，因此得名fat-tree

* 如果某个host想与相同边缘交换机下的host通信，只需通过特定的边缘交换机，而不需通过核心交换机。如果想与不同边缘交换机下的host通信，则必须先将数据包发送到任意核心交换机，再由核心交换机发送到目标边缘交换机

* 上图所示的设计中每个中间交换机有多个双亲节点，这与fat-tree的初始设计不同。因此有人说上图所示的设计不应被称为fat-tree。然而，上图所示这样的拓扑结构我们仍然称之为fat-tree

*****

####`Blocking`

* 上面提到的fat-tree网络在每个中间层保证: 通往上层（在二层网络中指的是边缘层通往核心层）的链路数量与通往下层（边缘层通往host）的链路数量相等（1: 1），这样的网络称之为non-blocking网络

* 通往上层与下层的链路数量之间的比例可以变化。假如这个比例为1: 2，那么网络的blocking因子为2。non-blocking网络的blocking因子为1

*****

####`Design`

* 利用36-port商品交换机构建non-blocking的二层fat-tree网络连接70个节点(N=70, Pc=Pe=36):

1. Eptn(edge ports to nodes)=Eptc(edge ports to core)=36/2=18
2. E(num of edge)=Ceil(N/Eptn)=Ceil(3.88)=4
3. B(link of bundle)=Pc div E=9
4. C(num of core)=Ceil(Eptc/B)=2
