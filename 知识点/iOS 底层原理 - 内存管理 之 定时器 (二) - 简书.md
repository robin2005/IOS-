\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[www.jianshu.com\](https://www.jianshu.com/p/d61aa0c87b4b)

[![](https://upload.jianshu.io/users/upload_avatars/1319505/9754fac0eb59?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96/format/webp)](https://www.jianshu.com/u/40dce98b3416)

0.0962019.04.22 19:03:20 字数 155 阅读 46

#### 面试题引发的思考：

**Q: 使用`CADisplayLink`、`NSTimer`是否准时？**

*   **`CADisplayLink`、`NSTimer`依赖于 RunLoop，而事件响应、手势识别、界面刷新等也是依赖于 RunLoop 实现的。**
*   **如果 RunLoop 的任务过于繁重，可能会导致`CADisplayLink`、`NSTimer`不准时。**

**Q: 如何解决`CADisplayLink`、`NSTimer`不准时的问题呢？**

*   **使用 GCD 的定时器会更加准时。**

* * *

### 1\. GCD 定时器使用：

```
@interface ViewController ()

@property (nonatomic, strong) dispatch\_source\_t timer;
@end

@implementation ViewController
- (void)viewDidLoad {
    \[super viewDidLoad\];

    
    dispatch\_queue\_t queue = dispatch\_get\_main\_queue();
    
    

    
    dispatch\_source\_t timer = dispatch\_source\_create(DISPATCH\_SOURCE\_TYPE\_TIMER, 0, 0, queue);

    
    dispatch\_source\_set\_timer(timer, DISPATCH\_TIME\_NOW, 1.0 \* NSEC\_PER\_SEC, 0 \* NSEC\_PER\_SEC);

    
    dispatch\_source\_set\_event\_handler(timer, ^{
        NSLog(@"timer - %@", \[NSThread currentThread\]);
    });

    
    dispatch\_resume(timer);

    self.timer = timer;
}
@end


```

* * *

### 2\. GCD 定时器封装：

下面对 GCD 定时器进行封装，以适用大多数场景：

```
@interface MYTimer : NSObject







+ (NSString \*)executeTask:(void(^)(void))task
                    start:(NSTimeInterval)start
                 interval:(NSTimeInterval)interval
                  repeats:(BOOL)repeats
                    async:(BOOL)async;








+ (NSString \*)executeTask:(id)target
                 selector:(SEL)selector
                    start:(NSTimeInterval)start
                 interval:(NSTimeInterval)interval
                  repeats:(BOOL)repeats
                    async:(BOOL)async;



+ (void)cancelTask:(NSString \*)name;

@end


@implementation MYTimer


static NSMutableDictionary \*timers\_;

dispatch\_semaphore\_t semaphore\_;


+ (void)initialize {
    static dispatch\_once\_t onceToken;
    dispatch\_once(&onceToken, ^{
        timers\_ = \[NSMutableDictionary dictionary\];
        
        semaphore\_ = dispatch\_semaphore\_create(1);
    });
}

+ (NSString \*)executeTask:(void (^)(void))task
                    start:(NSTimeInterval)start
                 interval:(NSTimeInterval)interval
                  repeats:(BOOL)repeats
                    async:(BOOL)async
{
    if (!task || start < 0 || (interval <= 0 && repeats)) return nil;
    
    
    dispatch\_queue\_t queue = async ? dispatch\_get\_global\_queue(0, 0) : dispatch\_get\_main\_queue();

    
    dispatch\_source\_t timer = dispatch\_source\_create(DISPATCH\_SOURCE\_TYPE\_TIMER, 0, 0, queue);

    
    dispatch\_source\_set\_timer(timer, DISPATCH\_TIME\_NOW, interval \* NSEC\_PER\_SEC, start \* NSEC\_PER\_SEC);

    
    dispatch\_semaphore\_wait(semaphore\_, DISPATCH\_TIME\_FOREVER);

    
    NSString \*name = \[NSString stringWithFormat:@"%zd", timers\_.count\];
    
    timers\_\[name\] = timer;

    
    dispatch\_semaphore\_signal(semaphore\_);

    
    dispatch\_source\_set\_event\_handler(timer, ^{
        
        task();

        if (!repeats) {
            
            \[self cancelTask:name\];
        }
    });
    
    
    dispatch\_resume(timer);
    
    return name;
}

+ (NSString \*)executeTask:(id)target
                 selector:(SEL)selector
                    start:(NSTimeInterval)start
                 interval:(NSTimeInterval)interval
                  repeats:(BOOL)repeats
                    async:(BOOL)async
{
    if (!target || !selector) return nil;
    
    return \[self executeTask:^{
        if (\[target respondsToSelector:selector\]) {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Warc-performSelector-leaks"
            \[target performSelector:selector\];
#pragma clang diagnostic pop
        }
    } start:start interval:interval repeats:repeats async:async\];
}

+ (void)cancelTask:(NSString \*)name {
    if (name.length == 0) return;

    
    dispatch\_semaphore\_wait(semaphore\_, DISPATCH\_TIME\_FOREVER);
    
    dispatch\_source\_t timer = timers\_\[name\];
    if (timer) {
        dispatch\_source\_cancel(timer);
        \[timers\_ removeObjectForKey:name\];
    }
    
    
    dispatch\_semaphore\_signal(semaphore\_);
}
@end


```

那么`MYTimer`调用方法如下：

```
@interface ViewController ()
@property (nonatomic, copy) NSString \*task;
@end

@implementation ViewController

- (void)viewDidLoad {
    \[super viewDidLoad\];
    
    self.task = \[MYTimer executeTask:^{
        NSLog(@"timer - %@", \[NSThread currentThread\]);
    } start:2.0 interval:1.0 repeats:YES async:YES\];
}

- (void)touchesBegan:(NSSet<UITouch \*> \*)touches withEvent:(UIEvent \*)event {
    \[MYTimer cancelTask:self.task\];
}
@end


```

"小礼物走一走，来简书关注我"

还没有人赞赏，支持一下

[![](https://upload.jianshu.io/users/upload_avatars/1319505/9754fac0eb59?imageMogr2/auto-orient/strip|imageView2/1/w/100/h/100/format/webp)](https://www.jianshu.com/u/40dce98b3416)

总资产 1 (约 0.16 元) 共写了 4.8W 字获得 24 个赞共 23 个粉丝

### 被以下专题收入，发现更多相似内容

### 推荐阅读[更多精彩内容](https://www.jianshu.com/)

*   面试题引发的思考： Q: 修饰词 atomic 和 nonatomic 修饰属性的区别？ atomic 用于保证属性 sett...
    
    [![](https://upload.jianshu.io/users/upload_avatars/1319505/9754fac0eb59?imageMogr2/auto-orient/strip|imageView2/1/w/48/h/48/format/webp)阡陌紫](https://www.jianshu.com/u/40dce98b3416)阅读 7
    
    [![](https://upload-images.jianshu.io/upload_images/970779-0f92d861382cd707.jpg?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)](https://www.jianshu.com/p/02f438df17ef)
*   Ajax 介绍 Ajax 最早产生于 2005 年，Ajax 表示 Asynchronous JavaScript and X...
    
    [![](https://upload-images.jianshu.io/upload_images/1696952-578bab34a9a805de.png?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)](https://www.jianshu.com/p/46fd608c58dc)
*   Swift1> Swift 和 OC 的区别 1.1> Swift 没有地址 / 指针的概念 1.2> 泛型 1.3> 类型严谨 对...