---
title: "[译]Unity优化最佳实践6-字符串和文本"
date: 2018-11-19T16:34:02+08:00
draft: false
tags: [Unity3D]
categories: [translate]
---


> 本篇是对Unity官方最佳实践的翻译，[原文地址](https://docs.unity3d.com/Manual/BestPracticeUnderstandingPerformanceInUnity5.html)



# 字符串和文本

操作字符串和文本是Unity项目中常见的性能问题来源。在C#中，所有的字符串都是不可变的。对字符串的任何修改都会导致分配一个全新的字符串。这样做的代价相对较高，并且对大字符串、大数据集或在高频的循环中重复进行字符串连接可能会造成性能问题。

更进一步，由于N个字符串的拼接需要生成N-1个中间字符串，连续的拼接也可能是造成托管内存压力的主要原因。

如果必须在高频循环或每帧对字符串进行连接，使用StringBuilder来进行实际的连接操作。还可以复用这个StringBuilder来进一步降低不必要的内存分配。

微软维护了一个关于C#中字符串的最佳实践列表，可以在[MSDN页面](https://msdn.microsoft.com/en-us/library/dd465121(v=vs.110).aspx)找到。

## 字符集强制转换和按序比较

在和字符串相关的代码中，一个关键的性能问题是无心的使用了缓慢的默认字符串API。这些API是为商业应用构建的，它们会尝试根据文本中的字符处理来自不同文化和语言规则的字符串。

举个例子，下面的示例代码当运行在US-English字符集时返回true，但是在很多欧洲字符集下返回false。（注意：Unity 5.3和5.4版本中，Unity脚本运行时总是运行在en-US字符集下）

```C#
    String.Equals("encyclopedia", “encyclopædia”);
```

对于大部分Unity项目来说，这是完全没有必要的。使用按序比较类型会快大概10倍，它使用类似C和C++的比较方式：简单的按序对比字符串中字节的值，而并不关心这个字节所代表的字符是什么。

转换成按序比较字符串很简单，只要把`StringComparison.Ordinal`作为最后一个参数传入`String.Equals`：

```C#
myString.Equals(otherString, StringComparison.Ordinal);
```

## 低效的内置字符串API

在切换成按序比较之外，一些C#的字符串API也是十分低效的。包括`String.Format`，`String.StartsWith`和`String.EndsWith`。`String.Format`不太容易被代替，但是低效的字符串比较方法可以被很容易的优化。

尽管微软推荐在不需要考虑字符集转换的情况下，把`StringComparison.Ordinal`作为参数传入字符串比较函数，Unity的基准测试表明，这样做带来的优化比使用自定义的实现带来的影响小很多。

| 方法 | 100k短字符串消耗的时间（ms） |
|-----|---------------------------|
| `String.StartsWith`，默认字符集 | 137 |
| `String.EndsWith`，默认字符集 | 542 |
| `String.StartsWith`，按序 | 115 |
| `String.EndsWith`，按序 | 34 |
| 自定义`StartsWith` | 4.5 |
| 自定义`EndsWith` | 4.5 |

`String.StartsWith`和`String.EndsWith`可以被简单的手写代码版本代替，和下面的示例类似：

```C#
    public static bool CustomEndsWith(string a, string b) {
        int ap = a.Length - 1;
        int bp = b.Length - 1;

        while (ap >= 0 && bp >= 0 && a [ap] == b [bp]) {
            ap--;
            bp--;
        }
        return (bp < 0 && a.Length >= b.Length) || 

                (ap < 0 && b.Length >= a.Length);
    }

    public static bool CustomStartsWith(string a, string b) {
        int aLen = a.Length;
        int bLen = b.Length;
        int ap = 0; int bp = 0;

        while (ap < aLen && bp < bLen && a [ap] == b [bp]) {
        ap++;
        bp++;
        }

        return (bp == bLen && aLen >= bLen) || 

                (ap == aLen && bLen >= aLen);
    }

```

## 正则表达式

虽然正则表达式是匹配和处理字符串的有力工具，它们的性能可能非常的低效。进一步，由于C#库对正则表达式的实现方式，即使简单的boolean查询`IsMatch`也会在底层分配大量的临时数据结构。这个临时的托管内存波动被认为是不可接受的，除非是在初始化阶段。

如果必须要使用正则表达式，强烈推荐不要使用接受一个字符串正则表达式为参数的静态方法`Regex.Match`或`Regex.Replace`。这些方法动态的编译正则表达式，并且不会缓存生成的对象。

下面是一行看似无害的示例代码：

```C#
Regex.Match(myString, "foo");
```

然而，每次它被执行，都会生成5KB的垃圾。一个简单的重构会消除大部分的垃圾：

```C#
var myRegExp = new Regex("foo");

myRegExp.Match(myString);
```

在这个例子中，每次调用`myRegExp.Match`“仅仅”会产生320字节的垃圾。尽管这对于一个简单的匹配操作来说还是很多，它对于之前的例子是一个很大的改进。

因此，如果正则表达式是一个不变的字符串，把它作为正则对象构造函数的参数传入以进行预编译是更高效的。这些预编译的正则也可以被复用。

## XML，JSON和其他长文本解析

解析文本通常是加载过程中最重的操作。在某些情况下，用于解析文本上的时间可能会超过用于加载和初始化资源的时间。

这种情况的原因依赖于使用的解析方法。C#的内置XML解析器非常灵活，但是也因此，对于特定的数据结构并不高效。

很多第三方解析器都基于反射。虽然反射在开发过程中是个非常好的选择（因为它使得解析器能够快速适应数据结构的变化），总所周知它也很缓慢。

Unity通过其内置的[JSONUtility](https://docs.unity3d.com/ScriptReference/JsonUtility.html)API引入了部分解决方案，它为Unity的读写JSON的序列化系统提供了接口。在大部分基准测试中，它比纯C#的解析器要快，但是它和其他Unity序列化接口一样有所限制-在没有额外代码实现的情况下，它不能序列化很多复杂的数据结构，例如字典（参考[ISerializationCallbackReceiver](https://docs.unity3d.com/ScriptReference/ISerializationCallbackReceiver.html)接口来简单的为Unity序列化过程添加转换复杂数据结构的必要的处理）。

在遇到文本数据解析引起的性能问题时，考虑下面三种替代方案。

### 选择1:在构建时解析

避免文本解析消耗最好的办法是完全消除在运行时解析的情况。通常情况下，这意味着在某个构建步骤中把文本数据“烘焙”到二进制格式中。

大部分采用这个方法的开发者把他们的数据转移到继承自ScriptableObject的类中，然后通过AssetBundle发布。在油管上的[Richard Fine’s Unite 2016 talk](https://www.youtube.com/watch?v=VBA1QCoEAX4)演讲提供了关于使用ScriptableObject的一个绝佳的讨论。

这个策略提供了最好的性能，但是只适用于不需要动态生成的数据。这最适用于游戏设计参数及其他内容。

### 选择2:切分和懒加载

第二个可行方案是把要解析的数据拆分成小块。在拆分后，解析数据的消耗可以被分散到多个帧。理想情况下，识别出用户需要的部分并只加载这些部分。

一个简单的例子：如果项目是一个平台游戏，没必要把所有关卡的数据序列化到一个大的文件中。数据可以被按关卡拆分成单独的资源，或者按照关卡里的区域拆分，这样只有在玩家靠近这些区域时才需要解析这些数据。

尽管这听起来很简单，实际上这需要投入可观的精力在工具代码上，还可能需要重新组织代码结构。

### 选择3:线程

对于解析成纯C#对象的数据，且不需要Unity API参与的情况下，可以把解析操作转移到工作线程中进行。

这个选择对于有多个CPU核心的平台可能很有效（注意：iOS设备最多有2个核心。大部分Android有2-4个。这个技术对于构建PC和主机目标更有效。译注：现代的手机已经有更多的核心了）。不过，这要求更小心的编码以避免产生死锁和竞争。

项目中实现多线程，通常会采用C#内置的[Thread](https://msdn.microsoft.com/en-us/library/system.threading.thread(v=vs.110).aspx)和[ThreadPool](https://msdn.microsoft.com/en-us/library/system.threading.threadpool(v=vs.110).aspx)类来管理工作线程，并配合标准C#同步类使用。