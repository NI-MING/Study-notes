# Fragment

## 什么是Fragment

为了让界面可以在平板上更好的展示，Android在3.0版引入了Fragment功能，它非常类似于Activity，可以像Activity一样包含布局，但比Activity更加轻量。Fragment通常嵌套在Activity中使用。Activity作为视图的承接和管理在很多场景下过与繁重，需要一个简化的视图管理器来替代Activity的操作。

## Fragment的生命周期

![Fragment](C:\Users\wlk\Desktop\Fragment.png)

这是Activity与Fragment生命周期的关系图，可以看出，Fragment的生命周期是与Activity极为相似的，但比Activity多了几个方法：

* onAttach

  当Fragment与Activity关联时调用。在这个阶段，我们可以用来传入初始的参数。可以将context转化成Activity保存下来，避免后期频繁的获取Activity。这样传入参数会导致Fragment重建时初始化参数丢失；如果不想丢失初始化参数，就应该把参数用setArgument来注入，这样重建时就会自动从Argument中取出参数。

* onCreateView

  创建该Fragment的视图。

* onActivityCreated

  当Activity中的onCreate方法执行完时调用。

* onDestroyView

  与onCreateView方法相对应，当该Fragment的视图被移除时调用。

  当使用replace的时候，只会走到当前状态，而不会继续回调到onDestroy，这样是因为在销毁视图的同时，Fragment状态得以保留，方便下次加载时可以直接走onCreateView，比冷加载更快。

* onDetach

  Fragment和Activity解除关联时调用。

生命周期的场景：

* 创建Fragment： `onAttach() —> onCreate() —> onCreateView() —> onActivityCreated() —> onStart() —> onResume()`

* 按下Home键回到桌面 / 锁屏 ：`onPause() —> onStop()`

* 从桌面回到Fragment / 解锁： `onStart() —> onResume()`

* 切换到其他Fragment： `onPause() —> onStop() —> onDestroyView()`

* 切换回本身的Fragment： `onCreateView() —> onActivityCreated() —> onStart() —> onResume()`

* 按下Back键退出： `onPause() —> onStop() —> onDestroyView() —> onDestroy() —> onDetach()`

## Fragment的使用

### 静态使用

1. 创建一个类继承Fragment，重写onCreateView方法，来确定Fragment要显示的布局。
2. 在布局文件中添加fragment，使用与普通View无异。

```java
public class MyFragment extends Fragment {

    @Nullable
    @Override
    public View onCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        return inflater.inflate(R.layout.myfragment_layout,container,false);
    }
    
}
```

```xml
<fragment
    android:id="@+id/myfragment"
    android:name="com.nullpointerexception.task1.MyFragment"
    android:layout_width="match_parent"
    android:layout_height="match_parent"/>
```

这样，我们的Fragment就嵌入了Activity。

### 动态使用

1. 在Activity对应的布局中写一个FramLayout用来指定Fragment添加的位置，这样添加Fragment就可以灵活的展示在你想要的位置。

2. 编写类继承Fragment，与静态使用无异。
3. 在Activity中，创建FragmentManager。
4. 使用FragmentManager实例化出FragmentTransaction。
5. 调用transaction的方法对Fragment进行添加替换。
6. 最后不要忘记commit事务。

```java
MyFragment fragment = new MyFragment();
getSupportFragmentManager()
        .beginTransaction()
        .add(R.id.fragment,fragment)
        .commit();
```

transaction中方法介绍：

* add：向Activity中添加一个Fragment
* remove：从Activity中移除一个Fragment，如果被移除的Fragment没有添加到回退栈，这个Fragment实例将会被销毁。
* replace：使用一个Fragment替换当前的Fragment。
* hide：隐藏当前的Fragment，仅设为不可见，并不销毁。
* show：显示之前隐藏的Fragment。
* detach：将View从UI中移除，和remove不同，此时Fragment的状态依然由FragmentManager维护。
* attach：重建View视图，附加到UI上并显示。
* commit：提交事务。

## Fragment的回退栈

Fragment回退栈是用来保存每一次Fragment事务发生的变化。如果将Fragment任务添加到回退栈，当用户点击返回按钮时，将看到上一次保存的Fragment。一旦Fragment完全从回退栈中弹出，用户再次点击返回键，则退出当前Activity。

如何添加到回退栈？使用**addToBackStack**。

transaction的replace方法相当于remove和add的结合体，如果不添加事务到回退栈，前一个Fragment实例便会被销毁。如果调用了addToBackStack方法，前一个Fragment的实例不会被销毁，但视图层次依然被销毁，即会调用onDestroyView方法，使用onCreateView重建。

## Fragment与Activity之间通信

* 在Fragment可以通过getActivity得到当前绑定的Activity的实例，然后进行操作。
* 数据库
* handler
* 回调
* Eventbus
* 广播

## 为何视图控制器建议使用Fragment而不是Activity

- Fragment可以被混淆，Activity不行
  - Android中我们一般使用ProGuard进行代码混淆，但是ProGuard是Java的混淆器，他不能混淆XML文件，这样就会导致如果Activity被混淆，那么Manifest就会找不到Activity文件而导致程序崩溃；四大组件中的其它三个也类似；view也类似(在layout的文件中的指定文件包名))
- Fragment不是组件，资源占用更少
- 切换速度比Activity快太多

## 懒加载

### 场景一：ViewPager + TabLayout + Fragment

ViewPager在左右滑动向用户展示数据时，会预加载左右的界面，当你滑动到下一页的时候，Fragment中的数据已经加载好了。

即使通过设置`viewPager.setOffscreenPageLimit(0);`也会加载一项，因为当你传入参数小于1时会使用默认值，也就是1。

```java
public class ViewPagerFragment extends Fragment {

    @Nullable
    @Override
    public View onCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        return inflater.inflate(R.layout.viewpager_fragment,container,false);
    }

    @Override
    public void onActivityCreated(@Nullable Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        String label = getArguments().getString("label");
        TextView textView = getView().findViewById(R.id.tv);
        new Thread(() -> {
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            textView.post(() -> textView.setText(label));
        }).start();
    }
    
    public static ViewPagerFragment newInstance(String label){
        Bundle args = new Bundle();
        args.putString("label", label);
        ViewPagerFragment fragment = new ViewPagerFragment();
        fragment.setArguments(args);
        return fragment;
    }
}
```

这里使用子线程睡眠一秒模拟从网上获取数据。可以发现ViewPager在未显示出左右两边的Fragment时也会预加载。这可能导致用户流量的浪费。秉着从用户角度出发，需要在显示此Fragment时在加载相关的数据，否则就不加载。这时我们就要用到Fragment中的一个方法：setUserVisibleHint。这个方法是Fragment中的一个回调方法。当前Fragment对用户可见时，setUserVisibleHint回调，其中参数isVisibleToUser=true。当前Fragment由可见到不可见或者实例化时，setUserVisibleHint回调，其中参数isVisibleToUser=false。

按照上面的规则，我们一般在onActivityCreated中加载数据，这时候只要判断Fragment是否对用户可见，调用fragment.getUserVisibleHint可以获得isVisibleToUser的值，如果为true，表示可见，就加载数据，如果不可见，就不加载数据。预加载时，我们的Fragment生命周期已经走到了onResume，当我们切换到下一个Fragment时，只要在setUserVisibleHint方法中添加加载数据的方法，就可以实现懒加载。注意，我们此时应该设置一个标志位，来记录数据是否加载过，如果加载过，则不重复加载，因为我们在ViewPager间来回切换时会回调多次setUserVisibleHint方法。

这是androidx 1.1.0之前实现懒加载的方式，在android 1.1.0之后，setUserVisibleHint方法被废弃，官方推荐我们使用setMaxLifecycle来管理Fragment的生命周期，同时，也提供了新的懒加载实现方式。setMaxLifecycle是FragmentTransaction中的一个方法，使用和add等方法相同，它有两个参数，第一个自然是我们的Fragment，第二个是Lifecycle.State。State是一个枚举类：

```java
public enum State {
        /**

   		* Destroyed state for a LifecycleOwner. After this event, this Lifecycle will not 		  *dispatchmore events. For instance, for an {@link android.app.Activity}, this 		*state is reached
        * <b>right before</b> Activity's {@link android.app.Activity#onDestroy() 				*onDestroy} call.
        */

	DESTROYED,
/**
     * Initialized state for a LifecycleOwner. For an {@link android.app.Activity}, this is
     * the state when it is constructed but has not received
     * {@link android.app.Activity#onCreate(android.os.Bundle) onCreate} yet.
     */
    INITIALIZED,

    /**
     * Created state for a LifecycleOwner. For an {@link android.app.Activity}, this state
     * is reached in two cases:
     * <ul>
     *     <li>after {@link android.app.Activity#onCreate(android.os.Bundle) onCreate} 		 * call;
     *     <li><b>right before</b> {@link android.app.Activity#onStop() onStop} call.
     * </ul>
     */
    CREATED,

    /**
     * Started state for a LifecycleOwner. For an {@link android.app.Activity}, this 		 * state
     * is reached in two cases:
     * <ul>
     *     <li>after {@link android.app.Activity#onStart() onStart} call;
     *     <li><b>right before</b> {@link android.app.Activity#onPause() onPause} call.
     * </ul>
     */
    STARTED,

    /**
     * Resumed state for a LifecycleOwner. For an {@link android.app.Activity}, this state
     * is reached after {@link android.app.Activity#onResume() onResume} is called.
     */
    RESUMED;

    /**
     * Compares if this State is greater or equal to the given {@code state}.
     *
     * @param state State to compare with
     * @return true if this State is greater or equal to the given {@code state}
     */
    public boolean isAtLeast(@NonNull State state) {
        return compareTo(state) >= 0;
    }
}
```
在setMaxLifecycle中接收的生命周期状态要求不能低于CREATED，否则会抛出一个IllegalArgumentException的异常。

当我们设置为CREATED的时候，Fragment的生命周期走到onCreate就会停下，并且Activity中不会显示Fragment。当我们的Fragment已经执行到了onResume，此时设置为CREATED，Fragment会依次回调onPause、onStop、onDestroyView。其余情况类似。所以得出结论：

> 通过setMaxLifecycle方法可以精确控制Fragment生命周期的状态，如果Fragment的生命周期状态小于被设置的最大生命周期，则当前Fragment的生命周期会执行到被设置的最大生命周期，反之，如果Fragment的生命周期状态大于被设置的最大生命周期，那么则会回退到被设置的最大生命周期。

知道了这些，如何去实现懒加载呢？androidx 1.1.0版本中的FragmentStatePagerAdapter已经帮我们实现了，只需要传入简单的参数即可。

**BEHAVIOR_RESUME_ONLY_CURRENT_FRAGMENT**

将behavior参数设置成这个，就意味着当前可见的Fragment会被执行到onResume，而其他不可见的Fragment的生命周期只会执行到onStart。这样，懒加载的实现就及其简单了，只要在onResume中判断是否是第一次加载该数据即可。

### 场景二：FragmentManager + FragmentTransaction+ Fragment

这种场景通常用在配合底部tab使用，类似微信底部tab切换页面。通常这种切换是配合FragmentTransaction的hide和show方法使用。当Fragment被add我们的FragmentManager时，Fragment中已经有数据了，但这违背了我们想要实现的懒加载。照搬场景一的老方法？但setUserVisibleHint又不会被回调，何况已被废弃。这时，有一个方法就可以派上用场：**onHiddenChanged（boolean hidden）**

onHiddenChanged方法是当Fragment的隐藏状态变化示被调用，当Fragment没有被隐藏时即调用show方法，当前onHiddenChanged回调，其中参数hidde=false，当Fragment被隐藏时即调用hide了方法，onHiddenChanged()回调，其中参数hidde=true。还有一点注意的是使用hide和show时，fragment的所有生命周期方法都不会调用，除了onHiddenChanged（）。

```java
@Override
public void onHiddenChanged(boolean hidden) {
    super.onHiddenChanged(hidden);
    //1、onHiddenChanged调用在Resumed之前，所以此时可能fragment被add, 但还没resumed
    if(!hidden && !this.isResumed())
        return;
    //2、使用hide和show时，fragment的所有生命周期方法都不会调用，除了onHiddenChanged（）
    if(!hidden && isFirstVisible && this.isAdded()){
        onLazyLoadData();
        isFirstVisible = false;
    }
}
```

Fragment至此也就介绍完了，Fragment提供给我们一个非常优秀的实现界面的方式，Google也是将其收录到了Jetpack中，足以证明其实用价值。掌握好了Fragment，不仅适配了平板端，减少了代码复用，提高开发效率，更使我们的界面轻量化。