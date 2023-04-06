# Retrofit2源码分析

## 简单使用

一般的写法
```
//创建请求服务接口
interface AppApis {
    /**
     * 获取用户信息
     */
    @GET("user/getUserInfo")
    fun getUserInfo(@Query("userId") userId: Int): Call<UserInfo>
}

//创建Retrofit实例
val retrofit = Retrofit.Builder()
    .baseUrl("") //url
    .client(okHttpClient)
    .addConverterFactory(GsonConverterFactory.create())
    .build()
    
//获取服务
val appApis = retrofit.create(AppApis::class.java)

Call<UserInfo> call = appApis.getUserInfo(id)
//异步请求
call.enqueue(object : Callback<UserInfo> {
    override fun onResponse(call: Call<UserInfo>, response: Response<UserInfo>) {
        //请求成功响应
    }

    override fun onFailure(call: Call<UserInfo>, t: Throwable) {
        //请求失败
    }
})
```

> 使用Call的异步请求需要添加接口回调，而接口回调在子线程中，需要runOnUiThread，容易造成回调地狱。

使用`Retrofit + 协程`写法
```
//创建请求服务接口
//方法中使用suspend挂起函数修饰符
interface AppApis {
    /**
     * 获取用户信息
     */
    @GET("user/getUserInfo")
    suspend fun getUserInfo(@Query("userId") userId: Int): Response<UserInfo>
}

//创建Retrofit实例
val retrofit = Retrofit.Builder()
    .baseUrl("") //url
    .client(okHttpClient)
    .addConverterFactory(GsonConverterFactory.create())
    .build()

//获取服务
val appApis = retrofit.create(AppApis::class.java)

//在协程中调用服务请求数据
MainScope().launch(Dispatchers.IO) {
    UserInfo response = appApis.getUserInfo(id)
}

```

## 源码分析

Retrofit类的组成
```
public final class Retrofit {
    //网络请求配置对象
    private final Map<Method, ServiceMethod<?>> serviceMethodCache = new ConcurrentHashMap<>();
    
    //网络请求器工厂
    final okhttp3.Call.Factory callFactory;
    
    //请求地址
    final HttpUrl baseUrl;
    
    //数据转换器工厂集合
    final List<Converter.Factory> converterFactories;
    
    //网络请求适配器工厂
    final List<CallAdapter.Factory> callAdapterFactories;
    
    //回调执行器
    final @Nullable Executor callbackExecutor;
    
    //是否提前验证注解
    final boolean validateEagerly;
  
    Retrofit(
          okhttp3.Call.Factory callFactory,
          HttpUrl baseUrl,
          List<Converter.Factory> converterFactories,
          List<CallAdapter.Factory> callAdapterFactories,
          @Nullable Executor callbackExecutor,
          boolean validateEagerly) {
          
        this.callFactory = callFactory;
        this.baseUrl = baseUrl;
        this.converterFactories = converterFactories; // Copy+unmodifiable at call site.
        this.callAdapterFactories = callAdapterFactories; // Copy+unmodifiable at call site.
        this.callbackExecutor = callbackExecutor;
        this.validateEagerly = validateEagerly;
    }
  ...
}
```

Retrofit使用构建者模式，利用Builder对象简化构建过程，Retrofit#Builder()
```
public Builder() {
    //无参构造调用了有参构造
    this(Platform.get());
}

//有参构造
Builder(Platform platform) {
    this.platform = platform;
}

//平台
class Platform {
  //
  private static final Platform PLATFORM = findPlatform();

  static Platform get() {
    //静态成员变量，调用findPlatform返回Platform
    return PLATFORM;
  }
  
  //查询当前平台
  private static Platform findPlatform() {
    return "Dalvik".equals(System.getProperty("java.vm.name"))
        ? new Android() //Android平台
        : new Platform(true);
  }
  
  //Android
  static final class Android extends Platform {
    Android() {
      super(Build.VERSION.SDK_INT >= 24);
    }

    @Override
    public Executor defaultCallbackExecutor() {
      //返回一个回调执行器MainThreadExecutor
      //用于线程的切换
      return new MainThreadExecutor();
    }

    @Nullable
    @Override
    Object invokeDefaultMethod(
        Method method, Class<?> declaringClass, Object object, Object... args) throws Throwable {
      if (Build.VERSION.SDK_INT < 26) {
        throw new UnsupportedOperationException(
            "Calling default methods on API 24 and 25 is not supported");
      }
      return super.invokeDefaultMethod(method, declaringClass, object, args);
    }

    //MainThreadExecutor实现Executor接口
    static final class MainThreadExecutor implements Executor {
      //创建绑定主线Looper的Handler
      private final Handler handler = new Handler(Looper.getMainLooper());

      @Override
      public void execute(Runnable r) {
        //利用Handler的post回到主线程
        handler.post(r);
      }
    }
  }
  
  //默认网络请求适配器工厂
  List<? extends CallAdapter.Factory> defaultCallAdapterFactories(
      @Nullable Executor callbackExecutor) {
    DefaultCallAdapterFactory executorFactory = new DefaultCallAdapterFactory(callbackExecutor);
    
    //返回一个默认的网络请求适配器工厂列表
    return hasJava8Types
        ? asList(CompletableFutureCallAdapterFactory.INSTANCE, executorFactory)
        : singletonList(executorFactory);
  }
}

//回来再看
```

设置请求地址，Builder#baseUrl
```
public Builder baseUrl(String baseUrl) {
    Objects.requireNonNull(baseUrl, "baseUrl == null");
    return baseUrl(HttpUrl.get(baseUrl));
}

//设置请求地址，请求地址必须以'/'结尾
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

设置网路请求工厂，Builder#callFactory
```
public Builder client(OkHttpClient client) {
    //OkHttpClient实现了okhttp3.Call.Factory接口
    return callFactory(Objects.requireNonNull(client, "client == null"));
}

//设置网路请求工厂
public Builder callFactory(okhttp3.Call.Factory factory) {
    //callFactory == OkHttpClient
    this.callFactory = Objects.requireNonNull(factory, "factory == null");
    return this;
}
```

设置数据转换器，Builder#addConverterFactory
```
public Builder addConverterFactory(Converter.Factory factory) {
    //添加数据转换器
    converterFactories.add(Objects.requireNonNull(factory, "factory == null"));
    return this;
}
```

根据配置创建Retrofit对象，Builder#build
```
public Retrofit build() {
      if (baseUrl == null) {
        throw new IllegalStateException("Base URL required.");
      }

      okhttp3.Call.Factory callFactory = this.callFactory;
      //如果没有配置网路请求工厂，那么默认就使用OkHttpClient
      if (callFactory == null) {
        callFactory = new OkHttpClient();
      }

      Executor callbackExecutor = this.callbackExecutor;
      //如果没有设置回调执行器，那么使用platform默认的，Android默认是MainThreadExecutor
      if (callbackExecutor == null) {
        callbackExecutor = platform.defaultCallbackExecutor();
      }

      // Make a defensive copy of the adapters and add the default Call adapter.
      //网络请求适配器列表，可以在此前设置this.callAdapterFactories添加默认值
      List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>(this.callAdapterFactories);
      //设置平台的默认值，优先会使用我们设置的适配器
      callAdapterFactories.addAll(platform.defaultCallAdapterFactories(callbackExecutor));

      // Make a defensive copy of the converters.
      //数据转换器
      List<Converter.Factory> converterFactories =
          new ArrayList<>(
              1 + this.converterFactories.size() + platform.defaultConverterFactoriesSize());

      // Add the built-in converter factory first. This prevents overriding its behavior but also
      // ensures correct behavior when using converters that consume all types.
      //添加默认的转换器
      converterFactories.add(new BuiltInConverters());
      //添加自定义的转换器
      converterFactories.addAll(this.converterFactories);
      //添加平台的转换器
      converterFactories.addAll(platform.defaultConverterFactories());

      //创建Retrofit对象
      return new Retrofit(
          callFactory,
          baseUrl,
          unmodifiableList(converterFactories),
          unmodifiableList(callAdapterFactories),
          callbackExecutor,
          validateEagerly);
}
```

分析请求服务创建过程
```
val appApis = retrofit.create(AppApis::class.java)
```

```
public <T> T create(final Class<T> service) {
    validateServiceInterface(service);
    return (T)
        Proxy.newProxyInstance(
            service.getClassLoader(),
            new Class<?>[] {service},
            new InvocationHandler() {
              private final Platform platform = Platform.get();
              //先定一个空Object数组
              private final Object[] emptyArgs = new Object[0];

              @Override
              public @Nullable Object invoke(Object proxy, Method method, @Nullable Object[] args)
                  throws Throwable {
                // If the method is a method from Object then defer to normal invocation.
                //如果该方法是来自Object的方法，则正常调用
                if (method.getDeclaringClass() == Object.class) {
                  return method.invoke(this, args);
                }
                //参数
                args = args != null ? args : emptyArgs;
                return platform.isDefaultMethod(method)
                    //是Default方法，使用Lookup调用
                    ? platform.invokeDefaultMethod(method, service, proxy, args)
                    //加载ServiceMethod，再调用
                    : loadServiceMethod(method).invoke(args);
              }
            });
}
```

查看validateServiceInterface方法
```
private void validateServiceInterface(Class<?> service) {
    //判断服务Class是否是接口
    if (!service.isInterface()) {
      throw new IllegalArgumentException("API declarations must be interfaces.");
    }

    Deque<Class<?>> check = new ArrayDeque<>(1);
    check.add(service);
    while (!check.isEmpty()) {
      Class<?> candidate = check.removeFirst();
      //服务Class不能包含泛型参数
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

    //是否提前校验
    if (validateEagerly) {
      Platform platform = Platform.get();
      for (Method method : service.getDeclaredMethods()) {
        //method不是Default方法，不是静态方法，才加载ServiceMethod
        if (!platform.isDefaultMethod(method) && !Modifier.isStatic(method.getModifiers())) {
          loadServiceMethod(method);
        }
      }
    }
}
```

查看loadServiceMethod方法
```
ServiceMethod<?> loadServiceMethod(Method method) {
    //保证每个ServiceMethod为单例
    //先查找serviceMethodCache缓存中是否存在，有则返回
    ServiceMethod<?> result = serviceMethodCache.get(method);
    if (result != null) return result;

    //同步锁
    synchronized (serviceMethodCache) {
      result = serviceMethodCache.get(method);
      if (result == null) {
        //解析Method，创建ServiceMethod，并缓存到serviceMethodCache
        result = ServiceMethod.parseAnnotations(this, method);
        serviceMethodCache.put(method, result);
      }
    }
    return result;
}
```

分析ServiceMethod#parseAnnotations
```
static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Method method) {
    //创建RequestFactory，解析Method中的注解和参数
    RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);

    //校验返回类型
    Type returnType = method.getGenericReturnType();
    if (Utils.hasUnresolvableType(returnType)) {
      throw methodError(
          method,
          "Method return type must not include a type variable or wildcard: %s",
          returnType);
    }
    if (returnType == void.class) {
      throw methodError(method, "Service methods cannot return void.");
    }

    //根据请求工厂RequestFactory，创建ServiceMethod
    return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);
}
```

```
static RequestFactory parseAnnotations(Retrofit retrofit, Method method) {
    return new Builder(retrofit, method).build();
}

//构造方法
Builder(Retrofit retrofit, Method method) {
      this.retrofit = retrofit;
      this.method = method;
      this.methodAnnotations = method.getAnnotations();
      this.parameterTypes = method.getGenericParameterTypes();
      this.parameterAnnotationsArray = method.getParameterAnnotations();
}

//构建出RequestFactory对象
RequestFactory build() {
      //遍历方法的注解，转换
      for (Annotation annotation : methodAnnotations) {
        parseMethodAnnotation(annotation);
      }

      //请求方式不能为空
      if (httpMethod == null) {
        throw methodError(method, "HTTP method annotation is required (e.g., @GET, @POST, etc.).");
      }

      if (!hasBody) {
        if (isMultipart) {
          throw methodError(
              method,
              "Multipart can only be specified on HTTP methods with request body (e.g., @POST).");
        }
        if (isFormEncoded) {
          throw methodError(
              method,
              "FormUrlEncoded can only be specified on HTTP methods with "
                  + "request body (e.g., @POST).");
        }
      }

      //对每个参数进行解析，保存到parameterHandlers中
      int parameterCount = parameterAnnotationsArray.length;
      parameterHandlers = new ParameterHandler<?>[parameterCount];
      for (int p = 0, lastParameter = parameterCount - 1; p < parameterCount; p++) {
        parameterHandlers[p] =
            parseParameter(p, parameterTypes[p], parameterAnnotationsArray[p], p == lastParameter);
      }

      if (relativeUrl == null && !gotUrl) {
        throw methodError(method, "Missing either @%s URL or @Url parameter.", httpMethod);
      }
      if (!isFormEncoded && !isMultipart && !hasBody && gotBody) {
        throw methodError(method, "Non-body HTTP method cannot contain @Body.");
      }
      if (isFormEncoded && !gotField) {
        throw methodError(method, "Form-encoded method must contain at least one @Field.");
      }
      if (isMultipart && !gotPart) {
        throw methodError(method, "Multipart method must contain at least one @Part.");
      }

      return new RequestFactory(this);
}
```

重点看看HttpServiceMethod#parseAnnotations方法做了什么
```
static <ResponseT, ReturnT> HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(
      Retrofit retrofit, Method method, RequestFactory requestFactory) {
    //是否是kotlin的suspend方法
    boolean isKotlinSuspendFunction = requestFactory.isKotlinSuspendFunction;
    //是否想包含Response
    boolean continuationWantsResponse = false;
    boolean continuationBodyNullable = false;

    Annotation[] annotations = method.getAnnotations();
    Type adapterType;
    if (isKotlinSuspendFunction) {
      Type[] parameterTypes = method.getGenericParameterTypes();
      //参数最后一个是Continuation，用它来获取返回值类型
      Type responseType =
          Utils.getParameterLowerBound(
              0, (ParameterizedType) parameterTypes[parameterTypes.length - 1]);
      //判断类型的原始类是否是Response，类型是否是泛型类型
      if (getRawType(responseType) == Response.class && responseType instanceof ParameterizedType) {
        // Unwrap the actual body type from Response<T>.
        responseType = Utils.getParameterUpperBound(0, (ParameterizedType) responseType);
        //标志包含Response
        continuationWantsResponse = true;
      } else {
        // TODO figure out if type is nullable or not
        // Metadata metadata = method.getDeclaringClass().getAnnotation(Metadata.class)
        // Find the entry for method
        // Determine if return type is nullable or not
      }

      //包装返回值类型
      adapterType = new Utils.ParameterizedTypeImpl(null, Call.class, responseType);
      annotations = SkipCallbackExecutorImpl.ensurePresent(annotations);
    } else {
      //不是kotlin的suspend
      //使用方法的实际返回类型
      adapterType = method.getGenericReturnType();
    }

    //创建网络请求适配器CallAdapter
    CallAdapter<ResponseT, ReturnT> callAdapter =
        createCallAdapter(retrofit, method, adapterType, annotations);
    //获取返回值类型
    Type responseType = callAdapter.responseType();
    //类型不能是okhttp3.Response
    if (responseType == okhttp3.Response.class) {
      throw methodError(
          method,
          "'"
              + getRawType(responseType).getName()
              + "' is not a valid response body type. Did you mean ResponseBody?");
    }
    //返回值需要带泛型
    if (responseType == Response.class) {
      throw methodError(method, "Response must include generic type (e.g., Response<String>)");
    }
    // TODO support Unit for Kotlin?
    //请求方式是HEAD，则返回值类型需要Void
    if (requestFactory.httpMethod.equals("HEAD") && !Void.class.equals(responseType)) {
      throw methodError(method, "HEAD method must use Void as response type.");
    }

    //创建返回值数据转换器
    Converter<ResponseBody, ResponseT> responseConverter =
        createResponseConverter(retrofit, method, responseType);

    //网络请求的工厂，默认是OkhttpClient
    okhttp3.Call.Factory callFactory = retrofit.callFactory;
    if (!isKotlinSuspendFunction) {
      //非kotlin-Suspend
      //返回CallAdapted
      return new CallAdapted<>(requestFactory, callFactory, responseConverter, callAdapter);
    } else if (continuationWantsResponse) {
      //noinspection unchecked Kotlin compiler guarantees ReturnT to be Object.
      //包含Response泛型类型
      return (HttpServiceMethod<ResponseT, ReturnT>)
          new SuspendForResponse<>(
              requestFactory,
              callFactory,
              responseConverter,
              (CallAdapter<ResponseT, Call<ResponseT>>) callAdapter);
    } else {
      //noinspection unchecked Kotlin compiler guarantees ReturnT to be Object.
      //kotlin-Suspend
      return (HttpServiceMethod<ResponseT, ReturnT>)
          new SuspendForBody<>(
              requestFactory,
              callFactory,
              responseConverter,
              (CallAdapter<ResponseT, Call<ResponseT>>) callAdapter,
              continuationBodyNullable);
    }
}
```


