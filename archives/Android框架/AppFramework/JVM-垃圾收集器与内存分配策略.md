## 1. JVM内存模型
![JVM内存模型](http://upload-images.jianshu.io/upload_images/74811-415aea436bc96d41.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 方法区和堆属于共享数据区；
- 程序计数器、虚拟机栈、本地方法栈属于线程私有区；
- 方法区包含运行时常量池；

## 2.1 对象已死亡吗？
在堆里存放着Java世界几乎所有的对象实例，垃圾收集器在对堆进行回收前，第一件事就是判断对象是否存活；

### 2.2.1 引用计数法
原理：
给对象中添加一个引用计数器，每当有一个对方引用它，计数器值加1；当引用失效时，计数器值就减一；任何时刻计数器为0的对象就是不可能再被使用的；

![引用计数法](http://upload-images.jianshu.io/upload_images/74811-7af502cc67d92624.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
缺陷：无法处理循环引用

![引用记数法循环引用](http://upload-images.jianshu.io/upload_images/74811-a132b6e89cd287e5.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 2.2.2 可达性计数法
原理：
通过一系列称为"GC Roots"的对象作为起始点，从这些节点开始向下搜索，搜索所走过的路径称为引用链(Reference Chain)，当一个对象到GC Roots没有任何引用链相连，则证明此对象不可用；
GC Roots包括：
- 虚拟机栈(栈桢中的本地变量表)中的引用对象；
- 方法区中类静态属性引用的对象；
- 方法区中常量引用的对象；
- 本地方法栈中JNI引用的对象；

![可达性计数法](http://upload-images.jianshu.io/upload_images/74811-7c020e04856f0516.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 2.3 再谈引用

###### 强引用
类似 Object obj = new Object，只要强引用还存在，垃圾收集器永不回收被引用对象；

###### 软引用
用来描述一些还有用但非必需的对象。对于软引用关联着的对象，在系统将要发生内存溢出之前，将会把这些对象列进回收范围之中进行第二次回收。如果这次回收还没有足够的内存，才会抛出内存溢出异常；

###### 弱引用
描述非必需对象，但是它的强度比软引用更弱一些，被弱引用关联的对象只能生存到**下一次垃圾收集**发生之前。当垃圾收集器工作时，无论当前内存是否足够，都会回收掉只被弱引用关联的对象。

###### 虚引用
对象是否拥有虚引用存在，完全不对其生存时间构成影响，也无法通过虚引用来取得一个对象实例；
为对象设置虚引用关联的唯一目的：能在这个对象被垃圾收集器回收时收到**系统通知**；

###### 引用强度
强引用 -> 软引用 -> 弱引用 -> 虚引用

## 3.1 垃圾收集算法

### 3.1.1  标记-清除算法
![标记-清除](http://upload-images.jianshu.io/upload_images/74811-439c99f79581a55c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###### 缺陷
- 效率：标记-清除过程效率不高；
- 空间：标记-清除 之后产生大量不连续的内存碎片，导致以后程序运行过程中需要分配较大对象时，无法找到足够连续内存而不得不提前触发另一次垃圾收集动作；

### 3.1.2 复制算法
![复制算法](http://upload-images.jianshu.io/upload_images/74811-0b8f5cf50a1f89d0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
将内存按照容量分为大小相等的两块，每次只使用其中一块；内存清理时将还存活的对象移动到另一块空间，然后一次清理此块内存；

###### 缺陷
- 内存缩小一半；

### 3.1.3 标记-整理算法
![标记-整理算法](http://upload-images.jianshu.io/upload_images/74811-60cac722fd8d02a4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

标记过程和标记-清除算法一样，然后让所有存活的对象移向另一端，最后清理掉端边界以外的内存；

### 3.1.4 分代搜集算法
根据对象存活周期将堆划分为新生代、老年代，根据各个年代的算法采用最适当的搜集算法：
- 在新生代中，每次垃圾收集都发现有大批对象死去，只有少量对象存活，那么对存活对象采用复制算法；
- 在老年代中，因为对象存活率较高，没有额外空间对它进行分配担保，则使用标记-清理或标记整理；

## 4.1 内存分配与回收策略
1.  对象优先在Eden分配；
2.  大对象直接进入老年代；
3.  长期存活的对象将进入老年代；
4.  动态对象年龄判定;
  并不要求年龄的对象必须达到MaxTenuringThreshold才晋升老年代，如果在Survivor空间中相同年龄所有对象大小总和大于Survivor空间的一半，年龄大于或等于该年龄的对象就可以进入老年代；
5. 空间分配担保
```
if 老年代的最大连续分配空间 > 新生代所有对象总空间 
then mingc确保安全
if 老年代的最大连续分配空间 > 历次晋升到老年代对象的平均大小;
    Minor GC
else
    fullgc
// fullgc发生在老年代，一般伴随一次mingc 
```

## 5.1 GC流程
1 首先，任何对象分配在Eden分区， survivor中的from to 首次分配时为空；
![Object Allocation](http://upload-images.jianshu.io/upload_images/74811-5cc01393e8d2b47d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2 当Eden分区满时，MinGC将会被触发
![Filling the Eden Space](http://upload-images.jianshu.io/upload_images/74811-cb0bdb4c4b89c9f2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

3 MinGC后引用的对象被移动到survivor中的s0(From survivor space)分区，并且对象生命周期加1，Eden中不存在引用的对象随着Eden分区清理被释放；
![Copying Referenced Objects](http://upload-images.jianshu.io/upload_images/74811-5f2f7b056ddb37c6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

4 下次MinGC时，Eden、From survivor space中存在引用的对象被移动到To survivor space, 并且额生命周期加1, 然后清理Eden、From survivor space 未被引用的对象；

![Object Aging](http://upload-images.jianshu.io/upload_images/74811-edb7dd9c03c0d68a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

5 此时MinGC时，Eden、To survivor space分区中的对象被移动到From survivor space，并且生命周期加1，Eden、To survivor space中未被引用的对象将被释放;

![Additional Aging](http://upload-images.jianshu.io/upload_images/74811-a45cfb6ff4ba26e2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

6 经过数次MinGC后，某些存活对象的生命达到了可以移动到年老代的阈值，将被移动到年老代；
![Promotion](http://upload-images.jianshu.io/upload_images/74811-0f2b25b7a4b81094.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

7 随着MinGC继续，符合条件的对象逐步移动到年老代
![PromotionMulti](http://upload-images.jianshu.io/upload_images/74811-0222e1be6fdb043b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

8 随着年老代对象增加多将触发FullGC
![GC Process Summary](http://upload-images.jianshu.io/upload_images/74811-e3f3b36f00860951.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###### 参考
[垃圾回收机制中，引用计数法是如何维护所有对象引用的？](https://www.zhihu.com/question/21539353/answer/95667088 "垃圾回收机制中，引用计数法是如何维护所有对象引用的？")
[JVM内存模型](http://gityuan.com/2016/01/09/java-memory/ "JVM内存模型")
[Java Garbage Collection Basics](http://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/index.html "Java Garbage Collection Basics")
