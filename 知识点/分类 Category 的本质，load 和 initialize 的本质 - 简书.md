\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[www.jianshu.com\](https://www.jianshu.com/p/3bc48e1a3678)

[![](https://upload.jianshu.io/users/upload_avatars/14929066/e2df19dd-3e29-41eb-91a3-3bc5762cc419.png?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96/format/webp)](https://www.jianshu.com/u/a53f6c36858c)

0.3682020.05.11 13:34:12 字数 4,701 阅读 119

### 问题

1.  `Category`的实现原理，以及`Category`为什么只能加方法不能加属性。
2.  `Category`和`Extension`的区别是什么？
3.  `Category`中有`load`方法吗？`load`方法是什么时候调用的？`load`方法能继承吗？
4.  `load`、`initialize`的区别，以及它们在`Category`重写的时候的调用的次序。
5.  `Category`能否添加成员变量？如果可以，如何给`Category`添加成员变量？

1\. `Category`的使用
-----------------

使用下面的这一段简单代码来分析：

```
#import <Foundation/Foundation.h>
@interface Preson : NSObject
{
    int \_age;
}
- (void)run;
@end


#import "Preson.h"
@implementation Preson
- (void)run
{
    NSLog(@"Person - run");
}
@end



#import "Preson.h"
@interface Preson (Test) <NSCopying>
- (void)test;
+ (void)abc;
@property (assign, nonatomic) int age;
- (void)setAge:(int)age;
- (int)age;
@end


#import "Preson+Test.h"
@implementation Preson (Test)
- (void)test
{
}

+ (void)abc
{
}
- (void)setAge:(int)age
{
}
- (int)age
{
    return 10;
}
@end



#import "Preson.h"
@interface Preson (Test2)
@end


#import "Preson+Test2.h"
@implementation Preson (Test2)
- (void)run
{
    NSLog(@"Person (Test2) - run");
}
@end


```

我们之前讲到过实例对象的`isa`指针指向类对象，类对象的`isa`指针指向元类对象，当`p`调用`run`方法时，通过实例对象的`isa`指针找到类对象，然后在类对象中查找对象方法，如果没有找到，就通过类对象的`super_class`指针找到父类对象，接着去寻找`run`方法。

那么当调用分类的方法时，步骤是否和调用对象方法一样呢？

分类中的对象方法依然是存储在类对象中的，同本类对象方法在同一个地方，调用步骤也同调用对象方法一样。如果是类方法的话，也同样是存储在元类对象中。  
那么分类方法是如何存储在类对象中的，我们来通过源码看一下分类的底层结构。

2\. 分类的底层结构
-----------

扩展的方法不是在编译时期合并至原来的类，而是在运行时合并的。

我们将`OC`编译的文件生成底层的`C++`实现

```
xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc Preson+Test.m


```

#### 2.1 分类结构体

在分类转化为`C++`文件中可以找到`_category_t` 结构体中，存放着类名，对象方法列表，类方法列表，协议列表，以及属性列表。

```
struct \_category\_t {
    const char \*name; 
    struct \_class\_t \*cls;
    const struct \_method\_list\_t \*instance\_methods; 
    const struct \_method\_list\_t \*class\_methods; 
    const struct \_protocol\_list\_t \*protocols; 
    const struct \_prop\_list\_t \*properties; 
};


```

#### 2.2 分类结构体的成员列表

##### 2.2.1 分类的实例方法结构体

存放实例方法`_method_list_t`类型的结构体，如下所示

```
static struct  {
    unsigned int entsize;  
    unsigned int method\_count; 
    struct \_objc\_method method\_list\[3\]; 
} \_OBJC\_$\_CATEGORY\_INSTANCE\_METHODS\_Person\_$\_Test \_\_attribute\_\_ ((used, section ("\_\_DATA,\_\_objc\_const"))) = {
    sizeof(\_objc\_method),
    3,
    {{(struct objc\_selector \*)"test", "v16@0:8", (void \*)\_I\_Person\_Test\_test},
    {(struct objc\_selector \*)"setAge:", "v20@0:8i16", (void \*)\_I\_Person\_Test\_setAge\_},
    {(struct objc\_selector \*)"age", "i16@0:8", (void \*)\_I\_Person\_Test\_age}}
};


```

上面中我们发现这个结构体 `_OBJC_$_CATEGORY_INSTANCE_METHODS_Person_$_Test` 从名称可以看出是`INSTANCE_METHODS`对象方法，并且一一对应为上面结构体内赋值。我们可以看到结构体中存储了方法占用的内存，方法数量，以及方法列表。并且从上图中找到分类中我们添加的对象方法，`test` , `setAge`, `age`三个方法。

##### 2.2.2 分类的类方法结构体

存放类方法`_method_list_t`类型的类方法结构体，如下所示

```
static struct  {
    unsigned int entsize;  
    unsigned int method\_count;
    struct \_objc\_method method\_list\[1\];
} \_OBJC\_$\_CATEGORY\_CLASS\_METHODS\_Person\_$\_Test \_\_attribute\_\_ ((used, section ("\_\_DATA,\_\_objc\_const"))) = {
    sizeof(\_objc\_method),
    1,
    {{(struct objc\_selector \*)"abc", "v16@0:8", (void \*)\_C\_Person\_Test\_abc}}
};


```

同上面实例方法列表一样，这个我们可以看出是类方法列表结构体 `_OBJC_$_CATEGORY_CLASS_METHODS_Person_$_Test`，同对象方法结构体相同，同样可以看到我们实现的类方法`abc`。

##### 2.2.3 分类的协议方法结构体

存放协议列表结构体`_protocol_list_t`，假如我们实现了`NSCopying`协议

```
static const char \*\_OBJC\_PROTOCOL\_METHOD\_TYPES\_NSCopying \[\] \_\_attribute\_\_ ((used, section ("\_\_DATA,\_\_objc\_const"))) = 
{
    "@24@0:8^{\_NSZone=}16"
};

static struct  {
    unsigned int entsize;  
    unsigned int method\_count;
    struct \_objc\_method method\_list\[1\];
} \_OBJC\_PROTOCOL\_INSTANCE\_METHODS\_NSCopying \_\_attribute\_\_ ((used, section ("\_\_DATA,\_\_objc\_const"))) = {
    sizeof(\_objc\_method),
    1,
    {{(struct objc\_selector \*)"copyWithZone:", "@24@0:8^{\_NSZone=}16", 0}}
};

struct \_protocol\_t \_OBJC\_PROTOCOL\_NSCopying \_\_attribute\_\_ ((used)) = {
    0,
    "NSCopying",
    0,
    (const struct method\_list\_t \*)&\_OBJC\_PROTOCOL\_INSTANCE\_METHODS\_NSCopying,
    0,
    0,
    0,
    0,
    sizeof(\_protocol\_t),
    0,
    (const char \*\*)&\_OBJC\_PROTOCOL\_METHOD\_TYPES\_NSCopying
};
struct \_protocol\_t \*\_OBJC\_LABEL\_PROTOCOL\_$\_NSCopying = &\_OBJC\_PROTOCOL\_NSCopying;

static struct  {
    long protocol\_count; 
    struct \_protocol\_t \*super\_protocols\[1\]; 
} \_OBJC\_CATEGORY\_PROTOCOLS\_$\_Person\_$\_Test \_\_attribute\_\_ ((used, section ("\_\_DATA,\_\_objc\_const"))) = {
    1,
    &\_OBJC\_PROTOCOL\_NSCopying
};


```

通过上述源码可以看到先将协议方法通过`_method_list_t`结构体存储

之后通过`_protocol_t`结构体存储在`_OBJC_CATEGORY_PROTOCOLS_$_Person_$_Test`中同`_protocol_list_t`结构体一一对应，分别为`protocol_count`协议数量以及存储了协议方法的`_protocol_t`结构体。

##### 2.2.4 分类的属性列表

最后我们可以看到属性列表结构体`_prop_list_t`

```
static struct  {
    unsigned int entsize;  
    unsigned int count\_of\_properties; 
    struct \_prop\_t prop\_list\[1\]; 
} \_OBJC\_$\_PROP\_LIST\_Person\_$\_Test \_\_attribute\_\_ ((used, section ("\_\_DATA,\_\_objc\_const"))) = {
    sizeof(\_prop\_t),
    1,
    {{"age","Ti,N"}} 
};



```

属性列表结构体`_OBJC_$_PROP_LIST_Person_$_Test`同`_prop_list_t`结构体对应，存储属性的占用空间，属性属性数量，以及属性列表，从上图中可以看到我们自己添加的`age`属性。

##### 2.2.5 分类`_category_t`结构体总结

最后我们可以看到定义了`_OBJC_$_CATEGORY_Person_$_Test`结构体，并且将我们上面着重分析的结构体一一赋值。

```
struct \_category\_t {
    const char \*name;
    struct \_class\_t \*cls;
    const struct \_method\_list\_t \*instance\_methods;
    const struct \_method\_list\_t \*class\_methods;
    const struct \_protocol\_list\_t \*protocols;
    const struct \_prop\_list\_t \*properties;
};



extern "C" \_\_declspec(dllimport) struct \_class\_t OBJC\_CLASS\_$\_Person;

static struct \_category\_t \_OBJC\_$\_CATEGORY\_Person\_$\_Test \_\_attribute\_\_ ((used, section ("\_\_DATA,\_\_objc\_const"))) = 
{
    "Person",
    0, 
    (const struct \_method\_list\_t \*)&\_OBJC\_$\_CATEGORY\_INSTANCE\_METHODS\_Person\_$\_Test,
    (const struct \_method\_list\_t \*)&\_OBJC\_$\_CATEGORY\_CLASS\_METHODS\_Person\_$\_Test,
    (const struct \_protocol\_list\_t \*)&\_OBJC\_CATEGORY\_PROTOCOLS\_$\_Person\_$\_Test,
    (const struct \_prop\_list\_t \*)&\_OBJC\_$\_PROP\_LIST\_Person\_$\_Test,
};
static void OBJC\_CATEGORY\_SETUP\_$\_Person\_$\_Test(void ) {
    \_OBJC\_$\_CATEGORY\_Person\_$\_Test.cls = &OBJC\_CLASS\_$\_Person;
}


```

并且我们看到定义原类`_class_t`类型的`OBJC_CLASS_$_Preson`结构体

最后将分类`_OBJC_$_CATEGORY_Person_$_Test`的`cls`指针指向原类`OBJC_CLASS_$_Preson`结构体地址。我们这里可以看出，`cls`指针指向的应该是原类的类对象的地址。

3\. 源码分析
--------

通过查看分类的源码我们可以找到底层分类`category_t`结构体。

objc 源码路径：runtime/objc-runtime-new.h

```
struct category\_t {
    const char \*name; 
    classref\_t cls;
    struct method\_list\_t \*instanceMethods; 
    struct method\_list\_t \*classMethods; 
    struct protocol\_list\_t \*protocols; 
    struct property\_list\_t \*instanceProperties; 
    
    struct property\_list\_t \*\_classProperties;

    method\_list\_t \*methodsForMeta(bool isMeta) {
        if (isMeta) return classMethods;
        else return instanceMethods;
    }

    property\_list\_t \*propertiesForMeta(bool isMeta, struct header\_info \*hi);
};


```

从源码基本可以看出我们平时使用`categroy`的方式，对象方法，类方法，协议，和属性都可以找到对应的存储方式。**并且我们发现分类结构体中是不存在成员变量的，因此分类中是不允许添加成员变量的**。分类中添加的属性并不会帮助我们自动生成成员变量，只会生成`get set`方法的声明，需要我们自己去实现。

通过源发现，分类的方法，协议，属性等好像确实是存放在`categroy`结构体里面的，那么他又是如何合并存储到原类的类对象中的？

#### 3.1 分类是如何存储方法，属性，协议的

通过我们`runtime`的初始化函数`_objc_init`来探寻答案。

objc 源码路径：runtime/objc-os.mm

入口函数：`_objc_init`:

```
void \_objc\_init(void)
{
    static bool initialized = false;
    if (initialized) return;
    initialized = true;
    
    
    environ\_init();
    tls\_init();
    static\_init();
    lock\_init();
    exception\_init();

    
    \_dyld\_objc\_notify\_register(&map\_images, load\_images, unmap\_image);
}


```

我们来到`&map_images`读取模块（`images`这里代表模块）

↓↓↓

来到`map_images_nolock`函数

↓↓↓

来到`_read_images`函数

最终在`_read_images`函数中找到分类相关代码:

objc 源码路径：runtime/objc-runtime-new.mm

```
    for (EACH\_HEADER) {
        
        category\_t \*\*catlist = \_getObjc2CategoryList(hi, &count); 
        bool hasClassProperties = hi->info()->hasCategoryClassProperties();

        
        
        for (i = 0; i < count; i++) {
            category\_t \*cat = catlist\[i\];
            Class cls = remapClass(cat->cls);

            if (!cls) {
                
                
                catlist\[i\] = nil;
                if (PrintConnecting) {
                    \_objc\_inform("CLASS: IGNORING category \\?\\?\\?(%s) %p with "
                                 "missing weak-linked target class", 
                                 cat->name, cat);
                }
                continue;
            }

            
            
            
            
            bool classExists = NO;
            if (cat->instanceMethods ||  cat->protocols  
                ||  cat->instanceProperties) 
            {
                addUnattachedCategoryForClass(cat, cls, hi);
                if (cls->isRealized()) {
                    
                    remethodizeClass(cls);
                    classExists = YES;
                }
                if (PrintConnecting) {
                    \_objc\_inform("CLASS: found category -%s(%s) %s", 
                                 cls->nameForLogging(), cat->name, 
                                 classExists ? "on existing class" : "");
                }
            }

            if (cat->classMethods  ||  cat->protocols  
                ||  (hasClassProperties && cat->\_classProperties)) 
            {
                addUnattachedCategoryForClass(cat, cls->ISA(), hi);
                if (cls->ISA()->isRealized()) {
                    remethodizeClass(cls->ISA());
                }
                if (PrintConnecting) {
                    \_objc\_inform("CLASS: found category +%s(%s)", 
                                 cls->nameForLogging(), cat->name);
                }
            }
        }
    }


```

从上述代码中我们可以知道这段代码是用来查找有没有分类的。通过`_getObjc2CategoryList`函数获取到分类列表之后，进行遍历，获取其中的方法，协议，属性等。可以看到最终都调用了`remethodizeClass(cls)`函数。我们来到`remethodizeClass(cls)`函数内部查看:

↓↓↓

`remethodizeClass(cls)`函数:

```
static void remethodizeClass(Class cls)
{
    category\_list \*cats;
    bool isMeta;

    runtimeLock.assertLocked();

    isMeta = cls->isMetaClass();

    
    if ((cats = unattachedCategoriesForClass(cls, false))) {
        if (PrintConnecting) {
            \_objc\_inform("CLASS: attaching categories to class '%s' %s", 
                         cls->nameForLogging(), isMeta ? "(meta)" : "");
        }
        
        
        attachCategories(cls, cats, true );        
        free(cats);
    }
}


```

通过上述代码我们发现`attachCategories`函数接收了类对象`cls`和分类数组`cats`，如我们一开始写的代码所示，一个类可以有多个分类。之前我们说到分类信息存储在`category_t`结构体中，那么多个分类则保存在`category_list`中。

↓↓↓

我们来到`attachCategories()`函数内部:

```
static void 
attachCategories(Class cls, category\_list \*cats, bool flush\_caches)
{
    if (!cats) return;
    if (PrintReplacedMethods) printReplacements(cls, cats);

    bool isMeta = cls->isMetaClass();

    
    
    
    
    method\_list\_t \*\*mlists = (method\_list\_t \*\*)
        malloc(cats->count \* sizeof(\*mlists));
    
    
    property\_list\_t \*\*proplists = (property\_list\_t \*\*)
        malloc(cats->count \* sizeof(\*proplists));
    
    
    protocol\_list\_t \*\*protolists = (protocol\_list\_t \*\*)
        malloc(cats->count \* sizeof(\*protolists));

    
    int mcount = 0;
    int propcount = 0;
    int protocount = 0;
    int i = cats->count;
    bool fromBundle = NO;
    while (i--) { 
        
        auto& entry = cats->list\[i\];
        
        
        method\_list\_t \*mlist = entry.cat->methodsForMeta(isMeta);
        if (mlist) {
            mlists\[mcount++\] = mlist;
            fromBundle |= entry.hi->isBundle();
        }
    
        
        property\_list\_t \*proplist = 
            entry.cat->propertiesForMeta(isMeta, entry.hi);
        if (proplist) {
            proplists\[propcount++\] = proplist;
        }

        
        protocol\_list\_t \*protolist = entry.cat->protocols;
        if (protolist) {
            protolists\[protocount++\] = protolist;
        }
    }

    
    
    auto rw = cls->data();

    prepareMethodLists(cls, mlists, mcount, NO, fromBundle);
    
    
    rw->methods.attachLists(mlists, mcount);
    free(mlists);
    if (flush\_caches  &&  mcount > 0) flushCaches(cls);
    
    
    rw->properties.attachLists(proplists, propcount);
    free(proplists);

    
    rw->protocols.attachLists(protolists, protocount);
    free(protolists);
}


```

上述源码中可以看出，首先根据方法列表，属性列表，协议列表，`malloc`分配内存，根据多少个分类以及每一块方法需要多少内存来分配相应的内存地址。

之后将每一个分类的所有的方法、属性、协议分别放入对应`mlist`、`proplists`、`protolosts`数组中，这三个数组放着所有分类的所有方法，属性和协议。

之后通过类对象的`data()`方法，拿到类对象的`class_rw_t`结构体`rw`，在`class`结构中我们介绍过，`class_rw_t`中存放着类对象的方法，属性和协议等数据，`rw`结构体通过类对象的`data`方法获取，所以`rw`里面存放这类对象里面的数据。

之后分别通过`rw`调用方法列表、属性列表、协议列表的`attachList`函数，将所有的分类的方法、属性、协议列表数组传进去。

我们大致可以猜想到在`attachList`方法内部将分类和本类相应的对象方法，属性，和协议进行了合并。

↓↓↓

我们来到`attachLists`函数内部:

objc 源码路径：runtime/objc-runtime-new.h

```
void attachLists(List\* const \* addedLists, uint32\_t addedCount) {
        if (addedCount == 0) return;

        if (hasArray()) {
            
            uint32\_t oldCount = array()->count;
            
            
            
            uint32\_t newCount = oldCount + addedCount;
            
            setArray((array\_t \*)realloc(array(), array\_t::byteSize(newCount)));
            
            array()->count = newCount;
           
            
            
            
            
            
            memmove(array()->lists + addedCount, array()->lists, 
                    oldCount \* sizeof(array()->lists\[0\]));
            
             
             
             
             
             
            memcpy(array()->lists, addedLists, 
                   addedCount \* sizeof(array()->lists\[0\]));;
        }
        else if (!list  &&  addedCount == 1) {
            
            list = addedLists\[0\];
        } 
        else {
            
            List\* oldList = list;
            uint32\_t oldCount = oldList ? 1 : 0;
            uint32\_t newCount = oldCount + addedCount;
            setArray((array\_t \*)malloc(array\_t::byteSize(newCount)));
            array()->count = newCount;
            if (oldList) array()->lists\[addedCount\] = oldList;
            memcpy(array()->lists, addedLists, 
                   addedCount \* sizeof(array()->lists\[0\]));
        }
    }


```

由上面的源码中可以看出，新添加的分类的其实是相当于移动到了原来存储的列表之前的，所以原类和分类重名的时候会优先调用分类的方法。

另外，当多个分类有重名的方法的时候由源码的`attachCategories()`函数可以知道，其实是有优先调用最后参与编译的分类的方法：

```
...

while (i--) { 
        
        auto& entry = cats->list\[i\];
        ...
}
...


```

#### 3.2 `Extension`类扩展

类扩展其实就是我们平常写的`@interface`

```
@interface AppDelegate ()

@end


@implementation AppDelegate

@end


```

和分类`categroy`不同的是：类扩展的信息是在编译的时候已经合并在了类对象中，而分类是在运行时合并至原来中的。

#### 3.3 `memmove`和`memcpy`

上面的源代码中有两个重要的数组:

1.  `array()->lists`： 原类对象原来的方法列表，属性列表，协议列表。
2.  `addedLists`：传入所有分类的方法列表，属性列表，协议列表。

`attachLists`函数中最重要的两个方法为`memmove`内存移动和`memcpy`内存拷贝。我们先来分别看一下这两个函数:

```
void    \*memmove(void \*\_\_dst, const void \*\_\_src, size\_t \_\_len);



void    \*memcpy(void \*\_\_dst, const void \*\_\_src, size\_t \_\_n);


```

下面我们图示经过`memmove`和`memcpy`方法过后的内存变化。

##### 3.3.1 内存移动`memmove`

![](http://upload-images.jianshu.io/upload_images/14929066-2a1904767c169e2e.png)

memmove\_bofore

经过`memmove`方法之后，内存变化为:

```
memmove(array()->lists + addedCount, array()->lists, 
                  oldCount \* sizeof(array()->lists\[0\]));


```

![](http://upload-images.jianshu.io/upload_images/14929066-622bdf22893b16e0.png)

memmove\_after

经过`memmove`方法之后，我们发现，虽然本类的方法，属性，协议列表会分别后移，但是本类的对应数组的指针依然指向原始位置。

##### 3.3.2 内存复制`memcpy`

```
memcpy(array()->lists, addedLists, 
               addedCount \* sizeof(array()->lists\[0\]));


```

![](http://upload-images.jianshu.io/upload_images/14929066-2fbfb4b3e6a1497f.png)

memcopy

我们发现原来指针并没有改变，至始至终指向开头的位置。并且经过`memmove`和`memcpy`方法之后，分类的方法，属性，协议列表被放在了类对象中原本存储的方法，属性，协议列表前面。

那么为什么要将分类方法的列表追加到本来的对象方法前面呢，这样做的目的是为了保证分类方法优先调用，我们知道当分类重写本类的方法时，会覆盖本类的方法。  
其实经过上面的分析我们知道本质上并不是覆盖，而是优先调用。本类的方法依然在内存中的。我们可以通过打印所有类的所有方法名来查看

```
2020-01-14 11:47:02.458927+0800 分类Category的本质\[84870:4069262\] Person (Test2) - run
2020-01-14 11:47:02.459080+0800 分类Category的本质\[84870:4069262\] Person类： age, run, run, setAge:, test, 


```

调用的是`Test2`中的`run`方法，并且`Person`类中存储着两个`run`方法。

#### 3.4 总结

##### `Category`的实现原理，以及`Category`为什么只能加方法不能加属性?

分类的实现原理是将`Category`中的方法，属性，协议数据放在`category_t`结构体中，然后将结构体内的方法列表拷贝到类对象的方法列表中。

`Category`可以添加属性，但是并不会自动生成成员变量及`set get`方法。因为`category_t`结构体中并不存在成员变量。

通过之前对对象的分析我们知道成员变量是存放在实例对象中的，并且编译的那一刻就已经决定好了。而分类是在运行时才去加载的。那么我们就无法再程序运行时将分类的成员变量中添加到实例对象的结构体中。因此分类中不可以添加成员变量。

##### `Category`和`Extension`的区别是什么？

和分类`Categroy`不同的是：类扩展的信息是在编译的时候已经合并在了类对象中，而分类是在运行时合并至原类中的。

4\. `load`和`initialize`
-----------------------

#### 4.1 `load`方法

##### 4.4.1 基本使用

`load`方法是 runtime 在加载类和分类的时候会调用，是在程序入口调用函数`main`之前调用，而且只会调用一次。

通过代码验证一下调用本类的`load`方法调用。

我们添加`Student`继承`Presen`类，并添加`Student+Test`分类，分别重写`+load`方法。

```
2020-01-14 14:07:53.689561+0800 分类Category的本质\[92689:4179307\] Person - load
2020-01-14 14:07:53.690079+0800 分类Category的本质\[92689:4179307\] Student - load
2020-01-14 14:07:53.690142+0800 分类Category的本质\[92689:4179307\] Student (Test) - load


```

通过验证我们发现不仅调用了分类的`load`方法，而且调用了原类的`load`方法，这和上面我们验证的**优先调用分类的方法**的逻辑相冲突，到底为什么会调用原类的方法，我们通过底层的源码一探究竟。

##### 4.4.2 调用原理

同样的我们从`runtime`的入口`_objc_init`函数的`load_images`函数寻找答案:

```
void \_objc\_init(void)
{
    static bool initialized = false;
    if (initialized) return;
    initialized = true;
    
    
    environ\_init();
    tls\_init();
    static\_init();
    lock\_init();
    exception\_init();

    
    \_dyld\_objc\_notify\_register(&map\_images, load\_images, unmap\_image);
}


```

↓↓↓

最终来到了`call_load_methods`函数

objc 源码路径：runtime/objc-loadmethod.mm

```
void call\_load\_methods(void)
{
    static bool loading = NO;
    bool more\_categories;

    loadMethodLock.assertLocked();

    
    if (loading) return;
    loading = YES;

    void \*pool = objc\_autoreleasePoolPush();

    do {
        
        while (loadable\_classes\_used > 0) {
            
            call\_class\_loads(); 
        }

        
        
        more\_categories = call\_category\_loads(); 

        
    } while (loadable\_classes\_used > 0  ||  more\_categories);

    objc\_autoreleasePoolPop(pool);

    loading = NO;
}


```

↓↓↓

先调用原类的`load`方法：

```
static void call\_class\_loads(void)
{
    int i;
    
    
    struct loadable\_class \*classes = loadable\_classes;
    int used = loadable\_classes\_used;
    loadable\_classes = nil;
    loadable\_classes\_allocated = 0;
    loadable\_classes\_used = 0;
    
    for (i = 0; i < used; i++) {
        Class cls = classes\[i\].cls;
        
        load\_method\_t load\_method = (load\_method\_t)classes\[i\].method;
        if (!cls) continue; 

        if (PrintLoading) {
            \_objc\_inform("LOAD: +\[%s load\]\\n", cls->nameForLogging());
        }
        
        (\*load\_method)(cls, @selector(load));
    }
    
    
    if (classes) free(classes);
}



```

↓↓↓

再调用分类的`load`方法：

```
static bool call\_category\_loads(void)
{
    int i, shift;
    bool new\_categories\_added = NO;
    
    
    struct loadable\_category \*cats = loadable\_categories;
    int used = loadable\_categories\_used;
    int allocated = loadable\_categories\_allocated;
    loadable\_categories = nil;
    loadable\_categories\_allocated = 0;
    loadable\_categories\_used = 0;

    
    for (i = 0; i < used; i++) {
        Category cat = cats\[i\].cat;
        
        
        load\_method\_t load\_method = (load\_method\_t)cats\[i\].method;
        Class cls;
        if (!cat) continue;
        cls = \_category\_getClass(cat);
        if (cls  &&  cls->isLoadable()) {
            if (PrintLoading) {
                \_objc\_inform("LOAD: +\[%s(%s) load\]\\n", 
                             cls->nameForLogging(), 
                             \_category\_getName(cat));
            }
            
            
            (\*load\_method)(cls, @selector(load));
            cats\[i\].cat = nil;
        }
    }
    
    ...
    ...
    ...
}



```

和`runtime`的消息发送机制不同的是，消息发送机制是需要通过`isa`指针去逐层寻找，而`load`方法是不需要通过消息发送的，而是直接通过函数的地址来调用的

```
struct loadable\_class {
    Class cls;  
    IMP method; 
};

struct loadable\_category {
    Category cat;  
    IMP method; 
};


```

##### 4.4.3 调用顺序

即使是再复杂继承关系，原类、分类、子类的`load`方法都会被调用，并且是按照一定的顺序调用的。

通过上面的源码可以看到在调用原类和分类的`load`方法的时候，都是分别通过一个数组`loadable_classes`、`loadable_categories`进行`for`循环去遍历的，所以知道数组的顺序，就可以知道方法的调用顺序

我们回到之前的入口函数`load_images`, 发现在调用`call_load_methods()`函数之前调用了`prepare_load_methods()`方法：

```
void
load\_images(const char \*path \_\_unused, const struct mach\_header \*mh)
{
    
    if (!hasLoadMethods((const headerType \*)mh)) return;

    recursive\_mutex\_locker\_t lock(loadMethodLock);

    
    {
        mutex\_locker\_t lock2(runtimeLock);
        
        prepare\_load\_methods((const headerType \*)mh);
    }

    
    call\_load\_methods();
}


```

↓↓↓

```
void prepare\_load\_methods(const headerType \*mhdr)
{
    size\_t count, i;

    runtimeLock.assertLocked();

    
    
    classref\_t const \*classlist = 
        \_getObjc2NonlazyClassList(mhdr, &count);
    for (i = 0; i < count; i++) {
        
        
        schedule\_class\_load(remapClass(classlist\[i\]));
    }

    
    
    category\_t \* const \*categorylist = \_getObjc2NonlazyCategoryList(mhdr, &count);
    for (i = 0; i < count; i++) {
        category\_t \*cat = categorylist\[i\];
        Class cls = remapClass(cat->cls);
        if (!cls) continue;  
        if (cls->isSwiftStable()) {
            \_objc\_fatal("Swift class extensions and categories on Swift "
                        "classes are not allowed to have +load methods");
        }
        realizeClassWithoutSwift(cls, nil);
        ASSERT(cls->ISA()->isRealized());
        add\_category\_to\_loadable\_list(cat);
    }
}



```

↓↓↓

原类的`load`方法添加到`loadable_classes`数组的顺序:

```
static void schedule\_class\_load(Class cls)
{
    if (!cls) return;
    ASSERT(cls->isRealized());  

    if (cls->data()->flags & RW\_LOADED) return;

    
    
    schedule\_class\_load(cls->superclass);

    
    add\_class\_to\_loadable\_list(cls);
    cls->setInfo(RW\_LOADED); 
}


void add\_class\_to\_loadable\_list(Class cls)
{
    IMP method;

    loadMethodLock.assertLocked();

    method = cls->getLoadMethod();
    if (!method) return;  
    
    if (PrintLoading) {
        \_objc\_inform("LOAD: class '%s' scheduled for +load", 
                     cls->nameForLogging());
    }
    
    if (loadable\_classes\_used == loadable\_classes\_allocated) {
        loadable\_classes\_allocated = loadable\_classes\_allocated\*2 + 16;
        loadable\_classes = (struct loadable\_class \*)
            realloc(loadable\_classes,
                              loadable\_classes\_allocated \*
                              sizeof(struct loadable\_class));
    }
    
    loadable\_classes\[loadable\_classes\_used\].cls = cls;
    loadable\_classes\[loadable\_classes\_used\].method = method;
    loadable\_classes\_used++;
}


```

分类的`load`方法添加到`loadable_categories`数组的顺序:

```
void add\_category\_to\_loadable\_list(Category cat)
{
    IMP method;

    loadMethodLock.assertLocked();

    method = \_category\_getLoadMethod(cat);

    
    if (!method) return;

    if (PrintLoading) {
        \_objc\_inform("LOAD: category '%s(%s)' scheduled for +load", 
                     \_category\_getClassName(cat), \_category\_getName(cat));
    }
    
    if (loadable\_categories\_used == loadable\_categories\_allocated) {
        loadable\_categories\_allocated = loadable\_categories\_allocated\*2 + 16;
        loadable\_categories = (struct loadable\_category \*)
            realloc(loadable\_categories,
                              loadable\_categories\_allocated \*
                              sizeof(struct loadable\_category));
    }

    loadable\_categories\[loadable\_categories\_used\].cat = cat;
    loadable\_categories\[loadable\_categories\_used\].method = method;
    loadable\_categories\_used++;
}



```

通过以上源码的逻辑处理，我们发现数组的添加顺序导致了`load`方法的调用顺序：先将元类添加到数组，同时会先去将父类添加到数组，再讲子类添加到数组，最后将分类添加到数组。所以在调用`load`方法的时候也会是按照类的添加顺序来调用。

##### 4.4.4 总结

1.  先调用所有原类的`laod`方法
    
    *   按照编译顺序调用 (可以手动设置编译顺序)
    *   调用子类的`load`之前会先调用父类的`load`方法
2.  再调用分类的`laod`方法
    
    *   按照编译顺序调用 (可以手动设置编译顺序)

##### `Category`中有`load`方法吗？`load`方法是什么时候调用的？`load` 方法能继承吗？

`Category`中有`load`方法，`load`方法在程序加载了类和分类的时候就会调用，在`main`函数之前调用。`load`方法可以继承。调用子类的`load`方法之前，会先调用父类的`load`方法。一般我们不会手动去调用`load`方法，而是让系统去调用。

如果非要手动调用`load`方法，那么就会按照消息发送机制通过`isa`指针来寻找方法。

#### 4.2 `initialize`方法

我们为`Preson`、`Person+Test`、`Student` 、`Student+Test` 添加`initialize`方法。

##### 4.2.1 基本使用

`initialize`类第一次接收到消息时，就会调用，相当于第一次使用类的时候就会调用`initialize`方法。

`initialize`方法的调用是通过消息发送机制调用的，不像`load`方法是直接通过函数指针去调用。

再来验证一下调用顺序：

```
2020-04-28 21:22:18.874260+0800 分类Category的本质\[90053:17829138\] Person (Test) initialize
2020-04-28 21:22:18.874726+0800 分类Category的本质\[90053:17829138\] Student (Test) initialize


```

调用子类的`initialize`之前，会先保证调用父类的`initialize`方法。如果之前已经调用过`initialize`，就不会再调用`initialize`方法了。

当分类重写`initialize`方法时会先调用分类的方法不再调用原类的方法。

`initialize`是通过消息发送机制调用的，消息发送机制通过`isa`指针找到对应的方法与实现，因此优先找到分类方法中的实现，会优先调用分类方法中的实现。

另外还有一点需要注意：如果子类没有实现`initialize`方法，那么会调用父类的`initialize`方法，这一点我们会通过源码去验证。

##### 4.2.2 源码分析

在底层源码里面通过函数`class_getInstanceMethod`和函数`class_getClassMethod`来找到实例方法和类方法

```
Method class\_getInstanceMethod(Class cls, SEL sel)
{
    if (!cls  ||  !sel) return nil;

    

    
    

    Method meth;
    meth = \_cache\_getMethod(cls, sel, \_objc\_msgForward\_impcache);
    if (meth == (Method)1) {
        
        return nil;
    } else if (meth) {
        return meth;
    }
        
    
    lookUpImpOrForward(nil, sel, cls, LOOKUP\_INITIALIZE | LOOKUP\_RESOLVER);

    meth = \_cache\_getMethod(cls, sel, \_objc\_msgForward\_impcache);
    if (meth == (Method)1) {
        
        return nil;
    } else if (meth) {
        return meth;
    }

    return \_class\_getMethod(cls, sel);
}


```

↓↓↓

```
IMP lookUpImpOrForward(id inst, SEL sel, Class cls, int behavior)
{
    Class curClass;
    IMP methodPC = nil;
    Method meth;
    bool triedResolver = NO;

    methodListLock.assertUnlocked();

    
    if (behavior & LOOKUP\_CACHE) {
        methodPC = \_cache\_getImp(cls, sel);
        if (methodPC) goto out\_nolock;
    }

    
    if (cls == \_class\_getFreedObjectClass())
        return (IMP) \_freedHandler;

    
    if ((behavior & LOOKUP\_INITIALIZE)  &&  !cls->isInitialized()) {
        
        initializeNonMetaClass (\_class\_getNonMetaClass(cls, inst));
        
        
        
        
    }
    
    ...
    ...
    ...
}



```

↓↓↓

```
void initializeNonMetaClass(Class cls)
{
    ASSERT(!cls->isMetaClass());

    Class supercls;
    bool reallyInitialize = NO;

    
    supercls = cls->superclass;
    if (supercls  &&  !supercls->isInitialized()) {
        initializeNonMetaClass(supercls);
    }
    
    ...
    ...
    ...
    
#if \_\_OBJC2\_\_
        @try
#endif
        {
            
            callInitialize(cls);

            if (PrintInitializing) {
                \_objc\_inform("INITIALIZE: thread %p: finished +\[%s initialize\]",
                             objc\_thread\_self(), cls->nameForLogging());
            }
        }
#if \_\_OBJC2\_\_
        @catch (...) {
            if (PrintInitializing) {
                \_objc\_inform("INITIALIZE: thread %p: +\[%s initialize\] "
                             "threw an exception",
                             objc\_thread\_self(), cls->nameForLogging());
            }
            @throw;
        }
        @finally
#endif
        {
            
            lockAndFinishInitializing(cls, supercls);
        }
        return;
    }
    
    else if (cls->isInitializing()) {
        ...
        ...
        ...
    }
    
    else if (cls->isInitialized()) {
        
        return;
    }
    
    else {
        
        \_objc\_fatal("thread-safe class init in objc runtime is buggy!");
    }
}



```

↓↓↓

```
void callInitialize(Class cls)
{
    ((void(\*)(Class, SEL))objc\_msgSend)(cls, SEL\_initialize);
    asm("");
}


```

最终来到了消息发送`objc_msgSend`函数，给类对象发送`initialize`消息，消息发送机制通过`isa`指针找到对应的方法与实现。

上面我们说到的如果子类没有实现`initialize`方法，那么当我们有多个类继承了父类的时候，父类的`initialize`方法有可能会调用多次：

```
\[\[Student alloc\] init\];
\[\[Student1 alloc\] init\];


```

`Student`和`Student1`都继承自`Person`，只实现了两个子类的`initialize`方法，没有实现父类的，那么当第一次调用两个子类的时候会输出如下：

```
2020-04-28 22:20:48.690528+0800 分类Category的本质\[92558:17870282\] Person initialize
2020-04-28 22:20:48.690715+0800 分类Category的本质\[92558:17870282\] Person initialize
2020-04-28 22:20:48.690859+0800 分类Category的本质\[92558:17870282\] Person initialize


```

我们发现父类的`initialize`方法调用了 3 次，我们通过上面的源码我们已经知道了只有当类在第一次接收消息的时候才会被调用，也就是说每个类只会`initialize`一次，但是父类的 \`\`initialize\`\`\` 为什么会被多次调用呢？

我们通过底层的逻辑将上面的代码转化为伪代码：

```
if (Student没有调用initialize) {
    if (Student的父类Person没有调用initialize) {
        
        objc\_msgSend)(Person类, SEL\_initialize)
    }
}


objc\_msgSend)(Student类, SEL\_initialize)


if (Student1没有调用initialize) {
    
    if (Student1的父类Person没有调用initialize) {
        
        objc\_msgSend)(Person类, SEL\_initialize)
    }
}


objc\_msgSend)(Student1类, SEL\_initialize)



```

所以其实相当于最后发送了 3 次消息：

```
objc\_msgSend)(Person类, SEL\_initialize)
objc\_msgSend)(Student类, SEL\_initialize)
objc\_msgSend)(Student1类, SEL\_initialize)


```

消息发送机制是通过`isa`指针寻找到各自的元类对象中的类方法`initialize`的实现，但是两个子类都没有实现`initialize`方法，所以会通过`super_class`指针找到父类的实现，所有最后才会来到父类的`initialize`方法。

注意：虽然父类被多次调用`initialize`方法，但是父类也是只初始化了一次。

#### 4.3 总结

##### `load`、`initialize`的区别，以及它们在`category`重写的时候的调用的次序。

区别在于调用方式和调用时刻

调用方式：`load`是根据函数地址直接调用，`initialize`是通过`objc_msgSend`消息发送调用

调用时刻：`load`是`runtime`加载类、分类的时候调用（只会调用 1 次），`initialize`是类第一次接收到消息的时候调用，每一个类只会`initialize`一次（父类的`initialize`方法可能会被调用多次）

调用顺序：先调用本类的`load`方法，先编译那个类，就先调用`load`。在调用`load`之前会先调用父类的`load`方法。分类中`load`方法不会覆盖本类的`load`方法，先编译的分类优先调用`load`方法。`initialize`先初始化父类，之后再初始化子类。如果子类没有实现`+initialize`，会调用父类的`+initialize`（所以父类的`+initialize`可能会被调用多次），如果分类实现了`+initialize`，就覆盖类本身的`+initialize`调用。

"小礼物走一走，来简书关注我"

还没有人赞赏，支持一下

[![](https://upload.jianshu.io/users/upload_avatars/14929066/e2df19dd-3e29-41eb-91a3-3bc5762cc419.png?imageMogr2/auto-orient/strip|imageView2/1/w/100/h/100/format/webp)](https://www.jianshu.com/u/a53f6c36858c)

[CoderJRHuo](https://www.jianshu.com/u/a53f6c36858c "CoderJRHuo") 人这一辈子没法做太多的事情，所以每一件都要做得精彩绝伦，慢一点，追本溯源！

总资产 0.246 (约 0.02 元) 共写了 4.6W 字获得 5 个赞共 0 个粉丝

### 被以下专题收入，发现更多相似内容