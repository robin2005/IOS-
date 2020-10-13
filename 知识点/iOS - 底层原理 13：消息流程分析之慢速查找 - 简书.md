\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[www.jianshu.com\](https://www.jianshu.com/p/f7d9f6d86145)

[![](https://upload.jianshu.io/users/upload_avatars/2251862/4b8d73cd-8776-46d3-8c5e-19c7f71ef0bd.JPG?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96/format/webp)](https://www.jianshu.com/u/1e8432e01e5a)

0.6332020.09.23 01:35:15 字数 1,549 阅读 784

> [iOS 底层原理 文章汇总](https://www.jianshu.com/p/412b20d9a0f6)

在上一篇 [iOS - 底层原理 12：objc\_msgSend 流程分析之快速查找](https://www.jianshu.com/p/89ab04a91cbc)文章中，我们分析了`快速查找流程`，如果快速查不到，则需要进入`慢速查找流程`，以下是慢速查找的`分析过程`

objc\_msgSend 慢速查找流程分析
----------------------

### 慢速查找 - 汇编部分

在快速查找流程中，如果没有找到方法实现，无论是走到`CheckMiss`还是`JumpMiss`，最终都会走到`__objc_msgSend_uncached`汇编函数

*   在`objc-msg-arm64.s`文件中查找`__objc_msgSend_uncached`的汇编实现，其中的核心是`MethodTableLookup（即查询方法列表）`，其源码如下

```
STATIC\_ENTRY \_\_objc\_msgSend\_uncached
UNWIND \_\_objc\_msgSend\_uncached, FrameWithNoSaves



    
MethodTableLookup 
TailCallFunctionPointer x17

END\_ENTRY \_\_objc\_msgSend\_uncached


```

*   搜索`MethodTableLookup`的汇编实现，其中的核心是`_lookUpImpOrForward`，汇编源码实现如下

```
.macro MethodTableLookup
    
    
    SignLR
    stp fp, lr, \[sp, #-16\]!
    mov fp, sp

    
    sub sp, sp, #(10\*8 + 8\*16)
    stp q0, q1, \[sp, #(0\*16)\]
    stp q2, q3, \[sp, #(2\*16)\]
    stp q4, q5, \[sp, #(4\*16)\]
    stp q6, q7, \[sp, #(6\*16)\]
    stp x0, x1, \[sp, #(8\*16+0\*8)\]
    stp x2, x3, \[sp, #(8\*16+2\*8)\]
    stp x4, x5, \[sp, #(8\*16+4\*8)\]
    stp x6, x7, \[sp, #(8\*16+6\*8)\]
    str x8,     \[sp, #(8\*16+8\*8)\]

    
    
    mov x2, x16
    mov x3, #3
    bl  \_lookUpImpOrForward 

    
    mov x17, x0
    
    
    ldp q0, q1, \[sp, #(0\*16)\]
    ldp q2, q3, \[sp, #(2\*16)\]
    ldp q4, q5, \[sp, #(4\*16)\]
    ldp q6, q7, \[sp, #(6\*16)\]
    ldp x0, x1, \[sp, #(8\*16+0\*8)\]
    ldp x2, x3, \[sp, #(8\*16+2\*8)\]
    ldp x4, x5, \[sp, #(8\*16+4\*8)\]
    ldp x6, x7, \[sp, #(8\*16+6\*8)\]
    ldr x8,     \[sp, #(8\*16+8\*8)\]

    mov sp, fp
    ldp fp, lr, \[sp\], #16
    AuthenticateLR

.endmacro


```

**验证**

上述汇编的过程，可以`通过汇编调`试来`验证`

*   在`main`中，例如`[person sayHello]`对象方法调用处加一个断点，并且`开启汇编调试【Debug -- Debug worlflow -- 勾选Always show Disassembly】`，运行程序  
    
    ![](http://upload-images.jianshu.io/upload_images/2251862-c1a4ff3dae3e9a13.jpg)
    
    汇编验证 - 1
    
*   汇编中`objc_msgSend`加一个断点，执行断住，按住`control + stepinto`，进入`objc_msgSend`的汇编  
    
    ![](http://upload-images.jianshu.io/upload_images/2251862-65d17f72df6d33cc.jpg)
    
    汇编验证 - 2
    
*   在`_objc_msgSend_uncached`加一个断点，执行断住，按住`control + stepinto`，进入汇编  
    
    ![](http://upload-images.jianshu.io/upload_images/2251862-8f69da9a9ec9c911.jpg)
    
    汇编验证 - 3
    
    从上可以看出最后走到的就是`lookUpImpOrForward`，此时并不是汇编实现

> 注：  
> 1、`C/C++`中调用 `汇编` ，去`查找汇编`时，C/C++ 调用的`方法`需要`多加一个下划线`  
> 2、`汇编` 中调用 `C/C++方法`时，去`查找C/C++`方法，需要将汇编调用的`方法去掉一个下划线`

### 慢速查找 - C/C++ 部分

*   根据汇编部分的提示，全局续搜索`lookUpImpOrForward`，最后在`objc-runtime-new.mm`文件中找到了源码实现，这是一个`c实现的函数`

```
IMP lookUpImpOrForward(id inst, SEL sel, Class cls, int behavior)
{
    
    const IMP forward\_imp = (IMP)\_objc\_msgForward\_impcache; 
    IMP imp = nil;
    Class curClass;

    runtimeLock.assertUnlocked();

    
    
    if (fastpath(behavior & LOOKUP\_CACHE)) { 
        imp = cache\_getImp(cls, sel);
        if (imp) goto done\_nolock;
    }
    
    
    runtimeLock.lock();
    
    
    checkIsKnownClass(cls); 
    
    
    if (slowpath(!cls->isRealized())) { 
        cls = realizeClassMaybeSwiftAndLeaveLocked(cls, runtimeLock);
    }

    
    if (slowpath((behavior & LOOKUP\_INITIALIZE) && !cls->isInitialized())) { 
        cls = initializeAndLeaveLocked(cls, inst, runtimeLock);
    }

    runtimeLock.assertLocked();
    curClass = cls;

    
    
    
    
    for (unsigned attempts = unreasonableClassCount();;) { 
        
        Method meth = getMethodNoSuper\_nolock(curClass, sel);
        if (meth) {
            imp = meth->imp;
            goto done;
        }
        
        if (slowpath((curClass = curClass->superclass) == nil)) {
            
            imp = forward\_imp;
            break;
        }

        
        if (slowpath(--attempts == 0)) {
            \_objc\_fatal("Memory corruption in class list.");
        }

        
        imp = cache\_getImp(curClass, sel);
        if (slowpath(imp == forward\_imp)) { 
            
            break;
        }
        if (fastpath(imp)) {
            
            goto done;
        }
    }

    

    if (slowpath(behavior & LOOKUP\_RESOLVER)) {
        
        behavior ^= LOOKUP\_RESOLVER; 
        return resolveMethod\_locked(inst, sel, cls, behavior);
    }

 done:
    
    log\_and\_fill\_cache(cls, imp, sel, inst, curClass); 
    
    runtimeLock.unlock();
 done\_nolock:
    if (slowpath((behavior & LOOKUP\_NIL) && imp == forward\_imp)) {
        return nil;
    }
    return imp;
}


```

其整体的慢速查找流程如图所示

![](http://upload-images.jianshu.io/upload_images/2251862-fd926e375c27a218.png)

慢速查找流程

主要有以下几步：

*   【第一步】`cache`缓存中进行查找，即`快速查找`，找到则直接返回`imp`，反之，则进入【第二步】
    
*   【第二步】判断`cls`
    
    *   是否是`已知类`，如果不是，则`报错`
        
    *   类是否`实现`，如果没有，则需要先实现，确定其父类链，此时实例化的目的是为了确定父类链、ro、以及 rw 等，方法后续数据的读取以及查找的循环
        
    *   是否`初始化`，如果没有，则初始化
        
*   【第三步】`for循环`，按照`类继承链 或者 元类继承链`的顺序查找
    
    *   当前 cls 的`方法列表`中使用`二分查找算法`查找方法，如果找到，则`进入cache写入流程`（在 [iOS - 底层原理 11：objc\_class 中 cache 原理分析](https://www.jianshu.com/p/3ad9166c02e5)文章中已经详述过），并`返回imp`，如果`没有找到`，则返回`nil`
        
    *   `当前cls`被赋值`为父类`，如果父类`等于nil`，则`imp = 消息转发，并终止递归`，进入【第四步】
        
    *   如果`父类链`中存在`循环`，则报错，`终止循环`
        
    *   `父类缓存`中查找方法
        
        *   如果`未找到`，则直接返回`nil`，继续`循环查找`
            
        *   如果`找到`，则直接`返回imp`，执行`cache写入流程`
            
*   【第四步】`判断`是否执行过`动态方法解析`
    
    *   ，如果`没有`，执行`动态方法解析`
        
    *   如果`执行过`一次动态方法解析，则走到`消息转发流程`
        

以上就是方法的`慢速查找流程`，下面在分别详细解释`二分查找原理` 以及 `父类缓存查找`详细步骤

#### getMethodNoSuper\_nolock 方法：二分查找方法列表

`查找方法列表`的`流程`如下所示，  

![](http://upload-images.jianshu.io/upload_images/2251862-062150549fa3aef8.png)

二分方法列表查找流程

其二分查找核心的源码实现如下

```
ALWAYS\_INLINE static method\_t \*
findMethodInSortedMethodList(SEL key, const method\_list\_t \*list)
{
    ASSERT(list);

    const method\_t \* const first = &list->first;
    const method\_t \*base = first;
    const method\_t \*probe;
    uintptr\_t keyValue = (uintptr\_t)key; 
    uint32\_t count;
    
    for (count = list->count; count != 0; count >>= 1) {
        
        probe = base + (count >> 1); 
        
        uintptr\_t probeValue = (uintptr\_t)probe->name;
        
        
        if (keyValue == probeValue) { 
            
            while (probe > first && keyValue == (uintptr\_t)probe\[-1\].name) {
                
                
                probe--;
            }
            return (method\_t \*)probe;
        }
        
        
        if (keyValue > probeValue) { 
            base = probe + 1;
            count--;
        }
    }
    
    return nil;
}


```

`算法原理`简述为：从第一次查找开始，每次都取`中间位置`，与想查找的`key的value值`作比较，如果`相等`，则需要`排除分类方法`，然后将查询到的位置的方法实现返回，如果`不相等`，则需要`继续二分查找`，如果循环至`count = 0`还是`没有找到`，则直接返回`nil`，如下所示：  

![](http://upload-images.jianshu.io/upload_images/2251862-3e3b988e3e02b757.png)

二分查找方法列表原理

以查找`LGPerson`类的`say666实例方法`为例，其二分查找过程如下  

![](http://upload-images.jianshu.io/upload_images/2251862-aa435b76daac0ba6.png)

举例说明二分查找过程

#### cache\_getImp 方法：父类缓存查找

`cache_getImp`方法是通过`汇编_cache_getImp实现`，传入的`$0` 是 `GETIMP`，如下所示  

![](http://upload-images.jianshu.io/upload_images/2251862-ef4b179ef44f5f88.png)

父类缓存查找流程

*   如果`父类缓存`中找到了方法实现，则跳转至`CacheHit`即命中，则直接`返回imp`
    
*   如果在`父类缓存`中，`没有找到`方法实现，则跳转至`CheckMiss` 或者 `JumpMiss`，通过判断`$0` 跳转至`LGetImpMiss`，直接返回`nil`
    

#### 总结

*   对于`对象方法（即实例方法）`，即在`类中查找`，其慢速查找的`父类链`是：`类--父类--根类--nil`
    
*   对于`类方法`，即在`元类中查找`，其慢速查找的`父类链`是：`元类--根元类--根类--nil`
    
*   如果`快速查找、慢速查找`也没有找到方法实现，则尝试`动态方法决议`
    
*   如果`动态方法决议`仍然没有找到，则进行`消息转发`
    

### 常见方法未实现报错源码

如果在快速查找、慢速查找、方法解析流程中，均没有找到实现，则使用消息转发，其流程如下

![](http://upload-images.jianshu.io/upload_images/2251862-093c2f8819a89a88.png)

消息转发流程

消息转发会实现

*   其中\_objc\_msgForward\_impcache 是汇编实现，会跳转至`__objc_msgForward`，其核心是`__objc_forward_handler`

```
STATIC\_ENTRY \_\_objc\_msgForward\_impcache


b   \_\_objc\_msgForward

END\_ENTRY \_\_objc\_msgForward\_impcache


ENTRY \_\_objc\_msgForward

adrp    x17, \_\_objc\_forward\_handler@PAGE
ldr p17, \[x17, \_\_objc\_forward\_handler@PAGEOFF\]
TailCallFunctionPointer x17
    
END\_ENTRY \_\_objc\_msgForward


```

*   汇编实现中查找`__objc_forward_handler`，并没有找到，在源码中去掉一个下划线进行全局搜索`_objc_forward_handler`，有如下实现，本质是调用的`objc_defaultForwardHandler`方法

```
\_\_attribute\_\_((noreturn, cold)) void
objc\_defaultForwardHandler(id self, SEL sel)
{
    \_objc\_fatal("%c\[%s %s\]: unrecognized selector sent to instance %p "
                "(no message forward handler is installed)", 
                class\_isMetaClass(object\_getClass(self)) ? '+' : '-', 
                object\_getClassName(self), sel\_getName(sel), self);
}
void \*\_objc\_forward\_handler = (void\*)objc\_defaultForwardHandler;


```

看着`objc_defaultForwardHandler`有没有很眼熟，这就是我们在日常开发中最常见的错误：`没有实现函数，运行程序，崩溃时报的错误提示`。

"小礼物走一走，来简书关注我"

还没有人赞赏，支持一下

[![](https://upload.jianshu.io/users/upload_avatars/2251862/4b8d73cd-8776-46d3-8c5e-19c7f71ef0bd.JPG?imageMogr2/auto-orient/strip|imageView2/1/w/100/h/100/format/webp)](https://www.jianshu.com/u/1e8432e01e5a)

总资产 15 (约 1.34 元) 共写了 14.3W 字获得 534 个赞共 355 个粉丝

### 被以下专题收入，发现更多相似内容