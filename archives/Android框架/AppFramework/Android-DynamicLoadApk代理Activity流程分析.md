## 1.1 DynamicLoadApk Activity相关框架


![DynamicLoadApk Activity相关类图](http://upload-images.jianshu.io/upload_images/74811-ea2f83f74dd2be56.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


- `DLPluginManager`:插件管理模块，负责插件的加载、管理以及启动插件组件；
- `DLPlugin`:定义`Activity`的生命周期接口，是`DLPluginActivity`在`DLProxyImpl`的引用接口；
- `DLProxyImpl`:负责绑定`DLPluginActivity` `DLProxyActivity`
- `DLProxyActivity`:是`DLPluginActivity`运行的容器，需要在`AndroidManifest.xml`注册；

## 1.2  插件Activity加载流程
![插件Activity加载流程](http://upload-images.jianshu.io/upload_images/74811-5fcd9c9379af3732.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 1.3 加载插件PackageInfo Resource
1. `PackageInfo`由`PackageMS`的接口`getPackageArchiveInfo`解析得到；
2. 通过反射`AssetManager`的`addAssetPath`方法传入插件`activity`的路径得到插件的`AssetManager`，然后通过`AssetManager`创建插件的`Resources`对象
3. `DexClassLoader`加载`class`
4. `so`拷贝到宿主的`NativeLib`目录下

## 1.4 DLProxyActivity DLPluginActivity相互绑定
`DLProxyActivity`有`AMS`启动管理，`onCreate`阶段，相互绑定`DLProxyActivity`、`DLPluginActivity`；
`DLPluginActivity`需要`DLProxyActivity`所在的环境；
`DLProxyActivity`代理执行`DLPluginActivity`业务；
