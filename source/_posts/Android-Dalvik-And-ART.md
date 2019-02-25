---
title: Android-Dalvik And ART
date: 2019-02-25 12:01:56
tags:
	- Android
	- Dalvik
	- ART

category: Android
comments: ture
---

熟悉 `Android`的都知道， `Android` 应用程序的开发 通常采用 `Java`语言进行开发，不同于 `Java` 的是，`Java` 环境编译后的生成的是 `.class`文件，而 Android 的 可执行文件是 `.dex`文件(Dalvik Executable)，它是 `.class`文件经过 工具`dx`处理后的文件格式，可被 `Dalvik`虚拟机直接运行(目前 都转为使用更先进的`d8`编译器来处理 `.class`文件来生成 `.dex`文件)。执行这些 `dex`文件的 运行环境就是 `Dalvik`，后来Google 推出了更先进的 `ART`。  

`ART`和`Dalvik`是服务于Android应用和Android系统服务的.`Dalvik `在Android项目一开始的时候就被引入，而`ART`在`Android4.4`的时候被引入，并对开发人员可见，在`Android 5.0`的时候，`ART`虚拟机这个是成为默认的虚拟机。 `ART`的推出就是为了替代`Dalvik`虚拟机来获得更好的性能的。`ART`和 `Dalvik`都支持运行 `dex`字节码，因此为 `Dalvik`开发的应用程序，也可以直接被 `ART`虚拟机运行(当然有一些Dalvik的技术特性不被支持了，比如 显示的调用`System.gc()`，具体可参考[Verifying App Behavior on the Android Runtime (ART)](https://developer.android.com/guide/practices/verifying-apps-art.html))。  

### ART 的新特性

#### AOT编译 -- Ahead-of-time (AOT) compilation

ART引入了提前（AOT）编译，可以提高应用程序性能。 ART还具有比Dalvik更严格的安装时间验证。  
在安装App时，ART使用设备上的`dex2oat`工具编译应用程序。此实用程序接受`DEX`文件作为输入，并为目标设备生成已编译的应用程序可执行文件`.oat`,由于`Dalvik`虚拟机使用了这么多年，整个系统中可能很多地方硬编码使用了`.odex`文件，为了无缝的替换`Dalvik`虚拟机，通过`dex2oat`生成的 `.oat`文件在有些地方可能会命名为`.odex`。  
例如/data/app/{pkg}/oat/{cpuType}下,在我的手机上是`/data/app/{pkg}-xxx/oat/arm64/`，目录下的文件是以 `.odex`结尾。  
虽然名字是`.odex`,但是不同于通过`dex2opt`生成的`.odex`文件。`dex2oat`输出的`.odex`文件其实就是 `.oat`文件，是 elf文件格式，里面包含着本地机器码。 
<mark>由于是针对 当前设备的，所以生成的可执行文件不能被别的机器使用。</mark>  

随便说一下，`/data/app/{pkg}/oat/{cpuType}` 目录下 可能还包含`.art`和 `.vdex`文件。 `.vdex` 这是`Android8`引入的，它存储是 app 安装时执行 `dex2oat`经过验证后的dex文件，以后可以使用`.vdex`文件，避免再次验证Dex文件的合法性。  
`.art`文件 主要存储apk 启动的热点函数相关地址的记录，方便寻址相关；该文件不会在程序安装时就生成，而是运行了一段时间后，收集到了足够过的JIT热点方法信息，会再次进行`dex2oat`，将热点方法编译成`native code`,这就是`.art`文件的作用。


#### 改进GC机制

参考之前的[JVM-垃圾收集器](/JVM-垃圾收集器/)一文，JVM 的垃圾收集器往 并发收集方向前进，同样，为了尽量减少GC对 用户进程的卡顿影响，改进的GC机制也采用了并行收集和并发处理。  

`ART`虚拟机 默认使用的垃圾收集器是 `CMS`收集器，我们知道 `CMS`收集器采用的是`标记-清除`算法，没有`内存-整理`，所以 当 Android 应用程序进入后台后 缓存状态时，虚拟机会执行 `堆内存压缩整理`，减少内存碎片的产生。

除了垃圾收集器的升级换代，针对内存的分配方面，`ART`也引入了全新的 基于 位图的内存分配器，称为RosAlloc（插槽分配器)。这个新的分配器具有分片锁定，针对小内存分配，增加了 `Thread Local`的 buffers，提高 小内存的分配性能。

#### 开发和调试改进

ART提供了许多功能来改进应用程序开发和调试。比如:  

- 支持 `采样分析`
- 增加更多的 虚拟机调试功能，比如可以通过`ART`知道 一个给定的 Class 有多少个实例，已经对象的引用状态
- 改进了异常和崩溃报告中的诊断细节

#### ART的JIT编译器

我们知道 在 `Android 2.2` 的时候，为了提升 `Dalvik`的性能，引入了`JIT（Just-In-Time ）`技术。这是一种在 运行时 编译 频繁使用的 Class，成为 `native code` 机器码，提升 效率，免得在程序运行期间 每次都重新翻译成 机器码的尴尬。但是这个性能提升也不是绝对的，比如 Class 中的大部分代码执行较少，那么JIT编译花费的时间不一定少于执行dex的时间。所以 `JIT`不对所有dex代码进行编译，而是只编译执行次数较多的dex为本地机器码。而且，`JIT`将 dex 字节码编译成native code 是发生在程序运行期间的，而且生成的 `native code` 也不是一劳永逸的，下次运行，还是需要重新执行`JIT` 过程。

使用过早期Nexus5 搭配 Android5.0的人可能知道，安装一个App，简直是龟速，其实就是安装的时候如果全量编译成机器码很耗时，为了照顾安装效率，就取消了全量编译，这时候再引入 `JIT` 就很有意义了，针对比较热门的代码，使用`JIT`编译成 native code，
所以在 `ART` 中实现了 `JIT`编译器是很有必要的，在Android7.0中重新引入了`ART`下的`JIT`技术。  

以下是 `JIT`在 `ART`系统中的结构示意图

![](/img/art/art-jit-arch.png)

可以看出，如果 `ART` 虚拟机 面对 非 `oat`可执行文件时，还是有必要的。

以下是 `JIT`的工作流程示意图:  

![](/img/art/jit-workflow.png)

`JIT`是运行时编译，是动态编译，`AOT`可以被认为是静态编译，二者配合和互补，在效率做平衡。

当然也可以不适用 `JIT`，可以通过以下命令禁止: 
 
```shell
adb root
adb shell stop
adb shell setprop dalvik.vm.usejit false
adb shell start
```

[参考](https://source.android.com/devices/tech/dalvik)










