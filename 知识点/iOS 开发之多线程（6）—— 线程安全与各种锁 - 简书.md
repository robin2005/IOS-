\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[www.jianshu.com\](https://www.jianshu.com/p/9c0b73c9f752)

[![](https://upload.jianshu.io/users/upload_avatars/8900795/90987efc-b432-4f85-90cb-51aa3bc07f5a?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96/format/webp)](https://www.jianshu.com/u/83ecdf2d05e0)

0.1972020.07.15 17:37:49 字数 2,791 阅读 148

[iOS 开发之多线程（1）—— 概述](https://www.jianshu.com/p/169a808e5d82)  
[iOS 开发之多线程（2）—— Thread](https://www.jianshu.com/p/8fc27a1ad521)  
[iOS 开发之多线程（3）—— GCD](https://www.jianshu.com/p/9002c1f1e9e3)  
[iOS 开发之多线程（4）—— Operation](https://www.jianshu.com/p/9e51a7bba654)  
[iOS 开发之多线程（5）—— Pthreads](https://www.jianshu.com/p/dbdb34ebd192)  
[iOS 开发之多线程（6）—— 线程安全与各种锁](https://www.jianshu.com/p/9c0b73c9f752)

1.  线程安全  
    1.1 线程不安全示例  
    1.2 线程安全  
    1.3 互斥
2.  锁  
    dispatch\_semaphore 信号量  
    OSSpinLock 自旋锁  
    pthread\_mutex 互斥锁  
    pthread\_mutex (RECURSIVE) 递归锁  
    pthread\_mutex + pthread\_cond 条件锁  
    NSLock 互斥锁  
    NSRecursiveLock 递归锁  
    NSCondition 条件锁  
    NSConditionLock 条件锁  
    @synchronized

### 1.1 线程不安全示例

线程可与其同属一个进程的其他的线程共享进程所拥有的全部资源, 由于线程间是可以并发执行的, 这就可能导致多个线程同时访问 (读或写) 同一内存地址 (即[数据竞赛](https://links.jianshu.com/go?to=https%3A%2F%2Fen.wikipedia.org%2Fwiki%2FRace_condition%23Data_race)), 从而导致一些不可预计的结果.  
**示例**  
模拟买票系统, 没有使用锁, 是线程不安全的:

```
@property (nonatomic, assign)   int tickets;

- (void)initData {
    
    self.tickets = 10;
}

- (void)sample {
    
    dispatch\_async(dispatch\_get\_global\_queue(0, 0), ^{
        while (self.tickets) {
            \[self sellTickets\];
        }
        NSLog(@"窗口A, 票卖完了");
    });
    
    dispatch\_async(dispatch\_get\_global\_queue(0, 0), ^{
        while (self.tickets) {
            \[self sellTickets\];
        }
        NSLog(@"窗口B, 票卖完了");
    });
}


- (void)sellTickets {
            
    if (self.tickets) {
        sleep(0.2);         
        self.tickets--;
        NSLog(@"出票一张, 余票:%d", self.tickets);
    }
}


log:
出票一张, 余票:9
出票一张, 余票:9
出票一张, 余票:8
出票一张, 余票:8
出票一张, 余票:7
出票一张, 余票:7
出票一张, 余票:6
出票一张, 余票:5
出票一张, 余票:4
出票一张, 余票:3
出票一张, 余票:2
出票一张, 余票:1
出票一张, 余票:0
窗口A, 票卖完了
窗口B, 票卖完了


```

> 反复运行, 很容易出现问题. 结果显然不是我们想要的.  
> 另需注意, 这里没有在窗口 A 或者 B 中打印余票, 是因为某个窗口将要打印时, 有可能被另一窗口卖掉一张从而影响余票. 正确做法是在 sellTickets 里实时监听票数然后通知窗口 A 和 B 来刷新显示, 这里不做演示.

这里只列举一个例子, 类似这样的案例很多, 但归根还是因多个线程同时访问同一内存地址导致的不可预计结果.  
为了解决这类问题, 我们需要引入线程安全机制.

### 1.2 线程安全

> 线程安全是一种适用于多线程代码的计算机编程概念。线程安全代码仅以确保所有线程正常运行并满足其设计规范的方式操作共享数据结构，而不会发生意外交互。

建立线程安全数据结构的策略多种多样, 可分为两类方法:

第一类方法着重于避免共享状态, 包括:

1.  [重入](https://links.jianshu.com/go?to=https%3A%2F%2Fen.wikipedia.org%2Fwiki%2FReentrant_%28subroutine%29)
2.  [线程本地存储](https://links.jianshu.com/go?to=https%3A%2F%2Fen.wikipedia.org%2Fwiki%2FThread-local_storage)
3.  [不变的对象](https://links.jianshu.com/go?to=https%3A%2F%2Fen.wikipedia.org%2Fwiki%2FImmutable_object)

第二类方法与同步相关, 并且在无法避免共享状态的情况下使用:

1.  [互斥](https://links.jianshu.com/go?to=https%3A%2F%2Fen.wikipedia.org%2Fwiki%2FMutual_exclusion)
2.  [原子操作](https://links.jianshu.com/go?to=https%3A%2F%2Fen.wikipedia.org%2Fwiki%2FLinearizability)

第一类方法着重于避免数据共享, 而我们要讨论的是如何在多线程间安全地进行数据共享, 故此类方法在这里不做讨论.  
第二类方法着重于同步机制 (线程同步和数据同步). 第一种互斥将在下文 1.3 小节单独介绍; 而第二种原子操作 (原子操作 atomic 是不可分割的操作, 在原子操作执行完毕之前, 其不会被任何其它任务或事件中断). atomic 在 iOS 系统中比较耗费资源从而基本不用, 它比较适用于 macOS 中. 而且, 苹果开发文档已经明确指出: _Atomic 不能保证对象多线程的安全_. 它只是能保证你访问的时候给你返回一个完好无损的 Value 而已.

### 1.3 互斥

> wikipedia  
> 在[计算机科学中](https://links.jianshu.com/go?to=https%3A%2F%2Fen.wikipedia.org%2Fwiki%2FComputer_science)，**互斥**是[并发控制](https://links.jianshu.com/go?to=https%3A%2F%2Fen.wikipedia.org%2Fwiki%2FConcurrency_control)的属性，它是为了防止[竞争条件而建立的](https://links.jianshu.com/go?to=https%3A%2F%2Fen.wikipedia.org%2Fwiki%2FRace_condition)。它是要求，即一个[执行线程](https://links.jianshu.com/go?to=https%3A%2F%2Fen.wikipedia.org%2Fwiki%2FThread_%28computing%29)不会进入其[临界段](https://links.jianshu.com/go?to=https%3A%2F%2Fen.wikipedia.org%2Fwiki%2FCritical_section)在该另一同一时间[的并发](https://links.jianshu.com/go?to=https%3A%2F%2Fen.wikipedia.org%2Fwiki%2FConcurrent_computing)执行的线程进入其自己的关键部分，它是指一个时间间隔，在此期间执行的线程访问共享资源，如[共享内存](https://links.jianshu.com/go?to=https%3A%2F%2Fen.wikipedia.org%2Fwiki%2FShared_memory_%28interprocess_communication%29)。

互斥解决的问题是资源共享的问题, 防止在同一时间有两个 (或以上) 线程争夺同一资源. 互斥也可以认为是一种同步机制.  
互斥的解决方案也有很多种:

*   [锁](https://links.jianshu.com/go?to=https%3A%2F%2Fen.wikipedia.org%2Fwiki%2FLock_%28computer_science%29)
*   [读写器锁](https://links.jianshu.com/go?to=https%3A%2F%2Fen.wikipedia.org%2Fwiki%2FReaders%25E2%2580%2593writer_lock)
*   [递归锁](https://links.jianshu.com/go?to=https%3A%2F%2Fen.wikipedia.org%2Fwiki%2FReentrant_mutex)
*   [信号量](https://links.jianshu.com/go?to=https%3A%2F%2Fen.wikipedia.org%2Fwiki%2FSemaphore_%28programming%29)
*   [监控器](https://links.jianshu.com/go?to=https%3A%2F%2Fen.wikipedia.org%2Fwiki%2FMonitor_%28synchronization%29)
*   [讯息传递](https://links.jianshu.com/go?to=https%3A%2F%2Fen.wikipedia.org%2Fwiki%2FMessage_passing)
*   [元组空间](https://links.jianshu.com/go?to=https%3A%2F%2Fen.wikipedia.org%2Fwiki%2FTuple_space)

而我们要讨论的是在编码中常会用到的——锁.

锁是一种同步机制, 用于在存在许多执行线程的环境中强制限制对资源的访问.  
锁的分类五花八门, 各种名词层出不穷. 这里不打算花时间去做一个详细总结, 至少我不愿意那样做. 🤷‍♀️🤷‍♀️

加锁势必会带来额外的开销, 以下是 ibireme 对各种锁所做的性能测试, 具有参考价值 ([图片来源](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.ibireme.com%2F2016%2F01%2F16%2Fspinlock_is_unsafe_in_ios%2F)):  

![](http://upload-images.jianshu.io/upload_images/8900795-0b6aa2b11d633972.png)

lock\_benchmark.png

> OSSpinLock 耗时最短, 故性能最好 (可惜已不再安全); @synchronized 耗时最长, 性能最差.

为了便于讨论, 我们将介绍顺序重新整理如下:

*   `dispatch_semaphore` 信号量
*   `OSSpinLock` 自旋锁
*   `pthread_mutex` 互斥锁
*   `pthread_mutex (RECURSIVE)` 递归锁
*   `pthread_mutex + pthread_cond` 条件锁
*   `NSLock` 互斥锁 (封装 pthread\_mutex)
*   `NSRecursiveLock` 递归锁 (封装 pthread\_mutex (RECURSIVE))
*   `NSCondition` 条件锁 (封装 pthread\_mutex + pthread\_cond)
*   `NSConditionLock` 互斥锁 (封装 NSCondition + condition(NSInteger 类型))
*   `@synchronized`

### dispatch\_semaphore 信号量

> 如果信号量是一个任意的整数，通常被称为计数信号量（Counting semaphore），或一般信号量（general semaphore）；如果信号量只有二进制的 0 或 1，称为二进制信号量（binary semaphore）。在 linux 系统中，二进制信号量（binary semaphore）又称互斥锁（Mutex）

最简单的二进制信号量可以看做是一种互斥锁. 信号量可看成一种锁, 或者说是对锁的升级, 也是为了解决资源竞争问题.

信号量和互斥锁的区别是:

*   互斥锁: 提供对共享资源的互斥访问, 强调唯一性和排他性, 但是无法限制资源释放后其他线程申请的顺序问题. 比如说, 线程 A 正在占用资源, 同时线程 B 和线程 C 在等待者, 当 A 释放资源后, B 和 C 谁先抢得资源是随机的.
*   信号量: 在资源互斥的基础上, 实现了对线程的调度功能, 当然也保证了数据的同步. 比如上个例子, A 释放资源后, 可指定先分配给 B 或者 C.

GCD 中信号量只有三个方法:

```
dispatch\_semaphore\_t dispatch\_semaphore\_create(long value);

long dispatch\_semaphore\_wait(dispatch\_semaphore\_t dsema, dispatch\_time\_t timeout);

long dispatch\_semaphore\_signal(dispatch\_semaphore\_t dsema);


```

*   创建: 传入一个非负 long 值, 返回一个 dispatch\_semaphore\_t 类型信号量. 当前信号值即为传入的 long 值, 一般传 0. 如果传负数, 会返回 nil, 创建失败.
*   等待: 如果执行该操作将会对信号值减 1, 但是要求减之后信号值不能小于 0. 所以, 只有当前信号量大于等于 1 时, 才会执行该操作, 否则将一直等待, 直到 time out.
*   发送信号: 会使信号值加 1. 多次调用则多次加 1.

一般用法:

```
    
    dispatch\_semaphore\_t sem = dispatch\_semaphore\_create(0);
    
    dispatch\_async(dispatch\_get\_global\_queue(0, 0), ^{
        
        
        
        
        dispatch\_semaphore\_signal(sem);
    });
    
    
    dispatch\_semaphore\_wait(sem, DISPATCH\_TIME\_FOREVER);


```

**示例**

```
\- (void)semaphore {

    
    dispatch\_semaphore\_t sem = dispatch\_semaphore\_create(0);

    
    dispatch\_async(dispatch\_get\_global\_queue(0, 0), ^{
        NSLog(@"线程A");
        dispatch\_semaphore\_signal(sem);
    });
    dispatch\_semaphore\_wait(sem, DISPATCH\_TIME\_FOREVER);

    
    dispatch\_async(dispatch\_get\_global\_queue(0, 0), ^{
        NSLog(@"线程B");
        dispatch\_semaphore\_signal(sem);
    });
    dispatch\_semaphore\_wait(sem, DISPATCH\_TIME\_FOREVER);

    
    dispatch\_async(dispatch\_get\_global\_queue(0, 0), ^{
        NSLog(@"线程C");
        dispatch\_semaphore\_signal(sem);
    });
    dispatch\_semaphore\_wait(sem, DISPATCH\_TIME\_FOREVER);
}


log:
线程A
线程B
线程C


```

> 虽然都是异步执行, 但是由于使用信号量, 不管执行多少次, 结果都是按照线程 A->B->C 顺序执行的.

### OSSpinLock 自旋锁

OSSpinLock 属于自旋锁.  
它是为实现保护[共享资源](https://links.jianshu.com/go?to=https%3A%2F%2Fbaike.baidu.com%2Fitem%2F%25E5%2585%25B1%25E4%25BA%25AB%25E8%25B5%2584%25E6%25BA%2590)而提出一种锁机制。其实，自旋锁与[互斥锁](https://links.jianshu.com/go?to=https%3A%2F%2Fbaike.baidu.com%2Fitem%2F%25E4%25BA%2592%25E6%2596%25A5%25E9%2594%2581)比较类似，它们都是为了解决对某项资源的互斥使用。无论是[**互斥锁**](https://links.jianshu.com/go?to=https%3A%2F%2Fbaike.baidu.com%2Fitem%2F%25E4%25BA%2592%25E6%2596%25A5%25E9%2594%2581)，还是**自旋锁**，在任何时刻，最多只能有一个保持者，也就说，在任何时刻最多只能有一个执行单元获得锁。  
但是两者在调度机制上略有不同。对于互斥锁，如果资源已经被占用，资源申请者只能进入睡眠状态。但是自旋锁不会引起调用者睡眠，如果自旋锁已经被别的执行单元保持，调用者就一直循环在那里看是否该自旋锁的保持者已经释放了锁，"自旋" 一词就是因此而得名。  
由于不再安全, 这里不讨论用法了.

### pthread\_mutex 互斥锁

还是以卖票为例, 加锁后线程才是安全的:

```
#import <pthread.h>

@property (nonatomic, assign)   int globalInt;
@property (nonatomic, assign)   pthread\_mutex\_t mutex;

- (void)initData {
    
    self.tickets = 10;
    
    
    pthread\_mutex\_init(&self->\_mutex, NULL);
}


- (void)dealloc
{
    
    pthread\_mutex\_destroy(&\_mutex);
}

- (void)sample {
    
    dispatch\_async(dispatch\_get\_global\_queue(0, 0), ^{
        while (self.tickets) {
            \[self sellTickets\];
        }
        NSLog(@"窗口A, 票卖完了");
    });
    
    dispatch\_async(dispatch\_get\_global\_queue(0, 0), ^{
        while (self.tickets) {
            \[self sellTickets\];
        }
        NSLog(@"窗口B, 票卖完了");
    });
}


- (void)sellTickets {
        
    pthread\_mutex\_lock(&self->\_mutex);      
    
    if (self.tickets) {
        sleep(0.2);         
        self.tickets--;
        NSLog(@"出票一张, 余票:%d", self.tickets);
    }
        
    pthread\_mutex\_unlock(&self->\_mutex);    
}


log:
出票一张, 余票:9
出票一张, 余票:8
出票一张, 余票:7
出票一张, 余票:6
出票一张, 余票:5
出票一张, 余票:4
出票一张, 余票:3
出票一张, 余票:2
出票一张, 余票:1
出票一张, 余票:0
窗口A, 票卖完了
窗口B, 票卖完了


```

在上面例子中, 比如当线程 A 持有锁时, 线程 B 去获取锁, 线程 B 会被强制挂起等待, 直到线程 A 释放锁, 线程 B 才能获取锁继续运行.  
有时候我们不想挂起等待而去做些其他事情, 那么就可以使用 pthread\_mutex\_trylock() 尝试加锁:

```
\- (void)sellTickets {
        
    if (pthread\_mutex\_trylock(&self->\_mutex) == 0) {    
        
        if (self.tickets) {
            sleep(0.2);         
            self.tickets--;
            NSLog(@"出票一张, 余票:%d", self.tickets);
        }
        
        pthread\_mutex\_unlock(&self->\_mutex);            
        
    }else {
        
        
        NSLog(@"喝口水");
        sleep(0.5);
    }
}


```

### pthread\_mutex (RECURSIVE) 递归锁

如果函数存在递归调用, 那么重复 pthread\_mutex\_lock() 加锁会导致死锁. 我们可以改变初始化时的属性, 将锁类型改为递归锁.

```
\- (void)initData {
    
    self.tickets = 10;
    


    
    
    pthread\_mutexattr\_t attr;
    pthread\_mutexattr\_init(&attr);
    pthread\_mutexattr\_settype(&attr, PTHREAD\_MUTEX\_RECURSIVE);
    
    pthread\_mutex\_init(&self->\_mutex, &attr);
    
    pthread\_mutexattr\_destroy(&attr);
}


- (void)sellTickets {
    
    pthread\_mutex\_lock(&self->\_mutex);
    
    
    if (self.tickets) {
        sleep(0.2);         
        self.tickets--;
        NSLog(@"出票一张, 余票:%d", self.tickets);
        
        if (self.tickets) {
            \[self sellTickets\];
        }
    }
    
    pthread\_mutex\_unlock(&self->\_mutex);            
}


```

> 锁的默认属性是 PTHREAD\_MUTEX\_NORMAL, 不能用于递归实现.  
> 将属性改成 PTHREAD\_MUTEX\_RECURSIVE 后称为递归锁.

### pthread\_mutex + pthread\_cond 条件锁

还是卖票的例子, 我们加入一个退票窗口 C. 如果票卖完了, 窗口 A 和 B 进入等待, 直到窗口 C 有人退票后, 窗口 A 和 B 继续卖票. 其实也就是生产者 -- 消费者问题.

```
#import "ViewController.h"
#import <pthread.h>

@interface ViewController ()

@property (nonatomic, assign)   int tickets;
@property (nonatomic, assign)   pthread\_mutex\_t mutex;
@property (nonatomic, assign)   pthread\_cond\_t  cond;

@end

@implementation ViewController

- (void)viewDidLoad {
    \[super viewDidLoad\];
    
    \[self initData\];
    
    \[self sample\];
    
    sleep(5);
    
    \[self refundTicket\];
}


- (void)initData {
    
    self.tickets = 10;
    
    
    pthread\_mutex\_init(&self->\_mutex, NULL);

    
    pthread\_cond\_init(&self->\_cond, NULL);
}


- (void)dealloc
{
    pthread\_mutex\_destroy(&\_mutex);
    pthread\_cond\_destroy(&\_cond);
}


- (void)sample {
    
    dispatch\_async(dispatch\_get\_global\_queue(0, 0), ^{
        while (1) {
            \[self sellTickets\];
        }
    });
    
    dispatch\_async(dispatch\_get\_global\_queue(0, 0), ^{
        while (1) {
            \[self sellTickets\];
        }
    });
}


- (void)sellTickets {
    
    pthread\_mutex\_lock(&self->\_mutex);
    
    if (self.tickets == 0) {
        NSLog(@"无票, 等待中");
        
        pthread\_cond\_wait(&self->\_cond, &self->\_mutex);
    }
    
    sleep(0.2);         
    self.tickets--;
    NSLog(@"出票一张, 余票:%d", self.tickets);

    pthread\_mutex\_unlock(&self->\_mutex);
}


- (void)refundTicket {
    
    pthread\_mutex\_lock(&self->\_mutex);
    
    self.tickets++;
    NSLog(@"退票一张");
    
    
    pthread\_cond\_signal(&self->\_cond);

    

    
    pthread\_mutex\_unlock(&self->\_mutex);
}



log:
出票一张, 余票:9
出票一张, 余票:8
出票一张, 余票:7
出票一张, 余票:6
出票一张, 余票:5
出票一张, 余票:4
出票一张, 余票:3
出票一张, 余票:2
出票一张, 余票:1
出票一张, 余票:0
无票, 等待中
无票, 等待中
退票一张
出票一张, 余票:0
无票, 等待中


```

### NSLock 互斥锁

Apple 在 GitHub 上面开源了 Swift 版 Foundation 源码 ([https://github.com/apple/swift-corelibs-foundation](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fapple%2Fswift-corelibs-foundation)), 里面可以找到 NSLock 的定义实现.

![](http://upload-images.jianshu.io/upload_images/8900795-397cea6011137ad4.png)

swift\_fundation.png

由此可知, NSLock 内部封装了 pthread\_mutex. 类似的, NSCondition / NSConditionLock / NSRecursiveLock 都是基于 pthread\_mutex 封装的.

NSLock 封装 pthread\_mutex 的属性为 PTHREAD\_MUTEX\_ERRORCHECK, 它会损失一定性能换来错误提示. NSLock 比 pthread\_mutex 略慢的原因在于它需要经过方法调用, 同时由于缓存的存在, 多次方法调用不会对性能产生太大的影响.

NSLock 特性:

*   lock 和 unlock 必须在同一个线程完成;
*   两次调用 lock 会造成死锁, 也就是不能用于递归实现;
*   性能比 pthread\_mutex 略慢;
*   好用

NSLock 的四个方法:

```
\- (void)lock;
- (void)unlock;
- (BOOL)tryLock;
- (BOOL)lockBeforeDate:(NSDate \*)limit;  


```

用法 (参照之前 pthread\_mutex 例子):

```
\- (void)initData {
    
    self.tickets = 10;
    self.lock = \[\[NSLock alloc\] init\];
}

- (void)sellTickets {
        
    \[self.lock lock\];      
    
    if (self.tickets) {
        sleep(0.2);         
        self.tickets--;
        NSLog(@"出票一张, 余票:%d", self.tickets);
    }
        
    \[self.lock unlock\];    
}


```

### NSRecursiveLock 递归锁

为了弥补 NSLock 不能用于递归的问题.  
用法 (参照之前 pthread\_mutex 例子):

```
\- (void)initData {
    
    self.tickets = 10;
    self.recursiveLock = \[\[NSRecursiveLock alloc\] init\];
}

- (void)sellTickets {
    
    \[self.recursiveLock lock\];
    
    
    if (self.tickets) {
        sleep(0.2);         
        self.tickets--;
        NSLog(@"出票一张, 余票:%d", self.tickets);
        
        if (self.tickets) {
            \[self sellTickets\];
        }
    }
    
    \[self.recursiveLock unlock\];            
}


```

### NSCondition 条件锁

NSCondition 被 NSConditionLock 所封装, 但是只谈 NSCondition 的话, 两者就没有必然联系.  
虽然 NSCondition 不带 lock 字样, 但是他也是一种锁, 也遵循锁协议 NSLocking.  
NSCondition 封装了 pthread\_mutex + pthread\_cond.

用法 (参照之前 pthread\_mutex 例子):

```
\- (void)initData {
    
    self.tickets = 10;
    self.condition = \[\[NSCondition alloc\] init\];
}


- (void)sellTickets {
    

    \[self.condition lock\];
    
    if (self.tickets == 0) {
        NSLog(@"无票, 等待中");
        

        \[self.condition wait\];
    }
    
    sleep(0.2);         
    self.tickets--;
    NSLog(@"出票一张, 余票:%d", self.tickets);


    \[self.condition unlock\];
}


- (void)refundTicket {
    

    \[self.condition lock\];
    
    self.tickets++;
    NSLog(@"退票一张");
    
    

    \[self.condition signal\];
    
    


    

    \[self.condition unlock\];
}


```

### NSConditionLock 条件锁

NSConditionLock 封装了 NSCondition 和 一个 NSInteger 类型的 condition.  
NSConditionLock 可以称为条件锁, 只有 condition 参数与初始化时候的 condition 相等，lock 才能正确进行加锁操作.  
NSConditionLock 类似于信号量, 但不是对信号量的封装, 加入 condition 能实现线程间的依赖.

🌰

```
\- (void)initData {
    
    self.conditionLock = \[\[NSConditionLock alloc\] initWithCondition:0\];
}


- (void)sample2 {
    
    
    dispatch\_async(dispatch\_get\_global\_queue(0, 0), ^{
        \[self.conditionLock lockWhenCondition:1\];           
        NSLog(@"线程1");
        sleep(2);
        \[self.conditionLock unlock\];
    });
    
    
    dispatch\_async(dispatch\_get\_global\_queue(0, 0), ^{
        sleep(1);   
        if (\[self.conditionLock tryLockWhenCondition:0\]) {  
            NSLog(@"线程2");
            \[self.conditionLock unlockWithCondition:2\];     
            NSLog(@"线程2解锁成功");
        } else {
            NSLog(@"线程2尝试加锁失败");
        }
    });
    
    
    dispatch\_async(dispatch\_get\_global\_queue(DISPATCH\_QUEUE\_PRIORITY\_DEFAULT, 0), ^{
        sleep(2);   
        if (\[self.conditionLock tryLockWhenCondition:2\]) {
            NSLog(@"线程3");
            \[self.conditionLock unlock\];
            NSLog(@"线程3解锁成功");
        } else {
            NSLog(@"线程3尝试加锁失败");
        }
    });
    
    
    dispatch\_async(dispatch\_get\_global\_queue(DISPATCH\_QUEUE\_PRIORITY\_DEFAULT, 0), ^{
        sleep(3);   
        if (\[self.conditionLock tryLockWhenCondition:2\]) {  
            NSLog(@"线程4");
            \[self.conditionLock unlockWithCondition:1\];     
            NSLog(@"线程4解锁成功");
        } else {
            NSLog(@"线程4尝试加锁失败");
        }
    });
}


log:
线程2
线程2解锁成功
线程3
线程3解锁成功
线程4
线程4解锁成功
线程1


```

> unlockWithCondition: 并不是当 condition 符合条件时才解锁, 而是解锁之后, 修改 condition 的值.

### @synchronized

关于 synchronized, 这篇文章讲得比较深入 [http://rykap.com/objective-c/2015/05/09/synchronized/](https://links.jianshu.com/go?to=http%3A%2F%2Frykap.com%2Fobjective-c%2F2015%2F05%2F09%2Fsynchronized%2F)

一些注意事项:

*   @synchronized 既是互斥锁也是递归锁
*   @synchronized(NSObject) 括号里可以是任意 NSObject, 一般传入 VC 的 self
*   @synchronized(NSObject) 只有当 NSObject 是同一个时, 才满足互斥
*   如果在 @synchronized(NSObject) 内部 NSObject 被释放或被设为 nil, 不影响结果; 但如果 NSObject 一开始就是 nil, 则失去了锁的功能
*   @synchronized 会自动释放互斥锁
*   @synchronized 很耗性能, 应谨慎使用

用法:

```
@synchronized (self) {
    ...
}


```

"小礼物走一走，来简书关注我"

还没有人赞赏，支持一下

[![](https://upload.jianshu.io/users/upload_avatars/8900795/90987efc-b432-4f85-90cb-51aa3bc07f5a?imageMogr2/auto-orient/strip|imageView2/1/w/100/h/100/format/webp)](https://www.jianshu.com/u/83ecdf2d05e0)

总资产 7 (约 0.52 元) 共写了 8.3W 字获得 87 个赞共 84 个粉丝

### 被以下专题收入，发现更多相似内容