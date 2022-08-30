---
title: 关于iPhone X下Home键的隐藏和延迟响应
categories: iOS开发
date: 2018-02-08 10:15:57
tags:
 - UI
 - 屏幕适配
 - iOS 11
comments:
---

# iOS 11通用相关

## Edge Protect

iPhone X 刚出来的时候苹果第一时间更新了新设备的交互文档，其中针对了大家最关心的“系统手势和App自带手势冲突”的问题也给出了相应的解决办法:

![](https://wx3.sinaimg.cn/large/006tNc79gy1fo8u9uqjehj31kw0t7h2q.jpg)

虽然苹果用黑体字写着强烈不建议开发者干涉系统的手势，但是为了增强用户体验还是开出了接口，苹果管这个叫做 "edge protect" 因为进入App后系统手势都是从边缘触发，引起冲突的地方也会是在边缘中。

<!--more-->

根据官方文档描述，在冲突区域第一次执行手势的时候会优先触发App的内部手势，当短时间内再次进行同样的操作则会触发系统手势。也就是将系统手势延迟到下一次执行。

## API Discussion

根据官方文档找到对应的API

```
// Override to return a child view controller or nil. If non-nil, that view controller's screen edges deferring system gestures will be used. If nil, self is used. Whenever the return value changes, -setNeedsScreenEdgesDeferringSystemGesturesUpdate should be called.
- (nullable UIViewController *)childViewControllerForScreenEdgesDeferringSystemGestures API_AVAILABLE(ios(11.0)) API_UNAVAILABLE(watchos, tvos);

// Controls the application's preferred screen edges deferring system gestures when this view controller is shown. Default is UIRectEdgeNone.
- (UIRectEdge)preferredScreenEdgesDeferringSystemGestures API_AVAILABLE(ios(11.0)) API_UNAVAILABLE(watchos, tvos);

// This should be called whenever the return values for the view controller's screen edges deferring system gestures have changed.
- (void)setNeedsUpdateOfScreenEdgesDeferringSystemGestures API_AVAILABLE(ios(11.0)) API_UNAVAILABLE(watchos, tvos);
```
#### childViewControllerForScreenEdgesDeferringSystemGestures

该方法是用来控制子试图控制器是否允许开发者控制edge protect的开启或是关闭。如果实现了这个方法并且返回值不为空那么子VC的edge protect设置就会遵循父VC的设置，跟随父VC是否延迟执行系统手势。

![](https://wx4.sinaimg.cn/large/006tNc79gy1fo917ycqy6j319i0bqjtr.jpg)

#### preferredScreenEdgesDeferringSystemGestures

该方法是设置edge protect的方法，返回值是一个边界的枚举

```
typedef NS_OPTIONS(NSUInteger, UIRectEdge) {
    UIRectEdgeNone   = 0,
    UIRectEdgeTop    = 1 << 0,
    UIRectEdgeLeft   = 1 << 1,
    UIRectEdgeBottom = 1 << 2,
    UIRectEdgeRight  = 1 << 3,
    UIRectEdgeAll    = UIRectEdgeTop | UIRectEdgeLeft | UIRectEdgeBottom | UIRectEdgeRight
} NS_ENUM_AVAILABLE_IOS(7_0);
```

因为不论我们从shang、左、下、右边都可触发系统手势，所以方法保护了四个边框，将边界触发的手势延迟执行，这个方法从iOS11开始使用，不过枚举中虽然有左右的边界保护，但是系统手势中还不清楚左右滑动会触发什么效果，实验发现对于VC的左边界右滑动pop手势是无效的，也就是说这个pop手势一直有着最高的优先级。不过上下就很好理解，底部上拉出控制中心，顶部下拉是通知中心。

* 无限制

  当不做任何限制时候在顶部和底部很容易触发到系统的手势，他们会优先于Tab.eView的scroll手势执行，虽说屏幕大部分的界面还是执行TableView手势的，但是当用户误触到边界的时候还是会稍稍影响体验，尤其是在全屏模式下、相机、视频、游戏等

  ![](https://wx2.sinaimg.cn/large/006tNc79gy1fo909u85fpg308k0goe8d.gif)


* Edge Protent

  在对应的ViewControll中添加如下代码，我们这边开启的是所有边界限制其中包括了上、下边界。在下拉或者上拉的话会先触发App内部手势，同时出现一个小箭头然后在箭头消失之前再次滑动就会触发系统手势。
```
-(UIRectEdge)preferredScreenEdgesDeferringSystemGestures
{
    return UIRectEdgeAll;
}
```

  ![](https://wx4.sinaimg.cn/large/006tNc79gy1fo90rfo0dig308k0go7wu.gif)

#### setNeedsUpdateOfScreenEdgesDeferringSystemGestures

这个方法是在应用内部动态控制edge protect，我们可以在上个方法中返回一个BOOL变量，然后根据需要改变该变量的值，然后调用该方法进行刷新。

![](https://wx1.sinaimg.cn/large/006tNc79gy1fo912yvhsag308k0goe8h.gif)


# iPhone X使用相关

iPhone X在系统手势上面交互和其他设备还是有一定区别的，因为加入了Home Indicator的原因，引入了新的手势，同时对以往的手势也做了相应的调整。

## iPhone X Edge Protect

在iPhone X 中通知中心和控制中心全部都移动到了由顶部刘海处下拉和右上角下拉来触发。原本底部的所有手势都被Home Indicator占用。其实Edge Protect在这里依然适用，只是对于Home Indicator的手势有一个小插曲。正常来说他在底部，就应该受到UIRectEdgeBottom 或者是 UIRectEdgeAll控制，但是一开始苹果并没有这么做，不论怎么写代码，他都有着最高的优先级，在iPhone X刚发布我就试图去处理交互问题，因为海报工厂并没有传统的UITabBarController，且里面所有的tableView都是直通到底，但是始终都无法延迟执行与Home Indicator相关的任何手势。

![](https://wx1.sinaimg.cn/large/006tNc79gy1fo920zedwnj308w0j9gxa.jpg)

后来看了其他游戏，视频类App在iPhone X上的表现也都是如此。腾讯的王者荣耀，网易的吃鸡都是一样。腾讯官方给出的解释是暂时开起引导式访问，也仍然不方便。后来在今年1月25日苹果推送了iOS 11.2.5的版本更新，然后王者荣耀也跟着进行了一波更新，在进入游戏时候就会发现，底部的Home Indicator当你一段时间不去触碰它的时候由黑色或者白色(根据当前的屏幕显示的内容来决定)变成非常透明的灰色，当你第一次进行操作会默认执行App内手势，同时激活Home Indicator，短时间内进行第二次操作就可以返回桌面

![](https://wx2.sinaimg.cn/large/006tNc79gy1fo935pn8dsg30go07s1l2.gif)

一开以为是有新的API出现，不过看了交互文档并没有新的东西，而且小版本的系统更新应该也不会出现新的东西。所以找到了之前的edge protect 代码运行后确实可以达到效果。对于视频，游戏等App，确实可以起到很好的防误触的效果。遗憾的是并没有太多的人使用这个功能。目前主流的大型游戏，包括Gameloft出品的游戏都没做相应的处理。

![](https://wx1.sinaimg.cn/large/006tNc79gy1fo943bx7sog308h0gox6z.gif)

## iPhone X Home Indicator Hidden

如果说上面的Edge Protect适合在游戏中使用，那么Home Indicator Hidden则更适合在非游戏环境下增强App的沉浸感，尤其是全屏视屏播放、录制的时候。同样三个API，和Edge protect的用法完全一样。

```
// Override to return a child view controller or nil. If non-nil, that view controller's home indicator auto-hiding will be used. If nil, self is used. Whenever the return value changes, -setNeedsHomeIndicatorAutoHiddenUpdate should be called.
- (nullable UIViewController *)childViewControllerForHomeIndicatorAutoHidden API_AVAILABLE(ios(11.0)) API_UNAVAILABLE(watchos, tvos);

// Controls the application's preferred home indicator auto-hiding when this view controller is shown.
- (BOOL)prefersHomeIndicatorAutoHidden API_AVAILABLE(ios(11.0)) API_UNAVAILABLE(watchos, tvos);

// This should be called whenever the return values for the view controller's home indicator auto-hiding have changed.
- (void)setNeedsUpdateOfHomeIndicatorAutoHidden API_AVAILABLE(ios(11.0)) API_UNAVAILABLE(watchos, tvos);
```
上面写的是自动隐藏，也就是说系统会根据当时的使用情况来进行显示或者隐藏，而不是永久的隐藏掉，实际测试发当界面两秒内没有进行任何交互操作的时候Home Indicator会逐渐隐去，直达屏幕上出现了点击的操作，注意是点击，TableView的滑动并不能触发显示，不过只是是隐藏，但是手势依然可以使用。

![](https://wx2.sinaimg.cn/large/006tNc79gy1fo94hujx35g308h0go4qy.gif)

如果是feed流界面搭配酷一点的UI就会提高沉浸感，比如这样：

![](https://wx3.sinaimg.cn/large/006tNc79gy1fo94p0bmoqg308h0gox6t.gif)

有的人可能会问如果说点击的手势会触发它再次显示那我获取window上的交互每次在它即将显示的时候通过**setNeedsUpdateOfHomeIndicatorAutoHidden**在让他隐藏不就好了吗？这样一来既不影响系统手势也不会让它在显示出来，其实我自己试过不行的，毕竟苹果不会让你这样改。

## 坑点

需要注意的是：prefersHomeIndicatorAutoHidden和preferredScreenEdgesDeferringSystemGestures不可一起使用，如果一起使用的话后者是不生效的。
