# Android 的图形系统的 HIDL 接口与约定
* * *

# HIDL 简介 

HIDL 全称为 HAL interface definition language（发音为“hide-l”），是用于指定 HAL 和其用户之间的接口的一种接口描述语言 (IDL)。其诞生目的是：Android Framework 层可以在无需重新构建 HAL 的情况下进行替换。 

Android 系统在 Android 8.0 中被全面使用 Project Treble 的架构，是为了解决 Android 系统的碎片化问题，以及提高系统更新的效率，减少了 framework 和 HAL 的耦合性，进而引出了 HIDL 的概念。 

采用 HIDL 的结果是：HAL 将由供应商或 SOC 制造商构建，放置在设备的 /vendor 分区中，这样一来，Android Framework 层就可以在它自己的分区中通过 OTA 进行替换，而无需重新编译 HAL。这就是设计 Project Treble 框架要解决的问题。 

此外，采用 HIDL 的还带来另外一个好处：系统的稳定性。通过 HIDL 隔离了 Framework 与 HAL。将这两者分布在不同的进程空间里，通过 RPC 和共享内存协同操作。这样如果有一方因错误而崩溃，另外一方不会随之崩溃。通过系统状态监控程序重新装载，系统还可以继续运行。

# HIDL 设计原则 

HIDL 设计在以下方面之间保持了平衡：

* 互操作性。在可以使用各种架构、工具链和编译配置来编译的进程之间创建可互操作的可靠接口。HIDL 接口是分版本的，发布后不得再进行更改。 

* 效率。HIDL 会尝试尽可能减少复制操作的次数。HIDL 定义的数据以 C++ 标准布局数据结构传递至 C++ 代码，无需解压，可直接使用。此外，HIDL 还提供共享内存接口；由于 RPC 本身有点慢，因此 HIDL 支持两种无需使用 RPC 调用的数据传输方法：共享内存和快速消息队列 (FMQ)。 

* 直观。通过仅针对 RPC 使用 in 参数，HIDL 避开了内存所有权这一棘手问题；无法从方法高效返回的值将通过回调函数返回。无论是将数据传递到 HIDL 中以进行传输，还是从 HIDL 接收数据，都不会改变数据的所有权，也就是说，数据所有权始终属于调用函数。数据仅需要在函数被调用期间保留，可在被调用的函数返回数据后立即清除。 

同时，为了 Android 系统的稳定性，Android 系统从多线程协作模式转向多进程模式。HIDL 可以使得接口和实现可以跨进程分布，并且封装了 binder 编程细节，可以快速开发接口(interface)。

下表列出了 HIDL 的软件包前缀和代码位置：

| No. | Prefix of Package    | Directory                        |
|:---:|----------------------|----------------------------------|
|  1  | android.frameworks.* | frameworks/hardware/interfaces/* |
|  2  | android.hardware.*   | hardware/interfaces/*            |
|  3  | android.system.*     | system/hardware/interfaces/*     |
|  4  | android.hidl.*       | system/libhidl/transport/*       |

所生成的 C++ 代码也位于 out/ 目录相似的位置：
* out/soong/.intermediates/frameworks/hardware/interfaces/
* out/soong/.intermediates/hardware/interfaces/
* out/soong/.intermediates/system/hardware/interfaces/
* out/soong/.intermediates/system/libhidl/transport/

# 图形系统在 HIDL 层中的设计模式

图形系统做为 Android 里最重要的模块，其中 HIDL 层的设计也是该项目的经典。其中从 HAL Interface 到 HIDL Interface，大致分为如下层次：
1. HIDL Interface
2. HIDL Implement
3. passthrough Interface
4. passthrough Implement
5. HAL Interface

如果底层 HAL 驱动提供的是老版本的接口，下面还有两层：
6. Adapter Implement
7. old HAL Interface

如此复杂的层次，就是为了实现其设计目标：框架可以在无需重新构建 HAL 的情况下进行替换。为了这个目标，HIDL 层中的设计采用了多种设计模式：
* 代理模式(Proxy Pattern)或委托模式(Delegate Pattern)。
* 装饰者模式(Decorator Pattern)或适配器模式(Adapter Pattern)。
* 访问者模式(Visitor Pattern)。

注：在本文中，代理模式和委托模式，装饰者模式和适配器模式，没有多少本质区别。

## 设计模式回顾

开发经验表明，在开发中，少用继承，多用组合。在结构型模式中组合模式、装饰者模式、代理模式都能很好避免真实对象或者更好的遵循最小知识原则。代理模式或者委托模式是代码隔离非常有效的设计模式。

* 代理模式(Proxy Pattern)，为其他对象提供一个代理，并由代理对象控制原有对象的引用。也称为委托模式(Delegate Pattern)。
* 装饰者模式(Decorator Pattern)就是将某个类重新装扮一下，使得它比原来更“漂亮”，或者在功能上更强大，这就是装饰器模式所要达到的目的。
* 适配器模式(Adapter Pattern)就是把一个类的接口变换成客户端所能接受的另一种接口，从而使两个接口不匹配而无法在一起工作的两个类能够在一起工作。
* 装饰者和适配器模式都是包装模式(Wrapper Pattern)，装饰者模式也是一种特殊的代理模式。

访问者模式(Visitor Pattern)是一种将数据操作和数据结构分离的设计模式，是一种很复杂的设计模式。访问者模式通过使用访问者类，改变元素类的执行算法。它主要解决的问题是稳定的数据结构和易变的操作耦合问题。它的实现方式是在数据基础类里面有一个方法接受访问者，将自身引用传入访问者。

访问者模式的使用场景：
* 对象结构比较稳定，但经常需要在此对象结构上定义新的操作。
* 需要对一个对象结构中的对象进行很多不同的并且不相关的操作，而需要避免这些操作“污染”这些对象的类，也不希望在增加新操作时修改这些类。

![访问者模式的UML类图](https://upload-images.jianshu.io/upload_images/7345261-f1b6fe6189c026da.png?imageMogr2/auto-orient/strip|imageView2/2/w/840/format/webp)

角色介绍：

* Visitor：接口或者抽象类，定义了对每个 Element 访问的行为，它的参数就是被访问的元素，它的方法个数理论上与元素的个数是一样的，因此，访问者模式要求元素的类型要稳定，如果经常添加、移除元素类，必然会导致频繁地修改 Visitor 接口，如果出现这种情况，则说明不适合使用访问者模式。
* ConcreteVisitor：具体的访问者，它需要给出对每一个元素类访问时所产生的具体行为。
* Element：元素接口或者抽象类，它定义了一个接受访问者(accept)的方法，其意义是指每一个元素都要可以被访问者访问。
* ElementA、ElementB：具体的元素类，它提供接受访问的具体实现，而这个具体的实现，通常情况下是使用访问者提供的访问该元素类的方法。
* ObjectStructure：定义当中所提到的对象结构，对象结构是一个抽象表述，它内部管理了元素集合，并且可以迭代这些元素提供访问者访问。

## 设计模式应用

图形系统中的 HAL 接口版本，基本上是采用扩展方式升级，原有的方法保持不变。因此 HIDL 接口，就是 HAL 接口多版本组合的集合。所以 HIDL 接口对于 HAL 接口就是一种包装模式，具体的就是代理模式，或者装饰者模式。

在系统初始化时，HIDL 接口动态装载 HAL plugin，确定 HAL 接口版本。如果 HAL 接口是老版本而上层应用需要新版本的方法，则 HIDL 接口需要适配器模式。

HIDL 层的总体设计，就是一种访问者模式。具体可以查看后面的各类的组合关系图。

# 图形系统在 HIDL 层的模块划分

图形系统在 HIDL 层的代码主要位于 hardware/interfaces/graphics 目录。设计上还是贯彻 Producer-Consumer 模式的设计思路，模块分为：
* Producer 端：allocator/ & mapper/
* Queue ：bufferqueue/
* Consumer 端：composer/
* 通用数据结构：common/

另外，和图形系统相关的还有如下接口，主要是和 vr 与 power 相关：
* frameworks/hardware/interfaces/bufferhub
* frameworks/hardware/interfaces/displayservice
* frameworks/hardware/interfaces/power
* frameworks/hardware/interfaces/vr

下表是 graphics 在 HIDL 层的接口的版本：

| No. | Interface   | Versions        |
|:---:|-------------|-----------------|
|  1  | allocator   | 2.0 / 3.0       |
|  2  | bufferqueue | 1.0 / 2.0       |
|  3  | common      | 1.0 / 1.1 / 1.2 |
|  4  | composer    | 2.1 / 2.2 / 2.3 |
|  5  | mapper      | 2.0 / 2.1 / 3.0 |

下表是 graphics 在 HAL 层的接口的版本：

| No. | Interface  | Versions |
|:---:|------------|----------|
|  1  | gralloc    | v0 / v1  |
|  2  | hwcomposer | v1 / v2  |

注：hwcomposer v0 版本就是 EGL / OpenGLES。

这里有一个有趣的问题，HIDL 是为了弥补 HAL 版本碎片而出现。本来 HAL 版本就多，而随着 Android 版本的进化，HIDL 层的接口的版本也多起来，甚至比 HAL 层的版本都要多。

因为高版本接口要继承低版本，要有兼容性，所以是交叉映射。最终从类的协作图看，HIDL 的各版本接口的关系就是一个复杂的网状结构，难以理解，难以管理。于是在 3.0 版本的时候，干脆不使用继承方式定义功能，而是把前面所有版本的虚函数都复制到新的接口里，减少了继承关系，就减少了复杂度。

# Producer 端的 HIDL 接口

前面提到过，gralloc 模块有 6 个基本功能：
* 本地申请 alloc() / 释放 free() 缓冲区。
* 远程申请 retain() / 释放 release() 缓冲区。
* 锁定 lock() / 解锁 unlock() 缓冲区。

这几个功能调用的频率不一样，于是在实现 GraphicBuffer 时，将上述功能划分到含义更明确的两个类中：
* GraphicBufferAllocator : alloc() / free() & retain() / release()
* GraphicBufferMapper : lock() / unlock()

该层接口的代码位于：
* frameworks/native/include/ui/

更进一步的，设计 HIDL 接口时，就有了对应的两个接口：
* IAllocator
* IMapper

该层接口的代码分别位于：
* hardware/interfaces/graphics/allocator/
* hardware/interfaces/graphics/mapper/

于是，GraphicBufferAllocator / GraphicBufferMapper 不再直接调用 gralloc 模块，而是调用 IAllocator / IMapper 接口。这样不但解决了版本兼容性问题，还使得 GraphicBuffer 具有了跨进程访问的能力。

下图是 IAllocator 2.0 的关系组合图：

![IAllocator 2.0 的关系组合图]('allocator 2.0 IAllocator Component Class Diagram.svg')

下图是 IMapper 2.0 的关系组合图：

![IMapper 2.0 的关系组合图]('mapper 2.0 IMapper Component Class Diagram.svg')

下图是 IMapper 2.1 的关系组合图：

![IMapper 2.1 的关系组合图]('mapper 2.1 IMapper Component Class Diagram.svg')

# Queue 的 HIDL 接口

因为 Queue 的操作和硬件无关，所以这里没有涉及 HAL 模块。在这里的接口主要是转换接口。原本在 frameworks/native/include/gui/ 目录就定义有跨进程的接口：
* IGraphicBufferProducer
* IProducerListener

在 HIDL 层在 hardware/interfaces/graphics/bufferqueue/ 目录又再次定义类似接口：
* IGraphicBufferProducer
* IProducerListener

可以认为在 HIDL 层定义了虚拟硬件，而在 framework 层根据硬件做了实现。所以这两层接口需要做转换。在实现类的名字前缀中有如下规则：
* B : Binder / Framework Interface
* H : Hybrid / HIDL Interface
* B2H : from Framework Interface to HIDL Interface, realize H & required B, from up to bottom.
* H2B : from HIDL Interface to Framework Interface, realize B & required H, from bottom to up. 

关系组合图

# Consumer 端的 HIDL 接口

在 framework 接口中，为 Composer 设计了 3 个跨进程接口：
* ISurfaceComposer。对 SurfaceFlinger 的代理。接口中的方法很多，这里主要关注下面几个：
  + createConnection()：为客户端提供访问 Composer 的接口：ISurfaceComposerClient。
  + createDisplayEventConnection()：为客户端提供访问 VSYNC 的接口：IDisplayEventConnection。
* ISurfaceComposerClient：客户端访问 Composer 的接口。接口中的方法很多，这里主要关注下面几个：
  * createSurface()
* IDisplayEventConnection : 客户端访问/管理 VSYNC 的接口。

该层接口的代码位于：
* frameworks/native/include/gui/

以上接口，最终都是使用  hwcomposer 模块。设计 HIDL 接口时，对 hwcomposer 模块封装，就有了对应的 3 个接口：
* IComposer。接口中的方法很多，这里主要关注下面几个：
  + createClient()：为客户端提供访问 Composer 的接口：IComposerClient。
* IComposerClient：客户端访问 Composer 的接口。接口中的方法很多，这里主要关注下面几个：
  + createLayer()
  + destroyLayer()
* IComposerCallback：hwcomposer 模块上报事件的封装。有 3 个事件：
  + onHotplug()
  + onRefresh()
  + onVsync()

该层接口的代码位于：
* hardware/interfaces/graphics/composer/

下图是 composer 2.1 的接口的关系组合图：
![composer 2.1 的接口的关系组合图]('composer 2.1 IComposer Component Class Diagram.svg')

在图中，IComposer / IComposerClient 接口的数据流是从上到下的方向流动。数据源是 Application。IComposerCallback 接口的数据流是从下到上的方向流动。数据源是 hwcomposer 模块。

下图是 composer 2.2 的接口的关系组合图：
![composer 2.2 的接口的关系组合图]('composer 2.2 IComposer Component Class Diagram.svg')

下图是 composer 2.3 的接口的关系组合图：
![composer 2.3 的接口的关系组合图]('composer 2.3 IComposer Component Class Diagram.svg')

从上两个图可以看出，随着 HIDL 版本的增加，类的关系如同网格一样展开，会越来越复杂。

# 通用数据结构的 HIDL 接口

主要是给上述 Producer-Consumer 模式的 3 个模块提供通用的数据结构。没有什么特别的地方。

# VTS 程序看到的 HIDL 接口的协作图

Android Vendor Test Suite (VTS)，厂商测试套件，能对 HAL 和 Kernel 做自动测试，遍历每个功能。从 VTS 程序的角度，可以看到完整的 HIDL 中的接口的关系组合。

下图是 VTS 程序的对 composer 2.1 接口测试所得到的关系组合图：
![composer 2.1 的接口的关系组合图 - VTS]('composer 2.1 ComposerVts Component Class Diagram.svg')

下图是 VTS 程序的对 composer 2.2 接口测试所得到的关系组合图：
![composer 2.2 的接口的关系组合图 - VTS]('composer 2.2 ComposerVts Component Class Diagram.svg')

下图是 VTS 程序的对 composer 2.3 接口测试所得到的关系组合图：
![composer 2.3 的接口的关系组合图 - VTS]('composer 2.3 ComposerVts Component Class Diagram.svg')

从这些关系图可以看出：
* 高版本的 HIDL 接口可以充当低版本使用。
* 低版本的 HAL 接口也已经适配到高版本的 HIDL 接口，提供上层程序使用。在实际代码中，HAL plugin 是在初始化时动态加载。
* 这种设计使得系统具有无缝升级的能力。但是接口与类的关系，就如同渔网一样，越来越复杂。
* 这么复杂的关系，只有从设计模式角度出发才好设计和理解。这里的 HIDL 接口与实现就是一种访问者模式。

# 参考文件

1. [HIDL | Android Open Source Project](https://source.android.com/devices/architecture/hidl)
1. [Here comes Treble: A modular base for Android](https://android-developers.googleblog.com/2017/05/here-comes-treble-modular-base-for.html)
1. [Android's HIDL: Treble in the HAL](https://www.slideshare.net/opersys/androids-hidl-treble-in-the-hal)
1. [Android Treble: Blessing or Trouble?](https://www.slideshare.net/opersys/android-treble-blessing-or-trouble)
1. [Project Treble. What Makes Android 8 different?](https://events19.linuxfoundation.org/wp-content/uploads/2017/11/Project-Treble.-What-Makes-Android-8-Different_-Fedor-Tcymbal-Mera-Software-Services.pdf)
1. [ANDROID HIDL AND PROJECT TREBLE](https://devarea.com/android-hidl-and-project-treble/#.Xnq_f3UzbeQ)
1. [Android Treble 简介](https://blog.csdn.net/jingerppp/article/details/86513675)
1. [Android HIDL 详解](https://blog.csdn.net/jingerppp/article/details/86514997)
1. [访问者模式一篇就够了](https://www.jianshu.com/p/1f1049d0a0f4)

