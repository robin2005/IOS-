\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[www.jianshu.com\](https://www.jianshu.com/p/89ab04a91cbc)

[![](https://upload.jianshu.io/users/upload_avatars/2251862/4b8d73cd-8776-46d3-8c5e-19c7f71ef0bd.JPG?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96/format/webp)](https://www.jianshu.com/u/1e8432e01e5a)

0.7982020.09.20 20:47:45 字数 1,930 阅读 1,014

> [iOS 底层原理 文章汇总](https://www.jianshu.com/p/412b20d9a0f6)

本文的主要目的是理解`objc_msgSend`的`方法查找`流程

在上一篇文章 [iOS - 底层原理 11：objc\_class 中 cache 原理分析](https://www.jianshu.com/p/3ad9166c02e5)中，分析了 cache 的写入流程，在写入流程之前，还有一个 cache 读取流程，即`objc_msgSend` 和 `cache_getImp`

在分析之前，首先了解什么是 [Runtime](https://links.jianshu.com/go?to=https%3A%2F%2Fdeveloper.apple.com%2Flibrary%2Farchive%2Fdocumentation%2FCocoa%2FConceptual%2FObjCRuntimeGuide%2FIntroduction%2FIntroduction.html%23%2F%2Fapple_ref%2Fdoc%2Fuid%2FTP40008048)

Runtime 介绍
----------

runtime 称为运行时，它区别于编译时

*   `运行时` 是`代码跑起来，被装载到内存中`的过程，如果此时出错，则程序会崩溃，是一个`动态`阶段
    
*   `编译时` 是`源代码翻译成机器能识别的代码`的过程，主要是对语言进行最基本的检查报错，即词法分析、语法分析等，是一个`静态`的阶段
    

`runtime`的`使用`有以下三种方式，其三种实现方法与编译层和底层的关系如图所示

*   通过`OC代码`，例如 `[person sayNB]`
    
*   通过`NSObject方法`，例如`isKindOfClass`
    
*   通过`Runtime API`，例如`class_getInstanceSize`
    

![](http://upload-images.jianshu.io/upload_images/2251862-44cabd96ded9e60f.png)

Runtime 三种方式及底层的关系

其中的`compiler`就是我们了解的`编译器`，即`LLVM`，例如 OC 的`alloc` 对应底层的`objc_alloc`， `runtime system libarary` 就是`底层库`

探索方法的本质
-------

**方法的本质**

在 [iOS - 底层原理 07：isa 与类关联的原理](https://www.jianshu.com/p/7fd6241a7124)文章中，通过 clang 编译的源码，理解了`OC对象的本质`，同样的，使用 clang 编译 main.cpp 文件，通过查看 main 函数中方法调用的实现，如下所示

```
LGPerson \*person = \[LGPerson alloc\];
\[person sayNB\];
\[person sayHello\];


LGPerson \*person = ((LGPerson \*(\*)(id, SEL))(void \*)objc\_msgSend)((id)objc\_getClass("LGPerson"), sel\_registerName("alloc"));
((void (\*)(id, SEL))(void \*)objc\_msgSend)((id)person, sel\_registerName("sayNB"));
((void (\*)(id, SEL))(void \*)objc\_msgSend)((id)person, sel\_registerName("sayHello"));


```

通过上述代码可以看出，`方法的本质`就是`objc_msgSend消息发送`

为了验证，通过`objc_msgSend`方法来完成`[person sayNB]`的调用，查看其打印是否是一致

> 注：  
> 1、直接调用`objc_msgSend`，需要导入头文件`#import <objc/message.h>`  
> 2、需要将 target --> Build Setting --> 搜索`msg` -- 将`enable strict checking of obc_msgSend calls`由`YES` 改为`NO`，将严厉的检查机制关掉，否则 objc\_msgSend 的参数会报错

```
LGPerson \*person = \[LGPerson alloc\];   
objc\_msgSend(person,sel\_registerName("sayNB"));
\[person sayNB\];


```

其打印结果如下，发现是一致的，所以 `[person sayNB]`等价于`objc_msgSend(person,sel_registerName("sayNB"))`  

![](http://upload-images.jianshu.io/upload_images/2251862-bcc817b42b6dec63.png)

objc\_msgSend 与方法调用打印结果

**对象方法调用 - 实际执行是父类的实现**  
除了验证，我们还可以尝试让 person 的调用执行父类中实现，通过`objc_msgSendSuper`实现

*   定义两个类：LGPerson 和 LGTeacher，父类中实现了 sayHello 方法
    
    ![](http://upload-images.jianshu.io/upload_images/2251862-d3b3396cff60d732.png)
    
    自定义类
    

*   main 中的调用

```
LGPerson \*person = \[LGPerson alloc\];
LGTeacher \*teacher = \[LGTeacher alloc\];
\[person sayHello\];

struct objc\_super lgsuper;
lgsuper.receiver = person; 
lgsuper.super\_class = \[LGTeacher class\]; 
    

objc\_msgSendSuper(&lgsuper, sel\_registerName("sayHello"));


```

`objc_msgSendSuper`方法中有两个参数`（结构体，sel）`，其结构体类型是`objc_super`定义的结构体对象，且需要指定`receiver` 和 `super_class`两个属性，源码实现 & 定义如下

*   `objc_msgSendSuper` 方法参数  
    
    ![](http://upload-images.jianshu.io/upload_images/2251862-bff23f047a769625.png)
    
    objc\_msgSendSuper 方法对应的参数
    
*   `objc_super`源码定义  
    
    ![](http://upload-images.jianshu.io/upload_images/2251862-f207ac2e247b8029.png)
    
    objc\_super 源码
    

打印结果如下

![](http://upload-images.jianshu.io/upload_images/2251862-55e9ace3991b040c.png)

子类方法调用转为执行父类的实现的打印结果

发现不论是`[person sayHello]`还是`objc_msgSendSuper`都执行的是`父类`中`sayHello`的实现，所以这里，我们可以作一个猜测：`方法调用，首先是在类中查找，如果类中没有找到，会到类的父类中查找。`

带着我们的猜测，下面我们来探索 objc\_msgSend 的源码实现

objc\_msgSend 快速查找流程分析
----------------------

在 objc4-781 源码中，搜索`objc_msgSend`，由于我们日常开发的都是架构是 arm64，所以需要在`arm64.s`后缀的文件中查找`objc_msgSend`源码实现，发现是`汇编实现`，其汇编整体执行的流程图如下  

![](http://upload-images.jianshu.io/upload_images/2251862-aecc50c741fa734c.png)

方法快速查找流程

#### objc\_msgSend 汇编源码

`objc_msgSend`是消息发送的源码的入口，其使用汇编实现的，`_objc_msgSend`源码实现如下

```
ENTRY \_objc\_msgSend 

    UNWIND \_objc\_msgSend, NoFrame 
    

    cmp p0, #0          

#if SUPPORT\_TAGGED\_POINTERS
    b.le    LNilOrTagged        
#else

    b.eq    LReturnZero 
#endif 


    ldr p13, \[x0\]       

    GetClassFromIsa\_p16 p13     
LGetIsaDone:
    

    CacheLookup NORMAL, \_objc\_msgSend

#if SUPPORT\_TAGGED\_POINTERS
LNilOrTagged:

    b.eq    LReturnZero     

    
    adrp    x10, \_objc\_debug\_taggedpointer\_classes@PAGE
    add x10, x10, \_objc\_debug\_taggedpointer\_classes@PAGEOFF
    ubfx    x11, x0, #60, #4
    ldr x16, \[x10, x11, LSL #3\]
    adrp    x10, \_OBJC\_CLASS\_$\_\_\_NSUnrecognizedTaggedPointer@PAGE
    add x10, x10, \_OBJC\_CLASS\_$\_\_\_NSUnrecognizedTaggedPointer@PAGEOFF
    cmp x10, x16
    b.ne    LGetIsaDone

    
    adrp    x10, \_objc\_debug\_taggedpointer\_ext\_classes@PAGE
    add x10, x10, \_objc\_debug\_taggedpointer\_ext\_classes@PAGEOFF
    ubfx    x11, x0, #52, #8
    ldr x16, \[x10, x11, LSL #3\]
    b   LGetIsaDone

#endif

LReturnZero:
    
    mov x1, #0
    movi    d0, #0
    movi    d1, #0
    movi    d2, #0
    movi    d3, #0
    ret

    END\_ENTRY \_objc\_msgSend


```

主要有以下几步

*   【第一步】判断`objc_msgSend`方法的第一个参数`receiver`是否为空
    *   如果支持`tagged pointer`，跳转至`LNilOrTagged`，
        *   如果`小对象`为空，则直接返回空，即`LReturnZero`
        *   如果`小对象`不为空，则处理小对象的`isa`，走到【第二步】
    *   如果即不是小对象，`receiver`也不为空，有以下两步
        *   从`receiver`中取出`isa`存入`p13`寄存器，
        *   通过 `GetClassFromIsa_p16`中，`arm64`架构下通过 `isa & ISA_MASK` 获取`shiftcls`位域的类信息，即`class`，`GetClassFromIsa_p16`的汇编实现如下，然后走到【第二步】

```
.macro GetClassFromIsa\_p16 

#if SUPPORT\_INDEXED\_ISA 
    

    mov p16, $0         
    tbz p16, #ISA\_INDEX\_IS\_NPI\_BIT, 1f  
    

    adrp    x10, \_objc\_indexed\_classes@PAGE 

    add x10, x10, \_objc\_indexed\_classes@PAGEOFF

    ubfx    p16, p16, #ISA\_INDEX\_SHIFT, #ISA\_INDEX\_BITS  
    ldr p16, \[x10, p16, UXTP #PTRSHIFT\] 
1:


#elif \_\_LP64\_\_ 
    

    and p16, $0, #ISA\_MASK 

#else
    
    mov p16, $0

#endif

.endmacro


```

*   【第二步】获取 isa 完毕，进入慢速查找流程`CacheLookup NORMAL`

#### CacheLookup 缓存查找汇编源码

```
.macro CacheLookup 
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
LLookupStart$1:



    ldr p11, \[x16, #CACHE\]              

#if CACHE\_MASK\_STORAGE == CACHE\_MASK\_STORAGE\_HIGH\_16 

    and p10, p11, #0x0000ffffffffffff   
    

    and p12, p1, p11, LSR #48       


#elif CACHE\_MASK\_STORAGE == CACHE\_MASK\_STORAGE\_LOW\_4 
    and p10, p11, #~0xf         
    and p11, p11, #0xf          
    mov p12, #0xffff
    lsr p11, p12, p11               
    and p12, p1, p11                
#else
#error Unsupported cache mask storage for ARM64.
#endif



    add p12, p10, p12, LSL #(1+PTRSHIFT)   
                     
                     

    ldp p17, p9, \[x12\]      
    

1:  cmp p9, p1          

    b.ne    2f          

    CacheHit $0         
    
2:  

    CheckMiss $0            

    cmp p12, p10        

    b.eq    3f 

    ldp p17, p9, \[x12, #-BUCKET\_SIZE\]!  

    b   1b          

3:  
#if CACHE\_MASK\_STORAGE == CACHE\_MASK\_STORAGE\_HIGH\_16


    add p12, p12, p11, LSR #(48 - (1+PTRSHIFT)) 
                    
#elif CACHE\_MASK\_STORAGE == CACHE\_MASK\_STORAGE\_LOW\_4
    add p12, p12, p11, LSL #(1+PTRSHIFT)
                    
#else
#error Unsupported cache mask storage for ARM64.
#endif

    
    


    ldp p17, p9, \[x12\]      
    

1:  cmp p9, p1          

    b.ne    2f          

    CacheHit $0         
    
2:  

    CheckMiss $0            

    cmp p12, p10        
    b.eq    3f 

    ldp p17, p9, \[x12, #-BUCKET\_SIZE\]!  

    b   1b          

LLookupEnd$1:
LLookupRecover$1:
3:  


    JumpMiss $0 
.endmacro


.macro CacheHit
.if $0 == NORMAL
    TailCallCachedImp x17, x12, x1, x16 
.elseif $0 == GETIMP
    mov p0, p17
    cbz p0, 9f          
    AuthAndResignAsIMP x0, x12, x1, x16 
9:  ret             
.elseif $0 == LOOKUP
    
    
    AuthAndResignAsIMP x17, x12, x1, x16    
    ret             
.else
.abort oops
.endif
.endmacro

.macro CheckMiss
    
.if $0 == GETIMP 

    cbz p9, LGetImpMiss
.elseif $0 == NORMAL 

    cbz p9, \_\_objc\_msgSend\_uncached
.elseif $0 == LOOKUP 

    cbz p9, \_\_objc\_msgLookup\_uncached
.else
.abort oops
.endif
.endmacro

.macro JumpMiss
.if $0 == GETIMP
    b   LGetImpMiss
.elseif $0 == NORMAL
    b   \_\_objc\_msgSend\_uncached
.elseif $0 == LOOKUP
    b   \_\_objc\_msgLookup\_uncached
.else
.abort oops
.endif
.endmacro


```

主要分为以下几步

*   【第一步】通过`cache`首地址平移`16`字节（因为在 objc\_class 中，`首地址`距离`cache`正好`16`字节，即`isa首地址` 占`8`字节，`superClass`占`8`字节），获取`cahce`，cache 中`高16位存mask`，`低48位存buckets`，即`p11 = cache`
    
*   【第二步】从 cache 中分别取出 buckets 和 mask，并由 mask 根据哈希算法计算出哈希下标
    
    *   通过`cache`和`掩码（即0x0000ffffffffffff）`的 `&` 运算，将`高16位mask抹零`，得到 buckets 指针地址，即`p10 = buckets`
        
    *   将`cache`右移`48`位，得到`mask`，即`p11 = mask`
        
    *   将`objc_msgSend`的参数`p1`（即第二个参数\_cmd）`& msak`, 通过`哈希算法`，得到需要查找存储 sel-imp 的`bucket下标index`，即`p12 = index = _cmd & mask`, 为什么通过这种方式呢？因为在`存储sel-imp`时，也是通过同样`哈希算法计算哈希下标进行存储`，所以`读取`也需要通过`同样的方式读取`，如下所示  
        
        ![](http://upload-images.jianshu.io/upload_images/2251862-2f9f02c36962d402.png)
        
        cache\_t 存储 sel-imp 时计算哈希下标的哈希算法源码
        

*   【第三步】根据所得的`哈希下标index` 和 `buckets首地址`，取出哈希下标对应的`bucket`
    
    *   其中`PTRSHIFT`等于 3，左移 4 位（即 2^4 = 16 字节）的目的是计算出一个`bucket`实际占用的大小, 结构体`bucket_t`中`sel`占`8`字节，`imp`占`8`字节
        
    *   根据计算的哈希下标`index 乘以` 单个`bucket占用的内存大小`，得到`buckets`首地址在`实际内存`中的`偏移量`
        
    *   通过`首地址 + 实际偏移量`，获取哈希下标 index 对应的`bucket`
        
*   【第四步】根据获取的`bucket`，取出其中的`imp`存入`p17`，即`p17 = imp`，取出`sel`存入`p9`，即`p9 = sel`
    
*   【第五步】第一次递归循环
    
    *   比较获取的`bucket`中`sel` 与 `objc_msgSend`的第二个参数的`_cmd(即p1)`是否相等
        
    *   如果`相等`，则直接跳转至`CacheHit`，即`缓存命中`，返回`imp`
        
    *   如果不相等，有以下两种情况
        
        *   如果一直都找不到，直接跳转至`CheckMiss`，因为`$0`是`normal`，会跳转至`__objc_msgSend_uncached`，即进入`慢速查找流程`
            
        *   如果`根据index获取的bucket` 等于 `buckets的第一个元素`，则`人为`的将`当前bucket设置为buckets的最后一个元素`（通过`buckets首地址+mask右移44位`（等同于左移 4 位）直接`定位到bucker的最后一个元素`），然后继续进行递归循环（`第一个`递归循环嵌套`第二个`递归循环），即【第六步】
            
        *   如果`当前bucket`不等于`buckets的第一个元素`，则继续`向前查找`，进入`第一次递归循环`
            
*   【第六步】第二次递归循环：重复【第五步】的操作，与【第五步】中`唯一区别`是，如果`当前的bucket还是等于 buckets的第一个元素`，则直接跳转至`JumpMiss`，此时的`$0`是`normal`，也是直接跳转至`__objc_msgSend_uncached`，即进入`慢速查找流程`
    

以下是整个`快速查找`过程`值的变化`过程  

![](http://upload-images.jianshu.io/upload_images/2251862-b7345913a236c9bc.png)

值变化过程

"小礼物走一走，来简书关注我"

还没有人赞赏，支持一下

[![](https://upload.jianshu.io/users/upload_avatars/2251862/4b8d73cd-8776-46d3-8c5e-19c7f71ef0bd.JPG?imageMogr2/auto-orient/strip|imageView2/1/w/100/h/100/format/webp)](https://www.jianshu.com/u/1e8432e01e5a)

总资产 15 (约 1.34 元) 共写了 14.3W 字获得 534 个赞共 355 个粉丝

### 被以下专题收入，发现更多相似内容