\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[www.jianshu.com\](https://www.jianshu.com/p/a4307617a26d)

[![](https://upload.jianshu.io/users/upload_avatars/1635433/08c3f7594c77.jpg?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96/format/webp)](https://www.jianshu.com/u/0770de47bc70)

0.2572020.05.31 21:55:02 字数 1,160 阅读 668

摘录： [「想名真难」](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fu014600626%2Fjava%2Farticle%2Fdetails%2F102862777)、[「猴子的毛」](https://www.jianshu.com/p/58f5fb01ae4f)

简化核心函数 dispatch\_once\_f：

```
void dispatch\_once\_f(dispatch\_once\_t \*val, void \*ctxt, void (\*func)(void \*)){
    
    volatile long \*vval = val;
    if (dispatch\_atomic\_cmpxchg(val, 0l, 1l)) {
        func(ctxt); 
        dispatch\_atomic\_barrier();
        \*val = ~0l;
    } 
    else 
    {
        do
        {
            \_dispatch\_hardware\_pause();
        } while (\*vval != ~0l);
        dispatch\_atomic\_barrier();
    }
}


```

> 1、dispatch\_atomic\_cmpxchg，它是一个宏定义，原型为\_\_sync\_bool\_compare\_and\_swap((p), (o), (n)) ，这是 LockFree 给予 CAS 的一种原子操作机制，原理就是 如果 p==o，那么将 p 设置为 n，然后返回 true; 否则，不做任何处理返回 false

> 2、在多线程环境中，如果某一个线程 A 首次进入 dispatch\_once\_f，_val==0，这个时候直接将其原子操作设为 1，然后执行传入 dispatch\_once\_f 的 block，然后调用 dispatch\_atomic\_barrier，最后将_ val 的值修改为~ 0。

> 3、dispatch\_atomic\_barrier 是一种内存屏障，所谓内存屏障，从处理器角度来说，是用来串行化读写操作的，从软件角度来讲，就是用来解决顺序一致性问题的。编译器不是要打乱代码执行顺序吗，处理器不是要乱序执行吗，你插入一个内存屏障，就相当于告诉编译器，屏障前后的指令顺序不能颠倒，告诉处理器，只有等屏障前的指令执行完了，屏障后的指令才能开始执行。所以这里 dispatch\_atomic\_barrier 能保证只有在 block 执行完毕后才能修改 \* val 的值。

> 4、在首个线程 A 执行 block 的过程中，如果其它的线程也进入 dispatch\_once\_f，那么这个时候 if 的原子判断一定是返回 false，于是走到了 else 分支，于是执行了 dowhile 循环，其中调用了\_dispatch\_hardware\_pause，这有助于提高性能和节省 CPU 耗电，pause 就像 nop，干的事情就是延迟空等的事情。直到首个线程已经将 block 执行完毕且将 \* val 修改为 0，调用 dispatch\_atomic\_barrier 后退出。这么看来其它的线程是无法执行 block 的，这就保证了在 dispatch\_once\_f 的 block 的执行的唯一性，生成的单例也是唯一的。

*   `死锁方式1`：

> 1、某线程 T1() 调用单例 A，且为应用生命周期内首次调用，需要使用 dispatch\_once(&token, block()) 初始化单例。  
> 2、上述 block() 中的某个函数调用了 dispatch\_sync\_safe，同步在 T2 线程执行代码  
> 3、T2 线程正在执行的某个函数需要调用到单例 A，将会再次调用 dispatch\_once。  
> 4、这样 T1 线程在等 block 执行完毕，它在等待 T2 线程执行完毕，而 T2 线程在等待 T1 线程的 dispatch\_once 执行完毕，造成了相互等待，故而死锁

*   `死锁方式2`：

> 1、某线程 T1() 调用单例 A，且为应用生命周期内首次调用，需要使用 dispatch\_once(&token, block()) 初始化单例；  
> 2、block 中可能掉用到了 B 流程，B 流程又调用了 C 流程，C 流程可能调用到了单例 A，将会再次调用 dispatch\_once；  
> 3、这样又造成了相互等待。

##### dispatch\_once 也可以通过锁来实现，使用 dispatch\_semaphore,NSLock，@synchronized 这些都可以实现，但是效率没有 dispatch\_once 高。实测也是可以的。

```
\+ (instancetype)synchronizedManager {
 
    static Person \* m = nil;
    @synchronized (self) {
        
        if (m == nil) {
            
            sleep(3);
            m = \[\[self alloc\] init\];
            NSLog(@"synchronizedManager 只执行一次是对的");
        }
        
    }
    return m;
    
}


```

```
static dispatch\_semaphore\_t sem = nil;
+ (void)initialize {
    if (sem == nil) {
        sem = dispatch\_semaphore\_create(1);
    }
}
+ (instancetype)semaphoreManager {
    
    dispatch\_semaphore\_wait(sem, DISPATCH\_TIME\_FOREVER);
    static Person \* m = nil;
    if (m == nil) {
        
        sleep(3);
        m = \[\[self alloc\] init\];
        NSLog(@"semaphoreManager 只执行一次是对的");
    }
    dispatch\_semaphore\_signal(sem);
    return m;
}


```

### dispatch\_once 源码

Apple 对于 dispatch\_once 的[源码地址](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fapple%2Fswift-corelibs-libdispatch%2Fblob%2Fmaster%2Fsrc%2Fonce.c)

```
#include "internal.h"

#undef dispatch\_once
#undef dispatch\_once\_f

typedef struct \_dispatch\_once\_waiter\_s {
    volatile struct \_dispatch\_once\_waiter\_s \*volatile dow\_next;
    dispatch\_thread\_event\_s dow\_event;
    mach\_port\_t dow\_thread;
} \*\_dispatch\_once\_waiter\_t;

#define DISPATCH\_ONCE\_DONE ((\_dispatch\_once\_waiter\_t)~0l)

#ifdef \_\_BLOCKS\_\_
void
dispatch\_once(dispatch\_once\_t \*val, dispatch\_block\_t block)
{//第一步：我们调用dispatch\_once入口，接下来去看最下面dispatch\_once\_f的定义
    dispatch\_once\_f(val, block, \_dispatch\_Block\_invoke(block));
}
#endif

#if DISPATCH\_ONCE\_INLINE\_FASTPATH
#define DISPATCH\_ONCE\_SLOW\_INLINE inline DISPATCH\_ALWAYS\_INLINE
#else
#define DISPATCH\_ONCE\_SLOW\_INLINE DISPATCH\_NOINLINE
#endif 

DISPATCH\_ONCE\_SLOW\_INLINE
static void
dispatch\_once\_f\_slow(dispatch\_once\_t \*val, void \*ctxt, dispatch\_function\_t func)
{
#if DISPATCH\_GATE\_USE\_FOR\_DISPATCH\_ONCE
    dispatch\_once\_gate\_t l = (dispatch\_once\_gate\_t)val;

    if (\_dispatch\_once\_gate\_tryenter(l)) {
        \_dispatch\_client\_callout(ctxt, func);
        \_dispatch\_once\_gate\_broadcast(l);
    } else {
        \_dispatch\_once\_gate\_wait(l);
    }
#else//第三步：主要的流程（为什么走#else请看注解二）
    \_dispatch\_once\_waiter\_t volatile \*vval = (\_dispatch\_once\_waiter\_t\*)val;
    struct \_dispatch\_once\_waiter\_s dow = { };
    \_dispatch\_once\_waiter\_t tail = &dow, next, tmp;
    dispatch\_thread\_event\_t event;

//首次更改请求
    if (os\_atomic\_cmpxchg(vval, NULL, tail, acquire)) {
        dow.dow\_thread = \_dispatch\_tid\_self();
         //调用dispatch\_once内block回调
        \_dispatch\_client\_callout(ctxt, func);
         //利用while循环不断处理未完成的更改请求，直到所有更改结束
        next = (\_dispatch\_once\_waiter\_t)\_dispatch\_once\_xchg\_done(val);
        while (next != tail) {
            tmp = (\_dispatch\_once\_waiter\_t)\_dispatch\_wait\_until(next->dow\_next);
            event = &next->dow\_event;
            next = tmp;
            \_dispatch\_thread\_event\_signal(event);
        }
    } else {//非首次更改请求
        \_dispatch\_thread\_event\_init(&dow.dow\_event);
        next = \*vval;
        for (;;) {
//遍历每一个后续请求，如果状态已经是Done，直接进行下一个，同时该状态检测还用于避免在后续wait之前，信号量已经发出(signal)造成的死锁
            if (next == DISPATCH\_ONCE\_DONE) {
                break;
            }
//如果当前dispatch\_once执行的block没有结束，那么就将这些后续请求添加到链表当中
            if (os\_atomic\_cmpxchgv(vval, next, tail, &next, release)) {
                dow.dow\_thread = next->dow\_thread;
                dow.dow\_next = next;
                if (dow.dow\_thread) {
                    pthread\_priority\_t pp = \_dispatch\_get\_priority();
                    \_dispatch\_thread\_override\_start(dow.dow\_thread, pp, val);
                }
                \_dispatch\_thread\_event\_wait(&dow.dow\_event);
                if (dow.dow\_thread) {
                    \_dispatch\_thread\_override\_end(dow.dow\_thread, val);
                }
                break;
            }
        }
        \_dispatch\_thread\_event\_destroy(&dow.dow\_event);
    }
#endif
}

DISPATCH\_NOINLINE
void
dispatch\_once\_f(dispatch\_once\_t \*val, void \*ctxt, dispatch\_function\_t func)
{
#if !DISPATCH\_ONCE\_INLINE\_FASTPATH
    if (likely(os\_atomic\_load(val, acquire) == DLOCK\_ONCE\_DONE)) {
        return;
    }
#endif //第二步：进入dispatch\_once\_f\_slow(这个宏判断请看注解一)
    return dispatch\_once\_f\_slow(val, ctxt, func);
}



```

##### 注解一：

`DISPATCH_ONCE_INLINE_FASTPATH`这个宏的值由 CPU 架构决定，`__x86_64__`（64 位），`__i386__`（32 位），`__s390x__`（运行在 IBM z 系统 (_s390x_)，可能 Apple 和 IBM 比较熟，给他留后门了），以及`__APPLE__`这个就无从得知了，可能是 Apple 自身的平台架构，这些情况下`DISPATCH_ONCE_INLINE_FASTPATH = 1`，所以大部分情况也就是 1 了。

```
#if defined(\_\_x86\_64\_\_) || defined(\_\_i386\_\_) || defined(\_\_s390x\_\_)
#define DISPATCH\_ONCE\_INLINE\_FASTPATH 1
#elif defined(\_\_APPLE\_\_)
#define DISPATCH\_ONCE\_INLINE\_FASTPATH 1
#else
#define DISPATCH\_ONCE\_INLINE\_FASTPATH 0
#endif



```

##### 注解二：

`DISPATCH_GATE_USE_FOR_DISPATCH_ONCE`这个宏的值在`lock.h`中有定义：

```
#pragma mark - gate lock

#if HAVE\_UL\_UNFAIR\_LOCK || HAVE\_FUTEX
#define DISPATCH\_GATE\_USE\_FOR\_DISPATCH\_ONCE 1
#else
#define DISPATCH\_GATE\_USE\_FOR\_DISPATCH\_ONCE 0
#endif



```

而`HAVE_UL_UNFAIR_LOCK`的值和`HAVE_FUTEX`的值也在`lock.h`中有定义：

```
#ifdef \_\_linux\_\_
#define HAVE\_FUTEX 1
#else
#define HAVE\_FUTEX 0
#endif



```

```
#ifdef UL\_UNFAIR\_LOCK
#define HAVE\_UL\_UNFAIR\_LOCK 1
#endif



```

从上面的分析可以看出：  
1、dispatch\_once 不止是简单的执行一次，如果再次调用会进入非首次更改的模块，如果有未 DONE 的请求会被添加到链表中  
2、所以 dispatch\_once 本质上可以接受多次请求，会对此维护一个请求链表  
3、如果在 block 执行期间，多次进入调用同类的 dispatch\_once 函数（即单例函数），会导致整体链表无限增长，造成永久性死锁  
4、对于开始问题大致上和 A -> B -> A 的流程类似，理解 dispatch\_once 的内部流程有利于在使用中规避隐藏的问题。

"小礼物走一走，来简书关注我"

还没有人赞赏，支持一下

[![](https://upload.jianshu.io/users/upload_avatars/1635433/08c3f7594c77.jpg?imageMogr2/auto-orient/strip|imageView2/1/w/100/h/100/format/webp)](https://www.jianshu.com/u/0770de47bc70)

总资产 4 (约 0.31 元) 共写了 2.4W 字获得 53 个赞共 13 个粉丝

### 被以下专题收入，发现更多相似内容