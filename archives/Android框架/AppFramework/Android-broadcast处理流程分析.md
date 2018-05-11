## 1.1 动态广播注册后的数据结构图

![Broadcast相关数据结构类图](http://upload-images.jianshu.io/upload_images/74811-82001cd8f2293b5e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###### BroadcastFilter
继承自`IntentFilter`，它记录了当前需注册的`Receiver`的`IntentFilter`信息；
描述广播接收者；

###### ReceiverList
```
final class ReceiverList extends ArrayList<BroadcastFilter>implements IBinder.DeathRecipient
```
保存使用相同`InnerReceiver`对象来注册的广播接收者，并以`InnerReceiver.asBinder`作为关键字；

###### BroadcastRecord
记录`broadcast`的信息



## 2.1 AMS::broadcastIntentLocked 发送广播
```
/**
 * State of all active sticky broadcasts per user.  Keys are the action of the
 * sticky Intent, values are an ArrayList of all broadcasted intents with
 * that action (which should usually be one).  The SparseArray is keyed
 * by the user ID the sticky is for, and can include UserHandle.USER_ALL
 * for stickies that are sent to all users.
 */
final SparseArray<ArrayMap<String, ArrayList<Intent>>> mStickyBroadcasts =
        new SparseArray<ArrayMap<String, ArrayList<Intent>>>();

final int broadcastIntentLocked(ProcessRecord callerApp,
        String callerPackage, Intent intent, String resolvedType,
        IIntentReceiver resultTo, int resultCode, String resultData,
        Bundle resultExtras, String[] requiredPermissions, int appOp, Bundle bOptions,
        boolean ordered/*有序 无序标志*/, boolean sticky, int callingPid, int callingUid, int userId) {
    //1.  Flag处理
    // By default broadcasts do not go to stopped apps.
    intent.addFlags(Intent.FLAG_EXCLUDE_STOPPED_PACKAGES); // 默认不发送给force-stop的activity
    // If we have not finished booting, don't allow this to launch new processes. 系统未启动时，不允许启动新的进程
    if (!mProcessesReady && (intent.getFlags()&Intent.FLAG_RECEIVER_BOOT_UPGRADE) == 0) {
        intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY);
    }

    // 2. 权限管理 检查受保护的广播等
    // Verify that protected broadcasts are only being sent by system code,
    // and that system code is only sending protected broadcasts.
    final String action = intent.getAction();
    final boolean isProtectedBroadcastisProtectedBroadcast = AppGlobals.getPackageManager().isProtectedBroadcast(action);
    ...
     // 3. 处理系统广播
    switch (action) {
        case Intent.ACTION_UID_REMOVED:
        case Intent.ACTION_PACKAGE_REMOVED:
        ..
    }
    ...
    // 4. 处理粘性广播
    // Add to the sticky list if requested.
    if (sticky) { ... } // mStickyBroadcasts 保存粘性广播
    ...
    // 5. 收集静态 动态注册的广播
    List receivers = collectReceiverComponents(intent, resolvedType, callingUid, users);
    List<BroadcastFilter> registeredReceiversForUser = mReceiverResolver.queryIntent(intent, resolvedType, false, users[i]);

    ...
    // 6. 无序广播且存在动态注册的广播
    int NR = registeredReceivers != null ? registeredReceivers.size() : 0;
    if (!ordered && NR > 0) {
        // If we are not serializing this broadcast, then send the
        // registered receivers separately so they don't wait for the
        // components to be launched.
        final BroadcastQueue queue = broadcastQueueForIntent(intent); // 2.2
        BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
                callerPackage, callingPid, callingUid, resolvedType, requiredPermissions,
                appOp, brOptions, registeredReceivers/*动态注册的广播*/, resultTo, resultCode, resultData,
                resultExtras, ordered, sticky, false, userId);
        final boolean replaced = replacePending && queue.replaceParallelBroadcastLocked(r);
        if (!replaced) {
            queue.enqueueParallelBroadcastLocked(r); // 加入并行广播队列
            queue.scheduleBroadcastsLocked(); //2.3 通过发送消息触发系统处理广播BROADCAST_INTENT_MSG
        }
        registeredReceivers = null;
        NR = 0;
    }
    // 7. 合并广播
    // Merge into one list. 
    // 当前发送的是order广播，非order广播在6已经处理
    int ir = 0;
    int NT = receivers != null ? receivers.size() : 0;
    int it = 0;
    ResolveInfo curt = null;
    BroadcastFilter curr = null;
    while (it < NT && ir < NR) {
        if (curt == null) {
            curt = (ResolveInfo)receivers.get(it);
        }
        if (curr == null) {
            curr = registeredReceivers.get(ir);
        }
        if (curr.getPriority() >= curt.priority) { // 如果动态注册广播优先级大于静态注册的广播
            // Insert this broadcast record into the final list.
            receivers.add(it, curr);
            ir++;
            curr = null;
            it++;
            NT++;
        } else {
            // Skip to the next ResolveInfo in the final list.
            it++;
            curt = null;
        }
    }
    while (ir < NR) {
        if (receivers == null) {
            receivers = new ArrayList();
        }
        receivers.add(registeredReceivers.get(ir)); // 剩下动态注册的广播合并到静态广播队列
        ir++;
    }
    ...
    // 8. 发送广播
    if ((receivers != null && receivers.size() > 0)
            || resultTo != null) {
        BroadcastQueue queue = broadcastQueueForIntent(intent); // 2.2
        BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
                callerPackage, callingPid, callingUid, resolvedType,
                requiredPermissions, appOp, brOptions, receivers/* 静态 + 动态 */, resultTo, resultCode,
                resultData, resultExtras, ordered, sticky, false, userId);
        ...
        boolean replaced = replacePending && queue.replaceOrderedBroadcastLocked(r); 
        if (!replaced) {
            queue.enqueueOrderedBroadcastLocked(r); // 串行广播队列
            queue.scheduleBroadcastsLocked(); //2.3 通过发送消息触发系统处理广播BROADCAST_INTENT_MSG
        }
    ...
}
```

## 2.2 AMS::broadcastQueueForIntent
```
BroadcastQueue broadcastQueueForIntent(Intent intent) {
    final boolean isFg = (intent.getFlags() & Intent.FLAG_RECEIVER_FOREGROUND) != 0; // 是否是前台广播
    return (isFg) ? mFgBroadcastQueue : mBgBroadcastQueue;
}
```
根据`Intent.FLAG_RECEIVER_FOREGROUND`获取前台或者后台广播；

## 2.3 BroadcastQueue::handleMessage
```
private final class BroadcastHandler extends Handler {
    public BroadcastHandler(Looper looper) {
        super(looper, null, true);
    }

    @Override
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case BROADCAST_INTENT_MSG: {
                processNextBroadcast(true);  // 3.1 处理广播
            } break;
            case BROADCAST_TIMEOUT_MSG: {
                synchronized (mService) {
                    broadcastTimeoutLocked(true); // 3.6
                }
            } break;
            case SCHEDULE_TEMP_WHITELIST_MSG: {
                DeviceIdleController.LocalService dic = mService.mLocalDeviceIdleController;
                if (dic != null) {
                    dic.addPowerSaveTempWhitelistAppDirect(UserHandle.getAppId(msg.arg1),
                            msg.arg2, true, (String)msg.obj);
                }
            } break;
        }
    }
}
```
`fromMsg`:指`processNextBroadcast()`是否由`BroadcastHandler`所调用的。

## 3.1 BroadcastQueue::processNextBroadcast
```
final void processNextBroadcast(boolean fromMsg) {
    synchronized(mService) {
        // First, deliver any non-serialized broadcasts right away. // 无序广播
        while (mParallelBroadcasts.size() > 0) { // 遍历无序广播
            r = mParallelBroadcasts.remove(0);
            // when dispatch started on this set of receivers
            r.dispatchTime = SystemClock.uptimeMillis(); 
            r.dispatchClockTime = System.currentTimeMillis();
            final int N = r.receivers.size();
            for (int i=0; i<N; i++) { // 遍历当前无序广播的接收者
                Object target = r.receivers.get(i); // 参考动态广播的存储结构
                deliverToRegisteredReceiverLocked(r, (BroadcastFilter)target, false, i); // 3.2 处理动态广播
            }
            addBroadcastToHistoryLocked(r);
        }

        // Now take care of the next serialized(串行) one... // 开始处理有序广播

        // If we are waiting for a process to come up to handle the next // 如果正在等待进程处理下一个广播，则什么也不做
        // broadcast, then do nothing at this point.  Just in case, we // 以防万一，如果检查正在等待的进程是否存在
        // check that the process we're waiting for still exists.
        if (mPendingBroadcast != null) { // 该进程可能先需要启动，然后在处理广播（processCurBroadcastLocked）
            boolean isDead;
            synchronized (mService.mPidsSelfLocked) {
                ProcessRecord proc = mService.mPidsSelfLocked.get(mPendingBroadcast.curApp.pid); // 如果进程处理完curApp应该为null
                isDead = proc == null || proc.crashing;
            }
            if (!isDead) {
                // It's still alive, so keep waiting
                return;
            } else { // 如果该进程已经死掉，则重置mPendingBroadcast
                Slog.w(TAG, "pending app  ["
                        + mQueueName + "]" + mPendingBroadcast.curApp
                        + " died before responding to broadcast");
                mPendingBroadcast.state = BroadcastRecord.IDLE;
                mPendingBroadcast.nextReceiver = mPendingBroadcastRecvIndex;
                mPendingBroadcast = null;
            }
        }

        // 获取第一条有序广播
        do {
            if (mOrderedBroadcasts.size() == 0) { // 无有序广播时，通知AMS进行GC
                // No more broadcasts pending, so all done!
                mService.scheduleAppGcsLocked();
                ...
                return;
            }
            r = mOrderedBroadcasts.get(0); // 获取第一个有序广播
            boolean forceReceive = false;

            // This is only done if the system is ready so that PRE_BOOT_COMPLETED
            // receivers don't get executed with timeouts. They're intended for
            // one time heavy lifting after system upgrades and can take
            // significant amounts of time.
            int numReceivers = (r.receivers != null) ? r.receivers.size() : 0;
            if (mService.mProcessesReady && r.dispatchTime > 0) { // systemReady
                long now = SystemClock.uptimeMillis();
                if ((numReceivers > 0) &&
                        (now > r.dispatchTime + (2*mTimeoutPeriod*numReceivers)) &&
                        /// M: ANR debug mechanism for defer ANR
                        !mService.mANRManager.isAnrDeferrable()) { 
                    broadcastTimeoutLocked(false); // 3.6
                    // 1. finishReceiverLocked:: forcibly finish this broadcast 强制finish广播
                    // 2. mOrderedBroadcasts中下标为0的应用进程ANR
                    forceReceive = true;
                    r.state = BroadcastRecord.IDLE;
                    // 如果超时，当前广播的下一个及之后接收者接收不到广播
                }
            }
            // 1. 广播接收者已经全部分发
            // 2. 无下一个广播
            // 3. 有序广播被中断
            // 4. 强制finish广播
            if (r.receivers == null || r.nextReceiver >= numReceivers || r.resultAbort || forceReceive) { // 
                // No more receivers for this broadcast!  Send the final
                // result if requested...
                if (r.resultTo != null) {
                    try {
                        performReceiveLocked(r.callerApp, r.resultTo,
                            new Intent(r.intent), r.resultCode,
                            r.resultData, r.resultExtras, false, false, r.userId); // 3.3 结束当前广播
                        // Set this to null so that the reference
                        r.resultTo = null;
                }

                cancelBroadcastTimeoutLocked(); // 3.6 broadcast 取消ANR

                // ... and on to the next...
                addBroadcastToHistoryLocked(r);
                if (r.intent.getComponent() == null && r.intent.getPackage() == null
                        && (r.intent.getFlags()&Intent.FLAG_RECEIVER_REGISTERED_ONLY) == 0) {
                    // This was an implicit broadcast... let's record it for posterity.
                    mService.addBroadcastStatLocked(r.intent.getAction(), r.callerPackage,
                            r.manifestCount, r.manifestSkipCount, r.finishTime-r.dispatchTime);
                }
                mOrderedBroadcasts.remove(0);
                r = null;
                looped = true;
                continue;
            }
        } while (r == null); //



        // Get the next receiver... 获取下一条广播
        int recIdx = r.nextReceiver++;

        // Keep track of when this receiver started, and make sure there
        // is a timeout message pending to kill it if need be. 记录广播开始时间
        r.receiverTime = SystemClock.uptimeMillis();
        if (recIdx == 0) {
            r.dispatchTime = r.receiverTime;
            r.dispatchClockTime = System.currentTimeMillis();
        }
        if (! mPendingBroadcastTimeoutMessage) {
            long timeoutTime = r.receiverTime + mTimeoutPeriod;
            setBroadcastTimeoutLocked(timeoutTime); // 延迟发送ANR超时通知消息
        }

        final BroadcastOptions brOptions = r.options;
        final Object nextReceiver = r.receivers.get(recIdx);

        if (nextReceiver instanceof BroadcastFilter) {
            // Simple case: this is a registered receiver who gets
            // a direct call.
            BroadcastFilter filter = (BroadcastFilter)nextReceiver;
            deliverToRegisteredReceiverLocked(r, filter, r.ordered, recIdx); // 3.2 有序队列中的动态广播
            ...
            return;
        }
        ...
        ResolveInfo info = (ResolveInfo)nextReceiver; // 静态广播
        ComponentName component = new ComponentName(info.activityInfo.applicationInfo.packageName, info.activityInfo.name);
        ...
        if (skip) { // 权限等检查不通过
            r.delivery[recIdx] = BroadcastRecord.DELIVERY_SKIPPED;
            r.receiver = null;
            r.curFilter = null;
            r.state = BroadcastRecord.IDLE;
            scheduleBroadcastsLocked();
            return;
        }
        ...
        // Broadcast is being executed, its package can't be stopped.
        AppGlobals.getPackageManager().setPackageStoppedState(r.curComponent.getPackageName(), false, UserHandle.getUserId(r.callingUid)); // 不能处于stop状态

        // Is this receiver's application already running?
        if (app != null && app.thread != null) {
            app.addPackage(info.activityInfo.packageName,
                    info.activityInfo.applicationInfo.versionCode, mService.mProcessStats);
            processCurBroadcastLocked(r, app); // 3.7 处理静态广播
            return;
        } catch { // 异常
            finishReceiverLocked(r, r.resultCode, r.resultData, r.resultExtras, r.resultAbort, false);
            scheduleBroadcastsLocked();
        }

       if ((r.curApp=mService.startProcessLocked(targetProcess,
                info.activityInfo.applicationInfo, true,
                r.intent.getFlags() | Intent.FLAG_FROM_BACKGROUND,
                "broadcast", r.curComponent,
                (r.intent.getFlags()&Intent.FLAG_RECEIVER_BOOT_UPGRADE) != 0, false, false))
                        == null) {
            // 启动进程失败的处理
            return;
        }

        mPendingBroadcast = r;
        mPendingBroadcastRecvIndex = recIdx;
}
```

## 3.2 BroadcastQueue::deliverToRegisteredReceiverLocked
```
private void deliverToRegisteredReceiverLocked(BroadcastRecord r,
        BroadcastFilter filter, boolean ordered, int index) {
        ... // 权限检查
        // If this is not being sent as an ordered broadcast, then we 如果不是有序广播，则不记录
        // don't want to touch the fields that keep track of the current
        // state of ordered broadcasts.
        if (ordered) {
            r.receiver = filter.receiverList.receiver.asBinder();
            r.curFilter = filter;
            filter.receiverList.curBroadcast = r;
            r.state = BroadcastRecord.CALL_IN_RECEIVE;
            if (filter.receiverList.app != null) {
                // Bump hosting application to no longer be in background
                // scheduling class.  Note that we can't do that if there
                // isn't an app...  but we can only be in that case for
                // things that directly call the IActivityManager API, which
                // are already core system stuff so don't matter for this.
                r.curApp = filter.receiverList.app;
                filter.receiverList.app.curReceiver = r;
                mService.updateOomAdjLocked(r.curApp);
            }
        }
        ...
        if (filter.receiverList.app != null && filter.receiverList.app.inFullBackup) {
            // Skip delivery if full backup in progress 备份状态且是有序广播，则skip
            // If it's an ordered broadcast, we need to continue to the next receiver.
            if (ordered) {
                skipReceiverLocked(r);
            }
        } else { // skip == false
            performReceiveLocked(filter.receiverList.app, filter.receiverList.receiver,
                    new Intent(r.intent), r.resultCode, r.resultData,
                    r.resultExtras, r.ordered, r.initialSticky, r.userId); // 3.3
        }
        if (ordered) {
            r.state = BroadcastRecord.CALL_DONE_RECEIVE;
        }
}
```
## 3.3 BroadcastQueue::performReceiveLocked
```
void performReceiveLocked(ProcessRecord app, IIntentReceiver receiver,
        Intent intent, int resultCode, String data, Bundle extras,
        boolean ordered, boolean sticky, int sendingUser) throws RemoteException {
    // Send the intent to the receiver asynchronously using one-way binder calls.
    if (app != null) { // 动态注册的无序广播 app进程是存在的，走else
        if (app.thread != null) {
            // If we have an app thread, do the call through that so it is
            // correctly ordered with other one-way calls.
            app.thread.scheduleRegisteredReceiver(receiver, intent, resultCode, data, extras, ordered, sticky, sendingUser, app.repProcState);
        }
    } else {
        receiver.performReceive(intent, resultCode, data, extras, ordered,
                sticky, sendingUser); // 3.4
    }
}
```

## 3.4 LoadedApk::ReceiverDispatcher::InnerReceiver::performReceive
```
@Override
public void performReceive(Intent intent, int resultCode, String data,
        Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
    final LoadedApk.ReceiverDispatcher rd;
    ...
    rd = mDispatcher.get();
    ...
    if (rd != null) {
        rd.performReceive(intent, resultCode, data, extras,
                ordered, sticky, sendingUser);
    } else {
        // The activity manager dispatched a broadcast to a registered
        // receiver in this process, but before it could be delivered the
        // receiver was unregistered.  Acknowledge the broadcast on its
        ...
    }
}
```
## 3.5 LoadedApk::ReceiverDispatcher::performReceive
```
public void performReceive(Intent intent, int resultCode, String data,
        Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
    final Args args = new Args(intent, resultCode, data, extras, ordered,
            sticky, sendingUser);
    ...
    if (intent == null || !mActivityThread.post(args)) { 
        if (mRegistered && ordered) { // 有序广播，则 sendFinished
            ... 
            args.sendFinished(mgr);
        }
    }
}
```
- 通过`post`方式调用`Args::run`方法，最终调用`Receiver`的`onReceive`方法；
- `finish`委托调用`AMS::finishReceiver`

## 3.6 BroadcastQueue::broadcastTimeoutLocked
```
    final void broadcastTimeoutLocked(boolean fromMsg) {
        if (fromMsg) { // fromMsg = false
            ...
        }

        BroadcastRecord br = mOrderedBroadcasts.get(0); // 获取第一个有序广播
        if (br.state == BroadcastRecord. ) {
            // In this case the broadcast had already finished, but we had decided to wait // 在开始前，等待启动的服务完成
            // for started services to finish as well before going on.  So if we have actually
            // waited long enough time timeout the broadcast, let's give up on the whole thing
            // and just move on to the next. 如果超时，处理下一个广播
            Slog.i(TAG, "Waited long enough for: " + (br.curComponent != null
                    ? br.curComponent.flattenToShortString() : "(null)"));
            br.curComponent = null;
            br.state = BroadcastRecord.IDLE;
            processNextBroadcast(false); // 处理下一条广播
            return;
        }
        ...
        // Move on to the next receiver.
        finishReceiverLocked(r, r.resultCode, r.resultData, r.resultExtras, r.resultAbort, false);
        scheduleBroadcastsLocked();

        if (anrMessage != null) {
            // Post the ANR to the handler since we do not want to process ANRs while
            // potentially holding our lock.
            mHandler.post(new AppNotResponding(app, anrMessage));
        }
    }
```

## 3.7 BroadcastQueue::processCurBroadcastLocked
```
private final void processCurBroadcastLocked(BroadcastRecord r,
        ProcessRecord app) throws RemoteException {
    ...
    r.receiver = app.thread.asBinder();
    r.curApp = app;
    app.curReceiver = r;
    app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_RECEIVER);
    mService.updateLruProcessLocked(app, false, null);
    mService.updateOomAdjLocked();

    // Tell the application to launch this receiver.
    r.intent.setComponent(r.curComponent);

    boolean started = false;
    try {
        ...
        app.thread.scheduleReceiver(new Intent(r.intent), r.curReceiver,
                mService.compatibilityInfoForPackageLocked(r.curReceiver.applicationInfo),
                r.resultCode, r.resultData, r.resultExtras, r.ordered, r.userId,
                app.repProcState);
        ...
        started = true;
    } finally {
        if (!started) {
            if (DEBUG_BROADCAST)  Slog.v(TAG_BROADCAST,
                    "Process cur broadcast " + r + ": NOT STARTED!");
            r.receiver = null;
            r.curApp = null; // 处理完成之后置null
            app.curReceiver = null;
        }
    }
}
```

## 3.8 动态注册的广播`ActivityThread`中的处理:
最终调用`rd.innnerevicer`的`performreceiver`，然后调用其外部类的`performreceiver`，此时候会通过`Handler`消息`Args`分发在子线程加载(`classloader`) `BroadcastReceiver`，并回调`onReceiver`；

## 4.1 processNextBroadcast分发广播流程图
![processNextBroadcast分发广播流程图](http://my.csdn.net/uploads/201204/05/1333636353_4017.jpg)
###### 参考
[android Application Component研究之BroadcastReceiver](http://blog.csdn.net/windskier/article/details/7251742 "android Application Component研究之BroadcastReceiver")


## 4.2 广播发送整体流程图
![广播发送整体流程图](http://upload-images.jianshu.io/upload_images/74811-78ff82e021bcf314.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
