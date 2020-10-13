\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[www.jianshu.com\](https://www.jianshu.com/p/b44a32817bd8)

[![](https://upload.jianshu.io/users/upload_avatars/1319505/9754fac0eb59?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96/format/webp)](https://www.jianshu.com/u/40dce98b3416)

2018.10.12 13:54:36 字数 498 阅读 112

#### 面试题引发的思考：

**Q: RunLoop，项目中有用到吗？**

*   **解决`NSTimer`失效问题；**
*   **控制线程生命周期（线程保活）；**
*   **监控应用卡顿；**
*   **性能优化。**

**Q: 程序中添加每 3 秒响应一次的`NSTimer`，当拖动`tableview`时`timer`可能无法响应要怎么解决？**

*   **通过`[[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];`语句添加`NSTimer`即可。**
*   **在通用模式下，切换模式`NSTimer`也不会停止运行。**

* * *

### 1\. 解决`NSTimer`失效问题

#### (1) 什么情况下出现`NSTimer`失效问题？

在`ViewController`中加入`UITextView`，然后运行以下代码：

```
@implementation ViewController
- (void)viewDidLoad {
    \[super viewDidLoad\];
    
    static int count = 0;
    \[NSTimer scheduledTimerWithTimeInterval:1.0 repeats:YES block:^(NSTimer \* \_Nonnull timer) {
        NSLog(@"%.2d - %@", ++count, CFRunLoopCopyCurrentMode(CFRunLoopGetCurrent()));
    }\];
}
@end


```

滑动`UITextView`，然后查看打印结果：

![](http://upload-images.jianshu.io/upload_images/1319505-b6b0e602c67ce0c7.png)

打印结果

由打印结果可知：  
滑动`UITextView`时，定时器停止工作，原因如下：

*   RunLoop 同一时间只能运行在某一种模式；
*   定时器是在`kCFRunLoopDefaultMode`模式下工作的；
*   拖拽`UITextView`时，RunLoop 会切换到`UITrackingRunLoopMode`模式，定时器会停止工作。

#### (2) 如何解决`NSTimer`失效问题

在`ViewController`中加入`UITextView`，然后运行以下代码：

```
@implementation ViewController
- (void)viewDidLoad {
    \[super viewDidLoad\];
    
    static int count = 0;
    NSTimer \*timer = \[NSTimer timerWithTimeInterval:1.0 repeats:YES block:^(NSTimer \* \_Nonnull timer) {
        NSLog(@"%.2d - %@", ++count, CFRunLoopCopyCurrentMode(CFRunLoopGetCurrent()));
    }\];
    
    
    \[\[NSRunLoop currentRunLoop\] addTimer:timer forMode:NSRunLoopCommonModes\];
}
@end


```

滑动`UITextView`，然后查看打印结果：

![](http://upload-images.jianshu.io/upload_images/1319505-fa5f4ae65499f39e.png)

打印结果

由打印结果可知：  
滑动`UITextView`时，RunLoop 会切换到`UITrackingRunLoopMode`模式，定时器会继续工作；可以通过以上方法解决`NSTimer`失效问题。

* * *

### 2\. 控制线程生命周期（线程保活）

#### (1) 线程默认使用情况

```
@interface MYThread : NSThread
@end

@implementation MYThread
- (void)dealloc {
    NSLog(@"%s", \_\_func\_\_);
}
@end


@implementation ViewController
- (void)viewDidLoad {
    \[super viewDidLoad\];
    
    MYThread \*thread = \[\[MYThread alloc\] initWithTarget:self selector:@selector(run) object:nil\];
    \[thread start\];
}


- (void)run {
    NSLog(@"%s %@", \_\_func\_\_, \[NSThread currentThread\]);
}
@end


Demo\[1234:567890\] -\[ViewController run\] <MYThread: 0x600000500940>{number = 6, name = (null)}
Demo\[1234:567890\] -\[MYThread dealloc\]


```

由打印结果可知：  
默认调用完`run`方法后，子线程马上就销毁了；  
如果需要经常在子线程进行操作，应该如何处理呢？

#### 2) 如何控制线程生命周期？

```
typedef void (^MYPermenantThreadTask)(void);

@interface MYPermenantThread : NSObject

- (void)run;

- (void)executeTask:(MYPermenantThreadTask)task;

- (void)stop;
@end

@interface MYPermenantThread ()
@property (nonatomic, strong) NSThread \*innerThread;
@property (nonatomic, assign) BOOL isStopped;
@end

@implementation MYPermenantThread
#pragma mark -
#pragma mark - public methods
- (instancetype)init {
    self = \[super init\];
    if (self) {
        self.isStopped = NO;
        \_\_weak typeof(self) weakSelf = self;
        self.innerThread = \[\[NSThread alloc\] initWithBlock:^{
            \[\[NSRunLoop currentRunLoop\] addPort:\[\[NSPort alloc\] init\] forMode:NSDefaultRunLoopMode\];

            while (weakSelf && !weakSelf.isStopped) {
                \[\[NSRunLoop currentRunLoop\] runMode:NSDefaultRunLoopMode beforeDate:\[NSDate distantFuture\]\];
            }
        }\];
    }
    return self;
}
- (void)run {
    if (!self.innerThread) return;
    \[self.innerThread start\];
}
- (void)executeTask:(MYPermenantThreadTask)task {
    if (!self.innerThread || !task) return;
    \[self performSelector:@selector(\_\_executeTask:) onThread:self.innerThread withObject:task waitUntilDone:NO\];
}
- (void)stop {
    if (!self.innerThread) return;
    \[self performSelector:@selector(\_\_stop) onThread:self.innerThread withObject:nil waitUntilDone:YES\];
}
- (void)dealloc {
    NSLog(@"%s", \_\_func\_\_);
    
    \[self stop\];
}
#pragma mark -
#pragma mark - private methods
- (void)\_\_stop {
    self.isStopped = YES;
    CFRunLoopStop(CFRunLoopGetCurrent());
    self.innerThread = nil;
}
- (void)\_\_executeTask:(MYPermenantThreadTask)task {
    task();
}
@end


@interface ViewController ()
@property (nonatomic, strong) MYPermenantThread \*thread;
@end

@implementation ViewController
- (void)viewDidLoad {
    \[super viewDidLoad\];
    
    self.thread = \[\[MYPermenantThread alloc\] init\];
    \[self.thread run\];
}
- (void)touchesBegan:(NSSet<UITouch \*> \*)touches withEvent:(UIEvent \*)event {
    
    \[self.thread executeTask:^{
        NSLog(@"执行任务 - %@", \[NSThread currentThread\]);
    }\];
}
- (void)dealloc {
    NSLog(@"%s", \_\_func\_\_);
}
@end


Demo\[1234:567890\] 执行 -\[ViewController run\] <NSThread: 0x600001374900>{number = 6, name = (null)}
Demo\[1234:567890\] 执行 -\[ViewController run\] <NSThread: 0x600001374900>{number = 6, name = (null)}
Demo\[1234:567890\] -\[ViewController dealloc\]
Demo\[1234:567890\] -\[MYPermenantThread dealloc\]


```

由打印结果可知：  
封装一个`MYPermenantThread`类，用来控制子线程的开启、结束与任务执行，然后在`ViewController`类中，可以实现线程的开启、结束与任务执行。

* * *

### 3\. 监控应用卡顿

* * *

### 4\. 性能优化

"小礼物走一走，来简书关注我"

还没有人赞赏，支持一下

[![](https://upload.jianshu.io/users/upload_avatars/1319505/9754fac0eb59?imageMogr2/auto-orient/strip|imageView2/1/w/100/h/100/format/webp)](https://www.jianshu.com/u/40dce98b3416)

总资产 1 (约 0.16 元) 共写了 4.8W 字获得 24 个赞共 23 个粉丝

### 被以下专题收入，发现更多相似内容

### 推荐阅读[更多精彩内容](https://www.jianshu.com/)

*   今天 BitUN APP 版本更新，新增支持了 EOS、DASH 币种。截止现在，BitUN 已支持 BUC、BTC、ETH、...
    
    [![](https://upload-images.jianshu.io/upload_images/11264127-bd797ed0e9a4ba5e?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)](https://www.jianshu.com/p/683a668968d3)
*   GE 矩阵简介 GE 行业吸引力矩阵的开发初衷，是帮助市场营销人弥补波士顿矩阵的一系列缺陷。事实就是如此，波士顿矩阵重...
    
    [![](https://upload.jianshu.io/users/upload_avatars/2666028/68938c70-35e1-4cdd-8773-83cf2361ab52.jpeg?imageMogr2/auto-orient/strip|imageView2/1/w/48/h/48/format/webp)你的部长](https://www.jianshu.com/u/148639a31daa)阅读 14
    
*   一年一度的高考开始了，像往年一样，宝爸又去异地监考去了。一人牵动百家心，苦读寒窗为理真。莘莘学子经过 12 年的寒窗苦...