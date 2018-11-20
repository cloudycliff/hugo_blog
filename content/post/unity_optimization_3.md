---
title: "[译]Unity优化最佳实践3-协程"
date: 2018-11-16T15:30:08+08:00
draft: false
tags: [Unity3D]
categories: [translate]
---


> 本篇是对Unity官方最佳实践的翻译，[原文地址](https://docs.unity3d.com/Manual/BestPracticeUnderstandingPerformanceInUnity3.html)


# 协程

协程有着和其他脚本代码不同的执行方式。大部分脚本代码只会出现在性能追踪的一个地方，在某个特定的Unity回调执行之下。然而，协程的执行总是出现在两个地方。

一个协程的初始代码，从协程的启动直到它第一次yield，会出现在协程启动的地方。通常，它表现在`StartCoroutine`方法调用的地方。通过Unity回调产生的协程（如返回一个`IEnumerator`的`Start`回调）会第一次出现在它们对应的Unity回调中。

协程的所有其他代码-从它第一次恢复直到它完成执行-都出现在Unity主循环中的`DelayedCallManager`中。

为了理解为什么会有这种现象，要考虑线程是如何被实际执行的。

协程是由C#编译器自动生成的一个类的实例来支持的。虽然对程序员来说协程只是一个方法，需要用这个对象来追踪协程在多次执行过程中的状态。因为协程本地作用域的变量必须在`yield`调用之间保持有效，这些本地作用域变量会被挂到生成的类中，因此会在协程的执行过程中仍然在堆中保持分配状态。这个对象也追踪协程的内部状态：它记录了协程在yield之后必须在代码里的那一点恢复执行。

因为这个，启动一个协程的内存压力等于一个固定的占用加上它本地作用域变量的大小。

启动一个线程的代码创建并执行这个对象，然后Unity的`DelayedCallManager`会在协程的yield条件满足时再次执行它。由于协程通常开始于其他协程之外，这会把它们的执行拆分到上面描述的两个地方中。

![image0](/img/UnderstandingPerformanceinUnity-CoroutinesSection_image_0.jpg)

这可以通过上面的截图来观察到，`DelayedCallManager`恢复执行了多个不同的协程：`PopulateCharacters`，`AsyncLoad`和`LoadDatabase`。

如果可能，最好把一系列操作压缩到最少数量的独立协程中。虽然嵌套协程对于代码的清晰度和可维护度很有帮助，它们会产生更高的内存占用，因为会产生协程追踪对象。

如果一个协程几乎每帧都会执行且在长期操作中不会yield，把它替换成`Update`或`LateUpdate`回调通常会更加可读。对于长时间执行或无限循环协程更是如此。

当一个object被disabled时，协程不会被停止，只有在object被明确的destroy时才会停止。这允许协程保持执行，如果需要的话，也可以再次enable这个object。调用`Destroy(this)`会立刻触发`OnDisable`，且协程会被处理。最后，`OnDestroy`会在帧结束的时候被执行。（译注：这段感觉说的乱七八糟。。大概就是说disable不会停止协程，只有destroy才会停。。）

要记住协程不是线程。在协程中执行的同步操作仍然会运行在主线程上。如果目标是要减少主线程上CPU时间的占用，在协程中同样需要注意避免阻塞操作。

协程最好被用来处理长时间的异步操作，如等待HTTP传输，资源加载或文件I/O等。