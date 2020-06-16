# Buffer 与 Window 的设计
* * *

# Buffer 与 Window 的概念

从 Framework 层到应用程序，图形系统都会应用到的两个最重要的概念：Buffer & Window。这两个概念对应的是 ANativeWindowBuffer(Android Native Window Buffer) 与 ANativeWindow(Android Native Window) 这两个接口(interface)，名字不言自明。它们都是用 C 语言定义的接口，因为 EGL / OpenGLES / codec / camera 这些硬件加速模块大多是用 C 语言编写。

ANativeWindowBuffer / ANativeWindow，这两者都继承自 Android 本地接口(android_native_base_t)。声明位于这里：
* frameworks/native/libs/nativebase/include/nativebase/nativebase.h
* frameworks/native/libs/nativewindow/include/system/window.h
* system/core/include/cutils/native_handle.h

Android 本地接口(android_native_base_t)声明了引用计数接口(reference-counting interface)，可以在 C 语言级别实现智能指针，使得继承它的子类可以具有自我管理生命周期的功能。 

Android 本地资源接口(native_handle_t)，声明于 native_handle.h，用于描述用文件句柄方式跨越进程边界的共享资源。ANativeWindowBuffer 就是一种很典型的需要跨越进程边界的共享资源。Buffer 只有跨越进程边界进行共享，才能实现 zero-copy 功能。所以 ANativeWindowBuffer 就包含有一个 Buffer 资源接口(buffer_handle_t，native_handle_t 的别名)成员。

下图是 ANativeWindowBuffer 的类图
![ANativeWindowBuffer](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/framework-design/nativebase_nativebase%20Class%20Diagram.svg)

在这其中，结构成员"handle : buffer_handle_t"就是对底层 Buffer 的访问指针。HAL 中的 gralloc 模块，就是为了 ANativeWindowBuffer 能访问硬件。ANativeWindowBuffer 接口周期性使用的有 2 个方法：
* lock() 锁定缓冲区
* unlock() 解锁缓冲区

下图是 ANativeWindow 的类图
![ANativeWindow 的类图](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/framework-design/system_window%20Class%20Diagram.svg)

ANativeWindow 接口周期性使用的有 2 个方法：
* dequeueBuffer() 缓冲区出队。
* queueBuffer() 缓冲区入队。

这其实就是暗示着 ANativeWindow 接口背后是一个 Buffer Queue。

因为在 Android 文档里对 Buffer & Window 这两个接口(interface)没有太多的说明，对这两个接口的约定需要一点想象力。

首先我们假定从绘图到显示的 Producer-Consumer 模型是在两个进程里同步执行，则有如下步骤：
```
1. Producer : receive "Release" signal
2. Producer : dequeue buffer
3. Producer : lock buffer
4. Producer : draw to buffer
5. Producer : unlock buffer
6. Producer : receive "Retire" signal
7. Producer : queue buffer
8. Producer : sent "FrameAvailable" event

a. Consumer : receive "FrameAvailable" event
b. Consumer : acquire buffer
c. Consumer : lock buffer
d. Consumer : copy content
e. Consumer : unlock buffer
f. Consumer : release buffer
h. Consumer : sent "Release" signal
i. Consumer : show content
j. Consumer : sent "Retire" signal
```

之所以是同步执行，是因为 Single-Buffer 的原因。为了并行效率，在 Producer / Consumer 之间插入 Buffer Queue，里面管理 2~N 个 Buffer。同一时刻，Producer-Consumer 两端都有工作介质 Buffer。这样 Producer / Consumer 就可以并发执行，绘图和显示效率就会大大提高。

在这样的模型里，Buffer Queue 其实就是显示流水线的中心。它决定着 Consumer 在哪里。在 Android Framework 的设计里，BufferQueue 类将 Buffer 送给了 SurfaceFlinger，于是 SurfaceFlinger 就是内容的消费者。

而在 [Sailfish OS](https://sailfishos.org/) 项目里，他们实现的 ANativeWindow 接口，在 dequeueBuffer() / queueBuffer() 里使用的是另外的 Buffer Queue 实现。该 Buffer Queue 将 Buffer 送给了 wayland server，于是 wayland server 就是内容的消费者。也就是，对于 Android 代码复用，ANativeWindowBuffer / ANativeWindow 接口是必须要实现的，因为 EGL / OpenGLES / codec / camera 这些硬件加速模块需要 Buffer / Window 接口。但是消费端是可选的，由 Buffer Queue 决定。

# GraphicBuffer & Surface 的设计概略

GraphicBuffer 的类图和 Surface 的类图太大太复杂，不必要放在这里干扰思路。特别是 Surface 类，庞大又关联众多，是 Android 图形系统里第二复杂的类。第一复杂的类非 SurfaceFlinger 莫属了。如果对此感兴趣，可以查看后面的详细类图列表。这里只是从高度的概略角度探讨 GraphicBuffer & Surface 的设计。

下图是 GraphicBuffer 类的关系组合图：
![GraphicBuffer 类的关系组合图](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/framework-design/ui_GraphicBuffer%20Component%20Diagram.svg)

这个关系组合图只是表现了从 HIDL 层到 Framework 层的关系。如果要了解从 HAL 层到 HIDL 层的关系，需要查看前面的内容。如果在这个图里把这些内容加上，则关系图就会层次过深，不易把握。

下图是 Surface 类的关系组合图。这里主要关注的是 Surface 创建以及邻近类的关系。更广泛的关系见 BufferQueue 类的关系组合图。
![Surface 类的关系组合图](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/framework-design/gui_Surface%20Component%20Diagram.svg)

下面我们将简化这两个类，并做一些变形，只关注几个最重要的周期性使用的方法，以及最常用的调用分支，以查看这两个核心类的设计。对于 Surface 类，还将理解从 C 语言接口到 C++ 对象的实现技巧，以及调用路径。

# GraphicBuffer 的实现

在 android 系统里，gralloc 内存分配器会为 GraphicBuffer 进行缓冲区分配，并通过供应商特定的 HAL 接口来实现。在 HAL 层，老版本的 gralloc 模块，v0 版本，分成 2 个功能模块：
* gralloc_module_t
  - registerBuffer()
  - unregisterBuffer()
  - lock()
  - unlock()
* alloc_device_t
  - alloc()
  - free()

对应的 Framework 一层的封装就是：
* GraphicBufferMapper
* GraphicBufferAllocator

有 HIDL 之后，对应的封装就是：
* IMapper
* IAllocator

此外，gralloc 模块又设计了新版本，v1 版本：
* v0 : alloc_device_t & gralloc_module_t
* v1 : gralloc1_device_t

下面是两个版本的方法对照表：

| No. | alloc_device_t(v0) | gralloc_module_t(v0) | gralloc1_device_t(v1) |
|:---:|--------------------|----------------------|-----------------------|
|  1  | alloc()            |                      | allocate()            |
|  2  |                    | registerBuffer()     | retain()              |
|  3  |                    | unregisterBuffer()   | release()             |
|  4  |                    | lock()               | lock()                |
|  5  |                    | unlock()             | unlock()              |
|  6  | free()             |                      |                       |

gralloc 模块 v1 版本，取消了强制释放方法 free()，强调了跨进程资源通过计数自我管理生命周期的设计思路。只有在全局上，该 buffer 的所有持有者都用 release() 方法释放了该 buffer，计数为 0，则底层自动释放所有跨进程共享的资源。

经过简化和变形，可以看到如下的关系组合图：

![GraphicBuffer 类的调用路径](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/framework-design/ui_GraphicBuffer%20Component%20Diagram%20-%20Call%20Path.svg)

GraphicBuffer 的实现的功能，划分为 Mapper & Allocator 两大分支，主要是考虑：
* alloc() & free() 使用频率很少，只在 GraphicBuffer 类构造和析构的时候使用，所以 在 GraphicBuffer 类中都不保留 Allocator 的指针。要用的时候再取回单例指针就可以了。
* lock() & unlock() 使用频率很高，所以在 GraphicBuffer 类中保留 Mapper 的指针，以便快速访问。
* Mapper 中的 importBuffer() & freeBuffer() 方法，对应 HAL 层的 retain() & release() 方法。在 GraphicBuffer 类构造和析构的时候，将 Allocator 的功能引入到 Mapper 中。这样，就可以避免使用 Allocator 的指针。

# Surface 的实现

Surface 的实现，采用了 C/C++ 混编实现接口(Interface)的技巧。其中最典型的技巧就是用 C 语言结构(struct)头部一致的特性来实现单继承，再用函数指针做结构成员表达接口(interface)类，而该接口(interface)类在 C++ 的环境里可以被实现类继承，并实现其中的接口。而且该 C++ 的实现类还可以在 C 语言的代码库里，例如 EGL / OpenGLES，被调用到。

下表是应用程序，如 EGL / OpenGLES，调用 ANativeWindow 接口中的 dequeueBuffer() / queueBuffer() 时，实现类方法被调用的次序：

| No. | C function pointer (struct)    | C++ static method (class)     | C++ virtual method (class) | Required Binder Interface               |
|:---:|--------------------------------|-------------------------------|----------------------------|-----------------------------------------|
|  1  | ANativeWindow::dequeueBuffer() | Surface::hook_dequeueBuffer() | Surface::dequeueBuffer()   | IGraphicBufferProducer::dequeueBuffer() |
|  2  | ANativeWindow::queueBuffer()   | Surface::hook_queueBuffer()   | Surface::queueBuffer()     | IGraphicBufferProducer::queueBuffer()   |

下图是经过简化和变形后的关系组合图，可以看到 ANativeWindow 接口(Interface)从上到下的调用路径：

![Surface 类的调用路径](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/framework-design/gui_Surface%20Component%20Diagram%20-%20Call%20Path.svg)

# Buffer 基于生命周期的管理

## Buffer 有限状态机

在 Android 文档里，谈到过 Buffer 有限状态机：

![Buffer 有限状态机](https://source.android.com/devices/graphics/images/bufferqueue.png)

![](https://img-blog.csdn.net/20140930173508266?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvamluemh1b2p1bg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

但是，这些都是以 BufferQueue 的角度看待 Buffer 的状态，既不全面，也不本质。

实际上，对于Producer 端的用户，CPU 或 GPU，使用 Buffer 的目的就是为了绘图，只需要 lock() / unlock() 方法就足够了。因此，Buffer 最简单的有限状态机如下：
![Buffer - native](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/framework-design/Android%20Native%20Window%20Buffer%20Statemachine%20Diagram%20-%20native.svg)

这种情况多用于在程序内部申请实现了 ANativeWindowBuffer 接口的 Buffer 类，并将其生成为 EGLImage，然后绑定到 GLES Texture，最后，使用 GLES 的函数绘图，其结果就保存在 Buffer 上面了。在这个过程中，会有两重的 lock() / unlock() 操作。

如果使用实现了 ANativeWindow 接口的 Window 类，则暗示了背后有一个 BufferQueue，而该类则做为 BufferQueue 的生产端。所以 ANativeWindow 接口的最重要的方法就是 dequeueBuffer() / queueBuffer()。在这种情况下 Buffer 的有限状态机如下：
![Buffer - Producer](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/framework-design/Android%20Native%20Window%20Buffer%20Statemachine%20Diagram%20-%20Producer.svg)

需要注意的是，当该 Buffer 在 BufferQueue 里重新入队以后，Buffer 为 Queued 状态，Producer 端会通知 Consumer 端来取件。

因此相对应的，在 BufferQueue 的 Consumer 端，Buffer 的有限状态机如下：
![Buffer - Consumer](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/framework-design/Android%20Native%20Window%20Buffer%20Statemachine%20Diagram%20-%20Consumer.svg)

在这个跨进程的流程里，实现了 zero-copy 功能，不论是 BufferQueue 还是消费者(SurfaceFlinger)，都只有最小信息量通过 OS 传输。

将这两个 Buffer 的有限状态机放一起观察，将会发现两者的异同：
![Buffer - Producer-Consumer](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/framework-design/Android%20Native%20Window%20Buffer%20Statemachine%20Diagram%20-%20Producer-Consumer.svg)

而前面所说的 Dequeued / Queued / Acquired / Released 这4种状态，是指 Buffer 在 BufferQueue 里的状态。
![Buffer Queue State](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/pattern-design/03-bufferqueue-03.svg)

回头看直接使用 ANativeWindowBuffer 和通过 ANativeWindow 使用 ANativeWindowBuffer 的不同。最大的不同在于直接使用 ANativeWindowBuffer，绘制在 Buffer 的内容是没有自动出口的，不会自动渲染到屏幕上的。要想该 Buffer 里的内容能输出出来，则需要多做一些工作。

对于 GPU，则是通过 OpenGLES 函数把 GL Texture 里的内容贴到 EGL Surface上。而这 EGL Surface，其实就是一个 ANativeWindow 被绑定到 EGL 里生成的对象，从 GL Texture 贴来的内容，就被渲染到 BufferQueue 管理的 Buffer 上面。接着用户通过调用 eglSwapBuffers() 函数，底层就会调用 ANativeWindow::queueBuffer()，于是绘制内容就被发送出去了。底层接着再调用 ANativeWindow::dequeueBuffer()，于是上层用户又有新的 Buffer 可以贴图了。

对于 CPU，要想取出 GL Texture 里的内容，则要麻烦一些，因为在 GL Texture 里的 Buffer 是双重锁定，所以需要 删除 GL Texture 和删除 EGLImage 这两次解锁。然后 CPU 层面就可以使用 lock() 方法获取 Buffer 的指针，然后就可以读/写 Buffer 上的内容了。操作完成以后，记得用 unlock() 方法解锁。

## 从实现看假设

因为 Android 文档里没有谈到 ANativeWindowBuffer / ANativeWindow 的约定，所以我们只能从 Android Framework 现有的实现入手，反推其中的假设。

在 Android Framework 里，是用 GraphicBuffer 类实现了 ANativeWindowBuffer 接口(interface)，用 Surface 类实现了 ANativeWindow 接口(interface)。Surface 类是 Producer-Consumer 设计模式中 Producer 端。

ANativeWindow 接口周期性使用的有2个方法：
* dequeueBuffer() 缓冲区出队。
* queueBuffer() 缓冲区入队。

这其实就是表明 ANativeWindow 接口背后是一个 BufferQueue。那对于其中的"Queue"，会有一些重要假设：
* 这个队列是一个 FIFO 队列。里面的 Buffer 要按序循环使用。这是一个很重要的约定。这意味着：
  - 从这个队列里取出的 Buffer 要按照取出时的顺序放回去。
  - 一般是 dequeueBuffer() / queueBuffer() 调用成对出现。
  - 但也可以连续取出多个 Buffer，使用完后，在入队时要确保按照取出时的顺序放回去。这多用于 Camera / Video 应用。
* 这个队列里 Buffer 的个数允许调整。
  - 对于 Surface 类，一般是3个 Buffer。
  - 但也可以调整得更多个，如在 Camera / Video 应用中，以空间换时间。

在 Android 的实现里，BufferQueue 两端的接口是 IGraphicBufferProducer / IGraphicBufferConsumer。Surface 类就是通过 IGraphicBufferProducer 接口(interface)与 BufferQueue 类进行跨进程/线程打交道。

IGraphicBufferConsumer 是 Producer-Consumer 设计模式中 Consumer 端的接口(interface)。该接口周期性使用的有2个方法：
* acquireBuffer() 帧出队。
* releaseBuffer() 帧入队。

两端所用方法的名字的不同，是设计者想表明：
* IGraphicBufferProducer 一端，操作的是空闲缓冲区(Free Buffer)。IGraphicBufferConsumer 一端，操作的是带内容(Content)的帧(Frame)。
* IGraphicBufferProducer 一端，可以连续取出多个 Buffer，但是要确保按照取出的顺序放回去。IGraphicBufferConsumer 一端，按照生成端先入先出的顺序，一次只取一个 Frame。

# Buffer 与 EGL / OpenGLES 的关系

## EGL / OpenGLES 的 Android 接口

OpenGL ES (GLES) 定义了用于与 EGL 结合使用的图形渲染 API。EGL 是一个规定如何通过操作系统创建和访问窗口的库（要绘制纹理多边形，请使用 GLES 调用；要将渲染放到屏幕上，请使用 EGL 调用）。 

OpenGL ES 定义了一个渲染图形的 API，但没有定义窗口系统。为了让 GLES 能够适合各种平台，GLES 将与知道如何通过操作系统创建和访问窗口的库结合使用。用于 Android 的库称为 EGL。如果要绘制纹理多边形，应使用 GLES 调用；如果要在屏幕上进行渲染，应使用 EGL 调用。 

在使用 GLES 进行任何操作之前，需要创建一个 GL 上下文。在 EGL 中，这意味着要创建一个 EGLContext 和一个 EGLSurface。GLES 操作适用于当前上下文，该上下文通过线程局部存储访问，而不是作为参数进行传递。这意味着必须注意渲染代码在哪个线程上执行，以及该线程上的当前上下文。 

## EGLSurface 

EGLSurface 可以是由 EGL 分配的离屏缓冲区（称为"pbuffer"），或由操作系统分配的窗口。EGL 窗口 Surface 通过 eglCreateWindowSurface() 调用被创建。该调用将“窗口对象”作为参数，在 Android 上，该对象可以是 SurfaceView、SurfaceTexture、SurfaceHolder 或 Surface，所有这些对象下面都有一个 BufferQueue。当进行此调用时，EGL 将创建一个新的 EGLSurface 对象，并将其连接到窗口对象的 BufferQueue 的生产方接口。此后，渲染到该 EGLSurface 会导致一个缓冲区离开队列、进行渲染，然后排队等待消耗方使用。（术语“窗口”表示预期用途，但请注意，输出内容不一定会显示在显示屏上。） 

EGL 不提供锁定/解锁调用，而是由用户发出绘制命令，然后调用 eglSwapBuffers() 来提交当前帧。方法名称来自传统的前后缓冲区交换，但实际实现可能会有很大的不同。 

一个 Surface 一次只能与一个 EGLSurface 关联（你只能将一个生产方连接到一个 BufferQueue），但是如果您销毁该 EGLSurface，它将与该 BufferQueue 断开连接，并允许其他内容连接到该 BufferQueue。 

通过更改“当前” EGLSurface，指定线程可在多个 EGLSurface 之间进行切换。一个 EGLSurface 一次只能在一个线程上处于当前状态。 

要从原生代码创建 EGL 窗口 Surface，可将 EGLNativeWindowType 的实例传递到 eglCreateWindowSurface()。EGLNativeWindowType 是 ANativeWindow 的同义词，你可以自由地在它们之间转换。

下面是代码示例：
```C++
    Surface* mSurface;
    SurfaceComposerClient* mComposerClient;
    SurfaceControl* mSurfaceControl;

    mComposerClient = new SurfaceComposerClient;

    mSurfaceControl = mComposerClient->createSurface(
                String8("Test Surface"), width, height, PIXEL_FORMAT_RGBA_8888, 0);

    mSurface = mSurfaceControl->getSurface();

    ANativeWindow* anw = static_cast<ANativeWindow*>(mSurface);

    EGLSurface mEglSurface;
    mEglSurface = eglCreateWindowSurface(mEglDisplay, mEglConfig, (EGLNativeWindowType)anw, NULL);
```

最终，在调用 eglSwapBuffers() 时，EGL 模块会使用 ANativeWindow 对象的 queueBuffer() / dequeueBuffer() 方法来交换 Buffer，提交数据。GLES 模块会使用 ANativeWindowBuffer 对象的 lock() / unlock() 方法来锁定和解锁当前 Buffer。

## EGLImage

EGLImage 是一块可以由 GPU 直接渲染的缓冲区，可以附着到一个 Texture 或 FBO ID 上，然后由 GLES 将绘图结果渲染到下面绑定着的一个 ANativeWindowBuffer 中。所以，生成 EGLImage 需要的是一个 ANatvieWindowBuffer 对象。

下面是代码示例：
```C++
// .......
GraphicBuffer* buffer = new GraphicBuffer(width, height, 
                                          PIXEL_FORMAT_RGBA_8888,
                                          GraphicBuffer::USAGE_SW_WRITE_OFTEN |
                                          GraphicBuffer::USAGE_HW_TEXTURE);

unsigned char* bits = NULL;
buffer->lock(GraphicBuffer::USAGE_SW_WRITE_OFTEN, (void**)&bits);
// Write bitmap data into 'bits' here
buffer->unlock();

// Create the EGLImageKHR from the native buffer
EGLint eglImgAttrs[] = { EGL_IMAGE_PRESERVED_KHR, EGL_TRUE, EGL_NONE, EGL_NONE };
EGLImageKHR img = eglCreateImageKHR(eglGetDisplay(EGL_DEFAULT_DISPLAY), EGL_NO_CONTEXT,
                                    EGL_NATIVE_BUFFER_ANDROID,
                                    (EGLClientBuffer)buffer->getNativeBuffer(),
                                    eglImgAttrs);

// Create GL texture, bind to GL_TEXTURE_2D, etc.

// Attach the EGLImage to whatever texture is bound to GL_TEXTURE_2D
glEGLImageTargetTexture2DOES(GL_TEXTURE_2D, img);
```

最终，GLES 模块会调用 ANativeWindowBuffer 对象里的 lock() / unlock() 方法去使用该 Buffer。

## 接口(interface)实现的问题

因为 Android 图形系统采用硬件加速，所以 EGL / GLES 模块是 ANativeWindowBuffer 接口和 ANativeWindow 接口最主要的使用者。

因为 EGL / GLES 的函数都是 C 语言接口，参数也是 C 语言的结构体指针，所以可以基本假定这两个模块都是用 C 语言实现。所以如果用 C++ 语言实现 ANativeWindowBuffer 接口和 ANativeWindow 接口，就要考虑跨越 C/C++ 语言环境的问题。

在 Surface 类就设计了三个层次：
* ANativeWindow 接口是一个 C 语言结构体，里面的成员是函数指针。也就是用 C 语言定义了一个接口(interface)。Surface 类中包含了这个结构体，也就是 Surface 类是 ANativeWindow 接口的一个实现。
* Surface 类的头部单一继承了 ANativeWindow 接口。后面针对是 ANativeWindow 接口里的函数指针，定义了一批静态方法。类里的静态方法，参数个数和 C 语言定义的一样，C++编译器不会在参数表前添加本身的对象指针"this"。Surface 类在初始化时，将这批静态方法的函数地址赋给 ANativeWindow 接口里的函数指针。也就是 Surface 类实现了 ANativeWindow 接口，在 C 语言环境里调用也不会出现问题。在这批静态方法里，就是将传入的结构指针传换成为类指针，接着调用 Surface 类里该功能的具体实现。
* Surface 类针对是 ANativeWindow 接口里的方法，定义了一批普通的类方法。在这些方法里，对 ANativeWindow 接口的功能做了具体的实现。上面这批静态方法就是转换和调用到这批普通的类方法里。

经过这样三个层次的设计，ANativeWindow 接口就可以跨越 C/C++ 语言环境被调用到了。GraphicBuffer 类也是用相似的方法实现 ANativeWindowBuffer 接口。

## 限制性问题

在绘图或合成时，当我们有内容要向一个 gralloc buffer 绘制，我们必须用它创建一个 EGLImage，并创建一个 GL 纹理对象包装这个 EGLImage。 

我们没有的好方法从一个 GL 纹理中取消附着 gralloc buffer。一种方法是用另外一个新的 gralloc buffer 去附着这个 EGLImage，从而释放出旧的 gralloc buffer。 

另一种方法是只销毁 GL 纹理对象（然后每次重新创建它）。但这会有性能问题。

这是 GLES 函数接口的限制。如果我们想要在程序里使用多个 OpenGL 纹理，而每个纹理附着的 Buffer 想使用 Producer-Consumer 模式进行多进程/线程并发处理时，这会是一个问题。相比之下，SurfaceFlinger 并不需要担心这个，因为它仅使用一个 OpenGL 纹理，这样，当它为了它的合成绑定一个新的 GraphicBuffer 时，就会自动解除前一个 GraphicBuffer 的绑定！ 

这在实现新的 Buffer Queue 时会有影响。FireFox OS 在复用 Android HAL 代码时，在这里就碰到了问题。因为 FireFox Engine 是并发使用多个 OpenGL 纹理。使用时，只能用一个新的 gralloc buffer 顶出旧的 gralloc buffer，而旧的 gralloc buffer 不能主动释放。这样，生产者有一段时间可能会因为没有 Buffer 可用而陷入等待。


