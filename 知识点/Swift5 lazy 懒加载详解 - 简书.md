\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[www.jianshu.com\](https://www.jianshu.com/p/790971fc253a)

[![](https://upload.jianshu.io/users/upload_avatars/2974330/69c65ad4-f81f-4247-9bac-93ced9fa33f6.jpeg?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96/format/webp)](https://www.jianshu.com/u/9e7a872dc4e0)

0.1022020.09.18 10:41:28 字数 1,123 阅读 34

> lazy var 本质上是声明并执行的闭包，或一个有返回值的函数调用  
> **只执行一次**，使用的时候一定不为空  
> 瞬发闭包（Immdiately-applied closures），它是自动 @noescape 的。这就意味着在这个闭包中无需加 \[unowned self\]：这里不会产生引用循环。

*   分配独立的内存空间
*   消耗内存的
*   值一旦产生就不会再被改变：不会重走懒加载创建代码，所以懒加载控件不能赋值为 nil

用法一：控件懒加载
---------

```
lazy var btn = { () -> UIButton in
    
    
    return UIButton(type: .custom)
}()


```

常见写法

```
lazy var btn: UIButton = {
    return UIButton(type: .custom)
}()


lazy var btn: UIButton = UIButton(type: .custom)


```

用法二：避免必选属性在初始化阶段进行不必要的操作
------------------------

例如：如果没有 lazy，因为初始化阶段必选属性必须有值，smallImage 必须经历不必要的初始化。

```
class Avatar {
  lazy var smallImage: UIImage = self.image.resizedTo(Avatar.defaultSmallSize)

  var image: UIImage
  init(image: UIImage) {
    self.image = largeImage
  }
}


```

用法三：lazy sequences
------------------

> 适用于 SequenceType 和 CollectionType  
> 在高阶函数 (map flatMap) 之前加上 lazy

> 实际上，这些类型其实就是保留了一个对 “原序列” 的引用，又保留了一个对 “待调用闭包” 的引用，然后只在某个元素被访问时再对这个元素调用该闭包，做出实际的计算。  
> 弊端：计算出的返回值并没有被缓存（memoization），再次调用，再次进入闭包走流程

```
func increment(x: Int) -> Int {
  print("Computing next value of \\(x)")
  return x+1
}

let array = Array(0..<1000)
let incArray = array.map(increment)
print("Result:")
print(incArray\[0\], incArray\[4\])







```

对这段代码来说，在我们访问 incArray 的值之前，所有的输出值都被计算出来了。所以在 print("Result:")被执行之前你会看到有 1000 行 Computing next value of …！即使我们只读了 \[0\] 和\[4\]这两个条目，根本就没关心其他剩下的。

解决办法：Swift 标准库中，SequenceType 和 CollectionType 协议都有个叫 lazy 的计算属性，它能给我们返回一个特殊的 LazySequence 或者 LazyCollection。这些类型只能被用在 map，flatMap，filter 这样的高阶函数中，而且是以一种惰性的方式。

```
let array = Array(0..<1000)
let incArray = array.lazy.map(increment)
print("Result:")
print(incArray\[0\], incArray\[4\])






```

那些值被使用时才调用 increment 函数，而不是调用 map 的时候。并且只对那些被访问到的值使用，而不是对整个数组里面一千个值都使用！

对那些涉及到庞大的序列（比如这个有 1000 个元素的数组）、以及高计算度闭包的情景来说，使用这个技巧会带来质变。

### 将惰性序列级联

> 惰性序列可以把高阶函数的调用拼接起来调用

比如你可以让一个惰性序列以这种方式调用 map（或者 flatMap）：

```
func increment(x: Int) -> Int {
  print("Computing next value of \\(x)")
  return x+1
}
func double(x: Int) -> Int {
  print("Computing double value of \\(x)")
  return 2\*x
}
let array = Array(0..<1000)
let doubleArray = array.lazy.map(increment).map(double)
print(doubleArray\[3\])
print(doubleArray\[3\])









```

这样只有当 array\[3\] 被访问时，double(increment(array\[3\])) 才会被执行，被访问之前不会有这个计算，数组的其他元素也不会有这个计算！

与之相对，如果使用 array.map(increment).map(double)\[3\]（不带 lazy）会首先对整个 array 序列的所有元素进行计算，所有结果都计算出来之后再提取出第 4 个元素。更糟糕的是对数组的迭代要进行两次，每个 map 都会有一次。这对计算时间（computational time）来说是怎样的一种浪费！

不可 lazy let 的原因
---------------

这是由 lazy 的具体实现细节决定的：它在没有值的情况下以某种方式被初始化，然后在被访问时改变自己的值，这就要求该属性是可变的。

隐式 lazy
-------

> 在 class 中使用 static let 是 Swift 创建单例的最佳实践，原因在于 static let 是惰性的、线程安全的，而且只能被创建一次。

被声明在全局作用域下、或者被声明为一个类型属性（声明为 static let、而非声明为实例属性）的`常量`是自动具有惰性（lazy）的（还是线程安全的）

```
let foo: Int = {
  print("Global constant initialized")
  return 42
}()

class Cat {
  static let defaultName: String = {
    print("Type constant initialized")
    return "Felix"
  }()
}

@UIApplicationMain
class AppDelegate: UIResponder, UIApplicationDelegate {
  func application(application: UIApplication, didFinishLaunchingWithOptions launchOptions: \[NSObject: AnyObject\]?) -> Bool {
    print("Hello")
    print(foo)
    print(Cat.defaultName)
    print("Bye")
    return true
  }
}









```

证明了 foo 和 Cat.defaultName 这两个常量只在被访问时才被创建，而非初始化时创建。

别把这个和 class 或结构体里面的实例属性的情况搞混了。如果你声明一个 struct Foo ，那 bar 这个实例属性会在一个 Foo 实例被创建的时候就被计算出来（作为其初始化的一部分），而不是以惰性的形式。

```
struct Foo { 
    let bar = Bar() 
}


```

参考资料：  
[“懒” 点儿好](https://links.jianshu.com/go?to=https%3A%2F%2Fswift.gg%2F2016%2F03%2F25%2Fbeing-lazy%2F)

[![](https://upload.jianshu.io/users/upload_avatars/2974330/69c65ad4-f81f-4247-9bac-93ced9fa33f6.jpeg?imageMogr2/auto-orient/strip|imageView2/1/w/100/h/100/format/webp)](https://www.jianshu.com/u/9e7a872dc4e0)

总资产 2 (约 0.21 元) 共写了 1.4W 字获得 11 个赞共 7 个粉丝

### 被以下专题收入，发现更多相似内容