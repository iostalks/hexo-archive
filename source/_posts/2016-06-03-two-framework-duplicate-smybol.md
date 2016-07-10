---
title: 两个 framework 存在 duplicate symbol 的解决方案
date: 2016-06-03 17:37:47
tags:
---

## 铺垫
前两天老大让我把 SDK 中的家电协议库和网路库分开为两个 framework，本以为只需要将文件拆分到两个工程并编译适配 i386 x86_64 armv7 arm64 四种构架就 OK 了。然而当分别拖到项目中用的时候出现了 `duplicate symbol` 的错误，才想起来两个工程中都包含了相同的设备基类，如果任何一方删除都无法正常的使用。

由于是自己的源码所以第一反应就是将其中一个库重复的类改名字，然而总觉得这样不够「酷」。要是别人第三方的库出现这种情况该怎么办呢？经过老大的思路指导和 Google 了一圈之后，找到了解决办法。

<!--more-->

查阅过程中发现有很多讲解类似解决办法的，但是对于没有一点 iOS CPU 构架知识的同学来说可能看起来还是挺费劲的。所以这里记录了一下自己成功解决该问题的整个过程。

解决方法是：**选择一个 framework 将其中的的 .a 文件(fat)，解成只有一种处理器结构的 .a 文件(no fat)，再将这个 .a 文件拆开为 .o 文件，删除和另一个 framework 重复的 .o 文件。然后将这些 .o 文件重新合成 .a 文件。其他处理器结构下也是用同样的方法，最后将这些不同构架的下剔除重复 .o 文件的 .a 文件合成一个 fat 的 .a 文件。**

> 注：framework 里面的库文件可能不是 .a 后缀的，我生成的就是不含后缀的，但是操作方式是相同的。

上面说的不同处理器构架看下表：

**真机：**

| ARM 芯片 | iPhone 机型 | CPU 位数 | 
| :------------: | :-------------: |:---:|
| armv6  | iPhone， iPhone2， iPhone 3G  | 32位 |
| armv7  | iPhone 3GS, iPhone 4, iPhone 4S | 32位 |
| armv7s | iPhone5, iPhone5C  | 32位 |
| arm64  | iPhone 5s, iPhone6/6s, iPhone6/6s Plus  | 64位 |

**模拟器：**

| ARM 芯片 | CPU 位数 | 
| :------------: | :-------------: |
| xi386   | 32位 | 
| x86_64  | 64位 | 

## 开始
假设我们现在要拆解的 .a 文件名为 TestSDK.a

首先打开终端 `cd` 到要分节的 framework 目录，执行以下命令查看支持的处理器构架：
```BASH
$ lipo -info TestSDK.a
```

或者亦可以用更详细的命令：
```BASH
$ xcrun -sdk iphoneos lipo -info TestSDK.a
```

此时将会输出 framework 支持的几种构架：
```BASH
Architectures in the fat file: TestSDK.a are: armv7 i386 x86_64 arm64 
```

我们先拿 armv7 下手分离只包含 armv7 的 .a 文件， 命令如下：

```BASH
$ lipo -extract_family armv7 -output TestSDK-armv7.a TestSDK.a
```

`-output` 后面的 `TestSDK-armv7.a` 是我们为新的 .a 文件取的名字。

执行完后我们可以看到在同一目录下生成了 TestSDK-armv7.a 文件。

现在使用`lipo -info` 命令查看其处理器构架是这样：
```BASH
Architectures in the fat file: TestSDK.a are: armv7 
```

然而如果你的 framework 支持 armv7 和armv7s 那可能输出的会是这样： 
```BASH
Architectures in the fat file: TestSDK.a are: armv7, armv7s
```

由此可见该文件还是fat文件，包含两种处理器的，我们还要将其分解
```BASH
$ lipo TestSDK-armv7.a -thin armv7s -output TestSDK-armv7s.a
```

现在我们拿到了只有一种构架的 .a 文件，在将其拆分为 .o 文件前一定要在该目录下新建一个文件就叫 armv7, 并将 TestSDK-armv7.a 文件拖进去(不是复制)。再 cd 到 armv7 文件夹，执行拆解命令：
```BASH
$ ar -x TestSDK-armv7.a
```

找出不需要的 .o 文件删除，再将其合成为新的 .a 文件，命令如下：
```BASH
$ libtool -static -o ../TestSDK-armv7.a *.o
```

注意该命名执行完后，与 armv7 文件夹同一级目录下就会生成一个 TestSDK-armv7.a 文件，该文件就是剔除重复 .o 文件的之后的 .a 文件啦。

其他构架的剔除重复文件的方式同上，分别像这样：
```BASH
$ lipo -extract_family armv7 -output TestSDK-armv7.a TestSDK.a
$ lipo -extract_family i386 -output TestSDK-i386.a TestSDK.a
$ lipo -extract_family x86_64 -output TestSDK-x86_64.a TestSDK.a
$ lipo -extract_family arm64 -output TestSDK-arm64.a TestSDK.a
```

然而这里在 arm64 下会出问题，`lipo -info TestSDK-arm64.a`的时候会出现如下错误：
```BASH
can't figure out the architecture type of: TestSDK-arm64.a
```

arm64 下需要使用另外一种命令来生成：
```BASH
$ lipo TestSDK.a -thin arm64 -output TestSDK-arm64.a
```

这样就 OK 了。

最后我们会得到四个不同构架下的 .a 文件分别为：TestSDK.a-i386，TestSDK.a-x86_64.a，TestSDK.a-armv7，TestSDK.arm64

我们需要将这四个文件合回 fat，命令为：

```BASH
$ lipo -create -output TestSDK.a TestSDK.a-i386 TestSDK.a-x86_64.a TestSDK.a-armv7 TestSDK.arm64
```

现在得到的 TestSDK.a 就是最终剔除 duplicate smybol 的文件了


## 参考：

[Apple移动设备处理器指令集 armv6、armv7、armv7s及arm64](http://www.cocoachina.com/ios/20140915/9620.html)
[iOS程序开发引用的第三方库之间出现duplicate symbol时的处理方法](http://blog.k-res.net/archives/1024.html)