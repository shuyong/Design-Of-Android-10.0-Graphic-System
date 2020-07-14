# 两阶段(2-stage)流水线设计
* * *

# 接口与协作关系
在 Android 图形系统中，主要就是 Producer-Consumer 模式。体现 Producer-Consumer 模式的类就是 BufferQueue。而大多数接口(Interface)，基本上就是位于生产端/消费端这两端，定义着各种抽象功能，从而定义了数据的流向。

同时，Android 图形系统根据驱动节拍的划分，将整个图形处理的流水线划分为多个阶段(stage)。每个阶段的核心，都是 BufferQueue，而两端就是接口(Interface)的实现。这就体现了在不同阶段，对图像数据的不同处理。通过分析 BufferQueue 消费端的 ConsumerBase 接口(Interface)的具体实现，也就是管道的终点(endpoint)，就可以知道该 BufferQueue 所处的阶段。

典型的，显示在 LCD/HDMI 上面的界面就是两阶段(2-stage)的 Producer-Consumer 模式。而虚拟显示，则是多个阶段组合的 Producer-Consumer 模式。

而 ConsumerBase 接口(Interface)的具体实现有：

* Stage-1 Realized

| No. | Realized            | Usage               |
|:---:|---------------------|---------------------|
| 1   | BufferItemConsumer  | Camera Input Stream |
| 2   | BufferLayerConsumer | BufferQueueLayer    |
| 3   | CpuConsumer         | ScreenCapture       |
| 4   | GLConsumer          |                     |
| 5   | RingBufferConsumer  | Camera ZSL Stream   |
| 6   | SurfaceTexture      | hwui                |


* Stage-2 Realized

| No. | Realized              | Usage         |
|:---:|-----------------------|---------------|
| 1   | FramebufferSurface    | RenderSurface |
| 2   | VirtualDisplaySurface | RenderSurface |

本文主要讨论普通应用 UI 的显示流程，也就是从第一阶段的 BufferLayerConsumer/BufferQueueLayer 这个端点，到达第二阶段的 FramebufferSurface 这个端点的这条路径。

## Stage-1 两端的接口(Interface)

BufferQueue 一个阶段的生产消费循环，是 Surface-BufferQueue-ConsumerBase 这样的管道。一般分析 BufferQueue 的文档，大多数就是分析图形流水线的第一阶段。这阶段的 BufferQueue，由 BufferQueueLayer 类创建。内容(Content) 从 Surface 到达 BufferLayerConsumer 这一个生产消费循环。这在前面已经详细分析过，这里不再重复。

这阶段的 BufferQueue 的协作图，涉及到大多数图形相关的接口(Interface)：
* IGraphicBufferProducer
* IGraphicBufferConsumer
* IProducerListener
* IConsumerListener
* ANativeWindow
* ANativeWindowBuffer

相关的功能前面已经详细分析过，这里不再重复。这里需要再次说明一点，这一阶段的内容生产的节拍由 Application 来决定，也就是调用 eglSwapBuffers() 的频率。例如玩游戏时，键盘鼠标的输入，传感器的输入，网络上传来的队友和对手的消息，都将引发游戏引擎生产新的游戏画面，然后调用 eglSwapBuffers() 来刷新屏幕。游戏引擎对输入的反馈越灵敏越好。

## Stage-2 两端的接口(Interface)

图形流水线的第二阶段，就是合成内容(Content) 从 DisplayDevice 到达 DisplaySurface 这一个消费循环。

这阶段的 BufferQueue 的协作图，涉及到的接口(Interface)和前面差不多，新增的接口中有 2 个需要关注：
* DisplaySurface
* RenderSurface

DisplaySurface 接口(Interface)的主要功能：
- beginFrame() : 调用该方法以便发出渲染已开始的信号。在确定合成配置之前，在合成循环的开头调用 beginFrame。DisplaySurface 应该做它需要做的任何事情，以使 HWComposer 能够决定如何合成帧(frame)。我们传递进来参数 mustRecompose，这样我们就可以保持 VirtualDisplaySurface 的状态机正常工作，而无需在没有任何变化的情况下去排队等待缓冲区。如果必须重新合成整个帧，则 mustRecompose 应为 true。
- prepareFrame() : 准备渲染帧。prepareFrame 在合成配置确定之后，但在合成发生之前被调用。DisplaySurface 可以使用合成类型来决定如何管理关于此帧的在 GLES 和 HWC 之间的缓冲流。
- advanceFrame() : 通知 Surface，该帧的 GLES 合成已完成，Surface 应确保 HWComposer 具有该帧的正确缓冲区。有些实现可能只会在发生 GLES 合成时向 HWComposer 推送一个新的缓冲区，而其它实现则是在每一帧上推送一个新的缓冲区。调用 advanceFrame 之后必须跟着调用一次 onFrameCommitted，因为 advanceFrame 可能会再次被调用到。上层代码是在调用 queueBuffer() 将绘制好的缓冲区入队等待 HWC 使用时调用该方法。
- onFrameCommitted() : onFrameCommitted 在将帧提交给硬件合成器之后被调用。Surface 会收集该帧的缓冲区的释放栅栏(release fence)。上层代码是在 HWC 调用已完成后，调用该方法以表示该帧已显示到屏幕上。
- resizeBuffers() : 设置 Surface 的尺寸。
- getClientTargetAcquireFence() : 获取要传递给 HWC 的最新栅栏，以向 HWC 指示 Surface 缓冲区已完成渲染。

RenderSurface 接口是 DisplaySurface 接口的代理：

| No. | RenderSurface                 | DisplaySurface                |
|:---:|-------------------------------|-------------------------------|
| 1   | beginFrame()                  | beginFrame()                  |
| 2   | prepareFrame()                | prepareFrame()                |
| 3   | queueBuffer()                 | advanceFrame()                |
| 4   | onPresentDisplayCompleted()   | onFrameCommitted()            |
| 5   | setDisplaySize()              | resizeBuffers()               |
| 6   | getClientTargetAcquireFence() | getClientTargetAcquireFence() |


这里的接口和实现类设计得有点复杂：
* 在 SurfaceFlinger 中，一个显示设备对应一条 RenderSurface-DisplaySurface 管道。图形系统中：
  - 一定会有一个基本显示设备，一般是 LCD。
  - 可能会有一个扩展显示设备，一般是 HDMI。
  - 可能会有 0 个或多个虚拟显示设备，例如无线屏显。
* RenderSurface 接口是 DisplaySurface 接口的代理。
* 同时 RenderSurface 接口和 DisplaySurface 接口之间插入一个 BufferQueue。这是因为：
  - Layer 之间按照 z-order 合成，实际上就是产生了新的内容。这个内容由物理显示设备消费。
  - SurfaceFlinger 在合成时，只和 RenderSurface 打交道。RenderSurface 是新内容的生产端。由 RenderSurface 提供 ANativeWindowBuffer / GraphicBuffer，并生成 GLESRenderEngine 合成所用的 Texture。一个显示设备有一个 Surface，通过 dequeueBuffer() / queueBuffer() 方法轮流使用 BufferQueue 中的 GraphicBuffer。
  - 该 Surface 下面的 GraphicBuffer，由 DisplaySurface 提供相应显示设备的 Framebuffer 构造。最终，合成到 Framebuffer 上面的图像就显示到设备上。所以 DisplaySurface 是新内容的消费端。

这阶段的 BufferQueue，由 SurfaceFlinger 类委托超级工厂类 Factory 创建，只在 SurfaceFlinger 类内部使用。

下图是 RenderSurface 对内部的 IGraphicBufferProducer 的使用：
![NativeWindowSurface Class Diagram](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/general-design/services_surfaceflinger_NativeWindowSurface%20Class%20Diagram.svg)

从图中可以看到：
* 继续沿用以前的一些概念：在生产端的实现类还是类似 Surface，调用 IGraphicBufferProducer 接口。
* RenderSurface 会从这阶段的 BufferQueue 取出 ANativeWindowBuffer，并交给 GLESRenderEngine 使用。
* 这个 Buffer 将作为合成结果的底图，提交给 HWC 去显示。
* 也就是，在每个 VSYNC 信号到来时，SurfaceFlinger 将收集到的第一阶段生产的 Layer，按照 z-order 合成，结果就存放在这个 Buffer 上。然后又有一次 front / back buffer 交换。front buffer 就会显示到屏幕上。

下图是 DisplaySurface 对内部的 IGraphicBufferConsumer 的使用：
![DisplaySurface Component Diagram](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/general-design/services_surfaceflinger_CompositionEngine_include_compositionengine_DisplaySurface%20Class%20Diagram.svg)

从图中可以看到：
* DisplaySurface 接口的两个实现类：FramebufferSurface & VirtualDisplaySurface，同样也实现了 ConsumerBase 接口。因此它们可以调用 IGraphicBufferConsumer 接口获取包含内容的帧。
* 在这个阶段的 BufferQueue 生产消费循环里，DisplaySurface 无需监听 FrameAvailableListener 消息。因为这两个接口的实现类都在同一个进程里。RenderSurface 接口可以直接调用 DisplaySurface 接口的方法，这样就可以立即同步状态，而无需缓慢的 IConsumerListener 消息传播。

下图是第二阶段的 BufferQueue 的协作图：
![BufferQueue Component Diagram - FramebufferSurface](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/framework-design/gui_BufferQueue%20Component%20Diagram%20-%20FramebufferSurface.svg)

从图中可以看到：
* 在 RenderSurface 接口和 DisplaySurface 接口之间插入一个 BufferQueue。
* 合成引擎(Composition Engine)由渲染引擎(Render Engine)和硬件合成器(HWComposer)提供支持。SurfaceFlinger 合成第一阶段生产的 Layer 时，会根据 HWComposer 的提示，分别调用 Render Engine 和 HWComposer 做合成。
* 最终合成结果的底图，提交给 HWComposer 去显示。

# Stage-2 的调用序列

这里需要再次说明一点，第二阶段的内容消费的节拍由 VSYNC 信号决定。实际的合成操作在 SurfaceFlinger::handleMessageRefresh() 方法里，这是一个很固定的调用序列：
```C++
* preComposition();     //合成前的准备
* rebuildLayerStacks(); //重新构建layer栈: z-order
* calculateWorkingSet();//计算工作列表
* beginFrame();         //开始合成循环
* prepareFrame();       //HWComposer的准备
* doDebugFlashRegions();//显示调试信息
* doComposition();      //正式合成工作
* postFrame();          //记录合成帧状态
* postComposition();    //合成的后期工作
```

从 SurfaceFlinger-RenderSurface-DisplaySurface-HWComposer 这路径展开的调用序列如下：
```C++
00) SurfaceFlinger::handleMessageRefresh()
  01) preComposition();     //合成前的准备
  01) rebuildLayerStacks(); //重新构建layer栈: z-order
  01) calculateWorkingSet();//计算工作列表
  01) beginFrame();         //开始合成循环
    02) RenderSurface::beginFrame()                    // Called to signal that rendering has started.
      03) DisplaySurface::beginFrame()
  01) prepareFrame();       //HWComposer的准备
    02) RenderSurface::prepareFrame()                  // Prepares the frame for rendering
      03) HWComposer::prepare()
      03) DisplaySurface::prepareFrame()
  01) doDebugFlashRegions();//显示调试信息
  01) doComposition();      //正式合成工作
    02) doDisplayComposition()                         // repaint the framebuffer
      03) doComposeSurfaces()
        04) RenderSurface::dequeueBuffer()             // Allocates a buffer as scratch space for GPU composition
        04) RenderEngine::drawLayers()                 // Renders layers for a particular display via GPU composition.
      03) RenderSurface::queueBuffer()                 // Queues the drawn buffer for consumption by HWC.
        04) DisplaySurface::advanceFrame()             // Inform the surface that GLES composition is complete for this frame, and the surface should make sure that HWComposer has the correct buffer for this frame.
          05) HWComposer::setOutputBuffer()            // Set the output buffer and acquire fence for a virtual display.
          05) HWComposer::setClientTarget()
    02) RenderSurface::flip()                          // Called after the surface has been rendering to signal the surface should be made ready for displaying
    02) postFramebuffer()
      03) HWComposer::presentAndGetReleaseFences()     // Present layers to the display and read releaseFences.
      03) RenderSurface::onPresentDisplayCompleted()   // Called after the HWC calls are made to present the display
        04) DisplaySurface::onFrameCommitted()         // Called after the frame has been committed to the hardware composer.
          05) HWComposer::getPresentFence()            // get the present fence received from the last call to present.
      03) RenderSurface::getClientTargetAcquireFence() // Gets the latest fence to pass to the HWC to signal that the surface buffer is done rendering
        04) DisplaySurface::getClientTargetAcquireFence()
  01) postFrame();          //记录合成帧状态
  01) postComposition();    //合成的后期工作
    02) RenderSurface::getClientTargetAcquireFence()   // Gets the latest fence to pass to the HWC to signal that the surface buffer is done rendering
      03) DisplaySurface::getClientTargetAcquireFence()
```

调用序列图如下：
![RenderSurface-DisplaySurface](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/general-design/services_surfaceflinger_SurfaceFlinger%20Sequence%20Diagram%20-%20RenderSurface-DisplaySurface.svg)

DisplaySurface 接口有2个实现：
* FramebufferSurface
* VirtualDisplaySurface

下面就简单分析这两种实现的设计。

## FramebufferSurface 的设计

上一节的协作图就是 DisplaySurface-FramebufferSurfac 的类协作图。这里不再重复。

因为 FramebufferSurface 实现了 ConsumerBase 接口，所以可以调用 IGraphicBufferConsumer 接口的方法。但是如果看代码，FramebufferSurface 啥活都没干。因为 FramebufferSurface 管理的是物理显示设备的 Framebuffer，也就是 Buffer 中包含 HWC_FRAMEBUFFER_TARGET 这个属性。对于有 HWC_FRAMEBUFFER_TARGET 这个属性的 Buffer，HWC 会将其中的内容显示到屏幕上。因为该 Buffer 被安排在 z-order 的最后一层，承载着前面所有的 Layer 的内容的合成结果。因此可以认为 FramebufferSurface 是 Android 图形流水线的终点，HWC 会把剩下的活干完，所以 FramebufferSurface 其实无需做什么。

## VirtualDisplaySurface 的设计

VirtualDisplaySurface 管理的是普通的 GraphicBuffer。图形流水线到此，既是结束，也是开始。所以 VirtualDisplaySurface 的设计有点复杂，既需要调用 IGraphicBufferConsumer 接口，也需要调用 IGraphicBufferProducer 接口，本身又是 IGraphicBufferProducer 接口的实现。VirtualDisplaySurface 用 IGraphicBufferConsumer 接口获取承载合成内容的 Buffer，然后再用其它地方提供的 IGraphicBufferProducer 接口将其送出去，开始一段新的旅程。因为相关内容太多，这里不再展开讨论。

下图是 RenderSurface-VirtualDisplaySurface 的协作图：
![RenderSurface-VirtualDisplaySurface](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/framework-design/gui_BufferQueue%20Component%20Diagram%20-%20VirtualDisplaySurface.svg)

# 两阶段之间的连接

阶段一和阶段二之间的连接，就是做为阶段一的 BufferQueue 的 Consumer 端，调用 IGraphicBufferConsumer 接口的 acquireBuffer / releaseBuffer 方法和应用程序交换数据，获取新帧并进行合成；同时做为阶段二的 BufferQueue 的 Producer 端，调用 IGraphicBufferProducer 接口的 dequeueBuffer / queueBuffer 方法和 RenderSurface 交换数据，将合成帧送出去显示。这可以从下面的 Layer-RenderSurface 协作图看出来。
![Stage Linker](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/framework-design/gui_BufferQueue%20Component%20Diagram%20-%20link.svg)

在 SurfaceFlinger 中，合成工作分成两个步骤完成：
* handleMessageInvalidate() : 处理 VSYNC 信号。但不进行实际的合成操作，而是在事件队列里插入一个 REFRESH 信号，触发下一步的操作。与 Layer 的相关工作是：
  + latch buffer : 锁定 Layer 缓冲区。
  + update texture image : 绑定 Layer 缓冲区为 GL 纹理。
* handleMessageRefresh() : 处理 REFRESH 信号，进行实际的合成操作。最后 DisplaySurface 就会得到合成结果。

这是一个典型的 Command Pattern。之所以会这样，是因为在一个 VSYNC 周期中，如果一个暴露的 Window 没有更新内容，又需要显示出去，就采用旧的帧，于是在 Invalidate 阶段就不需要锁定和绑定 Layer 缓冲区的动作。然后在 Refresh 阶段，获取、锁定和绑定 DisplaySurface 缓冲区，并将其置于 z-order 的最底层。这样，该缓冲区就会承载全局最新合成结果。

与两阶段之间的连接相关的调用顺序如下：
```C++
00) SurfaceFlinger::handleMessageInvalidate()
  01) SurfaceFlinger::handlePageFlip()                     // latch a new buffer if available and compute the dirty region.
    02) Layer::latchBuffer() == BufferLayer::latchBuffer() // called each time the screen is redrawn and returns whether the visible regions need to be recomputed.
      03) BufferLayer::updateTexImage() == BufferQueueLayer::updateTexImage() // acquires the most recently queued buffer, and sets the image contents of the target texture to it.
        04) BufferLayerConsumer::updateTexImage()
          05) BufferLayerConsumer::acquireBufferLocked()   // fetches the next buffer from the BufferQueue and updates the buffer slot for the buffer returned.
            06) ConsumerBase::acquireBufferLocked()
              07) IGraphicBufferConsumer::acquireBuffer()
        04) BufferLayerConsumer::updateAndReleaseLocked()  // Release the previous buffer.
          05) ConsumerBase::releaseBufferLocked()          // relinquishes control over a buffer, returning that control to the BufferQueue.
            06) IGraphicBufferConsumer::releaseBuffer()
        04) BufferLayerConsumer::bindTextureImageLocked()  // Binds the current buffer to TEXTURE_EXTERNAL target.
          05) RenderEngine::bindExternalTextureBuffer()

00) SurfaceFlinger::handleMessageRefresh()
  01) preComposition();     //合成前的准备
  01) rebuildLayerStacks(); //重新构建layer栈: z-order
  01) calculateWorkingSet();//计算工作列表
  01) beginFrame();         //开始合成循环
    02) RenderSurface::beginFrame()                    // Called to signal that rendering has started.
      03) DisplaySurface::beginFrame()
  01) prepareFrame();       //HWComposer的准备
    02) RenderSurface::prepareFrame()                  // Prepares the frame for rendering
      03) HWComposer::prepare()
      03) DisplaySurface::prepareFrame()
  01) doDebugFlashRegions();//显示调试信息
  01) doComposition();      //正式合成工作
    02) doDisplayComposition()                         // repaint the framebuffer
      03) doComposeSurfaces()
        04) RenderSurface::dequeueBuffer()             // Allocates a buffer as scratch space for GPU composition
          05) ANativeWindow::dequeueBuffer() == Surface::dequeueBuffer()
            06) IGraphicBufferProducer::dequeueBuffer()
        04) BufferLayer::prepareClientLayer()          // constructs a RenderEngine layer for GPU composition.
          05) Layer::prepareClientLayer()              // populates a renderengine::LayerSettings to passed to RenderEngine::drawLayers.
        04) RenderEngine::drawLayers()                 // Renders layers for a particular display via GPU composition.
      03) RenderSurface::queueBuffer()                 // Queues the drawn buffer for consumption by HWC.
        04) ANativeWindow::dequeueBuffer() == Surface::dequeueBuffer()
          05) IGraphicBufferProducer::dequeueBuffer()
        04) DisplaySurface::advanceFrame()             // Inform the surface that GLES composition is complete for this frame, and the surface should make sure that HWComposer has the correct buffer for this frame.
          05) HWComposer::setOutputBuffer()            // Set the output buffer and acquire fence for a virtual display.
          05) HWComposer::setClientTarget()
    02) RenderSurface::flip()                          // Called after the surface has been rendering to signal the surface should be made ready for displaying
    02) postFramebuffer()
      03) HWComposer::presentAndGetReleaseFences()     // Present layers to the display and read releaseFences.
      03) RenderSurface::onPresentDisplayCompleted()   // Called after the HWC calls are made to present the display
        04) DisplaySurface::onFrameCommitted()         // Called after the frame has been committed to the hardware composer.
          05) HWComposer::getPresentFence()            // get the present fence received from the last call to present.
      03) RenderSurface::getClientTargetAcquireFence() // Gets the latest fence to pass to the HWC to signal that the surface buffer is done rendering
        04) DisplaySurface::getClientTargetAcquireFence()
  01) postFrame();          //记录合成帧状态
  01) postComposition();    //合成的后期工作
    02) RenderSurface::getClientTargetAcquireFence()   // Gets the latest fence to pass to the HWC to signal that the surface buffer is done rendering
      03) DisplaySurface::getClientTargetAcquireFence()
```

从上述调用序列可以看出：
* 在 Invalidate 阶段：调用 IGraphicBufferConsumer 接口的 acquireBuffer / releaseBuffer 方法和应用程序交换数据，获取新帧并绑定 Layer 缓冲区为 GL 纹理。
* 在 Refresh 阶段：调用 IGraphicBufferProducer 接口的 dequeueBuffer / queueBuffer 方法和 RenderSurface 交换数据，获取 DisplaySurface 缓冲区，绑定 GL 纹理，并将其置于 z-order 的最底层。调用 RenderEngine 进行一部分 Layer 的合成，接着调用 CompositionEngine / HWComposer 进行最后的合成。如果 DisplaySurface 缓冲区具有 HWC_FRAMEBUFFER_TARGET 属性，HWComposer 会将承载合成结果的缓冲区提交给底层的显示设备，显示在屏幕上。

相关的调用序列图如下：
![Stage Linker](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/general-design/services_surfaceflinger_SurfaceFlinger%20Sequence%20Diagram%20-%20Layer-DisplaySurface.svg)

前面强调过：内容生产阶段的节拍由 Application 来决定的；内容消费阶段的节拍由 VSYNC 信号决定。那两个阶段的工作节拍的协调，就是编舞者(Choreographer)框架要解决的问题。如同在一条有多个红绿灯的交通大道，红绿灯之间要协调，才能使得车流顺畅不阻塞。只有流水线上各个工作节拍相协调，才能使得流水线不会出现大的阻塞和性能抖动情况。

回过头看，Application 要想 UI 显示顺滑，就要适应 VSYNC 的工作模式。如果 VSYNC 以 60Hz 频率工作，Application 以 120Hz 频率生产内容是没有意义的。同时，具有复杂 UI 内容的 Application，如动作游戏，如果生产一帧的内容时间太长，最好使内容生产分段，每段的时间控制在一个时钟周期以内，如 60Hz 为 16.6ms，90Hz 为 11.1ms 以内，则 UI 显示就会顺滑很多。

# 两阶段的流水线

如果将两个阶段的接口和类的组合关系拼接在一起，并且加上 VSYNC 信号的组合关系，就得到下面这个无比复杂的关系组合图：(可能需要 8K 显示器才能完整地显示出来)
![2-stage pipeline](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/framework-design/gui_BufferQueue%20Component%20Diagram%20-%202-stage.svg)

从图中可以看出：
* 应用程序生产的内容是如何通过两个阶段的 BufferQueue 流动到屏幕上的。
* VSYNC 信号是如何转换和分发的，Choreographer 框架是如何工作的。

