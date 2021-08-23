# 用 C4 模型可视化软件架构
* * *

# 基本概念

C4 模型由一系列分层的软件架构图组成，这些架构图用于描述上下文、容器、组件和代码。C4 图的层次结构提供了不同的抽象级别，每种抽象级别都与不同的受众有关。

C4 代表上下文（Context）、容器（Container）、组件（Component）和代码（Code）—— 一系列分层的图表，可以用这些图表来描述不同缩放级别的软件架构，每种图表都适用于不同的受众。

C4 模型是一种在不同抽象层次上交流软件架构的简单方法，可以向不同的受众讲述不同的故事。这也是向软件开发团队介绍（通常是重新引入）严谨和轻量级建模的一种方式。

要为你的代码创建可视化地图，首先需要一组通用的抽象来创建一种无处不在的语言，用来描述软件系统的静态结构。C4 模型使用容器（应用程序、数据存储、微服务等）、组件和代码来描述一个软件系统的静态结构。它还考虑到使用软件系统的人。

* 第 1 层：系统上下文。第 1 层是系统上下文图，它显示了你正在构建的软件系统，以及系统与用户及其他软件系统之间的关系。
* 第 2 层：容器。第 2 层是一个容器图，将软件系统放大，显示组成该软件系统的容器（应用程序、数据存储、微服务等）。技术决策也是该图的关键部分。
* 第 3 层：组件。第 3 层是组件图，将单个容器放大，以显示其中的组件。这些组件映射到代码库中的真实抽象（例如一组代码）。
* 第 4 层：代码。第 4 层是以 UML 类图表示的代码，将单个组件放大，以显示该组件的实现方式。

# 分层用途

1. 系统上下文（System Context）用途

这样一个简单的图，可以告诉我们，要构建的系统是什么；它的用户是谁，谁会用它，它要如何融入已有的IT环境。这个图的受众可以是开发团队的内部人员、外部的技术或非技术人员。即：
a. 构建的系统是什么
b. 谁会用它
c. 如何融入已有的IT环境

2. 系统容器（System Context）用途

这个图的受众可以是团队内部或外部的开发人员，也可以是运维人员。用途可以罗列为：
a. 展现了软件系统的整体形态
b. 体现了高层次的技术决策
c. 系统中的职责是如何分布的，容器间的是如何交互的
d. 告诉开发者在哪里写代码

3. 组件（Component）用途

这个图主要是给内部开发人员看的，怎么去做代码的组织和构建。其用途有：
a. 描述了系统由哪些组件/服务组成
b. 厘清了组件之间的关系和依赖
c. 为软件开发如何分解交付提供了框架

“组件”一词在软件开发行业中是一个超负荷的术语，但在这里，组件是封装在定义良好的接口后面的一组相关功能。如果使用的是 Java 或 C# 之类的语言，那么考虑组件最简单的方式就是它是接口后面的实现类的集合。这些组件的打包方式（例如，每个 JAR 文件、DLL、共享库等一个组件与多个组件的打包方式）是一个单独的、正交的重要事情。

这里需要注意的一点是，容器中的所有组件通常在相同的进程空间中执行。在 C4 模型中，组件不是可单独部署的单元。

4. 代码（Code）用途

以 UML 类图描述代码，指导开发人员进行实现。

# 进一步的解释

C4 分别是上下文（Context）、容器（Container）、组件（Component）和代码（Code）层次的图表，可以用这些图表来描述不同级别的软件架构，每种图表都有着不同的受众。这个就有点像世界地图、国家地图、省地图、市区街道地图，逐层递进，展示里面的内容，而每种地图的关注点也不同。

## 语境图/上下文 (Context)

语境图（有些参考文章里面翻译的是上下文），它展示的是系统大局景观的广角视图，主要包括关键的系统依赖和参与者，关注的重点是人和系统。

在画语境图时，要明确的标出：
* 我们构建的软件系统是什么？
* 谁用这个系统？
* 如何融入已有的IT环境？

## 容器图(Container)

首先明确容器的概念，容器指的是组成软件系统的逻辑上的可执行文件或者过程。容器之间的通信一般是进程间的通信。

所以画一个容器的时候要包含：
* 名称：容器的逻辑名称（如“面向互联网的Web服务器”、“数据库”等）。
* 技术：容器的技术选择（如ApacheTomcat7等）。
* 职责：容器职责的高层次声明或清单。

之后就是画各个容器之间的通信：
* 容器之间交互的目的（如“读/写数据”、“发送报告“等）；
* 容器之间的通信方法（如Web服务、REST、Java远程方法调用、Java消息服务）；
* 容器之间的通信方式（如同步、异步、批量等）；
* 容器之间的协议和端口号（如HTTP、HTTPS、SOAP/HTTP、SMTP、FTP等）。

## 组件图(Component)

组件图就是将单个容器放大，它可以让你清晰的看到容器的关键逻辑组件、组件之间的层级关系和依赖。所以组件图要标明：
* 名称：组件的名称
* 技术：对组件的技术选择
* 职责：对组件职责的高层次的声明

## 类图/代码(Code)

类图是一个可选的细节层次。如果想解释某个组件被怎样实现，可以画更细层次的 UML 类图。

## 补充图

一旦对静态结构有了很好的了解，就可以补充 C4 图以显示其它方面。

### 系统格局图

C4 模型提供单个软件系统的静态视图，但是在现实世界中，软件系统永远不会孤立存在。因此，尤其是在用户负责一组软件系统时，了解所有这些软件系统如何在企业范围内融合在一起通常很有用。为此，只需添加另一个 C4 图“顶部”的图，以从 IT 角度显示系统格局。像系统上下文图一样，该图可以显示组织边界，内部/外部用户和内部/外部系统。

本质上，这是企业级软件系统的高级映射，其中每个感兴趣的软件系统都有 C4 向下钻取。从实践的角度来看，系统格局图实际上只是系统上下文图，而没有特别关注特定的软件系统。

* 适用范围：企业。
* 主要元素：范围内与企业相关的人员和软件系统。
* 目标受众：软件开发团队内部和外部的技术人员和非技术人员。

### 动态图

当用户想显示静态模型中的元素如何在运行时进行协作以实现用户故事，用例，功能等时，动态图可能很有用。该动态图基于UML通讯图 （以前称为“ UML”协作图”）。它类似于UML序列图， 但是它允许带有编号的交互作用的图元素的自由形式排列以指示顺序。

* 适用范围：企业，软件系统或容器。
* 主要和辅助元素：取决于图的范围；企业（请参阅系统架构图），软件系统（请参阅系统上下文或容器图），容器（请参阅组件图）。
* 目标受众：软件开发团队内部和外部的技术人员和非技术人员。

### 部署图 

部署图使用户能够说明如何将静态模型中的容器映射到基础结构。此部署图基于UML部署图，尽管略微简化以显示容器和部署节点之间的映射。部署节点类似于物理基础架构（例如物理服务器或设备），虚拟化基础架构（例如 IaaS，PaaS，虚拟机），容器化基础架构（例如 Docker 容器），执行环境（例如数据库服务器，Java EE）。 Web /应用程序服务器，Microsoft IIS）等。部署节点可以嵌套。

用户可能还希望包括基础结构节点，例如 DNS 服务，负载平衡器，防火墙等。

* 适用范围：单个软件系统。
* 主要元素：范围内软件系统内的部署节点和容器。 
* 支持元素：在软件系统部署中使用的基础结构节点。
* 目标受众：软件开发团队内部和外部的技术人员；包括软件架构师，开发人员，基础架构设计师和运营/支持人员。

# 小结

C4 代表上下文（Context）、容器（Container）、组件（Component）和代码（Code）：

1. 上下文：可以给非技术人员看，画清楚构建什么样的系统、谁使用它，以及如何融入自己已有的东西；
2. 容器：一般给开发人员看，画清楚软件整体形态、关键决策、职责划分，以及在哪里写代码；
3. 组件：一般给内部开发人员看，画清楚系统组成部件、部件之间的层级关系和依赖、细化交付的内容；
4. 代码：同样一般是给内部开发人员看，有时序图、类图、逻辑视图、流程图等等，进入代码细节。
