# Android 的图形系统服务端的设计
* * *

如果有谁研读 SurfaceFlinger 程序的代码，不被里面众多庞杂臃肿的类，还有类与类之间迷宫一样相互调用的关系所迷惑，那他就没有认真读代码。

要想分析 SurfaceFlinger 程序的设计，从进程/线程间的消息流动，以及基于模式(pattern)的设计这两方面入手，也许能找到一条出路。

# [SurfaceFlinger 模块划分](module.md)

# [SurfaceFlinger 中的消息与传递路径](message.md)

# [合成流程中的路径](compositing-path.md)

# 基于 Active Object 模式进行设计

# 架构设计与组合模式

# 小结

层次与模式

这是软件VSYNC事件相对于HWComposer报告的VSYNC事件的相位偏移（以纳秒为单位）。软件VSYNC事件是基于SurfaceFlinger和Choreographer的应用程序运行每个帧时。

此相位偏移允许调整从应用程序唤醒时间（由Choreographer）到显示结果窗口图像的时间的最小延迟。该值可以是正的（在HW VSYNC之后）或负（在HW VSYNC之前）。将其设置为0将导致两个VSYNC周期的较低延迟界限，因为应用程序和SurfaceFlinger将在HW VSYNC之后立刻运行。将其设置为正数将导致最小延迟为：
```
    (2 * VSYNC_PERIOD - (vsyncPhaseOffsetNs % VSYNC_PERIOD))
```
请注意，减少此延迟使应用程序更有可能没有及时准备好窗口内容图像。当这种情况发生时，延迟最终将是一个额外的VSYNC周期，动画将出现问题。因此，应该稍微保守地调整这个延迟（或者至少在意识到要进行权衡的情况下）。

# 参考文件
1. [SurfaceFlinger and Hardware Composer](https://source.android.com/devices/graphics/arch-sf-hwc)
1. [SurfaceTexture](https://source.android.com/devices/graphics/arch-st)
1. [Android graphic system (SurfaceFlinger) : Design Pattern's perspective](https://www.slideshare.net/BinChen3/android-graphic-system-surface-flinger-patternsperspective-external-version)
1. [Java多线程编程实战指南（设计模式篇）](http://www.broadview.com.cn/book/506)

