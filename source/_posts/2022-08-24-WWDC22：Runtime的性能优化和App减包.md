---
title: WWDC22：Runtime的性能优化和App减包
sticky: true
categories: 译文
date: 2022-08-24 15:04:39
tags:
    - iOS
    - WWDC2022
comment:
cover: https://tva1.sinaimg.cn/large/e6c9d24egy1h5q2kjo3j2j21400u0411.jpg
---
本Session讲了为了让你的应用包体积更小，运行更快，启动速度更快，我们对Swift和Objective-C运行时做了怎样的优化。同时通过本Session你将发现如何通过高效的协议检查，更小的消息发送，以及优化后的ARC机制，来提高你的App性能。

![](https://tva1.sinaimg.cn/large/e6c9d24egy1h5hvcgyqjij21400u0411.jpg)

<!-- more -->
	
{% media audio %}
    - https://music.163.com/#/song?id=1929370102
{% endmedia %}


## 前言

WWDC2022 上苹果更新了Xcode14，里面提到了一些相关的优化。其中讲了通过对Swift和Objective-C运行时做了一些优化，达到了包体积变得更小、运行速度更快，启动速度更快的目的。如果你是用Xcode14来构建App，那么会有其中三点优化

* 高效的协议检查（针对Swift protocol check）
* 更快的消息发送机制（message send）
* release 和return调用优化（release & retain）
* Autorelease elision的优化（自动释放省略）

![wwdc](https://tva1.sinaimg.cn/large/e6c9d24egy1h5npazwav0j214g0rm75u.jpg)



当你用Swift或Objective-C编写代码时，其实是会经历三个个步骤。

* 编码，通过Xcode编写代码
* 编译，使用了Swift和Clang编译器
* 运行，通过Swift和Objective-C运行时中完成

![image-20220823144936761](https://tva1.sinaimg.cn/large/e6c9d24egy1h5gp9rcbq1j215t0u0754.jpg)

此次的这些关键优化其实就是在第三步骤运行时来完成的，运行时嵌入在我们所有平台的操作系统中。编译器在构建时不能做的事情，运行时可以做。而此次所有的修改其实对于开发者来说是无感透明的，所以任何代码都不用改动，只要你使用Xcode14来进行打包编译，便会享受的这些优化点。



## Swift协议检查（Protocol checks）        

先来看一个例子!

```swift
// 定义一个协议
protocol CustomLoggable {
  // 协议中定义一个属性 customString，只读属性
    var customLogString: String { get }        
}

// 定义一个log函数，参数为Any类型
func log(value: Any) {
//如果value遵循CustomLoggable协议，就输出字符串
    if let value = value as? CustomLoggable {
        debugPrint(value.customLogString)        
    } else {
        ...        
    }       
}

// 定义一个Event类型 遵循协议，并实现customLogString
struct Event: CustomLoggable {
    var name: String
    var date: String
    
    var customLogString: String {
        return "\(self.name), on \(self.date)"    
    }        
}
```

看上面代码，因为log函数的参数需要输出字符串，所以在输出前要先判断这个value是否遵循CustomLoggable协议，Swift是静态语言，所以一般来说这样的检查都是发生在编译时期。但是编译器不一定能拿到足够的协议元数据信息来完成检查。比如这里并不知道每次传入的 Any 类型是哪个确定类型，也就无法确定是否遵循 `CustomLoggable`协议。所以这种检查常常发生运行时，系统借助计算好的协议检查元数据(protocol check metadata)，运行库知道这个特殊对象是否符合协议。



这些元数据的构建虽然大部分在编译期间，但是还是有一部分是要在运行时完成，比如上面的例子，而且一个项目中肯定不止有一个协议，所以随着协议越多运行时的效率就越低，对于用户来说这个时间大部分是启动时间，所以用户感知为启动时间变长。而Xcode14新推的的Swift Runtime解决了这个问题，只要你是用Xcode14编译且运行在iOS16及以上版即可。



按照苹果的说法，他们会把`是否遵循协议`的这个判断前置到build时期，也就是把`协议元数据计算`的步骤前置到build中，具体就是他把这些操作放在App可执行文件和启动时任何动态库的dyld 闭包的一部分



为什么这样做可以节省启动时间，需要先了解下app启动流程，需要一个知识背景`从iOS11开始dyld3被加入，iOS13第三方库也开始使用dyld3加载。`所以我们要看下dyld3的加载流程



![img](https://upload-images.jianshu.io/upload_images/2438680-b5edfa4c2bcdb205.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/1044/format/webp)

*dyld 3* 主要包含了两个过程 进程外（启动前）和进程内（启动后），我们来看启动前做了那些事情

- 进程外 *Mach-O* 分析器和编译器 （*out-of-process mach-o parser*）
  *dyld 3* 中将采用提前写入把结果数据缓存成文件的方式构成一个 *lauch closure*（可以理解为缓存文件）
- 分析依赖库
- 执行符号查找
- *Write closure* 缓存服务 (*launch closure cache* )
  系统程序的 *closure* 直接内置在 *shared cache* 中，而对于第三方APP，将在APP安装或更新时生成，这样就能保证 *closure* 总是在 APP 打开之前准备好。说白了就是把上面做的结果全都缓存起来

综上看来以前需要在in-process中做的事情，现在在out-of-process就可以完成，启动时或者运行时直接读取缓存数据即可，加快了启动速度和运行时的性能。其实在笔者看来当我们下载或者更新App 的时候App上的进度条其实是分两部分`正在下载`和`正在安装`，此次的优化可能略微提高安装的时长来降低启动速度，提高运行时性能。

`on apps that rely heavily in Swift, this could add up to half the launch time` 如果有条件的同学可以试下是否可以提高这么多的启动耗时。



## 消息发送优化（Message send）

直接抛结果，苹果这边给到的数据是使用Xcode14编译打包的数据可以让ARM64上发送消息消耗从12字节降低到8字节，二进制大小也有2%的降低，也就是苹果对包大小和性能都做了优化，默认是同时开启的，由苹果来平衡两者的关系，当然也可以使用`objc_stubs_small`来仅仅优化包大小。

![image-20220823161510950](https://tva1.sinaimg.cn/large/e6c9d24egy1h5grqpljrej215t0u0aax.jpg)



下面我们看下是怎么优化的，同样使用官方代码举例

```objective-c
// 声明一个日历对象
NScalendar *cal = [self makeCalendar];

// 声明一个日期对象并赋值
NSDateComponents *dateComponents = [[NSDateComponents alloc] init];
dateComponents.year = 2022;
dateComponents.month = 2022;
dateComponents.day = 2022;
S
// 把日期转换为date
NSDate *theDate = [cal dateFromComponents: dateComponents];

// 返回date
return theDate;
```

大家知道OC调用方法最终会走到`_objc_msgSend`，所以上面代码不算最终的return，会走7个 `_objc_msgSend`，其中每一个都需要一条指令来调用就是bl 如下图

![image-20220823163343886](https://tva1.sinaimg.cn/large/e6c9d24egy1h5gsa0ef9kj20n00gidgx.jpg)



该函数定义为`Id objc_msgSend(id self, SEL _cmd, ...) `，参数定义为 self是函数的调用方，SEL为具体调用哪个函数，具体的方法查找流程就不在这里赘述。

我们拿其中具体的一个函数调用来分析

```objective-c
NSDate *theDate = [cal dateFromComponents: dateComponents]; 
```

比如这个函数调用，转化为mesagesend的时候就变成这样

```c
objc_msgSend(cal, @selector(dateFromComponents))
```

为了告诉运行时调用哪个方法，我们必须传递一个Selector给这些objc_msgSend调用，就如上图的`@selector(dateFromComponents)`

我们再来看`Id objc_msgSend(id self, SEL _cmd, ...)`执行后他是怎么执行汇编指令的。

```assembly
// 使用adrp找到该方法的地址   消耗4字节
adrp x1, [selector "dateFromComponents"]  

// 将 地址加载到X1寄存器中  消耗4字节
ldr  x1, [x1, selector "dateFromComponents"] 

// 执行bl指令跳转到该方法并执行  消耗4字节
bl _objc_msgSend
```

从上面的代码看出每次执行方法调用都会 走以上三个步骤，每个步骤消耗4字节 一共消耗12字节，而前两步是准备selector，任何一次方法调用都会执行他，目前的策略是每调一个方法都会生成上面三步，那么此时优化空间就来了。



因为这里存在相同的代码（前两步），`我们可以考虑共享它，并且只在每个 selector 中触发它一次，而不是每次发送消息时都生成这段指令代码`。所以我们可以把这部分相同代码提取出来，放到一个小助手函数中(helper function), 并调用该函数。通过使用同一 selector 进行多次调用(通过传递参数不同，内部指令是相同的，现在封装成一个存根函数，以前是散落在各个 _objc_msgSend 调用处)，我们可以保存所有这些指令字节。所以可以理解为`把前两步封装一下`

![image-20220823183604633](https://tva1.sinaimg.cn/large/e6c9d24egy1h5gvtbfnh1j20n00git9h.jpg)

所以原来的调用就变成了

```assembly
bl _objc_msgSend$dateFromComponents 4字节
bl _objc_msgSend    4字节
```

这也就是苹果说的从12字节优化到8字节，其中`_objc_msgSend$dateFromComponents`也被称为`selector stub 存根函数`



同样`_objc_msgSend`本身也有一个存根函数写法

![image-20220823184401200](https://tva1.sinaimg.cn/large/e6c9d24egy1h5gw1kpzuzj20n00giwfd.jpg)



这样一来我们现在就有两个存根函数

* _objc_msgSend$dateFromComponents:
* _objc_msgSend:

这两个函数封装了一些通用的东西，共享了最多的代码，使代码尽可能的小，但是这样带来的不足是我需要连着两个bl跳转，这对操作系统来说开销较大。所以为了平衡包体积和性能，我们可以使用下面这种方法来提升这一点。我们可以把前面调用的两个存根函数封装成一个(都封装成_objc_msgSend$dateFromComponents)，这样，我们可以使代码更紧凑，不需要那么多调用。如下图这样

![image-20220823185626349](https://tva1.sinaimg.cn/large/e6c9d24egy1h5gwei63esj20bo07u74l.jpg)

这就回到了之前的问题，你可以通过`_objc_stubs_small`标记了只降低包大小，或者采用默认的方式让系统自动平衡，两者的区别在汇编层面就体现在如下图

![image-20220823185804687](https://tva1.sinaimg.cn/large/e6c9d24egy1h5gwg7hnugj20n00gidhc.jpg)



综上：这就是Meesage send 占用从 12 bytes 降低到 8 bytes和二进制大小下降12%的原因



## Retain and release

这个优化是苹果这边使Retain and release的开销更小，苹果的说法是Retain and release的调用开销从8字节降低到4字节，同时包体积也会有2%的优化

![image-20220823190226429](https://tva1.sinaimg.cn/large/e6c9d24egy1h5gwkqxaenj20n00gidg9.jpg)

我们知道ARC相比于MRC是开发者不需要再写retain、release这些代码，其实并不是不需要，而是编译器帮我们自动在需要的位置插入了这些代码，所以换句话说他们还是存在的，只是你看不到也不用在关心他们。



还是拿之前的例子来说

```objective-c
// Retain/release calls inserted by ARC
NScalendar *cal = [self makeCalendar]; // bl    _objc_retain

NSDateComponents *dateComponents = [[NSDateComponents alloc] init]; // bl    _objc_retain
dateComponents.year = 2022; 
dateComponents.month = 2022;
dateComponents.day = 2022;

NSDate *theDate = [cal dateFromComponents: dateComponents]; // bl    _objc_retain
return theDate;
// bl    _objc_release 
// bl    _objc_release 
// bl    _objc_release 
```



在变量创建的时候我们使用retain来增加的他的引用计数不被销毁，在方法结束后我们使用release来销毁不需要的变量，这也是iOS的内存管理机制。在ARC下这些都是编译器我们插入的代码，我们无需关心。



retain和release都是C语言的函数，他们携带一个参数就是被操作的对象，同时他遵循C语言的ABI，所以当你调用这些方法的时候系统还会为你做一些额外的事情，比如下图中的mov操作，而这些操正是我们优化的用武之地，通过自定义调用重新约定 retain/release 接口，我们可以根据对象指针的位置，适当的使用正确的变量，这样就可以不用移动它。简单的说，`就是修改了底层 ABI`。

![image-20220823191546398](https://tva1.sinaimg.cn/large/e6c9d24egy1h5gwylzjkuj20n00giab1.jpg)

我们是怎么做的优化呢？看下之前的流程，我们用下面这行代码举例

```c
objc_release(dateComponents); 
// mov  x0, x20                    消耗4 字节                                                                                                                                                           
// bl    _objc_release         消耗4字节
```

流程为

* 先执行 mov 把副本地址（X20,也就是对象的地址）存到寄存器 x0
* 然后bl跳转到`_objc_release`函数进行释放

根据之前讲的每个指令消耗4字节，所以这里消耗8字节



我们修改ABI之后其省掉调用mov指令 然后原本跳转到_objc_release函数 改为跳转到`_objc_release_x20`函数，而mov的指令放到C语言更底层的ABI里面去做，你可以理解为`我们封装了一个新的retain、release函数，你只要传入一个寄存器地址我就去更底层的地方完成mov操作，所以效率更高了`。现在因为只用执行一条指令，所以内存消耗为4字节。现在的流程看起来为

![image-20220823192452605](https://tva1.sinaimg.cn/large/e6c9d24egy1h5gx83ebs8j20n00giab9.jpg)



这么看来我们代码里大量的release和retain 都经过这样的样的优化所以整体的二进制包降低2% 同时调用内存消耗游8字节变为4字节，同时ABI 接口修改，去除冗余 mov 指令调用，下沉到 ABI。`由于 ABI 是内嵌系统`，这里新增 mov 指令占用可以忽略不计。



`Apple 果然是坚持用户体验优先，为了更好体验不惜修改 c 的 ABI`



## Autorelease elision（自动释放省略优化）

 iOS中除了使用release之外还有另一个 就是autorelease自动释放机制，同样在这个地方苹果也做了自动释放省略的优化让自动释放机制效率更高。我们来看下面这个例子

```objective-c
// Return Value Autoreleases 

theWWDCDate = [event getWWDCDate];

-(NSDate*)getWWDCDate {
    ...
    return theDate;
}
```

创建一个临时对象(theDate)，并将其返回给调用方(event)。`getWWDCDate()` 方法中返回临时的 theDate，然后调用完成(返回 theDate 之后，getWWDCDate 就调用完成)。这时调用方（event）将其保存到自己的变量中（theWWDCDate 中）。

根据系统插入retain和release的机制来说应该是这样的，但是明显retain处不能进行release，因为我需要吧theDate返回回去，如果这里释放了我就没办法呢返回了。

![image-20220823195254398](https://tva1.sinaimg.cn/large/e6c9d24egy1h5gy19928nj20n00gi3yz.jpg)



因此，为了解决上述问题，需要使用一个特殊的约定用来返回这个临时返回值。这就引入了Autorelease，这样调用者能够 retain 它。autorelease 在这里保证在调用方可以正常返回该值，而不被提前释放，延长释放生命周期。你之前可能看到过 autorelease 和 autoreleasePools：其实这是一种将 release 操作推迟到稍后某个时间的方法。所以上面的代码改为Autorelease

```objective-c
// Return Value Autoreleases 

theWWDCDate = [[event getWWDCDate] retain];

-(NSDate*)getWWDCDate {
    ...
    return [theDate autorelease]; 
}
```

系统并不知道他在什么时候会被释放，反正只要不在retain的时候释放就行，所以我在retain的时候先打个标记，标记他之后可能会被释放。但是这样的操作目前会带来一些开销，其实就是`我虽然打了release标记，但是我明明一会还要retain，没必要多此一举`，所以基于此我们之前引入了`Autorelease elision`来减少这部分开销（`如果Autorelease后紧接一个retain我就都不做了`）。我们先从汇编层面看下Autorelease elision做了什么

![image-20220823195959365](https://tva1.sinaimg.cn/large/e6c9d24egy1h5gy8n9wnzj20n00giwf7.jpg)

提炼出以下代码

```c
// What the compiler emits
 bl    _getWWDCDate 
 mov   x29, x29
 bl    _objc_retainAutoreleasedReturnValue

 b    _objc_autoreleaseReturnValue   // autorelease -> runtime -> _objc_autoreleaseReturnValue
```



其实就是以下步骤

* 当我们返回值调用Autorelease时候系统会调用`_objc_autoreleaseReturnValue`来返回一个`autoreleased value`
* 执行Autorelease后编译器会添加个标记`mov x29, x29`  而这句指令在实际运行中这个指令会变为二进制的形式变为`0xAA1D03FD`
* 后续的操作就运行时会先判断是否有对应的标记`0xAA1D03FD`，如果有，这意味着编译器告诉runtime, 我们将返回一个已经被标记，但是将立即被持有（retain） 的临时变量，后面就不需要再retain操作了

![image-20220823210550374](https://tva1.sinaimg.cn/large/e6c9d24egy1h5h056kleej20n00gigm5.jpg)

```Assembly
static ALWAYS_INLINE bool 
callerAcceptsOptimizedReturn(const void *ra)
{
    // fd 03 1d aa    mov fp, fp
    // arm64 instructions are well-aligned
    if (*(uint32_t *)ra == 0xaa1d03fd) {
        return true;
        // 返回true 需要优化 把release、rentain删掉
    }
    return false;
}
```





说白了就是在返回值身上调用`objc_autoreleaseReturnValue`方法时，runtime将这个返回值object标记（储存在TLS中），然后直接返回这个object（不调用autorelease）；同时，在外部接收这个返回值的`objc_retainAutoreleasedReturnValue`里，发现有之前的标记（TLS中正好存了这个对象），那么直接返回这个object（清楚之前的标记且不再调用retain）。

注意：TLS相关的含义可以参考[这里]([EarlGrey 源码阅读（一） | SeanChense](http://seanchense.github.io/2018/09/20/earlgrey-source-code-read-1/#TLS))



但是这里有一个问题，以二进制的形式来加载代码并不是很常见，而且我们不但要加载它还要比较他尤其在CPU上并不是最优策略，所以这里还是有开销的，因此我们看下如何优化。

同样执行流程，当执行完` _objc_autoreleaseReturnValue`函数时候我们会获得一个返回地址，这个地址是一个指针，指向了被标记为Autorelease的对象。然后代码继续执行到`_objc_retainAutoreleasedReturnValue` 这里要进行reatin，而被reatain的变量地址我们也可以拿到，所以只要比较这两个指针即可，这样一来我们也不再需要mov操作

![image-20220823213224122](https://tva1.sinaimg.cn/large/e6c9d24egy1h5h0ws62g0j20n00gidg3.jpg)



优化点

* 把原来的比较二进制数据改为比较指针。速度更快效率更高
* 减少mov指令 减少4字节，二进制大小预计降低2%



## 总结

这就是Xcode14+iOS16的编译期间优化，可以看出苹果也在帮我们完成OKR减少包体积，提高启动速度，增加代码执行效率，同时也能看出苹果在追求极致用户体验道路上所做的事情。本文部分翻译自[Improve app size and runtime performance](https://developer.apple.com/videos/play/wwdc2022/110363/)，同时也添加了自己的思考。

