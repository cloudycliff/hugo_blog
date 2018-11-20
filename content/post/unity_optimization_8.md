---
title: "[译]Unity优化最佳实践8-特殊优化"
date: 2018-11-19T20:11:57+08:00
draft: false
tags: [Unity3D]
categories: [translate]
---


> 本篇是对Unity官方最佳实践的翻译，[原文地址](https://docs.unity3d.com/Manual/BestPracticeUnderstandingPerformanceInUnity8.html)


# 特殊优化

上一章介绍了一些对所有项目适用的优化，本章介绍的优化不应该在没有profile数据支持前使用。这可能是因为这些优化的实现很费力，可能会为了性能牺牲代码清晰度和可维护性，或者所解决的问题只会在特定的量级上出现。

## 多维数组vs.数组的数组

依据这篇[StackOverflow回答](http://stackoverflow.com/questions/597720/what-are-the-differences-between-a-multidimensional-array-and-an-array-of-arrays)中的描述，通常遍历数组的数据比遍历多维数组更高效，因为多维数组需要进行函数调用。

注意：

* 数组的数据，被声明为`type[x][y]`，多维数组为`type[x,y]`
* 这可以通过使用ILSpy等工具来检查多维数组访问代码生成的IL来发现

在Unity 5.3下profile发现，对三维100x100x100的数组完全遍历100次，然后平均10次测试解决，得到下面的数据：

| 数组类型 | 总时间（100次循环）|
|---------|-----------------|
|一维数组  | 660 ms |
|数组的数组| 730 ms |
|多维数组  | 3470 ms|

对比多维数组和一维数组访问的悬殊时差，可以看出额外的函数调用所消耗的性能，对比数组的数组和一维数组的访问可以看出遍历非紧凑内存结构带来的损耗。

上面的数据可以看出，额外的函数调用远远超过了使用非紧凑内存结构带来的影响。

对于性能要求非常高的操作，推荐使用一维数组。对于其他需要使用多个维度的情况，使用数组的数组。多维数组不应该被使用。

## 粒子系统池化

当池化粒子系统时，要注意它们至少会占用3500字节的内存。而且随着粒子系统上激活的模块数，内存占用还会增长。这些内存不会随着粒子系统的禁用而释放，只有在它们被销毁时才会释放。

从Unity 5.3开始，大部分粒子系统的设置都可以在运行时控制。对于需要池化大量不同粒子效果的项目，把粒子系统的配置参数提取出来放到数据承载类或结构中可能是更高效的。

当需要一个粒子效果时，一个“通用”粒子效果池可以提供一个必须的粒子效果对象。然后可以把配置数据赋予这个对象来达到需要的图像效果。

相比于试着池化场景中所有不同配置的粒子系统，这会显著的提高内存效率，但同时也显著的提高了编码需求。

## Update管理器

在内部，Unity根据需要的回调（例如`Update`，`FixedUpdate`和`LateUpdate`）维护了一个对象列表。它们被维护为介入式链表（译注：intrusive linked list，不是很熟悉这个数据结构。。），用来保证链表以常量时间更新。在MonoBehaviour被启用或禁用时，它们被加入或移出这些链表。

虽然简单的按需向MonoBehaviour添加合适的回调很方便，这随着回调的增加会变得低效。尽管很小，从原生代码执行托管代码回调的消耗也是值得关注的。这不仅表现在执行大量逐帧方法时产生的帧率降低，也表现在初始化包含大量MonoBehaviour的Prefab时引起的初始化时间增加（注意：这个初始化消耗是因为执行prefab中每个Component上的Awake和OnEnable回调）。

当含有逐帧回调的MonoBehaviour的数量增长到几百上千时，去掉这些回调并把这些MonoBahaviour（甚至是标准的C#对象）附着到一个全局管理单例中是更有利的。这个全局管理单例可以分发`Update`，`LateUpdate`和其他回调到感兴趣的对象上。这样做有额外的好处，可以允许代码在不需要操作时智能的取消回调订阅，以减少每帧必须调用的函数数量。

最大的节约通常通过消除很少执行的回调来实现。考虑下面的伪代码：

```C#
void Update() {
    if(!someVeryRareCondition) { return; }
// … some operation …
}
```

如果大量的MonoBehaviour包含类似上面的Update回调，那么大量的时间都会被花费在从原生代码和托管代码之间切换，虽然这些调用很快就会退出。如果这些类只有在`someVeryRareCondition`是true时才向全局Update管理器订阅，在恰当的时候退订，就可以把切换和判断条件的时间节约下来。

### 在update管理器中使用C#代理

使用纯C#代理来实现这些回调是很吸引人的。然而，C#代理的实现以低频率的订阅和解除，少量的回调进行优化。C#的代理在每次一个回调被加入或移出时，会对回调列表进行一次深拷贝。回调列表很大时，或在单帧多次订阅/解除回调时，会在内部的`Delegate.Combine`方法中引起性能尖峰。

在添加/删除发生频率很高的场合，考虑使用能够快速插入/删除的数据结构来代替代理。

## 加载线程控制

Unity允许开发者控制用来加载数据的后台线程的优先级。这对于想要在后台从硬盘中加载AssetBundle是很重要的。

主线程和图形线程的优先级都是`ThreadPriority.Normal` - 任何有比这个高优先级的线程会阻塞主/图形线程，导致帧率不稳定，而低优先级的线程则不会。如果某线程和主线程有同样的优先级，CPU会尝试给它们分配相等的时间，在有多个后台线程在执行繁重的操作，如解压缩AssetBundle时，会导致帧率波动。

目前，这个优先级可以在三个地方控制。

首先，资源加载函数（如`Resources.LoadAsync`和`AssetBundle.LoadAssetAsync`）的优先级，是从[Application.backgroundLoadingPriority](https://docs.unity3d.com/ScriptReference/Application-backgroundLoadingPriority.html)设置获取的。依据文档，这个设置同时会限制主线程用来组合资源的时间（注：大部分Unity资源必须在主线程“组合”。在组合期间，资源的初始化过程会结束，并会执行一些线程安全的操作。这包括脚本回调执行，如Awake回调等。可以参考“资源管理”指南获取更多信息），以便限制加载资源带来的帧率影响。

其次，每个异步资源加载操作，以及每个UnityWebRequest请求，返回一个`AsyncOperation`对象来监视和管理这个操作。这个`AsyncOperation`对象暴露了一个[priority](https://docs.unity3d.com/ScriptReference/AsyncOperation-priority.html)属性，可以用来调节每个单独操作的优先级。

最后，WWW对象，例如从`WWW.LoadFromCacheOrDownload`调用返回的对象，暴露一个[threadPriority](https://docs.unity3d.com/ScriptReference/WWW-threadPriority.html)属性。要注意WWW对象不会自动使用`Application.backgroundLoadingPriority`设置作为其默认值 - WWW对象总是默认使用`ThreadPriority.Normal`。

要注意，这些API使用的底层系统是不同的。`Resources.LoadAsync`和`AssetBundle.LoadAssetAsync`被Unity内部的PreloadManager系统操作，它会管理自己的加载线程并执行自己的帧率限制。`UnityWebRequest`使用自己专用的线程池。`WWW`会在每次请求创建时生成一个全新的线程。

虽然其他所有的加载机制都有内置的队列系统，WWW并没有。在大量的压缩AssetBundle上调用`WWW.LoadFromCacheOrDownload`会生成等量的线程，然后会和主线程抢夺CPU时间。这很容易就造成帧率波动。

因此，在使用WWW加载和解压缩AssetBundle时，给每个生成的WWW对象设置一个合理的`threadPriority`被认为是最佳的做法。

## 大量物体移动和CullingGroups

在Transform操作一节中提到过，移动庞大的Transform层级会由于改变消息的传播引起较高的CPU消耗。然而，在实际的开发环境中，通常不可能把层级结构合并为合理的GameObject数量。

与此同时，只执行足够的可以保持游戏世界真实性的操作，而消除玩家不会注意的操作是个很好的开发实践 - 例如，在一个包含大量角色的场景，只针对在屏幕内的角色计算蒙皮和动画移动是更优化的。没有理由为不在屏幕上的角色浪费CPU时间去计算纯视觉上的元素。

这两个问题可以通过Unity 5.1引入的API来巧妙的解决：[CullingGroups](https://docs.unity3d.com/Manual/CullingGroupAPI.html)。

与直接操作场景中大量GameObject不同，让系统去操作CullingGroup中一组BoundingSphere的Vector3参数。每个BoundingSphere都作为一个游戏逻辑实体的世界坐标位置的权威存放点，它会在实体移动靠近/进入CullingGroup的主摄像机的frustum时收到回调。这些回调可以被用来启动/禁用控制行为的代码或组件（例如Animator），这些组件只应该在实体可见时被执行。

## 减少方法调用开销

C#的字符串库提供了很好的案例展示了在简单库代码中，添加额外的方法调用带来的性能消耗。在内置字符串API`String.StartsWith`和`String.EndsWith`一节提到，手写的替代代码比内置方法快10-100倍，即使在不必要的字符集转换被禁用的情况下。

这个性能差异的关键原因仅仅是在紧密的内部循环中加入了额外的方法调用。每次方法被调用必须在内存中定位方法的地址，在栈中推入新的一帧。这些操作都不是免费的，但是在大部分代码中它们小到可以被忽略。

不过，在紧密的循环中执行小的方法时，引入额外的方法调用的开销会变得显著 - 甚至会是最主要的消耗。

考虑下面两个简单的方法。

**示例1**

```C#
int Accum { get; set; }
Accum = 0;

for(int i = 0; i < myList.Count; i++) {
    Accum += myList[i];
}
```

**示例2**

```C#
int accum = 0;
int len = myList.Count;

for(int i = 0; i < len; i++) {
    accum += myList[i];
}
```

这两个方法都计算了一个C#泛型`List<int>`中所有整数的和。第一个例子更加“现代C#”一些，因为它使用了一个自动生成的属性来保存数据值。

虽然在表面上这两个代码段看起来差不多，在分析方法调用时差异就很明显了。

**示例1**

```C#
int Accum { get; set; }
Accum = 0;

for(int i = 0;
       i < myList.Count;    // call to List::getCount
       i++) {
    Accum       // call to set_Accum
+=      // call to get_Accum
myList[i];  // call to List::get_Value
}
```

所以每次循环执行都会有4次方法调用：

* `myList.Count`执行`Count`属性上的`get`方法
* `Accum`属性上的`get`和`set`方法必须被执行
* 调用`get`来获取当前`Accum`的值以传入加法操作
* 调用`set`来把加法操作结果赋值给`Accum`
* []运算符执行列表的`get_Value`方法来获取特定索引上的值

**示例2**

```C#
int accum = 0;
int len = myList.Count;

for(int i = 0;
    i < len; 
    i++) {
    accum += myList[i]; // call to List::get_Value
}
```

在第二个例子中，对`get_Value`的调用还存在，但是所有其他的方法都要么被消除或不会每次循环都执行。

* 由于`Accum`现在是一个原始类型而不是属性，在设置或获取其值时不需要进行方法调用
* 由于假设`myList.Count`不会在循环过程中变化，它的获取被移到了循环条件语句的外面，所以它不会在每次循环开始时执行。

对这两个版本的计时揭示出移除75%的方法调用带来的好处。在现代桌面机器上执行100000次的结果如下：

* 示例1需要324毫秒来执行
* 示例2需要128毫秒来执行

这里主要的问题是Unity很少进行方法inline。即使在IL2CPP下，很多方法都不会正常的inline。这对于属性来说更是如此。进一步，虚函数和接口根本就不能被inline。

因此，在C#代码中声明的方法调用很可能会在最终二进制程序中生成一个函数调用。

## Trivial属性

（译注：这里的trivial，感觉应该理解为一个常量，或者是很简短的一个语句的函数，反正不知道怎么翻译。。）

Unity为它的数据类型提供了很多“简单”常量来方便开发者。不过，从上文来看，要注意这些常量通常被实现为返回固定值的属性。

Vector3.zero属性的实现为：

```C#
get { return new Vector3(0,0,0); }
```

Quaternion.identity也很类似：

```C#
get { return new Quaternion(0,0,0,1); }
```

尽管访问这些属性带来的消耗相比于它们周围的代码要小，在每帧执行几千或更多次时还是会产生一些小的差异。

对于简单的原始类型，使用`const`值来代替。`const`值在编译时进行inline - 对`const`的引用会被替换为它的值。

注意：由于每个对于`const`的引用都被替换为值，不建议把长字符串或其他大数据类型声明为`const`。这会不必要的增大最终二进制的大小，因为这些数据在最终的字节码中产生了重复。

在`const`不合适时，使用`static readonly`变量来代替。在某些项目中，即使Unity的内置trivial属性都被替换成了`static readonly`，来获取性能上的小提升。

## Trivial方法

Trivial方法更加复杂一些。能够一次声明函数并在其他地方复用是非常有用的。不过，在紧密的循环中，可能需要偏离好的编程方式转而“手动inline”一些代码。

有些方法可以完全被移除。考虑`Quaternion.Set`，`Transform.Translate`或`Vector3.Scale`。它们执行非常单一的操作，可以被替换为简单的赋值语句。

对于更复杂的方法，需要根据profile结果来权衡是手动inline的收益大还是对更高效代码长期维护的开销更大。