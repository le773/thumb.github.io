## 1.1 IntentService相关类图
![IntentService相关类图](http://upload-images.jianshu.io/upload_images/74811-0174110398059384.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- `IntentService`是一个抽象类，通过继承此类重写`onHandlerIntent`实现业务逻辑；
- 业务在`HandlerThread`线程处理执行；
- `ServiceHandler`和`HandlerThread`持有同一个`MessageQueue`;
- `ServiceHandler`生命周期和`IntentService`一致；

## 2.1 IntentService::onCreate
```
public void onCreate() {
    // TODO: It would be nice to have an option to hold a partial wakelock
    // during processing, and to have a static startService(Context, Intent)
    // method that would launch the service & hand off a wakelock.

    super.onCreate();
    HandlerThread thread = new HandlerThread("IntentService[" + mName + "]"); 
    thread.start(); // 2.2

    mServiceLooper = thread.getLooper(); 
    mServiceHandler = new ServiceHandler(mServiceLooper); // ServiceHandler持有HandlerThread所在线程的MessageQueue
}
```

## 2.2 HandlerThread::run
```
int mPriority = Process.THREAD_PRIORITY_DEFAULT;
public void run() {
    mTid = Process.myTid();
    Looper.prepare();
    synchronized (this) {
        mLooper = Looper.myLooper();
        notifyAll(); // 消息机制创建完成之后，通知HandlerThread可以获取Looper
    }
    Process.setThreadPriority(mPriority);
    onLooperPrepared();
    Looper.loop();
    mTid = -1;
}
```
在`HandlerThread`所在线程创建`Handler`消息机制;

## 2.3 Service生命周期
![Service生命周期](http://upload-images.jianshu.io/upload_images/74811-066671cf09df8ed4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
由`Service`生命周期可知，下一步应该是`onStartCommand`;
## 2.4 IntentService::onStartCommand
```
public int onStartCommand(Intent intent, int flags, int startId) {
    onStart(intent, startId);
    return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
}
```

## 2.5 IntentService::onStart
```
public void onStart(Intent intent, int startId) {
	// 通过Handler消息池创建消息，并且Message默认Handler为ServiceHandler
    Message msg = mServiceHandler.obtainMessage(); 
    msg.arg1 = startId;
    msg.obj = intent;
    mServiceHandler.sendMessage(msg); // 2.6
}
```
## 2.6 ServiceHandler::handleMessage
```
public void handleMessage(Message msg) {
    onHandleIntent((Intent)msg.obj); // 抽象方法
    stopSelf(msg.arg1); // 任务完成之后通知AMS停止当前service
}
```

## 3.1 IntentService::onDestroy
```
public void onDestroy() {
    mServiceLooper.quit(); // Looper退出
}
```
