# ArrayList源码分析

## ArrayList结构

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```

ArrayList继承自AbstractList，实现了List、RandomAccess、cloneable、Serializable接口，支持随机访问、克隆和序列化。和Vector不同，ArrayList不是线程安全的容器，所以，建议在单线程中使用它，而在多线程中使用Vector或CopyOnWriteArrayList，或者用synchronizedList将其包装。

## 源码分析

### 属性

```java
// 序列号
private static final long serialVersionUID = 8683452581122892189L;

// ArrayList的默认初始容量
private static final int DEFAULT_CAPACITY = 10;

// 当实例化是指定容量为0时，返回该数组
private static final Object[] EMPTY_ELEMENTDATA = {};

// 空数组，实例化时没有指定容量，返回该数组
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

// 当前数据存放的地方
transient Object[] elementData; // non-private to simplify nested class access

// ArrayList中存储的数据量
private int size;

// 集合数组被修改次数标识
protected transient int modCount = 0;
```

### 构造

```java
// 创建一个指定初始容量大小的ArrayList
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
}

// 没有指定初始容量时，返回一个指定的空数组
// 当第一次add时，会扩容为10
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}

// 用collection创建一个ArrayList
public ArrayList(Collection<? extends E> c) {
    // 把集合转化为Object[]数组
    elementData = c.toArray();
    // 直接将数组长度赋值给size
    if ((size = elementData.length) != 0) {
        // 如果不是Object[]数组，利用copyOf构造一个Object[]数组
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, size, Object[].class);
    } else {
        // 如果size为0，就用空数组替换
        this.elementData = EMPTY_ELEMENTDATA;
    }
}
```

### 方法

#### add(E e)

```java
public boolean add(E e) {
    // 确保内部容量足够
    ensureCapacityInternal(size + 1);  
    // 将数据放在size上，将size++
    elementData[size++] = e;
    return true;
}
```

```java
private void ensureCapacityInternal(int minCapacity) {
    ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}
```

```java
private static int calculateCapacity(Object[] elementData, int minCapacity) {
    // 第一次添加数据，便取10
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        return Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    return minCapacity;
}
```

```java
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // 当元素个数大于数组容量，就需要扩容
    if (minCapacity - elementData.length > 0)
        // 扩容
        grow(minCapacity);
}
```

```java
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    // 扩容1.5倍
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    // 第一次扩容 0 - 10 < 0,所以取10
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // 拷贝到新数组
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

#### void add(int index,E element)

```java
public void add(int index, E element) {
    rangeCheckForAdd(index);

    ensureCapacityInternal(size + 1);  // Increments modCount!!
    System.arraycopy(elementData, index, elementData, index + 1,
                     size - index);
    elementData[index] = element;
    size++;
}

//检查下标是否越界
private void rangeCheckForAdd(int index) {
    if (index > size || index < 0)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```

#### remove(int index)

```java
public E remove(int index) {
    rangeCheck(index);

    modCount++;
    
    // 保存将要移除的旧值
    E oldValue = elementData(index);

    // 要移动的数组长度
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work

    return oldValue;
}
```

#### remove(Object o)

```java
public boolean remove(Object o) {
    if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
    } else {
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}
```

```java
private void fastRemove(int index) {
    modCount++;
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work
}
```

#### trimToSize()

```java
// 将数组大小调整到ArrayList存储元素的大小
public void trimToSize() {
    modCount++;
    if (size < elementData.length) {
        elementData = (size == 0)
          ? EMPTY_ELEMENTDATA
          : Arrays.copyOf(elementData, size);
    }
}
```

#### clear()

```java
public void clear() {
    modCount++;

    // clear to let GC do its work
    for (int i = 0; i < size; i++)
        elementData[i] = null;

    size = 0;
}
```

#### indexOf(Object o)

```java
public int indexOf(Object o) {
    if (o == null) {
        for (int i = 0; i < size; i++)
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = 0; i < size; i++)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
}
```

#### contains(Object o)

```java
public boolean contains(Object o) {
    return indexOf(o) >= 0;
}
```