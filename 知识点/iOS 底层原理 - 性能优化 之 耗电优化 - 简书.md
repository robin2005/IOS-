\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[www.jianshu.com\](https://www.jianshu.com/p/f4493d08ba8e)

[![](https://upload.jianshu.io/users/upload_avatars/1319505/9754fac0eb59?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96/format/webp)](https://www.jianshu.com/u/40dce98b3416)

2020.01.10 17:39:26 字数 944 阅读 27

#### 面试题引发的思考：

**Q: 项目优化从哪几方面着手？**

*   **耗电优化、启动优化、卡顿优化、APP 瘦身。**

**Q: 耗电优化的几个方面？**

*   **CPU 优化：**  
    **降低 CPU、GPU 功耗、少用定时器、优化 I/O 操作；**
*   **网络优化：**  
    **压缩网络数据、缓存、断点续传、无网超时操作、批量传输；**
*   **定位优化：**  
    **定位类型选择、实时更新设置、定位精度选择；**
*   **图像优化：**  
    **控制视图层数、图片数量、图片大小，减少透明视图，避免离屏渲染；**
*   **硬件检测优化：**  
    **不需要时，及时关闭加速度计、陀螺仪、磁力计等。**

* * *

### 1\. 耗电主要来源

![](http://upload-images.jianshu.io/upload_images/1319505-ab7e1747ce26023e.png)

耗电主要来源

耗电的主要来源如下：

> *   **CPU 处理 (Processing)**
> *   **网络 (Networking)**
> *   **定位 (Location)**
> *   **图像 (Graphics)**
> *   **硬件 (Hardware)**

* * *

### 2\. 耗电优化

#### (1) CPU 优化

*   **尽可能降低 CPU、GPU 功耗**；
*   **少用定时器**；
*   **优化 I/O(Input/Output) 操作**：
    *   尽量不要频繁写入小数据，最好 **批量一次性写入**；
    *   读写大量重要数据时，考虑用 **`dispatch_io`** ，其提供了基于 GCD 的异步操作文件 I/O 的 API；**`dispatch_io`**系统会优化磁盘访问；
    *   数据量比较大的，使用 **数据库** (比如 SQLite、CoreData)。

#### (2) 网络优化

*   **减少、压缩网络数据** (XML 体积比较大，json 体积比较小，google 的 protocol buffer) (上传压缩文件，服务器收到文件后再解压)；
*   如果多次请求的结果是相同的，尽量使用 **缓存**；
*   使用 **断点续传**，否则网络不稳定时可能多次传输相同的内容；
*   网络不可用时，不要尝试执行网络请求；
*   让用户可以取消长时间运行或者速度很慢的网络操作，设置合适的 **超时时间**；
*   **批量传输**，比如：下载视频流时，不要传输很小的数据包，直接下载整个文件或者一大块一大块地下载。

#### (3) 定位优化

*   如果只是需要快速确定用户位置，最好用**`CLLocationManager`的`requestLocation`方法**。定位完成后，会自动让 **定位硬件断电**；
*   如果不是导航应用，尽量 **不要实时更新位置**，定位完毕就关掉定位服务；
*   **尽量降低定位精度**，比如尽量不要使用精度最高的`kCLLocationAccuracyBest`；
*   需要后台定位时，尽量设置**`pausesLocationUpdatesAutomatically = YES`**，如果用户不太可能移动的时候系统会自动暂停位置更新；
*   尽量不要使用`startMonitoringSignificantLocationChanges`，优先考虑**`startMonitoringForRegion:`**。

#### (4) 图像优化

*   **视图层数控制**：尽量减少视图数量和层次；
*   **控制图片大小**：GPU 能处理的最大纹理尺寸是 4096x4096，一旦超过这个尺寸，就会占用 CPU 资源进行处理；
*   **减少图片数量**：尽量避免段时间内大量图片的显示，尽可能将多张图片合成一张图片显示；
*   **减少透明的视图**：少用透明图（`alpha<1`），不透明的就设置`opaque`为`YES`；
*   **尽量避免出现离屏渲染**：光栅化、切角、阴影、遮罩。

#### (5) 硬件检测优化

*   用户移动、摇晃、倾斜设备时，会产生 **动作 (motion) 事件**，这些事件由 **加速度计、陀螺仪、磁力计** 等硬件检测。
*   在不需要检测的场合，应该及时关闭这些硬件。

"小礼物走一走，来简书关注我"

还没有人赞赏，支持一下

[![](https://upload.jianshu.io/users/upload_avatars/1319505/9754fac0eb59?imageMogr2/auto-orient/strip|imageView2/1/w/100/h/100/format/webp)](https://www.jianshu.com/u/40dce98b3416)

总资产 1 (约 0.16 元) 共写了 4.8W 字获得 24 个赞共 23 个粉丝

### 被以下专题收入，发现更多相似内容

### 推荐阅读[更多精彩内容](https://www.jianshu.com/)

*   探索底层原理，积累从点滴做起 往期回顾 iOS 底层原理探索 — OC 对象的本质 iOS 底层原理探索 — class...
    
    [![](https://upload-images.jianshu.io/upload_images/1760191-9747305ab3b475d4.png?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)](https://www.jianshu.com/p/824930a58c1a)
*   亲爱的乖乖： 今天看了周洋阿姨在朋友圈分享的一个特别富有深意的短片《Alike》，妈妈也推荐给你，我们一起看了两遍...
    
    [![](https://upload-images.jianshu.io/upload_images/1316344-84e05cb50681e731.jpg?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)](https://www.jianshu.com/p/4c3e4e186488)
*   之前有一档新的综艺节目仅仅播出了一个先导篇就已经取得了不错的播放量，可见这档综艺节目要是开播的话岂不是能取得不错的...
    
    [![](https://upload-images.jianshu.io/upload_images/13389957-c376c0b575ab1adb.jpg?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)](https://www.jianshu.com/p/7891c17806fe)
*   5087 - 元姐姐 2018 年，我参加了剽悍读书营。在这里，我收获了一些读书营特有的标签——复盘达人、复盘小能手、坚...
    
    [![](https://upload.jianshu.io/users/upload_avatars/10622584/8d54892e-0e9c-4a1e-8295-45d4a6140e08?imageMogr2/auto-orient/strip|imageView2/1/w/48/h/48/format/webp)元姐姐](https://www.jianshu.com/u/d102c77c5cb7)阅读 6
    
    [![](https://upload-images.jianshu.io/upload_images/9682207-232529822eaa2a73.jpg?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)](https://www.jianshu.com/p/676fa508e8f8)