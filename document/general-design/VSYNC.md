# VSYNC 模块的设计
* * *

一般说来，在显示设备上都是按照一定的周期频率刷新显示内容。这个周期信号，在电子显像管时代叫作“VSYNC”。现在在液晶面板上，物理上已经没有这个信号，但是相应的概念和术语保留了下来。

设备显示按一定周期频率刷新，在手机和平板电脑上通常为每秒 60 帧。如果显示内容在刷新期间更新，则会出现撕裂现象；因此，请务必只在周期之间更新内容。在可以安全更新内容时，系统便会收到来自显示设备的信号。在 Android 系统中，这个信号由驱动程序定期上报。由于历史原因，我们仍将该信号称为 VSYNC 信号。 

刷新频率可能会随时间而变化，例如，一些移动设备的刷新率范围在 58 FPS 到 62 FPS 之间抖动，具体要视当前条件而定。对于连接了 HDMI 的电视，刷新率在理论上可以下降到 24 Hz 或 48 Hz，以便与视频相匹配。由于每个刷新周期只能更新屏幕一次，因此以 200 FPS 的刷新率为显示设备提交缓冲区只是在做无用功，因为大多数帧永远不会被看到。SurfaceFlinger 不会在应用提交缓冲区时执行显示操作，而是在显示设备准备好接收新的缓冲区时才会被唤醒。 

当 VSYNC 信号到达时，SurfaceFlinger 会遍历它的帧层列表，以寻找新的缓冲区。如果找到新的缓冲区，它会获取该缓冲区；否则，它会继续使用以前获取的缓冲区。SurfaceFlinger 总是需要可显示的内容，因此它会保留一个缓冲区。如果在某个层上没有提交缓冲区，则该层会被忽略。 

SurfaceFlinger 在收集可见层的所有缓冲区之后，便会询问 Hardware Composer 应如何进行合成。然后分别用 GPU 和 HWC 分别合成不同的帧层，最后合成好的内容也由 HWC 提交给真正的显示设备去显示。

以上就是显示内容从应用程序生成、到 SurfaceFlinger 合成、再到显示设备显示的简单过程。软件显示系统遵循这个频率工作十分必要。在设计上，就设计了 BufferQueue 系统使得 Producer-Consumer 两端可以并发工作，并且设计了 Choreographer 框架，使得 GPU 和 HWC 这些硬件设备能负载均衡。这就是著名的“Project Butter”的设计目标。所以，VSYNC 信号就是整个 Android 图形系统的工作节拍，或者说是编舞者(Choreographer)。因此弄明白 VSYNC 信号的工作流程就显得特别重要。

# VSYNC 的相位偏移信号

Choreographer 框架要协调 3 类事件的周期启动时间：
1. HW_VSYNC_0 - 屏幕开始显示一帧内容。
2. VSYNC - 应用提交已生成帧，并开始生成下一帧。
3. SF_VSYNC - SurfaceFlinger 开始为已生成帧进行合成。

这 3 类事件如果在同一时间点爆发，就会造成系统堵塞。于是这 3 类事件需要交错启动。同时，还要考虑应用提交的已生成帧，需要在最短的时间内显示到屏幕上。于是 VSYNC 的相位偏移信号因需而生。

其中，HW_VSYNC_0 对应的是硬件 VSYNC 信号，是时间基准。需要协调的是 VSYNC 和 SF_VSYNC 相对于 HW_VSYNC_0 的相位偏移。

因为在文档里用到 VSYNC 的术语比较多，下面对本文里的 VSYNC 相关术语做一个总结：
* HW VSYNC : 由显示设备提供的硬件信号，表示缓冲区里的内容可以安全送往显示设备上。在 Android 文档里也称为 HW_VSYNC_0。
* SW VSYNC : 由软件生成的、和上述 HW VSYNC 同步的信号。之所以有 SW VSYNC 信号，一是因为收到 HW VSYNC 有延迟，二是因为如果常开 HW VSYNC 信号，对功耗有影响。而且有些古老的硬件也不提供 HW VSYNC。在 Android 的文档里没有提到这个术语。这个 SW VSYNC 信号就是由 DispSync 模型控制的 DispSyncThread 线程周期性产生。
* VSYNC & SF VSYNC : 基于 SW VSYNC 信号提供的带相位偏移的 DISPLAY VSYNC 事件。在 Android 文档里，事件和信号这两个术语的意思相差无几。这个事件由 EventThread 线程向 DispSyncThread 线程订阅相位偏移后，被 DispSyncThread 线程定时唤醒，然后对外发布事件。Android 文档里如果没有特别的说明( HW / SW )，指的就是这两个术语。
  - VSYNC : 指示应用软件开始为绘图工作了。这是应用得最多的事件。在 Android 文档里大多数也指的是这个事件。但在实际代码和 Systrace 的输出里，这个事件被命名为"VSYNC-app"。
  - SF VSYNC :  指示 SurfaceFlinger 开始为合成工作了。但在实际代码和 Systrace 的输出里，这个事件被命名为"VSYNC-sf"。

本节讨论的是 EventThread 发送的 VSYNC 事件。该事件(VSYNC-app or VSYNC-sf)相对于标准的 HW VSYNC 信号有一个人为的相位偏移。这是因为当 HW VSYNC 信号发生时，系统正在往显示设备传送内容，系统处于忙碌状态。所以应用绘制触发事件(VSYNC-app)和 Surfaceflinger 的合成绘制触发事件(VSYNC-sf)有不同的相位偏移，能使三者错开时间运行。这样可以使得整个系统的负载均衡，从而提高用户体验。

通过 VSYNC 相位偏移，应用处理输入并渲染帧，SurfaceFlinger 接收缓冲区并合成帧，HWC 显示合成帧，所有这些操作都在一个时间段内完成。准确地说，应用和 SurfaceFlinger 渲染循环与硬件 VSYNC 同步，在一个 VSYNC 时段内，显示设备开始显示帧 N，而 SurfaceFlinger 开始为帧 N+1 合成窗口，应用处理等待的输入并生成帧 N+2。

## 相位偏移的设置

VSYNC 和 SF_VSYNC 的相位偏移，在系统属性中设置，由 PhaseOffsets 类管理：
* ro.surface_flinger.vsync_event_phase_offset_ns
* ro.surface_flinger.vsync_sf_event_phase_offset_ns

在 frameworks/native/services/surfaceflinger/Scheduler/PhaseOffsets.cpp 里，这两个值在类构造时被赋予两个变量：
```C++
    int64_t vsyncPhaseOffsetNs = vsync_event_phase_offset_ns(1000000);
    int64_t sfVsyncPhaseOffsetNs = vsync_sf_event_phase_offset_ns(1000000);
```

这两个属性的值需要根据高负载用例进行保守设置，例如在窗口过渡期间进行部分 GPU 合成或 Chrome 滚动显示包含动画的网页。这些偏移允许较长的应用渲染时间和较长的 GPU 合成时间。 

变量 vsyncPhaseOffsetNs 是应用软件 VSYNC 事件相对于 HWComposer 上报的 HW VSYNC 事件的相位偏移量（以纳秒计）。在 SurfaceFlinger 和基于 Choreographer 特性的应用程序在运行每一帧时，应用软件 VSYNC 事件都会产生。

此相位偏移允许最小延迟的调整，范围从应用程序（被 Choreographer 框架）唤醒的时间，到窗口图像最终结果被显示的时间。该值可以是正值（在 HW VSYNC 之后）或负值（在 HW VSYNC 之前）。将其设置为'0'将导致两个 VSYNC 周期的最小延迟，因为 App 和 Surfaceflinger 将在 HW VSYNC 到达时同时运行。将其设置为正数将导致最小延迟为(**这是十分重要的公式**)：
```C++
(2 * VSYNC_PERIOD - (vsyncPhaseOffsetNs % VSYNC_PERIOD))
```

请注意，减少这延迟量更可能会使应用程序没有及时准备好窗口内容映像。当这种情况发生时，延迟将最终导致增加一个额外的 VSYNC 周期，动画将停顿。因此，应该保守地调整这延迟量（或者至少在意识到正在进行权衡的情况下）。

变量 sfVsyncPhaseOffsetNs 是 SurfaceFlinger 执行合成操作时的相位偏移。

在一些手机里的两个值的列表：(单位：毫秒)
| 变量                 | 模拟器 | Nexus 5 | Pixel 3 |
|:--------------------:|--------|---------|---------|
| vsyncPhaseOffsetNs   | 0      | 7.5     | 2       |
| sfVsyncPhaseOffsetNs | 5      | 5       | 6       |

这两个值对显示延迟的影响很大。最终的设计造成的延迟体现在上面那个公式里。下面对此做一个分析。

如果我们把 Android 的模拟器看成一个理想情况，则有下图这样的显示节拍：

![VSYNC 理想情况](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/general-design/DispSync-02.svg)

在这种情况里，应用软件绘制的工作在 SurfaceFliner 合成工作开始前完成，这样，应用绘制的内容就可以在下一个 HW SYNC 信号到达时得到显示。这样的延迟最小。

但这是理想情况。而实际项目不能这样考虑。能在几个毫秒里完成绘制工作的一定是小程序，内容简单，人们也不会在乎它是否流畅。而人们在乎的那些软件，往往就是高负载的软件，例如，大型游戏、VR 游戏、视频播放器等等。因此相位偏移量需要从高负载的场景进行设置。

![VSYNC 通常情况](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/general-design/DispSync-03.svg)

因为高负载的软件，很可能在一个周期时间里是完不成绘制一帧的工作的。因此在实际项目里，要确保的是 SurfaceFlinger 的合成工作，在系统高负载的情况下，在一个周期时间里一定能完成。这样，在多进程/线程的工作环境里，在上一个周期时间能提交新的绘制内容的窗口，将在下一次 HW VSYNC 信号到达时能显示出去。所以，SurfaceFlinger 的合成工作，比应用的绘制工作要先启动，这样合成操作在一个周期时间里有更多的工作时间和资源。

所以，应用软件预测现在开始绘制的帧如果能在一个节拍内完成，要显示到屏幕上的最小延迟就是这个公式：
```C++
(2 * VSYNC_PERIOD - (vsyncPhaseOffsetNs % VSYNC_PERIOD))
```

所以，在一个 VSYNC 时间节拍中，就有了这样的显示管线：显示设备开始显示帧 N，而 SurfaceFlinger 开始为帧 N+1 合成窗口，应用处理等待的输入并生成帧 N+2。这是通常情况。

所以对这两个值的设置需要十分谨慎。超过一两毫秒的延迟时间是非常明显的。因此建议集成彻底的自动化误差测试，以便在不显著增加误差计数的前提下最大限度减少延迟时间。 

# VSYNC 信号的生成与传播

VSYNC 信号可同步显示管道。显示管线由应用渲染、SurfaceFlinger 合成以及用于在屏幕上显示图像的硬件混合渲染器 (HWC) 组成。VSYNC 可同步应用唤醒以开始渲染的时间、SurfaceFlinger 唤醒以合成屏幕的时间以及屏幕刷新周期。这种同步可以消除卡顿，并提升图形的视觉表现。

HWC 可生成 VSYNC 事件并通过回调将事件发送到 SurfaceFlinger：
```C++
typedef void (*HWC2_PFN_VSYNC)(hwc2_callback_data_t callbackData,
        hwc2_display_t display, int64_t timestamp);
```

SurfaceFlinger 通过调用 setVsyncEnabled() 来控制 HWC 是否生成 VSYNC 事件。SurfaceFlinger 使 setVsyncEnabled() 能够生成 VSYNC 事件，因此它可以与屏幕的刷新周期同步。当 SurfaceFlinger 同步到屏幕刷新周期时，SurfaceFlinger 会停用 setVsyncEnabled 以阻止 HWC 生成 VSYNC 事件。如果 SurfaceFlinger 检测到实际 VSYNC 与它先前建立的 VSYNC 之间存在差异，则 SurfaceFlinger 会重新启动 VSYNC 事件生成过程。

注意：HWC 从 HAL_PRIORITY_URGENT_DISPLAY 的线程触发 hwc2_callback_data_t，在实现时要求延迟时间尽可能短，通常要求小于 0.5 毫秒。

在 Android 系统引入 HIDL 层之后，这个问题显得更加复杂，见下面 HWComposer 的从 HAL 到 Framework 层的关系组合图：

![HWComposer 的关系组合图](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/general-design/services_surfaceflinger_DisplayHardware_HWComposer%20Component%20Diagram.svg)

下面是 HW VSYNC 信号回调的调用序列示意图：

![HW VSYNC 信号传播序列](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/general-design/services_surfaceflinger_SurfaceFlinger%20Component%20Diagram%20-%20ComposerCallback.svg)

这张图显示了 HW VSYNC 信号从 hwc2_device_t 到 SurfaceFlinger 的传播路径。从图中可以看出，传播链很长，还有跨进程的操作，调用延迟是少不了的。最终，这个问题使得 DispSync 模型的产生。具体见下面的分析。

# VSYNC 模块的总体设计

VSYNC 模块的代码主要位于 frameworks/native/services/surfaceflinger/Scheduler/ 目录。如其名字，调度器(Scheduler)，和 OS 使用硬件时钟做调度器一样，surfaceflinger 使用 VSYNC 信号做图形系统的调度器。

VSYNC 模块是典型的 Listener 模型，同样也是 Producer-Consumer 模型。

## 主要的接口和类列表

Scheduler/ 目录大致包含如下接口和类：

* DispSync - 维护显示的基于硬件的周期性 VSYNC 事件的模型，并使用该模型在 HW VSYNC 事件的特定相位偏移处执行周期回调。
* DispSync::Callback - DispSync 的回调接口，实际上就是 HW VSYNC 事件的消费者，有如下实现：
  + DispSyncSource - 提供基于 HW VSYNC 事件特定相位偏移处的 DISPLAY SYNC 事件源。有两个对象实例：
    - vsyncSrc - 为 mEventThread 提供"VSYNC-app"事件。
    - sfVsyncSrc - 为 mSFEventThread 提供"VSYNC-sf"事件。
  + SamplingOffsetCallback - 在空闲时刻，基于 HW VSYNC 事件特定相位偏移处对屏幕图像进行亮度(luma)采样。事件名字为"SamplingThreadDispSyncListener"。
  + ZeroPhaseTracer - DispSync 模型自身跟踪零偏移的时间数据，用于分析和调试。事件名字为"ZeroPhaseTracer"。
* EventControlThread - VSYNC 信号开关。
* EventThread - 通过回调给外界提供带相位偏移的 DISPLAY SYNC 事件的线程。该类有 2 个对象实例，一个线程提供一种有名字的 VSYNC 事件。
  + mEventThread : 给外界提供"VSYNC-app"事件。在Android文档里，这个事件又称为"VSYNC"事件。
  * mSFEventThread : 给外界提供"VSYNC-sf"事件。在Android文档里，这个事件又称为"SF VSYNC"事件。
* IdleTimer - 设置给定时间间隔的计时器类，并在时间间隔到期时调用回调。该类主要是提供给 RegionSamplingThread 判断采样时机。
* LayerHistory - 该类表示有关被视为当前图层的信息。
* LayerInfo - 该类表示有关 Individial Layers 的信息。
* MessageQueue & LambdaMessage - 主动对象之间通信的接口。
  + MessageQueue - SurfaceFlinger 向 mSFEventThread 注册回调函数，以便 surfaceflinger 主线程定时接收到 VSYNC-sf(INVALIDATE/REFRESH) 事件。
  + LambdaMessage - 在 Client / Server 模式下，在 Binder 线程中的类与主线程中的 SurfaceFlinger 通信的接口。
* PhaseOffsets - 封装特定相位偏移的接口。
* RefreshRateConfigs - 该类用于封装刷新率的配置。它保存有关设备上可用刷新率的信息，以及数字和人类可读名称之间的映射。
* RefreshRateStats - 该类用于封装显示正在使用的刷新率的统计信息。
* Scheduler - 基于 VSYNC 事件的调度器。
* VSyncModulator - VSYNC 调制器，根据当前 SurfaceFlinger 状态调整 VSYNC 偏移量。
* VSyncSource - DISPLAY SYNC 事件源接口，在周期事件发生时回调执行，有如下实现：
  + DispSyncSource
  + InjectVSyncSource - SurfaceFlinger 在回放屏幕画面时，模拟 DISPLAY SYNC 事件输入。
* VSyncSource::Callback - DISPLAY SYNC 事件的消费者，有如下实现：
  + EventThread

## 主动对象与线程

在多线程编程中，把对象分为主动对象(Active Object)和被动对象(Passive Object)。主动对象内部包含一个线程，可以自动完成动作或改变状态。而一般的被动对象只能通过被其他对象调用才有所作为。在多线程程序中，经常把一个线程封装到主动对象里面。

在 surfaceflinger 程序代码中总共有 6 个主动对象类，全部都涉及 VSYNC 信号的处理：
* SurfaceFlinger : HW VSYNC 信号的采集方，以及 DISPLAY SYNC 事件之一(VSYNC-sf)的消费方。
* DispSyncThread : 由 DispSync 模型控制的线程类。用途是周期性产生标准的 SW VSYNC 信号，并且根据 EventThread 对象实例注册的相位偏移量，为 EventThread 对象实例发送可用的带相位偏移 DISPLAY VSYNC 事件。
* EventControlThread : VSYNC 信号开关。只有一个对外的 setVsyncEnabled() 方法。
* EventThread : 向 DispSyncThread 对象实例注册的相位偏移量，给外界提供带相位偏移的 VSYNC 事件的类。
* IdleTimer : 定时线程，周期性调用 RegionSamplingThread::checkForStaleLuma()。
* RegionSamplingThread : 在 VSYNC 事件的特定相位偏移处采样，计算图像亮度(luma)。

除了(程序里限定的4个) Binder 线程之外，在 SurfaceFlinger 进程里大约还有 8 个线程，全部都涉及 VSYNC 信号的处理：
* HW VSYNC Thread : HWC 的回调接口，由 SurfaceFlinger 对象实现。回调方法被调用时的环境，在 HAL_PRIORITY_URGENT_DISPLAY 线程里。HW VSYNC 事件生产者。
* DispSyncThread : 由 DispSync 模型控制的线程。该类只有一个对象实例。DISPLAY SYNC 事件生产者。
* EventControlThread : VSYNC 信号开关线程。该类只有一个对象实例。
* EventThread : 给外界提供带相位偏移的 VSYNC 事件的线程。该类有 2 个对象实例，一个线程提供一种有名字的 VSYNC 事件。DISPLAY SYNC 事件生产者。
  * mEventThread : 给外界提供"VSYNC-app"事件。在Android文档里，这个事件又称为"VSYNC"事件。该事件的消费者是上层应用程序，封装在 Choreographer 框架里。
  * mSFEventThread : 给外界提供"VSYNC-sf"事件。在Android文档里，这个事件又称为"SF VSYNC"事件。
* Main Thread : SurfaceFlinger 对象，只有一个实例。接收"VSYNC-sf"事件，触发合成操作。"VSYNC-sf"事件消费者。
* IdleTimer : 该类只有一个对象实例。
* RegionSamplingThread : 该类只有一个对象实例。带相位偏移的 VSYNC 事件的消费者。

下图是涉及 VSYNC 信号的各个主动对象的协作图：

![DispSync Component Diagram](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/general-design/services_surfaceflinger_Scheduler_DispSync%20Component%20Diagram.svg)

其中 VSYNC 信号的流动见下一节的说明。

下面是在 DispSync 模块中其它回调函数列表：
| callback                          | belong to          | provided             | required  |
|-----------------------------------|--------------------|----------------------|-----------|
| SetVSyncEnabledFunction           | EventControlThread | SurfaceFlinger       | Scheduler |
| ResyncCallback                    | EventThread        | SurfaceFlinger       | Scheduler |
| InterceptVSyncsCallback           | EventThread        | SurfaceInterceptor   | Scheduler |
| ResetCallback                     | IdleTimer          | RegionSamplingThread | IdleTimer |
| TimeoutCallback                   | IdleTimer          | RegionSamplingThread | IdleTimer |
| GetCurrentRefreshRateTypeCallback | Scheduler          | SurfaceFlinger       | Scheduler |
| ChangeRefreshRateCallback         | Scheduler          | SurfaceFlinger       | Scheduler |
| GetVsyncPeriod                    | Scheduler          | SurfaceFlinger       | Scheduler |

## VSYNC 信号的流动

因为封装了线程，换个角度看，一个主动对象会分两个部分：
* 线程循环部分，完成该对象的设计目标。
* 对外接口部分，外届可以从这些接口中得到或修改内部状态。

在对于 VSYNC 信号的处理过程中，各个线程和各个相关类就互相牵扯，在一个主动对象里的线程循环部分调用另外一个主动对象的对外接口部分，这是很频繁的事情。这也因此完成了线程间的信息传递。在上面的协作图顺着信号传递的路径走，才能比较容易理解 VSYNC 模块的设计。

做为 HW VSYNC 的开关，EventControlThread类的接口很简单，也分成内外两个部分。
```C++
HW VSYNC enable/disable

EventControlThread outside portion:
called from other thread:
EventControlThread::setVsyncEnabled(true/false)

EventControlThread inside portion:
00) EventControlThread::threadMain()
  01) SurfaceFlinger::setPrimaryVsyncEnabled(enabled);
    02) HWComposer::setVsyncEnabled()
      03) // from HIDL to HAL : HWC2_PFN_VSYNC()
```

在 HW VSYNC 线程，对于 HW VSYNC 信号的接收路径见前面的[ VSYNC 信号的生成与传播](#VSYNC-信号的生成与传播)一节。

在 DispSyncThread 线程，发送 DISPLAY VSYNC 信号流程如下：
1. DispSyncThread
2. DispSync::Callback(可以注册多个带相位偏移的事件发生源)
3. DispSyncSource(两个带相位偏移的事件发生源：vsyncSrc & sfVsyncSrc)
4. EventThread(两个带相位偏移的事件发生源线程。mEventThread & mSFEventThread)

代码调用流程如下：
```C++
SW VSYNC trigger
DispSync thread:
00) DispSyncThread::threadLoop()
  01) DispSyncThread::gatherCallbackInvocationsLocked()
  01) DispSyncThread::fireCallbackInvocations()
    02) DispSync::Callback::onDispSyncEvent(nsecs_t when)
      03) VSyncSource::onDispSyncEvent(nsecs_t when) == DispSyncSource::onDispSyncEvent(nsecs_t when) -> (vsyncSrc & sfVsyncSrc)
        04) VSyncSource::Callback::onVSyncEvent(nsecs_t timestamp) == EventThread::onVSyncEvent(nsecs_t timestamp) -> (mEventThread & mSFEventThread)
          05) makeVSync()
            06) DisplayEventReceiver::Event event;
            06) event.header = {DisplayEventReceiver::DISPLAY_EVENT_VSYNC, displayId, timestamp};
            06) event.hotplug.connected = connected;
          05) mPendingEvents.push_back()
```

在 EventThread(mEventThread & mSFEventThread) 线程，发送带相位偏移的 SW VSYNC ("VSYNC-app" & "VSYNC-sf") 事件流程如下：
1. EventThread
2. DisplayEventReceiver
3. BitTube

代码调用流程如下：
```C++
Event thread(mEventThread & mSFEventThread)
00) EventThread::threadMain()
  01) event = mPendingEvents.front()
  01) EventThread::dispatchEvent()
    02) IDisplayEventConnection::postEvent() == EventThreadConnection::postEvent()
      03) DisplayEventReceiver::sendEvents()
        04) BitTube::sendObjects()
          05) BitTube::write()
            06) ::send()
```

回到上面的协作图，从类的继承关系看，最终是客户端的 DisplayEventReceiver::getEvents() 方法收到了"VSYNC-app"事件：
```
Server side
EventThread::Connection --|> BnDisplayEventConnection --|> IDisplayEventConnection

Client side
IDisplayEventConnection <|-- BpDisplayEventConnection <--- DisplayEventReceiver

Server side                                                      Client side
DisplayEventReceiver::sendEvents() --(cross process / thread)--> DisplayEventReceiver::getEvents()
```

对于 SurfaceFlinger 主线程，也是使用 MessageQueue 类引用 IDisplayEventConnection 接口获得"VSYNC-sf"事件。
```C++
mEventQueue->setEventConnection(mScheduler->getEventConnection(mSfConnectionHandle));
```

MessageQueue 对象发送和接收"VSYNC-sf"事件的代码调用流程如下：
```C++
Main Thread: MessageQueue as server side
00) MessageQueue::cb_eventReceiver()
  01) MessageQueue::eventReceiver()
    02) MessageQueue::Handler::dispatchInvalidate()
      03) Looper::sendMessage(MessageQueue::INVALIDATE)
```

在下一次事件接收循环里，SurfaceFlinger 主线程接收"VSYNC-sf"事件的代码调用流程如下：
```C++
Main Thread: SurfaceFlinger as client side
00) SurfaceFlinger::run()
  01) SurfaceFlinger::waitForEvent()
    02) MessageQueue::waitMessage()
      03) Looper::pollOnce(-1)
        04) epoll_wait
        04) MessageQueue::Handler::handleMessage(MessageQueue::INVALIDATE)
          05) SurfaceFlinger::onMessageReceived(MessageQueue::INVALIDATE)
```

至此，就完成了 HW VSYNC 信号向 DISPLAY VSYNC 信号的转换和发射，以及 VSYNC 信号的最终用户(app & sf)的接收过程。

就这样，应用程序、SurfaceFliner 和显示设备三者就在 VSYNC 信号指挥下按照统一的节拍进行工作，这就是 Choreographer 框架的目标。Choreographer，翻译过来就是编舞者，功能如其名。

# DispSync 

DispSync 是 Scheduler 模块中的核心概念。DispSync 维护显示设备基于硬件的周期性 VSYNC 事件的模型，并使用该模型在硬件 VSYNC 事件的特定相位偏移处周期性执行回调。该模型是通过 addResyncSample() 方法向 DispSync 对象提供连续的硬件事件时间戳来构建的。实际上，DispSync 就是前面所说的 SW VSYNC 信号的生产方，然后再生成特定相位偏移处的 DISPLAY VSYNC 信号。

DispSync 实质上是一个软件锁相回路 (PLL)，它可以生成由 Choreographer 和 SurfaceFlinger 使用的 VSYNC 和 SF VSYNC 信号，即使没有来自硬件 VSYNC 的偏移也是如此。

## 工作原理

DispSync 模型的工作原理见下图：

![DispSync flow](https://source.android.com/devices/graphics/images/dispsync.png)

DispSync 输入输出如下： 
* 参考。HW_VSYNC_0。 
* 输出。VSYNC 和 SF VSYNC。在代码和 Systrace 的输出中，这两个事件分别被命名为"VSYNC-app" / "VSYNC-sf"。
* 反馈。来自 Hardware Composer 的退出栅栏(Retire fence)有信号状态的时间戳。

之所以会有 DispSync 模型，主要有如下几个原因：
* 硬件只提供一个 HW VSYNC 信号，对应于主显示刷新。其它信号都基于这个信号的固定偏移量触发。所以需要 DispSync 模型来维护这些相位偏移信号。
* 从前面的[ VSYNC 信号的生成与传播](#VSYNC-信号的生成与传播)一节可知，HW VSYNC 信号传播路径很长。而该信号的微小延迟都会影响屏幕刷新。例如用户用手指移动地图，HW VSYNC 信号的任何延迟都会被用户视为缓慢的触摸响应。
* 常开 HW VSYNC 信号上报机制也会对功耗有影响。

所以最终是用 DispSync 模型维护一个软件可变定时信号，SW VSYNC。该信号与 HW VSYNC 信号同步。然后在平常情况下关闭 HW VSYNC 信号上报机制。这样，因为没有在实际的 HW VSYNC 信号上做任何工作，Choreographer 的延迟更小，效率更高。

因为 HW VSYNC 信号周期根据负荷会有抖动。用户根据应用场景，也会切换屏幕的刷新率，如 30Hz / 60Hz / 90Hz。所以需要 DispSync 模型有一些优化的算法用来同步 HW VSYNC 和 SW VSYNC 这两者。

反馈回路图中的“退出栅栏(Retire fence)时间戳”指的是一种后验的校准算法。退出栅栏(Retire fence)中的时间戳表示的是一帧内容在屏幕显示的时间点。两帧之间的时间差就是 HW VSYNC 的周期时间。DispSync 代码将使用退出栅栏(Retire fence)中的时间戳来查看它的 SW VSYNC 是否同步。如果不同步，则将临时重新启用硬件信号，重新计算 HW VSYNC 的周期时间，直到 SW VSYNC 与之同步。

该模型通过使用经过 addPresentFence() 方法传递给 DispSync 对象的退出栅栏(Retire fence)对象的时间戳来验证。这些 Fence 时间戳应该与硬件 VSYNC 事件相符，但它们不必是连续的硬件 VSYNC 时间。如果此方法确定当前模型准确地表现了硬件事件的时间周期，就返回'false'以指示重新同步(通过addresynchsample)过程停止。

退出栅栏(Retire fence)的有信号状态时间戳必须与 HW VSYNC 相符，即使在不使用相位偏移的设备上也是如此。否则，实际造成的误差会严重得多。智能面板通常有一个增量：退出栅栏(Retire fence)是对显示设备内存进行直接内存访问 (DMA) 的终点，但是实际的显示切换和 HW VSYNC 会晚一段时间。

这个延迟，以前用宏 PRESENT_TIME_OFFSET_FROM_VSYNC_NS 在设备的 BoardConfig.mk Makefile 中设置。后来它改名为 present_time_offset_from_vsync_ns，在系统属性中设置。它基于显示控制器和面板特性。从退出栅栏(Retire fence)时间戳到 HW VSYNC 信号的时间，将以纳秒为单位进行测量。

DispSync 和模型计算相关的方法和成员有这么几类：
* 与退出栅栏(Retire fence)相关的方法。
* 计算模型的方法。
* 存储计算值的成员。

## 与退出栅栏(Retire fence)相关的方法

addPresentFence() 向 DispSync 对象添加一个用于验证当前 VSYNC 事件模型的退出栅栏(Retire fence)。当调用 addPresentFence() 时，无需检查 Fence 信号。当 Fence 发出信号时，其时间戳应该对应于硬件 VSYNC 事件。与 addResyncSample() 方法不同，连续的 Fence 的时间戳不需要对应于连续的硬件 VSYNC 事件。

当 SurfaceFlinger 调用到 HwComposer 对象中每个会影响显示的方法的时候，该方法将传入退出栅栏(Retire fence)参数。

## 计算模型的方法

三个方法：beginResync() / addResyncSample() / endResync()，被用来从硬件 VSYNC 事件中重新同步 DispSync 的模型。重新同步过程，首先调用 beginResync()，接着用一系列的连续的硬件 VSYNC 事件时间戳参数调用 addResyncSample()，最后，当 addResyncSample() 返回'false'值，指示不再需要更多的样本时，调用 endResync()。

每当显示器打开时（即打开后立即执行一次），以及每当 addPresentFence() 返回true时（指示模型已偏离硬件 VSYNC 事件），都应执行此重新同步过程。

## 存储计算值的成员

这些值都以纳秒计量：
* 估计值：mPeriod，也就是 DispSync 模型里正在工作的 SW VSYNC 的时钟周期。
* 预期值：mIntendedPeriod，SW VSYNC 事件的预期周期。在理想条件下，如果不与 mPeriod 相同，则应与 mPeriod 相似，加减一些观测误差。
* 待决值：mPendingPeriod，候选值，周期变化后所提议的时钟周期值。如果 mPendingPeriod 与 mPeriod 不同且不为零，当我们检测到硬件切换 VSYNC 频率时，它将刷新为 mPeriod。
* 相位偏差：mPhase，也就是 SW VSYNC 与 HW VSYNC 的偏差值。
* 参考时间：mReferenceTime，它是重新同步后第一个 HW VSYNC 事件的时间戳。每次计算下一个 SW VSYNC 时间点时，都以该时间作为基准，这样可以减少累积误差。
* 误差值：mError，估计值和校验值之间的差值，模型误差。它基于估计的 SW VSYNC 事件时间与在 mPresentFences 数组中观察到的事件时间之间的差异。
* 观测值：mResyncSamples，也就是通过 addResyncSample() 加入的后验的 HW VSYNC 的时间戳。
* 校验值：mPresentFences / mPresentTimeOffset。退出栅栏(Retire fence)发出信号时的时间戳。该时间戳一定是 HW VSYNC 的时间戳，所以可以校验估计值。当误差值超过阈值，则启动重新同步流程，直到误差值低于阈值为止。

几个周期值的关系见下图：
![DispSync-05](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/general-design/DispSync-05.svg)

# VSYNC 偏移 

图形系统的运作周期与 HW VSYNC 同步会实现一致的延迟时间。它可以减少应用和 SurfaceFlinger 中的错误，以及内外显示设备之间的相位漂移。但是，这要假定应用和 SurfaceFlinger 的每帧时间没有很大变化。尽管如此，一帧内容从提交到显示，延迟至少为两帧。 

为了解决此问题，可以通过使应用的渲染信号和合成信号与 HW VSYNC 相关，但不是同时发生，也就是有相位偏差，从而利用 VSYNC 偏移减少输入设备到显示设备的延迟。这是有可能的，因为应用渲染加合成通常需要不到 33 毫秒的时间。这就是 Choreographer 框架的设计目标。参见上面的显示管线并行操作的演示图。最终结果就是这个延迟公式：
```C++
(2 * VSYNC_PERIOD - (vsyncPhaseOffsetNs % VSYNC_PERIOD))
```

VSYNC 偏移的结果产生具有相同周期和相位偏移的三个信号： 
* HW_VSYNC_0。显示设备开始显示下一帧。 
* VSYNC。应用读取输入内容并生成下一帧。在代码和Systrace的输出中，该事件名为"VSYNC-app"。
* SF VSYNC。SurfaceFlinger 开始为下一帧进行合成。 在代码和Systrace的输出中，该事件名为"VSYNC-sf"。

注意：VSYNC 偏移会缩短可用于应用和合成的时间，因此增加了出错几率。具体见算法里的分析。

## 偏移算法

DispSync 模型里对于 VSYNC 信号，关注的是周期时间(period)。也就是 SW VSYNC 的周期时间与 HW VSYNC 的周期时间之间的误差要控制在一定的阈值以内。然后两者之间的关系，用相位偏移(phase offset)来表示。

为了简化设计，DispSync 模型做了如下假设：
* 时间"0"点是发生第一个 HW VSYNC 信号的时间点。
* 不管 HW VSYNC 信号发生的周期如何抖动，都假定当前计算出来的周期，是从时间"0"点起一直运行到现在的。所以在算法里大量地使用了整数的除法运算"/"和求余"%"运算。
* 也就是对于 HW VSYNC 真实发生的时间点，DispSync 模型实际上并不关心。而是假设，HW VSYNC 信号应该就发生被周期时间(period)整除，或余数(误差)很小的时间点上。于是 SW VSYNC 信号的发生就定在这个时间点上。也就是，只要周期同步，有相位偏差也没关系，屏幕显示就会顺滑。
* 于是，相位偏移(phase offset)就简化为求 SW VSYNC 的周期时间与 HW VSYNC 的周期时间之间的偏差。

计算周期时间(period)和相位偏移(phase offset)的算法在 DispSync::updateModelLocked() 里。

计算周期时间 mPeriod，至少需要 3 个采样点，能生成 2 个时间段(duration)，这样才能求平均值。在实践中，因为在一些设备上开始启动的两个数据有问题，所以需要抛弃。所以实践中至少需要 6 个采样点，能生成 3 个时间段(duration)，算出的平均值才有效。至多需要 32 个采样点，能生成 29 个有效时间段(duration)。最后还要抛弃最大值和最小值才能计算平均值。

```C++
        // Exclude the min and max from the average
        durationSum -= minDuration + maxDuration;
        mPeriod = durationSum / (mNumResyncSamples - numSamplesSkipped - 2);
```

计算偏差 mPhase 的算法有点意思：
* 它将一个周期时间 mPeriod 映射(放大)到一个半径为"1"的圆周里，
* 每个样本求余数，也就是相对于"0"点偏差。
* 所有样本的偏差通过三角函数加和求平均，就是平均偏差，也就是 mPhase。
* 这是一种归一算法，可以快速求偏差。
* 这样的算法可以避免开方"sqrt()"运算。也许在计算机系统里，三角函数的运算速度比开方"sqrt()"快。

```C++
        double sampleAvgX = 0;
        double sampleAvgY = 0;
        double scale = 2.0 * M_PI / double(mPeriod);
        for (size_t i = 0; i < mNumResyncSamples; i++) {
            size_t idx = (mFirstResyncSample + i) % MAX_RESYNC_SAMPLES;
            nsecs_t sample = mResyncSamples[idx];
            double samplePhase = double(sample % mPeriod) * scale;
            sampleAvgX += cos(samplePhase);
            sampleAvgY += sin(samplePhase);
        }

        sampleAvgX /= double(mNumResyncSamples);
        sampleAvgY /= double(mNumResyncSamples);

        mPhase = nsecs_t(atan2(sampleAvgY, sampleAvgX) / scale);
```

![DispSync phase offset](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/general-design/DispSync-01.svg)

mPhase 做为平均偏差值，代表着 mPeriod 的变化趋势：
* 在系统平稳的情况下，mPhase 的值应该趋向于"0"。也就是，各个样本在"0"点(理想的信号发生时间点)左右来回抖动，但均值为"0"。
* 如果 mPhase > 0，则代表着 mPeriod 有变大的趋势，意味着系统变慢，刷新频率在下降。
* 反之，则代表着 mPeriod 有变小的趋势，意味着系统变快，刷新频率在上升。

每次 DispSync 模型计算完 mPeriod & mPhase，都会调用 DispSyncThread::updateModel(mPeriod, mPhase, mReferenceTime) 方法将数值同步到 DispSyncThread 线程里。

## 校准算法

校验算法在DispSync::updateErrorLocked()里。最多记录最近的 8 个有状态的 Fence 的时间戳。该算法是用最小二乘法，求最小均方误差。
```C++
    int numErrSamples = 0;
    nsecs_t sqErrSum = 0;

    for (size_t i = 0; i < NUM_PRESENT_SAMPLES; i++) {
        nsecs_t sample = mPresentTimes[i];
        if (sample > mPhase) {
            nsecs_t sampleErr = (sample - mPhase) % period;
            if (sampleErr > period / 2) {
                sampleErr -= period;
            }
            sqErrSum += sampleErr * sampleErr;
            numErrSamples++;
        }
    }
```

每次 DispSync 模型被外部用 DispSync::addPresentFence() 添加退出栅栏(Retire fence)时，都会用 Fence 的时间戳来校验相位偏移(mPhase)。如果最小均方误差大于阈值，则打开 HW VSYNC，启动重新同步过程，直到误差小于阈值，停止重新同步过程。

# VSYNC 和 SF VSYNC 的生成

## 事件的生成

DispSyncThread 线程的工作原理很简单：就是睡眠，直到已注册的离当前最近的事件时间点到来；给感兴趣的回调函数发送完事件后，接着进入下一个睡眠循环。

![VSYNC 唤醒时机](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/general-design/DispSync-04.svg)

VSYNC 和 SF VSYNC 的生成的算法在 DispSyncThread::computeListenerNextEventTimeLocked() 里：
```C++
        nsecs_t baseTime = now;
        nsecs_t phase = mPhase + listener.mPhase;
        nsecs_t numPeriods = baseTime / mPeriod;
        nsecs_t t = (numPeriods + 1) * mPeriod + phase;
```
变量的含义很简单：
```
mPhase          : VSYNC 周期时间的偏差值。一般为"0"。
listener        : vsyncSrc or sfVsyncSrc
listener.mPhase : 同步事件源期望相对于 SW VSYNC 信号的偏移值。
                  vsyncPhaseOffsetNs   = 7500000
                  sfVsyncPhaseOffsetNs = 5000000
                  mPeriod              =16666667 - 60Hz
phase           : 发生同步事件时相对于 SW VSYNC 信号的实际偏移值。
t               : 下次发生同步事件的绝对时间点。
```
另外还有一些数值记录在这里：
```
mWakeupLatency    <= 1500000   - VSYNC 事件发生的实际时间点比理想时间点有延迟。评价 Linux real-time 性能用。 
kErrorThreshold    =  400000^2 - 最小均方误差阈值。
kPresentTimeOffset = PRESENT_TIME_OFFSET_FROM_VSYNC_NS = 0 - 退出栅栏(Retire fence) 的时间戳和 HW VSYNC 时间点有偏移。
```

# 小结

DispSync 模型的工作原理是：
* HW VSYNC 信号因为传播路径长，有延迟，而且常开功耗大，所以采用 SW VSYNC 信号定时发生。也就是 SurfaceFlinger 用定时器主动猜测 HW VSYNC 信号发生的时间点。这样既能减少延迟，也方便管理多个带相位偏差的事件。
* 退出栅栏(Retire fence)的时间戳，做为后验的 HW VSYNC 信号的发生时间点，用于判断 SW VSYNC 信号是否与 HW VSYNC 信号同步。
* 如果两者不同步，打开 HW VSYNC 信号上报，采样并计算新的 SW VSYNC 信号周期时间，直到两者同步误差小于阈值，再关闭 HW VSYNC 信号。

#  参考文件
1. [Implementing VSYNC](https://source.android.com/devices/graphics/implement-vsync)
1. [Android SurfaceFlinger SW Vsync模型](https://www.jianshu.com/p/d3e4b1805c92)
1. [Android DispSync 详解](https://simowce.github.io/all-about-dispsync/)
1. [Android SurfaceFlinger 学习之路(五)----VSync 工作原理](http://windrunnerlihuan.com/2017/05/25/Android-SurfaceFlinger-%E5%AD%A6%E4%B9%A0%E4%B9%8B%E8%B7%AF-%E4%BA%94-VSync-%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86/)

# 详细的类图列表

下图是 DispSync 类图：

![DispSync Class Diagram](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/general-design/services_surfaceflinger_Scheduler_DispSync%20Class%20Diagram.svg)

下图是 DispSyncSource 类图：

![DispSyncSource Class Diagram](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/general-design/services_surfaceflinger_Scheduler_DispSyncSource%20Class%20Diagram.svg)

下图是 EventControlThread 类图：

![Event Control Thread Class Diagram](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/general-design/services_surfaceflinger_Scheduler_EventControlThread%20Class%20Diagram.svg)

下图是 EventThread 类图：

![Event Thread Class Diagram](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/general-design/services_surfaceflinger_Scheduler_EventThread%20Class%20Diagram.svg)

下图是 MessageQueue 类图：

![MessageQueue Class Diagram](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/general-design/services_surfaceflinger_Scheduler_MessageQueue%20Class%20Diagram.svg)

下图是 Scheduler 类图：

![Scheduler Class Diagram](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/general-design/services_surfaceflinger_Scheduler_Scheduler%20Class%20Diagram.svg)

