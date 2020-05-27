# 代码复用
* * *

Android OS 是一个极其成功的项目。Google 公司不但凭着高深的技术，也凭着高超的商业手段，将 Android 系统推广开来。在智能手机的世界里，Android 系统已经是占有率第一。在 Linux 世界里，Android 系统的占有率也是远远超过其它传统的 Linux 系统。在推广传统 Linux 系统时碰到的第一个拦路虎：缺少硬件平台和软件驱动的问题，在 Google 公司手里迎刃而解。以前要推广 Linux 系统，是求着厂商出板子、写驱动。而现在是厂商主动给 Android 系统出板子、写驱动。有了生态系统，什么都好说话。个中滋味，只有经历过的人才能体会。

于是有人就想着借着 Android 系统里的板子和驱动，去推广传统 Linux 系统。因为第一个拦路虎没有了，板子和驱动都是现成的，推广传统 Linux 系统的难度大大降低。不过，在这种尝试过程中，也出现了问题。

# 现有的问题

Android 系统，不知是有意无意，全部采用了有别于传统 Linux 系统的不同的技术方案。可以说，除了 Linux Kernel 相似之外，其它的地方都不同。这就形成了很高的技术壁垒，让传统 Linux 系统很难复用 Android 系统里的代码模块。

首先碰到的问题有2个：
1. C run-time library。在传统 Linux 系统里，C run-time library 是有名的 glibc。而在 Android 系统里，则是相对不出名的一个轻量级的 C 库：bionic。这就造成了一个问题，在传统 Linux 系统里的程序，既无法链接 Android 系统里的代码库，也无法直接调用代码库里的函数。让人有看着宝山却又无门而入的感觉。
2. Init scripts。在传统 Linux 系统里，虽然 init 系统有过升级，但脚本概念、程序空间的划分、进程间的通讯等等这些技术是相同的。也就是，可以更换多种 init 程序和脚本，但最后启动的图形工作环境是一样的。在最终的图形工作环境里，用户感觉不到差异。而 Android 系统的 init 程序、脚本概念、程序空间的划分、进程间的通讯等等这些技术和传统 Linux 系统都不一样。两者就没有交换的可能。

但是，难度再大也挡不住高手的挑战。这两个难题分别被人突破。于是，进一步的问题：分析 Android HAL 的接口与约定，复用 Android HAL 代码，则不但是有趣的事情，也是必要的事情。于是现在出现了两种解决方案：
1. [Mer Project](http://www.merproject.org/)
2. FireFox OS

# 现有的解决方案

## Mer Project

[Mer Project](http://www.merproject.org/)的解决方案是目前最成功，应用得最广的方案。除了 Mer Project 本身以外，还有这些项目在使用他们的方案：
1. [Sailfish OS](https://sailfishos.org/)
2. [Tizen OS](http://www.tizen.org/)
3. [Asteroid OS](https://asteroidos.org)
4. [webOS](http://webosose.org/)
5. 国内外各种号称能复用 Android 驱动的方案，大多也是采用了 Mer Project 的技术。

在这些项目里，Sailfish OS 是已经出产品，并且成功量产的项目。因此后面的分析，基本上就是针对他们的技术特点进行分析。

对于上述的复用的第一个问题，Mer Project 采用的是 [libhybris](https://github.com/libhybris/libhybris) 技术进行解决。

简单地说，glibc 程序之所以不能链接和调用 bionic 代码库，是因为编译系统里的 "linker & loader" 不一致造成的。这是不可调和的矛盾。但是，在 C 代码库里，有一个 "dlopen()/dlsym()" 动态装载和调用代码库的方式。能不能在 glibc 程序里动态装载 bionic 代码库呢？很遗憾，这个问题还是会回到 "linker & loader" 不一致的问题，因此造成装载失败。

但是，任何难题总会露有一线生机。用 glibc 版本的 "dlopen()" 不能装载 bionic 代码库，用 bionic 版本的呢？幸运的是，因为 bionic 相当简单，所以 bionic 版本的 "dlopen()" 也相当简单，代码在 "bionic/linker/" 目录里。于是经过一番艰苦的努力，在 glibc 程序里就可以用 bionic 版本的 "dlopen()/dlsym()" 动态装载和调用 bionic 代码库了。

这里不讨论 libhybris 技术，总之，第一个问题解决了。如果要想理解其中的奥秘，需要对 "linker & loader" 有一定的了解，还需要一点想象力，理解代码空间、符号解析、函数指针、C run-time library 的实现与参数的差异等等问题。理解 libhybris 技术还是一件蛮有趣的事情，对什么是运行程序这个问题会有更深的理解。

第二个问题也比较好解决：就是还是继续保留传统 Linux 系统里的 init 系统和脚本，使用这些脚本启动一些复用 Android HAL 所必须的 service。只要运行环境和参数给对了，这些 service 应该能正常运行。

接着，更深的问题出现了，Android HAL 的接口与约定是什么？怎么样从 Android Framework 的代码里推测出相应的知识，然后复用 Android HAL？经过各位高手的努力，这个工程也有很大的进展。

在 Mer Project 的图形系统里：
1. 直接采用 gralloc module，重新实现了 ANativeWindow / ANativeWindowBufer 接口，使得 glibc 程序可以复用 EGL / GLES 库。
2. 用 libhybris 技术重新封装了 EGL / GLES 库，使之实现了 wayland protocol。
3. 整个图形系统采用 qt5 / wayland。
4. 直接采用 HWC module，实现了 [qt5 / hwc qpa plugin](https://github.com/mer-hybris/qt5-qpa-hwcomposer-plugin)。
5. 可以说：gralloc module / HWC module / VSYNC / Fence 这些接口都复用到了，然后实现了一个和 Android 不一样的图形系统。

但是，在实践中，libhybris 技术也出现了一些问题：
1. 毕竟是采用 hack 技术，而不是正规的调用方式，bionic 代码运行时有可能会有诡异的崩溃。BUG 很难定位，定位了也很难理解，理解了也很难解决。
2. Android 每次大的升级，编译系统和代码库也会升级。libhybris 技术也许也需要随之升级。但是，bionic linker 每次的变动都是比较大的，移植到 libhybris 库里，工作量比较大。
3. 现在 libhybris 技术，也只是在高通的几个芯片方案里得到比较充分的验证。在其它平台和方案里，可靠性还有待验证。

但是，不管怎么样，在技术上，Mer Project 还是一个很成功的项目。差不多是复用 Android HAL 的唯一选择。

## FireFox OS

因为在 glibc 程序里复用 Android HAL 首先会碰到那两个问题。解决起来，不但费时费力，还会有可靠性问题。于是 FireFox OS 采用了另外的方案：就是将自己移植到 Android 的运行环境里，全部的代码都用 Android 的编译器编译一遍。这样，那两个问题就消失了，FireFox OS 也就可以直接调用 Android HAL 的接口了。

FireFox OS 之所以能这么做，是因为：
1. FireFox Browser 原本就可以运行在很多不同的、差异很大的 OS 里。于是 FireFox 本身的代码可移植性就很好，交叉编译脚本也很完善。
2. 为了做到上面一点，FireFox 差不多就是自包含的系统，例如有自己的图形系统和输入系统，只需要实现一个相对薄层的移植层，就可以运行在新的环境里。也就是，FireFox 离一个独立的 OS 已经很接近了。

其实，将大型软件系统移植到 也是Android 的运行环境里，是很普遍的事情。例如大型的视频系统，Kodi / VLC / Gstreamer，都有 Android 版本。只是这些程序是想让自己成为一个合格的 Android 程序，所以是尽量调用 Android 高层的接口。而 FireFox OS 是要抛开 Android 高层，只留下 Android Framework / C++ 层以及下面的 Android HAL，然后自己做 OS。

FireFox OS 复用 Android 代码库，也有自己的特色，就是尽可能复用 Android Framework / C++ 层的接口。例如，FireFox OS 复用的是 GraphicBuffer 类来使用 ANativeWindowBuffer 接口(interface)，而不是从 gralloc module 上面重新实现。这样可以减少工作量，也有利于提高可靠性。但是，他们为了不在 FireFox OS 的 C++ 代码里引入 Android C++ namespace，也采用了 C 语言级别的 hack 方式去调用 GraphicBuffer 类的代码。该方式相当的野蛮粗暴，也影响了可移植性。

接着，他们再实现自己的 ANativeWindow 接口(interface)以及背后的 BufferQueue，这是因为他们有他们自己的图形系统和跨进程协作方式。也正是因为这点，他们也遇到了麻烦。因为他们在进程中使用了多个 Texture ID 及其绑定的 EGLImage，这样可以并发渲染图形。但是因为 GLES 没有直接从 Texture ID 中解除绑定 EGLImage 的接口，从而也影响了并发的效率。

不管怎么样，FireFox OS 也是已经出产品，并且成功量产的项目。可惜因为商业的原因，该项目已经停止开发。现在回过头来看，类似复用 GraphicBuffer 类的方式，会有很大的可移植的问题。因为 Android 每次大的升级，Android Framework 都会有较大的改动，原来的方法就消失不见了，取而代之的是重新设计的新的方法。这是因为 Android 保证的是上下两层接口的兼容性。对上，保证的是 JAVA 层的兼容性，因为它们面对的是现有的应用程序。对下，保证的是 Android HAL 的兼容性，因为它们面对的是厂商现有的实现。而中间的 Android Framework，功能和性能是优先的，兼容性不是优先考虑的问题。

但是不管怎么样，这种技术路线可以带来工作量小，可靠性高的好处。不过有一点值得注意，采用这种技术路线可能会带来法律问题。这个问题不在这里讨论。

# 其它的可能性

## 复用的层次

现在复用 Android 代码，基本上都是从 HAL 层的代码开始。然后根据 Android 的实现，推测上层的接口所需要的功能，以及相应的有限状态机，然后做新的实现。

其实从编译器和编译依赖系统的角度看，整个 Android 公开的代码，去除最底层的 C run-time library : bionic，是完全可以用 gcc / glibc 重新编译的。但是是什么东西阻止了 Android 代码的复用？其实就是 HAL 层中的那些厂商的 close-module。就是因为闭源代码的存在，整个 HAL 层必须被原封不动地复用。

如果再想在更高层复用 Android 代码，则会碰到修改编译依赖系统 ninjia 的配置文件的问题。这也是一个庞大的工程。另外，Android 代码里复杂的 C++ namespace，也是令人头痛的问题。

## Binder Interface 的复用

如果是 glibc 空间的程序，想从高层，如 HIDL 或 Framework 层复用 Android 代码，有一种方法就是重新编译 Binder Interface 中的 Bp 类这一半。这样从远程调用 HAL 模块，这也是一种可能。这样一直到 Framework 层的代码再用 gcc / glibc 重新编译，glibc 空间的程序就可以复用 Android 系统而没有任何冲突。不过修改编译依赖系统 ninjia 的配置文件的工作量实在太大了。所有现在可行的方案还是复用 [Mer Project](http://www.merproject.org/) 方案。

