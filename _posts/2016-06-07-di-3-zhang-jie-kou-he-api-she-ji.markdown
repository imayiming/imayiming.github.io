---
layout: post
cover: 'assets/images/cover11.jpeg'
navigation: True
title: "第 3 章 接口和 API 设计"
date: 2016-06-07 11:25:23 +0800
tags: iOS
subclass: 'post tag-speeches'
logo: 'assets/images/ghost.png'
author: yiming
categories: yiming
description: "接口和 API 设计"
---

0.Objective-C 没有其他语言那种内置的命名空间（namespace）机制。鉴于此，我们在起名时要设法避免潜在的命名冲突，否则很容易就重名了。如果发生命名冲突，那么应用程序的链接过程就会出错，因为其中出现了重复符号,``duplicate symbol``错误。

1.避免此问题的唯一办法就是变相实现命名空间：为所有名称都加上适当前缀。使用 Cocoa 创建应用程序时一定要注意，Apple 宣称其保留使用所有“两字母前缀”的权利，所以自己选用的前缀应该是三个字母的。

2.不仅是类名，应用程序中的所有名称都应该加上前缀。如果要既有类新增 category，那么一定要给 “分类” 及分类中方法加上前缀。

3.在类中提供一个全能初始化方法，并于文档里指明。其他初始化方法均应调用此方法，若全能初始化方法与超类不同，则需复写超类中的对应方法（所谓的全能初始化方法就是，其他初始化方法都要调用它，如 ``NSDate`` 中的 ``initWithTimeIntervalSinceReferenceDate:`` 就是全能初始化方法）。

4.NSObject 协议中还有个方法要注意，那就是 ``debugDescription``，此方法的用意与 ``description`` 非常相似。二者区别在于，``debugDescription`` 方法是开发者在调试器（debugger）中以控制台命令打印对象时才调用的。在 ``NSObject`` 类的默认视线中，此方法知识直接调用了 ``description``。

5.实现 ``description`` 方法返回一个有意义的字符串，用以描述该实例。若想在调试时打印出更详尽的对象描述信息，则应实现 ``debugDescription`` 方法.

6.默认情况下，属性是“既可读又可写的”，这样设计出来的类都是“可变的”。一般情况下我们要建模的数据未必需要改变。具体到编程实践中，则应该尽量把对外公布出来的属性设为只读，而且只在确有必要时才将属性对外公布。再将属性在对象内部重新声明为 ``readwrite``。

7.给私有化方法名称添加前缀（``p_method``, ``p`` 表示私有）。苹果公司喜欢单用一个下划线作为私有方法的前缀。你也许也想照着苹果公司的办法指哪一个下划线作前缀，这样做可能会惹来大麻烦：如果苹果公司提供的某个类中继承了一个子类，那么你在子类里可能会无意间复写了父类的同名方法，鉴于此，苹果公司在文档中说，开发者不应该单用一个下划线做前缀。

8.若想令自己缩写的对象具有拷贝功能，则需要实现 ``NSCoping`` 协议

9.如果自订一个对象分为可变版本与不可变版本。那么就要同事实现 ``NSCoping`` 与 ``NSMutableCopying`` 协议。

10.在可变对象上调用 ``copy`` 方法会返回另外一个不可变类的实例。
```
[NSMutableArray copy] => NSArray
[NSArray mutableCopy] => NSMutableArray
```