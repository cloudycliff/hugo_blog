---
title: "[译]Unity资源最佳实践1-Assets，Objects和序列化"
date: 2018-10-23T18:57:27+08:00
draft: false
tags: [Unity3D]
categories: [translate]
---

> 本篇是对Unity官方最佳实践的翻译，原文地址[https://unity3d.com/learn/tutorials/topics/best-practices/assets-objects-and-serialization](https://unity3d.com/learn/tutorials/topics/best-practices/assets-objects-and-serialization) 

Assets，Objects和序列化
=

本章讲解了Unity序列化系统的深层实现，以及Unity如何在Editor和运行时保持不同Objects的灵活关联关系（译注：这里的序列化是指落地存储，反序列化就是把文件加载到内存里形成结构化数据，序列化在Unity引擎的实现中是一个很重要的概念，类中的数据可以在Editor中修改、在Play模式中修改数据和恢复等都是通过序列化来实现的）。然后讨论了Objects和Assets在技术上的区别。本章包含的知识点是理解如何在Unity中有效的load和unload资源的基础。合理的Asset管理对于缩短加载时间和降低内存占用都是十分关键的。

1.1 深入Assets和Objects
==

想要理解如何在Unity中合理的管理data，首先要理解Unity是如何标识和序列化data的。第一个要点是Assets和UnityEngine.Objects的区别。

一个Asset是一个存储在硬盘上的文件，位于Unity项目的Assets目录下。贴图、3D模型、音频等都是常见的Assets类型。有些Assets包含Unity自定义的数据格式，如材质materials。有些Assets需要经过处理转换成内部格式，如FBX文件（译注：3D建模软件导出的模型文件）。

一个UnityEngine.Object，是一组序列化的数据的集合，用来描述一个特定的资源。这可以是Unity使用的任何一种资源，如mesh、sprite、AudioClip或AnimationClip。所有的Objects都是UnityEngine.Object的子类。

在所有的内置Object类型中，有两个比较特殊的类型：

* ScriptableObject为开发者提供了一种可以方便的自定义数据类型的系统。这种类型可以被Unity原生的序列化和反序列化，并且可以方便的在Unity Editor的窗口中管理。（译注：ScriptableObject有很多的优点，特别适合配置表、配置文件等）

* MonoBehavior提供了一个链接到MonoScript的封装。MonoScript是一种内置的数据类型，它指向了一个特定的dll和namespace中的一个特定的脚本class。MonoScript本身并不包含任何可执行代码。

Assets和Objects是一种一对多的关系，一个Asset文件可以包含一个或多个Objects。

1.2 Object之间的引用
==

UnityEngine.Objects可以引用其他的Objects，这个Objects可以来自同一个Asset文件，也可以来自其他的Asset文件。例如，一个material Object通常会引用一个或多个texture Objects。这些texture Objects一般都是从一个或多个texture Asset文件中引入的（如png或jpg文件）。

在序列化存储时，这些引用包含两部分数据：File GUID和Local ID。File GUID标明了目标资源存储的Asset文件。由于一个Asset文件可能会包含多个Objects，每一个Object还会被分配一个本地唯一的Local ID作为标识。

File GUIDs存储在.meta文件中。这些.meta文件会在Unity第一次导入Asset时生成，并且会和Asset存储在同一个目录下。

以上的定义和引用系统可以通过文件编辑器查看：在Unity项目中，把Editor Settings修改为Visible Meta Files和serialize Assets as text（译注：这两个是Unity项目进行版本管理时的常用设置方式，现在好像已经成为默认的选项了？）。创建一个材质，并导入一个贴图。把材质赋到一个cube上，保存场景。

在文件编辑器中，打开材质对应的.meta文件。在文件顶部会有一行标为guid，这就是这个材质Asset的File GUID。想要查看Local ID，在文本编辑器中直接打开材质文件。这个材质文件类似下面的形式：

```
%YAML 1.1
%TAG !u! tag:unity3d.com,2011:
--- !u!21 &2100000
Material:
  serializedVersion: 6
.......
```

在上面的例子中，以&开头的数字就是材质的Local ID。如果这个material Object被存放在一个以File GUID “abcdefg”标识的Asset中，这个material Object就可以通过File GUID “abcdefg”和Local ID “2100000”来唯一的标识。

1.3 为什么同时需要File GUID和Local ID？
==

答案是在兼顾鲁棒性的同时，提供一个灵活的、平台无关的工作流程。

File GUID提供了一个文件存储位置的抽象。使用一个特定的File GUID来关联一个特定的文件，该文件的实际存储位置就可以解耦。可以随意的移动文件的位置，而不必要去修改所有引用了该文件的Objects。

由于一个Asset文件可能包含多个UnityEngine.Object资源，Local ID对于区分不同的Object就很有必要。

如果和一个Asset文件关联的File GUID丢失了，和这个Asset文件所关联的所有Objects都会丢失。所以保持.meta文件和他们所对应的资源使用同样的命名并存储到同样的目录下是很重要的。注意Unity会重新生成被删除的或错放的.meta文件。

Unity Editor维护了一个文件路径到File GUID的映射表。当一个Asset被加载或者导入时，会更新对应的记录条目。在Unity Editor打开的情况下，如果.meta文件被删除而且Asset的路径没有被修改，Editor可以保证这个Asset保留同样的File GUID（译注：应该是在说会再生成一个包含同样GUID的.meta文件？）。

在Unity Editor关闭时，删除.meta文件，或者移动Asset文件但不同步移动对应的.meta文件，这个Asset对应的Objects引用就会被破坏。（译注：这种情况下，Unity无法再用原来的GUID再生成一个.meta，即该Asset的GUID变了，所以外部的引用就找不到这个Asset了）

1.4 编辑Assets和Importers
==

前文中提到，非原生的Asset必须被导入到Unity中。这是通过asset importer来实现的。通常情况下importers会被自动执行，它们也通过AssetImporter API提供了脚本接口。例如，TextureImporter API提供了在导入贴图资源（如png文件）时，访问贴图设置的功能。

导入流程的产出是一个或多个UnityEngine.Objects。在Unity Editor中，可以以父Asset和多个子assets的方式查看，例如导入一个sprite图集后，会展现成一个贴图Asset下有多个sprites的形式。这些Objects共享同一个File GUID，因为他们的原始数据保存在同一个Asset文件中。它们通过贴图Asset中的Local ID来进行区分。

导入流程把原始Asset转换成适合Unity Editor中选定的目标平台的格式。此流程可能会包含很多复杂的操作，如图片的压缩。因为这种操作通常很耗时，导入的Asset会被缓存到Library文件夹中，减少下次启动时重新导入的时间。（TODO：这个难道不是导出安装包的时候进行的吗？还是说不同的平台会分别进行缓存？待后续验证）

具体来说，导入流程会把资源按照Asset的File GUID的前两位分文件夹进行存储，存储在Library/metadata/文件夹下。Asset对应的Objects被序列化到一个二进制文件中，以Asset的File GUID命名存储。

这个流程针对所有的Assets类型，不仅针对非原生Assets。原生Assets不需要经过冗长的转换流程或重新序列化。（译注：这里应该是说原生Asset也会进行缓存，不过不需要进行转换了）

1.5 序列化和实例化
==

尽管File GUID和Local ID方式很健壮，GUID的对比效率还是比较底下，在运行时需要一个更加高效的系统。在Unity内部维护了一个转换File GUID和Local ID到简单、运行session内唯一的integers的映射缓存（注：在Unity的内部实现中，这个缓存叫做PersistentManager）。这些被称为Instance ID，在新的Objects注册到缓存时，通过一种简单的单增顺序来生成。

这个缓存保持了Instance ID，定义了Object原数据的位置的File GUID和Local ID，还有内存中Object实例（如果有的话）之间的映射。这使得UnityEngine.Objects可以方便的互相引用。通过Instance ID可以快速的找到已加载的对应的Object。如果目标Object没有被加载，通过File GUID和Local ID可以解析出Object的原数据，让Unity可以及时的加载数据。

在启动时，会对所有立刻需要的Objects（例如被初始Scenes引用）和所有在Resources目录下的Objects建立Instance ID缓存。当新的资源在运行时被加载（注：例如在运行时通过脚本创建一个贴图资源：var myTexture = new Texture2D(1024, 768)。译注：这好像在一些渲染技术上比较常用，通过逻辑临时生成一些贴图）或者Objects从AssetBundles加载时，会向缓存中添加新的条目。只有当unload一个AssetBundle时，对应的资源Instance ID条目才会被删除，对应的映射关系都会被清除用来节省内存。如果AssetBundle被重新加载，AssetBundle里每个Object都会被分配新的Instance ID。

在后续的章节中，会针对AssetBundle的unload进行更深入的讨论。

在特定的平台上，一些事件会强行把Objects移出内存。例如，在iOS系统上，当应用被暂停时，图形Assets可能会从图形内存中删除。如果这些Objects源自一个被unload的AssetBundle，Unity无法重新加载这些Objects的原数据。这些Objects的外部引用都会无效。在这个例子中，这个场景会表现出不可见的网格或洋红色的贴图。（译注：确实在一些Unity做的游戏中遇到过这个情况。这段可以结合后面讲解资源生命周期的部分来看，即以非强制方式unload了AssetBundle后，Unity已经失去了对这些Objects的引用，这时Objects被OS移出内存，Unity无法用原来的Instance ID来再次加载它们，因为这些Instance ID对应的缓存条目已经删除了，无法找到对应的File GUID和Local ID）

实现细节提示：在运行时，上述的控制流程的描述并不完全准确。在运行时，在大量加载操作时，对比File GUID和Local ID并不足够有效率。在构建Unity项目时，File GUID和Local ID会被决定性的映射到一个更简单的形式。不过，这里的理念是一样的，而且在运行时用File GUID和Local ID来思考仍然是个有用的类比。这也是在运行时无法索引到Asset的File GUID的原因。

1.6 MonoScripts
==

重点：要明白一个MonoBehaviour包含一个对MonoScript的引用，一个MonoScript只是简单的包含了定位到一个特定脚本类的信息。这两种类都不包含可运行的脚本类代码。

一个MonoScript包含了三个字符串：assembly名，类名，namespace。

当构建一个项目时，Unity把Assets目录下松散的脚本文件编译到Mono assemblies中。在Plugins文件夹外面的C#脚本会被放到Assembly-CSharp.dll中。在Plugins目录下的脚本被放到Assembly-CSharp-firstpass.dll中。另外，从Unity 2017.3开始引入了自定义assemblies的功能。

这些assemblies和预编译的DLL文件，会被包含到最终的Unity应用中。这些也是MonoScript所引用的assemblies。在Unity应用启动时，所有的assemblies都会被加载。

MonoScript Object是AssetBundle（或者Scene或prefab）不包含任何可执行代码的原因。这允许不同的MonoBehaviours引用特定的共享类，即使这些MonoBehaviours处于不同的AssetBundles中。（TODO：这段不是很理解。。Mono可以放到AB中？）

1.7 资源生命周期
==

为了减少加载时间和管理应用的内存占用，理解UnityEngine.Objects资源的生命周期是很重要的。资源会在特定的时间加载或释放。

在以下条件都满足的情况下，Object会被自动加载：

* Object对应的Instance ID被寻址（译注：这里用的是dereferenced，类似指针的寻址，其实就是这个ID被解析了）
* 这个Object当前没有被加载到内存
* 这个Object的原数据可以被定位到

Objects也可以通过脚本显式的加载，可以是通过创建的方式或通过资源加载API（例如AssetBundle.LoadAsset）。当Object被加载时，Unity会尝试将所有引用到的资源的File GUID和Local ID转换为Instance ID。当一个Object的Instance ID被寻址时，在以下两个条件满足的情况下，Object会被按需加载：

* 这个Instance ID指向的Object当前没有被加载
* 在缓存中已经注册了这个Instance ID所对应的有效的File GUID和Local ID

上面的过程通常在引用被加载和解析后很短的时间内发生。

如果一个File GUID和Local ID无法对应到一个Instance ID，或者一个Instance ID对应的File GUID和Local ID是无效的，这个引用所对应的Object不会被加载。在Unity Editor中会表现为“Missing”引用状态。在运行时或在Scene视图中，“Missing”的Objects会根据类型表现出不同的形式，例如，丢失的mesh会无法显示，丢失的贴图会表现出洋红色。

在以下三个特定的场景下，Object会被unload：

* 在清理未使用的Asset时，Objects会被自动unload。这个过程会在以下情况下触发：当场景被破坏性的切换（如以非叠加的形式调用SceneManager.LoadScene时），或者当脚本调用Resources.UnloadUnusedAssets时。这个过程只会unload未被引用的Objects。只有在没有被Mono变量引用，或没有被其他活跃的Objects引用的情况下，Object才会被unload。另外要注意，被标记为HideFlags.DontUnloadUnusedAsset或HideFlags.HideAndDontSave的资源不会被unload。
* 加载自Resources目录的Objects可以被Resources.UnloadAsset显式的unload。其所对应的Instance ID还会保留，当Mono变量或其他Object需要用到这个Object时，会及时的进行重新加载。
* 源自AssetBundle的Objects会在调用AssetBundle.Unload(true)时被立刻unload。这个操作会使Object的Instance ID变得无效，其他的引用会变为“Missing”状态。在C#脚本中，尝试访问一个unload的Object的方法和属性会抛出一个NullReferenceException异常。

在AssetBundle.Unload(false)被调用时，AssetBundle中活跃的Objects不会被销毁，但是Unity会删除相关的Instance ID和File GUID和Local ID的对应关系。当这些Objects被从内存中unload，但是对他们的引用还存在时，Unity无法重新加载它们。（注：Objects在运行时被直接从内存移除而不通过unload的情况，通常发生在Unity失去对图形上下文控制时。这可能发生在移动app被暂停切到后台的时候。在这种情况下，移动操作系统会清空GPU内存中所有的图形资源。当app返回前台时，Unity必须在场景恢复渲染之前重新加载所有需要的贴图、shader、网格资源到GPU）（TODO：这段好像在说调用非强制unload时，会引起资源的不受管理，可能会导致资源的重复加载或内存的泄露？而且这种情况要怎么处理？如何判断这些资源被清除了？如何重新加载？还是说就应该避免使用非强制的方式？）

1.8 加载大的层级结构
==

在序列化Unity GameObjects的层级结构时，如在prefab的序列化过程中，很重要的一点是整个层级会被完整的序列化。也就是说，层级中的每个GameObject和组件都会独立的体现在序列化数据中。这会对GameObject的层级加载和初始化时间造成有趣的影响。

在创建GameObject层级时，CPU时间被消耗在以下方面：

* 读取原始数据（从存储介质中，从AssetBundle中，从另一个GameObject等）
* 建立Transform之间的父子关系（译注：这里会涉及到坐标系变换的计算等，包括位置和缩放的计算）
* 初始化GameObject和组件
* 在主线程上激活新的GameObject和组件

后三个时间消耗与层级的来源基本无关。但是读取原始数据消耗的时间是随着层级中GameObject和组件数量线性增长的，并且受到存储介质速度的影响。

在所有平台上，从内存中加载比从硬盘中加载要快很多。另外，不同平台的不同存储介质的表现也有差别。因此从较慢的存储介质加载prefab的时间会很快超过初始化这个prefab所需要的时间。这导致加载操作的效率直接和IO的效率相关。

上文提到过，当序列化一个巨大的prefab时，每个GameObject和组件的数据都会被单独的序列化，这可能会导致重复的数据。例如，一个包含了30个相同元素的UI界面会把这个相同的元素序列化30次，产生一个巨大的二进制数据文件。在加载时，这30个重复的元素都需要从硬盘中读取并初始化。磁盘的读取时间会显著的影响prefab的加载时间。大的层级结构应该按照模块进行初始化，然后在运行时被组合在一起。（TODO：这里是说一个元素在场景中出现了多次时，这些元素实际的数据会被多次存储？如果做成prefab还会不会这样？还是说就应该避免使用复杂层级？待进一步的调研）

Unity 5.4 提示：Unity 5.4修改了transform在内存中的表现形式。每个根transform的整个子层级会以紧凑的连续的方式在内存中存储。当初始化一个GameObject并需要在另一个层级中使用时，考虑使用可以指定parent节点的GameObject.Instantiate函数的重载。使用这个方法可以避免对根transform的内存分配。测试显示，这会最多提升5-10%的加载速度。
