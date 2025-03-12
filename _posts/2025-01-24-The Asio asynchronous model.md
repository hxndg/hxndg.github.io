---
layout:     post   				    # 使用的布局（不需要改）
title:      2025-01-24-The Asio asynchronous model
subtitle:   NS #副标题
date:       2025-01-24 				# 时间
author:     大狗 						# 作者
header-img: img/post-bg-map.jpg
catalog: true 						# 是否归档
tags:								
    - 学习
---

# The Asio asynchronous model



我这几天在看这篇论文《The Asio asynchronous model》，里面会穿插一部分官方文档（https://www.boost.org/doc/libs/1_87_0/doc/html/boost_asio/overview/core.html），官方文档会打散穿插到这篇论文里面，如果看到标题带有Plus或者++那就是官方文档里面的内容。

## The Asio asynchronous model 

### 1 引论

网络领域长期以来一直采用事件驱动和异步编程设计来开发高效、可扩展、面向网络的软件。基于proactor模型的事件模型，其函数快可视为连续的异步操作，对理解和组合提供了一个很好的概念模型。异步操作可以被连接，每个连接都会启动下一个操作。细粒度的连接可以抽象为单个、更高级别的异步操作连接。
然而，随着异步组合越来越多，纯粹基于回调的方法会大大增加明显的代码复杂度和损害代码可读性。程序员转而使用机制，如状态
机、纤程，C++20基于语言的协程，以提高代码清晰度，同时保留异步实现的好处。这里并没有什么银弹
本文从高层次对Asio库核心的异步模型进行了总结。这种模型将异步操作作为异步组合的基本构建块，但并未使用组合机制。Asio中的异步操作支持回调、future（eager和laze模式）、fiber、协程和尚未想到的方法。从而应用程序程序员根据适当的权衡选择一种方法。

```
//代码，未粘贴
```



### 2 动机

#### 2.1 同步形式作为灵感

最简单的网络程序采用 thread-per-connection 方法实现。这里我们来看看基本的 echo 服务器，下面是一个只用同步函数编写的Echo服务器：

```
```

这个程序的结构和流程很清晰，同步操作都是一个个的函数调用。这些函数带来了很多不错的的句法和语义属性，包括：

- 组合可以使用该语言来管理控制流（即 for、if、while 等）。
- 组合可以重构为使用在同一线程上运行的子函数（也就是直接调用子函数）而不改变功能。
- 如果同步操作需要临时资源（例如内存、文件描述符或线程），此资源在从函数返回之前被释放。
- 当同步操作是泛型的（就是模板）时，返回类型可以从函数及其参数确定性的推到出来
- 要传递给同步操作的参数的生命周期是明确的，即使是传递临时变量。

然而，使用每个线程处理一个连接的方法有几个问题限制了其普遍的可用性。

#### 2.2 线程的可扩展性有限

顾名思义，每个线程一个连接的设计使用单独的线程来处理每个连接。对于处理数千或数百万并发连接的服务器，这代表着程序占据巨量的资源使用，尽管近年来64位的广泛可用性操作系统缓解了这种情况。

从性能敏感的角度考虑，上下文切换的消耗可能更需要考虑在内。在通用操作系统线程之间进行上下文切换的成本以数千个CPU周期来衡量。当可运行线程数量超过执行资源（如 CPU）时，就会发生排队，而最后一个排队的任务会因多次上下文切换的成本而延迟。

// 差个图片

即使网络服务器看起来总体负载较轻，时间相关的事件仍然可能产生排队排队。例如，在金融市场中，所有参与者都在处理和响应相同的市场数据流，因此很可能不止一个参与者会通过向服务器发送交易来响应相同的刺激。这种排队增加了参与者所经历的平均延迟和抖动。

相比之下，专门为事件处理设计的调度程序可以在任务之间“上下文切换”速度提高一到两个数量级，在几十到几百个周期内就结束调度。这里排队可能仍然会发生，但处理相关的队列的总体开销大大减少。

最后，我们还必须注意到，我们的thread-per-connection回显服务器非常简单：线程一旦启动，就能够独立运行。在现实世界的用例中，服务器程序可能需要访问共享数据以响应客户端，处理同步成本、处理CPU之间的数据移动、并增加代码的复杂度。

#### 2.3 半双工和全双工协议


每个线程一个连接的方法对于简单的协议可能比较试用，比方说上面所示的回显服务器，协议是一个半双工协议。服务器要么发送，要么接收，但绝不能同时处理发送和接受。

然而，许多现实世界的应用协议是全双工的，这意味着数据可以在任何时候的任何一个方向传输。考虑一些基于FIX协议的消息：



//插入一些图片



如你所见，像这样的协议需要响应来自许多不同来源的事件。这里面隐藏几个含义：

- 协议逻辑中不同部分，它们可能并发执行（concurrent的含义请自行理解），可能需要访问共享的状态。
- 复杂事件处理流程可能不容易以线性形式表示（例如基于thread-per-connection机制设计，再比方说使用协程时流程）。

因此，我们经常发现这些协议的作者利用其他组合机制，例如状态机，作为管理复杂性和确保正确性的一种方式（状态机能简化理解和设计难度，对于状态机的应用理解有困难的同学，可以去看TCP状态机和TLS状态机，在RFC后面都有图片看。



#### 2.4 快速执行关乎性能表现

一些网络应用程序需要向许多消费者传递一条消息。比方说量化交易需要向所有参与者实时传播市场数据。异步地传递此信息时，一种常见的方法是将消息包装在引用计数指针中（例如shared_ptr）保持内存有效，直到它被传输到所有人再将之释放。

然而，出于效率原因，任何一个传输操作中都尝试投机性发送。符合期望的情况，从统计来说上几乎一直在发生（前提是确保未超出硬件和软件的负载能力），即投机发送数据成功并立刻传输数据。这种情况下，则无需再维护有效的共享指针。这避免了维护引用计数的开销。

相比较而言，原子计数计算成本可衡量为几十个CPU周期，与以数百CPU周期为基本耗时使用系统调用传输数据的操作相比。避免原子计数的额外成本在实践中可以获得5-10%的收益。（我对这句话有点迟疑，因为对网络程序而言，网络耗时可能更夸张，除非真的是纠结CPU周期）

惰性执行模型（lazy execution model）无法避免这种成本，因为它必须在操作第一次时复制共享数据调用。



#### 2.5 设计哲学

上述问题激发了以下异步模型的设计理念：

- 异步模型需要灵活支持组合机制，因为具体的选择因人（用例）而异。
- 尽可能多的支持同步操作的语义和句法属性，因为它们可以更简单地组合和抽象。
- 应用程序代码应该在很大程度上避免线程和同步的复杂性，因为线程 & 同步会带来自不同来源的事件的复杂性。



### 2 Plus

> https://www.boost.org/doc/libs/1_87_0/doc/html/boost_asio/overview/core/async.html

很明显Asio选择的是Proactor模式，和同步或者Reactor模式相比，Asio的Proactor模式的优缺点都很明显。当然Asio也支持Reqctor模式，可以嵌入第三方库，具体不翻译了，参考https://www.boost.org/doc/libs/1_87_0/doc/html/boost_asio/overview/core/reactor.html

先看看Asoi的实现当中有哪些东西。

Proactor模式的关键词（改编自[POSA2]）

- 异步操作：定义一个异步执行的操作，例如对套接字的异步读或写。
- 异步操作处理器（Asynchronous Operation Processor）：执行异步操作，并在操作完成时将事件排入完成事件队列。从高层次来看，像reactive_socket_service这样的内部服务是异步操作处理器。
- 完成事件队列 缓冲完成事件，直到被异步事件解复用器出列。
- 完成处理器 处理异步操作的结果。这些是函数对象，通常使用boost::bind创建（当然我个人有的时候用std::bind)。
- 异步事件解复用器(Asynchronous Event Demultiplexer)阻塞等待完成事件队列上的事件发生，并将已完成的事件返回给调用者。
- Proactor 调用异步事件解复用器出列事件，并分派与事件相关的完成处理程序（即调用函数对象）。这个抽象由io_context类表示。
- Initiator 特定于应用程序的代码，启动异步操作。发起者通过像basic_stream_socket这样的高层次接口与异步操作处理器交互，而后者又委托给像reactive_socket_service这样的服务。

Asio的Proactor模式实际上也是在Reactor模式上实现的

- 异步操作处理器（Asynchronous Operation Processor）使用select、epoll或kqueue实现的Reactor。当Reactor指示资源准备好执行操作时，处理器执行异步操作，并将相关的完成处理器排入完成事件队列。
- 完成事件队列 完成处理程序（即函数对象）的链表。
- 异步事件解复用器 这是通过等待事件或条件变量来实现的，直到完成事件队列中有可用的完成处理程序。

![](https://www.boost.org/doc/libs/1_87_0/doc/html/boost_asio/proactor.png)



- **可移植性**：许多操作系统都提供了原生异步 I/O API（例如Windows上的重叠 I/O），作为开发高性能网络应用程序的首选。这个库可以基于原生异步 I/O 来实现。然而，如果原生支持不可用，这个库也可以使用典型的反应堆模式的同步事件解复用器来实现，例如POSIXselect()。 
- **将线程与并发解耦**：长时间运行的操作由实现代表应用程序异步执行。因此，应用程序不需要生成许多线程来提高并发性。 
- **性能和可扩展性**：诸如每个连接一个线程（这是仅同步方法所需要的）这样的实现策略会降低系统性能，因为会增加 CPU 之间的上下文切换、同步和数据移动。使用异步操作，可以通过最小化操作系统线程的数量（通常是有限的资源），并且只激活有事件要处理的逻辑控制线程，来避免上下文切换的成本。 
- **简化应用程序同步**：异步操作完成处理程序可以编写得好像它们存在于单线程环境中，因此应用程序逻辑可以在很少或不考虑同步问题的情况下开发。 
- **函数组合**：函数组合是指实现函数以提供更高级别的操作，例如以特定格式发送消息。每个函数都是通过对较低级别读或写操作的多次调用来实现的。 例如，考虑一种协议，其中每个消息由固定长度的头部和可变长度的主体组成，主体的长度在头部中指定。一个假设的 read_message 操作可以使用两个较低级别的读取来实现，第一次读取接收头部，一旦知道长度，第二次读取接收主体。 在异步模型中组合函数时，异步操作可以链接在一起。也就是说，一个操作的完成处理程序可以启动下一个操作。可以封装链中的第一个调用，以便调用者不必知道高级操作是作为异步操作链实现的。 以这种方式组合新操作的能力简化了在网络库之上开发更高层次抽象的过程，例如支持特定协议的函数。









### 3 模型

#### 3.1 异步操作（Operation）

> 请注意，这里我将（completion handler)翻译为完成处理器，我之所没翻译为句柄，是因为句柄就有点类似鲁棒，属于你懂了才会理解，不懂根本就不理解的词语

异步操作是Asio异步模型中的基本组合单元。异步操作代表在后台启动和执行的工作，而用户的代码发起的工作可以继续做其他事情。

从概念上讲，异步操作的生命周期可以用以下事件和阶段来描述。

//需要插入一张图片

- 初始化函数是用户调用以启动异步操作，进行初始化操作的函数。
- 完成处理器是用户提供的、可移动（move-only)的函数对象，最多被调用一次，通知异步操作的结果。完成处理器的调用用于通知用户一些事情已经发生了：操作完成，操作的副作用产生了。

初始化函数和完成处理器被嵌入到用户的代码中，用法如下所示：

// 需要插入一张图片

同步操作的表现形式为单个函数，具有几个固有的语义属性。异步操作从这些同步操作选取一部分语意以便于灵活高效的组合。（这句话我觉得主要是针对设计哲学第二条说的，是想说Aiso的一步操作采用了同步操作的相同语意）

| 同步操作属性                                                 | 异步操作相同的语意                                           |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| 当同步操作是泛型的（即模板，其返回类型可由派生自函数及其参数确定 | 当异步操作是泛型时，完成处理器的参数类型和顺序可确定地从初始化函数函数及其参数推导。 |
| 如果同步操作需要临时资源（例如内存、文件描述符或线程），此资源在函数返回之前被释放。 | 如果异步操作需要临时资源（例如内存、文件描述符或线程），此资源在调用完成处理器之前就被释放掉。 |

第二个语意是异步操作的一个重要属性，因为它允许完成处理器在不重叠资源使用的情况下启动进一步的异步操作。想想下面这种琐碎的（也是相对常见的）的情况，在异步操作链中一遍又一遍地重复相同操作：

通过确保在完成处理器运行之前释放资源，我们避免了峰值翻倍运营链的资源使用。（这句话我也没看懂）



> 这里多插一句，Proactor & Reqctor模式是两种传统网络编程模式，这里Boost选择的是Proactor模式。因此这里需要考虑



#### 3.2 异步Agent

异步代理是异步操作的顺序组合。每个异步操作被认为是作为异步代理的一部分运行，即使该代理仅包含该单个操作。异步代理可以与其他异步代理同时执行工作的实体。异步代理之于异步操作就像线程之于同步操作一样。

然而，异步代理是一个纯粹的概念结构，它允许我们理解异步操作上下文如何组织，异步操作如何组合。库中没有“异步代理”这个名词
Agent如何组织异步操作也不重要。我们可以视异步Agent工作流程如下

// 插入图片

异步代理交替地等待异步操作完成，然后运行该操作的完成处理器。在异步Agent的上下文中，这些完成处理器表示不可分割的可调度工作。

#### 3.3 关联特性（Associated characteristics）和关联器（Associators）

> 这部分请注意，关联特性是Associated characteristics，关联器是associators

关联特性指的是异步操作组合为异步Agent工作流程中的一部分时，应当如何表现（就是问how），例如：
- 一个分配器，指明Agent的异步操作如何获取内存资源
- 取消槽（cancellation slot），指明Agent的异步操作取消时采取什么行为
- 一个执行器（executor），它指明代理的完成处理器（completion handlers）将如何排队和运行。

异步操作在异步Agent的执行流程中运行，它的实现代码可能会满足部分关联特性的要求（可以类比为满足golang里面的接口）。这些异步操作的完成处理器需要满足的c++ traits来满足这些关联特性的要求。每种关联特性（characteristic）都具有相应的关联器特征（traits）。

完成处理器的关联器特征（associator trait）可以按照如下的方式给出：

- 接受异步操作提供的默认关联特性（characteristic），按原样返回此默认值
- 返回和关联特性（characteristic）不相关的具体实现，或
- 调整提供的默认特征（characteristic）以引入完成处理器所需的其他行为。

> 如何定义关联器（associator)
>
> 给定一个名为associated_R的关联器特征，它应当具有：
>
> - 默认必须有的S类型的s，用于定义完成处理器，定义完成处理器类型，
> - 定义关联特性的语法（可以理解为函数原型）和语义要求（可以理解为语法上层做什么）的一组类型要求（或C++ concept），称之为R，以及
> - 满足上面要求R的C类型候选值c，由异步操作提供的，代表默认提供的满足关联特性的实现
>
> 异步操作真正计算时使用下述关联器特征：
>
> - 类型的定义，associated_R<S， C>::type
> - 实际实现（值）associated_R<S， C>::get（s，c）
>
> 上面两个满足R中定义的要求。为方便起见，这些也可以通过类型别名访问associated_R_t<S， C>或调用可能变化的函数get_associated_R(s，c)。
>
> 关联器特征的模板应当定义为：
>
> - 如果S::R_type格式良好，定义一个嵌套类型的别名为S::R_type和静态成员函数用于get此类型s.get_R（）
> - 其他情况，如果关联器<associated_R， S，C>::type已经定义清晰，直接继承关联器<associated_R， S，C>即可
> - 其他情况，将嵌套类型别名定义为C，再定义一个静态成员函数get返回c。



#### 3.4 子Agent

> 这段的子Agent和父Agent的关系我实际上感觉读起来乖乖的，总觉得是论文有问题

异步代理中的异步操作可以组合为子Agent（在Aiso里面，异步操作也被称为组合操作）。就父（异步）代理而言，它在等待最终异步操作的完成。构成子Agent的异步操作**依次**运行，子Agent执行完成，最后的完成处理器运行时父Agent才继续运行。

与单个异步操作一样，构成子Agent构建的异步操作必须在调用完成处理器之前释放临时资源。我们可以认为这些子Agent的生命周期在调用完成处理器之前结束。
当异步操作创建子Agent时，它可能会传播父代理的特征到子Agent，然后可以递归传播这些相关特征。这些传递的特征复制了同步操作的另一种属性。

| 同步操作                                                     | 异步操作                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| 同步操作的组合可以重构为在同一个线程使用相同的函数（即简单地调用）而不改变功能 | 异步Agent可以重构为共享父Agent关联特性的异步操作和子代理，而无需改变功能。 |

最终，这些异步操作可以实现为并发运行的多个子代理。在这种情况下，异步操作可以选择选择性地传播父代理的关联特性。



#### 3.5 Executors

每个异步代理都有一个关联的执行器。代理的执行器决定代理的完成处理器如何排队并最终运行。

执行器的例子包括：

- 协调一组操作共享数据结构的异步代理，确保代理的完成处理器永远不会同时运行(Aiso里面这种类型的执行器被称为strand)。
- 确保代理在靠近数据或数据的指定执行资源（例如CPU）上运行事件源（例如NIC）。
- 表示一组相关代理，从而启用动态线程池以进行更智能的调度决策（例如将代理作为一个单元在执行资源之间移动）。
- 将所有完成处理器排队以在GUI应用程序线程上运行，以便它们可以安全地更新用户交互界面元素。
- 返回异步操作的默认执行程序，尽可能快地运行完成处理器在触发操作完成的事件之后。
- 调整异步操作的默认执行器，以便在每个之前和之后运行诸如日志记录、用户授权或异常处理等行为
- 指定异步代理及其完成处理器的优先级。

异步代理中的异步操作使用代理的关联执行器来：

- 在异步操作未完成时，记录异步操作表示的工作的存在。
- 在操作完成时，确保完成处理器进入执行队列。
- 确保完成处理器不会重新运行，如果这样做可能导致疏忽递归和堆栈溢出。

因此，异步代理的关联执行器表示代理应该以何种方式、地点和时间的策略运行，是实际组成代理的代码的横切关注点。

> 这段我觉得读起来也有一些拗口，我目前看到asio的执行器（也包括执行器上下文）就是io_context（执行器上下文），thread_pool等就是单纯的供线程调用的对象，线程在这个上下文上调用boost::asio::io_context::run() 执行相关的任务。如果没有任务，那么boost::asio::io_context::run() 就会直接返回



#### 3.6 Allocators

每个异步代理都有一个关联的分配器。代理的分配器是组成代理的异步操作用以获取每个操作的稳定内存资源（POSM）使用的接口。这个名字反映了这样一个事实：内存是per操作的，因为内存仅在该操作的生命周期内保留；并且保证内存在整个异步操作过程中始终可用。

异步操作可以通过多种不同方式利用POSM：

- 该异步操作不需要任何POSM。例如，该操作包裹了一个现有的API执行自己的内存管理，或者将一些数据拷贝进环形队列。
- 异步操作未完成时，只使用单个固定大小的POSM。例如，将某些状态存储在链表中。
- 该异步操作使用单个，运行时确定大小的POSM。例如，异步操作存储用户提供的缓冲区的副本，或运行时确定大小的iovec结构数组。
- 该操作同时使用多个POSM。例如，链表调用固定大小POSM外加一个用于缓冲区的运行时大小的POSM。
- 操作串行使用多个POSM，大小可能会有所不同

POSM优化是组合异步操作的横切关注点（横切关注点是指在多种模块或组件中重复出现的功能或操作）。此外，使用分配器作为接口来获取POSM授予保证异步操作的实现和调用方的灵活性：

- 代码使用方可以忽略分配器的存在并接受应用程序采用的任何默认策略。
- 代码实现者可以忽略分配器，尤其是在操作不认为是性能的关键点的前提下。
- 用户可以为相关的异步操作共同定位POSM，以获得更好剧不行。
- 对于串行使用不同大小的POSM的组合，内存使用只需调用当前现存的POSM即可。例如，考虑一个短期异步操作的组合使用内存需求大POSM（连接建立和握手），然后进行长寿命异步操作它使用小型POSM（在对等点之间传输数据，存储这些数据）。

如前所述，在调用完成处理器之前必须释放所有资源，从而为代理内其他的后续异步操作回收内存。这允许应用程序即使有长寿命异步代理，也不会热路径（hot-path，“hot path”是指程序或系统中执行频率非常高的代码路径）内存分配，而用户代码并知道关联的分配器存在。



#### 3.7 Cancellation

在Asio中，许多对象，例如套接字和计时器，都支持通过关闭或取消成员函数进行对象范围的取消未完成的异步操作。但是，某些异步操作还支持单独的、有针对性的取消。这种操作取消通过指定异步代理关联的取消槽实现。

为了支持取消，异步操作将取消处理程序安装到代理的插槽中。该取消处理程序是一个函数对象，当用户发出取消信号时将调用它进入插槽。由于取消槽与单个代理相关联，因此该槽最多可容纳一个处理程序时间，安装新的取消处理程序将覆盖任何以前安装的处理程序。因此，相同的插槽可重用于代理内的后续异步操作。（这段我没看懂）

当异步操作包含多个子代理时，取消特别有用。例如，一个子代理可能已完成，另一个随后的子代理立刻被取消，子代理不会造成任何后续的副作用。



#### 3.8 Completion tokens

// 图片

Asio异步模型的一个关键目标是支持多种组合机制。用户通过将完成令牌传递给异步操作的启动函数来调用。按照惯例，完成令牌是异步的操作初始化函数的最后一个参数。

> 这段可能读起来有些拗口，实际上参考https://www.boost.org/doc/libs/1_87_0/doc/html/boost_asio/tutorial/tuttimer2.html的解释，a [completion token](https://www.boost.org/doc/libs/1_87_0/doc/html/boost_asio/overview/model/completion_tokens.html), which determines how the result will be delivered to a **completion handler** when an [asynchronous operation](https://www.boost.org/doc/libs/1_87_0/doc/html/boost_asio/overview/model/async_ops.html) completes.
>
> 完成令牌的实例参考https://www.boost.org/doc/libs/1_87_0/doc/html/boost_asio/tutorial/tuttimer3.html，如何将一些参数绑定传递至completion handler：使用bind将一些参数绑定为function对象，再将包裹之后的function传递为completion token，这里注意一下需要调用的是boost::bind()，如果调用的是std::bind，那么就需要传递boost::asio::placeholders::error，比方说https://www.boost.org/doc/libs/1_87_0/doc/html/boost_asio/tutorial/tutdaytime3.html 的
>
> ```c++
> void start_accept()
>   {
>     tcp_connection::pointer new_connection =
>       tcp_connection::create(io_context_);
> 
>     acceptor_.async_accept(new_connection->socket(),
>         std::bind(&tcp_server::handle_accept, this, new_connection,
>           boost::asio::placeholders::error));
>   }
> ```
>
> 

例如，如果用户传递一个lambda（或其他函数对象）作为完成令牌，异步操作行为如前所述：操作启动，当操作完成时，结果传递给lambda表达式。

当用户将use_future作为完成令牌调用（初始化函数）时，该操作的行为就像是根据调用promise和future对。启动函数并不只启动异步操作，还会返回一个future用于等待结果。

```
future<size_t> f =
 socket.async_read_some(
 buffer, use_future
 );
// ...
size_t n = f.get();
```



类似地，当用户传递use_awaitable为完成令牌，启动函数表现的好像它是一个协程。然而，在这种情况下，启动函数不会启动异步操作。它只返回awaitable对象，这反过来在处于co_await-ed等待状态时，启动操作。

```
awaitable<void> foo()
{
 size_t n =
 co_await socket.async_read_some(
 buffer, use_awaitable
 );
 // ...
}
```



最后一种，将纤程（Fiber）的让出操作作为完成令牌传递，看起来似乎初始化函数能感知到纤程的同步操作：除了开始异步操作，还会阻塞纤程，直到它完成。对纤程而言，这是一个同步操作。

```
void foo()
{
 size_t n = socket.async_read_some(
 buffer, fibers::yield
 );
 // ...
}
```



async_read_some 启动函数的实现需要支持上述所有这些用途。

为了实现这一点，异步操作必须首先指定一个完成签名（completion signature ）（或者简写为签名），该签名描述了将传递给完成处理器的参数。

然后，异步操作的启动函数获取完成签名、完成令牌及其内部实现，并将它们传递给 async_result 关联器特征（trait）。async_result 特征是一个自定义点，它结合这些参数先生成一个具体的完成处理器，然后启动操作。



为了在实践中看到这一点，让我们使用分离线程将同步操作调整为异步操作

```
template <class CompletionToken>
auto async_read_some(tcp::socket& s, const mutable_buffer& b, CompletionToken&& token)
{
 auto init = [](
 auto completion_handler,
 tcp::socket* s,
 const mutable_buffer& b)
 {
 std::thread(
 [](
 auto completion_handler,
 tcp::socket* s,
 const mutable_buffer& b
 )
 {
 error_code ec;
 size_t n = s->read_some(b, ec);
 std::move(completion_handler)(ec, n);
 },
 std::move(completion_handler),
 s,
 b
 ).detach();
 };
 return async_result<
 decay_t<CompletionToken>,
 void(error_code, size_t)
 >::initiate(
 init,
 std::forward<CompletionToken>(token),
 &s,
 b
  );
}
```



我们可以将完成令牌视为一种完成处理器的原型。在我们传递一个函数对象（如 lambda）作为完成令牌，它已经满足完成处理器的要求。async_result 主模板通过简单地转发参数调用我们的“完成处理器原型”来处理这种情况：

```
template <class CompletionToken, completion_signature... Signatures>
struct async_result
{
 template <
 class Initiation,
 completion_handler_for<Signatures...> CompletionHandler,
 class... Args>
 static void initiate(
 Initiation&& initiation,
 CompletionHandler&& completion_handler,
 Args&&... args)
 {
 std::forward<Initiation>(initiation)(
 std::forward<CompletionHandler>(completion_handler),
 std::forward<Args>(args)...);
 }
};
```



我们可以在这里看到，这个默认实现避免了拷贝所有参数，从而确保尽快初始化来实现尽可能高效。
另一方面，惰性完成令牌（如上面use_awaitable）可能会捕获这些参数来延迟异步操作的启动。例如，一个简单的延迟令牌的实现（把异步操作打包，用于稍后的操作）看起来可能像这样：

```
template <completion_signature... Signatures>
struct async_result<deferred_t, Signatures...>
{
 template <class Initiation, class... Args>
 static auto initiate(Initiation initiation, deferred_t, Args... args)
 {
 return [
 initiation = std::move(initiation),
 arg_pack = std::make_tuple(std::move(args)...)
 ](auto&& token) mutable
 {
 return std::apply(
 [&](auto&&... args)
 {
 return async_result<decay_t<decltype(token)>, Signatures...>::initiate(
 std::move(initiation),
 std::forward<decltype(token)>(token),
 std::forward<decltype(args)>(args)...
 );
 },
 std::move(arg_pack)
 );
 };
 }
};
```





#### 3.9 Asio库元素有哪些

RTFM https://www.boost.org/doc/libs/1_87_0/doc/html/boost_asio/reference.html

- completion_signature概念-定义有效的完成签名形式。
- completion_handler_for概念-确定完成处理器是否满足给定的完成签名。
- async_result特征-将完成签名和完成令牌转换为具体的完成处理函数，并启动异步操作。

  > 这里多扯一句，翻译自https://www.boost.org/doc/libs/1_87_0/doc/html/boost_asio/overview/core/streams.html#boost_asio.overview.core.streams.why_eof_is_an_error
  >
  > Boost Asio提供面的向流的 I/O 模型的对象满足以下一种或多种类型要求：
  >
  > - SyncReadStream，在这里同步读取操作是通过一个名为read_some()的成员函数来执行的。
  > - AsyncReadStream，其中异步读取操作是通过一个名为async_read_some()的成员函数执行的。
  > - SyncWriteStream，在这里同步写入操作通过一个名为write_some()的成员函数来执行。
  > - AsyncWriteStream，其中异步写操作是通过一个名为async_write_some()的成员函数来执行的。
  >
  > 不过，比方说TCP这种一般是有流的概念的（换言之不确定边界），可能会发生还没读完就触发对应用层的通知。
  >
  > 但程序通常希望传输确切数量的字节。当出现读取不足或写入不足的情况时，程序必须重新启动操作，并持续进行，直到传输所需数量的字节为止。Boost.Asio 提供了自动执行此操作的通用函数：read()、async_read()、write()和async_write()。
  >
  > 
  >
  > 许多常用的互联网协议是基于行的，这意味着它们具有由字符序列"\r\n"分隔的协议元素。asio可以使用async_read_until来对应这种情况，只需要第三个参数指定终止字符，终止符可以指定为单个char、一个std::string或一个boost::regex。
- async_initiate函数——帮助函数，以简化如何使用async_result特征。
- associator trait ——在代理这个抽象层里通过关联器传播，需要满足特定的trait。
- associated_executor特征-定义异步代理关联的执行器。
- associated_executor_t 模板类型，是上面这个的别名
- get_associated_executor函数
- associated_allocator特征-定义异步代理关联的分配器。
- associated_allocator_t 模板类型，是上面这个的别名
- get_associated_allocator 函数
- associated_cancellation_slot 特征-定义异步代理关联的取消槽。
- associated_cancellation_slot_t 模板类型，是上面这个的别名
- get_associated_cancellation_slot 函数





#### 3.10 高层次抽象

本文提出的异步模型为定义更高级别的抽象提供了基础，但这些概念的定义实际超出了本文的范围。本文的范围仅限于指定用于高层次组合异步操作是什么。

然而，Asio库在此核心模型的基础上提供一些额外的组件，例如：

- 基于此模型，提供异步操作的套接字和计时器。

  > 实际上我感觉还有buffer，就是数据的缓冲区

- 具体执行器，比方说io_context执行器、thread_pool执行器，还有比方说用来保证完成处理器不会并发执行的strand adapter。

  > 我个人觉得io_context是一个非常奇特的存在，它实际上是上下文，不是单纯的executor：比方说strand就可以通过io_context获得。参考https://www.boost.org/doc/libs/1_87_0/doc/html/boost_asio/tutorial/tuttimer5.html
  >
  > ```
  > printer(boost::asio::io_context& io)
  >  : strand_(boost::asio::make_strand(io)),
  >    timer1_(io, boost::asio::chrono::seconds(1)),
  >    timer2_(io, boost::asio::chrono::seconds(1)),
  >    count_(0)
  > {
  > ```
  >
  > 
  >
  > 这里的概念建议参考Asio的官方文档，https://www.boost.org/doc/libs/1_87_0/doc/html/boost_asio/overview/basics.html
  >
  > one **I/O execution context**, such as an `boost::asio::io_context` object, `boost::asio::thread_pool` object, or `boost::asio::system_context`. This **I/O execution context** represents **your program**'s link to the **operating system**'s I/O services.
  >
  >  
  >
  > 关于Strand
  >
  > Strand被定义为事件处理程序的严格顺序调用（即没有并发调用）。链的使用允许在多线程程序中执行代码而无需显式锁定（例如使用互斥锁）。strand可以显式或者非显式的定义，比方说
  >
  > - 仅从一个线程调用 io_context::run()意味着所有事件处理程序在隐式执行链中执行，因为 io_context 保证处理程序仅从 run()内部被调用，毕竟只有一个线程，所以同时只有一个在运行
  > - 如果与连接相关联的一系列异步操作只有单链条，（比方说在像 HTTP 这样的半双工协议实现中），不可能并发执行处理程序。这是一个隐式执行链。 
  > - 显式执行链是strand<>或io_context::strand的实例。所有事件处理函数对象都需要使用boost::asio::bind_executor()绑定到执行链，或者通过执行链对象进行发布/调度。
  >
  > 
  >
  > 在组合异步操作的情况下，例如async_read()或async_read_until()，如果一个完成处理器通过一个 strand，那么所有其他处于中间态的完成处理器也应该通过相同的 strand。这是为了确保对调用者和组合操作（比方说async_read()，调用者可以close()以取消操作，同时async_read又不会出现和当前操作出现冲突）之间共享的任何对象（比方说Socket）进行线程安全访问。
  >
  > 
  >
  > 使用get_associated_executor来获取对应的executor
  >
  > ```
  > boost::asio::associated_executor_t<Handler> a = boost::asio::get_associated_executor(h);
  > 
  > ```
  >
  > 
  >
  > 如果定义这种类型的executor需要定义嵌套类型executor_type和成员函数get_executor()针对特定的处理程序类型进行定制：

- 促进不同组合机制的完成令牌，如协程、纤程、future和deferred操作。

  > 多扯一句，想调试的话，可以定义BOOST_ASIO_ENABLE_HANDLER_TRACKING。具体参考https://www.boost.org/doc/libs/1_87_0/doc/html/boost_asio/overview/core/handler_tracking.html，就不多赘述了

- 对C++协程的高层次支持，将执行器和取消槽结合到一起，以便于协调并发异步代理的coordination。






## 结尾
唉，尴尬

![狗头的赞赏码.jpg](https://i.loli.net/2020/08/27/BFHNyfpx3EsIDUG.jpg)