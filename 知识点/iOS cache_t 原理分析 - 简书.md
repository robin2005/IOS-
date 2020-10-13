\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[www.jianshu.com\](https://www.jianshu.com/p/856031e38ddc)

[![](https://upload.jianshu.io/users/upload_avatars/2987980/bf0c968f4e57?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96/format/webp)](https://www.jianshu.com/u/7a85a6956d1d)

0.2312020.09.21 14:56:21 字数 923 阅读 30

> [iOS 底层探索 文章汇总](https://www.jianshu.com/p/77dae1109e07)

上一篇文章 [iOS 类的结构分析](https://www.jianshu.com/p/9f05d585144a)中我们分析了类的底层结构，知道了类中存在`cache_t cache`。那么`cache`中到底缓存了哪些数据，`cache_t`的底层结构又是怎样的呢？这篇文章我们就一起来分析类的底层结构到底是什么。  
类的底层代码如下:

```
struct objc\_class : objc\_object {
    
    Class superclass;          
    cache\_t cache;             
    class\_data\_bits\_t bits;    

....省略后面的代码.....
}


```

首先通过底层代码我们来看一下`cache_t`的底层是如何定义的：

```
struct cache\_t {
#if CACHE\_MASK\_STORAGE == CACHE\_MASK\_STORAGE\_OUTLINED -- macOS使用
    explicit\_atomic<struct bucket\_t \*> \_buckets;
    explicit\_atomic<mask\_t> \_mask;  
#elif CACHE\_MASK\_STORAGE == CACHE\_MASK\_STORAGE\_HIGH\_16 -- iOS使用
    explicit\_atomic<uintptr\_t> \_maskAndBuckets;
    mask\_t \_mask\_unused;
    
    
    static constexpr uintptr\_t maskShift = 48;
    static constexpr uintptr\_t maskZeroBits = 4;
    
    
    static constexpr uintptr\_t maxMask = ((uintptr\_t)1 << (64 - maskShift)) - 1;
    
    
    static constexpr uintptr\_t bucketsMask = ((uintptr\_t)1 << (maskShift - maskZeroBits)) - 1;
    
    
    static\_assert(bucketsMask >= MACH\_VM\_MAX\_ADDRESS, "Bucket field doesn't have enough bits for arbitrary pointers.");
#elif CACHE\_MASK\_STORAGE == CACHE\_MASK\_STORAGE\_LOW\_4 --其他设备使用
    explicit\_atomic<uintptr\_t> \_maskAndBuckets;
    mask\_t \_mask\_unused;

    static constexpr uintptr\_t maskBits = 4;
    static constexpr uintptr\_t maskMask = (1 << maskBits) - 1;
    static constexpr uintptr\_t bucketsMask = ~maskMask;
#else
#error Unknown cache mask storage type.
#endif
    
#if \_\_LP64\_\_
    uint16\_t \_flags;
#endif
    uint16\_t \_occupied;

public:
    static bucket\_t \*emptyBuckets();
    
    struct bucket\_t \*buckets();
    mask\_t mask();
    mask\_t occupied();  
    void incrementOccupied();
    void setBucketsAndMask(struct bucket\_t \*newBuckets, mask\_t newMask);
    void initializeToEmpty();

    unsigned capacity();
    bool isConstantEmptyCache();
    bool canBeFreed();

....省略后面的代码.....
};


```

#### 1\. 针对`x86架构`的`macOS`分析可知主要有以下字段：

*   explicit\_atomic<struct bucket\_t \*>`_buckets`;
*   explicit\_atomic<mask\_t> `_mask`;
*   uint16\_t `_flags`;
*   uint16\_t `_occupied`;

各变量的主要作用：

*   `_buckets`：数组，是`bucket_t结构体`的数组，`bucket_t`是用来存放方法的`SEL内存地址`和`IMP`的。
*   `_mask`的大小是`数组大小 - 1`，用作掩码。（因为这里维护的数组大小都是 2 的整数次幂，所以`_mask`的二进制位 000011, 000111, 001111）刚好可以用作`hash取余数`的掩码。刚好**保证相与后不超过缓存大小**。
*   `_occupied`是当前已缓存的方法数。即数组中已使用了多少位置。

#### 2\. `cache_t`中主要的方法：

*   struct bucket\_t \*`buckets()`; // 获取 buckets
*   mask\_t `mask()`; // 获取 mask
*   mask\_t `occupied()`; // 获取 occupied
*   void `incrementOccupied()`;// 增加 occupied
*   void `setBucketsAndMask(struct bucket_t *newBuckets, mask_t newMask)`;// 设置 Buckets 和 Mask

`bucket_t`底层定义如下：

```
struct bucket\_t {
private:
    explicit\_atomic<SEL> \_sel;
    explicit\_atomic<uintptr\_t> \_imp; 
....省略后面的代码.....

public:
    inline SEL sel() const { return \_sel.load(memory\_order::memory\_order\_relaxed); }
    inline IMP imp(Class cls) const {
        . . .
        return (IMP)imp;
    }
....省略后面的代码.....
}


```

`bucket_t`中只有`_sel`和`_imp`，所以类中的`cache`就是用来`缓存方法`的。

如果要探索`cache_t`的工作原理，我们需要从`cache`的`insert`方法开始探索：

```
ALWAYS\_INLINE
void cache\_t::insert(Class cls, SEL sel, IMP imp, id receiver)
{
#if CONFIG\_USE\_CACHE\_LOCK
    cacheUpdateLock.assertLocked();
#else
    runtimeLock.assertLocked();
#endif

    ASSERT(sel != 0 && cls->isInitialized());

    
    mask\_t newOccupied = occupied() + 1;
    unsigned oldCapacity = capacity(), capacity = oldCapacity;
    if (slowpath(isConstantEmptyCache())) {
        
        if (!capacity) capacity = INIT\_CACHE\_SIZE;
        reallocate(oldCapacity, capacity, false);
    }
    else if (fastpath(newOccupied + CACHE\_END\_MARKER <= capacity / 4 \* 3)) { 
        
    }
    else {
        capacity = capacity ? capacity \* 2 : INIT\_CACHE\_SIZE;  
        if (capacity > MAX\_CACHE\_SIZE) {
            capacity = MAX\_CACHE\_SIZE;
        }
        reallocate(oldCapacity, capacity, true);  
    }

    bucket\_t \*b = buckets();
    mask\_t m = capacity - 1;
    mask\_t begin = cache\_hash(sel, m);
    mask\_t i = begin;

    
    
    
    do {
        if (fastpath(b\[i\].sel() == 0)) {
            incrementOccupied();
            b\[i\].set<Atomic, Encoded>(sel, imp, cls);
            return;
        }
        if (b\[i\].sel() == sel) {
            
            
            return;
        }
    } while (fastpath((i = cache\_next(i, m)) != begin));

    cache\_t::bad\_cache(receiver, (SEL)sel, cls);
}



```

#### `insert`的工作流程如下：

1.  检查`Cache`容量  
    I. 如果`Cache`为空就分配内存并初始化  
    II. 如果最新一个插入后占用`小于等于`总容量的`3/4`就不扩容  
    III. 当总容量`大于3/4`就将当前容量`扩展两倍`，之前缓存的内存直接释放，不会将之前缓存的方法存到新内存中
    
2.  插入最新的方法缓存  
    I. 获取到`buckets`  
    II. 设置初始插入`index`为`cache_hash(sel, m); ------> return sel & mask`  
    III. 循环遍历`buckets`  
         i. 如果当前`bucket`中没有`sel`就存在当前位置并退出方法  
        ii. 如果当前`bucket`中存的`sel`和即将要缓存的`sel`一致就退出方法  
        iii. 下一个缓存的`index cache_next(i, m) ------> return (i+1) & mask`  
        iiii. 如果下一个缓存的`index`和`初始index`相等就退出循环  
    IIII. 如果缓存`sel`没有存储就执行`cache_t::bad_cache(receiver, (SEL)sel, cls);`方法
    

> **sel & mask = index： index 一定是 <=\_ mask**

3.  底层代码分析图：
    
    ![](http://upload-images.jianshu.io/upload_images/2987980-1ccd4305bdd09b57.png)
    
    Cache\_t 原理分析
    

#### `Cache`的整体流程在`objc-cache.mm`文件中可以看到：

```
 \* Cache readers      (PC-checked by collecting\_in\_critical())
 \* objc\_msgSend\*
 \* cache\_getImp
 \*
 \* Cache writers      (hold cacheUpdateLock while reading or writing; not PC-checked)
 \* cache\_fill         (acquires lock)
 \* cache\_expand       (only called from cache\_fill)
 \* cache\_create       (only called from cache\_expand)
 \* bcopy              (only called from instrumented cache\_expand)
 \* flush\_caches       (acquires lock)
 \* cache\_flush        (only called from cache\_fill and flush\_caches)
 \* cache\_collect\_free (only called from cache\_expand and cache\_flush)


```

*   `OC` 中实`例方法`缓存在`类`上面，`类方法`缓存在`元类`上面。
*   即使`子类`调用`父类`的方法，那么`父类`的这个方法也`不会缓存到子类`中。子类和父类`各种缓存自己的方法`。
*   `cache_t` 缓存会提前进行扩容`防止溢出`。
*   `方法缓存`是为了最大化的提高程序的`执行效率`。
*   苹果在方法缓存这里用的是`开放寻址法`来解决`哈希冲突`。
*   通过`cache_t` 我们可以进一步延伸去探究`objc_msgSend`，因为查找方法缓存是属于`objc_msgSend` 查找方法实现的`快速流程`。

[iOS - 底层原理 11：objc\_class 中 cache 原理分析](https://www.jianshu.com/p/3ad9166c02e5)

"小礼物走一走，来简书关注我"

还没有人赞赏，支持一下

[![](https://upload.jianshu.io/users/upload_avatars/2987980/bf0c968f4e57?imageMogr2/auto-orient/strip|imageView2/1/w/100/h/100/format/webp)](https://www.jianshu.com/u/7a85a6956d1d)

[differ\_iOS](https://www.jianshu.com/u/7a85a6956d1d "differ_iOS") 有理想爱学习的实力派…… 做喜欢的事，让喜欢的事有价值。 Alter what is ch...

总资产 11 (约 0.99 元) 共写了 3.5W 字获得 217 个赞共 73 个粉丝

### 被以下专题收入，发现更多相似内容