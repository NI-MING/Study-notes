# Retrofit原理分析

Retrofit是对OkHttp的二次封装，它通过大量的设计模式，使框架的逻辑变得更加的清晰，同时使用注解的方式，简化了OkHttp的使用，是一个符合RESTFUL风格的网络请求开源库。

Retrofit的使用这里不进行介绍，可以查阅官方文档或博客。

Retrofit使用了大量的设计模式，最主要的就是动态代理，它是在create方法中。

```kotlin
public <T> T create(final Class<T> service) {
  validateServiceInterface(service);
  return (T)
      Proxy.newProxyInstance(
          service.getClassLoader(),
          new Class<?>[] {service},
          new InvocationHandler() {
            private final Platform platform = Platform.get();
            private final Object[] emptyArgs = new Object[0];

            @Override
            public @Nullable Object invoke(Object proxy, Method method, @Nullable Object[] args)
                throws Throwable {
              // If the method is a method from Object then defer to normal invocation.
              if (method.getDeclaringClass() == Object.class) {
                return method.invoke(this, args);
              }
              args = args != null ? args : emptyArgs;
              return platform.isDefaultMethod(method)
                  ? platform.invokeDefaultMethod(method, service, proxy, args)
                  : loadServiceMethod(method).invoke(args);
            }
          });
}
```

这个service是我们写的网络请求方法的接口，create中首先会判断我们传入的参数是否为我们接口的class，接着就会使用动态代理模式，来代理我们接口中的所有方法。

在invocatHandler中，重写了invoke方法，在其中调用`loadServiceMethod(method).invoke(args)`，`loadServiceMethod`方法中，会先构建出出一个`ServiceMethod`对象，一个`ServiceMethod`对象对应于网络请求接口中的一个方法，在构造ServiceMethod时，他根据接口方法的返回值和注解,获得网络请求适配器，数据转换器，之后解析接口中的参数的注解。来一起构成一个ServiceMethod。在ServiceMethod几乎保存了一个网络请求所需要的数据。

在之后的invoke中，创建出了okhttpCall对象，将其传入adapt方法中，根据网络请求适配器工厂来确定返回的对象，如果是Retrofit默认的网络请求适配器，就返回一个Call对象，如果是Rxjava，就返回一个observable对象。同时还对回调方法做了线程切换。enqueue方法其实就是执行okhttpCall的enqueue方法，这就到了okhttp中的逻辑了。

Retrofit中的设计模式：

建造者模式：构建我们的Retrofit外观类。

外观模式：将复杂系统功能集成到一个外观对象中，外观对象负责暴露出所有的系统方法，屏蔽了内部的逻辑。

动态代理：代理我们接口中的方法

工厂模式：数据转换工厂

适配器模式：通过适配器来适配各个平台

