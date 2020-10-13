\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[www.jianshu.com\](https://www.jianshu.com/p/54a551afd604)

[![](https://upload.jianshu.io/users/upload_avatars/1319505/9754fac0eb59?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96/format/webp)](https://www.jianshu.com/u/40dce98b3416)

0.0622019.07.22 09:50:19 字数 1,639 阅读 61

#### 面试题引发的思考：

**Q: 谈一谈对手动内存管理的理解？**

*   **在 iOS 中，使用 _引用计数_ 来管理 OC 对象的内存；**
*   **一个新创建的 OC 对象引用计数默认是 1；  
    当引用计数减为 0，OC 对象就会销毁，释放其占用的内存空间；**
*   **调用`retain`会让 OC 对象的引用计数 + 1；  
    调用`release`会让 OC 对象的引用计数 - 1；**

**Q: 内存管理的经验总结：**

*   **当调用`alloc`、`new`、`copy`、`mutableCopy`方法返回了一个对象；  
    在不需要这个对象时，要调用`release`或者`autorelease`来释放它；**
*   **想拥有某个对象，就让它的引用计数 + 1；  
    不想再拥有某个对象，就让它的引用计数 - 1；**
*   **可以通过以下私有函数来查看[自动释放池](https://www.jianshu.com/p/5fcc28ba27b5)  
    的情况`extern void _objc_autoreleasePoolPrint(void);`**

**Q: 关键字`@synthesize`和`@dynamic`区别是什么？**

*   **`@synthesize`：**  
    **编译器期间，让编译器自动生成`getter`/`setter`方法；**  
    **当有自定义的存或取方法时，自定义会屏蔽自动生成该方法。**
*   **`@dynamic`：**  
    **告诉编译器，不自动生成`getter`/`setter`方法，避免编译期间产生警告；**  
    **然后由自己实现存取方法。**

* * *

### 1\. 内存管理介绍

#### (1) 内存管理

**iOS 的内存管理一般指的是 _OC 对象_ 的内存管理：**

> *   **因为 OC 对象分配在 _堆内存_，堆内存需要程序员自己去动态分配和回收；**
> *   **基础数据类型 (非 OC 对象) 则分配在 _栈内存_ 中，超过作用域就会由系统检测回收；**
> *   **如果我们在开发过程中，对内存管理得不到位，就有可能造成内存泄露。**

* * *

#### (2) 内存管理种类

我们通常讲的内存管理分为两种：**MRC 和 ARC**。

> *   **MRC(Manual Reference Counting)：指的是手动内存管理，  
>     在开发过程中需要开发者手动去编写内存管理的代码；**
> *   **ARC(Automatic Reference Counting)：指的是自动内存管理，  
>     由 LLVM 编译器和 OC 运行时库生成相应内存管理的代码。**

本文主要介绍关于 MRC 环境下的内存管理方法。

* * *

### 2\. 代码分析

#### (1) 示例一

首先在 “TARGETS” 的“Build Setting”分类下搜索“Automatic Reference Counting”，选择“NO”，关闭 ARC，将项目改成 MRC 手动内存管理。

创建一个`Person`类：

```
@interface Person : NSObject
@end

@implementation Person
- (void)dealloc {
    NSLog(@"%s", \_\_func\_\_);
    \[super dealloc\];
}
@end


```

##### 1> 手动释放

运行以下程序，可以手动进行`release`操作：

```
int main(int argc, const char \* argv\[\]) {
    @autoreleasepool {
        Person \*person = \[\[Person alloc\] init\];
        NSLog(@"%zd", person.retainCount);
        \[person release\];
    }
    return 0;
}


Demo\[1234:567890\] 1
Demo\[1234:567890\] -\[Person dealloc\]


```

以上写法是 MRC 代码的标准写法，如果创建`person`对象之后，不执行`[person release];`语句，就会造成内存泄漏，因为该释放的对象`person`没有释放。

##### 2> 自动释放

运行以下程序，通过`autorelease`自动进行`release`操作：

```
int main(int argc, const char \* argv\[\]) {
    @autoreleasepool {
        
        Person \*person = \[\[\[Person alloc\] init\] autorelease\];
        NSLog(@"start");
    }
    NSLog(@"end");
    return 0;
}


Demo\[1234:567890\] start
Demo\[1234:567890\] -\[Person dealloc\]
Demo\[1234:567890\] end


```

自动释放：在自动释放池结束的时候，会对调用`autorelease`的对象进行`release`操作。

* * *

#### (2) 示例二

创建一个`Dog`类：

```
@interface Dog : NSObject
- (void)run;
@end

@implementation Dog
- (void)run {
    NSLog(@"%s", \_\_func\_\_);
}
- (void)dealloc {
    NSLog(@"%s", \_\_func\_\_);
    \[super dealloc\];
}
@end


```

创建一个`Person`类：

```
@interface Person : NSObject {
    Dog \*\_dog;
}
- (void)setDog:(Dog \*)dog;
- (Dog \*)dog;
@end

@implementation Person
- (void)setDog:(Dog \*)dog {
    \_dog = dog; 
}
- (Dog \*)dog {
    return \_dog;
}
- (void)dealloc {
    NSLog(@"%s", \_\_func\_\_);
    \[super dealloc\];
}
@end


```

##### 1> 情况一：

运行以下程序：

```
int main(int argc, const char \* argv\[\]) {
    @autoreleasepool {
        Dog \*dog = \[\[Dog alloc\] init\];

        Person \*person = \[\[ Person alloc\] init\];
        \[person setDog:dog\];
        \[\[person dog\] run\];

        \[dog release\];
        \[person release\];
    }
    return 0;
}


Demo\[1234:567890\] -\[Dog run\]
Demo\[1234:567890\] -\[Dog dealloc\]
Demo\[1234:567890\] -\[Person dealloc\]


```

按照顺序释放对象没有问题。

##### 2> 情况二：

接下来运行以下程序：

```
int main(int argc, const char \* argv\[\]) {
    @autoreleasepool {
        Dog \*dog = \[\[Dog alloc\] init\];

        Person \*person = \[\[ Person alloc\] init\];
        \[person setDog:dog\];

        \[dog release\];
        \[\[person dog\] run\];
        \[person release\];
    }
    return 0;
}


Demo\[1234:567890\] -\[Dog dealloc\]
Demo\[1234:567890\] -\[Dog run\]: message sent to deallocated instance 0x1007a73b0


```

程序崩溃，报错

```
Thread 1: EXC\_BAD\_INSTRUCTION (code=EXC\_I386\_INVOP, subcode=0x0)


```

坏地址访问，因为执行`[[person dog] run];`语句时`dog`对象已经销毁。

**所以要保证`person`对象没被销毁，`dog`对象也不能被销毁。**

* * *

#### (3) 代码优化版本一

##### 0> 对`Person`类进行优化：

```
@interface Person : NSObject {
    Dog \*\_dog;
}
- (void)setDog:(Dog \*)dog;
- (Dog \*)dog;
@end

@implementation Person
- (void)setDog:(Dog \*)dog {
    \_dog = \[dog retain\]; 
}
- (Dog \*)dog {
    return \_dog;
}
- (void)dealloc {
    \[\_dog release\];  
    \_dog = nil;
    
    NSLog(@"%s", \_\_func\_\_);
    
    \[super dealloc\];
}
@end


```

在`Person`的`setDog:`方法实现时，对`dog`进行引用计数 + 1；  
在`Person`的`dealloc`方法实现时，对`_dog`进行释放，引用计数 - 1。

##### 1> 情况一：

```
int main(int argc, const char \* argv\[\]) {
    @autoreleasepool {
        Dog \*dog = \[\[Dog alloc\] init\]; 
        
        Person \*person = \[\[ Person alloc\] init\];
        \[person setDog:dog\]; 
        
        \[dog release\]; 
        
        \[\[person dog\] run\];
        
        \[person release\]; 
    }
    return 0;
}


Demo\[1234:567890\] -\[Dog run\]
Demo\[1234:567890\] -\[Dog dealloc\]
Demo\[1234:567890\] -\[Person dealloc\]


```

对以上代码进行分析可知:  
对`Person`类的优化满足：**`person`对象没被销毁，`dog`对象也不能被销毁**的需求。

##### 2> 情况二：

```
int main(int argc, const char \* argv\[\]) {
    @autoreleasepool {
        Dog \*dog1 = \[\[Dog alloc\] init\]; 
        Dog \*dog2 = \[\[Dog alloc\] init\]; 
        
        Person \*person = \[\[ Person alloc\] init\];
        \[person setDog:dog1\]; 
        \[person setDog:dog2\]; 
        
        \[dog1 release\]; 
        \[dog2 release\]; 
        \[person release\]; 
    }
    return 0;
}


Demo\[1234:567890\] -\[Dog dealloc\]
Demo\[1234:567890\] -\[Person dealloc\]


```

对以上代码进行分析可知:  
对`Person`类的优化不满足：**`person`对象持有`Dog`类创建的多个`dog`对象情况下，`person`对象没被销毁，`dog`对象也不能被销毁**的需求。

* * *

#### (4) 代码优化版本二

##### 0> 对`Person`类进行优化：

```
@interface Person : NSObject {
    Dog \*\_dog;
}
- (void)setDog:(Dog \*)dog;
- (Dog \*)dog;
@end

@implementation Person
- (void)setDog:(Dog \*)dog {
    \[\_dog release\]; 
    \_dog = \[dog retain\]; 
}
- (Dog \*)dog {
    return \_dog;
}
- (void)dealloc {
    \[\_dog release\];  
    \_dog = nil;
    
    NSLog(@"%s", \_\_func\_\_);
    
    \[super dealloc\];
}
@end


```

在`Person`的`setDog:`方法实现时，先释放以前的`_dog`，然后对`dog`进行引用计数 + 1；  
在`Person`的`dealloc`方法实现时，对`_dog`进行释放，引用计数 - 1。

##### 1> 情况一：

```
int main(int argc, const char \* argv\[\]) {
    @autoreleasepool {
        Dog \*dog1 = \[\[Dog alloc\] init\]; 
        Dog \*dog2 = \[\[Dog alloc\] init\]; 
        
        Person \*person = \[\[ Person alloc\] init\];
        \[person setDog:dog1\]; 
        \[person setDog:dog2\]; 
        
        \[dog1 release\]; 
        \[dog2 release\]; 
        \[person release\]; 
    }
    return 0;
}


Demo\[1234:567890\] -\[Dog dealloc\]
Demo\[1234:567890\] -\[Dog dealloc\]
Demo\[1234:567890\] -\[Person dealloc\]


```

对以上代码进行分析可知:  
对`Person`类的优化满足：**`person`对象持有`Dog`类创建的多个`dog`对象情况下，`person`对象没被销毁，`dog`对象也不能被销毁**的需求。

##### 2> 情况二：

```
int main(int argc, const char \* argv\[\]) {
    @autoreleasepool {
        Dog \*dog = \[\[Dog alloc\] init\]; 
        
        Person \*person = \[\[ Person alloc\] init\];
        \[person setDog:dog\]; 
        
        \[dog release\]; 
        
        \[person setDog:dog\]; 
        \[person setDog:dog\];
        
        \[person release\];
    }
    return 0;
}


```

程序崩溃，报错:

```
Thread 1: EXC\_BAD\_INSTRUCTION (code=EXC\_I386\_INVOP, subcode=0x0)


```

以上代码进行分析可知:  
对`Person`类的优化不满足：**`person`对象持有同一个`dog`对象情况下，`person`对象没被销毁，`dog`对象也不能被销毁**的需求。

* * *

#### (5) 代码优化版本三

##### 0> 对`Person`类进行优化：

```
@interface Person : NSObject {
    Dog \*\_dog;
}
- (void)setDog:(Dog \*)dog;
- (Dog \*)dog;
@end

@implementation Person
- (void)setDog:(Dog \*)dog {
    if (\_dog != dog) { 
        \[\_dog release\]; 
        \_dog = \[dog retain\]; 
    }
}
- (Dog \*)dog {
    return \_dog;
}
- (void)dealloc {


    self.dog = nil;
    
    NSLog(@"%s", \_\_func\_\_);
    
    \[super dealloc\];
}
@end


```

在`Person`的`setDog:`方法实现时，判断`dog`与`_dog`不相等时，进行以下操作  
：先释放以前的`_dog`，然后对`dog`进行引用计数 + 1；  
在`Person`的`dealloc`方法实现时，对`_dog`进行释放，引用计数 - 1。

##### 1> 情况一：

```
int main(int argc, const char \* argv\[\]) {
    @autoreleasepool {
        Dog \*dog = \[\[Dog alloc\] init\]; 
        
        Person \*person = \[\[ Person alloc\] init\];
        \[person setDog:dog\]; 
        
        \[dog release\]; 
        
        \[person setDog:dog\];
        \[person setDog:dog\];
        
        \[person release\]; 
    }
    return 0;
}


Demo\[1234:567890\] -\[Dog dealloc\]
Demo\[1234:567890\] -\[Person dealloc\]


```

以上代码进行分析可知:  
对`Person`类的优化满足：**`person`对象持有同一个`dog`对象情况下，`person`对象没被销毁，`dog`对象也不能被销毁**的需求。

* * *

### 3\. MRC 下关键字使用

#### (1) 关键字`@synthesize`和`@dynamic`

> *   **MRC 下使用`@property`只会生成`setter`、`getter`方法的声明；**
> *   **如果想生成`_age`成员变量和`setter`、`getter`方法的实现还要使用`@synthesize`关键字。**

以下是 MRC 环境下两个关键字的使用方法：

```
@interface Person : NSObject

@property(nonatomic, assign) int age;
@property(nonatomic, retain) Dog \*dog;
@end

@implementation Person

@synthesize age = \_age;
@synthesize dog = \_dog;

- (void)setAge:(int)age {
    \_age = age;
}
- (int)age {
    return \_age;
}

- (void)setDog:(Dog \*)dog {
    if (\_dog != dog) {
        \[\_dog release\];
        \_dog = \[dog retain\];
    }
}
- (Dog \*)dog {
    return \_dog;
}

- (void)dealloc {
    \[\_dog release\];
    \_dog = nil;
    \[super dealloc\];
}
@end


```

> *   **使用`assign`修饰生成的`setter`方法没有内存管理相关的东西，所以`assign`一般用来修饰基本数据类型；**
> *   **使用`retain`修饰生成的`setter`方法有内存管理相关的东西，所以`retain`一般用来修饰对象类型。**

#### (2) 简略写法

```
@interface Person : NSObject

@property(nonatomic, assign) int age;
@property(nonatomic, retain) Dog \*dog;

+ (instancetype)person;
@end

@implementation Person
+ (instancetype)person {
    return \[\[\[self alloc\] init\] autorelease\];
}
- (void)dealloc {
    self.dog = nil;
    \[super dealloc\];
}
@end


int main(int argc, const char \* argv\[\]) {
    @autoreleasepool {
        Person \*person = \[Person person\];
    }
    return 0;
}


```

MRC 环境下，一般会给类添加一个工厂方法，可以自动释放对象，操作简单。

* * *

### 4\. MRC 下代码写法

```
@interface ViewController ()
@property(nonatomic, retain) NSMutableArray \*array;
@end

@implementation ViewController
- (void)viewDidLoad {
    \[super viewDidLoad\];

    
    NSMutableArray \*array = \[\[NSMutableArray alloc\] init\];
    self.array = array;
    \[array release\];
    
    
    self.array = \[\[NSMutableArray alloc\] init\];
    \[self.array release\];
    
    
    self.array = \[\[\[NSMutableArray alloc\] init\] autorelease\];
    
    
    
    self.array = \[NSMutableArray array\];
}
- (void)dealloc {
    self.array = nil;
    \[super dealloc\];
}
@end


```

"小礼物走一走，来简书关注我"

还没有人赞赏，支持一下

[![](https://upload.jianshu.io/users/upload_avatars/1319505/9754fac0eb59?imageMogr2/auto-orient/strip|imageView2/1/w/100/h/100/format/webp)](https://www.jianshu.com/u/40dce98b3416)

总资产 1 (约 0.16 元) 共写了 4.8W 字获得 24 个赞共 23 个粉丝

### 被以下专题收入，发现更多相似内容

### 推荐阅读[更多精彩内容](https://www.jianshu.com/)

*   本文参考《Mac OS X and iOS Internals: To the Apple’s Core》 by ...
    
*   一. 定时器 1.CADisplayLink、NSTimer CADisplayLink、NSTimer 会对 ta...
    
    [![](https://upload-images.jianshu.io/upload_images/103735-a4082e3b5cc314f0.png?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)](https://www.jianshu.com/p/9e26c6ac2500)
*   Swift1> Swift 和 OC 的区别 1.1> Swift 没有地址 / 指针的概念 1.2> 泛型 1.3> 类型严谨 对...
    
*   1\. 下面代码执行结果如何 运行结果 分析: 因为 data 是 copy 属性，所以在其 set 方法里先执行判断，然后执行 re...
    
    [![](https://upload-images.jianshu.io/upload_images/1653926-7644658f002fe273.png?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)](https://www.jianshu.com/p/869ca6fbdd5c)
*   CADisplayLink、NSTimer 使用注意点: 1.CADisplayLink、NSTimer 会对 targ...
    
    [![](https://upload-images.jianshu.io/upload_images/2163717-afee26101f38992e.png?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)](https://www.jianshu.com/p/2f578b91083c)