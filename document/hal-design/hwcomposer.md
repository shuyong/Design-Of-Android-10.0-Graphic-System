# hwcomposer 模块
* * *

# 背景知识

从前面的文章我们了解到，Android 中的图形服务器(SurfaceFlinger)的最主要功能就是合成窗口内容，并将其送到显示设备上进行显示。窗口(Window)在 Android 里的专有对象就是表面(Surface)。SurfaceFlinger 实际上是使用 OpenGL(GPU) 来合成不同的屏幕和表面(Surface)。

在最初的描述和设计中，硬件合成器(Hardware Composer / HWC)是一个 SurfaceFlinger 中的插件小模块，它帮助将一些表面合成的负载转移到专门的显示硬件上。通常，这只是一个2D GPU 或 2D 硬件合成器(2D Hardware Composer)。但有些硬件方案在图形部分也有特殊的“覆盖层平面(Overlay planes)”可以使用。这在几个方面减轻了 GPU 负载，潜在地提高了性能。

因此可以认为，Android 硬件合成器(Hardware Composer / HWC)是一个抽象的 2D 合成器的软件库。它可以将 GPU 负责的一部分屏幕和表面(Surface)合成转移到它这里来，以便减轻 GPU 的负荷。基于 HWC HAL，芯片厂商可以实现他们专有的 2D 合成器的软件库，其中集成了芯片平台的 2D 硬件合成器。

如今，硬件合成器(Hardware Composer / HWC)对象已经是整个显示子系统的硬件抽象实现，是所有 Android 合成操作(composition)的核心，并且接管了显示设备的管理。SurfaceFlinger 可以将某些合成工作委托给 Hardware Composer 对象，以分担 GPU 上的工作量。SurfaceFlinger 只是充当另一个 OpenGL ES 客户端。有时，在 SurfaceFlinger 将一个或两个缓冲区合成到第三个缓冲区中的过程中，它会使用 OpenGL ES。 

Hardware Composer 对象则进行另一半的工作。首先是和 SurfaceFlinger 协商确定使用可用硬件合成缓冲区的最有效方法：
* Overlay planes
* GPU composition
* Blit / 2D engine

接着，SurfaceFlinger 和 HWC 各自用最有效方法完成合成，将最终合成帧(Frame)送到显示设备上。这样的工作模式使得合成的功耗比通过 GPU 执行所有计算更低。

另外，Hardware Composer 对象必须支持事件，其中之一是 VSYNC 事件，另一个是支持即插即用 HDMI 的热插拔事件。 

在Android 6.0 时，还是 HWComposer 1.1 的接口。这个版本在性能上，在同步约定上还有不完善的地方。在 Android 7.0以后，则使用更新的 HWComposer 2.0。其中接口约定上最大的升级就是栅栏(Fence)从推测性的变为非推测性的。一些简单的讨论见下面。更多的信息参见文档[[Implementing the Hardware Composer HAL](https://source.android.com/devices/graphics/implement-hwc)]和[[Hardware Composer 2.0](https://blog.linuxplumbersconf.org/2016/ocw/system/presentations/4185/original/LPC%20HWC%202.0%20&%20drm_hwcomposer%20.pdf)]。

# 显示系统相关设备接口

Android 的图形系统的底层会涉及到以下设备： 
* 帧缓冲设备(framebuffer_device_t)，声明于 fb.h，继承自设备接口(hw_device_t)。就是对显示设备(如 LCD / HDMI)的抽象描述。一台 Android 设备，至少有一个物理显示设备(LCD 或 HDMI)。也许会同时存在 LCD / HDMI 两个物理显示设备，同时具有多个虚拟显示设备。每一个显示设备，就对应一个 framebuffer_device_t 对象。 
* 合成器模块(hwc_module_t) / 合成器设备(hwc_composer_device_1_t)，声明于 hwcomposer.h / hwcomposer_defs.h。合成器模块(hwc_module_t)继承自模块接口(hw_module_t)。合成器设备(hwc_composer_device_1_t)继承自设备接口(hw_device_t)。合成器的概念和用途有点复杂。后面会详细讨论。 
* 电源模块 power_module_t，声明于 power.h，继承自模块接口(hw_module_t)。显示设备的开/关，会使用到这个模块接口。 

# 数据结构与概念

HWC 的设计也是围绕着用 C 语言结构定义的两个接口(interface)展开：
* hwc_composer_device_1 代表着 HWC 的硬件抽象。继承自 hw_device_t 接口，由 HAL 底层模块实现。
* hwc_procs 调用 HWC 接口的调用者必须实现并向 HWC 传入这个接口，代表着调用者必须监听并处理 HWC 上报的 3 个事件：
  - invalidate
  - vsync
  - hotplug

![HWC device](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/hal-design/hardware_hwcomposer%20Class%20Diagram.svg)

hwc_composer_device_1 接口(interface)周期性调用的是 2 个方法：
* (*prepare)() 调用者与 HWC 协商，准备调用 HWC。
  - 调用者在每个层(layer)的参数"hwc_layer_t::compositionType"里要求 HWC 采用 GPU 或 2D engine 处理该层。
  - HWC 在返回参数里提示调用者，因为资源的限制，有些层由调用者采用 GPU 处理，有些层由 HWC 采用 2D engine 处理。
* (*set)() 执行合成操作。
  - 调用者在调用 (*set)() 前，先将需要采用 GPU 处理的层都用 GLES 函数执行合成操作。Framebuffer 上就会有带各种空洞的内容。
  - HWC 在 (*set)() 里，采用 2D engine 对剩下的层都执行合成操作。实际上就是用剩下的层的内容填补了 Framebuffer 上的空洞。
  - 最后由 (*set)() 将 Framebuffer 送到显示设备上显示。

![graphics_pipeline](https://source.android.com/devices/graphics/images/graphics_pipeline.png)

之所以会有这样的两段操作，是因为：
* GPU 是通用型的，可以实现各种颜色格式的各种合成算法。而 2D engine 能实现的算法是有限的，通道资源是有限的。
* 因为厂商实现的不同，有些类型的合成操作，这家采用 GPU 更好，那家采用 2D engine 更快更省电。

上面 2 个方法的参数，就它们可以同时向给多个显示设备输送内容。在 Android 显示设备模型里，可以有3种类型的显示设备：
* 主显示设备(LCD)
* 扩展显示设备(热插拔的 HDMI)
* 虚拟显示设备(抓屏或无线屏显)

在 Android 显示设备模型里，必须存在一个主显示设备，否则系统在初始化时会崩溃。在手机设备里，这大多是 LCD 设备。在 PC 机里，则是 HDMI 设备。所以，在 Android for PC 的版本里，在 Android 启动过程中，必须插入 HDMI 设备，让 Android 图形系统初始化完成。后面才可以热插拔 HDMI 设备。

所以上面 2 个方法的参数，同时可以处理 N 个(N >= 2)显示设备。数组的个数要 N >= 2，如果不存在扩展显示设备，也需要该数组元素，只是里面的成员都清零。

由上面 2 个方法的参数可知，代表显示设备的数据结构就是 hwc_display_contents_1。该结构包含了本次合成操作中要处理的层(layer)，由数据结构 hwc_layer_1 代表。每个层都有一个 Frame，对应的就是客户端里的窗口(ANativeWindow)里某个瞬间的内容。每个显示设备要处理的层列表可以不一样。典型的就是用扩展或虚拟显示设备看视频，在手机屏幕上显示的是操作界面，在大屏幕上显示的是视频内容。

层结构 hwc_layer_t 里的参数影响着合成操作：
* hwc_layer_t::compositionType 经过协商和最后执行的层的合成类型。
  - HWC_FRAMEBUFFER 该层由调用者采用 GPU 绘制到显示内存(Framebuffer)上。
  - HWC_OVERLAY 该层由 HWC 采用 2D engine 处理合成。
  - HWC_BACKGROUND 该层是背景层，刷上单一的背景颜色。系统里只有一个背景层。
  - HWC_FRAMEBUFFER_TARGET 该层保存着 HWC_FRAMEBUFFER 层合成的结果。实际上就是显示内存(Framebuffer)。申请该 Buffer 时需要增加参数"GRALLOC_USAGE_HW_FB"。
  - HWC_SIDEBAND 该层的内容由边带缓冲数据流填充。
  - HWC_CURSOR_OVERLAY 该层是光标层，由 HWC 处理。HWC 通过 setCursorPositionAsync() 方法异步地更新光标位置。
* hwc_layer_t::hints 底层 HWC 给调用者的提示。
  - HWC_HINT_TRIPLE_BUFFER 提示该层的窗口需要三缓冲。
  - HWC_HINT_CLEAR_FB 提示调用者将该显示内存(Framebuffer)刷上透明的颜色（清零）。调用者必须处理这个标志。
* hwc_layer_t::flags 调用者给底层 HWC 的标志。
  - HWC_SKIP_LAYER 底层 HWC 跳过这层不处理。尽管该层设置了 HWC_OVERLAY 类型也是如此。
  - HWC_IS_CURSOR_LAYER 该层是光标层。底层 HWC 最后按照 HWC_CURSOR_OVERLAY 类型处理。

上述的参数和协商结果很容易让人迷惑。所以有必要列表说明。

在调用 (*prepare)() 时，调用者给底层 HWC 的方向，2个参数：
* hwc_layer_t::compositionType
  - HWC_BACKGROUND 当底层 HWC 不能处理该类型时，会把它更改为 HWC_FRAMEBUFFER 类型，提示调用者处理该层。
  - HWC_FRAMEBUFFER_TARGET
    + 告诉底层 HWC，该层为显示内存(Framebuffer)。底层 HWC 需要做特殊处理。
    + 如果底层 HWC 有能力和资源，可以把其它层都改为 HWC_OVERLAY 或 HWC_BACKGROUND。这样调用者就不需要采用 GPU 处理合成操作。在调用 (*set)() 操作时，底层 HWC 也会忽略该层。
    + 实际上该层就是合成结果的目标层。
  - HWC_FRAMEBUFFER 该层由调用者采用 GPU 处理。当调用者设置该类型时，同时会设置层表的标志为 HWC_GEOMETRY_CHANGED。这是层的合成类型的缺省参数。当底层 HWC 能处理该层时，更改类型为 HWC_OVERLAY。
  - HWC_SIDEBAND 当底层 HWC 不能处理该类型时，会把它更改为 HWC_FRAMEBUFFER 类型，提示调用者处理该层。
* hwc_layer_t::flags

在调用 (*prepare)() 后，底层 HWC 给调用者的方向，2个参数：
* hwc_layer_t::compositionType
  - HWC_FRAMEBUFFER 当底层 HWC 不能处理 HWC_BACKGROUND 或 HWC_SIDEBAND 类型时，更改为 HWC_FRAMEBUFFER，提示调用者处理该层。
  - HWC_OVERLAY 当底层 HWC 能处理该层时，更改类型为 HWC_OVERLAY。并且它的合成用的一定不是 GPU。
  - HWC_CURSOR_OVERLAY 当调用者设置 hwc_layer_t::flags 为 HWC_IS_CURSOR_LAYER 时，当底层 HWC 能处理该层时，更改为 HWC_CURSOR_OVERLAY 类型，否则更改为 HWC_FRAMEBUFFER 类型。
* hwc_layer_t::hints

相应的，设备也有一个标志参数 hwc_display_contents_1_t::flags
* HWC_GEOMETRY_CHANGED 调用者指示底层 HWC，虽然层列表里，层的 buffer handle 有可能没变化，但 buffer 的内容一定有变化。
  - 在调用 (*prepare)() 前，调用者将设备标志设置为 HWC_GEOMETRY_CHANGED，同时将每个应用窗口的层设置为缺省值 HWC_FRAMEBUFFER。应用有特殊要求的，更改为其它类型。
  - 底层 HWC 将自己能处理的层都更改为 HWC_OVERLAY 或 HWC_CURSOR_OVERLAY，不能处理的更改为 HWC_FRAMEBUFFER。
  - 在调用 (*set)() 后，调用者从设备标志去掉 HWC_GEOMETRY_CHANGED。
  - 如果不设置该标志，在前后两次调用 (*prepare)() 时，底层 HWC 可以做优化，如果层列表里，层的 buffer handle 没有变化，可以认为 buffer 的内容没有变化。

FIXME: crop 图

# 显示路径

在每个显示设备的层列表里：
* 在以 z-order 排序的应用窗口的层列表里，最下面一定要增加一个 HWC_FRAMEBUFFER_TARGET 层。
  - HWC_FRAMEBUFFER_TARGET 层，在申请时要增加参数"GRALLOC_USAGE_HW_FB"。
  - 往该层写入内容将会显示到显示设备上。
* 如果底层 HWC 底层不能处理的层，将由调用者采用 GPU 合成到 HWC_FRAMEBUFFER_TARGET 层上。
  - GLES 需要一个 EGLSurface，所以该层一定属于一种 Surface，背后有一个 BufferQueue。
  - 这种特殊的 FramebufferSurface 和普通的 Surface 最大的不同在于：
    + 普通的 Surface 可以更改几何尺寸和颜色格式。而 FramebufferSurface 初始化完以后就不可更改。因为显示设备的分辨率和颜色格式不容易更改，而且 Android 目前还没有支持动态分辨率。
    + 普通的 Surface 可以改变 BufferQueue 的大小，缺省是 3 个 Buffer。而 FramebufferSurface 不可改变 BufferQueue 的大小，缺省是2个 Buffer。
* 底层 HWC 底层能处理的 HWC_OVERLAY 层，可以假定其合成结果也存放到 HWC_FRAMEBUFFER_TARGET 层上。
* 所以，合成次序是先用 GPU 合成再用 2D engine 合成。最后由 (*set)() 将 HWC_FRAMEBUFFER_TARGET 层送到显示设备上显示。
* 所以，不论是 GLES 路径还是 2D engine 路径，最终的合成结果都会来到 HWC_FRAMEBUFFER_TARGET 层上。所谓的退出栅栏(Retire fence)信号，就是该 HWC_FRAMEBUFFER_TARGET 层显示到屏幕上时，底层 HWC 发出的信号。

# HWComposer 2.0 的改进

![HWC2 device](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/hal-design/hardware_hwcomposer2%20Class%20Diagram.svg)

HWC2 的主要改进是：
* 将暴露的方法改变为可扩展的 getCapabilities() / getFunction() 方式。
* prepare() / set() 方法改变为 validateDisplay() / presentDisplay()，更符合设计本意。
* 将释放栅栏(Release fence)和退出栅栏(Retire fence)从推测性的改变为非推测性的。

# Framebuffer Device 的接口与约定

Framebuffer Device 接口对应的就是古老的 Linux Framebuffer 设备驱动，可以被认为是 HWComposer 1.0。新的 HWComposer 1.1 接口已经替代 Framebuffer Device 接口。在 Android 代码里，虽然为兼容性还保留着相关代码，但是底层由 HWC 做具体实现，已经不会调用到没有硬件加速的 Linux Framebuffer 设备驱动了。因此，对于 Framebuffer Device，只需要知道有这么一个东西就好。关注点应该在 HWComposer 1.1 / 2.0 上面。

![Framebuffer Device](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/hal-design/hardware_fb%20Class%20Diagram.svg)


