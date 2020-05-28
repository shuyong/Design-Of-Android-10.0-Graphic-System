# Composer 的设计
* * *

在 surfaceflinger service 中的重要实现类 Layer / SurfaceFlinger，声明位于这里：
* frameworks/native/services/surfaceflinger/Layer.h
* frameworks/native/services/surfaceflinger/SurfaceFlinger.h

SurfaceFlinger 类，是图形内容的 Consumer 端。SurfaceFlinger 类暴露的接口，是一种代理模式：
* ISurfaceComposer。对 SurfaceFlinger 类的代理。接口中的方法很多，这里主要关注下面几个：
  + createConnection()：为客户端提供访问 Composer 的接口：ISurfaceComposerClient。
  + createDisplayEventConnection()：为客户端提供访问 VSYNC 的接口：IDisplayEventConnection。
* ISurfaceComposerClient：客户端访问 Composer 的接口。接口中的方法很多，这里主要关注下面几个：
  + createSurface()
* IDisplayEventConnection: 客户端访问/管理 VSYNC 的接口。

这些接口的声明位于这里：
* frameworks/native/include/gui/IDisplayEventConnection.h
* frameworks/native/include/gui/ISurfaceComposer.h
* frameworks/native/include/gui/ISurfaceComposerClient.h

此外，一路对应上来的硬件合成器(HWC)的接口，在 surfaceflinger 里底层的 DisplayHardware 模块里：
* Composer：对应 HAL 中的 hwcomposer 接口，以及 HIDL 中的 IComposer 接口。
* PowerAdvisor：对应 HAL 中的 power 接口，以及 HIDL 中的 IPower 接口。

下图是 Composer 接口的关系组合图：
[Composer 接口的关系组合图](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/server-design/services_surfaceflinger_DisplayHardware_ComposerHal%20Component%20Diagram.svg)

下图是 PowerAdvisor 接口的关系组合图：
[PowerAdvisor 接口的关系组合图](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/server-design/services_surfaceflinger_DisplayHardware_PowerAdvisor%20Class%20Diagram.svg)


