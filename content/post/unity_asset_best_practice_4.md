---
title: "Unity_asset_best_practice_4"
date: 2018-10-27T14:16:26+08:00
draft: true
---

> 本篇是对Unity官方最佳实践的翻译，原文地址[https://unity3d.com/learn/tutorials/topics/best-practices/assetbundle-usage-patterns](https://unity3d.com/learn/tutorials/topics/best-practices/assetbundle-usage-patterns) 


# AssetBundle使用模式

在前一章讨论了AssetBundle的基础，包括各种加载API的底层行为。本章讨论了在AssetBundle的实际使用过程中出现的各方面问题及可行的解决办法。

## 4.1 管理加载的Assets

在内存敏感的环境中，控制已加载资源的大小和数量是非常关键的。在Objects被从当前场景中移除时，Unity不会自动unload它们。Asset的清理工作会在特定的时间触发，并且可以手动触发。

AssetBundle本身必须被小心的管理。被从本地硬盘上加载的AssetBundle（无论是在Unity缓存中还是通过`AssetBundle.LoadFromFile`加载）有着最小的内存占用，很少占用超过几k的空间。然而，如果存在大量的AssetBundle，这部分内存占用仍然会成为问题。

大部分游戏都允许玩家重复体验某部分内容（比如重玩某关卡），知道在什么时候加载和卸载AssetBundle是十分重要的。如果AssetBundle被不恰当的卸载，可能会造成内存中重复存在某Object。不恰当卸载AssetBundle也可能会在特定环境下造成不符合预期的行为，比如贴图丢失。可以参照前文来了解为什么会产生这种情况。

在管理Asset和AssetBundle时，最重要的一点是要理解在调用`AssetBundle.Unload`时，参数`unloadAllLoadedObjects`传入`true`或`false`所产生的不同行为。

这个API在调用时，会卸载AssetBundle的头部信息。参数`unloadAllLoadedObjects`决定了是否要同时卸载从这个AssetBundle中实例化的所有Objects。如果设置为`true`，所有源自这个AssetBundle的Objects都会被立刻卸载，即使它们在当前场景中仍在使用。

举个例子，假设材质M是从AssetBundle AB中加载出来的，并且假设M在当前场景中被使用。

![ab2a](/img/ab2a.jpg)

如果调用了`AssetBundle.Unload(true)`，M就会从当前场景删除，销毁并卸载。另一方面，如果`AssetBundle.Unload(false)`被调用，AB的头部信息会被卸载，但是M仍在留在场景中并且保持可用的状态。调用`AssetBundle.Unload(false)`破坏了M和AB之间的联系。如果后面AB被再次加载，AB中包含的Objects会被重新初始化到内存中。

![ab2b](/img/ab2b.jpg)

如果AB被再次加载，一份AssetBundle的头部信息的新的拷贝会被重新加载。然而，M并不是从这个新的AB中加载进来的。Unity不会再AB的新拷贝和M之间建立新的链接。

![ab2c](/img/ab2c.jpg)

如果调用`AssetBundle.LoadAsset()`来重新加载M，Unity不会认为之前的M是AB的一个数据实例。因此，Unity会加载一份新的M，此时场景中就会有两份一样的M。

![ab2d](/img/ab2d.jpg)

在大部分项目中，这个行为是不受欢迎的。大部分项目都应该使用`AssetBundle.Unload(true)`，并使用一种方法来保证Objects不会重复。两个常用的方法为：

* 在应用的生命周期中，有一些明确定义的点用来卸载临时的AssetBundle，比如切换关卡或在加载界面。这是一种简单且被广泛使用的选择。
* 为每个单独的Objects保持一个引用计数，并且只在AssetBundle所关联的Objects都没有在使用时卸载它们。这允许应用在不产生内存副本的情况下卸载和重新加载单独的Objects。

如果应用必须使用`AssetBundle.Unload(false)`，单独的Objects只能通过以下两个方法被卸载：

* 在场景和代码中，消除对Object的所有引用。然后调用`Resources.UnloadUnusedAssets`。
* 使用非叠加方式加载场景。这会销毁当前场景中的所有Objects，并且会自动执行`Resources.UnloadUnusedAssets`。

如果项目中有明确定义的用户等待点用来加载和卸载Objects，例如在游戏模式或场景的切换时，在这些点应该尽可能的卸载不需要的Objects并且加载新的Objects。

实现这种功能最简单的办法是把项目按照场景进行划分，然后把这些场景及依赖打包到AssetBundle中。此时应用可以进入一个加载界面，完整的卸载包含老场景的AssetBundle，然后加载包含新场景的AssetBundle。

尽管这是一个最简化的流程，一些项目需要更复杂的AssetBundle管理。因为每个项目的情况都不相同，没有一个通用的AssetBundle设计模式。

当决定如何把Objects分组放到AssetBundle中时，最好从需要同时加载或更新的资源开始，把他们放到同一个AssetBundle中。例如，考虑一个RPG游戏。独立的地图和过场动画可以按照场景来分组到AssetBundle中，但是有些Objects是需要被大多数场景用到的。可以单独打一些AssetBundle来提供立绘、游戏内UI、不同的角色模型和贴图。后面这些Objects可以被分组到一些AssetBundle中，在启动时就加载然后在app整个生命周期中都保持加载。

另一个可能出现的问题是，当AssetBundle被卸载，Unity必须重加载AssetBundle中的Object时。在这种情况下，重加载会失败，在Unity Editor的层级视图里Object会表现为`Missing`状态。

这主要出现在Unity失去然后重新获取对图形上下文的控制时，例如当移动应用被暂停或用户锁屏了电脑时。在这种情况下，Unity必须重新上传贴图和shader到GPU。如果这些asset所来源的AssetBundle不可用，应用就会把这些Objects渲染为洋红色。（译注：这里还是没说要怎么处理这种情况？）

## 4.2 发布

有两个基本的途径可以把项目的AssetBundles发布给客户端：和项目一起安装或在安装后下载。

是否把AssetBundle一起安装还是安装后下载，是由目标平台的容量和限制决定的。移动项目通常采用安装后下载的形式，来减少初始安装的大小，使其保持在无线下载的限制以内。主机和PC项目通常把AssetBundles和安装包一起发布。

合理的架构设计允许为项目发布新的或修正内容补丁，无论AssetBundle是否是一开始就安装好的。这方面的更多信息，参见Unity手册的[AssetBundle补丁](https://docs.unity3d.com/Manual/AssetBundles-Patching.html?_ga=2.136884912.1651421058.1540542712-812634850.1540294356)。

### 4.2.1 和项目一起发布

把AssetBundles和项目一起发布是最简单的方式，因为不需要额外的下载管理代码。AssetBundles和项目一起发布通常有两个主要的原因：

* 减少项目构建时间，允许开发时更简单的迭代。如果AssetBundle不需要单独的进行更新，它们可以被放到Streaming Assets中。下文会有进一步的解释。
* 发布一个可更新内容的最初版本。这通常用来节省用户初始安装后的时间或用来提供一个基础的版本用来后续更新。Streaming Assets并不适合做这种用途。然而，当编写一个下载和缓存系统不可行时，可更新内容的初始版本可以从Streaming Assets中加载到Unity缓存，下文会有进一步的解释。

#### 4.2.1.1 Streaming Assets

在Unity应用安装时就提供内容（包括AssetBundles）的最简单的办法，是在构建项目前把它们放到`/Assets/StreamingAssets/`目录下。在构建时任何放在`StreamingAssets`下的内容都会被复制到最终的应用中。

在运行时，可以通过`Application.streamingAssetsPath`来获取StreamingAssets目录的完整路径。在大部分的平台上，AssetBundle可以通过`AssetBundle.LoadFromFile`来加载。

针对安卓开发者：在安卓系统上，放在StreamingAssets目录下的资源会被存储到APK中，如果被压缩的话需要更多的时间来加载，因为放在APK中的文件会采用不同的存储算法。实际使用的算法可能会根据Unity版本的不同而改变。你可以使用解压工具（如7-zip）打开APK来判断里面的文件是否被压缩了。如果被压缩了，`AssetBundle.LoadFromFile()`会执行的很缓慢。在这种情况下，你可以使用`UnityWebRequest.GetAssetBundle`来加载缓存的版本作为变通。通过使用UnityWebRequest，AssetBundle会在第一次加载时被解压缩并缓存，使后续的执行更快一些。注意这样会占用更多的存储空间，因为AssetBundle会被复制到缓存中。另一种方法，你可以导出你的Gradle工程，在编译时给AssetBundle添加一个扩展名。你可以编辑`build.gradle`文件，把这个扩展名添加到`noCompress`段。做完这些以后，你应该就可以使用`AssetBundle.LoadFromFile()`而不用担心压缩引起的效率问题。

注意：在部分平台上，Streaming Assets不是一个可写的路径。如果项目的AssetBundle需要在安装后更新，使用`WWW.LoadFromCacheOrDownload`或写自己写一个下载器。

### 4.2.2 安装后下载

