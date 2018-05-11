## 1. PackageMS相关框架类

![PackageMS](http://upload-images.jianshu.io/upload_images/74811-d9c6a659e96aa552.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 2.1 PackageMS启动过程
```
# SystemServer.java
private void startBootstrapServices() {
        ...
        mSystemServiceManager.startBootPhase(SystemService.PHASE_WAIT_FOR_DEFAULT_DISPLAY);
        ...
        mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
            mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
        mFirstBoot = mPackageManagerService.isFirstBoot();
        mPackageManager = mSystemContext.getPackageManager();
        ...
}
```
## 2.2 SystemServer::startOtherServices
```
private void startOtherServices() {
    ...
    mSystemServiceManager.startBootPhase(SystemService.PHASE_LOCK_SETTINGS_READY);
    mSystemServiceManager.startBootPhase(SystemService.PHASE_SYSTEM_SERVICES_READY);
    ...
    mPackageManagerService.systemReady();
    ...
    mSystemServiceManager.startBootPhase(SystemService.PHASE_ACTIVITY_MANAGER_READY);
}
```

## 2.3 PackageMS::main
```
public static PackageManagerService main(Context context, Installer installer,
        boolean factoryTest, boolean onlyCore) {
    ...
    PackageManagerService m = new PackageManagerService(context, installer,
    ...
    ServiceManager.addService("package", m);
    return m;
}
```

## 2.4 PackageMS初始化
```
public PackageManagerService(Context context, Installer installer,
        boolean factoryTest, boolean onlyCore) {
    ...
    // PMS_START
    // A.1 创建Settings
    mSettings = new Settings(mPackages);
    // A.2 system phone log nfc bluetooth shell 添加到Setting
    mSettings.addSharedUserLPw("android.uid.system", Process.SYSTEM_UID,
            ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
    mSettings.addSharedUserLPw("android.uid.phone", RADIO_UID,
            ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
    mSettings.addSharedUserLPw("android.uid.log", LOG_UID,
            ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
    mSettings.addSharedUserLPw("android.uid.nfc", NFC_UID,
            ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
    mSettings.addSharedUserLPw("android.uid.bluetooth", BLUETOOTH_UID,
            ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
    mSettings.addSharedUserLPw("android.uid.shell", SHELL_UID,
            ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
    ...
    mPackageDexOptimizer = new PackageDexOptimizer(installer, mInstallLock, context,
            "*dexopt*");
    ...
    // A.3 初始化SystemConfig
    SystemConfig systemConfig = SystemConfig.getInstance();
    mGlobalGids = systemConfig.getGlobalGids();
    mSystemPermissions = systemConfig.getSystemPermissions();
    mAvailableFeatures = systemConfig.getAvailableFeatures();

    synchronized (mInstallLock) {
    synchronized (mPackages) {
        mHandlerThread = new ServiceThread(TAG,
                Process.THREAD_PRIORITY_BACKGROUND, true /*allowIo*/);
        mHandlerThread.start();
        // A.4 创建PackageHandler
        mHandler = new PackageHandler(mHandlerThread.getLooper());
        Watchdog.getInstance().addThread(mHandler, WATCHDOG_TIMEOUT);
        ...
        ArrayMap<String, String> libConfig = systemConfig.getSharedLibraries();
        for (int i=0; i<libConfig.size(); i++) {
            mSharedLibraries.put(libConfig.keyAt(i),
                    new SharedLibraryEntry(libConfig.valueAt(i), null));
        }

        // PMS_SYSTEM_SCAN_START
        ...
        final String bootClassPath = System.getenv("BOOTCLASSPATH");
        final String systemServerClassPath = System.getenv("SYSTEMSERVERCLASSPATH");
        ...
        // B.1 对满足调价的Jar Apk执行dex优化
        mInstaller.dexopt(lib, Process.SYSTEM_UID, dexCodeInstructionSet, dexoptNeeded, DEXOPT_PUBLIC /*dexFlags*/,
                getCompilerFilterForReason(REASON_SHARED_APK), StorageManager.UUID_PRIVATE_INTERNAL, SKIP_SHARED_LIBRARY_CHECK);
        ...
        // B.2 扫描system/app system/priv-app
        // Collect ordinary system packages.
        final File systemAppDir = new File(Environment.getRootDirectory(), "app");
        scanDirTracedLI(systemAppDir, mDefParseFlags
                | PackageParser.PARSE_IS_SYSTEM
                | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);
        ...
        // Collect all OEM packages.
        final File oemAppDir = new File(Environment.getOemDirectory(), "app");
        scanDirTracedLI(oemAppDir, mDefParseFlags
                | PackageParser.PARSE_IS_SYSTEM
                | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);
        ...

        //look for any incomplete package installations
        ArrayList<PackageSetting> deletePkgsList = mSettings.getListOfIncompleteInstallPackagesLPr();
        for (int i = 0; i < deletePkgsList.size(); i++) {
            // Actual deletion of code and data will be handled by later
            // reconciliation step
            final String packageName = deletePkgsList.get(i).name;
            logCriticalInfo(Log.WARN, "Cleaning up incompletely installed app: " + packageName);
            synchronized (mPackages) {
                mSettings.removePackageLPw(packageName);
            }
        }

        //delete tmp files
        deleteTempPackageFiles();

        // Remove any shared userIDs that have no associated packages
        mSettings.pruneSharedUsersLPw();

        // PMS_DATA_SCAN_START
        
        //C.1 处理非系统App //data/app 、 自定义app路径
        if (!mOnlyCore) {
            scanDirTracedLI(mAppInstallDir, 0, scanFlags | SCAN_REQUIRE_KNOWN, 0);
            ...

            /**
             * Remove disable package settings for any updated system
             * apps that were removed via an OTA. If they're not a
             * previously-updated app, remove them completely.
             * Otherwise, just revoke their system-level permissions.
             */
            for (String deletedAppName : possiblyDeletedUpdatedSystemApps) {
                PackageParser.Package deletedPkg = mPackages.get(deletedAppName);
                mSettings.removeDisabledSystemPackageLPw(deletedAppName);

                String msg = "Updated system app + " + deletedAppName
                            + " no longer present; removing system privileges for "
                            + deletedAppName;

                    deletedPkg.applicationInfo.flags &= ~ApplicationInfo.FLAG_SYSTEM;

                    /// M: [Operator] Revoke operator permissions for the original operator package
                    /// under operator folder was gone due to OTA
                    deletedPkg.applicationInfo.flagsEx &= ~ApplicationInfo.FLAG_EX_OPERATOR;

                    PackageSetting deletedPs = mSettings.mPackages.get(deletedAppName);
                    deletedPs.pkgFlags &= ~ApplicationInfo.FLAG_SYSTEM;

                    /// M: [Operator] Revoke vendor permissions
                    deletedPs.pkgFlagsEx &= ~ApplicationInfo.FLAG_EX_OPERATOR;
                }
            }

            /**
             * Make sure all system apps that we expected to appear on
             * the userdata partition actually showed up. If they never
             * appeared, crawl back and revive the system version.
             */
            for (int i = 0; i < mExpectingBetter.size(); i++) {
                final String packageName = mExpectingBetter.keyAt(i);
                if (!mPackages.containsKey(packageName)) {
                    final File scanFile = mExpectingBetter.valueAt(i);
                    ...
                    scanPackageTracedLI(scanFile, reparseFlags, scanFlags, 0, null);
                }
            }
        }
        ...
        // PMS_SCAN_END
        // D.1 回写package.xml
        // can downgrade to reader
        mSettings.writeLPr();
        ...
        // PMS_READY
        // E1 创建PackageInstallerService
        mInstallerService = new PackageInstallerService(context, this);
        ...
    } // synchronized (mPackages)
    } // synchronized (mInstallLock)
}
```

## 2.5 PackageMS::scanDirLi
![scanDirLi](http://upload-images.jianshu.io/upload_images/74811-1057af60b03520a4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###### 参考
[PackageManager启动篇](http://gityuan.com/2016/11/06/packagemanager/ "PackageManager启动篇")
