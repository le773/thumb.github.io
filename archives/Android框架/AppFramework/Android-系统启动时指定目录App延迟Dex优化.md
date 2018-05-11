## 思路
`PakcageMS`启动时添加`Flag SCAN_NO_DEX`, 当扫描执行目录时，跳过做`Dex`优化步骤，其次在`SystemServer::startOtherServices`时启动`JobSchedule`任务,延迟一分钟进行`Dex`任务；


## 1.1 扫描指定目录添加flag
```
public PackageManagerService(Context context, Installer installer, boolean factoryTest, boolean onlyCore) {
    ...
    scanDirLI(customFrameworkDir, PackageParser.PARSE_IS_SYSTEM
            | PackageParser.PARSE_IS_SYSTEM_DIR,
            scanFlags | SCAN_NO_DEX, 0); //添加flag
    ...
}
```

## 1.2 SCAN_NO_DEX 
```
private PackageParser.Package scanPackageDirtyLI(PackageParser.Package pkg, int parseFlags,
        int scanFlags, long currentTime, UserHandle user) throws PackageManagerException {
     ...
     if ((scanFlags & SCAN_NO_DEX) == 0) { // PackageMS启动时不做Dex优化
         int result = mPackageDexOptimizer.performDexOpt(pkg, null /* instruction sets */,
                 forceDex, (scanFlags & SCAN_DEFER_DEX) != 0, false /* inclDependencies */);
         if (result == PackageDexOptimizer.DEX_OPT_FAILED) {
             throw new PackageManagerException(INSTALL_FAILED_DEXOPT, "scanPackageLI");
         }
     }
    ...
}
```

## 1.3 启动JobSchedule延迟Dex优化
```
private void startOtherServices() {
...
    mSystemServiceManager.startService(JobSchedulerService.class);
    traceBeginAndSlog("StartBackgroundDexOptService");
    try {
        BackgroundDexOptService.schedule(context);
    } catch (Throwable e) {
        reportWtf("starting BackgroundDexOptService", e);
    }
    Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
    ...
    mSystemServiceManager.startBootPhase(SystemService.PHASE_LOCK_SETTINGS_READY);
    ...
}
```

## 1.4 BackgroundDexOptService延迟执行Dex优化
```
public class BackgroundDexOptService extends JobService {
    static final String TAG = "BackgroundDexOptService";

    static final long RETRY_LATENCY = 4 * AlarmManager.INTERVAL_HOUR;

    static final int JOB_IDLE_OPTIMIZE = 800;
    static final int JOB_POST_BOOT_UPDATE = 801;

    private static ComponentName sDexoptServiceName = new ComponentName(
            "android",
            BackgroundDexOptService.class.getName());

    /**
     * Set of failed packages remembered across job runs.
     */
    static final ArraySet<String> sFailedPackageNames = new ArraySet<String>();

    /**
     * Atomics set to true if the JobScheduler requests an abort.
     */
    final AtomicBoolean mAbortPostBootUpdate = new AtomicBoolean(false);
    final AtomicBoolean mAbortIdleOptimization = new AtomicBoolean(false);

    /**
     * Atomic set to true if one job should exit early because another job was started.
     */
    final AtomicBoolean mExitPostBootUpdate = new AtomicBoolean(false);

    private final File dataDir = Environment.getDataDirectory();

    public static void schedule(Context context) {
        JobScheduler js = (JobScheduler) context.getSystemService(Context.JOB_SCHEDULER_SERVICE);

        // Schedule a one-off job which scans installed packages and updates
        // out-of-date oat files.
        js.schedule(new JobInfo.Builder(JOB_POST_BOOT_UPDATE, sDexoptServiceName)
                    .setMinimumLatency(TimeUnit.MINUTES.toMillis(1)) // 延迟一分钟
                    .setOverrideDeadline(TimeUnit.MINUTES.toMillis(1)) 
                    .build());

        // Schedule a daily job which scans installed packages and compiles
        // those with fresh profiling data.
        js.schedule(new JobInfo.Builder(JOB_IDLE_OPTIMIZE, sDexoptServiceName)
                    .setRequiresDeviceIdle(true)
                    .setRequiresCharging(true)
                    .setPeriodic(TimeUnit.DAYS.toMillis(1)) // 每天
                    .build());
    }

    // Returns the current battery level as a 0-100 integer.
    private int getBatteryLevel() {
        IntentFilter filter = new IntentFilter(Intent.ACTION_BATTERY_CHANGED);
        Intent intent = registerReceiver(null, filter);
        int level = intent.getIntExtra(BatteryManager.EXTRA_LEVEL, -1);
        int scale = intent.getIntExtra(BatteryManager.EXTRA_SCALE, -1);

        if (level < 0 || scale <= 0) {
            // Battery data unavailable. This should never happen, so assume the worst.
            return 0;
        }

        return (100 * level / scale);
    }

    private long getLowStorageThreshold() {
        @SuppressWarnings("deprecation")
        final long lowThreshold = StorageManager.from(this).getStorageLowBytes(dataDir);
        if (lowThreshold == 0) {
            Log.e(TAG, "Invalid low storage threshold");
        }
        return lowThreshold;
    }

    private boolean runPostBootUpdate(final JobParameters jobParams,
            final PackageManagerService pm, final ArraySet<String> pkgs) {
        if (mExitPostBootUpdate.get()) {
            // This job has already been superseded. Do not start it.
            return false;
        }

        // Load low battery threshold from the system config. This is a 0-100 integer.
        final int lowBatteryThreshold = getResources().getInteger(
                com.android.internal.R.integer.config_lowBatteryWarningLevel);

        final long lowThreshold = getLowStorageThreshold();

        mAbortPostBootUpdate.set(false);
        new Thread("BackgroundDexOptService_PostBootUpdate") {
            @Override
            public void run() {
                for (String pkg : pkgs) {
                    if (mAbortPostBootUpdate.get()) {
                        // JobScheduler requested an early abort.
                        return;
                    }
                    if (mExitPostBootUpdate.get()) {
                        // Different job, which supersedes this one, is running.
                        break;
                    }
                    if (getBatteryLevel() < lowBatteryThreshold) {
                        // Rather bail than completely drain the battery.
                        break;
                    }
                    long usableSpace = dataDir.getUsableSpace();
                    if (usableSpace < lowThreshold) {
                        // Rather bail than completely fill up the disk.
                        Log.w(TAG, "Aborting background dex opt job due to low storage: " +
                                usableSpace);
                        break;
                    }


                    // Update package if needed. Note that there can be no race between concurrent
                    // jobs because PackageDexOptimizer.performDexOpt is synchronized.

                    // checkProfiles is false to avoid merging profiles during boot which
                    // might interfere with background compilation (b/28612421).
                    // Unfortunately this will also means that "pm.dexopt.boot=speed-profile" will
                    // behave differently than "pm.dexopt.bg-dexopt=speed-profile" but that's a
                    // trade-off worth doing to save boot time work.
                    pm.performDexOpt(pkg,
                            /* checkProfiles */ false,
                            PackageManagerService.REASON_BOOT,
                            /* force */ false);//dex优化
                }
                // Ran to completion, so we abandon our timeslice and do not reschedule.
                jobFinished(jobParams, /* reschedule */ false);
            }
        }.start();
        return true;
    }

    private boolean runIdleOptimization(final JobParameters jobParams,
            final PackageManagerService pm, final ArraySet<String> pkgs) {
        // If post-boot update is still running, request that it exits early.
        mExitPostBootUpdate.set(true);

        mAbortIdleOptimization.set(false);

        final long lowThreshold = getLowStorageThreshold();

        new Thread("BackgroundDexOptService_IdleOptimization") {
            @Override
            public void run() {
                for (String pkg : pkgs) {
                    if (mAbortIdleOptimization.get()) {
                        // JobScheduler requested an early abort.
                        return;
                    }
                    if (sFailedPackageNames.contains(pkg)) {
                        // Skip previously failing package
                        continue;
                    }

                    long usableSpace = dataDir.getUsableSpace();
                    if (usableSpace < lowThreshold) {
                        // Rather bail than completely fill up the disk.
                        Log.w(TAG, "Aborting background dex opt job due to low storage: " +
                                usableSpace);
                        break;
                    }

                    // Conservatively add package to the list of failing ones in case performDexOpt
                    // never returns.
                    synchronized (sFailedPackageNames) {
                        sFailedPackageNames.add(pkg);
                    }
                    // Optimize package if needed. Note that there can be no race between
                    // concurrent jobs because PackageDexOptimizer.performDexOpt is synchronized.
                    if (pm.performDexOpt(pkg,
                            /* checkProfiles */ true,
                            PackageManagerService.REASON_BACKGROUND_DEXOPT,
                            /* force */ false)) {
                        // Dexopt succeeded, remove package from the list of failing ones.
                        synchronized (sFailedPackageNames) {
                            sFailedPackageNames.remove(pkg);
                        }
                    }
                }
                // Ran to completion, so we abandon our timeslice and do not reschedule.
                jobFinished(jobParams, /* reschedule */ false);
            }
        }.start();
        return true;
    }

    @Override
    public boolean onStartJob(JobParameters params) {
        // NOTE: PackageManagerService.isStorageLow uses a different set of criteria from
        // the checks above. This check is not "live" - the value is determined by a background
        // restart with a period of ~1 minute.
        PackageManagerService pm = (PackageManagerService)ServiceManager.getService("package");
        if (pm.isStorageLow()) {
            return false;
        }

        final ArraySet<String> pkgs = pm.getOptimizablePackages();
        if (pkgs == null || pkgs.isEmpty()) {
            return false;
        }

        if (params.getJobId() == JOB_POST_BOOT_UPDATE) {
            return runPostBootUpdate(params, pm, pkgs);
        } else {
            return runIdleOptimization(params, pm, pkgs);
        }
    }

    @Override
    public boolean onStopJob(JobParameters params) {
        if (params.getJobId() == JOB_POST_BOOT_UPDATE) {
            mAbortPostBootUpdate.set(true);
        } else {
            mAbortIdleOptimization.set(true);
        }
        return false;
    }
}
```
