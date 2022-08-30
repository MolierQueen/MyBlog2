---
title: 关于物理效果的动画引擎UIDynamic介绍
categories: iOS开发
date: 2018-01-23 21:18:42
tags:
  - 动画
  - UI
comments:
---
这几天事情超多，实在很难静下心来研究一个东西，但是这个类我也是关注好久了，早就想总结下写出来， 可能这篇文章并不会像之前额那样那么详细，按理说写动画相关的东西应该是配gif的，但是真的是没有心思再去搞那些东西，代码并不难，大家可以照着代码写一下看下效果。
        为了实现动力UI，需要注册一套UI行为的体系，之后UI便会按照预先的设定进行运动了。我们应该了解的新的基本概念有如下四个：
UIDynamicItem：用来描述一个力学物体的状态，其实就是实现了UIDynamicItem委托的对象，或者抽象为有面积有旋转的质点； 简单的说就是一个控件，就是你想往谁上面加动画，这个就是谁。
UIDynamicBehavior：动力行为的描述，用来指定UIDynamicItem应该如何运动，即定义适用的物理规则。一般我们使用这个类的子类对象来对一组UIDynamicItem应该遵守的行为规则进行描述；简单的说就是动画效果，这个类是动画效果的一个父类，它的子类大家可以用运行时的方法输出一下看一下，或者一会看我介绍，一个子类是一个效果，各种效果比如重力啊碰撞啊，链接啊之类的。
UIDynamicAnimator；动画的播放者，动力行为（UIDynamicBehavior）的容器，添加到容器内的行为将发挥作用；
ReferenceView：等同于力学参考系，如果你的初中物理不是语文老师教的话，我想你知道这是啥..只有当想要添加力学的UIView是ReferenceView的子view时，动力UI才发生作用。下面看下我们给一个button加一个重力下坠的动画 使用self.View做参考系来建立动画
<!--more-->
![](https://wx4.sinaimg.cn/large/006tNc79gy1fo6m073lu9j30ft02n74g.jpg)

然后

![](https://wx3.sinaimg.cn/large/006tNc79gy1fo6m0h0bvej307900s0si.jpg)

你可以吧这里航代码写到button的点击事件中，这样你一点就会下坠。很简单吧。
        再看下一个碰撞

![](https://wx3.sinaimg.cn/large/006tNc79gy1fo6m1auc65j30mx02ht92.jpg)

我这里写碰撞动画的时候用了两个button，其实大家可以猜到我是让两个button来碰撞的，碰撞的过程中也是会走代理方法的，开始碰撞啊，碰撞结束啊之类的。最后那句话的意思是吧他的参考系(这里是的self.view)的边界作为碰撞边界，就是说这段代码运行后这两个 这两控件撞到屏幕self.view的边框会发生物理的碰撞反弹效果。想这样(点我开始那个按钮)

![](https://wx2.sinaimg.cn/large/006tNc79gy1fo6m1l2ecwg308r0fl75h.gif)         

除了重力和碰撞，iOS SDK还预先帮我们实现了一些其他的有用的物理行为，它们包括
 UIAttachmentBehavior 描述一个view和一个锚相连接的情况，也可以描述view和view之间的连接。attachment描述的是两个点之间的连接情况，可以通过设置来模拟无形变或者弹性形变的情况（再次希望你还记得这些概念，简单说就是木棒连接和弹簧连接两个物体）。当然，在多个物体间设定多个；UIAttachmentBehavior，就可以模拟多物体连接了..有了这些，似乎可以做个老鹰捉小鸡的游戏了- -…
UISnapBehavior 将UIView通过动画吸附到某个点上。初始化的时候设定一下UISnapBehavior的initWithItem:snapToPoint:就行，因为API非常简单，视觉效果也很棒，估计它是今后非游戏app里会被最常用的效果之一了；
UIPushBehavior 可以为一个UIView施加一个力的作用，这个力可以是持续的，也可以只是一个冲量。当然我们可以指定力的大小，方向和作用点等等信息。
UIDynamicItemBehavior 其实是一个辅助的行为，用来在item层级设定一些参数，比如item的摩擦，阻力，角阻力，弹性密度和可允许的旋转等等

其实流程很简单创建animator  然后创建behivator   设置behivator属性 然后animator addBehivator 。就是这个么流程。写代码要学会举一反三触类旁通。    这篇博客写的比较急，但是总体上来说功能没问题，细节上有什么问题，大家找我一起交流
