---
layout: post
category : lessons
tagline: ""
tags : [tech]
---
{% include JB/setup %}

Floodlight控制器入门教程：模块加载过程

*****

我们知道，作为目前主流控制器之一的Floodlight控制器自身存在许多亮点。其中一点就是为我们提供了一个`模块加载系统`(module loading system)，以便于对控制器功能进行拓展。

下面就让我们一起来看一看这个模块加载系统是如何工作的。

控制器的主程序入口在net.floodlightcontroller.core.Main。OK，现在打开这个Main.java文件，首先映入眼帘的是下面这段代码，它的作用是帮助我们解析控制器的启动命令。

    CmdLineSettings settings = new CmdLineSettings();
    CmdLineParser parser = new CmdLineParser(settings);
    try {
        parser.parseArgument(args);
    } catch (CmdLineException e) {
        parser.printUsage(System.out);
        System.exit(1);
    }

当我们用如下所示的命令行方式启动控制器时，上面的代码就会帮助我们解析这些命令。`-cp`可以指定多个.jar文件，`-cf`指定其他.properties文件(默认的.properties文件为src/main/resources/floodlightdefault.properties)，`-D`可以override默认的.properties文件中的相应内容。

	java -cp floodlight.jar:/path/to/other.jar net.floodlightcontroller.core.Main -cf path/to/config/file.properties
	java -Dnet.floodlightcontroller.restserver.RestApiServer.port=8080 -jar floodlight.jar

接着，就要用到重要的`FloodlightModuleLoader`类进行模块加载。

    FloodlightModuleLoader fml = new FloodlightModuleLoader();
    IFloodlightModuleContext moduleContext = fml.loadModulesFromConfig(settings.getModuleFile());

那么，就让我们来看看FloodlightModuleLoader中定义的成员变量和方法。首先，FloodlightModuleLoader中定义了3个Map成员变量：`serviceMap`(映射一个IFloodlightService到它的service provider collection)、`moduleServiceMap`(映射一个IFloodlightModule到它的provided service collection)、`moduleNameMap`(映射一个IFloodlightModule的名称到它自身)。

然后，`loadModulesFromConfig(String fName)`方法实现从-cp指定的fName中，或从默认的src/main/resources/floodlightdefault.properties中读取要加载模块，并将这些信息交由loadModulesFromList(Collection<String> configMods, Properties prop, Collection<IFloodlightService> ignoreList)方法处理。

    public IFloodlightModuleContext loadModulesFromConfig(String fName) throws FloodlightModuleException {
        return loadModulesFromList(configMods, prop);

    public IFloodlightModuleContext loadModulesFromList(Collection<String> configMods, Properties prop) throws FloodlightModuleException {
        return loadModulesFromList(configMods, prop, null);

    protected IFloodlightModuleContext loadModulesFromList(Collection<String> configMods, Properties prop, 
            Collection<IFloodlightService> ignoreList) throws FloodlightModuleException {
        findAllModules(configMods);

来到`loadModulesFromConfig`方法内，首先，调用`findAllModules(configMods)`方法，从src/main/resources/META-INF/services/net.floodlightcontroller.core.module.IFloodlightModule中读取所有模块，并生成3个Map。然后，结合configMods中的要加载模块信息确保没有为某个IFloodlightService指定多个service provider module。需要注意的是，若我们在floodlightdefault.properties中要加载提供相同IFloodlightService的两个不同模块，在这里会error。

	protected static void findAllModules(Collection<String> mList) throws FloodlightModuleException {
        // Make sure they haven't specified duplicate modules in the config
        int dupInConf = 0;
        for (IFlMod : mods) {
            if (mList.contains(cMod.getClass().getCanonicalName()))
                dupInConf += 1;
        }
        
        if (dupInConf > 1) {
            String duplicateMods = "";
            for (IFloodlightModule mod : mods) {
                duplicateMods += mod.getClass().getCanonicalName() + ", ";
            }
            throw new FloodlightModuleException("ERROR! The configuraiton" +
                    " file specifies more than one module that provides the service " +
                    s.getCanonicalName() +". Please specify only ONE of the " +
                    "following modules in the config file: " + duplicateMods);
        }		

接着，回到`loadModulesFromConfig`方法，将要加载模块的信息全部入队，然后依次出队。

	moduleQ.addAll(configMods);
	Set<String> modsVisited = new HashSet<String>();
        
    while (!moduleQ.isEmpty()) {
        String moduleName = moduleQ.remove();
        if (modsVisited.contains(moduleName))
            continue;
        modsVisited.add(moduleName);
        IFloodlightModule module = moduleNameMap.get(moduleName);

然后，查看出队模块的依赖提供模块是否已加载或已入队，若已加载或已入队，ok；若未入队且该提供模块仅存在一个，将其入队；若未入队且该提供模块不存在，error。

	// Add the module to be loaded
    addModule(moduleMap, moduleSet, module);
    // Add it's dep's to the queue
    Collection<Class<? extends IFloodlightService>> deps = module.getModuleDependencies();
    if (deps != null) {
        for (Class<? extends IFloodlightService> c : deps) {
            IFloodlightModule m = moduleMap.get(c);
            if (m == null) {
	            Collection<IFloodlightModule> mods = serviceMap.get(c);
	            // Make sure only one module is loaded
	            if ((mods == null) || (mods.size() == 0)) {
	            	throw new FloodlightModuleException("ERROR! Could not " + "find an IFloodlightModule that provides service " + c.toString());
	            } else if (mods.size() == 1) {
	                IFloodlightModule mod = mods.iterator().next();
	                if (!modsVisited.contains(mod.getClass().getCanonicalName()))
	                    moduleQ.add(mod.getClass().getCanonicalName());
	            } else {
	                boolean found = false;
	                for (IFloodlightModule moduleDep : mods) {
	                    if (configMods.contains(moduleDep.getClass().getCanonicalName())) {
	                        // Module will be loaded, we can continue
	                        found = true;
	                        break;
	                    }
	                }
	                if (!found) {
	                    String duplicateMods = "";
	                    for (IFloodlightModule mod : mods) {
	                        duplicateMods += mod.getClass().getCanonicalName() + ", ";
	                    }
	                    throw new FloodlightModuleException("ERROR! Found more " + 
	                        "than one (" + mods.size() + ") IFloodlightModules that provides " +
	                        "service " + c.toString() + 
	                        ". This service is required for " + moduleName + 
	                        ". Please specify one of the following modules in the config: " + 
	                        duplicateMods);
                    }
                }
            }
        }
    }

至此，我们得到了所有要加载模块的集合`moduleSet`(在这个过程中我们得到的副产品是`floodlightModuleContext`)，接着，对这个集合中的模块调用其自身的`init`方法和`startUp`方法。

	initModules(moduleSet);
	startupModules(moduleSet);

由此也可以看出所有要加载的二级模块的init方法和startUp方法的调用顺序是`不确定`的。

*****

后续：让我们回到Main.java，启动`ReST` Server。

    IRestApiService restApi = moduleContext.getServiceImpl(IRestApiService.class);
    restApi.run();

然后，启动控制器的主模块net.floodlightcontroller.core.internal.Controller

    IFloodlightProviderService controller = moduleContext.getServiceImpl(IFloodlightProviderService.class);
    // This call blocks, it has to be the last line in the main
    controller.run();