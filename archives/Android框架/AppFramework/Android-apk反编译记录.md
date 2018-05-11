## 1.1 解压apk
```
apktool.bat d -f xxx_ifly.apk -o ifly
```

## 1.2 错误
```java
S: Could not decode file, replacing by FALSE value: drawable-v4/notification_bg_low_normal.9.png
S: Could not decode file, replacing by FALSE value: drawable-ldrtl-xxxhdpi-v17/btn_tabstrip_new_tab_normal.png
S: Could not decode file, replacing by FALSE value: layout-v17/lightweight_fre_tos.xml
S: Could not decode file, replacing by FALSE value: drawable-ldrtl-xxxhdpi-v17/ic_omnibox_magnifier.pngb
```

## 1.3 解决 添加framework-res
```
F:\xx\AndroidTools\apktool
$ apktool.bat if framework-res.apk -p .
I: Framework installed to: .\1.apk

F:\xx\AndroidTools\apktool
$ java -jar apktool.jar d -f Chrome.apk -o Chrome
I: Using Apktool 2.2.2 on Chrome.apk
I: Loading resource table...
I: Decoding AndroidManifest.xml with resources...
I: Loading resource table from file: C:\Users\xxx\AppData\Local\apktool\framework\1.apk
I: Regular manifest package...
I: Decoding file-resources...
I: Decoding values */* XMLs...
I: Baksmaling classes.dex...
I: Copying assets and libs...
I: Copying unknown files...
I: Copying original files...
```
![添加framework-res.png](http://upload-images.jianshu.io/upload_images/74811-73dc13838668341d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 1.4 打包
```
apktool.bat b ifly
```

## 2 签名
```
java -jar signapk.jar testkey.x509.pem testkey.pk8 xxx.apk xxx_signed.apk
```

## 3.1 反编译odex
```
java -jar baksmali-2.2.1.jar deodex Wtwd.odex
```

## 3.2 结果
![odex2smali](http://upload-images.jianshu.io/upload_images/74811-c663d9b10b00eeb4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 3.3 错误： Error Cannot find dependency
```java
Error occurred while loading class path files. Aborting.
org.jf.dexlib2.analysis.ClassPathResolver$ResolveException: Error while loading oat file boot.oat
        at org.jf.dexlib2.analysis.ClassPathResolver.loadEntry(ClassPathResolver.java:250)
        at org.jf.dexlib2.analysis.ClassPathResolver.loadLocalClassPathEntry(ClassPathResolver.java:179)
        at org.jf.dexlib2.analysis.ClassPathResolver.loadLocalOrDeviceBootClassPathEntry(ClassPathResolver.java:191)
        at org.jf.dexlib2.analysis.ClassPathResolver.<init>(ClassPathResolver.java:120)
        at org.jf.dexlib2.analysis.ClassPathResolver.<init>(ClassPathResolver.java:105)
        at org.jf.baksmali.AnalysisArguments.loadClassPathForDexFile(AnalysisArguments.java:129)
        at org.jf.baksmali.AnalysisArguments.loadClassPathForDexFile(AnalysisArguments.java:86)
        at org.jf.baksmali.DisassembleCommand.getOptions(DisassembleCommand.java:207)
        at org.jf.baksmali.DeodexCommand.getOptions(DeodexCommand.java:71)
        at org.jf.baksmali.DisassembleCommand.run(DisassembleCommand.java:181)
        at org.jf.baksmali.Main.main(Main.java:102)
Caused by: org.jf.dexlib2.analysis.ClassPathResolver$NotFoundException: Cannot find dependency boot-okhttp.oat in null
        at org.jf.dexlib2.analysis.ClassPathResolver.loadOatDependencies(ClassPathResolver.java:270)
        at org.jf.dexlib2.analysis.ClassPathResolver.loadEntry(ClassPathResolver.java:248)
```
解决：将system/framework/arm/ 所有文件导入和baksmali-2.2.1.jar同级


## 3.4 smali.jar打包成dex
```java
 java -jar smali-2.2.1.jar ass out -o [target.dex] // 不指定targetdex，默认生成out.dex
```

## 3.5 dex反编译到jar, 使用jd-gui查看源码
![Jar2class](http://upload-images.jianshu.io/upload_images/74811-6445774684e56dd2.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 4.1 java到dex错误记录
```
E:\AndroidStudio\SDK\build-tools\25.0.2\lib
$ javac hello.java

E:\AndroidStudio\SDK\build-tools\25.0.2\lib
$ java -jar dx.jar --dex --output=hello.dex hello.class

PARSE ERROR:
unsupported class file version 52.0
...while parsing hello.class
1 error; aborting // jdk sdk版本不匹配
```
解决：
1. 使用Java1.7将Java转成class
2. sdk 25.0.2 dx.jar做dex优化
