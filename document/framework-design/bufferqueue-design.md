# BufferQueue 的设计
* * *

# 基本的设计思想

BufferQueue 类是 Android 中所有图形处理操作的核心。它的作用很简单：将生成图形数据缓冲区的一方（生产方）连接到接受数据以进行显示或进一步处理的一方（消费方）。几乎所有在系统中移动图形数据缓冲区的内容都依赖于 BufferQueue。 

BufferQueue 连接着生产方和消费方。因此基本用法很简单：
1. 生产方请求一个可用的缓冲区 (dequeueBuffer())，并指定一组特性，包括宽度、高度、像素格式和用法标记。
2. 生产方填充缓冲区并将其返回到队列 (queueBuffer())，并且向 IConsumerListener 发送 onFrameAvailable() 消息。
3. 随后，消费方获取该缓冲区 (acquireBuffer()) 并使用该缓冲区的内容。
4. 当消费方操作完毕后，将该缓冲区返回到队列 (releaseBuffer())。 

最新的 Android 设备支持“同步框架”，这使得系统能够在与可以异步处理图形数据的硬件组件结合使用时提高工作效率。例如，生产方可以提交一系列 OpenGL ES 绘制命令，然后在渲染完成之前将输出缓冲区加入队列。该缓冲区伴有一个栅栏(Fence)，当内容准备就绪时，栅栏会发出信号。当该缓冲区返回到空闲列表时，会伴有第二个栅栏，因此消耗方可以在内容仍在使用期间释放该缓冲区。该方法缩短了缓冲区通过系统时的延迟时间，并提高了吞吐量。

用于显示 Surface 的 BufferQueue 通常配置为三重缓冲；但按需分配缓冲区。因此，如果生产方足够缓慢地生成缓冲区 --- 也许是以 30fps 的速度在 60fps 的显示屏上播放动画 --- 队列中可能只有两个分配的缓冲区。这有助于最小化内存消耗。 

# 从 GraphicBuffer 的用途看 BufferQueue 的设计

从 BufferQueue 管理的对象(GraphicBuffer)的用途看 BufferQueue 的功能，更容易理解 BufferQueue 的设计。

对于本地申请的 GraphicBuffer，主要用于生成 EGLImage，并绑定到某个 Texture ID 上。

![GraphicBuffer - native](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/framework-design/ui_GraphicBuffer%20Component%20Diagram%20-%20native.svg)
 
考虑到跨进程的 Client / Server 应用模式，在 Android 6.x 以前提供了一个跨进程的 IGraphicBufferAlloc 接口：

![GraphicBuffer - Client-Server](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/framework-design/Graphic%20Buffer%20Alloc%20Component%20Diagram%20-%20Client-Server.svg)

但是上图的应用方式有一个问题：如果客户端改变了 GraphicBuffer 的属性(width / height / format)，服务端无法自动感知对方属性的变化，因此管理起来就很麻烦。因此在 android 代码里没有跨进程方式的 IGraphicBufferAlloc 接口的应用，而是为了便于管理，提供给 Surface 使用的 GraphicBuffer，都集中到 BufferQueue 中进行统一申请和管理。BufferQueue 对 Producer 端，提供的是 back buffer，最主要的功能就是 queue/dequeue buffer；对 Consumer 端，提供的是 front buffer，最主要的功能就是 acquire/release buffer。后来引入 HIDL 项目以后，整个 Framework 层都浮在 HIDL 层之上，就取消了 IGraphicBufferAlloc 接口。

因此，从下面的协作图可以得知，客户端软件对 Surface 下面的 GraphicBuffer 的使用，是通过 IGraphicBufferProducer 接口与 BufferQueue 交互实现的，这就包括了属性(width / height / format)的修改。

![Buffer Queue - GraphicBuffer Allocation](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/framework-design/gui_BufferQueue%20Component%20Diagram%20-%20GraphicBuffer%20Allocation.svg)

相对应的，服务端软件 GraphicBuffer 的使用，是通过 IGraphicBufferConsumer 接口与 BufferQueue 交互实现的。因此，从以 BufferQueue 为中心的协作图来看，BufferQueue 是一种很对称的设计，这既包括了 Producer-Consumer 设计模式，也包括了 Listener 设计模式。

# BufferQueue 的系统设计

以 BufferQueue 为中心的缓冲区系统是有界且是闭环的。所谓有界，就是指 Buffer Pool 里的 GraphicBuffer 个数是有限的。有限的个数会制约生产和消费的速率。所谓闭环，就是指在闭环路径中，存在从消费者返回到生产者的缓冲区路径。 

类 BufferQueue 有一个静态的工厂方法，BufferQueue::createBufferQueue，用于创建 BufferQueue 实例。这里应用了工厂设计模式。

类 BufferQueue 只是类 BufferQueueCore 的一个很薄的外观(facade)。这里应用了外观(facade)设计模式。BufferQueueCore 包含实际的实现逻辑。后面为了简化讨论，除非特别指明，不会区分这些类。 

使用 BufferQueue 相当简单。首先，生产者和消费者连接到缓冲队列。 
* 生产者从 BufferQueue 中取出一个“空”缓冲区（dequeueBuffer） 
* 生产者（例如照相机）将图像或图形数据复制到缓冲器中。 
* 生产者将“已填充好”的缓冲区返回到 BufferQueue（queueBuffer）。 
* 消费者接收到“已填充好”的缓冲区的存在（通过回调函数）的指示。 
* 消费者从 BufferQueue（acquireBuffer）中获取此缓冲区。 
* 当消费者完成消费时，缓冲区被释放回到 BufferQueue（releaseBuffer）。 

文档[[BufferQueue](https://www.codeproject.com/Articles/990983/Androids-Graphics-Buffer-Management-System-Part-II)]给出了一个简化交互图，显示了照相机（图像缓冲区生产者）和显示器（图像缓冲区消费者）之间的协作关系。

![BufferQueue - sample](https://4.bp.blogspot.com/-Tb1SF0rztvo/VTKmJ0BLmCI/AAAAAAAATE0/gdjePoXPGXc/s1600/gfx-sample_flow.png)

在图中，生产者和消费者可以分布在不同的进程中，并使用 Binder 接口完成跨进程的交互。 

BufferQueueProducer 是 IGraphicBufferProducer 后面做粗活的类。BufferQueueProducer 与 BufferQueueCore 保持密切关系，并直接访问其成员变量，包括互斥体、条件和其他重要成员（例如其指向 IGraphicBufferAlloc 的指针）。就我个人而言，我不喜欢这个实现 --- 它既混乱又脆弱。 

当 BufferQueue 被生产者使用 dequeueBuffer 请求提供一个空白缓冲区时，它试图从 BufferQueueCore 那里获取一个缓冲区。BufferQueueCore 维护着一组缓冲区及其状态（DEQUEUED、QUEUED、ACQUIRED、FREE）。如果在缓冲区数组中找到空闲槽(Free Slot)，但是这个槽不包含缓冲区，或者如果生产者被明确地要求重新分配缓冲区，那么 BufferQueueProducer 使用 BufferQueueCore 的方法来分配新的缓冲区。 

下图是在 BufferQueue 中申请 GraphicBuffer 的序列图。 
![BufferQueue - allocator](https://4.bp.blogspot.com/-r1cOrB9Tpwo/VTKy6JAmfuI/AAAAAAAATFo/NggW0ocLRU4/s1600/gfx-allocator_flow.png)

最初，所有的 dequeueBuffer 调用都会导致新缓冲区的分配。但是，因为这是一个闭环系统，缓冲区使用者在缓冲区消耗完内容后会返回缓冲区（通过调用 releaseBuffer），我们将看到系统在很短的时间内达到平衡。请注意，尽管 BufferQueueCore 可以维护可变大小的 GraphicBuffer 对象数组，但是将所有缓冲区设置为相同大小是明智的。否则，每次调用 dequeueBuffer 可能需要分配一个新的 GraphicBuffer 实例。 

类 BufferQueueCore 不直接存储 GraphicBuffer --- 它使用类 BufferItem，其中包含指向 GraphicBuffer 实例的指针，包含有各种其他元数据。 

异步通知接口 IConsumerListener 和 IProducerListener 用于向侦听器报警事件，例如缓冲区已准备好可以消费（IConsumerListener::onFrameAvailable）或一个空缓冲区已可用（IProducerListener::onBufferReleased）。这些回调接口也使用 Binder，所以可以跨越进程边界。 

下图是文档[[BufferQueue](https://www.codeproject.com/Articles/990983/Androids-Graphics-Buffer-Management-System-Part-II)]给出的类 BufferQueueCore 的协作图。
![](https://1.bp.blogspot.com/-xr2slfzKBXM/VTKn0NLp38I/AAAAAAAATFA/MXN3G59cs2U/s1600/gfx-buffer_queue_classes.png)

# 从全局看 BufferQueue 的设计

BufferQueue 是 Android 图形系统的核心之一。我们将从全局视角看更详细一些的 BufferQueue 相关的设计思路。

下图是以 BufferQueue 为中心管理 GraphicBuffer 时所涉及的接口和类的协作图。

![BufferQueue Component Diagram](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/framework-design/gui_BufferQueue%20Component%20Diagram.svg)

这是一个很复杂的协作图，但本质上就是一个 Producer-Queue-Consumer 模型。从图中我们可以看到一系列对称关系：
* BufferQueue 的设计是对称的。
  - Producer-Consumer 设计模式对应的是2个跨进程接口(interface)：IGraphicBufferProducer & IGraphicBufferConsumer。
  - Listener 设计模式对应的是2个跨进程接口(interface)：IProducerListener & IConsumerListener。
* 为 Producer-Consumer 设计模式所设计的类是对称的。
  - 在客户端的接口是 ANativeWindow，对应的服务端的接口是 Layer。
  - 在客户端的实现类是 Surface，对应的服务端的实现类是 BufferQueueLayer。
  - 也就是，一对 Surface-Layer，就是一对 Producer-Consumer 关系，中间存在一个 BufferQueue。
* 对 BufferQueue 的操控是对称的。
  - 在客户端是 Surface 调用 IGraphicBufferProducer 接口操控 BufferQueue。最常用的就是 dequeueBuffer() & queueBuffer() 这2个方法。
  - 在服务端是 SurfaceFlingerConsumer 侦听 IConsumerListener 接口的消息，然后调用 IGraphicBufferConsumer 接口操控 BufferQueue。最常用的就是 acquireBuffer() & releaseBuffer() 这2个方法。
  - 类 BufferLayerConsumer 由实现类 BufferQueueLayer 所创建。基于 C/S 模型，一个 Surface 对应一个 BufferLayerConsumer。

我们再稍微细化一下协作图，显示 MonitoredProducer 类的相关协作图。

![MonitoredProducer Component Diagram](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/server-design/services_surfaceflinger_BufferQueueLayer%20Component%20Diagram.svg)

当 Surface 调用 queueBuffer() 将新帧放入队列后，会主动调用 IConsumerListener 接口发送 FrameAvailable 消息。从上图可以看出，为了将 FrameAvailable 消息绑定到 Layer 接口上，使得 Consumer 端的 BufferLayerConsumer 类能被立即调用，于是 MonitoredProducer 类就产生了。

MonitoredProducer 类也是一个 IGraphicBufferProducer 接口的实现，但是它包装了 BufferQueue 提供的 IGraphicBufferProducer 接口。最终在 Surface 类中使用的 IGraphicBufferProducer 接口就是 MonitoredProducer 类做的实现。当 Surface 类发送 FrameAvailable 消息时，MonitoredProducer 类会转发给 Consumer 端的 Layer 接口。然后再经过几个跳转，BufferLayerConsumer 类就收到了 FrameAvailable 消息。具体的消息的流动，见下一章节的说明。

之所以有 MonitoredProducer 类这样一个包装类，可以和 SurfaceFlinger 类直接打交道，就是为了当 IGraphicBufferProducer 接口被析构的时候，MonitoredProducer 类可以直接通知 SurfaceFlinger 类，在它管理的列表里有一个 Layer 被销毁了。

# Listener 消息的流动

在 BufferQueue 的设计中，Listener 设计模式是对称的：IProducerListener & IConsumerListener。但在实际应用中，并没有使用IProducerListener 接口。消费端侦听 IConsumerListener 消息是为了能及时消费 Frame。而消费端有其它更好的方法影响生产端的生产速率。一是在有限和闭环的模型中，当消费速率跟不上生产速率，Buffer Pool 中的 Free Buffer 为空，自然就阻塞了生产。二是 Android 图形系统的显示(消费)，由 VSYNC 信号所驱动。VSYNC 信号由 IDisplayEventConnection 接口传递回应用上层，由应用上层控制生产速率，这样更合理一些。这就是 Choreographer 项目要解决的问题。

所以下面我们主要分析 IConsumerListener::onFrameAvailable() 的传输路径。当客户端调用 IGraphicBufferProducer::queueBuffer() 将包含内容的 Frame 放回 Buffer Pool 时，最终在 Binder thread 中调用了服务端的 IGraphicBufferProducer 接口的实现类 BufferQueueProducer，由此开始了 onFrameAvailable() 消息的旅行。

代码的调用次序如下：
```C++
00) Surface::queueBuffer() - on Application process
  01) IGraphicBufferProducer::queueBuffer() == BnGraphicBufferProducer::queueBuffer() == MonitoredProducer::queueBuffer() - on Binder thread of surfaceflinger
    02) IGraphicBufferProducer::queueBuffer() == BnGraphicBufferProducer::queueBuffer() == BufferQueueProducer::queueBuffer()
    02) BufferQueueProducer::queueBuffer()
      03) sp<IConsumerListener> frameAvailableListener;
      03) frameAvailableListener = mCore->mConsumerListener;
      03) frameAvailableListener->onFrameAvailable(item);
      03) IConsumerListener::onFrameAvailable() == BnConsumerListener::onFrameAvailable() == ProxyConsumerListener::onFrameAvailable()
        04) ConsumerListener::onFrameAvailable() == ConsumerBase::onFrameAvailable()
          05) FrameAvailableListener::onFrameAvailable() == BufferQueueLayer::onFrameAvailable()
            06) mQueueItems.push_back(item);
            06) mQueueItemCondition.broadcast();
            06) mConsumer->onBufferAvailable(item);
            06) BufferLayerConsumer::onBufferAvailable()
              07) mImages[item.mSlot] = std::make_shared<Image>(item.mGraphicBuffer, mRE);
```

下图是相关的协作图。从图中可以看到相关的调用序列。
![IConsumerListener Component](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/framework-design/gui_IConsumerListener%20Component%20Diagram.svg)

对上图做一些简化和变形，可以清楚地看到消息的流动过程：
![IConsumerListener Component - Call Path](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/framework-design/gui_IConsumerListener%20Component%20Diagram%20-%20Call%20Path.svg)

从上面的一系列图可知应用 GraphicBuffer 的 Producer-Consumer 模式的工作流程如下：
* 当生产端的 Surface 调用 IGraphicBufferProducer::queueBuffer() 时，意味着有新帧产生。
* BufferQueueProducer::queueBuffer() 会发送 IConsumerListener::onFrameAvailable() 出来。
* 最终是在消费端对应 ANativeWindow 接口的 Layer 接口的实现类 BufferQueueLayer 收到了经过 2 次跳转的 onFrameAvailable() 消息。
* BufferQueueLayer 类记录下新帧编号，并向 mSFEventThread 发送 requestNextVsync() 消息。然后返回。
* mSFEventThread 在下一个 VSYNC 信号到达时会向 SurfaceFlinger 发送 INVALIDATE 信号。然后返回。
* SurfaceFlinger 收到 INVALIDATE 信号后，通过 SurfaceFlingerConsumer 调用 IGraphicBufferConsumer::acquireBuffer() 方法获得该 Layer 的新帧，与其它 Layer 的新帧一起合成并显示出去。
* 上述步骤是异步执行的，queueBuffer() 位于 Binder thread，处理 VSYNC 信号位于 mSFEventThread，合成操作位于 Main thread。也就是，生产端的新帧的生成时机，由应用软件决定。消费端的合成与显示，由 VSYNC 信号所驱动。当然，两者如果能协调速率，则界面显示就更顺滑。这就是 Choreographer 项目要解决的问题。

# BufferQueue 的工作原理

BufferQueue 的实现，采用不少技巧。

BufferQueue 只存在于服务端。具体的 Buffer Pool，由 BufferQueueCore::mSlots 指向和管理。另外有几个成员协作管理：
* mDefaultMaxBufferCount: 本 BufferQueue 实例中能管理多少个 GraphicBuffer。
* mQueue: 保存新帧的向量，state = QUEUED。
* mFreeBuffers: 保存 Free Buffer 的向量，state = FREE。
* mFreeSlots: 保存 Free Slot 的向量，state = FREE。和上一个成员不同的地方在于 Free Slot 中没有申请 GraphicBuffer。

客户端的 Surface 通过 IGraphicBufferProducer 接口操控 BufferQueue。但它也同样有一个指针 Surface::mSlots，保存 BufferQueue 里面 GraphicBuffer 的申请状态：NULL or ALLOC。
 
BufferQueue 的具体实现，还需要考虑很多问题。下面我们通过一个具体的例子来进行说明。

## 假设

假设有一个 BufferQueue 实例：
* mDefaultMaxBufferCount: 原本为4个，生产端刚刚扩充为8个。原本的生产负荷很重，所以从 Surface::mSlots 角度看，[00# ~ 03#].state = ALLOC，新扩展的 [04# ~ 07#].state = NULL。
* mQueue: [00#, 01#]，state = QUEUED，size = 2。注意：如果生产端为 EGL 模块，则只允许保存一个新帧。这也许是为了严格保证 FIFO 的特性吧。这里按照一般情况讨论。
* mFreeBuffers: [03#]，state = FREE，size = 1。
* mFreeSlots: [04#, 05#, 06#, 07#]，state = FREE，size = 4。
* 生产端刚刚调用过 dequeueBuffer(): [02#]，state = DEQUEUED。

![BufferQueue-00](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/framework-design/BufferQueue-00.svg)

## 第1个使用瞬间

在这个时间片里：
* 生产端调用 queueBuffer()，跨进程调用 IGraphicBufferProducer::queueBuffer()，[02#]，state = QUEUED。
* 接着生产端调用 dequeueBuffer()，跨进程调用 IGraphicBufferProducer::dequeueBuffer()。
  - 从 mFreeBuffers 中最小的编号选取：[03#]，state = DEQUEUED。
* 消费端收到 onFrameAvailable() 消息，在下一个 VSYNC 信号到达时调用 IGraphicBufferConsumer::acquireBuffer()，按照 FIFO 要求，[00#]，state = ACQUIRED。

![BufferQueue-01](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/framework-design/BufferQueue-01.svg)

所以 BufferQueue 此时的状态是：
* mQueue: [01#, 02#]，state = QUEUED，size = 2。
* mFreeBuffers: empty，size = 0。
* mFreeSlots: [04#, 05#, 06#, 07#]，state = FREE，size = 4。
* [03#], state = DEQUEUED。
* [00#]，state = ACQUIRED。

## 第2个使用瞬间

在这个时间片里：
* 生产端调用 queueBuffer()，跨进程调用 IGraphicBufferProducer::queueBuffer()，[03#]，state = QUEUED。
* 接着生产端调用 dequeueBuffer()，跨进程调用 IGraphicBufferProducer::dequeueBuffer()。
  - 从 mFreeBuffers 中最小的编号选取。因为 mFreeBuffers: empty，失败，进入下一个选择。
  - 从 mFreeSlots 中最小的编号选取。接着服务端调用 IGraphicBufferProducer::requestBuffer() 申请 GraphicBuffer，并同步回生产端。
  - Surface::mSlots[04#]，state = ALLOC。
  - BufferQueueCore::mSlots[04#]，state = DEQUEUED。

![BufferQueue-02](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/framework-design/BufferQueue-02.svg)

所以 BufferQueue 此时的状态是：
* mQueue: [01#, 02#, 03#]，state = QUEUED，size = 3。
* mFreeBuffers: empty，size = 0。
* mFreeSlots: [05#, 06#, 07#]，state = FREE，size = 3。
* [04#], state = DEQUEUED。
* [00#]，state = ACQUIRED。

## 第3个使用瞬间

在这个时间片里：
* 消费端完成合成任务的提交，调用 IGraphicBufferConsumer::releaseBuffer()，[00#]，state = FREE。
* 消费端收到 onFrameAvailable() 消息，在下一个 VSYNC 信号到达时调用 IGraphicBufferConsumer::acquireBuffer()，按照 FIFO 要求，[01#]，state = ACQUIRED。

![BufferQueue-03](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/framework-design/BufferQueue-03.svg)

所以 BufferQueue 此时的状态是：
* mQueue: [02#, 03#]，state = QUEUED，size = 2。
* mFreeBuffers: [00#]，size = 1。
* mFreeSlots: [05#, 06#, 07#]，state = FREE，size = 3。
* [04#], state = DEQUEUED。
* [01#]，state = ACQUIRED。

## 第4个使用瞬间

* 生产端调用 queueBuffer()，跨进程调用 IGraphicBufferProducer::queueBuffer()，[04#]，state = QUEUED。
* 接着生产端调用 dequeueBuffer()，跨进程调用 IGraphicBufferProducer::dequeueBuffer()。
  - 从 mFreeBuffers 中最小的编号选取：[00#]，state = DEQUEUED。
* 消费端完成合成任务的提交，调用 IGraphicBufferConsumer::releaseBuffer()，[01#]，state = FREE。
* 消费端收到 onFrameAvailable() 消息，在下一个 VSYNC 信号到达时调用 IGraphicBufferConsumer::acquireBuffer()，按照 FIFO 要求，[02#]，state = ACQUIRED。

![BufferQueue-04](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/framework-design/BufferQueue-04.svg)

所以 BufferQueue 此时的状态是：
* mQueue: [03#, 04#]，state = QUEUED，size = 2。
* mFreeBuffers: [01#]，size = 1。
* mFreeSlots: [05#, 06#, 07#]，state = FREE，size = 3。
* [00#], state = DEQUEUED。
* [02#]，state = ACQUIRED。

## BufferQueue 实现的特点

所以 BufferQueue 的具体实现，除了完成 Producer-Consumer 模型的功能，还需要实现几个特别的功能：
* 对于生产端，在 Free Resource Pool 中按照 GraphicBuffer Slot 的最小编号选取。
  - 这可以使得申请的 GraphicBuffer 的个数最小。
  - 相应的，也使得已申请的 GraphicBuffer 尽快被复用到，使得新帧可以尽快被显示出去，延时最小。
  - 如果消费速率远大于生产速率，则可以变成 Single-Buffer 模式，消耗内存最少。
  - 一般情况下，消费速率与生产速率相当，就变成 Double-Buffer 模式。
  - 偶尔有性能的波动，消费速率小于生产速率，就变成 Triple-Buffer 模式。用空间换时间，减少生产端的阻塞。
  - Triple-Buffer 模式有一个缺点：新帧需要多一个 VSYNC 周期才能显示出去，所以显示延时会变大。当过一段时间，消费速率与生产速率相当以后，根据最小编号选取的算法，BufferQueue 又会变成 Double-Buffer 模式，减少显示延时。
* 对于生产端，应用对于 Buffer 属性的修改，如 width / height / format 等，是在下一次 dequeueBuffer() 时才改变。lazy-change 可以减少阻塞。
* 对于生产端，应用可以连续多次调用 IGraphicBufferProducer::dequeueBuffer()，获得多个 GraphicBuffer 进行并行图像处理，然后随即调用 IGraphicBufferProducer::queueBuffer() 将 GraphicBuffer 送回 BufferQueue中。
  - 这种应用特例在 Camera 软件中会应用到。例如连拍功能，先取出多个 GraphicBuffer 保存原始图像后，在后台进行并行图像处理。最后，完成处理的次序不一定是原来的顺序。
  - BufferQueue 需要处理这种情况，将送回来的 GraphicBuffer 按照编号重新排序，并在恰当的时机发送 onFrameAvailable() 消息。
* 对于消费端，对于 Frame，要严格保证 FIFO 的顺序。
* 对于消费端，要处理 Frame 超时等异常情况。这在视频播放中会遇到。及时抛弃超时的 Frame，可以减轻系统的负荷。

# 不同的视角的 GraphicBuffer 有限状态机

在前面的 BufferQueue 的协作图里面可以看到，GraphicBuffer 类有 4 种不同的应用场景，相对应的就会有 4 种不同的有限状态机。前面已经给出过。下图是在 Surface 中所使用的 GraphicBuffer 的对应于生命周期的有限状态机。可以看到，从 Producer-BufferQueue-Consumer 这三者来看，不同的视角(Viewport)有不同的有限状态机。最终就是使得在有限资源有限时间条件下，新帧能从生产端流动到消费端。

![GraphicBuffer Statemachine](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/framework-design/Graphic%20Buffer%20Statemachine%20Diagram%20-%20Producer-Queue-Consumer.svg)

