\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[www.jianshu.com\](https://www.jianshu.com/p/898e24db24d6)

[![](https://upload.jianshu.io/users/upload_avatars/1880020/6fdf4121da39.jpg?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96/format/webp)](https://www.jianshu.com/u/e90a0cf1753c)

0.3392020.05.13 17:21:53 字数 1,253 阅读 144

`Method Swizzing`方法交换，在`Objective-C`中使用还是比较常见的，要搞清它的本质，要首先理解方法的本质。

### 一、方法的本质

Objective-C 中，方法是由`SEL`和`IMP`组成的，前者叫做方法编号，后者叫方法实现。Objective-C 中调用方法，其实就是通过`SEL`查找`IMP`的过程。  
`SEL`：表示选择器，通常可以把它理解为一个字符串。运行时维护着一张全局的`SEL表`，将相同字符串的方法名映射到唯一一个`SEL`。 通过`sel_registerName(char *name)`方法，可以查找到这张表中方法名对应的`SEL`。苹果提供了一个语法糖`@selector`用来方便地调用该函数。  
`IMP`：是一个函数指针。objc 中的方法最终会被转换成纯 C 的函数，IMP 就是为了表示这些函数的地址。

### 二、Method Swizzing 原理

方法混淆就是利用 runtime 特性以及方法的组成本质实现的。比如有方法`sel1`对应`imp1`，`sel2`对应`imp2`，经过方法混淆，使得 runtime 在方法查找时将`sel1`的查找结果变为`imp2`，如下图：  

![](http://upload-images.jianshu.io/upload_images/1880020-dc2cf9bdab663fae.png)

Method Swizzing.png

### 三、安全实现

新建一个项目，添加两个类：`Animal`、`Dog`（父类为`Animal`），添加一个方法`-eat`。

```
\- (void)eat{
    NSLog(@"%s", \_\_func\_\_);
}


```

添加一个`Dog`的 Method Swizzing 分类，

```
\+ (void)load {
    Method oriMethod = class\_getInstanceMethod(self, @selector(eat));
    Method swiMethod = class\_getInstanceMethod(self, @selector(swi\_eat));
    
    method\_exchangeImplementations(oriMethod, swiMethod);
}
- (void)swi\_eat{
    NSLog(@"%s", \_\_func\_\_);
    
    \[self swi\_eat\];
}


```

下面具体分析一下可能遇到的坑点，以及如何避免。

#### 3.1 多次交互

最好写成单例，避免方法多次交换。例如手动调用`+ load`方法。

viewController 执行下面的代码：

```
    \[Dog load\];
    
    Dog \*dog = Dog.new;
    \[dog eat\];


```

打印结果为：

这是因为执行了两次方法交换，方法被换回去了，为了避免这种意外，最好写成单例。

#### 3.2 子类交换父类实现的方法

*   父类`Animal`实现了`-eat`方法，子类`Dog`没有重写
*   `Dog分类`进行方法交换

`Animal`和`Dog`分别调用`-eat`方法，子类正常打印，但是父类确崩溃了，为什么呢?  
因为`Dog`交换方法时，先在本类查找`-eat`方法，再往父类查找。在父类 `Animal`找到方法实现，就执行了方法交换。但是新方法`-swi_eat`在子类`Dog`，`Animal`找不到就报异常：`reason: '-[Animal swi_eat]: unrecognized selector sent to instance 0x600000267590'`。

所以安全的实现，应该**只交换子类的方法，不动父类方法**。

```
\+ (void)load {
    
    static dispatch\_once\_t onceToken;
    dispatch\_once(&onceToken, ^{
        
        Method oriMethod = class\_getInstanceMethod(self, @selector(eat));
        Method swiMethod = class\_getInstanceMethod(self, @selector(swi\_eat));
        
        
        
        BOOL didAddMethod = class\_addMethod(self, @selector(eat), method\_getImplementation(swiMethod), method\_getTypeEncoding(swiMethod));
        if (didAddMethod) {
            
            
            class\_replaceMethod(self, @selector(swi\_eat), method\_getImplementation(oriMethod), method\_getTypeEncoding(oriMethod));
        }else{
            
            method\_exchangeImplementations(oriMethod, swiMethod);
        }
        
    });
}


```

1.  `class_addMethod`方法：取得新方法`swiMethod`的实现和方法类型签名，把新方法 (swiMethod) 实现放到旧方法 (oriMethod) 名中，此刻调用`-eat`就是调用`-swi_eat`。
2.  `didAddMethod`：  
    2.1 添加成功，说明之前本类没有实现`-eat`方法，此时不必做方法交换，直接将 swiMethod 的 IMP 替换为父类 (oriMethod) 的 IMP 即可。 那么此时调用`-swi_eat`，实则是调用父类的`-eat`。  
    2.2 添加失败，说明本类之前已经实现，直接交换即可。

> `class_addMethod`不会覆盖本类中已存在的实现，只会覆盖本类从父类继承的方法实现

#### 3.3 只有方法声明，没有方法实现，却做了方法交换——会造成死循环

为什么呢：

1.  因为本类没有`-eat`的实现，所以执行`class_addMethod`成功，此时调用`-eat`就是调用`-swi_eat`。
2.  此时`didAddMethod`为`YES`，添加方法成功，进行方法交换`-class_replaceMethod`。那么此时调用`-swi_eat`，实则是调用本类的`-eat`。最终造成死循环。

改变代码逻辑如下：

```
\+ (void)load {
    
    static dispatch\_once\_t onceToken;
    dispatch\_once(&onceToken, ^{
        
        Method oriMethod = class\_getInstanceMethod(self, @selector(eat));
        Method swiMethod = class\_getInstanceMethod(self, @selector(swi\_eat));
        
        
        if (!oriMethod) {
            class\_addMethod(self, @selector(eat), method\_getImplementation(swiMethod), method\_getTypeEncoding(swiMethod));
            method\_setImplementation(swiMethod, imp\_implementationWithBlock(^(id self, SEL \_cmd){
                NSLog(@"%@ 方法未实现", self);
            }));
        }
        
        
        
        BOOL didAddMethod = class\_addMethod(self, @selector(eat), method\_getImplementation(swiMethod), method\_getTypeEncoding(swiMethod));
        if (didAddMethod) {
            
            
            class\_replaceMethod(self, @selector(swi\_eat), method\_getImplementation(oriMethod), method\_getTypeEncoding(oriMethod));
        }else{
            
            method\_exchangeImplementations(oriMethod, swiMethod);
        }
        
    });
}


```

1.  未实现方法时添加方法，此时调用`-eat`就是调用`-swi_eat`。
2.  如果经过正常的方法交换，`-swi_eat`方法内部还是调用自己`-eat`。
3.  所以未实现方法时，用`block`修改`-swi_eat`的实现，就可以断开死循环了。

### 四、总结

*   尽可能在 + load 方法中交换方法
*   最好使用单例保证只交换一次
*   自定义方法名不能产生冲突
*   对于系统方法要调用原始实现，避免对系统产生影响
*   做好注释（因为方法交换比较绕）
*   迫不得已情况下才去使用方法交换

目前有两类常用的 Method Swizzling 实现方案，诸如 [RSSwizzle](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Frabovik%2FRSSwizzle) 和 [jrswizzle](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Frentzsch%2Fjrswizzle) 这种较为复杂且周全的一些实现方案可供参考。

> **block hook**：  
> 关键在于理解，将 Block\_layout 的 invoke 指针，强行指向\_objc\_msgForward 指针，从而启动消息转发机制，在消息转发最后一步，将副本和 hook block 取出包装成 NSInvocation 进行调用。  
> 可以参考：[Block hook 正确姿势](https://links.jianshu.com/go?to=https%3A%2F%2Fjuejin.im%2Fpost%2F5c653921e51d457fa676eafc%23heading-4)、[yulingtianxia/BlockHook](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fyulingtianxia%2FBlockHook)、[Block Hook+libffi](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fqq_32792839%2Farticle%2Fdetails%2F99842250)  
> 拓展：抖音技术团队 [Objective-C & Swift 最轻量级 Hook 方案](https://links.jianshu.com/go?to=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2FwxigL1Clem1dR8Nkt8LLMw)

"小礼物走一走，来简书关注我"

还没有人赞赏，支持一下

[![](https://upload.jianshu.io/users/upload_avatars/1880020/6fdf4121da39.jpg?imageMogr2/auto-orient/strip|imageView2/1/w/100/h/100/format/webp)](https://www.jianshu.com/u/e90a0cf1753c)

总资产 4 (约 0.34 元) 共写了 3.0W 字获得 55 个赞共 23 个粉丝

### 被以下专题收入，发现更多相似内容