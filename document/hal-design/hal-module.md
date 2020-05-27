# Android 的图形系统的 HAL 接口与约定
* * *

# HAL接口(Interface)的约定

Android 系统大致可以分成上下两个部分。上部是和硬件无关的抽象的平台，下部则是和硬件相关的、需要移植的底层。之间则是用硬件抽象层(Hardware Abstraction Layer / HAL)来划分： 
* Android HAL(硬件抽象层)是硬件和软件之间的桥梁。 
* Android HAL 允许 Android 应用程序/框架与设备驱动程序进行通信，并操控硬件。
* Android HAL 提供了 C/C++ 层面的接口(interface)，下面是一个基于厂商 Soc 的特定实现。
* 大多数基于特定厂商的硬件的设备操控的实现都是在 Android HAL 中完成，而不是在设备驱动程序中完成，其中一个原因就是：设备驱动程序（开源许可证 GPL）和 HAL（Apache 许可证）之间的许可证差异。所以，HAL 的设计可以为厂商提供更多的抽象级别的接口和知识产权保护。 

在 Android 系统的进化过程中，HAL 一直尽可能地保持着稳定。旧接口一般不会修改，而是通过扩展新接口方式保证后向兼容。而且，HAL 还要求保证 C/C++ 编程语言兼容。因为从 Linux 设备驱动到底层的接口库，还有 EGL / OpenGLES，都是 C 语言的函数接口。而 Android framework 是用 C++ 编程。所以为了保证从上到下调用无碍，又要体现面向对象的编程思想，HAL 采用了很多 C/C++ 混编的技巧。其中最典型的技巧就是用 C 语言结构(struct)头部一致的特性来实现单继承，再用函数指针做结构成员表达接口(interface)类，而该接口(interface)类在 C++ 的环境里可以被实现类继承，并实现其中的接口。而且该 C++ 的实现类还可以在 C 语言的代码库里，例如 EGL / OpenGLES，被调用到。上下通畅无碍，这是很值得借鉴的编程经验。

相比之下，Android framework 层的代码变动就大很多。典型的就是 Graphic 相关的类，例如 GraphicBuffer 类和 Surface 类，为了增加新功能和优化显示效率，基本上 Android 每一个大版本的升级都会有大变动。这为代码复用带来了很大的麻烦。同时，在 Android 的文档库里，经常将 HAL 的接口约定和 Framework 层的实现混杂在一起进行说明。这对于分析和理解 Android 本身的代码没有问题。但是如果要复用 HAL 代码而重新实现一套 Framework 层，要将两者区分开，也是一个挑战。

因此，我们不但要知道 HAL 所实现的功能的目标，还需要知道 HAL 完成功能所需要的条件。这些条件，不但包括调用的参数的定义和范围，还包括调用对象的有限状态机，对象间的序列图和协作图，以及各种假设。只有了解了这些，才可以更好地了解 Android 系统的工作方式，以及更好地复用 Android 系统的代码。

另外，再怎么强调稳定，HAL 定义的接口(interface)还是会进化，还是会出现以前没有预料的情况。为解决这种问题，Android 采用了两种解决方案。一个是升级接口(interface)风格，采用通用的方法：getCapabilities() / getFunction()。设备所具有的能力和操控 API，就从这两个方法中获得。这种方案，和 UNIX / Linux 的扩展系统调用 ioctl() 的思路一样，就是造一个魔术包裹，什么东西都从里面变出来。

第二个方案是提供 HAL Interface Definition Language (HIDL)。因为各个厂商跟进 Android 新版本的状况不一样， HAL 有多版本共存，市面上的 Android 设备出现了碎片化。HIDL 就是要解决这种情况。同时，为了 Android 系统的稳定性，Android 系统从多线程协作模式转向多进程模式。HIDL 可以使得接口和实现可以跨进程分布，并且封装了 binder 编程细节，可以快速开发接口(interface)。

# HAL 设备通用模型 

在文件 hardware.h 中，声明了 Android HAL 对设备描述的最通用的模型。设备模型被划分为模块接口(hw_module_t)和设备接口(hw_device_t)。模块接口(hw_module_t)一定会有模块方法(hw_module_methods_t)。模块方法(hw_module_methods_t)至少含有一个 open() 方法，以便能打开硬件设备，获得相应的设备接口(hw_device_t)。设备接口(hw_device_t)至少包含一个 close() 方法，以便能关闭硬件设备。相关的概念和用法，见后面具体各类的类图。

![hw_module_t类图](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/hal-design/hardware_hardware%20Class%20Diagram.svg)

# Android 的图形系统的 HAL 接口(interface)类的代码分布 

图形系统有关的 HAL Interface 主要分成以下 4 个大类：
* Producer 端：Graphic Alloc (gralloc) 模块。ANativeWindowBuffer 依赖该模块。
* Queue ：OS 底层提供的原子级的同步栅栏(Fence)接口。基于多线程/进程的队列管理需要显式同步机制。
* Consumer 端：hwcomposer 模块。硬件合成器(HWC)是整个显示子系统的硬件抽象，包括显示设备，是所有 Android 合成操作(composition)的核心。
* 图形系统的驱动力：VSYNC 信号是 Android 图形系统运转的心跳(heartbeat)，或者说是编舞者(Choreographer)。

此外，还涉及一个电源管理模块 Power。因为显示设备会有开关的需求，所以上层的 ISurfaceComposer 会调用该模块实现休眠/唤醒功能。

Android 的图形系统的 HAL 接口(interface)类的声明所涉及到的头文件并不多：
* gralloc 模块
  + hardware/libhardware/include/hardware/gralloc1.h
  + hardware/libhardware/include/hardware/gralloc.h
* Fence 接口
  + system/core/libsync/include/android/sync.h
  + system/core/libsync/include/ndk/sync.h
  + system/core/libsync/include/sync/sync.h
* hwcomposer 模块
  + hardware/libhardware/include/hardware/fb.h
  + hardware/libhardware/include/hardware/hwcomposer2.h
  + hardware/libhardware/include/hardware/hwcomposer_defs.h
  + hardware/libhardware/include/hardware/hwcomposer.h
* VSYNC 信号：只是一个函数指针藏在 HWComposer 模块里。
* power 模块
  + hardware/libhardware/include/hardware/power.h

最终，Producer 端用 gralloc 模块要实现的 Framework Interface 为 ANativeWindowBuffer / ANativeWindow，声明位于这里：
* frameworks/native/libs/nativebase/include/nativebase/nativebase.h
* frameworks/native/libs/nativewindow/include/system/window.h
* system/core/include/cutils/native_handle.h

在 Framework 层对应的实现类 GraphicBuffer / Surface，声明位于这里：
* frameworks/native/include/ui/GraphicBuffer.h
* frameworks/native/include/gui/Surface.h

Consumer 端用 hwcomposer 模块要实现的类为 HWComposer，声明位于这里：
* frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.h

HWComposer 是 SurfaceFlinger 对于整个硬件显示子系统的抽象，包括显示设备，是所有 Android 合成操作的核心。

从系统总体实现的角度看，Interface 是最重要的。它们是功能锚点，定义了系统功能。在图形系统中，目标就是用 HAL Interface 实现 Framework Interface。至于中间层的实现(Implementation)，可以各用各的软件包，各有各的实现。Android Framework 只是其中一个经典实现。而 [Sailfish OS](https://sailfishos.org/) 则用 Qt5 做了另外的实现，并且在 Consumer 端没有使用 surfaceflinger service，而是实现了 wayland server。也就是，在执行 queueBuffer() 的时刻，Buffer 被送给了另外的 HWComposer 实现。当然，最终 wayland server 还是调用了 hwcomposer 模块，将应用所绘制的内容交给了显示设备。

本章只关注接口(interface)的约定，具体的实现将在后面的文章讨论。下面是这些模块的功能介绍。 

# [gralloc 模块](gralloc.md)

# [同步栅栏(Fence)](fence.md)

# [hwcomposer 模块](hwcomposer.md)

# [VSYNC 信号](VSYNC.md)

# power 模块

显示设备是手机里的用电大户。power 模块相对简单，就是提供给 hwcomposer 模块开关设备。主要就是在休眠/唤醒时被 hwcomposer 模块使用到。

![power_module_t类图](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/hal-design/hardware_power%20Class%20Diagram.svg)

# [代码复用](reuse.md)

# 小结

Android HAL 中和图形系统有关的模块，有这几方面：
* Producer 端的支撑为 gralloc 模块。目标是要实现 framework 层的 ANativeWindowBuffer 接口。
* Consumer 端的支撑为 hwcomposer 模块。目标是要实现 SurfaceFlinger 的 HWComposer 模块。硬件合成器(HWC)是整个硬件显示子系统的抽象，包括显示设备，是所有 Android 合成操作的核心。
* 对于 Queue，依靠 OS 底层提供的原子级的 Fence 接口，Android 图形系统采用并实现了显式同步机制。
* VSYNC 信号是 Android 图形系统运转的心跳(heartbeat)，或者说是编舞者(Choreographer)。

# 参考文件
1. [Android's Graphics Buffer Management System (Part I: gralloc)](https://www.codeproject.com/Articles/991640/Androids-Graphics-Buffer-Management-System-Part-I)
1. [The Android Graphics microconference](https://lwn.net/Articles/569704/)
1. [Hardware Composer 2.0](https://blog.linuxplumbersconf.org/2016/ocw//system/presentations/4185/original/LPC%20HWC%202.0%20&%20drm_hwcomposer%20.pdf)
1. [Android Hwcomposer on KMS](https://www.slideshare.net/linaroorg/kms-hwcomposer)
1. [Implementing the Hardware Composer HAL](https://source.android.com/devices/graphics/implement-hwc)
1. [Implementing VSYNC](https://source.android.com/devices/graphics/implement-vsync) 
1. [Synchronization framework](https://source.android.com/devices/graphics/index.html#synchronization_framework)
1. [Android Sync](https://blog.linuxplumbersconf.org/2014/ocw/system/presentations/2355/original/03%20-%20sync%20&%20dma-fence.pdf)
1. [VSync信号](http://windrunnerlihuan.com/2017/05/21/VSync%E4%BF%A1%E5%8F%B7/)
 

