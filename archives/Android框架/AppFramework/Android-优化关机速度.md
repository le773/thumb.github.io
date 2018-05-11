## 1.1 Android reboot流程
[图片上传失败...(image-6eeea-1526054751305)]

**思路：  **执行到`shutdown::run`时，
a.设置`Flag` `QuickShutdown=true`
b.以广播通知系统一键加速清理进程，此时联网、影响用户体验、最经使用的`TopN`应用均需清理；
###### 一键加速原理参考：
[手机管理应用研究【4】—— 手机加速篇](https://blog.csdn.net/zhgxhuaa/article/details/34097509 "手机管理应用研究")

## 2.1 AMS::startProcessLocked
```
    private final void startProcessLocked(ProcessRecord app, String hostingType,
            String hostingNameStr, String abiOverride, String entryPoint, String[] entryPointArgs) {
    if(QuickShutdown) return;
    ...
}
```

## 2.2 AMS::handleAppDiedLocked
```
/**
 * Main function for removing an existing process from the activity manager
 * as a result of that process going away.  Clears out all connections
 * to the process.
 */
private final void handleAppDiedLocked(ProcessRecord app,
        boolean restarting, boolean allowRestart) {
    if(QuickShutdown) { // 禁止Process重启
        restarting = false;
        allowRestart = false;
    }
...
}
```
###### handleAppDiedLocked相关调用栈
[图片上传失败...(image-f39c4e-1526054751305)]
左边分支`cleanUpApplicationRecordLocked`用于清理死亡进程中运行的四大组件`service`, `BroadcastReceiver`, `ContentProvider`相关信息; 
右边分支`ASS.handleAppDiedLocked`清理死亡进程中运行的`activity`相关信息

参考：
[binder_died](http://gityuan.com/2016/10/02/binder-died/ "binder_died")

## 2.3 AMS::killPackageProcessesLocked
```
private final boolean killPackageProcessesLocked(String packageName, int appId, int userId, int minOomAdj, boolean callerWillRestart, boolean allowRestart,
        boolean doit, boolean evenPersistent, String reason) {
    if (QuickShutdown) {
        callerWillRestart = false;
        allowRestart = false;
        evenPersistent = true; //persistent 进程需清理
        doit = true; //app.removed=true时，添加进程到procs中，procs中的进程将被removeProcessLocked
    }
...
}
```
