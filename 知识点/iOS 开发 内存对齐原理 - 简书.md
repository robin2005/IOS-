\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[www.jianshu.com\](https://www.jianshu.com/p/8def5db38fb2)

[![](https://upload.jianshu.io/users/upload_avatars/8868881/e4b89026-442c-48c6-828f-ac2ad1469f54.jpg?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96/format/webp)](https://www.jianshu.com/u/e63188880f2e)

2020.09.08 16:30:46 字数 578 阅读 26

*   数据成员对⻬规则：结构 (struct)(或联合(union)) 的数据成员，第一个数据成员放在 offset 为 0 的地方，以后每个数据成员存储的起始位置要从该成员大小或者成员的子成员大小 (只要该成员有子成员，比如说是数组，结构体等) 的整数倍开始(比如 int 为 4 字节，则要从 4 的整数倍地址开始存储。
*   结构体作为成员：如果一个结构里有某些结构体成员，则结构体成员要从其内部最大元素大小的整数倍地址开始存储 (struct a 里存有 struct b，b 里有 char，int ，double 等元素，那 b 应该从 8 的整数倍开始存储)。
*   收尾工作：结构体的总大小，也就是 sizeof 的结果，必须是其内部最大成员的整数倍，不足的要补⻬。

实践之前，我们先从一张图了解各数据类型所占字节数：

![](http://upload-images.jianshu.io/upload_images/8868881-3298e55ed7f17973.png)

图片. png

案例：

```
struct LGStruct1 {
    double a;    
    char b;      
    int c;       
    short d;     
}struct1;


```

分析：  
a 占 8 个字节，b 占一个字节，c 占 4 个字节，d 占 2 个字节；  
根据内存对齐原理，struct1 要从字节最大的 8 开始存起，即 a 占 0~7；  
当存储 b 的时候，存储起始位置 8 是 1 的整数倍，所以存储一个字节，即 b 占 8；  
当存储 c 的时候，存储起始位置 9 不是 4 的整数倍，则向后寻找最近的 4 的整数倍，即 c 占 12~15；  
当存储 d 的时候，存储起始位置 16 是 2 的整数倍，则存储两个字节，即 c 占 16~17；  
这样 struct1 总共的存储大小便是 0~17，即内存大小为 18；  
再根据上面对齐原理最后一条，结构体的总大小是其内部最大成员的整数倍，而 struct1 内部最大成员是 a，即 8 的整数倍，即为 18 后面的 24，所以 struct1 总大小为 24。

利用空间换时间，提高读取效率

"小礼物走一走，来简书关注我"

还没有人赞赏，支持一下

[![](https://upload.jianshu.io/users/upload_avatars/8868881/e4b89026-442c-48c6-828f-ac2ad1469f54.jpg?imageMogr2/auto-orient/strip|imageView2/1/w/100/h/100/format/webp)](https://www.jianshu.com/u/e63188880f2e)

[魔杰西](https://www.jianshu.com/u/e63188880f2e "魔杰西") hello world.

总资产 1 (约 0.09 元) 共写了 5102 字获得 6 个赞共 1 个粉丝

### 被以下专题收入，发现更多相似内容