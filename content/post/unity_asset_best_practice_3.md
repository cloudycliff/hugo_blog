---
title: "Unity_asset_best_practice_3"
date: 2018-10-23T18:57:32+08:00
draft: false
tags: [Unity3D]
categories: [translate]
---

> 本篇是对Unity官方最佳实践的翻译，原文地址[https://unity3d.com/learn/tutorials/topics/best-practices/assetbundle-fundamentals](https://unity3d.com/learn/tutorials/topics/best-practices/assetbundle-fundamentals) 


# AssetBundle基础


本章讨论了AssetBundle。介绍了AssetBundle所基于的基础系统，还有和AssetBundle交互的核心API。详细来说，讨论了加载和卸载AssetBundle本身，以及加载和卸载AssetBundle中特定的Asset和Object。

## 3.1 总览

AssetBundle系统提供了一种Unity可以索引和序列化的，把多个文件存储到一个归档格式的方法。AssetBundle是Unity在应用安装后投递和更新非代码内容的主要工具。这允许开发者提交一个小的应用包，最小化运行时内存压力，根据用户设备有选择的加载最优的资源。

理解AssetBundle的工作原理是为移动设备构建成功项目的关键。可以通过[AssetBundle文档](https://docs.unity3d.com/Manual/AssetBundlesIntro.html?_ga=2.126189709.184694638.1540294356-812634850.1540294356)获取关于AssetBundle的整体描述。

## 3.2 AssetBundle布局

总的来说，一个AssetBundle由两部分组成：头部和数据部。

头部包含AssetBundle的信息，如id，压缩类型和manifest文件。这个manifest文件是一个以Object名为key的查询表。其中的每一条都提供了一个查询数据位置的索引。在大部分平台上，这个查询表以平衡搜索树的形式实现。特殊的，Windows和OSX平台（包括iOS）使用的是红黑树。因此，构建manifest的时间会以超过线性的速度随资源数增长而增长。

数据部包含了Asset序列化后的数据。如果使用的是LZMA压缩方式，所有的资源会一起压缩。如果使用了LZ4压缩方式，每个资源会单独进行压缩。如果不压缩，数据部就是原始的数据流。

在Unity 5.3之前，AssetBundle中的Objects不能被单独压缩。后果就是如果5.3之前版本的Unity需要从一个压缩了的AssetBundle中读取一个Objects，Unity必须把整个AssetBundle解压缩。通常，Unity会缓存解压后的AssetBundle来加快后续加载的速度。

## 3.3 加载AssetBundle

AssetBundle可以通过4个不同的API加载。这4个API以两个标准区分：

* AssetBundle是LZMA压缩、LZ4压缩还是不压缩
* AssetBundle加载的平台（译注：这里的平台应该是指从本地还是从网络加载？）

这些API是：

* AssetBundle.LoadFromMemory(Async optional)
* AssetBundle.LoadFromFile(Async optional)
* UnityWebRequest中的DownloadHandlerAssetBundle
* WWW.LoadFromCacheOrDownload (Unity 5.6及以后版本)

### 3.3.1 AssetBundle.LoadFromMemory(Async)

**Unity不建议使用这个API**

`AssetBundle.LoadFromMemoryAsync`从一个托管字节数组（C#中的byte[]）里加载AssetBundle。它总是从托管字节数组中复制出数据，拷贝到一块新分配的、连续的原生内存块中。如果AssetBundle以LZMA压缩，它会在复制时进行解压。未压缩和LZ4压缩的AssetBundle会被原封不动的复制。

这个API所消耗的最高内存至少是AssetBundle大小的2倍：一份是原生内存中创建出来的内存，一份是托管代码里传给API的字节数组。通过这个API加载的Asset则会在内存中被复制3遍：一份在托管代码字节数组，一份在原生内存，第三份是asset自身占用的GPU或系统内存。（TODO：这段所说的托管代码，应该是指C#和C交互所做的转换？这部分待进一步研究）

在Unity 5.3.3之前，这个API叫做`AssetBundle.CreateFromMenory`。其功能没有改变。

### 3.3.2 AssetBundle.LoadFromFile(Async)

`AssetBundle.LoadFromFile`是一个高效的API，用来从本地存储（如硬盘、SD卡）加载未压缩或LZ4压缩的AssetBundle。

在桌面系统、游戏主机和移动平台上，这个API只会加载AssetBundle的头部，把数据部分留在硬盘上。只有在调用加载方法（如AssetBundle.Load）或Object的Instance ID被解析时，AssetBundle的数据才会被加载。这种场景下，不会产生额外的内存占用。在Unity Editor中，这个API会把整个AssetBundle加载进内存，和`AssetBundle.LoadFromMemoryAsync`的加载方式类似。如果在Editor模式下进行profile，这个API会造成内存的尖峰。这应该不会影响在设备上的性能表现，在采取修复措施之前，应该在设备上针对这些尖峰再次进行测试。

注意：在Unity 5.3及之前，Android设备上调用这个API从streaming asset path加载AssetBundle会出错。这个问题已经在Unity 5.4修复。

在Unity 5.3之前，这个API叫做`AssetBundle.CreateFromFile`。其功能没有发生变化。

### 3.3.3 AssetBundleDownloadHandler

`UnityWebRequest`API允许开发者详细的指定如何处理下载数据，来消除不必要的内存使用。最简单的下载AssetBundle的方式是`UnityWebRequest.GetAssetBundle`。

在本教程中，我们重点关注`DownloadHandlerAssetBundle`。通过一个工作线程，它把下载数据流入一个定长的buffer中，然后根据DownloadHandle的配置，把缓存的数据传输到临时存储中或AssetBundle缓存中。所有这些操作都发生在原生代码中，排除了扩展托管堆的风险。另外，这个DownloadHandler不保持下载数据的原生数据拷贝，进一步减少了下载AssetBundle的内存占用。

LZMA压缩的AssetBundle会在下载过程中解压，并使用LZ4压缩方式缓存。这个行为可以通过`Caching.CompressionEnabled`配置来修改。

当下载完成时，可以通过DownloadHandler的`assetBundle`属性获取到下载AssetBundle的访问，和调用了`AssetBundle.LoadFromFile`类似。

如果所请求的AssetBundle已经存在于Unity的缓存中，这个AssetBundle可以立即被访问，和调用`AssetBundle.LoadFromFile `的效果一样。

在Unity 5.6之前，UnityWebRequest系统使用一个固定大小的工作线程池和内部的job系统来防止出现过重的并发下载。线程池的大小不可配置。在Unity 5.6，这个安全措施被移除，来适应更现代的硬件，并允许使用更快的HTTP相应码和头。

### 3.3.4 WWW.LoadFromCacheOrDownload

**注意：从Unity 2017.1开始，WWW.LoadFromCacheOrDownload只是简单的包装了一下UnityWebRequest。因此，使用Unity 2017.1以后的版本进行开发应该使用UnityWebRequest，WWW.LoadFromCacheOrDownload在未来会被弃用**

下面的信息适用于Unity 5.6及更老的版本。

`WWW.LoadFromCacheOrDownload`允许从远程服务器和本地存储加载Objects。可以通过`file://`从本地加载文件。如果AssetBundle已经存在于Unity缓存中，这个API和`AssetBundle.LoadFromFile`一致。

如果AssetBundle没有被缓存，`WWW.LoadFromCacheOrDownload`会从源文件读取数据。如果AssetBundle被压缩，它会使用工作线程进行解压并写入到缓存中。如果未压缩，则直接通过工作线程写入缓存。一旦AssetBundle被缓存，这个API会从缓存中加载头部信息。然后的表现就和`AssetBundle.LoadFromFile`完全一致。这个缓存是由`WWW.LoadFromCacheOrDownload`和`UnityWebRequest`共享的。通过一个API下载的AssetBundle也会对另一个API可见。

尽管数据会被解压然后通过一个定长的buffer写入到缓存中，WWW对象会在原生内存中持有一个完整的AssetBundle数据拷贝。这个额外的拷贝是用来提供`WWW.bytes`属性访问的。

由于在WWW对象中存在内存的额外存储，AssetBundle应该保持尽量的小，最大几M。在后续的章节中，会有对AssetBundle大小的进一步讨论。

和UnityWebRequest不同，每次调用这个API都会生成一个新的工作线程。因此，在内存受限的平台上，如移动设备上，同时只允许一个AssetBundle被下载，来避免内存的尖峰。要注意多次调用会产生额外的线程。如果有超过5个AssetBundle需要下载，应该写一个脚本来管下载队列，以保证同时只有少量的AssetBundle被下载。

### 3.3.5 建议

通常情况下，应该尽量使用`AssetBundle.LoadFromFile`。这个API在速度、硬盘使用和运行时内存占用都是最高效的。

对于需要下载和更新AssetBundle的项目，强烈推荐使用UnityWebRequest（Unity 5.3以后）或WWW.LoadFromCacheOrDownload（Unity 5.2之前）。下一章中会详细讲到，it is possible to prime the AssetBundle Cache with Bundles included within a project's installer（TODO：恕我真的没看懂。。等看到下一章相关章节再看看。。）

在使用`UnityWebRequest`或`WWW.LoadFromCacheOrDownload`时，要保证在加载AssetBundle后正确的调用了`Dispose`方法。另外，C#的using语句可以很方便的保证WWW或UnityWebRequest被正确的dispose了。

如果项目有一定规模的开发团队，且有特殊的缓存和下载需求，可以考虑开发一个自定义的下载器。开发下载器不是一个轻松的工作，而且下载器应该和`AssetBundle.LoadFromFile`兼容。在下一章中会有更多的讨论。

## 3.4 从AssetBundle中加载Assets

UnityEngine.Objects可以通过AssetBundle类提供的3个不同的API从AssetBundle加载，它们都可传入同步和异步的参数：

* LoadAsset (LoadAssetAsync)
* LoadAllAssets (LoadAllAssetsAsync)
* LoadAssetWithSubAssets (LoadAssetWithSubAssetsAsync)

这些API的同步方法总是要比异步的要快一些，至少会快1帧。

异步加载方法会在每帧都去加载Objects，具体使用多少时间根据它们的时间片限制来决定。下文会针对这个行为进行更底层的技术讨论。

在加载多个不互相依赖的UnityEngine.Objects时，可以使用`LoadAllAssets`方法。只有当一个AssetBundle中大部分或全部Objects都需要被加载时，才应该使用这个方法。和其他的API相比，`LoadAllAssets`会比多次单独调用`LoadAssets`要快一点。因此，如果需要加载的资源量很大，但是AssetBundle中66%以下的资源需要被加载，就应该考虑拆分AssetBundle并使用`LoadAllAssets`。

`LoadAssetWithSubAssets`应该在加载包含多个嵌套Objects的复合Asset时使用，如包含内置动画的FBX模型文件，或包含多个sprites的sprite图集。如果需要加载的Objects都来自同一个Asset，但是共同存储的AssetBundle中还有其他很多无关的Objects，可以使用这个API。

在其他的使用情况下，应该使用`LoadAsset`和`LoadAssetAsync`。

### 3.4.1 底层加载细节

UnityEngine.Object的加载并不在主线程进行，而是通过工作线程从硬盘读取的。任何不会影响到Unity线程敏感（脚本、图形渲染）的事情，都会被转换到工作线程上进行。例如，从mesh生成VBO，解压缩贴图等。

从Unity 5.3以后，Object可以被并行的加载。在工作线程上，可以同时对多个Objects进行反序列化、处理和整合。当Object完成加载时，它的`Awake`回调会被执行，这个Object会在下一帧以后对Unity引擎可见。

同步的`AssetBundle.Load`方法会暂停主线程，直到Object的加载结束。Object的加载会被按时间切片，避免在每一帧占有超过一定的毫秒。每帧可占有的毫秒数可以通过`Application.backgroundLoadingPriority`来设置。

* ThreadPriority.High: 每帧最大50ms
* ThreadPriority.Normal: 每帧最大10ms
* ThreadPriority.BelowNormal: 每帧最大4ms
* ThreadPriority.Low: 每帧最大2ms

从Unity 5.2开始，在帧时间限制未达到的情况下，会继续尝试加载更多的Objects。在其他因素都相同的情况下，异步加载API总会比对应的同步方法要慢，这是由于异步方法调用和Object对引擎可见之间至少会有一帧的延迟。

### 3.4.2 AssetBundle依赖

根据运行环境的不同，AssetBundles之间的依赖关系通过两个不同的API来跟踪。在Unity Editor中，AssetBundle依赖可以通过`AssetDatabase`API来获取。AssetBundle的指定和依赖可以通过`AssetImporter`API来获取和修改。在运行时，Unity提供了一个可选的API，基于`ScriptableObject`的`AssetBundleManifest`，用来获取AssetBundle的依赖信息。

当AssetBundle中的Objects引用了其他AssetBundle中的Objects时，就产生了依赖关系。在系列第一篇中有对互相引用的描述。

在那篇中的序列化和实例化段落中描述过，AssetBundle中的Object通过FileGUID和LocalID来唯一指定，AssetBundle作为它们的源数据。（TODO：这话说的也太绕了。。。）

因为当Instance ID被解析时，Object会被自动加载，又因为当AssetBundle被加载时，Object会被赋予一个有效的Instance ID，不同AssetBundles的加载顺序是不重要的。（TODO：这个原因是在说啥。。。）但是在AssetBundle被加载之前，先加载其依赖的Object是很重要的。Unity不会自动加载依赖AssetBundles。

**举例：**

假设material A引用texture B。MaterialA被打包进了AssetBundle 1，texture B被打包进了AssetBundle 2。

![ab1](/img/ab1.jpg)

在这种情况下，AssetBundle 2必须在从AssetBundle 1中加载Material A之前加载。

这并不是说AssetBundle 2必须在AssetBundle 1之前加载，也不是说Texture B必须从AssetBundle 2中显式的加载。只要在从AssetBundle 1中加载Material A之前，加载好AssetBundle 2就足够了。

然而，当AssetBundle 1被加载时，Unity不会自动加载AssetBundle 2。这必须通过脚本手动的去实现。

可以参考[manual](https://docs.unity3d.com/Manual/AssetBundles-Dependencies.html?_ga=2.148172343.184694638.1540294356-812634850.1540294356)来获取关于AssetBundle依赖的更多信息。

### 3.4.3 AssetBundle manifest文件

当通过`BuildPipeline.BuildAssetBundles`来执行AssetBundle构建流程时，Unity会把每个AssetBundle的依赖信息存储到一个Object中。这个数据会被保存到一个单独的AssetBundle中，只包含一个`AssetBundleManifest`类型的Object。

这个Asset会被存储到一个AssetBundle中，和构建目录同名存储。如果项目在目录`(projectroot)/build/Client/`中构建AssetBundle，那么包含manifest信息的AssetBundle会被存储到`(projectroot)/build/Client/Client.manifest`。

这个包含了manifest的AssetBundle可以和其他AssetBundle一样被加载、缓存和卸载。

`AssetBundleManifest`对象提供了`GetAllAssetBundles`API来获取构建时的所有AssetBundles，还提供了两个方法来获取某个AssetBundle的依赖：

* `AssetBundleManifest.GetAllDependencies`返回某个AssetBundle所有的层级依赖结构，不仅包含它的直接子依赖，还包含更下级的信息。
* `AssetBundleManifest.GetDirectDependencies`只返回直接依赖关系。

注意这两个API都会分配字符串数组。因此，它们应该被很少使用，并且不应该在应用生命周期的性能敏感阶段使用。

### 3.4.4 建议

大部分情况下，在玩家进入性能关键区（主游戏关卡或世界）之前，应该尽可能的加载需要用到的资源。这在移动平台上尤其重要，因为访问本地存储很慢，而且加载和卸载Objects引起的内存波动会触发垃圾回收。

如果项目需要在应用可交互阶段加载和卸载Objects，可以参考在下一篇文章中的管理加载后的资源中，对如何卸载Objects和AssetBundles的信息。