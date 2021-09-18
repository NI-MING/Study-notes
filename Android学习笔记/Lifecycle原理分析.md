# Lifecycle原理分析

Lifecycle是Google官方提供的方便管理生命周期事件的方式，可以更加方便的监听生命周期变化事件，它以注解的方式作用在方法上，当生命周期方法调用时，它也会跟随调用。

**Lifecycle是基于观察者模式实现的，那么，我们就要搞清楚谁是观察者，谁是被观察者。**

**Activity和Fragment都实现了LifecycleOwner接口，他们是被观察者，被观察的是生命周期函数的回调，观察者就是通过调用getLifecycle().addObserver()被添加进去的。**

**Lifecycle如何观察到Activity和Fragment的生命周期**

我们的Activity和Fragment都间接继承了ComponentActivity，在onCreate方法中会调用ReportFragment的 injectIfNeededIn方法，该方法是将一个无UI的Fragment与我们的组件进行绑定，在这个Fragment中，来监听我们组件的生命周期。

当ReportFragment的生命周期发生变化时，调用dispatch方法并将当前生命周期对应的事件Event分发，交给当前所属的实现了Lifecycle或LifecycleOwner的Activity处理，dispatch中，调用了LifecycleRegistry的handleLifecycleEvent方法，该方法会根据传入的Event来判断当前组件所应该处于的State，然后调用moveToState去同步State，这个过程是sync方法执行的，sync中得到了LifecycleOwner，也就是被观察者，通过比较state和ObserverWithState中的state的大小，可以判断出我们想要要回调的方法了。

- 如果ObserverWithState的state小于当前state，那么就调用forwardPass方法，
- 如果大于当前state，那么就调用backwardPass方法。

```java
void dispatchEvent(LifecycleOwner owner, Event event) {
            //获取当前事件的下一个状态
            State newState = getStateAfter(event);
            mState = min(mState, newState);
            //生命周期的分发，最终在这里调用到 自己实现的
            mLifecycleObserver.onStateChanged(owner, event);
            //这里会在回调完成后，才更新状态
            mState = newState;
        }
```

onStateChanged中通过反射来调用我们的观察者中的方法。

