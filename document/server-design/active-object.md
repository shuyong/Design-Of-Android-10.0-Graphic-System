# 基于 Active Object 模式进行设计

根据[[ Java多线程编程实战指南（设计模式篇）](http://www.broadview.com.cn/book/506)]一书第8章的描述，Active Object 模式的主要参与者和职责如下：
* Proxy: 负责对外暴露异步方法接口。
  + asyncService: 该异步方法负责创建与该方法相应的 MethodRequest 参与者实例，并将其提交给 Scheduler 参与者实例。该方法的返回值是一个 Future 参与者实例，客户端代码可以通过它获得异步方法对应的任务的执行结果。
* MethodRequest: 负责将客户端代码对 Proxy 实例的异步方法的调用封装为一个对象。该对象保留了异步方法的名称以及客户端代码传递的参数等上下文信息。它使得 Proxy 的异步方法的调用和执行分离成为可能。
  + call: 根据其所属 MethodRequest 实例所包含的上下文信息调用 Servant 实例的相应方法。
* ActivationQueue: 缓冲区，用于临时存储由 Proxy 的异步方法被调用时所创建的 MethodRequest 实例。
  + enqueue: 将 MethodRequest 实例放入缓冲区。
  + dequeue: 从缓冲区中取出一个 MethodRequest 实例。
* Scheduler: 负责将 Proxy 的异步方法所创建的 MethodRequest 实例存入其维护的缓冲区中，并根据一定的调度策略，对其维护的缓冲区中的 MethodRequest 实例进行执行。其调度策略可以根据实际需要来定，如 FIFO、LIFO 和根据 MethodRequest 中包含的信息所定的优先级等。
  + enqueue: 接收一个 MethodRequest 实例，并将其存入缓冲区。
  + dispatch: 反复地从缓冲区中取出 MethodRequest 实例进行执行。
* Servant: 负责 Proxy 所暴露的异步方法的具体实现。
  + method: 执行 Proxy 所暴露的异步方法对应的任务。
* Future: 负责存储和获取 Active Object 异步方法的执行结果。
  + get: 获取异步方法对应的任务的执行结果。
  + set: 设置异步方法对应的任务的执行结果。

Active Object 模式的主要参与者之间的关系的示意图如下：
![Active Object Class Diagram](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/server-design/Active%20Object%20Class%20Diagram.svg)

Active Object 模式的主要参与者的常见调用序列如下：
![Active Object Sequence Diagram](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/server-design/Active%20Object%20Sequence%20Diagram.svg)

对应到前面的 15 种消息和传递路径，就有如下表格的角色和职责划分。其中的 Servant 角色，除了 EventThread::Connection::postEvent() 运行在 mSFEventThread 中，剩下的所有的 Servant::method，也就是 SurfaceFlinger 中的方法，都运行在 Main thread 中。

| No. | Client                                            | Interface                                         | Proxy                                      | MethodRequest            | ActivationQueue::enqueue    | ActivationQueue::dequeue    | Scheduler::enqueue       | Scheduler::dispatch | Servant::method                                    | Future                          |
|:---:|---------------------------------------------------|---------------------------------------------------|--------------------------------------------|--------------------------|-----------------------------|-----------------------------|--------------------------|---------------------|----------------------------------------------------|---------------------------------|
| 1   | Surface::~Surface()                               | IGraphicBufferProducer::~IGraphicBufferProducer() | MonitoredProducer::~MonitoredProducer()    | cleanUpListMessage       | MessageQueue::postMessage() | MessageQueue::waitMessage() | Looper::sendMessage()    | Looper::pollOnce()  | SurfaceFlinger::mGraphicBufferProducerList.erase() | N/A                             |
| 2   | Surface::queueBuffer()                            | IConsumerListener::onFrameAvailable()             | EventThreadConnection::requestNextVsync()  | requestNextVsync         | VSyncRequest                | VSyncRequest                | pthread_cond_broadcast() | pthread_cond_wait() | DispSyncThread::addEventListener()                 | N/A                             |
| 3   | SurfaceComposerClient::createSurface()            | ISurfaceComposerClient::createSurface()           | Client::createSurface()                    | getNewTexture            | MessageQueue::postMessage() | MessageQueue::waitMessage() | Looper::sendMessage()    | Looper::pollOnce()  | RenderEngine::genTextures()                        | texture name                    |
| 4   | WindowManagerService::performEnableScreen()       | ISurfaceComposer::bootFinished()                  | SurfaceFlinger::bootFinished()             | bootFinished             | MessageQueue::postMessage() | MessageQueue::waitMessage() | Looper::sendMessage()    | Looper::pollOnce()  | SurfaceFlinger::setRefreshRateTo()                 | N/A                             |
| 5   | ScreenshotClient::capture()                       | ISurfaceComposer::captureScreen()                 | SurfaceFlinger::captureScreen()            | captureScreen            | MessageQueue::postMessage() | MessageQueue::waitMessage() | Looper::sendMessage()    | Looper::pollOnce()  | SurfaceFlinger::captureScreenImplLocked()          | GraphicBuffer                   |
| 6   | SurfaceComposerClient::enableVSyncInjections()    | ISurfaceComposer::enableVSyncInjections()         | SurfaceFlinger::enableVSyncInjections()    | enableVSyncInjections    | MessageQueue::postMessage() | MessageQueue::waitMessage() | Looper::sendMessage()    | Looper::pollOnce()  | MessageQueue::setEventThread()                     | mInjectVSyncs                   |
| 7   | SurfaceComposerClient::setActiveColorMode()       | ISurfaceComposer::setActiveColorMode()            | SurfaceFlinger::setActiveColorMode()       | setActiveColorMode       | MessageQueue::postMessage() | MessageQueue::waitMessage() | Looper::sendMessage()    | Looper::pollOnce()  | Display::setColorMode()                            | N/A                             |
| 8   | SurfaceComposerClient::setAllowedDisplayConfigs() | ISurfaceComposer::setAllowedDisplayConfigs()      | SurfaceFlinger::setAllowedDisplayConfigs() | setAllowedDisplayConfigs | MessageQueue::postMessage() | MessageQueue::waitMessage() | Looper::sendMessage()    | Looper::pollOnce()  | SurfaceFlinger::setAllowedDisplayConfigsInternal() | N/A                             |
| 9   | SurfaceComposerClient::setDisplayPowerMode()      | ISurfaceComposer::setPowerMode()                  | SurfaceFlinger::setPowerMode()             | setPowerMode             | MessageQueue::postMessage() | MessageQueue::waitMessage() | Looper::sendMessage()    | Looper::pollOnce()  | SurfaceFlinger::setPowerModeInternal()             | N/A                             |
|10   | main()                                            | (inside of process)                               | SurfaceFlinger::init()                     | requestDisplay           | MessageQueue::postMessage() | MessageQueue::waitMessage() | Looper::sendMessage()    | Looper::pollOnce()  | vrFlingerRequestDisplayCallback()                  | mVrFlingerRequestsDisplay       |
|11   | PriorityDump::dumpCritical()                      | PriorityDumper::dumpCritical()                    | SurfaceFlinger::dumpCritical()             | dumpProtoFromMainThread  | MessageQueue::postMessage() | MessageQueue::waitMessage() | Looper::sendMessage()    | Looper::pollOnce()  | SurfaceFlinger::dumpDrawingStateProto()            | layersProto                     |
|12   | Scheduler::Scheduler()                            | (inside of process)                               | SurfaceFlinger::setPrimaryVsyncEnabled()   | setPrimaryVsyncEnabled   | MessageQueue::postMessage() | MessageQueue::waitMessage() | Looper::sendMessage()    | Looper::pollOnce()  | SurfaceFlinger::setPrimaryVsyncEnabledInternal()   | N/A                             |
|13   | SurfaceFlinger::init()                            | (inside of process)                               | SurfaceFlinger::initializeDisplays()       | initializeDisplays       | MessageQueue::postMessage() | MessageQueue::waitMessage() | Looper::sendMessage()    | Looper::pollOnce()  | SurfaceFlinger::onInitializeDisplays()             | N/A                             |
|14   | SurfaceFlinger::signalLayerUpdate()               | (inside of process)                               | MessageQueue::invalidate()                 | MessageQueue::INVALIDATE | MessageQueue::Handler::dispatchInvalidate() | MessageQueue::Handler::handleMessage(INVALIDATE) | Looper::sendMessage() | Looper::pollOnce() | SurfaceFlinger::onMessageReceived(INVALIDATE) | N/A |
|15   | SurfaceFlinger::signalRefresh()                   | (inside of process)                               | MessageQueue::refresh()                    | MessageQueue::REFRESH    | MessageQueue::Handler::dispatchRefresh()    | MessageQueue::Handler::handleMessage(REFRESH)    | Looper::sendMessage() | Looper::pollOnce() | SurfaceFlinger::onMessageReceived(REFRESH)    | N/A |

在 SurfaceFlinger 程序里应用到的 Active Object 模式，是一个根据其应用特点高度定制的 Active Object 模式。在上表里的角色划分可知，有些类在不同的路径中有不同的角色。在 SurfaceFlinger 程序中整个 Active Object 模式的设计，大概有 3 种模型。

## VSYNC 消息的处理流程

VSYNC 消息的处理流程不是很明显的 Active Object 模式。但是从设计中也可以看到线程的相互间的影响和消息的串联触发关系。

| No. | Participant     | Class                 |
|:---:|-----------------|-----------------------|
| 0   | Client          | Surface               |
| 1   | Proxy           | EventThreadConnection |
| 2   | MethodRequest   | EventThread           |
| 3   | ActivationQueue | VSyncRequest          |
| 4   | Scheduler       | pthread               |
| 5   | Servant         | DispSyncThread        |
| 6   | Future          | N/A                   |

VSYNC 相关的类的关系图：
![Active Object Class Diagram - EventThread](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/server-design/Active%20Object%20Class%20Diagram%20-%20EventThread.svg)

VSYNC 相关的类的调用序列图：
![Active Object Sequence Diagram - EventThread](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/server-design/Active%20Object%20Sequence%20Diagram%20-%20EventThread.svg)

## 内部消息的处理流程

内部消息的处理流程不是很典型的 Active Object 模式。但是从设计中也可以看到消息的串联触发关系，以及批量处理事务(Commit transaction)的时机，也就是处理外部消息的时机，是在 VSYNC 消息到达之后。

| No. | Participant     | Class                 |
|:---:|-----------------|-----------------------|
| 0   | Client          | EventThreadConnection |
| 1   | Proxy           | MessageQueue          |
| 2   | MethodRequest   | MessageQueue::enum    |
| 3   | ActivationQueue | MessageQueue::Handler |
| 4   | Scheduler       | Looper                |
| 5   | Servant         | SurfaceFlinger        |
| 6   | Future          | N/A                   |

内部消息相关的类的关系图：
![Active Object Class Diagram - internal](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/server-design/Active%20Object%20Class%20Diagram%20-%20internal.svg)

内部消息相关的类的调用序列图：
![Active Object Sequence Diagram - internal](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/server-design/Active%20Object%20Sequence%20Diagram%20-%20internal.svg)

## 外部消息的处理流程

跨进程的外部消息的处理流程，也就是事务(transaction)处理流程，是异步处理流程，是最经典的 Active Object 模式。

我们以 getNewTexture 消息处理流程来看 Active Object 模式的设计。其中各类的角色划分如下：

| No. | Participant     | Class                 |
|:---:|-----------------|-----------------------|
| 0   | Client          | SurfaceComposerClient |
| 1   | Proxy           | Client                |
| 2   | MethodRequest   | getNewTexture         |
| 3   | ActivationQueue | MessageQueue          |
| 4   | Scheduler       | Looper                |
| 5   | Servant         | SurfaceFlinger        |
| 6   | Future          | texture name          |

外部消息相关的类的关系图如下：
![Active Object Class Diagram - SurfaceComposerClient](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/server-design/Active%20Object%20Class%20Diagram%20-%20SurfaceComposerClient.svg)

外部消息相关的类的调用序列图如下：
![Active Object Sequence Diagram - SurfaceComposerClient](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/server-design/Active%20Object%20Sequence%20Diagram%20-%20SurfaceComposerClient.svg)

captureScreen 消息处理流程也是经典的 Active Object 模式的设计。其中各类的角色划分如下：

| No. | Participant     | Class            |
|:---:|-----------------|------------------|
| 0   | Client          | ScreenshotClient |
| 1   | Proxy           | SurfaceFlinger   |
| 2   | MethodRequest   | captureScreen    |
| 3   | ActivationQueue | MessageQueue     |
| 4   | Scheduler       | Looper           |
| 5   | Servant         | SurfaceFlinger   |
| 6   | Future          | GraphicBuffer    |

其它关于 SurfaceComposerClient 类跨进程访问 SurfaceFlinger 内部方法的基本上都是经典的 Active Object 模式的设计。

