# 架构设计与组合模式

在 SurfaceFlinger 程序里应用了 Active Object 设计模式，主要是考虑到以下几点：
* 在 SurfaceFlinger 程序里最主要的功能就是合成各层的内容。同时这也是最耗时的功能。在操作过程中不可打断，否则显示内容会出问题。
* 同时客户端软件提交的事务请求也需要尽快响应。这些请求数量大，同时又大多是耗时不长的功能。
* 在现代的 SoC 大多是多核异构解决方案，为并发设计提供了可能。同时，GPU 和 HWComposer 硬件的操作特点，最好是批量和同步完成。

Active Object 模式属于并发设计模式(Design Patterns for Concurrency)，对应到并发架构模式(Architectural Patterns for Concurrency)，就是半同步/半异步(Half-sync / Half-async)模式。Active Object 模式和 Half-sync / Half-async 模式，它们两个是对同一件事情不同视角的描述。半同步/半异步(Half-sync / Half-async)模式是一个分层架构。它包含三个层次：异步任务层、同步任务层和队列层。半同步/半异步(Half-sync / Half-async)模式的核心思想是如何将系统中的任务进行恰当的分解，使各个子任务落入合适的层次中。在 SurfaceFlinger 里，Surface 发送来的请求，数量大，又是轻量级，所以需要异步处理，以提高响应度。对合成操作，要求一致性高，又是重量级，所以在同步任务里完成。

再回到设计模式。Android 图形系统最根本的就是 Producer-Consumer 模式。而 Active Object 模式就是 Producer-Consumer 模式的一种变种。同样的， Thread Pool 模式也是 Producer-Consumer 模式的一种变种。在 SurfaceFlinger 程序里同样应用了 Thread Pool 设计模式。

| No. | Thread               | Count |
|:---:|----------------------|------:|
| 1   | Binder thread        | 4     |
| 2   | HW VSYNC Thread      | 1     |
| 3   | DispSyncThread       | 1     |
| 4   | EventControlThread   | 1     |
| 5   | EventThread          | 2     |
| 6   | Main thread          | 1     |
| 7   | IdleTimer            | 1     |
| 8   | RegionSamplingThread | 1     |
|     |                      |       |
|     | Total                | 12    |

由此可见，SurfaceFlinger 里面的角色就是采用 Thread Pool 模式来实现。跨进程交互的 Binder 接口被限定为 4 个线程。此外的其它角色都对应一个线程，每个线程从自己的任务队列里顺序处理任务请求。这样既提高了业务吞吐量又提高了任务的响应能力。

所有这一切，都是为了利用多核异构方案，增加并行的可能。例如：
* 不直接涉及 SurfaceFlinger 类中方法的事务请求，例如生产端循环的 dequeueBuffer / queueBuffer 操作，就是在 Binder thread 中随机到达并完成返回。而消费端循环的 acquireBuffer / releaseBuffer 操作，则是在 Main thread 中定时完成。
* 同样的，直接涉及 SurfaceFlinger 类中方法的事务请求，则是在 Binder thread 中随机到达，在 Main thread 中定时完成。
* 能延后处理的任务就要延后处理，让事务请求尽快返回，尽可能不在 Main thread 中执行与合成无关的操作。例如对 MessageCreateLayer 消息的处理，返回时，并没有为对应的 BufferQueue 申请 GraphicBuffer。只有在需要时，BufferQueue 才会为 Surface-Layer 申请 GraphicBuffer。这时是在 Binder thread 中完成，不会影响 Main thread 中的操作。
* 另外，根据显示是由 VSYNC 信号驱动的特点，协调生产端和消费端的速率，使得两端趋向于同步错位运行。这就是编舞者(Choreographer)框架的目标。

从这里也可以看到[[相位偏移量的设置](../general-design/VSYNC.md#相位偏移的设置)]一节中所讨论的 SurfaceFlinger 程序的合成操作与 Application 程序的渲染操作错位运行的合理性：
* N# HW_VSYNC 信号到达，HWComposer 模块底层将 Loop N-1 中合成好的 framebuffer N-1 送到显示设备上。
* HW_VSYNC 信号过了 sfVsyncPhaseOffsetNs 纳秒，SurfaceFlinger 程序收到 VSYNC-sf 消息，开始 Loop N，进行合成操作。
  + 从 Loop N-1 看，大多数 Application 程序已经完成了上一帧的渲染操作，在 Binder thread 中开始提交新帧。所以没有多少用户在竞争 GPU。
  + 从当前看，HWComposer 模块底层应该已经向底层硬件提交了显示申请，不会多用 CPU。
  + 合成分成两个阶段。第一阶段是部分 Layer 采用 GPU 进行合成。这时候 SurfaceFlinger 程序的 Main thread 是最主要的 CPU 和 GPU 的用户。
* HW_VSYNC 信号过了 vsyncPhaseOffsetNs 纳秒，各个 Application 程序收到 VSYNC-app 消息，开始进行渲染操作。
  + 因为 vsyncPhaseOffsetNs 不等于 sfVsyncPhaseOffsetNs，错位启动，所以这时候 SurfaceFlinger 程序采用 GPU 进行合成的任务应该完成了部分，Application 程序可以竞争到一些 GPU 资源。
  + 接着，SurfaceFlinger 程序的合成任务进入第二阶段，采用 HWComposer 模块合成剩下的 Layer。
  + 这时候 GPU 资源主要由 Application 程序使用。而 HWComposer 模块应该完成了显示工作，可以交给 Main thread 用来进行合成工作。
* SurfaceFlinger 程序必须在 N+1# HW_VSYNC 信号到达前完成 framebuffer N 的合成工作。如果工作频率为 60Hz，则它的合成时间为
  + `16666667 - sfVsyncPhaseOffsetNs` 纳秒
  + 需要让这个值尽可能的大，以便 SurfaceFlinger 程序有更多的时间进行合成。
* 对于 Application 程序程序来说，当前渲染的 frame 要想在 Loop N+1 时能提交合成，如果工作频率为 60Hz，则它的渲染时间为
  + `16666667 - (vsyncPhaseOffsetNs - sfVsyncPhaseOffsetNs)` 纳秒
  + 需要让这个值尽可能的大，以便 Application 程序有更多的时间进行渲染。
* 但是这两个相位偏移量又不能为"0"，否则显示、合成与渲染任务在同一时间爆发，会让系统负荷更糟糕。所以这两个相位偏移量的选择真是煞费苦心。需要选择很多场景做很多实验，然后要通盘考虑才行。
* 所以，在一个 VSYNC 事件中，显示操作、合成操作、渲染操作错位运行，显示设备开始显示帧 N，SurfaceFlinger 开始为帧 N+1 合成窗口，应用处理等待的输入并生成帧 N+2。这是一个负载均衡的、合理的、可长期运转的方案。

另外，在 surfaceflinger 程序的实现细节上，还应用了几种设计模式：
* 跨进程接口(Interface)的实现，应用了 Proxy 模式和 Observer 模式。
* 对 VSYNC 信号的处理，应用了命令模式(Command Pattern)。
* 在 Active Object 模式中，为获取异步任务的结果，应用了 Promise 模式。
* 对应 Active Object 模式，一般同时会应用 Monitor Object 模式。在 surfaceflinger 程序里，这个模式并不明显，但也通过 Mutex 类来保护危险区域的代码。保证在多核条件下，在任一时间内，只有唯一的公共的成员方法，被唯一的线程所执行。
* 对于 Thread Pool 模式的结束，应用了 Two-phase Termination 模式。

总之，像 surfaceflinger 这样有着复杂需求的大程序里，要根据应用特点，需要同时应用多种设计模式，才能可靠高效地实现需求。

