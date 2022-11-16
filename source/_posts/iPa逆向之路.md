---
title: iPa逆向之路
categories: iOS开发
date: 2022-11-15 19:40:49
cover: https://tva1.sinaimg.cn/large/008vxvgGgy1h861sxytvij30ka0aoq3b.jpg
tags:
    - iOS
    - 逆向
comments:
---
## 背景

逆向工程相对于正向的开发，可能关注的没有那么高，尤其是相比于安卓或者其他平台，苹果的安全机制更严格，逆向的流程也会更繁琐，除了有ASLR（地址空间布局随机化），还有FairPlay DRM的iPA加密方式，也就是我们俗称的壳。这个给逆向工作带来了更多的挑战。但是`更好更安全的加密方式也只是增加破解的成本，并不是绝对的安全`，这也是逆向的前提。



最近也正在做一些调研的工作，需要从技术层面去分析其他App的一些底层逻辑，要用到iOS的逆向相关的技术，但是由于笔者做这些工作的时候正处于MacOS、iOS、Xcode三个系统的大版本更新期间，一些系统的运行方式和逻辑发生变化，所以导致网上能找到的资料基本都失效了，所以写文档记录下。



## 前置工作

### 环境

* Mac架构： Intel架构
* MacOS： 13.0.1 (22A400)
* Xcode版本：Version 14.1 (14B47b)
* iOS系统版本：iOS 16.0

### 相关工具

* [MachOView](https://sourceforge.net/projects/machoview/)
  * 用来查看Mach-o的文件结构，以及各个部分的信息
* [class-dump](http://stevenygard.com/projects/class-dump/)
  * class-dump，顾名思义，就是用来dump目标对象 的class信息的工具。它利用Objective-C语言的runtime 特性，将存储在Mach-O文件中的头文件信息提取出 来，并生成对应的.h文件。
* [MonkeyDev](https://github.com/AloneMonkey/MonkeyDev)
  * 非越狱开发插件，可以进行动态库注入，hook相关操作
* [Hopper Disassembler](https://www.hopperapp.com/)
  * Hopper Disassembler是Mac上的一款二进制反汇编器，基本上满足了工作上的反汇编的需要，包括伪代码以及控制流图(Control Flow Graph)，支持ARM指令集并针对Objective-C的做了优化。



## iPa下载

iOS App的逆向的所有操作都是基于iPa的操作，所以大前提是要有目标iPa，这里提供三种方式来进行iPa下载，大家可以选择适合自己的方式下载。



### 方式1：三方应用市场

现在这样的应用市场比较多，多是平替iTunes的一些软件
* [爱思助手](https://www.i4.cn/)
* [iTools](https://www.itools.cn/)

使用如上的三方软件可以很快的下载对应的ipa包，但是由于上述市场都是镜像自AppStore的内容，并且自己重签名，所以更新的及时性可能没有那么快，也没有那么全，而且因为是被第三方进行了修改重签，所以内容也不一定保证和官方的一致。如果不在乎这些的话还是可以采取这类的方式下载。



### 方式2：[Apple Configuration](https://support.apple.com/apple-configurator)

可以直接从Mac 上的Apple Store上下载，官方出品，原本是给手机上安装app的。用此方式其实是利用了该App的App下载机制来进行ipa导出的

![image-20221114144926709](https://tva1.sinaimg.cn/large/008vxvgGgy1h84np4us32j310y0io75z.jpg)

选择添加App，然后在弹出的弹窗中选择App并且下载

![image-20221114145027227](https://tva1.sinaimg.cn/large/008vxvgGgy1h84nq4zkpwj30ys0na0vm.jpg)

这个时候如果你手机上没有安装该App，则直接会安装成功，此时我们再点击安装下载，然后就会收到`设备上已经存在相同的App，是否覆盖安装的提示`的弹窗，此时我们<font color='red'> 不要理会 </font>这个弹窗。

![image-20221114145604307](https://tva1.sinaimg.cn/large/008vxvgGgy1h84nvzsytmj30u00y6mz0.jpg)

然后到如下路径就可以取到对应的ipa

```
~/Library/Group Containers/K36BKF7T3D.group.com.apple.configurator/Library/Caches/Assets/TemporaryItems/MobileApps/
```



### 方式3 [DumpApp]([https://dumpapp.com](https://dumpapp.com/))

是一个第三的网站，同在线砸壳+ipa下载的服务，因为我们最终想要的就是一个砸壳之后的ipa，所以这个网站直接帮我们做好了，只不过是收费的，每个app是9元，但是有多个境外的App市场，比较全面。

![image-20221114154450405](https://tva1.sinaimg.cn/large/008vxvgGgy1h84par73tqj30be07saa1.jpg)



## iPa砸壳

如果iPa的获取方式选择方式3，则可以略过砸壳步骤

app 上传到AppStore后   苹果使用 fairplay DRM来加密，就是我们所说的壳DRM全称Digital Rights Management，即数字版权保护。苹果为了保护App Store分发的音乐/视频/书籍/App免于盗版，开发了Fairplay DRM技术。

所有逆向都是建立在砸壳的前提下，砸壳的方式有两种：

### 静态砸壳

就是不依赖程序运行，直接用ipa包就可以进行砸壳解密，比如说我已经知道了他的加密算法，或者我通过暴力破解了他的加密算法，然后对ipa进行解密，但是这样的方法难度较大，而且如果人家一旦换了加密方式或者有其他的改动，那解密方式就不生效了，常见的静态砸壳工具有以下

* [fouldecrypt]([NyaMisty/fouldecrypt: A lightweight and simpling iOS binary decryptor (github.com)](https://github.com/NyaMisty/fouldecrypt))
* [Iridium](https://github.com/Lakr233/Iridium)

### 动态砸壳

与静态相反，动态砸壳就是依赖运行时的原理来进行解密，不过与其说是解密，倒不如说是内存提取，因为无论ipa包用什么加密方式，最终都是解密后运行到内存里面的，所以我们可以认为`一个ipa在内存上的数据是未加密的`，所以此时我们只要把内存上的数据提取出来即可，整个过程也不涉及到解密操作，及时后面Apple更换加密方式，也不影响动态砸壳的过程。



动态砸壳的方式和工具有很多，现在基本已经流水线化了，可以使用以下方式和工具来进行处理，前提是要有一个越狱的手机。

* [dumpdecrypted](https://juejin.cn/post/6844903893499904008)

* [Clutch](https://juejin.cn/post/6844903893499904014)

### 成果检验

砸壳后需要检查是否砸壳成功，找到对应砸壳后的的ipa，点进去找到mach-o文件，执行如下命令，然后在输出查看`cryptid`字段如果为`0`就说明砸壳成功。XXX = mach-o名字

```
otool -l XXXXX |grep cry
```

![image-20221114170141723](https://tva1.sinaimg.cn/large/008vxvgGgy1h84ripqxklj30tx02imxg.jpg)



## 头文件导出

砸壳后的的第一步就是将ipa文件的头.h文件导出，然后根据 头文件的方法和属性进行逆向分析，在找到对应的hook点。通常我们使用class-dump，可以去他的官网下载对应的文件，然后将文件拷贝到对应的目录下。

```shell
sudo cp class-dump /usr/local/bin   
```

这一步没什么问题，拷贝完成重启终端就可以调用class-dump的方法了.



### 导出

执行下面的命令，导出头文件，需要注意的是：导出后会有上万个个文件，所以目标目录最好不要选Desktop或者其他的根目录

```shell
class-dump -S -s -H XXXXX -o /path/to/headers/
```

有的时候会收到这样的错误

```shell
Error:Cannot find offset for address 0xd80000000101534a in stringAtAddress:
```

这是因为项目使用了Oc和Swift的混编，需要赋予class-dump文件权限即可

```shell
sudo chmod 777 /usr/local/bin/class-dump
```

之后就可以导出成功了。

## MonkeyDev

这是一个为越狱和非越狱开发人员准备的工具，主要包括四个模块:

* Logos Tweak

  * 使用[theos](https://github.com/theos/theos/wiki/Installation)提供的`logify.pl`工具将`*.xm`文件转成`*.mm`文件进行编译，集成了`CydiaSubstrate`，可以使用`MSHookMessageEx`和`MSHookFunction`来`Hook` OC函数和指定地址。

    

* CaptainHook Tweak

  * 使用[CaptainHook](https://github.com/rpetrich/CaptainHook/)提供的头文件进行OC 函数的Hook以及属性的获取。

    

* Command-line Tool

  * 可以直接创建运行于越狱设备的命令行工具

    

* MonkeyApp
  * 这是自动给第三方应用集成Reveal、Cycript和注入dylib的模块，支持调试dylib和第三方应用，支持Pod给第三放应用集成SDK，只需要准备一个砸壳后的ipa或者app文件即可。

### 安装

Monkeydev依赖[Theos](https://theos.dev/docs/installation).Theos是一个越狱开发工具包，由iOS越狱界知名人士 Dustin Howett 开发并分享到 GitHub 上。Theos 与其他越狱开发工具相比，最大特点就是简单：下载安装简单、Logos语法简单、编译发布简单，可以让使用者将精力都放在开发工作上去。

#### 安装Thoes

```shell
sudo git clone --recursive https://github.com/theos/theos.git /opt/theos
```

#### 安装Monkeydev

```shell
sudo /bin/sh -c "$(curl -fsSL https://raw.githubusercontent.com/AloneMonkey/MonkeyDev/master/bin/md-install)"
```

#### 卸载Monkeydev

```shell
sudo /bin/sh -c "$(curl -fsSL https://raw.githubusercontent.com/AloneMonkey/MonkeyDev/master/bin/md-uninstall)"
```

#### 更新Monkeydev

```shell
sudo /bin/sh -c "$(curl -fsSL https://raw.githubusercontent.com/AloneMonkey/MonkeyDev/master/bin/md-update)"
```

#### 安装问题

在安装过程中，修改用户 `profile` 文件时，找不到 `MacOSX Package Types.xcspec` 和 `MacOSX Product Types.xcspec` 文件

```shell
File /Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/Library/Xcode/Specifications/MacOSX Package Types.xcspec not found

```

这个是因为最新的Xcode14中 这个路径已经改变，所以原路径无法找到，不过如果大家需要逆向的事iOS的App到这一步可以不用关心，这个是MacOS相关的模板文件。此时打开Xcode如果有以下模版文件，并能成功创建工程即可。

![image-20221114173807449](https://tva1.sinaimg.cn/large/008vxvgGgy1h84skm4dlej30li0fbjsb.jpg)



#### 编译报错

通过上一步的模板文件创建好工程后，直接真机编译运行，这个时候会提示编译错误

```
iOS file not found: /usr/lib/libstdc++.dylib
```

这是因为`Xcode 10`之后删除的`libstdc++`库。可以参考此[解决方案](https://github.com/dbmz502/MonkeyDev_Xcode14)。之后就可以编译成功了，并且手机上可以跑起来。



第二个错误是Fishhook中的错误，这个是是由于Fishhook用的是比较老的版本，本身存在bug，只要去github官网找到fishhook最新代码 copy过来即可。



### 文件结构

文件结构如下如图

![image-20221114174320781](https://upload-images.jianshu.io/upload_images/1609369-0a4b973075bc5df2.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/854/format/webp)

这是一个标准的MonkeyDemo的结构

- `TargetApp`：放目标`ipa`的文件，将需要逆向的破壳`ipa`放在此处

  

- `Logos`：编写相关`hook`的文件，所有`hook`操作在此处，但是因为该文件下要用了logos语句，有一定的学习成本，所以后面的hook函数可以直接写在上面的MonkeyDeomDyLib.m中

  

- `fishhook`：用来`hook`系统函数的库

上方的MonkeyDeomDyLib就是我们即将注入进去的动态库。



### 动态库注入

运行demo后动态库注入成功，控制台会有如下输出

![image-20221114174950556](https://tva1.sinaimg.cn/large/008vxvgGgy1h84swti61vj31le0qn43z.jpg)

```
               🎉!!！congratulations!!！🎉\n👍----------------insert dylib success----------------👍
```

但是如果是和我一样的运行环境，你是大概率看不到的，因为会注入失败。这里尝试了两种方式 

* insert_dylib 同样注入失败，
* optool 注入成功

下面说下optool使用

* 下载编译optool

  ```
  git clone https://github.com/alexzielenski/optool.git
  cd optool
  git submodule update --init --recursive
  ```

  

* 找到编译产物

  ![image-20221114175525945](https://tva1.sinaimg.cn/large/008vxvgGgy1h84t2mlrmaj30aa0b7wer.jpg)

* 把编译产物拷贝到`/opt/MonkeyDev/bin`下

* 修改`/opt/MonkeyDev/Tools/pack.sh`

  ```shell
  顶部插入
  OPTOOL="${MONKEYDEV_PATH}/bin/optool"
  
  同上面一样
  修改插入动态库工具代码
  "$OPTOOL" install -c load -p "@executable_path/Frameworks/lib""${TARGET_NAME}""Dylib.dylib" -t "${BUILD_APP_PATH}/${APP_BINARY}"
  ```

* 然后保存重新运行即可注入成功

### pod使用

在调试App时候我们会用到类似lookIn或者FLEX的等工具来看App 的层级结构和 沙盒文件，同样需要pod来接入。

* 像平时创建podfile文件一样 进入到工程目录`pod init`

* 在生成的podfile中添加pod，但是要注意是在<font color='red'> DemoLib </font>的trarget中添加，因为我们的pod是打入动态库的，然后由动态库带入App

  ```
  # Uncomment the next line to define a global platform for your project
  # platform :ios, '9.0'
  
  target 'Demo' do
    # Comment the next line if you don't want to use dynamic frameworks
    use_frameworks!
  
  
  end
  
  target 'DemoDylib' do
    # Comment the next line if you don't want to use dynamic frameworks
    use_frameworks!
  
      pod 'FLEX'
  
      pod 'LookinServer'
  
  end
  ```

* 然后`pod install` 即可看到效果

  ![image-20221115102451908](https://tva1.sinaimg.cn/large/008vxvgGgy1h85lo8xilij31b90u0ada.jpg)

  

### 代码Hook

通过Lookin 我们可以找到入手点和对应的类名，然后通过之前导出的头文件可以查看类名对应的函数，接下来就是要看下函数里面做了哪些事情，就要用到Hook手段，MonkeyDev给我们封装好了Hook相关的方法，包括OC和C的Hook函数

* CHDeclareClass 

  注册类名。也就是注册要被hook的函数所在的类，比如

  ```objective-c
  CHDeclareClass(MYViewController)
  ```

* CHOptimizedMethod1

  hook实例方法，你会发现后面跟了数字1~10，代表被hook 的函数的参数的个数，比如我将要hook的函数只有一个参数 那么就使用CHOptimizedMethod1参数含义为

  * 第一个参数，一般传self

  * 第二个参数，传返回值类型，没有返回值就是void

  * 第三个参数，函数所在的类名

  * 第四个参数，方法名

  * 第五个参数，函数参数的类型

  * 第六个参数，函数参数的变量

    其中第五第六个参数在CHOptimizedMethod1 ~10 中会重复1~10次

    ```objective-c
    CHOptimizedMethod1(self, void, MYViewController, appMethod, id, para) {
        NSLog(@"appMethod被Hook = %@", para);
    }
    ```

    

* CHOptimizedClassMethod3

  hook类方法，所有函数定义同上



* CHConstructor结构

  用来注册刚才的hook操作

  ```objective-c
  CHConstructor{
    // 注册将要hook的类
      CHLoadLateClass(MYViewController);
  	// 注册将要hook 的方法
      CHHook1(MYViewController, appMethod);
  
  }
  ```

  上面流程执行完成后就可以看到函数被Hook了



## Hopper Disassembler

上面的步骤讲了如何通过lookin或者reveal等工具来定位类名，然后通过类名在头文件中找到函数名，然后通过hook手段来改变函数的一些表现，但是在如何没有拿到.m文件的前提下看到某个函数的实现呢？比如一个函数中都做了哪些操作，调用了哪些其他函数，以及调用链是怎样的？



这个时候就需要用到`Hopper Disassembler`或者`IDA Pro`这样的工具了，不过目前遇到的困难是在笔者的系统环境下，这两个软件的破解版无法安装，而且`IDA Pro`的官方试用版还不支持Arm的汇编，所以只能使用Hopper Disassembler来举例子。打开软件，将`对应App 的Mach-o`文件拖入Hopper中等待它分析完成

![image-20221115104755241](https://tva1.sinaimg.cn/large/008vxvgGgy1h85mc4b62oj31ec0u075n.jpg)



处理完后的界面左边会显示方法名，支持搜索查询，中间区域显示的是汇编代码，我们搜索一个在之前dump出的头文件中的一个函数名试下。

```objective-c
[XXXXXXXX listenerDownloadLyricWithSongId:resultBlock:]
```

可以看到中间的部分显示出来函数所对应的汇编代码

![image-20221115193733635](https://tva1.sinaimg.cn/large/008vxvgGgy1h861n7q3isj317b0u0dlg.jpg)

然后按快捷键`Option+enter`即可转为伪OC代码,虽然包含一些的寄存器信息，但是也足以分析了。同时双击可以跳转到对应的函数内部。



## 最后

以上就是目前的逆向调研过程，这里先记录下，后面还会深入研究，有新的发现会同步更新此文章。


