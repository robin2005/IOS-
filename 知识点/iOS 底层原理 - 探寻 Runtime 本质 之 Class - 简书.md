\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[www.jianshu.com\](https://www.jianshu.com/p/3eb0f2678362)

[![](https://upload.jianshu.io/users/upload_avatars/1319505/9754fac0eb59?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96/format/webp)](https://www.jianshu.com/u/40dce98b3416)

2018.05.09 09:45:45 字数 1,696 阅读 112

#### 面试题引发的思考：

**Q: 简述 Class 原理？**

*   **通过 `bits & FAST_DATA_MASK` 可以得到 `class_rw_t` 结构体；**
*   **`class_rw_t`结构体 中存放 _方法列表、属性列表、协议列表_ 等，  
    并包括一个指向 `class_ro_t`结构体 的指针`ro`；**
*   **`class_ro_t`结构体 中存放 _方法列表、属性列表、协议列表_ 等，  
    并包括 _成员变量列表_；包括所有 _类的初始内容_。**

**Q: 简述方法缓存`cache_t`原理？**

*   **iOS 采用方法缓存`cache_t`，用散列表（哈希表）缓存曾经调用过的方法，可以提高方法的查找速度。**  
    **调用方法时优先去 _缓存_ 中查找此方法；  
    缓存 中没有再去 _类对象_ 中查找；  
    然后将其加入 _缓存_ 中方便下次调用。**

* * *

### 1\. `Class`结构的本质

上一章对`isa`结构的本质做了探究，下面探究`Class`的内部结构。

由 [iOS 底层原理 - OC 对象的本质（二）](https://www.jianshu.com/writer#/notebooks/9629880/notes/5691271/preview)可知：

![](http://upload-images.jianshu.io/upload_images/1319505-7c5d3e7bd01816be.png)

isa 指针指向图示

> *   **class 对象和 meta-class 对象结构相似，meta-class 对象是特殊的 class 对象；**
> *   **关于存储方法：**  
>     **class 对象存储对象方法，meta-class 对象存储类方法。**

* * *

#### (1) `Class`的结构

由 **[OC 源码](https://links.jianshu.com/go?to=https%3A%2F%2Fopensource.apple.com%2Ftarballs%2Fobjc4%2F)**可以找出`Class`的结构：

![](http://upload-images.jianshu.io/upload_images/1319505-4ce531659b5bc39c.png)

Class 的结构

由`Class`的结构可知：

> *   **通过`bits & FAST_DATA_MASK`可以得到`class_rw_t`结构体；**
> *   **`class_rw_t`结构体中存放方法列表、属性列表、协议列表等，并包括一个指向`class_ro_t`结构体的指针`ro`；**（`rw`即`readwrite`可读可写）
> *   **`class_ro_t`结构体中存放类的初始内容，包括方法列表、属性列表、协议列表等，并包括成员变量列表。**（`ro`即`readonly`只读）

* * *

#### (2) `class_rw_t` 结构

`class_rw_t`结构体中存放方法列表（`method_array_t`类型）、属性列表 （`property_array_t`类型）、协议列表（`protocol_array_t`类型），接下来分析三者的结构：

![](http://upload-images.jianshu.io/upload_images/1319505-6566718aec7e511c.png)

method\_array\_t、property\_array\_t、protocol\_array\_t 结构

通过以上源码可知：  
**`method_array_t`、`property_array_t`、`protocol_array_t`结构相同；**

下面只对`method_array_t`的内部结构进行分析，`property_array_t`和`protocol_array_t`的可以类推。

![](http://upload-images.jianshu.io/upload_images/1319505-d675965f536313b1.png)

method\_array\_t 内部结构

由`method_array_t`内部结构可知：  
**`method_array_t`是一个二维数组，每个元素是`method_list_t`；**  
**`method_list_t`是一维数组，每个元素是`method_t`；**  
**`class_rw_t`里面的`methods`、`properties`、`protocols`是二维数组，是可读可写的，包含了类的初始内容、分类的内容。**

* * *

#### (3) `class_ro_t`结构

`class_ro_t`结构体中存放方法列表（`method_list_t`类型）、属性列表 （`property_list_t`类型）、协议列表（`protocol_list_t`类型），所以`method_list_t`的内部结构如下：

![](http://upload-images.jianshu.io/upload_images/1319505-2b7a38ff93acdf96.png)

method\_list\_t 内部结构

由`method_list_t`内部结构可知：  
**`method_list_t`是一维数组，每个元素是`method_t`；**  
**`class_ro_t`里面的`baseMethodList`、`baseProtocols`、`ivars`、`baseProperties`是一维数组，是只读的，包含了类的初始内容。**

* * *

#### (4) `class_rw_t`结构与`class_ro_t`结构

由 [iOS 底层原理 - 探寻 Category 本质（一）](https://www.jianshu.com/p/0c031d339e75)可知：

> **`class_rw_t`结构体通过`attachList`函数内的`memmove`和`memcpy`操作将所有分类的对象方法附加到类对象的方法列表中。**

通过源码进入`realizeClass`函数，查看`class_rw_t`与`class_ro_t`两者之间的关系：

![](http://upload-images.jianshu.io/upload_images/1319505-f86a3f75f2252434.png)

realizeClass 函数

由`realizeClass`函数可知：

> **类的初始信息（方法、属性、协议、成员变量等）存放在`class_ro_t`中；**  
> **程序运行时，`class_ro_t`中的列表和分类中的列表合并起来存放在`class_rw_t`中。**

* * *

#### (5) `method_t`结构

由以上分析可知：  
**`class_rw_t`结构体和`class_ro_t`结构体的方法列表，其最小单位都是`method_t`结构体。**

通过源码进入`method_t`结构体：

![](http://upload-images.jianshu.io/upload_images/1319505-39d384d684d35d34.png)

method\_t 结构

由源码可知：`method_t`结构体包含三个成员变量，接下对三者进行分析：

##### 1> `SEL name;`函数名

> *   **`SEL`代表方法 / 函数名，即选择器，底层结构和`char`指针类似；**  
>     **通过`@selector()`或`sel_registerName()`获得；**  
>     **通过`sel_getName()`或`NSStringFromSelector()`将`SEL`转成字符串；**
> *   **不同类中相同名字的方法，所对应的方法选择器是相同的。**

验证如下：

```
int main(int argc, const char \* argv\[\]) {
    @autoreleasepool {
        SEL sel1 = @selector(init);
        SEL sel2 = sel\_registerName("init");
        NSLog(@"%p %p", sel1, sel2);

        const char \*str1 = sel\_getName(sel1);
        NSString \*str2 = NSStringFromSelector(sel2);
        NSLog(@"%s %@", str1, str2);
    }
    return 0;
}


Demo\[1234:567890\] 0x7fff7c4a6b8b 0x7fff7c4a6b8b
Demo\[1234:567890\] init init


```

打印的地址与方法名都相同，验证成功。

##### 2> `const char *types;`编码（返回值类型，参数类型）

再`Person`类中写一个方法：

```
@interface Person : NSObject
@end

@implementation Person
- (int)testHeight:(float)height age:(int)age {
    return 0;
}
@end


```

将`Person.m`文件转为 C++ 语言，找到`_class_ro_t`结构体，因为它存着类的初始信息：

![](http://upload-images.jianshu.io/upload_images/1319505-71cea67c125e02ae.png)

\_class\_ro\_t 源码

由以上源码可知：  
方法名为：`testHeight:age:`；  
编码`types`为：`"i24@0:8f16i20"`；  
函数地址为：`_I_Person_testHeight_age_`。

编码`types`为：`"i24@0:8f16i20"`，采用了 **iOS 的 @encode 的指令，该指令将具体的类型表示成字符串编码。**部分编码如下：

![](http://upload-images.jianshu.io/upload_images/1319505-6a9026fc430ae0f1.png)

Objective-C type encodings

我们通过该表进行分析：

```
\- (int)testHeight:(float)height age:(int)age；
types - i24@0:8f16i20
types - i    24    @    0    :    8    f    16    I    20
        int        id        SEL      float      int
       返回值      self      \_cmd      height     age

24 - 所有参数占24个字节
0 - self是从第0个字节开始存储，id类型占8个字节
8 - \_cmd是从第8个字节开始存储，SEL类型占8个字节
16 - height是从第16个字节开始存储，float类型占4个字节
20 - age是从第20个字节开始存储，int类型占4个字节


```

##### 3> `IMP imp;`指向函数的指针（函数地址）

`IMP`的内部实现如下：  
`typedef id _Nullable (*IMP)(id _Nonnull, SEL _Nonnull, ...);`

`IMP`代表函数的具体实现，存储着函数地址。

* * *

### 2\. 方法缓存`cache_t`

由 [iOS 底层原理 - OC 对象的本质（二）](https://www.jianshu.com/p/f073aee3ad20)可知：

![](http://upload-images.jianshu.io/upload_images/1319505-4daf24c898726086.png)

isa 指针、superclass 指针指向汇总图示

如果调用对象方法：

*   首先通过 instance 的`isa`指向 Class，  
    然后在 Class 对象中`class_rw_t`结构体的`methods`数组中查找方法；
*   如果 Class 对象中找不到该方法，  
    需要通过`superclass`指针找到父类的 Class 对象，  
    然后在父类的 Class 对象中`class_rw_t`结构体的`methods`数组中查找方法
*   如果父类的 Class 对象中找不到该方法，再次通过`superclass`指针找上一级的父类，如此循环。

如果一个方法需要调用许多次的话，需要循环遍历多次以上步骤；

**iOS 采用方法缓存`cache_t`，用散列表（哈希表）缓存曾经调用过的方法，可以提高方法的查找速度。**

调用方法时优先去缓存中查找此方法，缓存中没有再去类对象中查找，然后将其加入缓存中方便下次调用。

* * *

#### (1) `cache_t`缓存方式

由 **[OC 源码](https://links.jianshu.com/go?to=https%3A%2F%2Fopensource.apple.com%2Ftarballs%2Fobjc4%2F)**可以找出`cache_t`的结构：

![](http://upload-images.jianshu.io/upload_images/1319505-bf547d0e928e9266.png)

cache\_t 结构

由`cache_t`结构可知：

*   **`cache_t`结构体存储`_buckets`、`_mask`、`_occupied`；**
*   **`_buckets`散列表，存放方法选择器充当的`Key`值，以及函数的内存地址`IMP`；**
*   **`_mask`散列表的长度减 1，任何数通过与`_mask`进行按位与运算之后获得的值都会小于等于`_mask`，不会出现数组溢出的情况；**
*   **`_occupied`已经缓存的方法数量。**

`cache_t`结构可以归纳如下：

![](http://upload-images.jianshu.io/upload_images/1319505-a80f6e6ffa829cc4.png)

cache\_t 结构

接下来查看 OC 内部如何处理缓存：

![](http://upload-images.jianshu.io/upload_images/1319505-d6af8a0aa8d17087.png)

cache\_t::find 函数

"小礼物走一走，来简书关注我"

还没有人赞赏，支持一下

[![](https://upload.jianshu.io/users/upload_avatars/1319505/9754fac0eb59?imageMogr2/auto-orient/strip|imageView2/1/w/100/h/100/format/webp)](https://www.jianshu.com/u/40dce98b3416)

总资产 1 (约 0.16 元) 共写了 4.8W 字获得 24 个赞共 23 个粉丝

### 被以下专题收入，发现更多相似内容

### 推荐阅读[更多精彩内容](https://www.jianshu.com/)

*   世上有五件不可改变的事情，我们的父母不可能改变，我们不能选择我们生在谁家，父母生了我们，这是上天的恩赐，父母的生养...
    
*   \[玫瑰\]\[玫瑰\]\[玫瑰\] 亲爱的家人们，大家好，我是《马龙飞 · 情绪能量密码》经典课程六期学员周雪琴，很开心和大家一起...
    
*   这一周，没有那么想女朋友的事情，因为开发任务完成不了，十分有挫败感，我一直怀疑我是否适合通信互联网行业，但是又没有...
    
    [![](https://upload-images.jianshu.io/upload_images/9970792-204f4f756b69cada.png?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)](https://www.jianshu.com/p/f1422c2f8aae)
*   平时喜欢穿运动休闲，舒服的衣服，
    
    [![](https://cdn2.jianshu.io/assets/default_avatar/11-4d7c6ca89f439111aff57b23be1c73ba.jpg)贝玲妃](https://www.jianshu.com/u/86b08cd6069b)阅读 2
    
    [![](https://upload-images.jianshu.io/upload_images/9021438-6468c6824a24846e.jpg?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)](https://www.jianshu.com/p/f1957814b629)