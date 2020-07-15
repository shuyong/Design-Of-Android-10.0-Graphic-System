# SurfaceFlinger 中的消息与传递路径

SurfaceFlinger 程序的总体设计，是按照 Active Object 模式进行设计的。其中的 MessageQueue 类，就是内部消息汇总和分发的地方，也就是开发者常说的消息泵(message pump)。

MessageQueue 相关各类的关系组合图如下：
![MessageQueue Class Diagram](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/server-design/services_surfaceflinger_Scheduler_MessageQueue%20Class%20Diagram.svg)

SurfaceFlinger 程序的主线程，是一个传统的消息处理循环：
```C++
00) SurfaceFlinger::run()
  01) SurfaceFlinger::waitForEvent()
    02) MessageQueue::waitMessage()
      03) Looper::pollOnce(-1)
        04) epoll_wait
        04) MessageQueue::Handler::handleMessage(MessageType)
```

循环代码里表面上只处理 2 个消息：
* INVALIDATE: 代表 VSYNC 信号到达。主线程先批量处理客户端的事务(transaction)请求，再锁定(latch)各层的 Buffer。最后给自己发送一个 REFRESH 消息。这样做主要是为了使得功能区域的边界清晰，所以把处理流程分段。这是应用了 Command Pattern。
* REFRESH: 主线程进行实际的合成工作。

但是，在 Android 提供的 Looper 类中，都是用 Message-MessageHandler 绑定方式的消息处理机制，在每次循环，在 Looper::pollOnce() 方法中得到处理。本质上，继承至 MessageHandler 接口的 MessageBase 与 MessageQueue::Handler 都是用这个机制在主线程的循环里处理消息。

INVALIDATE / REFRESH 这 2 个消息，用的是 MessageQueue::Handler 处理路径。

而跨进程的消息，用的是 MessageBase 处理路径。MessageBase 接口(Interface)的具体实现为 LambdaMessage，用于其它类向 SurfaceFlinger 发送消息。消息类型有:
* bootFinished
* getNewTexture
* requestDisplay
* setActiveColorMode
* enableVSyncInjections
* setPrimaryVsyncEnabled
* initializeDisplays
* setPowerMode
* dumpProtoFromMainThread
* captureScreen
* setAllowedDisplayConfigs
* cleanUpListMessage

MessageBase 接口(Interface)的抽象处理流程的序列图如下：
![MessageBase Sequence Diagram](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/server-design/services_surfaceflinger_Scheduler_MessageBase%20Sequence%20Diagram.svg)

如果我们将[ Listener 消息的流动 ](../framework-design/bufferqueue-design.md#Listener-消息的流动)与[ VSYNC 信号的流动 ](../general-design/VSYNC.md#VSYNC-信号的流动)的内容串连起来，就得到 VSYNC 信号的申请到发送给 SurfaceFlinger 的调用序列。

```C++
Step 1 - on Binder thread of surfaceflinger
05) FrameAvailableListener::onFrameAvailable() == BufferQueueLayer::onFrameAvailable()
  06) SurfaceFlinger::signalLayerUpdate()
    07) MessageQueue::invalidate()
      08) EventThreadConnection::requestNextVsync()
        09) EventThread::requestNextVsync()
          10) ResyncCallback <- Scheduler::makeResyncCallback()
            11) SurfaceFlinger::getVsyncPeriod()
          10) connection->vsyncRequest = VSyncRequest::Single;
  06) BufferLayerConsumer::onBufferAvailable()

Step 2 - on mSFEventThread
00) EventThread::threadMain()
  01) // Wait for client registration.
  01) Condition::wait(lock)
  01) vsyncRequested |= connection->vsyncRequest != VSyncRequest::None;
  01) VSyncSource::setVSyncEnabled(enable) == DispSyncSource::setVSyncEnabled(enable)
    02) DispSync::addEventListener() | DispSync::removeEventListener()
      03) Thread interface => DispSyncThread implement
      03) DispSyncThread::addEventListener() | DispSyncThread::removeEventListener()

Step 3 - on DispSyncThread
00) DispSyncThread::threadLoop()
  01) DispSyncThread::gatherCallbackInvocationsLocked()
  01) DispSyncThread::fireCallbackInvocations()
    02) DispSync::Callback::onDispSyncEvent(nsecs_t when)
      03) VSyncSource::onDispSyncEvent(nsecs_t when) == DispSyncSource::onDispSyncEvent(nsecs_t when) -> sfVsyncSrc
        04) VSyncSource::Callback::onVSyncEvent(nsecs_t timestamp) == EventThread::onVSyncEvent(nsecs_t timestamp) -> mSFEventThread
          05) makeVSync()
            06) DisplayEventReceiver::Event event;
            06) event.header = {DisplayEventReceiver::DISPLAY_EVENT_VSYNC, displayId, timestamp};
            06) event.hotplug.connected = connected;
          05) mPendingEvents.push_back()

Step 4 - on mSFEventThread
00) EventThread::threadMain()
  01) // Wait for event request.
  01) Condition::wait(lock)
  01) event = mPendingEvents.front()
  01) EventThread::dispatchEvent()
    02) IDisplayEventConnection::postEvent() == EventThreadConnection::postEvent()
      03) DisplayEventReceiver::sendEvents()

Step 5 - on Main Thread: MessageQueue as server side
00) MessageQueue::cb_eventReceiver()
  01) MessageQueue::eventReceiver()
    02) MessageQueue::Handler::dispatchInvalidate()
      03) Looper::sendMessage(MessageQueue::INVALIDATE)

Step 6 - on Main Thread: SurfaceFlinger as client side
00) SurfaceFlinger::run()
  01) SurfaceFlinger::waitForEvent()
    02) MessageQueue::waitMessage()
      03) Looper::pollOnce(-1)
        04) epoll_wait
        04) MessageQueue::Handler::handleMessage(MessageQueue::INVALIDATE)
          05) SurfaceFlinger::onMessageReceived(MessageQueue::INVALIDATE)

Step 7 - on Main Thread: SurfaceFlinger self-excited
00) SurfaceFlinger::run()
  01) SurfaceFlinger::waitForEvent()
    02) MessageQueue::waitMessage()
      03) Looper::pollOnce(-1)
        04) epoll_wait
        04) MessageQueue::Handler::handleMessage(MessageQueue::REFRESH)
          05) SurfaceFlinger::onMessageReceived(MessageQueue::REFRESH)
```

下面是相应的调用序列图：
![SurfaceFlinger Sequence Diagram - VSYNC](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/server-design/services_surfaceflinger_SurfaceFlinger%20Sequence%20Diagram%20-%20VSYNC.svg)

最终，加上 VSYNC 信号的处理，在 surfaceflinger 程序中有下列类型的消息：
* 在 Client / Server 模式下，跨进程的消息，用 LambdaMessage 处理路径。从 Binder thread 向 Main thread 发送。
* 由 queueBuffer() 操作触发的 requestNextVsync() 消息。从 Binder thread 向 EventThread 发送。最终还是到达 SurfaceFlinger 做处理。
* 由 EventThread 向 Main thread 发送的 VSYNC(INVALIDATE) 消息。用 MessageQueue::Handler 处理路径。
* Main thread 自己向自己发送的消息：REFRESH。用 MessageQueue::Handler 处理路径。

由前面的章节分析可知，在 SurfaceFlinger 中，在 Client / Server 模式下，跨越进程/线程的接口(Interface)的调用路径如下：

| No. |   Client's Class      | Interface               | Server's Class   | Running on     | Cross type    |
|:---:|-----------------------|-------------------------|------------------|----------------|---------------|
| 1   | Surface               | IGraphicBufferProducer  | BufferQueueLayer | Binder thread  | Cross process |
| 2   | SurfaceComposerClient | ISurfaceComposerClient  | Client           | Binder thread  | Cross process |
| 3   | ComposerService       | ISurfaceComposer        | SurfaceFlinger   | Binder thread  | Cross process |
| 4   | SurfaceFlinger        | IDisplayEventConnection | EventThread      | mSFEventThread | Cross thread  |

SurfaceFlinger 主线程中的代码的最主要的功能就是合成 Layer，运行在 Main thread 中。上述运行在其它线程中的主动对象和 SurfaceFlinger 对象就是通过下面 15 种消息和传递路径进行交互：

| No. | Caller                                       | Callee                                     | Message                         | final SurfaceFlinger method        |
|:---:|----------------------------------------------|--------------------------------------------|---------------------------------|------------------------------------|
|  1  | Surface::~Surface()                          | MonitoredProducer::~MonitoredProducer()    | cleanUpListMessage              | mGraphicBufferProducerList.erase() |
|  2  | IGraphicBufferProducer::queueBuffer()        | BufferQueueLayer::onFrameAvailable()       | EventThread::requestNextVsync() | signalLayerUpdate()                |
|  3  | ISurfaceComposerClient::createSurface()      | BufferLayer::BufferLayer()                 | getNewTexture                   | RenderEngine::genTextures()        |
|  4  | ISurfaceComposer::bootFinished()             | SurfaceFlinger::bootFinished()             | bootFinished                    | bootFinished()                     |
|  5  | ISurfaceComposer::captureScreen()            | SurfaceFlinger::captureScreen()            | captureScreen                   | captureScreenImplLocked()          |
|  6  | ISurfaceComposer::enableVSyncInjections()    | SurfaceFlinger::enableVSyncInjections()    | enableVSyncInjections           | enableVSyncInjections()            |
|  7  | ISurfaceComposer::setActiveColorMode()       | SurfaceFlinger::setActiveColorMode()       | setActiveColorMode              | Display::setColorMode()            |
|  8  | ISurfaceComposer::setAllowedDisplayConfigs() | SurfaceFlinger::setAllowedDisplayConfigs() | setAllowedDisplayConfigs        | setAllowedDisplayConfigsInternal() |
|  9  | ISurfaceComposer::setPowerMode()             | SurfaceFlinger::setPowerMode()             | setPowerMode                    | setPowerModeInternal()             |
| 10  | main()                                       | SurfaceFlinger::init()                     | requestDisplay                  | mVrFlingerRequestsDisplay          |
| 11  | PriorityDumper::dumpCritical()               | SurfaceFlinger::doDump()                   | dumpProtoFromMainThread         | dumpDrawingStateProto()            |
| 12  | Scheduler::Scheduler()                       | SurfaceFlinger::setPrimaryVsyncEnabled()   | setPrimaryVsyncEnabled          | setPrimaryVsyncEnabledInternal()   |
| 13  | SurfaceFlinger::init()                       | SurfaceFlinger::initializeDisplays()       | initializeDisplays              | onInitializeDisplays()             |
| 14  | SurfaceFlinger::signalLayerUpdate()          | MessageQueue::eventReceiver(VSYNC)         | MessageQueue::INVALIDATE        | onMessageReceived(INVALIDATE)      |
| 15  | SurfaceFlinger::signalRefresh()              | MessageQueue::refresh()                    | MessageQueue::REFRESH           | onMessageReceived(REFRESH)         |

注：ISurfaceComposer::bootFinished() 是在 JAVA 层代码被调用，由 WindowManagerService::performEnableScreen() 调用。

其中与合成操作相关的消息是串联激发的：

| No. | Caller                                | Message                         | Message                  |
|:---:|---------------------------------------|---------------------------------|--------------------------|
| 1   | IGraphicBufferProducer::queueBuffer() | EventThread::requestNextVsync() | MessageQueue::INVALIDATE |
| 2   | SurfaceFlinger::signalLayerUpdate()   | MessageQueue::INVALIDATE        | MessageQueue::REFRESH    |

* 当 Surface 调用 queueBuffer() 提交新帧的时候，SurfaceFlinger 不是立刻将新帧绘制到显示屏幕上，而是等待下一个 VSYNC 信号到达才开始合成和显示操作。在 Binder thread 中，SurfaceFlinger::signalLayerUpdate() 触发的是 EventThread::requestNextVsync() 消息，影响 mSFEventThread。
* 当下一个 VSYNC 信号到达，mSFEventThread 发送消息给 MessageQueue。在 Binder thread，MessageQueue::eventReceiver() 收到 DISPLAY_EVENT_VSYNC 消息，就给 Main thread 发送 INVALIDATE 消息。
* 在 Main thread，在 SurfaceFlinger 的下一个消息处理循环，收到 INVALIDATE 消息，先调用 handleTransaction() 处理客户端的事务请求，再调用 handleMessageInvalidate() 锁定各层当前的 GraphicBuffer，最后调用 signalRefresh() 给自己发送 REFRESH 消息。
* 在 Main thread，在 SurfaceFlinger 的下一个消息处理循环，收到 REFRESH 消息，就立刻调用 handleMessageRefresh() 对已经锁定的 Layer 进行实际的合成和显示操作。
* 上述的执行步骤，是一个比较典型的 Command Pattern。具体流程见前面[ 两阶段流水线设计 ](../general-design/2-stage.md)章节的内容。

