---
title: How to create a Swift Command Line tool
categories: iOS开发
date: 2023-03-28 17:42:10
tags:
comments:
---
## 为什么要做Cli

命令行工具**Cli(command-line interface) 是开发者必不可少的工具之一，编写命令行工具来处理一些工作上的事情也是开发者必备的技能。对于一些重复性的工作，使用命令行脚本可以将任务自动化，这将极大的提升我们的工作效率。同时也能避免因为人为的因素导致错误。比如批量处理一些文件、表格等，在APP开发中**持续集成**的概念也是建立在一些列的自动化命令行脚本的基础之上。

![](https://cdn.jsdelivr.net/gh/MolierQueen/resource@main/111.png)

## 为什么使用Swift

命令行的开发语言有很多，比如最基本的`shell脚本`,还有常见的`Python`、`Go`、`Ruby`等。而iOS开发者工作中最常用也是最熟悉的就是Objective-c和Swift语言，所以这会带来几个个问题。

* App开发者编写脚本语言需要一定的学习成本。
* App开发者编写脚本时候需要频繁的进行语言切换。
* Swift作为苹果主推的语言，iOS相关的脚本都已经使用Swift开发，比如一些`重签`、`动态库注入`脚本等。Swift开发的命令行程序不但能像正常的脚本语言一样完成各种批处理任务，而且对iOS的项目有天然的优势。



虽然`Objective-c`编写的代码也可以通过`clang XXXXXX.m -framework Foundation -o XXXXX` 来编译成可执行文件来使用，但是毕竟Objective-c一开始设计的初衷并不包含命令行程序，所以一些功能上还在存在不少缺陷，比如参数的传入与处理。而Swift在官方一推出的时候就宣传了Swift可以开发App 、脚本、后台服务、前端等，应用广泛。



## Swift CLI基本流程

### 1：工程创建

使用[Swift Package Manager（SPM）](https://www.swift.org/package-manager/)来创建工程，SPM是苹果官方提供的一个用于管理源代码分发的工具，类似于 Cocoapods或者Carthage，但是更轻量化，并且Xcode原生支持，无需配置各种环境，可以直接使用。

```shell 工程创建
$ cd CLIDemo  // 进入到你的文件夹
$ swift package init --type executable
```



执行完命令后会生成所需要的文件

![](https://cdn.jsdelivr.net/gh/MolierQueen/resource@main/202304011746092.png)

其中

* Package.swift ：类似Cocoapods中的 Podfile文件，里面描述了一些库的引用依赖关系，和工程配置。

* Source/CLIDemo文件夹：我们的工程目录，后续我们新加源代码或者文件都放到该目录下。

* CLIDemo.swift：命令行程序入口，不可更改文件名字，里面包含main函数。

  ```swift 入口main函数
  @main
  public struct CLIDemo {
      public private(set) var text = "Hello, World!"
  
      public static func main() {
          print(CLIDemo().text)
      }
  }
  ```

  

* Tests文件夹：测试工程，与正常的Xcode工程类似。

### 2：使用Xcode开发

工程文件结构创建好之后目前还缺少`XXX.xcodeproj`文件，没办法用Xcode直接打开，使用如下命令创建Xcode入口

```shell Xcode入口创建
$ swift package generate-xcodeproj
```

然后打开生成的`CLIDemo.xcodeproj`文件，将运行设备选择为Mac，然后编译运行后就可以在Xcode的控制台看到输出的Hello World文案，截止到此，我们的整个命令行开发工程就已经搭建完成。

![](https://cdn.jsdelivr.net/gh/MolierQueen/resource@main/202304011801713.png)



同样除了使用Xcode GUI的当时编译运行之外也可以使用命令行方式进行

```shell 编译运行
$ swift run CLIDemo
```

然后得到相同的输出

![](https://cdn.jsdelivr.net/gh/MolierQueen/resource@main/202304011813359.png)

### 3：编译成可执行文件

我们假设已经完成了命令行程序的编写，最终要达到的目是执行我们的命令行程序然后输出Hello World!，那么首先我们需要把代码编译成可执行文件，通过如下命令

```shell 编译成可执行文件
& swift build -c release
```

编译之后我们可以在工程目录下找到我们产物

![](https://cdn.jsdelivr.net/gh/MolierQueen/resource@main/202304011810603.png)



这样一来我们就可以把该文件进行分发，让其他人或者服务器端使用我们的的命令行工具了，如果有需要可以把该文件放到/usr/local/bin/ 目录下，这样可以在任意路径下使用

![](https://cdn.jsdelivr.net/gh/MolierQueen/resource@main/202304011817712.png)

## Swift CLI进阶

上面讲了一个Swift CLI 工具从开发到使用的完整流程，但是一个真正的命令行工具一定不仅仅是输出一个Hello World，需要有`子命令`、`参数`、`标记位`，`二次输入`等
