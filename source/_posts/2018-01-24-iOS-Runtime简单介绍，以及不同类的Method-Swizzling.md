---
title: iOS Runtime简单介绍，以及不同类的Method Swizzling
categories: iOS开发
date: 2018-01-24 12:18:50
tags:
  - runtime
  - 底层
comments:
---
##### Runtime介绍：
 runtime顾名思义就是运行时，其实我们的App从你按下command+R开始一直到App运行起来经历了大致两个阶段，1：编译时，2：运行时。还记得一道很经典的面试题
![](https://wx3.sinaimg.cn/large/006tNc79gy1fo6kbsn9yxj30kk03et8z.jpg)

这里给大家解释下：首先， * testObject 是告诉编译器，testObject是一个指向某个Objective-C对象的指针。因为不管指向的是什么类型的对象，
<!--more-->
一个指针所占的内存空间都是固定的，所以这里声明成任何类型的对象，最终生成的可执行代码都是没有区别的。这里限定了NSString只不过是告诉编译器，请把testObject当做一个NSString来检查，如果后面调用了非NSString的方法，会产生警告。接着，你创建了一个NSData对象，然后把这个对象所在的内存地址保存在testObject里。那么运行时(从这段代码执行开始，到程序结束)，testObject指向的内存空间就是一个NSData对象。你可以把testObject当做一个NSData对象来用。 所以编译时是NSString，运行时是NSData。
runtime是什么：
在runtime中，所有的类在OC中都会被定义成一个结构体，像这样
类在runtime中的表示
struct objc_class {
    Class isa;//指针，顾名思义，表示是一个什么，  //实例的isa指向类对象，类对象的isa指向元类
```
#if !__OBJC2__
        Class super_class;  //指向父类
        const char *name;  //类名
        long version;     //类的版本信息，默认初始化为 0。我们可以在运行期对其进行修改（class_setVersion）或获取（class_getVersion）。
        long info;   /*供运行期使用的一些位标识。有如下一些位掩码：
                        CLS_CLASS (0x1L) 表示该类为普通 class ，其中包含实例方法和变量；
                      CLS_META (0x2L) 表示该类为 metaclass，其中包含类方法；
                      CLS_INITIALIZED (0x4L) 表示该类已经被运行期初始化了，这个标识位只被 objc_addClass 所设置；
                      CLS_POSING (0x8L) 表示该类被 pose 成其他的类；（poseclass 在ObjC 2.0中被废弃了）；
                      CLS_MAPPED (0x10L) 为ObjC运行期所使用
                      CLS_FLUSH_CACHE (0x20L) 为ObjC运行期所使用
                      CLS_GROW_CACHE (0x40L) 为ObjC运行期所使用
                      CLS_NEED_BIND (0x80L) 为ObjC运行期所使用
                      CLS_METHOD_ARRAY (0x100L) 该标志位指示 methodlists 是指向一个 objc_method_list 还是一个包含 objc_method_list 指针的数组；*/
        long instance_size  //该类的实例变量大小（包括从父类继承下来的实例变量）；
        struct objc_ivar_list *ivars //成员变量列表
        struct objc_method_list **methodLists; //方法列表
        struct objc_cache *cache;//缓存   一种优化，调用过的方法存入缓存列表，下次调用先找缓存
        struct objc_protocol_list *protocols //协议列表
        #endif
} OBJC2_UNAVAILABLE;
```
相关的定义
/// 描述类中的一个方法
typedef struct objc_method *Method;

/// 实例变量
typedef struct objc_ivar *Ivar;

/// 类别Category
typedef struct objc_category *Category;

/// 类中声明的属性
typedef struct objc_property *objc_property_t;

ObjC 为每个类的定义生成两个 objc_class ，一个即普通的 class，另一个即 metaclass。我们可以在运行期创建这两个 objc_class 数据结构，然后使用 objc_addClass 动态地创建新的类定义。

##### runtime能干什么：
* ：1：获取一个类中的列表比如方法列表、属性列表、协议列表、成员变量列表像如下这样 其中获取到的属性、方法都是可以获取public和private的。

```
unsigned int count;
    Class clas = [WKWebViewController class]; //是我自己的类，之所以不用系统的类是因为系统的类方法属性太多了

    objc_property_t * propertyList = class_copyPropertyList(clas, &count);
    for (int i = 0; i < count; i++) {
        const char *propertyName = property_getName(propertyList[i]);
        NSLog(@"  %@  属性(包括私有) -------->>>>>    %@",clas,[NSString stringWithUTF8String:propertyName]);
    }
    NSLog(@"-------------------------------------------------------------------------------------------------------------- ");

    Method * methodList = class_copyMethodList(clas, &count);
    for (int i = 0; i < count; i++) {
        Method methodName = methodList[i];
        NSLog(@"  %@ 方法(包括私有)  -------->>>>>    %@",clas,NSStringFromSelector(method_getName(methodName)));
    }
    NSLog(@"-------------------------------------------------------------------------------------------------------------- ");


    Ivar *ivarList = class_copyIvarList(clas, &count);
    for (int i = 0; i<count; i++) {
        Ivar myIvar = ivarList[i];
        const char *ivarName = ivar_getName(myIvar);
        NSLog(@"%@ 成员变量(包括私有) -------->>>>> %@",clas, [NSString stringWithUTF8String:ivarName]);
    }
    NSLog(@"-------------------------------------------------------------------------------------------------------------- ");


    //获取协议列表
    __unsafe_unretained Protocol **protocolList = class_copyProtocolList([self class], &count);
    for (int i = 0; i<count; i++) {
        Protocol *myProtocal = protocolList[i];
        const char *protocolName = protocol_getName(myProtocal);
        NSLog(@"%@ 协议 -------->>>>> %@",clas, [NSString stringWithUTF8String:protocolName]);
    }
```
输出后的结果是
![image.png](https://wx1.sinaimg.cn/large/006tNc79gy1fo6kc7w3zcj30pk0ajdks.jpg)
其中也包括了私有方法。

* 2：拦截方法调用
有的时候我们用一个类或者一个实例变量去调用一个方法，由于操作失误或者是其他原因，导致这个所被调用的方法并不存在，报出这样的错误，然后闪退！
![image.png](https://wx1.sinaimg.cn/large/006tNc79gy1fo6kcfrky6j30tr0120sy.jpg)

这个时候如果我们想避免这些崩溃，我们就需要在运行时对其做一些手脚。iOS中方法调用的流程：其实调用方法就是发送消息，所有调用方法的代码例如   [obj aaa]  在运行时runtime会将这段代码转换为objc_msgSend(obj, [@selector]);（本质就是发送消息）然后obj会通过其中isa指针去该类的缓存中(cache)查找对应函数的Method, 如果没有找到，再去该类的方法列表（methodList）中查找，如果没有找到再去该类的父类找，如果找到了，就先将方法添加到缓存中，以便下次查找，然后通过method中的指针定位到指定方法执行。如果一直没有找到，便会走完如下四个方法之后崩溃。

```
/**
    如果调用的是不存在的实例方法则会在奔溃前进入该方法，防止崩溃可以在此处做处理
 */
+(BOOL)resolveInstanceMethod:(SEL)sel {
    return YES;
}

/**
 如果调用的是不存在的类方法则会在奔溃前进入该方法，防止崩溃可以在此处做处理
 */
+(BOOL)resolveClassMethod:(SEL)sel {
    return YES;
}

/**
 这个方法会把你所调用的不存在的方法重定向到一个声明了该方法的类中，只需要你返回一个有该方法的
 类就可以，如果你重定向的这个类仍然不具有该方法那么会继续崩溃
 */
-(id)forwardingTargetForSelector:(SEL)aSelector {

}

/**
 将你不存在的方法打包成NSInvocation对象，做完你自己的处理之后
 调用invokeWithTarget让某个target来处理该方法
 */
-(void)forwardInvocation:(NSInvocation *)anInvocation {
    [anInvocation invokeWithTarget:self];
}
```

* 3：动态添加方法
因为我们调用了一个不存在的方法导致崩溃，那么我们在判断出不存在后就动态添加上一个方法吧 这样不就不会蹦了吗？我们先写一个方法用来给我们做出提示

```
- (void) errorMethod {
    NSLog(@"no method!!!!!!!");
}
```
如果调用了没有的方法，那么就把这个方法添加进去，然后把被调用的方法的指针指向这个error1：，那么一旦调用了没有的方法就会走这个。我们来看代码

```
+(BOOL)resolveInstanceMethod:(SEL)sel {
    Method errorMethod =  class_getInstanceMethod([self class], @selector(errorMethod));
    if ([NSStringFromSelector(sel) isEqualToString:@"testMethod"]) {
        BOOL isAdd =  class_addMethod([self class], sel, method_getImplementation(errorMethod), method_getTypeEncoding(errorMethod));
        NSLog(@"tinajia  = %d",isAdd);
    }
    //Do something
    return YES;
}
```

主要用到

```
/**
    添加方法
     @param class] 在哪个类里添加
     @param sel 添加的方法的名字
     @param errorMethod 添加的方法的实现IMP指
     @param types 方法的标示符
     @return 是否添加成功
         */
BOOL isAdd =  class_addMethod([self class], sel, method_getImplementation(errorMethod), method_getTypeEncoding(errorMethod));

```

然后运行下：

```
WKWebViewController * vc= [[WKWebViewController alloc] init];
[vc performSelector:@selector(testMethod)];

```

我调用了并不存在的testMethod方法并没有崩溃并且方法已经成功添加了

![image.png](https://wx3.sinaimg.cn/large/006tNc79gy1fo6kexyuzgj30f801mwel.jpg)

* 4：动态交换方法（也叫iOS黑魔法，慎用）
没什么好例子，用一个网上说的例子(引用别人的东西，懒得复制了，就截了图)

  ![](https://wx4.sinaimg.cn/large/006tNc79gy1fo6kg3i5z6j30hv0fj0z9.jpg)

  其实本质即使SEL和IMP的交换，原理是这样的：在iOS中每一个类中都有一个叫dispatch table的东西，里面存放在SEL 和他所对应的IMP指针，之前也说过方法调用就是通过sel找IMP指针然后指针定位调用方法。方法交换就是对这个dispatch table进行操作。让A的SEL去对应B的IMP，B的SEL对应A的IMP，如图

  ![](https://wx3.sinaimg.cn/large/006tNc79gy1fo6kgrq52oj30f80betcz.jpg)

  这样就达到方法交换的目的，下面看代码：

```
+ (void)changeMethod {
    //  如果是类方法 要使用 !
    //  如果是系统的集合类的属性要用元类 比如 __NSSetM = NSMutableSet
    //  Class  class = NSClassFromString(@"__NSSetM");
    //  Class metaClass = objc_getMetaClass([NSStringFromClass(class) UTF8String]);
    Class systemClass = NSClassFromString(__NSSetM);

    SEL sel_System = NSSelectorFromString(addObject:);
    SEL sel_Custom = @selector(swizzle_addObject:);

    Method method_System = class_getInstanceMethod(systemClass, sel_System);
    Method method_Custom = class_getInstanceMethod([self class], sel_Custom);

    IMP imp_System = method_getImplementation(method_System);
    IMP imp_Custom = method_getImplementation(method_Custom);

    method_exchangeImplementations(method_System, method_Custom);
}

- (void)swizzle_addObject:(id) obj {
    if (obj) {
        [self swizzle_addObject:obj];
    }
}

```

主要代码  method_exchangeImplementations(method1, method2); 这两个参数很简单，就是两个需要交换的方法。
最后我调用了m1但是实际上走了m2。

##### 动态交换方法的原理以及交换过程中指针的变化

在通常的方法交换中我们通常有两种情景，一种是我会针对被交换的类建一个category，然后hook的方法会写在category中。另一种是自己创建一个Tool类里面放些常用的工具方法其中包含了方法交换。可能大家普遍选择第一种方法，但是如果你需要hook的类非常多的(我实际项目中就遇到这样的问题)那你就需要针对不同的类创建category，就会导致文件过多，且每一个文件中只有一个hook方法，这样一来左侧一堆文件，所以我用了第二种方法，但是在使用过程中出现一个问题，先看下我的代码结构

![image.png](https://wx3.sinaimg.cn/large/006tNc79gy1fo6khw8c97j30740ag74v.jpg)

我要hook的是ViewController中的viewDidLoad方法，我建立了两个类一个是ViewController的category，另一个是Tool类，为了一会区别演示不同类hook的不同(两个类中hook的代码完全一样)
* ViewController中将要被替换的系统方法

![被替换的方法(系统方法)](https://wx3.sinaimg.cn/large/006tNc79gy1fo6kir8y63j309a02rglq.jpg)


* Category中将要用来替换的自定义方法

![用来替换的方法(自定义方法)](https://wx1.sinaimg.cn/large/006tNc79gy1fo6kj5dcykj308z02lwep.jpg)
* 然后在ViewController中的load中做方法替换

![进行方法替换](https://wx1.sinaimg.cn/large/006tNc79gy1fo6kjlzvdqj30f90b43zw.jpg)

运行一下的输出结果想必大家已经猜到了先执行custom再执行system，这是通常情况下大家的做法。
![结果](https://wx1.sinaimg.cn/large/006tNc79gy1fo6kjxsheej30d701h3yl.jpg)

下面再来看下如果我将替换方法写在不同类中会怎样，调用Tool中的交换方法

![执行Tool中的交换方法](https://wx1.sinaimg.cn/large/006tNc79gy1fo6kka2s8uj30dx0anwfp.jpg)

然后直接看结果了，因为代码都是一模一样的我直接复制过去的

![结果](https://wx3.sinaimg.cn/large/006tNc79gy1fo6kliitupj30yd08w422.jpg)

发生了crash，原因是ViewController中没有swizzel_viewDidLoad_custom这个方法，为什么不同类的交换会出现这种问题，我们用个图来说明下

![image.png](https://wx3.sinaimg.cn/large/006tNc79gy1fo6km0wogkj30yg0pz43q.jpg)

解决的办法是我们在交换方法之前要先像其中添加方法，也就是说把customMethod添加到SystemClass中，但是注意要把customMethod的实现指向syetemMethod的实现。这样一来就可以达到SystemClass调用customMethod却执行systemMethod的代码的效果，实现以上要求我们需要在交换之前执行这个方法。

```
class_addMethod(systemClass, sel_Custom, imp_System, method_getTypeEncoding(method_System))
```

其中第一个参数是需要往哪个类添加；第二个参数是要添加的方法的方法名；第三个参数是所添加的方法的方法实现，第四个是方法的标识符。经过就该之后我们的代码是这样

```
.
.
之前的都一样就省略
.
.
if (class_addMethod(systemClass, sel_Custom, imp_System, method_getTypeEncoding(method_System))) {
        class_replaceMethod(systemClass, sel_System, imp_Custom, method_getTypeEncoding(method_System));
    } else {
        method_exchangeImplementations(method_System, method_Custom);
    }

```

我们来看下执行完add操作之后此时的方法和类的对应关系(红色的为add的修改)


![关系](https://wx4.sinaimg.cn/large/006tNc79gy1fo6kmhkzidj30yg0g378f.jpg)


因为SystemClass中本身不包含customMethod所以add一定是成功的，也就是说会进入判断执行replace方法。

```
class_replaceMethod(systemClass, sel_System, imp_Custom, method_getTypeEncoding(method_System));

```

第一个参数：需要修改的方法的所在的类；第二个参数：需要替换其实现的方法名；第三个参数：需要把哪个实现替换给他；第四个参数：方法标识符。此时看下我们做完replace之后的类与方法名以及他们实现的关系(红色的为replace的修改)。

![关系](https://wx1.sinaimg.cn/large/006tNc79gy1fo6kn9m5btj30yg0ifgqa.jpg)

此时大家已经看出来了，虽然没有执行exchange方法，但是我已经达到了方法交换的目的。系统执行systemMethod时候会走customMethod的实现但是因为在customMethod方法中我会递归执行[self customMethod]，所以又会走到systemMethod的实现，因为之前进行了方法添加，所以此时A类中有了customMethod方法，不会再发生之前的crash。达到一个不同类进行Method Swizzling的目的。

##### 综上来看一个完整严谨的MethodSwizzling应该在交换前先add，并且add方法的参数不能错

```
+ (void)changeMethod {

    Class systemClass = NSClassFromString(@"你的类");

    SEL sel_System = @selector(系统方法);
    SEL sel_Custom = @selector(你自己的方法);

    Method method_System = class_getInstanceMethod(systemClass, sel_System);
    Method method_Custom = class_getInstanceMethod([self class], sel_Custom);

    IMP imp_System = method_getImplementation(method_System);
    IMP imp_Custom = method_getImplementation(method_Custom);

    if (class_addMethod(systemClass, sel_Custom, imp_System, method_getTypeEncoding(method_System))) {
        class_replaceMethod(systemClass, sel_System, imp_Custom, method_getTypeEncoding(method_System));
    } else {
        method_exchangeImplementations(method_System, method_Custom);
    }
}

```

##### 以上代码无论是写在工具类中还是category中都是没有问题的。
