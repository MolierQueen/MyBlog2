---
title: NSURLProtocol对WKWebView的处理
categories: iOS开发
date: 2018-01-24 12:24:06
tags:
  - webview
  - hook
comments:
---
之前写过一篇[文章](http://www.jianshu.com/p/cd4c1bf1fd5f)是关于基于NSURLProtocol做的DNS解析，其中对NSURLProtocol也有了简单的介绍，我们都知道他可以拦截所有基于URL Loading System 中的请求，但是对于WKWebview里面所发出的请求即使他是http/https 也无能为力，先来简单的了解下WKWebView.
<!--more-->
##### WKWebview
  iOS8以后，苹果推出了新框架Webkit，提供了替换UIWebView的组件WKWebView。各种UIWebView的问题没有了，速度更快了，占用内存少了，一句话，WKWebView是App内部加载网页的最佳选择！我们做开发最关系的是内存问题，基本上网上所有的资料都在说WKWebview的内存占用会更少，但是到底少了多少我这边做了下测试，同样是加载163的首页

![使用UIWebView的内存](https://cdn.cdnjson.com/tvax3.sinaimg.cn/large/006tNc79gy1fo6k80vn82j3087036wem.jpg)
![使用WKWebview的内存](https://cdn.cdnjson.com/tvax3.sinaimg.cn/large/006tNc79gy1fo6k8agxuoj308002smx3.jpg)

从上图看出内存大概能优化百分之八十左右，而且从网页的滑动上也确实有所改善。这么明显的性能提升但是苹果并没有完全放弃UIWebView也一定有他的道理，就拿本文要讲的NSURLProtocol拦截请求来说，WKWebview的兼容并不UIWebView好，还需要开发者做一些操作。
##### WebKit源码分析
  由于WKWebview是基于webkit内核来做的，所以我们在使用的时候需要导入一个这样的东西。
    #import <WebKit/WebKit.h>
通过这个我们可以猜到WKWebview中所有的请求以及一些逻辑肯定走的都是webkit里面的东西，所以他对于网页的加载之之类的操作也不会走系统本省的URL Loading System，这么说来他的请求不能被NSURLProtocol拦截也是理所当然的了。不过WKWebview是否真的和NSURLProtocol一点关系都没有还需要去研究，幸好webkit是开源的，github上很容易找到[源码](https://github.com/WebKit/webkit)（大小大概是1G多点的zip，花了我将近一天时间来看）。拉下代码直接搜索NSURLProtocol，看看有没有有关的信息
![搜索结果](https://cdn.cdnjson.com/tvax3.sinaimg.cn/large/006tNc79gy1fo6k8kigkxj30i80t0tg8.jpg)

看来的确是有和NSURLProtocol有关系，后面通过断点的调用栈中也找到了
    + [NSURLProtocol canInitWithRequest:]
这样的字样，再通过网上查一些资料也证实了我的猜想，其实WKWebview在一开始时候是会调用到NSURLProtocol中的入口方法canInitWithRequest的，但是就没有然后了，也就是说WKWebview是和NSURLProtocol有一定关联，只是在NSURLProtocol的入口处返回NO所以导致NSURLProtocol不接管WKWebview的请求。我们点进webkit源码中的CustomProtocol可以看到，整体的结构我们都差不多，但是我注意到每个CustomProtocol的入口函数都有这样一个判断：
![入口函数1](https://cdn.cdnjson.com/tvax3.sinaimg.cn/large/006tNc79gy1fo6k8wgu6tj30yg04x40p.jpg)

![入口函数2](https://cdn.cdnjson.com/tvax3.sinaimg.cn/large/006tNc79gy1fo6k93edfij30rw05ijsb.jpg)
(粉色的可以暂时认定为是它内部的一个custom字符串)通过这个可以猜想，WKWebview并不是不走NSURLProtocol，而是需要满足他的一个规则，他才会在入口函数这里返回YES来给你放行，这个规则便是你所请求的URL的Scheme要和它内部配置的CustomScheme相同。不过这里有一个疑问，苹果在使用webkit时候为什么会把http/https这样大众化的scheme过滤掉，看来他是不建议开发者来使用NSURLProtocol。接下来我们来看这个CustomScheme，既然苹果内部规定好的那么一定能通过某种方式来注册一个自己的scheme，实在不行就hook嘛。通过翻他的源码发现最终都指向一句代码    

    [WKBrowsingContextController registerSchemeForCustomProtocol:testScheme];
方法实现为
    + (void)registerSchemeForCustomProtocol:(NSString *)scheme
    {
        WebProcessPool::registerGlobalURLSchemeAsHavingCustomProtocolHandlers(scheme);
    }
```
void WebProcessPool::registerGlobalURLSchemeAsHavingCustomProtocolHandlers(const String& urlScheme)
{
    if (!urlScheme)
        return;

    globalURLSchemesWithCustomProtocolHandlers().add(urlScheme);
    for (auto* processPool : allProcessPools())
        processPool->registerSchemeForCustomProtocol(urlScheme);
}
```
通过方法名字可以看出这个就是那个向webkit注册CustomScheme的方法，只要我们在注册完我们自己的CustomProtocol之后在调用该方法应该就可以了。通过他的源码也进一步印证了我的猜想(他也是这么写的)
![webkit源码](https://cdn.cdnjson.com/tvax3.sinaimg.cn/large/006tNc79gy1fo6k9nobq7j30yg04tmzt.jpg)
##### 具体实施
  找到了方法就要去实施，不过因为registerSchemeForCustomProtocol是WKBrowsingContextController的类方法，所以只能用WKBrowsingContextController去调用，但是在webkit的头文件发现WKBrowsingContextController并没有开放出来，所以我们采用NSClassFromString和NSSelectorFromString方法来拿到类和对应的方法，整体代码如下
```
    //注册自己的protocol
    [NSURLProtocol registerClass:[CustomProtocol class]];

    //创建WKWebview
    WKWebViewConfiguration * config = [[WKWebViewConfiguration alloc] init];
    WKWebView * wkWebView = [[WKWebView alloc] initWithFrame:CGRectMake(0, 0, [UIScreen mainScreen].bounds.size.width, [UIScreen mainScreen].bounds.size.height) configuration:config];
    [wkWebView loadRequest:webViewReq];
    [self.view addSubview:wkWebView];

    //注册scheme
    Class cls = NSClassFromString(@"WKBrowsingContextController");
    SEL sel = NSSelectorFromString(@"registerSchemeForCustomProtocol:");
    if ([cls respondsToSelector:sel]) {
        // 通过http和https的请求，同理可通过其他的Scheme 但是要满足ULR Loading System
        [cls performSelector:sel withObject:@"http"];
        [cls performSelector:sel withObject:@"https"];
    }
```
  实现效果。我将网页中所有的图片替换成了柴犬图片

![效果](https://cdn.cdnjson.com/tvax3.sinaimg.cn/large/006tNc79gy1fo6ka7rv18g307o0ds7wl.gif)

##### 值得注意
   * 关于私有API

  因为WKBrowsingContextController和registerSchemeForCustomProtocol应该是私有的所以使用时候需要对字符串做下处理，用加密的方式或者其他就可以了，实测可以过审核的。

* 关于post请求
大家会发现拦截不了post请求(拦截到的post请求body体为空)，这个其实和WKWebview没有关系，这个是苹果为了提高效率加快流畅度所以在NSURLProtocol拦截之后索性就不复制body体内的东西，因为body的大小没有限制，开发者可能会把很大的数据放进去那就不好办了。我们可以采取httpbodystream的方式拿到body，这个在之前的[文章](http://www.jianshu.com/p/cd4c1bf1fd5f)也有提过

```  
#pragma mark -
#pragma mark 处理POST请求相关POST  用HTTPBodyStream来处理BODY体
- (NSMutableURLRequest *)handlePostRequestBodyWithRequest:(NSMutableURLRequest *)request {
    NSMutableURLRequest * req = [request mutableCopy];
    if ([request.HTTPMethod isEqualToString:@"POST"]) {
        if (!request.HTTPBody) {
            uint8_t d[1024] = {0};
            NSInputStream *stream = request.HTTPBodyStream;
            NSMutableData *data = [[NSMutableData alloc] init];
            [stream open];
            while ([stream hasBytesAvailable]) {
                NSInteger len = [stream read:d maxLength:1024];
                if (len > 0 && stream.streamError == nil) {
                    [data appendBytes:(void *)d length:len];
                }
            }
            req.HTTPBody = [data copy];
            [stream close];
        }
    }
    return req;
}
```
