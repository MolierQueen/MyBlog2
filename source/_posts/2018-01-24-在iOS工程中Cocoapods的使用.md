---
title: 在iOS工程中Cocoapods的使用
categories: iOS开发
date: 2018-01-24 12:42:28
tags:
  - Cocoapods
  - 架构
comments:
---
我们在开发iOS程序的时候，往往都会根据需要导入很多的第三方框架，但是不同的框架完成的功能不同，所以导入的方式也不同，并不是把它直接拖进工程中就完事了，我们需要配置各种环境，链接各种库文件等等。有的时候我们并不能一个不漏的完成导入，而一旦出了问题，尤其是那些需要框架比较多的工程(比如早期的百度地图框架)，将很难解决，而且，如果遇到了第三方库升级，更新了方法那么我们还需要把之前的旧版本删掉，再重复一下前面的工作，这将是非常的繁琐，极大地影响了开发的效率。这个时候我们就需要用到cocoapods来管理我们的第三方了，在我们有了CocoaPods这个工具之后，只需要将用到的第三方开源库放到一个名为Podfile的文件中，
<!--more-->
然后在命令行执行$ pod install命令。CocoaPods就会自动将这些第三方开源库的源码下载下来，并且为我的工程设置好相应的系统依赖和编译参数，不管是导入还是更新或者移除，都是一句命令就搞定的。网上也有类似的教程，但是有些很旧，有些写的不详细，导致新手在使用的时候整的一头雾水，我就来说下。
### 第一步 ：
首先安装cocoapods要在ruby环境下进行，虽然我们的mac系统都是自带了ruby，但是为了保险起见我们还是要先更新一下ruby环境：在这里我们直接使用   sudo gem update --system   命令来更新，网上有的说使用 gem update --system 前面少了sudo，其实加sudo的目的就是用管理员的权限去执行这句更新命令，不加的话容易出现这个错误

![](https://wx4.sinaimg.cn/large/006tNc79gy1fo6jyxwn6cj30ta06kn1a.jpg)

意思是你没有权限去执行这个命令，等出现了RubyGems system software updated 这句话的时候就证明升级成功了。

### 第二步：
安装cocoapods时候我们要访问cocoapods.org这个网站，不用想这个网站已经被墙了，所以我们可以用淘宝的ruby的镜像来访问该网站。首先我们输入 gem sources -l 来看一下我们现在有什么，我目前里面只有一个

![](https://wx3.sinaimg.cn/large/006tNc79gy1fo6jzdqmrlj30ma03ydjy.jpg)

也就是我们需要的，不过可能有些人的里面不止一个，会有其他的东西，这时候我们先用gem sources --remove XXXXXXXXXXXXXXX 来把其他的source删除掉，只保留这一个，如果没有的话就手动添加用这个命令 gem sources -a https://ruby.taobao.org/ 来将我们需要的源添加进去，最后再用那个查看命令 最后只有确保像我里面一样只有那一个就好，要注意的是 https  网上好多教程写的是 http，那个已经作废了

![](https://wx4.sinaimg.cn/large/006tNc79gy1fo6jzxkmf5j31kw0ask74.jpg)
### 第三步：
安装是cocoapods 使用 sudo gem install cocoapods 命令来安装cocoapods，你输入完这个命令回车后会提示你输入密码，这时候是没有光标提示的，也不会移动，凭感觉输入对之后就回车吧，然后就是等，时间长短是根据你的网速来的。安装完成后终端就进入待命阶段了。
### 第四步：
使用search命令来搜索类库，这个是支持模糊搜索的，记不清全名，打一部分名也行，不过那样的就要从搜出来的东西里找你想要的类库了。比如我想找afnetworking 我就输入 pod search afn 回车后就会输出所有以afn开头的类库名字，像这样

![](https://wx1.sinaimg.cn/large/006tNc79gy1fo6k0kkcjjj311u0wq4qp.jpg)

搜出很多，其中第三就是我们想要的，afnetworking，用红圈圈起来是一会编辑podfile时候需要用的，可以先把他复制下来省的手打，后面的小数代表的是类库的版本，一般是最新的显示在这里，下面的version是历史版本，如果有需要可以直接导入它的历史版本，就是把后面的版本号替换下就好。
###### 值得注意的是如果你不是第一次安装cocoapods， 那么之前的缓存会对你有影响search先清理下缓存 使用这个命之后就可以了 rm ~/Library/Caches/CocoaPods/search_index.json

### 第五步：
进入你的工程目录，这里建议直接右键你工程中.xcodeproj文件选择在终端中打开，然后 在终端中输入命令cd ..  就会跳到.xcodeproj所在的目录，也就是我们需要的目录，很多新手在这个地方容易出错。然后输入命令 vim Podfile熟悉Linux的用户都知道这是创建一个Podfile文件并打开编辑，按 “i” 进入编辑模式，将第五部粘贴的东西拷贝进来，然后依次操作esc键  ->  ":"  ->  输入wq

![](https://wx4.sinaimg.cn/large/006tNc79gy1fo6k16wn0yj311s0wqth2.jpg)

然后输入命令来安装 pod update --verbose --no-repo-update 等待过后就安装完成啦，其实使用pod install也可以，只是后者需要更新一个仓库，这是相当耗时的，我们可以使用前者来避免更新仓库就好，很快就结束了。

![](https://wx1.sinaimg.cn/large/006tNc79gy1fo6k1nkijrj311u0wq1g6.jpg)

安装成功！以后打卡工程就直接打开这个文件就好啦

![](https://wx1.sinaimg.cn/large/006tNc79gy1fo6k1y1x80j306q0cb0sy.jpg)

其中podfile文件中显示了我们这个工程中所以集成的第三方，

![](https://wx2.sinaimg.cn/large/006tNc79gy1fo6k27zcbxj30ln0bfmxn.jpg)

想修改版本的话就把后面的版本号改成你所需要的版本号就好，想删除的话就把这行删掉，想添加的话就用第五部的搜索命令去搜索然后同样把搜索结果中以pod开头的那句话复制进来就好。注意以上所有的增删改操作完成之后需要在去终端中相应的目录下使用 pod install --verbose --no-repo-update 命令来更新，这样才会真正的生效。

### 第六步：
关于cocoapods的更新。有的时候在pod install的时候会出现[!] The 'master' repo requires CocoaPods 0.32.1 - 这样的错误，是由于你cocoapods版本过低的原因，这时候需要进行更新，跟新的过程其实就是把以上所有的从新走一遍就相当于安装遍就好了。                                  
##### 值得注意1         
经常遇到的错误比如下面这个

![](https://wx3.sinaimg.cn/large/006tNc79gy1fo6k2hpy06j30ej01gt8q.jpg)

通常出现在OS X 10.11系统上 这是由于从这个系统开始苹果开始使用无根安装，这时你再用这个方法就会报这个错，这时只需

![](https://wx4.sinaimg.cn/large/006tNc79gy1fo6k2pt7n9j30hy02zwev.jpg)

这个命令就可以成功升级啦      

##### 值得注意2
有的时候大家在pod search的时候搜不到，但是明明有这个类库别人都可以都到课时就是自己搜不到，其实原因是这样的：pod search只会搜索你本地缓存的框架，如果你想搜索到最新的第三方框架或者某个框架的最新版本，必须先使用pod repo update（推荐）或者pod setup将远程仓库的框架信息更新到本地。其实，从pod search的响应速度飞快，也可以猜出它并没有连接服务器，仅仅是搜索了本地的框架信息[呵呵]
    此外，如果你的框架更新比较慢，可以尝试执行下面2条指令更换镜像服务器
1：pod repo remove master
2：pod repo add master http://git.oschina.net/akuandev/Specs.git
    更换镜像完毕后，以后执行pod repo update的速度就会快很多。在说明一点上面两条指令如果第二条无法执行提示403错误像这样

![](https://wx3.sinaimg.cn/large/006tNc79gy1fo6k2xgrwuj30i101h3yg.jpg)

那么在执行完第一条之后直接pod search 命令就好 这样他会自动找合适的配置了，因为第二条那个网址可能会变。
### 总结：
关于使用cocoapods在自己的项目中集成第三方就这些内容。有什么不懂的欢迎来找我交流，本人才疏学浅，如果那里写的不对请及时批评指正，免得误导新人。 新浪微博小耗子上桌子也是我，大家也可以去看下微博多多关注 谢谢！
