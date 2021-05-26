# BroadCast

## 什么是广播

广播是一种消息型组件，用于在不同组件甚至不同应用之间传递消息。

## 广播的分类

* 标准广播

  完全异步的广播，多个广播接收器可以同时接受广播，且无法被截断。

* 有序广播

  有序广播通过sendOrderedBroadcast来发送，所有的广播接收器按照优先级的顺序依次接收，广播的优先级可以通过receiver的intent-filter中android priority属性设置，数值越大优先级越高（-1000~1000）。可以使用abortBroadcast来截断该广播。

* 本地广播

  当我们并不需要将广播的内容发送给外部App使用时，仅在自己App内使用就可以使用本地广播。

* 粘性广播

  粘性消息的意思就是发送后一直存在于系统消息容器中，等待对应的处理器去处理，当处理完消息后，通过removeStickyBroadcast取消粘性消息。在Android5.0后被废弃。

## 模型描述

Broadcast是设计模式中的观察者模式：基于消息的发布订阅事件模型

三个角色

* 广播接收者（订阅者）
* 广播发送者（发布者）
* AMS（消息中心）

广播发送者发送广播，通过Binder进程通信机制发送给AMS，请求AMS处理，由AMS在注册列表中寻找合适的广播接收者。广播接收者也是通过Binder机制在AMS中注册。广播接收者通过回调onReceive方法处理消息。

## 注册方式

### 静态注册

静态注册，就是在AndroidManifest.xml文件中通过添加<receiver>标签注册此广播接收者。

```xml
<receiver
            android:name=".MyReceiver"
            android:enabled="true"
            android:exported="true">

            <intent-filter
                android:priority="10">
                <action android:name="自定义匹配规则"/>
            </intent-filter>

</receiver>
```

静态注册的广播不会收到任何组件生命周期的影响，即使应用未被打开或者应用关闭后，仍然可以接收到广播。这方便了开发者，但是由于开发者在App中大量的滥用静态广播，导致App从后台被唤醒，严重的影响到手机的电量性能，所以官方在每个都会削减静态广播的功能。在8.0之后，所有的隐式广播不允许使用静态注册的方式来接收（少部分系统广播可用），想要用静态注册的接收者发送广播时需要加上包名。

### 动态注册

动态注册，即使用Java代码在Activity中注册

```java
broadCastReceiver = new MyBroadCastReceiver();
IntentFilter filter = new IntentFilter();
filter.addAction("");

registerReceiver(broadCastReceiver,filter);
```

注意，注册完之后一定要在某个地方接触注册

```java
unregisterReceiver(broadCastReceiver);
```

