---
title: dubbo源码学习之服务注册与发现
date: 2019-11-18 21:40:54
categories: Java
tags:
- dubbo
---

本文记录源码学习dubbo服务注册与发现实现原理笔记。

<!--more-->

## dubbo服务注册
dubbo服务提供者的核心入口为ServiceBean。ServiceBean继承了ServiceConfig类，实现了Initializingbean、DisposableBean、ApplicationContextAware、ApplicationListener、BeanNameAware、ApplicationEventPublisherAware等接口。

### 源码分析afterPropertiesSet()方法
```java
// ①
if (getProvider() == null) {
    Map<String, ProviderConfig> providerConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ProviderConfig.class, false, false);
    // ...
}
// ②
if (getApplication() == null && (getProvider() == null || getProvider().getApplication() == null)) {
    Map<String, ApplicationConfig> applicationConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ApplicationConfig.class, false, false);
    // ...
}
// ③
if (getModule() == null && (getProvider() == null || getProvider().getModule() == null)) {
    Map<String, ModuleConfig> moduleConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ModuleConfig.class, false, false);
    // ...
}
// ④
if ((CollectionUtils.isEmpty(getRegistries())) && (getProvider() == null || CollectionUtils.isEmpty(getProvider().getRegistries())) && (getApplication() == null || CollectionUtils.isEmpty(getApplication().getRegistries()))) {
    Map<String, RegistryConfig> registryConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, RegistryConfig.class, false, false);
    // ...
}
// ⑤
if (getMetadataReportConfig() == null) {
    // ...
}
// ⑥
if (getConfigCenter() == null) {
    // ...
}
// ⑦
if (getMonitor() == null&& (getProvider() == null || getProvider().getMonitor() == null) && (getApplication() == null || getApplication().getMonitor() == null)) {
    // ...
}
// ⑧
if (CollectionUtils.isEmpty(getProtocols()) && (getProvider() == null || CollectionUtils.isEmpty(getProvider().getProtocols()))) {
    // ...
}
// ⑨
if (!supportedApplicationListener) {
    export();
}
```  
①中如果<dubbo:service>标签没有配置provider属性，则从applicationContext查找一个<dubbo:provider>配置，获取该实例，如果存在多个，则会抛出异常，提示"Duplicate provider configs xxx"。
②中如果<dubbo:service>标签没有配置application属性，则从applicationContext中查找一个<dubbo:application>配置，获取该实例，如果存在多个，则会抛出异常，提示"Duplicate application configs xxx"。
③中如果<dubbo:service>标签没有配置module属性，则从ApplicationContext查找一个<dubbo:module>配置，获取该实例，如果存在多个，则会抛出异常，提示"Duplicate module configs xxx"。
④中如果<dubbo:service>标签中没有配置registry属性，则从ApplicationContext查找<dubbo:registry>配置，获取该实例，可以配置多个registry。
⑦中如果<dubbo:service>标签中没有配置monitor属性，则从ApplicationContext查找<dubbo:monitor>配置，获取该实例，存在多个，则会抛出异常， 提示"Duplicate monitor configs: xxx"。
⑧中如果<dubbo:service>标签没有配置Protocol属性，则从ApplicationContext查找<dobbo:protocol>配置，获取该实例，可以配置多个。
⑨执行服务暴露。

### 源码分析expor()方法
在serviceBean经过上述的一下配置加载之后，接下来就是执行服务暴露的过程。
```java
public void export() {
    super.export();
    // Publish ServiceBeanExportedEvent
    publishExportEvent();
}
```
首先调用父类(ServiceConfig)的export()方法，来看看父类的export方法都做了什么操作。
```java
public synchronized void export() {
    checkAndUpdateSubConfigs(); // ①

    if (!shouldExport()) { // ②
        return;
    }

    if (shouldDelay()) { // ③
        DELAY_EXPORT_EXECUTOR.schedule(this::doExport, getDelay(), TimeUnit.MILLISECONDS);
    } else {
        doExport(); //④
    }
}
```
首先①中进行配置检测和更新。配置检测和更新而分为以下几个步骤:
- 调用completeCompoundConfigs()方法，完成在全局配置上显式定义的默认配置。
- 检测provider是否为空，如果为空，则从ConfigManager获取默认的provider。
- 检测Protocol，如果protocols为空，则将protocolId转化成protocol。
- 检测Application，如果Application为空，从ConfigManager获取默认的Application，如果application配置非法，则抛出异常
- 如果服务不是暴露在本地，则检测registry，如果registry配置非法，也抛出异常。
- 校验ref与interface属性。如果ref是GenericService，则为dubbo的泛化实现，然后验证interface接口与ref引用的类型是否一致。
- ...
②检测配置的export属性是否为true，如果为false直接返回，不暴露服务。
③判断是否应该延迟暴露，获取delay配置的值，delay的值不为空并且大于0则表示需要延迟暴露，延迟的时间为delay的具体值，单位毫秒。然后会启动一个定时任务去执行服务暴露工作。
④执行服务暴露

### 源码分析doExport()方法
```java
protected synchronized void doExport() {
    if (unexported) {
        throw new IllegalStateException("The service " + interfaceClass.getName() + " has already unexported!");
    }
    if (exported) {
        return;
    }
    exported = true;

    if (StringUtils.isEmpty(path)) {
        path = interfaceName;
    }
    doExportUrls();
}
```
该方法加锁了，针对一个服务，只会暴露一次，如果exported为true，就不会进行暴露。如果path(服务名称)为空，则取接口名。接着调用doExportUrls()方法

### 源码分析doExportUrls()方法
```java
private void doExportUrls() {
    List<URL> registryURLs = loadRegistries(true); // ①
    for (ProtocolConfig protocolConfig : protocols) {
        String pathKey = URL.buildKey(getContextPath(protocolConfig).map(p -> p + "/" + path).orElse(path), group, version); // ②
        ProviderModel providerModel = new ProviderModel(pathKey, ref, interfaceClass);
        ApplicationModel.initProviderModel(pathKey, providerModel);
        doExportUrlsFor1Protocol(protocolConfig, registryURLs); // ③
    }
}
```
①首先加载注册地址，从registries列表中解析；②构建服务路径，路径信息包含protocol、group、verison、path(服务名称)等信息；③调用doExportUrlsFor1Protocol方法，按照指定的协向各个注册地址暴露服务。

### 源码分析doExportUrlsFor1Protocol方法

```java
// ①
String name = protocolConfig.getName();
if (StringUtils.isEmpty(name)) {
name = DUBBO;
}
// ②
Map<String, String> map = new HashMap<String, String>();
map.put(SIDE_KEY, PROVIDER_SIDE); // ③ 
appendRuntimeParameters(map);
appendParameters(map, metrics);
appendParameters(map, application);
appendParameters(map, module);
appendParameters(map, provider);
appendParameters(map, protocolConfig);
appendParameters(map, this);
```
①如果protocol为空，默认为dubbo协议；②map容器存放一些配置信息；③表示为provider
```java
if (CollectionUtils.isNotEmpty(methods)) {
    for (MethodConfig method : methods) {
        appendParameters(map, method, method.getName()); // ①
        String retryKey = method.getName() + ".retry"; // ②
        if (map.containsKey(retryKey)) {
            String retryValue = map.remove(retryKey);
            if ("false".equals(retryValue)) {
                map.put(method.getName() + ".retries", "0");
            }
        }
        List<ArgumentConfig> arguments = method.getArguments(); // ③
        if (CollectionUtils.isNotEmpty(arguments)) {
            for (ArgumentConfig argument : arguments) {
                // convert argument type
                if (argument.getType() != null && argument.getType().length() > 0) {
                    Method[] methods = interfaceClass.getMethods();
                    // visit all methods
                    if (methods != null && methods.length > 0) {
                        for (int i = 0; i < methods.length; i++) {
                            String methodName = methods[i].getName();
                            // target the method, and get its signature
                            if (methodName.equals(method.getName())) {
                                Class<?>[] argtypes = methods[i].getParameterTypes();
                                // one callback in the method
                                if (argument.getIndex() != -1) {
                                    if (argtypes[argument.getIndex()].getName().equals(argument.getType())) {
                                        appendParameters(map, argument, method.getName() + "." + argument.getIndex());
                                    } else {
                                        // throw exception
                                    }
                                } else {
                                    // multiple callbacks in the method
                                    for (int j = 0; j < argtypes.length; j++) {
                                        Class<?> argclazz = argtypes[j];
                                        if (argclazz.getName().equals(argument.getType())) {
                                            appendParameters(map, argument, method.getName() + "." + j);
                                            if (argument.getIndex() != -1 && argument.getIndex() != j) {
                                                // throw exception
                                            }
                                        }
                                    }
                                }
                            }
                        }
                    }
                } else if (argument.getIndex() != -1) {
                    appendParameters(map, argument, method.getName() + "." + argument.getIndex());
                } else {
                    // handle exception
                }

            }
        }
    } // end of methods for
}
```
解析服务的方法列表。①将服务方法信息放入map；②方法级别的retry配置，如果配置的是false，则retry为0；再存入map；③解析方法参数信息，存入map。

```java
if (ProtocolUtils.isGeneric(generic)) { 
    map.put(GENERIC_KEY, generic);
    map.put(METHODS_KEY, ANY_VALUE);
} else {
    String revision = Version.getVersion(interfaceClass, version);
    if (revision != null && revision.length() > 0) {
        map.put(REVISION_KEY, revision);
    }

    String[] methods = Wrapper.getWrapper(interfaceClass).getMethodNames();
    if (methods.length == 0) {
        logger.warn("No method found in service interface " + interfaceClass.getName());
        map.put(METHODS_KEY, ANY_VALUE);
    } else {
        map.put(METHODS_KEY, StringUtils.join(new HashSet<String>(Arrays.asList(methods)), ","));
    }
}
```
如果是泛化实现，则填充generic=true，和methods="*"的值到map中。如果不是泛化实现，填充method版本信息。
```java
if (!ConfigUtils.isEmpty(token)) {
    if (ConfigUtils.isDefault(token)) {
        map.put(TOKEN_KEY, UUID.randomUUID().toString());
    } else {
        map.put(TOKEN_KEY, token);
    }
}
```
如果启用了令牌机制，将令牌信息放入map。如果令牌配置为default，通过uuid生成。
```java
String host = this.findConfigedHosts(protocolConfig, registryURLs, map);
Integer port = this.findConfigedPorts(protocolConfig, name, map);
URL url = new URL(name, host, port, getContextPath(protocolConfig).map(p -> p + "/" + path).orElse(path), map);
```
解析服务提供者的host和ip，然后构建服务暴露的URl。

```java
if (!SCOPE_NONE.equalsIgnoreCase(scope)) { // ①
    // ②
    if (!SCOPE_REMOTE.equalsIgnoreCase(scope)) {
        exportLocal(url);
    }
    // ③
    if (!SCOPE_LOCAL.equalsIgnoreCase(scope)) {
        if (!isOnlyInJvm() && logger.isInfoEnabled()) {
            logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url);
        }
        if (CollectionUtils.isNotEmpty(registryURLs)) { // ④
            for (URL registryURL : registryURLs) {
                //if protocol is only injvm ,not register
                if (LOCAL_PROTOCOL.equalsIgnoreCase(url.getProtocol())) { // ⑤
                    continue;
                }
                url = url.addParameterIfAbsent(DYNAMIC_KEY, registryURL.getParameter(DYNAMIC_KEY));
                URL monitorUrl = loadMonitor(registryURL); // ⑥
                if (monitorUrl != null) {
                    url = url.addParameterAndEncoded(MONITOR_KEY, monitorUrl.toFullString());
                }
                if (logger.isInfoEnabled()) {
                    logger.info("Register dubbo service " + interfaceClass.getName() + " url " + url + " to registry " + registryURL);
                }
                // For providers, this is used to enable custom proxy to generate invoker
                String proxy = url.getParameter(PROXY_KEY);
                if (StringUtils.isNotEmpty(proxy)) {
                    registryURL = registryURL.addParameter(PROXY_KEY, proxy);
                }
                Invoker<?> invoker = PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(EXPORT_KEY, url.toFullString())); // ⑦
                DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

                Exporter<?> exporter = protocol.export(wrapperInvoker); // ⑧
                exporters.add(exporter);
            }
        } else {
            Invoker<?> invoker = PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, url);
            DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);
            Exporter<?> exporter = protocol.export(wrapperInvoker);
            exporters.add(exporter);
        }
        MetadataReportService metadataReportService = null;
        if ((metadataReportService = getMetadataReportService()) != null) {
            metadataReportService.publishProvider(url);
        }
    }
}
```
①如果scope配置不为none，则进行服务暴露；②如果scope配置不为remote，则在本地暴露服务；③如果scope配置不为local，则在remote暴露服务。④循环每一个注册地址，进行服务暴露；⑤如果protocol为local，则不暴露；⑥根据注册地址，加载monitor地址；⑦通过动态代理机制创建Invoker；⑧调用RegistryProtocol方法暴露服务。    

## dubbo服务发现


未完待续... 