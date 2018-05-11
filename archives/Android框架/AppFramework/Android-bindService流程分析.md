## 1.1 Service相关数据结构
![Service相关数据结构](http://upload-images.jianshu.io/upload_images/74811-f41c0457323f7b23.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 1.2 Service Bind & Unbind流程
![BindService相关流程图](http://upload-images.jianshu.io/upload_images/74811-4e3b82e06cacb32a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 2.1 bindServiceLocked 解析
###### bindServiceLocked
- 通过`retrieveServiceLocked()`, 根据用户传递进来`Intent`来检索相对应的服务
- 通过`retrieveAppBindingLocked()`。创建`AppBindRecord`对象记录着当前`ServiceRecord`, `intent`以及发起方的进程信息。
- 通过`bringUpServiceLocked()`拉起目标服务;

## 2.2 retrieveServiceLocked
查找相应的`servicerecord`，如果没有则创建；

## 2.3 retrieveAppBindingLocked
创建`AppBindRecord`对象；
数据流程
```
servicerecord -> appbindrecord -> connectionrecord --save->ActiveServices
```

## 2.4 bringUpServiceLocked
作用：拉起`service` 和 `startService`一致；
- `bringUpServiceLocked` 中根据`app ?= null` , 调用`startProcessLocked`启动新进程；
- `realStartServiceLocked` 中根据传递的命令调用`CreateService BindService UnBindeService`
- `RunConnection` 分发命令，调用客户端`onServiceConnect & onServiceDisConnect`

## 2.5 绑定service解析
`handleBindService` 最终调用`service onBind`，获得服务端返回的`binder`；通过`pulishService`发布到`AMS`；

1. `AMS`中`service`相关的执行方法被委托给`ActiveServices`；`ActiveServices`根据`ServiceRecord`找到`ConnectionRecord`，然后调用对应`IServiceConnection`的`connected`方法；
2. `IServiceConnection.Stub.Proxy` 会调用`servicedispatcher`的`connected`方法，
3. 在`sd`的`connected`中，会往主线程`post RunConnection`对象，`RunConnection`是线程对象，`run`方法中会调用`sd`的`doConnected`方法，
4. 由于`sd`在创建保存了`ServiceConnection`对象，所以也就调用了`Client`的的`conn`对象，并传递了`service`端返回的`binder`；
5. `ConnectionRecord`创建于`ActivieServices`中，被添加到`mServiceConnections`中，`mServiceConnections`根据`connection.asBinder`，记录了不同`servicerecord`的连接；

## 2.6 PublishServie
调用`service onBind`之后，通过`publish`向AMS发布该`service`，目的是让`Client`通过`AMS`获取该`service Binder`

## 2.7 ActiveServices::publishServiceLocked
```
void publishServiceLocked(ServiceRecord r, Intent intent, IBinder service) {
    final long origId = Binder.clearCallingIdentity();
    try {
        if (r != null) {
            Intent.FilterComparison filter = new Intent.FilterComparison(intent);
            IntentBindRecord b = r.bindings.get(filter);
            if (b != null && !b.received) {
                b.binder = service;
                b.requested = true;
                b.received = true;
                for (int conni=r.connections.size()-1; conni>=0; conni--) {
                    ArrayList<ConnectionRecord> clist = r.connections.valueAt(conni);
                    for (int i=0; i<clist.size(); i++) {
                        ConnectionRecord c = clist.get(i);
                        if (!filter.equals(c.binding.intent.intent)) {
                            continue;
                        }
                        if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Publishing to: " + c);
                        try {
                            c.conn.connected(r.name, service); // 调用ServiceConnection connected
                        } catch (Exception e) {
                        }
                    }
                }
            }

            serviceDoneExecutingLocked(r, mDestroyingServices.contains(r), false); // remove ANR message
        }
    } finally {
        Binder.restoreCallingIdentity(origId);
    }
}
```
`AMS`遍历找到`ConnectionRecord`
