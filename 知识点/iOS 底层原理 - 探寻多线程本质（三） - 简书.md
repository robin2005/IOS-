\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[www.jianshu.com\](https://www.jianshu.com/p/6e0067299ea6)

[![](https://upload.jianshu.io/users/upload_avatars/1319505/9754fac0eb59?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96/format/webp)](https://www.jianshu.com/u/40dce98b3416)

2019.01.21 15:00:50 字数 1,229 阅读 118

#### 面试题引发的思考：

**Q: 线程安全的处理手段有哪些？**

*   **线程同步 9 种方案，加锁。**
*   **加锁避免 _同一时间、多个线程_ 同时 _读取并修改_ 一个值 而产生混乱。**

**Q: 锁的简介？**

![](http://upload-images.jianshu.io/upload_images/1319505-6c4a315edab54163.png)

QQ20201013-170005@2x.png

*   **自旋锁：等待锁的线程会处于 忙等状态；**
*   **互斥锁：等待锁的线程会处于 休眠状态；**
*   **条件锁：等待 符合某个条件 才能继续处理；**
*   **递归锁：同一个线程 对 同一把锁 加锁多次；**

* * *

### 1\. 多线程安全

#### (1) 多线程安全隐患

**当多个线程访问同一块资源时，很容易引发数据错乱和数据安全问题；**

比如多个线程访问同一个对象、同一个变量、同一个文件；  
具体场景为：存钱取钱、购买车票等。

#### (2) 多线程安全隐患分析

![](http://upload-images.jianshu.io/upload_images/1319505-da6f98917b0b0155.png)

多线程安全隐患分析

由以上图例可知：  
随 Time 时间线往右走，首先 Thread A 读取变量值 17，然后 Thread B 读取变量值 17，接着 Thread A、Thread B 分别对读数加 1 得到 18，都写进变量 18，此时会出现问题。

#### (3) 多线程安全隐患解决方案 - 线程同步

解决方案：使用线程同步技术（同步，就是协同步调，按预定的先后次序进行）；  
常见的线程同步技术是：加锁。

![](http://upload-images.jianshu.io/upload_images/1319505-cb268d9fc7ca07e3.png)

解决方案 - 加锁

由以上图例可知：  
随 Time 时间线往右走，首先 Thread A 读取变量值 17，然后对其加锁，接着 Thread A 对读数加 1 得到 18，然后对其解锁，解锁后 Thread B 才能对变量进行读取操作，保证线程安全。

* * *

### 2\. iOS 中的线程同步方案

#### (1) iOS 中的线程同步方案分析

> *   **`OSSpinLock` - 自旋锁：等待锁的线程会处于 忙等状态**
> *   **`os_unfair_lock` - 互斥锁：取代`OSSpinLock`**
> *   **`pthread_mutex` - 互斥锁：等待锁的线程会处于 休眠状态**  
>     (包括普通锁和递归锁，递归锁：同一个线程对同一把锁加锁多次)
> *   **`NSLock` - 对`mutex`普通锁的封装**
> *   **`NSRecursiveLock` - 对`mutex`递归锁的封装**
> *   **`NSCondition` - 对`mutex`和`cond`的封装**
> *   **`NSConditionLock` - 对`NSCondition`的封装**
> *   **`dispatch_queue(DISPATCH_QUEUE_SERIAL)`**
> *   **`dispatch_semaphore` - 信号量**
> *   **`@synchronized` - 对`mutex`递归锁的封装**

接下来逐一对以上方案进行介绍，相关代码放到 GitHub 的 [ThreadLock](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2FManXiaoYi%2FThreadLock) 中。

##### 1> 浅析`OSSpinLock`

> *   **`OSSpinLock`叫做” 自旋锁”，等待锁的线程会处于忙等（`busy-wait`）状态，一直占用着 CPU 资源；**
> *   **目前已经不再安全，可能会出现优先级反转问题；**
> *   **如果等待锁的线程优先级较高，它会一直占用着 CPU 资源，优先级低的线程就无法释放锁；**
> *   **需要导入头文件`#import <libkern/OSAtomic.h>`。**

```
OSSpinLock lock = OS\_SPINLOCK\_INIT;

bool result = OSSpinLockTry(&lock);

OSSpinLockLock(&lock);

OSSpinLockUnlock(&lock);


```

##### 2> 浅析`os_unfair_lock`

> *   **`os_unfair_lock`用于取代不安全的`OSSpinLock`，从 iOS10 开始才支持；**
> *   **从底层调用看，等待`os_unfair_lock`锁的线程会处于休眠状态，并非忙等；**
> *   **需要导入头文件`#import <os/lock.h>`。**

```
os\_unfair\_lock lock = OS\_UNFAIR\_LOCK\_INIT;

bool result = os\_unfair\_lock\_trylock(&lock);

os\_unfair\_lock\_lock(&lock);

os\_unfair\_lock\_unlock(&lock);


```

##### 3> 浅析`pthread_mutex`

> *   **`mutex`叫做” 互斥锁”，等待锁的线程会处于休眠状态；**
> *   **需要导入头文件`#import <pthread.h>`。**

###### a> `pthread_mutex`

```
#define PTHREAD\_MUTEX\_NORMAL        0 普通锁
#define PTHREAD\_MUTEX\_ERRORCHECK    1
#define PTHREAD\_MUTEX\_RECURSIVE     2  递归锁
#define PTHREAD\_MUTEX\_DEFAULT       PTHREAD\_MUTEX\_NORMAL


pthread\_mutexattr\_t attr;
pthread\_attr\_init(&attr);
pthread\_mutexattr\_settype(&attr, PTHREAD\_MUTEX\_NORMAL);

pthread\_mutex\_t mutex;
pthread\_mutex\_init(&mutex, &attr);

pthread\_mutex\_trylock(&mutex);

pthread\_mutex\_lock(&mutex);

pthread\_mutex\_unlock(&mutex);

pthread\_mutexattr\_destroy(&attr);
pthread\_mutex\_destroy(&mutex);


```

###### b> `pthread_mutex`递归锁

```
pthread\_mutexattr\_t attr;
pthread\_attr\_init(&attr);
pthread\_mutexattr\_settype(&attr, PTHREAD\_MUTEX\_RECURSIVE);

pthread\_mutex\_t mutex;
pthread\_mutex\_init(&mutex, &attr);


```

###### c> `pthread_mutex`条件

```
pthread\_mutex\_t mutex;
pthread\_mutex\_init(&mutex, NULL);

pthread\_cond\_t condition;
pthread\_cond\_init(&condition, NULL);

pthread\_cond\_wait(&condition, &mutex);

pthread\_cond\_signal(&condition);

pthread\_cond\_broadcast(&condition);

pthread\_mutex\_destroy(&mutex);
pthread\_cond\_destroy(&condition);


```

##### 4> 浅析`NSLock`

> **`NSLock`是对`mutex`普通锁的封装**

```
NSLock \*lock = \[\[NSLock alloc\] init\];

\[lock tryLock\];

\[lock lockBeforeDate:\[NSDate date\]\];

\[lock lock\];

\[lock unlock\];


```

##### 5> 浅析`NSRecursiveLock`

> **`NSRecursiveLock`是对`mutex`递归锁的封装，API 跟`NSLock`基本一致**

```
NSRecursiveLock \*lock = \[\[NSRecursiveLock alloc\] init\];

\[lock tryLock\];

\[lock lockBeforeDate:\[NSDate date\]\];

\[lock lock\];

\[lock unlock\];


```

##### 6> 浅析`NSCondition`

> **`NSCondition`是对`mutex`和`cond`的封装**

```
NSCondition \*condition = \[\[NSCondition alloc\] init\];

\[condition lock\];

\[condition wait\];

\[condition waitUntilDate:\[NSDate date\]\];

\[condition signal\];

\[condition broadcast\];

\[condition unlock\];


```

##### 7> 浅析`NSConditionLock`

> **`NSConditionLock`是对`NSCondition`的进一步封装，可以设置具体的条件值**

```
NSConditionLock \*condition = \[\[NSConditionLock alloc\] init\];
NSConditionLock \*condition = \[\[NSConditionLock alloc\] initWithCondition:1\];

\[condition lockWhenCondition:1 beforeDate:\[NSDate date\]\];
\[condition lockBeforeDate:\[NSDate date\]\];
\[condition lockWhenCondition:1\];
\[condition lock\];
\[condition tryLock\];
\[condition tryLockWhenCondition:1\];

\[condition unlockWithCondition:1\];
\[condition unlock\];



```

##### 8> 浅析`dispatch_queue(DISPATCH_QUEUE_SERIAL)`

> **使用 GCD 的串行队列，实现线程同步**

```
dispatch\_queue\_t queue = dispatch\_queue\_create("lock\_queue", DISPATCH\_QUEUE\_SERIAL);
dispatch\_sync(queue, ^{
    
});


```

##### 9> 浅析`dispatch_semaphore`

> *   **`semaphore`叫做” 信号量”**
> *   **信号量的初始值，可以用来控制线程并发访问的最大数量**
> *   **信号量的初始值为 1，代表同时只允许 1 条线程访问资源，保证线程同步**

```
int value = 1;

dispatch\_semaphore\_t semaphore = dispatch\_semaphore\_create(value);


dispatch\_semaphore\_wait(semaphore, DISPATCH\_TIME\_FOREVER);

dispatch\_semaphore\_signal(semaphore);


```

##### 10> 浅析`@synchronized`

> *   **`@synchronized`是对`mutex`递归锁的封装**
> *   **`@synchronized(obj)`内部会生成`obj`对应的递归锁，然后进行加锁、解锁操作**

* * *

#### (2) iOS 中的线程同步方案性能比较

性能从高到低排序：

> *   **`os_unfair_lock`**
> *   **`OSSpinLock`**
> *   **`dispatch_semaphore`**
> *   **`pthread_mutex`**
> *   **`dispatch_queue(DISPATCH_QUEUE_SERIAL)`**
> *   **`NSLock`**
> *   **`NSCondition`**
> *   **`pthread_mutex(recursive)`**
> *   **`NSRecursiveLock`**
> *   **`NSConditionLock`**
> *   **`@synchronized`**

* * *

#### (3) 自旋锁、互斥锁比较

**Q: 什么情况使用自旋锁比较划算？**

*   **预计线程等待锁的时间很短**
*   **加锁的代码（临界区）经常被调用，但竞争情况很少发生**
*   **CPU 资源不紧张**
*   **多核处理器**

**Q: 什么情况使用互斥锁比较划算？**

*   **预计线程等待锁的时间较长**
*   **单核处理器**
*   **临界区有 IO 操作**
*   **临界区代码复杂或者循环量大**
*   **临界区竞争非常激烈**

"小礼物走一走，来简书关注我"

还没有人赞赏，支持一下

[![](https://upload.jianshu.io/users/upload_avatars/1319505/9754fac0eb59?imageMogr2/auto-orient/strip|imageView2/1/w/100/h/100/format/webp)](https://www.jianshu.com/u/40dce98b3416)

总资产 1 (约 0.16 元) 共写了 4.8W 字获得 24 个赞共 23 个粉丝

### 被以下专题收入，发现更多相似内容

### 推荐阅读[更多精彩内容](https://www.jianshu.com/)

*   不以规矩．不能成方圆。--《孟子 · 离娄上》 说到指令集以及 CPU 架构体系，大家就会想到计算机专业课程里面的计算机体...
    
    [![](https://upload-images.jianshu.io/upload_images/1432482-e434b21a866788e6.jpeg?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)](https://www.jianshu.com/p/54884ce976ca)
*   第九章 软件架构设计 9.1 软件架构概述 9.1.1 软件架构的定义 定义 1：软件或计算机系统的软件架构是该系统...
    
    [![](https://upload.jianshu.io/users/upload_avatars/5408072/1a4b47fb-2b9e-4749-9fe0-41e448c15804.jpeg?imageMogr2/auto-orient/strip|imageView2/1/w/48/h/48/format/webp)步积](https://www.jianshu.com/u/4aec87ce0f2b)阅读 16
    
*   没有好的标题不用谈 “点击”，没有好的内容谈不上用户认可，而没有较高的可转载的价值性，也就缺少了再次传播的能力，所以...
    
*   青岛思妤妈妈 M9 任务一：听一曲音乐《宝贝你听到了吗》？认真听了这首对孩子充满爱的歌曲，自责愧疚的泪水划过脸庞，眼...
    
*   想要追寻自己的目标，还是继续得过且过的日子？我喜欢你什么，为什么就这么放不下，这是我想了好久都没有相通的问题，虽然...