# 合成流程中的路径

这里简单回顾在 Main thread 中合成操作相关的消息处理流程，就是为了处理这 2 个消息：
* INVALIDATE
* REFRESH

```C++
00) SurfaceFlinger::run() - Main thread
  01) SurfaceFlinger::waitForEvent()
    02) MessageQueue::waitMessage()
      03) Looper::pollOnce(-1)
        04) epoll_wait
        04) MessageQueue::Handler::handleMessage(MessageType)
          05) case INVALIDATE:
          05) case REFRESH:
```

VSYNC 信号到达后，会进行 INVALIDATE 消息处理，接着会触发 REFRESH 消息，在下一个循环中会进行 REFRESH 消息处理：
```C++
00) MessageQueue::cb_eventReceiver() - Binder thread
  01) MessageQueue::eventReceiver(DISPLAY_EVENT_VSYNC)
    02) MessageQueue::Handler::dispatchInvalidate()
      03) Looper::sendMessage(MessageQueue::INVALIDATE)

next loop: - Main thread
case INVALIDATE:
03) Looper::pollOnce(-1)
  04) epoll_wait
  04) MessageQueue::Handler::handleMessage(MessageQueue::INVALIDATE)
    05) SurfaceFlinger::onMessageReceived(MessageQueue::INVALIDATE)
      06) SurfaceFlinger::handleMessageTransaction()
        07) SurfaceFlinger::handleTransaction()
          08) SurfaceFlinger::handleTransactionLocked()
      06) SurfaceFlinger::handleMessageInvalidate()
      06) SurfaceFlinger::signalRefresh()
        07) // sends REFRESH message immediately
        07) MessageQueue::refresh()
          08) MessageQueue::Handler::dispatchRefresh()
            09) Looper::sendMessage(MessageQueue::REFRESH)

next loop: - Main thread
case REFRESH:
03) Looper::pollOnce(-1)
  04) epoll_wait
  04) MessageQueue::Handler::handleMessage(MessageQueue::REFRESH)
    05) SurfaceFlinger::onMessageReceived(MessageQueue::REFRESH)
      06) SurfaceFlinger::handleMessageRefresh()
        07) SurfaceFlinger::preComposition();
        07) SurfaceFlinger::rebuildLayerStacks();
        07) SurfaceFlinger::calculateWorkingSet();
        07) SurfaceFlinger::beginFrame();
        07) SurfaceFlinger::prepareFrame();
        07) SurfaceFlinger::doDebugFlashRegions();
        07) SurfaceFlinger::doComposition();
        07) SurfaceFlinger::postFrame();
        07) SurfaceFlinger::postComposition();
```

在后面我们主要关注页面更新流程算法和合成操作流程算法：
* handleMessageInvalidate()
* handleMessageRefresh()

相应的操作流程见前面[ 两阶段流水线设计 ](../general-design/2-stage.md)章节的内容。

