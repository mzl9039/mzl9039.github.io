---
layout:     post
title:      微服务初探
subtitle:   微服务相关知识学习
date:       2018-1-18 13:50:59
author:     mzl
catalog:    true
tags:
    - micro service
    - API网关
    - API Gateway
    - Microservices
---

{:toc}

**注意：大部分文字内容转自互联网**
# 微服务相关知识
微服务的概念源于 2014 年 3 月5 日 Martin Fowler 所写的一篇文章 [<< Microservices >>](https://www.martinfowler.com/articles/microservices.html)。
文中内容提到：微服务架构是一种架构模式，它提倡将单一应用程序划分成一组小的服务，服务之间互相协调、互相配合，为用户提供最终价值。

原文的中文翻译版见较多，如[这里](http://www.cnblogs.com/liuning8023/p/4493156.html)。

原文中讲到：简而言之，微服务架构风格，就像是把一个单独的应用程序开发为一套小服务，每个小服务运行在自己的进程中，并使用轻量级机制通信，通常是 HTTP API。这些服务围绕业务能力来构建，并通过完全自动化部署机制来独立部署。这些服务使用不同的编程语言书写，以及不同数据存储技术，并保持最低限度的集中式管理。

再简化一些，就是一个词：解耦。即将业务代码解耦，打包成各种各样的小服务。

## 微服务介绍
微服务架构，作为一个名词并没有多大意义，也没有定义某种固定形式的架构设计模式。关于微服务架构与整体服务风格的对比，请参数 Martin Fowler 的文章或中文翻译版本，另外主要参考了DaoCloud社区的 [Chris Richardson 微服务系列文章](http://blog.daocloud.io/microservices-1/)。这里只是摘取部分个人需要的要点。

微服务架构风格，即表示：把应用程序构建为一套服务。这些服务，可以独立部署和扩展，每个服务提供了一个坚实的模块边界，甚至不同的服务可以用不同的编程语言编写。它们可以被不同的团队管理。

具体来说：一个微服务一般完成某个特定的功能，比如订单管理、客户管理等。每个微服务都是一个微型应用，有着自己六边形架构，包括商业逻辑和各种接口。有的微服务通过暴露 API 被别的微服务或者应用客户端所用；有的微服务则通过网页 UI 实现。在运行时，每个实例通常是一个云虚拟机或者 Docker 容器。

整体风格的应用的问题主要有：
1. 系统复杂：多模块紧密耦合，牵一发而动全身
2. 运维困难：变更或升级的影响难以分析，一个小改动可能导致整个应用故障
3. 无法扩展：无法拆分部署，出现性能瓶颈后只能靠增加节点解决

**微服务不是什么新东西，它至少可以追溯到 Unix 的设计原则，只是当时没有被重视**
### 微服务特点
####  组件化与服务

组件：组件（component）是一个可独立替换和升级的软件单元。

微服务架构（Microservice architectures）会使用库（libraries），但组件化软件的主要方式是把它拆分成服务。我们把库（libraries）定义为组件，这些组件被链接到程序，并通过内存中函数调用（in-memory function calls）来调用，而服务（services ）是进程外组件（out-of-process components），他们利用
某个机制通信，比如 WebService 请求，或远程过程调用（remote procedure call）。组件和服务在很多面向对象编程中是不同的概念。

把服务当成组件（而不是组件库）的一个主要原因是，服务可以独立部署。但要注意要服务治理过程中，解耦服务的边界，通过某种进化机制来避免服务接口改变带来的问题（如swagger可以做到发现接口参数变化等）。

另一个考虑是，把服务当组件将拥有更清晰的组件接口。

####  围绕业务功能进行组织

传统形式下，设计一个系统时，将人员划分为 UI 团队，中间件团队，DBA 团队，那么相应地，软件系统也就会自然地被划分为 UI 界面，中间件系统，数据库。我们已经看到的主要问题是，这种组件形式会导致很多的依赖。如果整体应用程序（monolithic applications）跨越很多模块边界（modular boundaries ），那么对于团队的每个成员短期内修复它们是很困难的。

微服务倾向围绕业务功能来分割服务，每个服务都可能包括：用户界面，持久化存储，任何的外部协作。因此，团队是跨职能的（cross-functional）。但微服务更细小的功能分割使快速迭代成为可能，而强大的灵活性则使自动化成为可能，等等。

#### 产品不是项目

传统的项目模式：至力于提供一些被认为是完整的软件。交付一个他们认为完成的软件。软件移交给运维组织，然后，解散构建软件的团队。

微服务（Microservice ）的支持者则提议团队应该负责产品的整个生命周期，即“**你构建，你运维(you build, you run it)**”。

#### 强化终端及弱化通道

微服务的应用致力松耦合和高内聚：采用单独的业务逻辑，表现的更像经典Unix意义上的过滤器一样，接受请求、处理业务逻辑、返回响应。倾向于简单的REST风格，而不是复杂的协议（如WS或BPEL或集中框架）。

通过轻量级消息总线来发布消息。这种的通信协议非常的单一（单一到只负责消息路由），像RabbitMQ或者ZeroMQ这样的简单的实现甚至像可靠的异步机制都没提供，以至于需要依赖产生或者消费消息的终端或者服务来处理这类问题

#### 分散治理

把整体式框架中的组件，拆分成不同的服务，我们在构建它们时有更多的选择。可以使用c++构建高性能组件，也可以使用node.js开发报表页面，一切皆有可能。

#### 分散数据管理

对概念模式下决心进行分散管理时，微服务也决定着分散数据管理。当整体式的应用使用单一逻辑数据库对数据持久化时，企业通常选择在应用的范围内使用一个数据库，这些决定也受厂商的商业权限模式驱动。微服务让每个服务管理自己的数据库：无论是相同数据库的不同实例，或者是不同的数据库系统。这种方法叫Polyglot Persistence。

微服务的数据分析管理，意味着管理数据更新难度更大。传统形式下通过事务处理数据更新的方法不再适用，而分布式事务非常难以实施。因此微服务架构强调服务间事务的协调，并清楚的认识一致性只能是 **最终一致性** 以及通过补偿运算处理问题。

#### 基础项目自动化

许多使用微服务架构的产品或者系统，它们的团队拥有丰富的持集部署以及它的前任持续集成的经验。团队使用这种方式构建软件致使更广泛的依赖基础设施自动化技术。从本地的编译/单元测试/功能测试，到部署服务器的接口测试，集成环境的集成测试，UAT环境的用户接口测试，最后
到性能环境下的性能测试，直到最后发布，需要尽可能自动化甚至全程自动化运行。

微服务的团队更加依赖于基础设施的自动化。

#### 容错设计

以服务作为组件的一个结果在于应用需要有能容忍服务的故障的设计。任务服务可能因为供应商的不可靠而故障，客户端需要尽可能的优化这种场景的响应。跟整体构架相比，这是一个缺点，因为它带来的额外的复杂性。这将让微服务团队时刻的想到服务故障的情况下用户的体验。Netflix 的Simian Army可以为每个应用的服务及数据中心提供日常故障检测和恢复。

由于服务可以随时故障，快速故障检测，乃至，自动恢复变更非常重要。微服务应用把实时的监控放在应用的各个阶段中，检测构架元素（每秒数据库的接收的请求数）和业务相关的指标（把分钟接收的定单数）。监控系统可以提供一种早期故障告警系统，让开发团队跟进并调查。对于微服务框架来说，这相当重要，因为微服务相互的通信可能导致紧急意外行为。

#### 设计改进

决定拆分我们应用的原则是什么呢？首要的因素，组件可以被独立替换和更新的；事实上，许多的微服务小组给它进一步的预期：服务应该能够报废的，而不是要长久的发展的。

### 微服务的不足
1. 我们还没看到足够多的系统运行足够长时间时，我们不能肯定微服务构架是成熟的。
2. 组件化方面的任何努力，其成功都依赖于软件如何拆分成适合的组件。指出组件化的准确边界应该在那，这是非常困难的。
3. 服务边界上的代码迁移是困难的，任务接口的变更需要参与者的共同协作，向后兼容的层次需要被增加，测试也变更更加复杂。

## 微服务涉及的内容
客户端与微服务的直接通信存在如下问题：
1. 客户端需求和每个微服务暴露的细粒度 API 不匹配。如某些情况下，一个客户端需要发送7个独立请求，其它场景下，可能需要发送数百个独立请求。
2. 部分服务使用的协议对 web 并不友好。某个服务可能使用Thrift二进制RPC，另个一个使用AMQP消息传递协议，不管哪种协议对于浏览器或防火墙都不够友好，最好是内部使用。在防火墙之外，应用程序应该使用诸如 HTTP 和 WebSocket 之类的协议。
3. 它会使得微服务难以重构。如当我们想要删除某个服务或合并拆分某些服务时，执行这类重构将很复杂。

### API 网关通信
API 网关是一个服务器，也可以说是进入系统的唯一节点。它封装内部系统的架构，并且提供 API 给各个客户端。它还可能还具备授权、监控、负载均衡、缓存、请求分片和管理、静态响应处理等功能。

* 优点：使用 API 网关的最大优点是，它封装了应用程序的内部结构。
* 不足：它增加了一个我们必须开发、部署和维护的高可用组件。还有一个风险是，API 网关变成了开发瓶颈。

API网关通信要考虑的问题：
* 性能和可扩展性： API 网关构建在一个支持异步、I/O 非阻塞的平台上。JVM上基于NIO的框架，如Netty/Vertx/Spring Reactor/JBoss Undertow; 非JVM的node.js,以及Nginx Plus
* 响应式编程模式：使用传统的异步回调方法编写 API 组合代码会让你迅速坠入回调地狱。代码会变得混乱、难以理解且容易出错。一个更好的方法是使用响应式方法，以一种声明式样式编写 API 网关代码。响应式抽象概念的例子有 Scala 中的 Future、Java 8 中的 CompletableFuture 和 JavaScript 中的Promise
* 服务调用:基于微服务的应用程序是一个分布式系统，必须使用一种进程间通信机制。两种类型的进程间通信机制：一种是使用异步的、基于消息传递的机制；一种进程间通信类型是诸如 HTTP 或 Thrift 那样的同步机制。总之，API 网关需要支持多种通信机制。
* 服务发现：像系统中的其它服务客户端一样，API 网关需要使用系统的服务发现机制，可以是服务器端发现，也可以是客户端发现。需要注意的是，如果系统使用客户端发现，那么 API 网关必须能够查询服务注册中心，这是一个包含所有微服务实例及其位置的数据库。
* 处理局部失败：在实现 API 网关时，还需要处理局部失败的问题。在编写代码调用远程服务方面，Netflix Hystrix 是一个格外有用的库。Hystrix 会暂停超出特定阈限的调用。它实现了一个“断路器（circuit breaker）”模式，可以防止客户端对无响应的服务进行不必要的等待。如果服务的错误率超出了设定的阈值，那么 Hystrix 会启动断路器，所有请求会立即失败并持续一定时间。

### 服务的进行间通信(IPC)
#### 服务之间的交付模式
服务之间的交互问题，需要从两个方面考虑：
1. 一对一还是一对多
2. 交互是同步还是异步

|---|一对一               |一对多             |
|---|:----               |:----             |
|同步|请求/响应        |--                    |
|异步|通知                  |发布/订阅       |
|异步|请求/异步响应|发布/异步响应|

#### 定义API
1. API定义依赖于选定的IPC机制。
2. API的不断进化。注意新老API共存在同一应用中的情况，及相关问题的处理策略。
3. 处理局部失败。分布式系统普遍存在局部失败的问题。为了预防这种问题，设计服务时候必须要考虑部分失败的问题。Netfilix 提供了一个比较好的解决方案，具体的应对措施包括：
    * 网络超时:等待响应时，不设置无限期阻塞，而是采用超时策略，确保资源不被无限期占有。
    * 限制请求的次数:为客户端对某特定服务的请求设置一个访问上限。如果请求已达上限，就要立刻终止请求服务。
    * 断路器模式（Circuit Breaker Pattern）：记录成功和失败请求的数量。如果失效率超过一个阈值，触发断路器使得后续的请求立刻失败。
    * 提供回滚:当一个请求失败后可以进行回滚逻辑。Netflix Hystrix 是一个实现相关模式的开源库。如果使用 JVM，推荐使用Hystrix。

#### IPC技术
现在有很多不同的 IPC 技术。服务间通信可以使用同步的请求/响应模式，比如基于 HTTP 的 REST 或者 Thrift。另外，也可以选择异步的、基于消息的通信模式，比如 AMQP 或者 STOMP。此外，还可以选择 JSON 或者 XML 这种可读的、基于文本的消息格式。当然，也还有效率更高的二进制格式，比如 Avro 和 Protocol Buffer。

##### 基于消息的异步通信
* 消息机制有很多优点：
    * 解耦客户端和服务端：客户端只需要将消息发送到正确的渠道。
    * 消息缓冲：在 HTTP 这样的同步请求/响应协议中，所有的客户端和服务端必须在交互期间保持可用。而在消息模式中，消息中间人将所有写入渠道的消息按照队列方式管理，直到被消费者处理。
    * 客户端-服务端的灵活交互：消息机制支持以上说的所有交互模式。
    * 清晰的进程间通信
* 消息机制也有自己的缺点：
    * 额外的操作复杂性：消息系统需要单独安装、配置和部署。消息broker（代理）必须高可用，否则系统可靠性将会受到影响。
    * 实现基于请求/响应交互模式的复杂性：请求/响应交互模式需要完成额外的工作。每个请求消息必须包含一个回复渠道 ID 和相关 ID。服务端发送一个包含相关 ID 的响应消息到渠道中，使用相关 ID 来将响应对应到发出请求的客户端。

##### 基于请求/响应的同步IPC
使用同步的、基于请求/响应的 IPC 机制的时候，客户端向服务端发送请求，服务端处理请求并返回响应。另外一些客户端可能使用异步的、基于事件驱动的客户端代码，这些代码可能通过 Future 或者 Rx Observable 封装。这个模式中有很多可选的协议，但最常见的两个协议是 REST 和 Thrift。

REST：基于HTTP协议
* 优点：
    * HTTP 非常简单并且大家都很熟悉。
    * 可以使用浏览器扩展（比如 Postman）或者 curl 之类的命令行来测试 API。
    * 内置支持请求/响应模式的通信。
    * HTTP 对防火墙友好。
    * 不需要中间代理，简化了系统架构。
* 缺点：
    * 只支持请求/响应模式交互。尽管可以使用 HTTP 通知，但是服务端必须一直发送 HTTP 响应。
    * 由于客户端和服务端直接通信（没有代理或者缓冲机制），在交互期间必须都保持在线。
    * 客户端必须知道每个服务实例的 URL。这也是个烦人的问题。客户端必须使用服务实例发现机制。
最近社区诞生了包括RAML和Swagger在内的服务框架。Swagger 这样的 IDL 允许定义请求和响应消息的格式，而 RAML 允许使用 JSON Schema 这种独立的规范。对于描述 API，IDL 通常都有工具从接口定义中生成客户端存根和服务端框架。

Apache Thrift:REST的替代品，实现多语言RPC客户端和服务端调用。

##### 消息格式
如果使用消息系统或者 REST，就需要选择消息格式。无论哪种情况，使用跨语言的消息格式非常重要。即便你现在使用单一语言实现微服务，但很有可能未来需要用到其它语言。

目前有文本和二进制这两种主要的消息格式。文本格式包括 JSON 和 XML。这种格式的优点在于不仅可读，而且是自描述的。
文本消息格式的缺点是：
1. 消息会变得冗长，特别是 XML。由于消息是自描述的，所以每个消息都包含属性和值。
2. 解析文本的负担过大。

二进制的格式也有很多。如果使用的是 Thrift RPC，那可以使用二进制 Thrift。如果选择消息格式，常用的还包括 Protocol Buffers 和 Apache Avro，二者都提供类型 IDL 来定义消息结构。差异之处在于 Protocol Buffers 使用添加标记的字段（tagged fields），而 Avro 消费者需要了解模式来解析消息。

### 微服务服务发现
对于基于云端的、现代化的微服务应用而言，服务实例的网络位置都是动态分配的。由于扩展、失败和升级，服务实例会经常动态改变，因此，客户端代码需要使用更加复杂的服务发现机制。服务发现有两大模式：**客户端发现模式** 和 **服务端发现模式**。

#### 客户端发现模式
使用客户端发现模式时，客户端决定相应服务实例的网络位置，并且对请求实现负载均衡。客户端查询服务注册表，后者是一个可用服务实例的数据库；然后使用负载均衡算法从中选择一个实例，并发出请求。

服务实例的网络位置在启动时被记录到服务注册表，等实例终止时被删除。服务实例的注册信息通常使用心跳机制来定期刷新。

客户端发现模式优缺点兼有。这一模式相对直接，除了服务注册外，其它部分无需变动。此外，由于客户端知晓可用的服务实例，能针对特定应用实现智能负载均衡，比如使用哈希一致性。这种模式的一大缺点就是客户端与服务注册绑定，要针对服务端用到的每个编程语言和框架，实现客户端的服务发现逻辑。

#### 服务端发现模式
客户端通过负载均衡器向某个服务提出请求，负载均衡器查询服务注册表，并将请求转发到可用的服务实例。如同客户端发现，服务实例在服务注册表中注册或注销。

AWS Elastic Load Balancer（ELB）是服务端发现路由的例子。HTTP 服务器与类似 NGINX PLUS 和 NGINX 这样的负载均衡起也能用作服务端的发现均衡器。Graham Jenson 的 [Scalable Architecture DR CoN: Docker, Registrator, Consul, Consul Template and Nginx](https://www.airpair.com/scalable-architecture-with-docker-consul-and-nginx) 一文就描述如何使用 Consul Template 来动态配置 NGINX 反向代理。Consul Template 定期从 Consul Template 注册表中的配置数据中生成配置文件；文件发生更改即运行任意命令。在这篇文章中，Consul Template 生成 nginx.conf 文件，用于配置反向代理，然后运行命令，告诉 NGINX 重新加载配置文件。

Kubernetes 和 Marathon 这样的部署环境会在每个集群上运行一个代理，将代理用作服务端发现的负载均衡器。客户端使用主机 IP 地址和分配的端口通过代理将请求路由出去，向服务发送请求。代理将请求透明地转发到集群中可用的服务实例。

服务端发现模式兼具优缺点。它最大的优点是客户端无需关注发现的细节，只需要简单地向负载均衡器发送请求，这减少了编程语言框架需要完成的发现逻辑。这种模式也有缺点。除非负载均衡器由部署环境提供，否则会成为一个需要配置和管理的高可用系统组件。

#### 服务注册表
服务注册表是服务发现的核心部分，是包含服务实例的网络地址的数据库。服务注册表需要高可用而且随时更新。客户端能够缓存从服务注册表中获取的网络地址，然而，这些信息最终会过时，客户端也就无法发现服务实例。因此，服务注册表会包含若干服务端，使用复制协议保持一致性。

Netflix Eureka 是服务注册表的上好案例，为注册和请求服务实例提供了 REST API。服务实例使用 POST 请求来注册网络地址，每三十秒使用 PUT 请求来刷新注册信息。注册信息也能通过 HTTP DELETE 请求或者实例超时来被移除。以此类推，客户端能够使用 HTTP GET 请求来检索已注册的服务实例。

其它的服务注册表：
1. etcd – 高可用、分布式、一致性的键值存储，用于共享配置和服务发现。Kubernetes 和 Cloud Foundry 是两个使用 etcd 的著名项目。
2. consul – 发现和配置的服务，提供 API 实现客户端注册和发现服务。Consul 通过健康检查来判断服务的可用性。
3. Apache ZooKeeper – 被分布式应用广泛使用的高性能协调服务。

#### 服务注册的方式
服务实例必须在注册表中注册和注销。注册和注销有两种不同的方法。
* 方法一是服务实例自己注册，也叫自注册模式（self-registration pattern）；
    * Netflix OSS Eureka 客户端是非常好的案例，它负责处理服务实例的注册和注销。Spring Cloud 能够执行包括服务发现在内的各种模式，使得利用 Eureka 自动注册服务实例更简单，只需要给 Java 配置类注释 @EnableEurekaClient。
    * 自注册模式优缺点兼备。它相对简单，无需其它系统组件。然而，它的主要缺点是把服务实例和服务注册表耦合，必须在每个编程语言和框架内实现注册代码。
* 另一种是采用管理服务实例注册的其它系统组件，即第三方注册模式。
    * 使用第三方注册模式，服务实例则不需要向服务注册表注册；相反，被称为服务注册器的另一个系统模块会处理。服务注册器会通过查询部署环境或订阅事件的方式来跟踪运行实例的更改。一旦侦测到有新的可用服务实例，会向注册表注册此服务。服务管理器也负责注销终止的服务实例。
    * Registrator 是一个开源的服务注册项目，它能够自动注册和注销被部署为 Docker 容器的服务实例。
    * NetflixOSS Prana 是另一个服务注册器，主要面向非 JVM 语言开发的服务
    * 第三方注册模式也是优缺点兼具。在第三方注册模式中，服务与服务注册表解耦合，无需为每个编程语言和框架实现服务注册逻辑；相反，服务实例通过一个专有服务以中心化的方式进行管理。它的不足之处在于，除非该服务内置于部署环境，否则需要配置和管理一个高可用的系统组件。

### 事件驱动
整体架构的应用中，使用单个数据库甚至关系数据库，后者提供了ACID事务，也提供了SQL。但在微服务架构中，每个微服务拥有的数据专门用于该微服务，仅通过其API访问，这种数据封装保证了微服务松散耦合，并且可以独立更新。但如果多个服务访问相同数据，架构更新会耗费时间、也需要所有服务的协调更新。

#### 微服务及分布式数据管理存在的问题
而且，不同的微服务通常使用不同类型的数据库。包括关系数据库/Nosql数据库等等，这给分布式数据管理带来了挑战。
* 如何实现业务逻辑，保持多种服务的一致性。如可能有多个消费者服务消费同一条数据，消费速度快慢会带来查询服务的不一致性。而且，这个过程中也存在消费者服务失败等情况，而大多数NoSQL数据库不支持2PC/3PC等分布式保证数据一致性的协议。
* 如何实现检索多个服务数据的查询。又比如一个查询服务可能涉及模型服务的查询，同时也涉及属性服务的查询，在一些只支持key-value查询的Nosql数据库中，不知道另一个服务中数据的key的情况下，无法检索所需数据。

#### 事件驱动的架构
对于许多应用，解决方案就是事件驱动的架构。在这一架构里，当有显著事件发生时，譬如更新业务实体，某个微服务会发布事件，其它微服务则订阅这些事件。当某一微服务接收到事件就可以更新自己的业务实体，实现更多事件被发布。

事件驱动的架构有优点也有缺点。
* 优点：它使得事务跨多个服务并提供最终一致性，也可以让应用维护物化视图。
* 缺点
    * 它的编程模型要比使用 ACID 事务的更加复杂。为了从应用级别的失效中恢复，还需要完成补偿性事务，例如，如果信用检查不成功则必须取消订单。此外，由于临时事务造成的改变显而易见，因而应用必须处理不一致的数据。此外，如果应用从物化视图中读取的数据没有更新时，也会遇到不一致的问题。
    * 用户必须检测并忽略重复事件。

#### 实现原子化
事件驱动的架构还存在以原子粒度更新数据库并发布事件的问题。如某两个操作需要原子化实现：若在一个服务更新数据库之后，事件发布之前被崩溃了，则系统将变得不一致。确保原子化的标准做法是使用包含数据库和消息代理的分布式事务。然而，基于以上描述的 CAP 理论，这并非我们所想。

##### 使用本地事务发布事件
实现原子化的方法是使用多步骤进程来发布事件，该进程只包含本地事务。诀窍就是在本地存储业务实体状态的数据库中，有一个事件表来充当消息队列。在插入或更新业务实体时，应用启动一个（本地）数据库事务，更新业务实体的状态，并在事件表中插入一个事件，并提交该事务。独立的应用线程或进程查询事件表，将事件发布到消息代理，然后使用本地事务标注事件并发布。

这种方法优缺点兼具。
* 优点:
    * 保证每个更新都有对应的事件发布，并且无需依赖 2PC。
    * 此外，应用发布业务级别的事件，消除了推断事件的需要.
* 缺点:
    * 开发者必须牢记发布事件，因此有很大可能出错。
    * 这一方法对于某些使用 NoSQL 数据库的应用是个挑战，因为 NoSQL 本身交易和查询能力有限。

#### 挖掘数据库事务日志
无需 2PC 实现原子化的另一种方式是由线程或者进程通过挖掘数据库事务或提交日志来发布事件。应用更新数据库，数据库的事务日志记录这些变更。事务日志挖掘线程或进程读取这些日志，并把事件发布到消息代理。
范例：
1. 开源的 LinkedIn Databus 项目
2. AWS DynamoDB 采用的流机制，AWS DynamoDB 是一个可管理的 NoSQL 数据库。每个 DynamoDB 流包括 DynamoDB 表在过去 24 小时之内的时序变化，包括创建、更新和删除操作。应用能够读取这些变更，将其作为事件发布。

* 优点：
    * 能保证无需使用 2PC 就能针对每个更新发布事件。
    * 通过将日志发布于应用的业务逻辑分离，事务日志挖掘能够简化应用。
* 缺点：
    * 事务日志的格式与每个数据库对应，甚至随着数据库版本而变化。
    * 很难从底层事务日志更新记录中逆向工程这些业务事件。

#### 使用事件源
消除更新，只依赖事件。通过采用一种截然不同的、以事件为中心的方法来留存业务实体，事件源无需 2PC 实现了原子化。不同于存储实体的当前状态，应用存储状态改变的事件序列。应用通过重播事件来重构实体的当前状态。每当业务实体的状态改变，新事件就被附加到事件列表。鉴于保存事件是一个单一的操作，本质上也是原子化的。一种简单的理解，类似于spark的DAG，当task失败的时候，由于存储了这个task的DAG，可以从DAG重新恢复这个task，对于这里的事件，由于存储了重建事件状态的所有数据，因为可以重建整个事件。

事件长期保存在事件数据库，使用 API 添加和检索实体的事件。事件存储类似上文提及的消息代理，通过 API 让服务订阅事件，将所有事件传达到所有感兴趣的订阅者。事件存储是事件驱动的微服务架构的支柱。

* 优点：
    * 它解决了实施事件驱动的微服务架构时的一个关键问题，能够只要状态改变就可靠地发布事件。
    * 它也解决了微服务架构中的数据一致性问题。
    * 事件源的另一大优势在于业务逻辑由松耦合的、事件交换的业务实体构成，便于从单体应用向微服务架构迁移。
* 缺点：
    * 采用了不同或不熟悉的编程风格，会有学习曲线。事件存储只直接支持通过主键查询业务实体，用户还需要使用 Command Query Responsibility Segregation (CQRS) 来完成查询。

## 引用的内容
1. [放弃Dubbo，选择最流行的Spring Cloud微服务架构实践与经验总结](http://developer.51cto.com/art/201710/554633.htm)
2. [微服务（Microservices）——Martin Flower【翻译】](http://www.cnblogs.com/liuning8023/p/4493156.html)
3. [Microservices](https://www.martinfowler.com/articles/microservices.html)
4. [Chris Richardson 微服务系列——微服务架构的优势与不足](http://blog.daocloud.io/microservices-1/)
5. [Chris Richardson 微服务系列——使用 API 网关构建微服务](http://blog.daocloud.io/microservices-2/)
6. [Chris Richardson 微服务系列——微服务架构中的进程间通信](http://blog.daocloud.io/microservices-3/)
7. [Chris Richardson 微服务系列——服务发现的可行方案以及实践案例](http://blog.daocloud.io/microservices-4/)
8. [Scalable Architecture DR CoN: Docker, Registrator, Consul, Consul Template and Nginx](https://www.airpair.com/scalable-architecture-with-docker-consul-and-nginx)
