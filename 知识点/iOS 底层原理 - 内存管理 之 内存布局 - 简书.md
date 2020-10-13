\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[www.jianshu.com\](https://www.jianshu.com/p/86352ef01c49)

[![](https://upload.jianshu.io/users/upload_avatars/1319505/9754fac0eb59?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96/format/webp)](https://www.jianshu.com/u/40dce98b3416)

2019.05.22 10:08:38 字数 111 阅读 121

#### 面试题引发的思考：

**Q: 内存的几大区域？**

*   **按照内存地址由低到高分配依次为以下区域：**  
    **预留区域、代码段、数据段、堆、栈、内核区。**

* * *

### iOS 程序的内存布局

![](http://upload-images.jianshu.io/upload_images/1319505-43453d5908e20777.png)

内存布局

由以上内存分布分析图可分析一下代码：

```
int a = 10; 
int b; 

int main(int argc, char \* argv\[\]) {
    @autoreleasepool {

        static int c = 20; 
        static int d; 

        int e = 30; 
        int f; 

        NSString \*str = @"123"; 

        NSObject \*obj = \[\[NSObject alloc\] init\]; 

        NSLog(@"\\n&a =%p\\n&b =%p\\n&c =%p\\n&d =%p\\n&e =%p\\n&f =%p\\nstr=%p\\nobj=%p",
              &a, &b, &c, &d, &e, &f, str, obj);

        return UIApplicationMain(argc, argv, nil, NSStringFromClass(\[AppDelegate class\]));
    }
}


```

打印结果如下：

```
&a =0x104af7eb8
&b =0x104af7f84
&c =0x104af7ebc
&d =0x104af7f80
&e =0x7ffeeb109cdc
&f =0x7ffeeb109cd8
str=0x104af7080
obj=0x600003ba8090


```

根据内存分布分析图可知其中一些特性：

```
 
 str=0x104af7080

 
 &a =0x104af7eb8
 &c =0x104af7ebc

 
 &b =0x104af7f84
 &d =0x104af7f80

 
 obj=0x600003ba8090

 
 &f =0x7ffeeb109cd8
 &e =0x7ffeeb109cdc


```

"小礼物走一走，来简书关注我"

还没有人赞赏，支持一下

[![](https://upload.jianshu.io/users/upload_avatars/1319505/9754fac0eb59?imageMogr2/auto-orient/strip|imageView2/1/w/100/h/100/format/webp)](https://www.jianshu.com/u/40dce98b3416)

总资产 1 (约 0.16 元) 共写了 4.8W 字获得 24 个赞共 23 个粉丝

### 被以下专题收入，发现更多相似内容

### 推荐阅读[更多精彩内容](https://www.jianshu.com/)

*   一. 定时器 1.CADisplayLink、NSTimer CADisplayLink、NSTimer 会对 ta...
    
    [![](https://upload-images.jianshu.io/upload_images/103735-a4082e3b5cc314f0.png?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)](https://www.jianshu.com/p/9e26c6ac2500)
*   CADisplayLink、NSTimer 使用注意点: 1.CADisplayLink、NSTimer 会对 targ...
    
    [![](https://upload-images.jianshu.io/upload_images/2163717-afee26101f38992e.png?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)](https://www.jianshu.com/p/2f578b91083c)
*   1\. 下面代码执行结果如何 运行结果 分析: 因为 data 是 copy 属性，所以在其 set 方法里先执行判断，然后执行 re...
    
    [![](https://upload-images.jianshu.io/upload_images/1653926-7644658f002fe273.png?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)](https://www.jianshu.com/p/869ca6fbdd5c)
*   Ajax 介绍 Ajax 最早产生于 2005 年，Ajax 表示 Asynchronous JavaScript and X...
    
    [![](https://upload-images.jianshu.io/upload_images/1696952-578bab34a9a805de.png?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)](https://www.jianshu.com/p/46fd608c58dc)
*   文章结构 1. 内存管理的基本规则 2.autoreleasePool3.ARC 管理方法 3.1 ARC 引入的四个 ow...
    
    [![](https://upload-images.jianshu.io/upload_images/2461501-4af51201cc5e0180.png?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)](https://www.jianshu.com/p/0071d57f361f)