---
title: java集合理解之List
date: 2018-12-05 23:34:56
categories: Java
tags:
- Java数据结构
---
深入理解java中的集合类List，以及其主要的实现类。

<!--more-->
<div align=left>![List的主要实现类](List的主要实现.png)
如图所示，在java.util包中，ArrayList、Vector、LinkedList是对List的几个主要的实现类。三者有各自的特点：
- ArrayList: 底层数据存储是有个数组实现。对数据的插入删除效率低、查询快、线程不安全。
- Vector: 线层安全的list实现。
- LinkedList: 数据存储使用链表实现。数据的插入删除效率高，查询慢、线程不安全。

接下来逐个分析其实现原理。

## ArrayList的实现
ArrayList基于动态数组存储元素，本身不是线程安全的。针对数据的插入删除，效率不高。在对数据的插入删除操作涉及数组容器的变化，ArrayList每次扩量每次增加50%。ArrayList的元素可重复，允许null值，且是顺序的。

### ArrayList添加元素
ArrayList提供了两个添加元素的方法，boolean add(E e)和void add(int index, E element)。前者新加的元素直接放在数组尾部，后者新加的元素放在给定的index，如果该index之后还有元素，其它元素往后移。

```java
 public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}
```
该方法首先确定是否需要进行数组扩容，扩容机制是在原有的size基础上扩容50%，如果原有数组为空数组，则默认数组容量为10；扩容之后，size自增1，在对应的位置放入新增元素。
```java
public void add(int index, E element) {
    rangeCheckForAdd(index);

    ensureCapacityInternal(size + 1);  // Increments modCount!!
    System.arraycopy(elementData, index, elementData, index + 1,
                     size - index);
    elementData[index] = element;
    size++;
}
```
该方法首先检测传入的index是否合法，判断条件是`index > size || index < 0`，如果成立则会抛出IndexOutOfBoundsException异常；再执行扩容操作；`System.arraycopy(elementData, index, elementData, index + 1,size - index);`对原数组进行复制，产生的新数组在将改index后的数据（包括原来该index的数据）往后移动，index的位置先用原来的数据占位，然后element[index] = element，替换成目标值，最后进行size+1。
![add_index](add_index.png)

### ArrayList的获取元素
ArrayList提供了一个get方法实现，该get方法接收一个int类型的下标，返回该下标的元素。
```java
public E get(int index) {
    rangeCheck(index);
    return elementData(index);
}
```
首先检测index的下标值是否合法，然后返回数组对应下标值。因为ArrayList存储由数组实现，所有查询很快，时间复杂度为O(1);

### modCount 和 ConcurrentModificationException
对集合进行循环时，然后在循环体内对集合进行新增或删除元素，则有可能发生ConcurrentModificationException。看下面代码:
```java
public void test1() {
    List<Integer> nums = new ArrayList<>();
    for (int i = 0; i < 10; i++) {
        nums.add(i);
    }

    for (int idx = 0; idx < nums.size(); idx++) {
        nums.remove(nums.get(idx));
    }
}
public void test2() {
    List<Integer> nums = new ArrayList<>();
    for (int i = 0; i < 10; i++) {
        nums.add(i);
    }
    for (Integer i : nums) {
        nums.remove(i);
    }
}
public void test3() {
    List<Integer> nums = new ArrayList<>();
    for (int i = 0; i < 10; i++) {
        nums.add(i);
    }
    Iterator it = nums.iterator();
    while (it.hasNext()) {
        Integer i = (Integer) it.next();
        nums.remove(i);
    }
}
```
执行test1方法，能正常完成，但是nums中的元素并不能全部被移除；test2方法执行会发生ConcurrentModificationException。`for(Integer i : nums)`这种写法是java中语法糖，经过编译之后，生成的其实是Iterator迭代器类；test3方法和test2效果相同。

nums.iterator()返回的是一个Itr对象，Itr实现了Iterator接口。
```java
private class Itr implements Iterator<E> {
    int cursor; // 返回元素的下标
    int lastRet = -1; // 最后一个元素的下标，如果没有返回-1
    int expectedModCount = modCount; // 期望的modCount值，初始为ArrayList的modCount值
}
```
下面看下它的next方法
```java
public E next() {
    checkForComodification();
    int i = cursor;
    if (i >= size)
        throw new NoSuchElementException();
    Object[] elementData = ArrayList.this.elementData;
    if (i >= elementData.length)
        throw new ConcurrentModificationException();
    cursor = i + 1;
    return (E) elementData[lastRet = i];
}
```
首先会对ArrayList的modCount值的判断，如果modCount!=expectedModcount值，这会抛出ConcurrentModificationException。这里就是上面test2和test3方法的问题所在: 当list使用迭代器进行循环迭代时，如果在循环体内执行了list.remove(o)操作，ArrayList的remove方法会对modCount值进行+1，这个时候迭代器（Itr类）中的expectModCount值并没有更新，然后迭代器再次去下一个元素时，就会发现modCount!=expectedModcount。

那么针对test2和test3情况如果解决呢？既然已经通过list获取了它的迭代器，那么就是用它的迭代器进行元素删除即可。
```java
public void remove() {
    if (lastRet < 0)
        throw new IllegalStateException();
    checkForComodification();

    try {
        ArrayList.this.remove(lastRet);
        cursor = lastRet;
        lastRet = -1;
        expectedModCount = modCount;
    } catch (IndexOutOfBoundsException ex) {
        throw new ConcurrentModificationException();
    }
}
```
在try...catch体内，首先内部类调用了其外部类（ArrayList）的remove方法，进行元素删除，这是modCount值+1，随后将变化了的modCount值赋给了expectModCount。这样的话，在执行next方法判断modCount!=expectedModcount时就不会抛出异常。

## Vector类的实现
Vector和ArrayList类似，底层存储也是使用数据实现、初始化容量大小为10、扩容默认为当前的一倍、线程安全。它之所以能做到线程安全，是因为所有的方法都是synchronized修饰的。Vector对集合元素的操作和ArrayList基本相同，所以就不做过多的了解。

## LinkedList实现
LinkedList底层存储基于链表实现，链表的特点就是每个节点相互连接，非头结点和尾结点都有其前驱节点后后继节点，这在存储上变现为持有前驱节点和后继节点的指针，在对链表进行插入和删除元素操作时，只需要改变对应的前驱结点和后继节点的指向即可，而且没有长度限制，所以针对插入删除操作表ArrayList快。但是在进行遍历时，就需要从头结点或尾结点逐个查找，无法通过下标直接定位，所有在查询的场景下没有ArrayList快。

```java
transient int size = 0; // LinkedList的长度
transient Node<E> first; // 头结点
transient Node<E> last; // 尾结点

private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```
Node是LinkedList的内部类，它持有当前元素以及前驱节点和后继节点的引用，这方便访问其上一个和下一个元素。

### LinkedList的添加元素
LinkedList提供了以下几个方法添加一个元素:
- add(int index, E e); // 向指定的下标新增一个元素
- add(E e); // 向链表尾部新增一个元素
- addFirst(E e); // 在链表头部新增一个元素
- addLast(E e);  // 在链表尾部新一个元素

#### 向尾部新增一个元素
add(E e)方法默认在链表的尾部新增一个元素，和addLast(E e)方法一样，其实其内部实现也一致，都是调用了linkLast方法，向尾部追加一个元素。
```java
void linkLast(E e) {
    final Node<E> l = last;
    final Node<E> newNode = new Node<>(l, e, null);
    last = newNode;
    if (l == null)
        first = newNode;
    else
        l.next = newNode;
    size++;
    modCount++;
}
```
首先将原尾结点赋值给l； 然后新增一个Node，并设置前驱节点为l,后继节点为null,e为当前元素；将LinkedList的last设置为新Node；如果原尾结点l为null，则说明本次是第一次新增元素，将LinkedList的头结点设置为新增节点，否者的话，设置原尾结点l的next节点为新增节点。

#### 向头部新增一个元素
向LinkedList头节点新增一个元素，内部调用linkFirst方法实现。
```java
private void linkFirst(E e) {
    final Node<E> f = first;
    final Node<E> newNode = new Node<>(null, e, f);
    first = newNode;
    if (f == null)
        last = newNode;
    else
        f.prev = newNode;
    size++;
    modCount++;
}
```
首先将原头节点赋值为f；然后新增一个Node，并设置前驱接点为null，后继节点为f，e为当前元素；将LinkedList的first设置为新Node；如果原头结点f为null，则说明是第一个新增元素，设置LinkedList的尾结点为新增节点，否者的话，设置原头节点的prev节点为新节点。

#### 向指定下标新增元素
```java
public void add(int index, E element) {
    checkPositionIndex(index);

    if (index == size)
        linkLast(element);
    else
        linkBefore(element, node(index));
}
```
首先会检查下标的合法性，如果非法则会抛出下标越界异常；然后判断如果传过的index等于链表长度，则直接在链表尾部新增一个节点，否则的话，先调用node方法，获取该index的前一个节点，然后调用linkBefore方法插入数据。
```java
Node<E> node(int index) {
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```
该方法是获取指定index的节点，这个方法做了优化，`size>>1`向右移一位，得到size的一半，然后判断index是链表的前半部分还是后半部分，这样的话，只需要遍历链表的一半获取该index的节点，然后将需要新增的元素和当前index的节点传入linkBefore方法。
接下来看下linkBefore的实现。
```java
void linkBefore(E e, Node<E> succ) {
    // assert succ != null;
    final Node<E> pred = succ.prev;
    final Node<E> newNode = new Node<>(pred, e, succ);
    succ.prev = newNode;
    if (pred == null)
        first = newNode;
    else
        pred.next = newNode;
    size++;
    modCount++;
}
```
首先获取指定index节点的上一个节点，赋值给pred；然后新增一个Node，设置前驱节点为pred，后继节点为index的节点，元素为e；然后设置index的节点的前驱节点为新Node；如果index的节点的前驱节点为null，那么这个节点原来的头节点，然后设置头节点为新Node；否则设置index的节点的前驱节点的下一个节点为节点。

### LinkedList获取一个元素
LinkedList提供了以下几个方法获取一个元素:
- get(int index); // 获取指定下标的元素
- getFirst(); // 获取头节点元素
- getLast(); // 获取尾结点元素

get(int index)方法首先判断index的合法性，然后遍历节点，遍历节点调用的node(int index)方法，node方法在上面已经分析过了，这里不在赘述。getFirst()和getLast()见名思意了，直接返回头节点和尾结点的元素，如果头节点和尾结点为null，则抛出NoSuchElementException。

### LinkedList的其它方法了解
- E peek(): 返回链表的头节点元素，但是不移除；如果链表为kong，返回null；
- E poll(): 返回链表的头节点元素，并且移除；如果链表weikong，返回null；
- E remove(): 返回链表头节点元素，并移除；如果链表为空，抛出NoSuchElementException；
- boolean offer(E e): 在链表尾部添加一个指定元素；
- boolean offerFirst(E e): 在链表的头部添加一个指定的元素；
- boolean offerLast(E e ): 在链表的尾部添加一个指定的元素；
... 还有其它很多类似peek，poll，push的方法，大同小异。

