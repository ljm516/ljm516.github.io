---
title: dubbo源码学习之SPI机制
date: 2019-11-13 21:13:27
categories: Java
tags:
- dubbo
---

本文记录源码学习dubbo spi扩展机制笔记。

<!--more-->

SPI 是Service Provider Interface的缩写，Java SPI 使用了策略模式，一个接口多种实现。与Java SPI相比，Dubbo SPI做了一定的改进和优化，官方文档是这么说的:
![dubbo spi的改进及优化](dubbo-spi改进.jpg)

## 扩展点的配置规范 
dubbo spi 需要在 `META-INF/dubbo/`下放置对应的SPI配置文件，文件名需要命名为接口的全路径名。配置文件的内容为key=扩展点实现类全路径名，如有多个，使用换行符分割。key会作为dubbo spi注解中的传入参数。dubbo spi也兼容了java spi的配置路径和内容配置方式。在dubbo启动时，会默认扫描`META-INF/services/`,`META-INF/dubbo/`,`META-INF/dubbo/internal/`这三个目录的配置文件。
![spi配置规范](spi配置规范.jpg)

## 扩展点的特性
扩展类一共包含四种特性: 自动包装、自动加载、自适应和自动激活。

### 自动包装
自动包装是一种被缓存的扩展类，ExtensionLoader在加载扩展时，如果发现这个扩展类包含其它扩展点作为构造函数的参数，则这个扩展类就会被认为是Wrapper类。
```java
public class ProtocolFilterWrapper implements Protocol {

    private final Protocol protocol;
    // 实现了Protocol，但构造函数中又传了一个Protocol类型参数
    public ProtocolFilterWrapper(Protocol protocol) {
        if (protocol == null) {
            throw new IllegalArgumentException("protocol == null");
        }
        this.protocol = protocol;
    }
}
```
ProtocolFilterWrapper类虽然实现了Protocol接口，但是其构造函数中又注入了一个Protocol的参数，这样ProtocolFilterWrapper就会被认定为是Wrapper类。这是一种装饰器模式，把通用的抽象逻辑进行封装或对子类进行增强，让子类可以更加专注具体的实现。

### 自动加载
如果某个扩展类是另外一个扩展点类的成员属性，并且拥有setter方法，那么框架也会自动注入对应的扩展点实例。ExtensionLoader在执行扩展点初始化时，会自动通过setter方法注入对应的实现类。那如果扩展类属性是一个接口 ，它有多种实现，具体注入又是那个？这就涉及第三个特性---自适应。

### 自适应
在dubbo spi中，使用@Adaptive注解，可以动态地通过URL中的参数来确定使用哪个具体的实现类。从而解决自动加载中的实例注入问题。
```java
/**
 * 描述类的功能.
 * SPI注解，有一个value属性，通过传入这个值，设置该接口的默认实现类。
 *
 * @author lijm
 */
@SPI(value = "dog")
public interface Animal {
    /**
     * 如果Adaptive注解中没有key参数，
     * 则默认会把类名转换成key，Animal类就会转化成animal。
     * 如果url参数中没有animal这个参数，则会取默认值，也就是spi的默认实现。
     */
    @Adaptive(value = {"name"})
    void run(URL url, String s);
}
```
@Adaptive注解的value属性是一个数组类型，可以接收多个参数，@Adaptive注解会依次取value值去匹配，如果能匹配上某个扩展实现类则直接使用对应的实现类。如果都不匹配则抛出异常。
这种动态寻找实现类的方式比较灵活，但是只能激活一个具体实现类，如果需要多个同时激活呢（根据条件不同，同时激活多个实现类）？这就涉及最后一个特性---自动激活。

### 自动激活
使用@Activate注解，可以标记对应的扩展点默认被激活启用。该注解还可以传入不同的参数，设置扩展点在不同的条件下被自动激活。
```java
@SPI(value = "dog")
public interface Animal {
    /**
     * 可以标记在类，接口，枚举类和方法上。主要使用在有多个扩展点实现、需要根据不同条件被激活的场景中。
     * @param url url
     */
    @Activate(group = "default", value = {"dogImp", "sheepImpl"})
    void eat(URL url);
}
// Test
public void testActivate() {
    ExtensionLoader<Animal> extensionLoader = ExtensionLoader.getExtensionLoader(Animal.class);
    Map<String, String> keyPair = new HashMap<>();
    keyPair.put("dogImpl", "dog");
    keyPair.put("sheepImpl", "sheep");
    URL url = new URL("dobbo", "localhost", 1111, keyPair);
    // 根据条件激活
    List<Animal> animalList = extensionLoader.getActivateExtension(url, "dogImpl", "default");
    for (Animal animal : animalList) {
        animal.eat(url);
    }
}
```
## 扩展点相关注解

### @SPI注解
该注解可以用在类、接口和枚举类上，dubbo都是用在接口上。主要用来标记这个接口是一个dubbo spi接口，不同的内置或用户有不同的定义实现，运行时需要通过配置找到具体的类。
```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
public @interface SPI {
    /**
     * default extension name
     */
    String value() default "";
}
```
@SPI注解只有一个value属性，通过这个属性，可以传入不同的参数来设置这个接口的默认实现。
例如:
```java
/**
 * 描述类的功能.
 * SPI注解，有一个value属性，通过传入这个值，设置该接口的默认实现类。
 *
 * @author lijm
 */
@SPI(value = "dog")
public interface Animal {
    
}
// test for get spi default
/**
 * 扩展点注解测试
 * @SPI注解，可以通过传入value参数，设置接口的默认实现类
 */
@Test
public void testSPI() {
    ExtensionLoader<Animal> extensionLoader = ExtensionLoader.getExtensionLoader(Animal.class);
    Animal animal = extensionLoader.getDefaultExtension(); // 通过这个方法获取默认实现
    animal.talk(); // 这里应该是dog，因为Animal类的默认扩展实现类是Dog
}
```
自定一个Animal接口，通过@SPI注解，传入dog表示Dog类这个接口的默认实现类。

### @Adaptive注解
该注解可以定义在类、接口、枚举类和方法上，如果定义在接口的方法上，则可以通过参数动态获得实现类（自适应特性）。方法级别注解在第一个getExtension时，会自动生成和编译一个动态的Adaptive类，从而达到动态实现类的效果。
```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface Adaptive {
    String[] value() default {};
}
```
在上面的自适应特性的地方已经给出了例子，这里不重复。在扩展点接口的多个实现里，只能有一个实现上可以加@Adaptive注解，如果多个实现类都有该注解，则会抛出异常`More than 1 adptive calss found`。从@Adaptive注解的代码知道，也传入一个value传输，value参数是一个数组，可以传入多个值，对传入的URL进行key匹配，如果第一个没匹配上，就匹配第二个，如果都没匹配上，则使用@SPI注解中的默认实现。如果@SPI也没默认值，则抛出IllegalStateException。

### @Activate注解
该注解使用在类、接口、枚举和方法上。主要使用在有个扩展点实现、需要根据不同条件被激活的场景中。
```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface Activate {
    String[] group() default {}; // URL中的分组如果匹配则激活，可以配置多个
    String[] value() default {}; // 查找URL中如果含有该key值，则会激活
    @Deprecated
    String[] before() default {}; // 填写扩展点列表，表示哪些扩展点要在本扩展点之前（已废弃）
    @Deprecated
    String[] after() default {}; // 同上，表示哪些需要在本扩展点之后（已废弃）
    int order() default 0; // 直接的排序信息
}
```
自动激活特性中给出了获取自动激活实现的例子，不在重复。

## ExtensionLoader原理
ExtensionLoader是整个扩展机制的主要逻辑类，在这个类里面实现了配置的加载、扩展类缓存、自适应对象生成等所有工作。ExtensionLoader的逻辑入口分为getExtension、getAdaptiveExtension和getActivateExtension三个，分别是获取普通扩展类、获取自适应扩展类、获取自动激活的扩展类。

### getExtension的实现原理
下面将从getExtension方法为入口，跟着方法调用栈逐步分析一下几个方法：
- getExtension(String name)
- createExtension(String name)
- getExtensionClasses()
- loadExtensionClasses()
- loadDirectory(Map<String, Class<?>> extensionClasses, String dir, String type)
- loadResource(Map<String, Class<?>> extensionClasses, ClassLoader classLoader, java.net.URL resourceURL)
- loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL, Class<?> clazz, String name)

这几个方法的实现一起组合成getExtension方法实现原理，也是接下来创建自适应扩展类和自动激活扩展类的数据基础。

#### 源码分析getExtension(String name)方法
```java
/**
 * 根据给定的name参数，找到扩展类，如果找不到则抛出IllegalStateException
 */
public T getExtension(String name) {
    if (StringUtils.isEmpty(name)) {
        throw new IllegalArgumentException("Extension name == null");
    }
    if ("true".equals(name)) { // ①
        return getDefaultExtension();
    }
    final Holder<Object> holder = getOrCreateHolder(name); // ②
    Object instance = holder.get();
    if (instance == null) {
        synchronized (holder) {
            instance = holder.get();
            if (instance == null) {
                instance = createExtension(name); // ③
                holder.set(instance);
            }
        }
    }
    return (T) instance;
}
```
①中如果name是true，则表示去默认扩展类；②获取扩展类的holder，Holder类持有扩展类的实例，获取Holder的逻辑是先从缓存中通过name获取，如果没有获取到就new一个；③获取扩展类的实例，是先从第二步中获取的Holer类中尝试获取扩展类实例，如果没有获取到，则通过createExtension方法创建实例。

#### 源码分析createExtension(String name)方法
```java
private T createExtension(String name) {
    Class<?> clazz = getExtensionClasses().get(name); // ①
    if (clazz == null) {
        throw findException(name);
    }
    try {
        T instance = (T) EXTENSION_INSTANCES.get(clazz); // ②
        if (instance == null) {
            EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
        injectExtension(instance); // ③
        Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        if (CollectionUtils.isNotEmpty(wrapperClasses)) {
            for (Class<?> wrapperClass : wrapperClasses) { // ④
                instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance)); 
            }
        }
        return instance;
    } catch (Throwable t) {
        throw new IllegalStateException("Extension instance (name: " + name + ", class: " +
                type + ") couldn't be instantiated: " + t.getMessage(), t);
    }
}
```
①中从缓存中通过name获取实例的clazz，如果获取不到，抛出异常；②中通过class从缓存个中获取实例，如果没有获取到，通过反射的方式创建实例；③向扩展类注入其它依赖的属性，如果扩展A又依赖了扩展类B。④遍历所有的扩展点包装类，用于初始化包装类实例，然后找到构造函数为type(也就是扩展类的类型)的包装类，为期注入扩展类实例。(cachedWrapperClasses)的来源在getExtensionClasses()方法。

#### 源码分析getExtensionClasses()方法
```java
private Map<String, Class<?>> getExtensionClasses() {
    Map<String, Class<?>> classes = cachedClasses.get();
    if (classes == null) {
        synchronized (cachedClasses) {
            classes = cachedClasses.get();
            if (classes == null) {
                classes = loadExtensionClasses();
                cachedClasses.set(classes);
            }
        }
    }
    return classes;
}
```
该方法逻辑也是先从缓存中获取扩展类，如果缓存没有获取，这通过loadExtensionClasses()方法加载，所以现在可以看出，获取扩展类的核心就在loadExtensionClasses()方法。

#### 源码分析loadExtensionClasses()方法
```java
private Map<String, Class<?>> loadExtensionClasses() {
    cacheDefaultExtensionName(); // ①

    Map<String, Class<?>> extensionClasses = new HashMap<>();
    // ②
    loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY, type.getName());
    loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
    loadDirectory(extensionClasses, DUBBO_DIRECTORY, type.getName());
    loadDirectory(extensionClasses, DUBBO_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
    loadDirectory(extensionClasses, SERVICES_DIRECTORY, type.getName());
    loadDirectory(extensionClasses, SERVICES_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
    return extensionClasses;
}
```
①中主要是默认扩展类的加载逻辑，通过扫描@SPI注解，获取@SPI注解的value值，作为默认扩展的key。如果存在多个默认实现则抛出异常。②则为加载扩展类配置文件，主要扫描`META-INF/dubbo/interanl/`,`META-INF/dubbo/`,`META-INF/services/`这个是那个文件夹。

#### 源码分析loadDirectory(Map<String, Class<?>> extensionClasses, String dir, tring type)方法
```java
private void loadDirectory(Map<String, Class<?>> extensionClasses, String dir, String type) {
    String fileName = dir + type; // ①
    try {
        Enumeration<java.net.URL> urls;
        ClassLoader classLoader = findClassLoader(); // ②
        if (classLoader != null) {
            urls = classLoader.getResources(fileName); // ③
        } else {
            urls = ClassLoader.getSystemResources(fileName);
        }
        if (urls != null) {
            while (urls.hasMoreElements()) {
                java.net.URL resourceURL = urls.nextElement();
                loadResource(extensionClasses, classLoader, resourceURL); // ④
            }
        }
    } catch (Throwable t) {
        logger.error("Exception occurred when loading extension class (interface: " +
                type + ", description file: " + fileName + ").", t);
    }
}
```
①构建文件名，文件的命名方式为类的全限定名；②获取类加载器；③urls中存放的文件里配置的可选可扩展类，通过classLoader加载classes文件目录下的扩展类配置文件（打包后的路径）的路径，然后放入urls中；④通过loadResource方法加载扩展类。

#### 源码分析loadResource(Map<String, Class<?>> extensionClasses, ClassLoader classLoader, java.net.URL resourceURL) 方法
```java
private void loadResource(Map<String, Class<?>> extensionClasses, ClassLoader classLoader, java.net.URL resourceURL) {
    try {
        try (BufferedReader reader = new BufferedReader(new InputStreamReader(resourceURL.openStream(), StandardCharsets.UTF_8))) { // ①
            String line;
            while ((line = reader.readLine()) != null) {
                final int ci = line.indexOf('#');
                if (ci >= 0) {
                    line = line.substring(0, ci);
                }
                line = line.trim();
                if (line.length() > 0) {
                    try {
                        String name = null;
                        int i = line.indexOf('=');
                        if (i > 0) {
                            // ②
                            name = line.substring(0, i).trim(); //这就是配置的key
                            line = line.substring(i + 1).trim(); //这是扩展类的全限定名
                        }
                        if (line.length() > 0) {
                            loadClass(extensionClasses, resourceURL, Class.forName(line, true, classLoader), name); //③
                        }
                    } catch (Throwable t) {
                       // handle Exception
                    }
                }
            }
        }
    } catch (Throwable t) {
        // handle Exception
    }
}
```
①根据扩展类配置文件路径，创建BufferedReader，循环读取每一行；②解析出扩展类的key和类全限名，通过`=`分割；③然后执行loadClass方法。

#### 源码分析loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL, Class<?> clazz, String name)方法
在调用该方法之前，会先通过Class.forName方法得到扩展类的clazz。

```java
private void loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL, Class<?> clazz, String name) throws NoSuchMethodException {
    if (!type.isAssignableFrom(clazz)) {
        throw new IllegalStateException("Error occurred when loading extension class (interface: " +
                type + ", class line: " + clazz.getName() + "), class "
                + clazz.getName() + " is not subtype of interface.");
    }
    if (clazz.isAnnotationPresent(Adaptive.class)) { // ①
        cacheAdaptiveClass(clazz);
    } else if (isWrapperClass(clazz)) { // ②
        cacheWrapperClass(clazz);
    } else {
        clazz.getConstructor();
        if (StringUtils.isEmpty(name)) {
            name = findAnnotationName(clazz); // ③
            if (name.length() == 0) {
                throw new IllegalStateException("No such extension name for the class " + clazz.getName() + " in the config " + resourceURL);
            }
        }

        String[] names = NAME_SEPARATOR.split(name);
        if (ArrayUtils.isNotEmpty(names)) {
            cacheActivateClass(clazz, names[0]); // ④
            for (String n : names) {
                cacheName(clazz, n);  // ⑤
                saveInExtensionClass(extensionClasses, clazz, n); // ⑥
            }
        }
    }
}
```
①判断当前扩展类是否为自适应扩展类，如果是将clazz复制给cachedAdaptiveClass。②判断当前扩展类是否为包装类，判断依据就是看当前类是否有一个public的参数为要实现扩展的接口的构造函数。如果是包装类，将clazz添加到cachedWrapperClasses（set集合）。③如果传过来的name参数（扩展类配置文件的key）为空，则通过findAnnotationName方法获取name，如果还是获取不到，则抛出异常，提示找不到扩展名。④将clazz和name的信息存入cachedActivates（map），key=name，value=@Activite注解；⑤保存class和name的信息到cachedNames（map），key=clazz，value=name；⑥将clazz放入extensionClasses（map），key=name, value=clazz。


#### getExtension实现小结
从上述分析，主要是通过getExtensionClasses方法获取所有的可扩展类，具体需要的那个扩展类就通过name参数获取。在getExtensionClasses方法中，会将扩展类需要的自适应扩展类和自动激活扩展类缓存起来，用于后续使用。需要提一下的是:扩展类实例被创建后，会放入EXTENSION_INSTANCES（map），key=clazz，value=类实例。自适应扩展类缓存cachedAdaptiveClass，自适应扩展类只能有一个。自动激活扩展类缓存为cachedActivates（map），key=扩展类配置文件的key，value=@Activate。包装类缓存在cachedWrapperClasses（set），存储的是扩展类的clazz。

### getAdaptiveExtension的实现原理
```java
 public T getAdaptiveExtension() {
    Object instance = cachedAdaptiveInstance.get(); // ①
    if (instance == null) {
        if (createAdaptiveInstanceError == null) {
            synchronized (cachedAdaptiveInstance) {
                instance = cachedAdaptiveInstance.get();
                if (instance == null) {
                    try {
                        instance = createAdaptiveExtension(); // ②
                        cachedAdaptiveInstance.set(instance);
                    } catch (Throwable t) {
                        // handle exception
                    }
                }
            }
        } else {
            // handle exception
        }
    }
    return (T) instance;
}
```
①获取缓存获取，如果没有去创建。②创建自适应扩展类。

#### 源码分析createAdaptiveExtension()方法
```
private T createAdaptiveExtension() {
    try {
        return injectExtension((T) getAdaptiveExtensionClass().newInstance());
    } catch (Exception e) {
        throw new IllegalStateException("Can't create adaptive extension " + type + ", cause: " + e.getMessage(), e);
    }
}
```
这个方法主要还是调用getAdaptiveExtensionClass方法，获取扩展类clazz，然后通过反射创建类实例。然后执行injectExtension方法，为其注入引入的其它扩展类。下面看下getAdaptiveExtensionClass的实现。

#### 源码分析getAdaptiveExtensionClass()方法
```java
private Class<?> getAdaptiveExtensionClass() {
    getExtensionClasses(); // ①
    if (cachedAdaptiveClass != null) {
        return cachedAdaptiveClass;
    }
    return cachedAdaptiveClass = createAdaptiveExtensionClass(); // ②
}
```
①首先执行getExtensionClasses()方法，确定扩展类都被加载了。cachedAdaptiveClass是创建扩展附加产物，如果扩展类没有指定自适应扩展类，则通过②来获取自适应扩展类。

#### 源码分析createAdaptiveExtensionClass()方法
```java
private Class<?> createAdaptiveExtensionClass() {
    String code = new AdaptiveClassCodeGenerator(type, cachedDefaultName).generate(); 
    ClassLoader classLoader = findClassLoader();
    org.apache.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(org.apache.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
    return compiler.compile(code, classLoader);
}
```
该方法的逻辑就是根据默认的扩展类实现，默认的扩展类实现是@SPI注解的value，然后生成默认扩展类的类文件代码，通过编译器编译生成类。

### getActivateExtension的实现原理
```java
public List<T> getActivateExtension(URL url, String[] values, String group) {
    List<T> exts = new ArrayList<>();
    List<String> names = values == null ? new ArrayList<>(0) : Arrays.asList(values);
    if (!names.contains(REMOVE_VALUE_PREFIX + DEFAULT_KEY)) {
        getExtensionClasses();
        for (Map.Entry<String, Object> entry : cachedActivates.entrySet()) {
            String name = entry.getKey();
            Object activate = entry.getValue();

            String[] activateGroup, activateValue;

            if (activate instanceof Activate) {
                activateGroup = ((Activate) activate).group();
                activateValue = ((Activate) activate).value();
            } else if (activate instanceof com.alibaba.dubbo.common.extension.Activate) {
                activateGroup = ((com.alibaba.dubbo.common.extension.Activate) activate).group();
                activateValue = ((com.alibaba.dubbo.common.extension.Activate) activate).value();
            } else {
                continue;
            }
            if (isMatchGroup(group, activateGroup)) {
                T ext = getExtension(name);
                if (!names.contains(name)
                        && !names.contains(REMOVE_VALUE_PREFIX + name)
                        && isActive(activateValue, url)) {
                    exts.add(ext);
                }
            }
        }
        exts.sort(ActivateComparator.COMPARATOR);
    }
    List<T> usrs = new ArrayList<>();
    for (int i = 0; i < names.size(); i++) {
        String name = names.get(i);
        if (!name.startsWith(REMOVE_VALUE_PREFIX)
                && !names.contains(REMOVE_VALUE_PREFIX + name)) {
            if (DEFAULT_KEY.equals(name)) {
                if (!usrs.isEmpty()) {
                    exts.addAll(0, usrs);
                    usrs.clear();
                }
            } else {
                T ext = getExtension(name);
                usrs.add(ext);
            }
        }
    }
    if (!usrs.isEmpty()) {
        exts.addAll(usrs);
    }
    return exts;
}
```
主要逻辑为根据传入的value数组(选择的自动激活的扩展类)创建扩展类，value数组是URL中指定的key（多个用逗号隔开）。exts.sort(ActivateComparator.COMPARATOR) 排序是根据@Activate中配置的 before、after、order等参数进行排序的。需要注意的是，如果URL参数中传入了`-default`，则所有的默认@Activate都不会被激活，只有URL参数中指定的扩展点会被激活。如果传入`-`符开头的扩展名，则该扩展点不会被激活。

### ExtensionFactory的实现原理
从上面获取扩展类的实现看，ExtensionLoader是整个扩展类实现的核心，那么ExtensionLoader又是怎么创建的呢？
首先看ExtensionLoader的构造函数:
```java
private ExtensionLoader(Class<?> type) {
    this.type = type;
    objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
}
```
ExtensionLoader是通过ExtensionFactory创建的，ExtensionFactory有三个实现类，AdaptiveExtensionFactory、SpiExtensionFactory和SpringExtensionFactory，也就是说dubbo spi管理容器还可以从spring容器中获取。其中AdaptiveExtensionFactory类上加上了@Adaptive注解。

#### 源码分析SpringExtensionFactory的getExtension方法
```java
public <T> T getExtension(Class<T> type, String name) {
    //SPI should be get from SpiExtensionFactory
    if (type.isInterface() && type.isAnnotationPresent(SPI.class)) {
        return null;
    }
    // ①
    for (ApplicationContext context : CONTEXTS) {
        if (context.containsBean(name)) {
            Object bean = context.getBean(name);
            if (type.isInstance(bean)) {
                return (T) bean;
            }
        }
    }
    // log
    if (Object.class == type) {
        return null;
    }
    // ②
    for (ApplicationContext context : CONTEXTS) {
        try {
            return context.getBean(type);
        } catch (NoUniqueBeanDefinitionException multiBeanExe) {
            // log
        } catch (NoSuchBeanDefinitionException noBeanExe) {
            // log
        }
    }
    // log
    return null;
}
```
①的逻辑就是遍历所有spring上下文，先根据名字从spring容器中查找；②如果①中根据名字没找到，则直接根据类型查找。

那么Spring的上下文又是在什么时候保存的呢？在ReferenceBean和ServiceBean创建时，会调用静态方法保存spring上下文，即一个服务被发布和被引用的时候，对应的spring上下文被保存下来，这个逻辑在下一遍解析。

#### 源码分析SpiExtensionFactory的getExtension方法
SpiExtensionFactory主要是获取扩展类接口对应的Adaptive实现类。
```java
public class SpiExtensionFactory implements ExtensionFactory {
    @Override
    public <T> T getExtension(Class<T> type, String name) {
        if (type.isInterface() && type.isAnnotationPresent(SPI.class)) {
            ExtensionLoader<T> loader = ExtensionLoader.getExtensionLoader(type); // ①
            if (!loader.getSupportedExtensions().isEmpty()) { // ②
                return loader.getAdaptiveExtension(); 
            }
        }
        return null;
    }
}
```
①根据类型获取所有的扩展点加载器；②loader.getSupportedExtension()方法获取缓存的扩展类，如果缓存的扩展类不为空，调用ExtensionLoader的getAdaptiveExtension方法创建扩展类。

#### 源码分析AdaptiveExtensionFactory 
```java
@Adaptive
public class AdaptiveExtensionFactory implements ExtensionFactory {

    private final List<ExtensionFactory> factories;

    public AdaptiveExtensionFactory() {
        ExtensionLoader<ExtensionFactory> loader = ExtensionLoader.getExtensionLoader(ExtensionFactory.class);
        List<ExtensionFactory> list = new ArrayList<ExtensionFactory>();
        for (String name : loader.getSupportedExtensions()) { // ①
            list.add(loader.getExtension(name));
        }
        factories = Collections.unmodifiableList(list);
    }
    @Override
    public <T> T getExtension(Class<T> type, String name) {
        for (ExtensionFactory factory : factories) {
            T extension = factory.getExtension(type, name); // ②
            if (extension != null) {
                return extension;
            }
        }
        return null;
    }
}
```
factories工厂列表，也是通过spi实现的，因此可以在这里获取所有工厂的扩展点加载器。被AdaptiveExtensionFactory 缓存的工厂会通过TreeSet进行排序，SPI排在前面，Spring排在后面。当调用getExtension方法时，优先从SPI容器中获取扩展类，如果没找到，在从spring容器中找。
①遍历所有的工厂名称，获取对应的工厂，并保存到factories列表中。②在getExtension的实现中，只是遍历了factories，最终还是调用SPI或者Sping工厂实现的getExtension方法。

## 扩展点动态编译实现
dubbo中有三种代码编译器，分别是JDK、Javassist和AdaptiveCompiler这三种编译器。都实现了Compiler接口。Compiler接口上有个@SPI注解，默认值为@SPI("javassist")，表示Javassist作为默认编译器器。
```java
@SPI("javassist")
public interface Compiler {
    /**
     * Compile java source code.
     *
     * @param code        Java source code
     * @param classLoader classloader
     * @return Compiled class
     */
    Class<?> compile(String code, ClassLoader classLoader);
}
```
可以通过`<dubbo:application compiler="jdk">`来指定编译器类型。

### 源码分析AdaptiveCompiler
```java
@Adaptive
public class AdaptiveCompiler implements Compiler {

    private static volatile String DEFAULT_COMPILER;

    public static void setDefaultCompiler(String compiler) {
        DEFAULT_COMPILER = compiler; 
    }

    @Override
    public Class<?> compile(String code, ClassLoader classLoader) {
        Compiler compiler;
        ExtensionLoader<Compiler> loader = ExtensionLoader.getExtensionLoader(Compiler.class);
        String name = DEFAULT_COMPILER; // copy reference
        if (name != null && name.length() > 0) {
            compiler = loader.getExtension(name);
        } else {
            compiler = loader.getDefaultExtension();
        }
        return compiler.compile(code, classLoader);
    }
}
```
AdaptiveCompiler上有@Adaptive注解，说明AdaptiveCompiler会固定为默认实现，这个compiler的作用和AdaptiveExtensionFactory类型，用来管理其他的compiler。

setDefaultCompiler方法为设置默认的编译器名称。compile方法，先通过ExtensionLoader获取对应的编译器扩展类实现，然后调用真正的compile做编译。

## 本文总结
本文主要学习了dubbo spi机制，了解了spi的一些概要信息，包括spi的新特性、配置规范、内部缓存等。然后学习了spi最重要的三个注解:@SPI、@Adaptive、@Activate，以及这三个注解的实现原理。然后结合ExtensionLoader类，源码分析了getExtension、getAdaptiveExtension、getActivateExtension这几个重要方法。最后简单了解了ExtensionFactory和自适应机制中的动态编译实现。
