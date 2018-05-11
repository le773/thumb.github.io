# 1. AsyncTask 框架图 #



![AsyncTask框架.jpg](http://upload-images.jianshu.io/upload_images/74811-d29222d59332b5c2.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


a.  线程相关接口
`Callable` 表示的任务可以抛出受检查的或未受检查的异常，异常被封装为`ExecutionException中`，并在`Future::get()`中重新抛出；
`Future` 表示一个任务的生命周期，并提供了相应的方法来判断是否已经完成或取消，以及获取任务的结果和取消任务。
`RunnableFuture` 是接口类，继承`Runnable、Future`, 既包含抽象业务接口run,又添加监控线程生命周期方法；

b. 线程实现类
`FutureTask` 表示一种抽象的可生成结果的计算；可以处于等待运行、正在运行、运行完成；
运行完成 表示计算的所有可能结束方式，包括正常结束、由于取消、异常结束结束；
`WorkerRunnable` 保存`AsyncTask execute`传递的的具体业务的参数, 重写`Callable call`方法，调用`doBackground`执行业务处理；

c. 线程池相关类
Executor 线程池接口
`SerialExecutor`是`AsyncTas`k默认的线程池类，作用使用`ArrayDeque`排队任务，具体由**ThreadPoolExecutor处理业务**；


d. 消息处理类
`InternalHandler` 持有主线程的消息队列

## 2.1 AsyncTask初始化 ##
AsyncTask构造函数
```
public AsyncTask() {
    mWorker = new WorkerRunnable<Params, Result>() {
        public Result call() throws Exception {
            mTaskInvoked.set(true);
            Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
            Result result = doInBackground(mParams); // a. 业务处理，并返回结果 b. 调用publishProgress返回实时进度
            Binder.flushPendingCommands();
            return postResult(result); // 通过InternalHandler发送消息 MESSAGE_POST_RESULT 通知并更新UI主线程 参考2.7
        }
    };

    mFuture = new FutureTask<Result>(mWorker) {
        @Override
        protected void done() {
            try {
                postResultIfNotInvoked(get()/*2.8 参考FutureTask::get()*/); // 先get结果，然后通过postResult更新UI
            } catch (InterruptedException e) {
            } catch (CancellationException e) {
                postResultIfNotInvoked(null);
            }
        }
    };
}
```
a. 创建`WorkerRunnable`用于保存任务的参数（`params`）；
b. `FutureTask`封装`WorkerRunable`, 可获取任务的生命周期

## 2.2 AsyncTask::execute() ##
```
private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR; // 2.4 AsyncTask::SerialExecutor
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
    return executeOnExecutor(sDefaultExecutor, params); // 2.3
}
```

其中，`sDefaultExecutor`为`SerialExecutor`， `SerialExecutor`中`ArrayDeque`保存当前`AsyncTask`的任务队列；

## 2.3 AsyncTask::executeOnExecutor ##
```
public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
        Params... params) {
    if (mStatus != Status.PENDING) {
        switch (mStatus) {
            case RUNNING:
            case FINISHED:
        } // 检查线程所处的执行状态，只有PENDING才可以向下执行
    }
    mStatus = Status.RUNNING;
    onPreExecute(); // 任务执行前业务逻辑

    mWorker.mParams = params;
    exec.execute(mFuture); // 2.4  AsyncTask::SerialExecutor::execute()

    return this;
}
```

## 2.4 AsyncTask::SerialExecutor ##
```
private static final BlockingQueue<Runnable> sPoolWorkQueue = new LinkedBlockingQueue<Runnable>(128);
THREAD_POOL_EXECUTOR = new ThreadPoolExecutor(.., sPoolWorkQueue, .. );
private static class SerialExecutor implements Executor {
    final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>(); // 保存任务
    Runnable mActive;

    public synchronized void execute(final Runnable r) {
        mTasks.offer(new Runnable() {
            public void run() {
                try {
                    r.run();
                } finally {
                    scheduleNext();
                }
            }
        });
        if (mActive == null) {
            scheduleNext(); 
        }
    }

    protected synchronized void scheduleNext() {
        if ((mActive = mTasks.poll()) != null) { // 业务逻辑的处理交给线程池ThreadPoolExecutor处理
            THREAD_POOL_EXECUTOR.execute(mActive); // 2.5 FutureTask::run()
        }
    }
}
```
## 2.5 FutureTask::run() ##
```
public void run() {
    if (state != NEW ||
        !U.compareAndSwapObject(this, RUNNER, null, Thread.currentThread()))
        return;
    try {
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
                result = c.call(); // 回调WorkerRunnable::run(),参考2.1 AsyncTask初始化 ,
                // 从此处可分析：业务执行是由doInBackground完成
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                setException(ex);
            }
            if (ran)
                set(result); // 保存结果，通过 get->report 可获取
        }
    } finally {
        // runner must be non-null until state is settled to
        // prevent concurrent calls to run()
        runner = null;
        // state must be re-read after nulling runner to prevent
        // leaked interrupts
        int s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}
```

## 2.6 InternalHandler ## 
```
private static class InternalHandler extends Handler {
    public InternalHandler() {
        super(Looper.getMainLooper()); // 获取UI主线程MessageQueue
    }

    @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
    @Override
    public void handleMessage(Message msg) {
        AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
        switch (msg.what) {
            case MESSAGE_POST_RESULT:
                // There is only one result
                result.mTask.finish(result.mData[0]); // 任务执行完成，调用当前AsyncTask的finish方法
                break;
            case MESSAGE_POST_PROGRESS:
                result.mTask.onProgressUpdate(result.mData); // 更新UI
                break;
        }
    }
}
```

## 2.7 通知并更新消息 postResult ##
```
private Result postResult(Result result) {
    @SuppressWarnings("unchecked")
    Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
            new AsyncTaskResult<Result>(this/*当前AsyncTask对象*/, result));
    message.sendToTarget();
    return result;
}
```


## 2.8 AsyncTask::get() ##
```
public final Result get() throws InterruptedException, ExecutionException {
    return mFuture.get();
}
```
`FutureTask`回调`call`执行完成之后，将结果保存在`outcome`, `FutureTask`的`get`取得此结果
