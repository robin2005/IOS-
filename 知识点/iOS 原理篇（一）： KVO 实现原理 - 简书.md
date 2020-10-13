\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[www.jianshu.com\](https://www.jianshu.com/p/65b4ea295da6)

[![](https://upload.jianshu.io/users/upload_avatars/1883010/2518b53f-52b0-4e43-895c-f593dc123d07.JPG?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96/format/webp)](https://www.jianshu.com/u/75b9020bd6db)

0.0982019.04.26 10:32:03 字数 1,068 阅读 114

*   **什么是 KVO**
*   **KVO 基本使用**
*   **KVO 的本质**
*   **总结**

一 、 什么是 KVO
-----------

`KVO（Key-Value Observing）`键值监听，是`OC`对观察者模式的实现。当被观察对象的某个属性发生变化时，观察者对象会获得通知。

二 、 KVO 基本使用
------------

使用`KVO`分三个步骤：

*   注册观察者，实施监听
    *   通过`adObserver:forKeyPath:context:`方法注册观察者，观察者可以接受`keyPath`属性的变化事件
*   观察者实现回调方法，处理属性发生的变化
    *   在观察者中实现`observeValueForKeyPath:ofObject:change:context:`方法
*   移除观察者
    *   当观察者不需要监听时，调用`removeObserver:forKeyPath:`方法移除`KVO`；调用`removeObserver`需要在观察者消失之前，否则会导致`crash`

```
@interface DJTPerson : NSObject
@property (nonatomic, assign)int age;
@end

@implementation ViewController
- (void)viewDidLoad {
    \[super viewDidLoad\];
    DJTPerson \*person1 = \[\[DJTPerson alloc\] init\];
    DJTPerson \*person2 = \[\[DJTPerson alloc\] init\];
    person1.age = 1;
    person1.age = 2;
    
    \[person1 addObserver:self forKeyPath:@"age" options:NSKeyValueObservingOptionNew |NSKeyValueObservingOptionOld context:nil\];
    person1.age = 10;
    person1.age = 2;
    
    \[person1 removeObserver:self forKeyPath:@"age"\];
}

- (void)observeValueForKeyPath:(NSString \*)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> \*)change context:(void \*)context
{
    NSLog(@"%@对象的%@属性改变了，change字典为：%@",object,keyPath,change);
    NSLog(@"属性新值为：%@",change\[NSKeyValueChangeNewKey\]);
    NSLog(@"属性旧值为：%@",change\[NSKeyValueChangeOldKey\]);
}
@end


```

打印结果：

```
<DJTPerson: 0x600001e6eda0\\>对象的\`age\`属性改变了，\`change\`字典为：{
    kind = 1;
    new = 10;
    old = 1;
}
属性新值为：10
属性旧值为：1


```

三、 KVO 的本质
----------

![](http://upload-images.jianshu.io/upload_images/1883010-c7896c5f9674ed2d.png)

Key-Value Observing Programming Guide

`KVO`的实现依赖于`Objective-C`强大的`Runtime`，从 Apple 官方文档解释来看，被观察对象的`isa`指针会指向一个中间类，而不是原来真正的类，Apple 并不希望暴露更多`KVO`实现细节。  
在上面代码中，`person1`添加了键值监听，`person2`没有，而给两个实例对象的`age`赋值实质是调用了`set`方法，我们在`DJTPerson`类中重写`set`方法

**存在疑问：**  
既然改变`person`的`age`值其实是调用`setAge:`方法，那`person1`和`person2`都调用同一个方法，按理说执行的都是相同代码，为什么`person1`改变`age`会跑到`observeValueForKeyPath:`方法里呢？它是怎么通知的呢？

```
@interface DJTPerson : NSObject
@property (assign, nonatomic) int age;
@end

@implementation DJTPerson
- (void)setAge:(int)age
{
    \_age = age;
}
@end


```

**本质分析：**既然方法调用是一样的，说明问题是出现在`person1`对象本身；在`touchesBegan:`方法中，设置`person1`和`person2`的`age`值，断点调试并查看`person1` 和 `person2` 的 `isa` 指针内容，`person1`和`person2`都是实例对象，它们的`isa`指向对应的类对象，打印结果显示`person1`的类对象发生了改变，变成`NSKVONotifying_DJTPerson`，而不是`DJTPerson`。

![](http://upload-images.jianshu.io/upload_images/1883010-0b3f80921e0de4df.jpg)

对比 isa

`NSKVONotifying_DJTPerson`是如何来的呢？  
其实它是在`person1`添加`KVO`后由`Runtime`动态创建的一个类，并且是`DJTPerson`的子类：

**下面给出使用 KVO 和未使用 KVO 监听的对象变化：**

*   **_未使用 KVO 监听的对象：_**  
    
    ![](http://upload-images.jianshu.io/upload_images/1883010-f94c74c1e4a346f0.jpg)
    
    未使用 KVO
    
*   **_使用了 KVO 监听的对象：_**
    

![](http://upload-images.jianshu.io/upload_images/1883010-dc9b61bfad77350c.jpg)

使用了 KVO

从上图看出：`person1`改变`age`的值，先通过`isa`找到类对象`NSKVONotifying_DJTPerson`，调用这个类对象的`setAge:`方法`NSKVONotifying_DJTPerson的setAge:`方法中会调用 `Foudation`的`_NSSetIntValueAndNotify()` , 在这个函数中会做下面三件事情：

```
1 \[self willChangeValueForKey:@"age"\];
2 \[super setAge:age\];
3 \[self didChangeValueForKey:@"age"\];


```

而在`didChangeValueForKey:`函数中，会通知监听器，某某属性发生了改变：

```
\[observer observeValueForKeyPath:ofObject:change:context\];


```

如果自己实现这个子类，伪代码如下：

```
@interface NSKVONotifying\_DJTPerson : DJTPerson
@end

@implementation NSKVONotifying\_DJTPerson

- (void)setAge:(int)age
{
    \_NSSetIntValueAndNotify();
}


void \_NSSetIntValueAndNotify()
{
    \[self willChangeValueForKey:@"age"\];
    
    \[super setAge:age\];
    \[self didChangeValueForKey:@"age"\];
}

- (void)didChangeValueForKey:(NSString \*)key
{
    
    \[oberser observeValueForKeyPath:key ofObject:self change:nil context:nil\];
}
@end


```

`person2`对象改变`age`的值，由于它没有使用`KVO`，它的`isa`指针指向`DJTPerson`类对象，所以直接调用`DJTPerson`中的`setAge:`方法。

**_本质验证_**

```
\- (void)viewDidLoad {
    \[super viewDidLoad\];
    
    self.person1 = \[\[DJTPerson alloc\] init\];
    self.person1.age = 1;

    self.person2 = \[\[DJTPerson alloc\] init\];
    self.person2.age = 2;
    
    NSLog(@"person1添加KVO监听之前 - %@ %@",
          object\_getClass(self.person1),
          object\_getClass(self.person2)
          );
    
    NSLog(@"person1添加KVO监听之前 - %p %p",
          \[self.person1 methodForSelector:@selector(setAge:)\],
          \[self.person2 methodForSelector:@selector(setAge:)\]
          );
    
    
    NSKeyValueObservingOptions options = NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld;
    \[self.person1 addObserver:self forKeyPath:@"age" options:options context:@"123"\];
    
    NSLog(@"person1添加KVO监听之后 - %@ %@",
          object\_getClass(self.person1),
          object\_getClass(self.person2)
          );
    NSLog(@"person1添加KVO监听之后 - %p %p",
          \[self.person1 methodForSelector:@selector(setAge:)\],
          \[self.person2 methodForSelector:@selector(setAge:)\]
          );
    
    
    NSLog(@"类对象 - %@ %@",
          object\_getClass(self.person1),
          object\_getClass(self.person2)
          );
    NSLog(@"元类对象 - %@ %@",
          object\_getClass(object\_getClass(self.person1)),
          object\_getClass(object\_getClass(self.person2))
          );
}


```

打印结果为：

![](http://upload-images.jianshu.io/upload_images/1883010-ac64665842065f10.jpg)

断点调试并通过`p`命令在控制台打印方法地址内容，可以看到在添加`KVO`监听之前是调用的`[DJTPerson setAge:]`方法，而添加监听之后，调用的是`Foundation`框架的`_NSSetIntValueAndNotify`函数；

再打印一下`person1`和`person2`的类方法名：

```
\- (void)printMethodNamesOfClass:(Class)cls
{
    unsigned int count;
    
    Method \*methodList = class\_copyMethodList(cls, &count);
    
    
    NSMutableString \*methodNames = \[NSMutableString string\];
    
    
    for (int i = 0; i < count; i++) {
        
        Method method = methodList\[i\];
        
        NSString \*methodName = NSStringFromSelector(method\_getName(method));
        
        \[methodNames appendString:methodName\];
        \[methodNames appendString:@", "\];
    }
    
    free(methodList);
    
    NSLog(@"%@ %@", cls, methodNames);
}


```

```
\[self printMethodNamesOfClass:object\_getClass(self.person1)\];
\[self printMethodNamesOfClass:object\_getClass(self.person2)\];


```

打印结果为：

![](http://upload-images.jianshu.io/upload_images/1883010-ccf1c51085352b5b.jpg)

发现 `NSKVONotifying_DJTPerson` 还重写了`class`方法，我们打印

```
NSLog(@"person1添加KVO监听之后 - %@ %@",
          \[self.person1 class\],
          \[self.person2 class\]
          );


```

打印结果为：

![](http://upload-images.jianshu.io/upload_images/1883010-7c3d4f5bf9a75d19.jpg)

发现都是`DJTPerson`，这是 Apple 做的一层包装，不想把底层实现暴露给开发者，而调用`runtime`中`object_getClass(self.person1)`看到的才是它真正所属的类。

### 四、总结：

**_`KVO`的本质_**

*   利用`Runtime API`动态生成一个子类，并且让`instance`对象的`isa`指向这个全新的子类
*   当修改`instance`对象的属性时，会调用`Foundation`的`_NSSetXXXValueAndNotify`函数 (本文中`XXX=Int`)
*   `willChangeValueForKey:`
*   父类原来的`setter`
*   `didChangeValueForKey:`
*   内部会触发监听器 (`Observer`) 的监听方法 (`observeValueForKeyPath:ofObject:change:context`)

**_直接修改成员变量会触发`KVO`吗？_**

*   不会触发`KVO`，因为没有调用`set`方法，或者说没有调用`willChangeValueForKey:`和`didChangeValueForKey:`

**_如何手动触发`KVO`_**

*   手动调用`willChangeValueForKey:`和`didChangeValueForKey:`

"小礼物走一走，来简书关注我"

还没有人赞赏，支持一下

[![](https://upload.jianshu.io/users/upload_avatars/1883010/2518b53f-52b0-4e43-895c-f593dc123d07.JPG?imageMogr2/auto-orient/strip|imageView2/1/w/100/h/100/format/webp)](https://www.jianshu.com/u/75b9020bd6db)

总资产 2 (约 0.20 元) 共写了 1.6W 字获得 17 个赞共 11 个粉丝

### 被以下专题收入，发现更多相似内容