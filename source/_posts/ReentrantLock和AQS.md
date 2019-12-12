---
title: ReentrantLock和AQS
date: 2019-10-28 00:37:37
categories: Java
tags:
- JUC
- 多线程
---

本文记录学习ReentrantLock和JUC下的一些控制线程安全的工具类，以及从ReentrantLock的实现理解AQS的原理。

<!--more-->

## ReentrantLock的简介
重入锁是另一种线程安全的实现手段，与Synchronized关键字相比，在JDK1.5之前，重复锁的性能远远优于Synchronized关键字，但是从JDK1.6开始，对Synchronized关键字做了大量的优化，使得两者性能差距不大。但是重入锁有这显示的操作过程，开发人员必须手动指定何时加锁，何时释放锁。重入锁的重入体现在这种锁是可以重复进入的，当然这是局限在一个线程中，也就是说当一个线程获取这个锁，这个线程就可以重入进入锁:
```
ReentrantLock lock = new ReentrantLock();
try {
    lock.lock();
    lock.lock();
} finally {
    lock.unlock();
    lock.unlock();
}
```
一个线程连续两次获得同一把锁时允许的，不然第二次获取的时候，就会出现自己把自己锁死的情况。但是一个线程多次获取锁，在释放锁的时候，就需要释放对应的次数。否则的话，其它线程就无法获取锁。Synchronized关键字也是支持可重入的锁，不过它是通过monitorenter和monitorexit指令控制锁的进入和释放的。相比Synchronized关键字，ReentrantLock提供了一些高级的操作，如可中断的获取锁、可以指定锁的公平性、有时间限制的获取锁等。下面罗列一些常用的ReentrantLock的方法:
```java
lock.lock(); // 获取锁，如果没有获取成功，则进入等待
lock.unlock(); // 释放锁，需要事先获取到锁
lock.lockInterruptibly(); // 可中断的获取锁，需要接收一个中断通知
boolean result= lock.tryLock(); // 尝试获取锁，获取成功返回true，失败立即返回false，不进行等待
boolean waitReuslt = lock.tryLock(1000, TimeUnit.MILLISECONDS); // 带时间的尝试，获取失败，等待指定时间

lock.isLocked(); // 判断该锁失败是锁定状态，state!=0
lock.isHeldByCurrentThread(); // 判断锁是否被当前线程持有 exclusiveOwnerThread == Thread.currentThread
lock.getHoldCount(); // 锁进入的次数，获取的是state的值，每进入一次，state+1
boolean fair = lock.isFair(); // 是否为公平锁
```

## ReentrantLock的内部实现
ReentrantLock内部通过Sync实现，Sync继承AbstractQueuedSynchronizer(简称AQS)。ReentrantLock的所有方法实现都是调用Sync的方法，Sync是一个抽象类，它有两个子类，NonfairSync和FairSync，前者是非公平锁的实现，后者是公平锁的实现。底层是通过AQS实现。
![重入锁相关类图](重入锁相关类图.png)

下面主要通过公平锁和非公平锁的lock()和unlock()方法去学习ReentrantLock的实现。

## 公平锁
ReentrantLock默认是不具有公平性的，可以通过
```java
ReentrantLock fairLock = new ReentrantLock(true); // 默认为非公平锁，true表示为公平锁
```
指定公平性。公平锁需要系统维护一个有序队列，因此公平锁实现成本高，性能且比较低下。默认情况下，锁时非公平的。下面是一个公平锁的简单例子:
```java
private static void fairLock() throws InterruptedException {
    FairLockTask fairLockTask = new FairLockTask();
    Thread t1 = new Thread(fairLockTask);
    Thread t2 = new Thread(fairLockTask);
    t1.start();
    t2.start();
    t1.join();
    t2.join();
}
public static class FairLockTask implements Runnable {

    @Override
    public void run() {
        for (int i = 0; i < 10; i++) {
            try {
                fairLock.lock();
                System.out.println(Thread.currentThread().getName() + "获取到锁");
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                fairLock.unlock();
            }
        }
    }
}
```
输出：
```
Thread-0获取到锁
Thread-1获取到锁
Thread-0获取到锁
Thread-1获取到锁
Thread-0获取到锁
Thread-1获取到锁
Thread-0获取到锁
Thread-1获取到锁
Thread-0获取到锁
Thread-1获取到锁
Thread-0获取到锁
Thread-1获取到锁
.....
```
可以看出，两个线程交替的获取锁，显示其公平性。

### 公平锁的lock()实现
在指定ReentrantLock是公平锁时，其内部的Sync的实现为FairSync。ReentrantLock的lock方法，最终调用AQS的acquire()方法实现:
```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```
如果tryAcquire(获取锁)失败，则将改线程放入等待队列。回到FairSync，看tryAcquire()方法的实现:
```java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState(); // ①
    if (c == 0) {
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) { // ②
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) { // ③
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```
①:获取获取锁的状态。
②:如果锁的状态值为0（表示为锁定状态），这里分为两步，首先判断等待队列是否已经有等待的线程，调用hasQueuedPredecessors()方法实现，如果存在等待的线程，则返回false，获取失败；否则通过CAS操作，将锁的状态值设置为1，如果成功设置锁的独占线程为当前线程，返回true。
③:如果锁的状态值不为0（表示已有线程获取到锁），判断持有该锁的线程是否为当前线程，如果是，设置锁状态值自增1，返回true；否则返回false。

第二步中判断是否已有线程在等待:
```java
public final boolean hasQueuedPredecessors() {
    // The correctness of this depends on head being initialized
    // before tail and on head.next being accurate if the current
    // thread is first in queue.
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```
该方法返回true的场景:
- h != t && (s = h.next) == null: head已经指向了头节点，而tail还为null。这是因为当一个线程获取锁失败后，需要加入到队列，如果队列是第一次初始化，会创建一个空节点为头节点，然后通过CAS操作，设置head指向该空节点，这里就可能存在还未将tail指向此结点。这就说明，有其它线程正在初始化队列（比当前线程来的早），这里就要返回true。
- h != t && (s = h.next) != null & s.thread != Thread.currentThread(): 这说明队列里存在有效节点，但是有效节点的持有线程非当前线程，所有当前线程获取锁失败。返回true。

等待队列的初始化在第一次出现获取锁失败后，进行入队创建。

回到acquire()方法，如果锁获取失败，将线程入队。`acquireQueued(addWaiter(Node.EXCLUSIVE), arg))`包含两步操作，首先将当前线程入队，然后继续尝试获取锁，这次的获取是在等待队列不断地尝试。先来看看addWaiter()方法的实现:
```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode); // ①
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail; // ②
    if (pred != null) { 
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node); // ③
    return node;
}
private Node enq(final Node node) {
    for (;;) { // ④
        Node t = tail;
        if (t == null) { // Must initialize // ⑤
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) { // ⑥
                t.next = node;
                return t;
            }
        }
    }
}
```
①:根据当前线程和锁模式创建一个新的Node节点
②:pred指向tail尾结点指针，如果pred不为null，表示当前队列已经有数据了。设置新的Node节点的前驱节点为pred，然后通过CAS操作设置尾结点为新Node节点。因为等待队列是双向队列，所以需要设置pred的后继节点为新Node节点。
③:如果pred为null，说明队列中没有元素。或者第二步的CAS操作失败(提前被其它线程设置了)。走enq()方法逻辑。
④:自旋操作
⑤:t指向tail尾结点的指针。再次判断尾结点是否为空（因为可能会被其它线程创建），说明没有元素。则进行初始化队列。然后通过CAS操作设置头节点，这里的头节点是一个空的Node节点，并没有线程信息和锁模式信息。如果CAS操作失败，不管本次CAS操作是否失败，都会进行下一次自旋。再进行下一次自旋的时候，此时t不会为空，因为不管CAS成功与否，都会创建新的空Node为头节点，且tail也执行了头节点指针。
⑥:通过第⑤不的不断自旋，等待队列被初始化完成。设置新加入的节点Node的前驱节点为t（尾结点），然后通过CAS操作更新尾结点为新节点Node，如果CAS操作失败（被其它线程提前更新），进行下一次自旋。最后在设置原尾结点的后继节点为新节点Node

通过addWaiter()方法，将当前线程加入到等待队列，并返回当前线程所在节点Node。接下来执行acquireQueued()方法为这个节点不断尝试获取锁。
```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true; // 是否获取失败
    try {
        boolean interrupted = false; // 是否被中断
        for (;;) {
            final Node p = node.predecessor(); // ①
            if (p == head && tryAcquire(arg)) { // ②
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt()) // ③
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
该方法设置了两个标识，failed表示获取锁失败，interrupted表示线程是否被中断。方法的主体还是一个自旋操作，通过自旋不断的尝试进行获取锁。
①:获取当前节点的前驱节点，设置p指向其前驱结点指针
②:如果p为头结点（头结点是一个空节点），说明当前轮到了当前节点获取锁。然后执行tryAcquire()方法尝试获取锁。如果成功了，设置头结点为当前节点Node，setNode()方法执行的操作:现将head执行当前节点Node的直接，然后设置其前驱节点和线程信息为null(表示为一个空Node)。然后将原头结点的后继节点置为null。设置failed标识为false（表示获取成功）。然后返回interrupted(false)，则不会执行acquire()方法if条件成立后的selfInterrupt()方法。
③:如果当前节点的前驱节点p不是头结点，或者是再次获取锁失败，然后判断当前线程是否需要挂起，如果需要挂起当前线程，则执行parkAndCheckInterrupt()方法。

判断当前线程是否需要挂起的方法是shouldParkAfterFailedAcquire，判断依据是根据当前Node的前驱节点的waitStatus值:
```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus; // 获取前驱结点的状态
    if (ws == Node.SIGNAL) // 前驱结点处于唤醒状态 
        /*
        如果前驱节点的状态为SIGNAL，这意味着当前节点Node可以安心的挂起，
        因为其前驱节点获取到锁后会将它唤醒。此时返回true，将当前线程挂起。
        */
        return true; 
    if (ws > 0) { // 通过Node的枚举值可知，只有cancelled状态值才会大于0
        /* 如果前驱结点的状态为cancelled，则从当前node节点往前找，
        找到第一个waitStatus不为cancelled的节点，设置该节点为node节点的前驱节点。
        然后返回false，表示先不挂起当前线程，回到acquireQueued方法，再次自旋，尝试获取锁。
        */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /* 这里表示当前节点需要接收一个信号，将node节点的waitStatus值通过
        CAS设置为SIGNAL，然后先不急于挂起当前线程，而是返回false，
        回到回到acquireQueued方法，再次自旋，尝试获取锁
        */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```
从上面的分析可知，只有当当前节点Node的前驱节点的状态为SIGNAL时，才会去将当前线程挂起。parkAndCheckInterrupt()方法通过调用LockSupport.park()方法将当前线程挂起，直到有线程释放锁，得到一个通知，然后唤醒该线程，再次去获取锁。

在shouldParkAfterFailedAcquire中判断前驱节点是否cancelled状态，是什么时候将节点状态设置为cancelled的呢？回到acquireQueued()方法，如果终止获取锁失败，进入最后的finally代码块，执行cancelledAcquire逻辑。
```java
private void cancelAcquire(Node node) {
    // 忽略空节点
    if (node == null)
        return;
    // 设置当前节点不关联任何线程
    node.thread = null;

    /* 跳过所有状态为cancelled的节点，找出第一个waitState不为cancelled，
    设置当前node的pre为这个节点
    */
    Node pred = node.prev;
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;

    Node predNext = pred.next; // 获取过滤之后的前驱结点的后继节点

    node.waitStatus = Node.CANCELLED; // 将当前node的waitStatus设置为cancelled

    // 如果当前节点为尾结点，设置当前节点的前驱节点为尾结点，并设置新的尾结点的next为null
    if (node == tail && compareAndSetTail(node, pred)) {
        compareAndSetNext(pred, predNext, null);
    } else {  // 如果当前节点不是尾结点，
        int ws;
        /*
        如果当前节点不是头结点的后继节点（pred != head)并且以下两个条件满足其中之一:
            1.判断当前节点的前驱节点的状态是否为SIGNAL
            2.当前节点的前驱节点的状态不为cancelled，且可以通过CAS操作将此次节点的状态值更新为SIGNAL，
        且当前节点的前驱节点关联的线程不为null。

        满足上面的条件，则把当前节点的前驱节点的后继节点指向当前节点的后继节点

        */
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
             (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
                compareAndSetNext(pred, predNext, next);
        } else { // 如果当前节点是头节点的后继节点，或则上述条件不满足，则唤醒当前节点的后继节点
            unparkSuccessor(node);
        }

        node.next = node; // help GC
    }
}
```

## 非公平锁的lock()实现
ReentrantLock的默认实现是非公平的。ReentrantLock的非公平锁实现是通过其内部的NonfairSync类实现，来看下它的lock方法:
```java
final void lock() {
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}
```
首先它会直接执行CAS操作，尝试去更新锁状态值，如果成功，设置所得独占线程为当前线程。否则执行acquire()方法。该方法在公平锁中已经介绍过了，这个方法是Sync的基类AQS的方法。在acquire()方法内部，首先调用tryAcquire方法，这里调用的是NofairSync类的tryAcquire方法，具体实现是通过Sync类的nonfairTryAcquire方法。
```java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {  // @
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```
咋一看该方法和FairSync的tryAcquire()方法很相似，主要区别在于注释@那行，公平锁会先判断在当前线程之前是否已经有线程排在它前面。而非公平锁不会，它直接通过CAS操作，尝试更新锁状态值，如果成功，设置独占锁为当前线程。其它操作和公平锁一样。在tryAcquire失败之后，进入等待队列。然后在队列中尝试获取锁。

下面的流程图大致描述了整个获取锁的过程:
![lock流程](lock流程.png)

## 解锁unlock过程
ReentrantLock需要手动显示的进行lock和unlock，如果进行了lock之后，忘记执行unlock操作，或导致其它线程无法获取到锁。下面的这段代码，没有显示的进行unlock操作，就会导致整个程序一直处于阻塞状态。
```java
public static class ForgetUnlock implements Runnable {
    @Override
    public void run() {
        lock.lock();
        System.out.println(Thread.currentThread().getName() + "获取到锁");
        try {
            for (int i = 0; i < 10000; i++) {
                nums++;
            }
            Thread.currentThread().interrupt();
            System.out.println(Thread.currentThread().getName() + "中断");
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            // do nothing
        }
    }
}
public static void forgetUnlock() throws InterruptedException {
    ForgetUnlock task = new ForgetUnlock();
    Thread t1 = new Thread(task);
    Thread t2 = new Thread(task);

    t1.start();
    t2.start();

    t1.join();
    t2.join();

    System.out.println(nums);
}
```
程序会有如下输出，且无法正常的终止。
![forgetUnlock](forgetUnlock.png)

这是因为Thread-0在获取到锁之后，完成了自己的任务，但是并没有显示的调用unlock方法，即使它进行了中断。所以一般的，会在finally代码块内，显示的调用unlock()方法，无论try...catch代码块中发生了什么，确保最后需要释放锁资源。

下面看看ReentrantLock是如何释放锁资源的。调用ReentrantLock的unlock()方法，具体内部是调用了AQS的release()方法:
```java
public final boolean release(int arg) {
    if (tryRelease(arg)) { // 释放锁资源
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h); // 唤醒等待队列中的线程
        return true;
    }
    return false;
}
```
首先调用Sync的tryRelease()方法，释放锁资源。
```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases; // 锁状态值 - 1
    if (Thread.currentThread() != getExclusiveOwnerThread()) 
        // 如果当前线程不是持有锁的线程，抛异常，防止其它线程在没有获取到锁时，去释放锁
        throw new IllegalMonitorStateException();
    boolean free = false; // 标志是否释放
    if (c == 0) { // 只有到锁状态值==0，才表示锁被释放，设置free为true，独占线程为null
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c); // 更新锁状态值
    return free;
}
```
这里就说明了为什么一个线程如果多次进入同一个锁，就需要在释放锁的时候，释放同样多次，这样，锁资源才会被真正释放。

### 释放锁之后的操作
回到release()方法，在锁释放成功后，根据头节点去唤醒等待队列的其后继节点。下面看unparkSuccessor()方法:
```java
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus; // 获取node的waitStatus值
    if (ws < 0) 
        compareAndSetWaitStatus(node, ws, 0);

    // 获取node的后继节点
    Node s = node.next;
    // 如果下一个节点为null或者waitStatus值为Cancelled
    if (s == null || s.waitStatus > 0) {
        s = null;
        // 从队列尾部开始找，找到第一个waitStatus不为cancelled的节点
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    // 如果最终找的节点不为空，唤醒该节点的线程
    if (s != null)
        LockSupport.unpark(s.thread);
}
```
为什么在查找队列的第一个waitStatus值不为cancelled的节点的时候是从后往前找呢？上面说到如果线程获取锁失败，需要将线程入队(addWaiter()方法)。
```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
```
因为入队操作不是原子的，整个过程分为三步：①将新Node节点的前驱节点指向原尾结点；②然后CAS操作更新尾结点为新Node节点；③最后将原尾结点的后继节点指向新节点Node。如果在第三步还未完成的时候，执行了unparkSuccessor()方法，那就无法从前往后查找。还有就是在执行cancellAcquire()方法的时候，产生CANCELLED节点时，先断开的也是next指针。

在调用LockSupport.unpark()方法之后，释放当前节点的线程，回到加锁失败，然后线程被挂起的地方。
```java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted(); // 获取中断标识，并清除
}
```
线程被唤醒之后，继续执行，parkAndCheckInterrupt()方法返回当前线程是否中断的标识。不管parkAndCheckInterrupt()返回是true还是false，都会再次循环尝试获取锁。如果parkAndCheckInterrupt()返回true，那么acquireQueued()方法的interrupted标识置为true。如果这次获取锁成功，则返回interrupted值。如果acquireQueued()方法返回的true，那么就会执行selfInterrupt()方法。该方法的作用是将线程中断，但是为什么获取到了锁还要将线程中断呢？原因有以下两点:
- 当中断线程被唤醒时，并不知道被唤醒的原因，可能是当前线程在等待中被中断，也可能是释放了锁以后被唤醒。因此我们通过Thread.interrupted()方法检查中断标记（该方法返回了当前线程的中断状态，并将当前线程的中断标识设置为False），并记录下来，如果发现该线程被中断过，就再中断一次。
- 线程在等待资源的过程中被唤醒，唤醒后还是会不断地去尝试获取锁，直到抢到锁为止。也就是说，在整个流程中，并不响应中断，只是记录中断记录。最后抢到锁返回了，那么如果被中断过的话，就需要补充一次中断。

## 其它获取锁的方法介绍
```java
lock.lockInterruptibly(); // 可中断的获取锁，需要接收一个中断通知
boolean result= lock.tryLock(); // 尝试获取锁，获取成功返回true，失败立即返回false，不进行等待
boolean waitReuslt = lock.tryLock(1000, TimeUnit.MILLISECONDS); // 带时间的尝试，获取失败，等待指定时间
```

