[TOC]

# Java基础总结二（集合）

## 集合概述

### 什么是集合

* 集合框架：集合就是存储数据的容器。集合框架是为表示和操作集合而规定的一种统一的标准的体系结构。
  任何集合框架都包含三大块内容：对外的接口、接口的实现和对集合运算的算法。
* 接口：表示集合的抽象数据类型。接口允许我们操作集合时不必关注具体实现，从而达到“多态”。在面向对象编程语言中，接口通常用来形成规范。
* 实现：集合接口的具体实现，是重用性很高的数据结构。
* 算法：在一个实现了某个集合框架中的接口的对象身上完成某种有用的计算的方法，例如查找、排序等。这些算法通常是多态的，因为相同的方法可以在同一个接口被多个类实现时有不同的表现。事实上，算法是可复用的函数。

集合框架通过提供有用的数据结构和算法使你能集中注意力于你的程序的重要部分上，而不是为了让程序能正常运转而将注意力于低层设计上。
通过这些在无关API之间的简易的互用性，使你免除了为改编对象或转换代码以便联合这些API而去写大量的代码。 它提高了程序速度和质量。

### 集合和数组的区别

* 数组是固定长度的，集合是可变长度的。
* 数组可以存储基本数据类型，也可以存储引用数据类型。
* 数组存储的元素必须是同一个数据类型；集合存储的对象可以是不同数据类型。

### 使用集合框架的好处

1. 容量自增长。
2. 提供了高性能的数据结构和算法，使编码更轻松，提高程序速度和质量。
3. 可以方便的扩展或改写集合，提高代码的复用性和操作性。

### 常见的集合类有哪些

Map接口和Collection接口是所有集合框架的父接口：

1. Collection接口的子接口包括：Set接口和List接口。
2. Map接口的主要实现类有：HashMap、TreeMap、HashTable、ConcurrentHashMap、Properties等。
3. Set接口的主要实现类有：HashSet、TreeSet、LinkedHashSet等。
4. List接口的主要实现类有：ArrayList、LinkedList、Stack、Vector等。

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pbWcyMDE4LmNuYmxvZ3MuY29tL290aGVyLzE0MDgxODMvMjAxOTExLzE0MDgxODMtMjAxOTExMTkxODQxNDk1NTktMTU3MTU5NTY2OC5qcGc?x-oss-process=image/format,png)

### 简述List、Set、Map三者区别

* Map不是Collection的子接口，List和Set是Collection的子接口。
* List：一个有序的容器（元素的存入集合的顺序和取出的顺序一致），元素可以重复，可以插入多个null元素，常用实现类有ArrayList、LinkedList、Vector。
* Set：一个无序的容器，不可以存储重复的元素，只允许存入一个null元素，必须保证元素的唯一性。Set 接口常用实现类是 HashSet、LinkedHashSet 以及 TreeSet。
* Map：一个存储键值对的容器。key无序且唯一，value允许重复。Map 的常用实现类：HashMap、TreeMap、HashTable、LinkedHashMap、ConcurrentHashMap。

### 集合框架底层数据结构

* List
  * ArrayList：数组。
  * Vector：数组。
  * LinkedList：双向循环链表。

* Set
  * HashSet（无序、唯一）：基于HashMap实现，底层采用HashMap来保存元素。
  * LinkedHashSet ：LinkedHashSet继承与HashSet，并且内部是通过LinkedHashMap来实现的。
  * TreeSet（有序、唯一）：红黑树（自平衡排序二叉树）。

* Map
  * HashMap：JDK1.8之前是数组+链表组成的，数组是HashMap的主体，链表这是为了解决哈希冲突而存在。JDK1.8之后再解决哈希冲突时有了较大的变化，当链表大于阈值（默认为8），并且数组长度大于64，会将链表转化为红黑树，来减少搜索时间。如果数组小于64，则会先扩容。
  * LinkedHashMap ：LinkedHashMap 继承自 HashMap，所以它的底层仍然是基于拉链式散列结构即由数组和链表或红黑树组成。另外，LinkedHashMap 在上面结构的基础上，增加了一条双向链表，使得上面的结构可以保持键值对的插入顺序。同时通过对链表进行相应的操作，实现了访问顺序相关逻辑。
  * TreeMap： 红黑树（自平衡的排序二叉树）。

### 哪些集合类是线程安全的

* Vector：就比ArrayList多了同步化机制，因为效率低，现在已经不太建议使用。
* Stack：堆栈类，先进后出。
* HashTable：比HashMap多了线程安全。

### 集合的快速失败机制 “fail-fast”

是java集合的一种错误检测机制，当多个线程对集合进行结构上的改变的操作时，有可能会产生 fail-fast 机制。

例如：假设存在两个线程（线程1、线程2），线程1通过Iterator在遍历集合A中的元素，在某个时候线程2修改了集合A的结构（是结构上面的修改，而不是简单的修改集合元素的内容），那么这个时候程序就会抛出 ConcurrentModificationException 异常，从而产生fail-fast机制。

原因：迭代器在遍历时直接访问集合中的内容，并且在遍历过程中使用一个 modCount 变量。集合在被遍历期间如果内容发生变化，就会改变modCount的值。每当迭代器使用hashNext()/next()遍历下一个元素之前，都会检测modCount变量是否为expectedmodCount值，是的话就返回遍历；否则抛出异常，终止遍历。

解决办法：

1. 使用并发包中的容器，如CopyOnWriteArrayList来替换ArrayList。
2. 使用迭代器中的方法。

## Collection接口

### List接口

#### 迭代器Iterator是什么？

Iterator 接口提供遍历任何 Collection 的接口。我们可以从一个 Collection 中使用迭代器方法来获取迭代器实例。迭代器取代了 Java 集合框架中的 Enumeration，迭代器允许调用者在迭代过程中移除元素。

#### Iterator怎么使用？有什么特点？

Iterator 使用代码如下：

```java
List<String> list = new ArrayList<>();
Iterator<String> it = list. iterator();
while(it. hasNext()){
  String obj = it. next();
  System. out. println(obj);
}
```

Iterator 的特点是只能单向遍历，但是更加安全，因为它可以确保，在当前遍历的集合元素被更改的时候，就会抛出 ConcurrentModificationException 异常。

#### 如何边遍历边移除 Collection 中的元素？

```java
Iterator<Integer> it = list.iterator();
while(it.hasNext()){
   *// do something*
   it.remove();
}
```

一种最常见的**错误**代码如下：

```java
for(Integer i : list){
   list.remove(i)
}
```

#### ArrayList的优缺点

* 优点：ArrayList底层以数组实现，是一种随机访问模式，查找的时候非常快。ArrayList在顺序添加一个元素时候非常方便。
* 缺点：插入和删除元素时需要做复制操作，如果元素很多，那么就比较耗费性能。

#### ArrayList和LinkedList的区别

* 数据结构实现：ArrayList是动态数组数据结构实现，而LinkedList是双向链表的数据结构实现。
* 访问效率：ArrayList通过索引访问效率要高，LinkedList是线性的数据存储方式，所以需要移动指针从前往后依次查找。
* 增加和删除效率：在非尾部的增加和删除操作，LinkedList要比ArrayList效率要高。
* 内存空间占用：LinkedList的元素要更占内存，因为LinkedList的节点除了存储数据，还要存储两个引用，分别指向前一个元素和后一个元素。
* 线程安全：都不保证线程安全。

#### ArrayList 和 Vector 的区别是什么？

* 线程安全：Vector使用Synchronized实现线程同步，是线程安全的，而ArrayList是非线程安全的。
* 性能：ArrayList的性能要优于Vector。
* 扩容：ArrayList 和 Vector 都会根据实际的需要动态的调整容量，只不过在 Vector 扩容每次会增加 1 倍，而 ArrayList 只会增加 50%。

Vector类的所有方法都是同步的。可以由两个线程安全地访问一个Vector对象、但是一个线程访问Vector的话代码要在同步操作上耗费大量的时间。

ArrayList不是同步的，所以在不需要保证线程安全时时建议使用ArrayList。

#### 多线程场景下如何使用 ArrayList？

```java
List<String> synchronizedList = Collections.synchronizedList(list);
synchronizedList.add("aaa");
synchronizedList.add("bbb");

for (int i = 0; i < synchronizedList.size(); i++) {
    System.out.println(synchronizedList.get(i));
}
```

### Set接口

#### HashSet的实现原理

HashSet 是基于 HashMap 实现的，HashSet的值存放于HashMap的key上，HashMap的value统一为PRESENT（Object对象），因此 HashSet 的实现比较简单，相关 HashSet 的操作，基本上都是直接调用底层 HashMap 的相关方法来完成，HashSet 不允许重复的值。

#### HashSet与HashMap的区别

| HashMap                                                | **HashSet**                                                  |
| ------------------------------------------------------ | ------------------------------------------------------------ |
| 实现了Map接口                                          | 实现Set接口                                                  |
| 存储键值对                                             | 仅存储对象                                                   |
| 调用put（）向map中添加元素                             | 调用add（）方法向Set中添加元素                               |
| HashMap使用键（Key）计算Hashcode                       | HashSet使用成员对象来计算hashcode值，对于两个对象来说hashcode可能相同，所以equals()方法用来判断对象的相等性，如果两个对象不同的话，那么返回false |
| HashMap相对于HashSet较快，因为它是使用唯一的键获取对象 | HashSet较HashMap来说比较慢                                   |

### Queue

#### BlockingQueue是什么？

Java.util.concurrent.BlockingQueue是一个队列，在进行检索或移除一个元素的时候，它会等待队列变为非空；当在添加一个元素时，它会等待队列中的可用空间。BlockingQueue接口是Java集合框架的一部分，主要用于实现生产者-消费者模式。我们不需要担心等待生产者有可用的空间，或消费者有可用的对象，因为它都在BlockingQueue的实现类中被处理了。Java提供了集中BlockingQueue的实现，比如ArrayBlockingQueue、LinkedBlockingQueue、PriorityBlockingQueue,、SynchronousQueue等。

#### 在 Queue 中 poll()和 remove()有什么区别？

* 相同点：都是返回第一个元素，并在队列中删除返回的对象。
* 不同点：如果没有元素poll()会返回null，而remove()会直接抛出NoSuchElementException异常。

## Map

### HashMap的实现原理

HashMap概述：HashMap是基于哈希表的Map接口非同步实现。此实现提供了所有可选的映射操作，并允许使用null值和null键。此类不保证映射的顺序。

HashMap的数据结构：链表散列，即数组和链表的结合体（JDK1.8之后加入红黑树）。

HashMap基于Hash算法实现的：

1. 当我们往HashMap中put元素时，利用key的hashCode重新hash计算出当前对象元素在数组中的下标。
2. 存储时，如果出现hash值相同的key，此时有两种情况
   1. 如果key相同，则覆盖原始值。
   2. 如果key不同（出现hash冲突），则将当前节点放入链表中。
3. 获取时，直接找到hash对应的下表，在比较key是否相同，从而找到对应的值。

需要注意JDK1.8中对HashMap的实现做了优化，当链表中的节点数据超过八个之后，该链表会转为红黑树来提高查询效率，从原来的O(n)到O(logn)

### HashMap在JDK1.7和JDK1.8中有哪些不同

| **不同**                 | **JDK 1.7**                                                  | JDK 1.8                                                      |
| ------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 存储结构                 | 数组 + 链表                                                  | 数组 + 链表 + 红黑树                                         |
| 初始化方式               | 单独函数：`inflateTable()`                                   | 直接集成到了扩容函数`resize()`中                             |
| hash值计算方式           | 扰动处理 = 9次扰动 = 4次位运算 + 5次异或运算                 | 扰动处理 = 2次扰动 = 1次位运算 + 1次异或运算                 |
| 存放数据的规则           | 无冲突时，存放数组；冲突时，存放链表                         | 无冲突时，存放数组；冲突 & 链表长度 < 8：存放单链表；冲突 & 链表长度 > 8：树化并存放红黑树 |
| 插入数据方式             | 头插法（先讲原位置的数据移到后1位，再插入数据到该位置）      | 尾插法（直接插入到链表尾部/红黑树）                          |
| 扩容后存储位置的计算方式 | 全部按照原来方法进行计算（即hashCode ->> 扰动函数 ->> (h&length-1)） | 按照扩容后的规律计算（即扩容后的位置=原位置 or 原位置 + 旧容量） |

具体HashMap源码我们分另一篇文章来总结。

### HashMap与HashTable有什么区别？

1. 线程安全：HashMap是非线程安全的，而HashTable是线程安全的；HashTable 内部的方法基本都经过 `synchronized` 修饰。
2. 性能：因为线程安全的问题，HashMap 要比 HashTable 效率高一点。另外，HashTable 基本被淘汰，不要在代码中使用它；
3. 对Null key 和Null value的支持：HashMap 中，null 可以作为键，这样的键只有一个，可以有一个或多个键所对应的值为 null。但是在 HashTable 中 put 进的键值只要有一个 null，直接抛NullPointerException。
4. 初始容量大小和每次扩充容量大小的不同：①创建时如果不指定容量初始值，HashTable 默认的初始大小为11，之后每次扩充，容量变为原来的2n+1。HashMap 默认的初始化大小为16。之后每次扩充，容量变为原来的2倍。②创建时如果给定了容量初始值，那么 HashTable 会直接使用你给定的大小，而 HashMap 会将其扩充为2的幂次方大小。
5. 底层数据结构： JDK1.8 以后的 HashMap 在解决哈希冲突时有了较大的变化，当链表长度大于阈值（默认为8）时，将链表转化为红黑树，以减少搜索时间。HashTable 没有这样的机制。

### HashMap和ConcurrentHashMap的区别

1. ConcurrentHashMap对整个数组进行了分割分段（Segment），然后在每一个分段上都用lock锁进行保护，相对于HashTable的synchronized锁的粒度更细，并发性更好，而HashMap没有锁机制，不是线程安全的。（JDK1.8之后ConcurrentHashMap启用了一种全新的方式实现,利用CAS算法）。

ConcurrentHashMap会单独放一篇文章来总结。

## 辅助工具类

### 如何实现 Array 和 List 之间的转换？

- Array 转 List： Arrays. asList(array) ；
- List 转 Array ：List 的 toArray() 方法。

### comparable 和 comparator的区别？

- comparable接口实际上是出自java.lang包，它有一个 compareTo(Object obj)方法用来排序。
- comparator接口实际上是出自 java.util 包，它有一个compare(Object obj1, Object obj2)方法用来排序。

### Collection 和 Collections 有什么区别？

* java.util.Collection 是一个集合接口（集合类的一个顶级接口）。它提供了对集合对象进行基本操作的通用接口方法。Collection接口在Java 类库中有很多具体的实现。Collection接口的意义是为各种具体的集合提供了最大化的统一操作方式，其直接继承接口有List与Set。
* Collections则是集合类的一个工具类/帮助类，其中提供了一系列静态方法，用于对集合中元素进行排序、搜索以及线程安全等各种操作。

### TreeMap 和 TreeSet 在排序时如何比较元素？

TreeSet 要求存放的对象所属的类必须实现 Comparable 接口，该接口提供了比较元素的 compareTo()方法，当插入元素时会回调该方法比较元素的大小。TreeMap 要求存放的键值对映射的键必须实现 Comparable 接口从而根据键对元素进行排序。

