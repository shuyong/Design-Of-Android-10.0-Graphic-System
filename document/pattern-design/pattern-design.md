# 按照设计模式进行设计
* * *

设计模式是在软件工程实践过程中，程序员们总结出的良好的编程方法。也就是，设计模式针对特定问题提供了最佳实践。使用设计模式能够增加系统的健壮性，易修改性和可扩展性。当所进行开发的软件规模比较大的时候，良好的设计模式会给编程带来便利，让系统更加稳定，也更容易理解。 

在 Android 图形系统中，也采用了很多设计模式。在文档[1]中有很精彩的论述。下面，我们先简单复习 Android 图形系统中常用的几个设计模式的知识：
* Observer
* Proxy
* Visitor Pattern
* Producer-Consumer
* Thread Pool
* Command Pattern
* Active Object
* Half-Sync/Half-Async

# 常用设计模式简介 

## 观察者(Observer)模式 

观察者模式（有时又被称为模型(Model)-视图(View)模式、源(Source)-收听者(Listener)模式、从属者模式、或者Publish-Subscribe模式）是软件设计模式的一种。在此种模式中，一个目标物件管理所有相依于它的观察者物件，并且在它本身的状态改变时主动发出通知。 

观察者模式 Observer 属于对象行为型的设计模式。  

### 1.概述

一些面向对象的编程方式，提供了一种构建对象间复杂网络互连的能力。当对象们连接在一起时，它们就可以相互提供服务和信息。 

通常来说，当某个对象的状态发生改变时，你仍然需要对象之间能互相通信。但是出于各种原因，你也许并不愿意因为代码环境的改变而对代码做大的修改。也许，你只想根据你的具体应用环境而改进通信代码。或者，你只想简单的重新构造通信代码来避免类和类之间的相互依赖与相互从属。 

### 2.问题

当一个对象的状态发生改变时，你如何通知其他对象？是否需要一个动态方案――一个就像允许脚本的执行一样，允许自由连接的方案？ 

### 3.解决方案

观测模式：定义对象间的一种一对多的依赖关系,当一个对象的状态发生改变时, 所有依赖于它的对象都得到通知并被自动更新。 

![Observer Implementation](https://www.oodesign.com/images/design_patterns/behavioral/observer_implementation_-_uml_class_diagram.gif)

观测模式允许一个对象关注其他对象的状态，并且，观测模式还为被观测者提供了一种观测结构，或者说是一个主体和一个客体。主体，也就是被观测者，可以用来联系所有的观测它的观测者。客体，也就是观测者，用来接受主体状态的改变 观测就是一个可被观测的类（也就是主题）与一个或多个观测它的类（也就是客体）的协作。不论什么时候，当被观测对象的状态变化时，所有注册过的观测者都会得到通知。 

观测模式将被观测者（主体）从观测者（客体）中分离出来。这样，每个观测者都可以根据主体的变化分别采取各自的操作。（观测模式和 Publish-Subscribe 模式一样，也是一种有效描述对象间相互作用的模式。）观测模式灵活而且功能强大。对于被观测者来说，那些查询哪些类需要自己的状态信息和每次使用那些状态信息的额外资源开销已经不存在了。另外，一个观测者可以在任何合适的时候进行注册和取消注册。你也可以定义多个具体的观测类，以便在实际应用中执行不同的操作。 

将一个系统分割成一系列相互协作的类有一个常见的副作用：需要维护相关对象间的一致性。我们不希望为了维持一致性而使各类紧密耦合，因为这样降低了它们的可重用性。 

### 4.适用性 

在以下任一情况下可以使用观察者模式:
* 当一个抽象模型有两个方面, 其中一个方面依赖于另一方面。将这二者封装在独立的对象中以使它们可以各自独立地改变和复用。 
* 当对一个对象的改变需要同时改变其它对象 , 而不知道具体有多少对象有待改变。 
* 当一个对象必须通知其它对象，而它又不能假定其它对象是谁。换言之 , 你不希望这些对象是紧密耦合的。 

## 代理(Proxy)模式

1. 定义
给目标对象提供一个代理对象，并由代理对象控制对目标对象的引用。

2. 主要作用
通过引入代理对象的方式来间接访问目标对象。

3. 解决的问题
防止直接访问目标对象给系统带来的不必要复杂性。

![Proxy Implementation](https://www.oodesign.com/images/design_patterns/structural/proxy-design-pattern-implementation-uml-class-diagram.png)

为什么要采用这种间接的形式来调用对象呢？一般是因为客户端不想直接访问实际的对象，或者访问实际的对象存在困难，因此通过一个代理对象来完成间接的访问。

## 生产者-消费者(Producer-Consumer)模式

Producer-Consumer 模式通过在数据的生产者和消费者之间引入一个通道(Channel，可以将其简单地理解为一个队列 Queue)对二者进行解耦(Decouping)：
* 生产者将其生产的数据放入通道，消费者从相应通道中取出数据进行消费。
* 生产者和消费者各自运行在各自的线程中，从而使双方处理速率互不影响。

为什么要有 Producer-Consumer 模式呢？
* 消除了生产者与消费者之间的代码依赖。
* 实现线程间的协调运行。生产者与消费者之间的运行速率不同，直接调用，数据处理会产生延迟，导致程序响应性下降。

它由以下角色组成：
* Producer：负责生成 Product，传入 Channel。
* Product：由 Producer 生成，供 Consumer 使用。
* Channel：Producer 和 Consumer 之间的缓冲区，用于传递 Product。
* Consumer：从 Channel 获取 Product 使用。

通道Channel积压问题：

当消费者的处理能力低于生产者的处理能力时，随着时间的推移，通道中存储的“产品”会越来越多而出现挤压现象。常见的处理方法有两种：  
1. 使用有界阻塞队列，使用有界阻塞队列作为 Channel 参与者的实现可以实现将消费者的处理压力“反弹”给生产者。具体来说，当通道的有界阻塞队列逐渐积压到队列满，此时生产者线程会被阻塞直到相应的消费者“消费”了队列的一些”产品“，使得队列非满。
2. 使用带流量控制的无界阻塞队列。使用无界阻塞队列，借助控制流量实现，即对同一时间内可以有多少个生产者线程往通道中存储”产品“进行限制。一般采用 Semaphore 信号量来控制。

线程的启动与停止问题：

* 消费者线程和生产者线程哪个先启动？一般是先启动消费者。这样，当生产者生产出 Product，放入 Channel 后，就立刻被消费，不会出现丢失情况。
* 消费者线程和生产者线程哪个先停止？一般是先停止生产者，等 Channel 剩余 Product 备份后，或者被消费者处理完后，再停止消费者。

Thread-Pool 模式，Active Object 模式等都是 Producer-Consumer 模式的变种，同时 Producer-Consumer 模式中的生产者和消费者阻塞唤醒机制也可以通过 Guarded Suspension 模式实现。

## 线程池(Thread Pool)模式

线程池在多线程编程中经常要用到，其基本模型仍是生产者/消费者模型，线程池一般由线程池管理器(ThreadPool)，工作线程(PoolWorker)，任务(Task)，任务队列(TaskQueue)四部分组成，其中：
* 线程池管理器(ThreadPool)：用于创建并管理线程池，包括 创建线程池，销毁线程池，添加新任务；
* 工作线程(PoolWorker)：线程池中线程，在没有任务时处于等待状态，可以循环的执行任务；
* 任务接口(Task)：每个任务必须实现的接口，以供工作线程调度任务的执行，它主要规定了任务的入口，任务执行完后的收尾工作，任务的执行状态等；
* 任务队列(TaskQueue)：用于存放没有处理的任务。提供一种缓冲机制。

![A sample thread pool (green boxes) with waiting tasks (blue) and completed tasks (yellow)](https://upload.wikimedia.org/wikipedia/commons/0/0c/Thread_pool.svg)

为什么要使用线程池模式？

因为线程在执行任务时需要消耗CPU时间和内存等资源，线程对象(Thread 实例)本身以及线程所需的调用栈(Call Stack)也占用内存，并且在程序中创建一个线程往往意味着占用宝贵操作系统的本地线程(Native Thread)资源，所以复用一定的线程就成为了一种解决该问题的做法，故线程池模式应运而生。 

Thread Pool 模式的本质是使用有限的资源去处理相对无限的任务。核心思想是使用队列对待处理的任务进行缓存，并复用一定数量的工作者线程去取队列中的任务进行执行。

## 命令模式(Command Pattern)

### 问题

某个类中需要定义一个方法，该方法要实现的功能是不确定的，需要等到程序执行该方法的时候才确定下来。

### 解决方案

按照面向对象程序设计的原则，如果在设计类的时候，成员方法在不同的类中具体实现也是不同的，这种变化的方法可以抽取出来，形成接口。接口可以作为具体业务类的成员变量或者方法参数进行使用。这种设计模式即为：命令模式(Command Pattern)，即向一个方法传入行为。 

### 说明

命令模式(Command Pattern)：
* 将一个请求封装为一个对象，从而让我们可用不同的请求对客户进行参数化；
* 对请求排队或者记录请求日志，以及支持可撤销的操作。
* 命令模式是一种对象行为型模式，其别名为动作(Action)模式或事务(Transaction)模式。

在命令模式结构图中包含如下4个角色：
* Command(抽象命令类)：抽象命令类一般是一个抽象类或接口，在其中声明了用于执行请求的 execute() 等方法，通过这些方法可以调用请求接收者的相关操作。
* ConcreteCommand(具体命令类)：具体命令类是抽象命令类的子类，实现了在抽象命令类中声明的方法，它对应具体的接收者对象，将接收者对象的动作绑定其中。在实现 execute() 方法时，将调用接收者对象的相关操作(Action)。 
* Invoker(调用者)：调用者即请求发送者，它通过命令对象来执行请求。一个调用者并不需要在设计时确定其接收者，因此它只与抽象命令类之间存在关联关系。在程序运行时可以将一个具体命令对象注入其中，再调用具体命令对象的 execute() 方法，从而实现间接调用请求接收者的相关操作。 
* Receiver(接收者)：接收者执行与请求相关的操作，它具体实现对请求的业务处理。 

![Command Implementation](https://www.oodesign.com/images/design_patterns/behavioral/command_implementation_-_uml_class_diagram.gif)

## 活动对象(Active Object)设计模式

活动对象(Active Object)设计模式是面向并发对象(Concurrent Object)的设计模式。 

什么是并发对象(Concurrent Object)？不同于一般的对象，并发对象指的是该对象方法的调用与方法的执行不在同一个线程內，也即：该对象方法被异步执行。这其实是在多线程编程环境下典型的计算特征，线程引入了异步。从另一个角度来看，并发对象其实是面向对象的概念应用于多线程计算环境下的产物。 

活动对象(Active Object)设计模式是将对象的方法执行(method execution)从方法调用(method invocation)中解耦，使两者位于各自的控制线程中。该设计模式的目标是通过使用异步方法调用和处理请求的调度程序引入并发性。 

### 为什么会设计如此复杂的设计模式？

我们知道，基于多线程机制的并发编程模式可以极大提高应用的 QoS(Quality of Service)。典型的例子如，在开发服务器端的应用时，通行的做法就是通过多线程机制并发地服务于客户端提交上来的请求，以期提高服务器对客户端的反应度(Responsiveness)。同时，相比于单线程的应用，并发的多线程应用相对要复杂得多。在多线程的计算环境里，并发对象被所有的调用者线程所共享。一般来说，并发对象的设计实现需要考虑下面的几个重要因素：
* 并发对象的任何一次的方法执行，不允许无限地或者长时间阻止其它方法的调用执行，从而影响应用的 QoS。 
* 由于并发对象被调用者线程所共享，其内部状态必须保证是线程安全的，必须受限于某些线程同步约束，并且这些约束对调用者来说是透明的，不可见的。从调用者的角度来看，并发对象与普通对象没有多大区别。
* 并发对象的设计实现，应该能够透明地利用底层平台所提供的并发机制。这样做的好处是，当并发对象运行在诸如多核处理器这类底层硬件平台上时，我们的应用能够充分挖掘底层平台带来的并发优势，以获得足够好的应用性能。

Active Object 设计模式就是用来解决这些问题的模式。

### 说明

Active Object 设计模式的本质是解耦合方法的调用(Method invocation)与方法的执行(Method execution)，方法调用发生在调用者线程上下文中，而方法的执行发生在独立于调用者线程的 Active Object 线程上下文中。并且重要的一点是，该方法与其它普通的对象成员方法对于调用者来说，没有什么特别的不同。从运行时的角度来看，这里涉及到两类线程，一个是调用者线程，另外一个是 Active Object 线程。在下面对 Android Surfaceflinger 代码的分析中，会更详细地讨论。 

在 Active Object 模式中，有以下6个参与者：
* 代理(Proxy)：代理是 Active Object 所定义的对于调用者的公共接口。运行时，代理运行在调用者线程的上下文中，负责把调用者的方法调用转换成相应的方法请求(Method Request)，并将其插入相应的 Activation List，最后返回给调用者 Future 对象。 
* 方法请求：方法请求定义了方法执行所需要的上下文信息，诸如调用参数等。
* Activation List：负责存储所有由代理创建的，等待执行的方法请求。从运行时来看，Activation List 会被包括调用者线程及其 Active Object 线程并发存取访问。所以，Activation List 实现应当是线程安全的。 
* 调度者(Scheduler)：调度者运行在 Active Object 线程中，调度者来决定下一个执行的方法请求，而调度策略可以基于很多种标准，比如根据方法请求被插入的顺序 FIFO 或者 LIFO，比如根据方法请求的优先级等等。 
* Servant: Servant 定义了 Active Object 的行为和状态，它是 Proxy 所定义的接口的事实实现。 
* Future: 调用者调用 Proxy 所定义的方法，获得 Future 对象。调用者可以从该 Future 对象获得方法执行的最终结果。在真实的实现里，Future 对象会保留一个私有的空间，用来存放 Servant 方法执行的结果。 

下图是 Active Object 模式的原作者对该模式所画的描述图：

![Active Object Design Pattern](https://i.stack.imgur.com/bLo5h.gif)

Active Object 模式的序列图很复杂，参见文档[5]。

### 优点
Active Object 模式还有个好处是它可以将任务(MethodRequest)的提交（调用异步方法）和任务的执行策略(Execution Policy)解耦。任务的执行策略被封装在 Scheduler 的实现类之内，因此它对外是“不可见”的，一旦需要变动也不会影响其它代码，降低了系统的耦合性。任务的执行策略可以反映以下一些问题： 
* 采用什么顺序去执行任务，如 FIFO、LIFO、或者基于任务中包含的信息所定的优先级？
* 多少个任务可以并发执行？
* 多少个任务可以被排队等待执行？
* 如果有任务由于系统过载被拒绝，此时哪个任务该被选中作为牺牲品，应用程序该如何被通知到？
* 任务执行前、执行后需要执行哪些操作？

这意味着，任务的执行顺序可以和任务的提交顺序不同，可以采用单线程也可以采用多线程去执行任务等等。 

### 问题
当然，好处的背后总是隐藏着代价，Active Object 模式实现异步编程也有其代价。该模式的参与者有6个之多，其实现过程也包含了不少中间的处理：MethodRequest 对象的生成、MethodRequest 对象的移动（进出缓冲区）、MethodRequest 对象的运行调度和线程上下文切换等。这些处理都有其空间和时间的代价。因此，Active Object 模式适合于分解一个比较耗时的任务（如涉及 I/O 操作的任务）：将任务的发起和执行进行分离，以减少不必要的等待时间。

* 错误隔离
* 缓冲区监控
* 缓冲区饱和处理策略
* Scheduler 空闲工作者线程清理

# 采用设计模式进行分析和设计

设计模式就是一种对常见问题的通用的可复用的解决方案。

在 Android 图形系统里，大量使用了观察者(Observer)模式，或者说 Publish-Subscribe 模式。在 Android 图形系统里，设计者更喜欢命名为 Listener 类(Source-Listener设计模式)。例如：
* ConsumerBase 类为了及时知道入队(enqueue)事件的到达，订阅(subscribe)/侦听(Listener)了 BufferQueue 类里的消息。 
* SurfaceFlinger 类为了知道 hotplug / vsync 信号到达，订阅(subscribe)/侦听(Listener)了 HWC 类的消息。 

另外，在 Android 系统里，也大量使用了代理(Proxy)模式。最著名最广泛使用的就是： 
* sp implementation - Smart Reference 
* Binder - Remote Proxy 

通过智能指针(smart pointer / 'sp<>')的包装，使得被包装的Android类就具有自我生命周期的管理能力。使得代码简化很多，也更不容易出错。 

Binder 是很典型的跨进程/线程访问对象的应用。 
* Bp 是 Binder Proxy 端； 
* Bn 意味着 Binder Native 端； 
* 这两端会实现相同的接口； 
* Proxy 端只是通过 binder ipc 发送一个 binder transaction； 
* Native 端是真正做事情，再将结果返回。 

但是，在 Android 图形系统的整体设计里，应用得最大的设计模式就是 Producer–Consumer 模式及其变种的模式。

## Producer–Consumer 模式的应用 

从前面图形系统原始的设想中可以看出，Android 图形系统是一个很典型的 Producer–Consumer 模式：
* 客户端做为生产者，负责生产产品：将要被显示的内容；
* 服务端做为消费者，消费产品，将内容放在显示器上显示。 

Producer–Consumer 模式，核心类的就是中间的 BufferQueue 类，实质就是一个 Buffer Pool 的管理。 

![Producer-Consumer-01](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/pattern-design/01-bufferqueue-01.svg)

在 Android 图形系统中，有限的资源是 Graphic Buffer，相对无限的任务是客户生产的内容。因此，Buffer 在 Resource Pool (BufferQueue) 中是循环使用的。同时，从 Graphic Buffer 的状态图得知，它有四种基本状态：Dequeued / Enqueued / Acquired / Free。

在这里，需要考虑到一个情况：Client / Server 两端分别运行在不同的进程/线程里。因此需要考虑： 
* 两端如何同步。 
* 需要对 Buffer Pool 里的空/满状态定义并制定等待/唤醒策略。 

这里提出一个问题：BufferQueue 为什么对于客户端的接口是 Dequeue() / Enqueue()，而对于服务端的接口是 Acquire() / Release()？为什么两端不用同一套接口？ 

这个问题在讨论 GraphicBuffer 类的设计时回答。

Pool 模式的本质就是使用有限的资源去处理相对无限的任务。这里再次用文档[1]里的例子举例说明一下：
在饭店里，能生产的饭菜相对无限，能消费饭菜的顾客也相对无限。但是装饭菜的盘子是有限的，不能卖一道菜就送一个盘子吧，那样商店会亏本的:worried:。因此，饭店里的运转能形象地体现 Producer–Consumer 模式的特点。 

参与者 - Participants
* 店员 - Producer         - SurfaceComoserClient / ANativeWindow 
* 顾客 - Consumer         - SurfaceFlingerConsumer / FramebufferSurface 
* 转盘 - Resource Pool    - BufferQueue 
* 盘子 - Resource         - Graphic Buffer 
* 饭菜 - Resource Content - Content of Graphic Buffer 

顾客什么时候能吃到饭菜？店员炒好菜喊一声，顾客听到(Listen)就去端来消费。 

因此，Consumer 类继承一个 Listener 类，以便在 BufferQueue 类完成 Enqueue() 动作后，Consumer 类会收到 FrameAvailable 消息，然后就可以进行 Acquire() 操作。

![Listener Pattern](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/pattern-design/02-bufferqueue-02.svg) 

不能炒菜的师傅在餐厅招呼客人吧。所以将厨房和餐厅分割开，厨房由大师傅负责炒菜、生产，餐厅由店小二招呼顾客，代理模式。 

因此，因为跨越了进程/线程，在 Producer 类和 BufferQueue 类增加 Proxy(Bp + Bn) 类，简化代码，减少出错的可能。

![Proxy(Bp + Bn)](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/pattern-design/03-bufferqueue-03.svg) 

顾客使用中的盘子，顾客刚使用完的盘子，厨房不能立刻拿来复用吧。所以需要设置等待和检查(Fence)机制，等盘子回收清洗完，干净可用了，厨房才能复用。否则，只能等待。 

另外，饭店里也有一道菜很能体现 Fence 的功能，就是大家经常吃的“铁板烧”。厨师将准备好的配料倒入烧热的铁板里，盖上盖子就可以上菜了。“铁板烧”端到顾客面前时，菜还没有熟。顾客要等待一会儿，等到店小二开盖的时候，表明菜熟了。这时候，顾客关注的是菜什么时候熟，而厨师早就在忙下一道菜了。这里，厨师代表的是生产端的 CPU，顾客代表的是消费端的 CPU，烧热的铁板代表的是 GPU，配料代表配置，菜代表最终的内容，店小二代表 OS。菜是在传菜和顾客的等待中烧熟的。店小二根据信息(如时间和温度)知道菜熟了就开盖，顾客就可以在第一时间知道菜熟了。但是厨师并不关心这道菜真正烧熟的瞬间，他关心的是下一道菜该怎么处理。

因此，Consumer 类会生成一个新的 Fence 类，在 Release() 操作时，跨越进程/线程传递给 Producer 类。Producer 类在 Dequeue() 操作后，在使用 Buffer 类前，需要检查 Fence 类，查看前一阶段对该 Buffer 对象的操作是否真正完成。

![Fence](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/pattern-design/04-bufferqueue-04.svg)

在 Android 4.x 以后的版本里，BufferQueue 类提供了非阻塞的带 Fence 参数返回值的 dequeue() 接口。为什么要提供非阻塞接口？为什么申请到的 Buffer 资源不能立刻使用，需要 Fence 的同步状态？

这是为了增加执行的并发性。因为现代的 SoC 大多是异构架构，执行操作具有异构并行的特点。例如：
* 当 CPU 设置完参数并将一个 Buffer 对象传送给 HWComposer 以后，后面的合成工作将完全由 HWComposer 完成。该 Buffer 对象的所有权已经移交给 HWComposer，没 CPU 什么事了。
* 如果在结果返回前，没有所有权的 CPU 却为此陷入等待，则是浪费了 CPU 的能力。
* 因此 CPU 设置一个 Fence，其实就是一个轻量级的、跨越进程/线程的消息同步机制，当 HWComposer 完成工作时，会发送完成的消息，而 CPU 转身可以做其它事情。
* 当获取了该 Buffer 对象的线程在**真正**要使用该对象前，读取/等待 Fence 的同步消息。
  * 如果前一阶段的工作还没有完成，则该线程陷入读阻塞，直到前面阶段的工作完成。
  * 如果运气好，在进程/线程的调度切换过程中，HWComposer 就已经把合成操作完成，早就释放了该 Buffer 对象的所有权。这样，就没有一个线程被阻塞。

总之，Fence 的同步，意味着**伴随着的对象的所有权的移交**。

顾客来得很多，总有一个先来后到吧，同一个蛋炒饭有很多人点吧。所以需要一个领班，调度器(Scheduler)。调度器的工作是由时钟催促的，VSYNC/trigger。厨房，Producer，也会向调度器(Scheduler)发送各种请求，postMessage。为了提高效率，调度器(Scheduler)对于各种请求，先放在一个文件篮里，MessageQueue，等有机会就处理，handleMessage。

最终，调度器(Scheduler)的工作体现在：在什么时间点，分配(Dispatch)给哪些 Consumer 什么样的 Content，并且这些 Consumer 会以什么样的方式消费这些 Content。

![Scheduler + MessageQueue](https://raw.github.com/shuyong/Design-Of-Android-10.0-Graphic-System/master/document/pattern-design/05-bufferqueue-05.svg)

## Thread Pool 模式的应用

根据系统的需求，上图的 Producer-Consumer 模式已经进化得相当复杂，出现了角色划分。各个角色各司其职，按照各自的轨迹运转，相对独立，又相互配合。这自然而然就会使用 Thread Pool 模式。

Thread Pool 模式是指通过创建多个线程来执行提交到任务队列中的任务。Thread Pool 模式是 Producer-Consumer 模式的变种。其目标就是为了提高吞吐量和提高响应能力。

Android 的图形服务器 SurfaceFlinger 里面的角色就是采用 Thread Pool 模式来实现。跨进程交互的 Binder 接口被限定为 4 个线程。此外，SurfaceFlinger 中每个角色对应一个线程，每个线程从自己的任务队列里顺序处理任务请求。这样既提高了业务吞吐量又提高了任务的响应能力。

## Active Object 模式的应用

在 Android 的图形系统里，每个窗口都是生产者，系统里也可能有多个消费者，如同时存在 LCD、HDMI、WIFI 投屏等多个显示设备。生产者的产品有队列，BufferQueue，请求有队列，MessageQueue，消费者的信息发布也有队列，如 VSYNC 到达，hotplug 事件等待。对这样一个复杂的多任务多线程编程需求，通常的 Thread Pool 模式已经不能应付。于是选择 Active Object 模式就是一种水到渠成的事情。

Active Object 模式既是 Producer-Consumer 模式的变种，也是 Thread Pool 模式的增强。

SurfaceFlinger 就是采用 Active Object 模式实现的。SurfaceFlinger 将轻量级的随机任务，如生产者的请求，放入 MessageQueue 中，这样，生产者可以不陷入等待中，而异步去做其它事情。SurfaceFlinger 将重量级的任务，如窗口合成工作，由 VSYNC 信号定时触发，将以前提交的修改一次性同步完成。同时，引入 Fence 同步机制，不但可以在进程/线程间同步，也可以在异构设备上同步。这样提高了并发性，使得系统效率大大提高。

上述的描述，很容易让人联想到半同步/半异步(Half-sync/Half-async)模式。其实，在开发中，Active Object 模式属于设计模式，而 Half-sync/Half-async 模式属于架构模式。它们两个属于对同一件事情不同视角的描述。半同步/半异步(Half-sync/Half-async)模式是一个分层架构。它包含三个层次：异步任务层、同步任务层和队列层。半同步/半异步(Half-sync/Half-async)模式的核心思想是如何将系统中的任务进行恰当的分解，使各个子任务落入合适的层次中。在 SurfaceFlinger 里，Surface 发送来的请求，数量大，又是轻量级，所以需要异步处理，以提高响应度。对合成操作，要求一致性高，又是重量级，所以在同步任务里完成。

对 SurfaceFlinger 实现的具体分析，将在后面的章节中描述。

## 命令模式(Command Pattern)的应用

命令模式(Command Pattern)，又叫事务(Transaction)模式。它的用途是解耦调用者和接收者，使得命令可以在不同的时间里排队、传输、存储和/或执行。

在 SurfaceFlinger 里，对 Surface 发送来的请求的处理，就是典型的事务(Transaction)模式。 

# 小结

Android 图形系统很复杂，所以需要采用设计模式进行设计和分析。因为需求复杂，所以在大的架构上，是多种设计模式的组合(combination)。

Android 图形系统本质上属于 Producer-Consumer 模式，为了提高吞吐量和提高响应能力，进化成了 Thread Pool 模式。

为了充分挖掘底层平台带来的并发优势，以获得足够好的应用性能，以及足够灵敏的对客户端的反应度，Android 图形服务器 SurfaceFlinger 总体上采用了一种很复杂的设计模式：Active Object 模式。Active Object 模式既是 Producer-Consumer 模式的变种，也是 Thread Pool 模式的增强。

Active Object 模式适合于分解一个比较耗时的任务（如涉及 I/O 操作的任务）：将任务的发起和执行进行分离，以减少不必要的等待时间，增加了并发性。好处是明显的。但是问题也是明显的：太复杂了，参与的角色很多，需要考虑的因素也很多，以至于难以理解和难以实现。具体的问题在后面讨论 SurfaceFlinger 的实现时进行探讨。

# 参考文件
1. [Android graphic system (SurfaceFlinger) : Design Pattern's perspective](https://www.slideshare.net/BinChen3/android-graphic-system-surface-flinger-patternsperspective-external-version)
1. [Object Oriented Design](https://www.oodesign.com/)
1. [Thread pool](https://en.wikipedia.org/wiki/Thread_pool)
1. [Active object](https://en.wikipedia.org/wiki/Active_object)
1. [Active Object Design Pattern](https://www.slideshare.net/jeremiahdjordan/active-object-design-pattern)
1. [Active Object 并发模式在 Java 中的应用](https://www.ibm.com/developerworks/cn/java/j-lo-activeobject/)
1. [Java 多线程编程模式实战指南一：Active Object 模式（上）](https://www.infoq.cn/article/Java-multithreaded-programming-mode-active-object-part1)
1. [Java 多线程编程模式实战指南一：Active Object 模式（下）](https://www.infoq.cn/article/Java-multithreaded-programming-mode-active-object-part2)
1. [Java多线程编程实战指南（设计模式篇）](http://www.broadview.com.cn/book/506)

