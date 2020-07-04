# Android 的图形系统客户端的设计
* * *

# 应用程序的绘图流程

从以 Buffer Queue 为中心的 Producer-Consumer 模式看，客户端的绘图流程大致分为：
```
A. 绘图阶段
   1. lock buffer
   2. render buffer
   3. unlock buffer
B. 提交阶段
   1. queue buffer
   2. dequeue buffer
```

这个绘图流程，不论是对于 CPU 绘图，还是 GPU 绘图，或者是 codec 解码视频，都是一样。不同的硬件只是在 render buffer 时所用的方法不一样，但是都有 lock / unlock / queue / dequeue buffer 这些调用序列。

对于 EGL/OpenGLES 程序，一个绘图循环的结束的标志，就是调用了 eglSwapBuffers() 函数。该函数在底层就会执行 queue front buffer / dequeue back buffer 这一对调用序列。front buffer 入队，意味着刚被渲染好的帧被提交给显示服务器，将被显示到屏幕上。back buffer 出队，意味着有新的缓冲区提供给应用做渲染，将做为下一帧显示。显示服务器，在 Android 显示系统里就是 surfaceflinger service。

严格地说，应用程序的绘图循环可以有自己的节拍，节拍也可以不规律。显示服务器都要应付各种异常情况。但是，如果应用程序的绘图循环的节拍如果和显示服务器同步，听从显示服务器的指挥，则会使得应用程序的画面更顺滑流畅。

这样的编程需要了解 2 个方面的内容：
* Choreographer
* Frame Pacing Library

# Choreographer

Choreographer 用以协调动画、输入和绘图的时间安排。

Choreographer 从显示子系统接收定时脉冲(例如垂直同步信号 VSYNC)，做为显示系统的一部分被触发，然后为显示下一帧调度任务。

应用程序通常使用动画框架或视图层次结构中更高级别的抽象来间接地与 Choreographer 交互。

Choreographer 的引入，主要是基于 HW VSYNC，给应用程序的渲染动作在恰当的时机提供一个 VSYNC 事件。也就是 VSYNC 事件到来的时候，就是应用程序的渲染并提交帧的时机。对于 EGL/OpenGLES 程序，以调用 eglSwapBuffers() 函数表示为一个循环的结束。显示子系统通过对 VSYNC 信号周期的调整，来控制每一帧绘制操作的时机。

为了使得显示更加顺滑流畅，新版本的 Android 显示系统要能应付更高刷新率的情况，如 90Hz / 120 Hz 等。在 Android 11 以后，还要能处理应用可变刷新率。这时应用程序不能再做刷新率为 60Hz 这个假设。实现新功能需要做不少工作，挑战更大。具体见[ High refresh rate rendering on Android ](https://android-developers.googleblog.com/2020/04/high-refresh-rate-rendering-on-android.html)一文的说明。

一般关于 Choreographer 的文档都是说明位于客户端的 JAVA 层的代码。Choreographer 框架分布于 Producer-Consumer 模式的两端，是一个运行关系很复杂的系统。要理解 Android 显示系统就必须对 Choreographer 框架的设计有了解。Choreographer 框架的具体工作原理见《Android 的图形系统的总体设计》的[ VSYNC 模块的设计](../general-design/VSYNC.md)中的内容。

# Frame Pacing Library

Android Frame Pacing 库以前称为 Swappy，是 Android 游戏 SDK 的一部分。它帮助使用 OpenGL 和 Vulkan 的游戏在 Android 上采用正确的帧速度并实现平滑渲染。

帧间隔(Frame pacing)将游戏逻辑和渲染循环与操作系统的显示子系统和底层显示硬件的同步。Android 显示子系统的设计是为了避免某些视觉瑕疵，比如撕裂。显示子系统通过执行以下操作来避免撕裂：
* 内部缓存过去的帧
* 检测延迟的帧提交
* 当检测到延迟帧时继续显示当前帧

不一致的帧显示时间是由于游戏的渲染循环以不同于本机显示硬件支持的速率运行所导致的。当游戏的渲染循环对于底层显示硬件运行得太慢，导致显示时间不一致时，就会出现问题。例如，当以 30 fps 的速度运行的游戏试图在本机支持 60 fps 的设备上进行渲染时，游戏的渲染循环会导致重复帧在屏幕上多停留 16ms。这种类型的断开会在帧时间上造成大量不一致，如 33ms、16ms、49ms 等等。过于复杂的场景进一步加剧了这个问题，因为它们会导致错过帧。

Frame Pacing 库执行以下任务：
* 弥补由于游戏打顿造成的短帧(图像的显示时长不够)。
  + 添加显示示时间戳，以便帧能够按时而不是提前显示。
  + 使用显示时间戳扩展 EGL_ANDROID_presentation_time 和 VK_GOOGLE_display_timing。
* 对打顿和延迟的长帧(图像的显示时间过长)使用同步栅栏。
  + 向应用程序注入等待操作。这样可以让显示管道跟上，而不是让负荷增大。
  + 使用同步同步栅栏(EGL_KHR_fence_sync 和 VkFence)。
* 如果你的设备支持多种刷新率，选择合适的刷新率以提供灵活和流畅的显示。
  + 使用[ frame stats ](https://developer.android.com/games/sdk/frame-pacing/#frame_stats)提供帧状态信息，以便进行调试和分析。

# 代码复用

应用程序要想在 C++ 层复用 Android 的图形系统，如 ANativeWindow，则需要了解 Window 更多的概念。

在图形系统中，Window 这个概念实际上分为联系紧密的两层。在上层，Window 是做为输入输出的聚合对象提供给应用。在下层，Window 是做为可绘图的对象提供给应用。

对于上层概念，就有了 Widget / layout / input focus / event process 这些概念。再往上堆叠的就是 GTK / QT 这些被称之为图形库、控件库的东西。

对于下层概念，对应的就是 Cairo / Skia / hwui 这些被称之为渲染库的东西。渲染库只是单纯地渲染图像，不会处理输入事件。

对于 ANativeWindow，也只能单纯地用于渲染图像，没有输入焦点，光标处理这些东西。Android 的图形系统，上层概念是在 JAVA 层完成的。如果要想在 C++ 层复用 Android 的图形系统，则需要用 C++ 代码补齐这些东西，然后再往上移植 GTK / QT 这些图形库、控件库。这样，原先的 Linux 世界里的东西就可以在 Android 中运行起来。

# 参考文件

1. [Graphics Architecture](https://source.android.com/devices/graphics/architecture)
1. [Developer Reference : Choreographer](https://developer.android.com/reference/android/view/Choreographer)
1. [NDK Reference : Choreographer](https://developer.android.com/ndk/reference/group/choreographer)
1. [Frame Pacing Library Overview](https://source.android.com/devices/graphics/frame-pacing)
1. [High refresh rate rendering on Android](https://android-developers.googleblog.com/2020/04/high-refresh-rate-rendering-on-android.html)
1. [Android Game SDK](https://android-developers.googleblog.com/2019/12/android-game-sdk.html)
1. [Not all 120Hz smartphone displays are made equally — here’s why](https://www.androidauthority.com/120hz-displays-1112345/)
1. [Android 基于 Choreographer 的渲染机制详解](https://zhuanlan.zhihu.com/p/87954949)
1. [Android Choreographer 源码分析](https://www.jianshu.com/p/996bca12eb1d/)

