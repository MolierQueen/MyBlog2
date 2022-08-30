---
title: 关于实现唱吧清唱功能的理解
categories: iOS开发
date: 2018-06-19 14:44:56
tags:
  - 音频
  - 唱吧
  - K歌
  - AVFoundation
comments:
---
## 简介

#### AVFoundation

在iOS上多媒体的处理主要依赖的是AVFoundation框架，而AVFoundation是基于CoreAudio、CoreVideo、CoreMedia、CoreAnimation之上高层框架，在AVFoundation框架之上苹果还提供给我们更高层一些处理媒体数据的框架。

![](https://wx4.sinaimg.cn/large/006tNc79gy1fsgu5859czj30ol0et75h.jpg)
<!-- more -->

如AVKit、iOS的UIKit、OS的AppKit。AVFoundation提供了大量强大的工具集，可通过这个框架处理音视频编程，但是如同苹果中的的Kit一样，封装的越高级，个性化就会困难些，一些实际项目中的奇葩需求难以实现。本章所讲的内容是AVFoundation上层加下层的AVAudioEngine实现。

#### AVAudioEngine

AVAudioEngine是Objective-C的音频API接口，具有低延迟(low-latency)和实时(real-time)的音频功能，并且具有如下特点：

* 读写所有Core Audio支持的格式音频文件

* 播放和录音使用 (files) 和音频缓冲区 (buffers)

* 动态配置音频处理模块 (audio processing blocks)

* 可以进行音频挖掘处理 (tap processing)

* 可以进行立体声音频信号混合和3d效果的混合

* 音乐设备数字接口MIDI 回放和控制，通过乐器的采样器

AVAudioEngine的工作原理可以简单的分为三个部分:

![](https://wx2.sinaimg.cn/large/006tNc79gy1fsgusaq6ttj316208i41t.jpg)

从图中可以看出AVAudioEngine的每一步操作都是一个音频操作节点(Node)，每个完整的操作都包含输入节点和输出节点以及经中间的若干个处理节点，包括但不限于，添加音效、混音、音频处理等。整体的流程和GPUImage的流程差不多，都是链式结构，通过节点来链接成一个完整的流水线，其中每个节点都有自己特有的属性，可以通过改变属性的值来改变经由该节点后的音频输出效果，用音效节点举例：一个声音流通过这个音效节点，假如这个节点可以给该段声音添加一个回响的效果，那么通过该节点特有的属性可以设置回想的间隔、干湿程度等，这样一来经过这个节点处理过的声音流就会变成我们想要的样子，然后他作为为一个输入了再次流入其他节点。上图的Mixer其实是包含若干个这样的音效节点

![](https://wx1.sinaimg.cn/large/006tNc79gy1fsgvxmt21cj310c0dpq4l.jpg)


## 原理

清唱的功能很简单，就是通过麦克风录制声音，然后添加音效或者做一些处理之后再输出，因为不要配乐，所以省略了一大部分操作(添加配乐完整K歌在下期会讲到)，但是有一个问题就是耳返，也叫返送：

![](https://wx1.sinaimg.cn/large/006tNc79gy1fsgv4t4pd6j30n602ujsg.jpg)

这个东西是必不可少的，因为有了耳返你就可以实时调整自己的声音，极大的降低了走调的风险和尴尬，一个很简单的例子，现在有不少人喜欢在水房唱歌或者是洗澡的时候唱歌，原因就是在水房或者是卫生间通常会有回音，而回音就是天然的耳返，所以在有回音的地方唱歌就会感觉自己的声音洪亮而且音准很好(因为你可以实时的通过回音来调整自己的声调)。演唱会上唱歌的人的耳机中都是耳返。而且耳返要有一个要求就是，你所听到的你自己的声音一定要和观众或者是其他的人听到的一样，不然就不会有作用，我们平时自己说话自己能听到是因为声音通过骨传导到达我们的耳朵，而听众听到的是通过空气介质传播，所以是否有耳返直接决定了你演唱质量的好坏。

使用AVAudioEngine来完成这个功能其实就是运用了他的实时音频的特点，他可以几乎在没有延迟的情况下同时创建音频的输入和输出，而且对这个做了高度的封装使我们能更加关心音效调整

## 实现

###### 创建音频文件用来接收待录制的声音：

```
//创建音频文件。
   NSString * path = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES)[0];
   NSString * filePath = [path stringByAppendingPathComponent:@"123.caf"];
   NSURL * url = [NSURL fileURLWithPath:filePath];
```

###### 创建AVAudioEngine，并打通输入和输出节点：
* 创建AVAudioEngine，并初始化。这里要弄成属性不然会被释放，没有效果

  ```
  @interface ViewController (){

  @property (nonatomic, strong) AVAudioEngine * engine;
  @property (nonatomic, strong) AVAudioMixerNode * mixer;
  @end

  ```

  ```
  self.engine = [[AVAudioEngine alloc] init];
  self.mixer = [[AVAudioMixerNode alloc] init];
  ```

* 打通输入和输出节点：

  ```
  [_engine connect:_engine.inputNode to:_engine.outputNode format:[_engine.inputNode inputFormatForBus:AVAudioPlayerNodeBufferLoops]];

  ```
  所使用的是如下方法。
  ```
  /*!	@method connect:to:format:
  	@abstract
  		Establish a connection between two nodes
  	@discussion
  		This calls connect:to:fromBus:toBus:format: using bus 0 on the source node,
  		and bus 0 on the destination node, except in the case of a destination which is a mixer,
  		in which case the destination is the mixer's nextAvailableInputBus.
  */
  - (void)connect:(AVAudioNode *)node1 to:(AVAudioNode *)node2 format:(AVAudioFormat * __nullable)format;
  ```

* 开启AVAudioEngine:

  该方法可能会开启失败，需要开发者自定去处理

  ```
  [_engine startAndReturnError:nil];

  ```

  以上步骤走完后并且开启成功你就会发现你从耳机里面可以实时的听到你的声音了。

* 音效：

  正常来说光有耳返还不够，因为清唱虽然没有配乐伴奏，但是是支持用户调节音效的，类似于变声。这就用到AVAudioEngine中的AVAudioUnitEffect类。

  ![](https://wx2.sinaimg.cn/large/006tNc79gy1fsgxpydeq8j30hw0cwq32.jpg)

  * 1.AVAudioUnitReverb:混响，混响可以模拟咱们在一个空旷的环境，比如教堂、大房间等，这样咱们在说话的时候，就会有回音，并且声音也比较有立体感。其中该类别下面又分为
  ```
  typedef NS_ENUM(NSInteger, AVAudioUnitReverbPreset) {
    AVAudioUnitReverbPresetSmallRoom       = 0,
    AVAudioUnitReverbPresetMediumRoom      = 1,
    AVAudioUnitReverbPresetLargeRoom       = 2,
    AVAudioUnitReverbPresetMediumHall      = 3,
    AVAudioUnitReverbPresetLargeHall       = 4,
    AVAudioUnitReverbPresetPlate           = 5,
    AVAudioUnitReverbPresetMediumChamber   = 6,
    AVAudioUnitReverbPresetLargeChamber    = 7,
    AVAudioUnitReverbPresetCathedral       = 8,
    AVAudioUnitReverbPresetLargeRoom2      = 9,
    AVAudioUnitReverbPresetMediumHall2     = 10,
    AVAudioUnitReverbPresetMediumHall3     = 11,
    AVAudioUnitReverbPresetLargeHall2      = 12
    } NS_ENUM_AVAILABLE(10_10, 8_0);
  ```
  从名字可以看出是在模拟不同环境下的音效，比如其中的大中小屋子，大厅等。

    该类别可以自定义的属性是wetDryMix，就是可以让我们的声音更空灵。

    ```
      /*! @property wetDryMix
        @abstract
        Blend of the wet and dry signals
        Range:      0 (all dry) -> 100 (all wet)
        Unit:       Percent
      */
    @property (nonatomic) float wetDryMix;
    ```

    可以通过如下方式创建AVAudioUnitReverb
    ```
    AVAudioUnitReverb * reverd = [[AVAudioUnitReverb alloc] init];
    reverd.wetDryMix = 100;
    [reverd loadFactoryPreset:AVAudioUnitReverbPresetLargeRoom];
    ```

  * 2.AVAudioUnitEQ:均衡器，咱们可以使用均衡器来调节咱们音频的各个频段，比如，我想让我的低音更加浑厚，我就可以调节EQ的20-150HZ的频段，如果你想让你的声音更加明亮，那可以调节500-1KHZ的频段,这个调节涉及到一些专业方面的知识，如果只是想让用户去使用的话，可以用苹果给我们更封装好的几个效果即可，这个就类似于photoshop和美图秀秀的区别。

    ```
    typedef NS_ENUM(NSInteger, AVAudioUnitEQFilterType) {
      AVAudioUnitEQFilterTypeParametric        = 0,
      AVAudioUnitEQFilterTypeLowPass           = 1,
      AVAudioUnitEQFilterTypeHighPass          = 2,
      AVAudioUnitEQFilterTypeResonantLowPass   = 3,
      AVAudioUnitEQFilterTypeResonantHighPass  = 4,
      AVAudioUnitEQFilterTypeBandPass          = 5,
      AVAudioUnitEQFilterTypeBandStop          = 6,
      AVAudioUnitEQFilterTypeLowShelf          = 7,
      AVAudioUnitEQFilterTypeHighShelf         = 8,
      AVAudioUnitEQFilterTypeResonantLowShelf  = 9,
      AVAudioUnitEQFilterTypeResonantHighShelf = 10,
      } NS_ENUM_AVAILABLE(10_10, 8_0);
    ```

    上面是一些苹果帮助我们定义好的滤波器，比如低通滤波器 衰弱高频、可以引发共鸣的 低通滤波器
  不过一般在清唱的时候这个用处不大，这个效果主要用到在配合伴奏的时候，如果伴奏音调过高，可以使用该方法适当的提高人声音调或者降低伴奏的音调，

    可以通过如下方式使用,然后更改这个节点一些属性值。
    ```
    AVAudioUnitEQ * eq = [[AVAudioUnitEQ alloc] initWithNumberOfBands:1];
    AVAudioUnitEQFilterParameters * filter = eq.bands.firstObject;
    filter.filterType = AVAudioUnitEQFilterTypeResonantHighShelf;
    filter.bandwidth = 10;
    filter.gain = 20;
    ```

  * 3.AVAudioUnitDistortion：失真，这个就是我们常说的电音，一般说唱或者摇滚，重金属之类的曲风会用到这个效果，同样苹果给我们提供了预设的几个效果，如果不是有专业的需求我们可以直接使用。

    ```
    typedef NS_ENUM(NSInteger, AVAudioUnitDistortionPreset) {
      AVAudioUnitDistortionPresetDrumsBitBrush           = 0,
      AVAudioUnitDistortionPresetDrumsBufferBeats        = 1,
      AVAudioUnitDistortionPresetDrumsLoFi               = 2,
      AVAudioUnitDistortionPresetMultiBrokenSpeaker      = 3,
      AVAudioUnitDistortionPresetMultiCellphoneConcert   = 4,
      AVAudioUnitDistortionPresetMultiDecimated1         = 5,
      AVAudioUnitDistortionPresetMultiDecimated2         = 6,
      AVAudioUnitDistortionPresetMultiDecimated3         = 7,
      AVAudioUnitDistortionPresetMultiDecimated4         = 8,
      AVAudioUnitDistortionPresetMultiDistortedFunk      = 9,
      AVAudioUnitDistortionPresetMultiDistortedCubed     = 10,
      AVAudioUnitDistortionPresetMultiDistortedSquared   = 11,
      AVAudioUnitDistortionPresetMultiEcho1              = 12,
      AVAudioUnitDistortionPresetMultiEcho2              = 13,
      AVAudioUnitDistortionPresetMultiEchoTight1         = 14,
      AVAudioUnitDistortionPresetMultiEchoTight2         = 15,
      AVAudioUnitDistortionPresetMultiEverythingIsBroken = 16,
      AVAudioUnitDistortionPresetSpeechAlienChatter      = 17,
      AVAudioUnitDistortionPresetSpeechCosmicInterference = 18,
      AVAudioUnitDistortionPresetSpeechGoldenPi          = 19,
      AVAudioUnitDistortionPresetSpeechRadioTower        = 20,
      AVAudioUnitDistortionPresetSpeechWaves             = 21
  } NS_ENUM_AVAILABLE(10_10, 8_0);
    ```
    其实里面有些还是比较有感觉的，比如扭曲的立方体，或者外星人的喋喋不休等。有兴趣的可以说都试试

    使用方式同之前的效果一样

    ```
    AVAudioUnitDistortion * dist = [[AVAudioUnitDistortion alloc] init];
    [dist loadFactoryPreset:AVAudioUnitDistortionPresetDrumsBitBrush];
    dist.preGain = 4;
    dist.wetDryMix = 100;
    ```

  * 4.AVAudioUnitDelay:延迟，延迟就是 发出一个声音之后，过段时间再次发出，一直衰减到听不见。类似咱们的回声。可以通过里面的属性去细微的调节延迟的时间、速度等。


* 添加音效：
  主要流程就是链式关系

  **input(Mic或者音频文件) ->效果器->output**

  如果是多个音效

  **input(Mic或者音频文件) ->效果器1->效果器2->output**

  我们以AVAudioUnitReverb效果为例

  ```
  AVAudioUnitReverb * reverb = [[AVAudioUnitReverb alloc] init];
  [reverb loadFactoryPreset:AVAudioUnitReverbPresetLargeRoom];
  reverb.wetDryMix = 100;
  //把混响附着到音频引擎
  [_engine attachNode:reverb];
  //依次链接输入-> 混响 -> 输出
  [_engine connect:_engine.inputNode to:reverb format:[_engine.inputNode inputFormatForBus:AVAudioPlayerNodeBufferLoops]];
  [_engine connect:reverb to:_engine.outputNode format:[_engine.inputNode inputFormatForBus:AVAudioPlayerNodeBufferLoops]];
  //启动引擎
  [_engine startAndReturnError:nil];

  ```
  同理添加多个音效则需要严格按照 **input(Mic或者音频文件) ->效果器1->效果器2->output** 顺序来添加

  综上：完成了以上所有操作后你就可以实时在耳机中听到自己经过音效处理过的声音了，而且这样带着耳机唱歌效果会非常好，声音洪亮不易跑调。还可以针对不同的曲风调整自己的音效。


###### 声音混合、写入本地：
我们需要把我们清唱的歌曲录制到本地，正常的录制时使用AVAudioRecorder来进行录制的，像这样

```
AVAudioSession * session = [AVAudioSession sharedInstance];
    [session setCategory:AVAudioSessionCategoryPlayAndRecord error:nil];
    [session setActive:YES error:nil];


    NSString * path = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES)[0];
    self.filePath = [path stringByAppendingPathComponent:@"SoWeak"];
    self.recordFileUrl = [NSURL fileURLWithPath:self.filePath];

    //设置参数
    NSDictionary *recordSetting = [[NSDictionary alloc] initWithObjectsAndKeys:
                                   //采样率  8000/11025/22050/44100/96000（影响音频的质量）
                                   [NSNumber numberWithFloat: 8000.0],AVSampleRateKey,
                                   // 音频格式
                                   [NSNumber numberWithInt: kAudioFormatLinearPCM],AVFormatIDKey,
                                   //采样位数  8、16、24、32 默认为16
                                   [NSNumber numberWithInt:16],AVLinearPCMBitDepthKey,
                                   // 音频通道数 1 或 2
                                   [NSNumber numberWithInt: 2], AVNumberOfChannelsKey,
                                   //录音质量
                                   [NSNumber numberWithInt:AVAudioQualityHigh],AVEncoderAudioQualityKey,
                                   nil];
    self.recorder = [[AVAudioRecorder alloc] initWithURL:self.recordFileUrl settings:recordSetting error:nil];

    if (self.recorder) {
        _recorder.meteringEnabled = YES;
        [_recorder prepareToRecord];
        [_recorder record];
    }
```

但是很明显这样录制声音需要开启session 而声音的session是一个单利，如果这样开启了那么我们后面就不能用AVAudioEngine来进行音频采集了，也就没有之前的效果。所有根据以往的经验，AVAudioEngine在开启引擎之后一定会有一个delegate或者是block回调出采集到的数据的。于是我们找到了AudioNode中的这个方法：
```
- (void)installTapOnBus:(AVAudioNodeBus)bus bufferSize:(AVAudioFrameCount)bufferSize format:(AVAudioFormat * __nullable)format block:(AVAudioNodeTapBlock)tapBlock;
```
其中的block的buffer 便是我们采集到的数据。

```
/*!	@typedef AVAudioNodeTapBlock
	@abstract A block that receives copies of the output of an AVAudioNode.
	@param buffer
		a buffer of audio captured from the output of an AVAudioNode
	@param when
		the time at which the buffer was captured
	@discussion
		CAUTION: This callback may be invoked on a thread other than the main thread.
*/
typedef void (^AVAudioNodeTapBlock)(AVAudioPCMBuffer *buffer, AVAudioTime *when);
```

我们需要把buffer转成AVAudioFile然后通过AVAudioFile的write方法写入
```
  初始化AVAudioFile
  AVAudioFile * audioFile = [[AVAudioFile alloc] initForWriting:url settings:@{} error:nil];
  然后在block中实现
  [audioFile writeFromBuffer:buffer error:nil];

```
这个时候写入成功然后播放本地录音文件发现只有自己的原生，并没有后面添加的音效，回音等效果。

其实是因为我们虽然添加了音效但是我们没有把音效和原生混合在一起，即使我们实时听到的是没有问题的，但是当保存到本地之后如果没有做混合，系统会默认将最原始的声音写入本地，这里我们需要用到

**AVAudioMixerNode**

他是继承与AVAudioNode 也属于一个特殊音频处理节点，使用方式和之前的音效节点一样，添加在所有的处理之后、输出之前即可，像这样

**input(Mic或者音频文件) ->效果器1->效果器2->Mixer->output**

不过唯一需要注意的是这个mixer最好也写成属性、不然会出问题。

所以一个完整的带音效的清唱录制为：
```
//创建音频文件。
    NSString * path = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES)[0];
    NSString * filePath = [path stringByAppendingPathComponent:@"123.caf"];
    NSURL * url = [NSURL fileURLWithPath:filePath];
    AVAudioFile * audioFile = [[AVAudioFile alloc] initForWriting:url settings:@{} error:nil];
    self.recordFileUrl = url;


    AVAudioUnitReverb * reverd = [[AVAudioUnitReverb alloc] init];
    reverd.wetDryMix = 100;
    [reverd loadFactoryPreset:AVAudioUnitReverbPresetLargeRoom];
    [self.engine attachNode:reverd];

    [self.engine attachNode:_mixer];


    [self.engine connect:self.engine.inputNode to:reverd format:audioFile.processingFormat];
    [self.engine connect:reverd to:_mixer format:audioFile.processingFormat];
    [self.engine connect:_mixer to:self.engine.outputNode format:audioFile.processingFormat];

    [_mixer installTapOnBus:0 bufferSize:4096 format:[_engine.inputNode inputFormatForBus:AVAudioPlayerNodeBufferLoops] block:^(AVAudioPCMBuffer * _Nonnull buffer, AVAudioTime * _Nonnull when) {
        [audioFile writeFromBuffer:buffer error:nil];
        NSLog(@"我录制到的数据是 === %@", buffer);
    }];

    [self.engine startAndReturnError:nil];
```
## 总结

通过如上方法可以完整的实现清唱功能，但是唱吧清唱使用的是AudioUnit，AudioUnit是iOS中音频的非常底层的实现，由C语言实现，因为唱吧中除了清唱之外还有很多非常复杂的音频处理功能，所以只有AudioUnit可以满足，但是对于清唱这个功能来说，两种实现方式达到了同样的效果，本文介绍的更加轻量级，不过关于AudioUnit也正在学习过程，后续会输出相应的文章。
