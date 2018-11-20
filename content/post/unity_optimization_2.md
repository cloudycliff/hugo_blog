---
title: "[译]Unity优化最佳实践2-内存"
date: 2018-11-16T09:52:00+08:00
draft: false
tags: [Unity3D]
categories: [translate]
---

> 本篇是对Unity官方最佳实践的翻译，[原文地址](https://docs.unity3d.com/Manual/BestPracticeUnderstandingPerformanceInUnity2.html)
> 
> 本篇中提到的工具，好像已经有一段时间没有维护过了。。不确定现在的版本是否还能继续使用，也不确定现在内置的Profiler能否完全替代它，以后在实际项目中尝试一下。不过不是我说，老哥你这个工具解析一次花的时间也太长了吧。。真的实用吗。。。

# 内存

内存消耗是一个关键的性能指标，在内存资源受限的平台（如低端移动设备）上尤其重要。

## Profiling内存消耗

在Unity中分析内存问题最好的工具是在Unity的Bitbucket上的[开源内存可视化工具](https://bitbucket.org/Unity-Technologies/memoryprofiler)。集成这个工具非常简单，只需要下载链接中的代码然后放到项目的Editor目录下。

这个工具可以在任何Unity 5.3以后的版本使用。当挂载到使用IL2CPP构建的应用上以后，它可以捕获有关原生和托管内存的大量信息。

想使用这个工具，只需要简单的把项目用IL2CPP进行构建，然后发布到合适的设备上。挂载上Unity常规的CPU profiler，然后打开Menory Profiler窗口（菜单：Window > MemoryProfilerWindow），然后选择`Take Snapshot`。

在设备上的应用会暂停一下来采集数据并传输到Unity Editor中。然后，Unity Editor会暂停来解析收到的数据；这可能会需要大量的时间。对于一些内存集中的应用，一次追踪可能要花费10-30分钟来解析。

在这个解析和加载操作中，推荐你有足够的耐心。。。

![image0](/img/UnderstandingPerformanceinUnity-MemorySection_image_0.jpg)

上面的截图来自运行在iOS设备上的标准资源场景，体现出超过3/4的内存使用可以归结于4个非常大的贴图，全都和飞机的机身有关。

这个视图可以被缩放。可以点击每个方块来查看更多的信息。

### 发现重复贴图

一个常见的内存问题是内存中的重复资源。由于贴图经常是项目中最占内存的资源，贴图重复是Unity项目中最常见的内存问题。

重复的资源可以通过定位两个具有同样类型和大小的objects来发现，其表现出是从同一个资源加载而来的。在新的内存Profiler工具的详情面板中，对于疑似相同的objects检查它们的Name和InstanceID字段。

Name字段是基于object所加载的资源文件名；通常，这是不带文件路径和扩展名的文件名。InstanceID字段表示Unity运行时分配的内部标识符；这个数字在单次Unity运行期间是唯一的。

![image0](/img/UnderstandingPerformanceinUnity-MemorySection_image_1.png)

上面的截图展示了这个问题的一个例子。截图的左右两边分别来自5.4版的内存Profiler的详情面板。截图中展示的资源是在内存中分别加载的两个贴图。这两个贴图有同样的名字和大小，表明他们可能是重复的。通过检查项目的Assets目录，可以发现只有一个资源叫做`wood-floorboards-texture`，更加明显的指出这是个资源重复。

在内存中每个独立的UnityEngine.Object都有一个唯一的instance ID，在object被创建是赋值给它。鉴于这两个贴图有不同的instance ID，很明显它们表示在内存中属于不同的贴图实例。

由于文件名和资源大小一样，但是instance ID不同，可以确定这两个objects表示同一个贴图在内存中产生了重复（注：如果项目中有同样的贴图名，这个判断就不是绝对的，但是在文件大小一致的情况下，可以更加肯定这个判断）。

#### AssetBundle和资源重复

在内存中出现贴图和资源的重复最常见的原因是不恰当的卸载AssetBundle。可以参考[AssetBundle最佳实践](http://unity3d.com/learn/tutorials/topics/best-practices/guide-assetbundles-and-resources?playlist=30089)来获取更多信息。关键章节是[管理加载的资源](http://unity3d.com/learn/tutorials/topics/best-practices/assetbundle-usage-patterns?playlist=30089)。（译注：之前也翻译过这一系列文章）

### 检查图像缓存、图像效果和RenderTexture内存使用

在内存视图中，也可以把需要提供给图像效果和RenderTexture物体的渲染缓存进行可视化。（译注：这里还不太明白这几个概念都是什么。。待我慢慢看相关的东西。。）

![image0](/img/UnderstandingPerformanceinUnity-MemorySection_image_2.png)

上面的截图展示了一个包含多个Unity的Cinematic图像效果的简单场景。图像效果分配了临时的渲染缓存来进行相应的计算；更详细的，Bloom效果分配了多个大小递减的缓存。由于Retina iOS设备的高分辨率，这些临时的缓存占用了比项目其他内存之和大得多的内存。

考虑一个渲染分辨率在2048x1536的iPad Air 2-这已经超过了通常发布在主机和PC上的1080p的分辨率，但仅仅运行在一个平板设备上。根据不同的缓存格式，一个全屏的临时渲染缓存就会占用了整整24或36MB的内存。如果把渲染缓存的像素尺寸减半，就可以减少75%的空间。这通常并不会明显的降低渲染结果的视觉质量。

一种优化图像效果占用渲染缓存和其他GPU资源的方法是一次性创建一个“超级”图像效果来进行所有的不同的计算。在使用Unity 5.5或以后的版本时，可以使用新的[UberFX](https://github.com/Unity-Technologies/PostProcessing)（可以在[github](https://github.com/Unity-Technologies/PostProcessing)获取；这个包提供了一种可配置的“超级”图像效果来执行Cinematic提供的所有操作，这样比单独进行每个图像效果计算使用更少的内存）。