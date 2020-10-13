\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[www.jianshu.com\](https://www.jianshu.com/p/471b2753bfb6)

[![](https://upload.jianshu.io/users/upload_avatars/1319505/9754fac0eb59?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96/format/webp)](https://www.jianshu.com/u/40dce98b3416)

2019.09.26 09:43:38 字数 927 阅读 46

#### 面试题引发的思考：

**Q: ARC 都帮我们做了什么？**

*   **ARC 是 LLVM 编译器和 Runtime 系统相互协作的一个结果。**

**Q: 谈一谈`weak`指针的实现原理。**

*   **利用哈希表对 `weak`指针 与 被指向的对象 进行标记、关联；**
*   **当对象销毁释放内存时，通过 标记 对 `weak`指针地址 进行查找，把 `weak`指针 逐个置为`nil`。**

**Q: 指针类型的区别？**

*   **`__strong`：对对象进行`retain`；**
*   **`__weak`：不会对对象进行`retain`，当对象销毁时，会自动指向`nil`；**
*   **`__unsafe_unretained`：不会对对象进行`retain`，当对象销毁时，依然指向之前的内存空间 (野指针)。**

* * *

### 1\. 案例分析

#### (1) 案例一

```
@interface Person : NSObject
@end

@implementation Person
- (void)dealloc {
    NSLog(@"%s", \_\_func\_\_);
}
@end


- (void)viewDidLoad {
    \[super viewDidLoad\];

    NSLog(@"begin");
    {
        
        Person \*person = \[\[Person alloc\] init\];
    }
    NSLog(@"end");
}


Demo\[1234:567890\] begin
Demo\[1234:567890\] -\[Person dealloc\]
Demo\[1234:567890\] end


```

由以上代码可知：

强指针`person`是大括号内部局部变量，大括号执行结束后`person`会被销毁，此时没有指针指向`Person`对象，`Person`对象会被释放，打印结果可以验证。

#### (2) 案例二

```
\- (void)viewDidLoad {
    \[super viewDidLoad\];
    
    \_\_strong Person \*person1;

    NSLog(@"begin");
    {
        Person \*person = \[\[Person alloc\] init\];
        
        person1 = person;
    }
    NSLog(@"end - %@", person1);
}


Demo\[1234:567890\] begin
Demo\[1234:567890\] end - <Person: 0x600003dc8640>
Demo\[1234:567890\] -\[Person dealloc\]


```

由以上代码可知：

强指针`person1`指向`Person`的对象，`viewDidLoad`执行结束后以后`person1`才会销毁，此时没有指针指向`Person`对象，`Person`对象会被释放，打印结果可以验证。

#### (3) 案例三

```
\- (void)viewDidLoad {
    \[super viewDidLoad\];
    
    \_\_weak Person \*person2;

    NSLog(@"begin");
    {
        Person \*person = \[\[Person alloc\] init\];
        person2 = person;
    }
    NSLog(@"end - %@", person2);
}


Demo\[1234:567890\] begin
Demo\[1234:567890\] -\[Person dealloc\]
Demo\[1234:567890\] end - (null)


```

由以上代码可知：

弱指针`person2`指向`Person`的对象，大括号执行结束后`person`会被销毁，此时没有指针指向`Person`对象，`Person`对象会被释放，打印结果可以验证。

还可以发现：弱指针指向的对象销毁，弱指针的值会自动清空，所以`person2`打印结果为`null`。

#### (4) 案例四

```
\- (void)viewDidLoad {
    \[super viewDidLoad\];
    
    \_\_unsafe\_unretained Person \*person3;

    NSLog(@"begin");
    {
        Person \*person = \[\[Person alloc\] init\];
        person3 = person;
    }
    NSLog(@"end - %@", person3);
}


Demo\[1234:567890\] begin
Demo\[1234:567890\] -\[Person dealloc\]
Demo\[1234:567890\] Thread 1: EXC\_BAD\_ACCESS (code=EXC\_I386\_GPFLT)


```

由以上代码可知：

不安全弱指针`person3`指向`Person`的对象，大括号执行结束后`person`会被销毁，此时没有指针指向`Person`对象，`Person`对象会被释放，打印结果可以验证。

还可以发现：不安全弱指针指向的对象销毁，不安全弱指针依然指向之前的内存空间，所以`person3`会导致坏地址访问。

#### (5) 总结

> *   **`__strong`：对对象进行`retain`；**
> *   **`__weak`：不会对对象进行`retain`，当对象销毁时，会自动指向`nil`；**
> *   **`__unsafe_unretained`：不会对对象进行`retain`，当对象销毁时，依然指向之前的内存空间 (野指针)。**

* * *

### 2\. 源码分析

#### (1) `dealloc`方法

由 [OC 源码](https://links.jianshu.com/go?to=https%3A%2F%2Fopensource.apple.com%2Ftarballs%2Fobjc4%2F)查找`dealloc`方法：

![](http://upload-images.jianshu.io/upload_images/1319505-4f95957add86322e.png)

dealloc 函数分析

由 [OC 源码](https://links.jianshu.com/go?to=https%3A%2F%2Fopensource.apple.com%2Ftarballs%2Fobjc4%2F)可知：

> **调用`dealloc`方法，会清除成员变量，移除关联对象，并将指向当前对象的弱指针置为`nil`。**

#### (2) 弱指针置为`nil`的具体操作

由 [iOS 底层原理 - 探寻 Runtime 本质（一）](https://www.jianshu.com/p/18cd64b16eb4)可知：

> **`isa`的结构中的信息`has_sidetable_rc`作用为：**
> 
> *   **判断引用计数器是否过大无法存储在`isa`中；如果为 1，那么引用计数会存储在一个叫`SideTable`的类的属性中。**

属性`SideTable`结构如下：

![](http://upload-images.jianshu.io/upload_images/1319505-e7cee83741dcb94e.png)

SideTable 结构

接下来跳到`clearDeallocating`方法，查看如何将指向当前对象的弱指针置为`nil`：

![](http://upload-images.jianshu.io/upload_images/1319505-ed72f2dcce104ffc.png)

弱指针处理

由 [OC 源码](https://links.jianshu.com/go?to=https%3A%2F%2Fopensource.apple.com%2Ftarballs%2Fobjc4%2F)可知：

> *   **当一个对象`object`被`weak`指针指向时，这个`weak`指针会以`object`作为`key`，被存储到`sideTable`类的`weak_table`这个散列表上对应的一个`weak`指针数组里面。**
> *   **当一个对象`object`的`dealloc`方法被调用时，Runtime 会以`object`为`key`，从`sideTable`的`weak_table`散列表中，找出对应的`weak`指针列表，然后将里面的`weak`指针逐个置为`nil`。**

"小礼物走一走，来简书关注我"

还没有人赞赏，支持一下

[![](https://upload.jianshu.io/users/upload_avatars/1319505/9754fac0eb59?imageMogr2/auto-orient/strip|imageView2/1/w/100/h/100/format/webp)](https://www.jianshu.com/u/40dce98b3416)

总资产 1 (约 0.16 元) 共写了 4.8W 字获得 24 个赞共 23 个粉丝

### 被以下专题收入，发现更多相似内容

### 推荐阅读[更多精彩内容](https://www.jianshu.com/)

*   一. 定时器 1.CADisplayLink、NSTimer CADisplayLink、NSTimer 会对 ta...
    
    [![](https://upload-images.jianshu.io/upload_images/103735-a4082e3b5cc314f0.png?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)](https://www.jianshu.com/p/9e26c6ac2500)
*   1\. 下面代码执行结果如何 运行结果 分析: 因为 data 是 copy 属性，所以在其 set 方法里先执行判断，然后执行 re...
    
    [![](https://upload-images.jianshu.io/upload_images/1653926-7644658f002fe273.png?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)](https://www.jianshu.com/p/869ca6fbdd5c)

*   养育方式分为三大类，专制型、沟通型和行为改进型，三种方式各有长短。 专制型养育强调父母的权威现象，孩子必须服从父母...
    
*   最近有一个关于免费基础教育视频平台的设想，是在用网易公开课看 TED 演讲时想到的。网易公开课上有国际名校的公开课，有...
    
    [![](https://upload.jianshu.io/users/upload_avatars/2764252/01ab9f2f-a6a4-4611-ba52-edabb275c325.jpg?imageMogr2/auto-orient/strip|imageView2/1/w/48/h/48/format/webp)李之琴](https://www.jianshu.com/u/2ab023e4f121)阅读 2