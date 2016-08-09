---
title: Frame 和 Bounds 的区别
date: 2016-08-09 20:08:55
tags: 
---

**Bounds:** 描述了该视图在其自身坐标系中的位置和大小。

**Frame:** 定义该视图在其父视图中的位置和大小。

日常开发中，我们通过视图的 Frame 来确定视图的位置和大小； 但是现在有一个问题，如果修改视图的 Bounds 对自其 Frame 和子视图的 Frame 会不会有什么影响？

<!--more-->

先来看一段代码：

```ObjC

    UIView *lightGrayView = [UIView new];
    lightGrayView.backgroundColor = [UIColor lightGrayColor];
    lightGrayView.frame = CGRectMake(0, 100, 300, 300);
    [self.view addSubview:lightGrayView];
    
    UIButton *blueButton = [UIButton buttonWithType:UIButtonTypeCustom];
    blueButton.backgroundColor = [UIColor blueColor];
    blueButton.frame = CGRectMake(0, 0, 100, 100);
    [lightGrayView addSubview:blueButton];
    
    UIButton *redButton = [UIButton buttonWithType:UIButtonTypeCustom];
    redButton.backgroundColor = [UIColor redColor];
    redButton.frame = CGRectMake(0, 200, 100, 100);
    [lightGrayView addSubview:redButton];
   
```

这段代码创建了一个 lightGrayView，并将 blueButton 和 redButton 设为其子视图。

![image](http://7xsto7.com1.z0.glb.clouddn.com/20160809_bounds_origin.png)

现在我们修改 lightGrayView 的 Bounds:

	lightGrayView.bounds = CGRectMake(0, 100, 300, 300);

此时视图位置关系变为了：

![image](http://7xsto7.com1.z0.glb.clouddn.com/20160809_bounds_modified.png)

这是怎么回事？

打印各视图的 Frame：

View | Frame
---|---
lightGrayView | frame = (0 100; 300 300)
redButton  | frame = (0 0; 100 100)
blueButton | frame = (0 200; 100 100)

各个视图的 Frame 并没有发生变化，说明修改 Bounds 对 Frame 不会有任何的影响。

那么如何解释 redButton 和 blueButton 位置的上移呢？
其实修改视图的 Bounds 可以理解为修改了视图自身的坐标系，从而改变子视图的相对位置。看图：

![image](http://7xsto7.com1.z0.glb.clouddn.com/20160809_bounds_sys_origin.png)
修改后：
![image](http://7xsto7.com1.z0.glb.clouddn.com/20160809_bounds_sys_mofified.png)

修改 lightGrayView Bounds 后相对自身的坐标系从(0, 0, 100, 300)变为(0, 100, 300, 300)，其结果等同于将自身的坐标系上移了100px。子视图为了保持在其父视图 lightGrayView 上的 Frame 不变，所以也跟随的坐标系一起上移。

我们可以使用公式来计算视图的视图的 origin：

```
CompositedPosition.x = View.frame.origin.x - Superview.bounds.origin.x;
CompositedPosition.y = View.frame.origin.y - Superview.bounds.origin.y;
```

当父视图的 Bounds 的 origin 是{0，0} 时，得到的公式为：

```
	CompositedPosition.x = View.frame.origin.x;
	CompositedPosition.y = View.frame.origin.y;
```

当父视图 Bounds 的 origin 是{0, 100}时，子视图的相对位置就上移了 100px.
 
默认情况下，视图的子视图如果超出父视图的范围（Bounds）仍旧可见（clipsToBounds = YES）,但无法响应用户的点击事件，所以这里的 blueButton 在 lightGrayView 修改 Bounds 之后超出的父视图的 Bounds 点击事件就失效。

现在给 lightGrayView 添加一个 yellowButton 子视图，该视图位于 lightGrayView Bounds 外，显然是无法响应点击事件的。效果如下：


![image](http://7xsto7.com1.z0.glb.clouddn.com/20160809_bounds_append_yellow.png)


当修改 lightGrayView Bounds 后，yellowButton 进入的 lightGrayView 的 Bounds，此时 yellowButton 就可以响应点击事件了。

![image](http://7xsto7.com1.z0.glb.clouddn.com/20160809_bounds_append_yellow_after.png)


**View 的 Bounds 对于其子视图来说就相当于一个窗口，修改 Bounds 就如同移动这个窗口，用来展示本来位于视图外部的子视图，以及响应视图的点击事件。嗯，可以联想一下 UIScrollView。**
