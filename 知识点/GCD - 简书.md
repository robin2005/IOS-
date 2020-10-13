\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[www.jianshu.com\](https://www.jianshu.com/p/92e706c407ee)

[![](https://upload.jianshu.io/users/upload_avatars/3028784/0c2dcc366330?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96/format/webp)](https://www.jianshu.com/u/3502959642ff)

0.2142020.01.18 16:56:06 字数 876 阅读 100

#### 两个特殊队列：

`dispatch_get_main_queue()`主队列，串行队列。  
`dispatch_get_global_queue(0, 0)`全局并发队列。这个队列是系统维护的，会被用来处理很多系统级事件。

#### 最简单的 block:

`dispatch_block_t`这是个无参无返回值的 block。

#### 关于同步异步函数：

`dispatch_sync`同步函数没有开启线程的能力。所有的代码都会在当前线程立即执行。  
`dispatch_async`异步函数有开启线程的能力。

#### 关于串行并行队列：

`dispatch_queue_create(0,0)`  
`DISPATCH_CURRENT_QUEUE_LABEL`串行队列遵循 FIFO 原则，先进先出。  
`DISPATCH_QUEUE_CONCURRENT`并行队列，之间不会相互影响会各自执行。执行顺序与加入队列顺序有关。  
排列组合之后，就有了这么一套机制，如下图：  

![](http://upload-images.jianshu.io/upload_images/3028784-5f50f4aabf5f4dad.png)

image.png

知识点较多的主要是同步函数串行队列。产生堵塞的原因本质上还是任务执行顺序的问题。如下经典代码就会产生堵塞（死锁）：

```
 
    dispatch\_queue\_t queue = dispatch\_queue\_create("xxx", DISPATCH\_QUEUE\_SERIAL);
    NSLog(@"1");
    
    dispatch\_async(queue, ^{
        NSLog(@"2");
        
        dispatch\_sync(queue, ^{
             NSLog(@"3");
        });
          NSLog(@"4");
    });
    NSLog(@"5");


```

分析图如下

![](http://upload-images.jianshu.io/upload_images/3028784-5c6434122c51a21e.png)

image.png

本质上，异步函数里的代码是加入队列后，按顺序执行的。所以会执行`2->同步->4`。  
但是，同步函数的性质又会让代码立即执行。所以在执行同步函数的时候，会要求立即执行`log3`。  
但是`log3`这个任务因为串行队列顺序的原因，必须等到 log4 执行完毕之后才会执行。  
此时发生了`log4`与`log3`相互等待的情况，而产生了堵塞。

这只是同步串行会出现问题的一种方式。单纯的在主线程中使用同步串行，是没有问题，而且借助其一定会按顺序执行的特性。还能达到某些锁的功能：如经典的购票问题：

```
\- (void)saleTickes {
    self.tickets = 20;
    \_queue = dispatch\_queue\_create("xxx", DISPATCH\_QUEUE\_SERIAL);
    while (self.tickets > 0) {
        
        dispatch\_sync(\_queue, ^{
            
            if (self.tickets > 0) {
                self.tickets--;
            }
        });
    }
}


```

代码很简单，不需要解释。这个 demo 只是说明同步串行的效果。其实这段代码意义不大，因为实现购票必定要在异步并发队列里才会更好的达到效果。如下：

```
\- (void)saleTickes {
    NSLock \*lock = \[NSLock new\];
    \_tickets = 20;
    \_queue = dispatch\_queue\_create("Cooci", DISPATCH\_QUEUE\_CONCURRENT);
    while (self.tickets > 0) {
        \[lock lock\];
        dispatch\_async(\_queue, ^{
            
            if (self.tickets > 0) {
                self.tickets--;
                NSLog(@"还剩 %zd %@", self.tickets, \[NSThread currentThread\]);
            } else {
                NSLog(@"没有票了");
            }
            \[lock unlock\];
        });
    }
}



```

#### 栅栏函数：

栅栏函数的用法，多用于控制`并发队列`的执行时机，并且只用于控制唯一一个并发队列 (控制串行队列没有意义)。其中的`barrier`单词很好的说明了他的作用。就是个挡路的：**在我之前的任务都要被我挡住，等我执行完毕之后，之后的任务才会执行**。  
`dispatch_barrier_sync`同步栅栏函数：不仅仅会阻挡 **并发队列的任务**，还会阻挡 **当前线程的任务** 。直至该函数 **之前的并发队列任务** 执行完毕后，才会继续执行 **当前线程的任务** 与 **后面的并发任务** 。  
`dispatch_barrier_async`异步栅栏函数：只阻挡 **并发队列的任务** 。不会阻挡 **当前线程的任务** 。所以 **当前线程的任务** ，都会比 **所有的并发队列的任务** 先执行。  
demo 如下：就不解释了

```
\- (void)demo2{
    dispatch\_queue\_t concurrentQueue = dispatch\_queue\_create("cooci", DISPATCH\_QUEUE\_CONCURRENT);
    
    dispatch\_async(concurrentQueue, ^{
        for (NSUInteger i = 0; i < 5; i++) {
            NSLog(@"download1-%zd-%@",i,\[NSThread currentThread\]);
        }
    });
    
    dispatch\_async(concurrentQueue, ^{
        for (NSUInteger i = 0; i < 5; i++) {
            NSLog(@"download2-%zd-%@",i,\[NSThread currentThread\]);
        }
    });
    
    
    dispatch\_barrier\_sync(concurrentQueue, ^{
        NSLog(@"---------------------%@------------------------",\[NSThread currentThread\]);
    });
    NSLog(@"加载那么多,喘口气!!!");
    
    dispatch\_async(concurrentQueue, ^{
        for (NSUInteger i = 0; i < 5; i++) {
            NSLog(@"日常处理3-%zd-%@",i,\[NSThread currentThread\]);
        }
    });
    NSLog(@"\*\*\*\*\*\*\*\*\*\*起来干!!");
    
    dispatch\_async(concurrentQueue, ^{
        for (NSUInteger i = 0; i < 5; i++) {
            NSLog(@"日常处理4-%zd-%@",i,\[NSThread currentThread\]);
        }
    });
}


```

#### 调度组：

用这个也可以控制任务调度顺序。用法如下：

```
 - (void)groupDemo {
    dispatch\_queue\_t queue = dispatch\_get\_global\_queue(0, 0);
    dispatch\_group\_t group = dispatch\_group\_create();
    
    dispatch\_group\_enter(group);
    dispatch\_async(queue, ^{
        NSLog(@"第一个走完了");
        dispatch\_group\_leave(group);
    });
    
    dispatch\_group\_enter(group);
    dispatch\_async(queue, ^{
        NSLog(@"第二个走完了");
        dispatch\_group\_leave(group);
    });
    
    dispatch\_group\_notify(group, dispatch\_get\_main\_queue(), ^{
        NSLog(@"所有任务完成,可以更新UI");
        
    });
}


```

注意：  
1.`dispatch_group_enter`与 `dispatch_group_leave`需要成对出现。  
2.`dispatch_group_enter` 多于 `dispatch_group_leave` 不会调用通知  
3.`dispatch_group_enter` 少于 `dispatch_group_leave` 会奔溃  
4\. 所有的`dispatch_group_enter`都要在`dispatch_group_notify`之前执行。

"小礼物走一走，来简书关注我"

还没有人赞赏，支持一下

[![](https://upload.jianshu.io/users/upload_avatars/3028784/0c2dcc366330?imageMogr2/auto-orient/strip|imageView2/1/w/100/h/100/format/webp)](https://www.jianshu.com/u/3502959642ff)

总资产 13 (约 1.21 元) 共写了 1.2W 字获得 73 个赞共 23 个粉丝

### 被以下专题收入，发现更多相似内容

### 推荐阅读[更多精彩内容](https://www.jianshu.com/)

*   前言 各路大神对 GCD 的原理解析和使用方法网上到处都是, 可以轻松搜索到。那为什么笔者还要自己动手写一篇所谓的 " 葵花...
    
    [![](https://cdn2.jianshu.io/assets/default_avatar/5-33d2da32c552b8be9a0548c7a4576607.jpg)不董\_](https://www.jianshu.com/u/a101fe5a851c)阅读 0
    
*   简介 GCD（Grand Central Dispatch）是在 macOS10.6 提出来的，后来在 iOS4.0 被引...
    
    [![](https://upload-images.jianshu.io/upload_images/1208202-c8290d87487adaa8.png?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)](https://www.jianshu.com/p/d8abe8aab5bf)