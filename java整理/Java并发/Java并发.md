# Java多线程

为什么会有并发问题？

由于Java内存模型规定了所有的变量必须存储在主内存中，每一条线程都用自己的工作内存，线程的工作内存保存了主存中变量副本的拷贝，线程对变量的操作都必须在工作内存中进行，不能直接读写在主存中。线程首先在工作内存中修改变量，然后在同步回主存。当多条线程同时操作一个变量时，一条线程操作变量尚未刷新到主存就被另一条线程修改，就会发生数据的错误。

JMM的作用？
Java内存模型作用于工作内存和主存之间数据同步过程，它规定了如何做数据同步以及什么时候做数据同步。

并发三要素？

原子性：在一个操作中，CPU不可以在中途暂停然后再调度，即不会被中断，要么执行完成，要么不执行。

可见性：多个线程访问同一个变量，一个线程修改变量的值，其他线程能够立即看见修改的值。

有序性：程序执行的顺序是按照代码的先后顺序执行的。

如何处理并发？

结合不同的场景分析问题，采取恰当的方法。

1. volatile

   保证可见性，不保证原子性。

   > 当写一个volatile变量时，JVM会把本地内存中的变量强制刷新到主存中
   >
   > 这个写操作会使其他线程中的缓存无效，其他线程读取时，会从主存中读。volatile的写操作对其他线程是可见的。

   缓存一致性协议：每个处理器通过嗅探在总线上传播的数据来检查自己的缓存是不是过期了，当处理器发现自己缓存行对应的内存地址被修改，就会将当前处理器的缓存行设置成无效状态，当处理器对这个数据进行修改操纵的时候，会重新从系统中把数据读到处理器缓存中。因此经过我们分析得出以下结论：

   Lock前缀的指令会引起处理器缓存写回内存；

   一个处理器的缓存写回到主存会导致其他处理器的缓存失效；

   当处理器发现工作内存中的数据失效后，就会从主存中重读该变量数据，即可以获取当前新值。

   禁止指令重排。

   >指令重排是指编译器和处理器为了优化程序性能对指令进行排序的一种手段，需要遵从一定的规则。

   happens-before关于volatile：

   对一个volatile域的写，happens-before于任意后续对这个volatile域的读。

   volatile内存语义的实现：

   在volatile变量的读写前后添加内存屏障

   1. 在每个volatile写操作的**前面**插入一个StoreStore屏障；
   2. 在每个volatile写操作的**后面**插入一个StoreLoad屏障；
   3. 在每个volatile读操作的**后面**插入一个LoadLoad屏障；
   4. 在每个volatile读操作的**后面**插入一个LoadStore屏障。

   **StoreStore屏障**：禁止上面的普通写和下面的volatile写重排序；

   **StoreLoad屏障**：防止上面的volatile写与下面可能有的volatile读/写重排序

   **LoadLoad屏障**：禁止下面所有的普通读操作和上面的volatile读重排序

   **LoadStore屏障**：禁止下面所有的普通写操作和上面的volatile读重排序

2. synchronized





1. final

   在Java中变量可以分为成员变量和局部变量。

   final成员变量

   通常每个类中的成员变量可以分为类变量（static修饰的变量）以及实例变量。针对这两种变量类型赋初始值的时机是不同的，类变量是在声明变量时候或者在静态代码块中赋初始值。实例变量在声明时或非静态初始化块及构造器中赋初始值。当final变量未初始化时系统不会进行隐式初始化





AbstractQueuedSynchronizer（AQS）队列同步器

同步器是用来构建锁与其他同步组件的基础框架，它的实现主要依赖一个int成员变量来表示同步状态以及通过一个FIFO队列构成等待队列。它的子类必须重写AQS的几个protected修饰的用来改变同步状态的方法，其他方法主要实现了排队和阻塞机制。

子类被推荐定义为自定义同步组件的静态内部类，同步器自身没有实现任何同步接口，



同步组件的实现者通过使用AQS提供的模板方法实现同步组件语义，AQS则实现了对同步状态的管理，以及对阻塞队列进行排队，等待通知等一些底层实现处理。AQS的核心也包括了这些方面：同步队列，独占式获取和释放，共享锁的获取和释放以及可中断锁，超时等待锁获取这些特性的实现



同步队列是AQS的基石

当共享资源被某个线程占有时，其他请求该资源的线程将会阻塞，从而进入同步队列。就数据结构而言，队列的实现方式无外乎是通过数组的形式，或者通过链表的形式。AQS中同步队列则是通过链式方式进行实现。

下面我们就来探讨AQS对独占锁的实现方式





ReentrantLock

ReentrantLock重入锁，是实现Lock接口的一个类，在实际编程中使用频率高，支持重入性，表示能够对共享资源重复加锁，即当前线程获取该锁再次获取不会被阻塞。ReentrantLock还支持公平锁和非公平锁。想要完全掌握ReentrantLock的话，就要搞清重入性的实现原理和公平锁/非公平锁。



并发容器

ConcurrentHashMap

由于HashMap不是线程安全的容器，在多线程情况下使用会发生问题，我们可以使用HashTable，该类基本上所有的方法都采用了synchronized进行线程安全控制，可想而知，在高并发情况下，虽然可以保证不会出现问题，但会大大影响程序的效率。另外一种方式通过Collections的synchronizedMap将HashMap包装成一个线程安全的map,实际上SynchronizedMap实现依然是采用Synchronized独占锁的方式进行线程安全控制。针对这种情况ConcurrentHashMap利用分段锁的思想提高了并发度。

在jdk1.8以前，ConcurrentHashMap的关键要素是segment： segment继承了ReentrantLock充当锁的角色，为每一个segment提供了线程安全的保障。segment维护了哈希散列表的若干个桶，每个桶由HashEntry构成的链表。

关键属性及类

关键属性

table：装载Node的数组，作为ConcurrentHashMap的数据结构，采用懒加载的方式，直到第一次插入数据时才会进行初始化，数组大小总为2的幂次方。

nextTable：扩容时使用，平时为null，只有在扩容时才为非null

Node：Node类实现了Map.Entry接口，主要存放key-value，并且具有next域。

TreeNode树节点，继承于承载数据的Node类，而红黑树的操作是针对TreeBin类的，从该类的注释中可以看到，也就是TreeBin会将包装TreeNode

ConcurrentHashMap的构造方法：

```java
public ConcurrentHashMap() {
}

public ConcurrentHashMap(int initialCapacity) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException();
        int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
                   MAXIMUM_CAPACITY :
                   tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
        this.sizeCtl = cap;
    }

public ConcurrentHashMap(Map<? extends K, ? extends V> m) {
        this.sizeCtl = DEFAULT_CAPACITY;
        putAll(m);
    }

public ConcurrentHashMap(int initialCapacity, float loadFactor) {
        this(initialCapacity, loadFactor, 1);
    }

public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel) {
        if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
            throw new IllegalArgumentException();
        if (initialCapacity < concurrencyLevel)   // Use at least as many bins
            initialCapacity = concurrencyLevel;   // as estimated threads
        long size = (long)(1.0 + (long)initialCapacity / loadFactor);
        int cap = (size >= (long)MAXIMUM_CAPACITY) ?
            MAXIMUM_CAPACITY : tableSizeFor((int)size);
        this.sizeCtl = cap;
    }
```

我们重点关注第二个

```java
public ConcurrentHashMap(int initialCapacity) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException();
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
               MAXIMUM_CAPACITY :
               tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
    this.sizeCtl = cap;
}
```

这个方法设置了初始化容量，如果小于0，就直接抛出异常，如果指定值大于了所允许的最大值就取最大值，否则，在对指定值做进一步处理。最后将cap赋值给sizeCtl，当调用构造器方法之后，sizeCtl的大小应该就代表了ConcurrentHashMap的大小，即table数组的长度。

tableSizeFor：

```java
private static final int tableSizeFor(int c) {
    int n = c - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

这个方法会将在调用构造器方法时指定的大小转换成一个2的幂次方数。另外注意的是，调用构造器方法的时候并未构造出table数组，只是算出table数组的长度，当第一次向ConcurrentHashMap插入数据的时候才真正完成初始化创建table数组的工作。

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

以上是数组的初始化函数，我们具体分析。当多个线程调用这个方法进行初始化时，为了保证正确初始化，会在第一步中通过if进行判断，若当前已经有一个线程正在初始化即sizeCtl值变为-1，这个时候其他线程在if判断为true从而调用Thread.yield()让出CPU的时间片。正在进行初始化的线程会调用U.compareAndSwapInt方法将sizeCtl改为-1即正在初始化的状态。初始化之后就会将sizeCtl大小改为数组可以使用的容量，即长度*加载因子。

put方法

put方法的实现是调用了putVal方法

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
```

对于每一个放入的值，首先利用spread方法对每一个key的hashcode进行一次hash计算，由此来确定这个值在table中的位置；

如果这个table还未初始化，先将table数组进行初始化操作；

如果这个位置是null，那就使用CAS操作直接将值放入。

如果这个位置存在节点，说明发生了hash碰撞，首先判断这个节点的类型。如果这个节点是MOVED，表明数组正在扩容；

如果是链表节点（fh > 0），则得到的节点就是hash值相同的节点组成的链表的头节点。需要依次向后遍历确定这个新加入的值所在的位置，如果遇到key相同的节点，则用新值覆盖旧值即可。否则依次向后遍历，直到链表尾节点插入这个节点。

如果这个节点的类型是Treebin的话，直接调用红黑树的插入方法进行插入新的节点；

插入完节点之后再次检查链表的长度，如果长度大于8，就把这个链表转换成红黑树；

对当前容量进行检查，如果超过了临界值（实际大小*加载因子），就需要扩容。

get方法

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```

get方法相当简单，就是     



transfer方法

当ConcurrentHashMap容量不足时，需要对它进行扩容。这个方法的基本思想跟HashMap是很像的，但是由于它是支持并发扩容的，所以要复杂的多。原因是它支持多线程进行扩容操作，而且并没有加锁。

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    if (nextTab == null) {            // initiating
        try {
            @SuppressWarnings("unchecked")
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        transferIndex = n;
    }
    int nextn = nextTab.length;
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    boolean advance = true;
    boolean finishing = false; // to ensure sweep before committing nextTab
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        while (advance) {
            int nextIndex, nextBound;
            if (--i >= bound || finishing)
                advance = false;
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            if (finishing) {
                nextTable = null;
                table = nextTab;
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        else {
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    if (fh >= 0) {
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}
```

第一部分是构建一个nextTable，它的容量是原来的两倍，这个操作是单线程完成的。

第二部分是将原来table中的元素复制到nextTable中，主要是遍历复制的过程。根据运算得到当前遍历的数组的位置i，然后利用tabAt方法获取i位置的元素再进行判断：

1. 如果这个位置为空，就在源table中的i位置放入forwardNode节点，这个也是触发并发扩容的关键点；
2. 如果这个位置是Node节点，如果它是链表的一个头节点，就构造一个反序列链表，把它们分别放在nextTable的i和i+n的位置上；
3. 如果这个位置是TreeBin节点，也做一个反序处理，并且判断是否需要untreefi，把处理的结果分别放在nextTable的i和i+n的位置上；
4. 遍历过所有节点以后就完成了复制工作，这时让nextTable作为新的table，并将更新sizeCtl为新容量的0.75，完成扩容。





CopyOnWriteArrayList

ArrayList并不是线程安全的，在读线程在读取ArrayList的时候如果有写线程在写数据的时候，基于fast-fail，会抛出ConcurrentModificationException异常，当然我们可以使用Vector，或者使用Collections的静态方法将ArrayList包装成一个线程安全的类，但这些方式都是采用synchronized对方法进行修饰，利用独占锁来保证线程安全的。但是，独占锁在同一时刻只有一个线程可以获取到对象的监视器，显然这种方法效率显然不高。

我们的程序往往都是读多写少



为什么使用线程池

在实际使用中，线程是很占用系统资源，如果线程管理不善很容易导致系统问题，在大多数并发框架中都会使用线程池来管理线程，使用线程池管理线程主要有如下好处：

1. 降低资源消耗。通过复用已存在的线程和降低关闭线程的次数来尽可能降低系统损耗；
2. 提升系统的响应速度。通过复用线程，省去创建线程的过程，因此整体上提升了系统的响应速度；
3. 提高线程的可管理性。线程是稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统稳定性，因此需要线程池来管理线程。

线程池的工作原理

1. 将一个并发任务提交给线程池，线程池分配线程去执行任务。
2. 如果当前运行的线程少于核心线程数，则会创建新的线程来执行任务，此线程成为核心线程；
3. 如果运行的线程数等于或大于核心线程数，则会将提交的任务存放到阻塞队列workQueue中；
4. 如果当前队列已满，则会创建新的线程来执行任务；
5. 如果线程个数超过最大线程数，则会使用饱和策略来进行处理；

线程池的关闭

关闭线程池，可以通过shutdown 和 shutdownNow 这两个个方法。它们的原理都是遍历线程池中所有的线程，然后依次中断线程。

shutdownNow：首先将线程池的状态设置为STOP，然后尝试停止所有正在执行和未执行任务的线程，并返回等待执行任务的列表；

shutdown：只是将线程池的状态设置成SHUTDOWN状态，然后中断所有没有正在执行任务的线程；

如何合理配置线程池参数？

可以从以下角度：

1. 任务的性质：CPU密集型任务，IO密集型任务和混合型任务。
2. 任务优先级：高，中和低。
3. 任务执行时间：长，中和短。
4. 任务的依赖性：是否依赖其他系统资源，如数据库连接。



ThreadPoolExecutor构造方法

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```

参数含义：

corePoolSize：核心线程数，当有新任务提交时，会执行以下判断

* 如果运行线程少于corePoolSize，则创建新线程来处理任务，即使线程池中的其他线程是空闲的；
* 如果线程池中的线程数量大于等于 corePoolSize 且小于 maximumPoolSize，则只有当workQueue满时才创建新的线程去处理任务；
* 如果设置的corePoolSize 和 maximumPoolSize相同，则创建的线程池的大小是固定的，这时如果有新任务提交，若workQueue未满，则将请求放入workQueue中，等待有空闲的线程去从workQueue中取任务并处理；
* 如果运行的线程数量大于等于maximumPoolSize，这时如果workQueue已经满了，则通过handler所指定的策略来处理任务；

maximumPoolSize：最大线程数量；

workQueue：等待队列，当任务提交时，如果线程池中的线程数量大于等于corePoolSize的时候，把该任务封装成一个Worker对象放入等待队列；

KeepAliveTime：线程池维护线程所允许的空闲时间。当线程池中的线程数量大于corePoolSize的时候，如果这时候没有新任务提交，核心线程外的线程不会被立即销毁，而是会等待，直到等待的时间超过了KeepAliveTime；

threadFactory：用来创建新的线程；

handler：它是RejectedExecutionHandler类型的变量，表示线程池的饱和策略。



execute方法：

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    /*
     * clt记录着runState和workerCount
     */
    int c = ctl.get();
    /*
     * workerCountOf方法取出低29位的值，表示当前活动的线程数；
     * 如果当前活动线程数小于corePoolSize，则新建一个线程放入线程池中；
     * 并把任务添加到该线程中。
     */
    if (workerCountOf(c) < corePoolSize) {
        /*
         * addWorker中的第二个参数表示限制添加线程的数量是根据corePoolSize来判断还是maximumPoolSize来判断；
         * 如果为true，根据corePoolSize来判断；
         * 如果为false，则根据maximumPoolSize来判断
         */
        if (addWorker(command, true))
            return;
        /*
         * 如果添加失败，则重新获取ctl值
         */
        c = ctl.get();
    }
    /*
     * 如果当前线程池是运行状态并且任务添加到队列成功
     */
    if (isRunning(c) && workQueue.offer(command)) {
        // 重新获取ctl值
        int recheck = ctl.get();
        // 再次判断线程池的运行状态，如果不是运行状态，由于之前已经把command添加到workQueue中了，
        // 这时需要移除该command
        // 执行过后通过handler使用拒绝策略对该任务进行处理，整个方法返回
        if (! isRunning(recheck) && remove(command))
            reject(command);
        /*
         * 获取线程池中的有效线程数，如果数量是0，则执行addWorker方法
         * 这里传入的参数表示：
         * 1. 第一个参数为null，表示在线程池中创建一个线程，但不去启动；
         * 2. 第二个参数为false，将线程池的有限线程数量的上限设置为maximumPoolSize，添加线程时根据maximumPoolSize来判断；
         * 如果判断workerCount大于0，则直接返回，在workQueue中新增的command会在将来的某个时刻被执行。
         */
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    /*
     * 如果执行到这里，有两种情况：
     * 1. 线程池已经不是RUNNING状态；
     * 2. 线程池是RUNNING状态，但workerCount >= corePoolSize并且workQueue已满。
     * 这时，再次调用addWorker方法，但第二个参数传入为false，将线程池的有限线程数量的上限设置为maximumPoolSize；
     * 如果失败则拒绝该任务
     */
    else if (!addWorker(command, false))
        reject(command);
}
```

简单来说，在执行execute()方法时如果状态一直是RUNNING时，的执行过程如下：

1. 如果`workerCount < corePoolSize`，则创建并启动一个线程来执行新提交的任务；
2. 如果`workerCount >= corePoolSize`，且线程池内的阻塞队列未满，则将任务添加到该阻塞队列中；
3. 如果`workerCount >= corePoolSize && workerCount < maximumPoolSize`，且线程池内的阻塞队列已满，则创建并启动一个线程来执行新提交的任务；
4. 如果`workerCount >= maximumPoolSize`，并且线程池内的阻塞队列已满, 则根据拒绝策略来处理该任务, 默认的处理方式是直接抛异常。

这里要注意一下`addWorker(null, false);`，也就是创建一个线程，但并没有传入任务，因为任务已经被添加到workQueue中了，所以worker在执行的时候，会直接从workQueue中获取任务。所以，在`workerCountOf(recheck) == 0`时执行`addWorker(null, false);`也是为了保证线程池在RUNNING状态下必须要有一个线程来执行任务。

![executor.png](http://www.ideabuffer.cn/2017/04/04/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Java%E7%BA%BF%E7%A8%8B%E6%B1%A0%EF%BC%9AThreadPoolExecutor/executor.png)