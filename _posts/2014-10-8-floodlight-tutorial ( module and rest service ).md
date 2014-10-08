---
layout: post
category : lessons
tagline: ""
tags : [tech]
---
{% include JB/setup %}

Floodlight控制器入门教程：自定义模块、提供ReST服务

*****

Floodlight控制器架构由两部分组成：核心模块(core module)，负责监听OpenFlow消息，并将其转换为一个个事件(event)；二级模块(secondary module)向核心模块注册，从而监听事件并进行处理。本文的目的就是向大家展示该如何实现一个自定义的二级模块，并让这个模块向外提供ReST服务接口。

首先，我们要在Floodlight控制器中实现一个自定义模块`PktInHistory`，该模块的作用是每当控制器收到一个`PACKET_IN`消息时，缓存相应的`<dpID, Message>`值。

具体方法如下：

* 在src/main/java下创建类`PktInHistory.java`，package为`net.floodlightcontroller.pktinhistory`，实现接口`IFloodlightModule`、`IOFMessageListener`。

		package net.floodlightcontroller.pktinhistory;

		import java.util.Collection;
		import java.util.Map;

		import org.openflow.protocol.OFMessage;
		import org.openflow.protocol.OFType;

		import net.floodlightcontroller.core.FloodlightContext;
		import net.floodlightcontroller.core.IOFMessageListener;
		import net.floodlightcontroller.core.IOFSwitch;
		import net.floodlightcontroller.core.module.FloodlightModuleContext;
		import net.floodlightcontroller.core.module.FloodlightModuleException;
		import net.floodlightcontroller.core.module.IFloodlightModule;
		import net.floodlightcontroller.core.module.IFloodlightService;

		public class PktInHistory implements IFloodlightModule, IOFMessageListener {

		    @Override
		    public String getName() {
		        // TODO Auto-generated method stub
		        return null;
		    }

		    @Override
		    public int getId() {
		        // TODO Auto-generated method stub
		        return 0;
		    }

		    @Override
		    public boolean isCallbackOrderingPrereq(OFType type, String name) {
		        // TODO Auto-generated method stub
		        return false;
		    }

		    @Override
		    public boolean isCallbackOrderingPostreq(OFType type, String name) {
		        // TODO Auto-generated method stub
		        return false;
		    }

		    @Override
		    public net.floodlightcontroller.core.IListener.Command receive(IOFSwitch sw, OFMessage msg, FloodlightContext cntx) {
		        // TODO Auto-generated method stub
		        return null;
		    }

		    @Override
		    public Collection<Class<? extends IFloodlightService>> getModuleServices() {
		        // TODO Auto-generated method stub
		        return null;
		    }

		    @Override
		    public Map<Class<? extends IFloodlightService>, IFloodlightService> getServiceImpls() {
		        // TODO Auto-generated method stub
		        return null;
		    }

		    @Override
		    public Collection<Class<? extends IFloodlightService>> getModuleDependencies() {
		        // TODO Auto-generated method stub
		        return null;
		    }

		    @Override
		    public void init(FloodlightModuleContext context) throws FloodlightModuleException {
		        // TODO Auto-generated method stub
		    }

		    @Override
		    public void startUp(FloodlightModuleContext context) {
		        // TODO Auto-generated method stub
		    }

		}

* 接着，我们不对`isCallbackOrderingPrereq`和`isCallbackOrderingPostreq`方法进行改动，让它们返回默认的false。因为我们暂时不关心PACKET_IN消息的处理顺序。

* 然后，我们定义两个引用：`floodlightProvider`和`buffer`

		protected IFloodlightProviderService floodlightProvider;
		protected ConcurrentCircularBuffer<SwitchMessagePair> buffer;

* 前者是核心Service类`IFloodlightProviderService`的引用，而后者是`ConcurrentCircularBuffer<SwitchMessagePair>`(定义环状缓冲区，用于缓存<dpID, Message>)的引用。ConcurrentCircularBuffer.java如下所示。

		package net.floodlightcontroller.pktinhistory;

		import java.util.concurrent.atomic.AtomicInteger;
		import java.lang.reflect.Array;

		public class ConcurrentCircularBuffer <T> {
		    private final AtomicInteger cursor = new AtomicInteger();
		    private final Object[]      buffer;
		    private final Class<T>      type;

		    public ConcurrentCircularBuffer (final Class <T> type, 
		                                     final int bufferSize) 
			{
			    if (bufferSize < 1) {
				throw new IllegalArgumentException(
		                "Buffer size must be a positive value"
								   );
			    }

			    this.type   = type;
			    this.buffer = new Object [ bufferSize ];
			}

		    public void add (T sample) {
		        buffer[ cursor.getAndIncrement() % buffer.length ] = sample;
		    }

		    @SuppressWarnings("unchecked")
			public T[] snapshot () {
		        Object[] snapshots = new Object [ buffer.length ];
		            
		        /* Identify the start-position of the buffer. */
		        int before = cursor.get();

		        /* Terminate early for an empty buffer. */
		        if (before == 0) {
		            return (T[]) Array.newInstance(type, 0);
		        }

		        System.arraycopy(buffer, 0, snapshots, 0, buffer.length);

		        int after          = cursor.get();
		        int size           = buffer.length - (after - before);
		        int snapshotCursor = before - 1;

		        /* The entire buffer was replaced during the copy. */
		        if (size <= 0) {
		            return (T[]) Array.newInstance(type, 0);
		        }

		        int start = snapshotCursor - (size - 1);
		        int end   = snapshotCursor;

		        if (snapshotCursor < snapshots.length) {
		            size   = snapshotCursor + 1;
		            start  = 0;
		        }

		        /* Copy the sample snapshot to a new array the size of our stable
		         * snapshot area.
		         */
		        T[] result = (T[]) Array.newInstance(type, size);

		        int startOfCopy = start % snapshots.length;
		        int endOfCopy   = end   % snapshots.length;

		        /* If the buffer space wraps the physical end of the array, use two
		         * copies to construct the new array.
		         */
		        if (startOfCopy > endOfCopy) {
		            System.arraycopy(snapshots, startOfCopy,
		                             result, 0, 
		                             snapshots.length - startOfCopy);
		            System.arraycopy(snapshots, 0,
		                             result, (snapshots.length - startOfCopy),
		                             endOfCopy + 1);
		        }
		        else {
		            /* Otherwise it's a single continuous segment, copy the whole thing
		             * into the result.
		             */
		            System.arraycopy(snapshots, startOfCopy, result, 0, size);
		        }

		        return (T[]) result;
		    }
		}

* 接着，我们在`init`方法中对`floodlightProvider`和`buffer`进行实例化。

		@Override
		public void init(FloodlightModuleContext context) throws FloodlightModuleException {
		    floodlightProvider = context.getServiceImpl(IFloodlightProviderService.class);
		    buffer = new ConcurrentCircularBuffer<SwitchMessagePair>(SwitchMessagePair.class, 100);
		}

* 然后，我们在`startUp`方法中为PACKET_IN消息注册监听器。

		@Override
		public void startUp(FloodlightModuleContext context) {
		    floodlightProvider.addOFMessageListener(OFType.PACKET_IN, this);
		}

* 我们需要在`getName`方法中说明该OpenFlow消息监听器的ID。

		@Override
		public String getName() {
		    return PktInHistory.class.getSimpleName();
		}

* 另外，我们需要在`getModuleDependencies`方法中说明该自定义模块的依赖。

		@Override
		public Collection<Class<? extends IFloodlightService>> getModuleDependencies() {
		    Collection<Class<? extends IFloodlightService>> l = new ArrayList<Class<? extends IFloodlightService>>();
		    l.add(IFloodlightProviderService.class);
		    return l;
		}

* 最关键的，我们在`receive`方法中定义了PACKET_IN消息的处理。注意，返回值`Command.CONTINUE`表示允许该消息继续被其他PACKET_IN监听者处理。

		@Override
		public Command receive(IOFSwitch sw, OFMessage msg, FloodlightContext cntx) {
		    switch(msg.getType()) {
		        case PACKET_IN:
		            buffer.add(new SwitchMessagePair(sw, msg));
		            break;
		        default:
		            break;
		    }
		    return Command.CONTINUE;
		}

* 现在，我们有net.floodlightcontroller.pktinhistory下的两个文件：`PktInHistory.java`和`ConcurrentCircularBuffer.java`，PktInHistory.java如下所示。

		package net.floodlightcontroller.pktinhistory;

		import java.util.ArrayList;
		import java.util.Collection;

		import org.openflow.protocol.OFMessage;
		import org.openflow.protocol.OFType;

		import net.floodlightcontroller.core.FloodlightContext;
		import net.floodlightcontroller.core.IFloodlightProviderService;
		import net.floodlightcontroller.core.IOFMessageListener;
		import net.floodlightcontroller.core.IOFSwitch;
		import net.floodlightcontroller.core.module.FloodlightModuleContext;
		import net.floodlightcontroller.core.module.FloodlightModuleException;
		import net.floodlightcontroller.core.module.IFloodlightModule;
		import net.floodlightcontroller.core.module.IFloodlightService;
		import net.floodlightcontroller.core.types.SwitchMessagePair;


		public class PktInHistory implements IFloodlightModule, IOFMessageListener {
			
			protected IFloodlightProviderService floodlightProvider;
			protected ConcurrentCircularBuffer<SwitchMessagePair> buffer;

			@Override
			public String getName() {
				return PktInHistory.class.getSimpleName();
			}

			@Override
			public boolean isCallbackOrderingPrereq(OFType type, String name) {
				// TODO Auto-generated method stub
				return false;
			}

			@Override
			public boolean isCallbackOrderingPostreq(OFType type, String name) {
				// TODO Auto-generated method stub
				return false;
			}

			@Override
			public net.floodlightcontroller.core.IListener.Command receive(IOFSwitch sw, OFMessage msg, FloodlightContext cntx) {
				switch(msg.getType()) {
					case PACKET_IN:
						buffer.add(new SwitchMessagePair(sw, msg));
						break;
					default:
						break;
		        }
				return Command.CONTINUE;	
			}

			@Override
			public Collection<Class<? extends IFloodlightService>> getModuleServices() {
				// TODO Auto-generated method stub
				return null;
			}

			@Override
			public Map<Class<? extends IFloodlightService>, IFloodlightService> getServiceImpls() {
				// TODO Auto-generated method stub
				return null;
			}

			@Override
			public Collection<Class<? extends IFloodlightService>> getModuleDependencies() {
				Collection<Class<? extends IFloodlightService>> l = new ArrayList<Class<? extends IFloodlightService>>();
			    l.add(IFloodlightProviderService.class);
			    return l;
			}

			@Override
			public void init(FloodlightModuleContext context) throws FloodlightModuleException {
				floodlightProvider = context.getServiceImpl(IFloodlightProviderService.class);
				buffer = new ConcurrentCircularBuffer<SwitchMessagePair>(SwitchMessagePair.class, 100);
			}

			@Override
			public void startUp(FloodlightModuleContext context) {
				floodlightProvider.addOFMessageListener(OFType.PACKET_IN, this);
			}

		}

* 至此，我们就取得了阶段性的成果，实现了一个自定义模块PktInHistory。

* 现在，我们需要将该模块的路径写到`src/main/resources/META-INF/services`下的`net.floodlightcontroller.core.module.IFloodlightModule`文件中，让控制器知晓该模块的存在。

		net.floodlightcontroller.pktinhistory.PktInHistory

* 同时，还要将其写入`src/main/resources`下的`floodlightdefault.properties`文件中，控制器启动时默认加载该文件中的模块。

*****

启动控制器，模块正常加载并运行。但伴随了一个问题，我们该如何查看缓存中的OF消息呢？

接下来，我们要在自定义模块`PktInHistory`中加入访问环状缓冲区的方法，然后通过接口将该方法向外暴露，最后将其包装为ReST服务。

具体方法如下：

* 在net.floodlightcontroller.pktinhistory.PktInHistory中，添加方法`getBuffer()`用于访问缓存。

		@Override
		public ConcurrentCircularBuffer<SwitchMessagePair> getBuffer() {
			return buffer;
		}

* 在net.floodlightcontroller.pktinhistory下创建接口类`IPktinHistoryService`(命名规范为IxxxService)，PktInHistory类通过实现该接口，将其getBuffer()方法向外暴露。

		package net.floodlightcontroller.pktinhistory;

		import net.floodlightcontroller.core.module.IFloodlightService;
		import net.floodlightcontroller.core.types.SwitchMessagePair;

		public interface IPktinHistoryService extends IFloodlightService {
		    public ConcurrentCircularBuffer<SwitchMessagePair> getBuffer();
		}

* 我们需要在PktInHistory类的`getModuleServices`方法中告诉模块系统，该模块提供`IPktInHistoryService`。

		@Override
		public Collection<Class<? extends IFloodlightService>> getModuleServices() {
			Collection<Class<? extends IFloodlightService>> l = new ArrayList<Class<? extends IFloodlightService>>();
		    l.add(IPktinHistoryService.class);
		    return l;
		}

* 我们需要在`getServiceImpls`方法中告诉模块系统，该模块是`IPktInHistoryService`的实现者。

		@Override
		public Map<Class<? extends IFloodlightService>, IFloodlightService> getServiceImpls() {
			Map<Class<? extends IFloodlightService>, IFloodlightService> m = new HashMap<Class<? extends IFloodlightService>, IFloodlightService>();
		    m.put(IPktinHistoryService.class, this);
		    return m;
		}

* 接着，我们需要在PktInHistory中，添加对`IRestApiService`的引用。

		protected IFloodlightProviderService floodlightProvider;
		protected ConcurrentCircularBuffer<SwitchMessagePair> buffer;
		protected IRestApiService restApi;

* 在`init`和`getModuleDependencies`方法中，与IFloodlightProviderService相同，我们需要对其引用进行实例化，并在依赖中添加它。

		@Override
		public void init(FloodlightModuleContext context) throws FloodlightModuleException {
			floodlightProvider = context.getServiceImpl(IFloodlightProviderService.class);
			restApi = context.getServiceImpl(IRestApiService.class);
			buffer = new ConcurrentCircularBuffer<SwitchMessagePair>(SwitchMessagePair.class, 100);
		}

		@Override
		public Collection<Class<? extends IFloodlightService>> getModuleDependencies() {
			Collection<Class<? extends IFloodlightService>> l = new ArrayList<Class<? extends IFloodlightService>>();
		    l.add(IFloodlightProviderService.class);
		    l.add(IRestApiService.class);
		    return l;
		}

* 然后，我们需要在net.floodlightcontroller.pktinhistory下创建类`PktInHistoryResource`和`PktInHistoryWebRoutable`。PktInHistoryResource负责处理ReST API请求(返回缓冲区快照)，PktInHistoryWebRoutable为PktInHistoryResource注册ReST API(将URL与resource绑定)。

		package net.floodlightcontroller.pktinhistory;

		import java.util.ArrayList;
		import java.util.List;

		import net.floodlightcontroller.core.types.SwitchMessagePair;

		import org.restlet.resource.Get;
		import org.restlet.resource.ServerResource;


		public class PktInHistoryResource extends ServerResource {
		    @Get("json")
		    public List<SwitchMessagePair> retrieve() {
		        IPktinHistoryService pihr = 
		        	(IPktinHistoryService)getContext().getAttributes()
		        		.get(IPktinHistoryService.class.getCanonicalName());
		        List<SwitchMessagePair> l = new ArrayList<SwitchMessagePair>();
		        l.addAll(java.util.Arrays.asList(pihr.getBuffer().snapshot()));
		        return l;
		    }
		}


		package net.floodlightcontroller.pktinhistory;

		import org.restlet.Context;
		import org.restlet.Restlet;
		import org.restlet.routing.Router;

		import net.floodlightcontroller.restserver.RestletRoutable;


		public class PktInHistoryWebRoutable implements RestletRoutable {
		    @Override
		    public Restlet getRestlet(Context context) {
		        Router router = new Router(context);
		        router.attach("/history/json", PktInHistoryResource.class);
		        return router;
		    }

		    @Override
		    public String basePath() {
		        return "/wm/pktinhistory";
		    }
		}

* 最后，我们需要在net.floodlightcontroller.pktinhistory.PktInHistory的`startUp`方法中添加该ReST API服务

		@Override
		public void startUp(FloodlightModuleContext context) {
			floodlightProvider.addOFMessageListener(OFType.PACKET_IN, this);
			restApi.addRestletRoutable(new PktInHistoryWebRoutable());
		}

* OK，大功告成。现在net.floodlightcontroller.pktinhistory下共有5个文件：PktInHistory.java，PktInHistoryResource.java，PktInHistoryWebRoutable.java，PktinHistoryService.java，ConcurrentCircularBuffer.java。

*****

运行net.floodlightcontroller.core.Main，启动控制器加载PktInHistory模块。

如下所示，使用Mininet运行一个简单的OpenFlow网络。

>
>![]({{ site.img_url }}/2014-10-8/1.JPG)
>

访问ReST API(python -mjson.tool用于将JSON的字符串美化输出)

	root@mininet-vm:~# curl -s http://192.168.0.19:8080/wm/pktinhistory/history/json | python -mjson.tool
	[
	    {
	        "message": {
	            "bufferId": 258,
	            "inPort": 1,
	            "length": 116,
	            "lengthU": 116,
	            "packetData": "WsiaVD2upilZ7a9UCABFAABUAJ1AAEABJgoKAAABCgAAAggAdqUUcwABcso0VM3EDQAICQoLDA0ODxAREhMUFRYXGBkaGxwdHh8gISIjJCUmJygpKissLS4vMDEyMzQ1Njc=",
	            "reason": "NO_MATCH",
	            "totalLength": 98,
	            "type": "PACKET_IN",
	            "version": 1,
	            "xid": 0
	        },
	        "switch": {
	            "actions": 4095,
	            "attributes": {
	                "DescriptionData": {
	                    "datapathDescription": "None",
	                    "hardwareDescription": "Open vSwitch",
	                    "length": 1056,
	                    "manufacturerDescription": "Nicira, Inc.",
	                    "serialNumber": "None",
	                    "softwareDescription": "2.0.1"
	                },
	                "FastWildcards": 4194303,
	                "supportsOfppFlood": true,
	                "supportsOfppTable": true
	            },
	            "buffers": 256,
	            "capabilities": 199,
	            "connectedSince": 1412745904102,
	            "dpid": "00:00:00:00:00:00:00:01",
	            "featuresReplyFromSwitch": {
	                "cancelled": false,
	                "done": false,
	                "transactionId": 62
	            },
	            "inetAddress": "/192.168.0.19:63240",
	            "ports": [
	                {
	                    "advertisedFeatures": 0,
	                    "config": 1,
	                    "currentFeatures": 0,
	                    "hardwareAddress": "de:06:57:b9:20:67",
	                    "name": "s1",
	                    "peerFeatures": 0,
	                    "portNumber": 65534,
	                    "state": 1,
	                    "supportedFeatures": 0
	                },
	                {
	                    "advertisedFeatures": 0,
	                    "config": 0,
	                    "currentFeatures": 192,
	                    "hardwareAddress": "00:00:00:00:00:01",
	                    "name": "s1-eth2",
	                    "peerFeatures": 0,
	                    "portNumber": 2,
	                    "state": 0,
	                    "supportedFeatures": 0
	                },
	                {
	                    "advertisedFeatures": 0,
	                    "config": 0,
	                    "currentFeatures": 192,
	                    "hardwareAddress": "00:00:00:00:00:01",
	                    "name": "s1-eth1",
	                    "peerFeatures": 0,
	                    "portNumber": 1,
	                    "state": 0,
	                    "supportedFeatures": 0
	                }
	            ],
	            "role": null,
	            "tables": -2
	        }
	    }
	]

由于在net.floodlightcontroller.pktinhistory.PktInHistory的`Command`方法中，调用了buffer.add(new SwitchMessagePair(sw, msg))，这里的sw是对IOFSwitch(实际上是对`OFSwitchImpl`)的引用，因此查询缓存时会把交换机的信息全部返回。

如果我们不想显示太多交换机信息，比如我们只关心交换机的dpID，那么我们可以在net.floodlightcontroller.core.web.serializers下创建类`OFSwitchImplJSONSerializer`。

	package net.floodlightcontroller.core.web.serializers;

	import java.io.IOException;

	import net.floodlightcontroller.core.internal.OFSwitchImpl;

	import org.codehaus.jackson.JsonGenerator;
	import org.codehaus.jackson.JsonProcessingException;
	import org.codehaus.jackson.map.JsonSerializer;
	import org.codehaus.jackson.map.SerializerProvider;
	import org.openflow.util.HexString;

	public class OFSwitchImplJSONSerializer extends JsonSerializer<OFSwitchImpl> {

	    /**
	     * Handles serialization for OFSwitchImpl
	     */
	    @Override
	    public void serialize(OFSwitchImpl switchImpl, JsonGenerator jGen,
	                          SerializerProvider arg2) throws IOException,
	                                                  JsonProcessingException {
	        jGen.writeStartObject();
	        jGen.writeStringField("dpid", HexString.toHexString(switchImpl.getId()));
	        jGen.writeEndObject();
	    }

	    /**
	     * Tells SimpleModule that we are the serializer for OFSwitchImpl
	     */
	    @Override
	    public Class<OFSwitchImpl> handledType() {
	        return OFSwitchImpl.class;
	    }

	}

然后，在`net.floodlightcontroller.core.internal.OFSwitchImpl`中作如下所示的修改。

	import net.floodlightcontroller.core.web.serializers.OFSwitchImplJSONSerializer;

	@JsonSerialize(using=OFSwitchImplJSONSerializer.class)
	public class OFSwitchImpl implements IOFSwitch {

重新访问ReST API，交换机信息这次只显示了dpID。

	root@mininet-vm:~# curl -s http://192.168.0.19:8080/wm/pktinhistory/history/json | python -mjson.tool
	[
	    {
	        "message": {
	            "bufferId": 258,
	            "inPort": 1,
	            "length": 116,
	            "lengthU": 116,
	            "packetData": "WsiaVD2upilZ7a9UCABFAABUAKBAAEABJgcKAAABCgAAAggAdg0UlAAB1co0VHE7BwAICQoLDA0ODxAREhMUFRYXGBkaGxwdHh8gISIjJCUmJygpKissLS4vMDEyMzQ1Njc=",
	            "reason": "NO_MATCH",
	            "totalLength": 98,
	            "type": "PACKET_IN",
	            "version": 1,
	            "xid": 0
	        },
	        "switch": {
	            "dpid": "00:00:00:00:00:00:00:01"
	        }
	    }
	]

*****

* 现在，让我们停下来重新看一看Floodlight控制器启动时的输出信息。首先是加载模块。

>
 [main] INFO  n.f.c.module.FloodlightModuleLoader - Loading default modules
>
 [main] DEBUG n.f.c.module.FloodlightModuleLoader - Starting module loader
>

* 接着，由于我们将该模块的路径写到src/main/resources/META-INF/services下的net.floodlightcontroller.core.module.IFloodlightModule文件中，控制器找到了该模块。

>
 [main] DEBUG n.f.c.module.FloodlightModuleLoader - Found module net.floodlightcontroller.pktinhistory.PktInHistory
>

* 设定net.floodlightcontroller.core.internal.Controller为net.floodlightcontroller.core.IFloodlightProviderService的具体实现，设定net.floodlightcontroller.pktinhistory.PktInHistory为net.floodlightcontroller.pktinhistory.IPktinHistoryService的具体实现。

* 还记得在`init`方法中，对floodlightProvider引用进行实例化时，调用了`floodlightProvider = context.getServiceImpl(IFloodlightProviderService.class)`吗，实际上真正完成实例化工作的是`net.floodlightcontroller.core.internal.Controller`类，而`IFloodlightProviderService`是该类向外暴露的服务。同样地，`IPktinHistoryService`是`PktInHistory`类向外暴露的服务，在`net.floodlightcontroller.pktinhistory.PktInHistoryResources`中的`retrieve`方法，通过调用`IPktinHistoryService pihr = (IPktinHistoryService)getContext().getAttributes().get(IPktinHistoryService.class.getCanonicalName())`实现类中的`getBuffer()`方法。

>
 [main] DEBUG n.f.c.module.FloodlightModuleLoader - Setting net.floodlightcontroller.core.internal.Controller@3cde8a82  as provider for net.floodlightcontroller.core.IFloodlightProviderService
>
 [main] DEBUG n.f.c.module.FloodlightModuleLoader - Setting net.floodlightcontroller.pktinhistory.PktInHistory@64650ddb  as provider for net.floodlightcontroller.pktinhistory.IPktinHistoryService
>

* 该模块调用其init方法进行初始化。

>
 [main] DEBUG n.f.c.module.FloodlightModuleLoader - Initializing net.floodlightcontroller.pktinhistory.PktInHistory
>

* 该模块调用其startUp方法进行启动。

>
[main] DEBUG n.f.c.module.FloodlightModuleLoader - Starting net.floodlightcontroller.pktinhistory.PktInHistory
>

* 开启在net.floodlightcontroller.pktinhistory.PktInHistoryWebRoutable中定义的ReST服务接口。

>
 [main] DEBUG n.f.restserver.RestApiServer - REST API routables: PktInHistoryWebRoutable (/wm/pktinhistory)
>

* PktInHistory向核心模块注册了OF消息监听器。

>
 [main] DEBUG n.f.core.internal.Controller - OFListeners for PACKET_IN: PktInHistory
>

* 根据配置文件进行相应的启动。

>
 [main] DEBUG n.f.core.internal.Controller - OpenFlow port set to 6633
>
 [main] DEBUG n.f.core.internal.Controller - Number of worker threads set to 0
>
 [main] DEBUG n.f.core.internal.Controller - ControllerId set to localhost
>
 [main] INFO  n.f.core.internal.Controller - Controller role set to null
>
 [main] DEBUG n.f.forwarding.Forwarding - FlowMod idle timeout set to 5 seconds
>
 [main] DEBUG n.f.forwarding.Forwarding - FlowMod hard timeout set to 0 seconds
>
 [main] DEBUG n.f.restserver.RestApiServer - REST port set to 8080
>
 [main] DEBUG n.f.flowcache.FlowReconcileManager - FlowReconcile is true
>
 [main] INFO  o.r.Component.InternalRouter.Server - Starting the Simple [HTTP/1.1] server on port 8080
>
 [main] INFO  n.f.core.internal.Controller - Listening for switch connections on 0.0.0.0/0.0.0.0:6633
>
