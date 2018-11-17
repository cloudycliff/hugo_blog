---
title: "Unity_optimization_1"
date: 2018-11-15T15:59:57+08:00
draft: false
tags: [Unity3D]
categories: [translate]
---

> 本篇是对Unity官方最佳实践的翻译，[原文地址](https://docs.unity3d.com/Manual/BestPracticeUnderstandingPerformanceInUnity1.html)

> 本章的标题，好像并没有很合适的翻译，最后决定还是保持原文最能体现其意思。。

# Profiling

在讨论性能时，很重要的一点是要记住所有的优化尝试必须经由一个发现过程开始。对应用做profiling来发现它的热点是必须的第一步，然后才是针对项目的技术和资源结构对profiling的结果做分析。

注意：本章节中profiling追踪时使用的原生代码函数名，均来自于Unity 5.3版本。在未来的Unity版本中，这些函数名可能会变化。

## 工具

对于Unity开发者，有很多profiling工具可以使用。Unity有一些列内置的工具，例如[CPU Profiler](https://docs.unity3d.com/Manual/ProfilerCPU.html)，[Memory Profiler](https://docs.unity3d.com/Manual/ProfilerMemory.html)和5.3新增加的内存分析器。

不过，最好的分析数据往往来自平台特定的工具。包括：

* iOS：Instruments和XCode Frame Debugger
* Android：Snapdragon Profiler
* 运行在Intel CPU/GPU的平台：VTune和Intel GPA
* PS4：Razor套件和VR Trace
* Xbox：Pix工具

这些工具差不多覆盖了所有可以通过IL2CPP来产生项目C++版本代码的平台。这些原生版本的代码提供了透明的调用栈和高分辨率的方法执行时间，这些是运行在Mono框架时提供不了的。

Unity已经编写了一个使用Instruments来profile iOS游戏的[基础教程](http://blogs.unity3d.com/2016/02/01/profiling-with-instruments/)。

## 剖析启动追踪

在观察启动时间追踪信息时，有两个关键的方法需要检查。这两个方法是项目的配置、资源和代码能够影响启动时间的主要地方。

注意启动时间在不同平台上展现方式不同。在大部分平台上，它被表现为一个静态的启动界面。

![image0](/img/UnderstandingPerformanceinUnity-ProfilingSection_image_0.png)

上面的截图来自一个运行在iOS的示例项目的Instruments分析。在平台特定的`startUnity`方法中，注意`UnityInitApplicationGraphics`和`UnityLoadApplication`这两个方法。

`UnityInitApplicationGraphics`执行了很多内部工作，如创建图形设备和初始化多种Unity内部系统。另外，它初始化了Resources系统。在此过程中，它必须加载所有包含在Resources系统中文件的索引。（译注：这里只是加载了index吗？我记得是所有资源都会加载呢。。如果只是加载索引其实对性能的影响没那么大吧。。可能是我对原句的理解有误。。总之大量使用Resources文件夹是不好的。。）

在每个命名为`Resources`文件夹下的每个文件（注：这只适用于在项目Assets文件夹下的Resources，也包含其中的子文件夹）都会包含在Resource系统的数据中。因此，初始化Resources系统所需要的时间至少随着Resources文件夹下文件数量呈线性增长。

`UnityLoadApplication`包含了加载和初始化项目第一个场景的方法。这包括反序列化和初始化显示第一个场景所需要的所有数据，例如编译shaders，上传贴图到GPU和初始化GameObjects。另外，第一个场景所有的MonoBehaviour的`Awake`回调都会在这时候被执行。

这些操作意味着如果项目第一个场景的`Awake`回调里有耗时很长的代码，可能就会导致项目启动时间变长。为了解决这个问题，要不消除耗时的代码，或者在应用的生命周期其他地方执行。

## 剖析运行时追踪

在启动之后的profiling中，最需要关注的方法是`PlayerLoop`。这是Unity的主循环，其中的代码每一帧会执行一次。

![image0](/img/UnderstandingPerformanceinUnity-ProfilingSection_image_1.png)

上面的截图来自一个Unity 5.4的示例项目，体现了几个`PlayerLoop`中最有趣的方法。注意在`PlayerLoop`中的函数名可能会随着Unity版本更新而变化。

`PlayerRender`是执行Unity渲染系统的方法。这包括剔除物体，计算动态批，向GPU提交渲染指令。任何图片效果或基于渲染的脚本回调（例如`OnWillRenderObject`）也在这里执行。总的来说，这应该是项目在可交互期间消耗CPU时间最多的地方。

`BaseBehaviourManager`调用`CommonUpdate`的三个泛型版本。这些执行了在当前场景中，活跃GameObject上挂载的MonoBehaviour中特定的回调。

* `CommonUpdate<UpdateManager>`调用`Update`回调
* `CommonUpdate<LateUpdateManager>`调用`LateUpdate`回调
* `CommonUpdate<FixedUpdateManager>`在物理系统触发的情况下调用`FixedUpdate`回调

总的来说，`BaseBehaviourManager::CommonUpdate<UpdateManager>`是最需要审查的方法，因为它是Unity项目中大部分脚本代码的入口。

还有其他几个方法也值得关注：

在项目使用了Unity UI的情况下，`UI::CanvasManager`会执行一些不同的回调。这包含了Unity UI的和批计算和布局更新，这两个操作经常会导致CanvasManager出现在profiler中。

`DelayedCallManager::Update`执行协程。在后续的协程章节中会有更详细的描述。

`PhysicsManager::FixedUpdate`执行PhysX物理系统。这主要涉及到了执行PhysX的内部代码，并且会受到当前场景中物理物体（如刚体和碰撞体）的数量影响。并且，基于物理的回调也会出现在这里，包括`OnTriggerStay`和`OnCollisionStay`。

如果项目使用了2D物理，会在`Physics2DManager::FixedUpdate`下出现一些类似的调用。

## 剖析脚本方法

当脚本运行在使用IL2CPP交叉编译的平台上时，寻找包含`ScriptingInvocation`的行。这是Unity的内部原生代码为了执行脚本代码，向脚本运行时过渡的点（注：技术上来说，在经过IL2CPP之后，C#/JS代码也变成了原生代码。不过，这些交叉编译的代码主要通过IL2CPP运行时框架执行方法，而并不和手写的C++类似）。

![image0](/img/UnderstandingPerformanceinUnity-ProfilingSection_image_2.png)

上面的截图来自于Unity 5.4的一个示例项目。所有在`RuntimeInvoker_Void`下面的方法都是每帧执行的交叉编译的C#脚本。

这些追踪行非常易读：每一个都是原始类名跟着一个下划线和原始方法名。在这个例子中，可以看到`EventSystem.Update`，`PlayerShooting.Update`和其他几个`Update`方法。这些是在大部分MonoBehaviour中出现的Unity标准`Update`回调。

通过展开这些方法，可以发现其中具体哪些方法在占用CPU时间。这包括了项目中其他脚本的方法，Unity的API和C#库代码等。

上面的追踪展现出`StandaloneInputModule.Process`方法会每一帧都对整个UI做一次射线，目的是判断是否有触摸事件悬浮在或者激活任何UI元素。主要的消耗都在轮询所有的UI元素，检测鼠标的位置是否在它们的包围盒之内。（译注：好像在那个版本的Unity里，即使没有鼠标也会进行这个检测，所以很影响性能，然后在某个版本的Unity里已经被修复了，现在已经不需要担心这个问题了）

## 资源加载

资源加载也可以在CPU追踪中被识别。表明资源加载的主要方法是`SerializedFile::ReadObject`。这个方法连接了二进制数据流（从文件中）和Unity的序列化系统，通过一个叫`Transfer`的方法来进行操作。这个`Transfer`方法可以在所有的资源类型上找到，如贴图，MonoBehaviour和粒子系统。

![image0](/img/UnderstandingPerformanceinUnity-ProfilingSection_image_3.png)

在上面的截图中，一个场景正在被加载。这需要Unity读取和反序列化场景中所有的资源，表示为`SerializedFile::ReadObject`下的多个`Transfer`调用。

通常情况下，如果在运行时出现卡顿，且性能追踪表明`SerializedFile::ReadObject`使用了大量时间，这个帧率下降就是由资源加载引起的。注意，在大部分情况下，只有在通过`SceneManager`，`Resources`或AssetBundle API调用同步资源加载函数时，`SerializedFile::ReadObject`才会出现在主线程中。

这种卡顿可以通过常规的方法来修复：你可以异步的加载资源（把繁重的`ReadObject`调用转移到工作线程），或提前加载大的资源。

注意`Transfer`调用也会出现在复制Object（在追踪中表现为`CloneObject`）时。如果对`Transfer`的调用出现在`CloneObject`之下，这个资源就不是从存储中加载的。相对的，这个老Object的数据正在被传输到新Object中。Unity通过序列化老Object然后反序列化数据到新Object来实现这个操作。