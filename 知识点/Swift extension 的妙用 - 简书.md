\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[www.jianshu.com\](https://www.jianshu.com/p/01e83d56d773)

[![](https://upload.jianshu.io/users/upload_avatars/6180564/c48bf605-b3fb-4008-a78b-3000628d8d88?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96/format/webp)](https://www.jianshu.com/u/475a09787a4d)

0.112020.08.05 09:29:17 字数 174 阅读 80

### 1\. 简单介绍 extension

在 Swift 中扩展可以:

*   添加计算实例属性和计算类型属性；
*   定义实例方法和类型方法；
*   提供新构造器；
*   定义下标；
*   定义和使用新的嵌套类型；
*   使现有类型符合协议；

> 注意：  
> 1\. 扩展可以向类型添加新功能，但不能覆盖现有功能。2. 扩展可以添加新的计算属性，但它们不能添加存储属性，或向现有属性添加属性观察者。

#### 语法

### 应用

#### 1\. 字体拓展

先看一下拓展后的🌰：

```
let label = UILabel()


label.font = UIFont.pfLight(ofSize: 13.0)


label.font = UIFont.pfRegular(10.0)


label.font = UIFont.pfMedium(12.0)


label.font = UIFont.pfSemibold(ofSize: 13.0)



```

### 具体实现的代码

```
extension UIFont {
    
    enum FontName {
        
        case pingfang, pingfangLight, pingfangMedium, pingfangSemibold
        var name: String {
            switch self {
            case .pingfang:
                return "PingFangSC-Regular"
            case .pingfangLight:
                return "PingFangSC-Light"
            case .pingfangMedium:
                return "PingFangSC-Medium"
            case .pingfangSemibold:
                return "PingFangSC-Semibold"
            }
        }
    }
    
    class func pfRegular(\_ fontSize: CGFloat) -> UIFont {
        let font = UIFont.font(type: .pingfang, fontSize: fontSize)
        return font
    }
    
    class func pfLight(\_ fontSize: CGFloat) -> UIFont {
        let font = UIFont.font(type: .pingfangLight, fontSize: fontSize)
        return font
    }
    
    class func pfMedium(\_ fontSize: CGFloat) -> UIFont {
        let font = UIFont.font(type: .pingfangMedium, fontSize: fontSize)
        return font
    }
    
    class func pfSemibold(\_ fontSize: CGFloat) -> UIFont {
        let font = UIFont.font(type: .pingfangSemibold, fontSize: fontSize)
        return font
    }
    
    class func font(type: FontName, fontSize size: CGFloat) -> UIFont {
        guard let font = UIFont(name: type.name, size: size) else {
            return .systemFont(ofSize: size)
        }
        return font
    }
}



```

### 2\. 图片拓展

```
extension UIImage {
    
    @objc
    public static func from(color: UIColor) -> UIImage? {
        
        let rect = CGRect(x: 0, y: 0, width: 1, height: 1)
        
        UIGraphicsBeginImageContext(rect.size)
        
        guard let context = UIGraphicsGetCurrentContext() else { return nil }
        
        context.setFillColor(color.cgColor)
        
        context.fill(rect)
        
        let image = UIGraphicsGetImageFromCurrentImageContext()
        
        UIGraphicsEndImageContext()
        
        return image
    }
}


```

"小礼物走一走，来简书关注我"

还没有人赞赏，支持一下

[![](https://upload.jianshu.io/users/upload_avatars/6180564/c48bf605-b3fb-4008-a78b-3000628d8d88?imageMogr2/auto-orient/strip|imageView2/1/w/100/h/100/format/webp)](https://www.jianshu.com/u/475a09787a4d)

总资产 1 (约 0.11 元) 共写了 6598 字获得 3 个赞共 2 个粉丝

### 被以下专题收入，发现更多相似内容