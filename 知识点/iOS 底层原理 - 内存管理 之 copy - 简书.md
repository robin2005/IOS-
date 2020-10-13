\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[www.jianshu.com\](https://www.jianshu.com/p/4fcf739f62cd)

[![](https://upload.jianshu.io/users/upload_avatars/1319505/9754fac0eb59?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96/format/webp)](https://www.jianshu.com/u/40dce98b3416)

2019.08.20 08:55:09 字数 1,281 阅读 111

#### 面试题引发的思考：

**Q: 谈一谈对`copy`、`mutableCopy`、浅拷贝、深拷贝的理解？**

*   **`copy`：不可变拷贝，产生不可变副本；**
*   **`mutableCopy`：可变拷贝，产生可变副本；**
*   **浅拷贝：指针拷贝，没有产生新的对象；**
*   **深拷贝：内容拷贝，产生新的对象。**

**Q: iOS 项目中`copy`修饰词的使用？**

*   **字符串一般用`copy`修饰，用于 UI 控件显示的时候不会有问题。**
*   **属性不存在`mutableCopy`修饰词，只有部分`Foundation`框架自带的一些类可以用`mutableCopy`修饰。**

**Q: 自定义的类实现 `copy` 功能？**

*   **遵守 `NSCopying` 协议；**
*   **实现 `copyWithZone:` 方法。**

* * *

### 1\. 原理介绍

**拷贝的目的：产生一个副本对象，跟源对象互不影响**

> *   **修改了源对象，不会影响副本对象；**
> *   **修改了副本对象，不会影响源对象。**

**iOS 提供了 2 个拷贝方法：**

> *   **`copy`：不可变拷贝，产生不可变副本；**
> *   **`mutableCopy`：可变拷贝，产生可变副本。**

**浅拷贝和深拷贝：**

> *   **浅拷贝：指针拷贝，没有产生新的对象；**  
>     **不可变对象`NSString`、`NSArray`、`NSDictionary`调用`copy`；**
> *   **深拷贝：内容拷贝，产生新的对象；**  
>     **深拷贝：其他情况。**

* * *

### 2\. 案例分析

#### (1) `copy`和`mutableCopy`分析

```
int main(int argc, const char \* argv\[\]) {
    @autoreleasepool {
        
        NSString \*str1 = \[NSString stringWithFormat:@"test"\];
        NSString \*str2 = \[str1 copy\]; 
        NSMutableString \*str3 = \[str1 mutableCopy\]; 

        \[str3 appendString:@"ceshi"\];

        NSLog(@"%@ %@ %@", str1, str2, str3);
    }
    return 0;
}


Demo\[1234:567890\] test test testceshi


```

```
int main(int argc, const char \* argv\[\]) {
    @autoreleasepool {
        
        NSMutableString \*str1 = \[NSMutableString stringWithFormat:@"test"\];
        NSString \*str2 = \[str1 copy\]; 
        NSMutableString \*str3 = \[str1 mutableCopy\]; 

        \[str3 appendString:@"ceshi"\];

        NSLog(@"%@ %@ %@", str1, str2, str3);
    }
    return 0;
}


Demo\[1234:567890\] test test testceshi


```

由以上代码可知：

> *   **`copy`：不可变拷贝，产生不可变副本；**
> *   **`mutableCopy`：可变拷贝，产生可变副本。**

* * *

#### (2) 浅拷贝和深拷贝分析

将环境改为手动内存管理，我们知道：

> **当调用`alloc`、`new`、`copy`、`mutableCopy`方法返回了一个对象，在不需要这个对象时，要调用`release`或者`autorelease`来释放它。**

```
int main(int argc, const char \* argv\[\]) {
    @autoreleasepool {
        
        
        NSString \*str1 = \[\[NSString alloc\] initWithFormat:@"test123456789"\];
        NSString \*str2 = \[str1 copy\]; 
        NSMutableString \*str3 = \[str1 mutableCopy\]; 

        NSLog(@"%@ %@ %@", str1, str2, str3);
        NSLog(@"%p %p %p", str1, str2, str3);

        \[str3 release\];
        \[str2 release\];
        \[str1 release\];
    }
    return 0;
}


Demo\[1234:567890\] test123456789 test123456789 test123456789
Demo\[1234:567890\] 0x1005b9b60 0x1005b9b60 0x100604a40


```

由打印结果可知：  
`str1`、`str2`的地址是一样的，说明`str1`、`str2`是指向的是同一个对象。其指针分析如下：

![](http://upload-images.jianshu.io/upload_images/1319505-7f0938d4bdea83ad.png)

指针分析

```
int main(int argc, const char \* argv\[\]) {
    @autoreleasepool {
        
        NSMutableString \*str1 = \[\[NSMutableString alloc\] initWithFormat:@"test123456789"\];
        NSString \*str2 = \[str1 copy\]; 
        NSMutableString \*str3 = \[str1 mutableCopy\]; 

        \[str1 appendFormat:@"111"\];
        \[str3 appendFormat:@"333"\];

        NSLog(@"%@ %@ %@", str1, str2, str3);
        NSLog(@"%p %p %p", str1, str2, str3);

        \[str3 release\];
        \[str2 release\];
        \[str1 release\];
    }
    return 0;
}


Demo\[1234:567890\] test123456789111 test123456789 test123456789333
Demo\[1234:567890\] 0x10049d9b0 0x10049ded0 0x10049dfd0


```

由打印结果可知：  
`str1`、`str2`、`str3`的地址都是不一样的，说明`str1`、`str2`、`str3`是分别指向的是不同的对象。其指针分析如下：

![](http://upload-images.jianshu.io/upload_images/1319505-eb0444d0f929ca2c.png)

指针分析

* * *

#### (3) 扩展分析

##### 1> 对于数组

```
int main(int argc, const char \* argv\[\]) {
    @autoreleasepool {
        
        NSArray \*array1 = \[\[NSArray alloc\] initWithObjects:@"1", @"2", nil\];
        NSArray \*array2 = \[array1 copy\]; 
        NSMutableArray \*array3 = \[array1 mutableCopy\]; 

        NSLog(@"%p %p %p", array1, array2, array3);

        \[array1 release\];
        \[array2 release\];
        \[array3 release\];
    }
    return 0;
}


Demo\[1234:567890\] 0x10049eca0 0x10049eca0 0x10049ef50


```

由打印结果可知：  
`array1`、`array2`的地址是一样的，说明`array1`、`array2`是指向的是同一个对象。

```
int main(int argc, const char \* argv\[\]) {
    @autoreleasepool {
        
        NSMutableArray \*array1 = \[\[NSMutableArray alloc\] initWithObjects:@"1", @"2", nil\];
        NSArray \*array2 = \[array1 copy\]; 
        NSMutableArray \*array3 = \[array1 mutableCopy\]; 

        NSLog(@"%p %p %p", array1, array2, array3);

        \[array1 release\];
        \[array2 release\];
        \[array3 release\];
    }
    return 0;
}


Demo\[1234:567890\] 0x10313c020 0x10313c700 0x10313c720


```

由打印结果可知：  
`array1`、`array2`、`array3`的地址都是不一样的，说明`array1`、`array2`、`array3`是分别指向的是不同的对象。

##### 2> 对于字典

```
int main(int argc, const char \* argv\[\]) {
    @autoreleasepool {
        
        NSDictionary \*dict1 = \[\[NSDictionary alloc\] initWithObjectsAndKeys:@"jack", @"name", nil\];
        NSDictionary \*dict2 = \[dict1 copy\]; 
        NSMutableDictionary \*dict3 = \[dict1 mutableCopy\]; 

        NSLog(@"%p %p %p", dict1, dict2, dict3);

        \[dict1 release\];
        \[dict2 release\];
        \[dict3 release\];
    }
    return 0;
}


Demo\[1234:567890\] 0x100511810 0x100511810 0x1005118f0


```

由打印结果可知：  
`dict1`、`dict2`的地址是一样的，说明`dict1`、`dict2`是指向的是同一个对象。

```
int main(int argc, const char \* argv\[\]) {
    @autoreleasepool {
        
        NSMutableDictionary \*dict1 = \[\[NSMutableDictionary alloc\] initWithObjectsAndKeys:@"jack", @"name", nil\];
        NSDictionary \*dict2 = \[dict1 copy\]; 
        NSMutableDictionary \*dict3 = \[dict1 mutableCopy\]; 

        NSLog(@"%p %p %p", dict1, dict2, dict3);

        \[dict1 release\];
        \[dict2 release\];
        \[dict3 release\];
    }
    return 0;
}


Demo\[1234:567890\] 0x103132250 0x103132850 0x103132870


```

由打印结果可知：  
`dict1`、`dict2`、`dict3`的地址都是不一样的，说明`dict1`、`dict2`、`dict3`是分别指向的是不同的对象。

* * *

#### (4) 总结

根据以上代码及分析可总结如下：

> *   **`copy`：不可变拷贝，产生不可变副本；**
> *   **`mutableCopy`：可变拷贝，产生可变副本；**
> *   **浅拷贝：指针拷贝，没有产生新的对象；**
> *   **深拷贝：内容拷贝，产生新的对象；**
> *   **不可变对象`NSString`、`NSArray`、`NSDictionary`调用`copy`时是浅拷贝；**
> *   **其他情况是深拷贝。**

总结图例如下：

![](http://upload-images.jianshu.io/upload_images/1319505-607746d29437923d.png)

总结图例

* * *

### 3\. 修饰词`copy`的使用分析

#### (1) 总结修饰词`copy`和`retain`的区别

对于`retain`修饰词修饰：

```
@interface Person : NSObject
@property(nonatomic, retain) NSArray \*dataArray;
@end

@implementation Person

- (void)setDataArray:(NSArray \*)dataArray {
    if (\_dataArray != dataArray) {
        \[\_dataArray release\];
        \_dataArray = \[dataArray retain\]; 
    }
}
- (void)dealloc {
    self.dataArray = nil;
    \[super dealloc\];
}
@end


```

对于`copy`修饰词修饰：

```
@interface Person : NSObject
@property(nonatomic, copy) NSArray \*dataArray;
@end

@implementation Person

- (void)setDataArray:(NSArray \*)dataArray {
    if (\_dataArray != dataArray) {
        \[\_dataArray release\];
        \_dataArray = \[dataArray copy\]; 
    }
}
- (void)dealloc {
    self.dataArray = nil;
    \[super dealloc\];
}
@end


```

因为用`copy`修饰词修饰的`dataArray`，会先把旧的`_dataArray`先`release`，然后调用`copy`赋值给`_dataArray`，那么`_dataArray`就是不可变类型。

得出结论：

> **`copy`修饰词只能声明不可变类型。**

* * *

#### (2) 案例分析

##### 1> 对于`copy`修饰词修饰不可变数组`NSArray`：

```
@interface Person : NSObject
@property(nonatomic, copy) NSArray \*dataArray;
@end

@implementation Person
@end


int main(int argc, const char \* argv\[\]) {
    @autoreleasepool {
        
        Person \*person = \[\[Person alloc\] init\];

        person.dataArray = @\[@"jack", @"rose"\];
        
        

        NSLog(@"%@", person.dataArray);

        \[person release\];
    }
    return 0;
}


Demo\[1234:567890\] (
    jack,
    rose
)


```

运行正常，没有问题。

##### 2> 对于`copy`修饰词修饰可变数组`NSMutableArray`：

```
@interface Person : NSObject
@property(nonatomic, copy) NSMutableArray \*dataArray;
@end

@implementation Person
@end


int main(int argc, const char \* argv\[\]) {
    @autoreleasepool {
        
        Person \*person = \[\[Person alloc\] init\];

        person.dataArray = \[NSMutableArray array\];
        

        \[person.dataArray addObject:@"jack"\];
        \[person.dataArray addObject:@"rose"\];

        NSLog(@"%@", person.dataArray);

        \[person release\];
    }
    return 0;
}


Demo\[1234:567890\] '-\[\_\_NSArray0 addObject:\]: unrecognized selector sent to instance 0x7fff898d1060'


```

运行崩溃，提示 “方法找不到错误”。

因为**`可变数组`**`dataArray`用`copy`修饰词修饰，语句`person.dataArray = [NSMutableArray array];`相当于**`可变的数组`**赋值给`person.dataArray`，然后调用`copy`返回一个**`不可变数组`**，所以**`不可变数组`**`person.dataArray`没有`addObject:`方法，运行崩溃。

* * *

#### (3) iOS 项目中`copy`修饰词的使用

##### 1> UI 控件显示使用`copy`修饰

对于`UITextField`的`text`属性内部代码如下：

![](http://upload-images.jianshu.io/upload_images/1319505-e394c906e78e382d.png)

UITextField 属性

我们发现 iOS 底层代码对于字符串相关的基本上都是使用`copy`修饰。

具体事例分析如下：

```
\- (void)viewDidLoad {
    \[super viewDidLoad\];
    
    NSMutableString \*text = \[NSMutableString stringWithFormat:@"123"\];

    UITextField \*textField;
    
    textField.text = text;

    
    \[text appendString:@"abc"\];
}


```

由以上分析可知：

> **字符串一般用`copy`修饰，用于 UI 控件显示的时候不会有问题。**

属性不存在`mutableCopy`修饰词，因为只有部分`Foundation`框架自带的一些类可以用`mutableCopy`修饰，比如以下类：

```
NSString, NSMutableString;
NSArray, NSMutableArray;
NSDictionary, NSMutableDictionary;
NSData, NSMutableData;
NSSet, NSMutableSet;
等等...


```

##### 2> 自定义的类实现`copy`功能

```
@interface Person : NSObject <NSCopying>
@property (nonatomic, assign) int age;
@property (nonatomic, copy) NSString \*name;
@end

@implementation Person

- (nonnull id)copyWithZone:(nullable NSZone \*)zone {
    Person \*person = \[\[Person allocWithZone:zone\] init\];
    person.age = self.age;
    person.name = self.name;
    return person;
}
- (NSString \*)description {
    return \[NSString stringWithFormat:@"age = %d, weight = %@", self.age, self.name\];
}
@end


int main(int argc, const char \* argv\[\]) {
    @autoreleasepool {
        Person \*p1 = \[\[Person alloc\] init\];
        p1.age = 20;
        p1.name = @"Jack";

        
        Person \*p2 = \[p1 copy\];
        p2.age = 30;

        NSLog(@"%@", p1);
        NSLog(@"%@", p2);

        \[p2 release\];
        \[p1 release\];
    }
    return 0;
}


Demo\[1234:567890\] age = 20, weight = Jack
Demo\[1234:567890\] age = 30, weight = Jack


```

**自定义的类实现 `copy` 功能需要：**

> *   **遵守 `NSCopying` 协议；**
> *   **实现 `copyWithZone:` 方法。**

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
*   Swift1> Swift 和 OC 的区别 1.1> Swift 没有地址 / 指针的概念 1.2> 泛型 1.3> 类型严谨 对...
    
*   CADisplayLink、NSTimer 使用注意点: 1.CADisplayLink、NSTimer 会对 targ...
    
    [![](https://upload-images.jianshu.io/upload_images/2163717-afee26101f38992e.png?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)](https://www.jianshu.com/p/2f578b91083c)
*   昨天我突然想学习色彩性格分析，就开始背课件。昨天背了一种性格的一页灯片。今天和同事聊起，同事就让我给她讲讲我背的那...