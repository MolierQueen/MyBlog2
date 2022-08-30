---
title: CFNetwork的介绍和使用
categories: iOS开发
date: 2018-01-23 21:04:15
tags:
  - 网络
  - 底层
comments:
---
### CFNetwork背景简介
CFNetwork是ISO中一个比较底层的网络框架，C语言编写，可以控制一些更底层的东西，如各种常用网络协议、socket通讯等，我们通常使用的NSURL则更倾向于API数据请求等，虽然框架也提供了一些操作，但是远不如CFNetwork丰富。CFNetwork已经接近于UNIX系统的socket通信了，使用CFHttpMessageRef进行HTTP连接的好处就是控制的粒度更细了，例如你可以设置SSL连接的PeerName，证书验证的方式，还可以控制每个响应包的接收。不过CFNetwork本质上还是应用层上的封装的通用API。使用者可以不用关心底层协议的实际细节。下图是CFNetwork在iOS系统中的位置(图片来源于官方文档)。

<!--more-->
![image.png](https://wx2.sinaimg.cn/large/006tNc79gy1fo6mit8tsij30bn06lwen.jpg)

由上图可以看出目前iOS的网络编程分四层：
* WebKit：属于Cocoa层，苹果很多地方用到的页面渲染引擎WKWebview；

* NSURL：也属于Cocoa层，对各类URL请求的封装(NSURLRequest)；

* CFNetwork：属于Core Foundation层，基于C的封装，同样的还有CFNetServices(write/readstream)；

* BSD sockets：属于OS层，也是基于C的封装；

### CFNetwork结构

![image.png](https://wx2.sinaimg.cn/large/006tNc79gy1fo6mlt3gwcj30od0a474w.jpg)
上图也是官方文档的图片，描述了CFNetwork的结构，下面逐一讲解。

##### CFSocket API

Socket是网络通讯的底层基础，两个socket端口可以互发数据。我们通常使用的是BSD socket，CFSocket则是BSD socket的抽象，基本上实现了几乎所有BSD socket的功能，并且还融入了run loop。

##### CFStream API
CFStream API提供了数据读写的方法，即读写流，使用它可以为内存、文件、网络（使用socket）的数据建立stream，我们进行网络请求就是对数据的读写，CFStream提供API对两种CFType对象提供抽象：CFReadStream and CFWriteStream。它同时也是CFHTTP和CFFTP的基础。stream有一个很重要的特性就是一旦数据流被提供或者被消耗，就不能从流中重新取出。比如这样

```
uint8_t d[1024] = {0};
//循环条件：流中是否有可用数据(被读过的数据不可用了)
while ([self.inputStream hasBytesAvailable]) {
    //读取相应长度的数据数据
    NSInteger len = [self.inputStream read:d maxLength:1024];
    //如果读取到数据，便将数据快拼接
    if (len > 0 && !self.inputStream.streamError) {
        [data appendBytes:(void *)d length:len];
    } else {
        break;
    }
}
```

##### CFFTP API
对用FTP协议通信的封装，能下载、上传文件和目录到FTP服务器。CFFTP建立的连接可以是同步或者异步，此次不做详解。

##### CFHTTP API
是HTTP协议的抽象，主要对象是CFHTTPMessageRef(类似于我们通常的NSURLRequest)我们需要像构建NSURLRequest那样来构建CFHTTPMessageRef，同样包含一下几个元素
* 必须元素

   * 请求方法 (类型为CFStringRef)：POST、GET、DELETE等..

   * 请求的URL地址 (类型为CFURLRef)：https://www.baidu.com

   * 请求的HTTP版本(类型为CFStringRef)：通常使用kCFHTTPVersion1_1

   * kCFAllocatorDefault：用于创建消息引用的指定默认的系统内存分配器。

* 可选参数
   * body体(类型为CFDataRef)

```

CFHTTPMessageSetBody(CFHTTPMessageRef message, CFDataRef bodyData) CF_AVAILABLE(10_1, 2_0);
```
   * 消息头部，如User-Agent等；

```
CFHTTPMessageSetHeaderFieldValue(CFHTTPMessageRef message, CFStringRef headerField, CFStringRef __nullable value) CF_AVAILABLE(10_1, 2_0);

```
### CFNetwork请求过程
##### 1：构造并创建CFHTTPMessageRef对象
```
//构造的方式上一步已讲
CFHTTPMessageCreateRequest(CFAllocatorRef __nullable alloc, CFStringRef requestMethod, CFURLRef url, CFStringRef httpVersion) CF_AVAILABLE(10_1, 2_0);
```
##### 2：使用CFHTTPMessageRef对象创建输入流
```
//第一个参数传默认
CFReadStreamCreateForHTTPRequest(CFAllocatorRef __nullable alloc, CFHTTPMessageRef request) CF_DEPRECATED(10_2, 10_11, 2_0, 9_0, "Use NSURLSession API for http requests");
```
##### 3：适配SNI环境（一个 IP 地址上可以为不同域名分配使用不同的 SSL 证书；这同时意味着，共享 IP 的虚拟主机也可实现 SSL/TLS 连接。）
因为配置sni环境的所有配置都是基于输入流来操作，所以我们构建完成输入流之后来处理sni，像这样
```
[self.inputStream setProperty:NSStreamSocketSecurityLevelNegotiatedSSL forKey:NSStreamSocketSecurityLevelKey];
//请求的URL的Host
NSDictionary *sslProperties = @{ (__bridge id) kCFStreamSSLPeerName : host };
[self.inputStream setProperty:sslProperties forKey:(__bridge_transfer NSString *) kCFStreamPropertySSLSettings];
```
##### 4：打开输入流
打开输入流分为两步
* 设置代理：[self.inputStream setDelegate:weakSelf]
* 加入当前的runloop：        
```
[_inputStream removeFromRunLoop:self.runloop forMode:[self runloopMode]];
```
* 调用Open方法

##### 5：收到代理数据回调
```
- (void)stream:(NSStream *)aStream handleEvent:(NSStreamEvent)eventCode;
```
其中分为几个状态
```
typedef NS_OPTIONS(NSUInteger, NSStreamEvent) {
    NSStreamEventNone = 0,
    NSStreamEventOpenCompleted = 1UL << 0,
    NSStreamEventHasBytesAvailable = 1UL << 1,
    NSStreamEventHasSpaceAvailable = 1UL << 2,
    NSStreamEventErrorOccurred = 1UL << 3,
    NSStreamEventEndEncountered = 1UL << 4
};
```
通常我们会关心NSStreamEventOpenCompleted、NSStreamEventHasBytesAvailable、NSStreamEventErrorOccurred、
由于数据是以流的形式回来，我们需要在在NSStreamEventHasBytesAvailable下取出数据然后做数据拼接，拼接好完整的数据才可使用，像这样
```
 case NSStreamEventHasBytesAvailable:
{
    UInt8 buffer[BUFFER_SIZE]; //设置缓存区
    NSInteger numBytesRead = 0;
    NSInputStream *inputstream = (NSInputStream *) aStream;
    // Read data
    do {
        numBytesRead = [inputstream read:buffer maxLength:sizeof(buffer)];
        if (numBytesRead > 0) {
            [self.resultData appendBytes:buffer length:numBytesRead];
        }
    } while (numBytesRead > 0);
}
break;
```
循环结束后我们的resultData就是完整的返回数据了。
