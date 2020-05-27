# VSYNC 信号的接口与约定

对于 VSYNC 信号，在文档[[VSync信号](http://windrunnerlihuan.com/2017/05/21/VSync%E4%BF%A1%E5%8F%B7/)]中有详细的介绍。其实，现在的硬件抽象已经封装得很高级了，在 Linux OS 层面看到的 VSYNC 信号，已经不是当初 VSYNC 信号的含义了，但术语被保留了下来。一个明显的例子就是 VSYNC 信号到达的时候，软件标注为"case INVALIDATE:"，其实可以理解为显示器向大家广播：屏幕上的内容已经无效了，你们赶快送新内容过来吧:hamburger:。因此，做为 Android 软件的开发人员，只需要记住，见到了 VSYNC 信号，这就表明抽象的显示屏可以用了，于是各路人马齐欢腾:fist:。

实现 VSYNC 的目标很简单：VSYNC 可将某些事件同步到显示设备的刷新周期。应用总是在 VSYNC 边界上开始绘制，而 SurfaceFlinger 总是在 VSYNC 边界上进行合成。这样可以消除卡顿，并提升图形的视觉表现。 

VSYNC 的接口很简单，它归属于显示设备，因此在 Hardware Composer (HWC) 接口中提供。在 HWC 接口(interface)中，VSYNC 接口就是一个函数指针，用于指示要为 VSYNC 实现的函数：
```
int (waitForVsync*) (int64_t *timestamp) 
```

LCD 显示器一般的刷新率为 60HZ，也就是大约 16.67ms 产生硬件 VSYNC 信号。而这个 VSYNC 接口关注点在于可供软件预测的时间戳。在发生 VSYNC 并返回实际 VSYNC 的时间戳之前，这个函数会处于阻塞状态。每次发生 VSYNC 时，都必须发送一条消息。客户端会以指定的间隔收到 VSYNC 时间戳，或者以 "1" 为间隔连续收到 VSYNC 时间戳。你必须实现最大延迟时间为 1 毫秒（建议 0.5 毫秒或更短）的 VSYNC，因此返回的时间戳必须非常准确。

因为打开硬件 VSYNC 信号时，对手机的功耗会有影响。所以一个开发经验就是，只在必要的时候打开硬件 VSYNC 信号，采样并建立模型，预测硬件 VSYNC 信号发生的时间点，然后关闭硬件 VSYNC 信号，只使用软件 VSYNC 信号。这样对于降低手机功耗有好处。


