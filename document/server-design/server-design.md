# Android 的图形系统服务端的设计
* * *

如果有谁研读 SurfaceFlinger 程序的代码，不被里面众多庞杂臃肿的类，还有类与类之间迷宫一样相互调用的关系所迷惑，那他就没有认真读代码。

要想分析 SurfaceFlinger 程序的设计，从进程/线程间的消息流动，以及基于模式(pattern)的设计这两方面入手，也许能找到一条出路。

# [SurfaceFlinger 模块划分](module.md)

# [SurfaceFlinger 中的消息与传递路径](message.md)

# [基于 Active Object 模式进行设计](active-object.md)

# [架构设计与组合模式](component-pattern.md)

# 小结

分析 Android 图形系统，最大的收获就是在软件层次与设计模式方面：
* 为了最大保证兼容性和灵活性，纵向采用访问者模式(Visitor Pattern)实现 HIDL。间接也保证了 Android 系统的健壮性。
* Android 图形系统本质上就是 Producer-Consumer 模式。为了保证性能，横向采用活动对象模式(Active Object Pattern)。这是 Producer-Consumer 模式的一个变种。可以利用多核异构的特点提高并发性。
* 最终 Android 图形系统就在性能、兼容性、健壮性等方面寻找平衡点。于是程序就复杂无比，层次多，模式复杂，失去了易读性。

# 参考文件
1. [SurfaceFlinger and Hardware Composer](https://source.android.com/devices/graphics/arch-sf-hwc)
1. [SurfaceTexture](https://source.android.com/devices/graphics/arch-st)
1. [Android graphic system (SurfaceFlinger) : Design Pattern's perspective](https://www.slideshare.net/BinChen3/android-graphic-system-surface-flinger-patternsperspective-external-version)
1. [Java多线程编程实战指南（设计模式篇）](http://www.broadview.com.cn/book/506)

