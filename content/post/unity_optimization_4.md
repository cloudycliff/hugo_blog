---
title: "[译]Unity优化最佳实践4-资源审查"
date: 2018-11-16T17:40:09+08:00
draft: false
tags: [Unity3D]
categories: [translate]
---

> 本篇是对Unity官方最佳实践的翻译，[原文地址](https://docs.unity3d.com/Manual/BestPracticeUnderstandingPerformanceInUnity4.html)


# 资源审查

在实际项目中很多问题都出于一些无意的错误：临时的“测试性”修改或劳累的开发者误点击都会无声的添加性能很差的资源，或者修改已存在资源的导入设置。

对于任何有一定规模的项目，最好能够构建防止人为错误的第一道防线。编写一小段程序来保证没有人可以添加一个4K未压缩贴图到项目中是相对简单的。

然而，这是一个惊人的常见问题。一个4K未压缩贴图占用60MB的内存。在低端移动设备（如iPhone 4S）上，占用超过约180-200MB的内存是很危险的。如果错误的添加，这个贴图会不经意的占用约1/3至1/4应用的内存预算，产生难以分析的内存不足错误。

虽然可以使用Unity 5.3的内存Profiler来追踪这些问题，保证这些错误一开始就不会发生可以说是更好的办法。

## 使用AssetPostprocessor

Unity Editor中的`AssetPostprocessor`类可以用来在Unity项目中强制一些最低的标准。在资源被导入时，这个类会收到回调。想使用它，继承`AssetPostprocessor`类并实现一个或多个`OnPreprocess`方法。值得关注的方法包括：

* `OnPreprocessTexture`
* `OnPreprocessModel`
* `OnPreprocessAnimation`
* `OnPreprocessAudio`

可以查看[AssetPostprocessor](https://docs.unity3d.com/ScriptReference/AssetPostprocessor.html)来了解更多的`OnPreprocess`方法。

```c#

public class ReadOnlyModelPostprocessor : AssetPostprocessor {

   public void OnPreprocessModel() {

        ModelImporter modelImporter =

 (ModelImporter)assetImporter;

        if(modelImporter.isReadable) {

            modelImporter.isReadable = false;

            modelImporter.SaveAndReimport();

        }

    }

}


```

这是一个在项目中使用`AssetPostprocessor`强制规则的简单例子：

这个类会在每次有模型被导入到项目，或每次某模型的导入设置被修改时被调用。这段代码只是简单的检查`Read/Write enabled`（通过`isReadable`属性表示）标记是否被设置为`true`。如果是true，它会强制属性为`false`并且保存，然后重新导入资源。

注意调用`SaveAndReimport`会导致上面的代码片段再次执行！然而，因为这时候可以保证`isReadable`是false，这段代码不会产生无限的导入循环。

做这个修改的原因会在下面的模型一节进行讨论。

## 通用资源规则

### 贴图

#### 禁用read/write enabled标记

`read/write enabled`标记会导致贴图在内存中保存两份：一份在GPU，一份在CPU可寻址内存（这是因为，在大部分平台，从GPU内存读取回来非常的慢。从GPU内存读取贴图到临时缓存中供CPU代码使用（如`Texture.GetPixel`）是非常低效的）。在Unity中，这个设置被默认禁止，但是它可能被意外的打开。

`read/write enabled`标记只有在需要在Shader以外操作贴图数据（例如`Texture.GetPixel`和`Texture.SetPixel`）时才需要打开，应该尽可能的避免。

#### 尽可能禁用Mipmap

对于相对于摄像机Z深度不变的物体，可以通过禁用mipmap来节约大概1/3的加载贴图需要的内存。但是如果一个物体的Z深度会改变，禁用mipmap可能会导致GPU上对贴图采样的效率变低。

通常，这对于UI贴图和其他在屏幕上大小不变的贴图很有用。

#### 压缩所有的贴图

采用一种平台合适的贴图压缩格式对于节约内存是至关重要的。

如果选择的压缩格式不适合目标平台，Unity会在贴图加载时解压缩贴图，占用额外的CPU时间和内存。这对于Android设备是个常见的问题，根据采用的芯片不同，其支持的贴图压缩格式也是多种多样的。

#### 强制实用的贴图大小限制

尽管这很简单，人们很容易忘记对贴图进行缩放或不经意的修改贴图大小导入设置。为不同种类的贴图确定一种实用的最大尺寸，并且通过代码来限制它。

对于大部分移动应用，2048x2048或1024x1024对于贴图图集是足够的，512x512对于3D模型的贴图是足够的。（译注：几年之后，不知道这个标准是不是还适用。。。）

### 模型

#### 禁用read/write enabled标记

对于模型来说，`read/write enabled`和之前贴图的操作是一样的。不过，它为模型默认打开了。

如果项目需要在运行时通过脚本修改Mesh或Mesh被用来作为MeshCollider组件的基础，就需要把这个标记打开。如果模型不会在MeshCollider被用到且不会通过脚本去操作，就可以禁用这个标记来节约一半的内存。

#### 在非人物模型上禁用rig

默认情况下，Unity会为非人物模型引入一个通用的rig。这会导致模型在初始化时，添加一个`Animator`组件。如果模型不会被动画系统控制，这个添加会给动画系统增加额外的负担，因为每个活跃的Animator都需要每帧tick一次。

对于没有动画的模型，禁用rig可以避免自动添加Animator组件，也可以避免无心的添加非预期的动画。

#### 为有动画的模型开启Optimize Game Objects选项

**Optimize Game Objects**选项对于动画模型的性能有显著的影响。当这个选项被禁用时，Unity会在模型初始化时，创建一个庞大的与模型骨骼结构对应的transform层级结构。对于这个transform层级的更新需要很大的代价，在有其他组件（如粒子系统或碰撞体）附着的时候尤其如此。这也限制了Unity的多线程Mesh蒙皮能力和骨骼动画计算。

如果模型骨骼结构的特定位置需要暴露出来（例如把模型的手部暴露出来以便动态附加武器模型），这些位置可以被添加到`Extra Transforms`白名单中。

在Unity的手册[模型导入](https://docs.unity3d.com/Manual/FBXImporter-Rig.html)中可以找到更多信息。

#### 可能情况下使用Mesh压缩

开启Mesh压缩会减少表示模型数据不同通道的浮点数的位数。这可能会导致精度的丢失，在用于最终项目之前，需要经过美工的检查。

特定的压缩等级会使用多少为数字可以在[ModelImporterMeshCompression](https://docs.unity3d.com/ScriptReference/ModelImporterMeshCompression.html)查看。

注意可以为不同的通道使用不同的压缩等级，所以一个项目可以选择只压缩切线和法线，保留UV和定点位置不压缩。

#### 注意：Mesh Renderer设置

在为Prefab或GameObject添加Mesh Renderer时，要注意组件上的各种设置。默认情况下，Unity开启Shadow casting和receiving，Light Probe sampling，Reflection Probe sampling和Motion Vector calculation。（译注：这些都是设置选项，就不翻译了，方便到面板中查找，其实也能看懂都是什么东西。。）

如果项目不需要某些功能，可以通过脚本来确保它们被默认关闭。任何在运行时动态添加MeshRenderer的代码也需要处理这些设置。

对于2D游戏，意外向场景中添加一个开启阴影选项的MeshRenderer会给渲染循环添加一个完整的阴影pass。这一般都是在浪费性能。

### 音频

#### 适合平台的压缩设置

对于音频，选择一个匹配对应硬件的压缩格式。所有的iOS设备都有硬件支持的MP3解压支持，大部分Android都自带Vorbis支持。

更进一步，应该向Unity导入未压缩的音频文件。Unity在构建项目时，总是会重新压缩音频。导入压缩音频然后重新压缩是没必要的；这只会降低最终音频的质量。

#### 强制音频为单声道

只有很少的移动设备真正拥有立体声扬声器。在移动项目中，强制导入的音频为单声道可以使其内存占用减半。这个设置也适用于没有立体声效果的音频，例如UI声效。

#### 降低音频比特率

尽管这需要向音频设计师咨询，尝试去降低音频的比特率可以进一步的降低内存占用和项目大小。