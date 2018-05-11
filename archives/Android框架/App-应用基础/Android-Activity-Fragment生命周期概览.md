
![activity_fragment生命周期.jpg](http://upload-images.jianshu.io/upload_images/74811-2bbad4bc6928ec90.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


`Activity`从`onResume->onPause` 以及 `onPause->onStop`均会检查`Activity.state`判断是否调用`onRestoreInstanceState`；
