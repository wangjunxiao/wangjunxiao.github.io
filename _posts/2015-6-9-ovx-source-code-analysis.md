---
layout: post
category : lessons
tagline: ""
tags : [tech]
---
{% include JB/setup %}

OpenVirteX核心源代码解析

*****

>
>![]({{ site.img_url }}/2015-6-9/1.png)
>

--api

-----server

         JettyServer.java： 利用开源Servlet容器——Jetty，实现一个支持http和https的JSON WEB Server，可以将Jetty容器实例化成一个对象，可以迅速为一些独立运行（stand-alone）的Java应用提供网络和web连接，这里创建了三种权限——user, admin and ui，每种角色可以访问相应的资源——/tenant, /admin and /status
         
         OVXLoginService.java： 实现用户登录，这里创建了三类用户——tenant, admin and ui，检查用户名和密码，对应不同的权限和资源，参照/utils/embedder.py OVXClient

------service
      
-------------handlers

---------------------monitoring： RPC的监测功能的API，继承ApiHandler

---------------------tenant： RPC的虚拟化功能的API，继承ApiHandler

                     AbstractHandler.java： Handler基类

                     ApiHandler.java： API基类

                     HandlerUtils.java： 检测API非法输入

                     MonitoringHandler.java： 将监测功能的RPC定向到相应的API，继承AbstractHandler

                     TenantHandler.java： 将虚拟化功能的RPC定向到相应的API，继承AbstractHandler

             AbstractService.java： Service基类

             AdminService.java： 未实现，继承AbstractService

             MonitoringService.java： 将监测功能的RPC分配给MonitoringHandler来处理，继承AbstractService

             TenantService.java： 将虚拟化功能的RPC分配给TenantHandler来处理，继承AbstractService

    JSONRPCAPI.java： 将不同的请求根据url——/admin, /tenant和/status，定向到相应的服务——AdminService, MonitoringService和TenantService

--core

------cmd

          CmdLineSettings.java： 定义系统变量以及它们的默认值

------io

          ClientChannelPipeline.java： 创建与Controller之间的通信管道，在管道中可以加入需要的Handler，继承OpenflowChannelPipeline

          ControllerChannelHandler.java： 工作在ClientChannelPipeline中的Handler，处理与控制器之间的OF消息，继承OFChannelHandler，通过重写messageReceived接收OF消息，通过JBoss.netty的Channel.write发送OF消息

          HandshakeTimeoutHandler.java： 工作在ClientChannelPipeline和SwitchChannelPipeline的Handler，处理OF三次握手（Hello<->Hello<->Hello）超时，继承SimpleChannelUpstreamHandler

          OFChannelHandler.java： 基类，继承IdleStateAwareChannelHandler

          OpenflowChannelPipeline.java： 基类，实现ChannelPipelineFactory, ExternalResourceReleasable

          OVXEventHandler.java： 定义Switch<T>的handleIO接口

          OVXMessageDecoder.java： 工作在ClientChannelPipeline和SwitchChannelPipeline的Handler，继承FrameDecoder，从IObuffer中解析OF消息

          OVXMessageEncoder.java： 工作在ClientChannelPipeline和SwitchChannelPipeline的Handler，继承OneToOneEncoder，将OF消息转化为IObuffer

          OVXSendMsg.java： 定义Switch<T>, Network<T1,T2,T3>, StatisticsManager和SwitchDiscoveryManager的sendMsg接口

          ReconnectHandler.java： 工作在ClientChannelPipeline中的Handler，继承SimpleChannelHandler

          SwitchChannelHandler.java： 工作在SwitchChannelPipeline中的Handler，处理与交换机之间的OF消息，继承OFChannelHandler，通过重写messageReceived接收OF消息，通过JBoss.netty的Channel.write发送OF消息

          SwitchChannelPipeline.java： 创建与Switch之间的通信管道，在管道中可以加入需要的Handler，继承OpenflowChannelPipeline

    OpenVirteX.java： 入口

    OpenVirteXController.java： 程序启动后自动与交换机建立连接，根据RPC请求与控制器建立连接

    OpenVirtexShutdownHook.java： 加入OpenVirteXController的Run——Runtime.getRuntime().addShutdownHook

--db： mongodb数据库连接，持久化

--elements

---------address： PhysicalIPAddress，OVXIPAddress，IPMapper

---------datapath

---------------role： 

---------------statistics： StatisticsManager，对应每个PhysicalSwitch，以轮询方式向交换机请求FlowStat和PortStat信息

               DPIDandPort.java

               DPIDandPortPair.java

               FlowTable.java

               OVXBigSwitch.java

               OVXFlowEntry.java

               OVXFlowTable.java

               OVXSingleSwitch.java

               OVXSwitch.java

               OVXSwitchCapablities.java

               OVXSwitchSerializer.java

               PhysicalSwitch.java

               PhysicalSwitchSerializer.java

               Switch.java

               XidPair.java

               XidTranslator.java

---------host

             Host.java

             HostSerializer.java

---------link

             Link.java

             OVXLink.java

             OVXLinkField.java

             OVXLinkUtils.java

             PhysicalLink.java

---------network

             Network.java

             OVXNetwork.java

             PhysicalNetwork.java

             Uplink.java

---------port

             LinkPair.java

             OVXPort.java

             OVXPortSerializer.java

             PhysicalPort.java

             PhysicalPortSerializer.java

             Port.java

             PortFeatures.java

    Component.java： 组件(Host, Link<T1,T2>, Network<T1,T2,T3>, OVXPort, OVXSwitch, PhysicalLink, PhysicalPort, Switch<T>)的基类.
    组件有四种状态： INIT——just initialized, not accessible or usable，ACTIVE——capable of handling events and accumulating state (flow entries, counters), etc，INACTIVE——halted, not capable of handling events or accumulating state (e.g. a downed interface)，STOPPED——removed from network representation (e.g. a failed switch)，Some Components can influence the state of other Components (subcomponents) with their own. In general, a subcomponent's state won't affect the component, but the reverse is not true e.g. ports are removed if a switch is removed, but a switch port can be removed without affecting the whole switch.
    定义组件的四种接口： register——Adds this component to mappings, storage, and makes it accessible to OVX as subscribers to components that this one depends on (sets state to INACTIVE from INIT)，boot——Initializes component's services and sets its state up to be activated，unregister——Removes this component and its subcomponents from global mapping and unsubsribes this component from others. (sets state to STOPPED). If this component is a Physical entity, it will also attempt to unregister() Virtual components mapped to it，tearDown——Halts event processing at this component (sets state to INACTIVE) and its subcomponents. If this component is a Physical entity, it will also attempt to tearDown() Virtual components mapped to it

    Mappable.java： 为OVXMap定义接口

    OVXMap.java： This singleton class maintains all the virtual-to-physical and reverse mappings. These encompass switch mappings, link mappings, switch route mappings, the IP address mappings, the tenant ID to virtual network mapping, and the list of MAC addresses.

    Persistable.java： 定义组件的数据库持久化接口

    Resilient.java： The interface that a Component must implement if capable of recovering from failure

--exceptions： 定义各种异常

--linkdiscovery

            LLDPEventHandler.java： 定义handleLLDP接口

            SwitchDiscoveryManager.java： Run discovery process from a physical switch. Ports are initially labeled as slow ports. When an LLDP is successfully received, label the remote port as fast. Every probeRate milliseconds, loop over all fast ports and send an LLDP, send an LLDP for a single slow port. Based on FlowVisor topology discovery implementation

--messages： 继承OF消息，定义OVX消息，实现Devirtualizable和Virtualizable

--packet： 定义报文数据结构

--protocol
            OVXMatch.java： The Class OVXMatch. This class extends the OFMatch class, in order to carry some useful informations for OpenVirteX, as the cookie (used by flowMods messages) and the packet data (used by packetOut messages)

--routing： 定义大交换机内部间路由，虚拟邻居交换机间路由

--util： 实用功能封装



* More Details in [OpenVirteX官方网站][1]

[1]: http://ovx.onlab.us/    "OpenVirteX ovx.onlab.us"