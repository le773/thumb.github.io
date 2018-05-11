watchdog 主要监控运行在system_server中服务的线程，比如ams，一旦发现阻塞将输出调用栈信息，甚至重启system_server进程

# 1. watchdog框架图 #

![watchdog.png](http://upload-images.jianshu.io/upload_images/74811-551e16afbf5fa3c3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


a. watchdog 继承自Thread
b. HandlerChecker作为watchdog的内部类，实现Runnable接口，用于检查保存Handler线程的状态、回调监视器方法；
c. RebootRequestReceiver 监听重启广播(`Intent.ACTION_REBOOT`)；当收到广播后，调用`PowerMS`重启系统；
d. BinderThreadMonitor 监视`binder`线程是否可用；
监视器将阻塞直到有一个可用的`binder`线程处理来自IPC的请求，目的是为了确保其他进程可以与服务通信；

----------

# 2. watchdog 初始化过程 #

### 2.1 watchdog启动 ###

```
#SystemServer.java
private void startOtherServices() {
        ...
        mSystemServiceManager.startBootPhase(SystemService.PHASE_WAIT_FOR_DEFAULT_DISPLAY);
        ...
        final Watchdog watchdog = Watchdog.getInstance();
        watchdog.init(context, mActivityManagerService); // 注册RebootRequestReceiver
        ...
        mActivityManagerService.systemReady(new Runnable() {
            @Override
            public void run() {
                Slog.i(TAG, "Making services ready");
                mSystemServiceManager.startBootPhase(
                        SystemService.PHASE_ACTIVITY_MANAGER_READY);
                ...
                startSystemUi(context); // 启动SystemUI
                ...
                Watchdog.getInstance().start(); // 启动watchdog线程
                ...
                mSystemServiceManager.startBootPhase(
                        SystemService.PHASE_THIRD_PARTY_APPS_CAN_START);
                ...
            }
        }
}
```

### 2.2 watchdog构造函数初始化 ###
```
static final long DEFAULT_TIMEOUT = DB ? 10*1000 : 60*1000; // 判断阻塞默认时间60s

private Watchdog() {
    // The shared foreground thread is the main checker.  It is where we
    // will also dispatch monitor checks and do other work.
    mMonitorChecker = new HandlerChecker(FgThread.getHandler(),
            "foreground thread", DEFAULT_TIMEOUT);
    mHandlerCheckers.add(mMonitorChecker);
    // Add checker for main thread.  We only do a quick check since there
    // can be UI running on the thread.
    mHandlerCheckers.add(new HandlerChecker(new Handler(Looper.getMainLooper()),
            "main thread", DEFAULT_TIMEOUT));
    // Add checker for shared UI thread.
    mHandlerCheckers.add(new HandlerChecker(UiThread.getHandler(),
            "ui thread", DEFAULT_TIMEOUT));
    // And also check IO thread.
    mHandlerCheckers.add(new HandlerChecker(IoThread.getHandler(),
            "i/o thread", DEFAULT_TIMEOUT));
    // And the display thread.
    mHandlerCheckers.add(new HandlerChecker(DisplayThread.getHandler(),
            "display thread", DEFAULT_TIMEOUT));
    // Initialize monitor for Binder threads.
    addMonitor(new BinderThreadMonitor());
}
```
此阶段主要添加监听的线程 前台（`foreground thread`）、主线程（`main thread`）、UI线程（`ui thread`）、IO线程（`i/o thread`）、Disaplay（`display thread`）以及Binder（`BinderThreadMonitor`）到`mHandlerCheckers`。

----------


# 3. WatchDog 监听机制 #
线程运行在`SystemServer`中
```
public void run() {
    boolean waitedHalf = false;
    boolean mSFHang = false;
    while (true) {
        final ArrayList<HandlerChecker> blockedCheckers;
        String subject;
        mSFHang = false;
        final boolean allowRestart;
        synchronized (this) {
            long timeout = CHECK_INTERVAL; // (DB ? 10*1000 : 60*1000) /2
            long SFHangTime;
            for (int i=0; i<mHandlerCheckers.size(); i++) {
                HandlerChecker hc = mHandlerCheckers.get(i);
                hc.scheduleCheckLocked(); //3.1  向watchdog 监控的线程的Handler 发送消息
            }
            long start = SystemClock.uptimeMillis();
            while (timeout > 0) { // 等待30s在向下执行
                try {
                    wait(timeout); 
                } catch (InterruptedException e) {
                    Log.wtf(TAG, e);
                }
                timeout = CHECK_INTERVAL - (SystemClock.uptimeMillis() - start); // ?> 30s
            }
            
            final int waitState = evaluateCheckerCompletionLocked(); // 3.3 计算HandlerChecker的状态
            if (waitState == COMPLETED) {
                // The monitors have returned; reset
                continue;
            } else if (waitState == WAITING) {
                // still waiting but within their configured intervals; back off and recheck
                continue;
            } else if (waitState == WAITED_HALF) {
                if (!waitedHalf) { // 首次阻塞超过30s
                    // We've waited half the deadlock-detection interval.  Pull a stack
                    // trace and wait another half.
                ArrayList<Integer> pids = new ArrayList<Integer>();
                pids.add(Process.myPid());
                ActivityManagerService.dumpStackTraces(true, pids, null, null,
                        NATIVE_STACKS_OF_INTEREST); // 第一次 打印调用栈
                    waitedHalf = true;
                }
                continue;
            }
            blockedCheckers = getBlockedCheckersLocked(); // 3.5 获取超时(debug?10s:60s)的线程队列
            subject = describeCheckersLocked(blockedCheckers); // 3.7  获取描述信息
            allowRestart = mAllowRestart;
        }

        // If we got here, that means that the system is most likely hung.
        // First collect stack traces from all threads of the system process.
        // Then kill this process so that the system will restart.
        // 系统可能挂机，手机栈信息，然后重启system_server
        ArrayList<Integer> pids = new ArrayList<Integer>();
        pids.add(Process.myPid());
        if (mPhonePid > 0) pids.add(mPhonePid);
        // Pass !waitedHalf so that just in case we somehow wind up here without having
        // dumped the halfway stacks, we properly re-initialize the trace file.
        final File stack = ActivityManagerService.dumpStackTraces(
                !waitedHalf, pids, null, null, NATIVE_STACKS_OF_INTEREST);
        // 打印NATIVE_STACKS_OF_INTEREST native 进程栈信息
       /*
    // Which native processes to dump into dropbox's stack traces
    public static final String[] NATIVE_STACKS_OF_INTEREST = new String[] {
        "/system/bin/audioserver",
        "/system/bin/cameraserver",
        "/system/bin/drmserver",
        "/system/bin/mediadrmserver",
        "/system/bin/mediaserver",
        "/system/bin/sdcard",
        "/system/bin/surfaceflinger",
        "media.codec",     // system/bin/mediacodec
        "media.extractor", // system/bin/mediaextractor
        "com.android.bluetooth",  // Bluetooth service
        };
        */        

        // Give some extra time to make sure the stack traces get written.
        // The system's been hanging for a minute, another second or two won't hurt much.
        SystemClock.sleep(2000); // 确保栈信息打印完成


        // Pull our own kernel thread stacks as well if we're configured for that
        if (RECORD_KERNEL_THREADS) {
            dumpKernelStackTraces();
        }

        // Trigger the kernel to dump all blocked threads, and backtraces on all CPUs to the kernel log
        // dump kernel中线程阻塞的信息
        doSysRq('w');
        doSysRq('l');

        // Try to add the error to the dropbox, but assuming that the ActivityManager
        // itself may be deadlocked.  (which has happened, causing this statement to
        // deadlock and the watchdog as a whole to be ineffective)
        Thread dropboxThread = new Thread("watchdogWriteToDropbox") {
                public void run() {
                    mActivity.addErrorToDropBox(
                            "watchdog", null, "system_server", null, null,
                            name, null, stack, null);
                }
            };
        dropboxThread.start();
        
        Slog.v(TAG, "** save all info before killnig system server **");
        mActivity.addErrorToDropBox("watchdog", null, "system_server", null, null, subject, null, null, null);
        
        
            if ((mSFHang == false) && (controller != null)) {
                Slog.i(TAG, "Reporting stuck state to activity controller");
                try {
                    Binder.setDumpDisabled("Service dumps disabled due to hung system process.");
                    Slog.i(TAG, "Binder.setDumpDisabled");
                    // 1 = keep waiting, -1 = kill system
                    int res = controller.systemNotResponding(subject);
                    if (res >= 0) {
                        Slog.i(TAG, "Activity controller requested to coninue to wait");
                        waitedHalf = false;
                        continue;
                    }
                    Slog.i(TAG, "Activity controller requested to reboot");
                } catch (RemoteException e) {
                }
            }

            // Only kill the process if the debugger is not attached.
            if (Debug.isDebuggerConnected()) {
                debuggerWasConnected = 2;
            }
            if (debuggerWasConnected >= 2) {
                Slog.w(TAG, "Debugger connected: Watchdog is *not* killing the system process");
            } else if (debuggerWasConnected > 0) {
                Slog.w(TAG, "Debugger was connected: Watchdog is *not* killing the system process");
            } else if (!allowRestart) {
                Slog.w(TAG, "Restart not allowed: Watchdog is *not* killing the system process");
            } else {
                Slog.w(TAG, "*** WATCHDOG KILLING SYSTEM PROCESS: " + subject);
                // 打印每一个超时线程的栈信息
                for (int i=0; i<blockedCheckers.size(); i++) {
                    Slog.w(TAG, blockedCheckers.get(i).getName() + " stack trace:");
                    StackTraceElement[] stackTrace
                            = blockedCheckers.get(i).getThread().getStackTrace();
                    for (StackTraceElement element: stackTrace) {
                        Slog.w(TAG, "    at " + element);
                    }
                }
                Slog.w(TAG, "*** GOODBYE!");
                Process.killProcess(Process.myPid()); // kill system_server
                System.exit(10);
            }
    }
}
```

### 3.1 HandlerChecker::scheduleCheckLocked ###
```
public void scheduleCheckLocked() {
    if (mMonitors.size() == 0 && mHandler.getLooper().getQueue().isPolling()) {
        // If the target looper has recently been polling, then
        // there is no reason to enqueue our checker on it since that
        // is as good as it not being deadlocked.  This avoid having
        // to do a context switch to check the thread.  Note that we
        // only do this if mCheckReboot is false and we have no
        // monitors, since those would need to be executed at this point.
        mCompleted = true;
        return;
    }

    if (!mCompleted) {
        // we already have a check in flight, so no need
        // 同一时刻，只允许一个任务
        return;
    }

    mCompleted = false; // 任务未完成
    mCurrentMonitor = null;
    mStartTime = SystemClock.uptimeMillis(); // 记录开始时间
    mHandler.postAtFrontOfQueue(this); // 将消息发送到监听线程的MQ, 将调用HandlerChecker的run方法处理
}
```

### 3.2 HandlerChecker::run ###
```
public void run() {
    final int size = mMonitors.size();
    for (int i = 0 ; i < size ; i++) {
        synchronized (Watchdog.this) {
            mCurrentMonitor = mMonitors.get(i);
        }
        mCurrentMonitor.monitor(); // 调用具体服务的monitor方法
    }

    synchronized (Watchdog.this) {
        mCompleted = true;
        mCurrentMonitor = null;
    }
}
```

#### 3.2.1 AMS::monitor ####
```
/** In this method we try to acquire our lock to make sure that we have not deadlocked */
public void monitor() {
    synchronized (this) { }
}
```
通过尝试获取锁判断是否发生死锁；

### 3.3 WatchDog::evaluateCheckerCompletionLocked ###

```
private int evaluateCheckerCompletionLocked() {
    int state = COMPLETED;
    for (int i=0; i<mHandlerCheckers.size(); i++) {
        HandlerChecker hc = mHandlerCheckers.get(i);
        state = Math.max(state, hc.getCompletionStateLocked()); // 获取状态
        // state: 取watchdog所监控线程中 最大的state
    }
    return state;
}
```
a. 当COMPLETED或WAITING,则不处理;
b. 当WAITED_HALF(超过30s)且为首次, 则第一次输出system_server 栈信息;
c. 当OVERDUE, 则输出更多信息（AMS::dumpStackTraces、Kernel::dumpKernelStackTraces、dropbox信息）

### 3.4 WatchDog::getCompletionStateLocked ###
```
public int getCompletionStateLocked() {
    if (mCompleted) {
        return COMPLETED;
    } else {
        long latency = SystemClock.uptimeMillis() - mStartTime; // 检测monitor的消息添加到Handler时间 mStartTime
        if (latency < mWaitMax/2) { // 0~30s
            return WAITING;
        } else if (latency < mWaitMax) { // 30~60s
            return WAITED_HALF;
        }
    }
    return OVERDUE; // > 60s
}
```

### 3.5 获取阻塞的Checkers ###
```
private ArrayList<HandlerChecker> getBlockedCheckersLocked() {
    ArrayList<HandlerChecker> checkers = new ArrayList<HandlerChecker>();
    for (int i=0; i<mHandlerCheckers.size(); i++) {
        HandlerChecker hc = mHandlerCheckers.get(i);
        if (hc.isOverdueLocked()) {
            checkers.add(hc);
        }
    }
    return checkers;
}
```
### 3.6 WatchDog::isOverdueLocked 超时判断 ###
```
public boolean isOverdueLocked() {
    // mWaitMax = (DEFAULT_TIMEOUT = DB ? 10*1000 : 60*1000)
    return (!mCompleted) && (SystemClock.uptimeMillis() > mStartTime + mWaitMax);
}
```

### 3.7 WatchDog::describeCheckersLocked ###
```
private String describeCheckersLocked(ArrayList<HandlerChecker> checkers) {
    StringBuilder builder = new StringBuilder(128);
    for (int i=0; i<checkers.size(); i++) {
        if (builder.length() > 0) {
            builder.append(", ");
        }
        builder.append(checkers.get(i).describeBlockedStateLocked()); // 所有阻塞HandlerChecker的信息
    }
    return builder.toString();
}
```
### 3.8 HandlerChecker::describeBlockedStateLocked  ###
```
public String describeBlockedStateLocked() {
    if (mCurrentMonitor == null) { // 非前台
        return "Blocked in handler on " + mName + " (" + getThread().getName() + ")";
    } else { // mMonitorChecker(foreground thread)
        return "Blocked in monitor " + mCurrentMonitor.getClass().getName()
                + " on " + mName + " (" + getThread().getName() + ")";
    }
}
```

----------

# 4. 定制watchdog保活进程 #
1. 定制watchdog生命周期依赖原生watchdog
    a. watchdog调用systemReady时，启动线程并处于等待状态wait
    b. 在定制的watchdog提前设定好要启动线程intent
2. 监听进程生命周期
    a. ams::startProcessLocked
    b. ams::handleAppDiedLocked
3. 重启
    a. 当AMS通过handleAppDiedLocked通知线程死亡，则在1启动线程的队列中添加任务，唤醒（notifyAll)watchdog 重启进程；
