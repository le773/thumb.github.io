## Context框架
![Context框架](http://upload-images.jianshu.io/upload_images/1187237-1b4c0cd31fd0193f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
Android程序Context的数量= Activity + Service + Application
ContextThemeWrapper、ContextWrapper功能委托给ContextImpl实现；

## 1. ContextImpl::getSystemService
```
public Object getSystemService(String name) {
    return SystemServiceRegistry.getSystemService(this, name);
}

public String getSystemServiceName(Class<?> serviceClass) { // 2
    return SystemServiceRegistry.getSystemServiceName(serviceClass);
}
```

## 2. SystemServiceRegistry::getSystemService 
```
public static Object getSystemService(ContextImpl ctx, String name) {
    ServiceFetcher<?> fetcher = SYSTEM_SERVICE_FETCHERS.get(name);
    return fetcher != null ? fetcher.getService(ctx) : null;
}

public static String getSystemServiceName(Class<?> serviceClass) {
    return SYSTEM_SERVICE_NAMES.get(serviceClass); // 4
}
```

## 3. SystemServiceRegistry::SYSTEM_SERVICE_NAMES初始化
```
static {
    ...
    registerService(Context.ACTIVITY_SERVICE, ActivityManager.class,
            new CachedServiceFetcher<ActivityManager>() {
        @Override
        public ActivityManager createService(ContextImpl ctx) {
            return new ActivityManager(ctx.getOuterContext(), ctx.mMainThread.getHandler());
        }});
    ...
}
```
static块，会在类被加载的时候执行且仅会被执行一次，一般用来初始化静态变量和调用静态方法。

## 4. SystemServiceRegistry::registerService
```
private static <T> void registerService(String serviceName, Class<T> serviceClass,
        ServiceFetcher<T> serviceFetcher) {
    SYSTEM_SERVICE_NAMES.put(serviceClass, serviceName);
    SYSTEM_SERVICE_FETCHERS.put(serviceName, serviceFetcher);
}
```

## 5. SystemServiceRegistry::SYSTEM_SERVICE_FETCHERS 保存服务引用且final
```
// Service registry information.
// This information is never changed once static initialization has completed.
private static final HashMap<String, ServiceFetcher<?>> SYSTEM_SERVICE_FETCHERS =
        new HashMap<String, ServiceFetcher<?>>();
private static final HashMap<Class<?>, String> SYSTEM_SERVICE_NAMES =
        new HashMap<Class<?>, String>();
```
