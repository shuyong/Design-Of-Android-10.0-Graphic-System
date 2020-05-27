# Android 的图形系统的 Framework 接口与约定
* * *

# Framework 代码分布

与图形相关的 Framework 代码主要分布以下目录：
* frameworks/native/include/ui/
* frameworks/native/libs/ui/
* frameworks/native/include/gui/
* frameworks/native/libs/gui/

各目录所要实现的功能如下：
* "ui/" 目录主要是 ANativeWindowBuffer 接口的实现：GraphicBuffer
* "gui/" 目录主要是 ANativeWindow 接口的实现：Surface
  + 相对应的是 ANativeWindow 接口背后的 Queue 的实现：BufferQueue
  + 以及在 BufferQueue 两端跨进程的、Producer-Consumer / Listener 模型的接口：
    - IConsumerListener
    - IGraphicBufferConsumer
    - IGraphicBufferProducer
    - IProducerListener
  + 以及在 Queue 中传输的 GraphicBuffer 的封装：BufferItem / BufferSlot；
  + 对于如何消费 GraphicBuffer，Producer 对 Consumer 的实现的约定接口：
    - ConsumerBase
    - CpuConsumer
    - GLConsumer
  + 还有 Producer 期待 SurfaceFlinger 暴露的接口：
    - IDisplayEventConnection
    - ISurfaceComposerClient
    - ISurfaceComposer

ISurfaceComposer / ISurfaceComposerClient，声明位于这里：
* frameworks/native/include/gui/ISurfaceComposer.h
* frameworks/native/include/gui/ISurfaceComposerClient.h

在 surfaceflinger service 中的重要实现类 Layer / SurfaceFlinger，声明位于这里：
* frameworks/native/services/surfaceflinger/Layer.h
* frameworks/native/services/surfaceflinger/SurfaceFlinger.h

# [Buffer 与 Window 的设计](buffer-window-design.md)

# [BufferQueue 的设计](bufferqueue-design.md)

# Composer 的设计

在 ui / gui 目录里的接口，对应的是 Producer 和 Queue 段的内容，相应的层次对应 Consumer 段的内容，在 SurfaceFlinger 里底层的 DisplayHardware 模块里：
* Composer：对应 HAL 中的 hwcomposer 接口。
* Power：对应 HAL 中的 power 接口。



# 小节

Android HAL 中和图形系统有关的，有这几方面的知识：
* 面向 Application，用 GraphicBuffer 类实现 ANativeWindowBuffer 接口，用 Surface 类实现 ANativeWindow 接口。EGL / OpenGLES 模块是 ANativeWindowBuffer 接口和 ANativeWindow 接口最主要的使用者。
* 面向 Graphic Server，用 Composer 实现合成器设备(hwc_composer_device_1_t)接口，用 FrameBufferSurface 类管理帧缓冲(Framebuffer)设备。最重要的，硬件合成器(HWC)是整个显示子系统的硬件抽象，包括显示设备，是所有 Android 合成操作的核心。

# 参考文件
1. [The Android Graphics microconference](https://lwn.net/Articles/569704/)
1. [Graphics](https://source.android.com/devices/graphics/index.html)
1. [BufferQueue and gralloc](https://source.android.com/devices/graphics/arch-bq-gralloc)
1. [Android's Graphics Buffer Management System (Part II: BufferQueue)](https://www.codeproject.com/Articles/990983/Androids-Graphics-Buffer-Management-System-Part-II)
1. [Implementing the Hardware Composer HAL](https://source.android.com/devices/graphics/implement-hwc)

