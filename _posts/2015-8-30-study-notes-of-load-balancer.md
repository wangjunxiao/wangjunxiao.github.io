---
layout: post
category : notes
tagline: ""
tags : [research]
---

{% include JB/setup %}

应用负载均衡总结

*****

LB部署方式：路由模式（推荐）<br>
服务器的网关必须设置成负载均衡机的LAN口地址，且与WAN口分署不同的逻辑网络，因此所有返回的流量也都经过负载均衡。这种方式对网络的改动小，能均衡任何下行流量。

>
>![]({{ site.img_url }}/2015-8-30/1.jpg)
>

桥接模式（不推荐）<br>
负载均衡的WAN口和LAN口分别连接上行设备和下行服务器，LAN口不需要配置IP（WAN口与LAN口是桥连接），所有的服务器与负载均衡均在同一逻辑网络中。由于这种安装方式容错性差，网络架构缺乏弹性，对广播风暴及其他生成树协议循环相关联的错误敏感。

>
>![]({{ site.img_url }}/2015-8-30/2.jpg)
>

DSR（Direct Server Reply）模式<br>
负载均衡的LAN口不使用，WAN口与服务器在同一个网络中，互联网的客户端访问负载均衡的虚IP（VIP），VIP对应负载均衡机的WAN口，负载均衡根据策略将流量分发到服务器上，服务器直接响应客户端的请求。由于返回的流量是不经过负载均衡的，因此这种方式适用大流量高带宽要求的服务。

>
>![]({{ site.img_url }}/2015-8-30/3.jpg)
>

*****

负载均衡应用领域：<br>
DNS负载均衡、代理服务器负载均衡、地址转换网关负载均衡、协议内部支持负载均衡、NAT负载均衡、反向代理负载均衡、混合型负载均衡。

当第一个包发给服务器时，根据负载均衡策略选择后端的服务器，建立相应session，第一个包走的是所谓的slow path，得到session之后，直接将包发给相同的服务器，这时包走的是所谓的fast path。

负载均衡设备，作为OSI 4-7层交换机，自然需要兼容2层和3层的交换协议，也就是说可以把负载均衡设备当作一个路由器或者交换机使用。<br>
负载均衡设备的应用场景，也是要求其必须支持2-3层的交换协议。不过肯定当不了核心网的路由器。这样的话，为了支持7层交换，其必须支持各种7层协议，且要有辅助的一些功能，如health check，简单的DDOS保护等。

	Client –> Destination NAT -> Real Server
	Client请求的是VIP
	Client -> Source NAT –> Real Server
	Real Server看到是LB的IP

在DNAT的基础上引入SNAT的好处是避免了Client与Server之间产生紧耦合，并且可以等到三次握手后，产生七层请求时再选择合适的Real Server。

对SLB来说，health check是一个重要的功能，需要实时确定后端Server的服务是否可用，这样才能为Client提供可靠的服务。Health check分为in-band和out-band。<br>
in-band指通过普通的数据流来确定Server是否可用，例如TCP连接Server超时等。而out-band指LB自发去检测Server的可用性，可以通过不同的方法实现，例如是否Ping通，是否连通，是否可以获取指定内容等，可以自定义。

为了减小SLB的负载，使用DSR模式，LB只转发Client的请求包，而Server的响应直接回复到Client不经过LB。

在DSR模式下，LB设备只转换Client发送包的目的MAC地址（转换为Real Server的MAC地址），而目的IP地址不做任何转换，仍然为VIP（一般情况下，IP一旦转换，响应时就不得不经过LB）。<br>
由于是利用MAC地址来将请求发给Real Server，所以Real Server需要和LB设备处于同一个L2 Domain。但不足之处就是这样Server拿到是Client的Real IP，使得Client和Server之间存在耦合。

>
>![]({{ site.img_url }}/2015-8-30/4.jpg)
>

在DSR模式下，为了使Real Server接收请求数据包，我们必须让Real server也拥有VIP。但是在同一个L2 Domain中，怎么能让两个不同的Host拥有一个相同的IP呢？这里实际上是一个小trick。<br>
在Real server上，我们需要将VIP配置在其loopback上。loopback是一个逻辑的IP接口，在其上面配置的IP不会响应任何的ARP请求。所以即使Real Server的loopback上配置VIP，除了它自己，其它任何设备都不知道，从而也就不会引发ARP冲突。
