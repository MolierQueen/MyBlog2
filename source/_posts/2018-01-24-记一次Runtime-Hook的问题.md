---
title: 记一次Runtime Hook的问题
categories: iOS开发
date: 2018-01-24 11:35:57
tags:
  - 底层
  - runtime
  - Hook
comments:
---
#### 背景
项目中遇到一个问题，需要引入两个SDK，我们暂且命名为A 和 B，由于业务需要这两个SDK都需要对一个系统函数C进行hook, 但是有一个前提，由于B 所做的是一个统计相关的SDK，所以B要监控App内的所有代码这其中也包括了 SDK A 所做的一些操作，所以我们必须确保B在hook C函数时候  A已经对C函数hook完毕，其实这就涉及到hook顺序的问题。
<!--more-->
#### 研究
先看下代码，我用hookMethod来模仿系统方法。
```
- (void) TEST_HOOK_TWICE {
    [self changeOrginalSelectorName:@"hookedMethod" inClass:@"RootViewController" withCustomSelectorName:@"swizzle_hookedMethod1" isClassMethod:NO];

    [self changeOrginalSelectorName:@"hookedMethod" inClass:@"RootViewController" withCustomSelectorName:@"swizzle_hookedMethod2" isClassMethod:NO];

    [self hookedMethod];

}

- (void)hookedMethod {
    NSLog(@"原始方法");
}

- (void)swizzle_hookedMethod1 {
    NSLog(@"1");
    [self swizzle_hookedMethod1];
}

- (void)swizzle_hookedMethod2 {
    NSLog(@"2");
    [self swizzle_hookedMethod2];
}
```
然后看下没有hook之前的样子


![原本的样子](https://wx2.sinaimg.cn/large/006tNc79gy1fo6ll1zt21j30j80jk3zc.jpg)


然后我们执行代码
```
//第一步：交换A中的方法和系统方法
 [self changeOrginalSelectorName:@"hookedMethod" inClass:@"RootViewController" withCustomSelectorName:@"swizzle_hookedMethod1" isClassMethod:NO];
//第二步：交换B中的方法和系统方法
[self changeOrginalSelectorName:@"hookedMethod" inClass:@"RootViewController" withCustomSelectorName:@"swizzle_hookedMethod2" isClassMethod:NO];
//第三步：调用系统方法
[self hookedMethod];
```
然后我们一步一步来看，先看调用第一步之后是什么样子的(红色箭头为第一步之后的样子)

![第一步之后](https://wx3.sinaimg.cn/large/006tNc79gy1fo6llhzf0uj30ki0k2q3z.jpg)

然后看第二步调用完之后的样子(绿色是第二步调用)

![第二部之后的样子](https://wx2.sinaimg.cn/large/006tNc79gy1fo6lm7mfshj30jo0iu75m.jpg)

接下来我们调用系统方法也就是第三步，然后我们看下流程是怎样的(每个方法实现里面都会递归调用下自身，为了是hook时候不改变原有逻辑)

![调用顺序](https://wx3.sinaimg.cn/large/006tNc79gy1fo6lmi43vmj30yg03v756.jpg)

这样一来就很明显 如果想想监控住所有的代码那就需要在A IMP 这步，因为之前的Hook顺序是先A -> B -> System 这样一来只要我们改一下顺序改为 B -> A -> System就可以让B SDK监控到所有的代码。

![调用顺序](https://wx4.sinaimg.cn/large/006tNc79gy1fo6ln0wmlyj30yg07ign0.jpg)
