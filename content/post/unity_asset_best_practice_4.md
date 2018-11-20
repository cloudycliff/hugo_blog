---
title: "[译]Unity资源最佳实践4-AssetBundle使用模式"
date: 2018-10-27T14:16:26+08:00
draft: false
tags: [Unity3D]
categories: [translate]
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

合理的架构设计允许为项目发布新的或修正内容补丁，无论AssetBundle是否是一开始就安装好的。这方面的更多信息，参见Unity手册的[AssetBundle补丁](https://docs.unity3d.com/Manual/AssetBundles-Patching.html)。

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

向移动设备提供AssetBundle优先的方式是在应用安装后下载。这也能允许在安装后更新内容，而不必强制用户重新下载整个应用。在很多平台上，应用的二进制包必须经过一个高代价且冗长的重新审核流程。因此，开发一个安装后下载系统是很重要的。

最简单的提供AssetBundle的方法是把它们放在web服务器上，通过`UnityWebRequest`来下载。Unity会自动把下载的AssetBundle缓存在本地存储上。如果下载的AssetBundle是LZMA压缩的，Unity会以未压缩方式存储或者重新压缩为LZ4格式（根据`Caching.compressionEnable`设置来决定），以便后续更快的加载。如果下载的AssetBundle是LZ4格式，则会以压缩的方式存储。如果缓存空间满了，Unity会根据LRU来删除。下一节会对此有更细节的描述。

通常情况下推荐尽可能的使用`UnityWebRequest`，或者在Unity 5.2及以前的版本使用`WWW.LoadFromCacheOrDownload`。只有在内置API的内存消耗、缓存行为或性能不可接受时，或为了实现需求必须使用平台特定的代码时，才需要投入时间开发一个自定义的下载系统。

在以下情况下，使用`UnityWebRequest`或`WWW.LoadFromCacheOrDownload`是不够的：

* 对AssetBundle缓存需要细粒度的管理
* 项目需要使用一种自定义的压缩策略
* 项目希望使用平台特定的API来实现特殊的需求，例如需要在应用不活动时在后台下载数据（例如，使用iOS的后台任务API实现在应用不活动时下载数据）（译注：好像我只见过智龙迷城实现了这种功能，应用有更新时，切后台以后还可以继续下更新，但是不确定这个游戏是不是Unity做的。。）
* 在Unity不提供SSL支持的平台（如PC）上，需要通过SSL提供AssetBundle下载

### 4.2.3 内置缓存

Unity有一套内置的AssetBundle缓存系统，可以用来缓存通过`UnityWebRequest`下载的AssetBundle，此API还有一个带版本号的重载方法。这个版本号并不保存在AssetBundle内部，也不是由AssetBundle系统生成的。

缓存系统保存了传递给`UnityWebRequest`的最后一个版本号。当这个API以版本号为参数被调用时，缓存系统会通过比对版本号来检查是否有缓存的AssetBundle。如果版本号相同，系统会加载缓存的AssetBundle。如果版本号不同，或者没找到缓存的AssetBundle，Unity会去下载一份新的数据拷贝。这个新的拷贝会和新的版本号关联。

**在缓存系统中，AssetBundle只通过文件名来区分**，并不是通过它们下载的完成URL。这意味着同一个AssetBundle可以被放在不同的下载位置，例如CDN。只要文件名相同，缓存系统就会把它们识别为相同的AssetBundle。

每个应用应该负责制定一个合理的针对AssetBundle的版本号策略，用来在调用`UnityWebRequest`时传入。这个版本号可以来自唯一生成的id，例如CRC校验值。注意尽管`AssetBundleManifest.GetAssetBundleHash()`可能被用作此目的，我们不推荐使用这个函数做版本号，因为它提供的只是一个估计值，而不是真正的hash计算值。

更详细的信息请参考Unity手册的[用AssetBundle打补丁](https://docs.unity3d.com/Manual/AssetBundles-Patching.html)。

从Unity 2017.1以后，缓存API被扩展以提供更细粒度的控制，允许开发者从多个缓存中选择活跃的缓存。之前版本的Unity只能通过修改`Caching.expirationDelay`和`Caching.maximumAvailableDiskSpace`来删除缓存内容（这些变量仍然保留在Unity 2017.1的Cache类里）。

`Caching.expirationDelay`是AssetBundle被自动删除之前最少经过的秒数。如果在这段时间AssetBundle没有被访问过，就会被自动删除。

`Caching.maximumAvailableDiskSpace`指定了本地存储在通过`expirationDelay`规则删除缓存之前可用的字节数。在存储用尽后，Unity会删除最早打开（或通过`Caching.MarkAsUsed`标记过）的缓存。Unity会持续删除缓存直到有足够的空间来下载新的内容。

#### 4.2.3.1 事先准备缓存

因为AssetBundle只通过文件名来区分，随着应用安装包事先准备好缓存AssetBundle是可行的。可以把初始或基础版本的AssetBundle放到`/Assets/StreamingAssets/`目录下。这个流程和前文中和项目一起发布中描述的流程是一样的。

在应用第一次运行时，可以通过从`Application.streamingAssetsPath`加载AssetBundle来生成缓存。在这以后，应用可以正常的使用`UnityWebRequest`（UnityWebRequest也可以被用来第一次从StreamingAssets路径加载AssetBundle）。

### 4.2.3 自定义下载器

编写自定义下载器可以让应用对AssetBundle的下载、解压缩和存储有完全的控制。由于涉及到的开发工作并不轻松，我们只推荐大团队采用这种方式。在编写自定义下载器时，有四个主要方面需要考虑：

* 下载机制
* 存储位置
* 压缩方式
* 补丁机制

关于AssetBundle补丁，请参考[用AssetBundle打补丁](https://docs.unity3d.com/Manual/AssetBundles-Patching.html)。

#### 4.2.3.1 下载

对于大部分应用，HTTP是下载AssetBundle最简单的方式。不过，实现一个基于HTTP的下载器并不是个简单的任务。自定义下载器必须避免过多的内存分配，过量的线程使用和过量的线程唤醒。Unity的WWW类并不适合做此用途，在前文基础篇里有详尽的描述。

在编写自定义下载器时，有三个选择：

* C#的`HttpWebRequest`和`WebClient`类
* 自定义原生插件
* AssetStore包

##### 4.2.3.1.1 C#类

如果应用不需要HTTPS/SSL支持，C#的`WebClient`类提供了最简单的机制可以用来下载AssetBundle。它可以异步的把任何文件直接下载到本地存储，而不会有过多的托管内存分配。

要使用WebClient来下载AssetBundle，创建一个WebClient实例，把AssetBundle的URL和存储路径作为参数传进去。如果需要对请求的参数有更多的控制，可以用C#的`HttpWebRequest`类写一个下载器：

1. 通过`HttpWebResponse.GetResponseStream`获取一个字节流
2. 在栈上分配一段固定长度的buffer
3. 从响应流读取数据到buffer
4. 使用C#的`File.IO`API或其他流IO操作，把buffer写到硬盘中

##### 4.2.3.1.2 AssetStore包

一些asset store包提供了基于原生代码的通过HTTP、HTTPS或其他协议下载文件的实现。在为Unity编写自定义的原生插件之前，推荐评估一下AssetStore上现有的包。

##### 4.2.3.1.3 自定义原生插件

编写自定义原生插件最花费时间，但是为Unity下载数据提供了最灵活的方法。由于需要大量的编码时间和很高的技术风险，只推荐在其他方法都无法满足应用需求时采用这种方法。例如，在Unity不支持C# SSL的平台上，必须使用SSL通讯时，自定义原生插件才可能是必须的。

自定义原生插件一般是对目标平台的原生下载API的包装。如iOS的`NSURLConnect`和Android的`java.net.HttpURLConnection`。关于这些API的使用细节，请参考各平台的文档。

#### 4.2.3.2 存储

在所有平台上，`Application.persistentDataPath`指向了一个可写的位置，应该被用来存放持久化数据。在开发自定义下载器时，强烈推荐使用`Application.persistentDataPath`的子目录来存储下载数据。

`Application.streamingAssetPath`不可写，不应该被用来做AssetBundle缓存。以下是`streamingAssetPath`位置的举例：

* OSX： 在.app包中，不可写
* Windows： 在安装目录（如`Program Files`），一般不可写
* iOS： 在.ipa包中，不可写
* Android： 在.apk文件中，不可写

## 4.3 Asset分配策略

决定如何把项目的资源分配到AssetBundle中并不简单。采用简单策略很诱人，如把所有的Object放到自己的AssetBundle中或只使用一个AssetBundle，但是这些方案有很明显的缺点：

* AssetBundle太少，会增加运行时内存占用，增加加载时间，需要下载更大的包
* AssetBundle太多，会增加构建时间，让开发变得复杂，增加整体下载时间

如何把Object分组到AssetBundle中是很关键的决定。主要的策略有：

* 逻辑实体
* Object类型
* 同时使用的内容

关于这些分组策略的更多内容可以参考[文档](https://docs.unity3d.com/Manual/AssetBundles-Preparing.html)

## 4.4 常见的坑

本节描述了项目中使用AssetBundle时出现的一些常见问题。

### 4.4.1 Asset重复

Unity 5的AssetBundle系统会在Object被构建进AssetBundle时，寻找所有相关的依赖。这个依赖信息被用来决定需要放进AssetBundle的Object集合。

被明确指定到一个AssetBundle的Objects只会被构建进指定的AssetBundle。一个Object被“明确指定”，是指Object的AssetImporter的`assetBundleName`变量被设置成非空字符串。这可以在Unity Editor中，在Object的Inspector窗口里选择一个AssetBundle来实现，或者通过Editor脚本实现。

还可以通过[AssetBundle构建映射](https://docs.unity3d.com/ScriptReference/AssetBundleBuild.html)来把Objects分配到AssetBundle中，需要配合接收AssetBundleBuild数组为参数的重载方法[BuildPipeline.BuildAssetBundles()](https://docs.unity3d.com/ScriptReference/BuildPipeline.BuildAssetBundles.html)使用。

**任何没有明确分配到某AssetBundle中的Object会被包含到每个对这些未指明Object有引用的AssetBundle中**

举个例子，如果两个不同的Objects被分配到两个不同的AssetBundles中，但是它们都包含对一个公用Object的依赖，那么这个被依赖的Object就会被复制到两个AssetBundles中。这个重复的依赖也会被实例化，意味着这个被依赖Object的两个拷贝会被认为是不同的Object，拥有不同的id。这会增加应用AssetBundle的总体大小。如果应用加载了这两个父Objects，这个依赖Objects会在内存里加载两遍。

有几种方法可以解决这个问题：

1. 确保打到不同AssetBundle中的Objects不会有共同的依赖。任何有共同依赖的Objects可以放到同一个AssetBundle中来避免复制依赖。这个方法通常不适应于有很多共享依赖的项目。这会产生很大的单一AssetBundle，需要经常重新构建和下载，影响便利性和效率。
2. 合理划分AssetBundle，让共享依赖的AssetBundle不会被同时加载。这个方法可能会对一些特定类型的项目有效，比如按关卡划分的游戏。然而这仍然会不必要的增加项目AssetBundle的大小，增加构建和加载时间。
3. 确保所有的依赖资源都被构建到自己的AssetBundle中。这可以完全的消除重复资源的风险，但同时增加了复杂性。应用必须自己跟踪AssetBundle之间的依赖关系，同时确保在使用`AssetBundle.LoadAsset`之前，被依赖的AssetBundle都被加载了。

Object之间的依赖可以通过`UnityEditor`命名空间下的`AssetDatabase`API来获取。通过命名空间可以知道，这个API只在Editor中有效，而在运行时无效。`AssetDatabase.GetDependencies`可以被用来定位特定Object或Asset的所有直接依赖。注意这些依赖可能还会有它们自己的依赖。另外，`AssetImporter`API可以被用来查询某个Object被分配到了哪个AssetBundle中。

通过配合使用`AssetDatabase`和`AssetImporter`API，可以编写一个Editor脚本，来确保一个AssetBundle的直接或间接依赖都被分配到了某AssetBundle中，或者没有任何两个AssetBundle的共同依赖没有被打到AssetBundle中。由于重复资源会带来额外的内存占用，推荐所有的项目都编写这样的脚本。（译注：这里也没说明运行时如何获取依赖，也没提供这种脚本的实现，官方文档还真是懒。。。）

### 4.5.2 Sprite图集重复

任何自动生成的sprite图集都会被分配到生成它们的Sprite Objects所在的AssetBundle中。如果sprite Objects被分配到了多个AssetBundles中，这个sprite图集就不会被分配到AssetBundle中，然后就会重复（译注：应该是上一节中提到的那个原因）。如果Sprite Objects没有被分配到AssetBundle中，sprite图集也不会被分配到AssetBundle中。

为了确保sprite图集不会重复，需要确保所有被标记在同一个sprite图集中的sprites都被分配到同一个AssetBundle中。

注意在Unity 5.2.2p3及之前，自动生成的sprite图集不会被分配到AssetBundle中。因此，它们会被包含在任何包含或引用构成它们的sprites所在的AssetBundle中。处于这个问题，强烈建议使用Unity的sprite打包功能的项目更新到Unity 5.2.2p4，5.3或更新的版本。

### 4.5.3 Android材质

因为Android生态中存在严重的设备碎片化问题，通常有必要把贴图压缩成多种不同的格式。虽然所有的Android设备都支持ETC1，ETC1并不支持带透明通道的贴图。如果应用不要求支持OpenGL ES 2的话，最简单的解决方法是使用ETC2，它被所有Android OpenGL ES 3设备所支持。

大部分应用都需要发布到不支持ETC2的老设备上。解决这个问题的一个方法是通过Unity 5的AssetBundle变体（其他的解决方法，可以参考Unity的Android优化指导）。

要使用AssetBundle变体，所有不能使用ETC1压缩的贴图必须被拆分到一个只包含贴图的AssetBundle。下一步，为不支持ETC2的设备创建足够多的AssetBundle变体，使用设备特定的贴图压缩方式，如DXT5，PVRTC或ATITC。针对每一个变体，根据压缩格式对应的修改所包含的贴图的TextureImporter设置。

在运行时，可以使用`SystemInfo.SupportsTextureFormat`来检查所支持的不同贴图压缩格式。然后可以使用这个信息来根据支持的格式加载不同的AssetBundle变体。

获取关于Android贴图压缩格式的更多信息，可以参考[这里](http://developer.android.com/guide/topics/graphics/opengl.html#textures)。

### 4.5.4 iOS文件句柄滥用

**当前版本的Unity已经不受此问题的影响**

在Unity 5.3.2p2之前，Unity在AssetBundle加载后的整个使用期间，都会持有其文件句柄。这在大部分平台上都不是个问题。然而，iOS限制了一个进程最多可以同时打开255个文件句柄。如果加载了太多AssetBundle导致到达了上限，加载就会报错`Too Many Open File Handles`。

如果项目里拆分了几百或几千的AssetBundle，这就会成为一个问题。

对于不能更新Unity版本的项目，临时的解决方案有：

* 通过合并相关的AssetBundle来减少其数量
* 使用`AssetBundle.Unload(false)`来关闭AssetBundle的文件句柄，然后手动管理加载Objects的生命周期

## 4.5 AssetBundle变体

AssetBundle系统的一个关键特性是引入了AssetBundle变体。变体的目的是允许应用自行调整内容来更好的适应运行时环境。变体允许在不同的AssetBundle文件中的UnityEngine.Objects在加载和解析Instance ID引用时表现的像同一个Object。概念上，它允许两个UnityEngine.Objects在表现上共享同一个File GUID和Local ID，然后通过一个字符串Variant ID来识别实际需要加载的UnityEngine.Object。

这个系统有两个主要的用例：

1. 变体简化了为特定平台加载合适AssetBundle的流程
	* 举例：某构建系统可能会为DirectX 11 windows平台创建一个AssetBundle包含高分辨率贴图和复杂shader，和另一个为Android系统的包含低配内容的AssetBundle。在运行时，项目的资源加载代码可以根据平台加载合适的AssetBundle变体，传入AssetBundle.Load函数的Object名不需要改变。

2. 变体允许应用在同一平台上根据不同硬件加载不同内容
	* 这是支持广大移动设备的关键功能。iPhone 4是无法以同样的精确度展示可以在最新版iPhone上展示的内容的。
	* 在Android上，AssetBundle变体可以用来处理屏幕分辨率比例和DPI碎片化问题。

### 4.5.1 限制

AssetBundle变体的一个主要限制是它要求变体通过不同的Asset来构建。即使这些Asset只是存在导入设置上的差异也受限于此。如果要构建到变体A和变体B的贴图的唯一区别是在Unity贴图导入设置中选择的贴图的压缩算法，变体A和变体B仍然必须是完全不同的两个Asset。这意味着变体A和变体B必须是硬盘上的不同文件。

这个限制让大项目的版本管理变得复杂，因为某Asset的多个版本拷贝都需要保存在版本控制中。当开发者像修改Asset的内容时，每个拷贝都需要被修改。对于这个问题，没有内置的解决办法。

大多数团队实现了他们自己的AssetBundle变体形式。通过给AssetBundle文件名添加合理定义的后缀，来区分给定的AssetBundle代表哪个变体。通过自定义的代码，以程序的方式在构建AssetBundle时对所包含的Assets的导入设置进行修改。一些开发者还扩展了自定义的系统，允许修改挂在prefab上的组建的参数。

## 4.6 压缩还是不压缩

是否压缩AssetBundle需要基于几个重要的点来考虑：

* 加载时间：未压缩的AssetBundle比压缩的加载的更快
* 构建时间：LZMA和LZ4算法在压缩文件时非常缓慢，而且Unity Editor以串行的方式处理AssetBundle。项目里如果有大量的AssetBundle，将会花费很多时间在压缩上
* 应用大小：如果AssetBundle随应用发布，压缩会减少应用的总大小。或者也可以通过后下载的方式提供AssetBundle
* 内存占用：在Unity 5.3之前，Unity的解压机制要求在解压之前，把整个压缩的AssetBundle都加载到内存中。如果内存使用情况很关键，可以使用未压缩或LZ4压缩方式
* 下载时间：只有在AssetBundle很大，或用户处于带宽受限场合（如以低速下载或流量受限），压缩才可能是必须的。如果只是在高速连接的PC上提供几M的数据，是完全可以忽略压缩的

### 4.6.1 Crunch压缩

主要由DXT压缩贴图构成的，使用了Crunch压缩算法的AssetBundle，不应该再使用其他压缩方式。

## 4.7 AssetBundle和WebGL

由于当前的Unity WebGL导出不支持工作线程，所有WebGL项目的AssetBundle加载和解压缩都必须在主线程上进行。下载AssetBundle被代理到了浏览器的XMLHttpRequest中。下载完成后，压缩的AssetBundle会在主线程解压，然后会根据包的大小来阻塞其他内容的执行。

Unity推荐开发者采用小的包来避免可能会引起的性能问题。这种方式会比使用大包更高效的利用内存。Unity WebGL只支持LZ4压缩和未压缩格式，然而，可以在生成的包上再应用gzip/brotli压缩方式。这种情况下，你应该配置你的web服务器，以便浏览器在下载文件时进行解压。更多细节参考[这里](https://docs.unity3d.com/Manual/webgl-deploying.html)。

如果你使用的是Unity 5.5及以后版本，考虑避免使用LZMA方式转而使用LZ4方式，它采用更高效的按需解压方式。Unity 5.6已经针对WebGL平台移除了LZMA压缩选项。