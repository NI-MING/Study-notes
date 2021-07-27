[TOC]

# Java基础总结三（泛型）

## 泛型的创建

### 泛型类

我们最常用泛型的地方就是集合，因此，我们编写自己的List来体会泛型：

```java
public class TestList<T> {

    private Object[] instances = new Object[0];

    public T get(int index) {
        return (T) instances[index];
    }

    public void set(int index,T newInstance) {
        instances[index] = newInstance;
    }

    public void add(T newInstance) {
        instances = Arrays.copyOf(instances,instances.length + 1);
        instances[instances.length - 1] = newInstance;
    }
}
```

看一下使用方法：

```java
TestList<String> testList = new TestList<>();
testList.add("wlk");
String str = testList.get(0);
System.out.println(str); // wlk
```

### 泛型接口

我们除了创建泛型类，泛型接口与其很类似：

```java
public interface Shop <T>{
    T buy();
    float refund(T item);
}

// 实现
public class RealShop<E> implements Shop<E>{
    @Override
    public E buy() {
        return null;
    }

    @Override
    public float refund(E item) {
        return 0;
    }
}
```

再比如，我们想要实现一个水果商店：

```java
public class Fruit {
}

public class Apple extends Fruit{
    
}

public class FruitShop<E> implements Shop<E>{
    @Override
    public E buy() {
        return null;
    }

    @Override
    public float refund(E item) {
        return 0;
    }
}

FruitShop<Apple> fruitShop = new FruitShop<>(); // 这个商店卖苹果
FruitShop<Phone> stringShop = new FruitShop<>(); // 这个水果商店卖手机??? 当然不和逻辑
```

由于我们的商店泛型 E 没有加以限制，会导致错误，我们来给他加上限制：

```java
public class FruitShop<E extends Fruit> implements Shop<E>{
    @Override
    public E buy() {
        return null;
    }

    @Override
    public float refund(E item) {
        return 0;
    }
}
```

现在这个水果商店只可以卖 Fruit 或者 Fruit 的子类，如Apple，而不能卖手机。

### 泛型方法

自己声明了泛型的方法

泛型类，是在实例化类的时候指明泛型的具体类型；泛型方法，是在调用方法的时候指明泛型的具体类型 。



## 泛型的上界与下界

## 什么时候使用泛型

类型检查和自动转型