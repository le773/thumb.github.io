
## 1.1 Android volley通过网络请求图片流程

![VolleyRequestImage](http://upload-images.jianshu.io/upload_images/74811-32451f74c03188d2.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###### 1.2.1 ImageLoader的get方法
1. 首先`ImageCache`缓存中查找
2. 缓存不存在则
  a. 创建`ImageContainer`，设置回调`imageview`的`ImageListener`
  b. 通知`ImageView`设置默认的`Bitmap`
3. 创建`ImageRequest`
4. 以`cachekey`保存在`mInFlightRequests`中，`value`为`BatchedImageRequest`

###### 1.2.2 RequestQueue的add方法
添加`request`到`requestqueue`，如果`request`不可缓存，则添加到`mNetworkQueue`由`NetworkDispatcher`线程执行网络调度；
否则添加到由添加`mCacheQueue`由`CacheDispatcher`调度，如果缓存为空或者过期则重新发送到`mNetworkQueue`；

###### 1.2.3 Other
`BasicNetWork`交给`HttpStack`负责网络请求相关；
`ExecutorDelivery`通过引用线程的`Handler`根据`response`的结果分发给对应的`request`;
`ImageLoader`创建`ImageRequest`时，为其设置了`Response.Listener`监听实现；
`ImageRequest`将结果发送给所有其`mBatchedResponses`持有的所有`BatchedImageRequest`;
