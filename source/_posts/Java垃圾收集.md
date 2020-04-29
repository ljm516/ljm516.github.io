---
title: Java垃圾收集
date: 2019-02-24 22:02:12
categories: Java
tags:
- JVM
---

本文记录阅读周志明老师的《深入理解Java虚拟机》关于Java垃圾收集的笔记。

<!--more-->

## 什么是垃圾
Jvm在对垃圾进行回收之前，会先判断对象是否可以回收，Jvm提供了两种算法来确定对象是否可以回收。
- 引用计数器
- 可达性分析

### 引用计数器
给对象中添加一个引用计数器，每当有一个地方引用，计数器值就加1。当引用失效时，计数器值就减1。任何时刻计数器为0的对象就是不可能再被使用的。
优点: 实现简单，判定效率高。
缺点: 无法解决相互循环引用问题。

### 可达性分析算法
这个算法的基本思路就是通过一系列成为“GC Roots”的对象作为起始点，从这些点开始向下搜索，搜索所走过的路径成为引用链，当一个对象到GC Roots没有任何引用链相连时，则证明此对象是不可用的。

可以作为GC Roots的对象包括以下几种:
- 虚拟机栈中引用的对象
- 方法区中类静态属性引用的对象
- 方法区中常量引用的对象
- 本地方法中JNI引用的对象。


## 垃圾收集算法

### 标记-清除算法
最基础的垃圾收集算法。分为”标记“和“清除”两个阶段：首先标记处所有需要回收的对象，在标记完成后统一回收所有被标记的对象，之所以称之为最基础的算法，是因为后续的收集算法都是基于这种思路并对其不足进行改进而得到的。主要不足有两个：一个效率问题，标记和清除两个过程效率都不高；另一个是空间问题，标记清除之后会产生大量不连续的内存碎片，空间碎片太多可能会导致以后再程序运营过程中需要分配较大对象时，无法找到足够的连续内存而不得不提前触发另一次垃圾收集动作。

### 复制算法
为了解决效率问题，一种称之为“复制”的收集算法出现了。它将可用的内存按容量划分为大小相等的两块，每次只使用其中的一块。当这一块内存用完了，就将还存活着的对象复制到另一块上面，然后再把已使用过的内存空间一次清理掉。这个算法适用于对象存活量较少的区域（新生代），这样只需要通过很小的代价就可以实现存活对象的转移，并且不会产生内存碎片。缺点，每次只能使用一半的内存区域，而且需要复制的过程。

### 标记-整理算法
根据老年代的特点，有人提出了另一种“标记-整理”算法，标记过程仍然与“标记-清除”算法一样，但后续步骤不是直接对可回收对象进行清理，而是让所有存活的对象都向一端移动，然后直接清理掉端边界以外的内存。

### 分代收集算法
这种算法并没有新的思想只是根据对象存活周期的不同将内存划分为几块，一般是把Java堆分为新生代和老生代，这样就可以根据各个年代的特点采用最适当的收集算法。

## 垃圾收集器
HotSpot虚拟机的垃圾收集器
![垃圾收集器](垃圾收集器.jpg)

### Serial 收集器
一个单线程的收集器，作用于新生代。它在进行垃圾收集时，必须暂停其他所有的工作线程，直到它收集结束。对应的新生年代在gc日志中的名称为"[DefNew"

### ParNew 收集器
其实是Serial收集器的多线程版本，作用于新生代。对应的新生代在gc日志的名称为"[ParNew"。

### Parallel Scavenge 收集器
它是一个新生代收集器，也是使用复制算法的收集器，又是并行的多线程收集器。它的特点是它的关注点与其他收集器不同，CMS等收集器的关注点尽可能地缩短垃圾收集时用户线程的停顿时间，而Parallel Scavenge收集器的目标则是达到一个可控制的吞吐量。所谓的吞吐量就是CPU用于运行用户代码的时间与CPU总消耗时间的比值，即吞吐量=运行用户代码时间/（运行用户代码时间+垃圾收集时间）。对应的新生代在gc日志的名称为"[PSYoungGen"

### Serial Old 收集器
Serial Old是Serial收集器的老年代版本，它同样是一个单线程收集器，使用“标记-整理”算法。这个收集器的主要意义也是在于给client模式下的虚拟机使用。

### Parallel old 收集器
它是Parallel Scavenge收集器的老年代版本，使用多线程和“标记-整理”算法。

### CMS 收集器
CMS（Concurrent Mark Sweep）收集器是一种以获取最短回收停顿时间为目标的收集器。它是基于“标记-清除”算法实现的，它的运行过程相对于前面几种收集器来说更复杂一些，整个过程分为4个步骤：
* 初始化标记（CMS initial mark）
* 并发标记（CMS concurrent mark）
* 重新标记（CMS remark）
* 并发清除（CMS concurrent sweep）

初始标记、重新标记这两个步骤仍然需要停止用户程序。初始标记仅仅是标记一下GC Roots能直接关联到的对象，速度很快，并发标记阶段就是进行GC Roots Tracing的过程，而重新标记阶段则是为了修正并发标记期间因用户程序继续运行而导致标记产生变动的那一部分对象标记记录。由于整个过程耗时最长的并发标记和并发清除过程收集器线程可以和用户线程一起执行，所以，从整体上说，CMS收集器的内存回收过程是与用户线程一起并发执行的。

缺点:
1. 对cpu资源敏感。虽然在并发阶段，不会导致用户线程停顿，但是会因为占用了一部分线程而导致应用程序变慢，总吞吐量会降低。
2. 无法处理浮动垃圾。由于cms并发清理阶段用户线程还在运行着，自然会新的垃圾产生，这部分垃圾出现在标记过程之后，cms无法在本次垃圾回收过程处理掉，只能等待下一次GC时在清理。它不能像其他的垃圾回收器一样，等待老年代几乎填满再去回收，需要预留一部分空间提供并发收集时的程序运行使用。
3. 基于标记-清除算法，会产生大量空间碎片。对大对象的分配就会造成很大麻烦，可能因为无法找到一个连续的空间分配个大对象，而提前触发一次fullGC。


### G1 收集器
G1（Garbage-First）收集器是当今收集器技术发展的最前沿成果之一。G1是一款面向服务端应用的垃圾收集器，与其他GC收集器相比，G1具备如下特点：
* 并行和并发：充分利用多CPU、多核环境下的硬件优势，使用多个CPU和缩短stop-the-world停顿时间，部分其他收集器原本需要停顿Java线程执行的GC动作，G1收集器仍然可以通过并发的方式让Java程序继续执行。
* 分代收集：与其他收集器一样，分代概念在G1中依然得以保留。
* 空间整合：与CMS的“标记-清理”算法不同，G1从整体来看是基于“标记-整理”算法实现的收集器，从局部上来看是基于“复制”算法实现的，但无论如何，这两种算法都意味着G1运行期间不会产生内存空间碎片，收集后能提供规整的可用内存。
* 可预测的停顿：这是G1相对于CMS的另一大优势，降低停顿时间是G1和CMS共同的关注点，但G1除了追求低停顿外，还能建立可预测的停顿时间模型，能让使用者明确指定在一个长度为M毫秒的时间片段内，消耗在垃圾收集的时间不得超过N毫秒，这几乎是实时Java的垃圾收集器的特征了。

### 垃圾收集器参数总结
| 参数 | 描述 |
| --- | --- |
| UseSerialGC | 虚拟机运行在client模式下的默认值，打开此开关后，使用Serial+Serial old的收集器组合进行内存回收 |
| UseParNewGC  | 打开此开关后，使用ParaNew+Serial Old的收集器组合进行内存回收 |
| UseConcMarkSweepGC | 打开此开关后，使用ParaNew+CMS+Serial Old的收集器组合进行内存回收。Serial Old收集器将作为CMS收集器出现Concurrent Mode Failure失败后的后备收集器使用 |
| UseParallelGC | 虚拟机运行在Server模式下的默认值，打开此开关后，使用Parallel Scavenge+Serial Old（ps MarkSweep）的收集器组合进行内存回收  |
| UseParallelOldGC | 打开此开关后，使用Parallel Scavenge + Parallel Old 的收集器组合进行内存回收 |
| SurvivorRatio | 新生代中Eden区域的Survivor区域的容量比值，默认为8，代表Eden:Survivor=8:1 |
| PretenureSizeThreshold | 直接晋升到老年代的对象大小，设置这个参数后，大于这个参数的对象直接在老年代分配 |
| MaxTenuringThreshold | 晋升到老年代的对象年龄，每个对象在坚持过一次Minor GC之后，年龄就加1，当超过这个参数值就进入老年代。 |
| UseAdaptiveSizePolicy | 动态调整Java堆中各个区域的大小以及进入老年代的年龄 |
| HandlePromotionFailure | 是否允许分配担保失败，即老年代的剩余空间不足以应付新生代的整个Eden和Survivor区的所有对象都存活的极端情况。 |
| ParallelGCThreads | 设置并行GC时进行内存回收的线程数。 |
| GCTimeRatio | GC时间占总时间的比率， 默认值时99，即允许1%的GC时间。仅在使用Parallel Scavenge收集器时生效 |
| MaxGCPauseMillis | 设置GC的最大停顿时间，仅在使用Parallel Scavenge收集器时生效 |
| CMSInitiatingOccupancyFraction | 设置CMS收集器在老年代空间被使用多少后触发垃圾收集，默认值为68%，仅在使用CMS收集器时生效。  |
| UseCMSCompactAtFullCollection | 设置CMS收集器在完成垃圾收集后是否要进行一次内存碎片整理。仅在使用CMS收集器时生效 |
| CMSFullGCsBeforeCompaction | 设置CMS收集器在进行若干次垃圾收集后再启动一次内存碎片整理。仅在使用CMS收集器事生效。 |

## 几个常用的JVM性能监控命令行工具

- jps
JVM Process Status Tool，显示指定系统内所有的HotSpot虚拟机进程
- jstat
JVM Statistics Monitoring Tool，用于收集HotSpot虚拟机各方面的运行数据
- jinfo
显示虚拟机配置信息
- jmap
Memory Map for Java，生成虚拟机的内存转储快照（heapdump文件）
- jhat
JVM Heap Dump Browser，用于分析heaphump文件，它会建立一个Http/html服务器，让用户可以在浏览器上查看分析结果
- jstack
Stack Trace for Java，显示虚拟机的线程快照

### jps： 虚拟机进程状况工具
jps（JVM Process Status）列出正在运行的虚拟机进程，并显示虚拟机执行主类名称以及这些进程的本地虚拟机唯一ID（Local Virtual Machine Identifier， LVMID）。

命令格式：
jps [ options ] [ hostid ]

- jps -q 
只输出LVMID，省略主类的名称
- jps -m
输出虚拟机进程启动时传递给主类main()函数的参数
- jps -l
输出主类的全名，如果进程执行的是Jar包，输出 Jar 路径
- jps -v
输出虚拟机进程启动时JVM参数

### jstat: 虚拟机统计信息监视工具
jstat(JVM Statistics Monitoring Tool)用于监视虚拟机各种运行状态信息的命令行工具。可以显示本地或者远程虚拟机进程中的类装载、内存、垃圾收集、JIT编译等运行数据。

命令格式:
jstat [ option vmid [interval [s|ms] [count]] ]

如果是远程虚拟机进程，VMID 的格式是: `[protocol:][//]lvmid[@hostname[:port]/servername]`

interval 和 count 分别代表查询间隔和次数。如果省略这两个参数，表示只查询一次。例如：`jstat -gc 2746 250 20` 表示每250ms查看一些vm进程号为2746的gc情况，查询20次。

option 代表用户希望查询的虚拟机信息，主要分三类: 类装载、垃圾收集、运行期编译状况。

- -class
监视类装载、卸载数量、总空间以及类装载所耗费的时间
- -gc
监视java堆状况，包括Eden区、两个Survivor区、老年代、永久代等的容量、已用空间、GC时间合计等信息
- -gccapacity
监视内容与-gc基本相同，但输出主要关注Java堆各个区域使用到的最大、最小空间
- -gcutil
监视内容与-gc基本相同，但输出主要关注已使用空间占总空间的百分比
- -gccause
与-gcunit功能一样，但是会额外输出导致上一次GC产生的原因
- -gcnew
监视新生代gc状况
- -gcnewcapacity
监视内容和-gcnew基本相同，输出的用户要关注使用到的最大、最小空间
- -gcold
监视老年代gc状况
- -gcoldcapacity
监视内容和-gcold基本相同，输出主要关注使用到的最大、最小空间
- -gcpermecapaticy
输出永久代使用的最大、最小空间
- -compiler
输出JIT编译器编译过的方法、耗时等信息
- -printcompilation
输出已经被JIT编译的方法

### jinfo: Java配置信息工具
jinfo(Configuration info for java)作用是实时地查看和调整虚拟机各项参数。使用jps命令的-v参数可以查看虚拟机启动时显式指定的参数列表，可以使用jinfo -flag 选项进行查询未被显式指定的参数。可以使用-flag[+/-]name或者-flag name=value修改一部分运行期间可写的虚拟机参数值。

jinfo 命令格式:

jinfo [ option ] pid

### jmap: Java内存映像工具
jmap(Memory Map for Java)命令用于生成堆转存储快照（一般称为heapdump或dump文件）。不用jmap命令情况下，使用-XX:+HeapDumpOnOutOfMemoryError参数，让jvm在OOM异常是自动生成dump文件。通过-XX:+HeapDumpOnCtrlBreak参数则可以使用[Ctrl]+[Break]键让jvm生成dump文件。又或者在Linux系统下，通过kill -3命令发送进程退出信号，也可以拿到dump文件。

jmap 命令格式:

jmap [ option ] vmid

#### 主要选项

- -dump
生成java堆转储快照。格式为: `-dump:[live, ]format=b, file=<filename>`，其中live子参数说明是否只dump储存活对象。
- -finalizerinfo
显示在F-Queue中等待Finalizer线程执行finalize方式的对象。只在Linux/Solaris平台下有效。
- -heap
显示Java堆详细信息，如使用哪种回收器、参数配置、分代状况等，只在Linux/Solaris平台下有效。
- -histo
显示堆中对象统计信息，包括类、实例数量、合计容量
- -permstat
以ClassLoader为统计口径显示永久代内存状态。只在Linux/Solaris平台下有效。
- -F
当前jvm进程对-dump选项没有响应时，可使用这个选项强制生层dump快照。只在Linux/Solaris平台下有效。

### jhat: 虚拟机堆转储快照分析工具
jhat(JVM Heap Analysis Tool)和jmap搭配使用，来分析jmap生成的堆转储快照。`jhat <dumpfile>`

### jstack: Java堆栈跟踪工具

jstack(Stack Trace for Java)命令用于生成虚拟机当前时刻的线程快照（一般称为threaddump或者javacore文件）。线程快照就是当前虚拟机内每一条线程正在执行的方法堆栈的集合，生成线程快照的主要目的是定位线程出现长时间停顿的原因，如线程死锁、
死循坏、请求外部资源导致的长时间等待等都是导致线程长时间停顿的常见原因。

jstack 命令格式:

jstack [ option ] vmid

#### 主要选项
- -F
当正常输出的请求不被响应时，强制输出线程堆栈
- -l
除堆栈外，显示关于锁的附加信息
- -m
如果调用到本地方法的话，可以显示C/C++的堆栈





