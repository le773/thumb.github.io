##  1.1 selinux 语法结构

![Selinux语法结构](http://upload-images.jianshu.io/upload_images/74811-a19afd36a397b169.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
含义：允许`mediaserver`进程对`app_data_file`类型的文件（`file`）,进行`rw_file_perms`操作

###### 1.1.1 attribute 和 type的关系
1. `type atrribute` 位于同一个命名空间，（不能用`type attribute` 定义相同名字的东西）
2. `attribute`类似于`typegroup`，即`type A (attribute)B`，`Type A`属于`attribute B`；

## 2.1 selinux 源码快速验证
```
make sepolicy -j[线程数目]
```
###### 2.1.1 生成中间文件目录
```
out/target/product/xxx/obj/ETC/sepolicy_intermediates
```

## 3.1 SELinux 状态
•`enforcing`：强制模式，代表 `SELinux` 运作中，且已经正确的开始限制 `domain/type` 了；
•`permissive`：宽容模式：代表 `SELinux` 运作中，不过仅会有警告讯息并不会实际限制 `domain/type` 的存取。
•`disabled`：关闭，`SELinux` 并没有实际运作。

###### 3.1.1 查看状态
```
getenforce
```
###### 3.1.2 临时关闭
```
setenforce 0
```
