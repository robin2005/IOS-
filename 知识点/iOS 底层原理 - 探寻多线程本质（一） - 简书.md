\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[www.jianshu.com\](https://www.jianshu.com/p/3d4d634065e8)

[![](https://upload.jianshu.io/users/upload_avatars/1319505/9754fac0eb59?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96/format/webp)](https://www.jianshu.com/u/40dce98b3416)

2018.11.12 10:23:23 字数 1,185 阅读 210

#### 面试题引发的思考：

**Q: iOS 的多线程方案有哪几种？你更倾向于哪一种？**

*   **常见的方案有四种：Pthread、NSThread、GCD、NSOperation；**
*   **比较常用的是 GCD、NSOperation。**

**Q: GCD 的队列类型？**

*   **并发队列（Concurrent Dispatch Queue）**
*   **串行队列（Serial Dispatch Queue）**

**Q: 何时会产生死锁？**

*   **使用`dispatch_sync`函数 往 当前串行队列 中添加任务，会卡住当前的串行队列（产生死锁）。**

**Q: OperationQueue 和 GCD 的区别，以及各自的优势？**

*   **如下常见的多线程方案介绍**

* * *

### 1\. 常见的多线程方案

![](http://upload-images.jianshu.io/upload_images/1319505-4e3150e40c112882.png)

常见多线程方案

* * *

### 2\. GCD 的常用函数

#### (1) 2 个执行任务的函数：

> **同步执行任务：**
> 
> *   `dispatch_sync(dispatch_queue_t queue, dispatch_block_t block);`
> 
> **异步执行任务：**
> 
> *   `dispatch_async(dispatch_queue_t queue, dispatch_block_t block);`
> 
> **同步、异步区别：能否开启新的线程**
> 
> *   **同步：**在 **_当前线程_** 中执行任务，**_不具备_** 开启新线程的能力；
> *   **异步：**在 **_新的线程_** 中执行任务，**_具备_** 开启新线程的能力。

#### (2) 2 个队列类型：

> **并发队列（Concurrent Dispatch Queue）：**
> 
> *   让多个任务并发（同时）执行（自动开启多个线程同时执行任务）；
> *   并发功能只有在异步（`dispatch_async`）函数下才有效。
> 
> **串行队列（Serial Dispatch Queue）：**
> 
> *   让任务一个接着一个地执行（一个任务执行完毕后，再执行下一个任务）。

#### (3) 各种队列的执行效果

![](http://upload-images.jianshu.io/upload_images/1319505-387b7875bf400850.png)

各种队列的执行效果

总结可知：

> *   **同步函数或者主队列：没有开启新线程、串行执行任务；**
> *   **异步函数且非主队列：有开启新线程、执行方式取决与队列类型并发或串行**

* * *

### 3\. 面试题分析

#### (1) Q: 以下代码会不会产生死锁？

```
\- (void)viewDidLoad {
    \[super viewDidLoad\];
    
    NSLog(@"执行任务 - 1");

    
    dispatch\_queue\_t queue = dispatch\_get\_main\_queue();
    
    dispatch\_sync(queue, ^{
        
        NSLog(@"执行任务 - 2");
    });

    NSLog(@"执行任务 - 3");
}


Demo\[1234:567890\] 执行任务 - 1
(lldb)


```

根据程序做出分析图：

![](http://upload-images.jianshu.io/upload_images/1319505-73bd3e8829ae5716.png)

分析图

由分析图可知：  
主线程执行任务 1 之后，需要同步 (`dispatch_sync`) 执行任务 2；  
**而`dispatch_sync`要求立马在当前线程同步执行任务**；  
但是主队列中有`viewDidLoad`需要执行完毕才能执行任务 2；  
因此会产生死锁。

#### (2) Q: 以下代码会不会产生死锁？

```
\- (void)viewDidLoad {
    \[super viewDidLoad\];
    
    NSLog(@"执行任务 - 1");

    dispatch\_queue\_t queue = dispatch\_get\_main\_queue();
    
    dispatch\_async(queue, ^{
        NSLog(@"执行任务 - 2");
    });

    NSLog(@"执行任务 - 3");
}


Demo\[1234:567890\] 执行任务 - 1
Demo\[1234:567890\] 执行任务 - 3
Demo\[1234:567890\] 执行任务 - 2


```

根据程序做出分析图：

![](http://upload-images.jianshu.io/upload_images/1319505-3071ad686f39dfa3.png)

QQ20190423-102140@2x.png

由分析图可知：  
主线程执行任务 1 之后，需要异步 (`dispatch_async`) 执行任务 2；  
**而`dispatch_async`不要求立马在当前线程同步执行任务**；  
所以主线程接着执行任务 3，最后异步执行任务 2；  
因此不会产生死锁。

#### (3) Q: 以下代码会不会产生死锁？

```
\- (void)viewDidLoad {
    \[super viewDidLoad\];
    
    NSLog(@"执行任务 - 1");

    dispatch\_queue\_t queue = dispatch\_queue\_create("myQueue", DISPATCH\_QUEUE\_SERIAL);
    dispatch\_async(queue, ^{ 
        NSLog(@"执行任务 - 2");

        dispatch\_sync(queue, ^{ 
            NSLog(@"执行任务 - 3");
        });

        NSLog(@"执行任务 - 4");
    });

    NSLog(@"执行任务 - 5");
}


Demo\[1234:567890\] 执行任务 - 1
Demo\[1234:567890\] 执行任务 - 5
Demo\[1234:567890\] 执行任务 - 2
(lldb)


```

根据程序做出分析图：

![](http://upload-images.jianshu.io/upload_images/1319505-eee2b6d129adced1.png)

QQ20190423-102155@2x.png

由分析图可知：  
子线程执行任务 1 之后，需要异步 (`dispatch_async`) 执行任务 2；**而`dispatch_sync`要求立马在当前线程同步执行任务**；  
所以主线程接着执行任务 5；  
接着需要在串行队列中同步 (`dispatch_sync`) 执行任务 3；  
但是在串行队列中需要执行完任务 4 之后才能执行任务 3；  
因此会产生死锁。

#### (4) Q: 以下代码会不会产生死锁？

```
\- (void)viewDidLoad {
    \[super viewDidLoad\];
    
    NSLog(@"执行任务 - 1");

    dispatch\_queue\_t queue = dispatch\_queue\_create("myQueue", DISPATCH\_QUEUE\_SERIAL);
    dispatch\_queue\_t queue2 = dispatch\_queue\_create("myQueue2", DISPATCH\_QUEUE\_CONCURRENT);
    dispatch\_async(queue, ^{ 
        NSLog(@"执行任务 - 2");

        dispatch\_sync(queue2, ^{ 
            NSLog(@"执行任务 - 3");
        });

        NSLog(@"执行任务 - 4");
    });

    NSLog(@"执行任务 - 5");
}


Demo\[1234:567890\] 执行任务 - 1
Demo\[1234:567890\] 执行任务 - 5
Demo\[1234:567890\] 执行任务 - 2
Demo\[1234:567890\] 执行任务 - 3
Demo\[1234:567890\] 执行任务 - 4


```

根据程序做出分析图：

![](http://upload-images.jianshu.io/upload_images/1319505-b689e7c5eeb06417.png)

QQ20190423-102210@2x.png

由分析图可知：  
子线程执行任务 1 之后，需要异步 (`dispatch_async`) 执行任务 2；  
所以先执行主线程的任务 5，然后执行任务 2；  
接着需要在并发队列中同步 (`dispatch_sync`) 执行任务 3；  
然后执行串行队列中的任务 4；  
因此不会产生死锁。

#### (5) Q: 以下代码会不会产生死锁？

```
\- (void)viewDidLoad {
    \[super viewDidLoad\];
    
    NSLog(@"执行任务 - 1");

    dispatch\_queue\_t queue = dispatch\_queue\_create("myQueue", DISPATCH\_QUEUE\_SERIAL);
    dispatch\_queue\_t queue2 = dispatch\_queue\_create("myQueue2", DISPATCH\_QUEUE\_SERIAL);
    dispatch\_async(queue, ^{ 
        NSLog(@"执行任务 - 2");

        dispatch\_sync(queue2, ^{ 
            NSLog(@"执行任务 - 3");
        });

        NSLog(@"执行任务 - 4");
    });

    NSLog(@"执行任务 - 5");
}


Demo\[1234:567890\] 执行任务 - 1
Demo\[1234:567890\] 执行任务 - 5
Demo\[1234:567890\] 执行任务 - 2
Demo\[1234:567890\] 执行任务 - 3
Demo\[1234:567890\] 执行任务 - 4


```

根据程序做出分析图：

![](http://upload-images.jianshu.io/upload_images/1319505-f54bef29c95feb20.png)

QQ20190423-102226@2x.png

由分析图可知：  
子线程执行任务 1 之后，需要异步 (`dispatch_async`) 执行任务 2；  
所以先执行主线程的任务 5，然后执行任务 2；  
接着需要在串行队列中同步 (`dispatch_sync`) 执行任务 3；  
然后执行串行队列中的任务 4；  
因此不会产生死锁。

##### 6> Q: 以下代码会不会产生死锁？

```
\- (void)viewDidLoad {
    \[super viewDidLoad\];
    
    NSLog(@"执行任务 - 1");

    dispatch\_queue\_t queue = dispatch\_queue\_create("myQueue", DISPATCH\_QUEUE\_CONCURRENT);
    dispatch\_async(queue, ^{ 
        NSLog(@"执行任务 - 2");

        dispatch\_sync(queue, ^{ 
            NSLog(@"执行任务 - 3");
        });

        NSLog(@"执行任务 - 4");
    });

    NSLog(@"执行任务 - 5");
}


Demo\[1234:567890\] 执行任务 - 1
Demo\[1234:567890\] 执行任务 - 5
Demo\[1234:567890\] 执行任务 - 2
Demo\[1234:567890\] 执行任务 - 3
Demo\[1234:567890\] 执行任务 - 4


```

根据程序做出分析图：

![](http://upload-images.jianshu.io/upload_images/1319505-cb38ffa999a84494.png)

QQ20190423-102237@2x.png

由分析图可知：  
子线程执行任务 1 之后，需要异步 (`dispatch_async`) 执行任务 2；  
所以先执行主线程的任务 5，然后执行任务 2；  
接着需要在并发队列中同步 (`dispatch_sync`) 执行任务 3；  
然后执行并发队列中的任务 4；  
因此不会产生死锁。

#### (6) 总结可知：

> **使用 _`dispatch_sync`函数_ 往 _当前串行队列_ 中添加任务，会卡住当前的串行队列（产生死锁）。**

"小礼物走一走，来简书关注我"

还没有人赞赏，支持一下

[![](https://upload.jianshu.io/users/upload_avatars/1319505/9754fac0eb59?imageMogr2/auto-orient/strip|imageView2/1/w/100/h/100/format/webp)](https://www.jianshu.com/u/40dce98b3416)

总资产 1 (约 0.16 元) 共写了 4.8W 字获得 24 个赞共 23 个粉丝

### 被以下专题收入，发现更多相似内容

### 推荐阅读[更多精彩内容](https://www.jianshu.com/)

*   面试题引发的思考： Q: RunLoop 内部实现逻辑？ Q: RunLoop 和线程的关系？ 每条线程都有唯一的一个...
    
    [![](https://upload.jianshu.io/users/upload_avatars/1319505/9754fac0eb59?imageMogr2/auto-orient/strip|imageView2/1/w/48/h/48/format/webp)阡陌紫](https://www.jianshu.com/u/40dce98b3416)阅读 0
    
    [![](https://upload-images.jianshu.io/upload_images/3959864-5a91fff299f8206d?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)](https://www.jianshu.com/p/1ab04bf5e3c3)
*   背景——我公司 2.2 举办室内拓展。 第一个项目：三国争霸。 游戏规则：把公司全员随机分成魏蜀吴三国国民，且每一国内...
    
    [![](https://upload-images.jianshu.io/upload_images/1980214-955709976dad4947.jpg?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)](https://www.jianshu.com/p/01cc33b1a5ac)
*   简书搜索【肆 HAO 看福彩】点击关注，随时更新！ 8 码复试（01235678） 7 码复试（0123469） 4 码复试...
    
    [![](https://upload-images.jianshu.io/upload_images/8954490-470cddaf1e9ef8f5.jpg?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)](https://www.jianshu.com/p/fdf3f215f036)