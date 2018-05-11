----------

## 1.1  Window窗口管理框架图

![WMS窗口管理](http://upload-images.jianshu.io/upload_images/74811-4d61a0b2d9830b1c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

窗口的添加从`Activity::makeVisible`开始，由`WindowManagerImpl`委托给`WindowManagerGlobal`处理，`WindowManagerGlobal`内部的`mViews`、`mRoots`、`mParams`、`mDyingViews`分别管理窗口的试图、主线程、布局参数以及死亡过程中的视图；`ViewRootImpl`持有`Session`的代理端与WMS通信；


----------

## 2.1 ActivityThread::handleLaunchActivity
```
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent, String reason) {
    ...
    // Initialize before creating the activity
    WindowManagerGlobal.initialize(); // WindowManagerGlobal 初始化
    Activity a = performLaunchActivity(r, customIntent); // 2.2 ActivityThread::performLaunchActivity
    handleResumeActivity(r.token, false, r.isForward, !r.activity.mFinished && !r.startsNotResumed, r.lastProcessedSeq, reason); // 2.6 ActivityThread::handleResumeActivity
}
```
## 2.2 ActivityThread::performLaunchActivity
```
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    Activity activity = null;
    ...
    java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
    activity = mInstrumentation.newActivity(
            cl, component.getClassName(), r.intent); //newInstance
    ...
    activity.attach(appContext, this, getInstrumentation(), r.token,
        r.ident, app, r.intent, r.activityInfo, title, r.parent,
        r.embeddedID, r.lastNonConfigurationInstances, config,
        r.referrer, r.voiceInteractor, window); // 2.3 Activity::attach
    ...
    mInstrumentation.callActivityOnCreate(activity, r.state); //OnCreate
    ...
    activity.performStart();//Onstart
    ...
    mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state); // OnRestoreInstanceState
}
```

## 2.3 Activity::attach
```
final void attach(Context context, ActivityThread aThread,
        Instrumentation instr, IBinder token, int ident,
        Application application, Intent intent, ActivityInfo info,
        CharSequence title, Activity parent, String id,
        NonConfigurationInstances lastNonConfigurationInstances,
        Configuration config, String referrer, IVoiceInteractor voiceInteractor,
        Window window) {
    attachBaseContext(context); // 

    mWindow = new PhoneWindow(this, window); // 新建PhoneWindow
    mWindow.setWindowControllerCallback(this);
    mWindow.setCallback(this); // Window.CallBack的实现类为Activity
    mWindow.setOnWindowDismissedCallback(this);
    ...
    mToken = token; // 添加一个Activity窗口所需要的令牌
    ...
    mWindow.setWindowManager(
            (WindowManager)context.getSystemService(Context.WINDOW_SERVICE), /* 2.4.1 获取WindowManagerImpl */
            mToken, mComponent.flattenToString(),
            (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0); // 2.5.1 PhoneWindow::setWindowManager
    if (mParent != null) {
        mWindow.setContainer(mParent.getWindow());
    }
    mWindowManager = mWindow.getWindowManager(); // mWindowManager is WindowManagerImpl
    mCurrentConfig = config;
}
```

##  2.4.1 ContextImpl::getSystemService
```
public Object getSystemService(String name) {
    return SystemServiceRegistry.getSystemService(this, name); // 2.4.2 SystemServiceRegistry::registerService
}
```
##  2.4.2 SystemServiceRegistry::registerService
```
registerService(Context.WINDOW_SERVICE, WindowManager.class,
        new CachedServiceFetcher<WindowManager>() {
    @Override
    public WindowManager createService(ContextImpl ctx) {
        return new WindowManagerImpl(ctx);
}});
```

## 2.5.1 PhoneWindow::setWindowManager
```
public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
        boolean hardwareAccelerated) { // wm is WindowManagerImpl
    mAppToken = appToken;
    mAppName = appName;
    mHardwareAccelerated = hardwareAccelerated
            || SystemProperties.getBoolean(PROPERTY_HARDWARE_UI, false);
    if (wm == null) {
        wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
    }
    mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
}
```
## 2.5.2 WindowManagerImpl::createLocalWindowManager
```
public WindowManagerImpl createLocalWindowManager(Window parentWindow) {
    return new WindowManagerImpl(mContext, parentWindow);
}
```

## 2.6 ActivityThread::handleResumeActivity
```
final void handleResumeActivity(IBinder token,
        boolean clearHide, boolean isForward, boolean reallyResume, int seq, String reason) {
    ...
    r = performResumeActivity(token, clearHide, reason);
    // r.activity.performResume() -> Activity::OnResume
    ...
    if (r != null) {
        final Activity a = r.activity;
        ...
        if (r.window == null && !a.mFinished && willBeVisible) {
            r.window = r.activity.getWindow(); // PhoneWindow
            View decor = r.window.getDecorView(); // 2.6.1 PhoneWindow::getDecorView
            decor.setVisibility(View.INVISIBLE); // Activity的窗口在初创时不可见，因为尚不确定是否真的要显示窗口给用户
            ViewManager wm = a.getWindowManager(); // WindowManagerImpl
            WindowManager.LayoutParams l = r.window.getAttributes();
            a.mDecor = decor;
            l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION; // 类型：属于Acitivity的窗口
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
            if (a.mVisibleFromClient && !a.mWindowAdded) {
                a.mWindowAdded = true;
                wm.addView(decor, l); // WM is windowManagerImpl，添加到WMS
            }
        }
        // The window is now visible if it has been added, we are not
        // simply finishing, and we are not starting another activity.
        if (!r.activity.mFinished && willBeVisible
                && r.activity.mDecor != null && !r.hideForNow) {
            ...
            WindowManager.LayoutParams l = r.window.getAttributes();
            ...
            if (r.activity.mVisibleFromClient) {
                ViewManager wm = a.getWindowManager();
                View decor = r.window.getDecorView();
                wm.updateViewLayout(decor, l); // 通知wms更新布局
            }
            ...
                r.activity.makeVisible(); // 2.7 设置可见
            ...
        }
        ...
        if (!r.onlyLocalRequest) {
            r.nextIdle = mNewActivities;
            mNewActivities = r;
            Looper.myQueue().addIdleHandler(new Idler()); // 
        }
        ActivityManagerNative.getDefault().activityResumed(token); // 通知AMS Activity处于Resumed状态

    } else {
        // If an exception was thrown when trying to resume, then
        // just end this activity.
        // resume过程发生异常，则 finishActivity
            ActivityManagerNative.getDefault()
                .finishActivity(token, Activity.RESULT_CANCELED, null,
                        Activity.DONT_FINISH_TASK_WITH_ACTIVITY);
    }
}
```

## 2.6.1 PhoneWindow::getDecorView
```
public final View getDecorView() {
    if (mDecor == null || mForceDecorInstall) {
        installDecor(); //  2.6.2 
    }
    return mDecor;
}
```

##  2.6.2 PhoneWindow::installDecor
```
private void installDecor() {
    mForceDecorInstall = false;
    if (mDecor == null) {
        mDecor = generateDecor(-1); // 2.6.3 
        mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
        mDecor.setIsRootNamespace(true);
        if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
            mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
        }
    } else {
        mDecor.setWindow(this); // PhoneWindow::mWindow is PhoneWindow
    }
}
```
## 2.6.3 PhoneWindow::generateDecor
```
protected DecorView generateDecor(int featureId) {
    // System process doesn't have application context and in that case we need to directly use
    // the context we have. Otherwise we want the application context, so we don't cling to the
    // activity.
    Context context;
    ...
    context = getContext();
    ...
    return new DecorView(context, featureId, this, getAttributes());
}
```
## 2.6.4 ActivityDecorView相关结构图
![ActivityDecorView相关结构图](http://upload-images.jianshu.io/upload_images/74811-d924a1df6c26d80c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
`Activity`中`DecorView`为根视图

## 2.7 Activity::makeVisible 设置窗口可见
```
void makeVisible() {
    if (!mWindowAdded) {
        ViewManager wm = getWindowManager();
        wm.addView(mDecor, getWindow().getAttributes()); // 2.9 委托WindowManagerGlobal添加到wms
        mWindowAdded = true;
    }
    mDecor.setVisibility(View.VISIBLE); // 设置DecorView可见
}
```

## 2.8 WindowManagerImpl相关框架图
![WindowManagerImpl相关架构图](http://upload-images.jianshu.io/upload_images/74811-e182b1271b719901.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

接口`ViewManager`定义了`view`的基本操作增（`addView`）、删（`removeView`）、更新（`updateViewLayout`）;
接口`WindowManger`继承自`ViewManager`；
`WindowManagerImpl`实现`WindowManager`中的方法，并委托给`WindowMangerGlobal`处理；
`WindowManagerGlobal`中的`mViews`、`mRoots`、`mParams`、`mDyingViews`分别管理窗口的试图、主线程、布局参数以及死亡的试图；

## 2.9 WindowManagerGlobal::addView
```
    public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
        // 参数检查
        ...
        ViewRootImpl root;
        View panelParentView = null;

        synchronized (mLock) {
            ...
            int index = findViewLocked(view, false);
            if (index >= 0) {
                if (mDyingViews.contains(view)) {
                    // Don't wait for MSG_DIE to make it's way through root's queue.
                    mRoots.get(index).doDie(); // 3.1 如果已经包含此view,则先remove
                } 
            }
            ...
            root = new ViewRootImpl(view.getContext(), display); // 2.12 ViewRootImpl构造过程获取WindowSession
            view.setLayoutParams(wparams);
            mViews.add(view);
            mRoots.add(root);
            mParams.add(wparams); // 保存当前window
        }
        try {
            root.setView(view, wparams, panelParentView); // 2.12 更新布局
        } catch (RuntimeException e) {
            synchronized (mLock) {
                final int index = findViewLocked(view, false);
                if (index >= 0) {
                    removeViewLocked(index, true); // remove试图
                }
            }
        }
    }
```
## 2.10 WindowManagerGlobal::addView 相关调用栈
```
WindowManagerImpl.addView()
    new ViewRoot
    // 保存到 mViews  mRoots  mParams
    ViewRoot.setView(xx)
        requestLayout // 检查主线程
            scheduleTraversals
                sendEmptyMessage
                handleMessage
                    performTraversals  // 2.11 测量 布局 绘制
                        ViewRoot.relayoutWindow
                            IWindowSession.relayout // 参考relayout的调用栈
                        draw
                            View.draw
                        scheduleTralScheduled // try again
        mWindowSession.addToDisplay // 2.15 与requestLayout同级
        // sWindowSession.add
            WMS.addWindow // 2.16
                new WindowState // 
                WindowState.attach // 
                    Session.windowAddedLocked
                        new SurfaceSession // 开辟一块内存，由SurfaceFlinger进行混合处理
```

## 2.11 测量 布局 绘制流程
![performTraversals基本流程](http://upload-images.jianshu.io/upload_images/74811-4f491dd0bf06f8d3.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 2.12 WindowMangerGlobal::getWindowSession
```
public static IWindowSession getWindowSession() {
    synchronized (WindowManagerGlobal.class) {
        if (sWindowSession == null) {
            try {
                InputMethodManager imm = InputMethodManager.getInstance();
                IWindowManager windowManager = getWindowManagerService(); // wms
                sWindowSession = windowManager.openSession( // 2.13 wms::opensession
                        new IWindowSessionCallback.Stub() {
                            @Override
                            public void onAnimatorScaleChanged(float scale) {
                                ValueAnimator.setDurationScale(scale);
                            }
                        },
                        imm.getClient(), imm.getInputContext());
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        }
        return sWindowSession;
    }
}
```

##  2.13 WindowMangerService::openSession
```
public IWindowSession openSession(IWindowSessionCallback callback, IInputMethodClient client,
        IInputContext inputContext) {
    if (client == null) throw new IllegalArgumentException("null client");
    if (inputContext == null) throw new IllegalArgumentException("null inputContext");
    Session session = new Session(this, callback, client, inputContext);
    return session;
}
```

## 2.14 WMS::Session与ViewRootImpl关系
![Session与ViewRootImpl关系](http://upload-images.jianshu.io/upload_images/74811-5e91467c7042b1be.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
`ViewRootImpl`获取`Session`的代理类，通过`Binder::IPC`通信；
`Session`持有`WMS`服务；
`InputState`为事件传递相关的类；
`W`代表当前操作的窗口, 是`ViewRootImpl`的内部类，且持有`ViewRootImpl`的弱引用

## 2.15 Session::addToDisplay
```
public int addToDisplay(IWindow window, int seq, WindowManager.LayoutParams attrs,
        int viewVisibility, int displayId, Rect outContentInsets, Rect outStableInsets,
        Rect outOutsets, InputChannel outInputChannel) {
    return mService.addWindow(this, window, seq, attrs, viewVisibility, displayId,
            outContentInsets, outStableInsets, outOutsets, outInputChannel);
}
```

## 2.16 WMS::addWindow
```
/**
 * All currently active sessions with clients.
 */
final ArraySet<Session> mSessions = new ArraySet<>();

/**
 * Mapping from an IWindow IBinder to the server's Window object.
 * This is also used as the lock for all of our state.
 * NOTE: Never call into methods that lock ActivityManagerService while holding this object.
 */
final HashMap<IBinder, WindowState> mWindowMap = new HashMap<>();

/**
 * Mapping from a token IBinder to a WindowToken object.
 */
final HashMap<IBinder, WindowToken> mTokenMap = new HashMap<>();

public int addWindow(Session session, IWindow client, int seq,
            WindowManager.LayoutParams attrs, int viewVisibility, int displayId,
            Rect outContentInsets, Rect outStableInsets, Rect outOutsets,
            InputChannel outInputChannel) {
        int res = mPolicy.checkAddPermission(attrs, appOp); // PhoneWindowManager进行权限检查
        // 接下来依然是做一些检查
        ...
        WindowState win = new WindowState(this, session, client, token,
                attachedWindow, appOp[0], seq, attrs, viewVisibility, displayContent);
        ...
        mPolicy.adjustWindowParamsLw(win.mAttrs);// 调整window参数
        ...
        res = mPolicy.prepareAddWindowLw(win, attrs);
        ...
        win.openInputChannel(outInputChannel); // 窗口端IMS管道


        win.attach(); // 2.17
        mWindowMap.put(client.asBinder(), win); // 保存到mWindowMap

        boolean imMayMove = true;

        if (type == TYPE_INPUT_METHOD) {
            addInputMethodWindowToListLocked(win);
        } else if (type == TYPE_INPUT_METHOD_DIALOG) {
            addWindowToListInOrderLocked(win, true);
        } else {
            addWindowToListInOrderLocked(win, true);
        }
        ...
        final WindowStateAnimator winAnimator = win.mWinAnimator;
        winAnimator.mEnterAnimationPending = true;
        winAnimator.mEnteringAnimation = true;
        ...
        boolean focusChanged = false;
        if (win.canReceiveKeys()) {
            focusChanged = updateFocusedWindowLocked(UPDATE_FOCUS_WILL_ASSIGN_LAYERS,
                    false /*updateInputWindows*/); // 更新焦点窗口
        }
        ...
        if (focusChanged) {
            mInputMonitor.setInputFocusLw(mCurrentFocus, false /*updateInputWindows*/);
        }
        mInputMonitor.updateInputWindowsLw(false /*force*/);
    }
}
```
`mTokenMap`： 保存所有的显示令牌, 用于窗口管理;
`mWindowMap`：保存所有窗口的状态信息（`windowstate`）, 用于窗口管理;
`mSession`：保存当前所有想向WMS寻求窗口管理服务的客户端;

## 2.17 WindowState::attach
```
void attach() {
    mSession.windowAddedLocked(); // 2.18
}
```

## 2.18 WindowState::windowAddedLocked
```
void windowAddedLocked() {
    if (mSurfaceSession == null) {
        mSurfaceSession = new SurfaceSession(); // 在内存中创建一块空间，用于SurfaceFlinger
        mService.mSessions.add(this); //记录当前Session
        if (mLastReportedAnimatorScale != mService.getCurrentAnimatorScale()) {
            mService.dispatchNewAnimatorScaleLocked(this);
        }
    }
    mNumWindow++;
}
```

----------

## 3.1 ViewRootImpl::doDie
```
void doDie() {
    checkThread(); // 检查主线程
    synchronized (this) {
        mRemoved = true; // 清楚标记
        if (mAdded) {
            dispatchDetachedFromWindow(); // 3.2 清理窗口的入口
        }

        if (mAdded && !mFirst) {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "destroyHardwareRenderer");
            destroyHardwareRenderer();
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);

            if (mView != null) {
                int viewVisibility = mView.getVisibility();
                boolean viewVisibilityChanged = mViewVisibility != viewVisibility;
                if (mWindowAttributesChanged || viewVisibilityChanged) {
                    // If layout params have been changed, first give them
                    // to the window manager to make sure it has the correct
                    // animation info.
                    try {
                        if ((relayoutWindow(mWindowAttributes, viewVisibility, false)
                                & WindowManagerGlobal.RELAYOUT_RES_FIRST_TIME) != 0) {
                            mWindowSession.finishDrawing(mWindow);
                        }
                    } catch (RemoteException e) {
                        Log.e(mTag, "RemoteException when finish draw window " + mWindow
                                + " in " + this, e);
                    }
                }

                mSurface.release();
            }
        }
        mAdded = false;
    }
    WindowManagerGlobal.getInstance().doRemoveView(this); //3.3 移除WindowManagerGlobal中的数据
}
```
## 3.2 ViewRootImpl::dispatchDetachedFromWindow
```
void dispatchDetachedFromWindow() {
    if (mView != null && mView.mAttachInfo != null) {
        mAttachInfo.mTreeObserver.dispatchOnWindowAttachedChange(false);
        mView.dispatchDetachedFromWindow(); // 
    }
    ...
    mView.assignParent(null);
    mView = null;
    mAttachInfo.mRootView = null;

    mSurface.release(); // surface释放

    if (mInputQueueCallback != null && mInputQueue != null) {
        mInputQueueCallback.onInputQueueDestroyed(mInputQueue);
        mInputQueue.dispose(); // 事件队列清理
        mInputQueueCallback = null;
        mInputQueue = null;
    }
    if (mInputEventReceiver != null) {
        mInputEventReceiver.dispose(); // 输入事件清理
        mInputEventReceiver = null;
    }
    try {
        mWindowSession.remove(mWindow); // 通知wms移除window
    } catch (RemoteException e) {
        Log.e(mTag, "RemoteException remove window " + mWindow + " in " + this, e);
    }

    // Dispose the input channel after removing the window so the Window Manager
    // doesn't interpret the input channel being closed as an abnormal termination.
    if (mInputChannel != null) {
        mInputChannel.dispose(); // IMS管道清理
        mInputChannel = null;
    }

    mDisplayManager.unregisterDisplayListener(mDisplayListener);
    unscheduleTraversals();
}
```

## 3.3 ViewRootImpl::doRemoveView
```
void doRemoveView(ViewRootImpl root) {
    synchronized (mLock) {
        final int index = mRoots.indexOf(root);
        if (index >= 0) {
            mRoots.remove(index);
            mParams.remove(index);
            final View view = mViews.remove(index);
            mDyingViews.remove(view);
        }
    }
}
```
