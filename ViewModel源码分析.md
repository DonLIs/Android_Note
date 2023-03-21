# ViewModel源码分析

创建ViewModel实例的几种方式
```
//不带工厂的创建方式
val viewModel = ViewModelProvider(this).get(MainViewModel::class.java)

//带工厂的创建方式
val viewModel = ViewModelProvider(this, MainViewModelFactory()).get(MainViewModel::class.java)

class MainViewModelFactory : ViewModelProvider.Factory {
    override fun <T : ViewModel> create(modelClass: Class<T>): T {
        return MainViewModel() as T
    }
}

//kotlin中的创建方式
//使用by委托属性
val viewModel by viewModels<MainViewModel>()

//在Fragment中创建，根据作用域分两种情况
//使用Activity的作用域
val viewModel by activityViewModels<MainViewModel>()

//使用Fragment的作用域
val viewModel by viewModels<MainViewModel>()

```

> activityViewModels和viewModels区别在于使用的viewModelStore不同，即作用域不同，activityViewModels的viewModelStore是从activity中获取的，
> 而viewModels是从FragmentManager中获取的。

以上几种创建ViewModel的方式，实际上都使用了ViewModelProvider类来创建，重点查看ViewModelProvider的创建过程。
```
//ViewModelProvider无工厂的构造函数
//调用自身次构造函数，传入viewModelStore和默认工厂
public constructor(
    owner: ViewModelStoreOwner
) : this(owner.viewModelStore, defaultFactory(owner))

//默认工厂
//如果ViewModelStoreOwner是HasDefaultViewModelProviderFactory就使用ViewModelStoreOwner提供的
//否则使用instance，即NewInstanceFactory
internal fun defaultFactory(owner: ViewModelStoreOwner): Factory =
                if (owner is HasDefaultViewModelProviderFactory)
                    owner.defaultViewModelProviderFactory else instance

//instance              
@JvmStatic
public val instance: NewInstanceFactory
    @RestrictTo(RestrictTo.Scope.LIBRARY_GROUP)
    get() {
        if (sInstance == null) {
            sInstance = NewInstanceFactory()
        }
        return sInstance!!
    }
    
//其中ComponentActivity实现了HasDefaultViewModelProviderFactory接口
@NonNull
@Override
public ViewModelProvider.Factory getDefaultViewModelProviderFactory() {
    if (getApplication() == null) {
        throw new IllegalStateException("Your activity is not yet attached to the "
                + "Application instance. You can't request ViewModel before onCreate call.");
    }
    if (mDefaultFactory == null) {
        mDefaultFactory = new SavedStateViewModelFactory(
                getApplication(),
                this,
                getIntent() != null ? getIntent().getExtras() : null);
    }
    return mDefaultFactory;
}

```

查看ViewModelProvider#get方法
```
//只能在主线程调度
@MainThread
public open operator fun <T : ViewModel> get(modelClass: Class<T>): T {
    val canonicalName = modelClass.canonicalName
        ?: throw IllegalArgumentException("Local and anonymous classes can not be ViewModels")
    return get("$DEFAULT_KEY:$canonicalName", modelClass)
}

@MainThread
public open operator fun <T : ViewModel> get(key: String, modelClass: Class<T>): T {
    //store即ViewModelStore,内部使用Map<String, ViewModel>类型存储
    var viewModel = store[key]
    if (modelClass.isInstance(viewModel)) {
        (factory as? OnRequeryFactory)?.onRequery(viewModel)
        return viewModel as T
    } else {
        @Suppress("ControlFlowWithEmptyBody")
        if (viewModel != null) {
            // TODO: log a warning.
        }
    }
    //创建viewModel实例
    viewModel = if (factory is KeyedFactory) {
        factory.create(key, modelClass)
    } else {
        factory.create(modelClass)
    }
    //缓存viewModel
    store.put(key, viewModel)
    return viewModel
}
```

> ViewModel是存储在ViewModelStore中，而ViewModelStore是在ViewModelProvider的构造函数中传入。
> ComponentActivity实现了ViewModelStoreOwner接口。


查看getViewModelStore方法
```
@NonNull
@Override
public ViewModelStore getViewModelStore() {
    if (getApplication() == null) {
        throw new IllegalStateException("Your activity is not yet attached to the "
                + "Application instance. You can't request ViewModel before onCreate call.");
    }
    //重点
    ensureViewModelStore();
    return mViewModelStore;
}

void ensureViewModelStore() {
    if (mViewModelStore == null) {
        //获取非配置实例
        NonConfigurationInstances nc =
                (NonConfigurationInstances) getLastNonConfigurationInstance();
        //如果是因非配置原因导致Activity重建，获取非配置实例不为空，例如旋转屏幕。
        //这也是什么Activity重建不会导致viewModel重建。
        if (nc != null) {
            // Restore the ViewModelStore from NonConfigurationInstances
            //把非配置实例中的viewModelStore赋值给mViewModelStore
            mViewModelStore = nc.viewModelStore;
        }
        if (mViewModelStore == null) {
            mViewModelStore = new ViewModelStore();
        }
    }
}

@Nullable
public Object getLastNonConfigurationInstance() {
    return mLastNonConfigurationInstances != null
            ? mLastNonConfigurationInstances.activity : null;
}

//NonConfigurationInstances
static final class NonConfigurationInstances {
    Object activity;
    HashMap<String, Object> children;
    FragmentManagerNonConfig fragments;
    ArrayMap<String, LoaderManager> loaders;
    VoiceInteractor voiceInteractor;
}
```

那mLastNonConfigurationInstances如何获取呢？
查看Activity的attach方法
```
final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window, ActivityConfigCallback activityConfigCallback, IBinder assistToken,
            IBinder shareableActivityToken) {
    attachBaseContext(context);

    mFragments.attachHost(null /*parent*/);

    mWindow = new PhoneWindow(this, window, activityConfigCallback);
    mWindow.setWindowControllerCallback(mWindowControllerCallback);
    mWindow.setCallback(this);
    mWindow.setOnWindowDismissedCallback(this);
    mWindow.getLayoutInflater().setPrivateFactory(this);
    if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
        mWindow.setSoftInputMode(info.softInputMode);
    }
    if (info.uiOptions != 0) {
        mWindow.setUiOptions(info.uiOptions);
    }
    mUiThread = Thread.currentThread();

    mMainThread = aThread;
    mInstrumentation = instr;
    mToken = token;
    mAssistToken = assistToken;
    mShareableActivityToken = shareableActivityToken;
    mIdent = ident;
    mApplication = application;
    mIntent = intent;
    mReferrer = referrer;
    mComponent = intent.getComponent();
    mActivityInfo = info;
    mTitle = title;
    mParent = parent;
    mEmbeddedID = id;
    
    //在这里把外部传入的lastNonConfigurationInstances赋值给mLastNonConfigurationInstances
    mLastNonConfigurationInstances = lastNonConfigurationInstances;
    if (voiceInteractor != null) {
        if (lastNonConfigurationInstances != null) {
            mVoiceInteractor = lastNonConfigurationInstances.voiceInteractor;
        } else {
            mVoiceInteractor = new VoiceInteractor(voiceInteractor, this, this,
                    Looper.myLooper());
        }
    }

    mWindow.setWindowManager(
            (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
            mToken, mComponent.flattenToString(),
            (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
    if (mParent != null) {
        mWindow.setContainer(mParent.getWindow());
    }
    mWindowManager = mWindow.getWindowManager();
    mCurrentConfig = config;

    mWindow.setColorMode(info.colorMode);
    mWindow.setPreferMinimalPostProcessing(
            (info.flags & ActivityInfo.FLAG_PREFER_MINIMAL_POST_PROCESSING) != 0);

    setAutofillOptions(application.getAutofillOptions());
    setContentCaptureOptions(application.getContentCaptureOptions());
}

//
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ActivityInfo aInfo = r.activityInfo;
    
    ···

    ContextImpl appContext = createBaseContextForActivity(r);
    Activity activity = null;
    
    ···

    try {
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);

        ···

        if (activity != null) {
            ···
            
            //调用attach方法，获取ActivityClientRecord的lastNonConfigurationInstances
            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor, window, r.configCallback,
                    r.assistToken, r.shareableActivityToken);

            
        }
        r.setState(ON_CREATE);

    } catch (SuperNotCalledException e) {
        throw e;

    } catch (Exception e) {
        if (!mInstrumentation.onException(activity, e)) {
            throw new RuntimeException(
                "Unable to start activity " + component
                + ": " + e.toString(), e);
        }
    }

    return activity;
}

//ActivityClientRecord是ActivityThread中的一个成员变量
//ArrayMap<IBinder, ActivityClientRecord> mActivities = new ArrayMap<>();
```

那mLastNonConfigurationInstances是什么时候进行保存呢？
```
//在ActivityThread中，执行Activity的destroy时，如果是非配置原因销毁，才执行retainNonConfigurationInstances方法
void performDestroyActivity(ActivityClientRecord r, boolean finishing,
            int configChanges, boolean getNonConfigInstance, String reason) {
    ···
    //非配置情况
    if (getNonConfigInstance) {
        try {
            //执行activity的retainNonConfigurationInstances方法
            //赋值给ActivityClientRecord的lastNonConfigurationInstances
            r.lastNonConfigurationInstances = r.activity.retainNonConfigurationInstances();
        } catch (Exception e) {
            if (!mInstrumentation.onException(r.activity, e)) {
                throw new RuntimeException("Unable to retain activity "
                        + r.intent.getComponent().toShortString() + ": " + e.toString(), e);
            }
        }
    }
    
    ···
}

//Activity中的retainNonConfigurationInstances
NonConfigurationInstances retainNonConfigurationInstances() {
    Object activity = onRetainNonConfigurationInstance();
    
    //调用onRetainNonConfigurationChildInstances，ComponentActivity重写了该方法
    HashMap<String, Object> children = onRetainNonConfigurationChildInstances();
    FragmentManagerNonConfig fragments = mFragments.retainNestedNonConfig();

    // We're already stopped but we've been asked to retain.
    // Our fragments are taken care of but we need to mark the loaders for retention.
    // In order to do this correctly we need to restart the loaders first before
    // handing them off to the next activity.
    mFragments.doLoaderStart();
    mFragments.doLoaderStop(true);
    ArrayMap<String, LoaderManager> loaders = mFragments.retainLoaderNonConfig();

    if (activity == null && children == null && fragments == null && loaders == null
            && mVoiceInteractor == null) {
        return null;
    }

    NonConfigurationInstances nci = new NonConfigurationInstances();
    nci.activity = activity;
    nci.children = children;
    nci.fragments = fragments;
    nci.loaders = loaders;
    if (mVoiceInteractor != null) {
        mVoiceInteractor.retainInstance();
        nci.voiceInteractor = mVoiceInteractor;
    }
    return nci;
}

@Override
@Nullable
@SuppressWarnings("deprecation")
public final Object onRetainNonConfigurationInstance() {
    // Maintain backward compatibility.
    Object custom = onRetainCustomNonConfigurationInstance();

    //当前ViewModelStore实例
    ViewModelStore viewModelStore = mViewModelStore;
    if (viewModelStore == null) {
        // No one called getViewModelStore(), so see if there was an existing
        // ViewModelStore from our last NonConfigurationInstance
        NonConfigurationInstances nc =
                (NonConfigurationInstances) getLastNonConfigurationInstance();
        if (nc != null) {
            viewModelStore = nc.viewModelStore;
        }
    }

    if (viewModelStore == null && custom == null) {
        return null;
    }

    NonConfigurationInstances nci = new NonConfigurationInstances();
    nci.custom = custom;
    //保存到NonConfigurationInstances中
    nci.viewModelStore = viewModelStore;
    return nci;
}

```
