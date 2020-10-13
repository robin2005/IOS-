\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[www.jianshu.com\](https://www.jianshu.com/p/c9f854718933)

#### 关于 RxSwift 的 Observable 作用，这篇文章里有详细解释 [Swift - RxSwift 的使用详解 3（Observable 介绍、创建可观察序列） - 简书](https://www.jianshu.com/p/63f1681236fd)。本文不研究使用，只探究原理。

先看一下 Observable 的效果

![](http://upload-images.jianshu.io/upload_images/3028784-6cabeb766beceb09.png)

1

输出结果：

> 订阅到：\["1", "2", "3", "4"\]

这样就实现了，订阅效果。只要调用 onNext，那么 sbuscribe 的 onNext 就会被调用。

#### 一：先看一下 Observable 类是个什么。

点进去查看源码

![](http://upload-images.jianshu.io/upload_images/3028784-3c232b4e479bc207.png)

3

发现其继承自 ObservableType。

#### 二：我们接下来看一下 Observable 的 create。

点进去之后发现，这只是个静态方法，无法进一步查看实现。这时候我们阅读该方法的注释如图：

![](http://upload-images.jianshu.io/upload_images/3028784-74d067b73e562dcc.png)

4

从注释可以看出，官方给的路由里，有个叫 create.html 的界面。我们试一下在 Rx 的源码里，是不是也有一个叫 create 的文件。

![](http://upload-images.jianshu.io/upload_images/3028784-60b18849993103bd.png)

5

嘿嘿，果然有。点进去这个文件里，映入眼帘的就是我们要找的 create 方法的声明。

![](http://upload-images.jianshu.io/upload_images/3028784-c02b94fd04a0971e.png)

6

我们发现该方法返回了一个 AnonymousObservable 对象，并把传进来的闭包作为参数传了进去。

点进 AnonymousObservable 里，查看

![](http://upload-images.jianshu.io/upload_images/3028784-0f48263161e4050f.png)

7

进来之后我们发现，AnonymousObservable 类的初始化方法直接保存了传进来的 block。这里我们大概也就看到了其用意，因为保存起来之后，后边需要使用的时候直接就能拿来用。注意备注一，后边用的到。

这里我们还应该注意到一个点。AnonymousObservable 继承自一个叫 Producer 的类。这里后面用到再展开。

至此，我们就看到了整个创建的过程。其实就是把 create 的闭包，保存了起来。

#### 三：我们再看 subscribe 的实现逻辑。

我们把相对次要的代码删除后的截图如下

![](http://upload-images.jianshu.io/upload_images/3028784-81a5a944f84e7eed.png)

8

实现比较长，我截取核心的几点：

1、方法名。我们发现此方法把所有要实现的 block 作为参数传了进来。如下

![](http://upload-images.jianshu.io/upload_images/3028784-01ed0a5d288b817f.png)

9

2、核心逻辑。通过截图我们能看到，这里把整个闭包的实现声明成了一个临时变量。不难发现，① 本闭包的核心就是通过 event 的判断，执行不同的闭包 ；② 我们要调用此闭包，就需要调用 observer 这个对象。现在关键问题就在于 observer 调用时机。

![](http://upload-images.jianshu.io/upload_images/3028784-0bcbcb855442e9d1.png)

10

3、核心调用。 我们来到 return 方法。可以发现，在 return 方法里，把 observer 作为参数传给了一个叫 subscribe 的方法。如下图：

![](http://upload-images.jianshu.io/upload_images/3028784-478f4c2584170ad9.png)

11

另外：asObservable() 返回就是自己，见下图。这样做的目的，我猜测可能是为了强调自己是 observable。

![](http://upload-images.jianshu.io/upload_images/3028784-92513127dacc9fad.png)

12

#### 四：现在就看这个 subscribe 实现了什么

之前我们提到过，observer 的 create 方法，实际上是返回了一个 AnonymousObservable（见图 6）。而这个 AnonymousObservable 则继承自一个叫 Producer 的东西，我们来看看这个 Producer 的实现。

![](http://upload-images.jianshu.io/upload_images/3028784-3883dab934bd257f.png)

13、subscribe 方法里实际上是实现了一个 run 方法，并把这个 observer 继续传了出去

看图中的方法一，我们找到了 subscribe 这个方法。并发现了 observer 被再次传了出去

往下走，这个 run 调用的是子类的方法，也就是我们之前的备注一（图 7）的方法。我们再来看一下，这段代码：我们发现 observer 作为 AnonymousObservableSink 类的参数，参与创建了一个临时变量 sink。并执行了 sink 的 run 方法。

![](http://upload-images.jianshu.io/upload_images/3028784-b467bceb8791619a.png)

14

那么接下来，我们再看一下这个 AnonymousObservableSink 类的实现，点进去，如下图

![](http://upload-images.jianshu.io/upload_images/3028784-7df3e719b78e2663.png)

15

我们通过这个 run 方法可以清楚的看到。这里调用了之前步骤二也就是 create 方法里保存的那个闭包。并把 AnonymousObservableSink 作为参数传了进去。

#### 五：由此为什么 subscribe 能获得 onNext 的参数的铺垫逻辑，就通顺了

**第一步：在创建 Observable 类的 ob 对象时，调用的 create 方法，实际上是保存了我们的 create 实现的闭包。**

**第二步：在 ob 调用 subscribe 时，已经把订阅的实现逻辑封装到了闭包内，并且把这个闭包封装成了一个叫 observer 的临时变量，然后调用了 AnonymousObservable 的 subscribe 方法。**

**第三步：AnonymousObservable 的 subscribe 方法，默认创建了 AnonymousObservableSink 类，并把 observer 保存成自己属性，又通过该类的 run 方法调用了第一步里保存的闭包，并把自己作为参数穿进去，用以让第一步保存的闭包成功获得这个 observer。**

**第四步：这样就保证了订阅方法 subscribe 能获得 onNext 里的数据。因为 subscribe 已经持有了 Observable 创建时声明的闭包。**

#### 六：我们再来看这个 subscribe 到底是怎么获取这个 onNext 的参数的。

我们再来看一下 subscribe 的核心实现

![](http://upload-images.jianshu.io/upload_images/3028784-2a4f8e01d8b7121f.png)

16

我们清晰的看到，只要 event 是. next 事件，那就会调用 onNext 闭包。关键在于这个 event 事件是什么时候有的值。那么我们首先想到的是创建 ob 时 onNext 的实现是不是给这 event 赋值了。点进去查看

这里把. next 作为参数，调用了一个 on 方法。我们看这个 on 方法的实现

![](http://upload-images.jianshu.io/upload_images/3028784-a46ea002af076291.png)

17

看上去感觉这里好像是持有了这个 event 事件，我们继续点进去查看

![](http://upload-images.jianshu.io/upload_images/3028784-375c5474ac5edacd.png)

18

我们明显能看到，这个 forwardOn 方法，是 Sink 类的方法。而这个 sink 类就是 AnonymousObservableSink 的父类。这里确实是用 sink 持有了这个 event。

![](http://upload-images.jianshu.io/upload_images/3028784-b3a7140d0983eb6f.png)

19

#### 七：由此我们得出结论。

**第一步：在 AnonymousObservable 类调用 subscribe 时，实际上是创建了一个 sink 对象。**

**第二步：在这个 sink 类调用我们保存的闭包的时候，也把闭包内的 onNext 方法方法作为 event 事件做了持有操作。并在调用 observer 对象时，把 event 做为参数传了进来。**

**第三步：AnonymousObserver 类根据传进来的 event，调用不同的实现闭包。**

**附上流程图**

![](http://upload-images.jianshu.io/upload_images/3028784-19ae317da67432f9.jpg)