*Kubernetes 原理剖析与实战应用**

[toc]

















# 云原生基石：初识 Kubernetes?



## 01 | 前世今生：Kubernetes 是如何火起来的？



### 云计算平台

说来也巧，“云计算”这个概念也是由 Google 提出的，可见这家公司对计算机技术发展的贡献有多大。自云计算 2006 年被提出后，已经逐渐成为信息技术产业发展的战略重点，你可能也会切身感受到变化。我们平时在讨论技术的时候，经常会被问到诸如“你们公司的业务是否要考虑上云”的问题，而国内相关的云计算大会近几年也如雨后春笋般地召开，可见其有多么火热。

而云计算之所以可以这么快地发展起来，主要原因还是可以为企业带来便利，同时又能降低成本，国内的各大传统型企业的基础设施纷纷向云计算转型，从阿里云、腾讯云每年的发展规模我们就可以看出来云计算市场对人才的需求有多大。

这里，我们可以将经典的云计算架构分为三大服务层：也就是 IaaS（Infrastructure as a Service，基础设施即服务）、PaaS（Platform as a Service，平台即服务）和 SaaS（Software as a Service，软件即服务）。

- IaaS 层通过虚拟化技术提供计算、存储、网络等基础资源，可以在上面部署各种 OS 以及应用程序。开发者可以通过云厂商提供的 API 控制整个基础架构，无须对其进行物理上的维护和管理。

- PaaS 层提供软件部署平台（runtime），抽象掉了硬件和操作系统，可以无缝地扩展（scaling）。开发者只需要关注自己的业务逻辑，不需要关注底层。

- SaaS 层直接为开发者提供软件服务，将软件的开发、管理、部署等全部都交给第三方，用户不需要再关心技术问题，可以拿来即用。

![Drawing 0.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl9DYFeAW4ZYAAF1UiD0s0A670.png)



这样解释起来可能会有点抽象，我们可以想象自己要去一个地方旅行，那么首先就需要解决住的问题，而 IaaS 服务就相当于直接在当地购买了一套商品房，像搭建系统、维护运行环境这种“装修”的事情就必须由我们自己来，但优点是“装修风格”可以自己定。

![Drawing 1.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F9DYGiAV-pgAAcUl1vxuA4620.png)

PaaS 则要简单一点，我们到了一个陌生的城市，可以选择住民宿或青旅，这样就不需要考虑装修和买家具的事情了，系统和环境都是现成的，我们只需要安装自己的运行程序就可以了。

![Drawing 4.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F9DYHOAdfWeAAcj5M-SolM624.png)

而 SaaS 就更简单了，相当于直接住酒店，一切需求都由供应商搞定了，我们只需要选择自己喜欢的房间风格和户型就可以了，这时从操作系统到运行的具体软件都不再需要我们自己操心了。

![Drawing 5.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F9DYH-AHwidAAxW6hYIBhw502.png)

上面，我们了解了云计算的概念，既然上云可以给我们带来这么多的便利，那么我们该如何让系统上云呢？

以前主流的做法就是申请或创建一批云服务（Elastic Compute Service），比如亚马逊的 AWS EC2、阿里云 ECS 或者 OpenStack 的虚拟机，然后通过 Ansible、Puppet 这类部署工具在机器上部署应用。

但随着应用的规模变得越来越庞大，逻辑也越来越复杂，迭代更新也越来越频繁，这时我们就逐渐发现了一些问题，比如：

- **性价比低，资源利用率低**

有时候用户只是希望运行一些简单的程序而已，比如跑一个小进程，为了不相互影响，就要建立虚拟机。这显然会浪费不少资源，毕竟 IaaS 层产品都是按照资源进行收费的。同时操作也比较复杂，花费时间也会比较长；

- **迁移成本高**

如果想要迁移整个自己的服务程序，就要迁移整个虚拟机。显然，迁移过程也会很复杂；

- **环境不一致**

在进行业务部署发布的过程中，服务之间的各种依赖，比如操作系统、开发语言及其版本、依赖的库/包等，都给业务开发和升级带来很大的制约。

如果没有 Docker 的横空出世，这些问题解决起来似乎有些困难。



### Docker

Docker 这个新的容器管理引擎大大降低了使用容器技术的门槛，轻量、可移植、跨平台、镜像一致性保障等优异的特性，一下子解放了生产力。开发者可以根据自己的喜好选择合适的编程语言和框架，然后通过微服务的方式组合起来。交付同学可以利用容器保证交付版本和交付环境的一致性，也便于各个模块的单独升级。测试人员也可以只针对单个功能模块进行测试，加快测试验证的速度。

在某一段时期内，大家一提到 Docker，就和容器等价起来，认为 Docker 就是容器，容器就是Docker。其实容器是一个相当古老的概念，并不是 Docker发明的，但 Docker 却为其注入了新的灵魂——Docker 镜像。

Docker 镜像解决了环境打包的问题，它直接打包了应用运行所需要的整个“操作系统”，而且不会出现任何兼容性问题，它赋予了本地环境和云端环境无差别的能力，这样避免了用户通过“试错”来匹配不同环境之间差异的痛苦过程， 这便是 Docker 的精髓。

它通过简单的 Dockerfile 来描述整个环境，使开发者可以随时随地构建无差别的镜像，方便了镜像的分发和传播。相较于以往通过光盘、U盘、ISO文件等方式进行环境的拷贝复制，Docker镜像无疑把开发者体验提高到了前所未有的高度。这也是 Docker 风靡全球的一个重要原因。

有了 Docker，开发人员可以轻松地将其生产环境复制为可立即运行的容器应用程序，让工作更有效率。越来越多的机构在容器中运行着生产业务，而且这一使用比例也越来越高。我们来看看CNCF （Cloud Native Computing Foundation，云计算基金会）在2019年做的调研报告。

![Drawing 6.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl9DYJiAYUrzAAHq834oYlA699.png)



容器使用数量低于249 个的比例自2018年开始下降了 26%，而使用容器数目高于 250 个的比例增加了 28%。可以预见的是，未来企业应用容器化会越来越常见，使用容器进行交付、生产、部署是大势所趋，也是企业进行技术改造，业务快速升级的利器。



### 我们为什么需要容器调度平台

有了容器，开发人员就只需要考虑如何恰当地扩展、部署，以及管理他们新开发的应用程序。但如果我们大规模地使用容器，就不得不考虑容器调度、部署、跨多节点访问、自动伸缩等问题。

接下来，我们来看看一个容器编排引擎到底需要哪些能力才能解决上述这些棘手的问题。

![Drawing 7.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F9DYKWAW3w_AACD_6ySCwY186.png)



如表所示，首先容器调度平台可以自动生成容器实例，然后是生成的容器可以相邻或者相隔，帮助提高可用性和性能，还有健康检查、容错、可扩展、网络等功能，它几乎完美地解决了需求与资源的匹配编排问题。

既然容器调度平台功能这样强大，市场竞争必定是风云逐鹿的，其实主流的容器管理调度平台有三个，分别是Docker Swarm、Mesos Marathon和Kubernetes，它们有各自的特点。但是同时满足上面所述的八大能力的容器调度平台，其实非 Kubernetes 莫属了。

![Drawing 8.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl9DYMeAJMZBAALljjjSsQ8305.png)



Swarm 是 Docker 公司自己的产品，会直接调度 Docker 容器，并且使用标准的 Docker API 语义，为用户提供无缝衔接的使用体验。 Swarm 更多的是面向于开发者，而且对容错性支持不够好。

Mesos 是一个分布式资源管理平台，提供了 Framework 注册机制。接入的框架必须有一个Framework Scheduler 模块负责框架内部的任务调度，还有一个 Framework Executor 负责启动运行框架内的任务。Mesos 采用了双层调度架构，首先 Mesos 将资源分配给框架，然后每个框架使用自己的调度器将资源分配给内部的各个任务使用，比如 Marathon 就是这样的一个框架，主要负责为容器工作负载提供扩展、自我修复等功能。

Kubernetes 的目标就是消除编排物理或者虚拟计算、网络和存储等基础设施负担，让应用运营商和开发工作者可以专注在以容器为核心的应用上面，同时可以优化集群的资源利用率。Kubernetes 采用了 Pod 和 Label 这样的概念，把容器组合成一个个相互依赖的逻辑单元，相关容器被组合成 Pod 后被共同部署和调度，就形成了服务，这也是 Kuberentes 和其他两个调度管理系统最大的区别。

相对来说，Kubernetes 采用这样的方式简化了集群范围内相关容器被共同调度管理的复杂性。换种角度来说，Kubernetes 能够相对容易的支持更强大、更复杂的容器调度算法。

根据 StackRox 的统计数据表明，Kubernetes 在容器调度领域占据了 86% 的市场份额，虽说Kubernetes 的部署方式千差万别。以前绝大多数人都是采用自建的方式来管理 Kubernetes 集群的，现在已经逐渐采用公有云的 Kubernetes 服务。可见，Kubernetes 越来越成熟，也越来越受到市场的青睐。

![Drawing 9.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F9DYNiARjZ5AAMhoyp7brI603.png)

（https://www.stackrox.com/kubernetes-adoption-and-security-trends-and-market-share-for-containers/）

依据 Google Trends 收集的数据，自 Kubernetes出现以后，便呈黑马态势一路开挂，迅速并牢牢占据头把交椅位置，最终成为容器调度的事实标准。

![Drawing 10.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F9DYOGAH5IcAAF8ZEwwV5s160.png)



### Kubernetes 成为事实标准

2014 年 6 月 7 日的第一个 commit 拉开了 Kubernetes 的序幕。它基于 Google 内部超过 15 年历史的大规模集群管理系统 Borg ，集结其精华，可以对容器进行编排管理、自动部署、弹性伸缩等操作，它于 2015 年 7 月 21 日正式对外发布第一版本，走进了大众视线。

其实，Kubernetes 这个词由 Joe Beda，Brendan Burns和 Craig McLuckie 所创建，源于希腊语的“舵手”或者“领航员”，简称“K8s”，用 8 代替 8 个字符“ubernete”而成的缩写。

经过 6 年的时间，Kubernetes 成为云厂商的“宠儿”，而且国内的诸多大厂已经在生产环境中大规模使用 Kubernetes，用于运行自己的核心业务系统，无数中小企业也都在进行业务容器化探索以及云原生化改造，比如阿里的蚂蚁金服已经有线上业务在使用了。

![Drawing 11.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F9DYQOAIffnAAnOzS-nfj0022.png)



我们来看看 Kubernetes 是如何凭借自身的优势走红的。

首先，**Kubernetes 的成功离不开 Borg**。目前Google 内部依然大规模运行着 Borg，上面跑着我们熟悉的 Google 搜索、Gmail、Youtube 等业务。正是因为从 Borg 当中借鉴了相当多优秀的设计经验，并改进了很多不合理的设计。这些无疑都是促成 Kubernetes 迈向成功的先决条件。Kubernetes 提供了很多方便使用的抽象定义帮助我们对容器进行一些操作，同时可扩展能力强，便于用户定制开发。我们将在后面的课程里慢慢了解到 Kubernetes 这些设计理念和实现方式。

其次，**Kubernetes 并不会跟任何平台绑定，它可以跑在任何环境里**，包括公有云、私有云、物理机，甚至笔记本和树莓派都可以运行，避免了用户对于厂商锁定 (vendor-lockin) 的担忧。

接着，**Kubernetes 的上手门槛很低**。通过 minikube 这类工具，就可以在你的笔记本上快速搭建一套 Kubernetes 集群出来。我会在后面的课程里，教你更多集群的搭建方法，毕竟搭建生产使用的集群还是需要花费一些功夫的。这相比较于 mesos + marathon 方便了很多，你可以更快更直接地上手 Kubernetes。

再次，**Kubernetes 使用了声明式API**，这也是从 Borg 中借鉴来的设计。所谓声明式，就是指你只需要提交一个定义好的 API 对象来声明你所期望的对象是什么样子即可，而无须过多关注如何达到最终的状态。Kubernetes 可以在无外界干预的情况下，完成对实际状态和期望状态的调和（Reconcile）工作。

在Kubernetes中，你可以直接通过 YAML 或者 JSON 进行声明，然后通过 PATCH 的方式就可以完成对 API 对象的多次修改，而无须关心原始 YAML 或 JSON 文件的内容。可以说声明式 API 也是 Kubernetes 项目能够成为事实标准的一个核心所在。

最后，围绕着 Kubernetes 的相关生态异常活跃，它有自己的开发者社区，为开发者提供了交流各种方案的平台，社区开放包容，而且协作程度高。目前 CNCF 也吸引着诸多孵化中的新项目。

可以预见的是，未来Kubernetes的使用会更加普遍，围绕着Kubernetes构建的生态也会越来越好。



**写在最后**

Kubernetes 是为数不多的能够成长为基础技术的技术之一，就像 Linux、OS 虚拟化和 Git一样成为各自领域的佼佼者。简单来说，Kubernetes是如今所有云应用程序开发机构能做出的最安全的投资，如果运用得当，它可以帮助大幅提升开发和交付的速度以及质量。







## 02 | 高屋建瓴：Kubernetes 的架构为什么是这样的？



在本节课程中，我们会一起学习 Kubernetes 的架构设计，以及背后的设计哲学。

Google 使用 Linux 容器有超过 15 年的时间，期间共创建了三套容器调度管理系统，分别是 Borg、Omega 和 Kubernetes。虽然是出于某些特殊诉求偏好先后开发出来的，但是在差异中我们仍然可以看到，后代系统中存在着前一代系统的影子，也就是说，它们之间传承了很多优良的设计。这也是为什么 Kubernetes 在登场之初，就可以吸引到诸多大厂的关注，自此一炮而红，名震江湖。

Kubernetes 的架构设计参考了 Borg 的架构设计，现在我们先来看看 Borg 架构长什么样？



### Borg 的架构

在这里我们不打算深入讲解 Borg 系统的方方面面，如果你有兴趣，可以阅读 Google 在 2015 年发表的这篇 Borg 论文。

我们先来看看 Borg 定义的两个概念，**Cell 和 Cluster**。

Borg 用**Cell 来定义一组机器资源**。Google 内部一个中等规模的 Cell 可以管理 1 万台左右的服务器，这些服务器的配置可以是异构的，比如内存差异、CPU 差异、磁盘空间等。Cell 内的这些机器是通过高速网络进行连通的，以此保证网络的高性能。

Cluster 即集群，一个数据中心可以同时运行一个或者多个集群，每个集群又可以有多个 Cell，比如一个大 Cell 和多个小 Cell。通常来说尽量不要让一个 Cell 横跨两个机房，这样会带来一些性能损失。这也同样适用于 Kubernetes 集群，我们在规划和搭建 Kuberentes 集群的时候要注意这一点。

有了上面这两个概念，我们再来看看下面这幅 Borg 的架构图。

![image (3).png](D:\学习资料\笔记\k8s\k8s图\CgqCHl9E1ASAbkKKAAKzRXDsLZM764.png)

Borg 采用了分布式系统中比较常见的 Server/Agent 架构，主要模块包括 BorgMaster、Borglet 和调度器，这些组件都是通过 C++ 来实现的。

Borglet 运行在每个节点或机器上，这个 agent 程序主要负责任务的启停，如果任务失败了，它会对任务进行重启等一系列操作。运行的这些任务一般都是通过容器来做资源隔离，Borglet 会对任务的运行状态进行监控和汇报。

同时，每个集群中会有对应的 BorgMaster。为了避免单点故障，满足高可用的需要，我们往往需要部署多个 BorgMaster 副本，就如我们图里面显示的那样。这五个副本分别维护一份整个集群状态的内存拷贝，并持久化到一个高可用的、基于 Paxos 的存储上。

通过 Paxos，从这 5 个 BorgMaster 副本中选择出一个 Leader，负责处理整个集群的变更请求，而其余四个都处于 Standby 状态。如果该 Leader 宕机了，会重新选举出另外一个 Leader 来提供服务，整个过程一般在几秒内完成。如果集群规模非常大的话，估计需要近 1 分钟的时间。

BorgMaster 进程主要用于处理跟各个 Borglet 进程进行交互。除此之外，BorgMaster 还提供了Web UI，供用户通过浏览器访问。

Borg 在这里面有个特殊的设计，BorgMaster 和 Borglet 之间的交互方式是 **BorgMaster 主动请求 Borglet** 。即使出现诸如整个 Cell 突然意外断电恢复，也不会存在大量的 Borglet 主动连 BorgMaster，所以就避免了大规模的流量将 BorgMaster 打挂的情况出现。这种方式由 BorgMaster 自己做流控，还可以避免每次 BorgMaster 自己重启后，自己被打挂的情形发生。

我们再来仔细看下架构图，你会发现 BorgMaster 跟 Borglet 之间还加了一层。这是因为 BorgMaster 每次从 Borglet 拿到的数据都是全量的，通过 link shard 模块就可以只把 Borglet 的变化传给 BorgMaster，减少沟通的成本，也降低了 BorgMaster 的处理成本。如果 Borglet 有多次没有对 Borgmaster 的请求进行响应，Borgmaster 就认为运行 Borglet 的这台机器挂掉了，然后对其上的 task 进行重新调度。

下面我们来看看 Kuberenetes 的架构。



### Kubernetes 的架构

Kubernetes 借鉴了 Borg 的整体架构思想，主要由 Master 和 Node 共同组成。

![image (4).png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F9E1BOABW52AAHPVgKdC98447.png)



我们需要注意 Master 和 Node 两个概念。其中 Master 是控制节点，部署着 Kubernetes 的控制面，负责整个集群的管理和控制。Node 为计算节点，或者叫作工作负载节点，每个 Node 上都会运行一些负载容器。

跟 Borg 一样，为了保证高可用，我们也需要部署多个 Master 实例。根据我的生产实践经验，最好为这些 Master 节点选择一些性能好且规格大的物理机或者虚拟机，毕竟控制面堪称 Kubernetes 集群的大脑，要尽力避免这些实例宕机导致集群故障。

同样在 Kubernetes 集群中也采用了分布式存储系统 Etcd，用于保存集群中的所有对象以及状态信息。有的时候，我们会将 Etcd 集群也一起部署到 Master 上。但是在集群节点资源足够的情况下，我个人建议可以考虑将 Etcd 集群单独部署，因为Etcd中的数据可是至关重要的，必须要保证 Etcd 数据的安全。Etcd 采用 Raft 协议实现，和 Borg 中基于 Paxos 的存储系统不太一样。关于 Raft 和 Paxos 这两个协议的差异对比，我们在这里就不展开讲了，你可以通过《 [Paxos 和 Raft 的前世今生](https://cloud.tencent.com/developer/article/1352070)》这篇文章了解一二。



### Kubernetes 的组件

Kubernetes 的控制面包含着 **kube-apiserver、kube-scheduler、kube-controller-manager 这三大组件**，我们也称为 Kubernetes 的三大件。下面我们逐一来讲一下它们的功能及作用。

首先来看 **kube-apiserver**，它是整个 Kubernetes 集群的“灵魂”，是信息的汇聚中枢，提供了所有内部和外部的 API 请求操作的唯一入口。同时也负责整个集群的认证、授权、访问控制、服务发现等能力。

用户可以通过命令行工具 kubectl 和 APIServer 进行交互，从而实现对集群中进行各种资源的增删改查等操作。APIServer 跟 BorgMaster 非常类似，会将所有的改动持久到 Etcd 中，同时也保存着一份内存拷贝。

这也是为什么我们希望 Master 节点可以性能好、资源规格大，尤其是当集群规模很大的时候，APIServer 的吞吐量以及占用的 CPU 和内存都要很大。APIServer 还提供很多可扩展的能力，方便增强自己的功能。

再来看**Kube-Controller-Manager**，它负责维护整个 Kubernetes 集群的状态，比如多副本创建、滚动更新等。Kube-controller-manager 并不是一个单一组件，内部包含了一组资源控制器，在启动的时候，会通过 goroutine 拉起多个资源控制器。这些控制器的逻辑仅依赖于当前状态，因为在分布式系统中没办法保证全局状态的同步。

同时在实现的时候避免使用过于复杂的状态机，因此每个控制器仅仅对自己对应的资源对象做操作。而且控制器做了很多容错处理，比如增加 retry 机制等。

最后来看**Kube-scheduler**，它的工作简单来说就是监听未调度的 Pod，按照预定的调度策略绑定到满足条件的节点上。这个工作虽说看起来是三大件中最简单的，但是做的事情可一点不少。

我们会在后续的课程里，专门讲 Kubernetes 的调度器原理，敬请期待。这个调度器是 Kubernetes 的默认调度器，可插拔，你可以根据需要使用其他的调度器，或者通过目前调度器的扩展功能增加自己的特性进去。

了解完了控制面组件，我们再来看看 Node 节点。一般来说 Node 节点上会运行以下组件。

- 容器运行时主要负责容器的镜像管理以及容器创建及运行。大家都知道的 Docker 就是很常用的容器，此外还有 Kata、Frakti等。只要符合 CRI（Container Runtime Interface，容器运行时接口）规范的运行时，都可以在 Kubernetes 中使用。
- **Kubelet** 负责维护 Pod 的生命周期，比如创建和删除 Pod 对应的容器。同时也负责存储和网络的管理。一般会配合 CSI、CNI 插件一起工作。
- **Kube-Proxy** 主要负责 Kubernetes 内部的服务通信，在主机上维护网络规则并提供转发及负载均衡能力。

除了上述这些核心组件外，通常我们还会在 Kubernetes 集群中部署一些 Add-on 组件，常见的有：

1. **CoreDNS** 负责为整个集群提供 DNS 服务；

2. **Ingress Controller** 为服务提供外网接入能力；

3. **Dashboard** 提供 GUI 可视化界面；

4. **Fluentd + Elasticsearch** 为集群提供日志采集、存储与查询等能力。

了解完 Kubernetes 的各大组件，我们再来看看 Master 和 Node 是如何交互的。



### Master 和 Node 的交互方式

在这一点上，Kubernetes 和 Borg 完全相反。Kubernetes 中所有的状态都是采用上报的方式实现的。APIServer 不会主动跟 Kubelet 建立请求链接，所有的容器状态汇报都是由 Kubelet 主动向 APIServer 发起的。到这里你也许有疑惑，这种方式会不会对 APIServer 产生很大的流量影响？

当集群资源不足的时候，可以按需增加Node 节点。**一旦启动 Kubelet 进程以后，它会主动向 APIServer 注册自己**，这是 Kubernetes 推荐的 Node 管理方式。当然你也可以在Kubelet 启动参数中去掉自动注册的功能，不过一般都是默认开启这个模式的。

一旦新增的 Node 被 APIServer 纳管进来后，**Kubelet 进程就会定时向 APIServer 汇报“心跳”，即汇报自身的状态，包括自身健康状态、负载数据统计**等。当一段时间内心跳包没有更新，那么此时 kube-controller-manager 就会将其标记为**NodeLost**（失联）。这也是 Kubernetes 跟 Borg 有区别的一个地方。

Kubernetes 中各个组件都是以 APIServer 为中心，通过松耦合的方式进行。借助声明式 API，各部件通过 **watch** 的机制就可以根据各个对象的变化，很快地做出相应的处理操作。



### 写在最后

虽说 Kubernetes 跟 Borg 系统有不少差异，但是总体架构还是相似的。从Kubernetes的架构以及各组件的工作模式可以看到，**Kubernetes 系统在设计的时候很注重容错性和可扩展性**。

它假定有发生任何错误的可能，通过 **backoff retry、多副本、滚动升级等机制，增强集群的容错性**，提高 Kubernetes 系统的稳定性。同时对各个组件增加可扩展能力，保证 Kubernetes 对新功能的接入能力，让人们可以对 Kubernetes 进行个性化定制。







## 03 | 集群搭建：手把手教你玩转 Kubernetes 集群搭建



### 在线 Kubernetes 集群

这里介绍一个在线的、免费的 Kubernetes 试用集群。如果你目前手头上没有闲置的物理资源，就可以通过自己的浏览器访问 [Kubernetes Playground](https://www.katacoda.com/courses/kubernetes/playground)，参照里面的说明去搭建。注意，这个集群仅仅可以用作自己的试用环境，千万别在里面保留你的重要数据，因为这个环境随时可能被销毁掉。

这个网站同时还提供了其他一些[交互课程](https://www.katacoda.com/courses/kubernetes)，方便你在线熟悉 Kubernetes 的方方面面。

所谓实践出真知，希望你能动手搭建一套线下环境，这里你是不是想问，自己搭建 Kubernetes 集群难吗？



### Kubernetes 集群搭建难吗？

其实，搭建一个简单自用的 Kubernetes 集群还是比较简单的，只需要配置启动参数即可。但是想要搭建一个生产可用，而且相对安全的集群，可就没那么容易了。

为什么这么说？

对于一个分布式系统而言，要想达到生产可用，**就必须要具备身份认证和权限授权能力**。一般来说，各内部组件之间相互通信会采用自签名的 TLS 证书，通过 HTTPS 来加强安全访问。同时，为了能够确定各自的身份和权限，常常借助于 mTLS （mutual TLS，双向 TLS）认证。

现在，我们来回顾一下上一节课中提到的 Kubernetes 整体架构：

![Drawing 0.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F9ODKeAZpfbAAHPVgKdC98766.png)



我们可以看到，Kubernetes 集群如果要生产可用，就需要签发一些证书，我们具体来看一下都需要哪些证书。

- etcd 集群内部各 member 之间需要一对 CA （Certificate Authority）用于签发证书，然后签发出一对 TLS 证书用于内部各 member 之间的数据互访和同步。
- 同时 etcd 集群需要对外暴露服务，方便 kube-apiserver 可以读写数据，这个时候就需要一对 CA 和 一对 TLS 证书。一般来说，为了方便，我们这里可以使用同一份 CA 证书来签发证书。
- kube-apiserver 跟 etcd 集群之间的访问，我们也需要用 etcd 的 CA 证书，单独为 kube-apiserver 签发一对 TLS 证书。
- Kubernetes 集群内部其他各组件，包括 kube-controller-manager、kube-scheduler、kubelet、kube-proxy 等，与 kube-apiserver 都需要安全访问，因此我们还需要一对 CA 证书，用于签发各个组件的 TLS 证书。同时 kube-apiserver 有时需要主动向 kubelet 发起连接，那么这里还需要为 kube-apiserver 签发一对 TLS 证书。



可见要搭建一个生产可用的 Kubernetes 集群，到目前为止至少需要签发 2 份 CA 证书，8 份 TLS 证书。而实际上 Kubernetes 还支持很多其他的能力，比如 ServiceAccount、aggregation server 等，因此还需要签发更多的证书。

到这里，你可能也意识到了， Kubernetes 集群中有一个“鸡生蛋，蛋生鸡”的问题。那就是，我们应该先设置集群的权限还是先搭建集群呢？而且，在内部各组件接入时该怎么设置权限呢？

其实，在 Kubernetes 中，kube-apiserver 启动时会预设一些权限，用于各内部组件的接入访问。那么各个组件在签发证书时，就需要使用各自预设的 CN（Common Name）来标识自己的身份，比如 kube-scheduler 使用的 CN 是`system:kube-scheduler`。

到这里，我们就知道了证书的签发并不是那么随意，而想要从零开始搭建一套安全性高的集群，其难度远不止如此，我们这里还需要额外考虑到，譬如证书的有效期、过期替换、证书签发的密钥类型、签名算法等等问题。

除了证书签发外，搭建集群还要关注到各个组件的启动参数配置。在那么多的配置参数中，我们该关注哪些呢？

其实，我们可以借助一些工具，它们可以帮助我们轻松解决参数等集群搭建中会遇到的问题，使得搭建更容易，下面我就来为你介绍几种“趁手”的方法。



### 常见的集群搭建方法

我们先来看 [Kind](https://github.com/kubernetes-sigs/kind)，它的名字取自 Kubernetes IN Docker 的简写，Kind 最初仅仅是用来在 Docker 中搭建本地的 Kubernetes 开发测试环境。如果你本地没有太多的物理资源，这个工具比较适合你。使用前，你可以通过 [这个文档](https://docs.docker.com/engine/install/) 安装 Docker；然后在 [官方文档](https://kind.sigs.k8s.io/docs/user/quick-start) 中有安装 kind 的指令，这里我就不赘述啦，你可以在其中了解它的详细使用方法和参数配置，安装的时候注意 kind 的版本号，以及其支持的 Kubernetes 版本。

![Drawing 1.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl9ODL6AT7NmAADHwHBbdN8589.png)

（https://github.com/kubernetes-sigs/kind/blob/master/logo/logo.png）



接下来我们来看一下 [Minikube](https://github.com/kubernetes/minikube)，和 Kind 相比，Minikube 的功能就强大的多了。虽说两者都是用于搭建本地集群的，但是 minikube 支持虚拟化的能力。minikube 可以借助于本地的虚拟化能力，通过 Hyperkit、Hyper-V、KVM、Parallels、Podman、VirtualBox 和 VMWare 等创建出虚拟机，然后在虚拟机中搭建出 Kubernetes 集群来。这里可以根据自己的实际情况，选择合适的 driver。

https://raw.githubusercontent.com/kubernetes/minikube/master/images/logo/logo.png



当然，Minikube 也支持和 Kind 相似的能力，直接利用 Docker [创建集群](https://minikube.sigs.k8s.io/docs/drivers/docker/)：

```sh
$ minikube start --driver=docker \
  --imageRepository=registry.cn-hangzhou.aliyuncs.com/google_containers \
  --imageMirrorCountry=cn
```

关于 Minikube 的其他命令方法，请查阅这份 [官方文档](https://minikube.sigs.k8s.io/docs/handbook/controls/)。



第三个是 [Kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/)，我想给你重点介绍一下它。这个工具是我平时使用最多，也是最推荐你去使用的。上面介绍的 Kind 和 Minikube 这两个工具主要是用于快速搭建本地的开发测试环境，没办法用来搭建生产集群。

![Drawing 3.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F9ODPOAFmDaAAIPc4_Rx-E545.png)



（https://raw.githubusercontent.com/kubernetes/kubeadm/master/logos/horizontal/color/kubeadm-horizontal-color.png）

Kubeadm 是社区官方持续维护的集群搭建工具，在 Kubernertes v1.13 版本的时候就已经 GA 了（GA 即 General Availability，指官方开始推荐广泛使用），它跟着 Kubernetes 的版本一起发布，目前 Kubeadm 代码放在 Kubernetes 的主代码库中。

看到这里，你是不是隐约觉得用 Kubeadm 搭建集群很靠谱，毕竟使用 Kubeadm 随时可以搭建出最新的集群。

没错！这是 Kubeadm 相比于其他集群搭建工具一个很大的“杀手锏”。其他的工具在 Kubernetes 新版本出来以后，都需要做相应的适配工作，开发周期基本上都要晚于社区 1~3 个月的时间。而且这些工具会用自己单独的版本号来标识，所以在使用时，需要额外地注意这些工具的版本跟 Kubernetes 的版本兼容度，很是“令人头大”。



当然 Kubeadm 的能力不止如此，它还有如下这些优势。

- 使用 Kubeadm 可以快速搭建出符合 [一致性测试认证](https://www.cncf.io/certification/software-conformance/)（Conformance Test）的集群。
- Kubeadm 用户体验非常优秀，使用起来非常方便，并且可以用于搭建生产环境，支持 [搭建高可用集群](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/high-availability/)。
- Kubeadm 的代码设计采用了可组合的模块方式，所以你可以只使用 Kubeadm 的部分功能，比如使用 Kubeadm 帮你生成各个组件的证书，也可以基于 kubeadm 开发专属的集群部署工具，比如通过 Ansible 借助于 Kubeadm 的子功能来定制 Kubernetes 集群的搭建。你可以通过 [Kubeadm init phase](https://kubernetes.io/zh/docs/reference/setup-tools/kubeadm/kubeadm-init-phase/) 和 [Kubeadm join phase](https://kubernetes.io/zh/docs/reference/setup-tools/kubeadm/kubeadm-init-phase/)，去了解更多 Kubeadm 创建集群时内部各个子阶段的功能，并根据需要选择合适的子功能。
- 最为关键的是，**Kubeadm 可以向下兼容低一个小版本的 Kubernetes**，也就意味着，你可以用 v1.18.x 的 kubeadm 搭建 v1.17.y 版本的 Kubernetes。
- 同时**kubeadm 还支持集群平滑升级到高版本**，即你可以使用 v1.17.x 版本的 Kubeadm 将 v1.16.y 版本的 Kubernetes 集群升级到 v1.17.z。同理，你可以继续使用高版本的 Kubeadm 将你的集群一点点升到想要的版本上去。



可以看到，Kubeadm 在设计之初的定位就是只关心集群的 bootstrapping，并不负责物理资源的管理和申请。在集群 bootstrapping 搭建完成后，你可以根据自己的需要，在集群中部署自己的 add-on 组件，比如 CNI 插件、Dashboard 等。

![Drawing 4.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl9ODQyAV3kOAAUDNPm292s107.png)



知道了这些，现在我们来详细说一下用 Kubeadm 如何搭建集群。

首先，参照官方文档 [下载安装 Kubeadm 及依赖的组件](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)。然后运行`kubeadm init`在 master 节点上搭建出集群的控制面。

```sh
$ kubeadm init --pod-network-cidr=10.244.0.0/16
```

如果你的 master 节点有多块网卡，可以通过参数 apiserver-advertise-address 来指定你想要暴露的服务地址，比如：

```sh
$ kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.131.128
```

运行完成后，会出现下面这些信息告诉你安装成功，以及一些常规指令：

```sh
Your Kubernetes control-plane has initialized successfully!
To start using your cluster, you need to run the following as a regular user:
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
You should now deploy a Pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  /docs/concepts/cluster-administration/addons/
You can now join any number of machines by running the following on each node
as root:
  kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

我们在 master 节点上拷贝一下 kubeconfig 文件到 kubectl 默认的 kubeconfig 路径下：

```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

然后直接拷贝前面显示的**kubeadm join**命令 ，依次在各个 node 节点上运行将其加入集群中即可。

如果想要搭建生产环境，那么你可以参照这份 [官方文档](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/high-availability/)，去搭建一个高可用的集群，这里就不再赘述了。

除此之外，社区还有一些其他项目可以用来搭建集群，比如 [kubespray](https://github.com/kubernetes-sigs/kubespray) 和 [kops](https://github.com/kubernetes/kops)。其中 kubespray是通过一堆 Ansible Playbook 来安装 Kubernetes 集群；而 kops 使用起来就像 kubectl 一样方便，可以帮助你在各大公有云上搭建 Kubernetes 集群。目前 AWS（Amazon Web Services）官方支持最好的 GCE 和 Openstack 还在 beta 开发阶段。

我们这样就完成了集群搭建，但其实这只是第一步，后续的运维和升级才是“大 Boss”。



### 集群升级

说道集群升级，我们先来了解一下目前社区的版本支持策略。

跟其他开源项目一样，目前 Kubernetes 社区并没有一个所谓的“TLS 版本”。Kubernetes 以 x.y.z 的格式来发布版本，其中 x 是主版本号（major version），y 是小版本号（minor version），而 z 是补丁版本号（patch version）。

Kubernetes 社区异常活跃，每隔三个月就会发布一个小版本，比如从 v1.18 到 v1.19。在这期间，还会有一些补丁版本的迭代在不停地发布，比如 v1.18.1、v1.18.2。每个补丁版本基本上只会涉及一些 bugfix 的工作，新功能都是随着小版本的更新而发布。

但是官方只会维护最新的三个小版本，比如目前最新的小版本是 v1.18，官方就只维护 v1.18、 v1.17 和 v1.16。至于还在使用 v1.15 版本的用户如果想得到社区的支持，就只能升级到 v1.16 版本了。而且如果 v1.19 版本发布了以后，还在使用 v1.16 版本的用户，也要开始考虑升级的问题了。

每一年 Kubernetes 都会发布四个版本，相比较于其他大的开源项目，版本迭代速度可以说非常快了。社区内部其实也有在讨论，要不要放慢版本的迭代速度，改为通用的一年两个小版本策略。不过，到目前为止，这个讨论结果还没有达成统一。

知道了这些，我们来看具体的升级策略，一般来说有三类。

第一种，永远升级到最高最新的版本。这个策略最激进，我不是特别推荐在生产环境中使用。一般每次新的小版本出来后，比如 v1.19.0，通常会带有一些新功能，也隐含着一些 bug，我们可以等后续对应的补丁版本出来后再升级。

第二种，每半年升级一次，这样会落后社区 1~2 个小版本。这是我个人比较推荐的做法，等到各个小版本的补丁版本稳定后，再对集群做升级操作，这样比较保险。

第三种，一年升级一次小版本，或者更长。这样会导致集群落后社区太多，毕竟一年内社区会发布 4 个小版本。



### 集群升级的建议

现在，不少大厂都已经在 Kubernetes 集群中运行着实际的生产业务，而线上的这些业务对可用性的要求通常都非常高，有些场景也异常复杂。因此，即使最微小的集群变更也要非常小心，慎重操作，最好通过“**轮转+灰度**”的升级策略来逐个集群升级。

这样你就会发现，跟随社区版本频繁地进行升级其实很吃力，尤其集群规模比较大的时候，很多大厂其实也吃不消。这往往需要经过多轮的演练、测试，踩完一些“坑”以后，才敢在生产集群进行升级实操，正如上面所说，升级的版本要落后社区至少 1 到 2 个版本。升级的时候，还需要紧密配合监控大盘一起，及时“止血”，避免大规模的生产故障。



在这里，我想给你分享一些集群升级的注意事项。

1. 升级前请务必备份所有重要组件及数据，例如 etcd 的数据备份、各组件的启动配置等。
2. 千万不要跨小版本进行升级，比如直接把 Kubernetes 从 v1.16.x 的版本升到 v1.18.x 的版本。因为社区的一些 API 以及行为改动有些只会保留两个大版本，跨版本升级很容易导致集群故障。
3. 注意观察容器的状态，避免引发某些有状态业务发生异常。在 v1.16 版本以前，升级的时候，如果 Pod spec 的哈希值已更改，则会引发 Pod 重建。

这个 bug 在 v1.16 版本时候已经做了优化，即如果 Pod 是在 v1.16 版本以上创建的，在后续集群升级时，pod spec 的哈希值基本上会保持不变。但是如果 Pod 在低于 v1.16 版本之前创建的，那么每次集群升级时，如果 pod spec 的定义发生了变化，比如新增字段等，都会导致 spec 的哈希值发生变化。



一旦经过了 v1.16 版本的升级，比如从 v1.16 升到 v1.17，后面再次升级时就会避免这种情况了。

1. 每次升级之前，切记一定要**认真阅读官方的 release note**，重点关注中间的 highlight 说明，一般文档中会注明哪些变化会对集群升级产生影响。
2. **谨慎使用还在 alpha 阶段的功能**。社区迭代的时候，这些 alpha 阶段的功能会随时发生变化，比如启动参数、配置方式、工作模式等等。甚至有些 alpha 阶段的功能会被下线掉。

社区推荐的集群升级基本流程：**先升级主控制平面节点，再升级其他控制平面节点，最后升级工作节点**。

这里如果你是使用 Kubeadm 进行集群搭建的，可以参考这份社区官方的 [Kubeadm 集群升级指南](https://kubernetes.io/zh/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)。Kubeadm 在代码开发阶段中，会有对应的 CI 流水线负责验证集群的稳定升级能力。



### 写在最后

集群搭建只是第一步，重要的是后续集群的维护工作，比如集群组件宕机、集群版本升级等。所以选择合适的工具很重要，因为这可以很大程度降低升级的风险以及运维难度。最后我还想再强调一下，**千万不要跨小版本进行升级，要按小版本依次升上来**。







## 04 | 核心定义：Kubernetes 是如何搞定“不可变基础设施”的？



本节课我们会学习 Kubernetes 中最重要、也最核心的对象——Pod。

在了解 Pod 之前，我们先来看一下CNCF 官方是怎么定义云原生的。

> 云原生技术有利于各组织在公有云、私有云和混合云等新型动态环境中，构建和运行可弹性扩展的应用。云原生的代表技术包括容器、服务网格、微服务、**不可变基础设施**和声明式API。
> 这些技术能够构建容错性好、易于管理和便于观察的松耦合系统。结合可靠的自动化手段，云原生技术使工程师能够轻松地对系统作出频繁和可预测的重大变更。

有没有注意到，云原生的代表技术里面提到了一个概念——不可变基础设施（Immutable Infrastructure）。其他的代表技术，像容器、微服务等概念早已深入人心，声明式 API 我们在第一讲 Kubernetes 的前世今生中也有所提及。那么这个不可变基础设施到底是什么含义，又与我们今天要讲的 Pod 有什么关系？



### 怎么理解不可变基础设施？

不可变基础设施，这个名词最早由 Chad Fowler 于 2013 年在他的文章“[Trash Your Servers and Burn Your Code: Immutable Infrastructure and Disposable Components](http://chadfowler.com/2013/06/23/immutable-deployments.html)*”*中提出来。随后，Docker 带来的“容器革命”以及 Kubernetes 引领的“云原生时代”，让不可变基础设施这个概念变得越来越流行。

这里的基础设施，我们可以理解为服务器、虚拟机或者是容器。

跟不可变基础设施相对的，我们称之为**可变基础设施**。在以往传统的开发运维体系中，软件开发完成后，需要工程师或管理员通过SSH 连接到他们的服务器上，然后进行一些脚本安装、deb/rpm 包的安装工作，并逐个机器地调整对应的配置参数及文件。后续还会根据需要对该环境进行不断更改，比如 kernel 升级、配置更新、打补丁等。

随着这种类似变更的操作越来越多，没有人能弄清楚这个环境具体经历了哪些操作，而后续的变更也经常会遇到各种意想不到的诡异事情，比如软件包的循环依赖、参数的配置不一致、版本漂移等问题。

基础设施会变得越来越脆弱、敏感，一些小的改动都有可能引发大的不可预知的结果，这令广大开发者和环境管理员异常抓狂，他们需要凭借自己丰富的技术积累，耗费大量的时间去排查解决。云计算的出现降低了环境标准化的成本，但是业务的交付管理成本依然很高。

通常来说，这种可变基础设施会导致以下问题：

- 持续的变更修改给服务运行态引入过多的中间态，增加了不可预知的风险；
- 故障发生时，难以及时快速构建出新的服务副本；
- 不易标准化，交付运维过程异常痛苦，虽然可以通过 Ansible、Puppet 等部署工具进行交付，但是也很难保证对底层各种异构的环境支持得很好，还有随时会出现的版本漂移问题。比如你可能经常遇到的，某个软件包几个月之前安装还能够正常运行，现在到一个新环境安装后，竟然无法正常工作了。

不可变基础设施则是另一种思路，部署完成以后，便成为一种只读状态，不可对其进行任何更改。如果需要更新或修改，就使用新的环境或服务器去替代旧的。不可变基础设施带来了更一致、更可靠、更可预测的设计理念，可以缓解或完全避免可变基础设施中遇到的各种常见问题。

同时，借助容器技术我们可以自动化地构建出不可变的、可版本化管理的、可一致性交付的应用服务体系，这里包括了标准化实例、运行环境等。还可以依赖持续部署系统，进行应用服务的自动化部署更新，加快迭代和部署效率。

**Kubernetes 中的不可变基础设施就是 Pod。**



### Pod 是什么

Pod 由一个或多个容器组成，如下图所示。Pod 中的容器不可分割，会作为一个整体运行在一个 Node 节点上，也就是说 Pod 是你在 Kubernetes 中可以创建和部署的最原子化的单位。

![image (19).png](D:\学习资料\笔记\k8s\k8s图\CgqCHl9QuZeAeFxvAAtms5EcUcs313.png)



同一个 Pod 中的容器共享网络、存储资源。

- 每个 Pod 都会拥有一个独立的网络空间，其内部的所有容器都共享网络资源，即 IP 地址、端口。内部的容器直接通过 localhost 就可以通信。
- Pod 可以挂载多个共享的存储卷（Volume），这时内部的各个容器就可以访问共享的 Volume 进行数据的读写。

既然一个 Pod 内支持定义多个容器，是不是意味着我可以任意组合，甚至将无关紧要的容器放进来都无所谓？不！这不是我们推荐的方式，也不是使用 Pod 的正确打开方式。

通常来说，如果在一个 Pod 内有多个容器，那么这几个容器最好是密切相关的，且可以共享一些资源的，比如网络、存储等。

我们来看看 [官方文档中给的一个例子](https://kubernetes.io/zh/docs/concepts/workloads/pods/#pod-%E6%80%8E%E6%A0%B7%E7%AE%A1%E7%90%86%E5%A4%9A%E4%B8%AA%E5%AE%B9%E5%99%A8)。这个 Pod 里面运行了两个容器 File Puller 和 Web Server。其中 File Puller 负责定期地从外部 Content Manager 同步内容，更新到挂载的共享存储卷（Volume）中，而 Web Server 只负责对外提供访问服务。两个容器之间通过共享的存储卷共享数据。

![image (20).png](D:\学习资料\笔记\k8s\k8s图\CgqCHl9QuaWAHvj5AACaJbulm4c584.png)



类似这样紧密耦合的业务容器，就比较适合放置在同一个 Pod 中，可以保证很高的通信效率。

一般来说，在一个 Pod 内运行多个容器，比较适应于以下这些场景。

- 容器之间会发生文件交换等，上面提到的例子就是这样。一个写文件，一个读文件。
- 容器之间需要本地通信，比如通过 localhost 或者本地的 Socket。这种方式有时候可以简化业务的逻辑，因为此时业务就不用关心另外一个服务的地址，直接本地访问就可以了。
- 容器之间需要发生频繁的 RPC 调用，出于性能的考量，将它们放在一个 Pod 内。
- 希望为应用添加其他功能，比如日志收集、监控数据采集、配置中心、路由及熔断等功能。这时候可以考虑利用边车模式（Sidecar Pattern），既不需要改动原始服务本身的逻辑，还能增加一系列的功能。比如 Fluentd 就是利用边车模式注入一个对应 log agent 到 Pod 内，用于日志的收集和转发。 Istio 也是通过在 Pod 内放置一个 Sidecar 容器，来进行无侵入的服务治理。



### Pod 背后的设计理念

看完上面 Pod 的存在形式，你也许会有下面两个疑问。

#### 1. 为什么 Kubernetes 不直接管理容器，而用 Pod 来管理呢？

直接管理一个容器看起来更简单，但为了能够更好地管理容器，Kubernetes 在容器基础上做了更高层次的抽象，即 Pod。

因为使用一个新的逻辑对象 Pod 来管理容器，可以在不重载容器信息的基础上，添加更多的属性，而且也方便跟容器运行时进行解耦，兼容度高。比如：

- 存活探针（Liveness Probe）可以从应用程序的角度去探测一个进程是否还存活着，在容器出现问题之前，就可以快速检测到问题；
- 容器启动后和终止前可以进行的操作，比如，在容器停止前，可能需要做一些清理工作，或者不能马上结束进程；
- 定义了容器终止后要采取的策略，比如始终重启、正常退出才重启等；



#### 2.为什么要允许一个 Pod 内可以包含多个容器？

再回答这个问题之前，我们思考一下另外一个问题 “为什么不直接在单个容器里运行多个程序？”。

**由于容器实际上是一个“单进程”的模型**，这点非常重要。因为如果你在容器里启动多个进程，这将会带来很多麻烦。不仅它们的日志记录会混在一起，它们各自的生命周期也无法管理。毕竟只有一个进程的 PID 可以为 1，如果 PID 为 1 的进程这个时候挂了，或者说失败退出了，那么其他几个进程就会自然而然地成为“孤儿”，无法管理，也无法回收资源。

很多公司在刚开始容器化改造的时候，都会这么去使用容器，把容器当作 VM 来使用，有时候也叫作**富容器模式**。这其实是一种非常不好的尝试，也不符合不可变基础设施的理念。我们可以接受将富容器当作容器化改造的一个短暂的过渡形态，但不能将其作为改造的终态。后续，还需要进一步对这些富容器进行拆分、解耦。

看到这里，第二个问题的答案已经呼之欲出了。用一个 Pod 管理多个容器，既能够保持容器之间的隔离性，还能保证相关容器的环境一致性。使用粒度更小的容器，不仅可以使应用间的依赖解耦，还便于使用不同技术栈进行开发，同时还可以方便各个开发团队复用，减少重复造轮子。



### 如何声明一个 Pod

在 Kubernetes 中，所有对象都可以通过一个相似的 API 模板来描述，即元数据 （metadata）、规范（spec）和状态（status）。这种方式也是从 Borg 吸取的经验，避免过多的 API 定义设计，不利于统一和对接。Kubernetes 有了这种统一风格的 API 定义，方便了通过 REST 接口进行开发和管理。



#### 元数据（metadata）

metadata 中一般要包含如下 3 个对该对象至关重要的元信息：namespace（命名空间）、name（对象名）和 uid（对象 ID）。

- namespace是 Kubernetes 中比较重要的一个概念，是对一组资源和对象的抽象集合，namespace 主要用于逻辑上的隔离。Kubernetes 中有几个内置的 namespace：
  - **default**，这是默认的缺省命名空间；
  - **kube-system**，主要是部署集群最关键的核心组件，比如一般会将 CoreDNS 声明在这个 namespace 中；
  - **kube-public**，是由 kubeadm 创建出来的，主要是保存一些集群 bootstrap 的信息，比如 token 等；
  - **kube-node-lease**，是从 v1.12 版本开始开发的，到 v1.14 版本变为 beta 可用版本，在 v1.17 的时候已经正式 GA 了，它要用于 node 汇报心跳，每一个节点都会有一个对应的 Lease 对象。

- 对象名比较好理解，就是用来**标识对象的名称，在 namespace 内具有唯一性**，在不同的 namespace 下，可以创建相同名字的对象。
- uid 是由系统自动生成的，主要用于 Kubernetes 内部标识使用，比如某个对象经历了删除重建，单纯通过名字是无法判断该对象的新旧，这个时候就可以通过 uid 来进行唯一确定。

当然， Kubernetes 中并不是所有对象都是 namespace 级别的，还有一些对象是集群级别的，并不需要 namespace 进行隔离，比如 Node 资源等。

除此以外，还可以在 metadata 里面用各种标签 （labels）和注释（annotations）来标识和匹配不同的对象，比如用户可以用标签`env=dev`来标识开发环境，用`env=testing`来标识测试环境。



#### 规范 （Spec）

在 Spec 中描述了该对象的详细配置信息，即用户希望的状态（Desired State）。Kubernetes 中的各大组件会根据这个配置进行一系列的操作，将这种定义从“抽象”变为“现实”，我们称之为调和（Reconcile）。用户不需要过度关心怎么达到终态，也不用参与。



#### 状态（Status）

在这个字段里面，包含了该对象的一些状态信息，会由各个控制器定期进行更新。也是不同控制器之间进行相互通信的一个渠道。在 Kubernetes 中，各个组件都是分布式部署的，围绕着 kube-apiserver 进行通信，那么不同组件之间进行信息同步，就可以通过 status 进行。像 Node 的 status 就记录了该节点的一些状态信息，其他的控制器，就可以通过 status 知道该 Node 的情况，做一些操作，比如节点宕机修复、可分配资源等。

现在我们来看一个 Pod 的 API 长什么样子。



#### 一个 Pod 的真实例子

下面是我用 Yaml 写的一个 Pod 定义，我做了注释让你一目了然：

```yaml
apiVersion: v1 #指定当前描述文件遵循v1版本的Kubernetes API
kind: Pod #我们在描述一个pod
metadata:
  name: twocontainers #指定pod的名称
  namespace: default #指定当前描述的pod所在的命名空间
  labels: #指定pod标签
    app: twocontainers
  annotations: #指定pod注释
    version: v0.5.0
    releasedBy: david
    purpose: demo
spec:
  containers:
  - name: sise #容器的名称
    image: quay.io/openshiftlabs/simpleservice:0.5.0 #创建容器所使用的镜像
    ports:
    - containerPort: 9876 #应用监听的端口
  - name: shell #容器的名称
    image: centos:7 #创建容器所使用的镜像
    command: #容器启动命令
      - "bin/bash"
      - "-c"
      - "sleep 10000"
```

你可以通过 kubectl 命令在集群中创建这个 Pod。kubectl 的功能比较强大、也比较灵活。

```sh
$ kubectl create -f ./twocontainers.yaml
$ kubectl get pods
NAME                      READY     STATUS    RESTARTS   AGE
twocontainers             2/2       Running   0          7s
```

创建出来后，稍微等待一下，我们就可以看到，该 Pod 已经运行成功了。现在我们可以通过 exec 进入shell这个容器，来访问sise服务：

```sh
$ kubectl exec twocontainers -c shell -i -t -- bash
[root@twocontainers /]# curl -s localhost:9876/info
{"host": "localhost:9876", "version": "0.5.0", "from": "127.0.0.1"}
```



### 写在最后

Pod 是 Kubernetes 项目中实现“容器设计模式”的最佳实践之一，也是 Kubernetes 进行复杂应用编排的基础依赖。引入 Pod 主要基于可管理性和资源共享的目的，希望你能够仔细理解和揣摩 Pod 的这种设计思想，对今后的容器化改造颇有受益。







## 05 | K8s Pod：最小调度单元的使用进阶及实践



在实际生产使用的过程中，通过 kubectl 可以很方便地部署一个 Pod。但是 Pod 运行过程中还会出现一些意想不到的问题，比如：

- Pod 里的某一个容器异常退出了怎么办？
- 有没有“健康检查”方便你知道业务的真实运行情况，比如容器运行正常，但是业务不工作了？
- 容器在启动或删除前后，如果需要做一些特殊处理怎么办？比如做一些清理工作。
- .......

在了解 Pod 的高阶用法之前，我们先聊聊 Pod 的运行状态。



### Pod 的运行状态

```yaml
apiVersion: v1 #指定当前描述文件遵循v1版本的Kubernetes API
kind: Pod #我们在描述一个pod
metadata:
  name: twocontainers #指定pod的名称
  namespace: default #指定当前描述的pod所在的命名空间
  labels: #指定pod标签
    app: twocontainers
  annotations: #指定pod注释
    version: v1
    releasedBy: david
    purpose: demo
spec:
  containers:
  - name: sise #容器的名称
    image: quay.io/openshiftlabs/simpleservice:0.5.0 #创建容器所使用的镜像
    ports:
    - containerPort: 9876 #应用监听的端口
  - name: shell #容器的名称
    image: centos:7 #创建容器所使用的镜像
    command: #容器启动命令
      - "bin/bash"
      - "-c"
      - "sleep 10000"
```



我们通过 kubectl 创建 Pod 成功后，可以通过如下命令看到 Pod 的状态:

```sh
$ kubectl get pod twocontainers -o=jsonpath='{.status.phase}'
Pending
```

> 注：我们这里使用了 kubectl 命令行 JSONPATH 模板能力，你可以将这条命令当作一个 tip，在日常工作中使用。

我们看到，这个时候 Pod 处于Pending状态，具体的值来自 Pod 对象的`status.phase`字段。

你也可以使用 `kubectl get` 命令来查看容器的状态：

```sh
$ kubectl get pod twocontainers
NAME            READY   STATUS              RESTARTS   AGE
twocontainers   0/2     ContainerCreating   0          13s
```

看到这里，你会发现这个地方显示的是ContainerCreating，这和上面的Pending不一致啊！先别急，我们来 describe 一下（这里我只截取跟 Pod 状态最相关的片段）：

```sh
$ kubectl describe pod twocontainers
Name:         twocontainers
Namespace:    default
...
Status:       Pending
IP:
IPs:          <none>
Containers:
  sise:
    Container ID:
    Image:          quay.io/openshiftlabs/simpleservice:0.5.0
    ...
    State:          Waiting
      Reason:       ContainerCreating
    Ready:          False
    Restart Count:  0
    ...
  shell:
    Container ID:
    Image:         centos:7
    Image ID:
    ...
    State:          Waiting
      Reason:       ContainerCreating
    Ready:          False
    ...
...
Events:
  Type    Reason     Age        From               Message
  ----    ------     ----       ----               -------
  Normal  Scheduled  <unknown>  default-scheduler  Successfully assigned default/twocontainers to node-1
  Normal  Pulling    3m57s      kubelet, node-1    Pulling image "quay.io/openshiftlabs/simpleservice:0.5.0"
```

可以看到，这边 Status 依然是**Pending**。其实这是 kubectl 在显示时做的转换，它会遍历容器的 State，如果容器的状态为**Waiting**的话，就读取**State.Reason**字段作为 Pod 的 Status。这个时候由于镜像在本地不存在，需要去镜像中心拉取。

一般来说，处于**Pending**状态的 Pod，不外乎以下 2 个原因：

1. Pod 还未被调度；
2. Pod 内的容器镜像在待运行的节点上不存在，需要从镜像中心拉取。

等待镜像拉取结束，再来查看 Pod 的状态，已经变为**Running**状态。

```sh
$ kubectl get pod twocontainers -o=jsonpath='{.status.phase}'
Running
$ kubectl describe pod twocontainers
Name:         twocontainers
Namespace:    default
...
Start Time:   Wed, 26 Aug 2020 16:49:11 +0800
...
Status:       Running
...
Containers:
  sise:
    Container ID:   docker://4dc8244a19e6766b151b36d986b9b3661f3bf05260aedd2b76dd5f0fcd6e637f
    Image:          quay.io/openshiftlabs/simpleservice:0.5.0
    Image ID:       docker-pullable://quay.io/openshiftlabs/simpleservice@sha256:72bfe1acc54829c306dd6683fe28089d222cf50a2df9d10c4e9d32974a591673
    ...
    State:          Running
      Started:      Wed, 26 Aug 2020 17:00:52 +0800
    Ready:          True
    ...
  shell:
    Container ID:  docker://1b6137b4cef60d0309412f5cdba7f0ff743ee03c1112112f6aadd78f9981bbaa
    Image:         centos:7
    Image ID:      docker-pullable://centos@sha256:19a79828ca2e505eaee0ff38c2f3fd9901f4826737295157cc5212b7a372cd2b
    ...
    State:          Running
      Started:      Wed, 26 Aug 2020 17:01:46 +0800
    Ready:          True
    ...
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
...
```

这个时候，就标志着 Pod 内的所有容器均被创建出来了，且至少有一个容器为在运行状态中。那么如果想知道 Pod 内所有的容器是否都在运行中呢？我们可以通过 kubectl get 来看到：

```sh
$ kubectl get pod twocontainers
NAME            READY   STATUS    RESTARTS   AGE
twocontainers   2/2     Running   0          2m
```

在这里，我们看到 `2/2`。前一个 2 表示目前正在运行的容器数量，后一个 2 表示定义的容器数量。当这两个数值相等的时候，就可以标识着 Pod 内所有容器均正常运行。



Pod 的 Status 除了上述的`Pending`、`Running`以外，官方还定义了下面这些状态：

- `Succeeded`来表示 Pod 内的所有容器均成功运行结束，即正常退出，退出码为 0；
- `Failed`来表示 Pod 内的所有容器均运行终止，且至少有一个容器终止失败了，一般这种情况，都是由于容器运行异常退出，或者被系统终止掉了；
- `Unknown`一般是由于 Node 失联导致的 Pod 状态无法获取到。

既然 Pod 内的容器会出现异常退出状态，那么有没有一些重启策略可以让 Kubelet 对容器进行重启呢？



### Pod 的重启策略

Kubernetes 中定义了如下三种重启策略，可以通过`spec.restartPolicy`字段在 Pod 定义中进行设置。

- Always 表示一直重启，这也是默认的重启策略。Kubelet 会定期查询容器的状态，一旦某个容器处于退出状态，就对其执行重启操作；
- OnFailure 表示只有在容器异常退出，即退出码不为 0 时，才会对其进行重启操作；
- Never 表示从不重启；

> 注：在 Pod 中设置的重启策略适用于 Pod 内的所有容器。

虽然我们可以设置一些重启策略，确保容器异常退出时可以重启。但是对于运行中的容器，是不是就意味着容器内的服务正常了呢？

比如某些 Java 进程启动速度非常慢，在容器启动阶段其实是无法提供服务的，虽然这个时候该容器是处于运行状态。

再比如，有些服务的进程发生阻塞，导致无法对外提供服务，这个时候容器对外还是显示为运行态。

那么我们该如何解决此类问题呢？有没有一些方法，比如可以通过一些周期性的检查，来确保容器中运行的业务没有任何问题。



### Pod 中的健康检查

为此，Kubernetes 中提供了一系列的健康检查，可以定制调用，来帮助解决类似的问题，我们称之为 Probe（探针）。

目前有如下三种 Probe：

- **livenessProbe **可以用来探测容器是否真的在“运行”，即“探活”。如果检测失败的话，这个时候 kubelet 就会停掉该容器，容器的后续操作会受到其 [重启策略](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy) 的影响。
- **readinessProbe** 常常用于指示容器是否可以对外提供正常的服务请求，即“就绪”，比如 nginx 容器在 reload 配置的时候无法对外提供 HTTP 服务。
- **startupProbe** 则可以用于判断容器是否已经启动好，就比如上面提到的容器启动慢的例子。我们可以通过参数，保证有足够长的时间来应对“超长”的启动时间。 如果检测失败的话，同**livenessProbe** 的操作。这个 Probe 是在 1.16 版本才加入进来的，到 1.18 版本变为 beta。也就是说如果你的 Kubernetes 版本小于 1.18 的话，你需要在 kube-apiserver 的启动参数中，显式地在 feature gate 中开启这个功能。可以参考 [这个文档](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/feature-gates/) 查看如何配置该参数。

如果某个 Probe 没有设置的话，我们默认其是成功的。

为了简化一些通用的处理逻辑，Kubernetes 也为这些 Probe 内置了如下三个 Handler：

- **ExecAction** 可以在容器内执行 shell 脚本；
- **HTTPGetAction** 方便对指定的端口和 IP 地址执行 HTTP Get 请求；
- **TCPSocketAction** 可以对指定端口进行 TCP 检查；

在这里 Probe 还提供了其他配置字段，比如 failureThreshold （失败阈值）等，你可以到 [这个官方文档](https://kubernetes.io/zh/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#configure-probes) 中查看更详细的解释。

> 注：对于每一种 Probe，Kubelet 只会执行其中一种 Handler。如果你定义了多个 Handler，则会按照 Exec、HTTPGet、TCPSocket 的优先级顺序，选择第一个定义的 Handler。

下面我们通过一个例子，来了解这三个 Probe 的工作流程。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probe-demo
  namespace: demo
spec:
  containers:
  - name: sise
    image: quay.io/openshiftlabs/simpleservice:0.5.0
    ports:
    - containerPort: 9876
    readinessProbe:
      tcpSocket:
        port: 9876
      periodSeconds: 10
    livenessProbe:
      periodSeconds: 5
      httpGet:
        path: /health
        port: 9876
    startupProbe:
      httpGet:
        path: /health
        port: 9876
      failureThreshold: 3
      periodSeconds: 2
```

在这个例子中，我们在命名空间 demo 下面创建了一个名为 probe-demo 的 Pod。在这个 Pod 里，我们配置了三种 Probe。在 Kubelet 创建好对应的容器以后，会先运行 startupProbe 中的配置，这里我们用 HTTP handler 每隔 2 秒钟通过 http://localhost:9876/health 来判断服务是不是启动好了。这里我们会尝试 3 次检测，如果 6 秒以后还未成功，那么这个容器就会被干掉。而是否重启，这就要看 Pod 定义的重启策略。

一旦容器通过了 startupProbe 后，Kubelet 会每隔 5 秒钟进行一次探活检测 （livenessProbe），每隔 10 秒进行一次就绪检测（readinessProbe）。

**在平常使用中，建议你对全部服务同时设置 readiness 和 liveness 的健康检查。**

有一点需要注意的是，通过 TCP 对端口进行检查，仅适用于端口已关闭或者进程停止的情况。因为即使服务异常，只要端口是打开状态，健康检查依然是通过的。

除了健康检查以外，我们有时候在容器退出前要做一些清理工作，比如利用 Nginx 自带的停止功能停掉进程，而不是强制杀掉该进程，这可以避免一些正在处理的请求中断。此时我们就需要一个 hook（钩子程序）来帮助我们达到这个目的了。



### 容器生命周期内的 hook

目前在 Kubernetes 中，有如下两种 hook。

- PostStart 可以在容器启动之后就执行。但需要**注意的是，此 hook 和容器里的 ENTRYPOINT 命令的执行顺序是不确定的。**
- PreStop 则在容器被终止之前被执行，是一种阻塞式的方式。执行完成后，Kubelet 才真正开始销毁容器。

同上面的 Probe 一样，hook 也有类似的 Handler：

- Exec 用来执行 Shell 命令；
- HTTPGet 可以执行 HTTP 请求。

我们来看个例子：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
  namespace: demo
spec:
  containers:
  - name: lifecycle-demo-container
    image: nginx:1.19
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo Hello from the postStart handler > /usr/share/message"]
      preStop:
        exec:
          command: ["/usr/sbin/nginx","-s","quit"]
```

可以看出来，我们可以借助preStop以优雅的方式停掉 Nginx 服务，从而避免强制停止容器，造成正在处理的请求无法响应。



### init 容器    

在 Kubernetes 中还有一种特殊的容器，即 init 容器。看名字就知道，这个容器工作在正常容器（为了方便区分，我们这里称为应用容器）启动之前，通常用来做一些初始化工作，比如环境检测、OSS 文件下载、工具安装，等等。

应用容器专注于业务处理，其他一些无关的初始化任务就可以放到 init 容器中。这种解耦有利于各自升级，也降低相互依赖。

一个 Pod 中允许有一个或多个 init 容器。init 容器和其他一般的容器非常像，其与众不同的特点主要有：

- 总是运行到完成，可以理解为一次性的任务，不可以运行常驻型任务，因为会 block 应用容器的启动运行；
- 顺序启动执行，下一个的 init 容器都必须在上一个运行成功后才可以启动；
- 禁止使用 readiness/liveness 探针，可以使用 Pod 定义的**activeDeadlineSeconds**，这其中包含了 Init Container 的启动时间；
- 禁止使用 lifecycle hook。

我们来看一个 Init 容器的例子：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  namespace: demo
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.31
    command: [‘sh’, ‘-c’, ‘echo The app is running! && sleep 3600‘]
  initContainers:
  - name: init-myservice
    image: busybox:1.31
    command: ['sh', '-c', 'until nslookup myservice; do echo waiting for myservice; sleep 2; done;']
  - name: init-mydb
    image: busybox:1.31
    command: ['sh', '-c', 'until nslookup mydb; do echo waiting for mydb; sleep 2; done;']
```

在 myapp-container 启动之前，它会依次启动 init-myservice、init-mydb，分别来检查依赖的服务是否可用。



### 写在最后

其实作为 Kubernetes 内部最核心的对象之一，Pod 承载了太多的功能。 为了增加可扩展、可配置性，Kubernetes 增加了各种 Probe、Hook 等，以此方便使用者进行接入配置。所以在一开始使用的时候，会觉得 Pod 中配置项太多。

这里的 init container 有很多用处，比如优化系统内核参数，像ingress-nginx这种镜像，我们可以通过init container 进行tcp连接的优化，以及系统打开最大的文件句柄数的优化等。







# Kubernetes 进阶：部署高可用的业务



## 06 | 无状态应用：剖析 Kubernetes 业务副本及水平扩展底层原理



每一个 Pod 都是应用的一个实例，但是通常来说你不会直接在 Kubernetes 中创建和运行单个 Pod。因为 Pod 的生命周期是短暂的，即“用后即焚”。理解这一点很重要，这也是“不可变基础设施”这一理念在 Kubernetes 中的最佳实践。同样，对于你后续进行业务改造和容器化上云也具有指导意义。

当一个 Pod 被创建出来，不管是由你直接创建，还是由其他工作负载控制器（Workload Controller）自动创建，经过调度器调度以后，就永久地“长”在某个节点上了，直到该 Pod 被删除，或者因为资源不够被驱逐，抑或由于对应的节点故障导致宕机等。因此单独地用一个 Pod 来承载业务，是没办法保证高可用、可伸缩、负载均衡等要求，而且 Pod 也无法“自愈”。

这时我们就需要在 Pod 之上做一层抽象，通过多个副本（Replica）来保证可用 Pod 的数量，避免业务不可用。在介绍 Kubernetes 中对这种抽象的实现之前，我们先来看看应用的业务类型。



### 有状态服务 VS 无状态服务

一般来说，业务的服务类型可分为无状态服务和有状态服务。举个简单的例子，像打网络游戏这类的服务，就是有状态服务，而正常浏览网页这类服务一般都是无状态服务。其实判断两种请求的关键在于，**两个来自相同发起者的请求在服务器端是否具备上下文关系**。

- 如果是有状态服务，其请求是状态化的，服务器端需要保存请求的相关信息，这样每个请求都可以默认地使用之前的请求上下文。
- 而无状态服务就不需要这样，每次请求都包含了需要的所有信息，每次请求都和之前的没有任何关系。

有状态服务和无状态服务分别有各自擅长的业务类型和技术优势，**在 Kubernetes 中，分别有不同的工作负载控制器来负责承载这两类服务。**



### Kubernetes 中的无状态工作负载

Kubernetes 中各个对象的 metadata 字段都有 [label](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/labels/)（标签）和 [annotation](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/annotations/)（注解） 两个对象，可以用来标识一些元数据信息。**不论是 label 还是 annotation，都是一组键值对，键值不允许相同**。

- **annotation** 主要用来记录一些非识别的信息，并不用于标识和选择对象。
- **label** 主要用来标识一些有意义且和对象密切相关的信息，用来支持**labelSelector**（标签选择器）以及一些查询操作，还有选择对象。

为了让这种抽象的对象可以跟 Pod 关联起来，**Kubernetes 使用了labelSelector来跟 label 进行选择匹配，从而达到这种松耦合的关联效果**。

```sh
$ kubectl get pod -l label1=value1,label2=value2 -n my-namespace
```

比如，我们就可以通过上述命令，查询出 my-namespace 这个命名空间下面，带有标签`label1=value1`和`label2=value2`的 pod。label 中的键值对在匹配的时候是“**且**”的关系。



#### ReplicationController

Kubernetes 中有一系列的工作负载可以用来部署无状态服务。在最初，Kubernetes 中使用了ReplicationController来做 Pod 的副本控制，即确保该服务的 Pod 数量维持在特定的数量。为了简洁并便于使用，ReplicationController通常缩写为“rc”，并作为 kubectl 命令的快捷方式，例如：

```sh
$ kubectl get rc -n my-namespace
```

如果副本数少于预定的值，则创建新的 Pod。如果副本数大于预定的值，就删除多余的副本。因此即使你的业务应用只需要一个 Pod，你也可以使用 rc 来自动帮你维护和创建 Pod。



#### ReplicaSet

随后社区开发了下一代的 Pod 控制器 ReplicaSet（可简写为 rs） 用来替代 ReplicaController。虽然 ReplicaController 目前依然可以使用，但是社区已经不推荐继续使用了。这两者的功能和目的完全相同，但是 ReplicaSet 具备更强大的[基于集合的标签选择器](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/labels/#基于集合-的需求)，这样你可以通过一组值来进行标签匹配选择。目前支持三种操作符：`in`、`notin`和`exists`。

例如，你可以用`environment in (production, qa)`来匹配 label 中带有`environment=production`或`environment=qa`的 Pod。

或者你可以用 partition来匹配 label 中带有 partition 这个 key 的 Pod。

了解了标签选择器，我们就可以通过如下的 kubectl 命令查找 Pod：

```sh
$ kubectl get pods -l environment=production,tier=frontend
```

虽然 Replicaset 可以独立使用，但是为了能够更好地协调 Pod 的创建、删除以及更新等操作，我们都是直接**使用更高级的 Deployment**来管理 Replicaset，社区也是一直这么定位和推荐的。比如一些业务升级的场景，使用单一的 ReplicaController 或者 Replicaset 是无法实现滚动升级的诉求，至少需要定义两个该对象才能实现，而且这两个对象使用的标签选择器中的 label 至少要有一个不相同。通过不断地对这两个对象的副本进行增减，也可以称为**调和**（Reconcile），才可以完成滚动升级。这样使用起来不方便，也增加了用户的使用门槛，极大地降低了业务发布的效率。



#### Deployment

通过 Deployment，我们就不需要再关心和操作 ReplicaSet 了。

![image (3).png](D:\学习资料\笔记\k8s\k8s图\CgqCHl9Z5ayANEsBAAPzg_vpPBs686.png)

Deployment、ReplicaSet 和 Pod 这三者之间的关系见上图。通过 Deployment，我们可以管理多个 label 各不相同的 ReplicaSet，每个 ReplicaSet 负责保证对应数目的 Pod 在运行。

我们来看一个定义 Deployment 的例子：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment-demo
  namespace: demo
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
        version: v1
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

我们这里定义了副本数`spec.replicas`为 3，同时在`spec.selector.matchLabels`中设置了`app=nginx`的 label，用于匹配`spec.template.metadata.labels`的 label。我们还增加了`version=v1`的 label 用于后续滚动升级做比较。

我们将上述内容保存到`deploy-demo.yaml`文件中。

注意，`spec.selector.matchLabels`中写的 label 一定要能匹配得了`spec.template.metadata.labels`中的 label。否则该 Deployment 无法关联创建出来的 ReplicaSet。

然后我们通过下面这些命令，便可以在命名空间 demo 中创建名为 nginx-deployment-demo 的 Deployment。

```sh
$ kubectl create ns demo
$ kubectl create -f deploy-demo.yaml
deployment.apps/nginx-deployment-demo created
```

创建完成后，我们可以查看自动创建出来的 rs。

```sh
$ kubectl get rs -n demo
NAME                               DESIRED   CURRENT   READY   AGE
nginx-deployment-demo-5d65f98bd9   3         3         0       5s
```

接下来，我们可以通过如下的几个命令不断地查询系统，看看对应的 Pod 是否运行成功。

```sh
$ kubectl get pod -n demo -l app=nginx,version=v1
NAME                                     READY   STATUS              RESTARTS   AGE
nginx-deployment-demo-5d65f98bd9-7w5gp   0/1     ContainerCreating   0          30s
nginx-deployment-demo-5d65f98bd9-d78fx   0/1     ContainerCreating   0          30s
nginx-deployment-demo-5d65f98bd9-ssstk   0/1     ContainerCreating   0          30s

$ kubectl get pod -n demo -l app=nginx,version=v1
NAME                                     READY   STATUS    RESTARTS   AGE
nginx-deployment-demo-5d65f98bd9-7w5gp   1/1     Running   0          63s
nginx-deployment-demo-5d65f98bd9-d78fx   1/1     Running   0          63s
nginx-deployment-demo-5d65f98bd9-ssstk   1/1     Running   0          63s
```

我们可以看到，从 Deployment 到这里，最后的 Pod 已经创建成功了。

现在我们试着做下镜像更新看看。更改`spec.template.metadata.labels`中的`version=v1`为`version=v2`，同时更新镜像`nginx:1.14.2`为`nginx:1.19.2`。

你可以直接通过下述命令来直接更改：

```sh
$ kubectl edit deploy nginx-deployment-demo -n demo 
```

也可以更改`deploy-demo.yaml`这个文件：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment-demo
  namespace: demo
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
        version: v2
    spec:
      containers:
      - name: nginx
        image: nginx:1.19.2
        ports:
        - containerPort: 80
```

然后运行这些命令：

```sh
$ kubectl apply -f deploy-demo.yaml
Warning: kubectl apply should be used on resource created by either kubectl create --save-config or kubectl apply
deployment.apps/nginx-deployment-demo configured
```

这个时候，我们来看看 ReplicaSet 有什么变化：

```sh
$ kubectl get rs -n demo
NAME                               DESIRED   CURRENT   READY   AGE
nginx-deployment-demo-5d65f98bd9   3         3         3       4m10s
nginx-deployment-demo-7594578db7   1         1         0       3s
```

可以看到，这个时候自动创建了一个新的 rs 出来：`nginx-deployment-demo-7594578db7`。一般 Deployment 的默认更新策略是 RollingUpdate，即先创建一个新的 Pod，并待其成功运行以后，再删除旧的。

还有一个更新策略是Recreate，即先删除现有的 Pod，再创建新的。关于 Deployment，我们还可以设置最大不可用（`maxUnavailable`）和最大增量（`maxSurge`），来更精确地控制滚动更新操作，具体配置可参照[这个文档](https://kubernetes.io/zh/docs/concepts/workloads/controllers/deployment/#滚动更新-deployment)。

我建议你使用默认的策略来保证可用性。后面 Deployment 控制器会不断对这两个 rs 进行调和，直到新的 rs 副本数为 3，老的 rs 副本数为 0。

我们可以通过如下命令，能够观察到 Deployment 的调和过程。

```sh
$ kubectl get pod -n demo -l app=nginx,version=v1
NAME                                     READY   STATUS    RESTARTS   AGE
nginx-deployment-demo-5d65f98bd9-7w5gp   1/1     Running   0          4m15s
nginx-deployment-demo-5d65f98bd9-d78fx   1/1     Running   0          4m15s
nginx-deployment-demo-5d65f98bd9-ssstk   1/1     Running   0          4m15s

$ kubectl get pod -n demo -l app=nginx
NAME                                     READY   STATUS              RESTARTS   AGE
nginx-deployment-demo-5d65f98bd9-7w5gp   1/1     Running             0          4m22s
nginx-deployment-demo-5d65f98bd9-d78fx   1/1     Running             0          4m22s
nginx-deployment-demo-5d65f98bd9-ssstk   1/1     Running             0          4m22s
nginx-deployment-demo-7594578db7-zk8jq   0/1     ContainerCreating   0          15s

$ kubectl get pod -n demo -l app=nginx,version=v2
NAME                                     READY   STATUS              RESTARTS   AGE
nginx-deployment-demo-7594578db7-zk8jq   0/1     ContainerCreating   0          19s

$ kubectl get pod -n demo -l app=nginx,version=v2
NAME                                     READY   STATUS              RESTARTS   AGE
nginx-deployment-demo-7594578db7-4g4fk   0/1     ContainerCreating   0          1s
nginx-deployment-demo-7594578db7-zk8jq   1/1     Running             0          31s

$ kubectl get pod -n demo -l app=nginx,version=v1
NAME                                     READY   STATUS        RESTARTS   AGE
nginx-deployment-demo-5d65f98bd9-7w5gp   1/1     Running       0          4m40s
nginx-deployment-demo-5d65f98bd9-d78fx   1/1     Running       0          4m40s
nginx-deployment-demo-5d65f98bd9-ssstk   1/1     Terminating   0          4m40s

$ kubectl get pod -n demo -l app=nginx,version=v2
NAME                                     READY   STATUS              RESTARTS   AGE
nginx-deployment-demo-7594578db7-4g4fk   1/1     Running             0          5s
nginx-deployment-demo-7594578db7-ftzmf   0/1     ContainerCreating   0          2s
nginx-deployment-demo-7594578db7-zk8jq   1/1     Running             0          35s

$ kubectl get pod -n demo -l app=nginx,version=v1
NAME                                     READY   STATUS        RESTARTS   AGE
nginx-deployment-demo-5d65f98bd9-7w5gp   0/1     Terminating   0          4m52s
nginx-deployment-demo-5d65f98bd9-ssstk   0/1     Terminating   0          4m52s

$ kubectl get pod -n demo -l app=nginx,version=v2
NAME                                     READY   STATUS    RESTARTS   AGE
nginx-deployment-demo-7594578db7-4g4fk   1/1     Running   0          17s
nginx-deployment-demo-7594578db7-ftzmf   1/1     Running   0          14s
nginx-deployment-demo-7594578db7-zk8jq   1/1     Running   0          47s

$ kubectl get rs -n demo
NAME                               DESIRED   CURRENT   READY   AGE
nginx-deployment-demo-5d65f98bd9   0         0         0       5m5s
nginx-deployment-demo-7594578db7   3         3         3       58s

$ kubectl get pod -n demo -l app=nginx,version=v1
No resources found in demo namespace.
```

或者可以通过 kubectl 的 watch 功能：

```sh
$ kubectl get pod -n demo -l app=nginx -w
```

至此，我们完成了 Deployment 的升级过程。



### 写在最后

Kubernetes 中这些高阶的抽象对象，都是通过标签选择器来控制 Pod 的，包括我们下一节课要讲的有状态服务控制器。通过这些标签选择器，我们也可以通过 kubectl 命令行方便地查询一些对象。

有了 Deployment 这个高级对象，我们可以很方便地完成无状态服务的发布、更新升级，无须多余的人工参与，就能保证业务的高可用性。这也是 Kubernetes 迷人之处——声明式 API。



### 问题

用yaml创建deploy的时候也会同时创建rs吗

> 是的，这部分的 rs 是有 kube-controller-manager 自己来创建和管理的，我们不用直接操作。







## 07 | 有状态应用：Kubernetes 如何通过 StatefulSet 支持有状态应用？



Kubernetes 中的另外一种工作负载 StatefulSet。从名字就可以看出，这个工作负载主要用于有状态的服务发布。

我们从一个具体的例子来逐渐了解、认识 StatefulSet。在 kubectl 命令行中，我们一般将 StatefulSet 简写为 sts。在部署一个 StatefulSet 的时候，有个前置依赖对象，即 Service（服务）：

```yaml
$ cat nginx-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-demo
  namespace: demo
  labels:
    app: nginx
spec:
  clusterIP: None
  ports:
  - port: 80
    name: web
  selector:
    app: nginx
```

上面这段 yaml 的意思是，在 demo 这个命名空间中，创建一个名为 nginx-demo 的服务，这个服务暴露了 80 端口，可以访问带有`app=nginx`这个 label 的 Pod。

我们现在利用上面这段 yaml 在集群中创建出一个 Service：

```sh
$ kubectl create ns demo
$ kubectl create -f nginx-svc.yaml
service/nginx-demo created
$ kubectl get svc -n demo
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
nginx-demo   ClusterIP   None             <none>        80/TCP    5s
```

创建好了这个前置依赖的 Service，下面我们就可以开始创建真正的 StatefulSet 对象，可参照如下的 yaml 文件：

```yaml
$ cat web-sts.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web-demo
  namespace: demo
spec:
  serviceName: "nginx-demo"
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.19.2-alpine
        ports:
        - containerPort: 80
          name: web

$ kubectl create -f web-sts.yaml
$ kubectl get sts -n demo
NAME       READY   AGE
web-demo   0/2     9s
```

可以看到，到这里我已经将名为web-demo的StatefulSet部署完成了。

下面我们来一点点探索 StatefulSet 的秘密，看看它有哪些特性，为何可以保障服务有状态的运行。



### StatefulSet 的特性

通过 kubectl 的**watch**功能（命令行加参数`-w`），我们可以观察到 Pod 状态的一步步变化。

```sh
$ kubectl get pod -n demo -w
NAME         READY   STATUS              RESTARTS   AGE
web-demo-0   0/1     ContainerCreating   0          18s
web-demo-0   1/1     Running             0          20s
web-demo-1   0/1     Pending             0          0s
web-demo-1   0/1     Pending             0          0s
web-demo-1   0/1     ContainerCreating   0          0s
web-demo-1   1/1     Running             0          2s
```

**通过 StatefulSet 创建出来的 Pod 名字有一定的规律**，即`$(statefulset名称)-$(序号)`，比如这个例子中的web-demo-0、web-demo-1。

这里面还有个有意思的点，web-demo-0 这个 Pod 比 web-demo-1 优先创建，而且在 web-demo-0 变为 Running 状态以后，才被创建出来。为了证实这个猜想，我们在一个终端窗口观察 StatefulSet 的 Pod：

```sh
$ kubectl get pod -n demo -w -l app=nginx
```

我们再开一个终端端口来 watch 这个 namespace 中的 event：

```sh
$ kubectl get event -n demo -w
```

现在我们试着改变这个 StatefulSet 的副本数，将它改成 5：

```sh
$ kubectl scale sts web-demo -n demo --replicas=5
statefulset.apps/web-demo scaled
```

此时我们观察到另外两个终端端口的输出：

```sh
$ kubectl get pod -n demo -w
NAME         READY   STATUS    RESTARTS   AGE
web-demo-0   1/1     Running   0          20m
web-demo-1   1/1     Running   0          20m
web-demo-2   0/1     Pending   0          0s
web-demo-2   0/1     Pending   0          0s
web-demo-2   0/1     ContainerCreating   0          0s
web-demo-2   1/1     Running             0          2s
web-demo-3   0/1     Pending             0          0s
web-demo-3   0/1     Pending             0          0s
web-demo-3   0/1     ContainerCreating   0          0s
web-demo-3   1/1     Running             0          3s
web-demo-4   0/1     Pending             0          0s
web-demo-4   0/1     Pending             0          0s
web-demo-4   0/1     ContainerCreating   0          0s
web-demo-4   1/1     Running             0          3s
```

我们再一次看到了 StatefulSet 管理的 Pod 按照 2、3、4 的顺序依次创建，名称有规律，跟上一节通过 Deployment 创建的随机 Pod 名有很大的区别。

通过观察对应的 event 信息，也可以再次证实我们的猜想。

```sh
$ kubectl get event -n demo -w
LAST SEEN   TYPE     REASON             OBJECT                 MESSAGE
20m         Normal   Scheduled          pod/web-demo-0         Successfully assigned demo/web-demo-0 to kraken
20m         Normal   Pulling            pod/web-demo-0         Pulling image "nginx:1.19.2-alpine"
20m         Normal   Pulled             pod/web-demo-0         Successfully pulled image "nginx:1.19.2-alpine"
20m         Normal   Created            pod/web-demo-0         Created container nginx
20m         Normal   Started            pod/web-demo-0         Started container nginx
20m         Normal   Scheduled          pod/web-demo-1         Successfully assigned demo/web-demo-1 to kraken
20m         Normal   Pulled             pod/web-demo-1         Container image "nginx:1.19.2-alpine" already present on machine
20m         Normal   Created            pod/web-demo-1         Created container nginx
20m         Normal   Started            pod/web-demo-1         Started container nginx
20m         Normal   SuccessfulCreate   statefulset/web-demo   create Pod web-demo-0 in StatefulSet web-demo successful
20m         Normal   SuccessfulCreate   statefulset/web-demo   create Pod web-demo-1 in StatefulSet web-demo successful
0s          Normal   SuccessfulCreate   statefulset/web-demo   create Pod web-demo-2 in StatefulSet web-demo successful
0s          Normal   Scheduled          pod/web-demo-2         Successfully assigned demo/web-demo-2 to kraken
0s          Normal   Pulled             pod/web-demo-2         Container image "nginx:1.19.2-alpine" already present on machine
0s          Normal   Created            pod/web-demo-2         Created container nginx
0s          Normal   Started            pod/web-demo-2         Started container nginx
0s          Normal   SuccessfulCreate   statefulset/web-demo   create Pod web-demo-3 in StatefulSet web-demo successful
0s          Normal   Scheduled          pod/web-demo-3         Successfully assigned demo/web-demo-3 to kraken
0s          Normal   Pulled             pod/web-demo-3         Container image "nginx:1.19.2-alpine" already present on machine
0s          Normal   Created            pod/web-demo-3         Created container nginx
0s          Normal   Started            pod/web-demo-3         Started container nginx
0s          Normal   SuccessfulCreate   statefulset/web-demo   create Pod web-demo-4 in StatefulSet web-demo successful
0s          Normal   Scheduled          pod/web-demo-4         Successfully assigned demo/web-demo-4 to kraken
0s          Normal   Pulled             pod/web-demo-4         Container image "nginx:1.19.2-alpine" already present on machine
0s          Normal   Created            pod/web-demo-4         Created container nginx
0s          Normal   Started            pod/web-demo-4         Started container nginx
```

现在我们试着进行一次缩容：

```sh
$ kubectl scale sts web-demo -n demo --replicas=2
statefulset.apps/web-demo scaled
```

此时观察另外两个终端窗口，分别如下：

```sh
web-demo-4   1/1     Terminating   0          11m
web-demo-4   0/1     Terminating   0          11m
web-demo-4   0/1     Terminating   0          11m
web-demo-4   0/1     Terminating   0          11m
web-demo-3   1/1     Terminating   0          12m
web-demo-3   0/1     Terminating   0          12m
web-demo-3   0/1     Terminating   0          12m
web-demo-3   0/1     Terminating   0          12m
web-demo-2   1/1     Terminating   0          12m
web-demo-2   0/1     Terminating   0          12m
web-demo-2   0/1     Terminating   0          12m
web-demo-2   0/1     Terminating   0          12m
```

```sh
0s          Normal   SuccessfulDelete   statefulset/web-demo   delete Pod web-demo-4 in StatefulSet web-demo successful
0s          Normal   Killing            pod/web-demo-4         Stopping container nginx
0s          Normal   Killing            pod/web-demo-3         Stopping container nginx
0s          Normal   SuccessfulDelete   statefulset/web-demo   delete Pod web-demo-3 in StatefulSet web-demo successful
0s          Normal   SuccessfulDelete   statefulset/web-demo   delete Pod web-demo-2 in StatefulSet web-demo successful
0s          Normal   Killing            pod/web-demo-2         Stopping container nginx
```

可以看到，在缩容的时候，StatefulSet 关联的 Pod 按着 4、3、2 的顺序依次删除。

可见，**对于一个拥有 N 个副本的 StatefulSet 来说**，**Pod 在部署时按照 {0 …… N-1} 的序号顺序创建的**，**而删除的时候按照逆序逐个删除**，这便是我想说的第一个特性。

接着我们来看，**StatefulSet 创建出来的 Pod 都具有固定的、且确切的主机名**，比如：

```sh
$ for i in 0 1; do kubectl exec web-demo-$i -n demo -- sh -c 'hostname'; done
web-demo-0
web-demo-1
```

我们再看看上面 StatefulSet 的 API 对象定义，有没有发现跟我们上一节中 Deployment 的定义极其相似，主要的差异在于`spec.serviceName`这个字段。它很重要，StatefulSet 根据这个字段，为每个 Pod 创建一个 DNS 域名，这个**域名的格式**为`$(podname).(headless service name)`，下面我们通过例子来看一下。

当前 Pod 和 IP 之间的对应关系如下：

```sh
$ kubectl get pod -n demo -l app=nginx -o wide
NAME         READY   STATUS    RESTARTS   AGE     IP            NODE     NOMINATED NODE   READINESS GATES
web-demo-0   1/1     Running   0          3h17m   10.244.0.39   kraken   <none>           <none>
web-demo-1   1/1     Running   0          3h17m   10.244.0.40   kraken   <none>           <none>
```

Pod web-demo-0 的IP 地址是 10.244.0.39，web-demo-1的 IP 地址是 10.244.0.40。这里我们通过`kubectl run`在同一个命名空间`demo`中创建一个名为 dns-test 的 Pod，同时 attach 到容器中，类似于`docker run -it --rm`这个命令。
我么在容器中运行 nslookup 来查询它们在集群内部的 DNS 地址，如下所示：

```sh
$ kubectl run -it --rm --image busybox:1.28 dns-test -n demo
If you don't see a command prompt, try pressing enter.
/ # nslookup web-demo-0.nginx-demo
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
Name:      web-demo-0.nginx-demo
Address 1: 10.244.0.39 web-demo-0.nginx-demo.demo.svc.cluster.local
/ # nslookup web-demo-1.nginx-demo
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
Name:      web-demo-1.nginx-demo
Address 1: 10.244.0.40 web-demo-1.nginx-demo.demo.svc.cluster.local
```

可以看到，每个 Pod 都有一个对应的 [A 记录](https://baike.baidu.com/item/A记录/1188077?fr=aladdin)。
我们现在删除一下这些 Pod，看看会有什么变化：

```sh
$ kubectl delete pod -l app=nginx -n demo
pod "web-demo-0" deleted
pod "web-demo-1" deleted
$ kubectl get pod -l app=nginx -n demo -o wide
NAME         READY   STATUS    RESTARTS   AGE   IP            NODE     NOMINATED NODE   READINESS GATES
web-demo-0   1/1     Running   0          15s   10.244.0.50   kraken   <none>           <none>
web-demo-1   1/1     Running   0          13s   10.244.0.51   kraken   <none>           <none>
```

删除成功后，可以发现 StatefulSet 立即生成了新的 Pod，但是 Pod 名称维持不变。唯一变化的就是 IP 发生了改变。

我们再来看看 DNS 记录：

```sh
$ kubectl run -it --rm --image busybox:1.28 dns-test -n demo
If you don't see a command prompt, try pressing enter.
/ # nslookup web-demo-0.nginx-demo
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
Name:      web-demo-0.nginx-demo
Address 1: 10.244.0.50 web-demo-0.nginx-demo.demo.svc.cluster.local
/ # nslookup web-demo-1.nginx-demo
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
Name:      web-demo-1.nginx-demo
Address 1: 10.244.0.51 web-demo-1.nginx-demo.demo.svc.cluster.local
```

可以看出，**DNS 记录中 Pod 的域名没有发生变化**，**仅仅 IP 地址发生了更换**。因此当 Pod 所在的节点发生故障导致 Pod 飘移到其他节点上，或者 Pod 因故障被删除重建，Pod 的 IP 都会发生变化，但是 Pod 的域名不会有任何变化，这也就意味着**服务间可以通过不变的 Pod 域名来保障通信稳定**，**而不必依赖 Pod IP**。

有了`spec.serviceName`这个字段，保证了 StatefulSet 关联的 Pod 可以有稳定的网络身份标识，即 Pod 的序号、主机名、DNS 记录名称等。

最后一个我想说的是，对于有状态的服务来说，每个副本都会用到持久化存储，且各自使用的数据是不一样的。

StatefulSet 通过 PersistentVolumeClaim（PVC）可以保证 Pod 的存储卷之间一一对应的绑定关系。同时，删除 StatefulSet 关联的 Pod 时，不会删除其关联的 PVC。



### 如何更新升级 StatefulSet

那么，如果想对一个 StatefulSet 进行升级，该怎么办呢？

在 StatefulSet 中，支持两种更新升级策略，即 RollingUpdate 和 OnDelete。

RollingUpdate策略是**默认的更新策略**。可以实现 Pod 的滚动升级，跟我们上一节课中 Deployment 介绍的RollingUpdate策略一样。比如我们这个时候做了镜像更新操作，那么整个的升级过程大致如下，先逆序删除所有的 Pod，然后依次用新镜像创建新的 Pod 出来。这里你可以通过`kubectl get pod -n demo -w -l app=nginx`来动手观察下。

同时使用 RollingUpdate 更新策略还支持通过 partition 参数来分段更新一个 StatefulSet。所有序号大于或者等于 partition 的Pod 都将被更新。你这里也可以手动更新 StatefulSet 的配置来实验下。

当你把更新策略设置为 OnDelete 时，我们就必须手动先删除 Pod，才能触发新的 Pod 更新。



### 写在最后

现在我们就总结下 StatefulSet 的特点：

- 具备固定的网络标记，比如主机名，域名等；
- 支持持久化存储，而且最好能够跟实例一一绑定；
- 可以按照顺序来部署和扩展；
- 可以按照顺序进行终止和删除操作；
- 在进行滚动升级的时候，也会按照一定顺序。

借助 StatefulSet 的这些能力，我们就可以去部署一些有状态服务，比如 MySQL、ZooKeeper、MongoDB 等。你可以跟着这个[教程](https://kubernetes.io/zh/docs/tutorials/stateful-application/zookeeper/)在 Kubernetes 中搭建一个 ZooKeeper 集群。





## 08 | 配置管理：Kubernetes 管理业务配置方式有哪些？



在使用过程中，我们常常需要对 Pod 进行一些配置管理，比如参数配置文件怎么使用，敏感数据怎么保存传递，等等。有些人可能会觉得，为什么不把这些配置（不限于参数、配置文件、密钥等）打包到镜像中去啊？乍一听，好像有点可行，但是这种做法“硬伤”太多。

- 有些不变的配置是可以打包到镜像中的，那可变的配置呢？
- 信息泄漏，很容易引发安全风险，尤其是一些敏感信息，比如密码、密钥等。
- 每次配置更新后，都要重新打包一次，升级应用。镜像版本过多，也给镜像管理和镜像中心存储带来很大的负担。
- 定制化太严重，可扩展能力差，且不容易复用。

所以这里的一个最佳实践就是将配置信息和容器镜像进行解耦，以“不变应万变”。在 Kubernetes 中，一般有 ConfigMap 和 Secret 两种对象，可以用来做配置管理。



### ConfigMap

首先我们来讲一下 ConfigMap 这个对象，它主要用来保存一些非敏感数据，可以用作环境变量、命令行参数或者挂载到存储卷中。

![image (2).png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F9gkFGAAFxaAAB80T5b-WU361.png)

ConfigMap 通过键值对来存储信息，是个 namespace 级别的资源。在 kubectl 使用时，我们可以简写成 cm。

我们来看一下两个 ConfigMap 的 API 定义：

```yaml
$ cat cm-demo-mix.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cm-demo-mix # 对象名字
  namespace: demo # 所在的命名空间
data: # 这是跟其他对象不太一样的地方，其他对象这里都是spec
  # 每一个键都映射到一个简单的值
  player_initial_lives: "3" # 注意这里的值如果数字的话，必须用字符串来表示
  ui_properties_file_name: "user-interface.properties"
  # 也可以来保存多行的文本
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true

$ cat cm-demo-all-env.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cm-demo-all-env
  namespace: demo
data:
  SPECIAL_LEVEL: very
  SPECIAL_TYPE: charm
```

可见，我们通过 ConfigMap 既可以存储简单的键值对，也能存储多行的文本。

现在我们来创建这两个 ConfigMap：

```sh
$ kubectl create -f cm-demo-mix.yaml
configmap/cm-demo-mix created
$ kubectl create -f cm-demo-all-env.yaml
configmap/cm-demo-all-env created
```

创建 ConfigMap，你也可以通过`kubectl create cm`基于[目录](https://kubernetes.io/zh/docs/tasks/configure-pod-container/configure-pod-configmap/#create-configmaps-from-directories)、[文件](https://kubernetes.io/zh/docs/tasks/configure-pod-container/configure-pod-configmap/#create-configmaps-from-files)或者[字面值](https://kubernetes.io/zh/docs/tasks/configure-pod-container/configure-pod-configmap/#create-configmaps-from-literal-values)来创建，详细可参考这个[官方文档](https://kubernetes.io/zh/docs/tasks/configure-pod-container/configure-pod-configmap/#使用-kubectl-create-configmap-创建-configmap)。

创建成功后，我们可以通过如下方式来查看创建出来的对象。

```sh
$ kubectl get cm -n demo
NAME              DATA   AGE
cm-demo-all-env   2      30s
cm-demo-mix       4      2s

$ kubectl describe cm cm-demo-all-env -n demo
Name:         cm-demo-all-env
Namespace:    demo
Labels:       <none>
Annotations:  <none>
Data
====
SPECIAL_LEVEL:
----
very
SPECIAL_TYPE:
----
charm
Events:  <none>

$ kubectl describe cm cm-demo-mix -n demo
Name:         cm-demo-mix
Namespace:    demo
Labels:       <none>
Annotations:  <none>
Data
====
user-interface.properties:
----
color.good=purple
color.bad=yellow
allow.textmode=true
game.properties:
----
enemy.types=aliens,monsters
player.maximum-lives=5
player_initial_lives:
----
3
ui_properties_file_name:
----
user-interface.properties
Events:  <none>
```

下面我们看看怎么和 Pod 结合起来使用。在使用的时候，有几个地方需要特别注意：

- **Pod 必须和 ConfigMap 在同一个 namespace 下面；**
- **在创建 Pod 之前，请务必保证 ConfigMap 已经存在，否则 Pod 创建会报错。**

```yaml
$ cat cm-demo-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: cm-demo-pod
  namespace: demo
spec:
  containers:
    - name: demo
      image: busybox:1.28
      command:
        - "bin/sh"
        - "-c"
        - "echo PLAYER_INITIAL_LIVES=$PLAYER_INITIAL_LIVES && sleep 10000"
      env:
        # 定义环境变量
        - name: PLAYER_INITIAL_LIVES # 请注意这里和 ConfigMap 中的键名是不一样的
          valueFrom:
            configMapKeyRef:
              name: cm-demo-mix         # 这个值来自 ConfigMap
              key: player_initial_lives # 需要取值的键
        - name: UI_PROPERTIES_FILE_NAME
          valueFrom:
            configMapKeyRef:
              name: cm-demo-mix
              key: ui_properties_file_name
      envFrom:  # 可以将 configmap 中的所有键值对都通过环境变量注入容器中
        - configMapRef:
            name: cm-demo-all-env
      volumeMounts:
      - name: full-config # 这里是下面定义的 volume 名字
        mountPath: "/config" # 挂载的目标路径
        readOnly: true
      - name: part-config
        mountPath: /etc/game/
        readOnly: true
  volumes: # 您可以在 Pod 级别设置卷，然后将其挂载到 Pod 内的容器中
    - name: full-config # 这是 volume 的名字
      configMap:
        name: cm-demo-mix # 提供你想要挂载的 ConfigMap 的名字
    - name: part-config
      configMap:
        name: cm-demo-mix
        items: # 我们也可以只挂载部分的配置
        - key: game.properties
          path: properties
```

在上面的这个例子中，几乎囊括了 ConfigMap 的几大使用场景：

- 命令行参数；
- 环境变量，可以只注入部分变量，也可以全部注入；
- 挂载文件，可以是单个文件，也可以是所有键值对，用每个键值作为文件名。

我们接着来创建：

```sh
$ kubectl create -f cm-demo-pod.yaml
pod/cm-demo-pod created
```

创建成功后，我们 exec 到容器中看看：

```sh
$ kubectl exec -it cm-demo-pod -n demo sh
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl kubectl exec [POD] -- [COMMAND] instead.
/ # env
KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT=tcp://10.96.0.1:443
UI_PROPERTIES_FILE_NAME=user-interface.properties
HOSTNAME=cm-demo-pod
SHLVL=1
HOME=/root
SPECIAL_LEVEL=very
TERM=xterm
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
PLAYER_INITIAL_LIVES=3
KUBERNETES_SERVICE_HOST=10.96.0.1
PWD=/
SPECIAL_TYPE=charm
/ # ls /config/
game.properties            ui_properties_file_name
player_initial_lives       user-interface.properties
/ # ls -alh /config/
total 12
drwxrwxrwx    3 root     root        4.0K Aug 27 09:54 .
drwxr-xr-x    1 root     root        4.0K Aug 27 09:54 ..
drwxr-xr-x    2 root     root        4.0K Aug 27 09:54 ..2020_08_27_09_54_31.007551221
lrwxrwxrwx    1 root     root          31 Aug 27 09:54 ..data -> ..2020_08_27_09_54_31.007551221
lrwxrwxrwx    1 root     root          22 Aug 27 09:54 game.properties -> ..data/game.properties
lrwxrwxrwx    1 root     root          27 Aug 27 09:54 player_initial_lives -> ..data/player_initial_lives
lrwxrwxrwx    1 root     root          30 Aug 27 09:54 ui_properties_file_name -> ..data/ui_properties_file_name
lrwxrwxrwx    1 root     root          32 Aug 27 09:54 user-interface.properties -> ..data/user-interface.properties
/ # cat /config/game.properties
enemy.types=aliens,monsters
player.maximum-lives=5
/ # cat /etc/game/properties
enemy.types=aliens,monsters
player.maximum-lives=5
```

可以看到，环境变量都已经正确注入，对应的文件和目录也都挂载进来了。
在上面`ls -alh /config/`后，我们看到挂载的文件中存在软链接，都指向了`..data`目录下的文件。这样做的好处，是 kubelet 会定期同步检查已经挂载的 ConfigMap 是否是最新的，如果更新了，就是创建一个新的文件夹存放最新的内容，并同步修改`..data`指向的软链接。

一般我们只把一些非敏感的数据保存到 ConfigMap 中，敏感的数据就要保存到 Secret 中了。



### Secret

我们可以用 Secret 来保存一些敏感的数据信息，比如密码、密钥、token 等。在使用的时候， 跟 ConfigMap 的用法基本保持一致，都可以用来作为环境变量或者文件挂载。

Kubernetes 自身也有一些内置的 Secret，主要用来保存访问 APIServer 的 service account token。

除此之外，还可以用来保存私有镜像中心的身份信息，这样 kubelet 可以拉取到镜像。

> 注： 如果你使用的是 Docker，也可以提前在目标机器上运行`docker login yourprivateregistry.com`来保存你的有效登录信息。Docker 一般会将私有仓库的密钥保存在`$HOME/.docker/config.json`文件中，将该文件分发到所有节点即可。

我们看看如何通过 kubectl 来创建 secret，通过命令行 help 可以看到 kubectl 能够创建多种类型的 Secret。

```sh
$ kubectl create secret  -h
 Create a secret using specified subcommand.
Available Commands:
   docker-registry Create a secret for use with a Docker registry
   generic         Create a secret from a local file, directory or literal value
   tls             Create a TLS secret
Usage:
   kubectl create secret [flags] [options]
Use "kubectl  --help" for more information about a given command.
 Use "kubectl options" for a list of global command-line options (applies to all commands).
```

我们先来创建一个 Secret 来保存访问私有容器仓库的身份信息：

```sh
$ kubectl create secret -n demo docker-registry regcred \
   --docker-server=yourprivateregistry.com \
   --docker-username=allen \
   --docker-password=mypassw0rd \
   --docker-email=allen@example.com
 secret/regcred created
 
 $ kubectl get secret -n demo regcred
 NAME      TYPE                             DATA   AGE
 regcred   kubernetes.io/dockerconfigjson   1      28s
```

这里我们可以看到，创建出来的 Secret 类型是`kubernetes.io/dockerconfigjson`：

```sh
$ kubectl describe secret -n demo regcred
Name:         regcred
Namespace:    demo
Labels:       <none>
Annotations:  <none>
Type:  kubernetes.io/dockerconfigjson
Data
====
.dockerconfigjson:  144 bytes
```

为了防止 Secret 中的内容被泄漏，`kubectl get`和`kubectl describe`会避免直接显示密码的内容。但是我们可以通过拿到完整的 Secret 对象来进一步查看其数据：

```yaml
$ kubectl get secret -n demo regcred -o yaml
apiVersion: v1
data: # 跟 configmap 一样，这块用于保存数据信息
  .dockerconfigjson: eyJhdXRocyI6eyJ5b3VycHJpdmF0ZXJlZ2lzdHJ5LmNvbSI6eyJ1c2VybmFtZSI6ImFsbGVuIiwicGFzc3dvcmQiOiJteXBhc3N3MHJkIiwiZW1haWwiOiJhbGxlbkBleGFtcGxlLmNvbSIsImF1dGgiOiJZV3hzWlc0NmJYbHdZWE56ZHpCeVpBPT0ifX19
kind: Secret
metadata:
  creationTimestamp: "2020-08-27T12:18:35Z"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:data:
        .: {}
        f:.dockerconfigjson: {}
      f:type: {}
    manager: kubectl
    operation: Update
    time: "2020-08-27T12:18:35Z"
  name: regcred
  namespace: demo
  resourceVersion: "1419452"
  selfLink: /api/v1/namespaces/demo/secrets/regcred
  uid: 6d34123e-4d79-406b-9556-409cfb4db2e7
type: kubernetes.io/dockerconfigjson
```

这里我们发现`.dockerconfigjson`是一段乱码，我们用 base64 解压试试看：

至此，我们发现 Secret 和 ConfigMap 在数据保存上的最大不同。**Secret 保存的数据都是通过 base64 加密后的数据**。

我们平时使用较为广泛的还有另外一种`Opaque`类型的 Secret：ku

```yaml
$ cat secret-demo.yaml
apiVersion: v1
kind: Secret
metadata:
  name: dev-db-secret
  namespace: demo
type: Opaque
data: # 这里的值都是 base64 加密后的
  password: UyFCXCpkJHpEc2I9
  username: ZGV2dXNlcg==
```

或者我们也可以通过如下等价的 kubectl 命令来创建出来：

```sh
$ kubectl create secret generic dev-db-secret -n demo \
  --from-literal=username=devuser \
  --from-literal=password='S!B\*d$zDsb='
```

或通过文件来创建对象，比如：

```sh
$ echo -n 'username=devuser' > ./db_secret.txt
$ echo -n 'password=S!B\*d$zDsb=' >> ./db_secret.txt
$ kubectl create secret generic dev-db-secret -n demo \
  --from-file=./db_secret.txt
```

有时候为了方便，你也可以使用`stringData`，这样可以避免自己事先手动用 base64 进行加密。

```yaml
$ cat secret-demo-stringdata.yaml
apiVersion: v1
kind: Secret
metadata:
  name: dev-db-secret
  namespace: demo
type: Opaque
stringData:
  password: devuser
  username: S!B\*d$zDsb=
```

下面我们在 Pod 中使用 Secret：

```yaml
$ cat pod-secret.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-test-pod
  namespace: demo
spec:
  containers:
    - name: demo-container
      image: busybox:1.28
      command: [ "/bin/sh", "-c", "env" ]
      envFrom:
      - secretRef:
          name: dev-db-secret
  restartPolicy: Never

$ kubectl create -f pod-secret.yaml
pod/secret-test-pod created
```

创建成功后，我们来查看下：

```sh
$ kubectl get pod -n demo secret-test-pod
NAME              READY   STATUS      RESTARTS   AGE
secret-test-pod   0/1     Completed   0          14s

$ kubectl logs -f -n demo secret-test-pod
KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT=tcp://10.96.0.1:443
HOSTNAME=secret-test-pod
SHLVL=1
username=devuser
HOME=/root
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
KUBERNETES_PORT_443_TCP_PORT=443
password=S!B\*d$zDsb=
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_SERVICE_HOST=10.96.0.1
PWD=/
```

我们可以在日志中看到命令`env`的输出，看到环境变量`username`和`password`已经正确注入。类似地，我们也可以将 Secret 作为 Volume 挂载到 Pod 内。



### 写在最后

ConfigMap 和 Secret 是 Kubernetes 常用的保存配置数据的对象，你可以根据需要选择合适的对象存储数据。通过 Volume 方式挂载到 Pod 内的，kubelet 都会定期进行更新。但是通过环境变量注入到容器中，就无法感知到 ConfigMap 或 Secret 的内容更新。

目前如何让 Pod 内的业务感知到 ConfigMap 或 Secret 的变化，还是一个待解决的问题。但是我们还是有一些 Workaround 的。

- 如果业务自身支持 reload 配置的话，比如`nginx -s reload`，可以通过 inotify 感知到文件更新，或者直接定期进行 reload（这里可以配合我们的 readinessProbe 一起使用）。
- 如果我们的业务没有这个能力，考虑到不可变基础设施的思想，我们是不是可以采用滚动升级的方式进行？没错，这是一个非常好的方法。目前有个开源工具[Reloader](https://github.com/stakater/Reloader)，它就是采用这种方式，通过 watch ConfigMap 和 Secret，一旦发现对象更新，就自动触发对 Deployment 或 StatefulSet 等工作负载对象进行滚动升级。具体使用方法，可以参考项目的文档说明。

对于这个问题，社区其实也一直在讨论比较好的解法，我们可以拭目以待。





## 09 | 存储类型：如何挑选合适的存储插件？



在以前玩虚拟机的时代，大家比较少考虑存储的问题，因为在通过底层 IaaS 平台申请虚拟机的时候，大多数情况下，我们都会事先预估好需要的容量，方便虚拟机起来后可以稳定的使用这些存储资源。

但是容器与生俱来就是按照可以“运行在任何地方”（run anywhere）这一想法来设计的，对外部存储有着天然的诉求和依赖，并且由于容器本身的生命周期很短暂，在容器内保存数据是件很危险的事情，所以 Docker 通过挂载 Volume 来解决这一问题，如下图所示。

![Drawing 0.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F9pyf-ATyKeAAA4X7loNvA990.png)

一般来说，这些 Volume 都是和容器的生命周期进行绑定的。当然也可以单独创建，然后按需挂载到容器中。大家有兴趣可以查看目前 [Docker 都适配了哪些 volume plugins](https://docs.docker.com/engine/extend/legacy_plugins/#volume-plugins)（卷插件）。

现在，我们先来看看 Kubernetes 中的 Volume 跟 Docker 中的设计有什么不同。



### Kubernetes 中的 Volume 是如何设计的？

Kubernetes 中的 Volume 在设计上，跟 Docker 略有不同。

我们都知道在Kubernetes 中，Pod 里包含了一组容器，这些容器是可以共享存储的，如下图所示。同时 Pod 内的容器又受制于各自的重启策略，我们需要保证容器重启不会对这些存储产生影响。因此 Kubernetes 中 Volume 的生命周期是直接和 Pod 挂钩的，而不是 Pod 内的某个容器，即 Pod 在 Volume 在。在 Pod 被删除时，才会对 Volume 进行解绑（unmount）、删除等操作。至于 Volume 中的数据是否会被删除，取决于Volume 的具体类型。

![Drawing 1.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl9pyhuAVPi9AAGYrUBxa7Y940.png)

为了丰富可以对接的存储后端，Kubernetes 中提供了很多volume plugin可供使用。我将目前的一些 plugins 做了如下的分类，方便你进行初步的了解和比较。

![Drawing 2.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F9pyiKAUanvAACt-6jm3jw792.png)

如下图所示，Kubelet 内部调用相应的 plugin 实现，将外部的存储挂载到 Pod 内。类似于CephFS、NFS以及 awsEBS 这一类插件，是需要管理员提前在对应的存储系统中申请好的，Kubernetes 本身其实并不负责这些Volume 的申请。

![Drawing 3.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl9pyiyAavysAAEPwHj90U4290.png)



### 常见的几种内置 Volume 插件

这里我介绍几个日常工作和生产环境中经常使用到的几个插件。

#### ConfigMap 和 Secret

首先来看 ConfigMap 和 Secret，这两类对象都可以通过 Volume 形式挂载到 Pod 内。



#### Downward API

再来看看DownwardAPI，这是个非常有用的插件，可以帮助你获取 Pod 对象中定义的字段，比如 Pod 的标签（Labels）、Pod 的 IP 地址及 Pod 所在的命名空间（namespace）等。Downward API 有两种使用方法，既支持环境变量注入，也支持通过 Volume 挂载。

我们来看个 Volume 挂载的例子，如下是一个 Pod 的 yaml 文件：

```yaml
$ cat downwardapi-volume-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  name: downwardapi-volume-demo
  namespace: demo
  labels:
    zone: us-east-coast
    cluster: downward-api-test-cluster1
    rack: rack-123
  annotations:
    annotation1: "345"
    annotation2: "456"
spec:
  containers:
    - name: volume-test-container
      image: busybox:1.28
      command: ["sh", "-c"]
      args:
      - while true; do
          if [[ -e /etc/podinfo/labels ]]; then
            echo -en '\n\n'; cat /etc/podinfo/labels; fi;
          if [[ -e /etc/podinfo/annotations ]]; then
            echo -en '\n\n'; cat /etc/podinfo/annotations; fi;
          sleep 5;
        done;
      volumeMounts:
        - name: podinfo
          mountPath: /etc/podinfo
  volumes:
    - name: podinfo
      downwardAPI:
        items:
          - path: "labels"
            fieldRef:
              fieldPath: metadata.labels
          - path: "annotations"
            fieldRef:
              fieldPath: metadata.annotations
```

我们先创建这个 Pod，并通过`kubectl logs`来查看它的输出日志：

```sh
$ kubectl create -f downwardapi-volume-demo.yaml
pod/downwardapi-volume-demo created

$ kubectl get pod -n demo
NAME                      READY   STATUS    RESTARTS   AGE
downwardapi-volume-demo   1/1     Running   0          5s

$ kubectl logs -n demo -f downwardapi-volume-demo
cluster="downward-api-test-cluster1"
rack="rack-123"
zone="us-east-coast"
annotation1="345"
annotation2="456"
kubernetes.io/config.seen="2020-09-03T12:01:58.1728583Z"
kubernetes.io/config.source="api"
cluster="downward-api-test-cluster1"
rack="rack-123"
zone="us-east-coast"
annotation1="345"
annotation2="456"
kubernetes.io/config.seen="2020-09-03T12:01:58.1728583Z"
kubernetes.io/config.source="api"
```

从上面的日志输出，我们可以看到 Downward API 可以通过 Volume 挂载到 Pod 里面，并被容器获取。



#### EmptyDir

在 Kubernetes 中，我们也可以使用临时存储，类似于创建一个 temp dir。我们将这种类型的插件叫作 EmptyDir，从名字就可以知道，在刚开始创建的时候，就是空的临时文件夹。在 Pod 被删除后，也一同被删除，所以并不适合保存关键数据。

在使用的时候，可以参照如下的方式使用 EmptyDir：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: empty-dir-vol-demo
  namespace: demo
spec:
  containers:
  - image: busybox:1.28
    name: volume-test-container
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir: {}
```

一般来说，EmptyDir 可以用来做一些临时存储，比如为耗时较长的计算任务存储中间结果或者作为共享卷为同一个 Pod 内的容器提供数据等等。

除此之外，我们也可以将`emptyDir.medium`字段设置为“Memory”，来挂载 tmpfs （一种基于内存的文件系统）类型的 EmptyDir。比如下面这个例子:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: empty-dir-vol-memory-demo
  namespace: demo
spec:
  containers:
  - image: busybox:1.28
    imagePullPolicy: IfNotPresent
    name: myvolumes-container
    command: ['sh', '-c', 'echo container is Running ; df -h ; sleep 3600']
    volumeMounts:
    - mountPath: /demo
      name: demo-volume
  volumes:
  - name: demo-volume
    emptyDir:
      medium: Memory   
```



#### HostPath

我们再来看 HostPath，它和 EmptyDir 一样，都是利用宿主机的存储为容器分配资源。但是两者有个很大的区别，就是 HostPath 中的数据并不会随着 Pod 被删除而删除，而是会持久地存放在该节点上。

使用 HostPath 非常方便，既不需要依赖外部的存储系统，也不需要复杂的配置，还能持续存储数据。但是这里我要提醒你避免滥用：

1. 避免通过容器恶意修改宿主机上的文件内容；
2. 避免容器恶意占用宿主机上的存储资源而打爆宿主机；
3. 要考虑到 Pod 自身的声明周期，而且 Pod 是会“漂移”重新“长”到别的节点上的，所以要避免过度依赖本地的存储。

同时使用的时候也需要额外注意，因为 Hostpath 中定义的路径是宿主机上真实的绝对路径，那么就会存在同一节点上的多个 Pod 共用一个 Hostpath 的情形，比如同一工作负载的不同实例调度到同一节点上，这会造成数据混乱，读写异常。这个时候我们就需要额外设置一些调度策略，避免这种情况发生。

下面是一个使用 HostPath 的例子：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-demo
  namespace: demo
spec:
  containers:
  - image: nginx:1.19.2
    name: container-demo
    volumeMounts:
      - mountPath: /test-pd
        name: hostpath-volume
  volumes:
  - name: hostpath-volume 
    hostPath:
      path: /data  # 对应宿主机上的绝对路径
      type: Directory # 可选字段，默认是 Directory
```

在上面的例子中，我们要注意`hostpath.type`这个可以缺省的字段。为了保证后向兼容性，默认值是 Directory。目前这个字段还支持 DirectoryOrCreate、FileOrCreate 、File 、Socket 、CharDevice 和 BlockDevice，你可以到[官方文档](https://kubernetes.io/zh/docs/concepts/storage/volumes/#hostpath)中去了解这几个类型的具体含义。

这个 type 可以帮助你做一些预检查，比如你期望挂载的是单个文件，如果检测到挂载路径是个目录，这个时候就会报异常，这样可以有效地避免一些误配置。

上述介绍的这几款插件，目前依然能够照常使用，也是社区自身稳定支持的插件。但是对于一些云厂商和第三方的插件，社区已经不推荐继续使用内置的方式了，而是推荐你通过 CSI（Container Storage Interface，容器存储接口）来使用这些插件。



### 为什么社区要采用 CSI

一开始，上述这些云厂商以及第三方的卷插件（volume plugin），都是直接内置在 Kubernetes 代码库中进行开发的，目前代码库中包含 20 多个插件。但这种方式带来了很多问题。

1. 这些插件对 Kubernetes 代码本身的稳定性以及安全性引入了很多未知的风险，一个很小的 Bug 都有可能导致集群受到攻击或者无法工作。
2. 这些插件的维护和 Kubernetes 的正常迭代紧密耦合在一起，一起打包和编译。即便是某个单一插件出现了 Bug，都需要通过升级 Kubernetes 的版本来修复。
3. 社区需要维护所有的 volume plugin，并且要经过完整的测试验证流程，来保证可用性，这给社区的正常迭代平添了很多麻烦。
4. 各个卷插件依赖的包也都要算作 Kubernetes 项目的一部分，这会让 Kubernetes 的依赖变得臃肿。
5. 开发者被迫要将这些插件代码进行开源。

为此，社区早在 v1.2 版本就开始尝试用 FlexVolume 插件来解决，在 v1.8 版本 GA，并停止接收任何新增的内置 volume plugin 了。用户需要遵循 FlexVolume 约定的接口规范，自己开发可执行的程序，比如二进制程序、Shell脚本等，以命令行参数作为输入，并返回 JSON 格式的结果，这样 Kubelet 就可以通过 exec 的方式调用用户的插件程序了，如下图所示。这种方式方便了各个插件的开发、更新、维护和升级，同时也和 Kubernetes 进行了解耦。在使用的时候，需要用户提前将这些二进制的文件放到各个节点上指定的目录里面（默认是`/usr/libexec/kubernetes/kubelet-plugins/volume/exec/`），方便 Kubelet 可以动态发现和调用。

![Drawing 4.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl9pylGAMLzrAAC5tLcwqrg002.png)

但是在实际使用中，FlexVolume 还是有很多局限性的。比如:

- 需要一些前置依赖包，像 ceph 就需要安装`ceph-common`等依赖包;
- 部署很麻烦，而且往往需要很高的执行权限，要可以访问宿主机上的根文件系统。

为了彻底解决FlexVolume 开发过程中遇到的问题，CSI采用了容器化的方式进行部署，并基于 Kubernetes 的语义使用各种已定义的对象，比如下节课我们要讲的 PV、PVC、StorageClass，等等。各家厂商只需要按照 CSI 的规范实现各自的接口即可。大家可以通过[这份官方的清单列表](https://kubernetes-csi.github.io/docs/drivers.html)，来查看可以用于生产环境的 CSI Driver。



### 写在最后

本节课讲的 Configmap、Secret、Downward API、EmptyDir 以及 Hostpath 都是日常频繁会使用到的 volume plugin，数据都会放在 Pod 所在的宿主机上。但是对于一些云厂商或者第三方的存储系统，我建议你直接通过 CSI 来使用。





## 10 | 存储管理：怎样对业务数据进行持久化存储？



这里考虑如下几个问题：

1. **共享 Volume**。目前 Pod 内的 Volume 其实跟 Pod 是存在静态的一一绑定关系，即生命周期绑定。这导致不同 Pod 之间无法共享 Volume。
2. **复用 Volume 中的数据**。当 Pod 由于某种原因失败，被工作负载控制器删除重新创建后，我们需要能够复用 Volume 中的旧数据。
3. **Volume 自身的一些强关联诉求**。对于有状态工作负载 StatefulSet 来说，当其管理的 Pod 由于所在的宿主机出现一些硬件或软件问题，比如磁盘损坏、kernel 异常等，Pod 重新“长”到别的节点上，这时该如何保证 Volume 和 Pod 之间强关联的关系？
4. **Volume 功能及语义扩展**，比如容量大小、标签信息、扩缩容等。

为此我们在 Kubernetes 中引入了一个专门的对象 Persistent Volume（简称 PV），将计算和存储进行分离，可以使用不同的控制器来分别管理。

同时通过 PV，我们也可以和 Pod 自身的生命周期进行解耦。一个 PV 可以被几个 Pod 同时使用，即使 Pod 被删除后，PV 这个对象依然存在，其他新的 Pod 依然可以复用。为了更好地描述这种关联绑定关系，易于使用，并且屏蔽更多用户并不关心的细节参数（比如 PV 由谁提供、创建在哪个 zone/region、怎么去访问到，等等），我们通过一个抽象对象 Persistent Volume Claim（PVC）来使用 PV。

我们可以把 PV 理解成是对实际的物理存储资源描述，PVC 是便于使用的抽象 API。在 Kubernetes 中，我们都是在 Pod 中通过PVC 的方式来使用 PV 的，见下图。

![image.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F9sO-OAQHEDAABbxhBYt9I383.png)



在 Kubernetes 中，创建 PV（PV Provision） 有两种方式，即静态和动态，如下图所示。

![image (1).png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F9sO-6AfcmHAAJfQ6jac2Q610.png)



### 静态 PV

![image (2).png](D:\学习资料\笔记\k8s\k8s图\CgqCHl9sO_WAAgmIAALzurk_uD0139.png)

我们先来看静态 PV（Static PV），管理员通过手动的方式在后端存储平台上创建好对应的 Volume，然后通过 PV 定义到 Kubernetes 中去。开发者通过 PVC 来使用。我们来看个 HostPath 类型的 PV 例子：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: task-pv-volume # pv 的名字
  labels: # pv 的一些label
    type: local
spec:
  storageClassName: manual
  capacity: # 该 pv 的容量
    storage: 10Gi
  accessModes: # 该 pv 的接入模式
    - ReadWriteOnce
  hostPath: # 该 pv 使用的 hostpath 类型，还支持通过 CSI 接入其他 plugin
    path: "/mnt/data"
```

这里，我们定义了一个名为task-pv-volume的 PV，PV 是集群的资源，并不属于某个 namespace。其中storageClassName这个字段是某个StorageClass对象的名字。

对于每一个 PV，我们都要为其设置存储能力，目前只支持对存储空间的设置，比如我们这里设置了 10G 的空间大小。未来社区还会加入其他的配置，诸如 IOPS（Input/Output Operations Per Second，每秒输入输出次数）、吞吐量等。

这里头accessMode可以指定该 PV 的几种访问挂载方式：

- ReadWriteOnce（RWO）表示该卷只可以以读写方式挂载到一个 Pod 内；
- ReadOnlyMany（ ROX）表示该卷可以挂载到多个节点上，并被多个 Pod 以只读方式挂载；
- ReadWriteMany（RWX）表示卷可以被多个节点以读写方式挂载供多个 Pod 同时使用。

注意一个 PV 只能有一种访问挂载模式。不同的 volume plugin 支持的 accessMode 并不相同，在使用的时候，你可以参照官方的[这个表格](https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes/#access-modes)进行选择。

我们创建好后查看这个 PV：

```sh
$ kubectl get pv task-pv-volume
NAME             CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS      CLAIM     STORAGECLASS   REASON    AGE
task-pv-volume   10Gi       RWO           Retain          Available             manual                   4s
```

可以看到，这个 PV 的状态为Available（可用）。
这里我们还看到上面`kubectl get`的输出里面有个 ReclaimPolicy 字段，该字段表明对 PV 的回收策略，默认是 Retain，即 PV 使用完后数据保留，需要由管理员手动清理数据。除了 Retain 外，还支持如下策略：

- Recycle，即回收，这个时候会清除 PV 中的数据；
- Delete，即删除，这个策略常在云服务商的存储服务中使用到，比如 AWS EBS。

下面我们再创建一个 PVC：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: task-pv-claim
  namespace: demo
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
```

创建好了以后，Kubernetes 会为 PVC 匹配满足条件的 PV。我们在 PVC 里面指定storageClassName为 manua，这个时候就只会去匹配storageClassName同样为 manual 的 PV。一旦发现合适的 PV 后，就可以绑定到该 PV 上。

PVC 是 namespace 级别的资源，我们来创建看看:

```sh
$ kubectl get pvc -n demo
NAME            STATUS   VOLUME           CAPACITY   ACCESS MODES   STORAGECLASS   AGE
task-pv-claim   Bound    task-pv-volume   10Gi       RWO            manual         9s
```

我们可以看到 这个 PVC 已经和我们上面的 PV 绑定起来了。我们再来查看下task-pv-volume这个 PV 对象，可以看到它的状态也从Available变成了 Bound。

PV 一般会有如下五种状态：

1. Pending 表示目前该 PV 在后端存储系统中还没创建完成；
2. Available 即闲置可用状态，这个时候还没有被绑定到任何 PVC 上；
3. Bound 就像上面例子里似的，这个时候已经绑定到某个 PVC 上了；
4. Released 表示已经绑定的 PVC 已经被删掉了，但资源还未被回收掉；
5. Failed 表示回收失败。

同样，对于 PVC 来说，也有如下三种状态：

1. Pending 表示还未绑定任何 PV；
2. Bound 表示已经和某个 PV 进行了绑定；
3. Lost 表示关联的 PV 失联。

下面我们来看看，如何在 Pod 中使用静态的 PV。看如下的例子：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: task-pv-pod
  namespace: demo
spec:
  volumes:
    - name: task-pv-storage
      persistentVolumeClaim:
        claimName: task-pv-claim
  containers:
    - name: task-pv-container
      image: nginx:1.14.2
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: task-pv-storage
```

创建完成以后：

```sh
$ kubectl get pod task-pv-pod -n demo
NAME          READY   STATUS    RESTARTS   AGE
task-pv-pod   1/1     Running   1          82s

$ kubectl exec -it task-pv-pod -n demo -- /bin/bash
root@task-pv-pod:/# df -h
Filesystem      Size  Used Avail Use% Mounted on
overlay          40G  5.0G   33G  14% /
tmpfs            64M     0   64M   0% /dev
tmpfs           996M     0  996M   0% /sys/fs/cgroup
/dev/vda1        40G  5.0G   33G  14% /etc/hosts
shm              64M     0   64M   0% /dev/shm
overlay         996M  4.0M  992M   1% /usr/share/nginx/html
tmpfs           996M   12K  996M   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs           996M     0  996M   0% /proc/acpi
tmpfs           996M     0  996M   0% /sys/firmware
```

可以看到，PV 已经正确挂载到 Pod 内。
静态 PV 最大的问题就是使用起来不够方便，都是管理员提前创建好一批指定规格的 PV，无法做到按需创建。使用过程中，经常会遇到由于资源大小不匹配，规格不对等，造成 PVC 无法绑定 PV 的情况。同时还会造成资源浪费，比如一个只需要 1G 空间的 Pod，绑定了 10G 的 PV。

这些问题，都可以通过动态 PV 来解决。



### 动态 PV

要想动态创建 PV，我们需要一些参数来帮助我们创建 PV。这里我们用StorageClass这个对象来描述，你可以在 Kubernetes 中定义很多的 StorageClass，如下就是一个 Storage 的定义例子：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-rbd-sc
  annotation:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/rbd # 必填项，用来指定volume plugin来创建PV的物理资源
parameters: # 一些参数
  monitors: 10.16.153.105:6789
  adminId: kube
  adminSecretName: ceph-secret
  adminSecretNamespace: kube-system
  pool: kube
  userId: kube
  userSecretName: ceph-secret-user
  userSecretNamespace: default
  fsType: ext4
  imageFormat: "2"
  imageFeatures: "layering"
```

你可以通过注释`storageclass.kubernetes.io/is-default-class`来指定默认的 StorageClass。这样新创建出来的 PVC 中的 storageClassName 字段就会自动使用默认的 StorageClass。

这里有个 provisioner 字段是必填项，主要用于指定使用那个 volume plugin 来创建 PV。没错，这里正是对应我们上节课讲过的 CSI driver 的名字。

现在我们来讲一下动态 PV 工作的过程:

![image (3).png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F9sPBCAbuyAAAoLbbMYu80674.png)



首先我们定义了一个 StorageClass。当用户创建好 Pod 以后，指定了 PVC，这个时候 Kubernetes 就会根据 StorageClass 中定义的 Provisioner 来调用对应的 plugin 来创建 PV。PV 创建成功后，跟 PVC 进行绑定，挂载到 Pod 中使用。



### StatefulSet 中怎么使用 PV 和 PVC？

还记得我们之前讲 StatefulSet 中遗留的问题吗？对于 StatefulSet 管理的 Pod，每个 Pod 使用的 Volume 中的数据都不一样，而且相互之间关系是需要强绑定的。这个时候就不能在 StatefulSet 的`spec.template`去直接指向 PV 和 PVC了。于是我们在 StatefulSet 中使用了volumeClaimTemplate，有了这个 template 我们就可以为每一个 Pod 生成一个单独的 PVC，并且绑定 PV 了，从而实现有状态服务各个 Pod 都有自己专属的存储。这里生成的 PVC 名字跟 StatefulSet 的 Pod 名字一样，都是带有特定的序列号的。

你可以看看这里 StatefulSet 的例子:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: k8s.gcr.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```



### 写在最后

这节课我们讲了 PV、PVC 以及 StorageClass，它们直接的关系以及设计思路。你也许刚接触这几个概念的时候，有些稀里糊涂，但是通过分析各个对象要解决的问题，可以帮助你更好地掌握它们。







## 11 | K8s Service：轻松搞定服务发现和负载均衡



经过前面几节课的学习，我们已经可以发布高可用的业务了，通过 PV 持久化地保存数据，通过 Deployment或Statefulset 这类工作负载来管理多实例，从而保证服务的高可用。

想一想，这个时候如果有别的应用来访问我们的服务的话，该怎么办呢？直接访问后端的 Pod IP 吗？不，这里我们还需要做服务发现（Service Discovery）。



### 为什么需要服务发现？

传统的应用部署，服务实例的网络位置是固定的，即在给定的机器上进行部署，这个时候的服务地址一般是机器的 IP 加上某个特定的端口号。

但是在 Kubernetes 中，这是完全不同的。业务都是通过 Pod 来承载的，每个 Pod 的生命周期又很短暂，用后即焚，IP 地址也都是随机分配，动态变化的。而且，我们还经常会遇到一些高并发的流量进来，这时候往往需要快速扩容，服务的实例数也会随之动态调整。因此我们在这里就不能用传统的基于 IP 的方式去访问某个服务了。这个对于所有云上的系统，以及微服务应用体系，都是一个大难题。这时我们就需要做服务发现来确定服务的访问地址。

今天我们就来聊聊 Kubernetes 中的服务发现 —— Service。



### Kubernetes 中的 Service

在之前的课程中，我们知道 Deployment、StatefulSet 这类工作负载都是通过 labelSelector 来管理一组 Pod 的。那么 Kubernetes 中的 Service 也采用了同样的做法，如下图。

![image (4).png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F9sPFSAMnmPAAO5U6ZAsB0716.png)

这样一个 Service 会选择集群所有 label 中带有`app=nginx`和`env=prod`的 Pod。

我们来看看这样的一个 Service 是如何定义的：

```yaml
$ cat nginx-svc.yaml
apiVersion: v1
kind: Service
metadata:  
  name: nginx-prod-svc-demo
  namespace: demo # service 是 namespace 级别的对象
spec:
  selector:       # Pod选择器
    app: nginx
    env: prod
  type: ClusterIP # service 的类型
  ports:  
  - name: http 
    port: 80       # service 的端口号
    targetPort: 80 # 对应到 Pod 上的端口号
    protocol: TCP  # 还支持 udp，http 等
```

现在我们先来看如下一个 Deployment的定义：

```yaml
$ cat nginx-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-prod-deploy
  namespace: demo
  labels:
    app: nginx
    env: prod
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
      env: prod
  template:
    metadata:
      labels:
        app: nginx
        env: prod
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

我们创建好这个 Deployment后，查看其 Pod 状态：

```sh
$ kubectl get deploy -n demo
NAME                READY   UP-TO-DATE   AVAILABLE   AGE
nginx-prod-deploy   3/3     3            3           5s

$ kubectl get pod -n demo -o wide
NAME                                 READY   STATUS    RESTARTS   AGE   IP          NODE             NOMINATED NODE   READINESS GATES
nginx-prod-deploy-6fb6fbb77d-h2gn4   1/1     Running   0          87s   10.1.0.31   docker-desktop   <none>           <none>
nginx-prod-deploy-6fb6fbb77d-r78k9   1/1     Running   0          87s   10.1.0.29   docker-desktop   <none>           <none>
nginx-prod-deploy-6fb6fbb77d-xm8tp   1/1     Running   0          87s   10.1.0.30   docker-desktop   <none>           <none>
```

我们再来创建下上面定义的 Service：

```sh
$ kubectl create -f nginx-svc.yaml
service/nginx-prod-svc-demo created

$ kubectl get svc -n demo
NAME                  TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
nginx-prod-svc-demo   ClusterIP   10.111.193.186   <none>        80/TCP    6s

# 可以看到，这个 Service 分配到了一个地址为10.111.193.186的 Cluster IP，这是一个虚拟 IP（VIP）地址，集群内所有的 Pod 和 Node 都可以通过这个虚拟 IP 地址加端口的方式来访问该 Service。这个 Service 会根据标签选择器，把匹配到的 Pod 的 IP 地址都挂载到后端。我们使用`kubectl describe`来看看这个 Service：

$ kubectl describe svc -n demo nginx-prod-svc-demo
Name:              nginx-prod-svc-demo
Namespace:         demo
Labels:            <none>
Annotations:       <none>
Selector:          app=nginx,env=prod
Type:              ClusterIP
IP:                10.111.193.186
Port:              http  80/TCP
TargetPort:        80/TCP
Endpoints:         10.1.0.29:80,10.1.0.30:80,10.1.0.31:80
Session Affinity:  None
Events:            <none>
```

可以看到，这时候 Service 关联的 Endpoints 里面有三个 IP 地址，和我们上面看到的 Pod IP 地址完全吻合。

我们试着来缩容 Deployment 的副本数，再来看看 Service 关联的 Pod IP 地址有什么变化：

```sh
$ kubectl scale --replicas=2 deploy -n demo nginx-prod-deploy
deployment.apps/nginx-prod-deploy scaled

$ kubectl get pod -n demo -o wide
NAME                                 READY   STATUS    RESTARTS   AGE   IP          NODE             NOMINATED NODE   READINESS GATES
nginx-prod-deploy-6fb6fbb77d-r78k9   1/1     Running   0          11m   10.1.0.29   docker-desktop   <none>           <none>
nginx-prod-deploy-6fb6fbb77d-xm8tp   1/1     Running   0          11m   10.1.0.30   docker-desktop   <none>           <none>

$ kubectl describe svc -n demo nginx-prod-svc-demo
Name:              nginx-prod-svc-demo
Namespace:         demo
Labels:            <none>
Annotations:       <none>
Selector:          app=nginx,env=prod
Type:              ClusterIP
IP:                10.111.193.186
Port:              http  80/TCP
TargetPort:        80/TCP
Endpoints:         10.1.0.29:80,10.1.0.30:80
Session Affinity:  None
Events:            <none>
```

可见当 Pod 的生命周期发生变化时，比如缩容或者异常退出，Service 会自动把有问题的 Pod 从后端地址中摘除。这样实现的好处在于，我们可以始终通过一个虚拟的稳定 IP 地址来访问服务，而不用关心其后端真正实例的变化。
Kubernetes 中 Service 一共有四种类型，除了上面讲的 ClusterIP，还有 NodePort、LoadBalancer 和 ExternalName。

其中 LoadBalancer 在云上用的较多，使用的时候需要跟各家云厂商做适配，比如部署对应的 cloud-controller-manager。有兴趣的话，可以查看 [这个文档](https://kubernetes.io/zh/docs/concepts/services-networking/service/#loadbalancer)，看看如何在云上使用。LoadBalancer主要用于做外部的服务发现，即暴露给集群外部的访问。

ExternalName 类型的 Service 在实际中使用的频率不是特别高，但是对于某些特殊场景还是有一些用途的。比如在云上或者内部已经运行着一个应用服务，但是暂时没有运行在 Kubernetes 中，如果想让在 Kubernetes 集群中的 Pod 访问该服务，这时当然可以直接使用它的域名地址，也可以通过 ExternalName 类型的 Service 来解决。这样就可以直接访问 Kubernetes 内部的 Service 了。

这样一来方便后续服务迁移到 Kubernetes 中，二来也方便随时切换到备份的服务上，而不用更改 Pod 内的任何配置。由于使用频率并不高，我们不做重点介绍，有兴趣可以参考 [这篇文档](https://kubernetes.io/zh/docs/concepts/services-networking/service/#externalname)。

我们最后来看下另外一种 NodePort 类型的 Service：

```yaml
apiVersion: v1
kind: Service
metadata:  
  name: my-nodeport-service
  namespace: demo
spec:
  selector:    
    app: my-app
  type: NodePort # 这里设置类型为 NodePort
  ports:  
  - name: http
    port: 80
    targetPort: 80
    nodePort: 30000
    protocol: TCP
```

顾名思义，这种类型的 Service 通过任一 Node 节点的 IP 地址，再加上端口号就可以访问 Service 后端负载了。我们看下面这个流量图，方便理解。

![image (5).png](D:\学习资料\笔记\k8s\k8s图\CgqCHl9sPGuAbYFBAAR9dQ-LWLw022.png)



NodePort 类型的 Service 创建好了以后，Kubernetes 会在每个 Node 节点上开个端口，比如这里的 30000 端口。这个时候我们可以访问任何一个 Node 的 IP 地址，通过 30000 端口即可访问该服务。

那么如果在集群内部，该如何访问这些 Service 呢？



### 集群内如何访问 Service？

一般来说，在 Kubernetes 集群内，我们有两种方式可以访问到一个 Service。

1. 如果该 Service 有 ClusterIP，我们就可以直接用这个虚拟 IP 去访问。比如我们上面创建的 nginx-prod-svc-demo 这个 Service，我们通过`kubectl get svc nginx-prod-svc-demo -n dmeo` 就可以看到其 Cluster IP 为 10.111.193.186，端口号为 80。那么我们通过 http(s)://10.111.193.186:80 就可以访问到该服务。

2. 当然我们也可以使用该 Service 的域名，依赖于集群内部的 DNS 即可访问。还是以上面的例子做说明，同 namespace 下的 Pod 可以直接通过 nginx-prod-svc-demo 这个 Service 名去访问。如果是不同 namespace 下的 Pod 则需要加上该 Service 所在的 namespace 名，即nginx-prod-svc-demo.demo去访问。

如果在某个 namespace 下，Service 先于 Pod 创建出来，那么 kubelet 在创建 Pod 的时候，会自动把这些 namespace 相同的 Service 访问信息当作环境变量注入 Pod 中，即`{SVCNAME}_SERVICE_HOST`和`{SVCNAME}_SERVICE_PORT`。这里SVCNAME对应是各个 Service 的大写名称，名字中的横线会被自动转换成下划线。比如：

```sh
# env
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_SERVICE_PORT=443
HOSTNAME=nginx-prod-deploy2-68d8fb9586-4m5hr
NGINX_PROD_SVC_DEMO_SERVICE_PORT_HTTP=80
NGINX_PROD_SVC_DEMO_SERVICE_HOST=10.111.193.186
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
NGINX_PROD_SVC_DEMO_SERVICE_PORT=80
NGINX_PROD_SVC_DEMO_PORT=tcp://10.111.193.186:80
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_PROTO=tcp
NGINX_PROD_SVC_DEMO_PORT_80_TCP_ADDR=10.111.193.186
NGINX_PROD_SVC_DEMO_PORT_80_TCP_PORT=80
NGINX_PROD_SVC_DEMO_PORT_80_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_SERVICE_HOST=10.96.0.1
NGINX_PROD_SVC_DEMO_PORT_80_TCP=tcp://10.111.193.186:80
...
```

知道了这两种访问方式，我们就可以在启动 Pod 的时候，通过注入环境变量、启动参数或者挂载配置文件等方式，来指定要访问的 Service 信息。如果是同 namespace 的 Pod，可以直接从自己的环境变量中知道同 namespace 下的其他 Service 的访问方式。

那么这样通过该 Service 进行访问时，Kubernetes 又是如何实现负载均衡的呢，即将流量打到后端挂载的各个 Pod 上面去？



### 集群内部的负载均衡如何实现？

这一切都是通过 kube-proxy 来实现的。所有的节点上都会运行着一个 kube-proxy的服务，主要监听 Kubernetes 中的 Service 和 Endpoints。当 Service 或 Endpoints 发生变化时，就会调用相应的接口创建对应的规则出来，常用模式主要是 iptables 模式和 IPVS 模式。iptables 模式比较简单，使用起来也方便。而 IPVS 支持更高的吞吐量以及复杂的负载均衡策略，你可以通过 [官方文档](https://kubernetes.io/zh/docs/concepts/services-networking/service/#proxy-mode-ipvs) 了解更多 IPVS 模式的工作原理。

目前 kube-proxy 默认的工作方式是 iptables 模式，我们来通过如下一个 iptables 模式的例子来看一下实际访问链路是什么样的。

![image (6).png](D:\学习资料\笔记\k8s\k8s图\CgqCHl9sPH6AMn4jAA32mDhWECM868.png)

当你通过 Service 的域名去访问时，会先通过 CoreDNS 解析出 Service 对应的 Cluster IP，即虚拟 IP。然后请求到达宿主机的网络后，就会被kube-proxy所配置的 iptables 规则所拦截，之后请求会被转发到每一个实际的后端 Pod 上面去，这样就实现了负载均衡。



### Headless Service

如果我们在定义 Service 的时候，将spec.clusterIP设置为 None，这个时候创建出来的 Service 并不会分配到一个 Cluster IP，此时它就被称为Headless Service。

现在我们来通过一个例子来看看 Headless Service 有什么特殊的地方。我们在上面的 Service 基础上，增加了spec.clusterIP为None，并命名为nginx-prod-demo-headless-svc：

```yaml
apiVersion: v1
kind: Service
metadata:  
  name: nginx-prod-demo-headless-svc
  namespace: demo 
spec:
  clusterIP: None
  selector: 
    app: nginx
    env: prod
  type: ClusterIP
  ports:  
  - name: http 
    port: 80 
    targetPort: 80 
    protocol: TCP 
```

通过 kubectl 创建成功后，我们现在kubectl get一下看看：

```sh
$ kubectl get svc -n demo
NAME                           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
nginx-prod-demo-headless-svc   ClusterIP   None             <none>        80/TCP    4s
nginx-prod-svc-demo            ClusterIP   10.111.193.186   <none>        80/TCP    3d5h
```

可以看到这个叫 nginx-prod-demo-headless-svc 的 Service 并没有分配到一个 ClusterIP，符合预期，毕竟我们已经设置了 spec.clusterIP 为 None。

我们来创建一个 Pod，看看 DNS 记录有没有什么差别。 Pod 的 yaml 文件如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: headless-svc-test-pod
  namespace: demo
spec:
  containers:
  - name: dns-test
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
```

该 Pod 创建出来后，我们通过kubectl exec进入 Pod 中，运行如下两条 nslookup 查询命令，依次查看两个 Service 对应的 DNS 记录：

```sh
$ kubectl exec -it -n demo headless-svc-test-pod sh
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl kubectl exec [POD] -- [COMMAND] instead.
/ # nslookup nginx-prod-demo-headless-svc
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
Name:      nginx-prod-demo-headless-svc
Address 1: 10.1.0.32 10-1-0-32.nginx-prod-demo-headless-svc.demo.svc.cluster.local
Address 2: 10.1.0.33 10-1-0-33.nginx-prod-demo-headless-svc.demo.svc.cluster.local
/ # nslookup nginx-prod-svc-demo
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
Name:      nginx-prod-svc-demo
Address 1: 10.111.193.186 nginx-prod-svc-demo.demo.svc.cluster.local
```

我们可以看到正常 Service`nginx-prod-svc-demo`对应的 DNS 记录的是与虚拟 IP`10.111.193.166`有关的记录，而 Headless Service`nginx-prod-demo-headless-svc`则解析到所有后端的 Pod 的地址。
总结下， Headless Service 主要有如下两种场景。

1. 用户可以自己选择要连接哪个 Pod，通过查询 Service 的 DNS 记录来获取后端真实负载的 IP 地址，自主选择要连接哪个 IP；

2. 可用于部署有状态服务。回顾下，我们在 StatefulSet 那节课也有 Headless Service 例子，每个 StatefulSet 管理的 Pod 都有一个单独的 DNS 记录，且域名保持不变，即`<PodName>.<ServiceName>.<NamespaceName>.svc.cluster.local`。这样 Statefulset 中的各个 Pod 就可以直接通过 Pod 名字解决相互间身份以及访问问题。



### 写在最后

Service 是 Kubernetes 很重要的对象，主要负责为各种工作负载暴露服务，方便各个服务之间互访。通过对一组 Pod 提供统一入口，Service 极大地方便了用户使用，用户只需要与 Service 打交道即可，而不用过多地关心后端实例的变动，比如扩缩容、容器异常、节点宕机，等等。

正是因为有了 Service 的支持，你在 Kubernetes 部署业务会非常方便，这是相比较于 Docker Swarm 以及 Mesos Marathon 巨大的技术优势，可以说，它是 Kubernetes 是运行大规模微服务的最佳载体。








## 12 | Helm Charts：如何在生产环境中释放部署生产力？



Kubernetes 是一个强大的容器调度系统，你可以通过一些声明式的定义，很方便地在 Kubernetes 中部署业务。

现在你一定很想尝试在 Kubernetes 中部署一个稍微复杂的系统，比如下面这个典型的三层架构：前端、后端和数据层。

![image (7).png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F9sPOKACHGPAADXdRLPJwo750.png)



也许你内心里已经开始跃跃欲试，但是转念一想，事情好像没那么简单啊。你感觉到这里面要写一堆的 YAML 文件，每一层架构都至少要写 2 个 YAML 文件，写的时候还需要理解各种对象、注意各种缩进的层级关系、还要设置一堆配置参数等。这对于刚接触 Kubernetes 的人来说是一个很高的门槛，简直无法跨越。同时，对于业务交付的同学，也将是一个大灾难，因为如果缺少任何一个 YAML 文件，都会导致整个系统无法正常工作。

看到这里，你也许会想到能不能通过一种包的方式进行管理呢，类似于 Node.js 通过 npm 来管理包，Debian 系统通过 dpkg 来管理包，而 Python 通过 pip 来管理包。

那么在 Kubernetes 中， 这个答案就是**Helm**。

Helm 降低了使用 Kubernetes 的门槛，你无须从头开始编写应用部署文件，甚至都不需要了解 Kubernetes 中的各个对象以及相应的 YAML 语义。直接通过 Helm 就可以在自己的 Kubernetes 中一键部署需要的应用。

Helm 降低了使用 Kubernetes 的门槛，你无须从头开始编写应用部署文件，甚至都不需要了解 Kubernetes 中的各个对象以及相应的 YAML 语义。直接通过 Helm 就可以在自己的 Kubernetes 中一键部署需要的应用。

对于开发者而言，通过 Helm 进行打包，管理应用内部的各种依赖关系，极其方便。并且可以借助于版本化的概念，方便自身的迭代和分发。除此之外，Helm 还有一些其他强大的功能，请跟我一起开始今天 Helm 的学习之旅。

作为 CNCF 社区官方的项目，Helm 在今年四月份已经宣布毕业，这进一步彰显了 Helm 的市场接受度、生产环境采用率以及技术影响力。

我们先来理解下 Helm 中几个重要的概念。



### Helm 中的几个概念

在 Helm 中，有三个非常核心的概念—— Chart、Config 和 Release。

Chart 可以理解成应用的安装包，通常包含了一组我们在 Kubernetes 要部署的 YAML 文件。一个 Chart 就是一个目录，我们可以将这个目录进行压缩打包，比如打包成 some-chart-version.tgz 类型的压缩文件，方便传输和存储。

每个这样的 Chart 包内都必须有一个 Chart.yaml 文件，类似这样：

```yaml
apiVersion: v1
name: redis
version: 11.0.0
appVersion: 6.0.8
description: Open source, advanced key-value store. It is often referred to as a data structure server since keys can contain strings, hashes, lists, sets and sorted sets.
keywords:
  - redis
  - keyvalue
  - database
home: https://github.com/bitnami/charts/tree/master/bitnami/redis
icon: https://bitnami.com/assets/stacks/redis/img/redis-stack-220x234.png
sources:
  - https://github.com/bitnami/bitnami-docker-redis
  - http://redis.io/
maintainers:
  - name: Bitnami
    email: containers@bitnami.com
  - name: desaintmartin
    email: cedric@desaintmartin.fr
engine: gotpl
annotations:
  category: Database
```

这个 Chart.yaml 主要用来描述该 Chart 的名字、版本号，以及关键字等信息。

有了这个 Chart，我们就可以在 Kubernetes集群中部署了。每一次的安装部署，我们称为一个 Release。在同一个 Kubernetes 集群中，可以有多个 Release。你可以将 Release 理解成是 Chart 包部署后的一个 Chart（应用）实例。

> 注： 这个 Release 和我们通常概念中理解的“版本”有些差异。

同时，为了能够让 Chart 实现参数可配置，即 Config，**我们在每个 Chart 包内还有一个 values.yaml 文件，用来记录可配置的参数和其默认值。**在每个 Release 中，我们也可以指定自己的 values.yaml 文件用来覆盖默认的配置。

此外，我们还需要统一存放这些 Chart，即 Repository（仓库）。这个概念和我们以前的 yum repo 是一样的。本质上来说，**Repository 就是一个 Web 服务器，不仅保存了一系列的 Chart 软件包让用户下载安装使用，还有对应的清单文件以供查询。**

通过 Helm，我们可以同时使用和管理多个不同的 Repository，就像我们可以给 yum 配置多个源一样。

掌握了这些概念，我们现在再拉远视角，整体来看一下。



### Helm 的安装及架构组成

首先，Helm 的安装非常简单，你可以通过如下的命令安装最新版本的 Helm 可执行文件：

```sh
$ curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 11213  100 11213    0     0  12109      0 --:--:-- --:--:-- --:--:-- 12096
Downloading https://get.helm.sh/helm-v3.3.1-darwin-amd64.tar.gz
Verifying checksum... Done.
Preparing to install helm into /usr/local/bin
Password:
helm installed into /usr/local/bin/helm
```

当然你也可以去 [官方的文档](https://helm.sh/docs/intro/install/) 里选择合适自己的安装方式。

安装完成后，我们来查看下 Helm 的版本：

```sh
$ helm version
version.BuildInfo{Version:"v3.3.1", GitCommit:"249e5215cde0c3fa72e27eb7a30e8d55c9696144", GitTreeState:"clean", GoVersion:"go1.14.7"}
```

可以看到目前我们安装的版本是v3.3.1，这是 Helm 3 的一个版本。
其实在 Helm 3 之前，还有 Helm 2，目前还有不少人在继续使用 Helm 2 的版本。现在我们来简单了解下 Helm 2 和 Helm 3 。



### Helm 2

Helm 2 于 2015 年在 KubeCon 上亮相。其架构如下图所示，是个常见的 CS 架构，由 Helm Client 和 Tiller 两部分组成。

![image (8).png](D:\学习资料\笔记\k8s\k8s图\CgqCHl9sPQ2AWA7kAAKDpOtvIDk020.png)

Tiller 是 Helm 的服务端， 用于接收来自 Helm Client 的请求，主要负责部署 Helm Chart、管理 Release（比如升级、删除、回滚等操作），以及在 Kubernetes 集群中创建应用。TIller 通常部署在 Kubernetes 中，直接通过`helm init`就可以在 Kuberentes 集群中拉起服务。

Helm Client 主要负责跟用户进行交互，通过命令行就可以完成 Chart 的安装、升级、删除等操作。Helm Client 可以从公有或者私有的 Helm 仓库中拉取 Chart 包，通过用户指定的 Config，就可以直接传给后端的 Tiller 服务，使之在集群内进行部署。

这里，我们可以看到 Tiller 在 Helm 2 和 Kubernetes 集群中间其实起到了一个中转的作用，因此 Tiller 实际上是代替用户在 Kubernetes 集群中执行操作，所以 Tiller 在运行中往往需要一个很高的权限。这里 Tiller 存在着两大安全风险：

1. 要对外暴露自身的端口；

2. 自身运行过程中，跟 Kubernetes 交互需要很高的权限，这样才可以在 Kuberentes 中创建、删除各种各样的资源。

当然了，之所以有这种架构设计是由于在 Helm 2 开发的时候，Kubernetes 的 RBAC 体系还没有建立完成。Kubernetes 社区直到 2017 年 10 月份 v1.8 版本才默认采用了 RBAC 权限体系。随着 Kubernetes 的各项功能日益完善，主要是 RBAC 能力，Tiller 这个组件存在的必要性进一步降低，社区也在一直想着怎么去解决 TIller 的安全问题。

我们再来看看 Helm 3。



### Helm 3

Helm 3 是在 Helm 2 之上的一次大更改，于 2019 年 11 月份正式推出。相比较于 Helm 2， Helm 3 简单了很多，移除了 Tiller，只剩下一个 Helm Client。在使用中，你只需要这一个二进制可执行文件即可，使用你当前用的 kubeconfig 文件，即可在 Kubernetes 集群中部署 Chart、管理 Release。

至此，Helm 2 开始要退出历史的舞台了。目前 Helm 社区官方也逐步开始暂停维护 Helm 2，最终的 Bugfix 修复截止到今年的 8 月份，版本号是 v2.16.10，而安全相关的 patch 会继续支持，直到 今年的 11 月份。

如果你还在使用 Helm 2，建议你尽快切换到 Helm 3 上来。Helm 社区也提供了相关的插件，帮助你从 Helm 2 迁移到 Helm 3 中，可以参考这篇 Helm 的 [官方迁移文档](https://helm.sh/docs/topics/v2_v3_migration/)，这里我就不再赘述了。

通过这个版本迭代的过程，你应该也清楚了 Helm 的原理，对它更加熟悉了，接下来我们一起动手实践看看。



### 如何创建和部署 Helm Chart

创建一个 Chart 很简单，通过`helm create`的命令就可以创建一个 Chart 模板出来：

```sh
$ helm create hello-world
Creating hello-world
```

我们现在来看看这个 Chart 的目录里面有什么：

```sh
$ tree ./hello-world
./hello-world
├── Chart.yaml
├── charts
├── templates
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── service.yaml
│   ├── serviceaccount.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml
3 directories, 10 files
```

这里面包含了我们上面讲的 Chart.yaml 和 values.yaml。

先看看这里 Chart.yaml 的内容：

```yaml
apiVersion: v2
name: hello-world
description: A Helm chart for Kubernetes
# A chart can be either an 'application' or a 'library' chart.
#
# Application charts are a collection of templates that can be packaged into versioned archives
# to be deployed.
#
# Library charts provide useful utilities or functions for the chart developer. They're included as
# a dependency of application charts to inject those utilities and functions into the rendering
# pipeline. Library charts do not define any templates and therefore cannot be deployed.
type: application
# This is the chart version. This version number should be incremented each time you make changes
# to the chart and its templates, including the app version.
# Versions are expected to follow Semantic Versioning (https://semver.org/)
version: 0.1.0
# This is the version number of the application being deployed. This version number should be
# incremented each time you make changes to the application. Versions are not expected to
# follow Semantic Versioning. They should reflect the version the application is using.
appVersion: 1.16.0
```

可以看到，这里 Chart 的名字就是我们 Chart 所在目录的名字。你在使用的时候，就可以在这个文件里面添加 keyword，以及 annotation，方便后续查找和分类。
values.yaml 里面包含了一些可配置的参数，你可以根据自己的需要进行添加，或者修改默认值等。

这个目录下面有个`charts`文件夹，主要用来放置一些子 Chart 包，也可以是 tar 包、 Chart 目录。你可以根据自己的场景，决定需不需要放置子 Chart 包。

这个目录里面还有个`templates`文件夹，这个文件夹里面的文件是一些模板文件，Helm 会根据values.yaml的参数进行渲染，然后在 Kubernetes 集群中创建出来。

你可以根据自己的需要，在这个`templates`目录下，添加和更改 yaml 文件。

你这样创建好了一个 Chart 包后，如果想要更改其中一些参数，可以将它们放到其他的文件里面，比如 myvalues.yaml 文件。

然后通过`helm install`命令，你就可以在 Kubernetes 集群里面部署 hello-world 这个 Chart 包：

```sh
$ helm install -f myvalues.yaml hello-world ./hello-world
```

或者，你可以通过添加 Repository 直接使用别人的 Chart，使用helm search来查找自己想要的 Chart：

```sh
$ helm repo add brigade https://brigadecore.github.io/charts
"brigade" has been added to your repositories
$ helm search repo brigade
NAME                          CHART VERSION APP VERSION DESCRIPTION
brigade/brigade               1.3.2         v1.2.1      Brigade provides event-driven scripting of Kube...
brigade/brigade-github-app    0.4.1         v0.2.1      The Brigade GitHub App, an advanced gateway for...
brigade/brigade-github-oauth  0.2.0         v0.20.0     The legacy OAuth GitHub Gateway for Brigade
brigade/brigade-k8s-gateway   0.1.0                     A Helm chart for Kubernetes
brigade/brigade-project       1.0.0         v1.0.0      Create a Brigade project
brigade/kashti                0.4.0         v0.4.0      A Helm chart for Kubernetes
```

更多 Helm 的使用命令，可以参考 [官方的使用文档](https://helm.sh/docs/intro/using_helm/)，使用起来非常简单。



### 写在最后

目前 Helm 是 CNCF 基金会旗下已经“毕业”的独立的项目。它简化了 Kubernetes 应用的部署和管理，大大提高了效率，越来越多的人在生产环境中使用 Helm 来部署和管理应用。







# 守护神：业务的日志与监控



## 13 | 服务守护进程：如何在 Kubernetes 中运行 DaemonSet 守护进程？



通过前面课程的学习，我们对 Kubernetes 中一些常见工作负载已经有所了解。比如无状态工作负载 Dployment 可以帮助我们运行指定数目的服务副本，并维护其状态，而对于有状态服务来说，我们同样可以采用 StatefulSet 来做到这一点。

但是，在实际使用的时候，有些场景，比如监控各个节点的状态，使用 Deployment 或者 StatefulSet 都无法满足我们的需求，因为这个时候我们可能会有以下这些需求。

1. 希望每个节点上都可以运行一个副本，且只运行一个副本。虽然通过调整 `spec.replicas` 的数值，可以使之等于节点数目，再配合一些调度策略可以实现这一点。但是如果节点数目发生了变化呢？

2. 希望在新节点上也快速拉起副本。比如集群扩容，这个时候会有一些新节点加入进来，如何立即感知到这些节点，并在上面部署新的副本。

3. 希望节点下线的时候，对应的 Pod 也可以被删除。

4. ……

Kubernetes 提供的 DaemonSet 就可以完美地解决上述问题，其主要目的就是可以在集群内的每个节点上（或者指定的一堆节点上）都只运行一个副本，即 Pod 和 Node 是一一对应的关系。DaemonSet 会结合节点的情况来帮助你管理这些 Pod，见下面的拓扑结构：

![Lark20201009-105149.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F9_0FqACmcBAABg43Rbbow934.png)

今天我们就来学习一下 DaemonSet，先来看看其主要的使用场景。



### DaemonSet 的使用场景

跟 Deployment 和 StatefulSet 一样，DaemonSet 也是一种工作负载，可以管理一些 Pod。 通常来说，主要有以下的用法：

- 监控数据收集，比如可以将节点信息收集上报给 Prometheus；

- 日志的收集、轮转和清理；

- 监控节点状态，比如运行 node-problem-detector 来监测节点的状态，并上报给 APIServer；

- 负责在每个节点上网络、存储等组件的运行，比如 glusterd、ceph、flannel 等；

现在我们来尝试部署一个 DaemonSet。



### 部署你的第一个 DaemonSet

这里是一个 DaemonSet 的 YAML 文件：

```yaml
apiVersion: apps/v1 # 这个地方已经不是 extension/v1beta1 了，在1.16版本已经废弃了，请使用 apps/v1
kind: DaemonSet # 这个是类型名
metadata:
  name: fluentd-elasticsearch # 对象名
  namespace: kube-system # 所属的命名空间
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      restartPolicy: Always
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

在这个 YAML 文件里，我们有一个地方需要特别注意，就 restartPolicy这个字段，它是缺省字段，默认值是 Always。而且如果你想显式地去设置，你也只能设置为 Always。

其他的配置和写法，跟我们之前了解的 Deployment 和 StatefulSet 是类似的。

我们将上面 YAML 文件保存到本地的 fluentd-elasticsearch-ds.yaml 中，然后用 kubectl apply创建出来：

```sh
$ kubectl apply -f fluentd-elasticsearch-ds.yaml
daemonset.apps/fluentd-elasticsearch created
```

创建好后，我们来查看这个 DaemonSet 关联 Pod 的情况：

```sh
$ kubectl get pod -n kube-system -l name=fluentd-elasticsearch
NAME                          READY   STATUS    RESTARTS   AGE
fluentd-elasticsearch-m9zjb   1/1     Running   0          85s
```

可以看到，集群中只有一个 Pod 被创建了出来。我们再来看看集群中有多少个节点：

```sh
$ kubectl get node
NAME             STATUS   ROLES    AGE   VERSION
docker-desktop   Ready    master   22d   v1.16.6-beta.0
```

由于目前集群中就只有一个节点，所以 Kubernetes 只为这一个节点生成了 Pod。我们来查看下该 DaemonSet 的整体状态：

```sh
$ kubectl get ds -n kube-system fluentd-elasticsearch
NAME                    DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
fluentd-elasticsearch   1         1         0       1            0           <none>          18
```

在 kubectl 使用的时候，我们常常将DaemonSet 缩写成 `ds`。这里输出列表的含义如下：
DESIRED 代表该 DaemonSet 期望要创建的 Pod 个数，即我们需要几个，它跟节点数目息息相关；

- `CURRENT`代表当前已经存在的 Pod 个数；

- `READY` 代表目前已就绪的 Pod 个数；

- `UP-TO-DATE` 代表最新创建的个数；

- `AVAILABLE` 代表目前可用的 Pod个数；

- `NODE SELECTOR`表示节点选择标签，这个在 DaemonSet 中非常有用。有时候我们只希望在部分节点上运行一些 Pod，比如我们只节点上带有 app=logging-node 的节点上运行一些服务，就可以通过这个标签选择器来实现。



### 限定 DaemonSet 运行的节点

现在我们来看看如何限定一个 DaemonSet，让其只在某些节点上运行，比如只在带有 `app=logging-node` 的节点上运行，可以看这张图：

![1.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F9xnpWAKFwRAAB-Lq1l0YU648.png)

此时，我们就可以通过 DaemonSet 的 selector 来实现：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: my-ds
  namespace: demo
  Labels:
    key: value
spec:
  selector: # 通过这个 selector，我们就可以让 daemonset pod 只在指定的节点上运行
    matchLabels:
      app: logging-node
  template:
    metadata:
      labels:
        name: my-daemonset-container
    ...
```

这样一个 DaemonSet 就会只匹配集群中所有带有标签 `app=logging-node` 的节点，并分别在这些节点上运行一个 Pod。

如果集群中节点的标签发生了变化，这个时候`DaemonSetsController`会立刻为新匹配上的节点创建 Pod，同时删除不匹配的节点上的 Pod。

我们再来看如何调度。



### DaemonSet 的 Pod 是如何被调度的

早期 Kubernetes 是通过`DaemonSetsController`（在 kube- controller-manager 组件内以 goroutine 方式运行）调度 DaemonSet 管理的 Pod，这些 Pod 在创建的时候，就在 Pod 的 spec 中提前指定了节点名称，即 `spec.nodeName`。这些 Pod 由于指定了节点，所以不会经过默认调度器进行调度，这就导致了一些问题。

- 不一致的 Pod 行为：其他的 Pod 都是通过默认调度器进行调度的，初始状态都是 Pending（等待调度），而 DaemonSet 的这些 Pod 的起始状态却不是 Pending。
- `DaemonSetsController` 并不会感知到节点的资源变化；
- 默认调度器的一些高级特性需要在 、DaemonSetsController 中二次实现。
- 多组件负责调度会导致 Pod 抢占等功能实现起来非常困难；
- ...

你可以看看设计文档 [Schedule DaemonSet Pods by default scheduler, not DaemonSet controller](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/scheduling/schedule-DS-pod-by-scheduler.md) ，它详细介绍了`DaemonSetsController` 调度时遇到的各种问题，并给出了详细的解决方案。

简单来说，DaemonSet Pod 依然由 `DaemonSetsController` 进行创建，但是不预先指定`spec.nodeName`了，而通过节点的亲和性，交由默认调度器进行调度。

我们回过头来看看上面`fluentd-elasticsearch`这个 DaemonSet 创建的 Pod：

```yaml
$ kubectl get pod -n kube-system fluentd-elasticsearch-m9zjb -o yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2020-09-25T12:01:31Z"
  generateName: fluentd-elasticsearch-
  labels:
    controller-revision-hash: 5b5b9c8855
    name: fluentd-elasticsearch
    pod-template-generation: "1"
  name: fluentd-elasticsearch-m9zjb
  namespace: kube-system
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: DaemonSet
    name: fluentd-elasticsearch
    uid: 33dc29aa-60b0-4486-8645-731daa85f25d
  ...
spec:
  affinity:
    nodeAffinity: # daemonset 就是利用了 nodeAffinity 的能力
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchFields:
          - key: metadata.name
            operator: In
            values:
            - docker-desktop
  containers:
  - image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
    imagePullPolicy: IfNotPresent
    name: fluentd-elasticsearch
    ...
  ...
```

`DaemonSetsController` 创建的这个 Pod，自动添加了 `spec.affinity.nodeAffinity`指定节点的名称，替换以前`spec.nodeName` 的方式。

除了这个`nodeAffinity`之外，`DaemonSetsController`还会自动加一些 Toleration 到 Pod 中。有兴趣可以查看 [这个清单](https://kubernetes.io/zh/docs/concepts/workloads/controllers/daemonset/#taint-and-toleration)。

接下来，我们看看它的升级方法。



### 如何升级一个 Daemonset

升级一个 DaemonSet 其实非常简单，你可以通过kubectl edit 直接编辑对应的 DaemonSet 对象：

```sh
$ kubectl edit ds -n kube-system fluentd-elasticsearch
```

也可以直接在`fluentd-elasticsearch-ds.yaml` 中修改，然后使用`kubectl apply` ：

```sh
$ kubectl apply -f fluentd-elasticsearch-ds.yaml
```

那么修改后，DaemonSet 的这些 Pod 又是如何更新的呢？

DaemonSet 中定义了两种更新策略。

**第一种是 OnDelete**，顾名思义，当指定这种策略时，我们只有先手动删除原有的 Pod 才会触发新的 DaemonSet Pod 的创建。否则，不论你怎么修改 DaemonSet ，都不会触发新的 Pod 生成。

**第二种是 RollingUpdate**，这是默认的更新策略，使用这种策略可以实现滚动更新，同时你还可以更精细地控制更新的范围，比如通过 `maxUnavailable` 为 1 来控制更新的速度（你也可以理解为更新时设置的步长），这表示每次只会先删除 1 个 Pod，待其重新创建并Ready 后，再更新同样的方法更新其他的 Pod。在更新期间，每个节点上最多只能有 DaemonSet 的一个 Pod。

下面这个 YAML 是一个使用了`RollingUpdate`更新策略的 DaemonSet ：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  updateStrategy: # 这里指定了更新策略
    type: RollingUpdate # 进行滚动更新
    rollingUpdate:
      maxUnavailable: 1 # 这是默认的值
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```



### 写在最后

从业务状态的角度来看，Kubernetes 使用 Deployment 和 StatefulSet 来分别支持无状态服务和有状态服务。而 DaemonSet 则从另外的一个角度来为集群中的所有节点提供基础服务，比如网络、存储等。

通过 DaemonSet，我们可以确保在所有满足条件的 Node 上只运行一个 Pod 实例，通常使用于日志收集、监控、网络等场景。Kubernetes 的组件 kube-prox 有时也可以借助 DaemonSet 拉起，这样每个节点上就会只运行一个 kube-proxy 的实例。Kubeadm 拉起的集群就是这样部署 kube-proxy 的。

同时 DaemonSet 也帮助我们解决了集群中节点动态变化时业务实例的部署和运维能力，比如扩容、缩容、节点宕机等场景。





## 14 | 日志采集：如何在 Kubernetes 中做日志收集与管理？



说到日志，你应该不陌生。日志中不仅记录了代码运行的实时轨迹，往往还包含着一些关键的数据、错误信息，等等。日志方便我们进行分析统计及监控告警，尤其是在后期问题排查的时候，我们通过日志可以很方便地定位问题、现场复现及问题修复。日志也是做可观测性（Observability）必不可少的一部分。

因此在使用 Kubernetes 的过程中，对应的日志收集也是我们不得不考虑的问题。我们需要日志去了解集群内部的运行状况。

我们先来看看 Kubernetes 的日志收集和以往的日志收集有什么差别，以及为什么我们需要为 Kubernetes 的日志收集单独设计方案。



### Kubernetes 中的日志收集 VS 传统日志收集

对于传统的应用来说，它们大都都是直接运行在宿主机上的，会将日志直接写入本地的文件中或者由 systemd-journald 直接管理。在做日志收集的时候，只需要访问这些日志所在的目录即可。此类日志系统解决方案非常多，也相对比较成熟，在这里就不再过多说明。我们重点来看看 Kubernetes 的日志系统建设问题。

在 Kubernetes 中，日志采集相比传统虚拟机、物理机方式要复杂很多。

首先，日志的形式非常多样化。日志需求主要集中在如下三个部分：

- 系统各组件的日志，比如 Kubernetes 自身各大组件的日志（包括 kubelet、kube-proxy 等），容器运行时的日志（比如 Docker）；

- 以容器化方式运行的应用程序自身的日志，比如 Nginx、Tomcat 的运行日志；

- Kubernetes 内部各种 Event（事件），比如通过`kubebctl create`创建一个 Pod 后，可以通过`kubectl describe pod pod-xxx`命令查看到的这个 Pod 的 Event 信息。

其次，集群环境时刻在动态变化。我们都知道 Pod “用完即焚”，Pod 销毁后日志也会一同被删除。但是这个时候我们仍然希望可以看到具体的日志，用于查看和分析业务的运行情况，以及帮助我们发现出容器异常的原因。

同时新的 Pod 可能随时会“飘”到别的节点上重新“生长”出来，我们无法提前预知具体是哪个节点。而且 Kubernetes 的节点也会存在宕机等异常情况。所以说，Kubernetes 的日志系统在设计的时候，必须得独立于节点和 Pod 的生命周期，且保证日志数据可以实时采集到服务端，即完全独立于 Kubernetes 系统，使用自己的后端存储和查询工具。

再次，日志规模会越来越大。很多人在 Kubernetes 中喜欢使用 hostpath 来保存 Pod 的日志，并且不做日志轮转（可以配置 Docker 的`log-opts`来 [设置容器的日志轮转](https://docs.docker.com/config/containers/logging/configure/#configure-the-default-logging-driver) ），这很容易将宿主机的磁盘“打爆”。这里你是不是觉得如果做了轮转，磁盘打爆的问题就可以完美解决了？

其实虽有所缓解，并不会让你安全无忧。虽说日志轮转可以有效减少日志的文件大小，但是你会丢失掉不少日志，后续想要分析和排查问题时就无从下手了。想想看如果容器内的应用出现异常并疯狂报错，这个时候又有大量的并发请求，那么日志就会急剧增多。

配置了日志轮转，会让你丢失很多重要的上下文信息。如果没有配置日志轮转，这些日志很快就会将磁盘打爆。还有可能引发该节点的 Kubelet 异常，导致该节点上的 Pod 被驱逐。我们在一些生产实践中，就遇到过这种情况。同时当 Pod 被删除后，这些 hostpath 的文件并不会被及时删除，会继续占用很多磁盘空间。此外，随着业务逐渐增长，在这个节点上运行过的 Pod 也会变多，这就会残留大量的日志文件。

此外，日志非常分散且种类多变。单纯查找一个应用的日志，就需要查看其关联的分散在各个节点上的各个 Pod 的日志。在出现紧急情况需要排查的时候，这种方式极其低效，会严重影响到问题修复和服务恢复。如果这个应用还通过 Ingress 对外暴露服务，并使用了 Service Mesh 等，那么此时做日志收集就更复杂了。

随着在 Kubernetes 上落地越来越多的微服务，各个服务之间的依赖也越来越多。这个时候各个维度的日志关联也是一个非常困难的问题。那么，下面我们就来看看如何对 Kubernetes 做日志收集。



### 几种常见的 Kubernetes 日志收集架构

Kubernetes 集群本身其实并没有提供日志收集的解决方案，但依赖 Kubernetes 自身提供的各项能力，可以帮助我们解决日志收集的诉求。根据上面提到的三大基本日志需求，一般来说我们有如下有三种方案来做日志收集：

1. 直接在应用程序中将日志信息推送到采集后端；

2. 在节点上运行一个 Agent 来采集节点级别的日志；

3. 在应用的 Pod 内使用一个 Sidecar 容器来收集应用日志。

我们来分别看看这三种方式。

先来看**直接在应用程序中将日志信息推送到采集后端，即Pod 内的应用直接将日志写到后端的日志中心。**

![image (2).png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F9zCLCALLfHAAA7edHbxKE531.png)

通常有两种做法，一个就是应用程序通过对应的日志 SDK 进行接入，不过这种做法一般不推荐，和应用本身耦合太严重，也不方便后续对接其他的日志系统。

还有一种做法就是通过容器运行时提供的 Logging Driver 来实现。以最常用的 Docker 为例，目前已经支持了[十多种 Logging Driver](https://docs.docker.com/config/containers/logging/configure/#supported-logging-drivers)。比如你可以配置为`fluentd`，这个时候 Docker 就会将容器的标准输出日志（stdout、stderr）直接写到`fluentd`中。你也可以设置成`awslogs`，这样就会直接将日志写到 Amazon CloudWatch Logs 中。但是在和 Kubernetes 一起使用的时候，使用较多的是`json-file`，这也是 Docker 默认的 Logging Driver。

```sh
$ docker info |grep 'Logging Driver'
Logging Driver: json-file
```

你经常使用的`kubectl logs`就是基于`json-flle`这种 Logging Driver 来实现的，目前 Kubernetes 也只支持`json-flle`这一种 Logging Driver。

所以在 Kubernetes 的这套体系中，直接将日志写到后端日志采集系统中去，并不是特别好的做法。

我们来看第二种方法，**在节点上运行一个 Agent 来采集节点级别的日志**。如下图所示，我们可以在每一个 Kubernetes Node 上都部署一个 Agent，该 Agent 负责对该节点上运行的所有容器进行日志收集，并推送到后端的日志存储系统里。这个 Agent 通常需要可以访问到宿主机上的指定目录，比如`/var/lib/docker/containers/`。

![image (3).png](D:\学习资料\笔记\k8s\k8s图\CgqCHl9zCNiAEVCiAABrnSxfaQg197.png)

由于这样的 Agent 需要在每个 Node 上都运行，因此我们都是通过 上节课讲到的`DaemonSet`的方式来部署的。对于 Kubernetes 集群来说，这种使用节点级的`DaemonSet`日志代理是最常用，也是最被推荐的方式，不仅可以节约资源，而且对于应用来说也是无侵入的。

但是这种方式也有个缺点，就是只适应于容器内应用日志是标准输出的场景，即应用把日志输出到`stdout`和`stderr`。

最后来看**通过 Sidecar 来收集容器日志**。 在 Pod 里面，容器的输出日志可以是stdout、stderr和日志文件。那么基于这三种形式，我们可以借助于 Sidecar 容器将基于文件的日志来帮助我们。

- 通过Sidecar 容器读取日志文件，并定向到自己的标准输出。如下图所示，这里streaming container就是一个 Sidecar 容器，可以将app-container的日志文件重新定向到自己的标准输出。同时还可以归并多个日志文件。而且这里也可以使用多个 Sidecar 容器，你可以参考[这个例子](https://github.com/kubernetes/website/blob/master/content/en/examples/admin/logging/two-files-counter-pod-streaming-sidecar.yaml)。

![image (4).png](D:\学习资料\笔记\k8s\k8s图\CgqCHl9zCOWAI1-SAAB3nPjxdMA390.png)

- Sidecar容器运行一个日志代理，配置该日志代理以便从应用容器收集日志。这种方式就解决了我们上面方案一的问题，将日志处理部分和应用程序本身进行了解耦，可以方便切换到其他的日志系统中。可以参考这个 [使用 fluentd 的例子](https://github.com/kubernetes/website/blob/master/content/en/examples/admin/logging/two-files-counter-pod-agent-sidecar.yaml)。

![image (5).png](D:\学习资料\笔记\k8s\k8s图\CgqCHl9zCOuAFQm_AABLBPDcBz4058.png)

可以看到,通过 Sidecar 的方式来收集日志，会增加额外的开销。在集群规模较小的情况下可以忽略不计,但是对于大规模集群来说，这些开销还是不可忽略的。

那么，上面的这几套方案，在实际使用的时候，又该如何选择呢？社区又有什么推荐方案呢？



### 基于 Fluentd + ElasticSearch 的日志收集方案

Kubernetes 社区官方推荐的方案是使用 Fluentd+ElasticSearch+Kibana 进行日志的收集和管理，通过Fluentd将日志导入到Elasticsearch中，用户可以通过Kibana来查看到所有的日志。

![image (6).png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F9zCPSATIwRAAA_ddgWuO0667.png)

关于 ElasticSearch 和 Kibana 如何部署，在此就不过多地介绍了，你可以通过添加 helm 的 repo即`helm repo add elastic https://helm.elastic.co`，然后通过 helm来快速地自行部署，相关的 Chart 见https://github.com/elastic/helm-charts。

现在我们就来看看这套方案的几个技术点。

Fluentd 提供了强大的日志统一接入能力，同时内置了插件，可以对接 ElasticSearch。这里 Fluentd 主要有如下四个配置：

- `fluent.conf`这个文件主要是用来设置一些地址，比如 ElasticSearch 的地址等；.

- `kubernetes.conf`这个文件记录了与 Kubernetes 相关的配置，比如 Kubernetes 各组件的日

  志配置、容器的日志收集规则，等等；

- `prometheus.conf`这个文件定义了 Prometheus 的地址，方便 Fluentd 暴露自己的统计指标；

- `systemd.conf`这个文件可以配置 Fluentd 通过 systemd-journal 来收集哪些服务的日志，比

  如 Docker 的日志、Kubelet 的日志等。

上面的这些配置，都默认内置到了 [fluent/fluentd-kubernetes-daemonset](https://hub.docker.com/r/fluent/fluentd-kubernetes-daemonset/tags?page=1&name=elasticsearch) 的镜像中，你可以使用官方的默认配置。如果你想要定制化更改一些，可以参照 [这份默认示例配置](https://github.com/fluent/fluentd-kubernetes-daemonset/tree/master/docker-image/v1.11/debian-elasticsearch7/conf)。

如下的 YAML 是一段 fluentd 的 DaemonSet 定义，源自 [这里](https://github.com/fluent/fluentd-kubernetes-daemonset/blob/master/fluentd-daemonset-elasticsearch.yaml)：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
    version: v1
spec:
  selector:
    matchLabels:
      k8s-app: fluentd-logging
      version: v1
  template:
    metadata:
      labels:
        k8s-app: fluentd-logging
        version: v1
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:v1-debian-elasticsearch
        env:
          - name:  FLUENT_ELASTICSEARCH_HOST
            value: "elasticsearch-logging"
          - name:  FLUENT_ELASTICSEARCH_PORT
            value: "9200"
          - name: FLUENT_ELASTICSEARCH_SCHEME
            value: "http"
          # Option to configure elasticsearch plugin with self signed certs
          # ================================================================
          - name: FLUENT_ELASTICSEARCH_SSL_VERIFY
            value: "true"
          # Option to configure elasticsearch plugin with tls
          # ================================================================
          - name: FLUENT_ELASTICSEARCH_SSL_VERSION
            value: "TLSv1_2"
          # X-Pack Authentication
          # =====================
          - name: FLUENT_ELASTICSEARCH_USER
            value: "elastic"
          - name: FLUENT_ELASTICSEARCH_PASSWORD
            value: "changeme"
          # Logz.io Authentication
          # ======================
          - name: LOGZIO_TOKEN
            value: "ThisIsASuperLongToken"
          - name: LOGZIO_LOGTYPE
            value: "kubernetes"
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```



### 写在最后

在实际采集日志的时候，你可以根据自己的场景和集群规模选择适用的方案。或者也可以将上面的这几种方案进行合理地组合。





## 15 | Prometheus：Kubernetes 怎样实现自动化服务监控告警？



通过之前的学习，我们已经对 Kubernetes 有了一定的理解，也知道如何在 Kubernetes 中部署自己的业务系统。

Kubernetes 强大的能力让我们非常方便地使用容器部署业务。Kubernetes 自带的副本保持能力，可以避免部署的业务系统出现单点故障，提高可用性。各种探针也可以帮助我们对运行中的容器进行一些简单的周期性检查。

但是要想知道业务真实运行的一些指标，比如并发请求数、IOPS、线程数，等等，就需要通过监控系统来获取了。此外，我们还需要监控系统支持将不同数据源的指标进行收集、分类、汇总聚合以及可视化的展示等。

今天我们就来聊聊 Kubernetes 的监控体系，以及目前主流的监控方案。首先来看看 Kubernetes 监控系统所遇到的挑战。



### Kubernetes 监控系统面临的挑战

以往的业务应用在部署的时候，大都固定在某几台节点上，各种指标监控起来也很方便。通过在每个节点上部署对应的监控 agent，用来收集对应的监控指标。

而到了Kubernetes 体系中，监控的问题开始变得复杂起来。

首先，业务应用的 Pod 会在集群中“漂移”且捉摸不定。你无法预知 Pod 会在哪些节点上运行，而且运行时间有可能也只是短暂的，随时可能会被新的、健康的 Pod 所取代。而且业务还有需要扩缩容的场景，你无法提前预知哪些 Pod 会被销毁掉。这里还有两点非常值得关注，那就是：

1. 除了 StatefulSet 管理的 Pod 名字是固定不变的以外，通过 Deployment/ReplicaSet/DaemonSet 等工作负载管理的 Pod 的名字是随机的；
2. Pod 的 IP 不是固定不变的。

这样的话，如果我们还是通过以往固定 IP 或者固定域名的方式去拿监控数据，这就不太可行了。

其次，转向了微服务架构以后，就不可避免地出现“碎片化”的问题。从以前的单体架构，变成微服务架构，模块和功能被逐步拆解成一组小的、松耦合的服务。各个小的服务可以单独部署，这也给监控带来了麻烦。

再次，通过前面的学习，你已经知道Kubernetes 通过 label 和 annotation 来管理 Pod ，这一全新的理念对以往传统的监控方式带来了冲击。所以评判一个监控方案是不是能够完美适配 Kubernetes 体系的标准就是，它有没有采用 label 和 annotation 这套思路来收集监控指标。

最后，Kubernetes 强大的能力，让我们在部署应用的时候更加随心所欲。Kubernetes可以对接各个 Cloud Provider （云服务提供商），比如 AWS、GCP、阿里云、VMWare，等等。这就意味着我们部署业务的时候，可以选择让 Pod 运行在公有云、私有云或者混合云中。这也给监控体系带来前所未有的挑战。

尽管存在这么多的挑战，我们还是有很多方案可以采用的。现在我们先来看看 Kubernetes 中几种常见的监控数据类别。



### 常见的监控数据类别

在 Kubernetes 中，监控数据大致分为两大类。

一种是**应用级别的数据**，主要帮助我们了解应用自身的一些监控指标。常见的应用级别的监控数据包括：CPU 使用率、内存使用率、磁盘空间、网络延迟、并发请求数、请求失败率、线程数，等等。其中并发请求数这类指标，就需要应用自己暴露出监控指标的接口。

除了应用自身的监控数据外，另外一种就是**集群级别的数据**。这个数据非常重要，它可以帮助我们时刻了解集群自身的运行状态。通过监控各个 Kubelet 所在节点的 CPU、Memory、网络吞吐量等指标，方便我们及时了解 Kubelet 的负载情况。还有 Kubernetes 各个组件的状态，比如 ETCD、kube-scheduler、kube-controller-manager、coredns 等。

在集群运行过程中，Kubernetes 产生的各种 Event 数据，以及 Deployment/StatefulSet/DaemonSet/Pod 等的状态、资源请求、调度和 API 延迟等数据指标也是必须要收集的。

下面我们先来看看如何来收集这些监控数据。



### 常见的监控数据采集工具

Kubernetes 集群的数据采集工具主要有以下几种工具。

#### 1. cAdvisor

cAdvisor 是 Google 专门为容器资源监控和性能分析而开发的开源工具，且不支持跨主机的数据监控，需要将采集的数据存到外部数据库中，比如 influxdb，再通过图形展示工具进行可视化。一开始 cAdvisor 也内置到了 Kubelet 中，不需要额外部署。但是社区从 v1.10 版本开始，废弃了 cAdvisor 的端口，即默认不开启，并于 v1.12 版本正式移除，转而使用 Kubelet 自身的`/metrics`接口进行暴露。

#### 2. Heapster

Heapster 是一个比较早期的方案，通过 cAdvisor 来收集汇总各种性能数据，是做自动伸缩（Autoscale）所依赖的组件。你在网上查找 Kubernetes 各种监控方案的时候可能还会见到。但是现在 Heapster 已经被社区废弃掉了，后续版本中推荐你使用 metrics-server 代替。我们在下一节中，会来介绍如何通过 metrics-server 来实现自动扩缩容，控制资源水位。

#### 3. metrics-server

 是一个集群范围内的监控数据聚合工具，用来替代 Heapster 。

#### 4. Kube-state-metrics

[Kube-state-metrics](https://github.com/kubernetes/kube-state-metrics) 可以监听 kube-apiserver 中的数据，并生成有关资源对象的新的状态指标，比如 Deployment、Node、Pod。这也是 kube-state-metrics 和 metrics-server 的 [最大区别](https://github.com/kubernetes/kube-state-metrics#kube-state-metrics-vs-metrics-server)。

#### 5. node-exporter

[node-exporter](https://github.com/prometheus/node_exporter) 是 Prometheus 的一个 exporter，可以帮助我们获取到节点级别的监控指标，比如 CPU、Memory、磁盘空间、网络流量，等等。

当然，还有很多类似的工具可以采集数据，在此不一一列举。有了这么多数据采集工具可以帮助我们采集数据，**我们就可以根据自己的需要自由组合**，形成我们的监控方案体系。

在 Kubernetes 1.12 版本以后，我们通常选择 Prometheus + Grafana 来搭建我们的监控体系，这也是社区推荐的方式。



### Prometheus + Grafana

Prometheus 功能非常强大，于 2012 年由 SoundCloud  公司开发的开源项目，并于 2016 年加入 CNCF，成为继 Kubernetes 之后第二个被 CNCF 接管的项目，于 2018 年8月份毕业，这意味着 Prometheus 具备了一定的成熟度和稳定性，我们可以放心地在生产环境中使用，也可以集成到我们自建的监控体系中（很多厂商的自建监控体系就是这么打造的）。

Prometheus 在早期开发的时候，参考了Google 内部 Borg 的监控实现 Borgmon。所以非常合适和源自 Borg 的 Kubernetes 搭配使用。

我们来看一个例子，如下图是 Prometheus + Grafana 组成的监控方案。

![image.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F-FUAaABRE2AAFto-2ifvc966.png)

最左边的 Prometheus targets 就是我们要采集的数据。Retrieval 负责采集这些数据，并同时支持 Push 和 Pull 两种采集方式。

- Pull 模式是最常用的拉取式数据采集方式，大部分使用数据都是通过这种方式采集的。只需要在应用里面实现一个`/metrics`接口，然后把这个接口写到Prometheus 的配置文件即可。你可以参照 [这份官方文档](https://prometheus.io/docs/guides/go-application/)，学习如何在自己的应用中添加监控指标。

- Push 模式则是由各个数据采集目标主动向 PushGateway 推送指标，再由服务器端拉取。

为了保证数据持久化，Prometheus 采集到的这些监控数据会通过时间序列数据库 TSDB 进行存储，支持以时间为索引进行存储。TSDB 在提供存储的同时，还提供了非常强大的数据查询、数据处理功能，这也是告警系统以及可视化页面的基础。

此外，Prometheus 还内置了告警模块 Alertmanager，它支持多种告警方式，比如 pagerduty、邮件等，还有对告警进行动态分组等功能。

最后这些监控数据通过 Grafana 进行多维度的可视化展示，方便通过大盘进行分析。



### 写在最后

Kubernetes 的副本保持及自愈能力，可以尽可能地保持应用程序的运行，但这并不意味着我们就不需要了解应用程序运行的情况了。因此，当我们开始将业务部署到 Kubernetes 中时，还需要去部署一套监控系统，来帮助我们了解到业务内部运行的一些细节情况，同时我们也需要监控 Kubernetes 系统本身，帮助我们了解整个集群当前的形态，指导我们作出决策。Prometheus 和 Grafana 是目前使用较广泛的监控组合方案，很多监控方案也都是基于此做了二次开发和定制。





## 16 | 迎战流量峰值：Kubernetes 怎样控制业务的资源水位？



通过前面的学习，相信你已经见识到了 Kubernetes 的强大能力，它能帮你轻松管理大规模的容器服务，尤其是面对复杂的环境时，比如节点异常、容器异常退出等，Kubernetes 内部的 Service、Deployment 会动态地进行调整，比如增加新的副本、关联新的 Pod 等。

当然 Kubernetes 的这种自动伸缩能力可不止于此。

我们今天就来看看如果利用这些伸缩能力，帮我们在应对大促这样的大流量活动时，控制业务的资源水位，以提供稳定的服务，避免容器的负载过高被打爆，出现流量下跌、业务抖动等情况，从而引发业务故障。



### Pod 水平自动伸缩（Horizontal Pod Autoscaler，HPA）

一般来说，我们在遇到这种大流量的场景时，映入我们脑海中的一个想法就是水平扩展，即增加一些实例来分担流量压力。像 Deployment 这种支持多副本的工作负载，我们就可以通过调整`spec.replicas`来增加或减少副本数，从而改变整体的业务水位满足我们的需求，即整体负载高时就增加一些实例，负载低就适当减少一些实例来节省资源。

当然人为不断地调整`spec.replicas`的数值显然是不太现实的，[HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) 可以根据应用的 CPU 利用率等水位信息，动态地增加或者减少 Pod 副本数量，帮你自动化地完成这一调和过程。

HPA 大多数是用来自动扩缩（Scale）一些无状态的应用负载，比如 Deployment，或者你自己定义的其他类型的无状态工作负载。当然你也可以用来扩缩有状态的应用负载，比如 StatefulSet。

我们来看下面这张图，它描述了 HPA 通过动态地调整Deployment的副本数，从而控制 Pod 的数量。

![Drawing 0.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl-H4QiAJarCAAQ5OqhzE0c569.png)

在使用 HPA 的时候，你需要提前部署好 [metrics-server](https://github.com/kubernetes-sigs/metrics-server)，可以通过 `kubectl apply -f`一键部署完成（如果你想了解更多关于 metrics-server 的部署，可以参考 [官方文档](https://github.com/kubernetes-sigs/metrics-server#deployment)）。

```sh
$ kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.7/components.yaml
clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader created
clusterrolebinding.rbac.authorization.k8s.io/metrics-server:system:auth-delegator created
rolebinding.rbac.authorization.k8s.io/metrics-server-auth-reader created
apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io created
serviceaccount/metrics-server created
deployment.apps/metrics-server created
service/metrics-server created
clusterrole.rbac.authorization.k8s.io/system:metrics-server created
clusterrolebinding.rbac.authorization.k8s.io/system:metrics-server created
```

我们来查看下`metrics-server`对应的 Deployment 的状态，如下图它已经处于 Ready 状态。

```sh
$ kubectl get deploy -n kube-system metrics-server
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
metrics-server   1/1     1            1           26s
```

你可以通过`kubectl logs`来查看对应 Pod 的日志，来确定其是否正常工作。如果有异常，可以参考 [这里](https://github.com/kubernetes-sigs/metrics-server#configuration) 调整部署参数。

现在我们来看看如何来使用 HPA。

首先我们部署一个 Deployment，其 YAML 配置如下（你可以保存为`nginx-deploy-hpa.yaml`文件）：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: demo
spec:
  selector:
    matchLabels:
      app: nginx
      usage: hpa
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
        usage: hpa
    spec:
      containers:
      - name: nginx
        image: nginx:1.19.2
        ports:
        - containerPort: 80
        resources:
          requests: # 这里我们把quota设置得小一点，方便做压力测试
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
```

下面我们通过 kubectl 来创建这个 Deployment：

```sh
$ kubectl create -f nginx-deploy-hpa.yaml
deployment.apps/nginx-deployment created
```

现在我们来查看一下该 Deployment 的状态，通过下面几条命令可以看到它已经 ready ：

```sh
$ kubectl get deploy -n demo
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   1/1     1            1           4s
$ kubectl get pod -n demo
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-6d4b885966-zngnd   1/1     Running   0          8s
```

可以看到，我们创建的 Deployment 运行成功。
现在我们来创建一个 Service ，来关联这个 Deployment，我们后续就可以通过这个 Service 来访问 Deployment 的各个 Pod 实例了。通过如下命令，我们就可以快速创建一个对应的 Service 出来，当然你也可以通过 YAML 文件来创建。

```sh
$ kubectl expose deployment/nginx-deployment -n demo
service/nginx-deployment exposed
```

我们用`kubectl get`来查看一下这个 Service 和对应的 Endpoints 对象：

```sh
$ kubectl get svc -n demo
NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
nginx-deployment   ClusterIP   10.109.93.199   <none>        80/TCP    4s
$ kubectl get endpoints -n demo
NAME               ENDPOINTS                     AGE
nginx-deployment   10.1.0.163:80,10.1.0.164:80   12s
```

后面我们就可以通过访问这个 Service 的10.109.93.199地址来访问该服务。

现在我们来创建一个 HPA 对象。通过`kubectl autoscale`的命令，我们就可以将 Deployment 的副本数控制在 1~10 之间，CPU 利用率保持在 50% 以下，即当该 Deployment 所关联的 Pod 的平均 CPU 利用率超过 50% 时，就增加副本数，直到小于该阈值。当平均 CPU 利用率低于 50% 时，就减少副本数：

```sh
$ kubectl autoscale deploy nginx-deployment -n demo --cpu-percent=50 --min=1 --max=10
horizontalpodautoscaler.autoscaling/nginx-deployment autoscaled
```

我们来查看一下刚才创建出来的 HPA 对象：

```sh
$ kubectl get hpa -n demo
NAME               REFERENCE                     TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
nginx-deployment   Deployment/nginx-deployment   0%/50%          1         10        0      
```

在 kube-controller-manager 中有对应的 HPAController 负责算出合适的副本数，它会根据 metrics-server 上报的对应 Pod metrics 进行计算，具体的 [算法细节](https://kubernetes.io/zh/docs/tasks/run-application/horizontal-pod-autoscale/#algorithm-details) 可以通过官方文档来了解。

好了，到现在，一切准备就绪，我们开始见证 HPA 的魔力！

我们现在新开两个终端，分别运行命令`kubectl get deploy -n demo -w`和`kubectl get hpa -n demo -w`来观察其状态变化。

现在创建一个 Pod 来增加上面 Nginx 服务的访问压力。这里我们用压测工具ApacheBench 进行压力测试看看：

```sh
$ kubectl run demo-benchmark --image httpd:2.4.46-alpine -n demo -it sh
/usr/local/apache2 # ab -n 50000 -c 500 -s 60 http://10.109.93.199/
This is ApacheBench, Version 2.3 <$Revision: 1879490 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/
Benchmarking 10.109.93.199 (be patient)
Completed 5000 requests
Completed 10000 requests
Completed 15000 requests
Completed 20000 requests
Completed 25000 requests
Completed 30000 requests
Completed 35000 requests
Completed 40000 requests
Completed 45000 requests
Completed 50000 requests
Finished 50000 requests
Server Software:        nginx/1.19.2
Server Hostname:        10.109.93.199
Server Port:            80
Document Path:          /
Document Length:        612 bytes
Concurrency Level:      500
Time taken for tests:   74.783 seconds
Complete requests:      50000
Failed requests:        2
   (Connect: 0, Receive: 0, Length: 1, Exceptions: 1)
Total transferred:      42250000 bytes
HTML transferred:       30600000 bytes
Requests per second:    668.60 [#/sec] (mean)
Time per request:       747.830 [ms] (mean)
Time per request:       1.496 [ms] (mean, across all concurrent requests)
Transfer rate:          551.73 [Kbytes/sec] received
Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0   95 347.7     10    3134
Processing:     9   49 295.1     33   60048
Waiting:        5   44 122.8     30    3393
Total:         19  144 477.2     50   60048
Percentage of the requests served within a certain time (ms)
  50%     50
  66%     61
  75%     65
  80%     69
  90%     83
  95%   1071
  98%   1143
  99%   1946
 100%  60048 (longest request)
```

在上述 ab 命令运行的同时，我们可以切回之前打开的两个终端窗口看看输出信息：

```sh
$ kubectl get hpa -n demo -w
NAME               REFERENCE                     TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
nginx-deployment   Deployment/nginx-deployment   0%/50%    1         10        1          48m
nginx-deployment   Deployment/nginx-deployment   125%/50%   1         10        1         50m
nginx-deployment   Deployment/nginx-deployment   125%/50%   1         10        3         50m
nginx-deployment   Deployment/nginx-deployment   0%/50%     1         10        3         51m
nginx-deployment   Deployment/nginx-deployment   0%/50%     1         10        3         56m
nginx-deployment   Deployment/nginx-deployment   0%/50%     1         10        1         56m

$ kubectl get deploy -n demo -w
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   1/1     1            1           48m
nginx-deployment   1/3     1            1           50m
nginx-deployment   1/3     1            1           50m
nginx-deployment   1/3     1            1           50m
nginx-deployment   1/3     3            1           50m
nginx-deployment   2/3     3            2           50m
nginx-deployment   3/3     3            3           50m
nginx-deployment   3/1     3            3           56m
nginx-deployment   3/1     3            3           56m
nginx-deployment   1/1     1            1           56m
```

可以看到，随着访问压力的增加，Pod 的平均利用率也直线上升，一度达到了 125%，超过我们的阈值 50%。这个时候，Deployment 的副本数被调整到了 3，随之 2 个新 Pod 被拉起，负载很快降到了 50% 以下。而后随着压测结束，HPA 又将 Deployment 调整为了 1，维持在低水位。
我们现在回过头来看看 metrics-server 创建的对象PodMetrics：

```sh
$ kubectl get podmetrics -n demo
NAME                                AGE
nginx-deployment-6d4b885966-zngnd   0s
nginx-deployment-6d4b885966-lgwd9   0s
nginx-deployment-6d4b885966-hhk7v   0s

$ kubectl get podmetrics -n demo nginx-deployment-6d4b885966-zngnd -o yaml
apiVersion: metrics.k8s.io/v1beta1
containers:
- name: nginx
  usage:
    cpu: "0"
    memory: 5524Ki
kind: PodMetrics
metadata:
  creationTimestamp: "2020-10-13T09:38:15Z"
  name: nginx-deployment-6d4b885966-zngnd
  namespace: demo
  selfLink: /apis/metrics.k8s.io/v1beta1/namespaces/demo/pods/nginx-deployment-6d4b885966-zngnd
timestamp: "2020-10-13T09:37:55Z"
```

HPAController 就是通过这些 PodMetrics 来计算平均的 CPU 使用率，从而确定 `spec.replicas` 的新数值。

除了 CPU 以外，HPA 还支持其他自定义度量指标，有兴趣可以参考 [官方文档](https://kubernetes.io/zh/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/#%E5%9F%BA%E4%BA%8E%E5%A4%9A%E9%A1%B9%E5%BA%A6%E9%87%8F%E6%8C%87%E6%A0%87%E5%92%8C%E8%87%AA%E5%AE%9A%E4%B9%89%E5%BA%A6%E9%87%8F%E6%8C%87%E6%A0%87%E8%87%AA%E5%8A%A8%E6%89%A9%E7%BC%A9)。

HPA 能够自适应地伸缩 Pod 的数目，但是如果集群中资源不够了怎么办？比如节点紧张无法支撑新的 Pod 运行？

这个时候我们就可以添加新的节点资源到集群中，那么有没有类似 HPA 的做法可以自动化地扩容集群的节点资源？

当然是有的！我们来看看 Cluster Autoscaler。



### Cluster Autoscaler

[Cluster Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler)（下文统称 CA）目前对接了 [阿里云](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/alicloud/README.md)、[AWS](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/aws/README.md)、[Azure](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/azure/README.md)、[百度云](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/baiducloud/README.md)、[华为云](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/huaweicloud/README.md)、[Openstack](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/magnum/README.md) 等云厂商，你可以参照各个厂商的部署要求，进行部署。在部署的时候，请注意 CA 和 Kubernetes 版本要对应，[最好两者版本一样](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler#releases)。

这里描述了 CA 和 HPA 一起使用的情形。两者一般可以配合起来一起使用。

![Drawing 1.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F-H5LCAEEBiAAB2SWxVmmg879.png)

我们来看看 CA 是如何工作的。CA 主要用来监听（watch）集群中未被调度的 Pod （即 Pod 暂时由于某些调度策略、抑或资源不满足，导致无法被成功调度），然后确定是否可以通过增加节点资源来解决无法调度的问题。

如果可以的话，就会调用对应的 cloud provider 接口，向集群中增加新的节点。当然 CA 在创建新的节点资源前，也会尝试是否可以将正在运行的一部分 Pod “挤压”到某些节点上，从而让这些未被调度的 Pod 可以被调度，如果可行的话，CA 会将这些 Pod 进行驱逐。这些被驱逐的 Pod 会被重新调度（reschedule）到其他的节点上。我们会在后续调度章节来深入讨论有关 Pod 驱逐的话题。

目前 CA 仅支持 [部分云厂商](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler#deployment)，如果不满足你的需求，你可以考虑基于 [社区的代码](https://github.com/kubernetes/autoscaler) 进行二次开发，链接中有详细说明我就不赘述了。



### 写在最后

除了 HPA 和 CA 以外，还有 [Vertical Pod Autoscaler (VPA)](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)可以帮我们确定 Pod 中合适的 CPU 和 Memory 区间，有兴趣可以了解下。限于篇幅，本节不多做介绍。在实际使用的时候，注意千万不要同时使用 HPA 和 VPA，以免造成异常。

使用 HPA 的时候，也尽量对 Deployment 这类对象进行操作，避免对 ReplicaSet 操作。毕竟 ReplicaSet 由 Deployment 管理着，一旦 Deployment 更新了，旧的ReplicaSet 会被新的 ReplicaSet 替换掉。





## 17 | 案例实战：教你快速搭建 Kubernetes 监控平台



[Prometheus](https://prometheus.io/) 和 [Grafana](https://grafana.com/) 可以说是 Kubernetes 监控解决方案中最知名的两个。Prometheus 负责收集、存储、查询数据，而 Grafana 负责将 Prometheus 中的数据进行可视化展示，当然 Grafana 还支持其他平台，比如 ElasticSearch、InfluxDB、Graphite 等。[CNCF 博客](https://www.cncf.io/blog/2020/04/24/prometheus-and-grafana-the-perfect-combo/) 也将这两者称为黄金组合，目前一些公有云提供的托管式 Kubernetes （Managed Kubernetes） 都已经默认安装了 Prometheus 和 Grafana。

![Drawing 0.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl-OrgyAPH_iAAFeouU_wAY811.png)

我们今天就来学习如何在 Kubernetes 集群中搭建 Prometheus 和 Grafana，使之帮我们监控 Kubernetes 集群。



### 通过 Helm 一键安装 Prometheus 和 Grafana

还记得之前讲过的 Helm 吗？我们今天就通过 Helm 来安装相关的 Charts，一键式搭建 Prometheus 和 Grafana。

首先我们先通过`helm repo add`添加一个 Helm repo，见如下命令：

```sh
$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
"prometheus-community" has been added to your repositories
```

我们从这个刚刚添加的`prometheus-community`中安装 Prometheus 的 Chart。

然后你通过helm repo list就可以看到当前已添加的所有 repo 列表：

```sh
$ helm repo list
NAME                    URL
prometheus-community    https://prometheus-community.github.io/helm-charts
```

如果你已经添加了这个 repo，那么可以运行`helm repo update`来更新内容：

```sh
$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "prometheus-community" chart repository
...Successfully got an update from the "bitnami" chart repository
Update Complete. ⎈Happy Helming!⎈
```

正如我们之前讲 Helm 提到过的，Helm 的这些 repo 非常类似于我们熟悉的 YUM 源、debian 源。

回到正题上，添加了这个 helm repo 了以后，我们就可以开始安装 Prometheus 的 Chart 了。通常来说，使用 Helm 安装 Chart 的最佳实践是单独创建一个 namespace （命名空间），方便区分、隔离和管理。这里我们就创建一个名为`monitor`的 namespace:

```sh
$ kubectl create ns monitor
namespace/monitor created
```

创建好了 namespace，我们直接运行如下`helm install`命令进行安装:

```sh
$ helm install prometheus-stack prometheus-community/kube-prometheus-stack -n monitor
NAME: prometheus-stack
LAST DEPLOYED: Mon Oct 19 11:07:42 2020
NAMESPACE: monitor
STATUS: deployed
REVISION: 1
NOTES:
kube-prometheus-stack has been installed. Check its status by running:
  kubectl --namespace monitor get pods -l "release=prometheus-stack"
Visit https://github.com/prometheus-operator/kube-prometheus for instructions on how to create & configure Alertmanager and Prometheus instances using the Operator.
```

下面我来解释下上述命令中的参数含义。

其中`prometheus-stack`是这个 helm release 的名字，当然你在实际使用时候可以换成其他名字。从这个名字的后缀stack就可以看出来，`prometheus-community/kube-prometheus-stack`这个 Chart 其实包含了几个组件，类似于我们之前听说过的 [ELKStack](https://www.elastic.co/cn/what-is/elk-stack) 等。除了安装 Prometheus 相关的组件外，该 Chart 的 `requirements.yaml` 中还定义了三个依赖组件

- [stable/kube-state-metrics](https://github.com/helm/charts/tree/master/stable/kube-state-metrics)；

- [stable/prometheus-node-exporter](https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus-node-exporter)；

- [grafana/grafana](https://github.com/grafana/helm-charts/tree/main/charts/grafana)。

从这个 YAML 文件可以看到这个 Chart 不仅一起安装了 Grafana，还安装了我们之前第 15 讲提到的`kube-state-metrics`和`prometheus-node-exporter`组件。

接着看`prometheus-community/kube-prometheus-stack`，它就是`prometheus-community` repo里面名为`kube-prometheus-stack`的 Chart。

最后看`monitor`，它是我们刚创建的新命名空间，在部署的时候使用。

当然如果你不想安装这些依赖，也可以通过 [文档中的提及方法](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack#multiple-releases) 更改（override）这些默认设置。

现在我们运行刚才`helm install`输出结果中的命令，来查看我们相关 Pod 的状态：

```sh
$ kubectl --namespace monitor get pods -l "release=prometheus-stack"
NAME                                                   READY   STATUS    RESTARTS   AGE
prometheus-stack-kube-prom-operator-6998d5c5b7-kgv8b   2/2     Running   0          2m41s
prometheus-stack-prometheus-node-exporter-l7pr9        1/1     Running   0          2m41s
```

这里有一个名为`prometheus-stack-kube-prom-operator-6998d5c5b7-kgv8b`的 Pod，它是 [prometheus-operator](https://github.com/prometheus-operator/prometheus-operator) 的一个实例，主要用于为我们创建、管理和维护 Prometheus 集群，其具体架构图如下。

![Drawing 1.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F-OrkWAXJsQAAEHWw6G6rM837.png)

Operator 的工作流程相对复杂一些，我会在后面单独行介绍，在此先略过。有兴趣的话可以查看 [官方文档](https://coreos.com/operators/prometheus/docs/latest/user-guides/getting-started.html) 简单了解下。

另外一个叫 `prometheus-stack-prometheus-node-exporter-l7pr9` 的 Pod 是一个 [node-exporter](https://github.com/prometheus/node_exporter)，主要来采集 kubelet 所在节点上的 metrics。

这两个 Pod 都运行成功，我们再来看看`monitor`这个 namespace 下面其他依赖组件的状态：

```sh
$ kubectl get all -n monitor
NAME                                                         READY   STATUS    RESTARTS   AGE
pod/alertmanager-prometheus-stack-kube-prom-alertmanager-0   2/2     Running   0          11m
pod/prometheus-prometheus-stack-kube-prom-prometheus-0       3/3     Running   1          11m
pod/prometheus-stack-grafana-5b6dd6b5fb-rtp6z                2/2     Running   0          11m
pod/prometheus-stack-kube-prom-operator-6998d5c5b7-kgv8b     2/2     Running   0          11m
pod/prometheus-stack-kube-state-metrics-c7c69c8c9-bhgjv      1/1     Running   0          11m
pod/prometheus-stack-prometheus-node-exporter-l7pr9          1/1     Running   0          11m
NAME                                                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/alertmanager-operated                       ClusterIP   None             <none>        9093/TCP,9094/TCP,9094/UDP   11m
service/prometheus-operated                         ClusterIP   None             <none>        9090/TCP                     11m
service/prometheus-stack-grafana                    ClusterIP   10.108.122.155   <none>        80/TCP                       11m
service/prometheus-stack-kube-prom-alertmanager     ClusterIP   10.103.37.81     <none>        9093/TCP                     11m
service/prometheus-stack-kube-prom-operator         ClusterIP   10.97.75.165     <none>        8080/TCP,443/TCP             11m
service/prometheus-stack-kube-prom-prometheus       ClusterIP   10.102.82.76     <none>        9090/TCP                     11m
service/prometheus-stack-kube-state-metrics         ClusterIP   10.109.78.8      <none>        8080/TCP                     11m
service/prometheus-stack-prometheus-node-exporter   ClusterIP   10.101.221.185   <none>        9100/TCP                     11m
NAME                                                       DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/prometheus-stack-prometheus-node-exporter   1         1         1       1            1           <none>          11m
NAME                                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/prometheus-stack-grafana              1/1     1            1           11m
deployment.apps/prometheus-stack-kube-prom-operator   1/1     1            1           11m
deployment.apps/prometheus-stack-kube-state-metrics   1/1     1            1           11m
NAME                                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/prometheus-stack-grafana-5b6dd6b5fb              1         1         1       11m
replicaset.apps/prometheus-stack-kube-prom-operator-6998d5c5b7   1         1         1       11m
replicaset.apps/prometheus-stack-kube-state-metrics-c7c69c8c9    1         1         1       11m
NAME                                                                    READY   AGE
statefulset.apps/alertmanager-prometheus-stack-kube-prom-alertmanager   1/1     11m
statefulset.apps/prometheus-prometheus-stack-kube-prom-prometheus       1/1
```

可以看到全部 Pod 都已经部署成功且运行。

prometheus-operator 以 Deployment 的形式部署，并帮助我们创建了名为 prometheus-prometheus-stack-kube-prom-prometheus 的 StatefulSet，副本数设置为 1。

接下来就可以本地来访问 Prometheus 了，通过如下命令，我们在本地通过 http://127.0.0.1:9090 来访问 Prometheus：

```sh
$ kubectl port-forward -n monitor prometheus-prometheus-stack-kube-prom-prometheus-0 9090
```

![Drawing 2.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl-OrlyAf9PiAAW6ru2bHZI707.png)

同样地，我们也可以使用如下命令，在本地通过 http://127.0.0.1:3000 来访问 Grafana：

```sh
$ kubectl port-forward -n monitor prometheus-stack-grafana-5b6dd6b5fb-rtp6z 3000
```

![Drawing 3.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl-OrmeAXd8PABEbqH6I_pE362.png)

这里默认用户名是 admin，密码是 prom-operator。当然你也可以通过 Grafana 的配置来拿到，我们来看看如何操作。

下面是刚才 Chart 部署的 grafana Deployment 的配置：

```yaml
$ kubectl get deploy -n monitor prometheus-stack-grafana -o yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-stack-grafana
  namespace: monitor
  ...
spec:
  ...
  template:
    ...
    spec:
      containers:
      ...
      - env:
        - name: GF_SECURITY_ADMIN_USER
          valueFrom:
            secretKeyRef:
              key: admin-user
              name: prometheus-stack-grafana
        - name: GF_SECURITY_ADMIN_PASSWORD
          valueFrom:
            secretKeyRef:
              key: admin-password
              name: prometheus-stack-grafana
        image: grafana/grafana:7.2.0
        ...
      ...
status:
  ...
```

可以看到环境变量`GF_SECURITY_ADMIN_USER`和`GF_SECURITY_ADMIN_PASSWORD`就是 grafana 的登录用户名和密码，具体的值来自 `Secretprometheus-stack-grafana`。

我们接着看这个 Secret：

```sh
$ kubectl get secret prometheus-stack-grafana -n monitor -o jsonpath='{.data}'
map[admin-password:cHJvbS1vcGVyYXRvcg== admin-user:YWRtaW4= ldap-toml:]
```

分别通过 base64 对其进行解码就可以得到用户名和密码：

```sh
$ kubectl get secret prometheus-stack-grafana -n monitor -o jsonpath='{.data.admin-user}' | base64 --decode
admin
$ kubectl get secret prometheus-stack-grafana -n monitor -o jsonpath='{.data.admin-password}' | base64 --decode
prom-operator
```

使用上述用户名和密码登录，进来后就可以看到 grafana 的主页面了。

![Drawing 4.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl-OroiAZBPUABgqqawSub0949.png)

按照上述图示点进来，你就可以看到已经配置好的各个 Dashboard：

![Drawing 5.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl-OrpKAG4VoAAs8370oK_k082.png)

这些都是 `prometheus-community/kube-prometheus-stack`这个 Chart 预先配置好的，基本上包括我们对 Kubernetes 的各项监控大盘，你可以随意点击几个 Dashboard 进行了解。

![Drawing 6.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl-OrtSASkD9ABXvtUrLlxI610.png)

你可以参考 [官方文档](https://grafana.com/docs/grafana/latest/dashboards/) 学习 Dashboard 的创建和数据展示能力。



### 生产环境中一些重要的关注指标

#### 集群状态、性能以及各个 API 对象

kube-state-metrics 可以帮助我们汇聚 Kubernetes 集群中的各大信息，比如 Pod 数量，APIServer 访问请求数，Pod 调度性能，等等。你可以通过如下命令可本地访问http://127.0.0.1:8080/metrics拿到相关的 metrics。

```sh
$ kubectl port-forward -n monitor prometheus-stack-kube-state-metrics-c7c69c8c9-bhgjv 8080
```

![Drawing 7.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl-Oru-AevJ5AB_ZvYnlO60668.png)



#### 各节点的监控指标

各个节点承载着 Pod 的运行，因此对各个节点的监控至关重要，比如节点的 CPU 使用率、内存使用率、平均工作负载等。你可以通过上面 Chart 预配置好的 Node dashboard 来查看：

![Drawing 8.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F-OrvaARKZ2ABNH_bHh8sg135.png)

你可以通过切换节点来查看不同节点的状态，或者单独创建一个 Dashboard，添加一些指标，诸如：

- `kube_node_status_capacity_cpu_cores`代表节点 CPU 容量；

- `kube_node_status_capacity_memory_bytes`代表节点的 Memory容量；

- `kubelet_running_container_count`代表节点上运行的容器数量；

- ……



#### 设置 Alert

除了监控指标的收集和展示以外，我们还需要设置报警（Alert）。你可以通过设置合适的 Alert rules，在触发的时候通过钉钉、邮件、Slack、PagerDuty 等多种方式通知你。

![Drawing 9.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl-Orv6ACQ9XAAqhwgoMegs538.png)

在此不做太多介绍，可以通过 [官方文档](https://grafana.com/docs/grafana/latest/alerting/create-alerts/) 来设置你的 Alert。



### 写在最后

在这一小节，我们介绍了快速搭建基于 Prometheus + Grafana 的 Kubernetes 监控体系。当然安装的方式有很多，今天这里只写了最方便、最快捷的方式。Chart 里预先配置了多个 Dashboard，方便你开箱即用。







# 安全无忧：集群的安全性与稳定性



## 18 | 权限分析：Kubernetes 集群权限管理那些事儿



通过前面的课程学习，你已经学会了使用`kubectl`命令行，或者直接发送 REST 请求，以及使用各种语言的 [client 库](https://kubernetes.io/docs/reference/using-api/client-libraries/) 来跟 APIServer 进行交互。那么你是否知道在这其中Kubernetes 是如何对这些请求进行认证、授权的呢？

任何请求访问 Kubernetes 的 kube-apiserver 时，都要依次经历三个阶段：认证（Authentication，有时简写成 AuthN）、授权（Authorization，有时简写成 AuthZ）和准入控制（Admission Control）。

![image (2).png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F-RVYyAeaf5AABESlN1pJg327.png)

其中，**认证**这个概念比较好理解，就是做身份校验，解决“你是谁？”的问题。

**授权**负责做权限管控，解决“你能干什么？”的问题，给予你能访问 Kubernetes 集群中的哪些资源以及做哪些操作的权利，比如你是否能创建一个 Pod，是否能删除一个 Pod。

**准入控制**，从字面上你可能不太好理解。其实就是由一组控制逻辑级联而成，对对象进行拦截校验、更改等操作。比如你打算在一个名为 test 的 namespace 中创建一个 Pod，如果这个 namespace 不存在，集群要不要自动创建出来？或者直接拒绝掉该请求？这些逻辑都可以通过准入控制进行配置。

当然如果你的 Kubernetes 集群开启了 HTTP 访问，就不再需要认证和授权流程了。显然这种配置相当于敞开大门，风险太大了，所以在生产环境我建议你不要这么配置。

上述三个阶段，任何一个阶段验证失败都会导致该请求被拒绝访问，我们现在深入认识一下每个阶段。



### 认证

在认证阶段最重要的就是身份（Identity），我们需要从中获取到用户的相关信息。通常，Identity 信息可以通过 User 或者 Group 来定义，但是 Kubernetes 中其实并没有相关定义，所以你无法通过 Kubernetes API 接口进行创建、查询和管理。

Kubernetes 认为 User 这类信息由外部系统来管理，自己并不负责管理和设计，这样可以避免重复定义用户模型，方便第三方用户权限平台进行对接。

所以 Kuberentes 是如何做身份认证的呢？这里我们简单介绍一下 Kubernetes 中几种常见的用户认证方式。



#### x509 证书

Kubernetes 使用 [mTLS](https://developers.cloudflare.com/access/service-auth/mtls) 进行身份验证，通过解析请求使用的 客户端（client）证书，将其中的 subject 的通用名称（Common Name）字段（例如"/CN=bob"）作为用户名。Kubernetes 集群各个组件间相互通信，都是基于 x509 证书进行身份校验的。关于证书的创建、查看等，可以参考 [官方文档](https://kubernetes.io/zh/docs/concepts/cluster-administration/certificates/)。

比如使用`openssl`命令行工具生成一个证书签名请求（CSR）：

```sh
$ openssl req -new -key zhangsan.pem -out zhangsan-csr.pem -subj "/CN=zhangsan/O=app1/O=app2"
```

这条命令会使用用户名zhangsan生成一个 CSR，且它属于 "app1" 和 "app2" 两个用户组，然后我们使用 CA 证书根据这个 CSR 就可以签发出一对证书。当用这对证书去访问 APIServer 时，它拿到的用户就是zhangsan了。



#### Token

我们接着来看基于 Token 的验证方式，它又可以分为几种类型。

第一种是**用户自己提供的静态 Token**，可以将这些 Token 写到一个 CSV 文件中，其中至少包含 3 列：分别为 Token、用户名和用户 ID。你可以增加一个可选列包含 group 的信息，比如：

```csv
token1,user1,uid1,"group1,group2,group3" 
```

APIServer 在启动时会通过参数`--token-auth-file=xxxx`来指定读取的 CSV 文件。不过这种方式，实际环境中基本上不会使用到。毕竟这种静态文件不方便修改，每次修改后还需要重新启动 APIServer。

还有一种是**ServiceAccount Token**。这是 Kubernetes 中使用最为广泛的认证方式之一，主要用来给 Pod 提供访问 APIServer 的权限，通过使用 Volume 的方式挂载到 Pod 中。我们之前讲 Pod、Volume 的时候提到过，你也可以参考 [这份文档](https://kubernetes.io/zh/docs/tasks/configure-pod-container/configure-service-account/) 重温下如何为 Pod 配置 ServiceAccount。

当你创建一个 ServiceAccount 的时候，kube-controller-manager 会自动帮你创建出一个Secret来保存 Token，比如：

```sh
$ kubectl create sa demo
serviceaccount/demo created
$ kubectl get sa demo
NAME   SECRETS   AGE
demo   1         6s
$ kubectl describe sa demo
Name:                demo
Namespace:           default
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   demo-token-fvsjg
Tokens:              demo-token-fvsjg
Events:              <none>
```

可以看到这里自动创建了一个名为`demo-token-fvsjg`的 Secret，我们来查看一下其中的内容：

```sh
$ kubectl get secret demo-token-fvsjg
NAME               TYPE                                  DATA   AGE
demo-token-fvsjg   kubernetes.io/service-account-token   3      27s

$ kubectl describe secret demo-token-fvsjg
Name:         demo-token-fvsjg
Namespace:    default
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: demo
              kubernetes.io/service-account.uid: f8fe4799-9add-4a36-8c29-a6b2744ba9a2
Type:  kubernetes.io/service-account-token
Data
====
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6Ildhc3plOFE0UXBlRE8xTFdsRThRVXktRF93T0otcW55VXI3R0RvV1IzVjgifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImRlbW8tdG9rZW4tZnZzamciLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGVtbyIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImY4ZmU0Nzk5LTlhZGQtNGEzNi04YzI5LWE2YjI3NDRiYTlhMiIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmRlbW8ifQ.t0sNk8PiRIgMXaauzM8v8eW_AF7J5Su_TpyURaeBQX0BuFDpU-rrTC6h9sjetAQOLa6iLFsrG2tqH04pNo2N6TB73-K-M54veVpP6FGhceLO0Vsz_iTJuOsnkS_6agqfW0UrtaY8_oCk1OZipdbTJYZlmwlb6opcfP1j7bFHCdDbwfbWx8ilHqTaJ5_r7zrdhxsmC5ogtBRDaJfEYmsIBZSgnt_0qMHNj9L47n2mj_wo-aQQ25o0lbO_7RSeVA17kkgbAJ2wzwvv8-i3f7K23DSxab3k8trbuIRt1R3Gl33fv5WIwiz9okYIFfus4pe0MCtjBxxTMa526cdezKS4LQ
ca.crt:     1025 bytes
namespace:  7 bytes
```

Data 里面的 token 我们可以到https://jwt.io/中 Decode 出来：

![image (3).png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F-RVduACXUqABUdlGrHMh4648.png)

从解析出来的 Payload 中可以看到， ServiceAccount 所属的 namespace、name 等信息。

除此之外，还支持[Bootstrap Token](https://kubernetes.io/zh/docs/reference/access-authn-authz/authentication/#bootstrap-tokens)、[OpeID Connect Token (OIDC Token)](https://kubernetes.io/zh/docs/reference/access-authn-authz/authentication/#openid-connect-tokens)、[Webhook Token](https://kubernetes.io/zh/docs/reference/access-authn-authz/authentication/#webhook-token-authentication) 等，在此不一一介绍。

值得一提的是，Bootstrap Token 是 kubeadm 引入的方便创建新集群所使用的 Token 类型，通过如下类似的命令就可以将其加入新的节点中：

```sh
$ kubeadm join --discovery-token abcdef.1234567890abcdef
```

不管是上面哪种 Token，我们都可以拿来向 APIServer 发送请求，只需要向 HTTP 请求的 Header 中添加一个键为 Authorization ，值为`Bearer THETOKEN`的字段，类似如下：

```sh
Authorization: Bearer this-is-a-very-very-very-long-token 
```

还有一种是静态用户密码文件，在 1.16 版本以前，Kubernetes 支持指定一个 CSV 文件，里面包含了用户名以及密码等信息。但是在 1.16 版本开始已经 [被废弃掉了](https://github.com/kubernetes/kubernetes/pull/81152)，这里我提出来是想告诉你请不要再使用这种方式了。

这些也是使用最多的认证方式，通过认证就来到了授权阶段。



### 授权

Kubernetes 内部有多种授权模块，比如 [Node](https://kubernetes.io/zh/docs/reference/access-authn-authz/node/)、[ABAC](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=447#/detail/pc?id=4535)、[RBAC](https://kubernetes.io/zh/docs/reference/access-authn-authz/rbac/)、[Webhook](https://kubernetes.io/zh/docs/reference/access-authn-authz/webhook/)。授权阶段会根据从 AuthN 拿到的用户信息，依次按配置的授权次序逐一进行权限验证。任一授权模块验证通过，即允许该请求继续访问。

我们这里简要讲一下 ABAC 和 RBAC 这两种授权模式。



#### ABAC

ABAC (Attribute Based Access Control) 是一种基于属性的访问控制，可以给 APIServer 指定一个 JSON 文件`--authorization-policy-file=SOME_FILENAME`，该文件描述了一组属性组合策略，比如：

```json
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user": "zhangsan", "namespace": "*", "resource": "pods", "readonly": true}}
```

这条策略表示，用户 zhangsan 可以以**只读**的方式读取任何 namespace 中的 Pod。这里有一份 [示例文件](https://raw.githubusercontent.com/kubernetes/kubernetes/master/pkg/auth/authorizer/abac/example_policy_file.jsonl)，你可以参考一下。

不过在实际使用中，ABAC 使用得比较少，跟我们上面 AuthN 中提到的静态 Token 一样，不方便修改、更新，每次变更后都需要重启 APIServer。所以在实际使用中，RBAC 最常见，使用更广泛。



#### RBAC

相对而言，RBAC 使用起来就非常方便了，通过 Kubernetes 的对象就可以直接进行管理，也便于动态调整权限。

RBAC 中引入了角色，所有的权限都是围着这个角色进行展开的，每个角色里面定义了可以操作的资源以及操作方式。在 Kubernetes 中有两种角色，一种是 namespace 级别的 Role ，一种是集群级别的 ClusterRole 。

在 Kubernetes 中有些资源是 namespace 级别的，比如 Pod、Service 等，而有些则属于集群级别，比如 Node、PV 等。所以有了 Role 和 ClusterRole 两种角色。

我们来看个例子：

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # 空字符串""表明使用core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

这样一个角色表示可以访问 default 命名空间下面的 Pods，并可以对其进行 get、watch 以及 list 操作。

ClusterRole 除了可以定义集群方位的资源外，比如 Node，还可以定义跨 namespace 的资源访问，比如你想访问所有命名空间下面的 Pod，就可以这么定义：

```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  # 鉴于ClusterRole是集群范围对象，所以这里不需要定义"namespace"字段
  name: pods-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

RBAC 中定义这些角色包含了一组权限规则，这些规则纯粹以累加形式组合，没有互斥的概念。
定义好角色，我们就可以做权限绑定了。这里分别有两个 API 对象，RoleBinding 和 ClusterRoleBinding。角色绑定就是把一个权限授予一个或者一组用户，例如：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
# 此角色绑定使得用户 "jane" 或者 "manager"组（Group）能够读取 "default" 命名空间中的 Pods
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: jane # 名称区分大小写
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: manager # 名称区分大小写
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role # 这里可以是 Role，也可以是 ClusterRole
  name: pods-reader # 这里的名称必须与你想要绑定的 Role 或 ClusterRole 名称一致
  apiGroup: rbac.authorization.k8s.io
```

这里定义的 RoleBinding 对象在 default 命名空间中将 pods-reader 角色授予用户 jane和组 manager。 这一授权将允许用户 jane 和 manager 组中的各个用户从default 命名空间中读取到 Pod。



### 准入控制

准入控制可以帮助我们在 APIServer 真正处理对象前做一些校验以及修改的工作。

这里官方列出了很多 [准入控制组件](https://kubernetes.io/zh/docs/reference/access-authn-authz/admission-controllers/#%E6%AF%8F%E4%B8%AA%E5%87%86%E5%85%A5%E6%8E%A7%E5%88%B6%E5%99%A8%E7%9A%84%E4%BD%9C%E7%94%A8%E6%98%AF%E4%BB%80%E4%B9%88)，官方已经将使用方法写得很清楚了，你可以根据自己的需要，选择合适的准入控制插件，我就不再赘述了。



### 写在最后

作为一个企业级的容器管理调度平台，Kubernetes 在 Auth 方面设计得很完善，支持多种后端身份认证授权系统，比如 LDAP (Lightweight Directory Access Protocol)、Webhook 等。AuthN 负责完成对用户的认证，并获取用户的相关信息（比如 Username、Groups 等），而 AuthZ 则会根据 AuthN 返回的用户信息，根据配置的授权模式进行权限校验，来决定是否允许对某个 API 资源的操作请求。





## 19 | 资源限制：如何保障你的 Kubernetes 集群资源不会被打爆



前面的课时中，我们曾提到通过 HPA 控制业务的资源水位，通过 ClusterAutoscaler 自动扩充集群的资源。但如果集群资源本身就是受限的情况下，或者一时无法短时间内扩容，那么我们该如何控制集群的整体资源水位，保障集群资源不会被“打爆”？

今天我们就来看看 Kubernetes 中都有哪些能力可以帮助我们保障集群资源？



### 设置 Requests 和 Limits

Kubernetes 中对容器的资源限制实际上是通过 [CGroup](https://www.ibm.com/developerworks/cn/linux/1506_cgroup/index.html) 来实现的。CGroup 是 Linux 内核的一个功能，用来限制、控制与分离一个进程组的资源（如 CPU、内存、磁盘输入输出等）。每一种资源比如 CPU、内存等，都有对应的 CGroup 。如果我们没有给 Pod 设置任何的 CPU 和 内存限制，这就意味着 Pod 可以消耗宿主机节点上足够多的 CPU 和 内存。

所以一般来说，我们都会对 Pod 进行资源限制， Kubernetes 通过给 Pod 设置资源请求（Requests）和资源限制（Limits）来实现这个资源限制。

- Requests 表示容器可以得到的资源，或者可以理解为 Pod 运行的最低资源要求。
- Limits 表示着容器最多可以得到的资源。Pod 运行过程中，比如 CPU 使用量会增加，那么最多能使用多少内存，这就是资源限制。

这里有一点需要注意的就是，Limits 永远不要低于 Requests，如果设置不对，Kubernetes 也会拒绝 Pod 的创建。

通过设置 Requests 和 Limits，我们既保证了 Pod 可以运行，又限制 Pod 能使用多少资源。这样能避免某些恶意的容器“吞噬”宿主机的资源，也可以避免某些容器异常导致宿主机 OOM，从而引起该节点上的所有 Pod 异常，甚至导致整个集群“雪崩”。

我们来看看个 Requests 和 Limits 的例子：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-resource-demo
  namespace: demo
spec:
  containers:
  - name: demo-container-1
    image: nginx:1.19
    resources:
    requests:
      memory: "64Mi"
      cpu: "250m"
    limits:
      memory: "128Mi"
      cpu: "500m"
  - name: demo-container-2
    image: nginx:1.19
    resources:
    requests:
      memory: "64Mi"
      cpu: "250m"
    limits:
      memory: "128Mi"
      cpu: "500m"
```

如上所示，Pod 中的每个容器都可以设置自己的 Requests 和 Limits，每个容器使用的资源都不能超过各自的限制。当 Pod 在调度时，会把这些容器的 Requests 和 Limits 进行相加，当作整个 Pod 的资源申请量。因此在上面的示例中，Pod 的总 Requests 为 500 mCPU，128 MiB 内存，总 Limits 为 1 CPU和 256 MiB。关于单位的含义，[官方文档](https://kubernetes.io/zh/docs/concepts/configuration/manage-resources-containers/#pod-%E5%92%8C-%E5%AE%B9%E5%99%A8%E7%9A%84%E8%B5%84%E6%BA%90%E8%AF%B7%E6%B1%82%E5%92%8C%E7%BA%A6%E6%9D%9F) 有更详细的说明。

一旦 Pod 成功被调度后，Kubernetes 会将其调度到可以为其提供该资源的节点上。

而根据设置的 Requests 和 Limit，Kubernetes 又将其分为不同的 QoS (Quality of Service)级别。Kubernetes 中 Pod 是最小的单元，所以 QoS 是对整个 Pod 而言而非某个容器。

Kubernetes 支持了三种 QoS 级别，分别为BestEffort、Burstable 和 Guranteed，当资源紧张时 Kubernetes 会根据它们的分级决定调度和驱逐策略，这三个分级分别代表：

- BestEffort表示 Pod 中没有一个容器设置了 Requests 或 Limits，它的优先级最低；
- Burstable表示 Pod 中每个容器至少定义了 CPU 或 Memory 的 Requests，或者 Requests 和 Limits 不相等，它属于中等优先级；
- Guranteed则表示 Pod 中每个容器 Requests 和 Limits 都相等，这类 Pod 的运行优先级最高。简单来说就是`cpu.limits = cpu.requests`，`memory.limits = memory.requests`。

你可以通过 QoS 的代码来研究下 Kubernetes 是如何确定 Pod 对应的 QoS 的。这里，我们通过一个 Burstable Pod 的例子，来直观感受下 Kubernetes 的资源限制能力。



### 一个 Burstable Pod 的例子

这是一个 Burstable Pod 的 YAML 文件，该 Pod 内只有一个容器，且为容器的内存设置了 Requests 和 Limits，分别为 50 Mi 和 100 Mi。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-burstable-demo
  namespace: demo
spec:
  containers:
  - name: memory-demo
    image: polinux/stress
    resources:
      requests:
        memory: "50Mi"
      limits:
        memory: "100Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "250M", "--vm-hang", "1"]
```

我们通过kubectl create创建好了以后，来查看该 Pod 的状态：

```sh
$ kubectl -n demo get po
NAME                                  READY     STATUS        RESTARTS   AGE
memory-burstable-demo      0/1       OOMKilled     1          11s
```

可以看到该 Pod 被 OOM 杀掉了，因为限制使用100M，而实际使用 250M。那么如果是 CPU 使用超过了 Limits 呢？

这是一个为容器的 CPU 资源设置了 Requests 和 Limits 的 Pod YAML 文件：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cpu-burstable-demo
  namespace: demo
spec:
  containers:
  - name: cpu-demo
    image: vish/stress
    resources:
      limits:
        cpu: "1"
      requests:
        cpu: "0.5"
    args:
    - -cpus
    - "2"
```

这里我们同样先用`kubectl create`创建，然后用`kubectl top`来查看容器 cpu-demo 的资源使用情况：

```sh
$  kubectl -n demo top pod cpu-burstable-demo 
NAME       CPU(cores)   MEMORY(bytes)
cpu-demo   1000m        0Mi
```

可以看到 Pod 的内存使用虽然超过了 Limits，实际使用的 CPU 被限制只有 1000 m，但是不会被 OOM 掉，这是因为 CPU 不同于内存，CPU 是可压缩资源（[Compressible Resource](https://www.bmc.com/blogs/kubernetes-compute-resources/)），而内存是不可压缩资源(Incompressible Resource)。

如果只是为了限制资源，用 Requests 和 Limits 就足够了，那么为何 Kubernetes 还要单独引入 QoS 的概念呢？要回答这个问题，我们就要来看看 QoS 的主要作用。



### QoS 的主要作用

集群运行一段时间以后，Node 上会有很多 Running 的 Pod。当 Node 上的资源紧张时，可能由于某些BestEffort的 Pod 使用的 CPU 和 Memory 越来越多，或者宿主机某些进程（例如 Kubelet、Docker）占用了 CPU 和 Memory，这个时候Kubernetes 就会根据 QoS 的优先级来选择 Kill 掉一部分 Pod，哪些会先被 Kill 掉呢？

当然是优先级最低的，即BestEffort类型的 Pod，占用的资源越多越优先被 Kill 掉。如果所有BestEffort的 Pod 都被杀死了但是资源依旧紧张，那么接下来会选择 Kill 中等优先级的，即Burstable类型的，之后以此类推。

这里 QoS 的一个作用就是跟oom_score进行挂钩。Kubernetes 会根据 QoS 设置 OOM 的评分调整参数oom_score_adj，有兴趣可以阅读 [详细的计算代码](https://github.com/kubernetes/kubernetes/blob/master/pkg/kubelet/qos/policy.go#L34)。当发生 OOM 时，oom_score_adj数值越高就越优先被 Kill。这里我给你展示了三个 QoS 对应的oom_score_adj计算公式。

![image.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F-X4kSAWDiXAABGicJIZQU816.png)

除此之外，QoS 还与 Pod 驱逐有关系。当节点的内存、CPU 资源不足时，Kubelet 会开始驱逐节点上的 Pod，它会依据 QoS 的优先级确定驱逐的顺序，跟上面 OOM kill 的次序一样。

在实际使用的时候，我们可能会担心某些 Pod 申请了过大的资源，恶意占用，那么我们又该如何避免呢？



### 通过 LimitRange 设置资源防线

Kubernetes 提供了 LimitRange 可以帮助你限定 CPU 和 Memory 的申请范围。

这是一个完整的 LimitRange 定义，你可以根据需要按需选择进行配置。

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-limit-range
  namespace: example
spec:
  limits:
  - default:  # 默认 limit
      memory: 512Mi
      cpu: 2
    defaultRequest:  # 默认 request
      memory: 256Mi
      cpu: 0.5
    max:  # 最大 limit
      memory: 800Mi
      cpu: 3
    min:  # 最小 request
      memory: 100Mi
      cpu: 0.3
    maxLimitRequestRatio:  # limit/request 的最大比率
      memory: 2
      cpu: 2
    type: Container # 支持 Container / Pod / PersistentVolumeClaim 三种类型
```

- default 字段可以设置 Pod 中容器的默认 Limits；
- defaulRequest 字段可以设置 Pod 中容器的默认 Requests；
- max 字段可以设置 Pod 中容器可以设置的最大 Limits，default 字段不能高于此值。同样，在容器上设置的 Limits 也不能高于此值。在使用的时候需要注意的是，如果设置了该字段而又没有设置 default，那么所有未显式设置这些值的容器都将使用此处的最大值作为 Limits。
- min 字段可以设置 Pod 中容器可以设置的最小 Requests。defaulRequest 字段不能低于此值。同样，在容器上设置的 Requests 也不能低于此值。同样需要注意的是，如果设置了该字段而又没有设置 defaulRequest，那么所有未显式设置这些值的容器都将使用此处的最小值作为 Requests。

LimitRange 会设置默认的申请、限制的值，它会自动在 Pod 创建时就注入 Container 中。

> 你可以参照如下几个官方文档中的详细例子学习体会一下：
>
> [如何配置每个命名空间最小和最大的 CPU 约束](https://kubernetes.io/zh/docs/tasks/administer-cluster/manage-resources/cpu-constraint-namespace/)
>
> [如何配置每个命名空间最小和最大的内存约束](https://kubernetes.io/zh/docs/tasks/administer-cluster/manage-resources/memory-constraint-namespace/)
>
> [如何配置每个命名空间默认的CPU 申请值和限制值](https://kubernetes.io/zh/docs/tasks/administer-cluster/manage-resources/cpu-default-namespace/) 
>
> [如何配置每个命名空间默认的内存申请值和限制值](https://kubernetes.io/zh/docs/tasks/administer-cluster/manage-resources/memory-default-namespace/)
>
> [如何配置每个命名空间最小和最大存储使用量](https://kubernetes.io/zh/docs/tasks/administer-cluster/limit-storage-consumption/#limitrange-to-limit-requests-for-storage)

除了对单个 Pod、Container、PVC 做资源限制外，我们还可以对某个 namespace 下的资源总量进行限制。



### ResourceQuota 设置资源总量限制

我们可以使用 ResourceQuota 对 namespace 内的资源总量进行限制，比如这个例子：

```yaml
apiVersion: v1
kind: ResourceQuota 
metadata: 
  name: compute-resources
  namespace: demo                  #在demo空间下
spec: 
  hard:
    requests.cpu: "10"             #cpu预配置10
    requests.memory: 100Gi         #内存预配置100Gi
    limits.cpu: "40"               #cpu最大不超过40
    limits.memory: 200Gi           #内存最大不超过200Gi
```

你可以看到它有四个部分，每个部分都是可选的，你可以根据自己的需要进行组合。

- `requests.cpu` 是该命名空间中所有容器的 CPU Requests 总和。在上面的例子中，你可以拥有10 个具有 1 个 CPU 请求的容器，或者 5 个具有 2 个 CPU 请求的容器。只要命名空间 demo 中所有容器的 CPU Requests 总和小于 10 即可。

- `requests.memory` 是该命名空间中所有容器的 Memory Requests 总和。同 CPU 一样，只要该命名空间中内存的总请求小于100Gi 即可。

- `limits.cpu` 是命名空间中所有容器的 CPU Limits 的总和。和 requests.cpu 一样，只不过这里是 Limits。

- `limits.memory` 是命名空间中所有容器的内存 Limits 的总和。和 requests.memory 一样，这里也是指 Limits。

除了 CPU 和内存这类资源以外，ResourceQuota 还支持扩展资源，详见 [官方文档的说明](https://kubernetes.io/zh/docs/concepts/policy/resource-quotas/#%E6%89%A9%E5%B1%95%E8%B5%84%E6%BA%90%E7%9A%84%E8%B5%84%E6%BA%90%E9%85%8D%E9%A2%9D)。

ResourceQuota 的功能非常强大，还可以对对象的数量进行限制。比如这个例子：

```yaml
apiVersion: v1 
kind: ResourceQuota 
metadata: 
  name: object-counts 
  namespace: demo                  #在demo命名空间下
spec: 
  hard: 
    configmaps: "10"               #最多10个configmap
    pods: "20"                     #最多20个pod
    persistentvolumeclaims: "4"    #最多10个pvc
    replicationcontrollers: "20"   #最多20个rc
    secrets: "10"                  #最多10个secrets
    services: "10"                 #最多10个service
    services.loadbalancers: "2"    #最多10个lb类型的service
    requests.nvidia.com/gpu: 4        #最多10个GPU
```

我们就可以限制该命名空间下最多可以创建 20 个 Pod，10 个 Configmap 等。



### 写在最后

对于一些重要的线上应用，我们要合理地设置 Requests 和 Limits，且最好使两者的设置相等，当节点资源不足时，Kubernetes 会优先保证这些 Pod 的正常运行。

此外，你可以用 ResourceQuota 限制命名空间中所有容器的内存请求总量、内存限制总量、CPU 请求总量、CPU 限制总量等。而如果你想对单个容器而不是所有容器进行限制，就可以使用 LimitRange。





## 20 | 资源优化：Kubernetes 中有 GC（垃圾回收）吗？



Garbage Collector 即垃圾回收，通常简称 GC，和你之前在其他编程语言中了解到的 GC 基本上是一样的，用来清理一些不用的资源。Kubernetes 中有各种各样的资源，当然需要 GC啦，今天我们就一起来了解下 Kubernetes 中的 GC。

你可能最先想到的就是容器的清理，即 Kubelet 侧的 GC，清理许多处于退出（Exited）状态的容器，



### Kubelet GC

GC 在 Kubelet 中非常重要，它不仅可以清理无用的容器，还可以清理未使用的镜像以达到节省空间的目的。当然 Kubelet 清理的这些容器都是 Kubernetes 自己创建的容器，你通过 Docker 手动创建的容器均不在 GC 的范围内，所以不必过于担心。

**Kubelet 会对容器每分钟执行一次 GC 操作，对容器镜像每 5 分钟执行一次 GC 操作**，这样可以保障 Kubelet 节点的稳定性，避免节点出现资源紧缺的情况。Kubelet 刚启动时并不会立即执行 GC 操作，而是在启动 1 分钟后开始执行第一次对容器的 GC 操作，启动 5 分钟后开始执行第一次对容器镜像的回收操作。这里建议你最好不用使用其他外部的 GC 工具，有可能会破坏 Kubelet 的 GC 逻辑。

目前 Kubelet 提供了 3 个参数，可以方便你调整容器镜像的 GC 参数：

- `--minimum-image-ttl-duration`表示一个镜像在清理前的最小存活时间；

- `--image-gc-high-threshold`表示磁盘使用率的上限阈值，默认值是 90%，即当磁盘使用率达到 90% 的时候会触发对镜像的 GC 操作；

- `--image-gc-low-threshold`表示磁盘使用率的下限阈值，默认值是 80%，即当磁盘使用率降到 80% 的时候，GC 操作结束。

对镜像的 GC 操作，就是逐个删除最久最少使用（Least Recently Used）的镜像。

对于容器的 GC 操作，Kubelet 也提供了 3 个参数供你使用调整：

- `--minimum-container-ttl-duration`表示已停止的容器在被清理之前最小的存活时间，**默认值是 1 分钟**，即容器停止超过 1 分钟才会被标记可被 GC 清理；

- `--maximum-dead-containers-per-container`表示一个 Pod 内可以保留的已停止的容器数量，**默认值是 2**。Kubernetes 是以 Pod 为单位进行容器管理的。有时候 Pod 内运行失败的容器，比如容器自身的问题，或者健康检查失败，会被 kubelet 自动重启，这将产生一些停止的容器；

- `--maximum-dead-containers`表示在本节点上可以保留的已停止容器的最大数量，**默认值是240**。毕竟这些容器也会消耗额外的磁盘空间，所以超过这个上限阈值后，就会触发 Kubelet 的 GC 操作，来帮你自动清理这些已停止的容器，释放磁盘空间。

当然，如果你想要关闭容器的 GC 操作，只需要将`--minimun-container-ttl-duration`**设置为0**，把`--maximum-dead-containers-per-container`和`--maximum-dead-containers`都**设置为负数**即可。

在有些场景中，容器的日志需要保留在本地，如果直接清理掉这些容器会丢失日志。所以这里我强烈建议你将`--maximum-dead-containers-per-container`**设置为一个足够大的值**，以便每个容器至少有一个退出的实例。这里，你就可以根据自己的场景进行配置。

提到的这些 flag，目前仍能继续使用，在未来的版本中，Kubernetes 会用新的 flag 进行替换，详见 [官方文档](https://kubernetes.io/zh/docs/concepts/cluster-administration/kubelet-garbage-collection/#deprecation)。

除了这些基本的 GC 以外，Kubernetes 内部也有很多操作对象，而且这些对象之间还存在着一定的“从属关系”，比如 Deployment 管理着 ReplicaSet。下面我们就来了解下 Kubernetes 内部对象的 GC。



### Kubernetes 内部对象的 GC

我们已经知道创建好一个 Deployment 以后，kube-controller-manager 会帮助我们创建对应的 ReplicaSet。这些 ReplicaSet 会自动跟我们创建的 Deployment 进行关联，那 Kubernetes 是怎么样维护这种从属关系的呢？

在 Kubernetes 中，每个对象都可以设置多个 OwnerReference，即该对象从属于谁。

我们先来看看 OwnerReference 的定义：

```go
// OwnerReference contains enough information to let you identify an owning
// object. An owning object must be in the same namespace as the dependent, or
// be cluster-scoped, so there is no namespace field.
type OwnerReference struct {
    // API version of the referent.
    APIVersion string `json:"apiVersion" protobuf:"bytes,5,opt,name=apiVersion"`
    // Kind of the referent.
    // More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds
    Kind string `json:"kind" protobuf:"bytes,1,opt,name=kind"`
    // Name of the referent.
    // More info: http://kubernetes.io/docs/user-guide/identifiers#names
    Name string `json:"name" protobuf:"bytes,3,opt,name=name"`
    // UID of the referent.
    // More info: http://kubernetes.io/docs/user-guide/identifiers#uids
    UID types.UID `json:"uid" protobuf:"bytes,4,opt,name=uid,casttype=k8s.io/apimachinery/pkg/types.UID"`
    // If true, this reference points to the managing controller.
    // +optional
    Controller *bool `json:"controller,omitempty" protobuf:"varint,6,opt,name=controller"`
    // If true, AND if the owner has the "foregroundDeletion" finalizer, then
    // the owner cannot be deleted from the key-value store until this
    // reference is removed.
    // Defaults to false.
    // To set this field, a user needs "delete" permission of the owner,
    // otherwise 422 (Unprocessable Entity) will be returned.
    // +optional
    BlockOwnerDeletion *bool `json:"blockOwnerDeletion,omitempty" protobuf:"varint,7,opt,name=blockOwnerDeletion"`
}
```

在 OwnerReference 中，我们可以确定该对象所“从属于”的对象，从而建立两者之间的从属关系。我们通过一个例子，直观了解下这个“从属”关系：

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  annotations:
    deployment.kubernetes.io/desired-replicas: "2"
    deployment.kubernetes.io/max-replicas: "3"
    deployment.kubernetes.io/revision: "1"
  creationTimestamp: "2020-09-03T07:22:35Z"
  generation: 1
  labels:
    k8s-app: kube-dns
    pod-template-hash: 5644d7b6d9
  name: coredns-5644d7b6d9
  namespace: kube-system
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: Deployment
    name: coredns
    uid: 37ae660a-dba8-4ff9-a152-7d6f420e624d
  resourceVersion: "1542272"
  selfLink: /apis/apps/v1/namespaces/kube-system/replicasets/coredns-5644d7b6d9
  uid: fa3d9859-43d4-484b-9716-7536243acd0f
spec:
  replicas: 2
  ...
status:
  ...
```

这里我截取了一个 ReplicaSet 中的 metadata 的部分。注意看这个 ReplicaSet 的`ownerReferences`字段标识了一个名为 coredns 的 Deployment 对象。

同样，我们来看看该 ReplicaSet 管理的 Pod。这里 ReplicaSet 的副本数是 2，我们任意选择其中一个 Pod：

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2020-09-03T07:22:35Z"
  generateName: coredns-5644d7b6d9-
  labels:
    k8s-app: kube-dns
    pod-template-hash: 5644d7b6d9
  name: coredns-5644d7b6d9-sz4qj
  namespace: kube-system
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: ReplicaSet
    name: coredns-5644d7b6d9
    uid: fa3d9859-43d4-484b-9716-7536243acd0f
  resourceVersion: "1542270"
  selfLink: /api/v1/namespaces/kube-system/pods/coredns-5644d7b6d9-sz4qj
  uid: c52d630b-1840-4502-88d1-b67bed2dd625
spec:
  ...
```

可以看到该 Pod 的`ownerReferences`指向刚才的 ReplicaSet，名字和 UID 都与之相对应。

至此通过观察这几个对象中的ownerReferences的信息，我们可以建立起如下的“从属关系”，即：

- Deployment（owner）—> ReplicaSet (dependent)；
- ReplicaSet (owner) —> Pod (dependent)。

了解了如上从属关系，我们后续就可以进行 GC 了。比如当你想彻底删除一个 Deployment 的时候，这时候 Kubernetes 会自动帮你把相关联的 ReplicaSet、Pod 等也一并删除掉，那么这种删除行为也称之为级联删除（Cascading Deletion），这也是 Kubernetes 默认的删除行为。

对于级联删除，Kubernetes 提供了两种模式，分别为后台（Background）模式和前台（Foreground）模式。

我们以后台级联删除 Deployment 为例。直观的体验就是，当你使用后台模式删除时，发送完请求，Kuberentes 会立即删除主对象，比如 Deployment，之后 Kubernetes 会在后台 GC 其附属的对象，比如 ReplicaSet。

而对于 [前台级联删除](https://kubernetes.io/zh/docs/concepts/workloads/controllers/garbage-collection/#%E5%89%8D%E5%8F%B0%E7%BA%A7%E8%81%94%E5%88%A0%E9%99%A4)，会先删除其所属的对象，然后再删除主对象。依然以 Deployment 为例，这时候主对象 Deployment 首先进入“删除中”的状态，虽然你依然能够通过 REST API 看到该 Deployment，但是该对象的`deletionTimestamp`字段被设置非空即“删除中”。

同时该对象的 `metadata.finalizers` 字段为 foregroundDeletion，有了这个 fianlizer 的存在，该对象就不会被删除，并会一直处于“删除中”的状态。

我们可以通过 `deleteOptions.propagationPolicy` 这个字段，来控制删除的策略，取值包括上面提到的三种方式，即 Orphan、Foreground 或者 Background。

如果我们使用 kubectl 进行后台删除的时候，可以通过如下命令进行操作：

```sh
$ kubectl proxy --port=8080
$ curl -X DELETE localhost:8080/apis/apps/v1/namespaces/default/replicasets/my-replicaset \
  -d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Background"}' \
  -H "Content-Type: application/json"
```

当然如果你使用的是 client-go 这类的库，也可以通过库中提供的函数，通过设置 DeleteOption 进行后台删除。
同样，我们也可以通过如下命令进行前台级联删除：

```sh
$ kubectl proxy --port=8080
$ curl -X DELETE localhost:8080/apis/apps/v1/namespaces/default/replicasets/my-replicaset \
  -d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Foreground"}' \
  -H "Content-Type: application/json"
```

这里如果你只想删除 ReplicaSet，但是并不像删除其关联的 Pod，你可以这么操作：

```sh
$ kubectl proxy --port=8080
$ curl -X DELETE localhost:8080/apis/apps/v1/namespaces/default/replicasets/my-repset \
  -d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Orphan"}' \
  -H "Content-Type: application/json"
```

kubectl 命令行在删除操作的时候，默认是进行级联删除的，如果你不想级联删除，可以这么操作：

```sh
$ kubectl delete replicaset my-repset --cascade=false
```



### 写在最后

Kubernetes 默认开启了 GC 的能力，不管是对于内部的各种 API 对象，还是对于 kubelet 节点上的冗余镜像以及退出的容器。这些默认的配置，已经基本上满足我们绝大多数的使用需要，不需要额外配置。当然你也可以通过调整一些参数和策略，实现自己的业务场景和逻辑。

如果要GC调整这些值，可以在下面的在文件当中进行配置：
 https://kubernetes.io/zh/docs/concepts/cluster-administration/kubelet-garbage-collection/







## 21 | 优先级调度：你必须掌握的 Pod 抢占式资源调度



随着我们在 Kubernetes 集群中部署越来越多的业务，势必要考虑集群的资源利用率问题。尤其是当集群资源比较紧张的时候，如果此时还要部署一些比较重要的关键业务，那么该如何去提前“抢占”集群资源，从而使得关键业务在集群中跑起来呢？

这里一个最常见的做法就是采用优先级方案。通过给 Pod 设置高优先级，让其比其他 Pod 显得更为重要，通过这种“插队”的方式优先获得调度器的调度。现在我们就来看看 Kubernetes 是如何定义这种优先级的。



### PriorityClass

Kubernetes 通过对象 PriorityClass 来定义上述优先级，我们可以通过之前说过的`kubectl create`命令来创建这个对象：

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "This priority class should be used for XYZ service pods only."
```

这里我们创建了就是一个名为 high-priority 的 PriorityClass 定义，使用的 API 版本是 scheduling.k8s.io/v1。这个对象是个集群级别的定义，并不属于任何 namespace，可以被全局使用，即各个 namespace 中创建的 Pod 都能使用 PriorityClass。我们在定义这样一个 PriorityClass 对象的时候，名字不可以包含 system- 这个前缀。

关于这个数值的设置，有几点要说明。系统自定义了两个优先级值：

```go
// HighestUserDefinablePriority is the highest priority for user defined priority classes. Priority values larger than 1 billion are reserved for Kubernetes system use.
HighestUserDefinablePriority = int32(1000000000)
// SystemCriticalPriority is the beginning of the range of priority values for critical system components.
SystemCriticalPriority = 2 * HighestUserDefinablePriority
```

那么当你自己创建 PriorityClass 的时候，定义的优先级值是不能高于这里的 HighestUserDefinablePriority 的， 即 1000000000 （10 亿）。

Kubernetes 在初始化的时候就自带了两个 PriorityClasses，即 system-cluster-critical 和 system-node-critical，优先级值为 2000000000。 你可以通过下面的命令查看到这两个最高优先级的 PriorityClasse：

```sh
$ kubectl get priorityclass
NAME                      VALUE        GLOBAL-DEFAULT   AGE
system-cluster-critical   2000000000   false            59d
system-node-critical      2000001000   false            59d
```

这是两个公共的优先级类，主要用来确保 Kubernetes 系统的关键组件或者关键插件总是能够优先被调度，比如 coredns 等。因为一旦这些关键组件、关键插件被驱逐，或者由于镜像需要升级，新建的 Pod 无法被调度，始终处于 Pending 的状态，这些都会导致整个集群异常，甚至停止工作。所以对于这些关键的组件，我们务必保证其优先级是最高的，这样才会被最先调度。我们可以来参考下 coredns 的做法，以下是截取的一部分配置：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  ...
  name: coredns
  namespace: kube-system
  ...
spec:
  ...
  template:
    ...
    spec:
      ...
      priorityClassName: system-cluster-critical
      ...
status:
  ...
```

下面我们来看看如何在 Pod 中使用 PriorityClass：通过 spec.priorityClassName 即可指定要使用的 PriorityClass：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
  priorityClassName: high-priority
```

通过`kubectl create`创建好该 Pod 后，我们查看详情可以看到 PriorityClass 的名字和优先级数值都被写入了 Pod 中：

```sh
$ kubectl describe pod nginx
Name:                 nginx
Namespace:            default
Priority:             1000000
Priority Class Name:  high-priority
```

注意，在使用的时候不允许自己去设置 spec.priority 的数值，我们只能通过 `spec.priorityClassName` 来设置优先级。现在，我们回过头再来观察下上面定义的这个 high-priority 的 PriorityClass 对象，里面有两个可选字段：globalDefault 和 description。

globalDefault 用来表明是否将该 PriorityClass 的数值作为默认值，并将其应用在所有未设置 priorityClassName 的 Pod 上。 既然是默认值，那么就意味着整个 Kubernetes 集群中只能存在一个 globalDefault 设为 true 的 PriorityClass 对象。

如果没有任何一个 PriorityClass 对象的 globalDefault 被设置为 true 的话，就意味着所有未设置 `spec.priorityClassName` 的 Pod 的优先级为 0。即便你新建了某个 PriorityClass 对象，并将其 globalDefault 设置为 true，那么也不影响集群中的存量 Pod，只会对新建的 Pod 生效。

你可以在 description 字段中对该 PriorityClass 进行一些使用上的解释说明。

现在我们就来做个优先级抢占的测试，使用一个单节点的集群做演示，该节点的 CPU 为 2 核。先创建一个优先级值更小的名为 low-priority 的 PriorityClass，其 YAML 文件如下：

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 1000
globalDefault: false
```

创建好了以后，通过kubectl get可以查看到创建的这两个 PriorityClass：

```sh
$ kubectl get priorityclass | grep -v system
NAME                      VALUE        GLOBAL-DEFAULT   AGE
high-priority             1000000      false            30m
low-priority              1000         false            8m35s
```

现在我们分别使用这两个 PriorityClass 创建两个 Pod。先来创建一个使用 low-priority 作为优先级的 Pod，下面是其对应的 YAML 文件：

```yaml
apiVersion: v1
kind: Pod
metadata:
 name: nginx-low-pc
spec:
 containers:
 - name: nginx
   image: nginx
   imagePullPolicy: IfNotPresent
   resources:
    requests:
     memory: "64Mi"
     cpu: "1200m"       #CPU需求设置较大
    limits:
     memory: "128Mi"
     cpu: "1300m"
 priorityClassName: low-priority     #使用低优先级
```

这里我们指定了一个较大比重的 CPU 资源。我们假定集群中的节点为 2 核，把 Pod 需要的最低 CPU 需求设置为1200m，超过了 2核 CPU 总算力的一半。

同样我们使用 high-priority 作为其优先级创建一个 Pod，其对应的 YAML 文件如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
 name: nginx-high-pc
spec:
 containers:
 - name: nginx
   image: nginx
   imagePullPolicy: IfNotPresent
   resources:
    requests:
     memory: "64Mi"
     cpu: "1200m"
    limits:
     memory: "128Mi"
     cpu: "1300m"
 priorityClassName: high-priority        #使用高优先级
```

如此一来，这两个 Pod 无法被调度到同一个只有 2 核 CPU 的节点上的，因为 2 个 Pod 加起来的 CPU 请求总量超过了 2 核。

我们可以先创建低优先级 Pod 命名为 nginx-low-pc ，可以看到该 Pod 已经成功运行，并消耗了超过一半的 CPU：

```sh
$ kubectl get pods
NAME           READY   STATUS    RESTARTS   AGE
nginx-low-pc   1/1     Running   0          22s
$ kubectl describe pod nginx-low-pc
...
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests     Limits
--------           --------     ------
  cpu                1220m (61%)  1300m (65%)       #Node的CPU使用率已经过半
  memory             64Mi (1%)    128Mi (3%)
  ephemeral-storage  0 (0%)       0 (0%)
```

现在我们再创建高优先级 Pod 命名为 nginx-high-pc，可以看到低优先级 Pod nginx-low-pc 由于高优先级的资源抢占被驱逐，运行状态也从 Runing 状态变为了 Terminating：

```sh
$ kubectl get pods
NAME            READY   STATUS        RESTARTS   AGE
nginx-high-pc   0/1     Pending       0          7s
nginx-low-pc    0/1     Terminating   0          87s
$ kubectl get pods
NAME            READY   STATUS    RESTARTS   AGE
nginx-high-pc   1/1     Running   0          12s
```

最终，高优先级 Pod 成功完成了抢占，整个节点上只有它在运行。当新 Pod 被创建后，它们就进入了调度器的队列，等待被调度。调度器会根据 PriorityClass 的优先级来挑选出优先级最高的 Pod，并且尝试把它调度到一个节点上。

如果这个时候没有任何一个节点能够满足这个 Pod的所有要求，包括资源、调度偏好等要求，就会触发抢占逻辑。调度器会尝试寻找一个节点，通过移除一个或者多个比该 Pod 的优先级低的 Pod， 尝试使目标 Pod 可以被调度。

如果找到了这样一个节点，那么运行在该节点上的一个或者多个低优先级的 Pod 就会被驱逐，这样这个高优先级的 Pod 就可以调度到目标节点上了。哈哈，是不是有点“鸠占鹊巢”的感觉！

当然对于某些场景，如果你并不希望 Pod 被驱逐掉，只是希望可以优先调度，那么你可以使用非抢占式的 PriorityClass。比如这是一个非抢占式的 PriorityClass 的配置：

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority-nonpreempting
value: 1000000
preemptionPolicy: Never
globalDefault: false
description: "This priority class will not cause other pods to be preempted."
```

这个配置跟我们上面的定义基本一致，只不过多了一个字段 `preemptionPolicy`。这样就可以避免属于该 PriorityClass 的 Pod 抢占其他 Pod。当然你也可以通过给 kube-scheduler 设置参数禁用抢占能力。但是关键组件的 Pod 还是依赖调度器这种抢占机制，以便在集群资源压力较大时可以优先得到调度。因此，不建议你禁用抢占能力。

kube-scheduler 的抢占能力是通过 disablePreemption 这个参数来控制的，该标志默认为 false。如果你还是坚持禁用抢占，可以将 disablePreemption 设置为true，目前这一参数的配置，只能通过 KubeSchedulerConfiguration 来设置，即：

```yaml
apiVersion: kubescheduler.config.k8s.io/v1alpha1
kind: KubeSchedulerConfiguration
algorithmSource:
  provider: DefaultProvider
...
disablePreemption: true
```



### 写在最后

提高集群的资源利用率最常见的做法就是采用优先级的方案。在实际使用的时候，要避免恶意用户创建高优先级的 Pod，这样会导致其他 Pod 被驱逐或者无法调度，严重影响集群的稳定性和安全性。集群管理员可以为特定用户创建特定优先级级别，来防止他们恶意使用高优先级的 PriorityClass。





## 22 | 安全机制：Kubernetes 如何保障集群安全？



Kubernetes 作为一个分布式集群的管理工具，提供了非常强大的可扩展能力，可以帮助你管理容器，实现业务的高可用性和弹性能力，保障业务的规模。现在也有越来越多的企业正在逐步将核心应用部署到 Kubernetes 集群中。

但是当业务规模扩大，集群的承载能力变大的时候，Kubernetes 平台自身的安全性就不得不考虑起来。

那么 Kubernetes 平台自身身的安全问题如何解决？我们又该采取什么的策略来保证我们业务应用的安全部署？



### Kubernetes 的安全性

[A Security Checklist for Cloud Native Kubernetes Environments](https://thenewstack.io/a-security-checklist-for-cloud-native-kubernetes-environments/) 这篇文章对 Kubernetes 的安全性总结得非常好，将它 Kubernetes 的安全性归纳为了以下四个方面： Infrastructure（基础设施）、Kubernetes 集群自身、Containers（容器）及其运行时和 Applications（业务应用）。

![img](D:\学习资料\笔记\k8s\k8s图\48151844-pasted-image-0-2-1024x589.png)

建议你详细阅读一下这篇文档，我们这里只是做一些简短的总结介绍。

Infrastructure，即基础设施层。正所谓“万丈高楼平地起”，基础设施的安全性是最基础的，也是最重要和关键的，却常常被忽略。这里基础设施，主要包括网络、存储、物理机、操作系统，等等。

Kubernetes 其实对用户屏蔽了底层的基础架构，所以我们在初期规划和设计网络的时候，要提前做好规划，比如同时支持基于第 2 层 VLAN 的分段和基于第 3 层 VXLAN 的分段，以隔离不同租户或不同应用程序之间的流量。

如果你的 Kubernetes 集群搭建在云上，社区也给出了 [云上 Kubernetes 集群的 9 大安全最佳实践供你参考](https://rancher.com/blog/2019/2019-01-17-101-more-kubernetes-security-best-practices/)，如果你使用了一些 Cloud Provider 也可以参考这篇文档。



### 最佳实践

在底下基础设施安全的情况下，我们下一步要增强安全性的就是 Kubernetes 集群自身了。我们主要详细来看看 Kubernetes 集群及业务层安全性的十个最佳实践。

#### 1. 集群版本更新及 CVE 漏洞

首先，也是最重要的，你要时刻关注社区 Kubernetes 的版本更新，以及披露的 [CVE 漏洞](https://www.cvedetails.com/vulnerability-list/vendor_id-15867/product_id-34016/Kubernetes-Kubernetes.html)，及时地把 CVE 的修复方案变更到你的集群中去。

同时你需要保证跟社区的版本不要太脱节，跟社区保持 1 到 2 个大版本的差异。



#### 2. 保护好 Etcd 集群

Etcd 中保存着整个 Kubernetes 集群中最重要的数据，比如 Pod 信息、Secret、Token 等。一旦这些数据遭到攻击，造成的影响面非常巨大。我们必须确保 Etcd 集群的安全。

对于部署 Etcd 集群的各个节点，我们应该被授予最小的访问权限，同时还要尽量避免这些节点被用作其他用途。由于 Etcd 对数据的读写要求很高，这里磁盘最好是 SSD 类型。

Etcd 集群要配置双向 TLS 认证（mTLS），用于 Etcd 各个节点之间的通信。同时 APIServer 对 Etcd 集群的访问最好也要基于 mTLS。通过 Kubeadm 搭建出来的集群，默认已经采取这种配置方式。



#### 3. 限制对 Kubernetes APIServer 的访问

APIServer 是整个 Kubernetes 的大脑，及流量入口，所有的数据都在此进行交互。Kubernetes 的安全机制也都是围绕着保护 APIServer 进行设计的，正如我们第 18 讲介绍的认证（Authentication）、鉴权（Authorization）和准入控制（Admission Control），这三大机制保护了 APIServer 的安全。

显而易见，APIServer 也必须得使用 TLS 进行通信，尽量不要开启不安全的 HTTP 方式，尤其是在云上的环境中，切记一定要关闭，你可以通过`--insecure-port=0`参数来关闭。

同时要避免使用 AlwaysAllow 这种鉴权模式，这种模式会允许所有请求。一般来说，我建议这么配置鉴权模式，即`--authorization-mode=RBAC,Node`。RBAC（基于角色的访问控制）会将传入的用户/组与一组绑定到角色的权限进行匹配。这些权限将 HTTP 请求的动作（GET，POST，DELETE）和 Kubernetes 内部各种资源对象，比如 Pod、Service 或者 Node，在命名空间或者集群范围内有机地结合起来。



#### 4. 减少集群的端口暴露

当然你还需要尽可能减少集群的端口暴露。如下是 Kubernetes 各组件需要的端口，在使用的时候，可以限制只允许固定节点对这些端口的访问。

![图片1.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F-j-c-AbItzAAERT0Op8K0385.png)

#### 5. 限制对 Kubelet 的访问

Kubelet 上运行着我们的业务容器，同时还暴露了 10250 端口，可以用来查询容器，支持容器 exec，获取容器日志等功能。因此在生产级别的集群，我们要启用 [Kubelet 身份验证和授权](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kubelet-authentication-authorization/)，同时关闭匿名访问`--anonymous-auth=false`。



#### 6. 开启审计能力

APIServer 支持对所有的请求进行 [审计](https://kubernetes.io/zh/docs/tasks/debug-application-cluster/audit/)（Audit），你可以通过`--audit-log-path`来指定用来写入审计事件的日志文件路径，默认是不指定的。通过这个开启审计能力，再结合 RBAC，我们可以对整个集群的请求访问进行详细的监控和分析。

这个审计功能提供了与安全相关的，按时间顺序排列的记录集，每一个请求都会有对应的审计事件，记录着：

- 发生了什么？
- 什么时候发生的？
- 谁触发的？
- 活动发生在哪个（些）对象上？
- 在哪观察到的？
- 它从哪触发的？
- 活动的后续处理行为是什么？

通过这些审计事件，集群管理员可以很容易地分析出集群内的安全风险，以采取应对策略。



#### 7. 通过 namespace 进行隔离

通过 namespace 来隔离工作负载和数据，比如区分不用的用户，不同的业务场景，以及一些关键的业务。你可以通过对这些 namespace 内的资源设置一些 RBAC 的规则，来进一步增强安全性。

![Drawing 3.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F-j3vOAFJcHAAF4VRsEQdc795.png)



#### 8. 使用网络策略进行限制

Kubernetes 提供了基于 namespace 的 [网络策略](https://kubernetes.io/zh/docs/tasks/administer-cluster/declare-network-policy/)（Network Policy），可以允许和限制不同 namespace 中的 Pod 互访。

网络策略通过 [网络插件](https://kubernetes.io/zh/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/) 来实现，所以你要使用这种能力，要选择支持 NetworkPolicy 的网络解决方案，比如 [Calico](https://kubernetes.io/zh/docs/tasks/administer-cluster/network-policy-provider/calico-network-policy/)、[Cilium](https://kubernetes.io/zh/docs/tasks/administer-cluster/network-policy-provider/cilium-network-policy/)、[Weave](https://kubernetes.io/zh/docs/tasks/administer-cluster/network-policy-provider/weave-network-policy/) 等。



#### 9. 容器镜像安全

容器的运行依赖容器的镜像，保证镜像安全是最基本的。要避免使用一些未知来源的镜像，同时自己构建镜像的时候，要使用安全的、确知的、官方的基础镜像。

同时，要定期地对镜像进行漏洞扫描，你可以使用 [harbor-scanner-aqua](https://github.com/aquasecurity/harbor-scanner-aqua)、[Clair](https://github.com/quay/clair) 等工具对你的镜像中心、镜像进行扫描，来查找已知的漏洞。

除此之外，我们还要限制特权容器，避免在容器镜像中使用 root 用户，防止特权升级。



#### 10. 应用程序的安全性

应用程序自身的安全性也很重要。如果你的应用程序要暴露给集群外部，可以使用 [入口控制器](https://kubernetes.io/zh/docs/concepts/services-networking/ingress-controllers/)（Ingress Controller），比如 [Nginx](https://github.com/kubernetes/ingress-nginx/blob/master/README.md)。

应用容器也要使用 TLS 进行通信，包括跟入口控制器之间的通信。不需要定期扫描源代码及依赖项，查找是否有新漏洞，还要持续测试你的应用程序，查看是否会受到攻击，比如 SQL 注入，DDoS 攻击等。



### 使用 kube-bench 对 Kubernetes 集群的安全性进行检测

介绍一个开源工具 [kube-bench](https://github.com/aquasecurity/kube-bench)，它能帮助你排查 95% Kubernetes 集群的配置风险。这里推荐你使用 kube-bench 对你的Kubernetes 集群进行一次全面的安全性检测，并根据反馈采取安全加固措施。

![Drawing 4.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl-j3xiAe4TMAAFscw-4NYk411.png)

这个工具使用起来也非常方便快捷，你可以参考 [官方文档](https://github.com/aquasecurity/kube-bench) 进行操作。



### 写在最后

在云原生时代，拥有强大的安全非常重要，我们不能掉以轻心。在一开始我们就应该尽量把安全性纳入开发周期，将安全风险消灭在萌芽状态。





## 23 | 最后的防线：怎样对 Kubernetes 集群进行灾备和恢复？



Kubernetes 隐藏了所有容器编排的复杂细节，让我们可以专注在应用本身，而无须过多关注如何去做部署和维护。此外，Kubernetes 还支持多副本，可以保证我们业务的高可用性。而对于集群本身而言，我们一样也要保证其高可用性，你可以参考官方文档：利用 [Kubeadm 来创建高可用集群](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/high-availability/)。

但是这些并不足以让我们高枕无忧，因为 Kubernetes 在帮助我们编排调度容器的同时，往往还保存了很多关键数据，比如集群自身关键数据、密钥、业务配置信息、业务数据等。我们在使用 Kubernetes 的时候，非常有必要进行灾备，防止出现操作失误（比如大规模误删除）、自然灾害、磁盘损坏无法修复、网络异常、机房断电等情况导致的数据丢失，严重时甚至会导致整个集群不可用。

所以在使用 Kubernetes 的时候，我们最好做个灾备以方便对集群进行恢复，回滚到早期的一个稳定的状态。



### Kubernetes 需要备份哪些东西

在对 Kubernetes 集群做备份之前，我们首先得知道要备份哪些东西。

我们从整个 Kubernetes 的架构为出发点，来看看整个集群的组件。 Kubernetes 的官方架构图，如下：

![Drawing 0.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F-qTFCAfuayAAHPVgKdC98338.png)

从上图可以看出，整个 Kubernetes 集群可分为Master 节点（左侧）和 Node 节点（右侧）。

在 Master 节点上，我们运行着 Etcd 集群以及 Kubernetes 控制面的几大组件，比如 kube-apiserver、kube-controller-manager、kube-scheduler 和 cloud-controller-manager（可选）等。

在这些组件中，除了 Etcd，其他都是无状态的服务。只要保证 Etcd 的数据正常，其他几个组件不管出现什么问题，我们都可以通过重启或者新建实例来解决，并不会受到任何影响。因此我们**只需要备份 Etcd 中的数据。**

看完了 Master 节点，我们再来看看 Node 节点。

Node 节点上运行着 kubelet、kube-proxy 等服务。Kubelet 负责维护各个容器实例，以及容器使用到的存储。为了保证数据的持久化存储，对于关键业务的关键数据，建议通过 PV（Persistent Volume）来保存和使用。鉴于这一点，我们还需要对 PV 进行备份。

如果是节点出现了问题，我们可以向集群中增加新的节点，替换掉有问题的节点。

看完 Kubernetes 的官方架构图之后，下面我们就来看看该如何备份 Etcd 中的数据和 PV。



### 对 Etcd 数据进行备份及恢复

Etcd 官方也提供了 [备份的文档](https://etcd.io/docs/v3.4.0/op-guide/recovery/)，这里总结了一些实际操作，以便你后续可以借鉴并进行手动的备份和恢复。命令行里面的一些证书路径以及 endpoint 地址需要根据自己的集群参数进行更改。实际操作代码如下：

```sh
# 0. 数据备份
$ ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
--cacert=/etc/kubernetes/pki/etcd/ca.crt \
--key=/etc/kubernetes/pki/etcd/peer.key \
--cert=/etc/kubernetes/pki/etcd/peer.crt \
snapshot save ./new.snapshot.db

# 1. 查看 etcd 集群的节点
$ ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \ 
--cacert=/etc/kubernetes/pki/etcd/ca.crt \ 
--cert=/etc/kubernetes/pki/etcd/peer.crt \ 
--key=/etc/kubernetes/pki/etcd/peer.key \
member list

# 2. 停止所有节点上的 etcd！（注意是所有！！）
## 如果是 static pod，可以听过如下的命令进行 stop
## 如果是 systemd 管理的，可以通过 systemctl stop etcd
$ mv /etc/kubernetes/manifests/etcd.yaml /etc/kubernetes/

# 3. 数据清理
## 依次在每个节点上，移除 etcd 数据
$ rm -rf /var/lib/etcd

# 4. 数据恢复
## 依次在每个节点上，恢复 etcd 旧数据
## 里面的 name，initial-advertise-peer-urls，initial-cluster=controlplane
## 等参数，可以从 etcd pod 的 yaml 文件中获取到。
$ ETCDCTL_API=3 etcdctl snapshot restore ./old.snapshot.db \
--data-dir=/var/lib/etcd \
--name=controlplane \
--initial-advertise-peer-urls=https://172.17.0.18:2380 \
--initial-cluster=controlplane=https://172.17.0.18:2380

# 5. 恢复 etcd 服务
## 依次在每个节点上，拉起 etcd 服务
$ mv /etc/kubernetes/etcd.yaml /etc/kubernetes/manifests/
$ systemctl restart kubelet
```

上述这些备份，都需要手动运行命令行进行操作。如果你的 Etcd 集群是运行在 Kubernetes 集群中的，你可以通过以下的定时 Job (CronJob) 来帮你自动化、周期性（如下的 YAML 文件中会每分钟对 Etcd 进行一次备份）地备份 Etcd 的数据。自动备份代码如下：

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: backup
  namespace: kube-system
spec:
  # activeDeadlineSeconds: 100
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            # Same image as in /etc/kubernetes/manifests/etcd.yaml
            image: k8s.gcr.io/etcd:3.2.24
            env:
            - name: ETCDCTL_API
              value: "3"
            command: ["/bin/sh"]
            args: ["-c", "etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt --key=/etc/kubernetes/pki/etcd/healthcheck-client.key snapshot save /backup/etcd-snapshot-$(date +%Y-%m-%d_%H:%M:%S_%Z).db"]
            volumeMounts:
            - mountPath: /etc/kubernetes/pki/etcd
              name: etcd-certs
              readOnly: true
            - mountPath: /backup
              name: backup
          restartPolicy: OnFailure
          hostNetwork: true
          volumes:
          - name: etcd-certs
            hostPath:
              path: /etc/kubernetes/pki/etcd
              type: DirectoryOrCreate
          - name: backup
            hostPath:
              path: /data/backup
              type: DirectoryOrCreate
```



### 对 PV 的数据进行备份

对于 PV 来讲，备份就比较麻烦了。Kubernetes 自身不提供存储能力，它依赖各个存储插件对存储进行管理和使用。因此对于存储的备份操作，尤其是 PV 的备份操作，我们需要依赖各个云提供商的 API 来做 snapshot。

但是上述对于 Etcd 和 PV 的备份操作并不是很方便，我推荐你通过 [Velero](https://github.com/vmware-tanzu/velero) 来备份 Kubernetes。Velero 功能强大，但是操作起来很简单，它可以帮你做到以下 3 点：

1. 对 Kubernets 集群做备份和恢复。

2. 对集群进行迁移。

3. 对集群的配置和对象进行复制，比如复制到其他的开发和测试集群中去。

而且 Velero 还提供针对单个 Namespace 进行备份的能力，如果你只想备份某些关键的业务和数据，这是一个十分方便的功能。

说了这么多，下面我们来看看 Velero 是如何备份 Kubernetes 的。



### 使用 Velero 对 Kubernetes 进行备份

这是 Velero 的架构图：

![Drawing 1.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl-qTK6ADCLwAAFaThL2Fxk754.png)

Velero 由两部分组成：

- **一个命令行客户端**，你可以运行在本地，通过命令行完成对 Etcd 以及 PV 的备份操作；你也可以像使用 kubectl 操作 Kubernetes 资源一样备份 Kubernetes。

- **一个运行在 kubernetes 集群中的服务**（BackupController），负责执行具体的备份和恢复操作。

我们来看看具体使用时的流程：

1. 通过本地 Velero 客户端发送备份命令，比如图中的`velero backup create test-project-s2i --include-namespaces test`，这条命令会向 APIServer 中创建一个 Backup 对象。

2. BackupController 会去监测并验证这个 Backup 对象的合法性，比如参数的定义。

3. BackupController 通过向 APIServer 查询相关数据开始备份工作。

4. BackupController 将查询到的数据备份到远端的对象存储中。

Velero 在 Kubernetes 集群中创建了很多 CRD （Custome Resource Definition）以及相关的控制器，通过这些进行备份恢复等操作。因此，对集群的备份和恢复，实质上是对这些相关 CRD 的操作。BackupController 会根据 CRD 来确定该进行何种操作。

Velero 支持两种关于后端存储的 CRD，分别是 BackupStorageLocation 和 VolumeSnapshotLocation。

- BackupStorageLocation 主要用来定义 Kubernetes 集群资源的数据存放位置，也就是集群对象数据，而不是 PVC 和 PV 的数据。你可以从这个 [支持列表](https://velero.io/docs/main/supported-providers/) 里面找到目前官方和第三方支持的后端存储服务，主要是以支持 S3 兼容的存储为主，比如 AWS S3、阿里云 OSS、Minio 等。
- VolumeSnapshotLocation 主要用来给 PV 做快照，快照功能通常由 Amazon EBS Volumes、Azure Managed Disks、Google Persistent Disks 等云厂商提供，你可以根据需要选择使用各个云厂商的服务。或者你使用专门的备份工具 [Restic](https://github.com/restic/restic)，把 PV 数据备份到 [Azure Files](https://docs.microsoft.com/en-us/azure/aks/azure-files-dynamic-pv)、阿里云 OSS 中去。阿里云目前已经提供了 [基于 Velero 的插件](https://github.com/AliyunContainerService/velero-plugin)。

除此之外，BackupController 在工作过程中，还会创建其他的 CRD，主要用于内部的逻辑处理。你可以参考 [阿里云的文档](https://developer.aliyun.com/article/726863) 进一步学习。

如果你没有阿里云的 OSS，或者集群是线下的内部集群，你也可以自行搭建 Minio，作为对象存储服务来代替阿里云的 OSS。你可以参考 [官方的文档](https://velero.io/docs/main/contributions/minio/) 进行详细的安装配置工作。



### 写在最后

在分布式的世界里，我们很难保证万无一失。当你在 Kubernetes 集群中部署越来越多的业务的时候，对集群和数据的灾备是非常有必要的。在今年 7 月份，我们常用的代码托管平台 Github 就发生了 Kubernetes 故障 ，导致了持续 4 个半小时的严重故障。所以，我建议对于关键性的业务数据，要记得时常备份。









# 知其所以然：底层核心原理及可扩展性



## 24 | 调度引擎：Kubernetes 如何高效调度 Pod？



### Kubernetes 调度器工作原理简介

kube-scheduler 作为 Kubernetes 的调度器，它的主要任务就是给新创建的 Pod 或者是未被调度的 Pod 挑选一个合适的节点供 Pod 运行，满足 Pod 对资源等的要求。这样对应节点上的 Kubelet 就可以监听到该 Pod，并将其创建、运行。

整个调度过程听起来很简单，但是要考虑到的问题其实有很多，比如优先级、资源高效利用、高性能等。

- **优先级**。高优先级的 Pod 肯定要优先被调度。

- **资源高效利用**。比如我们要避免 Pod 都被调度到一个或某几个节点上，造成节点负载太大；或是避免同一个工作负载（如 Deployment）的几个副本，跑在同一个节点上，以免这个节点宕机对整个业务造成影响。

- **高性能**。我们需要支持快速地完成大规模 Pod 的调度工作，这样才能够支撑大规模的集群。

- 可扩展性强。方便用户自己增加调度逻辑，可以参考 [官方文档](https://kubernetes.io/zh/docs/concepts/scheduling-eviction/scheduling-framework/) 中的内容。

- ……

总的来说，调度的过程主要分为两个大步骤。

1. **过滤一些不满足条件的节点**，这个过程也称为 Predict。

2. **调度器会对这些合适的节点进行打分排序，从中选择一个最优的节点**，这个过程也称为 Priority。

这里面其实包含了很多的调度策略，可以阅读这份 [调度策略列表](https://kubernetes.io/zh/docs/reference/scheduling/policies/)，了解各个策略对应的含义。

在实际使用的过程中，你可以直接使用调度器的默认配置，不需要对其做过多的定制化。当然，如果你有特殊的需求，也可以构建自己的调度器，具体可以参考 [这份文档](https://kubernetes.io/zh/docs/reference/scheduling/config/) 来更改默认调度器的调度策略、调度插件以及调度行为。

下面我们主要来认识一下调度器都有哪些高级特性。



### 调度器的高级特性

调度器的高级特性有 NodeName 和 NodeSelector、亲和性和反亲性、污点和容忍，我们依次来了解一下。

#### NodeName 和 NodeSelector

首先是 NodeName 和 NodeSelector。

我们可以通过 `spec.nodeName` 强制约束在某个指定的 Node 上运行 Pod，如下所示：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-nodename
  namespace: demo
spec:
  nodeName: node1 #指定调度节点 node1 上
  containers:
  - name: nginx-demo
    image: nginx:1.19.4
```

上面这个 Pod 就被约束在 node1 上。通过这种方式指定节点，会跳过 kube-scheduler 的调度逻辑，即**不需要经过调度**。

除了这种强制指定节点的方式，我们还可以通过 NodeSelector 的方式来选择节点。调度器的调度策略 MatchNodeSelector 会匹配 Node 的 label，从而达到节点筛选的目的。比如下面这个例子：

```sh
# 我们先对节点进行打标
$ kubectl label nodes node1 abc.com/role=dev

# 通过如下命令可以查看该节点目前的所有 label
$ kubectl get node node1 --show-labels
NAME    STATUS   ROLES    AGE   VERSION          LABELS
node1   Ready    master   75d   v1.16.6-beta.0   abc.com/role=dev,beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=docker-desktop,kubernetes.io/os=linux,node-role.kubernetes.io/master=
```

对节点打好标以后，就可以通过 `spec.nodeSelector` 来将 Pod 调度到带有指定 label 标记的节点上。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-nodename
  namespace: demo
spec:
  nodeSelector:
    abc.com/role: dev #指定调度到带有 abc.com/role=dev 这种 label 标记的节点上
  containers:
  - name: nginx-demo
    image: nginx:1.19.4
```

nodeSelector 提供了一种非常简单的方法，方便我们将 Pod 约束到带有特定 label 的节点上。

除此之外，kube-scheduler 还提供了更加动态的方式，即亲和性和反亲和性，可以帮助我们完成更高级的 Pod 调度，比如将某些 Pod 都调度到某个节点上。

下面我们就来认识一下亲和性和反亲和性。



#### 亲和性和反亲和性

Kubernetes 提供了如下 3 种类型：

- nodeAffinity（节点亲和性）；

- podAffinity（Pod 亲和性）；

- podAntiAffinity（Pod 反亲和性）。

这 3 种亲和性和反亲和性策略支持更广泛的操作符，差异如下表所示：

![image (1).png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F-0wO2AaNBNAAB8jlKE3Qo120.png)

对于上述的亲和性和反亲和性功能，每种都有 3 种规则可以设置。

- `RequiredDuringSchedulingRequiredDuringExecution`：在 Pod 调度期间要求满足亲和性或者反亲和性的规则要求。如果不能满足这些指定的规则，那该 Pod 不能被调度到对应的主机上。而且在之后的运行过程中，如果因为某些原因（比如 label 被修改了）导致规则不再满足了，系统就会尝试把该 Pod 从主机上驱逐掉。

- `RequiredDuringSchedulingIgnoredDuringExecution`：在 Pod 调度期间要求满足亲和性或者反亲和性规则。如果不能满足的话，那么该 Pod 不能被调度到对应的主机上。在之后的运行过程中，系统也不会再去检查这些规则是否还继续满足。

- `PreferredDuringSchedulingIgnoredDuringExecution`：在 Pod 调度期间要尽量地指定的亲和性和反亲和性规则。即使不能满足，Pod 也有可能被调度到对应的主机上。在之后的运行过程中，系统也不会再去检查这些规则是否继续满足。

我们这里来说说具体的使用场景。

对于 **nodeAffinity**，主要有两个使用场景：

- 帮助我们将一个工作负载的所有 Pod 部署到指定的 label 的主机上，这点和 nodeSelector 是类似的；帮助我们将 Pod 部署到不带有特定 label 的主机上，即 Notin，比如不在 Master 节点上部署该 Pod。

下面是一个 [官方](https://kubernetes.io/zh/docs/concepts/scheduling-eviction/assign-pod-node/) 使用的 nodeAffinity 的例子：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/e2e-az-name
            operator: In
            values:
            - e2e-az1
            - e2e-az2
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
  containers:
  - name: with-node-affinity
    image: k8s.gcr.io/pause:2.0
```

这个 Pod 指定必须运行在 label 带有 `kubernetes.io/e2e-az-name=e2e-az1` 或 `kubernetes.io/e2e-az-name=e2e-az2` 的节点上。如果没有任何一个节点有这些 label，则该 Pod 不会被调度。同时这些节点上最好还带有 `another-node-label-key=another-node-label-vale` 的标签。

对于 podAffinity 和 podAntiAffinity，你可以基于已经在节点上运行的 Pod 的标签来约束新 Pod 可以调度到的节点，而不是基于节点上的标签。

如下是一个 [官方](https://kubernetes.io/zh/docs/concepts/scheduling-eviction/assign-pod-node/) 使用的 podAffinity 和 podAntiAffinity 的例子：

```yaml
a piVersion: v1
kind: Pod
metadata:
  name: with-pod-affinity
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: security
            operator: In
            values:           
            - S1
        topologyKey: topology.kubernetes.io/zone
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: security
              operator: In
              values:
              - S2
          topologyKey: topology.kubernetes.io/zone
  containers:
  - name: with-pod-affinity
    image: k8s.gcr.io/pause:2.0
```

上述这个例子中使用了 podAffinity 和 podAntiAffinity。其中亲和性这块的规则表示该 Pod 必须部署在一个节点上，这个节点上至少有一个处于正在运行状态的带有 `security=s1` 标签的 Pod，并且要求部署的节点同正在运行的 Pod 所在节点都在相同的云服务区域中，也就是 `topologyKey:topology.kubernetes.io/zone`。

换言之，一旦某个区域出了问题，我们希望这些 Pod 能够再次迁移到同一个区域。当然， topologyKey 可以是任何合法的标签键，比如 kubernetes.io/hostname，你可以参考 [官方文档](https://kubernetes.io/zh/docs/concepts/scheduling-eviction/assign-pod-node/#pod-%E4%BD%BF%E7%94%A8-pod-%E4%BA%B2%E5%92%8C-%E7%9A%84%E7%A4%BA%E4%BE%8B)，看看这个值在使用上的限制。

对于例子中 Pod 反亲和性规则，表示该 Pod 不希望部署在一个运行着带有 label 为 security=s2 的 Pod 的节点上。

除了在 Pod 层面进行限制外，我们还可以对 Node 进行操作，可以禁止某些 Pod 调度上来。



#### 污点和容忍（Taints and Tolerations）

最后我们来看污点和容忍（Taints and Tolerations）。

我们可以给节点设置污点，通过这个污点就可以避免 Pod 调度上来，除非在 Pod 上设置了污点容忍。

每个污点的组成如下：

```sh
key=value:effect
```

每个污点规则都有一个 key 和 value，其中 value 可以为空，effect 描述污点的作用。当前 taint effect 支持如下 3 个选项：

- NoSchedule 表示不会将 Pod 调度到带该污点的 Node 上；

- PreferNoSchedule 表示尽量避免将 Pod 调度到带该污点的 Node 上；

- NoExecute 表示不会将 Pod 调度到带有该污点的 Node 上，同时会将 Node 上已经运行中的 Pod 驱逐出去。

我们使用 kubectl 命令就可以快速地设置和去除污点，命令如下：

```sh
# 设置污点
$ kubectl taint nodes node1 key1=value1:NoSchedule

# 去除污点
$ kubectl taint nodes node1 key1:NoSchedule-
```

我们可以在 Pod 上设置容忍（Toleration），这样就可以将 Pod 调度到存在污点的 Node 上。在 Pod 的 spec 中设置 tolerations 字段可以给 Pod 设置上容忍点 Toleration，如下所示：

```json
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"
  tolerationSeconds: 3600
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
- key: "key2"
  operator: "Exists"
  effect: "NoSchedule"
```

其中 key、vaule、effect 要与 Node 上设置的 taint 保持一致；operator 的值为 Exists 将会忽略 value 值；tolerationSeconds 用于描述当 Pod 需要被驱逐时可以在 Pod 上继续保留运行的时间。



### 写在最后

这一讲我带你了解了 Kubernetes 调度器的工作原理以及调度器的一些高级特性，也介绍了 Kubernetes 是如何高效调度 Pod 的。你可以在实际使用中慢慢体会调度器的这些高级特性。





## 25 | 稳定基石：带你剖析容器运行时以及 CRI 原理



当一个 Pod 在 Kube-APIServer 中被创建出来以后，会被调度器调度，然后确定一个合适的节点，最终被这个节点上的 Kubelet 拉起，以容器状态运行。

那么 Kubelet 是如何跟容器打交道的呢，它是如何进行创建容器、获取容器状态等操作的呢？



### 容器运行时 （Container Runtime）

Kubelet 负责运行具体的 Pod，并维护其整个生命周期，为 Pod 提供存储、网络等必要的资源。但 Kubelet 本身并不负责真正的容器创建和逻辑管理，这些全部都是通过容器运行时（Container Runtime）完成的。大家平常熟知的 Docker 其实就是一种容器运行时，除此之外，还有[containerd](https://kubernetes.io/zh/docs/setup/production-environment/container-runtimes/#containerd)、[cri-o](https://kubernetes.io/zh/docs/setup/production-environment/container-runtimes/#cri-o)、[kata](https://katacontainers.io/)、[gVisor](https://gvisor.dev/) 等等。

下图就是 Kubelet 跟[容器运行时进行的交互图](https://www.threatstack.com/blog/diving-deeper-into-runtimes-kubernetes-cri-and-shims)：

![Drawing 0.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl-3Y6CAIjzjAAEbwUIQ2pI143.png)

Kubelet 负责跟 kube-apiserver 进行数据同步，获取新的 Pod，并上报本机 Pod 的各个状态数据。Kubelet 通过调用容器运行时的接口完成容器的创建、容器状态的查询等工作。下图就是使用 Docker 作为容器的运行时。

![Drawing 1.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F-3Y8OAGglUAAESe6PzHHQ855.png)

Docker 作为 Kubelet 内置支持的主要容器运行时，也是目前使用最为官方的容器运行时之一。

除了 Docker，在 Kubernetes v1.5 之前，Kubelet 还内置了对 rkt 的支持。在这个阶段，如果我们想要自己去定义容器运行时，或者更改容器运行时的部分逻辑行为，是非常痛苦的，需要通过修改 Kubelet 的代码来实现。这些改动如果更新到上游社区，也会给社区造成很大的困扰，毕竟 Kubelet 自身的稳定性关乎着整个集群的稳定性。因此，这些改动在上游社区的合并通常都很谨慎，往往就需要开发者自己维护这些代码，维护成本非常高，也不方便升级。

介于这一点，很多开发者都希望 Kubernetes 可以支持更多的容器运行时。因此，从 v1.5 版本开始，社区引入了 [CRI](https://kubernetes.io/blog/2016/12/container-runtime-interface-cri-in-kubernetes/)（Container Runtime Interface）来解决这个问题。

CRI 接口的引入带来了两个好处：一是它很好地将 Kubelet 与容器运行时进行了解耦，这样我们每次对容器运行时进行更新升级等操作时，都再不需要对 Kubelet 做任何的更改；二是解放了 Kubelet，减少了 Kubelet 的负担，能够保证 Kubernetes 的代码质量和整个系统的稳定性。

下面我们就来了解一下CRI。



### CRI

CRI 接口可以分为两部分。

一个是容器运行时服务 RuntimeService，它主要负责管理 Pod 和容器的生命周期，比如创建容器、删除容器、查询容器状态等等。下面就是用[Protocol Buffers](https://developers.google.com/protocol-buffers)

定义的 RuntimeService 的接口：

```go
// Runtime service defines the public APIs for remote container runtimes
service RuntimeService {
    // Version returns the runtime name, runtime version, and runtime API version.
    rpc Version(VersionRequest) returns (VersionResponse) {}
    // RunPodSandbox creates and starts a pod-level sandbox. Runtimes must ensure
    // the sandbox is in the ready state on success.
    rpc RunPodSandbox(RunPodSandboxRequest) returns (RunPodSandboxResponse) {}
    // Start a sandbox pod which was forced to stop by external factors.
    // Network plugin returns same IPs when input same pod names and namespaces
    rpc StartPodSandbox(StartPodSandboxRequest) returns (StartPodSandboxResponse) {}
    // StopPodSandbox stops any running process that is part of the sandbox and
    // reclaims network resources (e.g., IP addresses) allocated to the sandbox.
    // If there are any running containers in the sandbox, they must be forcibly
    // terminated.
    // This call is idempotent, and must not return an error if all relevant
    // resources have already been reclaimed. kubelet will call StopPodSandbox
    // at least once before calling RemovePodSandbox. It will also attempt to
    // reclaim resources eagerly, as soon as a sandbox is not needed. Hence,
    // multiple StopPodSandbox calls are expected.
    rpc StopPodSandbox(StopPodSandboxRequest) returns (StopPodSandboxResponse) {}
    // RemovePodSandbox removes the sandbox. If there are any running containers
    // in the sandbox, they must be forcibly terminated and removed.
    // This call is idempotent, and must not return an error if the sandbox has
    // already been removed.
    rpc RemovePodSandbox(RemovePodSandboxRequest) returns (RemovePodSandboxResponse) {}
    // PodSandboxStatus returns the status of the PodSandbox. If the PodSandbox is not
    // present, returns an error.
    rpc PodSandboxStatus(PodSandboxStatusRequest) returns (PodSandboxStatusResponse) {}
    // ListPodSandbox returns a list of PodSandboxes.
    rpc ListPodSandbox(ListPodSandboxRequest) returns (ListPodSandboxResponse) {}
    // CreateContainer creates a new container in specified PodSandbox
    rpc CreateContainer(CreateContainerRequest) returns (CreateContainerResponse) {}
    // StartContainer starts the container.
    rpc StartContainer(StartContainerRequest) returns (StartContainerResponse) {}
    // StopContainer stops a running container with a grace period (i.e., timeout).
    // This call is idempotent, and must not return an error if the container has
    // already been stopped.
    // TODO: what must the runtime do after the grace period is reached?
    rpc StopContainer(StopContainerRequest) returns (StopContainerResponse) {}
    // RemoveContainer removes the container. If the container is running, the
    // container must be forcibly removed.
    // This call is idempotent, and must not return an error if the container has
    // already been removed.
    rpc RemoveContainer(RemoveContainerRequest) returns (RemoveContainerResponse) {}
    // PauseContainer pauses the container.
    rpc PauseContainer(PauseContainerRequest) returns (PauseContainerResponse) {}
    // UnpauseContainer unpauses the container.
    rpc UnpauseContainer(UnpauseContainerRequest) returns (UnpauseContainerResponse) {}
    // ListContainers lists all containers by filters.
    rpc ListContainers(ListContainersRequest) returns (ListContainersResponse) {}
    // ContainerStatus returns status of the container. If the container is not
    // present, returns an error.
    rpc ContainerStatus(ContainerStatusRequest) returns (ContainerStatusResponse) {}
    // UpdateContainerResources updates ContainerConfig of the container.
    rpc UpdateContainerResources(UpdateContainerResourcesRequest) returns (UpdateContainerResourcesResponse) {}
    // ReopenContainerLog asks runtime to reopen the stdout/stderr log file
    // for the container. This is often called after the log file has been
    // rotated. If the container is not running, container runtime can choose
    // to either create a new log file and return nil, or return an error.
    // Once it returns error, new container log file MUST NOT be created.
    rpc ReopenContainerLog(ReopenContainerLogRequest) returns (ReopenContainerLogResponse) {}
    // ExecSync runs a command in a container synchronously.
    rpc ExecSync(ExecSyncRequest) returns (ExecSyncResponse) {}
    // Exec prepares a streaming endpoint to execute a command in the container.
    rpc Exec(ExecRequest) returns (ExecResponse) {}
    // Attach prepares a streaming endpoint to attach to a running container.
    rpc Attach(AttachRequest) returns (AttachResponse) {}
    // PortForward prepares a streaming endpoint to forward ports from a PodSandbox.
    rpc PortForward(PortForwardRequest) returns (PortForwardResponse) {}
    // ContainerStats returns stats of the container. If the container does not
    // exist, the call returns an error.
    rpc ContainerStats(ContainerStatsRequest) returns (ContainerStatsResponse) {}
    // ListContainerStats returns stats of all running containers.
    rpc ListContainerStats(ListContainerStatsRequest) returns (ListContainerStatsResponse) {}
    // UpdateRuntimeConfig updates the runtime configuration based on the given request.
    rpc UpdateRuntimeConfig(UpdateRuntimeConfigRequest) returns (UpdateRuntimeConfigResponse) {}
    // Status returns the status of the runtime.
    rpc Status(StatusRequest) returns (StatusResponse) {}
}
```

另一个部分是镜像服务 ImageService，主要负责容器镜像的生命周期管理，比如拉取镜像、删除镜像、查询镜像等等，如下所示：

```go
// ImageService defines the public APIs for managing images.
service ImageService {
    // ListImages lists existing images.
    rpc ListImages(ListImagesRequest) returns (ListImagesResponse) {}
    // ImageStatus returns the status of the image. If the image is not
    // present, returns a response with ImageStatusResponse.Image set to
    // nil.
    rpc ImageStatus(ImageStatusRequest) returns (ImageStatusResponse) {}
    // PullImage pulls an image with authentication config.
    rpc PullImage(PullImageRequest) returns (PullImageResponse) {}
    // RemoveImage removes the image.
    // This call is idempotent, and must not return an error if the image has
    // already been removed.
    rpc RemoveImage(RemoveImageRequest) returns (RemoveImageResponse) {}
    // ImageFSInfo returns information of the filesystem that is used to store images.
    rpc ImageFsInfo(ImageFsInfoRequest) returns (ImageFsInfoResponse) {}
}
```

每一个容器运行时都需要自己实现一个 CRI shim，即完成对 CRI 这个抽象接口的具体实现。这样容器运行时就可以接收来自 Kubelet 的请求。

我们现在就来看看有了 CRI 接口以后，Kubelet 是如何和容器运行时进行交互的，见下图：

![Drawing 2.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F-3ZBCAfnwJAACHtbND3KI539.png)

从上图可以看出，新增的 CRI shim 是 Kubelet 和容器运行时之间的交互纽带，Kubelet 只需要跟 CRI shim 进行交互。Kubelet 调用 CRI shim 的接口，CRI shim 响应请求后会调用底层的运行容器时，完成对容器的相关操作。

这里我们需要将 Kubelet、CRI shim 以及容器运行时都部署在同一个节点上。一般来说，大多数的容器运行时都默认实现了 CRI 的接口，比如 [containerd](https://containerd.io/docs/)。

目前 Kubelet 内部内置了对 Docker 的 [CRI shim](https://dzone.com/articles/evolution-of-k8s-worker-nodes-cri-o) 的实现，见下图：

![Drawing 4.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F-3ZBmAdEVFAAAnf6SSCkk798.png)

而对于其他的容器运行时，比如[containerd](https://kubernetes.io/zh/docs/setup/production-environment/container-runtimes/#containerd)，我们就需要配置 kubelet 的 `--container-runtime` 参数为 remote，并设置 `--container-runtime-endpoint` 为对应的容器运行时的监听地址。

Kubernetes 自 v1.10 版本已经完成了和 containerd 1.1版本 的 GA 集成，你可以直接按照 [这份文档](https://kubernetes.io/zh/docs/setup/production-environment/container-runtimes/#containerd) 来部署 containerd 作为你的容器运行时。

![Drawing 5.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl-3ZCKAA7C9AABJ2r60MV4161.png)

containerd 1.1 版本已经内置了对 CRI 的实现，比直接使用 Docker 的性能要高很多。



### 写在最后

Kubernetes 作为容器编排调度领域的事实标准，其优秀的架构设计还体现在其可扩展接口上。比如 CRI 提供了简单易用的扩展接口，方便各个容器运行时跟 Kubelet 进行交互接入，极大地方便了用户进行定制化。

通过 CRI 对容器运行时进行抽象，我们无须修改 Kubelet 就可以天然地支持多种容器运行时，这极大地方便了开发者的对接，也减少了升级和维护成本。CRI 的出现也促进了容器运行时的繁荣，也为强隔离、多租户等复杂的场景带来了更多的选择。

除了 CRI 以外，在 Kubernetes 中还可以为不同的 Pod 设置不同的容器运行时（Container Runtime），以提供性能与安全性之间的平衡。从 1.12 版本开始，Kuberentes 就提供了 **RuntimeClass** 来实现这个功能。你可以阅读 [这份官方文档](https://kubernetes.io/zh/docs/concepts/containers/runtime-class/)，来学习如何使用 RuntimeClass。





## 26 | 网络插件：Kubernetes 搞定网络原来可以如此简单？



理论上，Kubernetes 可以跑在任何环境中，比如公有云、私有云、物理机、虚拟机、树莓派，但是任何基础设施（Infrastructure）对网络的需求都是最基本的。网络同时也是 Kubernetes 中比较复杂的一部分。

我们今天就来聊聊 Kubernetes 中的网络，先来看看 Kubernetes 中定义的网络模型。



### Kubernetes 网络模型

Kubernetes 的网络模型在设计的时候就遵循了一个基本原则，即**每一个 Pod 都拥有一个自己的独立的 IP 地址，且这些 Pod 之间可以不通过任何 NAT 互相连通。**

基于这样一个 IP-per-Pod 的方式，用户在使用的时候就不需要再额外考虑如何建立 Pod 之间的连接，也不用考虑如何将容器的端口映射到主机端口，毕竟动态地分配映射端口不仅会给整个系统增加很大的复杂度，还不方便服务间做服务发现。而且由于主机端口的资源也是有限的，端口映射还会给集群的可扩展性带来很大挑战。

每个 Pod 有了独立的 IP 地址以后，通信双方看到的 IP 地址就是对方实际的地址，即不经过 NAT。不管 Pod 是否在同一个宿主机上，都可以直接基于对方 Pod 的 IP 进行访问。

基于这样的一个网络模型，我们来看看 Kubernetes 中的如下 4 种场景是如何进行网络通信的：

- Pod 内容器之间的网络通信；
- Pod 之间的网络通信；
- Pod 到 Service 之间的网络通信；
- 集群外部与内部组件之间的网络通信。



#### 同一 Pod 内的网络通信

Kubernetes 会为每一个 Pod 创建独立的网络命名空间（netns），Pod 内的容器共享同一个网络命名空间。这也就是说，Pod 内的所有容器共享相同的网络设备、路由表设置、端口等等，就像在同一个主机上运行不同的程序一样，这些容器之间可以通过 localhost 直接访问彼此的端口。



#### Pod 之间的网络通信

Pod 间的相互网络通信，可以分为同主机通信和跨主机通信。

我们先来看看同主机上 Pod 间的通信。

![Drawing 0.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F-8tLuAWX-tAACIXQBtkeo535.png)

在网络命名空间里，每个 Pod 都有各自独立的网络堆栈，通过一对虚拟以太网对（virtual ethernet pair）连接到根网络命名空间里（root netns）。这个 veth pair 一端在 Pod 的 netns 内，另一端在 root netns 里，对应上图中的 eth0 和 vethxxx。

从上图可以看到，Pod1 和 Pod2 都通过 veth pair 连接到同一个 cbr0 的网桥上，它们的 IP 地址都是从 cbr0 所在的网段上动态获取的，也就是说 Pod1、Pod2 和网桥本身的 IP 是同一个网段的。由于 Pod1 和 Pod2 处于同一局域网内，它们之间的通信就可以通过 cbr0 直接路由通信。

下面我们再来看看处于不同主机上的 Pod 之间是如何进行网络通信的。

按照 Kubernetes 设定的网络基本原则，Pod 是需要跨节点可达的。而 Kubernetes 并不关心底层怎么处实现，我们可以借助 L2（ARP跨节点）、L3（IP路由跨节点，就像云提供商的路由表）、Overlay 网络、弹性网卡等方案，只要能够保证流量可以到达另一个节点的期望 Pod 就好。因此，如果要实现这种跨节点的 Pod 之间的网络通信，至少要满足下面 3 个条件：

1. 在整个 Kubernetes 集群中，每个 Pod 的 IP 地址必须是唯一的，不能与其他节点上的 Pod IP 冲突；
2. 从 Pod 中发出的数据包不应该进行 NAT ，这样通信双方看到的 IP 地址就是对方 Pod 实际的地址；
3. 我们得知道 Pod IP 和所在节点 Node IP 之间的映射关系。

我们以下图来说明基于 L2 的跨节点 Pod 互相访问时的网络流量走向。

![Drawing 1.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl-8tOWAFzVSAAIezV_GXxk921.png)

假设一个数据包要从 Pod1 到达 Pod4，这两个 Pod 分属两个不同的节点：

1. 数据包从 Pod1 中 netns 的 eth0 网口离开，通过 vethxxx 进入到 root netns；
2. 数据包到达 cbr0 后，cbr0 发送 ARP 请求来找到目标地址；
3. 由于 Pod1 所在的当前节点上并没有 Pod4，数据包会从 cbr0 流到宿主机的主网络接口 eth0；
4. 数据包的源地址为 Pod1，目标地址设置为了 Pod4，随后离开 Node1；
5. 路由表上记录着每个节点的 CIDR 的路由设定，这时候会把数据包路由到包含 Pod4 IP 地址的节点 Node2；
6. 数据包到达 Node2 的 eth0 网口后，由于节点配置了 IP forward，节点上的路由表寻找到了能匹配 Pod4 IP 地址的 cbr0，数据包转发到 cbr0；
7. cbr0 网桥接收了该数据包，发送 ARP 请求查询 Pod4 的 IP 地址，发现 IP 属于 vethyyy；
8. 这时候数据包跨过管道对到达 Pod4。



#### Pod到 Service 的网络

当我们创建一个 Service 时，Kubernetes 会为这个服务分配一个虚拟 IP。我们通过这个虚拟 IP 访问服务时，会由 iptables 负责转发。iptables 的规则由 Kube-Proxy 负责维护，当然也支持 [ipvs 模式](https://kubernetes.io/zh/docs/concepts/services-networking/service/#proxy-mode-ipvs)。

我们这里以 iptables 模式为例，如下图所示：

![Drawing 2.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl-8tPSABkDCAAQPXfFW4NM912.png)

Kube-Proxy 在每个节点上都运行，并会不断地查询和监听 Kube-APIServer 中 Service 与 Endpoints 的变化，来更新本地的 iptables 规则，实现其主要功能，包括为新创建的 Service 打开一个本地代理对象，接收请求针对发生变化的Service列表，kube-proxy 会逐个处理，处理流程如下：

1. 对已经删除的 Service 进行清理，删除不需要的 iptables 规则；
2. 如果一个新的 Service 没设置 ClusterIP，则直接跳过，不做任何处理；
3. 获取该 Service 的所有端口定义列表，并逐个读取 Endpoints 里各个示例的 IP 地址，生成或更新对应的 iptables 规则。



#### 外部和服务间通信

Pod 和 Service 都是 Kubernetes 中的概念，而从集群外是无法直接通过 Pod IP 或者 Service 的虚拟 IP 地址加端口号访问到的。这里我们就可以用 [Ingress](https://kubernetes.io/zh/docs/concepts/services-networking/ingress/) 来解决这个问题。

Ingress 可以将集群内 [服务](https://kubernetes.io/zh/docs/concepts/services-networking/service/) 的 HTTP 和 HTTPS 暴露出来，以方便从集群外部访问。流量路由 Ingress 资源上定义的规则控制。

下图就是一个简单的将所有流量都发送到同一 Service 的 Ingress 示例：

![Drawing 3.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl-8tQSAAn3KAACHOYfrYmI292.png)

关于 Ingress 的更多资料，可以参考 [这份官方文档](https://kubernetes.io/zh/docs/concepts/services-networking/ingress/)。

上面我们介绍了 Kubernetes 中的网络模型以及常见的 4 种网络通信情形，那么这些网络底层到底是如何实现的呢？这就有了 CNI。



### CNI（Container Network Interface）

网络方案没有银弹，没有任何一个“普适”的网络可以满足不同的基础架构，所以 Kubernetes 早在 v1.1 版本就开始采用了 CNI（Container Network Interface）这种插件化的方式来集成不同的网络方案。

CNI 其实就是定义了一套标准的、通用的接口。CNI 关心容器的网络连接，在创建容器时分配网络资源，删除容器时释放网络资源。这套框架非常开放，可以支持不同的网络模式，并且很容易实现。

正是因为有了 CNI，才使得 Kubernetes 在各个平台上都有很好的移植性。如果每出现一个新的网络解决方案，Kubernetes 都需要进行适配，那工作量必然非常巨大，维护成本也极高。所以我们只需要提供一套标准的接口，就完美地解决了上述问题。任何新的网络方案，都只需要实现 CNI 的接口。

容器网络的最终实现都依赖这些 CNI 插件，每个 CNI 插件本质上就是一个可以执行的二进制文件。总的来说，这些 CNI 插件的执行过程包括如下 3 个部分：

- 解析配置信息；
- 执行具体的网络配置 ADD 或 DEL；
- 对于 ADD 操作还需输出结果。

如果你想自己动手写一个 CNI 插件，可以参考这个 [GitHub 项目](https://github.com/eranyanay/cni-from-scratch)。

目前已经有很多的 CNI 插件可供选择，你可以直接从 [这份插件列表](https://kubernetes.io/zh/docs/concepts/cluster-administration/networking/#%E5%A6%82%E4%BD%95%E5%AE%9E%E7%8E%B0-kubernetes-%E7%9A%84%E7%BD%91%E7%BB%9C%E6%A8%A1%E5%9E%8B) 里选择适合的 CNI ，比如 Flannel、Calico、Cilium。



### 写在最后

基于 IP-per-Pod 的基本网络原则，Kubernetes 设计出了 Pod - Deployment - Service 这样经典的 3 层服务访问机制，极大地方便开发者在 Kubernetes 部署自己的服务。在生产实践中，大家可以根据自己的业务需要选择合适的 CNI 插件。





## 27 | K8s CRD：如何根据需求自定义你的 API？



随着使用的深入，你会发现 Kubernetes 中内置的对象定义，比如 Deployment、StatefulSet、Configmap，可能已经不能满足你的需求了。你很希望在 Kubernetes 定义一些自己的对象，一来可以通过 kube-apiserver 提供统一的访问入口，二来可以像其他内置对象一样，通过 kubectl 命令管理这些自定义的对象。

Kubernetes 中提供了两种自定义对象的方式，一种是聚合 API，另一种是 CRD。



### 聚合 API

聚合 API（Aggregation API，AA）是自 Kubernetes v1.7 版本就引入的功能，主要目的是方便用户将自己定义的 API 注册到 kube-apiserver 中，并且可以像使用其他内置的 API 一样，通过 APIServer 的 URL 就可以访问和操作。

在使用 API 聚合之前，我们还需要通过一些参数在 kube-apiserver 中启用这个功能：

```sh
--requestheader-client-ca-file=<path to aggregator CA cert>
--requestheader-allowed-names=front-proxy-client
--requestheader-extra-headers-prefix=X-Remote-Extra-
--requestheader-group-headers=X-Remote-Group
--requestheader-username-headers=X-Remote-User
--proxy-client-cert-file=<path to aggregator proxy cert>
--proxy-client-key-file=<path to aggregator proxy key>
```

Kubernetes 在 kube-apiserver 中引入了一个 API 聚合层（API Aggregation Layer），可以将访问请求转发到后端用户自己的扩展 APIServer 中。你可以参考 [这个文档](https://kubernetes.io/zh/docs/tasks/extend-kubernetes/setup-extension-api-server/) 学习如何安装一个扩展的 APIServer。当然，扩展的 APIServer 需要你自己修改并实现相关的代码逻辑。

- Kubernetes apiserver 会判断 `--requestheader-client-ca-file` 指定的 CA 证书中的 CN 是否是 `--requestheader-allowed-names` 提供的列表名称之一。如果不是，则该请求被拒绝。如果名称允许，则请求会被转发。
- 接着 Kubernetes apiserver 将使用由 `--proxy-client-*-file` 指定的文件来访问用户的扩展 APIServer。

下图就是用户发起请求后一个完整的身份认证流程，你可以阅读 [官方文档](https://kubernetes.io/zh/docs/tasks/extend-kubernetes/configure-aggregation-layer/#%E8%BA%AB%E4%BB%BD%E8%AE%A4%E8%AF%81%E6%B5%81%E7%A8%8B) 来详细了解步骤 。

![聚合层认证流程](D:\学习资料\笔记\k8s\k8s图\aggregation-api-auth-flow.png)

配置好了上述参数后， 为了让 kube-apiserver 知道我们自定义的 API，我们需要创建一个 APIService 的对象，比如：

```yaml
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  name: v1alpha1.wardle.example.com # 对象名称
spec:
  insecureSkipTLSVerify: true
  group: wardle.example.com # 扩展 Apiserver 的 API group 名称
  groupPriorityMinimum: 1000 # APIService 对对应 group 的优先级
  versionPriority: 15 # 优先考虑 version 在 group 中的排序
  service:
    name: myapi # 扩展 Apiserver 服务的 name
    namespace: wardle # 扩展 Apiserver 服务的 namespace 
  version: v1alpha1 # 扩展 Apiserver 的 API version
```

通过上述 YAML 文件，我们就在 kube-apiserver 中创建一个 API 对象组，其对应的组（Group）为 wardle.example.com，版本号为 v1alpha1。创建成功后，我们就可以通过 `/apis/wardle.example.com/v1alpha1` 这个路径访问了。至于这个对象组里提供了哪些对象，就需要我们在扩展的 APIServer 中声明出来了。

那 kube-apiserver 又是怎么知道将 /apis/wardle.example.com/v1alpha1 这个路径的所有请求转发到后端的扩展 APIServer 中的呢？我们注意到上面的 APIService 定义中，还有一个 `spec.service` 字段，就是在这里，我们指定了扩展 APIServer 的服务名，也就是说 kube-apiserver 会将 `/apis/wardle.example.com/v1alpha1` 这个路径的所有访问通过 API 聚合层转发到后端的服务 `myapi.wardle.svc` 上。

扩展 APIServer 的具体代码设计及逻辑，你可以参考[sample-apiserver](https://github.com/kubernetes/sample-apiserver) 或者使用 [apiserver-builder](https://github.com/kubernetes-sigs/apiserver-builder-alpha)。

这种聚合 API 的方式对代码要求很高，但支持对 API 行为进行完全的掌控，比如你可以自己定义数据如何存储、API 各个版本的切换、各个对象的逻辑控制。

如果你只想简简单单地在 Kubernetes 中定义个对象，可以直接通过下面要介绍的 CRD 定义。



### CRD

CRD（CustomResourceDefinitions）在 v1.7 刚引入进来的时候，其实是 ThirdPartyResources（TPR）的升级版本，而 TPR 在 v1.8 的版本被剔除了。CRD 目前使用非常广泛，各个周边项目都在使用它，比如 Ingress、Rancher。

我们来看一下官方的一个例子，通过如下的 YAML 文件，我们可以创建一个 API：

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # 名字必须与下面的 spec 字段匹配，并且格式为 '<名称的复数形式>.<组名>'
  name: crontabs.stable.example.com
spec:
  # 组名称，用于 REST API: /apis/<组>/<版本>
  group: stable.example.com
  # 列举此 CustomResourceDefinition 所支持的版本
  versions:
    - name: v1
      # 每个版本都可以通过 served 标志来独立启用或禁止
      served: true
      # 其中一个且只有一个版本必需被标记为存储版本
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                cronSpec:
                  type: string
                image:
                  type: string
                replicas:
                  type: integer
  # 可以是 Namespaced 或 Cluster
  scope: Namespaced
  names:
    # 名称的复数形式，用于 URL：/apis/<组>/<版本>/<名称的复数形式>
    plural: crontabs
    # 名称的单数形式，作为命令行使用时和显示时的别名
    singular: crontab
    # kind 通常是单数形式的驼峰编码（CamelCased）形式。你的资源清单会使用这一形式。
    kind: CronTab
    # shortNames 允许你在命令行使用较短的字符串来匹配资源
    shortNames:
    - ct
```

我们可以像创建其他对象一样，通过 kubectl create 命令创建。创建好了以后，一个类型为 CronTab 的对象就在 kube-apiserver 中注册好了，你可以通过如下的 REST 接口访问，比如 LIST 命名空间 ns1 下的 CronTab 对象，可以通过这个 `URL“/apis/stable.example.com/v1/namespaces/ns1/crontabs/”`访问。这种接口跟 Kubernetes 内置的其他对象的接口风格是一模一样的。

声明好了 CronTab，下面我们就来看看如何创建一个 CronTab 类型的对象。我们来看看官方给的一个例子：

```yaml
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "* * * * */5"
  image: my-awesome-cron-image
```

通过 kubectl create 创建 my-new-cron-object 后，就可以通过 `kubectl get` 查看，并使用 kubectl 管理你的 CronTab 对象了。例如：

```sh
$ kubectl get crontab
NAME                 AGE
my-new-cron-object   6s
```

这里的资源名是大小写不敏感的，我们在这里可以使用缩写 `kubectl get ct`，也可以使用 kubectl get crontabs。

同时原生内置的 API 对象一样，这些 CRD 不仅可以通过 kubectl 来创建、查看、修改，删除等操作，还可以给其配置 RBAC 规则。

我们还能开发自定义的控制器，来感知和操作这些自定义的 API。

我们在创建 CRD 的时候，还可以一起定义 /status 和 /scale 子资源，如下所示：

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  group: stable.example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                cronSpec:
                  type: string
                image:
                  type: string
                replicas:
                  type: integer
            status:
              type: object
              properties:
                replicas:
                  type: integer
                labelSelector:
                  type: string
      # subresources 描述定制资源的子资源
      subresources:
        # status 启用 status 子资源
        status: {}
        # scale 启用 scale 子资源
        scale:
          # specReplicasPath 定义定制资源中对应 scale.spec.replicas 的 JSON 路径
          specReplicasPath: .spec.replicas
          # statusReplicasPath 定义定制资源中对应scale.status.replicas 的JSON 路径 
          statusReplicasPath: .status.replicas
          # labelSelectorPath  定义定制资源中对应scale.status.selector 的JSON 路径 
          labelSelectorPath: .status.labelSelector
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
    shortNames:
    - ct
```

这些子资源的定义，可以帮助我们在更新对象的时候，只更改期望的部分，比如只更新 status 部分，避免误更新 spec 部分。

同时 CRD 定义的时候还支持 [合法性检查](https://kubernetes.io/zh/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#validation)、[设置默认值](https://kubernetes.io/zh/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#efaulting) 等操作，你可以参照文档来实践。



### 写在最后

当然，你也可以选择在其他地方定义新的 API，你可以参考 [这份](https://kubernetes.io/zh/docs/concepts/extend-kubernetes/api-extension/custom-resources/#%E6%88%91%E6%98%AF%E5%90%A6%E5%BA%94%E8%AF%A5%E5%90%91%E6%88%91%E7%9A%84-kubernetes-%E9%9B%86%E7%BE%A4%E6%B7%BB%E5%8A%A0%E5%AE%9A%E5%88%B6%E8%B5%84%E6%BA%90) 文档确定是否需要在 Kubernetes 中定义 API，还是让你的 API 独立运行。

你可以通过这个 [网址](https://kubernetes.io/zh/docs/concepts/extend-kubernetes/api-extension/custom-resources/#advanced-features-and-flexibility) 中的内容来比较 CRD 和 聚合 API 的功能差异。简单来说，CRD 更为易用，而聚合 API 更为灵活。

Kubernetes 提供这两种选项以满足不同用户的需求，这样就既不会牺牲易用性也不会牺牲灵活性。





## 28 | 面向 K8s 编程：如何通过 Operator 扩展 Kubernetes API？



在上一讲，我们学习了如何通过一个 YAML 文件来定义一个 CRD，即扩展 API。这种扩展 API 跟 Kubernetes 内置的其他 API 同等地位，都可以通过 kubectl 或者 REST 接口访问，在使用过程中不会有任何差异。

但只是定义一个 CRD 并没有什么作用。虽说 kube-apiserver 会将其数据存放到 etcd 中，并暴露出相应的 REST 接口，然而并不涉及该对象的核心处理逻辑。

如何对这些 CRD 定义的对象进行一些逻辑处理，需要由用户自己来定义和实现，也就是通过控制器来实现。对此，我们有个专门的名字：Operator。



### 什么是 Kubernetes Operator

你可能对 Operator 这个名字比较陌生。这个名字最早由 CoreOS 在 2016 年提出来，我们来看看他们给出的定义：

> An operator is a method of packaging, deploying and managing a Kubernetes application. A Kubernetes application is an application that is both deployed on Kubernetes and managed using the Kubernetes APIs and kubectl tooling.
>
> To be able to make the most of Kubernetes, you need a set of cohensive APIs to extend in order to service and manage your applications that run on Kubernetes. You can think of Operators as the runtime that manages this type of application on Kubernetes.

总结一下，所谓的 Kubernetes Operator 其实就是借助 Kubernetes 的控制器模式，配合一些自定义的 API，完成对某一类应用的操作，比如资源创建、资源删除。

这里对 Kubernetes 的控制器模式做个简要说明。**Kubernetes 通过声明式 API 来定义对象，各个控制器负责实时查看对应对象的状态，确保达到定义的期望状态。**这就是 Kubernetes 的控制器模式。

![Drawing 0.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F_HAmyACHLHAAHSt7ZcZoY464.png)

kube-controller-manager 就是由这样一组控制器组成的。我们以 StatefulSet 为例来简单说明下控制器的具体逻辑。

假设你声明了一个 StatefulSet，并将其副本数设置为 2。kube-controller-manager 中以 goroutine 方式运行的 StatefulSet 控制器在观察 kube-apiserver 的时候，发现了这个新创建的对象，它会先创建一个 index 为 0 的 Pod ，并实时观察这个 Pod 的状态，待其状态变为 Running 后，再创建 index 为 1 的 Pod。后续该控制器会一直观察并维护这些 Pod 的状态，保证 StatefulSet 的有效副本数始终为 2。

所以我们在声明完成 CRD 之后，也需要创建一个控制器，即 Operator，来完成对应的控制逻辑。

了解了 Operator 的概念和控制器模式后，我们来看看 Operator 是如何工作的。



### Kubernetes Operator 是如何工作的

Operator 工作的时候采用上述的控制器模式，会持续地观察 Kubernetes 中的自定义对象，即 CR（Custom Resource）。我们通过 CRD 来定义一个对象，CR 则是 CRD 实例化的对象。

![Drawing 1.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F_HAnWAZUN3AAGfGj4K8Gw651.png)

Operator 会持续跟踪这些 CR 的变化事件，比如 ADD、UPDATE、DELETE，然后采取一系列操作，使其达到期望的状态。

那么具体的代码层面，整个逻辑又如何实现呢？

下面就是 Operator 代码层面的工作流程图：

![image.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl_HLoWAHjvEAAPol71Pgh8456.png)

如上图所示，上半部分是一个 Informer，它的机制就是不断地 list/watch kube-apiserver 中特定的资源，比如你只关心 Pod，那么就只 list/watch Pod。Informer 主要有两个方法：一个是 ListFunc，一个是 WatchFunc。

- ListFunc 可以把某类资源的所有资源都列出来，当然你可以指定某个命名空间。
- WatchFunc 则会和 apiserver 建立一个长链接，一旦有一个对象更新或者新对象创建，apiserver 就会反向推送回来，告诉 Informer 有一个新的对象创建或者更新等操作。

当 Informer 接收到了这些操作，就会调用对应的函数（比如 AddFunc、UpdateFunc 和 DeleteFunc），并将其按照 key 值的格式放到一个先入先出的队列中。

key 值的命名规则就是 “namespace/name”，name 是对应的资源的名字，比如在 default 的 namespace 中创建一个 foo 类型的资源 example-foo，那么它的 key 值就是 “default/example-foo”。

我们一般会给 Operator 设置多个 Worker，并行地从上面的队列中拿到对象去操作处理。工作完成之后，就把这个 key 丢掉，表示已经处理完成。但如果处理过程中有错误，则可以把这个 key 重新放回到队列中，后续再重新处理。

看得出来，上述的流程其实还是有些复杂的，尤其是对初学者有一定的门槛。幸好社区提供了一些脚手架，方便我们快速地构建自己的 Operator。



### 构建一个自己的 Kubernetes Operator

目前社区有一些可以用于创建 Kubernetes Operator 的开源项目，例如：[Operator SDK](https://github.com/operator-framework/operator-sdk)、[Kubebuilder](https://github.com/kubernetes-sigs/kubebuilder)、[KUDO](https://github.com/kudobuilder/kudo)。

有了这些脚手架，我们就可以快速创建出 Operator 的框架。这里我以 kubebuilder 为例。

我们可以先通过如下命令安装 kustomize：

```sh
$ curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
$ mv kustomize /usr/local/bin/
```

再安装 kubebuilder。你可以选择通过源码安装：

```sh
$ git clone https://github.com/kubernetes-sigs/kubebuilder
$ cd kubebuilder
$ git checkout v2.3.1
$ make build
$ cp bin/kubebuilder $GOPATH/bin
```

如果你本地有些代码拉不下来，可以用 proxy：

```sh
$ export GOPROXY=https://goproxy.cn
```

也可以直接下载二进制文件：

```sh
$ os=$(go env GOOS)
$ arch=$(go env GOARCH)
# download kubebuilder and extract it to tmp
$ curl -sL https://go.kubebuilder.io/dl/2.3.1/${os}/${arch} | tar -xz -C /tmp/
$ sudo mv /tmp/kubebuilder_2.3.1_${os}_${arch} /usr/local/bin/kubebuilder
```

安装好以后，我们就可以通过 kubebuilder 来创建 Operator 了：

```sh
$ cd $GOPATH/src
$ mkdir -p github.com/zhangsan/operator-demo
$ cd github.com/zhangsan/operator-demo
$ kubebuilder init --domain abc.com --license apache2 --owner "zhangsan"
$ kubebuilder create api --group foo --version v1 --kind Bar
```

通过上面 kubebuilder 的命令，我们会在当前目录创建一个 Operator 的框架，并声明了一个 Bar 类型的 API。

你通过 make manifests 即可生成所需要的 yaml 文件，包括 CRD、RBAC等。

通过如下的命令即可安装 CRD、RBAC等对象：

```sh
$ make install # 安装CRD
```

然后我们就可以看到创建的CRD了：

```sh
$ kubectl get crd
NAME               AGE
bars.foo.abc.com   2m
```

再来创建一个 Bar 类型的对象：

```sh
$ kubectl apply -f config/samples/
$ kubectl get bar 
NAME         AGE
bar-sample   25s
```

在本地开发阶段，我们可以通过 make run 命令，在本地运行。运行起来以后，你可以从输出日志中看到我们刚创建的 default 命名空间下的 bar-sample，即 key 为 “default/bar-sample”。

我们在开发的时候，只需要修改 “api/v1/bar_types.go”和“controllers/bar_controller.go”这两个文件即可。这两个文件中有注释，会告诉你新增的对象定义和具体逻辑写哪里。这里你也可以参考 kubebuilder 的文档。

你开发完成之后，就可以构建一个镜像出来，方便部署：

```sh
$ make docker-build docker-push IMG=zhangsan/operator-demo
$ make deploy
```



### 写在最后

到这里，你就完成了对扩展 Kubernetes API 的学习。这一讲的难点不在于 Operator 本身，而是要学会理解它的行为。





## 29 | Kubernetes 中也有定时任务吗？



前面我们学习了 Deployment、Statefulset、Daemonset 这些工作负载，它们可以帮助我们在不同的场景下运行长伺型（Long Running）的服务。

但是有一类业务（一次性作业和定时任务）运行完就结束了，不需要长期运行，如果使用上述的那些工作负载就无法满足我们的要求。比如 Pod 运行结束后，会被 Deployment、Statefulset 控制器重启或者创建新的副本替换掉，而这并不是我们期望的行为。

所以说，对于这类作业任务，我们需要新的工作负载类型来描述。在 Kubernetes 中，我们分别用 Job 和 Cronjob 来描述一次性任务和定时任务。

我们先来看看 Job。



### Job

我通过一个官方的例子来带你了解这个工作负载类型：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  backoffLimit: 4
```

这个 Job 负责计算 π 到小数点后的 2000 位，并将结果打印出来。

我们可以通过 kubectl create 命令将该 Job 创建出来：

```sh
$ kubectl create -f https://kubernetes.io/examples/controllers/job.yaml
job.batch/pi created
```

创建好了以后，我们来看下这个 Job：

```sh
$ kubectl describe jobs/pi
Name:           pi
Namespace:      default
Selector:       controller-uid=4f8027d0-cac1-42ea-b5f8-dbb4d9c9f67a
Labels:         controller-uid=4f8027d0-cac1-42ea-b5f8-dbb4d9c9f67a
                job-name=pi
Annotations:    <none>
Parallelism:    1
Completions:    1
Start Time:     Mon, 02 Dec 2020 15:04:52 +0200
Completed At:   Mon, 02 Dec 2020 15:06:39 +0200
Duration:       65s
Pods Statuses:  0 Running / 1 Succeeded / 0 Failed
Pod Template:
  Labels:  controller-uid=c9948307-e56d-4b5d-8302-ae2d7b7da67c
           job-name=pi
  Containers:
   pi:
    ...
Events:
  Type    Reason            Age   From            Message
  ----    ------            ----  ----            -------
  Normal  SuccessfulCreate  4m    job-controller  Created pod: pi-jk2k7
```

在这段代码中，有几点需要特别注意下。

1. 系统自动给 Job 添加了 Selector，即 `controller-uid=4f8027d0-cac1-42ea-b5f8-dbb4d9c9f67a`，后面的这个 uid 就是指该 Job 自己的 uid。

2. Job 上会自动被加上了 Label，即 `controller-uid=4f8027d0-cac1-42ea-b5f8-dbb4d9c9f67a` 和 `job-name=pi`。

3. Job 中 `spec.podTemplate` 中也被加上了 Label，即 `controller-uid=4f8027d0-cac1-42ea-b5f8-dbb4d9c9f67a` 和 `job-name=pi`。这样 Job 就可以通过这些 Label 和 Pod 关联起来，从而控制 Pod 的创建、删除等操作。

我们可以通过这些 Label 来找到对应的 Pod。你可以直接使用 Job 的名字，这个最简洁最方便：

```sh
$ kubectl get pods --selector=job-name=pi
```

或者你也可以选择使用 Job 的 uid：

```sh
$ kubectl get pods --selector=controller-uid=4f8027d0-cac1-42ea-b5f8-dbb4d9c9f67a
```

我们可以看到，由 Job 创建出来的 Pod 已经运行结束，为 Completed 状态。

```sh
NAME       READY    STATUS       RESTARTS    AGE
pi-jk2k7   0/1      Completed    0           2m
```

如果说 Pod 运行过程中异常退出了，那就会根据的 Job 中 PodTemplate 定义的重启策略（restart policy）来操作。对于 Job 来说，我们当然不希望一直重启，因此这里的 restartPolicy 只能为 Never 或者 OnFailure。

如果说创建出来的 Pod 一直由于某些原因，导致运行不成功，怎么办呢？这个时候 **Job 控制器会根据 spec.backoffLimit 中定义的数值来限制 Pod 失败的次数**。默认值是 6，我们在例子中设置为 4。达到这个次数以后，Job controller 便不会再新建 Pod，并直接停止运行这个 Job，将其标记为 Failure。

Job 还支持创建多个 Pod 并发地运行，我们来看另一个官方的例子“[使用工作队列进行精细的并行处理](https://kubernetes.io/zh/docs/tasks/job/fine-parallel-processing-work-queue/)”：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-wq-2
spec:
  parallelism: 2
  template:
    metadata:
      name: job-wq-2
    spec:
      containers:
      - name: c
        image: gcr.io/myproject/job-wq-2
      restartPolicy: OnFailure
```

在 Job 的定义中通过 `spec.parallelism` 字段，我们可以指定并发运行的 Pod 数目。我们这里指定为 2，也就是会创建 2 个同时运行的 Pod。

例子中的 2 个 Pod 都会不停地从队列中获取数据进行处理，直到队列为空后退出，运行结束。

Job 还支持其他的工作模式，比如通过模板渲染 Job 来支持批量任务的处理等等。你可以参照 [官方文档](https://kubernetes.io/zh/docs/concepts/workloads/controllers/job/#job-patterns) 来解锁 Job 更多地使用场景，并动手学习实践。

通常来说，Job 运行结束，即状态为 Completed 或 Failure 时，我们并不需要在系统中继续保留该对象，尤其是这种对象较多的时候，会给 kube-apiserver 的 cache 以及系统访问带来很大的压力。

这个时候我们就可以使用 [TTL 控制器](https://kubernetes.io/zh/docs/concepts/workloads/controllers/ttlafterfinished/) 提供 的 TTL 能力了。

我们只需要在 Job 的 `spec.ttlSecondsAfterFinished` 字段设置一下，就可以让该控制器帮我们自动清理掉已经结束的资源，包括 Job 本身及其关联的 Pod 对象。

我们来看下面这个例子：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-with-ttl
spec:
  ttlSecondsAfterFinished: 100
  template:
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
```

该 Job 在运行结束 100 秒之后就被自动清理删除了，包括创建出来的 Pod。

目前这种 TTL 的能力还处于 Alpha 阶段，如果你要使用的话，需要手动开启 TTLAfterFinished 这个 feature gate，具体可以参考 TTL 控制器的 [文档](https://kubernetes.io/zh/docs/concepts/workloads/controllers/ttlafterfinished/) 学习如何打开和使用这个功能。

我们再来看看 CronJob。



### CronJob

从名字就可以看出来，这个工作负载是用于定时任务的，比如每隔 1 分钟执行 1 次任务。

我们来看一个官方的 CronJob 的例子，这个例子比较简单明了：

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello # cronjob 名字
spec:
  schedule: "*/1 * * * *" # job执行的周期，通过 cron 格式来标明
  jobTemplate: # job模板
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            imagePullPolicy: IfNotPresent
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```

CronJob 通过 `spec.schedule` 字段来标明 Job 被创建和执行的周期，该字段段用Cron格式编写。Cron 的基本格式为：

```sh
<分钟> <小时> <日> <月> <星期>
```

其中分钟的值从 0 到 59，小时的值从 0 到 23，日的值从 1 到 31，月的值从 1 到 12，星期的值从 0 到 6，0 表示星期日。

Cron 还支持“`*,-/`”等字符，其中 `*` 是个通配符，可以匹配任何值；`/` 则表示起始时间触发，然后每隔一个固定时间触发一次。例如我们如果在分钟中设置 10/20，则表示第一次触发在第 10 分钟，接下来每隔 20 分钟触发一次，也就是第 30 分钟、第 50 分钟等依次往后的时间点触发一次。

所以我们例子中的"`*/1 * * * *`"表示每隔一分钟触发一次新 Job 的执行。

例子中的 `spec.jobTemplate` 指定了 Job 的模板，CronJob 控制器会根据 Cron 设置的时间触发新的 Job 创建。因此，我们修改 CronJob 的 spec，只会影响新 Job 的spec 配置，对于已经创建的 Job spec 不会有任何影响。

CronJob 会帮助我们管理 Job，比如自动清理运行完的 Job；也就是说，由 CronJob 管理的 Job，我们不需要去配置上文提到的 TTL 自动清理，CronJob 控制器会自动帮我们清理。CronJob 通过 `spec.successfulJobsHistoryLimit` 和 `spec.failedJobsHistoryLimit` 来限制保留已完成的 Job 数量，确保不会有大量的 Job 残留在系统中。默认值分别为 3 和 1。

当然在 CronJob 被触发，创建新的 Job 的时候，还会出现一种情形：上一次触发的 Job 还未执行完成。如果这个时候触发了另一个新 Job 的创建，势必会导致任务重叠。此时就需要你结合自己的业务来考虑这种行为对业务的影响了。

你可以在 `spec.concurrentPolicy` 中配置：

- 设置为 Allow，这也是默认的值，允许并发任务的执行；

- 设置为 Forbid，不允许并发任务执行；

- 设置为 Replace，用新的 Job 来替换当前正在运行的老的 Job。



### 写在最后

这一讲，我带你了解了如何在 Kubernetes 中设置定时任务。所有 Cronjob 中的 schedule 字段中的时间都是基于 kube-controller-manager 的时区。我们在搭建环境的时候，最好将各个节点上的时间进行同步，这样可以避免很多奇奇怪怪的问题。



 

## 30 | Kubectl 命令行工具使用秘笈



kubectl 是我们日常操纵整个 Kubernetes 的利器，操作方便，功能强大。接下来，我会向你介绍常用的七个功能。



### 自动补全

我们可以通过如下命令进行命令行的自动补全，方便我们使用。

- 如果你使用的是 bash，可以通过如下命令：

```sh
#你需要先安装 bash-completion
$ source <(kubectl completion bash) 

#这样就不需要每次都 source 一下了
$ echo "source <(kubectl completion bash)" >> ~/.bashrc 
```

- 如果你使用的是 zsh，也有可用的命令：

```sh
$ source <(kubectl completion zsh)

#这样就不需要每次都 source 一下了
$ echo "[[ $commands[kubectl] ]] && source <(kubectl completion zsh)" >> ~/.zshrc
```



### 命令行帮助

遇到任何关于命令行使用的问题，都可以通过“kubectl -h”命令来查看有哪些子命令、包含哪些参数、使用例子等等。

这条命令也是我使用比较多的，可以帮助你更快地熟悉和了解 kubectl 的使用。



### 集群管理

我们可以通过“`kubectl cluster-info`”命令查看整个集群信息，比如 apiserver 暴露的地址、dns 的地址、metrics-server 的地址。

还可以通过“`kubectl version`”命令查看到 kubectl 以及 apiserver 的版本。毕竟 apiserver 的版本对整个集群至关重要，决定了各个 api 的版本、feature gate、准入控制等等，

而各个 kubelet 节点的版本，你可以通过“`kubectl get node`”命令查看。

你可以通过这些版本号了解到整个集群的版本信息，对集群的维护和升级很有帮助。



### 资源查询

通过 kubectl 命令来查询集群中的资源是我们日常使用频率最高的。

你可以通过 kubectl get 查询到某类资源对象，代码如下：

```sh
$ kubectl get [资源]
```

假如我们想要查看集群中的所有节点，可以在代码的“资源”处输入“nodes”，如下所示：

````sh
$ kubectl get nodes
````

这里的资源名，我们可以使用资源名称的单数，比如 node；也可以使用其复数，比如 nodes；还可以使用其缩写名。

对于集群中定义的资源信息，比如资源名、对应的缩写、是否是 namespace 级别的资源，你可以通过“`kubectl api-resources`”命令获取。

如果我们想要查询某个资源对象，我们同样可以通过“`kubectl get`”命令，只不过要在原先的资源名后面加上“/对象名”。如下所示：

```sh
$ kubectl get [资源]/[对象名]
```

还是以 node 为例，我想查询 node01 节点，就可以通过“`kubectl get node/node01`”命令完成。当然，不使用“/”也是允许的，代码如下所示：

```sh
$ kubectl get [资源] [对象名]
```

如果你想看到关于这个对象更详细的信息，你可以“`kubectl describe`”一下，即：

```sh
$ kubectl describe [资源]/[对象名]
```

对于 namesppace 级别的资源，我们只需要在上述命令后面加上“-n [命名空间]”或“--namespace [命名空间]”就可以了。代码如下所示：

```sh
$ kubectl get [资源]/[对象名] -n [命名空间]
```

比如：

```sh
$ kubectl get pod pod-example -n demo
```

如果你没有指定 namespace 的话，默认是名为 default 的命名空间。

此外，如果你想要查看所有命名空间下的某类资源，可以在“资源”后面加上“--all-namespaces”。代码如下：

```sh
$ kubectl get [资源] --all-namespaces
```

比如：

```sh
$ kubectl get pod --all-namespaces
```



### 资源创建、更改、删除

你还可以通过 kubectl create 进行资源创建，代码如下：

```sh
$ kubectl create -f demo.yaml
```

通过在 yaml 文件中定义各种资源及对象，我们通过这条命令将其在集群中创建出来。

当然，kubectl create 还提供了一些子命令，方便通过命令行直接创建资源对象。你可以通过“kubectl create -h”查看其支持的子命令。

假如我们想要在命名空间 demo 下创建一个名为 sa-demo 的 ServiceAccount 对象，我们可以通过如下命令进行：

```sh
$ kubectl create sa sa-demo -n demo
```

通常来说，从零开始写一个 yaml 文件很难，一般我们都是找一些资源的 yaml 例子拿来自己修改下。我并不推荐自己一点点去写 yaml，效率低下，而且还会出现缩进的问题。

我推荐通过 **kubectl create 的这些命令来解决**。通过命令行参数，我们可以让 kubectl 帮我们自动生成一些 yaml 文件，比如你可以通过下面的命令拿到了一个 deployment 的 yaml 文件，然后就可以对这个文件进一步地修改以达到你的期望定义。

```sh
$ kubectl create deploy my-deployment -n demo --image=busybox --dry-run=server -o yaml > my-deployment.yaml
```

这儿主要是用到了 dry-run 的能力。

你还可以通过 kubectl edit 直接修改这些资源：

```sh
$ kubectl edit [资源]/[对象名] -n [命名空间]
```

 如果是集群级别的资源对象，那么代码中就不用加“-n”了。

或者你也可以通过修改 yaml 文件，然后 apply 到集群中：

```sh
$ kubectl apply -f demo.yaml
```

当然，这条命令还能被用来创建对象，如果对象已经存在就会对它进行更新。

对于资源对象的删除，可以直接通过 kubectl delete 进行：

```sh
$ kubectl delete [资源]/[对象名] [-n [命名空间]]
```

比如：

```sh
$ kubectl delete pod/pod-demo -n demo
```

如果你是通过 yaml 文件创建的某些资源对象，比如 flannel；yaml 文件中包含了很多对象，一个个删除太麻烦，也容易遗漏，你就可以通过如下命令删除：

```sh
$ kubectl delete -f flannel.yaml
```



### 日志

如果 pod 内只有一个容器，你可以通过“`kubectl logs [pod 名] -n [命名空间]`”命令查看该容器的日志。

如果有多个容器，就需要在“`pod 名”后插入“-c [容器名]`”，如下所示：

```sh
$ kubectl logs [pod 名] -c [容器名] -n [命名空间]
```

你还可以通过“-f”参数来实时查看容器最新的日志。更多参数，就需要你自己来探索了。



### 快速创建一个 Pod

我们可以通过如下命令快速创建一个 Pod，在我们做环境 debug 的时候非常方便：

```sh
$ kubectl run [pod 名] -n [命名空间] --image [镜像] [.....]
```

比如：

```sh
$ kubectl run pod1 -n debug-pod --image network-debug -- bash
```

其他参数，你可以通过 “`kubectl run -h`” 查看；也可以直接 exec 到容器里查看，代码如下：

```sh
$ kubectl exec -it [pod 名] -n [命名空间] bash
```

同样，如果有多个容器的话，需要知道一个容器名：

```sh
$ kubectl exec -it [pod 名] -c [容器名] -n [命名空间] bash
```



### 写在最后

kubectl 支持 JSONPath 模版，在 [官方文档](https://kubernetes.io/zh/docs/reference/kubectl/jsonpath/) 中有详细的说明和例子，这里就不再重复了。

kubectl 能力非常强大，随着你不断地使用，你会发现更多 kubectl 中的小技巧，也会慢慢积累一些自己的使用技巧。以上我只是列举了一些常用的命令操作，你可以点击链接查看 [官方的 kubectl 小抄](https://kubernetes.io/zh/docs/reference/kubectl/cheatsheet/) 来学习。

本文中的例子，只使用到了部分参数，你可以通过 “-h”来查看其具体支持的各个参数。





## kubect 小抄



### Kubectl 上下文和配置

设置 `kubectl` 与哪个 Kubernetes 集群进行通信并修改配置信息。 查看[使用 kubeconfig 跨集群授权访问](https://kubernetes.io/zh/docs/tasks/access-application-cluster/configure-access-multiple-clusters/) 文档获取配置文件详细信息。

```bash
$ kubectl config view # 显示合并的 kubeconfig 配置。

# 同时使用多个 kubeconfig 文件并查看合并的配置
KUBECONFIG=~/.kube/config:~/.kube/kubconfig2 kubectl config view

# 获取 e2e 用户的密码
kubectl config view -o jsonpath='{.users[?(@.name == "e2e")].user.password}'

kubectl config view -o jsonpath='{.users[].name}'    # 显示第一个用户
kubectl config view -o jsonpath='{.users[*].name}'   # 获取用户列表
kubectl config get-contexts                          # 显示上下文列表
kubectl config current-context                       # 展示当前所处的上下文
kubectl config use-context my-cluster-name           # 设置默认的上下文为 my-cluster-name

# 添加新的用户配置到 kubeconf 中，使用 basic auth 进行身份认证
kubectl config set-credentials kubeuser/foo.kubernetes.com --username=kubeuser --password=kubepassword

# 在指定上下文中持久性地保存名字空间，供所有后续 kubectl 命令使用
kubectl config set-context --current --namespace=ggckad-s2

# 使用特定的用户名和名字空间设置上下文
kubectl config set-context gce --user=cluster-admin --namespace=foo \
  && kubectl config use-context gce

kubectl config unset users.foo                       # 删除用户 foo
```



### Kubectl apply

`apply` 通过定义 Kubernetes 资源的文件来管理应用。 它通过运行 `kubectl apply` 在集群中创建和更新资源。 这是在生产中管理 Kubernetes 应用的推荐方法。 参见 [Kubectl 文档](https://kubectl.docs.kubernetes.io/)。



### 创建对象

Kubernetes 配置可以用 YAML 或 JSON 定义。可以使用的文件扩展名有 `.yaml`、`.yml` 和 `.json`。

```bash
$ kubectl apply -f ./my-manifest.yaml           # 创建资源
$ kubectl apply -f ./my1.yaml -f ./my2.yaml     # 使用多个文件创建
$ kubectl apply -f ./dir                        # 基于目录下的所有清单文件创建资源
$ kubectl apply -f https://git.io/vPieo         # 从 URL 中创建资源
$ kubectl create deployment nginx --image=nginx # 启动单实例 nginx

# 创建一个打印 “Hello World” 的 Job
$ kubectl create job hello --image=busybox -- echo "Hello World" 

# 创建一个打印 “Hello World” 间隔1分钟的 CronJob
$ kubectl create cronjob hello --image=busybox   --schedule="*/1 * * * *" -- echo "Hello World"    

$ kubectl explain pods                          # 获取 pod 清单的文档说明

# 从标准输入创建多个 YAML 对象
$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: busybox-sleep
spec:
  containers:
  - name: busybox
    image: busybox
    args:
    - sleep
    - "1000000"
---
apiVersion: v1
kind: Pod
metadata:
  name: busybox-sleep-less
spec:
  containers:
  - name: busybox
    image: busybox
    args:
    - sleep
    - "1000"
EOF

# 创建有多个 key 的 Secret
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  password: $(echo -n "s33msi4" | base64 -w0)
  username: $(echo -n "jane" | base64 -w0)
EOF
```

### 查看和查找资源

```bash
# get 命令的基本输出

# 列出当前命名空间下的所有 services
$ kubectl get services        

# 列出所有命名空间下的全部的 Pods
$ kubectl get pods --all-namespaces  

# 列出当前命名空间下的全部 Pods，并显示更详细的信息
$ kubectl get pods -o wide 

# 列出某个特定的 Deployment
$ kubectl get deployment my-dep  

# 列出当前命名空间下的全部 Pods
$ kubectl get pods   

# 获取一个 pod 的 YAML
$ kubectl get pod my-pod -o yaml              

# describe 命令的详细输出
$ kubectl describe nodes my-node
$ kubectl describe pods my-pod

# 列出当前名字空间下所有 Services，按名称排序
$ kubectl get services --sort-by=.metadata.name

# 列出 Pods，按重启次数排序
$ kubectl get pods --sort-by='.status.containerStatuses[0].restartCount'

# 列举所有 PV 持久卷，按容量排序
$ kubectl get pv --sort-by=.spec.capacity.storage

# 获取包含 app=cassandra 标签的所有 Pods 的 version 标签
$ kubectl get pods --selector=app=cassandra -o \
  jsonpath='{.items[*].metadata.labels.version}'

# 检索带有 “.” 键值，例： 'ca.crt'
$ kubectl get configmap myconfig \
  -o jsonpath='{.data.ca\.crt}'

# 获取所有工作节点（使用选择器以排除标签名称为 'node-role.kubernetes.io/master' 的结果）
$ kubectl get node --selector='!node-role.kubernetes.io/master'

# 获取当前命名空间中正在运行的 Pods
$ kubectl get pods --field-selector=status.phase=Running

# 获取全部节点的 ExternalIP 地址
$ kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="ExternalIP")].address}'

# 列出属于某个特定 RC 的 Pods 的名称
# 在转换对于 jsonpath 过于复杂的场合，"jq" 命令很有用；可以在 https://stedolan.github.io/jq/ 找到它。
sel=${$(kubectl get rc my-rc --output=json | jq -j '.spec.selector | to_entries | .[] | "\(.key)=\(.value),"')%?}
echo $(kubectl get pods --selector=$sel --output=jsonpath={.items..metadata.name})

# 列出POD的Quality of Service (QoS) class
$ kubectl get pod whoami -o json | jq --raw-output .status.qosClass
BestEffort

# 列出deploy的annotations
$ kubectl get deployments.apps nginx -o json | jq .metadata.annotations
{
  "deployment.kubernetes.io/revision": "3"
}


# 显示所有 Pods 的标签（或任何其他支持标签的 Kubernetes 对象）
$ kubectl get pods --show-labels

# 检查哪些节点处于就绪状态
JSONPATH='{range .items[*]}{@.metadata.name}:{range @.status.conditions[*]}{@.type}={@.status};{end}{end}' \
 && kubectl get nodes -o jsonpath="$JSONPATH" | grep "Ready=True"

# 列出被一个 Pod 使用的全部 Secret
$ kubectl get pods -o json | jq '.items[].spec.containers[].env[]?.valueFrom.secretKeyRef.name' | grep -v null | sort | uniq

# 列举所有 Pods 中初始化容器的容器 ID（containerID）
# Helpful when cleaning up stopped containers, while avoiding removal of initContainers.
$ kubectl get pods --all-namespaces -o jsonpath='{range .items[*].status.initContainerStatuses[*]}{.containerID}{"\n"}{end}' | cut -d/ -f3

# 列出事件（Events），按时间戳排序
$ kubectl get events --sort-by=.metadata.creationTimestamp

# 比较当前的集群状态和假定某清单被应用之后的集群状态
$ kubectl diff -f ./my-manifest.yaml

# 生成一个句点分隔的树，其中包含为节点返回的所有键
# 在复杂的嵌套JSON结构中定位键时非常有用
$ kubectl get nodes -o json | jq -c 'path(..)|[.[]|tostring]|join(".")'

# 生成一个句点分隔的树，其中包含为pod等返回的所有键
$ kubectl get pods -o json | jq -c 'path(..)|[.[]|tostring]|join(".")'
```



### 更新资源

```bash
# 滚动更新 "frontend" Deployment 的 "www" 容器镜像
$ kubectl set image deployment/frontend www=image:v2 

# 检查 Deployment 的历史记录，包括版本
$ kubectl rollout history deployment/frontend  

# 回滚到上次部署版本
$ kubectl rollout undo deployment/frontend

# 回滚到特定部署版本
$ kubectl rollout undo deployment/frontend --to-revision=2       

# 监视 "frontend" Deployment 的滚动升级状态直到完成
$ kubectl rollout status -w deployment/frontend  

# 轮替重启 "frontend" Deployment
$ kubectl rollout restart deployment/frontend                      

# 通过传入到标准输入的 JSON 来替换 Pod
$ cat pod.json | kubectl replace -f -                              

# 强制替换，删除后重建资源。会导致服务不可用。
$ kubectl replace --force -f ./pod.json

# 为多副本的 nginx 创建服务，使用 80 端口提供服务，连接到容器的 8000 端口。
$ kubectl expose rc nginx --port=80 --target-port=8000

# 将某单容器 Pod 的镜像版本（标签）更新到 v4
$ kubectl get pod mypod -o yaml | sed 's/\(image: myimage\):.*$/\1:v4/' | kubectl replace -f -

# 添加标签
$ kubectl label pods my-pod new-label=awesome     

# 添加注解
$ kubectl annotate pods my-pod icon-url=http://goo.gl/XXBTWq  

# 对 "foo" Deployment 自动伸缩容
$ kubectl autoscale deployment foo --min=2 --max=10                
```



### 部分更新资源

```bash
# 部分更新某节点
$ kubectl patch node k8s-node-1 -p '{"spec":{"unschedulable":true}}'

# 更新容器的镜像；spec.containers[*].name 是必须的。因为它是一个合并性质的主键。
$ kubectl patch pod valid-pod -p '{"spec":{"containers":[{"name":"kubernetes-serve-hostname","image":"new image"}]}}'

# 使用带位置数组的 JSON patch 更新容器的镜像
$ kubectl patch pod valid-pod --type='json' -p='[{"op": "replace", "path": "/spec/containers/0/image", "value":"new image"}]'

# 使用带位置数组的 JSON patch 禁用某 Deployment 的 livenessProbe
$ kubectl patch deployment valid-deployment  --type json   -p='[{"op": "remove", "path": "/spec/template/spec/containers/0/livenessProbe"}]'

# 在带位置数组中添加元素
$ kubectl patch sa default --type='json' -p='[{"op": "add", "path": "/secrets/1", "value": {"name": "whatever" } }]'
```



### 编辑资源

使用你偏爱的编辑器编辑 API 资源。

```bash
$ kubectl edit svc/docker-registry                    # 编辑名为docker-registry的服务

$KUBE_EDITOR="nano" kubectl edit svc/docker-registry   # 使用其他编辑器
```



### 对资源进行伸缩

```bash
# 将名为 'foo' 的副本集伸缩到 3 副本
$ kubectl scale --replicas=3 rs/foo       

# 将在 "foo.yaml" 中的特定资源伸缩到 3 个副本
$ kubectl scale --replicas=3 -f foo.yaml          

# 如果名为 mysql 的 Deployment 的副本当前是 2，那么将它伸缩到 3
$ kubectl scale --current-replicas=2 --replicas=3 deployment/mysql 

 # 伸缩多个副本控制器
$ kubectl scale --replicas=5 rc/foo rc/bar rc/baz                  
```



### 删除资源

```bash
 # 删除在 pod.json 中指定的类型和名称的 Pod
$ kubectl delete -f ./pod.json        

# 删除名称为 "baz" 和 "foo" 的 Pod 和服务
$ kubectl delete pod,service baz foo   

# 删除包含 name=myLabel 标签的 pods 和服务
$ kubectl delete pods,services -l name=myLabel 

# 删除在 my-ns 名字空间中全部的 Pods 和服务
$ kubectl -n my-ns delete pod,svc --all 

# 删除所有与 pattern1 或 pattern2 awk 模式匹配的 Pods
$ kubectl get pods  -n mynamespace --no-headers=true | awk '/pattern1|pattern2/{print $1}' | xargs  kubectl delete -n mynamespace pod
```



### 与运行中的 Pods 进行交互

```bash
# 获取 pod 日志（标准输出）
$ kubectl logs my-pod    

# 获取含 name=myLabel 标签的 Pods 的日志（标准输出）
$ kubectl logs -l name=myLabel      

# 获取上个容器实例的 pod 日志（标准输出）
$ kubectl logs my-pod --previous    

# 获取 Pod 容器的日志（标准输出, 多容器场景）
$ kubectl logs my-pod -c my-container   

# 获取含 name=myLabel 标签的 Pod 容器日志（标准输出, 多容器场景）
$ kubectl logs -l name=myLabel -c my-container    

# 获取 Pod 中某容器的上个实例的日志（标准输出, 多容器场景）
$ kubectl logs my-pod -c my-container --previous     

# 流式输出 Pod 的日志（标准输出）
$ kubectl logs -f my-pod      

 # 流式输出 Pod 容器的日志（标准输出, 多容器场景）
$ kubectl logs -f my-pod -c my-container      

# 流式输出含 name=myLabel 标签的 Pod 的所有日志（标准输出）
$ kubectl logs -f -l name=myLabel --all-containers    

 # 以交互式 Shell 运行 Pod
$ kubectl run -i --tty busybox --image=busybox -- sh 

# 在指定名字空间中运行 nginx Pod
$ kubectl run nginx --image=nginx -n mynamespace  

# 运行 ngins Pod 并将其规约写入到名为 pod.yaml 的文件
$ kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml

# 挂接到一个运行的容器中
$ kubectl attach my-pod -i        

# 在本地计算机上侦听端口 5000 并转发到 my-pod 上的端口 6000
$ kubectl port-forward my-pod 5000:6000         

# 在已有的 Pod 中运行命令（单容器场景）
$ kubectl exec my-pod -- ls /      

# 使用交互 shell 访问正在运行的 Pod (一个容器场景)
$ kubectl exec --stdin --tty my-pod -- /bin/sh        

# 在已有的 Pod 中运行命令（多容器场景）
$ kubectl exec my-pod -c my-container -- ls /     

# 显示给定 Pod 和其中容器的监控数据
$ kubectl top pod POD_NAME --containers              
```



### 与节点和集群进行交互

```sh
# 标记 my-node 节点为不可调度
$ kubectl cordon my-node

# 对 my-node 节点进行清空操作，为节点维护做准备
$ kubectl drain my-node

# 标记 my-node 节点为可以调度
$ kubectl uncordon my-node

# 显示给定节点的度量值
$ kubectl top node my-node  

# 显示主控节点和服务的地址
$ kubectl cluster-info  

# 将当前集群状态转储到标准输出
$ kubectl cluster-info dump      

# 将当前集群状态输出到 /path/to/cluster-state
$ kubectl cluster-info dump --output-directory=/path/to/cluster-state   

# 如果已存在具有指定键和效果的污点，则替换其值为指定值。
$ kubectl taint nodes foo dedicated=special-user:NoSchedule
```



### 资源类型

列出所支持的全部资源类型和它们的简称、[API 组](https://kubernetes.io/zh/docs/concepts/overview/kubernetes-api/#api-groups), 是否是[名字空间作用域](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/namespaces) 和 [Kind](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/kubernetes-objects)。

```bash
$ kubectl api-resources
```

用于探索 API 资源的其他操作：

```bash
$ kubectl api-resources --namespaced=true      # 所有命名空间作用域的资源
$ kubectl api-resources --namespaced=false     # 所有非命名空间作用域的资源
$ kubectl api-resources -o name                # 用简单格式列举所有资源（仅显示资源名称）
$ kubectl api-resources -o wide                # 用扩展格式列举所有资源（又称 "wide" 格式）
$ kubectl api-resources --verbs=list,get       # 支持 "list" 和 "get" 请求动词的所有资源
$ kubectl api-resources --api-group=extensions # "extensions" API 组中的所有资源
```



### 格式化输出

要以特定格式将详细信息输出到终端窗口，将 `-o`（或者 `--output`）参数添加到支持的 `kubectl` 命令中。

| 输出格式                            | 描述                                                         |
| ----------------------------------- | ------------------------------------------------------------ |
| `-o=custom-columns=<spec>`          | 使用逗号分隔的自定义列来打印表格                             |
| `-o=custom-columns-file=<filename>` | 使用 `<filename>` 文件中的自定义列模板打印表格               |
| `-o=json`                           | 输出 JSON 格式的 API 对象                                    |
| `-o=jsonpath=<template>`            | 打印 [jsonpath](https://kubernetes.io/zh/docs/reference/kubectl/jsonpath) 表达式中定义的字段 |
| `-o=jsonpath-file=<filename>`       | 打印在 `<filename>` 文件中定义的 [jsonpath](https://kubernetes.io/zh/docs/reference/kubectl/jsonpath) 表达式所指定的字段。 |
| `-o=name`                           | 仅打印资源名称而不打印其他内容                               |
| `-o=wide`                           | 以纯文本格式输出额外信息，对于 Pod 来说，输出中包含了节点名称 |
| `-o=yaml`                           | 输出 YAML 格式的 API 对象                                    |

使用 `-o=custom-columns` 的示例：

```bash
# 集群中运行着的所有镜像
$ kubectl get pods -A -o=custom-columns='DATA:spec.containers[*].image'

 # 除 "k8s.gcr.io/coredns:1.6.2" 之外的所有镜像
$ kubectl get pods -A -o=custom-columns='DATA:spec.containers[?(@.image!="k8s.gcr.io/coredns:1.6.2")].image'

# 输出 metadata 下面的所有字段，无论 Pod 名字为何
$ kubectl get pods -A -o=custom-columns='DATA:metadata.*'
```

有关更多示例，请参看 kubectl [参考文档](https://kubernetes.io/zh/docs/reference/kubectl/overview/#custom-columns)。



### Kubectl 日志输出详细程度和调试

Kubectl 日志输出详细程度是通过 `-v` 或者 `--v` 来控制的，参数后跟一个数字表示日志的级别。 Kubernetes 通用的日志习惯和相关的日志级别在 [这里](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-instrumentation/logging.md) 有相应的描述。

| 详细程度 | 描述                                                         |
| -------- | ------------------------------------------------------------ |
| `--v=0`  | 用于那些应该 *始终* 对运维人员可见的信息，因为这些信息一般很有用。 |
| `--v=1`  | 如果您不想要看到冗余信息，此值是一个合理的默认日志级别。     |
| `--v=2`  | 输出有关服务的稳定状态的信息以及重要的日志消息，这些信息可能与系统中的重大变化有关。这是建议大多数系统设置的默认日志级别。 |
| `--v=3`  | 包含有关系统状态变化的扩展信息。                             |
| `--v=4`  | 包含调试级别的冗余信息。                                     |
| `--v=5`  | 跟踪级别的详细程度。                                         |
| `--v=6`  | 显示所请求的资源。                                           |
| `--v=7`  | 显示 HTTP 请求头。                                           |
| `--v=8`  | 显示 HTTP 请求内容。                                         |
| `--v=9`  | 显示 HTTP 请求内容而且不截断内容。                           |



### 接下来

- 参阅 [kubectl 概述](https://kubernetes.io/zh/docs/reference/kubectl/overview/)，进一步了解[JsonPath](https://kubernetes.io/zh/docs/reference/kubectl/jsonpath)。
- 参阅 [kubectl](https://kubernetes.io/zh/docs/reference/kubectl/kubectl/) 选项。
- 参阅 [kubectl 使用约定](https://kubernetes.io/zh/docs/reference/kubectl/conventions/)来理解如何在可复用的脚本中使用它。
- 查看社区中其他的 [kubectl 备忘单](https://github.com/dennyzhang/cheatsheet-kubernetes-A4)。





## Kubectl Kubernetes CheatSheet



### 1.1 Common Commands

| Name                                                         | Command                                                      |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| Run curl test temporarily                                    | `kubectl run --generator=run-pod/v1 --rm mytest --image=yauritux/busybox-curl -it` |
| Run wget test temporarily                                    | `kubectl run --generator=run-pod/v1 --rm mytest --image=busybox -it wget` |
| Run nginx deployment with 2 replicas                         | `kubectl run my-nginx --image=nginx --replicas=2 --port=80`  |
| Run nginx pod and expose it                                  | `kubectl run my-nginx --restart=Never --image=nginx --port=80 --expose` |
| Run nginx deployment and expose it                           | `kubectl run my-nginx --image=nginx --port=80 --expose`      |
| List authenticated contexts                                  | `kubectl config get-contexts`, `~/.kube/config`              |
| Set namespace preference                                     | `kubectl config set-context <context_name> --namespace=<ns_name>` |
| List pods with nodes info                                    | `kubectl get pod -o wide`                                    |
| List everything                                              | `kubectl get all --all-namespaces`                           |
| Get all services                                             | `kubectl get service --all-namespaces`                       |
| Get all deployments                                          | `kubectl get deployments --all-namespaces`                   |
| Show nodes with labels                                       | `kubectl get nodes --show-labels`                            |
| Get resources with json output                               | `kubectl get pods --all-namespaces -o json`                  |
| Validate yaml file with dry run                              | `kubectl create --dry-run --validate -f pod-dummy.yaml`      |
| Start a temporary pod for testing                            | `kubectl run --rm -i -t --image=alpine test-$RANDOM -- sh`   |
| kubectl run shell command                                    | `kubectl exec -it mytest -- ls -l /etc/hosts`                |
| Get system conf via configmap                                | `kubectl -n kube-system get cm kubeadm-config -o yaml`       |
| Get deployment yaml                                          | `kubectl -n denny-websites get deployment mysql -o yaml`     |
| Explain resource                                             | `kubectl explain pods`, `kubectl explain svc`                |
| Watch pods                                                   | `kubectl get pods -n wordpress --watch`                      |
| Query healthcheck endpoint                                   | `curl -L http://127.0.0.1:10250/healthz`                     |
| Open a bash terminal in a pod                                | `kubectl exec -it storage sh`                                |
| Check pod environment variables                              | `kubectl exec redis-master-ft9ex env`                        |
| Enable kubectl shell autocompletion                          | `echo "source <(kubectl completion bash)" >>~/.bashrc`, and reload |
| Use minikube dockerd in your laptop                          | `eval $(minikube docker-env)`, No need to push docker hub any more |
| Kubectl apply a folder of yaml files                         | `kubectl apply -R -f .`                                      |
| Get services sorted by name                                  | `kubectl get services –sort-by=.metadata.name`               |
| Get pods sorted by restart count                             | `kubectl get pods –sort-by=’.status.containerStatuses[0].restartCount’` |
| List pods and images                                         | `kubectl get pods -o=’custom-columns=PODS:.metadata.name,Images:.spec.containers[*].image’` |
| List all container images                                    | [list-all-images.sh](https://github.com/dennyzhang/cheatsheet-kubernetes-A4/blob/master/list-all-images.sh#L14-L17) |
| kubeconfig skip tls verification                             | [skip-tls-verify.md](https://github.com/dennyzhang/cheatsheet-kubernetes-A4/blob/master/skip-tls-verify.md) |
| [Ubuntu install kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) | =”deb https://apt.kubernetes.io/ kubernetes-xenial main”=    |
| Reference                                                    | [GitHub: kubernetes releases](https://github.com/kubernetes/kubernetes/tags) |
| Reference                                                    | [minikube cheatsheet](https://cheatsheet.dennyzhang.com/cheatsheet-minikube-A4), [docker cheatsheet](https://cheatsheet.dennyzhang.com/cheatsheet-docker-A4), [OpenShift CheatSheet](https://cheatsheet.dennyzhang.com/cheatsheet-openshift-A4) |



### 1.2 Check Performance

| Name                                         | Command                                              |
| -------------------------------------------- | ---------------------------------------------------- |
| Get node resource usage                      | `kubectl top node`                                   |
| Get pod resource usage                       | `kubectl top pod`                                    |
| Get resource usage for a given pod           | `kubectl top <podname> --containers`                 |
| List resource utilization for all containers | `kubectl top pod --all-namespaces --containers=true` |



### 1.3 Resources Deletion

| Name                                    | Command                                                  |
| --------------------------------------- | -------------------------------------------------------- |
| Delete pod                              | `kubectl delete pod/<pod-name> -n <my-namespace>`        |
| Delete pod by force                     | `kubectl delete pod/<pod-name> --grace-period=0 --force` |
| Delete pods by labels                   | `kubectl delete pod -l env=test`                         |
| Delete deployments by labels            | `kubectl delete deployment -l app=wordpress`             |
| Delete all resources filtered by labels | `kubectl delete pods,services -l name=myLabel`           |
| Delete resources under a namespace      | `kubectl -n my-ns delete po,svc --all`                   |
| Delete persist volumes by labels        | `kubectl delete pvc -l app=wordpress`                    |
| Delete state fulset only (not pods)     | `kubectl delete sts/<stateful_set_name> --cascade=false` |



### 1.4 Log & Conf Files

| Name                      | Comment                                                      |
| ------------------------- | ------------------------------------------------------------ |
| Config folder             | `/etc/kubernetes/`                                           |
| Certificate files         | `/etc/kubernetes/pki/`                                       |
| Credentials to API server | `/etc/kubernetes/kubelet.conf`                               |
| Superuser credentials     | `/etc/kubernetes/admin.conf`                                 |
| kubectl config file       | `~/.kube/config`                                             |
| Kubernetes working dir    | `/var/lib/kubelet/`                                          |
| Docker working dir        | `/var/lib/docker/`, `/var/log/containers/`                   |
| Etcd working dir          | `/var/lib/etcd/`                                             |
| Network cni               | `/etc/cni/net.d/`                                            |
| Log files                 | `/var/log/pods/`                                             |
| log in worker node        | `/var/log/kubelet.log`, `/var/log/kube-proxy.log`            |
| log in master node        | `kube-apiserver.log`, `kube-scheduler.log`, `kube-controller-manager.log` |
| Env                       | `/etc/systemd/system/kubelet.service.d/10-kubeadm.conf`      |
| Env                       | export KUBECONFIG=/etc/kubernetes/admin.conf                 |



### 1.5 Pod

| Name                                                         | Command                                                      |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| List all pods                                                | `kubectl get pods`                                           |
| List pods for all namespace                                  | `kubectl get pods -all-namespaces`                           |
| List all critical pods                                       | `kubectl get -n kube-system pods -a`                         |
| List pods with more info                                     | `kubectl get pod -o wide`, `kubectl get pod/<pod-name> -o yaml` |
| Get pod info                                                 | `kubectl describe pod/srv-mysql-server`                      |
| List all pods with labels                                    | `kubectl get pods --show-labels`                             |
| [List all unhealthy pods](https://github.com/kubernetes/kubernetes/issues/49387) | `kubectl get pods –field-selector=status.phase!=Running –all-namespaces` |
| List running pods                                            | `kubectl get pods –field-selector=status.phase=Running`      |
| Get Pod initContainer status                                 | `kubectl get pod --template '{{.status.initContainerStatuses}}' <pod-name>` |
| kubectl run command                                          | `kubectl exec -it -n “$ns” “$podname” – sh -c “echo $msg >>/dev/err.log”` |
| Watch pods                                                   | `kubectl get pods -n wordpress --watch`                      |
| Get pod by selector                                          | `kubectl get pods –selector=”app=syslog” -o jsonpath=’{.items[*].metadata.name}’` |
| List pods and images                                         | `kubectl get pods -o=’custom-columns=PODS:.metadata.name,Images:.spec.containers[*].image’` |
| List pods and containers                                     | `-o=’custom-columns=PODS:.metadata.name,CONTAINERS:.spec.containers[*].name’` |
| Reference                                                    | [Link: kubernetes yaml templates](https://cheatsheet.dennyzhang.com/kubernetes-yaml-templates) |



### 1.6 Label & Annotation

| Name                             | Command                                                      |
| -------------------------------- | ------------------------------------------------------------ |
| Filter pods by label             | `kubectl get pods -l owner=denny`                            |
| Manually add label to a pod      | `kubectl label pods dummy-input owner=denny`                 |
| Remove label                     | `kubectl label pods dummy-input owner-`                      |
| Manually add annotation to a pod | `kubectl annotate pods dummy-input my-url=https://dennyzhang.com` |



### 1.7 Deployment & Scale

| Name                         | Command                                                      |
| ---------------------------- | ------------------------------------------------------------ |
| Scale out                    | `kubectl scale --replicas=3 deployment/nginx-app`            |
| online rolling upgrade       | `kubectl rollout app-v1 app-v2 --image=img:v2`               |
| Roll backup                  | `kubectl rollout app-v1 app-v2 --rollback`                   |
| List rollout                 | `kubectl get rs`                                             |
| Check update status          | `kubectl rollout status deployment/nginx-app`                |
| Check update history         | `kubectl rollout history deployment/nginx-app`               |
| Pause/Resume                 | `kubectl rollout pause deployment/nginx-deployment`, `resume` |
| Rollback to previous version | `kubectl rollout undo deployment/nginx-deployment`           |
| Reference                    | [Link: kubernetes yaml templates](https://cheatsheet.dennyzhang.com/kubernetes-yaml-templates), [Link: Pausing and Resuming a Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#pausing-and-resuming-a-deployment) |



### 1.8 Quota & Limits & Resource

| Name                          | Command                                                      |
| ----------------------------- | ------------------------------------------------------------ |
| List Resource Quota           | `kubectl get resourcequota`                                  |
| List Limit Range              | `kubectl get limitrange`                                     |
| Customize resource definition | `kubectl set resources deployment nginx -c=nginx --limits=cpu=200m` |
| Customize resource definition | `kubectl set resources deployment nginx -c=nginx --limits=memory=512Mi` |
| Reference                     | [Link: kubernetes yaml templates](https://cheatsheet.dennyzhang.com/kubernetes-yaml-templates) |



### 1.9 Service

| Name                            | Command                                                      |
| ------------------------------- | ------------------------------------------------------------ |
| List all services               | `kubectl get services`                                       |
| List service endpoints          | `kubectl get endpoints`                                      |
| Get service detail              | `kubectl get service nginx-service -o yaml`                  |
| Get service cluster ip          | `kubectl get service nginx-service -o go-template='{{.spec.clusterIP}}'` |
| Get service cluster port        | `kubectl get service nginx-service -o go-template='{{(index .spec.ports 0).port}}'` |
| Expose deployment as lb service | `kubectl expose deployment/my-app --type=LoadBalancer --name=my-service` |
| Expose service as lb service    | `kubectl expose service/wordpress-1-svc --type=LoadBalancer --name=ns1` |
| Reference                       | [Link: kubernetes yaml templates](https://cheatsheet.dennyzhang.com/kubernetes-yaml-templates) |



### 1.10 Secrets

| Name                             | Command                                                      |
| -------------------------------- | ------------------------------------------------------------ |
| List secrets                     | `kubectl get secrets --all-namespaces`                       |
| Generate secret                  | `echo -n 'mypasswd', then redirect to base64 --decode`       |
| Get secret                       | `kubectl get secret denny-cluster-kubeconfig`                |
| Get a specific field of a secret | `kubectl get secret denny-cluster-kubeconfig -o jsonpath="{.data.value}"` |
| Create secret from cfg file      | `kubectl create secret generic db-user-pass –from-file=./username.txt` |
| Reference                        | [Link: kubernetes yaml templates](https://cheatsheet.dennyzhang.com/kubernetes-yaml-templates), [Link: Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) |



### 1.11 StatefulSet

| Name                               | Command                                                      |
| ---------------------------------- | ------------------------------------------------------------ |
| List statefulset                   | `kubectl get sts`                                            |
| Delete statefulset only (not pods) | `kubectl delete sts/<stateful_set_name> --cascade=false`     |
| Scale statefulset                  | `kubectl scale sts/<stateful_set_name> --replicas=5`         |
| Reference                          | [Link: kubernetes yaml templates](https://cheatsheet.dennyzhang.com/kubernetes-yaml-templates) |



### 1.12 Volumes & Volume Claims

| Name                      | Command                                                      |
| ------------------------- | ------------------------------------------------------------ |
| List storage class        | `kubectl get storageclass`                                   |
| Check the mounted volumes | `kubectl exec storage ls /data`                              |
| Check persist volume      | `kubectl describe pv/pv0001`                                 |
| Copy local file to pod    | `kubectl cp /tmp/my <some-namespace>/<some-pod>:/tmp/server` |
| Copy pod file to local    | `kubectl cp <some-namespace>/<some-pod>:/tmp/server /tmp/my` |
| Reference                 | [Link: kubernetes yaml templates](https://cheatsheet.dennyzhang.com/kubernetes-yaml-templates) |



### 1.13 Events & Metrics

| Name                            | Command                                                   |
| ------------------------------- | --------------------------------------------------------- |
| View all events                 | `kubectl get events --all-namespaces`                     |
| List Events sorted by timestamp | `kubectl get events –sort-by=.metadata.creationTimestamp` |



### 1.14 Node Maintenance

| Name                                      | Command                       |
| ----------------------------------------- | ----------------------------- |
| Mark node as unschedulable                | `kubectl cordon $NODE_NAME`   |
| Mark node as schedulable                  | `kubectl uncordon $NODE_NAME` |
| Drain node in preparation for maintenance | `kubectl drain $NODE_NAME`    |



### 1.15 Namespace & Security

| Name                                                         | Command                                                      |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| List authenticated contexts                                  | `kubectl config get-contexts`, `~/.kube/config`              |
| Set namespace preference                                     | `kubectl config set-context <context_name> --namespace=<ns_name>` |
| Switch context                                               | `kubectl config use-context <context_name>`                  |
| Load context from config file                                | `kubectl get cs --kubeconfig kube_config.yml`                |
| Delete the specified context                                 | `kubectl config delete-context <context_name>`               |
| List all namespaces defined                                  | `kubectl get namespaces`                                     |
| List certificates                                            | `kubectl get csr`                                            |
| [Check user privilege](https://kubernetes.io/docs/concepts/policy/pod-security-policy/) | `kubectl --as=system:serviceaccount:ns-denny:test-privileged-sa -n ns-denny auth can-i use pods/list` |
| [Check user privilege](https://kubernetes.io/docs/concepts/policy/pod-security-policy/) | `kubectl auth can-i use pods/list`                           |
| Reference                                                    | [Link: kubernetes yaml templates](https://cheatsheet.dennyzhang.com/kubernetes-yaml-templates) |



### 1.16 Network

| Name                               | Command                                                  |
| ---------------------------------- | -------------------------------------------------------- |
| Temporarily add a port-forwarding  | `kubectl port-forward redis-134 6379:6379`               |
| Add port-forwarding for deployment | `kubectl port-forward deployment/redis-master 6379:6379` |
| Add port-forwarding for replicaset | `kubectl port-forward rs/redis-master 6379:6379`         |
| Add port-forwarding for service    | `kubectl port-forward svc/redis-master 6379:6379`        |
| Get network policy                 | `kubectl get NetworkPolicy`                              |



### 1.17 Patch

| Name                          | Summary                                                      |
| ----------------------------- | ------------------------------------------------------------ |
| Patch service to loadbalancer | `kubectl patch svc $svc_name -p ‘{“spec”: {“type”: “LoadBalancer”}}’` |



### 1.18 Extenstions

| Name                                    | Summary                    |
| --------------------------------------- | -------------------------- |
| Enumerates the resource types available | `kubectl api-resources`    |
| List api group                          | `kubectl api-versions`     |
| List all CRD                            | `kubectl get crd`          |
| List storageclass                       | `kubectl get storageclass` |



### 1.19 Components & Services

#### 1.19.1 Services on Master Nodes

| Name                                                         | Summary                                                      |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| [kube-apiserver](https://github.com/kubernetes/kubernetes/tree/master/cmd/kube-apiserver) | API gateway. Exposes the Kubernetes API from master nodes    |
| [etcd](https://coreos.com/etcd/)                             | reliable data store for all k8s cluster data                 |
| [kube-scheduler](https://github.com/kubernetes/kubernetes/tree/master/cmd/kube-scheduler) | schedule pods to run on selected nodes                       |
| [kube-controller-manager](https://github.com/kubernetes/kubernetes/tree/master/cmd/kube-controller-manager) | Reconcile the states. node/replication/endpoints/token controller and service account, etc |
| cloud-controller-manager                                     |                                                              |



#### 1.19.2 Services on Worker Nodes

| Name                                                         | Summary                                                      |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| [kubelet](https://github.com/kubernetes/kubernetes/tree/master/cmd/kubelet) | A node agent makes sure that containers are running in a pod |
| [kube-proxy](https://github.com/kubernetes/kubernetes/tree/master/cmd/kube-proxy) | Manage network connectivity to the containers. e.g, iptable, ipvs |
| [Container Runtime](https://github.com/docker/engine)        | Kubernetes supported runtimes: dockerd, cri-o, runc and any [OCI runtime-spec](https://github.com/opencontainers/runtime-spec) implementation. |



#### 1.19.3 Addons: pods and services that implement cluster features

| Name                          | Summary                                                      |
| ----------------------------- | ------------------------------------------------------------ |
| DNS                           | serves DNS records for Kubernetes services                   |
| Web UI                        | a general purpose, web-based UI for Kubernetes clusters      |
| Container Resource Monitoring | collect, store and serve container metrics                   |
| Cluster-level Logging         | save container logs to a central log store with search/browsing interface |



#### 1.19.4 Tools

| Name                                                         | Summary                                                      |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| [kubectl](https://github.com/kubernetes/kubernetes/tree/master/cmd/kubectl) | the command line util to talk to k8s cluster                 |
| [kubeadm](https://github.com/kubernetes/kubernetes/tree/master/cmd/kubeadm) | the command to bootstrap the cluster                         |
| [kubefed](https://kubernetes.io/docs/reference/setup-tools/kubefed/kubefed/) | the command line to control a Kubernetes Cluster Federation  |
| Kubernetes Components                                        | [Link: Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/) |



### 1.20 More Resources

License: Code is licensed under [MIT License](https://www.dennyzhang.com/wp-content/mit_license.txt).

https://kubernetes.io/docs/reference/kubectl/cheatsheet/

https://codefresh.io/kubernetes-guides/kubernetes-cheat-sheet/











## Kubernetes 最佳安全实践指南



对于大部分 Kubernetes 用户来说，安全是无关紧要的，或者说没那么紧要，就算考虑到了，也只是敷衍一下，草草了事。实际上 Kubernetes 提供了非常多的选项可以大大提高应用的安全性，只要用好了这些选项，就可以将绝大部分的攻击抵挡在门外。本文所述的最佳安全实践仅限于 Pod 层面，也就是容器层面，于容器的生命周期相关，至于容器之外的安全配置（比如操作系统啦、k8s 组件啦），以后有机会再唠。



### 为容器配置 Security Context

大部分情况下容器不需要太多的权限，我们可以通过 `Security Context` 限定容器的权限和访问控制，只需加上 SecurityContext 字段：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: <Pod name>
spec:
  containers:
  - name: <container name>
  image: <image>
    securityContext:
```



### 禁用 allowPrivilegeEscalation

`allowPrivilegeEscalation=true` 表示容器的任何子进程都可以获得比父进程更多的权限。最好将其设置为 false，以确保 `RunAsUser` 命令不能绕过其现有的权限集。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: <Pod name>
spec:
  containers:
  - name: <container name>
  image: <image>
    securityContext:
      allowPrivilegeEscalation: false
```



### 不要使用 root 用户

为了防止来自容器内的提权攻击，最好不要使用 root 用户运行容器内的应用。UID 设置大一点，尽量大于 `3000`。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: <name>
spec:
  securityContext:
    runAsUser: <UID higher than 1000>
    runAsGroup: <UID higher than 3000>
```



###  限制 CPU 和内存资源

pod资源 requests 和 limits 都加上。



###  不必挂载 Service Account Token

ServiceAccount 为 Pod 中运行的进程提供身份标识，怎么标识呢？当然是通过 Token 啦，有了 Token，就防止假冒伪劣进程。如果你的应用不需要这个身份标识，可以不必挂载：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: <name>
spec:
  automountServiceAccountToken: false
```



### 确保 seccomp 设置正确

对于 Linux 来说，用户层一切资源相关操作都需要通过系统调用来完成，那么只要对系统调用进行某种操作，用户层的程序就翻不起什么风浪，即使是恶意程序也就只能在自己进程内存空间那一分田地晃悠，进程一终止它也如风消散了。seccomp（secure computing mode）就是一种限制系统调用的安全机制，可以可以指定允许那些系统调用。

对于 Kubernetes 来说，大多数容器运行时都提供一组允许或不允许的默认系统调用。通过使用 `runtime/default` 注释或将 Pod 或容器的安全上下文中的 seccomp 类型设置为 `RuntimeDefault`，可以轻松地在 Kubernetes 中应用默认值。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: <name>
  annotations:
    seccomp.security.alpha.kubernetes.io/pod: "runtime/default"
```

默认的 seccomp 配置文件应该为大多数工作负载提供足够的权限。如果你有更多的需求，可以自定义配置文件。



### 限制容器的 capabilities

容器依赖于传统的 Unix 安全模型，通过控制资源所属用户和组的权限，来达到对资源的权限控制。以 root 身份运行的容器拥有的权限远远超过其工作负载的要求，一旦发生泄露，攻击者可以利用这些权限进一步对网络进行攻击。

默认情况下，使用 Docker 作为容器运行时，会启用 `NET_RAW` capability，这可能会被恶意攻击者进行滥用。因此，建议至少定义一个`PodSecurityPolicy`(PSP)，以防止具有 NET_RAW 功能的容器启动。

通过限制容器的 capabilities，可以确保受攻击的容器无法为攻击者提供横向攻击的有效路径，从而缩小攻击范围。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: <name>
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: <specific user>
  capabilities:
  drop:
    -NET_RAW
    -ALL
```

- [👉Linux Capabilities 入门教程：概念篇](http://mp.weixin.qq.com/s?__biz=MzU1MzY4NzQ1OA==&mid=2247484610&idx=1&sn=0f75f48b1651f03163bef421280c25f8&chksm=fbee440fcc99cd19c786acd3de00fee7914664171395013bb3fb4e1dfb1a84f618fc1a9b042e&scene=21#wechat_redirect)
- [👉Linux Capabilities 入门教程：基础实战篇](http://mp.weixin.qq.com/s?__biz=MzU1MzY4NzQ1OA==&mid=2247484631&idx=1&sn=6d65cfd09e61f0b56967867a190ca48e&chksm=fbee441acc99cd0cfcd0042021e4d1bac0eb67a6797d7a73234dc746cca6fc3494e38b6c0c4c&scene=21#wechat_redirect)
- [👉Linux Capabilities 入门教程：进阶实战篇](http://mp.weixin.qq.com/s?__biz=MzU1MzY4NzQ1OA==&mid=2247488377&idx=1&sn=430ba2812e467ccfcf12b29f88e0215a&chksm=fbee53b4cc99daa22d9c67bb1f09d93da08d02010752e204b49aca4b35858993d4bab82e8629&scene=21#wechat_redirect)



### 只读

如果容器不需要对根文件系统进行写入操作，最好以只读方式加载容器的根文件系统，可以进一步限制攻击者的手脚。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: <Pod name>
spec:
  containers:
  - name: <container name>
  image: <image>
  securityContext:
    readOnlyRootFilesystem: true
```



### 总结

总之，Kubernetes 提供了非常多的选项来增强集群的安全性，没有一个放之四海而皆准的解决方案，所以需要对这些选项非常熟悉，以及了解它们是如何增强应用程序的安全性，才能使集群更加稳定安全。





## Kubernetes 调度详解



Kubernetes Scheduler 是 Kubernetes 控制平面的核心组件之一。它在控制平面上运行，将 Pod 分配给节点，同时平衡节点之间的资源利用率。将 Pod 分配给新节点后，在该节点上运行的 kubelet 会在 Kubernetes API 中检索 Pod 定义，根据节点上的 Pod 规范创建资源和容器。换句话说，**Scheduler 在控制平面内运行，并将工作负载分配给 Kubernetes 集群**。

本文将对 Kubernetes Scheduler 进行深入研究，首先概述一般的调度以及具有亲和力（affinity）和 taint 的驱逐调度，然后讨论调度程序的瓶颈以及生产中可能遇到的问题，最后研究如何微调调度程序的参数以适合集群。



### 调度简介

**Kubernetes 调度是将 Pod 分配给集群中匹配节点的过程。**Scheduler 监控新创建的 Pod，并为其分配最佳节点。它会根据 Kubernetes 的调度原则和我们的配置选项选择最佳节点。最简单的配置选项是直接在 `PodSpec` 设置 nodeName：

![image-20210204162931651](D:\学习资料\笔记\k8s\k8s图\image-20210204162931651.png)

上面的 nginx pod 默认情况下将在 node-01 上运行，但是 nodeName 有许多限制导致无法正常运行 Pod，例如云中节点名称未知、资源节点不足以及节点网络间歇性问题等。因此，除了测试或开发期间，我们最好不使用 nodeName。

如果要在一组特定的节点上运行 Pod，可以使用 nodeSelector。我们在 `PodSpec` 中将 nodeSelector 定义为一组键值对：

![image-20210204163019057](D:\学习资料\笔记\k8s\k8s图\image-20210204163019057.png)

对于上面的 nginx pod，Kubernetes Scheduler 将找到一个磁盘类型为 ssd 的节点。当然，该节点可以具有其他标签。我们可以在 Kubernetes 参考文档中查看标签的完整列表。

地址：https://kubernetes.io/docs/reference/kubernetes-api/labels-annotations-taints/

使用 `nodeSelector` 有约束 Pod 可以在有特定标签的节点上运行。但它的使用仅受标签及其值限制。Kubernetes 中有两个更全面的功能来表达更复杂的调度需求：节点亲和力（node affinity），标记容器以将其吸引到一组节点上；taint 和 toleration，标记节点以排斥 Pod。这些功能将在下面讨论。



### 节点亲和力

**节点亲和力（Node Affinity）是在 Pod 上定义的一组约束，用于确定哪些节点适合进行调度，即使用亲和性规则为 Pod 的节点分配定义硬性要求和软性要求。**例如可以将 Pod 配置为仅运行带有 GPU 的节点，并且最好使用 NVIDIA_TESLA_V100 运行深度学习工作负载。Scheduler 会评估规则，并在定义的约束内找到合适的节点。与 `nodeSelectors` 相似，节点亲和性规则可与节点标签一起使用，但它比 `nodeSelectors` 更强大。

我们可以为 podspec 添加四个相似性规则：

- requiredDuringSchedulingIgnoredDuringExecution
- requiredDuringSchedulingRequiredDuringExecution
- preferredDuringSchedulingIgnoredDuringExecution
- preferredDuringSchedulingRequiredDuringExecution

**这四个规则由两个条件组成：必需或首选条件，以及两个阶段：计划和执行。**以 required 开头的规则描述了必须满足的严格要求。以 preferred 开头的规则是软性要求，将强制执行但不能保证。调度阶段是指将 Pod 首次分配给节点。执行阶段适用于在调度分配后节点标签发生更改的情况。

如果规则声明为 IgnoredDuringExecution，Scheduler 在第一次分配后不会检查其有效性。但如果使用 RequiredDuringExecution 指定了规则，Scheduler 会通过将容器移至合适的节点来确保规则的有效性。

以下是示例：

![image-20210204163414834](D:\学习资料\笔记\k8s\k8s图\image-20210204163414834.png)

上面的 Nginx Pod 具有节点亲和性规则，该规则让 Kubernetes Scheduler 将 Pod 放置在 us-east 的节点上。第二条规则指示优先使用 us-east-1 或 us-east-2。

使用亲和性规则，我们可以让 Kubernetes 调度决策适用于自定义需求。



### Taint 与 Toleration

集群中并非所有 Kubernetes 节点都相同。某些节点可能具有特殊的硬件，例如 GPU、磁盘或网络功能。同样，我们可能需要将一些节点专用于测试、数据保护或用户组。我们可以将 Taint 添加到节点以排斥 Pod，如以下示例所示：

```sh
kubectl taint nodes node1 test-environment=true:NoSchedule
```

使用 `test-environment=true:NoScheduletaint` 时，除非在 podspec 具有匹配的 toleration，否则 Kubernetes Scheduler 将不会分配任何 pod ：

![image-20210204163647507](D:\学习资料\笔记\k8s\k8s图\image-20210204163647507.png)

taint 和 tolerations 共同发挥作用，让 Kubernetes Scheduler 专用于某些节点并分配特定 Pod。



### 调度瓶颈

尽管 Kubernetes Scheduler 能选择最佳节点，但是在 Pod 开始运行之后，“最佳节点”可能会改变。所以从长远来看，Pod 的资源使用及其节点分配可能存在问题。

**资源请求（Request）和限制（Limit）：“Noisy Neighbor”**

“Noisy Neighbor”并不特定于 Kubernetes。任何多租户系统都是它们的潜在地。假设有两个容器 A 和 B，它们在同一节点上运行。如果 Pod B 试图通过消耗所有 CPU 或内存来创造 noise，Pod A 将出现问题。如果我们为容器设置了资源请求和限制就能控制住 neighbor。Kubernetes 将确保为容器安排其请求的资源，并且不会消耗超出其资源限制的资源。如果在生产中运行 Kubernetes，最好设置资源请求和限制以确保系统可靠。

**系统进程资源不足**

Kubernetes 节点主要是连接到 Kubernetes 控制平面的虚拟机。因此，节点上也有自己的操作系统和相关进程。如果 Kubernetes 工作负载消耗了所有资源，则这些节点将无法运行，并会发生各种问题问题。我们需要在 kubelet 中使用 –system -reserved 设置保留资源，以防止发生这种情况。

**抢占或调度 Pod**

如果 Kubernetes Scheduler 无法将 Pod 调度到可用节点，则可以从节点抢占（preempt）或驱逐（evict）一些 Pod 以分配资源。如果看到 Pod 在集群中移动而没有发现特定原因，可以使用优先级类对其进行定义。同样，如果没有调度好 Pod，并且正在等待其他 Pod，也需要检查其优先级。

以下是示例：

![image-20210204165013355](D:\学习资料\笔记\k8s\k8s图\image-20210204165013355.png)

可以通过以下方式在 podspec 中为分配优先级：

![image-20210204165104916](D:\学习资料\笔记\k8s\k8s图\image-20210204165104916.png)



### 调度框架

Kubernetes Scheduler 具有可插拔的调度框架架构，可向框架添加一组新的插件。插件实现 Plugin API，并被编译到调度程序中。下面我们将讨论调度框架的工作流、扩展点和 Plugin API。

**工作流和扩展点**

调度 Pod 包括两个阶段：调度周期（scheduling cycle）和绑定周期（binding cycle）。在调度周期中，Scheduler 会找到一个可用节点，然后在绑定过程中，将决策应用于集群。

下图说明了阶段和扩展点的流程：

![image-20210204165207114](D:\学习资料\笔记\k8s\k8s图\image-20210204165207114.png)

工作流中的以下几点对插件扩展开放：

- QueueSort：对队列中的 Pod 进行排序
- PreFilter：检查预处理 Pod 的相关信息以安排调度周期
- Filter：过滤不适合该 Pod 的节点
- PostFilter：如果找不到可用于 Pod 的可行节点，调用该插件
- PreScore：运行 PreScore 任务以生成一个可共享状态供 Score 插件使用
- Score：通过调用每个 Score 插件对过滤的节点进行排名
- NormalizeScore：合并分数并计算节点的最终排名
- Reserve：在绑定周期之前选择保留的节点
- Permit：批准或拒绝调度周期结果
- PreBind：执行任何先决条件工作，例如配置网络卷
- Bind：将 Pod 分配给 Kubernetes API 中的节点
- PostBind：通知绑定周期的结果

插件扩展实现了 Plugin API，是 Kubernetes Scheduler 的一部分。我们可以在 Kubernetes 存储库中检查。插件应使用以下名称进行注册：

![image-20210204165538489](D:\学习资料\笔记\k8s\k8s图\image-20210204165538489.png)

插件还实现了相关的扩展点，如下所示：

![image-20210204165555412](D:\学习资料\笔记\k8s\k8s图\image-20210204165555412.png)



### Scheduler 性能调整

Kubernetes Scheduler 有一个工作流来查找和绑定 Pod 的可行节点。当集群中的节点数量非常多时，Scheduler 的工作量将成倍增加。在大型集群中，可能需要很长时间才能找到最佳节点，因此要微调调度程序的性能，以在延迟和准确性之间找到折中方案。

percentageOfNodesToScore 将限制节点的数量来计算自己的分数。默认情况下，Kubernetes 在 100 节点集群的 50％ 和 5000 节点集群的 10％ 之间设置线性阈值。默认最小值为 5％，它要确保至少考虑集群中 5％ 节点的调度。

下面的示例展示了如何通过性能调整 kube-scheduler 来手动设置阈值：

![image-20210204165814270](D:\学习资料\笔记\k8s\k8s图\image-20210204165814270.png)

如果有一个庞大的集群并且 Kubernetes 工作负载不能承受 Kubernetes Scheduler 引起的延迟，那么更改百分比是个好主意。



### 总结

本文涵盖了 Kubernetes 调度的大多方面，从 Pod 和节点的配置开始，包括 nodeSelector、亲和性规则、taint 和 toleration，然后介绍了 Kubernetes Scheduler 框架、扩展点、API 以及可能发生的与资源相关的瓶颈，最后展示了性能调整设置。尽管 Kubernetes Scheduler 能简单地将 Pod 分配给节点，但是了解其动态性并对其进行配置以实现可靠的生产级 Kubernetes 设置至关重要。











































