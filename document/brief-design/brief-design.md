# Android 的图形系统的简略设计
* * *

# 契约式编程

Android 系统十分复杂。为了使得整个系统在进化过程中可控，保持相同的功能和相同的接口，所以 Android 系统是基于程序接口(interface)进行设计。

Android 系统接口(interface)的设计，具有分层、网格化(mesh)、对称性的特点。类似于在一张纸上打上网格，节点就是各种接口(interface)，节点之间就是各种实现(implementation)。通过接口(interface)做为锚点(anchor)固定住系统功能，这样整个系统功能实现才会如设计时所想，才会保持兼容性，不会出现偏差。

本文主要关注 Android 的图形系统，并且只是 C/C++ 层面的代码。

# Android 的图形系统的层次划分

在 Android 的源代码中，C/C++ 层面的代码主要分布在下面 4 个目录中：
* frameworks/native/include/
* frameworks/native/libs/
* hardware/libhardware/
* external/

分别对应如下层次：

| No. | Directory                  | Level                    | Description                           |
|:---:|----------------------------|--------------------------|---------------------------------------|
| 1   | frameworks/native/include/ | Framework Interface      | Application Required Interface        |
| 2   | frameworks/native/libs/    | Framework Implementation | Implementation of Framework Interface |
| 3   | hardware/libhardware/      | HAL Interface            | OS / Vendor Provided Interface        |
| 4   | external/                  | Linux Support Library    | Low Level Library in Linux World      |

后来，因为要提供新的底层功能，HAL Interface 也要进化。而厂商(vendor)基于自身的考虑不愿意跟进，不愿实现新的 HAL Implementaion。于是市场上的 Android 系统就出现了碎片化。Google 再推广新版本的 Android 系统是就遇到了困难。于是他们就开发了新的一层代码，以便在新版本的 Android 系统之下能覆盖住不同版本的 HAL Interface。那个项目就是 [Project Treble](https://source.android.com/devices/architecture/treble)。新一层的代码就叫 HAL Interface Definition Language (HIDL)。其实 HIDL 就是在 Framework / HAL 两层之间的一个适配层，并且使得接口和实现可以跨进程分布。HIDL 在 Android 8.0 版本之后引入。

HIDL 本身的实现代码和依赖代码位于：
* system/libhidl
* system/libhwbinder

HIDL 所定义的接口(interface)命名为 "xxx.hal"，位于：

| No. | Prefix of Package    | Directory                        |
|:---:|----------------------|----------------------------------|
|  1  | android.frameworks.* | frameworks/hardware/interfaces/* |
|  2  | android.hardware.*   | hardware/interfaces/*            |
|  3  | android.system.*     | system/hardware/interfaces/*     |
|  4  | android.hidl.*       | system/libhidl/transport/*       |

最后所生成的 C/C++ 代码位于 out/soong/.intermediates/ 目录中。

如果要将 HIDL 放入前一个表格划分的层次中，则为：

| No. | Directory                  | Level                    | Description                           |
|:---:|----------------------------|--------------------------|---------------------------------------|
| 1   | frameworks/native/include/ | Framework Interface      | Application Required Interface        |
| 2   | frameworks/native/libs/    | Framework Implementation | Implementation of Framework Interface |
| 3   | HIDL Directory             | Adaptor Interface        | Adaptor Interface for Framework       |
| 4   | hardware/libhardware/      | HAL Interface            | OS / Vendor Provided Interface        |
| 5   | external/                  | Linux Support Library    | Low Level Library in Linux World      |

# Android 的图形系统的模块划分

图形系统本质上就是一个 Producer-Consumer 模型。所以 Android 的图形系统就是按照 Producer-Consumer 模式进行设计。

Producer-Consumer 模式有 3 个成员：
* Producer
* Channel：在 Android 系统中称为 Queue。
* Consumer

此外，Producer-Consumer 模式往往伴生有 Observer 模式，在 Android 系统中称为 Listener，位于 Queue 的两端。

Android 的图形系统就是按照这 4 大模块进行横向划分。本章就从高度抽象的角度对此进行简略描述。

## Producer 端的接口(interface)

Producer 端的程序就是人们常说的图形应用程序。是开发人员最熟悉的地方。这里提供给应用程序的是两个最重要的接口(interface)：
* Window : ANativeWindow
* Buffer : ANativeWindowBuffer

一个 Window，下面必定包含 2~N 个 Buffer。这种假设是和 X11 系统不一样的地方。在 X11 中，一个 Window 只包含一个固定不变的 Buffer。ANativeWindow 最重要的用途就是为 EGL 模块提供 EGLSurface。

此外，ANativeWindowBuffer 在应用程序中也可以单独申请和使用，最重要的用途就是为 EGL 模块提供 EGLImage，从而绑定到 OpenGLES 模块中做为 Texture / FBO 使用。

### Buffer 的应用与实现

Buffer 的实现和硬件相关，需要 HAL 提供的 Graphic Alloc (gralloc) 模块，v0 版本。该模块提供如下接口(interface)：
* gralloc_module_t
* alloc_device_t

Buffer 的应用与实现层次如下：

| No. | Level                    | Interface / Class                 |
|:---:|--------------------------|-----------------------------------|
|  0  | Application              | EGLImage                          |
|  1  | Framework Interface      | ANativeWindowBuffer               |
|  2  | Framework Implementation | GraphicBuffer                     |
|  3  | Framework Internal       | GraphicBuffer Mapper / Allocator  |
|  4  | HAL Interface            | gralloc_module_t / alloc_device_t |
|  5  | HAL plugin               | Vendor's Implementation           |

在 Android 7.0 之后，gralloc 模块升级，v1 版本，提供新接口(interface)：
* gralloc_module_t
* gralloc1_device_t

在 Android 8.0 之后，HIDL 层的代码就是使得 framework 层中的实现代码能够无缝地使用不同版本的 gralloc 模块。

于是新的 Buffer 的应用与实现层次如下：

| No. | Level                    | Interface / Class                          |
|:---:|--------------------------|--------------------------------------------|
|  0  | Application              | EGLImage                                   |
|  1  | Framework Interface      | ANativeWindowBuffer                        |
|  2  | Framework Implementation | GraphicBuffer                              |
|  3  | Framework Internal       | GraphicBuffer Allocator / Mapper           |
|  4  | HIDL Interface           | IAllocator(2.0/3.0) / IMapper(2.0/2.1/3.0) |
|  5  | HIDL Implementation      | Adapt (2.0/2.1/3.0) to (v0 / v1)           |
|  6  | HAL Interface            | gralloc (v0 / v1)                          |
|  7  | HAL plugin               | Vendor's Implementation                    |

### Window 的应用与实现

Window 本质上就是 Buffer 的管理代码，所以和硬件无关，因此不需要 HAL 层的支持。在 Android 图形系统中，Window 就是对 Buffer Queue 的应用。

Window 的应用与实现层次如下：

| No. | Level                    | Interface / Class       |
|:---:|--------------------------|-------------------------|
|  0  | Application              | EGLSurface              |
|  1  | Framework Interface      | ANativeWindow           |
|  2  | Framework Implementation | Surface                 |
|  3  | Framework Internal       |                         |
|  4  | Binder Interface         | IGraphicBufferProducer  |
|  5  | HIDL Interface           | bufferqueue (1.0 / 2.0) |

## Queue 中的接口(interface)

在 Queue 中管理的是 Buffer Pool。对于 Producer 端提交过来的 Buffer，因为包含内容(content)，所以称之为帧(frame)。对于 Consumer 端消费完的，等待 Producer 端使用的则为 Free Buffer，简称 Buffer。

因为需求很复杂，Android 图形系统实现了一个很复杂的 Queue。分为工厂类 BufferQueue，和实际工作的类 BufferQueueCore。后面都简称为 BufferQueue。
* BufferQueue 对于 Producer 端提供的是：queue() 和 dequeue() 方法。
* BufferQueue 对于 Consumer 端提供的是：acquireBuffer() 和 releaseBuffer() 方法。

## Consumer 端的接口(interface)

Producer 端的程序就是著名的 surfaceflinger software。surfaceflinger(sw) 的实现和硬件相关，需要 HAL 提供的 HWComposer 模块，v1 版本。v0 版本为 EGL/OpenGLES。后来该模块升级为 v2 版本。各自提供的接口(interface)为：
* v1 : hwc_composer_device_1_t
* v2 : hwc2_device

Composer 的应用与实现层次如下：

| No. | Level               | Interface / Class                |
|:---:|---------------------|----------------------------------|
|  0  | Application         | SurfaceFlinger                   |
|  1  | Binder Interface    | ISurfaceComposer                 |
|  2  | HIDL Interface      | IComposer (2.1/2.2/2.3)          |
|  3  | HIDL Implementation | Adapt (2.1/2.2/2.3) to (v1 / v2) |
|  4  | HAL Interface       | HWComposer (1.0 / 2.0)           |
|  5  | HAL plugin          | Vendor's Implementation          |

此外，基于 BufferQueue，Producer 端的 Surface，在 Consumer 端有一个对称的接口：Layer。
* Surface 使用 BufferQueue 的 queue() 和 dequeue() 方法。
* Layer 使用 BufferQueue 的 acquireBuffer() 和 releaseBuffer() 方法。

## Listener 的接口(interface)

Producer-Consumer 模式往往伴生有 Listener 模式。在 BufferQueue 的两端分别提供有 2 个接口(interface)：
* IProducerListener
* IConsumerListener

当程序在 Buffer 中渲染完成，在使用 BufferQueue::queue() 将包含内容(content)的帧(frame)入队后，需要主动调用 IConsumerListener 去通知 Consumer 端的程序。这样，Consumer 端的程序就可以及时消费内容。

虽然是对称设计，但是 IProducerListener 并没有被实际使用。因为 Consumer 端消费内容(content)的含义很微妙。为了更快的并发操作，并不是 Consumer 端的程序将 Buffer 通过 BufferQueue::releaseBuffer() 放回队列，就意味着 Consumer 端的程序已经将内容(content)复制完成。并不是 Consumer 端的程序复制完内容(content)，就意味着内容(content)已经被消费。只有当内容(content)被显示到屏幕上时，才是真正的已被消费。这是一个异步过程，因此 Consumer 端的程序无法在 releaseBuffer() 时刻，使用 IProducerListener 通知 Producer 端的程序，内容已被消费。

这个问题的解决方法是使用显式同步机制 Fence。在 Android 7.0 之后，新增的编舞者(Choreographer)机制还提供更精确的渲染节拍信号，供感兴趣的应用程序使用。

在 Android 8.0 之后，IProducerListener 也由 HIDL 中的 bufferqueue 接口提供实现。

# 接口(interface)汇总表

以上所涉及到的接口(interface)与实现分别位于如下位置：

| Description            | Interface / Class   | Files                                                             |
|:----------------------:|---------------------|-------------------------------------------------------------------|
| Window Interface       | ANativeWindow       | frameworks/native/libs/nativewindow/include/nativewindow/window.h |
| Buffer Interface       | ANativeWindowBuffer | frameworks/native/libs/nativebase/include/nativebase/nativebase.h |
| Window Implementation  | Surface             | frameworks/native/include/ui/Surface*.h                           |
| Buffer Implementation  | GraphicBuffer       | frameworks/native/include/ui/GraphicBuffer*.h                     |
| Queue Implementation   | BufferQueue         | frameworks/native/include/gui/BufferQueue*.h                      |
| Producer Listener      | IProducerListener   | frameworks/native/include/gui/IProducerListener.h                 |
| Consumer Listener      | IConsumerListener   | frameworks/native/include/gui/IConsumerListener.h                 |
| Composer Interface     | ISurfaceComposer    | frameworks/native/include/gui/ISurfaceComposer*.h                 |
| Binder Interface       |                     | frameworks/native/include/gui/I*.h                                |
| HIDL Interface         |                     | hardware/interfaces/graphics/                                     |
| HAL Buffer             | gralloc             | hardware/libhardware/include/hardware/gralloc*.h                  |
| HAL Composer           | hwcompose           | hardware/libhardware/include/hardware/hwcomposer*.h               |

主要类图的层次分布示意图：

内容(content)在接口(interface)中的横向流动示意：

| Level                    | Producer      | Content              | Queue              | Consumer           | Consumer           |
|--------------------------|---------------|----------------------|--------------------|--------------------|--------------------|
| Application              | EGLSurface    | EGLImage             | surfaceflinger(sw) | surfaceflinger(sw) | surfaceflinger(sw) |
| Framework Interface      | ANativeWindow | ANativeWindowBuffer  |                    |                    | ISurfaceComposer   |
| Framework Implementation | Surface       | GraphicBuffer        | BufferQueue        | Layer              | SurfaceFlinger     |
| HIDL Interface           | bufferqueue   | IAllocator / IMapper |                    |                    | IComposer          |
| HAL Interface            |               | gralloc              |                    |                    | HWComposer         |

以上表格只是横向流动的简化示意。只是为了理解概念使用。实际的 Interface / Class 的分类和数据流动，比这个复杂很多。这些将在后面的章节讨论。

# 图形软件的工作流程与假设

前面提到过，在 Android 图形系统中，一个 Window，下面必定包含 2~N 个 Buffer。这种假设是和 X11 系统不一样的地方。在 X11 中，一个 Window 只包含一个固定不变的 Buffer。这是由于 Android 图形系统一开始设计时的假设不一样。

Android 图形系统，是针对多核、异构的硬件进行设计的。
* CPU 有多核，所以可以设计多线程/进程协作架构。
* 能够绘图的设备有 CPU / GPU / codec / camera 等。
* 能够合成(Composition)图像的设备有 CPU / GPU / HWComposer。
* 需要考虑并发设计，以便使得挂在从绘图到显示这个路径上的各个设备都可以满负荷工作。减少设备忙等待的时间。

因此，在这样的设计中，BufferQueue 是显示路径的中心。BufferQueue 中至少包含 2 个 Buffer，使得在同一时段，有一个 Back Buffer 提供给 Producer 端，为下一帧做渲染工作，有一个 Front Buffer 提供给 Consumer 端，为当前屏幕提供显示内容(content)。这样，不论 Producer 端还是 Consumer 端，都不会因为对方忙而没有工作介质 Buffer 而陷入等待。

从普通的图形程序的工作流程看：
1. 申请一个 Surface，做为 ANativeWindow 提供给 EGL 模块，生成渲染用的底图 EGLSurface。
2. 可能会单独申请若干个 GraphicBuffer，做为 ANativeWindowBuffer 提供给 EGL 模块，生成 EGLImage，以便提供给 OpenGLES 模块绑定为 Texture / Framebuffer。
3. 使用 OpenGLES 模块进行绘图。
4. 当一帧图像绘制完成，调用 eglSwapBuffers() 函数提交显示内容。
5. 重复 3~4 步直至程序结束。

本文不讨论第 3 步的内容，主要集中分析第 4 步所涉及到的 Interface / Class 的设计。
当第 4 步发生时，Producer-Consumer 模型可能执行如下操作：

| No. | Producer                                           | Queue                                  | Consumer                                |
|:---:|----------------------------------------------------|----------------------------------------|-----------------------------------------|
|  1  | Surface call BufferQueue::queue()                  | -> insert to BufferQueue::mQueue       |                                         |
|  2  | Surface call IProducerListener::onFrameAvailable() |                                        | Layer::onFrameAvailable()               |
|  3  |                                                    | get from BufferQueue::mQueue        -> | IGraphicBufferConsumer::acquireBuffer() |
|  4  | Surface call BufferQueue::dequeue()                | <- get from BufferQueue::mFreeBuffers  |                                         |
|  6  |                                                    |                                        | GLConsumer::updateTexImage()            |
|  5  |                                                    | insert to BufferQueue::mFreeBuffers <- | IGraphicBufferConsumer::releaseBuffer() |

值得注意的是：上述的操作因为是并发操作，执行循序有可能被打乱。

