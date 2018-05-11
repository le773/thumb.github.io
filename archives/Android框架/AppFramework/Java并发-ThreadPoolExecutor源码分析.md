## 1. ThreadPoolExecutor相关框架图

![](http://upload-images.jianshu.io/upload_images/74811-eca76f1865324fa5.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

a. `Executors`提供`newCachedThreadPool`, `newFixedThreadPool`, `newScheduledThreadPool`, `newSingleThreadExecutor`创建创建线程池。
b. `RunnableAdapter`是`Executors`的内部类，提供`Runnable`转接为`Callable`的方法;
c. `DefaultThreadFactory`是`Executors`内部的线程工厂类, `PrivilegedThreadFactory`重写其`newThread`方法;
d. `Worker`重写AQS获取锁的方法,并包装业务逻辑; `ThreadPoolExecutor::workers`保存所有的工作业务, 当其满时缓存到`workQueue`;

## 2.1 线程池中断策略
```
ThreadPoolExecutor.AbortPolicy: 丢弃任务并抛出
RejectedExecutionException异常。默认
ThreadPoolExecutor.DiscardPolicy：也是丢弃任务，但是不抛出异常。
ThreadPoolExecutor.DiscardOldestPolicy：丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）
ThreadPoolExecutor.CallerRunsPolicy：由调用线程处理该任务
```

## 2.2 自定义中断策略
```
class SelfRejected extends RejectedExecutionHandler
```

## 2.3.1 线程池状态
```
//接收新任务，并且执行缓存任务队列中的任务
private static final int RUNNING    = -1

// 不在接收新的任务，但是会执行缓存中的任务
private static final int SHUTDOWN   =  0

// 不接收新的任务，不执行缓存中的任务，中断正在运行的任务
private static final int STOP       =  1

// 所有任务已经终止，workCount = 0;
private static final int TIDYING    =  2

// terminated 方法调用完成
private static final int TERMINATED =  3
```

## 2.3.2 线程池状态切换
```
RUNNING -> SHUTDOWN
 *    On invocation of shutdown(), perhaps implicitly in finalize()
(RUNNING or SHUTDOWN) -> STOP
 *    On invocation of shutdownNow() // shutdownNow
SHUTDOWN -> TIDYING
 *    When both queue and pool are empty // 缓存队列、线程池均空闲
STOP -> TIDYING
 *    When pool is empty
TIDYING -> TERMINATED
 *    When the terminated() hook method has completed

ctl
    The main pool control state, ctl, is an atomic integer packing two conceptual fields
workerCount
    indicating the effective number of threads // 有效线程数量
runState
    indicating whether running, shutting down etc // 运行状态
```


## 2.3.3 其他成员
```
// 缓存任务阻塞队列
private final BlockingQueue<Runnable> workQueue;

// 线程池主锁
private final ReentrantLock mainLock = new ReentrantLock();

// 工作线程
private final HashSet<Worker> workers = new HashSet<Worker>();

// mainLock上的终止条件量，用于支持awaitTermination
private final Condition termination = mainLock.newCondition();

// 记录曾经创建的最大线程数
private int largestPoolSize;

// 已经完成任务数
private long completedTaskCount;

private volatile ThreadFactory threadFactory;

// 设置默认任务拒绝策略
private static final RejectedExecutionHandler defaultHandler = new AbortPolicy();
```


## 3.1 Executors::newFixedThreadPool 创建固定大小线程池

newFixedThreadPool创建固定大小线程池，无非核心线程，超出的线程保存在LinkedBlockingQueue中等待；
```
public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
    return new ThreadPoolExecutor(nThreads, nThreads, // corePoolSize == maximumPoolSize, 无非核心线程
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>(),
                                  threadFactory);
}
```

## 3.2 ThreadPoolExecutor::execute
```
/**
 * Executes the given task sometime in the future.  The task
 * may execute in a new thread or in an existing pooled thread.
 *
 * If the task cannot be submitted for execution, either because this
 * executor has been shutdown or because its capacity has been reached,
 * the task is handled by the current {@code RejectedExecutionHandler}.
 */
public void execute(Runnable command) {
    /*
     * Proceed in 3 steps:
     *
     * 1. If fewer than corePoolSize threads are running, try to
     * start a new thread with the given command as its first
     * task.  The call to addWorker atomically checks runState and
     * workerCount, and so prevents false alarms that would add
     * threads when it shouldn't, by returning false.
     * 
     * 当工作者线程小于核心线程，则new thread, runnable作为其第一个task,
     * addWorker将检查线程运行时状态 和 工作者总数，
     * 目的是： 当不允许添加线程时，防止假唤醒；
     * 
     * 2. If a task can be successfully queued, then we still need
     * to double-check whether we should have added a thread
     * (because existing ones died since last checking) or that
     * the pool shut down since entry into this method. So we
     * recheck state and if necessary roll back the enqueuing if
     * stopped, or start a new thread if there are none.
     * 
     * 当核心线程已满，线程池处于运行时状态，则向缓存队列workQueue添加；
     *    此时，需要double-check
     *    因为，当调用此方法后线程池可能被shutdown、 线程died
     * 如果，工作者线程为0，添加一个null task的worker
     * 3. If we cannot queue task, then we try to add a new
     * thread.  If it fails, we know we are shut down or saturated
     * and so reject the task.
     * 此时，缓存队列已满，则创建一个线程，添加到works
     */
    int c = ctl.get();
    // 1. 如果工作线程数小于核心线程数，则添加新的线程
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true)) // 3.3.1 
            return; // 添加成功则返回
        c = ctl.get(); // 否则获取线程池状态
    }
    // 2. 工作线程数大于等于核心线程数，则将任务放入缓存任务队列
    // 如果线程池正在运行，而且成功将任务插入缓存任务队列两个条件
    if (isRunning(c) && workQueue.offer(command)) { // 第一次检查
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command)) // 第二次检查
            reject(command); // 如果不处于运行状态，则将任务从任务缓存队列移除
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false); //启动无初始任务的非核心线程 // 3.3.1
    }
    // 3. 任务入队失败，说明任务缓存任务队列已满，尝试添加新的线程处理
    else if (!addWorker(command, false)) // 3.3.1
        reject(command);
}
```

## 3.3.1 ThreadPoolExecutor::addWorker
```
    /**
     * Checks if a new worker can be added with respect to current
     * pool state and the given bound (either core or maximum). If so,
     * the worker count is adjusted accordingly, and, if possible, a
     * new worker is created and started, running firstTask as its
     * first task. This method returns false if the pool is stopped or
     * eligible to shut down. It also returns false if the thread
     * factory fails to create a thread when asked.  If the thread
     * creation fails, either due to the thread factory returning
     * null, or due to an exception (typically OutOfMemoryError in
     * Thread.start()), we roll back cleanly.
     *
     * @param firstTask the task the new thread should run first (or
     * null if none). Workers are created with an initial first task
     * (in method execute()) to bypass queuing when there are fewer
     * than corePoolSize threads (in which case we always start one),
     * or when the queue is full (in which case we must bypass queue).
     * Initially idle threads are usually created via
     * prestartCoreThread or to replace other dying workers.
     *
     * @param core if true use corePoolSize as bound, else
     * maximumPoolSize. (A boolean indicator is used here rather than a
     * value to ensure reads of fresh values after checking other pool
     * state).
     * @return true if successful
     */
    private boolean addWorker(Runnable firstTask, boolean core) {
        // 对 runState进行循环获取和判断，如果不满足添加条件则返回false
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c); // 线程池状态

            // Check if queue empty only if necessary.
            // 对线程池状态进行判断，是否适合添加新的线程
            if (rs >= SHUTDOWN && // 线程池关闭
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                // CAS 设置成功，则跳出最外层循环
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                // 如果workerCount没有设置成功，而且runState发生变化，
                // 则继续最外层的循环，对runState重新获取和判断
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask); // 3.4
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    // 获取锁后，对runState进行再次检查
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w); // workers是HashSet<Worker>, 包含池中所有worker, 只有当持有才可mainlock访问。
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    t.start(); // 3.4 如果添加成功，则启动线程，执行worker::run
                    workerStarted = true;
                }
            }
        } finally { // 因为中途发生异常而没有让添加的线程启动，则回滚
            if (! workerStarted)
                addWorkerFailed(w); // 3.7 
        }
        return workerStarted;
    }
```
1. 在外循环对运行状态进行判断，内循环通过CAS机制对`workerCount`进行增加，当设置成功，则跳出外循环，否则进行进行内循环重试;
2. 外循环之后，获取全局锁，再次对运行状态进行判断，符合条件则添加新的工作线程，并启动工作线程，如果在最后对添加线程没有开始运行（可能发生内存溢出，操作系统无法分配线程等等）则对添加操作进行回滚，移除之前添加的线程;

## 3.3.2 线程池添加任务流程

![线程池添加任务流程](http://upload-images.jianshu.io/upload_images/74811-232e007b51c6b9ca.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

核心线程 -> 工作队列 -> 线程池 -> 饱和策略

## 3.4 worker 线程封装类
```
private final class Worker extends AbstractQueuedSynchronizer implements Runnable
{
    Worker(Runnable firstTask) { // 5. worker框架参考
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this); // 新建thread to save runable
    }

    /** Delegates main run loop to outer runWorker. */
    public void run() {
        runWorker(this); // 3.5 工作线程主函数
    }

    // 同步锁，重写AQS中相关锁的方法
    protected boolean tryAcquire(int unused) {
        if (compareAndSetState(0, 1)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }
    protected boolean tryRelease(int unused) {
        setExclusiveOwnerThread(null);
        setState(0);
        return true;
    }
    public void lock()        { acquire(1); }
    public boolean tryLock()  { return tryAcquire(1); }
    public void unlock()      { release(1); }
    public boolean isLocked() { return isHeldExclusively(); }
}
```

## 3.5 工作线程主函数 ThreadPoolExecutor::runWorker
```
/**
* Main worker run loop.  Repeatedly gets tasks from queue and
* executes them, while coping with a number of issues:
*
* 1. We may start out with an initial task, in which case we
* don't need to get the first one. Otherwise, as long as pool is
* running, we get tasks from getTask. If it returns null then the
* worker exits due to changed pool state or configuration
* parameters.  Other exits result from exception throws in
* external code, in which case completedAbruptly holds, which
* usually leads processWorkerExit to replace this thread.
*
* 2. Before running any task, the lock is acquired to prevent
* other pool interrupts while the task is executing, and then we
* ensure that unless pool is stopping, this thread does not have
* its interrupt set.
*
* 3. Each task run is preceded by a call to beforeExecute, which
* might throw an exception, in which case we cause thread to die
* (breaking loop with completedAbruptly true) without processing
* the task.
*
* 4. Assuming beforeExecute completes normally, we run the task,
* gathering any of its thrown exceptions to send to afterExecute.
* We separately handle RuntimeException, Error (both of which the
* specs guarantee that we trap) and arbitrary Throwables.
* Because we cannot rethrow Throwables within Runnable.run, we
* wrap them within Errors on the way out (to the thread's
* UncaughtExceptionHandler).  Any thrown exception also
* conservatively causes thread to die.
*
* 5. After task.run completes, we call afterExecute, which may
* also throw an exception, which will also cause thread to
* die. According to JLS Sec 14.20, this exception is the one that
* will be in effect even if task.run throws.
*
* The net effect of the exception mechanics is that afterExecute
* and the thread's UncaughtExceptionHandler have as accurate
* information as we can provide about any problems encountered by
* user code.
*
* @param w the worker
*/
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        // 获取任务，如果没有任务可以获取，则此循环终止，
        // 这个工作线程将结束工作，等待被清理前的登记工作
        while (task != null || (task = getTask()) != null) { // 3.8 getTask获取任务
            w.lock(); // 执行任务之前获取工作线程锁
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt

            // 如果线程池关闭，则确保线程被中断
            // 如果线程池没有关闭，则确保线程不被中断
            // 这就要求在第二种情况下，进行重新检查，处理shutdownNow正在运行同时清除中断
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task); // 执行前业务处理
                try {
                    task.run(); // 业务处理
                } catch (RuntimeException x) {
                    ...
                } finally {
                    afterExecute(task, thrown);  // 执行后业务处理
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        // 登记信息，移除结束线程，然后根据情况添加新的线程等
        processWorkerExit(w, completedAbruptly); // 3.6
    }
}
```

## 3.6 ThreadPoolExecutor::processWorkerExit
```
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
        decrementWorkerCount();

    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        completedTaskCount += w.completedTasks;
        workers.remove(w); // 释放worker
    } finally {
        mainLock.unlock();
    }

    tryTerminate(); // 终止线程
}
```
## 3.7 ThreadPoolExecutor::addWorkerFailed
```
/**
 * Rolls back the worker thread creation.
 * - removes worker from workers, if present
 * - decrements worker count
 * - rechecks for termination, in case the existence of this
 *   worker was holding up termination
 */
private void addWorkerFailed(Worker w) {
    // 对线程回滚需要获取全局锁
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        if (w != null)
            workers.remove(w);
        decrementWorkerCount(); // 减少工作线程计数
        tryTerminate(); // 尝试终止线程
    } finally {
        mainLock.unlock();
    }
}
```

## 3.8 ThreadPoolExecutor::getTask
```
    /**
     * Performs blocking or timed wait for a task, depending on
     * current configuration settings, or returns null if this worker
     * must exit because of any of: 
     * //退出业务处理
     * 1. There are more than maximumPoolSize workers (due to
     *    a call to setMaximumPoolSize).
     * 2. The pool is stopped.
     * 3. The pool is shutdown and the queue is empty.
     * 4. This worker timed out waiting for a task, and timed-out
     *    workers are subject to termination (that is,
     *    {@code allowCoreThreadTimeOut || workerCount > corePoolSize})
     *    both before and after the timed wait, and if the queue is
     *    non-empty, this worker is not the last thread in the pool.
     */
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount(); //减少workerCount数量
                return null;
            }

            int wc = workerCountOf(c);

            // Are workers subject to culling?
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
            // 包含对超时的判断，如果发生超时，则说明该worker已经空闲了
            // keepAliveTime时间，则应该返回null，这样会使工作线程正常结束，
            // 并被移除
            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take(); // workQueue中keepAliveTime内获取任务
                // 如果在keepAliveTime时间内获取到任务则返回
                if (r != null)
                    return r;
                timedOut = true; // 否则将超时标志设置为true
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```

## 4.1 Executors::newCachedThreadPool
```
public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory) {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE, // 核心线程为0
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>(), // SynchronousQueue
                                  threadFactory);
}
```

## 4.2 Executors::newSingleThreadExecutor
`FinalizableDelegatedExecutorService` 内部封装`ThreadPoolExecutor`，且只有一个核心线程；
```
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1/*corePoolSize*/, 1/*maximumPoolSize*/, // corePoolSize == maximumPoolSize == 1
                                0L/*keepAliveTime*/, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```

## 4.3 Executors::FinalizableDelegatedExecutorService
```
private static class FinalizableDelegatedExecutorService
        extends DelegatedExecutorService {
    FinalizableDelegatedExecutorService(ExecutorService executor) {
        super(executor);
    }
    protected void finalize() {
        super.shutdown();
    }
}
```
`FinalizableDelegatedExecutorService`继承自`DelegatedExecutorService`，并实现`finalize`方法；
`DelegatedExecutorService` 封装了`ThreadPoolExecutor`



## 5. Worker 相关框架

![Worker 相关框架](http://upload-images.jianshu.io/upload_images/74811-67fee4247aa45930.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

a. `AbstractQueuedSynchronizer` 提供了一个基于FIFO队列，可以用于构建锁或者其他相关同步装置的基础框架；
b. Node是`AbstractQueuedSynchronizer`的内部类，Node保存着线程引用和线程状态的容器，每个线程对同步器的访问，都可以看做是队列中的一个节点；
c. Worker 实现`AbstractQueuedSynchronizer`中`acquire`相关方法；

AQS 中Node节点

![AQS::Node](http://upload-images.jianshu.io/upload_images/74811-3dd93be82ee7e447.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 参考：

[AbstractQueuedSynchronizer的介绍和原理分析](http://ifeve.com/introduce-abstractqueuedsynchronizer/ "AbstractQueuedSynchronizer的介绍和原理分析")
[Java线程池分析](http://gityuan.com/2016/01/16/thread-pool/ "Java线程池分析")
