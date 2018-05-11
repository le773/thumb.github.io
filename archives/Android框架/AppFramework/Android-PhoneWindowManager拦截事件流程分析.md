#####  PhoneWindowManager初始化
```
wms::wms
    wms::initPolicy
        UiThread::Handler::runWithScissors
            PhoneWindowManager::init
```
##### interceptKeyBeforeQueueing 调用栈
```
InputDispatcher::injectInputEvent // 触发调用1
InputDispatcher::notifyKey // 触发调用2
    com_android_server_input_InputManagerService::NativeInputManager::interceptKeyBeforeQueueing
        IMS::WindowManagerCallbacks::interceptKeyBeforeQueueing // InputMontor是WindowManagerCallbacks的实现类
            InputMontor::interceptKeyBeforeQueueing
                WMS::PhoneWindowManager::interceptKeyBeforeQueueing
```

##### interceptKeyBeforeDispatching调用栈
```
InputDispatcher::dispatchOnce
    InputDispatcher::dispatchOnceInnerLocked(
        //InputDispatcher::mPolicy // mPolicy:: com_android_server_input_InputManagerService.cpp
        InputDispatcher::dispatchKeyLocked
            InputDispatcher::doInterceptKeyBeforeDispatchingLockedInterruptible
                com_android_server_input_InputManagerService::NativeInputManager::interceptKeyBeforeDispatching
                // jni InputManagerService::nativeInit中初始化
                    IMS::WindowManagerCallbacks::interceptKeyBeforeDispatching // InputMontor是WindowManagerCallbacks的实现类
                        InputMontor::interceptKeyBeforeDispatching
                            WMS::PhoneWindowManager::interceptKeyBeforeDispatching
```
PhoneWindowManager 相关类图

![PhoneWindowManager.png](http://upload-images.jianshu.io/upload_images/74811-6d23c91d8364f325.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

  1. InputMonitor 实现IMS::WindowManagerCallbacks接口，并且持有WMS引用；
  2. WMS持有WindowManagerPolicy接口的实现类PhoneWindowManager；
  3. PhoneWindowManager的内部类PolicyHandler分发业务逻辑；
  4. PhoneWindowManager的初始化在android.ui 线程；
