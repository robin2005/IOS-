\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[www.jianshu.com\](https://www.jianshu.com/p/6302999ecd92)

[![](https://upload.jianshu.io/users/upload_avatars/1319505/9754fac0eb59?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96/format/webp)](https://www.jianshu.com/u/40dce98b3416)

2019.06.20 09:39:40 字数 848 阅读 26

#### 面试题引发的思考：

**Q: 以下 2 段代码运行结果是？具体原因是什么？**

```
dispatch\_queue\_t queue = dispatch\_get\_global\_queue(0, 0);
for (int i = 0; i < 1000; i++) {
    dispatch\_async(queue, ^{
        self.name = \[NSString stringWithFormat:@"abcdefghijk"\];
    });
}


```

```
dispatch\_queue\_t queue = dispatch\_get\_global\_queue(0, 0);
for (int i = 0; i < 1000; i++) {
    dispatch\_async(queue, ^{
        self.name = \[NSString stringWithFormat:@"abc"\];
    });
}


```

*   **代码一运行，程序崩溃；代码二运行，程序正常运行。**
*   **代码一使用 动态分配内存 存储数据；**
*   **代码二使用 Tagged Pointer 技术存储数据。**
*   **崩溃原因如下：**
*   **多条线程同时执行`[_name release];`这行代码，会引起`EXC_BAD_ACCESS`坏内存访问，导致崩溃。**

```
self.name = \[NSString stringWithFormat:@"abcdefghijk"\];


\[self setName:\[NSString stringWithFormat:@"abcdefghijk"\]\];


- (void)setName:(NSString \*)name {
    if (\_name != name) {
        \[\_name release\]; 
        \_name = \[name copy\]; 
    }
}


```

**Q: 以上问题解决方案？**

*   **`atomic`原子性**
*   **加锁**
*   **同步执行**
*   **Tagged Pointer**

**Q: 简述 Tagged Pointer？**

*   **从 64bit 开始，用于优化`NSNumber`、`NSDate`、`NSString`等小对象的存储；**
*   **`NSNumber`指针里面存储的数据变成了：`Tag + Data`，也就是将数据直接存储在了指针中；**
*   **当指针不够存储数据时，才会使用动态分配内存的方式来存储数据。**

* * *

### 1\. Tagged Pointer 介绍

#### (1) 什么是 Tagged Pointer ？

在 2013 年 9 月，苹果推出了 [iPhone5s](https://links.jianshu.com/go?to=http%3A%2F%2Fen.wikipedia.org%2Fwiki%2FIPhone_5S)，与此同时，iPhone5s 配备了首个采用 64 位架构的 [A7 双核处理器](https://links.jianshu.com/go?to=http%3A%2F%2Fen.wikipedia.org%2Fwiki%2FApple_A7)，为了节省内存和提高执行效率，苹果提出了 **Tagged Pointer** 的概念。对于 64 位程序，引入 **Tagged Pointer** 后，相关逻辑能减少一半的内存占用，以及 3 倍的访问速度提升，100 倍的创建、销毁速度提升。

* * *

#### (2) Tagged Pointer 详解

##### 1> Tagged Pointer 优点

> *   **从 64bit 开始，iOS 引入了 Tagged Pointer 技术，用于优化`NSNumber`、`NSDate`、`NSString`等小对象的存储。**
>     
> *   **在没有使用 Tagged Pointer 之前：**  
>     **`NSNumber`等对象需要动态分配内存、维护引用计数等，`NSNumber`指针存储的是堆中`NSNumber`对象的地址值；**
>     
> *   **使用 Tagged Pointer 之后：**  
>     **`NSNumber`指针里面存储的数据变成了：`Tag + Data`，也就是将数据直接存储在了指针中；**  
>     **当指针不够存储数据时，才会使用动态分配内存的方式来存储数据。**
>     
> *   **`objc_msgSend`能识别 Tagged Pointer ，比如`NSNumber`的`intValue`方法，直接从指针提取数据，节省了以前的调用开销。**
>     

##### 2> 如何判断一个指针是否为 Tagged Pointer ？

由 **[OC 源码](https://links.jianshu.com/go?to=https%3A%2F%2Fopensource.apple.com%2Ftarballs%2Fobjc4%2F)**可知判断是否为 **Tagged Pointer** 的函数为`_objc_isTaggedPointer`：

![](http://upload-images.jianshu.io/upload_images/1319505-b6aab0f209456f97.png)

\_objc\_isTaggedPointer 函数

由以上源码可知判断方法为：

> **将编译后的地址与`_OBJC_TAG_MASK`进行 &(与运算) 得到的值是否与`_OBJC_TAG_MASK`相等。**

那么查看`_OBJC_TAG_MASK`的值：

![](http://upload-images.jianshu.io/upload_images/1319505-30416bb90dfa6234.png)

\_OBJC\_TAG\_MASK 取值

由以上源码可以简化为：

![](http://upload-images.jianshu.io/upload_images/1319505-6b9b3d6602db41e3.png)

\_OBJC\_TAG\_MASK 取值简化

由以上源码可知：

> **Q：判断一个指针是否为 Tagged Pointer 的方法：**
> 
> *   **Mac 平台，地址最低有效位是 1**
> *   **iOS 平台，地址最高有效位是 1 (第 64bit)**

所以判断方法如下：

```
\- (BOOL)isTaggedPointer:(id)pointer {
    return ((uintptr\_t)pointer & \_OBJC\_TAG\_MASK);
}


```

* * *

### 2\. Tagged Pointer 示例

#### (1) 面试题

```
@interface ViewController ()
@property (nonatomic, copy) NSString \*name;
@end

@implementation ViewController
- (void)viewDidLoad {
    \[super viewDidLoad\];
    
    dispatch\_queue\_t queue = dispatch\_get\_global\_queue(0, 0);
    for (int i = 0; i < 1000; i++) {
        dispatch\_async(queue, ^{
            self.name = \[NSString stringWithFormat:@"abcdefghijk"\];
        });
    }
}
@end


```

运行程序：

![](http://upload-images.jianshu.io/upload_images/1319505-1312bae106516505.png)

程序崩溃

程序崩溃，提示`EXC_BAD_ACCESS`坏内存访问，而且报错为`libobjc.A.dylib objc_release:`，释放对象时出错。

崩溃原因如下：

```
self.name = \[NSString stringWithFormat:@"abcdefghijk"\];


\[self setName:\[NSString stringWithFormat:@"abcdefghijk"\]\];


- (void)setName:(NSString \*)name {
    if (\_name != name) {
        \[\_name release\]; 
        \_name = \[name copy\]; 
    }
}


```

而多条线程同时执行`[_name release];`这行代码，会引起`EXC_BAD_ACCESS`坏内存访问，导致崩溃。

* * *

#### (2) 解决方案

##### 1> 解决方案一：`atomic`原子性

```
@interface ViewController ()
@property (atomic, copy) NSString \*name;
@end

@implementation ViewController
- (void)viewDidLoad {
    \[super viewDidLoad\];
    
    dispatch\_queue\_t queue = dispatch\_get\_global\_queue(0, 0);
    for (int i = 0; i < 1000; i++) {
        dispatch\_async(queue, ^{
            self.name = \[NSString stringWithFormat:@"abcdefghijk"\];
        });
    }
}
@end


```

##### 2> 解决方案二：加锁

```
@interface ViewController ()
@property (nonatomic, copy) NSString \*name;
@end

@implementation ViewController
- (void)viewDidLoad {
    \[super viewDidLoad\];
    
    
    dispatch\_queue\_t queue = dispatch\_get\_global\_queue(0, 0);
    for (int i = 0; i < 1000; i++) {
        dispatch\_async(queue, ^{
            
            self.name = \[NSString stringWithFormat:@"abcdefghijk"\];
            
        });
    }
}
@end


```

##### 3> 解决方案三：同步执行

```
@interface ViewController ()
@property (nonatomic, copy) NSString \*name;
@end

@implementation ViewController
- (void)viewDidLoad {
    \[super viewDidLoad\];
    
    
    dispatch\_queue\_t queue = dispatch\_get\_global\_queue(0, 0);
    for (int i = 0; i < 1000; i++) {
        dispatch\_sync(queue, ^{
            self.name = \[NSString stringWithFormat:@"abcdefghijk"\];
        });
    }
}
@end


```

##### 4> 解决方案四： Tagged Pointer

```
\- (void)viewDidLoad {
    \[super viewDidLoad\];

    NSString \*str1 = \[NSString stringWithFormat:@"abcdefghijk"\];
    NSString \*str2 = \[NSString stringWithFormat:@"abc"\];

    NSLog(@"\\n%p %@", str1, \[str1 class\]);
    NSLog(@"\\n%p %@", str2, \[str2 class\]);
}


 0x600003d04c00 \_\_NSCFString
 0xc075c16dfff62d83 NSTaggedPointerString


```

"小礼物走一走，来简书关注我"

还没有人赞赏，支持一下

[![](https://upload.jianshu.io/users/upload_avatars/1319505/9754fac0eb59?imageMogr2/auto-orient/strip|imageView2/1/w/100/h/100/format/webp)](https://www.jianshu.com/u/40dce98b3416)

总资产 1 (约 0.16 元) 共写了 4.8W 字获得 24 个赞共 23 个粉丝

### 被以下专题收入，发现更多相似内容

### 推荐阅读[更多精彩内容](https://www.jianshu.com/)

*   一. 定时器 1.CADisplayLink、NSTimer CADisplayLink、NSTimer 会对 ta...
    
    [![](https://upload-images.jianshu.io/upload_images/103735-a4082e3b5cc314f0.png?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)](https://www.jianshu.com/p/9e26c6ac2500)
*   Swift1> Swift 和 OC 的区别 1.1> Swift 没有地址 / 指针的概念 1.2> 泛型 1.3> 类型严谨 对...
    
*   1\. 下面代码执行结果如何 运行结果 分析: 因为 data 是 copy 属性，所以在其 set 方法里先执行判断，然后执行 re...
    
    [![](https://upload-images.jianshu.io/upload_images/1653926-7644658f002fe273.png?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)](https://www.jianshu.com/p/869ca6fbdd5c)
*   CADisplayLink、NSTimer 使用注意点: 1.CADisplayLink、NSTimer 会对 targ...
    
    [![](https://upload-images.jianshu.io/upload_images/2163717-afee26101f38992e.png?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)](https://www.jianshu.com/p/2f578b91083c)
*   Ajax 介绍 Ajax 最早产生于 2005 年，Ajax 表示 Asynchronous JavaScript and X...
    
    [![](https://upload-images.jianshu.io/upload_images/1696952-578bab34a9a805de.png?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)](https://www.jianshu.com/p/46fd608c58dc)