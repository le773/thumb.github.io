设置`LOCAL_DEX_PREOPT`控制`App`是否`odex`优化，下面是实例以及日志
## 1.1 模块编译方法
```
mmm qcom/opensource/bluetooth/hidtestapp
```

## 1.2  Android.mk 
```
LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)

LOCAL_MODULE_TAGS := optional
src_dirs:= src/org/codeaurora/bluetooth/hidtestapp \ // 源码位置

LOCAL_SRC_FILES := \
        $(call all-java-files-under, $(src_dirs)) \

LOCAL_PACKAGE_NAME := hid //apk module名字
LOCAL_CERTIFICATE := platform // 采用platform签名

LOCAL_DEX_PREOPT := false // 不做odex优化

LOCAL_PROGUARD_ENABLED := disabled

LOCAL_MULTILIB:= 32 // 只编译32位应用

include $(BUILD_PACKAGE)
```

## 2.1 日志
```
PRODUCT_COPY_FILES device/qcom/common/media/media_profiles.xml:system/etc/media_profiles.xml ignored.
PRODUCT_COPY_FILES device/qcom/common/media/media_codecs.xml:system/etc/media_codecs.xml ignored.
No private recovery resources for TARGET_DEVICE project
616+0 records in
616+0 records out
630784 bytes (631 kB) copied, 0.00118089 s, 534 MB/s
Starting build with ninja
ninja: Entering directory `.'
[ 33% 1/3] Ensure Jack server is installed and started
Jack server already installed in "/home/xxx/.jack-server"
Server is already running
[ 66% 2/3] target Package: hid (out/target/product/project/obj/APPS/hid_intermediates/package.apk)
warning: ignoring flag -c hdpi-v4. Use --preferred-density instead.
warning: ignoring flag -c mdpi-v4. Use --preferred-density instead.
warning: ignoring flag -c hdpi-v4. Use --preferred-density instead.
warning: ignoring flag -c mdpi-v4. Use --preferred-density instead.
Warning: AndroidManifest.xml already defines minSdkVersion (in http://schemas.android.com/apk/res/android); using existing value in manifest.
Warning: AndroidManifest.xml already defines targetSdkVersion (in http://schemas.android.com/apk/res/android); using existing value in manifest.
[100% 3/3] Install: out/target/product/project/system/app/hid/hid.apk
make: Leaving directory `/data2/.../android'
```
## 2.2 查询结果
```
xxx@(none):/data2/../android/vendor$ ls ../out/target/product/xx/system/app/hid/ -l
total 16
-rw-rw-r-- 1 xxx xxx 13502 Oct 19 20:16 hid.apk
```

## 3.1 Odex优化
开启odex
```
LOCAL_DEX_PREOPT := true
```
`nostripping` 不删除`apk`中的`classes.dex`

## 3.2 日志
```
PRODUCT_COPY_FILES device/qcom/common/media/media_profiles.xml:system/etc/media_profiles.xml ignored.
PRODUCT_COPY_FILES device/qcom/common/media/media_codecs.xml:system/etc/media_codecs.xml ignored.
No private recovery resources for TARGET_DEVICE project
616+0 records in
616+0 records out
630784 bytes (631 kB) copied, 0.00224366 s, 281 MB/s
Starting build with ninja
ninja: Entering directory `.'
[ 33% 2/6] Ensure Jack server is installed and started
Jack server already installed in "/home/xxx/.jack-server"
Server is already running
[ 50% 3/6] target Package: hid (out/target/product/project/obj/APPS/hid_intermediates/package.apk)
warning: ignoring flag -c hdpi-v4. Use --preferred-density instead.
warning: ignoring flag -c mdpi-v4. Use --preferred-density instead.
warning: ignoring flag -c hdpi-v4. Use --preferred-density instead.
warning: ignoring flag -c mdpi-v4. Use --preferred-density instead.
Warning: AndroidManifest.xml already defines minSdkVersion (in http://schemas.android.com/apk/res/android); using existing value in manifest.
Warning: AndroidManifest.xml already defines targetSdkVersion (in http://schemas.android.com/apk/res/android); using existing value in manifest.
[100% 6/6] Copy: out/target/product/project/system/app/hid/oat/arm/hid.odex
```
## 3.3 查询结果
```
xx@(none):/data2/../android/vendor$ ls ../out/target/product/xxx/system/app/hid/oat/arm/ -l
total 44
-rw-r--r-- 1 xxx xxx 41440 Oct 19 20:22 hid.odex
```
