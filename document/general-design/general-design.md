# Android 的图形系统的总体设计
* * *

应用程序有自己的绘图节拍，它可以在任意时间向图形服务器提交绘图内容。图形服务器，surfaceflinger，基于 VSYNC，定期收集应用程序提交的绘图内容，合成并显示出去。于是 Android 图形系统里存在两类节拍：应用程序绘图节拍和图形显示节拍。所以 Android 图形系统的流水线是两阶段模型，每个阶段都是一个 Producer-Consumer 流程。应用程序渲染窗口，生产了新内容，交给了图形服务器。图形服务器收集多个程序提交的内容，基于 z-order，合成最终屏幕图像，实际上也生产了新内容，最终交给 HWC，显示到屏幕上。于是基于 Producer-Consumer 模式，每一类节拍对应一类 BufferQueue。

应用程序绘图节拍，每一对 Surface-Layer 存在一个 BufferQueue。图形显示节拍，只在 surfaceflinger service 内部存在，一个显示设备对应一个 BufferQueue。

图形显示节拍，基于 VSYNC 设计。所以首先要理解 VSYNC 模块的设计。

# [VSYNC 模块的设计](VSYNC.md)

# [基于接口(interface)的对称设计](symmetrical-design.md)

# [两阶段(2-stage)设计](2-stage.md)

# 多进程协作的设计

# 小结

图形和媒体管道可以认为是一条挂载着多个摆弄缓冲区(Buffer)的设备的流水线。同时，Android 图形系统根据功能域的划分，将整个图形处理的流水线划分为两阶段(2-stage)，并且可以组合。每个阶段的核心，就是 BufferQueue。从应用软件生产显示内容(Content)到 SurfaceFlinger 消费内容，通常的操作，就是一个两阶段的生产-消费循环。

生产循环，将一个窗口某个时段的内容从应用软件，也就是客户端(Client side)，送达服务端(Server side)，也就是 SurfaceFlinger。从 Surface 类开始，最后到达 Layer 类。每一对 Surface-Layer 对应一个 BufferQueue。因为系统中有多个应用软件，每个软件有多个 Surface。每个进程/线程不可能协调一致，所以每个承载着内容(Content)的 GraphicBuffer，也就是帧(Frame)的到达，是异步和随机的。

消费循环，将所有在显示区域能看到内容的 Layer 进行合成。从 DisplayDevice 类开始，最后到达 DisplaySurface 接口(Interface)的实现。每个显示设备对应一个 BufferQueue。这阶段的工作节拍就是定时的 VSYNC 信号。FramebufferSurface 类是 DisplaySurface 接口(Interface)的一个实现。LCD 显示设备就是用 DisplayDevice-FramebufferSurface 这一对生产-消费关系来表达。

因为内容的生产是随机和异步的，而内容消费，也就是显示设备的工作原理是基于 VSYNC 信号的，是定时的。如果生产和消费在一个循环里，则有可能内容的到达会打断合成操作，从而造成显示内容的撕裂；也有可能耗时的合成操作阻塞了内容的到达，从而阻塞了生产。所以为了显示顺滑，将生产-消费分成了两个阶段。同时为了显示更顺滑，生产循环和消费循环的工作节拍要用 VSYNC 信号协调，如同大马路上的红绿灯要协调一样。这就是编舞者(Choreographer)框架要解决的问题。

为了在多核异构系统中提高并行的效率，Android 系统引入了显式同步机制。在图形系统中，主要有这3类同步栅栏(Fence):
* 获取栅栏(Acquire fence)。每层(Layer)一个。代表着该 GraphicBuffer 承载的内容生成结束，HWComposer 可以进行读取。
* 释放栅栏(Release fence)。每层(Layer)一个。代表着 HWComposer 对内容读取完毕，该 GraphicBuffer 可以交给应用进行写入。
* 退出栅栏(Retire fence)。整个图形框架就一个。代表着当前合成完的 framebuffer 已经由 HWComposer 显示到屏幕上。

所以合成服务器(HWComposer)是内容汇总的地方，同时也是同步栅栏(Fence)集中处理的地方。

| No. | Fence         | Producer                                               | Consumer                                                |
|:---:|---------------|--------------------------------------------------------|---------------------------------------------------------|
| 1   | Acquire fence | eglSwapBuffers() / before ANativeWindow::queueBuffer() | HWComposer::commit() / hwc_composer_device_1::set()     |
| 2   | Release fence | HWComposer::commit() / hwc_composer_device_1::set()    | eglSwapBuffers() / after ANativeWindow::dequeueBuffer() |
| 3   | Retire  fence | HWComposer::commit() / hwc_composer_device_1::set()    | DispSync::addPresentFence()                             |

所以从总的架构看，客户端(Client side)和服务端(Server side)互为 Producer-Consumer 关系。

# 参考文件
1. [The Android Graphics microconference](https://lwn.net/Articles/569704/)
1. [Graphics](https://source.android.com/devices/graphics/index.html)


