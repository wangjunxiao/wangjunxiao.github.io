---
layout: post
category : lessons
tagline: ""
tags : [tech]
---
{% include JB/setup %}

使用mininet python api构建fat-tree网络

*****

`CLI`提供mininet的命令行服务

`log`提供一些记录的服务

`net`里面包含了最重要的类——`Mininet`类，用于定义网络

Mininet类的构造函数:

	class Mininet( object ):
		def __init__( self,
	 		topo = None,
	 		switch = OVSKernelSwitch,
	 		host = Host,
	 		controller = DefaultController,
	 		link = Link,
	 		intf = Intf,
	 		build = True,
	 		xterms = False,
	 		cleanup = False,
	 		ipBase = '10.0.0.0/8',
	 		inNamespace = False,
	 		autoSetMacs = False,
	 		autoStaticArp = False,
	 		autoPinCpus = False,
	 		listenPort = None,
	 		waitConnected = False )

	Create Mininet object.

	Parameters
		topo            Topo (topology) object or None
		switch          default Switch class
		host            default Host class/constructor
		controller      default Controller class/constructor
		link            default Link class/constructor
		intf            default Intf class/constructor
		ipBase          base IP address for hosts,
		build           build now from topo?
		xterms          if build now, spawn xterms?
		cleanup         if build now, cleanup before creating?
		inNamespace     spawn switches and controller in net namespaces?
		autoSetMacs     set MAC addrs automatically like IP addresses?
		autoStaticArp   set all-pairs static MAC addrs?
		autoPinCpus     pin hosts to (real) cores (requires CPULimitedHost)?
		listenPort      base listening port to open; will be incremented for each additional switch in the net if inNamespace=False

*****

Mininet类的重要函数:

	def mininet.net.Mininet.addController( self,
	 		name = 'c0',
	 		controller = None,
	 		params )

	Add controller.
	Parameters
		controller   Controller class


	def mininet.net.Mininet.addHost( self,
	 		name,
	 		cls = None,
	 		params )

	Add host.
	Parameters
		name     name of host to add
		cls      custom host class/constructor (optional)
		params   parameters for host
	Returns
		added host


	def mininet.net.Mininet.addLink( self,
		 	node1,
		 	node2,
		 	port1 = None,
		 	port2 = None,
		 	cls = None,
		 	params )	

	Add a link from node1 to node2.
	Parameters
		node1	source node
		node2	dest node
		port1	source port
		port2	dest port
	Returns
		link object


	def mininet.net.Mininet.addSwitch( self,
		 	name,
		 	cls = None,
		 	params )		

	Add switch.
	Parameters
		name    name of switch to add
		cls     custom switch class/constructor (optional)
	Returns
		added switch side
	Parameters
		effect	increments listenPort ivar.


	def mininet.net.Mininet.iperf( self,
		 	hosts = None,
		 	l4Type = 'TCP',
		 	udpBw = '10M',
		 	format = None )

	Run iperf between two hosts.
	Parameters
		hosts	list of hosts; if None, uses opposite hosts
		l4Type	string, one of [ TCP, UDP ]
	Returns
		results two-element array of server and client speeds


	def mininet.net.Mininet.ping( self,
	 		hosts = None,
	 		timeout = None )

	Ping between all specified hosts.
	Parameters
		hosts	list of hosts
		timeout	time to wait for a response, as string
	Returns
		ploss packet loss percentage		

*****

node中的RemoteController类定义远程连接的控制器

	class RemoteController( Controller ):
		def __init__( self, ip='127.0.0.1', port=6633, **kwargs )

*****

node中的OVSKernelSwitch类定义OVS交换机

	class OVSLegacyKernelSwitch( Switch ):
		def __init__( self, name, dp=None, **kwargs )

	class OVSSwitch( Switch ):

	class Switch( Node ):
		portBase = 1
		dpidLen = 16
		def __init__( self, name, dpid=None, opts='', listenPort=None, **params ) 

*****

构建k-port的fat-tree网络

	"""Custom fat tree topology"""

	from mininet.cli import CLI
	from mininet.log import setLogLevel, info
	from mininet.net import Mininet
	from mininet.node import RemoteController, OVSKernelSwitch


	def myTopo ( key ):

	    net=Mininet(listenPort=6634,autoSetMacs=True,ipBase='10.0.0.0/8',autoStaticArp=False)

	    info( "****creating network****\n" )

	    info('****add controller****\n')

	    mycontroller=RemoteController("floodlight",ip="192.168.0.18",port=6633)

	    net.addController(mycontroller)

	    info('****add core switches****\n')

	    for i in range( 1, key*key/4+1 ):
	        switch_c = net.addSwitch( 's%d%d' %(key+1,i) )

	    info('****add aggregation switches****\n')

	    for i in range( 1, key+1 ):
	            for j in range( 1, key/2+1 ):
	                switch_au = net.addSwitch( 's%d%d' %(i,j) )
	            for j in range( key/2+1, key+1 ):
	                switch_ad = net.addSwitch( 's%d%d' %(i,j) )

	    info('****add hosts****\n')

	    for i in range( 1, key*key*key/4+1 ):
	            host = net.addHost( 'h%d' %i )

	    info('****add links between core switches and aggregation switches****\n')

	    for i in range( 1, key*key/4+1 ):
	        switch_c = net.get( 's%d%d' %(key+1,i) )
	        for j in range( 1, key+1 ):
	            swtich_au = net.get( 's%d%d' %(j,(i-1)/(key/2)+1) )
	            net.addLink( switch_c, swtich_au )

	    info('****Add links between two layer aggregation switches****\n')

	    for i in range( 1, key+1 ):
	        for j in range( 1, key/2+1 ):
	            switch_au = net.get( 's%d%d' %(i,j) )
	            for k in range( key/2+1, key+1 ):
	                switch_ad = net.get( 's%d%d' %(i,k) )
	                net.addLink( switch_au, switch_ad )

	    info('****Add links between aggregation switches and hosts****\n')

	    for i in range( 1, key+1 ):
	        for j in range( 1, key/2+1 ):
	            switch_ad = net.get( 's%d%d' %(i,j+key/2) )
	            for k in range( 1, key/2+1 ):
	                host = net.get( 'h%d' %((i-1)*key*key/4+(j-1)*(key/2)+k ) )
	                net.addLink( switch_ad, host )

	    info('****starting network****\n')

	    net.start()

	    CLI(net)

	    net.stop()


	if __name__ == '__main__':

	    setLogLevel( "info" )

	    OVSKernelSwitch.setup()

	    myTopo(4)