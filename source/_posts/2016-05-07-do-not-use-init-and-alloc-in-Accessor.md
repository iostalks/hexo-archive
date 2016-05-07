---
title: 不要在 init 和 dealloc 中调用存取方法
date: 2016-05-07 11:41:39
tags:
---

在整理上一篇[内存管理](http://blog.iostalks.com/2016/05/04/C-ObjC-manage-memory/)的时候，查阅官方文档里内存管理文章。

文章中有一节题目是：Don’t Use Accessor Methods in Initializer Methods and dealloc

意思是：不要在 `init` 和 `dealloc` 方法中调用 Accessor，而是应该使用成员变量，代码如下：

<!--more-->

```objC
- (id)init { 
     self = [super init]; 
     if (self) {
          _count = [[NSNumber alloc] initWithInteger:0]; 
     }
     return self;
}
```

对于带参数的 `init` 方法应该这样：

```objC
- (id)initWithCount:(NSNumber *)startingCount { 
     self = [super init]; 
     if (self) {
          _count = [startingCount copy];
     }
     return self;
}
```

对于在 `dealloc` 中，对应的写法应该是调 `release`:

```objC
- (void)dealloc { 
     [_count release]; 
     [super dealloc];
}
```

官方没有给出具体原因，为什么不能用。

记得之前看过[唐巧的技术博客](http://blog.devtang.com/2011/08/10/do-not-use-accessor-in-init-and-dealloc-method/)的相关的文章，里面提到的原因是：

> 在 init 和 dealloc 中，对象的存在与否还不确定，所以给对象发消息可能不会成功。

开始觉得是很疑惑的，这里指出的「对象」是指 `self`，而我们在 init 方法中都是使用 `if` 对 `self` 做过判断，能确保对象的存在的。

又在 stackoverflow 搜了一通，看到个比较有道理的说法（看了半天才看懂。。），意思是这样的：

>可能有子类重载了 Accessors 方法，并在 Accessors 做了一些其他的操作，操作中会认为子类已经初始化完成了，从而使用了 `self`。而这个 Accessors 方法会在子类初始化的时候被调用，然而这时子类还没有初始化完成，那么在 Accessors 方法中使用了 `self` （或者其他未初始化的变量）可能程序会出错。

>同理在 `delloc` 中调用了 Accessors， 可能在 Accessors 中使用了已经释放的对象，造成 Crash。

写了个无厘头的子类：

```objC
- (void)initWithCount:(NSNumber *)startingCount {
    self = [super initWithCount:startingCount];
    if (self) {
        self.count = [startingCount copy];
        self.array = [[NSArray alloc] init];
    }
    return self;
}
- (void)setCount:(NSNumber *)count
{
	[super setCount:count];
	[self.array addObject:count]; // array has't alloc
	_count = count
}
```

现在回头看唐大解释似乎是对的，但是没有强调是子类，会让人摸不着头脑。

当然并不是说一定不能在 `init` 方法里面调用 Accessors 比如某个属性是在超类（Super Class）中定义的，那只有通过 `self` 才能够访问到。

## 参考链接
[https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmPractical.html](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmPractical.html)
[http://stackoverflow.com/questions/3424382/why-shoudnt-i-use-accessor-methods-in-init-methods](http://stackoverflow.com/questions/3424382/why-shoudnt-i-use-accessor-methods-in-init-methods)
[http://blog.devtang.com/2011/08/10/do-not-use-accessor-in-init-and-dealloc-method/](http://blog.devtang.com/2011/08/10/do-not-use-accessor-in-init-and-dealloc-method/)
