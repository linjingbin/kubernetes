[toc]



## 持久化存储实践分析



### 1 背景

某银行已经落地容器云平台，并已经承载运行部分关键微服务应用：

1. 越来越多的有状态服务也开始考虑通过该容器运平台进行调度管理和运行，诸如spark、mysql以及各类消息中间件等

2. 目前主要以FC-SAN、vSAN作为应用存储解决方案，大部分应用使用本地物理磁盘

3. 容器云平台目前提供Ceph块存储作为存储解决方案，通过StatefulSet解决容器应用持久化存储需求



### 2 方案需求

1. 存储卷管理需求：容器的一次调度或者故障恢复，会带来卷的挂载和卸载，卷的挂载和卸载一方面需要具有足够的稳定性，同时也需要考虑性能需求，避免应用延迟。

2. 存储卷快照管理需求：传统的存储卷快照策略主要从资源角度进行管理，而容器的存储卷往往动态分配而来，快照策略需要与容器应用备份需求保持一致。

3. 容量扩容需求：存储卷容量需要考虑扩容的能力，最大程度避免对应用运行的影响。

4. 运维管理需求：随着容器有状态应用的增长，对传统存储运维工作也会带来挑战，整体方案需要兼顾运维敏捷和安全。

5. 分布式存储需求：分布式应用的横向扩展，带来存储横向扩展的需求，方案需考虑引入分布式存储适配分布式应用在容器云部署的需求。



### 3 Kubernetes CSI原理及方案设计

#### 3.1 Kubernetes存储动态供应

PV后端的存储可以是iSCSI，RBD，NFS等存储资源，供集群消费，PersistentVolumeClaim（PVC）就是对这些存储资源的请求。PVC消耗的资源是PV。于是，在Kubernetes中Pod可以不直接使用PV，而是通过PVC来使用PV。

通过PVC可以将Pod和PV解耦，Pod不需要知道确切的文件系统和支持它的持久化引擎，可以把PVC理解为存储的抽象，把底层存储细节给隔离了。另一方面，如果使用PV作为存储的话，需要集群管理员事先创建好这些PV，但是如果使用PVC的话则不需要提前准备好PV，而是通过StorageClass把存储资源定义好，Kubernetes会在需要使用的时候动态创建，这种方式称为动态供应 (Dynamic Provisioning)。

![image-20210204175617462](D:\学习资料\笔记\k8s\image-20210204175617462.png)

详细流程分析如下：

1. Pod加载存储卷，请求PVC

2. PVC根据存储类型(此处为rbd)找到存储类StorageClass

3. Provisioner根据StorageClass动态生成一个持久卷PV

4. 持久卷PV和PVC最终形成绑定关系

5. 持久卷PV开始提供给Pod使用



#### 3.2 实现存储动态供应

动态供应比静态供应灵活更多，而且这种方式还解耦了Kubernetes系统的计算层和存储层，更重要的是它给存储供应商提供了可插拔式的开发模型，存储供应商只需要根据这个模型开发相应的卷插件即可为Kubernetes提供存储服务。

有三种方法实现卷插件:

◎ 第一种，Kubernetes内部代码中实现了一些存储插件，用于支持一些主流网络存储，叫作In-treeVolume Plugin。

◎ 第二种 Out-of-tree Provisioner：如果官方的插件不能满足要求，存储供应商可以根据需要去定制或者优化存储插件并集成到Kubernetes系统。

◎ 第三种是容器存储接口CSI (Container Storage Interface)，是Kubernetes对外开放的存储接口，实现这个接口即可集成到Kubernetes系统中。CSI特性在刚过去的12月正式GA，同时社区也宣布未来将不再对In tree/Out of tree继续开发，并将已有功能全部迁移到CSI上，所以对于存储供应商和使用者来说，第三种CSI是更推荐的解决方案。



#### 3.3 CSI 插件原理及设计方案

原理

![image-20210204180136637](D:\学习资料\笔记\k8s\k8s图\image-20210204180136637.png)

CSI 插件要遵循 CSI 的标准实现相关的接口，在实现接口中，可以调用底层存储服务端的 API，CSI 插件在容器平台和存储资源体系结构中起到承上启下的作用。

容器平台可以通过 gRPC 调用 CSI 插件，对接 CSI 接口，支撑容器平台对于存储卷管理的功能，如创建、挂载、删除等操作。而 CSI 插件实现 CSI 接口，是通过调用存储服务端 API 实现相关业务逻辑。

Kubernetes CSI 对接组件和 CSI 存储插件共同构成存储插件层，在 Kubernetes 和存储服务之间。

部署架构

![image-20210204180257872](D:\学习资料\笔记\k8s\k8s图\image-20210204180257872.png)

蓝色部分指的是 K8S 与 CSI 的对接组件，这是由 Kubernetes 团队维护的，绿色部分是开发的 CSI 存储插件。

首先可以从 Controller 服务来看，CSI 插件在部署时分为 Controller 类服务和 Node 类服务，Controller类服务是从存储服务端的角度对存储资源进行相应的操作。

Controller 服务相当于 CSI 插件大脑，只需要部署 Controller 服务的一个副本即可。在 Controller 服务里，封装了 K8S 和 CSI 的对接组件，包括 External Provisioner、External Attacher 和开发的 CSI 存储插件也封装在这个服务里。主要流程是通过 External Provisioner 和 External Attacher，他们会主动获取 K8S API Server 存储的相关资源信息，再调用 Controller 里的 CSI 存储插件所使实现的 CSI 接口，对存储服务端的存储资源进行相关操作。

Node 服务使用 K8S DaemonSet 进行创建，在 Node 服务里，会封装一个 Driver Registrar 这么一个K8S 和 CSI 的对接组件。这个组件的主要功能是将 Node 服务注册。

在 Node 服务中，也有个 CSI 存储插件的容器，在这个容器中，接收到本 Node 上 Kubelet 的调用，Kubelet 可以直接通过 UDS 调用 CSI 插件，Node 服务 CSI 存储插件的容器实现的相关 CSI 接口对主机上的存储卷进行操作，比如对主机上的存储卷进行分区格式化或者将存储卷挂载至某个容器的指定路径下，或者从某个容器的指定路径中进行卸载操作。



### 4 Ceph RBD/Ceph FS provisioner部署案例

根据容器云平台目前提供Ceph块存储作为存储解决方案的背景，可通过Ceph RBD/Ceph FS provi-sioner部署，实现容器平台使用Ceph存储的动态供应。

rbd-provisioner为kubernetes 1.5+版本提供了类似于kubernetes.io/rbd的ceph rbd持久化存储动态配置实现。

**部署rbd-provisioner案例**

参考：https://github.com/kubernetes-incubator/external-storage

部署完rbd-provisioner，还需要创建StorageClass。创建SC前，我们还需要创建相关用户的secret。

创建secret保存client.admin和client.kube用户的key，client.admin和client.kube用户的secret可以放在kube-system namespace，但如果其他namespace需要使用ceph rbd的dynamic provisioning功能的话，要在相应的namespace创建secret来保存client.kube用户key信息；

**说明**

在OpenShift、Rancher、SUSE CaaS以及本Handbook的二进制文件方式部署，在安装好ceph-common软件包的情况下，定义StorageClass时使用kubernetes.io/rbd即可正常使用ceph rbd provisioning功能。

![image-20210205150759335](D:\学习资料\笔记\k8s\k8s图\image-20210205150759335.png)

![image-20210205150925886](D:\学习资料\笔记\k8s\k8s图\image-20210205150925886.png)



### 5 在容器云的卷管理、快照、容量管理方案及适用场景

可参考：https://github.com/container-storage-interface/spec/blob/master/spec.md

#### 5.1 CSI中可实现的控制器服务RPC

**CreateVolume**

如果控制器插件具有CREATE_DELETE_VOLUME控制器功能，则必须实现此RPC调用。该RPC将由CO调用，以代表用户（将其作为块设备或已挂载的文件系统使用）来供应新卷。

此操作必须是幂等的。如果已经存在与指定卷“名称”相对应的卷，可以从accessibility_requirements访问该卷，并且与CreateVolumeRequest中指定的capacity_range，volume_capabilities和parameters兼容，则插件必须回复“0单击确定，并带有相应的CreateVolumeResponse。

插件可以创建3种类型的卷：

◎ 空卷，当插件支持CREATE_DELETE_VOLUME可选功能时。

◎ 从现有快照中，当插件支持CREATE_DELETE_VOLUME和CREATE_DELETE_SNAPSHOT可选功能时。

◎ 从现有的卷。当插件支持克隆并报告可选功能CREATE_DELETE_VOLUME和CLONE_VOLUME时。

**DeleteVolume**

如果控制器插件具有CREATEDELETEVOLUME功能，则必须实现此RPC调用。CO将调用此RPC取消供应卷。

此操作必须是幂等的。如果对应于指定volume_id的卷不存在或与该卷关联的工件不再存在，则插件务必回复0 OK。

如果Controller Plugin支持在不影响其现有快照的情况下删除卷，则这些快照必须仍然可以完全正常运行，并且可以作为新卷的源，并且在删除卷后出现在ListSnapshot调用中。

**ControllerPublishVolume**

如果控制器插件具有PUBLISHUNPUBLISHVOLUME控制器功能，则必须实现此RPC调用。当想要将使用该卷的工作负载放置到节点上时，CO将调用此RPC。插件应该执行使卷在给定节点上可用所需的工作。插件不得假定此RPC将在将使用该卷的节点上执行。

**ControllerUnpublishVolume**

这个RPC是ControllerPublishVolume的反向操作。在卷上的所有NodeUnstageVolume和NodeUnpublishVolume被调用并成功之后，必须调用它。插件应该执行使卷准备好被其他节点消耗的必要工作。插件不得假定此RPC将在先前使用该卷的节点上执行。

**ValidateVolumeCapabilities**

控制器插件必须实现此RPC调用。CO将调用此RPC，以检查预配置的卷是否具有CO所需的所有功能。仅当支持请求中指定的所有卷功能时，此RPC调用才应返回“已确认”（请参见下面的注意事项）。此操作必须是幂等的。

**ListVolumes**

如果控制器插件具有LIST_VOLUMES功能，则必须实现此RPC调用。插件应返回有关它所知道的所有卷的信息。如果在CO同时通过ListVolumes结果分页的同时创建和/或删除卷，则CO可能见证列表中的重复卷，不见证现有的卷，或两者兼而有之。当通过多次调用ListVolumes遍历卷列表时，CO不会期望所有卷具有一致的“视图”。



#### 5.2 快照

要实现快照特性，CSI驱动程序必须添加对附加控制器功能CREATE_DELETE_SNAPSHOT和list_snapshot的支持，并实现附加控制器RPCs: CreateSnapshot、DeleteSnapshot和listsnapshot。

![image-20210205152513558](D:\学习资料\笔记\k8s\k8s图\image-20210205152513558.png)

**使用Kubernetes创建一个新的快照**

一旦定义了VolumeSnapshotClass对象，并且有了要快照的卷，就可以通过创建VolumeSnapshot对象创建新的快照。

快照的源指定创建快照的卷。它有两个参数:

◎ kind - 必须是PersistentVolumeClaim

◎ name - PVC API对象名称

假定快照卷的NameSpace与VolumeSnapshot对象的NameSpace相同。

![image-20210205152609826](D:\学习资料\笔记\k8s\k8s图\image-20210205152609826.png)

在VolumeSnapshot规范中，用户可以指定VolumeSnapshotClass，该类包含关于应该使用哪个CSI驱动程序创建快照的信息。当创建VolumeSnapshot对象时，参数fakeSnapshotOption: foo和VolumeSnapshotClass中引用的任何秘密都会传递给CSI plugin com.example。通过CreateSnapshot调用驱动csi。

**使用Kubernetes导入现有快照**

我们可以通过手动创建一个VolumeSnapshotContent对象来表示现有快照，从而将现有快照导入Kubernetes。因为VolumeSnapshotContent不是NameSpace API的对象，所以只有系统管理员才有权创建它。

创建了VolumeSnapshotContent对象之后，用户可以创建指向VolumeSnapshot对象的VolumeSnapshot对象。外部快照控制器将在验证快照是否存在以及VolumeSnapshot和VolumeSnapshotContent对象之间的绑定是否正确之后，将快照标记为ready。一旦绑定，快照就可以在Kubernetes中使用了。

VolumeSnapshotContent对象应该使用以下字段创建，以表示预先准备好的快照：

◎ csiVolumeSnapshotSource- 快照识别信息。

◎ snapshotHandle - 快照的名称/标识符。这个字段是必需的。

◎ driver-用于处理此卷的CSI驱动程序。这个字段是必需的。它必须匹配快照控制器中的快照器名称。

◎ creationTime和restoreSize - 这些字段对于预先准备的卷是不需要的。外部快照控制器将在创建后自动更新它们。

◎ volumeSnapshotRef-在此对象之前，应先绑定VolumeSnapshot对象。

◎ name和namespace - 它指定内容绑定到的VolumeSnapshot对象的名称和命名空间。

◎ UID - 这些字段对于预先准备的卷是不需要的。外部快照控制器将在绑定后自动更新字段。如果用户指定UID字段，他/她必须确保它与绑定快照的UID匹配。如果指定的UID不匹配绑定快照的UID，则将该内容视为孤立对象，控制器将删除它及其关联的快照。

◎ snapshotClassName - 这个字段是可选的。外部快照控制器将在绑定后自动更新字段。

![image-20210205153553028](D:\学习资料\笔记\k8s\k8s图\image-20210205153553028.png)

创建VolumeSnapshot对象，允许用户使用快照：

◎ snapshotClassName - 卷快照类的名称。这个字段是可选的。如果设置了快照，快照类中的snapshotter字段必须匹配快照控制器的snapshotter名称。如果没有设置，快照控制器将尝试找到一个默认快照类。

◎ snapshotContentName - 卷快照内容的名称。这个字段对于预先准备的卷是必需的。

![image-20210205153654549](D:\学习资料\笔记\k8s\k8s图\image-20210205153654549.png)

一旦创建了这些对象，快照控制器将把它们绑定在一起，并将Ready (under Status)字段设置为True，以指示快照可以使用了。

使用Kubernetes从快照中提供一个新卷要提供预填充快照对象数据的新卷，请使用PersistentVolumeClaim中的新数据源字段。它有三个参数:

◎ name - 表示要用作源的快照的VolumeSnapshot对象的名称

◎ kind - 必须是 VolumeSnapshot

◎ apiGroup - 必须是 snapshot.storage.k8s.io

假定源VolumeSnapshot对象的NameSpace与PersistentVolumeClaim对象的NameSpace相同。

![image-20210205153817318](D:\学习资料\笔记\k8s\k8s图\image-20210205153817318.png)

当创建PersistentVolumeClaim对象时，它将触发一个新的卷的供应，该卷预先填充了来自指定快照的数据。



### 6 分布式存储的适用场景

体现分布式存储技术特点：多副本一致性、容灾性、扩展性、分级存储、高性能，及对比传统块存储和文件存储的收益；从业务收益、IT运维、TCO分析、社会效果、等等

#### 6.1 主流分布式存储对比

**HDFS**

主要用于大数据的存储场景，是Hadoop大数据架构中的存储组件。HDFS在开始设计的时候，就已经明确的它的应用场景，就是为大数据服务。主要的应用场景有：

◎ 对大文件存储的性能比较高，例如几百兆，几个G的大文件。因为HDFS采用的是以元数据的方式进行文件管理，而元数据的相关目录和块等信息保存在NameNode的内存中，文件数量的增加会占用大量的NameNode内存。如果存在大量的小文件，会占用大量内存空间，引起整个分布式存储性能下降，所以尽量使用HDFS存储大文件比较合适。

◎ 适合低写入，多次读取的业务。就大数据分析业务而言，其处理模式就是一次写入、多次读取，然后进行数据分析工作，HDFS的数据传输吞吐量比较高，但是数据读取延时比较差，不适合频繁的数据写入。

◎ HDFS采用多副本数据保护机制，使用普通的X86服务器就可以保障数据的可靠性，不推荐在虚拟化环境中使用。

**Ceph**

Ceph是一个开源的存储项目，是目前应用最广泛的开源分布式存储系统，已得到众多厂商的支持，许多超融合系统的分布式存储都是基于Ceph深度定制。而且Ceph已经成为LINUX系统和OpenStack的“标配”，用于支持各自的存储系统。Ceph可以提供对象存储、块设备存储和文件系统存储服务。

Ceph没有采用HDFS的元数据寻址的方案，而且采用CRUSH算法，数据分布均衡，并行度高。而且在支持块存储特性上，数据可以具有强一致性，可以获得传统集中式存储的使用体验。对象存储服务，Ceph支持Swift和S3的API接口。在块存储方面，支持精简配置、快照、克隆。在文件系统存储服务方面，支持Posix接口，支持快照。但是目前Ceph支持文件存储的性能一般，部署稍显复杂，性能也稍弱，一般都将Ceph应用于块和对象存储。

Ceph是去中心化的分布式解决方案，需要提前做好规划设计，对技术团队的要求能力比较高。特别是在Ceph扩容时，由于其数据分布均衡的特性，会导致整个存储系统性能的下降。

**GFS**

GFS是google分布式文件存储，是为了存储海量搜索数据而设计的专用文件系统。和HDFS 比较类似，而且HDFS系统最早就是根据GFS的概念进行设计实现的。

GFS同样适合大文件读写，不合适小文件存储。适合处理大量的文件读取，需要高带宽，而且数据访问延时不敏感的搜索类业务。同样不适合多用户同时写入。

HDFS和GFS的主要区别是，对GFS中关于数据的写入进行了一些改进，同一时间只允许一个客户端写入或追加数据。而GFS  是支持并发写入的。这样会减少同时写入带来的数据一致性问题，在写入流程上，架构相对比较简单，容易实现。

**GPFS**

GPFS是 IBM 的共享文件系统，它是一个并行的磁盘文件系统，可以保证在资源组内的所有节点可以并行访问整个文件系统。GPFS 允许客户共享文件，而这些文件可能分布在不同节点的不同硬盘上。GPFS提供了许多标准的 UNIX 文件系统接口，允许应用不需修改或者重新编辑就可以在其上运行。

GPFS和其他分布式存储不同的是，GPFS是由网络共享磁盘（NSD）和物理磁盘组成。网络共享磁盘（NSD）是由物理磁盘映射出来的虚拟设备，与磁盘之间是一一对应的关系。

GPFS文件系统允许在同一个节点内的多个进程使用标准的UNIX文件系统接口并行的访问相同文件进行读写，性能比较高。GPFS支持传统集中式存储的仲裁机制和文件锁，保证数据安全和数据的正确性，这是其他分布式存储系统无法比拟的。GPFS主要用于IBM 小型机和UNIX系统的文件共享和数据容灾等场景。

**Swift**

Swift主要面向的是对象存储。和Ceph提供的对象存储服务类似。主要用于解决非结构化数据存储问题。它和Ceph的对象存储服务的主要区别是：

◎ 客户端在访问对象存储系统服务时，Swift要求客户端必须访问Swift网关才能获得数据。而Ceph使用一个运行在每个存储节点上的OSD（对象存储设备）获取数据信息，没有一个单独的入口点，比Swift更灵活一些。

◎ 在数据一致性方面，Swift的数据是最终一致，在海量数据的处理效率上要高一些，但是主要面向对数据一致性要求不高，但是对数据处理效率要求比较高的对象存储业务。而Ceph是始终跨集群强一致性。



#### 6.2 传统存储架构的局限性和分布式存储的优点

传统存储架构的局限性主要体现在以下几个方面：

◎ 横向扩展性较差：受限于前端控制器的对外服务能力，纵向扩展磁盘数量无法有效提升存储设备对外提供服务的能力。同时，前端控制器横向扩展能力非常有限，业界最多仅能实现几个控制器的横向。因此，前端控制器成为整个存储性能的瓶颈。

◎ 不同厂家传统存储之间的差异性带来的管理问题：不同厂商设备的管理和使用方式各有不同，由于软硬件紧耦合、管理接口不统一等限制因素无法做到资源的统一管理和弹性调度，也会带来存储利用率较低的现象。因此，不同存储的存在影响了存储使用的便利性和利用率。

◎ 分布式存储往往采用分布式的系统结构，利用多台存储服务器分担存储负荷，利用位置服务器定位存储信息。它不但提高了系统的可靠性、可用性和存取效率，还易于扩展，将通用硬件引入的不稳定因素降到最低。



分布式存储主要优点如下：

◎ 高性能：一个具有高性能的分布式存户通常能够高效地管理读缓存和写缓存，并且支持自动的分级存储。分布式存储通过将热点区域内数据映射到高速存储中，来提高系统响应速度；一旦这些区域不再是热点，那么存储系统会将它们移出高速存储。而写缓存技术则可使配合高速存储来明显改变整体存储的性能，按照一定的策略，先将数据写入高速存储，再在适当的时间进行同步落盘。

◎ 弹性扩展：得益于合理的分布式架构，分布式存储可预估并且弹性扩展计算、存储容量和性能。分布式存储的水平扩展有以下几个特性：节点扩展后，旧数据会自动迁移到新节点，实现负载均衡，避免单点过热的情况出现；水平扩展只需要将新节点和原有集群连接到同一网络，整个过程不会对业务造成影响；当节点被添加到集群，集群系统的整体容量和性能也随之线性扩展，此后新节点的资源就会被管理平台接管，被用于分配或者回收。

◎ 支持分级存储：由于通过网络进行松耦合链接，分布式存储允许高速存储和低速存储分开部署，或者任意比例混布。在不可预测的业务环境或者敏捷应用情况下，分层存储的优势可以发挥到最佳。解决了目前缓存分层存储最大的问题是当性能池读不命中后，从冷池提取数据的粒度太大，导致延迟高，从而给造成整体的性能的抖动的问题。

◎ 多副本的一致性：与传统的存储架构使用RAID模式来保证数据的可靠性不同，分布式存储采用了多副本备份机制。在存储数据之前，分布式存储对数据进行了分片，分片后的数据按照一定的规则保存在集群节点上。为了保证多个数据副本之间的一致性，分布式存储通常采用的是一个副本写入，多个副本读取的强一致性技术，使用镜像、条带、分布式校验等方式满足租户对于可靠性不同的需求。在读取数据失败的时候，系统可以通过从其他副本读取数据，重新写入该副本进行恢复，从而保证副本的总数固定；当数据长时间处于不一致状态时，系统会自动数据重建恢复，同时租户可设定数据恢复的带宽规则，最小化对业务的影响。

◎ 容灾与备份：在分布式存储的容灾中，一个重要的手段就是多时间点快照技术，使得用户生产系统能够实现一定时间间隔下的各版本数据的保存。特别值得一提的是，多时间点快照技术支持同时提取多个时间点样本同时恢复，这对于很多逻辑错误的灾难定位十分有用，如果用户有多台服务器或虚拟机可以用作系统恢复，通过比照和分析，可以快速找到哪个时间点才是需要回复的时间点，降低了故障定位的难度，缩短了定位时间。这个功能还非常有利于进行故障重现，从而进行分析和研究，避免灾难在未来再次发生。多副本技术，数据条带化放置，多时间点快照和周期增量复制等技术为分布式存储的高可靠性提供了保障。

◎ 存储系统标准化：随着分布式存储的发展，存储行业的标准化进程也不断推进，分布式存储优先采用行业标准接口进行存储接入。在平台层面，通过将异构存储资源进行抽象化，将传统的存储设备级的操作封装成面向存储资源的操作，从而简化异构存储基础架构的操作，以实现存储资源的集中管理，并能够自动执行创建、变更、回收等整个存储生命周期流程。基于异构存储整合的功能，用户可以实现跨不同品牌、介质地实现容灾，如用中低端阵列为高端阵列容灾，用不同磁盘阵列为闪存阵列容灾等等，从侧面降低了存储采购和管理成本。



### 7 容器云的NBU数据备份

Veritas有推出可作为容器部署的NetBackup Client。NetBackup Client Container利用NetBackup策略来运行备份。根据所需的保护级别，NetBackup Client Container 可通过以下方式保护容器化应用程序：

◎ 保护存储在永久卷上的应用程序数据

◎ 使用暂存区域保护应用程序数据

◎ 将应用程序部署在同一应用程序容器中，以执行应用程序一致性备份

![image-20210205165343639](D:\学习资料\笔记\k8s\k8s图\image-20210205165343639.png)

#### 7.1 获取NetBackup Client image

Veritas NetBackup为Docker认证的容器备份解决方案：

https://www.veritas.com/protection/netbackup/resources

https://hub.docker.com/_/veritas-netbackup-client?tab=description

![image-20210205170653821](D:\学习资料\笔记\k8s\k8s图\image-20210205170653821.png)



#### 7.2 部署方式一（Sidecar部署方式）

NetBackup Client Container可与应用程序共存并从应用程序运行所在的节点操作。为此，为每个应用程序Pod部署一款NetBackup Client Container作为Sidecar。

在此方法中：

◎ NetBackup Client Container作为应用程序Pod的一部分一起运行。这可确保应用程序和NetBackupClient Container的生命周期相同。

◎ 需要保护的PV卷必须同时可以被应用程序和NetBackup Client Container访问。

该解决方案可：

◎ 简化备份管理员的管理操作

◎ 优化网络备份性能

◎ 有效利用NetBackup核心技术，例如Accelerator、Client Direct Backup等

◎ 通过添加备份策略（policy）来分类不同应用程序数据

◎ 一致的标准客户端数据恢复操作。

下图为Sidecar 的部署模式：

![image-20210205170853730](D:\学习资料\笔记\k8s\k8s图\image-20210205170853730.png)



#### 7.3 部署方式二（单一POD部署方式）

NetBackup Client Container 从可访问各卷的单个点保护永久存储。由此，仅需部署一个 NetBackup Client Container 即可保护多个应用程序 Pod。

在以下情形中，可部署单一NetBackup Client Container保护多个应用程序Pod：

◎ 应用程序所有者不希望将NetBackup Client Container容器纳入到自己的Pod中

◎ 群集可用的外部 IP 接口的数量有限

◎ 为了尽可能减少NetBackup在群集中的占用空间

可在Kubernetes群集的每个节点部署一款NetBackup Client Container，或每个群集部署一款NetBackup Client Container，具体取决于：

◎ PV的访问模式：“ReadWriteOnce”卷仅在一个节点上可用，而“ReadWriteMany”卷在群集的多个节点上可用

◎ 所需的客户端吞吐量。

在此方法中，在NetBackup Client Container中装入所有需要保护的卷（PV）。Pod启动后无法装入卷。管理员必须提前了解需要保护的卷，且必须在创建Pod之前在NetBackup Client Container模板中定义这些卷。所有受保护的应用程序被记录为相同的NetBackup客户端名称。因此，应该为每一个不同的应用程序创建一项NetBackup策略(Backup Policy)，并为该类应用程序指定特定的关键字，数据恢复时将可按该关键字进行快速查找。

![image-20210205171053631](D:\学习资料\笔记\k8s\k8s图\image-20210205171053631.png)



#### 7.4 部署方式三（Dump & Sweep部署方式）

此外，还可通过装入转储卷在第二种部署模式中使用“Dump & Sweep”方式。

在“Dump & Sweep”方式中，需要保护的应用程序Pod不仅装入数据卷，还装入转储卷。应用程序所有者将应用程序数据转储到转储卷。NetBackup使用NetBackup标准策略定期清除转储卷。

所有受保护的应用程序均记录相同的NetBackup客户端名称。因此，为每个应用程序创建一项 NetBackup 策略，并为该应用程序指定特定的关键字。

下图说明了为多个Pod 部署一款NetBackup Container时使用的“Dump & Sweep”方法。

![image-20210205171209985](D:\学习资料\笔记\k8s\k8s图\image-20210205171209985.png)



#### 8 容器云存储整体架构面临的挑战与经验

**性能**

在容器的使用场景，通常会有非常多的Pod或者在Pod上会挂载较多的卷，需要消耗一定的资源，特别如今云原生时代，非常适合要求轻量，快速的容器云平台，一分钟内就能启动上百个pod，对存储IO性能，并发创建能力也有很高要求。

**跨数据中心双活等问题**

稳定性、数据一致性、网络故障等等问题带来的运维复杂度。

私有云存储大都采用了集群分离方式部署的高可用架构，而跨数据中心的云存储双活，要在两端有效部署私有云存储的双活，需要注意的问题也要复杂很多，比如：如何保证在实施了双活后，私有云存储中数据可以足够分散；如何实现存储数据的容量均衡，使每个存储设备数据达到另一个相对平等的水平。





## Rook 1.5.1 部署 Ceph



### 一、Rook概述

#### 1.1 Rook简介

Rook 是一个开源的cloud-native storage编排, 提供平台和框架；为各种存储解决方案提供平台、框架和支持，以便与云原生环境本地集成。目前主要专用于Cloud-Native环境的文件、块、对象存储服务。它实现了一个自我管理的、自我扩容的、自我修复的分布式存储服务。

Rook支持自动部署、启动、配置、分配（provisioning）、扩容/缩容、升级、迁移、灾难恢复、监控，以及资源管理。为了实现所有这些功能，Rook依赖底层的容器编排平台，例如 kubernetes、CoreOS 等。。

Rook 目前支持Ceph、NFS、Minio Object Store、Edegefs、Cassandra、CockroachDB 存储的搭建。

项目地址：https://github.com/rook/rook

网站：https://rook.io/



#### 1.2 Rook组件

Rook的主要组件有三个，功能如下：

1. Rook Operator

2. - Rook与Kubernetes交互的组件
   - 整个Rook集群只有一个

3. Agent or Driver

   已经被淘汰的驱动方式，在安装之前，请确保k8s集群版本是否支持CSI，如果不支持，或者不想用CSI，选择flex.

   默认全部节点安装，你可以通过 node affinity 去指定节点

4. - Ceph CSI Driver
   - Flex Driver

5. Device discovery

   发现新设备是否作为存储，可以在配置文件`ROOK_ENABLE_DISCOVERY_DAEMON`设置 enable 开启。



#### 1.3 Rook & Ceph框架

Rook 如何集成在kubernetes 如图：

![image-20210207171217299](D:\学习资料\笔记\k8s\k8s图\image-20210207171217299.png)

使用Rook部署Ceph集群的架构图如下：

![image-20210207171301803](D:\学习资料\笔记\k8s\k8s图\image-20210207171301803.png)



部署的Ceph系统可以提供下面三种`Volume Claim`服务：

- Block Storage：目前最稳定；
- FileSystem：需要部署MDS，有内核要求；
- Object：部署RGW；



### 二、ROOK 部署

#### 2.1 准备工作

##### 2.1.1 版本要求

kubernetes v.11 以上



##### 2.1.2 存储要求

> rook部署的ceph 是不支持lvm direct直接作为osd存储设备的，如果想要使用lvm，可以使用pvc的形式实现。方法在后面的ceph安装会提到

为了配置 Ceph 存储集群，至少需要以下本地存储选项之一：

- 原始设备（无分区或格式化的文件系统）
- 原始分区（无格式文件系统） 可以 lsblk -f查看，如果 FSTYPE不为空说明有文件系统
- 可通过 block 模式从存储类别获得 PV



##### 2.1.3 系统要求

本次安装环境

- kubernetes 1.18
- centos 7.8
- kernel 5.4.65-200.el7.x86_64
- calico 3.16



###### 2.1.3.1 需要安装lvm包

```sh
sudo yum install -y lvm2
```



###### 2.1.3.2 内核要求

**RBD**

一般发行版的内核都编译有，但你最好确定下：

```sh
foxchan@~$ lsmod|grep rbd
rbd                   114688  0 
libceph               368640  1 rbd
```

可以用以下命令放到开机启动项里

```sh
cat > /etc/sysconfig/modules/rbd.modules << EOF
modprobe rbd
EOF
```

**CephFS**

如果你想使用cephfs,内核最低要求是4.17。



#### 2.2 部署ROOK

Github上下载Rook最新release

```sh
git clone --single-branch --branch v1.5.1 https://github.com/rook/rook.gits
```

安装公共部分

```sh
cd rook/cluster/examples/kubernetes/ceph
kubectl create -f crds.yaml -f common.yaml
```

安装operator

```sh
kubectl apply -f operator.yaml
```

> 如果放到生产环境，请提前规划好。operator的配置在ceph安装后不能修改，否则rook会删除集群并重建。

修改内容如下：

```sh
# 启用cephfs 
ROOK_CSI_ENABLE_CEPHFS: "true"
# 开启内核驱动替换ceph-fuse
CSI_FORCE_CEPHFS_KERNEL_CLIENT: "true"
#修改csi镜像为私有仓，加速部署时间
ROOK_CSI_CEPH_IMAGE: "harbor.foxchan.com/google_containers/cephcsi/cephcsi:v3.1.2"
ROOK_CSI_REGISTRAR_IMAGE: "harbor.foxchan.com/google_containers/k8scsi/csi-node-driver-registrar:v2.0.1"
ROOK_CSI_RESIZER_IMAGE: "harbor.foxchan.com/google_containers/k8scsi/csi-resizer:v1.0.0"
ROOK_CSI_PROVISIONER_IMAGE: "harbor.foxchan.com/google_containers/k8scsi/csi-provisioner:v2.0.0"
ROOK_CSI_SNAPSHOTTER_IMAGE: "harbor.foxchan.com/google_containers/k8scsi/csi-snapshotter:v3.0.0"
ROOK_CSI_ATTACHER_IMAGE: "harbor.foxchan.com/google_containers/k8scsi/csi-attacher:v3.0.0"
# 可以设置NODE_AFFINITY 来指定csi 部署的节点
# 我把plugin 和 provisioner分开了，具体调度方式看你集群资源。
CSI_PROVISIONER_NODE_AFFINITY: "app.rook.role=csi-provisioner"
CSI_PLUGIN_NODE_AFFINITY: "app.rook.plugin=csi"
#修改metrics端口，可以不改，我因为集群网络是host，为了避免端口冲突
# Configure CSI CSI Ceph FS grpc and liveness metrics port
CSI_CEPHFS_GRPC_METRICS_PORT: "9491"
CSI_CEPHFS_LIVENESS_METRICS_PORT: "9481"
# Configure CSI RBD grpc and liveness metrics port
CSI_RBD_GRPC_METRICS_PORT: "9490"
CSI_RBD_LIVENESS_METRICS_PORT: "9480"
# 修改rook镜像，加速部署时间
image: harbor.foxchan.com/google_containers/rook/ceph:v1.5.1
# 指定节点做存储
        - name: DISCOVER_AGENT_NODE_AFFINITY
          value: "app.rook=storage"
# 开启设备自动发现
        - name: ROOK_ENABLE_DISCOVERY_DAEMON
          value: "true"
```



#### 2.3 部署ceph集群

cluster.yaml文件里的内容需要修改，一定要适配自己的硬件情况，**请详细阅读配置文件里的注释，避免我踩过的坑。**

修改内容如下：

> 此文件的配置，除了增删osd设备外，其他的修改都要重装ceph集群才能生效，所以请提前规划好集群。如果修改后不卸载ceph直接apply，会触发ceph集群重装，导致集群异常挂掉

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
# 命名空间的名字，同一个命名空间只支持一个集群
  name: rook-ceph
  namespace: rook-ceph
spec:
# ceph版本说明
# v13 is mimic, v14 is nautilus, and v15 is octopus.
  cephVersion:
#修改ceph镜像，加速部署时间
    image: harbor.foxchan.com/google_containers/ceph/ceph:v15.2.5
# 是否允许不支持的ceph版本
    allowUnsupported: false
#指定rook数据在节点的保存路径
  dataDirHostPath: /data/rook
# 升级时如果检查失败是否继续
  skipUpgradeChecks: false
# 从1.5开始，mon的数量必须是奇数
  mon:
    count: 3
# 是否允许在单个节点上部署多个mon pod
    allowMultiplePerNode: false
  mgr:
    modules:
    - name: pg_autoscaler
      enabled: true
# 开启dashboard，禁用ssl，指定端口是7000，你可以默认https配置。我是为了ingress配置省事。
  dashboard:
    enabled: true
    port: 7000
    ssl: false
# 开启prometheusRule
  monitoring:
    enabled: true
# 部署PrometheusRule的命名空间，默认此CR所在命名空间
    rulesNamespace: rook-ceph
# 开启网络为host模式，解决无法使用cephfs pvc的bug
  network:
    provider: host
# 开启crash collector，每个运行了Ceph守护进程的节点上创建crash collector pod
  crashCollector:
    disable: false
# 设置node亲缘性，指定节点安装对应组件
  placement:
    mon:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
          - matchExpressions:
            - key: ceph-mon
              operator: In
              values:
              - enabled

    osd:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
          - matchExpressions:
            - key: ceph-osd
              operator: In
              values:
              - enabled

    mgr:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
          - matchExpressions:
            - key: ceph-mgr
              operator: In
              values:
              - enabled 
# 存储的设置，默认都是true，意思是会把集群所有node的设备清空初始化。
  storage: # cluster level storage configuration and selection
    useAllNodes: false     #关闭使用所有Node
    useAllDevices: false   #关闭使用所有设备
    nodes:
    - name: "192.168.1.162"  #指定存储节点主机
      devices:
      - name: "nvme0n1p1"    #指定磁盘为nvme0n1p1
    - name: "192.168.1.163"
      devices:
      - name: "nvme0n1p1"
    - name: "192.168.1.164"
      devices:
      - name: "nvme0n1p1"
    - name: "192.168.1.213"
      devices:
      - name: "nvme0n1p1"
```

更多 cluster 的 CRD 配置参考：

- https://github.com/rook/rook/blob/master/Documentation/ceph-cluster-crd.md

执行安装

```sh
kubectl apply -f cluster.yaml
# 需要等一段时间，所有pod都已正常启动
[foxchan@k8s-master ceph]$ kubectl get pods -n rook-ceph 
NAME                                                      READY   STATUS      RESTARTS   AGE
csi-cephfsplugin-b5tlr                                    3/3     Running     0          19h
csi-cephfsplugin-mjssm                                    3/3     Running     0          19h
csi-cephfsplugin-provisioner-5cf5ffdc76-mhdgz             6/6     Running     0          19h
csi-cephfsplugin-provisioner-5cf5ffdc76-rpdl8             6/6     Running     0          19h
csi-cephfsplugin-qmvkc                                    3/3     Running     0          19h
csi-cephfsplugin-tntzd                                    3/3     Running     0          19h
csi-rbdplugin-4p75p                                       3/3     Running     0          19h
csi-rbdplugin-89mzz                                       3/3     Running     0          19h
csi-rbdplugin-cjcwr                                       3/3     Running     0          19h
csi-rbdplugin-ndjcj                                       3/3     Running     0          19h
csi-rbdplugin-provisioner-658dd9fbc5-fwkmc                6/6     Running     0          19h
csi-rbdplugin-provisioner-658dd9fbc5-tlxd8                6/6     Running     0          19h
prometheus-rook-prometheus-0                              2/2     Running     1          3d17h
rook-ceph-mds-myfs-a-5cbcdc6f9c-7mdsv                     1/1     Running     0          19h
rook-ceph-mds-myfs-b-5f4cc54b87-m6m6f                     1/1     Running     0          19h
rook-ceph-mgr-a-f98d4455b-bwhw7                           1/1     Running     0          20h
rook-ceph-mon-a-5d445d4b8d-lmg67                          1/1     Running     1          20h
rook-ceph-mon-b-769c6fd76f-jrlc8                          1/1     Running     0          20h
rook-ceph-mon-c-6bfd8954f5-tbsnd                          1/1     Running     0          20h
rook-ceph-operator-7d8cc65dc-8wtl8                        1/1     Running     0          20h
rook-ceph-osd-0-c558ff759-bzbgw                           1/1     Running     0          20h
rook-ceph-osd-1-5c97d69d78-dkxbb                          1/1     Running     0          20h
rook-ceph-osd-2-7dddc7fd56-p58mw                          1/1     Running     0          20h
rook-ceph-osd-3-65ff985c7d-9gfgj                          1/1     Running     0          20h
rook-ceph-osd-prepare-192.168.1.213-pw5gr                 0/1     Completed   0          19h
rook-ceph-osd-prepare-192.168.1.162-wtkm8                 0/1     Completed   0          19h
rook-ceph-osd-prepare-192.168.1.163-b86r2                 0/1     Completed   0          19h
rook-ceph-osd-prepare-192.168.1.164-tj79t                 0/1     Completed   0          19h
rook-discover-89v49                                       1/1     Running     0          20h
rook-discover-jdzhn                                       1/1     Running     0          20h
rook-discover-sl9bv                                       1/1     Running     0          20h
rook-discover-wg25w                                       1/1     Running     0          20h
```



#### 2.4 增删osd

##### 2.4.1 添加相关label

```sh
kubectl label nodes 192.168.1.165 app.rook=storage
kubectl label nodes 192.168.1.165 ceph-osd=enabled
```



##### 2.4.2 修改cluster.yaml

```yaml
    nodes:
    - name: "192.168.1.162"
      devices:
      - name: "nvme0n1p1" 
    - name: "192.168.1.163"
      devices:
      - name: "nvme0n1p1"
    - name: "192.168.1.164"
      devices:
      - name: "nvme0n1p1"
    - name: "192.168.17.213"
      devices:
      - name: "nvme0n1p1"
  #添加165的磁盘信息 
    - name: "192.168.1.165"
      devices:
      - name: "nvme0n1p1"    
```



##### 2.4.3 apply cluster.yaml

```
kubectl apply -f cluster.yaml
```



##### 2.4.4 删除osd

cluster.yaml去掉相关节点，再apply



#### 2.5 安装dashboard

> 这是我自己的traefik ingress，yaml目录里有很多dashboard暴露方式，自行选择

dashboard已经在前述的步骤中包含了，这里只需要把dashboard service的服务暴露出来。有多种方法，我使用的是ingress的方式来暴露：

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: Ingre***oute
metadata:
  name: traefik-ceph-dashboard
  annotations:
    kubernetes.io/ingress.class: traefik-v2.3
spec:
  entryPoints:
    - web
  routes:
  - match: Host(`ceph.foxchan.com`)
    kind: Rule
    services:
    - name: rook-ceph-mgr-dashboard
      namespace: rook-ceph
      port: 7000
    middlewares:
      - name: gs-ipwhitelist
```

登录 dashboard 需要安全访问。Rook 在运行 Rook Ceph 集群的名称空间中创建一个默认用户，admin 并生成一个称为的秘密`rook-ceph-dashboard-admin-password`

要检索生成的密码，可以运行以下命令：

```sh
kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}" | base64 --decode && echo
```



#### 2.6 安装toolbox

执行下面的命令：

```sh
kubectl apply -f toolbox.yaml
```

成功后，可以使用下面的命令来确定toolbox的pod已经启动成功：

```sh
kubectl -n rook-ceph get pod -l "app=rook-ceph-tools"
```

然后可以使用下面的命令登录该pod，执行各种ceph命令：

```sh
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash
```

比如：

- `ceph status`
- `ceph osd status`
- `ceph df`
- `rados df`

删除toolbox

```sh
kubectl -n rook-ceph delete deploy/rook-ceph-tools
```



#### 2.7 prometheus监控

> 监控部署很简单，利用Prometheus Operator，独立部署一套prometheus

**安装prometheus operator**

```sh
kubectl apply -f https://raw.githubusercontent.com/coreos/prometheus-operator/v0.40.0/bundle.yaml
```

**安装prometheus**

```sh
git clone --single-branch --branch v1.5.1 https://github.com/rook/rook.git
cd rook/cluster/examples/kubernetes/ceph/monitoring
kubectl create -f service-monitor.yaml
kubectl create -f prometheus.yaml
kubectl create -f prometheus-service.yaml
```

默认是nodeport方式暴露

```sh
echo "http://$(kubectl -n rook-ceph -o jsonpath={.status.hostIP} get pod prometheus-rook-prometheus-0):30900"
```

**开启Prometheus Alerts**

> 此操作必须在ceph集群安装之前

安装rbac

```sh
kubectl create -f cluster/examples/kubernetes/ceph/monitoring/rbac.yaml
```

确保cluster.yaml 开启

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
[...]
spec:
[...]
  monitoring:
    enabled: true
    rulesNamespace: "rook-ceph"
[...]
```

**Grafana Dashboards**

Grafana 版本大于等于 7.2.0

推荐一下dashboard

- Ceph - Cluster
- Ceph - OSD (Single)
- Ceph - Pools



#### 2.8 删除ceph集群

> 删除ceph集群前，请先清理相关pod

删除块存储和文件存储

```sh
kubectl delete -n rook-ceph cephblockpool replicapool
kubectl delete storageclass rook-ceph-block
kubectl delete -f csi/cephfs/filesystem.yaml
kubectl delete storageclass csi-cephfs rook-ceph-block
```

删除operator和相关crd

```sh
kubectl delete -f operator.yaml
kubectl delete -f common.yaml
kubectl delete -f crds.yaml
```

清除主机上的数据

删除Ceph集群后，在之前部署Ceph组件节点的/data/rook/目录，会遗留下Ceph集群的配置信息。

若之后再部署新的Ceph集群，先把之前Ceph集群的这些信息删除，不然启动monitor会失败；

```sh
# cat clean-rook-dir.sh
hosts=(
  192.168.1.213
  192.168.1.162
  192.168.1.163
  192.168.1.164
)

for host in ${hosts[@]} ; do
  ssh $host "rm -rf /data/rook/*"
done
```

清除device

```sh
#!/usr/bin/env bash
DISK="/dev/nvme0n1p1"
# Zap the disk to a fresh, usable state (zap-all is important, b/c MBR has to be clean)
# You will have to run this step for all disks.
sgdisk --zap-all $DISK
# hdd 用以下命令
dd if=/dev/zero of="$DISK" bs=1M count=100 oflag=direct,dsync
# ssd 用以下命令
blkdiscard $DISK

# These steps only have to be run once on each node
# If rook sets up osds using ceph-volume, teardown leaves some devices mapped that lock the disks.
ls /dev/mapper/ceph-* | xargs -I% -- dmsetup remove %
# ceph-volume setup can leave ceph-<UUID> directories in /dev (unnecessary clutter)
rm -rf /dev/ceph-*
```

如果因为某些原因导致删除ceph集群卡主，可以先执行以下命令， 再删除ceph集群就不会卡主了

```sh
kubectl -n rook-ceph patch cephclusters.ceph.rook.io rook-ceph -p '{"metadata":{"finalizers": []}}' --type=merge
```



#### 2.9 rook升级

##### 2.9.1 小版本升级

Rook v1.5.0 to Rook v1.5.1

```
git clone --single-branch --branch v1.5.1 https://github.com/rook/rook.gits
cd $YOUR_ROOK_REPO/cluster/examples/kubernetes/ceph/
kubectl apply -f common.yaml -f crds.yaml
kubectl -n rook-ceph set image deploy/rook-ceph-operator rook-ceph-operator=rook/ceph:v1.5.1
```



##### 2.9.2 跨版本升级

Rook v1.4.x to Rook v1.5.x.

**准备**

设置环境变量

```sh
# Parameterize the environment
export ROOK_SYSTEM_NAMESPACE="rook-ceph"
export ROOK_NAMESPACE="rook-ceph"
```

> 升级之前需要保证集群健康

所有pod 是running

```sh
kubectl -n $ROOK_NAMESPACE get pods
```

通过tool 查看ceph集群状态是否正常

```sh
TOOLS_POD=$(kubectl -n $ROOK_NAMESPACE get pod -l "app=rook-ceph-tools" -o jsonpath='{.items[0].metadata.name}')
kubectl -n $ROOK_NAMESPACE exec -it $TOOLS_POD -- ceph status
  cluster:
    id:     194d139f-17e7-4e9c-889d-2426a844c91b
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum a,b,c (age 25h)
    mgr: a(active, since 5h)
    mds: myfs:1 {0=myfs-b=up:active} 1 up:standby-replay
    osd: 4 osds: 4 up (since 25h), 4 in (since 25h)

  task status:
    scrub status:
        mds.myfs-a: idle
        mds.myfs-b: idle

  data:
    pools:   4 pools, 97 pgs
    objects: 2.08k objects, 7.6 GiB
    usage:   26 GiB used, 3.3 TiB / 3.3 TiB avail
    pgs:     97 active+clean

  io:
    client:   1.2 KiB/s rd, 2 op/s rd, 0 op/s wr
```



**升级operator**

1、 升级common和crd

```sh
git clone --single-branch --branch v1.5.1 https://github.com/rook/rook.gits
cd rook/cluster/examples/kubernetes/ceph
kubectl apply -f common.yaml -f crds.yaml
```

2、升级 Ceph CSI versions

可以修改cm来自己制定镜像版本，如果是默认的配置，无需修改

```sh
kubectl -n rook-ceph get configmap rook-ceph-operator-config

ROOK_CSI_CEPH_IMAGE: "harbor.foxchan.com/google_containers/cephcsi/cephcsi:v3.1.1"
ROOK_CSI_REGISTRAR_IMAGE: "harbor.foxchan.com/google_containers/k8scsi/csi-node-driver-registrar:v2.0.1"
ROOK_CSI_PROVISIONER_IMAGE: "harbor.foxchan.com/google_containers/k8scsi/csi-provisioner:v2.0.0"
ROOK_CSI_SNAPSHOTTER_IMAGE: "harbor.foxchan.com/google_containers/k8scsi/csi-snapshotter:v3.0.0"
ROOK_CSI_ATTACHER_IMAGE: "harbor.foxchan.com/google_containers/k8scsi/csi-attacher:v3.0.0"
ROOK_CSI_RESIZER_IMAGE: "harbor.foxchan.com/google_containers/k8scsi/csi-resizer:v1.0.0"
```

3、升级 Rook Operator

```sh
kubectl -n $ROOK_SYSTEM_NAMESPACE set image deploy/rook-ceph-operator rook-ceph-operator=rook/ceph:v1.5.1
```

4、等待集群 升级完毕

```sh
watch --exec kubectl -n $ROOK_NAMESPACE get deployments -l rook_cluster=$ROOK_NAMESPACE -o jsonpath='{range .items[*]}{.metadata.name}{"  \treq/upd/avl: "}{.spec.replicas}{"/"}{.status.updatedReplicas}{"/"}{.status.readyReplicas}{"  \trook-version="}{.metadata.labels.rook-version}{"\n"}{end}'
```

5、验证集群升级完毕

```sh
kubectl -n $ROOK_NAMESPACE get deployment -l rook_cluster=$ROOK_NAMESPACE -o jsonpath='{range .items[*]}{"rook-version="}{.metadata.labels.rook-version}{"\n"}{end}' | sort | uniq
```

##### 

**升级ceph 版本**

> 如果集群状态不监控，operator会拒绝升级

1、升级ceph镜像

```sh
NEW_CEPH_IMAGE='ceph/ceph:v15.2.5'
CLUSTER_NAME=rook-ceph  
kubectl -n rook-ceph patch CephCluster rook-ceph --type=merge -p "{\"spec\": {\"cephVersion\": {\"image\": \"$NEW_CEPH_IMAGE\"}}}"
```

2、观察pod 升级

```sh
watch --exec kubectl -n $ROOK_NAMESPACE get deployments -l rook_cluster=$ROOK_NAMESPACE -o jsonpath='{range .items[*]}{.metadata.name}{"  \treq/upd/avl: "}{.spec.replicas}{"/"}{.status.updatedReplicas}{"/"}{.status.readyReplicas}{"  \tceph-version="}{.metadata.labels.ceph-version}{"\n"}{end}'
```

3、查看ceph集群是否正常

```sh
kubectl -n $ROOK_NAMESPACE get deployment -l rook_cluster=$ROOK_NAMESPACE -o jsonpath='{range .items[*]}{"ceph-version="}{.metadata.labels.ceph-version}{"\n"}{end}' | sort | uniq
```



### 三、部署块存储

#### 3.1 创建pool和StorageClass

```yaml
# 定义一个块存储池
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  # 每个数据副本必须跨越不同的故障域分布，如果设置为host，则保证每个副本在不同机器上
  failureDomain: host
  # 副本数量
  replicated:
    size: 3
    # Disallow setting pool with replica 1, this could lead to data loss without recovery.
    # Make sure you're *ABSOLUTELY CERTAIN* that is what you want
    requireSafeReplicaSize: true
    # gives a hint (%) to Ceph in terms of expected consumption of the total cluster capacity of a given pool
    # for more info: https://docs.ceph.com/docs/master/rados/operations/placement-groups/#specifying-expected-pool-size
    #targetSizeRatio: .5
---
# 定义一个StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: rook-ceph-block
# 该SC的Provisioner标识，rook-ceph前缀即当前命名空间
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
    # clusterID 就是集群所在的命名空间名
    # If you change this namespace, also change the namespace below where the secret namespaces are defined
    clusterID: rook-ceph

    # If you want to use erasure coded pool with RBD, you need to create
    # two pools. one erasure coded and one replicated.
    # You need to specify the replicated pool here in the `pool` parameter, it is
    # used for the metadata of the images.
    # The erasure coded pool must be set as the `dataPool` parameter below.
    #dataPool: ec-data-pool
    # RBD镜像在哪个池中创建
    pool: replicapool

    # RBD image format. Defaults to "2".
    imageFormat: "2"

    # 指定image特性，CSI RBD目前仅仅支持layering
    imageFeatures: layering

    # Ceph admin 管理凭证配置,由operator 自动生成
    # in the same namespace as the cluster.
    csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
    csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
    csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
    csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
    csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
    csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
    # 卷的文件系统类型，默认ext4，不建议xfs，因为存在潜在的死锁问题（超融合设置下卷被挂载到相同节点作为OSD时）
    csi.storage.k8s.io/fstype: ext4
# uncomment the following to use rbd-nbd as mounter on supported nodes
# **IMPORTANT**: If you are using rbd-nbd as the mounter, during upgrade you will be hit a ceph-csi
# issue that causes the mount to be disconnected. You will need to follow special upgrade steps
# to restart your application pods. Therefore, this option is not recommended.
#mounter: rbd-nbd
allowVolumeExpansion: true
reclaimPolicy: Delete
```



#### 3.2 demo示例

推荐pvc 和应用写到一个yaml里面

```yaml
#创建pvc
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rbd-demo-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: rook-ceph-block
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: csirbd-demo-pod
  labels:
    test-cephrbd: "true"
spec:
  replicas: 1
  selector:
    matchLabels:
      test-cephrbd: "true"
  template:
    metadata:
      labels:
        test-cephrbd: "true"
    spec:
      containers:
       - name: web-server-rbd
         image: harbor.foxchan.com/sys/nginx:1.19.4-alpine
         volumeMounts:
           - name: mypvc
             mountPath: /usr/share/nginx/html
      volumes:
       - name: mypvc
         persistentVolumeClaim:
           claimName: rbd-demo-pvc
           readOnly: false
```



### 四、部署文件系统

#### 4.1 创建CephFS

> CephFS的CSI驱动使用Quotas来强制应用PVC声明的大小，仅仅4.17+内核才能支持CephFS quotas。
>
> 如果内核不支持，而且你需要配额管理，配置Operator环境变量 CSI_FORCE_CEPHFS_KERNEL_CLIENT: **false**来启用FUSE客户端。
>
> 使用FUSE客户端时，升级Ceph集群时应用Pod会断开mount，需要重启才能再次使用PV。

```yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: myfs
  namespace: rook-ceph
spec:
  # The metadata pool spec. Must use replication.
  metadataPool:
    replicated:
      size: 3
      requireSafeReplicaSize: true
    parameters:
      # Inline compression mode for the data pool
      # Further reference: https://docs.ceph.com/docs/nautilus/rados/configuration/bluestore-config-ref/#inline-compression
      compression_mode: none
        # gives a hint (%) to Ceph in terms of expected consumption of the total cluster capacity of a given pool
      # for more info: https://docs.ceph.com/docs/master/rados/operations/placement-groups/#specifying-expected-pool-size
      #target_size_ratio: ".5"
  # The list of data pool specs. Can use replication or erasure coding.
  dataPools:
    - failureDomain: host
      replicated:
        size: 3
        # Disallow setting pool with replica 1, this could lead to data loss without recovery.
        # Make sure you're *ABSOLUTELY CERTAIN* that is what you want
        requireSafeReplicaSize: true
      parameters:
        # Inline compression mode for the data pool
        # Further reference: https://docs.ceph.com/docs/nautilus/rados/configuration/bluestore-config-ref/#inline-compression
        compression_mode: none
          # gives a hint (%) to Ceph in terms of expected consumption of the total cluster capacity of a given pool
        # for more info: https://docs.ceph.com/docs/master/rados/operations/placement-groups/#specifying-expected-pool-size
        #target_size_ratio: ".5"
  # Whether to preserve filesystem after CephFilesystem CRD deletion
  preserveFilesystemOnDelete: true
  # The metadata service (mds) configuration
  metadataServer:
    # The number of active MDS instances
    activeCount: 1
    # Whether each active MDS instance will have an active standby with a warm metadata cache for faster failover.
    # If false, standbys will be available, but will not have a warm cache.
    activeStandby: true
    # The affinity rules to apply to the mds deployment
    placement:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
          - matchExpressions:
            - key: app.storage
              operator: In
              values:
              - rook-ceph
    #  topologySpreadConstraints:
    #  tolerations:
    #  - key: mds-node
    #    operator: Exists
    #  podAffinity:
      podAntiAffinity:
         requiredDuringSchedulingIgnoredDuringExecution:
         - labelSelector:
             matchExpressions:
             - key: ceph-mds
               operator: In
               values:
               - enabled
            # topologyKey: kubernetes.io/hostname will place MDS across different hosts
           topologyKey: kubernetes.io/hostname
         preferredDuringSchedulingIgnoredDuringExecution:
         - weight: 100
           podAffinityTerm:
             labelSelector:
               matchExpressions:
               - key: ceph-mds
                 operator: In
                 values:
                  - enabled
              # topologyKey: */zone can be used to spread MDS across different AZ
              # Use <topologyKey: failure-domain.beta.kubernetes.io/zone> in k8s cluster if your cluster is v1.16 or lower
              # Use <topologyKey: topology.kubernetes.io/zone>  in k8s cluster is v1.17 or upper
             topologyKey: topology.kubernetes.io/zone
    # A key/value list of annotations
    annotations:
    #  key: value
    # A key/value list of labels
    labels:
    #  key: value
    resources:
    # The requests and limits set here, allow the filesystem MDS Pod(s) to use half of one CPU core and 1 gigabyte of memory
    #  limits:
    #    cpu: "500m"
    #    memory: "1024Mi"
    #  requests:
    #    cpu: "500m"
    #    memory: "1024Mi"
    # priorityClassName: my-priority-class
```



#### 4.2 创建StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-cephfs
provisioner: rook-ceph.cephfs.csi.ceph.com
parameters:
  # clusterID is the namespace where operator is deployed.
  clusterID: rook-ceph

  # CephFS filesystem name into which the volume shall be created
  fsName: myfs

  # Ceph pool into which the volume shall be created
  # Required for provisionVolume: "true"
  pool: myfs-data0

  # Root path of an existing CephFS volume
  # Required for provisionVolume: "false"
  # rootPath: /absolute/path

  # The secrets contain Ceph admin credentials. These are generated automatically by the operator
  # in the same namespace as the cluster.
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-cephfs-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph

  # (optional) The driver can use either ceph-fuse (fuse) or ceph kernel client (kernel)
  # If omitted, default volume mounter will be used - this is determined by probing for ceph-fuse
  # or by setting the default mounter explicitly via --volumemounter command-line argument.
  #使用kernel client
  mounter: kernel
reclaimPolicy: Delete
allowVolumeExpansion: true
mountOptions:
  # uncomment the following line for debugging
  #- debug
```



#### 4.3 创建pvc

> 在创建cephfs 的pvc 发现一直处于pending状态，社区有人认为是网络组件的差异，目前我的calico无法成功，只能改为host模式，flannel可以。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cephfs-pvc
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: rook-cephfs
```



#### 4.4 demo示例

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cephfs-demo-pvc
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: rook-cephfs
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: csicephfs-demo-pod
  labels:
    test-cephfs: "true"
spec:
  replicas: 2
  selector:
    matchLabels:
      test-cephfs: "true"
  template:
    metadata:
      labels:
        test-cephfs: "true"
    spec:
      containers:
      - name: web-server
        image: harbor.foxchan.com/sys/nginx:1.19.4-alpine
        imagePullPolicy: Always
        volumeMounts:
        - name: mypvc
          mountPath: /usr/share/nginx/html
      volumes:
      - name: mypvc
        persistentVolumeClaim:
          claimName: cephfs-demo-pvc
          readOnly: false
```



### 五、遇到问题

#### 5.1 lvm direct 不能直接做osd 存储

官方issue：https://github.com/rook/rook/issues/5751

解决方式：可以手动创建本地pvc，把lvm挂载上在做osd设备。如果手动闲麻烦，可以使用 `local-path-provisioner`



#### 5.2 Cephfs pvc pending

官方issue：https://github.com/rook/rook/issues/6183

解决方式：更换 k8s 网络组件，或者把 ceph 集群网络开启 host







## 数据库容器化部署在PaaS平台上的12个难点总结



### 1、数据库容器化部署在PaaS平台对性能方面的影响？

【问题描述】数据库容器化部署在PaaS平台在性能方面有多大的影响，如何进行性能调优？

@强哥之神 上汽云计算中心 容器云架构师及技术经理：

容器化，本质上是对运行在宿主机上的线程（容器本身就是一个线程）限制了CPU/MEM等资源（使用 cgroups），并给它一个隔离空间（用 namespace来实现)，所以只要不特别限制资源上限，基本和非容器化的效果是一样的。

而数据库一般是自身实现数据的一致性保证，也包括一些中间件服务，比如 redis ，kafka 等，所以只要使用本地存储来实现数据一致性、持久化或数据同步就和使用主机部署是一样的。有些数据库如果数据很大的话，建议使用 LocalPV 来做持久卷，如果对性能要求不高的，还可以使用Ceph rbd/CephFS 等网络存储卷。

@沈天真 浪潮商用机器企业云创新中心：

理论上把数据库容器化，又增加了一层中间层，这个性能肯定是下降的；除非能显著提升资源利用率，或者说叠加很强的多租户需求，使得整机性能显著提升，可能还值得。否则个人感觉，不要没事找事搞数据库容器化，还带来一堆的运维，监控，数据一致性保证等等问题。



### 2、数据库容器化部署有哪些优缺点？如何保证的系统的性能的高可用性？

@mtming333 甜橙金融翼支付 系统架构师：

比如TIDB，融合了关系型数据库和非关系型数据库的技术特性，实现了高度兼容MySQL协议和生态、在线水平扩展、强一致分布式事务、多副本数据安全、实时分析等重要特性

@lzj7618937 cib 质控经理：

使用Docker的好处举些例子：

1.屏蔽底层物理资源

2.提升资源利用率 (CPU、内存)

3.提升运维效率

有问题的地方：

1.数据安全问题

2.相对于物理机性能问题

3.资源隔离方面

所以综合还是要看不同的适用场景，像下面几种可以考虑容器化部署：

1.对数据丢失不敏感的业务(例如用户搜索商品)就可以数据化，利用数据库分片来来增加实例数，从而增加吞吐量。

2.docker适合跑轻量级或分布式数据库，当docker服务挂掉，会自动启动新容器，而不是继续重启容器服务。

3.数据库利用中间件和容器化系统能够自动伸缩、容灾、切换、自带多个节点，也是可以进行容器化的。



### 3、在数据库容器化部署架构中，如何设计和优化数据库网络？

@强哥之神 上汽云计算中心 容器云架构师及技术经理：

为了提升数据库容器化时的网络性能，一般可以采用网络性能和宿主机一致的 HostNetwork 方式来实现，如果在某些公司或者公有云环境下，出于安全考虑不允许该模式的话，建议采用 calico 的 BGP 模式，但这个也受限于单个子网段，BGP如果要跨网段通信，由于不是采用传统的 overlay 技术实现，所以需要用路由器打通，在其路由表里直接连接起各网段的 BGP 信息。 

@lzj7618937 cib 质控经理：

不同的方案有不同的优缺点，选择方案的时候还是要根据应用场景去选择。例如考虑使用数据库应用服务器和数据库服务器这间是什么网络连接方式，是docker内直接连接还是在不同的集群甚至是在不同的局域网内。总体还是选择目前主流，官方推荐，较好扩展的方案。



### 4、容器环境下基于本地高性能存储的MySQL集群节点如何迁移？

【问题描述】某些场景下，容器云平台可能无法使用nas，san等存储，只能使用节点本地的NVMe固态。那么在这种场景下，mysql容器的存储就被限制在了所在节点。当所在节点故障、需要维护，以及因为压力大需要疏散时，就需要考虑mysql容器要如何迁移？

@lzj7618937 cib 质控经理：

我觉得这个在不是容器环境下也得考虑的问题。说到底还是数据的如何迁移和备份。这种建议使用另外的SAN或NAS云盘备份本地的硬盘数据，出现问题直接从云盘恢复相关数据。

@强哥之神 上汽云计算中心 容器云架构师及技术经理：

我们一般会通过实现自己的 Operator 来保证 Mysql 集群中的数据存储的一致性，这个具体一点就是：

以多节点来实现mysql 的容器化部署，根据现有的场景需要，是主从，主备还是主主，还是一主多从，一主多备，这些场景中，无一不体现数据迁移的保证。在 K8S 中，一般都是通过基于 LocalPV 的挂卷方式来实现，通过 bin log 或者定期备份来保证数据恢复时所需。mysql 备份有逻辑与物理备份方法，比如 INSERT 语句文件的恢复 ， 纯数据文本备份的恢复 ，InnoDB存储引擎备份与恢复，NDB Cluster 存储引擎备份与恢复等等。



### 5、数据库是否适合容器化部署？系统稳定性是如何保证的是如何保证的？

@强哥之神 上汽云计算中心 容器云架构师及技术经理：

目前中间件容器化趋势比较明显了，数据库作为主要代表，也适合容器化部署，这个得利于容器的环境集成封装及容器平台为运行于其上的容器的高可用、弹性伸缩的特性。

系统稳定性主要还是通过使用 K8S 的 Operator 技术来实现数据库服务的高可用性来实现。有了 Operator 技术，我们可以自定义数据库容器的运行行为，包括主主、主从、主备或者一主多备等各种场景需求。我们公司目前就通过自己实现 Operator 来定义这些行为来保证业务所需。



### 6、数据库容器化部署的IO性能如何？如何评估数据库是否适合容器化？

@lzj7618937 cib 质控经理：

(1)数据库程序与数据分离

如果使用Docker 跑 MySQL，数据库程序与数据需要进行分离，将数据存放到共享存储，程序放到容器里。如果容器有异常或 MySQL 服务异常，自动启动一个全新的容器。另外，建议不要把数据存放到宿主机里，宿主机和容器共享卷组，对宿主机损坏的影响比较大。

(2)跑轻量级或分布式数据库

Docker 里部署轻量级或分布式数据库，Docker 本身就推荐服务挂掉，自动启动新容器，而不是继续重启容器服务。

(3)合理布局应用

对于IO要求比较高的应用或者服务，将数据库部署在物理机或者KVM中比较合适。目前TX云的TDSQL和阿里的Oceanbase都是直接部署在物理机器，而非Docker 。



### 7、关系型数据就是否适合容器化部署及存在的问题？

【问题描述】我们公司目前目前构建了了一台spring cloud的的微服务微服务开发平台，数据库oracle还得还是是部署部署在虚机上年上面，考虑到到容器适合无状态化部署部署和数据安全性、网络网络方面问题问题，不知道现在现在解决解决的的怎么样或是或是有更好的解决方案，也请各位专家指导指导。

@lzj7618937 cib 质控经理：

建议从以下几方面去考虑：

1、数据安全问题，例如 容器突然崩溃，数据库未正常关闭，可能会损坏数据。另外，容器里共享数据卷组，对物理机硬件损伤也比较大。

2、性能问题， 关系型数据库，对IO要求较高。当一台物理机跑多个时，IO就会累加，导致IO瓶颈，大大降低 读写性能。

3、网络问题， 要理解 Docker 网络，您必须对网络虚拟化有深入的了解。也必须准备应付好意外情况。你可能需要在没有支持或没有额外工具的情况下，进行 bug 修复。

4、状态， 在 Docker 中打包无状态服务是很酷的，可以实现编排容器并解决单点故障问题。但是数据库呢？将数据库放在同一个环境中，它将会是有状态的，并使系统故障的范围更大。下次您的应用程序实例或应用程序崩溃，可能会影响数据库。

5、资源隔离， 资源隔离方面，Docker 确实不如虚拟机KVM，Docker是利用Cgroup实现资源限制的，只能限制资源消耗的最大值，而不能隔绝其他程序占用自己的资源。

6、运行数据库的环境需求，常看到 DBMS 容器和其他服务运行在同一主机上。然而这些服务对硬件要求是非常不同的。数据库（特别是关系型数据库）对 IO 的要求较高。一般数据库引擎为了避免并发资源竞争而使用专用环境。如果将你的数据库放在容器中，那么将浪费你的项目的资源。因为你需要为该实例配置大量额外的资源。在公有云，当你需要 34G 内存时，你启动的实例却必须开 64G 内存。在实践中，这些资源并未完全使用。



### 8、在k8s中部署mysql主从问题？

【问题描述】能不能介绍下，在K8S集群中部署mysql主从同步数据一致性的问题，以及主down掉情况下的自动切换，有没有最佳实践？

@强哥之神 上汽云计算中心 容器云架构师及技术经理：

这个在K8S中，mysql 主从问题是讨论的比较多的，也是在生产中应用的比较多的案例。

说到主从，无非就是数据是以谁为主，主挂的时候，备是否能顶上？主恢复的时候，备是否被换下？

早期业界有两种做法，一种是使用头脑分裂法，在容器的启停时去实现数据的同步及主从切换。这个见官方的使用 statefulset 的做法，见：https://kubernetes.io/docs/tasks/run-application/run-replicated-stateful-application/

还有一种是使用最近比较火的 Operator ，这个也有开源的，比如：https://github.com/oracle/mysql-operator

我是比较推荐使用 Operator的方法，因为这种才是自动化运维 mysql 数据库的绝佳推荐方式，不会担心其他的意外情况发生。



### 9、面对跑在k8s上的MySQL，中间件如何选型？

【问题描述】MYSQL中间件作为分布式数据库的关键核心组件，承担着分表分库，读写分离，高可用等核心功能，将mysql运行在k8s上，必定面临着如上各种问题，如何选型和结合k8s自由功能进行融合？

@lzj7618937 cib 质控经理：

我觉得中间件就是为了让你屏蔽底层数据库的具体配置实现，而使用k8s也是为了方便数据库集群的高可用性、方便部署等问题，这两部分觉得应该分开的考虑，而不应该耦合在一起考虑。不然只会加大查问题及使用的复杂度，那就没必要使用这种方案了。



### 10、数据库容器化部署如何保证数据安全性？

@lzj7618937 cib 质控经理：

传统虚拟化技术与Docker容器技术在运行时的安全性差异主要体现在隔离性方面，包括进程隔离、文件系统隔离、设备隔离、进程间通信隔离、网络隔离、资源限制等。

在Docker容器环境中，由于各容器共享操作系统内核，而容器仅为运行在宿主机上的若干进程，其安全性特别是隔离性与传统虚拟机相比在理论上与实际上都存在一定的差距。

如果要保证容器化的部署，还是建议重源头避免，通过镜像的安全相关扫描避免。



### 11、容器上面可以部署有持久化数据的应用或者数据库吗？

【问题描述】如果是用了容器云，有持久化数据需求的应用应该如何部署，后端是否只能接nas，是否能在容器上部署oracle数据库，升级oracle数据库时能否直接部署一个新版本的数据库再挂载原有的数据升级数据字典即可，如果可以应该如何规划？是否又成功案例？

@nameless 某云计算厂商 技术总监：

1、数据库可以容器化，但一般在开发测试环境，生产环境不建议数据库容器化；

2、有状态应用发布后端存储可以用NAS，也可以使用分布式存储，如ceph、glusterfs等存储，具体需根据业务使用场景，如有业务对性能要求较高，一般的存储可能都不行；

3、oracle可以容器化，主要是拉起来比较快，使用很方便，但oracle的一些高级特性在容器化以后无法使用或者配置很麻烦；

@Steven99  软件架构设计师：

可以是可以，但是不建议。

oracle数据库通常是追求稳定、而且笨重，IO要求也比较高，不太适合容器化部署。

因为应用通常是分层的，所以应用服务是可以容器化部署，而数据库尽可能在物理服务器上部署，通过网络访问数据库。

测试环境可以把oracle等部署到容器，为的是敏捷、一致性考虑，容器重部署则状态重置，就不需要每次清理测试数据，对于回归测试等还是很方便的。

总的来说，要根据实际考虑合适的技术特性和应用场景，选择合适的方法。

@dyjiafeimao  技术总监：

一、容器本身是不能做数据持久化，可以加入数据卷的概念，数据卷绑定存储系统，比如nas等，把需要做持久化的数据保存在数据卷中就做到了数据持久化。

二、容器中的应用需要访问Oracle数据库，利用容器编排技术创建jdbc service和jdbc endpoint服务访问外部的Oracle数据库。

三、Mysql数据库可以做成镜像运行在容器中，如果需要做数据持久化，需要把mysql数据库的数据文件保存在数据卷中。



### 12、不做传统数据库容器化，但应用部署在PaaS平台上，如何去对接数据层和应用层？

@zhuqibs Mcd 软件开发工程师：

（1）你的应用必定在paas，你所指的paas，应该是容器云的tke，就是全托管的kubenetes集群，这在各种公有云现在基本上都提供了。

（2）数据库不在容器云中，这也是必然，数据库软件由于性能问题，必定不会包括在容器云内。而数据库的存储也不可能在容器云内，而分布式存储性能太差。

（3）数据库可以也使用paas平台

全部paas化，降低运维成本，专注于业务。

所以，你要求的这个架构是最最最通常的架构，也就是，目前大家都是这么玩的。对接数据库层和应用层都是标准的做法，可以ip对接，也可以域名对接。

@赵锡漪 红帽企业级开源解决方案中心 资深电信行业解决方案架构师：

1、通常为了对原有业务的兼容以及容器化改造的简易性，我们容器化的最初阶段都是直接使用原有数据对接模型的。比如仍然使用数据原有连接池，匹配外部数据库连接地址。但如我在PPT内容中谈到的，当应用具备一定规模，这一定会导致原有数据使用压力的增加，因此我们需要在业务的敏态改造中逐步提高数据对接层面的合理性与规划性。

2、为了提高数据对接层面的合理性与规划性，最佳方案是先通过一些技术对数据直连进行隔离。有Kafka就通过Kafka 隔离出读写层，没有 Kafka 可以通过 JBoss Data Virtualization 这样的数据源虚拟化工具进行面向原逻辑的抽象隔离，但需要注意的是 JBoss Data Virtualization 这样的数据源抽象工具对复杂 SQL 使用场景非常容易产生兼容性问题或性能问题。因此将原有的复杂 SQL 这种过程处理逻辑进行一定程度的面向对象改造是很有必要的。

3、如果服务无法完全进行数据源抽象适应，例如无法拆分复杂SQL，那么我们就必须通过一定的开发来实现面向对象数据源改造。此时可以配合Red Hat Data Grid 这样的数据网格技术来实现就近数据源适应的改造了。不要忘记了 Redhat DataGrid 产品可以直接支持 Redis 的所有能力，可以完全兼容 Memcache 使用，因此非常适合将单体数据缓存改为云化适应的数据网格。

4、有了中间的数据隔离技术，Kafka/JBoss Data Grid/JBoss Data Virtulazition 这些技术就可以与原有数据元之间的关系进行梳理。比使用 Redhat Change Data Capture 工具可以很轻松的帮助用户实现如数据同步，多数据中心同步，数据应急，数据灾备，数据异地多中心同步，数据一致性传递，多数据源一致性同步等等目标，同时这些技术之间的梳理就会变的比以前简单，顺便可以重新梳理原有的分库分表结构，读写分离结构等，更进一步提高原有数据使用质量。

@xuyuting 北京顶象技术有限公司 项目经理：

数据库对接没有什么大的变化，对接应用层需要考虑应用IP地址会动态变化的问题。可以使用网关+注册中心的方式。





## 揭秘有状态服务上 Kubernetes 的核心技术

唐聪，腾讯云资深工程师，极客时间专栏《etcd实战课》作者，etcd活跃贡献者，主要负责腾讯云大规模K8s/etcd平台、有状态服务容器化、在离线混部等产品研发设计工作。



### 背景

----

随着 Kubernetes 成为云原生的最热门的解决方案，越来越多的传统服务从虚拟机、物理机迁移到 Kubernetes，各云厂商如腾讯自研上云也主推业务通过Kubernetes来部署服务，享受 Kubernetes 带来的弹性扩缩容、高可用、自动化调度、多平台支持等益处。然而，目前大部分基于 Kubernetes 的部署的服务都是无状态的，为什么有状态服务容器化比无状态服务更难呢？它有哪些难点？各自的解决方案又是怎样的?

本文将结合我对 Kubernetes 理解、丰富的有状态服务开发、治理、容器化经验，为你浅析有状态容器化的疑难点以及相应的解决方案，希望通过本文，能帮助你理解有状态服务的容器化疑难点，并能基于自己的有状态服务场景能灵活选择解决方案，高效、稳定地将有状态服务容器化后跑在 Kubernetes 上，提高开发运维效率和产品竞争力。



### 有状态服务容器化挑战

---

为了简化问题，避免过度抽象，我将以常用的 Redis 集群为具体案例，详解如何将一个 Redis 集群进行容器化，并通过这个案例进一步分析、拓展有状态服务场景中的共性问题。

下图是 Redis 集群解决方案 codis 的整体架构图（引用自 **Codis项目**[1])。

![image-20210602220710603](D:\学习资料\笔记\k8s\k8s图\image-20210602220710603.png)

codis 是一个基于 proxy 的分布式 Redis 集群解决方案，它由以下核心组件组成:

- zookeeper/etcd， 有状态元数据存储，一般奇数个节点部署
- codis-proxy， 无状态组件，通过计算 key 的 crc16 哈希值，根据保存在 zookeeper/etcd 内的 preshard 路由表信息，将key转发到对应的后端 codis-group
- codis-group 由一组 Redis 主备节点组成，一主多备，负责数据的读写存储
- codis-dashboard 集群控制面API服务，可以通过它增删节点、迁移数据等
- redis-sentinel，集群高可用组件，负责探测、监听 Redis 主的存活，主故障时发起备切换

那么我们如何基于 Kubernetes 容器化 codis 集群，通过 kubectl 操作资源就能一键创建、高效管理 codis 集群呢？

在容器化类似 codis 这种有状态服务案例中，我们需要解决以下问题：

- 如何用 Kubernetes 的语言描述你的有状态服务？
- 如何为你的有状态服务选择合适的 workload 部署？
- 当 kubernetes 内置的 workload 无法直接描述业务场景时，又该选择什么样的 Kubernetes 扩展机制呢？
- 如何对有状态服务进行安全变更？
- 如何确保你的有状态服务主备实例 Pod 调度到不同故障域？
- 有状态服务实例故障如何自愈？
- 如何满足有状态服务的容器化后的高网络性能需求？
- 如何满足有状态服务的容器化后的高存储性能需求？
- 如何验证有状态服务容器化后的稳定性？

下方是我用思维导图系统性的梳理了容器化有状态的服务的技术难点，接下来我分别从以上几个方面为你阐述容器化的解决方案。

![image-20210602220820379](D:\学习资料\笔记\k8s\k8s图\image-20210602220820379.png)



### 负载类型

有状态服务的容器化首要问题是如何用 Kubernetes 式的 API、语言来描述你的有状态服务？

Kubernetes 为复杂软件世界中的各类业务场景抽象、内置了 Pod、Deployment、StatefulSet 等负载类型(Workload)， 那么各个 Workload 的使用场景分别是什么呢？

Pod，它是最小的调度、部署单位，由一组容器组成，它们共享网络、数据卷等资源。为什么 Pod 设计上它是一组容器组成而不是一个呢？

因为在实际复杂业务场景中，往往一个业务容器无法独立完成某些复杂功能，比如你希望使用一个辅助容器帮助你下载冷备快照文件、做日志转发等，得益于 Pod 的优秀设计，辅助容器可以和你的 Redis、MySQL、etcd、zookeeper 等有状态容器共享同个网络命名空间、数据卷，帮助主业务容器完成以上工作。这种辅助容器在 Kubernetes 里面叫做 sidecar， 广泛应用于日志、转发、service mesh 等辅助场景，已成为一种 Kubernetes 设计模式。Pod 优秀设计来源于 Google 内部 Borg 十多年运行经验的总结和升华，可显著地降低你将复杂的业务容器化的成本。

通过 Pod 成功将业务进程容器化了，然而 Pod 本身并不具备高可用、自动扩缩容、滚动更新等特性，因此为了解决以上挑战，Kubernetes 提供了更高级的 Workload Deployment， 通过它你可以实现Pod故障自愈、滚动更新、并结合 HPA 组件可实现按 CPU、内存或自定义指标实现自动扩缩容等高级特性，它一般是用来描述无状态服务场景的，因此特别适合我们上面讨论的有状态集群中的无状态组件，比如 codis 集群的 proxy 组件等。

那么 Deployment 为什么不适合有状态呢？主要原因是 Deployment 生成的 Pod 名称是变化、无稳定的网络标识身份、无稳定的持久化存储、滚动更新中过程中也无法控制顺序，而这些对于有状态而言，是非常重要的。一方面有状态服务彼此通过稳定的网络身份标识进行通信是其高可用、数据可靠性的基本要求，如在 etcd 中，一个日志提交必须要经过集群半数以上节点确认并持久化，在 Redis 中，主备根据稳定的网络身份建立主从同步关系。另一方面，不管是 etcd 还是 Redis 等其他组件，Pod 异常重建后，业务往往希望它对应的持久化数据不能丢失。

为了解决以上有状态服务场景的痛点，Kubernetes 又设计实现了 StatefulSet 来描述此类场景，它可以为每个 Pod 提供唯一的名称、固定的网络身份标识、持久化数据存储、有序的滚动更新发布机制。基于 StatefulSet 你可以比较方便的将 etcd、zookeeper 等组件较单一的有状态服务进行容器化部署。

通过 Deployment、StatefulSet 我们能将大部分现实业务场景的服务进行快速容器化，但是业务诉求是多样化的，各自的技术栈、业务场景也是迥异的，有的希望实现 Pod 固定IP的，方便快速对接传统的负载均衡，有的希望实现发布过程中，Pod不重建、支持原地更新的，有的希望能指定任意 Statefulset Pod 更新的，那么 Kubernetes 如何满足多样化的诉求呢？



### 扩展机制

---

Kubernetes 设计上对外提供了一个强大扩展体系，如下图所示（引用自 kubernetes blog），从 kubectl plugin 到 Aggreated API Server、再到 CRD、自定义调度器、再到 operator、网络插件(CNI）、存储插件(CSI)。一切皆可扩展，充分赋能业务，让各个业务可基于Kubernetes扩展机制进行定制化开发，满足大家的特定场景诉求。

![image-20210602220940184](D:\学习资料\笔记\k8s\k8s图\image-20210602220940184.png)



![image-20210602221006557](D:\学习资料\笔记\k8s\k8s图\image-20210602221006557.png)

#### CRD 和 Aggreated API Server

当你遇到 Deployment、StatefulSet 无法满足你诉求的时候，Kubernetes 提供了 CRD 和 Aggreated API Server、Operator 等机制给你扩展 API 资源、结合你特定的领域和应用知识，实现自动化的资源管理和运维任务。

CRD 即 CustomResourceDefinition，是 Kubernetes 内置的一种资源扩展方式，在 apiserver 内部集成了 kube-apiextension-server， 不需要在 Kubernetes 集群运行额外的 Apiserver，负责实现 Kubernetes API CRUD、Watch 等常规API操作，支持 kubectl、认证、授权、审计，但是不支持 SubResource log/exec 等定制，不支持自定义存储，存储在 Kubernetes 集群本身的 etcd 上，如果涉及大量 CRD 资源需要存储则对 Kubernetes 集群etcd 性能有一定的影响，同时限制了服务从不同集群间迁移的能力。

Aggreated API Server，即聚合 ApiServer， 像我们常用的 metrics-server 属于此类，通过此特性 Kubernetes 将巨大的单 apiserver 按资源类别拆分成多个聚合 apiserver， 扩展性进一步加强，新增API无需依赖修改 Kubernetes 代码，开发人员自己编写 ApiServer 部署在 Kubernetes 集群中， 并通过 apiservice 资源将自定义资源的 group name 和 apiserver 的 service name 等信息注册到 Kubernetes 集群上，当 Kubernetes ApiServer 收到自定义资源请求时，根据 apiservice 资源信息转发到自定义的 apiserver， 支持 kubectl、支持配置鉴权、授权、审计，支持自定义第三方 etcd 存储，支持 subResource log/exec 等其他高级特性定制化开发。

总体来说，CRD提供了简单、无需任何编程的扩展资源创建、存储能力，而 Aggreated API Server 提供了一种机制，让你能对 API 行为有更精细化的控制能力，允许你自定义存储、使用 Protobuf 协议等。



#### 增强型 Workload

为了满足业务上述的原地更新、指定Pod更新等高级特性需求，腾讯内部及社区都提供了相应的解决方案。腾讯内部有经过大规模生产环境检验的 StatefulSetPlus（未开源的）和 **tkestack TAPP**[2]（已开源），社区也还有有阿里的开源项目 Openkruise，pingcap 为了解决 StatefulSet 指定 Pod 更新问题也推出了一个目前还处于试验状态的 advanced-statefulset 项目。

StatefulSetPlus 是为了满足腾讯内部大量传统业务上 Kubernetes 而设计的， 它在兼容 StatefulSet 全部特性的基础上，支持容器原地升级，对接了 TKE 的 ipamd 组件，实现了固定IP，支持 HPA，支持 Node 不可用时，Pod 自动漂移实现自愈，支持手动分批升级等特性。

Openkruise 包含一系列 Kubernetes 增强型的控制器组件，包括 CloneSet、Advanced StatefulSet、SideCarSet等，CloneSet 是个专注解决无状态服务痛点的 Workload，支持原地更新、指定 Pod 删除、滚动更新过程中支持Partition， Advanced StatefulSet 顾名思义，是个加强版的 StatefulSet， 同时支持原地更新、暂停和最大不可用数。

使用增强版的 workload 组件后，你的有状态服务就具备了传统虚拟机、物理机部署模式下的原地更新、固定IP等优越特性。不过，此时你是直接基于 StatefulSetPlus、TAPP 等 workload 容器化你的服务还是基于 Kubernetes 扩展机制定义一个自定义资源， 专门用于描述你的有状态服务各个组件，并基于 StatefulSetPlus、TAPP 等workload 编写自定义的 operator 呢？

前者适合于简单有状态服务场景，它们组件少、管理方便，同时不需要你懂任何 Kubernetes 编程知识，无需开发。后者适用于较复杂场景，要求你懂 Kubernetes 编程模式，知道如何自定义扩展资源、编写控制器。你可以结合你的有状态服务领域知识，基于 StatefulSetPlus、TAPP 等增强型 workload 编写一个非常强大的控制器，帮助你一键完成一个复杂的、多组件的有状态服务创建和管理工作，并具备高可用、自动扩缩容等特性。



#### 基于 operator 扩展

在我们上文的 codis 集群案例中，就可以选择通过 Kubernetes 的 CRD 扩展机制，自定义一个 CRD 资源来描述一个完整的 codis 集群，如下图所示。

![image-20210602221201259](D:\学习资料\笔记\k8s\k8s图\image-20210602221201259.png)

通过 CRD 实现声明式描述完你的有状态业务对象后，我们还需要通过 Kubernetes 提供的 operator 机制来实现你的业务逻辑。**Kubernetes operator 它的核心原理就是控制器思想，从 API Server 获取、监听业务对象的期望状态、实际状态，对比期望状态与实际状态的差异，执行一致性调谐操作，使实际状态符合期望状态。**

![image-20210602221231865](D:\学习资料\笔记\k8s\k8s图\image-20210602221231865.png)

它的核心工作原理如上图（引用自社区）所示。

- 通过 Reflector 组件的 List 操作，从 kube-apiserver 获取初始状态数据（CRD等)。
- 从 List 请求返回结构中获取资源的 ResourceVersion，通过 Watch 机制指定 ResourceVersion 实时监听 List之后的数据变化。
- 收到事件后添加到 Delta FIFO 队列，由 Informer 组件进行处理。
- Informer 将 delta FIFO 队列中的事件转发给 Indexer 组件，Indexer 组件将事件持久化存储在本地的缓存中。
- operator开发者可通过 Informer 组件注册 Add、Update、Delete 事件的回调函数。Informer 组件收到事件后会回调业务函数，比如典型的控制器使用场景，一般是将各个事件添加到 WorkQueue 中，operator 的各个协调 goroutine 从队列取出消息，解析 key，通过 key 从 Informer 机制维护的本地 Cache 中读取数据。
- 比如当收到创建一个 Codis CRD 对象的事件后，发现实际无这个对象相关的 Deployment/TAPP 等组件在运行，这时候你就可以通过的 Deployment API 创建 proxy 服务，TAPP API创建Redis服务等。



### 调度

----

在解决完如何基于 Kubernetes 内置的 workload 和其扩展机制描述、承载你的有状态服务后，你面临的第二个问题就是如何确保有状态服务中“等价”Pod跨故障域部署，确保有状态服务的高可用性？

首先如何理解“等价” Pod 呢？在 codis、TDSQL 集群中，一组 Redis/MySQL 主备实例，负责处理同一个数据分片的请求，通过主备实现高可用。因主备实例 Pod 负责的是同数据分片，因此我们称之为等价 Pod，生产环境期望它们应跨故障域部署。

其次如何理解故障域？故障域表示潜在的故障影响范围，可按范围分为主机级、机架级、交换机级、可用区级等。一组 Redis 主备案例，至少应该实现主机级高可用，任意一个分片所在的主实例所在的节点故障，备实例应自动提升为主，整个 Redis 集群所有分片仍可提供服务。同样，在 TDSQL 集群中，一组 MySQL 实例，至少应该实现交换机、可用区级别容灾，以确保核心的存储服务高可用。

那么如何实现上面所述等价 Pod 跨故障域部署呢？

答案是调度。Kubernetes 内置的调度器可根据你的Pod所需资源和调度策略，自动化的将 Pod 分配到最佳节点，同时它还提供了强大的调度扩展机制，让你轻松实现自定义调度策略。一般情况下，在简单的有状态服务场景下，你可以基于 Kubernetes 提供的亲和和反亲和高级调度策略，实现 Pod 跨故障域部署。

假设希望通过容器化、高可用部署一个含三节点的 etcd 集群，故障域为可用区，每个etcd节点要求分布在不同可用区节点上，我们如何基于 Kubernetes 提供的亲和 (affinity) 和反亲和 (anti affinity) 特性实现跨可用区部署呢？



#### 亲和与反亲和

很简单，我们可以通过给部署 etcd 的 workload 添加如下的反亲和性配置，声明目的 etcd 集群 Pod 的标签，拓扑域为 node 可用区，同时是硬亲和规则，若 Pod不 满足规则将无法调度。

那么调度器又遇到被添加了反亲和配置的 Pod 后是如何调度的呢？

![image-20210602221347371](D:\学习资料\笔记\k8s\k8s图\image-20210602221347371.png)

首先调度器监听到 etcd workload 生成的的待调度 Pod 后，通过反亲和配置中的标签查询出已调度 Pod 的节点、可用区信息，然后在调度器的筛选阶段，若候选节点可用区与已调度 Pod 可用区一致，则淘汰，最后进入评优阶段的节点都是满足 Pod 跨可用区部署等条件限制的节点，根据调度器配置的评优策略，选择出一个最优节点，将 Pod 绑定到此节点上，最终实现 Pod 跨可用区部署、容灾。

然而在 codis 集群、TDSQL 分布式集群等复杂场景中，Kubernetes 自带的调度器可能就无法满足你的诉求了，但是它提供了如下的扩展机制帮助你自定义调度策略，实现各种复杂场景的调度诉求。



#### 自定义调度策略、extend scheduler 等

首先你可以修改调度器的筛选/断言 (predicates) 和评分/优先级 (priorities) 策略， 配置满足你业务诉求的调度策略。比如你希望降低成本，用最小的节点数支撑集群所有服务，那么我们需要让 Pod 尽量优先往满足其资源诉求、已分配资源较多的节点上调度。此场景，你就可以通过修改 priorities 策略，配置 MostRequestedPriority 策略，调大权重。

然后你可以基于 Kubernetes 调度器实现 extend scheduler， 在调度器的 predicates 和 priorities 阶段，回调你的扩展调度服务，已满足你的调度诉求。比如你希望负责同一个数据分片的一组 MySQL 或 Redis 主备实例实现跨节点容灾，那么你就可以实现自己的predicates 函数，将同组已调度 Pod 的节点从候选节点中删除，保证进入 priorities 流程的节点都是满足你业务诉求的。

接着你可以基于 Kubernetes 的调度器实现自己独立的调度器，部署独立的调度器到集群后，你只需要通过将 Pod的 schedulerName 声明为你独立的调度器即可。

![image-20210602221432608](D:\学习资料\笔记\k8s\k8s图\image-20210602221432608.png)



#### scheduler framwork

最后 Kubernetes 在1.15版本中推出了一个新的调度器扩展框架，它在调度器的核心流程前后都增加了 hook。选择待调度 Pod，支持自定义排队算法，筛选流程提供了 PreFilter 和 Filter 接口，评分流程增加了 PreScore，Score，NormalizeScore 等接口，绑定流程提供 PreBind 和 Bind，PostBind 三个接口。基于新的调度器扩展框架，业务可更加精细化、低成本的控制调度策略，自定义调度策略更加简单、高效。

![image-20210602221510198](D:\学习资料\笔记\k8s\k8s图\image-20210602221510198.png)



### 高可用

---

解决完调度问题后，我们的有状态服务已经可以高可用的部署了。然而高可用部署不代表服务能高可用的对外的提供服务，容器化后我们也许会遇到比传统物理机、虚拟机模式部署更多的稳定性挑战。稳定性挑战可能来自业务编写的 operator、Kubernetes 组件、docker/containerd 等运行时组件、linux 内核等，那如何应对以上各种因素导致的稳定性问题呢？

我们在设计上应把 Pod 异常当作常态化案例处理，任一 Pod 异常后，在容器化场景中，我们应当具备自愈的机制。若是无状态服务，我们只需为业务Pod添加合理的存活和就绪检查即可，Pod 异常后自动重建，节点故障 Pod 自动漂移到其他节点。然而在有状态服务场景中，即便承载你有状态服务的 workload，支持节点故障后 Pod 自动漂移功能，却也可能会因 Pod 自愈时间过长和数据安全性等无法满足业务诉求，为什么呢？

假设在 codis 集群中，一个 Redis 主节点所在node突然”失联“了，此时若等待5分钟才进入自愈流程，那么对外将造成5分钟的不可用性， 显然对重要的有状态服务场景是无法接受的。即便你缩小节点失联自愈时间，你也无法保证其数据安全性，万一此时集群网络出现了脑裂，失联节点也在对外提供服务，那么将出现多个 master 双写，最终可能导致数据丢失。

那么有状态的服务安全的高可用解决方案是什么呢？这取决于有状态服务本身高可用实现机制，Kubernetes 容器平台层是无法提供安全的解决方案。常用的有状态服务高可用解决方案有主备复制、去中心化复制、raft/paxos 等共识算法，下面我分别简易阐述三者的区别和优劣势，以及介绍在容器化过程中的注意事项。



#### 主备复制

像我们上面讨论的 codis 集群案例、TDSQL 集群案例都是基于主备复制实现的高可用，实现上相比去中心化复制、共识算法较简单。主备复制又可分为主备全同步复制、异步复制、半同步复制。

全同步复制是指主收到一个写请求后，必须等待全部从节点确认返回后，才能返回给客户端成功，因此若一个从节点故障，整个系统就会不可用，这种方案为了保证多副本集的一致性，而牺牲了可用性，一般使用不多。

异步复制是指主收到一个写请求后，可及时返回给 client，异步将请求转发给各个副本， 但是若还未将请求转发到副本前就故障了，则可能导致数据丢失，但可用性是最高的。

半同步复制介于全同步复制、异步复制之间，它是指主收到一个写请求后，至少有一个副本接收数据后，就可以返回给客户端成功，在数据一致性、可用性上实现了平衡和取舍。

基于主备复制模式实现的有状态服务，业务需要实现、部署主备切换的 HA 服务，HA服务按实现架构，可分为主动上报型和分布式探测型。主动上报型以 TDSQL 中 MySQL 主备切换为例，各个 MySQL 节点上部署有 agent， 向元数据存储集群 (zookeeper/etcd) 上报心跳，若 master 心跳丢失， HA 服务将快速发起主备切换。分布式探测型以 Redis sentinel 为例，部署奇数个哨兵节点，各个哨兵节点定时探测Redis主备实例的可用性，彼此之间通过 gossip 协议交互探测结果，若对一个主 Redis 节点故障达到多数派认可，那么就由其中一个哨兵发起主备切换流程。

总体来说，基于主备复制的有状态服务，在传统的部署模式，节点故障后，依赖运维、人工替换节点。容器化后的有状态服务，可通过 operator 实现故障节点自动替换、快速垂直扩容等特性，显著降低运维复杂度，但是 Pod 可能会发生重建等，应部署负责主备切换的HA服务，负责主备 Pod 的切换，以提高可用性。若业务对数据一致性非常敏感，较频繁的切换的可能会导致增大丢失数据的概率，可通过使用 dedicated 节点、稳定及较新的运行时和Kubernetes 版本等减少不稳定因素。



#### 去中心化复制

跟主从复制相反的就是去中心化复制，它是指在一个n副本节点集群中，任意节点都可接受写请求，但一个成功的写入需要w个节点确认，读取也必须查询至少r个节点。你可以根据实际业务场景对数据一致性的敏感度，设置合适w/r参数。比如你希望每次写入后，任意client都能读取到新值，若n是3个副本，你可以将w和r设置为2，这样当你读两个节点时候，必有一个节点含有最近写入的新值，这种读我们称之为法定票数读 (quorum read)。

AWS 的 dynamo 系统就是基于无中心化的复制算法实现的，它的优点是节点角色都是平等的，降低运维复杂度，可用性更高，容器化难度更低，无需部署HA组件等，但缺陷是去中心化复制，务必会导致各种写入冲突，业务需要关注冲突处理等。



#### 共识算法

基于复制算法实现的数据库，为了保证服务可用性，大多数提供的是最终一致性，不管是主从复制还是去中心化复制，都存在一定的缺陷，无法满足数据强一致、高可用的诉求。

如何解决以上复制算法的困境呢？

答案就是 raft/paxos 共识算法，它最早是基于复制状态机背景下提出来的，由共识模块、日志模块、状态机组成， 如下图（引用自 Raft 论文)。通过共识模块保证各个节点日志的一致性，然后各个节点基于同样的日志、顺序执行指令，最终各个复制状态机的结果是一致性的。这里我以 raft 算法为例，它由 leader 选举、日志复制、安全性组成，leader 节点故障后，follower 节点可快速发起新的 leader 选举，并确保数据安全性，follwer 节点故障后，只要多数节点存活，就不影响集群整体可用性。

![image-20210602221646797](D:\学习资料\笔记\k8s\k8s图\image-20210602221646797.png)

基于共识算法实现的有状态服务，典型案例是 etcd/zookeeper/tikv 等，在此架构中，服务本身集成了 leader 选举算法、数据同步机制，使得运维和容器化复杂度相比主备复制的服务要显著降低，容器化更加安全。即便容器化过程中遇上 Bug 导致 leader 节点故障，得益于共识算法，数据安全和服务可用性几乎不受任何影响，因此优先推荐将使用共识算法的有状态服务进行容器化。



### 高性能

---

实现完有状态服务在 Kubernetes 中更稳的运行的目标后，下一步目标则是追求高性能、更快，而有状态服务的高性能又依托底层容器化网络方案、磁盘 IO 方案。在传统的物理机、虚拟机部署模式中，有状态服务拥有固定的IP、高性能的 underlay 网络、高性能的本地 SSD 磁盘，那么在容器化后，如何达到传统模式的性能呢？我将分别从网络和存储分别简易阐述 Kubernetes 的解决方案。



#### 可扩展的网络解决方案

首先是可扩展、插件化的网络解决方案。得益于 Google 多年的 Borg 容器化运行经验和教训，在 Kubernetes 的网络模型中，每个 Pod 拥有独立的IP，各个 Pod 可以跨主机通信而需NAT， 同时 Pod 也可以与 Node 节点实现网络互通。Kubernetes 优秀的网络模型良好的兼容了传统的物理机、虚拟机业务的网络方案，让传统业务上Kubernetes 更加简单。最重要的是，Kubernetes 提供了开放的 CNI 网络插件标准，它描述了如何为 Pod 分配 IP和实现 Pod 跨节点容器互通，各个开源、云厂商可以基于自己业务业务场景、底层网络，实现高性能、低延迟的CNI插件，最终达到跨节点容器互通。

在基于 CNI 实现的各种 Kubernetes 的网络解决方案中，按数据包的收发模式实现可分为 underlay 和 overlay 两类。前者是直接基于底层网络，实现互联互通，拥有良好的性能，后者是基于隧道转发，它是在底层网络的基础上，加上隧道技术，构建一个虚拟的网络，因此存在一定的性能损失。

这里我分别以开源的 flannel 和 tke 集群网络方案为例，阐述各自的解决方案、优缺点。

在 flannel 中，它设计上后端支持 udp、vxlan、host-gw 等多种转发模式。udp 和 vxlan 转发模式是基于 overlay隧道转发模式实现，它支持将原始请求封装在 udp、vxlan 数据包内，然后基于 underlay 网络转发给目的容器。udp 是在用户态进行数据的封解包操作，性能较差，一般用于debug和不支持 vxlan 协议的低版本内核。vxlan 是在内核态完成了数据的封解包操作，性能损失较小。host-gw 模式则是直接通过下发每个子网的IP路由信息到各个节点上，实现跨主机的 Pod 网络通信，无需数据包的封解包操作，相比 udp/vxlan，性能最佳，但要求各主机节点的二层网络是连通的。

在tke集群网络方案中，我们也支持多种网络通信方案，经历了从 global route、VPC-CNI 到 Pod 独立网卡的三种模式的演进。global route 即全局路由，每个节点加入集群时，会分配一个唯一的 Pod cidr， tke 会通过 VPC 的接口下发全局路由到用户 VPC 的子机所在的母机上。当用户 VPC 的容器、节点访问的ip属于此 Pod cir 时，就会匹配到此全局路由规则，转发到目标节点上。此方案中 Pod CIDR 并不属于VPC资源，因此它不是 VPC 的一等公民，无法使用 VPC 的安全组等特性，但是其简单、同时在用户VPC层不需要任何的数据解封包操作，性能无较大的损失。

为了解决容器 Pod IP 不是 VPC 一等公民而导致一系列 VPC 特性无法使用的问题，tke 集群实现了 VPC-CNI 网络模式，Pod IP 来自用户 VPC 的子网，跨节点容器网络通信、节点与容器通信与 VPC 内的 CVM 节点通信原理一致，底层都是基于 VPC 的 GRE 隧道路由转发实现，数据包在节点内通过策略路由转发到目标容器。基于此方案，容器 Pod IP 可享受 VPC 的特性，实现CLB直连Pod，固定IP等一系列高级特性。

近期为了满足游戏、存储等业务对容器网络性能更加极致的要求，TKE 团队又推出了下一代网络方案，Pod 独占弹性网卡的 VPC-CNI 模式，不再经过节点的网络协议栈，极大缩短容器访问链路和延时，并使 PPS 可以达到整机上限。基于此方案我们实现了 Pod 绑定 EIP/NAT，不再依赖节点的外网访问能力，支持 Pod 绑定安全组，实现Pod级别的安全隔离，详细可阅读文章末尾的相关文章。

基于 Kubernetes 的可扩展网络模型，业务可以实现特定场景的高性能网络插件。比如腾讯内部的 tenc 平台，基于 SR-IOV 技术的实现了 sriov-cni CNI 插件，它可以给 Kubernetes 提供高性能的二层VLAN网络解决方案。特别是对网络性能要求高的场景，比如分布式机器学习训练，游戏后端服务等。



#### 可扩展的存储解决方案

介绍完可扩展的网络解决方案后，有状态服务的另一大核心瓶颈则是高性能存储IO诉求。在传统的部署模式中，有状态服务一般使用的是本地硬盘，并根据服务的类型、规格、对外的 SLA，选择 HDD、SSD 等不同类型的磁盘。那么在 Kubernetes 中如何满足不同场景下的存储诉求呢？

在 Kubernetes 存储体系中，此问题被拆分成若干个子问题来优雅解决，并具备良好的可扩展性、可维护性，无论是本地盘、还是云盘、NFS等文件系统都可基于其扩展实现相应的插件， 并实现了开发、运维职责分离。

那么 Kubernetes 的存储体系是如何构建的呢？

我通过如何给你的有状态Pod应用挂载一个数据存储盘为案例，来介绍 Kubernetes 的可扩展存储体系，它可以分为以下步骤:

- 应用如何申请一个存储盘呢？（消费者)
- Kubernetes 存储体系是如何描述一个存储盘的呢？人工创建存储盘呢还是自动化按需创建存储盘？(生产者)
- 如何将存储资源池的盘与存储盘申请者的诉求进行匹配？(控制器)
- 如何描述存储盘的类型、数据删除策略、以及此类型盘的服务提供者信息呢？(storageClass)
- 如何实现对应的存储数据卷插件？(FlexVolume、CSI）

首先 Kubernetes 中提供了一个名为PVC的资源，描述应用申请的存储盘的类型、访问模式、容量规格，比如你想给etcd服务申请一个存储类为cbs， 大小100G的云盘，你可以创建一个如下的PVC。

![image-20210602221804925](D:\学习资料\笔记\k8s\k8s图\image-20210602221804925.png)

其次 Kubernetes 中提供了一个名为 PV 的资源，描述存储盘的类型、访问模式、容量规格，它对应一块真实的磁盘，支持通过人工和自动创建两种模式。下图描述的是一个 100G 的 cbs 硬盘。

![image-20210602221828472](D:\学习资料\笔记\k8s\k8s图\image-20210602221828472.png)

接着，当应用创建一个 PVC 资源时，Kubernetes 的控制器就会尝试将其与PV进行匹配，存储盘的类型是否一致、PV的容量大小是否满足 PVC 的诉求，若匹配成功，此 PV 的状态会变成绑定， 控制器会进一步的将此PV对应的存储资源attach到应用 Pod 所在节点上，attach 成功后，节点上的 kubelet 组件会将对应的数据目录挂载到存储盘上，进而实现读写。

以上就是应用申请一个盘的流程，那么在容器中如何通过 PV/PVC 这套体系实现支持多种类型的块存储和网络文件系统呢？比如块存储服务支持普通 HHD 云盘，SSD 高性能云盘，SSD 云盘，本地盘等，远程网络文件系统支持NFS等。其次是Kubernetes 控制器如何按需动态的去创建PV呢？

为了支持多种类型的存储诉求，Kubernetes 提供了一个 StorageClass 的资源来描述一个存储类。它描述了存储盘的类别、绑定和删除策略、由什么服务组件提供资源创建。比如高性能版和基础版的 MySQL 服务依赖不同类型的存储磁盘，你只需要创建 PVC 的时候填写相应的 storageclass 名字即可。

最后，Kubernetes 为了支持开源社区、云厂商众多的存储数据卷，提供了存储数据卷扩展机制，从早期的 in-tree 的内置数据卷、到 FlexVolume 插件、再到现在已经 GA 的的容器化存储 CSI 插件机制， 存储服务提供商可将任意的存储系统集成到Kubernetes存储体系中。比如 storage cbs 的 provisioner 是腾讯云的 TKE 团队，我们会基于 Kubernetes 的 flexvolume/CSI 扩展机制，通过腾讯云 CBS 的 API 实现创建、删除 cbs 硬盘。

![image-20210602221850285](D:\学习资料\笔记\k8s\k8s图\image-20210602221850285.png)

为了满足有状态等服务对磁盘IO性能的极致追求，Kubernetes 基于以上介绍的 PV/PVC 存储体系，实现了 local pv 机制，它可避免网络 IO 开销，让你的服务拥有更高的IO读写性能。local pv 核心是通过将本地盘、lvm 分区抽象成 PV，使用 local pv 的 Pod，依赖延迟绑定特性实现准确调度到目标节点。

local pv的关键核心技术点是容量隔离(lvm、xfs quota)、IO隔离(cgroup v1一般要定制内核，cgroup v2支持buffer io等)、动态provision等问题，为了解决以上或部分痛点，社区也诞生了一系列的开源项目，如TopoLVM（支持动态provision、lvm)，sig-storage-local-static-provisioner等项目，各云厂商如腾讯内部也有相应的local pv解决方案。总体而言，local pv适用于磁盘io敏感型的etcd、MySQL、tidb等存储服务，如pingcap的tidb项目就推荐在生产环境使用local pv。

local pv 的缺点是节点故障后，数据无法访问、可能丢失、无法垂直扩容(受限于节点磁盘容量等)。因此这对有状态服务本身和其 operator 提出了更高要求，服务本身需要通过主备复制协议和共识算法，保证数据安全性。任一节点故障后，operator 能及时扩容新节点，从冷备、leader 快照进行数据恢复。如 tidb 的 tikv 服务，当检测到实例有异常后，会自动扩容新实例，通过 raft 协议完成数据的复制等。



### 混沌工程

---

通过以上技术方案，解决了负载类型选型、自定义资源扩展、调度、高可用、高性能网络、高性能存储、稳定性测试等一系列痛点后，我们可基于 Kubernetes 的构建稳定、高可用、弹性伸缩的有状态服务。

那么如何验证容器化后的有状态服务稳定性呢？

社区提供了多个基于 Kubernetes 实现的混沌工程开源项目，比如 pingcap 的 chaos-mesh， 提供了 Pod chaos/Network chaos/IO chaos 等多种故障注入。基于 chaos mesh，你可以快速注入 Pod 故障、磁盘IO、网络IO等异常到集群中任意 Pod，帮助你快速发现有状态服务本身和 operator Bug、检验集群的稳定性。在 TKE 团队中，我们基于 chaos mesh 排查、复现 etcd Bug， 压测 etcd 集群的稳定性，极大的降低了我们复现复杂 Bug 的难度，帮助我们提升 etcd 内核的稳定性。



### 总结

---

本文通过从有状态集群中的各个组件 workload 选型、扩展机制的选择，介绍了如何使用 K8s 的描述、部署你的有状态服务。有状态服务出于其特殊性，数据安全、高可用、高性能是其核心目标，为了保证服务的高可用，可通过调度和HA服务来实现。通过 Kubernetes 的多种调度器扩展机制，你可以将你的有状态服务的等价 Pod 完成跨故障域部署。通过主备切换服务和共识算法，你可以完成主节点故障后，备节点自动提升为主，以保证服务的高可用性。高性能主要取决于网络和存储性能，Kubernetes 提供了 CNI 网络模型和 PV/PVC 存储体系、CSI 扩展机制来满足各种业务场景下的定制需求。最后介绍了混沌工程在有状态服务中的应用，通过混沌工程你可以模拟各类异常场景下，你的有状态服务容错性，帮助你检验和提升系统的稳定性。



#### 参考资料

[1]Codis项目: *https://github.com/CodisLabs/codis*

[2]tkestack TAPP: *https://github.com/tkestack/tapp*

[3]custom resources: *https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/*

[4]Extend Kubernetes: *https://kubernetes.io/docs/concepts/extend-kubernetes/*

[5]tidb-operator: *https://github.com/pingcap/tidb-operator*

[6]chaos mesh: *https://github.com/chaos-mesh/chaos-mesh*

[7] ***[腾讯云容器服务TKE推出新一代零损耗容器网络](http://mp.weixin.qq.com/s?__biz=Mzg5NjA1MjkxNw==&mid=2247489272&idx=1&sn=c9a260a9a7cc85c233fc2d60a580153c&chksm=c007ad22f7702434a3b19194b4509f6e91faee140dfc01b6fe1a14e2abad3b043b4933b9aa81&scene=21#wechat_redirect)\***









































