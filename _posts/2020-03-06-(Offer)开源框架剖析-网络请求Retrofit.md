---
layout:     post
title:      开源框架剖析-网络请求Retrofit
subtitle:   它以精妙的设计让我们能以轻松简洁的方式进行网络数据的请求
date:       2020-03-6
author:     ZYT
header-img: img/post-bg-offer.jpeg
catalog: true
tags:
    - Android
    - Offer
    - 网络请求
    - 开源框架剖析
    - retrofit
---

# 开源框架剖析-网络请求Retrofit

### 简单介绍下使用原理
首先，通过Builder创建Retrofit对象，在create方法中，通过JDK动态代理的方式，生成实现类，在调用接口方法时，会触发InvocationHandler的invoke方法，将接口的空方法转换成ServiceMethid, 然后生成okhttp请求，通过callAdapterFactory找到对应的执行器，比如RxJava2CallAdapterFactory，最后通过ConverterFactory将返回数据解析成JavaBena，使用者只需要关心请求参数，内部实现由retrofit封装完成，底层请求还是基于okhttp实现的。
### 使用流程
```
   public interface MyInterface {
        @GET("user/{user}/repos")
        Call<List<MyResponse>> getCall(@Path("user") String user);
   }
    
    
   Retrofit retrofit = new Retrofit.Builder()
                .baseUrl("http://xxx.xxx.com/")//请求的url地址
                .addConverterFactory(GsonConverterFactory.create())//数据解析
                .addCallAdapterFactory(RxJava2CallAdapterFactory.create())//网络请求适配器
                .build();
    MyInterface myInterface = retrofit.create(MyInterface.class);
    Call<List<MyResponse>> call = myInterface.getCall("name");
    call.enqueue(new Callback<List<MyResponse>>() {
        @Override
        public void onResponse(Call<List<MyResponse>> call, Response<List<MyResponse>> response) {
            
        }

        @Override
        public void onFailure(Call<List<MyResponse>> call, Throwable t) {

        }
    });
        
         
```
使用顺序
        
1，添加Retrofit库的依赖，添加网络权限

2，创建接收服务器返回数据的类

3，创建用于描述网络请求的接口

4，创建Retrofit实例

5，创建网络请求接口实例

6，发送网络请求（异步、同步）

7，处理服务器返回的数据


### Retrofit整体架构
![](https://tva1.sinaimg.cn/large/00831rSTly1gcx507wg33j30or0ne76p.jpg)

### Retrofit核心类/方法分析
##### Retrofit的属性和内部Build构造方法
```
public final class Retrofit {
  //用于记录请求RequestApi中某个请求方法对应的处理方式
  //ServiceMethod是核心处理类，解析接口中定义的请求方法参数和注解
  private final Map<Method, ServiceMethod> serviceMethodCache = new LinkedHashMap<>();
  
  //生产Call的工厂，这里的Call是okhttp包下面的Call，CallFactory默认就是OkHttpClien
  private final okhttp3.Call.Factory callFactory;
  
  //封装的基础请求url，具体的请求就是将参数附加在其上得到具体的api地址
  private final HttpUrl baseUrl;
  
  //响应数据解析器列表。数据解析器Converter，将response通过converterFactory转换成对应的JavaBean数据形式，常见解析器有，GsonConverterFactory，FastJsonConverterFactory，当然也有xml的。
  private final List<Converter.Factory> converterFactories;
  
  //请求调用转换器列表。通过calladapter将原始Call进行封装，找到对应的执行器。如rxjavaCallFactory对应的Observable，转换形式Call<T> --> Observable<T>
  private final List<CallAdapter.Factory> adapterFactories;
  
  //主线程执行器，返回结果在UI线程执行
  private final Executor callbackExecutor;
  
  //是否需要立即生成请求方法对应的处理对象，也就是将接口类中的方法全部转换成ServiceMethod，如果是，在create创建接口时就会生成一系列的ServiceMethod对象
  private final boolean validateEagerly;
  
  public static final class Builder {
    public Retrofit build() {
      if (baseUrl == null) {
        throw new IllegalStateException("Base URL required.");
      }

      //没有指定OkHttpClient，则使用默认创建的OkHttpClient对象，它是实际处理底层网络请求的
      okhttp3.Call.Factory callFactory = this.callFactory;
      if (callFactory == null) {
        callFactory = new OkHttpClient();
      }

      //android平台，默认callbackExecutor是ExecutorCallAdapterFactory，也就是将响应post到主线程处理
      Executor callbackExecutor = this.callbackExecutor;
      if (callbackExecutor == null) {
        callbackExecutor = platform.defaultCallbackExecutor();
      }

      //调用转换器列表
      List<CallAdapter.Factory> adapterFactories = new ArrayList<>(this.adapterFactories);
      adapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));

      //响应数据解析器列表
      List<Converter.Factory> converterFactories = new ArrayList<>(this.converterFactories);

      //生成Retrofit对象
      return new Retrofit(callFactory, baseUrl, converterFactories, adapterFactories,
          callbackExecutor, validateEagerly);
    }
  }
  
}
```
##### 通过Retrofit创建接口对象
```
public final class Retrofit {
  public <T> T create(final Class<T> service) {
    //验证是否接口，不是的话抛异常
    Utils.validateServiceInterface(service);
    //判断是否需要提前加载service中所有的method对应的ServiceMethod
    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }
    
    //使用动态代理的方式创建一个实现接口的对象
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          //获取平台，这里是android平台AndroidPlatform
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, Object... args)
              throws Throwable {
            //默认是接口，不执行
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            //默认AndroidPlatform返回false，不执行
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            
            //最终执行这里
            //加载调用方法对应的处理方式ServiceMethod
            ServiceMethod serviceMethod = loadServiceMethod(method);
            //创建一个原始的OkHttpCall，它内部直接使用Okhttp进行网络请求操作
            OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
            //这里将OkHttpCall封装转换成另一种类型的对象，比如说Observable
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
  }

  //提前加载service中所有的method对应的ServiceMethod
  private void eagerlyValidateMethods(Class<?> service) {
    Platform platform = Platform.get();
    for (Method method : service.getDeclaredMethods()) {
      if (!platform.isDefaultMethod(method)) {
        loadServiceMethod(method);
      }
    }
  }

  //加载service中的method对应的ServiceMethod
  ServiceMethod loadServiceMethod(Method method) {
    ServiceMethod result;
    synchronized (serviceMethodCache) {
      result = serviceMethodCache.get(method);
      if (result == null) {
        result = new ServiceMethod.Builder(this, method).build();
        serviceMethodCache.put(method, result);
      }
    }
    return result;
  }
}
```
##### ServiceMethod的创建
它主要用于解析对应的接口方法。
```
final class ServiceMethod<T> {
  static final class Builder<T> {
    final Retrofit retrofit;
    //对应方法
    final Method method;
    //方法上的注解
    final Annotation[] methodAnnotations;
    //方法参数上的注解
    final Annotation[][] parameterAnnotationsArray;
    //方法参数类型
    final Type[] parameterTypes;
    //返回的响应类型
    Type responseType;
    
    //判断是否找到了对应的注解
    boolean gotField;
    boolean gotPart;
    boolean gotBody;
    boolean gotPath;
    boolean gotQuery;
    boolean gotUrl;
    //http请求方法，Get,Post等
    String httpMethod;
    //是否有请求体
    boolean hasBody;
    boolean isFormEncoded;
    boolean isMultipart;
    //方法上注解的相对地址url
    String relativeUrl;
    Headers headers;
    MediaType contentType;
    Set<String> relativeUrlParamNames;
    //参数解析器，对诸如@GET("/api/4/news/{id}")进行解析，得到最终的url地址
    ParameterHandler<?>[] parameterHandlers;
    //响应内容转换器，将ResponseBody转换为其他类型
    Converter<ResponseBody, T> responseConverter;
    //请求调用转换器
    CallAdapter<?> callAdapter;
  }
  
  //构建ServiceMethod对象
  public ServiceMethod build() {
      //创建CallAdapter
      callAdapter = createCallAdapter();
      //callAdapter的泛型类型，如某个Bean
      responseType = callAdapter.responseType();
      if (responseType == Response.class || responseType == okhttp3.Response.class) {
        throw methodError("'"
            + Utils.getRawType(responseType).getName()
            + "' is not a valid response body type. Did you mean ResponseBody?");
      }
      //创建响应数据转换器
      responseConverter = createResponseConverter();
      //解析方法注解
      for (Annotation annotation : methodAnnotations) {
        parseMethodAnnotation(annotation);
      }

      if (httpMethod == null) {
        throw methodError("HTTP method annotation is required (e.g., @GET, @POST, etc.).");
      }

      if (!hasBody) {
        if (isMultipart) {
          throw methodError(
              "Multipart can only be specified on HTTP methods with request body (e.g., @POST).");
        }
        if (isFormEncoded) {
          throw methodError("FormUrlEncoded can only be specified on HTTP methods with "
              + "request body (e.g., @POST).");
        }
      }

      //解析参数注解
      int parameterCount = parameterAnnotationsArray.length;
      parameterHandlers = new ParameterHandler<?>[parameterCount];
      for (int p = 0; p < parameterCount; p++) {
        Type parameterType = parameterTypes[p];
        if (Utils.hasUnresolvableType(parameterType)) {
          throw parameterError(p, "Parameter type must not include a type variable or wildcard: %s",
              parameterType);
        }

        Annotation[] parameterAnnotations = parameterAnnotationsArray[p];
        if (parameterAnnotations == null) {
          throw parameterError(p, "No Retrofit annotation found.");
        }

        parameterHandlers[p] = parseParameter(p, parameterType, parameterAnnotations);
      }

      if (relativeUrl == null && !gotUrl) {
        throw methodError("Missing either @%s URL or @Url parameter.", httpMethod);
      }
      if (!isFormEncoded && !isMultipart && !hasBody && gotBody) {
        throw methodError("Non-body HTTP method cannot contain @Body.");
      }
      if (isFormEncoded && !gotField) {
        throw methodError("Form-encoded method must contain at least one @Field.");
      }
      if (isMultipart && !gotPart) {
        throw methodError("Multipart method must contain at least one @Part.");
      }

      //创建最终ServiceMethod对象
      return new ServiceMethod<>(this);
  }
  
}
```
```
final class ServiceMethod<T> {
  private CallAdapter<?> createCallAdapter() {
      //方法返回值类型
      Type returnType = method.getGenericReturnType();
      if (Utils.hasUnresolvableType(returnType)) {
        throw methodError(
            "Method return type must not include a type variable or wildcard: %s", returnType);
      }
      if (returnType == void.class) {
        throw methodError("Service methods cannot return void.");
      }
      
      //方法注解
      Annotation[] annotations = method.getAnnotations();
      try {
        return retrofit.callAdapter(returnType, annotations);
      } catch (RuntimeException e) { // Wide exception range because factories are user code.
        throw methodError(e, "Unable to create call adapter for %s", returnType);
      }
  }
}

public final class Retrofit {
  public CallAdapter<?> callAdapter(Type returnType, Annotation[] annotations) {
    return nextCallAdapter(null, returnType, annotations);
  }

  public CallAdapter<?> nextCallAdapter(CallAdapter.Factory skipPast, Type returnType,
      Annotation[] annotations) {
    checkNotNull(returnType, "returnType == null");
    checkNotNull(annotations, "annotations == null");

    int start = adapterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = adapterFactories.size(); i < count; i++) {
      CallAdapter<?> adapter = adapterFactories.get(i).get(returnType, annotations, this);
      //Factory获取的CallAdapter不为空，说明找到合适的CallAdapter了
      if (adapter != null) {
        return adapter;
      }
    }

    ...
  }
}

```
##### CallAdapter的转换过程
它是在OkhttpCall中进行响应数据解析转换的，不论是它的enqueue还是excute方法，响应结果都会交给parseResponse处理
```
final class OkHttpCall<T> implements Call<T> {
  Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
    //响应体
    ResponseBody rawBody = rawResponse.body();

    //用原始响应体封装一个新的响应体，是能够进行类型转换
    // Remove the body's source (the only stateful object) so we can pass the response along.
    rawResponse = rawResponse.newBuilder()
        .body(new NoContentResponseBody(rawBody.contentType(), rawBody.contentLength()))
        .build();

    //响应码
    int code = rawResponse.code();
    if (code < 200 || code >= 300) {
      //错误响应
      try {
        // Buffer the entire body to avoid future I/O.
        ResponseBody bufferedBody = Utils.buffer(rawBody);
        return Response.error(bufferedBody, rawResponse);
      } finally {
        rawBody.close();
      }
    }

    if (code == 204 || code == 205) {
      //响应码为无内容，没有数据时，成功，但返回空
      return Response.success(null, rawResponse);
    }

    ExceptionCatchingRequestBody catchingBody = new ExceptionCatchingRequestBody(rawBody);
    try {
      //这里调用serviceMethod去将响应体数据转换为指定类型
      T body = serviceMethod.toResponse(catchingBody);
      return Response.success(body, rawResponse);
    } catch (RuntimeException e) {
      // If the underlying source threw an exception, propagate that rather than indicating it was
      // a runtime exception.
      catchingBody.throwIfCaught();
      throw e;
    }
  }
}
```
enqueue或者excute方法执行得到响应数据后会到parseResponse将响应体数据转换为指定类型，是调用当前serviceMethod的toResponse方法进行转换，里面会调用responseConverter进行convert转换，如果我们在Retrofit配置时添加了GsonConverterFactory支持，那么就会使用GsonConverterFactory中的GsonResponseBodyConverter进行类型转换，而GsonResponseBodyConverter内部是使用gson进行类型转换。
```
final class GsonResponseBodyConverter<T> implements Converter<ResponseBody, T> {
  private final Gson gson;
  private final TypeAdapter<T> adapter;

  GsonResponseBodyConverter(Gson gson, TypeAdapter<T> adapter) {
    this.gson = gson;
    this.adapter = adapter;
  }

  @Override public T convert(ResponseBody value) throws IOException {
    //使用gson对ResponseBody转换为指定类型
    JsonReader jsonReader = gson.newJsonReader(value.charStream());
    try {
      return adapter.read(jsonReader);
    } finally {
      value.close();
    }
  }
}
```

### 使用到的设计模式

构建者模式

工厂方法

静态工厂

外观模式

策略模式

适配器模式

动态代理

观察者模式

### 总结

1，Retrofit可扩展性强，底层网络请求集成了Okhttp，异步处理可集成RxJava，内容解析可集成Gson，Jackson等。

2，全面支持Restful请求，并且通过注解的方式，支持链式调用，使用简洁方便。

3，简洁精妙的设计模式，各功能用途层次分工明确，解耦性极强。

