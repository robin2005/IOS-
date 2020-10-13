\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[www.jianshu.com\](https://www.jianshu.com/p/03a69961bd35)

[![](https://upload.jianshu.io/users/upload_avatars/2202264/3097ab5a995b.jpg?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96/format/webp)](https://www.jianshu.com/u/752f38fc08ca)

32019.05.24 18:15:16 字数 842 阅读 995

最近学习戴铭大神的课程，其中一篇文章介绍了如何利用 RunLoop 监测卡顿，在此做个记录。

一、监测卡顿的原理
---------

文中介绍到：

> RunLoop 是用来监听输入源，进行调度处理的。如果 RunLoop 的线程进入睡眠前方法的执行时间过长而导致无法进入睡眠，或者线程唤醒后接收消息时间过长而无法进入下一步，就可以认为是线程受阻了。如果这个线程是主线程的话，表现出来的就是出现了卡顿。

RunLoop 有以下几种状态：

```
typedef CF\_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry , 
    kCFRunLoopBeforeTimers , 
    kCFRunLoopBeforeSources , 
    kCFRunLoopBeforeWaiting , 
    kCFRunLoopAfterWaiting , 
    kCFRunLoopExit , 
    kCFRunLoopAllActivities  
}


```

而文中提到的 2 种线程受阻情况，它们的状态分别是：kCFRunLoopBeforeSources 和 kCFRunLoopAfterWaiting。

二、思路
----

根据原理，可以得到一个监测卡顿的思路：

监测主线程 RunLoop 的状态，如果状态在一定时长内都是`kCFRunLoopBeforeSources`或者`kCFRunLoopAfterWaiting`，则认为卡顿。

步骤如下：

1.  创建一个 RunLoop 的观察者（CFRunLoopObserverRef）
2.  把观察者加入主线程的 kCFRunLoopCommonModes 模式中，以监测主线程
3.  创建一个子线程来维护观察者
4.  根据主线程 RunLoop 的状态来判断是否卡顿

三、实现方法
------

#### 1\. 实现代码

```
@interface AppDelegate ()
{
    int timeoutCount;
    CFRunLoopObserverRef runLoopObserver;
@public
    dispatch\_semaphore\_t dispatchSemaphore;
    CFRunLoopActivity runLoopActivity;
}
@end

@implementation AppDelegate


- (void)beginMonitor {

    if (runLoopObserver) {
        return;
    }
    
    
    
    dispatchSemaphore = dispatch\_semaphore\_create(0); 
    
    
    CFRunLoopObserverContext context = {0,(\_\_bridge void\*)self,NULL,NULL};
    runLoopObserver = CFRunLoopObserverCreate(kCFAllocatorDefault,
                                              kCFRunLoopAllActivities,
                                              YES,
                                              0,
                                              &runLoopObserverCallBack,
                                              &context);
    
    CFRunLoopAddObserver(CFRunLoopGetMain(), runLoopObserver, kCFRunLoopCommonModes);
    
    
    dispatch\_async(dispatch\_get\_global\_queue(0, 0), ^{
        
        while (1) {
            
            
            long semaphoreWait = dispatch\_semaphore\_wait(self->dispatchSemaphore, dispatch\_time(DISPATCH\_TIME\_NOW, 20\*NSEC\_PER\_MSEC));

            if (semaphoreWait != 0) {
                if (!self->runLoopObserver) {
                    self->timeoutCount = 0;
                    self->dispatchSemaphore = 0;
                    self->runLoopActivity = 0;
                    return;
                }
                
                
                if (self->runLoopActivity == kCFRunLoopBeforeSources || self->runLoopActivity == kCFRunLoopAfterWaiting) {
                    NSLog(@"runloop状态：%@",@(self->runLoopActivity));
                    
                    if (++self->timeoutCount < 3) {
                        continue;
                    }
                    NSLog(@"发生卡顿...");
                }
            }
            self->timeoutCount = 0;
        }
    });
    
}

static void runLoopObserverCallBack(CFRunLoopObserverRef observer, CFRunLoopActivity activity, void \*info) {

    AppDelegate \*delegate = (\_\_bridge AppDelegate\*)info;
    delegate->runLoopActivity = activity;
    
    dispatch\_semaphore\_t semaphore = delegate->dispatchSemaphore;
    
    dispatch\_semaphore\_signal(semaphore);
}

- (BOOL)application:(UIApplication \*)application didFinishLaunchingWithOptions:(NSDictionary \*)launchOptions {
    
    \[self beginMonitor\];
    
    return YES;
}

@end



```

监测到卡顿后，使用 [PLCrashRepoter](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.plcrashreporter.org%2Fdownload%2F) 获取堆栈信息，根据这些信息找到造成卡顿的方法。

```
#import <CrashReporter/CrashReporter.h>


NSData \*lagData = \[\[\[PLCrashReporter alloc\]
                    initWithConfiguration:\[\[PLCrashReporterConfig alloc\] initWithSignalHandlerType:PLCrashReporterSignalHandlerTypeBSD symbolicationStrategy:PLCrashReporterSymbolicationStrategyAll\]\] generateLiveReport\];

PLCrashReport \*lagReport = \[\[PLCrashReport alloc\] initWithData:lagData error:NULL\];

NSString \*lagReportString = \[PLCrashReportTextFormatter stringValueForCrashReport:lagReport withTextFormat:PLCrashReportTextFormatiOS\];

NSLog(@"lag happen, detail below: \\n %@",lagReportString);


```

#### 2\. 流程说明：

为了保证子线程的同步监测，刚开始创建一个信号量是 0 的`dispatch_semaphore`。当监测到主线程的 RunLoop 的状态发生变化，触发回调：

```
static void runLoopObserverCallBack(CFRunLoopObserverRef observer, CFRunLoopActivity activity, void \*info) {


```

在回调里面发送信号，使信号量 + 1，值变为 1：

```
dispatch\_semaphore\_signal(semaphore)


```

当`dispatch_semaphore_wait`接收到信号量不为 0，会返回一个不为 0 的值（semaphoreWait），并把信号量减 1：

```
long semaphoreWait = dispatch\_semaphore\_wait(self->dispatchSemaphore, dispatch\_time(DISPATCH\_TIME\_NOW, 20\*NSEC\_PER\_MSEC));


```

然后触发一次监测功能，记录 RunLoop 状态。此时信号量是 0，继续等待下一个信号量。

反复监测后，如果状态连续 3 次都是`kCFRunLoopBeforeSources`或者`kCFRunLoopAfterWaiting`，则判定为卡顿。

#### 3\. 关于触发卡顿的时间阈值

代码中定义了卡顿阈值是 3\*20ms=60ms，在线下做测试的话，这个值是合理的。但是如果在线上使用这种监测卡顿的方法，大神建议把该值设为`3秒`，文中介绍如下：

> 其实，触发卡顿的时间阈值，我们可以根据 WatchDog 机制来设置。WatchDog 在不同状态下设置的不同时间，如下所示：

> 启动（Launch）：20s；  
> 恢复（Resume）：10s；  
> 挂起（Suspend）：10s；  
> 退出（Quit）：6s；  
> 后台（Background）：3min（在 iOS 7 之前，每次申请 10min； 之后改为每次申请 3min，可连续申请，最多申请到 10min）。

> 通过 WatchDog 设置的时间，我认为可以把启动的阈值设置为 10 秒，其他状态则都默认设置为 3 秒。总的原则就是，要小于 WatchDog 的限制时间。当然了，这个阈值也不用小得太多，原则就是要优先解决用户感知最明显的体验问题。

参考项目：[https://github.com/ming1016/DecoupleDemo/blob/master/DecoupleDemo/SMLagMonitor.m](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fming1016%2FDecoupleDemo%2Fblob%2Fmaster%2FDecoupleDemo%2FSMLagMonitor.m)

"小礼物走一走，来简书关注我"

还没有人赞赏，支持一下

[![](https://upload.jianshu.io/users/upload_avatars/2202264/3097ab5a995b.jpg?imageMogr2/auto-orient/strip|imageView2/1/w/100/h/100/format/webp)](https://www.jianshu.com/u/752f38fc08ca)

总资产 8 (约 0.66 元) 共写了 3152 字获得 96 个赞共 47 个粉丝

### 被以下专题收入，发现更多相似内容

### 推荐阅读[更多精彩内容](https://www.jianshu.com/)

*   深入理解 RunLoop 由 ibireme| 2015-05-18 |iOS, 技术 RunLoop 是 iOS 和 ...
    
    [![](https://cdn2.jianshu.io/assets/default_avatar/13-394c31a9cb492fcb39c27422ca7d2815.jpg)橙娃](https://www.jianshu.com/u/cb4db3ed5740)阅读 4
    
    [![](https://upload-images.jianshu.io/upload_images/3362328-1e574709ba42cc76.png?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)](https://www.jianshu.com/p/11c4c47a93af)