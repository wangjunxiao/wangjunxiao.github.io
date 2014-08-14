---
layout: post
category : lessons
tagline: ""
tags : [tech]
---
{% include JB/setup %}

Centos 6.5中使用floodlight+openvswitch+kvm构建虚拟环境  

### 环境信息

>项目 | 名称 | 版本号
>------------ | ------------- | ------------
>操作系统 | Centos 6.5  | 2.6.32-431.el6.x86_64
>控制器 | Floodlight  | 0.9
>虚拟交换机 | openvswitch  | 1.9.3 LTS
>虚拟主机 | Kernel-based Virtual Machine  | 操作系统自带

### 安装Centos

>光盘安装，使用的镜像为CentOS-6.5-x86_64-bin-DVD1.iso

### 安装Floodlight

>1.安装依赖软件包
>
>     yum install build-essential default-jdk ant python-dev git
>
>2.从github下载floodlight的源码，编译
>	
>     git clone https://github.com/floodlight/floodlight.git
>     ant
>
>3.运行floodlight
>
>     java -jar /services/floodlight/target/floodlight.jar

### 安装OpenvSwitch

>1.安装依赖软件包
>
>     yum install gcc make python-devel openssl-devel graphviz automake rpm-build redhat-rpm-config libtool git
>
>2.安装内核源码头文件包
>
>     yum install kernel-headers kernel-devel kernel-debug-devel
>    
>3.从openvswitch官网下载LTS版本的源码，解压
>
>     wget http://openvswitch.org/releases/openvswitch-1.9.3.tar.gz
>     tar zxvf openvswitch-1.9.3.tar.gz
>
>4.在解压文件夹中执行配置命令
>
>     ./boot.sh
>     ./configure --prefix=/usr localstatedir=/var
>     ./configure --with-linux=/lib/modules/`uname -r`/build
>
>5.编译，安装
>
>     make && make install
>
>6.移除Linux中的Bridge模块，加载OVS的核心模块openvswitch.ko以及KVM用到的网桥兼容模块brcompat.ko
>
>     rmmod bridge
>     insmod datapath/linux/openvswitch_mod.ko
>     insmod datapath/linux/brcompat.ko
>
>7.初始化设备openvswitch，创建ovsdb数据库
>
>     mkdir -p /usr/local/etc/openvswitch
>     ovsdb-tool create /usr/local/etc/openvswitch/conf.db /usr/local/share/openvswitch/vswitch.ovsschema
>
>8.启动ovsdb-server
>
>     ovsdb-server /usr/local/etc/openvswitch/conf.db --remote=punix:/usr/local/var/run/openvswitch/db.sock --pidfile --detach
>
>9.初始化数据库
>
>     ovs-vsctl --no-wait init
>
>10.启动OpenvSwitch daemon，连接到同样的Unix domain socket上
>
>     ovs-vswitchd --pidfile --detach
>
>11.启动OpenvSwitch brcompat daemon
>
>     ovs-brcompatd --pidfile –-detach
>
>12.创建setup.sh脚本，每次启动时执行该脚本即可完成启动
>
>     rmmod bridge
>     insmod datapath/linux/openvswitch.ko
>     insmod datapath/linux/brcompat.ko
>     ovsdb-server /usr/local/etc/openvswitch/conf.db --remote=punix:/usr/local/var/run/openvswitch/db.sock --pidfile --detach
>     ovs-vsctl --no-wait init
>     ovs-vswitchd --pidfile --detach
>     ovs-brcompatd --pidfile --detach
>
>13.创建setdown.sh脚本，每次停止Open vSwitch daemons时执行该脚本即可
>
>     kill `cd /usr/local/var/run/openvswitch && cat ovsdb-server.pid ovs-vswitchd.pid`
>


	
	