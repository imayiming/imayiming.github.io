---
layout: post
cover: 'assets/images/cover11.jpeg'
navigation: True
title: "第 6 章 块与大中枢派发"
date: 2016-06-10 11:32:44 +0800
tags: iOS
subclass: 'post tag-speeches'
logo: 'assets/images/ghost.png'
author: yiming
categories: yiming
description: "块与大中枢派发"
---

0.如果块所捕获的变量是对象类型，那么就会自动保留它。系统在释放这个块的时候，也会将其一并释放。

1.如果将块定义在 Objective-C 类的实例方法中，那么除了可以访问类的所有实例变量之外，还可以使用 ``self`` 变量。块总能修改实例变量。所以在声明时无须加 ``__block``。不过，如果通过读取或写入操作捕获了实例变量，那么也会自动把 ``self`` 变量一并捕获了，因为实例变量是与 ``self`` 所只带的实例关联在一起的。

2.在 block 中 直接访问实例变量和通过 ``self`` 来访问是等效的。

3.一定要记住：``self`` 也是个对象，因而快在捕获它时也会将其保留。如果 ``self`` 所指点的那个对象同时也保留了块，那么这种情况通常就会导致“保留环”。

4.除了 “栈块” 和 “堆块”之外，还有一类块叫做 “全局块”（global block）。这种块不会捕捉任何状态（比如外围的变量等），运行时也无须有状态来参与。块所使用的整个内存区域，在编译期已经完全确定了，因此，全局块可以生命在全局内存里，而不需要在每次用到的时候于栈中创建。另外，全局块的拷贝操作是个空操作，因为全局块决不可能为系统所回收。这种块实际上相当于单例。

下面两种方式都是全局块（global block）：


```
void (^myFirstBlock)() = ^{
    NSLog(@"123");
};

void (^mySecondBlock)(int a,int b) = ^(int a,int b){
    NSLog(@"a + b = %@",@(a + b));
};
```

5.以 ``typedef`` 重新定义块类型，可令块变量用起来更加简单。

6.不妨为同一个块签名定义多个类型别名。如果要重构的代码使用了块类型的某个别名，那么只需修改相应 ``typedef`` 中的块签名即可，无须改动其他 ``typedef``。

7.与使用委托模式的代码相比，用块写出来的代码更为整洁。委托模式有个缺点：如果类要分别使用多个获取器下载不同数据，那么就得在 ``delegate`` 回调方法里根据传入的获取器参数来切换。

8.建议使用同一个块来处理成功与失败情况，苹果公司似乎也是这样设计 API 的。

9.如果块所捕获的对象直接或间接地保留了块本身，那么就得当心保留环问题。一定要找个适当的时机解除保留环，而不能把责任推给 API 的调用者。

10.在 Objective-C 中，如果有多个线程要执行同一份代码，那么有时可能会出现问题。这种情况下，通常要使用锁来实现某种同步机制。在 GCD 出现之前，有两种办法:

- 内置的 ``@synchronize``
- 使用 ``NSLock`` 对象

11.滥用 ``@synchronized(self)`` 则会降低代码效率，因为共用同一个锁的那些同步块，都必须按顺序执行。若是在 ``self`` 对象上频繁枷锁，那么程序可能要等另一段于此无关的代码执行完毕，才能继续执行当前代码，这样做其实并没有必要。在极端情况下，``@synchronize`` 块会导致死锁，另外，其效率也不见得很高。

12.派发队列可用来表述同步寓意，这样做法要比使用 ``@synchronized`` 块或 ``NSLock`` 对象更简单。

13.有种简单高效的办法可以代替同步块或锁对象，那就是使用“串行同步队列”。 

14.如果将串行同步队列改为串行异步队列，貌似看起来效率更高些，但这么改动有个坏处：如果你测一下程序性能，那么可能会发现这种写法比原来慢，因为执行异步派发时，需要拷贝块。若拷贝块所用的时间明显超过执行块所花的实现，则这种做法将比原来更慢。

15.多个获取方法可以并发执行，而获取方法与设置方法之间不能并发执行，利用这个特点，还能写出更快一些的代码块。利用``dispatch_brarrier_async``，我们可以使用并行队列。并发队列如果发现接下来要处理的块是个栅栏块。待栅栏块执行过后，再按正常方式继续向下处理。

16.多用 GCD，少用 ``performSelector``。

17.使用 ``performSelector``,编译器并不知道将要调用的选择子是什么，因此，也就不了解其方法签名及返回值，甚至连是否有返回值都不清楚。而且，由于编译器不知道方法名，所以就没办法运用 ARC 的内存管理规则来判定返回值是不是应该释放。鉴于此，ARC 采用了比较谨慎的做法，就是不添加释放操作。然而这么做可能导致内存泄漏，因为方法在返回对象时可能已经将其保留了。

18.延后执行可以用 ``dispatch_after`` 来实现。

19.GCD VS NSOperation

GCD 的优点：

- GCD 提供的 ``dispatch_after`` 支持调度下一个操作的开始时间而不是直接进入睡眠。
- ``NSOperation`` 中没有类似``dispatch_source_t``, ``dispatch_io,dispatch_data_t``, ``dispatch_semaphore_t`` 等操作。  

``NSOperation`` 的优点：

- GCD 没有操作依赖。我们可以让一个 Operation 依赖于另一个 Operation，这样的话尽管两个 Operation 处于同一个并行队列中，但前者会直到后者执行完毕后再执行；
GCD 没有操作优先级（GCD 有队列优先级），能够使同一个并行队列中的任务区分先后地执行，而在 GCD 中，我们只能区分不同任务队列的优先级，如果要区分block任务的优先级，也需要大量的复杂代码；
- GCD 没有 KVO。``NSOperation`` 可以监听一个 Operation 是否完成或取消，这样能比 GCD 更加有效地掌控我们执行的后台任务
- 在 ``NSOperationQueue`` 中，我们可以随时取消已经设定要准备执行的任务(当然，已经开始的任务就无法阻止了)，而 GCD 没法停止已经加入 queue 的 block (其实是有的，但需要许多复杂的代码)
- 我们能够对 ``NSOperation`` 进行继承，在这之上添加成员变量与成员方法，提高整个代码的复用度，这比简单地将 block 任务排入执行队列更有自由度，能够在其之上添加更多自定制的功能。

20.使用 Dispatch Group,并行执行 A、B 任务，最后执行 C 任务：

```
dispatch_queue_t concurrentQueue = dispatch_queue_create("me.iYiming.concurrentQueue", DISPATCH_QUEUE_CONCURRENT);

dispatch_group_t group = dispatch_group_create();

dispatch_group_async(group, concurrentQueue, ^{
   // A 任务
});

dispatch_group_async(group, concurrentQueue, ^{
   // B 任务
});

dispatch_group_notify(group, concurrentQueue, ^{
   // C 任务
});
```

21.使用 ``dispatch_once`` 创建单例：

单例：**保证只分配一次内存**  

- 调用 ``alloc`` 方法的时候，内部会调用 ``allocWithZone`` 方法，所以控制好 ``allocWithZone`` 方法的内存开辟操作就能控制 ``alloc`` 
- ``copy``、``mutableCopy`` 同样要控制，直接返回调用者就好（因为 ``copy`` 和 ``mutableCopy`` 是对象方法，所以如果第一次内存分配控制好了，这里直接返回 ``self``）  

- MRC 下 ``retain``、``release``、``retainCount`` 处理

```
  //保存单例对象的静态全局变量
  static id _instance;
  + (instancetype)sharedTools {
      return [[self alloc]init];
  }
  //在调用alloc方法之后，最终会调用allocWithZone方法
  + (instancetype)allocWithZone:(struct _NSZone *)zone {
      //保证分配内存的代码只执行一次
      static dispatch_once_t onceToken;
      dispatch_once(&onceToken, ^{
          _instance = [super allocWithZone:zone];
      });
      return _instance;
  }
  //这是个对象方法，既然有对象而且是单例，那么调用者就是这个单例对象了，那就返回调用的对象就行
  - (id)copyWithZone:(NSZone *)zone {
      return self;
  }
  //这是个对象方法，既然有对象而且是单例，那么调用者就是这个单例对象了，那就返回调用的对象就行
  - (id)mutableCopyWithZone:(NSZone *)zone {
      return self;
  }
  #if __has_feature(objc_arc)
  //如果是ARC环境
  #else
  //如果不是ARC环境

  //既然是单例对象，总不能被人给销毁了吧，一旦销毁了，分配内存的代码已经执行过了，就再也不能创建对象了。所以覆盖掉release操作
  - (oneway void)release {
  }
  //这是个对象方法，既然有对象而且是单例，那么调用者就是这个单例对象了，那就返回调用的对象就行
  - (instancetype)retain {
      return self;
  }
  //为了便于识别，这里返回 MAXFLOAT ，别的程序员看到这个数据，就能意识到这是单例。
  - (NSUInteger)retainCount {
      return MAXFLOAT;
  }
  #endif
```

22.``dispatch_sync`` 执行 block 所在的 Queue 如果和当前 Queue 是同一个 Queue，那么会造成死锁。比如：

```
- (void)viewDidLoad {
  [super viewDidLoad];
  
  dispatch_sync(dispatch_get_main_queue(),^{
    NSLog(@"Hello !");
  });
  
  NSLog(@"结束");
}
```

为什么？因为 ``dispatch_sync`` 是同步的，又在 ``viewDidLoad`` 里，所以主线程等待它执行完才能打印“结束“，然而 ``dispatch_sync`` 块需要在 ``dispatch_get_main_queue()`` 主线程里添加 block ``^{
    NSLog(@"Hello !");
  })``,因为主线程现在在等待中，所以 Block 永远无法添加进主线程队列中去，互相等待，从而造成死锁。