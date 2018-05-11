### applyOomAdjLocked 概览
![applyOomAdjLocked](http://upload-images.jianshu.io/upload_images/74811-6bcccfcf5129d5df.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### applyOomAdjLocked 分析
```
private final boolean applyOomAdjLocked(ProcessRecord app, boolean doingAll, long now,
        long nowElapsed) {
    boolean success = true;

    if (app.curRawAdj != app.setRawAdj) {
        String seempStr = "app_uid=" + app.uid
            + ",app_pid=" + app.pid + ",oom_adj=" + app.curAdj
            + ",setAdj=" + app.setAdj + ",hasShownUi=" + (app.hasShownUi ? 1 : 0)
            + ",cached=" + (app.cached ? 1 : 0)
            + ",fA=" + (app.foregroundActivities ? 1 : 0)
            + ",fS=" + (app.foregroundServices ? 1 : 0)
            + ",systemNoUi=" + (app.systemNoUi ? 1 : 0)
            + ",curSchedGroup=" + app.curSchedGroup
            + ",curProcState=" + app.curProcState + ",setProcState=" + app.setProcState
            + ",killed=" + (app.killed ? 1 : 0) + ",killedByAm=" + (app.killedByAm ? 1 : 0)
            + ",debugging=" + (app.debugging ? 1 : 0);
        android.util.SeempLog.record_str(385, seempStr);
        app.setRawAdj = app.curRawAdj;
    }

    int changes = 0;

    if (app.curAdj != app.setAdj) {
        ProcessList.setOomAdj(app.pid, app.info.uid, app.curAdj); //通过socket通知lmkd
        if (DEBUG_SWITCH || DEBUG_OOM_ADJ) Slog.v(TAG_OOM_ADJ,
                "Set " + app.pid + " " + app.processName + " adj " + app.curAdj + ": "
                + app.adjType);
        app.setAdj = app.curAdj;
        app.verifiedAdj = ProcessList.INVALID_ADJ;
    }

    if (app.setSchedGroup != app.curSchedGroup) {
        int oldSchedGroup = app.setSchedGroup;
        app.setSchedGroup = app.curSchedGroup;
        if (DEBUG_SWITCH || DEBUG_OOM_ADJ) Slog.v(TAG_OOM_ADJ,
                "Setting sched group of " + app.processName
                + " to " + app.curSchedGroup);
        if (app.waitingToKill != null && app.curReceiver == null
                && app.setSchedGroup == ProcessList.SCHED_GROUP_BACKGROUND) {
            app.kill(app.waitingToKill, true);
            success = false;
        } else {
            int processGroup;
            switch (app.curSchedGroup) {
                case ProcessList.SCHED_GROUP_BACKGROUND:
                    processGroup = Process.THREAD_GROUP_BG_NONINTERACTIVE;
                    break;
                case ProcessList.SCHED_GROUP_TOP_APP:
                case ProcessList.SCHED_GROUP_TOP_APP_BOUND:
                    processGroup = Process.THREAD_GROUP_TOP_APP;
                    break;
                default:
                    processGroup = Process.THREAD_GROUP_DEFAULT;
                    break;
            }
            long oldId = Binder.clearCallingIdentity();
            try {
                Process.setProcessGroup(app.pid, processGroup);
                if (app.curSchedGroup == ProcessList.SCHED_GROUP_TOP_APP) {
                    // do nothing if we already switched to RT
                    if (oldSchedGroup != ProcessList.SCHED_GROUP_TOP_APP) {
                        // Switch VR thread for app to SCHED_FIFO
                        if (mInVrMode && app.vrThreadTid != 0) {
                            try {
                                Process.setThreadScheduler(app.vrThreadTid,
                                    Process.SCHED_FIFO | Process.SCHED_RESET_ON_FORK, 1);
                            } catch (IllegalArgumentException e) {
                                // thread died, ignore
                            }
                        }
                        if (mUseFifoUiScheduling) {
                            // Switch UI pipeline for app to SCHED_FIFO
                            app.savedPriority = Process.getThreadPriority(app.pid);
                            try {
                                Process.setThreadScheduler(app.pid,
                                    Process.SCHED_FIFO | Process.SCHED_RESET_ON_FORK, 1);
                            } catch (IllegalArgumentException e) {
                                // thread died, ignore
                            }
                            if (app.renderThreadTid != 0) {
                                try {
                                    Process.setThreadScheduler(app.renderThreadTid,
                                        Process.SCHED_FIFO | Process.SCHED_RESET_ON_FORK, 1);
                                } catch (IllegalArgumentException e) {
                                    // thread died, ignore
                                }
                                if (DEBUG_OOM_ADJ) {
                                    Slog.d("UI_FIFO", "Set RenderThread (TID " +
                                        app.renderThreadTid + ") to FIFO");
                                }
                            } else {
                                if (DEBUG_OOM_ADJ) {
                                    Slog.d("UI_FIFO", "Not setting RenderThread TID");
                                }
                            }
                        } else {
                            // Boost priority for top app UI and render threads
                            Process.setThreadPriority(app.pid, -10);
                            if (app.renderThreadTid != 0) {
                                try {
                                    Process.setThreadPriority(app.renderThreadTid, -10);
                                } catch (IllegalArgumentException e) {
                                    // thread died, ignore
                                }
                            }
                        }
                    }
                } else if (oldSchedGroup == ProcessList.SCHED_GROUP_TOP_APP &&
                           app.curSchedGroup != ProcessList.SCHED_GROUP_TOP_APP) {
                    // Reset VR thread to SCHED_OTHER
                    // Safe to do even if we're not in VR mode
                    if (app.vrThreadTid != 0) {
                        Process.setThreadScheduler(app.vrThreadTid, Process.SCHED_OTHER, 0);
                    }
                    if (mUseFifoUiScheduling) {
                        // Reset UI pipeline to SCHED_OTHER
                        Process.setThreadScheduler(app.pid, Process.SCHED_OTHER, 0);
                        Process.setThreadPriority(app.pid, app.savedPriority);
                        if (app.renderThreadTid != 0) {
                            Process.setThreadScheduler(app.renderThreadTid,
                                Process.SCHED_OTHER, 0);
                            Process.setThreadPriority(app.renderThreadTid, -4);
                        }
                    } else {
                        // Reset priority for top app UI and render threads
                        Process.setThreadPriority(app.pid, 0);
                        if (app.renderThreadTid != 0) {
                            Process.setThreadPriority(app.renderThreadTid, 0);
                        }
                    }
                }
            } catch (Exception e) {
                Slog.w(TAG, "Failed setting process group of " + app.pid
                        + " to " + app.curSchedGroup);
                e.printStackTrace();
            } finally {
                Binder.restoreCallingIdentity(oldId);
            }
        }
    }
    if (app.repForegroundActivities != app.foregroundActivities) {
        app.repForegroundActivities = app.foregroundActivities;
        changes |= ProcessChangeItem.CHANGE_ACTIVITIES;
    }
    if (app.repProcState != app.curProcState) {
        app.repProcState = app.curProcState;
        changes |= ProcessChangeItem.CHANGE_PROCESS_STATE;
        if (app.thread != null) {
            try {
                if (false) {
                    //RuntimeException h = new RuntimeException("here");
                    Slog.i(TAG, "Sending new process state " + app.repProcState
                            + " to " + app /*, h*/);
                }
                app.thread.setProcessState(app.repProcState);
            } catch (RemoteException e) {
            }
        }
    }
    if (app.setProcState == ActivityManager.PROCESS_STATE_NONEXISTENT
            || ProcessList.procStatesDifferForMem(app.curProcState, app.setProcState)) {
        if (false && mTestPssMode && app.setProcState >= 0 && app.lastStateTime <= (now-200)) {
            // Experimental code to more aggressively collect pss while
            // running test...  the problem is that this tends to collect
            // the data right when a process is transitioning between process
            // states, which well tend to give noisy data.
            long start = SystemClock.uptimeMillis();
            long pss = Debug.getPss(app.pid, mTmpLong, null);
            recordPssSampleLocked(app, app.curProcState, pss, mTmpLong[0], mTmpLong[1], now);
            mPendingPssProcesses.remove(app);
            Slog.i(TAG, "Recorded pss for " + app + " state " + app.setProcState
                    + " to " + app.curProcState + ": "
                    + (SystemClock.uptimeMillis()-start) + "ms");
        }
        app.lastStateTime = now;
        app.nextPssTime = ProcessList.computeNextPssTime(app.curProcState, true,
                mTestPssMode, isSleepingLocked(), now);
        if (DEBUG_PSS) Slog.d(TAG_PSS, "Process state change from "
                + ProcessList.makeProcStateString(app.setProcState) + " to "
                + ProcessList.makeProcStateString(app.curProcState) + " next pss in "
                + (app.nextPssTime-now) + ": " + app);
    } else {
        if (now > app.nextPssTime || (now > (app.lastPssTime+ProcessList.PSS_MAX_INTERVAL)
                && now > (app.lastStateTime+ProcessList.minTimeFromStateChange(
                mTestPssMode)))) {
            requestPssLocked(app, app.setProcState);
            app.nextPssTime = ProcessList.computeNextPssTime(app.curProcState, false,
                    mTestPssMode, isSleepingLocked(), now);
        } else if (false && DEBUG_PSS) Slog.d(TAG_PSS,
                "Not requesting PSS of " + app + ": next=" + (app.nextPssTime-now));
    }
    if (app.setProcState != app.curProcState) {
        if (DEBUG_SWITCH || DEBUG_OOM_ADJ) Slog.v(TAG_OOM_ADJ,
                "Proc state change of " + app.processName
                        + " to " + app.curProcState);
        boolean setImportant = app.setProcState < ActivityManager.PROCESS_STATE_SERVICE;
        boolean curImportant = app.curProcState < ActivityManager.PROCESS_STATE_SERVICE;
        if (setImportant && !curImportant) {
            // This app is no longer something we consider important enough to allow to
            // use arbitrary amounts of battery power.  Note
            // its current wake lock time to later know to kill it if
            // it is not behaving well.
            BatteryStatsImpl stats = mBatteryStatsService.getActiveStatistics();
            synchronized (stats) {
                app.lastWakeTime = stats.getProcessWakeTime(app.info.uid,
                        app.pid, nowElapsed);
            }
            app.lastCpuTime = app.curCpuTime;

        }
        // Inform UsageStats of important process state change
        // Must be called before updating setProcState
        maybeUpdateUsageStatsLocked(app, nowElapsed);

        app.setProcState = app.curProcState;
        if (app.setProcState >= ActivityManager.PROCESS_STATE_HOME) {
            app.notCachedSinceIdle = false;
        }
        if (!doingAll) {
            setProcessTrackerStateLocked(app, mProcessStats.getMemFactorLocked(), now);
        } else {
            app.procStateChanged = true;
        }
    } else if (app.reportedInteraction && (nowElapsed-app.interactionEventTime)
            > USAGE_STATS_INTERACTION_INTERVAL) {
        // For apps that sit around for a long time in the interactive state, we need
        // to report this at least once a day so they don't go idle.
        maybeUpdateUsageStatsLocked(app, nowElapsed);
    }

    if (changes != 0) {
        if (DEBUG_PROCESS_OBSERVERS) Slog.i(TAG_PROCESS_OBSERVERS,
                "Changes in " + app + ": " + changes);
        int i = mPendingProcessChanges.size()-1;
        ProcessChangeItem item = null;
        while (i >= 0) {
            item = mPendingProcessChanges.get(i);
            if (item.pid == app.pid) {
                if (DEBUG_PROCESS_OBSERVERS) Slog.i(TAG_PROCESS_OBSERVERS,
                        "Re-using existing item: " + item);
                break;
            }
            i--;
        }
        if (i < 0) {
            // No existing item in pending changes; need a new one.
            final int NA = mAvailProcessChanges.size();
            if (NA > 0) {
                item = mAvailProcessChanges.remove(NA-1);
                if (DEBUG_PROCESS_OBSERVERS) Slog.i(TAG_PROCESS_OBSERVERS,
                        "Retrieving available item: " + item);
            } else {
                item = new ProcessChangeItem();
                if (DEBUG_PROCESS_OBSERVERS) Slog.i(TAG_PROCESS_OBSERVERS,
                        "Allocating new item: " + item);
            }
            item.changes = 0;
            item.pid = app.pid;
            item.uid = app.info.uid;
            if (mPendingProcessChanges.size() == 0) {
                if (DEBUG_PROCESS_OBSERVERS) Slog.i(TAG_PROCESS_OBSERVERS,
                        "*** Enqueueing dispatch processes changed!");
                mUiHandler.obtainMessage(DISPATCH_PROCESSES_CHANGED_UI_MSG).sendToTarget();
            }
            mPendingProcessChanges.add(item);
        }
        item.changes |= changes;
        item.processState = app.repProcState;
        item.foregroundActivities = app.repForegroundActivities;
        if (DEBUG_PROCESS_OBSERVERS) Slog.i(TAG_PROCESS_OBSERVERS,
                "Item " + Integer.toHexString(System.identityHashCode(item))
                + " " + app.toShortString() + ": changes=" + item.changes
                + " procState=" + item.processState
                + " foreground=" + item.foregroundActivities
                + " type=" + app.adjType + " source=" + app.adjSource
                + " target=" + app.adjTarget);
    }

    return success;
}
```
