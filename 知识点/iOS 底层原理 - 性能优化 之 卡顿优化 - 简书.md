\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[www.jianshu.com\](https://www.jianshu.com/p/e2b9cb1a9e61)

[![](https://upload.jianshu.io/users/upload_avatars/1319505/9754fac0eb59?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96/format/webp)](https://www.jianshu.com/u/40dce98b3416)

2020.03.20 10:32:34 字数 3,035 阅读 28

#### 面试题引发的思考：

**Q: 列表卡顿的原因？如何优化？**

*   **卡顿主要是因为在主线程执行了比较耗时的操作。**

**Q: 卡顿解决方法？**

*   **CPU：**
    *   **尽量用轻量级的对象：`CALayer`、`int`等；**
    *   **不要频繁调用`UIView`的相关属性；**
    *   **提前计算好布局；**
    *   **`Autolayout`会比直接设置`frame`消耗更多的 CPU 资源；**
    *   **图片的`size`最好刚好跟`UIImageView`的`size`保持一致；**
    *   **控制一下线程的最大并发数量；**
    *   **尽量把耗时的操作放到子线程，充分利用 CPU 的多核。**
*   **GPU：**
    *   **尽量避免短时间内大量图片的显示，尽可能将多张图片合成一张进行显示**
    *   **GPU 能处理的最大纹理尺寸是 4096x4096**
    *   **尽量减少视图数量和层级**
    *   **减少透明的视图（`alpha<1`），不透明的就设置`opaque`为`YES`**
    *   **尽量避免出现离屏渲染**

**Q: 如何检测卡顿？**

*   **通过添加`Observer`到主线程 RunLoop 中，监听 RunLoop 状态切换的耗时，以达到监控卡顿的目的。**

**Q: 何为离屏渲染？**

*   **当前屏幕渲染：GPU 在当前屏幕缓冲区 进行渲染操作；**
*   **离屏渲染：GPU 在当前屏幕缓冲区以外 新开辟一个缓冲区进行渲染操作。**

**Q: 离屏渲染触发时机？**

*   **图层属性的混合体 在没有预合成之前，不能直接在屏幕中绘制时，会触发离屏渲染。**

**Q: 为什么离屏渲染消耗性能？为何要避免离屏渲染？**

*   **创建新的缓冲区；**
*   **多次切换上下文切换。**
*   **触发离屏渲染时，会增加 GPU 的工作量，可能会导致 CPU 和 GPU 的总工作耗时超出 16.7ms，进而导致 UI 的卡顿和掉帧。**

**Q: 哪些操作会触发离屏渲染？**

*   **圆角：**`cornerRadius`和`masksToBounds`同时设置时；  
    解决方案：UI 提供圆角图片、**CoreGraphics** 绘制裁剪圆角；
*   **阴影：**`layer.shadowXXX`  
    如果设置了`layer.shadowPath`就不会产生离屏渲染。
*   **光栅化：**`layer.shouldRasterize = YES`；
*   **图层蒙版：**`layer.mask`；

* * *

### 1\. 屏幕成像原理

#### (1) CPU 和 GPU 的作用

在屏幕成像过程中，CPU 和 GPU 起着至关重要的作用：

> *   **CPU (Central Processing Unit，中央处理器)：**  
>     **作用：对象的创建和销毁、对象属性的调整、布局计算、文本的计算和排版、图片的格式转换和解码、图像的绘制 (Core Graphics)；**
> *   **GPU (Graphics Processing Unit，图形处理器)：**  
>     **作用：纹理的渲染。**

* * *

#### (2) 屏幕成像流程

屏幕成像流程如下：

![](http://upload-images.jianshu.io/upload_images/1319505-90a2b016e274ed02.png)

屏幕成像流程

> *   **CPU** 进行 UI 布局、文本计算、图片解码等等，计算完毕将数据提交给 **GPU**；
> *   **GPU** 对数据进行渲染，渲染完毕将数据放到 **帧缓存** 里面；
> *   **视频控制器** 从 **帧缓存** 读取数据；并将数据显示到 **屏幕** 上。

在 iOS 中是**双缓冲机制，有前帧缓存、后帧缓存**。上图的帧缓存有两块区域，当一块区域满了或一块区域正在进行其他操作，此时 GPU 会使用另外一块区域缓存，提升效率。

* * *

#### (3) 屏幕成像原理

屏幕成像原理如下：

![](http://upload-images.jianshu.io/upload_images/1319505-92def113db07d8ef.png)

屏幕成像原理

手机屏幕上的动画是通过一帧一帧 (或者说一页) 数据组成的。

> *   当屏幕需要显示一帧数据的时候，会发送一个**垂直同步信号**；然后逐行发送**水平同步信号**，直到填充整个屏幕，此时一帧数据就显示完成。
> *   接下来再发送一个**垂直同步信号**；然后逐行发送**水平同步信号**，直到完成这一帧。

*   当所有帧数据发送完成之后，这些帧连起来就是屏幕上的动画了。

* * *

### 2\. 卡顿产生原因

![](http://upload-images.jianshu.io/upload_images/1319505-5e1fad9a0681aec3.png)

屏幕成像流程

如上图，红色箭头是 CPU 计算需要的时间，蓝色箭头是 GPU 渲染需要的时间。

注意：发送一个**垂直同步信号**，就会立即把 **CPU** 计算和 **GPU** 渲染完成的数据从**帧缓存**中读取显示到**屏幕**上，并且立即开始下一帧的操作。

*   第一帧的数据需要显示，发送一个**垂直同步信号**，此时将 **CPU** 计算和 **GPU** 渲染完成的数据从**帧缓存**中读取显示到**屏幕**上，就完成了第一帧的显示。
    
*   第二帧的数据 **CPU** 计算和 **GPU** 渲染的比较快，在下一次**垂直同步信号**来之后，将 **CPU** 计算和 **GPU** 渲染完成的数据从**帧缓存**中读取显示到**屏幕**上，完成了第二帧的显示。
    
*   第三帧的数据 **CPU** 计算和 **GPU** 渲染的比较慢，在下一次**垂直同步信号**来到之后还没处理完毕，此时**帧缓存**里面还是第二帧的数据，将第二帧的数据从**帧缓存**中读取显示到**屏幕**上，如此便产生**掉帧现象**，也就是我们说的**卡顿**。
    
*   第三帧的数据会在下一帧的**垂直同步信号**来到之后再显示到**屏幕**上，慢了一帧的时间。
    

* * *

### 3\. 卡顿解决方法

卡顿解决方法是尽可能减少 CPU、GPU 资源消耗。

> *   一般**帧率 FPS(Frames Per Second 每秒传输帧数)** 达到 60FPS 就会感觉不到卡顿；
> *   按照 60FPS 的刷帧率，每隔 16ms 就会有一次 VSync 信号 (1000ms / 60 = 16.667ms)。

#### (1) 卡顿优化 CPU

#### 1> CPU 卡顿优化方式

*   尽量用轻量级的对象，比如无需事件处理的地方使用`CALayer`取代`UIView`、能用`int`就不用`NSNumber`；
    
*   不要频繁地调用`UIView`的相关属性，比如`frame`、`bounds`、`transform`等属性，尽量减少不必要的修改；
    
*   尽量提前计算好布局，在有需要时一次性调整对应的属性，不要多次修改属性；
    
*   `Autolayout`会比直接设置`frame`消耗更多的 CPU 资源，因为`Autolayout`本身性能就不是很高；
    
*   图片的`size`最好刚好跟`UIImageView`的`size`保持一致，如果不一致 CPU 就会对图片进行伸缩操作，这样比较消耗 CPU 资源；
    
*   控制一下线程的最大并发数量，不要无限制的并发，这样比较消耗 CPU 资源；
    
*   尽量把耗时的操作放到子线程，充分利用 CPU 的多核。
    

#### 2> 耗时操作：文本处理 (尺寸计算、绘制)

比如：`boundingRectWithSize`计算文字宽高是可以放到子线程去计算的，或者`drawWithRect`文本绘制，也是可以放到子线程去绘制的，如下：

```
\- (void)text {
    
    
    \[@"text" boundingRectWithSize:CGSizeMake(100, MAXFLOAT) 
    options:NSStringDrawingUsesLineFragmentOrigin attributes:nil context:nil\];

    
    \[@"text" drawWithRect:CGRectMake(0, 0, 100, 100) 
    options:NSStringDrawingUsesLineFragmentOrigin attributes:nil context:nil\];
}


```

#### 3> 耗时操作：图片处理 (解码、绘制)

我们经常会写如下代码加载图片：

```
\- (void)image {
    UIImageView \*imageView = \[\[UIImageView alloc\] init\];
    imageView.frame = CGRectMake(100, 100, 100, 56);
    
    imageView.image = \[UIImage imageNamed:@"timg"\]; 
    \[self.view addSubview:imageView\];
    self.imageView = imageView;
}


```

通过`imageNamed`加载图片，加载完成后是不会直接显示到屏幕上面的，因为加载后的是经过压缩的图片二进制，当真正想要渲染到屏幕上的时候再拿到图片二进制解码成屏幕显示所需要的格式，然后进行渲染显示，而这种解码一般默认是在主线程操作的，如果图片数据比较多比较大的话也会产生卡顿。

一般的做法是在子线程提前解码图片二进制，不需要主线程解码，这样在图片渲染显示之前就已经解码出来了，主线程拿到解码后的数据进行渲染显示就可以了，这样主线程就不会卡顿。网上很多图片处理框架都有这个异步解码功能的。下面演示一下：

![](http://upload-images.jianshu.io/upload_images/1319505-658695eabc6ffbf5.png)

图片异步解码

上面代码，不单单通过`imageNamed`加载的本地图片可以提前渲染，通过`imageWithContentsOfFile`加载的网络图片也可以这样进行提前渲染，只要获取到`UIImage`对象都可以对`UIImage`对象进行提前渲染。

* * *

#### (2) 卡顿优化 GPU

#### 1> GPU 卡顿优化方式

*   尽量避免短时间内大量图片的显示，尽可能将多张图片合成一张进行显示，这样只渲染一张图片，渲染更快；
    
*   GPU 能处理的最大纹理尺寸是 4096x4096，一旦超过这个尺寸，就会占用 CPU 资源进行处理，所以纹理尽量不要超过这个尺寸；
    
*   尽量减少视图数量和层级，视图层级太多会增加渲染时间；
    
*   减少透明的视图（`alpha<1`），不透明的就设置`opaque`为`YES`，因为一旦有透明的视图就会进行很多混合计算增加渲染的资源消耗；
    
*   尽量避免出现离屏渲染。
    

#### 2> 离屏渲染

在 OpenGL 中，GPU 有 2 种渲染方式：

*   **On-Screen Rendering：当前屏幕渲染，在当前用于显示的屏幕缓冲区进行渲染操作；**
*   **Off-Screen Rendering：离屏渲染，在当前屏幕缓冲区以外新开辟一个缓冲区进行渲染操作。**

当前用于显示的屏幕缓冲区就是下图的帧缓存：

![](http://upload-images.jianshu.io/upload_images/1319505-90a2b016e274ed02.png)

屏幕成像流程

Q: 离屏渲染触发时机？

*   **图层属性的混合体 在没有预合成之前 不能直接在屏幕中绘制时，会触发离屏渲染。**

Q: 为什么离屏渲染消耗性能？为何要避免离屏渲染

*   **创建新的缓冲区；**
*   **多次切换上下文切换。**
*   **触发离屏渲染时，会增加 GPU 的工作量，可能会导致 CPU 和 GPU 的总工作耗时超出 16.7ms，进而导致 UI 的卡顿和掉帧。**

Q: 哪些操作会触发离屏渲染？

*   **圆角：**`cornerRadius`和`masksToBounds`同时设置时；  
    解决方案：UI 提供圆角图片、**CoreGraphics** 绘制裁剪圆角；
*   **阴影：**`layer.shadowXXX`  
    如果设置了`layer.shadowPath`就不会产生离屏渲染。
*   **光栅化：**`layer.shouldRasterize = YES`；
*   **图层蒙版：**`layer.mask`；

Q: 为什么要开辟新的缓冲区？

因为上面进行的那些操作比较耗性能、资源，当前屏幕缓冲区不够用 (就算是双缓冲机制也不够用)，所以才会开辟新的缓冲区。

* * *

#### (3) 卡顿检测

“卡顿” 主要是因为在主线程执行了比较耗时的操作；

通过添加`Observer`到主线程 RunLoop 中，监听 RunLoop 状态切换的耗时，以达到监控卡顿的目的。

如下图，主线程的大部分操作，比如点击事件的处理，`view`的计算、 绘制等基本上都在`source0`和`source1`。我们只要监控一下从结束休眠 (步骤 08) 处理`soure1`(步骤 09-C) 一直到绕回来处理`source0`(步骤 05)， 如果发现中间消耗的时间比较长，那么就可以证明这些操作比较耗时。

![](http://upload-images.jianshu.io/upload_images/1319505-11f02f49f574ec85.png)

RunLoop 运行逻辑

借助可以监控哪个方法卡顿的第三方库 **<LXDAppFluecyMonitor>**，进行检测：

```
\- (void)viewDidLoad {
    \[super viewDidLoad\];
    
    \[\[LXDAppFluecyMonitor sharedMonitor\] startMonitoring\];
    \[self.tableView registerClass: \[UITableViewCell class\] forCellReuseIdentifier: @"cell"\];
}

#pragma mark -
#pragma mark - UITableViewDataSource
- (NSInteger)tableView: (UITableView \*)tableView numberOfRowsInSection: (NSInteger)section {
    return 1000;
}

- (UITableViewCell \*)tableView: (UITableView \*)tableView cellForRowAtIndexPath: (NSIndexPath \*)indexPath {
    UITableViewCell \* cell = \[tableView dequeueReusableCellWithIdentifier: @"cell"\];
    cell.textLabel.text = \[NSString stringWithFormat: @"%lu", indexPath.row\];

    if (indexPath.row > 0 && indexPath.row % 30 == 0) {
        
        sleep(2.0);
    }
    return cell;
}


```

运行以上代码，打印结果如下：

![](http://upload-images.jianshu.io/upload_images/1319505-19b0ba7d2823ab6c.png)

打印结果

由打印结果可知：可以检测到`cellForRowAtIndexPath`的卡顿。

下面简单介绍下 **LXDAppFluecyMonitor** 框架的核心代码：

**LXDAppFluecyMonitor** 框架里面就两个文件：  
**LXDBacktraceLogger** 文件里面是关于方法调用栈的一些代码；  
**LXDAppFluecyMonitor** 文件就是卡顿检测文件。

进入 **LXDAppFluecyMonitor** 文件的`startMonitoring`方法：

![](http://upload-images.jianshu.io/upload_images/1319505-f49dd9bc445faf92.png)

startMonitoring 方法

"小礼物走一走，来简书关注我"

还没有人赞赏，支持一下

[![](https://upload.jianshu.io/users/upload_avatars/1319505/9754fac0eb59?imageMogr2/auto-orient/strip|imageView2/1/w/100/h/100/format/webp)](https://www.jianshu.com/u/40dce98b3416)

总资产 1 (约 0.16 元) 共写了 4.8W 字获得 24 个赞共 23 个粉丝

### 被以下专题收入，发现更多相似内容

### 推荐阅读[更多精彩内容](https://www.jianshu.com/)

*   探索底层原理，积累从点滴做起 往期回顾 iOS 底层原理探索 — OC 对象的本质 iOS 底层原理探索 — class...
    
    [![](https://upload-images.jianshu.io/upload_images/1760191-9747305ab3b475d4.png?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)](https://www.jianshu.com/p/824930a58c1a)
*   昨天没写日记，昨天工地上没什么事。现在已经回到上高了，昨天吃晚饭前说好的给我们两个实习生先放假，然后我们兴奋...
    
*   说起警犬大家会觉得它们 勇敢、专业而又神秘 警犬可谓是警队里的 “小明星” 深受人们的喜爱 每次一出现立即 成为大家...
    
*   2017 年 3 月 6 日 晴 压水井的年代 我是几岁或是十几岁 夏天有知了的暑假好长 跳的很高盛出一股股清爽 我调皮就咕...
    
    [![](https://upload.jianshu.io/users/upload_avatars/3097951/6a844a33cb87.jpg?imageMogr2/auto-orient/strip|imageView2/1/w/48/h/48/format/webp)鲜栗子](https://www.jianshu.com/u/354eb4dd3217)阅读 0
    
    [![](https://upload-images.jianshu.io/upload_images/9837085-e90f1e3567b2abaf.jpg?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)](https://www.jianshu.com/p/b6f348c6fe9b)