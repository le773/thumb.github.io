## 1.1 init.zygote.rc 配置
```
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server // 1.2 app_process
    class main
    socket zygote stream 660 root system // 创建一个名为zygote，类型为stream，权限为660的socket
    onrestart write /sys/android_power/request_state wake // zygote重启的时候，写入字符
    onrestart write /sys/power/state on
    onrestart restart audioserver // zygote重启的时候，重启audioserver
    onrestart restart cameraserver
    onrestart restart media
    onrestart restart netd 
    writepid /dev/cpuset/foreground/tasks /dev/stune/foreground/tasks
```

## 1.2 app_process 模块
```
# frameworks/base/cmds/app_process/Android.mk
LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)

LOCAL_SRC_FILES:= \
    app_main.cpp // 解析init.rc zygote
...
LOCAL_SHARED_LIBRARIES := \
    libdl \
    libcutils \
    libutils \
    liblog \
    libbinder \
    libnativeloader \
    libandroid_runtime // 1.3 libandroid_runtime 模块

LOCAL_MODULE:= app_process
LOCAL_MULTILIB := both
LOCAL_MODULE_STEM_32 := app_process32 // 32位
LOCAL_MODULE_STEM_64 := app_process64
...

include $(BUILD_EXECUTABLE)
```
- app_process 依赖共享库 libandroid_runtime；
- app_process 模块会构建出可执行文件app_process32 app_process64 ，对应于zygote32 zygote64

## 1.3 libandroid_runtime 模块
```
# frameworks/base/core/jni/Android.mk
LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)
...
LOCAL_SRC_FILES:= \ // framework下所有JNI
    AndroidRuntime.cpp \
    android_util_Binder.cpp \
    com_android_internal_os_Zygote.cpp 

LOCAL_C_INCLUDES += \
    ...
    $(TOP)/system/core/include \

LOCAL_SHARED_LIBRARIES := \
    liblog \
    libbinder \
    libnetutils \
    libinput \
    libinputflinger \
    ...
    libmedia \
    libjpeg \

# we need to access the private Bionic header
# <bionic_tls.h> in com_google_android_gles_jni_GLImpl.cpp
LOCAL_C_INCLUDES += bionic/libc/private

# AndroidRuntime.h depends on nativehelper/jni.h
LOCAL_EXPORT_C_INCLUDE_DIRS := libnativehelper/include

LOCAL_MODULE:= libandroid_runtime
...
include $(BUILD_SHARED_LIBRARY)
include $(call all-makefiles-under,$(LOCAL_PATH))

```
- 封装了系统frameworks/base/core/jni 中的jni库，
- 系统libcore、system 等仓库C环境的头文件
- 并依赖libmedia 等其他系统库文件；

## 1.4 App_main.cpp 解析init.zygote.rc
```
int main(int argc, char* const argv[])
{
    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
    ...
    // Parse runtime arguments.  Stop at first unrecognized option.
    bool zygote = false;
    bool startSystemServer = false;
    bool application = false;
    String8 niceName;
    String8 className;
    
    ++i;  // Skip unused "parent dir" argument.
    while (i < argc) { // 参数处理
        const char* arg = argv[i++];
        if (strcmp(arg, "--zygote") == 0) {
            zygote = true;
            niceName = ZYGOTE_NICE_NAME;
        } else if (strcmp(arg, "--start-system-server") == 0) {
            startSystemServer = true;
        } else if (strcmp(arg, "--application") == 0) {
            application = true;
        } else if (strncmp(arg, "--nice-name=", 12) == 0) {
            niceName.setTo(arg + 12);
        } else if (strncmp(arg, "--", 2) != 0) {
            className.setTo(arg);
            break;
        } else {
            --i;
            break;
        }
    }

    Vector<String8> args;
    if (!className.isEmpty()) {
        args.add(application ? String8("application") : String8("tool"));
        runtime.setClassNameAndArgs(className, argc - i, argv + i);
    } else {
        ...
        if (startSystemServer) { // 需要启动SystemServer
            args.add(String8("start-system-server"));
        }
        ...
        for (; i < argc; ++i) {
            args.add(String8(argv[i]));
        }
    }

    if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote); // 1.6
    } else if (className) {
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    }
}
```

## 1.5 AppRuntime类图
![AppRuntime](http://upload-images.jianshu.io/upload_images/74811-cfbc4d8b5f59d562.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
AppRuntime继承自AndroidRuntime，并重写部分方法

## 1.6 AndroidRuntime::start
```
 string.
 */
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
{
    ...
    /* start the virtual machine */
    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);
    JNIEnv* env;
    if (startVm(&mJavaVM, &env, zygote) != 0) { // 1.7 创建虚拟机
        return;
    }
    onVmCreated(env);

    /*
     * Register android functions.
     */
    if (startReg(env) < 0) { // 注册JNI
        ALOGE("Unable to register all android natives\n");
        return;
    }
    ...
    /*
     * Start VM.  This thread becomes the main thread of the VM, and will
     * not return until the VM exits.
     */
    char* slashClassName = toSlashClassName(className);
    jclass startClass = env->FindClass(slashClassName);
    if (startClass == NULL) {
        ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
        /* keep going */
    } else {
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
        if (startMeth == NULL) {
            ALOGE("JavaVM unable to find main() in '%s'\n", className);
            /* keep going */
        } else {
            env->CallStaticVoidMethod(startClass, startMeth, strArray); // 2.1 com.android.internal.os.ZygoteInit main方法
        }
    }
    ...
}
```

## 1.7 AndroidRuntime::startVM
```
// Start the Dalvik Virtual Machine.
int AndroidRuntime::startVm(JavaVM** pJavaVM, JNIEnv** pEnv, bool zygote)
{
    ...
    /*
     * The default starting and maximum size of the heap.  Larger
     * values should be specified in a product property override.
     */
    parseRuntimeOption("dalvik.vm.heapstartsize", heapstartsizeOptsBuf, "-Xms", "4m");
    parseRuntimeOption("dalvik.vm.heapsize", heapsizeOptsBuf, "-Xmx", "16m");
    ...
    /*
     * JIT related options.
     */
    parseRuntimeOption("dalvik.vm.jitmaxsize", jitmaxsizeOptsBuf, "-Xjitmaxsize:");
    ...
     /*
     * Initialize the VM.
     *
     * The JavaVM* is essentially per-process, and the JNIEnv* is per-thread.
     * If this call succeeds, the VM is ready, and we can start issuing
     * JNI calls.
     */
    if (JNI_CreateJavaVM(pJavaVM, pEnv, &initArgs) < 0) {
        ALOGE("JNI_CreateJavaVM failed\n");
        return -1;
    }
}
```

----------

## 2.1 ZygoteInit::main
```
public static void main(String argv[]) {
    ...
    String socketName = "zygote";
    // Mark zygote start. This ensures that thread creation will throw
    // an error.
    ZygoteHooks.startZygoteNoThreadCreation();
    try {
        boolean startSystemServer = false;
        String abiList = null;
        for (int i = 1; i < argv.length; i++) { // 参数解析
            if ("start-system-server".equals(argv[i])) {
                startSystemServer = true;
            } else if (argv[i].startsWith(ABI_LIST_ARG)) {
                abiList = argv[i].substring(ABI_LIST_ARG.length());
            } else if (argv[i].startsWith(SOCKET_NAME_ARG)) {
                socketName = argv[i].substring(SOCKET_NAME_ARG.length());
            } else {
                throw new RuntimeException("Unknown command line argument: " + argv[i]);
            }
        }

        registerZygoteSocket(socketName); // 2.2 注册Socket
        ... 
        preloadByName(socketName);  // 2.3 预加载资源

        if (startSystemServer) { // 2.4 启动SystemServer
            startSystemServer(abiList, socketName);
        }

        runSelectLoop(abiList); // 2.5 循环监听请求
        closeServerSocket(); 
    } catch (MethodAndArgsCaller caller) {
        caller.run(); // 调用Application::main或者SystemServer::main; 这是SystemServer参考4.1
    } catch (RuntimeException ex) {
        closeServerSocket(); // RuntimeException，则关闭Socket
    }
}
```

## 2.2 ZygoteInit::registerZygoteSocket
```
/**
 * Registers a server socket for zygote command connections
 *
 * @throws RuntimeException when open fails
 */
private static void registerZygoteSocket(String socketName) {
    if (sServerSocket == null) {
        int fileDesc;
        final String fullSocketName = ANDROID_SOCKET_PREFIX + socketName;
        try {
            String env = System.getenv(fullSocketName);
            fileDesc = Integer.parseInt(env);
        } catch (RuntimeException ex) {
            throw new RuntimeException(fullSocketName + " unset or invalid", ex);
        }

        try {
            FileDescriptor fd = new FileDescriptor();
            fd.setInt$(fileDesc);
            sServerSocket = new LocalServerSocket(fd); // 监听zygote
        } catch (IOException ex) {
            throw new RuntimeException(
                    "Error binding to local socket '" + fileDesc + "'", ex);
        }
    }
}
```

## 2.3 ZygoteInit::preload
```
static void preload() {
    preloadClasses(); // /system/etc/preloaded-classes
    preloadResources();
    // frameworks/base/core/res/res/values/arrays.xml 中的
        // preloaded_drawables、preloaded_color_state_lists、preloaded_freeform_multi_window_drawables
    preloadOpenGL();
    preloadSharedLibraries(); // android、compiler_rt、jnigraphics
    preloadTextResources();
    WebViewFactory.prepareWebViewInZygote();
    endIcuCachePinning();
    warmUpJcaProviders();
}
```

## 2.4 ZygoteInit::startSystemServer
```
private static boolean startSystemServer(String abiList, String socketName)
        throws MethodAndArgsCaller, RuntimeException {
    ...
    String args[] = {
        "--setuid=1000",
        "--setgid=1000",
        /// M: ANR mechanism for system_server add shell(2000) group to access
        ///    /sys/kernel/debug/tracing/tracing_on
        "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1032,2000," +
            "3001,3002,3003,3006,3007,3009,3010",
        "--capabilities=" + capabilities + "," + capabilities,
        "--nice-name=system_server",
        "--runtime-args",
        "com.android.server.SystemServer", // SystemServer
    };
    ZygoteConnection.Arguments parsedArgs = null;

    int pid;

    try {
        parsedArgs = new ZygoteConnection.Arguments(args); // 封装参数
        ZygoteConnection.applyDebuggerSystemProperty(parsedArgs);
        ZygoteConnection.applyInvokeWithSystemProperty(parsedArgs);

        /* Request to fork the system server process */
        pid = Zygote.forkSystemServer( // linux::fork
                parsedArgs.uid, parsedArgs.gid,
                parsedArgs.gids,
                parsedArgs.debugFlags,
                null,
                parsedArgs.permittedCapabilities,
                parsedArgs.effectiveCapabilities);
    } catch (IllegalArgumentException ex) {
        throw new RuntimeException(ex);
    }

    /* For child process */
    if (pid == 0) {
        if (hasSecondZygote(abiList)) {
            waitForSecondaryZygote(socketName);
        }

        handleSystemServerProcess(parsedArgs); // 3.1 进入SystemServer进程
    }

    return true;
}
```

## 2.5 ZygoteInit::runSelectLoop
```
private static void runSelectLoop(String abiList) throws MethodAndArgsCaller {
    ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();
    ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();

    fds.add(sServerSocket.getFileDescriptor());
    peers.add(null);

    while (true) {
        StructPollfd[] pollFds = new StructPollfd[fds.size()];
        for (int i = 0; i < pollFds.length; ++i) {
            pollFds[i] = new StructPollfd();
            pollFds[i].fd = fds.get(i);
            pollFds[i].events = (short) POLLIN;
        }
        try {
            Os.poll(pollFds, -1); // Linux poll机制，当有消息时唤醒
        } catch (ErrnoException ex) {
            throw new RuntimeException("poll failed", ex);
        }
        for (int i = pollFds.length - 1; i >= 0; --i) {
            if ((pollFds[i].revents & POLLIN) == 0) {
                continue;
            }
            if (i == 0) {
                ZygoteConnection newPeer = acceptCommandPeer(abiList);
                peers.add(newPeer);
                fds.add(newPeer.getFileDesciptor());
            } else {
                boolean done = peers.get(i).runOnce(); // ZygoteConnection::runOnce
                if (done) {
                    peers.remove(i);
                    fds.remove(i);
                }
            }
        }
    }
}
```

----------

## 3.1 ZygoteInit::handleSystemServerProcess
```
private static void handleSystemServerProcess(
        ZygoteConnection.Arguments parsedArgs)
        throws ZygoteInit.MethodAndArgsCaller {

    closeServerSocket(); // 关闭从zygote CopyonWrite来的Socket

    // set umask to 0077 so new files and directories will default to owner-only permissions.
    Os.umask(S_IRWXG | S_IRWXO);

    if (parsedArgs.niceName != null) {
        Process.setArgV0(parsedArgs.niceName); // 设置进程name
    }

    final String systemServerClasspath = Os.getenv("SYSTEMSERVERCLASSPATH");
    // /system/framework下的services.jar、ethernet-service.jar、wifi-service.jar做dex优化处理
    if (systemServerClasspath != null) {
        performSystemServerDexOpt(systemServerClasspath);
    }

    if (parsedArgs.invokeWith != null) { // 普通应用程序走这里
        String[] args = parsedArgs.remainingArgs;
        // If we have a non-null system server class path, we'll have to duplicate the
        // existing arguments and append the classpath to it. ART will handle the classpath
        // correctly when we exec a new process.
        if (systemServerClasspath != null) {
            String[] amendedArgs = new String[args.length + 2];
            amendedArgs[0] = "-cp";
            amendedArgs[1] = systemServerClasspath;
            System.arraycopy(parsedArgs.remainingArgs, 0, amendedArgs, 2, parsedArgs.remainingArgs.length);
        }

        WrapperInit.execApplication(parsedArgs.invokeWith,
                parsedArgs.niceName, parsedArgs.targetSdkVersion,
                VMRuntime.getCurrentInstructionSet(), null, args);
    } else {
        ClassLoader cl = null;
        if (systemServerClasspath != null) {
            cl = createSystemServerClassLoader(systemServerClasspath,
                                               parsedArgs.targetSdkVersion); // 3.2

            Thread.currentThread().setContextClassLoader(cl);
        }

        /*
         * Pass the remaining arguments to SystemServer.
         */
        RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs, cl); // 3.3
    }
}
```

## 3.2 ZygoteInit::createSystemServerClassLoader
```
private static PathClassLoader createSystemServerClassLoader(String systemServerClasspath,
                                                             int targetSdkVersion) {
  String librarySearchPath = System.getProperty("java.library.path"); // java.library.path=null

  return PathClassLoaderFactory.createClassLoader(systemServerClasspath, // system/framework/service.jar等
                                                  librarySearchPath, // null
                                                  null /* libraryPermittedPath */,
                                                  ClassLoader.getSystemClassLoader(), // 默认PathClassLoader
                                                  targetSdkVersion,
                                                  true /* isNamespaceShared */); // 返回 PathClassLoader
}
```

## 3.3 RuntimeInit::zygoteInit
```
public static final void zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
        throws ZygoteInit.MethodAndArgsCaller {
    redirectLogStreams(); // Redirect System.out and System.err to the Android log.

    commonInit(); // 设置未捕获异常、时区、重置log、网络统计等
    nativeZygoteInit(); // 启动Binder线程
    applicationInit(targetSdkVersion, argv, classLoader); // 3.4
}
```

## 3.4 RuntimeInit::applicationInit
```
private static void applicationInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
        throws ZygoteInit.MethodAndArgsCaller {
    ...
    // We want to be fairly aggressive about heap utilization, to avoid
    // holding on to a lot of memory that isn't needed.
    VMRuntime.getRuntime().setTargetHeapUtilization(0.75f);
    VMRuntime.getRuntime().setTargetSdkVersion(targetSdkVersion);
    final Arguments args = new Arguments(argv);
    // Remaining arguments are passed to the start class's static main
    invokeStaticMain(args.startClass, args.startArgs, classLoader); // 3.5
}
```

## 3.5 RuntimeInit::invokeStaticMain
```
private static void invokeStaticMain(String className, String[] argv, ClassLoader classLoader)
        throws ZygoteInit.MethodAndArgsCaller {
    Class<?> cl = Class.forName(className, true, classLoader); 
    Method m = cl.getMethod("main", new Class[] { String[].class });
    ...
    throw new ZygoteInit.MethodAndArgsCaller(m, argv); // 2.1
}
```
清空调用栈，使其看起来像直接进入main函数

----------

## 4.1 SystemServer::main
```
public static void main(String[] args) {
    new SystemServer().run();
}
```

## 4.2 SystemServer::run
```
private void run() {
    ...
    VMRuntime.getRuntime().clearGrowthLimit();

    // The system server has to run all of the time, so it needs to be
    // as efficient as possible with its memory usage.
    VMRuntime.getRuntime().setTargetHeapUtilization(0.8f);
    ...
    Looper.prepareMainLooper(); // 创建SystemServer主线程

    // Initialize native services.
    System.loadLibrary("android_servers");

    // Check whether we failed to shut down last time we tried.
    // This call may not return.
    performPendingShutdown();

    // Initialize the system context.
    createSystemContext(); // 4.3

    // Create the system service manager.
    mSystemServiceManager = new SystemServiceManager(mSystemContext);
    LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
    ...
    // Start services.
    try {
        startBootstrapServices(); // 5.1
        startCoreServices(); // 5.1
        startOtherServices(); // 5.1
    } catch (Throwable ex) {
        Slog.e("System", "******************************************");
        Slog.e("System", "************ Failure starting system services", ex);
        throw ex;
    }
    ...
    Looper.loop(); // 启动SystemServer主线程
    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

## 4.3 SystemServer::createSystemContext
```
    private void createSystemContext() {
        ActivityThread activityThread = ActivityThread.systemMain(); // 4.4 
        mSystemContext = activityThread.getSystemContext();
        mSystemContext.setTheme(DEFAULT_SYSTEM_THEME);
    }
```

## 4.4 ActivityThread::systemMain
```
public static ActivityThread systemMain() {
    ...
    ActivityThread thread = new ActivityThread();
    thread.attach(true); // 4.5 
    return thread;
}
```

## 4.5 ActivityThread::attach
```
private void attach(boolean system) {
    sCurrentActivityThread = this;
    mSystemThread = system;
    if (!system) {
    // 普通进程
    } else {
        android.ddm.DdmHandleAppName.setAppName("system_process",
                UserHandle.myUserId()); // 修改进程名字

        mInstrumentation = new Instrumentation();
        ContextImpl context = ContextImpl.createAppContext(
                this, getSystemContext().mPackageInfo); // 4.6
        mInitialApplication = context.mPackageInfo.makeApplication(true, null);
        mInitialApplication.onCreate(); 
    }
    ...
}
```
## 4.6 System Context创建流程
```
ActivityThread::getSystemContext
    ContextImpl::createSystemContext
        LoadedApk::LoadedApk
        new ContextImpl
```

## 4.7 LoadedApk构建流程
```
LoadedApk(ActivityThread activityThread) {
    mActivityThread = activityThread;
    mApplicationInfo = new ApplicationInfo();
    mApplicationInfo.packageName = "android"; // framework-res.apk
    mPackageName = "android";
    ...
    mClassLoader = ClassLoader.getSystemClassLoader(); 
    mResources = Resources.getSystem();
}
```

----------

## 5.1 SystemServer 服务启动流程

![SystemServer Service Start Phase](http://upload-images.jianshu.io/upload_images/74811-75db50ed42d148f0.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

启动SystemUI位于550-600之间,在AMS调用SystemReady过程中新启线程来执行的;

----------

## 6.1 ZygoteConnection::runOnce
``` 
boolean runOnce() throws ZygoteInit.MethodAndArgsCaller {

    String args[] = readArgumentList();

    int pid = -1;
    ...

    pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid, parsedArgs.gids,
            parsedArgs.debugFlags, rlimits, parsedArgs.mountExternal, parsedArgs.seInfo,
            parsedArgs.niceName, fdsToClose, parsedArgs.instructionSet,
            parsedArgs.appDataDir);
    ...
    if (pid == 0) {
        // in child
        IoUtils.closeQuietly(serverPipeFd);
        serverPipeFd = null;
        handleChildProc(parsedArgs, descriptors, childPipeFd, newStderr); // 6.2

        // should never get here, the child is expected to either
        // throw ZygoteInit.MethodAndArgsCaller or exec().
        return true;
    } else {
        // in parent...pid of < 0 means failure
        IoUtils.closeQuietly(childPipeFd);
        childPipeFd = null;
        return handleParentProc(pid, descriptors, serverPipeFd, parsedArgs);
    }
}
```
###### Zygote孵化新的进程
![Zygote孵化新的进程](http://upload-images.jianshu.io/upload_images/74811-e1df1a9f67edca68.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 6.2 ZygoteConnection::handleChildProc
```
private void handleChildProc(Arguments parsedArgs,
        FileDescriptor[] descriptors, FileDescriptor pipeFd, PrintStream newStderr)
        throws ZygoteInit.MethodAndArgsCaller {
    /**
     * By the time we get here, the native code has closed the two actual Zygote
     * socket connections, and substituted /dev/null in their place.  The LocalSocket
     * objects still need to be closed properly.
     */

    closeSocket();
    ZygoteInit.closeServerSocket(); // 关闭socket

    if (parsedArgs.niceName != null) {
        Process.setArgV0(parsedArgs.niceName);
    }

    // End of the postFork event.
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    if (parsedArgs.invokeWith != null) {
        WrapperInit.execApplication(parsedArgs.invokeWith,
                parsedArgs.niceName, parsedArgs.targetSdkVersion,
                VMRuntime.getCurrentInstructionSet(),
                pipeFd, parsedArgs.remainingArgs);
    } else {
        RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion,
                parsedArgs.remainingArgs, null /* classLoader */); // 3.3
    }
}
```
## 6.4 Java服务展现概览

![Java服务展现概览](http://upload-images.jianshu.io/upload_images/74811-9d1f7d829a0a3c81.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
###### 参考
[图解Android - Zygote, System Server 启动分析](http://www.cnblogs.com/samchen2009/p/3294713.html "图解Android - Zygote, System Server 启动分析")

## 7.1 Zygote 启动日志
```
$ adb logcat -s Zygote
--------- beginning of main
--------- beginning of system
01-01 08:01:21.453   284   284 D Zygote  : begin preload
01-01 08:01:21.453   284   284 I Zygote  : Installing ICU cache reference pinning...
01-01 08:01:21.453   284   284 I Zygote  : Preloading ICU data...
01-01 08:01:21.502   284   284 I Zygote  : Preloading ICU data --- End
01-01 08:01:21.503   284   284 I Zygote  : Preloading classes... // 加载class
01-01 08:01:22.557   284   284 I Zygote  : ...preloaded 4161 classes in 1054ms.
01-01 08:01:22.898   284   284 I Zygote  : Preloading resources...
01-01 08:01:23.279   284   284 I Zygote  : ...preloaded 114 resources in 381ms.
01-01 08:01:23.295   284   284 I Zygote  : ...preloaded 41 resources in 15ms.
01-01 08:01:23.296   284   284 I Zygote  : Preloading OpenGL...
01-01 08:01:23.465   284   284 I Zygote  : Preloading OpenGL --- End
01-01 08:01:23.465   284   284 I Zygote  : Preloading shared libraries...
01-01 08:01:23.472   284   284 I Zygote  : Preloading shared libraries --- End
01-01 08:01:23.472   284   284 I Zygote  : Preloading TextResources...
01-01 08:01:23.497   284   284 I Zygote  : Preloading TextResources --- End
01-01 08:01:23.499   284   284 I Zygote  : Uninstalled ICU cache reference pinning...
01-01 08:01:23.503   284   284 I Zygote  : Installed AndroidKeyStoreProvider in 4ms.
01-01 08:01:23.535   284   284 I Zygote  : Warmed up JCA providers in 32ms.
01-01 08:01:23.536   284   284 D Zygote  : end preload
01-01 08:01:23.676   284   284 I Zygote  : System server process 1022 has been created // SystemServer创建成功
01-01 08:01:23.681   284   284 I Zygote  : Accepting command socket connections // 监听通过Socket传递的请求
01-01 08:06:19.004  1022  1022 I Zygote  : Process: zygote socket opened, supported ABIS: armeabi-v7a,armeabi
```
## 7.2 SystemServer::startBootstrapServices
```
private void startBootstrapServices() {
    // Wait for installd to finish starting up so that it has a chance to
    // create critical directories such as /data/user with the appropriate
    // permissions.  We need this to complete before we initialize other services.
    Installer installer = mSystemServiceManager.startService(Installer.class);
    ...
    // Activity manager runs the show.
    mActivityManagerService = mSystemServiceManager.startService(
            ActivityManagerService.Lifecycle.class).getService();
    ...
    // Power manager needs to be started early because other services need it.
    // Native daemons may be watching for it to be registered so it must be ready
    // to handle incoming binder calls immediately (including being able to verify
    // the permissions for those calls).
    mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);

    // Manages LEDs and display backlight so we need it to bring up the display.
    mSystemServiceManager.startService(LightsService.class);

    // Display manager is needed to provide display metrics before package manager
    // starts up.
    mDisplayManagerService = mSystemServiceManager.startService(DisplayManagerService.class);

    // We need the default display before we can initialize the package manager.
    mSystemServiceManager.startBootPhase(SystemService.PHASE_WAIT_FOR_DEFAULT_DISPLAY);
    ...
    traceBeginAndSlog(""StartPackageManagerService""); // 7.3
}
```

## 7.3 System Service 启动日志
```
adb logcat -s SystemServer
--------- beginning of main
--------- beginning of system
01-01 08:01:43.754  1022  1022 I SystemServer: Entered the Android system server! // after zygote Accepting command socket connections
01-01 08:01:44.442  1022  1022 I SystemServer: StartPackageManagerService
01-01 08:04:20.324  1022  1022 I SystemServer: StartOtaDexOptService
01-01 08:04:20.462  1022  1022 I SystemServer: StartUserManagerService
01-01 08:04:20.517  1022  1022 I SystemServer: Reading configuration...
01-01 08:04:20.517  1022  1022 I SystemServer: StartSchedulingPolicyService
01-01 08:04:20.519  1022  1022 I SystemServer: StartTelephonyRegistry
01-01 08:04:20.524  1022  1022 I SystemServer: StartEntropyMixer
01-01 08:04:20.535  1022  1022 I SystemServer: Camera Service
01-01 08:04:20.538  1022  1022 I SystemServer: StartAccountManagerService
01-01 08:04:20.545  1022  1022 I SystemServer: StartContentService
01-01 08:04:20.548  1022  1022 I SystemServer: InstallSystemProviders
01-01 08:04:21.534  1022  1022 I SystemServer: StartVibratorService
01-01 08:04:21.539  1022  1022 I SystemServer: StartAlarmManagerService
01-01 08:04:21.565  1022  1022 I SystemServer: InitWatchdog
01-01 08:04:21.567  1022  1022 I SystemServer: StartInputManagerService
01-01 08:04:21.570  1022  1022 I SystemServer: StartWindowManagerService
01-01 08:04:21.680  1022  1022 I SystemServer: StartVrManagerService
01-01 08:04:21.812  1022  1022 I SystemServer: ConnectivityMetricsLoggerService
01-01 08:04:21.813  1022  1022 I SystemServer: PinnerService
01-01 08:04:21.890  1022  1022 I SystemServer: StartAccessibilityManagerService
01-01 08:06:12.596  1022  1022 I SystemServer: StartLockSettingsService
01-01 08:06:12.663  1022  1022 I SystemServer: StartStatusBarManagerService
01-01 08:06:12.668  1022  1022 I SystemServer: StartClipboardService
01-01 08:06:12.671  1022  1022 I SystemServer: StartNetworkManagementService
01-01 08:06:12.709  1022  1022 I SystemServer: StartNetworkScoreService
01-01 08:06:12.712  1022  1022 I SystemServer: StartNetworkStatsService
01-01 08:06:12.735  1022  1022 I SystemServer: StartNetworkPolicyManagerService
01-01 08:06:12.740  1022  1022 I SystemServer: No Wi-Fi NAN Service (NAN support Not Present)
01-01 08:06:13.042  1022  1022 I SystemServer: StartConnectivityService
01-01 08:06:13.101  1022  1022 I SystemServer: StartNsdService
01-01 08:06:13.107  1022  1022 I SystemServer: StartDataShapingService
01-01 08:06:13.110  1022  1022 I SystemServer: StartUpdateLockService
01-01 08:06:13.236  1022  1022 I SystemServer: StartLocationManagerService
01-01 08:06:13.242  1022  1022 I SystemServer: StartCountryDetectorService
01-01 08:06:13.243  1022  1022 I SystemServer: StartSearchManagerService
01-01 08:06:13.247  1022  1022 I SystemServer: Search Engine Service
01-01 08:06:13.252  1022  1022 I SystemServer: StartWallpaperManagerService
01-01 08:06:13.258  1022  1022 I SystemServer: StartAudioService
01-01 08:06:13.331  1022  1022 I SystemServer: StartWiredAccessoryManager
01-01 08:06:13.399  1022  1022 I SystemServer: StartSerialService
01-01 08:06:13.541  1022  1022 I SystemServer: Gesture Launcher Service
01-01 08:06:13.545  1022  1022 I SystemServer: StartDiskStatsService
01-01 08:06:13.546  1022  1022 I SystemServer: StartSamplingProfilerService
01-01 08:06:13.637  1022  1022 I SystemServer: StartNetworkTimeUpdateService
01-01 08:06:13.639  1022  1022 I SystemServer: StartCommonTimeManagementService
01-01 08:06:13.641  1022  1022 I SystemServer: CertBlacklister
01-01 08:06:13.645  1022  1022 I SystemServer: StartAssetAtlasService
01-01 08:06:13.691  1022  1022 I SystemServer: StartMediaRouterService
01-01 08:06:18.318  1022  1022 I SystemServer: StartBackgroundDexOptService
01-01 08:06:18.345  1022  1022 I SystemServer: RunningBoosterService
01-01 08:06:18.922  1022  1022 I SystemServer: Making services ready
01-01 08:06:19.056  1022  1022 I SystemServer: WebViewFactory preparation
01-01 00:00:00.216  1022  1022 I SystemServer: Enabled StrictMode for system server main thread.
```
