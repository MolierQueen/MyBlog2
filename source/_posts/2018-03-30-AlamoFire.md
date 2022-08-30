---
title: AlamoFire的使用(下载队列，断点续传)
categories: iOS开发
date: 2018-03-30 17:54:38
tags:
 - swift
 - 网络请求
comments:
---
## 前言

最近开始做了一个新项目，几乎没有时间来写自己的博客，大部分都在写feature（BUG），自己研究的东西很少，本来之前说好每个月要写两篇文章也没能坚持下来，最近在项目中遇到了一些问题，就在这里总结下吧。一些小的技巧而已，大神可以忽略了。

![](https://raw.githubusercontent.com/Alamofire/Alamofire/master/alamofire.png)
<!--more-->

## 背景

新项目包含了上传下载网络请求相关功能，由于是swift编写所以自然而然选择了 [AlamoFire](https://github.com/Alamofire/Alamofire) (好像也没得选)来做底层，正常的网络请求post、get等都是直接傻瓜式调用 AlamoFire 的接口，本文主要将一些细节问题

## 设置通用超时时间

使用Alamofire发起请求时候有这两个接口

```
/// Creates a `DataRequest` using the default `SessionManager` to retrieve the contents of the specified `url`,
/// `method`, `parameters`, `encoding` and `headers`.
///
/// - parameter url:        The URL.
/// - parameter method:     The HTTP method. `.get` by default.
/// - parameter parameters: The parameters. `nil` by default.
/// - parameter encoding:   The parameter encoding. `URLEncoding.default` by default.
/// - parameter headers:    The HTTP headers. `nil` by default.
///
/// - returns: The created `DataRequest`.
public func request(_ url: URLConvertible, method: Alamofire.HTTPMethod = default, parameters: Parameters? = default, encoding: ParameterEncoding = default, headers: HTTPHeaders? = default) -> Alamofire.DataRequest

/// Creates a `DataRequest` using the default `SessionManager` to retrieve the contents of a URL based on the
/// specified `urlRequest`.
///
/// - parameter urlRequest: The URL request
///
/// - returns: The created `DataRequest`.
public func request(_ urlRequest: URLRequestConvertible) -> Alamofire.DataRequest

```
而我们在调用的时候通常会直接这么用

```
let req : URLRequest = URLRequest(url: URL(fileURLWithPath: "32"), cachePolicy: .useProtocolCachePolicy, timeoutInterval: 10)

        // 第一种方法调用，后面参数直接用default
        Alamofire.request(URL(fileURLWithPath: "32"))

        // 第二中调用，使传入request
        Alamofire.request(req)
        let semaphore = DispatchSemaphore(value: 0)

```

其中第一种方法我们不能传入超时时间，第二中方法我们可以通过传入的URLRequest来设置超时时间，但是我们通常一个项目中大部分的请求，可能除了某些特殊的下载请求之外所有的超时时间都是一样的，这样的话我们需要同样的代码写好多遍，这个时候有两个办法

* 对生成Request的方法做一个封装，通用的参数如超时时间、header、请求方式 写死在方法里面，对于会变动的参数如 URL 和可以通过参数传入.

* 创建 `Alamofire.SessionManager` 通过sessionManager来设置超时时间等一些通用的东西

```
let networkManager : SessionManager = {
        let config : URLSessionConfiguration = URLSessionConfiguration.default
        config.timeoutIntervalForRequest = 10
        let manager = Alamofire.SessionManager.init(configuration: config)
        return manager
    }()
```

## 断点续传
Alamofire支持断点续传下载，原理就是将下载一半的数据保存到本地，然后下次再启动时候通过data的拼接来进行继续下载。用法也很简单，只是调用接口而已，关键是看开发者如何自己去维护这个已下载的数据，比如是存内存还是存硬盘，要存多久，淘汰策略是什么之类的。其实就是两个步骤， 断点和续传

#### 第一步 断点
监听下载中断，中断后将已经下载的数据进行保留，我这边用一个属性来存，具体到项目实现大家可以采用自己存储方式，存到硬盘或者数据库之类的

```
Alamofire.download("http://clips.vorwaerts-gmbh.de/big_buck_bunny.mp4", method: .get, parameters: nil, encoding: URLEncoding.default, headers: nil) { (url, response) -> (destinationURL: URL, options: DownloadRequest.DownloadOptions) in
            return (URL(fileURLWithPath: String(describing : NSSearchPathForDirectoriesInDomains(.documentDirectory, .userDomainMask, false)[0]+"123.mp4")), [.createIntermediateDirectories, .removePreviousFile])
            }.responseJSON { (response) in

                switch response.result {

                case .success:
                    print("success")
                case .failure:
                    //意外中断后在此处处理下载完成的部分
                    self.tmpData = response.resumeData

                default:
                    print("failed")
                }

        }
```

#### 第二步 续传

当下载再次启动时候，需要在上一步数据的基础上继续下载，我们调用Alamofire这个方法
```
/// Creates a `DownloadRequest` using the default `SessionManager` from the `resumeData` produced from a
/// previous request cancellation to retrieve the contents of the original request and save them to the `destination`.
///
/// If `destination` is not specified, the contents will remain in the temporary location determined by the
/// underlying URL session.
///
/// On the latest release of all the Apple platforms (iOS 10, macOS 10.12, tvOS 10, watchOS 3), `resumeData` is broken
/// on background URL session configurations. There's an underlying bug in the `resumeData` generation logic where the
/// data is written incorrectly and will always fail to resume the download. For more information about the bug and
/// possible workarounds, please refer to the following Stack Overflow post:
///
///    - http://stackoverflow.com/a/39347461/1342462
///
/// - parameter resumeData:  The resume data. This is an opaque data blob produced by `URLSessionDownloadTask`
///                          when a task is cancelled. See `URLSession -downloadTask(withResumeData:)` for additional
///                          information.
/// - parameter destination: The closure used to determine the destination of the downloaded file. `nil` by default.
///
/// - returns: The created `DownloadRequest`.
public func download(resumingWith resumeData: Data, to destination: Alamofire.DownloadRequest.DownloadFileDestination? = default) -> Alamofire.DownloadRequest
```
这个接口需要我们传入已存在的数据，然后基于我们传入的数据进行下载，它支持从新指定目的地路径，如果你有需要可以重新指定
```
Alamofire.download(resumingWith: tmpData!)
```
同样他返回一个request的对象，我们可以通过点语法来拿到进度、response等信息

## 批量下载

当我们需要同时下载很多东西的时候，往往需要我们自己维护一个下载队列，比如下一个载素材列表之类的。Alamo给我们提供了下载的接口，但是下载的线程队列需要我们自己去维护，其实就是一个多线程并发队列。

#### GCD

我们很自然而然的想到GCD，但是GCD有一个问题无法控制最大并发数，而且对队列的管理也并不完善，比如我们要下载100个文件，如果同时下载的话开辟100个线程，那肯定是不行的，先不说移动设备是否支持(最多70个左右)，即使支持了那这个开销太大。虽说GCD的话可以使用信号量进行线程控制，但是每个线程的暂停启动之类的又是问题，而且毕竟是曲线救国的方法。

#### OperationQueue

Operation及OperationQueue是基于GCD封装的对象，作为对象可以提供更多操作选择，可以用方法或block实现多线程任务，同时也可以利用继承、类别等进行一些其他操作；但同时实现代码相对复杂一些。但是他毕竟不像GCD那样使用C语言实现，所以效率会相比GCD低一些。但是对线程的控制的灵活性要远高于GCD，对于下载线程来说可以优先选择这个。

#### 实现

我们把每一个下载任务封装成一个operation。注意Operation不能直接使用，我们需要使用他的子类，这里我选择使用 `BlockOperation` 他的闭包则是需要执行的下载任务，然后我们把他添加进queue中便开始执行了任务

```
let op : BlockOperation = BlockOperation { [weak self] in
            Alamofire.download("http://clips.vorwaerts-gmbh.de/big_buck_bunny.mp4", method: .get, parameters: nil, encoding: URLEncoding.default, headers: nil) { (url, response) -> (destinationURL: URL, options: DownloadRequest.DownloadOptions) in
                return (URL(fileURLWithPath: String(describing : NSSearchPathForDirectoriesInDomains(.documentDirectory, .userDomainMask, false)[0]+"123.mp4")), [.createIntermediateDirectories, .removePreviousFile])
                }.downloadProgress { [weak self] (pro) in
                    let percent = Float(pro.completedUnitCount) / Float(pro.totalUnitCount)
                    if count == 0 {

                        self?.downLoadLabel.snp.remakeConstraints { (make) in
                            make.width.equalTo(300 * percent)
                            make.height.equalTo(30)
                            make.top.equalTo((self?.stopButton.snp.bottom)!).offset(30)
                            make.left.equalToSuperview().offset(30)
                        }
                    } else {
                        self?.downLoadLabel2.snp.remakeConstraints { (make) in
                            make.width.equalTo(300 * percent)
                            make.height.equalTo(30)
                            make.top.equalTo((self?.downLoadLabel.snp.bottom)!).offset(30)
                            make.left.equalToSuperview().offset(30)
                        }
                    }
                }.responseJSON { (response) in

                    switch response.result {

                    case .success:
                        print("success")
                    case .failure:
                        self?.tmpData = response.resumeData

                    default:
                        print("failed")
                    }

            }
        }

        queue.addOperation(op)
```
每一个opeeation对象我们都可以设置他的优先级、启动、暂停、等属性，简单的调用接口就可以，在此就不一一作解释了。然后我们需要对我们的queue进行设置，我们设置最大并发数，大家可以根据实际情况来设置，demo中我只有两个下载任务，所以我就设置最大并发数为1 这样就是一个一个下载。
```
let queue : OperationQueue = {
        let que : OperationQueue = OperationQueue()
        que.maxConcurrentOperationCount = 1
        return que
    }()

```
我们运行然后点击开始下载

![](https://wx1.sinaimg.cn/large/006tNc79gy1fpw0vlzbp6g308h0gnkjm.gif)

很奇怪我们发现他还是同时下载，我们又试了其他的个数，无论多少都是同时下载，最大线程数量完全不起作用，再反过来看下上面加入queue的任务。正常来说每一个operation都要等上一个operation完成后才会执行，而系统判断完成的标准就是上一个operation的闭包走完，我们闭包中放入的是一个下载任务，而Alamofire的下载都是异步执行，所以导致operation的闭包走完了，但是其实下载是异步在另一个线程执行的，实际上下载没有完成，知道原因我们对症下药，只需要保证operation闭包中的代码是同步执行的就OK了。而Alamofire是基于URLSession来实现的，并没有像connection那样提供同步的方法，所以我们使用信号量卡一下，像这样

![](https://wx1.sinaimg.cn/large/006tNc79gy1fpw1444jp0j31610qbgrr.jpg)

这样之后就会按照我们设置好的队列进行了

![](https://wx1.sinaimg.cn/large/006tNc79gy1fpw15riwhjg308h0gnb2i.gif)

有人会说下载同步进行会不会有影响，其实不会的首先我们实现同步的方式是信号量，本质上还是异步的只是我们阻塞的当前的下载线程，这个被阻塞线程一定不是主线程(除非Alamofire的开发者把他回调到主线程下载，这个基本不可能)，而且当我们把这个下载任务加到一个operation中之后，就注定不会在主线程中了，没一个operation都会被系统分配到一个非主线程的地方去做，所以这样不会性能有任何影响。

## 总结

因为时间紧迫，暂时做了这么多，也遇到了这些问题，所以写出了总结下，本文还会继续更新，会慢慢的整个网络层分享出来。就是可能更新会慢，毕竟工作量有点饱和。多谢关注

{% aplayerlist %}
{
    "autoplay": true,
    "showlrc": 0,
    "mutex": true,
    "music": [
        {
            "title": "Thank You Very Much",
            "author": "Margaret",
            "url": "https://molier-1256056152.cos.ap-guangzhou.myqcloud.com/Thankyou.mp3",
            "pic": "https://y.gtimg.cn/music/photo_new/T002R300x300M000000VbGX83hRicw.jpg?max_age=2592000",
            "lrc": "https://demo.meting.api.meto.moe/action/metingapi?server=tencent&type=lrc&id=001Ii3g54dIYpO"
        }
    ]
}
{% endaplayerlist %}
