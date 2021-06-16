# ARouter原理分析

https://juejin.cn/post/6995136681850437662

ARouter使用注解与APT技术，自动生成代码，在运行时调用。我们分为两部分学习，生成的代码和ARouter源代码。

我们想要在模块A中的Activity跳转到模块B中的Activity，只要得到了模块B的Activity.class就可以在模块A中使用intent跳转，而问题是当我们使用组件化开发时，这两个模块是没有任何依赖关系的，无法获取模块B的类。因此，问题的难点在于如何全局获取相互不依赖的类文件。



**编译期干的事—中间代码生成**

apt技术：apt技术就是先设定一套代码模板，然后在类加载期间，动态根据指定的代码模板生成一个.java文件，这个文件在运行时可以直接访问。

当我们在gralde中添加Arouter依赖后，那么在编译期就会在对应的module的/build/generated/source/kapt/debug/下生成com.alibaba.android.arouter.routes目录，Arouter生成的代码都放在这里。

编译期会生成很重要的类，该类实现了IRouteRoot接口，重写了loadInto方法，此方法的参数是一个Map类型，该Map就是存储了我们路径path与.class文件的映射关系。



```java
// 这一个IRouteRoot，看名字"ARouter$$Root$$app"，其中"ARouter$$Root"是前缀，"app"是group名字，也就是path里面以"/"分隔得到的第一个字符串，然后通过"$$"连接，
// 那么这玩意儿的完整类名就是"com.alibaba.android.arouter.routes.ARouter$$Root$$app"
public class ARouter$$Root$$app implements IRouteRoot {

    // 参数是一个map，value类型是 IRouteGroup
    @Override
    public void loadInto(Map<String, Class<? extends IRouteGroup>> routes) {
        // "app"就是@Route(path = "/app/activity_main") 中的"app"，在Arouter中叫做group，是以path中的"/"分隔得到的
        // 这个的value是:ARouter$$Group$$app.class，也就是下面的类
        routes.put("app", ARouter$$Group$$app.class);
    }
}

// 这是一个IRouteGroup，同理，前缀是"ARouter$$Group"
public class ARouter$$Group$$app implements IRouteGroup {
    @Override
    public void loadInto(Map<String, RouteMeta> atlas) {
        // "app/activity_main"就是我们通过@Route指定的path，后面RouteMeta保存了要启动的组件类型，以及对应的.class文件
        // 这个 RouteMeta.build()的参数很重要，后面要用到
        atlas.put("/app/activity_main", RouteMeta.build(RouteType.ACTIVITY, MainActivity.class, "/app/activity_main", "app", null, -1, -2147483648));
    }
}

// RouteMeta.build()方法，参数后面有用
// type就是: RouteType.ACTIVITY，
// destination就是MainActivity.class，
// path就是"/app/activity_main",
// group就是"app"
// paramsType是null
// priority是-1
// extra是-2147483648
public static RouteMeta build(RouteType type, Class<?> destination, String path, String group, Map<String, Integer> paramsType, int priority, int extra) {
    return new RouteMeta(type, null, destination, null, path, group, paramsType, priority, extra);
}

```



**运行时干的事—注入**

在Application中执行初始化操作：

```java
Arouter.init(this)
```

```java
// 调到了这里
protected static synchronized boolean init(Application application) {
    // 保存了mContext，后面有用
    mContext = application;

    // 初始化
    LogisticsCenter.init(mContext, executor);
    hasInit = true;
    mHandler = new Handler(Looper.getMainLooper());

    return true;
}

```

继续：

```java
public synchronized static void init(Context context, ThreadPoolExecutor tpe) throws HandlerException {
        mContext = context;
        executor = tpe;
        try {
            Set<String> routerMap;
            
            // 如果是debugable()或者更新了app的版本
            if (ARouter.debuggable() || PackageUtils.isNewVersion(context)) {
                // 那么就会重新获取所有的class，所以，当你的Arouter出现了route not found时候，更新版本号 或者 开启Arouter的debug就ok了。

                // 这里会获取所有"com.alibaba.android.arouter.routes"目录下的class文件的类名。
                routerMap = ClassUtils.getFileNameByPackageName(mContext, ROUTE_ROOT_PAKCAGE);

                // 这里缓存到SharedPreferences里面，方便下次获取。
                if (!routerMap.isEmpty()) {
                    context.getSharedPreferences(AROUTER_SP_CACHE_KEY, Context.MODE_PRIVATE).edit().putStringSet(AROUTER_SP_KEY_MAP, routerMap).apply();
                }

                // 将app的版本号缓存到SharedPreferences，方便下次使用。
                PackageUtils.updateVersion(context);
            } else {
                // 如果版本号没有更新，并且没开启debug，则从缓存中取出之前缓存的所有class
                routerMap = new HashSet<>(context.getSharedPreferences(AROUTER_SP_CACHE_KEY, Context.MODE_PRIVATE).getStringSet(AROUTER_SP_KEY_MAP, new HashSet<String>()));
            }

            // 遍历刚刚拿到的所有类名，并且反射调用它们的loadInto()方法，那么app module中的那些生成的类，它们的loadinto()就被调用了，并且注入到参数里面了。
            for (String className : routerMap) {
                // 拼接的字符串其实就是"com.alibaba.android.arouter.routes.ARouter$$Root"，这不就是编译时生成的那个"ARouter$$Root$$app"的前缀吗。
                if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_ROOT)) {
                    // 于是，调用了它的loadInto(map)，也就等价于调用了:map.put("app", ARouter$$Group$$app.class)，这个键值对 就放在了Warehouse.groupsIndex里面。
                    ((IRouteRoot) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.groupsIndex);

                    // 这个是"com.alibaba.android.arouter.routes.ARouter$$Interceptors"
                } else if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_INTERCEPTORS)) {
                    ((IInterceptorGroup) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.interceptorsIndex);

                    // 这个是"com.alibaba.android.arouter.routes.ARouter$$Providers"
                } else if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_PROVIDERS)) {
                    ((IProviderGroup) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.providersIndex);
                }
            }
        } catch (Exception e) {
            throw new HandlerException(TAG + "ARouter init logistics center exception! [" + e.getMessage() + "]");
        }
    }

```
根据上述，我们可以了解到：

1. 获取到所有编译后生成的文件路径，并进行缓存。
2. 遍历文件下的所有类名，调用loadinto(map)方法。
3. 调用的结果就是将所有的（key，group.class）存入Warehuose.groupsIndex里。

我们来看一下WareHouse：

```java
class Warehouse {
    // Cache route and metas

    // 这个就是我们刚刚注入的那个map，果然接收一个IRouteGroup，对上了。
    static Map<String, Class<? extends IRouteGroup>> groupsIndex = new HashMap<>();
    static Map<String, RouteMeta> routes = new HashMap<>();

    // Cache provider
    static Map<Class, IProvider> providers = new HashMap<>();
    static Map<String, RouteMeta> providersIndex = new HashMap<>();

    // Cache interceptor
    static Map<Integer, Class<? extends IInterceptor>> interceptorsIndex = new UniqueKeyTreeMap<>("More than one interceptors use same priority [%s]");
    static List<IInterceptor> interceptors = new ArrayList<>();

    static void clear() {
        routes.clear();
        groupsIndex.clear();
        providers.clear();
        providersIndex.clear();
        interceptors.clear();
        interceptorsIndex.clear();
    }
}
```

**运行时总结：**

1. 在Application的onCreate()里面我们调用了Arouter.init(this)。
2. 接着调用了ClassUtils.getFileNameByPackageNam()来获取所有"com.alibaba.android.arouter.routes"目录下的dex文件的路径。
3. 然后遍历这些dex文件获取所有的calss文件的完整类名。
4. 然后遍历所有类名，获取指定前缀的类，然后通过反射调用它们的loadInto(map)方法，这是个注入的过程，都注入到参数Warehouse的成员变量里面了。
5. 其中就有Arouter在编译时生成的"com.alibaba.android.arouter.routes.ARouter$$Root.ARouter$$Root$$app"类，它对应的代码:<"app", ARouter$$Group$$app.class>就被添加到Warehouse.groupsIndex里面了。

好，现在我们的注入过程就完事了，说白了就是: **app包名 -> 获取.dex -> 获取.class -> 找对应的.class -> 反射调用方法 -> 存入Warehouse中**，这个过程就是**注入**，Warehouse就是仓库，里面保存了需要的key和.class，好，我们来看**获取的过程**。

调用的代码很简单:

```java
// 这里的path就是我们通过@Route指定的，也就是"/app/activity_main"
ARouter.getInstance().build(path).navigation();

```

```java
// 核心函数1
public synchronized static void completion(Postcard postcard) {
    if (null == postcard) {
        throw new NoRouteFoundException(TAG + "No postcard!");
    }

    // 先从Warehouse.routes里面获取RouteMeta，我们上述代码的经历只用到了Warehouse.groupsIndex，所以肯定是null

    // 第二次过来了，现在Warehouse.routes有值了，就是根据path拿到的。
    RouteMeta routeMeta = Warehouse.routes.get(postcard.getPath());
    if (null == routeMeta) {
        
        // 接着跑这里，从Warehouse.groupsIndex获取IRouteGroup的class！终于用到我们前面注入的玩意儿了，postcard.getGroup()就是"app"，
        // 而我们前面调过 Warehouse.groupsIndex.put("app", ARouter$$Group$$app.class)，这里就直接取出来了。
        Class<? extends IRouteGroup> groupMeta = Warehouse.groupsIndex.get(postcard.getGroup());

        if (null == groupMeta) {
            throw new NoRouteFoundException(TAG + "There is no route match the path [" + postcard.getPath() + "], in group [" + postcard.getGroup() + "]");
        } else {
            try {
                // 开始反射了
                IRouteGroup iGroupInstance = groupMeta.getConstructor().newInstance();
                
                // 调了loadInfo，我们回去看下ARouter$$Group$$app.class的loadInto方法:
                // atlas.put("/app/activity_main", RouteMeta.build(RouteType.ACTIVITY, MainActivity.class, "/app/activity_main", "app", null, -1, -2147483648));
                // 直接put了，key是"/app/activity_main"，跟Postcard的path一样，这下就放在Warehouse.routes里面了，下次就能拿到了。
                iGroupInstance.loadInto(Warehouse.routes);

                // 把group扔掉，group的意义就是用来拿route的，现在拿到了已经没用了，删除省内存。
                Warehouse.groupsIndex.remove(postcard.getGroup());
            } catch (Exception e) {
                throw new HandlerException(TAG + "Fatal exception when loading group meta. [" + e.getMessage() + "]");
            }

            // 又调了自己，回到这个函数头重新看
            completion(postcard);
        }
    } else {
        // 第二次进来，跑这里，设置一堆属性，还记得很重要的那一堆参数吗
        // RouteMeta.build(RouteType.ACTIVITY, MainActivity.class, "/app/activity_main", "app", null, -1, -2147483648)
        postcard.setDestination(routeMeta.getDestination()); // MainActivity.class
        postcard.setType(routeMeta.getType()); // RouteType.ACTIVITY
        postcard.setPriority(routeMeta.getPriority()); // -1
        postcard.setExtra(routeMeta.getExtra()); // -2147483648


        // uri是null，不用看
        Uri rawUri = postcard.getUri();
        if (null != rawUri) {   // Try to set params into bundle.
            Map<String, String> resultMap = TextUtils.splitQueryParameters(rawUri);
            Map<String, Integer> paramsType = routeMeta.getParamsType();
            if (MapUtils.isNotEmpty(paramsType)) {
                for (Map.Entry<String, Integer> params : paramsType.entrySet()) {
                    setValue(postcard,
                            params.getValue(),
                            params.getKey(),
                            resultMap.get(params.getKey()));
                }

                postcard.getExtras().putStringArray(ARouter.AUTO_INJECT, paramsType.keySet().toArray(new String[]{}));
            }

            postcard.withString(ARouter.RAW_URI, rawUri.toString());
        }

        // 根据类型执行逻辑，我们的类型是RouteType.ACTIVITY，下面好像都没有
        switch (routeMeta.getType()) {
            case PROVIDER: 
                Class<? extends IProvider> providerMeta = (Class<? extends IProvider>) routeMeta.getDestination();
                IProvider instance = Warehouse.providers.get(providerMeta);
                if (null == instance) { // There's no instance of this provider
                    IProvider provider;
                    try {
                        provider = providerMeta.getConstructor().newInstance();
                        provider.init(mContext);
                        Warehouse.providers.put(providerMeta, provider);
                        instance = provider;
                    } catch (Exception e) {
                        throw new HandlerException("Init provider failed! " + e.getMessage());
                    }
                }
                postcard.setProvider(instance);
                postcard.greenChannel();
                break;
            case FRAGMENT:
                postcard.greenChannel();
            default:
                break;
        }
    }
}

```



**总结：ARouter是通过APT技术+反射，实现了将路径与类文件存储在同一个全局Map中，并在调用时通过路径找到对应的类文件，从而使用intent来进行activity的启动。**

**在编译期，APT生成一系列的类文件，以ARouter前缀开头，继承自IRouteGroup，重写loadInto方法，该方法主要是将键值对加载进指定的Map（Root类是将<group，group.class>加载进集合（Warehouse.groupIndex），Group类是将<path，类文件>加载进集合（Warehouse.route））。**

**在运行期初始化，从Application.onCreate调用init方法，该方法先从sp中找有没有缓存过的routeMap集合，没有就调用ClassUtils.getFileNameByPackageName方法遍历dex文件下的所有class类，然后反射将<group，group.class>添加到Warehouse.groupIndex中。**

**在调用跳转逻辑时，先调用build生成PostCard，这个类通过path，保存了group，path，通过group在Warehouse.groupIndex集合中找到group.class，然后反射调用该类中的loadInto方法，将<path,类文件>加载进Warehouse.route集合中，之后通过path找到这个类文件，然后使用Intent调用startActivity跳转。**