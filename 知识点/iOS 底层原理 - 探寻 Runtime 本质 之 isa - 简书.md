\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[www.jianshu.com\](https://www.jianshu.com/p/18cd64b16eb4)

[![](https://upload.jianshu.io/users/upload_avatars/1319505/9754fac0eb59?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96/format/webp)](https://www.jianshu.com/u/40dce98b3416)

2018.04.08 15:05:12 字数 1,768 阅读 87

#### 面试题引发的思考：

**Q: 简述`isa`指针？**

*   从`__arm64__`架构开始，`isa`指针：
*   **不只是存储了 _class 对象_ 或 _meta-class 对象_ 的地址；**
*   **而是使用 _共用体结构_ 存储了更多信息：**  
    **其中 `shiftcls` 存储了 _class 对象_ 或 _meta-class 对象_ 的地址；**  
    **需要 `shiftcls` 同 `ISA_MASK` 进行 `按位&` 运算取出。**

* * *

> Objective-C 扩展了 C 语言，并加入了面向对象特性和 Smalltalk 式的消息传递机制。而这个扩展的核心是一个用 C 和 编译语言 写的 Runtime 库。它是 Objective-C 面向对象和动态机制的基石。

要想学习 Runtime，首先要了解它底层的一些常用数据结构，比如`isa`指针：

由 [iOS 底层原理 - OC 对象的本质（二）](https://www.jianshu.com/p/f073aee3ad20)可知：

> *   **每个 OC 对象都有一个`isa`指针；**
> *   **instance 对象的`isa`指向 class 对象；**
> *   **class 对象的`isa`指向 meta-class 对象；**
> *   **OC 对象的`isa`指针并不是直接指向 class 对象或者 meta-class 对象，而是需要`&ISA_MASK`通过位运算才能获取到类对象或者元类对象的地址；**  
>     **a> 在`__arm64__`架构之前，`isa`就是一个普通的指针，存储着 Class 对象、Meta-Class 对象的内存地址；**  
>     **b> 从`__arm64__`架构开始，对`isa`进行了优化，变成了一个共用体 (`union`) 结构，还使用位域来存储更多的信息。**

进入 [OC 源码](https://links.jianshu.com/go?to=https%3A%2F%2Fopensource.apple.com%2Ftarballs%2Fobjc4%2F)查看`isa`指针，进行更深入的了解：

![](http://upload-images.jianshu.io/upload_images/1319505-2f1bad1bd337dd7d.png)

isa 结构

由上图可知：  
**`isa`指针其实是一个`isa_t`类型的共用体；  
此共用体中包含一个结构体；  
此结构体内部定义了若干变量，变量后面的值则表示该变量占用的位数，即位域。**

接下来一步步分析共用体的优势。

* * *

#### (1) 探究

```
@interface Person : NSObject

@property (nonatomic, assign, getter=isTall) BOOL tall;
@property (nonatomic, assign, getter=isRich) BOOL rich;
@property (nonatomic, assign, getter=isHandsome) BOOL handsome;
@end

@implementation Person
@end


int main(int argc, const char \* argv\[\]) {
    @autoreleasepool {
        Person \*person  = \[\[Person alloc\] init\];
        person.tall = YES;
        person.rich = NO;
        person.handsome = YES;
        NSLog(@"%zd", class\_getInstanceSize(\[Person class\]));
     }
    return 0;
}


Demo\[1234:567890\] 16


```

由以上代码可知：  
`isa`指针占 8 个字节，3 个`BOOL`类型的分别占 1 个字节，一共 11 个字节。由于内存对齐原则，所以`Person`类对象所占内存应该为 16 个字节。

`BOOL`值包含`0`或`1`，只需要 1 个二进制位就可以表示出来，而现在需要占用 1 个字节共 8 个二进制位。  
可以使用 1 个字节中的 3 个二进制位表示 3 个`BOOL`值，这样可以很大程度上节省内存空间。

如果声明属性，系统就会自动生成成员变量，占用的内存会大于 1 位；所以不可以声明属性，需要手动实现每个属性的`set`方法和`get`方法。

现在添加一个`char`类型的成员变量，`char`变量占用一个字节 (8 位) 的内存空间，使用最后 3 位来存储 3 个`BOOL`值。

![](http://upload-images.jianshu.io/upload_images/1319505-2e652a670c0b4eb4.png)

1 个 char 值存储 3 个 BOOL 值

* * *

#### (2) 位运算

1.  **补码（负数是以补码的形式表示）**  
    **十进制正整数转换为二进制数：除`2`取余即可；**  
    **十进制负整数转换为二进制数：除`2`取余，取反加`1`；**

```
0b0000 0000 0000 1010 -（10除2取余）
0b1111 1111 1111 0101 -（取反）
0b1111 1111 1111 0110 -（加1）
0b1111 1111 1111 0110 -（得出-10的二进制）


```

2.  **按位与（&）**  
    **同真为真，其余为假：清零特定位、取出特定位；**

```
  0b0000 1111 0000 1111
& 0b0000 0000 1111 1111
  ---------------------
  0b0000 0000 0000 1111


```

3.  **按位或（|）**  
    **同假为假，其余为真：特定位置`1`；**

```
  0b0000 1111 0000 1111
| 0b0000 0000 1111 1111
  ---------------------
  0b0000 1111 1111 1111


```

4.  **按位异或（^）**  
    **异值为真，同值为假：特定位取反、交换两变量的值；**

```
  0b0000 1111 0000 1111
^ 0b0000 0000 1111 1111
  ---------------------
  0b0000 1111 1111 0000


```

5.  **取反（~）**  
    **按位取反**

```
~ 0b0000 1111 0000 1111
  ---------------------
  0b1111 0000 1111 0000


```

6.  **左移（<<）**  
    **按位左移，高位丢弃，低位补`0`；**

```
  0b1111 0000 0000 1111 << 2
  ---------------------
  0b1100 0000 0011 1100


```

7.  **右移（>>）**  
    **按位右移，高位正数补`0`，负数补`1`；**

```
  0b1111 0000 0000 1111 >> 2
  ---------------------
  0b0011 1100 0000 0011


```

* * *

#### (3) 手动实现属性的`set`方法和`get`方法

```
#define TallMask (1<<0)
#define RichMask (1<<1)
#define HandsomeMask (1<<2)

@interface Person : NSObject {
    char \_tallRichHandsome;  
}
- (void)setTall:(BOOL)tall;
- (void)setRich:(BOOL)rich;
- (void)setHandsome:(BOOL)handsome;
- (BOOL)isTall;
- (BOOL)isRich;
- (BOOL)isHandsome;
@end

@implementation Person
- (void)setTall:(BOOL)tall {
    if (tall) { 
        \_tallRichHandsome |= TallMask;
    } else { 
        \_tallRichHandsome &= ~TallMask;
    }
}
- (void)setRich:(BOOL)rich {
    if (rich) {
        \_tallRichHandsome |= RichMask;
    } else {
        \_tallRichHandsome &= ~RichMask;
    }
}
- (void)setHandsome:(BOOL)handsome {
    if (handsome) {
        \_tallRichHandsome |= HandsomeMask;
    } else {
        \_tallRichHandsome &= ~HandsomeMask;
    }
}
- (BOOL)isTall {
    
    
    return !!(\_tallRichHandsome & TallMask);
}
- (BOOL)isRich {
    return !!(\_tallRichHandsome & RichMask);
}
- (BOOL)isHandsome {
    return !!(\_tallRichHandsome & HandsomeMask);
}
@end


int main(int argc, const char \* argv\[\]) {
    @autoreleasepool {
        Person \*person  = \[\[Person alloc\] init\];
        person.tall = YES;
        person.rich = NO;
        person.handsome = YES;
        NSLog(@"tall: %d, rich: %d, handsome: %d", person.isTall, person.isRich, person.isHandsome);
    }
    return 0;
}


Demo\[1234:567890\] tall: 1, rich: 0, handsome: 1


```

上述代码可以正常存值和取值，但是可拓展性和可读性差。

* * *

#### (4) 位域

> **位域声明：`位域名 : 位域长度`**
> 
> **使用位域需要注意以下 3 点：**
> 
> *   **如果一个字节所剩空间不够存放另一位域时，应从下一单元起存放该位域；  
>     也可以有意使某位域从下一单元开始。**
> *   **位域的长度不能大于数据类型本身的长度；  
>     比如`int`类型就不能超过 32 位二进制位。**
> *   **位域可以无位域名，这时它只用来作填充或调整位置；  
>     无名的位域是不能使用的。**

```
@interface Person : NSObject {
    
    struct {
        char tall : 1;
        char rich : 1;
        char handsome : 1;
    } \_tallRichHandsome;
}
- (void)setTall:(BOOL)tall;
- (void)setRich:(BOOL)rich;
- (void)setHandsome:(BOOL)handsome;
- (BOOL)isTall;
- (BOOL)isRich;
- (BOOL)isHandsome;
@end

@implementation Person
- (void)setTall:(BOOL)tall {
    \_tallRichHandsome.tall = tall;
}
- (void)setRich:(BOOL)rich {
    \_tallRichHandsome.rich = rich;
}
- (void)setHandsome:(BOOL)handsome {
    \_tallRichHandsome.handsome = handsome;
}
- (BOOL)isTall {
    
    
    return !!\_tallRichHandsome.tall;
}
- (BOOL)isRich {
    return !!\_tallRichHandsome.rich;
}
- (BOOL)isHandsome {
    return !!\_tallRichHandsome.handsome;
}
@end


Demo\[1234:567890\] tall: 1, rich: 0, handsome: 1


```

上述代码使用结构体的位域，不需要再使用掩码，缺点是效率比使用位运算时低。

* * *

#### (5) 共用体

```
#define TallMask (1<<0)
#define RichMask (1<<1)
#define HandsomeMask (1<<2)

@interface Person : NSObject {
    union { 
        char bits;
        
        struct {
            char tall : 1;
            char rich : 1;
            char handsome : 1;
        };
    }\_tallRichHandsome;
}
- (void)setTall:(BOOL)tall;
- (void)setRich:(BOOL)rich;
- (void)setHandsome:(BOOL)handsome;
- (BOOL)isTall;
- (BOOL)isRich;
- (BOOL)isHandsome;
@end

@implementation Person
- (void)setTall:(BOOL)tall {
    if (tall) {
        \_tallRichHandsome.bits |= TallMask;
    } else {
        \_tallRichHandsome.bits &= ~TallMask;
    }
}
- (void)setRich:(BOOL)rich {
    if (rich) {
        \_tallRichHandsome.bits |= RichMask;
    } else {
        \_tallRichHandsome.bits &= ~RichMask;
    }
}
- (void)setHandsome:(BOOL)handsome {
    if (handsome) {
        \_tallRichHandsome.bits |= HandsomeMask;
    } else {
        \_tallRichHandsome.bits &= ~HandsomeMask;
    }
}
- (BOOL)isTall {
    
    return !!(\_tallRichHandsome.bits & TallMask);
}
- (BOOL)isRich {
    return !!(\_tallRichHandsome.bits & RichMask);
}
- (BOOL)isHandsome {
    return !!(\_tallRichHandsome.bits & HandsomeMask);
}
@end


Demo\[1234:567890\] tall: 1, rich: 0, handsome: 1


```

上述代码使用共用体对数据进行位运算取值和赋值，效率高同时占用内存少，代码可读性高。

* * *

#### (6) `isa`详解

经过以上分析，我们可以清晰地了解到位运算、位域以及共用体的相关知识。  
下面我们深入分析`isa`的结构：

![](http://upload-images.jianshu.io/upload_images/1319505-2f1bad1bd337dd7d.png)

isa 结构

由上图可知：  
`isa_t`共用体存储了 8 个字节 64 位的值，所有信息都存储在`bits`中，这些值在结构体中展现出来，这些值通过对`bits`进行掩码位运算取出。

<table><thead><tr><th>名称</th><th>作用</th></tr></thead><tbody><tr><td><code>nonpointer</code></td><td><code>0</code>代表普通的指针，存储着 Class，Meta-Class 对象的内存地址；<code>1</code>代表优化过，使用位域存储更多的信息</td></tr><tr><td><code>has_assoc</code></td><td>是否有设置过关联对象，如果没有，释放时会更快</td></tr><tr><td><code>has_cxx_dtor</code></td><td>是否有 C++ 析构函数，如果没有，释放时会更快</td></tr><tr><td><code>shiftcls</code></td><td>存储着 Class、Meta-Class 对象的内存地址信息</td></tr><tr><td><code>magic</code></td><td>用于在调试时分辨对象是否未完成初始化</td></tr><tr><td><code>weakly_referenced</code></td><td>是否有被弱引用指向过，如果没有，释放时会更快</td></tr><tr><td><code>deallocating</code></td><td>对象是否正在释放</td></tr><tr><td><code>has_sidetable_rc</code></td><td>引用计数器是否过大无法存储在<code>isa</code>中；如果为<code>1</code>，那么引用计数会存储在一个叫<code>SideTable</code>的类的属性中</td></tr><tr><td><code>extra_rc</code></td><td>里面存储的值是引用计数器减<code>1</code></td></tr></tbody></table>

重点介绍一下`shiftcls`：

> *   **`shiftcls`存储着 Class、Meta-Class 对象的内存地址信息；**
> *   **通过`bits & ISA_MASK`取得`shiftcls`的值；**
> *   **而掩码`ISA_MASK`值为`0x0000000ffffffff8ULL`，说明 Class、Meta-Class 对象的内存地址值后三位一定为`0`。**

接下来验证一下：

```
int main(int argc, const char \* argv\[\]) {
    @autoreleasepool {
        Person \*person  = \[\[Person alloc\] init\];
        person.tall = YES;
        person.rich = NO;
        person.handsome = YES;
        NSLog(@"tall: %d, rich: %d, handsome: %d", person.isTall, person.isRich, person.isHandsome);
        NSLog(@"class: %p, meta-class: %p", \[Person class\], object\_getClass(\[Person class\]));
    }
    return 0;
}

Demo\[1234:567890\] tall: 1, rich: 0, handsome: 1
Demo\[1234:567890\] class: 0x100001260, meta-class: 0x100001238


```

由打印结果可知：  
class 对象的地址值为`0x100001260`，尾数是`0`；meta-class 对象的地址值为`0x100001238`，尾数是`8`；在 16 进制下，内存地址的后三位都为`0`；验证结束。

* * *

#### 总结可知：

> *   **从`__arm64__`架构开始，`isa`指针不只是存储了 class 对象或 meta-class 对象的地址；**
> *   **而是使用共用体结构存储了更多信息：**  
>     **其中`shiftcls`存储了 class 或 meta-class 的地址；**  
>     **`shiftcls`需要同`ISA_MASK`进行`按位&`运算才可以取出 class 对象或 meta-class 对象的地址。**

"小礼物走一走，来简书关注我"

还没有人赞赏，支持一下

[![](https://upload.jianshu.io/users/upload_avatars/1319505/9754fac0eb59?imageMogr2/auto-orient/strip|imageView2/1/w/100/h/100/format/webp)](https://www.jianshu.com/u/40dce98b3416)

总资产 1 (约 0.16 元) 共写了 4.8W 字获得 24 个赞共 23 个粉丝

### 被以下专题收入，发现更多相似内容

### 推荐阅读[更多精彩内容](https://www.jianshu.com/)

*   23 岁的第一天 我呆在酒店的沙发上一动不动整理了快四个小时的照片。 回头看 22 岁时候的日志觉得自己好惨哦 尤其是十...
    
*   昨天晚上是小十一出生以来哭得最凶的一次，双眼充满了恐惧，浑身发抖，哭得声嘶力竭，近乎一个小时，最后终于哭得太累，紧...
    
    [![](https://upload.jianshu.io/users/upload_avatars/7587613/9ecc5359-925c-41c2-ab40-d9c1260d82c4.jpg?imageMogr2/auto-orient/strip|imageView2/1/w/48/h/48/format/webp)小十一娘](https://www.jianshu.com/u/ea7d2e2d0d2c)阅读 0
    
*   位操作详解 位运算的操作符有：&、|、^、~、>>、<<，六种，分别对应与，或，异或，按位取反，右位移，左位移 1...