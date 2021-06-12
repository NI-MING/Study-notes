[TOC]

# View绘制流程一：布局是如何添加到界面上的

当我们打开一个App，便会看到一个个精美的页面，作为开发者的我们，有没有想过这些页面是如何添加到我们的界面上的？本片文章主要解决以下几个问题，如果你对以下问题都有明确的答案，可以不必浪费时间了。

1. 什么是DecorView，与我们编写的布局文件的关系。
2. DecorView何时被创建，如何被加载进布局。
3. 什么是ViewRootImpl，与DecorView的关系
4. DecorView如何实现与ViewRootImpl的关联
5. View的绘制流程在何时被触发
6. 为什么常说子线程不能更新UI

先放一张我总结的View绘制流程函数调用栈：

![view](C:\Users\wlk\Desktop\view.png)

为了不放过一丝细节，我将View是如何走到绘制的流程中的每个方法进行了梳理，画出来的函数调用情况也着实令我一惊，下面我将具体讲解这幅图。

## 1.切入点：setContentView

这个方法想必大家不陌生，我们在Activity的onCreate方法中，调用这个方法，传入我们写好的布局，就会在我们的页面中显示，所以，在毫无头绪的情况下，这个方法是我们分析的切入点。

setContentView：

```java
public void setContentView(@LayoutRes int layoutResID) {
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
}
```

我们可以看出，这个方法调用流程是`getWindow().setContentView()`，`getWindow()`是什么？

```java
public Window getWindow() {
    return mWindow;
}
```

返回了`mwindow`，`mwindow`是一个Window，而Window是一个抽象类，只有一个具体实现类，那就是`PhoneWindow`，于是，我们就找到了`PhoneWindow`中`setContentView`的实现：

```java
@Override
public void setContentView(int layoutResID) {
    // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
    // decor, when theme attributes and the like are crystalized. Do not check the feature
    // before this happens.
    if (mContentParent == null) {
        installDecor();
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        mContentParent.removeAllViews();
    }

    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                getContext());
        transitionTo(newScene);
    } else {
        mLayoutInflater.inflate(layoutResID, mContentParent);
    }
    mContentParent.requestApplyInsets();
    final Callback cb = getCallback();
    if (cb != null && !isDestroyed()) {
        cb.onContentChanged();
    }
    mContentParentExplicitlySet = true;
}
```

很明显，这里判断`mContentParent`为null，就进入了`installDecor()`这个方法，看这个方法的名字，就知道`DecorView`在此处创建：

```java
private void installDecor() {
    mForceDecorInstall = false;
    if (mDecor == null) {
        // 重点1
        mDecor = generateDecor(-1);
        mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
        mDecor.setIsRootNamespace(true);
        if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
            mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
        }
    } else {
        mDecor.setWindow(this);
    }
    if (mContentParent == null) {
        // 重点2
        mContentParent = generateLayout(mDecor);

        // Set up decor part of UI to ignore fitsSystemWindows if appropriate.
        mDecor.makeFrameworkOptionalFitsSystemWindows();

        final DecorContentParent decorContentParent = (DecorContentParent) mDecor.findViewById(
                R.id.decor_content_parent);

        if (decorContentParent != null) {
            mDecorContentParent = decorContentParent;
            mDecorContentParent.setWindowCallback(getCallback());
            if (mDecorContentParent.getTitle() == null) {
                mDecorContentParent.setWindowTitle(mTitle);
            }

            final int localFeatures = getLocalFeatures();
            for (int i = 0; i < FEATURE_MAX; i++) {
                if ((localFeatures & (1 << i)) != 0) {
                    mDecorContentParent.initFeature(i);
                }
            }

            mDecorContentParent.setUiOptions(mUiOptions);

            if ((mResourcesSetFlags & FLAG_RESOURCE_SET_ICON) != 0 ||
                    (mIconRes != 0 && !mDecorContentParent.hasIcon())) {
                mDecorContentParent.setIcon(mIconRes);
            } else if ((mResourcesSetFlags & FLAG_RESOURCE_SET_ICON) == 0 &&
                    mIconRes == 0 && !mDecorContentParent.hasIcon()) {
                mDecorContentParent.setIcon(
                        getContext().getPackageManager().getDefaultActivityIcon());
                mResourcesSetFlags |= FLAG_RESOURCE_SET_ICON_FALLBACK;
            }
            if ((mResourcesSetFlags & FLAG_RESOURCE_SET_LOGO) != 0 ||
                    (mLogoRes != 0 && !mDecorContentParent.hasLogo())) {
                mDecorContentParent.setLogo(mLogoRes);
            }

            // Invalidate if the panel menu hasn't been created before this.
            // Panel menu invalidation is deferred avoiding application onCreateOptionsMenu
            // being called in the middle of onCreate or similar.
            // A pending invalidation will typically be resolved before the posted message
            // would run normally in order to satisfy instance state restoration.
            PanelFeatureState st = getPanelState(FEATURE_OPTIONS_PANEL, false);
            if (!isDestroyed() && (st == null || st.menu == null) && !mIsStartingWindow) {
                invalidatePanelMenu(FEATURE_ACTION_BAR);
            }
        } else {
            mTitleView = findViewById(R.id.title);
            if (mTitleView != null) {
                if ((getLocalFeatures() & (1 << FEATURE_NO_TITLE)) != 0) {
                    final View titleContainer = findViewById(R.id.title_container);
                    if (titleContainer != null) {
                        titleContainer.setVisibility(View.GONE);
                    } else {
                        mTitleView.setVisibility(View.GONE);
                    }
                    mContentParent.setForeground(null);
                } else {
                    mTitleView.setText(mTitle);
                }
            }
        }

        if (mDecor.getBackground() == null && mBackgroundFallbackDrawable != null) {
            mDecor.setBackgroundFallback(mBackgroundFallbackDrawable);
        }

        // Only inflate or create a new TransitionManager if the caller hasn't
        // already set a custom one.
        if (hasFeature(FEATURE_ACTIVITY_TRANSITIONS)) {
            if (mTransitionManager == null) {
                final int transitionRes = getWindowStyle().getResourceId(
                        R.styleable.Window_windowContentTransitionManager,
                        0);
                if (transitionRes != 0) {
                    final TransitionInflater inflater = TransitionInflater.from(getContext());
                    mTransitionManager = inflater.inflateTransitionManager(transitionRes,
                            mContentParent);
                } else {
                    mTransitionManager = new TransitionManager();
                }
            }

            mEnterTransition = getTransition(mEnterTransition, null,
                    R.styleable.Window_windowEnterTransition);
            mReturnTransition = getTransition(mReturnTransition, USE_DEFAULT_TRANSITION,
                    R.styleable.Window_windowReturnTransition);
            mExitTransition = getTransition(mExitTransition, null,
                    R.styleable.Window_windowExitTransition);
            mReenterTransition = getTransition(mReenterTransition, USE_DEFAULT_TRANSITION,
                    R.styleable.Window_windowReenterTransition);
            mSharedElementEnterTransition = getTransition(mSharedElementEnterTransition, null,
                    R.styleable.Window_windowSharedElementEnterTransition);
            mSharedElementReturnTransition = getTransition(mSharedElementReturnTransition,
                    USE_DEFAULT_TRANSITION,
                    R.styleable.Window_windowSharedElementReturnTransition);
            mSharedElementExitTransition = getTransition(mSharedElementExitTransition, null,
                    R.styleable.Window_windowSharedElementExitTransition);
            mSharedElementReenterTransition = getTransition(mSharedElementReenterTransition,
                    USE_DEFAULT_TRANSITION,
                    R.styleable.Window_windowSharedElementReenterTransition);
            if (mAllowEnterTransitionOverlap == null) {
                mAllowEnterTransitionOverlap = getWindowStyle().getBoolean(
                        R.styleable.Window_windowAllowEnterTransitionOverlap, true);
            }
            if (mAllowReturnTransitionOverlap == null) {
                mAllowReturnTransitionOverlap = getWindowStyle().getBoolean(
                        R.styleable.Window_windowAllowReturnTransitionOverlap, true);
            }
            if (mBackgroundFadeDurationMillis < 0) {
                mBackgroundFadeDurationMillis = getWindowStyle().getInteger(
                        R.styleable.Window_windowTransitionBackgroundFadeDuration,
                        DEFAULT_BACKGROUND_FADE_DURATION_MS);
            }
            if (mSharedElementsUseOverlay == null) {
                mSharedElementsUseOverlay = getWindowStyle().getBoolean(
                        R.styleable.Window_windowSharedElementsUseOverlay, true);
            }
        }
    }
}
```

在重点1，生成一个`DecorView`：

```java
protected DecorView generateDecor(int featureId) {
    // System process doesn't have application context and in that case we need to directly use
    // the context we have. Otherwise we want the application context, so we don't cling to the
    // activity.
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
    return new DecorView(context, featureId, this, getAttributes());
}
```

`DecorView`继承自`FrameLayout`，创建完`DecorView`之后，就调用了`generateLayout(mDecor)`，去生成布局：

```java
protected ViewGroup generateLayout(DecorView decor) {
    
	......
    // Inflate the window decor.

    int layoutResource;
    int features = getLocalFeatures();

    ......
    // 加载布局
    mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);

    ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
    if (contentParent == null) {
        throw new RuntimeException("Window couldn't find content container view");
    }

    ......

    return contentParent;
}
```

这段代码很清晰了，`mDecor.onResourcesLoaded(mLayoutInflater, layoutResource)`使用`Inflater`去加载`layoutResource`，这个资源放在framework下，是一个`LinearLayout`布局的文件，其中包含`ActionBar`和`ContentView`，这个`ContentView`的id是`@android:id/content`，也是`FrameLayout`布局，在这段代码中也是通过`findViewById`绑定了这个View作为`contentParent`使用。

`installDecor()`方法介绍完之后，接着便有这样一行代码`mLayoutInflater.inflate(layoutResID, mContentParent)`，第一个参数就是我们传入的自己的布局文件，也是我们最初在调用`setContentView`时传入的布局，`mContentParent`不用我多说，就是刚刚`DecorView`中的`Content`，这样，`DecorView`就作为我们的根布局，加载进了我们的界面，同时还加载了我们自己编写的~~精美的~~界面。一切都顺理成章，只剩两个问题，`PhoneWindow`从何而来？打开App怎么就回调了`setContentView`方法？

## 2.PhoneWindow从何而来

`PhoneWindow`从何而来不着急，当务之急是先找到`setContentView`是如何被回调的，否则以上的分析都是镜中月、水中花。`setContentView`是在我们回调`onCreate`不久之后就被调用了，所以找到`onCreate`方法在哪里被调用就好。对Android有一定了解的同学都知道，Android中有一个很重要的类是ActivityThread，这个类可以说是管理着Android中四大组件的生命周期，Activity中onCreate方法，一定是在这个类里被回调。前面我写过一篇针对Activity的文章，里面有提到Activity的启动流程，谈及Activity中生命周期函数被调用的流程。

具体流程就不再这里分析了，我们直接看`handleLaunchActivity`方法，这个方法就是启动Activity时回调的：

```java
@Override
public Activity handleLaunchActivity(ActivityClientRecord r,
        PendingTransactionActions pendingActions, Intent customIntent) {
    // If we are getting ready to gc after going to the background, well
    // we are back active so skip it.
    unscheduleGcIdler();
    mSomeActivitiesChanged = true;

    if (r.profilerInfo != null) {
        mProfiler.setProfiler(r.profilerInfo);
        mProfiler.startProfiling();
    }

    // Make sure we are running with the most recent config.
    handleConfigurationChanged(null, null);

    if (localLOGV) Slog.v(
        TAG, "Handling launch of " + r);

    // Initialize before creating the activity
    if (!ThreadedRenderer.sRendererDisabled
            && (r.activityInfo.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0) {
        HardwareRenderer.preload();
    }
    WindowManagerGlobal.initialize();

    // Hint the GraphicsEnvironment that an activity is launching on the process.
    GraphicsEnvironment.hintActivityLaunch();

    final Activity a = performLaunchActivity(r, customIntent);

    if (a != null) {
        r.createdConfig = new Configuration(mConfiguration);
        reportSizeConfigurations(r);
        if (!r.activity.mFinished && pendingActions != null) {
            pendingActions.setOldState(r.state);
            pendingActions.setRestoreInstanceState(true);
            pendingActions.setCallOnPostCreate(true);
        }
    } else {
        // If there was an error, for any reason, tell the activity manager to stop us.
        try {
            ActivityTaskManager.getService()
                    .finishActivity(r.token, Activity.RESULT_CANCELED, null,
                            Activity.DONT_FINISH_TASK_WITH_ACTIVITY);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    }

    return a;
}
```

performLaunchActivity：

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ActivityInfo aInfo = r.activityInfo;
    if (r.packageInfo == null) {
        r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                Context.CONTEXT_INCLUDE_CODE);
    }

    ComponentName component = r.intent.getComponent();
    if (component == null) {
        component = r.intent.resolveActivity(
            mInitialApplication.getPackageManager());
        r.intent.setComponent(component);
    }

    if (r.activityInfo.targetActivity != null) {
        component = new ComponentName(r.activityInfo.packageName,
                r.activityInfo.targetActivity);
    }

    ContextImpl appContext = createBaseContextForActivity(r);
    Activity activity = null;
    try {
        java.lang.ClassLoader cl = appContext.getClassLoader();
        activity = mInstrumentation.newActivity(
                cl, component.getClassName(), r.intent);
        StrictMode.incrementExpectedActivityCount(activity.getClass());
        r.intent.setExtrasClassLoader(cl);
        r.intent.prepareToEnterProcess();
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
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);

        if (localLOGV) Slog.v(TAG, "Performing launch of " + r);
        if (localLOGV) Slog.v(
                TAG, r + ": app=" + app
                + ", appName=" + app.getPackageName()
                + ", pkg=" + r.packageInfo.getPackageName()
                + ", comp=" + r.intent.getComponent().toShortString()
                + ", dir=" + r.packageInfo.getAppDir());

        if (activity != null) {
            CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
            Configuration config = new Configuration(mCompatConfiguration);
            if (r.overrideConfig != null) {
                config.updateFrom(r.overrideConfig);
            }
            if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "
                    + r.activityInfo.name + " with config " + config);
            Window window = null;
            if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
                window = r.mPendingRemoveWindow;
                r.mPendingRemoveWindow = null;
                r.mPendingRemoveWindowManager = null;
            }

            // Activity resources must be initialized with the same loaders as the
            // application context.
            appContext.getResources().addLoaders(
                    app.getResources().getLoaders().toArray(new ResourcesLoader[0]));

            appContext.setOuterContext(activity);
            // 重点1
            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor, window, r.configCallback,
                    r.assistToken);

            if (customIntent != null) {
                activity.mIntent = customIntent;
            }
            r.lastNonConfigurationInstances = null;
            checkAndBlockForNetworkAccess();
            activity.mStartedActivity = false;
            int theme = r.activityInfo.getThemeResource();
            if (theme != 0) {
                activity.setTheme(theme);
            }

            activity.mCalled = false;
            // 重点2
            if (r.isPersistable()) {
                mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
            } else {
                mInstrumentation.callActivityOnCreate(activity, r.state);
            }
            if (!activity.mCalled) {
                throw new SuperNotCalledException(
                    "Activity " + r.intent.getComponent().toShortString() +
                    " did not call through to super.onCreate()");
            }
            r.activity = activity;
            mLastReportedWindowingMode.put(activity.getActivityToken(),
                    config.windowConfiguration.getWindowingMode());
        }
        r.setState(ON_CREATE);

        // updatePendingActivityConfiguration() reads from mActivities to update
        // ActivityClientRecord which runs in a different thread. Protect modifications to
        // mActivities to avoid race.
        synchronized (mResourcesManager) {
            mActivities.put(r.token, r);
        }

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
```

在1处，我们调用了`activity.attach`方法：

```java
final void attach(Context context, ActivityThread aThread,
        Instrumentation instr, IBinder token, int ident,
        Application application, Intent intent, ActivityInfo info,
        CharSequence title, Activity parent, String id,
        NonConfigurationInstances lastNonConfigurationInstances,
        Configuration config, String referrer, IVoiceInteractor voiceInteractor,
        Window window, ActivityConfigCallback activityConfigCallback, IBinder assistToken) {
    attachBaseContext(context);

    mFragments.attachHost(null /*parent*/);

    mWindow = new PhoneWindow(this, window, activityConfigCallback);
    ......
}
```

进入这个方法不久，我们就把`PhoneWindow`给new出来了。这就解决了我们的一个问题，`PhoneWindow`从何而来。第二个问题onCreate何时回调，我们继续看代码，在重点2处，看到了这样的代码`mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState)`，进去看看：

```java
public void callActivityOnCreate(Activity activity, Bundle icicle,
        PersistableBundle persistentState) {
    prePerformCreate(activity);
    activity.performCreate(icicle, persistentState);
    postPerformCreate(activity);
}
```

回调Activity的`performCreate`，最后会回调我们的onCreate方法。这样我们的setContentView方法也就会被调用。

我们现在已经知道了`PhoneWindow`有内部类`DecorView`，是`FramLayout`布局，`PhoneWindow`在onCreate方法之前被创建好，`DecorView`会通过`inflater`在Framework层绑定一个线性布局作为我们的根布局，我们自己编写的布局会绑定到`content`中。这样View树就形成了，可以进行后续的绘制操作。

## 3.绘制入口：ViewRootImpl

我们的View要进行绘制，就要执行View的三大方法：measure、layout、draw。而`DecorView`是我们的根View，理论上将所有的View的绘制都从它开始。那DecorView的这三大方法又由谁来掉用呢？由ViewRootImpl调用。下面我们从源码验证一下。

当我们的onResume方法调用完成之后，我们的页面就呈现在我们面前，（Google开发者文档中写到哦那Start方法调用使Activity对用户可见，我查看源码并没有发现对View的绘制工作，反而在onResume之后找到了它。）前面提到ActivityThread中管理了Activity的生命周期，我们在其中找到了这样一段代码：

```java
public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
        String reason) {
    // If we are getting ready to gc after going to the background, well
    // we are back active so skip it.
    unscheduleGcIdler();
    mSomeActivitiesChanged = true;

    // TODO Push resumeArgs into the activity for consideration
    // 进入onResume
    final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
    if (r == null) {
        // We didn't actually resume the activity, so skipping any follow-up actions.
        return;
    }
    if (mActivitiesToBeDestroyed.containsKey(token)) {
        // Although the activity is resumed, it is going to be destroyed. So the following
        // UI operations are unnecessary and also prevents exception because its token may
        // be gone that window manager cannot recognize it. All necessary cleanup actions
        // performed below will be done while handling destruction.
        return;
    }

    final Activity a = r.activity;

    if (localLOGV) {
        Slog.v(TAG, "Resume " + r + " started activity: " + a.mStartedActivity
                + ", hideForNow: " + r.hideForNow + ", finished: " + a.mFinished);
    }

    final int forwardBit = isForward
            ? WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION : 0;

    // If the window hasn't yet been added to the window manager,
    // and this guy didn't finish itself or start another activity,
    // then go ahead and add the window.
    boolean willBeVisible = !a.mStartedActivity;
    if (!willBeVisible) {
        try {
            willBeVisible = ActivityTaskManager.getService().willActivityBeVisible(
                    a.getActivityToken());
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
    
    // 重点1
    if (r.window == null && !a.mFinished && willBeVisible) {
        r.window = r.activity.getWindow();
        View decor = r.window.getDecorView();
        decor.setVisibility(View.INVISIBLE);
        ViewManager wm = a.getWindowManager();
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
            if (!a.mWindowAdded) {
                a.mWindowAdded = true;
                wm.addView(decor, l);
            } else {
                // The activity will get a callback for this {@link LayoutParams} change
                // earlier. However, at that time the decor will not be set (this is set
                // in this method), so no action will be taken. This call ensures the
                // callback occurs with the decor set.
                a.onWindowAttributesChanged(l);
            }
        }

        // If the window has already been added, but during resume
        // we started another activity, then don't yet make the
        // window visible.
    } else if (!willBeVisible) {
        if (localLOGV) Slog.v(TAG, "Launch " + r + " mStartedActivity set");
        r.hideForNow = true;
    }

    // Get rid of anything left hanging around.
    cleanUpPendingRemoveWindows(r, false /* force */);

    // The window is now visible if it has been added, we are not
    // simply finishing, and we are not starting another activity.
    if (!r.activity.mFinished && willBeVisible && r.activity.mDecor != null && !r.hideForNow) {
        if (r.newConfig != null) {
            performConfigurationChangedForActivity(r, r.newConfig);
            if (DEBUG_CONFIGURATION) {
                Slog.v(TAG, "Resuming activity " + r.activityInfo.name + " with newConfig "
                        + r.activity.mCurrentConfig);
            }
            r.newConfig = null;
        }
        if (localLOGV) Slog.v(TAG, "Resuming " + r + " with isForward=" + isForward);
        ViewRootImpl impl = r.window.getDecorView().getViewRootImpl();
        WindowManager.LayoutParams l = impl != null
                ? impl.mWindowAttributes : r.window.getAttributes();
        if ((l.softInputMode
                & WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION)
                != forwardBit) {
            l.softInputMode = (l.softInputMode
                    & (~WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION))
                    | forwardBit;
            if (r.activity.mVisibleFromClient) {
                ViewManager wm = a.getWindowManager();
                View decor = r.window.getDecorView();
                wm.updateViewLayout(decor, l);
            }
        }

        r.activity.mVisibleFromServer = true;
        mNumVisibleActivities++;
        if (r.activity.mVisibleFromClient) {
            r.activity.makeVisible();
        }
    }

    r.nextIdle = mNewActivities;
    mNewActivities = r;
    if (localLOGV) Slog.v(TAG, "Scheduling idle handler for " + r);
    Looper.myQueue().addIdleHandler(new Idler());
}
```

我们看到，进入这个方法不久后就调用了onResume，之后就走到了重点1，从`PhoneWindow`中取出`DecorView`，赋值给`decor`，然后创建出`ViewManager`，调用了`ViewManager`的`addView`方法，这个类由`WindowManagerImpl`实现：

```java
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    applyDefaultToken(params);
    mGlobal.addView(view, params, mContext.getDisplayNoVerify(), mParentWindow,
            mContext.getUserId());
}
```

调用`WindowManagerGlobal`的`addView`方法：

```java
public void addView(View view, ViewGroup.LayoutParams params,
        Display display, Window parentWindow, int userId) {
    if (view == null) {
        throw new IllegalArgumentException("view must not be null");
    }
    if (display == null) {
        throw new IllegalArgumentException("display must not be null");
    }
    if (!(params instanceof WindowManager.LayoutParams)) {
        throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
    }

    final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
    if (parentWindow != null) {
        parentWindow.adjustLayoutParamsForSubWindow(wparams);
    } else {
        // If there's no parent, then hardware acceleration for this view is
        // set from the application's hardware acceleration setting.
        final Context context = view.getContext();
        if (context != null
                && (context.getApplicationInfo().flags
                        & ApplicationInfo.FLAG_HARDWARE_ACCELERATED) != 0) {
            wparams.flags |= WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;
        }
    }

    ViewRootImpl root;
    View panelParentView = null;

    synchronized (mLock) {
        // Start watching for system property changes.
        if (mSystemPropertyUpdater == null) {
            mSystemPropertyUpdater = new Runnable() {
                @Override public void run() {
                    synchronized (mLock) {
                        for (int i = mRoots.size() - 1; i >= 0; --i) {
                            mRoots.get(i).loadSystemProperties();
                        }
                    }
                }
            };
            SystemProperties.addChangeCallback(mSystemPropertyUpdater);
        }

        int index = findViewLocked(view, false);
        if (index >= 0) {
            if (mDyingViews.contains(view)) {
                // Don't wait for MSG_DIE to make it's way through root's queue.
                mRoots.get(index).doDie();
            } else {
                throw new IllegalStateException("View " + view
                        + " has already been added to the window manager.");
            }
            // The previous removeView() had not completed executing. Now it has.
        }

        // If this is a panel window, then find the window it is being
        // attached to for future reference.
        if (wparams.type >= WindowManager.LayoutParams.FIRST_SUB_WINDOW &&
                wparams.type <= WindowManager.LayoutParams.LAST_SUB_WINDOW) {
            final int count = mViews.size();
            for (int i = 0; i < count; i++) {
                if (mRoots.get(i).mWindow.asBinder() == wparams.token) {
                    panelParentView = mViews.get(i);
                }
            }
        }
		// 1.
        root = new ViewRootImpl(view.getContext(), display);

        view.setLayoutParams(wparams);

        // 2.
        mViews.add(view);
        mRoots.add(root);
        mParams.add(wparams);

        // do this last because it fires off messages to start doing things
        try {
            // 3.
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

这个类管理这所有的View与ViewRootImpl，我们在2处也可以看出来，它将创建出的DecorView和ViewRootImpl添加到了内部的ArrayList中。在3处，我们调用了ViewRootImpl的`setView`方法：

```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView,
        int userId) {
    synchronized (this) {
        if (mView == null) {
            mView = view;

            ......

            requestLayout();
            
            ......

            view.assignParent(this);
            
			......
        }
    }
}
```

这个方法相当长，我将里面最关键的两段代码留了下来，第一个就是`requestLayout`，第二个是`view.assignParent(this)`，先说第二个，从名字就可以看出来，这个方法将ViewRootImpl设置为DecorView的parent，这样，DecorView的绘制就由ViewRootImpl调用。再来重点看`requestLayout`：

```java
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true;
        scheduleTraversals();
    }
}
```

先经过`checkThread()`：

```java
void checkThread() {
    if (mThread != Thread.currentThread()) {
        throw new CalledFromWrongThreadException(
                "Only the original thread that created a view hierarchy can touch its views.");
    }
}
```

我们发现这个方法跟线程有关，线程？难道常说子线程能更新UI与这个方法有关？的确如此，通常我们都是在主线程调用`WindowManagerGlobal`的`addView`方法，生成的ViewRootImpl中的mThread自然也是主线程，一旦我们在子线程中更改UI，当`requestLayout`走到ViewRootImpl的`requestLayout`时自然报错。

因此，我们只要在程序走到`view.assignParent(this)`之前，DecorView还未将parent设置成ViewRootImpl，自然不会报错了。或者，在子线程中调用`WindowManagerGlobal`的`addView`方法去添加一个View，这时，自然也可以在子线程中更改UI。

接着继续将`scheduleTraversals()`：

```java
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}
```

这里设置了同步屏障消息，它的作用是使下次Looper轮询的时候优先处理异步消息，所以这行代码之后就发送了一个异步消息给Handler（有关Handler的东西我会单独一篇博客来总结）。去处理了`mTraversalRunnable`这个Runnable接口的实现类：

```java
final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        doTraversal();
    }
}
```

doTraversal

```java
void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

        if (mProfile) {
            Debug.startMethodTracing("ViewAncestor");
        }

        performTraversals();

        if (mProfile) {
            Debug.stopMethodTracing();
            mProfile = false;
        }
    }
}
```

在这里移除同步屏障，防止一直阻塞Looper，接着调用`performTraversals()`，这个方法里会调用三大方法：`performMeasure`、`performLayout`、`performDraw`，每个方法里都会调用ViewRootImpl的`measure`、`layout`，`draw`，然后逐层调用完成绘制。

到此我也把最初那张图讲完了，文章开始的六个问题相信你也可以解决了，希望看完这篇文章的你有所收获，同时留下自己宝贵的意见。

# View绘制流程二：测量、布局、绘制

上篇博客我们详细分析了布局是如何添加到我们的界面上的，谈到ViewRootImpl的三个方法也就停止了，今天我们详细的分析measure、layout、draw三个过程。

## Measure

### 理解MeasureSpec

#### 什么是MeasureSpec

当我们要对我们的View进行测量时，要有一定的规则，我们不仅要考虑开发者在xml中给我们指定的layout，而且还要结合父View给我们指定的测量规则，这两个属性共同决定了我们子View的MeasureSpec，也就是子View的测量规则，按照这个MeasureSpec，我们就可以很轻松的解决测量时可能会遇到的问题。

MeasureSpec是一个32位的int值，高2位是SpecMode，它指的是测量模式，低30位是、SpecSize，它指的是在某种测量模式下的规格大小。MeasureSpec是将这两个打包成一个int值来避免过多对象内存分配，在MeasureSpec类内部提供了对这两个属性的操作。

#### MeasureSpec分类

* UNSPECIFIED

  父View不会对View有任何的限制，要多大给多大，这种情况一般用于系统内部。

* EXACTLY

  父View已经测出View所需的精确的大小，这个时候View的最终大小就是SpecSize所指定的值。它对应LayoutParams中的match_parent和具体数值这两种模式。

* AT_MOST

  父容器指定了一个可用大小即SpecSize，View的大小不能大于这个值，具体是什么要看不同View的具体实现。它对应wrap_content属性。

#### MeasureSpec的创建

通过getChildMeasure就可以创建出子View的MeasureSpec：

```java
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);

    int size = Math.max(0, specSize - padding);

    int resultSize = 0;
    int resultMode = 0;

    switch (specMode) {
    // Parent has imposed an exact size on us
    case MeasureSpec.EXACTLY:
        if (childDimension >= 0) {
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size. So be it.
            resultSize = size;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size. It can't be
            // bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;

    // Parent has imposed a maximum size on us
    case MeasureSpec.AT_MOST:
        if (childDimension >= 0) {
            // Child wants a specific size... so be it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size, but our size is not fixed.
            // Constrain child to not be bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size. It can't be
            // bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;

    // Parent asked to see how big we want to be
    case MeasureSpec.UNSPECIFIED:
        if (childDimension >= 0) {
            // Child wants a specific size... let him have it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size... find out how big it should
            // be
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size.... find out how
            // big it should be
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        }
        break;
    }
    //noinspection ResourceType
    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
```

上述代码一图以蔽之：

![u=3712111434,3753346401&fm=26&fmt=auto&gp=0](C:\Users\wlk\Desktop\u=3712111434,3753346401&fm=26&fmt=auto&gp=0.webp)

### View的measure

有了MeasureSpec，我们也可以放心的去测量我们的View了。View的measure过程是由其measure方法来完成，measure方法是一个final方法，子类不可以重写此方法，在View的measure方法中会调用View的onMeasure方法，因此我们直接看onMeasure方法：

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
            getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}
```

这个方法里，只用了一个方法就完成了View的measure过程，而通过方法名我们也知道这个方法时设置测量尺寸，那么参数就是测量过的尺寸了，所以我们看getDefaultSize这个方法：

```java
public static int getDefaultSize(int size, int measureSpec) {
    int result = size;
    int specMode = MeasureSpec.getMode(measureSpec);
    int specSize = MeasureSpec.getSize(measureSpec);

    switch (specMode) {
    case MeasureSpec.UNSPECIFIED:
        result = size;
        break;
    case MeasureSpec.AT_MOST:
    case MeasureSpec.EXACTLY:
        result = specSize;
        break;
    }
    return result;
}
```

这个方法也是很简单，UNSPECIFIED这个分支我们不去管他，而AT_MOST和EXACTLY这两个返回的结果是一样的，都是specSize，我们通过上面的表格也可以看出specSize是parentSize，是父View提供的可用空间。这就给我们自定义View提供了一个思路：继承View需要重写onMeasure并对wrap_content属性设置默认宽高，否则效果同match_parent一样，从源码我们也可以看出原因来。

### ViewGroup的measure

对于ViewGroup来说，除了完成自己的measure过程以外，还要遍历的去调用所有子元素的measure方法，各个子元素再递归去执行这个过程。和View不同的是，ViewGroup是一个抽象类，因此它没有重写View的onMeasure方法，但是它提供了一个叫measureChildren的方法：

```java
protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec) {
    final int size = mChildrenCount;
    final View[] children = mChildren;
    for (int i = 0; i < size; ++i) {
        final View child = children[i];
        if ((child.mViewFlags & VISIBILITY_MASK) != GONE) {
            measureChild(child, widthMeasureSpec, heightMeasureSpec);
        }
    }
}
```

这个方法里，会对每一个子元素去执行ViewGroup的measureChild方法：

```java
protected void measureChild(View child, int parentWidthMeasureSpec,
        int parentHeightMeasureSpec) {
    final LayoutParams lp = child.getLayoutParams();

    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight, lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom, lp.height);

    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```

这个方法里，获取了子View的MeasureSpec，然后对子View进行测量。在继承ViewGroup实现自定义ViewGroup时，测量完子View的宽高后，就可以确定ViewGroup自身的宽高了。

## Layout

Layout的作用是ViewGroup用来确定子元素的位置的，当ViewGroup的位置被确定后，它会在onLayout方法中遍历所有的子元素并调用其layout方法，在layout方法中，onLayout方法又会被调用。layout方法是确定其本身的位置，而onLayout方法是去确定所有子元素的位置。

简单看一看layout方法：

```java
public void layout(int l, int t, int r, int b) {
    if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
        onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
        mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
    }

    int oldL = mLeft;
    int oldT = mTop;
    int oldB = mBottom;
    int oldR = mRight;

    boolean changed = isLayoutModeOptical(mParent) ?
            setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
        onLayout(changed, l, t, r, b);

        if (shouldDrawRoundScrollbar()) {
            if(mRoundScrollbarRenderer == null) {
                mRoundScrollbarRenderer = new RoundScrollbarRenderer(this);
            }
        } else {
            mRoundScrollbarRenderer = null;
        }

        mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;

        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnLayoutChangeListeners != null) {
            ArrayList<OnLayoutChangeListener> listenersCopy =
                    (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
            int numListeners = listenersCopy.size();
            for (int i = 0; i < numListeners; ++i) {
                listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
            }
        }
    }

    final boolean wasLayoutValid = isLayoutValid();

    mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
    mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;

    if (!wasLayoutValid && isFocused()) {
        mPrivateFlags &= ~PFLAG_WANTS_FOCUS;
        if (canTakeFocus()) {
            // We have a robust focus, so parents should no longer be wanting focus.
            clearParentsWantFocus();
        } else if (getViewRootImpl() == null || !getViewRootImpl().isInLayout()) {
            // This is a weird case. Most-likely the user, rather than ViewRootImpl, called
            // layout. In this case, there's no guarantee that parent layouts will be evaluated
            // and thus the safest action is to clear focus here.
            clearFocusInternal(null, /* propagate */ true, /* refocus */ false);
            clearParentsWantFocus();
        } else if (!hasParentWantsFocus()) {
            // original requestFocus was likely on this view directly, so just clear focus
            clearFocusInternal(null, /* propagate */ true, /* refocus */ false);
        }
        // otherwise, we let parents handle re-assigning focus during their layout passes.
    } else if ((mPrivateFlags & PFLAG_WANTS_FOCUS) != 0) {
        mPrivateFlags &= ~PFLAG_WANTS_FOCUS;
        View focused = findFocus();
        if (focused != null) {
            // Try to restore focus as close as possible to our starting focus.
            if (!restoreDefaultFocus() && !hasParentWantsFocus()) {
                // Give up and clear focus once we've reached the top-most parent which wants
                // focus.
                focused.clearFocusInternal(null, /* propagate */ true, /* refocus */ false);
            }
        }
    }

    if ((mPrivateFlags3 & PFLAG3_NOTIFY_AUTOFILL_ENTER_ON_LAYOUT) != 0) {
        mPrivateFlags3 &= ~PFLAG3_NOTIFY_AUTOFILL_ENTER_ON_LAYOUT;
        notifyEnterOrExitForAutoFillIfNeeded(true);
    }

    notifyAppearedOrDisappearedForContentCaptureIfNeeded(true);
}
```

通过setFrame来设置View四个顶点的位置，即初始化mLeft、mRight、mTop、mBottom。四个顶点的位置确定了，View在父容器的位置也就确定了。

## Draw

Draw的过程就简单了，它的作用就是将View绘制到屏幕上面。View的绘制遵循着如下几步：

* 绘制背景 `background.draw(canvas)`
* 绘制自己 `onDraw`
* 绘制children `dispatchDraw`
* 绘制装饰 `onDrawScrollBars`

这四个步骤在源码中也明显可以看出。View的onDraw是空实现，不同的控件对内容的绘制都不同，所以留给控件自己实现，我们自定义View时需要重写onDraw方法来绘制我们自己的控件。在ViewGroup中，也有对dispatchDraw的实现，dispatchDraw中对子View进行遍历，并调用了drawChild方法，其中又调用了View的draw方法，完成对子View的绘制。

## 自定义View时注意以下问题

1. 让View支持wrap_content。
2. 如果有必要，支持padding。
3. 尽量不要在View中使用Handler。
4. View中如果有线程或者动画，及时停止。
5. View带有滑动嵌套情景时，处理好滑动冲突。