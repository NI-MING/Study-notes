# Service

## 什么是Service

Service是一个可以在后台执行长时间运行操作而不提供用户界面的应用组件。Service不依赖于任何用户界面，即使App被切到后台，Service仍可以正常运行。当某个程序进程被杀掉时，所有依赖于该进程的Service也会停止。

## Service与Thread的区别

线程是程序执行的最小单位，用来处理异步操作。

Service是Android提供的一种机制，Service运行于主线程中，是Context的子类，可以调用Context的所有方法，通过startService，stopService，bindService，unbindService来控制。由于Service默认运行在主线程中，所以不可以处理耗时任务，否则可能导致ANR。除非将Service放到子线程中执行，或者使用IntentService。

## Service的生命周期

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/af0927004qroiaQKuGI3mJPBUlmEklIImKibHYYpY3gGCpW3ODyTQLwUicicgwrcsoXibXFdNKYPKIL2qPgicNyDnbKw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

* 通过startService启动一个Serive，并且回调onStartCommand方法，如果服务尚未创建，则先回调onCreate方法，在调用onStartCommand方法。一个服务onCreate方法只会调用一次，而onStartCommand方法会调用多次（取决于startService次数）。启动之后，服务会一直处于运行状态，即使启动服务的组件销毁，Service也不会停止，除非进程结束，或者调用stopService，又或者自身调用stopSelf。服务停止并回调onDestroy方法。
* 通过bindService绑定一个服务，并且回调服务中的onBind方法。如果服务尚未创建，则先回调onCreate方法，在调用onbind方法。onbind方法中，返回一个IBinder对象实例，便于实现服务与组件之间的通信。

这两种启动方式并不冲突，当使用了startService启动Serivce之后，还可以再使用bindService绑定，只不过需要同时调用stopService和unbindService方法才能销毁服务。

## 普通Serive

启动方式：

```xml
// 在AndroidManifest.xml中注册
<service
    android:name=".MyService"
    android:enabled="true"
    android:exported="true">
</service>
```

```java
// 启动
Intent intent = new Intent(MainActivity.this,MyService.class);
startService(intent);

// 关闭
Intent intent = new Intent(MainActivity.this,MyService.class);
stopService(intent);
```

```java
public class MyService extends Service {
    public MyService() {
    }

    @Override
    public void onCreate() {
        super.onCreate();
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        
        // doSomething
        
        stopSelf();
        return super.onStartCommand(intent, flags, startId);
    }
    

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }


}
```

如果想要与其他组件交互如Activity：

```java
public class MyService extends Service {
    public MyService() {
    }

    MyBinder binder = new MyBinder();
    
    @Override
    public IBinder onBind(Intent intent) {
        return binder;
    }

    static class MyBinder extends Binder{

        public void doSomething(){}

    }

}
```

```java
private MyService.MyBinder binder;
    private ServiceConnection connection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            binder = (MyService.MyBinder) service;

        }

        @Override
        public void onServiceDisconnected(ComponentName name) {

        }
    };

// 绑定服务
bindService(intent,connection, Context.BIND_AUTO_CREATE);
// 交互
binder.doSomething();
// 解绑
unbindService(connection);
```

## 前台Service

前台Service一直会有一个正在运行的图标在系统的状态栏中，非常类似通知的效果。前台服务为了防止服务被回收掉。

要实现前台Service非常简单，先构建一个Notification之后，不需要NotificationManager将通知显示出来，而是调用startForeground方法。

## IntentService

IntentService可以自动创建一个异步的、会自动停止的服务。

新建类并继承IntentService

```java
public class MyIntentService extends IntentService {

    /**
     * Creates an IntentService.  Invoked by your subclass's constructor.
     *
     * @param name Used to name the worker thread, important only for debugging.
     */
    public MyIntentService(String name) {
        super(name);
    }

    @Override
    protected void onHandleIntent(@Nullable Intent intent) {

    }
}
```

在Activity中启动就与普通Service无异。

