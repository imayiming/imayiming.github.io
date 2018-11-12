---
layout: post
cover: 'assets/images/cover11.jpeg'
navigation: True
title: "第 7 章 系统框架"
date: 2016-06-11 11:34:05 +0800
tags: iOS
subclass: 'post tag-speeches'
logo: 'assets/images/ghost.png'
author: yiming
categories: yiming
description: "Core Foundation 与 Foundation 的那些事"
---

0.CoreFoundation 与 Foundation 不仅名字相似，而且还有更为紧密的联系。Foundation 框架中的许多功能，都可以在此框架中找到对应的 C 语言 API。有个功能叫做“无缝桥接”（tollfree bridging），可以把 CoreFoundation 中的 C 语言数据结构平滑转换为 Foundation 中的 Objective-C 对象，也可以反向旋转。

1.ARC 下只考虑 Objective-C 对象的内存，对于非 Objective-C 对象，比如 CoreFoundation 中的需要自己考虑内存管理问题。

2.Objective-C 编程的一项重要特点，那就是：经常需要使用底层的 C 语言级 API。用  C 语言来实现 API 的好处是，可以绕过 Objective-C 的运行时系统，从而提升执行速度。

3.多用块枚举，少用 for 循环。

```
[array enumerateObjectsUsingBlock:^(NSString *  _Nonnull name, NSUInteger idx, BOOL * _Nonnull stop) {

	...
	
	if ([name isEqualToString:@"Tom"]) {
		stop = YES;
	}
}];
```

4.块枚举优点：

- 遍历时可以直接从块里获取更多信息。在遍历数组时，可以知道当前所针对的下标。
- 能够修改块的方法签名，以免进行类型转换操作。从效果上讲，相当于把本来需要执行的类型转换操作交给方法签名来做。

5.Core Foundation 与 Foundation 内存问题


``__bridge`` 什么也不做，仅仅是转换。此种情况下：  

1. 从 Cocoa 转换到 Core，需要人工 ``CFRetain``，否则，Cocoa 指针释放后， 传出去的指针则无效。
2. 从 Core 转换到 Cocoa，需要人工 ``CFRelease``，否则，Cocoa 指针释放后，对象引用计数仍为1，不会被销毁。

``__bridge_retained`` 转换后自动调用 ``CFRetain``，即帮助自动解决上述 1 的情形。

``__bridge_transfer`` 转换后自动调用 ``CFRelease``，即帮助自动解决上述 2 的情形。  

- ``__bridge`` 用法

```
NSString *string = [NSString stringWithFormat:...];
CFStringRef cfString = (__bridge CFStringRef)string;
```
只是单纯地执行了类型转换，没有进行所有权的转移，也就是说，当 ``string`` 对象被释放的时候，``cfstring`` 也不能被使用了。

- ``__bridge_retained`` 用法

```
NSString *string = [NSString stringWithFormat:...];
CFStringRef cfString = (__bridge_retained CFStringRef)string;
...
CFRelease(cfString); // 由于Core Foundation的对象不属于ARC的管理范畴，所以需要自己release
```
使用 ``__bridge_retained`` 可以通过转换目标处（ ``cfString`` ）的 ``retain`` 处理，来使所有权转移。即使 ``string`` 变量被释放，``cfString`` 还是可以使用具体的对象。只是有一点，由于 Core Foundation 的对象不属于 ARC 的管理范畴，所以需要自己 ``release``。

可以用 ``CFBridgingRetain`` 替代 ``__bridge_retained`` 关键字：  

```
NSString *string = [NSString stringWithFormat:...];
CFStringRef cfString = CFBridgingRetain(string);
...
CFRelease(cfString); // 由于Core Foundation不在ARC管理范围内，所以需要主动release。
```

- ``__bridge_transfer``

```
CFStringRef cfString = CFStringCreate...();
NSString *string = (__bridge_transfer NSString *)cfString;
 
// CFRelease(cfString); 因为已经用 __bridge_transfer 转移了对象的所有权，所以不需要调用 release
```

所有权被转移的同时，被转换变量将失去对象的所有权。当 Core Foundation 对象类型向Objective-C 对象类型转换的时候，会经常用到 ``__bridge_transfer`` 关键字。

同样，我们可以使用 ``CFBridgingRelease()`` 来代替 ``__bridge_transfer`` 关键字。  

```
CFStringRef cfString = CFStringCreate...();
NSString *string = CFBridgingRelease(cfString);
```

6.``NSCache`` VS ``NSDictionary``

- ``NSCache`` 胜过 ``NSDictionary`` 之处在于，当系统资源将要耗尽时，它可以自动删减缓存。如果采用普通的字典，那么就要自己编写挂钩，在系统发出“低内存”通知时手工删减缓存。
- ``NSCache``还会先行删减 “最久未使用”对象。
- ``NSCache``并不会拷贝键，而是会保留它。
- ``NSCache`` 是线程安全的，而 ``NSDictionary`` 则绝对不具备此优势。

7.只有那种“重新计算起来很费事的”数据，才值得放入缓存，比如那些需要从网络获取或从磁盘获取的数据。

8.``+ (void)load`` 方法

- ``+ (void)load``，对于加入运行期系统的每个类以及分类来说，必定会调用此方法，而且仅调用一次。当包含类或分类的程序库载入系统时，就会执行此方法，而这通常就是指应用程序启动的时候，若程序是为 iOS 平台设计的，则肯定会在此时执行。
- 在 ``load`` 方法中使用其他类是不安全的。
- ``load`` 方法并不像普通的方法那样，它并不遵从那套继承规则。分类和所属的类里，都可能出现 load 方法。此时两种实现代码都会调用，类的实现要比分类的实现先执行。
- ``load`` 方法务必实现得精简些，也就是要尽量减少其所执行的操作，因为整个应用程序在执行 ``load`` 方法时都会阻塞。

9.``+ (void)initialize`` 方法

- 对于每个类来说，该方法会在程序首次用该类之前调用，且只调用一次。它是惰性调用的，只有程序用到了相关的类时，才会调用。
- 此方法与 ``+ (void)load``方法不同的是，运行期系统在执行该方法时，是处于正常状态的，因此，从运行期系统完整度上来讲，此时可以完全使用并调用任意类中的任意方法。
- ``+ (void) initialize`` 方法与其他消息一样，如果某个类未实现它，而其超类实现了，那么就会运行超类实现的代码。

10.``NSTimer`` 会保留其目标对象。如下：

```
#import "YMTimerCircularReferenceViewController.h"

@interface YMTimerCircularReferenceViewController ()

@property (nonatomic, strong) NSTimer *timer;

@end

@implementation YMTimerCircularReferenceViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.timer = [NSTimer timerWithTimeInterval:1 target:self selector:@selector(timerSelector) userInfo:nil repeats:YES];
    [[NSRunLoop currentRunLoop] addTimer:self.timer forMode:NSDefaultRunLoopMode];
}

- (void)timerSelector {
    NSLog(@"%@",[NSDate date]);
}

- (void)dealloc {
    NSLog(@"YMTimerCircularReferenceViewController dealloc");
}
```

当 ``YMTimerCircularReferenceViewController`` pop 时，上面的 ``dealloc ``方法不被调用。

解决办法，加一个 NSTimer 类别：

```
#import "NSTimer+YMBlock.h"

@implementation NSTimer (YMBlock)

+ (NSTimer * _Nonnull)ym_timerWithTimeInterval:(NSTimeInterval)ti block:(nullable void (^)())block userInfo:(nullable id)userInfo repeats:(BOOL)yesOrNo {
    return [self timerWithTimeInterval:ti target:self selector:@selector(ym_blockInvoke:) userInfo:[block copy] repeats:yesOrNo];
}

+ (void)ym_blockInvoke:(NSTimer *)timer {
    void (^block)() = timer.userInfo;
    
    if (block) {
        block();
    }
}

@end
```

注：此处虽然依然有保留换，``self`` 引用 ``self``，因为是类对象，无须回收。



