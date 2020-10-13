\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[www.jianshu.com\](https://www.jianshu.com/p/47d4724510dd)

[![](https://upload.jianshu.io/users/upload_avatars/1319505/9754fac0eb59?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96/format/webp)](https://www.jianshu.com/u/40dce98b3416)

2018.08.09 18:27:24 字数 326 阅读 70

#### 面试题引发的思考：

**Q: 什么是 Runtime？平时项目中有用过么？**

*   **OC 是一门动态性比较强的编程语言，允许很多操作推迟到程序运行时再进行；**
*   **OC 的动态性就是由 Runtime 来支撑和实现的，Runtime 是一套 C 语言的 API，封装了很多动态性相关的函数；**
*   **平时编写的 OC 代码，底层都是转换成了 Runtime API 进行调用。**

**Q: 项目中具体应用？**

*   **利用关联对象（`AssociatedObject`）给分类添加属性；**
*   **遍历类的所有成员变量（修改`textfield`的占位文字颜色、字典转模型、自动归档解档）；**
*   **交换方法实现（交换系统的方法）；**
*   **利用消息转发机制解决方法找不到的异常问题；**
*   **......**

* * *

### 1\. Runtime API

#### (1) Runtime API - 类

```
1.1 动态创建一个类（参数：父类，类名，额外的内存空间）
Class objc\_allocateClassPair(Class superclass, const char \*name, size\_t extraBytes)

1.2 注册一个类（要在类注册之前添加成员变量）
void objc\_registerClassPair(Class cls) 

1.3 销毁一个类
void objc\_disposeClassPair(Class cls)

1.4 获取isa指向的Class
Class object\_getClass(id obj)

1.5 设置isa指向的Class
Class object\_setClass(id obj, Class cls)

1.6 判断一个OC对象是否为Class
BOOL object\_isClass(id obj)

1.7 判断一个Class是否为元类
BOOL class\_isMetaClass(Class cls)

1.8 获取父类
Class class\_getSuperclass(Class cls)


```

#### (2) Runtime API – 成员变量

```
2.1 获取一个实例变量信息
Ivar class\_getInstanceVariable(Class cls, const char \*name)

2.2 拷贝实例变量列表（最后需要调用free释放）
Ivar \*class\_copyIvarList(Class cls, unsigned int \*outCount)

2.3 设置和获取成员变量的值
void object\_setIvar(id obj, Ivar ivar, id value)
id object\_getIvar(id obj, Ivar ivar)

2.4 动态添加成员变量（已经注册的类是不能动态添加成员变量的）
BOOL class\_addIvar(Class cls, const char \* name, size\_t size, uint8\_t alignment, const char \* types)

2.5 获取成员变量的相关信息
const char \*ivar\_getName(Ivar v)
const char \*ivar\_getTypeEncoding(Ivar v)


```

#### (3) Runtime API – 属性

```
3.1 获取一个属性
objc\_property\_t class\_getProperty(Class cls, const char \*name)

3.2 拷贝属性列表（最后需要调用free释放）
objc\_property\_t \*class\_copyPropertyList(Class cls, unsigned int \*outCount)

3.3 动态添加属性
BOOL class\_addProperty(Class cls, const char \*name, const objc\_property\_attribute\_t \*attributes, unsigned int attributeCount)

3.4 动态替换属性
void class\_replaceProperty(Class cls, const char \*name, const objc\_property\_attribute\_t \*attributes, unsigned int attributeCount)

3.5 获取属性的一些信息
const char \*property\_getName(objc\_property\_t property)
const char \*property\_getAttributes(objc\_property\_t property)


```

#### (4) Runtime API – 方法

```
4.1 获得一个实例方法、类方法
Method class\_getInstanceMethod(Class cls, SEL name)
Method class\_getClassMethod(Class cls, SEL name)

4.2 方法实现相关操作
IMP class\_getMethodImplementation(Class cls, SEL name) 
IMP method\_setImplementation(Method m, IMP imp)
void method\_exchangeImplementations(Method m1, Method m2) 

4.3 拷贝方法列表（最后需要调用free释放）
Method \*class\_copyMethodList(Class cls, unsigned int \*outCount)

4.4 动态添加方法
BOOL class\_addMethod(Class cls, SEL name, IMP imp, const char \*types)

4.5 动态替换方法
IMP class\_replaceMethod(Class cls, SEL name, IMP imp, const char \*types)

4.6 获取方法的相关信息（带有copy的需要调用free去释放）
SEL method\_getName(Method m)
IMP method\_getImplementation(Method m)
const char \*method\_getTypeEncoding(Method m)
unsigned int method\_getNumberOfArguments(Method m)
char \*method\_copyReturnType(Method m)
char \*method\_copyArgumentType(Method m, unsigned int index)

4.7 选择器相关
const char \*sel\_getName(SEL sel)
SEL sel\_registerName(const char \*str)

4.8 用block作为方法实现
IMP imp\_implementationWithBlock(id block)
id imp\_getBlock(IMP anImp)
BOOL imp\_removeBlock(IMP anImp)


```

* * *

### 2\. Runtime 应用

#### (1) 利用关联对象（`AssociatedObject`）给分类添加属性

```
@interface Person : NSObject 
@property (nonatomic, assign) int age;
@end

@implementation Person
@end


@interface Person (Category)
@property (nonatomic, copy) NSString \*name;
@end

@implementation Person (Category)
- (void)setName:(NSString \*)name {
    objc\_setAssociatedObject(self, @selector(name), name, OBJC\_ASSOCIATION\_COPY\_NONATOMIC);
}
- (NSString \*)name {
    return objc\_getAssociatedObject(self, \_cmd);
}
@end


- (void)viewDidLoad {
    \[super viewDidLoad\];
    

    Person \*person = \[\[Person alloc\] init\];
    person.age = 10;
    person.name = @"Tom";
    NSLog(@"age - %d, weight - %@", person.age, person.name);
}


Demo\[1234:567890\] age - 10, weight - Tom


```

#### (2) 遍历类的所有成员变量

##### 修改`textfield`的占位文字颜色

```
self.textField.attributedPlaceholder = \[\[NSMutableAttributedString alloc\] initWithString:@"请输入用户名" 
attributes:@{NSForegroundColorAttributeName : \[UIColor redColor\]}\];


self.textField.placeholder = @"请输入用户名";
\[self.textField setValue:\[UIColor redColor\] forKeyPath:@"\_placeholderLabel.textColor"\];


```

##### 字典转模型

```
@interface Person : NSObject
@property (nonatomic, assign) int weight;
@property (nonatomic, assign) int age;
@property (nonatomic, copy) NSString \*name;
@end

@implementation Person
@end


@interface NSObject (Json)
+ (instancetype)my\_objectWithJson:(NSDictionary \*)dict;
@end

@implementation NSObject (Json)
+ (instancetype)my\_objectWithJson:(NSDictionary \*)dict {
    id obj = \[\[self alloc\] init\];

    unsigned int count;
    Ivar \*ivars = class\_copyIvarList(self, &count);
    for (int i=0; i<count; i++) {
        Ivar ivar = ivars\[i\];
        NSMutableString \*name = \[NSMutableString stringWithUTF8String:ivar\_getName(ivar)\];
        \[name deleteCharactersInRange:NSMakeRange(0, 1)\];
        \[obj setValue:dict\[name\] forKey:name\];
    }
    free(ivars);
    
    return obj;
}
@end


int main(int argc, const char \* argv\[\]) {
    @autoreleasepool {
        
        NSDictionary \*dict = @{@"age": @20, @"weight": @60, @"name": @"jack"};
        Person \*person = \[Person my\_objectWithJson:dict\];
        NSLog(@"age - %d weight - %d name - %@", person.age, person.weight, person.name);  
    }
    return 0;
}


Demo\[1234:567890\] age - 20 weight - 60 name - jack


```

#### (3) 交换方法实现

##### 交换系统的方法

```
@interface NSMutableArray (Extension)
@end

@implementation NSMutableArray (Extension)
+ (void)load {
    
    Class cls = NSClassFromString(@"\_\_NSArrayM");
    Method method1 = class\_getInstanceMethod(cls, @selector(insertObject:atIndex:));
    Method method2 = class\_getInstanceMethod(cls, @selector(my\_insertObject:atIndex:));
    method\_exchangeImplementations(method1, method2);
}

- (void)my\_insertObject:(id)anObject atIndex:(NSUInteger)index {
    if (anObject == nil) {
        return;
    }
    \[self my\_insertObject:anObject atIndex:index\];
}
@end


- (void)viewDidLoad {
    \[super viewDidLoad\];
    

    NSString \*obj = nil;
    NSMutableArray \*array = \[NSMutableArray array\];
    \[array addObject:@"jack"\];
    \[array insertObject:obj atIndex:0\];
    NSLog(@"array - %@", array);
}


Demo\[1234:567890\] array - ( jack )


```

#### (4) 利用消息转发机制解决方法找不到的异常问题

```
@interface Person : NSObject
- (void)run;
- (void)test;
- (void)other;
@end

@implementation Person
- (void)run {
    NSLog(@"%s", \_\_func\_\_);
}
- (NSMethodSignature \*)methodSignatureForSelector:(SEL)aSelector {
    
    if (\[self respondsToSelector:aSelector\]) {
        return \[super methodSignatureForSelector:aSelector\];
    }
    
    return \[NSMethodSignature signatureWithObjCTypes:"v@:"\];
}
- (void)forwardInvocation:(NSInvocation \*)anInvocation {
    NSLog(@"找不到%@方法", NSStringFromSelector(anInvocation.selector));
}

@end


int main(int argc, const char \* argv\[\]) {
    @autoreleasepool {
        

        Person \*person = \[\[Person alloc\] init\];
        \[person run\];
        \[person test\];
        \[person other\];
    }
    return 0;
}


Demo\[1234:567890\] -\[Person run\]
Demo\[1234:567890\] 找不到test方法
Demo\[1234:567890\] 找不到other方法


```

"小礼物走一走，来简书关注我"

还没有人赞赏，支持一下

[![](https://upload.jianshu.io/users/upload_avatars/1319505/9754fac0eb59?imageMogr2/auto-orient/strip|imageView2/1/w/100/h/100/format/webp)](https://www.jianshu.com/u/40dce98b3416)

总资产 1 (约 0.16 元) 共写了 4.8W 字获得 24 个赞共 23 个粉丝

### 被以下专题收入，发现更多相似内容

### 推荐阅读[更多精彩内容](https://www.jianshu.com/)

*   RxSwift 文档 RxSwift QQ 交流群: 424180219 RxSwift 中文文档 持续更新 提供电...
    
    [![](https://upload-images.jianshu.io/upload_images/3788243-617bee7b94fd1613.jpg?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)](https://www.jianshu.com/p/f61a5a988590)
*   part01 工作上 本周是在总部的最后一周，了解了后续的工作内容。最后一天像丽姐了解了工资结构，主要是先确定职等...
    
    [![](https://upload-images.jianshu.io/upload_images/3606760-b5d2dacd59f780c7.jpg?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)](https://www.jianshu.com/p/cc6b33c5dd2e)
*   作者 快乐的人 zzm 2018-3-17 樽杯碰错香醇酒， 姊妹笑谈亮乐眸。 肉鱼鸡鸭相间素， 欢愉共乘幸福舟。
    
    [![](https://upload-images.jianshu.io/upload_images/10672358-eff750fbbfc6fa8c.jpg?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)](https://www.jianshu.com/p/e85513f461de)