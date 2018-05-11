## 1.1 ActivityManager::forceStopPackage
```
/**
 * Have the system perform a force stop of everything associated with
 * the given application package.  All processes that share its uid // 同UID
 * will be killed, all services it has running stopped, all activities // service activity
 * removed, etc.  In addition, a {@link Intent#ACTION_PACKAGE_RESTARTED} //ACTION_PACKAGE_RESTARTED广播
 * broadcast will be sent, so that any of its registered alarms can // alarm notifications
 * be stopped, notifications removed, etc.
 *
 * <p>You must hold the permission
 * {@link android.Manifest.permission#FORCE_STOP_PACKAGES} to be able to
 * call this method.
 *
 * @param packageName The name of the package to be stopped.
 * @param userId The user for which the running package is to be stopped.
 *
 * @hide This is not available to third party applications due to
 * it allowing them to break other applications by stopping their
 * services, removing their alarms, etc.
 */
public void forceStopPackageAsUser(String packageName, int userId) {
    try {
        ActivityManagerNative.getDefault().forceStopPackage(packageName, userId);
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
```
调用此方法，（`FORCE_STOP_PACKAGES`权限需要系统签名）
1. 系统会强制停止与指定包关联的所有事情，
2. 将会杀死使用同一个`uid`的所有进程，停止所有服务，移除所有`activity`；
3. 所有注册的定时器和通知也会移除；
4. 还可以达到禁止开机自启动和后台自启动的目的；

## 1.2 PackageManagerService::setPackageStoppedState 设置包停止状态

```
/**
 * Set whether the given package should be considered stopped, making
 * it not visible to implicit intents that filter out stopped packages.
 */
void setPackageStoppedState(String packageName, boolean stopped, int userId);
```

## 1.3 Settings::setPackageStoppedStateLPw
```
boolean setPackageStoppedStateLPw(PackageManagerService pm, String packageName,
        boolean stopped, boolean allowedByPermission, int uid, int userId) {
    int appId = UserHandle.getAppId(uid);
    final PackageSetting pkgSetting = mPackages.get(packageName);
    ... // 参数检查
    if (pkgSetting.getStopped(userId) != stopped) {
        pkgSetting.setStopped(stopped, userId);
        // pkgSetting.pkg.mSetStopped = stopped;
        if (pkgSetting.getNotLaunched(userId)) {
            if (pkgSetting.installerPackageName != null) {
                pm.notifyFirstLaunch(pkgSetting.name, pkgSetting.installerPackageName, userId);
            }
            pkgSetting.setNotLaunched(false, userId);
        }
        return true;
    }
    return false;
}
```

## 1.4 PackageUserState代表一个应用状态
```
// PackageUserState
public PackageUserState(PackageUserState o) {
    ceDataInode = o.ceDataInode;
    installed = o.installed;
    stopped = o.stopped; // force-stopped
    notLaunched = o.notLaunched;
    hidden = o.hidden;
    suspended = o.suspended;
    blockUninstall = o.blockUninstall;
    enabled = o.enabled;
    lastDisableAppCaller = o.lastDisableAppCaller;
    domainVerificationStatus = o.domainVerificationStatus;
    appLinkGeneration = o.appLinkGeneration;
    disabledComponents = ArrayUtils.cloneOrNull(o.disabledComponents);
    enabledComponents = ArrayUtils.cloneOrNull(o.enabledComponents);
}
```


## 1.5 Stop相关Flag
```
/**
 * If set, this intent will not match any components in packages that
 * are currently stopped.  If this is not set, then the default behavior
 * is to include such applications in the result.
 */
public static final int FLAG_EXCLUDE_STOPPED_PACKAGES = 0x00000010;
/**
 * If set, this intent will always match any components in packages that
 * are currently stopped.  This is the default behavior when
 * {@link #FLAG_EXCLUDE_STOPPED_PACKAGES} is not set.  If both of these
 * flags are set, this one wins (it allows overriding of exclude for
 * places where the framework may automatically set the exclude flag).
 */
public static final int FLAG_INCLUDE_STOPPED_PACKAGES = 0x00000020;
```
广播默认添加`FLAG_EXCLUDE_STOPPED_PACKAGES`

## 1.6 关于forcestop中的Alarm
`AlarmManagerService`处理`Intent#ACTION_PACKAGE_RESTARTED`广播
```
class UninstallReceiver extends BroadcastReceiver {
    public UninstallReceiver() {
        IntentFilter filter = new IntentFilter();
        filter.addAction(Intent.ACTION_PACKAGE_REMOVED);
        filter.addAction(Intent.ACTION_PACKAGE_RESTARTED);
        filter.addAction(Intent.ACTION_QUERY_PACKAGE_RESTART);
        filter.addDataScheme("package");
        getContext().registerReceiver(this, filter);
         // Register for events related to sdcard installation.
        IntentFilter sdFilter = new IntentFilter();
        sdFilter.addAction(Intent.ACTION_EXTERNAL_APPLICATIONS_UNAVAILABLE);
        sdFilter.addAction(Intent.ACTION_USER_STOPPED);
        sdFilter.addAction(Intent.ACTION_UID_REMOVED);
        getContext().registerReceiver(this, sdFilter);
    }

    @Override
    public void onReceive(Context context, Intent intent) {
        ...
        if (pkgList != null && (pkgList.length > 0)) {
            for (String pkg : pkgList) {
                removeLocked(pkg); // 移除闹钟相关
                mPriorities.remove(pkg);
                for (int i=mBroadcastStats.size()-1; i>=0; i--) {
                    ArrayMap<String, BroadcastStats> uidStats = mBroadcastStats.valueAt(i);
                    if (uidStats.remove(pkg) != null) {
                        if (uidStats.size() <= 0) {
                            mBroadcastStats.removeAt(i);
                        }
                    }
                }
            }
        }
    }
}

```
`AlarmManagerService`里面会接收这个广播，判断这个广播的 `package name`，如果在设置过了的 `alarm list` 中，就会把对应的 `alarm` 给删掉（`AlarmManagerSerivce` 用了几个 `ArrayList` 来保存不同 `type` 的 `alarm`）。


## 2.1 killBackgroundProcesses
```
/**
 * Have the system immediately kill all background processes associated
 * with the given package.  This is the same as the kernel killing those // 与在内核用kill -9
 * processes to reclaim memory; the system will take care of restarting // 进程可能重启
 * these processes in the future as needed.
 *
 * <p>You must hold the permission
 * {@link android.Manifest.permission#KILL_BACKGROUND_PROCESSES} to be able to
 * call this method.
 *
 * @param packageName The name of the package whose processes are to
 * be killed.
 */
public void killBackgroundProcesses(String packageName) {
    try {
        ActivityManagerNative.getDefault().killBackgroundProcesses(packageName,
                UserHandle.myUserId());
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
```

1. `ActivityManager`的`killBackgroundProcesses`方法，可以立即杀死与指定包相关联的所有后台进程，这与内核杀死那些进程回收内存是一样的(`binderDied` 处理)，但这些进程如果在将来某一时刻需要使用，会重新启动。
2. 该方法需要权限`android.permission.KILL_BACKGROUND_PROCESSES`。



## 2.2 Process::killProcessQuiet
```
/**
 * @hide
 * Private impl for avoiding a log message...  DO NOT USE without doing
 * your own log, or the Android Illuminati will find you some night and
 * beat you up.
 */
public static final void killProcessQuiet(int pid) {
    sendSignalQuiet(pid, SIGNAL_KILL);
}
```

## 2.3 Process::killProcess
```
/**
 * Kill the process with the given PID.
 * Note that, though this API allows us to request to
 * kill any process based on its PID, the kernel will
 * still impose standard restrictions on which PIDs you // 把标准的限制加于想要kill的pid
 * are actually able to kill.  Typically(通常) this means only
 * the process running the caller's packages/application 调用者进程
 * and any additional processes created by that app; packages
 * sharing a common UID will also be able to kill each // 同一个Uid下
 * other's processes.
 */
public static final void killProcess(int pid) {
    sendSignal(pid, SIGNAL_KILL);
}
```

## 2.4 Process::sendSignal
```
void android_os_Process_sendSignal(JNIEnv* env, jobject clazz, jint pid, jint sig)
{
    if (pid > 0) {
        ALOGI("Sending signal. PID: %" PRId32 " SIG: %" PRId32, pid, sig); // 区别
        kill(pid, sig);
    }
}

void android_os_Process_sendSignalQuiet(JNIEnv* env, jobject clazz, jint pid, jint sig)
{
    if (pid > 0) {
        kill(pid, sig);
    }
}
```
`sendSignal`和`sendSignalQuiet`的唯一区别就是在于是否有`ALOGI()`这一行代码。
`Process.kill`通过内核`kill -9`杀死进程，通过该进程的`IBinder.DeathRecipient`进行进程信息清理

###### 参考
[关于Android中App的停止状态](http://droidyue.com/blog/2014/07/14/look-inside-android-package-stop-state-since-honeycomb-mr1/ "关于Android中App的停止状态")
