# Window、DecorView、ViewRootImpl、View之间的关系，源码分析

一般我们会在Activity的onCreate方法中，通过setContentView方法设置页面的布局来展示内容。
```
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
    }
}
```

## Activity设置布局

跟踪setContentView方法
```
@Override
public void setContentView(@LayoutRes int layoutResID) {
    //创建视图树所有者，用来监听它们的变更
    initViewTreeOwners();
    //获取委托，调用setContentView方法
    //分析getDelegate
    getDelegate().setContentView(layoutResID);
}
```

分析getDelegate
```
@NonNull
public AppCompatDelegate getDelegate() {
    if (mDelegate == null) {
        //跟踪AppCompatDelegate.create
        mDelegate = AppCompatDelegate.create(this, this);
    }
    return mDelegate;
}

//跟踪AppCompatDelegate.create
@NonNull
public static AppCompatDelegate create(@NonNull Activity activity,
        @Nullable AppCompatCallback callback) {
    return new AppCompatDelegateImpl(activity, callback);
}

//AppCompatDelegateImpl构造函数
AppCompatDelegateImpl(Activity activity, AppCompatCallback callback) {
    this(activity, null, callback, activity);
}

private AppCompatDelegateImpl(Context context, Window window, AppCompatCallback callback,
        Object host) {
    mContext = context;  //activity
    mAppCompatCallback = callback;  //callback
    mHost = host;   //activity

    //mHost是Dialog，跳过这个代码块
    if (mLocalNightMode == MODE_NIGHT_UNSPECIFIED && mHost instanceof Dialog) {
        final AppCompatActivity activity = tryUnwrapContext();
        if (activity != null) {
            mLocalNightMode = activity.getDelegate().getLocalNightMode();
        }
    }
    if (mLocalNightMode == MODE_NIGHT_UNSPECIFIED) {
        // Try and read the current night mode from our static store
        final Integer value = sLocalNightModes.get(mHost.getClass().getName());
        if (value != null) {
            mLocalNightMode = value;
            // Finally remove the value
            sLocalNightModes.remove(mHost.getClass().getName());
        }
    }

    if (window != null) {
        attachToWindow(window);
    }

    AppCompatDrawableManager.preload();
}
```

回到getDelegate().setContentView(layoutResID)
```
@Override
public void setContentView(@LayoutRes int layoutResID) {
    initViewTreeOwners();
    //分析setContentView
    getDelegate().setContentView(layoutResID);
}
```

分析setContentView
```
@Override
public void setContentView(int resId) {
    //创建一个SubDecorView
    //分析ensureSubDecor
    ensureSubDecor();
    //查找SubDecorView中id为content的布局
    ViewGroup contentParent = mSubDecor.findViewById(android.R.id.content);
    contentParent.removeAllViews();
    LayoutInflater.from(mContext).inflate(resId, contentParent);
    mAppCompatWindowCallback.getWrapped().onContentChanged();
}
```

分析ensureSubDecor
```
private void ensureSubDecor() {
    //防止重复创建DecorView
    if (!mSubDecorInstalled) {
        //分析createSubDecor
        mSubDecor = createSubDecor();

        // If a title was set before we installed the decor, propagate it now
        //设置标题
        CharSequence title = getTitle();
        if (!TextUtils.isEmpty(title)) {
            if (mDecorContentParent != null) {
                mDecorContentParent.setWindowTitle(title);
            } else if (peekSupportActionBar() != null) {
                peekSupportActionBar().setWindowTitle(title);
            } else if (mTitleView != null) {
                mTitleView.setText(title);
            }
        }

        //调整content布局的尺寸
        applyFixedSizeWindow();

        onSubDecorInstalled(mSubDecor);

        mSubDecorInstalled = true;

        //创建菜单面板
        PanelFeatureState st = getPanelState(FEATURE_OPTIONS_PANEL, false);
        if (!mDestroyed && (st == null || st.menu == null)) {
            invalidatePanelMenu(FEATURE_SUPPORT_ACTION_BAR);
        }
    }
}
```

分析createSubDecor
```
private ViewGroup createSubDecor() {
    TypedArray a = mContext.obtainStyledAttributes(R.styleable.AppCompatTheme);

    ···
    //省略了设置布局属性
    
    a.recycle();

    // Now let's make sure that the Window has installed its decor by retrieving it
    //获取Activity的mWindow赋值给AppCompatDelegateImpl的mWindow
    ensureWindow();
    
    //真正创建DecorView的地方
    //分析mWindow.getDecorView
    mWindow.getDecorView();

    //创建以activity为上下文的LayoutInflater
    final LayoutInflater inflater = LayoutInflater.from(mContext);
    ViewGroup subDecor = null;

    //根据不同主题配置，创建SubDecorView，SubDecorView是一个ViewGroup
    //有标题的主题
    if (!mWindowNoTitle) {
        if (mIsFloating) {
            // If we're floating, inflate the dialog title decor
            subDecor = (ViewGroup) inflater.inflate(
                    R.layout.abc_dialog_title_material, null);

            // Floating windows can never have an action bar, reset the flags
            mHasActionBar = mOverlayActionBar = false;
        } else if (mHasActionBar) {
            /**
             * This needs some explanation. As we can not use the android:theme attribute
             * pre-L, we emulate it by manually creating a LayoutInflater using a
             * ContextThemeWrapper pointing to actionBarTheme.
             */
            TypedValue outValue = new TypedValue();
            mContext.getTheme().resolveAttribute(R.attr.actionBarTheme, outValue, true);

            Context themedContext;
            if (outValue.resourceId != 0) {
                themedContext = new ContextThemeWrapper(mContext, outValue.resourceId);
            } else {
                themedContext = mContext;
            }

            // Now inflate the view using the themed context and set it as the content view
            subDecor = (ViewGroup) LayoutInflater.from(themedContext)
                    .inflate(R.layout.abc_screen_toolbar, null);

            mDecorContentParent = (DecorContentParent) subDecor
                    .findViewById(R.id.decor_content_parent);
            mDecorContentParent.setWindowCallback(getWindowCallback());

            /**
             * Propagate features to DecorContentParent
             */
            if (mOverlayActionBar) {
                mDecorContentParent.initFeature(FEATURE_SUPPORT_ACTION_BAR_OVERLAY);
            }
            if (mFeatureProgress) {
                mDecorContentParent.initFeature(Window.FEATURE_PROGRESS);
            }
            if (mFeatureIndeterminateProgress) {
                mDecorContentParent.initFeature(Window.FEATURE_INDETERMINATE_PROGRESS);
            }
        }
    } else {
        //无标题的主题
        if (mOverlayActionMode) {
            subDecor = (ViewGroup) inflater.inflate(
                    R.layout.abc_screen_simple_overlay_action_mode, null);
        } else {
            subDecor = (ViewGroup) inflater.inflate(R.layout.abc_screen_simple, null);
        }
    }

    ···

    if (Build.VERSION.SDK_INT >= 21) {
        // If we're running on L or above, we can rely on ViewCompat's
        // setOnApplyWindowInsetsListener
        ViewCompat.setOnApplyWindowInsetsListener(subDecor, new OnApplyWindowInsetsListener() {
                    @Override
                    public WindowInsetsCompat onApplyWindowInsets(View v,
                            WindowInsetsCompat insets) {
                        final int top = insets.getSystemWindowInsetTop();
                        final int newTop = updateStatusGuard(insets, null);

                        if (top != newTop) {
                            insets = insets.replaceSystemWindowInsets(
                                    insets.getSystemWindowInsetLeft(),
                                    newTop,
                                    insets.getSystemWindowInsetRight(),
                                    insets.getSystemWindowInsetBottom());
                        }

                        // Now apply the insets on our view
                        return ViewCompat.onApplyWindowInsets(v, insets);
                    }
                });
    } else if (subDecor instanceof FitWindowsViewGroup) {
        // Else, we need to use our own FitWindowsViewGroup handling
        ((FitWindowsViewGroup) subDecor).setOnFitSystemWindowsListener(
                new FitWindowsViewGroup.OnFitSystemWindowsListener() {
                    @Override
                    public void onFitSystemWindows(Rect insets) {
                        insets.top = updateStatusGuard(null, insets);
                    }
                });
    }

    if (mDecorContentParent == null) {
        mTitleView = (TextView) subDecor.findViewById(R.id.title);
    }

    // Make the decor optionally fit system windows, like the window's decor
    ViewUtils.makeOptionalFitsSystemWindows(subDecor);

    //获取SubDecorView中的content布局
    final ContentFrameLayout contentView = (ContentFrameLayout) subDecor.findViewById(
            R.id.action_bar_activity_content);
    
    //兼容处理
    //如果获取DecorView的windowContentView不为空，把DecorView的windowContentView中的View转移到SubDecorView的contentView中
    //设置DecorView的windowContentView的id为View.NO_ID
    //设置SubDecorView的contentView的id为R.id.content
    final ViewGroup windowContentView = (ViewGroup) mWindow.findViewById(android.R.id.content);
    if (windowContentView != null) {
        // There might be Views already added to the Window's content view so we need to
        // migrate them to our content view
        while (windowContentView.getChildCount() > 0) {
            final View child = windowContentView.getChildAt(0);
            windowContentView.removeViewAt(0);
            contentView.addView(child);
        }

        // Change our content FrameLayout to use the android.R.id.content id.
        // Useful for fragments.
        windowContentView.setId(View.NO_ID);
        contentView.setId(android.R.id.content);

        // The decorContent may have a foreground drawable set (windowContentOverlay).
        // Remove this as we handle it ourselves
        if (windowContentView instanceof FrameLayout) {
            ((FrameLayout) windowContentView).setForeground(null);
        }
    }

    // Now set the Window's content view with the decor
    //把SubDecorView添加到DecorView中
    mWindow.setContentView(subDecor);

    contentView.setAttachListener(new ContentFrameLayout.OnAttachListener() {
        @Override
        public void onAttachedFromWindow() {}

        @Override
        public void onDetachedFromWindow() {
            dismissPopups();
        }
    });

    return subDecor;
}
```

分析mWindow.getDecorView
```
mWindow.getDecorView()

//mWindow是PhoneWindow，查看PhoneWindow的getDecorView方法
@Override
public final @NonNull View getDecorView() {
    if (mDecor == null || mForceDecorInstall) {
        installDecor();
    }
    return mDecor;
}

private void installDecor() {
    mForceDecorInstall = false;
    if (mDecor == null) {
        //分析generateDecor
        mDecor = generateDecor(-1);
        mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
        mDecor.setIsRootNamespace(true);
        if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
            mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
        }
    } else {
        mDecor.setWindow(this);
    }
    
    ···
}
```

分析generateDecor
```
protected DecorView generateDecor(int featureId) {
    Context context;
    if (mUseDecorContext) {
        Context applicationContext = getContext().getApplicationContext();
        if (applicationContext == null) {
            context = getContext();
        } else {
            context = new DecorContext(applicationContext, this);
            if (mTheme != -1) {
                context.setTheme(mTheme);
            }
        }
    } else {
        context = getContext();
    }
    //创建DecorView，它继承FrameLayout
    return new DecorView(context, featureId, this, getAttributes());
}
```

回到setContentView方法
```
@Override
public void setContentView(int resId) {
    //创建一个SubDecorView
    ensureSubDecor();
    //查找SubDecorView中id为content的布局
    ViewGroup contentParent = mSubDecor.findViewById(android.R.id.content);
    //移除子View
    contentParent.removeAllViews();
    //加载我们的布局到contentParent中
    //分析inflate
    LayoutInflater.from(mContext).inflate(resId, contentParent);
    mAppCompatWindowCallback.getWrapped().onContentChanged();
}
```

分析inflate
```
public static LayoutInflater from(@UiContext Context context) {
    LayoutInflater LayoutInflater =
            (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
    if (LayoutInflater == null) {
        throw new AssertionError("LayoutInflater not found.");
    }
    return LayoutInflater;
}

//LayoutInflater#inflate
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root) {
    //root为contentParent，所以attachToRoot = true
    return inflate(resource, root, root != null);
}

public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
    final Resources res = getContext().getResources();

    //尝试加载预编译的View，内部逻辑跟inflate差不多
    View view = tryInflatePrecompiled(resource, res, root, attachToRoot);
    if (view != null) {
        return view;
    }
    
    //获取xml解析器
    XmlResourceParser parser = res.getLayout(resource);
    try {
        //分析inflate
        return inflate(parser, root, attachToRoot);
    } finally {
        parser.close();
    }
}
```

分析inflate
```
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
    synchronized (mConstructorArgs) {
        ···

        final Context inflaterContext = mContext;
        final AttributeSet attrs = Xml.asAttributeSet(parser);
        Context lastContext = (Context) mConstructorArgs[0];
        mConstructorArgs[0] = inflaterContext;
        View result = root;

        try {
            advanceToRootNode(parser);
            final String name = parser.getName();

            //merge标签，root不能为空，并且attachToRoot=true
            if (TAG_MERGE.equals(name)) {
                if (root == null || !attachToRoot) {
                    throw new InflateException("<merge /> can be used only with a valid "
                            + "ViewGroup root and attachToRoot=true");
                }

                rInflate(parser, root, inflaterContext, attrs, false);
            } else {
                // Temp is the root view that was found in the xml
                //根据标签创建View
                final View temp = createViewFromTag(root, name, inflaterContext, attrs);

                ViewGroup.LayoutParams params = null;

                //如果root不为空，attachToRoot=false，则给View添加LayoutParams
                if (root != null) {
                    // Create layout params that match root, if supplied
                    params = root.generateLayoutParams(attrs);
                    if (!attachToRoot) {
                        // Set the layout params for temp if we are not
                        // attaching. (If we are, we use addView, below)
                        temp.setLayoutParams(params);
                    }
                }

                // Inflate all children under temp against its context.
                //加载该View的子View，以递归的方式加载所有子View
                rInflateChildren(parser, temp, attrs, true);

                // We are supposed to attach all the views we found (int temp)
                // to root. Do that now.
                //如果root不为空，attachToRoot=true，则把View添加到root上
                if (root != null && attachToRoot) {
                    //添加子View
                    //分析
                    root.addView(temp, params);
                }

                // Decide whether to return the root that was passed in or the
                // top view found in xml.
                //直接返回View
                if (root == null || !attachToRoot) {
                    result = temp;
                }
            }

        } catch (XmlPullParserException e) {
            final InflateException ie = new InflateException(e.getMessage(), e);
            ie.setStackTrace(EMPTY_STACK_TRACE);
            throw ie;
        } catch (Exception e) {
            final InflateException ie = new InflateException(
                    getParserStateDescription(inflaterContext, attrs)
                    + ": " + e.getMessage(), e);
            ie.setStackTrace(EMPTY_STACK_TRACE);
            throw ie;
        } finally {
            // Don't retain static reference on context.
            mConstructorArgs[0] = lastContext;
            mConstructorArgs[1] = null;

            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }

        return result;
    }
}
```

root.addView
```
public void addView(View child, int index, LayoutParams params) {
    // addViewInner() will call child.requestLayout() when setting the new LayoutParams
    // therefore, we call requestLayout() on ourselves before, so that the child's request
    // will be blocked at our level
    requestLayout();
    invalidate(true);
    addViewInner(child, index, params, false);
}

public void requestLayout() {
    if (mMeasureCache != null) mMeasureCache.clear();

    if (mAttachInfo != null && mAttachInfo.mViewRequestingLayout == null) {
        // Only trigger request-during-layout logic if this is the view requesting it,
        // not the views in its parent hierarchy
        ViewRootImpl viewRoot = getViewRootImpl();
        if (viewRoot != null && viewRoot.isInLayout()) {
            if (!viewRoot.requestLayoutDuringLayout(this)) {
                return;
            }
        }
        mAttachInfo.mViewRequestingLayout = this;
    }

    mPrivateFlags |= PFLAG_FORCE_LAYOUT;
    mPrivateFlags |= PFLAG_INVALIDATED;

    if (mParent != null && !mParent.isLayoutRequested()) {
        mParent.requestLayout();
    }
    if (mAttachInfo != null && mAttachInfo.mViewRequestingLayout == this) {
        mAttachInfo.mViewRequestingLayout = null;
    }
}
```

## Window创建过程

设置布局的过程没看到Activity的Window创建，而又调用了mWindow的方法，这里来分析Window是如果创建的。
Window是一个抽象类，它的实现类是PhoneWindow。
```
//ActivityThread类
//查看Activity的创建
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ActivityInfo aInfo = r.activityInfo;
    ···
    
    //创建上下文实现类
    ContextImpl appContext = createBaseContextForActivity(r);
    Activity activity = null;
    //创建Activity
    try {
        java.lang.ClassLoader cl = appContext.getClassLoader();

        activity = mInstrumentation.newActivity(
                cl, component.getClassName(), r.intent);
        StrictMode.incrementExpectedActivityCount(activity.getClass());
        r.intent.setExtrasClassLoader(cl);
        r.intent.prepareToEnterProcess(isProtectedComponent(r.activityInfo),
                appContext.getAttributionSource());
        if (r.state != null) {
            r.state.setClassLoader(cl);
        }
    } catch (Exception e) {
        if (!mInstrumentation.onException(activity, e)) {
            throw new RuntimeException(
                "Unable to instantiate activity " + component
                + ": " + e.toString(), e);
        }
    }

    try {
        //创建Application
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);

        ···

        if (activity != null) {
            ···

            //调用Activity的attach
            //分析activity.attach
            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor, window, r.configCallback,
                    r.assistToken, r.shareableActivityToken);

            ···
            
            //回调onCreate，此时decorView和window已经创建完
            if (r.isPersistable()) {
                mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
            } else {
                mInstrumentation.callActivityOnCreate(activity, r.state);
            }
            
            ···
        }
        //设置Activity的状态
        r.setState(ON_CREATE);

    } catch (SuperNotCalledException e) {
        throw e;

    } catch (Exception e) {
        ···
    }

    return activity;
}
```

分析activity.attach
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

    //Window的创建，实现类是PhoneWindow
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
    //当前线程为UI线程
    mUiThread = Thread.currentThread();

    ···

    //window持有WindowManager
    mWindow.setWindowManager(
            (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
            mToken, mComponent.flattenToString(),
            (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
    if (mParent != null) {
        mWindow.setContainer(mParent.getWindow());
    }
    //获取window中的WindowManager
    mWindowManager = mWindow.getWindowManager();
    mCurrentConfig = config;

    mWindow.setColorMode(info.colorMode);
    mWindow.setPreferMinimalPostProcessing(
            (info.flags & ActivityInfo.FLAG_PREFER_MINIMAL_POST_PROCESSING) != 0);

    setAutofillOptions(application.getAutofillOptions());
    setContentCaptureOptions(application.getContentCaptureOptions());
}
```

## Activity、WindowManager、ViewRootImpl、View之间的关系

在Activity的onResume中，通过WindowManager往Window中添加DecorView
```
//ActivityThread#handleResumeActivity
@Override
public void handleResumeActivity(ActivityClientRecord r, boolean finalStateRequest,
        boolean isForward, String reason) {
    ···

    final Activity a = r.activity;

    ···

    if (r.window == null && !a.mFinished && willBeVisible) {
        r.window = r.activity.getWindow();
        View decor = r.window.getDecorView();
        decor.setVisibility(View.INVISIBLE);
        //WindowManager
        ViewManager wm = a.getWindowManager();
        //创建LayoutParams
        WindowManager.LayoutParams l = r.window.getAttributes();
        a.mDecor = decor;
        l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
        l.softInputMode |= forwardBit;
        if (r.mPreserveWindow) {
            a.mWindowAdded = true;
            r.mPreserveWindow = false;
            // Normally the ViewRoot sets up callbacks with the Activity
            // in addView->ViewRootImpl#setView. If we are instead reusing
            // the decor view we have to notify the view root that the
            // callbacks may have changed.
            ViewRootImpl impl = decor.getViewRootImpl();
            if (impl != null) {
                impl.notifyChildRebuilt();
            }
        }
        if (a.mVisibleFromClient) {
            //没有添加view就调用addView
            if (!a.mWindowAdded) {
                a.mWindowAdded = true;
                //通过WindowManager把decorView显示到屏幕上
                //分析wm.addView
                wm.addView(decor, l);
            } else {
                // The activity will get a callback for this {@link LayoutParams} change
                // earlier. However, at that time the decor will not be set (this is set
                // in this method), so no action will be taken. This call ensures the
                // callback occurs with the decor set.
                a.onWindowAttributesChanged(l);
            }
        }
    } else if (!willBeVisible) {
        r.hideForNow = true;
    }

    ···
    Looper.myQueue().addIdleHandler(new Idler());
}
```

WindowManager
```
//WindowManager继承ViewManager接口

public interface ViewManager
{
    public void addView(View view, ViewGroup.LayoutParams params);
    public void updateViewLayout(View view, ViewGroup.LayoutParams params);
    public void removeView(View view);
}

public interface WindowManager extends ViewManager {
    ···
}
```

WindowManager的实现类是WindowManagerImpl，并委托给WindowManagerGlobal实现细节
```
public class WindowManagerImpl implements WindowManager {
    ···
}
```

WindowManagerGlobal初始化
```
public Activity handleLaunchActivity(ActivityClientRecord r,
            PendingTransactionActions pendingActions, Intent customIntent) {
    ···
    WindowManagerGlobal.initialize();
    ···       
}
```

WindowManagerGlobal#
```
public void addView(View view, ViewGroup.LayoutParams params,
        Display display, Window parentWindow, int userId) {
    ···

    ViewRootImpl root;
    View panelParentView = null;

    synchronized (mLock) {
        ···

        //创建ViewRootImpl
        root = new ViewRootImpl(view.getContext(), display);

        view.setLayoutParams(wparams);

        //缓存view
        //mViews：ArrayList<View>
        mViews.add(view);
        //mRoots：ArrayList<ViewRootImpl>
        mRoots.add(root);
        mParams.add(wparams);

        // do this last because it fires off messages to start doing things
        try {
            //分析setView
            root.setView(view, wparams, panelParentView, userId);
        } catch (RuntimeException e) {
            // BadTokenException or InvalidDisplayException, clean up.
            if (index >= 0) {
                removeViewLocked(index, true);
            }
            throw e;
        }
    }
}
```

分析setView，ViewRootImpl#setView
```
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView,
        int userId) {
    synchronized (this) {
        if (mView == null) {
            mView = view;

            mAttachInfo.mDisplayState = mDisplay.getState();
            mViewLayoutDirectionInitial = mView.getRawLayoutDirection();
            mFallbackEventHandler.setView(view);
            mWindowAttributes.copyFrom(attrs);
            if (mWindowAttributes.packageName == null) {
                mWindowAttributes.packageName = mBasePackageName;
            }
            mWindowAttributes.privateFlags |=
                    WindowManager.LayoutParams.PRIVATE_FLAG_USE_BLAST;

            attrs = mWindowAttributes;
            setTag();

            // Keep track of the actual window flags supplied by the client.
            mClientWindowLayoutFlags = attrs.flags;

            setAccessibilityFocus(null, null);

            if (view instanceof RootViewSurfaceTaker) {
                mSurfaceHolderCallback =
                        ((RootViewSurfaceTaker)view).willYouTakeTheSurface();
                if (mSurfaceHolderCallback != null) {
                    mSurfaceHolder = new TakenSurfaceHolder();
                    mSurfaceHolder.setFormat(PixelFormat.UNKNOWN);
                    mSurfaceHolder.addCallback(mSurfaceHolderCallback);
                }
            }

            // Compute surface insets required to draw at specified Z value.
            // TODO: Use real shadow insets for a constant max Z.
            if (!attrs.hasManualSurfaceInsets) {
                attrs.setSurfaceInsets(view, false /*manual*/, true /*preservePrevious*/);
            }

            CompatibilityInfo compatibilityInfo =
                    mDisplay.getDisplayAdjustments().getCompatibilityInfo();
            mTranslator = compatibilityInfo.getTranslator();

            // If the application owns the surface, don't enable hardware acceleration
            if (mSurfaceHolder == null) {
                // While this is supposed to enable only, it can effectively disable
                // the acceleration too.
                enableHardwareAcceleration(attrs);
                final boolean useMTRenderer = MT_RENDERER_AVAILABLE
                        && mAttachInfo.mThreadedRenderer != null;
                if (mUseMTRenderer != useMTRenderer) {
                    // Shouldn't be resizing, as it's done only in window setup,
                    // but end just in case.
                    endDragResizing();
                    mUseMTRenderer = useMTRenderer;
                }
            }

            boolean restore = false;
            if (mTranslator != null) {
                mSurface.setCompatibilityTranslator(mTranslator);
                restore = true;
                attrs.backup();
                mTranslator.translateWindowLayout(attrs);
            }

            if (!compatibilityInfo.supportsScreen()) {
                attrs.privateFlags |= WindowManager.LayoutParams.PRIVATE_FLAG_COMPATIBLE_WINDOW;
                mLastInCompatMode = true;
            }

            mSoftInputMode = attrs.softInputMode;
            mWindowAttributesChanged = true;
            mAttachInfo.mRootView = view;
            mAttachInfo.mScalingRequired = mTranslator != null;
            mAttachInfo.mApplicationScale =
                    mTranslator == null ? 1.0f : mTranslator.applicationScale;
            if (panelParentView != null) {
                mAttachInfo.mPanelParentWindowToken
                        = panelParentView.getApplicationWindowToken();
            }
            mAdded = true;
            int res; /* = WindowManagerImpl.ADD_OKAY; */

            // Schedule the first layout -before- adding to the window
            // manager, to make sure we do the relayout before receiving
            // any other events from the system.
            //分析requestLayout
            requestLayout();
            InputChannel inputChannel = null;
            if ((mWindowAttributes.inputFeatures
                    & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
                inputChannel = new InputChannel();
            }
            mForceDecorViewVisibility = (mWindowAttributes.privateFlags
                    & PRIVATE_FLAG_FORCE_DECOR_VIEW_VISIBILITY) != 0;

            if (mView instanceof RootViewSurfaceTaker) {
                PendingInsetsController pendingInsetsController =
                        ((RootViewSurfaceTaker) mView).providePendingInsetsController();
                if (pendingInsetsController != null) {
                    pendingInsetsController.replayAndAttach(mInsetsController);
                }
            }

            try {
                mOrigWindowType = mWindowAttributes.type;
                mAttachInfo.mRecomputeGlobalAttributes = true;
                collectViewAttributes();
                adjustLayoutParamsForCompatibility(mWindowAttributes);
                controlInsetsForCompatibility(mWindowAttributes);
                //调用wms显示视图
                res = mWindowSession.addToDisplayAsUser(mWindow, mWindowAttributes,
                        getHostVisibility(), mDisplay.getDisplayId(), userId,
                        mInsetsController.getRequestedVisibilities(), inputChannel, mTempInsets,
                        mTempControls);
                if (mTranslator != null) {
                    mTranslator.translateInsetsStateInScreenToAppWindow(mTempInsets);
                    mTranslator.translateSourceControlsInScreenToAppWindow(mTempControls);
                }
            } catch (RemoteException e) {
                mAdded = false;
                mView = null;
                mAttachInfo.mRootView = null;
                mFallbackEventHandler.setView(null);
                unscheduleTraversals();
                setAccessibilityFocus(null, null);
                throw new RuntimeException("Adding window failed", e);
            } finally {
                if (restore) {
                    attrs.restore();
                }
            }

            mAttachInfo.mAlwaysConsumeSystemBars =
                    (res & WindowManagerGlobal.ADD_FLAG_ALWAYS_CONSUME_SYSTEM_BARS) != 0;
            mPendingAlwaysConsumeSystemBars = mAttachInfo.mAlwaysConsumeSystemBars;
            mInsetsController.onStateChanged(mTempInsets);
            mInsetsController.onControlsChanged(mTempControls);
            computeWindowBounds(mWindowAttributes, mInsetsController.getState(),
                    getConfiguration().windowConfiguration.getBounds(), mTmpFrames.frame);
            setFrame(mTmpFrames.frame);
            
            ···

            registerListeners();
            if ((res & WindowManagerGlobal.ADD_FLAG_USE_BLAST) != 0) {
                mUseBLASTAdapter = true;
            }

            if (view instanceof RootViewSurfaceTaker) {
                mInputQueueCallback =
                    ((RootViewSurfaceTaker)view).willYouTakeTheInputQueue();
            }
            if (inputChannel != null) {
                if (mInputQueueCallback != null) {
                    mInputQueue = new InputQueue();
                    mInputQueueCallback.onInputQueueCreated(mInputQueue);
                }
                mInputEventReceiver = new WindowInputEventReceiver(inputChannel,
                        Looper.myLooper());

                if (ENABLE_INPUT_LATENCY_TRACKING && mAttachInfo.mThreadedRenderer != null) {
                    InputMetricsListener listener = new InputMetricsListener();
                    mHardwareRendererObserver = new HardwareRendererObserver(
                            listener, listener.data, mHandler, true /*waitForPresentTime*/);
                    mAttachInfo.mThreadedRenderer.addObserver(mHardwareRendererObserver);
                }
            }

            //分析assignParent
            view.assignParent(this);
            mAddedTouchMode = (res & WindowManagerGlobal.ADD_FLAG_IN_TOUCH_MODE) != 0;
            mAppVisible = (res & WindowManagerGlobal.ADD_FLAG_APP_VISIBLE) != 0;

            if (mAccessibilityManager.isEnabled()) {
                mAccessibilityInteractionConnectionManager.ensureConnection();
            }

            if (view.getImportantForAccessibility() == View.IMPORTANT_FOR_ACCESSIBILITY_AUTO) {
                view.setImportantForAccessibility(View.IMPORTANT_FOR_ACCESSIBILITY_YES);
            }

            // Set up the input pipeline.
            CharSequence counterSuffix = attrs.getTitle();
            mSyntheticInputStage = new SyntheticInputStage();
            InputStage viewPostImeStage = new ViewPostImeInputStage(mSyntheticInputStage);
            InputStage nativePostImeStage = new NativePostImeInputStage(viewPostImeStage,
                    "aq:native-post-ime:" + counterSuffix);
            InputStage earlyPostImeStage = new EarlyPostImeInputStage(nativePostImeStage);
            InputStage imeStage = new ImeInputStage(earlyPostImeStage,
                    "aq:ime:" + counterSuffix);
            InputStage viewPreImeStage = new ViewPreImeInputStage(imeStage);
            InputStage nativePreImeStage = new NativePreImeInputStage(viewPreImeStage,
                    "aq:native-pre-ime:" + counterSuffix);

            mFirstInputStage = nativePreImeStage;
            mFirstPostImeInputStage = earlyPostImeStage;
            mPendingInputEventQueueLengthCounterName = "aq:pending:" + counterSuffix;
        }
    }
}
```

分析requestLayout
```
@Override
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        //检查是否是主线程
        checkThread();
        //用来区分是否要测量和布局，而invalidae()的mLayoutRequested为false
        mLayoutRequested = true;
        scheduleTraversals();
    }
}

scheduleTraversals() -> performMeasure()、performLayout()、performDraw()
依次调用View的measure()、layout()、draw()

scheduleTraversals不会立即调用以上三个方法，而是先发送一个同步屏障，再发送一个异步消息，等待同步监听回调时才执行View的方法。
```

分析assignParent
```
void assignParent(ViewParent parent) {
    if (mParent == null) {
        mParent = parent;
    } else if (parent == null) {
        mParent = null;
    } else {
        throw new RuntimeException("view " + this + " being added, but"
                + " it already has a parent");
    }
}

parent为ViewRootImpl，在View中可以通过mParent调用ViewRootImpl的方法。
```

## View刷新与加载

接着ViewRootImpl的requestLayout方法
```
@Override
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true;
        //跟踪scheduleTraversals
        scheduleTraversals();
    }
}
```

scheduleTraversals
```
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        //往Looper的队列中发送一个同步屏障，同步屏障能阻止同步消息执行，并优先执行异步消息
        //分析同步屏障
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        
        //在mChoreographer中注册一个CALLBACK_TRAVERSAL
        //分析mChoreographer.postCallback
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}
```

同步屏障
```
//MessageQueue类

public int postSyncBarrier() {
    return postSyncBarrier(SystemClock.uptimeMillis());
}

private int postSyncBarrier(long when) {
    // Enqueue a new sync barrier token.
    // We don't need to wake the queue because the purpose of a barrier is to stall it.
    synchronized (this) {
        final int token = mNextBarrierToken++;
        
        //创建一个Message，该Message没有target，为同步屏障消息
        final Message msg = Message.obtain();
        msg.markInUse();
        msg.when = when;
        msg.arg1 = token;

        Message prev = null;
        Message p = mMessages;
        if (when != 0) {
            while (p != null && p.when <= when) {
                prev = p;
                p = p.next;
            }
        }
        if (prev != null) { // invariant: p == prev.next
            msg.next = p;
            prev.next = msg;
        } else {
            msg.next = p;
            mMessages = msg;
        }
        return token;
    }
}
```

分析mChoreographer.postCallback
```
public void postCallback(int callbackType, Runnable action, Object token) {
    postCallbackDelayed(callbackType, action, token, 0);
}

public void postCallbackDelayed(int callbackType,
        Runnable action, Object token, long delayMillis) {
    if (action == null) {
        throw new IllegalArgumentException("action must not be null");
    }
    if (callbackType < 0 || callbackType > CALLBACK_LAST) {
        throw new IllegalArgumentException("callbackType is invalid");
    }

    postCallbackDelayedInternal(callbackType, action, token, delayMillis);
}

private void postCallbackDelayedInternal(int callbackType,
        Object action, Object token, long delayMillis) {

    synchronized (mLock) {
        final long now = SystemClock.uptimeMillis();
        final long dueTime = now + delayMillis;
        //缓存TraversalRunnable
        mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);

        //先发送一个异步消息，通过Handler回调最终还是执行scheduleFrameLocked方法
        if (dueTime <= now) {
            scheduleFrameLocked(now);
        } else {
            Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
            msg.arg1 = callbackType;  //Choreographer.CALLBACK_TRAVERSAL
            msg.setAsynchronous(true);  //设置为异步消息
            mHandler.sendMessageAtTime(msg, dueTime);
        }
    }
}
```

scheduleFrameLocked
```
private void scheduleFrameLocked(long now) {
    if (!mFrameScheduled) {
        mFrameScheduled = true;
        if (USE_VSYNC) {

            // If running on the Looper thread, then schedule the vsync immediately,
            // otherwise post a message to schedule the vsync from the UI thread
            // as soon as possible.
            
            //先发送一个异步消息，最后还是回调到scheduleVsyncLocked方法
            if (isRunningOnLooperThreadLocked()) {
                scheduleVsyncLocked();
            } else {
                Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
                msg.setAsynchronous(true);
                mHandler.sendMessageAtFrontOfQueue(msg);
            }
        } else {
            final long nextFrameTime = Math.max(
                    mLastFrameTimeNanos / TimeUtils.NANOS_PER_MS + sFrameDelay, now);
                    
            Message msg = mHandler.obtainMessage(MSG_DO_FRAME);
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, nextFrameTime);
        }
    }
}
```

scheduleVsyncLocked
```
private void scheduleVsyncLocked() {
    try {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Choreographer#scheduleVsyncLocked");
        //DisplayEventReceiver.scheduleVsync()方法会通过native方法发送一个Vsync信号
        mDisplayEventReceiver.scheduleVsync();
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}
```

Vsync信号最后会在FrameDisplayEventReceiver类的onVsync方法中回调
```
private final class FrameDisplayEventReceiver extends DisplayEventReceiver
        implements Runnable {
    private boolean mHavePendingVsync;
    private long mTimestampNanos;
    private int mFrame;
    private VsyncEventData mLastVsyncEventData = new VsyncEventData();

    public FrameDisplayEventReceiver(Looper looper, int vsyncSource) {
        super(looper, vsyncSource, 0);
    }

    @Override
    public void onVsync(long timestampNanos, long physicalDisplayId, int frame,
            VsyncEventData vsyncEventData) {
        try {
            if (Trace.isTagEnabled(Trace.TRACE_TAG_VIEW)) {
                Trace.traceBegin(Trace.TRACE_TAG_VIEW,
                        "Choreographer#onVsync " + vsyncEventData.id);
            }

            long now = System.nanoTime();
            if (timestampNanos > now) {
                Log.w(TAG, "Frame time is " + ((timestampNanos - now) * 0.000001f)
                        + " ms in the future!  Check that graphics HAL is generating vsync "
                        + "timestamps using the correct timebase.");
                timestampNanos = now;
            }

            if (mHavePendingVsync) {
                Log.w(TAG, "Already have a pending vsync event.  There should only be "
                        + "one at a time.");
            } else {
                mHavePendingVsync = true;
            }

            mTimestampNanos = timestampNanos;
            mFrame = frame;
            mLastVsyncEventData = vsyncEventData;
            
            //创建一个异步消息，传入自己，因为自己也实现了Runnable接口，会执行run方法
            Message msg = Message.obtain(mHandler, this);
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }

    @Override
    public void run() {
        mHavePendingVsync = false;
        //分析doFrame
        doFrame(mTimestampNanos, mFrame, mLastVsyncEventData);
    }
}
```

分析doFrame
```
//回顾之前方法scheduleTraversals
mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
                
//最后
mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);

void doFrame(long frameTimeNanos, int frame,
        DisplayEventReceiver.VsyncEventData vsyncEventData) {
    final long startNanos;
    final long frameIntervalNanos = vsyncEventData.frameInterval;
    try {
        ···

        AnimationUtils.lockAnimationClock(frameTimeNanos / TimeUtils.NANOS_PER_MS);

        mFrameInfo.markInputHandlingStart();
        doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos, frameIntervalNanos);

        mFrameInfo.markAnimationsStart();
        doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos, frameIntervalNanos);
        doCallbacks(Choreographer.CALLBACK_INSETS_ANIMATION, frameTimeNanos,
                frameIntervalNanos);

        mFrameInfo.markPerformTraversalsStart();
        
        //前面通过scheduleTraversals方法向Choreographer的mCallbackQueues缓存了TraversalRunnable
        //doCallbacks取出对应的type的TraversalRunnable，并执行
        //分析doCallbacks
        doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos, frameIntervalNanos);

        doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos, frameIntervalNanos);
    } finally {
        AnimationUtils.unlockAnimationClock();
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }

}
```

分析doCallbacks
```
void doCallbacks(int callbackType, long frameTimeNanos, long frameIntervalNanos) {
    CallbackRecord callbacks;
    synchronized (mLock) {
        final long now = System.nanoTime();
        //取出callbacks
        callbacks = mCallbackQueues[callbackType].extractDueCallbacksLocked(
                now / TimeUtils.NANOS_PER_MS);
        if (callbacks == null) {
            return;
        }
        mCallbacksRunning = true;

        if (callbackType == Choreographer.CALLBACK_COMMIT) {
            final long jitterNanos = now - frameTimeNanos;
            
            if (jitterNanos >= 2 * frameIntervalNanos) {
                final long lastFrameOffset = jitterNanos % frameIntervalNanos
                        + frameIntervalNanos;
                
                frameTimeNanos = now - lastFrameOffset;
                mLastFrameTimeNanos = frameTimeNanos;
            }
        }
    }
    try {
        for (CallbackRecord c = callbacks; c != null; c = c.next) {
            //执行Runnable
            c.run(frameTimeNanos);
        }
    } finally {
        synchronized (mLock) {
            mCallbacksRunning = false;
            do {
                final CallbackRecord next = callbacks.next;
                recycleCallbackLocked(callbacks);
                callbacks = next;
            } while (callbacks != null);
        }
    }
}
```

查看mTraversalRunnable
```
//回到scheduleTraversals方法
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        
        //分析mTraversalRunnable
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}

final TraversalRunnable mTraversalRunnable = new TraversalRunnable();

final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        doTraversal();
    }
}

void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        //移除同步屏障
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

        //执行View的measure、layout、draw
        performTraversals();

    }
}
```

performTraversals()，省略大部分代码
```
private void performTraversals() {
    ···
    //执行View.measure
    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
    
    //执行View.layout
    performLayout(lp, mWidth, mHeight);
  
    //执行View.draw
    performDraw();
    ···  
}
```

performTraversals()，省略DEBUG代码，完整代码
```
private void performTraversals() {
    final View host = mView;

    if (host == null || !mAdded) {
        return;
    }

    if (mWaitForBlastSyncComplete) {
        mRequestedTraverseWhilePaused = true;
        return;
    }

    mIsInTraversal = true;
    mWillDrawSoon = true;
    boolean windowSizeMayChange = false;
    WindowManager.LayoutParams lp = mWindowAttributes;

    int desiredWindowWidth;
    int desiredWindowHeight;

    final int viewVisibility = getHostVisibility();
    final boolean viewVisibilityChanged = !mFirst
            && (mViewVisibility != viewVisibility || mNewSurfaceNeeded
            // Also check for possible double visibility update, which will make current
            // viewVisibility value equal to mViewVisibility and we may miss it.
            || mAppVisibilityChanged);
    mAppVisibilityChanged = false;
    final boolean viewUserVisibilityChanged = !mFirst &&
            ((mViewVisibility == View.VISIBLE) != (viewVisibility == View.VISIBLE));

    WindowManager.LayoutParams params = null;
    CompatibilityInfo compatibilityInfo =
            mDisplay.getDisplayAdjustments().getCompatibilityInfo();
    if (compatibilityInfo.supportsScreen() == mLastInCompatMode) {
        params = lp;
        mFullRedrawNeeded = true;
        mLayoutRequested = true;
        if (mLastInCompatMode) {
            params.privateFlags &= ~WindowManager.LayoutParams.PRIVATE_FLAG_COMPATIBLE_WINDOW;
            mLastInCompatMode = false;
        } else {
            params.privateFlags |= WindowManager.LayoutParams.PRIVATE_FLAG_COMPATIBLE_WINDOW;
            mLastInCompatMode = true;
        }
    }

    Rect frame = mWinFrame;
    if (mFirst) {
        mFullRedrawNeeded = true;
        mLayoutRequested = true;

        final Configuration config = getConfiguration();
        if (shouldUseDisplaySize(lp)) {
            // NOTE -- system code, won't try to do compat mode.
            Point size = new Point();
            mDisplay.getRealSize(size);
            desiredWindowWidth = size.x;
            desiredWindowHeight = size.y;
        } else if (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT
                || lp.height == ViewGroup.LayoutParams.WRAP_CONTENT) {
            // For wrap content, we have to remeasure later on anyways. Use size consistent with
            // below so we get best use of the measure cache.
            final Rect bounds = getWindowBoundsInsetSystemBars();
            desiredWindowWidth = bounds.width();
            desiredWindowHeight = bounds.height();
        } else {
            // After addToDisplay, the frame contains the frameHint from window manager, which
            // for most windows is going to be the same size as the result of relayoutWindow.
            // Using this here allows us to avoid remeasuring after relayoutWindow
            desiredWindowWidth = frame.width();
            desiredWindowHeight = frame.height();
        }

        mAttachInfo.mUse32BitDrawingCache = true;
        mAttachInfo.mWindowVisibility = viewVisibility;
        mAttachInfo.mRecomputeGlobalAttributes = false;
        mLastConfigurationFromResources.setTo(config);
        mLastSystemUiVisibility = mAttachInfo.mSystemUiVisibility;
        // Set the layout direction if it has not been set before (inherit is the default)
        if (mViewLayoutDirectionInitial == View.LAYOUT_DIRECTION_INHERIT) {
            host.setLayoutDirection(config.getLayoutDirection());
        }
        host.dispatchAttachedToWindow(mAttachInfo, 0);
        mAttachInfo.mTreeObserver.dispatchOnWindowAttachedChange(true);
        dispatchApplyInsets(host);
    } else {
        desiredWindowWidth = frame.width();
        desiredWindowHeight = frame.height();
        if (desiredWindowWidth != mWidth || desiredWindowHeight != mHeight) {
            mFullRedrawNeeded = true;
            mLayoutRequested = true;
            windowSizeMayChange = true;
        }
    }

    if (viewVisibilityChanged) {
        mAttachInfo.mWindowVisibility = viewVisibility;
        host.dispatchWindowVisibilityChanged(viewVisibility);
        if (viewUserVisibilityChanged) {
            host.dispatchVisibilityAggregated(viewVisibility == View.VISIBLE);
        }
        if (viewVisibility != View.VISIBLE || mNewSurfaceNeeded) {
            endDragResizing();
            destroyHardwareResources();
        }
    }

    // Non-visible windows can't hold accessibility focus.
    if (mAttachInfo.mWindowVisibility != View.VISIBLE) {
        host.clearAccessibilityFocus();
    }

    // Execute enqueued actions on every traversal in case a detached view enqueued an action
    getRunQueue().executeActions(mAttachInfo.mHandler);

    boolean layoutRequested = mLayoutRequested && (!mStopped || mReportNextDraw);
    if (layoutRequested) {

        final Resources res = mView.getContext().getResources();

        if (mFirst) {
            // make sure touch mode code executes by setting cached value
            // to opposite of the added touch mode.
            mAttachInfo.mInTouchMode = !mAddedTouchMode;
            ensureTouchModeLocally(mAddedTouchMode);
        } else {
            if (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT
                    || lp.height == ViewGroup.LayoutParams.WRAP_CONTENT) {
                windowSizeMayChange = true;

                if (shouldUseDisplaySize(lp)) {
                    // NOTE -- system code, won't try to do compat mode.
                    Point size = new Point();
                    mDisplay.getRealSize(size);
                    desiredWindowWidth = size.x;
                    desiredWindowHeight = size.y;
                } else {
                    final Rect bounds = getWindowBoundsInsetSystemBars();
                    desiredWindowWidth = bounds.width();
                    desiredWindowHeight = bounds.height();
                }
            }
        }

        // Ask host how big it wants to be
        windowSizeMayChange |= measureHierarchy(host, lp, res,
                desiredWindowWidth, desiredWindowHeight);
    }

    if (collectViewAttributes()) {
        params = lp;
    }
    if (mAttachInfo.mForceReportNewAttributes) {
        mAttachInfo.mForceReportNewAttributes = false;
        params = lp;
    }

    if (mFirst || mAttachInfo.mViewVisibilityChanged) {
        mAttachInfo.mViewVisibilityChanged = false;
        int resizeMode = mSoftInputMode & SOFT_INPUT_MASK_ADJUST;
        // If we are in auto resize mode, then we need to determine
        // what mode to use now.
        if (resizeMode == WindowManager.LayoutParams.SOFT_INPUT_ADJUST_UNSPECIFIED) {
            final int N = mAttachInfo.mScrollContainers.size();
            for (int i=0; i<N; i++) {
                if (mAttachInfo.mScrollContainers.get(i).isShown()) {
                    resizeMode = WindowManager.LayoutParams.SOFT_INPUT_ADJUST_RESIZE;
                }
            }
            if (resizeMode == 0) {
                resizeMode = WindowManager.LayoutParams.SOFT_INPUT_ADJUST_PAN;
            }
            if ((lp.softInputMode & SOFT_INPUT_MASK_ADJUST) != resizeMode) {
                lp.softInputMode = (lp.softInputMode & ~SOFT_INPUT_MASK_ADJUST) | resizeMode;
                params = lp;
            }
        }
    }

    if (mApplyInsetsRequested && !(mWillMove || mWillResize)) {
        dispatchApplyInsets(host);
        if (mLayoutRequested) {
            windowSizeMayChange |= measureHierarchy(host, lp,
                    mView.getContext().getResources(),
                    desiredWindowWidth, desiredWindowHeight);
        }
    }

    if (layoutRequested) {
        // Clear this now, so that if anything requests a layout in the
        // rest of this function we will catch it and re-run a full
        // layout pass.
        mLayoutRequested = false;
    }

    boolean windowShouldResize = layoutRequested && windowSizeMayChange
        && ((mWidth != host.getMeasuredWidth() || mHeight != host.getMeasuredHeight())
            || (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT &&
                    frame.width() < desiredWindowWidth && frame.width() != mWidth)
            || (lp.height == ViewGroup.LayoutParams.WRAP_CONTENT &&
                    frame.height() < desiredWindowHeight && frame.height() != mHeight));
    windowShouldResize |= mDragResizing && mResizeMode == RESIZE_MODE_FREEFORM;

    windowShouldResize |= mActivityRelaunched;

    final boolean computesInternalInsets =
            mAttachInfo.mTreeObserver.hasComputeInternalInsetsListeners()
            || mAttachInfo.mHasNonEmptyGivenInternalInsets;

    boolean insetsPending = false;
    int relayoutResult = 0;
    boolean updatedConfiguration = false;

    final int surfaceGenerationId = mSurface.getGenerationId();

    final boolean isViewVisible = viewVisibility == View.VISIBLE;
    final boolean windowRelayoutWasForced = mForceNextWindowRelayout;
    boolean surfaceSizeChanged = false;
    boolean surfaceCreated = false;
    boolean surfaceDestroyed = false;
    // True if surface generation id changes or relayout result is RELAYOUT_RES_SURFACE_CHANGED.
    boolean surfaceReplaced = false;

    final boolean windowAttributesChanged = mWindowAttributesChanged;
    if (windowAttributesChanged) {
        mWindowAttributesChanged = false;
        params = lp;
    }

    if (params != null) {
        if ((host.mPrivateFlags & View.PFLAG_REQUEST_TRANSPARENT_REGIONS) != 0
                && !PixelFormat.formatHasAlpha(params.format)) {
            params.format = PixelFormat.TRANSLUCENT;
        }
        adjustLayoutParamsForCompatibility(params);
        controlInsetsForCompatibility(params);
        if (mDispatchedSystemBarAppearance != params.insetsFlags.appearance) {
            mDispatchedSystemBarAppearance = params.insetsFlags.appearance;
            mView.onSystemBarAppearanceChanged(mDispatchedSystemBarAppearance);
        }
    }
    final boolean wasReportNextDraw = mReportNextDraw;

    if (mFirst || windowShouldResize || viewVisibilityChanged || params != null
            || mForceNextWindowRelayout) {
        mForceNextWindowRelayout = false;

        insetsPending = computesInternalInsets;

        if (mSurfaceHolder != null) {
            mSurfaceHolder.mSurfaceLock.lock();
            mDrawingAllowed = true;
        }

        boolean hwInitialized = false;
        boolean dispatchApplyInsets = false;
        boolean hadSurface = mSurface.isValid();

        try {

            if (mAttachInfo.mThreadedRenderer != null) {
                // relayoutWindow may decide to destroy mSurface. As that decision
                // happens in WindowManager service, we need to be defensive here
                // and stop using the surface in case it gets destroyed.
                if (mAttachInfo.mThreadedRenderer.pause()) {
                    // Animations were running so we need to push a frame
                    // to resume them
                    mDirty.set(0, 0, mWidth, mHeight);
                }
            }
            if (mFirst || viewVisibilityChanged) {
                mViewFrameInfo.flags |= FrameInfo.FLAG_WINDOW_VISIBILITY_CHANGED;
            }
            relayoutResult = relayoutWindow(params, viewVisibility, insetsPending);
            final boolean freeformResizing = (relayoutResult
                    & WindowManagerGlobal.RELAYOUT_RES_DRAG_RESIZING_FREEFORM) != 0;
            final boolean dockedResizing = (relayoutResult
                    & WindowManagerGlobal.RELAYOUT_RES_DRAG_RESIZING_DOCKED) != 0;
            final boolean dragResizing = freeformResizing || dockedResizing;
            if ((relayoutResult & WindowManagerGlobal.RELAYOUT_RES_BLAST_SYNC) != 0) {
                reportNextDraw();
                if (isHardwareEnabled()) {
                    mNextDrawUseBlastSync = true;
                }
            }

            final boolean surfaceControlChanged =
                    (relayoutResult & RELAYOUT_RES_SURFACE_CHANGED)
                            == RELAYOUT_RES_SURFACE_CHANGED;

            if (mSurfaceControl.isValid()) {
                updateOpacity(mWindowAttributes, dragResizing,
                        surfaceControlChanged /*forceUpdate */);
            }

            if (!mPendingMergedConfiguration.equals(mLastReportedMergedConfiguration)) {
                performConfigurationChange(new MergedConfiguration(mPendingMergedConfiguration),
                        !mFirst, INVALID_DISPLAY /* same display */);
                updatedConfiguration = true;
            }

            surfaceSizeChanged = false;
            if (!mLastSurfaceSize.equals(mSurfaceSize)) {
                surfaceSizeChanged = true;
                mLastSurfaceSize.set(mSurfaceSize.x, mSurfaceSize.y);
            }
            final boolean alwaysConsumeSystemBarsChanged =
                    mPendingAlwaysConsumeSystemBars != mAttachInfo.mAlwaysConsumeSystemBars;
            updateColorModeIfNeeded(lp.getColorMode());
            surfaceCreated = !hadSurface && mSurface.isValid();
            surfaceDestroyed = hadSurface && !mSurface.isValid();

            surfaceReplaced = (surfaceGenerationId != mSurface.getGenerationId()
                    || surfaceControlChanged) && mSurface.isValid();
            if (surfaceReplaced) {
                mSurfaceSequenceId++;
            }

            if (alwaysConsumeSystemBarsChanged) {
                mAttachInfo.mAlwaysConsumeSystemBars = mPendingAlwaysConsumeSystemBars;
                dispatchApplyInsets = true;
            }
            if (updateCaptionInsets()) {
                dispatchApplyInsets = true;
            }
            if (dispatchApplyInsets || mLastSystemUiVisibility !=
                    mAttachInfo.mSystemUiVisibility || mApplyInsetsRequested) {
                mLastSystemUiVisibility = mAttachInfo.mSystemUiVisibility;
                dispatchApplyInsets(host);
                // We applied insets so force contentInsetsChanged to ensure the
                // hierarchy is measured below.
                dispatchApplyInsets = true;
            }

            if (surfaceCreated) {
                // If we are creating a new surface, then we need to
                // completely redraw it.
                mFullRedrawNeeded = true;
                mPreviousTransparentRegion.setEmpty();

                // Only initialize up-front if transparent regions are not
                // requested, otherwise defer to see if the entire window
                // will be transparent
                if (mAttachInfo.mThreadedRenderer != null) {
                    try {
                        hwInitialized = mAttachInfo.mThreadedRenderer.initialize(mSurface);
                        if (hwInitialized && (host.mPrivateFlags
                                        & View.PFLAG_REQUEST_TRANSPARENT_REGIONS) == 0) {
                            // Don't pre-allocate if transparent regions
                            // are requested as they may not be needed
                            mAttachInfo.mThreadedRenderer.allocateBuffers();
                        }
                    } catch (OutOfResourcesException e) {
                        handleOutOfResourcesException(e);
                        return;
                    }
                }
            } else if (surfaceDestroyed) {
                // If the surface has been removed, then reset the scroll
                // positions.
                if (mLastScrolledFocus != null) {
                    mLastScrolledFocus.clear();
                }
                mScrollY = mCurScrollY = 0;
                if (mView instanceof RootViewSurfaceTaker) {
                    ((RootViewSurfaceTaker) mView).onRootViewScrollYChanged(mCurScrollY);
                }
                if (mScroller != null) {
                    mScroller.abortAnimation();
                }
                // Our surface is gone
                if (isHardwareEnabled()) {
                    mAttachInfo.mThreadedRenderer.destroy();
                }
            } else if ((surfaceReplaced
                    || surfaceSizeChanged || windowRelayoutWasForced)
                    && mSurfaceHolder == null
                    && mAttachInfo.mThreadedRenderer != null
                    && mSurface.isValid()) {
                mFullRedrawNeeded = true;
                try {
                    mAttachInfo.mThreadedRenderer.updateSurface(mSurface);
                } catch (OutOfResourcesException e) {
                    handleOutOfResourcesException(e);
                    return;
                }
            }

            if (mDragResizing != dragResizing) {
                if (dragResizing) {
                    mResizeMode = freeformResizing
                            ? RESIZE_MODE_FREEFORM
                            : RESIZE_MODE_DOCKED_DIVIDER;
                    final boolean backdropSizeMatchesFrame =
                            mWinFrame.width() == mPendingBackDropFrame.width()
                                    && mWinFrame.height() == mPendingBackDropFrame.height();
                    // TODO: Need cutout?
                    startDragResizing(mPendingBackDropFrame, !backdropSizeMatchesFrame,
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets, mResizeMode);
                } else {
                    // We shouldn't come here, but if we come we should end the resize.
                    endDragResizing();
                }
            }
            if (!mUseMTRenderer) {
                if (dragResizing) {
                    mCanvasOffsetX = mWinFrame.left;
                    mCanvasOffsetY = mWinFrame.top;
                } else {
                    mCanvasOffsetX = mCanvasOffsetY = 0;
                }
            }
        } catch (RemoteException e) {
        }

        mAttachInfo.mWindowLeft = frame.left;
        mAttachInfo.mWindowTop = frame.top;

        // !!FIXME!! This next section handles the case where we did not get the
        // window size we asked for. We should avoid this by getting a maximum size from
        // the window session beforehand.
        if (mWidth != frame.width() || mHeight != frame.height()) {
            mWidth = frame.width();
            mHeight = frame.height();
        }

        if (mSurfaceHolder != null) {
            // The app owns the surface; tell it about what is going on.
            if (mSurface.isValid()) {
                // XXX .copyFrom() doesn't work!
                //mSurfaceHolder.mSurface.copyFrom(mSurface);
                mSurfaceHolder.mSurface = mSurface;
            }
            mSurfaceHolder.setSurfaceFrameSize(mWidth, mHeight);
            mSurfaceHolder.mSurfaceLock.unlock();
            if (surfaceCreated) {
                mSurfaceHolder.ungetCallbacks();

                mIsCreating = true;
                SurfaceHolder.Callback[] callbacks = mSurfaceHolder.getCallbacks();
                if (callbacks != null) {
                    for (SurfaceHolder.Callback c : callbacks) {
                        c.surfaceCreated(mSurfaceHolder);
                    }
                }
            }

            if ((surfaceCreated || surfaceReplaced || surfaceSizeChanged
                    || windowAttributesChanged) && mSurface.isValid()) {
                SurfaceHolder.Callback[] callbacks = mSurfaceHolder.getCallbacks();
                if (callbacks != null) {
                    for (SurfaceHolder.Callback c : callbacks) {
                        c.surfaceChanged(mSurfaceHolder, lp.format,
                                mWidth, mHeight);
                    }
                }
                mIsCreating = false;
            }

            if (surfaceDestroyed) {
                notifyHolderSurfaceDestroyed();
                mSurfaceHolder.mSurfaceLock.lock();
                try {
                    mSurfaceHolder.mSurface = new Surface();
                } finally {
                    mSurfaceHolder.mSurfaceLock.unlock();
                }
            }
        }

        final ThreadedRenderer threadedRenderer = mAttachInfo.mThreadedRenderer;
        if (threadedRenderer != null && threadedRenderer.isEnabled()) {
            if (hwInitialized
                    || mWidth != threadedRenderer.getWidth()
                    || mHeight != threadedRenderer.getHeight()
                    || mNeedsRendererSetup) {
                threadedRenderer.setup(mWidth, mHeight, mAttachInfo,
                        mWindowAttributes.surfaceInsets);
                mNeedsRendererSetup = false;
            }
        }

        if (!mStopped || wasReportNextDraw) {
            boolean focusChangedDueToTouchMode = ensureTouchModeLocally(
                    (relayoutResult&WindowManagerGlobal.RELAYOUT_RES_IN_TOUCH_MODE) != 0);
            if (focusChangedDueToTouchMode || mWidth != host.getMeasuredWidth()
                    || mHeight != host.getMeasuredHeight() || dispatchApplyInsets ||
                    updatedConfiguration) {
                int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
                int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);


                 // Ask host how big it wants to be
                //执行View.measure
                performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);

                int width = host.getMeasuredWidth();
                int height = host.getMeasuredHeight();
                boolean measureAgain = false;

                if (lp.horizontalWeight > 0.0f) {
                    width += (int) ((mWidth - width) * lp.horizontalWeight);
                    childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(width,
                            MeasureSpec.EXACTLY);
                    measureAgain = true;
                }
                if (lp.verticalWeight > 0.0f) {
                    height += (int) ((mHeight - height) * lp.verticalWeight);
                    childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(height,
                            MeasureSpec.EXACTLY);
                    measureAgain = true;
                }

                if (measureAgain) {
                    //再次测量
                    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
                }

                layoutRequested = true;
            }
        }
    } else {
        // Not the first pass and no window/insets/visibility change but the window
        // may have moved and we need check that and if so to update the left and right
        // in the attach info. We translate only the window frame since on window move
        // the window manager tells us only for the new frame but the insets are the
        // same and we do not want to translate them more than once.
        maybeHandleWindowMove(frame);
    }

    if (surfaceSizeChanged || surfaceReplaced || surfaceCreated || windowAttributesChanged) {
        prepareSurfaces();
    }

    final boolean didLayout = layoutRequested && (!mStopped || wasReportNextDraw);
    boolean triggerGlobalLayoutListener = didLayout
            || mAttachInfo.mRecomputeGlobalAttributes;
    if (didLayout) {
        //执行View.layout
        performLayout(lp, mWidth, mHeight);

        if ((host.mPrivateFlags & View.PFLAG_REQUEST_TRANSPARENT_REGIONS) != 0) {
            // start out transparent
            host.getLocationInWindow(mTmpLocation);
            mTransparentRegion.set(mTmpLocation[0], mTmpLocation[1],
                    mTmpLocation[0] + host.mRight - host.mLeft,
                    mTmpLocation[1] + host.mBottom - host.mTop);

            host.gatherTransparentRegion(mTransparentRegion);
            if (mTranslator != null) {
                mTranslator.translateRegionInWindowToScreen(mTransparentRegion);
            }

            if (!mTransparentRegion.equals(mPreviousTransparentRegion)) {
                mPreviousTransparentRegion.set(mTransparentRegion);
                mFullRedrawNeeded = true;

                SurfaceControl sc = getSurfaceControl();
                if (sc.isValid()) {
                    mTransaction.setTransparentRegionHint(sc, mTransparentRegion).apply();
                }
            }
        }
    }

    // These callbacks will trigger SurfaceView SurfaceHolder.Callbacks and must be invoked
    // after the measure pass. If its invoked before the measure pass and the app modifies
    // the view hierarchy in the callbacks, we could leave the views in a broken state.
    if (surfaceCreated) {
        notifySurfaceCreated();
    } else if (surfaceReplaced) {
        notifySurfaceReplaced();
    } else if (surfaceDestroyed)  {
        notifySurfaceDestroyed();
    }

    if (triggerGlobalLayoutListener) {
        mAttachInfo.mRecomputeGlobalAttributes = false;
        mAttachInfo.mTreeObserver.dispatchOnGlobalLayout();
    }

    if (computesInternalInsets) {
        // Clear the original insets.
        final ViewTreeObserver.InternalInsetsInfo insets = mAttachInfo.mGivenInternalInsets;
        insets.reset();

        // Compute new insets in place.
        mAttachInfo.mTreeObserver.dispatchOnComputeInternalInsets(insets);
        mAttachInfo.mHasNonEmptyGivenInternalInsets = !insets.isEmpty();

        // Tell the window manager.
        if (insetsPending || !mLastGivenInsets.equals(insets)) {
            mLastGivenInsets.set(insets);

            // Translate insets to screen coordinates if needed.
            final Rect contentInsets;
            final Rect visibleInsets;
            final Region touchableRegion;
            if (mTranslator != null) {
                contentInsets = mTranslator.getTranslatedContentInsets(insets.contentInsets);
                visibleInsets = mTranslator.getTranslatedVisibleInsets(insets.visibleInsets);
                touchableRegion = mTranslator.getTranslatedTouchableArea(insets.touchableRegion);
            } else {
                contentInsets = insets.contentInsets;
                visibleInsets = insets.visibleInsets;
                touchableRegion = insets.touchableRegion;
            }

            try {
                mWindowSession.setInsets(mWindow, insets.mTouchableInsets,
                        contentInsets, visibleInsets, touchableRegion);
            } catch (RemoteException e) {
            }
        }
    }

    if (mFirst) {
        if (sAlwaysAssignFocus || !isInTouchMode()) {
            // handle first focus request
            if (mView != null) {
                if (!mView.hasFocus()) {
                    mView.restoreDefaultFocus();
                } else {

                }
            }
        } else {
            View focused = mView.findFocus();
            if (focused instanceof ViewGroup
                    && ((ViewGroup) focused).getDescendantFocusability()
                            == ViewGroup.FOCUS_AFTER_DESCENDANTS) {
                focused.restoreDefaultFocus();
            }
        }
    }

    final boolean changedVisibility = (viewVisibilityChanged || mFirst) && isViewVisible;
    final boolean hasWindowFocus = mAttachInfo.mHasWindowFocus && isViewVisible;
    final boolean regainedFocus = hasWindowFocus && mLostWindowFocus;
    if (regainedFocus) {
        mLostWindowFocus = false;
    } else if (!hasWindowFocus && mHadWindowFocus) {
        mLostWindowFocus = true;
    }

    if (changedVisibility || regainedFocus) {
        // Toasts are presented as notifications - don't present them as windows as well
        boolean isToast = mWindowAttributes.type == TYPE_TOAST;
        if (!isToast) {
            host.sendAccessibilityEvent(AccessibilityEvent.TYPE_WINDOW_STATE_CHANGED);
        }
    }

    mFirst = false;
    mWillDrawSoon = false;
    mNewSurfaceNeeded = false;
    mActivityRelaunched = false;
    mViewVisibility = viewVisibility;
    mHadWindowFocus = hasWindowFocus;

    mImeFocusController.onTraversal(hasWindowFocus, mWindowAttributes);

    // Remember if we must report the next draw.
    if ((relayoutResult & WindowManagerGlobal.RELAYOUT_RES_FIRST_TIME) != 0) {
        reportNextDraw();
    }

    boolean cancelDraw = mAttachInfo.mTreeObserver.dispatchOnPreDraw() || !isViewVisible;
    if (mBLASTDrawConsumer != null) {
        mNextDrawUseBlastSync = true;
    }

    if (!cancelDraw) {
        if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
            for (int i = 0; i < mPendingTransitions.size(); ++i) {
                mPendingTransitions.get(i).startChangingAnimations();
            }
            mPendingTransitions.clear();
        }
        //执行View.draw
        performDraw();
    } else {
        if (isViewVisible) {
            // Try again
            scheduleTraversals();
        } else {
            if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
                for (int i = 0; i < mPendingTransitions.size(); ++i) {
                    mPendingTransitions.get(i).endChangingAnimations();
                }
                mPendingTransitions.clear();
            }

            // We may never draw since it's not visible. Report back that we're finished
            // drawing.
            if (!wasReportNextDraw && mReportNextDraw) {
                mReportNextDraw = false;
                pendingDrawFinished();
            }
        }
    }

    if (mAttachInfo.mContentCaptureEvents != null) {
        notifyContentCatpureEvents();
    }

    mIsInTraversal = false;
    
}
```
