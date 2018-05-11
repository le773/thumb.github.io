## 1.1 ScheduledThreadPool框架图

![ScheduledThreadPool框架图.jpg](http://upload-images.jianshu.io/upload_images/74811-f5dd0f545aa5a0e8.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 2.1 newScheduledThreadPool 可指定核心线程数
```
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize/*指定核心线程个数*/); // 2.2 
}
```
创建可定时执行或周期执行任务的线程池。

## 2.2 ScheduledThreadPoolExecutor
```
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize/*核心线程数*/, Integer.MAX_VALUE/*线程池最大大小*/,
          DEFAULT_KEEPALIVE_MILLIS/*10L*/, MILLISECONDS,
          new DelayedWorkQueue());
}
```
DEFAULT_KEEPALIVE_MILLIS:非核心线程存活时间，默认10s

## 2.3.1 添加任务 schedule
```
public ScheduledFuture<?> schedule(Runnable command,
                                   long delay,
                                   TimeUnit unit) {
    RunnableScheduledFuture<Void> t = decorateTask(command, // 2.5 decorateTask
        new ScheduledFutureTask<Void>(command, null,
                                      triggerTime(delay, unit))); // 2.4 ScheduledFutureTask初始化
    delayedExecute(t); // 2.5 delayedExecute 添加到任务队列
    return t;
}
```
## 2.3.2 添加任务首次然后执行，然后周期循环 scheduleAtFixedRate
```
public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                              long initialDelay/*第一次延迟时间*/,
                                              long period/*周期*/,
                                              TimeUnit unit) {
    if (command == null || unit == null)
        throw new NullPointerException();
    if (period <= 0)
        throw new IllegalArgumentException();
    ScheduledFutureTask<Void> sft =
        new ScheduledFutureTask<Void>(command,
                                      null,
                                      triggerTime(initialDelay, unit), // 下次触发时间
                                      unit.toNanos(period));
    RunnableScheduledFuture<Void> t = decorateTask(command, sft); // 返回RunnableScheduledFuture
    sft.outerTask = t;
    delayedExecute(t); // 2.5 delayedExecute 添加到任务队列
    return t;
}
```


## 2.4 ScheduledFutureTask初始化
```
ScheduledFutureTask(Runnable r, V result, long triggerTime, long period) {
    super(r, result);
    this.time = triggerTime; // 触发时间
    this.period = period; // 执行周期
    this.sequenceNumber = sequencer.getAndIncrement();

----------

}
```

## 2.5 delayedExecute 添加到任务队列
```
private void delayedExecute(RunnableScheduledFuture<?> task) {
    if (isShutdown()) // 线程池关闭
        reject(task);
    else {
        super.getQueue().add(task); // 2.2super.getQueue() 为 初始化参数传入的 DelayedWorkQueue
        if (isShutdown() &&
            !canRunInCurrentRunState(task.isPeriodic()) &&
            remove(task))
            task.cancel(false);
        else
            ensurePrestart(); // 2.6 
    }
}
```

## 2.6 ThreadPoolExecutor::ensurePrestart
```
void ensurePrestart() {
    int wc = workerCountOf(ctl.get());
    if (wc < corePoolSize)
        addWorker(null, true); // 核心线程
    else if (wc == 0)
        addWorker(null, false); // 非核心线程
}
```
根据核心线程是否已满，启动线程池中一个线程，保证线程池处于工作状态；

剩下的处理参考ThreadPoolExecutor源码分析
