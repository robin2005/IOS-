\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[www.jianshu.com\](https://www.jianshu.com/p/05970babfbfc)

[![](https://upload.jianshu.io/users/upload_avatars/1948913/008144fb-c740-4829-8f85-df093af99f37.jpg?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96/format/webp)](https://www.jianshu.com/u/ad4630427de8)

0.2922020.07.15 18:42:01 字数 581 阅读 67

本文不对`AFNetworking`作全面的解析，仅对比解析一下`2.x`和`3.x`的差异。

1.  `AFNetworking`分为如下`5个功能模块`：

*   网络通信模块 (`AFURLSessionManager、AFHTTPSessionManger`)
*   网络状态监听模块 (`Reachability`)
*   网络通信安全策略模块 (`Security`)
*   网络通信信息序列化 / 反序列化模块 (`Serialization`)
*   对于`iOS UIKit`库的扩展 (`UIKit`)

2.  `AFNetworking 2.x`需要常驻线程而`3.x`不需要常驻线程  
    `2.x`常驻线程用来`并发请求和处理数据回调`，避免多个网络请求的线程开销 (`不用开辟一个线程，就保活一条线程`)；而`3.x`不需要常驻线程是因为`NSURLSession`可以指定回调`delegateQueue`，`NSURLConnection`不行；  
    `NSURLConnection`的一大痛点就是：发起请求后，需要一直处于`等待回调的状态`。而`3.x`后`NSURLSession`解决了这个问题；`NSURLSession`发起的请求，不再需要在当前线程进行回调，可以指定回调的`delegateQueue`，这样就不用为了等待代理回调方法而保活线程了
    
3.  `3.x`需要设置最大并发数为`1`(`self.operationQueue.maxConcurrentOperationCount = 1`)，`2.x`为什么不需要  
    功能不一样：`3.x`的`operationQueue`是用来接收`NSURLSessionDelegate`回调的，鉴于一些多线程数据访问的安全性考虑，设置了`maxConcurrentOperationCount = 1`来达到`并发的请求串行的进行回调`的效果。而`2.x`的`operationQueue`是用来添加`operation`进行`并发请求`的，所以不要设置为`1`  
    **注意：并发数并不等于所开辟的线程数，具体开辟几条线程由系统决定**
    
4.  `3.x`为什么要串行回调
    

```
\- (AFURLSessionManagerTaskDelegate \*)delegateForTask:(NSURLSessionTask \*)task {
    NSParameterAssert(task);
    AFURLSessionManagerTaskDelegate \*delegate = nil;
    \[self.lock lock\];
    
    delegate = self.mutableTaskDelegatesKeyedByTaskIdentifier\[@(task.taskIdentifier)\];
    \[self.lock unlock\];
    return delegate;
}


```

从代码可以看出，这边对`self.mutableTaskDelegatesKeyedByTaskIdentifier`的访问进行了加锁，目的是`保证多线程环境下的数据安全`。既然加了锁，就算`maxConcurrentOperationCount`不设为`1`，当某个请求正在回调时，下一个请求还是得等待一直到上个请求获取完所要的资源后解锁，所以这边并发回调也是没有意义的。相反多`task`回调导致的多线程并发，还会导致性能的浪费

[附：我的博客地址](https://links.jianshu.com/go?to=https%3A%2F%2Fgsl201600.github.io%2F2020%2F07%2F08%2FiOS%25E5%2588%259D%25E6%258E%25A2AFNetworking%2F)

"小礼物走一走，来简书关注我"

还没有人赞赏，支持一下

[![](https://upload.jianshu.io/users/upload_avatars/1948913/008144fb-c740-4829-8f85-df093af99f37.jpg?imageMogr2/auto-orient/strip|imageView2/1/w/100/h/100/format/webp)](https://www.jianshu.com/u/ad4630427de8)

[72 行代码](https://www.jianshu.com/u/ad4630427de8 "72行代码")[![](https://upload.jianshu.io/user_badge/d60756bb-83ca-4515-9eda-a8cd7c72e188)](https://www.jianshu.com/p/3bc50b869c89)程序是写给人读的，只是偶尔让计算机执行一下

总资产 178 (约 13.29 元) 共写了 5.0W 字获得 204 个赞共 62 个粉丝

### 被以下专题收入，发现更多相似内容