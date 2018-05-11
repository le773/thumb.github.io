## 1. PackageMS相关框架
![PackageMS相关框架](http://upload-images.jianshu.io/upload_images/74811-bde424c22562e9d7.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

----------

## 2.1 调用PackageMS解析
```
根据Intent找到apk路径，调用PackageUtil.getPackageInfo 解析
PackageInstallerActivity.onCreate
    PackageUtil.getPackageInfo(sourceFile)
        parser.parseMonolithicPackage
            parseBaseApk // 解析apk AndroidManifest.xml
    PackageParse.generatePackageInfo  // 转化为PackageInfo 对象
    ...
    initiateInstall // 判断App是否已经安装过 ApplicationInfo.FLAG_INSTALLED
 	  startInstallConfirm // 对话框，是否安装及app获得的那些权限
    ...
    startInstall 
        startActivity(newIntent) // 传递给PackageMS
```

## 2.2 PackageInstallerActivity::onClick
```
public void onClick(View v) {
    if (v == mOk) {
        if (mOkCanInstall || mScrollView == null) {
            if (mSessionId != -1) {
                mInstaller.setPermissionsResult(mSessionId, true);
                clearCachedApkIfNeededAndFinish();
            } else {
                startInstall(); // 2.3
            }
        } else {
            mScrollView.pageScroll(View.FOCUS_DOWN);
        }
    } else if (v == mCancel) {
        // Cancel and finish
        setResult(RESULT_CANCELED);
        if (mSessionId != -1) {
            mInstaller.setPermissionsResult(mSessionId, false);
        }
        clearCachedApkIfNeededAndFinish();
    }
}
```

## 2.3 PackageInstallerActivity::startInstall
```
private void startInstall() {
    Intent newIntent = new Intent();
    newIntent.putExtra(PackageUtil.INTENT_ATTR_APPLICATION_INFO,
            mPkgInfo.applicationInfo);
    newIntent.setData(mPackageURI);
    newIntent.setClass(this, InstallAppProgress.class);
    ...
    startActivity(newIntent);
    finish();
}
```

## 2.4 WearPackageInstallerService::installPackage
```
public int onStartCommand(Intent intent, int flags, int startId) {
    ...
    WearPackageArgs.setStartId(intentBundle, startId);
    WearPackageArgs.setPackageName(intentBundle, packageName);
    if (Intent.ACTION_INSTALL_PACKAGE.equals(intent.getAction())) {
        Message msg = mServiceHandler.obtainMessage(START_INSTALL);
        msg.setData(intentBundle);
        mServiceHandler.sendMessage(msg);
    } else if (Intent.ACTION_UNINSTALL_PACKAGE.equals(intent.getAction())) {
        Message msg = mServiceHandler.obtainMessage(START_UNINSTALL);
        msg.setData(intentBundle);
        mServiceHandler.sendMessage(msg);
    }
    return START_NOT_STICKY;
}
```

## 2.5 WearPackageInstallerService::installPackage
```
private void installPackage(Bundle argsBundle) {
    // Finally install the package.
    pm.installPackage(Uri.fromFile(tempFile),
            new PackageInstallObserver(this, lock, startId, packageName),
                installFlags, packageName); // 3.1
}
```

----------

## 3.1 PackageMS::installPackageAsUser
```
public void installPackageAsUser(...) {
    // 权限检查 
    ...
    final Message msg = mHandler.obtainMessage(INIT_COPY);
    final VerificationInfo verificationInfo = new VerificationInfo(
            null /*originatingUri*/, null /*referrer*/, -1 /*originatingUid*/, callingUid);
    final InstallParams params = new InstallParams(origin, null /*moveInfo*/, observer,
            installFlags, installerPackageName, null /*volumeUuid*/, verificationInfo, user,
            null /*packageAbiOverride*/, null /*grantedPermissions*/,
            null /*certificates*/);
    params.setTraceMethod("installAsUser").setTraceCookie(System.identityHashCode(params));
    msg.obj = params;

    mHandler.sendMessage(msg);// 3.2
}
```

## 3.1.2 权限检查逻辑
![installPackageAsUser 权限检查相关](http://upload-images.jianshu.io/upload_images/74811-79f7099339a5b897.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 3.2 PackageMS::PackageHandler::doHandleMessage
```
void doHandleMessage(Message msg) {
    switch (msg.what) {
        case INIT_COPY: {
            HandlerParams params = (HandlerParams) msg.obj;
            int idx = mPendingInstalls.size();
            // If a bind was already initiated we dont really
            // need to do anything. The pending install
            // will be processed later on.
            if (!mBound) {
                // If this is the only one pending we might
                // have to bind to the service again.
                if (!connectToService()) {  // 3.3
                    params.serviceError();
                    return;
                } else {
                    // Once we bind to the service, the first
                    // pending request will be processed.
                    mPendingInstalls.add(idx, params); // 连接成功，则添加到mPendingInstalls
                }
            } else {
                mPendingInstalls.add(idx, params);
                // Already bound to the service. Just make
                // sure we trigger off processing the first request.
                if (idx == 0) {
                    mHandler.sendEmptyMessage(MCS_BOUND); // 3.7
                }
            }
            break;
        }
    }
}
```

## 3.3 PackageMS::PackageHandler::connectToService
```
static final ComponentName DEFAULT_CONTAINER_COMPONENT = new ComponentName(DEFAULT_CONTAINER_PACKAGE,
    "com.android.defcontainer.DefaultContainerService");
final private DefaultContainerConnection mDefContainerConn = new DefaultContainerConnection(); // 3.4
private boolean connectToService() {
    Intent service = new Intent().setComponent(DEFAULT_CONTAINER_COMPONENT);
    Process.setThreadPriority(Process.THREAD_PRIORITY_DEFAULT);
    if (mContext.bindServiceAsUser(service, mDefContainerConn,
            Context.BIND_AUTO_CREATE, UserHandle.SYSTEM)) {
        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
        mBound = true;

        final long DEFCONTAINER_CHECK = 1 * 1000;
        final Message msg = mHandler.obtainMessage(MCS_CHECK);
        mHandler.sendMessageDelayed(msg, DEFCONTAINER_CHECK); // 3.5

        return true;
    }
    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
    return false;
}
```
## 3.4 PackageMS::DefaultContainerConnection 
```
    class DefaultContainerConnection implements ServiceConnection {
        public void onServiceConnected(ComponentName name, IBinder service) {
            mServiceConnected = true;
            mServiceCheck = 2111;
            IMediaContainerService imcs =
                IMediaContainerService.Stub.asInterface(service); // 3.4.2
            mHandler.sendMessage(mHandler.obtainMessage(MCS_BOUND, imcs));
        }

        public void onServiceDisconnected(ComponentName name) {
            mServiceConnected = false;
        }
    }
```
## 3.4.2 DefaultContainerService::mBinder
```
private IMediaContainerService.Stub mBinder = new IMediaContainerService.Stub() {
        public int copyPackage(String packagePath, IParcelFileDescriptorFactory target) {
            if (packagePath == null || target == null) {
                return PackageManager.INSTALL_FAILED_INVALID_URI;
            }

            PackageLite pkg = null;
            try {
                final File packageFile = new File(packagePath);
                pkg = PackageParser.parsePackageLite(packageFile, 0);
                return copyPackageInner(pkg, target); // copy相关调用栈 DefaultContainerService::copyFile  --> FileUtils.copyFile
            } catch (PackageParserException | IOException | RemoteException e) {
                Slog.w(TAG, "Failed to copy package at " + packagePath + ": " + e);
                return PackageManager.INSTALL_FAILED_INSUFFICIENT_STORAGE;
            }
        }
}
```
## 3.5 PackageMS::PackageHandler::doHandleMessage
```
void doHandleMessage(Message msg) {
    case MCS_CHECK: {
        mServiceCheck ++;
        if (!mServiceConnected && mServiceCheck <= 3) { // 共尝试连接4次
            connectToService();
        }
        break;
    }
}
```

## 3.6 INIT_COPY逻辑
![INIT_COPY逻辑](http://upload-images.jianshu.io/upload_images/74811-4fef24a4f81e9e31.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 3.7 PackageMS::PackageHandler::doHandleMessage
```
void doHandleMessage(Message msg) {
...
case MCS_BOUND: {
   if (msg.obj != null) {
        mContainerService = (IMediaContainerService) msg.obj;
    }
    if (mContainerService == null) {
        if (!mBound) {
            for (HandlerParams params : mPendingInstalls) {
                // Indicate service bind error
                params.serviceError(); // 未绑定处理
                return;
            }
            mPendingInstalls.clear();
        } 
    } else if (mPendingInstalls.size() > 0) {
        HandlerParams params = mPendingInstalls.get(0);
        if (params != null) { 
            boolean startCopy = params.startCopy(); // 3.8 
            if(startCopy) {
                if (mPendingInstalls.size() > 0) {
                    mPendingInstalls.remove(0);
                }
                if (mPendingInstalls.size() == 0) {
                    if (mBound) {
                        removeMessages(MCS_UNBIND);
                        Message ubmsg = obtainMessage(MCS_UNBIND); // 3.9
                        // Unbind after a little delay, to avoid
                        // continual thrashing.
                        sendMessageDelayed(ubmsg, 10000);
                    }
                } else {
                    mHandler.sendEmptyMessage(MCS_BOUND); // 3.10
                }
            }
        }
    }
}
```

## 3.8 PackageMS::HandlerParams::startCopy
```
final boolean startCopy() {
    boolean res;
    try {
        if (++++mRetries > MAX_RETRIES) {
            Slog.w(TAG, "Failed to invoke remote methods on default container service. Giving up");
            mHandler.sendEmptyMessage(MCS_GIVE_UP);
            handleServiceError();
            return false;
        } else {
            handleStartCopy(); // 3.11
            res = true;
        }
    } catch (RemoteException e) {
        if (DEBUG_INSTALL) Slog.i(TAG, "Posting install MCS_RECONNECT");
        mHandler.sendEmptyMessage(MCS_RECONNECT);
        res = false;
    }
    handleReturnCode(); // 3.12
    return res;
}
```

## 3.9 PackageMS::PackageHandler::doHandleMessage
```
case MCS_UNBIND: {
    if (mPendingInstalls.size() == 0 && mPendingVerification.size() == 0) {
        if (mBound) {
            if (DEBUG_INSTALL) Slog.i(TAG, "calling disconnectService()");

            disconnectService();
        }
    } else if (mPendingInstalls.size() > 0) {
        // There are more pending requests in queue.
        // Just post MCS_BOUND message to trigger processing
        // of next pending install.
        mHandler.sendEmptyMessage(MCS_BOUND);
    }

    break;
}
```
## 3.10 PackageMS::PackageHandler::doHandleMessage
```
case MCS_GIVE_UP: {
    if (DEBUG_INSTALL) Slog.i(TAG, "mcs_giveup too many retries");
    HandlerParams params = mPendingInstalls.remove(0);
    Trace.asyncTraceEnd(TRACE_TAG_PACKAGE_MANAGER, "queueInstall",
            System.identityHashCode(params));
    break;
}
```

## 3.11 PackageMS::InstallParams::handleStartCopy
```
public void handleStartCopy() throws RemoteException {
    int ret = PackageManager.INSTALL_SUCCEEDED;
    ...
    /*
     * If we have too little free space, try to free cache
     * before giving up.
     */
    mInstaller.freeCache(null, sizeBytes + lowThreshold);
    pkgLite = mContainerService.getMinimalPackageInfo(origin.resolvedPath,
            installFlags, packageAbiOverride);
    ...

    if (ret == PackageManager.INSTALL_SUCCEEDED) {
        ...
        if (selectInsLoc && volumeUuid == null) {
            String volumeuuid = null;
            final long sizeBytes = mContainerService.calculateInstalledSize(
                       origin.resolvedPath, isForwardLocked(), packageAbiOverride); // 安装大小
            volumeuuid = PackageHelper.resolveInstallVolume(mContext,
                       installerPackageName, pkgLite.installLocation, sizeBytes);
            ...
        }
    }
    ...
    final InstallArgs args = createInstallArgs(this); // 1. 根据安装位置创建InstallArgs 2. 根据安装参数修改apk安装位置
    mArgs = args;
    ...
    /*
     * No package verification is enabled, so immediately start
     * the remote call to initiate copy using temporary file.
     */
    ret = args.copyApk(mContainerService, true); // copy操作由MediaContainerService服务委托给DefaultContainerService::copy完成
    // app: data/local/tmp --> data/app/vmdl<回话ID>.tmp ，设置权限644
    // app so :/data/app/vmdl<回话ID>.tmp/lib/arm/
    mRet = ret;
}
```

## 3.11.2 handleStartCopy逻辑
![handleStartCopy逻辑](http://upload-images.jianshu.io/upload_images/74811-3876c3083b3fb0da.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 3.11.3 InstallArgs相关结构
![InstallArgs相关结构](http://upload-images.jianshu.io/upload_images/74811-856869cee3da4214.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 3.11.3 HandlerParams相关结构
![HandlerParams相关结构](http://upload-images.jianshu.io/upload_images/74811-a2dc6a0576a3e861.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 3.12 PackageMS::InstallParams::handleReturnCode
```
void handleReturnCode() {
    // If mArgs is null, then MCS couldn't be reached. When it
    // reconnects, it will try again to install. At that point, this
    // will succeed.
    if (mArgs != null) {
        processPendingInstall(mArgs, mRet);
    }
}
```

## 3.13 PackageMS::processPendingInstall
```
private void processPendingInstall(final InstallArgs args, final int currentStatus) {
    // Queue up an async operation since the package installation may take a little while.
    mHandler.post(new Runnable() {
        public void run() {
            ...
            if (res.returnCode == PackageManager.INSTALL_SUCCEEDED) {
                args.doPreInstall(res.returnCode); // 安装前处理 1. status != PackageManager.INSTALL_SUCCEEDED -> cleanUp
                synchronized (mInstallLock) {
                    installPackageTracedLI(args, res); // 3.14 最终 installPackageLI
                }
                args.doPostInstall(res.returnCode, res.uid); // 3.15
            }

            // A restore should be performed at this point if (a) the install
            // succeeded, (b) the operation is not an update, and (c) the new
            // package has not opted out of backup participation.
            final boolean update = res.removedInfo != null
                    && res.removedInfo.removedPackage != null;
            final int flags = (res.pkg == null) ? 0 : res.pkg.applicationInfo.flags;
            boolean doRestore = !update
                    && ((flags & ApplicationInfo.FLAG_ALLOW_BACKUP) != 0);

            // Set up the post-install work request bookkeeping.  This will be used
            // and cleaned up by the post-install event handling regardless of whether
            // there's a restore pass performed.  Token values are >= 1.
            ...

            PostInstallData data = new PostInstallData(args, res);
            mRunningInstalls.put(token, data);

            if (res.returnCode == PackageManager.INSTALL_SUCCEEDED && doRestore) {
                // Pass responsibility to the Backup Manager.  It will perform a
                // restore if appropriate, then pass responsibility back to the
                // Package Manager to run the post-install observer callbacks
                // and broadcasts.
                ...
            }

            if (!doRestore) {
                ...
                Message msg = mHandler.obtainMessage(POST_INSTALL, token, 0); // 3.16
                mHandler.sendMessage(msg);
            }
        }
    });
}
```

## 3.14 PackageMS::installPackageLI
![PackageMS-installPackageLI](http://upload-images.jianshu.io/upload_images/74811-e612369ed7baca67.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 3.15 PackageMS::doPostInstall
```
if status != PackageManager.INSTALL_SUCCEEDED -> cleanUp    
```

## 3.16 PackageMS::PackageHandler::doHandleMessage
```
case POST_INSTALL: {
    ...
    if (data != null) {
        InstallArgs args = data.args;
        PackageInstalledInfo parentRes = data.res;

        final boolean grantPermissions = (args.installFlags
                & PackageManager.INSTALL_GRANT_RUNTIME_PERMISSIONS) != 0;
        final boolean killApp = (args.installFlags
                & PackageManager.INSTALL_DONT_KILL_APP) == 0;
        final String[] grantedPermissions = args.installGrantPermissions;

        // Handle the parent package
        handlePackagePostInstall(parentRes, grantPermissions, killApp,
                grantedPermissions, didRestore, args.installerPackageName,
                args.observer); // 3.17
        ...
} break;
```
## 3.17 PackageMS::handlePackagePostInstall
```
private void handlePackagePostInstall(PackageInstalledInfo res, boolean grantPermissions,
        boolean killApp, String[] grantedPermissions,
        boolean launchedForRestore, String installerPackage,
        IPackageInstallObserver2 installObserver) {
        ...
            // Send added for users that see the package for the first time
            sendPackageBroadcast(Intent.ACTION_PACKAGE_ADDED, packageName,
                    extras, 0 /*flags*/, null /*targetPackage*/,
                    null /*finishedReceiver*/, firstUsers);

            // Send added for users that don't see the package for the first time
            if (update) {
                extras.putBoolean(Intent.EXTRA_REPLACING, true);
            }
            sendPackageBroadcast(Intent.ACTION_PACKAGE_ADDED, packageName,
                    extras, 0 /*flags*/, null /*targetPackage*/,
                    null /*finishedReceiver*/, updateUsers);

            // Send replaced for users that don't see the package for the first time
            if (update) {
                sendPackageBroadcast(Intent.ACTION_PACKAGE_REPLACED,
                        packageName, extras, 0 /*flags*/,
                        null /*targetPackage*/, null /*finishedReceiver*/,
                        updateUsers);
                sendPackageBroadcast(Intent.ACTION_MY_PACKAGE_REPLACED,
                        null /*package*/, null /*extras*/, 0 /*flags*/,
                        packageName /*targetPackage*/,
                        null /*finishedReceiver*/, updateUsers);
            }
            ...
}
```
通过广播通知系统，app安装成功

----------

## 4.1 PackageMS::deletePackageAsUser
```
public void deletePackageAsUser(String packageName, IPackageDeleteObserver observer, int userId,
        int flags) {
    deletePackage(packageName, new LegacyPackageDeleteObserver(observer).getBinder(), userId,
            flags);
}

```

## 4.2 PackageMS::deletePackage
```
public void deletePackage(final String packageName,
        final IPackageDeleteObserver2 observer, final int userId, final int deleteFlags) {
    // 权限检查
    mContext.enforceCallingOrSelfPermission(
            android.Manifest.permission.DELETE_PACKAGES, null);
    ...
    mHandler.post(new Runnable() {
        public void run() {
            mHandler.removeCallbacks(this);
            int returnCode;
            if (!deleteAllUsers) {
                returnCode = deletePackageX(packageName, userId, deleteFlags); // 4.3
            } else {
                    ...
                }
            }
            observer.onPackageDeleted(packageName, returnCode, null);
        } //end run
    });
}
```
## 4.3 PackageMS::deletePackageX
```
private int deletePackageX(String packageName, int userId, int deleteFlags) {
    ...
    /// M: Try to remove dex file.
    PackageSetting ps = mSettings.mPackages.get(packageName);
    if (ps != null) {
        if (isVendorApp(ps)) {
            ...
            mInstaller.rmdexcache(ps.pkg.baseCodePath, getPrimaryInstructionSet(ps.pkg.applicationInfo));
            ...
            }
        }
    }
    ...
    res = deletePackageLIF(packageName, UserHandle.of(removeUser), true, allUsers,
                        deleteFlags | REMOVE_CHATTY, info, true, null); // 4.6
    ...
    // Force a gc here.
    Runtime.getRuntime().gc();
    // Delete the resources here after sending the broadcast to let
    // other processes clean up before deleting resources.
    if (info.args != null) {
        synchronized (mInstallLock) {
            info.args.doPostDeleteLI(true); // 4.4
        }
    }
    return res ? PackageManager.DELETE_SUCCEEDED : PackageManager.DELETE_FAILED_INTERNAL_ERROR;
}
```

## 4.4 PackageMS::FileInstallArgs::doPostDeleteLI
```
boolean doPostDeleteLI(boolean delete) {
    // XXX err, shouldn't we respect the delete flag?
    cleanUpResourcesLI(); // 4.5
    return true;
}
```

## 4.5 PackageMS::FileInstallArgs::cleanUpResourcesLI
```
void cleanUpResourcesLI() {
    // Try enumerating all code paths before deleting
    List<String> allCodePaths = Collections.EMPTY_LIST;
    if (codeFile != null && codeFile.exists()) {
        try {
            final PackageLite pkg = PackageParser.parsePackageLite(codeFile, 0);
            allCodePaths = pkg.getAllCodePaths();
        } catch (PackageParserException e) {
            // Ignored; we tried our best
        }
    }

    cleanUp(); //removeCodePathLI
    removeDexFiles(allCodePaths, instructionSets);
}
```

## 4.6 PackageMS::deletePackageLIF
```
/*
 * This method handles package deletion in general
 */
private boolean deletePackageLIF(String packageName, UserHandle user,
        boolean deleteCodeAndResources, int[] allUserHandles, int flags,
        PackageRemovedInfo outInfo, boolean writeSettings,
        PackageParser.Package replacingPackage) {
    ...
    ret = deleteInstalledPackageLIF(ps, deleteCodeAndResources, flags, allUserHandles,
            outInfo, writeSettings, replacingPackage);
    ...
    return ret;
}
```
## 4.7 PackageMS::deleteInstalledPackageLIF
```
private boolean deleteInstalledPackageLIF(PackageSetting ps,
        boolean deleteCodeAndResources, int flags, int[] allUserHandles,
        PackageRemovedInfo outInfo, boolean writeSettings,
        PackageParser.Package replacingPackage) {
    ...
    // Delete package data from internal structures and also remove data if flag is set
    removePackageDataLIF(ps, allUserHandles, outInfo, flags, writeSettings);
    ...
    // Delete application code and resources only for parent packages
    if (ps.parentPackageName == null) {
        if (deleteCodeAndResources && (outInfo != null)) {
            outInfo.args = createInstallArgsForExisting(packageFlagsToInstallFlags(ps),
                    ps.codePathString, ps.resourcePathString, getAppDexInstructionSets(ps));
            if (DEBUG_SD_INSTALL) Slog.i(TAG, "args=" + outInfo.args);
        }
    }
    ...
    return true;
}
```
