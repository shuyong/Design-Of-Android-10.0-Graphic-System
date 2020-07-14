# SurfaceFlinger 模块划分
* * *

因为 surfaceflinger 程序本身的代码很复杂，依赖很多。为便于管理，开发者将其进行了模块划分。子模块以子目录存在，有：
* CompositionEngine
* DisplayHardware
* Effects
* EventLog
* Scheduler
* TimeStats
* layerproto
* sysprop

需要重点关注的是 CompositionEngine / DisplayHardware / Scheduler 这 3 个模块。其中 Scheduler 模块已经在前面的[ VSYNC 模块的设计 ](../general-design/VSYNC.md)章节描述过，这里不再重复。

surfaceflinger 程序所依赖的库，需要重点关注的是：
* renderengine

另外，SurfaceFlinger 类极其复杂，参与开发提交代码的人都超过 20 人之多。所以开发者也对其进行了内部模块划分，以便于理解和维护：
* main loop / main thread
* IBinder interface implement
* ISurfaceComposer interface implement
* DeathRecipient interface implement
* RefBase interface implement
* HWC2::ComposerCallback interface / HWComposer::EventHandler interface implement
* Message handling
* Transactions
* Layer management
* Boot animation
* Properties
* EGL
* Display and layer stack management
* H/W composer
* Compositing
* Display management
* VSync
* Display identification
* Debugging & dumpsys
* VrFlinger
* Attributes
* Feature prototyping
* Scheduler

下面将对一些重点的模块进行分析。

# 显示硬件抽象

这个模块主要是定义并实现显示硬件(Display Hardware)接口：
* HWComposer - 封装 Hardware Composer 的接口与实现。
* DisplaySurface - 实现在合成引擎中定义的显示后端接口。
* PowerAdvisor - 电源管理顾问。屏幕开关，刷新率变更，需要这个接口。

HWComposer 关系组合图如下：
![HWComposer Component Diagram](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/server-design/services_surfaceflinger_DisplayHardware_ComposerHal%20Component%20Diagram.svg)

其中的封装层次相当多。从 C++ namespace 划分：
* android::hardware::graphics::composer - HIDL 接口
  + IComposer
  + IComposerClient
  + IComposerCallback
* android::Hwc2 - 调用 HIDL 接口的接口
  + Composer
* HWC2 - Hardware Composer V2 的抽象
  + Device
  + ComposerCallback
* android - SurfaceFlinger 所在的名字空间
  + HWComposer

显示后端接口的实现，指的是 FramebufferSurface & VirtualDisplaySurface 这两个类。它们即实现了在 framework/gui 中定义的 ConsumerBase 接口，表明它们是 BufferQueue 的消费端，也实现了在合成引擎中定义的显示后端接口 DisplaySurface，表明了它们是图形流水线第二阶段的 BufferQueue 的消费端。这些内容已经在前面的[ 两阶段流水线的设计 ](../general-design/2-stage.md)的章节已讨论，这里不再重复。

PowerAdvisor 关系组合图如下，最终调用了 HIDL 接口：
![PowerAdvisor 接口的关系组合图](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/server-design/services_surfaceflinger_DisplayHardware_PowerAdvisor%20Class%20Diagram.svg)

# 合成引擎

在合成引擎里定义并实现很多接口：
* CompositionEngine
* Display
* DisplaySurface
* LayerFE
* Layer
* Output
* OutputLayer
* RenderSurface

如果以 CompositionEngine 为中心看关系组合图：
![CompositionEngine Component Diagram](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/server-design/services_surfaceflinger_CompositionEngine_include_compositionengine_CompositionEngine%20Component%20Diagram.svg)

在图中可以看到：
* 合成引擎 CompositionEngine 最终使用硬件合成器 HWComposer 与渲染引擎 RenderEngine 这两种方式完成合成操作。
* 显示设备 DisplayDevice 管理着 Display 与 Layer，并将两者联系在一起。
* Layer 是图形流水线第一阶段的消费端。RenderSurface 是第二阶段的生产端，由 Display 创建。DisplaySurface 是第二阶段的消费端。
* RenderSurface-DisplaySurface 流水线围绕合成引擎 CompositionEngine 设计。

上图的关系太杂乱。如果我们以 Layer 的概念为中心重新整理关系组合图，则有下图：
![Layer Component Diagram](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/server-design/services_surfaceflinger_Layer%20Component%20Diagram.svg)

前面说过，Surface-Layer 是图形流水线的第一阶段。但是 Layer 的概念很通用，在很多地方也会使用到。于是设计者对 Layer 的概念做了细分。从 C++ namespace 划分：
* android::compositionengine::LayerFE - Layer 前端。属于合成引擎的概念。定义 2 个方法，功能如其名：
  + latchCompositionState
  + onLayerDisplayed
* android::Layer - 就是 Surface-Layer 流水线中的 Layer，SurfaceFlinger 所使用的 Layer。继承自 LayerFE，表明对于第二阶段的流水线，android::Layer 是前端。它的主要实现有：
  + BufferLayer
    - BufferQueueLayer
    - BufferStateLayer
  + ColorLayer
  + ContainerLayer
* android::compositionengine::OutputLayer - 输出设备相关的 Layer，包含与输出设备相关的合成状态。属于合成引擎的概念。OutputLayer 将下面 3 种 Layer 绑定在一起：
  + android::Layer
  + android::compositionengine::Layer
  + HWC2::Layer
* android::compositionengine::Layer - 输出设备无关的 Layer。属于合成引擎的概念。主要是对于前端 Layer，也就是 android::Layer，包含与输出设备无关的合成状态。
* HWC2::Layer - HWComposer 所使用的 Layer。属于硬件底层的概念。

于是就有了下面的一些概念的相互关系：
* OutputLayer 的所有者是 Output，并且 OutputLayer 由 Output 提供。
* Display 是一种 Output，对外提供 RenderSurface。
* 最终是 DisplayDevice 创建了 Display。
* android::Layer 代表着应用程序生成的帧，由 WindowManager 管理生成 z-order 关系。RenderSurface 获取的 Buffer 插入到 z-order 的最底层。这样经过 CompositionEngine 的合成操作，该 Buffer 就承载了所有可见帧的合成结果。

# 渲染引擎

渲染引擎代码位于 frameworks/native/libs/renderengine/ 目录。关系组合图如下：

![RenderEngine Component Diagram](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/server-design/renderengine_include_renderengine_RenderEngine%20Component%20Diagram.svg)

渲染引擎由 OpenGLES 来实现。Surface-Layer 所拥有的 Buffer，就绑定为 GL Texture，由 GPU 做渲染。这种实现具有更大的灵活性。在 C/S 两端都会应用到。

# 内部模块

本文关注的 SurfaceFlinger 类的内部模块如下：
* main loop / main thread - 合成和显示功能所在的线程。其它模块大都运行在其它线程，都是围绕这个主功能进行配置和控制。
* ISurfaceComposer interface - 在[ 基于 Client / Server 模型的对称设计 ](../general-design/symmetrical-design.md#基于-Client--Server-模型的对称设计)章节中已讨论。不再重复。
* HWC2::ComposerCallback interface - VSYNC Path。在[ VSYNC 信号的生成与传播 ](../general-design/VSYNC.md#VSYNC-信号的生成与传播)章节中已讨论。不再重复。
* Message handling - 主线程接收消息并进行处理的代码。以及其它线程向主线程发送消息的工具类：LambdaMessage & MessageQueue。发送消息和处理消息的流程在后面的章节讨论。
  + 异常处理 - 如果合成速度太慢，在一次 VSYNC 周期中不能完成合成动作，会造成 VSYNC 信号积压。在下一次事件处理循环中，MessageQueue::eventReceiver() 方法会取出事件队列中所有 VSYNC 信号，但只给 SurfaceFlinger 发送一次 VSYNC 信号，其余都抛弃。这样就避免了多余的合成操作。
* Compositing - 在[ 两阶段流水线的设计 ](../general-design/2-stage.md)的章节中已讨论。不再重复。
* VSync - 获取和设置 VSYNC 周期。属于 VSYNC Path 的一部分。
* Scheduler - 在[ VSYNC 模块的设计 ](../general-design/VSYNC.md)章节中已讨论。不再重复。

