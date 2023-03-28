---
title: 可能是最全的iOS端HttpDns集成方案
categories: iOS开发
date: 2018-01-24 13:54:36
tags:
  - 网络
  - httpdns
  - 底层
comments:
---
# 科普片
##### 1、DNS劫持的危害
  不知道大家有没有发现这样一个现象，在打开一些网页的时候会弹出一些与所浏览网页不相关的内容比如这样奇(se)怪(qing)的东西

![图一](https://cdn.cdnjson.com/tvax3.sinaimg.cn/large/006tNc79gy1fo6jm5b190j30h60ugqkg.jpg)

或者这样
<!--more-->

![图二](https://cdn.cdnjson.com/tvax3.sinaimg.cn/large/006tNc79gy1fo6jmr1afuj30ww0ue4qp.jpg)，

其实造成这样的原因就是DNS劫持，在我们正常浏览的网页链接里面被恶意插入一些奇怪的东西。不止是这些，DNS劫持还会对我们的个人信息安全造成很大的伤害，钓鱼网站之类的，也许我们所访问的网站根本不是我们需要的网站，或者根本打不开网页，有时还会消耗我们过多的流量。
##### 2、什么是DNS解析
  现在假如我们访问一个网站www.baidu.com从按下回车到百度页面显示到我们的电脑上会经历如下几个步骤
* 1：计算机会向我们的运营商(移动、电信、联通等)发出打开www.baidu.com的请求。
* 2：运营商收到请求后会到自己的DNS服务器中找www.baidu.com这个域名所对应的服务器的IP地址(也就是百度的服务器的IP地址)，这里比如是180.149.132.47。
* 3：运营商用第二步得到的IP地址去找到百度的服务器请求得到数据后返回给我们。

其中第二步就是我们所说的DNS解析过程，域名和IP地址的关系其实就是我们的身份证号和姓名的关系，都是来标记一个人或者是一个网站的，只是IP地址\身份证号只是一串没有意义的数字，辨识度低，又不好记，所以就会在IP上加上一个域名以便区分，或是做的更加个性化，但是如果真的要来准确的区分还是要靠身份证号码或者是IP的，所以DNS解析就应运而生了。
##### 3：什么是DNS劫持
DNS劫持，是指在DNS解析过程中拦截域名解析的请求，然后做一些自己的处理，比如返回假的IP地址或者什么都不做使请求失去响应，其效果就是对特定的网络不能反应或访问的是假网址。根本原因就是以下两点：
* 1：恶意攻击，拦截运营商的解析过程，把自己的非法东西嵌入其中。
* 2：运营商为了利益或者一些其他的因素，允许一些第三方在自己的链接里打打广告之类的。

##### 4：防止DNS劫持
  了解了DNS劫持的相关资料后我们就知道了，防止NDS劫持就要从第二步入手，因为DNS解析过程是运营商来操作的，我们不能去干涉他们，不然我们也就成了劫持者了，所以我们要做的就是在我们请求之前对我们的请求链接做一些修改，将我们原本的请求链接www.baidu.com 修改为180.149.132.47，然后请求出去，这样的话就运营商在拿到我们的请求后发现我们直接用的就是IP地址就会直接给我们放行，而不会去走他自己DNS解析了，也就是说我们把运营商要做的事情自己先做好了。不走他的DNS解析也就不会存在DNS被劫持的问题，从根本是解决了。
# 技术篇
##### 5：项目中的实际操作

###### 5.1：DNSPOD相关
  我们知道要要把项目中请求的接口替换成成IP其实很简单，URL是字符串，域名替换IP，无非就是一个字符串替换而已，的确这块其实没有什么技术含量，而且现在像阿里云(没开源)，七牛云(开源)，等一些比较大的平台在这方面也都有了比较成熟的解决方案，一个SDK，传个普通的URL进去就会返回一个域名被替换成IP的URL出来，也比较好用，这里要说一下IP地址的来源，如何拿到一个域名所对应的IP呢？这里就是需要用到另一个服务——HTTPDNS，国内比较有名的就是DNSPOD，包括阿里，七牛等也是使用他们的DNS服务来解析，就是这个

![DNSPOD logo](https://cdn.cdnjson.com/tvax3.sinaimg.cn/large/006tNc79gy1fo6jnbe2zhj30sa0i8dwv.jpg)

![简介](https://cdn.cdnjson.com/tvax3.sinaimg.cn/large/006tNc79gy1fo6jnt2uw9j31kw0icava.jpg)

他会给我们提供一个接口，我们使用HTTP请求的方式去请求这个接口，参数带上我们的域名，他们就会把域名对应的IP列表返回回来。类似这样：
```
///这个请求URL的结构是固定的119.29.29.29是DNSPOD固定的服务器地址，ttl参数的意思是返回结果是否带ttl是个BOOL，dn就是我们需要解析的域名，id就是我们在dnspod上注册时候他给我们的一个KEY
NSString *url = [NSString stringWithFormat:@"http://119.29.29.29/d?ttl=1&dn=www.baidu.com&id=KEY"];
NSMutableURLRequest * request = [NSMutableURLRequest requestWithURL:[NSURL URLWithString:url] cachePolicy:NSURLRequestUseProtocolCachePolicy timeoutInterval:10];
NSData * data = [NSURLConnection sendSynchronousRequest:request returningResponse:nil error:&networkError];
```

这里使用同步还是异步都是可以的，具体根据你们业务需求。

###### 5.2：项目中的使用
 
其实dnspod最难的部分是接入的部分，因为不同的APP不同的网络环境会导致各种各样的问题，如果你是一个新的项目那么接入难度会大大降低，因为你完全可以自己封装一套网络请求，把DNS解析相关的逻辑都封装到自己的网络请求中，这样你就可以得到APP所有的网络层的控制权，想干什么就干什么，但是如果是在一个已经比较完善的APP中加入DNS防劫持的话那就是比较困难，因为你不能拿到所有网络请求的控制权这篇文章中我主要使用是NSURLProtocol + Runtime hook方式来处理这些东西的，NSURLProtocol属于iOS黑魔法的一种可以拦截任何从APP的 URL Loading System系统中发出的请求，其中包括如下
* File Transfer Protocol (ftp://)
* Hypertext Transfer Protocol (http://)
* Hypertext Transfer Protocol with encryption (https://)
* Local file URLs (file:///)
* Data URLs (data://)

如果你的请求不在以上列表中就不能进行拦截了，比如WKWebview，AVPlayer(比较特殊，虽然请求也是http/https但是就是不走这套系统，苹果爸爸就是这样~)等，其实对于正常来说光用已经NSURLProtocol足够了。
  NSURLProtocol这个类我们不能直接使用，我们需要自己创建一个他的子类然后在我们的子类中操作他们像这样
  ```
 // 注册自定义protocol
[NSURLProtocol registerClass:[CustomURLProtocol class]];
 NSURLSessionConfiguration *configuration = [NSURLSessionConfiguration defaultSessionConfiguration];
configuration.protocolClasses = @[[CustomURLProtocol class]];
```
在这个类中我们可以拦截到请求，然后进行处理。这个类中有四个非常重要的方法
```
+ (BOOL)canInitWithRequest:(NSURLRequest *)request;
+ (NSURLRequest *)canonicalRequestForRequest:(NSURLRequest *)request;
- (void)startLoading;
//对于拦截的请求，NSURLProtocol对象在停止加载时调用该方法
- (void)stopLoading;
```
###### + (BOOL)canInitWithRequest:(NSURLRequest *)request;
通过返回值来告诉NSUrlProtocol对进来的请求是否拦截，比如我只拦截HTTP的，或者是某个域名的请求之类
###### + (NSURLRequest *)canonicalRequestForRequest:(NSURLRequest *)request;
如果上面的方法返回YES那么request会传到这里，这个地方通常不做处理 直接返回request

###### - (void)startLoading;
这个地方就是对我们拦截的请求做一些处理，我们文中所做的IP对域名的替换就在这里进行，处理完之后将请求转发出去，比如这样
```
- (void)startLoading {
///其中customRequest是处理过的请求(域名替换后的)
    NSURLSession *session = [NSURLSession sessionWithConfiguration:[[NSURLSessionConfiguration alloc] init] delegate:self delegateQueue:nil];
    NSURLSessionDataTask *task = [session dataTaskWithRequest:customRequest];
    [task resume];
}
```
你可以在 - startLoading 中使用任何方法来对协议对象持有的 request 进行转发，包括 NSURLSession、 NSURLConnection 甚至使用 AFNetworking 等网络库，只要你能在回调方法中把数据传回 client，帮助其正确渲染就可以，比如这样：
```
- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask didReceiveResponse:(NSURLResponse *)response completionHandler:(void (^)(NSURLSessionResponseDisposition))completionHandler {
    [[self client] URLProtocol:self didReceiveResponse:response cacheStoragePolicy:NSURLCacheStorageAllowed];

    completionHandler(NSURLSessionResponseAllow);
}

- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask didReceiveData:(NSData *)data {
    [[self client] URLProtocol:self didLoadData:data];
}
```
client在后面会有讲解。
###### - (void)stopLoading;
请求完毕后调用
大概的执行流程是这样

![流程](https://cdn.cdnjson.com/tvax3.sinaimg.cn/large/006tNc79gy1fo6jtpp3z0j30ze1by122.jpg)

在NSURLProtocol中有一个贯穿始终的变量
```
/*!
    @method client
    @abstract Returns the NSURLProtocolClient of the receiver.
    @result The NSURLProtocolClient of the receiver.  
*/
@property (nullable, readonly, retain) id <NSURLProtocolClient> client;

```
你可以认为是这个是请求的发送者，打个比方，A想给B发送一个消息，由于距离遥远于是A去了邮局，A把消息内容告诉了邮局，并且A在邮局登记了自己名字方便B有反馈的时候邮局来通知A查收。这个例子中邮局就是NSURLProtocol，A在邮局登记的名字就是client。所有的 client 都实现了 NSURLProtocolClient 协议，协议的作用就是在 HTTP 请求发出以及接受响应时向其它对象传输数据：
```
@protocol NSURLProtocolClient <NSObject>
...
- (void)URLProtocol:(NSURLProtocol *)protocol didReceiveResponse:(NSURLResponse *)response cacheStoragePolicy:(NSURLCacheStoragePolicy)policy;

- (void)URLProtocol:(NSURLProtocol *)protocol didLoadData:(NSData *)data;

- (void)URLProtocolDidFinishLoading:(NSURLProtocol *)protocol;
...
@end
```
当然这个协议中还有很多其他的方法，比如 HTTPS 验证、重定向以及响应缓存相关的方法，你需要在合适的时候调用这些代理方法，对信息进行传递。
到此正常情况下的DNS的解析过程已经结束，如果你发现按照如上操作之后并没有达到预期效果那么请往下看，(通常情况下完成以上操作 原有的URL的就会变成http://123.456.789.123/XXX/XXX/XXX的格式。如果发现请求不成功就往下看吧)

##### 6：遇到的坑点
###### 6.1：我们知道运营商本来是根据域名来确定一个URL的，我们将域名改为IP之后虽然不用运营商帮我们解析了，但是运营商在收到一串数字的时候也是懵逼状态，我们还是需要将域名传给他们，但是不能用正常的方式传，我们需要把原来的域名加到http请求的Header中的host字段下，根据Http协议的规定，如果在URL中无法找到域名的话就会去Header中找，这样一来我们既把域名告诉了运营商同时也直接制定了IP地址，这个是必须配置的，不然的话是请求不成功的。
```
[mutableRequest setValue:self.request.URL.host forHTTPHeaderField:@"HOST"];
```
###### 加上Header再去请求就没问题了，不过有些特殊的情况下会需要带上cookie，同样也是加到Header中
```
[mutableRequest setValue:YOUR Cookie forHTTPHeaderField:@"Cookie"];
```
###### 6.2：关于AfNetworking的问题，现在大部分网络请求是基于Afnetworking的，这里有一个坑，我们知道我们注册CustomProtocol的时候是这样
```
 // 注册自定义protocol
[NSURLProtocol registerClass:[CustomURLProtocol class]];
 NSURLSessionConfiguration *configuration = [NSURLSessionConfiguration defaultSessionConfiguration];
configuration.protocolClasses = @[[CustomURLProtocol class]];
```
###### 在系统的configuration加入我们的CustomProtocol，protocolClasses是一个数组里面可以放很多各种不同的CustomProtocol，我们看一下afnetworking的初始化方法。
```
AFHTTPSessionManager * sessionManager = [AFHTTPSessionManager manager];
```
###### 我相信大家通常都会这么来创建，但是这里我要说下manager并不是一个单利，最后都会调到一个方法
```
- (instancetype)initWithSessionConfiguration:(NSURLSessionConfiguration *)configuration {
    self = [super init];
    if (!self) {
        return nil;
    }

    if (!configuration) {
        configuration = [NSURLSessionConfiguration defaultSessionConfiguration];
    }

    self.sessionConfiguration = configuration;
    self.operationQueue = [[NSOperationQueue alloc] init];
    self.operationQueue.maxConcurrentOperationCount = 1;

    self.session = [NSURLSession sessionWithConfiguration:self.sessionConfiguration delegate:self delegateQueue:self.operationQueue];
    .
    .
    .
}

```
###### 大家注意第二个判断，如果没有传入configuration的话他会创建一个默认的，这样以至于我们之前在configuration的protocolClasses中注册类全部被这个新的configuration替换掉了，所以无法解析。这里我采取的办法就是runtime hook，因为hook第三方的代码并不是一个很好的办法，所以我直接hook NSURLSession的sessionWithConfiguration方法，因为通过观察Afnetworking的源码最终都是走到这里的。Hook之后把自己的configuration换进去，像这样
```
+ (NSURLSession *)swizzle_sessionWithConfiguration:(NSURLSessionConfiguration *)configuration {

    NSURLSessionConfiguration *newConfiguration = configuration;
    // 在现有的Configuration中插入我们自定义的protocol
    if (configuration) {
        NSMutableArray *protocolArray = [NSMutableArray arrayWithArray:configuration.protocolClasses];
        [protocolArray insertObject:[CustomProtocol class] atIndex:0];
        newConfiguration.protocolClasses = protocolArray;
    }
    else {
        newConfiguration = [NSURLSessionConfiguration defaultSessionConfiguration];
        NSMutableArray *protocolArray = [NSMutableArray arrayWithArray:configuration.protocolClasses];
        [protocolArray insertObject:[CustomProtocol class] atIndex:0];
        newConfiguration.protocolClasses = protocolArray;
    }

    return [self swizzle_sessionWithConfiguration:newConfiguration];
}
```
###### 然后就完美解决了。不过要注意下系统的是有两个方法的
```
/*
 * Customization of NSURLSession occurs during creation of a new session.
 * If you only need to use the convenience routines with custom
 * configuration options it is not necessary to specify a delegate.
 * If you do specify a delegate, the delegate will be retained until after
 * the delegate has been sent the URLSession:didBecomeInvalidWithError: message.
 */
+ (NSURLSession *)sessionWithConfiguration:(NSURLSessionConfiguration *)configuration;
+ (NSURLSession *)sessionWithConfiguration:(NSURLSessionConfiguration *)configuration delegate:(nullable id <NSURLSessionDelegate>)delegate delegateQueue:(nullable NSOperationQueue *)queue;
```
###### 这两个方法不能确定最终会走那个，所以为了保险起见都hook下，hook的方式是一样的
###### 6.3：AVPlayer请求，AVPlayer是我们iOS系统中系统自带的播放视频的框架，用到地方也很多，但是这个是比较坑的，因为AVPlayer虽然也有http/https/file……请求这个概念，但是AVPlayer所有的请求都不会走URL Loading System，也就是说所有由AVPlayer发出的请求都不能被我们的CustomProtocol拦截，这时候大家也许会问，不对呀，我们正常调试的时候可以被拦截到的啊。其实苹果官方上是说AVPlayer在真机调试和模拟器调试时候走的完全不是一套策略，也就是说在模拟器运行时候是完全正常的，可以被拦截到也可以被解析，但是在真机上面就恰恰相反了，因为我们最后还是以真机为准，所以我们采取的办法还是hook，因为我们需要在媒体URL传给AVPlayer前就要将相关东西配置好，域名替换啊，加host啊之类的，所以我们要找AVPlayer的入口，先看初始化方法，我发现项目中使用一个AVURLAsset来初始化AVPlayer，那么AVURLAsset又是什么呢？继续查到AVURLAsset的初始化方法，可以发现这个方法：
```
/*!
  @method		initWithURL:options:
  @abstract		Initializes an instance of AVURLAsset for inspection of a media resource.
  @param		URL
				An instance of NSURL that references a media resource.
  @param		options
				An instance of NSDictionary that contains keys for specifying options for the initialization of the AVURLAsset. See AVURLAssetPreferPreciseDurationAndTimingKey and AVURLAssetReferenceRestrictionsKey above.
  @result		An instance of AVURLAsset.
*/
- (instancetype)initWithURL:(NSURL *)URL options:(nullable NSDictionary<NSString *, id> *)options NS_DESIGNATED_INITIALIZER;
```
###### 其中URL就是我们传给AVPlayer播放的URL，找到目标就Hook下就可以了，具体过程就不多说了还是字符串替换，但是有一点需要注意的是，我之前上文说过做完IP对域名的替换之后还需要设置下request的Host，但是这个地方只有一个URL并没有Request该如何处理呢？其实这个方法里面的opinion参数就是处理这个的，可以添加cookie之类的类似与httpheader的东西，可以添加这几个Key
```
AVF_EXPORT NSString *const AVURLAssetPreferPreciseDurationAndTimingKey NS_AVAILABLE(10_7, 4_0);
AVF_EXPORT NSString *const AVURLAssetReferenceRestrictionsKey NS_AVAILABLE(10_7, 5_0);
AVF_EXPORT NSString *const AVURLAssetHTTPCookiesKey NS_AVAILABLE_IOS(8_0);
AVF_EXPORT NSString *const AVURLAssetAllowsCellularAccessKey NS_AVAILABLE_IOS(10_0);
```
###### 但是并没有发现和Host相关的Key，其实这个key是有的就是AVURLAssetHTTPHeaderFieldsKey只是因为这个Key没暴露出来。这个地方不太确定是不是苹果的私有API，网上查了大量的资料也没有个说法，甚至我亲自去苹果开发者去问，苹果也没有给任何答复，各种说法都有，具体使用的话就是
```
[self swizzle_initWithURL:videoURL options:@{AVURLAssetHTTPHeaderFieldsKey : @{@"Host":host}}]
```
###### 这样使用是没有任何问题的，但是毕竟是没有暴露出来的方法，我们不能这样明目张胆的使用，其实对于字符串来说还是比较好规避的，只要不要明文出现这个KEY就可以，我在这里使用了一个加密，吧key变成密文然后这个地方通过解密获取，就像这样：
```
//加密后的KEY
const NSString * headerKey = @"35905FF45AFA4C579B7DE2403C7CA0CCB59AA83D660E60C9D444AFE13323618F";
.
.
.
//getRequestHeaderKey方法为解密方法
return [self swizzle_initWithURL:videoURL options:@{[self getRequestHeaderKey] : @{@"Host":host}}];
```
###### 这样之后就大功告成了，AVPlayer可以在DNS被劫持的情况下播放了，
###### 6.4：POST请求这块也算是一个大坑，我们知道http的post请求会包含一个body体，里面包含我们需要上传的参数等一些资料，对于POST请求我们的NSURLProtocol是可以正常拦截的，但是我们拦截之后发现无论怎么样我们获得的body体都为nil！后来查了一些资料发下又是苹果爸爸在做手脚。NSURLProtocol在拦截NSURLSession的POST请求时不能获取到Request中的HTTPBody，这个貌似早就国外的论坛上传开了，但国内好像还鲜有人知，据苹果官方的解释是Body是NSData类型，即可能为二进制内容，而且还没有大小限制，所以可能会很大，为了性能考虑，索性就拦截时就不拷贝了（内流满面脸）。为了解决这个问题，我们可以通过把Body数据放到Header中，不过Header的大小好像是有限制的，我试过2M是没有问题，不过超过10M就直接Request timeout了。。。而且当Body数据为二进制数据时这招也没辙了，因为Header里都是文本数据，另一种方案就是用一个NSDictionary或NSCache保存没有请求的Body数据，用URL为key，最后方法就是别用NSURLSession，老老实实用古老的NSURLConnection算了。。。你以为这么就结束了吗？并没有，后来查了大量的资料发现，既然post请求的httpbody没有苹果复制下来，那我们就不用httpbody，我们再往底层去看就会发现HTTPBodyStream这个东西我们可以通过他来获取请求的body体具体代吗如下
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
###### 这样之后的req就是携带了body体的request啦，可以愉快地做post请求啦。
###### 6.5：WKWebview是新出的浏览器控件，这里就不多说了，WKWebview不走URL Loading System，所以也不会被拦截，不过也是有办法的，但是因为这次项目中没有用到，所以没有过多的去研究，后续我会写一篇关于这个博客，不是很难，依旧是runtime大法。
###### 6.6：SNI环境，这个可是坑了我好久好久的东西，所以我会放在最后去说，SNI环境因为涉及到证书验证所以是在https的基础上来说的，SNI（Server Name Indication）是为了解决一个服务器使用多个域名和证书的扩展。一句话简述它的工作原理就是，在连接到服务器建立SSL链接之前先发送要访问站点的域名（Hostname），这样服务器根据这个域名返回一个合适的证书。其实关于SNI环境在这里就不过多解释，**[阿里云文档](https://help.aliyun.com/document_detail/30143.html)**有很明白的解释，同时他也有安卓和iOS在SNI环境下的处理文档，我们发现安卓部分写的很详细，可是已到了iOS这边就这样了：
![阿里云文档截图](https://cdn.cdnjson.com/tvax3.sinaimg.cn/large/006tNc79gy1fo6juzsqdqj31kw0bbn5v.jpg)
###### 三行文字加三个链接就完事了。其实在遇到这个坑的时候我也查过很多相关资料，无非就是这三行话加这三个链接复制来复制去，没有实质性的进展，大部分公司或者是项目没有这么重的Httpdns需求，所以也就不会有这个环境，即使遇到了也就直接关闭httpdns了，后来只能自己去用CFNetwork一点点实现。具体代码就不跟大家粘贴了因为涉及到一些公司内部的代码，不过我会把我**[主要的参考资料](https://github.com/Dave1991/alicloud-ios-demo/blob/master/httpdns_ios_demo/httpdns_ios_demo/CFHttpMessageURLProtocol.m)**发给大家。这里有个小技巧，因为都在说CFNetwork是比较底层的网络实现，好多东西需要开发者自行处理比如一些变量的释放之类的，所以我们能少用尽量少用，因为Cfnetwork是为SNI(https)环境服务,所以我们在拦截判断的时候可以区分是用上层的网络请求转发还是用底层的cfnetwork来转发，
```
 if ([self.request.URL.scheme isEqualToString:@"https"] ) {
//使用CFnetwork
        curRequest = req;
        self.task = [[CustomCFNetworkRequestTask alloc] initWithURLRequest:originalRequest swizzleRequest:curRequest delegate:self];
        if (self.task) {
            [self.task startLoading];
        }
    } else {
//使用普通网络请求
        NSURLSessionConfiguration *configuration = [NSURLSessionConfiguration defaultSessionConfiguration];
        self.session = [NSURLSession sessionWithConfiguration:configuration delegate:self delegateQueue:[NSOperationQueue mainQueue]];
        NSURLSessionTask *task = [self.session dataTaskWithRequest:req];
        [task resume];
    }
```
###### 我是这么做的。
###### 6.7：在NSURLProtocol中的那几个类方法中是可以发送同步请求的，但是在实例方法发送同步请求就会卡死，所以实例方法中不能有任何的阻塞，进行同步操作。不然就卡死。
##### 7：总结
  完成了以上的步骤之后你回发现在DNS坏掉的情况下手机里面除了微信QQ(他们也做了DNS解析)之外其他应用都不能上网了但是你的App依然可以正常浏览网络数据。这就是我最近在做的时候遇到的一些问题，有什么问题及时与我交流吧
