## 1.1 ZygoteInit::preload
```
private static final CountDownLatch mConnectedSignal = new CountDownLatch(3);
static void preload() {
    Log.d(TAG, "begin preload");
    /* preload class and preload resource*/
    preloadResourcesThread = new Thread("preloadResources"){
             public void run() {
                    preloadResources(); //1.2
                    mConnectedSignal.countDown();
            }
     };
    preloadClasses(); //1.3
    waitForLatch(mConnectedSignal);
    preloadOpenGL();
    preloadSharedLibraries();// android、compiler_rt、jnigraphics
    preloadTextResources();
	...
    Log.d(TAG, "end preload");
}
```

## 1.2 ZygoteInit::preloadResources
```
/*
 * Load in commonly used resources, so they can be shared across
 * processes.
 *
 * These tend to be a few Kbytes, but are frequently in the 20-40K
 * range, and occasionally even larger.
*/

private static void preloadResources() {
    final VMRuntime runtime = VMRuntime.getRuntime();

    try {
        mResources = Resources.getSystem();
        mResources.startPreloading();
        if (PRELOAD_RESOURCES) {
            Log.i(TAG, "Preloading resources...");

            long startTime = SystemClock.uptimeMillis();
            TypedArray ar = mResources.obtainTypedArray(
                    com.android.internal.R.array.preloaded_drawables);
            int N = preloadDrawables(runtime, ar);
            ar.recycle();
            Log.i(TAG, "...preloaded " + N + " resources in "
                    + (SystemClock.uptimeMillis()-startTime) + "ms.");

            startTime = SystemClock.uptimeMillis();
            ar = mResources.obtainTypedArray(
                    com.android.internal.R.array.preloaded_color_state_lists);
            N = preloadColorStateLists(runtime, ar);
            ar.recycle();
            Log.i(TAG, "...preloaded " + N + " resources in "
                    + (SystemClock.uptimeMillis()-startTime) + "ms.");
        }
        mResources.finishPreloading();
    } catch (RuntimeException e) {
        Log.w(TAG, "Failure preloading resources", e);
    }
}
```

## 1.3 ZygoteInit::preloadClasses
```
/**
 * Performs Zygote process initialization. Loads and initializes
 * commonly used classes.
 *
 * Most classes only cause a few hundred bytes to be allocated, but
 * a few will allocate a dozen Kbytes (in one case, 500+K).
 */
private static void preloadClasses() {
    final VMRuntime runtime = VMRuntime.getRuntime();
    InputStream is = new FileInputStream(PRELOADED_CLASSES);
    Log.i(TAG, "Preloading classes...");
    long startTime = SystemClock.uptimeMillis();
    ...
    // Alter the target heap utilization.  With explicit GCs this
    // is not likely to have any effect.
    float defaultUtilization = runtime.getTargetHeapUtilization();
    runtime.setTargetHeapUtilization(0.8f);

    try {
        BufferedReader br = new BufferedReader(new InputStreamReader(is), 256);
        int count = 0;
        String line = null;
        final ArrayList<String> lines = new ArrayList<String>();
        while ((line = br.readLine()) != null) {
            // Skip comments and blank lines.
            line = line.trim();
            if (line.startsWith("#") || line.equals("")) {
                continue;
            }
            lines.add(line);
            count++;
        }
        Log.d(TAG, "Number of total Classes to preload: " + count);

        int start = 0;
        int end = start + count / 5;
        final List<String> list1 = lines.subList(start, end);
        Log.d(TAG,"Classes to load for thread1 " + start + "~"+end);
        loadClassAsync(list1, 1, startTime);//1.4
        preloadResourcesThread.start(); // 1.1 preloadResources

        start = end + 1;
        end = start + count / 7;
        final List<String> list2 = lines.subList(start, end);
        Log.d(TAG,"Classes to load for thread2 " + start + "~"+end);
        loadClassAsync(list2, 2, startTime); //1.4

        start = end + 1;
        end = count - 1;
        final List<String> list3 = lines.subList(start, end);
        Log.d(TAG,"Classes to load for thread3 " + start + "~"+end);
        loadClassImpl(list3, 3, startTime); //1.5
    } catch (IOException e) {
        Log.e(TAG, "Error reading " + PRELOADED_CLASSES + ".", e);
    } finally {
        // Restore default.
        runtime.setTargetHeapUtilization(defaultUtilization);
        // Fill in dex caches with classes, fields, and methods brought in by preloading.
        runtime.preloadDexCaches();
        // Bring back root. We'll need it later if we're in the zygote.
        ...
    }
}
```
## 1.4 ZygoteInit::loadClassAsync
```
private static void loadClassAsync(final List<String> list, final int index, final long startTime){
    new Thread(new Runnable() {
        public void run() {
                loadClassImpl(list, index, startTime);
                mConnectedSignal.countDown();
        }
    }, ("...proclass" + index)).start();
}
```

## 1.5 ZygoteInit::loadClassImpl
```
private static void loadClassImpl(final List<String> list, final int index, final long startTime){
    for (String line : list) {
        Class.forName(line, true, null);
}
```


----------

## 2.1 PackageManagerService::main
方案1：多线程扫描不同的目录（粒度较大）
```
private  final CountDownLatch mConnectedSignal = new CountDownLatch(5);
...
new Thread("scanPrivilegedAppDir"){
    @Override
    public void run() {
        long startTime = SystemClock.uptimeMillis();
        scanDirLI(privilegedAppDir, PackageParser.PARSE_IS_SYSTEM
                | PackageParser.PARSE_IS_SYSTEM_DIR
                | PackageParser.PARSE_IS_PRIVILEGED, scanFlags, 0);
        Slog.w(TAG, "scanPrivilegedAppDir done,cost "+((SystemClock.uptimeMillis()-startTime)/1000f)+" seconds");
        mConnectedSignal.countDown();

    }
}.start();
```

----------

## 2.2 PackageManagerService::scanDirLI
方案2：多线程以apk为粒度进行扫描
```
private void scanDirLI(File dir, final int parseFlags, int scanFlags, long currentTime) {
    final File[] files = dir.listFiles();
    ...
    Log.d(TAG, "start scanDirLI:"+dir);
    // use multi thread to speed up scanning
    int iMultitaskNum = SystemProperties.getInt("persist.pm.multitask", 6);
    Log.d(TAG, "max thread:" + iMultitaskNum);
    final MultiTaskDealer dealer = (iMultitaskNum > 1) ? MultiTaskDealer.startDealer(
            MultiTaskDealer.PACKAGEMANAGER_SCANER, iMultitaskNum) : null;

    for (File file : files) {
        ...
        Runnable scanTask = new Runnable() {
            public void run() {
                try {
                    scanPackageTracedLI(ref_file, ref_parseFlags | PackageParser.PARSE_MUST_BE_APK,
                            ref_scanFlags, ref_currentTime, null);
                } catch (PackageManagerException e) {
                    Slog.w(TAG, "Failed to parse " + ref_file + ": " + e.getMessage());

                    // Delete invalid userdata apps
                    if ((ref_parseFlags & PackageParser.PARSE_IS_SYSTEM) == 0 &&
                            e.error == PackageManager.INSTALL_FAILED_INVALID_APK) {
                        logCriticalInfo(Log.WARN, "Deleting invalid package at " + ref_file);
                        removeCodePathLI(ref_file);
                    }
                }
            }
        };

        if (dealer != null)
            dealer.addTask(scanTask);
        else
            scanTask.run();
    }

    if (dealer != null)
        dealer.waitAll();
    Log.d(TAG, "end scanDirLI:"+dir);
}
```

## 2.3 MultiTaskDealer构造函数
```
private ThreadPoolExecutor mExecutor;
private int mTaskCount = 0;
private boolean mNeedNotifyEnd = false;
private Object mObjWaitAll = new Object();
private ReentrantLock mLock = new ReentrantLock();

public MultiTaskDealer(String name,int taskCount) {
    final String taskName = name;
    ThreadFactory factory = new ThreadFactory()
    {
        private final AtomicInteger mCount = new AtomicInteger(1);
        public Thread newThread(final Runnable r) {
            return new Thread(r, taskName + "-" + mCount.getAndIncrement());
        }
    };
    mExecutor = new ThreadPoolExecutor(taskCount, taskCount, 5, TimeUnit.SECONDS,
            new LinkedBlockingQueue<Runnable>(), factory){
        protected void afterExecute(Runnable r, Throwable t) {
            if(t!=null) {
                t.printStackTrace();
            }
            MultiTaskDealer.this.TaskCompleteNotify(r);
            super.afterExecute(r,t);
        }
        protected void beforeExecute(Thread t, Runnable r) {
            if (DEBUG_TASK) Log.d(TAG, "start task");
            super.beforeExecute(t,r);
        }
    };
}
```

## 2.4 MultiTaskDealer::startDealer
```
public static MultiTaskDealer startDealer(String name,int taskCount) {
    MultiTaskDealer dealer = getDealer(name);
    if(dealer==null) {
        dealer = new MultiTaskDealer(name,taskCount);
        WeakReference<MultiTaskDealer> ref = new WeakReference<MultiTaskDealer>(dealer);
        map.put(name,ref);
    }
    return dealer;
}
```

## 2.5 MultiTaskDealer::addTask
```
public void addTask(Runnable task) {
    synchronized (mObjWaitAll) {
        mTaskCount+=1;
    }
    mExecutor.execute(task);
}
```
