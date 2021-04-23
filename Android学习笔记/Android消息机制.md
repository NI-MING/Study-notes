# Android消息机制

Handler是Android消息机制的上层接口。

通过这个接口可以很方便的将消息切换到Handler所在的线程中去执行。Android开发工程师通常用它来更新UI，但是如果只认为它的作用是用来更新UI的话，未免将Handler的功能过于简单化。我甚至认为，Handler是我们Android应用的心脏，如新启动一个Activity是都会使用到Handler，所以，Handler的重要性不言而喻。

* 为什么UI更新要在主线程中：ViewRootImpl中会进行checkThread操作，所以不能在子线程中更新UI，提示，ViewRootImpl是在onResume之后才变得可见的，之前的过程是没有线程检查的。
* 主线程负责更新，子线程负责耗时操作，能够大大提升响应效率。
* 如果在不同的线程去控制用一个控件，由于网络延时或者大量耗时操作，会使UI绘制错乱，出了问题也很难去排查到底是哪个线程更新时出了问题。
* UI线程非安全线程，非线程安全不能加Lock线程锁，否则会阻塞其他线程对View的访问，效率就会变得低下。

Handler的工作是要与Looper与MessageQueue配合使用，所以，我们就要在有Looper的线程中创建Handler，或者在有Handler的线程中创建Looper。

* 我们在UI线程中为什么不需要创建Looper呢？那是因为当App启动时Looper已经自动创建了。我们只要找到App的入口函数，就可以看到Looper的创建过程了。众所周知，一段Java代码的入口函数是main()方法，而Android的main()函数在ActivityThread这个类里。![main](C:\Users\wlk\Desktop\扔物线\picture\main.png)

  我们可以看到，在main()函数中有这么一行代码Looper.prepareMainLooper()；我们的主线程就是在这里开启Looper的。有兴趣的同学可以自己看一看。

介绍Handler之前，我们还得再介绍一个比较重要的东西**ThreadLocal**，它并不是线程，它的作用是用来保证我们在不同线程中获取数据时，拿到的是自己线程中存储的数据，而Handler中的ThreadLocal中存储的就是每个线程中的Looper。在我的理解中，ThreadLocal中的数据是以线程为作用域的，在不同的线程中取到的都是自己线程中的数据，而不是同一个数据的副本，在多线程环境下，可以防止自己的变量被其他线程篡改。不采用ThreadLocal的话，其实可以维持一个全局的HashMap来指定查询对应线程下的值，对于Handler来说，就是对应线程下的Looper。

要了解ThreadLocal，首先要了解ThreadLocalMap，ThreadLocalMap是ThreadLocal的内部类，其实，ThreadLocal并不存储数据，只是提供对ThreadLocalMap的操作，ThreadLocalMap才是真正存数据的地方。具体过程是，Thread为每个线程创建一个ThreadLocalMap，ThreadLocalMap里面有一个Entry类型的数组，用来存每个Entry，而Entry是什么呢？它又是ThreadLocalMap里面的一个静态内部类，它通过自己的构造函数将ThreadLocal和数据按照键值对的形式存下来，至于Entry在数组中如何存储，是根据ThreadLocal的哈希值与数组长度-1进行与运算，得到 i 值，i 就是数组的下标，具体逻辑如下：

![hhh](C:\Users\wlk\Desktop\扔物线\picture\hhh.png)

我们看for循环里干的事情，用 i 在这个数组中取值，如果有值并且得到的 key 就是你要设置value的key，就直接设置值然后返回，有值但 key 为 null 了，就更新为新的。那如果有值，key也不为 null，也不与新的 key 相同呢？那就将 i + 1；直到找到一个符合的位置。这样，那我们也可以知道get方法的具体操作了。到这里，我们已经了解了数据隔离的本质了，总结来说，就是每个线程会维护属于自己线程的ThreadLocalMap，而存数据是使用到的ThreadLocal是你要存数据的键，根据这个键，你可以在不同的线程中的ThreadLocalMap取到不同的值，从而形成数据隔离。其实，对象都是存储在堆上的，只是通过了一些技巧修改了数据的可见性。

至此，我们理解ThreadLocal中的get/set方法就很容易了。

首先是ThreadLocal的get方法：

![ThreadLocal.get](C:\Users\wlk\Desktop\扔物线\picture\ThreadLocal.get.png)

先拿到当前线程 t，再通过 t 取到当前线程中的ThreadLocalMap，在ThreadLocalMap中通过 this 也就是ThreadLocal取到对应的 e，调用 e.value 取到想要的值，over。

再看set方法：

![ThreadLocal.set](C:\Users\wlk\Desktop\扔物线\picture\ThreadLocal.set.png)

emmmm....经过我们上面的分析，这个方法极为清晰，就不说了。

闲话少说，我们继续我们的主要工作 Handler。

我们从哪里看起呢？先回忆回忆Handler的使用：

![Handler](C:\Users\wlk\Desktop\扔物线\picture\Handler.png)

下面，我们就可以开始Handler的学习了。

说整套Handler消息机制，肯定不止会有Handler一个类在工作，具体是由Handler、Looper、MessageQueue、Message四个类配合工作。

Handler：

Handler的作用是投递消息和处理消息的，它会绑定一个Looper，一个线程可以有多个Handler，但只会有一个Looper，为什么？我们就可以看一看Handler是如何被我们创建出来的，我们通常调用Handler()这个空参构造来创建，它会通过重载最终调用到这个构造器：

![Handler()](C:\Users\wlk\Desktop\扔物线\picture\Handler().png)

Handler中的Looper通过Looper.myLooper()绑定，MessageQueue是通过mLooper间接绑定的。Handler还有一个主要的方法：handleMessage，它就是我们在线程中处理事务时自定义处理规则。有了处理事务的方法，那发送消息呢？当然是sendMessage()，走去瞅瞅

我大致看了一下，也是通过方法重载最终调用sendMessageAtTime()，然后调用enqueueMessage()，这个是Handler里的方法，在这个方法里面会调用queue.enqueueMessage()，这时，就会跳转到MessageQueue中的enqueueMessage()方法了。

MessageQueue：

MessageQueue 负责维护消息队列，插入消息和取出消息（具体的实现）。

MessageQueue是在哪里创建的，是在Looper中，待会我们细说。

我们先看enqueueMessage()方法：

![MessageQueue](C:\Users\wlk\Desktop\扔物线\picture\MessageQueue.png)

这个就是enqueueMessage()方法中最核心的代码，很简单，从头到尾遍历这个单链表，将 msg.next 设置为 null，再将 msg 放到链表的最末尾，当然特殊设置了 when 的话，会找到合适的位置将其插入。所以，这个单链表是有顺序的，它是按照处理时间顺序从近到远排序的。

* 这里简单介绍一下 when 这个字段，它是从系统开始的时间到调用这个方法的毫秒数 + delayMillis。

同时，它还会唤醒休眠的 Looper。

这就是将一个Message投放到队列的具体过程。我们下面分析取的过程。

取的动作是发生在 Looper 的 loop 中，它调用的是 MessageQueue 中的next()方法：

![next1](C:\Users\wlk\Desktop\扔物线\picture\next1.png)

![next2](C:\Users\wlk\Desktop\扔物线\picture\next2.png)

这个类中的逻辑很清晰，就是在MessageQueue中取出一个Message然后将他返回，我们看一看关键的地方：

* ptr 这是一个native code，如果为 0 的话，就会return null；这时 Looper 就会退出，我们在研究 loop 时就会看到。

OK，接下来看一看Looper：

Looper 负责不断的调用 MessageQueue 的 next() 方法取出消息并交给 Handler 处理。

![Looper](C:\Users\wlk\Desktop\扔物线\picture\Looper.png)

Looper的构造，绑定一个MessageQueue，绑定当前线程。

![prepare](C:\Users\wlk\Desktop\扔物线\picture\prepare.png)

Looper通过它初始化，要创建一个 Looper，都要先 prepare 一下，然后调用 loop 就可以开启一个 Looper 了。其中使用到的 ThreadLocal 我们前面已经介绍过了。我们使用 Looper.myLooper() 就可以得到当前线程的 Looper 了。接下来看一看 Looper 中最重要的方法，就是我们 loop 之后发生了什么：

![loop](C:\Users\wlk\Desktop\扔物线\picture\loop.png)

无限循环去拿消息，拿完消息就 dispatchMessage，dispatchMessage是 Handler 中的方法。这样，整套流程下来，就很自然完成了线程的切换。

在 dispatchMessage，完成了执行消息的过程：

![dispatchMessage](C:\Users\wlk\Desktop\扔物线\picture\dispatchMessage.png)

首先处理 msg 的 callback，这个 callback 就是一个 Runnable，它是使用 handler.post() 递交的消息队列里的。其次处理 handler 的 callback，最后处理 msg，我们可以在 handlerMessage() 中实现自己的处理逻辑。

Handler到此我也就介绍完了，其中还有很多知识点我没有说到，如 ThreadLocal 的内存泄漏，MessageQueue 的同步屏障，由于我能力有限，无法清晰的讲解，所以有兴趣的同学可以自己搜一搜。