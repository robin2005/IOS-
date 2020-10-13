\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[www.jianshu.com\](https://www.jianshu.com/p/8194076aa0ff)

[![](https://upload.jianshu.io/users/upload_avatars/1319505/9754fac0eb59?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96/format/webp)](https://www.jianshu.com/u/40dce98b3416)

2018.09.10 09:41:21 字数 1,473 阅读 149

#### 面试题引发的思考：

**Q: RunLoop 内部实现逻辑？**

![](http://upload-images.jianshu.io/upload_images/1319505-11f02f49f574ec85.png)

RunLoop 运行逻辑

**Q: RunLoop 和线程的关系？**

*   **每条线程都有唯一的一个与之对应的 RunLoop 对象；**
*   **RunLoop 保存在一个全局的`Dictionary`里，线程作为`key`，RunLoop 作为`value`；**
*   **主线程的 RunLoop 自动创建，子线程的 RunLoop 需要手动创建；**
*   **RunLoop 在第一次获取线程时创建，在线程结束时销毁。**

**Q: `timer`与 RunLoop 的关系？**

*   **`timer`运行在 RunLoop 里，相当于 RunLoop 来控制`timer`何时执行。**

**Q: RunLoop 是怎么响应用户操作的， 具体流程是什么样的？**

*   **先由`Source1`捕捉触摸事件，然后由`Source0`去处理触摸事件。**

**Q: RunLoop 有哪几种状态？**

![](http://upload-images.jianshu.io/upload_images/1319505-4584464fa1564273.png)

RunLoop 状态

**Q: RunLoop 的`mode`作用是什么？**

*   **把不同模式下的`Source0`/`Source1`/`Timer`/`Observer`隔离开来，互不影响，会使程序在当前模式下流畅运行。**

* * *

### 1\. RunLoop 简介

#### (1) RunLoop 概念

> **RunLoop 顾名思义，运行循环，在程序运行过程中循环做一些事情：**
> 
> *   如果没有 RunLoop 程序执行完毕就会立即退出；
> *   如果有 RunLoop 程序并不会马上退出，而是保持运行状态，等待处理程序的各种事件；
> *   RunLoop 可以保持程序的持续运行，在没有事件处理的时候使程序进入休眠模式，从而节省 CPU 资源，提高程序性能。

![](http://upload-images.jianshu.io/upload_images/1319505-0985dcb25b45f025.png)

没有 RunLoop

代码如上：没有 RunLoop，执行完第 14 行代码后，会立即退出程序。

![](http://upload-images.jianshu.io/upload_images/1319505-d9d2269f9401450c.png)

有 RunLoop

代码如上：有 RunLoop，程序并不会马上退出，而是保持运行状态。

其伪代码如下：

```
int main(int argc, char \* argv\[\]) {
    @autoreleasepool {
        int result = 0;
        do {
            
            int message = sleep\_and\_wait();
            
            result = process\_message(message);
        } while (result == 0);
        return 0;
    }
}


```

RunLoop 确实是`do while`通过判断`result`的值实现的。因此，我们可以把 RunLoop 看成一个死循环。

#### (2) RunLoop 应用

> **RunLoop 应用范畴：**
> 
> *   定时器（`Timer`）、`PerformSelector`
> *   `GCD Async Main Queue`
> *   事件响应、手势识别、界面刷新
> *   网络请求
> *   `AutoreleasePool`

* * *

### 2\. RunLoop 相关概念

#### (1) RunLoop 对象

*   iOS 中有 2 套 API 来访问和使用 RunLoop
*   `Foundation`：`NSRunLoop`
*   `Core Foundation`：`CFRunLoopRef`
*   `NSRunLoop`和`CFRunLoopRef`都代表着 RunLoop 对象
*   `NSRunLoop`是基于`CFRunLoopRef`的一层 OC 包装
*   [CFRunLoopRef 源码](https://links.jianshu.com/go?to=https%3A%2F%2Fopensource.apple.com%2Ftarballs%2FCF%2F)是开源的

```
\- (void)viewDidLoad {
    \[super viewDidLoad\];

    
    \[NSRunLoop currentRunLoop\]; 
    \[NSRunLoop mainRunLoop\]; 

    
    CFRunLoopGetCurrent(); 
    CFRunLoopGetMain(); 
}


```

#### (2) RunLoop 与线程

> *   每条线程都有唯一的一个与之对应的 RunLoop 对象；
> *   RunLoop 保存在一个全局的`Dictionary`里，线程作为`key`，RunLoop 作为`value`；
> *   主线程的 RunLoop 自动创建，子线程的 Runloop 需要手动创建；
> *   RunLoop 在第一次获取线程时创建，在线程结束时销毁。

以上结论都可以由以下[相关源码](https://links.jianshu.com/go?to=https%3A%2F%2Fopensource.apple.com%2Ftarballs%2FCF%2F)总结得出：

![](http://upload-images.jianshu.io/upload_images/1319505-72f289a67c5080bf.png)

CFRunLoopGetCurrent 函数

`CFRunLoopGetCurrent`调用`_CFRunLoopGet0`：

![](http://upload-images.jianshu.io/upload_images/1319505-85b6e208db2b25dd.png)

\_CFRunLoopGet0 函数

#### (3) RunLoop 相关的类

> **`Core Foundation`中关于 RunLoop 的 5 个类：**
> 
> *   `CFRunLoopRef`：获得当前 RunLoop 和主 RunLoop
> *   `CFRunLoopModeRef`：运行模式
> *   `CFRunLoopSourceRef`：事件源，输入源
> *   `CFRunLoopTimerRef`：定时器时间
> *   `CFRunLoopObserverRef`：观察者

由[相关源码](https://links.jianshu.com/go?to=https%3A%2F%2Fopensource.apple.com%2Ftarballs%2FCF%2F)可知：

![](http://upload-images.jianshu.io/upload_images/1319505-c154c4b3165322ba.png)

CFRunLoop 结构体

![](http://upload-images.jianshu.io/upload_images/1319505-5d59a4791f372ad4.png)

CFRunLoopMode 结构体

由以上代码可以得出 RunLoop 五个类直接的对应关系：

![](http://upload-images.jianshu.io/upload_images/1319505-edc148159b3e9c0b.png)

对应关系

> *   一个 RunLoop 包含若干个`Mode`，每个`Mode`又包含若干个`Source0`/`Source1`/`Timer`/`Observer`；
> *   RunLoop 启动时只能选择其中一个`Mode`，作为`_currentMode`
> *   如果需要切换`Mode`，只能退出当前`Loop`，再重新选择一个`Mode`进入；  
>     不同组的`Source0`/`Source1`/`Timer`/`Observer`能分隔开来，互不影响；
> *   如果`Mode`里没有任何`Source0`/`Source1`/`Timer`/`Observer`，RunLoop 会立马退出。
> 
> * * *
> 
> *   **`Source0`：** 触摸事件处理、`performSelector:onThread:`
> *   **`Source1`：** 基于`Port`的线程间通信、系统事件捕捉
> *   **`Timers`：** `NSTimer`、`performSelector:withObject:afterDelay:`
> *   **`Observers`：** 用于监听 RunLoop 的状态、UI 刷新（`BeforeWaiting`）、`Autorelease pool`（`BeforeWaiting`）

##### 1> `CFRunLoopModeRef`

> **常见的 2 种`Mode`：**
> 
> *   **`kCFRunLoopDefaultMode`(`NSDefaultRunLoopMode`)：**  
>     App 的默认`Mode`，通常主线程是在这个`Mode`下运行；
> *   **`UITrackingRunLoopMode`：**  
>     界面跟踪`Mode`，用于`ScrollView`追踪触摸滑动，保证界面滑动时不受其他`Mode`影响

##### 2> `CFRunLoopObserverRef`

RunLoop 的`Observer`状态类型如下：

![](http://upload-images.jianshu.io/upload_images/1319505-4584464fa1564273.png)

RunLoop 状态

监听 RunLoop 状态：

```
\- (void)viewDidLoad {
    \[super viewDidLoad\];
    

    
    CFRunLoopObserverRef observer = CFRunLoopObserverCreateWithHandler(kCFAllocatorDefault, kCFRunLoopAllActivities, YES, 0, ^(CFRunLoopObserverRef observer, CFRunLoopActivity activity) {
        switch (activity) {
            case kCFRunLoopEntry: {
                CFRunLoopMode mode = CFRunLoopCopyCurrentMode(CFRunLoopGetCurrent());
                NSLog(@"kCFRunLoopEntry - %@", mode);
                CFRelease(mode);
                break;
            }
            case kCFRunLoopExit: {
                CFRunLoopMode mode = CFRunLoopCopyCurrentMode(CFRunLoopGetCurrent());
                NSLog(@"kCFRunLoopExit - %@", mode);
                CFRelease(mode);
                break;
            }
            default:
                break;
        }
    });
    
    CFRunLoopAddObserver(CFRunLoopGetMain(), observer, kCFRunLoopCommonModes);
    
    CFRelease(observer);
}


```

在`ViewController`放一个`UITextView`控件，运行程序之后，滑动控件，查看打印结果：

![](http://upload-images.jianshu.io/upload_images/1319505-774a9bbe88c8b225.png)

打印结果

由打印结果可知：

*   控件开始滚动时，先退出`kCFRunLoopDefaultMode`，然后进入`UITrackingRunLoopMode`；
*   控件停止滚动时，先退出`UITrackingRunLoopMode`，然后进入`kCFRunLoopDefaultMode`。

如此便产生了模式的切换。

* * *

### 3\. RunLoop 运行逻辑

#### (1) 官方文档解读

由[苹果官方文档](https://links.jianshu.com/go?to=https%3A%2F%2Fdeveloper.apple.com%2Flibrary%2Farchive%2Fdocumentation%2FCocoa%2FConceptual%2FMultithreading%2FRunLoopManagement%2FRunLoopManagement.html)可知 RunLoop 的运行逻辑如下：

![](http://upload-images.jianshu.io/upload_images/1319505-0e8da6814bdba538.jpg)

RunLoop 运行逻辑

由以上图例可知：  
**RunLoop 在运行循环过程中，接收到`Input sources`或者`Timer sources`时会通过对应处理方式处理；没有事件消息传入的时候，就会使程序处于休眠状态。**

#### (2) 源码解读

下面通过[源码](https://links.jianshu.com/go?to=https%3A%2F%2Fopensource.apple.com%2Ftarballs%2FCF%2F)解读 RunLoop 底层实现。

![](http://upload-images.jianshu.io/upload_images/1319505-c7f14ecac2c71069.png)

RunLoop 入口函数

首先通过运行程序断点可以知道 RunLoop 入口函数为**`CFRunLoopRunSpecific`函数**。

对[相关源码](https://links.jianshu.com/go?to=https%3A%2F%2Fopensource.apple.com%2Ftarballs%2FCF%2F)进行简化，以便解读 RunLoop 底层实现：

![](http://upload-images.jianshu.io/upload_images/1319505-65f75b3ed412df16.png)

RunLoop 底层实现

RunLoop 底层实现代码流程图如下：

![](http://upload-images.jianshu.io/upload_images/1319505-11f02f49f574ec85.png)

RunLoop 运行逻辑

#### (3) 特殊要点 GCD 相关

一般 GCD 是有自己的处理逻辑，不依赖 RunLoop 实现；  
GCD 有一种特殊情况，需要交给 RunLoop 进行处理：

![](http://upload-images.jianshu.io/upload_images/1319505-db30b5eaf7897714.png)

GCD 特殊情况依赖 RunLoop

#### (4) RunLoop 休眠的实现原理

由 [RunLoop 底层实现](https://links.jianshu.com/go?to=https%3A%2F%2Fopensource.apple.com%2Ftarballs%2FCF%2F)可知：

![](http://upload-images.jianshu.io/upload_images/1319505-3352c0ad3858f639.png)

RunLoop 休眠

RunLoop 是通过`__CFRunLoopServiceMachPort`函数来休眠的：

![](http://upload-images.jianshu.io/upload_images/1319505-7f0a2723723298d2.png)

\_\_CFRunLoopServiceMachPort 函数

`__CFRunLoopServiceMachPort`函数主要方法为`mach_msg`函数，`mach_msg`是内核层面的 API，这是程序真正达到休眠的方式。

那么 RunLoop 休眠的实现原理如下：

![](http://upload-images.jianshu.io/upload_images/1319505-6ad46a113033b2eb.png)

RunLoop 休眠的实现原理

RunLoop 在用户态调用`mach_msg()`时，会自动转到内核态，调用内核态的`mach_msg()`，达到真正休眠的目的：  
没有消息就让线程休眠；有消息就唤醒线程，回到用户态处理消息。

"小礼物走一走，来简书关注我"

还没有人赞赏，支持一下

[![](https://upload.jianshu.io/users/upload_avatars/1319505/9754fac0eb59?imageMogr2/auto-orient/strip|imageView2/1/w/100/h/100/format/webp)](https://www.jianshu.com/u/40dce98b3416)

总资产 1 (约 0.16 元) 共写了 4.8W 字获得 24 个赞共 23 个粉丝

### 被以下专题收入，发现更多相似内容

### 推荐阅读[更多精彩内容](https://www.jianshu.com/)

*   昨天课程我们知道了，因为交易会产生大量的费用，所以资源的初始界定就变得很重要，如果资源没有落到用途更高的人手里，这...
    
*   没想到，最后还真是喝上了功夫荼！ 没想到，那堆纸箱和纸盒，最终还真是按照 1 块钱 1 斤的价格卖出去了！ 没想到，那天心...
    
    [![](https://upload.jianshu.io/users/upload_avatars/9411090/f518ddcb-226c-4d01-9954-79873735b040.jpg?imageMogr2/auto-orient/strip|imageView2/1/w/48/h/48/format/webp)见证真理](https://www.jianshu.com/u/de0b5666c34b)阅读 0
    
    [![](https://upload-images.jianshu.io/upload_images/9411090-2add282656323e42?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)](https://www.jianshu.com/p/6bb14858acf4)
*   [![](https://upload-images.jianshu.io/upload_images/8704834-84f95803ecf8677d.jpg?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)](https://www.jianshu.com/p/64318946ced6)