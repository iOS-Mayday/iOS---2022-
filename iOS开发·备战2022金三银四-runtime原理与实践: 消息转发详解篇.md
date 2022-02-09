# iOS开发·备战2022金三银四-runtime原理与实践: 消息转发详解篇
> **摘要**：编程，只了解原理不行，必须实战才能知道应用场景。本系列尝试阐述runtime相关理论的同时介绍一些实战场景，而本文则是本系列的**消息转发**篇。**本文中**，第一节将介绍方法消息发送相关的概念，第二节将总结一下2\. 动态特性：方法解析和消息转发（Method Resolution，Fast Rorwarding，Normal Forwarding），第三节将介绍方法交换几种的实战场景：特定奔溃预防处理（调用未实现方法），苹果系统迭代造成API不兼容的奔溃处理，第四节将总结消息转发的机制。

## 1.OC的方法与消息

在我们开始使用消息机制之前，我们可以约定我们的术语。例如，很多人不清楚“方法”与“消息”是什么，但这对于理解消息传递系统如何在低级别工作至关重要。

*   **方法**：与一个类相关的一段实际代码，并给出一个特定的名字。例：`- (int)meaning { return 42; }`
*   **消息**：发送给对象的名称和一组参数。示例：向0x12345678对象发送`meaning`并且没有参数。
*   **选择器**：表示消息或方法名称的一种特殊方式，表示为类型SEL。选择器本质上就是不透明的字符串，它们被管理，因此可以使用简单的指针相等来比较它们，从而提高速度。（实现可能会有所不同，但这基本上是他们在外部看起来的样子。）例如：`@selector(meaning)`。
*   **消息发送**：接收信息并查找和执行适当方法的过程。

#### 1.1 方法与消息发送

**消息**在OC中**方法调用**是一个**消息发送**的过程。OC方法最终被生成为C函数，并带有一些额外的参数。这个C函数`objc_msgSend`就负责**消息发送**。在runtime的`objc/message.h`中能找到它的API。

```
objc_msgSend(id _Nullable self, SEL _Nonnull op, ...)`

```

#### 1.2 消息发送的主要步骤

消息发送的时候，在C语言函数中发生了什么事情？编译器是如何找到这个**方法**的呢？消息发送的主要步骤如下：

1.  首先检查这个selector是不是要忽略。比如Mac OS X开发，有了垃圾回收就不会理会retain，release这些函数。
2.  检测这个selector的target是不是nil，OC允许我们对一个nil对象执行任何方法不会Crash，因为运行时会被忽略掉。
3.  如果上面两步都通过了，就开始查找这个类的实现IMP，先从cache里查找，如果找到了就运行对应的函数去执行相应的代码。
4.  如果cache中没有找到就找类的方法列表中是否有对应的方法。
5.  如果类的方法列表中找不到就到父类的方法列表中查找，一直找到NSObject类为止。
6.  如果还是没找到就要开始进入**动态方法解析**和**消息转发**，后面会说。

其中，为什么它被称为 “转发”？ 当某个对象没有任何响应某个 消息 的操作就 “转发” 该 消息。原因是这种技术主要是为了让对象让其他对象为他们处理 消息，从而 “转发”。

**消息转发**是一种功能强大的技术，可以大大增加Objective-C的表现力。什么是消息转发？简而言之，它允许未知的消息被困住并作出反应。换句话说，无论何时发送未知消息，它都会以一个很好的包发送到您的代码中，此时您可以随心所欲地执行任何操作。

#### 1.3 OC的方法本质

OC中的方法默认被隐藏了两个参数：`self`和`_cmd`。你可能知道`self`是作为一个隐式参数传递的，它最终成为一个明确的参数。鲜为人知的隐式参数`_cmd`（它保存了正在发送的消息的选择器）是第二个这样的隐式参数。总之，`self`指向对象本身，`_cmd`指向方法本身。举两个例子来说明：

*   例1：`- (NSString *)name`
    这个方法实际上有两个参数：`self`和`_cmd`。

*   例2：`- (void)setValue:(int)val`
    这个方法实际上有三个参数：`self`,`_cmd` 和 `val`。

在编译时你写的 OC 函数调用的语法都会被翻译成一个 C 的函数调用 `objc_msgSend()` 。比如，下面两行代码就是等价的：

*   OC

```
[array insertObject:foo atIndex:5];

```

*   C

```
objc_msgSend(array, @selector(insertObject:atIndex:), foo, 5);

```

其中的`objc_msgSend`就负责消息发送。

## 2\. 动态特性：方法解析和消息转发

没有方法的实现，程序会在运行时挂掉并抛出 `unrecognized selector sent to …` 的异常。但在异常抛出前，Objective-C 的运行时会给你三次拯救程序的机会：

*   Method resolution
*   Fast forwarding
*   Normal forwarding

#### 2.1 动态方法解析: Method Resolution

首先，Objective-C 运行时会调用 `+ (BOOL)resolveInstanceMethod:`或者 `+ (BOOL)resolveClassMethod:`，让你有机会提供一个函数实现。如果你添加了函数并返回 YES， 那运行时系统就会重新启动一次消息发送的过程。还是以 foo 为例，你可以这么实现：

```
void fooMethod(id obj, SEL _cmd)  
{
    NSLog(@"Doing foo");
}

+ (BOOL)resolveInstanceMethod:(SEL)aSEL
{
    if(aSEL == @selector(foo:)){
        class_addMethod([self class], aSEL, (IMP)fooMethod, "v@:");
        return YES;
    }
    return [super resolveInstanceMethod];
}

```

这里第一字符`v`代表函数返回类型`void`，第二个字符`@`代表self的类型`id`，第三个字符`:`代表_cmd的类型`SEL`。这些符号可在Xcode中的开发者文档中搜索Type Encodings就可看到符号对应的含义，更详细的官方文档传送门 [在这里](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100)，此处不再列举了。

![image](https://upload-images.jianshu.io/upload_images/24396915-9b8c269df0795964.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 2.2 快速转发: Fast Rorwarding

与下面2.3完整转发不同，Fast Rorwarding这是一种快速消息转发：只需要在指定API方法里面返回一个新对象即可，当然其它的逻辑判断还是要的（比如该SEL是否某个指定SEL？）。

消息转发机制执行前，runtime系统允许我们替换消息的接收者为其他对象。通过`- (id)forwardingTargetForSelector:(SEL)aSelector`方法。如果此方法返回的是nil 或者self,则会进入消息转发机制（`- (void)forwardInvocation:(NSInvocation *)invocation`），否则将会向返回的对象重新发送消息。

```
- (id)forwardingTargetForSelector:(SEL)aSelector {
    if(aSelector == @selector(foo:)){
        return [[BackupClass alloc] init];
    }
    return [super forwardingTargetForSelector:aSelector];
}

```

> [点击这里获取2022最新面试合集](https://jq.qq.com/?_wv=1027&k=GGrxKy72)

#### 2.3 完整消息转发: Normal Forwarding

与上面不同，可以理解成完整消息转发，是可以代替快速转发做更多的事。

```
- (void)forwardInvocation:(NSInvocation *)invocation {
    SEL sel = invocation.selector;
    if([alternateObject respondsToSelector:sel]) {
        [invocation invokeWithTarget:alternateObject];
    } else {
        [self doesNotRecognizeSelector:sel];
    }
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    NSMethodSignature *methodSignature = [super methodSignatureForSelector:aSelector];
    if (!methodSignature) {
        methodSignature = [NSMethodSignature signatureWithObjCTypes:"v@:*"];
    }
    return methodSignature;
}

```

`forwardInvocation:` 方法就是一个不能识别消息的分发中心，将这些不能识别的消息转发给不同的消息对象，或者转发给同一个对象，再或者将消息翻译成另外的消息，亦或者简单的“吃掉”某些消息，因此没有响应也不会报错。例如：我们可以为了避免直接闪退，可以当消息没法处理时在这个方法中给用户一个提示，也不失为一种友好的用户体验。

其中，参数`invocation`是从哪来的？在`forwardInvocation:`消息发送前，runtime系统会向对象发送`methodSignatureForSelector:`消息，并取到返回的方法签名用于生成NSInvocation对象。所以重写`forwardInvocation:`的同时也要重写`methodSignatureForSelector:`方法，否则会抛出异常。当一个对象由于没有相应的方法实现而无法响应某个消息时，运行时系统将通过`forwardInvocation:`消息通知该对象。每个对象都继承了`forwardInvocation:`方法，我们可以将消息转发给其它的对象。

#### 2.4 区别: Fast Rorwarding 对比 Normal Forwarding？

可能有朋友看到，这两个转发都是将消息转发给其它对象，那么这两个有什么区别？

*   需要重载的API方法的用法不同

    *   前者只需要重载一个API即可，后者需要重载两个API。
    *   前者只需在API方法里面返回一个新对象即可，后者需要对被转发的消息进行重签并手动转发给新对象（利用 `invokeWithTarget:`）。
*   转发给新对象的个数不同
    *   前者只能转发一个对象，后者可以连续转发给多个对象。例如下面是完整转发：

```
-(NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector
{
    if (aSelector==@selector(run)) {
        return [NSMethodSignature signatureWithObjCTypes:"v@:"];
    }
    return [super methodSignatureForSelector: aSelector];
}

-(void)forwardInvocation:(NSInvocation *)anInvocation
{
    SEL selector =[anInvocation selector];

    RunPerson *RP1=[RunPerson new];
    RunPerson *RP2=[RunPerson new];

    if ([RP1 respondsToSelector:selector]) {

        [anInvocation invokeWithTarget:RP1];
    }
    if ([RP2 respondsToSelector:selector]) {

        [anInvocation invokeWithTarget:RP2];
    }    
}

```

## 3\. 应用实战：消息转发

#### 3.1 特定奔溃预防处理

下面有一段因为没有实现方法而会导致奔溃的代码：

*   Test2ViewController

```
- (void)viewDidLoad {
    [super viewDidLoad];
    [self.view setBackgroundColor:[UIColor whiteColor]];
    self.title = @"Test2ViewController";

    //实例化一个button,未实现其方法
    UIButton *button = [UIButton buttonWithType:UIButtonTypeCustom];
    button.frame = CGRectMake(50, 100, 200, 100);
    button.backgroundColor = [UIColor blueColor];
    [button setTitle:@"消息转发" forState:UIControlStateNormal];
    [button addTarget:self
               action:@selector(doSomething)
     forControlEvents:UIControlEventTouchUpInside];
    [self.view addSubview:button];
}

```

为解决这个问题，可以专门创建一个处理这种问题的分类：

*   NSObject+CrashLogHandle

```
#import "NSObject+CrashLogHandle.h"

@implementation NSObject (CrashLogHandle)

- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    //方法签名
    return [NSMethodSignature signatureWithObjCTypes:"v@:@"];
}

- (void)forwardInvocation:(NSInvocation *)anInvocation {
    NSLog(@"NSObject+CrashLogHandle---在类:%@中 未实现该方法:%@",NSStringFromClass([anInvocation.target class]),NSStringFromSelector(anInvocation.selector));
}

@end

```

因为在category中复写了父类的方法，会出现下面的警告:

![image](https://upload-images.jianshu.io/upload_images/24396915-1ccb561b6bc60bcc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

解决办法就是在Xcode的Build Phases中的资源文件里，在对应的文件后面 -w ，忽略所有警告。

![image](https://upload-images.jianshu.io/upload_images/24396915-5715d29ff4c12cea.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 3.2 苹果系统API迭代造成API不兼容的奔溃处理

##### 3.2.1 兼容系统API迭代的传统方案

随着每年iOS系统与硬件的更新迭代，部分性能更优异或者可读性更高的API将有可能对原有API进行废弃与更替。与此同时我们也需要对现有APP中的老旧API进行版本兼容，当然进行版本兼容的方法也有很多种，下面笔者会列举常用的几种:

*   根据能否响应方法进行判断

```
if ([object respondsToSelector: @selector(selectorName)]) {
    //using new API
} else {
    //using deprecated API
}

```

*   根据当前版本SDK是否存在所需类进行判断

```
if (NSClassFromString(@"ClassName")) {    
    //using new API
}else {
    //using deprecated API
}

```

*   根据操作系统版本进行判断

```
#define isOperatingSystemAtLeastVersion(majorVersion, minorVersion, patchVersion)[[NSProcessInfo processInfo] isOperatingSystemAtLeastVersion: (NSOperatingSystemVersion) {
    majorVersion,
    minorVersion,
    patchVersion
}]

if (isOperatingSystemAtLeastVersion(11, 0, 0)) {
    //using new API
} else {
    //using deprecated API
}

```

##### 3.2.2 兼容系统API迭代的新方案

> **需求**：假设现在有一个利用新API写好的类，如下所示，其中有一行可能因为运行在低版本系统（比如iOS9）导致奔溃的代码：

*   Test3ViewController.m

```
- (void)viewDidLoad {
    [super viewDidLoad];
    [self.view setBackgroundColor:[UIColor whiteColor]];
    self.title = @"Test3ViewController";

    UITableView *tableView = [[UITableView alloc] initWithFrame:CGRectMake(0, 64, 375, 600) style:UITableViewStylePlain];
    tableView.delegate = self;
    tableView.dataSource = self;
    tableView.backgroundColor = [UIColor orangeColor];

    // May Crash Line
    tableView.contentInsetAdjustmentBehavior = UIScrollViewContentInsetAdjustmentNever;

    [tableView registerClass:[UITableViewCell class] forCellReuseIdentifier:@"UITableViewCell"];
    [self.view addSubview:tableView];
}

```

其中有一行会发出警告，Xcode也给出了推荐解决方案，如果你点击Fix它会自动添加检查系统版本的代码，如下图所示:

![image](https://upload-images.jianshu.io/upload_images/24396915-2cf644495609e3b2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> **方案1**：手动加入版本判断逻辑

以前的适配处理，可根据操作系统版本进行判断

```
if (isOperatingSystemAtLeastVersion(11, 0, 0)) {
    scrollView.contentInsetAdjustmentBehavior = UIScrollViewContentInsetAdjustmentNever;
} else {
    viewController.automaticallyAdjustsScrollViewInsets = NO;
}

```

> **方案2**：消息转发

在iOS11 Base SDK直接采取最新的API并且配合Runtime的消息转发机制就能实现一行代码在不同版本操作系统下采取不同的消息调用方式

*   UIScrollView+Forwarding.m

```
#import "UIScrollView+Forwarding.h"
#import "NSObject+AdapterViewController.h"

@implementation UIScrollView (Forwarding)

- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector { // 1

    NSMethodSignature *signature = nil;
    if (aSelector == @selector(setContentInsetAdjustmentBehavior:)) {
        signature = [UIViewController instanceMethodSignatureForSelector:@selector(setAutomaticallyAdjustsScrollViewInsets:)];
    }else {
        signature = [super methodSignatureForSelector:aSelector];
    }
    return signature;
}

- (void)forwardInvocation:(NSInvocation *)anInvocation { // 2

    BOOL automaticallyAdjustsScrollViewInsets  = NO;
    UIViewController *topmostViewController = [self cm_topmostViewController];
    NSInvocation *viewControllerInvocation = [NSInvocation invocationWithMethodSignature:anInvocation.methodSignature]; // 3
    [viewControllerInvocation setTarget:topmostViewController];
    [viewControllerInvocation setSelector:@selector(setAutomaticallyAdjustsScrollViewInsets:)];
    [viewControllerInvocation setArgument:&automaticallyAdjustsScrollViewInsets atIndex:2]; // 4
    [viewControllerInvocation invokeWithTarget:topmostViewController]; // 5
}

@end

```

*   NSObject+AdapterViewController.m

```
#import "NSObject+AdapterViewController.h"

@implementation NSObject (AdapterViewController)

- (UIViewController *)cm_topmostViewController {
    UIViewController *resultVC;
    resultVC = [self cm_topViewController:[[UIApplication sharedApplication].keyWindow rootViewController]];
    while (resultVC.presentedViewController) {
        resultVC = [self cm_topViewController:resultVC.presentedViewController];
    }
    return resultVC;
}

- (UIViewController *)cm_topViewController:(UIViewController *)vc {
    if ([vc isKindOfClass:[UINavigationController class]]) {
        return [self cm_topViewController:[(UINavigationController *)vc topViewController]];
    } else if ([vc isKindOfClass:[UITabBarController class]]) {
        return [self cm_topViewController:[(UITabBarController *)vc selectedViewController]];
    } else {
        return vc;
    }
}

@end

```

当我们在iOS10调用新API时，由于没有具体对应API实现，我们将其原有的消息转发至当前栈顶UIViewController去调用低版本API。

关于`[self cm_topmostViewController];`，执行之后得到的结果可以查看如下：

![image](https://upload-images.jianshu.io/upload_images/24396915-5a2c928fca2b40e7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> **方案2的整体流程**：

1.  为即将转发的消息返回一个对应的方法签名(该签名后面用于对转发消息对象(NSInvocation *)anInvocation进行编码用)

2.  开始消息转发((NSInvocation *)anInvocation封装了原有消息的调用，包括了方法名，方法参数等)

3.  由于转发调用的API与原始调用的API不同，这里我们新建一个用于消息调用的NSInvocation对象viewControllerInvocation并配置好对应的target与selector

4.  配置所需参数:由于每个方法实际是默认自带两个参数的:self和_cmd，所以我们要配置其他参数时是从第三个参数开始配置

5.  消息转发

##### 3.2.3 验证对比新方案

注意测试的时候，选择iOS10系统的模拟器进行验证（没有的话可以先Download Simulators），安装完后如下如选择：

![image](https://upload-images.jianshu.io/upload_images/24396915-36a5abee74cfa821.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

*   不注释并导入UIScrollView+Forwarding类

    ![image](https://upload-images.jianshu.io/upload_images/24396915-8566ec3b3ba7d01c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

*   注释掉UIScrollView+Forwarding的功能代码

会如下图所示奔溃：

![image](https://upload-images.jianshu.io/upload_images/24396915-b89851865e888935.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 4\. 总结

#### 4.1 模拟多继承

> **面试挖坑**：OC是否支持多继承？好，你说不支持多继承，那你有没有模拟多继承特性的办法？

转发和继承相似，可用于为OC编程添加一些多继承的效果，一个对象把消息转发出去，就好像他把另一个对象中放法接过来或者“继承”一样。消息转发弥补了objc不支持多继承的性质，也避免了因为多继承导致单个类变得臃肿复杂。

虽然转发可以实现继承功能，但是NSObject还是必须表面上很严谨，像`respondsToSelector:`和`isKindOfClass:`这类方法只会考虑继承体系，不会考虑转发链。

#### 4.2 消息机制总结

![image](https://upload-images.jianshu.io/upload_images/24396915-fe87d852ba442b73.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Objective-C 中给一个对象发送消息会经过以下几个步骤：

1.  在对象类的 dispatch table 中尝试找到该消息。如果找到了，跳到相应的函数IMP去执行实现代码；

2.  如果没有找到，Runtime 会发送 `+resolveInstanceMethod:` 或者 `+resolveClassMethod:`尝试去 resolve 这个消息；

3.  如果 resolve 方法返回 NO，Runtime 就发送 `-forwardingTargetForSelector:` 允许你把这个消息转发给另一个对象；

4.  如果没有新的目标对象返回， Runtime 就会发送`-methodSignatureForSelector:` 和 `-forwardInvocation:` 消息。你可以发送 `-invokeWithTarget:` 消息来手动转发消息或者发送 `-doesNotRecognizeSelector:` 抛出异常。

**金三银四即将到来~小编这里有大量的iOS书籍和iOS面试资料哦[**[点击进群-即可获取](https://link.zhihu.com/?target=https%3A//jq.qq.com/%3F_wv%3D1027%26k%3D00hjBgOq)**]**

![](https://upload-images.jianshu.io/upload_images/24396915-5205e91d96145987.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](https://upload-images.jianshu.io/upload_images/24396915-9b40638059e15195.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
