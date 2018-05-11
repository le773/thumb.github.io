### 1.1 `computeOomAdjLocked`计算`Adj`概览图
![computeOomAdjLocked](http://upload-images.jianshu.io/upload_images/74811-586afbdd6cfde93e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 2.1 `computeOomAdjLocked` 计算进程`Adj`分析
```
final ActivityRecord TOP_ACT = resumedAppLocked(); // 2.2
final ProcessRecord TOP_APP = TOP_ACT != null ? TOP_ACT.app : null;

private final int computeOomAdjLocked(ProcessRecord app, int cachedAdj, ProcessRecord TOP_APP,
            boolean doingAll, long now) {
    if (mAdjSeq == app.adjSeq) {
        // This adjustment has already been computed.
        return app.curRawAdj;
    }

    if (app.thread == null) {
        app.adjSeq = mAdjSeq;
        app.curSchedGroup = ProcessList.SCHED_GROUP_BACKGROUND;
        app.curProcState = ActivityManager.PROCESS_STATE_CACHED_EMPTY;
        return (app.curAdj=app.curRawAdj=ProcessList.CACHED_APP_MAX_ADJ);
    }

    app.adjTypeCode = ActivityManager.RunningAppProcessInfo.REASON_UNKNOWN;
    app.adjSource = null;
    app.adjTarget = null;
    app.empty = false;
    app.cached = false;

    final int activitiesSize = app.activities.size();
    // the process running the current foreground app adj 比前台进程高
    if (app.maxAdj <= ProcessList.FOREGROUND_APP_ADJ) {
        // The max adjustment doesn't allow this app to be anything
        // below foreground, so it is not worth doing work for it.
        app.adjType = "fixed";
        app.adjSeq = mAdjSeq;
        app.curRawAdj = app.maxAdj;
        app.foregroundActivities = false;
        app.curSchedGroup = ProcessList.SCHED_GROUP_DEFAULT;
        app.curProcState = ActivityManager.PROCESS_STATE_PERSISTENT;
        // System processes can do UI, and when they do we want to have
        // them trim their memory after the user leaves the UI.  To
        // facilitate this, here we need to determine whether or not it
        // is currently showing UI.
        app.systemNoUi = true;
        if (app == TOP_APP) {
            app.systemNoUi = false;
            app.curSchedGroup = ProcessList.SCHED_GROUP_TOP_APP;
            app.adjType = "pers-top-activity";
        } else if (app.hasTopUi) {
            app.systemNoUi = false;
            app.curSchedGroup = ProcessList.SCHED_GROUP_TOP_APP;
            app.adjType = "pers-top-ui";
        } else if (activitiesSize > 0) {
            for (int j = 0; j < activitiesSize; j++) {
                final ActivityRecord r = app.activities.get(j);
                if (r.visible) {
                    app.systemNoUi = false;
                }
            }
        }
        if (!app.systemNoUi) {// 存在Ui的情况
        // Process is a persistent system process and is doing UI
            app.curProcState = ActivityManager.PROCESS_STATE_PERSISTENT_UI;
        }
        return (app.curAdj=app.maxAdj);
    }

    app.systemNoUi = false;

    final int PROCESS_STATE_CUR_TOP = mTopProcessState;

    // 前台进程top app（launcher）, instrumentationClass 正在执行的广播 service
    // Determine the importance of the process, starting with most
    // important to least, and assign an appropriate OOM adjustment.
    int adj;
    int schedGroup;
    int procState;
    boolean foregroundActivities = false;
    BroadcastQueue queue;
    if (app == TOP_APP) {
        // The last app on the list is the foreground app.
        adj = ProcessList.FOREGROUND_APP_ADJ;
        schedGroup = ProcessList.SCHED_GROUP_TOP_APP;
        app.adjType = "top-activity";
        foregroundActivities = true;
        procState = PROCESS_STATE_CUR_TOP;
        if(app == mHomeProcess) {
            mHomeKilled = false;
            mHomeProcessName = mHomeProcess.processName;
        }
    } else if (app.instrumentationClass != null) {
        // Don't want to kill running instrumentation.
        adj = ProcessList.FOREGROUND_APP_ADJ;
        schedGroup = ProcessList.SCHED_GROUP_DEFAULT;
        app.adjType = "instrumentation";
        procState = ActivityManager.PROCESS_STATE_FOREGROUND_SERVICE;
    } else if ((queue = isReceivingBroadcast(app)) != null) {
        // An app that is currently receiving a broadcast also
        // counts as being in the foreground for OOM killer purposes.
        // It's placed in a sched group based on the nature of the
        // broadcast as reflected by which queue it's active in.
        adj = ProcessList.FOREGROUND_APP_ADJ;
        schedGroup = (queue == mFgBroadcastQueue)
                ? ProcessList.SCHED_GROUP_DEFAULT : ProcessList.SCHED_GROUP_BACKGROUND;
        app.adjType = "broadcast";
        procState = ActivityManager.PROCESS_STATE_RECEIVER;
    } else if (app.executingServices.size() > 0) {
        // An app that is currently executing a service callback also
        // counts as being in the foreground.
        adj = ProcessList.FOREGROUND_APP_ADJ;
        schedGroup = app.execServicesFg ?
                ProcessList.SCHED_GROUP_DEFAULT : ProcessList.SCHED_GROUP_BACKGROUND;
        app.adjType = "exec-service";
        procState = ActivityManager.PROCESS_STATE_SERVICE;
        //Slog.i(TAG, "EXEC " + (app.execServicesFg ? "FG" : "BG") + ": " + app);
    } else {
        // As far as we know the process is empty.  We may change our mind later.
        schedGroup = ProcessList.SCHED_GROUP_BACKGROUND;
        // At this point we don't actually know the adjustment.  Use the cached adj
        // value that the caller wants us to.
        adj = cachedAdj;
        procState = ActivityManager.PROCESS_STATE_CACHED_EMPTY;
        app.cached = true;
        app.empty = true;
        app.adjType = "cch-empty";

        if (mHomeKilled && app.processName.equals(mHomeProcessName)) {
            adj = ProcessList.PERSISTENT_PROC_ADJ;
            schedGroup = Process.THREAD_GROUP_DEFAULT;
            app.cached = false;
            app.empty = false;
            app.adjType = "top-activity";
        }
    }

    // Examine all activities if not already foreground. 非前台Activity
    if (!foregroundActivities && activitiesSize > 0) {
        int minLayer = ProcessList.VISIBLE_APP_LAYER_MAX;
        for (int j = 0; j < activitiesSize; j++) {
            final ActivityRecord r = app.activities.get(j);
            if (r.app != app) {
                Log.e(TAG, "Found activity " + r + " in proc activity list using " + r.app
                        + " instead of expected " + app);
                if (r.app == null || (r.app.uid == app.uid)) {
                    // Only fix things up when they look sane
                    r.app = app;
                } else {
                    continue;
                }
            }
            if (r.visible) { // Activity可见
                // App has a visible activity; only upgrade adjustment.
                if (adj > ProcessList.VISIBLE_APP_ADJ) {
                    adj = ProcessList.VISIBLE_APP_ADJ;
                    app.adjType = "visible";
                }
                if (procState > PROCESS_STATE_CUR_TOP) {
                    procState = PROCESS_STATE_CUR_TOP;
                }
                schedGroup = ProcessList.SCHED_GROUP_DEFAULT;
                app.cached = false;
                app.empty = false;
                foregroundActivities = true;
                if (r.task != null && minLayer > 0) {
                    final int layer = r.task.mLayerRank;
                    if (layer >= 0 && minLayer > layer) {
                        minLayer = layer;
                    }
                }
                break;
            } else if (r.state == ActivityState.PAUSING || r.state == ActivityState.PAUSED) { // 处于暂停或已经暂停
                if (adj > ProcessList.PERCEPTIBLE_APP_ADJ) {
                    adj = ProcessList.PERCEPTIBLE_APP_ADJ;
                    app.adjType = "pausing";
                }
                if (procState > PROCESS_STATE_CUR_TOP) {
                    procState = PROCESS_STATE_CUR_TOP;
                }
                schedGroup = ProcessList.SCHED_GROUP_DEFAULT;
                app.cached = false;
                app.empty = false;
                foregroundActivities = true;
            } else if (r.state == ActivityState.STOPPING) { 
                if (adj > ProcessList.PERCEPTIBLE_APP_ADJ) {
                    adj = ProcessList.PERCEPTIBLE_APP_ADJ;
                    app.adjType = "stopping";
                }
                // For the process state, we will at this point consider the
                // process to be cached.  It will be cached either as an activity
                // or empty depending on whether the activity is finishing.  We do
                // this so that we can treat the process as cached for purposes of
                // memory trimming (determing current memory level, trim command to
                // send to process) since there can be an arbitrary number of stopping
                // processes and they should soon all go into the cached state.
                if (!r.finishing) {
                    if (procState > ActivityManager.PROCESS_STATE_LAST_ACTIVITY) {
                        procState = ActivityManager.PROCESS_STATE_LAST_ACTIVITY;
                    }
                }
                app.cached = false;
                app.empty = false;
                foregroundActivities = true;
            } else {
                if (procState > ActivityManager.PROCESS_STATE_CACHED_ACTIVITY) {
                    procState = ActivityManager.PROCESS_STATE_CACHED_ACTIVITY;
                    app.adjType = "cch-act";
                }
            }
        }
        if (adj == ProcessList.VISIBLE_APP_ADJ) {
            adj += minLayer;
        }
    }

    if (adj > ProcessList.PERCEPTIBLE_APP_ADJ
            || procState > ActivityManager.PROCESS_STATE_FOREGROUND_SERVICE) {
        if (app.foregroundServices) { // 前台service
            // The user is aware of this app, so make it visible.
            adj = ProcessList.PERCEPTIBLE_APP_ADJ;
            procState = ActivityManager.PROCESS_STATE_FOREGROUND_SERVICE;
            app.cached = false;
            app.adjType = "fg-service";
            schedGroup = ProcessList.SCHED_GROUP_DEFAULT;
        } else if (app.forcingToForeground != null) {//
            // The user is aware of this app, so make it visible.
            adj = ProcessList.PERCEPTIBLE_APP_ADJ;
            procState = ActivityManager.PROCESS_STATE_IMPORTANT_FOREGROUND;
            app.cached = false;
            app.adjType = "force-fg";
            app.adjSource = app.forcingToForeground;
            schedGroup = ProcessList.SCHED_GROUP_DEFAULT;
        }
    }

    if (app == mHeavyWeightProcess) { // 解析init.rc启动的进程 AudioFlinger等
        if (adj > ProcessList.HEAVY_WEIGHT_APP_ADJ) {
            // We don't want to kill the current heavy-weight process.
            adj = ProcessList.HEAVY_WEIGHT_APP_ADJ;
            schedGroup = ProcessList.SCHED_GROUP_BACKGROUND;
            app.cached = false;
            app.adjType = "heavy";
        }
        if (procState > ActivityManager.PROCESS_STATE_HEAVY_WEIGHT) {
            procState = ActivityManager.PROCESS_STATE_HEAVY_WEIGHT;
        }
    }

    if (app == mHomeProcess) { 
    // This is the process holding the activity the user last visited that is in a different process from the one they are currently in.
        if (adj > ProcessList.HOME_APP_ADJ) {
            // This process is hosting what we currently consider to be the
            // home app, so we don't want to let it go into the background.
            adj = ProcessList.HOME_APP_ADJ;
            schedGroup = ProcessList.SCHED_GROUP_BACKGROUND;
            app.cached = false;
            app.adjType = "home";
        }
        if (procState > ActivityManager.PROCESS_STATE_HOME) {
            procState = ActivityManager.PROCESS_STATE_HOME;
        }
    }
    // The time at which the previous process was last visible 前一个可见
    if (app == mPreviousProcess && app.activities.size() > 0) {
        if (adj > ProcessList.PREVIOUS_APP_ADJ) { // qian
            // This was the previous process that showed UI to the user.
            // We want to try to keep it around more aggressively, to give
            // a good experience around switching between two apps.
            adj = ProcessList.PREVIOUS_APP_ADJ;
            schedGroup = ProcessList.SCHED_GROUP_BACKGROUND;
            app.cached = false;
            app.adjType = "previous";
        }
        if (procState > ActivityManager.PROCESS_STATE_LAST_ACTIVITY) {
            procState = ActivityManager.PROCESS_STATE_LAST_ACTIVITY;
        }
    }

    // By default, we use the computed adjustment.  It may be changed if
    // there are applications dependent on our services or providers, but
    // this gives us a baseline and makes sure we don't get into an
    // infinite recursion.
    app.adjSeq = mAdjSeq;
    app.curRawAdj = adj;
    app.hasStartedServices = false;

    //备份进程
    if (mBackupTarget != null && app == mBackupTarget.app) {
        // If possible we want to avoid killing apps while they're being backed up
        if (adj > ProcessList.BACKUP_APP_ADJ) {
            if (DEBUG_BACKUP) Slog.v(TAG_BACKUP, "oom BACKUP_APP_ADJ for " + app);
            adj = ProcessList.BACKUP_APP_ADJ;
            if (procState > ActivityManager.PROCESS_STATE_IMPORTANT_BACKGROUND) {
                procState = ActivityManager.PROCESS_STATE_IMPORTANT_BACKGROUND;
            }
            app.adjType = "backup";
            app.cached = false;
        }
        if (procState > ActivityManager.PROCESS_STATE_BACKUP) {
            procState = ActivityManager.PROCESS_STATE_BACKUP;
        }
    }
    ...
    // service contentprovider相关的adj计算
}
```

### 2.2 ActivityStackSupervisor::resumedAppLocked 获取Top_app
```
ActivityRecord resumedAppLocked() {
    ActivityStack stack = mFocusedStack; // 焦点ActivityStack
    if (stack == null) {
        return null;
    }
    ActivityRecord resumedActivity = stack.mResumedActivity; // 当前正在运行的Activity
    if (resumedActivity == null || resumedActivity.app == null) {
        resumedActivity = stack.mPausingActivity;// 当前正在暂停的Activity
        if (resumedActivity == null || resumedActivity.app == null) {
            resumedActivity = stack.topRunningActivityLocked(); // 遍历ArrayList<TaskRecord>查询
        }
    }
    return resumedActivity;
}
```
