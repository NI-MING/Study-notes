# ViewModel原理分析

ViewModel是MVVM中VM的重要实现，它承担了UI与数据的解耦和通信，我们可以在ViewModel中使用LivaData来达到数据更新实时通知界面的功能。

使用ViewModel有以下两个好处：

1. ViewModel的生命周期长于Activity，当Activity发生重建时可以保存我们UI层的数据。
2. 多个Activity或Fragment绑定同一个ViewModel，可以通过ViewModel来实现组件之间的数据共享。

首先来分析第一个好处：

ViewModel的存在时间范围是和获取到ViewModel传递的Lifecycle相关联的。也就是说，在Lifecycle彻底消失之前，ViewModel是一直存在在内存中。

这样就避免了内存泄漏，也是相比于MVP的一个好处之一。

我们先来讲ViewModel的创建，Activity中，我们会使用ViewModelProvider来获取ViewModel，它会创建一个AndroidViewModelFactory，这个类会从我们的Activity中获取NonConfigurationInstance（调用getLastNonConfigurationInstance），这个类包装了我们的ViewModelStore，ViewModel是一个存储ViewModel实例的HashMap，所以，ViewModel与页面没有直接的关联，页面只是从ViewModelStore中取出它想要的ViewModel而已。获取ViewModelStore是，获取到就直接返回，否则就新建一个返回。同理，当我们在ViewModelStore中获取ViewMdoel的缓存时，如果没有，就新建一个并将它put进ViewModelStore。

那么我们来回答第一个问题：Activity重建时如何恢复数据

Activity在发生资源改变时，执行onDestroy方法时会调用onRetrainNonConfigurationInstance方法，该方法会保存我们的NonConfigurationInstance，其中就有ViewModelStore，在重建时HandleLaunchActivity中的attach方法会通过getLastNonConfigurationInstance恢复NonConfigurationInstance。

至于第二个问题，多个组件间的数据共享，其实就是在HashMap中取到同一个ViewModel对象而已。

Activity以外被杀死ViewModel会保存吗？

不会，在HandleDestroyActivity中，会判断是否时资源变化导致销毁，如果不是就会执行ViewModelStore.clear()，所以不会保存我们的ViewModel。

