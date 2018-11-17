---
title: "Unity_optimization_5"
date: 2018-11-16T20:54:02+08:00
draft: false
tags: [Unity3D]
categories: [translate]
---


> 本篇是对Unity官方最佳实践的翻译，[原文地址](https://docs.unity3d.com/Manual/BestPracticeUnderstandingPerformanceInUnity4-1.html)


# 理解托管堆

Unity开发者面对的另一个常见问题是托管堆不可预期的扩展。在Unity中，托管堆更容易扩大而不是缩小。更进一步，Unity的垃圾回收策略容易导致内存碎片，这可能会阻碍大的堆进行缩小。

## 托管堆如何工作且它为什么会扩展

托管堆是一段被项目脚本运行时（Mono或IL2CPP）自动管理的内存。所有在托管代码里创建的对象都必须分配到托管堆上（严格的说，所有非空引用类型对象和所有装箱的值类型对象必须被分配到托管堆上）。

![image0](/img/UnderstandingPerformanceinUnity-AssetAuditingSection_image_0.png)

在上图中，白色的方框表示分配给托管堆的内存数量，其中有颜色的方框表示存储在托管堆内存中的数据值。当需要更多的值时，会从托管堆中分配更多的空间。

垃圾回收会定期的运行（注意：准确的时间因平台而异）。它会扫瞄堆上所有的对象，标记任何没有被引用的对象以便删除。未被引用的对象然后被删除，释放内存空间。

关键问题在于，Unity的垃圾回收-使用[Boehm GC算法](https://en.wikipedia.org/wiki/Boehm_garbage_collector)-是非世代且非紧凑的。“非世代”意味着GC在执行采集时，必须扫描整个堆，因此其性能会随着堆的扩展而降低。“非紧凑”意味着内存中的对象不会重新分配以便消除对象之间的空隙。

![image0](/img/UnderstandingPerformanceinUnity-AssetAuditingSection_image_1.png)


上面的图展示了关于内存碎片的例子。当对象被释放，其内存就空了出来。然而，这段空闲的内存**并没有**成为一整块大的空闲内存的一部分。在空闲内存两侧的对象可能仍然在被使用。因为这个，这段空闲内存在内存段中形成了一个空隙（在图中用红圈表示出来）。因此这段新空闲的空间只能被用来存储大小一样或更小的对象。

当为对象分配空间时，记住对象必须总是占据一块连续的内存空间。

这会导致内存碎片的核心问题：尽管堆中空闲内存的总量看起来很充足，可能这些空间部分或全部分散在已分配对象之间的小空隙中。在这种情况下，尽管看起来有足够的空间来满足某次分配，托管堆仍然无法找到一块足够大的连续内存用于这次分配。

![image0](/img/UnderstandingPerformanceinUnity-AssetAuditingSection_image_2.png)

无论如何，像上图中展示的那样，如果要分配一个大对象但是没有足够的连续空闲空间来安放这个对象，Unity内存管理器会执行下面两个操作。

首先，如果还没有运行过的话，垃圾回收机制会执行。这会尝试释放足够的内存来满足这次分配请求。

如果在GC执行完以后，还是没有足够的连续内存，就必须扩展堆。堆扩展时具体的大小是平台相关的；不过，大部分Unity平台会把托管堆的大小翻倍。

## 堆的关键问题

托管堆扩展带来的核心问题是双重的：

* Unity在托管堆扩展时，不会经常去释放分配给堆的内存页；它会乐观的保留扩展堆的引用，即使它大部分都是空的。这是为了防止在未来有大量分配发生时需要重新进行堆的扩展。
* 在大部分平台上，Unity最终会把托管堆中空闲部分的内存页释放回操作系统。但是这个行为发生的频率是无法保证的，且不应该被依赖。
* 托管堆使用的地址空间不会被释放回操作系统。
* 对于32位程序，在托管堆扩展和收缩多次后，可能会导致地址空间耗尽。如果程序的可用地址空间被耗尽，操作系统会终止程序。
* 对于64位程序，只要程序的运行时间不超过人类的平均寿命，地址空间是足够的且极不可能会耗尽。

## 临时分配

可以发现，很多Unity项目会在每帧都从托管堆分配几十或上百kb的临时数据。这对于项目的性能及其有害。考虑下面的计算：

如果一个程序每帧分配1kb的临时内存，运行在60fps，那么它会每秒分配60kb的临时内存。在一分钟之内，这会增加到3.6mb的垃圾内存。每秒都执行垃圾回收对于性能有很大的影响，但是每分钟都分配3.6mb的内存对于低端设备也是很有问题的。

更进一步，考虑加载操作。如果在一次繁重的资源加载操作中需要生成大量的临时对象，且这些对象会在操作完成前保持被引用，那么垃圾回收就无法释放这些临时对象，托管堆就需要扩展-即使这些对象很大一部分会在短时间内被释放。

![image0](/img/UnderstandingPerformanceinUnity-AssetAuditingSection_image_3.png)

保持对托管堆的监控相对简单。在Unity的CPU Profiler中，Overview里有“GC Alloc”一栏。这一栏展示了在特定帧托管堆上分配的字节数（注意：这并不等于在给定帧上临时分配的字节数。这个profile展示了特定帧上分配了的字节数，即使部分或全部内存在后续帧中被复用）（译注：这个注意没看太懂。。好像是说这里表示的是某帧的内存快照，而不是新分配的数量）。当开启了“Deep Profiling”选项时，可以追踪到发生分配时的方法。

**Unity Profiler不会追踪在主线程以外的内存分配。** 因此，“GC Alloc”栏不能被用来测量用户创建线程中的内存分配。为了debug的目的，可以把代码切换到主线程执行，或者使用[BeginThreadProfiling](https://docs.unity3d.com/ScriptReference/Profiling.Profiler.BeginThreadProfiling.html)在Timeline Profiler中展示采用结果。

应该总是在目标设备上使用`development build`在profile托管内存分配。

注意部分脚本方法会在Editor中引起内存分配，但是在项目构建以后不会产生分配。最常见的例子是`GetComponent`，这个方法在Editor中执行时总是会进行内存分配，但是在构建的项目中不会。

总的来说，强烈建议所有的开发者在项目处于可交互状态时，最小化托管堆的分配。在非交互操作时（如场景加载时）进行分配，会产生更少的问题。

Visual Studio的[Jetbrains Resharper Plugin](https://resharper-plugins.jetbrains.com/packages/ReSharper.HeapView/0.9.1)可以帮助定位代码中的内存分配。（译注：Rider也有这个插件`Heap Allocations Viewer`，可以在代码中标出会产生分配的地方）

使用Unity的[Deep Profile](https://docs.unity3d.com/Manual/ProfilerWindow.html)模式来定位托管堆分配的特定原因。在这种模式下，每个方法调用都会被单独记录，在方法调用树中提供了关于托管分配来源的清晰视图。注意Deep Profile模式不止能在Editor中运行，也可以通过命令行参数`-deepprofiling`在Android和桌面上运行。在profiling过程中，Deep Profiler按钮会被置灰。

## 内存节约的基础方法

有一些相对简单的技巧可以用来减少托管堆内存分配。

### 集合和数组复用

在使用C#的集合类或数组时，应尽可能的考虑复用或池化分配的集合或数据。集合类都会暴露`Clear`方法用来清除集合的值但是不会释放集合所分配的内存。

这对于临时分配的用来进行复杂运算的帮助集合是很有用的。下面的代码是一个很简单的例子：

```C#

void Update() {

    List<float> nearestNeighbors = new List<float>();

    findDistancesToNearestNeighbors(nearestNeighbors);

    nearestNeighbors.Sort();

    // … use the sorted list somehow …

}


```

在这个例子中，列表`nearestNeighbors`会每帧都分配一次，用来收集一组数据点。很容易就可以把这个列表提出到类中，避免了每帧都分配一个新的列表：

```C#

List<float> m_NearestNeighbors = new List<float>();

void Update() {

    m_NearestNeighbors.Clear();

    findDistancesToNearestNeighbors(NearestNeighbors);

    m_NearestNeighbors.Sort();

    // … use the sorted list somehow …

}

```

在这个版本中，这个List的内存被保留并在多个帧之间复用。只有在列表需要扩展时才会有新的内存分配。

### 协程和匿名方法

在使用协程和匿名方法时，有两点需要考虑。

首先，C#中所有方法引用都是引用类型，因此都会在堆上进行分配。通过传方法引用作为参数就可以很容易的创建临时的分配。这个分配无论被传入的方法是匿名方法还是预先定义的都是产生。

其次，把一个匿名方法转化为闭包会显著的增加把闭包传入方法时所需要的内存。

考虑如下代码：

```C#

List<float> listOfNumbers = createListOfRandomNumbers();

listOfNumbers.Sort( (x, y) =>

(int)x.CompareTo((int)(y/2)) 

);

```

这个代码片段使用了一个简单的匿名方法来控制第一行中创建的列表的排序规则。不过，如果一个程序员想让这段代码能够复用，把常量`2`替换为一个本地作用域的变量是很诱人的，像这样：

```C#

List<float> listOfNumbers = createListOfRandomNumbers();

int desiredDivisor = getDesiredDivisor();

listOfNumbers.Sort( (x, y) =>

(int)x.CompareTo((int)(y/desiredDivisor))

);

```

这个匿名方法现在需要能够访问方法作用域之外的变量，所以它变成了一个闭包。`desiredDivisor`变量必须以某种方法被传入闭包中，以便它在闭包的代码中能够使用。

为了实现这个功能，C#生成一个匿名类来保持闭包需要的外部作用域变量。在闭包被传入`Sort`方法时，这个类的一个拷贝被实例化，然后这个拷贝被使用`desiredDivisor`的值进行初始化。

由于执行闭包需要实例化它所生成的类的一个拷贝，而且C#中所有的类都是引用类型，所以执行闭包需要在托管堆上分配一个对象。

通常来说，最好在C#中尽可能避免闭包。在性能敏感的代码中，尽量少使用匿名方法和方法引用，在每帧都执行的代码中尤其如此。

### IL2CPP中的匿名方法

目前（译注：不确定现在的版本是不是还是这样，可以后续查一下），检查IL2CPP生成的代码发现，对`System.Function`类型变量的简单的声明和赋值会分配一个新的对象。无论这个变量是显式（在方法/类中声明）还是隐式（声明为一个方法的参数）都是如此。

因此，在使用IL2CPP作为脚本后端时，任何使用匿名方法时都会分配托管内存。在使用Mono脚本后端时不会这样。

而且，IL2CPP对于方法参数不同的声明方式，展现出显著的托管内存分配级别。闭包不负众望的会在调用时分配最多的内存。

违反直觉的是，在IL2CPP脚本后端中，传入预定义的方法作为参数时会分配和闭包几乎一样多的内存。匿名方法在堆上生成了最少的临时垃圾，比其他的少一个或多个数量级。

因此，如果项目要发布为IL2CPP脚本后端，推荐以下三点：

* 选择不需要传入方法为参数的编码风格
* 不可避免时，优先选择匿名方法而不是预定义方法
* 无论哪种脚本后端都应该避免闭包

### 装箱

装箱是Unity项目中最常见的非预期临时内存分配来源。它会在值类型的值被当作引用类型使用时发生；这大部分发生在把原生值类型变量（如`int`和`float`）传入对象类型方法时。

在这个极端简单的例子中，整形`x`被装箱，以便传入`object.Equals`方法，因为`object`的`Equals`方法要求传入一个`object`类型。

```C#

int x = 1;

object y = new object();

y.Equals(x);

```

C# IDE和编译器通常不会在装箱时产生警告（译注：上文提到过IDEA的插件时可以提示装箱内存分配的），即使这会导致未预期的内存分配。这是因为C#语言在开发时基于一个假设：小的临时分配会被分世代的GC和分配大小敏感的内存池高效处理。

由于Unity的分配器不会为不同大小的分配使用不同的内存池，且Unity的GC不是分世代的，因此不能高效的清除由于装箱引起的小的、频繁的临时分配。

在为Unity运行时写C#代码时，应该尽量避免装箱。

### 识别装箱

根据使用的脚本后端不同，装箱在CPU追踪中表现为一些不同的方法调用。它们通常会采用以下几种形式的一种，其中`<some class>`是类或结构体的名字，`...`是参数的数量：

* `<some class>::Box(…)`
* `Box(…)`
* `<some class>_Box(…)`

也可以通过搜索反编译器或IL查看器的输出来定位，例如ReSharper的IL查看工具或dotPeek反编译器。使用的IL指令是`box`。

### 字典和枚举

一个常见的引起装箱的原因是使用`enum`类型作为字典的key。声明一个`enum`会创建一个新的类型，其在幕后被当作整形来使用，但是会在编译时强制执行类型安全检查。

默认情况下，调用`Dictionary.add(key, value)`会引起调用`Object.getHashCode(Object)`。这个方法被用来获取字典key的合适的散列值，且会被用在所有接收key为参数的方法中：`Dictionary.tryGetValue`，`Dictionary.remove`等。

`Object.getHashCode`方法接收引用类型参数，但是`enum`值总是值类型。因此，对于使用枚举为key的字典，每次方法调用都会引起至少一次的key装箱。

下面的代码片段展示了一个上述装箱问题的简单例子：

```C#

enum MyEnum { a, b, c };

var myDictionary = 

new Dictionary<MyEnum, object>();

myDictionary.Add(MyEnum.a, new object());

```

为了解决这个问题，有必要实现一个自定义类实现`IEqualityComparer`接口，然后把这个类的实例设置为字典的comparer（注意：这个对象通常无状态，所以可以被多个字典复用来节约内存）。

下面是`IEqualityComparer`的一个简单的例子：

```C#

public class MyEnumComparer : IEqualityComparer<MyEnum> {

    public bool Equals(MyEnum x, MyEnum y) {

        return x == y;

    }

    public int GetHashCode(MyEnum x) {

        return (int)x;

    }

}

```

上面类的实例可以被传入字典的构造方法中。

（译注：使用枚举作为key好像还挺常见的，但是真的用起来这么麻烦的吗。。。需要进一步实践一下）

### Foreach循环

在Unity版本的Mono C#编译器中，使用`foreach`循环强制Unity在每次循环结束时进行一次装箱（注意：只有在循环整体结束运行时，才会进行一次装箱。并不是每执行一次循环装箱一次，因此无论循环了2次还是200次，这部分内存消耗是一样的）。这是由于Unity的C#编译器生成的IL会创建一个通用的值类型Enumerator以能够遍历值的集合（译注：这个原因没看太懂。。）。

这个Enumerator实现了`IDisposable`接口，其必须在循环结束时被调用。然而，在值类型（例如结构体和Enumerator）上调用接口方法需要把它们进行装箱。

可以查看下面这段非常简单的示例代码：

```C#

int accum = 0;

foreach(int x in myList) {

    accum += x;

}

```

上面的代码在经过Unity的C#编译器后，会产生下面的IL：

```IL

	.method private hidebysig instance void 

    ILForeach() cil managed 

  {

    .maxstack 8

    .locals init (

      [0] int32 num,

      [1] int32 current,

      [2] valuetype [mscorlib]System.Collections.Generic.List`1/Enumerator<int32> V_2

    )

    // [67 5 - 67 16]

    IL_0000: ldc.i4.0     

    IL_0001: stloc.0      // num

    // [68 5 - 68 74]

    IL_0002: ldarg.0      // this

    IL_0003: ldfld        class [mscorlib]System.Collections.Generic.List`1<int32> test::myList

    IL_0008: callvirt     instance valuetype [mscorlib]System.Collections.Generic.List`1/Enumerator<!0/*int32*/> class [mscorlib]System.Collections.Generic.List`1<int32>::GetEnumerator()

    IL_000d: stloc.2      // V_2

    .try

    {

      IL_000e: br           IL_001f

    // [72 9 - 72 41]

      IL_0013: ldloca.s     V_2

      IL_0015: call         instance !0/*int32*/ valuetype [mscorlib]System.Collections.Generic.List`1/Enumerator<int32>::get_Current()

      IL_001a: stloc.1      // current

    // [73 9 - 73 23]

      IL_001b: ldloc.0      // num

      IL_001c: ldloc.1      // current

      IL_001d: add          

      IL_001e: stloc.0      // num

    // [70 7 - 70 36]

      IL_001f: ldloca.s     V_2

      IL_0021: call         instance bool valuetype [mscorlib]System.Collections.Generic.List`1/Enumerator<int32>::MoveNext()

      IL_0026: brtrue       IL_0013

      IL_002b: leave        IL_003c

    } // end of .try

    finally

    {

      IL_0030: ldloc.2      // V_2

      IL_0031: box          valuetype [mscorlib]System.Collections.Generic.List`1/Enumerator<int32>

      IL_0036: callvirt     instance void [mscorlib]System.IDisposable::Dispose()

      IL_003b: endfinally   

    } // end of finally

    IL_003c: ret          

  } // end of method test::ILForeach

} // end of class test

```

最相关的代码是结尾附近的`__finally { … }__`代码块。`callvirt`指令在执行`IDisposable.Dispose`方法前，找出其在内存中的地址，执行它需要Enumerator被装箱。

总的来说，在Unity中需要避免`foreach`。不仅是因为它们会装箱，也因为通过Enumerator来遍历集合引起的方法调用消耗比手动通过`for`或`while`遍历要大的多。

注意在Unity 5.5中对C#编译器的升级显著的改进了Unity生成IL的能力。特别的，消除了`foreach`循环中的装箱操作。这消除了有关`foreach`循环的内存占用。然而，由于额外的函数调用，其CPU性能还是比基于数组的循环要差。

### 返回值为数组的Unity API

一个更有害且更不容易被发现的不必要的数组（译注：原文是spurious，好像没找到很恰当的翻译）分配的原因，是重复访问返回数组的Unity API。所以返回数组的Unity API，在每次被访问时都会创建一个新的数组拷贝。不必要的访问返回数组的Unity API是很低效的。（译注：写过C的同学应该都明白这个，可以传入一个指针作为返回值来避免这种不必要的分配，后面的章节还是视频中讲到了类似的优化）

作为例子，下面的代码不必要的在每次循环时都创建了4个`vertices`数组的拷贝。这个分配发生在每次`.vertices`属性被访问时。

```C#

for(int i = 0; i < mesh.vertices.Length; i++)

{

    float x, y, z;

    x = mesh.vertices[i].x;

    y = mesh.vertices[i].y;

    z = mesh.vertices[i].z;

    // ...

    DoSomething(x, y, z);   

}

```

这可以很容易的通过在进入循环前保存`vertices`数组，被重构为一次数组分配，而无论循环了多少次：

```C#

var vertices = mesh.vertices;

for(int i = 0; i < vertices.Length; i++)

{

    float x, y, z;

    x = vertices[i].x;

    y = vertices[i].y;

    z = vertices[i].z;

    // ...

    DoSomething(x, y, z);   

}

```

尽管访问一次属性的CPU消耗不是很高，在循环中重复的访问也会造成CPU性能热点。进一步的，不必要的重复访问会扩展托管堆。

这个问题在移动设备上非常常见，因为`Input.touches`API和上述的表现类似。项目很容易包含类似下面的代码，其中每次对`.touches`的访问都会引起分配。

```C#

for ( int i = 0; i < Input.touches.Length; i++ )

{

   Touch touch = Input.touches[i];

    // …

}


```

当然，这可以很容易的通过把数组分配提到循环之外来改进：

```C#

Touch[] touches = Input.touches;

for ( int i = 0; i < touches.Length; i++ )

{

   Touch touch = touches[i];

   // …

}

```

然而，现在很多Unity API都提供了不会引起内存分配的版本。在有这些方法时应该尽量使用。

把上面的例子改成更少分配的Touch API很简单：

```C#

int touchCount = Input.touchCount;

for ( int i = 0; i < touchCount; i++ )

{

   Touch touch = Input.GetTouch(i);

   // …

}

```

注意对于属性的访问`Input.touchCount`仍然被保持在循环之外，来节约执行属性的`get`方法的CPU消耗。（译注：这里直接写在for里也没问题吧好像。。for循环应该也只会访问一次而已吧。。不是很确定这么做的好处。。）

### 空数组复用

一些开发团队，在返回数组的方法需要返回空集合时，选择返回`null`。这个编码模式经常出现在托管语言中，尤其是C#和Java。

通常，当从方法返回一个长度为0的数组时，返回一个预先分配好的长度为0的数组单例比每次都创建一个空数组要更高效（注意：当然，在返回后如果改变了这个数组的大小，需要抛出异常）。