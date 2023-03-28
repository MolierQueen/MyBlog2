---
title: IOS下三种DNS解析方式分析（LocalDns）
date: 2018-01-23 18:02:29
tags:
 - DNS解析
 - LocalDNS
categories: iOS开发
comments:
---
### 背景
最近在做iOS的DNS解析，顺便研究了下iOS端本地的DNS解析方式（localDNS）,也就是不依赖Http请求，而是用原始的API进行解析，虽然有HttpDNS但是考虑到成本、第三方服务稳定性的问题，LocalDNS仍然是一个很重要的部分，在iOS系统下，localDNS的解析方式有三种，下面主要对三种方式进行下利弊分析及简单的原理介绍。
### 方式一
这个也是我一开始在项目中使用的方式。
```
1：struct hostent	*gethostbyname(const char *);
2：struct hostent	*gethostbyname2(const char *, int);
```
两个函数作用完全一样，返回值一样，但是第一个只能用于IPV4的网络环境，而第二个则IPV4和IPV6都可使用，可以通过第二个参数传入当前的网络环境。
<!---more--->
###### 使用方式：
```
 CFAbsoluteTime start = CFAbsoluteTimeGetCurrent();

    char   *ptr, **pptr;
    struct hostent *hptr;
    char   str[32];
    ptr = "www.meitu.com";
    NSMutableArray * ips = [NSMutableArray array];

     if((hptr = gethostbyname(ptr)) == NULL)
    {
        return;
    }

    for(pptr=hptr->h_addr_list; *pptr!=NULL; pptr++) {
         NSString * ipStr = [NSString stringWithCString:inet_ntop(hptr->h_addrtype, *pptr, str, sizeof(str)) encoding:NSUTF8StringEncoding];
         [ips addObject:ipStr?:@""];
    }

    CFAbsoluteTime end = CFAbsoluteTimeGetCurrent();
    NSLog(@"22222 === ip === %@ === time cost: %0.3fs", ips,end - start);
```

使用gethostbyname方法后会得到一个struct,也就是上文的struct hostent *hptr：
```
struct hostent {
	char	*h_name;	/* official name of host */
	char	**h_aliases;	/* alias list */
	int	h_addrtype;	/* host address type */
	int	h_length;	/* length of address */
	char	**h_addr_list;	/* list of addresses from name server */
#if !defined(_POSIX_C_SOURCE) || defined(_DARWIN_C_SOURCE)
#define	h_addr	h_addr_list[0]	/* address, for backward compatibility */
#endif /* (!_POSIX_C_SOURCE || _DARWIN_C_SOURCE) */
};
```

###### 参数解析：
 * hostent->h_name
   表示的是主机的规范名。例如www.baidu.com的规范名其实是www.a.shifen.com。

 * hostent->h_aliases
   表示的是主机的别名www.baidu.com的别名就是他自己。有的时候，有的主机可能有好几个别名，这些，其实都是为了易于用户记忆而为自己的网站多取的名字。

 * hostent->h_addrtype     
   表示的是主机ip地址的类型，到底是ipv4(AF_INET)，还是pv6(AF_INET6)

 * hostent->h_length       
   表示的是主机ip地址的长度

 * hostent->h_addr_lisst
   表示的是主机的ip地址，注意，这个是以网络字节序存储的。不要直接用printf带%s参数来打这个东西，会有问题的哇。所以到真正需要打印出这个IP的话，需要调用`const char *inet_ntop(int af, const void *src, char *dst, socklen_t cnt) `，来把它转成char。详细使用见上文
###### 缺点：
* 在进行网络切换的时候小概率卡死，自测十次有一两次左右。

* 在本地的LocalDns被破坏的时候会必卡死30秒，然后返回nil 。

* 缓存是个玄学东西，他会对自己解析出来的IP进行缓存（可能是运营商缓存）缓存时间不确定，有可能我即使切换了无数个网络，但是从早到晚同一个域名总是解析出同样的IP，

* 网上说的比较多的问题

  ![image.png](https://cdn.cdnjson.com/tvax3.sinaimg.cn/large/006tNc79gy1fo6myoq10ej30po01w0tb.jpg)

### 方式二
除了经常用到的gethostbyname(3)和gethostbyaddr(3)函数以外, Linux(以及其它UNIX/UNIX-like系统)还提供了一套用于在底层处理DNS相关问题的函数(这里所说的底层仅是相对gethostbyname和gethostbyaddr两个函数而言). 这套函数被称为地址解析函数(resolver functions)。曾经尝试过这个方式...
```
int		res_query __P((const char *, int, int, u_char *, int));
函数原型为：
int res_query(const char *dname, int class, int type, unsigned char *answer, int anslen)
```
这个方式需要在项目中添加`libresolv.tbd`库，因为要依赖于库中的函数去解析。`res_query`用来发出一个指定类(由参数class指定)和类型(由参数type指定)的DNS询问. dname是要查询的主机名. 返回信息被存储在answser指向的内存区域中. 信息的长度不能大于anslen个字节. 这个函数会创建一个DNS查询报文并把它发送到指定的DNS服务器。
###### 使用方式
```
CFAbsoluteTime start = CFAbsoluteTimeGetCurrent();

    unsigned char auResult[512];
    int nBytesRead = 0;

    nBytesRead = res_query("www.meitu.com", ns_c_in, ns_t_a, auResult, sizeof(auResult));

    ns_msg handle;
    ns_initparse(auResult, nBytesRead, &handle);

    NSMutableArray *ipList = nil;
    int msg_count = ns_msg_count(handle, ns_s_an);
    if (msg_count > 0) {
        ipList = [[NSMutableArray alloc] initWithCapacity:msg_count];
        for(int rrnum = 0; rrnum < msg_count; rrnum++) {
            ns_rr rr;
            if(ns_parserr(&handle, ns_s_an, rrnum, &rr) == 0) {
                char ip1[16];
                strcpy(ip1, inet_ntoa(*(struct in_addr *)ns_rr_rdata(rr)));
                NSString *ipString = [[NSString alloc] initWithCString:ip1 encoding:NSASCIIStringEncoding];
                if (![ipString isEqualToString:@""]) {

                    //将提取到的IP地址放到数组中
                    [ipList addObject:ipString];
                }
            }
        }
        CFAbsoluteTime end = CFAbsoluteTimeGetCurrent();
        NSLog(@"11111 === ip === %@ === time cost: %0.3fs", ipList,end - start);
    }
```
###### 参数解析

由于该逻辑是Linux底层提供的代码，苹果用宏做了一次封装，具体的函数含义还需要对Linux内核的理解，这里放一篇[参考资料](https://www.cnblogs.com/renhao/archive/2011/11/14/2248528.html)
###### 优点：
* 在LocalDns被破坏掉的情况下能及时响应不会延迟。
* 没有缓存，缓存由开发者控制
###### 缺点
* 在进行网络切换时候3G/4G切wify高概率出现卡死
这一个缺点是比较致命的，所以没有再继续使用。

### 方式三
苹果原生的DNS解析

```
Boolean CFHostStartInfoResolution (CFHostRef theHost, CFHostInfoType info, CFStreamError *error);
```
###### 使用方法：
```
    Boolean result,bResolved;
    CFHostRef hostRef;
    CFArrayRef addresses = NULL;
    NSMutableArray * ipsArr = [[NSMutableArray alloc] init];

    CFStringRef hostNameRef = CFStringCreateWithCString(kCFAllocatorDefault, "www.meitu.com", kCFStringEncodingASCII);

    hostRef = CFHostCreateWithName(kCFAllocatorDefault, hostNameRef);
    CFAbsoluteTime start = CFAbsoluteTimeGetCurrent();
    result = CFHostStartInfoResolution(hostRef, kCFHostAddresses, NULL);
    if (result == TRUE) {
        addresses = CFHostGetAddressing(hostRef, &result);
    }
    bResolved = result == TRUE ? true : false;

    if(bResolved)
    {
        struct sockaddr_in* remoteAddr;
        for(int i = 0; i < CFArrayGetCount(addresses); i++)
        {
            CFDataRef saData = (CFDataRef)CFArrayGetValueAtIndex(addresses, i);
            remoteAddr = (struct sockaddr_in*)CFDataGetBytePtr(saData);

            if(remoteAddr != NULL)
            {
                //获取IP地址
                char ip[16];
                strcpy(ip, inet_ntoa(remoteAddr->sin_addr));
                NSString * ipStr = [NSString stringWithCString:ip encoding:NSUTF8StringEncoding];
                [ipsArr addObject:ipStr];
            }
        }
    }
    CFAbsoluteTime end = CFAbsoluteTimeGetCurrent();
    NSLog(@"33333 === ip === %@ === time cost: %0.3fs", ipsArr,end - start);
    CFRelease(hostNameRef);
    CFRelease(hostRef);
```
###### 参数解析：
```
/*
 *  CFHostStartInfoResolution()
 *  
 *  Discussion:
 *	Performs a lookup for the given host.  It will search for the
 *	requested information if there is no other active request.
 *	Previously cached information of the given type will be released.
 *  
 *  Mac OS X threading:
 *	Thread safe
 *  
 *  Parameters:
 *
 *	theHost:  //需要被解决的CFHostRef的对象
 *	  The CFHostRef which should be resolved. Must be non-NULL. If
 *	  this reference is not a valid CFHostRef, the behavior is
 *	  undefined.
 *
 *	info: 返回值的类型 数组/Data/string..
 *	  The enum representing the type of information to be retrieved.
 *	  If the value is not a valid type, the behavior is undefined.
 *
 *	error: 错误
 *	  A reference to a CFStreamError structure which will be filled
 *	  with any error information should an error occur.  May be set
 *	  to NULL if error information is not wanted.
 *  
 *  Result: 解析结果成功还是失败
 *	Returns TRUE on success and FALSE on failure.  In asynchronous
 *	mode, this function will return immediately.  In synchronous
 *	mode, it will block until the resolve has completed or until the
 *	resolve is cancelled.
 *  
 */
CFN_EXPORT __nullable CFArrayRef
CFHostGetAddressing(CFHostRef theHost, Boolean * __nullable hasBeenResolved) CF_AVAILABLE(10_3, 2_0);
```
###### 优点：
* 在网络切换时候不会卡顿。
###### 缺点：
* 在本地DNS被破坏的情况下会出现卡死的现象(卡30s)
### 总结：
以上三个方法除了第二个方法会在网络切换时候卡死不可用之外，其他两个方法都是可选择的，关于那个本地LocalDns破坏会卡死的问题看来是无法避免，不过开发者可以自行通过ping等方式来判断LocalDns的正确性，在被破坏的情况下使用httpDns来进行解析即可。具体的demo可以到[这里](https://github.com/zhangninghao/LocalDns)查看
