---
layout: post
cover: 'assets/images/cover11.jpeg'
navigation: True
title: "第 5 章 内存管理"
date: 2016-06-09 11:29:20 +0800
tags: iOS
subclass: 'post tag-speeches'
logo: 'assets/images/ghost.png'
author: yiming
categories: yiming
description: "内存管理"
---

下图列出了一些 ARC 下的内存问题，就各个问题一一描述下： 

![2.jpg](/images/effective_objectivec_2.0/2.png)
 

## 循环引用

#### 普通的两个变量互相引用

上代码：

```
self.object1 = [[YMNormalCircularReferenceObject alloc] init];
self.object2 = [[YMNormalCircularReferenceObject alloc] init];

self.object1.data = self.object2;
self.object2.data = self.object1;
```

两个对象互相引用，解决办法一个使用 ``weak`` 标识属性。


#### Block 循环引用

```
#import "YMBlockCircularReferenceViewController.h"

typedef void(^YMDownloaderCompleteBlock)(id data);

@interface YMDownloader ()

@property (nonatomic, copy) YMDownloaderCompleteBlock downloaderCompleteBlock;

@end

@implementation YMDownloader


- (void)downloadDataWithURL:(NSURL *)url completeBlock:(YMDownloaderCompleteBlock)block {
    self.downloaderCompleteBlock = block;
    
    [self downloadDataWithURL:url];
}

- (void)downloadDataWithURL:(NSURL *)url {
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        sleep(2);
        
        self.downloaderCompleteBlock(url);
    });
}

@end


@interface YMBlockCircularReferenceViewController ()

@property (nonatomic, strong) YMDownloader *downloader;
@property (nonatomic, strong) id data;

@end

@implementation YMBlockCircularReferenceViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.downloader = [[YMDownloader alloc] init];
    [self.downloader downloadDataWithURL:[NSURL URLWithString:@""] completeBlock:^(id data) {
        self.data = data;
    }];
}

- (void)dealloc {
    NSLog(@"YMBlockCircularReferenceViewController dealloc 方法");
}

@end
```

``self`` 保留了 ``downloader`` ，``downloader`` 拷贝了块，块里保留了 ``self``,解决办法：

```
- (void)downloadDataWithURL:(NSURL *)url {
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        sleep(2);
        
        self.downloaderCompleteBlock(url);
        self.downloaderCompleteBlock = nil;
    });
}
```

#### NSTimer

``NSTimer`` 会保留其目标对象

继续上代码：

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

解决办法，加一个 ``NSTimer`` 类别：

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

注：此处虽然依然有保留换，``self`` 引用 ``self``，因为是类对象，无须回收。``CADisplayLink`` 类似。

#### 悬挂指针

在 Scheme 中开启即可，如下图：

![3.jpg](/images/effective_objectivec_2.0/3.png)

## 持有、释放不匹配

#### performselector

使用 ``performSelector``,编译器并不知道将要调用的选择子是什么，因此，也就不了解其方法签名及返回值，甚至连是否有返回值都不清楚。而且，由于编译器不知道方法名，所以就没办法运用 ARC 的内存管理规则来判定返回值是不是应该释放。鉴于此，ARC 采用了比较谨慎的做法，就是不添加释放操作。然而这么做可能导致内存泄漏，因为方法在返回对象时可能已经将其保留了。

#### CoreFoundation - Foundation

上代码：

```
- (IBAction)coreFoundationToFoundation:(id)sender {
   CFStringRef coreFoundationStr = CFStringCreateWithCString(NULL, "Hello World!", kCFStringEncodingUnicode);
    
    NSString *foundationStr = (__bridge NSString *)(coreFoundationStr);
    // NSString *foundationStr = CFBridgingRelease(coreFoundationStr);
    
    NSLog(@"foundationStr:%@",foundationStr);
}

- (IBAction)foundationToCoreFoundation:(id)sender {
    NSString *foundationStr = [[NSString alloc] initWithFormat:@"Hello World!"];
    
    CFStringRef coreFoundationStr = CFBridgingRetain(foundationStr);
    NSLog(@"coreFoundationStr:%@",coreFoundationStr);
}

- (IBAction)noMemoryManagement:(id)sender {
    // NSString *foundationStr = @"Hello World!";
    // CFStringRef coreFoundationStr = (__bridge CFStringRef)(foundationStr);
    
    
    // CFStringRef coreFoundationStr = CFStringCreateWithCString(NULL, "Hello World!", kCFStringEncodingUnicode);
    // NSString *foundationStr = (__bridge NSString *)(coreFoundationStr);
}
```

Core Foundation 与 Foundation 内存问题


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

所有权被转移的同时，被转换变量将失去对象的所有权。当 Core Foundation 对象类型向 Objective-C 对象类型转换的时候，会经常用到 ``__bridge_transfer`` 关键字。

同样，我们可以使用 ``CFBridgingRelease()`` 来代替 ``__bridge_transfer`` 关键字。  

```
CFStringRef cfString = CFStringCreate...();
NSString *string = CFBridgingRelease(cfString);
```

--- 华丽分割线 ---

其实看完上面的解释，就不用介绍上面代码出现的问题了。简要说下：

0.``- (IBAction)coreFoundationToFoundation:(id)sender`` 方法创建了 CFStringRef 对象，并未使用，需要使用下面代码释放：

```
NSString *foundationStr = CFBridgingRelease(coreFoundationStr);
```

1.``- (IBAction)foundationToCoreFoundation:(id)sender`` 方法里接手并``retain`` 了 Objective-C 对象的字符串，但是没有做到释放。

#### @try ... @catch

再上代码：

```
- (void)viewDidLoad {
    [super viewDidLoad];
    
    @try {
        NSArray *array = @[@"a", @"b", @"c"];
        [array objectAtIndex:3];
    } @catch (NSException *exception) {
        // 处理异常
        NSLog(@"throw an exception: %@", exception.reason);
    } @finally {
        NSLog(@"正常执行");
    }
}
```
当执行到 ``[array objectAtIndex:3];`` 发生崩溃，这时 ``array`` 并未释放，解决办法是开启编译器标志 ``-fobjc-arc-exceptions``


## 其他

#### @autoreleasepool block 降低内存峰值

```
for (id object in array) {
	@@autoreleasepool {
		...
	}
}
```

合理运用自动释放池，可降低应用程序的内存峰值。

是否应该用 ``@autoreleasepool { }`` 来优化效率，完全取决于具体的应用程序。首先得监控内存用量，判断其中有没有需要解决的问题，如果没完成这一步，那就别急着优化。尽管``@autoreleasepool { }`` 的开销不太大，但毕竟还是有的，所以尽量不要建立额外的自动释放池。

#### 需自己负责释放方法命名规则

在一开始的图中的几个方法，需要在 MRC 下需要对象自己手动释放。