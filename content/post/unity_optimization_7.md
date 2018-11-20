---
title: "[译]Unity优化最佳实践7-通用优化"
date: 2018-11-19T17:50:13+08:00
draft: false
tags: [Unity3D]
categories: [translate]
---

> 本篇是对Unity官方最佳实践的翻译，[原文地址](https://docs.unity3d.com/Manual/BestPracticeUnderstandingPerformanceInUnity7.html)


# 通用优化

有多少性能问题的原因，就有多少优化代码的方式。通常，强烈建议开发者在尝试进行CPU优化之前仔细的profile他们的程序。不过，也有一些很通用的简单CPU优化条目。

## 通过ID寻址属性

在Unity内部，对于Animator，Material和Shader属性的寻址不是通过字符串名称进行的（译注：这一节中的寻址address，应该不是真正意义上的内存寻址，而是应该理解为称呼这种，不过这里还是翻译成寻址比较好看一些。。。）。处于速度考虑，所有的属性都被哈希成属性ID，实际上通过这些ID来寻址属性。

因此，在Animator，Material和Shader上使用Set或Get方法时，使用整形参数的方法来代替字符串参数的方法。这些字符串方法只是简单的进行字符串的哈希，然后把哈希后的ID传递给整形参数的方法。

在一次运行过程中，通过字符串哈希而来的属性ID是确定的。使用它们最简单的办法是为每个属性名声明一个静态只读整形变量，然后在使用字符串的地方转而使用这些整形变量。它们会在启动时自动的初始化并且不需要额外的初始化代码。（译注：这里没太明白，我在哪声明静态只读变量？怎么就自动初始化了？不需要我手动哈希出来赋值保存吗？如果不需要那下段里给出的哈希方法是干啥的？需要在代码里实际去验证一下。。）

对于Animator属性名，对应的API是[Animator.StringToHash](https://docs.unity3d.com/ScriptReference/Animator.StringToHash.html)，对于Material和Shader属性名是[Shader.PropertyToID](https://docs.unity3d.com/ScriptReference/Shader.PropertyToID.html)。

## 使用非分配式物理API

在Unity 5.3及以后，对所有物理查找API都引入了非分配版本。使用[RaycastNonAlloc](https://docs.unity3d.com/ScriptReference/Physics.RaycastNonAlloc.html)替换[RaycastAll](https://docs.unity3d.com/ScriptReference/Physics.RaycastAll.html)，[SphereCastNonAlloc](https://docs.unity3d.com/ScriptReference/Physics.SphereCastNonAlloc.html)替换[SphereCastAll](https://docs.unity3d.com/ScriptReference/Physics.SphereCastAll.html)，等等。对于2D应用，也有2D物理的非分配版本。（译注：前文中也提到过，这其实就是自己传入一个结果数组，而避免方法返回结果数组，以避免在返回及赋值时产生的临时内存分配）

## 对UnityEngine.Object子类的null判断

Mono和IL2CPP运行时以一种特殊的方法来处理[UnityEngine.Object](https://docs.unity3d.com/ScriptReference/Object.html)子类的实例。在这些实例上执行方法，实际上会调用进引擎代码，这必须执行查找和验证工作来把脚本引用转化为原生引用。虽然很小，把这类变量和null进行对比会比和纯C#类对比代价高很多。处于这个原因，应该避免在频繁循环中或每帧执行的代码中进行null比较。

## Vector和quaternion数学运算和操作顺序

对于频繁循环中的vector和quaternion运行，记住整形运算比浮点运算要快，浮点运算比vector、matrix或quaternion运算要快。

因此，在交换律和结合律允许的情况下，尝试最小化每个计算的操作：

```C#
Vector3 x;

int a, b;

// 不够高效：导致两次vector相乘

Vector3 slow = a * x * b;

// 更高效：一次整形乘法，一次vector乘法

Vector3 fast = a * b * x;
```

## 内置ColorUtility

程序经常有需求把颜色在HTML格式字符串（#RRGGBBAA）和Unity内置的`Color`和`Color32`结构之间通过Unity社区的一个脚本进行转换。这个脚本既慢又会因为字符串处理引起额外的内存分配。

在Unity 5中，提供了一个内置的[ColorUtility](https://docs.unity3d.com/ScriptReference/ColorUtility.html)API来高效的执行这个转换。应该尽可能选择这个内置的API。

## Find和FindObjectOfType

通常，在生产代码中消除所有对`Object.Find`和`Object.FindObjectOfType`的使用是最佳实践。由于这些API需要Unity遍历内存中所有的GameObject和Component，它们随着项目规模的增长很快变得没有效率。

上面的规则有一个例外，就是在单例对象的get中。一个全局manager对象经常暴露一个“instance”属性，且经常在getter中使用`FindObjectOfType`来检查之前存在的单例实例：

```C#
class SomeSingleton {

    private SomeSingleton _instance;

    public SomeSingleton Instance {

        get {

            if(_instance == null) { 

                _instance =

                    FindObjectOfType<SomeSingleton>(); 

            }

            if(_instnace == null) { 

                _instance = CreateSomeSingleton();

            }

            return _instance;

        }

    }

}
```

尽管这个模式通常是可以接受的，重点是要检查代码来确保这个getter被调用的场景中不存在这个单例对象。如果这个getter不会自动创建不存在的单例实例，就会经常发现获取单例的代码会重复的调用`FindObjectOfType`（通常每帧很多次），这会造成不受欢迎的性能耗尽。

### 摄像机定位器

在内部，Unity的`Camera.main`属性调用`Object.FindObjectWithTag`方法，这是`Object.FindObject`的一个专用变体。访问这个属性并不比调用`Object.FindObjectOfType`更高效。如果代码必须获取主摄像机，强烈推荐使用下两种方法的一种：

* 在`Start`或`OnEnable`回调中访问`Camera.main`并缓存返回的引用
* 构建一个`Camera Manager`类来提供或注入活动摄像机的引用

## Debug代码和[conditional]属性

`UnityEngine.Debug`日志API不会在非开发构建中被去除，如果被调用的话仍然会写入日志文件。由于大部分开发者并不想在非开发构建中写入debug信息，推荐把仅用于开发的日志调用封装成自定义的方法，像这样：

```C#
    public static class Logger {

        [Conditional("ENABLE_LOGS")]

        public static void Debug(string logMsg) {

            UnityEngine.Debug.Log(logMsg);

        }

    }
```

通过使用[conditional]属性修饰这些方法，可以通过是否定义条件属性来控制这些方法是否被包含进供编译的编码。

如果这些属性中的条件都没有被定义，那么被修饰的方法和所有对这些方法的调用都会被编译器忽略。这个效果和使用`#if...#endif`预编译指令包裹相应代码是一致的。

可以访问[MSDN网站](https://msdn.microsoft.com/en-us/library/4xssyw96(v=vs.90).aspx)来获取关于`Conditional`属性的更多信息。