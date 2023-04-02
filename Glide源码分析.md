# Glide源码分析

## Glide一般使用例子
```
Glide.with(context).load(url).into(view)
```

## 源码分析

### Glide#with

with方法执行流程
```
@NonNull
public static RequestManager with(@NonNull Context context) {
    //分两步
    //getRetriever(context)
    //RequestManagerRetriever.get(context)
    return getRetriever(context).get(context);
}

@NonNull
  private static RequestManagerRetriever getRetriever(@Nullable Context context) {
    ···
    //分析with-1
    return Glide.get(context).getRequestManagerRetriever();
}

//RequestManagerRetriever.get(context)
@NonNull
public RequestManager get(@NonNull Context context) {
    //分析with-2
    if (context == null) {
      throw new IllegalArgumentException("You cannot start a load on a null Context");
    } else if (Util.isOnMainThread() && !(context instanceof Application)) {
      if (context instanceof FragmentActivity) {
        return get((FragmentActivity) context);
      } else if (context instanceof Activity) {
        return get((Activity) context);
      } else if (context instanceof ContextWrapper
          && ((ContextWrapper) context).getBaseContext().getApplicationContext() != null) {
        return get(((ContextWrapper) context).getBaseContext());
      }
    }
    return getApplicationManager(context);
}
```
* 初始化和创建Glide实例
* 使用RequestManagerRetriever返回RequestManager

分析with-1，Glide#get(context)，使用单例模式创建Glide实例
```
@NonNull
  public static Glide get(@NonNull Context context) {
    //创建Glide单例，使用双重检查锁
    if (glide == null) {
      //分析Glide-get-1
      GeneratedAppGlideModule annotationGeneratedModule =
          getAnnotationGeneratedGlideModules(context.getApplicationContext());
      synchronized (Glide.class) {
        if (glide == null) {
          //分析Glide-get-2
          checkAndInitializeGlide(context, annotationGeneratedModule);
        }
      }
    }

    return glide;
}
```

分析Glide-get-1，这里涉及自定义AppGlideModule的辅助生成类的创建
```
@Nullable
  private static GeneratedAppGlideModule getAnnotationGeneratedGlideModules(Context context) {
    GeneratedAppGlideModule result = null;
    try {
      //创建GeneratedAppGlideModule，GeneratedAppGlideModule是抽象类
      //GeneratedAppGlideModuleImpl实现类
      Class<GeneratedAppGlideModule> clazz =
          (Class<GeneratedAppGlideModule>)
              Class.forName("com.bumptech.glide.GeneratedAppGlideModuleImpl");
      result =
          clazz.getDeclaredConstructor(Context.class).newInstance(context.getApplicationContext());
    } catch (ClassNotFoundException e) {
      ···
    } catch (InstantiationException e) {
      throwIncorrectGlideModule(e);
    } catch (IllegalAccessException e) {
      throwIncorrectGlideModule(e);
    } catch (NoSuchMethodException e) {
      throwIncorrectGlideModule(e);
    } catch (InvocationTargetException e) {
      throwIncorrectGlideModule(e);
    }
    return result;
}
```

分析Glide-get-2，检查和初始化Glide
```
@GuardedBy("Glide.class")
  private static void checkAndInitializeGlide(
      @NonNull Context context, @Nullable GeneratedAppGlideModule generatedAppGlideModule) {
    if (isInitializing) {
      throw new IllegalStateException(
          "You cannot call Glide.get() in registerComponents(),"
              + " use the provided Glide instance instead");
    }
    isInitializing = true;
    //跟踪初始化代码
    initializeGlide(context, generatedAppGlideModule);
    isInitializing = false;
}
```

initializeGlide
```
private static void initializeGlide(
      @NonNull Context context, @Nullable GeneratedAppGlideModule generatedAppGlideModule) {
    //GlideBuilder：Glide默认的配置构建类
    initializeGlide(context, new GlideBuilder(), generatedAppGlideModule);
}

private static void initializeGlide(
      @NonNull Context context,
      @NonNull GlideBuilder builder,
      @Nullable GeneratedAppGlideModule annotationGeneratedModule) {
    //上下文
    Context applicationContext = context.getApplicationContext();
    //清单文件中注册的GlideModule列表
    List<com.bumptech.glide.module.GlideModule> manifestModules = Collections.emptyList();
    if (annotationGeneratedModule == null || annotationGeneratedModule.isManifestParsingEnabled()) {
      //清单文件解析类
      manifestModules = new ManifestParser(applicationContext).parse();
    }

    if (annotationGeneratedModule != null
        && !annotationGeneratedModule.getExcludedModuleClasses().isEmpty()) {
      //获取GlideModule类集合
      Set<Class<?>> excludedModuleClasses = annotationGeneratedModule.getExcludedModuleClasses();
      //检查excludedModuleClasses
      Iterator<com.bumptech.glide.module.GlideModule> iterator = manifestModules.iterator();
      while (iterator.hasNext()) {
        com.bumptech.glide.module.GlideModule current = iterator.next();
        if (!excludedModuleClasses.contains(current.getClass())) {
          continue;
        }
        if (Log.isLoggable(TAG, Log.DEBUG)) {
          Log.d(TAG, "AppGlideModule excludes manifest GlideModule: " + current);
        }
        iterator.remove();
      }
    }

    ···

    RequestManagerRetriever.RequestManagerFactory factory =
        annotationGeneratedModule != null
            ? annotationGeneratedModule.getRequestManagerFactory()
            : null;
            
    //GlideBuilder设置请求管理工厂
    builder.setRequestManagerFactory(factory);
    for (com.bumptech.glide.module.GlideModule module : manifestModules) {
      //回调GlideModule的applyOptions方法
      //可以在applyOptions回调中对GlideBuilder进行自定义设置
      module.applyOptions(applicationContext, builder);
    }
    if (annotationGeneratedModule != null) {
      annotationGeneratedModule.applyOptions(applicationContext, builder);
    }
    //通过GlideBuilder#build创建Glide
    //分析builder.build
    Glide glide = builder.build(applicationContext);
    for (com.bumptech.glide.module.GlideModule module : manifestModules) {
      try {
        //回调GlideModule的registerComponents
        module.registerComponents(applicationContext, glide, glide.registry);
      } catch (AbstractMethodError e) {
        ···
      }
    }
    if (annotationGeneratedModule != null) {
      annotationGeneratedModule.registerComponents(applicationContext, glide, glide.registry);
    }
    //Glide实现了ComponentCallbacks2，ComponentCallbacks2继承ComponentCallbacks
    applicationContext.registerComponentCallbacks(glide);
    //Glide， private static volatile Glide glide;
    Glide.glide = glide;
}
```

分析builder.build，GlideBuilder#build，使用建造者模式创建Glide
```
@NonNull
Glide build(@NonNull Context context) {
    //请求图片线程池
    if (sourceExecutor == null) {
      sourceExecutor = GlideExecutor.newSourceExecutor();
    }

    //硬盘缓存线程池
    if (diskCacheExecutor == null) {
      diskCacheExecutor = GlideExecutor.newDiskCacheExecutor();
    }

    //动画线程池
    if (animationExecutor == null) {
      animationExecutor = GlideExecutor.newAnimationExecutor();
    }

    //设备缓存计算器
    if (memorySizeCalculator == null) {
      memorySizeCalculator = new MemorySizeCalculator.Builder(context).build();
    }

    //连接器工厂
    if (connectivityMonitorFactory == null) {
      connectivityMonitorFactory = new DefaultConnectivityMonitorFactory();
    }

    if (bitmapPool == null) {
      //根据设备的屏幕密度和宽高尺寸获取pool的size
      int size = memorySizeCalculator.getBitmapPoolSize();
      if (size > 0) {
        bitmapPool = new LruBitmapPool(size);
      } else {
        bitmapPool = new BitmapPoolAdapter();
      }
    }

    if (arrayPool == null) {
      arrayPool = new LruArrayPool(memorySizeCalculator.getArrayPoolSizeInBytes());
    }

    //内存缓存
    if (memoryCache == null) {
      memoryCache = new LruResourceCache(memorySizeCalculator.getMemoryCacheSize());
    }

    //硬盘缓存工厂
    if (diskCacheFactory == null) {
      diskCacheFactory = new InternalCacheDiskCacheFactory(context);
    }

    //构建引擎，负责资源管理和执行任务
    if (engine == null) {
      engine =
          new Engine(
              memoryCache,
              diskCacheFactory,
              diskCacheExecutor,
              sourceExecutor,
              GlideExecutor.newUnlimitedSourceExecutor(),
              animationExecutor,
              isActiveResourceRetentionAllowed);
    }

    if (defaultRequestListeners == null) {
      defaultRequestListeners = Collections.emptyList();
    } else {
      defaultRequestListeners = Collections.unmodifiableList(defaultRequestListeners);
    }

    GlideExperiments experiments = glideExperimentsBuilder.build();
    //创建RequestManagerRetriever
    //requestManagerFactory是在initializeGlide方法中设置的RequestManagerFactory
    RequestManagerRetriever requestManagerRetriever =
        new RequestManagerRetriever(requestManagerFactory, experiments);

    return new Glide(
        context,
        engine,
        memoryCache,
        bitmapPool,
        arrayPool,
        requestManagerRetriever,
        connectivityMonitorFactory,
        logLevel,
        defaultRequestOptionsFactory,
        defaultTransitionOptions,
        defaultRequestListeners,
        experiments);
}
```

回到分析with-2，getRetriever(context).get(context)
```
@NonNull
  public RequestManager get(@NonNull Context context) {
    //根据不同上下文宿主，获取RequestManager
  
    if (context == null) {
      throw new IllegalArgumentException("You cannot start a load on a null Context");
    } else if (Util.isOnMainThread() && !(context instanceof Application)) {
      //在主线程
      //FragmentActivity分支
      if (context instanceof FragmentActivity) {
        return get((FragmentActivity) context);
      } else if (context instanceof Activity) {
      //Activity分支
        return get((Activity) context);
      } else if (context instanceof ContextWrapper
          && ((ContextWrapper) context).getBaseContext().getApplicationContext() != null) {
        return get(((ContextWrapper) context).getBaseContext());
      }
    }
    //Application分支
    return getApplicationManager(context);
}
```

FragmentActivity分支
```
@NonNull
  public RequestManager get(@NonNull FragmentActivity activity) {
    if (Util.isOnBackgroundThread()) {
      //如果不是主线程，调用的get的ApplicationContext重载的方法
      return get(activity.getApplicationContext());
    } else {
      assertNotDestroyed(activity);
      frameWaiter.registerSelf(activity);
      //获取activity的FragmentManager
      FragmentManager fm = activity.getSupportFragmentManager();
      //创建SupportRequestManagerFragment
      //分析supportFragmentGet
      return supportFragmentGet(activity, fm, /*parentHint=*/ null, isActivityVisible(activity));
    }
}
```

Activity分支
> 跟FragmentActivity分支差不多

Application分支
```
@NonNull
  private RequestManager getApplicationManager(@NonNull Context context) {
    // Either an application context or we're on a background thread.
    if (applicationManager == null) {
      synchronized (this) {
        if (applicationManager == null) {

          //关联到Application的RequestManager，将不接收到生命周期的回调
          //需要我们自己手动管理生命周期的回调
          Glide glide = Glide.get(context.getApplicationContext());
          applicationManager =
              factory.build(
                  glide,
                  new ApplicationLifecycle(),
                  new EmptyRequestManagerTreeNode(),
                  context.getApplicationContext());
        }
      }
    }

    return applicationManager;
}
```

分析supportFragmentGet
```
@NonNull
  private RequestManager supportFragmentGet(
      @NonNull Context context,
      @NonNull FragmentManager fm,
      @Nullable Fragment parentHint,
      boolean isParentVisible) {
      
    //SupportRequestManagerFragment继承Fragment
    //内部使用Map<FragmentManager, SupportRequestManagerFragment>存储SupportRequestManagerFragment
    //根据FragmentManager获取一个SupportRequestManagerFragment
    //分析getSupportRequestManagerFragment
    SupportRequestManagerFragment current = getSupportRequestManagerFragment(fm, parentHint);
    
    //获取RequestManager
    RequestManager requestManager = current.getRequestManager();
    if (requestManager == null) {
      Glide glide = Glide.get(context);
      
      //创建RequestManager
      //factory在RequestManagerRetriever的构造函数中初始化
      //分析factory.build
      requestManager =
          factory.build(
              glide, current.getGlideLifecycle(), current.getRequestManagerTreeNode(), context);
      
      //如果是可见的话，调用RequestManager的onStart
      if (isParentVisible) {
        requestManager.onStart();
      }
      //SupportRequestManagerFragment存储当前的RequestManager
      current.setRequestManager(requestManager);
    }
    return requestManager;
}
```

分析getSupportRequestManagerFragment
```
private SupportRequestManagerFragment getSupportRequestManagerFragment(
      @NonNull final FragmentManager fm, @Nullable Fragment parentHint) {

    SupportRequestManagerFragment current = pendingSupportRequestManagerFragments.get(fm);
    if (current == null) {
    
      //先通过FragmentManager查找tag为FRAGMENT_TAG的Fragment
      current = (SupportRequestManagerFragment) fm.findFragmentByTag(FRAGMENT_TAG);
      //没有的话，创建一个SupportRequestManagerFragment
      if (current == null) {
        //创建一个无界面的Fragment
        current = new SupportRequestManagerFragment();
        current.setParentFragmentHint(parentHint);
        //缓存SupportRequestManagerFragment
        pendingSupportRequestManagerFragments.put(fm, current);
        //开启事务，添加一个Fragment
        fm.beginTransaction().add(current, FRAGMENT_TAG).commitAllowingStateLoss();
        //发送一个消息，从pendingSupportRequestManagerFragments中移除SupportRequestManagerFragment
        handler.obtainMessage(ID_REMOVE_SUPPORT_FRAGMENT_MANAGER, fm).sendToTarget();
      }
    }
    return current;
}
```

看看SupportRequestManagerFragment里实现了什么
```
//默认构造中传入一个ActivityFragmentLifecycle实例
public SupportRequestManagerFragment() {
    //ActivityFragmentLifecycle实现Lifecycle接口
    //内部维护一个lifecycleListeners：Set<LifecycleListener>变量存储生命周期监听器
    this(new ActivityFragmentLifecycle());
}

//在Fragment的各个生命周期回调中回调lifecycle的回调
public SupportRequestManagerFragment(@NonNull ActivityFragmentLifecycle lifecycle) {
    this.lifecycle = lifecycle;
}

//无界面的Fragment
public class SupportRequestManagerFragment extends Fragment {
  ···

  @Override
  public void onAttach(Context context) {
    super.onAttach(context);

    //获取FragmentManager
    FragmentManager rootFragmentManager = getRootFragmentManager(this);
    if (rootFragmentManager == null) {
      if (Log.isLoggable(TAG, Log.WARN)) {
        // Not expected to occur; ancestor fragments should be attached before descendants.
        Log.w(TAG, "Unable to register fragment with root, ancestor detached");
      }
      return;
    }

    try {
      //注册Fragment到宿主Fragment上
      registerFragmentWithRoot(getContext(), rootFragmentManager);
    } catch (IllegalStateException e) {
      // OnAttach can be called after the activity is destroyed, see #497.
      if (Log.isLoggable(TAG, Log.WARN)) {
        Log.w(TAG, "Unable to register fragment with root", e);
      }
    }
  }

  @Override
  public void onDetach() {
    super.onDetach();
    parentFragmentHint = null;
    unregisterFragmentWithRoot();
  }

  @Override
  public void onStart() {
    super.onStart();
    lifecycle.onStart();
  }

  @Override
  public void onStop() {
    super.onStop();
    lifecycle.onStop();
  }

  @Override
  public void onDestroy() {
    super.onDestroy();
    lifecycle.onDestroy();
    unregisterFragmentWithRoot();
  }
  
  ···
}
```

分析factory.build
```
//首先查看factory的创建过程

//在initializeGlide方法中，factory可能为null
RequestManagerRetriever.RequestManagerFactory factory =
        annotationGeneratedModule != null
            ? annotationGeneratedModule.getRequestManagerFactory()
            : null;           
builder.setRequestManagerFactory(factory);

//通过GlideBuilder的setRequestManagerFactory设置requestManagerFactory
void setRequestManagerFactory(@Nullable RequestManagerFactory factory) {
    this.requestManagerFactory = factory;
}

//在builder.build中创建RequestManagerRetriever，传入requestManagerFactory
RequestManagerRetriever requestManagerRetriever =
        new RequestManagerRetriever(requestManagerFactory, experiments);
        
//查看RequestManagerRetriever的构造函数
public RequestManagerRetriever(
      @Nullable RequestManagerFactory factory, GlideExperiments experiments) {
    //如果factory为空，则有默认的实现DEFAULT_FACTORY
    this.factory = factory != null ? factory : DEFAULT_FACTORY;
    //持有主线程Looper的handler
    handler = new Handler(Looper.getMainLooper(), this /* Callback */);

    frameWaiter = buildFrameWaiter(experiments);
}

//RequestManagerFactory是一个接口
public interface RequestManagerFactory {
    @NonNull
    RequestManager build(
        @NonNull Glide glide,
        @NonNull Lifecycle lifecycle,
        @NonNull RequestManagerTreeNode requestManagerTreeNode,
        @NonNull Context context);
  }

//factory.build创建一个RequestManager实例
private static final RequestManagerFactory DEFAULT_FACTORY =
      new RequestManagerFactory() {
        @NonNull
        @Override
        public RequestManager build(
            @NonNull Glide glide,
            //lifecycle是SupportRequestManagerFragment中的ActivityFragmentLifecycle
            @NonNull Lifecycle lifecycle,
            @NonNull RequestManagerTreeNode requestManagerTreeNode,
            @NonNull Context context) {
            
          //分析RequestManager
          return new RequestManager(glide, lifecycle, requestManagerTreeNode, context);
        }
      };
```

分析RequestManager初始化过程
```
//RequestManager实现了LifecycleListener接口，能够监听宿主的生命周期

public RequestManager(
      @NonNull Glide glide,
      @NonNull Lifecycle lifecycle,
      @NonNull RequestManagerTreeNode treeNode,
      @NonNull Context context) {
    this(
        glide,
        lifecycle,
        treeNode,
        new RequestTracker(),
        glide.getConnectivityMonitorFactory(),
        context);
}

RequestManager(
      Glide glide,
      Lifecycle lifecycle,
      RequestManagerTreeNode treeNode,
      RequestTracker requestTracker,
      ConnectivityMonitorFactory factory,
      Context context) {
    this.glide = glide;
    this.lifecycle = lifecycle;
    this.treeNode = treeNode;
    this.requestTracker = requestTracker;
    this.context = context;

    //创建一个网络状态监听
    connectivityMonitor =
        factory.build(
            context.getApplicationContext(),
            new RequestManagerConnectivityListener(requestTracker));
    
    //如果在子线程，需要回到ui线程再添加生命周期监听
    //否则，直接添加监听
    if (Util.isOnBackgroundThread()) {
      Util.postOnUiThread(addSelfToLifecycle);
    } else {
      lifecycle.addListener(this);
    }
    lifecycle.addListener(connectivityMonitor);

    defaultRequestListeners =
        new CopyOnWriteArrayList<>(glide.getGlideContext().getDefaultRequestListeners());
    setRequestOptions(glide.getGlideContext().getDefaultRequestOptions());

    glide.registerRequestManager(this);
}
```

RequestManager中生命周期的回调
```
public class RequestManager
    implements ComponentCallbacks2, LifecycleListener, ModelTypes<RequestBuilder<Drawable>> {
  ···
  
  @Override
  public synchronized void onStart() {
    //requestTracker.resumeRequests()，会遍历requestTracker中全部的request并执行request.begin
    resumeRequests();
    targetTracker.onStart();
  }
  
  @Override
  public synchronized void onStop() {
    pauseRequests();
    targetTracker.onStop();
  }

  @Override
  public synchronized void onDestroy() {
    targetTracker.onDestroy();
    for (Target<?> target : targetTracker.getAll()) {
      clear(target);
    }
    targetTracker.clear();
    requestTracker.clearRequests();
    lifecycle.removeListener(this);
    lifecycle.removeListener(connectivityMonitor);
    Util.removeCallbacksOnUiThread(addSelfToLifecycle);
    glide.unregisterRequestManager(this);
  }
  
  ···   
}

public class SupportRequestManagerFragment extends Fragment {
···
    @Override
    public void onStart() {
        super.onStart();
        //调用ActivityFragmentLifecycle中的onStart()
        lifecycle.onStart();
    }
···
}

//ActivityFragmentLifecycle中的onStart()
void onStart() {
    isStarted = true;
    //遍历lifecycleListeners并调用onStart()
    //在构造RequestManager时传入的lifecycle是SupportRequestManagerFragment的ActivityFragmentLifecycle
    //所以会回调到RequestManager的onStart()
    for (LifecycleListener lifecycleListener : Util.getSnapshot(lifecycleListeners)) {
      lifecycleListener.onStart();
    }
}
```


### RequestManager#load

load的重载方法有很多，以String的重载方法为例子
```
@Override
  public RequestBuilder<Drawable> load(@Nullable String string) {
    //分为两步
    //asDrawable()
    //load(string)
    return asDrawable().load(string);
}
```

分析asDrawable()
```
public RequestBuilder<Drawable> asDrawable() {
    return as(Drawable.class);
}

//创建一个RequestBuilder
public <ResourceType> RequestBuilder<ResourceType> as(
      @NonNull Class<ResourceType> resourceClass) {
    //分析RequestBuilder
    return new RequestBuilder<>(glide, this, resourceClass, context);
}
```

分析load(string)
```
public RequestBuilder<TranscodeType> load(@Nullable String string) {
    return loadGeneric(string);
}

private RequestBuilder<TranscodeType> loadGeneric(@Nullable Object model) {
    if (isAutoCloneEnabled()) {
      return clone().loadGeneric(model);
    }
    this.model = model; //以string为例子，model就是图片的地址
    isModelSet = true;
    return selfOrThrowIfLocked();
}
```

分析RequestBuilder
```
protected RequestBuilder(
      @NonNull Glide glide,
      RequestManager requestManager,
      Class<TranscodeType> transcodeClass,
      Context context) {
    this.glide = glide;
    this.requestManager = requestManager;
    this.transcodeClass = transcodeClass;  //例如Drawable.class
    this.context = context;
    this.transitionOptions = requestManager.getDefaultTransitionOptions(transcodeClass);
    this.glideContext = glide.getGlideContext();
    
    //添加请求监听
    initRequestListeners(requestManager.getDefaultRequestListeners());
    //应用配置
    apply(requestManager.getDefaultRequestOptions());
}
```

### RequestBuilder#into(ImageView)

```
@NonNull
public ViewTarget<ImageView, TranscodeType> into(@NonNull ImageView view) {
    //必须在主线程直线
    //检查是否是主线程
    Util.assertMainThread();
    Preconditions.checkNotNull(view);

    BaseRequestOptions<?> requestOptions = this;
    if (!requestOptions.isTransformationSet()
        && requestOptions.isTransformationAllowed()
        && view.getScaleType() != null) {
      // Clone in this method so that if we use this RequestBuilder to load into a View and then
      // into a different target, we don't retain the transformation applied based on the previous
      // View's scale type.
      
      //使用requestOptions.clone()克隆一个RequestOptions，可以加载到不同的目标上
      //根据缩放类型进行转换
      switch (view.getScaleType()) {
        case CENTER_CROP:
          requestOptions = requestOptions.clone().optionalCenterCrop();
          break;
        case CENTER_INSIDE:
          requestOptions = requestOptions.clone().optionalCenterInside();
          break;
        case FIT_CENTER:
        case FIT_START:
        case FIT_END:
          requestOptions = requestOptions.clone().optionalFitCenter();
          break;
        case FIT_XY:
          requestOptions = requestOptions.clone().optionalCenterInside();
          break;
        case CENTER:
        case MATRIX:
        default:
          // Do nothing.
      }
    }

    return into(
        //根据返回类型创建对应的ViewTarget对象
        //分析buildImageViewTarget
        glideContext.buildImageViewTarget(view, transcodeClass),
        /*targetListener=*/ null,
        requestOptions,
        Executors.mainThreadExecutor());
}
```

进入into的重载方法
* target：目标对象
* targetListener：监听器
* options：请求配置
* callbackExecutor：回调线程执行器
```
private <Y extends Target<TranscodeType>> Y into(
      @NonNull Y target,
      @Nullable RequestListener<TranscodeType> targetListener,
      BaseRequestOptions<?> options,
      Executor callbackExecutor) {
    Preconditions.checkNotNull(target);
    
    //检查加载的路径
    if (!isModelSet) {
      throw new IllegalArgumentException("You must call #load() before calling #into()");
    }

    //创建一个请求对象
    //分析buildRequest
    Request request = buildRequest(target, targetListener, options, callbackExecutor);

    ···

    return target;
}
```

分析buildImageViewTarget
```
public <X> ViewTarget<ImageView, X> buildImageViewTarget(
      @NonNull ImageView imageView, @NonNull Class<X> transcodeClass) {
    //transcodeClass = Drawable.class
    return imageViewTargetFactory.buildTarget(imageView, transcodeClass);
}

//ImageViewTargetFactory
//transcodeClass是Drawable.class，返回DrawableImageViewTarget
public class ImageViewTargetFactory {
  @NonNull
  @SuppressWarnings("unchecked")
  public <Z> ViewTarget<ImageView, Z> buildTarget(
      @NonNull ImageView view, @NonNull Class<Z> clazz) {
    if (Bitmap.class.equals(clazz)) {
      return (ViewTarget<ImageView, Z>) new BitmapImageViewTarget(view);
    } else if (Drawable.class.isAssignableFrom(clazz)) {
      return (ViewTarget<ImageView, Z>) new DrawableImageViewTarget(view);
    } else {
      throw new IllegalArgumentException(
          "Unhandled class: " + clazz + ", try .as*(Class).transcode(ResourceTranscoder)");
    }
  }
}
```

回到into方法，查看requestManager.track方法
```
private <Y extends Target<TranscodeType>> Y into(
      @NonNull Y target,
      @Nullable RequestListener<TranscodeType> targetListener,
      BaseRequestOptions<?> options,
      Executor callbackExecutor) {
    ···
    
    //分析
    Request request = buildRequest(target, targetListener, options, callbackExecutor);

    //获取目标view上的请求
    Request previous = target.getRequest();
    //两次请求相同 && !(跳过内存缓存 && 旧的请求已完成)
    if (request.isEquivalentTo(previous)
        && !isSkipMemoryCacheWithCompletePreviousRequest(options, previous)) {
      //旧的请求没有执行
      if (!Preconditions.checkNotNull(previous).isRunning()) {
        //启动旧的请求
        previous.begin();
      }
      return target;
    }

    //requestManager中使用一个Set存储target
    requestManager.clear(target);
    //目标设置请求
    target.setRequest(request);
    //缓存目标并执行请求
    requestManager.track(target, request);

    return target;
}
```

缓存了Target，和执行请求
```
synchronized void track(@NonNull Target<?> target, @NonNull Request request) {
    targetTracker.track(target);
    //分析runRequest
    requestTracker.runRequest(request);
}

//targetTracker中使用Set<Target<?>>缓存Target
public final class TargetTracker implements LifecycleListener {
    private final Set<Target<?>> targets =
          Collections.newSetFromMap(new WeakHashMap<Target<?>, Boolean>());

    public void track(@NonNull Target<?> target) {
        targets.add(target);
    }
    ···
}
```

分析runRequest，执行请求
```
public void runRequest(@NonNull Request request) {
    //requests：Set<Request>
    requests.add(request);
    //没有暂停的话，开始请求，否则取消请求并加入等待的请求集合中
    if (!isPaused) {
      //分析begin
      request.begin();
    } else {
      request.clear();
      if (Log.isLoggable(TAG, Log.VERBOSE)) {
        Log.v(TAG, "Paused, delaying request");
      }
      pendingRequests.add(request);
    }
}
```

分析buildRequest，才能知道runRequest方法中Request的实现
```
private Request buildRequest(
      Target<TranscodeType> target,
      @Nullable RequestListener<TranscodeType> targetListener,
      BaseRequestOptions<?> requestOptions,
      Executor callbackExecutor) {
    return buildRequestRecursive(
        /*requestLock=*/ new Object(),
        target,
        targetListener,
        /*parentCoordinator=*/ null,
        transitionOptions,
        requestOptions.getPriority(),
        requestOptions.getOverrideWidth(),
        requestOptions.getOverrideHeight(),
        requestOptions,
        callbackExecutor);
}

private Request buildRequestRecursive(
      Object requestLock,
      Target<TranscodeType> target,
      @Nullable RequestListener<TranscodeType> targetListener,
      @Nullable RequestCoordinator parentCoordinator,
      TransitionOptions<?, ? super TranscodeType> transitionOptions,
      Priority priority,
      int overrideWidth,
      int overrideHeight,
      BaseRequestOptions<?> requestOptions,
      Executor callbackExecutor) {

    // Build the ErrorRequestCoordinator first if necessary so we can update parentCoordinator.
    //首先创建ErrorRequestCoordinator
    ErrorRequestCoordinator errorRequestCoordinator = null;
    if (errorBuilder != null) {
      errorRequestCoordinator = new ErrorRequestCoordinator(requestLock, parentCoordinator);
      parentCoordinator = errorRequestCoordinator;
    }

    //主要的请求
    //分析buildThumbnailRequestRecursive
    Request mainRequest =
        buildThumbnailRequestRecursive(
            requestLock,
            target,
            targetListener,
            parentCoordinator,
            transitionOptions,
            priority,
            overrideWidth,
            overrideHeight,
            requestOptions,
            callbackExecutor);

    if (errorRequestCoordinator == null) {
      return mainRequest;
    }

    int errorOverrideWidth = errorBuilder.getOverrideWidth();
    int errorOverrideHeight = errorBuilder.getOverrideHeight();
    if (Util.isValidDimensions(overrideWidth, overrideHeight) && !errorBuilder.isValidOverride()) {
      errorOverrideWidth = requestOptions.getOverrideWidth();
      errorOverrideHeight = requestOptions.getOverrideHeight();
    }

    Request errorRequest =
        errorBuilder.buildRequestRecursive(
            requestLock,
            target,
            targetListener,
            errorRequestCoordinator,
            errorBuilder.transitionOptions,
            errorBuilder.getPriority(),
            errorOverrideWidth,
            errorOverrideHeight,
            errorBuilder,
            callbackExecutor);
    errorRequestCoordinator.setRequests(mainRequest, errorRequest);
    return errorRequestCoordinator;
}
```

分析buildThumbnailRequestRecursive
```
private Request buildThumbnailRequestRecursive(
      Object requestLock,
      Target<TranscodeType> target,
      RequestListener<TranscodeType> targetListener,
      @Nullable RequestCoordinator parentCoordinator,
      TransitionOptions<?, ? super TranscodeType> transitionOptions,
      Priority priority,
      int overrideWidth,
      int overrideHeight,
      BaseRequestOptions<?> requestOptions,
      Executor callbackExecutor) {
      
    //判断是否有缩略图
    if (thumbnailBuilder != null) {
      // Recursive case: contains a potentially recursive thumbnail request builder.
      if (isThumbnailBuilt) {
        throw new IllegalStateException(
            "You cannot use a request as both the main request and a "
                + "thumbnail, consider using clone() on the request(s) passed to thumbnail()");
      }

      TransitionOptions<?, ? super TranscodeType> thumbTransitionOptions =
          thumbnailBuilder.transitionOptions;

      // Apply our transition by default to thumbnail requests but avoid overriding custom options
      // that may have been applied on the thumbnail request explicitly.
      if (thumbnailBuilder.isDefaultTransitionOptionsSet) {
        thumbTransitionOptions = transitionOptions;
      }

      Priority thumbPriority =
          thumbnailBuilder.isPrioritySet()
              ? thumbnailBuilder.getPriority()
              : getThumbnailPriority(priority);

      int thumbOverrideWidth = thumbnailBuilder.getOverrideWidth();
      int thumbOverrideHeight = thumbnailBuilder.getOverrideHeight();
      if (Util.isValidDimensions(overrideWidth, overrideHeight)
          && !thumbnailBuilder.isValidOverride()) {
        //设置了override(width, height)
        thumbOverrideWidth = requestOptions.getOverrideWidth();
        thumbOverrideHeight = requestOptions.getOverrideHeight();
      }

      //创建一个关联真正请求和缩略图请求的协调者
      ThumbnailRequestCoordinator coordinator =
          new ThumbnailRequestCoordinator(requestLock, parentCoordinator);
          
      //获取一个真正的请求
      //分析obtainRequest
      Request fullRequest =
          obtainRequest(
              requestLock,
              target,
              targetListener,
              requestOptions,
              coordinator,
              transitionOptions,
              priority,
              overrideWidth,
              overrideHeight,
              callbackExecutor);
      isThumbnailBuilt = true;
      // Recursively generate thumbnail requests.
      
      //创意一个缩略图请求
      Request thumbRequest =
          thumbnailBuilder.buildRequestRecursive(
              requestLock,
              target,
              targetListener,
              coordinator,
              thumbTransitionOptions,
              thumbPriority,
              thumbOverrideWidth,
              thumbOverrideHeight,
              thumbnailBuilder,
              callbackExecutor);
      isThumbnailBuilt = false;
      coordinator.setRequests(fullRequest, thumbRequest);
      return coordinator;
    } else if (thumbSizeMultiplier != null) {
      //设置了thumbSizeMultiplier缩略图比例
    
      // Base case: thumbnail multiplier generates a thumbnail request, but cannot recurse.
      ThumbnailRequestCoordinator coordinator =
          new ThumbnailRequestCoordinator(requestLock, parentCoordinator);
      Request fullRequest =
          obtainRequest(
              requestLock,
              target,
              targetListener,
              requestOptions,
              coordinator,
              transitionOptions,
              priority,
              overrideWidth,
              overrideHeight,
              callbackExecutor);
      BaseRequestOptions<?> thumbnailOptions =
          requestOptions.clone().sizeMultiplier(thumbSizeMultiplier);

      Request thumbnailRequest =
          obtainRequest(
              requestLock,
              target,
              targetListener,
              thumbnailOptions,
              coordinator,
              transitionOptions,
              getThumbnailPriority(priority),
              overrideWidth,
              overrideHeight,
              callbackExecutor);

      coordinator.setRequests(fullRequest, thumbnailRequest);
      return coordinator;
    } else {
      //没有缩略图
        
      // Base case: no thumbnail.
      return obtainRequest(
          requestLock,
          target,
          targetListener,
          requestOptions,
          parentCoordinator,
          transitionOptions,
          priority,
          overrideWidth,
          overrideHeight,
          callbackExecutor);
    }
}
```

分析obtainRequest
```
private Request obtainRequest(
      Object requestLock,
      Target<TranscodeType> target,
      RequestListener<TranscodeType> targetListener,
      BaseRequestOptions<?> requestOptions,
      RequestCoordinator requestCoordinator,
      TransitionOptions<?, ? super TranscodeType> transitionOptions,
      Priority priority,
      int overrideWidth,
      int overrideHeight,
      Executor callbackExecutor) {
    return SingleRequest.obtain(
        context,
        glideContext,
        requestLock,
        model,
        transcodeClass,
        requestOptions,
        overrideWidth,
        overrideHeight,
        priority,
        target,
        targetListener,
        requestListeners,
        requestCoordinator,
        glideContext.getEngine(),
        transitionOptions.getTransitionFactory(),
        callbackExecutor);
}

//最后创建了一个SingleRequest实例
public static <R> SingleRequest<R> obtain(
      Context context,
      GlideContext glideContext,
      Object requestLock,
      Object model,
      Class<R> transcodeClass,
      BaseRequestOptions<?> requestOptions,
      int overrideWidth,
      int overrideHeight,
      Priority priority,
      Target<R> target,
      RequestListener<R> targetListener,
      @Nullable List<RequestListener<R>> requestListeners,
      RequestCoordinator requestCoordinator,
      Engine engine,
      TransitionFactory<? super R> animationFactory,
      Executor callbackExecutor) {
    return new SingleRequest<>(
        context,
        glideContext,
        requestLock,
        model,
        transcodeClass,
        requestOptions,
        overrideWidth,
        overrideHeight,
        priority,
        target,
        targetListener,
        requestListeners,
        requestCoordinator,
        engine,
        animationFactory,
        callbackExecutor);
}
```

分析begin，已知Request是SingleRequest，SingleRequest#begin
```
public void begin() {
    //同步锁
    synchronized (requestLock) {
      assertNotCallingCallbacks();
      //如果回收了，则抛出异常
      stateVerifier.throwIfRecycled();
      startTime = LogTime.getLogTime();
      
      //model是请求的路径，url、uri、resId或者filePath等
      //model为null，则回调失败
      if (model == null) {
        if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
          width = overrideWidth;
          height = overrideHeight;
        }

        int logLevel = getFallbackDrawable() == null ? Log.WARN : Log.DEBUG;
        onLoadFailed(new GlideException("Received null model"), logLevel);
        return;
      }

      //运行中，抛异常
      if (status == Status.RUNNING) {
        throw new IllegalArgumentException("Cannot restart a running request");
      }
      
      //状态为完成则回调onResourceReady
      if (status == Status.COMPLETE) {
        onResourceReady(
            resource, DataSource.MEMORY_CACHE, /* isLoadedFromAlternateCacheKey= */ false);
        return;
      }

      //回调onRequestStarted
      experimentalNotifyRequestStarted(model);

      cookie = GlideTrace.beginSectionAsync(TAG);
      status = Status.WAITING_FOR_SIZE;
      
      //如果配置中的宽和高都>0，并且等于目标的宽和高，则直接调用onSizeReady
      if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
        //使用override()指定宽和高
        //分析onSizeReady
        onSizeReady(overrideWidth, overrideHeight);
      } else {
        //重新计算目标View的宽高，最终还是调用onSizeReady
        target.getSize(this);
      }

      if ((status == Status.RUNNING || status == Status.WAITING_FOR_SIZE)
          && canNotifyStatusChanged()) {
        //加载占位图
        target.onLoadStarted(getPlaceholderDrawable());
      }
    }
}
```

分析onSizeReady
```
public void onSizeReady(int width, int height) {
    stateVerifier.throwIfRecycled();
    synchronized (requestLock) {
      if (status != Status.WAITING_FOR_SIZE) {
        return;
      }
      
      //修改状态为正在运行
      status = Status.RUNNING;

      float sizeMultiplier = requestOptions.getSizeMultiplier();
      this.width = maybeApplySizeMultiplier(width, sizeMultiplier);
      this.height = maybeApplySizeMultiplier(height, sizeMultiplier);

      //交由引擎加载资源
      //分析Engine-load
      loadStatus =
          engine.load(
              glideContext,
              model,
              requestOptions.getSignature(),
              this.width,
              this.height,
              requestOptions.getResourceClass(),
              transcodeClass,
              priority,
              requestOptions.getDiskCacheStrategy(),
              requestOptions.getTransformations(),
              requestOptions.isTransformationRequired(),
              requestOptions.isScaleOnlyOrNoTransform(),
              requestOptions.getOptions(),
              requestOptions.isMemoryCacheable(),
              requestOptions.getUseUnlimitedSourceGeneratorsPool(),
              requestOptions.getUseAnimationPool(),
              requestOptions.getOnlyRetrieveFromCache(),
              this,
              callbackExecutor);

      // This is a hack that's only useful for testing right now where loads complete synchronously
      // even though under any executor running on any thread but the main thread, the load would
      // have completed asynchronously.
      if (status != Status.RUNNING) {
        loadStatus = null;
      }
    }
}
```

分析Engine-load
```
public <R> LoadStatus load(
      GlideContext glideContext,
      Object model,
      Key signature,
      int width,
      int height,
      Class<?> resourceClass,
      Class<R> transcodeClass,
      Priority priority,
      DiskCacheStrategy diskCacheStrategy,
      Map<Class<?>, Transformation<?>> transformations,
      boolean isTransformationRequired,
      boolean isScaleOnlyOrNoTransform,
      Options options,
      boolean isMemoryCacheable,
      boolean useUnlimitedSourceExecutorPool,
      boolean useAnimationPool,
      boolean onlyRetrieveFromCache,
      ResourceCallback cb,
      Executor callbackExecutor) {
    long startTime = VERBOSE_IS_LOGGABLE ? LogTime.getLogTime() : 0;

    //创建资源的key
    EngineKey key =
        keyFactory.buildKey(
            model,
            signature,
            width,
            height,
            transformations,
            resourceClass,
            transcodeClass,
            options);

    EngineResource<?> memoryResource;
    synchronized (this) {
      //根据key，从缓存中尝试加载
      //分析loadFromMemory
      memoryResource = loadFromMemory(key, isMemoryCacheable, startTime);

      ···
    }
    
    ···
}
```

分析loadFromMemory
```
private EngineResource<?> loadFromMemory(
      EngineKey key, boolean isMemoryCacheable, long startTime) {
    if (!isMemoryCacheable) {
      return null;
    }

    //从活动的资源中查找
    EngineResource<?> active = loadFromActiveResources(key);
    if (active != null) {
      if (VERBOSE_IS_LOGGABLE) {
        logWithTimeAndKey("Loaded resource from active resources", startTime, key);
      }
      return active;
    }

    //从缓存中查找
    EngineResource<?> cached = loadFromCache(key);
    if (cached != null) {
      if (VERBOSE_IS_LOGGABLE) {
        logWithTimeAndKey("Loaded resource from cache", startTime, key);
      }
      return cached;
    }

    return null;
}
```

再回到分析Engine-load，查看剩下部分内容
```
public <R> LoadStatus load(
      ··· ) {
      
    ···
    
    synchronized (this) {
      memoryResource = loadFromMemory(key, isMemoryCacheable, startTime);

      if (memoryResource == null) {
        //如果没有缓存资源，则执行请求任务
        //分析waitForExistingOrStartNewJob
        return waitForExistingOrStartNewJob(
            glideContext,
            model,
            signature,
            width,
            height,
            resourceClass,
            transcodeClass,
            priority,
            diskCacheStrategy,
            transformations,
            isTransformationRequired,
            isScaleOnlyOrNoTransform,
            options,
            isMemoryCacheable,
            useUnlimitedSourceExecutorPool,
            useAnimationPool,
            onlyRetrieveFromCache,
            cb,
            callbackExecutor,
            key,
            startTime);
      }
      
    cb.onResourceReady(
        memoryResource, DataSource.MEMORY_CACHE, /* isLoadedFromAlternateCacheKey= */ false);
    return null;
}
```

分析waitForExistingOrStartNewJob
```
private <R> LoadStatus waitForExistingOrStartNewJob(
      GlideContext glideContext,
      Object model,
      Key signature,
      int width,
      int height,
      Class<?> resourceClass,
      Class<R> transcodeClass,
      Priority priority,
      DiskCacheStrategy diskCacheStrategy,
      Map<Class<?>, Transformation<?>> transformations,
      boolean isTransformationRequired,
      boolean isScaleOnlyOrNoTransform,
      Options options,
      boolean isMemoryCacheable,
      boolean useUnlimitedSourceExecutorPool,
      boolean useAnimationPool,
      boolean onlyRetrieveFromCache,
      ResourceCallback cb,
      Executor callbackExecutor,
      EngineKey key,
      long startTime) {

    //查找是否有正在执行的任务
    //jobs：Map<Key, EngineJob<?>>
    EngineJob<?> current = jobs.get(key, onlyRetrieveFromCache);
    if (current != null) {
      current.addCallback(cb, callbackExecutor);
      
      return new LoadStatus(cb, current);
    }

    //创建一个引擎任务EngineJob
    EngineJob<R> engineJob =
        engineJobFactory.build(
            key,
            isMemoryCacheable,
            useUnlimitedSourceExecutorPool,
            useAnimationPool,
            onlyRetrieveFromCache);
    
    //创建一个执行任务DecodeJob
    //DecodeJob继承Runnable
    DecodeJob<R> decodeJob =
        decodeJobFactory.build(
            glideContext,
            model,
            key,
            signature,
            width,
            height,
            resourceClass,
            transcodeClass,
            priority,
            diskCacheStrategy,
            transformations,
            isTransformationRequired,
            isScaleOnlyOrNoTransform,
            onlyRetrieveFromCache,
            options,
            engineJob);

    //缓存engineJob
    jobs.put(key, engineJob);

    engineJob.addCallback(cb, callbackExecutor);
    //开始任务
    //分析EngineJob-start
    engineJob.start(decodeJob);

    return new LoadStatus(cb, engineJob);
}
```

分析EngineJob-start
```
public synchronized void start(DecodeJob<R> decodeJob) {
    this.decodeJob = decodeJob;
    //获取执行器
    GlideExecutor executor =
        decodeJob.willDecodeFromCache() ? diskCacheExecutor : getActiveSourceExecutor();
    //提交到线程池中执行耗时操作
    executor.execute(decodeJob);
}
```

DecodeJob是一个Runnable，executor.execute(decodeJob)会调用DecodeJob的run方法
```
@Override
  public void run() {
    ···
    DataFetcher<?> localFetcher = currentFetcher;
    try {
      if (isCancelled) {
        notifyFailed();
        return;
      }
      //分析runWrapped
      runWrapped();
    } catch (CallbackException e) {
      ···
    }
}
```

分析runWrapped
```
private void runWrapped() {
    switch (runReason) {
      case INITIALIZE:
        //获取下一阶段
        stage = getNextStage(Stage.INITIALIZE);
        //获取下一个生产者
        //分析getNextGenerator
        currentGenerator = getNextGenerator();
        //运行一个生产者
        //runGenerators
        runGenerators();
        break;
      case SWITCH_TO_SOURCE_SERVICE:
        runGenerators();
        break;
      case DECODE_DATA:
        decodeFromRetrievedData();
        break;
      default:
        throw new IllegalStateException("Unrecognized run reason: " + runReason);
    }
}
```

查看getNextStage
```
private Stage getNextStage(Stage current) {
    switch (current) {
      case INITIALIZE:
        return diskCacheStrategy.decodeCachedResource()
            ? Stage.RESOURCE_CACHE
            : getNextStage(Stage.RESOURCE_CACHE);
      case RESOURCE_CACHE:
        return diskCacheStrategy.decodeCachedData()
            ? Stage.DATA_CACHE
            : getNextStage(Stage.DATA_CACHE);
      case DATA_CACHE:
        // Skip loading from source if the user opted to only retrieve the resource from cache.
        return onlyRetrieveFromCache ? Stage.FINISHED : Stage.SOURCE;
      case SOURCE:
      case FINISHED:
        return Stage.FINISHED;
      default:
        throw new IllegalArgumentException("Unrecognized stage: " + current);
    }
}
```

分析getNextGenerator，根据不同阶段，返回对应的DataFetcherGenerator实现
```
private DataFetcherGenerator getNextGenerator() {
    switch (stage) {
      case RESOURCE_CACHE:  //缓存已处理过的资源的生产者
        return new ResourceCacheGenerator(decodeHelper, this);
      case DATA_CACHE:      //缓存原始资源的生产者
        return new DataCacheGenerator(decodeHelper, this);
      case SOURCE:          //原始资源的生产者
        return new SourceGenerator(decodeHelper, this);
      case FINISHED:
        return null;
      default:
        throw new IllegalStateException("Unrecognized stage: " + stage);
    }
}
```

分析runGenerators
```
private void runGenerators() {
    currentThread = Thread.currentThread();
    startFetchTime = LogTime.getLogTime();
    boolean isStarted = false;
    
    //循环，依次调用DataFetcherGenerator的startNext方法
    //继续获取下一阶段的DataFetcherGenerator
    while (!isCancelled
        && currentGenerator != null
        && !(isStarted = currentGenerator.startNext())) {
      stage = getNextStage(stage);
      currentGenerator = getNextGenerator();

      if (stage == Stage.SOURCE) {
        //reschedule方法中，把runReason赋值为SWITCH_TO_SOURCE_SERVICE
        //再调用runGenerators，当前的currentGenerator为SourceGenerator
        //在循环中执行SourceGenerator的startNext
        reschedule();
        return;
      }
    }
    // We've run out of stages and generators, give up.
    
    //阶段达到失败或取消时，回调失败接口
    if ((stage == Stage.FINISHED || isCancelled) && !isStarted) {
      notifyFailed();
    }
  }
```

分析SourceGenerator的startNext方法
```
@Override
public boolean startNext() {
    //dataToCache不为空的话，缓存数据
    if (dataToCache != null) {
      Object data = dataToCache;
      dataToCache = null;
      try {
        boolean isDataInCache = cacheData(data);
        if (!isDataInCache) {
          return true;
        }
      } catch (IOException e) {
       ···
      }
    }

    if (sourceCacheGenerator != null && sourceCacheGenerator.startNext()) {
      return true;
    }
    sourceCacheGenerator = null;

    loadData = null;
    boolean started = false;
    while (!started && hasNextModelLoader()) {
      //分析helper.getLoadData()
      loadData = helper.getLoadData().get(loadDataListIndex++);
      if (loadData != null
          && (helper.getDiskCacheStrategy().isDataCacheable(loadData.fetcher.getDataSource())
              || helper.hasLoadPath(loadData.fetcher.getDataClass()))) {
        started = true;
        //分析startNextLoad
        startNextLoad(loadData);
      }
    }
    return started;
}
```

分析startNextLoad，先看startNextLoad中做了什么操作，再看loadData到底是什么？
startNextLoad方法中执行了loadData的fetcher的loadData方法
```
private void startNextLoad(final LoadData<?> toStart) {
    loadData.fetcher.loadData(
        helper.getPriority(),
        new DataCallback<Object>() {
          @Override
          public void onDataReady(@Nullable Object data) {
            if (isCurrentRequest(toStart)) {
              onDataReadyInternal(toStart, data);
            }
          }

          @Override
          public void onLoadFailed(@NonNull Exception e) {
            if (isCurrentRequest(toStart)) {
              onLoadFailedInternal(toStart, e);
            }
          }
        });
}
```

分析helper.getLoadData()
```
List<LoadData<?>> getLoadData() {
    if (!isLoadDataSet) {
      isLoadDataSet = true;
      loadData.clear();
      
      //获取Registry的ModelLoader
      //分析ModelLoader
      List<ModelLoader<Object, ?>> modelLoaders = glideContext.getRegistry().getModelLoaders(model);
      //noinspection ForLoopReplaceableByForEach to improve perf
      for (int i = 0, size = modelLoaders.size(); i < size; i++) {
        ModelLoader<Object, ?> modelLoader = modelLoaders.get(i);
        //分析buildLoadData
        LoadData<?> current = modelLoader.buildLoadData(model, width, height, options);
        if (current != null) {
          loadData.add(current);
        }
      }
    }
    return loadData;
}
```

分析ModelLoader，ModelLoader是一个接口，接口方法buildLoadData返回内部类LoadData
```
public interface ModelLoader<Model, Data> {

  class LoadData<Data> {
    public final Key sourceKey;   //key
    public final List<Key> alternateKeys;
    public final DataFetcher<Data> fetcher;  //数据提取器

    public LoadData(@NonNull Key sourceKey, @NonNull DataFetcher<Data> fetcher) {
      this(sourceKey, Collections.<Key>emptyList(), fetcher);
    }

    public LoadData(
        @NonNull Key sourceKey,
        @NonNull List<Key> alternateKeys,
        @NonNull DataFetcher<Data> fetcher) {
      this.sourceKey = Preconditions.checkNotNull(sourceKey);
      this.alternateKeys = Preconditions.checkNotNull(alternateKeys);
      this.fetcher = Preconditions.checkNotNull(fetcher);
    }
  }

  @Nullable
  LoadData<Data> buildLoadData(
      @NonNull Model model, int width, int height, @NonNull Options options);

  boolean handles(@NonNull Model model);
}
```

HttpGlideUrlLoader#buildLoadData
```
@Override
  public LoadData<InputStream> buildLoadData(
      @NonNull GlideUrl model, int width, int height, @NonNull Options options) {
    // GlideUrls memoize parsed URLs so caching them saves a few object instantiations and time
    // spent parsing urls.
    GlideUrl url = model;
    if (modelCache != null) {
      url = modelCache.get(model, 0, 0);
      if (url == null) {
        modelCache.put(model, 0, 0, model);
        url = model;
      }
    }
    int timeout = options.get(TIMEOUT);
    //创建一个LoadData，传入参数fetcher：HttpUrlFetcher
    //分析HttpUrlFetcher
    return new LoadData<>(url, new HttpUrlFetcher(url, timeout));
}
```

分析HttpUrlFetcher
因为loadData.fetcher.loadData，直接查看HttpUrlFetcher#loadData
```
@Override
  public void loadData(
      @NonNull Priority priority, @NonNull DataCallback<? super InputStream> callback) {
    long startTime = LogTime.getLogTime();
    try {
      //加载数据或者重定向
      //分析loadDataWithRedirects
      InputStream result = loadDataWithRedirects(glideUrl.toURL(), 0, null, glideUrl.getHeaders());
      //最后调用onDataReady回调输入流
      //callback是分析startNextLoad中的DataCallback，回调onDataReady
      //分析callback.onDataReady
      callback.onDataReady(result);
    } catch (IOException e) {
      if (Log.isLoggable(TAG, Log.DEBUG)) {
        Log.d(TAG, "Failed to load data for url", e);
      }
      callback.onLoadFailed(e);
    } finally {
      if (Log.isLoggable(TAG, Log.VERBOSE)) {
        Log.v(TAG, "Finished http url fetcher fetch in " + LogTime.getElapsedMillis(startTime));
      }
    }
}
```

分析loadDataWithRedirects
```
private InputStream loadDataWithRedirects(
      URL url, int redirects, URL lastUrl, Map<String, String> headers) throws HttpException {
    if (redirects >= MAXIMUM_REDIRECTS) {
      throw new HttpException(
          "Too many (> " + MAXIMUM_REDIRECTS + ") redirects!", INVALID_STATUS_CODE);
    } else {
      // Comparing the URLs using .equals performs additional network I/O and is generally broken.
      // See http://michaelscharf.blogspot.com/2006/11/javaneturlequals-and-hashcode-make.html.
      try {
        if (lastUrl != null && url.toURI().equals(lastUrl.toURI())) {
          throw new HttpException("In re-direct loop", INVALID_STATUS_CODE);
        }
      } catch (URISyntaxException e) {
        // Do nothing, this is best effort.
      }
    }

    //创建连接
    urlConnection = buildAndConfigureConnection(url, headers);

    try {
      // Connect explicitly to avoid errors in decoders if connection fails.
      //连接
      urlConnection.connect();
      // Set the stream so that it's closed in cleanup to avoid resource leaks. See #2352.
      
      //获取输入流
      stream = urlConnection.getInputStream();
    } catch (IOException e) {
      throw new HttpException(
          "Failed to connect or obtain data", getHttpStatusCodeOrInvalid(urlConnection), e);
    }

    if (isCancelled) {
      return null;
    }

    final int statusCode = getHttpStatusCodeOrInvalid(urlConnection);
    if (isHttpOk(statusCode)) {
      //请求成功，从连接中获取资源流
      //分析getStreamForSuccessfulRequest
      return getStreamForSuccessfulRequest(urlConnection);
    } else if (isHttpRedirect(statusCode)) {
      //重定向
      String redirectUrlString = urlConnection.getHeaderField(REDIRECT_HEADER_FIELD);
      if (TextUtils.isEmpty(redirectUrlString)) {
        throw new HttpException("Received empty or null redirect url", statusCode);
      }
      URL redirectUrl;
      try {
        redirectUrl = new URL(url, redirectUrlString);
      } catch (MalformedURLException e) {
        throw new HttpException("Bad redirect url: " + redirectUrlString, statusCode, e);
      }
      // Closing the stream specifically is required to avoid leaking ResponseBodys in addition
      // to disconnecting the url connection below. See #2352.
      cleanup();
      return loadDataWithRedirects(redirectUrl, redirects + 1, url, headers);
    } else if (statusCode == INVALID_STATUS_CODE) {
      throw new HttpException(statusCode);
    } else {
      try {
        throw new HttpException(urlConnection.getResponseMessage(), statusCode);
      } catch (IOException e) {
        throw new HttpException("Failed to get a response message", statusCode, e);
      }
    }
}
```

分析getStreamForSuccessfulRequest
```
private InputStream getStreamForSuccessfulRequest(HttpURLConnection urlConnection)
      throws HttpException {
    try {
      if (TextUtils.isEmpty(urlConnection.getContentEncoding())) {
        int contentLength = urlConnection.getContentLength();
        stream = ContentLengthInputStream.obtain(urlConnection.getInputStream(), contentLength);
      } else {
        if (Log.isLoggable(TAG, Log.DEBUG)) {
          Log.d(TAG, "Got non empty content encoding: " + urlConnection.getContentEncoding());
        }
        stream = urlConnection.getInputStream();
      }
    } catch (IOException e) {
      throw new HttpException(
          "Failed to obtain InputStream", getHttpStatusCodeOrInvalid(urlConnection), e);
    }
    return stream;
}
```

当成功请求到数据后，通过DataCallback的onDataReady返回
```
InputStream result = loadDataWithRedirects(glideUrl.toURL(), 0, null, glideUrl.getHeaders());
callback.onDataReady(result);

private void startNextLoad(final LoadData<?> toStart) {
    loadData.fetcher.loadData(
        helper.getPriority(),
        new DataCallback<Object>() {
          @Override
          public void onDataReady(@Nullable Object data) {
            if (isCurrentRequest(toStart)) {
              //分析onDataReadyInternal
              onDataReadyInternal(toStart, data);
            }
          }

          @Override
          public void onLoadFailed(@NonNull Exception e) {
            if (isCurrentRequest(toStart)) {
              onLoadFailedInternal(toStart, e);
            }
          }
        });
}
```

分析onDataReadyInternal
```
@Synthetic
void onDataReadyInternal(LoadData<?> loadData, Object data) {
    //获取磁盘缓存策略
    DiskCacheStrategy diskCacheStrategy = helper.getDiskCacheStrategy();
    if (data != null && diskCacheStrategy.isDataCacheable(loadData.fetcher.getDataSource())) {
      dataToCache = data;
      // We might be being called back on someone else's thread. Before doing anything, we should
      // reschedule to get back onto Glide's thread.
      cb.reschedule();
    } else {
      //分析onDataFetcherReady
      cb.onDataFetcherReady(
          loadData.sourceKey,
          data,
          loadData.fetcher,
          loadData.fetcher.getDataSource(),
          originalKey);
    }
}
```

分析onDataFetcherReady
```
@Override
public void onDataFetcherReady(
      Key sourceKey, Object data, DataFetcher<?> fetcher, DataSource dataSource, Key attemptedKey) {
    this.currentSourceKey = sourceKey;
    this.currentData = data;
    this.currentFetcher = fetcher;
    this.currentDataSource = dataSource;
    this.currentAttemptingKey = attemptedKey;
    this.isLoadingFromAlternateCacheKey = sourceKey != decodeHelper.getCacheKeys().get(0);
    
    //如果不是当前的子线程，则通过执行一个任务回到子线程中
    if (Thread.currentThread() != currentThread) {
      //状态改为解析数据
      runReason = RunReason.DECODE_DATA;
      callback.reschedule(this);
    } else {
      GlideTrace.beginSection("DecodeJob.decodeFromRetrievedData");
      try {
        //分析decodeFromRetrievedData
        decodeFromRetrievedData();
      } finally {
        GlideTrace.endSection();
      }
    }
}
```

回到DecodeJob#runWrapped方法，查看decodeFromRetrievedData方法
```
private void runWrapped() {
    switch (runReason) {
      case INITIALIZE:
        stage = getNextStage(Stage.INITIALIZE);
        currentGenerator = getNextGenerator();
        runGenerators();
        break;
      case SWITCH_TO_SOURCE_SERVICE:
        runGenerators();
        break;
      case DECODE_DATA:
        //解析数据
        //分析decodeFromRetrievedData
        decodeFromRetrievedData();
        break;
      default:
        throw new IllegalStateException("Unrecognized run reason: " + runReason);
    }
}
```

分析decodeFromRetrievedData
```
private void decodeFromRetrievedData() {
    ···
    Resource<R> resource = null;
    try {
      //解析数据
      //分析decodeFromData
      resource = decodeFromData(currentFetcher, currentData, currentDataSource);
    } catch (GlideException e) {
      e.setLoggingDetails(currentAttemptingKey, currentDataSource);
      throwables.add(e);
    }
    if (resource != null) {
      notifyEncodeAndRelease(resource, currentDataSource, isLoadingFromAlternateCacheKey);
    } else {
      runGenerators();
    }
}
```

分析decodeFromData
```
private <Data> Resource<R> decodeFromData(
      DataFetcher<?> fetcher, Data data, DataSource dataSource) throws GlideException {
    try {
      if (data == null) {
        return null;
      }
      long startTime = LogTime.getLogTime();
      //跟踪decodeFromFetcher
      Resource<R> result = decodeFromFetcher(data, dataSource);
      if (Log.isLoggable(TAG, Log.VERBOSE)) {
        logWithTimeAndKey("Decoded result " + result, startTime);
      }
      return result;
    } finally {
      fetcher.cleanup();
    }
}

//decodeFromFetcher
private <Data> Resource<R> decodeFromFetcher(Data data, DataSource dataSource)
    throws GlideException {
    LoadPath<Data, ?, R> path = decodeHelper.getLoadPath((Class<Data>) data.getClass());
    //跟踪runLoadPath
    return runLoadPath(data, dataSource, path);
}

//runLoadPath
private <Data, ResourceType> Resource<R> runLoadPath(
      Data data, DataSource dataSource, LoadPath<Data, ResourceType, R> path)
      throws GlideException {
    Options options = getOptionsWithHardwareConfig(dataSource);
    DataRewinder<Data> rewinder = glideContext.getRegistry().getRewinder(data);
    try {
      // ResourceType in DecodeCallback below is required for compilation to work with gradle.
      
      //跟踪LoadPath#load
      return path.load(
          rewinder, options, width, height, new DecodeCallback<ResourceType>(dataSource));
    } finally {
      rewinder.cleanup();
    }
}

//LoadPath#load
public Resource<Transcode> load(
      DataRewinder<Data> rewinder,
      @NonNull Options options,
      int width,
      int height,
      DecodePath.DecodeCallback<ResourceType> decodeCallback)
      throws GlideException {
    List<Throwable> throwables = Preconditions.checkNotNull(listPool.acquire());
    try {
      //跟踪loadWithExceptionList
      return loadWithExceptionList(rewinder, options, width, height, decodeCallback, throwables);
    } finally {
      listPool.release(throwables);
    }
}

//loadWithExceptionList，返回Resource<Transcode>
private Resource<Transcode> loadWithExceptionList(
      DataRewinder<Data> rewinder,
      @NonNull Options options,
      int width,
      int height,
      DecodePath.DecodeCallback<ResourceType> decodeCallback,
      List<Throwable> exceptions)
      throws GlideException {
    Resource<Transcode> result = null;
    //noinspection ForLoopReplaceableByForEach to improve perf
    for (int i = 0, size = decodePaths.size(); i < size; i++) {
      DecodePath<Data, ResourceType, Transcode> path = decodePaths.get(i);
      try {
        result = path.decode(rewinder, width, height, options, decodeCallback);
      } catch (GlideException e) {
        exceptions.add(e);
      }
      if (result != null) {
        break;
      }
    }

    if (result == null) {
      throw new GlideException(failureMessage, new ArrayList<>(exceptions));
    }

    return result;
}
```

回到decodeFromRetrievedData，decodeFromData方法返回解析的数据
```
private void decodeFromRetrievedData() {
    ···
    
    Resource<R> resource = null;
    try {
      resource = decodeFromData(currentFetcher, currentData, currentDataSource);
    } catch (GlideException e) {
      e.setLoggingDetails(currentAttemptingKey, currentDataSource);
      throwables.add(e);
    }
    if (resource != null) {
      //分析notifyEncodeAndRelease
      notifyEncodeAndRelease(resource, currentDataSource, isLoadingFromAlternateCacheKey);
    } else {
      runGenerators();
    }
}
```

分析DecodeJob#notifyEncodeAndRelease
```
private void notifyEncodeAndRelease(
      Resource<R> resource, DataSource dataSource, boolean isLoadedFromAlternateCacheKey) {
    GlideTrace.beginSection("DecodeJob.notifyEncodeAndRelease");
    try {
      if (resource instanceof Initializable) {
        ((Initializable) resource).initialize();
      }

      Resource<R> result = resource;
      LockedResource<R> lockedResource = null;
      if (deferredEncodeManager.hasResourceToEncode()) {
        lockedResource = LockedResource.obtain(resource);
        result = lockedResource;
      }

      //分析notifyComplete
      notifyComplete(result, dataSource, isLoadedFromAlternateCacheKey);

      stage = Stage.ENCODE;
      try {
        if (deferredEncodeManager.hasResourceToEncode()) {
          deferredEncodeManager.encode(diskCacheProvider, options);
        }
      } finally {
        if (lockedResource != null) {
          lockedResource.unlock();
        }
      }

      onEncodeComplete();
    } finally {
      GlideTrace.endSection();
    }
}
```

分析notifyComplete
在Engine类中，创建DecodeJob时传入的callback是EngineJob
则调用EngineJob的onResourceReady方法
```
private void notifyComplete(
      Resource<R> resource, DataSource dataSource, boolean isLoadedFromAlternateCacheKey) {
    setNotifiedOrThrow();
    //分析onResourceReady
    callback.onResourceReady(resource, dataSource, isLoadedFromAlternateCacheKey);
}
```

分析onResourceReady，EngineJob#onResourceReady
```
@Override
public void onResourceReady(
      Resource<R> resource, DataSource dataSource, boolean isLoadedFromAlternateCacheKey) {
    synchronized (this) {
      this.resource = resource;
      this.dataSource = dataSource;
      this.isLoadedFromAlternateCacheKey = isLoadedFromAlternateCacheKey;
    }
    //分析notifyCallbacksOfResult
    notifyCallbacksOfResult();
}
```

分析notifyCallbacksOfResult
```
@Synthetic
void notifyCallbacksOfResult() {
    ResourceCallbacksAndExecutors copy;
    Key localKey;
    EngineResource<?> localResource;
    synchronized (this) {
      stateVerifier.throwIfRecycled();
      if (isCancelled) {
        resource.recycle();
        release();
        return;
      } else if (cbs.isEmpty()) {
        throw new IllegalStateException("Received a resource without any callbacks to notify");
      } else if (hasResource) {
        throw new IllegalStateException("Already have resource");
      }
      engineResource = engineResourceFactory.build(resource, isCacheable, key, resourceListener);
      
      hasResource = true;
      copy = cbs.copy();
      incrementPendingCallbacks(copy.size() + 1);

      localKey = key;
      localResource = engineResource;
    }

    //执行任务完成后的操作，缓存资源
    engineJobListener.onEngineJobComplete(this, localKey, localResource);

    for (final ResourceCallbackAndExecutor entry : copy) {
      //执行器执行CallResourceReady，回到主线程
      //分析CallResourceReady
      entry.executor.execute(new CallResourceReady(entry.cb));
    }
    decrementPendingCallbacks();
}
```

分析onEngineJobComplete
```
@Override
public synchronized void onEngineJobComplete(
      EngineJob<?> engineJob, Key key, EngineResource<?> resource) {
    // A null resource indicates that the load failed, usually due to an exception.
    if (resource != null && resource.isMemoryCacheable()) {
      activeResources.activate(key, resource);
    }

    jobs.removeIfCurrent(key, engineJob);
}
```

分析CallResourceReady，CallResourceReady#run
```
@Override
public void run() {
  // Make sure we always acquire the request lock, then the EngineJob lock to avoid deadlock
  synchronized (cb.getLock()) {
    synchronized (EngineJob.this) {
      if (cbs.contains(cb)) {
        // Acquire for this particular callback.
        engineResource.acquire();
        //跟踪callCallbackOnResourceReady
        callCallbackOnResourceReady(cb);
        removeCallback(cb);
      }
      decrementPendingCallbacks();
    }
  }
}
    
void callCallbackOnResourceReady(ResourceCallback cb) {
    try {
      //跟踪onResourceReady
      cb.onResourceReady(engineResource, dataSource, isLoadedFromAlternateCacheKey);
    } catch (Throwable t) {
      throw new CallbackException(t);
    }
}
```

SingleRequest类实现了ResourceCallback接口
查看SingleRequest的onResourceReady方法
```
@Override
public void onResourceReady(
      Resource<?> resource, DataSource dataSource, boolean isLoadedFromAlternateCacheKey) {
    stateVerifier.throwIfRecycled();
    Resource<?> toRelease = null;
    try {
      synchronized (requestLock) {
        loadStatus = null;
        if (resource == null) {
          GlideException exception =
              new GlideException(
                  "Expected to receive a Resource<R> with an "
                      + "object of "
                      + transcodeClass
                      + " inside, but instead got null.");
          onLoadFailed(exception);
          return;
        }

        Object received = resource.get();
        if (received == null || !transcodeClass.isAssignableFrom(received.getClass())) {
          toRelease = resource;
          this.resource = null;
          GlideException exception =
              new GlideException(
                  "Expected to receive an object of "
                      + transcodeClass
                      + " but instead"
                      + " got "
                      + (received != null ? received.getClass() : "")
                      + "{"
                      + received
                      + "} inside"
                      + " "
                      + "Resource{"
                      + resource
                      + "}."
                      + (received != null
                          ? ""
                          : " "
                              + "To indicate failure return a null Resource "
                              + "object, rather than a Resource object containing null data."));
          onLoadFailed(exception);
          return;
        }

        if (!canSetResource()) {
          toRelease = resource;
          this.resource = null;
          // We can't put the status to complete before asking canSetResource().
          status = Status.COMPLETE;
          GlideTrace.endSectionAsync(TAG, cookie);
          return;
        }

        //跟踪onResourceReady
        onResourceReady(
            (Resource<R>) resource, (R) received, dataSource, isLoadedFromAlternateCacheKey);
      }
    } finally {
      if (toRelease != null) {
        engine.release(toRelease);
      }
    }
}

private void onResourceReady(
      Resource<R> resource, R result, DataSource dataSource, boolean isAlternateCacheKey) {
    // We must call isFirstReadyResource before setting status.
    boolean isFirstResource = isFirstReadyResource();
    status = Status.COMPLETE;
    this.resource = resource;

    ···

    isCallingCallbacks = true;
    try {
      boolean anyListenerHandledUpdatingTarget = false;
      if (requestListeners != null) {
        for (RequestListener<R> listener : requestListeners) {
          anyListenerHandledUpdatingTarget |=
              listener.onResourceReady(result, model, target, dataSource, isFirstResource);
        }
      }
      anyListenerHandledUpdatingTarget |=
          targetListener != null
              && targetListener.onResourceReady(result, model, target, dataSource, isFirstResource);

      if (!anyListenerHandledUpdatingTarget) {
        Transition<? super R> animation = animationFactory.build(dataSource, isFirstResource);
        //调用目标的onResourceReady
        //例子中的target是DrawableImageViewTarget
        //分析target.onResourceReady
        target.onResourceReady(result, animation);
      }
    } finally {
      isCallingCallbacks = false;
    }

    notifyLoadSuccess();
    GlideTrace.endSectionAsync(TAG, cookie);
}
```

分析target.onResourceReady，DrawableImageViewTarget#onResourceReady，实际上调用
```
@Override
  public void onResourceReady(@NonNull Z resource, @Nullable Transition<? super Z> transition) {
    if (transition == null || !transition.transition(resource, this)) {
      //跟踪setResourceInternal
      setResourceInternal(resource);
    } else {
      maybeUpdateAnimatable(resource);
    }
}

private void setResourceInternal(@Nullable Z resource) {
    // Order matters here. Set the resource first to make sure that the Drawable has a valid and
    // non-null Callback before starting it.
    //分析setResource
    setResource(resource);
    maybeUpdateAnimatable(resource);
}

```

分析setResource
在load方法中是asDrawable()，在ImageViewTargetFactory中Drawable.class对应创建的是DrawableImageViewTarget
DrawableImageViewTarget#setResource
```
@Override
  protected void setResource(@Nullable Drawable resource) {
    view.setImageDrawable(resource);
}
```

into完整代码
```
private <Y extends Target<TranscodeType>> Y into(
      @NonNull Y target,
      @Nullable RequestListener<TranscodeType> targetListener,
      BaseRequestOptions<?> options,
      Executor callbackExecutor) {
    Preconditions.checkNotNull(target);
    
    //检查加载的路径
    if (!isModelSet) {
      throw new IllegalArgumentException("You must call #load() before calling #into()");
    }

    //创建一个请求对象
    Request request = buildRequest(target, targetListener, options, callbackExecutor);

    Request previous = target.getRequest();
    if (request.isEquivalentTo(previous)
        && !isSkipMemoryCacheWithCompletePreviousRequest(options, previous)) {
      //如果请求完成，重新开始将确保重新传递结果。
      //如果请求失败，重新开始将重新启动请求，使其再次有机会完成。
      //如果请求已经在运行，我们可以让它继续运行而不中断
      if (!Preconditions.checkNotNull(previous).isRunning()) {
        // Use the previous request rather than the new one to allow for optimizations like skipping
        // setting placeholders, tracking and un-tracking Targets, and obtaining View dimensions
        // that are done in the individual Request.
        previous.begin();
      }
      return target;
    }

    //requestManager中使用一个Set存储target
    requestManager.clear(target);
    //目标设置请求
    target.setRequest(request);
    //保存目标并执行请求
    requestManager.track(target, request);

    return target;
}
```




