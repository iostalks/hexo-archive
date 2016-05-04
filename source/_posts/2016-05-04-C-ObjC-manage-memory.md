---
title: C++/ObjC 内存管理
date: 2016-05-04 18:34:18
tags:
---

## 前言
内存管理，可以说是进阶一门编程语言不得不谈的话题。当掌握了一门语言的内存管理，就像是拥有了武学里的内功，再去学习其他语言只不过是招式上的变化了。

之前在微信公众号 iOSTalk 上推送过一篇《内存管理之引用计数》。本文在原来的基础上做了进一步的细化，将内存管理的知识做一个系统的整理。

<!-- more -->

## 内存的分配方式
ObjC/C++ 虽然各自带有自己的色彩，但是它们同出师门，仍旧带有浓烈的 C 的味道。
在C/C++ 编译的程序占用的内存分为五个部分：

**栈区（stack）**

函数调用时使用栈来保存函数现场，自动变量（即生命周期限制在某个 scope 的变量）也存放在栈中。

栈是用于存放本地变量，内部临时变量以及有关上下文的内存区域。程序在调用函数时，操作系统会自动通过压栈和弹栈完成保存函数现场等操作，不需要程序员手动干预。

栈是一块连续的内存区域，栈顶的地址和栈的最大容量是系统预先定好的。能从栈获得的空间较小，如果栈申请的空间超过了栈的剩余空间时，会造成栈溢出 Crash。

栈是机器系统提供的数据结构，计算机会在底层对栈提供支持：分配专门的寄存器存放栈的地址，压栈和出栈都有专门的指令执行，这就决定了栈的执行效率比较高。

当函数被调用时，以块（block）的形式被压入栈帧中，调用结束之后又会被自动的弹出栈帧，其中的局部变量被回收。其实现基理是动态的移动栈顶的指针，来控制栈中数据的进出。

**堆区（heap）**

使用 `malloc`, `realloc`, 和 `free` 函数控制的变量，堆在所有的线程，共享库，和动态加载的模块中被共享使用，

当使用 `malloc` 和 `free` 时就是操作堆中的内存。对于堆来说，释放工作由程序员控制，容易产生 Memory Leak。

堆是向高地址扩展的数据结构，是不连续的内存区域。这是由于系统是链表来存储的空闲内存地址的，自然是不连续的，而链表的遍历方向是由低地址向高地址。堆的大小受限于计算机中有效的虚拟内存。由此可见，堆或许的空间比较灵活，也比较大。

在面向对象编程中，对象都存在堆中，现在不管是 Objecive-C 和 Java 都编译器都为我们实现了内存的回收，不需要手动释放。

一般有程序员分配，并负责释放。若不释放会造成内存泄露，但最终还是会在程序退出的时候被操作系统回收。对于面向对象编程的语言，我们的对象就存储在堆中，深层次的将我们是在面向「堆」编程。

**全局/静态区**

用于存放全局变量和静态变量，未初始化的全局变量和静态变量分配在一块，初始化后的全局变量和静态变量分配在相邻的一块。全局/静态变量赋值后会一直一直存在内存中，直到程序退出的时候才被释放。

- 静态变量： 指使用 static 关键字修饰的变量，只初始化一次（地址不会变了），但可以修改，static 关键字用于限制变量的作用域，防止与全局变量冲突，类似全局变量，但是作用域受到限制，具体的限制如下：
	- 在函数外定义：全局变量，但是只在当前文件中可见（叫做 internal linkage）
	- 在函数内定义：全局变量，但是只在此函数内可见（同时，在多次函数调用中，变量的值不会丢失）
	- （C++）在类中定义：全局变量，但是只在此类中可见


**常量区**

用于存放常量，一旦定义之后就不能被修改。程序退出后释放。

**程序代码区**

存放编译后的二进制代码。


## 指针
指针是 C 语言进阶的门槛之一，了解到一些人就是因为指针而没能在编程领域进阶下去。其实指针是跟内存密不可分的一个概念，变相的可以说它是为了我们能够更好的理解内存，而引出的内存字面量的概念。

一段内存就像一片城市，城市中有着密密麻麻的街区，为了区分它们给家家户户编了一个地址。所以内存也有对应的地址，它们以「字节」为单位。相邻地址之间最小差一个字节。而这些地址长的像这样的`0xaffee345`，长度跟操作系统具体的位数有关。指针就是用来存放像这样的内容的。而这块地址上面可能存放在某些数据，我们想要使用指针来访问它。操作是这样：

当前指针 -> 取指针的内容(`0xaffee345`) -> 取(`0xaffee345`)地址中的内容

所以我们常常将指针操作称为间接访问。

「当前指针」保存着某个内存的地址，那么「当前指针」当然也会占用一段内存的，指针的内存会自动的被分配。正是因为这两个「内存」使得指针让人望而生畏，真正理解之后会发现指针让编程变得更有趣！

## MRC
引用计数是 Objective-C 内存管理的核心技术，将一个对象被引用的时候进行计数，引用一次计数+1，释放一次计数-1，直到计数为零时将这个对象的内存回收。

这种说法在关于引用计数的文章里面必提的概念，然而这并没有涉及到任何内存相关的内容。更深层的理解引用计数应该从栈和堆的角度，我们所说的对象是分配在堆内存上的，我们管理的其实也就是堆内存，然而我们怎么管理？答案是使用指针。

ObjC 对象都是通过指针来操作的，而绝大多数指针是保存在栈上的。来看个 ARC 中的例子

```objC
NSObject *obj = [NSObject new];
[obj method];
NSObject *newObj = obj;
```

在等号右边我们创建了一个对象，并将其赋值给了等号左边指针。实际的操作是在堆内存中开辟了一块内存存放刚创建的对象，并将这个对象的地址传给了栈中指针 `obj`。此时 `obj` 持有了这个对象，该对象的引用计数被记为为`1`。

接着看下一行，我们常常会专业的这么描述：向「对象`obj`」发送 `method` 消息。其实严格意义上讲这种说法是错误的， `obj` 并不算是一个对象，而是一个指针，是对象的一个引用，真正的对象是在等号的右边，位于堆中。我们是通过指针间接的实现了向这个对象发送`method`消息。

> 引用对象：指针的内容是某个对象的地址

最后一行，将 `Obj` 指针赋值给了一个新的指针，这时`newObj`指针也持有了`[NSObject new]` 出来的对象，对象的引用计数变为`2`。

现在出现了两个指针，为了日常编程中方便描述才将 `obj` 和 `newObj` 称为对象，这也更符合我们「面向对象编程」的概念，但是我们心里要清楚的知道真正的对象在哪里。此时 `obj` 和 `newObj` 持有同一个对象。


MRC 环境中需要使用 `retain` 和 `release` 手动来管理对象的引用计数。

```objC
// 自己生成自己持有的对象 引用计数为 1
id obj = [[NSObject alloc] init];
// 取的堆中存在的对象，但自己不持有
id newObj = obj;
// 持有非自己生成的对象 引用计数为 2
[newObj retain];
// 释放 obj 对象 引用计数为 1
[obj release]; 
// 释放 newObj 对象 引用计数为 0
[newObj release];
// 引用计数为0，对象自动调用`dealloc`方法，该段内存被回收
//
// 奔溃，无法释放非自己持有的对象（该内存已被回收）
// [newObj release];
```


## ARC
ARC 是苹果引入的一种自动内存管理机制，会根据引用计数自动监视对象的生存周期，实现方式是在编译时期自动在已有代码中插入合适的内存管理代码以及在 Runtime 做一些优化。

在 ARC 中为了管理对象的引用计数，引入了对象所有权修饰符。

- __strong
- __weak
- __unsage_unretained
- __autorelease

#### __strong

`__strong` 是 `id` 类型和对象类型默认的所有权修饰符。如下代码：

```objC
id obj = [[NSObject alloc] init];
```
`obj` 对象没有明确的给出对象所有权修饰符，默认是 `__strong` 修饰符。

使用 `__strong` 修饰的指针在被赋值之后会使得对内存中的对象的引用计数`+1`，而在指针变量操作作用域范围之后，自动将对象的引用计数`-1`。如下代码：

```objC
{
	// 自己生成并持有对象 对象的引用计数为 1
	id __strong obj = [[NSObject alloc] init];
}
// 指针变量操作作用域返回，在栈内存中被弹出，此时创建的 `NSObject` 的没有人持有，引用计数为0，编译器自动的执行 `dealloc` 将该对象内存回收。
```

当然我们也可以手动的让对象被回收，比如：

```objC
id __strong obj = [[NSObject alloc] init];
obj = nil
```
手动使用 `nil` 将 `obj` 指针销毁，那么其创建的对象失去了所有者，就会自动的被释放。


#### __weak

从前面的 `__stong` 似乎可以很好的解决在 ARC 中的内存管理问题了。但遗憾的是，仅通过 `__strong` 修饰符是不能解决有些重大问题的。

比如很典型的「循环引用」问题。

`__weak` 修饰的指针变量不持有对象，即不改变对象的引用计数。比如有如下声明：
```objC
id __weak obj = [[NSObject alloc] init];
```
上述附有 `__weak` 修饰对象的表达式会报警告，意思是将弱引用的指针变量在赋值之后，当对象超出作用域范围会立即被释放。对象释放后 `obj` 指针会被赋值为 `nil`

然而如下代码是可以正确打印的

```objC
NSNumber __weak *number = [NSNumber numberWithInt:100];
NSLog(@"number = %@", number);
```
这种情况下，weak 并不会立即释放，这涉及到 weak 的实现原理，它通过 `objc_loadWeak` 这个方法注册到 AutoreleasePool 中，来延长了生命周期，在 `[pool drain]` 的时候回自动的被回收。


#### __unsafe_unretained

附有 `__unsafe_unretained` 修饰符的变量同附有 `__weak` 修饰符的变量一样，自己生成的对象自己不持有，所以生成的对象如果没有其他强引用的指针，在超出作用域范围也会被的收回。与 `__weak` 不同的是，该对象释放后被 `__unsafe_unretained` 修饰的指针变量不会被置为 `nil`，它被称为「野指针」。如果继续向该指针变量（对象）发送消息，程序会出现不可预测的错误。


#### __autorelease

ARC 有效时，要通过将对象赋值给附有 `__autorelease` 修饰符的变量来替代调用 `-autorelease` 方法。对象赋值给附有 `__autorelease` 修饰符的变量等价于在 ARC 无效时调用对象的 `-autorelease` 方法，即对象被注册到 autoreleasepool。

## Autorelease Pool

#### autorelease

在 MRC 时代 autorelease 可以说是「自动」管理对象内存的一种方式。调用 autorelease 方法的对象，可以实现对象的延迟销毁。

执行 autorelease 方法的对象会像 C 语言的局部变量一样，当其超出作用域范围的时候，自动的调用 release 实例方法释放对象。那么执行 autorelease 方法的对象具体是在什么时候释放的呢？

autorelease 的具体使用方法如下：
（1）生成并持有 NSAutoreleasePool 对象；
（2）调用已分配对象的 autorelease 实例方法；
（3）废弃 NSAutoreleasePool 对象。

具体源代码如下：
```objC
NSAutoreleasePool *pool = [[NSAutoreleasePoll alloc] init];
id obj = [[NSObject alloc] init];
[obj autorelease];
[pool drain];
```
在执行`drain`方法的的时候等同调用于`[obj release]`。

而在我们实际的编程环境中并不需要自己来操作 NSAutoreleasePool，应用在启动的时候会创建 NSRunLoop， NSRunLoop 每次循环的时候都都会废弃旧的 NSAutorereleasePool 对象并生成新的对象的操作。关于 NSRunLoop 的具体内容可以参看[深入理解 NSRunLoop](http://blog.ibireme.com/2015/05/18/runloop/)

关于 Autorelease 的具体剖析例子，以及底层实现原理，可以看 sunnyxx 的[黑幕背后的Autorelease](http://blog.sunnyxx.com/2014/10/15/behind-autorelease/)

## 对象属性

对象属性(property) 实际上就是一种语法糖，每个属性背后都有实例变量(Ivar)做支持，编译器会帮我们自动生成有关的 setter 和 getter

假设我们有一个 property 定义如下：

```objC
@property (nonatomic, retain) NSObject *property;
```
在 ARC 下我们只要这样赋值就可以了

```objC
self.property = [[NSObject alloc] init];
```

而在 MRC 下会复杂很多，而只有能够熟练的的在 MRC 下管理对象的生命周期才能对内管管理有深入的理解。

在 MRC 下，我们应该使用：

```objC
self.property = [[[NSObject alloc] init] autorelease];
```

然后在 dealloc 方法中加入：

```objC
[_property release];
_property = nil;
```
	
这样内存的情况大体是这样的：

1. init 把 retain count 增加到 1
2. 赋值给 self.property ，把 retain count 增加到 2
3. 当 runloop circle 结束时，autorelease pool 执行 drain，把 retain count 减为 1
4. 当整个对象执行 dealloc 时， release 把 retain count 减为 0，对象被释放
可以看到没有内存泄露发生。

如果我们只是使用：


```objC
self.property = [[NSObject alloc] init];
```
	
这一条语句会导致 retain count 增加到 2，而我们少执行了一次 release，就会导致 retain count 不能被减为 0 。

另外，我们也可以使用临时变量：

```objC
NSObject *a = [[NSObject alloc] init];
self.property = a;
[a release];
```

这种情况，因为对 a 执行了一次 release，所有不会出现上面那种 retain count 不能减为 0 的情况。

**注意：**现在大家基本都是 ARC 写的比较多，会忽略这一点，但是根据上面的内容，我们看到在 MRC 中直接对 self.proprety 赋值和先赋给临时变量，再赋值给 self.property，确实是有区别的。

#### 参考链接：

[https://hit-alibaba.github.io/interview/basic/arch/Memory-Management.html](https://hit-alibaba.github.io/interview/basic/arch/Memory-Management.html)
