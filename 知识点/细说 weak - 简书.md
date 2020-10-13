\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[www.jianshu.com\](https://www.jianshu.com/p/10c0f49f4755)

[![](https://upload.jianshu.io/users/upload_avatars/2067180/c0c37aff-e45a-422f-8196-50adefdf7ea4.jpeg?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96/format/webp)](https://www.jianshu.com/u/42d133a6bae9)

0.4332018.08.04 18:40:41 字数 2,521 阅读 998

*   被 weak 修饰的对象在被释放的时候会发生什么？
*   是如何实现的？
*   知道 sideTable 么？
*   里面的结构可以画出来么？

在回答这些问题之前，我们先来了解一下 weak 的内部结构。

> 对于我们常说的 weak，其实是一个由 Runtime 维护的用于存储对象的所有 weak 指针的 hash 表。key 是所指对象的指针，value 是 weak 指针的地址（这个地址的值是所指对象的地址）数组。 光这么说还是不好理解，来看一个列子吧

```
NSObject \*b = \[NSObject new\];
\_\_weak id a = b;


```

> 这里的 b 就是 weak 表的 key,&a(a 的内存地址) 就是 value;

#### 那这个结论是怎样得到的？

> 在我们加入 runtime 源码进行调试的时候，我们发现 weak 修饰符变量是通过 objc\_initWeak 函数来初始化的，在变量作用域结束的时候通过 objc\_destroyWeak 函数来释放该变量的。这两个函数长下面这样👇

```
id objc\_initWeak(id \*location, id newObj)
{
    
    
    if (!newObj) {
        \*location = nil;
        return nil;
    }
    
    
    return storeWeak<false, true, true>
        (location, (objc\_object\*)newObj);
}


```

```
void objc\_destroyWeak(id \*location)
{
    (void)storeWeak<true, false, false>
        (location, nil);
}


```

从上两段代码不难发现它们都调用了 storeWeak 这个函数，但是两个方法传入的参数却稍有不同。  
init 方法中，第一个参数为 weak 修饰的变量，第二个参数为引用计数对象。但在 destoryWeak 函数，第一参数依旧为 weak 修饰的变量，第二个参数为 nil。那这块传入不同的参数到底代表什么，我们继续分析 storeWeak 这个函数。

```


template <bool HaveOld, bool HaveNew, bool CrashIfDeallocating>
static id storeWeak(id \*location, objc\_object \*newObj)
{
    assert(HaveOld  ||  HaveNew);
    if (!HaveNew) assert(newObj == nil);

    Class previouslyInitializedClass = nil;
    id oldObj;
    
    
    SideTable \*oldTable;
    SideTable \*newTable;

    
    
    
    
    
    
 retry:
    if (HaveOld) {
    
        oldObj = \*location;
        oldTable = &SideTables()\[oldObj\];
    } else {
        oldTable = nil;
    }
    if (HaveNew) {
    
        newTable = &SideTables()\[newObj\];
    } else {
        newTable = nil;
    }
    

    SideTable::lockTwo<HaveOld, HaveNew>(oldTable, newTable);


    if (HaveOld  &&  \*location != oldObj) {
        SideTable::unlockTwo<HaveOld, HaveNew>(oldTable, newTable);
        goto retry;
    }

    
    
    
    
    if (HaveNew  &&  newObj) {
        
        Class cls = newObj->getIsa();
        
        if (cls != previouslyInitializedClass  &&  
            !((objc\_class \*)cls)->isInitialized()) 
        {
            
            SideTable::unlockTwo<HaveOld, HaveNew>(oldTable, newTable);
            \_class\_initialize(\_class\_getNonMetaClass(cls, (id)newObj));

            
            
            
            
            
            
            
            previouslyInitializedClass = cls;

            goto retry;
        }
    }

    
    if (HaveOld) {
        weak\_unregister\_no\_lock(&oldTable->weak\_table, oldObj, location);
    }

    
    if (HaveNew) {
        newObj = (objc\_object \*)weak\_register\_no\_lock(&newTable->weak\_table, 
                                                      (id)newObj, location, 
                                                      CrashIfDeallocating);
        
        
        
        
        if (newObj  &&  !newObj->isTaggedPointer()) {
            newObj->setWeaklyReferenced\_nolock();
        }

        
        \*location = (id)newObj;
    }
    else {
        
    }
    
    SideTable::unlockTwo<HaveOld, HaveNew>(oldTable, newTable);

    return (id)newObj;
}


```

以上就是 store\_weak 这个函数的实现，它主要做了以下几件事：

*   声明了新旧散列表指针，因为 weak 修饰的变量如果之前已经指向一个对象，然后其再次改变指向另一个对象，那么按理来说我们需要从 oldTable 中删除 weak 变量的记录，也就是要释放该 weak 变量，然后再给 newTable 添加新记录（weak 变量）。这里的新旧散列表就是这个作用。
*   根据新旧变量的地址获取相应的 SideTable
*   对两个表进行加锁操作，防止多线程竞争冲突
*   进行线程冲突重处理判断
*   判断其 isa 是否为空，为空则需要进行初始化
*   如果存在旧值，调用 weak\_unregister\_no\_lock 函数清除旧值
*   调用 weak\_register\_no\_lock 函数分配新值
*   解锁两个表，并返回第二参数

#### tip

> 你可以把 objc\_storeWeak(id \*location, objc\_object \*newObj) 理解为：objc\_storeWeak(value, key)，并且当 key 变 nil，将 value 置 nil。  
> 结合最开始的例子我们可以理解为 objc\_storeWeak(&a, b)  
> 在 b 非 nil 时，a 和 b 指向同一个内存地址，在 b 变 nil 时，a 变 nil。此时向 a 发送消息不会崩溃：在 Objective-C 中向 nil 发送消息是安全的。  
> 而如果 a 是由 assign 修饰的，则： 在 b 非 nil 时，a 和 b 指向同一个内存地址，在 b 变 nil 时，a 还是指向该内存地址，变野指针。此时向 a 发送消息极易崩溃。

#### 初始化弱引用对象流程一览

![](http://upload-images.jianshu.io/upload_images/2726320-602af5d602df0877.png)

初始化弱引用对象流程一览

#### SideTable

可能大家对 sideTale 不太明白，sideTable 主要用于管理对象的引用计数和 weak 表。在 NSObject.mm 中声明其数据结构：

```
struct SideTable {

    spinlock\_t slock;
    
    RefcountMap refcnts;
    
    weak\_table\_t weak\_table;
}


```

对于 slock 和 refcnts 两个成员不用多说，第一个是为了防止竞争选择的自旋锁，第二个是协助对象的 isa 指针的 extra\_rc 共同引用计数的变量。这里主要看 weak\_table\_t 的结构与作用。

#### weak\_table\_t

weak\_table\_t 结构体存储了某个对象相关的的所有的弱引用信息。其定义如下 (具体定义在 objc-weak.h 中)：

```
struct weak\_table\_t {
    
    weak\_entry\_t \*weak\_entries;
    
    size\_t    num\_entries;
    
    uintptr\_t mask;
    
    uintptr\_t max\_hash\_displacement;
};


```

使用 weak 指针指向的对象的地址作为 key ，用 weak\_entry\_t 类型结构体对象作为 value 。其中的 weak\_entries 成员，从字面意思上看，即为弱引用表入口。

#### weak\_entry\_t

其中 weak\_entry\_t 是存储在 weak\_table\_t 中的一个内部结构体，它负责维护和存储指向一个对象的所有弱引用 hash 表。其定义如下：

```
typedef objc\_object \*\* weak\_referrer\_t;
struct weak\_entry\_t {
    DisguisedPtrobjc\_object> referent;
    union {
        struct {
            weak\_referrer\_t \*referrers;
            uintptr\_t        out\_of\_line : 1;
            uintptr\_t        num\_refs : PTR\_MINUS\_1;
            uintptr\_t        mask;
            uintptr\_t        max\_hash\_displacement;
        };
        struct {
            weak\_referrer\_t  inline\_referrers\[WEAK\_INLINE\_COUNT\];
        };
    }
}


```

**referent:**  
被指对象的地址。前面循环遍历查找的时候就是判断目标地址是否和他相等。

**referrers:**  
可变数组, 里面保存着所有指向这个对象的弱引用的地址。当这个对象被释放的时候，referrers 里的所有指针都会被设置成 nil。

**inline\_referrers:**  
只有 4 个元素的数组，默认情况下用它来存储弱引用的指针。当大于 4 个的时候使用 referrers 来存储指针。

#### 讲了这 3 个东西，我们用一张图来体现他们的关系

![](http://upload-images.jianshu.io/upload_images/2067180-053427ef2f04809b.png)

#### weak\_unregister\_no\_lock

结合刚刚讲到的从 oldTable 中删除 weak 变量的记录，来看看 weak\_unregister\_no\_lock 函数如何清除旧值的。

```
void weak\_unregister\_no\_lock(weak\_table\_t \*weak\_table, id referent\_id, 
                        id \*referrer\_id)
{
    objc\_object \*referent = (objc\_object \*)referent\_id;
    objc\_object \*\*referrer = (objc\_object \*\*)referrer\_id;

    weak\_entry\_t \*entry;

    if (!referent) return;

    if ((entry = weak\_entry\_for\_referent(weak\_table, referent))) {
        remove\_referrer(entry, referrer);
        bool empty = true;
        if (entry->out\_of\_line  &&  entry->num\_refs != 0) {
            empty = false;
        }
        else {
            for (size\_t i = 0; i < WEAK\_INLINE\_COUNT; i++) {
                if (entry->inline\_referrers\[i\]) {
                    empty = false; 
                    break;
                }
            }
        }

        if (empty) {
            weak\_entry\_remove(weak\_table, entry);
        }
    }

    
    
}


```

该方法主要作用是将旧对象在 weak\_table 中解除 weak 指针的对应绑定。根据函数名，称之为解除注册操作。  
来看看这个函数的逻辑。首先参数是 weak\_table\_t 表，键和值。声明 weak\_entry\_t 变量，如果 key（referent）为空，直接返回。根据全局入口表和键获取对应的 weak\_entry\_t 对象 entry。获取到 entry 后，将 entry 以及 weak\_table 作为参数传入 remove\_referrer 函数中，这个函数就是解除操作。然后判断 entry 是否为空，如果为空，从全局记录表中清除相应的 entry。

#### weak\_entry\_for\_referent

接下来，我们了解一下，如何获取这个 weak\_entry\_t 这个变量。

```
static weak\_entry\_t \*weak\_entry\_for\_referent(weak\_table\_t \*weak\_table, objc\_object \*referent)
{
    assert(referent);

    weak\_entry\_t \*weak\_entries = weak\_table->weak\_entries;

    if (!weak\_entries) return nil;

    size\_t index = hash\_pointer(referent) & weak\_table->mask;
    size\_t hash\_displacement = 0;
    while (weak\_table->weak\_entries\[index\].referent != referent) {
        index = (index+1) & weak\_table->mask;
        hash\_displacement++;
        if (hash\_displacement > weak\_table->max\_hash\_displacement) {
            return nil;
        }
    }

    return &weak\_table->weak\_entries\[index\];
}



```

这个函数的逻辑就是先获取全局 weak 表入口，然后将引用计数对象的地址进行 hash 化后与 weak\_table->mask 做与操作，作为下标，在全局 weak 表中查找，若找到，返回 entry，若没有，返回 nil。

#### remove\_referrer

再来了解一下解除对象的函数：

```
static void remove\_referrer(weak\_entry\_t \*entry, objc\_object \*\*old\_referrer)
{
    if (! entry->out\_of\_line) {
        for (size\_t i = 0; i < WEAK\_INLINE\_COUNT; i++) {
            if (entry->inline\_referrers\[i\] == old\_referrer) {
                entry->inline\_referrers\[i\] = nil;
                return;
            }
        }
        \_objc\_inform("Attempted to unregister unknown \_\_weak variable "
                     "at %p. This is probably incorrect use of "
                     "objc\_storeWeak() and objc\_loadWeak(). "
                     "Break on objc\_weak\_error to debug.\\n", 
                     old\_referrer);
        objc\_weak\_error();
        return;
    }

    size\_t index = w\_hash\_pointer(old\_referrer) & (entry->mask);
    size\_t hash\_displacement = 0;
    while (entry->referrers\[index\] != old\_referrer) {
        index = (index+1) & entry->mask;
        hash\_displacement++;
        if (hash\_displacement > entry->max\_hash\_displacement) {
            \_objc\_inform("Attempted to unregister unknown \_\_weak variable "
                         "at %p. This is probably incorrect use of "
                         "objc\_storeWeak() and objc\_loadWeak(). "
                         "Break on objc\_weak\_error to debug.\\n", 
                         old\_referrer);
            objc\_weak\_error();
            return;
        }
    }
    entry->referrers\[index\] = nil;
    entry->num\_refs--;
}



```

这个函数传入的是 weak\_entry 对象，当 out\_of\_line 为 0 时，遍历数组，找到对应的对象，置 nil，如果未找到，报错并返回。当 out\_of\_line 不为 0 时，根据对象的地址 hash 化并和 mask 做与操作作为下标，查找相应的对象，若没有，报错并返回，若有，相应的置为 nil，并减少元素数量，即 num\_refs 减 1。

清除旧值就讲完了  
来看看添加新值

#### weak\_register\_no\_lock

```
id weak\_register\_no\_lock(weak\_table\_t \*weak\_table, id referent\_id, 
                      id \*referrer\_id, bool crashIfDeallocating)
{
    objc\_object \*referent = (objc\_object \*)referent\_id;
    objc\_object \*\*referrer = (objc\_object \*\*)referrer\_id;

    if (!referent  ||  referent->isTaggedPointer()) return referent\_id;

    
    bool deallocating;
    if (!referent->ISA()->hasCustomRR()) {
        deallocating = referent->rootIsDeallocating();
    }
    else {
        BOOL (\*allowsWeakReference)(objc\_object \*, SEL) = 
            (BOOL(\*)(objc\_object \*, SEL))
            object\_getMethodImplementation((id)referent, 
                                           SEL\_allowsWeakReference);
        if ((IMP)allowsWeakReference == \_objc\_msgForward) {
            return nil;
        }
        deallocating =
            ! (\*allowsWeakReference)(referent, SEL\_allowsWeakReference);
    }

    if (deallocating) {
        if (crashIfDeallocating) {
            \_objc\_fatal("Cannot form weak reference to instance (%p) of "
                        "class %s. It is possible that this object was "
                        "over-released, or is in the process of deallocation.",
                        (void\*)referent, object\_getClassName((id)referent));
        } else {
            return nil;
        }
    }

    
    weak\_entry\_t \*entry;
    if ((entry = weak\_entry\_for\_referent(weak\_table, referent))) {
        append\_referrer(entry, referrer);
    } 
    else {
        weak\_entry\_t new\_entry;
        new\_entry.referent = referent;
        new\_entry.out\_of\_line = 0;
        new\_entry.inline\_referrers\[0\] = referrer;
        for (size\_t i = 1; i < WEAK\_INLINE\_COUNT; i++) {
            new\_entry.inline\_referrers\[i\] = nil;
        }

        weak\_grow\_maybe(weak\_table);
        weak\_entry\_insert(weak\_table, &new\_entry);
    }

    
    

    return referent\_id;
}



```

一大堆 if-else, 主要是为了判断该对象是不是 taggedPoint 以及是否正在调用 dealloca 等。下面操作开始，同样是先获取 weak\_entry，如果获取到，则调用 append\_referrer 插入对象，若没有，则新建一个 weak\_entry，默认为 out\_of\_line，然后将新对象放到 0 下标位置，其他位置置为 nil 。下面两个函数 weak\_grow\_maybe 是用来判断是否需要重申请内存重 hash，weak\_entry\_insert 函数是用来将新建的 weak\_entry 插入到全局 weak 表中。插入时同样是以对象地址的 hash 化和 mask 值相与作为下标来记录的。

接下来看看 append\_referrer 函数，源代码如下：

##### append\_referrer

```
static void append\_referrer(weak\_entry\_t \*entry, objc\_object \*\*new\_referrer)
{
    if (! entry->out\_of\_line) {
        
        for (size\_t i = 0; i < WEAK\_INLINE\_COUNT; i++) {
            if (entry->inline\_referrers\[i\] == nil) {
                entry->inline\_referrers\[i\] = new\_referrer;
                return;
            }
        }

        
        weak\_referrer\_t \*new\_referrers = (weak\_referrer\_t \*)
            calloc(WEAK\_INLINE\_COUNT, sizeof(weak\_referrer\_t));
        
        
        for (size\_t i = 0; i < WEAK\_INLINE\_COUNT; i++) {
            new\_referrers\[i\] = entry->inline\_referrers\[I\];
        }
        entry->referrers = new\_referrers;
        entry->num\_refs = WEAK\_INLINE\_COUNT;
        entry->out\_of\_line = 1;
        entry->mask = WEAK\_INLINE\_COUNT-1;
        entry->max\_hash\_displacement = 0;
    }

    assert(entry->out\_of\_line);

    if (entry->num\_refs >= TABLE\_SIZE(entry) \* 3/4) {
        return grow\_refs\_and\_insert(entry, new\_referrer);
    }
    size\_t index = w\_hash\_pointer(new\_referrer) & (entry->mask);
    size\_t hash\_displacement = 0;
    while (entry->referrers\[index\] != NULL) {
        index = (index+1) & entry->mask;
        hash\_displacement++;
    }
    if (hash\_displacement > entry->max\_hash\_displacement) {
        entry->max\_hash\_displacement = hash\_displacement;
    }
    weak\_referrer\_t &ref = entry->referrers\[index\];
    ref = new\_referrer;
    entry->num\_refs++;
}



```

当 out\_of\_line 为 0，并且静态数组里面还有位置存放，那么直接存放并返回。如果没有位置存放，则升级为动态数组，并加入。如果 out\_of\_line 不为 0，先判断是否需要扩容，然后同样的，使用对象地址的 hash 化和 mask 做与操作作为下标，找到相应的位置并插入。

#### 对象的销毁以及 weak 的置 nil 实现

释放时，调用 clearDeallocating 函数。clearDeallocating 函数首先根据对象地址获取所有 weak 指针地址的数组，然后遍历这个数组把其中的数据设为 nil，最后把这个 entry 从 weak 表中删除，最后清理对象的记录。

当 weak 引用指向的对象被释放时，又是如何去处理 weak 指针的呢？当释放对象时，其基本流程如下：

*   调用 objc\_release
*   因为对象的引用计数为 0，所以执行 dealloc
*   在 dealloc 中，调用了\_objc\_rootDealloc 函数
*   在 \_objc\_rootDealloc 中，调用了 objec\_dispose 函数
*   调用 objc\_destructInstance
*   最后调用 objc\_clear\_deallocating

#### objc\_clear\_deallocating 的具体实现如下：

```
void objc\_clear\_deallocating(id obj) 
{
    assert(obj);
    assert(!UseGC);

    if (obj->isTaggedPointer()) return;
    obj->clearDeallocating();
}



```

这个函数只是做一些判断以及更深层次的函数调用，

```
void objc\_object::sidetable\_clearDeallocating()
{
    SideTable& table = SideTables()\[this\];

    
    
    
    table.lock();
    
    RefcountMap::iterator it = table.refcnts.find(this);
    if (it != table.refcnts.end()) {
        if (it->second & SIDE\_TABLE\_WEAKLY\_REFERENCED) {
            weak\_clear\_no\_lock(&table.weak\_table, (id)this);
        }
        table.refcnts.erase(it);
    }
    table.unlock();
}


```

我们可以看到，在这个函数中，首先取出对象对应的 SideTable 实例，如果这个对象有关联的弱引用，则调用 weak\_clear\_no\_lock 来清除对象的弱引用信息, 我们在来深入一下

```
void 
weak\_clear\_no\_lock(weak\_table\_t \*weak\_table, id referent\_id) 
{
    
    objc\_object \*referent = (objc\_object \*)referent\_id;

    
    weak\_entry\_t \*entry = weak\_entry\_for\_referent(weak\_table, referent);
    if (entry == nil) {
        
        
        return;
    }

    
    weak\_referrer\_t \*referrers;
    size\_t count;
    
    if (entry->out\_of\_line()) {
        
        referrers = entry->referrers;
        count = TABLE\_SIZE(entry);
    } 
    else {
        
        referrers = entry->inline\_referrers;
        count = WEAK\_INLINE\_COUNT;
    }
    
    
    for (size\_t i = 0; i < count; ++i) {
        objc\_object \*\*referrer = referrers\[I\];
        if (referrer) {
            if (\*referrer == referent) {
                \*referrer = nil;
            }
            else if (\*referrer) {
                \_objc\_inform("\_\_weak variable at %p holds %p instead of %p. "
                             "This is probably incorrect use of "
                             "objc\_storeWeak() and objc\_loadWeak(). "
                             "Break on objc\_weak\_error to debug.\\n", 
                             referrer, (void\*)\*referrer, (void\*)referent);
                objc\_weak\_error();
            }
        }
    }
    
    
    weak\_entry\_remove(weak\_table, entry);
}



```

这个函数根据 out\_of\_line 的值，取得对应的记录表，然后根据引用计数对象，将相应的 weak 对象置 nil。最后清除相应的记录表。

通过上面的描述，我们基本能了解一个 weak 引用从生到死的过程。从这个流程可以看出，一个 weak 引用的处理涉及各种查表、添加与删除操作，还是有一定消耗的。所以如果大量使用\_\_weak 变量的话，会对性能造成一定的影响。那么，我们应该在什么时候去使用 weak 呢？《Objective-C 高级编程》给我们的建议是只在避免循环引用的时候使用\_\_weak 修饰符。

好了本文开篇提出的几个问题的答案都能在上面找到。

参看  
[iOS 中 weak 的实现](https://www.jianshu.com/p/dce1bec2199e)  
[iOS 管理对象内存的数据结构以及操作算法 --SideTables、RefcountMap、weak\_table\_t - 一](https://www.jianshu.com/p/ef6d9bf8fe59)  
[细说 weak](https://www.zybuluo.com/qidiandasheng/note/486829)  
[runtime 如何实现 weak 属性](https://dayon.gitbooks.io/-ios/content/chapter8.html)  
[iOS 底层解析 weak 的实现原理（包含 weak 对象的初始化，引用，释放的分析）](https://www.jianshu.com/p/13c4fb1cedea)

"小礼物走一走，来简书关注我"

还没有人赞赏，支持一下

[![](https://upload.jianshu.io/users/upload_avatars/2067180/c0c37aff-e45a-422f-8196-50adefdf7ea4.jpeg?imageMogr2/auto-orient/strip|imageView2/1/w/100/h/100/format/webp)](https://www.jianshu.com/u/42d133a6bae9)

总资产 68 (约 5.10 元) 共写了 7490 字获得 38 个赞共 7 个粉丝

### 被以下专题收入，发现更多相似内容

### 推荐阅读[更多精彩内容](https://www.jianshu.com/)

*   转至元数据结尾创建： 董潇伟，最新修改于： 十二月 23, 2016 转至元数据起始第一章: isa 和 Class 一....
    
*   1.1 什么是自动引用计数 概念：在 LLVM 编译器中设置 ARC(Automaitc Reference Co...
    
    [![](https://upload-images.jianshu.io/upload_images/3262069-3270d20eadf2eb78.png?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)](https://www.jianshu.com/p/c0ddaa71da2c)
*   在 Objective-C 中, 一般为了解决循环引用的问题, 我们会使用 weak 修饰, 使得一方不会持有另一方, 解决循环...
    
*   最近项目中把 rxjava 切换到 2.0 所以相对应的一些都要做出改变 新版本的 独立出来一个 Flowable 来...
    
    [![](https://upload.jianshu.io/users/upload_avatars/266018/69112fa6-2330-4b33-b77e-a016701555cf.jpg?imageMogr2/auto-orient/strip|imageView2/1/w/48/h/48/format/webp)往之\_](https://www.jianshu.com/u/b23a6e7527ab)阅读 3
    
*   班级：软件工程一班 姓名：张宏旭 学号：1505060127 内容：在 Windows 或 Linux 系统中，...
    
    [![](https://upload-images.jianshu.io/upload_images/7924526-72465a7a6a4b9da7.png?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)](https://www.jianshu.com/p/7bd58187ddd6)