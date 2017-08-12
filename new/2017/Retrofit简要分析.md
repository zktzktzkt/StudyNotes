# Retrofit简要分析
## Retrofit网络请求过程梳理

首先先看一下基本用法下默认构造的Retrofit实例包含的成员变量的值,即如下代码构造Retrofit实例时Retrofit对象成员变量的值

```
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://api.github.com/")
    .build();
```

对应build方法创建的Retrofit实例的成员变量的值如下

+ callFactory:OkHttpClient implements Cloneable, Call.Factory, WebSocket.Factory
+ baseUrl:HttpUrl
+ callbackExecutor:MainThreadExecutor
+ adapterFactories:UnmodifiableRandomAccessList,默认包含一个item-ExecutorCallAdapterFactory
+ converterFactories:UnmodifiableRandomAccessList默认包含一个item-BuiltInConverters
+ validateEagerly:false
```

接着从创建api接口的实现类开始,比如官方教程中的创建一个GitHub Api

```
GitHubService service = retrofit.create(GitHubService.class);
```

在retrofit中是这样的创建的

```
public <T> T create(final Class<T> service) {
    //略...
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();
          @Override public Object invoke(Object proxy, Method method, @Nullable Object[] args)
              throws Throwable {
            //默认方法的调用,略...
            ServiceMethod<Object, Object> serviceMethod =
                (ServiceMethod<Object, Object>) loadServiceMethod(method);
            OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
  }
```

所以当我们调用**Call<List<Repo>> repos = service.listRepos("octocat")**时实际上是执行的以下过程

```
ServiceMethod<Object, Object> serviceMethod = (ServiceMethod<Object, Object>) loadServiceMethod(method);
OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);//创建一个OKHttpCall对象
return serviceMethod.callAdapter.adapt(okHttpCall);//返回ExecutorCallbackCall<>(callbackExecutor, call);
```

整个过程的逻辑主要集中在第一步loadServiceMethod,先看返回值是一个ServiceMethod类型,源码的注释如下
>Adapts an invocation of an interface method into an HTTP call

所以说ServiceMethod是一个封装了关于一个HTTP请求的参数信息的类(毕竟Retrofit关于HTTP请求的大部分信息都是来源于对应api接口中的方法注解),loadServiceMethod的**核心逻辑**是这样的

```
ServiceMethod<?, ?> loadServiceMethod(Method method) {
  ServiceMethod<?, ?> result = serviceMethodCache.get(method);
  //缓存和同步逻辑略...
  if (result == null) {
    result = new ServiceMethod.Builder<>(this, method).build();
    serviceMethodCache.put(method, result);//缓存
  }
  return result;
}

Builder(Retrofit retrofit, Method method) {
  this.retrofit = retrofit;
  this.method = method;
  this.methodAnnotations = method.getAnnotations();//所有方法注解
  this.parameterTypes = method.getGenericParameterTypes();//返回类型
  this.parameterAnnotationsArray = method.getParameterAnnotations();//方法参数注解
}

public ServiceMethod build() {
  callAdapter = createCallAdapter();
  responseType = callAdapter.responseType();//api方法的返回类型,例如这里是List<Repo>
  //略...
  //遍历Retrofit实例的converterFactories,调用相应的fatory的responseBodyConverter方法获取response Converter
  responseConverter = createResponseConverter();
  for (Annotation annotation : methodAnnotations) {
    parseMethodAnnotation(annotation);//处理Retrofit中定义的方法注解,比如获取HTTP的请求方法、数据类型等
  }
  //略...
  int parameterCount = parameterAnnotationsArray.length;
  parameterHandlers = new ParameterHandler<?>[parameterCount];
  for (int p = 0; p < parameterCount; p++) {
    Type parameterType = parameterTypes[p];
    //略...
    Annotation[] parameterAnnotations = parameterAnnotationsArray[p];
    //略...  获取对应参数注解处理器
    parameterHandlers[p] = parseParameter(p, parameterType, parameterAnnotations);
  }
  //略..
  return new ServiceMethod<>(this);
}
```

这里简单的过了一下流程,具体的代码就不细看,如注释中所说这个方法主要将注解的信息以及方法参数的信息整合进一个ServiceMethod类以便用于HTTP请求的构造

### HTTP请求发送

有上面的分析,api调用返回的Call实际类型为返回ExecutorCallbackCall,其enqueue方法如下

```
@Override 
public void enqueue(final Callback<T> callback) {
  //delegate为上面创建OkHttpCall实例
  delegate.enqueue(new Callback<T>() {
    @Override public void onResponse(Call<T> call, final Response<T> response) {
      //callbackExecutor为MainThreadExecutor,用户将回调发送到主线程
      callbackExecutor.execute(//回调方法调用略...);
    }
    @Override public void onFailure(Call<T> call, final Throwable t) {
      callbackExecutor.execute(//回调方法调用略...);
    }
  });
}
```

上面可以看到实际的网络请求又delegate对象负责,delegate是一个OkHttpCall类型,它的enqueue方法如下

```
@Override public void enqueue(final Callback<T> callback) {
  //略...  
  if (call == null && failure == null) {
    try {
      call = rawCall = createRawCall();//创建一个okHTTP请求,在该方法中存在对于请求参数的转换
    } catch (Throwable t) {
      failure = creationFailure = t;
    }
  }
  //略...
  //okhttp发送网络请求并设置回调
  call.enqueue(new okhttp3.Callback() {
    @Override public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse) throws IOException {
      Response<T> response;
      try {
        response = parseResponse(rawResponse);//解析response,利用converter装换响应体
      }//略...
      callSuccess(response);//回调调用
    }
    //略
  });
}
```

至此Retrofit的网络请求过程就已经结束了,上面的代码过程说的都比较粗糙,下面主要针对Retrofit对于请求数据、响应数据的转换的细节来看.

## Retrofit数据转换
### 方法参数注解处理器

方法参数注解处理器的使用主要在两个地方

+ 一是创建ServiceMethod时解析参数注解时的parseParameter为每个参数注解创建注解参数处理器的时候,它将每个参数和对应的注解处理**建立映射关系(根据下标)**,以便后续参数转化时使用,主要实现逻辑在ServiceMethod类中的parseParameterAnnotation方法中,一共有300多行.主要就是根据对应的参数注解来返回不同的注解参数器,对应关系如下

* Url注解-new ParameterHandler.RelativeUrl();
* Path注解-

```
//返回值为BuiltInConverters.ToStringConverter
Converter<?, String> converter = retrofit.stringConverter(type, annotations);
return new ParameterHandler.Path<>(name, converter, path.encoded());
```

* Query注解

```
根据参数的数据类型共有三种类型,其中converter为BuiltInConverters.ToStringConverter
return new ParameterHandler.Query<>(name, converter, encoded).iterable();
return new ParameterHandler.Query<>(name, converter, encoded).array();
return new ParameterHandler.Query<>(name, converter, encoded);
```
* QueryName注解

```
根据参数的数据类型共有三种类型,其中converter为BuiltInConverters.ToStringConverter
return new ParameterHandler.QueryName<>(converter, encoded).iterable();
return new ParameterHandler.QueryName<>(converter, encoded).array();
return new ParameterHandler.QueryName<>(converter, encoded);
```

* QueryMap注解

```
converter为BuiltInConverters.ToStringConverter
return new ParameterHandler.QueryMap<>(valueConverter, ((QueryMap) annotation).encoded());
```

* Header注解

```
根据参数的数据类型共有三种类型,其中converter为BuiltInConverters.ToStringConverter
return new ParameterHandler.Header<>(name, converter).iterable();
return new ParameterHandler.Header<>(name, converter).array();
return new ParameterHandler.Header<>(name, converter);
```

* HeaderMap注解

```
converter为BuiltInConverters.ToStringConverter
return new ParameterHandler.HeaderMap<>(valueConverter);
```

* Field注解

```
根据参数的数据类型共有三种类型,其中converter为BuiltInConverters.ToStringConverter
return new ParameterHandler.Field<>(name, converter, encoded).iterable();
return new ParameterHandler.Field<>(name, converter, encoded).array();
return new ParameterHandler.Field<>(name, converter, encoded);
```

* FieldMap注解

```
converter为BuiltInConverters.ToStringConverter
return new ParameterHandler.FieldMap<>(valueConverter, ((FieldMap) annotation).encoded());
```

* Part注解

```
对应两种情况,Part注解value没有值
return ParameterHandler.RawPart.INSTANCE.iterable();
return ParameterHandler.RawPart.INSTANCE.array();
return ParameterHandler.RawPart.INSTANCE;

Part注解value有值其中Converter为Retrofit实例的converter factory所返回的requestBodyConverter

return new ParameterHandler.Part<>(headers, converter).iterable();
return new ParameterHandler.Part<>(headers, converter).array();
return new ParameterHandler.Part<>(headers, converter);
```

* PartMap注解

```
Converter<?, RequestBody> valueConverter = retrofit.requestBodyConverter(valueType, annotations, methodAnnotations);//requestBody converter
PartMap partMap = (PartMap) annotation;
return new ParameterHandler.PartMap<>(valueConverter, partMap.encoding());
```

* Body注解

```
converter = retrofit.requestBodyConverter(type, annotations, methodAnnotations);
return new ParameterHandler.Body<>(converter);
```

+ 二是createRawCall方法创建okhttp call时转换请求参数的时候如下

```
private okhttp3.Call createRawCall() throws IOException {
    Request request = serviceMethod.toRequest(args);//args是调用api方法传入的参数
    okhttp3.Call call = serviceMethod.callFactory.newCall(request);//callFactory值为OkHttpClient
    //略...
    return call;
  }
```

参数转换发生在toRequest方法中

```
Request toRequest(@Nullable Object... args) throws IOException {
    RequestBuilder requestBuilder = new RequestBuilder(httpMethod, baseUrl, relativeUrl, headers,
        contentType, hasBody, isFormEncoded, isMultipart);
    @SuppressWarnings("unchecked") // It is an error to invoke a method with the wrong arg types.
    ParameterHandler<Object>[] handlers = (ParameterHandler<Object>[]) parameterHandlers;
    //略
    for (int p = 0; p < argumentCount; p++) {
      handlers[p].apply(requestBuilder, args[p]);
    }
    return requestBuilder.build();
  }
```

对于每个参数调用对应处理器的apply方法进行数据转换,数据转换的逻辑可分为三种

+ 不转化:直接使用参数的值来组装请求-ParameterHandler.RawPart

+ 使用ToStringConverter或者直接调用参数的toString方法-ParameterHandler.RelativeUrl、ParameterHandler.Path(并填充{}占位)、ParameterHandler.Query、ParameterHandler.QueryName、ParameterHandler.QueryMap、
ParameterHandler.Header、ParameterHandler.HeaderMap、ParameterHandler.Field、ParameterHandler.FieldMap

+ 使用requestBodyConverter做数据转化,转化目标类型为RequestBody-ParameterHandler.Part、ParameterHandler.PartMap、ParameterHandler.Body

到这里说的是网络请求发送之前为了构造适应于OkHttp请求Retrofit对参数进行转化的一些操作,主要目的是弄清楚requestBodyConverter是用在那个地方,这样才能够正确的写出requestBodyConverter,看完网络请求前的数据转化再看网络请求结束后的数据转化,也就是Retrofit如何使用reponseBodyConverter

### Retrofit如何使用responseBodyConverter

在上面网络请求的流程中其实已经提到了responseBodyConverter被调用的地方,在网络请求的最后一步OkHttpCall的enqueue方法里面,部分代码如下

```

//okhttp发送网络请求并设置回调
call.enqueue(new okhttp3.Callback() {
  @Override public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse) throws IOException {
    Response<T> response;
    try {
      response = parseResponse(rawResponse);//解析response,利用converter装换响应体
    }//略...
    callSuccess(response);//回调调用
  }
});
```

核心方法parseResponse方法的代码如下

```
Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
  ResponseBody rawBody = rawResponse.body();
  // Remove the body's source (the only stateful object) so we can pass the response along.
  rawResponse = rawResponse.newBuilder()
      .body(new NoContentResponseBody(rawBody.contentType(), rawBody.contentLength()))
      .build();
  int code = rawResponse.code();
  //错误处理,略...
  ExceptionCatchingRequestBody catchingBody = new ExceptionCatchingRequestBody(rawBody);
  try {
    T body = serviceMethod.toResponse(catchingBody);//实际调用方法responseConverter.convert(body);
    return Response.success(body, rawResponse);
  } catch (RuntimeException e) {
    //...略
  }
}
```

ok,到这里就结束了,因为Retrofit是基于okhttp的网络请求的框架,所以我觉得对于Retrofit的理解应该侧重与它如何在自己的基础上将参数转化为OkHttp发送网络请求所需要的参数(requestBodyConverter)以及如何将OkHttp返回的reponse转化为用户所需要的数据结构(responseBodyConverter).最后对应的分析的源码版本为Retrofit 2.3(逃.