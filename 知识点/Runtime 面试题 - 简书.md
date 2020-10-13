\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[www.jianshu.com\](https://www.jianshu.com/p/6767c99034a1)

> ##### 一、objc 对象的 isa 的指针指向什么？有什么作用？

指向他的类对象, 从而可以找到对象上的方法  
详解：下图很好的描述了对象，类，元类之间的关系:

![](http://upload-images.jianshu.io/upload_images/8918169-3ec8cf9631a3078b.png)

图中实线是 super\_class 指针，虚线是 isa 指针

*   Rootclass(class) 其实就是 NSObject，NSObject 是没有超类的，所以 Rootclass(class) 的 superclass 指向 nil。
*   每个 Class 都有一个 isa 指针指向唯一的 Metaclass
*   Rootclass(meta) 的 superclass 指向 Rootclass(class)，也就是 NSObject，形成一个回路。
*   每个 Metaclass 的 isa 指针都指向 Rootclass(meta)。

> ##### 二、一个 NSObject 对象占用多少内存空间?

受限于内存分配的机制，一个 NSObject 对象都会分配 16byte 的内存空间。  
但是实际上在 64 位 下，只使用了 8byte;  
在 32 位下，只使用了 4byte  
一个 NSObject 实例对象成员变量所占的大小，实际上是 8 字节

```
#import <Objc/Runtime>
Class\_getInstanceSize(\[NSObject Class\])


```

本质是

```
size\_t class\_getInstanceSize(Class cls){
    if (!cls) return 0;
    return cls->alignedInstanceSize();
}


```

获取 Obj-C 指针所指向的内存的大小，实际上是 16 字节

```
#import<malloc/malloc.h>
malloc\_size((\_bridge const void \*)obj);


```

对象在分配内存空间时，会进行内存对齐，所以在 iOS 中，分配内存空间都是 16 字节 的倍数。  
可以通过以下网址 ：openSource.apple.com/tarballs 来查看源代码。

> ##### 三、说一下对 class\_rw\_t 的理解？

rw 代表可读可写  
objc 类中的属性、方法和遵循的协议等信息都保存在 class\_rw\_t

```
struct class\_ rw\_t {

  uint32\_ t flags;
  uint32\_ t version;
  const class\_ ro t \*ro; 
  
  method\_array\_ t methods; 
  property\_array\_t properties; 
  protocol)\_array\_t protocols; 

  Class firstSubclass;
  Class nextSiblingClass;
  
}


```

> ##### 四、说一下对 class\_ro\_t 的理解？

存储了当前类在编译期就已经确定的属性、方法以及遵循的协议。

```
struct class\_ro\_t {
  uint32\_t flags;
  uint32\_t instancestart ;
  uint32\_t instanceSize;
  uint32\_t reserved;

  const uint8\_t \* ivarLayout ;

  const char \* name ;
  method\_list\_t \* baseMethodlist;
  protocol\_list\_t \* baseProtocols;
  const ivar\_list\_t \* ivars; 

  const uint8\_t \* weakIvarLayout;
  property\_list\_t \* baseProperties;
};



```

> ##### 五、说一下对 isa 指针的理解

**说一下对 isa 指针的理解， 对象的 isa 指针指向哪里？isa 指针有哪两种类型？**  
isa 等价于 is kind of

*   实例对象 isa 指向类对象
*   类对象 isa 指向元类对象
*   元类对象的 isa 指向元类的基类

isa 有两种类型

*   纯指针，指向内存地址
*   NON\_POINTER\_ISA，除了内存地址，还存有一些其他信息

isa 源码分析

```
union isa\_t{
    Class cls;
    uintptr\_t bits; 
    #if \_\_arm64\_\_ 1 
    #define ISA\_MASK\*0000ffff8L 
    #define ISA\_MAGIC\_MASK 800003000000001ULL
    #define ISA\_MAGIC\_VALUE 0x000001000000001ULL
    struct {
        uintptr\_t nonpointer : 1; 
        uintptr\_t has\_assoc : 1; 
        uintptr\_t has\_cxx\_dtor :1; 
        uintptr\_t shiftcls  : 33; 
        uintptr\_t magic : 6; 
        uintptr\_t weakly\_referenced : 1; 
        uintptr\_t deallocating :1; 
        uintptr\_t has\_sidetable\_rc : 1; 
        uintptr\_t extra\_rc : 19; 
    #define RC\_ONE(1ULL<<45)
    #define RC\_HALF(1ULL<<18)
    };
    #elif \_\_x86\_64\_\_ 
    #define ISA\_MASK 0x00007ffffffffff8ULL
    #define ISA\_MAGIC\_MASK 0x001 F800000000001ULL
    #define ISA\_MAGIC\_VALUE 6x00180000000001ULL
    struct {
        uintptr\_t nonpointer : 1;
        uintptr\_t has\_assoc : 1;
        uintptr\_t has\_ CXX dtor : 1;
        uintptr\_t shiftcls : 44; 
        uintptr\_ t magic : 6;
        uintptr t weakly referenced : 1;
        uintptr. t deallocating :1;
        uintptr\_t has\_sidetable. rc : 1;
        uintptr t extra\_rc : 8;
        #define RC\_ONE(1ULL<<56)
        #define RC\_HALF(1ULL<<7)
    };
    #else
    #error unknown architecture for packed isa
    #endif


```

> ##### 六、说一下 Runtime 的方法缓存？存储的形式、数据结构以及查找的过程？

`cache_t`增量扩展的哈希表结构，哈希表内部存储的 `bucket_t`  
`bucket_t` 中存储的是 SEL 和 IMP 的键值对

*   如果是有序方法列表，采用二分查找
*   如果是无序方法列表，直接遍历查找

**cache\_t 结构体**

```
struct cache t {
  struct bucket\_t \* buckets; 
  mask\_t\_mask; 
  mask\_t\_occupied; 
}


```

```
struct bucket\_t {
  cache\_key\_t \_key; 
  IMP \_imp; 
  
}


```

散列表查找过程，在 objc-cache.mm 文件中

```
bucket\_t \* cache\_t: :find(cache\_key\_t K, 1d receiver)
{
  assert(k != 0); 
  bucket\_t \*b = buckets(); 
  mask\_t m = mask();
  mask\_t begin = cache\_hash(k, m);
  mask\_t i= begin; 
  do {
    if (b\[i\].key() == 0 || b\[i\].key() == k) {
    return &b\[i\];
  } while ((1 = cache\_next(i, m)) != begin);
    
    
  Class cls = (Class)((uintptr\_t)this - offsetof(objc\_class, cache));
  cache\_t::bad\_cache(receiver, (SEL)k, cls);
}


```

上面是查询散列表函数，其中 cache\_hash(k, m) 是静态内联方法，将传入的 key 和 mask 进行 & 操作返回 uint32\_t 索引值。do-while 循环查找过程，当发生冲突 cache\_next 方法将索引值减 1。

> ##### 七、使用 runtime Associate 方法关联的对象，需要在主对象 dealloc 的时候释放么？

无论在 MRC 下还是 ARC 下均不需要，被关联的对象在生命周期内要比对象本身释放的晚很多，它们会在 NSObject -dealloc 调用的 object\_dispose() 方法中释放。  
详解：

```
\>1. 调用-release: 引用计数变为零
对象正在被销毁，生命周期即将结束，
不能再有新的 weak 弱引用，否则将指向nil.
调用 \[self dealloc\]
>2. 父类调用 -dealloc 
继承关系中最直接继承的父类再调用 -dealloc 
如果是 MRC 代码则会手动释放实例变量们( iVars )
继承关系中每层的父类都再调用 -dealloc
>3. NSObject 调 -dealloc
只做一件事:调用Objective-C runtime 中object dispose() 方法
>4. 调用 object\_dispose()
为 C++ 的实例变量们( iVars )调用 destructors
为 ARC 状态下的实例变量们( iVars )调用 -release 
解除所有使用 runtime Associate方法关联的对象
解除所有 \_\_weak 引用
调用 free() 



```

> ##### 九、什么是 method swizzling（俗称黑魔法)

具体可以参看 Runtime 源代码，在文件 objc-private.h 的第 127-232 行。

```
struct objc\_object {
  isa\_t isa;
  
}


```

本质上 objc\_object 的私有属性只有一个 isa 指针，指向 类对象 的的内存地址。

> ##### 十、什么时候会报 unrecognized selector 的异常？

简单说就是进行方法交换  
在 Objective-C 中调用一个方法，其实是向一个对象发送消息，查找消息的唯一依据是 selector 的名字。利 用 Objective-C 的动态特性，可以实现在运行时偷换 selector 对应的方法实现，达到给方法挂钩的目的。 每个类都有一个方法列表，存放着方法的名字和方法实现的映射关系， selector 的本质其实就是方法名， IMP 有点类似函数指针，指向具体的 Method 实现，通过 selector 就可以找到对应的 IMP。  
换方法的几种实现方式：

*   利用 method\_exchangeImplementations 交换两个方法的实现
*   利用 class\_replaceMethod 替换方法的实现
*   利用 method\_setImplementation 来直接设置某个方法的 IMP
    
    ![](http://upload-images.jianshu.io/upload_images/8918169-18f4805277b78524.png)
    
    method swizzling
    

> ##### 十、什么时候会报 unrecognized selector 的异常？

objc 在向一个对象发送消息时，runtime 库会根据对象的 isa 指针找到该对象实际所属的类，然后在该类中的方法列表以及其父类方法列表中寻找方法运行，如果，在最顶层的父类中依然找不到相应的方法时，会进入消息转发阶段，如果消息三次转发流程仍未实现，则程序在运行时会挂掉并抛出异常 `unrecognized selector sent to XXX` 。

> ##### 十一、如何给 Category 添加属性？关联对象以什么形式进行存储？

查看的是 关联对象 的知识点。  
详细的说一下 关联对象。  
关联对象 以哈希表的格式，存储在一个全局的单例中。

```
@interface NSObject (Extension)
@property (nonatomic,copy) NSString \*name;
@end

@implementation NSObject (Extension)
- (void)setName:(NSString \*)name {
    objc\_ sessscedtbject(selfe, @seletor(name), name, OBJC\_ASSOCIATION\_COPY\_NONTOTC)
}
- (NSString \*)name {
    return objc\_getAssociatedObject((self, @selector(name)); 
}
@end



```

> ##### 十二、能否向编译后得到的类中增加实例变量？能否向运行时创建的类中添加实例变量？ 为什么？

不能向编译后得到的类中增加实例变量；  
能向运行时创建的类中添加实例变量；

*   因为编译后的类已经注册在 runtime 中, 类结构体中的 objc\_ivar\_list 实例变量的链表和 instance\_size 实 例变量的内存大小已经确定，同时 runtime 会调用 class\_setvarlayout 或 class\_setWeaklvarLayout 来处理 strong weak 引用. 所以不能向存在的类中添加实例变量。
*   运行时创建的类是可以添加实例变量，调用 class\_addIvar 函数. 但是的在调用 objc\_allocateClassPair 之 后，objc\_registerClassPair 之前, 原因同上.

> ##### 十三、类对象的数据结构？

具体可以参看 Runtime 源代码。  
类对象就是 objc\_class。

```
struct objc\_ class : objc\_ object {
  
  Class superclass;  
  cache\_t cache; 
  class\_data\_bit\_t bits; 
  class\_rw\_t \*data() {
    return bits.data(); 
  }
  
}


```

它的结构相对丰富一些。继承自 objc\_object 结构体，所以包含 isa 指针

*   isa：指向元类
*   superClass: 指向父类
*   Cache: 方法的缓存列表
*   data: 顾名思义，就是数据。是一个被封装好的 class\_rw\_t 。

> ##### 十四、runtime 如何通过 selector 找到对应的 IMP 地址？

每一个类对象中都一个方法列表, 方法列表中记录着方法的名称, 方法实现, 以及参数类型, 其实 selector 本质就是方法名称, 通过这个方法名称就可以在方法列表中找到对应的方法实现.

> ##### 十五、runtime 如何实现 weak 变量的自动置 nil？知道 SideTable 吗？

runtime 对注册的类会进行布局，对于 weak 修饰的对象会放入一个 hash 表中。 用 weak 指向的对象内存地址作为 key，当此对象的引用计数为 0 的时候会 dealloc，假如 weak 指向的对象内存地址是 a，那么就会以 a 为键， 在这个 weak 表中搜索，找到所有以 a 为键的 weak 对象，从而设置为 nil。

> ##### 十六、objc 中向一个 nil 对象发送消息将会发生什么？

如果向一个 nil 对象发送消息，首先在寻找对象的 isa 指针时就是 0 地址返回了，所以不会出现任何错误。也不会崩溃。  
**详解：**

*   如果一个方法返回值是一个对象，那么发送给 nil 的消息将返回 0(nil)；
*   如果方法返回值为指针类型，其指针大小为小于或者等于 sizeof(void\*) ，float，double，longdouble 或者 longlong 的整型标量，发送给 nil 的消息将返回 0；
*   如果方法返回值为结构体, 发送给 nil 的消息将返回 0。结构体中各个字段的值将都是 0；
*   如果方法的返回值不是上述提到的几种情况，那么发送给 nil 的消息的返回值将是未定义的。

> ##### 十七、objc 在向一个对象发送消息时，发生了什么？

objc 在向一个对象发送消息时，runtime 会根据对象的 isa 指针找到该对象实际所属的类，然后在该类中的方法列表以及其父类方法列表中寻找方法运行，如果一直到根类还没找到，转向拦截调用，走消息转发机制，一旦找到 ，就去执行它的实现 IMP 。

> ##### 十八、isKindOfClass 与 isMemberOfClass

下面的代码会输出什么？

```
@interface Sark ; Nsobject
@end
@imp1 ementation Sark
@end
int main(int argcj const char \* argv\[\]) t
    @autoreleasepool (
        BOOL res1 = \[(id)\[NSObject class\] isKindOfClass: \[nsobject class\]\];
        BOOL res2 = \[(id)\[Nsobject class\] i sMemberOfClass: \[nsobject class\]\];
        BOOL res3 = \[(id)\[Sark class\] isKindOfClass:\[Sark class\]\];
        BOOL res4 = \[(id)\[Sark class\] isMemberOfClass:\[Sark class\]\];
        NSLog(@"%d %d %d %d", res1, res2, res3, res4);
    }
    return 0;
}


```

答案：1000  
详解：

*   在 isKindOfClass 中有一个循环，先判断 class 是否等于 metaclass，不等就继续循环判断是否等于 metaclass 的 superclass，不等再继续取 superclass，如此循环下去。
*   \[NSObjectclass\] 执行完之后调用 isKindOfClass，第一次判断先判断 NSObject 和 NSObject 的 metaclass 是否 相等，之前讲到 metaclass 的时候放了一张很详细的图，从图上我们也可以看出，NSObject 的 metaclass 与本身不等。接着第二次循环判断 NSObject 与 metaclass 的 superclass 是否相等。还是从那张图上面我们 可以看到：Rootclass(meta) 的 superclass 就是 Root class(class)，也就是 NSObject 本身。所以第二次循环相等，于是第一行 res1 输出应该为 YES。
*   同理，\[Sarkclass\]执行完之后调用 isKindOfClass，第一次 for 循环，Sark 的 MetaClass 与 \[Sarkclass\] 不等，第 二次 for 循环，SarkMetaClass 的 superclass 指向的是 NSObjectMetaClass， 和 SarkClass 不相等。第三次 for 循环，NSObjectMetaClass 的 superclass 指向的是 NSObjectClass，和 SarkClass 不相等。第四次循环， NSObjectClass 的 superclass 指向 nil， 和 SarkClass 不相等。第四次循环之后，退出循环，所以第三行的 res3 输出为 NO。  
    isMemberOfClass 的源码实现是拿到自己的 isa 指针和自己比较，是否相等。
*   第二行 isa 指向 NSObject 的 MetaClass，所以和 NSObjectClass 不相等。 第四行， isa 指向 Sark 的 MetaClass， 和 SarkClass 也不等，所以第二行 res2 和第四行 res4 都输出 NO。

> ##### 十九、Category 在编译过后，是在什么时机与原有的类合并到一起的？

1.  程序启动后，通过编译之后，Runtime 会进行初始化，调用 \_objc\_init。
2.  然后会 map\_images。
3.  接下来调用 map\_images\_nolock。
4.  再然后就是 read\_images，这个方法会读取所有的类的相关信息。
5.  最后是调用 reMethodizeClass:，这个方法是重新方法化的意思。
6.  在 reMethodizeClass: 方法内部会调用 attachCategories: ，这个方法会传入 Class 和 Category ，会将方法列表，协议列表等与原有的类合并。最后加入到 class\_rw\_t 结构体中。

> ##### 二十、Category 有哪些用途？

*   给系统类添加方法、属性（需要关联对象）。
*   对某个类大量的方法，可以实现按照不同的名称归类。

> ##### 二十一、Category 的实现原理？

被添加在了 class\_rw\_t 的对应结构里。  
Category 实际上是 Category\_t 的结构体，在运行时，新添加的方法，都被以倒序插入到原有方法列 表的最前面，所以不同的 Category，添加了同一个方法，执行的实际上是最后一个。  
拿方法列表举例，实际上是一个二维的数组。  
Category 如果翻看源码的话就会知道实际上是一个 \_catrgory\_t 的结构体。  
例如我们在程序中写了一个 Nsobject+Tools 的分类，那么被编译为 C++ 之后，实际上是：

```
static struct \_catrgory\_t OBJC\_$\_CATEGORY\_NSObject\_$\_Tools \_\_attribute\_\_ ((used,section),("\_\_DATA, \_\_objc\_\_const"))
{
  
  
  
  
  
  
}


```

Category 在刚刚编译完的时候，和原来的类是分开的，只有在程序运行起来后，通过 Runtime ， Category 和原来的类才会合并到一起。  
mememove，memcpy：这俩方法是位移、复制，简单理解就是原有的方法移动到最后，根根新开辟的控件， 把前面的位置留给分类，然后分类中的方法，按照倒序依次插入，可以得出的结论就就是，越晚参与编译的分类，里面的方法才是生效的那个。

> ##### 二十二、\_objc\_msgForward 函数是做什么的，直接调用它将会发生什么？

\_objc\_msgForward 是 IMP 类型，用于消息转发的：当向一个对象发送一条消息，但它并没有实现的时候， \_objc\_msgForward 会尝试做消息转发。

*   详解：\_objc\_msgForward 在进行消息转发的过程中会涉及以下这几个方法：
    1.  ListitemresolveInstanceMethod: 方法 (或 resolveClassMethod:)。
    2.  ListitemforwardingTargetForSelector: 方法
    3.  ListitemmethodSignatureForSelector: 方法
    4.  ListitemforwardInvocation: 方法
    5.  ListitemdoesNotRecognizeSelector: 方法

> ##### 二十三、\[self class\] 与 \[super class\]

下面的代码输出什么？

```
@implementation Son : Father
- (id)init {
  self = \[super init\];
  if (self) { 
    NSLog(@"%@", NSStringFromClass(\[self class\]));
    NSLog(@"%@", NSStringFromClass(\[super class\]));
  }
  return self;
}
@end


```

`NSStringFromClass([selfclass])=Son`  
`NSStringFromClass([superclass])=Son`  
详解：这个题目主要是考察关于 Objective-C 中对 self 和 super 的理解。

*   super 本质是一个编译器标示符，而 self 是指向的同一个消息接受者。不同点在于：super 会告诉编译器， 当调用方法时，去调用父类的方法，而不是本类中的方法。
*   当使用 self 调用方法时，会从当前类的方法列表中开始找，如果没有，就从父类中再找；而当使用 super 时，则从父类的方法列表中开始找。然后调用父类的这个方法。
*   在调用 \[superclass\] 的时候，runtime 会去调用 objc\_msgSendSuper 方法，而不是 objc\_msgSend；

```
OBJC\_EXPORT void objc\_msgSendSuper(void  )

struct objc\_super {
  
  \_\_unsafe\_ \_unretained id receiver;
  
#if !defined(\_\_cplusplus) && !\_\_OBJC2\_\_
  
  \_\_unsafe\_unretained Class class;
#else
  \_\_unsafe\_unretained Class super\_class;
#endif
  
};


```

*   在 objc\_msgSendSuper 方法中，第一个参数是一个 objc\_super 的结构体，这个结构体里面有两个变量，一 个是接收消息的 receiver，一个是当前类的父类 super\_class。
*   objc\_msgSendSuper 的工作原理应该是这样的:  
    从 objc\_super 结构体指向的 superClass 父类的方法列表开始查找 selector，找到后以 objc->receiver 去调用父 类的这个 selector。注意，最后的调用者是 objc->receiver，而不是 super\_class！
*   那么 objc\_msgSendSuper 最后就转变成:

```
objc\_ msgSend(objc\_ super->receiver, @selector(class))
  
  \_\_unsafe\_unretained id receiver; 


- (Class)class {
  return object\_ getClass(self);
}


```

由于找到了父类 NSObject 里面的 class 方法的 IMP，又因为传入的入参 objc\_super->receiver = self。self 就是 son，调用 class，所以父类的方法 class 执行 IMP 之后，输出还是 son，最后输出两个都一样，都是输出 son。