\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[www.jianshu.com\](https://www.jianshu.com/p/b838f04a9249)

[![](https://upload.jianshu.io/users/upload_avatars/2251862/4b8d73cd-8776-46d3-8c5e-19c7f71ef0bd.JPG?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96/format/webp)](https://www.jianshu.com/u/1e8432e01e5a)

0.4932020.09.25 15:00:29 字数 4,365 阅读 485

> [iOS 底层原理 文章汇总](https://www.jianshu.com/p/412b20d9a0f6)

引子
--

在前面两篇文章 [iOS - 底层原理 12：objc\_msgSend 流程分析之快速查找](https://www.jianshu.com/p/89ab04a91cbc)和 [iOS - 底层原理 13：objc\_msgSend 流程分析之慢速查找](https://www.jianshu.com/p/f7d9f6d86145)中，分别分析了`objc_msgSend`的`快速查找`和`慢速查找`，在这两种都没找到方法实现的情况下，苹果给了两个建议

*   `动态方法决议`：慢速查找流程未找到后，会执行一次动态方法决议
*   `消息转发`：如果动态方法决议仍然没有找到实现，则进行消息转发

如果这两个建议都没有做任何操作，就会报我们日常开发中常见的`方法未实现`的`崩溃报错`，其步骤如下

*   定义`LGPerson`类，其中`say666`实例方法 和 `sayNB`类方法均`没有实现`  
    
    ![](http://upload-images.jianshu.io/upload_images/2251862-912e65ae6c9f13f7.jpg)
    
    自定义 LGPerson 类
    
*   main 中 分别调用`LGPerson`的`实例方法say666` 和`类方法sayNB`，运行程序，均会`报错`，提示`方法未实现`，如下所示
    
    *   调用实例方法 say666 的报错结果
        
        ![](http://upload-images.jianshu.io/upload_images/2251862-bc25272a2fa11946.png)
        
        实例方法报错
        
    *   调用类方法 sayNB 的报错结果
        
        ![](http://upload-images.jianshu.io/upload_images/2251862-d87c58f82b22a171.png)
        
        类方法报错
        

### 方法未实现报错源码

根据`慢速查找`的源码，我们发现，其报错最后都是走到`__objc_msgForward_impcache`方法，以下是报错流程的源码

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

下面，我们来讲讲如何在崩溃前，如何操作，可以防止方法未实现的崩溃。

三次方法查找的挽救机会
-----------

根据苹果的两个建议，我们一共有三次挽救的机会：

*   【第一次机会】`动态方法决议`
    
*   消息转发流程
    
    *   【第二次机会】`快速转发`
    *   【第三次机会】`慢速转发`

### 【第一次机会】动态方法决议

在`慢速查找`流程`未找到`方法实现时，首先会尝试一次`动态方法决议`，其源码实现如下：

```
static NEVER\_INLINE IMP
resolveMethod\_locked(id inst, SEL sel, Class cls, int behavior)
{
    runtimeLock.assertLocked();
    ASSERT(cls->isRealized());

    runtimeLock.unlock();
    
    if (! cls->isMetaClass()) { 
        
        resolveInstanceMethod(inst, sel, cls);
    } 
    else {
        
        
        resolveClassMethod(inst, sel, cls);
        
        if (!lookUpImpOrNil(inst, sel, cls)) { 
            resolveInstanceMethod(inst, sel, cls);
        }
    }

    
    
    
    return lookUpImpOrForward(inst, sel, cls, behavior | LOOKUP\_CACHE);
}


```

主要分为以下几步

*   判断类是否是元类
    *   如果是`类`，执行`实例方法`的动态方法决议`resolveInstanceMethod`
    *   如果是`元类`，执行`类方法`的动态方法决议`resolveClassMethod`，如果在元类中`没有找到`或者为`空`，则在`元类`的`实例方法`的动态方法决议`resolveInstanceMethod`中查找，主要是因为`类方法在元类中是实例方法`，所以还需要查找元类中实例方法的动态方法决议
*   如果`动态方法决议`中，将其`实现指向了其他方法`，则继续`查找指定的imp`，即继续慢速查找`lookUpImpOrForward`流程

其流程如下

![](http://upload-images.jianshu.io/upload_images/2251862-b949f3712d67ce08.png)

方法解析流程

#### 实例方法

针对`实例方法`调用，在快速 - 慢速查找均没有找到`实例方法`的实现时，我们有一次挽救的机会，即尝试一次`动态方法决议`，由于是`实例方法`，所以会走到`resolveInstanceMethod`方法，其源码如下

```
static void resolveInstanceMethod(id inst, SEL sel, Class cls)
{
    runtimeLock.assertUnlocked();
    ASSERT(cls->isRealized());
    SEL resolve\_sel = @selector(resolveInstanceMethod:);
    
    
    if (!lookUpImpOrNil(cls, resolve\_sel, cls->ISA())) {
        
        return;
    }

    BOOL (\*msg)(Class, SEL, SEL) = (typeof(msg))objc\_msgSend;
    bool resolved = msg(cls, resolve\_sel, sel); 

    
    
    
    IMP imp = lookUpImpOrNil(inst, sel, cls);

    if (resolved  &&  PrintResolving) {
        if (imp) {
            \_objc\_inform("RESOLVE: method %c\[%s %s\] "
                         "dynamically resolved to %p", 
                         cls->isMetaClass() ? '+' : '-', 
                         cls->nameForLogging(), sel\_getName(sel), imp);
        }
        else {
            
            \_objc\_inform("RESOLVE: +\[%s resolveInstanceMethod:%s\] returned YES"
                         ", but no new implementation of %c\[%s %s\] was found",
                         cls->nameForLogging(), sel\_getName(sel), 
                         cls->isMetaClass() ? '+' : '-', 
                         cls->nameForLogging(), sel\_getName(sel));
        }
    }
}


```

主要分为以下几个步骤：

*   在`发送resolveInstanceMethod消息`前，需要查找 cls`类`中是否有该方法的`实现`，即通过`lookUpImpOrNil`方法又会进入`lookUpImpOrForward`慢速查找流程查找`resolveInstanceMethod`方法
    *   如果没有，则直接返回
    *   如果有，则发送`resolveInstanceMethod`消息
*   再次慢速查找实例方法的实现，即通过`lookUpImpOrNil`方法又会进入`lookUpImpOrForward`慢速查找流程查找`实例方法`

**崩溃修改**

所以，针对`实例方法say666`未实现的报错崩溃，可以通过在`类`中`重写``resolveInstanceMethod`类方法，并将其指向其他方法的实现，即在`LGPerson`中重写`resolveInstanceMethod类方法`，将`实例方法say666`的实现指向`sayMaster`方法实现，如下所示

```
\+ (BOOL)resolveInstanceMethod:(SEL)sel{
    if (sel == @selector(say666)) {
        NSLog(@"%@ 来了", NSStringFromSelector(sel));
        
        IMP imp = class\_getMethodImplementation(self, @selector(sayMaster));
        
        Method sayMethod  = class\_getInstanceMethod(self, @selector(sayMaster));
        
        const char \*type = method\_getTypeEncoding(sayMethod);
        
        return class\_addMethod(self, sel, imp, type);
    }
    
    return \[super resolveInstanceMethod:sel\];
}


```

重新运行，其打印结果如下

![](http://upload-images.jianshu.io/upload_images/2251862-90c4acf2b311431a.png)

打印结果

从结果中可以发现，`resolveInstanceMethod`动态决议方法中 “来了” 打印了两次，这是为什么呢？通过堆栈信息可以看出  

![](http://upload-images.jianshu.io/upload_images/2251862-913e6e5795f1e0b0.png)

堆栈信息

*   【第一次动态决议】第一次的 “来了” 是在查找`say666`方法时会进入`动态方法决议`
*   【第二次动态决议】第二次 “来了” 是在慢速转发流程中调用了`CoreFoundation`框架中的`NSObject(NSObject) methodSignatureForSelector:`后，会再次`进入动态决议`

> 注：详细的分析流程请看文末的问题探索

#### 类方法

针对`类方法`，与实例方法类似，同样可以通过重写`resolveClassMethod`类方法来解决前文的崩溃问题，即在`LGPerson`类中重写该方法，并将`sayNB`类方法的实现`指向类方法lgClassMethod`

```
\+ (BOOL)resolveClassMethod:(SEL)sel{
    
    if (sel == @selector(sayNB)) {
        NSLog(@"%@ 来了", NSStringFromSelector(sel));
        
        IMP imp = class\_getMethodImplementation(objc\_getMetaClass("LGPerson"), @selector(lgClassMethod));
        Method lgClassMethod  = class\_getInstanceMethod(objc\_getMetaClass("LGPerson"), @selector(lgClassMethod));
        const char \*type = method\_getTypeEncoding(lgClassMethod);
        return class\_addMethod(objc\_getMetaClass("LGPerson"), sel, imp, type);
    }
    
    return \[super resolveClassMethod:sel\];
}


```

> `resolveClassMethod`类方法的重写需要注意一点，传入的`cls`不再是类，而`是元类`，可以通过`objc_getMetaClass`方法`获取类的元类`，原因是因为`类方法在元类中是实例方法`

**优化**

上面的这种方式是单独在每个类中重写，有没有更好的，一劳永逸的方法呢？其实通过方法慢速查找流程可以发现其查找路径有两条

*   实例方法：`类 -- 父类 -- 根类 -- nil`
*   类方法：`元类 -- 根元类 -- 根类 -- nil`

它们的共同点是如果前面没找到，都会来到`根类即NSObject中查找`，所以我们是否可以将上述的两个方法统一整合在一起呢？答案是可以的，可以通过`NSObject添加分类`的方式来`实现统一处理`，而且由于类方法的查找，在其继承链，查找的也是实例方法，所以可以将实例方法 和 类方法的统一处理放在`resolveInstanceMethod`方法中, 如下所示

```
\+ (BOOL)resolveInstanceMethod:(SEL)sel{
    if (sel == @selector(say666)) {
        NSLog(@"%@ 来了", NSStringFromSelector(sel));
        
        IMP imp = class\_getMethodImplementation(self, @selector(sayMaster));
        Method sayMethod  = class\_getInstanceMethod(self, @selector(sayMaster));
        const char \*type = method\_getTypeEncoding(sayMethod);
        return class\_addMethod(self, sel, imp, type);
    }else if (sel == @selector(sayNB)) {
        NSLog(@"%@ 来了", NSStringFromSelector(sel));
        
        IMP imp = class\_getMethodImplementation(objc\_getMetaClass("LGPerson"), @selector(lgClassMethod));
        Method lgClassMethod  = class\_getInstanceMethod(objc\_getMetaClass("LGPerson"), @selector(lgClassMethod));
        const char \*type = method\_getTypeEncoding(lgClassMethod);
        return class\_addMethod(objc\_getMetaClass("LGPerson"), sel, imp, type);
    }
    return NO;
}


```

这种方式的实现，正好与源码中针对类方法的处理逻辑是一致的，即完美阐述为什么调用了类方法动态方法决议，还要调用对象方法动态方法决议，其根本原因还是`类方法在元类中的实例方法`。

当然，上面这种写法还是会有其他的问题，比如`系统方法也会被更改`，针对这一点，是可以优化的，即我们`可以针对自定义类中方法统一方法名的前缀`，根据前缀来判断是否是自定义方法，然后`统一处理自定义方法`，例如可以在崩溃前 pop 到首页，主要是用于`app线上防崩溃的处理`，提升用户的体验。

### 消息转发流程

在慢速查找的流程中，我们了解到，如果快速 + 慢速没有找到方法实现，动态方法决议也不行，就使用`消息转发`，但是，我们找遍了源码也没有发现消息转发的相关源码，可以通过以下方式来了解，方法调用崩溃前都走了哪些方法

*   通过`instrumentObjcMessageSends`方式打印发送消息的日志
    
*   通过`hopper/IDA反编译`
    

**通过 instrumentObjcMessageSends**

*   通过`lookUpImpOrForward --> log_and_fill_cache --> logMessageSend`, 在 logMessageSend 源码下方找到`instrumentObjcMessageSends`的源码实现，所以，在 main 中调用  
    `instrumentObjcMessageSends`打印方法调用的日志信息，有以下两点准备工作
    *   1、打开 `objcMsgLogEnabled` 开关，即调用`instrumentObjcMessageSends`方法时，传入`YES`
        
    *   2、在`main`中通过`extern` 声明`instrumentObjcMessageSends`方法
        

```
extern void instrumentObjcMessageSends(BOOL flag);

int main(int argc, const char \* argv\[\]) {
    @autoreleasepool {

        LGPerson \*person = \[LGPerson alloc\];
        instrumentObjcMessageSends(YES);
        \[person sayHello\];
        instrumentObjcMessageSends(NO);
        NSLog(@"Hello, World!");
    }
    return 0;
}


```

*   通过`logMessageSend`源码，了解到消息发送打印信息存储在`/tmp/msgSends` 目录，如下所示  
    
    ![](http://upload-images.jianshu.io/upload_images/2251862-f6e36b60147e65b7.png)
    
    消息发送日志路径
    
*   运行代码，并前往`/tmp/msgSends` 目录，发现有`msgSends`开头的日志文件，打开发现在崩溃前，执行了以下方法
    
    *   两次`动态方法决议`：`resolveInstanceMethod`方法
    *   两次`消息快速转发`：`forwardingTargetForSelector`方法
    *   两次`消息慢速转发`：`methodSignatureForSelector` + `resolveInstanceMethod`  
        
        ![](http://upload-images.jianshu.io/upload_images/2251862-389902cbedf78d41.jpg)
        
        消息发送日志详情
        

**通过 hopper/IDA 反编译**

Hopper 和 IDA 是一个可以帮助我们静态分析可视性文件的工具，可以将可执行文件反汇编成伪代码、控制流程图等，下面以 Hopper 为例 (注：hopper 高级版本是一款收费软件，针对比较简单的反汇编需求来说，demo 版本足够使用了)

*   运行程序崩溃，查看堆栈信息
    
    ![](http://upload-images.jianshu.io/upload_images/2251862-e3f0e315d38f0fe2.jpg)
    
    查看堆栈打印信息
    
*   发现`___forwarding___`来自`CoreFoundation`  
    
    ![](http://upload-images.jianshu.io/upload_images/2251862-003411ba03be21dd.jpg)
    
    \_\_\_forwarding\_\_\_源码定位
    
*   通过`image list`，读取整个镜像文件, 然后搜索`CoreFoundation`，查看其可执行文件的路径  
    
    ![](http://upload-images.jianshu.io/upload_images/2251862-e4fedc385521e4bc.jpg)
    
    从镜像文件中查找 CoreFoundation
    
*   通过文件路径，找到`CoreFoundation`的`可执行文件`  
    
    ![](http://upload-images.jianshu.io/upload_images/2251862-d8145f81f812dd08.jpg)
    
    查找 CoreFoundation 的可执行文件
    
*   打开`hopper`，选择`Try the Demo`，然后将上一步的可执行文件拖入 hopper 进行反汇编，选择`x86(64 bits)`  
    
    ![](http://upload-images.jianshu.io/upload_images/2251862-5614d16722f20e3f.png)
    
    hopper 选择 Demo 版本
    
    ![](http://upload-images.jianshu.io/upload_images/2251862-14e3bac8cb80109e.png)
    
    hopper 反汇编
    
*   以下是反汇编后的界面，主要使用上面的三个功能，分别是 汇编、流程图、伪代码
    
    ![](http://upload-images.jianshu.io/upload_images/2251862-8e5a19034e2fc9e9.png)
    
    hoppper 主要使用的三个功能
    
*   通过左侧的搜索框搜索`__forwarding_prep_0___`，然后选择`伪代码`
    
    *   以下是`__forwarding_prep_0___`的汇编伪代码，跳转至`___forwarding___`  
        
        ![](http://upload-images.jianshu.io/upload_images/2251862-d1eea35429b42c71.png)
        
        伪代码 -\_\_\_forwarding\_\_\_
        
    *   以下是`___forwarding___`的伪代码实现，首先是查看是否实现`forwardingTargetForSelector`方法，如果没有响应，跳转至`loc_6459b`即快速转发没有响应，进入`慢速转发`流程，  
        
        ![](http://upload-images.jianshu.io/upload_images/2251862-b6116c32477e5ec4.png)
        
        伪代码 - forwardingTargetForSelector
        
    *   跳转至`loc_6459b`，在其下方判断是否响应`methodSignatureForSelector`方法，  
        
        ![](http://upload-images.jianshu.io/upload_images/2251862-fc98b24257514987.png)
        
        伪代码 - methodSignatureForSelector
        
        *   如果`没有响应`，跳转至`loc_6490b`，则直接报错
            
        *   如果获取`methodSignatureForSelector`的`方法签名`为 nil，也是直接报错  
            
            ![](http://upload-images.jianshu.io/upload_images/2251862-bbef258b355e22cd.png)
            
            伪代码 - methodSignatureForSelector 为 nil 时报错
            
*   如果`methodSignatureForSelector`返回值不为空，则在`forwardInvocation`方法中对`invocation`进行处理  
    
    ![](http://upload-images.jianshu.io/upload_images/2251862-e20ca7ad3d5abe6d.png)
    
    伪代码 - forwardInvocation
    

所以，通过上面两种查找方式可以验证，消息转发的方法有 3 个

*   【快速转发】`forwardingTargetForSelector`
*   【慢速转发】
    *   `methodSignatureForSelector`
    *   `forwardInvocation`

所以，综上所述，消息转发整体的流程如下

![](http://upload-images.jianshu.io/upload_images/2251862-c7a4fe688fd72ac3.png)

消息转发整体流程

消息转发的处理主要分为两部分：

*   【快速转发】当慢速查找，以及动态方法决议均没有找到实现时，进行消息转发，首先是进行`快速消息转发`，即走到`forwardingTargetForSelector`方法
    *   如果返回`消息接收者`，在消息接收者中还是没有找到，则进入另一个方法的查找流程
        
    *   如果返回`nil`，则进入慢速消息转发
        
*   【慢速转发】执行到`methodSignatureForSelector`方法
    *   如果返回的`方法签名`为`nil`，则直接`崩溃报错`
        
    *   如果返回的方法签名`不为nil`，走到`forwardInvocation`方法中，对 invocation 事务进行处理，如果不处理也不会报错
        

#### 【第二次机会】快速转发

针对前文的崩溃问题，如果动态方法决议也没有找到实现，则需要在`LGPerson`中重写`forwardingTargetForSelector`方法，将 LGPerson 的实例方法的`接收者指定为LGStudent`的对象（LGStudent 类中有 say666 的具体实现），如下所示

```
\- (id)forwardingTargetForSelector:(SEL)aSelector{
    NSLog(@"%s - %@",\_\_func\_\_,NSStringFromSelector(aSelector));


    
    return \[LGStudent alloc\];
}


```

执行结果如下

![](http://upload-images.jianshu.io/upload_images/2251862-5b57004e0e2b0ad8.png)

快速转发 - 指定消息接收者

也可以直接不指定消息接收者，`直接调用父类的该方法`，如果还是没有找到，则`直接报错`  

![](http://upload-images.jianshu.io/upload_images/2251862-f8204a9cc314a8f6.png)

快速转发 - 调用父类

#### 【第三次机会】慢速转发

针对`第二次机会即快速转发`中还是没有找到，则进入最后的一次挽救机会，即在`LGPerson`中重写`methodSignatureForSelector`，如下所示

```
\- (NSMethodSignature \*)methodSignatureForSelector:(SEL)aSelector{
    NSLog(@"%s - %@",\_\_func\_\_,NSStringFromSelector(aSelector));
    return \[NSMethodSignature signatureWithObjCTypes:"v@:"\];
}

- (void)forwardInvocation:(NSInvocation \*)anInvocation{
    NSLog(@"%s - %@",\_\_func\_\_,anInvocation);
}


```

打印结果如下，发现`forwardInvocation`方法中不对 invocation 进行处理，也不会崩溃报错  

![](http://upload-images.jianshu.io/upload_images/2251862-4ccccd536409b935.png)

invocation 未处理的打印

也可以`处理invocation事务`，如下所示，修改`invocation`的`target`为`[LGStudent alloc]`，调用 `[anInvocation invoke]` 触发 即`LGPerson`类的`say666`实例方法的调用会调用`LGStudent`的`say666`方法

```
\- (void)forwardInvocation:(NSInvocation \*)anInvocation{
    NSLog(@"%s - %@",\_\_func\_\_,anInvocation);
    anInvocation.target = \[LGStudent alloc\];
    \[anInvocation invoke\];
}



```

打印结果如下

![](http://upload-images.jianshu.io/upload_images/2251862-b3cd2dfc46779f7f.png)

invocation 处理后的打印

所以，由上述可知，无论在`forwardInvocation`方法中`是否处理invocation`事务，程序都`不会崩溃`。

“动态方法决议为什么执行两次？” 问题探索
---------------------

在前文中提及了`动态方法决议`方法执行了两次，有以下两种分析方式

**启用上帝视角的探索**

在慢速查找流程中，我们了解到`resolveInstanceMethod`方法的执行是通过`lookUpImpOrForward --> resolveMethod_locked --> resolveInstanceMethod`来到`resolveInstanceMethod`源码，在源码中通过发送`resolve_sel`消息触发，如下所示  

![](http://upload-images.jianshu.io/upload_images/2251862-ba0a47769ae59677.png)

resolveInstanceMethod 方法触发原理

所以可以在`resolveInstanceMethod`方法中`IMP imp = lookUpImpOrNil(inst, sel, cls);`处加一个断点，通过`bt`打印`堆栈信息`来看到底发生了什么

*   在`resolveInstanceMethod`方法中`IMP imp = lookUpImpOrNil(inst, sel, cls);`处加一个断点，运行程序，直到第一次`“来了”`，通过 bt 查看`第一次动态方法决议`的堆栈信息，此时的 sel 是`say666`  
    
    ![](http://upload-images.jianshu.io/upload_images/2251862-4a8c6bf5ebf79a66.png)
    
    第一次动态方法决议堆栈信息
    
*   继续往下执行，直到第二次`“来了”打印`，查看堆栈信息，在第二次中，我们可以看到是通过`CoreFoundation`的`-[NSObject(NSObject) methodSignatureForSelector:]`方法，然后通过`class_getInstanceMethod`再次进入动态方法决议，  
    
    ![](http://upload-images.jianshu.io/upload_images/2251862-da12cc13ba9dc29c.png)
    
    第二次动态方法决议堆栈信息
    
*   通过上一步的堆栈信息，我们需要去看看 CoreFoundation 中到底做了什么？通过`Hopper`反汇编`CoreFoundation`的可执行文件，查看`methodSignatureForSelector`方法的伪代码  
    
    ![](http://upload-images.jianshu.io/upload_images/2251862-5c27a254d62ef367.png)
    
    methodSignatureForSelector 伪代码进入方式
    
*   通过`methodSignatureForSelector`伪代码进入`___methodDescriptionForSelector`的实现  
    
    ![](http://upload-images.jianshu.io/upload_images/2251862-71932fb077753303.jpg)
    
    methodDescriptionForSelector 方法的伪代码
    
*   进入 `___methodDescriptionForSelector`的伪代码实现，结合汇编的堆栈打印，可以看到，在`___methodDescriptionForSelector`这个方法中调用了`objc4-781`的`class_getInstanceMethod`  
    
    ![](http://upload-images.jianshu.io/upload_images/2251862-b0b1bfe648c809dd.jpg)
    
    \_\_\_methodDescriptionForSelector 方法的伪代码调用了 class\_getInstanceMethod
    
*   在 objc 中的源码中搜索`class_getInstanceMethod`，其源码实现如下所示  
    
    ![](http://upload-images.jianshu.io/upload_images/2251862-942c209aed27fdd2.jpg)
    
    class\_getInstanceMethod 方法源码
    

这一点可以通过`代码调试`来验证，如下所示，在`class_getInstanceMethod`方法处加一个断点，在执行了`methodSignatureForSelector`方法后，返回了签名，说明方法签名是生效的，苹果在走到`invocation`之前，`给了开发者一次机会再去查询`，所以走到`class_getInstanceMethod`这里，又去走了一遍方法查询`say666`, 然后会再次走到`动态方法决议`  

![](http://upload-images.jianshu.io/upload_images/2251862-e84dd0973893cf52.jpg)

class\_getInstanceMethod 方法调试验证

所以，上述的分析也印证了前文中`resolveInstanceMethod`方法执行了两次的原因

**无上帝视角的探索**

如果在没有上帝视角的情况下，我们也可以`通过代码`来`推导`在哪里再次调用了动态方法决议

*   LGPerson 中重写`resolveInstanceMethod`方法，并加上`class_addMethod`操作即`赋值IMP`，此时`resolveInstanceMethod`会走两次吗？  
    
    ![](http://upload-images.jianshu.io/upload_images/2251862-d2543ae677020a1d.png)
    
    resolveInstanceMethod 方法调试验证
    
    【结论】：通过运行发现，`如果赋值了IMP，动态方法决议只会走一次`，说明不是在这里走第二次动态方法决议，

继续往下探索

*   去掉`resolveInstanceMethod`方法中的赋值 IMP，在`LGPerson`类中重写`forwardingTargetForSelector`方法，并指定返回值为`[LGStudent alloc]`，重新运行，如果`resolveInstanceMethod`打印了两次，说明是在`forwardingTargetForSelector`方法之前执行了 动态方法决议，反之，在`forwardingTargetForSelector`方法之后  
    
    ![](http://upload-images.jianshu.io/upload_images/2251862-7c7cac6dbb1afb8d.png)
    
    forwardingTargetForSelector 方法调试验证
    
    【结论】：发现`resolveInstanceMethod`中的打印还是只打印了一次，数排名第二次动态方法决议 在`forwardingTargetForSelector`方法后
*   在 LGPerson 中重写 `methodSignatureForSelector` 和 `forwardInvocation`，运行  
    
    ![](http://upload-images.jianshu.io/upload_images/2251862-ca328f2349d6e50d.png)
    
    methodSignatureForSelector+forwardInvocation 方法调试验证
    
    【结论】：`第二次动态方法决议`在 `methodSignatureForSelector` 和 `forwardInvocation`方法之间

第二种分析同样可以论证前文中`resolveInstanceMethod`执行了两次的原因

经过上面的论证，我们了解到其实在慢速小子转发流程中，在`methodSignatureForSelector` 和 `forwardInvocation`方法之间还有一次动态方法决议，即苹果再次给的一个机会，如下图所示  

![](http://upload-images.jianshu.io/upload_images/2251862-033fc5c4670face6.png)

消息转发流程 - 2

总结
--

到目前为止，objc\_msgSend 发送消息的流程就分析完成了，在这里简单总结下

*   `【快速查找流程】`首先，在类的`缓存cache`中查找指定方法的实现
    
*   `【慢速查找流程】`如果缓存中没有找到，则在`类的方法列表`中查找，如果还是没找到，则去`父类链的缓存和方法列表`中查找
    
*   `【动态方法决议】`如果慢速查找还是没有找到时，`第一次补救机会`就是尝试一次`动态方法决议`，即重写`resolveInstanceMethod`/`resolveClassMethod` 方法
    
*   `【消息转发】`如果动态方法决议还是没有找到，则进行`消息转发`，消息转发中有两次补救机会：`快速转发+慢速转发`
    
*   如果转发之后也没有，则程序直接报错崩溃`unrecognized selector sent to instance`
    

"小礼物走一走，来简书关注我"

还没有人赞赏，支持一下

[![](https://upload.jianshu.io/users/upload_avatars/2251862/4b8d73cd-8776-46d3-8c5e-19c7f71ef0bd.JPG?imageMogr2/auto-orient/strip|imageView2/1/w/100/h/100/format/webp)](https://www.jianshu.com/u/1e8432e01e5a)

总资产 15 (约 1.34 元) 共写了 14.3W 字获得 534 个赞共 355 个粉丝

### 被以下专题收入，发现更多相似内容