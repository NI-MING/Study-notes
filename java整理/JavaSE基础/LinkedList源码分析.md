# LinkedList源码分析

## LinkedList结构

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
```

LinkedList继承自`AbstractSequentialList<E>`，实现了List、Deque、Cloneable、Serializable接口。

LinkedList内部是一个双向链表结构。

## 源码分析

### 属性

```java
// 元素个数
transient int size = 0;

// 指向第一个节点
transient Node<E> first;

// 指向最后一个节点
transient Node<E> last;

// 内部类，用于实现链表
private static class Node<E> {
    E item; // 存储的元素
    Node<E> next; // 下一个节点 尾元素的next指向null
    Node<E> prev; // 上一个节点 头元素的prev指向null

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

### 构造

```java
// 默认构造方法，size为0，first和last为空
public LinkedList() {
}
```

```java
public LinkedList(Collection<? extends E> c) {
    this();
    addAll(c);
}
```

```java
public boolean addAll(Collection<? extends E> c) {
    return addAll(size, c);
}
```

```java
public boolean addAll(int index, Collection<? extends E> c) {
    checkPositionIndex(index);

    // 将集合转化为数组
    Object[] a = c.toArray();
    int numNew = a.length;
    // 数组长度为0，直接返回false
    if (numNew == 0)
        return false;
	// pred表示前驱节点，succ是当前要插入的位置
    Node<E> pred, succ;
    if (index == size) {
        // 添加到list的末尾
        succ = null;
        pred = last;
    } else {
        // 添加到指定位置，即index前
        succ = node(index);
        pred = succ.prev;
    }

    // 遍历数组，将其连成链表
    for (Object o : a) {
        @SuppressWarnings("unchecked") E e = (E) o;
        Node<E> newNode = new Node<>(pred, e, null);
        if (pred == null)
            // 如果list表为空，新节点为首节点
            first = newNode;
        else
            // 如果存在前节点，前节点会向后指向新加的节点
            pred.next = newNode;
        //新加的节点成为前一个节点
        pred = newNode;
    }
	//添加集合中的元素完成后，将指定位置后面的元素连到链表后方
    if (succ == null) {
        last = pred;
    } else {
        pred.next = succ;
        succ.prev = pred;
    }

    size += numNew;
    modCount++;
    return true;
}
```

### 方法

#### add(E e)

```java
// 将元素插入链表末尾
public boolean add(E e) {
    linkLast(e);
    return true;
}
```

```java
void linkLast(E e) {
    final Node<E> l = last;
    final Node<E> newNode = new Node<>(l, e, null);
    // 更新last节点为新节点
    last = newNode;
    if (l == null)
        // 如果last节点为null，将新节点设为first
        first = newNode;
    else
        // 否则，将last节点的next指向新节点
        l.next = newNode;
    size++;
    modCount++;
}
```

#### add(int index,E element)

```java
public void add(int index, E element) {
    checkPositionIndex(index);

    if (index == size)
        linkLast(element);
    else
        linkBefore(element, node(index));
}
```

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

```java
// 在指定节点前插入节点，节点succ不能为空
void linkBefore(E e, Node<E> succ) {
    //取出指定节点的前驱节点
    final Node<E> pred = succ.prev;
    final Node<E> newNode = new Node<>(pred, e, succ);
    //修改指定节点的前驱为新节点
    succ.prev = newNode;
    if (pred == null)
        first = newNode;
    else
        pred.next = newNode;
    size++;
    modCount++;
}
```

#### remove()

```java
public E remove() {
    return removeFirst();
}
```

```java
public E removeFirst() {
    final Node<E> f = first;
    if (f == null)
        throw new NoSuchElementException();
    return unlinkFirst(f);
}
```

```java
private E unlinkFirst(Node<E> f) {
    // assert f == first && f != null;
    final E element = f.item;// 取出首节点元素
    final Node<E> next = f.next;// 得到下一个节点
    f.item = null;// 将首节点元素置空
    f.next = null;// 将节点后继置空，帮助回收
    first = next;// 首节点的下一个节点成为新的首节点
    if (next == null)
        last = null;
    else
        next.prev = null;
    size--;
    modCount++;
    return element;
}
```

#### remove(Object o)

```java
public boolean remove(Object o) {
    if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null) {
                unlink(x);
                return true;
            }
        }
    } else {
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item)) {
                unlink(x);
                return true;
            }
        }
    }
    return false;
}
```

```java
E unlink(Node<E> x) {
    // assert x != null;
    final E element = x.item;//取出节点的元素
    final Node<E> next = x.next;//取出节点的下一个节点
    final Node<E> prev = x.prev;//取出节点的上一个节点

    if (prev == null) {
        //如果前一个节点为空(如当前节点为首节点)，后一个节点成为新的首节点
        first = next;
    } else {
        //如果前一个节点不为空，那么他的后继指向当前的下一个节点
        prev.next = next;
        x.prev = null;
    }

    if (next == null) {
        //如果后一个节点为空(如当前节点为尾节点)，当前节点前一个成为新的尾节点
        last = prev;
    } else {
        //如果后一个节点不为空，后一个节点向前指向当前的前一个节点
        next.prev = prev;
        x.next = null;
    }

    x.item = null;
    size--;
    modCount++;
    return element;
}
```

#### remove(int index)

```java
public E remove(int index) {
    checkElementIndex(index);
    return unlink(node(index));
}
```