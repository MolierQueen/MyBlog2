---
title: 自定义AlertView
categories: iOS开发
date: 2018-01-24 12:37:20
tags:
  - UI
  - AlertView
comments:
---
# CustomAlertView
一个自定义的AlertView，用户可以根据自己的需求来设置。
## 使用方法

![初始化方法](https://cdn.cdnjson.com/tvax3.sinaimg.cn/large/006tNc79gy1fo6k50sh34j30hy084abf.jpg)

类似于系统的初始化方法，如果没有值的话就传nil就好，不要传空字符串。最后一个参数传title数组就好了。

![使用](https://cdn.cdnjson.com/tvax3.sinaimg.cn/large/006tNc79gy1fo6k5axspej30r702rmy0.jpg)

然后调用showInViewWithAction方法显示出来
<!--more-->

![显示](https://cdn.cdnjson.com/tvax3.sinaimg.cn/large/006tNc79gy1fo6k5idk8ij30mp041aai.jpg)

最后一个参数是button的点击事件，根据tag值来区分不同的button点击，只有取消button的tag是0，其他的是1.2.3...依次往下排列就好

![可自定义的一些属性](https://cdn.cdnjson.com/tvax3.sinaimg.cn/large/006tNc79gy1fo6k5q4o6nj30kr0ox78u.jpg)

这些属性可以自定义，这里就不细说了，大家可以使试试。

## 样式截图

![样式截图](https://cdn.cdnjson.com/tvax3.sinaimg.cn/large/006tNc79gy1fo6k5zfyi2j30dc0notac.jpg)

![使用截图](https://cdn.cdnjson.com/tvax3.sinaimg.cn/large/006tNc79gy1fo6k66fmogg308o0figvd.gif)

大概就这么多，很简单的有问题随时联系我吧。
