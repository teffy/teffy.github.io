---
title: read-fucking-source-code-retrofit
date: 2018-03-08 18:59:46
categories: 
- read_fucking_source_code
tags: 
- read_fucking_source_code
- Retrofit
---

本文主要分享Retrofit一些核心代码和一些流程分析。

<!-- more -->
Read fucking source code:Retrofit

简单使用
```
   Retrofit retrofit = new Retrofit.Builder()
        .baseUrl(API_URL)
        .addConverterFactory(GsonConverterFactory.create())
        .build();
    // Create an instance of our GitHub API interface.
    GitHub github = retrofit.create(GitHub.class);
    // Create a call instance for looking up Retrofit contributors.
    Call<List<Contributor>> call = github.contributors("square", "retrofit");
     List<Contributor> contributors = call.execute().body();
    for (Contributor contributor : contributors) {
      System.out.println(contributor.login + " (" + contributor.contributions + ")");
    }
```
```
new Retrofit.Builder()
        .baseUrl(API_URL)
        .addConverterFactory(GsonConverterFactory.create())
        .build();
```
组织参数构建出一个Retrofit，参数包括url，请求参数组装工程类，返回数据转换工程类。
```
retrofit2.Retrofit#create
public <T> T create(final Class<T> service) {
    Utils.validateServiceInterface(service);
    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, @Nullable Object[] args)
              throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            ServiceMethod<Object, Object> serviceMethod =
                (ServiceMethod<Object, Object>) loadServiceMethod(method);
            OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
            return serviceMethod.adapt(okHttpCall);
          }
        });
  }
```
方法会返回一个动态代理生成的 GitHub接口。
然后下面调用
```
    Call<List<Contributor>> call = github.contributors("square", "retrofit");
```
在动态代理中，执行method，先通过loadServiceMethod，在缓存中找出包装过Method的 ServiceMethod对象，ServiceMethod对象在初始化的时候，会把Retrofit中的所有参数和上面的数据转换的工程类对象都拿到，以及 接口中定义的方法上所携带的所有注解，包括方法注解，和参数注解，参数注解会转换为ParameterHandler的子类和注解的名字一样的子类名字，方便后面使用
```
result = new ServiceMethod.Builder<>(this, method).build();

 ServiceMethod(Builder<R, T> builder) {
    this.callFactory = builder.retrofit.callFactory();
    this.callAdapter = builder.callAdapter;
    this.baseUrl = builder.retrofit.baseUrl();
    this.responseConverter = builder.responseConverter;
    this.httpMethod = builder.httpMethod;
    this.relativeUrl = builder.relativeUrl;
    this.headers = builder.headers;
    this.contentType = builder.contentType;
    this.hasBody = builder.hasBody;
    this.isFormEncoded = builder.isFormEncoded;
    this.isMultipart = builder.isMultipart;
    this.parameterHandlers = builder.parameterHandlers; // 方法中定义的参数的注解
  }
  
   public ServiceMethod build() {
      callAdapter = createCallAdapter();
      
  private CallAdapter<T, R> createCallAdapter() {
      Type returnType = method.getGenericReturnType();
      if (Utils.hasUnresolvableType(returnType)) {
        throw methodError(
            "Method return type must not include a type variable or wildcard: %s", returnType);
      }
      if (returnType == void.class) {
        throw methodError("Service methods cannot return void.");
      }
      Annotation[] annotations = method.getAnnotations();
      try {
        //noinspection unchecked
        return (CallAdapter<T, R>) retrofit.callAdapter(returnType, annotations);
      } catch (RuntimeException e) { // Wide exception range because factories are user code.
        throw methodError(e, "Unable to create call adapter for %s", returnType);
      }
    }
```
返回一个 DefaultCallAdapterFactory，然后封装为OkHttpCall，OkHttpCall提供了同步执行的方法或者异步执行的方法，和创建 okhttp3.Call 的方法 okhttp3.Call toCall(@Nullable Object... args)，而 toCall 方法中会有 okhttp3.Call.Factory callFactory; 
最后
```
serviceMethod.adapt(okHttpCall);
  T adapt(Call<R> call) {
    return callAdapter.adapt(call);
  }
```
将请求转换为一个Call请求执行的代理对象，在这个例子中其实就是上面的OkHttpCall，还有其他的转换器，如RxJava2CallAdapter，会把请求转为一个Observable<Response<R>>，然后拿到这个observable，就可以按照Rxjava的使用方式来执行请求了 。
到这里动态代理的使命已经完成。
下面是真正的执行阶段
```
 List<Contributor> contributors = call.execute().body();
 
   @Override public Response<T> execute() throws IOException {
    okhttp3.Call call;

    synchronized (this) {
      if (executed) throw new IllegalStateException("Already executed.");
      executed = true;

      if (creationFailure != null) {
        if (creationFailure instanceof IOException) {
          throw (IOException) creationFailure;
        } else if (creationFailure instanceof RuntimeException) {
          throw (RuntimeException) creationFailure;
        } else {
          throw (Error) creationFailure;
        }
      }

      call = rawCall;
      if (call == null) {
        try {
          call = rawCall = createRawCall();
        } catch (IOException | RuntimeException | Error e) {
          throwIfFatal(e); //  Do not assign a fatal error to creationFailure.
          creationFailure = e;
          throw e;
        }
      }
    }

    if (canceled) {
      call.cancel();
    }

    return parseResponse(call.execute());
  }
```
createRawCall方法生成真正的 okhttp3.Call对象，其中又会调用到serviceMethod.toCall(args);
```
  private okhttp3.Call createRawCall() throws IOException {
    okhttp3.Call call = serviceMethod.toCall(args);
    if (call == null) {
      throw new NullPointerException("Call.Factory returned null.");
    }
    return call;
  }
  ```
  
  ```
    okhttp3.Call toCall(@Nullable Object... args) throws IOException {
    RequestBuilder requestBuilder = new RequestBuilder(httpMethod, baseUrl, relativeUrl, headers,
        contentType, hasBody, isFormEncoded, isMultipart);
    @SuppressWarnings("unchecked") // It is an error to invoke a method with the wrong arg types.
    ParameterHandler<Object>[] handlers = (ParameterHandler<Object>[]) parameterHandlers;
    int argumentCount = args != null ? args.length : 0;
    if (argumentCount != handlers.length) {
      throw new IllegalArgumentException("Argument count (" + argumentCount
          + ") doesn't match expected count (" + handlers.length + ")");
    }
    for (int p = 0; p < argumentCount; p++) {
      handlers[p].apply(requestBuilder, args[p]);
    }
    return callFactory.newCall(requestBuilder.build());
  }
```
RequestBuilder类，是具体请求参数的组装类，最后通过okhttp3.Call.Factory创建出okhttp3.Call
```
return parseResponse(call.execute());
```
解析okhttp请求返回的结果
```
T body = serviceMethod.toResponse(catchingBody);
```
通过serviceMethod.toResponse，内部通过调用一开始传进来的 Converter 工厂类将执行结果转为我们需要的结果实体类。