##### 通知系统进行内存清理
```
IDLE_NOW_MSG
AMS::forceStopPackageLocked
    // native kill 
    ActivityStackSupervisor::scheduleIdleLocked

IDLE_TIMEOUT_MSG
completeResumeLocked
    scheduleIdleTimeoutLocked
```
##### activityIdle调用栈
```
activityIdleInternalLocked
    AMS::scheduleAppGcsLocked // 1. ActivityIdle 2. 串行广播处理完成
        mProcessesToGc// ProcessRecord
        Message// GC_BACKGROUND_PROCESSES_MSG
        performAppGcsIfAppropriateLocked
            if canGcNow
                performAppGcsLocked
                    performAppGcLocked // 
                        AppThread::scheduleLowMemory
            else
                scheduleAppGcsLocked   // Still not idle, wait some more.
    stops // ArrayList<ActivityRecord>
    if r.finishing
        finishCurrentActivityLocked
    else
        stopActivityLocked
    mFinishingActivities // ActivityRecord
    destroyActivityLocked
```
#####  instrumentationClass 清理调用栈
```
AppDeathRecipient::binderDied
    appDiedLocked // 限制引用被kill后，重新启动allowRestart
        handleAppDiedLocked // 如果引用在黑名单，则setPackageStoppedState 
            cleanUpApplicationRecordLocked
                // 正在启动的provider需要重启
            finishInstrumentationLocked
                    app.instrumentationClass = null
        willStart // restart
            addAppLocked
```
## trimApplications 触发时机
```
activityStopped
finishReceiver
    trimApplications
        mRemovedProcesses// empty process
        // crash 的进程
        // 5秒内没有响应并被用户选在强制关闭的进程
        //应用开发这调用 killBackgroundProcess 想要杀死的进程
        if pid > 0
            app.kill
        else
            AppThread::onTerminate
            Looper.quit
            
        cleanUpApplicationRecordLocked
        addAppLocked // persistent
```

## 各种关闭当前Activity引起的内存回收
![各种关闭当前Activity引起的内存回收](http://upload-images.jianshu.io/upload_images/74811-be69360df7e64df4.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
