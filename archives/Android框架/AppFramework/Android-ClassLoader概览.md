## 1.1 类加载器相关类图

![ClassLoader相关类图](http://upload-images.jianshu.io/upload_images/74811-23a19eaa06a3e5f3.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1. `DexClassLoader` `PathClassLoader`自身并无逻辑处理，都继承自`BaseDexClassLoader`;
1. `BaseDexClassLoader` 持有`DexPathList`，保存所有的dex文件以及保存app和系统的native库;
1. `BootClassLoader`作为class的默认加载器;
1. `ApplicationLoaders`持有`PathClassLoader`;
1. `DexFile`定义了加载类的方法

#### 1.2.1  DexClassLoader PathClassLoader 区别
区别在于`PathClassLoader`不能直接从zip包中得到dex，因此只支持直接操作dex文件或者已经安装过的apk（因为安装过的apk在cache中存在缓存的dex文件）;
而`DexClassLoader`可以加载外部的apk、jar或dex文件，并且会在指定的outpath路径存放其dex文件。  

## 2.1 class的类加载器
```
// class.java
public ClassLoader getClassLoader() {
    if (isPrimitive()) {
        return null;
    }
    return (classLoader == null) ? BootClassLoader.getInstance() : classLoader; // 2.5
}
```

## 2.2 系统默认加载器
```
//ClassLoader.java
public static ClassLoader getSystemClassLoader() {
    return SystemClassLoader.loader;
}
```

## 2.3 SystemClassLoader::loader
```
public abstract class ClassLoader {
    static private class SystemClassLoader {
        public static ClassLoader loader = ClassLoader.createSystemClassLoader();
    }
}
```

## 2.4 ClassLoader::createSystemClassLoader
```
private static ClassLoader createSystemClassLoader() {
    String classPath = System.getProperty("java.class.path", ".");
    String librarySearchPath = System.getProperty("java.library.path", "");
    // 创建PathClassLoader
    return new PathClassLoader(classPath, librarySearchPath, BootClassLoader.getInstance()); // 2.5
}
```

## 2.5 BootClassLoader::getInstance
```
class BootClassLoader extends ClassLoader {
    ...
    private static BootClassLoader instance;
    ...
    public static synchronized BootClassLoader getInstance() {
        if (instance == null) {
            instance = new BootClassLoader();
        }

        return instance;
    }
}
```

## 3.1 应用默认的加载器
```
// ContextImpl
public ClassLoader getClassLoader() {
    return mPackageInfo != null ? mPackageInfo.getClassLoader() : ClassLoader.getSystemClassLoader(); 
}
```
`mPackageInfo`的类型为`LoadedApk`， `mPackageInfo.getClassLoader`最终调用到`createOrUpdateClassLoaderLocked`创建`ClassLoader`;

## 3.2 LoadedApk::createOrUpdateClassLoaderLocked
```
// LoadedApk.java
private void createOrUpdateClassLoaderLocked(List<String> addedPaths) {
    if (mPackageName.equals("android")) { // 针对SystemServer
        ...
        if (mBaseClassLoader != null) {
            mClassLoader = mBaseClassLoader;
        } else {
            mClassLoader = ClassLoader.getSystemClassLoader(); // 2.2
        }
        return;
    }
    ...
    if (mClassLoader == null) {
        ...
        mClassLoader = ApplicationLoaders.getDefault().getClassLoader(zip,
                mApplicationInfo.targetSdkVersion, isBundledApp, librarySearchPath,
                libraryPermittedPath, mBaseClassLoader);
        ...
    }
    ...
```

## 3.3 ApplicationLoaders::getClassLoader
```
private final ArrayMap<String, ClassLoader> mLoaders = new ArrayMap<String, ClassLoader>();
public ClassLoader getClassLoader(String zip, int targetSdkVersion, boolean isBundled,
                                  String librarySearchPath, String libraryPermittedPath,
                                  ClassLoader parent) {
    ClassLoader baseParent = ClassLoader.getSystemClassLoader().getParent();

    synchronized (mLoaders) {
        if (parent == null) {
            parent = baseParent;
        }
        /*
         * If we're one step up from the base class loader, find
         * something in our cache.  Otherwise, we create a whole
         * new ClassLoader for the zip archive.
         */
        if (parent == baseParent) {
            ClassLoader loader = mLoaders.get(zip);
            if (loader != null) {
                return loader;
            }
            ...
            PathClassLoader pathClassloader = PathClassLoaderFactory.createClassLoader(
                                                  zip,
                                                  librarySearchPath,
                                                  libraryPermittedPath,
                                                  parent,
                                                  targetSdkVersion,
                                                  isBundled); // 创建PathClassLoader
            ...
            setupVulkanLayerPath(pathClassloader, librarySearchPath);
            ...
            mLoaders.put(zip, pathClassloader);
            return pathClassloader;
        }
        PathClassLoader pathClassloader = new PathClassLoader(zip, parent);
        return pathClassloader;
    }
}
```
## 4.1 PathClassLoader加载器的构造函数
```
public class PathClassLoader extends BaseDexClassLoader {
    public PathClassLoader(String dexPath, ClassLoader parent) {
        super(dexPath, null, null, parent); // 4.3
    }
    /**
     * @param dexPath the list of jar/apk files containing classes and resources
     * @param librarySearchPath the list of directories containing native libraries
     * @param parent the parent class loader
     */
    public PathClassLoader(String dexPath, String librarySearchPath, ClassLoader parent) {
        super(dexPath, null, librarySearchPath, parent);
    }
}
```

## 4.2 DexClassLoader加载器的构造函数
```
public class DexClassLoader extends BaseDexClassLoader {
    /**
     * @param dexPath the list of jar/apk files containing classes and resources 加载apk、jar的路径
     * @param optimizedDirectory directory where optimized dex files // 优化的dex文件
     * @param librarySearchPath the list of directories containing native libraries // 本地lib库
     * @param parent the parent class loader
     */
    public DexClassLoader(String dexPath, String optimizedDirectory,
            String librarySearchPath, ClassLoader parent) {
        super(dexPath, new File(optimizedDirectory), librarySearchPath, parent); // optimizedDirectory不为空 4.3
    }
}
```

## 4.3 BaseDexClassLoader加载器的构造函数
```
public class BaseDexClassLoader extends ClassLoader {
    private final DexPathList pathList;
    /**
     * @param dexPath the list of jar/apk files containing classes and resources
     * @param optimizedDirectory directory where optimized dex files
     * @param librarySearchPath the list of directories containing native libraries
     * @param parent the parent class loader
     */
    public BaseDexClassLoader(String dexPath, File optimizedDirectory,
            String librarySearchPath, ClassLoader parent) {
        super(parent); // 4.4
        this.pathList = new DexPathList(this, dexPath, librarySearchPath, optimizedDirectory); // 4.5
    }
    ...
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        List<Throwable> suppressedExceptions = new ArrayList<Throwable>();
        Class c = pathList.findClass(name, suppressedExceptions);
        ...
        return c;
    }
}
```

## 4.4 ClassLoader加载器的构造函数
```
protected ClassLoader() {
    this(checkCreateClassLoader()/*default null */, getSystemClassLoader()/*2.2*/);
}
```

## 4.5 DexPathList构造函数

```
/**
 * @param definingContext the context in which any as-yet unresolved classes should be defined
 * @param dexPath list of dex/resource path elements apk jar路径
 * @param librarySearchPath list of native library directory path elements  本地lib库
 * @param libraryPermittedPath is path containing permitted directories for
 * linker isolated namespaces (in addition to librarySearchPath which is allowed
 * implicitly). Note that this path does not affect the search order for the library
 * and intended for white-listing additional paths when loading native libraries
 * by absolute path.
 * @param optimizedDirectory directory where optimized {@code .dex} files 优化的dex路径
 */
public DexPathList(ClassLoader definingContext, String dexPath,
        String librarySearchPath, File optimizedDirectory) {
    // 参数检查
    ...
    this.definingContext = definingContext;

    ArrayList<IOException> suppressedExceptions = new ArrayList<IOException>();
    // save dexPath for BaseDexClassLoader 所有dex文件
    this.dexElements = makeDexElements(splitDexPath(dexPath), optimizedDirectory,
                                       suppressedExceptions, definingContext);

    // Native libraries may exist in both the system and
    // application library paths, and we use this search order:
    //
    //   1. This class loader's library path for application libraries (librarySearchPath):
    //   1.1. Native library directories
    //   1.2. Path to libraries in apk-files
    //   2. The VM's library path from the system property for system libraries
    //      also known as java.library.path
    //
    // This order was reversed prior to Gingerbread; see http://b/2933456.
    this.nativeLibraryDirectories = splitPaths(librarySearchPath, false);
    this.systemNativeLibraryDirectories =
            splitPaths(System.getProperty("java.library.path"), true);
    List<File> allNativeLibraryDirectories = new ArrayList<>(nativeLibraryDirectories);
    //所有native库 allNativeLibraryDirectories = systemNativeLibraryDirectories + nativeLibraryDirectories
    allNativeLibraryDirectories.addAll(systemNativeLibraryDirectories);
    this.nativeLibraryPathElements = makePathElements(allNativeLibraryDirectories,
                                                      suppressedExceptions,
                                                      definingContext);
    ...
}
```
- dexElements保存所有的dex文件；
- nativeLibraryPathElements 保存app +系统native库

## 5.1 调试
```
Log.d(TAG, "----------------------- ");
ClassLoader classLoader = getClassLoader();
while(classLoader != null) {
    Log.d(TAG, "default class delegate " + classLoader);
    classLoader = classLoader.getParent();
}

Log.d(TAG, "----------------------- ");


Log.d(TAG, "----------------------- ");
ClassLoader sysLoader = ClassLoader.getSystemClassLoader();
while(sysLoader != null) {
    Log.d(TAG, "system class delegate " + sysLoader);
    sysLoader = sysLoader.getParent();
}
Log.d(TAG, "----------------------- ");

Log.d(TAG, "buttonHost");
Log.d(TAG, "button class loader " + Button.class.getClassLoader());
Log.d(TAG, "buttonHost");
Log.d(TAG, "context class loader " + Context.class.getClassLoader());
```
- 应用默认的`ClassLoader`是`PathClassLoader`；zip文件是`/data/app/android.dynamicloadhost-1/base.apk`等，**nativeLibraryDirectories**包含以下 当前包名 + 系统lib库下的so `/data/app/android.dynamicloadhost-1/lib/arm, /system/lib, /vendor/lib, /system/vendor/lib`
- 系统默认的`ClassLoader`是`PathClassLoader`；和应用默认加载器**nativeLibraryDirectories**区别是不包含app下的so；
- Button和Context代表的Class的类加载器是`BootClassLoader`，每个class含有自己的加载器；


## 5.2 日志

```
01-03 14:50:48.557 29926 29926 D MainActivity: -----------------------
01-03 14:50:48.557 29926 29926 D MainActivity: default class delegate dalvik.system.PathClassLoader[DexPathList[[zip file "/data/app/android.dynamicloadhost-1/base.apk", zip file "/data/app/android.dynamicloadhost-1/split_lib_dependencies_apk.apk", zip file "/data/app/android.dynamicloadhost-1/split_lib_slice_0_apk.apk", zip file "/data/app/android.dynamicloadhost-1/split_lib_slice_1_apk.apk", zip file "/data/app/android.dynamicloadhost-1/split_lib_slice_2_apk.apk", zip file "/data/app/android.dynamicloadhost-1/split_lib_slice_3_apk.apk", zip file "/data/app/android.dynamicloadhost-1/split_lib_slice_4_apk.apk", zip file "/data/app/android.dynamicloadhost-1/split_lib_slice_5_apk.apk", zip file "/data/app/android.dynamicloadhost-1/split_lib_slice_6_apk.apk", zip file "/data/app/android.dynamicloadhost-1/split_lib_slice_7_apk.apk", zip file "/data/app/android.dynamicloadhost-1/split_lib_slice_8_apk.apk", zip file "/data/app/android.dynamicloadhost-1/split_lib_slice_9_apk.apk"],nativeLibraryDirectories=[/data/app/android.dynamicloadhost-1/lib/arm, /system/lib, /vendor/lib, /system/vendor/lib]]]
01-03 14:50:48.557 29926 29926 D MainActivity: default class delegate java.lang.BootClassLoader@c142905
01-03 14:50:48.557 29926 29926 D MainActivity: -----------------------


01-03 14:50:48.557 29926 29926 D MainActivity: -----------------------
01-03 14:50:48.557 29926 29926 D MainActivity: system class delegate dalvik.system.PathClassLoader[DexPathList[[directory "."],nativeLibraryDirectories=[/system/lib, /vendor/lib, /system/vendor/lib, /system/lib, /vendor/lib, /system/vendor/lib]]]
01-03 14:50:48.557 29926 29926 D MainActivity: system class delegate java.lang.BootClassLoader@c142905
01-03 14:50:48.558 29926 29926 D MainActivity: -----------------------
01-03 14:50:48.558 29926 29926 D MainActivity: buttonHost
01-03 14:50:48.558 29926 29926 D MainActivity: button class loader java.lang.BootClassLoader@c142905
01-03 14:50:48.558 29926 29926 D MainActivity: buttonHost
01-03 14:50:48.558 29926 29926 D MainActivity: context class loader java.lang.BootClassLoader@c142905
```
