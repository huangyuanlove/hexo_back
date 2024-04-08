---
title: Retrofit流程分析
tags: [Retrofit,Android]
date: 2024-04-07 15:35:44
keywords: Android和Retrofit，Retrofit如何创建对象,Retrofit发送请求流程
---

在之前的文章中介绍了《在kotlin协程中使用自定义CallAdapter处理错误》，既然选择了它，当然得先全面了解一下。
先下载一下源码，搭建一下环境，也没啥好说的，我下载的是2.11.0 版本的代码，使用的 IDEA2023.3.6。这都是小事情，只要能有代码跳转功能就好。首次配置需要下载相应依赖，时间会长一些，这不重要。等配置完成后，找到 simple module，有各种各样的示例代码。是可以直接运行的。
<!-- more-->
## 创建Retrofit对象

我们先从如何创建的`Retrofit`对象开始
先看一下我们可以设置哪些参数，撸一下源码，找一下`Retrofit.Builder`类
### 设置`OkHttpClient`
这里的`OkHttpClient`实现了`Call.Factory`接口
``` Java
public Builder client(OkHttpClient client) {
  return callFactory(Objects.requireNonNull(client, "client == null"));
}
public Builder callFactory(okhttp3.Call.Factory factory) {
  this.callFactory = Objects.requireNonNull(factory, "factory == null");
  return this;
}
```

### 设置 baseUrl
``` Java
public Builder baseUrl(URL baseUrl) {
  Objects.requireNonNull(baseUrl, "baseUrl == null");
  return baseUrl(HttpUrl.get(baseUrl.toString()));
}
public Builder baseUrl(String baseUrl) {
  Objects.requireNonNull(baseUrl, "baseUrl == null");
  return baseUrl(HttpUrl.get(baseUrl));
}
public Builder baseUrl(HttpUrl baseUrl) {
  Objects.requireNonNull(baseUrl, "baseUrl == null");
  List<String> pathSegments = baseUrl.pathSegments();
  if (!"".equals(pathSegments.get(pathSegments.size() - 1))) {
    throw new IllegalArgumentException("baseUrl must end in /: " + baseUrl);
  }
  this.baseUrl = baseUrl;
  return this;
}
```
### 序列化和反序列化
``` Java
public Builder addConverterFactory(Converter.Factory factory) {
  converterFactories.add(Objects.requireNonNull(factory, "factory == null"));
  return this;
}
```
### 支持'Call'对象之外的返回类型
``` Java
public Builder addCallAdapterFactory(CallAdapter.Factory factory) {
  callAdapterFactories.add(Objects.requireNonNull(factory, "factory == null"));
  return this;
}
```
### 调用 callBack 时使用的Executor
``` Java
public Builder callbackExecutor(Executor executor) {
  this.callbackExecutor = Objects.requireNonNull(executor, "executor == null");
  return this;
}
```
### 是否提前验证接口中定义的方法
``` Java
public Builder validateEagerly(boolean validateEagerly) {
  this.validateEagerly = validateEagerly;
  return this;
}
```
### 调用 build()方法创建 Retrofit 对象
``` Java
public Retrofit build() {
  if (baseUrl == null) {
      throw new IllegalStateException("Base URL required.");
  }

  okhttp3.Call.Factory callFactory = this.callFactory;
  if (callFactory == null) {
      callFactory = new OkHttpClient();
  }

  Executor callbackExecutor = this.callbackExecutor;
  if (callbackExecutor == null) {
      callbackExecutor = Platform.callbackExecutor;
  }

  BuiltInFactories builtInFactories = Platform.builtInFactories;

  // Make a defensive copy of the adapters and add the default Call adapter.
  List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>(this.callAdapterFactories);
  List<? extends CallAdapter.Factory> defaultCallAdapterFactories =
  builtInFactories.createDefaultCallAdapterFactories(callbackExecutor);
  callAdapterFactories.addAll(defaultCallAdapterFactories);

  // Make a defensive copy of the converters.
  List<? extends Converter.Factory> defaultConverterFactories =
  builtInFactories.createDefaultConverterFactories();
  int defaultConverterFactoriesSize = defaultConverterFactories.size();
  List<Converter.Factory> converterFactories =
  new ArrayList<>(1 + this.converterFactories.size() + defaultConverterFactoriesSize);

  // Add the built-in converter factory first. This prevents overriding its behavior but also
  // ensures correct behavior when using converters that consume all types.
  converterFactories.add(new BuiltInConverters());
  converterFactories.addAll(this.converterFactories);
  converterFactories.addAll(defaultConverterFactories);

  return new Retrofit(
      callFactory,
      baseUrl,
      unmodifiableList(converterFactories),
      defaultConverterFactoriesSize,
      unmodifiableList(callAdapterFactories),
      defaultCallAdapterFactories.size(),
      callbackExecutor,
      validateEagerly);
}
```
1. 首先检查是否设置了 baseUrl，没设置直接抛异常
2. 设置callFactory，默认为OkHttpClient
3. 设置callbackExecutor默认为platform.defaultCallbackExecutor()，Android平台为MainThreadExecutor，其他平台为 null。这里的 AndroidMainExecutor 只是简单的使用 handler 转发到主线程
``` Java
final class AndroidMainExecutor implements Executor {
  private final Handler handler = new Handler(Looper.getMainLooper());

  @Override
  public void execute(Runnable r) {
      handler.post(r);
  }
}
``` 
1. callAdapterFactories 默认添加 platform.defaultCallAdapterFactories,返回值为DefaultCallAdapterFactory，之后在 kotlin 中使用密闭类代替 callback 时就是抄的这个类中的方法
2. converterFactories
  ○ 先添加 new BuiltInConverters()
  ○ 再添加自定义的
  ○ 最后添加platform.defaultConverterFactories() 默认是空的
这样就配置好的 Retrofit 对象

## 如何发送请求


我们在定义网络请求时是这么写的
### 定义网络请求
``` Java
  public interface GitHub {
    @GET("/repos/{owner}/{repo}/contributors")
    Call<List<Contributor>> contributors(@Path("owner") String owner, @Path("repo") String repo);
  }
``` 
### 创建**GitHub对象** 和发送请求
```Java
  // Create an instance of our GitHub API interface.
  GitHub github = retrofit.create(GitHub.class);
  // Create a call instance for looking up Retrofit contributors.
  Call<List<Contributor>> call = github.contributors("square", "retrofit");
  // Fetch and print a list of the contributors to the library.
  List<Contributor> contributors = call.execute().body();
```

重点在我们调用`retrofit.create`方法时用到的动态代理商
``` Java
  public <T> T create(final Class<T> service) {
    validateServiceInterface(service);
    return (T)
        Proxy.newProxyInstance(
            service.getClassLoader(),
            new Class<?>[] {service},
            new InvocationHandler() {
              private final Object[] emptyArgs = new Object[0];

              @Override
              public @Nullable Object invoke(Object proxy, Method method, @Nullable Object[] args)
                  throws Throwable {
                // If the method is a method from Object then defer to normal invocation.
                if (method.getDeclaringClass() == Object.class) {
                  return method.invoke(this, args);
                }
                args = args != null ? args : emptyArgs;
                Reflection reflection = Platform.reflection;
                return reflection.isDefaultMethod(method)
                    ? reflection.invokeDefaultMethod(method, service, proxy, args)
                    : loadServiceMethod(service, method).invoke(proxy, args);
              }
            });
  }
```



### validateServiceInterface

创建 Github 对象时使用了动态代理，不过在创建之前，先调用了`validateServiceInterface(service);`
``` Java
private void validateServiceInterface(Class<?> service) {
    if (!service.isInterface()) {
      throw new IllegalArgumentException("API declarations must be interfaces.");
    }

    Deque<Class<?>> check = new ArrayDeque<>(1);
    check.add(service);
    while (!check.isEmpty()) {
      Class<?> candidate = check.removeFirst();
      if (candidate.getTypeParameters().length != 0) {
        StringBuilder message =
            new StringBuilder("Type parameters are unsupported on ").append(candidate.getName());
        if (candidate != service) {
          message.append(" which is an interface of ").append(service.getName());
        }
        throw new IllegalArgumentException(message.toString());
      }
      Collections.addAll(check, candidate.getInterfaces());
    }

    if (validateEagerly) {
      Reflection reflection = Platform.reflection;
      for (Method method : service.getDeclaredMethods()) {
        if (!reflection.isDefaultMethod(method)
            && !Modifier.isStatic(method.getModifiers())
            && !method.isSynthetic()) {
          loadServiceMethod(service, method);
        }
      }
    }
  }
```
先检查 Github 类是不是 Interface，接着定义一个双端队列，对当前接口及其父接口进行递归检查，这里是不支持泛型参数的。
在接下来判断一下在创建 Retrofit 时传入的validateEagerly参数，如果是 true，并且声明的方法不是默认、不是静态、不是合成方法，则调用loadServiceMethod方法解析接口中声明的方法。

### loadServiceMethod
首先调用`ServiceMethod.parseAnnotations(this, service, method);`，先从缓存的map中获取有没有已经解析好的ServiceMethod。如果没有则调用ServiceMethod.parseAnnotations(this, method)方法进行解析。

#### 第一步创建requestFactory对象
调用了 `RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, service, method); `，解析了例如请求方式（即 POST 还是 GET 等请求），请求头，contentType 等等参数并返回了一个`RequestFactory`对象

#### 第二步创建HttpServiceMethod子类CallAdapted
接着通过`HttpServiceMethod`的静态方法`parseAnnotations`进一步解析，入参有`Retrofit`对象，`Method`对象和刚才创建的`RequestFactory `对象。`HttpServiceMethod`继承自`ServiceMethod`，但也是个抽象类，这个类提供的`parseAnnotations`方法的主要内容是进一步解析，并通过`createCallAdapter`创建适配器、通过`createResponseConverter`创建转换器，最后利用`RequestFactory`对象，`OkHttp`对象，`转换器`对象，`适配器`对象创建了一个`CallAdapted`类型的对象并返回。

##### 创建适配器
如果我们前面没有调用`addCallAdapterFactory(CallAdapter.Factory factory)`添加自定义的Factory的话，这里返回的是默认的`DefaultCallAdapterFactory`对象。
在该类的`adapt`方法中返回的`CallAdapter`对象中，先判断了`executor`是不是空，这里的`executor`就是在创建`Retrofit`时调用`public Builder callbackExecutor(Executor executor)`方法时传入的对象，前面也说过，在**Android(Dalvik)**平台上默认是`AndroidMainExecutor`,其他平台默认为空。

简单讲，`DefaultCallAdapterFactory`大致如下
``` Java
class DefaultCallAdapterFactory extends CallAdapter.Factory {
    @Override
  public @Nullable CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
    final Executor executor =
        Utils.isAnnotationPresent(annotations, SkipCallbackExecutor.class)
            ? null
            : callbackExecutor;

    return new CallAdapter<Object, Call<?>>() {
      @Override
      public Type responseType() {
        return responseType;
      }

      @Override
      public Call<Object> adapt(Call<Object> call) {
        return executor == null ? call : new ExecutorCallbackCall<>(executor, call);
      }
    };
  }
}

static final class ExecutorCallbackCall<T> implements Call<T> {
   ExecutorCallbackCall(Executor callbackExecutor, Call<T> delegate) {
      this.callbackExecutor = callbackExecutor;
      this.delegate = delegate;
    }
    //。。。。一系列的方法，除了enqueue(final Callback<T> callback)方法外，都是调用delegate中对应的方法
}
```
##### 创建转换器

我们在使用的时候一般会传入一个ConverterFactory对象，比如MoshiConverterFactory、GsonConverterFactory等。最主要的就两个方法
`responseBodyConverter` and `requestBodyConverter`.在调用`HttpServiceMethod.createResponseConverter`时，兜兜转转最终还是调用了 `Retrofit.nextCallAdapter`方法，从我们一开始构建的 Retrofit 对象中查找对应的转换器

##### 创建CallAdapted对象
这里直接调用的构造方法
``` Java
return new CallAdapted<>(requestFactory, callFactory, responseConverter, callAdapter);
```
注意这里传入的`callAdapter`对象就是前面提到的调用`createCallAdapter`返回的默认`DefaultCallAdapterFactory`示例。
`CallAdapted`类是`HttpServiceMethod`的子类，实现了父类的抽象方法`adapt`。而`HttpServiceMethod`又实现了`ServiceMethod`的抽象方法`invoke`，在`invoke`方法里调用了`adapt`方法。

#### loadServiceMethod过程总结
所以前面说到的加载过程(loadServiceMethod)，最终就是返回了一个`CallAdapted`类型的对象，并存到缓存中。接下去就是调用了`CallAdapted`对象的`invoke`方法，显然最终调用了`CallAdapted`自身的`adapt`方法。`CallAdapted`提供的`adapt`方法里就一句，那就是调用适配器的`adapt`方法，并返回一个值。这个值就是我们定义的接口类型的代理对象.之后调用调用定义的接口方法获取到Call对象，调用enqueue异步执行；调用execute同步执行；


### invoke过程
前面知道了`loadServiceMethod`返回的是一个`CallAdapted`对象，然后紧接着调用了`invoke`方法。但`CallAdapted`并没有重写该方法，所以实际上还是调用的`HttpServiceMethod`类中的`invoke`
``` Java
  @Override
  final @Nullable ReturnT invoke(Object instance, Object[] args) {
    Call<ResponseT> call =
        new OkHttpCall<>(requestFactory, instance, args, callFactory, responseConverter);
    return adapt(call, args);
  }
```
![](image/Android/retrofit/NewOkHttpCall.png)
![](image/Android/retrofit/NewOkhttpCallArgs.png)

接着调用了`adapt(call, args)`方法，这个方法就需要子类实现了，这里的子类是`CallAdapted`,在`CallAdapted`中接着调用
``` Java
@Override
protected ReturnT adapt(Call<ResponseT> call, Object[] args) {
  return callAdapter.adapt(call);
}
```
该方法中的`callAdapter`对象就是前面提到的`DefaultCallAdapterFactory.get`方法中返回的`CallAdapter`,调用了该对应的`adapt`方法。在上面的介绍的**创建适配器**小结中提到，在**Android(Dalvik)**平台上默认有`AndroidMainExecutor`,其他平台默认为空。所以在没有额外添加`callbackExecutor`的情况下，Android 平台上返回的是`ExecutorCallbackCall`,在其他平台上默认返回的就是参数中的`call`对象，也就是`retrofit2.OkHttpCall`。 这个对象也就是我们调用接口中的方法返回的对象。

#### 发送请求
上面提到在 Android 平台上会返回`ExecutorCallbackCall`对象，其他平台返回`retrofit2.OkHttpCall`对象。但在创建`ExecutorCallbackCall`
对象的时候也会将`retrofit2.OkHttpCall`传进去，在调用各种方法的时候还是调用的`retrofit2.OkHttpCall`对象的方法，只不过在调用`void enqueue(Callback<T> callback);`方法的`callback`回调中使用`callbackExecutor`将回调切换回主线程而已。

在调用`retrofit2.OkHttpCall`的`enqueue`或者`execute`方法时，会调用`createRawCall`方法创建一个`okhttp3.Call`对象，实际上的网络请求还是 okhttp 发出的。
``` Java
  private okhttp3.Call createRawCall() throws IOException {
    okhttp3.Call call = callFactory.newCall(requestFactory.create(instance, args));
    if (call == null) {
      throw new NullPointerException("Call.Factory returned null.");
    }
    return call;
  }
```
这里的`requestFactory`是上面提到的`parseAnnotations`时创建的`RequestFactory`对象，包含了部分请求信息。这里又用它创建了`okhttp3.Request`对象。创建好之后接着就创建了`okhttp3.Call`对象，接下来就是调用`enqueue`或者`execute`方法发送请求。
当请求数据返回时，会调用`parseResponse(okhttp3.Response rawResponse)`方法，最终调用的是`responseConverter.convert(catchingBody)`方法，这里的`responseConverter`就是我们在创建`Retrofit`对象调用`addConverterFactory`时传入的解析方法.
至此，完成了一次网络请求。

----
以上