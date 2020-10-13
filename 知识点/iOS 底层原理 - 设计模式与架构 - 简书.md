\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[www.jianshu.com\](https://www.jianshu.com/p/c2115e4c3ed8)

[![](https://upload.jianshu.io/users/upload_avatars/1319505/9754fac0eb59?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96/format/webp)](https://www.jianshu.com/u/40dce98b3416)

0.2412020.08.20 14:47:41 字数 1,384 阅读 69

#### 面试题引发的思考：

**Q: 用过哪些设计模式？**

*   **iOS 中主要使用单例模式、代理模式、观察者模式 (通知、KVO)。**

**Q: 描述对 MVC、MVP、MVVM 模式的理解？**

*   **分析如下：**

![](http://upload-images.jianshu.io/upload_images/1319505-2f24b8b683cbe3d5.png)

MVC 模式

![](http://upload-images.jianshu.io/upload_images/1319505-1a495dcefe851daf.png)

MVC 模式变种

![](http://upload-images.jianshu.io/upload_images/1319505-1e024b089ce781d9.png)

MVP 模式

![](http://upload-images.jianshu.io/upload_images/1319505-3cff9c728e136e2c.png)

MVVM 模式

* * *

### 1\. 架构

> **架构 (Architecture)**
> 
> *   **软件开发中的设计方案；**
> *   **用来处理类与类之间、模块与模块之间、客户端与服务端之间的关系。**
> 
> **架构相关名词：**
> 
> *   **MVC、MVP、MVVM**
> *   **三层架构、四层架构**
> *   **......**

* * *

#### (1) MVC(Model-View-Controller)

MVC 模式如下图：

![](http://upload-images.jianshu.io/upload_images/1319505-2f24b8b683cbe3d5.png)

MVC 模式

1.  **`Controller`**创建并持有**`View`**，并且把**`View`**添加到窗口上显示；
2.  **`View`**通知**`Controller`**处理事件；  
    (比如**`View`**内部的点击事件、滚动事件，通知**`Controller`**去处理这些业务逻辑。)
3.  **`Controller`**发送网络请求或解析数据库加载数据，并且**`Controller`**拥有和管理 **`Model`**；
4.  **`Model`**发生改变，**`Controller`**会将最新的**`Model`**显示到**`View`**上面去。

由以上分析可知：

> **`Controller`是`Model`和`View`的桥梁，`Model`和`View`相互独立。**

MVC 模式最经典的案例是`UITableView`，案例如下：

```
@interface MYShopModel : NSObject
@property (nonatomic, copy) NSString \*name;
@property (nonatomic, copy) NSString \*price;
@end

@implementation MYShopModel
@end


@interface MYShopViewController : UITableViewController
@end

@interface MYShopViewController ()
@property (nonatomic, strong) NSMutableArray \*shopArray;
@end

@implementation MYShopViewController
- (void)viewDidLoad {
    \[super viewDidLoad\];
    
    \[self loadshopArray\];
}
- (void)loadshopArray {
    self.shopArray = \[\[NSMutableArray alloc\] init\];
    for (int i = 0; i < 20; i++) {
        MYShopModel \*shopModel = \[\[MYShopModel alloc\] init\];
        shopModel.name = \[NSString stringWithFormat:@"商品-%d", i\];
        shopModel.price = \[NSString stringWithFormat:@"￥19.%d", i\];
        \[self.shopArray addObject:shopModel\];
    }
}
#pragma mark -
#pragma mark - tableView代理方法
- (NSInteger)tableView:(UITableView \*)tableView numberOfRowsInSection:(NSInteger)section {
    return self.shopArray.count;
}
- (UITableViewCell \*)tableView:(UITableView \*)tableView cellForRowAtIndexPath:(NSIndexPath \*)indexPath {
    UITableViewCell \*cell = \[tableView dequeueReusableCellWithIdentifier:@"NewsCell" forIndexPath:indexPath\];
    MYShopModel \*shopModel = self.shopArray\[indexPath.row\];
    cell.detailTextLabel.text = shopModel.price;
    cell.textLabel.text = shopModel.name;
    return cell;
}
- (void)tableView:(UITableView \*)tableView didSelectRowAtIndexPath:(NSIndexPath \*)indexPath {
    NSLog(@"didSelectRowAtIndexPath");
}
@end


```

1.  `MYShopViewController`继承于`UITableViewController`，`UITableViewController`拥有并自动创建了`UITableView`添加到内部，所以①符合；
2.  `UITableView`点击事件是`MYShopViewController`内部的`didSelectRowAtIndexPath`方法处理的，所以②符合；
3.  `MYShopViewController`加载`MYShopModel`，并且保存到`MYShopViewController`里面，所以③符合；
4.  当数据发生改变时，`MYShopViewController`会刷新`cellForRowAtIndexPath`更新`UITableView`的信息展示，所以④符合。

由以上分析可知：

> **MVC 的优缺点：**
> 
> *   **优点：`View`、`Model`可以重复利用，可以独立使用；**
> *   **缺点：`Controller`的代码过于臃肿。**

* * *

#### (2) MVC(Model-View-Controller) 变种

MVC 变种是为了解决 MVC 中**`Controller`**的代码过于臃肿的问题。

MVC 变种模式如下图：

![](http://upload-images.jianshu.io/upload_images/1319505-1a495dcefe851daf.png)

MVC 模式变种

```
@interface MYAppModel : NSObject
@property (nonatomic, copy) NSString \*name;
@property (nonatomic, copy) NSString \*image;
@end

@implementation MYAppModel
@end


@class MYAppView;
@protocol MYAppViewDelegate <NSObject>
@optional
- (void)appViewClicked:(MYAppView \*)appView;
@end

@interface MYAppView : UIView

@property (nonatomic, strong) MYAppModel \*appModel;
@property (nonatomic, weak) id<MYAppViewDelegate> delegate;
@end

@interface MYAppView ()

@property (nonatomic, weak) UIImageView \*iconView;
@property (nonatomic, weak) UILabel \*nameLabel;
@end

@implementation MYAppView
- (instancetype)initWithFrame:(CGRect)frame {
    self = \[super initWithFrame:frame\];
    if (self) {
        UIImageView \*iconView = \[\[UIImageView alloc\] init\];
        iconView.frame = CGRectMake(0, 0, 100, 100);
        \[self addSubview:iconView\];
        \_iconView = iconView;

        UILabel \*nameLabel = \[\[UILabel alloc\] init\];
        nameLabel.frame = CGRectMake(0, 100, 100, 30);
        nameLabel.textAlignment = NSTextAlignmentCenter;
        \[self addSubview:nameLabel\];
        \_nameLabel = nameLabel;
    }
    return self;
}

- (void)setAppModel:(MYAppModel \*)appModel {
    \_appModel = appModel;
    self.iconView.image = \[UIImage imageNamed:appModel.image\];
    self.nameLabel.text = appModel.name;
}

- (void)touchesBegan:(NSSet<UITouch \*> \*)touches withEvent:(UIEvent \*)event {
    if (\[self.delegate respondsToSelector:@selector(appViewClicked:)\]) {
        \[self.delegate appViewClicked:self\];
    }
}
@end


@end@interface ViewController () <MYAppViewDelegate>
@end

@implementation ViewController
- (void)viewDidLoad {
    \[super viewDidLoad\];
    
    MYAppView \*appView = \[\[MYAppView alloc\] init\];
    appView.frame = CGRectMake(100, 100, 100, 150);
    appView.delegate = self;
    \[self.view addSubview:appView\];
    
    MYAppModel \*appModel = \[\[MYAppModel alloc\] init\];
    appModel.name = @"QQ";
    appModel.image = @"QQ";
    
    appView.appModel = appModel;
}
#pragma mark -
#pragma mark - MYAppViewDelegate
- (void)appViewClicked:(MYAppView \*)appView {
    NSLog(@"控制器监听到了appView的点击");
}
@end


```

> **MVC 变种：**
> 
> *   **一个`View`对应一个`Model`；**
> *   **`View`的控件不再暴露，给`View`赋值的逻辑被封装在`View`内部；**
> *   **其他类使用`View`只需给`View`传入一个`Model`即可。**

由以上分析可知：

> **MVC 变种的优缺点：**
> 
> *   **优点：对`Controller`进行瘦身，封装`View`内部的细节，外界不知道`View`内部的具体实现；**
> *   **缺点：`View`依赖于`Model`，不能独立使用。**

* * *

#### (3) MVP(Model-View-Presenter)

MVP 模式如下图：

![](http://upload-images.jianshu.io/upload_images/1319505-1e024b089ce781d9.png)

MVP 模式

> **MVP：**
> 
> *   **相当于用`Presenter`代替了 MVC 的`Controller`；**
> *   **`Model`和`View`相互独立。**

```
@interface MYAppModel : NSObject
@property (nonatomic, copy) NSString \*name;
@property (nonatomic, copy) NSString \*image;
@end

@implementation MYAppModel
@end


@class MYAppView;
@protocol MYAppViewDelegate <NSObject>
@optional

- (void)appViewClicked:(MYAppView \*)appView;
@end

@interface MYAppView : UIView

- (void)setName:(NSString \*)name image:(NSString \*)image;
@property (nonatomic, weak) id<MYAppViewDelegate> delegate;
@end

@interface MYAppView ()
@property (nonatomic, weak) UIImageView \*iconView;
@property (nonatomic, weak) UILabel \*nameLabel;
@end

@implementation MYAppView
- (instancetype)initWithFrame:(CGRect)frame {
    self = \[super initWithFrame:frame\];
    if (self) {
        UIImageView \*iconView = \[\[UIImageView alloc\] init\];
        iconView.frame = CGRectMake(0, 0, 100, 100);
        \[self addSubview:iconView\];
        \_iconView = iconView;

        UILabel \*nameLabel = \[\[UILabel alloc\] init\];
        nameLabel.frame = CGRectMake(0, 100, 100, 30);
        nameLabel.textAlignment = NSTextAlignmentCenter;
        \[self addSubview:nameLabel\];
        \_nameLabel = nameLabel;
    }
    return self;
}

- (void)setName:(NSString \*)name image:(NSString \*)image {
    self.iconView.image = \[UIImage imageNamed:image\];
    self.nameLabel.text = name;
}

- (void)touchesBegan:(NSSet<UITouch \*> \*)touches withEvent:(UIEvent \*)event {
    if (\[self.delegate respondsToSelector:@selector(appViewClicked:)\]) {
        \[self.delegate appViewClicked:self\];
    }
}
@end


@interface MYAppPresenter : NSObject
- (instancetype)initWithController:(UIViewController \*)controller;
@end

@interface MYAppPresenter() <MYAppViewDelegate>

@property (nonatomic, weak) UIViewController \*controller;
@end

@implementation MYAppPresenter
- (instancetype)initWithController:(UIViewController \*)controller {
    self = \[super init\];
    if (self) {
        self.controller = controller;

        
        MYAppView \*appView = \[\[MYAppView alloc\] init\];
        appView.frame = CGRectMake(100, 100, 100, 150);
        appView.delegate = self;
        \[controller.view addSubview:appView\];
        
        MYAppModel \*appModel = \[\[MYAppModel alloc\] init\];
        appModel.name = @"QQ";
        appModel.image = @"QQ";
        
        \[appView setName:appModel.name image:appModel.image\];
    }
    return self;
}
#pragma mark -
#pragma mark - MYAppViewDelegate
- (void)appViewClicked:(MYAppView \*)appView {
    NSLog(@"presenter监听到了appView的点击");
}
@end


@interface ViewController ()

@property (nonatomic, strong) MYAppPresenter \*presenter;
@end

@implementation ViewController
- (void)viewDidLoad {
    \[super viewDidLoad\];
    
    self.presenter = \[\[MYAppPresenter alloc\] initWithController:self\];
}
@end


```

> **MVP：**
> 
> *   **本来交给`Controller`来做的事情，现在交给`Presenter`来做了；**
> *   **控制器和`presenter`互相拥有，但`presenter`拥有`weak`类型的控制器，防止循环引用；**
> *   **`View`不拥有`Model`，又不想暴露控件，所以使用方法更新数据；**
> *   **`View`的点击事件也交给`presenter`处理了。**
> *   **示例只有一个`MYAppView`，如果有新的`View`，就用新的`presenter`来处理它的业务逻辑；**

由以上分析可知：

> **MVP 的优缺点：**
> 
> *   **优点：`Presenter`代替了 MVC 中的`Controller`；**  
>     **`View`、`Model`可以重复利用，可以独立使用；**
> *   **缺点：`Presenter`和`View`的耦合性太高；**  
>     **每个`View`对应一个`Presenter`，导致类太多。**

* * *

#### (4) MVVM(Model-View-ViewModel)

MVVM 模式如下图：

![](http://upload-images.jianshu.io/upload_images/1319505-3cff9c728e136e2c.png)

MVVM 模式

```
@interface MYAppModel : NSObject
@property (nonatomic, copy) NSString \*name;
@property (nonatomic, copy) NSString \*image;
@end

@implementation MYAppModel
@end


@class MYAppView, MYAppViewModel;
@protocol MYAppViewDelegate <NSObject>
@optional
- (void)appViewClicked:(MYAppView \*)appView;
@end

@interface MYAppView : UIView
@property (nonatomic, weak) MYAppViewModel \*viewModel;
@property (nonatomic, weak) id<MYAppViewDelegate> delegate;
@end

@interface MYAppView ()
@property (nonatomic, weak) UIImageView \*iconView;
@property (nonatomic, weak) UILabel \*nameLabel;
@end

@implementation MYAppView
- (instancetype)initWithFrame:(CGRect)frame {
    self = \[super initWithFrame:frame\];
    if (self) {
        UIImageView \*iconView = \[\[UIImageView alloc\] init\];
        iconView.frame = CGRectMake(0, 0, 100, 100);
        \[self addSubview:iconView\];
        \_iconView = iconView;
        
        UILabel \*nameLabel = \[\[UILabel alloc\] init\];
        nameLabel.frame = CGRectMake(0, 100, 100, 30);
        nameLabel.textAlignment = NSTextAlignmentCenter;
        \[self addSubview:nameLabel\];
        \_nameLabel = nameLabel;
    }
    return self;
}

- (void)setViewModel:(MYAppViewModel \*)viewModel {
    \_viewModel = viewModel;
    
    
    \_\_weak typeof(self) waekSelf = self;
    \[self.KVOController observe:viewModel keyPath:@"name" options:NSKeyValueObservingOptionNew block:^(id  \_Nullable observer, id  \_Nonnull object, NSDictionary<NSKeyValueChangeKey,id> \* \_Nonnull change) {
        waekSelf.nameLabel.text = change\[NSKeyValueChangeNewKey\];
    }\];
    
    
    \[self.KVOController observe:viewModel keyPath:@"image" options:NSKeyValueObservingOptionNew block:^(id  \_Nullable observer, id  \_Nonnull object, NSDictionary<NSKeyValueChangeKey,id> \* \_Nonnull change) {
        waekSelf.iconView.image = \[UIImage imageNamed:change\[NSKeyValueChangeNewKey\]\];
    }\];
}

- (void)touchesBegan:(NSSet<UITouch \*> \*)touches withEvent:(UIEvent \*)event {
    if (\[self.delegate respondsToSelector:@selector(appViewClicked:)\]) {
        \[self.delegate appViewClicked:self\];
    }
}
@end


@interface MYAppViewModel : NSObject

- (instancetype)initWithController:(UIViewController \*)controller;
@end

@interface MYAppViewModel() <MYAppViewDelegate>
@property (nonatomic, weak) UIViewController \*controller;
@property (copy, nonatomic) NSString \*name;
@property (copy, nonatomic) NSString \*image;
@end

@implementation MYAppViewModel

- (instancetype)initWithController:(UIViewController \*)controller {
    self = \[super init\];
    if (self) {
        self.controller = controller;
        
        
        MYAppView \*appView = \[\[MYAppView alloc\] init\];
        appView.frame = CGRectMake(100, 100, 100, 150);
        appView.delegate = self;
        appView.viewModel = self;
        \[controller.view addSubview:appView\];
        
        MYAppModel \*appModel = \[\[MYAppModel alloc\] init\];
        appModel.name = @"QQ";
        appModel.image = @"QQ";
        
        self.name = appModel.name;
        self.image = appModel.image;
        
        
    }
    return self;
}
#pragma mark -
#pragma mark - MYAppViewDelegate
- (void)appViewClicked:(MYAppView \*)appView {
    NSLog(@"viewModel监听到了appView的点击");
}
@end



@property (strong, nonatomic) MYAppViewModel \*viewModel;
@end

@implementation ViewController
- (void)viewDidLoad {
    \[super viewDidLoad\];
    
    self.viewModel = \[\[MYAppViewModel alloc\] initWithController:self\];
}
@end


```

MVVM+RAC 是最佳搭配，可以监听`ViewModel`里面属性的改变，但是 RAC 比较重，谨慎使用。这里我们使用了 FaceBook 的 [FBKVOController](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fhmilyyjm%2FFBKVOController-) 框架来监听。

> **MVP 和 MVVM：**
> 
> *   **共同点：对`Controller`进行瘦身；**  
>     **将`View`和`Model`的一些业务逻辑放在 Presenter 或 ViewModel 中；**
> *   **不同点：属性监听绑定；**  
>     **`View`拥有`ViewModel`并监听`ViewModel`内部属性的改变，当属性改变时会更新`View`。**

由以上分析可知：

> **MVVM 的优缺点：**
> 
> *   **优点：对`Controller`进行瘦身，实现双向绑定；**
> *   **缺点：类会变多、bug 不便调试。**

* * *

#### (5) 分层设计

分层设计如下：

![](http://upload-images.jianshu.io/upload_images/1319505-d238d0b634cb06e3.png)

分层设计

MVC、MVP、MVVM 属于界面层，分层设计时不同层级分别处理所在层级的任务。

新闻业务举例，代码如下：

```
@interface ViewController ()
@end

@implementation ViewController
- (void)viewDidLoad {
    \[super viewDidLoad\];
    
    \[MYNewsService loadNews:@{} success:^(NSArray \* \_Nonnull newsData) {

    } failure:^(NSError \* \_Nonnull error) {

    }\];
}
@end



@interface MYNewsService : NSObject
+ (void)loadNews:(NSDictionary \*)params success:(void (^)(NSArray \*newsData))success failure:(void (^)(NSError \*error))failure;
@end

@implementation MYNewsService
+ (void)loadNews:(NSDictionary \*)params success:(void (^)(NSArray \*newsData))success failure:(void (^)(NSError \*error))failure {
    
    \[MYDBTool loadLocalData\];
    
    
    \[MYHTTPTool GET:@"xxxx" params:nil success:^(id result) {
        
    } failure:failure\];
}
@end



@interface MYHTTPTool : NSObject
+ (void)GET:(NSString \*)URL params:(NSDictionary \*)params success:(void (^)(id result))success failure:(void (^)(NSError \*error))failure;
@end

@implementation MYHTTPTool
+ (void)GET:(NSString \*)URL params:(NSDictionary \*)params success:(void (^)(id))success failure:(void (^)(NSError \*))failure {
    
}
@end



@interface MYDBTool : NSObject
+ (void)loadLocalData;
@end

@implementation MYDBTool
+ (void)loadLocalData {

}
@end


```

以上[架构模式 Demo 代码](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2FManXiaoYi%2FArchitecture.git)点击自取。

* * *

### 2\. 架构与设计模式的区别

> **架构一般比设计模式大，架构层面的问题包括：**
> 
> *   **整个应用程序分为多少层架构；**
> *   **将类分成很多角色 (M、V、C、P、VM 等等)；**
> *   **......**
> 
> **设计模式 (Design Pattern)：**
> 
> *   **是一套被反复使用、代码设计经验的总结；**
> *   **可重用代码、更方便他人理解、保证代码可靠性；**
> *   **一般与编程语言无关，是一套比较成熟的编程思想。**

设计模式可以分为三大类：

> *   **创建型模式：对象实例化的模式，用于解耦对象的实例化过程；**  
>     **单例模式、工厂方法模式，等等。**
>     
> *   **结构型模式：把类或对象结合在一起形成一个更大的结构；**  
>     **代理模式、适配器模式、组合模式、装饰模式，等等。**
>     
> *   **行为型模式：类或对象之间如何交互，及划分责任和算法；**  
>     **观察者模式、命令模式、责任链模式，等等。**
>     
> *   **iOS 中主要使用单例模式、代理模式、观察者模式。**
>     

"小礼物走一走，来简书关注我"

还没有人赞赏，支持一下

[![](https://upload.jianshu.io/users/upload_avatars/1319505/9754fac0eb59?imageMogr2/auto-orient/strip|imageView2/1/w/100/h/100/format/webp)](https://www.jianshu.com/u/40dce98b3416)

总资产 1 (约 0.15 元) 共写了 4.8W 字获得 24 个赞共 23 个粉丝

### 被以下专题收入，发现更多相似内容