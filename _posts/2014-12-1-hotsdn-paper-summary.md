---
layout: post
category : lessons
tagline: ""
tags : [tech]
---
{% include JB/setup %}

2012-2014年HotSDN文章总结

*****

###网络操作系统
`——————————`<br>
应用层<br>
`——————————`<br>
北向管理层 (对控制器功能进行集成)<br>
`——————————`<br>
控制器层<br>
`——————————`<br>
交换机层<br>
`——————————`<br>
      
      ONOS: Towards an Open, Distributed SDN OS hotsdn14 Open Networking Laboratory

      The Beacon OpenFlow Controller hotsdn13 Stanford University

      Towards an Elastic Distributed SDN Controller hotsdn13 Purdue University
      distributed controller architecture dynamically adjust the load between switches and controllers

      Software Transactional Networking: Concurrent and Consistent Policy Composition hotsdn13 TU Berlin/T-Labs
      distributed sdn policy specification and control concurrency issues

      Logically Centralized? State Distribution Trade-offs in Software Defined Networks hotsdn12 TU Berlin/T-Labs
      sdn control plane state inconsistency significantly degrades performance of logically centralized control applications

      Corybantic: Towards the Modular Composition of SDN Control Programs HotNets13 HP Labs,Palo Alto

*****

###控制器层编程语言

      A Balance of Power: Expressive, Analyzable Controller Programming hotsdn13 Worcester Polytechnic Institute
      a language to build controller program

      Procera: A Language for High-Level Reactive Network Control hotsdn12 Yale University
      a declarative policy language

*****

###网络虚拟化和NFV
`——————————`<br>
应用层<br>
`——————————`<br>
控制器层<br>
`——————————`<br>
虚拟层 (北向控制器层接口, 向上提供虚拟资源, 南向交换机层接口, 维护上层虚拟资源与底层物理资源的映射)<br>
`——————————`<br>
交换机层<br>
`——————————`<br>
网络虚拟化：主旨是在流层面对网络进行逻辑划分(类似于对硬盘进行分区)，从而为现有网络创建逻辑分区。<br>
网络功能虚拟化：主旨是在网络虚拟化的基础上对四到七层功能进行虚拟化处理，其中包括防火墙、负载平衡机制、入侵检测系统，此举的目的在于帮助人们为虚拟机或者传输流创建一套服务配置方案。<br>

      OpenVirteX: Make Your Virtual SDNs Programmable hotsdn14 Open Networking Laboratory

      Towards Correct Network Virtualization hotsdn14 University of Illinois at Urbana-Champaign

      Fast, Accurate Simulation for SDN Prototyping hotsdn13 University of Wisconsin
      introduce fs-sdn that a simulation based tool and compare with mininet

      High-Fidelity Switch Models for Software-Defined Network Emulation hotsdn13 University of California, San Diego
      emulator proxy between controller and ovs

      Splendid Isolation: A Slice Abstraction for Software-Defined Networks

*****

###交换机层转发机制

      Application-aware Data Plane Processing in SDN hotsdn14 University of Minnesota
      stateless->stateful

      Using SDN To Facilitate Precisely Timed Actions On Real-Time Data Streams hotsdn14 FOX Network Engineering & Operations
      the changing of forwarding rules with a temporal accuracy
      
      Flow-level State Transition as a New Switch Primitive for SDN hotsdn14 University of Southern California
      preinstall state machine and automatically installed rules

      Optimizing Rules Placement in OpenFlow Networks: Trading Routing for Better Efficiency hotsdn14
      optimization problem trade-off between routing requirement and traffic delivering

      Pratyaastha: An Efficient Elastic Distributed SDN Control Plane hotsdn14 University of Wisconsin–Madison
      decrease in flow setup latency and reduction in controller operating costs

      EtherPIPE: an Ethernet character device for network scripting hotsdn13 Keio University
      fpga driver that access traffic data as file

      Resource/Accuracy Tradeoffs in Software-Defined Measurement hotsdn13 University of Southern California
      tradeoff space of resource usage versus accuracy of primitives

      OF.CPP: Consistent Packet Processing for OpenFlow hotsdn13 EPFL
      use transactional semantics at the controller to achieve consistent packet processing

      Cheap Silicon: a Myth or Reality? Picking the Right Data Plane Hardware for Software Defined Networking hotsdn13 Ericsson Research
      compare data plane programmable hardware with hard coded chip

      Splendid Isolation: A Slice Abstraction for Software-Defined Networks hotsdn12 Cornell
      abstract network slice with vlan

*****

###Rule Caching

      Flow Caching for High Entropy Packet Fields hotsdn14 Stanford University
      cache flow entries with high entropy that change often

      A Compressive Method for Maintaining Forwarding States in SDN Controller hotsdn14 Ericsson Research
      efficiently store redundancy across the flow tables of all switches

      Infinite CacheFlow in Software-Defined Networks hotsdn14 Princeton University
      efficient rules caching and solve the dependency

      CAB: A Reactive Wildcard Rule Caching System for Software-Defined Networks hotsdn14 New York University
      wildcard rules caching to preserve the flow table space and solve the dependency

*****

###Switching and Routing

      Using MAC Addresses as Efficient Routing Labels in Data Centers hotsdn14 University of Paderborn
      hierarchical mac addresses

      Shadow MACs: Scalable Label-switching for Commodity Ethernet hotsdn14 IBM Research
      label switching with mac addresses

      Fabric: A Retrospective on Evolving SDN hotsdn12 Nicira
      adopt the insight underlying MPLS in sdn network

*****

###Rule Updating

      ESPRES: Transparent SDN Update Scheduling hotsdn14 Université catholique de Louvain
      rules update installation->scheduling problem

      Compiling Minimum Incremental Update for Modular SDN Languages hotsdn14 Northwestern University
      handle policy updates and reduce redundant rule updates

      Incremental Update for a Compositional SDN Hypervisor hotsdn14 Princeton University
      Compositional SDN Hypervisor reduce computational overhead and the number of rule updates

      Incremental Consistent Updates hotsdn13 Princeton University
      tradeoff between time and rule space

      A Safe, Efficient Update Protocol for OpenFlow Networks hotsdn12 HP Labs,Palo Alto
      packet consistency->flow consistency

      Walk the Line: Consistent Network Updates with Bandwidth Guarantees hotsdn12 University of Illinois at Urbana-Champaign
      vm migration sequence and rule updating sequence

*****

###Middlebox
middlebox：是一种计算机网络设备，除了一般的转发行为外，还负责流量的转化、检测、过滤等。常见的例子包括防火墙机制(隔离恶意流量), 网络地址转换(修改数据报的源目地址)。专用的middlebox硬件设备被广泛地部署在企业级网络中, 以提升performance和security。

      Don’t Call Them Middleboxes, Call Them Middlepipes hotsdn14 IBM Research
      middlebox endpoint->pipe

      Enabling Fast, Dynamic Network Processing with ClickOS hotsdn13 NEC Europe Ltd
      dynamically instantiate and quickly move middlebox functionality

      FlowTags: Enforcing Network-Wide Policies in the Presence of Dynamic Middlebox Actions hotsdn13 Stony Brook University
      flow tracking

*****

###容错性

      Five Nines of Southbound Reliability in Software-Defined Networks hotsdn14 University of Murcia
   
      Fleet: Defending SDNs from Malicious Administrators hotsdn14 CMU/ETH Zurich

      Provable Data Plane Connectivity with Local Fast Failover hotsdn14 Ben Gurion Universit
      Introducing OpenFlow Graph Algorithms

      Cementing High Availability in OpenFlow with RuleBricks hotsdn13 IBM Research
      introduce key primitives express flow assignment and backup policies

      HotSwap: Correct and Efficient Controller Upgrades for Software-Defined Networks hotsdn13 Princeton University
      introduce a hypervisor to upgrade controller with disruption free and correct manner

      CAP for Networks hotsdn13 UC Berkeley
      theorem showed that it is impossible to achieve all three of strong consistency, availability and partition tolerance

      FatTire: Declarative Fault Tolerance for Software-Defined Networks hotsdn13 Cornell University
      a language for writing fault-tolerant network programs

*****

###OpenFlow协议改进

      tinyNBI: Distilling an API from essential OpenFlow abstractions hotsdn14 Texas A&M University
      openflow->simple API

      Protocol-Oblivious Forwarding: Unleash the Power of SDN through a Future-Proof Forwarding Plane hotsdn13 Huawei Technologies, USA
      separate controlling and forwarding

      Hey, You Darned Counters! Get Off My ASIC! hotsdn12 HP Labs
      replace traditional counters with a stream of rule-match records

      Hierarchical Policies for Software Defined Networks hotsdn12 Brown University
      openflow pipeline->tree

*****

###对接传统网络
      
      ProCel: Smart Trafﬁc Handling for a Scalable Software EPC hotsdn14 Stanford University
      EPC(Evolved Packet Core)，4G的核心网
      EPC主要有三部分：
	   1、MME（Mobility Management Entity：负责信令处理部分）
	   2、S-GW（Serving Gateway：负责本地网络用户数据处理部分）
	   3、P-GW（PDN Gateway：负责用户数据包与其他网络的处理）

      The FlowAdapter: Enable Flexible Multi-Table Processing on Legacy Hardware hotsdn13 Chinese Academy of Sciences
      solve the incompatibility of flow table pipeline between legacy switch hardware and the controller

      Using CPU as a Traffic Co-processing Unit in Commodity Switches hotsdn12 Microsoft Research Asia
      use CPU in the switches to handle data plane traffic

      Outsourcing Network Functionality hotsdn12 Stanford University
      ISP enterprise network for forwarding data and external feature provider for managing feature

      A Management Method of IP Multicast in Overlay Networks using OpenFlow hotsdn12 Fujitsu Laboratories Ltd
      management vxlan IP multicast by using openflow but IGMP

*****

###多域SDN网络
      
      A Resource Delegation Framework for Software Deﬁned Networks hotsdn14 RENCI/UNC Chapel Hill
      controller placement and coordination/big switch abstraction problem->resource management problem
      controller placement and coordinaton：
         The controller placement problem hotsdn12 Stanford University
         DISCO:Distributed Multi-domain SDN Controllers CoRR13
      single programmable big switch abstraction:
         Extending SDN to Large-Scale Networks ons13
         Open Transport Switch-A Software Defined Networking Architecture for Transport Networks hotsdn13 Infinera Corporation
      nfv

      Toward Systematic Detection and Resolution of Network Control Conﬂict hotsdn14 Naval Postgraduate School

      Exploiting Locality in Distributed SDN Control hotsdn13 TU Berlin&T-Labs,Germany
      combine distributed computing with controller local view

      Revisiting Routing Control Platforms with the Eyes and Muscles of Software-Defined Networking hotsdn12 CPqD
      combine sdn controller with routing control platform

      Kandoo: A Framework for Efficient and Scalable Offloading of Control Applications hotsdn12 University of Toronto
      process frequent events in the data plane->two layers of controllers

*****

###应用层

      Testing Stateful and Dynamic Data Planes with FlowTest hotsdn14 Carnegie Mellon University
      explore the state space of the network data plane

      Distributed and Collaborative Traffic Monitoring in Software Defined Networks hotsdn14 Unversity of Kentucky
      traffic monitoring

      An Assertion Language for Debugging SDN Applications hotsdn14 Princeton University
      verify and debug SDN applications with dynamically changing verification conditions

      SDN traceroute: Tracing SDN Forwarding without Changing Network Behavior hotsdn14 IBM Research
      traffic monitoring

      Enabling Layer 2 Pathlet Tracing through Context Encoding in Software-Defined Networking hotsdn14 NEC Laboratories
      traffic monitoring

      Compiling Path Queries in Software-Defined Networks hotsdn14 Princeton University
      traffic monitoring

      Leveraging SDN Layering to Systematically Troubleshoot Networks hotsdn13 Stanford University
      layering and troubleshoot

      VeriFlow: Verifying Network-Wide Invariants in Real Time hotsdn12 University of Illinois at Urbana-Champaign
      check for invariant violations as each rule is inserted

      Where is the Debugger for my Software-Defined Network? hotsdn12 Stanford University

      Dynamic Graph Query Primitives for SDN-based Cloud Network Management hotsdn12 IBM Research
      provide underlying gragh view

      Programming Your Network at Run-time for Big Data Applications IBM Research
      integrated control application management

*****

###Security

      FLOWGUARD: Building Robust Firewalls for Software-Defined Networks hotsdn14 Clemson University

      Towards Secure and Dependable Software-Defined Networks hotsdn13 University of Lisbon
      sketch the design of a secure and dependable SDN control platform

      A Security Enforcement Kernel for OpenFlow Networks hotsdn12 SRI International

      OpenFlow Random Host Mutation: Transparent Moving Target Defense using Software Defined Networking hotsdn12 University of North Carolina at Charlotte

*****

###Wireless

      SoftRAN: Software Defined Radio Access Network hotsdn13 Stanford University

      OpenRadio: A Programmable Wireless Dataplane hotsdn12 Stanford University

      Towards Programmable Enterprise WLANs with Odin hotsdn12 Instituto Superior Tecnico