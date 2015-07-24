---
layout: post
category : lessons
tagline: ""
tags : [tech]
---
{% include JB/setup %}

java中使用args4j解析命令行参数

*****

如果你经常使用Eclipse，那么你可能会想在运行应用时加入一些参数。例如，Run -> Run Configurations ... -> Arguments

args4j是一个小型的Java类库，它使用Option的Annotation来保存输入的参数。该Annotation有5个Filed（String name，Class<? extends OptionHandler> handler，String metaVar，boolean required，String usage），其中name是必须的，其他四个是可选的，另外，Option的执行逻辑根据输入参数类型的不同而变化。

Boolean Switch：

	在命令行中输入 -st 代表该boolean型输入参数被置为true，required指定该参数必须给定，
	由于未指定metaVar，Help String默认为-st BOOLEAN “set this value true”

	@Option(name = "-st", usage = "set this value true", required = true)
	private boolean value;

String Switch：

	在命令行中输入 -st “set new String” 代表该String型输入参数被置为“set new String”，required指定该参数不必给定
	
	@Option(name = "-st", usage = "set this String", required = false)
	private String value = "default string";

Enum Switch：

	在命令行中输入 -st Kate 代表该enum型输入参数被置为Kate，
	另外由于enum型的特性，所以输入参数是Lucy或Jimmy都可以，且大小写不敏感，但是“”和Bluesy不可以

	enum Value { kim, Kate, Lucy, Jimmy }
	@Option(name = "-st", usage = "set this Enum", required = false)
	private Value value = Kim;

File Switch：

	在命令行中输入 -st “/etc/network/interface” 代表该File型输入参数被置为“/etc/network/interface”路径下的文件

	@Option(name = "-st", usage = "set this File Path", required = false)
	private File value = new File(.);

下面使用一个较完整的示例说明args4j参数解析的使用方法：

	利用Option的Annotation来定义一个String型命令行参数，name指定该参数的名称，aliases指定该参数的别名，
	usage解释该参数的作用，metaVar解释该参数的类型，required指定该参数是否为必须给定的
	Help String的显示为-cf FILE “WANACC configuration file”

	package net.wanacc.core.internal;

	import org.kohsuke.args4j.Option;
	import org.kohsuke.args4j.CmdLineException;
	import org.kohsuke.args4j.CmdLineParser;

	/**
	 * Expresses the settings of WANACC.
	 */
	public class CmdLineSettings {
	    public static final String DEFAULT_CONFIG_FILE = "src/main/resources/wanaccdefault.properties";

	    @Option(name = "-cf", aliases = "--configFile", metaVar = "FILE", 
	    		usage = "WANACC configuration file", required = false)
	    private String configFile = DEFAULT_CONFIG_FILE;
	    
	    public String getModuleFile() {
	    	return configFile;
	    }

	    public static void main(String[] args) {
		CmdLineSettings settings = new CmdLineSettings();
			CmdLineParser parser = new CmdLineParser(settings);
			try {
				parser.parseArgument(args);
			} catch (CmdLineException e) {
				parser.printUsage(System.out);
				System.exit(1);
			}
		}
	}


`小结：
args4j取得命令行参数，同保存参数的Option属性类型进行比较，并转换为适当的值，免去编程者判断args的麻烦。
当默认的Handler无法满足解析的需求时，可通过扩展Handler的方法实现。`