---
title: NSRunLoop 学习笔记
date: 2016-05-25 09:04:47
tags:
---

NSRunLoop 在 iOS 中应该算是基础的概念，但在实际的编码中并不多见。最常用到的地方可能就是在有 ScrollView 的界面，为了防止定时器失效，需要将定时器加入到 NSRunLoopCommonModes 中去。 

然而，RunLoop 的用途远不止与此，我们在给程序下断点后，可以在主线程的调用栈中看到与 RunLoop 相关的函数，我们所写的代码都是通过 NSRunLoop 调用进去的。要深入理解 NSRunLoop 不仅需要反复理解基础概念，更重要的还是到 demo 中感受其运作方式。

<!--more-->

之前有个需求，要在子线程中持续处理定时器事件，就尝试实践了下 NSRunLoop，以下是在研究时提出的一些问题，并给出了理解之后的解答。


###### 1. 什么是 NSRunLoop

一般来说，一个线程一次只能执行一个任务，任务执行完毕后线程便会退出。如果我们想让线程持续的为我们处理事件（比如点击屏幕），就需要有一个循环来阻止线程的退出。但是如果线程中一直在循环某块代码，就如同死循环了，无法再接收外部的事件。

实际中，我们的应用程序是可以随时响应外部事件的，这就是 NSRunLoop 起的作用，RunLoop 为线程提供了事件循环的入口，并管理其需要处理的事件和消息。


###### 2. RunLoop 和线程之间的关系

iOS 中线程默认会绑定有 RunLoop，每一个线程对应一个 RunLoop 对象。我们并不能自己创建 RunLoop，但是可以在当前线程使用 `+ currentRunLoop` 获取，如果不主动获取，它就一直不会有。

NSRunLoop 是基于 CFRunLoopRef  的封装，提供面向对象的 API， 但是 NSRunLoop 是线程不安全的。除了主线程外，只能在当前线程内部获取其 RunLoop。

主线程的 RunLoop 会在 App 启动时自动的运行，子线程需要手动运行。在子线程中若只需要线性执行任务，那可以不用理会 RunLoop ；但是当需要在子线程中处理循环事件时，比如 NSTimer 事件，就必须要手动获取 RunLoop ，并保证线程不退出。

###### 3. RunLoop 退出（返回）是什么意思和线程退出有什么关系？

在自定义线程中，RunLoop 的退出和线程的退出本质上并没有什么联系。接口文档中 RunLoop 有一下几种方式进入循环状态：

	// 运行 NSRunLoop，运行模式为默认的 NSDefaultRunLoopMode 模式，没有超时限制
	- (void)run; 
	
	
	// 运行 NSRunLoop: 参数为运时间期限，运行模式为默认的 NSDefaultRunLoopMode 模式
	- (void)runUntilDate:(NSDate *)limitDate;
	
	
	// 运行 NSRunLoop: 参数为运行模式、时间期限，返回值为 YES 表示是处理事件后返回的，NO 表示是超时、强制停止运行或者当前 mode 下没有任何事件导致的返回
	- (BOOL)runMode:(NSString *)mode beforeDate:(NSDate *)limitDate;

进入循环状态后，就类似进入了一个 do-while 循环。之后只有两种情况，要么退出循环，要么一直循环。退出循环代表 RunLoop 返回了，一直循环若没有事件加入 RunLoop 便会沉睡，接收到下一次事件时又会被唤醒。

若 RunLoop 从循环状态中退出了，且线程的 main 函数中没有其他的循环包裹，那么执行完 main 方法后整个线程便会退出。

为了保证线程的不退出，我们一般会在运行 Runloop 的外层用 while 循环包裹，形势如下：

	- (void)main
	{
		 // 添加事件源
    	[self myInstallCustomInputSource];
        while (!self.isCancelled)
        {
            [self doSomeTask];
            BOOL ret = [[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode
                                                beforeDate:[NSDate distantFuture]];
            NSLog(@"Exiting runloop: %d", ret);
            
        }
	}
	
当线我们需要持续处理事件时，保持外部循环条件为 YES，这样即使 RunLoop 在执行事件时候退出了一次，也不会导致整个线程退出。


###### 4. RunLoop 何时退出？

RunLoop 的退出与当前 mode 下的 item 密切相关。一个 RunLoop 包含若干个 Mode，每个 mode 又包含若干个 Source/Timer/Observer。 在子线程的 main 函数中启动 RunLoop，都需要指定一种 mode，默认情况下为 NSDefaultRunLoopMode。 

Source/Timer/Observer 被称为 mode item，若 mode 中一个 item 也没有，那么 RunLoop 就会从这种 mode 中退出。

我们也可以手动调用 CFRunLoopStop(CFRunLoopRef  rl) 方法来强制 RunLoop 退出，然而它只对 `-runMode:beforeDate:` 启动的 RunLoop 有效，对 `-run` 启动的无效。

处理完 Input Source item 事件之后 RunLoop 就会退出，而处理 Timer item 事件不会导致 RunLoop 退出。

系统提供的几个 -performSelector: API调用的 selector 被分发一次就会导致 RunLoop 退出，然而 performSelector:onThread: 和 performSelector:afterDelay: 比调特殊，它会向当前线程中加入 Timer，当再次进入 RunLoop 时，若无其他事件处理，RunLoop不会退出，而是进入睡眠状态。

###### 5. 什么时候需要用到 RunLoop？

- 需要使用 Port 或者自定义 Input Source 与其他线程进行通信;
- 需要在线程中使用 NSTimer;
- 在子线程外部 `performSelector:onThread:` 方法到该线程。
- 使用子线程执行周期性的任务;
- NSURLConnection 在子线程发起异步请求调用;


参考：
[NSRunLoop Guide](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW1)
[深入理解 NSRunLoop](http://blog.ibireme.com/2015/05/18/runloop/)
[iOS多线程编程Part 1/3 - NSThread & Run Loop](http://www.hrchen.com/2013/06/multi-threading-programming-of-ios-part-1/)
[走进Run Loop的世界 (一)：什么是Run Loop？](http://chun.tips/blog/2014/10/20/zou-jin-run-loopde-shi-jie-%5B%3F%5D-:shi-yao-shi-run-loop%3F/)