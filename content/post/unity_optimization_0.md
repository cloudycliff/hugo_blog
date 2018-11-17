---
title: "Unity_optimization_0"
date: 2018-11-15T15:39:31+08:00
draft: false
tags: [Unity3D]
categories: [translate]
---

> 本文是对[Unity手册优化最佳实践](https://docs.unity3d.com/Manual/BestPracticeUnderstandingPerformanceInUnity.html)的翻译。原文应该主要针对Unity 5.x版本，所以一些建议可能已经过时了，但是很多思想还是可以借鉴的，而且很多方法和优化的点确实很有实际的指导意义。

# 理解Unity优化

本系列最佳实践是伴随Unite Unity Europe 2016上关于优化移动应用的演讲产生的。本文提供了很多相同的内容，同时也提供了很多补充资料。你可以在YouTube上观看原始演讲[视频](https://www.youtube.com/watch?v=j4YAY36xjwE)。（译注：这里强烈建议看一下这个视频，两个大哥讲的很激情，虽然我觉得一些建议有待商榷吧，但看完确实有很多启发）

尽管Unite上的演讲主要针对移动性能话题，演讲的很多内容对于普通听众也是适用的。在写本篇时，会尽力去讨论一些范围更广且平台无关的性能优化内容。

## 提纲

* [Profiling](/post/unity_optimization_1)
* [内存](/post/unity_optimization_2)
* [协程](/post/unity_optimization_3)
* [资源审核](/post/unity_optimization_4)
* [理解托管堆](/post/unity_optimization_5)
* [字符串和文本]()
* Resources文件夹  （译注：这篇几乎什么都没说，就是让大家去参考资源的最佳实践内容，所以就不单独一章了，可以参考我之前的翻译）
* [通用优化建议]()
* [特殊优化建议]()


