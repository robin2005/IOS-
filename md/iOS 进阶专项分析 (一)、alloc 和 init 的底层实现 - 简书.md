\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[www.jianshu.com\](https://www.jianshu.com/p/7b37f5819573)

[![](https://upload.jianshu.io/users/upload_avatars/2679624/1024e12e-a9f0-4985-ba99-d1ebce8e1d7d.jpg?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96/format/webp)](https://www.jianshu.com/u/b53af7c990d8)

0.3722020.06.18 08:50:37 字数 740 阅读 119

引子：

一个经典的面试问题：

```
Objective-C中alloc和init的区别是什么？


```

或者是问：下面这块代码执行后打印的结果

```
BMPerson \* person = \[BMPerson alloc\];
BMPerson \* p1 = \[person init\];
BMPerson \* p2 = \[person init\];
    
NSLog(@"\\n打印地址 %p - %p - %p", person, p1, p2);



```

执行一下结果发现这三个对象的地址是相同的

```
2020-06-17 15:47:20.525517+0800 TextProject\[12922:914030\] 
打印地址 0x6000004b8680 - 0x6000004b8680 - 0x6000004b8680



```

为什么呢？接下来我来详细分析一下 alloc 和 init 底层究竟干了些什么

### 一、alloc 底层探索

点击 alloc 方法在`NSObject.h`文件中只能看到方法声明，从这个声明我们能看出 alloc 这个方法是一个类方法，会返回一个 instancetype 类型的值。但是看不到具体信息了

```
\+ (instancetype)alloc OBJC\_SWIFT\_UNAVAILABLE("use object initializers instead");



```

想要看 alloc 底部具体实现，我们需要去苹果官方文档 open source 里面下载 objc 开放的源码 [苹果开放文档传送门](https://links.jianshu.com/go?to=https%3A%2F%2Fopensource.apple.com%2F)

下载完成打开`objc4-750`部分，直接搜索 `alloc {` 就能找到实现部分 (注意搜索输入的`alloc`和`{`之间有空格)。

```
\+ (id)alloc {
    return \_objc\_rootAlloc(self);
}



```

发现了一个方法 `_objc_rootAlloc` , 继续点击该方法定义，找到 `callAlloc`

```
id
\_objc\_rootAlloc(Class cls)
{
    return callAlloc(cls, false, true);
}



```

继续点击`callAlloc`, 发现该方法的实现代码

```
static ALWAYS\_INLINE id
callAlloc(Class cls, bool checkNil, bool allocWithZone=false)
{
    if (slowpath(checkNil && !cls)) return nil;

#if \_\_OBJC2\_\_
    if (fastpath(!cls->ISA()->hasCustomAWZ())) {
        
        
        
        if (fastpath(cls->canAllocFast())) {
            
            bool dtor = cls->hasCxxDtor();
            id obj = (id)calloc(1, cls->bits.fastInstanceSize());
            if (slowpath(!obj)) return callBadAllocHandler(cls);
            obj->initInstanceIsa(cls, dtor);
            return obj;
        }
        else {
            
            id obj = class\_createInstance(cls, 0);
            if (slowpath(!obj)) return callBadAllocHandler(cls);
            return obj;
        }
    }
#endif

    
    if (allocWithZone) return \[cls allocWithZone:nil\];
    return \[cls alloc\];
}



```

分析这部分代码，抛开优化的部分，我们不难挑出核心代码

```
id obj = class\_createInstance(cls, 0);
return obj;
            


```

继续进入方法 `class_createInstance`查看方法实现

```
static \_\_attribute\_\_((always\_inline)) 
id
\_class\_createInstanceFromZone(Class cls, size\_t extraBytes, void \*zone, 
                              bool cxxConstruct = true, 
                              size\_t \*outAllocatedSize = nil)
{
    if (!cls) return nil;

    assert(cls->isRealized());

    
    bool hasCxxCtor = cls->hasCxxCtor();
    bool hasCxxDtor = cls->hasCxxDtor();
    bool fast = cls->canAllocNonpointer();

    size\_t size = cls->instanceSize(extraBytes);
    if (outAllocatedSize) \*outAllocatedSize = size;

    id obj;
    if (!zone  &&  fast) {
        obj = (id)calloc(1, size);
        if (!obj) return nil;
        obj->initInstanceIsa(cls, hasCxxDtor);
    } 
    else {
        if (zone) {
            obj = (id)malloc\_zone\_calloc ((malloc\_zone\_t \*)zone, 1, size);
        } else {
            obj = (id)calloc(1, size);
        }
        if (!obj) return nil;

        
        
        obj->initIsa(cls);
    }

    if (cxxConstruct && hasCxxCtor) {
        obj = \_objc\_constructOrFree(obj, cls);
    }

    return obj;
}




```

我们发现 `_class_createInstanceFromZone`里面 调用了`cls->instanceSize` `calloc` 以及 `obj->initIsa` , 即 开辟空间以及初始化`isa`, 至此 `obj` 初始化完成，直接把 `obj` 返回出去。

到这里 alloc 的流程已经比较清楚了，alloc 通过我们的类创建了实例对象

### 二、init 底层探索

那么问题来了，既然`alloc`都已经把实例变量创建好了，为什么还需要再调用`init`呢？  
我们接下来在`objc750`源码中搜索 `init {`，发现了`init`这个实例方法调用了 `_objc_rootInit` 这个函数。

```
\- (id)init {
    return \_objc\_rootInit(self);
}



```

分析一下这里的调用，加上我们平时创建实例都是类似于 `[[BMPerson alloc]init]`, 所以此时的`self`就是我们在`alloc`里已经创建完成的实例对象，既然知道了此时的 `self` 是谁了，那么接下来我们继续进入 `_objc_rootInit` 这个函数，

```
id
\_objc\_rootInit(id obj)
{
    
    
    
    
    return obj;
}



```

进入之后我们发现，`_objc_rootInit`这个方法里面其实什么也没做，直接把传入的`self`给返回了出去。接下来我们看注释，发现了苹果设计时候的用意。`init` 就是苹果提供给我们自定义的。这里体现了一个`工厂设计模式`：`父类没有执行，提供给子类自定义`

### 三、总结

1、`alloc`&`init`底层调用流程图：

![](http://upload-images.jianshu.io/upload_images/2679624-9953852028d2c781.png)

alloc&init 底层调用. png

2、`alloc`方法底层已经完成开辟空间和创建类的实例变量了，`init`是专门为`NSObject`预留出来的，体现出`工厂设计模式`，即`父类没有执行，提供给子类自定义`。

[溪浣双鲤的技术摸爬滚打之路](https://www.jianshu.com/p/3fbecd65faae)

"多学习吧，即使不为文凭；多辛苦吧，只要不卖健康"

还没有人赞赏，支持一下

[![](https://upload.jianshu.io/users/upload_avatars/2679624/1024e12e-a9f0-4985-ba99-d1ebce8e1d7d.jpg?imageMogr2/auto-orient/strip|imageView2/1/w/100/h/100/format/webp)](https://www.jianshu.com/u/b53af7c990d8)

[溪浣双鲤](https://www.jianshu.com/u/b53af7c990d8 "溪浣双鲤")那些散碎在笔尖的光阴，寂静欢喜~

总资产 67 (约 4.76 元) 共写了 7.4W 字获得 248 个赞共 123 个粉丝

### 被以下专题收入，发现更多相似内容