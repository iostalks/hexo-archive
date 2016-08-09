---
title: 优化 App 的启动速度
date: 2016-07-10 15:23:54
tags:
---

App 的启动速度不仅影响我们调试，也直接关系到用户体验。之前有些很久没有打开过的项目，需要花费很长的时间才完成编译；对应的 App 在点击后，许久才出现启动画面。你是否为这些问题苦恼过呢？

这是我观看 WWDC2016 Sessions406 《Optimizing App Start Time》的笔记。

<!--more-->

内容主要包括理论和实践两部分：

理论：
1. Mach-O 文件的格式
2. 虚拟内存是什么
3. Mach-O 文件是如何被加载的

实践：
1. 如何测量启动时间
2. 如何优化启动时间

## 理论
#### Mach-O 是什么

Mach-O 文件类型:

1. Executable - 可执行文件，应用程序的主要二进制文件
2. Dylib - 动态库（比如 DSO 或者 DLL）
3. Bundle - 属于动态库，不能被链接，只能使用 dlopen() 方法打开；

举例：

Image - 可以是一个可执行的文件，动态库或者 Bunlde
Framework - 包含资源和头（header）的动态库


如何将我们的代码和 Mach-O 文件建立联系呢？我们先来做个实践，创建一个 helloworld.c 文件，用命令行执行：

```bash
$ xcrun clang helloworld.c
```

会生成一个名为 a.out 的可执行文件，接着我们可以使用 size 工具来分析这个文件：

```
$ xcrun size -x -l -m a.out 
Segment __PAGEZERO: 0x100000000 (vmaddr 0x0 fileoff 0)
Segment __TEXT: 0x1000 (vmaddr 0x100000000 fileoff 0)
	Section __text: 0x34 (addr 0x100000f50 offset 3920)
	Section __stubs: 0x6 (addr 0x100000f84 offset 3972)
	Section __stub_helper: 0x1a (addr 0x100000f8c offset 3980)
	Section __cstring: 0xe (addr 0x100000fa6 offset 4006)
	Section __unwind_info: 0x48 (addr 0x100000fb4 offset 4020)
	total 0xaa
Segment __DATA: 0x1000 (vmaddr 0x100001000 fileoff 4096)
	Section __nl_symbol_ptr: 0x10 (addr 0x100001000 offset 4096)
	Section __la_symbol_ptr: 0x8 (addr 0x100001010 offset 4112)
	total 0x18
Segment __LINKEDIT: 0x1000 (vmaddr 0x100002000 fileoff 8192)
total 0x100003000
```

该文件由4个 Segment（下划线加大写字母表示）组成，有些 segment 中又包含多个 Section（下划线加小写字母表示）。

我们还可以使用 `file` 命令来查看具体信息

```
$ file a.out
a.out: Mach-O 64-bit executable x86_64
```

可以看出 a.out 即为所谓的 Mach-O 文件，我们对其内存结构进行分析。其结构示意图如下：

![image](http://7xsto7.com1.z0.glb.clouddn.com/mach-o-segment.jpeg)

Mach-O 文件结构:

- __TEXT 头，包含被代码，只读，允许被进程执行，但不可修改。
- __DATA 包含所有可以被读写的内容，比如：变量等.
- __LINKEDIT 加载程序的「元数据」

在64位下 `__PAGEZERO` 大小为 4 GB。这 4 GB 不是文件的真实大小，但是规定了进程地址空间的前 4GB 被映射为不可执行，不可写和不可读。这个范围用于存放 NULL 指针以及异常错误信息，访问该内存时会得到 `EXC_BAD_ACCESS` 错误。

我们在构建自己 Framework 的需要适配多种构架，比如32位下的 armv7, 64位下的arm64，还有模拟器i386，x86_64，支持多种 CPU 构架的 Framework 被称为Mach-O Fat Files. 其结构如下：

![image](http://7xsto7.com1.z0.glb.clouddn.com/mach-o_universal.jpeg)

Fat Header 作为文件的头部，包含如下信息：

- 内存 Page 的大小
- 列出 Fat 文件支持的处理器构架以及对应的偏移

#### 虚拟内存

虚拟内存的定义：将被分隔成多个物理内存碎片，以及部分暂时存储在外部磁盘存储器上的内存空间进行映射，使应用程序认为这是一块连续的可用内存空间。

当加载一个可执行文件时，虚拟内存系统将 segment 映射到进程的地址空间上。可以简单的理解为将可执行文件加载到内存中，并且会按需加载。


#### Mach-O 文件是如何被加载的
在 mian() 函数执行之前，操作系统为我们完成一些列的操作？其过程类似于

exec() -> main()

exec()执行过程中操作系统会随机分配一段可用的内存给应用程序。但有个规则，不分配低地址的内存空间，该地址空间大小由处理器的位数决定，情况如下：

- 32位处理器保留 4K+
- 64位处理器保留 4G+

在理论部分 Mach-O 文件的结构图可看出。低地址保存用于空指针、异常错误信息等。

![image](http://7xsto7.com1.z0.glb.clouddn.com/map_address_1.jpeg)

内存分配完成后，处理器运行动态加载器（Dynamic loader）来加载动态库（dylibs）和资源文件。
动态加载过程：

1. 映射所有的直接依赖库，递归调用间接依赖库(Map all dependent dylibs,recurse).
2. 重建所有的图片(Rebase all images).
3. 绑定所有图片(Bind all images).
4. 准备代码层面的加载(ObjC prepare images)
5. 运行初始化构造器(Run initializes).

![image](http://7xsto7.com1.z0.glb.clouddn.com/process.jpeg)

以下是对各个过程具体的介绍：

######1. 加载动态库
直接加载：

- 解析所有的动态库
- 找出需要的 mach-O 文件
- 打开并读取文件
- 认证 mach-O 文件
- 登记代码签名
- 为每个 segment 调用 `mmap()` 映射函数

递归加载：
当所有直接依赖库加载完毕后，还存在直接依赖库依赖于其他库的情况。递归加载间接依赖库，一般 App 需要加载 100 到 400 个动态库！其中绝大多数为系统库，苹果已经最优化了系统库的加载。

######2. 重建（Rebasing）

将图片资源根据其地址进行加载，重建信息被编码在 `LINKEDIT` segment 中。重建的过程按照地址顺序执行，所以可以被内核预取。

######3. 绑定（Binding）
应用程序对动态库的引用只在符号层(symbolc)面，绑定过程中需要加载器通过函数名来查找，相对于重建这个过程需要更多的计算步骤。

![image](http://7xsto7.com1.z0.glb.clouddn.com/binding.png)

######4. ObjC 准备
- 完成重建和绑定后的配置工作
- 登记定义的 ObjC 类
- 更新实例变量对应的内存位置
- 分类的方法被插入到主类

######5. 初始化构造器（Initializer）
- 静态分配内存的对象的初始化
- 调用 +load 方法
- 调用相关联的动态库
- 最后，执行 main（）函数

## 实践
对于不同的设备启动速度都不相同，苹果认为完美的启动时间是在 400 毫秒以内。App 的最大允许启动时间为20秒，如果超过这个时间就会被 killed。

先介绍两个不太常听的概念：
- 热启动(Warm launch)：应用程序已驻后台的启动
- 冷启动(Cold launch)：完全没有运行的程序的启动

热启动速度明显比冷启动快很多。

我们可以通过编辑工程的 Schemes 来打印 App 的启动中各项时间花费，从而来分析和优化启动过程。
在 Arguments -> Environment Variables 中加入 `DYLY_PRINT_STATISTICS`。

![image](http://7xsto7.com1.z0.glb.clouddn.com/edit_scheme.png)

设置后，控制台会输出各项时间：

![image](http://7xsto7.com1.z0.glb.clouddn.com/conlose_print.png)

下面是几种比较耗时的的优化方式：

#####1. 加载动态库

加载动态库需要很大的开销，解决方法是尽可能减少引入不必要的动态库，合并现有可操作的动态库，使用静态文件，使用懒加载等

#####2. 重建和绑定过程

- 减少指针变量的使用
- 减少 Objective-C 类
- 减少 C++ 虚基类的使用(没用过。。)
- 建议使用 Swift 结构体
- 尽可能将属性设置为只读

####3. 初始化
- 将 `+load` 函数的操作尽可能在 `+initialize` 方法中执行
- 简化 C++ 构造函数
- 不要在初始化方法里面调用 `dlopen()`
- 不要在初始化方法里面创建线程

`+initialized` 会在任何类加载之前被调用，而 `+load` 是所在类被加载到系统的时候被调用，这通常比 `+initialized` 调用的时机要早。在 `+load` 函数里面做操作会增加系统的启动时间。

`dlopen()` 函数用于打开 Bundle 尽可能的延迟读取本地的 Bundld 资源，也有利于加快启动速度。

线程的创建需要很大的开销，不要在 App 启动的过程创建子线程。

## 参考阅读
[https://developer.apple.com/videos/play/wwdc2016/406/](https://developer.apple.com/videos/play/wwdc2016/406/)
[http://objccn.io/issue-6-3/](http://objccn.io/issue-6-3/)
[https://code.facebook.com/posts/1675399786008080/optimizing-facebook-for-ios-start-time/](https://code.facebook.com/posts/1675399786008080/optimizing-facebook-for-ios-start-time/)
