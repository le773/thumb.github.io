### 1.1 updateOomAdjLocked 概览
![updateOomAdjLocked](http://upload-images.jianshu.io/upload_images/74811-55e5c0bb381f9d36.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 1.2 updateOomAdjLocked中的变量初始化
```
mProcessLimit:ProcessList.MAX_CACHED_APPS // 系统默认32
mProcessLimit:emptyProcessLimit//16空进程上限+cachedProcessLimit//16缓存进程上限
LRU进程队列长度 = numEmptyProcs + mNumCachedHiddenProcs //缓存 + mNumNonCachedProcs //非缓存
numSlots = (CACHED_APP_MAX_ADJ//15 - CACHED_APP_MIN_ADJ//9 + 1) / 2 = 3 
emptyFactor = numEmptyProcs/numSlots //每个槽的进程数量

ProcessList
空进程存活时间 MAX_EMPTY_TIME  30min
MAX_EMPTY_APPS  = MAX_CACHED_APPS/2 = 16
TRIM_EMPTY_APPS = MAX_EMPTY_APPS/2 = 8
TRIM_CACHED_APPS= (MAX_CACHED_APPS - MAX_EMPTY_APPS )/3 = 5
TRIM_CRITICAL_THRESHOLD = 3
```

### 1.3 updateOomAdjLocked 分析
```
final void updateOomAdjLocked() {
    final ActivityRecord TOP_ACT = resumedAppLocked();
    final ProcessRecord TOP_APP = TOP_ACT != null ? TOP_ACT.app : null;
    final long now = SystemClock.uptimeMillis();
    final long nowElapsed = SystemClock.elapsedRealtime();
    final long oldTime = now - ProcessList.MAX_EMPTY_TIME;
    final int N = mLruProcesses.size();

    // Reset state in all uid records.
    for (int i=mActiveUids.size()-1; i>=0; i--) {
        final UidRecord uidRec = mActiveUids.valueAt(i);
        uidRec.reset();
    }

    mStackSupervisor.rankTaskLayersIfNeeded();

    mAdjSeq++;
    mNewNumServiceProcs = 0;
    mNewNumAServiceProcs = 0;

    final int emptyProcessLimit;
    final int cachedProcessLimit;
    if (mProcessLimit <= 0) {
        emptyProcessLimit = cachedProcessLimit = 0;
    } else if (mProcessLimit == 1) {
        emptyProcessLimit = 1;
        cachedProcessLimit = 0;
    } else {
        emptyProcessLimit = ProcessList.computeEmptyProcessLimit(mProcessLimit);
        cachedProcessLimit = mProcessLimit - emptyProcessLimit;
    }

    // Let's determine how many processes we have running vs.
    // how many slots we have for background processes; we may want
    // to put multiple processes in a slot of there are enough of
    // them.
    int numSlots = (ProcessList.CACHED_APP_MAX_ADJ
            - ProcessList.CACHED_APP_MIN_ADJ + 1) / 2;
    int numEmptyProcs = N - mNumNonCachedProcs - mNumCachedHiddenProcs;
    if (numEmptyProcs > cachedProcessLimit) {
        // If there are more empty processes than our limit on cached
        // processes, then use the cached process limit for the factor.
        // This ensures that the really old empty processes get pushed
        // down to the bottom, so if we are running low on memory we will
        // have a better chance at keeping around more cached processes
        // instead of a gazillion empty processes.
        numEmptyProcs = cachedProcessLimit;
    }
    int emptyFactor = numEmptyProcs/numSlots;
    if (emptyFactor < 1) emptyFactor = 1;
    int cachedFactor = (mNumCachedHiddenProcs > 0 ? mNumCachedHiddenProcs : 1)/numSlots;
    if (cachedFactor < 1) cachedFactor = 1;
    int stepCached = 0;
    int stepEmpty = 0;
    int numCached = 0;
    int numEmpty = 0;
    int numTrimming = 0;

    mNumNonCachedProcs = 0;
    mNumCachedHiddenProcs = 0;

    // First update the OOM adjustment for each of the
    // application processes based on their current state.
    int curCachedAdj = ProcessList.CACHED_APP_MIN_ADJ;
    int nextCachedAdj = curCachedAdj+1;
    int curEmptyAdj = ProcessList.CACHED_APP_MIN_ADJ;
    int nextEmptyAdj = curEmptyAdj+2;
    ProcessRecord selectedAppRecord = null;
    long serviceLastActivity = 0;
    int numBServices = 0;
    for (int i=N-1; i>=0; i--) {
        ProcessRecord app = mLruProcesses.get(i);
        if (mEnableBServicePropagation && app.serviceb
                && (app.curAdj == ProcessList.SERVICE_B_ADJ)) { // 寻找最小mMinBServiceAgingTime，意味着最老
            numBServices++;
            for (int s = app.services.size() - 1; s >= 0; s--) {
                ServiceRecord sr = app.services.valueAt(s);
                if (SystemClock.uptimeMillis() - sr.lastActivity
                        < mMinBServiceAgingTime) { // 当前活跃时间最长的service
                    continue;
                }
                if (serviceLastActivity == 0) {
                    serviceLastActivity = sr.lastActivity;
                    selectedAppRecord = app;
                } else if (sr.lastActivity < serviceLastActivity) {
                    serviceLastActivity = sr.lastActivity;
                    selectedAppRecord = app;
                }
            }
        }
        if (!app.killedByAm && app.thread != null) {// 根据缓存、空进程数量设置Adj级别
            app.procStateChanged = false;
            computeOomAdjLocked(app, ProcessList.UNKNOWN_ADJ, TOP_APP, true, now);

            // If we haven't yet assigned the final cached adj
            // to the process, do that now.
            if (app.curAdj >= ProcessList.UNKNOWN_ADJ) { // adj未知
                switch (app.curProcState) {
                    case ActivityManager.PROCESS_STATE_CACHED_ACTIVITY:
                    case ActivityManager.PROCESS_STATE_CACHED_ACTIVITY_CLIENT:
                        // This process is a cached process holding activities...
                        // assign it the next cached value for that type, and then
                        // step that cached level.
                        app.curRawAdj = curCachedAdj;
                        app.curAdj = app.modifyRawOomAdj(curCachedAdj);
  
                        if (curCachedAdj != nextCachedAdj) {
                            stepCached++;
                            if (stepCached >= cachedFactor) {
                                stepCached = 0;
                                curCachedAdj = nextCachedAdj;
                                nextCachedAdj += 2;
                                if (nextCachedAdj > ProcessList.CACHED_APP_MAX_ADJ) {
                                    nextCachedAdj = ProcessList.CACHED_APP_MAX_ADJ;
                                }
                            }
                        }
                        break;
                    default:
                        // For everything else, assign next empty cached process
                        // level and bump that up.  Note that this means that
                        // long-running services that have dropped down to the
                        // cached level will be treated as empty (since their process
                        // state is still as a service), which is what we want.
                        app.curRawAdj = curEmptyAdj;
                        app.curAdj = app.modifyRawOomAdj(curEmptyAdj);
                        if (DEBUG_LRU && false) Slog.d(TAG_LRU, "Assigning empty LRU #" + i
                                + " adj: " + app.curAdj + " (curEmptyAdj=" + curEmptyAdj
                                + ")");
                        if (curEmptyAdj != nextEmptyAdj) {
                            stepEmpty++;
                            if (stepEmpty >= emptyFactor) {
                                stepEmpty = 0;
                                curEmptyAdj = nextEmptyAdj;
                                nextEmptyAdj += 2;
                                if (nextEmptyAdj > ProcessList.CACHED_APP_MAX_ADJ) {
                                    nextEmptyAdj = ProcessList.CACHED_APP_MAX_ADJ;
                                }
                            }
                        }
                        break;
                }
            }

            applyOomAdjLocked(app, true, now, nowElapsed);

            // Count the number of process types.
            switch (app.curProcState) {
                case ActivityManager.PROCESS_STATE_CACHED_ACTIVITY:
                case ActivityManager.PROCESS_STATE_CACHED_ACTIVITY_CLIENT:
                    mNumCachedHiddenProcs++;
                    numCached++;
                    if (numCached > cachedProcessLimit) { // 根据LRU顺序杀死多余的缓存进程
                        app.kill("cached #" + numCached, true);
                    }
                    break;
                case ActivityManager.PROCESS_STATE_CACHED_EMPTY:
                    if (numEmpty > ProcessList.TRIM_EMPTY_APPS
                            && app.lastActivityTime < oldTime) {
                        app.kill("empty for "
                                + ((oldTime + ProcessList.MAX_EMPTY_TIME - app.lastActivityTime)
                                / 1000) + "s", true);
                    } else {
                        numEmpty++;
                        if (numEmpty > emptyProcessLimit) {//根据LRU顺序杀死多余的空进程
                            app.kill("empty #" + numEmpty, true);
                        }
                    }
                    break;
                default:
                    mNumNonCachedProcs++;
                    break;
            }

            if (app.isolated && app.services.size() <= 0) { // 孤立 且 无service
                // If this is an isolated process, and there are no
                // services running in it, then the process is no longer
                // needed.  We agressively kill these because we can by
                // definition not re-use the same process again, and it is
                // good to avoid having whatever code was running in them
                // left sitting around after no longer needed.
                app.kill("isolated not needed", true);
            } else {
                // Keeping this process, update its uid.
                final UidRecord uidRec = app.uidRecord;
                if (uidRec != null && uidRec.curProcState > app.curProcState) {
                    uidRec.curProcState = app.curProcState;
                }
            }

            if (app.curProcState >= ActivityManager.PROCESS_STATE_HOME
                    && !app.killedByAm) {
                numTrimming++;
            }
        }
    }
    if ((numBServices > mBServiceAppThreshold) && (true == mAllowLowerMemLevel)
            && (selectedAppRecord != null)) {
        ProcessList.setOomAdj(selectedAppRecord.pid, selectedAppRecord.info.uid,
                ProcessList.CACHED_APP_MAX_ADJ);
        selectedAppRecord.setAdj = selectedAppRecord.curAdj;
        if (DEBUG_OOM_ADJ) Slog.d(TAG,"app.processName = " + selectedAppRecord.processName
                    + " app.pid = " + selectedAppRecord.pid + " is moved to higher adj");
    }

    mNumServiceProcs = mNewNumServiceProcs;

    // 根据缓存 空进程的数量确定内存收缩等级
    // Now determine the memory trimming level of background processes.
    // Unfortunately we need to start at the back of the list to do this
    // properly.  We only do this if the number of background apps we
    // are managing to keep around is less than half the maximum we desire;
    // if we are keeping a good number around, we'll let them use whatever
    // memory they want.
    final int numCachedAndEmpty = numCached + numEmpty;
    int memFactor;
    if (numCached <= ProcessList.TRIM_CACHED_APPS
            && numEmpty <= ProcessList.TRIM_EMPTY_APPS) {
        if (numCachedAndEmpty <= ProcessList.TRIM_CRITICAL_THRESHOLD) {
            memFactor = ProcessStats.ADJ_MEM_FACTOR_CRITICAL;
        } else if (numCachedAndEmpty <= ProcessList.TRIM_LOW_THRESHOLD) {
            memFactor = ProcessStats.ADJ_MEM_FACTOR_LOW;
        } else {
            memFactor = ProcessStats.ADJ_MEM_FACTOR_MODERATE;
        }
    } else {
        memFactor = ProcessStats.ADJ_MEM_FACTOR_NORMAL;
    }
    // We always allow the memory level to go up (better).  We only allow it to go
    // down if we are in a state where that is allowed, *and* the total number of processes
    // has gone down since last time.
    if (DEBUG_OOM_ADJ) Slog.d(TAG_OOM_ADJ, "oom: memFactor=" + memFactor
            + " last=" + mLastMemoryLevel + " allowLow=" + mAllowLowerMemLevel
            + " numProcs=" + mLruProcesses.size() + " last=" + mLastNumProcesses);
    if (memFactor > mLastMemoryLevel) {
        if (!mAllowLowerMemLevel || mLruProcesses.size() >= mLastNumProcesses) {
            memFactor = mLastMemoryLevel;
            if (DEBUG_OOM_ADJ) Slog.d(TAG_OOM_ADJ, "Keeping last mem factor!");
        }
    }
    if (memFactor != mLastMemoryLevel) {
        EventLogTags.writeAmMemFactor(memFactor, mLastMemoryLevel);
    }
    mLastMemoryLevel = memFactor;
    mLastNumProcesses = mLruProcesses.size();
    boolean allChanged = mProcessStats.setMemFactorLocked(memFactor, !isSleepingLocked(), now);
    final int trackerMemFactor = mProcessStats.getMemFactorLocked();
    if (memFactor != ProcessStats.ADJ_MEM_FACTOR_NORMAL) {
        if (mLowRamStartTime == 0) {
            mLowRamStartTime = now;
        }
        int step = 0;
        int fgTrimLevel;
        switch (memFactor) {
            case ProcessStats.ADJ_MEM_FACTOR_CRITICAL:
                fgTrimLevel = ComponentCallbacks2.TRIM_MEMORY_RUNNING_CRITICAL;
                break;
            case ProcessStats.ADJ_MEM_FACTOR_LOW:
                fgTrimLevel = ComponentCallbacks2.TRIM_MEMORY_RUNNING_LOW;
                break;
            default:
                fgTrimLevel = ComponentCallbacks2.TRIM_MEMORY_RUNNING_MODERATE;
                break;
        }
        int factor = numTrimming/3;
        int minFactor = 2;
        if (mHomeProcess != null) minFactor++;
        if (mPreviousProcess != null) minFactor++;
        if (factor < minFactor) factor = minFactor;
        int curLevel = ComponentCallbacks2.TRIM_MEMORY_COMPLETE;
        for (int i=N-1; i>=0; i--) {
            ProcessRecord app = mLruProcesses.get(i);
            if (allChanged || app.procStateChanged) {
                setProcessTrackerStateLocked(app, trackerMemFactor, now);
                app.procStateChanged = false;
            }
            if (app.curProcState >= ActivityManager.PROCESS_STATE_HOME
                    && !app.killedByAm) {
                if (app.trimMemoryLevel < curLevel && app.thread != null) {
                    try {
                        if (DEBUG_SWITCH || DEBUG_OOM_ADJ) Slog.v(TAG_OOM_ADJ,
                                "Trimming memory of " + app.processName + " to " + curLevel);
                        app.thread.scheduleTrimMemory(curLevel);
                    } catch (RemoteException e) {
                    }
                    if (false) {
                        // For now we won't do this; our memory trimming seems
                        // to be good enough at this point that destroying
                        // activities causes more harm than good.
                        if (curLevel >= ComponentCallbacks2.TRIM_MEMORY_COMPLETE
                                && app != mHomeProcess && app != mPreviousProcess) {
                            // Need to do this on its own message because the stack may not
                            // be in a consistent state at this point.
                            // For these apps we will also finish their activities
                            // to help them free memory.
                            mStackSupervisor.scheduleDestroyAllActivities(app, "trim");
                        }
                    }
                }
                app.trimMemoryLevel = curLevel;
                step++;
                if (step >= factor) {
                    step = 0;
                    switch (curLevel) {
                        case ComponentCallbacks2.TRIM_MEMORY_COMPLETE:
                            curLevel = ComponentCallbacks2.TRIM_MEMORY_MODERATE;
                            break;
                        case ComponentCallbacks2.TRIM_MEMORY_MODERATE:
                            curLevel = ComponentCallbacks2.TRIM_MEMORY_BACKGROUND;
                            break;
                    }
                }
            } else if (app.curProcState == ActivityManager.PROCESS_STATE_HEAVY_WEIGHT) {
                if (app.trimMemoryLevel < ComponentCallbacks2.TRIM_MEMORY_BACKGROUND
                        && app.thread != null) {
                    try {
                        if (DEBUG_SWITCH || DEBUG_OOM_ADJ) Slog.v(TAG_OOM_ADJ,
                                "Trimming memory of heavy-weight " + app.processName
                                + " to " + ComponentCallbacks2.TRIM_MEMORY_BACKGROUND);
                        app.thread.scheduleTrimMemory(
                                ComponentCallbacks2.TRIM_MEMORY_BACKGROUND);
                    } catch (RemoteException e) {
                    }
                }
                app.trimMemoryLevel = ComponentCallbacks2.TRIM_MEMORY_BACKGROUND;
            } else {
                if ((app.curProcState >= ActivityManager.PROCESS_STATE_IMPORTANT_BACKGROUND
                        || app.systemNoUi) && app.pendingUiClean) {
                    // If this application is now in the background and it
                    // had done UI, then give it the special trim level to
                    // have it free UI resources.
                    final int level = ComponentCallbacks2.TRIM_MEMORY_UI_HIDDEN;
                    if (app.trimMemoryLevel < level && app.thread != null) {
                        try {
                            if (DEBUG_SWITCH || DEBUG_OOM_ADJ) Slog.v(TAG_OOM_ADJ,
                                    "Trimming memory of bg-ui " + app.processName
                                    + " to " + level);
                            app.thread.scheduleTrimMemory(level);
                        } catch (RemoteException e) {
                        }
                    }
                    app.pendingUiClean = false;
                }
                if (app.trimMemoryLevel < fgTrimLevel && app.thread != null) {
                    try {
                        if (DEBUG_SWITCH || DEBUG_OOM_ADJ) Slog.v(TAG_OOM_ADJ,
                                "Trimming memory of fg " + app.processName
                                + " to " + fgTrimLevel);
                        app.thread.scheduleTrimMemory(fgTrimLevel);
                    } catch (RemoteException e) {
                    }
                }
                app.trimMemoryLevel = fgTrimLevel;
            }
        }
    } else {
        if (mLowRamStartTime != 0) {
            mLowRamTimeSinceLastIdle += now - mLowRamStartTime;
            mLowRamStartTime = 0;
        }
        for (int i=N-1; i>=0; i--) {
            ProcessRecord app = mLruProcesses.get(i);
            if (allChanged || app.procStateChanged) {
                setProcessTrackerStateLocked(app, trackerMemFactor, now);
                app.procStateChanged = false;
            }
            if ((app.curProcState >= ActivityManager.PROCESS_STATE_IMPORTANT_BACKGROUND
                    || app.systemNoUi) && app.pendingUiClean) {
                if (app.trimMemoryLevel < ComponentCallbacks2.TRIM_MEMORY_UI_HIDDEN
                        && app.thread != null) {
                    try {
                        if (DEBUG_SWITCH || DEBUG_OOM_ADJ) Slog.v(TAG_OOM_ADJ,
                                "Trimming memory of ui hidden " + app.processName
                                + " to " + ComponentCallbacks2.TRIM_MEMORY_UI_HIDDEN);
                        app.thread.scheduleTrimMemory(
                                ComponentCallbacks2.TRIM_MEMORY_UI_HIDDEN);
                    } catch (RemoteException e) {
                    }
                }
                app.pendingUiClean = false;
            }
            app.trimMemoryLevel = 0;
        }
    }

    if (mAlwaysFinishActivities) {
        // Need to do this on its own message because the stack may not
        // be in a consistent state at this point.
        mStackSupervisor.scheduleDestroyAllActivities(null, "always-finish");
    }

    if (allChanged) {
        requestPssAllProcsLocked(now, false, mProcessStats.isMemFactorLowered());
    }

    // Update from any uid changes.
    for (int i=mActiveUids.size()-1; i>=0; i--) {
        final UidRecord uidRec = mActiveUids.valueAt(i);
        int uidChange = UidRecord.CHANGE_PROCSTATE;
        if (uidRec.setProcState != uidRec.curProcState) {
            if (DEBUG_UID_OBSERVERS) Slog.i(TAG_UID_OBSERVERS,
                    "Changes in " + uidRec + ": proc state from " + uidRec.setProcState
                    + " to " + uidRec.curProcState);
            if (ActivityManager.isProcStateBackground(uidRec.curProcState)) {
                if (!ActivityManager.isProcStateBackground(uidRec.setProcState)) {
                    uidRec.lastBackgroundTime = nowElapsed;
                    if (!mHandler.hasMessages(IDLE_UIDS_MSG)) {
                        // Note: the background settle time is in elapsed realtime, while
                        // the handler time base is uptime.  All this means is that we may
                        // stop background uids later than we had intended, but that only
                        // happens because the device was sleeping so we are okay anyway.
                        mHandler.sendEmptyMessageDelayed(IDLE_UIDS_MSG, BACKGROUND_SETTLE_TIME);
                    }
                }
            } else {
                if (uidRec.idle) {
                    uidChange = UidRecord.CHANGE_ACTIVE;
                    uidRec.idle = false;
                }
                uidRec.lastBackgroundTime = 0;
            }
            uidRec.setProcState = uidRec.curProcState;
            enqueueUidChangeLocked(uidRec, -1, uidChange);
            noteUidProcessState(uidRec.uid, uidRec.curProcState);
        }
    }

    if (mProcessStats.shouldWriteNowLocked(now)) {
        mHandler.post(new Runnable() {
            @Override public void run() {
                synchronized (ActivityManagerService.this) {
                    mProcessStats.writeStateAsyncLocked();
                }
            }
        });
    }
}
```
