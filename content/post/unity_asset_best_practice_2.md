---
title: "Unity_asset_best_practice_2"
date: 2018-10-23T18:57:29+08:00
draft: false
tags: [Unity3D]
categories: [translate]
---

> 本篇是对Unity官方最佳实践的翻译，原文地址[https://unity3d.com/learn/tutorials/topics/best-practices/resources-folder](https://unity3d.com/learn/tutorials/topics/best-practices/resources-folder) 


Resources目录
=

本章讨论了Resources系统。这个系统允许开发者在一个或多个Resources目录下存储资源，并能通过Resources API在运行时加载或卸载资源。

2.1 Resources系统最佳实践
==

**不要使用它。。。**

这个强烈的建议基于以下几个原因：

* 使用Resources目录会导致细粒度的内存管理更加困难。
* 对Resources目录不合理的使用，会增加应用的启动时间和构建时间。随着Resources目录的增加，管理这些目录下的资源会变得更加困难。（译注：这些资源会在应用启动时都被加载，所以一般只会在开发时使用，出包时会使用AssetBundle方式）
* Resources系统降低了针对不同平台提供不同内容的能力，也限制了后续资源更新的能力。AssetBundle是Unity针对不同设备提供不同内容的主要工具。（译注：Resources下的资源会直接进包，而且不可修改，所以很难差异化和更新，AssetBundle在这方面就更加灵活，当然使用起来也更加复杂。TODO：调研如何在开发时使用Resources，在发布时使用AssetBundle并不影响开发体验，是下一步的工作重点）

2.2 Rescources系统的合理使用场合
==

在以下两个场景中，可以在不破坏开发体验的情况下使用Resources系统：

* Resources系统使用起来比较简单，在快速构建原型阶段是非常好的工具。然而当项目向发布阶段转移时，需要限制Resources目录的使用。
* 如果内容满足以下情况，Resources系统可能会很有用：在项目整个生命周期都会被使用到；不会占用很多内存；不太会被修改，或者不会随不同平台或设备有差异；被用来进行最小化的启动。

这种使用情况的一些常见例子包括：管理prefabs的MonoBehaviour单例（TODO：这是啥？？），包含第三方配置信息的ScriptableObjects文件，如Facebook App ID等。

2.3 Resources序列化
==

在项目构建时，在所有命名为Resources目录下的Assets和Objects都会被合并到一个序列化的文件中（译注：无论你的Resources放在哪个层级，只要叫这个名字，都会被合并。不是很清楚Unity是怎么处理重名的问题的，而且好像如果重名的话，加载时如果不指定类型是会有问题的）。这个文件也包含metadata文件和索引数据，和AssetBundle类似（译注：其实感觉Resources就是一个特殊的内置的AssetBundle）。这个索引包含了一个序列化的查找树，用来把一个Object的名字解析成对用的File GUID和Local ID。它还包含了Object在序列化文件中的数据偏移量。

在大部分平台上，这个查找树数据结构是一个平衡搜索树，构建时间以O(nlog(n))增长。这会导致索引的构建时间随着Resources目录下的Objects数量的增长以非线性的速度增长。

这个操作发生在app启动阶段，在splash界面展示的时候，并且无法被跳过。在低端移动设备上，加载包含10000个资源的Resources系统会消耗好几秒钟的时间，即使这些资源并不一定会在app的第一个场景中被使用。
