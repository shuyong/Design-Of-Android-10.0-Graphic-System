# 责任(Responsibility)与关系(Relationship)

在 Android 图形系统中，主要就是 Producer-Consumer 模式。体现 Producer-Consumer 模式的类就是 BufferQueue。而大多数接口(Interface)，基本上就是位于生产端/消费端这两端，定义着各种抽象功能，从而定义了数据的流向。

同时，Android 图形系统根据功能域的划分，将整个图形处理的流水线划分为多个阶段(stage)。每个阶段的核心，都是 BufferQueue，而两端就是接口(Interface)的实现。这就体现了在不同阶段，对图像数据的不同处理。通过分析 BufferQueue 消费端的 ConsumerBase 接口(Interface)的具体实现，也就是管道的终点(endpoint)，就可以知道该 BufferQueue 所处的阶段。

典型的，显示在 LCD/HDMI 上面的界面就是两阶段(2-stage)的 Producer-Consumer 模式。而虚拟显示，则是多个两阶段组合的 Producer-Consumer 模式。

而 ConsumerBase 接口(Interface)的具体实现有：

* Stage-1 Realized

| No. | Realized           | Usage                  |
|:---:|--------------------|------------------------|
| 1   | BufferItemConsumer | Camera Input Stream    |
| 2   | CpuConsumer        | ScreenCapture          |
| 3   | GLConsumer         | SurfaceFlingerConsumer |
| 4   | GrallocConsumer    | renderscript           |
| 5   | RingBufferConsumer | Camera ZSL Stream      |

* Stage-2 Realized

| No. | Realized              | Usage         |
|:---:|-----------------------|---------------|
| 1   | FramebufferSurface    | DisplayDevice |
| 2   | VirtualDisplaySurface | DisplayDevice |

本文主要讨论普通应用 UI 的显示流程，也就是第一阶段的 GLConsumer/SurfaceFlingerConsumer 这个端点，以及第二阶段的 FramebufferSurface 这个端点。

## Stage-1 两端的接口(Interface)

一般分析 BufferQueue 的应用，大多数就是分析图形流水线的第一阶段，就是内容(Content) 从 Surface 到达 SurfaceFlingerConsumer 这一个生产循环。这在前面已经详细分析过。这里不再重复，只贴出 BufferQueue 的协作图做对比：

![Buffer Queue - Client-Server](https://github.com/shuyong/Design-Of-Android-6.0-Graphic-System/blob/master/image/general-design/Buffer%20Queue%20Component%20Diagram%20-%20Client-Server.svg)

这阶段的 BufferQueue 的协作图，涉及到大多数图形相关的接口(Interface)：
* IGraphicBufferProducer
* IGraphicBufferConsumer
* IProducerListener
* IConsumerListener
* ANativeWindow
* ANativeWindowBuffer

相关的功能前面已经详细分析过，这里不再重复。这里需要再次说明一点，这一阶段的内容生产的节拍由 Application 来决定，也就是调用 eglSwapBuffers() 的频率。例如玩游戏时，键盘鼠标的输入，传感器的输入，网络上传来的队友和对手的消息，都将引发游戏引擎生产新的游戏画面，然后调用 eglSwapBuffers() 来刷新屏幕。游戏引擎对输入的反馈越灵敏越好。

## Stage-2 两端的接口(Interface)

图形流水线的第二阶段，就是内容(Content) 从 DisplayDevice 到达 DisplaySurface 这一个消费循环。

这阶段的 BufferQueue 的协作图，涉及到的接口(Interface)和前面差不多，只新增了2个接口：
* DisplaySurface
* hwc_composer_device_1

DisplaySurface 接口(Interface)的主要功能：
- beginFrame() : 在确定合成配置之前，在合成循环的开头调用 beginFrame。DisplaySurface 应该做任何它需要做的事情，以使 HWComposer 能够决定如何合成帧(frame)。我们传递进来的参数 mustRecompose，是为了让 VirtualDisplaySurface 的状态机保持快乐，而不必在没有任何变化的情况下去排队等待缓冲区。
- prepareFrame() : prepareFrame 在合成配置确定之后，但在合成发生之前调用。DisplaySurface 可以使用合成类型来决定如何管理关于此帧的在 GLES 和 HWC 之间的缓冲流。
- compositionComplete() : 应当在帧的合成渲染完成时调用（但不一定已经调用了 eglSwapBuffers）。某些旧驱动程序要求用此方法进行同步。如果不再支持 HWC 1.0 版本，此方法将被废弃。
- advanceFrame() : 通知 Surface，此帧的 GLES 合成已完成，Surface 应确保 HWComposer 具有此帧的正确缓冲区。有些实现可能只会在发生 GLES 合成时向 HWComposer 推送一个新的缓冲区，而其它实现则是在每一帧上推送一个新的缓冲区。调用 advanceFrame 之后必须跟着调用一次 onFrameCommitted，因为 advanceFrame 可能会再次被调用到。
- onFrameCommitted() : onFrameCommitted 在将帧提交给硬件合成器之后调用。Surface 会收集此帧的缓冲区的释放栅栏(release fence)。

类 DisplayDevice 是 DisplaySurface 接口的代理：

| No. | DisplayDevice            | DisplaySurface        |
|:---:|--------------------------|-----------------------|
| 1   | beginFrame()             | beginFrame()          |
| 2   | prepareFrame()           | prepareFrame()        |
| 3   | swapBuffers()            | advanceFrame()        |
| 4   | compositionComplete()    | compositionComplete() |
| 5   | onSwapBuffersCompleted() | onFrameCommitted()    |
| 6   | setDisplaySize()         | resizeBuffers()       |

这里的类的关系的设计有点复杂：
* 在 SurfaceFlinger 中，一个显示设备对应一条 DisplayDevice-DisplaySurface 管道。图形系统中：
  - 一定会有一个基本显示设备，一般是 LCD。
  - 可能会有一个扩展显示设备，一般是 HDMI。
  - 可能会有0个或多个虚拟显示设备，例如无线屏显。
* 类 DisplayDevice 是 DisplaySurface 接口的代理。
* 同时类 DisplayDevice 和 DisplaySurface 接口之间插入一个 BufferQueue。这是因为：
  - Layer 之间按照 z-order 合成，实际上就是产生了新的内容。这个内容由物理显示设备消费。
  - SurfaceFlinger 在合成时，只和 DisplayDevice 打交道。DisplayDevice 是新内容的生产端。由 DisplayDevice 提供 Surface，并生成 GLES 合成所用的 EGLSurface。一个显示设备一个 Surface。
  - 该 Surface 下面的 GraphicBuffer，由 DisplaySurface 提供相应显示设备的 Framebuffer 构造。最终，合成到 Framebuffer 上面的图像就显示到设备上。所以 DisplaySurface 是新内容的消费端。

这里需要再次说明一点，这一阶段的内容消费的节拍由 VSYNC 信号决定。实际的合成操作在 SurfaceFlinger::handleMessageRefresh() 方法里，这是一个很固定的调用序列：
```C++
* preComposition();     //合成前的准备
* rebuildLayerStacks(); //重新构建layer栈: z-order
* setUpHWComposer();    //HWComposer的设定
* doDebugFlashRegions();//显示调试信息
* doComposition();      //正式合成工作
* postComposition();    //合成的后期工作
```

从 SurfaceFlinger 到 DisplayDevice 到 DisplaySurface 这3级调用序列如下：
```C++
00) SurfaceFlinger::handleMessageRefresh()
  01) preComposition();     //合成前的准备
  01) rebuildLayerStacks(); //重新构建layer栈: z-order
  01) setUpHWComposer();    //HWComposer的设定
    02) DisplayDevice::beginFrame()
      03) DisplaySurface::beginFrame()
    02) HWComposer::prepare()
      03) hwc_composer_device_1::prepare()
    02) DisplayDevice::prepareFrame()
      03) DisplaySurface::prepareFrame()
  01) doDebugFlashRegions();//显示调试信息
  01) doComposition();      //正式合成工作
    02) doDisplayComposition()
      03) DisplayDevice::swapBuffers()
        04) eglSwapBuffers()
        04) DisplaySurface::advanceFrame()
    02) DisplayDevice::compositionComplete()
      03) DisplaySurface::compositionComplete()
    02) postFramebuffer()
      03) HWComposer::commit()
        04) hwc_composer_device_1::set()
      03) DisplayDevice::onSwapBuffersCompleted()
        04) DisplaySurface::onFrameCommitted()
  01) postComposition();    //合成的后期工作
    02) DisplayDevice::onSwapBuffersCompleted()
      03) DisplaySurface::onFrameCommitted()
```

调用序列图如下：
![DisplayDevice-DisplaySurface](https://github.com/shuyong/Design-Of-Android-6.0-Graphic-System/blob/master/image/general-design/SurfaceFlinger%20Sequence%20Diagram%20-%20DisplayDevice-DisplaySurface.svg)

DisplaySurface 接口有2个实现：
* FramebufferSurface
* VirtualDisplaySurface

下面就简单分析这两种实现的设计。

### FramebufferSurface 的设计

下图是 DisplayDevice-FramebufferSurfac 的类协作图：

![DisplayDevice-FramebufferSurface](https://github.com/shuyong/Design-Of-Android-6.0-Graphic-System/blob/master/image/general-design/Buffer%20Queue%20%20Component%20Diagram%20-%20DisplayDevice-FramebufferSurface.svg)

因为 FramebufferSurface 管理的是物理显示设备的 Framebuffer。可以认为 FramebufferSurface 是 Android 图形流水线的终点。所以 FramebufferSurface 的设计和前面的图形消费端的类的实现没有什么特别的。
* FramebufferSurface 实现了 ConsumerBase 接口，所以可以调用 IGraphicBufferConsumer 接口的方法。
* FramebufferSurface 重载了 ConsumerBase::onFrameAvailable() 虚函数。所以 onFrameAvailable() 消息传递的路径和第一阶段的有点不一样。

```C++
FramebufferSurface::onFrameAvailable()
00) BufferQueueProducer::queueBuffer()
  01) IConsumerListener::onFrameAvailable() == BnConsumerListener::onFrameAvailable() == BufferQueue::ProxyConsumerListener::onFrameAvailable()
    02) ConsumerListener::onFrameAvailable() == FramebufferSurface::onFrameAvailable()
      03) HWComposer::fbPost()
        04) HWComposer::setFramebufferTarget()
```

调用序列图如下：
![DisplayDevice-FramebufferSurface](https://github.com/shuyong/Design-Of-Android-6.0-Graphic-System/blob/master/image/general-design/SurfaceFlinger%20Sequence%20Diagram%20-%20DisplayDevice-FramebufferSurface-HWComposer.svg)

### VirtualDisplaySurface 的设计

VirtualDisplaySurface 管理的是普通的 GraphicBuffer。图形流水线到此，既是结束，也是开始。所以 VirtualDisplaySurface 的设计有点复杂，既需要调用 IGraphicBufferConsumer 接口，也需要调用 IGraphicBufferProducer 接口。因为相关内容太多，这里不再展开讨论。

下图是 DisplayDevice-VirtualDisplaySurface 的协作图：
![DisplayDevice-VirtualDisplaySurface](https://github.com/shuyong/Design-Of-Android-6.0-Graphic-System/blob/master/image/general-design/Buffer%20Queue%20%20Component%20Diagram%20-%20DisplayDevice-VirtualDisplaySurface.svg)

下图是 DisplayDevice-VirtualDisplaySurface 的调用序列图：
![DisplayDevice-VirtualDisplaySurface](https://github.com/shuyong/Design-Of-Android-6.0-Graphic-System/blob/master/image/general-design/SurfaceFlinger%20Sequence%20Diagram%20-%20DisplayDevice-VirtualDisplaySurface-HWComposer.svg)

## 两阶段之间的连接

阶段一和阶段二之间的连接，可以从下面的 Layer-DisplayDevice 协作图看出来。
![Stage Linker](https://github.com/shuyong/Design-Of-Android-6.0-Graphic-System/blob/master/image/general-design/Buffer%20Queue%20%20Component%20Diagram%20-%20Linker.svg)

在 SurfaceFlinger 中，合成工作分成两个步骤完成：
* handleMessageInvalidate() : 处理 VSYNC 信号。但不进行实际的合成操作，而是在事件队列里插入一个 REFRESH 信号，触发下一步的操作。
* handleMessageRefresh() : 处理 REFRESH 信号，进行实际的合成操作。

这是一个典型的 Command Pattern。与两阶段之间的连接相关的调用顺序如下：
```C++
00) SurfaceFlinger::handleMessageInvalidate()
  01) SurfaceFlinger::handlePageFlip()
    02) Layer::latchBuffer()
      03) SurfaceFlingerConsumer::updateTexImage()
        04) SurfaceFlingerConsumer::acquireBufferLocked()
          05) GLConsumer::acquireBufferLocked()
            06) ConsumerBase::acquireBufferLocked()
        04) GLConsumer::updateAndReleaseLocked()
          05) GLConsumer::releaseBufferLocked()
            06) ConsumerBase::releaseBufferLocked()
        04) GLConsumer::bindTextureImageLocked()

00) SurfaceFlinger::handleMessageRefresh()
  01) SurfaceFlinger::doComposition()
    03) SurfaceFlinger::doDisplayComposition()
      04) SurfaceFlinger::doComposeSurfaces()
        05) Layer::draw()
          06) Layer::onDraw()
            07) SurfaceFlingerConsumer::bindTextureImage()
              08) GLConsumer::bindTextureImageLocked()
            07) Layer::drawWithOpenGL()
```

相关的调用序列图如下：
![Stage Linker](https://github.com/shuyong/Design-Of-Android-6.0-Graphic-System/blob/master/image/general-design/SurfaceFlinger%20Sequence%20Diagram%20-%20Linker.svg)

前面强调过：内容生产阶段的节拍由 Application 来决定的；内容消费阶段的节拍由 VSYNC 信号决定。那两个阶段的工作节拍的协调，就是编舞者(Choreographer)框架要解决的问题。如同在一条有多个红绿灯的交通大道，红绿灯之间要协调，才能使得车流顺畅不阻塞。只有流水线上各个工作节拍相协调，才能使得流水线不会出现大的阻塞和性能抖动情况。

回过头看，Application 要想 UI 显示顺滑，就要适应 VSYNC 的工作模式。VSYNC 以 60HZ 频率工作，Application 以 120HZ 频率生产内容是没有意义的。同时，具有复杂 UI 内容的 Application，如动作游戏，如果生产一帧的内容时间太长，最好使内容生产分段，每段的时间控制在 16.67ms 以内，则 UI 显示就会顺滑很多。


