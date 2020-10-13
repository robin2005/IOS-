\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[www.jianshu.com\](https://www.jianshu.com/p/b592ee20f09a)

[![](https://upload.jianshu.io/users/upload_avatars/1929699/53819a2fc1ea?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96/format/webp)](https://www.jianshu.com/u/fd4f9c1d72e2)

92019.01.07 14:39:33 字数 2,900 阅读 3,556

iOS 缓存设计（YYCache 思路）

> 前言：  
> 前段时间业务有缓存需求，于是结合 YYCache 和业务需求，做了缓存层（内存 & 磁盘）+ 网络层的方案尝试  
> 由于 YYCache 采用了内存缓存和磁盘缓存组合方式，性能优良，这里拿它的原理来说下如何设计一套缓存的思路，并结合网络整理一套完整流程

*   初步认识缓存
*   如何优化缓存 (YYCache 设计思想)
*   网络和缓存同步流程

1\. 什么是缓存？
----------

我们做一个缓存前，先了解它是什么，缓存是本地数据存储，存储方式主要包含两种：磁盘储存和内存存储

### 1.1 磁盘存储

磁盘缓存，磁盘也就是硬盘缓存，磁盘是程序的存储空间，磁盘缓存容量大速度慢，磁盘是永久存储东西的，iOS 为不同数据管理对存储路径做了规范如下：  
1、每一个应用程序都会拥有一个应用程序沙盒。  
2、应用程序沙盒就是一个文件系统目录。  
沙盒根目录结构：Documents、Library、temp。

磁盘存储方式主要有文件管理和数据库，其特性：

![](http://upload-images.jianshu.io/upload_images/1929699-e9724d8ddf1838e9.png)

image.png

### 1.2 内存存储

内存缓存，内存缓存是指当前程序运行空间，内存缓存速度快容量小，它是供 cpu 直接读取，比如我们打开一个程序，他是运行在内存中的，关闭程序后内存又会释放。  
iOS 内存分为 5 个区：栈区，堆区，全局区，常量区，代码区

**栈区 stack：**这一块区域系统会自己管理，我们不用干预，主要存一些局部变量，以及函数跳转时的现场保护。因此大量的局部变量, 深递归，函数循环调用都可能导致内存耗尽而运行崩溃。  
**堆区 heap：**与栈区相对，这一块一般由我们自己管理，比如 alloc，free 的操作，存储一些自己创建的对象。  
**全局区（静态区 static）：**全局变量和静态变量都存储在这里，已经初始化的和没有初始化的会分开存储在相邻的区域，程序结束后系统会释放  
**常量区：**存储常量字符串和 const 常量  
**代码区：**存储代码

在程序中声明的容器（数组 、字典）都可看做内存中存储，特性如下：

![](http://upload-images.jianshu.io/upload_images/1929699-e414d652b3d9525b.png)

image.png

2\. 缓存做什么？
----------

我们使用场景比如：离线加载，预加载，本地通讯录... 等，对非网络数据，使用本地数据管理的一种，具体使用场景有很多

3\. 怎么做缓存？
----------

简单缓存可以仅使用磁盘存储，iOS 主要提供四种磁盘存储方式：

*   `NSKeyedArchiver`: 采用归档的形式来保存数据, 该数据对象需要遵守 NSCoding 协议, 并且该对象对应的类必须提供 encodeWithCoder: 和 initWithCoder: 方法.

```
#import <Foundation/Foundation.h>
@interface Person : NSObject<NSCoding>
@property(nonatomic,copy) NSString \* name;

@end


#import "Person.h"
@implementation Person

- (void)encodeWithCoder:(NSCoder \*)aCoder {
    \[aCoder encodeObject:\_name forKey:@"name"\];
}

- (instancetype)initWithCoder:(NSCoder \*)aDecoder {
    if (self = \[super init\]) {
        \_name = \[aDecoder decodeObjectForKey:@"name"\];
    }
    return self;
}
@end


```

使用归档解档

```
  
  \[NSKeyedArchiver archiveRootObject:p toFile:path\];
  
  Person \*p = \[NSKeyedUnarchiver unarchiveObjectWithFile:path\];


```

缺点: 归档的形式来保存数据, 只能一次性归档保存以及一次性解压。所以只能针对小量数据, 如果想改动数据的某一小部分, 需要解压整个数据或者归档整个数据。

*   `NSUserDefaults`: 用来保存应用程序设置和属性、用户保存的数据。用户再次打开程序或开机后这些数据仍然存在。  
    NSUserDefaults 可以存储的数据类型包括: NSData、NSString、NSNumber、NSDate、NSArray、 NSDictionary。

```
  \[\[NSUserDefaults standardUserDefaults\] setObject:@"value" forKey:@"key"\];

  \[\[NSUserDefaults standardUserDefaults\] objectForKey:@"key"\];


```

*   `Write写入方式`: 永久保存在磁盘中。具体方法为:

```
  
  \[data writeToFile:FileName atomically:YES\];
  
  NSData  \*data=\[NSData dataWithContentsOfFile:FileName options:0 error:NULL\];


```

*   `SQLite`: 采用 SQLite 数据库来存储数据。SQLite 作为⼀一中小型数据库, 应用 ios 中跟其他三种保存方式相比, 相对复杂一些

```
  
  if (sqlite3\_open(\[databaseFilePath UTF8String\],   &database)==SQLITE\_OK) {
      NSLog(@"sqlite dadabase is opened."); 
  } else { return;}

  
  char \*error;
  const char \*createSql="create table(id integer primary key autoincrement, name text)"; if (sqlite3\_exec(database, createSql, NULL, NULL, &error)==SQLITE\_OK) {
      NSLog(@"create table is ok."); 
  } else {
      sqlite3\_free(error);
  }

  
  const char \*insertSql="insert into a person (name) values(‘gg’)";
  if (sqlite3\_exec(database, insertSql, NULL, NULL, &error)==SQLITE\_OK) {
      NSLog(@"insert operation is ok."); 
  } else {
      sqlite3\_free(error);
  }


```

上面提到的磁盘存储特性，具备空间大、可持久、但是读取慢，面对大量数据频繁读取时更加明显，以往测试中磁盘读取比内存读取保守测量低于几十倍，那我们怎么解决磁盘读取慢的缺点呢? 又如何利用内存的优势呢？

_YYCache 背景知识：_  
源码中由两个主要类构成

![](http://upload-images.jianshu.io/upload_images/1929699-20b6ebcceb9ff7ad.png)

Snip20190107\_2.png

*   YYMemoryCache (内存缓存)  
    操作 YYLinkedMap 中数据， 为实现内存优化，采用双向链表数据结构实现 LRU 算法，YYLinkedMapItem 为每个子节点
*   YYDiskCache （磁盘缓存）  
    不会直接操作缓存对象（sqlite/file），而是通过 YYKVStorage 来间接的操作缓存对象。

容量管理：

*   ageLimit ：时间周期限制，比如每天或每星期开始清理
*   costLimit： 容量限制，比如超出 10M 后开始清理内存
*   countLimit ： 数量限制， 比如超出 1000 个数据就清理

这里借用 YYCache 设计, 来讲述缓存优化

#### 1\. 磁盘 + 内存组合优化

利用内存和磁盘特性，融合各自优点，整合如下：

![](http://upload-images.jianshu.io/upload_images/1929699-efef57a62afede5b.png)

image.png

*   APP 会优先请求内存缓冲中的资源
*   如果内存缓冲中有，则直接返回资源文件， 如果没有的话，则会请求资源文件，这时资源文件默认资源为本地磁盘存储，需要操作文件系统或数据库来获取。
*   获取到的资源文件，先缓存到内存缓存，方便以后不再重复获取，节省时间。  
    然后就是从缓存中取到数据然后给 app 使用。  
    这样就充分结合两者特性，利用内存读取快特性减少读取数据时间，

`YYCache 源码解析`：

```
\- (id<NSCoding>)objectForKey:(NSString \*)key {
    
    id<NSCoding> object = \[\_memoryCache objectForKey:key\];
    if (!object) {
        
        object = \[\_diskCache objectForKey:key\];
        if (object) {
            
            \[\_memoryCache setObject:object forKey:key\];
        }
    }
    return object;
}


```

#### 2\. 内存优化 -- 提高内存命中率

但是我们想在基础上再做优化，比如想让经常访问的数据保留在内存中，提高内存的命中率，减少磁盘的读取，那怎么做处理呢？ -- LRU 算法

![](http://upload-images.jianshu.io/upload_images/1929699-37c789a04de1c0f4.png)

LRU 算法. png

LRU 算法：我们可以将链表看成一串数据链，每个数据是这个串上的一个节点，经常访问的数据移动到头部，等数据超出容量后从链表后面的一些节点销毁，这样经常访问数据在头部位置，还保留在内存中。

链表实现结构图：

![](http://upload-images.jianshu.io/upload_images/1929699-853dd9d2420a8bed.jpg)

\_LinkedMap.jpg

`YYCache 源码解析`

```
@interface \_YYLinkedMapNode : NSObject {
    @package
    \_\_unsafe\_unretained \_YYLinkedMapNode \*\_prev; 
    \_\_unsafe\_unretained \_YYLinkedMapNode \*\_next; 
    id \_key;
    id \_value;
    NSUInteger \_cost;
    NSTimeInterval \_time;
}
@end
@implementation \_YYLinkedMapNode
@end

@interface \_YYLinkedMap : NSObject {
    @package
    CFMutableDictionaryRef \_dic; 
    NSUInteger \_totalCost;
    NSUInteger \_totalCount;
    \_YYLinkedMapNode \*\_head; 
    \_YYLinkedMapNode \*\_tail; 
    BOOL \_releaseOnMainThread;
    BOOL \_releaseAsynchronously;
}



- (void)insertNodeAtHead:(\_YYLinkedMapNode \*)node;



- (void)bringNodeToHead:(\_YYLinkedMapNode \*)node;



- (void)removeNode:(\_YYLinkedMapNode \*)node;


- (\_YYLinkedMapNode \*)removeTailNode;


- (void)removeAll;

@end


```

`_YYLinkedMapNode *_prev` 为该节点的头指针，指向前一个节点  
`_YYLinkedMapNode *_next`为该节点的尾指针，指向下一个节点  
头指针和尾指针将一个个子节点串连起来，形成双向链表

来看下`bringNodeToHead:`的源码实现，它是实现 LRU 算法主要方法，移动 node 子结点到链头。

（详细已注释在代码中）

```
\- (void)bringNodeToHead:(\_YYLinkedMapNode \*)node {
    if (\_head == node) return; 
    
    
    if (\_tail == node) {
        \_tail = node->\_prev; 
        \_tail->\_next = nil; 
    } else {
        
        node->\_next->\_prev = node->\_prev; 
        node->\_prev->\_next = node->\_next; 
    }
    node->\_next = \_head; 
    node->\_prev = nil; 
    \_head->\_prev = node; 
    \_head = node; 
}


```

其他方法就不挨个举例了，具体可翻看源码，这些代码结构清晰，类和函数遵循单一职责，接口高内聚，低耦合，是个不错的学习示例！

#### 3\. 磁盘优化 - 数据分类存储

YYDiskCache 是一个线程安全的磁盘缓存，基于 sqlite 和 file 来做的磁盘缓存，我们的缓存对象可以自由的选择存储类型，  
下面简单对比一下：

*   sqlite: 对于小数据（例如 NSNumber）的存取效率明显高于 file。
*   file: 对于较大数据（例如高质量图片）的存取效率优于 sqlite。

所以 YYDiskCache 使用两者配合，灵活的存储以提高性能。

另外：  
YYDiskCache 具有以下功能：

*   它使用 LRU(least-recently-used) 来删除对象。
*   支持按 cost，count 和 age 进行控制。
*   它可以被配置为当没有可用的磁盘空间时自动驱逐缓存对象。
*   它可以自动抉择每个缓存对象的存储类型（sqlite/file）以便提供更好的性能表现。

`YYCache源码解析`

```
@interface YYKVStorageItem : NSObject
@property (nonatomic, strong) NSString \*key;                
@property (nonatomic, strong) NSData \*value;                
@property (nullable, nonatomic, strong) NSString \*filename; 
@property (nonatomic) int size;                             
@property (nonatomic) int modTime;                          
@property (nonatomic) int accessTime;                       
@property (nullable, nonatomic, strong) NSData \*extendedData; 
@end
 
 

@interface YYKVStorage : NSObject
 
#pragma mark - Attribute
@property (nonatomic, readonly) NSString \*path;        
@property (nonatomic, readonly) YYKVStorageType type;  
@property (nonatomic) BOOL errorLogsEnabled;           
 
#pragma mark - Initializer
- (nullable instancetype)initWithPath:(NSString \*)path type:(YYKVStorageType)type NS\_DESIGNATED\_INITIALIZER;
 
#pragma mark - Save Items
- (BOOL)saveItem:(YYKVStorageItem \*)item;
...
 
#pragma mark - Remove Items
- (BOOL)removeItemForKey:(NSString \*)key;
...
 
#pragma mark - Get Items
- (nullable YYKVStorageItem \*)getItemForKey:(NSString \*)key;
...
 
#pragma mark - Get Storage Status
- (BOOL)itemExistsForKey:(NSString \*)key;
- (int)getItemsCount;
- (int)getItemsSize;
 
@end


```

我们只需要看一下 YYKVStorageType 这个枚举，它决定着 YYKVStorage 的存储类型。

YYKVStorageType

```
typedef NS\_ENUM(NSUInteger, YYKVStorageType) {
    YYKVStorageTypeFile = 0, 
    YYKVStorageTypeSQLite = 1, 
    YYKVStorageTypeMixed = 2, 
};


```

### 总结：

这里说了 YYCache 几个主要设计优化之处，其实细节上也有很多不错的处理，比如：

*   线程安全  
    如果说 YYCache 这个类是一个纯逻辑层的缓存类（指 YYCache 的接口实现全部是调用其他类完成），那么 YYMemoryCache 与 YYDiskCache 还是做了一些事情的（并没有 YYCache 当甩手掌柜那么轻松），其中最显而易见的就是 YYMemoryCache 与 YYDiskCache 为 YYCache 保证了线程安全。  
    YYMemoryCache 使用了 pthread\_mutex 线程锁来确保线程安全，而 YYDiskCache 则选择了更适合它的 dispatch\_semaphore，上文已经给出了作者选择这些锁的原因。
    
*   性能
    

YYCache 中对于性能提升的实现细节：

1.  异步释放缓存对象
2.  锁的选择
3.  使用 NSMapTable 单例管理的 YYDiskCache
4.  YYKVStorage 中的 \_dbStmtCache
5.  甚至使用 CoreFoundation 来换取微乎其微的性能提升

结合网络层和缓存层，设计了一套接口缓存方式，比较灵活且速度得到提升; 比如首页界面可能由多个接口提供数据，没有采用整块存储而是将存储细分到每个接口中，有 API 接口控制，基本结构如下：

主要分为：

*   应用层 ：显示数据
*   管理层： 管理网络层和缓存层，为应用层提供数据支持
*   网络层： 请求网络数据
*   缓存层： 缓存数据

层级图：

![](http://upload-images.jianshu.io/upload_images/1929699-190129d794421eb3.png)

缓存流程. png

1.  服务端每套数据对应一个 version （或时间戳），若后台数据发生变更，则 version 发生变化，在返回客户端数据时并将 version 一并返回。
2.  当客户端请求网络时，将本地上一次数据对应 version 上传。
3.  服务端获取客户端传来得 version 后，与最新的 version 进行对比，若 version 不一致，则返回最新数据，若未发生变化，服务端不需要返回全部数据只需返回 304（No Modify) 状态值
4.  客户端接到服务端返回数据，若返回全部数据非 304，客户端则将最新数据同步到本地缓存中；客户端若接到 304 状态值后，表示服务端数据和本地数据一致，直接从缓存中获取显示  
    这也是 ETag 的大致流程；详细可以查看 [https://baike.baidu.com/item/ETag/4419019?fr=aladdin](https://links.jianshu.com/go?to=https%3A%2F%2Fbaike.baidu.com%2Fitem%2FETag%2F4419019%3Ffr%3Daladdin)

`源码示例`

```
\- (void)getDataWithPage:(NSNumber \*)page pageSize:(NSNumber \*)pageSize option:(DataSourceOption)option completion:(void (^)(HomePageListCardModel \* \_Nullable, NSError \* \_Nullable))completionBlock {
    NSString \*cacheKey = CacheKey(currentUser.userId, PlatIndexRecommendation);
   
    switch (option) {
        case DataSourceCache:
        {
            if (\[\_cache containsObjectForKey:cacheKey\]) {
                completionBlock((HomePageListCardModel \*)\[self->\_cache objectForKey:cacheKey\], nil);
            } else {
                completionBlock(nil, LJDError(400, @"缓存中不存在"));
            }
        }
            break;
        case DataSourceNetwork:
        {
            \[NetWorkServer requestDataWithPage:page pageSize:pageSize completion:^(id \_Nullable responseObject, NSError \* \_Nullable error) {
                if (responseObject && !error) {
                    HomePageListCardModel \*model = \[HomePageListCardModel yy\_modelWithJSON:responseObject\];
                    if (model.errnonumber == 304) { 
                        completionBlock((HomePageListCardModel \*)\[self->\_cache objectForKey:cacheKey\], nil);
                    } else {
                        completionBlock(model, error);
                        \[self->\_cache setObject:model forKey:cacheKey\]; 
                    }
                } else {
                    completionBlock(nil, error);
                }
            }\];
        }
            break;
             
        default:
            break;
    }
}


```

这样做好处：

*   对于不频繁更新数据的接口，节省了大量 JSON 数据转化时间
*   节约流量，节省加载时长
*   用户界面显示加快

总结：项目中并不一定完全这样做，有时候过渡设计也是一种浪费，多了解其他设计思路后，针对项目找到适合的才是最好的！

参考文献：  
YYCache： [https://github.com/ibireme/YYCache](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fibireme%2FYYCache)  
YYCache 设计思路 ：[https://blog.ibireme.com/2015/10/26/yycache/](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.ibireme.com%2F2015%2F10%2F26%2Fyycache%2F)

"如果觉得我的文章对您有用，请随意打赏。您的支持将鼓励我继续创作！"

还没有人赞赏，支持一下

[![](https://upload.jianshu.io/users/upload_avatars/1929699/53819a2fc1ea?imageMogr2/auto-orient/strip|imageView2/1/w/100/h/100/format/webp)](https://www.jianshu.com/u/fd4f9c1d72e2)

[\_方丈](https://www.jianshu.com/u/fd4f9c1d72e2 "_方丈")现代人的通病 -- 无条理，无原则，无深度，无目的! — 送给自己共勉...

总资产 46 (约 3.46 元) 共写了 3.3W 字获得 799 个赞共 729 个粉丝

### 被以下专题收入，发现更多相似内容