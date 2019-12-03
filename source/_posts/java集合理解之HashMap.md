---
title: java集合理解之Map
date: 2018-12-08 16:23:56
categories: Java
tags:
- Java数据结构
---

深入理解java中的集合类Map，以及其主要的实现类。

<!--more-->

Map的主要实现有HashMap、HashTable、TreeMap、LinkedHashMap等。

## HashMap的实现
在HashMap中，允许存放null键和值，元素是没有顺序的。HashMap容器的本质是一个哈希数组结构，在元素插入的时候，有存在hash冲突的可能。在HashMap中，为了解决hash冲突的问题，jdk1.7和1.8做了不同的实现。1.7中使用了数组+链表的数据结构存储，即当发生hash冲突，就将冲突的元素存放在链表中；1.8使用了数组+链表+红黑树的数据结构存储，相比1.7来说，多了红黑树的实现，当链表的长度超过8时，就将链表转换成红黑树。

接下来从以下几个方面深入学习HashMap:
- put方法
- 扩容实现
- get方法

首先看下HashMap几个成员变脸代表的含义:
- DEFAULT_INITIAL_CAPACITY= 1 << 4: 默认初始化map的容量大小=16
- MAXIMUM_CAPACITY = 1 << 30: 最大的map容量=2^30
- DEFAULT_LOAD_FACTOR = 0.75f: 默认负载因子
- TREEIFY_THRESHOLD = 8: 链表转成数的长度
- UNTREEIFY_THRESHOLD = 6: 取消树化的长度
- MIN_TREEIFY_CAPACITY = 64: 可以树化的最小容量
- Node<K,V>[] table: 节点容器，数组
- int size: 容器大小
- int modCount: 容器修改次数
- int threshold: 容器最大容量       
- float loadFactor: 负载因子

### put方法实现
```java
// 通过hash方法计算出key的hash值
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
// 通过putVal方法实现
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0) // ①
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null) // ②
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k)))) // ③
            e = p;
        else if (p instanceof TreeNode) // ④
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) { // ⑤
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash); // ⑥
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k)))) //⑦
                    break;
                p = e;
            }
        }
        // ⑧
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold) // ⑨
        resize();
    afterNodeInsertion(evict);
    return null;
}
```
①:如果table==null或者table的length为0，则进行resize扩容，n为容器的容量；
②:通过key的hash值，和n值进行位与运算，计算出新增元素的下标，然后获取下标值，赋给p；如果p为null，说明这个位置没有元素，则直接新增一个Node。
③:在②不成立的基础上，判断新插入的元素节点是否为p节点，如果是，执行⑧，对值进行覆盖
④:在③不成立的基础上，判断p节点是否是TreeNode，如果是在TreeNode实现元素新增。
⑤:在④不成立的基础上（那就是链表），循环链表元素，e为p的下一节点，如果e为null，则新增一个Node节点。然后判断链表的长度是否满足树化的条件，如果满足，执行⑥。
⑥:对链表进行树化。
⑦:在对链表进行遍历时，如果e满足hash值和key的值和新增元素相等，跳出循环执行⑧，对值进行覆盖
⑧:对存在的节点值进行覆盖。
⑨:完成新增后，如果容量大于阈值，进行扩容。

### resize方法实现
```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table; // 原node数组
    int oldCap = (oldTab == null) ? 0 : oldTab.length; // 原node数组的容量
    int oldThr = threshold; // 原node数组的扩容阈值
    int newCap, newThr = 0; // 初始化新的容量、新的扩容阈值
    if (oldCap > 0) { // ①
        if (oldCap >= MAXIMUM_CAPACITY) { ②
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY) // ③
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // ④
        newCap = oldThr;
    else {         
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) { // ⑤
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap]; // ⑥
    table = newTab;
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) { // ⑦
            Node<K,V> e;
            if ((e = oldTab[j]) != null) { // ⑧
                oldTab[j] = null;
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode) // ⑨
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // ⑩
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do { // ⑪
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) { // ⑫
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```
oldTab为原node数组、oldCap为原数组的容量、oldThr为原数组的扩容容量阈值、newCap新的容量、 newThr新的扩容阈值。

①:如果oldCap>0。执行②，否则执行④。
②:如果oldCap>=MAXIMUM_CAPACITY(2^30)，则设置扩容阈值为Integer的最大值(2^31)，返回oldCap，不进行扩容。
③:②不成立的基础上，newCap=oldCap<<1，即newCap为oldCap的两倍。如果newCap<2^30且oldCap>=16，设置newThr=oldThr<<1，即为oldThr的两倍。
④:在①不成立的基础上，如果oldThr>0，赋值newCap为oldThr。否则的话，newCap=DEFAULT_INITIAL_CAPACITY=16、newThr=DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY。
⑤:如果newThr为0，根据newCap计算出新的newThr值。最后将newThr赋值给threshold。
⑥:创建新的Node数组，数组大小为newCap。
⑦:遍历oldTab中的元素。
⑧:取出oldTab中的元素，赋值为e，e持有该节点的引用，然后将oldTable的该元素置为null。如果该节点没有next节点，说明这个节点只有一个元素（没有hash冲突），则在新数组的这个位置放入该元素即可。
⑨:如果当前节点e是个树，则进行分裂。
⑩:否则的话进行链表重hash。      
⑪:从链表的第一个元素遍历，当前元素的hash值和oldCap进行位与运算，运算结算如果是0，则将元素赋值给loTail，反之赋值给hiTail，在遍历过程中，元素都是放在尾部，然后不断更新loTail和hiTail。最终loTail和hiTail会形成一个链表（如果数据足够）。
⑫:将⑪生成的loTail放在第newTab的第j个位置（当前for循环的下标）；hiTail放在newTab的j+oldCap的位置。

第⑨步中，如果当前节点e是树，则进行树的分裂。大致的过程是: 首先遍历树的节点，根据节点的hash值和oldCap进行位与运算，运算结果如果为0，则将节点赋值给loTail，反之赋值给hiTail，和第⑪步的过程类似，但是会通过lc和hc计算两者的个数，用作判断是否进行去树化的依据。然后，以此判断lc和hc的值是否小于等于UNTREEIFY_THRESHOLD，如果满足，则对节点去树化，反之，进行树化。

untreeify去树化将所有的节点打平，用链表连接。treeify树化将链表转换为红黑树，会根据红黑树特性进行颜色转换、左旋、右旋等。

### get方法实现
通过指定的key，从hashmap中获取对应的值。
```java
// 主要是getNode方法实现
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) { // ①
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k)))) // ②
            return first;
        if ((e = first.next) != null) { // ③
            if (first instanceof TreeNode) 
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            do { 
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```
get(Object key)方法，首先算出key的hash值，然后调用getNode方法，获取对应的node节点，然后返回节点元素的值。检索的逻辑主要在getNode方法上。

①:首先判断table不为空，table的长度不为0，且头节点不为null。
②:总是先判断头节点是不是需要找的元素，判断的依据为节点的hash值和key值，是否和要查找的相等。
③:然后从头节点的下一个几点开始遍历，如果该节点是树，则调用TreeNode的getTreeNode方法查找；否则的话，依次判断本次遍历的节点是否满足条件。

### TreeNode的getNode方法实现
该方法内部调用find方法实现。

```java
final TreeNode<K,V> find(int h, Object k, Class<?> kc) {
    TreeNode<K,V> p = this;
    do {
        int ph, dir; K pk;
        TreeNode<K,V> pl = p.left, pr = p.right, q;
        if ((ph = p.hash) > h) // 判断要查询的元素的hash是否在树的左边
            p = pl;
        else if (ph < h) // 判断要查询的元素的hash是否在树的右边
            p = pr;
        else if ((pk = p.key) == k || (k != null && k.equals(pk))) // 判断当前节点的hash和查询元素相等
            return p;
        else if (pl == null)
            p = pr;
        else if (pr == null)
            p = pl;
        else if ((kc != null ||
                  (kc = comparableClassFor(k)) != null) &&
                 (dir = compareComparables(kc, k, pk)) != 0)
            p = (dir < 0) ? pl : pr;
        else if ((q = pr.find(h, k, kc)) != null) // 根据compareTo的结果在右孩子树继续插叙
            return q;
        else // 根据compareTo的结果在坐孩子树继续插叙
            p = pl;
    } while (p != null);
    return null;
}
```

## 其它集中map

### LinkedHashMap
LinkedHashMap继承了HashMap，实现了Map接口。内部使用链表数据结构来记录插入的顺序，使的元素插入和读取的顺序是相同的。和HashMap最大打不同在于，LinkedHashMap的元素是有序的。

### TreeMap
TreeMap能把保存的元素根据键排序，默认是按键值的升序排序，也可以指定排序的比较器，当Iterator遍历时，得到的元素是排序过的；

### HashTable
键值不能为null，与HashMap不同的是，方法都加了synchronized修饰，是线程安全的，但没有HashMap效率高。和HashMap区别在于，HashMap允许null的键值，HashTable不允许。HashMap非线程安全，HashTable线程安全。

但是，一般在多线程的情况下，都会使用JUC下的线程安全map---ConcurrentHashMap。即保证了线程安全，效率也优于HashTable。

