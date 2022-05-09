Kubernetes 杂文

[toc]



## Azure容器平台初体验：利用AKS 部署你的容器服务



为了帮助大家更好地了解并使用Azure平台来运行自己的容器化服务和应用程序，我们准备了**《Azure容器服务完全锻造手册》系列文章，围绕微软云原生服务Azure Web Apps，Azure Kubernetes服务理论与实践，Azure API以及Functions，帮助大家基于Azure相关服务打造容器运行平台。**

该系列的第一篇文章[《一文看懂有关容器的“Why”和“How”》](https://mp.weixin.qq.com/s?__biz=MzA4MzA1OTc1MA==&mid=2649851661&idx=1&sn=4e61b093bb3f666d741e3ad41af5c04a&scene=21#wechat_redirect)已发布，错过的童鞋可以点击标题回看。**本文将是这一系列的第二篇，会向大家介绍如何利用Azure Kubernetes Service部署我们自己的容器服务。**



### 背景和趋势分析

虽然容器部署方式有很多种，但Kubernetes永远是最火的那个。Kubernetes的出现，弥补了容器数量不断增加，容器间彼此依赖不断复杂，基于容器构建复杂系统时，所需要考虑的高可用、安全、扩展、数据持久化等一系列问题。

**Kubernetes作为容器编排集群，其管理的核心并没有改变，仍然是容器。**过去最流行的莫过于Docker，将来应该主要是containerd（貌似这是最近大家都热议的一个话题）。Kubernetes所起的作用就是针对成百上千的容器以及运行在其中的服务，提出一套实用且扩展性强的管理方案。其实回过头来看，Kubernetes中，能够看得见摸得着的就是一个个的Pod，里面包含的是一个个的Container，其他所有资源，如Deployment、Service、StatefulSet等，其实都是逻辑概念，用来实现对于容器的编排管理。

![image-20210207100558434](D:\学习资料\笔记\k8s\k8s图\image-20210207100558434.png)

----

从上图可以看到，Kubernetes中有大大小小几十种资源类型，搞懂这些类型的概念与用途，是用好Kubernetes的基础。同时也应该注意到，在开始使用Kubernetes搭建容器平台时，会涉及到很多方面，如开发工具的集成、镜像的生成和管理、计算/存储/网络等基础设施的搭建、安全等诸多问题。有些Kubernetes平台提供了实现方式，有些需要借助额外的开源项目，有些需要专业技能的人员进行实现。

![image-20210207101004378](D:\学习资料\笔记\k8s\k8s图\image-20210207101004378.png)

**为了加快容器化改造过程，越来越多企业借助公有云平台上的托管Kubernetes服务 + 平台提供的PaaS服务来完成。其实，这种扩展性也体现了Kubernetes作为一款大众接受的容器化平台，自身的良好移植性。**

首先，所有云平台中的托管Kubernetes集群，都是基于社区版Kubernetes进行构建，随着社区版本的更新而更新，并未对Kubernetes核心代码进行任何更改。所以功能上与社区保持一致，也就是说，同样一个YML部署资源的文件，跑在任何一家托管的Kubernetes集群上都应该差不多，无需额外适配。

其次，云平台更多地提供了自己的基础设施及PaaS服务，如何更好适配Kubernetes中的逻辑资源概念，Kubernetes中的逻辑资源，如Persistent Volume，描述了其功能，提供了统一API，并提供了标准接口。但具体里面存储的数据是存储在AWS的EBS还是存储在Azure的Managed Disk里，取决于集群部署在哪里。

![image-20210207101158616](D:\学习资料\笔记\k8s\k8s图\image-20210207101158616.png)

其实，这种实现方式对Kubernetes作为中立的社区发展是有好处的：不受任何一家厂商绑定。对于厂商而言这是一种变相的鼓励，每个厂商基于Kubernetes提供的标准接口方式，以及其管理理念，适配自己的云平台。也就是说，云厂商投入到开源项目中的越多，用户在用你平台的托管Kubernetes时，需要自己动手的地方就越少。目前Kubernetes的核心代码库已经将云厂商的代码剥离出去，代码结构有一些变化：

![image-20210207101558596](D:\学习资料\笔记\k8s\k8s图\image-20210207101558596.png)

----



### 借助AKS服务部署自己的容器平台

**随后我们就一起看看，目前在Azure公有云中使用Azure Kubernetes Service构建容器平台的实践建议和配套服务。**

![image-20210207101648685](D:\学习资料\笔记\k8s\k8s图\image-20210207101648685.png)





#### 1. 环境准备

本次实验将以此[Azure Kubernetes Service Workshop](https://docs.microsoft.com/en-us/learn/modules/aks-workshop/)为基础进行练习，建议大家至少完成到Exercise - Deploy an  ingress for the front end这一步骤。

实验场景十分简单：一个标准的三层架构，包括API、Web和MongoDB。虽然简单，但能够实践Kubernetes中很多核心资源的概念。

![image-20210207102943209](D:\学习资料\笔记\k8s\k8s图\image-20210207102943209.png)



#### 2. AKS集群的基础架构

Azure  Kubernetes Service作为托管的Kubernetes服务，在集群的创建、维护、管理等方面，给用户提供了很多方便。对于用户，在熟悉Kubernetes是如何工作的前提下，需要了解的是：针对于“托管”二字，AKS相比于自建的Kubernetes集群到底托管了哪些服务？答案也简单，主要托管了：控制平面，如API-Server、Controller Manager、Scheduler、ETCD等。

![image-20210207103031680](D:\学习资料\笔记\k8s\k8s图\image-20210207103031680.png)

在一个初期的AKS应用架构设计上，大体上会按照如下的架构进行搭建：

![image-20210207103226069](D:\学习资料\笔记\k8s\k8s图\image-20210207103226069.png)

AKS中部署的是容器，容器运行依赖容器镜像。因此企业私有镜像仓库是AKS项目所必须的。Azure Container Registry是云上的私有镜像仓库，与AKS密不可分，AKS项目中都会存在。ACR的核心是存储容器镜像，具有一些特性：

- 不只支持Docker镜像，还支持OCI格式的镜像，支持Helm Charts
- 容器镜像的安全性保障，支持容器镜像扫描，基础镜像安全更新后自动重新构建镜像，镜像内容认证
- 容器镜像的生成一般是通过Dockerfile在本地构建好，再上传到镜像仓库；ACR支持将构建镜像的操作转移到云端，在ACR中完成

![image-20210207103403703](D:\学习资料\笔记\k8s\k8s图\image-20210207103403703.png)

针对私有镜像仓库，在拉去镜像的过程中，需要用户名密码验证身份。通常的做法是创建一个Secret，然后在YML文件中使用此Secret：

![image-20210207103455819](D:\学习资料\笔记\k8s\k8s图\image-20210207103455819.png)

AKS与ACR的结合比较方便，通过一条命令可以将AKS集群与ACR连在一起，后续运行在AKS中的Pod，在镜像拉取过程中无需在YML文件中添加imagePullSecrets字段。AKS与ACR连接的命令如下：

```sh
az aks update -n $AKSNAME -g $AKSRG --attach-acr $ACRNAME
```

AKS集群的创建过程会需要选择一些配置，建议网络的选择使用Azure高级网络插件，即Azure CNI，而非基于kubenet的基础Kubernetes网络。一方面在有些项目中，需要与同一VNET的其他VM或PaaS进行通信，在同一个网络平面会使连接更加简单；另一方面，针对一些高级网络功能，也只有Azure CNI能够使用。

此外在规划每个节点上Pod的数量时需要注意，默认一个节点的Pod最大值为30，可以通过CLI在创建时调整此值。建议不要调整过小，考虑到每个节点上会存在一些系统Pod，规划过小会导致一个节点运行不了几个Pod。

![image-20210207103754061](D:\学习资料\笔记\k8s\k8s图\image-20210207103754061.png)



在Kubernetes中，Pod的创建销毁是非常随意的，因此不会直接通过Pod发布服务，一般会通过Kubernetes中的资源Service完成。在AKS中创建Service时，可配置使用Internal Load Balancer  & Public Load Balancer。建议使用Internal  Load Balancer，对外发布的服务在Service前面，使用Ingress进行服务的对外发布。对于Ingress Controller，比较流行的是NGINX。NGINX在AKS中可用，通过Helm可以安装NGINX作为Ingress Controller。但毕竟非业务应用部分多安装一个服务，就多了一份运维的复杂度，在Azure平台上，如果使用PaaS服务替换NGINX作为Ingress Controller，Application Gateway v2是一个比较好的选择。



**在AKS集群内部，有两个地方需要根据实际情况处理：**

**Node Pool**

我们会看到在创建集群时，需要选择机器型号以及操作系统，会根据用户选择创建一个Node Pool，且Node Pool中的机器型号无法更改，只能更改机器数量。建议在使用中，根据实际情况规划Node Pool，一个集群中可包含多个Node Pool，例如根据工作负载的不同，分为Linux Node Pool、Windows Node Pool、GPU Node Pool等。

这其中有个特别的选项为虚拟节点，即利用Virtual Kubelet + Azure  Container Instance实现的一个逻辑节点，无CPU和内存限制，根据实际需要动态创建删除ACI。这对于基于事件或批量的任务处理更有效。建议创建集群时打开，根据工作负载的内容，长期运行的仍然部署在Node Pool中，任务型部署在虚拟节点中。

![image-20210207103840195](D:\学习资料\笔记\k8s\k8s图\image-20210207103840195.png)

针对不同的Node Pool或虚拟节点的选择，是在部署Kubernetes资源时针对Pod或Deployment的YML文件，添加限制来实现，如：

![image-20210207104036418](D:\学习资料\笔记\k8s\k8s图\image-20210207104036418.png)



#### 3. AKS的运维管理

毕竟AKS是一款PaaS服务，相比于IaaS中的虚机什么都要自己来，AKS作为托管服务在集群管理运维方面还是提供了蛮多的方便。

![image-20210207104216586](D:\学习资料\笔记\k8s\k8s图\image-20210207104216586.png)

首先在功能性上，上文提到过AKS是Follow Kubernetes Upstream的代码及功能变更，并未对核心代码做任何更改及功能限制，因此Kubernetes中支持的功能，在AKS中的用法相同，可以使用kubectl来完成。用户在使用AKS时，可能会关心Kubernetes的版本支持程度及更新频率。

Kubernetes的版本命名规则为：[major].[minor].[patch]，目前版本为v1.20.0，每三个月会更新一次版本，并不定时的更新补丁。AKS会紧跟社区步伐支持次新版本，并维持三个大版本及每个大版本中的四个小版本，同时Kubernetes社区发布一个版本之后，AKS一般会在一个月之后发出此版本的预览。

```sh
# AKS 支持的版本可以通过如下命令查询
az aks get-versions -l southeastasia -o table
```

![image-20210207104443431](D:\学习资料\笔记\k8s\k8s图\image-20210207104443431.png)

![image-20210207104451316](D:\学习资料\笔记\k8s\k8s图\image-20210207104451316.png)

升级AKS集群需要注意的是：建议检查支持升级的版本，且制定好维护窗口。AKS无法跨版本升级，即目前如果是1.17，要先升级到1.18，再升级到1.19。因为涉及到节点升级过程中节点上工作负载的移动，一方面请注意节点池中的Node不要是1个，尽可能是单数，如3、5；同时部署服务的Replica个数尽可能大于1，确保在Pod移动过程中服务仍然可用。

另一个用户关心的话题就是AKS集群基础设施的安全性。针对Kubernetes Control Plane，安全补丁及OS补丁对用户来说是透明的；对于Work Node，AKS提供了安全优化过的虚拟机镜像，且不支持用户自行选择。AKS会对VM进行Daily的安全及OS补丁，有些更新需要重启机器，AKS并不会主动重启机器，用户可以通过kured来完成检查目前环境中需要重启的节点，并按照步骤进行重启。

![image-20210207104644935](D:\学习资料\笔记\k8s\k8s图\image-20210207104644935.png)

AKS针对集群的管理提供了一些便捷服务，如下：

![image-20210207104728944](D:\学习资料\笔记\k8s\k8s图\image-20210207104728944.png)

以Node Auto-repair为例，在AKS中，Work Node会有自己的状态，正常情况下其状态都是Ready，这样集群中的Scheduler才会向此Node安排Pod，集群才可以针对这个Node进行管理。

![image-20210207104833923](D:\学习资料\笔记\k8s\k8s图\image-20210207104833923.png)

当Node状态变为Not Ready或Unknown时，就说明此Node出了一些问题。那针对Node Auto-repair，AKS集群会不断轮询Node的健康状况，当10分钟没有回应或状态不Ready超过10分钟，AKS会自动采取动作对此Node进行修复，动作包括：重启此节点 & 如果不成功，Reimage此节点 & 如果仍然不成功，创建一个新的节点。

Azure  Advisor同样适用于AKS集群，能够从性能、高可用、安全方面分析AKS集群的配置及使用情况，并提供可参考的建议。



**针对实验中提到的集群，我们可以查看一下目前有哪些建议：**

![image-20210207104953060](D:\学习资料\笔记\k8s\k8s图\image-20210207104953060.png)

如图所示：现有环境还是有很多值得改进的空间，点击进入每一条建议，都会有相对应的循序渐进的整改方式及涉及到的资源，部分整改方式提供了一键执行的操作：

![image-20210207105015407](D:\学习资料\笔记\k8s\k8s图\image-20210207105015407.png)

除了建议，AKS集群还提供了相关诊断服务，大家可以理解为一个脚本或检查列表，会提供内置选项，当我们在使用集群时出现问题，可能网络连接不通，节点挂了等，但不知如何下手，可以利用AKS诊断服务辅助进行检查：

![image-20210207105037236](D:\学习资料\笔记\k8s\k8s图\image-20210207105037236.png)

在此Demo中，点击Node Health可以看到，经过一段时间运行，就会出来一个检查报告，这个检查报告能够帮助我们评估现有集群的各项指标，并辅助我们完成修复集群工作。

![image-20210207105418081](D:\学习资料\笔记\k8s\k8s图\image-20210207105418081.png)

![image-20210207105427774](D:\学习资料\笔记\k8s\k8s图\image-20210207105427774.png)



**对于集群运维管理人员，能够实时了解AKS集群中服务的运行状况是非常必要的，这是衡量业务应用稳定的前提。**

![image-20210207105447310](D:\学习资料\笔记\k8s\k8s图\image-20210207105447310.png)

众所周知，Azure Monitor是用来集中提供云平台监控的统一入口，监控信息的查看，告警信息，日志信息都可以通过Azure Monitor来获取。Azure Monitor同样针对一些服务提供了监控解决方案，就包括AKS，即Azure Monitor for  Container：

![image-20210207105515724](D:\学习资料\笔记\k8s\k8s图\image-20210207105515724.png)



在这里，我们能够了解集群的使用情况，查看每个Pod的实时日志以及部署的相关信息。除此之外，Kubernetes的社区中，监控解决方案大部分都是围绕着Promethues进行构建的，Promethues中的监控数据和监控力度与平台提供的可能会有一些差别，但可以将Prometheus的监控数据通过OMS Agent的方式导入Azure Monitor中一并查看管理。

![image-20210207105537846](D:\学习资料\笔记\k8s\k8s图\image-20210207105537846.png)



最后需要注意的是：在创建集群后，可根据实际的需求配置诊断日志。因为很多用户需要的Control Plane日志、Kubernetes Audit Log以及集群的一些Advanced Metrics，默认并未启用，需要通过诊断日志启用，并存储在Log Analytics中。

![image-20210207105615056](D:\学习资料\笔记\k8s\k8s图\image-20210207105615056.png)



当然，并不是每个用户的集群中运行的应用都是自己开发的。当在使用AKS的公司有开发团队，或需要对AKS中部署的服务进行快速迭代部署，AKS也同样提供了端到端工具的选择，利用无论是Azure Pipelines或Github Action，都可以实现用户的自动化部署发布流程，再结合Azure Monitor、Azure Policy等服务，实现DevSecOps中持续集成 & 持续发布 & 持续监控等多个环节。

![image-20210207105654165](D:\学习资料\笔记\k8s\k8s图\image-20210207105654165.png)



#### 4. AKS中应用服务的可用性

当我们将服务部署在Kubernetes集群中，最关注的就是应用服务的可用性及集群的扩展性。在IaaS时代，我们部署应用会使用集群的方式，即利用冗余的方式，通过部署多个虚机资源来实现应用的高可用，避免单点故障。这个概念在Kubernetes中同样适用，只不过在Kubernetes这个部分的实现是通过Service & Deployment来实现，通过Deployment中的Replica数量，控制Pod的数量实现冗余，并通过Affinity设置Pod的分布，避免同一个服务两个Pod部署在同一个Node，如果Node出现问题服务还是存在风险。

![image-20210207105802710](D:\学习资料\笔记\k8s\k8s图\image-20210207105802710.png)

另外一个Kubernetes的特性就是其自愈能力，说的有点夸大，Kubernetes会监测环境中的Pod状态，确保其数量及运行状态与YML文件定义的一致。当Pod被人为删掉，系统会自动创建一个新的Pod，并对接相关资源，这也在另一方面确保了服务的可用性。



**除了Kubernetes自身的特性确保应用的可用性之外，AKS同样提供了一些能力来帮助提升应用的可用性。**

![image-20210207105941006](D:\学习资料\笔记\k8s\k8s图\image-20210207105941006.png)

可用性的另一个表现是当应用的访问量突然上来时，集群的自动扩展能力。在这一方面，针对运行于Node Pool中的应用，主要是利用Cluster Autoscaler +  Horizontal Pod Autoscaler来实现的。Horizontal  Pod Autoscaler主要扩展Pod的数量，但Pod的创建前提是环境中得有可用的Node供它调度，所以需要Cluster Autoscaler来扩展Node的数量。

![image-20210207110020443](D:\学习资料\笔记\k8s\k8s图\image-20210207110020443.png)

![image-20210207110006826](D:\学习资料\笔记\k8s\k8s图\image-20210207110006826.png)

这里的任务是针对现有的集群及服务，确保Pod数量大于1，并开启Cluster Autoscaler &  Horizontal Pod Autoscaler。

针对部署在Virtual Node中的应用，主要利用Virtual Node的特性，可结合Horizontal Pod Autoscaler  & KEDA按需扩展所需资源。个人认为，针对Virtual Node，在应用类别上可能更偏重于任务型或基于事件响应的服务，不需要长期运行。

![image-20210207110140261](D:\学习资料\笔记\k8s\k8s图\image-20210207110140261.png)

#### 5. AKS中的安全

相信大家都很好奇，针对AKS，各方面的安全能力利用了Azure云上的哪些服务。在这一部分我们看到，架构图会有一些更新：

![image-20210207110218302](D:\学习资料\笔记\k8s\k8s图\image-20210207110218302.png)



从云端治理的角度来看，AKS的安全第一步是利用Azure Policy规范AKS集群的使用。在开源社区，同样存在着类似的需求，实现方式是通过OPA Gatekeeper来实现的。Azure Policy对于AKS集群的治理是依托于OPA Gatekeeper来完成的。在Azure Policy中，内置了很多关于Kubernetes的Policy。

![image-20210207112824685](D:\学习资料\笔记\k8s\k8s图\image-20210207112824685.png)

![image-20210207112857826](D:\学习资料\笔记\k8s\k8s图\image-20210207112857826.png)



默认AKS集群中，Azure Policy的插件是关闭的，可通过一条命令启用：

```sh
az aks enable-addons --addons azure-policy --name zj-blog-aks01 --resource-group zj-blog
```

Azure  AD RBAC与Kubernetes RBAC进行整合已经并不是一个新鲜的话题，从AKS流行开始，这个功能就在支持。除此之外，现在可以利用AKS集群中AKS-managed Azure Active  Directory integration功能，更好地将Azure  AD甚至Azure Managed Identity与Kubernetes做到更好的集成。开启这个功能非常的方便，只需要在集群中点击一下即可：

![image-20210207113002990](D:\学习资料\笔记\k8s\k8s图\image-20210207113002990.png)



这其实也容易理解，在一个集群中，当人多手杂的时候，我们可以利用Azure AD用户账号限制开发人员访问的Namespace以及运维人员访问的Namespace。Azure AD在这里面起到两个作用，就是身份验证，在操作Kubernetes集群前，利用Azure AD证明你是你；另一个功能是授权验证，利用RBAC的概念设置，限制用户能够访问到的资源。设置好后，当我们再去运行Kubernetes命令时，会提示需要AAD验证。

![image-20210207114526433](D:\学习资料\笔记\k8s\k8s图\image-20210207114526433.png)

![image-20210207114545751](D:\学习资料\笔记\k8s\k8s图\image-20210207114545751.png)

AKS集群在网络连接性上提供了几种选择。首先对外发布的服务，通过Ingress Controller进行控制，Ingress Controller利用Application Gateway v2实现，并开启Web Application Firewall功能，实现应用访问的安全。

AKS提供了针对访问集群IP的控制，对于想要连接到Kubernetes API-Server的IP，能够进行限制：

![image-20210207115500832](D:\学习资料\笔记\k8s\k8s图\image-20210207115500832.png)

因此对于集群的管理，尤其是安全性要求比较高的一些用户，一个比较好的做法是限制API Server的访问，只允许通过本地内网IP或云上内网IP进行访问，并通过Azure Bastion充当跳板机，对集群进行访问管理。因为在整个Kubernetes集群中，API Server的中邀请毋庸置疑，所有的操作，所有的组件都是通过API Server进行通信的，因为管理API Server的访问也是一个需要考虑的事情。

![image-20210207115537506](D:\学习资料\笔记\k8s\k8s图\image-20210207115537506.png)

对于专有集群要求的用户，AKS能够通过Private Link直接将AKS变为专有集群，API Server将只提供内网访问地址，用户可以通过Azure Bastion实现集群的连接管理，本地需要与集群进行连接交互时，可以使用VPN或专线方式。

在Kubernetes集群中，不可避免会涉及到的一个话题就是密码的存储。在我们这个案例中，连接Cosmos DB需要用到连接字符串，这里会涉及到连接Cosmos DB的验证信息，这个时候存在哪里是个问题。肯定不可能放在明文存在YML里，Kubernetes提供了一个资源叫Secret，可用来存储集群中所有凭据信息，存储的信息将会保存在ETCD中，以Base64加密的方式保存。

当然，在云平台上都会有密钥存储的服务，如Azure Key Vault，能够与Secret对接，最终使应用需要的凭据存储在Azure Key Vault中。且如今，平台上的很多PaaS服务都能与Managed Identity做集成，代替以往的Service Principal的方式，确保密码不会显示，更加安全。

![image-20210207115717485](D:\学习资料\笔记\k8s\k8s图\image-20210207115717485.png)

同时对于Azure上的部分PaaS服务，如Azure SQL，借助Managed Service Identity，Pod能够直接与Azure服务进行通信，进一步减少了配置难度。

![image-20210207115805091](D:\学习资料\笔记\k8s\k8s图\image-20210207115805091.png)

最后还有Azure Security Center。大家都知道，Azure Security Center是整个Azure云平台安全监控的统一入口，能够针对多个工作负载进行安全评估，风险提示等安全功能。在Azure Security Center覆盖的工作负载中，就包含AKS。利用Security Center，我们能知道AKS集群目前做的好不好，有哪些不到位的地方，可以按照紧急程度以及相关的建议进行更新。

![image-20210207115857660](D:\学习资料\笔记\k8s\k8s图\image-20210207115857660.png)

![image-20210207115922526](D:\学习资料\笔记\k8s\k8s图\image-20210207115922526.png)





## 云原生时代的 YAML 教程



YAML 是 "YAML Ain't a Markup Language" 的缩写，是一种可读性高的数据序列化语言，常用于配置管理中。在云原生时代，很多流行的开源项目、云平台等都是 YAML 格式表达的，比如 Kubernetes 中的资源对象、Ansible/Terraform 的配置文件以及流行 CI/CD 平台的配置文件等等。



### 基本格式

首先需要理解的是 YAML 主要是面向数据而非表达能力，所以 YAML 本身的语法非常简洁，其最基本的语法规则为：

- 大小写敏感
- 使用空格缩进表示层级关系（不可以使用 TAB）
- 以 # 表示注释，行内注释 # 前面必须要有空格
- 基本数据类型包括 Null、布尔、字符串、整数、浮点数、日期和时间等
- 基本数据结构包括字典、列表以及纯量（即单个基本类型的值），其他复杂数据结构都是通过这些基本数据结构组合而成
- 单个文件包括多个 YAML 数据结构时使用 `---` 分割



### 字典

字典有两种表达方式，两种方式是对等的，一般方式二用的多一些：

```yaml
# 方式一（行内表达法）
foo: { thing1: huey, thing2: louie, thing3: dewey }

# 方式二
foo:
  thing1: huey
  thing2: louie
  thing3: dewey
```



### 列表

列表也有两种表达方式，一般方式一用的多一些：

```yaml
# 方式一
files:
- foo.txt
- bar.txt
- baz.txt

# 方式二（行内表达法）
lists: [foo.txt, bar.txt, baz.txt]
```



### 字符串

YAML 中默认字符串不需要添加任何引号，但在容易导致混淆的地方则是需要添加引号的。比如

- 字符串格式的数字必须加上引号，比如 "20"
- 字符串格式的布尔值必须加上引号，比如 "true"

YAML 也支持多行字符串，可以使用 `>`（折叠换行） 或者 `|`（保留换行符）：

比如

```yaml
# 折叠换行符，即等同于 "bar : this is not a normal string it spans more than one line see?"
bar: >
this is not a normal string it
spans more than
one line
see?

# 保留换行符
bar: |
this is not a normal string it
spans more than
one line
see?
```



### 片段和引用（Snippet）

YAML 也支持片段和引用，这些构造复杂数据类型时非常有用。你可以用 `&` 来定义一个片段，随后使用 `*` 来引用这个片段。

![image-20210303113624116](D:\学习资料\笔记\k8s\k8s图\image-20210303113624116.png)



### Kubernetes 资源对象

Kubernetes 资源对象格式可以参考其官方 API 文档 https://kubernetes.io/docs/reference/kubernetes-api/。需要注意的是，每个资源对象在定义的时候必须包含以下的字段：

- apiVersion - 创建该对象所使用的 Kubernetes API 的版本
- kind - 想要创建的对象的类别
- metadata - 帮助唯一性标识对象的一些数据，包括一个 name 字符串、UID 和可选的 namespace

通常，你也需要提供对象的 spec 字段。对象 spec 的详细格式对每个 Kubernetes 对象来说是不同的，包含了特定于该对象的嵌套字段。比如，一个 Nginx Pod 的定义如下所示：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-6799fc88d8-tpx29
  namespace: default
  labels:
    app: nginx
spec:
  containers:
  - image: nginx
    imagePullPolicy: Always
    name: nginx
  restartPolicy: Always
```



### 辅助工具

最后再推荐几个常用的 YAML 工具，包括 yamllint、yq、以及 kube-score。



#### yamllit

yamllint 是一个用来检查 YAML 语法的工具，你可以通过 `pip install --user yamllint` 命令来安装该工具。

比如，对上面的 Nginx Pod 运行 yamllint 会得到如下的警告和错误：

```sh
$ yamllint nginx.yaml
nginx.yaml
  1:1       warning  missing document start "---"  (document-start)
  10:3      error    wrong indentation: expected 4 but found 2  (indentation)
```

根据这两个警告和错误，可以把其修改成如下的格式：

```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx-6799fc88d8-tpx29
  namespace: default
  labels:
    app: nginx
spec:
  containers:
    - image: nginx
      imagePullPolicy: Always
      name: nginx
  restartPolicy: Always
```



#### yq

yq 是一个 YAML 数据处理以及高亮显示的工具，你可以通过 `brew install yq` 来安装该工具（类似于 JSON 数据处理的 `jq` 工具）。

比如，你可以高亮显示上面的 Nginx Pod YAML：

![image-20210303113744527](D:\学习资料\笔记\k8s\k8s图\image-20210303113744527.png)

或者提取上述 YAML 的部分内容：

```sh
$ yq eval '.metadata.name' nginx.yaml
nginx-6799fc88d8-tpx29
```

或者修改 YAML 中 `metadata.name` 字段：

```yaml
# yq eval '.metadata.name = "nginx"' nginx.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: default
  labels:
    app: nginx
spec:
  containers:
    - image: nginx
      imagePullPolicy: Always
      name: nginx
  restartPolicy: Always
```



#### kube-score

kube-score 是一个 Kubernetes YAML 静态分析工具，用来检查 Kubernetes 资源对象的配置是否遵循了最佳实践。你可以通过 `kubectl krew install score` 来安装该工具。

比如，还是上述的 Nginx Pod，运行 `kubectl score nginx.yaml` 可以得到如下的错误：

![image-20210303113848536](D:\学习资料\笔记\k8s\k8s图\image-20210303113848536.png)

根据这些错误，可以发现容器资源、镜像标签、网络策略以及容器安全上下文等四个配置没有遵循最佳实践。



### 参考资料

更多 YAML 的使用细节可以参考如下的资料：

- YAML 语言规范 https://yaml.org/spec/1.2/spec.html
- Kubernetes 资源对象 https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/





## K8s 创建 pod 时，背后到底发生了什么？



全文大纲：

- K8s 组件启动过程
- kubectl（命令行客户端）
- kube-apiserver
- 写入 etcd
- Initializers
- Control loops（控制循环）
- Kubelet

---

本文试图回答以下问题：**敲下 `kubectl run nginx --image=nginx --replicas=3` 命令后**，**K8s 中发生了哪些事情？**

要弄清楚这个问题，我们需要：

1. 了解 K8s 几个核心组件的启动过程，它们分别做了哪些事情，以及
2. 从客户端发起请求到 pod ready 的整个过程。



### 0 K8s 组件启动过程

首先看几个核心组件的启动过程分别做了哪些事情。



#### 0.1 kube-apiserver 启动

**调用栈**

创建命令行（`kube-apiserver`）入口：

```go
main                                         // cmd/kube-apiserver/apiserver.go
 |-cmd := app.NewAPIServerCommand()          // cmd/kube-apiserver/app/server.go
 |  |-RunE := func() {
 |      Complete()
 |        |-ApplyAuthorization(s.Authorization)
 |        |-if TLS:
 |            ServiceAccounts.KeyFiles = []string{CertKey.KeyFile}
 |      Validate()
 |      Run(completedOptions, handlers) // 核心逻辑
 |    }
 |-cmd.Execute()
```

`kube-apiserver` 启动后，会执行到其中的 `Run()` 方法：

```
main                                         // cmd/kube-apiserver/apiserver.go
 |-cmd := app.NewAPIServerCommand()          // cmd/kube-apiserver/app/server.go
 |  |-RunE := func() {
 |      Complete()
 |        |-ApplyAuthorization(s.Authorization)
 |        |-if TLS:
 |            ServiceAccounts.KeyFiles = []string{CertKey.KeyFile}
 |      Validate()
 |      Run(completedOptions, handlers) // 核心逻辑
 |    }
 |-cmd.Execute()
```

`kube-apiserver` 启动后，会执行到其中的 `Run()` 方法：

```go
Run()          // cmd/kube-apiserver/app/server.go
 |-server = CreateServerChain()
 |           |-CreateKubeAPIServerConfig()
 |           |   |-buildGenericConfig
 |           |   |   |-genericapiserver.NewConfig()     // staging/src/k8s.io/apiserver/pkg/server/config.go
 |           |   |   |  |-return &Config{
 |           |   |   |       Serializer:             codecs,
 |           |   |   |       BuildHandlerChainFunc:  DefaultBuildHandlerChain, // 注册 handler
 |           |   |   |    }
 |           |   |   |
 |           |   |   |-OpenAPIConfig = DefaultOpenAPIConfig()  // OpenAPI schema
 |           |   |   |-kubeapiserver.NewStorageFactoryConfig() // etcd 相关配置
 |           |   |   |-APIResourceConfig = genericConfig.MergedResourceConfig
 |           |   |   |-storageFactoryConfig.Complete(s.Etcd)
 |           |   |   |-storageFactory = completedStorageFactoryConfig.New()
 |           |   |   |-s.Etcd.ApplyWithStorageFactoryTo(storageFactory, genericConfig)
 |           |   |   |-BuildAuthorizer(s, genericConfig.EgressSelector, versionedInformers)
 |           |   |   |-pluginInitializers, admissionPostStartHook = admissionConfig.New()
 |           |   |
 |           |   |-capabilities.Initialize
 |           |   |-controlplane.ServiceIPRange()
 |           |   |-config := &controlplane.Config{}
 |           |   |-AddPostStartHook("start-kube-apiserver-admission-initializer", admissionPostStartHook)
 |           |   |-ServiceAccountIssuerURL = s.Authentication.ServiceAccounts.Issuer
 |           |   |-ServiceAccountJWKSURI = s.Authentication.ServiceAccounts.JWKSURI
 |           |   |-ServiceAccountPublicKeys = pubKeys
 |           |
 |           |-createAPIExtensionsServer
 |           |-CreateKubeAPIServer
 |           |-createAggregatorServer    // cmd/kube-apiserver/app/aggregator.go
 |           |   |-aggregatorConfig.Complete().NewWithDelegate(delegateAPIServer)   // staging/src/k8s.io/kube-aggregator/pkg/apiserver/apiserver.go
 |           |   |  |-apiGroupInfo := NewRESTStorage()
 |           |   |  |-GenericAPIServer.InstallAPIGroup(&apiGroupInfo)
 |           |   |  |-InstallAPIGroups
 |           |   |  |-openAPIModels := s.getOpenAPIModels(APIGroupPrefix, apiGroupInfos...)
 |           |   |  |-for apiGroupInfo := range apiGroupInfos {
 |           |   |  |   s.installAPIResources(APIGroupPrefix, apiGroupInfo, openAPIModels)
 |           |   |  |   s.DiscoveryGroupManager.AddGroup(apiGroup)
 |           |   |  |   s.Handler.GoRestfulContainer.Add(discovery.NewAPIGroupHandler(s.Serializer, apiGroup).WebService())
 |           |   |  |
 |           |   |  |-GenericAPIServer.Handler.NonGoRestfulMux.Handle("/apis", apisHandler)
 |           |   |  |-GenericAPIServer.Handler.NonGoRestfulMux.UnlistedHandle("/apis/", apisHandler)
 |           |   |  |-
 |           |   |-
 |-prepared = server.PrepareRun()     // staging/src/k8s.io/kube-aggregator/pkg/apiserver/apiserver.go
 |            |-GenericAPIServer.AddPostStartHookOrDie
 |            |-GenericAPIServer.PrepareRun
 |            |  |-routes.OpenAPI{}.Install()
 |            |     |-registerResourceHandlers // staging/src/k8s.io/apiserver/pkg/endpoints/installer.go
 |            |         |-POST: XX
 |            |         |-GET: XX
 |            |
 |            |-openapiaggregator.BuildAndRegisterAggregator()
 |            |-openapiaggregator.NewAggregationController()
 |            |-preparedAPIAggregator{}
 |-prepared.Run() // staging/src/k8s.io/kube-aggregator/pkg/apiserver/apiserver.go
    |-s.runnable.Run()
```



**一些重要步骤**

1. **创建 server chain**。Server aggregation（聚合）是一种支持多 apiserver 的方式，其中 包括了一个 **generic apiserver**[3]，作为默认实现。
2. **生成 OpenAPI schema**，保存到 apiserver 的 **Config.OpenAPIConfig 字段**[4]。
3. 遍历 schema 中的所有 API group，为每个 API group 配置一个 **storage provider**[5]， 这是一个通用 backend 存储抽象层。
4. 遍历每个 group 版本，为每个 HTTP route **配置 REST mappings**[6]。稍后处理请求时，就能将 requests 匹配到合适的 handler。



#### 0.2 controller-manager 启动

**调用栈**

```go
NewDeploymentController
NewReplicaSetController
```



#### 0.3 kubelet 启动

**调用栈**

```go
main                                                                            // cmd/kubelet/kubelet.go
 |-NewKubeletCommand                                                            // cmd/kubelet/app/server.go
   |-Run                                                                        // cmd/kubelet/app/server.go
      |-initForOS                                                               // cmd/kubelet/app/server.go
      |-run                                                                     // cmd/kubelet/app/server.go
        |-initConfigz                                                           // cmd/kubelet/app/server.go
        |-InitCloudProvider
        |-NewContainerManager
        |-ApplyOOMScoreAdj
        |-PreInitRuntimeService
        |-RunKubelet                                                            // cmd/kubelet/app/server.go
        | |-k = createAndInitKubelet                                            // cmd/kubelet/app/server.go
        | |  |-NewMainKubelet
        | |  |  |-watch k8s Service
        | |  |  |-watch k8s Node
        | |  |  |-klet := &Kubelet{}
        | |  |  |-init klet fields
        | |  |
        | |  |-k.BirthCry()
        | |  |-k.StartGarbageCollection()
        | |
        | |-startKubelet(k)                                                     // cmd/kubelet/app/server.go
        |    |-go k.Run()                                                       // -> pkg/kubelet/kubelet.go
        |    |  |-go cloudResourceSyncManager.Run()
        |    |  |-initializeModules
        |    |  |-go volumeManager.Run()
        |    |  |-go nodeLeaseController.Run()
        |    |  |-initNetworkUtil() // setup iptables
        |    |  |-go Until(PerformPodKillingWork, 1*time.Second, neverStop)
        |    |  |-statusManager.Start()
        |    |  |-runtimeClassManager.Start
        |    |  |-pleg.Start()
        |    |  |-syncLoop(updates, kl)                                         // pkg/kubelet/kubelet.go
        |    |
        |    |-k.ListenAndServe
        |
        |-go http.ListenAndServe(healthz)
```



#### 0.4 小结

以上核心组件启动完成后，就可以从命令行发起请求创建 pod 了。



### 1 kubectl（命令行客户端）

#### 1.0 调用栈概览

```go
NewKubectlCommand                                    // staging/src/k8s.io/kubectl/pkg/cmd/cmd.go
 |-matchVersionConfig = NewMatchVersionFlags()
 |-f = cmdutil.NewFactory(matchVersionConfig)
 |      |-clientGetter = matchVersionConfig
 |-NewCmdRun(f)                                      // staging/src/k8s.io/kubectl/pkg/cmd/run/run.go
 |  |-Complete                                       // staging/src/k8s.io/kubectl/pkg/cmd/run/run.go
 |  |-Run(f)                                         // staging/src/k8s.io/kubectl/pkg/cmd/run/run.go
 |    |-validate parameters
 |    |-generators = GeneratorFn("run")
 |    |-runObj = createGeneratedObject(generators)   // staging/src/k8s.io/kubectl/pkg/cmd/run/run.go
 |    |           |-obj = generator.Generate()       // -> staging/src/k8s.io/kubectl/pkg/generate/versioned/run.go
 |    |           |        |-get pod params
 |    |           |        |-pod = v1.Pod{params}
 |    |           |        |-return &pod
 |    |           |-mapper = f.ToRESTMapper()        // -> staging/src/k8s.io/cli-runtime/pkg/genericclioptions/config_flags.go
 |    |           |  |-f.clientGetter.ToRESTMapper() // -> staging/src/k8s.io/kubectl/pkg/cmd/util/factory_client_access.go
 |    |           |     |-f.Delegate.ToRESTMapper()  // -> staging/src/k8s.io/kubectl/pkg/cmd/util/kubectl_match_version.go
 |    |           |        |-ToRESTMapper            // -> staging/src/k8s.io/cli-runtime/pkg/resource/builder.go
 |    |           |        |-delegate()              //    staging/src/k8s.io/cli-runtime/pkg/resource/builder.go
 |    |           |--actualObj = resource.NewHelper(mapping).XX.Create(obj)
 |    |-PrintObj(runObj.Object)
 |
 |-NewCmdEdit(f)      // kubectl edit   命令
 |-NewCmdScale(f)     // kubectl scale  命令
 |-NewCmdCordon(f)    // kubectl cordon 命令
 |-NewCmdUncordon(f)
 |-NewCmdDrain(f)
 |-NewCmdTaint(f)
 |-NewCmdExecute(f)
 |-...
```



#### 1.1 参数验证（validation）和资源对象生成器（generator）

**参数验证**

敲下 `kubectl` 命令后，它首先会做一些客户端侧的验证。如果命令行参数有问题，例如，**镜像名为空或格式不对**[7]， 这里会直接报错，从而避免了将明显错误的请求发给 kube-apiserver，减轻了后者的压力。

此外，kubectl 还会检查其他一些配置，例如

- 是否需要记录（record）这条命令（用于 rollout 或审计）
- 是否只是测试执行（`--dry-run`）



**创建 HTTP 请求**

所有**查询或修改 K8s 资源的操作**都需要与 kube-apiserver 交互，后者会进一步和 etcd 通信。

因此，验证通过之后，kubectl 接下来会创建发送给 kube-apiserver 的 HTTP 请求。



**Generators**

**创建 HTTP 请求用到了所谓的** **generator**[8]（**文档**[9]） ，它封装了资源的序列化（serialization）操作。例如，创建 pod 时用到的 generator 是 **BasicPod**[10]：

```go
// staging/src/k8s.io/kubectl/pkg/generate/versioned/run.go

type BasicPod struct{}

func (BasicPod) ParamNames() []generate.GeneratorParam {
    return []generate.GeneratorParam{
        {Name: "labels", Required: false},
        {Name: "name", Required: true},
        {Name: "image", Required: true},
        ...
    }
}
```

每个 generator 都实现了一个 `Generate()` 方法，用于生成一个该资源的运行时对象（runtime object）。对于 `BasicPod`，其**实现**[11]为：

```go
func (BasicPod) Generate(genericParams map[string]interface{}) (runtime.Object, error) {
    pod := v1.Pod{
        ObjectMeta: metav1.ObjectMeta{  // metadata 字段
            Name:        name,
            Labels:      labels,
            ...
        },
        Spec: v1.PodSpec{               // spec 字段
            ServiceAccountName: params["serviceaccount"],
            Containers: []v1.Container{
                {
                    Name:            name,
                    Image:           params["image"]
                },
            },
        },
    }

    return &pod, nil
}
```



#### 1.2 API group 和版本协商（version negotiation）

有了 runtime object 之后，kubectl 需要用合适的 API 将请求发送给 kube-apiserver。

**API Group**

K8s 用 API group 来管理 resource API。这是一种不同于 monolithic API（所有 API 扁平化）的 API 管理方式。

具体来说，**同一资源的不同版本的 API，会放到一个 group 里面**。例如 Deployment 资源的 API group 名为 `apps`，最新的版本是 `v1`。这也是为什么 我们在创建 Deployment 时，需要在 yaml 中指定 `apiVersion: apps/v1` 的原因。



**版本协商**

生成 runtime object 之后，kubectl 就开始**搜索合适的 API group 和版本**[12]：

```go
// staging/src/k8s.io/kubectl/pkg/cmd/run/run.go

    obj := generator.Generate(params) // 创建运行时对象
    mapper := f.ToRESTMapper()        // 寻找适合这个资源（对象）的 API group
```

然后**创建一个正确版本的客户端（versioned client）**[13]，

```go
// staging/src/k8s.io/kubectl/pkg/cmd/run/run.go

    gvks, _ := scheme.Scheme.ObjectKinds(obj)
    mapping := mapper.RESTMapping(gvks[0].GroupKind(), gvks[0].Version)
```

这个客户端能感知资源的 REST 语义。

以上过程称为版本协商。在实现上，kubectl 会**扫描 kube-apiserver 的 `/apis` 路径**（OpenAPI 格式的 schema 文档），获取所有的 API groups。

出于性能考虑，kubectl 会**缓存这份 OpenAPI schema**[14]， 路径是 `~/.kube/cache/discovery`。**想查看这个 API discovery 过程，可以删除这个文件**， 然后随便执行一条 kubectl 命令，并指定足够大的日志级别（例如 `kubectl get ds -v 10`）。



**发送 HTTP 请求**

现在有了 runtime object，也找到了正确的 API，因此接下来就是 将请求真正**发送出去**[15]：

```go
// staging/src/k8s.io/kubectl/pkg/cmd/cmd.go

        actualObj = resource.
            NewHelper(client, mapping).
            DryRun(o.DryRunStrategy == cmdutil.DryRunServer).
            WithFieldManager(o.fieldManager).
            Create(o.Namespace, false, obj)
```

发送成功后，会以恰当的格式打印返回的消息。



#### 1.3 客户端认证（client auth）

前面其实有意漏掉了一步：客户端认证。它发生在发送 HTTP 请求之前。

**用户凭证（credentials）一般都放在 kubeconfig 文件中，但这个文件可以位于多个位置**， 优先级从高到低：

- 命令行 `--kubeconfig <file>`
- 环境变量 `$KUBECONFIG`
- 某些**预定义的路径**[16]，例如 `~/.kube`。

**这个文件中存储了集群、用户认证等信息**，如下面所示：

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /etc/kubernetes/pki/ca.crt
    server: https://192.168.2.100:443
  name: k8s-cluster-1
contexts:
- context:
    cluster: k8s-cluster-1
    user: default-user
  name: default-context
current-context: default-context
kind: Config
preferences: {}
users:
- name: default-user
  user:
    client-certificate: /etc/kubernetes/pki/admin.crt
    client-key: /etc/kubernetes/pki/admin.key
```

有了这些信息之后，客户端就可以组装 HTTP 请求的认证头了。支持的认证方式有几种：

- **X509 证书**：放到 **TLS**[17] 中发送；
- **Bearer token**：放到 HTTP `"Authorization"` 头中**发送**[18]；
- **用户名密码**：放到 HTTP basic auth **发送**[19]；
- **OpenID auth**：需要先由用户手动处理，将其转成一个 token，然后和 bearer token 类似发送。



### 2 kube-apiserver

请求从客户端发出后，便来到服务端，也就是 kube-apiserver。



#### 2.0 调用栈概览

```go
buildGenericConfig
  |-genericConfig = genericapiserver.NewConfig(legacyscheme.Codecs)  // cmd/kube-apiserver/app/server.go

NewConfig       // staging/src/k8s.io/apiserver/pkg/server/config.go
 |-return &Config{
      Serializer:             codecs,
      BuildHandlerChainFunc:  DefaultBuildHandlerChain,
   }                          /
                            /
                          /
                        /
DefaultBuildHandlerChain       // staging/src/k8s.io/apiserver/pkg/server/config.go
 |-handler := filterlatency.TrackCompleted(apiHandler)
 |-handler = genericapifilters.WithAuthorization(handler)
 |-handler = genericapifilters.WithAudit(handler)
 |-handler = genericapifilters.WithAuthentication(handler)
 |-return handler


WithAuthentication
 |-withAuthentication
    |-resp, ok := AuthenticateRequest(req)
    |  |-for h := range authHandler.Handlers {
    |      resp, ok := currAuthRequestHandler.AuthenticateRequest(req)
    |      if ok {
    |          return resp, ok, err
    |      }
    |    }
    |    return nil, false, utilerrors.NewAggregate(errlist)
    |
    |-audiencesAreAcceptable(apiAuds, resp.Audiences)
    |-req.Header.Del("Authorization")
    |-req = req.WithContext(WithUser(req.Context(), resp.User))
    |-return handler.ServeHTTP(w, req)
```



#### 2.1 认证（Authentication）

kube-apiserver 首先会对请求进行认证（authentication），以确保用户身份是合法的（verify that the requester is who they say they are）。

具体过程：启动时，检查所有的**命令行参数**[20]，组织成一个 authenticator list，例如，

- 如果指定了 `--client-ca-file`，就会将 x509 证书加到这个列表；
- 如果指定了 `--token-auth-file`，就会将 token 加到这个列表；

不同 anthenticator 做的事情有所不同：

- **x509 handler**[21] 验证该 HTTP 请求是用 TLS key 加密的，并且有 CA root 证书的签名。
- **bearer token handler**[22] 验证请求中带的 token（HTTP Authorization 头中），在 apiserver 的 auth file 中是存在的（`--token-auth-file`）。
- **basicauth handler**[23] 对 basic auth 信息进行校验。

**如果认证成功，就会将 `Authorization` 头从请求中删除**，然后在上下文中**加上用户信息**[24]。这使得后面的步骤（例如鉴权和 admission control）能用到这里已经识别出的用户身份信息。

```go
// staging/src/k8s.io/apiserver/pkg/endpoints/filters/authentication.go

// WithAuthentication creates an http handler that tries to authenticate the given request as a user, and then
// stores any such user found onto the provided context for the request.
// On success, "Authorization" header is removed from the request and handler
// is invoked to serve the request.
func WithAuthentication(handler http.Handler, auth authenticator.Request, failed http.Handler,
    apiAuds authenticator.Audiences) http.Handler {
    return withAuthentication(handler, auth, failed, apiAuds, recordAuthMetrics)
}

func withAuthentication(handler http.Handler, auth authenticator.Request, failed http.Handler,
    apiAuds authenticator.Audiences, metrics recordMetrics) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, req *http.Request) {
        resp, ok := auth.AuthenticateRequest(req) // 遍历所有 authenticator，任何一个成功就返回 OK
        if !ok {
            return failed.ServeHTTP(w, req)       // 所有认证方式都失败了
        }

        if !audiencesAreAcceptable(apiAuds, resp.Audiences) {
            fmt.Errorf("unable to match the audience: %v , accepted: %v", resp.Audiences, apiAuds)
            failed.ServeHTTP(w, req)
            return
        }

        req.Header.Del("Authorization") // 认证成功后，这个 header 就没有用了，可以删掉

        // 将用户信息添加到请求上下文中，供后面的步骤使用
        req = req.WithContext(WithUser(req.Context(), resp.User))
        handler.ServeHTTP(w, req)
    })
}
```

`AuthenticateRequest()` 实现：遍历所有 authenticator，任何一个成功就返回 OK，

```go
// staging/src/k8s.io/apiserver/pkg/authentication/request/union/union.go

func (authHandler *unionAuthRequestHandler) AuthenticateRequest(req) (*Response, bool) {
    for currAuthRequestHandler := range authHandler.Handlers {
        resp, ok := currAuthRequestHandler.AuthenticateRequest(req)
        if ok {
            return resp, ok, err
        }
    }

    return nil, false, utilerrors.NewAggregate(errlist)
}
```



#### 2.2 鉴权（Authorization）

**发送者身份（认证）是一个问题，但他是否有权限执行这个操作（鉴权），是另一个问题**。因此确认发送者身份之后，还需要进行鉴权。

鉴权的过程与认证非常相似，也是逐个匹配 authorizer 列表中的 authorizer：如果都失败了， 返回 `Forbidden` 并停止 **进一步处理**[25]。如果成功，就继续。

内置的 **几种 authorizer 类型**：

- **webhook**[26]：与其他服务交互，验证是否有权限。
- **ABAC**[27]：根据静态文件中规定的策略（policies）来进行鉴权。
- **RBAC**[28]：根据 role 进行鉴权，其中 role 是 k8s 管理员提前配置的。
- **Node**[29]：确保 node clients，例如 kubelet，只能访问本机内的资源。

要看它们的具体做了哪些事情，可以查看它们各自的 `Authorize()` 方法。



#### 2.3 Admission control

至此，认证和鉴权都通过了。但这还没结束，K8s 中的其它组件还需要对请求进行检查， 其中就包括 **admission controllers**[30]。

**与鉴权的区别**

- 鉴权（authorization）在前面，关注的是**用户是否有操作权限**，
- Admission controllers 在更后面，**对请求进行拦截和过滤，确保它们符合一些更广泛的集群规则和限制**， 是**将请求对象持久化到 etcd 之前的最后堡垒**。



**工作方式**

- 与认证和鉴权类似，也是遍历一个列表，
- 但有一点核心区别：**任何一个 controller 检查没通过，请求就会失败**。



**设计：可扩展**

- 每个 controller 作为一个 plugin 存放在 **plugin/pkg/admission 目录**[31],
- 设计时已经考虑，只需要实现很少的几个接口
- 但注意，**admission controller 最终会编译到 k8s 的二进制文件**（而非独立的 plugin binary）



**类型**

Admission controllers 通常按不同目的分类，包括：**资源管理、安全管理、默认值管 理、引用一致性**（referential consistency）等类型。

例如，下面是资源管理类的几个 controller：

- `InitialResources`：为容器设置默认的资源限制（基于过去的使用量）；
- `LimitRanger`：为容器的 requests and limits 设置默认值，或对特定资源设置上限（例如，内存默认 512MB，最高不超过 2GB）。
- `ResourceQuota`：资源配额。



### 3 写入 etcd

至此，K8s 已经完成对请求的验证，允许它进行接下来的处理。

kube-apiserver 将对请求进行反序列化，构造 runtime objects**（ kubectl generator 的反过程），并将它们**持久化到 etcd。下面详细 看这个过程。



#### 3.0 调用栈概览

对于本文创建 pod 的请求，相应的入口是**POST handler**[32]，它又会进一步将请求委托给一个创建具体资源的 handler。

```go
registerResourceHandlers // staging/src/k8s.io/apiserver/pkg/endpoints/installer.go
 |-case POST:
```

```go
// staging/src/k8s.io/apiserver/pkg/endpoints/installer.go

        switch () {
        case "POST": // Create a resource.
            var handler restful.RouteFunction
            if isNamedCreater {
                handler = restfulCreateNamedResource(namedCreater, reqScope, admit)
            } else {
                handler = restfulCreateResource(creater, reqScope, admit)
            }

            handler = metrics.InstrumentRouteFunc(action.Verb, group, version, resource, subresource, .., handler)
            article := GetArticleForNoun(kind, " ")
            doc := "create" + article + kind
            if isSubresource {
                doc = "create " + subresource + " of" + article + kind
            }

            route := ws.POST(action.Path).To(handler).
                Doc(doc).
                Operation("create"+namespaced+kind+strings.Title(subresource)+operationSuffix).
                Produces(append(storageMeta.ProducesMIMETypes(action.Verb), mediaTypes...)...).
                Returns(http.StatusOK, "OK", producedObject).
                Returns(http.StatusCreated, "Created", producedObject).
                Returns(http.StatusAccepted, "Accepted", producedObject).
                Reads(defaultVersionedObject).
                Writes(producedObject)

            AddObjectParams(ws, route, versionedCreateOptions)
            addParams(route, action.Params)
            routes = append(routes, route)
        }

        for route := range routes {
            route.Metadata(ROUTE_META_GVK, metav1.GroupVersionKind{
                Group:   reqScope.Kind.Group,
                Version: reqScope.Kind.Version,
                Kind:    reqScope.Kind.Kind,
            })
            route.Metadata(ROUTE_META_ACTION, strings.ToLower(action.Verb))
            ws.Route(route)
        }
```



#### 3.1 kube-apiserver 请求处理过程

从 apiserver 的请求处理函数开始：

```go
// staging/src/k8s.io/apiserver/pkg/server/handler.go

func (d director) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    path := req.URL.Path

    // check to see if our webservices want to claim this path
    for _, ws := range d.goRestfulContainer.RegisteredWebServices() {
        switch {
        case ws.RootPath() == "/apis":
            if path == "/apis" || path == "/apis/" {
                return d.goRestfulContainer.Dispatch(w, req)
            }

        case strings.HasPrefix(path, ws.RootPath()):
            if len(path) == len(ws.RootPath()) || path[len(ws.RootPath())] == '/' {
                return d.goRestfulContainer.Dispatch(w, req)
            }
        }
    }

    // if we didn't find a match, then we just skip gorestful altogether
    d.nonGoRestfulMux.ServeHTTP(w, req)
}
```

如果能匹配到请求（例如匹配到前面注册的路由），它将**分派给相应的 handler**[33]；否则，fall back 到**path-based handler**[34]（`GET /apis` 到达的就是这里）；

基于 path 的 handlers：

```go
// staging/src/k8s.io/apiserver/pkg/server/mux/pathrecorder.go

func (h *pathHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    if exactHandler, ok := h.pathToHandler[r.URL.Path]; ok {
        return exactHandler.ServeHTTP(w, r)
    }

    for prefixHandler := range h.prefixHandlers {
        if strings.HasPrefix(r.URL.Path, prefixHandler.prefix) {
            return prefixHandler.handler.ServeHTTP(w, r)
        }
    }

    h.notFoundHandler.ServeHTTP(w, r)
}
```

如果还是没有找到路由，就会 fallback 到 non-gorestful handler，最终可能是一个 not found handler。

对于我们的场景，会匹配到一条已经注册的、名为**`createHandler`**[35]为的路由。



#### 3.2 Create handler 处理过程

```go
// staging/src/k8s.io/apiserver/pkg/endpoints/handlers/create.go

func createHandler(r rest.NamedCreater, scope *RequestScope, admit Interface, includeName bool) http.HandlerFunc {
    return func(w http.ResponseWriter, req *http.Request) {
        namespace, name := scope.Namer.Name(req) // 获取资源的 namespace 和 name（etcd item key）
        s := negotiation.NegotiateInputSerializer(req, false, scope.Serializer)

        body := limitedReadBody(req, scope.MaxRequestBodyBytes)
        obj, gvk := decoder.Decode(body, &defaultGVK, original)

        admit = admission.WithAudit(admit, ae)

        requestFunc := func() (runtime.Object, error) {
            return r.Create(
                name,
                obj,
                rest.AdmissionToValidateObjectFunc(admit, admissionAttributes, scope),
            )
        }

        result := finishRequest(ctx, func() (runtime.Object, error) {
            if scope.FieldManager != nil {
                liveObj := scope.Creater.New(scope.Kind)
                obj = scope.FieldManager.UpdateNoErrors(liveObj, obj, managerOrUserAgent(options.FieldManager, req.UserAgent()))
                admit = fieldmanager.NewManagedFieldsValidatingAdmissionController(admit)
            }

            admit.(admission.MutationInterface)
            mutatingAdmission.Handles(admission.Create)
            mutatingAdmission.Admit(ctx, admissionAttributes, scope)

            return requestFunc()
        })

        code := http.StatusCreated
        status, ok := result.(*metav1.Status)
        transformResponseObject(ctx, scope, trace, req, w, code, outputMediaType, result)
    }
}
```

1. 首先解析 HTTP request，然后执行基本的验证，例如保证 JSON 与 versioned API resource 期望的是一致的；

2. 执行审计和最终 admission；

3. 将资源最终**写到 etcd**[36]， 这会进一步调用到 **storage provider**[37]。

   **etcd key 的格式一般是** `<namespace>/<name>`（例如，`default/nginx-0`），但这个也是可配置的。

4. 最后，storage provider 执行一次 `get` 操作，确保对象真的创建成功了。如果有额外的收尾任务（additional finalization），会执行 post-create handlers 和 decorators。

5. 返回 **生成的**[38]HTTP response。

以上过程可以看出，apiserver 做了大量的事情。

总结：至此我们的 pod 资源已经在 etcd 中了。但是，此时 `kubectl get pods -n <ns>` 还看不见它。



### 4 Initializers

**对象持久化到 etcd 之后，apiserver 并未将其置位对外可见，它也不会立即就被调度**， 而是要先等一些 **initializers**[39] 运行完成。

#### 4.1 Initializer

Initializer 是与特定资源类型（resource type）相关的 controller，

- 负责**在该资源对外可见之前对它们执行一些处理**，
- 如果一种资源类型没有注册任何 initializer，这个步骤就会跳过，**资源对外立即可见**。

这是一种非常强大的特性，使得我们能**执行一些通用的启动初始化（bootstrap）操作**。例如，

- 向 Pod 注入 sidecar、暴露 80 端口，或打上特定的 annotation。
- 向某个 namespace 内的所有 pod 注入一个存放了测试证书（test certificates）的 volume。
- 禁止创建长度小于 20 个字符的 Secret （例如密码）。



#### 4.2 InitializerConfiguration

可以用 `InitializerConfiguration` **声明对哪些资源类型（resource type）执行哪些 initializer**。

例如，要实现所有 pod 创建时都运行一个自定义的 initializer `custom-pod-initializer`， 可以用下面的 yaml：

```yaml
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: InitializerConfiguration
metadata:
  name: custom-pod-initializer
initializers:
  - name: podimage.example.com
    rules:
      - apiGroups:
          - ""
        apiVersions:
          - v1
        resources:
          - pods
```

创建以上配置（`kubectl create -f xx.yaml`）之后，K8s 会将`custom-pod-initializer` **追加到每个 pod 的 `metadata.initializers.pending` 字段**。

在此之前需要启动 initializer controller，它会

- 定期扫描是否有新 pod 创建；
- 当**检测到它的名字出现在 pod 的 pending 字段**时，就会执行它的处理逻辑；
- 执行完成之后，它会将自己的名字从 pending list 中移除。

pending list 中的 initializers，每次只有第一个 initializer 能执行。当**所有 initializer 执行完成，`pending` 字段为空**之后，就认为**这个对象已经完成初始化了**（considered initialized）。

细心的同学可能会有疑问：**前面说这个对象还没有对外可见，那用户空间的 initializer controller 又是如何能检测并操作这个对象的呢？**答案是：kube-apiserver 提供了一个 `?includeUninitialized` 查询参数，它会返回所有对象， 包括那些还未完成初始化的（uninitialized ones）。



### 5 Control loops（控制循环）

至此，对象已经在 etcd 中了，所有的初始化步骤也已经完成了。下一步是设置资源拓扑（resource topology）。例如，一个 Deployment 其实就是一组 ReplicaSet，而一个 ReplicaSet 就是一组 Pod。K8s 是如何根据一个 HTTP 请求创建出这个层级关系的呢？靠的是 **K8s 内置的控制器**（controllers）。

K8s 中大量使用 "controllers"，

- 一个 controller 就是一个异步脚本（an asynchronous script），
- 不断检查资源的**当前状态**（current state）和**期望状态**（desired state）是否一致，
- 如果不一致就尝试将其变成期望状态，这个过程称为 **reconcile**。

每个 controller 负责的东西都比较少，**所有 controller 并行运行， 由 kube-controller-manager 统一管理**。



#### 5.1 Deployments controller

**Deployments controller 启动**

当一个 Deployment record 存储到 etcd 并（被 initializers）初始化之后， kube-apiserver 就会将其置为对外可见的。此后， Deployment controller 监听了 Deployment 资源的变动，因此此时就会检测到这个新创建的资源。

```go
// pkg/controller/deployment/deployment_controller.go

// NewDeploymentController creates a new DeploymentController.
func NewDeploymentController(dInformer DeploymentInformer, rsInformer ReplicaSetInformer,
    podInformer PodInformer, client clientset.Interface) (*DeploymentController, error) {

    dc := &DeploymentController{
        client:        client,
        queue:         workqueue.NewNamedRateLimitingQueue(),
    }
    dc.rsControl = controller.RealRSControl{ // ReplicaSet controller
        KubeClient: client,
        Recorder:   dc.eventRecorder,
    }

    // 注册 Deployment 事件回调函数
    dInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc:    dc.addDeployment,    // 有 Deployment 创建时触发
        UpdateFunc: dc.updateDeployment,
        DeleteFunc: dc.deleteDeployment,
    })
    // 注册 ReplicaSet 事件回调函数
    rsInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc:    dc.addReplicaSet,
        UpdateFunc: dc.updateReplicaSet,
        DeleteFunc: dc.deleteReplicaSet,
    })
    // 注册 Pod 事件回调函数
    podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
        DeleteFunc: dc.deletePod,
    })

    dc.syncHandler = dc.syncDeployment
    dc.enqueueDeployment = dc.enqueue

    return dc, nil
}
```



**创建 Deployment：回调函数处理**

在本文场景中，触发的是 controller **注册的 addDeployment() 回调函数**[40]其所做的工作就是将 deployment 对象放到一个内部队列：

```go
// pkg/controller/deployment/deployment_controller.go

func (dc *DeploymentController) addDeployment(obj interface{}) {
    d := obj.(*apps.Deployment)
    dc.enqueueDeployment(d)
}
```



**主处理循环**

worker 不断遍历这个 queue，从中 dequeue item 并进行处理：

```go
// pkg/controller/deployment/deployment_controller.go

func (dc *DeploymentController) worker() {
    for dc.processNextWorkItem() {
    }
}

func (dc *DeploymentController) processNextWorkItem() bool {
    key, quit := dc.queue.Get()
    dc.syncHandler(key.(string)) // dc.syncHandler = dc.syncDeployment
}

// syncDeployment will sync the deployment with the given key.
func (dc *DeploymentController) syncDeployment(key string) error {
    namespace, name := cache.SplitMetaNamespaceKey(key)

    deployment := dc.dLister.Deployments(namespace).Get(name)
    d := deployment.DeepCopy()

    // 获取这个 Deployment 的所有 ReplicaSets, while reconciling ControllerRef through adoption/orphaning.
    rsList := dc.getReplicaSetsForDeployment(d)

    // 获取这个 Deployment 的所有 pods, grouped by their ReplicaSet
    podMap := dc.getPodMapForDeployment(d, rsList)

    if d.DeletionTimestamp != nil { // 这个 Deployment 已经被标记，等待被删除
        return dc.syncStatusOnly(d, rsList)
    }

    dc.checkPausedConditions(d)
    if d.Spec.Paused { // pause 状态
        return dc.sync(d, rsList)
    }

    if getRollbackTo(d) != nil {
        return dc.rollback(d, rsList)
    }

    scalingEvent := dc.isScalingEvent(d, rsList)
    if scalingEvent {
        return dc.sync(d, rsList)
    }

    switch d.Spec.Strategy.Type {
    case RecreateDeploymentStrategyType:             // re-create
        return dc.rolloutRecreate(d, rsList, podMap)
    case RollingUpdateDeploymentStrategyType:        // rolling-update
        return dc.rolloutRolling(d, rsList)
    }
    return fmt.Errorf("unexpected deployment strategy type: %s", d.Spec.Strategy.Type)
}
```

controller 会通过 label selector 从 kube-apiserver 查询 与这个 deployment 关联的 ReplicaSet 或 Pod records（然后发现没有）。

如果发现当前状态与预期状态不一致，就会触发同步过程（（synchronization process））。这个同步过程是无状态的，也就是说，它并不区分是新记录还是老记录，一视同仁。



**执行扩容（scale up）**

如上，发现 pod 不存在之后，它会开始扩容过程（scaling process）：

```go
// pkg/controller/deployment/sync.go

// scale up/down 或新创建（pause）时都会执行到这里
func (dc *DeploymentController) sync(d *apps.Deployment, rsList []*apps.ReplicaSet) error {

    newRS, oldRSs := dc.getAllReplicaSetsAndSyncRevision(d, rsList, false)
    dc.scale(d, newRS, oldRSs)

    // Clean up the deployment when it's paused and no rollback is in flight.
    if d.Spec.Paused && getRollbackTo(d) == nil {
        dc.cleanupDeployment(oldRSs, d)
    }

    allRSs := append(oldRSs, newRS)
    return dc.syncDeploymentStatus(allRSs, newRS, d)
}
```

大致步骤：

1. Rolling out (例如 creating）一个 ReplicaSet resource
2. 分配一个 label selector
3. 初始版本好（revision number）置为 1

ReplicaSet 的 PodSpec，以及其他一些 metadata 是从 Deployment 的 manifest 拷过来的。

最后会更新 deployment 状态，然后重新进入 reconciliation 循环，直到 deployment 进入预期的状态。



**小结**

由于 **Deployment controller 只负责 ReplicaSet 的创建**，因此下一步 （ReplicaSet -> Pod）要由 reconciliation 过程中的另一个 controller —— ReplicaSet controller 来完成。



#### 5.2 ReplicaSets controller

上一步骤，Deployments controller 已经创建了 Deployment 的第一个 ReplicaSet，但此时还没有任何 Pod。下面就轮到 ReplicaSet controller 出场了。它的任务是监控 ReplicaSet 及其依赖资源（pods）的生命周期，实现方式也是注册事件回调函数。

**ReplicaSets controller 启动**

```go
// pkg/controller/replicaset/replica_set.go

func NewReplicaSetController(rsInformer ReplicaSetInformer, podInformer PodInformer,
    kubeClient clientset.Interface, burstReplicas int) *ReplicaSetController {

    return NewBaseController(rsInformer, podInformer, kubeClient, burstReplicas,
        apps.SchemeGroupVersion.WithKind("ReplicaSet"),
        "replicaset_controller",
        "replicaset",
        controller.RealPodControl{
            KubeClient: kubeClient,
        },
    )
}

// 抽象出 NewBaseController() 是为了代码复用，例如 NewReplicationController() 也会调用这个函数。
func NewBaseController(rsInformer, podInformer, kubeClient clientset.Interface, burstReplicas int,
    gvk GroupVersionKind, metricOwnerName, queueName, podControl PodControlInterface) *ReplicaSetController {

    rsc := &ReplicaSetController{
        kubeClient:       kubeClient,
        podControl:       podControl,
        burstReplicas:    burstReplicas,
        expectations:     controller.NewUIDTrackingControllerExpectations(NewControllerExpectations()),
        queue:            workqueue.NewNamedRateLimitingQueue()
    }

    rsInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc:    rsc.addRS,
        UpdateFunc: rsc.updateRS,
        DeleteFunc: rsc.deleteRS,
    })
    rsc.rsLister = rsInformer.Lister()

    podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: rsc.addPod,
        UpdateFunc: rsc.updatePod,
        DeleteFunc: rsc.deletePod,
    })
    rsc.podLister = podInformer.Lister()

    rsc.syncHandler = rsc.syncReplicaSet
    return rsc
}
```



**创建 ReplicaSet：回调函数处理**

**主处理循环**

当一个 ReplicaSet 被（Deployment controller）创建之后，

```go
// pkg/controller/replicaset/replica_set.go

// syncReplicaSet will sync the ReplicaSet with the given key if it has had its expectations fulfilled,
// meaning it did not expect to see any more of its pods created or deleted.
func (rsc *ReplicaSetController) syncReplicaSet(key string) error {

    namespace, name := cache.SplitMetaNamespaceKey(key)
    rs := rsc.rsLister.ReplicaSets(namespace).Get(name)

    selector := metav1.LabelSelectorAsSelector(rs.Spec.Selector)

    // 包括那些不匹配 rs selector，但有 stale controller ref 的 pod
    allPods := rsc.podLister.Pods(rs.Namespace).List(labels.Everything())
    filteredPods := controller.FilterActivePods(allPods) // Ignore inactive pods.
    filteredPods = rsc.claimPods(rs, selector, filteredPods)

    if rsNeedsSync && rs.DeletionTimestamp == nil { // 需要同步，并且没有被标记待删除
        rsc.manageReplicas(filteredPods, rs)        // *主处理逻辑*
    }

    newStatus := calculateStatus(rs, filteredPods, manageReplicasErr)
    updatedRS := updateReplicaSetStatus(AppsV1().ReplicaSets(rs.Namespace), rs, newStatus)
}
```

RS controller 检查 ReplicaSet 的状态， 发现当前状态和期望状态之间有偏差（skew），因此接下来调用 `manageReplicas()` 来 reconcile 这个状态，在这里做的事情就是增加这个 ReplicaSet 的 pod 数量。

```go
// pkg/controller/replicaset/replica_set.go

func (rsc *ReplicaSetController) manageReplicas(filteredPods []*v1.Pod, rs *apps.ReplicaSet) error {
    diff := len(filteredPods) - int(*(rs.Spec.Replicas))
    rsKey := controller.KeyFunc(rs)

    if diff < 0 {
        diff *= -1
        if diff > rsc.burstReplicas {
            diff = rsc.burstReplicas
        }

        rsc.expectations.ExpectCreations(rsKey, diff)
        successfulCreations := slowStartBatch(diff, controller.SlowStartInitialBatchSize, func() {
            return rsc.podControl.CreatePodsWithControllerRef( // 扩容
                // 调用栈 CreatePodsWithControllerRef -> createPod() -> Client.CoreV1().Pods().Create()
                rs.Namespace, &rs.Spec.Template, rs, metav1.NewControllerRef(rs, rsc.GroupVersionKind))
        })

        // The skipped pods will be retried later. The next controller resync will retry the slow start process.
        if skippedPods := diff - successfulCreations; skippedPods > 0 {
            for i := 0; i < skippedPods; i++ {
                // Decrement the expected number of creates because the informer won't observe this pod
                rsc.expectations.CreationObserved(rsKey)
            }
        }
        return err
    } else if diff > 0 {
        if diff > rsc.burstReplicas {
            diff = rsc.burstReplicas
        }

        relatedPods := rsc.getIndirectlyRelatedPods(rs)
        podsToDelete := getPodsToDelete(filteredPods, relatedPods, diff)
        rsc.expectations.ExpectDeletions(rsKey, getPodKeys(podsToDelete))

        for _, pod := range podsToDelete {
            go func(targetPod *v1.Pod) {
                rsc.podControl.DeletePod(rs.Namespace, targetPod.Name, rs) // 缩容
            }(pod)
        }
    }

    return nil
}
```

增加 pod 数量的操作比较小心，每次最多不超过 burst count（这个配置是从 ReplicaSet 的父对象 Deployment 那里继承来的）。

另外，创建 Pods 的过程是 **批处理的**[41], “慢启动”操，开始时是 `SlowStartInitialBatchSize`，每执行成功一批，下次的 batch size 就翻倍。这样设计是为了避免给 kube-apiserver 造成不必要的压力，例如，如果由于 quota 不足，这批 pod 大部分都会失败，那 这种方式只会有一小批请求到达 kube-apiserver，而如果一把全上的话，请求全部会打过去。同样是失败，这种失败方式比较优雅。



**Owner reference**

K8s **通过 Owner Reference**（子资源中的一个字段，指向的是其父资源的 ID）**维护对象层级**（hierarchy）。这可以带来两方面好处：

1. 实现了 cascading deletion，即父对象被 GC 时会确保 GC 子对象；
2. 父对象之间不会出现竞争子对象的情况（例如，两个父对象认为某个子对象都是自己的）

另一个隐藏的好处是：Owner Reference 是有状态的：如果 controller 重启，重启期间不会影响 系统的其他部分，因为资源拓扑（resource topology）是独立于 controller 的。这种隔离设计也体现在 controller 自己的设计中：**controller 不应该操作 其他 controller 的资源**（resources they don't explicitly own）。

有时也可能会出现“孤儿”资源（"orphaned" resources）的情况，例如

1. 父资源删除了，子资源还在；
2. GC 策略导致子资源无法被删除。

这种情况发生时，**controller 会确保孤儿资源会被某个新的父资源收养**。多个父资源都可以竞争成为孤儿资源的父资源，但只有一个会成功（其余的会收到一个 validation 错误）。



#### 5.3 Informers

很多 controller（例如 RBAC authorizer 或 Deployment controller）需要将集群信息拉到本地。

例如 RBAC authorizer 中，authenticator 会将用户信息保存到请求上下文中。随后， RBAC authorizer 会用这个信息获取 etcd 中所有与这个用户相关的 role 和 role bindings。

那么，controller 是如何访问和修改这些资源的？在 K8s 中，这是通过 informer 机制实现的。

**informer 是一种 controller 订阅存储（etcd）事件的机制**，能方便地获取它们感兴趣的资源。

- 这种方式除了提供一种很好的抽象之外，还负责处理缓存（caching，非常重要，因为可以减少 kube-apiserver 连接数，降低 controller 测和 kube-apiserver 侧的序列化 成本）问题。
- 此外，这种设计还使得 controller 的行为是 threadsafe 的，避免影响其他组件或服务。

关于 informer 和 controller 的联合工作机制，可参考**这篇博客**[42]。



#### 5.4 Scheduler（调度器）

以上 controllers 执行完各自的处理之后，etcd 中已经有了一个 Deployment、一个 ReplicaSet 和三个 Pods，可以通过 kube-apiserver 查询到。但此时，**这三个 pod 还卡在 Pending 状态，因为它们还没有被调度到任何节点**。**另外一个 controller —— 调度器** —— 负责做这件事情。

scheduler 作为控制平面的一个独立服务运行，但工作方式与其他 controller 是一样的：监听事件，然后尝试 reconcile 状态。

**调用栈概览**

```go
Run // pkg/scheduler/scheduler.go
  |-SchedulingQueue.Run()
  |
  |-scheduleOne()
     |-bind
     |  |-RunBindPlugins
     |     |-runBindPlugins
     |        |-Bind
     |-sched.Algorithm.Schedule(pod)
        |-findNodesThatFitPod
        |-prioritizeNodes
        |-selectHost
```



**调度过程**

```go
// pkg/scheduler/core/generic_scheduler.go

// 将 pod 调度到指定 node list 中的某台 node 上
func (g *genericScheduler) Schedule(ctx context.Context, fwk framework.Framework,
    state *framework.CycleState, pod *v1.Pod) (result ScheduleResult, err error) {

    feasibleNodes, diagnosis := g.findNodesThatFitPod(ctx, fwk, state, pod) // 过滤可用 nodes
    if len(feasibleNodes) == 0 {
        return result, &framework.FitError{}
    }

    if len(feasibleNodes) == 1 { // 可用 node 只有一个，就选它了
        return ScheduleResult{SuggestedHost:  feasibleNodes[0].Name}, nil
    }

    priorityList := g.prioritizeNodes(ctx, fwk, state, pod, feasibleNodes)
    host := g.selectHost(priorityList)

    return ScheduleResult{
        SuggestedHost:  host,
        EvaluatedNodes: len(feasibleNodes) + len(diagnosis.NodeToStatusMap),
        FeasibleNodes:  len(feasibleNodes),
    }, err
}

// Filters nodes that fit the pod based on the framework filter plugins and filter extenders.
func (g *genericScheduler) findNodesThatFitPod(ctx context.Context, fwk framework.Framework,
    state *framework.CycleState, pod *v1.Pod) ([]*v1.Node, framework.Diagnosis, error) {

    diagnosis := framework.Diagnosis{
        NodeToStatusMap:      make(framework.NodeToStatusMap),
        UnschedulablePlugins: sets.NewString(),
    }

    // Run "prefilter" plugins.
    s := fwk.RunPreFilterPlugins(ctx, state, pod)
    allNodes := g.nodeInfoSnapshot.NodeInfos().List()

    if len(pod.Status.NominatedNodeName) > 0 && featureGate.Enabled(features.PreferNominatedNode) {
        feasibleNodes := g.evaluateNominatedNode(ctx, pod, fwk, state, diagnosis)
        if len(feasibleNodes) != 0 {
            return feasibleNodes, diagnosis, nil
        }
    }

    feasibleNodes := g.findNodesThatPassFilters(ctx, fwk, state, pod, diagnosis, allNodes)
    feasibleNodes = g.findNodesThatPassExtenders(pod, feasibleNodes, diagnosis.NodeToStatusMap)
    return feasibleNodes, diagnosis, nil
}
```

它会过滤 **过滤 PodSpect 中 NodeName 字段为空的 pods**[43]，尝试为这样的 pods 挑选一个 node 调度上去。



**调度算法**

下面简单看下内置的默认调度算法。

**注册默认 predicates**

这些 predicates 其实都是函数，被调用到时，执行相应的**过滤**[44]。例如，**如果 PodSpec 里面显式要求了 CPU 或 RAM 资源，而一个 node 无法满足这些条件**， 那就会将这个 node 从备选列表中删除。

```go
// pkg/scheduler/algorithmprovider/registry.go

// NewRegistry returns an algorithm provider registry instance.
func NewRegistry() Registry {
    defaultConfig := getDefaultConfig()
    applyFeatureGates(defaultConfig)

    caConfig := getClusterAutoscalerConfig()
    applyFeatureGates(caConfig)

    return Registry{
        schedulerapi.SchedulerDefaultProviderName: defaultConfig,
        ClusterAutoscalerProvider:                 caConfig,
    }
}

func getDefaultConfig() *schedulerapi.Plugins {
    plugins := &schedulerapi.Plugins{
        PreFilter: schedulerapi.PluginSet{...},
        Filter: schedulerapi.PluginSet{
            Enabled: []schedulerapi.Plugin{
                {Name: nodename.Name},        // 指定 node name 调度
                {Name: tainttoleration.Name}, // 指定 toleration 调度
                {Name: nodeaffinity.Name},    // 指定 node affinity 调度
                ...
            },
        },
        PostFilter: schedulerapi.PluginSet{...},
        PreScore: schedulerapi.PluginSet{...},
        Score: schedulerapi.PluginSet{
            Enabled: []schedulerapi.Plugin{
                {Name: interpodaffinity.Name, Weight: 1},
                {Name: nodeaffinity.Name, Weight: 1},
                {Name: tainttoleration.Name, Weight: 1},
                ...
            },
        },
        Reserve: schedulerapi.PluginSet{...},
        PreBind: schedulerapi.PluginSet{...},
        Bind: schedulerapi.PluginSet{...},
    }

    return plugins
}
```

plugin 的实现见 `pkg/scheduler/framework/plugins/`，以 `nodename` filter 为例：

```go
// pkg/scheduler/framework/plugins/nodename/node_name.go

// Filter invoked at the filter extension point.
func (pl *NodeName) Filter(ctx context.Context, pod *v1.Pod, nodeInfo *framework.NodeInfo) *framework.Status {
    if !Fits(pod, nodeInfo) {
        return framework.NewStatus(UnschedulableAndUnresolvable, ErrReason)
    }
    return nil
}

// 如果 pod 没有指定 NodeName，或者指定的 NodeName 等于该 node 的 name，返回 true；其他返回 false
func Fits(pod *v1.Pod, nodeInfo *framework.NodeInfo) bool {
    return len(pod.Spec.NodeName) == 0 || pod.Spec.NodeName == nodeInfo.Node().Name
}
```



**对筛选出的 node 排序**

选择了合适的 nodes 之后，接下来会执行一系列 priority function **对这些 nodes 进行排序**。例如，如果算法是希望将 pods 尽量分散到整个集群，那 priority 会选择资源尽量空闲的节点。

这些函数会给每个 node 打分，**得分最高的 node 会被选中**，调度到该节点。

```go
// pkg/scheduler/core/generic_scheduler.go

// 运行打分插件（score plugins）对 nodes 进行排序。
func (g *genericScheduler) prioritizeNodes(ctx context.Context, fwk framework.Framework,
    state *framework.CycleState, pod *v1.Pod, nodes []*v1.Node,) (framework.NodeScoreList, error) {

    // 如果没有指定 priority 配置，所有 node 将都得 1 分。
    if len(g.extenders) == 0 && !fwk.HasScorePlugins() {
        result := make(framework.NodeScoreList, 0, len(nodes))
        for i := range nodes {
            result = append(result, framework.NodeScore{ Name:  nodes[i].Name, Score: 1 })
        }
        return result, nil
    }

    preScoreStatus := fwk.RunPreScorePlugins(ctx, state, pod, nodes)       // PreScoe 插件
    scoresMap, scoreStatus := fwk.RunScorePlugins(ctx, state, pod, nodes)  // Score 插件

    result := make(framework.NodeScoreList, 0, len(nodes))
    for i := range nodes {
        result = append(result, framework.NodeScore{Name: nodes[i].Name, Score: 0})
        for j := range scoresMap {
            result[i].Score += scoresMap[j][i].Score
        }
    }

    if len(g.extenders) != 0 && nodes != nil {
        combinedScores := make(map[string]int64, len(nodes))
        for i := range g.extenders {
            if !g.extenders[i].IsInterested(pod) {
                continue
            }
            go func(extIndex int) {
                prioritizedList, weight := g.extenders[extIndex].Prioritize(pod, nodes)
                for i := range *prioritizedList {
                    host, score := (*prioritizedList)[i].Host, (*prioritizedList)[i].Score
                    combinedScores[host] += score * weight
                }
            }(i)
        }

        for i := range result {
            result[i].Score += combinedScores[result[i].Name] * (MaxNodeScore / MaxExtenderPriority)
        }
    }

    return result, nil
}
```



**创建 `v1.Binding` 对象**

算法选出一个 node 之后，调度器会**创建一个 Binding 对象**[45]， Pod 的 **ObjectReference 字段的值就是选中的 node 的名字**。

```go
// pkg/scheduler/framework/runtime/framework.go

func (f *frameworkImpl) runBindPlugin(ctx context.Context, bp BindPlugin, state *CycleState,
    pod *v1.Pod, nodeName string) *framework.Status {

    if !state.ShouldRecordPluginMetrics() {
        return bp.Bind(ctx, state, pod, nodeName)
    }

    status := bp.Bind(ctx, state, pod, nodeName)
    return status
}
// pkg/scheduler/framework/plugins/defaultbinder/default_binder.go

// Bind binds pods to nodes using the k8s client.
func (b DefaultBinder) Bind(ctx, state *CycleState, p *v1.Pod, nodeName string) *framework.Status {
    binding := &v1.Binding{
        ObjectMeta: metav1.ObjectMeta{Namespace: p.Namespace, Name: p.Name, UID: p.UID},
        Target:     v1.ObjectReference{Kind: "Node", Name: nodeName}, // ObjectReference 字段为 nodeName
    }

    b.handle.ClientSet().CoreV1().Pods(binding.Namespace).Bind(ctx, binding, metav1.CreateOptions{})
}
```

如上，最后 `ClientSet().CoreV1().Pods(binding.Namespace).Bind()` 通过一个 **POST 请求发给 apiserver**。



**kube-apiserver 更新 pod 对象**

kube-apiserver 收到这个 Binding object 请求后，registry 反序列化对象，更新 Pod 对象的下列字段：

- 设置 NodeName
- 添加 annotations
- 设置 `PodScheduled` status 为 `True`

```go
// pkg/registry/core/pod/storage/storage.go

func (r *BindingREST) setPodHostAndAnnotations(ctx context.Context, podID, oldMachine, machine string,
    annotations map[string]string, dryRun bool) (finalPod *api.Pod, err error) {

    podKey := r.store.KeyFunc(ctx, podID)
    r.store.Storage.GuaranteedUpdate(ctx, podKey, &api.Pod{}, false, nil,
        storage.SimpleUpdate(func(obj runtime.Object) (runtime.Object, error) {

        pod, ok := obj.(*api.Pod)
        pod.Spec.NodeName = machine
        if pod.Annotations == nil {
            pod.Annotations = make(map[string]string)
        }
        for k, v := range annotations {
            pod.Annotations[k] = v
        }
        podutil.UpdatePodCondition(&pod.Status, &api.PodCondition{
            Type:   api.PodScheduled,
            Status: api.ConditionTrue,
        })

        return pod, nil
    }), dryRun, nil)
}
```



**自定义调度器**

> predicate 和 priority function 都是可扩展的，可以通过 `--policy-config-file` 指定。
>
> K8s 还可以自定义调度器（自己实现调度逻辑）。**如果 PodSpec 中 schedulerName 字段不为空**，K8s 就会 将这个 pod 的调度权交给指定的调度器。



#### 5.5 小结

总结一下前面已经完成的步骤：

1. HTTP 请求通过了认证、鉴权、admission control
2. Deployment, ReplicaSet 和 Pod resources 已经持久化到 etcd
3. 一系列 initializers 已经执行完毕，
4. 每个 Pod 也已经调度到了合适的 node 上。

但是，**到目前为止，我们看到的所有东西（状态），还只是存在于 etcd 中的元数据**。下一步就是将这些状态同步到计算节点上，然后计算节点上的 agent（kubelet）就开始干活了。



### 6 kubelet

每个 K8s node 上都会运行一个名为 kubelet 的 agent，它负责

- pod 生命周期管理。

  这意味着，它负责将 “Pod” 的逻辑抽象（etcd 中的元数据）转换成具体的容器（container）。

- 挂载目录

- 创建容器日志

- 垃圾回收等等



#### 6.1 Pod sync（状态同步）

**kubelet 也可以认为是一个 controller**，它

1. 通过 ListWatch 接口，从 kube-apiserver **获取属于本节点的 Pod 列表**（根据`spec.nodeName` **过滤**[46]），
2. 然后与自己缓存的 pod 列表对比，如果有 pod 创建、删除、更新等操作，就开始同步状态。

下面具体看一下同步过程。

##### 同步过程

```go
// pkg/kubelet/kubelet.go

// syncPod is the transaction script for the sync of a single pod.
func (kl *Kubelet) syncPod(o syncPodOptions) error {
    pod := o.pod

    if updateType == SyncPodKill { // kill pod 操作
        kl.killPod(pod, nil, podStatus, PodTerminationGracePeriodSecondsOverride)
        return nil
    }

    firstSeenTime := pod.Annotations["kubernetes.io/config.seen"] // 测量 latency，从 apiserver 第一次看到 pod 算起

    if updateType == SyncPodCreate { // create pod 操作
        if !firstSeenTime.IsZero() { // Record pod worker start latency if being created
            metrics.PodWorkerStartDuration.Observe(metrics.SinceInSeconds(firstSeenTime))
        }
    }

    // Generate final API pod status with pod and status manager status
    apiPodStatus := kl.generateAPIPodStatus(pod, podStatus)

    podStatus.IPs = []string{}
    if len(podStatus.IPs) == 0 && len(apiPodStatus.PodIP) > 0 {
        podStatus.IPs = []string{apiPodStatus.PodIP}
    }

    runnable := kl.canRunPod(pod)
    if !runnable.Admit { // Pod is not runnable; update the Pod and Container statuses to why.
        apiPodStatus.Reason = runnable.Reason
        ...
    }

    kl.statusManager.SetPodStatus(pod, apiPodStatus)

    // Kill pod if it should not be running
    if !runnable.Admit || pod.DeletionTimestamp != nil || apiPodStatus.Phase == v1.PodFailed {
        return kl.killPod(pod, nil, podStatus, nil)
    }

    // 如果 network plugin not ready，并且 pod 网络不是 host network 类型，返回相应错误
    if err := kl.runtimeState.networkErrors(); err != nil && !IsHostNetworkPod(pod) {
        return fmt.Errorf("%s: %v", NetworkNotReadyErrorMsg, err)
    }

    // Create Cgroups for the pod and apply resource parameters if cgroups-per-qos flag is enabled.
    pcm := kl.containerManager.NewPodContainerManager()

    if kubetypes.IsStaticPod(pod) { // Create Mirror Pod for Static Pod if it doesn't already exist
        ...
    }

    kl.makePodDataDirs(pod)                     // Make data directories for the pod
    kl.volumeManager.WaitForAttachAndMount(pod) // Wait for volumes to attach/mount
    pullSecrets := kl.getPullSecretsForPod(pod) // Fetch the pull secrets for the pod

    // Call the container runtime's SyncPod callback
    result := kl.containerRuntime.SyncPod(pod, podStatus, pullSecrets, kl.backOff)
    kl.reasonCache.Update(pod.UID, result)
}
```

1. 如果是 pod 创建事件，会记录一些 pod latency 相关的 metrics；

2. 然后调用 `generateAPIPodStatus()` **生成一个 v1.PodStatus 对象**，代表 pod 当前阶段（Phase）的状态。

   Pod 的 Phase 是对其生命周期中不同阶段的高层抽象，非常复杂，后面会介绍。

3. PodStatus 生成之后，将发送给 Pod status manager，后者的任务是异步地通过 apiserver 更新 etcd 记录。

4. 接下来会运行一系列 admission handlers，确保 pod 有正确的安全权限（security permissions）。

   其中包括 enforcing **AppArmor profiles and `NO_NEW_PRIVS`**[47]。在这个阶段被 deny 的 Pods 将无限期处于 Pending 状态。

5. 如果指定了 `cgroups-per-qos`，kubelet 将为这个 pod 创建 cgroups。可以实现更好的 QoS。

6. **为容器创建一些目录**。包括

7. - pod 目录 （一般是 `/var/run/kubelet/pods/<podID>`）
   - volume 目录 (`<podDir>/volumes`)
   - plugin 目录 (`<podDir>/plugins`).

8. volume manager 将 **等待**[48]`Spec.Volumes` 中定义的 volumes attach 完成。取决于 volume 类型，pod 可能会等待很长时间（例如 cloud 或 NFS volumes）。

9. 从 apiserver 获取 `Spec.ImagePullSecrets` 中指定的 **secrets，注入容器**。

10. **容器运行时（runtime）创建容器**（后面详细描述）。



##### Pod 状态

前面提到，`generateAPIPodStatus()` **生成一个 v1.PodStatus**[49]对象，代表 pod 当前阶段（Phase）的状态。

Pod 的 Phase 是对其生命周期中不同阶段的高层抽象，包括

- `Pending`
- `Running`
- `Succeeded`
- `Failed`
- `Unknown`

生成这个状态的过程非常复杂，一些细节如下：

1. 首先，顺序执行一系列 `PodSyncHandlers` 。每个 handler **判断这个 pod 是否还应该留在这个 node 上**。如果其中任何一个判断结果是否，那 pod 的 phase **将变为**[50]`PodFailed` 并最终会被从这个 node 驱逐。

   一个例子是 pod 的 `activeDeadlineSeconds` （Jobs 中会用到）超时之后，就会被驱逐。

2. 接下来决定 Pod Phase 的将是其 init 和 real containers。由于此时容器还未启动，因此将处于 **waiting**[51] **状态**。**有 waiting 状态 container 的 pod，将处于 `Pending`[52] Phase**。

3. 由于此时容器运行时还未创建我们的容器 ，因此它将把 **`PodReady` 字段置为 False**[53].



#### 6.2 CRI 及创建 pause 容器

至此，大部分准备工作都已完成，接下来即将创建容器了。**创建容器是通过 Container Runtime （例如 `docker` 或 `rkt`）完成的**。

为实现可扩展，kubelet 从 v1.5.0 开始，**使用 CRI（Container Runtime Interface）与具体的容器运行时交互**。简单来说，CRI 提供了 kubelet 和具体 runtime implementation 之间的抽象接口， 用 **protocol buffers**[54] 和 gRPC 通信。

##### CRI SyncPod

```go
// pkg/kubelet/kuberuntime/kuberuntime_manager.go

// SyncPod syncs the running pod into the desired pod by executing following steps:
//  1. Compute sandbox and container changes.
//  2. Kill pod sandbox if necessary.
//  3. Kill any containers that should not be running.
//  4. Create sandbox if necessary.
//  5. Create ephemeral containers.
//  6. Create init containers.
//  7. Create normal containers.
//
func (m *kubeGenericRuntimeManager) SyncPod(pod *v1.Pod, podStatus *kubecontainer.PodStatus,
    pullSecrets []v1.Secret, backOff *flowcontrol.Backoff) (result kubecontainer.PodSyncResult) {

    // Step 1: Compute sandbox and container changes.
    podContainerChanges := m.computePodActions(pod, podStatus)
    if podContainerChanges.CreateSandbox {
        ref := ref.GetReference(legacyscheme.Scheme, pod)
        if podContainerChanges.SandboxID != "" {
            m.recorder.Eventf("Pod sandbox changed, it will be killed and re-created.")
        } else {
            InfoS("SyncPod received new pod, will create a sandbox for it")
        }
    }

    // Step 2: Kill the pod if the sandbox has changed.
    if podContainerChanges.KillPod {
        if podContainerChanges.CreateSandbox {
            InfoS("Stopping PodSandbox for pod, will start new one")
        } else {
            InfoS("Stopping PodSandbox for pod, because all other containers are dead")
        }

        killResult := m.killPodWithSyncResult(pod, ConvertPodStatusToRunningPod(m.runtimeName, podStatus), nil)
        result.AddPodSyncResult(killResult)

        if podContainerChanges.CreateSandbox {
            m.purgeInitContainers(pod, podStatus)
        }
    } else {
        // Step 3: kill any running containers in this pod which are not to keep.
        for containerID, containerInfo := range podContainerChanges.ContainersToKill {
            killContainerResult := NewSyncResult(kubecontainer.KillContainer, containerInfo.name)
            result.AddSyncResult(killContainerResult)
            m.killContainer(pod, containerID, containerInfo)
        }
    }

    // Keep terminated init containers fairly aggressively controlled
    // This is an optimization because container removals are typically handled by container GC.
    m.pruneInitContainersBeforeStart(pod, podStatus)

    // Step 4: Create a sandbox for the pod if necessary.
    podSandboxID := podContainerChanges.SandboxID
    if podContainerChanges.CreateSandbox {
        createSandboxResult := kubecontainer.NewSyncResult(kubecontainer.CreatePodSandbox, format.Pod(pod))
        result.AddSyncResult(createSandboxResult)
        podSandboxID, msg = m.createPodSandbox(pod, podContainerChanges.Attempt)
        podSandboxStatus := m.runtimeService.PodSandboxStatus(podSandboxID)
    }

    // the start containers routines depend on pod ip(as in primary pod ip)
    // instead of trying to figure out if we have 0 < len(podIPs) everytime, we short circuit it here
    podIP := ""
    if len(podIPs) != 0 {
        podIP = podIPs[0]
    }

    // Get podSandboxConfig for containers to start.
    configPodSandboxResult := kubecontainer.NewSyncResult(ConfigPodSandbox, podSandboxID)
    result.AddSyncResult(configPodSandboxResult)
    podSandboxConfig := m.generatePodSandboxConfig(pod, podContainerChanges.Attempt)

    // Helper containing boilerplate common to starting all types of containers.
    // typeName is a label used to describe this type of container in log messages,
    // currently: "container", "init container" or "ephemeral container"
    start := func(typeName string, spec *startSpec) error {
        startContainerResult := kubecontainer.NewSyncResult(kubecontainer.StartContainer, spec.container.Name)
        result.AddSyncResult(startContainerResult)

        isInBackOff, msg := m.doBackOff(pod, spec.container, podStatus, backOff)
        if isInBackOff {
            startContainerResult.Fail(err, msg)
            return err
        }

        m.startContainer(podSandboxID, podSandboxConfig, spec, pod, podStatus, pullSecrets, podIP, podIPs)
        return nil
    }

    // Step 5: start ephemeral containers
    // These are started "prior" to init containers to allow running ephemeral containers even when there
    // are errors starting an init container. In practice init containers will start first since ephemeral
    // containers cannot be specified on pod creation.
    for _, idx := range podContainerChanges.EphemeralContainersToStart {
        start("ephemeral container", ephemeralContainerStartSpec(&pod.Spec.EphemeralContainers[idx]))
    }

    // Step 6: start the init container.
    if container := podContainerChanges.NextInitContainerToStart; container != nil {
        start("init container", containerStartSpec(container))
    }

    // Step 7: start containers in podContainerChanges.ContainersToStart.
    for _, idx := range podContainerChanges.ContainersToStart {
        start("container", containerStartSpec(&pod.Spec.Containers[idx]))
    }
}
```



##### CRI create sandbox

kubelet **发起 RunPodSandbox**[55] RPC 调用。

**“sandbox” 是一个 CRI 术语，它表示一组容器，在 K8s 里就是一个 Pod**。这个词是有意用作比较宽泛的描述，这样对其他运行时的描述也是适用的（例如，在基于 hypervisor 的运行时中，sandbox 可能是一个虚拟机）。

```go
// pkg/kubelet/kuberuntime/kuberuntime_sandbox.go

// createPodSandbox creates a pod sandbox and returns (podSandBoxID, message, error).
func (m *kubeGenericRuntimeManager) createPodSandbox(pod *v1.Pod, attempt uint32) (string, string, error) {
    podSandboxConfig := m.generatePodSandboxConfig(pod, attempt)

    // 创建 pod log 目录
    m.osInterface.MkdirAll(podSandboxConfig.LogDirectory, 0755)

    runtimeHandler := ""
    if m.runtimeClassManager != nil {
        runtimeHandler = m.runtimeClassManager.LookupRuntimeHandler(pod.Spec.RuntimeClassName)
        if runtimeHandler != "" {
            InfoS("Running pod with runtime handler", runtimeHandler)
        }
    }

    podSandBoxID := m.runtimeService.RunPodSandbox(podSandboxConfig, runtimeHandler)
    return podSandBoxID, "", nil
}
// pkg/kubelet/cri/remote/remote_runtime.go

// RunPodSandbox creates and starts a pod-level sandbox.
func (r *remoteRuntimeService) RunPodSandbox(config *PodSandboxConfig, runtimeHandler string) (string, error) {

    InfoS("[RemoteRuntimeService] RunPodSandbox", "config", config, "runtimeHandler", runtimeHandler)

    resp := r.runtimeClient.RunPodSandbox(ctx, &runtimeapi.RunPodSandboxRequest{
        Config:         config,
        RuntimeHandler: runtimeHandler,
    })

    InfoS("[RemoteRuntimeService] RunPodSandbox Response", "podSandboxID", resp.PodSandboxId)
    return resp.PodSandboxId, nil
}
```



##### Create sandbox：docker 相关代码

前面是 CRI 通用代码，如果我们的容器 runtime 是 docker，那接下来就会调用到 docker 相关代码。

在这种 runtime 中，**创建一个 sandbox 会转换成创建一个 “pause” 容器的操作**。Pause container 作为一个 pod 内其他所有容器的父角色，hold 了很多 pod-level 的资源， 具体说就是 Linux namespace，例如 IPC NS、Net NS、IPD NS。

"pause" container 提供了一种持有这些 ns、让所有子容器共享它们 的方式。例如，共享 netns 的好处之一是，pod 内不同容器之间可以通过 localhost 方式访问彼此。pause 容器的第二个用处是回收（reaping）dead processes。更多信息，可参考 **这篇博客**[56]。

Pause 容器创建之后，会被 checkpoint 到磁盘，然后启动。

```go
// pkg/kubelet/dockershim/docker_sandbox.go

// 对于 docker runtime，PodSandbox 实现为一个 holding 网络命名空间（netns）的容器
func (ds *dockerService) RunPodSandbox(ctx context.Context, r *RunPodSandboxRequest) (*RunPodSandboxResponse) {

    // Step 1: Pull the image for the sandbox.
    ensureSandboxImageExists(ds.client, image)

    // Step 2: Create the sandbox container.
    createConfig := ds.makeSandboxDockerConfig(config, image)
    createResp := ds.client.CreateContainer(*createConfig)
    resp := &runtimeapi.RunPodSandboxResponse{PodSandboxId: createResp.ID}

    ds.setNetworkReady(createResp.ID, false) // 容器 network 状态初始化为 false

    // Step 3: Create Sandbox Checkpoint.
    CreateCheckpoint(createResp.ID, constructPodSandboxCheckpoint(config))

    // Step 4: Start the sandbox container。 如果失败，kubelet 会 GC 掉 sandbox
    ds.client.StartContainer(createResp.ID)

    rewriteResolvFile()

    // 如果是 hostNetwork 类型，到这里就可以返回了，无需下面的 CNI 流程
    if GetNetwork() == NamespaceMode_NODE {
        return resp, nil
    }

    // Step 5: Setup networking for the sandbox with CNI
    // 包括分配 IP、设置 sandbox 内的路由、创建虚拟网卡等。
    cID := kubecontainer.BuildContainerID(runtimeName, createResp.ID)
    ds.network.SetUpPod(Namespace, Name, cID, Annotations, networkOptions)

    return resp, nil
}
```

最后调用的 `SetUpPod()` 为容器创建网络，它有会调用到 plugin manager 的同名方法：

```go
// pkg/kubelet/dockershim/network/plugins.go

func (pm *PluginManager) SetUpPod(podNamespace, podName, id ContainerID, annotations, options) error {
    const operation = "set_up_pod"
    fullPodName := kubecontainer.BuildPodFullName(podName, podNamespace)

    // 调用 CNI 插件为容器设置网络
    pm.plugin.SetUpPod(podNamespace, podName, id, annotations, options)
}
```

> Cgroup 也很重要，是 Linux 掌管资源分配的方式，docker 利用它实现资源隔离。更多信息，参考 **What even is a Container?**[57]



#### 6.3 CNI 前半部分：CNI plugin manager 处理

现在我们的 pod 已经有了一个占坑用的 pause 容器，它占住了 pod 需要用到的所有 namespace。接下来需要做的就是：**调用底层的具体网络方案**（bridge/flannel/calico/cilium 等等） 提供的 CNI 插件，**创建并打通容器的网络**。

CNI 是 Container Network Interface 的缩写，工作机制与 Container Runtime Interface 类似。简单来说，CNI 是一个抽象接口，不同的网络提供商只要实现了 CNI 中的几个方法，就能接入 K8s，为容器创建网络。kubelet 与 CNI 插件之间通过 JSON 数据交互（配置文件放在 `/etc/cni/net.d`），通过 stdin 将配置数据传递给 CNI binary (located in `/opt/cni/bin`)。

CNI 插件有自己的配置，例如，内置的 bridge 插件可能配置如下：

```json
{
    "cniVersion": "0.3.1",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "${POD_CIDR}"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
```

还会通过 `CNI_ARGS` 环境变量传递 pod metadata，例如 name 和 ns。



##### 调用栈概览

下面的调用栈是 CNI 前半部分：**CNI plugin manager 调用到具体的 CNI 插件**（可执行文件）， 执行 shell 命令为容器创建网络：

```go
SetUpPod                                                  // pkg/kubelet/dockershim/network/cni/cni.go
 |-ns = plugin.host.GetNetNS(id)
 |-plugin.addToNetwork(name, id, ns)                      // -> pkg/kubelet/dockershim/network/cni/cni.go
    |-plugin.buildCNIRuntimeConf
    |-cniNet.AddNetworkList(netConf)                      // -> github.com/containernetworking/cni/libcni/api.go
       |-for net := range list.Plugins
       |   result = c.addNetwork
       |              |-pluginPath = FindInPath(c.Path)
       |              |-ValidateContainerID(ContainerID)
       |              |-ValidateNetworkName(name)
       |              |-ValidateInterfaceName(IfName)
       |              |-invoke.ExecPluginWithResult(pluginPath, c.args("ADD", rt))
       |                        |-shell("/opt/cni/bin/xx <args>")
       |
       |-c.cacheAdd(result, list.Bytes, list.Name, rt)
```

最后一层调用 `ExecPlugin()`：

```go
// vendor/github.com/containernetworking/cni/pkg/invoke/raw_exec.go

func (e *RawExec) ExecPlugin(ctx, pluginPath, stdinData []byte, environ []string) ([]byte, error) {
    c := exec.CommandContext(ctx, pluginPath)
    c.Env = environ
    c.Stdin = bytes.NewBuffer(stdinData)
    c.Stdout = stdout
    c.Stderr = stderr

    for i := 0; i <= 5; i++ { // Retry the command on "text file busy" errors
        err := c.Run()
        if err == nil { // Command succeeded
            break
        }

        if strings.Contains(err.Error(), "text file busy") {
            time.Sleep(time.Second)
            continue
        }

        // All other errors except than the busy text file
        return nil, e.pluginErr(err, stdout.Bytes(), stderr.Bytes())
    }

    return stdout.Bytes(), nil
}
```

可以看到，经过上面的几层调用，最终是通过 shell 命令执行了宿主机上的 CNI 插件， 例如 `/opt/cni/bin/cilium-cni`，并通过 stdin 传递了一些 JSON 参数。



#### 6.4 CNI 后半部分：CNI plugin 实现

下面看 CNI 处理的后半部分：CNI 插件为容器创建网络，也就是可执行文件 `/opt/cni/bin/xxx` 的实现。

CNI 相关的代码维护在一个单独的项目 **github.com/containernetworking/cni**[58]。每个 CNI 插件只需要实现其中的几个方法，然后**编译成独立的可执行文件**，放在 `/etc/cni/bin` 下面即可。下面是一些具体的插件，

```sh
$ ls /opt/cni/bin/
bridge  cilium-cni  cnitool  dhcp  host-local  ipvlan  loopback  macvlan  noop
```



##### 调用栈概览

CNI 插件（可执行文件）执行时会调用到 `PluginMain()`，从这往后的调用栈 （**注意源文件都是 `github.com/containernetworking/cni` 项目中的路径**）：

```go
PluginMain                                                     // pkg/skel/skel.go
 |-PluginMainWithError                                         // pkg/skel/skel.go
   |-pluginMain                                                // pkg/skel/skel.go
      |-switch cmd {
          case "ADD":
            checkVersionAndCall(cmdArgs, cmdAdd)               // pkg/skel/skel.go
              |-configVersion = Decode(cmdArgs.StdinData)
              |-Check(configVersion, pluginVersionInfo)
              |-toCall(cmdArgs) // toCall == cmdAdd
                 |-cmdAdd(cmdArgs)
                   |-specific CNI plugin implementations

          case "DEL":
            checkVersionAndCall(cmdArgs, cmdDel)
          case "VERSION":
            versionInfo.Encode(t.Stdout)
          default:
            return createTypedError("unknown CNI_COMMAND: %v", cmd)
        }
```

可见对于 kubelet 传过来的 "ADD" 命令，最终会调用到 CNI 插件的 cmdAdd() 方法 —— 该方法默认是空的，需要由每种 CNI 插件自己实现。同理，删除 pod 时对应的是 `"DEL"` 操作，调用到的 `cmdDel()` 方法也是要由具体 CNI 插件实现的。



##### CNI 插件实现举例：Bridge

**github.com/containernetworking/plugins**[59]项目中包含了很多种 CNI plugin 的实现，例如 IPVLAN、Bridge、MACVLAN、VLAN 等等。

`bridge` CNI plugin 的实现见**plugins/main/bridge/bridge.go**[60]

执行逻辑如下：

1. 在默认 netns 创建一个 Linux bridge，这台宿主机上的所有容器都将连接到这个 bridge。

2. 创建一个 veth pair，将容器和 bridge 连起来。

3. 分配一个 IP 地址，配置到 pause 容器，设置路由。

   IP 从配套的网络服务 IPAM（IP Address Management）中分配的。最场景的 IPAM plugin 是`host-local`，它从预先设置的一个网段里分配一个 IP，并将状态信息写到宿主机的本地文件系统，因此重启不会丢失。`host-local` IPAM 的实现见 **plugins/ipam/host-local**[61]。

4. 修改 `resolv.conf`，为容器配置 DNS。这里的 DNS 信息是从传给 CNI plugin 的参数中解析的。

以上过程完成之后，容器和宿主机（以及同宿主机的其他容器）之间的网络就通了， CNI 插件会将结果以 JSON 返回给 kubelet。



##### CNI 插件实现举例：Noop

再来看另一种比较有趣的 CNI 插件：`noop`。这个插件是 CNI 项目自带的， 代码见 **plugins/test/noop/main.go**[62]。

```go
func cmdAdd(args *skel.CmdArgs) error {
    return debugBehavior(args, "ADD")
}

func cmdDel(args *skel.CmdArgs) error {
    return debugBehavior(args, "DEL")
}
```

从名字以及以上代码可以看出，这个 CNI 插件（几乎）什么事情都不做。用途：

1. **测试或调试**：它可以打印 debug 信息。

2. 给只支持 hostNetwork 的节点使用。

   每个 node 上必须有一个配置正确的 CNI 插件，kubelet 自检才能通过，否则 node 会处于 NotReady 状态。

   某些情况下，我们不想让一些 node（例如 master node）承担正常的、创建带 IP pod 的工作， 只要它能创建 hostNetwork 类型的 pod 就行了（这样就无需给这些 node 分配 PodCIDR， 也不需要在 node 上启动 IPAM 服务）。

   这种情况下，就可以用 noop 插件。参考配置：

   ```sh
   $ cat /etc/cni/net.d/98-noop.conf
   {
       "cniVersion": "0.3.1",
       "type": "noop"
   }
   ```



##### CNI 插件实现举例：Cilium

这个就很复杂了，做的事情非常多，可参考 Cilium Code Walk Through: CNI Create Network。



#### 6.5 为容器配置跨节点通信网络（inter-host networking）

这项工作不在 K8s 及 CNI 插件的职责范围内，是由具体网络方案 在节点上的 agent 完成的，例如 flannel 网络的 flanneld，cilium 网络的 cilium-agent。

简单来说，跨节点通信有两种方式：

1. 隧道（tunnel or overlay）
2. 直接路由

这里赞不展开，可参考迈入 Cilium+BGP 的云原生网络时代。



#### 6.6 创建 `init` 容器及业务容器

至此，网络部分都配置好了。接下来就开始启动真正的业务容器。

Sandbox 容器初始化完成后，kubelet 就开始创建其他容器。首先会启动 `PodSpec` 中指定的所有 init 容器，**代码**[63]然后才启动主容器（main containers）。

##### 调用栈概览

```go
startContainer
 |-m.runtimeService.CreateContainer                      // pkg/kubelet/cri/remote/remote_runtime.go
 |  |-r.runtimeClient.CreateContainer                    // -> pkg/kubelet/dockershim/docker_container.go
 |       |-new(CreateContainerResponse)                  // staging/src/k8s.io/cri-api/pkg/apis/runtime/v1/api.pb.go
 |       |-Invoke("/runtime.v1.RuntimeService/CreateContainer")
 |
 |  CreateContainer // pkg/kubelet/dockershim/docker_container.go
 |      |-ds.client.CreateContainer                      // -> pkg/kubelet/dockershim/libdocker/instrumented_client.go
 |             |-d.client.ContainerCreate                // -> vendor/github.com/docker/docker/client/container_create.go
 |                |-cli.post("/containers/create")
 |                |-json.NewDecoder().Decode(&resp)
 |
 |-m.runtimeService.StartContainer(containerID)          // -> pkg/kubelet/cri/remote/remote_runtime.go
    |-r.runtimeClient.StartContainer
         |-new(CreateContainerResponse)                  // staging/src/k8s.io/cri-api/pkg/apis/runtime/v1/api.pb.go
         |-Invoke("/runtime.v1.RuntimeService/StartContainer")
```



##### 具体过程

```go
// pkg/kubelet/kuberuntime/kuberuntime_container.go

func (m *kubeGenericRuntimeManager) startContainer(podSandboxID, podSandboxConfig, spec *startSpec, pod *v1.Pod,
     podStatus *PodStatus, pullSecrets []v1.Secret, podIP string, podIPs []string) (string, error) {

    container := spec.container

    // Step 1: 拉镜像
    m.imagePuller.EnsureImageExists(pod, container, pullSecrets, podSandboxConfig)

    // Step 2: 通过 CRI 创建容器
    containerConfig := m.generateContainerConfig(container, pod, restartCount, podIP, imageRef, podIPs, target)

    m.internalLifecycle.PreCreateContainer(pod, container, containerConfig)
    containerID := m.runtimeService.CreateContainer(podSandboxID, containerConfig, podSandboxConfig)
    m.internalLifecycle.PreStartContainer(pod, container, containerID)

    // Step 3: 启动容器
    m.runtimeService.StartContainer(containerID)

    legacySymlink := legacyLogSymlink(containerID, containerMeta.Name, sandboxMeta.Name, sandboxMeta.Namespace)
    m.osInterface.Symlink(containerLog, legacySymlink)

    // Step 4: 执行 post start hook
    m.runner.Run(kubeContainerID, pod, container, container.Lifecycle.PostStart)
}
```

过程：

1. **拉镜像**[64]。如果是私有镜像仓库，就会从 PodSpec 中寻找访问仓库用的 secrets。

2. 通过 CRI **创建 container**[65]。

   从 parent PodSpec 的 `ContainerConfig` struct 中解析参数（command, image, labels, mounts, devices, env variables 等等）， 然后通过 protobuf 发送给 CRI plugin。例如对于 docker，收到请求后会反序列化，从中提取自己需要的参数，然后发送给 Daemon API。过程中它会给容器添加几个 metadata labels （例如 container type, log path, sandbox ID）。

3. 然后通过 `runtimeService.startContainer()` 启动容器；

4. 如果注册了 post-start hooks，接下来就执行这些 hooks。**post Hook 类型**：

- `Exec`：在容器内执行具体的 shell 命令。
- `HTTP`：对容器内的服务（endpoint）发起 HTTP 请求。

如果 PostStart hook 运行时间过长，或者 hang 住或失败了，容器就无法进入 `running` 状态。



### 7 结束

至此，应该已经有 3 个 pod 在运行了，取决于系统资源和调度策略，它们可能在一台 node 上，也可能分散在多台。

#### 脚注

[1]What happens when ... Kubernetes edition!: *https://github.com/jamiehannaford/what-happens-when-k8s*

[2]`v1.21`: *https://github.com/kubernetes/kubernetes/tree/v1.21.1*

[3]generic apiserver: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/cmd/kube-apiserver/app/server.go#L219*

[4]Config.OpenAPIConfig 字段: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/staging/src/k8s.io/apiserver/pkg/server/config.go#L167*

[5]storage provider: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/staging/src/k8s.io/kube-aggregator/pkg/apiserver/apiserver.go#L204*

[6]配置 REST mappings: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/staging/src/k8s.io/apiserver/pkg/endpoints/groupversion.go#L92*

[7]镜像名为空或格式不对: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/staging/src/k8s.io/kubectl/pkg/cmd/run/run.go#L262*

[8]generator: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/staging/src/k8s.io/kubectl/pkg/cmd/run/run.go#L300*

[9]文档: *https://kubernetes.io/docs/user-guide/kubectl-conventions/#generators*

[10]`BasicPod`: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/staging/src/k8s.io/kubectl/pkg/generate/versioned/run.go#L233*

[11]实现: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/staging/src/k8s.io/kubectl/pkg/generate/versioned/run.go#L259*

[12]搜索合适的 API group 和版本: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/staging/src/k8s.io/kubectl/pkg/cmd/run/run.go#L610-L619*

[13]创建一个正确版本的客户端（versioned client）: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/staging/src/k8s.io/kubectl/pkg/cmd/run/run.go#L641*

[14]缓存这份 OpenAPI schema: *https://github.com/kubernetes/kubernetes/blob/v1.14.0/staging/src/k8s.io/cli-runtime/pkg/genericclioptions/config_flags.go#L234*

[15]发送出去: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/staging/src/k8s.io/kubectl/pkg/cmd/run/run.go#L654*

[16]预定义的路径: *https://github.com/kubernetes/client-go/blob/v1.21.0/tools/clientcmd/loader.go#L52*

[17]TLS: *https://github.com/kubernetes/client-go/blob/82aa063804cf055e16e8911250f888bc216e8b61/rest/transport.go#L80-L89*

[18]发送: *https://github.com/kubernetes/client-go/blob/c6f8cf2c47d21d55fa0df928291b2580544886c8/transport/round_trippers.go#L314*

[19]发送: *https://github.com/kubernetes/client-go/blob/c6f8cf2c47d21d55fa0df928291b2580544886c8/transport/round_trippers.go#L223*

[20]命令行参数: *https://kubernetes.io/docs/admin/kube-apiserver/*

[21]x509 handler: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/staging/src/k8s.io/apiserver/pkg/authentication/request/x509/x509.go#L60*

[22]bearer token handler: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/staging/src/k8s.io/apiserver/pkg/authentication/request/bearertoken/bearertoken.go#L38*

[23]basicauth handler: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/staging/src/k8s.io/apiserver/plugin/pkg/authenticator/request/basicauth/basicauth.go#L37*

[24]加上用户信息: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/staging/src/k8s.io/apiserver/pkg/endpoints/filters/authentication.go#L71-L75*

[25]进一步处理: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/staging/src/k8s.io/apiserver/pkg/endpoints/filters/authorization.go#L60*

[26]webhook: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/staging/src/k8s.io/apiserver/plugin/pkg/authorizer/webhook/webhook.go#L143*

[27]ABAC: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/pkg/auth/authorizer/abac/abac.go#L223*

[28]RBAC: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/plugin/pkg/auth/authorizer/rbac/rbac.go#L43*

[29]Node: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/plugin/pkg/auth/authorizer/node/node_authorizer.go#L67*

[30]admission controllers: *https://kubernetes.io/docs/admin/admission-controllers/#what-are-they*

[31]`plugin/pkg/admission` 目录: *https://github.com/kubernetes/kubernetes/tree/master/plugin/pkg/admission*

[32]POST handler: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/staging/src/k8s.io/apiserver/pkg/endpoints/installer.go#L815*

[33]分派给相应的 handler: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/staging/src/k8s.io/apiserver/pkg/server/handler.go#L136*

[34]path-based handler: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/staging/src/k8s.io/apiserver/pkg/server/mux/pathrecorder.go#L146*

[35]`createHandler`: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/staging/src/k8s.io/apiserver/pkg/endpoints/handlers/create.go#L37*

[36]写到 etcd: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/staging/src/k8s.io/apiserver/pkg/endpoints/handlers/create.go#L401*

[37]storage provider: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/staging/src/k8s.io/apiserver/pkg/registry/generic/registry/store.go#L362*

[38]生成的: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/staging/src/k8s.io/apiserver/pkg/endpoints/handlers/create.go#L131-L142*

[39]initializers: *https://kubernetes.io/docs/admin/extensible-admission-controllers/#initializers*

[40]注册的 addDeployment() 回调函数: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/pkg/controller/deployment/deployment_controller.go#L122*

[41]批处理的: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/pkg/controller/replicaset/replica_set.go#L487*

[42]这篇博客: *http://borismattijssen.github.io/articles/kubernetes-informers-controllers-reflectors-stores*

[43]过滤 PodSpect 中 NodeName 字段为空的 pods: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/plugin/pkg/scheduler/factory/factory.go#L190*

[44]过滤: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/plugin/pkg/scheduler/core/generic_scheduler.go#L117*

[45]创建一个 Binding 对象: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/plugin/pkg/scheduler/scheduler.go#L336-L342*

[46]过滤: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/pkg/kubelet/config/apiserver.go#L32*

[47]AppArmor profiles and `NO_NEW_PRIVS`: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/pkg/kubelet/kubelet.go#L883-L884*

[48]等待: *https://github.com/kubernetes/kubernetes/blob/2723e06a251a4ec3ef241397217e73fa782b0b98/pkg/kubelet/volumemanager/volume_manager.go#L330*

[49]生成一个 v1.PodStatus: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/pkg/kubelet/kubelet_pods.go#L1287*

[50]将变为: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/pkg/kubelet/kubelet_pods.go#L1293-L1297*

[51]waiting: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/pkg/kubelet/kubelet_pods.go#L1244*

[52]`Pending`: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/pkg/kubelet/kubelet_pods.go#L1258-L1261*

[53]`PodReady` 字段置为 False: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/pkg/kubelet/status/generate.go#L70-L81*

[54]protocol buffers: *https://github.com/google/protobuf*

[55]发起 `RunPodSandbox`: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/pkg/kubelet/kuberuntime/kuberuntime_sandbox.go#L51*

[56]这篇博客: *https://www.ianlewis.org/en/almighty-pause-container*

[57]What even is a Container?: *https://jvns.ca/blog/2016/10/10/what-even-is-a-container/*

[58]github.com/containernetworking/cni: *https://github.com/containernetworking/cni*

[59]github.com/containernetworking/plugins: *https://github.com/containernetworking/plugins*

[60]plugins/main/bridge/bridge.go: *https://github.com/containernetworking/plugins/blob/v0.9.1/plugins/main/bridge/bridge.go*

[61]plugins/ipam/host-local: *https://github.com/containernetworking/plugins/tree/v0.9.1/plugins/ipam/host-local*

[62]plugins/test/noop/main.go: *https://github.com/containernetworking/cni/blob/v0.8.1/plugins/test/noop/main.go#L184*

[63]代码: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/pkg/kubelet/kuberuntime/kuberuntime_manager.go#L690*

[64]拉镜像: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/pkg/kubelet/kuberuntime/kuberuntime_container.go#L140*

[65]创建 container: *https://github.com/kubernetes/kubernetes/blob/v1.21.0/pkg/kubelet/kuberuntime/kuberuntime_container.go#L179*





## [译]数据包在 Kubernetes 中的一生（1）

即使是对于具备一定虚拟网络和路由知识的人来说，Kubernetes 集群的网络也是个颇为麻烦的事情。本文尝试帮助读者理解 Kubernetes 网络的基础知识。初期目标是根据一个发往 Kubernetes 集群 Service 的 HTTP 请求的路线，来理解 Kubernetes 网络的复杂性。这中间会涉及到命名空间、CNI 以及 Calico。第一篇会从 Linux 网络开始，后续章节会涉及到其他主题。

### Linux 命名空间

Linux 命名空间包含了现代容器中的一些基础技术。从高层来看，这一技术允许把系统资源在进程之间进行隔离。例如 PID 命名空间会把进程 ID 空间进行隔离，这样同一个主机之中的两个进程就能隔离了。

这个级别的隔离对容器世界来说是很重要的。没有命名空间的话，A 容器中的进程可能会卸载 B 容器中的文件系统，或者修改 C 容器的主机名，又或删除 D 容器的网卡。将这些资源纳入命名空间进行管理，A 容器甚至无法感知 B、C、D 容器的存在。

1. Mount：

   隔离文件系统加载点；

2. UTS：

   隔离主机名和域名；

3. IPC：

   隔离跨进程通信（IPC）资源；

4. PID：

   隔离 PID 空间；

5. 网络：

   隔离网络接口；

6. 用户：

   隔离 UID/GID 空间；

7. Cgroup：

   隔离 cgroup 根目录。

绝大多数容器会使用上述命名空间在容器进程之间进行隔离。要注意 cgroup 命名空间出现较晚，相对其它命名空间来说，用的比较少。

### 容器网络（网络命名空间）

在进入 CNI 和 Docker 之前，首先看看容器网络的核心技术。Linux 内核有不少多租户方面的功能。命名空间对不同种类的资源进行了隔离，网络命名空间隔离的自然就是网络。

在主流 Linux 操作系统中都可以简单地用 `ip` 命令创建网络命名空间。接下来创建两个分别用于服务器和客户端的网络命名空间。

```sh
$ ip netns add client
$ ip netns add server
$ ip netns list
server
client
```

![图片](D:\学习资料\笔记\k8s\k8s图\101)

创建一对 `veth` 将命名空间进行连接，可以把 `veth` 想象为连接两端的网线。

```sh
$ ip link add veth-client type veth peer name veth-server
$ ip link list | grep veth
4: veth-server@veth-client: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
5: veth-client@veth-server: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
```

![图片](D:\学习资料\笔记\k8s\k8s图\102)

这一对 `veth` 是存在于主机的网络命名空间的，接下来我们把两端分别置入各自的命名空间：

```sh
$ ip link set veth-client netns client
$ ip link set veth-server netns server
$ ip link list | grep veth # doesn’t exist on the host network namespace now
```

![图片](D:\学习资料\笔记\k8s\k8s图\103)

从 `client` 命名空间检查一下命名空间中的 `veth` 状况：

```sh
$ ip netns exec client ip link
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
5: veth-client@if4: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether ca:e8:30:2e:f9:d2 brd ff:ff:ff:ff:ff:ff link-netnsid 1
```

然后是 `server` 命名空间：

```sh
$ ip netns exec server ip link
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
4: veth-server@if5: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 42:96:f0:ae:f0:c5 brd ff:ff:ff:ff:ff:ff link-netnsid 0
```

接下来给这些网络接口分配 IP 地址并启用：

```sh
$ ip netns exec client ip address add 10.0.0.11/24 dev veth-client
$ ip netns exec client ip link set veth-client up
$ ip netns exec server ip address add 10.0.0.12/24 dev veth-server
$ ip netns exec server ip link set veth-server up
$
$ ip netns exec client ip addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
5: veth-client@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether ca:e8:30:2e:f9:d2 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet 10.0.0.11/24 scope global veth-client
       valid_lft forever preferred_lft forever
    inet6 fe80::c8e8:30ff:fe2e:f9d2/64 scope link
       valid_lft forever preferred_lft forever
$
$ ip netns exec server ip addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
4: veth-server@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 42:96:f0:ae:f0:c5 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.0.0.12/24 scope global veth-server
       valid_lft forever preferred_lft forever
    inet6 fe80::4096:f0ff:feae:f0c5/64 scope link
       valid_lft forever preferred_lft forever
```

![图片](D:\学习资料\笔记\k8s\k8s图\104)

在 `client` 命名空间中使用 `ping` 命令检查一下两个网络命名空间的连接状况：

```sh
$ ip netns exec client ping 10.0.0.12
PING 10.0.0.12 (10.0.0.12) 56(84) bytes of data.
64 bytes from 10.0.0.12: icmp_seq=1 ttl=64 time=0.101 ms
64 bytes from 10.0.0.12: icmp_seq=2 ttl=64 time=0.072 ms
64 bytes from 10.0.0.12: icmp_seq=3 ttl=64 time=0.084 ms
64 bytes from 10.0.0.12: icmp_seq=4 ttl=64 time=0.077 ms
64 bytes from 10.0.0.12: icmp_seq=5 ttl=64 time=0.079 ms
```

如果要创建更网络命名空间并互相连接，用 `veth` 对将这些网络命名空间进行两两连接就很麻烦了。可以创建创建一个 Linux 网桥来连接这些网络命名空间。Docker 就是这样为同一主机内的容器进行连接的。

下面就创建网络命名空间并用网桥连接起来：

```sh
# All in one
BR=bridge1
HOST_IP=172.17.0.33
ip link add client1-veth type veth peer name client1-veth-br
ip link add server1-veth type veth peer name server1-veth-br
ip link add $BR type bridge
ip netns add client1
ip netns add server1
ip link set client1-veth netns client1
ip link set server1-veth netns server1
ip link set client1-veth-br master $BR
ip link set server1-veth-br master $BR
ip link set $BR up
ip link set client1-veth-br up
ip link set server1-veth-br up
ip netns exec client1 ip link set client1-veth up
ip netns exec server1 ip link set server1-veth up
ip netns exec client1 ip addr add 172.30.0.11/24 dev client1-veth
ip netns exec server1 ip addr add 172.30.0.12/24 dev server1-veth
ip netns exec client1 ping 172.30.0.12 -c 5
ip addr add 172.30.0.1/24 dev $BR
ip netns exec client1 ping 172.30.0.12 -c 5
ip netns exec client1 ping 172.30.0.1 -c 5
```

![图片](D:\学习资料\笔记\k8s\k8s图\105)

还是用 `ping` 命令检查两个网络命名空间的连接性：

```sh
$ ip netns exec client1 ping 172.30.0.12 -c 5
PING 172.30.0.12 (172.30.0.12) 56(84) bytes of data.
64 bytes from 172.30.0.12: icmp_seq=1 ttl=64 time=0.138 ms
64 bytes from 172.30.0.12: icmp_seq=2 ttl=64 time=0.091 ms
64 bytes from 172.30.0.12: icmp_seq=3 ttl=64 time=0.073 ms
64 bytes from 172.30.0.12: icmp_seq=4 ttl=64 time=0.070 ms
64 bytes from 172.30.0.12: icmp_seq=5 ttl=64 time=0.107 ms
```

从命名空间中 `ping` 一下主机 IP：

```sh
$ ip netns exec client1 ping $HOST_IP -c 2
connect: Network is unreachable
```

`Network is unreachable` 的原因是路由不通，加入一条缺省路由：

```sh
$ ip netns exec client1 ip route add default via 172.30.0.1
$ ip netns exec server1 ip route add default via 172.30.0.1
$ ip netns exec client1 ping $HOST_IP -c 5
PING 172.17.0.23 (172.17.0.23) 56(84) bytes of data.
64 bytes from 172.17.0.23: icmp_seq=1 ttl=64 time=0.053 ms
64 bytes from 172.17.0.23: icmp_seq=2 ttl=64 time=0.121 ms
64 bytes from 172.17.0.23: icmp_seq=3 ttl=64 time=0.078 ms
64 bytes from 172.17.0.23: icmp_seq=4 ttl=64 time=0.129 ms
64 bytes from 172.17.0.23: icmp_seq=5 ttl=64 time=0.119 ms
--- 172.17.0.23 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 3999ms
rtt min/avg/max/mdev = 0.053/0.100/0.129/0.029 ms
```

`default` 路由打通了网桥的通信，这样这个命名空间就能和外部网络进行通信了：

```sh
$ ping 8.8.8.8 -c 2
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=117 time=3.40 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=117 time=3.81 ms
--- 8.8.8.8 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 3.403/3.610/3.817/0.207 ms
```

### 从外部服务器连接内网

如你所见，这里演示用的机器已经安装了 Docker，也就是说已经创建了 `docker0` 网桥。测试场景需要所有网络命名空间的协同，进行 Web Server 的测试有些复杂，因此这里就借用一下 `docker0`：

```sh
docker0   Link encap:Ethernet  HWaddr 02:42:e2:44:07:39
          inet addr:172.18.0.1  Bcast:172.18.0.255  Mask:255.255.255.0
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

运行一个 nginx 容器并进行观察：

```sh
$ docker run -d --name web --rm nginx
efff2d2c98f94671f69cddc5cc88bb7a0a5a2ea15dc3c98d911e39bf2764a556
$ WEB_IP=`docker inspect -f "{{ .NetworkSettings.IPAddress }}" web`
$ docker inspect web --format '{{ .NetworkSettings.SandboxKey }}'
/var/run/docker/netns/c009f2a4be71
```

Docker 创建的 `netns` 没有保存在缺省位置，所以 `ip netns list` 是看不到这个网络命名空间的。我们可以在缺省位置创建一个符号链接：

```sh
$ container_id=web
$ container_netns=$(docker inspect ${container_id} --format '{{ .NetworkSettings.SandboxKey }}')
$ mkdir -p /var/run/netns
$ rm -f /var/run/netns/${container_id}
$ ln -sv ${container_netns} /var/run/netns/${container_id}
'/var/run/netns/web' -> '/var/run/docker/netns/c009f2a4be71'
$ ip netns list
web (id: 3)
server1 (id: 1)
client1 (id: 0)
```

看看 `web` 命名空间的 IP 地址：

```sh
$ ip netns exec web ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
11: eth0@if12: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:12:00:03 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.18.0.3/24 brd 172.18.0.255 scope global eth0
       valid_lft forever preferred_lft forever
```

然后看看容器里的 IP 地址：

```sh
$ WEB_IP=`docker inspect -f "{{ .NetworkSettings.IPAddress }}" web`
$ echo $WEB_IP
172.18.0.3
```

从主机访问一下 `web` 命名空间的服务：

```sh
$ curl $WEB_IP
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

加入端口转发规则，其它主机就能访问这个 nginx 了：

```sh
$ iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination $WEB_IP:80
$ echo $HOST_IP
172.17.0.23
```

使用主机 IP 访问 Nginx：

```sh
$ curl 172.17.0.23
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>
<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>
<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

![图片](D:\学习资料\笔记\k8s\k8s图\106)

CNI 插件会执行上面的过程（不完全相同，但是类似）来设置 `loopback`、`eth0`，并给容器分配 IP。容器运行时调用 CNI 设置 Pod 网络，接下来讨论一下 CNI。

### CNI 是什么

> CNI 插件负责在容器网络命名空间中插入一个网络接口（也就是 `veth` 对中的一端）并在主机侧进行必要的变更（把 `veth` 对中的另一侧接入网桥）。然后给网络接口分配 IP，并调用 IPAM 插件来设置相应的路由。

看起来很眼熟吧？是的，我们在前面的容器网络部分已经说了这些内容。

CNI 是一个 CNCF 项目，其中包含了在 Linux 容器进行网络配置的规范和库。**CNI 的主要工作就是容器网络的连接能力，并在容器销毁时移除相应的已分配资源。**这种专注性使得 CNI 易于实现，因此被广泛接受。

![图片](D:\学习资料\笔记\k8s\k8s图\107)

> 此处所说的运行时可能是 Kubernetes、Podman 等等。

### CNI 规范

```
https://github.com/containernetworking/cni/blob/master/SPEC.md
```

在我首次阅读时，注意到了一些点：

- 因为 Docker 等运行时会为每个容器新建一个网络命名空间，所以规范把容器定义为 Linux 网络命名空间；

- CNI 的网络定义用 JSON 格式存储；

- 网络定义通过 STDIN 发送给插件；

  换句话说主机上并没有网络配置文件；

- 其他参数通过环境变量进行传递；

- CNI 插件是可执行文件；

- CNI 插件负责容器的网络；

  换句话说，它需要完成所有容器接入网络所需的工作。

  在 Docker 中会包含把容器网络命名空间连回主机的工作；

- CNI 插件负责 IPAM 工作，其中包括 IP 地址分配和路由设置。

> 接下来尝试脱离 Kubernetes 模拟创建 Pod，并使用 CNI 插件而非 CLI 命令进行 IP 分配。完成 Demo 就会更好地理解 Kubernetes 中 Pod 的本质。

第一步：下载 CNI 插件：

```sh
$ mkdir cni
$ cd cni
$ curl -O -L https://github.com/containernetworking/cni/releases/download/v0.4.0/cni-amd64-v0.4.0.tgz
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   644  100   644    0     0   1934      0 --:--:-- --:--:-- --:--:--  1933
100 15.3M  100 15.3M    0     0   233k      0  0:01:07  0:01:07 --:--:--  104k
$ tar -xvf cni-amd64-v0.4.0.tgz
./
./macvlan
./dhcp
./loopback
./ptp
./ipvlan
./bridge
./tuning
./noop
./host-local
./cnitool
./flannel
```

第二步，创建一个 JSON 格式的 CNI 配置（`00-demo.conf`）：

```json
{
    "cniVersion": "0.2.0",
    "name": "demo_br",
    "type": "bridge",
    "bridge": "cni_net0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "subnet": "10.0.10.0/24",
        "routes": [
            { "dst": "0.0.0.0/0" },
            { "dst": "1.1.1.1/32", "gw":"10.0.10.1"}
        ]    }}
```

CNI 配置参数：

```sh
-:CNI generic parameters:-
cniVersion: The version of the CNI spec in which the definition works with
name: The network name
type: The name of the plugin you wish to use.  In this case, the actual name of the plugin executable
args: Optional additional parameters
ipMasq: Configure outbound masquerade (source NAT) for this network
ipam:
    type: The name of the IPAM plugin executable
    subnet: The subnet to allocate out of (this is actually part of the IPAM plugin)
    routes:
        dst: The subnet you wish to reach
        gw: The IP address of the next hop to reach the dst.  If not specified the default gateway for the subnet is assumed
dns:
    nameservers: A list of nameservers you wish to use with this network
    domain: The search domain to use for DNS requests
    search: A list of search domains
    options: A list of options to be passed to the receiver
```

第三步：创建一个网络为 `none` 的容器，这个容器没有网络地址。可以用任意的镜像创建该容器，这里我用 `pause` 来模拟 Kubernetes：

```sh
$ docker run --name pause_demo -d --rm --network none kubernetes/pause
Unable to find image 'kubernetes/pause:latest' locally
latest: Pulling from kubernetes/pause
4f4fb700ef54: Pull complete
b9c8ec465f6b: Pull complete
Digest: sha256:b31bfb4d0213f254d361e0079deaaebefa4f82ba7aa76ef82e90b4935ad5b105
Status: Downloaded newer image for kubernetes/pause:latest
763d3ef7d3e943907a1f01f01e13c7cb6c389b1a16857141e7eac0ac10a6fe82
$ container_id=pause_demo
$ container_netns=$(docker inspect ${container_id} --format '{{ .NetworkSettings.SandboxKey }}')
$ mkdir -p /var/run/netns
$ rm -f /var/run/netns/${container_id}
$ ln -sv ${container_netns} /var/run/netns/${container_id}
'/var/run/netns/pause_demo' -> '/var/run/docker/netns/0297681f79b5'
$ ip netns list
pause_demo
$ ip netns exec $container_id ifconfig
lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

第四步：用前面的配置来调用 CNI 插件：

```sh
$ CNI_CONTAINERID=$container_id CNI_IFNAME=eth10 CNI_COMMAND=ADD CNI_NETNS=/var/run/netns/$container_id CNI_PATH=`pwd` ./bridge </tmp/00-demo.conf
2020/10/17 17:32:37 Error retriving last reserved ip: Failed to retrieve last reserved ip: open /var/lib/cni/networks/demo_br/last_reserved_ip: no such file or directory
{
    "ip4": {
        "ip": "10.0.10.2/24",
        "gateway": "10.0.10.1",
        "routes": [
            {
                "dst": "0.0.0.0/0"
            },
            {
                "dst": "1.1.1.1/32",
                "gw": "10.0.10.1"
            }
        ]
    },
    "dns": {}
```

- `CNI_COMMAND=ADD`：

  动作，可选范围包括 `ADD`、`DEL` 和 `CHECK`；

- `CNI_CONTAINER=pause_demo`：

  通知 CNI 对 `pause_demo` 网络命名空间进行操作；

- `CNI_NETNS=/var/run/netns/pause_demo`：

  命名空间所在路径；

- `CNI_IFNAME=eth10`：

  在容器端创建的网络接口名称；

- `CNI_PATH=pwd`：

  CNI 插件的可执行文件的位置，在本例中我们的当前目录已经是 `cni` 目录，因此这个环境变量设置为 `pwd`  即可.

> 强烈建议阅读 CNI 规范以获知更多 CNI 插件及其功能的信息。在同一个 JSON 文件中可以使用多个插件形成调用链，可以用于建立防火墙规则等类似操作。

第五步，运行上面的命令会返回一些内容。

首先是因为 IPAM 驱动在本地找不到保存 IP 信息的文件而报错。但是因为第一次运行插件时会创建这个文件，所以在其他命名空间再次运行这个命令就不会出现这个问题了。

其次是得到一个说明插件已经完成相应 IP 配置的 JSON 信息。在本例中，网桥的 IP 地址应该是 `10.0.10.1/24`，命名空间网络接口的地址则是 `10.0.10.2/24`。另外还会根据我们的 JSON 配置文件，加入缺省路由以及 `1.1.1.1/32` 路由。检查一下：

```sh
$ ip netns exec pause_demo ifconfig
eth10     Link encap:Ethernet  HWaddr 0a:58:0a:00:0a:02
          inet addr:10.0.10.2  Bcast:0.0.0.0  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:18 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:1476 (1.4 KB)  TX bytes:0 (0.0 B)
lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
$ ip netns exec pause_demo ip route
default via 10.0.10.1 dev eth10
1.1.1.1 via 10.0.10.1 dev eth10
10.0.10.0/24 dev eth10  proto kernel  scope link  src 10.0.10.2
```

CNI 创建了网桥并根据 JSON 信息进行了相应配置：

```sh
$ ifconfig
cni_net0  Link encap:Ethernet  HWaddr 0a:58:0a:00:0a:01
          inet addr:10.0.10.1  Bcast:0.0.0.0  Mask:255.255.255.0
          inet6 addr: fe80::c4a4:2dff:fe4b:aa1b/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:7 errors:0 dropped:0 overruns:0 frame:0
          TX packets:20 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:1174 (1.1 KB)  TX bytes:1545 (1.5 KB)
```

第六步，启动 Web Server 并共享 `pause` 容器命名空间：

```sh
$ docker run --name web_demo -d --rm --network container:$container_id nginx
8fadcf2925b779de6781b4215534b32231685b8515f998b2a66a3c7e38333e30
```

第七步，使用 `pause` 容器的 IP 地址访问 Web Server：

```sh
$ curl `cat /var/lib/cni/networks/demo_br/last_reserved_ip`
<!DOCTYPE html>
<html>
<head>
<title>Welcome to ngi,nx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
...
```

接下来看看 Pod 的定义。

### Pod 网络命名空间

接触 Kubernetes 最应该知道的一个问题就是，Pod 不等于容器，而是一组容器。这一组容器会共享同一个网络栈。每个 Pod 都会包含有 `pause` 容器，Kubernetes 通过这个容器来管理 Pod 的网络。所有其他容器都会附着在 `pause` 容器的网络命名空间中，而 `pause` 除了网络之外，再无其他作用。因此同一个 Pod 中的不同容器，可以通过 `localhost` 进行互访：

![图片](D:\学习资料\笔记\k8s\k8s图\108)



## [译]数据包在 Kubernetes 中的一生（2）

如前文所述，CNI 插件是 Kubernetes 网络的重要组件。目前有很多第三方 CNI 插件，Calico 就是其中之一，因为它的易用性和网络能力，得到很多工程师的青睐。它支持很多不同的平台，例如 Kubernetes、OpenShift、Docker EE、OpenStack 以及裸金属服务。Calico Node 组件以 Docker 容器的形式运行在 Kubernetes 的所有 Master 和 Node 节点上。Calico-CNI 插件会直接集成到 Kubernetes 每个节点的 Kubelet 进程中，一旦发现了新建的 Pod，就会将其加入 Calico 网络。

下面的内容会涉及安装、Calico 模块（Felix、BIRD 以及 Confd）和路由模式，但是不会包含网络策略方面的内容。

### CNI 的任务

1. 创建 `veth` 对，并移入容器
2. 鉴别正确的 POD CIDR
3. 创建 CNI 配置文件
4. IP 地址的分配和管理
5. 在容器中加入缺省路由
6. 把路由广播给所有 Peer 节点（不适用于 VxLan）
7. 在主机上加入路由
8. 实施网络策略

其实还有很多别的需求，但是上面几个点是最基础的。看看 Master 和 Worker 节点上的路由表。每个节点都有一个容器，容器有一个 IP 地址和缺省的容器路由。

![图片](D:\学习资料\笔记\k8s\k8s图\111)

上面的路由表说明，Pod 能够通过 3 层网络进行互通。什么模块负责添加路由，如何获取远端路由呢？为什么这里缺省网关是 `169.254.1.1` 呢？我们接下来会讨论这些问题。

Calico 的核心包括 Bird、Felix、ConfD、ETCD 以及 Kubernetes API Server。Calico 需要保存一些配置信息，例如 IP 池、端点信息、网络策略等，数据存储位置是可以配置的，本例中我们使用 Kubernetes 进行存储。

### BIRD（BGP）

Bird 是一个 BGP 守护进程，运行在每个节点上，负责相互交换路由信息。通常的拓扑关系是节点之间构成的网格：

![图片](D:\学习资料\笔记\k8s\k8s图\112)

然而集群规模较大的时候，就会很麻烦了。可以使用 Route Reflector（部分 BGP 节点能够配置为 Route Reflector）来完成路由的传播工作，从而降低 BGP 连接数量。路由广播会发送给 Route Reflector，再由 Route Reflector 进行传播，更多信息可以参考 RFC4456。

![图片](D:\学习资料\笔记\k8s\k8s图\113)

BIRD 实例负责向其它 BIRD 实例传递路由信息。缺省配置方式就是 `BGP Mesh`，适用于小规模部署。在大规模集群中，建议使用 `Route Reflector` 来克服这个缺点。可以使用多个 RR 来达成高可用目的，另外还可以使用外部 RR 来替代 BIRD。

### ConfD

ConfD 是一个简单的配置管理工具，运行在 Calico Node 容器中。它会从 ETCD 中读取数据（Calico 的 BIRD 配置），并写入磁盘文件。它会循环读取网络和子网，并应用配置数据（CIDR 键），组装为 BIRD 能够使用的配置。这样不管网络如何变化，BIRD 都能够得到通知并在节点之间广播路由。

### Felix

Calico Felix 守护进程在 Calico Node 容器中运行，完成如下功能：

- 从 Kubernetes ETCD 中读取信息
- 构建路由表
- 配置 iptables 或者 IPVS

看看集群中所有的 Calico 模块：

![图片](D:\学习资料\笔记\k8s\k8s图\114)

是不是有点不同？`veth` 的一端是“悬空”的，没有连接。

数据包如何被路由到 Peer 节点的？

1. Master 上的 Pod 尝试 Ping `10.0.2.11`
2. Pod 向网关发送一个 ARP 请求
3. 从 ARP 响应中得到 MAC 地址
4. **但是谁响应的 ARP 请求？**

容器是怎样路由到一个不存在的 IP 的？容器的缺省路由指向了 `169.254.1.1`。容器的 eth0 需要访问这个地址，因此在使用缺省路由的时候会对这个 IP 进行 ARP 查询。

如果能捕获 ARP 响应信息，会发现 `veth` 另外一侧的（`cali123`） MAC 地址。所以到底是怎样响应一个没有 IP 接口的 ARP 请求的呢？答案是 `proxy-arp`，如果我们检查一下主机侧的 `veth` 接口，会看到启用了 `proxy-arp`：

```sh
$ cat /proc/sys/net/ipv4/conf/cali123/proxy_arp
1
```

> Proxy ARP 技术能用特定网络上的代理设备来响应针对本网络不存在的 IP 地址的 ARP 查询。这个代理知道流量的目标，会以自己的 MAC 地址进行响应。如此一来，流量就转给 Proxy，通常会被 Proxy 使用其它网络接口或者隧道路由到原定目标。这种以自己 MAC 地址响应其他 IP 地址的 ARP 请求，完成代理任务的行为有时也被称为发布。

仔细看看 Worker 节点：

![图片](D:\学习资料\笔记\k8s\k8s图\115)

数据包进入内核之后，会根据路由表进行路由。

> 入栈流量：首先进入Worker 节点内核。
> 内核把数据包发给 `cali123`。

### 路由模式

Calico 支持三种路由模式，本节中会对几种模式的优劣和适用场景进行讨论。

- **IP-in-IP**：

  缺省，有封装行为；

- **Direct/NoEncapMode**：

  无封包（推荐）；

- **VxLan**：

  有封包（无 BGP）

#### IP-in-IP

这是一种简单的对 IP 包进行再封包的方式。传输中的数据包带有一个外层头部，其中描述了源主机和目的 IP，还有一个内层头部，包含源 Pod 和目标 IP。

目前 Azure 还不支持 IP-IP，因此这种环境中无法使用该模式，建议关掉 IP-IP 以提高性能。

#### NoEncapMode

这种模式下数据包是用 Pod 发出时的原始格式发出来的。因为没有封包和解包的开销，这种模式比较有性能优势。

AWS 中要使用这种模式需要关闭源 IP 校验。

#### VXLAN

Calico 3.7 以后的版本才支持 VXLAN 路由。

> VXLAN 是 Virtual Extensible LAN 的缩写。VXLAN 是一种封包技术，二层数据帧被封装为 UDP 数据包。VXLAN 是一种网络虚拟化技术。当设备在软件定义的数据中心里进行通信时，会在这些设备之间建立 VXLAN 隧道。这些隧道能建立在物理或虚拟交换机之上。这些交换端口被称为 VXLAN Tunnel Endpoints（VTEPs），负责 VXLAN 的封包和解包工作。不支持 VXLAN 的设备可以连接到 VTEP，由 VTEP 提供 VXLAN 的出入转换工作。

VXLAN 对于不支持 IP-in-IP 的网络非常有用，例如 Azure 或者其它不支持 BGP 的数据中心。

![图片](D:\学习资料\笔记\k8s\k8s图\116)

#### 演示—— IPIP 和 UnEncapMode

在没安装 Calico 之前检查一下集群：

``` sh
$ kubectl get nodes
NAME           STATUS     ROLES    AGE   VERSION
controlplane   NotReady   master   40s   v1.18.0
node01         NotReady   <none>   9s    v1.18.0

$ kubectl get pods --all-namespaces
NAMESPACE     NAME                                   READY   STATUS    RESTARTS   AGE
kube-system   coredns-66bff467f8-52tkd               0/1     Pending   0          32s
kube-system   coredns-66bff467f8-g5gjb               0/1     Pending   0          32s
kube-system   etcd-controlplane                      1/1     Running   0          34s
kube-system   kube-apiserver-controlplane            1/1     Running   0          34s
kube-system   kube-controller-manager-controlplane   1/1     Running   0          34s
kube-system   kube-proxy-b2j4x                       1/1     Running   0          13s
kube-system   kube-proxy-s46lv                       1/1     Running   0          32s
kube-system   kube-scheduler-controlplane            1/1     Running   0          33s
```

检查 CNI 的二进制文件和目录。其中没有任何配置文件或者 Calico 二进制，Calico 安装过程会用加载卷来填充其中的内容：

```sh
$ cd /etc/cni
-bash: cd: /etc/cni: No such file or directory
$ cd /opt/cni/bin
$ ls
bridge  dhcp  flannel  host-device  host-local  ipvlan  loopback  macvlan  portmap  ptp  sample  tuning  vlan
```

在 Master/Worker 节点上检查 `ip route`：

```sh
$ ip route
default via 172.17.0.1 dev ens3
172.17.0.0/16 dev ens3 proto kernel scope link src 172.17.0.32
172.18.0.0/24 dev docker0 proto kernel scope link src 172.18.0.1 linkdown
```

在集群环境中下载并提交 `calico.yaml`：

```sh
$ curl https://docs.projectcalico.org/manifests/calico.yaml -O
$ kubectl apply -f calico.yaml
```

看看其中的配置参数：

```json
cni_network_config: |-
    {
      "name": "k8s-pod-network",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "calico", >>> Calico's CNI plugin
          "log_level": "info",
          "log_file_path": "/var/log/calico/cni/cni.log",
          "datastore_type": "kubernetes",
          "nodename": "__KUBERNETES_NODE_NAME__",
          "mtu": __CNI_MTU__,
          "ipam": {
              "type": "calico-ipam" >>> Calico's IPAM instaed of default IPAM
          },
          "policy": {
              "type": "k8s"
          },
          "kubernetes": {
              "kubeconfig": "__KUBECONFIG_FILEPATH__"
          }
        },
        {
          "type": "portmap",
          "snat": true,
          "capabilities": {"portMappings": true}
        },
        {
          "type": "bandwidth",
          "capabilities": {"bandwidth": true}
        }
      ]
    }
# Enable IPIP
- name: CALICO_IPV4POOL_IPIP
    value: "Always" >> Set this to 'Never' to disable IP-IP
# Enable or Disable VXLAN on the default IP pool.
- name: CALICO_IPV4POOL_VXLAN
    value: "Never"
```

安装完毕之后，检查 Pod 和节点状态。

```sh
$ kubectl get pods --all-namespaces
NAMESPACE     NAME                                       READY   STATUS              RESTARTS   AGE
kube-system   calico-kube-controllers-799fb94867-6qj77   0/1     ContainerCreating   0          21s
kube-system   calico-node-bzttq                          0/1     PodInitializing     0          21s
kube-system   calico-node-r6bwj                          0/1     PodInitializing     0          21s
kube-system   coredns-66bff467f8-52tkd                   0/1     Pending             0          7m5s
kube-system   coredns-66bff467f8-g5gjb                   0/1     ContainerCreating   0          7m5s
kube-system   etcd-controlplane                          1/1     Running             0          7m7s
kube-system   kube-apiserver-controlplane                1/1     Running             0          7m7s
kube-system   kube-controller-manager-controlplane       1/1     Running             0          7m7s
kube-system   kube-proxy-b2j4x                           1/1     Running             0          6m46s
kube-system   kube-proxy-s46lv                           1/1     Running             0          7m5s
kube-system   kube-scheduler-controlplane                1/1     Running             0          7m6s
$ kubectl get nodes
NAME           STATUS   ROLES    AGE     VERSION
controlplane   Ready    master   7m30s   v1.18.0
node01         Ready    <none>   6m59s   v1.18.0
```

Kubelet 需要 CNI 的配置文件来设置网络：

```json
$ cd /etc/cni/net.d/
$ ls
10-calico.conflist  calico-kubeconfig
$
$
$ cat 10-calico.conflist
{
  "name": "k8s-pod-network",
  "cniVersion": "0.3.1",
  "plugins": [
    {
      "type": "calico",
      "log_level": "info",
      "log_file_path": "/var/log/calico/cni/cni.log",
      "datastore_type": "kubernetes",
      "nodename": "controlplane",
      "mtu": 1440,
      "ipam": {
          "type": "calico-ipam"
      },
      "policy": {
          "type": "k8s"
      },
      "kubernetes": {
          "kubeconfig": "/etc/cni/net.d/calico-kubeconfig"
      }
    },
    {
      "type": "portmap",
      "snat": true,
      "capabilities": {"portMappings": true}
    },
    {
      "type": "bandwidth",
      "capabilities": {"bandwidth": true}
    }
  ]
}
```

检查 CNI 的二进制文件：

```sh
$ ls
bandwidth  bridge  calico  calico-ipam dhcp  flannel  host-device  host-local  install  ipvlan  loopback  macvlan  portmap  ptp  sample  tuning  vlan
```

安装 `calicoctl` 来获取 Calico 的更多信息并能修改 Calico 配置：

```sh
$ cd /usr/local/bin/
$ curl -O -L  https://github.com/projectcalico/calicoctl/releases/download/v3.16.3/calicoctl
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   633  100   633    0     0   3087      0 --:--:-- --:--:-- --:--:--  3087
100 38.4M  100 38.4M    0     0  5072k      0  0:00:07  0:00:07 --:--:-- 4325k
$ chmod +x calicoctl
$ export DATASTORE_TYPE=kubernetes
$ export KUBECONFIG=~/.kube/config
# Check endpoints - it will be empty as we have't deployed any POD
$ calicoctl get workloadendpoints
WORKLOAD   NODE   NETWORKS   INTERFACE
```

检查 BGP Peer 的状态，会看到 Worker 节点是一个 Peer。

```sh
$ calicoctl node status
Calico process is running.
IPv4 BGP status
+--------------+-------------------+-------+----------+-------------+
| PEER ADDRESS |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+--------------+-------------------+-------+----------+-------------+
| 172.17.0.40  | node-to-node mesh | up    | 00:24:04 | Established |
+--------------+-------------------+-------+----------+-------------+
```

创建一个两副本 Pod，并设置 `tolerations`，使之可以运行在 Master 节点：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: busybox-deployment
spec:
  selector:
    matchLabels:
      app: busybox
  replicas: 2
  template:
    metadata:
      labels:
        app: busybox
    spec:
      tolerations:
      - key: "node-role.kubernetes.io/master"
        operator: "Exists"
        effect: "NoSchedule"
      containers:
      - name: busybox
        image: busybox
        command: ["sleep"]
        args: ["10000"]
```

获取 Pod 和端点状态：

```sh
$ kubectl get pods -o wide
NAME                                 READY   STATUS    RESTARTS   AGE   IP                NODE           NOMINATED NODE   READINESS GATES
busybox-deployment-8c7dc8548-btnkv   1/1     Running   0          6s    192.168.196.131   node01         <none>           <none>
busybox-deployment-8c7dc8548-x6ljh   1/1     Running   0          6s    192.168.49.66     controlplane   <none>           <none>
$ calicoctl get workloadendpoints
WORKLOAD                             NODE           NETWORKS             INTERFACE
busybox-deployment-8c7dc8548-btnkv   node01         192.168.196.131/32   calib673e730d42
busybox-deployment-8c7dc8548-x6ljh   controlplane   192.168.49.66/32     cali9861acf9f07
```

获取 Pod 所在主机上的 VETH 信息：

```sh
$ ifconfig cali9861acf9f07
cali9861acf9f07: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1440
        inet6 fe80::ecee:eeff:feee:eeee  prefixlen 64  scopeid 0x20<link>
        ether ee:ee:ee:ee:ee:ee  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 5  bytes 446 (446.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

获取 Pod 网络界面的信息：

```sh
$ kubectl exec busybox-deployment-8c7dc8548-x6ljh -- ifconfig
eth0      Link encap:Ethernet  HWaddr 92:7E:C4:15:B9:82
          inet addr:192.168.49.66  Bcast:192.168.49.66  Mask:255.255.255.255
          UP BROADCAST RUNNING MULTICAST  MTU:1440  Metric:1
          RX packets:5 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:446 (446.0 B)  TX bytes:0 (0.0 B)
lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
$ kubectl exec busybox-deployment-8c7dc8548-x6ljh -- ip route
default via 169.254.1.1 dev eth0
169.254.1.1 dev eth0 scope link
$ kubectl exec busybox-deployment-8c7dc8548-x6ljh -- arp
```

获取主节点路由：

```sh
$ ip route
default via 172.17.0.1 dev ens3
172.17.0.0/16 dev ens3 proto kernel scope link src 172.17.0.32
172.18.0.0/24 dev docker0 proto kernel scope link src 172.18.0.1 linkdown
blackhole 192.168.49.64/26 proto bird
192.168.49.65 dev calic22dbe57533 scope link
192.168.49.66 dev cali9861acf9f07 scope link
192.168.196.128/26 via 172.17.0.40 dev tunl0 proto bird onlink
```

尝试 Ping Worker 节点来触发 ARP：

```sh
$ kubectl exec busybox-deployment-8c7dc8548-x6ljh -- ping 192.168.196.131 -c 1
PING 192.168.196.131 (192.168.196.131): 56 data bytes
64 bytes from 192.168.196.131: seq=0 ttl=62 time=0.823 ms
$ kubectl exec busybox-deployment-8c7dc8548-x6ljh -- arp
? (169.254.1.1) at ee:ee:ee:ee:ee:ee [ether]  on eth0
```

注意上面的 MAC 地址。发出流量时，内核根据 IP 路由将数据包写入 `tunl0`，Proxy ARP 的配置：

```sh
$ cat /proc/sys/net/ipv4/conf/cali9861acf9f07/proxy_arp
1
```



### 目标节点如何处理数据包

```sh
node01 $ ip route
default via 172.17.0.1 dev ens3
172.17.0.0/16 dev ens3 proto kernel scope link src 172.17.0.40
172.18.0.0/24 dev docker0 proto kernel scope link src 172.18.0.1 linkdown
192.168.49.64/26 via 172.17.0.32 dev tunl0 proto bird onlink
blackhole 192.168.196.128/26 proto bird
192.168.196.129 dev calid4f00d97cb5 scope link
192.168.196.130 dev cali257578b48b6 scope link
192.168.196.131 dev calib673e730d42 scope link
```

接收到数据包之后，内核会根据路由表将数据包发给对应的 `veth`。

如果抓包的话会看出 IP-IP 协议。据我所知，Azure 不支持 IP-IP，也就是说我们无法在这种环境里使用 IP-IP。关闭 IP-IP 能获得更高性能，下面一节尝试一下。

#### 禁用 IP-IP

更新 `ippool.yaml` 设置 IPIP 为 `Never`，然后用 `calicoctl` 应用配置：

```sh
$ calicoctl get ippool default-ipv4-ippool -o yaml > ippool.yaml
$ vi ippool.yaml
...
$ calicoctl apply -f ippool.yaml
Successfully applied 1 'IPPool' resource(s)
```

再次检查 `ip route`：

```sh
$ ip route
default via 172.17.0.1 dev ens3
172.17.0.0/16 dev ens3 proto kernel scope link src 172.17.0.32
172.18.0.0/24 dev docker0 proto kernel scope link src 172.18.0.1 linkdown
blackhole 192.168.49.64/26 proto bird
192.168.49.65 dev calic22dbe57533 scope link
192.168.49.66 dev cali9861acf9f07 scope link
192.168.196.128/26 via 172.17.0.40 dev ens3 proto bird
```

设备不再是 `tunl0`，而是变成 Master 节点的管理界面（`ens3`）。

Ping 一下 Worker 节点，验证工作情况，此时不再使用 IPIP 协议：

```sh
$ kubectl exec busybox-deployment-8c7dc8548-x6ljh -- ping 192.168.196.131 -c 1
PING 192.168.196.131 (192.168.196.131): 56 data bytes
64 bytes from 192.168.196.131: seq=0 ttl=62 time=0.653 ms
--- 192.168.196.131 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 0.653/0.653/0.653 ms
```

> 注意在 AWS 环境中使用这种模式需要禁用源 IP 检查。

#### 演示 VXLAN

重新进行集群初始化，并下载 `calico.yaml` 文件，进行如下变更：

从 `livenessProbe` 和 `readinessProbe` 中删除 `bird`：

```yaml
livenessProbe:
            exec:
              command:
              - /bin/calico-node
              - -felix-live
              - -bird-live >> Remove this
            periodSeconds: 10
            initialDelaySeconds: 10
            failureThreshold: 6
          readinessProbe:
            exec:
              command:
              - /bin/calico-node
              - -felix-ready
              - -bird-ready >> Remove this
```

把 `calico_backend` 修改为 `vxlan`，不再需要 BGP：

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: calico-config
  namespace: kube-system
data:
  # Typha is disabled.
  typha_service_name: "none"
  # Configure the backend to use.
  calico_backend: "vxlan"
```

禁用 IPIP：

```yaml
# Enable IPIP
- name: CALICO_IPV4POOL_IPIP
    value: "Never" >> Set this to 'Never' to disable IP-IP
# Enable or Disable VXLAN on the default IP pool.
- name: CALICO_IPV4POOL_VXLAN
    value: "Never"
```

应用这个 YAML：

```sh
$ ip route
default via 172.17.0.1 dev ens3
172.17.0.0/16 dev ens3 proto kernel scope link src 172.17.0.15
172.18.0.0/24 dev docker0 proto kernel scope link src 172.18.0.1 linkdown
192.168.49.65 dev calif5cc38277c7 scope link
192.168.49.66 dev cali840c047460a scope link
192.168.196.128/26 via 192.168.196.128 dev vxlan.calico onlink
vxlan.calico: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1440
        inet 192.168.196.128  netmask 255.255.255.255  broadcast 192.168.196.128
        inet6 fe80::64aa:99ff:fe2f:dc24  prefixlen 64  scopeid 0x20<link>
        ether 66:aa:99:2f:dc:24  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 11 overruns 0  carrier 0  collisions 0
```

获取 Pod 状态：

```sh
$ kubectl get pods -o wide
NAME                                 READY   STATUS    RESTARTS   AGE   IP                NODE           NOMINATED NODE   READINESS GATES
busybox-deployment-8c7dc8548-8bxnw   1/1     Running   0          11s   192.168.49.67     controlplane   <none>           <none>
busybox-deployment-8c7dc8548-kmxst   1/1     Running   0          11s   192.168.196.130   node01         <none>           <none>
```

查看 `ip route`：

```sh
$ kubectl exec busybox-deployment-8c7dc8548-8bxnw -- ip route
default via 169.254.1.1 dev eth0
169.254.1.1 dev eth0 scope link
```

执行 Ping，触发 ARP 查询：

```sh
$ kubectl exec busybox-deployment-8c7dc8548-8bxnw -- arp
master $ kubectl exec busybox-deployment-8c7dc8548-8bxnw -- ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: seq=0 ttl=116 time=3.786 ms
^C
$ kubectl exec busybox-deployment-8c7dc8548-8bxnw -- arp
? (169.254.1.1) at ee:ee:ee:ee:ee:ee [ether]  on eth0
```

概念和前一种模式相似，区别在于数据包抵达 `vxland` 的时候，会把节点 IP 以及 MAC 地址封装并发送。另外 `vxland` 的 UDP 端口是 4789。这里会从 etcd 获取可用节点以及节点支持的 IP 范围，从而让 `vxlan-calico` 据此构建数据包。

> VxLan 模式需要更多系统开销

![图片](D:\学习资料\笔记\k8s\k8s图\117)



## [译]数据包在 Kubernetes 中的一生（3）

本章我们会讨论一下 Kubernetes 的 `kube-proxy` 是如何使用 `iptables` 控制流量的。注意，`kube-proxy` + `iptables` 的组合并非完成该任务的唯一选择。

我们会从 Kubernetes 的多种通信模型和实现开始，如果读者已经了解了 Service、ClusterIP 以及 NodePort 的概念，可以直接跳到 `kube-proxy/iptables` 一节。



### Pod 到 Pod

CNI 会配置节点和 Pod 的路由，`kube-proxy` 不会介入 Pod 到 Pod 之间的通信过程。所有的容器都无需 NAT 就能互相通信；节点和容器之间的通信也是无需 NAT 的。

Pod 的 IP 地址是不固定的（也有办法做成静态 IP，但是缺省配置是不提供这种保障的）。在 Pod 重启时 CNI 会给他分配新的 IP 地址，CNI 不负责维护 IP 地址和 Pod 的映射。Pod 名称在 Deployment 之中也是不固定的。

![图片](D:\学习资料\笔记\k8s\k8s图\118)

Deployment 中的 Pod 是无状态的，一个应用可能会有多个 Pod 副本，因此需要一个负载均衡之类的东西来负责对外开放服务，Kubernetes 中的 Service 对象负责完成这个任务。



### Pod 到外部

Kubernetes 会使用 SNAT 完成从 Pod 向外发出的访问。SNAT 会将 Pod 的内部 `IP:Port` 替换为主机的 `IP:Port`。返回数据包到达节点时，`IP:Port` 又会换回 Pod。这个过程对于原始 Pod 是透明无感知的。



### Pod 到 Service

#### Cluster IP

Kubernetes 有一个叫做 `Service` 的对象，是一个通向 Pod 的 4 层负载均衡。`Service` 对象有很多类型，最基本的类型叫做 `ClusterIP`，这种类型的 Service 有一个唯一的 VIP 地址，其路由范围仅在集群内部有效。

Kubernetes 集群中，Pod 可能发生移动、重启、升级或者扩缩容，因此向应用 Pod 发送流量是有困难的，另外应用通常有多个副本，我们需要一些方法来进行负载均衡。

Kubernetes 使用 `Service` 对象来解决这个问题。Service 是一个 API 对象，它用一个虚拟 IP 映射到一组 Pod。另外 Kubernetes 为每个 Service 的名称及其虚拟 IP 建立了 DNS 记录，因此可以轻松地根据名称进行寻址。

虚拟 IP 到 Pod IP 的转换是通过每个节点上的 `kube-proxy` 完成的。在 Pod 向外发起通信时，这个进程会通过 iptables 或者 IPVS 自动把 VIP 转为 Pod IP，每个连接都有跟踪，所以数据包返回时候，地址还能够被正确地转回原样。IPVS 和 iptables 在 VIP 和 Pod IP 之间承担着负载均衡的角色，IPVS 能够提供更多的负载均衡算法。虚拟 IP 并不存在于网络接口上，而是在 iptable 中：

![图片](D:\学习资料\笔记\k8s\k8s图\119)

FrontEnd Deployment：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  labels:
    app: webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
```

Backend Deployment：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth
  labels:
    app: auth
spec:
  replicas: 2
  selector:
    matchLabels:
      app: auth
  template:
    metadata:
      labels:
        app: auth
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
```

Service：

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  labels:
    app: frontend
spec:
  ports:
  - port: 80
    protocol: TCP
  type: ClusterIP
  selector:
    app: webapp
---
apiVersion: v1
kind: Service
metadata:
  name: backend
  labels:
    app: backend
spec:
  ports:
  - port: 80
    protocol: TCP
  type: ClusterIP
  selector:
    app: auth
...
```

现在 FrontEnd Pod 能够通过 ClusterIP 或者 DNS 名称来访问 Backend 了。CoreDNS 这样的 DNS 服务器具备 Kubernetes 集群感知的能力，他们会对 Kubernetes API 进行监控，一旦新建了 Service，就会新建对应的 DNS 记录。如果集群中启用的 DNS，所有 Pod 都能够自动的根据 DNS 名称来解析到 Service。

![图片](D:\学习资料\笔记\k8s\k8s图\120)



#### NodePort（外部到 Pod）

在集群内部可以用 DNS 访问 Service。然而 Service 的 IP 是私有的和虚拟的，所以集群外是无法访问的。

试试看从外部访问 frontEnd 的 Pod（此时还没有给 frontEnd 创建 Service）：

![图片](D:\学习资料\笔记\k8s\k8s图\121)

Pod IP 是私有的，无法路由。

接下来创建一个 NodePort 类型的 Service 把 FrontEnd 服务开放给外部世界。如果把 `type` 字段设置为 `NodePort`，Kubernetes 控制面使用 `--service-node-port-range` 参数为 NodePort 服务分配了一个端口范围。每个节点都会会把这个端口映射给特定的服务。Service 使用 `.spec.ports[*].nodePort` 字段来指定该端口：

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  type: NodePort
  selector:
    app: webapp
  ports:
      # By default and for convenience, the `targetPort` is set to the same value as the `port` field.
    - port: 80
      targetPort: 80
      # Optional field
      # By default and for convenience, the Kubernetes control plane will allocate a port from a range (default: 30000-32767)
      nodePort: 31380
...
```

![图片](D:\学习资料\笔记\k8s\k8s图\122)

这样就可以在集群外使用任意节点的 nodePort 来访问服务了。还可以给 `nodePort` 赋值以指定特定开放端口。这种情况下，为了防止端口冲突，需要自行管理端口，并且指定端口也必须在参数中声明的端口范围之内。

#### ExternalTrafficPolicy

`ExternalTrafficPolicy` 字段表明所属 Service 对象会把来自外部的流量路由给本节点还是集群范围内的端点。如果赋值为 `Local`，会保留客户端源 IP 同时避免 `NodePort` 类型服务的多余一跳，但是有流量分配不均匀的隐患；如果设置为 `Cluster`，会抹掉客户端的源 IP，并导致到其它节点的一跳，但会获得相对较好的均衡效果。

##### Cluster

这是 Kubernetes Service 的缺省 `ExternalTrafficPolicy`。这个选项会把流量平均分配给该 Service 的所有 Pod 上。

这种策略的一个弱点是会存在不必要的节点间网络跳转。例如在一个节点的 NodePort 上接收到流量时，即使本节点上存在可用 Pod，流量还是可能会随机地把流量路由到另外一个节点上的 Pod，造成不必要的跳转。

在 `Cluster` 策略下，数据包的流向：

- 客户端把数据包发送给 `node2:31380`；
- `node2` 替换源 IP 地址（SNAT）为自己的 IP 地址；
- `node2` 将目标地址替换为 Pod IP；
- 数据包被路由到 `node1` 或者 `node3`，然后到达 Pod；
- Pod 的响应返回到 `node2`；
- Pod 的响应返回到客户端。

![图片](D:\学习资料\笔记\k8s\k8s图\123)

##### Local

这种策略中，`kube-proxy` **只会在存在目标 Pod 的节点上**加入 NodePort 的代理规则。API Server 要求只有使用 `LoadBalancer` 或者 `NodePort` 类型的 Service 才能够使用这种策略。这是因为 `Local` 策略只跟外部访问相关。

如果使用了 `Local` 策略，`kube-proxy` 只会代理到本地 `endpoint` 的流量，不会向其它节点转发。如果本地没有相应端点，发送到该节点的流量就会被丢弃，所以数据包中会保留正确的源 IP，可以放心的在数据包处理规则中使用。

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  type: NodePort
  externalTrafficPolicy: Local
  selector:
    app: webapp
  ports:
      # By default and for convenience, the `targetPort` is set to the same value as the `port` field.
    - port: 80
      targetPort: 80
      # Optional field
      # By default and for convenience, the Kubernetes control plane will allocate a port from a range (default: 30000-32767)
      nodePort: 31380
...
```

`Local` 策略下的数据包：

- 客户端发送数据包到 `node1:31380`，这个端点上存在目标 Pod；
- `node1` 把数据包路由到端点，其中带有正确的源 IP；
- 因为策略限制，`node1` 不会把数据包发给 `node3`；
- 客户端发送数据包给 `node2:31380`，该节点上不存在目标 Pod；
- 数据包被丢弃。

![图片](D:\学习资料\笔记\k8s\k8s图\124)

##### LoadBalancer Service 类型中的 Local 策略

如果在 Google GKE 上使用 `Local` 策略，由于健康检查的原因，会把不运行对应 Pod 的节点从负载均衡池中剔除，所以不会发生丢弃流量的问题。这种模型对于需要处理大量外部入栈流量，需要避免跨节点跳转从而降低延迟的应用非常有帮助。另外因为不需要进行 SNAT，从而让源 IP 得以保存。然而官方文档声明，这种策略存在不够均衡的短板。

![图片](D:\学习资料\笔记\k8s\k8s图\125)



### Kube-Proxy（iptable）

Kubernetes 中负责 Service 对象的组件就是 `kube-proxy`。它在每个节点上运行，为 Pod 和 Service 生成复杂的 iptables 规则，完成所有的过滤和 NAT 工作。如果登录到 Kubernetes 节点上，运行 `iptables-save`，会看到 Kubernetes 或者其它组件生成的规则。最重要的是 `KUBE-SERVICE`、`KUBE-SVC-*` 以及 `KUBE-SEP-*`：

- `KUBE-SERVICE` 是 `Service` 包的入口。

  它负责匹配 IP:Port，并把数据包发给对应的 `KUBE-SVC-*`。

- `KUBE-SVC-*` 担任负载均衡的角色，会平均分配数据包到 `KUBE-SEP-*`。

  每个 `KUBE-SVC-*` 都有和 `Endpoint` 同样数量的 `KUBE-SEP-*`。

- `KUBE-SEP-*` 代表的是 `Service` 的 `EndPoint`，它负责的是 DNAT，会把 `Service` 的 IP:Port 替换为 Pod 的 IP:Port。

Conntrack 会介入 DNAT 过程，使用状态机来跟踪连接状态。为了记住目标地址的变更，并在回包时候进行恢复，这些状态是必须保存的。iptables 还可以根据 conntrack 状态（ctstate）来决定数据包的目标。下面四个 conntrack 状态尤其重要：

- `NEW`：

  conntrack 对该数据包一无所知，该状态出现在收到 `SYN` 的时候。

- `ESTABLISHED`：

  conntrack 知道该数据包属于一个已发布连接，该状态出现于握手完成之后。

- `RELATED`：

  这个数据包不属于任何连接，但是他是隶属于其它连接的，在 FTP 之类的协议里常用。

- `INVALID`：

  有问题的数据包，conntrack 不知道如何处理。

  这种状态是 Kubernetes 问题的常客。

Service 和 Pod 之间的 TCP 连接过程如下：

- 左侧的客户端 Pod 发送数据包到一个 Service：

  `2.2.2.10:80`；

- 数据包经过客户端节点的 iptables 规则，目标改为 `1.1.1.20:80`；

- 服务端 Pod 处理数据包，发送一个响应包到 `1.1.1.20`；

- 数据包回到客户端节点，conntrack 认出这个数据包，把源地址改回 `2.2.2.10:80`；

- 客户端 Pod 收到响应包。



### iptables

在 Linux 操作系统中使用 `netfilter` 处理防火墙工作。这是一个内核模块，决定是否放行数据包。iptables 是 netfilter 的前端。二者经常被混为一谈。

#### 链

每条链负责一种特定任务。

- `PREROUTING`：

  决定数据包刚刚进入网络端口时的对策。

  有几种不同的选择，例如修改数据包（NAT），丢弃数据包或者什么都不做使其通过；

- `INPUT`：

  其中经常包含一些用于防止恶意行为的严格规则，防止系统遭到入侵。

  开放或者屏蔽端口的行为就是在这里进行的；

- `FORWARD`：

  顾名思义，负责数据包的转发。

  在将服务器作为路由器的时候，就需要在这里完成任务。

- `OUTPUT`：

  这里负责所有的网络浏览的行为。

  这里可以限制所有数据包的发送。

- `POSTROUTING`：

  发生在数据包离开服务器之前，数据包最后的可跟踪位置。

![图片](D:\学习资料\笔记\k8s\k8s图\126)

`FORWARD` 仅在 `ip_forward` 启用时才有效。所以下面的命令在 Kubernetes 中很重要：

```sh
$ sysctl -w net.ipv4.ip_forward=1
net.ipv4.ip_forward = 1
$ cat /proc/sys/net/ipv4/ip_forward
1
```

上面的变更是暂时性的，要持久化这个变更，需要在 `/etc/sysctl.conf` 中写入 `net.ipv4.ip_forward = 1`。

#### 表

接下来会讨论 NAT 表，除此之外还有几个：

- `Filter`：

  缺省表，这里决定是否允许数据包出入本机，因此可以在这里进行屏蔽等操作；

- `Nat`：

  是网络地址转换的缩写。

  下面会有例子说明；

- `Mangle`：

  仅对特定包有用。

  它的功能是在包出入之前修改包中的内容；

- `RAW`：

  用于处理原始数据包，主要用在跟踪连接状态，下面有一个放行 SSH 连接的例子。

- `Security`：

  负责在 Filter 之后保障安全。

#### Kubernetes 中的 iptables 配置

部署一个 2 副本 Nginx 应用，导出 iptables 规则。

> 服务类型 NodePort

```sh
$ kubectl get svc webapp
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
webapp NodePort 10.103.46.104 <none> 80:31380/TCP 3d13h
$ kubectl get ep webapp 
NAME ENDPOINTS AGE
webapp 10.244.120.102:80,10.244.120.103:80 3d13h
```

ClusterIP 是一个存在于 iptables 中的虚拟 IP，Kubernetes 会把这个地址存在 CoreDNS 中。

```sh
$ kubectl exec -i -t dnsutils -- nslookup webapp.default
Server:  10.96.0.10
Address: 10.96.0.10#53
Name: webapp.default.svc.cluster.local
Address: 10.103.46.104
```

为了能够进行包过滤和 NAT，Kubernetes 会创建一个 `KUBE-SERVICES` 链，把所有 `PREROUTING` 和 `OUTPUT` 流量转发给 `KUBE-SERVICES`：

```sh
sudo iptables -t nat -L PREROUTING | column -t
Chain            PREROUTING  (policy  ACCEPT)                                                                    
target           prot        opt      source    destination                                                      
cali-PREROUTING  all         --       anywhere  anywhere     /*        cali:6gwbT8clXdHdC1b1  */                 
KUBE-SERVICES    all         --       anywhere  anywhere     /*        kubernetes             service   portals  */
DOCKER           all         --       anywhere  anywhere     ADDRTYPE  match                  dst-type  LOCAL
```

使用 `KUBE-SERVICES` 介入包过滤和 NAT 之后，Kubernetes 会监控通向 Service 的流量，并进行 SNAT/DNAT 的处理。在 `KUBE-SERVICES` 链尾部，会写入另一个链 `KUBE-SERVICES`，用于处理 `NodePort` 类型的 Service。

`KUBE-SVC-2IRACUALRELARSND` 链会处理针对 `ClusterIP` 的流量，否则的话就会进入 `KUBE-NODEPORTS`：

```sh
$ sudo iptables -t nat -L KUBE-SERVICES | column -t
Chain                      KUBE-SERVICES  (2   references)                                                                                                                                                                             
target                     prot           opt  source          destination                                                                                                                                                             
KUBE-MARK-MASQ             tcp            --   !10.244.0.0/16  10.103.46.104   /*  default/webapp                   cluster  IP          */     tcp   dpt:www                                                                          
KUBE-SVC-2IRACUALRELARSND  tcp            --   anywhere        10.103.46.104   /*  default/webapp                   cluster  IP          */     tcp   dpt:www                                                                                                                                             
KUBE-NODEPORTS             all            --   anywhere        anywhere        /*  kubernetes                       service  nodeports;  NOTE:  this  must        be  the  last  rule  in  this  chain  */  ADDRTYPE  match  dst-type  LOCAL
```

看看 `KUBE-NODEPORTS` 的内容：

```sh
$ sudo iptables -t nat -L KUBE-NODEPORTS | column -t
Chain                      KUBE-NODEPORTS  (1   references)                                            
target                     prot            opt  source       destination                               
KUBE-MARK-MASQ             tcp             --   anywhere     anywhere     /*  default/webapp  */  tcp  dpt:31380
KUBE-SVC-2IRACUALRELARSND  tcp             --   anywhere     anywhere     /*  default/webapp  */  tcp  dpt:31380
```

看起来 `ClusterIP` 和 `NodePort` 处理过程是一样的，那么看看下面的处理流程：

```sh
# statistic  mode  random -> Random load-balancing between endpoints.
$ sudo iptables -t nat -L KUBE-SVC-2IRACUALRELARSND | column -t
Chain                      KUBE-SVC-2IRACUALRELARSND  (2   references)                                                                             
target                     prot                       opt  source       destination                                                                
KUBE-SEP-AO6KYGU752IZFEZ4  all                        --   anywhere     anywhere     /*  default/webapp  */  statistic  mode  random  probability  0.50000000000
KUBE-SEP-PJFBSHHDX4VZAOXM  all                        --   anywhere     anywhere     /*  default/webapp  */

$ sudo iptables -t nat -L KUBE-SEP-AO6KYGU752IZFEZ4 | column -t
Chain           KUBE-SEP-AO6KYGU752IZFEZ4  (1   references)                                               
target          prot                       opt  source          destination                               
KUBE-MARK-MASQ  all                        --   10.244.120.102  anywhere     /*  default/webapp  */       
DNAT            tcp                        --   anywhere        anywhere     /*  default/webapp  */  tcp  to:10.244.120.102:80

$ sudo iptables -t nat -L KUBE-SEP-PJFBSHHDX4VZAOXM | column -t
Chain           KUBE-SEP-PJFBSHHDX4VZAOXM  (1   references)                                               
target          prot                       opt  source          destination                               
KUBE-MARK-MASQ  all                        --   10.244.120.103  anywhere     /*  default/webapp  */       
DNAT            tcp                        --   anywhere        anywhere     /*  default/webapp  */  tcp  to:10.244.120.103:80

$ sudo iptables -t nat -L KUBE-MARK-MASQ | column -t
Chain   KUBE-MARK-MASQ  (24  references)                         
target  prot            opt  source       destination            
MARK    all             --   anywhere     anywhere     MARK  or  0x4000
```

> 注意：输出内容已经被精简。

- ClusterIP：`KUBE-SERVICES` → `KUBE-SVC-XXX` → `KUBE-SEP-XXX`
- NodePort：`KUBE-SERVICES` → `KUBE-NODEPORTS` → `KUBE-SVC-XXX` → `KUBE-SEP-XXX`

> NodePort 服务会有一个 ClusterIP 用于处理内外部通信。

上述规则的可视化表达：

![图片](D:\学习资料\笔记\k8s\k8s图\127)

#### ExtrenalTrafficPolicy: Local

如前文所述，使用 `ExtrenalTrafficPolicy: Local` 会保留源 IP，并在到达节点上没有 Endpoint 的时候丢弃流量。没有本地 Endpoint 的节点上，iptables 的规则会怎样？

使用 `ExtrenalTrafficPolicy: Local` 部署 Nginx 服务：

```sh
$ kubectl get svc webapp -o wide -o jsonpath={.spec.externalTrafficPolicy}
Local

$ kubectl get svc webapp -o wide
NAME     TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE   SELECTOR
webapp   NodePort   10.111.243.62   <none>        80:30080/TCP   29m   app=webserver
```

检查一下没有本地 Endpoint 的节点上的 iptables 规则：

```sh
$ sudo iptables -t nat -L KUBE-NODEPORTS
Chain KUBE-NODEPORTS (1 references)
target prot opt source destination
KUBE-MARK-MASQ tcp — 127.0.0.0/8 anywhere /* default/webapp */ tcp dpt:30080
KUBE-XLB-2IRACUALRELARSND tcp — anywhere anywhere /* default/webapp */ tcp dpt:30080
```

再看一下 `KUBE-XLB-2IRACUALRELARSND`：

```sh
$ iptables -t nat -L KUBE-XLB-2IRACUALRELARSND
Chain KUBE-XLB-2IRACUALRELARSND (1 references)
target prot opt source destination
KUBE-SVC-2IRACUALRELARSND all — 10.244.0.0/16 anywhere /* Redirect pods trying to reach external loadbalancer VIP to clusterIP */
KUBE-MARK-MASQ all — anywhere anywhere /* masquerade LOCAL traffic for default/webapp LB IP */ ADDRTYPE match src-type LOCAL
KUBE-SVC-2IRACUALRELARSND all — anywhere anywhere /* route LOCAL traffic for default/webapp LB IP to service chain */ ADDRTYPE match src-type LOCAL
KUBE-MARK-DROP all — anywhere anywhere /* default/webapp has no local endpoints */
```

这里就会看到，集群级别的流量没什么问题，但是 NodePort 流量会被丢弃。



### Headless Service

有的应用并不需要负载均衡和服务 IP。在这种情况下就可以使用 `headless` Service，只要设置 `.spec.clusterIP` 为 `None` 即可。

可以借助这种服务类型和其他服务发现机制协作，无需和 Kubernetes 绑定。`kube-proxy` 不对这种没有 IP 的服务提供支持，也就没有什么负载均衡和代理之类的能力了。DNS 的配置要根据 Selector 来确定。

#### 有 Selector

定义了 Selector 的 Headless Service，Endpoint 控制器会创建 `Endpoint` 记录，并修改 DNS 记录来直接返回 Service 后端的 Pod 地址。

```sh
$ kubectl get svc webapp-hs
NAME        TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
webapp-hs   ClusterIP   None         <none>        80/TCP    24s
$ kubectl get ep webapp-hs
NAME        ENDPOINTS                             AGE
webapp-hs   10.244.120.109:80,10.244.120.110:80   31s
```

#### 无 Selector

没有定义 Selector 的 Headless Service，也就没有 Endpoint 记录。然而 DNS 系统会尝试配置：

- `ExternalName` 类型的服务，会产生 `CNAME` 记录；
- 其他类型则是所有 Endpoint 共享服务名称。

如果外部 IP 被路由到集群节点上，Kubernetes Service 可以用 `externalIPs` 开放出来。通过 `externalIP` 进入集群的流量，会被路由到 Service Endpoint 上。`externalIPs` 不是 Kubernetes 管理的，需要集群管理员自行维护。



### 网络策略

阅读至此，Kubernetes 网络策略的实现方法已经呼之欲出了——是的，就是 iptables。目前是 CNI 而非 `kube-proxy` 负责实现网络策略。这部分内容本来应该写在第二篇 Calico 的内容里，然而我认为这里写出来可能更合适。

我们创建三个服务：frontend、backend 和 db。

缺省情况下，Pod 没有任何隔离，会接受任何来源的通信。

![图片](D:\学习资料\笔记\k8s\k8s图\128)

想要制定规则，禁止 frontend 访问 db：

![图片](D:\学习资料\笔记\k8s\k8s图\129)

这里推荐阅读 Guide to Kubernetes Ingress Network Policies 了解网络策略配置方面的更多内容。本节内容关注的是 Kubernetes 中策略的实现方式，而非配置知识。

创建一个策略把 db 和 frontend 隔离开，这样一来 frontend 和 db 之间的流量就会被阻断。

> 上图中为了简单起见，写的是 Service 而非 Pod，安全策略的控制对象实际上是 Pod。

策略实施之后会产生如下效果，frontend 的 Pod 能访问 backend 但是无法访问 db。backend 的 Pod 可以访问 db。

```sh
$ kubectl exec -it frontend-8b474f47-zdqdv -- /bin/sh
$ curl backend
backend-867fd6dff-mjf92
$ curl db
curl: (7) Failed to connect to db port 80: Connection timed out

$ kubectl exec -it backend-867fd6dff-mjf92 -- /bin/sh
$ curl db
db-8d66ff5f7-bp6kf
```

看看这里用到的网络策略：只允许 `‘allow-db-access` 标签设置为 `true` 的 Pod 访问 db。

Calico 会把 Kubernetes 网络策略翻译成 Calico 格式：

```yaml
$ calicoctl get networkPolicy --output yaml
apiVersion: projectcalico.org/v3
items:
- apiVersion: projectcalico.org/v3
  kind: NetworkPolicy
  metadata:
    creationTimestamp: "2020-11-05T05:26:27Z"
    name: knp.default.allow-db-access
    namespace: default
    resourceVersion: /53872
    uid: 1b3eb093-b1a8-4429-a77d-a9a054a6ae90
  spec:
    ingress:
    - action: Allow
      destination: {}
      source:
        selector: projectcalico.org/orchestrator == 'k8s' && networking/allow-db-access
          == 'true'
    order: 1000
    selector: projectcalico.org/orchestrator == 'k8s' && app == 'db'
    types:
    - Ingress
kind: NetworkPolicyList
metadata:
  resourceVersion: 56821/56821
```

iptables 的 `filter` 表在网络策略的实现中起了很重要的作用。Calico 中用到了 `ipsec` 等高级概念，难于进行反向工程。在这个规则中可以看到，只有来自 backend 的流量才被允许发给 db。

使用 `calicoctl` 获取 endpoint 详情：

```sh
$ calicoctl get workloadEndpoint
WORKLOAD                         NODE       NETWORKS        INTERFACE         
backend-867fd6dff-mjf92          minikube   10.88.0.27/32   cali2b1490aa46a   
db-8d66ff5f7-bp6kf               minikube   10.88.0.26/32   cali95aa86cbb2a   
frontend-8b474f47-zdqdv          minikube   10.88.0.24/32   cali505cfbeac50
```

`cali95aa86cbb2a` 就是 db Pod veth 的主机侧。

看看跟这个网络接口有关的 iptables 规则：

```sh
$ sudo iptables-save | grep cali95aa86cbb2a
:cali-fw-cali95aa86cbb2a - [0:0]
:cali-tw-cali95aa86cbb2a - [0:0]
...
-A cali-tw-cali95aa86cbb2a -m comment --comment "cali:pm-LK-c1ra31tRwz" -m mark --mark 0x0/0x20000 -j cali-pi-_tTE-E7yY40ogArNVgKt
-A cali-tw-cali95aa86cbb2a -m comment --comment "cali:q_zG8dAujKUIBe0Q" -m comment --comment "Return if policy accepted" -m mark --mark 0x10000/0x10000 -j RETURN
-A cali-tw-cali95aa86cbb2a -m comment --comment "cali:FUDVBYh1Yr6tVRgq" -m comment --comment "Drop if no policies passed packet" -m mark --mark 0x0/0x20000 -j DROP
-A cali-tw-cali95aa86cbb2a -m comment --comment "cali:X19Z-Pa0qidaNsMH" -j cali-pri-kns.default
-A cali-tw-cali95aa86cbb2a -m comment --comment "cali:Ljj0xNidsduxDGUb" -m comment --comment "Return if profile accepted" -m mark --mark 0x10000/0x10000 -j RETURN
-A cali-tw-cali95aa86cbb2a -m comment --comment "cali:0z9RRvvZI9Gud0Wv" -j cali-pri-ksa.default.default
-A cali-tw-cali95aa86cbb2a -m comment --comment "cali:pNCpK-SOYelSULC1" -m comment --comment "Return if profile accepted" -m mark --mark 0x10000/0x10000 -j RETURN
-A cali-tw-cali95aa86cbb2a -m comment --comment "cali:sMkvrxvxj13WlTMK" -m comment --comment "Drop if no profiles matched" -j DROP

$ sudo iptables-save -t filter | grep cali-pi-_tTE-E7yY40ogArNVgKt
:cali-pi-_tTE-E7yY40ogArNVgKt - [0:0]
-A cali-pi-_tTE-E7yY40ogArNVgKt -m comment --comment "cali:M4Und37HGrw6jUk8" -m set --match-set cali40s:LrVD8vMIGQDyv8Y7sPFB1Ge src -j MARK --set-xmark 0x10000/0x10000
-A cali-pi-_tTE-E7yY40ogArNVgKt -m comment --comment "cali:sEnlfZagUFRSPRoe" -m mark --mark 0x10000/0x10000 -j RETURN
```

检查一下 ipset，会看到只有来自 backend pod 的 `10.88.0.27` 才能访问 db。



## [译]数据包在 Kubernetes 中的一生（4）

原文：Life of a Packet in Kubernetes — Part 4

链接：`https://dramasamy.medium.com/life-of-a-packet-in-kubernetes-part-4-4dbc5256050a)`

本篇内容会跟进 Kubernetes 的 Ingress 和 Ingress 控制器。Ingress 控制器会关注 API Server 中 Ingress 对象的更新，并据此配置 Ingress 的负载均衡。



### Nginx 控制器和负载均衡/代理服务器

Ingress 控制器一般会是一个以 Pod 形式运行在 Kubernetes 集群中的应用，它会根据集群中的 Ingress 对象的变化对负载均衡器进行配置。这里说的负载均衡器可以是一个集群内运行的软件，也可以是一个硬件，还可以是集群外部运行在云基础设施中。不同的负载均衡器需要不同的 Ingress 控制器。

![图片](D:\学习资料\笔记\k8s\k8s图\130)

Ingress 的基本目标是提供一个相对高级的流量（尤其是 http(s)）管理能力。使用 Ingress 可以在无需创建多个负载均衡或者对外开放多个 Service 的条件下，为服务流量进行路由。可以给服务配置外部 URL、进行负载均衡、终结 SSL 以及根据主机名或者内容进行路由等。



### 配置选项

在把 Ingress 对象转换为负载均衡配置之前，Kubernetes Ingress 控制器会用 Ingress Class 对 Kubernetes 的 Ingress 对象进行过滤。这样就允许多个 Ingress 控制器共存，各司其职。

#### 基于前缀

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prefix-based
  annotations:
    kubernetes.io/ingress.class: "nginx-ingress-inst-1"
spec:
  rules:
  - http:
      paths:
      - path: /video
        pathType: Prefix
        backend:
          service:
            name: video
            port:
              number: 80
      - path: /store
        pathType: Prefix
        backend:
          service:
            name: store
            port:
              number: 80
```

#### 基于主机

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-based
  annotations:
    kubernetes.io/ingress.class: "nginx-ingress-inst-1"
spec:
  rules:
  - host: "video.example.com"
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: video
            port:
              number: 80
  - host: "store.example.com"
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: store
            port:
              number: 80
```

#### 主机加前缀

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: host-prefix-based
  annotations:
    kubernetes.io/ingress.class: "nginx-ingress-inst-1"
spec:
  rules:
  - host: foo.com
    http:
      paths:
      - backend:
          serviceName: foovideo
          servicePort: 80
        path: /video
      - backend:
          serviceName: foostore
          servicePort: 80
        path: /store
  - host: bar.com
    http:
      paths:
      - backend:
          serviceName: barvideo
          servicePort: 80
        path: /video
      - backend:
          serviceName: barstore
          servicePort: 80
        path: /store
```

Ingress 是一个内置 API 对象，但是 Kubernetes 并没有内置任何 Ingress 控制器，需要另行安装控制器才能真正地使用 Ingress。Ingress 功能是由 API 对象和控制器协同完成的。Ingress 对象负责描述集群中 Service 对象的开放需求。而控制器则负责真正的实现 Ingress API，根据 Ingress 对象的定义内容来完成实际工作。市面上有很多不同的 Ingress 控制器，需要根据实际用例谨慎地进行选择使用。

同一集群里可以有多个 Ingress 控制器，并为每个 Ingress 直接指派具体的控制器，在同一个集群中可以根据不同需要为不同服务配置不同的 Ingress。例如某服务用于一个 Ingress 处理来自集群外的 SSL 流量，另外一个用于处理集群内的明文通信。



### 部署选项

#### Contour + Envoy

Contour Ingress 控制器由两部分组成：

- Envoy 提供了高性能的反向代理服务；
- Contour 负责对 Envoy 进行管理，为其下发配置。

这些容器是各自部署的，Contour 是一个 Deployment，而 Envoy 则是一个 Daemonset，当然也可以用其他模式进行部署。Contour 是 Kubernetes API 的客户端，会跟踪 Ingress、HTTPProxy、Secret、Service 以及 Endpoint 对象，并承担管理 Envoy 的职责，它会把它的对象缓存转换为 Envoy 的 JSON 报文，Service 转换为 CDS、Ingress 转为 RDS、Endpoint 转换为 EDS 等。

下面的例子展示了启用 Host Network 的 EnvoyProxy：

![图片](D:\学习资料\笔记\k8s\k8s图\131)

#### Nginx

Nginx Ingress 控制器的主要能力之一就是生成配置文件（`nginx.conf`）。这个实现还有个需要就是在配置发生变化之后重载 Nginx。在只有 `upstream` 发生变化时（例如部署调整时产生的 Endpoint 变化）不会进行重载，而是通过 lua-nginx-module 完成任务。

每次 Endpoint 发生变动，控制器会从所有服务中拉取 Endpoint，生成对应的后端对象。这些对象会被发送给 Nginx 中运行的 Lua 处理器。Lua 代码会把这些对象保存到共享内存区域。每次 `balancer_by_lua` 都会检查一下 upstream 中的有效节点，以此为目标按照预配置的算法进行负载均衡。如果在一个较大的集群中有比较频繁的发布行为，这种避免重载的方式能够大幅减少重载次数，从而更好地保障了响应的延迟时间，达成较高的负载均衡水平。

#### Nginx+ Keepalived —— 高可用部署

Keepalived 守护进程可以监控服务或者系统，如果发现问题，能够进行自动的切换。配置一个能在节点之间转移的浮动 IP。如果节点宕机，浮动 IP 会自动漂移到其它节点，Nginx 可以绑定到新的 IP 地址。

![图片](D:\学习资料\笔记\k8s\k8s图\132)

#### MetalLB —— 面向具备少量公有 IP 池的私有集群的负载均衡服务

部署到 Kubernetes 中的 MetalLB 为集群提供了一个负载均衡的实现。简单说来，MetalLB 能够在非公有云 Kubernetes 环境中对 LoadBalancer 类型的 Service 提供支持。在托管 Kubernetes 环境中，申请一个负载均衡之后，云平台会给这个新的负载均衡分配 IP；MetalLB 可以负责这个分配过程。MetalLB 给 Service 分配外部 IP 之后，需要声明该 IP 属于本集群，它使用标准路由协议来完成这一任务：ARP、NDP 或 BGP。

在 2 层模式中，集群的一个节点获取这个 Service 的所有权，然后使用标准的地址发现协议（IPv4 使用 ARP、IPv6 使用 NDP）在本地网中让次 IP 可达。从局域网的角度来看，这个节点只是多了一个 IP 地址。

在 BGP 模式中，集群中的所有节点都会对附近的路由器发起 BGP 对等会话，告知路由器如何将流量转发给这些服务。BGP 的策略机制有细粒度的流量控制能力，能真正地在多个节点之间进行负载均衡。

MetalLB 的 Pod：

- Controller（Deployment）是集群级的 MetalLB 控制器，负责 IP 分配。
- Speaker（Daemonset）在每个节点上运行，使用多种发布策略公告服务和外部 IP 的对应关系。

![图片](D:\学习资料\笔记\k8s\k8s图\133)

> MetalLB 能够用在集群里的任何 `LoadBalancer` 类型的 Service 中，但是 MetalLB 为大型 IP 地址池工作就不太现实了。



## kube-proxy IPVS 模式的工作原理

`Kubernetes` 中的 `Service` 就是一组同 label 类型 `Pod` 的服务抽象，为服务提供了负载均衡和反向代理能力，在集群中表示一个微服务的概念。`kube-proxy` 组件则是 Service 的具体实现，了解了 kube-proxy 的工作原理，才能洞悉服务之间的通信流程，再遇到网络不通时也不会一脸懵逼。

kube-proxy 有三种模式：`userspace`、`iptables` 和 `IPVS`，其中 `userspace` 模式不太常用。`iptables` 模式最主要的问题是在服务多的时候产生太多的 iptables 规则，非增量式更新会引入一定的时延，大规模情况下有明显的性能问题。为解决 `iptables` 模式的性能问题，v1.11 新增了 `IPVS` 模式（v1.8 开始支持测试版，并在 v1.11 GA），采用增量式更新，并可以保证 service 更新期间连接保持不断开。

目前网络上关于 `kube-proxy` 工作原理的文档几乎都是以 `iptables` 模式为例，很少提及 `IPVS`，本文就来破例解读 kube-proxy IPVS 模式的工作原理。为了理解地更加彻底，本文不会使用 Docker 和 Kubernetes，而是使用更加底层的工具来演示。

我们都知道，Kubernetes 会为每个 Pod 创建一个单独的网络命名空间 (Network Namespace) ，本文将会通过手动创建网络命名空间并启动 HTTP 服务来模拟 Kubernetes 中的 Pod。

本文的目标是通过模拟以下的 `Service` 来探究 kube-proxy 的 `IPVS` 和 `ipset` 的工作原理：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  clusterIP: 10.100.100.100
  selector:
    component: app
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
```

跟着我的步骤，最后你就可以通过命令 `curl 10.100.100.100:8080` 来访问某个网络命名空间的 HTTP 服务。为了更好地理解本文的内容，推荐提前阅读以下的文章：

1. **How do Kubernetes and Docker create IP Addresses?!**[1]
2. **iptables: How Docker Publishes Ports**[2]
3. **iptables: How Kubernetes Services Direct Traffic to Pods**[3]

> 注意：本文所有步骤皆是在 Ubuntu 20.04 中测试的，其他 Linux 发行版请自行测试。



### 准备实验环境

首先需要开启 Linux 的路由转发功能：

```sh
$ sysctl --write net.ipv4.ip_forward=1
```

接下来的命令主要做了这么几件事：

- 创建一个虚拟网桥 `bridge_home`
- 创建两个网络命名空间 `netns_dustin` 和 `netns_leah`
- 为每个网络命名空间配置 DNS
- 创建两个 veth pair 并连接到 `bridge_home`
- 给 `netns_dustin` 网络命名空间中的 veth 设备分配一个 IP 地址为 `10.0.0.11`
- 给 `netns_leah` 网络命名空间中的 veth 设备分配一个 IP 地址为 `10.0.021`
- 为每个网络命名空间设定默认路由
- 添加 iptables 规则，允许流量进出 `bridge_home` 接口
- 添加 iptables 规则，针对 `10.0.0.0/24` 网段进行流量伪装

```sh
$ ip link add dev bridge_home type bridge
$ ip address add 10.0.0.1/24 dev bridge_home

$ ip netns add netns_dustin
$ mkdir -p /etc/netns/netns_dustin
echo "nameserver 114.114.114.114" | tee -a /etc/netns/netns_dustin/resolv.conf
$ ip netns exec netns_dustin ip link set dev lo up
$ ip link add dev veth_dustin type veth peer name veth_ns_dustin
$ ip link set dev veth_dustin master bridge_home
$ ip link set dev veth_dustin up
$ ip link set dev veth_ns_dustin netns netns_dustin
$ ip netns exec netns_dustin ip link set dev veth_ns_dustin up
$ ip netns exec netns_dustin ip address add 10.0.0.11/24 dev veth_ns_dustin

$ ip netns add netns_leah
$ mkdir -p /etc/netns/netns_leah
echo "nameserver 114.114.114.114" | tee -a /etc/netns/netns_leah/resolv.conf
$ ip netns exec netns_leah ip link set dev lo up
$ ip link add dev veth_leah type veth peer name veth_ns_leah
$ ip link set dev veth_leah master bridge_home
$ ip link set dev veth_leah up
$ ip link set dev veth_ns_leah netns netns_leah
$ ip netns exec netns_leah ip link set dev veth_ns_leah up
$ ip netns exec netns_leah ip address add 10.0.0.21/24 dev veth_ns_leah

$ ip link set bridge_home up
$ ip netns exec netns_dustin ip route add default via 10.0.0.1
$ ip netns exec netns_leah ip route add default via 10.0.0.1

$ iptables --table filter --append FORWARD --in-interface bridge_home --jump ACCEPT
$ iptables --table filter --append FORWARD --out-interface bridge_home --jump ACCEPT

$ iptables --table nat --append POSTROUTING --source 10.0.0.0/24 --jump MASQUERADE
```

在网络命名空间 `netns_dustin` 中启动 HTTP 服务：

```
$ ip netns exec netns_dustin python3 -m http.server 8080
```

打开另一个终端窗口，在网络命名空间 `netns_leah` 中启动 HTTP 服务：

```
$ ip netns exec netns_leah python3 -m http.server 8080
```

测试各个网络命名空间之间是否能正常通信：

```
$ curl 10.0.0.11:8080
$ curl 10.0.0.21:8080
$ ip netns exec netns_dustin curl 10.0.0.21:8080
$ ip netns exec netns_leah curl 10.0.0.11:8080
```

整个实验环境的网络拓扑结构如图：

![图片](D:\学习资料\笔记\k8s\k8s图\134)



### 安装必要工具

为了便于调试 IPVS 和 ipset，需要安装两个 CLI 工具：

```sh
$ apt install ipset ipvsadm --yes
```

> 本文使用的 ipset 和 ipvsadm 版本分别为 `7.5-1~exp1` 和 `1:1.31-1`。



### 通过 IPVS 来模拟 Service

下面我们使用 `IPVS` 创建一个虚拟服务 (Virtual Service) 来模拟 Kubernetes 中的 Service :

```sh
$ ipvsadm \
  --add-service \
  --tcp-service 10.100.100.100:8080 \
  --scheduler rr
```

- 这里使用参数 `--tcp-service` 来指定 TCP 协议，因为我们需要模拟的 Service 就是 TCP 协议。
- IPVS 相比 iptables 的优势之一就是可以轻松选择调度算法，这里选择使用轮询调度算法。

> 目前 kube-proxy 只允许为所有 Service 指定同一个调度算法，未来将会支持为每一个 Service 选择不同的调度算法，详情可参考文章 **IPVS-Based In-Cluster Load Balancing Deep Dive**[4]。

创建了虚拟服务之后，还得给它指定一个后端的 `Real Server`，也就是后端的真实服务，即网络命名空间 `netns_dustin` 中的 HTTP 服务：

```sh
$ ipvsadm \
  --add-server \
  --tcp-service 10.100.100.100:8080 \
  --real-server 10.0.0.11:8080 \
  --masquerading
```

该命令会将访问 `10.100.100.100:8080` 的 TCP 请求转发到 `10.0.0.11:8080`。这里的 `--masquerading` 参数和 iptables 中的 `MASQUERADE` 类似，如果不指定，IPVS 就会尝试使用路由表来转发流量，这样肯定是无法正常工作的。

测试是否正常工作：

```sh
$ curl 10.100.100.100:8080
```

实验成功，请求被成功转发到了后端的 HTTP 服务！



### 在网络命名空间中访问虚拟服务

上面只是在 Host 的网络命名空间中进行测试，现在我们进入网络命名空间 `netns_leah` 中进行测试：

```sh
$ ip netns exec netns_leah curl 10.100.100.100:8080
```

哦豁，访问失败！

要想顺利通过测试，只需将 `10.100.100.100` 这个 IP 分配给一个虚拟网络接口。至于为什么要这么做，目前我还不清楚，我猜测可能是因为网桥 `bridge_home` 不会调用 IPVS，而将虚拟服务的 IP 地址分配给一个网络接口则可以绕过这个问题。

#### dummy 接口

当然，我们不需要将 IP 地址分配给任何已经被使用的网络接口，我们的目标是模拟 Kubernetes 的行为。Kubernetes 在这里创建了一个 dummy 接口，它和 loopback 接口类似，但是你可以创建任意多的 dummy 接口。它提供路由数据包的功能，但实际上又不进行转发。dummy 接口主要有两个用途：

- 用于主机内的程序通信
- 由于 dummy 接口总是 up（除非显式将管理状态设置为 down），在拥有多个物理接口的网络上，可以将 service 地址设置为 loopback 接口或 dummy 接口的地址，这样 service 地址不会因为物理接口的状态而受影响。

看来 dummy 接口完美符合实验需求，那就创建一个 dummy 接口吧：

```sh
$ ip link add dev dustin-ipvs0 type dummy
```

将虚拟 IP 分配给 dummy 接口 `dustin-ipvs0` :

```sh
$ ip addr add 10.100.100.100/32 dev dustin-ipvs0
```

到了这一步，仍然访问不了 HTTP 服务，还需要另外一个黑科技：`bridge-nf-call-iptables`。在解释 `bridge-nf-call-iptables` 之前，我们先来回顾下容器网络通信的基础知识。

#### 基于网桥的容器网络

Kubernetes 集群网络有很多种实现，有很大一部分都用到了 Linux 网桥:

![图片](D:\学习资料\笔记\k8s\k8s图\136)

- 每个 Pod 的网卡都是 veth 设备，veth pair 的另一端连上宿主机上的网桥。
- 由于网桥是虚拟的二层设备，同节点的 Pod 之间通信直接走二层转发，跨节点通信才会经过宿主机 eth0。

#### Service 同节点通信问题

不管是 iptables 还是 ipvs 转发模式，Kubernetes 中访问 Service 都会进行 DNAT，将原本访问 `ClusterIP:Port` 的数据包 DNAT 成 Service 的某个 `Endpoint (PodIP:Port)`，然后内核将连接信息插入 `conntrack` 表以记录连接，目的端回包的时候内核从 `conntrack` 表匹配连接并反向 NAT，这样原路返回形成一个完整的连接链路:

![图片](D:\学习资料\笔记\k8s\k8s图\137)

但是 Linux 网桥是一个虚拟的二层转发设备，而 iptables conntrack 是在三层上，所以如果直接访问同一网桥内的地址，就会直接走二层转发，不经过 conntrack:

1. Pod 访问 Service，目的 IP 是 Cluster IP，不是网桥内的地址，走三层转发，会被 DNAT 成 PodIP:Port。
2. 如果 DNAT 后是转发到了同节点上的 Pod，目的 Pod 回包时发现目的 IP 在同一网桥上，就直接走二层转发了，没有调用 conntrack，导致回包时没有原路返回 (见下图)。

![图片](D:\学习资料\笔记\k8s\k8s图\138)

由于没有原路返回，客户端与服务端的通信就不在一个 “频道” 上，不认为处在同一个连接，也就无法正常通信。

#### 开启 bridge-nf-call-iptables

启用 `bridge-nf-call-iptables` 这个内核参数 (置为 1)，表示 bridge 设备在二层转发时也去调用 iptables 配置的三层规则 (包含 conntrack)，所以开启这个参数就能够解决上述 Service 同节点通信问题。

所以这里需要启用 `bridge-nf-call-iptables` :

```sh
$ modprobe br_netfilter
$ sysctl --write net.bridge.bridge-nf-call-iptables=1
```

现在再来测试一下连通性：

```sh
$ ip netns exec netns_leah curl 10.100.100.100:8080
```

终于成功了！



### 开启 Hairpin（发夹弯）模式

虽然我们可以从网络命名空间 `netns_leah` 中通过虚拟服务成功访问另一个网络命名空间 `netns_dustin` 中的 HTTP 服务，但还没有测试过从 HTTP 服务所在的网络命名空间 `netns_dustin` 中直接通过虚拟服务访问自己，话不多说，直接测一把：

```sh
$ ip netns exec netns_dustin curl 10.100.100.100:8080
```

啊哈？竟然失败了，这又是哪里的问题呢？不要慌，开启 `hairpin` 模式就好了。那么什么是 `hairpin` 模式呢？这是一个网络虚拟化技术中常提到的概念，也即交换机端口的 VEPA 模式。这种技术借助物理交换机解决了虚拟机间流量转发问题。很显然，这种情况下，源和目标都在一个方向，所以就是从哪里进从哪里出的模式。

怎么配置呢？非常简单，只需一条命令：

```sh
$ brctl hairpin bridge_home veth_dustin on
```

再次进行测试：

```sh
$ ip netns exec netns_dustin curl 10.100.100.100:8080
```

还是失败了。。。

然后我花了一个下午的时间，终于搞清楚了启用混杂模式后为什么还是不能解决这个问题，因为混杂模式和下面的选项要一起启用才能对 IPVS 生效：

```sh
$ sysctl --write net.ipv4.vs.conntrack=1
```

最后再测试一次：

```sh
$ ip netns exec netns_dustin curl 10.100.100.100:8080
```

这次终于成功了，但我还是不太明白为什么启用 conntrack 能解决这个问题，有知道的大神欢迎留言告诉我！



### 开启混杂模式

如果想让所有的网络命名空间都能通过虚拟服务访问自己，就需要在连接到网桥的所有 veth 接口上开启 `hairpin` 模式，这也太麻烦了吧。有一个办法可以不用配置每个 veth 接口，那就是开启网桥的混杂模式。

什么是混杂模式呢？普通模式下网卡只接收发给本机的包（包括广播包）传递给上层程序，其它的包一律丢弃。混杂模式就是接收所有经过网卡的数据包，包括不是发给本机的包，即不验证 MAC 地址。

**如果一个网桥开启了混杂模式，就等同于将所有连接到网桥上的端口（本文指的是 veth 接口）都启用了 `hairpin` 模式**。可以通过以下命令来启用 `bridge_home` 的混杂模式：

```sh
$ ip link set bridge_home promisc on
```

现在即使你把 veth 接口的 `hairpin` 模式关闭：

```sh
$ brctl hairpin bridge_home veth_dustin off
```

仍然可以通过连通性测试：

```
$ ip netns exec netns_dustin curl 10.100.100.100:8080
```



### 优化 MASQUERADE

在文章开头准备实验环境的章节，执行了这么一条命令：

```sh
$ iptables \
  --table nat \
  --append POSTROUTING \
  --source 10.0.0.0/24 \
  --jump MASQUERADE
```

这条 iptables 规则会对所有来自 `10.0.0.0/24` 的流量进行伪装。然而 Kubernetes 并不是这么做的，它为了提高性能，只对来自某些具体的 IP 的流量进行伪装。

为了更加完美地模拟 Kubernetes，我们继续改造规则，先把之前的规则删除：

```sh
$ iptables \
  --table nat \
  --delete POSTROUTING \
  --source 10.0.0.0/24 \
  --jump MASQUERADE
```

然后添加针对具体 IP 的规则：

```sh
$ iptables \
  --table nat \
  --append POSTROUTING \
  --source 10.0.0.11/32 \
  --jump MASQUERADE
```

果然，上面的所有测试都能通过。先别急着高兴，又有新问题了，现在只有两个网络命名空间，如果有很多个怎么办，每个网络命名空间都创建这样一条 iptables 规则？我用 IPVS 是为了啥？就是为了防止有大量的 iptables 规则拖垮性能啊，现在岂不是又绕回去了。

不慌，继续从 Kubernetes 身上学习，使用 `ipset` 来解决这个问题。先把之前的 iptables 规则删除：

```sh
$ iptables \
  --table nat \
  --delete POSTROUTING \
  --source 10.0.0.11/32 \
  --jump MASQUERADE
```

然后使用 `ipset` 创建一个集合 (set) ：

```sh
$ ipset create DUSTIN-LOOP-BACK hash:ip,port,ip
```

这条命令创建了一个名为 `DUSTIN-LOOP-BACK` 的集合，它是一个 `hashmap`，里面存储了目标 IP、目标端口和源 IP。

接着向集合中添加条目：

```sh
$ ipset add DUSTIN-LOOP-BACK 10.0.0.11,tcp:8080,10.0.0.11
```

现在不管有多少网络命名空间，都只需要添加一条 iptables 规则：

```sh
$ iptables \
  --table nat \
  --append POSTROUTING \
  --match set \
  --match-set DUSTIN-LOOP-BACK dst,dst,src \
  --jump MASQUERADE
```

网络连通性测试也没有问题：

```sh
$ curl 10.100.100.100:8080
$ ip netns exec netns_leah curl 10.100.100.100:8080
$ ip netns exec netns_dustin curl 10.100.100.100:8080
```



### 新增虚拟服务的后端

最后，我们把网络命名空间 `netns_leah` 中的 HTTP 服务也添加到虚拟服务的后端：

```sh
$ ipvsadm \
  --add-server \
  --tcp-service 10.100.100.100:8080 \
  --real-server 10.0.0.21:8080 \
  --masquerading
```

再向 ipset 的集合 `DUSTIN-LOOP-BACK` 中添加一个条目：

```sh
$ ipset add DUSTIN-LOOP-BACK 10.0.0.21,tcp:8080,10.0.0.21
```

终极测试来了，试着多运行几次以下的测试命令：

```sh
$ curl 10.100.100.100:8080
```

你会发现轮询算法起作用了：

![图片](D:\学习资料\笔记\k8s\k8s图\139)



### 总结

相信通过本文的实验和讲解，大家应该理解了 kube-proxy IPVS 模式的工作原理。在实验过程中，我们还用到了 ipset，它有助于解决在大规模集群中出现的 kube-proxy 性能问题。如果你对这篇文章有任何疑问，欢迎和我进行交流。



### 参考文章

- **为什么 kubernetes 环境要求开启 bridge-nf-call-iptables ?**[5]

#### 脚注

[1]How do Kubernetes and Docker create IP Addresses?!: *https://dustinspecker.com/posts/how-do-kubernetes-and-docker-create-ip-addresses/*

[2]iptables: How Docker Publishes Ports: *https://dustinspecker.com/posts/iptables-how-docker-publishes-ports/*

[3]iptables: How Kubernetes Services Direct Traffic to Pods: *https://dustinspecker.com/posts/iptables-how-kubernetes-services-direct-traffic-to-pods/*

[4]IPVS-Based In-Cluster Load Balancing Deep Dive: *https://kubernetes.io/blog/2018/07/09/ipvs-based-in-cluster-load-balancing-deep-dive/#ipvs-based-kube-proxy*

[5]为什么 kubernetes 环境要求开启 bridge-nf-call-iptables ?: *https://imroc.cc/post/202105/why-enable-bridge-nf-call-iptables/*

原文链接：**https://dustinspecker.com/posts/ipvs-how-kubernetes-services-direct-traffic-to-pods/**



## 从 iptables 谈 ServiceMesh 流量拦截

最近研究学习 Kubernetes 和 ServiceMesh 过程中都看到了 **iptables** 发挥重要作用独挡一面的场景。
Kubernetes 中 iptables 作为 kube-proxy 里控制流量转发的核心模式，通过在目标 node 的 iptables 中增加一些自定义链对流经到该 node 的数据包做DNAT和SNAT操作以实现路由、负载均衡和地址转换。
ServiceMesh 服务网格中 Istio 通过 init 容器（启动命令为istio-iptables）给 Sidecar 容器即 Envoy 代理做初始化，设置 iptables 端口转发，从而实现流量劫持&转发控制等服务治理相关。



### 一、iptables 基础

提起 iptables 相信大家的第一反应是--防火墙，不知道你们是不是，反正我的第一反应是这样的，偷笑……
但其实 Linux 中包过滤防火墙全称为 Netfilter/Iptables。位于内核空间的 Netfilter 才是防火墙真正的安全框架，它可以根据规则实现数据包的过滤、网络地址转换、内容修改等。Iptables 是位于用户空间的一个命令行工具，用来操作 Netfilter，对 NetFilter 中的规则进行增删改查，从而将用户的安全设定同步进内核空间的 Netfilter。

后面用 iptables 代替 Netfilter/Iptables。我们先用一个思维导图来看下 iptables 的核心概念（按优先级画的表，实际上使用频率及场景的话 filter 表和 nat 表使用的更多些）。

![图片](D:\学习资料\笔记\k8s\k8s图\140)

根据思维导图我们可以看到 iptables 中有 **五条链** -- PREROUTING路由前/INPUT流入/FORWARD转发/OUTPUT流出/POSTROUTING 路由后 和 **四张表** -- Raw/Mangle/Nat/Filter，各个表和链中又可以设定多条规则，从而对数据包进行控制。

里面有几个名词需要解释下：
**链(Chain)**：之所以先解释链是因为内核源码里最直接的 iptables 线索是通过链来体现的（思维导图里为了按功能区分先画的表）。链其实是按数据包流向划分的一些钩子函数 HOOK，内核源码里搜索 NF_HOOK 可以看到相关内容，根据数据包所处的不同网络协议层，都有不同的链 HOOK，如 ARP 层、Bridge 层、IP 层、Application 层，具体各层的 HOOK 细节可以参考**NetFilter hooks**[1]

![图片](D:\学习资料\笔记\k8s\k8s图\141)

**表(Table)**：表是按功能化分的规则集合，如nat表主要负责网络地址转换，filter表主要负责过滤，mangle表主要负责修改数据包标记等，就如同一个餐厅里的服务员、大厨、配菜员、洗消工人、保安等分工不同。每条链中根据功能不同可以使用特定的几个或全部表的规则。
**规则(Policy)**：按指定的条件来匹配数据包，匹配成功后执行相应的通过/丢弃等动作处理。

打个比方，有一家高档的西餐厅：
**进门时，**服务员会先审视一遍顾客，因为餐厅规定需着正装进入所以光膀子大裤衩人字拖的就被拒之门外了，此时顾客还不服气大吵大闹的，这时候保安只能按违反公共环境保持安静的要求把他们带走了。另外一位顾客穿着得体但是带了宠物，服务员指着旁边告示牌上写的告示礼貌的提醒顾客将宠物暂存在了前台准备的宠物笼中。
**第二步开始点餐，**顾客点了一份七分熟的牛排和一份鱼香肉丝，等会儿，这是西餐厅不卖鱼香肉丝啊，西餐厅出现鱼香肉丝会影响餐厅形象的，服务员非常有礼貌的拒绝了顾客鱼香肉丝的需求并告知顾客鱼香肉丝可以去旁边的川菜馆享用，同时正常下单了牛排的需求。
**第三步制作阶段，**配菜员配菜，需按餐厅宣传的牛排肉质的要求及分量进行配菜，然后交给大厨进行烹饪，这里注意牛排要求七分熟，所以大厨特别认真的盯紧火候生怕过大或者过小，毕竟火候过了或者欠了都不满足顾客需求，都会被顾客投诉。
**第四步买单阶段，**顾客享用完美味的餐食后，离开餐桌准备买单，这时服务员会负责引导顾客到达收银处进行买单，同时洗消工人进行餐桌餐具及厨余垃圾的洗消工作。

结合这个场景屡一下链、表和规则的关系：
上客阶段（**链1**）：服务员（**表1**，保障餐厅形象）-- **规则1**要求穿着得体、**规则2**要求禁止携带宠物；保安（**表5**，维持餐厅秩序） -- **规则1**禁止大吵大闹
点餐阶段（**链2**）：服务员（**表1**，保障餐厅形象）-- 拒绝提供非西餐食品
制作阶段（**链3**）：配菜员（**表2**，保障餐厅食材品质）-- 严格筛选高品质肉类及蔬菜；大厨（**表3**，保障餐厅食物口感）-- 严格控制火候、调味剂分量及每个步骤时长等
买单阶段（**链4**）：服务员（**表1**，保障餐厅形象）、洗消工人（**表4**，保障餐厅卫生环境）-- 都有具体的卫生要求标准

现在是不是清楚些了，iptables中链是主线条，然后每条链按需要执行特定功能表中的一条或多条规则，如上客阶段需要服务员表中的规则和保安表中的规则。

iptables中的四张表按优先级raw –> mangle –> nat –> filter执行。然后具体的规则按照从前往后的优先级执行，要求越严格的规则应该放在越靠前面。
思维导图中对四张表分别做了简要的功能概述，接下来我们看下五条链以及他们根据功能需要用到哪些表的规则。

![图片](D:\学习资料\笔记\k8s\k8s图\142)

现在我们对数据包的流向及相应的表和链的控制有了个大概的了解，如果还需要更详细的逐步的查看请参考**iptables四个表五条链**[2]。



### 二、iptables 命令使用

官方定义的命令格式如下：

```sh
iptables -t 表名 <-A/I/D/R/L/F> 规则链名 [规则号] <-i/o 网卡名> -p 协议名 <-s 源IP> --sport 源端口 <-d 目标IP> --dport 目标端口 -j 动作  

-t 指定表名，未指定表名时默认为 Filter 表
-A和-I都是添加规则，-A增加的规则放在现有规则的最后，-I添加的规则放在规则号指定的位置，该位置原先的规则往后顺位。
-D 删除规则号指定的规则
-R 替换规则号指定的规则
-L 查看相应的规则
-F 清楚某条链或者表的规则
-i/o 指定输入和输出的网卡
-p 指定数据包协议，如 tcp、udp、icmp 等，这里支持简单的表达式，如 -p !tcp 去除 tcp 外的所有协议
-s和-sport分别指定数据包源 IP 地址及端口
-d和-dport分别指定数据包目标 IP 地址及端口
-j 指定前述的参数匹配上数据包以后执行的动作。常用的处理动作包括 ACCEPT 放行、REJECT 拒绝、DROP 丢弃、REDIRECT 重定向、DNAT 修改目的 IP 及端口、SNAT 修改源 IP 及端口等等
```

另外还有一些高级的参数，如参数 tcp-flags (只过滤 TCP 中的一些包，比如 SYN 包，ACK 包，FIN 包，RST 包等等)、参数 limit 限制数据包的平均流量、参数 state 过滤特定状态(如Established、Invalid 等)的数据包，类似的参数还有不一一列举了。

下面结合一些实际应用场景看看具体怎么使用吧~~

1.查看服务器上 nat 表的所有规则

```sh
$ iptables -L -n -v -t nat
Chain PREROUTING (policy ACCEPT 11 packets, 2250 bytes)
pkts bytes target     prot opt in     out     source       destination

Chain INPUT (policy ACCEPT 11 packets, 2250 bytes)
pkts bytes target     prot opt in     out     source       destination

Chain OUTPUT (policy ACCEPT 35 packets, 2838 bytes)
pkts bytes target     prot opt in     out     source       destination

Chain POSTROUTING (policy ACCEPT 35 packets, 2838 bytes)
pkts bytes target     prot opt in     out     source       destination
```

这里再次看到了 nat 表只包含了 PREROUTING/INPUT/OUTPUT/POSTROUTING 四条链的规则，不包含 FORWARD 链的规则。

2.禁用SSHD默认的22端口

```sh
$ iptables -t filter -A INPUT -p tcp --dport 22 -j DROP
```

3.只允许特定网段10.160.0.0/16访问本机的10.160.100.1的SSHD(22端口)服务

```sh
 #设置默认的drop，再允许特定的网段进入和出去
$ iptables -P INPUT DROP
$ iptables -P OUTPUT DROP
$ iptables -P FORWARD DROP

$ iptables -t filter -A INPUT -s 10.160.0.0/16 -d 10.160.100.1 -p tcp --dport 22 -j ACCEPT
$ iptables -t filter -A OUTPUT -s 10.160.100.1 -d 10.160.0.0/16 -p tcp --dport 22 -j ACCEPT
```

4.过滤掉状态有问题的http包。只允许http80端口且限定连接状态为Established和Related的数据包

```sh
$ iptables -A INPUT -p tcp  --sport 80 -m state --state ESTABLISHED,RELATED -j ACCEPT
```

5.开启儿童上网模式，星期一到星期五的8:00-21:00禁止游戏相关网页”game“

```sh
$ iptables -I FORWARD -s 192.168.0.0/24 -m string --string "game" -m time --timestart 8:00 --timestop 21:00 --days Mon,Tue,Wed,Thu,Fri -j DROP
```

6.生产环境mysql数据库仅允许内网特定ip访问

```sh
$ iptables –A INPUT –s 10.160.41.1 –p tcp –dport 3306 –j ACCEPT
```

7.将目的IP为10.160.132.55且目的端口为9090的我们做DNAT修改目标地址处理，重定向到10.162.37.1:8080

```sh
$ iptables  -A INPUT -d 10.160.132.55 -p tcp --dport 9090 -j DNAT --to 10.162.37.1:8080
```

8.拦截所有入站tcp80端口和8080端口数据包重定向到某个代理服务的15001端口进行统一处理

```sh
$ iptables -A INPUT -p tcp --dport 80,8080 -j REDIRECT --to-ports 15001
```

ps:这里已经看到 ServiceMesh 服务网格中 sidecar 模式的 Envoy 代理里实现流量拦截从而进行统一的服务治理的一点身影了~~~



### 三、ServiceMesh 中 iptables 的发挥

ServiceMesh 中 iptables 怎么发挥作用的呐？Istio 通过为 Pod 注入 Init 容器来为 Pod 设置 iptables 规则进行端口转发。Init 容器是一种专用容器，它在应用程序容器启动之前运行，用来包含一些应用镜像中不存在的实用工具或安装脚本。它的启动命令是

```sh
$ istio-iptables -p 15001 -z 15006 -u 1337 -m REDIRECT -i '*' -x "" -b '*'
```

感兴趣可以具体查看istio中源码https://github.com/istio/istio/blob/master/tools/istio-iptables，这里不再单独拿出来说了。

istio中iptables命令格式如下：

```sh
$ istio-iptables -p PORT -u UID -g GID [-m mode] [-b ports] [-d ports] [-i CIDR] [-x CIDR] [-h]

  -p: 指定重定向所有 TCP 流量的 Envoy 端口（默认为 $ENVOY_PORT = 15001）
  -u: 指定未应用重定向的用户的 UID。通常，这是代理容器的 UID（默认为 $ENVOY_USER 的 uid，istio_proxy 的 uid 或 1337）
  -g: 指定未应用重定向的用户的 GID。（与 -u param 相同的默认值）
  -m: 指定入站连接重定向到 Envoy 的模式，“REDIRECT” 或 “TPROXY”（默认为 $ISTIO_INBOUND_INTERCEPTION_MODE)
  -b: 逗号分隔的入站端口列表，其流量将重定向到 Envoy（可选）。使用通配符 “*” 表示重定向所有端口。为空时表示禁用所有入站重定向（默认为 $ISTIO_INBOUND_PORTS）
  -d: 指定要从重定向到 Envoy 中排除（可选）的入站端口列表，以逗号格式分隔。使用通配符“*” 表示重定向所有入站流量（默认为 $ISTIO_LOCAL_EXCLUDE_PORTS）
  -i: 指定重定向到 Envoy（可选）的 IP 地址范围，以逗号分隔的 CIDR 格式列表。使用通配符 “*” 表示重定向所有出站流量。空列表将禁用所有出站重定向（默认为 $ISTIO_SERVICE_CIDR）
  -x: 指定将从重定向中排除的 IP 地址范围，以逗号分隔的 CIDR 格式列表。使用通配符 “*” 表示重定向所有出站流量（默认为 $ISTIO_SERVICE_EXCLUDE_CIDR）。
```

启动之后netstat查看就发现相关端口了

![图片](D:\学习资料\笔记\k8s\k8s图\143)

咱们上面用到的那行启动命令，直白的讲就是**让 Envoy 代理可以拦截所有的进出 pod 的流量，所有入站（inbound）流量重定向到 15006 端口（sidecar），再拦截应用容器的出站（outbound）流量经过 sidecar 处理（通过 15001 端口监听）后再出站**。厉害了，大有一种不管什么牛鬼蛇神都要来我这里报个到，我来决定你们下一步的走向的架势，就像windows的电脑管家拿到了绝对控制权还不是想弹个啥窗就弹个啥，所有流量都拦截了再进行流量控制、熔断限流、负载均衡等服务治理就大有可为了。
这个命令其实是 iptables 命令的变形，但是换汤不换药，根本原理是一样的，最终也是生成了一系列的 iptables 规则，可以服务器上自己验证下，为了方便标注每一步的过程我按代码格式贴出来了~

```sh
$ iptables -t nat -L 
# PREROUTING 链：将所有入站 TCP 流量跳转到 ISTIO_INBOUND 链上，这里不要好奇怎么又多出来了个上面没讲的链，iptables是可以自定义链的哈，k8s和istio中都有大量的自定义链
Chain PREROUTING (policy ACCEPT)
target         prot opt     source               destination
ISTIO_INBOUND  tcp  --      anywhere             anywhere

# INPUT 链：没有规则，正常跳转到OUTPUT链
Chain INPUT (policy ACCEPT)
 target     prot opt   source               destination

# OUTPUT 链：将所有出站数据包跳转到 ISTIO_OUTPUT 链上
Chain OUTPUT (policy ACCEPT)
target        prot opt    source               destination
ISTIO_OUTPUT  tcp  --     anywhere             anywhere

# POSTROUTING 链：没有规则
Chain POSTROUTING (policy ACCEPT)
target        prot opt    source               destination

# ISTIO_INBOUND 链：将所有目的地为 9080 端口的入站流量重定向到 ISTIO_IN_REDIRECT 链上
Chain ISTIO_INBOUND (1 references)
target             prot opt   source               destination
ISTIO_IN_REDIRECT  tcp  --    anywhere             anywhere      tcp dpt:9080

# ISTIO_IN_REDIRECT 链：将所有的入站流量跳转到本地的 15001 端口，至此成功将流量拦截到了Envoy代理
Chain ISTIO_IN_REDIRECT (1 references)
target     prot opt    source               destination
REDIRECT   tcp  --     anywhere             anywhere             redir ports 15001

# ISTIO_OUTPUT 链：这里需要注意区分两种情况，所有非 localhost 的流量全部转发到 ISTIO_REDIRECT。为了避免流量在该 Pod 中无限循环，所有到 istio-proxy 用户空间的流量(启动命令行中通过参数有进行过滤)都返回到它的调用点中的下一条规则即 OUTPUT 链，因为跳出 ISTIO_OUTPUT 规则之后就进入下一条链 POSTROUTING。如果目的地非 localhost 就跳转到 ISTIO_REDIRECT；如果流量是来自 istio-proxy 用户空间的，那么就跳出该链，返回它的调用链继续执行下一条规则（OUPT 的下一条规则，无需对流量进行处理）；所有的非 istio-proxy 用户空间的目的地是 localhost 的流量就跳转到 ISTIO_REDIRECT
Chain ISTIO_OUTPUT (1 references)
target          prot opt     source               destination
ISTIO_REDIRECT  all  --      anywhere            !localhost
RETURN          all  --      anywhere             anywhere     owner UID match istio-proxy
RETURN          all  --      anywhere             anywhere     owner GID match istio-proxy 
RETURN          all  --      anywhere             localhost
ISTIO_REDIRECT  all  --      anywhere             anywhere

# ISTIO_REDIRECT 链：将所有流量重定向到 Envoy的 15001 端口
Chain ISTIO_REDIRECT (2 references)
target     prot opt    source               destination
REDIRECT   tcp  --     anywhere             anywhere      redirect ports 15001
```

结合官网给的过程图更容易理解些，特别详细了，顺着不同颜色的箭头按数字顺序看就OK

![图片](D:\学习资料\笔记\k8s\k8s图\144)

怎么样，看似高深莫测的 ServiceMesh 服务网格经由这么一看是不是感觉没那么神秘了呐，当然也不是说这就是它的全部了，iptables 只是其流量拦截的基本技术面，其他内容后面有空可以再一起深挖。
与此同时，不知道你有没有发现问题，与传统服务模式相比，这种所有数据包要到达目的地每一步都要经过一系列的拦截代理转发，势必会对服务的性能及负载等产生或多或少的影响，这也是网格技术如此火热却长期有价无市没有普及应用的一个重要原因，当然还有其他的一些原因，同时社区也在竭尽全力的去优化这部分，相信未来可期。



### 四、总结

本文中我们一起梳理了 iptables 的基础概念，如各种规则、四个表、五条链等，又一起根据实际场景探讨了 iptables 命令的使用，而后进一步将其结合时下火热的 ServiceMesh 技术中 Envoy 代理进行流量拦截的原理对 sidecar 模式进行了理解，Kubernetes 中也有 iptables 的深入应用，后面我们找时间再具体谈论这块儿，敬请期待~~

#### 参考资料

[1]NetFilter hooks: *https://wiki.nftables.org/wiki-nftables/index.php/Netfilter_hooks*

[2]iptables四个表五条链: *https://blog.csdn.net/longbei9029/article/details/53056744*





