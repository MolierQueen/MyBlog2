---
title: How to create Swift CLI
sticky: true
categories: iOS开发
cover: https://cdn.jsdelivr.net/gh/MolierQueen/resource@main/202304081654215.png
date: 2023-03-28 17:42:10
tags:
comments:
---
## 为什么要做CLI

命令行工具**CLI(command-line interface)** 是开发者必不可少的工具之一，编写命令行工具来处理一些工作上的事情也是开发者必备的技能。对于一些重复性的工作，使用命令行脚本可以将任务自动化，这将极大的提升我们的工作效率。同时也能避免因为人为的因素导致错误。比如批量处理一些文件、表格等，在APP开发中**持续集成**的概念也是建立在一些列的自动化命令行脚本的基础之上。

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

### 3：参数传递与处理

#### 方式1：系统API解析参数

上面讲述工程创建和命令行编写，通常我们在调用命令行的时候会带有参数

```shell 命令参数
$ command 参数1 参数2 参数3 ....
```

同样在代码层面也有解析参数的API

```swift CommandLine
/// Command-line arguments for the current process.
@frozen public enum CommandLine {

    /// Access to the raw argc value from C.
    public static var argc: Int32 { get }

    /// Access to the raw argv value from C. Accessing the argument vector
    /// through this pointer is unsafe.
    public static var unsafeArgv: UnsafeMutablePointer<UnsafeMutablePointer<Int8>?> { get }

    public static var arguments: [String]
}
```

使用方式比较简单

```swift 解析参数
// 解析外部传进来的参数
let arguments = CommandLine.arguments

// 第一个参数
let firstArg = arguments[1]

// 第二个参数
let secondtArg = arguments[2]
print("My args = \(arguments)  first = \(firstArg)  second = \(secondtArg)");
```

需要注意这里返回的参数数组中第一个元素是可执行文件本身路径，然后用户真正的输入的第一个参数是从第二个元素开始，类似与iOS中`objcMsgSend`函数，其中第一个参数是self。然后可以通过解析这些参数来达到不同的目的

![](https://cdn.jsdelivr.net/gh/MolierQueen/resource@main/202304031744585.png)



#### 方式2：使用SwiftArgumentParser

在实际使用中，一个完善的命令行参数一定不会这么简单，而且我们在解析参数的时候也不知道使用方传入参数的顺序，一些简单的命令，或者只有一个参数的情况下可以使用CommandLine 的API，更复杂的情况下需要时用[SwiftArgumentParser](https://swiftpackageindex.com/apple/swift-argument-parser/1.2.2/documentation/argumentparser/gettingstarted)来进行处理。

`SwiftArgumentParser`是苹果开源的一个用Swift编写的参数解析器，用于解析命令行参数（command-line arguments），具有直观、易用、简洁、安全的特点。虽然是苹果自己开发的，但是毕竟还是外部库需要使用`Swift package`打包进来，对`Package`文件进行编写

```swift Package.swift
import PackageDescription
let package = Package(
    name: "CLIDemo",
    dependencies: [
        //引入swift-argument-parser解析器
        .package(url: "https://github.com/apple/swift-argument-parser", from: "1.2.0"),
    ],
    targets: [
        .executableTarget(
            name: "CLIDemo",
            dependencies: [
                //将解析器依赖到target
                .product(name: "ArgumentParser", package: "swift-argument-parser"),
            ]),
        .testTarget(
            name: "CLIDemoTests",
            dependencies: ["CLIDemo"]),
    ]
)
```



同时`CLIDemo.swift`代码文件也要相应的进行修改

* 1：引入**ArgumentParser**
* 2：将struct改为Class（方便后续的开发），并遵循`ParsableCommand`协议
* 3：修改main函数为run：因为遵循协议后，原来的main被`ParsableCommand`接管入口，内部会调用函数名为`run`的函数作为入口。

```swift CLIDemo.swift
import ArgumentParser
@main
class CLIDemo: ParsableCommand {
    required init() {
    }
    func run() {
        // 解析外部传进来的参数
        let arguments = CommandLine.arguments
        // 第一个参数
        let firstArg = arguments[1]
        // 第二个参数
        let secondtArg = arguments[2]
        print("My args = \(arguments)  first = \(firstArg)  second = \(secondtArg)");
    }
}
```

工程修改完后已经具备了`ArgumentParser`的开发环境，ArgumentParser的参数分为三类

* @Argument：无标记位参数，与上面介绍的直接使用CommandLine的API解析方式相似，该类型的参数没有别名标记位，而且必须按照用户传入的顺序做解析。
* @Option：带有标记位参数，这个类型的参数就是通过别名或者标记为来标识的，也是我们常见的参数用法比如 `-n myName`或者`--name myName`。其中`-n`和`--name`就是该参数的长别名和短别名，同样因为有了别名，所以解析时候不用关系用户输入参数的顺序。
* @Flag：标记位，是一个bool变量，比如常用 `--verbose`，`-h`等

```swift 参数解析
static var configuration = CommandConfiguration(abstract: "这是一个测试Demo")
@Argument(help: "这是一个Argument 参数")
var argumentArg: String = "Argument"

@Option(name: [.short, .long], help: "这是一个option参数")
var optionArg: String = "option"

@Flag(name: [.short, .long], help: "这是一个Flag参数")
var flagArg: Bool = false
```

关于参数的描述系统提供以下定义，通常使用`short`和`long`

```swift 参数定义
internal enum Representation: Hashable {
  case long // 参数原标记位，就是变量名
  case customLong(_ name: String, withSingleDash: Bool)  // 自定义标记位
  case short // 参数短标记位  为-加上变量名第一个字母
  case customShort(_ char: Character, allowingJoined: Bool) // 自定义短标记位
}
```





`ArgumentParser`默认集成了`-h`参数，完成以上参数定义后，通过`-h`输出我们命令行Demo帮助文档

![](https://cdn.jsdelivr.net/gh/MolierQueen/resource@main/202304032145234.png)

:::warning

注意：参数命名时候如果使用驼峰结构，最终的参数会被添加`-`比如上面的我定义的`flagArg`，最终命令行的Flag参数为`--flag-arg`。所以这里尽量不用驼峰结构。

:::

### 4：调试运行

因为我们是在Xcode中编程开发，所以不用每次都跑到命令行中取执行`Swift run CLIDemo XXX`来编译运行我们的工具，这样不然切来切去影响工作效率，而且没法使用断点调试，正确的方式是像正常的iOS开发一样直接在Xcode中编译运行。而参数传递可以在Xcode上方的`Edit Scheme`中处理

![](https://cdn.jsdelivr.net/gh/MolierQueen/resource@main/202304032152232.png)

![](https://cdn.jsdelivr.net/gh/MolierQueen/resource@main/202304032153290.png)

然后编译运行即可，以上行为等价于在终端中输入`swift run CLIDemo arg1 -f -o option`



### 5：编译成可执行文件

我们假设已经完成了命令行程序的编写，最终要达到的目是执行我们的命令行程序然后输出Hello World!，那么首先我们需要把代码编译成可执行文件，通过如下命令

```shell 编译成可执行文件
& swift build -c release
```

编译之后我们可以在工程目录下找到我们产物

![](https://cdn.jsdelivr.net/gh/MolierQueen/resource@main/202304011810603.png)



这样一来我们就可以把该文件进行分发，让其他人或者服务器端使用我们的的命令行工具了，如果有需要可以把该文件放到/usr/local/bin/ 目录下，这样可以在任意路径下使用

![](https://cdn.jsdelivr.net/gh/MolierQueen/resource@main/202304011817712.png)

## Swift CLI实战(iPa下载器)

上面讲了一个Swift CLI 工具从开发到使用的完整流程，但是一个真正的命令行工具一定不仅仅是输出一个Hello World，需要有`子命令`、`公共参数`、`二次输入`，`敏感输入`，`终端输出样式`, `进度回调`等功能。本节内容会通过实现一个ipa下载器，来介绍下Swift CLI的一些进阶用法，这些用法几乎能覆盖之后百分之九十的工作场景。

一个iPa下载器可以从Appstore下载App，同时集成了Appstore相关能力，如`登录`,`搜索`,`下载`等。我们可以将这些能力封装成不同的子命令来进行调用，像如下这样

![](https://cdn.jsdelivr.net/gh/MolierQueen/resource@main/202304062017381.png)



### 1：子命令

子命令也是`ArgumentParser`的能力项之一，可以在[这里](https://swiftpackageindex.com/apple/swift-argument-parser/1.2.2/documentation/argumentparser/commandsandsubcommands)查看官方文档，具体代码

```swift 子命令
static var configuration = CommandConfiguration(abstract: "一个iPa下载工具", subcommands: [Search.self, Login.self, Download.self])
```

其中要创建子命令对应的`.swift`文件。且每个文件中都应像之前的CLIDemo.swift的结构一样，定义自己的类，且遵循`ParsableCommand`协议，以Search举例，其他`同级`子命令同理，子命令`嵌套`子命令，结构类似，以此类推。

```swift 子命令
class Search: ParsableCommand {
   required init() {
   }
   static var configuration = CommandConfiguration(abstract: "搜索appstore上的App")
   func run() {
   }
}
```



### 2：公共参数

当有多个子命令的时候我们一定会有一些参数是公用的，比如上面展示的`--verbose`，如果每个子命令文件都写一遍显然不现实，所以`ArgumentParser`提供了[OptionGroup选项组](https://swiftpackageindex.com/apple/swift-argument-parser/1.2.2/documentation/argumentparser/optiongroup)的能力。

我们可以在一个公共的类或结构体中定义一系列公用参数,然后在需要使用公共参数的子命令文件中定义`@OptionGroup`如下图。在解析的时候可以用`GlobalOptions.verbose`来取值

```swift 参数组
struct GlobalOptions: ParsableArguments {
    @Flag(name: .shortAndLong)
    var verbose: Bool

    @Argument var values: [Int]
}

class Search: ParsableArguments {
    @Option var name: String
    @OptionGroup var globals: GlobalOptions
}
```



### 3：Appstore登录

首先登录需要输入用户名密码，所以Login文件的参数一定是包含username，password，使用上面提到的方式很容易将这两个参数传入，但是输入密码的时候如果是明文的话就太不安全了，终端输入密码的方式都是`隐式输入`，同样我们的工具也要具备这个能力，使用`getpass`函数可以达到隐式输入图的目的，这样打字就不会显示到终端中，也不用为密码单独分配一个参数。

```
CommonMethod().showCommonMessage(text: "请输入密码：")
  guard let psd = getpass("") else {
 	 CommonMethod().showErrorMessage(text: "需要输入密码")
   Login.exit()
}
```

![](https://cdn.jsdelivr.net/gh/MolierQueen/resource@main/202304062055076.png)

* 登录Api `"https://p25-buy.itunes.apple.com/WebObjects/MZFinance.woa/wa/authenticate?guid=MAC地址"`

### 4：二次输入

拿到用户名和密码可以进行Appstore API请求进行登录了，但是Appstore 是有二次认证的，所以我们还需要输入一个授权码。此时我们可以通过Appstore服务端返回的信息来提示用户输入授权码，需要授权码的错误信息为`MZFinance.BadLogin.Configurator_message`，此时我们的进程还未结束，需要用户`二次输入`,对应的api为`readLine`

```swift 二次输入
CommonMethod().showWarningMessage(text: "请输入双重认证的Code：")
let authCode = readLine();
self.authCode = authCode ?? ""
```

收到用户的授权码后，携带授权码重新请求Appstore API接口即可，



### 5：本地持久化

登录成功后会获得`DSID Token`以及相关Cookie信息，需要把这些信息持久化到本地，避免每次使用该工具都要走登录流程，持久化的方式可以使用数据库、UserDefault、写文件等方式进行，这些对于iOS开发人员来说并不陌生。



### 6：文件搜索

文件搜索比较简单，我们可以通过APP的名字进行搜索，入参为：

* appname ：APP名称
* appid：APP在applestore上的ID（非必要）
* limit：结果条数限制（非必要）
* Country：APP所在国家（非必要）

整个搜索流程为：

![](https://cdn.jsdelivr.net/gh/MolierQueen/resource@main/202304081440040.png)

App 搜索的API为`https://itunes.apple.com/search`

### 7：文件下载

通过上一步拿到的bundleid调用AppStore的下载接口可以实现ipa包的下载，所以这里的入参为：

* bundleid：App的bundleid
* path：下载路径

![](https://cdn.jsdelivr.net/gh/MolierQueen/resource@main/202304081526063.png)

拼接好请求后很容易就进入下载流程开始下载了。在Swift中可以使用系统原生的NSUrlsession或者使用一些开源三方空类似Alamofire、moya等。

### 8：命令行输出样式

在执行下载任务或者一些耗时任务，我们需要提供进度条来给使用者一定的提示，终端中的进度条其实也是通过各种各样的字符编码组成的图案，同时通过不同的颜色来区分不同的状态

* 下载中

  ![](https://cdn.jsdelivr.net/gh/MolierQueen/resource@main/202304081506567.png)

* 下载完成

  ![](https://cdn.jsdelivr.net/gh/MolierQueen/resource@main/202304081508595.png)

* 下载失败

  ![](https://cdn.jsdelivr.net/gh/MolierQueen/resource@main/202304081518284.png)

进度条图案是由两部分组成

* 表示完成字符：█
* 表示剩余字符：░

#### 进度

默认生成50个░，然后每次下载进度回调回来我们会根据百分比把已完成的部分用█替换，这样就展示了类似一个进度条在前进的样式。如果追求精细化，可以根据命令行窗口的宽度来动态调整进度条的长度，避免窗口过小，导致进度条会折行显示。

同时为了保证进度条保持在一行，所以每次展示都要把光标移到开始位置然后在该行重新展示，这里使用\r打头，同时去掉print函数末尾自带的\n 操作。这里封装一个显示进度的函数

```swift 进度条
func showProcess(process:Float, customEnd:String) -> Void {
		//  宽度50
    let barW = 50
    let com = Int(Float(barW)*Float(process))
    let rem = barW - com
		//  自定义结尾文案和颜色
    var endStr = ""
    var color = ""
    if customEnd.count > 0 {
      endStr = customEnd
    }

  	//  下载完成样式
    if com == 50 {
      endStr = "下载完成：【100%】"
      color = "\u{001B}[0;32m"
    }

	  // 进度条
    let bar = String(repeating: "█", count: com) + String(repeating: "░", count: rem)
  
    // 打印进度条
    print("\r\(color)\(bar) \(endStr)", terminator: "")
  
  	// 刷新输出缓冲区
    fflush(stdout)
}
```



#### 颜色

在文本前添加相应的编码可以更改文本的样式，比如

* 文字颜色 `\u{001b}[?m`   其中` ? ∈ [30, 37]`。

  ```shell 颜色
  黑（black）：\u{001b}[30m
  红（red）：\u{001b}[31m
  绿（green）：\u{001b}[32m
  黄（yellow）：\u{001b}[33m
  蓝（blue）：\u{001b}[34m
  品红（magenta）：\u{001b}[35m
  蓝绿（cyan）：\u{001b}[36m
  白（white）：\u{001b}[37m
  还原初始（reset） ：\u{001b}[0m
  ```

  

* 文字背景颜色`\u{001b}[?m`，其中 `? ∈ [40, 47]`。

  ```shell 背景
  黑（black）：\u{001b}[40m
  红（red）：\u{001b}[41m
  绿（green）：\u{001b}[42m
  黄（yellow）：\u{001b}[43m
  蓝（blue）：\u{001b}[44m
  品红（magenta）：\u{001b}[45m
  蓝绿（cyan）：\u{001b}[46m
  白（white）：\u{001b}[47m
  ```

  

* 字体样式

  ```shell 字体
  加粗加亮：\u{001b}[1m
  降低亮度：\u{001b}[2m
  斜体：\u{001b}[3m
  下划线：\u{001b}[4m
  反色：\u{001b}[7m
  ```

以上命令可以单独使用，也可以组合使用，如将以下条件组合在一起

* \u{001b}[1m ：加粗加亮
* \u{001b}[4m：下划线
* \u{001b}[42m：绿色背景
* \u{001b}[31m：红色字体

```swift 示例文字
print("\n\u{001b}[1m\u{001b}[4m\u{001b}[42m\u{001b}[31m 这是一段绿色背景红色字体加粗带有下划线的文字")
```

![](https://cdn.jsdelivr.net/gh/MolierQueen/resource@main/202304081605787.png)



## 总结

做完以上操作后一个ipa下载器就完成了，具体的源码可以查看[此处](https://github.com/MolierQueen/MyIpaManager)。使用Swift编写CLI可以极大的提高iOS开发者的开发效率，降低脚本语言的学习成本，同时随时Apple对Swift的不断更新迭代，未来也许能用swift做更多的事情。
