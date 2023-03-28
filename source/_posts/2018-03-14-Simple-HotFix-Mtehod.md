---
title: 简单的iOS线上热修复方案
categories: iOS开发
date: 2018-03-14 14:04:46
tags:
      - 热修复
      - JSPath
comments:
---
## 向大佬致敬
总是喜欢把参考资料、致谢等写在文章最前面，毕竟是站在人家的肩膀上，向大佬致敬，写这篇文章的也是参考他的 然后加上一些自己的思考，主要目的还是自己再写一遍Demo和文档，以便加深记忆，也帮助自己更好的理解，有句话说：看懂的东西不一定就是学会了，自己能在不看资料的前提下写出来才算是略知一二。
![](https://cdn.cdnjson.com/tvax3.sinaimg.cn/large/006tNc79gy1fpcbkfx1z9j319e0kkdmp.jpg)

以下是** [原文链接](http://limboy.me/tech/2018/03/04/ios-lightweight-hotfix.html)**有兴趣的还可以看下**[大佬博客](http://limboy.me/)**
<!-- more -->

## 工作原理分析
要实现热修复其实原理就是我们可以动态的修改代码，在方法前、中、后插入自己想要的东西或者代码。其实这个需求并不难，iOS的运行时机制可以满足我们的这个要求，但是如果是已经上架了的APP,已经打成了Ipa包我们该如何修改呢？这里就需要服务端去控制，通过下发不同的内容来达到我们想要的目的，但是这里有一个要求，服务端所下发的内容并不能是任意的，而是要通过下发的内容调起我们App内的RunTime机制然后进行偷梁换柱。满足这个要求的数据格式只有字符串化的JS代码，因为我们知道在iOS中JS代码是可以调用OC的代码。综上所述打到热修复整套流程所需的技术如下：
* Runtime：

  可以在本站搜索Runtime关键字找到Runtime相关资料

* 与服务器交互：

  现在大部分APP都具有于服务端交互的能力，就是我们常说的网络请求AFNetWorking等

* JS与OC交互：

  大家可以参考[这篇文章](https://www.jianshu.com/p/d19689e0ed83)，主要参考方式二，使用JavaScriptCore进行交互

进行了上述操作后每次用户启动，App都会进行如下操作

![](https://cdn.cdnjson.com/tvax3.sinaimg.cn/large/006tNc79gy1fpcce8vzimj30vg0t0wva.jpg)

这样一来如果开发在项目发布出去后发现有Crash那么可以立即通过服务器下发JS代码来制定APp每次执行新方法(新方法的定义也是在下发的JS代码中)，可以避免一些问题。

## 实际使用

#### 第三方

这里用到一个第三方库[Aspects](https://github.com/steipete/Aspects)这个库可以理解为一个iOS中的Runtime库，我们不用写繁琐的代码，直接调用他的接口即可，

```
+ (id<AspectToken>)aspect_hookSelector:(SEL)selector
                           withOptions:(AspectOptions)options
                            usingBlock:(id)block
                                 error:(NSError **)error;
```

其中的枚举就是选择我们要插入方法的位置，其中包含

```
typedef NS_OPTIONS(NSUInteger, AspectOptions) {
    AspectPositionAfter   = 0,            /// Called after the original implementation (default)
    AspectPositionInstead = 1,            /// Will replace the original implementation.
    AspectPositionBefore  = 2,            /// Called before the original implementation.

    AspectOptionAutomaticRemoval = 1 << 3 /// Will remove the hook after the first execution.
};

```
这个库据说是对上线没有影响。

#### 配置工程

用实际代码来证明下，这是我Controller中的一个代码，很明显会产生数组越界的Crash，假如我们在上线后才发现了这个问题，这时候需要修复

```
#import "ViewController.h"

@interface ViewController ()

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    [self crashMethod:0];

    // Do any additional setup after loading the view, typically from a nib.
}



- (void)crashMethod:(NSInteger)argument
{
    if (argument == 0) {
        NSArray * arr = @[@"1"];
        [arr objectAtIndex:2];
    }
}

- (void)didReceiveMemoryWarning {
    [super didReceiveMemoryWarning];
    // Dispose of any resources that can be recreated.
}


@end

```
导入上面说的那个第三方.h和.m 然后自己建立一个桥接类，用来处理JS和O的交互，大概的结构就是这样

![](https://cdn.cdnjson.com/tvax3.sinaimg.cn/large/006tNc79gy1fpcdbh9ahbj307k0duwfd.jpg)

其中交互类中暴露出如下接口

```
#import <Foundation/Foundation.h>
#import "Aspects.h"
#import <objc/runtime.h>
#import <JavaScriptCore/JavaScriptCore.h>
@interface Felix : NSObject

/**
 初始化
 */
+ (void)fixIt;


/**
 开始执行JS代码

 @param javascriptString 需要执行的JS
 */
+ (void)evalString:(NSString *)javascriptString;
@end

```

.m文件的内容可以到大佬博客中参考这里不放出，不然篇幅太长。

#### 开始使用
因为我们最好用能控制代码里面的所有方法，所以我们要尽早的注册交互类，在APpdelegate中如下注册

```
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    // Override point for customization after application launch.


    [Felix fixIt];
    NSString *fixScriptString = @" \
    fixInstanceMethodReplace('ViewController', 'crashMethod:', function(instance, originInvocation, originArguments){ \
    if (originArguments[0] == 0) { \
    console.log('crash！！！！！'); \
    } else { \
    runInvocation(originInvocation); \
    } \
    }); \
    \
    ";
    [Felix evalString:fixScriptString];

//    如果是多个方法建议用循环执行
//    NSArray * hotFixStr = @[fixScriptString];
//    for (int i = 0; i < hotFixStr.count; i ++) {
//        [Felix evalString:hotFixStr[i]];
//    }

    return YES;
}

```
其中JS的代码就是我们所要修改的内容，可以看到当参数为0的时候输出crash！！！然后不再继续执行了。实际项目中这段代码是由服务器动态返回的，如果我们要修改多个方法，就需要服务器返回JS字符串数组我们这边来进行循环处理即可。这时候在运行下代码不会崩溃，下面会输出一个crash！！！！

![](https://cdn.cdnjson.com/tvax3.sinaimg.cn/large/006tNc79gy1fpcdjfkqkmj30i4034t8s.jpg)

## 思考

这个方法相比较之前的JSpatch 是非常轻量级的，而且也只是实现了简单的容错功能，并不能做一些复杂的操作，比如生成一个对象之类的，不过对于一般的控制已经可以满足了，毕竟在苹果爸爸这么严厉的管制下能有这样的方法也还不错啊。

{% aplayerlist %}
{
    "autoplay": true,
    "showlrc": 0,
    "mutex": true,
    "music": [
        {
            "title": "不知归期的故人",
            "author": "房东的猫",
            "url": "https://molier-1256056152.cos.ap-guangzhou.myqcloud.com/%E4%B8%8D%E7%9F%A5%E5%BD%92%E6%9C%9F%E7%9A%84%E6%95%85%E4%BA%BA.mp3",
            "pic": "https://y.gtimg.cn/music/photo_new/T002R300x300M000004NFJ230yX0Nz.jpg?max_age=2592000",
            "lrc": "https://demo.meting.api.meto.moe/action/metingapi?server=tencent&type=lrc&id=000zreoj2VtcID"
        }
    ]
}
{% endaplayerlist %}
