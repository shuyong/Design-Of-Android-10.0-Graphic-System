# gralloc 模块
* * *

# 图形系统处理的介质：Buffer 

在 framework 层，有两个图形系统最重要的概念：Buffer / Window。对应的两个接口(interface)分别是：ANativeWindowBuffer(Android Native Window Buffer) / ANativeWindow(Android Native Window)。 

ANativeWindowBuffer 是实际和 HAL 打交道的接口，最终会调用到 gralloc 接口。gralloc 接口由厂商根据硬件方案做实现，在系统初始化的时候以 plugin 的方式装载进来。

gralloc 模块的实现，还受限于图形格式(format)、图形变换(transformation)算法、颜色空间(colorspace)的相关定义，声明于 graphics.h。 
* system/core/include/system/graphics-base.h
* system/core/include/system/graphics-base-v1.0.h
* system/core/include/system/graphics-base-v1.1.h
* system/core/include/system/graphics-base-v1.2.h
* system/core/include/system/graphics.h
* system/core/include/system/graphics-sw.h

由此看出这些信息是系统底层(System)对中间架构层(Framework)的限制。

gralloc(graphics alloc) 模块管理着 Buffer 资源(buffer_handle_t)，声明于 gralloc.h。gralloc 模块就是对硬件级别的 Buffer 的管理模块，参见文档[[gralloc](https://www.codeproject.com/Articles/991640/Androids-Graphics-Buffer-Management-System-Part-I)]。该模块也是很通常地分为两个部分：gralloc_module_t(继承自 hw_module_t) / alloc_device_t(继承自 hw_device_t)。后面对 gralloc 模块的表述，是指整体功能，不区分具体哪个类。 

本章所说的 Buffer，与 gralloc buffer 是同一概念，是 Android 里常用的术语。

# Buffer 的底层设计 

## 管理 Buffer 的基本功能

做为 Buffer 的管理模块，gralloc 模块有 6 个基本功能： 
* 申请 alloc() / 释放 free() 缓冲区。 
* 锁定 lock() / 解锁 unlock() 缓冲区。
  - 锁定 lock() 缓冲区，意味着该缓冲区内存对于当前调用者可用。当前调用者将立即在内存中绘制内容。
  - 解锁 unlock() 缓冲区，意味着当前调用者结束绘制，该缓冲区可以交给其它设备使用。 
* 注册 registerBuffer() / 解除注册 unregisterBuffer() 缓冲区。用于跨进程的操作。
  - 当本进程得到一个别的进程已申请好的 Buffer 时，需要向底层系统注册 registerBuffer()，以便底层系统进行相应的操作，例如映射内存，使得本进程能操作相应的 Buffer。
  - 反之，当本进程不再需要该 Buffer 时，需要进行解除注册 unregisterBuffer() 操作，以便底层系统进行相应的操作，例如解除映射，对象自减计数等等，使得跨进程的 Buffer 对象状态一致，可以进行自我生命周期管理。
  - 在 Android 7.0 及以后的版本中，在 gralloc1 模块中，这两个函数改名为：retain() / release()。一个是增加引用计数，一个是减少引用计数。这样更能体现设计本义。

下面是 gralloc v0 模块的类图：
![gralloc 模块的类图](Gralloc%20Class%20Diagram.svg)

下面是 gralloc v1 模块的类图：
![gralloc1 模块的类图](Gralloc%20Class%20Diagram.svg)

## 共享机制

由于 gralloc buffer 跨多个进程共享，必须真正地跨进程共享内容和引用计数。这是一个 gralloc buffer 的概念，它基于文件描述符引用计数。 

把 gralloc buffer 想像为一个文件，存在多个文件描述符共同访问同一个文件。就像 kernel 对待普通文件会发生什么一样，kernel 用打开的文件描述符来跟踪它。要跨进程传输一个 gralloc buffer，你可以使用标准的 kernel 功能，在一个 socket() 带外数据结构中传输一个文件的描述符。 

因此，在 Android Framebuffer 层的实现里，当一个 gralloc buffer 在两个进程之间共享，每个进程都有它自己的 GraphicBuffer 对象，每个对象都有它自己的引用计数(refcount)，这些都是共享同一个底层的 gralloc buffer（但有不同的打开着的描述符）。共享通过：服务端调用 GraphicBuffer::flatten 序列化发送，和客户端调用 GraphicBuffer::unflatten 反序列化接收它。GraphicBuffer::unflatten 将调用 BufferMapper::registerBuffer，以确保 OS 底层的缓存区句柄的引用计数是正确的。 

当 GraphicBuffer 的引用计数变为零，析构函数会调用 free_handle，名字为 BufferMapper::unregisterBuffer 的方法，它将关闭文件描述符，从而递减 gralloc buffer 的引用计数。 

## 实现 Buffer 时需要考虑的硬件因素 

ANativeWindowBuffer 最终与 gralloc 模块打交道。它对上层屏蔽了硬件细节。但是如果要做具体实现，则需要了解很多底层的硬件特性。 

ANativeWindowBuffer 的设计，有 5 个基本参数： 
* width 
* height 
* stride 
* format 
* usage 

宽度(width)和高度(height)，代表图形的 2D 尺寸。当描述图形缓冲区的尺寸时，需要注意两点。首先，我们需要理解维度的单位。如果尺寸用像素表示，那么我们需要理解如何将像素转换为位。为此我们需要知道颜色编码格式。 

颜色格式(format)就是用于标识颜色编码格式。常用颜色格式有 RGBA_8888，是每像素32位(每个像素有4个组分：红、绿、蓝和 alpha 混合，每个8位)，这个与 OpenGL 的常用格式一致。RGB_565 是每个像素 16 位(5位为红色和蓝色，6位为绿色)。此外，还有 YUV 格式。 

影响图形缓冲区物理尺寸的第二个重要因素是它的步幅(stride)。有些地方也称为 pitch。为了理解步幅，可见下图。 
 
![buffer stride 1](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/hal-design/window-01.svg)

我们可以把内存缓冲区看作像素行和列的矩阵排列。列组成行。步幅被定义为所需要的从一条缓冲线(扫描线)的开始计数到下一条缓冲线的像素的数量(或字节，取决于描述的单位)。如上图所示，步幅必须至少等于缓冲区的宽度，但是也可以无碍地大于宽度。步幅和宽度(stride-width)之间有差别只是浪费了内存，而由此带来的特性是，用于存储图像或图形的内存可能不是连续的。那么，步幅是从哪里来的呢？由于硬件实现的复杂性、存储器带宽优化和其他限制，访问图形存储器的硬件可能要求缓冲区是若干字节的倍数。例如，如果对于特定的硬件模块，线路地址需要对齐到64字节，那么存储器宽度需要是64字节的倍数。如果此约束导致得到比请求时更长的扫描线，则缓冲步幅与宽度不同。步幅的另一个动机是缓冲区重用：想象一下，你想在另一个图像中引用裁剪图像。在这种情况下，裁剪（内部）图像具有不同于宽度的步幅。见下图。 
 
![buffer stride 2](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/hal-design/window-02.svg)

已分配的缓冲存储器当然可以通过用户空间的代码，其实就是 CPU，写入或读取，但是首先它可由不同的硬件模块写入或读取，例如 GPU (图形处理单元)、照相机、合成引擎、DMA引擎、显示控制器等。在一个经典的片上系统(SoC)中，这些硬件模块来自不同的厂商，对缓冲存储器有不同的约束，如果它们要共享缓冲区，则需要协调这些约束。例如，被 GPU 写入的缓冲器，对显示控制器应该是可读取的。缓冲区上的不同约束不一定是异构组件供应商的结果，也可能是因为优化点不同。在任何情况下，Buffer 都需要确保图像格式和内存布局对图像生产者和消费者都是一致的。这就是 usage 参数发挥作用的地方。 

参数 usage 有3个功能：一是描述软件如何读取缓冲区（从不，很少，经常）；二是描述软件如何写入缓冲区（从不，很少，经常）。三是描述硬件如何使用缓冲区：作为 OpenGL ES 纹理(texture)或 OpenGL ES 渲染(render)目标；由 2D 硬件位块传送器(blitter)、HWComposer、framebuffer 设备或硬件视频编码器写入；或由硬件相机流水线读取；做为零快门延迟相机队列的一部分；做为 RenderScript 分配缓存；在外部显示器上全屏显示；或用作光标。 

显然，颜色格式和 usage 标志之间可能存在某种耦合。例如，如果 usage 参数指示缓冲区由相机写入并由视频编码器读取，则格式必须为两个硬件模块所接受。 

如果软件需要访问缓冲区内容（或读或写），那么 Buffer 需要确保从物理地址空间到CPU的虚拟地址(virtual address)空间的映射以及缓存(cache)保持一致。 

## 影响图形内存布局的其它因素

除了前面介绍的参数以外，还有其他因素影响如何分配图形和图像内存，以及如何存储(内存布局)和访问图像： 

### 对齐(Alignment)

再一次，不同的硬件可能施加或硬或软的内存对齐要求。不遵守硬要求将导致硬件无法执行其功能，而不遵守软要求将导致硬件的不最佳使用（通常体现在功率、发热和性能方面）。

### 颜色空间，格式和内存布局(Color Space, Formats and Memory Layout)

这里有几种颜色空间，其中最熟悉的是 YCbCr（图像）和 RGB（图形）。在每个颜色空间内，信息可以不同地编码。一些 RGB 编码例子包括 RGB565（16 位；红色和蓝色各 5 位，绿色 6 位）、RGB888（24 位）或ARGB8888（32 位；具有 α 混合通道）。YCbCr 编码格式通常采用色度子采样。参见文档[[gralloc](https://www.codeproject.com/Articles/991640/Androids-Graphics-Buffer-Management-System-Part-I)]的说明。

### 堆砌(Tiling)

如果片上系统(SoC)的硬件使用访问相邻像素块为主的算法，那么安排图像内存布局，使得相邻像素以线状布局，而不是通常的位置的布局，这可能更有效率。这称为平铺。 

一些图形/图像硬件使用更精细的拼接，例如支持两种拼接大小：一组小拼接可以以某种扫描顺序排列在较大的拼接内。参见文档[[gralloc](https://www.codeproject.com/Articles/991640/Androids-Graphics-Buffer-Management-System-Part-I)]的说明。

### 压缩(Compression)

如果生产者和消费者都是同一片上系统(SoC)的硬件组件，则其可以写和读通用的专有压缩数据格式，并动态地解压缩数据(例如，在处理像素数据之前使用芯片内的内存，通常是 SRAM)。 

### 内存连续性(Memory Contiguity)

一些较老的图像硬件模块（相机、显示器等）没有MMU或不支持分散-聚集(scatter-gather)DMA。在这种情况下，设备DMA使用指向一段物理地址连续的内存来编程。这不会影响内存布局，但肯定是 gralloc 在分配内存时需要注意的特定于平台的约束。 

## 缓冲区所有权管理(Buffer Ownership Management)

内存是一个共享资源。它可以在图形硬件模块和 CPU 之间共享，也可以在两个图形模块之间共享。如果 CPU 正在向图形缓冲区渲染，我们必须确保显示控制器在 CPU 开始读取缓冲存储器之前等待 CPU 完成写入。这是使用系统级同步完成的。但是这种同步不足以确保显示控制器将访问存储器的一致视图。在上面的示例中，CPU 写入的缓冲区的最终更新可能尚未从缓存刷新到系统内存。如果发生这种情况，显示器可能会显示图形缓冲区的错误视图。因此，我们需要某种低级的原子同步机制来显式地管理内存缓冲区所有权的转移，从而验证内存“所有者”是否看到了内存的一致视图。 

对缓冲存储器（不论对于硬件还是软件，不论读还是写）的访问由 Buffer 的用户显式管理（这可以同步或异步完成）。这是通过锁定和解锁缓冲存储器来完成的。可以有多个线程同时具有读锁，但只有一个线程可以持有写锁。关于同步机制，见后面的章节介绍。

### 缓存一致性(Cache Coherence)

如果软件需要访问图形缓冲区，那么 CPU 的读取和/或写入，需要访问正确的数据。保持缓存一致性是 Buffer 的职责之一。不必要的刷新缓存，或者在一些 SoC 上启用总线侦听，以保持跨图形硬件和 CPU 的内存视图的一致性，这些行为会浪费电力，并且可能增加延迟。因此，Buffer 也需要采用特定于平台的同步机制。见后面章节的介绍。

### 在内存中锁定页面(Locking Pages in RAM)

在 CPU 和图形硬件之间共享内存的另一个方面是确保在硬件使用内存页时，不会将内存页刷新到交换文件。目前在 Android 设备上的 Linux 没有使能交换文件的配置，但使能该配置确实是可行的。所以，lock() 函数应该在内存中锁定内存页。 

一个相关的问题是页面重新映射，当分配给一个物理页面的虚拟页面被动态地重新分配到另一个物理页面（页面迁移）时发生。内核可能选择这样做的一个原因是通过重新配置物理内存分配来防止碎片化。从 CPU 的角度来看，只要新的物理页面包含正确的内容，这就是好了。但从一个图形硬件模块的角度来看，这是从它们脚下拖动地毯。所以，与硬件共享的页面应该设计为不可移动的。


