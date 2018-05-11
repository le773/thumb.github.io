##  1.1 Handler相关类图
![Handler相关类图](http://upload-images.jianshu.io/upload_images/74811-c0bb56f5adbbb17f.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


- 图中右边是`Android Handler`消息机制，左边衍生产品
- Looper持有`MessageQueue`，`MessageQueue`以单链表保存`Message`，`Message`保存发送消息的`Handler`对象，`Handler`持有`Looper`以及`Looper`的`Message`
- `Messager`可以实现跨进程通信
- `BlockingRunnable`是一个阻塞的队列，`Handler`发送消息之后，然后等待`task`执行完成通知，最后返回执行结果
## 2.1 Handler默认无参数构造函数
```
public Handler() {
    this(null, false);
}

public Handler(Callback callback, boolean async) {
    ...
    mLooper = Looper.myLooper(); // 通过Looper.prepare构造Looper 参考2.4
    if (mLooper == null) { // Looper 需要被提前创建
        throw new RuntimeException(...);
    }
    mQueue = mLooper.mQueue;
    mCallback = callback; // Handler 自身的CallBack
    mAsynchronous = async;
}
```
## 2.2 Handler有参默认数构造函数
```
 /*
 * @param looper The looper, must not be null.
 * @param callback The callback interface in which to handle messages, or null.
 * @param async If true, the handler calls {@link Message#setAsynchronous(boolean)} for
 * each {@link Message} that is sent to it or {@link Runnable} that is posted to it.
 */
public Handler(Looper looper, Callback callback, boolean async) {
    mLooper = looper;
    mQueue = looper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```

## 2.3 Handler回调接口 Handler.CallBack
```
/**
 * Callback interface you can use when instantiating a Handler to avoid
 * having to implement your own subclass of Handler.
 * /
Handler mHandler = new Handler(thread.getLooper(), mHandlerCallback); // 最终调用到 2.2

private Handler.Callback mHandlerCallback = new Handler.Callback() {
    @Override
    public boolean handleMessage(Message msg) {
    }
}
```

## 2.4 Looper::prepare
```
private static void prepare(boolean quitAllowed) {
    ... // 检查当前线程只能有一个Looper
    sThreadLocal.set(new Looper(quitAllowed)); // MainLooper 不允许退出，其它默认可以 参考2.5 
}
```

## 2.5 Looper构造函数
```
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed); // 为当前线程创建MessageQueue
    mThread = Thread.currentThread();
}
```
`Looper`持有所在线程的`MessageQueue`；


## 3.1 Handler发送消息
```
public final boolean post(Runnable r)
{
   return  sendMessageDelayed(getPostMessage(r)/*3.2*/, 0); // 3.4
}
```

## 3.2 通过Message的消息池生产消息
```
private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain(); // 3.3
    m.callback = r;
    return m;
}
```
## 3.3 Message::obtain
```
private static final int MAX_POOL_SIZE = 50; // 消息池最大数量
public static Message obtain() {
    synchronized (sPoolSync) {
        if (sPool != null) {
            Message m = sPool; //本质是由Message构成的链表
            sPool = m.next;
            m.next = null;
            m.flags = 0; // clear in-use flag
            sPoolSize--;
            /// M: Add message protect mechanism
            m.hasRecycle = false;
            return m;
        }
    }
    return new Message(); // 如果消息池消息已全部被使用，则创建Message
}
```
回收`Message`由`recycleUnchecked`处理，其对`Message`信息进行清理，然后添加到消息池所在的链表；

`Handler`通过`Message`消息池获取消息的方法还有`obtainMessage`和`postXxx`；

## 3.4 Handler::sendMessageAtTime
```
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    ... // 检查消息队列是否存在
    return enqueueMessage(queue, msg, uptimeMillis);
}
```
`post`和`sendMessage` 发送消息最终都交给`sendMessageAtTime`处理；
###### post与sendMessage区别
`post`通过消息池获取消息，并设置`Message`的回调(`callback`)为线程；
`sendMessage`发送的对象为`Message`;

## 3.5 Handler::enqueueMessage
```
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this; // Message的Handler
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis); // 3.6
}
```

## 3.6 MessageQueue::enqueueMessage
```
boolean enqueueMessage(Message msg, long when) {
    ... // 参数检查
    synchronized (this) {
        if (mQuitting) {
            ... // Handler正在退出
            return false;
        }

        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            // New head, wake up the event queue if blocked.
            // 创建头节点
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            // Inserted within the middle of the queue.  Usually we don't have to wake
            // up the event queue unless there is a barrier at the head of the queue
            // and the message is the earliest asynchronous message in the queue.
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) { // Message以when为key升序排列
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p; // invariant: p == prev.next
            prev.next = msg; // 单链表插入
        }

        // We can assume mPtr != 0 because mQuitting is false.
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```

## 4.1 Looper::loop
```
public static void loop() {
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    final MessageQueue queue = me.mQueue;
    ...
    for (;;) {
        Message msg = queue.next(); //4.2  might block 
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return; // 如果无消息，退出loop循环
        }
        ...
        msg.target.dispatchMessage(msg); // 4.3 分发消息
        ...
        msg.recycleUnchecked(); // 回收消息
    }
}
```

## 4.2 MessageQueue::next
```
Message next() {
    // Return here if the message loop has already quit and been disposed.
    // This can happen if the application tries to restart a looper after quit
    // which is not supported.
    final long ptr = mPtr;
    if (ptr == 0) {
        return null;
    }

    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    int nextPollTimeoutMillis = 0;
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }

        nativePollOnce(ptr, nextPollTimeoutMillis); //  might block 

        synchronized (this) {
            // Try to retrieve the next message.  Return if found.
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null && msg.target == null) {
                // Stalled by a barrier.  Find the next asynchronous message in the queue.
                do { // 查找下一个asynchronous信息
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) { // 消息链表头不为null
                if (now < msg.when) {
                    // Next message is not ready.  Set a timeout to wake up when it is ready.
                    // 阻塞nextPollTimeoutMillis，在尝试处理消息
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // Got a message.
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next; // 单链表中间位置删除
                    } else {
                        mMessages = msg.next; // 单链表头节点删除
                    }
                    msg.next = null;
                    msg.markInUse();
                    return msg;
                }
            } else {
                // No more messages.
                nextPollTimeoutMillis = -1; // 永远阻塞
            }

            // Process the quit message now that all pending messages have been handled.
            if (mQuitting) { // 正在退出
                dispose();
                return null;
            }
            // 此时没有要处理的消息，处理一些闲时任务
            // If first time idle, then get the number of idlers to run.
            // Idle handles only run if the queue is empty or if the first message
            // in the queue (possibly a barrier) is due to be handled in the future.
            if (pendingIdleHandlerCount < 0
                    && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size(); // 第一次进入next时，才进入此逻辑
            }
            if (pendingIdleHandlerCount <= 0) {
                // No idle handlers to run.  Loop and wait some more.
                mBlocked = true; // 没有闲时任务处理
                continue; // 下一次循环
            }

            if (mPendingIdleHandlers == null) {
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }

        // Run the idle handlers. 
        // We only ever reach this code block during the first iteration.
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; // release the reference to the handler

            boolean keep = false;
            ...
            keep = idler.queueIdle(); // 业务
            ...
            if (!keep) {
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }

        // Reset the idle handler count to 0 so we do not run them again.
        pendingIdleHandlerCount = 0;

        // While calling an idle handler, a new message could have been delivered
        // so go back and look again for a pending message without waiting.
        nextPollTimeoutMillis = 0;
    }
}
```

## 4.4 Handler::dispatchMessage
```
public void dispatchMessage(Message msg) {
    if (msg.callback != null) { // Message自己的回调函数，参考3.5
        handleCallback(msg); 
    } else {
        if (mCallback != null) { // 可以是Handler.CallBack 参考2.3
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg); // Handler的重载类
    }
}
```
Handler enqueueMessage的时候为Message设置target，参考3.5

## 5.1 Android Handler消息机制总结
![Handler消息循环概览](http://upload-images.jianshu.io/upload_images/74811-e863d8c53b4eaf2e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 消息被发送到`Looper`所在线程的`MessageQueue`;
- `MessageQueue`会不断循环取消息;
- `Handler`消息是多生产者与单消费者模型，消息由源消息发送者(`Handler`)进行分发；
- `Handler`首先由`Message.Callback`处理，其次`Handler`的回调，通常可以是`Handler.Callback`接口的实现类，最后是重写`Handler::handleMessage`的类；

## 6.1 Messager 构造函数
```
public Messenger(Handler target) {
    mTarget = target.getIMessenger();
}

public Messenger(IBinder target) {
    mTarget = IMessenger.Stub.asInterface(target);
}
```

## 6.2 获取MessengerImpl
```
final IMessenger getIMessenger() {
    synchronized (mQueue) {
        if (mMessenger != null) {
            return mMessenger;
        }
        mMessenger = new MessengerImpl();
        return mMessenger;
    }
}
```
## 6.3 Messager::send
```
public void send(Message message) throws RemoteException {
    mTarget.send(message); // 回调 replyTo::Messenger 
}
```
## 6.4 Messager::send
```
private final class MessengerImpl extends IMessenger.Stub {
    public void send(Message msg) {
        msg.sendingUid = Binder.getCallingUid();
        Handler.this.sendMessage(msg); // 消息由MessengerImpl的外部Handler类处理 参考3.4
    }
}
```

## 7.1 Handler::runWithScissors
```
public final boolean runWithScissors(final Runnable r, long timeout) {
    ... 参数检查
    if (Looper.myLooper() == mLooper) { // Looper所在线程和当前线程为同一个，直接运行
        r.run(); // 7.3
        return true;
    }

    BlockingRunnable br = new BlockingRunnable(r);
    return br.postAndWait(this, timeout); // this为当前Hander 参考 7.2
}
```
###### 使用场景
```
// WindowMS
UiThread.getHandler().runWithScissors(new Runnable())..
DisplayThread.getHandler().runWithScissors(new Runnable())..
```
## 7.2 BlockingRunnable::postAndWait
```
public boolean postAndWait(Handler handler, long timeout) {
    if (!handler.post(this)) { // 向handler所持有的MessageQueue发送消息 参考3.1
        return false;// 发送失败
    }

    synchronized (this) {
        if (timeout > 0) {
            final long expirationTime = SystemClock.uptimeMillis() + timeout;
            while (!mDone) {
                long delay = expirationTime - SystemClock.uptimeMillis();
                if (delay <= 0) {
                    return false; // timeout
                }
                try {
                    wait(delay); // 发送成功之后等待delay时间
                } catch (InterruptedException ex) {
                }
            }
        } else {
            while (!mDone) {
                try {
                    wait(); // 发送成功之后等待
                } catch (InterruptedException ex) {
                }
            }
        }
    }
    return true;
}
```

## 7.3 BlockingRunnable::run
```
public void run() {
    try {
        mTask.run(); // 执行业务
    } finally {
        synchronized (this) {
            mDone = true;
            notifyAll(); // 接着postAndWait返回true
        }
    }
}
```
