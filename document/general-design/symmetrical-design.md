# 基于接口(interface)的对称设计
* * *

从 HAL 到 Framework，Android 系统就是围绕着接口(interface)进行设计与实现。接口(interface)如同工业管线里的管道阀门，控制着数据的流量和流向。理解了接口(interface)的功能和作用，基本上就理解了系统的运作模式。

# 基于接口(interface)的设计

## 跨越进程/线程的接口(Interface)

相关接口(interface)的定义，位于"frameworks/native/include/gui/"目录。前面已经做过说明。这里再次回顾以下，因为 SurfaceFlinger 都有应用到这些接口：
* IConsumerListener
  - onFrameAvailable()
  - onFrameReplaced()
  - onBuffersReleased()
  - onSidebandStreamChanged()
* IDisplayEventConnection
  - getDataChannel()
  - setVsyncRate()
  - requestNextVsync()
* IGraphicBufferConsumer
  - acquireBuffer()
  - releaseBuffer()
* IGraphicBufferProducer
  - dequeueBuffer()
  - queueBuffer()
* IProducerListener
  - onBufferReleased()
* ISurfaceComposerClient
  - createSurface()
  - destroySurface()
* ISurfaceComposer
  - createConnection()
  - createDisplayEventConnection()
  - createDisplay()
  - destroyDisplay()

| No. | Interface               | Realized                | Usage                  |
|:---:|-------------------------|-------------------------|------------------------|
| 1   | IConsumerListener       | ProxyConsumerListener   | BufferQueueProducer    |
| 2   | IDisplayEventConnection | EventThread::Connection | DisplayEventReceiver   |
| 3   | IGraphicBufferConsumer  | BufferQueueConsumer     | SurfaceFlingerConsumer |
| 4   | IGraphicBufferProducer  | BufferQueueProducer     | Surface                |
| 5   | IProducerListener       | DummyProducerListener   |                        |
| 6   | ISurfaceComposerClient  | Client                  | SurfaceFlinger         |
| 7   | ISurfaceComposer        | SurfaceFlinger          | SurfaceFlinger         |

## BufferQueue 消费端相关的接口(interface)
* ConsumerBase
  - SubClass ConsumerListener
  - Create ProxyConsumerListener
  - Required IGraphicBufferConsumer
  - Required FrameAvailableListener
* ConsumerListener
  - onFrameAvailable()
  - onFrameReplaced()
  - onBuffersReleased()
  - onSidebandStreamChanged()
* ContentsChangedListener
  - SubClass FrameAvailableListener
  - onSidebandStreamChanged()
* FrameAvailableListener
  - onFrameAvailable()
  - onFrameReplaced()

| No. | Interface               | Realized                | Usage                  |
|:---:|-------------------------|-------------------------|------------------------|
| 1   | ConsumerBase            | SurfaceFlingerConsumer  | SurfaceFlinger         |
| 2   | ConsumerListener        | ConsumerBase            | IConsumerListener      |
| 3   | ContentsChangedListener | Layer                   | ConsumerListener       |
| 4   | FrameAvailableListener  | ContentsChangedListener | ConsumerListener       |
| 5   | IConsumerListener       | ProxyConsumerListener   | BufferQueueProducer    |
| 6   | IGraphicBufferConsumer  | BufferQueueConsumer     | SurfaceFlingerConsumer |
| 7   | IGraphicBufferProducer  | BufferQueueProducer     | Surface                |
| 8   | IProducerListener       | DummyProducerListener   |                        |

## 与 Display 相关的接口(interface)
* DisplaySurface
  - beginFrame()
  - prepareFrame()
  - compositionComplete()
  - advanceFrame()
  - onFrameCommitted()
* RenderEngine
* hwc_composer_device_1 (v1)
  - prepare()
  - set()
* hwc_procs (v1)
  - invalidate()
  - vsync()
  - hotplug()
* hwc2_device_t (v2)
  - validateDisplay()
  - presentDisplay()

| No. | Interface             | Realized           | Usage          |
|:---:|-----------------------|--------------------|----------------|
| 1   | DisplaySurface        | FramebufferSurface | SurfaceFlinger |
| 2   | RenderEngine          | GLES20RenderEngine | SurfaceFlinger |
| 3   | hwc_composer_device_1 | HAL plugin         | HWComposer     |
| 4   | hwc_procs             | HWComposer         | SurfaceFlinger |
| 5   | hwc2_device_t         | HAL plugin         | HWComposer     |

# 对称设计

## 基于 Producer-Consumer 模型的对称设计

BufferQueue 是 Android 图形流水线的中心。上面所列的各个接口都围绕 BufferQueue 来设计。它的设计是对称的。在前面分析 BufferQueue 的章节已经有过说明，这里回顾一下。

下图是以 BufferQueue 为中心的接口和类的协作图。

![BufferQueue Component Diagram](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/framework-design/gui_BufferQueue%20Component%20Diagram.svg)

* BufferQueue 的设计是对称的。
  - Producer-Consumer 设计模式对应的是2个跨进程接口(interface)：IGraphicBufferProducer & IGraphicBufferConsumer。
  - Listener 设计模式对应的是2个跨进程接口(interface)：IProducerListener & IConsumerListener。
* 为 Producer-Consumer 设计模式所设计的类是对称的。
  - 在客户端的接口是 ANativeWindow，对应的服务端的接口是 Layer。
  - 在客户端的实现类是 Surface，对应的服务端的实现类是 BufferQueueLayer。
  - 也就是，一对 Surface-Layer，就是一对 Producer-Consumer 关系，中间存在一个 BufferQueue。
* 对 BufferQueue 的操控是对称的。
  - 在客户端是 Surface 调用 IGraphicBufferProducer 接口操控 BufferQueue。最常用的就是 dequeueBuffer() & queueBuffer() 这 2 个方法。
  - 在服务端是 SurfaceFlingerConsumer 侦听 IConsumerListener 接口的消息，然后调用 IGraphicBufferConsumer 接口操控 BufferQueue。最常用的就是 acquireBuffer() & releaseBuffer() 这 2 个方法。
  - 类 BufferLayerConsumer 由实现类 BufferQueueLayer 所创建。基于 C/S 模型，一个 Surface 对应一个 BufferLayerConsumer。

## 基于 Client / Server 模型的对称设计

从 UI 程序的角度看，最先需要理解的是 Client / Server 程序之间的跨进程接口(Interface)的数据流动方向。Android 图形系统，Client / Server 程序之间其实是互为 Producer-Consumer 的模式。具体见下面的类协作图：

![ISurfaceComposer Component Diagram](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/framework-design/gui_ISurfaceComposer%20Component%20Diagram.svg)

图的上半部分，是 Surface 的 Client / Server 协作图，涉及 3 个跨进程接口(Interface)：
* IGraphicBufferProducer
* ISurfaceComposerClient
* ISurfaceComposer

这其实就是内容(Content)的 Producer-Consumer 的传输路径，从 Surface 生产，经过 IGraphicBufferProducer 到达 Layer，最后到达 SurfaceFlinger 进行消费。

图的下半部分，是 VSYNC 的 Client / Server 协作图，涉及 2 个跨进程接口(Interface)：
* ISurfaceComposer
* IDisplayEventConnection

这其实就是 VSYNC 信号的 Producer-Consumer 的传输路径，从 SurfaceFlinger 生产，经过 IDisplayEventConnection 到达 Application 进行消费。

这张图还体现了 Client / Server 两端类的对称设计。同时各层级类的生成和调用关系也是对称的。

| No. |   Client's Class      | Interface               | Server's Class |
|:---:|-----------------------|-------------------------|----------------|
| 1   | Surface               | IGraphicBufferProducer  | Layer          |
| 2   | SurfaceComposerClient | ISurfaceComposerClient  | Client         |
| 3   | ComposerService       | ISurfaceComposer        | SurfaceFlinger |
| 4   | DisplayEventReceiver  | IDisplayEventConnection | EventThread    |

# 参考文件
1. [BufferQueue and gralloc](https://source.android.com/devices/graphics/arch-bq-gralloc)
1. [Android's Graphics Buffer Management System (Part II: BufferQueue)](https://www.codeproject.com/Articles/990983/Androids-Graphics-Buffer-Management-System-Part-II)


