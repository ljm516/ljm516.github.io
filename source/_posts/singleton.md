---
title: Java设计模式之单例模式 
date: 2017-10-12
categories: Java
tags:
- 设计模式
---

## 懒汉式
 全局单例在第一次使用时被构建
 
- 简单实现

<!-- more -->

```java
public class Singleton {
    private static Singleton singleton;
    public static Singleton getInstance(){
        if (singleton == null) {
            singleton = new Singleton()
        }
        
        return singleton;
    } 
}
``` 

- 进一步优化

将构造器私有化，能防止被外部类调用

```java
public class Singleton {
    private static Singleton singleton;
    private Singleton(){}
    public static Singleton getInstance(){
        if (singleton == null) {
            singleton = new Singleton()
        }
        
        return singleton;
    } 
}
```

OK，如果现在加上多线程，多个线程同时运行if (singleton == null), 发现是true，那么都会去创建singleton实例，那就不是单列了！

- 加上同步锁

```java
public class Singleton {
    private static Singleton singleton;
    private Singleton(){}
    public static synchronized Singleton getInstance(){
        if (singleton == null) {
            singleton = new Singleton()
        }
        return singleton;
    } 
}
```
这可以避免多线程同时工作时产生上面的那种情况。但是给getInstance方法加锁，虽然会避免了可能会出现的多个实例问题，但是会强制其它线程等待，实际上会对程序的执行效率造成负面影响

- 双重检查

```java
public class Singleton {
    private static Singleton singleton;
    private Singleton(){}
    public static Singleton getInstance(){
        if (singleton == null) {
            synchronized (Singleton.class){
                if (singleton == null){
                    singleton = new Singleton()
                }
            }
        }
        return singleton;
    } 
}
```
第一个if (singleton == null) 是为了解决效率问题

第二个if (singleton == null) 是为了防止出现多个实例的情况

等等，还有终极版，引入了原子性操作的知识点，然而并没有看懂。留下[链接](http://www.importnew.com/24272.html)。

- 终极版

```java
public class Singleton {
    private static volatile Singleton singleton;
    private Singleton(){}
    public static Singleton getInstance(){
        if (singleton == null) {
            synchronized (Singleton.class){
                if (singleton == null){
                    singleton = new Singleton()
                }
            }
        }
        return singleton;
    } 
}
```

volatile关键字的一个作用是禁止指令重排（上面的链接里有解释）。把singleton声明为volatile后，对它的操作就会有了一个内存屏障，这样，在它赋值完成之前，就不会进行读操作。

### 饿汉式

全局单例在类初始化时被创建

由于类加载的过程是由类加载器来执行的，这个过程也是由JVM来保证同步的，所以它能免疫许多由线程引发的问题。

```java
public Singleton {
    private static final Singleton SINGLETON = new Singleton();
    private Singleton (){}
    public static Singleton getInstance (){
        return SINGLETON;
    }
}
```
缺点:

 - 由于INSTANCE的初始化是在类加载时进行的，而类的加载是由ClassLoader来做的，所以开发者本来对于它初始化的时机就很难去准确把握。
 - 可能由于初始化的太早，造成资源的浪费
 - 如果初始化本身依赖于一些其他数据，那么也就很难保证其他数据会在它初始化之前准备好。


