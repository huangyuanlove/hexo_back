---
title: 在kotlin协程中使用自定义CallAdapter处理错误
tags: [Kotlin,Retrofit]
date: 2024-03-29 18:14:05
keywords: Kotlin,Retrofit,suspend,CallAdapter
---

## 前言
Retrofit在2019-06-05发布的2.6.0版本中就已经支持Kotlin 中的 suspend修饰符了，目前正准备在项目中使用 Kotlin，顺便替换一下网络库。这里先做一下调研和基础建设，方便后续的接入工作。
问就是15 年 16 年的老项目，之前并没有使用 Kotlin 的打算。

Retrofit已经在2024-03-28更新到2.11.0版本了，就用这个来做调研好了。
### 添加依赖
``` groovy
implementation 'com.squareup.retrofit2:retrofit:2.11.0'
implementation 'com.squareup.retrofit2:converter-gson:2.11.0'
implementation 'com.squareup.okhttp3:okhttp:4.12.0'
```
由于需要对 OKHttpClient 做一些操作和定制，这里添加了 okhttp 的依赖。实体类的转换使用了 gson，为啥用 gson，问就是项目里面就是用的 gson，后面再介绍一下其他的converter。
* Gson: com.squareup.retrofit2:converter-gson
* Jackson: com.squareup.retrofit2:converter-jackson
* Moshi: com.squareup.retrofit2:converter-moshi
* Protobuf: com.squareup.retrofit2:converter-protobuf
* Wire: com.squareup.retrofit2:converter-wire
* Simple XML: com.squareup.retrofit2:converter-simplexml
* JAXB: com.squareup.retrofit2:converter-jaxb
* Scalars (primitives, boxed, and String): com.squareup.retrofit2:converter-scalars
  
### 声明请求接口
``` Kotlin
interface MainPageApi{
  @GET("app_interface/home_pag/")
  fun getMainPageInfoWithRow():Call<MainPageInfo>
}
```

### 创建 Retrofit 对象

``` Kotlin
val retrofit = Retrofit.Builder()
    .baseUrl(BASE_URL)
    .addConverterFactory(GsonConverterFactory.create())
    .build()
```
### 发送请求

``` Kotlin
val mainPageApi = retrofit.create(MainPageApi::class.java)
mainPageApi.getMainPageInfoWithCall().enqueue(object:retrofit2.Callback<MainPageInfo>{
    override fun onResponse(
        call: Call<MainPageInfo>,
        response: retrofit2.Response<MainPageInfo>
    ) {
        Log.e("KotlinActivity","getMainPageInfoWithCall onResponse")
    }

    override fun onFailure(call: Call<MainPageInfo>, t: Throwable) {
        Log.e("KotlinActivity","getMainPageInfoWithCall onFailure")
    }
})
```
到这里为止，我们还没有使用任何协程相关的特性，并且没有都得写回调，和 Java 写起来也没啥差别。

### 支持协程
我们对接口的声明加上`suspend`修饰
``` Kotlin
@GET("app_interface/home_pag/")
suspend fun getMainPageInfoWithRow():Call<MainPageInfo>
```
这时候上面直接发送请求的代码会报错：
![suspend_retrofit_error](image/Android/kotlin/suspend_retrofit_error.png)
提示我们需要在协程中调用，这也简单，kotlin 对 activity 有个扩展的`lifecycleScope`成员变量，稍微修改一下：
``` Kotlin
lifecycleScope.launch(Dispatchers.IO) {
  mainPageApi.getMainPageInfoWithCall().enqueue(.....)
}
```
不习惯这么写的话，可以将网络请求写在 ViewModel 中，通过 LiveData创建一个可观察对象实现数据绑定。

不出意外的出意外了，应用崩溃，错误信息
``` xml
java.lang.IllegalArgumentException: Suspend functions should not return Call, as they already execute asynchronously.
Change its return type to class com.huangyuanlove.androidtest.kotlin.retrofit.MainPageInfo
```
意思是在协程中发起请求已经是异步的了，不需要再返回 Call 对象了，直接返回对应的实体即可。
简单，修改一下接口声明
``` Kotlin
@GET("app_interface/home_page/")
suspend fun getMainPageInfoWithRow():MainPageInfo
```
然后修改一下请求
``` Kotlin
lifecycleScope.launch(Dispatchers.IO) {
  val mainPageInfo = mainPageApi.getMainPageInfo()
  withContext(Dispatchers.Main) {
    refreshUI(mainPageInfo)
  }
}
```
运行一下，一切正常。我们修改一下接口，请求一个不存在的地址，会返回404，不出意外，应用还是崩溃
``` xml
retrofit2.HttpException: HTTP 404 
at retrofit2.KotlinExtensions$await$2$2.onResponse(KotlinExtensions.kt:53)
at retrofit2.OkHttpCall$1.onResponse(OkHttpCall.java:164)
at okhttp3.internal.connection.RealCall$AsyncCall.run(RealCall.kt:519)
at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
at java.lang.Thread.run(Thread.java:929)
Suppressed: kotlinx.coroutines.internal.DiagnosticCoroutineContextException: [StandaloneCoroutine{Cancelling}@ffa6ad2, Dispatchers.IO]
```
哦~异常没有处理，粗暴点，直接 try-catch，kotlin 中还有`runCatching`这个语法糖
``` Kotlin
val mainPageInfoRow = runCatching { mainPageApi.getMainPageInfoWithRow() }
if (mainPageInfoRow.isFailure) {
    ToastUtils.showToast("请求失败")
} else if (mainPageInfoRow.isSuccess) {
    ToastUtils.showToast("请求成功")
    withContext(Dispatchers.Main) {
        if (mainPageInfoRow.getOrNull() == null) {
            ToastUtils.showToast("请求结果为空")
        } else {
            refreshViewWithLaunch(mainPageInfoRow.getOrNull()!!)
        }

    }
}
```

但是有时候我们会用`HTTP状态码`来表示一些业务上逻辑错误，并且不同的状态码返回的 JSON 结构还可能不一样。 别问为啥要这么搞，应该是HTTP 状态码就应该表示网络请求的状态，业务状态应该放在返回的数据中约定字段来处理。问就是15年的老代码，之前就是这么搞的，并且大范围应用，涉及到的部门、业务占半数以上。
这时候我们需要自定义`CallAdapter`了

### 自定义 CallAdapter
这时候就应该翻一下源码了，在`example`有个`ErrorHandlingAdapter.java`,路径在[samples/src/main/java/com/example/retrofit/ErrorHandlingAdapter.java](https://github.com/square/retrofit/blob/trunk/samples/src/main/java/com/example/retrofit/ErrorHandlingAdapter.java)。
我们来仿写一下，最关键的点在实现自己的 Call 类的时候，对callback 的处理。

#### 定义不同的返回状态
第一步，创建密闭类，来表示不同的状态，这里暂且定义了三种情况
* Success:HTTP状态码在`[200,300)`这个区间
* NetError:HTTP状态码不在`[200,300)`这个区间
* UnknownError:其他错误

sealed class NetworkResponse<out T : Any, out U : Any> {
    data class Success<T : Any>(val body: T) : NetworkResponse<T, Nothing>()
    data class NetError(val httpCode:Int?,val errorMsg:String?,val exception: Throwable?) : NetworkResponse<Nothing, Nothing>()
    data class UnknownError(val error: Throwable?) : NetworkResponse<Nothing, Nothing>()
}

#### 创建自己的Call类
这里为了简化方便，除了`enqueue`之外必须重写的方法，都是直接调用`delegate`对应的方法

``` Kotlin
internal class NetworkResponseCall<S : Any, E : Any>(
    private val delegate: Call<S>,
    private val errorConverter: Converter<ResponseBody, E>
) : Call<NetworkResponse<S, E>> {
    override fun clone(): Call<NetworkResponse<S, E>> {
        return NetworkResponseCall(delegate.clone(), errorConverter);
    }

    override fun execute(): Response<NetworkResponse<S, E>> {
        throw UnsupportedOperationException("NetworkResponseCall doesn't support execute")
    }

    override fun isExecuted(): Boolean {
        return delegate.isExecuted;
    }

    override fun cancel() {
        delegate.cancel()
    }

    override fun isCanceled(): Boolean {
        return delegate.isCanceled
    }

    override fun request(): Request {
        return delegate.request()
    }

    override fun timeout(): Timeout {
        return delegate.timeout();
    }
}
```
下面是关键的`enqueue`方法,在这里面，将所有的请求都用`Response.success`返回，不再走`Response.error`.并且根据不同的 HTTP 状态码，返回的数据等条件转成一开始定义的密闭类。
``` Kotlin

override fun enqueue(callback: Callback<NetworkResponse<S, E>>) {
    return delegate.enqueue(object : Callback<S> {
        override fun onResponse(call: Call<S>, response: Response<S>) {
            val body = response.body()
            val code = response.code()
            val error = response.errorBody()

            if (response.isSuccessful) {
                if (body != null) {
                    callback.onResponse(
                        this@NetworkResponseCall,
                        Response.success(NetworkResponse.Success(body))
                    )
                } else {
                    
                    callback.onResponse(
                        this@NetworkResponseCall,
                        Response.success(NetworkResponse.UnknownError(null))
                    )
                }
            } else {
                val errorBody = when {
                    error == null -> null
                    error.contentLength() == 0L -> null
                    else -> NetworkResponse.NetError(code, error.toString(), null)
                }
                if (errorBody != null) {
                    callback.onResponse(
                        this@NetworkResponseCall,
                        Response.success(errorBody)
                    )
                } else {
                    callback.onResponse(
                        this@NetworkResponseCall,
                        Response.success(NetworkResponse.UnknownError(null))
                    )
                }
            }


        }

        override fun onFailure(call: Call<S>, t: Throwable) {
            val networkResponse = when (t) {
                is Exception -> NetworkResponse.NetError(null,null,t)
                else -> NetworkResponse.UnknownError(t)
            }
            callback.onResponse(this@NetworkResponseCall, Response.success(networkResponse))
        }

    })
}
```

#### 创建 CallAdapter
``` Kotlin
class NetworkResponseAdapter<S : Any, E : Any>(
    private val successType: Type,
    private val errorBodyConverter: Converter<ResponseBody, E>
) : CallAdapter<S, Call<NetworkResponse<S, E>>> {

    override fun responseType(): Type = successType

    override fun adapt(call: Call<S>): Call<NetworkResponse<S, E>> {
        return NetworkResponseCall(call, errorBodyConverter)
    }
}
```

#### 创建CallAdapterFactory
``` Kotlin
class  NetworkResponseAdapterFactory:CallAdapter.Factory(){
    override fun get(
        returnType: Type,
        annotations: Array<out Annotation>,
        retrofit: Retrofit
    ): CallAdapter<*, *>? {
        // suspend functions wrap the response type in `Call`
        if(Call::class.java != getRawType(returnType)){
            return null
        }
        check(returnType is ParameterizedType){
            "return type must be parameterized as Call<NetworkResponse<<Foo>> or Call<NetworkResponse<out Foo>>"
        }
        // get the response type inside the `Call` type
        val responseType = getParameterUpperBound(0,returnType)
        // if the response type is not ApiResponse then we can't handle this type, so we return null
        if(getRawType(responseType) != NetworkResponse::class.java){
            return null
        }


        // the response type is ApiResponse and should be parameterized
        check(responseType is ParameterizedType) { "Response must be parameterized as NetworkResponse<Foo> or NetworkResponse<out Foo>" }

        val successBodyType = getParameterUpperBound(0, responseType)
        val errorBodyType = getParameterUpperBound(1, responseType)

        val errorBodyConverter =
            retrofit.nextResponseBodyConverter<Any>(null, errorBodyType, annotations)

        return NetworkResponseAdapter<Any, Any>(successBodyType, errorBodyConverter)
    }
}
```

#### 构建 Retrofit 实例时添加该 Factory
``` Kotlin
val retrofit = Retrofit.Builder()
    .baseUrl(BASE_URL)
    .addCallAdapterFactory(NetworkResponseAdapterFactory())
    .addConverterFactory(GsonConverterFactory.create())
    .build()
```

#### 使用typealias简化返回类型(可选)

``` Kotlin
data class HttpError(val httpCode:Int,val errorMsg:String?,val exception: Throwable?)
// before
interface DemoApiService {
    suspend fun mainPageInfo(): NetworkResponse<MainPageInfo, HttpError>
}
// after
typealias GenericResponse<S> = NetworkResponse<S, HttpError>

interface ApiService {
    suspend fun mainPageInfo(): GenericResponse<MainPageInfo>
}
```

#### 使用
在 Activity 中直接使用lifecycleScope启动协程。
``` Kotlin
lifecycleScope.launch(Dispatchers.IO) {
    Log.e("KotlinActivity", "lifecycleScope.launch -->>" + Thread.currentThread().name);
    val mainPageInfo = mainPageApi.getMainPageInfo()

    withContext(Dispatchers.Main) {
        Log.e(
            "KotlinActivity",
            "withContext(Dispatchers.Main) -->>" + Thread.currentThread().name
        );
        when(mainPageInfo){

            is NetworkResponse.NetError -> Log.e("KotlinActivity",
                "NetError->$mainPageInfo"
            )
            is NetworkResponse.Success ->  refreshViewWithLaunch(mainPageInfo.body)
            is NetworkResponse.UnknownError -> Log.e("KotlinActivity","UnknownError->" + mainPageInfo.error)
        }
    }
}
```
或者在 ViewModel 中借助 LiveData 将返回值转化为可观察对象
``` Kotlin
class MainPageInfoViewModel:ViewModel() {
    private val _mainPageInfo  = MutableLiveData<MainPageInfo>()
    val mainPageInfo: LiveData<MainPageInfo> get() = _mainPageInfo
    fun getMainPageInfo(){
        viewModelScope.launch(Dispatchers.IO){
            val result = mainPageApi.getMainPageInfo()
            withContext(Dispatchers.Main){
                when(result){
                    is NetworkResponse.NetError -> Log.e("MainPageInfoViewModel",
                        "NetError->$result"
                    )
                    is NetworkResponse.Success ->  _mainPageInfo.value =  result.body
                    is NetworkResponse.UnknownError -> Log.e("MainPageInfoViewModel","UnknownError->" + result.error)
                }

            }
        }
    }

}
```
在 Activity 中使用
``` Kotlin
mainPageInfoModel = ViewModelProvider(this).get(MainPageInfoViewModel::class.java)
mainPageInfoModel.mainPageInfo.observe(this, Observer {
    if (it != null) {
        Log.e("KotlinActivity", "viewmodel获取结果成功")
        refreshViewWithViewModelResult(it);
    } else {
        Log.e("KotlinActivity", "viewmodel获取结果为空")
    }
})
mainPageInfoModel.getMainPageInfo()
```
暂时先这样吧，基本上够用了

----
以上