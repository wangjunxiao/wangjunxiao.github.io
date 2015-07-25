---
layout: post
category : lessons
tagline: ""
tags : [tech]
---
{% include JB/setup %}

java中使用annotation注解配置log信息

*****

annotation注解，又叫元数据，是JDK5中引入的一种以通用格式为程序提供配置信息的方式。

首先介绍一下JDK中内置的三种常用的annotation注解，这三个内置的注解在java.lang包下：

	@Override：
	这个注解常用在继承类或实现接口的子类方法上，表面该方法是子类覆盖父类的方法，该方法的方法签名要遵循覆盖方法的原则：即访问控制权限必能比父类更严格，不能比父类抛出更多的异常。

	@Deprecated：
	这个注解告诉编译器该元素是过时的，即在目前的JDK版本中已经有新的元素代替该元素。

	@SuppressWarnings：
	该注解关闭编译器中不合适的警告，即强行压制编译器的警告提示。

annotation注解的定义和接口类似，形式如下：

	import java.lang.annotation.*;

	@Target(ElementType.*)
	@Retention(RetentionPolicy.*)
	public @interface Test {
		Object value() default null;
	}

在声明注解的时候往往需要使用@Target，@Retention等注解，这种注解被称为注解的注解(元数据注解)，即是专门用于处理annotation注解本身的。

@Target注解——用于指示注解所应用的目标程序元素种类，ElementType枚举提供了java程序中声明的元素类型如下：

	ANNOTATION_TYPE：注释类型声明。
	CONSTRUCTOR：构造方法声明。
	FIELD：字段声明(包括枚举常量)。
	LOCAL_VARIABLE：局部变量声明。
	METHOD：方法声明。
	PACKAGE：包声明。
	PARAMETER：参数声明。
	TYPE：类，接口或枚举声明。

@Retention注解——该注解用于指示所定义的注解类型的注释在程序声明周期中得保留范围，RetentionPolicy枚举常量定义了注解在代码中的保留策略：

	CLASS：编译器把注解记录在类文件中，但在运行时JVM不需要保留注解。
	RUNTIME：编译器把注解记录在类文件中，在运行时JVM将保留注解，因此可以通过反射机制读取注解。
	SOURCE：仅保留在源码中，编译器在编译时就要丢弃掉该注解。

需要注意的是，annotation注解中的元素只能是下面的数据类型：
(1) 基本类型，如int, boolean等等，如果可以自动装箱和拆箱，则可以使用对应的对象包装类型。
(2) String类型。
(3) Class类型。
(4) enum类型。
(5) annotation类型。
(6) 上面类型的数组。
除了这些类型以外，如果在注解中定义其他类型的数据，编译器将会报错。

*****

下面使用一个较完整的示例说明annotation注解的使用方法：

	@Target({ElementType.TYPE, ElementType.METHOD})
	public @interface LogMessageDocs {
	    /**
	     * A list of {@link LogMessageDoc} elements
	     * @return the list of log message doc
	     */
	    LogMessageDoc[] value();
	}

	@Target({ElementType.TYPE, ElementType.METHOD})
	public @interface LogMessageDoc {
		public static final String UNKNOWN_ERROR = "An unknown error occured";
		/**
	     * The log level for the log message
	     * @return the log level as a String
	     */
	    String level() default "INFO";
	    /**
	     * The message that will be printed
	     * @return the message
	     */
	    String message() default UNKNOWN_ERROR;	
	}

	public class WANACCModuleLoader {

		@LogMessageDocs({
			@LogMessageDoc(level="INFO",
				message="Loading modules from file {file name}"),
			@LogMessageDoc(level="INFO",
				message="Loading default modules"),
			@LogMessageDoc(level="ERROR",
				message="Could not load module configuration file"),
			@LogMessageDoc(level="ERROR",
				message="Could not load default modules")
		})
		public IWANACCModuleContext loadModulesFromConfig(String fName)
				throws WANACCModuleException {
			Properties prop = new Properties();
			File f = new File(fName);
			if (f.isFile()) {
				logger.info("Loading modules from file {}", fName);
				try {
					prop.load(new FileInputStream(fName));
					} catch (Exception e) {
						logger.error("Could not load module configuration file", e);
						System.exit(1);
					}
			} else {
				logger.info("Loading default modules");
				InputStream is = this.getClass().getClassLoader().
				getResourceAsStream(COMPILED_CONF_FILE);
				try {
					prop.load(is);
				} catch (IOException e) {
					logger.error("Could not load default modules", e);
					System.exit(1);
	            }
	        }
	    }
	}


`小结：
annotation注解可以使元数据写在程序源码中，使得代码看起来简洁，同时编译器也提供了对annotation注解的类型检查，使得在编译期间就可以排除语法错误。`