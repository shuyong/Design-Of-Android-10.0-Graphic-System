# Android 的图形系统的总体设计
* * *

应用程序有自己的绘图节拍，它可以在任意时间向图形服务器提交绘图内容。图形服务器，surfaceflinger，基于 VSYNC，定期收集应用程序提交的绘图内容，合成并显示出去。于是 Android 图形系统里存在两类节拍：应用程序绘图节拍和图形显示节拍。所以 Android 图形系统的流水线是两阶段模型，每个阶段都是一个 Producer-Consumer 流程。应用程序渲染窗口，生产了新内容，交给了图形服务器。图形服务器收集多个程序提交的内容，基于 z-order，合成最终屏幕图像，实际上也生产了新内容，最终交给 HWC，显示到屏幕上。于是基于 Producer-Consumer 模式，每一类节拍对应一类 BufferQueue。

应用程序绘图节拍，每一对 Surface-Layer 存在一个 BufferQueue。图形显示节拍，每一对 RenderSurface-DisplaySurface，对应一个显示设备，存在一个 BufferQueue，只在 surfaceflinger service 内部存在，

图形显示节拍，基于 VSYNC 设计。所以首先要理解 VSYNC 模块的设计。

# [VSYNC 模块的设计](VSYNC.md)

# [基于接口(interface)的对称设计](symmetrical-design.md)

# [两阶段(2-stage)流水线设计](2-stage.md)

# 多进程协作的设计

Android 图形系统，原本是简单的 C/S 架构，surfaceflinger 是图形合成显示服务器。引入 HIDL 以后，情况变得复杂，变成了多进程协作式设计。surfaceflinger 更像是居中调度的中间件，不再直接管理硬件，而是委托 HIDL 层的服务器进行管理。

Android 图形系统通用的服务器位于 /system/bin/ 目录：
* gpuservice - GPU 状态服务器，graphicsenv 库使用。最终是应用程序所使用的 OpenGLES 和 Vulkan 使用到。
* surfaceflinger - 图形合成显示服务器。

Android 图形系统的 HIDL 服务器位于 /vendor/bin/hw/ 目录：
* android.hardware.graphics.allocator@2.0-service
* android.hardware.graphics.composer@2.1-service

Android 图形系统的 HIDL 服务器动态加载的模块位于 /system/lib64/ 目录：
* android.frameworks.displayservice@1.0.so
* android.frameworks.vr.composer@1.0.so
* android.hardware.graphics.allocator@2.0.so
* android.hardware.graphics.bufferqueue@1.0.so
* android.hardware.graphics.common@1.0.so
* android.hardware.graphics.common@1.1.so
* android.hardware.graphics.composer@2.1.so
* android.hardware.graphics.composer@2.2.so
* android.hardware.graphics.mapper@2.0.so
* android.hardware.graphics.mapper@2.1.so

厂商提供的本硬件定制的动态加载的模块位于 /system/lib64/vndk-*/ 目录。总的说来，HIDL 服务器管理使用 HAL 模块，是实际与硬件驱动打交道的地方。

对于应用而言，以前是向 surfaceflinger 服务器申请 GraphicBuffer，后来是向 android.hardware.graphics.allocator@2.0-service 申请。同样的，原本是 surfaceflinger 服务器管理 HWC 驱动接口并进行合成操作，后来是由 android.hardware.graphics.composer@2.1-service 进行实际的合成操作。

由此看，应用绘制画面并显示的动作，涉及到很多次跨进程的操作。于是 Android 图形系统极其依赖 OS 的 Soft-Realtime 特性。所以 Android Linux Kernel 打过 Realtime 补丁(Preempt RT patches)，打开了 “PREEMPT” 特性，是完全的可抢占式内核(fully preemptible (real-time) kernel)。进程切换时间，大多数情况 < 1ms，但极限不会 < 0.1ms。

但是，函数调用切换的时间是大大小于进程切换时间好几个数量级。加上 HIDL 的引入，调用 Binder Interface 会增加调用延时。这时候需要从全局考虑多进程协作设计的优劣。

Android 应用程序，最耗时的动作是对大尺寸图像的渲染。假如渲染一帧需要 5ms，而进程切换回来需要 0.5ms，则应用还需要等待 4.5ms。在这 4.5ms 中，应用可以切换到其它线程上做很多事情了。

所以，Android 图形系统是针对多核异构式的硬件系统进行设计。通过多进程多线程模式，使得 GPU / HWC / codec 这些独占硬件能批量地满负荷运行。虽然接口调用有损耗，但系统总体运行效率是大大提高了。

另外，分布式多进程协作模式，使得系统更容易升级了，并且系统健壮性更好。

# 小结

图形和媒体管道可以认为是一条挂载着多个摆弄缓冲区(Buffer)的设备的流水线。同时，Android 图形系统根据驱动节拍的划分，将整个图形处理的流水线划分为多个阶段(stage)，并且可以组合。每个阶段的核心，就是 BufferQueue。从应用软件生产显示内容(Content)到 SurfaceFlinger 消费内容，通常的操作，就是一个两阶段(2-stage)的生产-消费循环。

生产循环，将一个窗口某个时段的内容从应用软件，也就是客户端(Client side)，送达服务端(Server side)，也就是 SurfaceFlinger。从 Surface 类开始，最后到达 Layer 类。每一对 Surface-Layer 对应一个 BufferQueue。因为系统中有多个应用软件，每个软件有多个 Surface。每个进程/线程不可能协调一致，所以每个承载着内容(Content)的 GraphicBuffer，也就是帧(Frame)的到达，是异步和随机的。

消费循环，将所有在显示区域能看到内容的 Layer 进行合成。从 RenderSurface 接口(Interface)开始，最后到达 DisplaySurface 接口(Interface)。每个显示设备对应一个 BufferQueue。这阶段的工作节拍就是定时的 VSYNC 信号。FramebufferSurface 类是 DisplaySurface 接口(Interface)的一个实现。LCD 显示设备就是用 RenderSurface-FramebufferSurface 这一对生产-消费关系来表达。

因为内容的生产是随机和异步的，而内容消费，也就是显示设备的工作原理是基于 VSYNC 信号的，是定时的。如果生产和消费在一个循环里，则有可能内容的到达会打断合成操作，从而造成显示内容的撕裂；也有可能耗时的合成操作阻塞了内容的到达，从而阻塞了生产。所以为了显示顺滑，将生产-消费分成了两个阶段。同时为了显示更顺滑，生产循环和消费循环的工作节拍要用 VSYNC 信号协调，如同大马路上的红绿灯要协调一样。这就是编舞者(Choreographer)框架要解决的问题。

为了在多核异构系统中提高并行的效率，Android 系统引入了显式同步机制。在图形系统中，主要有这3类同步栅栏(Fence):
* 获取栅栏(Acquire fence)。每层(Layer)一个。代表着该 GraphicBuffer 承载的内容生成结束，HWComposer 可以进行读取。
* 释放栅栏(Release fence)。每层(Layer)一个。代表着 HWComposer 对内容读取完毕，该 GraphicBuffer 可以交给应用进行写入。
* 退出栅栏(Retire fence)。整个图形框架就一个。代表着当前合成完的 framebuffer 已经由 HWComposer 显示到屏幕上。该信号在新版本中改名为显示栅栏(Present fence)。

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


