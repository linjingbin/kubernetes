k8s 笔记

[toc]

# K8s 学习者的知识图谱

容器服务 Kubernetes 知识图谱，部分内容参考网上一知识图谱，更加结合阿里云容器服务。

![图片](https://mmbiz.qpic.cn/mmbiz_png/yvBJb5Iiafvk3b362srzPuh3gSXbZYEQcicvEjDYNYAoljEyCOUBSFn7FSnOtokibjbtLv3jpiajnjqY7CgEQ3ibfRA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

原图链接地址

https://www.processon.com/view/link/5ac64532e4b00dc8a02f05eb#map



**知识链接和备注**



### **Docker 原理**

- KVM--> ECS

https://blog.csdn.net/weixin_43695104/article/details/88554443#32_kvm_web_192

- 网络隧道技术-->VPC

https://blog.csdn.net/wangjianno2/article/details/75208036

- NameSpace

https://blog.csdn.net/a352193394/article/details/53344167

备注：Linux 容器中用来实现“隔离”的技术手段：Namespace，Namespace 技术实际上修改了应用进程看待整个计算机的范围，它的访问范围被操作系统做了限制，只能“看到”某些指定的内容。

- CGroup

https://blog.csdn.net/wudongxu/article/details/8474198

备注：Linux Control Group。它最主要的作用，就是限制一个进程组能够使用的资源上限，包括 CPU、内存、磁盘、网络带宽等等。

- RootFS(Union FS)

https://coolshell.cn/articles/17061.html

备注：rootfs 只是一个操作系统所包含的文件、配置和目录，并不包括操作系统内核。在 Linux 操作系统中，这两部分是分开存放的，操作系统只有在开机启动时才会加载指定版本的内核镜像。

- windows 2019

备注：windowserver 2019开始支持 namespace



### **容器服务部署**

- Docker Desktop

https://www.docker.com/products/docker-desktop

备注：Mac 机器上强烈建议安装该软件作为学习使用

- kubernetes

http://docs.kubernetes.org.cn/

备注：kubernetes 集群，aliyun容器服务支持

- DashBoard

https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/

备注：kubernetes 集群的图形界面管理工具，容器服务控制台整合了该应用并扩展

- EasyPack

https://github.com/liumiaocn/easypack

备注：一批部署 kubernetes 等集群的脚本集合

- minikube

https://kubernetes.io/docs/tasks/tools/install-minikube/ 

备注：mini 新 K8s



### **工具组件**

- kubectl

http://docs.kubernetes.org.cn/61.html

备注：kubectl 用于运行 Kubernetes 集群命令的管理工具

- kubeadm

https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm/Kubernetes

备注：官方提供的用于快速安装配置 Kubernetes 集群的工具

- Helm

https://helm.sh/zh/docs/

备注：类似 rpm，yum，是 K8s 用于安装组件（软件包：chart）的工具

- APP Hub

https://developer.aliyun.com/hub

备注：在开放云原生应用中心当中，所有默认的 Helm Charts（Helm 格式的应用），都定时同步自 Helm Hub 北美官方站并托管在 Github 上。在这个过程中，云原生应用中心会自动对同步过来的所有 Charts 进行“本地化”操作。

- CFSSL

https://github.com/cloudflare/cfssl

备注：CFSSL 是开源的一款 PKI/TLS 工具，常用于 K8s 证书制作



### **镜像仓库**

- aliyun 私有镜像仓库

https://cr.console.aliyun.com/aliyun 

备注：推出的镜像仓库，建议采用企业版

- 云效配置镜像仓库

https://cn.aliyun.com/product/yunxiao

备注：云效企业设置，配置支持从阿里云私有镜像仓库拉取镜像

- Harbor 镜像仓库

https://goharbor.io

备注：开源免费的存储和分发Docker镜像的企业级Registry服务器



### **组件**

- kube-apiserver(Master)

https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/

备注：在 generic server 上封装的一层官方默认的 apiserver(static pod)

- etcd(Master)

https://etcd.io

备注：类 zk 基于 Raft 协议的实现，启动进程

- Kube-scheduler(Master)

https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/

备注：负责 pod 分布到 Node 上的调度器 (static pod)

- kube-controller-manager(Master)

https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/

备注：Deployment 等基础对象的控制器 (static pod)

- cloud-controller-manager(Master)

https://kubernetes.io/docs/reference/command-line-tools-reference/cloud-controller-manager/

备注：用于云资源使用的控制器，是云服务进行集成的控制器 (Daemonset)

- kubelet(Node)

https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/

备注：与 Master 通信,对 worker(Node) 进行生命周期管理

- kube-proxy(Node)

https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/

备注：节点上运行的网络代理 (Daemonset)

- containner runtime(Node)

备注：CRI 接口

- DNS

https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/

备注：aliyun容器服务采用 CoreDNS(deployment)

- Ingress controller

https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/

备注：aliyun容器服务采用nginx ingress controller, 可以作为 https 服务的统一路由(deployment)

- Heapster & influxdb 

备注：监控数据采集与存储用的时序数据库(Deployment)

- Federation

https://kubernetes.io/docs/concepts/cluster-administration/federation/

备注：集群联盟，实现高可用，同步资源等

- kube-flannel

备注：官方网络插件，aliyun 另外提供了自己开发的 Terway 组件(daemonset)

- logtail

https://help.aliyun.com/document_detail/28979.html?spm=a2c4g.11186623.6.595.439d7218wQhzsH

备注：aliyun 日志采集组件 (daemonset)



### **基础对象**

- POD

http://docs.kubernetes.org.cn/312.html  

容器组，运行应用容器基本单位，kubectl get pods 

- Node

http://docs.kubernetes.org.cn/304.html

集群节点服务器，Kubernetes中的工作节点。

- NameSpace

http://docs.kubernetes.org.cn/242.html

备注：用以区分和隔离应用

- Deployement

http://docs.kubernetes.org.cn/317.html

备注：无状态部署，最常用部署配置

- Daemonset

https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/

备注：类似守护进程

- StatefulSet

http://docs.kubernetes.org.cn/443.html

备注：有状态部署

- Job & CronJob

https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/

备注：调度任务

- Static POD

https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/

备注：静态 pod 配置，yaml 位于 Master

- HPA

https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/

备注：水平伸缩调度器

- Service

https://kubernetes.io/docs/concepts/services-networking/service/

备注：服务暴露配置，包括 Cluster,NodePort,SLB 等

- Ingress

https://www.kubernetes.org.cn/1885.html

备注：路由，阿里云默认提供 nginx ingress

- Secret

https://kubernetes.io/docs/concepts/configuration/secret/

备注：保密字典，包括 tls,私有仓库密钥，Opaque 几种

- ServiceAccount

https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/

备注：用于资源对象的账号，比如给一个 Namespace 授予某私有镜像访问权限

- RBAC

https://kubernetes.io/docs/reference/access-authn-authz/rbac/

备注：K8s 基于角色的访问控制，role,rolebinding

- Volume

https://kubernetes.io/docs/concepts/storage/volumes/

备注：映射磁盘

- Storge Class

https://kubernetes.io/docs/concepts/storage/storage-classes/

- CustomResourceDefinition

备注：自定义扩展资源



### **插件扩展**

- CNI(Falnnel/Terway)

https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/

备注：容器网络接口

- FlexVolume

https://github.com/fstab/cifs

备注：开源 Volume 实现插件，阿里云使用中

- Cloud Provider

备注：云服务供应接口



### **容器服务优化-最佳实践**

- Master 选型及磁盘规格

[1] https://yq.aliyun.com/articles/599169?spm=5176.11065265.1996646101.searchclickresult.7bea1a8bgCTYH7

[2] https://yq.aliyun.com/articles/621108?

- 网络选择

https://yq.aliyun.com/articles/594943?

- Worker 节点选型

https://yq.aliyun.com/articles/602932?spm=a2c4e

- Ingress Controller 独立部署

- Master 变配

https://help.aliyun.com/document_detail/123661.html?spm=5176.10695662.1996646101.searchclickresult.20d0328c6WG7jc

- 节点变配或重启、摘除、加入
- 基础镜像开发
- Service 与 SLB 结合

- 集群审计

https://help.aliyun.com/document_detail/91406.html?spm=5176.10695662.1996646101.searchclickresult.45266c92kGHQrP

- Deployment实现分批发布
- StatefulSet 分批发布

https://yq.aliyun.com/articles/622898?spm=a2c4e.11155435.0.0.1b8e3312bSGmSe

- 堡垒机上按照应用设置权限

https://yq.aliyun.com/articles/715809

- Pod 均匀分布部署

https://yq.aliyun.com/articles/715808

- 应用优雅下线，优雅退出



- ApiServer 访问



- 监控



### **服务治理**

- Istio

https://istio.io

备注：当前最流行的网格服务架构，aliyun支持

- Linkerd

https://linkerd.io/2/overview/

备注：最早提出网格服务公司的产品 

- 云效

https://www.aliyun.com/product/yunxiao

备注：支持容器服务 K8s 的 CI/CD 阿里云上产

- Jenkins

https://jenkins.io/zh/

备注：著名的最常用的 CI/CD 产品，容器服务由一键安装产品



**云原生技术公开课**

https://edu.aliyun.com/roadmap/cloudnative





# Kubernetes 基础



## Containerd 的前世今生和保姆级入门教程



### **1. Containerd 的前世今生**

很久以前，Docker 强势崛起，以“镜像”这个大招席卷全球，对其他容器技术进行致命的降维打击，使其毫无招架之力，就连 Google 也不例外。Google 为了不被拍死在沙滩上，被迫拉下脸面（当然，跪舔是不可能的），希望 Docker 公司和自己联合推进一个开源的容器运行时作为 Docker 的核心依赖，不然就走着瞧。Docker 公司觉得自己的智商被侮辱了，走着瞧就走着瞧，谁怕谁啊！

很明显，Docker 公司的这个决策断送了自己的大好前程，造成了今天的悲剧。

紧接着，Google 联合 Red Hat、IBM 等几位巨佬连哄带骗忽悠 Docker 公司将 `libcontainer` 捐给中立的社区（OCI，Open Container Intiative），并改名为 `runc`，不留一点 Docker 公司的痕迹~~

这还不够，为了彻底扭转 Docker 一家独大的局面，几位大佬又合伙成立了一个基金会叫 `CNCF`（Cloud Native Computing Fundation），这个名字想必大家都很熟了，我就不详细介绍了。CNCF 的目标很明确，既然在当前的维度上干不过 Docker，干脆往上爬，升级到大规模容器编排的维度，以此来击败 Docker。

Docker 公司当然不甘示弱，搬出了 Swarm 和 Kubernetes 进行 PK，最后的结局大家都知道了，Swarm 战败。然后 Docker 公司耍了个小聪明，将自己的核心依赖 `Containerd` 捐给了 CNCF，以此来标榜 Docker 是一个 PaaS 平台。

很明显，这个小聪明又大大加速了自己的灭亡。



### **2. Containerd 架构**

时至今日，Containerd 已经变成一个工业级的容器运行时了，连口号都有了：超简单！超健壮！可移植性超强！

当然，为了让 Docker 以为自己不会抢饭碗，Containerd 声称自己的设计目的主要是为了嵌入到一个更大的系统中（暗指 Kubernetes），而不是直接由开发人员或终端用户使用。

事实上呢，Containerd 现在基本上啥都能干了，开发人员或者终端用户可以在宿主机中管理完整的容器生命周期，包括容器镜像的传输和存储、容器的执行和管理、存储和网络等。大家可以考虑学起来了。

先来看看 Containerd 的架构：

![image-20210111173206883](D:\学习资料\笔记\k8s\k8s图\image-20210111173206883.png)

可以看到 Containerd 仍然采用标准的 C/S 架构，服务端通过 `GRPC` 协议提供稳定的 API，客户端通过调用服务端的 API 进行高级的操作。

为了解耦，Containerd 将不同的职责划分给不同的组件，每个组件就相当于一个**子系统**（subsystem）。连接不同子系统的组件被称为模块。

总体上 Containerd 被划分为两个子系统：

- **Bundle** : 在 Containerd 中，`Bundle` 包含了配置、元数据和根文件系统数据，你可以理解为容器的文件系统。而 **Bundle 子系统**允许用户从镜像中提取和打包 Bundles。
- **Runtime** : Runtime 子系统用来执行 Bundles，比如创建容器。

其中，每一个子系统的行为都由一个或多个**模块**协作完成（架构图中的 `Core` 部分）。每一种类型的模块都以**插件**的形式集成到 Containerd 中，而且插件之间是相互依赖的。例如，上图中的每一个长虚线的方框都表示一种类型的插件，包括 `Service Plugin`、`Metadata Plugin`、`GC Plugin`、`Runtime Plugin` 等，其中 `Service Plugin` 又会依赖 Metadata Plugin、GC Plugin 和 Runtime Plugin。每一个小方框都表示一个细分的插件，例如 `Metadata Plugin` 依赖 Containers Plugin、Content Plugin 等。总之，万物皆插件，插件就是模块，模块就是插件。

![image-20210111173510434](D:\学习资料\笔记\k8s\k8s图\image-20210111173510434.png)

这里介绍几个常用的插件：

- **Content Plugin** : 提供对镜像中可寻址内容的访问，所有不可变的内容都被存储在这里。
- **Snapshot Plugin** : 用来管理容器镜像的文件系统快照。镜像中的每一个 layer 都会被解压成文件系统快照，类似于 Docker 中的 `graphdriver`。
- **Metrics** : 暴露各个组件的监控指标。

从总体来看，Containerd 被分为三个大块：`Storage`、`Metadata` 和 `Runtime`，可以将上面的架构图提炼一下：

![image-20210111173708946](D:\学习资料\笔记\k8s\k8s图\image-20210111173708946.png)

这是使用 **bucketbench**[1] 对 `Docker`、`crio` 和 `Containerd` 的性能测试结果，包括启动、停止和删除容器，以比较它们所耗的时间：

![image-20210111173754617](D:\学习资料\笔记\k8s\k8s图\image-20210111173754617.png)

可以看到 Containerd 在各个方面都表现良好，总体性能还是优越于 `Docker` 和 `crio` 的。



### 3. Containerd 安装

了解了 Containerd 的概念后，就可以动手安装体验一把了。本文的演示环境为 `Ubuntu 18.04`。

#### 安装依赖

为 seccomp 安装依赖：

```sh
$ sudo apt-get update
$ sudo apt-get install libseccomp2
```



#### 下载并解压 Containerd 程序

Containerd 提供了两个压缩包，一个叫 `containerd-${VERSION}.${OS}-${ARCH}.tar.gz`，另一个叫 `cri-containerd-${VERSION}.${OS}-${ARCH}.tar.gz`。其中  `cri-containerd-${VERSION}.${OS}-${ARCH}.tar.gz` 包含了所有 Kubernetes 需要的二进制文件。如果你只是本地测试，可以选择前一个压缩包；如果是作为 Kubernetes 的容器运行时，需要选择后一个压缩包。

Containerd 是需要调用 `runc` 的，而第一个压缩包是不包含 `runc` 二进制文件的，如果你选择第一个压缩包，还需要提前安装 runc。所以我建议直接使用 `cri-containerd` 压缩包。

首先从 **release 页面**[2]下载最新版本的压缩包，当前最新版本为 1.4.3：

```sh
$ wget https://github.com/containerd/containerd/releases/download/v1.4.3/cri-containerd-cni-1.4.3-linux-amd64.tar.gz

# 也可以替换成下面的 URL 加速下载
$ wget https://download.fastgit.org/containerd/containerd/releases/download/v1.4.3/cri-containerd-cni-1.4.3-linux-amd64.tar.gz
```

可以通过 tar 的 `-t` 选项直接看到压缩包中包含哪些文件：

```sh
$ tar -tf cri-containerd-cni-1.4.3-linux-amd64.tar.gz
etc/
etc/cni/
etc/cni/net.d/
etc/cni/net.d/10-containerd-net.conflist
etc/crictl.yaml
etc/systemd/
etc/systemd/system/
etc/systemd/system/containerd.service
usr/
usr/local/
usr/local/bin/
usr/local/bin/containerd-shim-runc-v2
usr/local/bin/ctr
usr/local/bin/containerd-shim
usr/local/bin/containerd-shim-runc-v1
usr/local/bin/crictl
usr/local/bin/critest
usr/local/bin/containerd
usr/local/sbin/
usr/local/sbin/runc
opt/
opt/cni/
opt/cni/bin/
opt/cni/bin/vlan
opt/cni/bin/host-local
opt/cni/bin/flannel
opt/cni/bin/bridge
opt/cni/bin/host-device
opt/cni/bin/tuning
opt/cni/bin/firewall
opt/cni/bin/bandwidth
opt/cni/bin/ipvlan
opt/cni/bin/sbr
opt/cni/bin/dhcp
opt/cni/bin/portmap
opt/cni/bin/ptp
opt/cni/bin/static
opt/cni/bin/macvlan
opt/cni/bin/loopback
opt/containerd/
opt/containerd/cluster/
opt/containerd/cluster/version
opt/containerd/cluster/gce/
opt/containerd/cluster/gce/cni.template
opt/containerd/cluster/gce/configure.sh
opt/containerd/cluster/gce/cloud-init/
opt/containerd/cluster/gce/cloud-init/master.yaml
opt/containerd/cluster/gce/cloud-init/node.yaml
opt/containerd/cluster/gce/env
```

直接将压缩包解压到系统的各个目录中：

```sh
$ sudo tar -C / -xzf cri-containerd-cni-1.4.3-linux-amd64.tar.gz
```

将 `/usr/local/bin` 和 `/usr/local/sbin` 追加到 `~/.bashrc` 文件的 `$PATH` 环境变量中：

```sh
export PATH=$PATH:/usr/local/bin:/usr/local/sbin
```

立即生效：

```sh
source ~/.bashrc
```

查看版本：

```sh
$ ctr version
Client:
  Version:  v1.4.3
  Revision: 269548fa27e0089a8b8278fc4fc781d7f65a939b
  Go version: go1.15.5

Server:
  Version:  v1.4.3
  Revision: 269548fa27e0089a8b8278fc4fc781d7f65a939b
  UUID: d1724999-91b3-4338-9288-9a54c9d52f70
```



#### 生成配置文件

Containerd 的默认配置文件为  `/etc/containerd/config.toml`，我们可以通过命令来生成一个默认的配置：

```sh
$ mkdir /etc/containerd
$ containerd config default > /etc/containerd/config.toml
```



#### 镜像加速

由于某些不可描述的因素，在国内拉取公共镜像仓库的速度是极慢的，为了节约拉取时间，需要为 Containerd 配置镜像仓库的 `mirror`。Containerd 的镜像仓库 mirror 与 Docker 相比有两个区别：

- Containerd 只支持通过 `CRI` 拉取镜像的 mirror，也就是说，只有通过 `crictl` 或者 Kubernetes 调用时 mirror 才会生效，通过 `ctr` 拉取是不会生效的。
- `Docker` 只支持为 `Docker Hub` 配置 mirror，而 `Containerd` 支持为任意镜像仓库配置 mirror。

配置镜像加速之前，先来看下 Containerd 的配置结构，乍一看可能会觉得很复杂，复杂就复杂在 plugin 的配置部分：

```go
[plugins]
  [plugins."io.containerd.gc.v1.scheduler"]
    pause_threshold = 0.02
    deletion_threshold = 0
    mutation_threshold = 100
    schedule_delay = "0s"
    startup_delay = "100ms"
  [plugins."io.containerd.grpc.v1.cri"]
    disable_tcp_service = true
    stream_server_address = "127.0.0.1"
    stream_server_port = "0"
    stream_idle_timeout = "4h0m0s"
    enable_selinux = false
    sandbox_image = "k8s.gcr.io/pause:3.1"
    stats_collect_period = 10
    systemd_cgroup = false
    enable_tls_streaming = false
    max_container_log_line_size = 16384
    disable_cgroup = false
    disable_apparmor = false
    restrict_oom_score_adj = false
    max_concurrent_downloads = 3
    disable_proc_mount = false
    [plugins."io.containerd.grpc.v1.cri".containerd]
      snapshotter = "overlayfs"
      default_runtime_name = "runc"
      no_pivot = false
      [plugins."io.containerd.grpc.v1.cri".containerd.default_runtime]
        runtime_type = ""
        runtime_engine = ""
        runtime_root = ""
        privileged_without_host_devices = false
      [plugins."io.containerd.grpc.v1.cri".containerd.untrusted_workload_runtime]
        runtime_type = ""
        runtime_engine = ""
        runtime_root = ""
        privileged_without_host_devices = false
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
          runtime_type = "io.containerd.runc.v1"
          runtime_engine = ""
          runtime_root = ""
          privileged_without_host_devices = false
    [plugins."io.containerd.grpc.v1.cri".cni]
      bin_dir = "/opt/cni/bin"
      conf_dir = "/etc/cni/net.d"
      max_conf_num = 1
      conf_template = ""
    [plugins."io.containerd.grpc.v1.cri".registry]
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
          endpoint = ["https://registry-1.docker.io"]
    [plugins."io.containerd.grpc.v1.cri".x509_key_pair_streaming]
      tls_cert_file = ""
      tls_key_file = ""
  [plugins."io.containerd.internal.v1.opt"]
    path = "/opt/containerd"
  [plugins."io.containerd.internal.v1.restart"]
    interval = "10s"
  [plugins."io.containerd.metadata.v1.bolt"]
    content_sharing_policy = "shared"
  [plugins."io.containerd.monitor.v1.cgroups"]
    no_prometheus = false
  [plugins."io.containerd.runtime.v1.linux"]
    shim = "containerd-shim"
    runtime = "runc"
    runtime_root = ""
    no_shim = false
    shim_debug = false
  [plugins."io.containerd.runtime.v2.task"]
    platforms = ["linux/amd64"]
  [plugins."io.containerd.service.v1.diff-service"]
    default = ["walking"]
  [plugins."io.containerd.snapshotter.v1.devmapper"]
    root_path = ""
    pool_name = ""
    base_image_size = ""
```

每一个顶级配置块的命名都是 `plugins."io.containerd.xxx.vx.xxx"` 这种形式，其实每一个顶级配置块都代表一个插件，其中 `io.containerd.xxx.vx` 表示插件的类型，vx 后面的 xxx 表示插件的 `ID`。可以通过 `ctr` 一览无余：

```sh
$ ctr plugin ls
TYPE                            ID                    PLATFORMS      STATUS
io.containerd.content.v1        content               -              ok
io.containerd.snapshotter.v1    btrfs                 linux/amd64    error
io.containerd.snapshotter.v1    devmapper             linux/amd64    error
io.containerd.snapshotter.v1    aufs                  linux/amd64    ok
io.containerd.snapshotter.v1    native                linux/amd64    ok
io.containerd.snapshotter.v1    overlayfs             linux/amd64    ok
io.containerd.snapshotter.v1    zfs                   linux/amd64    error
io.containerd.metadata.v1       bolt                  -              ok
io.containerd.differ.v1         walking               linux/amd64    ok
io.containerd.gc.v1             scheduler             -              ok
io.containerd.service.v1        containers-service    -              ok
io.containerd.service.v1        content-service       -              ok
io.containerd.service.v1        diff-service          -              ok
io.containerd.service.v1        images-service        -              ok
io.containerd.service.v1        leases-service        -              ok
io.containerd.service.v1        namespaces-service    -              ok
io.containerd.service.v1        snapshots-service     -              ok
io.containerd.runtime.v1        linux                 linux/amd64    ok
io.containerd.runtime.v2        task                  linux/amd64    ok
io.containerd.monitor.v1        cgroups               linux/amd64    ok
io.containerd.service.v1        tasks-service         -              ok
io.containerd.internal.v1       restart               -              ok
io.containerd.grpc.v1           containers            -              ok
io.containerd.grpc.v1           content               -              ok
io.containerd.grpc.v1           diff                  -              ok
io.containerd.grpc.v1           events                -              ok
io.containerd.grpc.v1           healthcheck           -              ok
io.containerd.grpc.v1           images                -              ok
io.containerd.grpc.v1           leases                -              ok
io.containerd.grpc.v1           namespaces            -              ok
io.containerd.internal.v1       opt                   -              ok
io.containerd.grpc.v1           snapshots             -              ok
io.containerd.grpc.v1           tasks                 -              ok
io.containerd.grpc.v1           version               -              ok
io.containerd.grpc.v1           cri                   linux/amd64    ok
```

顶级配置块下面的子配置块表示该插件的各种配置，比如 cri 插件下面就分为 `containerd`、`cni` 和 `registry` 的配置，而 containerd 下面又可以配置各种 runtime，还可以配置默认的 runtime。

镜像加速的配置就在 cri 插件配置块下面的 registry 配置块，所以需要修改的部分如下：

```json
    [plugins."io.containerd.grpc.v1.cri".registry]
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
          endpoint = ["https://dockerhub.mirrors.nwafu.edu.cn"]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."k8s.gcr.io"]
          endpoint = ["https://registry.aliyuncs.com/k8sxio"]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."gcr.io"]
          endpoint = ["xxx"]
```

- **registry.mirrors."xxx"** : 表示需要配置 mirror 的镜像仓库。例如，`registry.mirrors."docker.io"` 表示配置 docker.io 的 mirror。
- **endpoint** : 表示提供 mirror 的镜像加速服务。例如，这里推荐使用西北农林科技大学提供的镜像加速服务作为 `docker.io` 的 mirror。

**至于 `gcr.io`，目前还没有公共的加速服务。我自己掏钱搭了个加速服务，拉取速度大概是 `3M/s` 左右。**



#### 存储配置

Containerd 有两个不同的存储路径，一个用来保存持久化数据，一个用来保存运行时状态。

```sh
root = "/var/lib/containerd"
state = "/run/containerd"
```

`root`用来保存持久化数据，包括 `Snapshots`, `Content`, `Metadata` 以及各种插件的数据。每一个插件都有自己单独的目录，Containerd 本身不存储任何数据，它的所有功能都来自于已加载的插件，真是太机智了。

```sh
$ tree -L 2 /var/lib/containerd/
/var/lib/containerd/
├── io.containerd.content.v1.content
│   ├── blobs
│   └── ingest
├── io.containerd.grpc.v1.cri
│   ├── containers
│   └── sandboxes
├── io.containerd.metadata.v1.bolt
│   └── meta.db
├── io.containerd.runtime.v1.linux
│   └── k8s.io
├── io.containerd.runtime.v2.task
├── io.containerd.snapshotter.v1.aufs
│   └── snapshots
├── io.containerd.snapshotter.v1.btrfs
├── io.containerd.snapshotter.v1.native
│   └── snapshots
├── io.containerd.snapshotter.v1.overlayfs
│   ├── metadata.db
│   └── snapshots
└── tmpmounts

18 directories, 2 files
```

`state` 用来保存临时数据，包括 sockets、pid、挂载点、运行时状态以及不需要持久化保存的插件数据。

```sh
$ tree -L 2 /run/containerd/
/run/containerd/
├── containerd.sock
├── containerd.sock.ttrpc
├── io.containerd.grpc.v1.cri
│   ├── containers
│   └── sandboxes
├── io.containerd.runtime.v1.linux
│   └── k8s.io
├── io.containerd.runtime.v2.task
└── runc
    └── k8s.io

8 directories, 2 files
```



#### OOM

还有一项配置需要留意：

```
oom_score = 0
```

Containerd 是容器的守护者，一旦发生内存不足的情况，理想的情况应该是先杀死容器，而不是杀死 Containerd。所以需要调整 Containerd 的 `OOM` 权重，减少其被 **OOM Kill** 的几率。最好是将 `oom_score` 的值调整为比其他守护进程略低的值。这里的 oom_socre 其实对应的是 `/proc/<pid>/oom_socre_adj`，在早期的 Linux 内核版本里使用 `oom_adj` 来调整权重, 后来改用 `oom_socre_adj` 了。该文件描述如下：

> The value of `/proc/<pid>/oom_score_adj` is added to the badness score before it is used to determine which task to kill.  Acceptable values range from -1000 (OOM_SCORE_ADJ_MIN) to +1000 (OOM_SCORE_ADJ_MAX).  This allows userspace to polarize the preference for oom killing either by always preferring a certain task or completely disabling it.  The lowest possible value, -1000, is equivalent to disabling oom killing entirely for that task since it will always report a badness score of 0.

在计算最终的 `badness score` 时，会在计算结果是中加上 `oom_score_adj` ,这样用户就可以通过该在值来保护某个进程不被杀死或者每次都杀某个进程。其取值范围为 `-1000` 到 `1000`。

如果将该值设置为 `-1000`，则进程永远不会被杀死，因为此时 `badness score` 永远返回0。

建议 Containerd 将该值设置为 `-999` 到 `0` 之间。如果作为 Kubernetes 的 Worker 节点，可以考虑设置为 `-999`。



#### Systemd 配置

建议通过 systemd 配置 Containerd 作为守护进程运行，配置文件在上文已经被解压出来了：

```sh
$ cat /etc/systemd/system/containerd.service
# Copyright The containerd Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target

[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd

Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity
LimitNOFILE=1048576
# Comment TasksMax if your systemd version does not supports it.
# Only systemd 226 and above support this version.
TasksMax=infinity
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
```

这里有两个重要的参数：

- **Delegate** : 这个选项允许 Containerd 以及运行时自己管理自己创建的容器的 `cgroups`。如果不设置这个选项，systemd 就会将进程移到自己的 `cgroups` 中，从而导致 Containerd 无法正确获取容器的资源使用情况。

- **KillMode** : 这个选项用来处理 Containerd 进程被杀死的方式。默认情况下，systemd 会在进程的 cgroup 中查找并杀死 Containerd 的所有子进程，这肯定不是我们想要的。`KillMode`字段可以设置的值如下。

  我们需要将 KillMode 的值设置为 `process`，这样可以确保升级或重启 Containerd 时不杀死现有的容器。

- - **control-group**（默认值）：当前控制组里面的所有子进程，都会被杀掉
  - **process**：只杀主进程
  - **mixed**：主进程将收到 SIGTERM 信号，子进程收到 SIGKILL 信号
  - **none**：没有进程会被杀掉，只是执行服务的 stop 命令。

现在到了最关键的一步：启动 Containerd。执行一条命令就完事：

```sh
$ systemctl enable containerd --now
```

接下来进入本文最后一部分：Containerd 的基本使用方式。本文只会介绍 Containerd 的本地使用方法，即本地客户端 `ctr` 的使用方法，不会涉及到 `crictl`，后面有机会再介绍 `crictl`。



### **4. ctr 使用**

ctr 目前很多功能做的还没有 docker 那么完善，但基本功能已经具备了。下面将围绕**镜像**和**容器**这两个方面来介绍其使用方法。

#### 镜像

**镜像下载：**

```sh
$ ctr i pull docker.io/library/nginx:alpine
docker.io/library/nginx:alpine:                                                   resolved       |++++++++++++++++++++++++++++++++++++++|
index-sha256:efc93af57bd255ffbfb12c89ec0714dd1a55f16290eb26080e3d1e7e82b3ea66:    done           |++++++++++++++++++++++++++++++++++++++|
manifest-sha256:6ceeeab513f7d15cea38c1f8dfe5455323b5a1bfd568516b3b0ee70406f75247: done           |++++++++++++++++++++++++++++++++++++++|
config-sha256:0fde4fb87e476fd1655b3f04f55aa5b4b3ef7de7c701eb46573bb5a5dcf66fd2:   done           |++++++++++++++++++++++++++++++++++++++|
layer-sha256:abaddf4965e5e9ce9953f2e136b3bf9cc15365adbcf0c68b108b1cc26c12b1be:    done           |++++++++++++++++++++++++++++++++++++++|
layer-sha256:05e7bc50f07f000e9993ec0d264b9ffcbb9a01a4d69c68f556d25e9811a8f7f4:    done           |++++++++++++++++++++++++++++++++++++++|
layer-sha256:c78f7f670e47cf98494e7dbe08e463d34c160bf6a5939a2155ff4438cb8b0e80:    done           |++++++++++++++++++++++++++++++++++++++|
layer-sha256:ce77cf6a2ede66c463dcdd39f1a43cfbac3723a99e94f697bc20faee0f7cce1b:    done           |++++++++++++++++++++++++++++++++++++++|
layer-sha256:3080fd9f46494247c9298a6a3d9694f03f6a32898a07ffbe1c17a0752bae5c4e:    done           |++++++++++++++++++++++++++++++++++++++|
elapsed: 17.3s                                                                    total:  8.7 Mi (513.8 KiB/s)
unpacking linux/amd64 sha256:efc93af57bd255ffbfb12c89ec0714dd1a55f16290eb26080e3d1e7e82b3ea66...
done
```

**本地镜像列表查询：**

```sh
$ ctr i ls
REF  TYPE  DIGEST   SIZE   PLATFORMS    LABELS
docker.io/library/nginx:alpine                                    application/vnd.docker.distribution.manifest.list.v2+json sha256:efc93af57bd255ffbfb12c89ec0714dd1a55f16290eb26080e3d1e7e82b3ea66 9.3 MiB   linux/386,linux/amd64,linux/arm/v6,linux/arm/v7,linux/arm64/v8,linux/ppc64le,linux/s390x -
```

这里需要注意PLATFORMS，它是镜像的能够运行的平台标识。



**将镜像挂载到主机目录：**

```sh
$ ctr i mount docker.io/library/nginx:alpine /mnt

$ tree -L 1 /mnt
/mnt
├── bin
├── dev
├── docker-entrypoint.d
├── docker-entrypoint.sh
├── etc
├── home
├── lib
├── media
├── mnt
├── opt
├── proc
├── root
├── run
├── sbin
├── srv
├── sys
├── tmp
├── usr
└── var

18 directories, 1 file
```

**将镜像从主机目录上卸载：**

```sh
ctr i unmount /mnt
```

**将镜像导出为压缩包：**

````sh
ctr i export nginx.tar.gz docker.io/library/nginx:alpine
````

**从压缩包导入镜像：**

```sh
 ctr i import nginx.tar.gz
```

**其他操作可以自己查看帮助：**

```sh
ctr i --help
NAME:
   ctr images - manage images

USAGE:
   ctr images command [command options] [arguments...]

COMMANDS:
   check       check that an image has all content available locally
   export      export images
   import      import images
   list, ls    list images known to containerd
   mount       mount an image to a target path
   unmount     unmount the image from the target
   pull        pull an image from a remote
   push        push an image to a remote
   remove, rm  remove one or more images by reference
   tag         tag an image
   label       set and clear labels for an image

OPTIONS:
   --help, -h  show help
```

对镜像的更高级操作可以使用子命令 `content`，例如在线编辑镜像的 `blob` 并生成一个新的 `digest`：

```sh
$ ctr content ls
DIGEST         SIZE AGE  LABELS
...
...
sha256:fdd7fff110870339d34cf071ee90fbbe12bdbf3d1d9a14156995dfbdeccd7923 740B 7 days  containerd.io/gc.ref.content.2=sha256:4e537e26e21bf61836f827e773e6e6c3006e3c01c6d59f4b058b09c2753bb929,containerd.io/gc.ref.content.1=sha256:188c0c94c7c576fff0792aca7ec73d67a2f7f4cb3a6e53a84559337260b36964,containerd.io/gc.ref.content.0=sha256:b7199797448c613354489644be1f60aa2d8e9c2278989100c72ede3001334f7b,containerd.io/distribution.source.ghcr.fuckcloudnative.io=yangchuansheng/grafana-backup-tool
```



#### 容器

创建容器：

```sh
$ ctr c create docker.io/library/nginx:alpine nginx

$ ctr c ls
CONTAINER    IMAGE                             RUNTIME
nginx        docker.io/library/nginx:alpine    io.containerd.runc.v2
```

查看容器的详细配置：

```sh
# 和 docker inspect 类似
$ ctr c info nginx
```



#### 任务

上面 `create` 的命令创建了容器后，并没有处于运行状态，只是一个静态的容器。一个 container 对象只是包含了运行一个容器所需的资源及配置的数据结构，这意味着 namespaces、rootfs 和容器的配置都已经初始化成功了，只是用户进程(这里是 `nginx`)还没有启动。

然而一个容器真正的运行起来是由 task 对象实现的，`task` 代表任务的意思，可以为容器设置网卡，还可以配置工具来对容器进行监控等。

所以还需要通过 task 启动容器：

```sh
$ ctr task start -d nginx

$ ctr task ls
TASK     PID       STATUS
nginx    131405    RUNNING
```

当然，也可以一步到位直接创建并运行容器：

```sh
$ ctr run -d docker.io/library/nginx:alpine nginx
```

进入容器：

```sh
# 和 docker 的操作类似，但必须要指定 --exec-id，这个 id 可以随便写，只要唯一就行
$ ctr task exec --exec-id 0 -t nginx sh
```

暂停容器：

```sh
# 和 docker pause 类似
$ ctr task pause nginx
```

容器状态变成了 PAUSED：

```sh
$ ctr task ls
TASK     PID       STATUS
nginx    149857    PAUSED
```

恢复容器：

```sh
$ ctr task resume nginx
```

**ctr 没有 stop 容器的功能，只能暂停或者杀死容器。**

杀死容器：

```sh
ctr task kill nginx
```

获取容器的 cgroup 信息：

```sh
# 这个命令用来获取容器的内存、CPU 和 PID 的限额与使用量。
$ ctr task metrics nginx
ID       TIMESTAMP
nginx    2020-12-15 09:15:13.943447167 +0000 UTC

METRIC                   VALUE
memory.usage_in_bytes    77131776
memory.limit_in_bytes    9223372036854771712
memory.stat.cache        6717440
cpuacct.usage            194187935
cpuacct.usage_percpu     [0 335160 0 5395642 3547200 58559242 0 0 0 0 0 0 6534104 5427871 3032481 2158941 8513633 4620692 8261063 3885961 3667830 0 4367411 356280 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 1585841 0 7754942 5818102 21430929 0 0 0 0 0 0 1811840 2241260 2673960 6041161 8210604 2991221 10073713 1111020 3139751 0 640080 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0]
pids.current             97
pids.limit               0
```

查看容器中所有进程的 `PID`：

```sh
$ ctr task ps nginx
PID       INFO
149857    -
149921    -
149922    -
149923    -
149924    -
149925    -
149926    -
149928    -
149929    -
149930    -
149932    -
149933    -
149934    -
```

注意：这里的 PID 是宿主机看到的 PID，不是容器中看到的 PID。



#### 命名空间

除了 k8s 有命名空间以外，Containerd 也支持命名空间。

```
$ shctr ns ls
NAME    LABELS
default
```

如果不指定，`ctr` 默认是 `default` 空间。

目前 Containerd 的定位还是解决运行时，所以目前他还不能完全替代 `dockerd`，例如使用 `Dockerfile` 来构建镜像。其实这不是什么大问题，我再给大家介绍一个大招：**Containerd 和 Docker 一起用！**



#### Containerd + Docker

事实上，Docker 和 Containerd 是可以同时使用的，只不过 Docker 默认使用的 Containerd 的命名空间不是 default，而是 `moby`。下面就是见证奇迹的时刻。

首先从其他装了 Docker 的机器或者 GitHub 上下载 Docker 相关的二进制文件，然后使用下面的命令启动 Docker：

```sh
$ dockerd --containerd /run/containerd/containerd.sock --cri-containerd
```

接着用 Docker 运行一个容器：

```sh
$ docker run -d --name nginx nginx:alpine
```

现在再回过头来查看 Containerd 的命名空间：

```sh
$ ctr ns ls
NAME    LABELS
default
moby
```

查看该命名空间下是否有容器：

```sh
ctr -n moby c ls
CONTAINER                                                           IMAGE    RUNTIME
b7093d7aaf8e1ae161c8c8ffd4499c14ba635d8e174cd03711f4f8c27818e89a    -        io.containerd.runtime.v1.linux
```

最后提醒一句：Kubernetes 用户不用惊慌，Kubernetes 默认使用的是 Containerd 的 `k8s.io` 命名空间，所以 `ctr -n k8s.io` 就能看到 Kubernetes 创建的所有容器啦，也不用担心 `crictl` 不支持 load 镜像了，因为 `ctr -n k8s.io` 可以 load 镜像





##  Kubernetes集群管理工具kubectl



###  概述

kubectl是Kubernetes集群的命令行工具，通过kubectl能够对集群本身进行管理，并能够在集群上进行容器化应用的安装和部署



### 命令格式

命令格式如下

```
kubectl [command] [type] [name] [flags]
```

参数

- command：指定要对资源执行的操作，例如create、get、describe、delete
- type：指定资源类型，资源类型是大小写敏感的，开发者能够以单数 、复数 和 缩略的形式

例如：

```
kubectl get pod pod1
kubectl get pods pod1
kubectl get po pod1
```

![image-20201114095544185](https://gitee.com/moxi159753/LearningNotes/raw/master/K8S/6_Kubernetes%E9%9B%86%E7%BE%A4%E7%AE%A1%E7%90%86%E5%B7%A5%E5%85%B7kubectl/images/image-20201114095544185.png)

- name：指定资源的名称，名称也是大小写敏感的，如果省略名称，则会显示所有的资源，例如

```
kubectl get pods
```

- flags：指定可选的参数，例如，可用 -s 或者 -server参数指定Kubernetes API server的地址和端口



### 常见命令

#### kubectl help 获取更多信息

通过 help命令，能够获取帮助信息

```
# 获取kubectl的命令
kubectl --help

# 获取某个命令的介绍和使用
kubectl get --help
```



### 基础命令

常见的基础命令

| 命令    | 介绍                                           |
| ------- | ---------------------------------------------- |
| create  | 通过文件名或标准输入创建资源                   |
| expose  | 将一个资源公开为一个新的Service                |
| run     | 在集群中运行一个特定的镜像                     |
| set     | 在对象上设置特定的功能                         |
| get     | 显示一个或多个资源                             |
| explain | 文档参考资料                                   |
| edit    | 使用默认的编辑器编辑一个资源                   |
| delete  | 通过文件名，标准输入，资源名称或标签来删除资源 |



### 部署命令

| 命令           | 介绍                                               |
| -------------- | -------------------------------------------------- |
| rollout        | 管理资源的发布                                     |
| rolling-update | 对给定的复制控制器滚动更新                         |
| scale          | 扩容或缩容Pod数量，Deployment、ReplicaSet、RC或Job |
| autoscale      | 创建一个自动选择扩容或缩容并设置Pod数量            |



### 集群管理命令

| 命令         | 介绍                           |
| ------------ | ------------------------------ |
| certificate  | 修改证书资源                   |
| cluster-info | 显示集群信息                   |
| top          | 显示资源(CPU/M)                |
| cordon       | 标记节点不可调度               |
| uncordon     | 标记节点可被调度               |
| drain        | 驱逐节点上的应用，准备下线维护 |
| taint        | 修改节点taint标记              |
|              |                                |



### 故障和调试命令

| 命令         | 介绍                                                         |
| ------------ | ------------------------------------------------------------ |
| describe     | 显示特定资源或资源组的详细信息                               |
| logs         | 在一个Pod中打印一个容器日志，如果Pod只有一个容器，容器名称是可选的 |
| attach       | 附加到一个运行的容器                                         |
| exec         | 执行命令到容器                                               |
| port-forward | 转发一个或多个                                               |
| proxy        | 运行一个proxy到Kubernetes API Server                         |
| cp           | 拷贝文件或目录到容器中                                       |
| auth         | 检查授权                                                     |



### 其它命令

| 命令         | 介绍                                                |
| ------------ | --------------------------------------------------- |
| apply        | 通过文件名或标准输入对资源应用配置                  |
| patch        | 使用补丁修改、更新资源的字段                        |
| replace      | 通过文件名或标准输入替换一个资源                    |
| convert      | 不同的API版本之间转换配置文件                       |
| label        | 更新资源上的标签                                    |
| annotate     | 更新资源上的注释                                    |
| completion   | 用于实现kubectl工具自动补全                         |
| api-versions | 打印受支持的API版本                                 |
| config       | 修改kubeconfig文件（用于访问API，比如配置认证信息） |
| help         | 所有命令帮助                                        |
| plugin       | 运行一个命令行插件                                  |
| version      | 打印客户端和服务版本信息                            |



### 目前使用的命令

```sh
# 创建一个nginx镜像
kubectl create deployment nginx --image=nginx

# 对外暴露端口
kubectl expose deployment nginx --port=80 --type=NodePort

# 查看资源
kubectl get pod, svc
```





## Kubernetes集群YAML文件详解



### 概述

k8s 集群中对资源管理和资源对象编排部署都可以通过声明样式（YAML）文件来解决，也就是可以把需要对资源对象操作编辑到YAML 格式文件中，我们把这种文件叫做资源清单文件，通过kubectl 命令直接使用资源清单文件就可以实现对大量的资源对象进行编排部署了。一般在我们开发的时候，都是通过配置YAML文件来部署集群的。

YAML文件：就是资源清单文件，用于资源编排



### YAML文件介绍

#### YAML概述

YAML ：仍是一种标记语言。为了强调这种语言以数据做为中心，而不是以标记语言为重点。

YAML 是一个可读性高，用来表达数据序列的格式。



#### YAML 基本语法

- 使用空格做为缩进
- 缩进的空格数目不重要，只要相同层级的元素左侧对齐即可
- 低版本缩进时不允许使用Tab 键，只允许使用空格
- 使用#标识注释，从这个字符一直到行尾，都会被解释器忽略
- 使用 --- 表示新的yaml文件开始



#### YAML 支持的数据结构

##### 对象

键值对的集合，又称为映射(mapping) / 哈希（hashes） / 字典（dictionary）

```
# 对象类型：对象的一组键值对，使用冒号结构表示
name: Tom
age: 18

# yaml 也允许另一种写法，将所有键值对写成一个行内对象
hash: {name: Tom, age: 18}
```



##### 数组

```
# 数组类型：一组连词线开头的行，构成一个数组
People
- Tom
- Jack

# 数组也可以采用行内表示法
People: [Tom, Jack]
```

###  

### YAML文件组成部分

主要分为了两部分，一个是控制器的定义 和 被控制的对象

#### 控制器的定义

![image-20201114110444032](https://gitee.com/moxi159753/LearningNotes/raw/master/K8S/7_Kubernetes%E9%9B%86%E7%BE%A4YAML%E6%96%87%E4%BB%B6%E8%AF%A6%E8%A7%A3/images/image-20201114110444032.png)



#### 被控制的对象

包含一些 镜像，版本、端口等

![image-20201114110600165](https://gitee.com/moxi159753/LearningNotes/raw/master/K8S/7_Kubernetes%E9%9B%86%E7%BE%A4YAML%E6%96%87%E4%BB%B6%E8%AF%A6%E8%A7%A3/images/image-20201114110600165.png)



### 属性说明

在一个YAML文件的控制器定义中，有很多属性名称

| 属性名称   | 介绍       |
| ---------- | ---------- |
| apiVersion | API版本    |
| kind       | 资源类型   |
| metadata   | 资源元数据 |
| spec       | 资源规格   |
| replicas   | 副本数量   |
| selector   | 标签选择器 |
| template   | Pod模板    |
| metadata   | Pod元数据  |
| spec       | Pod规格    |
| containers | 容器配置   |



### 如何快速编写YAML文件

一般来说，我们很少自己手写YAML文件，因为这里面涉及到了很多内容，我们一般都会借助工具来创建



#### 使用kubectl create命令

这种方式一般用于资源没有部署的时候，我们可以直接创建一个YAML配置文件

```
# 尝试运行,并不会真正的创建镜像
kubectl create deployment web --image=nginx -o yaml --dry-run
```

或者我们可以输出到一个文件中

```
kubectl create deployment web --image=nginx -o yaml --dry-run > hello.yaml
```

然后我们就在文件中直接修改即可



#### 使用kubectl get命令导出yaml文件

可以首先查看一个目前已经部署的镜像

```
kubectl get deploy
```

![image-20201114113115649](https://gitee.com/moxi159753/LearningNotes/raw/master/K8S/7_Kubernetes%E9%9B%86%E7%BE%A4YAML%E6%96%87%E4%BB%B6%E8%AF%A6%E8%A7%A3/images/image-20201114113115649.png)

然后我们导出 nginx的配置

```
kubectl get deploy nginx -o=yaml --export > nginx.yaml
```

然后会生成一个 `nginx.yaml` 的配置文件

![image-20201114184538797](https://gitee.com/moxi159753/LearningNotes/raw/master/K8S/7_Kubernetes%E9%9B%86%E7%BE%A4YAML%E6%96%87%E4%BB%B6%E8%AF%A6%E8%A7%A3/images/image-20201114184538797.png)





## Kubernetes集群安全机制



### 概述

当我们访问K8S集群时，需要经过三个步骤完成具体操作

- 认证
- 鉴权【授权】
- 准入控制

进行访问的时候，都需要经过 apiserver， apiserver做统一协调，比如门卫

- 访问过程中，需要证书、token、或者用户名和密码
- 如果访问pod需要serviceAccount

![image-20210107111815658](D:\学习资料\笔记\k8s\k8s图\image-20210107111815658.png)

​                                                 

### 认证

对外不暴露8080端口，只能内部访问，对外使用的端口6443

客户端身份认证常用方式

- https证书认证，基于ca证书
- http token认证，通过token来识别用户
- http基本认证，用户名 + 密码认证



### 鉴权

基于RBAC进行鉴权操作

基于角色访问控制



### 准入控制

就是准入控制器的列表，如果列表有请求内容就通过，没有的话 就拒绝



### RBAC介绍

基于角色的访问控制，为某个角色设置访问内容，然后用户分配该角色后，就拥有该角色的访问权限

![image-20210107111923582](D:\学习资料\笔记\k8s\k8s图\image-20210107111923582.png)

k8s中有默认的几个角色

- role：特定命名空间访问权限
- ClusterRole：所有命名空间的访问权限

角色绑定

- roleBinding：角色绑定到主体
- ClusterRoleBinding：集群角色绑定到主体

主体

- user：用户
- group：用户组
- serviceAccount：服务账号



### RBAC实现鉴权

- 创建命名空间



### 创建命名空间

我们可以首先查看已经存在的命名空间

```bash
kubectl get namespace
```

然后我们创建一个自己的命名空间  roledemo

```bash
kubectl create ns roledemo
```



### 命名空间创建Pod

为什么要创建命名空间？因为如果不创建命名空间的话，默认是在default下

```bash
kubectl run nginx --image=nginx -n roledemo
```



### 创建角色

我们通过 rbac-role.yaml进行创建

![image-20210107112110476](D:\学习资料\笔记\k8s\k8s图\image-20210107112110476.png)

tip：这个角色只对pod 有 get、list权限

然后通过 yaml创建我们的role

```bash
# 创建
kubectl apply -f rbac-role.yaml
# 查看
kubectl get role -n roledemo
```



### 创建角色绑定

我们还是通过 role-rolebinding.yaml 的方式，来创建我们的角色绑定

![image-20210107112221734](D:\学习资料\笔记\k8s\k8s图\image-20210107112221734.png)

然后创建我们的角色绑定

```bash
# 创建角色绑定
kubectl apply -f rbac-rolebinding.yaml
# 查看角色绑定
kubectl get role, rolebinding -n roledemo
```



### 使用证书识别身份

我们首先得有一个 rbac-user.sh 证书脚本

![image-20201118095541427](D:\学习资料\笔记\k8s\k8s图\image-20201118095541427.png)

![image-20201118095627954](D:\学习资料\笔记\k8s\k8s图\image-20201118095627954.png)

这里包含了很多证书文件，在TSL目录下，需要复制过来

通过下面命令执行我们的脚本

```bash
./rbac-user.sh
```

最后我们进行测试

```bash
# 用get命令查看 pod 【有权限】
kubectl get pods -n roledemo
# 用get命令查看svc 【没权限】
kubectl get svc -n roledmeo
```





## Kubernetes核心技术Helm



Helm就是一个包管理工具【类似于npm】

![img](D:\学习资料\笔记\k8s\k8s图\892532-20180224212352306-705544441.png)



### 为什么引入Helm

首先在原来项目中都是基于yaml文件来进行部署发布的，而目前项目大部分微服务化或者模块化，会分成很多个组件来部署，每个组件可能对应一个deployment.yaml,一个service.yaml,一个Ingress.yaml还可能存在各种依赖关系，这样一个项目如果有5个组件，很可能就有15个不同的yaml文件，这些yaml分散存放，如果某天进行项目恢复的话，很难知道部署顺序，依赖关系等，而所有这些包括

- 基于yaml配置的集中存放
- 基于项目的打包
- 组件间的依赖

但是这种方式部署，会有什么问题呢？

- 如果使用之前部署单一应用，少数服务的应用，比较合适
- 但如果部署微服务项目，可能有几十个服务，每个服务都有一套yaml文件，需要维护大量的yaml文件，版本管理特别不方便

Helm的引入，就是为了解决这个问题

- 使用Helm可以把这些YAML文件作为整体管理
- 实现YAML文件高效复用
- 使用helm应用级别的版本管理



### Helm介绍

Helm是一个Kubernetes的包管理工具，就像Linux下的包管理器，如yum/apt等，可以很方便的将之前打包好的yaml文件部署到kubernetes上。

Helm有三个重要概念

- helm：一个命令行客户端工具，主要用于Kubernetes应用chart的创建、打包、发布和管理
- Chart：应用描述，一系列用于描述k8s资源相关文件的集合
- Release：基于Chart的部署实体，一个chart被Helm运行后将会生成对应的release，将在K8S中创建出真实的运行资源对象。也就是应用级别的版本管理
- Repository：用于发布和存储Chart的仓库



### Helm组件及架构

Helm采用客户端/服务端架构，有如下组件组成

- Helm CLI是Helm客户端，可以在本地执行
- Tiller是服务器端组件，在Kubernetes集群上运行，并管理Kubernetes应用程序
- Repository是Chart仓库，Helm客户端通过HTTP协议来访问仓库中Chart索引文件和压缩包

![image-20201119095458328](D:\学习资料\笔记\k8s\k8s图\image-20201119095458328.png)



### Helm v3变化

2019年11月13日，Helm团队发布了Helm v3的第一个稳定版本

该版本主要变化如下

- 架构变化

  - 最明显的变化是Tiller的删除
  - V3版本删除Tiller
  - relesase可以在不同命名空间重用

V3之前

![image-20201118171523403](https://gitee.com/moxi159753/LearningNotes/raw/master/K8S/15_Kubernetes%E6%A0%B8%E5%BF%83%E6%8A%80%E6%9C%AFHelm/images/image-20201118171523403.png)

 V3版本

![image-20201118171956054](D:\学习资料\笔记\k8s\k8s图\image-20201118171956054.png)



### helm配置

首先我们需要去 [官网下载](https://helm.sh/docs/intro/quickstart/)

- 第一步，[下载helm](https://github.com/helm/helm/releases)安装压缩文件，上传到linux系统中
- 第二步，解压helm压缩文件，把解压后的helm目录复制到 usr/bin 目录中
- 使用命令：helm

我们都知道yum需要配置yum源，那么helm就就要配置helm源



### helm仓库

添加仓库

```bash
helm repo add 仓库名  仓库地址 
```

例如

```bash
# 配置微软源
helm repo add stable http://mirror.azure.cn/kubernetes/charts
# 配置阿里源
helm repo add aliyun https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
# 配置google源
helm repo add google https://kubernetes-charts.storage.googleapis.com/

# 更新
helm repo update
```

然后可以查看我们添加的仓库地址

```bash
# 查看全部
helm repo list
# 查看某个
helm search repo stable
```

![image-20201118195732281](D:\学习资料\笔记\k8s\k8s图\image-20201118195732281.png)

或者可以删除我们添加的源

```bash
helm repo remove stable
```



#### helm基本命令

- chart install
- chart upgrade
- chart rollback



### 使用helm快速部署应用

#### 使用命令搜索应用

首先我们使用命令，搜索我们需要安装的应用

```bash
# 搜索 weave仓库
helm search repo weave
```

![image-20201118200603643](D:\学习资料\笔记\k8s\k8s图\image-20201118200603643.png)



#### 根据搜索内容选择安装

搜索完成后，使用命令进行安装

```bash
helm install ui aliyun/weave-scope
```

可以通过下面命令，来下载yaml文件

```bash
kubectl apply -f weave-scope.yaml
```

安装完成后，通过下面命令即可查看

```bash
helm list
```

![image-20201118203727585](D:\学习资料\笔记\k8s\k8s图\image-20201118203727585.png)

同时可以通过下面命令，查看更新具体的信息

```bash
helm status ui
```

但是我们通过查看 svc状态，发现没有对象暴露端口

![image-20201118205031343](D:\学习资料\笔记\k8s\k8s图\image-20201118205031343.png)

所以我们需要修改service的yaml文件，添加NodePort

```bash
kubectl edit svc ui-weave-scope
```

![image-20201118205129431](D:\学习资料\笔记\k8s\k8s图\image-20201118205129431.png)

这样就可以对外暴露端口了

![image-20201118205147631](D:\学习资料\笔记\k8s\k8s图\image-20201118205147631.png)

然后我们通过 ip + 32185 即可访问



### 如果自己创建Chart

使用命令，自己创建Chart

```bash
helm create mychart
```

创建完成后，我们就能看到在当前文件夹下，创建了一个 mychart目录

![image-20201118210755621](D:\学习资料\笔记\k8s\k8s图\image-20201118210755621.png)

#### 目录格式

- templates：编写yaml文件存放到这个目录
- values.yaml：存放的是全局的yaml文件
- chart.yaml：当前chart属性配置信息



#### 在templates文件夹创建两个文件

我们创建以下两个

- deployment.yaml
- service.yaml

我们可以通过下面命令创建出yaml文件

```bash
# 导出deployment.yaml
kubectl create deployment web1 --image=nginx --dry-run -o yaml > deployment.yaml

# 导出service.yaml 【可能需要创建 deployment，不然会报错】
kubectl expose deployment web1 --port=80 --target-port=80 --type=NodePort --dry-run -o yaml > service.yaml
```



#### 安装mychart

执行命令创建

```bash
helm install web1 mychart
```

![image-20201118213120916](D:\学习资料\笔记\k8s\k8s图\image-20201118213120916.png)



#### 应用升级

当我们修改了mychart中的东西后，就可以进行升级操作

```bash
helm upgrade web1 mychart
```



#### chart模板使用

通过传递参数，动态渲染模板，yaml内容动态从传入参数生成

![image-20201118213630083](D:\学习资料\笔记\k8s\k8s图\image-20201118213630083.png)

刚刚我们创建mychart的时候，看到有values.yaml文件，这个文件就是一些全局的变量，然后在templates中能取到变量的值，下面我们可以利用这个，来完成动态模板

- 在values.yaml定义变量和值
- 具体yaml文件，获取定义变量值
- yaml文件中大题有几个地方不同
  - image
  - tag
  - label
  - port
  - replicas



#### 定义变量和值

在values.yaml定义变量和值

![image-20201118214050899](D:\学习资料\笔记\k8s\k8s图\image-20201118214050899.png)



#### 获取变量和值

我们通过表达式形式 使用全局变量  `{{.Values.变量名称}} `

例如： `{{.Release.Name}}`

![image-20201118214413203](D:\学习资料\笔记\k8s\k8s图\image-20201118214413203.png)



#### 安装应用

在我们修改完上述的信息后，就可以尝试的创建应用了

```bash
helm install --dry-run web2 mychart
```

![image-20201118214727058](D:\学习资料\笔记\k8s\k8s图\image-20201118214727058.png)







##  Kubernetes搭建高可用集群



### 前言

之前我们搭建的集群，只有一个master节点，当master节点宕机的时候，通过node将无法继续访问，而master主要是管理作用，所以整个集群将无法提供服务

![image-20201121164522945](D:\学习资料\笔记\k8s\k8s图\image-20201121164522945.png)



### 高可用集群

下面我们就需要搭建一个多master节点的高可用集群，不会存在单点故障问题

但是在node 和 master节点之间，需要存在一个 LoadBalancer组件，作用如下：

- 负载
- 检查master节点的状态

![image-20201121164931760](D:\学习资料\笔记\k8s\k8s图\image-20201121164931760.png)

对外有一个统一的VIP：虚拟ip来对外进行访问



### 高可用集群技术细节

高可用集群技术细节如下所示：

![image-20201121165325194](D:\学习资料\笔记\k8s\k8s图\image-20201121165325194.png)

- keepalived：配置虚拟ip，检查节点的状态
- haproxy：负载均衡服务【类似于nginx】
- apiserver：
- controller：
- manager：
- scheduler：



### 高可用集群步骤

我们采用2个master节点，一个node节点来搭建高可用集群，下面给出了每个节点需要做的事情

![image-20201121170351461](D:\学习资料\笔记\k8s\k8s图\image-20201121170351461.png)



### 初始化操作

我们需要在这三个节点上进行操作

```sh 
# 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

# 关闭selinux
# 永久关闭
sed -i 's/enforcing/disabled/' /etc/selinux/config  
# 临时关闭
setenforce 0  

# 关闭swap
# 临时
swapoff -a 
# 永久关闭
sed -ri 's/.*swap.*/#&/' /etc/fstab

# 根据规划设置主机名【master1节点上操作】
hostnamectl set-hostname master1
# 根据规划设置主机名【master2节点上操作】
hostnamectl set-hostname master2
# 根据规划设置主机名【node1节点操作】
hostnamectl set-hostname node1

# 添加hosts
cat >> /etc/hosts << EOF
192.168.44.158  k8smaster
192.168.44.155 master01.k8s.io master1
192.168.44.156 master02.k8s.io master2
192.168.44.157 node01.k8s.io node1
EOF


# 将桥接的IPv4流量传递到iptables的链【3个节点上都执行】
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

# 生效
sysctl --system  

# 时间同步
yum install ntpdate -y
ntpdate time.windows.com
```



### 部署keepAlived

下面我们需要在所有的master节点【master1和master2】上部署keepAlive



#### 安装相关包

```sh
# 安装相关工具
yum install -y conntrack-tools libseccomp libtool-ltdl
# 安装keepalived
yum install -y keepalived
```



#### 配置master节点

添加master1的配置

```sh
cat > /etc/keepalived/keepalived.conf <<EOF 
! Configuration File for keepalived

global_defs {
   router_id k8s
}

vrrp_script check_haproxy {
    script "killall -0 haproxy"
    interval 3
    weight -2
    fall 10
    rise 2
}

vrrp_instance VI_1 {
    state MASTER 
    interface ens33 
    virtual_router_id 51
    priority 250
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass ceb1b3ec013d66163d6ab
    }
    virtual_ipaddress {
        192.168.44.158
    }
    track_script {
        check_haproxy
    }

}
EOF
```

添加master2的配置

```sh
cat > /etc/keepalived/keepalived.conf <<EOF 
! Configuration File for keepalived

global_defs {
   router_id k8s
}

vrrp_script check_haproxy {
    script "killall -0 haproxy"
    interval 3
    weight -2
    fall 10
    rise 2
}

vrrp_instance VI_1 {
    state BACKUP 
    interface ens33 
    virtual_router_id 51
    priority 200
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass ceb1b3ec013d66163d6ab
    }
    virtual_ipaddress {
        192.168.44.158
    }
    track_script {
        check_haproxy
    }

}
EOF
```



#### 启动和检查

在两台master节点都执行

```
# 启动keepalived
systemctl start keepalived.service
# 设置开机启动
systemctl enable keepalived.service
# 查看启动状态
systemctl status keepalived.service
```

启动后查看master的网卡信息

```
ip a s ens33
```

![image-20201121171619497](D:\学习资料\笔记\k8s\k8s图\image-20201121171619497.png)



### 部署haproxy

haproxy主要做负载的作用，将我们的请求分担到不同的node节点上

#### 安装

在两个master节点安装 haproxy

```
# 安装haproxy
yum install -y haproxy
# 启动 haproxy
systemctl start haproxy
# 开启自启
systemctl enable haproxy
```

启动后，我们查看对应的端口是否包含 16443

```
netstat -tunlp | grep haproxy
```

![image-20201121181803128](D:\学习资料\笔记\k8s\k8s图\image-20201121181803128.png)



#### 配置

两台master节点的配置均相同，配置中声明了后端代理的两个master节点服务器，指定了haproxy运行的端口为16443等，因此16443端口为集群的入口

```sh
cat > /etc/haproxy/haproxy.cfg << EOF
#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    # to have these messages end up in /var/log/haproxy.log you will
    # need to:
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog
    # 2) configure local2 events to go to the /var/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #
    #    local2.*                       /var/log/haproxy.log
    #
    log         127.0.0.1 local2
    
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon 
       
    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats
#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------  
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000
#---------------------------------------------------------------------
# kubernetes apiserver frontend which proxys to the backends
#--------------------------------------------------------------------- 
frontend kubernetes-apiserver
    mode                 tcp
    bind                 *:16443
    option               tcplog
    default_backend      kubernetes-apiserver    
#---------------------------------------------------------------------
# round robin balancing between the various backends
#---------------------------------------------------------------------
backend kubernetes-apiserver
    mode        tcp
    balance     roundrobin
    server      master01.k8s.io   192.168.44.155:6443 check
    server      master02.k8s.io   192.168.44.156:6443 check
#---------------------------------------------------------------------
# collection haproxy statistics message
#---------------------------------------------------------------------
listen stats
    bind                 *:1080
    stats auth           admin:awesomePassword
    stats refresh        5s
    stats realm          HAProxy\ Statistics
    stats uri            /admin?stats
EOF
```



### 安装Docker、Kubeadm、kubectl

所有节点安装Docker/kubeadm/kubelet ，Kubernetes默认CRI（容器运行时）为Docker，因此先安装Docker



#### 安装Docker

首先配置一下Docker的阿里yum源

```
cat >/etc/yum.repos.d/docker.repo<<EOF
[docker-ce-edge]
name=Docker CE Edge - \$basearch
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/\$basearch/edge
enabled=1
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg
EOF
```

然后yum方式安装docker

```sh
# yum安装
yum -y install docker-ce

# 查看docker版本
docker --version  

# 启动docker
systemctl enable docker
systemctl start docker
```

配置docker的镜像源

```sh
cat >> /etc/docker/daemon.json << EOF
{
  "registry-mirrors": ["https://b9pmyelo.mirror.aliyuncs.com"]
}
EOF
```

然后重启docker

```
systemctl restart docker
```



#### 添加kubernetes软件源

然后我们还需要配置一下yum的k8s软件源

```sh
cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```



#### 安装kubeadm，kubelet和kubectl

由于版本更新频繁，这里指定版本号部署：

```sh
# 安装kubelet、kubeadm、kubectl，同时指定版本
yum install -y kubelet-1.18.0 kubeadm-1.18.0 kubectl-1.18.0
# 设置开机启动
systemctl enable kubelet
```



### 部署Kubernetes Master【master节点】

#### 创建kubeadm配置文件

在具有vip的master上进行初始化操作，这里为master1

```sh
# 创建文件夹
mkdir /usr/local/kubernetes/manifests -p
# 到manifests目录
cd /usr/local/kubernetes/manifests/
# 新建yaml文件
vi kubeadm-config.yaml
```

yaml内容如下所示：

```yaml
apiServer:
  certSANs:
    - master1
    - master2
    - master.k8s.io
    - 192.168.44.158
    - 192.168.44.155
    - 192.168.44.156
    - 127.0.0.1
  extraArgs:
    authorization-mode: Node,RBAC
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta1
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controlPlaneEndpoint: "master.k8s.io:16443"
controllerManager: {}
dns: 
  type: CoreDNS
etcd:
  local:    
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: v1.16.3
networking: 
  dnsDomain: cluster.local  
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.1.0.0/16
scheduler: {}
```

然后我们在 master1 节点执行

```sh
kubeadm init --config kubeadm-config.yaml
```

执行完成后，就会在拉取我们的进行了【需要等待...】

![image-20201121194928988](D:\学习资料\笔记\k8s\k8s图\image-20201121194928988.png)

按照提示配置环境变量，使用kubectl工具

```sh
# 执行下方命令
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
# 查看节点
kubectl get nodes
# 查看pod
kubectl get pods -n kube-system
```

**按照提示保存以下内容，一会要使用：**

```sh
kubeadm join master.k8s.io:16443 --token jv5z7n.3y1zi95p952y9p65 \
    --discovery-token-ca-cert-hash sha256:403bca185c2f3a4791685013499e7ce58f9848e2213e27194b75a2e3293d8812 \
    --control-plane 
```

> --control-plane ： 只有在添加master节点的时候才有

查看集群状态

```sh
# 查看集群状态
kubectl get cs
# 查看pod
kubectl get pods -n kube-system
```



### 安装集群网络

从官方地址获取到flannel的yaml，在master1上执行

```
# 创建文件夹
mkdir flannel
cd flannel
# 下载yaml文件
wget -c https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

安装flannel网络

```
kubectl apply -f kube-flannel.yml 
```

检查

```
kubectl get pods -n kube-system
```



### master2节点加入集群



#### 复制密钥及相关文件

从master1复制密钥及相关文件到master2

```sh
$ ssh root@192.168.44.156 mkdir -p /etc/kubernetes/pki/etcd
$ scp /etc/kubernetes/admin.conf root@192.168.44.156:/etc/kubernetes  
$ scp /etc/kubernetes/pki/{ca.*,sa.*,front-proxy-ca.*} root@192.168.44.156:/etc/kubernetes/pki
$ scp /etc/kubernetes/pki/etcd/ca.* root@192.168.44.156:/etc/kubernetes/pki/etcd
```



#### master2加入集群

执行在master1上init后输出的join命令,需要带上参数`--control-plane`表示把master控制节点加入集群

```sh
kubeadm join master.k8s.io:16443 --token ckf7bs.30576l0okocepg8b     --discovery-token-ca-cert-hash sha256:19afac8b11182f61073e254fb57b9f19ab4d798b70501036fc69ebef46094aba --control-plane
```

检查状态

```sh
$ kubectl get node

$ kubectl get pods --all-namespaces
```



### 加入Kubernetes Node

在node1上执行

向集群添加新节点，执行在kubeadm init输出的kubeadm join命令：

```sh
$ kubeadm join master.k8s.io:16443 --token ckf7bs.30576l0okocepg8b     --discovery-token-ca-cert-hash sha256:19afac8b11182f61073e254fb57b9f19ab4d798b70501036fc69ebef46094aba
```

**集群网络重新安装，因为添加了新的node节点**

检查状态

```sh
$ kubectl get node
$ kubectl get pods --all-namespaces
```



### 测试kubernetes集群

在Kubernetes集群中创建一个pod，验证是否正常运行：

```sh
# 创建nginx deployment
$ kubectl create deployment nginx --image=nginx
# 暴露端口
$ kubectl expose deployment nginx --port=80 --type=NodePort
# 查看状态
$ kubectl get pod,svc
```

然后我们通过任何一个节点，都能够访问我们的nginx页面







## Kubernetes持久化存储



### 前言

之前我们有提到数据卷：`emptydir` ，是本地存储，pod重启，数据就不存在了，需要对数据持久化存储

对于数据持久化存储【pod重启，数据还存在】，有两种方式

- nfs：网络存储【通过一台服务器来存储】



### 步骤

#### 持久化服务器上操作

- 找一台新的服务器nfs服务端，安装nfs
- 设置挂载路径

使用命令安装nfs

```
yum install -y nfs-utils
```

首先创建存放数据的目录

```
mkdir -p /data/nfx
```

设置挂载路径

```
# 打开文件
vim /etc/exports
# 添加如下内容
/data/nfs *(rw,no_root_squash)
```

执行完成后，即部署完我们的持久化服务器

- 常用选项：
  - ro：客户端挂载后，其权限为只读，默认选项；
  - rw:读写权限；
  - sync：同时将数据写入到内存与硬盘中；
  - async：异步，优先将数据保存到内存，然后再写入硬盘；
  - Secure：要求请求源的端口小于1024
- 用户映射：
  - root_squash:当NFS客户端使用root用户访问时，映射到NFS服务器的匿名用户；
  - no_root_squash:当NFS客户端使用root用户访问时，映射到NFS服务器的root用户；
  - all_squash:全部用户都映射为服务器端的匿名用户；
  - anonuid=UID：将客户端登录用户映射为此处指定的用户uid；
  - anongid=GID：将客户端登录用户映射为此处指定的用户gid



**设置开机启动并启动**

- rpcbind

```bash
$ systemctl restart rpcbind
```

- nfs

```bash
$ systemctl enable nfs && systemctl restart nfs
```

**查看是否有可用的NFS地址**

```bash
$ showmount -e 127.0.0.1
```



#### 客户端配置

**1、安装nfs-utils和rpcbind**

```bash
$ yum install -y nfs-utils rpcbind
```

**2、创建挂载的文件夹**

```bash
$ mkdir -p /nfs-data
```

**3、挂载nfs**

```bash
$ mount -t nfs -o nolock,vers=4 192.168.2.31:/nfs /nfs-data
```

- 参数解释：
  - mount：挂载命令
  - -o：挂载选项
  - nfs :使用的协议
  - nolock :不阻塞
  - vers : 使用的NFS版本号
  - IP : NFS服务器的IP（NFS服务器运行在哪个系统上，就是哪个系统的IP）
  - /nfs: 要挂载的目录（Ubuntu的目录）
  - /nfs-data : 要挂载到的目录（开发板上的目录，注意挂载成功后，/mnt下原有数据将会被隐藏，无法找到）
- 查看挂载

```bash
$ df -h
```

- 卸载挂载

```bash
$ umount /nfs-data
```

- 查看nfs版本

```bash
# 查看nfs服务端信息
$ nfsstat -s

# 查看nfs客户端信息
$ nfsstat -c
```



#### Node节点上操作

然后需要在k8s集群node节点上安装nfs，这里需要在 node1 和 node2节点上安装

```
yum install -y nfs-utils
```

执行完成后，会自动帮我们挂载上



#### 启动nfs服务端

下面我们回到nfs服务端，启动我们的nfs服务

```
systemctl start nfs
```

![image-20201119082047766](D:\学习资料\笔记\k8s\k8s图\image-20201119082047766.png)



#### K8s集群部署应用

最后我们在k8s集群上部署应用，使用nfs持久化存储

```
# 创建一个pv文件
mkdir pv
# 进入
cd pv
```

然后创建一个yaml文件 `nfs-nginx.yaml`

![image-20201119082317625](D:\学习资料\笔记\k8s\k8s图\image-20201119082317625.png)

通过这个方式，就挂载到了刚刚我们的nfs数据节点下的 /data/nfs 目录

最后就变成了： /usr/share/nginx/html -> 192.168.44.134/data/nfs 内容是对应的

我们通过这个 yaml文件，创建一个pod

```
kubectl apply -f nfs-nginx.yaml
```

创建完成后，我们也可以查看日志

```
kubectl describe pod nginx-dep1
```

![image-20201119083444454](D:\学习资料\笔记\k8s\k8s图\image-20201119083444454.png)

可以看到，我们的pod已经成功创建出来了，同时下图也是出于Running状态

![image-20201119083514247](D:\学习资料\笔记\k8s\k8s图\image-20201119083514247.png)

下面我们就可以进行测试了，比如现在nfs服务节点上添加数据，然后在看数据是否存在 pod中

```
# 进入pod中查看
kubectl exec -it nginx-dep1 bash
```

![image-20201119095847548](D:\学习资料\笔记\k8s\k8s图\image-20201119095847548.png)



### PV和PVC

对于上述的方式，我们都知道，我们的ip 和端口是直接放在我们的容器上的，这样管理起来可能不方便

![image-20201119082317625](D:\学习资料\笔记\k8s\k8s图\image-20201119082317625.png)

所以这里就需要用到 pv 和 pvc的概念了，方便我们配置和管理我们的 ip 地址等元信息

PV：持久化存储，对存储的资源进行抽象，对外提供可以调用的地方【生产者】

PVC：用于调用，不需要关心内部实现细节【消费者】

PV 和 PVC 使得 K8S 集群具备了存储的逻辑抽象能力。使得在配置Pod的逻辑里可以忽略对实际后台存储 技术的配置，而把这项配置的工作交给PV的配置者，即集群的管理者。存储的PV和PVC的这种关系，跟 计算的Node和Pod的关系是非常类似的；PV和Node是资源的提供者，根据集群的基础设施变化而变 化，由K8s集群管理员配置；而PVC和Pod是资源的使用者，根据业务服务的需求变化而变化，由K8s集 群的使用者即服务的管理员来配置。



#### 实现流程

- PVC绑定PV
- 定义PVC
- 定义PV【数据卷定义，指定数据存储服务器的ip、路径、容量和匹配模式】



#### 举例

创建一个 pvc.yaml

![image-20201119101753419](D:\学习资料\笔记\k8s\k8s图\image-20201119101753419.png)

第一部分是定义一个 deployment，做一个部署

- 副本数：3
- 挂载路径
- 调用：是通过pvc的模式

然后定义pvc

![image-20201119101843498](D:\学习资料\笔记\k8s\k8s图\image-20201119101843498.png)

然后在创建一个 `pv.yaml`

![image-20201119101957777](D:\学习资料\笔记\k8s\k8s图\image-20201119101957777.png)

然后就可以创建pod了

```
kubectl apply -f pv.yaml
```

然后我们就可以通过下面命令，查看我们的 pv 和 pvc之间的绑定关系

```
kubectl get pv, pvc
```

![image-20201119102332786](D:\学习资料\笔记\k8s\k8s图\image-20201119102332786.png)

到这里为止，我们就完成了我们 pv 和 pvc的绑定操作，通过之前的方式，进入pod中查看内容

```
kubect exec -it nginx-dep1 bash
```

然后查看 /usr/share/nginx.html

![image-20201119102448226](D:\学习资料\笔记\k8s\k8s图\image-20201119102448226.png)

也同样能看到刚刚的内容，其实这种操作和之前我们的nfs是一样的，只是多了一层pvc绑定pv的操作



# 云原生存储



## 云原生应用的基石



存储服务支撑了应用的状态、数据的持久化，是计算机系统中的重要组成部分，也是所有应用得以运行的基础，其重要性不言而喻。在存储服务演进过程中，每一种业务类型、新技术方向都会对存储的架构、性能、可用性、稳定性等提出新的要求，而在当今技术浪潮走到云原生技术普及的时代，存储服务需要哪些特性来支持应用呢？



### **1. 概念**

要理解云原生存储，我们首先要了解云原生技术的概念，CNCF 对云原生定义如下：

> 云原生技术有利于各组织在公有云、私有云和混合云等新型动态环境中，构建和运行可弹性扩展的应用。云原生的代表技术包括容器、服务网格、微服务、不可变基础设施和声明式 API。
>
> 这些技术能够构建容错性好、易于管理和便于观察的松耦合系统。结合可靠的自动化手段，云原生技术使工程师能够轻松地对系统作出频繁和可预测的重大变更。



简言之：云原生应用和传统应用并没有一个标准的划分界限，其描述的是一种技术倾向，即越符合以下特征的应用越云原生：

- 应用容器化
- 服务网格化
- 声明式 API
- 运行可弹性扩展
- 自动化的 DevOps
- 故障容忍和自愈
- 平台无关，可移植的



云原生应用是一簇应用特征能力的集合，而实现了这些能力的应用在可用性、稳定性、扩展性、性能等核心能力都会有大幅的优化。优异的能力代表了技术的方向，云原生应用正在引领各个应用领域实现云原生化，同时也在深刻改变着应用服务的方方面面。存储作为应用运行的基石，也在服务云原生化过程中提出了更多的需求。

云原生存储的概念来源于云原生应用，顾名思义：一个应用为了满足云原生特性的要求，其对存储所要求的特性是云原生存储的特性，而满足这些特性的存储方案，可以称其为倾向云原生的存储。



### **2. 云原生存储特征**

#### **1）可用性**

存储系统的可用性定义了在系统故障情况下访问数据的能力，故障可以是由存储介质、传输、控制器或系统中的其他组件造成的。可用性定义系统故障时如何继续访问数据，以及在部分节点不可用时如何将对数据的访问重新路由到其他的可访问节点。

可用性定义了故障的恢复时间目标（RTO），即故障发生与服务恢复之间的时长。可用性通常计算为应用运行时间中的可用时间的百分比（例如 99.9%），以及以时间单位度量的 MTTF（平均故障时间）或 MTTR（平均修复时间）来度量。



#### **2）可扩展性**

存储的可扩展性主要衡量以下参数指标：

- 扩展可以访问存储系统的客户端数量的能力，例如：一个 NAS 存储卷同时支持多少客户端挂载使用；
- 扩展单个接口的吞吐量和 IO 性能；
- 扩展存储服务单实例的容量能力，例如：云盘的扩容能力。



#### **3）性能**

衡量存储的性能通常有两个标准：

- 每秒支持的最大存储操作数 - IOPS；
- 每秒支持的最大存储读写量 - 吞吐量；

云原生应用在大数据分析、AI 等场景得到广泛应用，在这些重吞吐 / IO 场景中对存储的需求也非常高，同时云原生应用的快速扩容、极致伸缩等特性也会考验存储服务在短时间内迎接峰值流量的能力。



#### **4）一致性**

存储服务的一致性是指在提交新数据或更新数据后，访问这些新数据的能力；根据数据一致性的延迟效果，可以将存储分为：“最终一致”和“强一致”存储。

不同的服务对存储一致性的敏感度是不一样的，像数据库这类对底层数据准确性和时效性要求非常高的应用，对存储的要求是具有强一致性的能力。



#### **5）持久性**

多个因素应用数据的持久性：

- 系统冗余等级；
- 存储介质的耐久性（如：SSD 或 HDD）；
- 检测数据损坏的能力，以及使用数据保护功能重建或恢复损坏数据的能力。



### **3. 数据访问接口**

在云原生应用系统中，应用访问存储服务有多种方式。从访问接口类型上可以分为：数据卷方式、API 方式。

![image-20210104114036992](D:\学习资料\笔记\k8s\k8s图\image-20210104114036992.png)

> **数据卷**：将存储服务映射为块或文件系统方式共应用直接访问，使用方式类似于应用在操作系统中直接读写本地目录文件。例如：可以将块存储、文件存储挂载到本地，应用可以像访问本地文件一样对数据卷进行访问；
>
> **API**：有些存储类型并不能使用挂载数据卷的方式进行访问，而需要通过调用 API 的方式进行。例如：数据库存储、KV 存储、对象存储，都是通过 API 的方式进行数据的读写。



需要说明的是对象存储类型，其标准使用方式是通过 Restful API 对外提供了文件的读写能力，但也可以使用用户态文件系统的方式进行存储挂载，使用方式模拟了块、文件存储数据卷挂载的方式。

下面表格列举了各种存储接口的优缺点：

| 接口     | 优点                                           | 缺点                     |
| -------- | ---------------------------------------------- | ------------------------ |
| 块       | 高可用、低延迟、单应用吞吐更高                 | 容量弹缩弱、数据共享性差 |
| 文件系统 | 多负载共享数据、多负载吞吐更高                 | 共享数据时，文件锁性能差 |
| 对象存储 | 高可用、大容量、多负载共享数据、多负载吞吐更高 | 时延高                   |



### **4. 云原生存储分层**

![image-20210104114358728](D:\学习资料\笔记\k8s\k8s图\image-20210104114358728.png)



#### **1）编排和操作系统层**

这一层定义了存储数据的对外访问接口，即应用访问数据时存储所表现的形式。同上节所述，可以分为数据卷方式和 API 访问方式。这一层是容器服务存储编排团队关注的重点，云原生存储对敏捷性、可操作性、扩展性等方面的需求都在这一层集成、实现。



#### **2）存储拓扑层**

这一层定义了存储系统的拓扑结构和架构设计，定义了系统不同部分是如何相互关联和连接的（如存储设备、计算节点和数据）。拓扑结构在构建时影响存储系统的许多属性，因此必须加以考虑。

存储拓扑可以是集中式的、分布式的、或超融合的。



#### **3）数据保护层**

定义如何通过冗余的方式对数据进行保护。在存储系统出现故障时，如何通过数据冗余对数据进行保护、恢复非常重要。通常有以下几种数据保护方案：

- RAID（独立磁盘冗余阵列）：用于在多个磁盘之间的分发数据技术，同时考虑到冗余；
- 擦除编码：数据被分成多个片段，这些片段被编码，并与多个冗余集合一起存储，保证数据可恢复；
- 副本：将数据分布在多个服务器上，实现数据集的多个完整副本。



#### **4）数据服务**

数据服务补充了核心存储功能以外的存储能力，例如可以提供存储快照、数据恢复、数据加密等服务。

数据服务所提供的补充能力也正是云原生存储所需要的，云原生存储正是通过集成丰富数据服务来实现其敏捷、稳定、可扩展等能力。



#### **5）物理层**

这一层定义了存储数据的实际物理硬件。物理硬件的选择影响系统的整体性能和存储数据的持续性。



### **5. 存储编排**

云原生意味着容器化，而容器服务场景中通常需要某种管理系统或者应用编排系统。这些编排系统与存储系统的交互可以实现工作负载与存储数据的关联。

![image-20210104153537595](D:\学习资料\笔记\k8s\k8s图\image-20210104153537595.png)

如上图所示：

- 负载：表示应用实例，会消费底层存储资源；
- 编排系统：类似于 K8s 一样的容器编排系统，负责应用的管理和调度；
- 控制平面接口：指编排系统调度、运维底层存储资源的标准接口，例如 K8s 里面的 Flexvolume，已经容器存储通用的 CSI 接口；
- 工具集：指控制平面接口运维存储资源时所依赖的三方工具、框架；
- 存储系统：分为控制平台数据数据平面。控制台平面对外暴露接口，提供存储资源的接入、接出能力。数据平面提供数据存储服务。



当应用负载定义了存储资源需求时，编排系统会为应用负载去准备这些存储资源。编排系统通过调用控制平面接口，进而实现对存储系统控制平面的调用，这样就实现了应用负载对存储服务的接入、接出操作。当实现了存储系统接入后，应用负载可以直接访问存储系统的数据平面，即可以直接访问数据。





### 常见的云原生存储方案



#### **1. 公有云存储**

每个公有云服务提供商都会提供各种云资源，其中也包含了各种云存储服务。以阿里云为例，其几乎提供了能够满足所有业务需要的存储类型，包括：对象存储、块存储、文件存储、数据库等等。公有云存储的优势是规模效应，足够大的体量实现了巨大研发、运维投入的同时，提供低廉的价格优势。而且公有云存储在稳定性、性能、扩展性方面都能轻松满足您的业务需求。

随着云原生技术的发展，各个公有云厂商都开始对其云服务进行云原生化改造或适配，提供更加敏捷、高效的服务来适应云原生应用的需求。阿里云存储服务也在云原生应用适配做了很多优化，阿里云容器服务团队实现的 CSI 存储驱动无缝的衔接了云原生应用和存储服务之间的数据接口。实现了用户使用存储资源时对底层存储无感知，而专注于自己的业务开发。

![image-20210104154220274](D:\学习资料\笔记\k8s\k8s图\image-20210104154220274.png)

优点如下所示：

- 高可靠性：多数云厂商都可以提供服务稳定性、数据可考虑下都非常优异的服务，例如：阿里云ebs提供了9个9的可靠性服务，为您的数据安全提供了强有力的基础保障能力；

- 高性能：公有云对不同的服务提供了不同等级的存储性能适配，几乎可以满足所有应用类型对存储性能的需求。阿里云 EBS 可以提供百万级别的 IOPS 能力，接近本地盘的访问性能。NAS 服务最大提供每秒数十 G 的吞吐能力，在数据共享的应用场景满足您的高性能需求。而 CPFS 高性能并发文件系统最高可以提供近 TB 级别的吞吐能力，更是可以满足一些极端高性能计算对存储的需求；

- 扩展性好：公有云存储服务一般都提供了容量扩容能力，让您在应用对存储的需求增加的时候可以动态的实现容量伸缩，且实现应用的无感知；

- 安全性高：不同的云存储服务都提供了数据安全的保护机制，通过 KMS、AES 等加密技术实现数据的加密存储，同时也实现了客户端到服务的链路加密方案，让数据传输过程中也得到加密保护；

- 成熟的云原生存储接口：提供了兼容所有存储类型的云原生存储接口，让您的应用无缝的接入不同存储服务。阿里云容器服务提供的 CSI 接口驱动已经支持了：云盘、OSS、NAS、本地盘、内存、LVM等多种存储类型，可以让应用无感知的访问任何的存储服务类型；

- 免运维：相对于自建存储服务来说，公有云存储方案省去了运维的难度



缺点如下所示：

- 定制化能力差：由于公有云存储方案需要服务所有用户场景，其能力主要集中在一些通用需求，而对某些用户个性化需求很难满足。



#### **2. 商业化云存储**

在很多私有云环境中，业务方为了实现数据的高可靠性通常会购买商业化的存储服务。这种方案为用户提供了高可用、高效、便捷的存储服务，且运维服务、后期保障等也都有保证。私有云存储提供商也意识到云原生应用的普及，也会为用户提供完善的、成熟的云原生存储接口实现方案。



优点如下所示：

- 安全性好：私有云部署，可以从物理上实现数据的安全隔离；
- 高可靠、高性能：很多云存储提供商都是在存储技术上深耕多年，具有优异的技术能力和运维能力，其商业化存储服务可以满足多数应用的性能、可靠性的需求；
- 云原生存储接口：从多家存储服务提供商开源的项目可以看出，其对云原生应用的支持已经实现或者展开。



缺点如下所示：

- 贵：商业化的存储服务多数都很昂贵；
- 云原生存储接口兼容性：商业化的云原生存储接口都是针对其一家的存储类型。多数用户会使用不同的存储类型，而使用了不同家的存储服务，很难实现统一的存储接入能力。



#### **3. 自建存储服务**

对于一些 SLA 不是很高的业务数据，很多公司都会选择使用自建的方式提供存储服务。业务方需要通过当前的开源的存储方案，并结合自建的业务需求进行方案选择。

- **文件存储**：考虑 CephFS、GlusterFS、NFS 等方案。其中 CephFS，GlusterFS 在技术的成熟度上还需要进一步验证，且在高可靠、高性能场景上也存在不足。而 NFS 虽然已经成熟，但是自建集群在性能上很难达到高性能应用的需求；
- **块存储**：例如 RBD、SAN 等是常见的块存储方案，技术也相对比较成熟，已经有较多的公司将其应用在自己的业务上。但其复杂度也相对很高，需要有专业的团队来运维支持。



优点如下所示：

- **业务匹配度高、灵活性好**：可以在众多开源方案中选择最适合自己业务的方案，且可以在原生代码的基础上进行二次开发来优化业务场景；
- **安全性好**：如果搭建在公司内部使用，则具有物理隔离的安全性；
- **云原生存储接口**：常用的开源存储方案都可以在社区找到云原生存储接口的实现，且可以在其基础上进行开发、优化。



缺点如下所示：

- 性能欠佳：多数开源的存储方案其原生的性能表现并不是很好。当然您可以通过架构设计、物理硬件升级、二次开发等方案进行优化；
- 可靠性差：开源存储方案在可靠性方面无法和商业化的存储比较，所以更多场景是应用在 SLA 低的数据存储场景；
- 云原生存储插件鱼龙混杂：目前网上开源的云原生存储驱动版本众多，且质量参差不齐，有些项目存在 bug，且长期无人维护，所以使用时需要更多的甄别和调测工作；
- 专业的团队支撑：自己搭建的服务需要自己负责，面对并不是十分成熟的开源方案，需要组建一个具有较强技术能力的专业团队来运维、开发存储系统。



#### **4. 本地存储**

一些业务类型不需要高可用分布式存储服务，而会选择使用性能表现更优的本地存储方案。

数据库类服务：对存储的 IO 性能、访问时延有很多的要求，一般的块存储服务并不能很好的满足这方面的需求。且其应用本身已经实现了数据的高可用设计，不再需要底层实现多副本的能力，即分布式存储的多副本设计对这类应用是一种浪费。

存储作为缓存：部分应用期望保存一些不重要的数据，数据在程序执行完成即可以丢掉，且对存储的性能要求较高，其本质是将存储作为缓存使用。云盘的高可用能力对这样的业务并没有太大的意义，且云盘在 IO 性能、价格方面的表现（相对本地盘）也没有优势。

所以本地盘存储在很多关键能力上要比分布式块存储弱很多，但在特定场景下仍然有其使用的优势。阿里云存储服务提供了基于 NVMe 的本地盘存储方案，以更好的性能表现和更低的价格优势，在特定的应用场景得到了用户的青睐。

![image-20210104164608746](D:\学习资料\笔记\k8s\k8s图\image-20210104164608746.png)

阿里云 CSI 驱动供了云原生应用使用本地存储的接入实现，支持：lvm 卷、本地盘裸设备、本地目录映射等多种接入形式，实现数据的高性能访问、quota、iops 配置等众多适配能力。

优点如下所示：

- 性能高：提供相对分布式存储更优的 IOPS、吞吐能力；
- 低价：通过物理裸设备直接提供本地盘，在价格上相对于多副本的分布式存储具有优势。



缺点如下所示：

- 数据可靠性差：本地盘保存的数据丢失后不能找回，需要从应用层实现数据的高可用设计；
- 灵活性差：不能像云盘一样实现数据迁移到其他节点使用。



#### **5. 开源容器存储**

随着云原生技术的发展，社区提供了一些开源的云原生存储方案。



##### **1）Rook**

Rook 作为第一个 CNCF 存储项目，是一个集成了 Ceph、Minio 等分布式存储系统的云原生存储方案，意图实现一键式部署、管理方案，且和容器服务生态深度融合，提供适配云原生应用的各种能力。从实现上，可以认为 Rook 是一个提供了 Ceph 集群管理能力的 Operator。其使用 CRD 方式来对 Ceph、Minio 等存储资源进行部署和管理。

![image-20210104164813586](D:\学习资料\笔记\k8s\k8s图\image-20210104164813586.png)

Rook 组件：

- Operator：实现自动启动存储集群，并监控存储守护进程，并确保存储集群的健康；
- Agent：在每个存储节点上运行，并部署一个 CSI / FlexVolume 插件，和 Kubernetes 的存储卷控制框架进行集成。Agent 处理所有的存储操作，例如挂载存储设备、加载存储卷以及格式化文件系统等；
- Discovers：检测挂接到存储节点上的存储设备。



Rook 将 Ceph 存储服务作为 Kubernetes 的一个服务进行部署，MON、OSD、MGR 守护进程会以 pod 的形式在 Kubernetes 进行部署，而 rook 核心组件对 ceph 集群进行运维管理操作。

Rook 通过 ceph 可以对外提供完备的存储能力，支持对象、块、文件存储服务，让你通过一套系统实现对多种存储服务的需求。同时 rook 默认部署云原生存储接口的实现，通过 CSI / Flexvolume 驱动将应用服务与底层存储进行衔接，其设计之初即为 Kubernetes 生态所服务，对容器化应用的适配非常友好。

Rook 官方文档参考：https://rook.io/



##### **2）OpenEBS**

OpenEBS 是一种模拟了 AWS 的 EBS、阿里云的云盘等块存储实现的开源版本。OpenEBS 是一种基于 CAS 理念的容器解决方案，其核心理念是存储和应用一样采用微服务架构，并通过 Kubernetes 来做资源编排。其架构实现上，每个卷的 Controller 都是一个单独的 Pod，且与应用 Pod 在同一个 Node，卷的数据使用多个 Pod 进行管理。

![image-20210104165507850](D:\学习资料\笔记\k8s\k8s图\image-20210104165507850.png)

架构上可以分为数据平面（Data Plane）和控制平面（Control Plane）两部分：

- 数据平面：为应用程序提供数据存储；
- 控制平面：管理 OpenEBS 卷容器，通常会用到容器编排软件的功能；



数据平面：

> OpenEBS 持久化存储卷通过 Kubernetes 的 PV 来创建，使用 iSCSI 来实现，数据保存在 node 节点上或者云存储中。OpenEBS 的卷完全独立于用户的应用的生命周期来管理，和 Kuberentes 中 PV 的思路一致。
>
> OpenEBS 卷为容器提供持久化存储，具有针对系统故障的弹性，更快地访问存储，快照和备份功能。同时还提供了监控使用情况和执行 QoS 策略的机制。



控制平面：

> OpenEBS 控制平面 maya 实现了创建超融合的 OpenEBS，并将其挂载到如 Kubernetes 调度引擎上，用来扩展特定的容器编排系统提供的存储功能。
>
> OpenEBS 的控制平面也是基于微服务的，通过不同的组件实现存储管理功能、监控、容器编排插件等功能。



更多关于 OpenEBS 的介绍可以参考：https://openebs.io/



##### **3）Heketi**

类似于 Rook 是 Ceph 开源存储系统在云原生编排平台（Kubernetes）的一个落地方案，Glusterfs 同样也有一个云原生实践方案。Heketi 提供了一个 Restful 管理接口，可用于管理 Gluster 存储卷的生命周期。使用 Heketi，Kubernetes 可以动态地为 Gluster 存储卷提供任何支持的持久性类型。Heketi 将自动确定集群中 brick 的位置，确保在不同的故障域中放置 brick 及其副本。Heketi 还支持任意数量的 Gluster 存储集群，为云服务提供网络文件存储。

使用 Heketi，管理员不再管理或配置块、磁盘或存储池。Heketi 服务将为系统管理所有硬件，使其能够按需分配存储。任何在 Heketi 注册的物理存储必须以裸设备方式提供，然后 Heketi 在提供的磁盘上使用 LVM 进行管理。

![image-20210104165747668](D:\学习资料\笔记\k8s\k8s图\image-20210104165747668.png)

更多详解参考：https://github.com/heketi/heketi



#### **6. 优势**

- 上述几种云原生存储方案其设计之初既充分考虑了存储与云原生编排系统的融合，具有很好的容器数据卷接入能力；
- 在 Quota 配置、QoS 限速、ACL 控制、快照、备份等方面有较好的云原生集成实现，云原生应用使用存储资源时更加灵活、便利；
- 开源方案，社区较为活跃，网络资源、使用方案丰富，让您更容易入手。



#### **7. 劣势**

- 成熟度较低，目前上述方案多在公司内部测试环境或者 SLA 较低的应用中使用，很少存储关键应用数据；
- 性能差：和公有云存储、商业化存储相比，上述云原生存储方案在 IO 性能、吞吐、时延等方面都表现欠佳，很难应用在高性能服务场景；
- 后期维护成本高：虽然上面方案部署、入门都很简单，但一旦运行中出现问题解决起来非常棘手。上述项目属于初期阶段，并不具备生产级别的服务能力，如果使用此方案需要有强有力的技术团结加以保障。



### 现状和挑战



#### **1. 敏捷化需求**

云原生应用场景对服务的敏捷度、灵活性要求非常高，很多场景期望容器的快速启动、灵活的调度，这样即需要存储卷也能敏捷的根据 Pod 的变化而调整。

需求表现在：

- 云盘挂载、卸载效率提高：可以灵活的将块设备在不同节点进行快速的挂载切换；
- 存储设备问题自愈能力增强：提供存储服务的问题自动修复能力，减少人为干预；
- 提供更加灵活的卷大小配置能力。



#### **2. 监控能力需求**

多数存储服务在底层文件系统级别已经提供了监控能力，然后从云原生数据卷角度的监控能力仍需要加强，目前提供的 PV 监控数据维度较少、监控力度较低；

具体需求：

- 提供更细力度（目录）的监控能力；
- 提供更多维度的监控指标：读写时延、读写频率、IO 分布等指标。



#### **3. 性能要求**

在大数据计算场景同时大量应用访问存储的需求很高，这样对存储服务带来的性能需求成为应用运行效率的关键瓶颈。

具体需求：

- 底层存储服务提供更加优异的存储性能服务，优化 CPFS、GPFS 等高性能存储服务满足业务需求；
- 容器编排层面：优化存储调度能力，实现存储就近访问、数据分散存储等方式降低单个存储卷的访问压力。



#### **4. 共享存储的隔离性**

共享存储提供了多个 Pod 共享数据的能力，方便了不同应用对数据的统一管理、访问，但在多租的场景中，不同租户对存储的隔离性需求成为一个需要解决的问题。

> 底层存储提供目录间的强隔离能力，让共享文件系统的不同租户之间实现文件系统级别的隔离效果。
>
> 容器编排层实现基于命名空间、PSP 策略的编排隔离能力，保证不同租户从应用部署侧即无法访问其他租户的存储卷服务。





## 容器存储与 K8s 存储卷



云原生存储的两个关键领域：Docker 存储卷、K8s 存储卷；

- **Docker 存储卷**：容器服务在单节点的存储组织形式，关注数据存储、容器运行时的相关技术；
- **K8s 存储卷**：关注容器集群的存储编排，从应用使用存储的角度关注存储服务。



### Docker 存储

容器服务之所以如此流行，一大优势即来自于运行容器时容器镜像的组织形式。容器通过复用容器镜像的技术，实现在相同节点上多个容器共享一个镜像资源（更细一点说是共享某一个镜像层），避免了每次启动容器时都拷贝、加载镜像文件，这种方式既节省了主机的存储空间，又提高了容器启动效率。



#### 1. 容器读写层

为了提高节点存储的使用效率，容器不光在不同运行的容器之间共享镜像资源，而且还实现了在不同镜像之间共享数据。共享镜像数据的实现原理：镜像是分层组合而成的，即一个完整的镜像会包含多个数据层，每层数据相互叠加、覆盖组成了最终的完整镜像。

为了实现多个容器间共享镜像数据，容器镜像每一层都是只读的。而通过实践我们得知，使用镜像启动一个容器的时候，其实是可以在容器里随意读写的，这是如何实现的呢？

容器使用镜像时，在多个镜像分层的最上面还添加了一个读写层。每一个容器在运行时，都会基于当前镜像在其最上层挂载一个读写层，用户针对容器的所有操作都在读写层中完成。一旦容器销毁，这个读写层也随之销毁。

![1.png](D:\学习资料\笔记\k8s\k8s图\44f677428a8abfb81003d33da82b6cc6.png)

如上图所示例子，一个节点上共有 3 个容器，分别基于 2 个镜像运行。

镜像存储层说明如下：

该节点上共包含 6 个镜像层：Layer 1~6。

- 镜像 1 由：Layer 1、3、4、5 组成；
- 镜像 2 由：Layer 2、3、5、6 组成。

所以两个镜像共享了 Layer 3、5 两个镜像层；



容器存储说明：

> - 容器 1：使用镜像 1 启动
> - 容器 2：使用镜像 1 启动
> - 容器 3：使用镜像 2 启动
>
> 容器 1 和容器 2 共享镜像 1，且每个容器有自己的可写层；
>
> 容器 1（2）和容器 3 共享镜像 2 个层（Layer3、5）；



通过上述例子可以看到，通过容器镜像分层实现数据共享可以大幅减少容器服务对主机存储的资源需求。

上面给出了容器读写层结构，而读写的原则：

> 对于读：容器由这么多层的数据组合而成，当不同层次的数据重复时，读取的原则是上层数据覆盖下层数据；
>
> 对于写：容器修改某个文件时，都是在最上层的读写层进行。主要实现技术有：写时复制、用时配置。



##### **1）写时复制**

写时复制（CoW：copy-on-write），表示只在需要写时才去复制，是针对已有文件的修改场景。CoW 技术可以让所有的容器共享 image 的文件系统，所有数据都从 image 中读取，只有当要对文件进行写操作时，才从 image 里把要写的文件复制到最上面的读写层进行修改。所以无论有多少个容器共享同一个 image，所做的写操作都是对从 image 中复制后在复本上进行，并不会修改 image 的源文件，且多个容器操作同一个文件，会在每个容器的文件系统里生成一个复本，每个容器修改的都是自己的复本，相互隔离，相互不影响。



##### **2）用时分配**

用时分配：在镜像中原本没有某个文件的场景，只有在要新写入一个文件时才分配空间，这样可以提高存储资源的利用率。比如启动一个容器，并不会为这个容器预分配一些磁盘空间，而是当有新文件写入时，才按需分配新空间。



#### **2. 存储驱动**

存储驱动是指如何对容器的各层数据进行管理，已达到上述需要实现共享、可读写的效果。即：容器存储驱动实现了容器读写层数据的存储和管理。常见的存储驱动：

- AUFS
- OverlayFS
- Devicemapper
- Btrfs
- ZFS



以 AUFS 为例，我们来讲述一下存储驱动的工作原理：

![image-20210104175311076](D:\学习资料\笔记\k8s\k8s图\image-20210104175311076.png)

AUFS 是一种联合文件系统（UFS），是文件级的存储驱动。

> AUFS 是一个能透明叠加一个或多个现有文件系统的层状文件系统，把多层文件系统合并成单层表示。即：支持将不同目录挂载到同一个虚拟文件系统下的文件系统。
>
> 可以一层一层地叠加修改文件，其底层都是只读的，只有最上层的文件系统是可写的。
>
> 当需要修改一个文件时，AUFS 创建该文件的一个副本，使用 CoW 将文件从只读层复制到可写层进行修改，结果也保存在可写层。
>
> 在 Docker 中，底下的只读层就是 image，可写层就是 Container 运行时。



#### **3. Docker 数据卷介绍**

容器中的应用读写数据都是发生在容器的读写层，镜像层+读写层映射为容器内部文件系统、负责容器内部存储的底层架构。当我们需要容器内部应用和外部存储进行交互时，需要一个类似于计算机 U 盘一样的外置存储，容器数据卷即提供了这样的功能。

另一方面：容器本身的存储数据都是临时存储，在容器销毁的时候数据会一起删除。而通过数据卷将外部存储挂载到容器文件系统，应用可以引用外部数据，也可以将自己产出的数据持久化到数据卷中，所以容器数据卷是容器进行数据持久化的实现方式。

```
容器存储组成：只读层（容器镜像） + 读写层 + 外置存储（数据卷）
```

容器数据卷从作用范围可以分为：单机数据卷 和 集群数据卷。单机数据卷即为容器服务在一个节点上的数据卷挂载能力，docker volume 是单机数据卷的代表实现；集群数据卷则关注的是集群级别的数据卷编排能力，K8s 数据卷则是集群数据卷的主要应用方式。

Docker Volume 是一个可供多个容器使用的目录，它绕过 UFS，包含以下特性：

- 数据卷可以在容器之间共享和重用；
- 相比通过存储驱动实现的可写层，数据卷读写是直接对外置存储进行读写，效率更高；
- 对数据卷的更新是对外置存储读写，不会影响镜像和容器读写层；
- 数据卷可以一直存在，直到没有容器使用。



##### **1）Docker 数据卷类型**

Bind：将主机目录/文件直接挂载到容器内部。

> - 需要使用主机的上的绝对路径，且可以自动创建主机目录；
> - 容器可以修改挂载目录下的任何文件，是应用更具有便捷性，但也带来了安全隐患。



Volume：使用第三方数据卷的时候使用这种方式。

> - Volume 命令行指令：docker volume (create/rm)；
> - 是 Docker 提供的功能，所以在非 docker 环境下无法使用；
> - 分为命名数据卷和匿名数据卷，其实现是一致的，区别是匿名数据卷的名字为随机码；
> - 支持数据卷驱动扩展，实现更多外部存储类型的接入。



Tmpfs：非持久化的卷类型，存储在内存中。

> 数据易丢失。



##### **2）Bind 挂载方式语法**

-v: src:dst:opts 只支持单机版。

> - Src：表示卷映射源，主机目录或文件，需要是绝对地址；
> - Dst：容器内目标挂载地址；
> - Opts：可选，挂载属性：ro, consistent, delegated, cached, z, Z；
> - Consistent, delegated, cached：为 mac 系统配置共享传播属性；
> - Z、z：配置主机目录的 selinux label。



示例：

```sh
$ docker run -d --name devtest -v /home:/data:ro,rslave nginx
$ docker run -d --name devtest --mount type=bind,source=/home,target=/data,readonly,bind-propagation=rslave nginx
$ docker run -d --name devtest -v /home:/data:z nginx
```



##### **3）Volume 挂载方式语法**

-v: src:dst:opts 只支持单机版。

> - Src：表示卷映射源，数据卷名、空
> - Dst：容器内目标目录
> - Opts：可选，挂载属性：ro（只读）



示例：

```sh
$ docker run -d --name devtest -v myvol:/app:ro nginx
$ docker run -d --name devtest --mount source=myvol2,target=/app,readonly nginx
```



#### **4. Docker 数据卷使用**

Docker 数据卷使用方式：

##### **1）Volume 类型**

- 匿名数据卷：`docker run –d -v /data3 nginx`；
- 会在主机上默认创建目录：`/var/lib/docker/volumes/{volume-id}/_data` 进行映射；
- 命名数据卷：`docker run –d -v nas1:/data3 nginx`；
- 如果当前找不到 nas1 卷，会创建一个默认类型（local）的卷。

 

##### **2）Bind 方式**

> `docker run -d -v /test:/data nginx`
>
> 如果主机上没有/test目录，则默认创建此目录。



##### **3）数据卷容器**

数据卷容器是一个运行中的容器，其他容器可以继承此容器中的挂载数据卷，则此容器的所有挂载都会在引用容器中体现。

> `docker run -d --volumes-from nginx1 -v /test1:/data1 nginx`
>
> 继承所有来自配置容器的数据卷，并包含自己定义的卷。



##### **4）数据卷的挂载传播**

Docker volume 支持挂载传播的配置：Propagation。

- Private：挂载不传播，源目录和目标目录中的挂载都不会在另一方体现；
- Shared：挂载会在源和目的之间传播；
- Slave：源对象的挂载可以传播到目的对象，反之不行；
- Rprivate：递归 Private，默认方式；
- Rshared：递归 Shared；
- Rslave：递归 Slave。



示例：

```sh
$ docker run –d -v /home:/data:shared nginx
表示：主机/home下面挂载的目录，在容器/data下面可用，反之可行；

$ docker run –d -v /home:/data:slave nginx
表示：主机/home下面挂载的目录，在容器/data下面可用，反之不行；
```



##### **5）数据卷挂载的可见性**

Volume 挂载可见性：

- 本地空目录、镜像空目录：无特殊处理；
- 本地空目录、镜像非空目录：镜像目录的内容拷贝到主机；(是拷贝，不是映射；即使容器删除内容也会保存)；
- 本地非空目录、镜像空目录：本地目录内容映射到容器；
- 本地非空目录、镜像非空目录：本地目录内容映射到容器,容器目录的内容被隐藏。



Bind 挂载可见性：以主机目录为准。

- 本地空目录、镜像空目录：无特殊处理；
- 本地空目录、镜像非空目录：容器目录变成空；
- 本地非空目录、镜像空目录：本地目录内容映射到容器；
- 本地非空目录、镜像非空目录：本地目录内容映射到容器，容器目录的内容被隐藏。



#### **5. Docker 数据卷插件**

Docker 数据卷实现了将容器外部存储挂载到容器文件系统的方式。为了扩展容器对外部存储类型的需求，docker 提出了通过存储插件的方式挂载不同类型的存储服务。扩展插件统称为 Volume Driver，可以为每种存储类型开发一种存储插件。

- 单个节点上可以部署多个存储插件；
- 一个存储插件负责一种存储类型的挂载服务。

![image-20210104181439138](D:\学习资料\笔记\k8s\k8s图\image-20210104181439138.png)



Docker Daemon 与 Volume driver 通信方式有：

- Sock 文件：linux 下放在 `/run/docker/plugins` 目录下
- Spec 文件：`/etc/docker/plugins/convoy.spec` 定义
- Json 文件：`/usr/lib/docker/plugins/infinit.json` 定义



实现接口：

> Create, Remove, Mount, Path, Umount, Get, List, Capabilities;

使用示例：

```sh
$ docker volume create --driver nas -o diskid="" -o host="10.46.225.247" -o path=”/nas1" -o mode="" --name nas1
```

Docker Volume Driver 适用在单机容器环境或者 swarm 平台进行数据卷管理，随着 K8s 的流行其使用场景已经越来越少，关于 VolumeDriver 的详细介绍这里不在细讲。

有兴趣可以参考：https://docs.docker.com/engine/extend/plugins_volume/



### K8s 存储卷



#### **1. 基础概念**

根据之前的描述，为了实现容器数据的持久化我们需要使用数据卷的功能，在 K8s 编排系统中如何为运行的负载(Pod)定义存储呢？K8s 是一个容器编排系统，其关注的是容器应用在整个集群的管理和部署形式，所以在考虑 K8s 应用存储的时候就需要从集群角度考虑。K8s 存储卷定义了在 K8s 系统中应用与存储的关联关系。其包含以下概念：



##### **1）Volume 数据卷**

数据卷定义了外置存储的细节，并内嵌到 Pod 中作为 Pod 的一部分。其实质是外置存储在 K8s 系统的一个记录对象，当负载需要使用外置存储的时候，从数据卷中查到相关信息并进行存储挂载操作。

- 生命周期和 Pod 一致，即 pod 被删除的时候数据卷也一起消失（注意不是数据删除）；
- 存储细节定义在编排模板中，应用编排感知存储细节；
- 一个负载（Pod）中可以同时定义多个 volume，可以是相同类型或不同类型的存储；
- Pod 的每个 container 可以引用一个或多个 volume，不同 container 可以同时使用相同 volume。



K8S Volume 常用类型：

- **本地存储**：如 HostPath、emptyDir，这些存储卷的特点是，数据保存在集群的特定节点上，并且不能随着应用飘逸，节点宕机时数据即不再可用；
- **网络存储**：Ceph、Glusterfs、NFS、Iscsi 等类型，这些存储卷的特点是数据不在集群的某个节点上，而是在远端的存储服务上，使用存储卷时需要将存储服务挂载到本地使用；
- **Secret/ConfigMap**：这些存储卷类型，其数据是集群的一些对象信息，并不属于某个节点，使用时将对象数据以卷的形式挂载到节点上供应用使用；
- **CSI/Flexvolume**：这是两种数据卷扩容方式，可以理解为抽象的数据卷类型。每种扩展方式都可再细化成不同的存储类型；
- **PVC**：一种数据卷定义方式，将数据卷抽象成一个独立于 pod 的对象，这个对象定义（关联）的存储信息即存储卷对应的真正存储信息，供 K8s 负载挂载使用。



一些 volume 模板示例如下：

```yaml
volumes:
  - name: hostpath
    hostPath:
      path: /data
      type: Directory
---
  volumes:
  - name: disk-ssd
    persistentVolumeClaim:
      claimName: disk-ssd-web-0
  - name: default-token-krggw
    secret:
      defaultMode: 420
      secretName: default-token-krggw
---
  volumes:
    - name: "oss1"
      flexVolume:
        driver: "alicloud/oss"
        options:
          bucket: "docker"
          url: "oss-cn-hangzhou.aliyuncs.com"
```



##### **2）PVC 和 PV**

- K8s 存储卷是一个集群级别的概念，其对象作用范围是整个 K8s 集群，而不是而一个节点；
- K8s 存储卷包含一些对象（PVC、PV、SC），这些对象和应用负载（Pod）是独立，通过编排模板进行关联；
- K8s 存储卷可以有自己的独立生命周期，不依附于 Pod。



PVC 是 PersistentVolumeClaim 的缩写，译为存储声明；PVC 是在 K8s 中一种抽象的存储卷类型，代表了某个具体类型存储的数据卷表达。其设计意图是：存储与应用编排分离，将存储细节抽象出来并实现存储的编排（存储卷）。这样 K8s 中存储卷对象独立于应用编排而单独存在，在编排层面使应用和存储解耦。

PV 是 PersistentVolume 的缩写，译为持久化存储卷；PV 在 K8s 中代表一个具体存储类型的卷，其对象中定义了具体存储类型和卷参数。即目标存储服务所有相关的信息都保存在 PV 中，K8s 引用 PV 中的存储信息执行挂载操作。

应用负载、PVC、PV 的关联关系为：

![image-20210104182324008](D:\学习资料\笔记\k8s\k8s图\image-20210104182324008.png)

从实现上看，只要有了 PV 既可以实现存储和应用的编排分离，也能实现数据卷的挂载，为何要用 PVC + PV 两个对象呢？

K8s 这样设计是从应用角度对存储卷进行二次抽象；由于 PV 描述的是对具体存储类型，需要定义详细的存储信息，而应用层用户在消费存储服务的时候往往不希望对底层细节知道的太多，让应用编排层面来定义具体的存储服务不够友好。这时对存储服务再次进行抽象，只把用户关系的参数提炼出来，用 PVC 来抽象更底层的 PV。所以 PVC、PV 关注的对象不一样，PVC 关注用户对存储需求，给用户提供统一的存储定义方式；而 PV 关注的是存储细节，可以定义具体存储类型、存储挂载使用的详细参数等。

使用时应用层会声明一个对存储的需求（PVC），而 K8s 会通过最佳匹配的方式选择一个满足 PVC 需求的 PV，并与之绑定。所以从职责上 PVC 是应用所需要的存储对象，属于应用作用域（和应用处于一个命名空间）；PV 是存储平面的存储对象，属于整个存储域（不属于某个命名空间）；

下面给出 PVC、PV 的一些属性：

- PVC 和 PV 总是成对出现的，PVC 必须与 PV 绑定后才能被应用（Pod）消费；
- PVC 和 PV 是一一绑定关系，不存在一个 PV 被多个 PVC 绑定，或者一个 PVC 绑定多个 PV 的情况；
- PVC 是应用层面的存储概念，是属于具体的命名空间的；
- PV 是存储层面的存储概念，是集群级别的，不属于某个命名空间；PV 常由专门的存储运维人员负责管理；
- 消费关系上：Pod 消费 PVC，PVC 消费 PV，而 PV 定义了具体的存储介质。



##### **3）PVC 详细定义**

PVC 定义的模板如下：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: disk-ssd-web-0
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: alicloud-disk-available
  volumeMode: Filesystem
```



PVC 定义的存储接口包括：存储的读写模式、资源容量、卷模式等；主要参数说明如下：

**accessModes**：存储卷的访问模式，支持：ReadWriteOnce、ReadWriteMany、ReadOnlyMany 三种模式。

> - ReadWriteOnce 表示 pvc 只能同时被一个 pod 以读写方式消费；
> - ReadWriteMany 可以同时被多个 pod 以读写方式消费；
> - ReadOnlyMany 表示可以同时被多个 pod 以只读方式消费。
>
> **注意**：这里定义的访问模式只是编排层面的声明，具体应用在读写存储文件的时候是否可读可写，需要具体的存储插件实现确定。

**storage**：定义此 PVC 对象期望提供的存储容量，同样此处的数据大小也只是编排声明的值，具体存储容量要看底层存储服务类型。

**volumeMode**：表示存储卷挂载模式，支持 FileSystem、Block 两种模式；

> FileSystem：将数据卷挂载成文件系统的方式供应用使用；
>
> Block：将数据卷挂载成块设备的形式供应用使用。



##### **4）PV 详细定义**

下面为云盘数据卷 PV 对象的编排示例：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  labels:
    failure-domain.beta.kubernetes.io/region: cn-shenzhen
    failure-domain.beta.kubernetes.io/zone: cn-shenzhen-e
  name: d-wz9g2j5qbo37r2lamkg4
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 30Gi
  flexVolume:
    driver: alicloud/disk
    fsType: ext4
    options:
      VolumeId: d-wz9g2j5qbo37r2lamkg4
  persistentVolumeReclaimPolicy: Delete
  storageClassName: alicloud-disk-available
  volumeMode: Filesystem
```



> - accessModes：存储卷的访问模式，支持：ReadWriteOnce、ReadWriteMany、ReadOnlyMany 三种模式；具体含义同 PVC 字段；
> - capacity：定义存储卷容量；
> - persistentVolumeReclaimPolicy：定义回收策略，即删除 pvc 的时候如何处理 PV；支持 Delete、Retain 两种类型，动态数据卷部分会详细说明此参数；
> - storageClassName：表示存储卷的使用的存储类名字，动态数据卷部分会详细说明此参数；
> - volumeMode：同 PVC 中的 volumeMode 定义；
> - Flexvolume：此字段表示具体的存储类型，这里 Flexvolume 为一种抽象的存储类型，并在 flexvolume 的子配置项中定义了具体的存储类型、存储参数。



##### **5）PVC/PV 绑定**

PVC 只有绑定了 PV 之后才能被 Pod 使用，而 PVC 绑定 PV 的过程即是消费 PV 的过程，这个过程是有一定规则的，下面规则都满足的 PV 才能被 PVC 绑定：

- **VolumeMode**：被消费 PV 的 VolumeMode 需要和 PVC 一致；
- **AccessMode**：被消费 PV 的 AccessMode 需要和 PVC 一致；
- **StorageClassName**：如果 PVC 定义了此参数，PV 必须有相关的参数定义才能进行绑定；
- **LabelSelector**：通过 label 匹配的方式从 PV 列表中选择合适的 PV 绑定；
- **storage**：被消费 PV 的 capacity 必须大于或者等于 PVC 的存储容量需求才能被绑定。



满足上述所有需要的 PV 才可以被 PVC 绑定。

> 如果同时有多个 PV 满足需求，则需要从 PV 中选择一个更合适的进行绑定；通常选择容量最小的，如果容量最小的也有多个，则随机选择。
>
> 如果没有满足上述需求的 PV 存储，则 PVC 会处于 Pending 状态，等待有合适的 PV 出现了再进行绑定。



#### **2. 静态、动态存储卷**

从上面的讨论我们了解到，PVC 是针对应用服务对存储的二次抽象，具有简洁的存储定义接口。而 PV 是具有繁琐存储细节的存储抽象，一般有专门的集群管理人员定义、维护。

根据 PV 的创建方式可以将存储卷分为动态存储和静态存储卷：

- 静态存储卷：由管理员创建的 PV
- 动态存储卷：由 Provisioner 插件创建的 PV



##### **1）静态存储卷**

一般先由集群管理员分析集群中存储需求，并预先分配一些存储介质，同时创建对应的 PV 对象，创建好的 PV 对象等待 PVC 来消费。如果负载中定义了 PVC 需求，K8s 会通过相关规则实现 PVC 和匹配的 PV 进行绑定，这样就实现了应用对存储服务的访问能力。



##### **2）动态存储卷**

由集群管理员配置好后端的存储池，并创建相应的模板（storageclass），等到有 PVC 需要消费 PV 的时候，根据 PVC 定义的需求，并参考 storageclass 的存储细节，由 Provisioner 插件动态创建一个 PV。

两种卷的比较：

- 动态存储卷和静态存储卷最终的效果都是：Pod -> PVC -> PV 的使用路径，且对象的具体模板定义都是一致的；
- 动态存储卷和静态存储卷区别是：动态卷是插件自动创建 PV，而静态卷是集群管理员手动创建 PV。



提供动态存储卷的优势：

- 动态卷让 K8s 实现了 PV 的自动化生命周期管理，PV 的创建、删除都通过 Provisioner 完成；
- 自动化创建 PV 对象，减少了配置复杂度和系统管理员的工作量；
- 动态卷可以实现 PVC 对存储的需求容量和 Provision 出来的 PV 容量一致，实现存储容量规划最优。



##### **3）动态卷的实现流程**

当用户声明一个 PVC 时，如果在 PVC 中添加了 StorageClassName 字段，其意图为：当 PVC 在集群中找不到匹配的 PV 时，会根据 StorageClassName 的定义触发相应的 Provisioner 插件创建合适的 PV 供绑定，即创建动态数据卷；动态数据卷时由 Provisioner 插件创建的，并通过 StorageClassName 与 PVC 进行关联。

StorageClass 可译为存储类，表示为一个创建 PV 存储卷的模板；在 PVC 触发自动创建PV的过程中，即使用 StorageClass 对象中的内容进行创建。其内容包括：目标 Provisioner 名字，创建 PV 的详细参数，回收模式等配置。

StorageClasss 模板定义如下：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: alicloud-disk-topology
parameters:
  type: cloud_ssd
provisioner: diskplugin.csi.alibabacloud.com
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

- provisioner：为一个注册插件的名字，此插件实现了创建 PV 的功能；一个 StorageClass 只能定义一个Provisioner；
- parameters：表示创建数据卷的具体参数；例如这里表示创建一个 SSD 类型的云盘；
- reclaimPolicy：用来指定创建 PV 的 persistentVolumeReclaimPolicy 字段值，支持 Delete/Retain；Delete 表示动态创建的 PV，在销毁的时候也会自动销毁；Retain 表示动态创建的 PV，不会自动销毁，而是由管理员来处理；
- allowVolumeExpansion：定义由此存储类创建的 PV 是否运行动态扩容，默认为 false；是否能动态扩容是有底层存储插件来实现的，这里只是一个开关；
- volumeBindingMode：表示动态创建 PV 的时间，支持 Immediate/WaitForFirstConsumer；分别表示立即创建和延迟创建。

![image-20210104183710323](D:\学习资料\笔记\k8s\k8s图\image-20210104183710323.png)

用户创建一个 PVC 声明时，会在集群寻找合适的 PV 进行绑定，如果没有合适的 PV 与之绑定，则触发下面流程：

- Volume Provisioner 会 watch 到这个 PVC 的存在，若这个 PVC 定义了 StorageClassName，且 StorageClass 对象中定义的 Provisioner 插件是自己，Provisioner 会触发创建 PV 的流程；
- Provisioner 根据 PVC 定义的参数（Size、VolumeMode、AccessModes）以及 StorageClass 定义的参数（ReclaimPolicy、Parameters）执行 PV 创建；
- Provisioner 会在存储介质端创建数据卷（通过 API 调用，或者其他方式），完成后会创建 PV 对象；
- PV 创建完成后，实现与 PVC 的绑定；以满足后续的 Pod 启动流程。



##### **4）延迟绑定动态数据卷**

某种存储（阿里云云盘）在挂载属性上有所限制，只能将相同可用区的数据卷和 Node 节点进行挂载，不在同一个可用区不可以挂载。这种类型的存储卷通常遇到如下问题：

- 创建了 A 可用区的数据卷，但是 A 可用区的节点资源已经耗光，导致 Pod 启动无法完成挂载；
- 集群管理员在规划 PVC、PV 的时候不能确定在哪些可用区创建多个 PV 来备用。



StorageClass 中的 volumeBindingMode 字段正是用来解决此问题，如果将 volumeBindingMode 配置为 WaitForFirstConsumer 值，则表示 Provisioner 在收到 PVC Pending 的时候不会立即进行数据卷创建，而是等待这个 PVC 被 Pod 消费的时候才执行创建流程。

其实现原理是：

- Provisioner 在收到 PVC Pending 状态的时候不会立即进行数据卷创建，而是等待这个 PVC 被 Pod 消费；
- 如果有 Pod 消费此 PVC，调度器发现 PVC 是延迟绑定，则 pv 继续完成调度功能（后续会详细讲解存储调度）；且调度器会将调度结果 patch 到 PVC 的 metadata 中；
- 当 Provisioner 发现 PVC 中写入了调度信息时，会根据调度信息获取创建目标数据卷的位置信息（zone、Node），并触发 PV 的创建流程。

通过上述流程可见：延迟绑定会先让应用负载进行调度（确定有充足的资源供 pod 使用），然后再触发动态卷的创建流程，这样就避免了数据卷所在可用区没有资源的问题，也避免了存储预规划的不准确性问题。

在多可用区集群环境中，更推荐使用延迟绑定的动态卷方案，目前阿里云 ACK 集群已经支持上述配置方案。



#### **3. 使用示例**

下面给出一个 pod 消费 PVC、PV 的例子：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nas-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
  selector:
    matchLabels:
      alicloud-pvname: nas-csi-pv
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nas-csi-pv
  labels:
    alicloud-pvname: nas-csi-pv
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  flexVolume:
    driver: "alicloud/nas"
    options:
      server: "***-42ad.cn-shenzhen.extreme.nas.aliyuncs.com"
      path: "/share/nas"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-nas
  labels:
    app: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx1
        image: nginx:1.8
      - name: nginx2
        image: nginx:1.7.9
        volumeMounts:
          - name: nas-pvc
            mountPath: "/data"
      volumes:
        - name: nas-pvc
          persistentVolumeClaim:
            claimName: nas-pvc
```

模板解析：

- 此应用为 Deployment 方式编排的一个 Nginx 服务，每个 pod 包含 2 个容器：nginx1、nginx2；
- 模板中定义了 Volumes 字段，说明期望挂载数据卷给应用使用，此例中使用了 PVC 这种数据卷定义方式；
- 应用内部：将数据卷 nas-pvc 挂载到 nginx2容器的 /data 目录上；nginx1 容器并没有挂载；
- PVC（nas-pvc）定义为一个不小于 50G 容量、读写方式为 ReadWriteOnce 的存储卷需求，且对 PV 有 Label 设置的需求；
- PV（nas-csi-pv）定义为一个容量为 50G、读写方式为 ReadWriteOnce、回收模式为 Retain、类型为 Flexvolume 抽象类型的存储卷，且具有 Label 配置；

根据 PVC、PV 绑定的逻辑，此 PV 符合 PVC 消费要求，则 PVC 会和此 PV 进行绑定，并供 pod 挂载使用。



### 总结

此篇文章较为详细的讲述了容器存储的整体面貌，包括单机范围的 Docker 数据卷、和集群式的 K8s 数据卷；K8s 数据卷更多关注的时候集群级别的存储编排能力，同时也在节点上实现了具体的数据卷挂载流程。K8s 为了实现上述复杂的存储卷编排能力，其实现架构也较为复杂，下节内容我们将为您介绍 K8s 的存储架构和实现流程。





## Kubernetes 存储资源 PV、PVC 和 StorageClass



**系统环境：**

- kubernetes 版本：1.14.0

**Kubernetes 官方文档地址:**

- [PC、PVC 官方文档](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [StorageClalss 官方文档](https://kubernetes.io/docs/concepts/storage/storage-classes/)



### 一、存储机制介绍

​    在 Kubernetes 中，存储资源和计算资源(CPU、Memory)同样重要，Kubernetes 为了能让管理员方便管理集群中的存储资源，同时也为了让使用者使用存储更加方便，所以屏蔽了底层存储的实现细节，将存储抽象出两个 API 资源 `PersistentVolume` 和 `PersistentVolumeClaim` 对象来对存储进行管理。

- **PersistentVolume（持久化卷）：** `PersistentVolume` 简称 `PV`， 是对底层共享存储的一种抽象，将共享存储定义为一种资源，它属于集群级别资源，不属于任何 `Namespace`，用户使用 PV 需要通过 PVC 申请。PV 是由管理员进行创建和配置，它和具体的底层的共享存储技术的实现方式有关，比如说 Ceph、GlusterFS、NFS 等，都是通过插件机制完成与共享存储的对接，且根据不同的存储 PV 可配置参数也是不相同。
- **PersistentVolumeClaim（持久化卷声明）：** `PersistentVolumeClaim` 简称 `PVC`，是用户存储的一种声明，类似于对存储资源的申请，它属于一个 `Namespace` 中的资源，可用于向 `PV` 申请存储资源。`PVC` 和 `Pod` 比较类似，`Pod` 消耗的是 `Node` 节点资源，而 `PVC` 消耗的是 `PV` 存储资源，`Pod` 可以请求 CPU 和 Memory，而 `PVC` 可以请求特定的存储空间和访问模式。

![image-20210113170554728](D:\学习资料\笔记\k8s\k8s图\image-20210113170554728.png)

​    上面两种资源 `PV` 和 `PVC` 的存在很好的解决了存储管理的问题，不过这些存储每次都需要管理员手动创建和管理，如果一个集群中有很多应用，并且每个应用都要挂载很多存储，那么就需要创建很多 `PV` 和 `PVC` 与应用关联。为了解决这个问题 Kubernetes 在 1.4 版本中引入了 `StorageClass` 对象。

​    当我们创建 `PVC` 时指定对应的 `StorageClass` 就能和 `StorageClass` 关联，`StorageClass` 会交由与他关联 Provisioner 存储插件来创建与管理存储，它能帮你创建对应的 `PV` 和在远程存储上创建对应的文件夹，并且还能根据设定的参数，删除与保留数据。所以管理员只要在 `StorageClass` 中配置好对应的参数就能方便的管理集群中的存储资源。



### 二、PersistentVolume 详解

#### 1、PV 支持存储的类型

PersistentVolume 类型实现为插件,目前 Kubernetes 支持以下插件：

- RBD：Ceph 块存储。
- FC：光纤存储设备。
- NFS：网络问卷存储卷。
- iSCSI：iSCSI 存储设备。
- CephFS：开源共享存储系统。
- Flocker：一种开源共享存储系统。
- Glusterfs：一种开源共享存储系统。
- Flexvolume：一种插件式的存储机制。
- HostPath：宿主机目录，仅能用于单机。
- AzureFile：Azure 公有云提供的 File。
- AzureDisk：Azure 公有云提供的 Disk。
- ScaleIO Volumes：DellEMC 的存储设备。
- StorageOS：StorageOS 提供的存储服务。
- VsphereVolume：VMWare 提供的存储系统。
- Quobyte Volumes：Quobyte 提供的存储服务。
- Portworx Volumes：Portworx 提供的存储服务。
- GCEPersistentDisk：GCE 公有云提供的 PersistentDisk。
- AWSElasticBlockStore：AWS 公有云提供的 ElasticBlockStore。



#### 2、PV 的生命周期

PV 生命周期总共四个阶段：

- **Available：** 可用状态，尚未被 PVC 绑定。
- **Bound：** 绑定状态，已经与某个 PVC 绑定。
- **Failed：** 当删除 PVC 清理资源，自动回收卷时失败，所以处于故障状态。
- **Released：** 与之绑定的 PVC 已经被删除，但资源尚未被集群回收。



#### 3、基于 NFS 的 PV 示例

Kubernetes 支持多种存储，这里使用最广泛的 NFS 存储为例来介绍，下面是一个 PV 的例子：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv1
  label:
    app: example
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: slow
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /nfs/space
    server: 192.168.2.11
```



#### 4、PV 的常用配置参数

##### （1）、存储能力 （capacity）

PV 可以通过配置 `capacity` 中的 `storage` 参数，对 PV 挂多大存储空间进行设置

> 目前 capacity 只有一个设置存储大小的选项，未来可能会增加。

```yaml
capacity:
    storage: 5Gi
```



##### （2）、存储卷模式（volumeMode）

PV 可以通过配置 `volumeMode` 参数，对存储卷类型进行设置，可选项包括：

- **Filesystem：** 文件系统，默认是此选项。
- **Block：** 块设备

> 目前 Block 模式只有 AWSElasticBlockStore、AzureDisk、FC、GCEPersistentDisk、iSCSI、LocalVolume、RBD、VsphereVolume 等支持）。

```yaml
volumeMode: Filesystem
```



##### （3）、访问模式（accessModes）

PV 可以通过配置 `accessModes` 参数，设置访问模式来限制应用对资源的访问权限，有以下机制访问模式：

- **ReadWriteOnce（RWO）：** 读写权限，只能被单个节点挂载。
- **ReadOnlyMany（ROX）：** 只读权限，允许被多个节点挂载读。
- **ReadWriteMany（RWX）：** 读写权限，允许被多个节点挂载。

```yaml
accessModes:
  - ReadWriteOnce
```

不过不同的存储所支持的访问模式也不相同，具体如下：

| Volume Plugin        | ReadWriteOnce | ReadOnlyMany | ReadWriteMany |
| :------------------- | :------------ | :----------- | :------------ |
| AWSElasticBlockStore | √             | -            | -             |
| AzureFile            | √             | √            | √             |
| AzureDisk            | √             | -            | -             |
| CephFS               | √             | √            | √             |
| Cinder               | √             | -            | -             |
| FC                   | √             | √            | -             |
| FlexVolume           | √             | √            | -             |
| Flocker              | √             | -            | -             |
| GCEPersistentDisk    | √             | √            | -             |
| GlusteFS             | √             | √            | √             |
| HostPath             | √             | -            | -             |
| iSCSI                | √             | √            | -             |
| PhotonPersistentDisk | √             | -            | -             |
| Quobyte              | √             | √            | √             |
| NFS                  | √             | √            | √             |
| RBD                  | √             | √            | -             |
| VsphereVolume        | √             | -            | -             |
| PortworxVolume       | √             | -            | √             |
| ScaleIO              | √             | √            | -             |
| StorageOS            | √             | -            | -             |



##### （4）、挂载参数（mountOptions）

PV 可以根据不同的存储卷类型，设置不同的挂载参数，每种类型的存储卷可配置参数都不相同。如 NFS 存储，可以设置 NFS 挂载配置，如下：

> 下面例子只是 NFS 支持的部分参数，其它参数请自行查找 NFS 挂载参数。

```yaml
mountOptions:
  - hard
  - nfsvers=4.1
```



##### （5）、存储类 （storageClassName）

PV 可以通过配置 `storageClassName` 参数指定一个存储类 `StorageClass` 资源，具有特定 `StorageClass` 的 `PV` 只能与指定相同 `StorageClass` 的 `PVC` 进行绑定，没有设置 `StorageClass` 的 `PV` 也是同样只能与没有指定 `StorageClass` 的 `PVC` 绑定。

```yaml
storageClassName: slow
```



##### （6）、回收策略（persistentVolumeReclaimPolicy）

PV 可以通过配置 `persistentVolumeReclaimPolicy` 参数设置回收策略，可选项如下：

- **Retain（保留）：** 保留数据，需要由管理员手动清理。
- **Recycle（回收）：** 删除数据，即删除目录下的所有文件，比如说执行 `rm -rf /thevolume/*` 命令，目前只有 NFS 和 HostPath 支持。
- **Delete（删除）：** 删除存储资源，仅仅部分云存储系统支持，比如删除 AWS EBS 卷，目前只有 AWS EBS，GCE PD，Azure 磁盘和 Cinder 卷支持删除。

```yaml
persistentVolumeReclaimPolicy: Recycle
```



### 三、PersistentVolumeClaim 详解

#### 1、PVC 示例

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc1
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 8Gi
  storageClassName: slow
  selector:
    matchLabels:
      release: "stable"
    matchExpressions:
      - key: environment
        operator: In
        values: dev
```



#### 2、PVC 的常用配置参数

##### （1）、筛选器（selector）

PVC 可以通过在 `Selecter` 中设置 `Laberl` 标签，筛选出带有指定 `Label` 的 `PV` 进行绑定。`Selecter` 中可以指定 `matchLabels` 或 `matchExpressions`，如果两个字段都设定了就需要同时满足才能匹配。

```yaml
selector:
  matchLabels:
    release: "stable"
  matchExpressions:
    - key: environment
      operator: In
      values: dev
```



##### （2）、资源请求（resources）

PVC 设置目前只有 `requests.storage` 一个参数，用于指定申请存储空间的大小。

```yaml
resources:
  requests:
    storage: 8Gi
```



##### （3）、存储类（storageClass）

PVC 要想绑定带有特定 `StorageClass` 的 `PV` 时，也必须设定 `storageClassName` 参数，且名称也必须要和 `PV` 中的 `storageClassName` 保持一致。如果要绑定的 `PV` 没有设置 `storageClassName` 则 `PVC` 中也不需要设置。

当 PVC 中如果未指定 `storageClassName` 参数或者指定为空值，则还需要考虑 `Kubernetes` 中是否设置了默认的 `StorageClass`：

- 未启用 DefaultStorageClass：等于 storageClassName 值为空。
- 启用 DefaultStorageClass：等于 storageClassName 值为默认的 StorageClass。

> 如果设置 storageClassName=““，则表示该 PVC 不指定 StorageClass。

```yaml
storageClassName: slow
```



##### （4）、访问模式（accessModes）

PVC 中可设置的访问模式与 PV 种一样，用于限制应用对资源的访问权限。



##### （5）、存储卷模式（volumeMode）

PVC 中可设置的存储卷模式与 PV 种一样，分为 `Filesystem` 和 `Block` 两种。



### 四、StorageClass 详解



#### 1、StorageClass 示例

这里使用 NFS 存储，创建 StorageClass 示例：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage
  annotations:
    storageclass.kubernetes.io/is-default-class: "true" #设置为默认的 StorageClass
provisioner: nfs-client
mountOptions: 
  - hard
  - nfsvers=4
parameters:
  archiveOnDelete: "true"
```



#### 2、StorageClass 的常用配置参数

##### （1）、提供者（provisioner）

在创建 `StorageClass` 之前需要 `Kubernetes` 集群中存在 `Provisioner`（存储分配提供者）应用，如 NFS 存储需要有 `NFS-Provisioner` （NFS 存储分配提供者）应用，如果集群中没有该应用，那么创建的 `StorageClass` 只能作为标记,而不能提供创建 `PV` 的作用。

> NFS 的存储 NFS Provisioner 可以参考：[创建 NFS-Provisioner 博文](http://www.mydlq.club/article/24/)

```yaml
provisioner: nfs-client
```



##### （2）、参数（parameters）

后端存储提供的参数，不同的 Provisioner 可与配置的参数也是不相同。例如 NFS Provisioner 可与提供如下参数：

```yaml
parameters:
  archiveOnDelete: "true" #删除 PV 后是否保留数据
```



##### （3）、挂载参数（mountOptions）

在 `StorageClass` 中，可以根据不同的存储来指定不同的挂载参数，此参数会与 `StorageClass` 绑定的 `Provisioner` 创建 `PV` 时，将此挂载参数与创建的 `PV` 关联。

```yaml
mountOptions: 
  - hard
  - nfsvers=4
```



##### （4）、设置默认的 StorageClass

可与在 Kubernetes 集群中设置一个默认的 StorageClass，这样当创建 PVC 时如果未指定 StorageClass 则会使用默认的 StorageClass。

```yaml
metadata:
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
```



### 五、PV 和 PVC 的生命周期

PV 是 Kubernetes 集群的存储资源，而 PVC 则是对存储资源的需求，创建 PVC 需要对 PV 发起使用申请，即和 PV 进行绑定。 PV 和 PVC 是一一对应的关系，它们二者的交互遵循如下生命周期：

![image-20210113170927051](D:\学习资料\笔记\k8s\k8s图\image-20210113170927051.png)



#### 1、存储供给

存储供给（Provisioning）是指为 PVC 准备可用的 PV 的一种机制。Kubernetes 支持 PV 供给方式有 `静态供给` 和 `动态供给` 两种：

- **静态供给：** 指由集群管理员手动创建一定数量的 PV，创建 PV 时需要根据后端存储的不同，配置的参数也不同。
- **动态供给：** 指不需要集群管理员手动创建 PV，将 PV 的创建工作交由 StorageClass 关联的 Provisioner 进行创建，会根据存储的不同自动配置相关的参数。创建完成 PV 后系统会自动将 PVC 与其绑定。



#### 2、存储绑定

​    在静态模式下，在用户定义好 PVC 后，Kubernetes 将根据 PVC 提出的“申请空间的大小”、“访问模式”从集群中寻找已经存在且满足条件的 PV 进行绑定，如果集群中没有匹配的 PV 则 PVC 将处于 Pending 等待状态，知道系统创建了符合条件的 PV 再与其绑定。PV 与 PVC 绑定后就不能和别的 PVC 进行绑定。

​    在动态模式下，当创建 PVC 并且指定 StorageClass 后，与 StorageClass 关联的存储插件会自动创建对应的 PV 与该 PVC 进行绑定。



#### 3、存储回收

完成存储卷的使用目标之后删除 PVC 对象，以便进行资源回收。不过，至于如何操作则取决于 PV 的回收策略 ，目前有三种策略：

- **保存策略（Retain）：** 删除 PVC 之后，Kubernetes 系统不会自动删除 PV ，而仅是将它标识为 `Released` 状态，不过处于 `Released` 状态的 PV **不能被其他 PVC 申请所绑定，因为之前 PVC 绑定的应用数据仍然存在，需要由管理员手动清理数据，然后决定如何处理 PV 的使用。
- **回收策略：** 当 PVC 删除后，此 PV 变成 `Available` 可用状态。不过此策略需要后端存储插件的支持。
- **删除策略：** 当删除 PVC 后 PV 和存储中的数据会被立即删除，不过此策略也需要后端存储插件的支持。



### 六、PVC 使用示例

#### Deployment 中使用 PVC

一般 `Deployment` 中使用 PVC，大部分都是静态供给方式创建，即先创建 PV，再创建 PVC 与 PV 绑定，在设置应用于 PVC 关联。

下面是一个 NFS 存储创建 PV 的例子，如下：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nginx-pv
  labels:
    app: nginx-pv
spec:
  accessModes:       
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  capacity:          
    storage: 1Gi
  mountOptions:
  - hard  
  - nfsvers=4.1  
  nfs:
    server: 192.168.2.11
    path: /nfs/data/nginx
```

创建 PVC 与 PV 进行关联绑定：

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: nginx-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  selector:
    matchLabels:
      app: nginx-pv
```

创建应用于 PVC 进行关联：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      terminationGracePeriodSeconds: 1
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - name: server
          containerPort: 80
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: nginx-pvc
```



#### StatefulSet 中使用 PVC

在有状态的应用中，我们经常使用动态供给方式创建 PV 和 PVC，不过提前需要集群拥有:

- StorageClass：存储类
- Provisioner：与存储类关联的管理后端存储的插件

只有拥有上面两种资源同时存在时才能使用动态存储，本人这里使用的是 `NFS` 存储，关于如何创建 NFS Provisioner 可以查看 [NFS Provisioner 一文](http://www.mydlq.club/article/24/)，假如 Kubernetes 集群中使用 NFS 存储，且存在 `Provisioner` 的名称为 `nfs-client` 那就可以下面创建 `StorageClass` 示例：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: nfs-client
parameters:
  archiveOnDelete: "true"
mountOptions: 
  - hard
  - nfsvers=4
```

然后 StatefulSet 可以按下方式，在 `volumeClaimTemplates` 参数中指定使用的 `StorageClass`，然后与 `StorageClass` 关联的 `NFS Provisioner` 会执行创建 PVC 和 PV，然后两者进行绑定，下面是 `StatefulSet` 方式使用 `volumeClaimTemplates`挂载存储的示例：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx
  serviceName: "nginx"
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      terminationGracePeriodSeconds: 1
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "nfs-storage"
      resources:
        requests:
          storage: 1Gi
```



















































## Kubernetes 中部署 NFS Provisioner 为 NFS 提供动态分配卷



**环境说明：**

- NFS Provisioner Github 地址：https://github.com/kubernetes-incubator/external-storage/tree/master/nfs-client
- NFS 示例部署文件 Github 地址：https://github.com/my-dlq/blog-example/tree/master/kubernetes/nfs-provisioner-deploy



### 一、NFS Provisioner 简介

NFS Provisioner 是一个自动配置卷程序，它使用现有的和已配置的 NFS 服务器来支持通过持久卷声明动态配置 Kubernetes 持久卷。

- 持久卷被配置为：`${namespace}-${pvcName}-${pvName}`。



### 二、创建 NFS Server 端

本篇幅是具体介绍如何部署 NFS 动态卷分配应用 “NFS Provisioner”，所以部署前请确认已经存在 NFS Server 端，关于如何部署 NFS Server 请看之前写过的博文 [“CentOS7 搭建 NFS 服务器”](http://www.mydlq.club/article/3/)，如果非 Centos 系统，请先自行查找 NFS Server 安装方法。

**这里 NFS Server 环境为：**

- IP地址：192.168.2.11
- 存储目录：/nfs/data



### 三、创建 ServiceAccount

现在的 Kubernetes 集群大部分是基于 RBAC 的权限控制，所以创建一个一定权限的 ServiceAccount 与后面要创建的 “NFS Provisioner” 绑定，赋予一定的权限。

> 提前修改里面的 Namespace 值设置为要部署 “NFS Provisioner” 的 Namespace 名

**nfs-rbac.yaml**

```yaml
kind: ServiceAccount
apiVersion: v1
metadata:
  name: nfs-client-provisioner
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: kube-system      #替换成你要部署NFS Provisioner的 Namespace
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: kube-system      #替换成你要部署NFS Provisioner的 Namespace
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
```

**创建 RBAC**

- -n: 指定应用部署的 Namespace

```bash
$ kubectl apply -f nfs-rbac.yaml -n kube-system
```



### 四、部署 NFS Provisioner

设置 NFS Provisioner 部署文件，这里将其部署到 “kube-system” Namespace 中。

**nfs-provisioner-deploy.yaml**

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: nfs-client-provisioner
  labels:
    app: nfs-client-provisioner
spec:
  replicas: 1
  strategy:
    type: Recreate      #---设置升级策略为删除再创建(默认为滚动更新)
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: quay.io/external_storage/nfs-client-provisioner:v3.1.0-k8s1.11
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: nfs-client     #--- nfs-provisioner的名称，以后设置的storageclass要和这个保持一致
            - name: NFS_SERVER
              value: 192.168.2.11   #---NFS服务器地址，和 valumes 保持一致
            - name: NFS_PATH
              value: /nfs/data      #---NFS服务器目录，和 valumes 保持一致
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.2.11    #---NFS服务器地址
            path: /nfs/data         #---NFS服务器目录
```

**创建 NFS Provisioner**

- -n: 指定应用部署的 Namespace

```bash
$ kubectl apply -f nfs-provisioner-deploy.yaml -n kube-system
```



### 五、创建 NFS SotageClass

创建一个 StoageClass，声明 NFS 动态卷提供者名称为 “nfs-storage”。

**nfs-storage.yaml**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage
  annotations:
    storageclass.kubernetes.io/is-default-class: "true" #---设置为默认的storageclass
provisioner: nfs-client    #---动态卷分配者名称，必须和上面创建的"provisioner"变量中设置的Name一致
parameters:
  archiveOnDelete: "true"  #---设置为"false"时删除PVC不会保留数据,"true"则保留数据
mountOptions: 
  - hard        #指定为硬挂载方式
  - nfsvers=4   #指定NFS版本，这个需要根据 NFS Server 版本号设置
```

**创建 StorageClass**

```bash
$ kubectl apply -f nfs-storage.yaml
```



### 六、创建 PVC 和 Pod 进行测试

#### 1、创建测试 PVC

在 “kube-public” Namespace 下创建一个测试用的 PVC 并观察是否自动创建是 PV 与其绑定。

**test-pvc.yaml**

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-pvc
spec:
  storageClassName: nfs-storage #---需要与上面创建的storageclass的名称一致
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Mi
```

**创建 PVC**

- -n：指定创建 PVC 的 Namespace

```bash
$ kubectl apply -f test-pvc.yaml -n kube-public
```

**查看 PVC 状态是否与 PV 绑定**

利用 Kubectl 命令获取 pvc 资源，查看 STATUS 状态是否为 “Bound”。

```bash
$ kubectl get pvc test-pvc -n kube-public

NAME       STATUS   VOLUME                 CAPACITY   ACCESS MODES   STORAGECLASS
test-pvc   Bound    pvc-be0808c2-9957-11e9 1Mi        RWO            nfs-storage
```



#### 2、创建测试 Pod 并绑定 PVC

创建一个测试用的 Pod，指定存储为上面创建的 PVC，然后创建一个文件在挂载的 PVC 目录中，然后进入 NFS 服务器下查看该文件是否存入其中。

**test-pod.yaml**

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: test-pod
spec:
  containers:
  - name: test-pod
    image: busybox:latest
    command:
      - "/bin/sh"
    args:
      - "-c"
      - "touch /mnt/SUCCESS && exit 0 || exit 1"  #创建一个名称为"SUCCESS"的文件
    volumeMounts:
      - name: nfs-pvc
        mountPath: "/mnt"
  restartPolicy: "Never"
  volumes:
    - name: nfs-pvc
      persistentVolumeClaim:
        claimName: test-pvc
```

**创建 Pod**

- -n：指定创建 Pod 的 Namespace

```bash
$ kubectl apply -f test-pod.yaml -n kube-public
```



#### 3、进入 NFS Server 服务器验证是否创建对应文件

进入 NFS Server 服务器的 NFS 挂载目录，查看是否存在 Pod 中创建的文件：

```bash
$ cd /nfs/data
$ ls -l

-rw-r--r-- kube-public-test-pvc-pvc-3dc54156-b81d-11e9-a8b8-000c29d98697

$ cd /kube-public-test-pvc-pvc-3dc54156-b81d-11e9-a8b8-000c29d98697
$ ls -l

-rw-r--r-- 1 root root 0 Jun 28 12:44 SUCCESS
```

可以看到已经生成 SUCCESS 该文件，并且可知通过 NFS Provisioner 创建的目录命名方式为 “`namespace名称`-`pvc名称`-`pv名称`”，pv 名称是随机字符串，所以每次只要不删除 PVC，那么 Kubernetes 中的与存储绑定将不会丢失，要是删除 PVC 也就意味着删除了绑定的文件夹，下次就算重新创建相同名称的 PVC，生成的文件夹名称也不会一致，因为 PV 名是随机生成的字符串，而文件夹命名又跟 PV 有关,所以删除 PVC 需谨慎。





## Kubernetes 使用 ceph-csi 消费 RBD 作为持久化存储



### 前言

本文详细介绍了如何在 Kubernetes 集群中部署 `ceph-csi`（v3.1.0），并使用 `RBD` 作为持久化存储。

需要的环境参考下图：

![image-20210111210906238](D:\学习资料\笔记\k8s\k8s图\image-20210111210906238.png)



**本文使用的环境版本信息：**

Kubernetes 版本：

```sh
$ kubectl get node
NAME       STATUS   ROLES    AGE   VERSION
sealos01   Ready    master   23d   v1.18.8
sealos02   Ready    master   23d   v1.18.8
sealos03   Ready    master   23d   v1.18.8
```

Ceph 版本：

```sh
$ ceph version
ceph version 14.2.11 (f7fdb2f52131f54b891a2ec99d8205561242cdaf) nautilus (stable)
```

以下是详细部署过程：



### 1. 新建 Ceph Pool

创建一个新的 ceph 存储池（pool） 给 Kubernetes 使用：

```sh
$ ceph osd pool create kubernetes
pool ' kubernetes' created
```

查看所有的 `pool`：

```sh
$ ceph osd lspools
1 cephfs_data
2 cephfs_metadata
3 .rgw.root
4 default.rgw.control
5 default.rgw.meta
6 default.rgw.log
7 kubernetes
```



### 2. 新建用户

为 Kubernetes 和 ceph-csi 单独创建一个新用户：

```sh
$ ceph auth get-or-create client.kubernetes mon 'profile rbd' osd 'profile rbd pool=kubernetes' mgr 'profile rbd pool=kubernetes'

[client.kubernetes]
    key = AQBnz11fclrxChAAf8TFw8ROzmr8ifftAHQbTw==
```

后面的配置需要用到这里的 key，如果忘了可以通过以下命令来获取：

```sh
$ ceph auth get client.kubernetes
exported keyring for client.kubernetes
[client.kubernetes]
 key = AQBnz11fclrxChAAf8TFw8ROzmr8ifftAHQbTw==
 caps mgr = "profile rbd pool=kubernetes"
 caps mon = "profile rbd"
 caps osd = "profile rbd pool=kubernetes"
```



### 3. 部署 ceph-csi

拉取 ceph-csi 的**最新 release 分支（v3.1.0）**[1]：

```sh
$ git clone --depth 1 --branch v3.1.0 https://gitclone.com/github.com/ceph/ceph-csi
```

- 这里使用 **gitclone**[2] 来加速拉取。

#### 修改 Configmap

获取 `Ceph` 集群的信息：

```sh
$ ceph mon dump

dumped monmap epoch 1
epoch 1
fsid 154c3d17-a9af-4f52-b83e-0fddd5db6e1b
last_changed 2020-09-12 16:16:53.774567
created 2020-09-12 16:16:53.774567
min_mon_release 14 (nautilus)
0: [v2:172.16.1.21:3300/0,v1:172.16.1.21:6789/0] mon.sealos01
1: [v2:172.16.1.22:3300/0,v1:172.16.1.22:6789/0] mon.sealos02
2: [v2:172.16.1.23:3300/0,v1:172.16.1.23:6789/0] mon.sealos03
```

这里需要用到两个信息：

- **fsid** : 这个是 Ceph 的集群 ID。
- 监控节点信息。目前 ceph-csi 只支持 `v1` 版本的协议，所以监控节点那里我们只能用 `v1` 的那个 IP 和端口号（例如，`172.16.1.21:6789`）。

进入 ceph-csi 的 `deploy/rbd/kubernetes` 目录：

```sh
$ cd deploy/rbd/kubernetes

$ ls -l ./
total 36
-rw-r--r-- 1 root root  100 Sep 14 04:49 csi-config-map.yaml
-rw-r--r-- 1 root root 1686 Sep 14 04:49 csi-nodeplugin-psp.yaml
-rw-r--r-- 1 root root  858 Sep 14 04:49 csi-nodeplugin-rbac.yaml
-rw-r--r-- 1 root root 1312 Sep 14 04:49 csi-provisioner-psp.yaml
-rw-r--r-- 1 root root 3105 Sep 14 04:49 csi-provisioner-rbac.yaml
-rw-r--r-- 1 root root 5497 Sep 14 04:49 csi-rbdplugin-provisioner.yaml
-rw-r--r-- 1 root root 5852 Sep 14 04:49 csi-rbdplugin.yaml
```

将以上获取的信息写入 `csi-config-map.yaml`：

```yaml
---
apiVersion: v1
kind: ConfigMap
data:
  config.json: |-
    [
      {
        "clusterID": "154c3d17-a9af-4f52-b83e-0fddd5db6e1b",
        "monitors": [
          "172.16.1.21:6789",
          "172.15.1.22:6789",
          "172.16.1.23:6789"
        ]
      }
    ]
metadata:
  name: ceph-csi-config
```

创建一个新的 namespace 专门用来部署 ceph-csi：

```sh
$ kubectl create ns ceph-csi
```

将此 Configmap 存储到 Kubernetes 集群中：

```sh
$ kubectl -n ceph-csi apply -f csi-config-map.yaml
```



#### 新建 Secret

使用创建的 kubernetes 用户 ID 和 `cephx` 密钥生成 `Secret`：

```sh
$ cat <<EOF > csi-rbd-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: csi-rbd-secret
  namespace: ceph-csi
stringData:
  userID: kubernetes
  userKey: AQBnz11fclrxChAAf8TFw8ROzmr8ifftAHQbTw==
EOF
```

部署 Secret：

```sh
$ kubectl apply -f csi-rbd-secret.yaml
```



#### RBAC 授权

将所有配置清单中的 `namespace` 改成 `ceph-csi`：

```sh
$ sed -i "s/namespace: default/namespace: ceph-csi/g" $(grep -rl "namespace: default" ./)
$ sed -i -e "/^kind: ServiceAccount/{N;N;a\  namespace: ceph-csi  # 输入到这里的时候需要按一下回车键，在下一行继续输入
  }" $(egrep -rl "^kind: ServiceAccount" ./)
```

创建必须的 `ServiceAccount` 和 RBAC ClusterRole/ClusterRoleBinding 资源对象：

```sh
$ kubectl create -f csi-provisioner-rbac.yaml
$ kubectl create -f csi-nodeplugin-rbac.yaml
```

创建 PodSecurityPolicy：

```sh
$ kubectl create -f csi-provisioner-psp.yaml
$ kubectl create -f csi-nodeplugin-psp.yaml
```



#### 部署 CSI sidecar

将 `csi-rbdplugin-provisioner.yaml` 和 `csi-rbdplugin.yaml` 中的 kms 部分配置：

![image-20210111212414655](D:\学习资料\笔记\k8s\image-20210111212414655.png)

![image-20210111212430047](D:\学习资料\笔记\k8s\image-20210111212430047.png)

部署 `csi-rbdplugin-provisioner`：

```sh
$ kubectl -n ceph-csi create -f csi-rbdplugin-provisioner.yaml
```

这里面包含了 6 个 Sidecar 容器，包括 `external-provisioner`、`external-attacher`、`csi-resizer` 和 `csi-rbdplugin`。



#### 部署 RBD CSI driver

最后部署 `RBD CSI Driver`：

```
$ kubectl -n ceph-csi create -f csi-rbdplugin.yaml
```

Pod 中包含两个容器：`CSI node-driver-registrar` 和 `CSI RBD driver`。



#### 创建 Storageclass

```
$ cat <<EOF > storageclass.yaml
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: csi-rbd-sc
provisioner: rbd.csi.ceph.com
parameters:
   clusterID: 154c3d17-a9af-4f52-b83e-0fddd5db6e1b
   pool: kubernetes
   imageFeatures: layering
   csi.storage.k8s.io/provisioner-secret-name: csi-rbd-secret
   csi.storage.k8s.io/provisioner-secret-namespace: ceph-csi
   csi.storage.k8s.io/controller-expand-secret-name: csi-rbd-secret
   csi.storage.k8s.io/controller-expand-secret-namespace: ceph-csi
   csi.storage.k8s.io/node-stage-secret-name: csi-rbd-secret
   csi.storage.k8s.io/node-stage-secret-namespace: ceph-csi
   csi.storage.k8s.io/fstype: ext4
reclaimPolicy: Delete
allowVolumeExpansion: true
mountOptions:
   - discard
EOF
```

- 这里的 `clusterID` 对应之前步骤中的 `fsid`。
- `imageFeatures` 用来确定创建的 image 特征，如果不指定，就会使用 RBD 内核中的特征列表，但 Linux 不一定支持所有特征，所以这里需要限制一下。



### 4. 试用 ceph-csi

Kubernetes 通过 `PersistentVolume` 子系统为用户和管理员提供了一组 API，将存储如何供应的细节从其如何被使用中抽象出来，其中 `PV`（PersistentVolume） 是实际的存储，`PVC`（PersistentVolumeClaim） 是用户对存储的请求。

下面通过官方仓库的示例来演示如何使用 ceph-csi。

先进入 ceph-csi 项目的 `example/rbd` 目录，然后直接创建 PVC：

```sh
$ kubectl apply -f pvc.yaml
```

查看 PVC 和申请成功的 PV：

```sh
$ kubectl get pvc
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
rbd-pvc   Bound    pvc-44b89f0e-4efd-4396-9316-10a04d289d7f   1Gi        RWO            csi-rbd-sc     8m21s

$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                STORAGECLASS   REASON   AGE
pvc-44b89f0e-4efd-4396-9316-10a04d289d7f   1Gi        RWO            Delete           Bound    default/rbd-pvc      csi-rbd-sc              8m18s
```

再创建示例 Pod：

```sh
$ kubectl apply -f pod.yaml
```

进入 Pod 里面测试读写数据：

```sh
$ kubectl exec -it csi-rbd-demo-pod bash
root@csi-rbd-demo-pod:/# cd /var/lib/www/
root@csi-rbd-demo-pod:/var/lib/www# ls -l
total 4
drwxrwxrwx 3 root root 4096 Sep 14 09:09 html
root@csi-rbd-demo-pod:/var/lib/www# echo "https://fuckcloudnative.io" > sealos.txt
root@csi-rbd-demo-pod:/var/lib/www# cat sealos.txt
https://fuckcloudnative.io
```

列出 kubernetes `pool` 中的 rbd `images`：

```sh
$ rbd ls -p kubernetes
csi-vol-d9d011f9-f669-11ea-a3fa-ee21730897e6
```

查看该 image 的特征：

```sh
$ rbd info csi-vol-d9d011f9-f669-11ea-a3fa-ee21730897e6 -p kubernetes
rbd image 'csi-vol-d9d011f9-f669-11ea-a3fa-ee21730897e6':
 size 1 GiB in 256 objects
 order 22 (4 MiB objects)
 snapshot_count: 0
 id: 8da46585bb36
 block_name_prefix: rbd_data.8da46585bb36
 format: 2
 features: layering
 op_features:
 flags:
 create_timestamp: Mon Sep 14 09:08:27 2020
 access_timestamp: Mon Sep 14 09:08:27 2020
 modify_timestamp: Mon Sep 14 09:08:27 2020
```

可以看到对 image 的特征限制生效了，这里只有 `layering`。

实际上这个 `image` 会被挂载到 node 中作为一个块设备，可以通过 `rbd` 命令查看映射信息：

```sh
$ rbd showmapped
id pool       namespace image                                        snap device
0  kubernetes           csi-vol-d9d011f9-f669-11ea-a3fa-ee21730897e6 -    /dev/rbd0
```

在 node 上查看挂载信息：

```sh
$ lsblk -l|grep rbd
rbd0                                                                                               252:32   0     1G  0 disk /var/lib/kubelet/pods/15179e76-e06e-4c0e-91dc-e6ecf2119f4b/volumes/kubernetes.io~csi/pvc-44b89f0e-4efd-4396-9316-10a04d289d7f/mount
```

在 容器中查看挂载信息：

```sh
$ kubectl exec -it csi-rbd-demo-pod bash
root@csi-rbd-demo-pod:/# lsblk -l|grep rbd
rbd0                                                                                               252:32   0     1G  0 disk /var/lib/www/html
```

一切正常！



### 参考资料

[1]最新 release 分支（v3.1.0）: *https://github.com/ceph/ceph-csi/tree/v3.1.0*[2]gitclone: *https://gitclone.com*





## 通过 Helm 搭建 Docker 镜像仓库 Harbor



**参数地址：**

- [在 Kubernetes 在快速安装 Harbor](https://www.qikqiak.com/post/harbor-quick-install/)
- [Harbor-Helm 的 Github](https://github.com/goharbor/harbor-helm)

**系统环境：**

- kubernetes 版本：1.18.5
- Traefik Ingress 版本：2.2.8
- Harbor Chart 版本：1.4.2
- Harbor 版本：2.0.2
- Helm 版本：3.2.1
- 持久化存储驱动：NFS



### 一、Harbor 简介

#### 1、简介

​    Harbor 是一个开放源代码容器镜像注册表，可通过基于角色权限的访问控制来管理镜像，还能扫描镜像中的漏洞并将映像签名为受信任。Harbor 是 CNCF 孵化项目，可提供合规性，性能和互操作性，以帮助跨 Kubernetes 和 Docker 等云原生计算平台持续，安全地管理镜像。



#### 2、特性

- 管理：多租户、可扩展
- 安全：安全和漏洞分析、内容签名与验证



### 二、准备环境

#### 1、安装 Helm

关于如何安装 Helm 3，请查看之前的博文 [安装 Helm3 管理 Kubernetes 应用](http://www.mydlq.club/article/51/) 进行安装。



#### 2、创建 Namespace

由于 Harbor 组件较多，一般我们会采取新建一个 Namespace 专用于部署 Harbor 相关组件，输入下面命令创建名为 `mydlq-hub` 的命名空间。

```bash
$ kubectl create namespace mydlq-hub
```



#### 3、挂载 NFS 与创建目录

> 这里使用的是 NFS 存储驱动，如果使用其他存储驱动，请自行配置。

```bash
#挂载 NFS
$ mount -o vers=4.1 192.168.2.11:/nfs/ /nfs

#创建文件夹
mkdir -p /nfs/harbor/registry
mkdir -p /nfs/harbor/chartmuseum
mkdir -p /nfs/harbor/jobservice
mkdir -p /nfs/harbor/database
mkdir -p /nfs/harbor/redis
mkdir -p /nfs/harbor/trivy
```



#### 4、创建 PV 与 PVC

**(1)、创建 PV 部署文件 harbor-pv.yaml**

harbor-pv.yaml

```yaml
#registry-PV
apiVersion: v1
kind: PersistentVolume
metadata:
  name: harbor-registry
  labels:
    app: harbor-registry
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /nfs/harbor/registry
    server: 192.168.2.11
---
#harbor-chartmuseum-pv
apiVersion: v1
kind: PersistentVolume
metadata:
  name: harbor-chartmuseum
  labels:
    app: harbor-chartmuseum
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /nfs/harbor/chartmuseum
    server: 192.168.2.11
---
#harbor-jobservice-pv
apiVersion: v1
kind: PersistentVolume
metadata:
  name: harbor-jobservice
  labels:
    app: harbor-jobservice
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /nfs/harbor/jobservice
    server: 192.168.2.11
---
#harbor-database-pv
apiVersion: v1
kind: PersistentVolume
metadata:
  name: harbor-database
  labels:
    app: harbor-database
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /nfs/harbor/database
    server: 192.168.2.11
---
#harbor-redis-pv
apiVersion: v1
kind: PersistentVolume
metadata:
  name: harbor-redis
  labels:
    app: harbor-redis
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /nfs/harbor/redis
    server: 192.168.2.11
---
#harbor-trivy-pv
apiVersion: v1
kind: PersistentVolume
metadata:
  name: harbor-trivy
  labels:
    app: harbor-trivy
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /nfs/harbor/trivy
    server: 192.168.2.11
```

执行 Kuberctl 命令创建 PV 资源：

- -f：指定资源配置文件
- -n：指定创建资源的命名空间

```bash
$ kubectl apply -f harbor-pv.yaml
```

**(2)、创建 PVC 部署文件 harbor-pvc.yaml**

harbor-pvc.yaml

```yaml
#harbor-registry-pvc
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: harbor-registry
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
  selector:
    matchLabels:
      app: harbor-registry
---
#harbor-chartmuseum-pvc
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: harbor-chartmuseum
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  selector:
    matchLabels:
      app: harbor-chartmuseum
---
#harbor-jobservice-pvc
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: harbor-jobservice
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  selector:
    matchLabels:
      app: harbor-jobservice 
---
#harbor-database-pvc
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: harbor-database
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  selector:
    matchLabels:
      app: harbor-database  
---
#harbor-redis-pvc
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: harbor-redis
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  selector:
    matchLabels:
      app: harbor-redis
---
#harbor-trivy-pvc
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: harbor-trivy
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  selector:
    matchLabels:
      app: harbor-trivy
```

执行 Kuberctl 命令创建 PVC 资源：

- -f：指定要创建资源的文件
- -n：指定应用创建的命名空间

```bash
$ kubectl apply -f harbor-pvc.yaml -n mydlq-hub
```



### 三、创建自定义证书

安装 Harbor 我们会默认使用 HTTPS 协议，需要 TLS 证书，如果我们没用自己设定自定义证书文件，那么 Harbor 将自动创建证书文件，不过这个有效期只有一年时间，所以这里我们生成自签名证书，为了避免频繁修改证书，将证书有效期为 10 年，操作如下：

#### 1、生成证书文件：

下面执行步骤时，需要输入一些证书信息，其中 Common Name 必须要设置为和你要给 Harbor 的域名保持一致，如 `Common Name (eg, your name or your server's hostname) []:hub.mydlq.club`。

```bash
## 获得证书
$ openssl req -newkey rsa:4096 -nodes -sha256 -keyout ca.key -x509 -days 3650 -out ca.crt

## 生成证书签名请求
$ openssl req -newkey rsa:4096 -nodes -sha256 -keyout tls.key -out tls.csr

## 生成证书
$ openssl x509 -req -days 3650 -in tls.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out tls.crt
```



#### 2、生成 Secret 资源

创建 Kubernetes 的 Secret 资源，且将证书文件导入：

- \- n：指定创建资源的 Namespace
- --from-file：指定要导入的文件地址

```bash
$ kubectl create secret generic hub-mydlq-tls --from-file=tls.crt --from-file=tls.key --from-file=ca.crt -n mydlq-hub
```

查看是否创建成功：

```bash
$ kubectl get secret hub-mydlq-tls -n mydlq-hub
```

可以观察到：

```bash
NAME            TYPE     DATA   AGE
hub-mydlq-tls   Opaque   3      52m
```



### 四、设置 Harbor 配置清单

​    由于我们需要通过 Helm 安装 Harbor 仓库，需要提前创建 Harbor Chart 的配置清单文件，里面是对要创建的应用 Harbor 进行一系列参数配置，由于参数过多，关于都有 Harbor Chart 都能够配置哪些参数这里就不一一罗列，可以通过访问 [Harbor-helm 的 Github 地址](https://github.com/goharbor/harbor-helm) 进行了解。

下面描述下，需要的一些配置参数：

**values.yaml**

```yaml
#Ingress 网关入口配置
expose:
  type: ingress
  tls:
    ### 是否启用 https 协议，如果不想启用 HTTPS，则可以设置为 false
    enabled: true                   
  ingress:                          
    hosts:
      ### 配置 Harbor 的访问域名，需要注意的是配置 notary 域名要和 core 处第一个单词外，其余保持一致
      core: hub.mydlq.club   
      notary: notary.mydlq.club
    controller: default
    annotations:
      ingress.kubernetes.io/ssl-redirect: "true"
      ingress.kubernetes.io/proxy-body-size: "0"
      #### 如果是 traefik ingress，则按下面配置：
      kubernetes.io/ingress.class: "traefik"
      traefik.ingress.kubernetes.io/router.tls: 'true'
      traefik.ingress.kubernetes.io/router.entrypoints: websecure
      #### 如果是 nginx ingress，则按下面配置：
      #nginx.ingress.kubernetes.io/ssl-redirect: "true"
      #nginx.ingress.kubernetes.io/proxy-body-size: "0"
  ## 如果不想使用 Ingress 方式，则可以配置下面参数，配置为 NodePort                  
  #clusterIP:
  #  name: harbor
  #  ports:
  #    httpPort: 80
  #    httpsPort: 443
  #    notaryPort: 4443
  #nodePort:
  #  name: harbor
  #  ports:
  #    http:
  #      port: 80
  #      nodePort: 30011
  #    https: 
  #      port: 443
  #      nodePort: 30012
  #    notary: 
  #      port: 4443
  #      nodePort: 30013

## 如果Harbor部署在代理后，将其设置为代理的URL，这个值一般要和上面的 Ingress 配置的地址保存一致
externalURL: https://hub.mydlq.club

### Harbor 各个组件的持久化配置，并设置各个组件 existingClaim 参数为上面创建的对应 PVC 名称
persistence:
  enabled: true
  ### 存储保留策略，当PVC、PV删除后，是否保留存储数据
  resourcePolicy: "keep"    
  persistentVolumeClaim:
    registry:
      existingClaim: "harbor-registry"
      size: 100Gi
    chartmuseum:
      existingClaim: "harbor-chartmuseum"
      size: 5Gi
    jobservice:
      existingClaim: "harbor-jobservice"
      size: 5Gi
    database:
      existingClaim: "harbor-database"
      size: 5Gi
    redis:
      existingClaim: "harbor-redis"
      size: 5Gi
    trivy:
      existingClaim: "harbor-trivy"
      size: 5Gi

### 默认用户名 admin 的密码配置，注意：密码中一定要包含大小写字母与数字
harborAdminPassword: "Mydlq123456"

### 设置日志级别
logLevel: info

#各个组件 CPU & Memory 资源相关配置
nginx:
  resources:
    requests:
      memory: 256Mi
      cpu: 500m
portal:
  resources:
    requests:
      memory: 256Mi
      cpu: 500m  
core:
  resources:
    requests:
      memory: 256Mi
      cpu: 1000m
jobservice:
  resources:
    requests:
      memory: 256Mi
      cpu: 500m
registry:
  registry:
    resources:
      requests:
        memory: 256Mi
        cpu: 500m
  controller:
    resources:
      requests:
        memory: 256Mi
        cpu: 500m
clair:
  clair:
    resources:
      requests:
        memory: 256Mi
        cpu: 500m
  adapter:
    resources:
      requests:
        memory: 256Mi
        cpu: 500m
notary:
  server:
    resources:
      requests:
        memory: 256Mi
        cpu: 500m
  signer:
    resources:
      requests:
        memory: 256Mi
        cpu: 500m
database:
  internal:
    resources:
      requests:
        memory: 256Mi
        cpu: 500m
redis:
  internal:
    resources:
      requests:
        memory: 256Mi
        cpu: 500m
trivy:
  enabled: true
  resources:
    requests:
      cpu: 200m
      memory: 512Mi
    limits:
      cpu: 1000m
      memory: 1024Mi

#开启 chartmuseum，使 Harbor 能够存储 Helm 的 chart
chartmuseum:
  enabled: true
  resources:
    requests:
     memory: 256Mi
     cpu: 500m     
```



### 五、安装 Harbor

#### 1、添加 Helm 仓库

```bash
$ helm repo add harbor https://helm.goharbor.io
```



#### 2、部署 Harbor

```bash
$ helm install harbor harbor/harbor --version 1.4.2 -f values.yaml -n mydlq-hub
```



#### 3、查看应用是否部署完成

```bash
$ kubectl get deployment -n mydlq-hub

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
harbor-harbor-chartmuseum     1/1     1            1           5m
harbor-harbor-clair           1/1     1            1           5m
harbor-harbor-core            1/1     1            1           5m
harbor-harbor-jobservice      1/1     1            1           5m
harbor-harbor-notary-server   1/1     1            1           5m
harbor-harbor-notary-signer   1/1     1            1           5m
harbor-harbor-portal          1/1     1            1           5m
harbor-harbor-registry        1/1     1            1           5m
```



#### 4、Host 配置域名

​    接下来配置 Hosts，客户端想通过域名访问服务，必须要进行 DNS 解析，由于这里没有 DNS 服务器进行域名解析，所以修改 hosts 文件将 Harbor 指定节点的 IP 和自定义 host 绑定。打开电脑的 Hosts 配置文件，往其加入下面配置：

```bash
192.168.2.11  hub.mydlq.club
```



#### 5、访问 Harbor

输入地址 `https://hub.mydlq.club` 访问 Harbor 仓库。

- 用户：admin
- 密码：Mydlq123456 (在安装配置中自定义的密码)



### 六、服务器配置镜像仓库

#### 1、下载 Harbor 证书

由于 Harbor 是基于 Https 的，故而需要提前配置 tls 证书，进入：**Harobr主页->配置管理->系统配置->镜像库根证书**

![image-20210113111753439](D:\学习资料\笔记\k8s\k8s图\image-20210113111753439.png)

点击下载按钮下载证书，下载完后打开，可以看到证书文件如下：

![image-20210113111821523](D:\学习资料\笔记\k8s\k8s图\image-20210113111821523.png)



#### 2、服务器 Docker 中配置 Harbor 证书

然后进入服务器，在服务器上 `/etc/docker` 目录下创建 `certs.d` 文件夹，然后在 `certs.d` 文件夹下创建 Harobr 域名文件夹，可以输入下面命令创建对应文件夹：

```bash
$ mkdir -p /etc/docker/certs.d/hub.mydlq.club
```

然后再 `/etc/docker/certs.d/hub.mydlq.club` 目录下创建上面的 ca 证书文件：

```bash
$ cat > /etc/docker/certs.d/hub.mydlq.club/ca.crt << EOF
-----BEGIN CERTIFICATE-----
MIIC9TCCAd2gAwIBAgIRALztT/b8wlhjw50UECEOTR8wDQYJKoZIhvcNAQELBQAw
FDESMBAGA1UEAxMJaGFyYm9yLWNhMB4XDTIwMDIxOTA3NTgwMFoXDTIxMDIxODA3
NTgwMFowFDESMBAGA1UEAxMJaGFyYm9yLWNhMIIBIjANBgkqhkiG9w0BAQEFAAOC
AQ8AMIIBCgKCAQEArYbsxYmNksU5eQhVIM3OKac4l6MV/5u5belAlWSdpbbQCwMF
G/gAliTSQMgqcmhQ3odYTKImvx+5zrhP5b1CWXCQCVOlOFSLrs3ZLv68ZpKoDLkg
6XhoQFVPLM0v5V+YzWCGAson81LfX3tDhltnOItSpe2KESABVH+5L/2vo25P7Mvw
4bWEWMyY4AS/3toiDZjhwNMrMb2lpICrlH9Sc3dAOzUteyVznA5/WF8IyPI64aKn
tl0gxLOZgUBTkBoxVhPj7dNNZu8lMnqAYXmhWt+oRr7t1HHp2lOtk2u/ndyV0kKL
xufx5FYVJQel2yRBGc/C1QLN18nC1y6u5pITaQIDAQABo0IwQDAOBgNVHQ8BAf8E
BAMCAqQwHQYDVR0lBBYwFAYIKwYBBQUHAwEGCCsGAQUFBwMCMA8GA1UdEwEB/wQF
MAMBAf8wDQYJKoZIhvcNAQELBQADggEBACFT92PWBFeCT7By8y8+EkB2TD1QVMZm
NDpBS75q5s2yIumFwJrbY6YsHtRkN1Zx9jc4LiJFHC6r0ES3tbCDapsxocvzn7dW
XLNTtnSx0zPxNXZzgmTsamfunBd4gszdXMshJ+bKsEoTXhJEXVjZq/k0EZS8L4Mp
NZ7ciPqwAI1Tg+mFGp5UOvzxYLyW8nCLPykC73y3ob1tiO6xdyD/orTAbA6pIMc9
7ajTfwYj4Q6JPY/QAmu0S+4hJHs724IrC6hiXUlQNVVRW/d3k+nXbYttnnmPnQXC
RyK2ru7R8H43Zlwj26kQJo6naQoQ0+Xcjcyk5llPqJxCrk3uoHF0r4U=
-----END CERTIFICATE-----
EOF
```



#### 3、登录 Harbor 仓库

只有登录成功后才能将镜像推送到镜像仓库，所以配置完证书后尝试登录，测试是否能够登录成功：

> 如果提示 ca 证书错误，则重建检测证书配置是否有误。

```bash
$ docker login -u admin -p Mydlq123456 hub.mydlq.club
```



### 七、服务器配置 Helm Chart 仓库

#### 1、配置 Helm 证书

跟配置 Docker 仓库一样，配置 Helm 仓库也得提前配置证书，首先进入 ca 签名目录

> 如果下面执行的目录不存在，请用 yum 安装 ca-certificates 包。

```bash
$ cat > /etc/pki/ca-trust/source/anchors/ca.crt << EOF
-----BEGIN CERTIFICATE-----
MIIC9TCCAd2gAwIBAgIRALztT/b8wlhjw50UECEOTR8wDQYJKoZIhvcNAQELBQAw
FDESMBAGA1UEAxMJaGFyYm9yLWNhMB4XDTIwMDIxOTA3NTgwMFoXDTIxMDIxODA3
NTgwMFowFDESMBAGA1UEAxMJaGFyYm9yLWNhMIIBIjANBgkqhkiG9w0BAQEFAAOC
AQ8AMIIBCgKCAQEArYbsxYmNksU5eQhVIM3OKac4l6MV/5u5belAlWSdpbbQCwMF
G/gAliTSQMgqcmhQ3odYTKImvx+5zrhP5b1CWXCQCVOlOFSLrs3ZLv68ZpKoDLkg
6XhoQFVPLM0v5V+YzWCGAson81LfX3tDhltnOItSpe2KESABVH+5L/2vo25P7Mvw
4bWEWMyY4AS/3toiDZjhwNMrMb2lpICrlH9Sc3dAOzUteyVznA5/WF8IyPI64aKn
tl0gxLOZgUBTkBoxVhPj7dNNZu8lMnqAYXmhWt+oRr7t1HHp2lOtk2u/ndyV0kKL
xufx5FYVJQel2yRBGc/C1QLN18nC1y6u5pITaQIDAQABo0IwQDAOBgNVHQ8BAf8E
BAMCAqQwHQYDVR0lBBYwFAYIKwYBBQUHAwEGCCsGAQUFBwMCMA8GA1UdEwEB/wQF
MAMBAf8wDQYJKoZIhvcNAQELBQADggEBACFT92PWBFeCT7By8y8+EkB2TD1QVMZm
NDpBS75q5s2yIumFwJrbY6YsHtRkN1Zx9jc4LiJFHC6r0ES3tbCDapsxocvzn7dW
XLNTtnSx0zPxNXZzgmTsamfunBd4gszdXMshJ+bKsEoTXhJEXVjZq/k0EZS8L4Mp
NZ7ciPqwAI1Tg+mFGp5UOvzxYLyW8nCLPykC73y3ob1tiO6xdyD/orTAbA6pIMc9
7ajTfwYj4Q6JPY/QAmu0S+4hJHs724IrC6hiXUlQNVVRW/d3k+nXbYttnnmPnQXC
RyK2ru7R8H43Zlwj26kQJo6naQoQ0+Xcjcyk5llPqJxCrk3uoHF0r4U=
-----END CERTIFICATE-----
EOF
```

执行更新命令，使证书生效:

```bash
$ update-ca-trust extract 
```



#### 2、添加 Helm 仓库

添加 Helm 仓库:

```bash
$ helm repo add myrepo --username=admin --password=Mydlq123456 https://hub.mydlq.club/chartrepo/library
```

- --username：harbor仓库用户名
- --password：harbor仓库密码
- --ca-file：指向ca.crt证书地址
- chartrepo：如果是chart仓库地址，中间必须加chartrepo
- library：仓库的项目名称

查看仓库列表：

```bash
$ helm repo list

NAME            URL                                                       
stable          https://kubernetes-charts.storage.googleapis.com          
harbor          https://helm.goharbor.io                                  
myrepo          https://hub.mydlq.club/chartrepo/library 
```



### 八、测试功能

#### 1、推送与拉取 Docker 镜像

这里为了测试推送镜像，先下载一个用于测试的 `helloworld` 小镜像，然后推送到 `hub.mydlq.club` 仓库：

```bash
### 拉取 Helloworld 镜像
$ docker pull hello-world:latest

### 将下载的镜像使用 tag 命令改变镜像名
$ docker tag hello-world:latest hub.mydlq.club/library/hello-world:latest

### 推送镜像到镜像仓库
$ docker push hub.mydlq.club/library/hello-world:latest
```

将之前的下载的镜像删除，然后测试从 `hub.mydlq.club` 下载镜像进行测试：

```bash
### 删除之前镜像
$ docker rmi hello-world:latest
$ docker rmi hello-world:latest hub.mydlq.club/library/hello-world:latest

### 测试从 `hub.mydlq.club` 下载新镜像
$ docker pull hub.mydlq.club/library/hello-world:latest
```



#### 2、推送与拉取 Chart

Helm 要想推送 Chart 到 Helm 仓库，需要提前安装上传插件：

```bash
$ helm plugin install https://github.com/chartmuseum/helm-push
```

然后创建一个测试的 Chart 进行推送测试：

```bash
### 创建一个测 试chart
$ helm create hello

### 打包chart，将chart打包成tgz格式
$ helm package hello

### 推送 chart 进行测试
$ helm push hello-0.1.0.tgz myrepo

Pushing hello-0.1.0.tgz to myrepo...
Done.
```

























































## 在 Kubernetes 中部署高可用 Harbor 镜像仓库



**系统环境：**

- kubernetes 版本：1.18.10
- Harbor Chart 版本：1.5.2
- Harbor 版本：2.1.2
- Helm 版本：3.3.4
- 持久化存储驱动：Ceph RBD



### 1. Harbor 简介

#### 简介

Harbor 是一个开放源代码容器镜像注册表，可通过基于角色权限的访问控制来管理镜像，还能扫描镜像中的漏洞并将映像签名为受信任。Harbor 是 CNCF 孵化项目，可提供合规性，性能和互操作性，以帮助跨 Kubernetes 和 Docker 等云原生计算平台持续，安全地管理镜像。



#### 特性

- 管理：多租户、可扩展
- 安全：安全和漏洞分析、内容签名与验证



### 2. 创建自定义证书

安装 Harbor 我们会默认使用 HTTPS 协议，需要 TLS 证书，如果我们没用自己设定自定义证书文件，那么 Harbor 将自动创建证书文件，不过这个有效期只有一年时间，所以这里我们生成自签名证书，为了避免频繁修改证书，将证书有效期为 100 年，操作如下：

#### 安装 cfssl

fssl 是 CloudFlare 开源的一款 PKI/TLS 工具,cfssl 包含一个`命令行工具`和一个用于`签名`，验证并且捆绑 TLS 证书的`HTTP API服务`,使用 Go 语言编写.

github: https://github.com/cloudflare/cfssl

下载地址: https://pkg.cfssl.org/

macOS 安装步骤：

```bash
$ brew install cfssl
```

通用安装方式：

```bash
$ wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 -O /usr/local/bin/cfssl
$ wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 -O /usr/local/bin/cfssljson
$ wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64 -O /usr/local/bin/cfssl-certinfo
$ chmod +x /usr/local/bin/cfssl*
```



#### 获取默认配置

```bash
$ cfssl print-defaults config > ca-config.json
$ cfssl print-defaults csr > ca-csr.json
```



#### 生成 CA 证书

将`ca-config.json`内容修改为：

```json
{
    "signing": {
        "default": {
            "expiry": "876000h"
        },
        "profiles": {
            "harbor": {
                "expiry": "876000h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth"
                ]
            }
        }
    }
}
```

修改`ca-csr.json`文件内容为：

```json
{
  "CN": "CA",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "hangzhou",
      "L": "hangzhou",
      "O": "harbor",
      "OU": "System"
    }
  ]
}
```

修改好配置文件后,接下来就可以生成 CA 证书了：

```bash
$ cfssl gencert -initca ca-csr.json | cfssljson -bare ca
2020/12/30 00:45:55 [INFO] generating a new CA key and certificate from CSR
2020/12/30 00:45:55 [INFO] generate received request
2020/12/30 00:45:55 [INFO] received CSR
2020/12/30 00:45:55 [INFO] generating key: rsa-2048
2020/12/30 00:45:56 [INFO] encoded CSR
2020/12/30 00:45:56 [INFO] signed certificate with serial number 529798847867094212963042958391637272775966762165
```

此时目录下会出现三个文件：

```bash
$ tree
├── ca-config.json #这是刚才的json
├── ca.csr
├── ca-csr.json    #这也是刚才申请证书的json
├── ca-key.pem
├── ca.pem
```

这样 我们就生成了:

- 根证书文件: `ca.pem`
- 根证书私钥: `ca-key.pem`
- 根证书申请文件: `ca.csr` (csr 是不是 client ssl request?)



#### 签发证书

创建`harbor-csr.json`,内容为：

```json
{
    "CN": "harbor",
    "hosts": [
        "example.net",
        "*.example.net"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "US",
            "ST": "CA",
            "L": "San Francisco",
	    "O": "harbor",
	    "OU": "System"
        }
    ]
}
```

使用之前的 CA 证书签发 harbor 证书：

```bash
$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=harbor harbor-csr.json | cfssljson -bare harbor
2020/12/30 00:50:31 [INFO] generate received request
2020/12/30 00:50:31 [INFO] received CSR
2020/12/30 00:50:31 [INFO] generating key: rsa-2048
2020/12/30 00:50:31 [INFO] encoded CSR
2020/12/30 00:50:31 [INFO] signed certificate with serial number 372641098655462687944401141126722021767151134362
```

此时目录下会多几个文件：

```bash
$ tree -L 1
├── etcd.csr
├── etcd-csr.json
├── etcd-key.pem
├── etcd.pem
```

至此，harbor 的证书生成完成。



#### 生成 Secret 资源

创建 Kubernetes 的 Secret 资源，且将证书文件导入：

- \- n：指定创建资源的 Namespace
- –from-file：指定要导入的文件地址

```bash
$ kubectl create ns harbor
$ kubectl -n harbor create secret generic harbor-tls --from-file=tls.crt=harbor.pem --from-file=tls.key=harbor-key.pem --from-file=ca.crt=ca.pem
```

查看是否创建成功：

```bash
$ kubectl -n harbor get secret harbor-tls
NAME         TYPE     DATA   AGE
harbor-tls   Opaque   3      1m
```



### 3. 使用 Ceph S3 为 Harbor chart 提供后端存储

#### 创建 radosgw

如果你是通过 `ceph-deploy` 部署的，可以通过以下步骤创建 `radosgw`：

先安装 radosgw：

```bash
$ ceph-deploy install --rgw 172.16.7.1 172.16.7.2 172.16.7.3
```

然后创建 radosgw：

```bash
$ ceph-deploy rgw create 172.16.7.1 172.16.7.2 172.16.7.3
```



如果你是通过 `cephadm` 部署的，可以通过以下步骤创建 `radosgw`：

cephadm 将 radosgw 部署为管理特定**领域**和**区域**的守护程序的集合。例如，要在 `172.16.7.1` 上部署 1 个服务于 mytest 领域和 myzone 区域的 rgw 守护程序：

```bash
#如果尚未创建领域，请首先创建一个领域：
$ radosgw-admin realm create --rgw-realm=mytest --default

#接下来创建一个新的区域组：
$ radosgw-admin zonegroup create --rgw-zonegroup=myzg --master --default

#接下来创建一个区域：
$ radosgw-admin zone create --rgw-zonegroup=myzg --rgw-zone=myzone --master --default

#为特定领域和区域部署一组radosgw守护程序：
$ ceph orch apply rgw mytest myzone --placement="1 172.16.7.1"
```



查看服务状态：

```bash
$ ceph orch ls|grep rgw
rgw.mytest.myzone      1/1  5m ago     7w   count:1 k8s01  docker.io/ceph/ceph:v15     4405f6339e35
```

测试服务是否正常：

```bash
$ curl -s http://172.16.7.1
```

正常返回如下数据：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ListAllMyBucketsResult xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
  <Owner>
    <ID>anonymous</ID>
    <DisplayName></DisplayName>
  </Owner>
  <Buckets></Buckets>
</ListAllMyBucketsResult>
```

查看 `zonegroup`：

```json
$ radosgw-admin zonegroup get
{
    "id": "ed34ba6e-7089-4b7f-91c4-82fc856fc16c",
    "name": "myzg",
    "api_name": "myzg",
    "is_master": "true",
    "endpoints": [],
    "hostnames": [],
    "hostnames_s3website": [],
    "master_zone": "650e7cca-aacb-4610-a589-acd605d53d23",
    "zones": [
        {
            "id": "650e7cca-aacb-4610-a589-acd605d53d23",
            "name": "myzone",
            "endpoints": [],
            "log_meta": "false",
            "log_data": "false",
            "bucket_index_max_shards": 11,
            "read_only": "false",
            "tier_type": "",
            "sync_from_all": "true",
            "sync_from": [],
            "redirect_zone": ""
        }
    ],
    "placement_targets": [
        {
            "name": "default-placement",
            "tags": [],
            "storage_classes": [
                "STANDARD"
            ]
        }
    ],
    "default_placement": "default-placement",
    "realm_id": "e63c234c-e069-4a0d-866d-1ebdc69ec5fe",
    "sync_policy": {
        "groups": []
    }
}
```



#### Create Auth Key

```bash
$ ceph auth get-or-create client.radosgw.gateway osd 'allow rwx' mon 'allow rwx' -o /etc/ceph/ceph.client.radosgw.keyring
```

分发 `/etc/ceph/ceph.client.radosgw.keyring` 到其它 radosgw 节点。



#### 创建对象存储用户和访问凭证

1. Create a radosgw user for s3 access

   ```bash
   $ radosgw-admin user create --uid="harbor" --display-name="Harbor Registry"
   ```

2. Create a swift user

   ```bash
   $ adosgw-admin subuser create --uid=harbor --subuser=harbor:swift --access=full
   ```

3. Create Secret Key

   ```bash
   $ radosgw-admin key create --subuser=harbor:swift --key-type=swift --gen-secret
   ```

   记住 `keys` 字段中的 `access_key` & `secret_key`



#### 创建存储桶（bucket）

首先需要安装 `awscli`：

```bash
$ pip3 install awscli  -i https://pypi.tuna.tsinghua.edu.cn/simple
```

查看秘钥：

```bash
$ radosgw-admin user info --uid="harbor"|jq .keys
[
  {
    "user": "harbor",
    "access_key": "VGZQY32LMFQOQPVNTDSJ",
    "secret_key": "YZMMYqoy1ypHaqGOUfwLvdAj9A731iDYDjYqwkU5"
  }
]
```

配置 awscli：

```bash
$ aws configure --profile=ceph
AWS Access Key ID [None]: VGZQY32LMFQOQPVNTDSJ
AWS Secret Access Key [None]: YZMMYqoy1ypHaqGOUfwLvdAj9A731iDYDjYqwkU5
Default region name [None]:
Default output format [None]: json
```

配置完成后，凭证将会存储到 `~/.aws/credentials`：

```bash
$ cat ~/.aws/credentials
[ceph]
aws_access_key_id = VGZQY32LMFQOQPVNTDSJ
aws_secret_access_key = YZMMYqoy1ypHaqGOUfwLvdAj9A731iDYDjYqwkU5
```

配置将会存储到 `~/.aws/config`：

```bash
$ cat ~/.aws/config
[profile ceph]
region = cn-hangzhou-1
output = json
```

创建存储桶（bucket）：

```bash
$ aws --profile=ceph --endpoint=http://172.16.7.1 s3api create-bucket --bucket harbor
```

查看存储桶（bucket）列表：

```
$ radosgw-admin bucket list
[
    "harbor"
]
```

查看存储桶状态：

```bash
$ radosgw-admin bucket stats
[
    {
        "bucket": "harbor",
        "num_shards": 11,
        "tenant": "",
        "zonegroup": "ed34ba6e-7089-4b7f-91c4-82fc856fc16c",
        "placement_rule": "default-placement",
        "explicit_placement": {
            "data_pool": "",
            "data_extra_pool": "",
            "index_pool": ""
        },
        "id": "650e7cca-aacb-4610-a589-acd605d53d23.194274.1",
        "marker": "650e7cca-aacb-4610-a589-acd605d53d23.194274.1",
        "index_type": "Normal",
        "owner": "harbor",
        "ver": "0#1,1#1,2#1,3#1,4#1,5#1,6#1,7#1,8#1,9#1,10#1",
        "master_ver": "0#0,1#0,2#0,3#0,4#0,5#0,6#0,7#0,8#0,9#0,10#0",
        "mtime": "2020-12-29T17:19:02.481567Z",
        "creation_time": "2020-12-29T17:18:58.940915Z",
        "max_marker": "0#,1#,2#,3#,4#,5#,6#,7#,8#,9#,10#",
        "usage": {},
        "bucket_quota": {
            "enabled": false,
            "check_on_raw": false,
            "max_size": -1,
            "max_size_kb": 0,
            "max_objects": -1
        }
    }
]
```

查看存储池状态

```bash
$ rados df
POOL_NAME                    USED  OBJECTS  CLONES  COPIES  MISSING_ON_PRIMARY  UNFOUND  DEGRADED    RD_OPS       RD     WR_OPS       WR  USED COMPR  UNDER COMPR
.rgw.root                 2.3 MiB       13       0      39                   0        0         0       533  533 KiB         21   16 KiB         0 B          0 B
cache                         0 B        0       0       0                   0        0         0         0      0 B          0      0 B         0 B          0 B
device_health_metrics     3.2 MiB       18       0      54                   0        0         0       925  929 KiB        951  951 KiB         0 B          0 B
kubernetes                735 GiB    72646      99  217938                   0        0         0  48345148  242 GiB  283283048  7.3 TiB         0 B          0 B
myzone.rgw.buckets.index  8.6 MiB       11       0      33                   0        0         0        44   44 KiB         11      0 B         0 B          0 B
myzone.rgw.control            0 B        8       0      24                   0        0         0         0      0 B          0      0 B         0 B          0 B
myzone.rgw.log              6 MiB      206       0     618                   0        0         0   2188882  2.1 GiB    1457026   32 KiB         0 B          0 B
myzone.rgw.meta           960 KiB        6       0      18                   0        0         0        99   80 KiB         17    8 KiB         0 B          0 B

total_objects    72908
total_used       745 GiB
total_avail      87 TiB
total_space      88 TiB
```



#### 3. 设置 Harbor 配置清单

由于我们需要通过 Helm 安装 Harbor 仓库，需要提前创建 Harbor Chart 的配置清单文件，里面是对要创建的应用 Harbor 进行一系列参数配置，由于参数过多，关于都有 Harbor Chart 都能够配置哪些参数这里就不一一罗列，可以通过访问 [Harbor-helm 的 Github 地址](https://github.com/goharbor/harbor-helm) 进行了解。

下面描述下，需要的一些配置参数：

**values.yaml**

```yaml
#入口配置，我只在内网使用，所以直接使用 cluserIP
expose:
  type: clusterIP
  tls:
    ### 是否启用 https 协议
    enabled: true
    certSource: secret
    auto:
      # The common name used to generate the certificate, it's necessary
      # when the type isn't "ingress"
      commonName: "harbor.example.net"
    secret:
      # The name of secret which contains keys named:
      # "tls.crt" - the certificate
      # "tls.key" - the private key
      secretName: "harbor-tls"
      # The name of secret which contains keys named:
      # "tls.crt" - the certificate
      # "tls.key" - the private key
      # Only needed when the "expose.type" is "ingress".
      notarySecretName: ""

## 如果Harbor部署在代理后，将其设置为代理的URL
externalURL: https://harbor.example.net

### Harbor 各个组件的持久化配置，并将 storageClass 设置为集群默认的 storageClass
persistence:
  enabled: true
  # Setting it to "keep" to avoid removing PVCs during a helm delete
  # operation. Leaving it empty will delete PVCs after the chart deleted
  # (this does not apply for PVCs that are created for internal database
  # and redis components, i.e. they are never deleted automatically)
  resourcePolicy: "keep"
  persistentVolumeClaim:
    registry:
      # Use the existing PVC which must be created manually before bound,
      # and specify the "subPath" if the PVC is shared with other components
      existingClaim: ""
      # Specify the "storageClass" used to provision the volume. Or the default
      # StorageClass will be used(the default).
      # Set it to "-" to disable dynamic provisioning
      storageClass: "csi-rbd-sc"
      subPath: ""
      accessMode: ReadWriteOnce
      size: 100Gi
    chartmuseum:
      existingClaim: ""
      storageClass: "csi-rbd-sc"
      subPath: ""
      accessMode: ReadWriteOnce
      size: 5Gi
    jobservice:
      existingClaim: ""
      storageClass: "csi-rbd-sc"
      subPath: ""
      accessMode: ReadWriteOnce
      size: 5Gi
    # If external database is used, the following settings for database will
    # be ignored
    database:
      existingClaim: ""
      storageClass: "csi-rbd-sc"
      subPath: ""
      accessMode: ReadWriteOnce
      size: 5Gi
    # If external Redis is used, the following settings for Redis will
    # be ignored
    redis:
      existingClaim: ""
      storageClass: "csi-rbd-sc"
      subPath: ""
      accessMode: ReadWriteOnce
      size: 5Gi
    trivy:
      existingClaim: ""
      storageClass: "csi-rbd-sc"
      subPath: ""
      accessMode: ReadWriteOnce
      size: 5Gi

### 默认用户名 admin 的密码配置，注意：密码中一定要包含大小写字母与数字
harborAdminPassword: "Mydlq123456"

### 设置日志级别
logLevel: info

#各个组件 CPU & Memory 资源相关配置
nginx:
  resources:
    requests:
      memory: 256Mi
      cpu: 500m
portal:
  resources:
    requests:
      memory: 256Mi
      cpu: 500m
core:
  resources:
    requests:
      memory: 256Mi
      cpu: 1000m
jobservice:
  resources:
    requests:
      memory: 256Mi
      cpu: 500m
registry:
  registry:
    resources:
      requests:
        memory: 256Mi
        cpu: 500m
  controller:
    resources:
      requests:
        memory: 256Mi
        cpu: 500m
clair:
  clair:
    resources:
      requests:
        memory: 256Mi
        cpu: 500m
  adapter:
    resources:
      requests:
        memory: 256Mi
        cpu: 500m
notary:
  server:
    resources:
      requests:
        memory: 256Mi
        cpu: 500m
  signer:
    resources:
      requests:
        memory: 256Mi
        cpu: 500m
database:
  internal:
    resources:
      requests:
        memory: 256Mi
        cpu: 500m
redis:
  internal:
    resources:
      requests:
        memory: 256Mi
        cpu: 500m
trivy:
  enabled: true
  resources:
    requests:
      cpu: 200m
      memory: 512Mi
    limits:
      cpu: 1000m
      memory: 1024Mi

#开启 chartmuseum，使 Harbor 能够存储 Helm 的 chart
chartmuseum:
  enabled: true
  resources:
    requests:
     memory: 256Mi
     cpu: 500m

  imageChartStorage:
    # Specify whether to disable `redirect` for images and chart storage, for
    # backends which not supported it (such as using minio for `s3` storage type), please disable
    # it. To disable redirects, simply set `disableredirect` to `true` instead.
    # Refer to
    # https://github.com/docker/distribution/blob/master/docs/configuration.md#redirect
    # for the detail.
    disableredirect: false
    # Specify the "caBundleSecretName" if the storage service uses a self-signed certificate.
    # The secret must contain keys named "ca.crt" which will be injected into the trust store
    # of registry's and chartmuseum's containers.
    # caBundleSecretName:

    # Specify the type of storage: "filesystem", "azure", "gcs", "s3", "swift",
    # "oss" and fill the information needed in the corresponding section. The type
    # must be "filesystem" if you want to use persistent volumes for registry
    # and chartmuseum
    type: s3
    s3:
      region: cn-hangzhou-1
      bucket: harbor
      accesskey: VGZQY32LMFQOQPVNTDSJ
      secretkey: YZMMYqoy1ypHaqGOUfwLvdAj9A731iDYDjYqwkU5
      regionendpoint: http://172.16.7.1
      #encrypt: false
      #keyid: mykeyid
      secure: false
      #skipverify: false
      #v4auth: true
      #chunksize: "5242880"
      #rootdirectory: /s3/object/name/prefix
      #storageclass: STANDARD
      #multipartcopychunksize: "33554432"
      #multipartcopymaxconcurrency: 100
      #multipartcopythresholdsize: "33554432"
```



### 4. 安装 Harbor

#### 添加 Helm 仓库

```bash
$ helm repo add harbor https://helm.goharbor.io
```



#### 部署 Harbor

```bash
$ helm install harbor harbor/harbor -f values.yaml -n harbor
```



#### 查看应用是否部署完成

```bash
$ kubectl -n harbor get pod
NAME                                          READY   STATUS    RESTARTS   AGE
harbor-harbor-chartmuseum-55fb975fbd-74vnh    1/1     Running   0          3m
harbor-harbor-clair-695c7f9c69-7gpkh          2/2     Running   0          3m
harbor-harbor-core-687cfb49b6-zmwxr           1/1     Running   0          3m
harbor-harbor-database-0                      1/1     Running   0          3m
harbor-harbor-jobservice-88994b9b7-684vb      1/1     Running   0          3m
harbor-harbor-nginx-6758559548-x9pq6          1/1     Running   0          3m
harbor-harbor-notary-server-6d55b785f-6jsq9   1/1     Running   0          3m
harbor-harbor-notary-signer-9696cbdd8-8tfw9   1/1     Running   0          3m
harbor-harbor-portal-6f474574c4-8jzh2         1/1     Running   0          3m
harbor-harbor-redis-0                         1/1     Running   0          3m
harbor-harbor-registry-5b6cbfb4cf-42fm9       2/2     Running   0          3m
harbor-harbor-trivy-0                         1/1     Running   0          3m
```



#### Host 配置域名

接下来配置 Hosts，客户端想通过域名访问服务，必须要进行 DNS 解析，由于这里没有 DNS 服务器进行域名解析，所以修改 hosts 文件将 Harbor 指定 `clusterIP` 和自定义 host 绑定。首先查看 nginx 的 clusterIP：

```bash
$ kubectl -n harbor get svc harbor-harbor-nginx
NAME                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
harbor-harbor-nginx   ClusterIP   10.109.50.142   <none>        80/TCP,443/TCP   22h
```

打开主机的 Hosts 配置文件，往其加入下面配置：

```bash
10.109.50.142 harbor.example.net
```

如果想在集群外访问，建议将 Service nginx 的 type 改为 `nodePort` 或者通过 `ingress` 来代理。当然，如果你在集群外能够直接访问 clusterIP，那更好。

输入地址 `https://harbor.example.net` 访问 Harbor 仓库。

- 用户：admin
- 密码：Mydlq123456 (在安装配置中自定义的密码)



#### 进入后可以看到 Harbor 的管理后台：

![img](D:\学习资料\笔记\k8s\k8s图\20201230163549.png)



### 5. 服务器配置镜像仓库

对于 Containerd 来说，不能像 docker 一样 `docker login` 登录到镜像仓库，需要修改其配置文件来进行认证。`/etc/containerd/config.toml` 需要添加如下内容：

```toml
    [plugins."io.containerd.grpc.v1.cri".registry]
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
        ...
      [plugins."io.containerd.grpc.v1.cri".registry.configs]
        [plugins."io.containerd.grpc.v1.cri".registry.configs."harbor.example.net".tls]
          insecure_skip_verify = true
        [plugins."io.containerd.grpc.v1.cri".registry.configs."harbor.example.net".auth]
          username = "admin"
          password = "Mydlq123456"
```

由于 Harbor 是基于 Https 的，理论上需要提前配置 tls 证书，但可以通过 `insecure_skip_verify` 选项跳过证书认证。

当然，如果你想通过 Kubernetes 的 secret 来进行用户验证，配置还可以精简下：

```toml
    [plugins."io.containerd.grpc.v1.cri".registry]
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
        ...
      [plugins."io.containerd.grpc.v1.cri".registry.configs]
        [plugins."io.containerd.grpc.v1.cri".registry.configs."harbor.example.net".tls]
          insecure_skip_verify = true
```

Kubernetes 集群使用 `docker-registry` 类型的 Secret 来通过镜像仓库的身份验证，进而拉取私有映像。所以需要创建 Secret，命名为 `regcred`：

```bash
$ kubectl create secret docker-registry regcred \
  --docker-server=<你的镜像仓库服务器> \
  --docker-username=<你的用户名> \
  --docker-password=<你的密码> \
  --docker-email=<你的邮箱地址>
```

然后就可以在 Pod 中使用该 secret 来访问私有镜像仓库了，下面是一个示例 Pod 配置文件：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-reg
spec:
  containers:
  - name: private-reg-container
    image: <your-private-image>
  imagePullSecrets:
  - name: regcred
```

如果你不嫌麻烦，想更安全一点，那就老老实实将 CA、证书和秘钥拷贝到所有节点的 `/etc/ssl/certs/` 目录下。`/etc/containerd/config.toml` 需要添加的内容更多一点：

```toml
    [plugins."io.containerd.grpc.v1.cri".registry]
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
        ...
      [plugins."io.containerd.grpc.v1.cri".registry.configs]
        [plugins."io.containerd.grpc.v1.cri".registry.configs."harbor.example.net".tls]
          ca_file = "/etc/ssl/certs/ca.pem"
          cert_file = "/etc/ssl/certs/harbor.pem"
          key_file  = "/etc/ssl/certs/harbor-key.pem"
```



### 6. 测试功能

这里为了测试推送镜像，先下载一个用于测试的 `helloworld` 小镜像，然后推送到 `harbor.example.net` 仓库：

```bash
### 拉取 Helloworld 镜像
$ ctr i pull bxsfpjcb.mirror.aliyuncs.com/library/hello-world:latest
bxsfpjcb.mirror.aliyuncs.com/library/hello-world:latest:                          resolved       |++++++++++++++++++++++++++++++++++++++|
index-sha256:1a523af650137b8accdaed439c17d684df61ee4d74feac151b5b337bd29e7eec:    done           |++++++++++++++++++++++++++++++++++++++|
manifest-sha256:90659bf80b44ce6be8234e6ff90a1ac34acbeb826903b02cfa0da11c82cbc042: done           |++++++++++++++++++++++++++++++++++++++|
layer-sha256:0e03bdcc26d7a9a57ef3b6f1bf1a210cff6239bff7c8cac72435984032851689:    done           |++++++++++++++++++++++++++++++++++++++|
config-sha256:bf756fb1ae65adf866bd8c456593cd24beb6a0a061dedf42b26a993176745f6b:   done           |++++++++++++++++++++++++++++++++++++++|
elapsed: 15.8s                                                                    total:  2.6 Ki (166.0 B/s)
unpacking linux/amd64 sha256:1a523af650137b8accdaed439c17d684df61ee4d74feac151b5b337bd29e7eec...
done

### 将下载的镜像使用 tag 命令改变镜像名
$ ctr i tag bxsfpjcb.mirror.aliyuncs.com/library/hello-world:latest harbor.example.net/library/hello-world:latest
harbor.example.net/library/hello-world:latest

### 推送镜像到镜像仓库
$ ctr i push --user admin:Mydlq123456 --platform linux/amd64 harbor.example.net/library/hello-world:latest
manifest-sha256:90659bf80b44ce6be8234e6ff90a1ac34acbeb826903b02cfa0da11c82cbc042: done           |++++++++++++++++++++++++++++++++++++++|
config-sha256:bf756fb1ae65adf866bd8c456593cd24beb6a0a061dedf42b26a993176745f6b:   done           |++++++++++++++++++++++++++++++++++++++|
layer-sha256:0e03bdcc26d7a9a57ef3b6f1bf1a210cff6239bff7c8cac72435984032851689:    done           |++++++++++++++++++++++++++++++++++++++|
elapsed: 2.2 s                                                                    total:  4.5 Ki (2.0 KiB/s)
```

镜像仓库中也能看到：

![img](D:\学习资料\笔记\k8s\k8s图\20201230171408.png)

将之前的下载的镜像删除，然后测试从 `harbor.example.net` 下载镜像进行测试：

```bash
### 删除之前镜像
$ ctr i rm harbor.example.net/library/hello-world:latest
$ ctr i rm bxsfpjcb.mirror.aliyuncs.com/library/hello-world:latest

### 测试从 harbor.example.net 下载新镜像
$ ctr i pull harbor.example.net/library/hello-world:latest
harbor.example.net/library/hello-world:latest:                                   resolved       |++++++++++++++++++++++++++++++++++++++|
manifest-sha256:90659bf80b44ce6be8234e6ff90a1ac34acbeb826903b02cfa0da11c82cbc042: done           |++++++++++++++++++++++++++++++++++++++|
layer-sha256:0e03bdcc26d7a9a57ef3b6f1bf1a210cff6239bff7c8cac72435984032851689:    done           |++++++++++++++++++++++++++++++++++++++|
config-sha256:bf756fb1ae65adf866bd8c456593cd24beb6a0a061dedf42b26a993176745f6b:   done           |++++++++++++++++++++++++++++++++++++++|
elapsed: 0.6 s                                                                    total:  525.0  (874.0 B/s)
unpacking linux/amd64 sha256:90659bf80b44ce6be8234e6ff90a1ac34acbeb826903b02cfa0da11c82cbc042...
done
```



### 参考

- [通过 Helm 搭建 Docker 镜像仓库 Harbor](http://www.mydlq.club/article/66/)



## Kubernetes Pod 突然就无法挂载 Ceph RBD 存储卷了。。



### **前言**

Kubernetes 坑不坑？坑！Ceph 坑不坑？坑！他俩凑到一起呢？巨坑！

之前在 Kubernetes 集群中部署了高可用 Harbor 镜像仓库，并使用 Ceph RBD 提供持久化存储。本来是挺美滋滋的，谁料昨天有一台节点 `NotReady` 了，导致 Harbor 的某个组件所在的 Pod 被重新调度了，但是重新调度后的 Pod 并没有启动成功。

进一步通过 describe pod 查看 `events`，发现如下 Warning：

```sh
Events:
  Type     Reason              Age   From                     Message
  ----     ------              ----  ----                     -------
  Normal   Scheduled           23s   default-scheduler        Successfully assigned harbor/harbor-harbor-registry-5796cdddd7-kxzp9 to k8s03
  Warning  FailedAttachVolume  22s   attachdetach-controller  Multi-Attach error for volume "pvc-ec045b5e-2471-469d-9a1b-6e7db0e938b3" Volume is already exclusively attached to one node and can't be attached to another
```

好家伙，当前的 `PV` 所对应的 RBD `image` 还在被另一个 Pod 占用着，所以无法挂载到新 Pod 中。我到 `NotReady` 的节点中通过 `docker rm -vf xxx` 直接将之前的 Pod 删除，仍然不起作用。

现在看来我只能从之前的 Pod 所在节点中将 RBD image 映射的块设备强行 `unmount` 了。首先得找到该 PV 所对应的 RBD image，直接查看 PV 的信息：

```sh
$ kubectl -n harbor get pv pvc-ec045b5e-2471-469d-9a1b-6e7db0e938b3 -o go-template='{{.spec.csi.volumeAttributes.imageName}}'

csi-vol-bf0dc641-4a5a-11eb-988c-6ab597a1411c
```

到 Ceph 管理节点中查看该 image 正在被谁使用：

```sh
$ rbd status kubernetes/csi-vol-bf0dc641-4a5a-11eb-988c-6ab597a1411c
Watchers:
 watcher=172.16.7.1:0/3619044864 client.195600 cookie=18446462598732840980
```

找到了罪魁祸首，于是登录到 `172.16.7.1` 将块设备强行卸载：

```sh
$ docker ps|grep csi
77255fe4f26b        650757c4f32d                  "/usr/local/bin/ceph…"   3 weeks ago         Up 3 weeks                              k8s_liveness-prometheus_csi-rbdplugin-hscf8_ceph-csi_2b7da817-3f4a-4e8f-9f99-a39da07c5b94_5
fb4e5e10f064        650757c4f32d                  "/usr/local/bin/ceph…"   3 weeks ago         Up 3 weeks                              k8s_csi-rbdplugin_csi-rbdplugin-hscf8_ceph-csi_2b7da817-3f4a-4e8f-9f99-a39da07c5b94_5
5330c84529e9        37c1d9ea538b                  "/csi-node-driver-re…"   3 weeks ago         Up 3 weeks                              k8s_driver-registrar_csi-rbdplugin-hscf8_ceph-csi_2b7da817-3f4a-4e8f-9f99-a39da07c5b94_6
4452755ffccf        k8s.gcr.io/pause:3.2          "/pause"                 3 weeks ago         Up 3 weeks                              k8s_POD_csi-rbdplugin-hscf8_ceph-csi_2b7da817-3f4a-4e8f-9f99-a39da07c5b94_5

$ docker exec -it fb4e5e10f064 bash
[root@k8s01 /]# rbd showmapped|grep csi-vol-bf0dc641-4a5a-11eb-988c-6ab597a1411c
4   kubernetes             csi-vol-bf0dc641-4a5a-11eb-988c-6ab597a1411c  -     /dev/rbd4

[root@k8s01 /]# rbd unmap -o force /dev/rbd4
```

现在在来看新 Pod，已经启动成功了：

```sh
Events:
  Type     Reason                  Age                   From                     Message
  ----     ------                  ----                  ----                     -------
  Normal   Scheduled               18m                   default-scheduler        Successfully assigned harbor/harbor-harbor-registry-5796cdddd7-kxzp9 to k8s03
  Warning  FailedAttachVolume      18m                   attachdetach-controller  Multi-Attach error for volume "pvc-ec045b5e-2471-469d-9a1b-6e7db0e938b3" Volume is already exclusively attached to one node and can't be attached to another
  Warning  FailedMount             14m                   kubelet, k8s03           Unable to attach or mount volumes: unmounted volumes=[registry-data], unattached volumes=[default-token-phjbz registry-data registry-root-certificate registry-htpasswd registry-config]: timed out waiting for the condition
  Normal   SuccessfulAttachVolume  12m                   attachdetach-controller  AttachVolume.Attach succeeded for volume "pvc-ec045b5e-2471-469d-9a1b-6e7db0e938b3"
  Warning  FailedMount             12m                   kubelet, k8s03           Unable to attach or mount volumes: unmounted volumes=[registry-data], unattached volumes=[registry-htpasswd registry-config default-token-phjbz registry-data registry-root-certificate]: timed out waiting for the condition
  Warning  FailedMount             5m21s (x2 over 16m)   kubelet, k8s03           Unable to attach or mount volumes: unmounted volumes=[registry-data], unattached volumes=[registry-config default-token-phjbz registry-data registry-root-certificate registry-htpasswd]: timed out waiting for the condition
  Warning  FailedMount             3m5s (x2 over 9m55s)  kubelet, k8s03           Unable to attach or mount volumes: unmounted volumes=[registry-data], unattached volumes=[registry-root-certificate registry-htpasswd registry-config default-token-phjbz registry-data]: timed out waiting for the condition
  Warning  FailedMount             2m54s (x9 over 11m)   kubelet, k8s03           MountVolume.MountDevice failed for volume "pvc-ec045b5e-2471-469d-9a1b-6e7db0e938b3" : rpc error: code = Internal desc = rbd image kubernetes/csi-vol-bf0dc641-4a5a-11eb-988c-6ab597a1411c is still being used
  Warning  FailedMount             50s (x2 over 7m39s)   kubelet, k8s03           Unable to attach or mount volumes: unmounted volumes=[registry-data], unattached volumes=[registry-data registry-root-certificate registry-htpasswd registry-config default-token-phjbz]: timed out waiting for the condition
  Normal   Pulling                 15s                   kubelet, k8s03           Pulling image "goharbor/registry-photon:v2.1.2"
  Normal   Pulled                  12s                   kubelet, k8s03           Successfully pulled image "goharbor/registry-photon:v2.1.2"
  Normal   Created                 12s                   kubelet, k8s03           Created container registry
  Normal   Started                 12s                   kubelet, k8s03           Started container regis
```





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





# kubenetes 监控



## Prometheus 监控 kubernetes



### 基础环境

软件版本

- kubernetes 1.18
- prometheus 2.21
- node-exporter 1.0.1
- kube-state-metrics v2.0.0-alpha.1



### 一、概述

本篇主要针对Kubernetes部署Prometheus相关配置介绍，自己编写prometheus 的相关yaml。

另外还有开源的方案：[prometheus-operator](https://github.com/prometheus-operator)/[kube-prometheus](https://github.com/prometheus-operator/kube-prometheus)。



### 二、安装node-exporter

通过node_exporter获取，node_exporter就是抓取用于采集服务器节点的各种运行指标，目前node_exporter几乎支持所有常见的监控点，比如cpu、distats、loadavg、meminfo、netstat等，详细的监控列表可以参考github repo

这里使用DeamonSet控制器来部署该服务，这样每一个节点都会运行一个Pod，如果我们从集群中删除或添加节点后，也会进行自动扩展

**node-exporter.yaml**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: node-exporter
  name: node-exporter-9400
  namespace: default
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      containers:
      - args:
        - --web.listen-address=0.0.0.0:9400
        - --path.procfs=/host/proc
        - --path.sysfs=/host/sys
        - --path.rootfs=/host/root
        image: harbor.foxchan.com/prom/node-exporter:v1.0.1
        imagePullPolicy: IfNotPresent
        name: node-exporter-9400
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /host/proc
          name: proc
        - mountPath: /host/sys
          name: sys
        - mountPath: /host/root
          mountPropagation: HostToContainer
          name: root
          readOnly: true
      dnsPolicy: Default
      hostNetwork: true
      hostPID: true
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
      - hostPath:
          path: /proc
          type: ""
        name: proc
      - hostPath:
          path: /sys
          type: ""
        name: sys
      - hostPath:
          path: /
          type: ""
        name: root
      tolerations:
      - key: "node-role.kubernetes.io/master"
        operator: "Exists"
        effect: "NoSchedule"
  updateStrategy:
    type: OnDelete
```



### 三、安装prometheus

#### 3.1 创建prometheus rbac

01-prom-rbac.yaml

> 和别人的配置不同在于，添加了ingress等权限，便于后期安装`kube-state-metric`等第三方插件，因为权限不足无法获取metric的问题

```yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: 
  - ""
  resources:
  - configmaps
  - secrets
  - nodes
  - nodes/proxy
  - nodes/metrics
  - pods
  - services
  - resourcequotas
  - replicationcontrollers
  - limitranges
  - persistentvolumeclaims
  - persistentvolumes
  - namespaces
  - endpoints
  verbs: 
  - get
  - list
  - watch
- apiGroups:
  - extensions
  - networking.k8s.io
  resources:
  - daemonsets
  - deployments
  - replicasets
  - ingresses
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - apps
  resources:
  - daemonsets
  - deployments
  - replicasets
  - ingresses
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - batch
  resources:
  - cronjobs
  - jobs
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - autoscaling
  resources:
  - horizontalpodautoscalers
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - policy
  resources:
  - poddisruptionbudgets
  verbs:
  - get
  - list
  - watch
- nonResourceURLs: 
  - /metrics
  verbs: 
  - get

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: kube-system

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: kube-system
You have mail in /var/spool/mail/root
```



#### 3.2 prometheus配置文件

02-prom-cm.yaml

> 和官方yaml 不一样的地方在于，修改了node 和cadvisor的metric 地址，避免频繁请求`apiserver` 造成`apiserver` 压力过大

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: prometheus
  namespace: kube-system
data:
  prometheus.yml: |
    global:
      scrape_interval:     15s
      evaluation_interval: 15s  
      scrape_timeout:      10s
    rule_files:
    - /prometheus-rules/*.rules.yml
    scrape_configs:
      - job_name: 'kubernetes-apiservers'
        kubernetes_sd_configs:
        - role: endpoints
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
        - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
          action: keep
          regex: default;kubernetes;https

      - job_name: 'kubernetes-controller-manager'
        kubernetes_sd_configs:
        - role: endpoints
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
          insecure_skip_verify: true
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
        - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
          action: keep
          regex: kube-system;kube-controller-manager;https-metrics

      - job_name: 'kubernetes-scheduler'
        kubernetes_sd_configs:
        - role: endpoints
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
          insecure_skip_verify: true
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
        - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
          action: keep
          regex: kube-system;kube-scheduler;https-metrics

      - job_name: 'kubernetes-nodes'
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
          insecure_skip_verify: true
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

        kubernetes_sd_configs:
        - role: node

        relabel_configs:
        - action: labelmap
          regex: __meta_kubernetes_node_label_(.+)

      - job_name: 'kubernetes-cadvisor'
        metrics_path: /metrics/cadvisor
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
          insecure_skip_verify: true
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

        kubernetes_sd_configs:
        - role: node

        relabel_configs:
        - action: labelmap
          regex: __meta_kubernetes_node_label_(.+)

        metric_relabel_configs:
        - source_labels: [instance]
          regex: (.+)
          target_label: node
          replacement: $1
          action: replace

      - job_name: 'kubernetes-service-endpoints'
        scrape_interval: 1m
        scrape_timeout: 10s
        metrics_path: /metrics
        scheme: http

        kubernetes_sd_configs:
        - role: endpoints

        relabel_configs:
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
          regex: "true"
          action: keep
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_metric_path]
          action: replace
          target_label: __metrics_path__
          regex: (.+)
        - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
          action: replace
          regex: ([^:]+)(?::\d+)?;(\d+)
          replacement: $1:$2
          target_label: __address__
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape_scheme]
          action: replace
          target_label: __scheme__
          regex: (https?)
        - action: labelmap
          regex: __meta_kubernetes_service_label_(.+)
        - source_labels: [__meta_kubernetes_namespace]
          action: replace
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_service_name]
          action: replace
          target_label: kubernetes_name

      - job_name: 'kubernetes-services'
        metrics_path: /probe
        params:
          module: [http_2xx]        

        kubernetes_sd_configs:
        - role: service

        relabel_configs:
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_should_be_probed]
          action: keep
          regex: true   
        - source_labels: [__address__]
          target_label: __param_target
        - target_label: __address__
          replacement: blackbox-exporter.default.svc.cluster.local:9115
        - source_labels: [__param_target]
          target_label: instance
          action: labelmap
          regex: __meta_kubernetes_service_label_(.+)
        - source_labels: [__meta_kubernetes_namespace]
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_service_name]
          target_label: kubernetes_name

      - job_name: 'kubernetes-ingresses'
        metrics_path: /probe
        params:
          module: [http_2xx]

        kubernetes_sd_configs:
        - role: ingress

        relabel_configs:
        - source_labels: [__meta_kubernetes_ingress_annotation_prometheus_io_should_be_probed]
          action: keep
          regex: true
        - source_labels: [__meta_kubernetes_ingress_scheme,__address__,__meta_kubernetes_ingress_path]
          regex: (.+);(.+);(.+)
          replacement: ${1}://${2}${3}
          target_label: __param_target
        - target_label: __address__
          replacement: blackbox-exporter.default.svc.cluster.local:9115
        - source_labels: [__param_target]
          target_label: instance
        - action: labelmap
          regex: __meta_kubernetes_ingress_label_(.+)
        - source_labels: [__meta_kubernetes_namespace]
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_ingress_name]
          target_label: kubernetes_name

      - job_name: 'kubernetes-pods'

        kubernetes_sd_configs:
        - role: pod

        relabel_configs:
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_should_be_scraped]
          action: keep
          regex: true
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_metric_path]
          action: replace
          target_label: __metrics_path__
          regex: (.+)
        - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
          action: replace
          regex: ([^:]+)(?::\d+)?;(\d+)
          replacement: $1:$2
          target_label: __address__
        - action: labelmap
          regex: __meta_kubernetes_pod_label_(.+)
        - source_labels: [__meta_kubernetes_namespace]
          action: replace
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_pod_name]
          action: replace
          target_label: kubernetes_pod_name
```



#### 3.3 部署prometheus deployment

03-prom-deploy.yaml

> 添加了`node selector` 和 `hostnetwork`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: kube-system
  labels:
    app: prometheus
spec:
  ports:
  - name: web
    port: 9090
    protocol: TCP
    targetPort: web
  selector:
    app: prometheus
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      serviceAccountName: prometheus
      serviceAccount: prometheus
      initContainers:
      - name: "init-chown-data"
        image: "harbor.foxchan.com/sys/busybox:latest"
        imagePullPolicy: "IfNotPresent"
        command: ["chown", "-R", "65534:65534", "/prometheus/data"]
        volumeMounts:
        - name: prometheusdata
          mountPath: /prometheus/data
          subPath: ""
      volumes:
      - hostPath:
          path: /data/prometheus
        name: prometheusdata
      containers:
      - name: prometheus
        image: harbor.foxchan.com/prom/prometheus:v2.21.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 9090
          name: web
          protocol: TCP
        args:
          - '--config.file=/etc/prometheus/prometheus.yml'
          - '--web.listen-address=:9090'
          - '--web.enable-admin-api'
          - '--web.enable-lifecycle'
          - '--log.level=info'
          - '--storage.tsdb.path=/prometheus/data'
          - '--storage.tsdb.retention=15d'
        volumeMounts:
        - name: config-volume
          mountPath: /etc/prometheus
        - name: prometheusdata
          mountPath: /prometheus/data
        - mountPath: /prometheus-rules
          name: prometheus-rules-all
      volumes:
      - name: config-volume
        configMap:
          name: prometheus-gs
      - hostPath:
          path: /data/prometheus
        name: prometheusdata  
      - name: prometheus-rules-all
        projected:
          defaultMode: 420
          sources:
          - configMap:
              name: prometheus-k8s-rules
          - configMap:
              name: prometheus-nodeexporter-rules
      dnsPolicy: ClusterFirst
      nodeSelector:
        monitor: prometheus-gs
```



#### 3.4 创建rules configmap

prom-node-rules-cm.yaml

```yaml
apiVersion: v1
data:
  node-exporter.rules.yml: |
    groups:  
    - name: node-exporter.rules
      rules:
      - expr: |
          count without (cpu) (
            count without (mode) (
              node_cpu_seconds_total{job="node-exporter"}
            )
          )
        record: instance:node_num_cpu:sum
      - expr: |
          1 - avg without (cpu, mode) (
            rate(node_cpu_seconds_total{job="node-exporter", mode="idle"}[1m])
          )
        record: instance:node_cpu_utilisation:rate1m
      - expr: |
          (
            node_load1{job="node-exporter"}
          /
            instance:node_num_cpu:sum{job="node-exporter"}
          )
        record: instance:node_load1_per_cpu:ratio
      - expr: |
          1 - (
            node_memory_MemAvailable_bytes{job="node-exporter"}
          /
            node_memory_MemTotal_bytes{job="node-exporter"}
          )
        record: instance:node_memory_utilisation:ratio
      - expr: |
          rate(node_vmstat_pgmajfault{job="node-exporter"}[1m])
        record: instance:node_vmstat_pgmajfault:rate1m
      - expr: |
          rate(node_disk_io_time_seconds_total{job="node-exporter", device=~"nvme.+|rbd.+|sd.+|vd.+|xvd.+|dm-.+|dasd.+"}[1m])
        record: instance_device:node_disk_io_time_seconds:rate1m
      - expr: |
          rate(node_disk_io_time_weighted_seconds_total{job="node-exporter", device=~"nvme.+|rbd.+|sd.+|vd.+|xvd.+|dm-.+|dasd.+"}[1m])
        record: instance_device:node_disk_io_time_weighted_seconds:rate1m
      - expr: |
          sum without (device) (
            rate(node_network_receive_bytes_total{job="node-exporter", device!="lo"}[1m])
          )
        record: instance:node_network_receive_bytes_excluding_lo:rate1m
      - expr: |
          sum without (device) (
            rate(node_network_transmit_bytes_total{job="node-exporter", device!="lo"}[1m])
          )
        record: instance:node_network_transmit_bytes_excluding_lo:rate1m
      - expr: |
          sum without (device) (
            rate(node_network_receive_drop_total{job="node-exporter", device!="lo"}[1m])
          )
        record: instance:node_network_receive_drop_excluding_lo:rate1m
      - expr: |
          sum without (device) (
            rate(node_network_transmit_drop_total{job="node-exporter", device!="lo"}[1m])
          )
        record: instance:node_network_transmit_drop_excluding_lo:rate1m
kind: ConfigMap
metadata:
  name: prometheus-nodeexporter-rules
  namespace: kube-system
```

prom-k8s-rules-cm.yaml

> 其余的rules 因为太大，放到git[仓库](https://gitee.com/foxchenlei/k8syaml)



### 四、kube-state-metrics集群资源监控

#### 4.1 概述

已经有了cadvisor，几乎容器运行的所有指标都能拿到，但是下面这种情况却无能为力：

- 我调度了多少个replicas？现在可用的有几个？
- 多少个Pod是running/stopped/terminated状态？
- Pod重启了多少次？
- 我有多少job在运行中

而这些则是kube-state-metrics提供的内容，它基于client-go开发，轮询Kubernetes API，并将Kubernetes的结构化信息转换为metrics。



#### 4.2 功能

kube-state-metrics提供的指标，按照阶段分为三种类别：

- 1.实验性质的：k8s api中alpha阶段的或者spec的字段。
- 2.稳定版本的：k8s中不向后兼容的主要版本的更新
- 3.被废弃的：已经不在维护的。

`kube-state-metrics` 和`kubernetes` 版本对应列表

| kube-state-metrics | **Kubernetes 1.15** | **Kubernetes 1.16** | **Kubernetes 1.17** | **Kubernetes 1.18** | **Kubernetes 1.19** |
| ------------------ | ------------------- | ------------------- | ------------------- | ------------------- | ------------------- |
| **v1.8.0**         | ✓                   | -                   | -                   | -                   | -                   |
| **v1.9.7**         | -                   | ✓                   | -                   | -                   | -                   |
| **v2.0.0-alpha.1** | -                   | -                   | ✓                   | ✓                   | ✓                   |
| **master**         | -                   | -                   | ✓                   | ✓                   | ✓                   |

- `✓` Fully supported version range.
- `-` The Kubernetes cluster has features the client-go library can't use (additional API objects, deprecated APIs, etc).



#### 4.3 安装

**创建rbac**

01-kube-state-rbac.yaml

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  labels:
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/version: 2.0.0-alpha
  name: kube-state-metrics
  namespace: kube-system
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
- apiGroups:
  - apps
  resourceNames:
  - kube-state-metrics
  resources:
  - statefulsets
  verbs:
  - get
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/version: 2.0.0-alpha
  name: kube-state-metrics
rules:
- apiGroups:
  - ""
  resources:
  - configmaps
  - secrets
  - nodes
  - pods
  - services
  - resourcequotas
  - replicationcontrollers
  - limitranges
  - persistentvolumeclaims
  - persistentvolumes
  - namespaces
  - endpoints
  verbs:
  - list
  - watch
  - get
- apiGroups:
  - extensions
  resources:
  - daemonsets
  - deployments
  - replicasets
  - ingresses
  verbs:
  - list
  - watch
- apiGroups:
  - apps
  resources:
  - statefulsets
  - daemonsets
  - deployments
  - replicasets
  verbs:
  - list
  - watch
  - get
- apiGroups:
  - batch
  resources:
  - cronjobs
  - jobs
  verbs:
  - list
  - watch
- apiGroups:
  - autoscaling
  resources:
  - horizontalpodautoscalers
  verbs:
  - list
  - watch
- apiGroups:
  - authentication.k8s.io
  resources:
  - tokenreviews
  verbs:
  - create
- apiGroups:
  - authorization.k8s.io
  resources:
  - subjectacce***eviews
  verbs:
  - create
- apiGroups:
  - policy
  resources:
  - poddisruptionbudgets
  verbs:
  - list
  - watch
- apiGroups:
  - certificates.k8s.io
  resources:
  - certificatesigningrequests
  verbs:
  - list
  - watch
- apiGroups:
  - storage.k8s.io
  resources:
  - storageclasses
  - volumeattachments
  verbs:
  - list
  - watch
- apiGroups:
  - admissionregistration.k8s.io
  resources:
  - mutatingwebhookconfigurations
  - validatingwebhookconfigurations
  verbs:
  - list
  - watch
- apiGroups:
  - networking.k8s.io
  resources:
  - networkpolicies
  verbs:
  - list
  - watch
- apiGroups:
  - coordination.k8s.io
  resources:
  - leases
  verbs:
  - list
  - watch
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/version: 2.0.0-alpha
  name: kube-state-metrics
  namespace: kube-system

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/version: 2.0.0-alpha
  name: kube-state-metrics
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: kube-state-metrics
subjects:
- kind: ServiceAccount
  name: kube-state-metrics
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/version: 2.0.0-alpha
  name: kube-state-metrics
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kube-state-metrics
subjects:
- kind: ServiceAccount
  name: kube-state-metrics
  namespace: kube-system
```

**创建sts**

02-kube-state-sts.yaml

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/version: 2.0.0-alpha
  name: kube-state-metrics
  namespace: kube-system
spec:
  replicas: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: kube-state-metrics
  serviceName: kube-state-metrics
  template:
    metadata:
      labels:
        app.kubernetes.io/name: kube-state-metrics
        app.kubernetes.io/version: 2.0.0-alpha
    spec:
      containers:
      - args:
        - --pod=$(POD_NAME)
        - --pod-namespace=$(POD_NAMESPACE)
        env:
        - name: POD_NAME
          value: ""
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          value: ""
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        image: harbor.foxchan.com/coreos/kube-state-metrics:v2.0.0-alpha.1
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 5
          timeoutSeconds: 5
        name: kube-state-metrics
        ports:
        - containerPort: 8080
          name: http-metrics
        - containerPort: 8081
          name: telemetry
        readinessProbe:
          httpGet:
            path: /
            port: 8081
          initialDelaySeconds: 5
          timeoutSeconds: 5
        securityContext:
          runAsUser: 65534
      nodeSelector:
        kubernetes.io/os: linux
      serviceAccountName: kube-state-metrics
  volumeClaimTemplates: []
---
apiVersion: v1
kind: Service
metadata:
  annotations:
    prometheus.io/scrape: 'true'
    prometheus.io/port: "8080"
  labels:
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/version: 2.0.0-alpha
  name: kube-state-metrics
  namespace: kube-system
spec:
  clusterIP: None
  ports:
  - name: http-metrics
    port: 8080
    targetPort: http-metrics
  - name: telemetry
    port: 8081
    targetPort: telemetry
  selector:
    app.kubernetes.io/name: kube-state-metrics
```



### 五、添加Scheduler和Controller配置

#### 创建kube-controller-manager和kube-scheduler service

高版本k8s 默认使用https，kube-controller-manager端口10257 kube-scheduler端口10259

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kube-controller-manager
  namespace: kube-system
  labels:
    component: kube-controller-manager # 与pod中的labels匹配,便于自动生成endpoints
spec:
  selector:
    component: kube-controller-manager
  type: ClusterIP
  clusterIP: None
  ports:
  - name: https-metrics
    port: 10257
    protocol: TCP
    targetPort: 10257

---
apiVersion: v1
kind: Service
metadata:
  name: kube-scheduler
  namespace: kube-system
  labels:
    component: kube-scheduler # 与pod中的labels匹配,便于自动生成endpoints
spec:
  selector:
    component: kube-scheduler
  type: ClusterIP
  clusterIP: None
  ports:
  - name: https-metrics
    port: 10259
    targetPort: 10259
    protocol: TCP
```



#### 修改yaml文件

默认情况下，`kube-controller-manager`和`kube-scheduler`绑定IP为127.0.0.1,需要改监听地址。

metrics的访问需要证书验证，需要取消证书验证。

修改`kube-controller-manager.yaml` 和 `kube-scheduler.yaml`

```sh
    --bind-address=0.0.0.0
    --authorization-always-allow-paths=/healthz,/metrics
```



#### 添加prometheus 配置文件

```yaml
      - job_name: 'kubernetes-controller-manager'
        kubernetes_sd_configs:
        - role: endpoints
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
          insecure_skip_verify: true
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
        - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
          action: keep
          regex: kube-system;kube-controller-manager;https-metrics

      - job_name: 'kubernetes-scheduler'
        kubernetes_sd_configs:
        - role: endpoints
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
          insecure_skip_verify: true
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
        - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
          action: keep
          regex: kube-system;kube-scheduler;https-metrics
```



### 六、其他exporter

#### 6.1 blackbox-exporter

官方提供的 exporter 之一，可以提供 http、dns、tcp、icmp 的监控数据采集

**Blackbox_exporter 应用场景**

- HTTP 测试
  定义 Request Header 信息
  判断 Http status / Http Respones Header / Http Body 内容
- TCP 测试
  业务组件端口状态监听
  应用层协议定义与监听
- ICMP 测试
  主机探活机制
- POST 测试
  接口联通性
- SSL 证书过期时间

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    prometheus.io/scrape: 'true'
  labels:
    app: blackbox-exporter
  name: blackbox-exporter
spec:
  ports:
  - name: blackbox
    port: 9115
    protocol: TCP
  selector:
    app: blackbox-exporter
  type: ClusterIP

---
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    app: blackbox-exporter
  name: blackbox-exporter
data:
  blackbox.yml: |-
    modules:
      http_2xx:
        prober: http
        timeout: 30s
        http:
          valid_http_versions: ["HTTP/1.1", "HTTP/2"]
          valid_status_codes: [200,302,301,401,404]
          method: GET
          preferred_ip_protocol: "ip4"

---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: blackbox-exporter
  name: blackbox-exporter
spec:
  replicas: 1
  selector:
    matchLabels:
      app: blackbox-exporter
  template:
    metadata:
      labels:
        app: blackbox-exporter
    spec:
      containers:
      - image: harbor.foxchan.com/prom/blackbox-exporter:v0.16.0
        imagePullPolicy: IfNotPresent
        name: blackbox-exporter
        ports:
        - name: blackbox-port
          containerPort: 9115
        readinessProbe:
          tcpSocket:
            port: 9115
          initialDelaySeconds: 5
          timeoutSeconds: 5
        volumeMounts:
        - name: config
          mountPath: /etc/blackbox_exporter
        args:
        - --config.file=/etc/blackbox_exporter/blackbox.yml
        - --log.level=debug
        - --web.listen-address=:9115
      volumes:
      - name: config
        configMap:
          name: blackbox-exporter
      nodeSelector:
        monitorexport: "blackbox"
      tolerations:
      - key: "node-role.kubernetes.io/master"
        effect: "NoSchedule"
```



### 七、grafana根据不同prometheus server统计数据

场景：由于采集的数据量巨大，所以部署了多台prometheus server服务器。根据业务领域(k8s集群)分片采集，减轻prometheus server单节点的压力。

#### 1.首先，在grafana中添加多个prometheus server数据源。

截图中的Prometheus就是我不同机房的数据源

![Prometheus 监控 kubernetes](D:\学习资料\笔记\k8s\k8s图\watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)



#### 2.在dashboard的变量中，添加datasource变量

![Prometheus 监控 kubernetes](D:\学习资料\笔记\k8s\k8s图\12)



#### 3.在panel中，选择datasource变量作为数据源

![Prometheus 监控 kubernetes](D:\学习资料\笔记\k8s\k8s图\135)



























































































































































# Kubernetes Ingress



## Canary deployment using the nginx-ingress controller



> In the following example, we shift traffic between 2 applications using the [canary annotations of the Nginx ingress controller](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#canary).

### Steps to follow

1. version 1 is serving traffic
2. deploy version 2
3. create a new "canary" ingress with traffic splitting enabled
4. wait enought time to confirm that version 2 is stable and not throwing unexpected errors
5. delete the canary ingress
6. point the main application ingress to send traffic to version 2
7. shutdown version 1



### config

**mandatory.yaml**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---

kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
kind: ConfigMap
apiVersion: v1
metadata:
  name: tcp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
kind: ConfigMap
apiVersion: v1
metadata:
  name: udp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress-serviceaccount
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: nginx-ingress-clusterrole
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - endpoints
      - nodes
      - pods
      - secrets
    verbs:
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - services
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - "extensions"
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - events
    verbs:
      - create
      - patch
  - apiGroups:
      - "extensions"
    resources:
      - ingresses/status
    verbs:
      - update

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:
  name: nginx-ingress-role
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - pods
      - secrets
      - namespaces
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - configmaps
    resourceNames:
      # Defaults to "<election-id>-<ingress-class>"
      # Here: "<ingress-controller-leader>-<nginx>"
      # This has to be adapted if you change either parameter
      # when launching the nginx-ingress-controller.
      - "ingress-controller-leader-nginx"
    verbs:
      - get
      - update
  - apiGroups:
      - ""
    resources:
      - configmaps
    verbs:
      - create
  - apiGroups:
      - ""
    resources:
      - endpoints
    verbs:
      - get

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: nginx-ingress-role-nisa-binding
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: nginx-ingress-role
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: nginx-ingress-clusterrole-nisa-binding
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: nginx-ingress-clusterrole
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/part-of: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
      annotations:
        prometheus.io/port: "10254"
        prometheus.io/scrape: "true"
    spec:
      serviceAccountName: nginx-ingress-serviceaccount
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.22.0
          args:
            - /nginx-ingress-controller
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
            - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
            - --publish-service=$(POD_NAMESPACE)/ingress-nginx
            - --annotations-prefix=nginx.ingress.kubernetes.io
          securityContext:
            allowPrivilegeEscalation: true
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
            # www-data -> 33
            runAsUser: 33
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
            - name: https
              containerPort: 443
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1

---
```



**app-v1.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-v1
  labels:
    app: my-app
spec:
  ports:
  - name: http
    port: 80
    targetPort: http
  selector:
    app: my-app
    version: v1.0.0
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-v1
  labels:
    app: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
      version: v1.0.0
  template:
    metadata:
      labels:
        app: my-app
        version: v1.0.0
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9101"
    spec:
      containers:
      - name: my-app
        image: containersol/k8s-deployment-strategies
        ports:
        - name: http
          containerPort: 8080
        - name: probe
          containerPort: 8086
        env:
        - name: VERSION
          value: v1.0.0
        livenessProbe:
          httpGet:
            path: /live
            port: probe
          initialDelaySeconds: 5
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /ready
            port: probe
          periodSeconds: 5
```



**app-v2.yaml** 

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-v2
  labels:
    app: my-app
spec:
  ports:
  - name: http
    port: 80
    targetPort: http
  selector:
    app: my-app
    version: v2.0.0
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-v2
  labels:
    app: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
      version: v2.0.0
  template:
    metadata:
      labels:
        app: my-app
        version: v2.0.0
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9101"
    spec:
      containers:
      - name: my-app
        image: containersol/k8s-deployment-strategies
        ports:
        - name: http
          containerPort: 8080
        - name: probe
          containerPort: 8086
        env:
        - name: VERSION
          value: v2.0.0
        livenessProbe:
          httpGet:
            path: /live
            port: probe
          initialDelaySeconds: 5
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /ready
            port: probe
          periodSeconds: 5
```



**ingress-v1.yaml**

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: my-app
  labels:
    app: my-app
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: my-app.com
    http:
      paths:
      - backend:
          serviceName: my-app-v1
          servicePort: 80
```



**ingress-v2.yaml**

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: my-app
  labels:
    app: my-app
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: my-app.com
    http:
      paths:
      - backend:
          serviceName: my-app-v2
          servicePort: 80
```



**ingress-v2-canary.yaml**

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: my-app-canary
  labels:
    app: my-app
  annotations:
    kubernetes.io/ingress.class: "nginx"

    # Enable canary and send 10% of traffic to version 2
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
spec:
  rules:
  - host: my-app.com
    http:
      paths:
      - backend:
          serviceName: my-app-v2
          servicePort: 80
```





### In practice

```sh
# Deploy the ingress-nginx controller
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.22.0/deploy/mandatory.yaml

# Expose the ingress-nginx
$ kubectl expose deployment \
    -n ingress-nginx nginx-ingress-controller \
    --port 80 \
    --type LoadBalancer \
    --name ingress-nginx

# Wait for nginx to be running
$ kubectl rollout status deploy nginx-ingress-controller -n ingress-nginx -w
deployment "nginx-ingress-controller" successfully rolled out

# Deploy version 1 and expose the service via an ingress
$ kubectl apply -f ./app-v1.yaml -f ./ingress-v1.yaml

# Deploy version 2
$ kubectl apply -f ./app-v2.yaml

# In a different terminal you can check that requests are responding with version 1
$ nginx_service=$(minikube service ingress-nginx -n ingress-nginx --url)
$ while sleep 0.1; do curl "$nginx_service" -H "Host: my-app.com"; done

# Create a canary ingress in order to split traffic: 90% to v1, 10% to v2
$ kubectl apply -f ./ingress-v2-canary.yaml

# Now you should see that the traffic is being splitted

# When you are happy, delete the canary ingress
$ kubectl delete -f ./ingress-v2-canary.yaml

# Then finish the rollout, set 100% traffic to version 2
$ kubectl apply -f ./ingress-v2.yaml
```



### Cleanup

```sh
$ kubectl delete all -l app=my-app
```





## ingress-nginx 



Kubernetes Ingress 只是 Kubernetes 中的一个普通资源对象，需要一个对应的 Ingress 控制器来解析 Ingress 的规则，暴露服务到外部，比如 ingress-nginx，本质上来说它只是一个 Nginx Pod，然后将请求重定向到其他内部（ClusterIP）服务去，这个 Pod 本身也是通过 Kubernetes 服务暴露出去，最常见的方式是通过 LoadBalancer 来实现的。

我们可以使用 Ingress 来使内部服务暴露到集群外部去，它为你节省了宝贵的静态 IP，因为你不需要声明多个 LoadBalancer 服务了，此次，它还可以进行更多的额外配置。下面我们通过一个简单的示例来对 Ingress 进行一些说明吧。



### **简单HTTP server**

首先，我们先回到容器、Kubernetes 之前的时代。

之前我们更多会使用一个（Nginx）HTTP server 来托管我们的服务，它可以通过 HTTP 协议接收到一个特定文件路径的请求，然后在文件系统中检查这个文件路径，如果存在则就返回即可。

![image-20210105103400264](D:\学习资料\笔记\k8s\k8s图\image-20210105103400264.png)



例如，在 Nginx 中，我们可以通过下面的配置来实现这个功能。

```nginx
location /folder {
    root /var/www/;
    index index.html;
}
```

除了上面提到的功能之外，我们可以当 HTTP server 接收到请求后，将该请求重定向到另一个服务器（意味着它作为代理）去，然后将该服务器的响应重定向到客户端去。对于客户端来说，什么都没有改变，接收到的结果仍然还是请求的文件（如果存在的话）。

![image-20210105103540182](D:\学习资料\笔记\k8s\k8s图\image-20210105103540182.png)

同样如果在 Nginx 中，重定向可以配置成下面的样子：

```nginx
location /folder {
    proxy_pass http://second-nginx-server:8000;
}
```

这意味着 Nginx 可以从文件系统中提供文件，或者通过代理将响应重定向到其他服务器并返回它们的响应。



### **简单的Kubernetes示例**

#### **使用 ClusterIP 服务**

在 Kubernetes 中部署应用后，比如我们有两个 worker 节点，有两个服务 service-nginx 和 service-python，它们指向不同的 pods。这两个服务没有被调度到任何特定的节点上，也就是在任何节点上都有可能，如下图所示：

![image-20210105103917217](D:\学习资料\笔记\k8s\k8s图\image-20210105103917217.png)

在集群内部我们可以通过他们的 Service 服务来请求到 Nginx pods 和 Python pods 上去，现在我们想让这些服务也能从集群外部进行访问，按照前文提到的我们就需要将这些服务转换为 LoadBalancer 服务。



#### **使用 LoadBalancer 服务**

当然使用 LoadBalancer 服务的前提是我们的 Kubernetes 集群的托管服务商要能支持才行，如果支持我们可以将上面的 ClusterIP 服务转换为 LoadBalancer 服务，可以创建两个外部负载均衡器，将请求重定向到我们的节点 IP，然后重定向到内部的 ClusterIP 服务。

![image-20210105104054060](D:\学习资料\笔记\k8s\k8s图\image-20210105104054060.png)

我们可以看到两个 LoadBalancers 都有自己的 IP，如果我们向 LoadBalancer 22.33.44.55发送请求，它请被重定向到我们的内部的 service-nginx 服务去。如果发送请求到 77.66.55.44，它将被重定向到我们的内部的 service-python 服务。

这个确实很方便，但是要知道 IP 地址是比较稀有的，而且价格可不便宜。想象下我们 Kubernetes 集群中不只是两个服务，有很多的话，我们为这些服务创建 LoadBalancers 成本是不是就成倍增加了。

那么是否有另一种解决方案可以让我们只使用一个 LoadBalancer 就可以把请求转发给我们的内部服务呢？我们先通过手动（非 Kubernetes）的方式来探讨下这个问题。



#### **手动配置 Nginx 代理服务**

我们知道 Nginx 可以作为一个代理使用，所以我们可以很容易想到运行一个 Nginx 来代理我们的服务。如下图所示，我们新增了一个名为 service-nginx-proxy 的新服务，它实际上是我们唯一的一个 LoadBalancer 服务。service-nginx-proxy 仍然会指向一个或多个 Nginx-pod-endpoints（为了简单没有在图上标识），之前的另外两个服务转换为简单的 ClusterIP 服务了。

![image-20210105104244949](D:\学习资料\笔记\k8s\k8s图\image-20210105104244949.png)

可以看到我们只分配了一个 IP 地址为 11.22.33.44 的负载均衡器，对于不同的 http 请求路径我们用黄色来进行标记，他们的目标是一致的，只是包含的不同的请求 URL。

service-nginx-proxy 服务会根据请求的 URL 来决定他们应该将请求重定向到哪个服务去。

在上图中我们有两个背后的服务，分别用红色和蓝色进行了标记，红色会重定向到 service-nginx 服务，蓝色重定向到 service-python 服务。对应的 Nginx 代理配置如下所示：

```nginx
location /folder {
    proxy_pass http://service-nginx:3001;
}
location /other {
    proxy_pass http://service-python:3002;
}
```

只是目前我们需要去手动配置 service-nginx-proxy 服务，比如新增了一个请求路径需要路由到其他服务去，我们就需要去重新配置 Nginx 的配置让其生效，但是这个确实是一个可行的解决方案，只是有点麻烦而已。

而 Kubernetes Ingress 就是为了让我们的配置更加容易、更加智能、更容易管理出现的，所以在 Kubernetes 集群中我们会用 Ingress 来代替上面的手动配置的方式将服务暴露到集群外去。



### **使用Kubernetes Ingress**

现在我们将上面手动配置代理的方式转换为 Kubernetes Ingress 的方式，如下图所示，我们只是使用了一个预先配置好的 Nginx（Ingress），它已经为我们做了所有的代理重定向工作，这为我们节省了大量的手动配置工作了。

![image-20210105104502884](D:\学习资料\笔记\k8s\k8s图\image-20210105104502884.png)

这其实就已经说明了 Kubernetes Ingress 是什么，下面让我们来看看一些配置实例吧。



#### **安装 Ingress 控制器**

Ingress 只是 Kubernetes 的一种资源对象而已，在这个资源中我们可以去配置我们的服务路由规则，但是要真正去实现识别这个 Ingress 并提供代理路由功能，还需要安装一个对应的控制器才能实现。

```sh
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.24.1/deploy/mandatory.yaml
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.24.1/deploy/provider/cloud-generic.yaml
```

使用下面的命令，可以看到安装在命名空间 ingress-nginx 中的 k8s 资源。

![image-20210105104716180](D:\学习资料\笔记\k8s\k8s图\image-20210105104716180.png)

我们可以看到一个正常的 LoadBalancer 服务，有一个外部 IP 和一个所属的 pod，我们可以使用命令 kubectl exec 进入该 pod，里面包含一个预配置的 Nginx 服务器。

![image-20210105104959873](D:\学习资料\笔记\k8s\k8s图\image-20210105104959873.png)

其中的 nginx.conf 文件就包含各种代理重定向设置和其他相关配置。



#### **Ingress 配置示例**

我们所使用的 Ingress yaml 例子可以是这样的。

```yaml
# just example, not tested
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
  namespace: default
  name: test-ingress
spec:
  rules:
  - http:
      paths:
      - path: /folder
        backend:
          serviceName: service-nginx
          servicePort: 3001
  - http:
      paths:
      - path: /other
        backend:
          serviceName: service-python
          servicePort: 3002
```

和其他资源对象一样，通过 `kubectl create -f ingress.yaml` 来创建这个资源对象即可，创建完成后这个 Ingress 对象会被上面安装的 Ingress 控制器转换为对应的 Nginx 配置。

如果你的一个内部服务，即 Ingress 应该重定向到的服务，是在不同的命名空间里，怎么办？因为我们定义的 Ingress 资源是命名空间级别的。在 Ingress 配置中，只能重定向到同一命名空间的服务。

如果你定义了多个 Ingress yaml 配置，那么这些配置会被一个单一的Ingress 控制器合并成一个 Nginx 配置。也就是说所有的人都在使用同一个 LoadBalancer IP。



#### **配置 Ingress Nginx**

有时候我们需要对 Ingress Nginx 进行一些微调配置，我们可以通过 Ingress 资源对象中的 annotations 注解来实现，比如我们可以配置各种平时直接在 Nginx 中的配置选项。

```yaml
kind: Ingress
metadata:
  name: ingress
  annotations:
      kubernetes.io/ingress.class: nginx
      nginx.ingress.kubernetes.io/proxy-connect-timeout: '30'
      nginx.ingress.kubernetes.io/proxy-send-timeout: '500'
      nginx.ingress.kubernetes.io/proxy-read-timeout: '500'
      nginx.ingress.kubernetes.io/send-timeout: "500"
      nginx.ingress.kubernetes.io/enable-cors: "true"
      nginx.ingress.kubernetes.io/cors-allow-methods: "*"
      nginx.ingress.kubernetes.io/cors-allow-origin: "*"
...
```

此外也可以做更细粒度的规则配置，如下所示：

```yaml
nginx.ingress.kubernetes.io/configuration-snippet: |
  if ($host = 'www.qikqiak.com' ) {
    rewrite ^ https://qikqiak.com$request_uri permanent;
  }
```

这些注释都将被转换成 Nginx 配置，你可以通过手动连接(kubectl exec)到 nginx pod 中检查这些配置。

关于 ingress-nginx 更多的配置使用可以参考官方文档相关说明：

- https://github.com/kubernetes/ingress-nginx/tree/master/docs/user-guide/nginx-configuration
- https://github.com/kubernetes/ingress-nginx/blob/master/docs/user-guide/nginx-configuration/annotations.md#lua-resty-waf



#### **查看 ingress-nginx 日志**

要排查问题，通过查看 Ingress 控制器的日志非常有帮助。

![image-20210105152118940](D:\学习资料\笔记\k8s\k8s图\image-20210105152118940.png)



#### **使用 Curl 测试**

如果我们想测试 Ingress 重定向规则，最好使用 `curl -v [yourhost.com](http://yourhost.com)` 来代替浏览器，可以避免缓存等带来的问题。



#### **重定向规则**

在本文的示例中我们使用 /folder 和 /other/directory 等路径来重定向到不同的服务，此外我们也可以通过主机名来区分请求，比如将 api.myurl.com 和 site.myurl.com 重定向到不同的内部 ClusterIP 服务去。

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: simple-fanout-example
spec:
  rules:
  - host: api.myurl.com
    http:
      paths:
      - path: /foo
        backend:
          serviceName: service1
          servicePort: 4200
      - path: /bar
        backend:
          serviceName: service2
          servicePort: 8080
  - host: website.myurl.com
    http:
      paths:
      - path: /
        backend:
          serviceName: service3
          servicePort: 3333
```



#### **SSL/HTTPS**

可能我们想让网站使用安全的 HTTPS 服务，Kubernetes Ingress 也提供了简单的 TLS 校验，这意味着它会处理所有的 SSL 通信、解密/校验 SSL 请求，然后将这些解密后的请求发送到内部服务去。

如果你的多个内部服务使用相同（可能是通配符）的 SSL 证书，这样我们就只需要在 Ingress 上配置一次，而不需要在内部服务上去配置，Ingress 可以使用配置的 TLS Kubernetes Secret 来配置 SSL 证书。

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: tls-example-ingress
spec:
  tls:
  - hosts:
    - sslexample.foo.com
    secretName: testsecret-tls
  rules:
    - host: sslexample.foo.com
      http:
        paths:
        - path: /
          backend:
            serviceName: service1
            servicePort: 80
```

不过需要注意的是如果你在不同的命名空间有多个 Ingress 资源，那么你的 TLS secret 也需要在你使用的 Ingress 资源的所有命名空间中可用。



### 总结

这里我们简单介绍了 Kubernetes Ingress 的原理，简单来说：它不过是一种轻松配置 Nginx 服务器的方法，它可以将请求重定向到其他内部服务去。这为我们节省了宝贵的静态 IP 和 LoadBalancers 资源。

另外需要注意的是还有其他的 Kubernetes Ingress 类型，它们内部没有设置 Nginx 服务，但可能使用其他代理技术，一样也可以实现上面的所有功能。

> 原文链接：https://codeburst.io/kubernetes-ingress-simply-visually-explained-d9cad44e4419





##  ingress-nginx 的使用



Ingress Controller 可以理解为一个监听器，通过不断地监听 kube-apiserver，实时的感知后端 Service、Pod 的变化，当得到这些信息变化后，Ingress Controller 再结合 Ingress 的配置，更新反向代理负载均衡器，达到服务发现的作用。其实这点和服务发现工具 consul、 consul-template 非常类似。

![image-20210105154710325](D:\学习资料\笔记\k8s\k8s图\image-20210105154710325.png)

现在可以供大家使用的 Ingress Controller 有很多，比如 traefik、nginx-controller、Kubernetes Ingress Controller for Kong、HAProxy Ingress controller，当然你也可以自己实现一个 Ingress Controller，现在普遍用得较多的是 traefik 和 nginx-controller，traefik 的性能较 nginx-controller 差，但是配置使用要简单许多，我们这里会重点给大家介绍 nginx-controller 的使用。



### 安装

NGINX Ingress Controller 是使用 Kubernetes Ingress 资源对象构建的，用 ConfigMap 来存储 Nginx 配置的一种 Ingress Controller 实现。

要使用 Ingress 对外暴露服务，就需要提前安装一个 Ingress Controller，我们这里就先来安装 NGINX Ingress Controller，由于 nginx-ingress 所在的节点需要能够访问外网，这样域名可以解析到这些节点上直接使用，所以需要让 nginx-ingress 绑定节点的 80 和 443 端口，所以可以使用 hostPort 来进行访问，当然对于线上环境来说为了保证高可用，一般是需要运行多个 nginx-ingress 实例的，然后可以用一个 nginx/haproxy 作为入口，通过 keepalived 来访问边缘节点的 vip 地址。

> 所谓的边缘节点即集群内部用来向集群外暴露服务能力的节点，集群外部的服务通过该节点来调用集群内部的服务，边缘节点是集群内外交流的一个 Endpoint。

所以我们这里需要更改下资源清单文件：

```sh
$ helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
$ helm repo update
$ helm fetch ingress-nginx/ingress-nginx
$ tar -xvf ingress-nginx-3.15.2.tgz
```

我们这里测试环境只有 master1 节点可以访问外网，这里我们就直接讲 ingress-nginx 固定到 master1 节点上，采用 hostNetwork 模式(生产环境可以使用 LB + DaemonSet hostNetwork 模式)。然后新建一个名为 values-prod.yaml 的 Values 文件，用来覆盖 ingress-nginx 默认的 Values 值，对应的数据如下所示：

```yaml
# values-prod.yaml
controller:
  name: controller
  image:
    repository: cnych/ingress-nginx
    tag: "v0.41.2"
    digest: 

  dnsPolicy: ClusterFirstWithHostNet
   
  hostNetwork: true

  publishService:  # hostNetwork 模式下设置为false，通过节点IP地址上报ingress status数据
    enabled: false

  kind: DaemonSet

  nodeSelector: 
    role: lb

  service:  # HostNetwork 模式不需要创建service
    enabled: false

defaultBackend:
  enabled: true
  name: defaultbackend
  image:
    repository: cnych/ingress-nginx-defaultbackend
    tag: "1.5"
```

然后使用如下命令安装 ingress-nginx 应用到 ingress-nginx 的命名空间中：

```yaml
$ kubectl create ns ingress-nginx

$ helm install --namespace ingress-nginx ingress-nginx ./ingress-nginx -f ./ingress-nginx/values-prod.yaml 
NAME: ingress-nginx
LAST DEPLOYED: Fri Dec 11 14:19:05 2020
NAMESPACE: ingress-nginx
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The ingress-nginx controller has been installed.
Get the application URL by running these commands:
  export POD_NAME=$(kubectl --namespace ingress-nginx get pods -o jsonpath="{.items[0].metadata.name}" -l "app=ingress-nginx,component=controller,release=ingress-nginx")
  kubectl --namespace ingress-nginx port-forward $POD_NAME 8080:80
  echo "Visit http://127.0.0.1:8080 to access your application."

An example Ingress that makes use of the controller:

  apiVersion: networking.k8s.io/v1beta1
  kind: Ingress
  metadata:
    annotations:
      kubernetes.io/ingress.class: nginx
    name: example
    namespace: foo
  spec:
    rules:
      - host: www.example.com
        http:
          paths:
            - backend:
                serviceName: exampleService
                servicePort: 80
              path: /
    # This section is only required if TLS is to be enabled for the Ingress
    tls:
        - hosts:
            - www.example.com
          secretName: example-tls

If TLS is enabled for the Ingress, a Secret containing the certificate and key must also be provided:

  apiVersion: v1
  kind: Secret
  metadata:
    name: example-tls
    namespace: foo
  data:
    tls.crt: <base64 encoded cert>
    tls.key: <base64 encoded key>
  type: kubernetes.io/tls
```

部署完成后查看 Pod 的运行状态：

```sh
$ kubectl get svc -n ingress-nginx
NAME                                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
ingress-nginx-controller-admission   ClusterIP   10.110.143.167   <none>        443/TCP   2m21s
ingress-nginx-defaultbackend         ClusterIP   10.104.156.141   <none>        80/TCP    2m21s

$ kubectl get pods -n ingress-nginx
NAME                                            READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-596955b554-vfmhq       1/1     Running   0          31s
ingress-nginx-defaultbackend-7bf9445d94-lkgw5   1/1     Running   0          3m52s

$ POD_NAME=$(kubectl get pods -l app.kubernetes.io/name=ingress-nginx -n ingress-nginx -o jsonpath='{.items[0].metadata.name}')

$ kubectl exec -it $POD_NAME -n ingress-nginx -- /nginx-ingress-controller --version
-------------------------------------------------------------------------------
NGINX Ingress controller
  Release:       v0.41.2
  Build:         d8a93551e6e5798fc4af3eb910cef62ecddc8938
  Repository:    https://github.com/kubernetes/ingress-nginx
  nginx version: nginx/1.19.4

-------------------------------------------------------------------------------
```

当看到上面的信息证明 ingress-nginx 部署成功了。



### Ingress

安装成功后，现在我们来为一个 nginx 应用创建一个 Ingress 资源，如下所示：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      app: my-nginx
  template:
    metadata:
      labels:
        app: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    app: my-nginx
spec:
  ports:
  - port: 80
    protocol: TCP
    name: http
  selector:
    app: my-nginx
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: my-nginx
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: ngdemo.qikqiak.com  # 将域名映射到 my-nginx 服务
    http:
      paths:
      - path: /
        backend:
          serviceName: my-nginx  # 将所有请求发送到 my-nginx 服务的 80 端口
          servicePort: 80     # 不过需要注意大部分Ingress controller都不是直接转发到Service
                              # 而是只是通过Service来获取后端的Endpoints列表，直接转发到Pod，这样可以减少网络跳转，提高性能
```

直接创建上面的资源对象：

```sh
$ kubectl apply -f ngdemo.yaml
deployment.apps "my-nginx" created
service "my-nginx" created
ingress.extensions "my-nginx" created
```

注意我们在 Ingress 资源对象中添加了一个 annotations：`kubernetes.io/ingress.class: "nginx"`，这就是指定让这个 Ingress 通过 nginx-ingress 来处理。

上面资源创建成功后，然后我们可以将域名 `ngdemo.qikqiak.com` 解析到 `ingress-nginx` 所在的**边缘节点**中的任意一个，当然也可以在本地`/etc/hosts`中添加对应的映射也可以，然后就可以通过域名进行访问了。

下图显示了客户端是如果通过 Ingress Controller 连接到其中一个 Pod 的流程，客户端首先对 `ngdemo.qikqiak.com` 执行 DNS 解析，得到 Ingress Controller 所在节点的 IP，然后客户端向 Ingress Controller 发送 HTTP 请求，然后根据 Ingress 对象里面的描述匹配域名，找到对应的 Service 对象，并获取关联的 Endpoints 列表，将客户端的请求转发给其中一个 Pod。

![image-20210105163257095](D:\学习资料\笔记\k8s\k8s图\image-20210105163257095.png)



### URL Rewrite

NGINX Ingress Controller 很多高级的用法可以通过 Ingress 对象的 `annotation` 进行配置，比如常用的 URL Rewrite 功能，比如我们有一个 todo 的前端应用，代码位于 https://github.com/cnych/todo-app，直接部署这个应用进行测试：

```sh
$ kubectl apply -f https://github.com/cnych/todo-app/raw/master/k8s/mongo.yaml
$ kubectl apply -f https://github.com/cnych/todo-app/raw/master/k8s/web.yaml
$ kubectl get pods
NAME                        READY   STATUS    RESTARTS   AGE
mongo-5c9fd978bb-txn9j      1/1     Running   0          149m
todo-566957d785-tdgs6       1/1     Running   0          3m31s
......
$ kubectl get svc
NAME                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
kubernetes                 ClusterIP   10.96.0.1        <none>        443/TCP                      54d
mongo                      ClusterIP   10.96.95.11      <none>        27017/TCP                    150m
todo                       ClusterIP   10.111.105.47    <none>        3000/TCP                     145m
......
```

对应的 Ingress 资源对象如下所示：

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: todo
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: todo.qikqiak.com
    http:
      paths:
      - path: /
        backend:
          serviceName: todo
          servicePort: 3000
```

就是一个很常规的 `Ingress` 对象，部署该对象后，将域名解析后就可以正常访问到应用：

![image-20210105163523134](D:\学习资料\笔记\k8s\k8s图\image-20210105163523134.png)

现在我们需要对访问的 URL 路径做一个 Rewrite，比如在 PATH 中添加一个 app 的前缀，关于 Rewrite 的操作在 ingress-nginx 官方文档中也给出对应的说明:

![image-20210105163549999](D:\学习资料\笔记\k8s\k8s图\image-20210105163549999.png)

按照要求我们需要在 `path` 中匹配前缀 `app`，然后通过 `rewrite-target` 指定目标，修改后的 Ingress 对象如下所示：

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: todo
  namespace: default
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
  - host: todo.qikqiak.com
    http:
      paths:
      - backend:
          serviceName: todo
          servicePort: 3000
        path: /app(/|$)(.*)
```

更新后，我们可以遇见到直接访问域名肯定是不行了，因为我们没有匹配 `/` 的 path 路径：

![image-20210105163717905](D:\学习资料\笔记\k8s\k8s图\image-20210105163717905.png)



但是我们带上 `app` 的前缀再去访问:

![image-20210105163737680](D:\学习资料\笔记\k8s\k8s图\image-20210105163737680.png)

我们可以看到已经可以访问到页面内容了，这是因为我们在 `path` 中通过正则表达式 `/app(/|$)(.*)` 将匹配的路径设置成了 `rewrite-target` 的目标路径了，所以我们访问 `todo.qikqiak.com/app` 的时候实际上相当于访问的就是后端服务的 `/` 路径，但是我们也可以发现现在页面的样式没有了：

![image-20210105163822836](D:\学习资料\笔记\k8s\k8s图\image-20210105163822836.png)

这是因为应用的静态资源路径是在 `/stylesheets` 路径下面的，现在我们做了 url rewrite 过后，要正常访问也需要带上前缀才可以：`http://todo.qikqiak.com/stylesheets/screen.css`，对于图片或者其他静态资源也是如此，当然我们去更改页面引入静态资源的方式为相对路径也是可以的，但是毕竟要修改代码，这个时候我们可以借助 `ingress-nginx` 中的 `configuration-snippet` 来对静态资源做一次跳转，如下所示：

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: todo
  namespace: default
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/configuration-snippet: |
      rewrite ^/stylesheets/(.*)$ /app/stylesheets/$1 redirect;  # 添加 /app 前缀
      rewrite ^/images/(.*)$ /app/images/$1 redirect;  # 添加 /app 前缀
spec:
  rules:
  - host: todo.qikqiak.com
    http:
      paths:
      - backend:
          serviceName: todo
          servicePort: 3000
        path: /app(/|$)(.*)
```

更新 Ingress 对象后，这个时候我们刷新页面可以看到已经正常了：

![image-20210105163953061](D:\学习资料\笔记\k8s\k8s图\image-20210105163953061.png)

要解决我们访问主域名出现 404 的问题，我们可以给应用设置一个 `app-root` 的注解，这样当我们访问主域名的时候会自动跳转到我们指定的 `app-root` 目录下面，如下所示：

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: todo
  namespace: default
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/app-root: /app/
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/configuration-snippet: |
      rewrite ^/stylesheets/(.*)$ /app/stylesheets/$1 redirect;  # 添加 /app 前缀
      rewrite ^/images/(.*)$ /app/images/$1 redirect;  # 添加 /app 前缀
spec:
  rules:
  - host: todo.qikqiak.com
    http:
      paths:
      - backend:
          serviceName: todo
          servicePort: 3000
        path: /app(/|$)(.*)
```

这个时候我们更新应用后访问主域名 `http://todo.qikqiak.com` 就会自动跳转到 `http://todo.qikqiak.com/app/` 路径下面去了。但是还有一个问题是我们的 path 路径其实也匹配了 `/app` 这样的路径，可能我们更加希望我们的应用在最后添加一个 `/` 这样的 slash，同样我们可以通过 `configuration-snippet` 配置来完成，如下 Ingress 对象：

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: todo
  namespace: default
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/app-root: /app/
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/configuration-snippet: |
      rewrite ^(/app)$ $1/ redirect;
      rewrite ^/stylesheets/(.*)$ /app/stylesheets/$1 redirect;
      rewrite ^/images/(.*)$ /app/images/$1 redirect;
spec:
  rules:
  - host: todo.qikqiak.com
    http:
      paths:
      - backend:
          serviceName: todo
          servicePort: 3000
        path: /app(/|$)(.*)
```

更新后我们的应用就都会以 `/` 这样的 slash 结尾了。这样就完成了我们的需求，如果你原本对 nginx 的配置就非常熟悉的话应该可以很快就能理解这种配置方式了。



### Basic Auth

同样我们还可以在 Ingress Controller 上面配置一些基本的 Auth 认证，比如 Basic Auth，可以用 htpasswd 生成一个密码文件来验证身份验证。

```sh
$ htpasswd -c auth foo
New password:
Re-type new password:
Adding password for user foo
```

然后根据上面的 auth 文件创建一个 secret 对象：

```yaml
$ kubectl create secret generic basic-auth --from-file=auth
secret/basic-auth created

$ kubectl get secret basic-auth -o yaml
apiVersion: v1
data:
  auth: Zm9vOiRhcHIxJFNjcVhZcFN6JDc4Nm5ISFNaeDdwN2VscDM2WUo0YS8K
kind: Secret
metadata:
  creationTimestamp: "2019-12-08T06:40:39Z"
  name: basic-auth
  namespace: default
  resourceVersion: "9197951"
  selfLink: /api/v1/namespaces/default/secrets/basic-auth
  uid: 6b2aa299-b511-412e-85ea-d0e91e578af0
type: Opaque
```

然后对上面的 my-nginx 应用创建一个具有 Basic Auth 的 Ingress 对象：

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-with-auth
  annotations:
    # 认证类型
    nginx.ingress.kubernetes.io/auth-type: basic
    # 包含 user/password 定义的 secret 对象名
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    # 要显示的带有适当上下文的消息，说明需要身份验证的原因
    nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required - foo'
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - path: /
        backend:
          serviceName: my-nginx
          servicePort: 80
```

直接创建上面的资源对象，然后通过下面的命令或者在浏览器中直接打开配置的域名：

```sh
$ curl -v http://k8s.qikqiak.com -H 'Host: foo.bar.com'
* Rebuilt URL to: http://k8s.qikqiak.com/
*   Trying 123.59.188.12...
* TCP_NODELAY set
* Connected to k8s.qikqiak.com (123.59.188.12) port 80 (#0)
> GET / HTTP/1.1
> Host: foo.bar.com
> User-Agent: curl/7.54.0
> Accept: */*
>
< HTTP/1.1 401 Unauthorized
< Server: openresty/1.15.8.2
< Date: Sun, 08 Dec 2019 06:44:35 GMT
< Content-Type: text/html
< Content-Length: 185
< Connection: keep-alive
< WWW-Authenticate: Basic realm="Authentication Required - foo"
<
<html>
<head><title>401 Authorization Required</title></head>
<body>
<center><h1>401 Authorization Required</h1></center>
<hr><center>openresty/1.15.8.2</center>
</body>
</html>
```

我们可以看到出现了 401 认证失败错误，然后带上我们配置的用户名和密码进行认证：

```sh
$ curl -v http://k8s.qikqiak.com -H 'Host: foo.bar.com' -u 'foo:foo'
* Rebuilt URL to: http://k8s.qikqiak.com/
*   Trying 123.59.188.12...
* TCP_NODELAY set
* Connected to k8s.qikqiak.com (123.59.188.12) port 80 (#0)
* Server auth using Basic with user 'foo'
> GET / HTTP/1.1
> Host: foo.bar.com
> Authorization: Basic Zm9vOmZvbw==
> User-Agent: curl/7.54.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Server: openresty/1.15.8.2
< Date: Sun, 08 Dec 2019 06:46:27 GMT
< Content-Type: text/html
< Content-Length: 612
< Connection: keep-alive
< Vary: Accept-Encoding
< Last-Modified: Tue, 19 Nov 2019 12:50:08 GMT
< ETag: "5dd3e500-264"
< Accept-Ranges: bytes
<
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

可以看到已经认证成功了。当然出来 Basic Auth 这一种简单的认证方式之外，NGINX Ingress Controller 还支持一些其他高级的认证，比如 OAUTH 认证之类的。



### 灰度发布

在日常工作中我们经常需要对服务进行版本更新升级，所以我们经常会使用到滚动升级、蓝绿发布、灰度发布等不同的发布操作。而 `ingress-nginx` 支持通过 Annotations 配置来实现不同场景下的灰度发布和测试，可以满足金丝雀发布、蓝绿部署与 A/B 测试等业务场景。ingress-nginx 的 Annotations 支持以下 4 种 Canary 规则：

- `nginx.ingress.kubernetes.io/canary-by-header`：基于 Request Header 的流量切分，适用于灰度发布以及 A/B 测试。当 Request Header 设置为 always 时，请求将会被一直发送到 Canary 版本；当 Request Header 设置为 never时，请求不会被发送到 Canary 入口；对于任何其他 Header 值，将忽略 Header，并通过优先级将请求与其他金丝雀规则进行优先级的比较。
- `nginx.ingress.kubernetes.io/canary-by-header-value`：要匹配的 Request Header 的值，用于通知 Ingress 将请求路由到 Canary Ingress 中指定的服务。当 Request Header 设置为此值时，它将被路由到 Canary 入口。该规则允许用户自定义 Request Header 的值，必须与上一个 annotation (即：canary-by-header) 一起使用。
- `nginx.ingress.kubernetes.io/canary-weight`：基于服务权重的流量切分，适用于蓝绿部署，权重范围 0 - 100 按百分比将请求路由到 Canary Ingress 中指定的服务。权重为 0 意味着该金丝雀规则不会向 Canary 入口的服务发送任何请求，权重为 100 意味着所有请求都将被发送到 Canary 入口。
- `nginx.ingress.kubernetes.io/canary-by-cookie`：基于 cookie 的流量切分，适用于灰度发布与 A/B 测试。用于通知 Ingress 将请求路由到 Canary Ingress 中指定的服务的cookie。当 cookie 值设置为 always 时，它将被路由到 Canary 入口；当 cookie 值设置为 never 时，请求不会被发送到 Canary 入口；对于任何其他值，将忽略 cookie 并将请求与其他金丝雀规则进行优先级的比较。

> 需要注意的是金丝雀规则按优先顺序进行排序：`canary-by-header - > canary-by-cookie - > canary-weight`



总的来说可以把以上的四个 annotation 规则划分为以下两类：

- 基于权重的 Canary 规则

![image-20210105164956753](D:\学习资料\笔记\k8s\k8s图\image-20210105164956753.png)

- 基于用户请求的 Canary 规则

![image-20210105165048146](D:\学习资料\笔记\k8s\k8s图\image-20210105165048146.png)



下面我们通过一个示例应用来对灰度发布功能进行说明。

#### **第一步. 部署 Production 应用**

首先创建一个 production 环境的应用资源清单：

```yaml
# production.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: production
  labels:
    app: production
spec:
  selector:
    matchLabels:
      app: production
  template:
    metadata:
      labels:
        app: production
    spec:
      containers:
      - name: production
        image: cnych/echoserver
        ports:
        - containerPort: 8080
        env:
          - name: NODE_NAME
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
          - name: POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
          - name: POD_IP
            valueFrom:
              fieldRef:
                fieldPath: status.podIP
---
apiVersion: v1
kind: Service
metadata:
  name: production
  labels:
    app: production
spec:
  ports:
  - port: 80
    targetPort: 8080
    name: http
  selector:
    app: production
```

然后创建一个用于 production 环境访问的 Ingress 资源对象：

```yaml
# production-ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: production
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: echo.qikqiak.com
    http:
      paths:
      - backend:
          serviceName: production
          servicePort: 80
```

直接创建上面的几个资源对象：

```sh
$ kubectl apply -f production.yaml
$ kubectl apply -f production-ingress.yaml
$ kubectl get pods -l app=production
NAME                         READY   STATUS    RESTARTS   AGE
production-856d5fb99-d6bds   1/1     Running   0          2m50s
$ kubectl get ingress              
NAME         CLASS    HOSTS                ADDRESS        PORTS   AGE
production   <none>   echo.qikqiak.com     10.151.30.11   80      90s
```

应用部署成功后，将域名 `echo.qikqiak.com` 映射到 master1 节点（ingress-nginx 所在的节点）的外网 IP，然后即可正常访问应用了：

```sh
$ curl http://echo.qikqiak.com


Hostname: production-856d5fb99-d6bds

Pod Information:
 node name: node1
 pod name: production-856d5fb99-d6bds
 pod namespace: default
 pod IP: 10.244.1.111

Server values:
 server_version=nginx: 1.13.3 - lua: 10008

Request Information:
 client_address=10.244.0.0
 method=GET
 real path=/
 query=
 request_version=1.1
 request_scheme=http
 request_uri=http://echo.qikqiak.com:8080/

Request Headers:
 accept=*/*
 host=echo.qikqiak.com
 user-agent=curl/7.64.1
 x-forwarded-for=171.223.99.184
 x-forwarded-host=echo.qikqiak.com
 x-forwarded-port=80
 x-forwarded-proto=http
 x-real-ip=171.223.99.184
 x-request-id=e680453640169a7ea21afba8eba9e116
 x-scheme=http

Request Body:
 -no body in request-
```



#### **第二步. 创建 Canary 版本**

参考将上述 Production 版本的 `production.yaml` 文件，再创建一个 Canary 版本的应用。

```yaml
# canary.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: canary
  labels:
    app: canary
spec:
  selector:
    matchLabels:
      app: canary
  template:
    metadata:
      labels:
        app: canary
    spec:
      containers:
      - name: canary
        image: cnych/echoserver
        ports:
        - containerPort: 8080
        env:
          - name: NODE_NAME
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
          - name: POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
          - name: POD_IP
            valueFrom:
              fieldRef:
                fieldPath: status.podIP
---
apiVersion: v1
kind: Service
metadata:
  name: canary
  labels:
    app: canary
spec:
  ports:
  - port: 80
    targetPort: 8080
    name: http
  selector:
    app: canary
```

接下来就可以通过配置 Annotation 规则进行流量切分了。



#### **第三步. Annotation 规则配置**

**1. 基于权重**：基于权重的流量切分的典型应用场景就是蓝绿部署，可通过将权重设置为 0 或 100 来实现。例如，可将 Green 版本设置为主要部分，并将 Blue 版本的入口配置为 Canary。最初，将权重设置为 0，因此不会将流量代理到 Blue 版本。一旦新版本测试和验证都成功后，即可将 Blue 版本的权重设置为 100，即所有流量从 Green 版本转向 Blue。

创建一个基于权重的 Canary 版本的应用路由 Ingress 对象。

```yaml
# canary-ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: canary
  annotations:
    kubernetes.io/ingress.class: nginx 
    nginx.ingress.kubernetes.io/canary: "true"   # 要开启灰度发布机制，首先需要启用 Canary
    nginx.ingress.kubernetes.io/canary-weight: "30"  # 分配30%流量到当前Canary版本
spec:
  rules:
  - host: echo.qikqiak.com
    http:
      paths:
      - backend:
          serviceName: canary
          servicePort: 80
```

直接创建上面的资源对象即可：

```sh
$ kubectl apply -f canary.yaml
$ kubectl apply -f canary-ingress.yaml
$ kubectl get pods
NAME                         READY   STATUS    RESTARTS   AGE
canary-66cb497b7f-48zx4      1/1     Running   0          7m48s
production-856d5fb99-d6bds   1/1     Running   0          21m
......
$ kubectl get svc
NAME                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
canary                     ClusterIP   10.106.91.106    <none>        80/TCP                       8m23s
production                 ClusterIP   10.105.182.15    <none>        80/TCP                       22m
......
$ kubectl get ingress
NAME         CLASS    HOSTS                ADDRESS        PORTS   AGE
canary       <none>   echo.qikqiak.com     10.151.30.11   80      108s
production   <none>   echo.qikqiak.com     10.151.30.11   80      22m
```

Canary 版本应用创建成功后，接下来我们在命令行终端中来不断访问这个应用，观察 Hostname 变化：

```sh
$ for i in $(seq 1 10); do curl -s echo.qikqiak.com | grep "Hostname"; done
Hostname: production-856d5fb99-d6bds
Hostname: canary-66cb497b7f-48zx4
Hostname: production-856d5fb99-d6bds
Hostname: production-856d5fb99-d6bds
Hostname: production-856d5fb99-d6bds
Hostname: production-856d5fb99-d6bds
Hostname: production-856d5fb99-d6bds
Hostname: canary-66cb497b7f-48zx4
Hostname: canary-66cb497b7f-48zx4
Hostname: production-856d5fb99-d6bds
```

由于我们给 Canary 版本应用分配了 30% 左右权重的流量，所以上面我们访问10次有3次访问到了 Canary 版本的应用，符合我们的预期。



**2. 基于 Request Header**: 基于 Request Header 进行流量切分的典型应用场景即灰度发布或 A/B 测试场景。

在上面的 Canary 版本的 Ingress 对象中新增一条 annotation 配置 `nginx.ingress.kubernetes.io/canary-by-header: canary`（这里的 value 可以是任意值），使当前的 Ingress 实现基于 Request Header 进行流量切分，由于 `canary-by-header` 的优先级大于 `canary-weight`，所以会忽略原有的 `canary-weight` 的规则。

```yaml
annotations:
  kubernetes.io/ingress.class: nginx 
  nginx.ingress.kubernetes.io/canary: "true"   # 要开启灰度发布机制，首先需要启用 Canary
  nginx.ingress.kubernetes.io/canary-by-header: canary  # 基于header的流量切分
  nginx.ingress.kubernetes.io/canary-weight: "30"  # 会被忽略，因为配置了 canary-by-headerCanary版本
```

更新上面的 Ingress 资源对象后，我们在请求中加入不同的 Header 值，再次访问应用的域名。

> 注意：当 Request Header 设置为 never 或 always 时，请求将不会或一直被发送到 Canary 版本，对于任何其他 Header 值，将忽略 Header，并通过优先级将请求与其他 Canary 规则进行优先级的比较。

```sh
$ for i in $(seq 1 10); do curl -s -H "canary: never" echo.qikqiak.com | grep "Hostname"; done
Hostname: production-856d5fb99-d6bds
Hostname: production-856d5fb99-d6bds
Hostname: production-856d5fb99-d6bds
Hostname: production-856d5fb99-d6bds
Hostname: production-856d5fb99-d6bds
Hostname: production-856d5fb99-d6bds
Hostname: production-856d5fb99-d6bds
Hostname: production-856d5fb99-d6bds
Hostname: production-856d5fb99-d6bds
Hostname: production-856d5fb99-d6bds
```

这里我们在请求的时候设置了 `canary: never` 这个 Header 值，所以请求没有发送到 Canary 应用中去。如果设置为其他值呢：

```sh
$ for i in $(seq 1 10); do curl -s -H "canary: other-value" echo.qikqiak.com | grep "Hostname"; done
Hostname: production-856d5fb99-d6bds
Hostname: production-856d5fb99-d6bds
Hostname: canary-66cb497b7f-48zx4
Hostname: production-856d5fb99-d6bds
Hostname: production-856d5fb99-d6bds
Hostname: production-856d5fb99-d6bds
Hostname: production-856d5fb99-d6bds
Hostname: canary-66cb497b7f-48zx4
Hostname: production-856d5fb99-d6bds
Hostname: canary-66cb497b7f-48zx4
```

由于我们请求设置的 Header 值为 `canary: other-value`，所以 ingress-nginx 会通过优先级将请求与其他 Canary 规则进行优先级的比较，我们这里也就会进入 `canary-weight: "30"` 这个规则去。

这个时候我们可以在上一个 annotation (即 `canary-by-header`）的基础上添加一条 `nginx.ingress.kubernetes.io/canary-by-header-value: user-value` 这样的规则，就可以将请求路由到 Canary Ingress 中指定的服务了。

```yaml
annotations:
  kubernetes.io/ingress.class: nginx 
  nginx.ingress.kubernetes.io/canary: "true"   # 要开启灰度发布机制，首先需要启用 Canary
  nginx.ingress.kubernetes.io/canary-by-header-value: user-value  
  nginx.ingress.kubernetes.io/canary-by-header: canary  # 基于header的流量切分
  nginx.ingress.kubernetes.io/canary-weight: "30"  # 分配30%流量到当前Canary版本
```

同样更新 Ingress 对象后，重新访问应用，当 Request Header 满足 `canary: user-value`时，所有请求就会被路由到 Canary 版本：

```sh
$ for i in $(seq 1 10); do curl -s -H "canary: user-value" echo.qikqiak.com | grep "Hostname"; done
Hostname: canary-66cb497b7f-48zx4
Hostname: canary-66cb497b7f-48zx4
Hostname: canary-66cb497b7f-48zx4
Hostname: canary-66cb497b7f-48zx4
Hostname: canary-66cb497b7f-48zx4
Hostname: canary-66cb497b7f-48zx4
Hostname: canary-66cb497b7f-48zx4
Hostname: canary-66cb497b7f-48zx4
Hostname: canary-66cb497b7f-48zx4
Hostname: canary-66cb497b7f-48zx4
```



**3. 基于 Cookie**：与基于 Request Header 的 annotation 用法规则类似。例如在 A/B 测试场景下，需要让地域为北京的用户访问 Canary 版本。那么当 cookie 的 annotation 设置为 `nginx.ingress.kubernetes.io/canary-by-cookie: "users_from_Beijing"`，此时后台可对登录的用户请求进行检查，如果该用户访问源来自北京则设置 cookie `users_from_Beijing` 的值为 `always`，这样就可以确保北京的用户仅访问 Canary 版本。

同样我们更新 Canary 版本的 Ingress 资源对象，采用基于 Cookie 来进行流量切分，

```yaml
annotations:
  kubernetes.io/ingress.class: nginx 
  nginx.ingress.kubernetes.io/canary: "true"   # 要开启灰度发布机制，首先需要启用 Canary
  nginx.ingress.kubernetes.io/canary-by-cookie: "users_from_Beijing"  # 基于 cookie
  nginx.ingress.kubernetes.io/canary-weight: "30"  # 会被忽略，因为配置了 canary-by-cookie
```

更新上面的 Ingress 资源对象后，我们在请求中设置一个 `users_from_Beijing=always` 的 Cookie 值，再次访问应用的域名。

```sh
$ for i in $(seq 1 10); do curl -s -b "users_from_Beijing=always" echo.qikqiak.com | grep "Hostname"; done
Hostname: canary-66cb497b7f-48zx4
Hostname: canary-66cb497b7f-48zx4
Hostname: canary-66cb497b7f-48zx4
Hostname: canary-66cb497b7f-48zx4
Hostname: canary-66cb497b7f-48zx4
Hostname: canary-66cb497b7f-48zx4
Hostname: canary-66cb497b7f-48zx4
Hostname: canary-66cb497b7f-48zx4
Hostname: canary-66cb497b7f-48zx4
Hostname: canary-66cb497b7f-48zx4
```

我们可以看到应用都被路由到了 Canary 版本的应用中去了，如果我们将这个 Cookie 值设置为 never，则不会路由到 Canary 应用中。



### HTTPS

如果我们需要用 HTTPS 来访问我们这个应用的话，就需要监听 443 端口了，同样用 HTTPS 访问应用必然就需要证书，这里我们用 `openssl` 来创建一个自签名的证书：

```sh
$ openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=foo.bar.com"
```

然后通过 Secret 对象来引用证书文件：

```sh
# 要注意证书文件名称必须是 tls.crt 和 tls.key
$ kubectl create secret tls foo-tls --cert=tls.crt --key=tls.key
secret/who-tls created
```

这个时候我们就可以创建一个 HTTPS 访问应用的：

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-with-auth
  annotations:
    # 认证类型
    nginx.ingress.kubernetes.io/auth-type: basic
    # 包含 user/password 定义的 secret 对象名
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    # 要显示的带有适当上下文的消息，说明需要身份验证的原因
    nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required - foo'
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - path: /
        backend:
          serviceName: my-nginx
          servicePort: 80
  tls:
  - hosts:
    - foo.bar.com
    secretName: foo-tls
```

除了自签名证书或者购买正规机构的 CA 证书之外，我们还可以通过 `letsencrypt` 来自动生成合法的证书。



### CertManager 自动 HTTPS

#### 安装配置

cert-manager 是一个云原生证书管理开源项目，用于在 Kubernetes 集群中提供 HTTPS 证书并自动续期，支持 `Let's Encrypt/HashiCorp/Vault` 这些免费证书的签发。在 Kubernetes 中，可以通过 Kubernetes Ingress 和 Let's Encrypt 实现外部服务的自动化 HTTPS。

![image-20210105174909300](D:\学习资料\笔记\k8s\k8s图\image-20210105174909300.png)

上面是官方给出的架构图，可以看到 cert-manager 在 Kubernetes 中定义了两个自定义类型资源：`Issuer(ClusterIssuer)` 和 `Certificate`。

- 其中 `Issuer` 代表的是证书颁发者，可以定义各种提供者的证书颁发者，当前支持基于 `Let's Encrypt/HashiCorp/Vault` 和 CA 的证书颁发者，还可以定义不同环境下的证书颁发者。
- 而 `Certificate` 代表的是生成证书的请求，一般其中存入生成证书的元信息，如域名等等。

一旦在 Kubernetes 中定义了上述两类资源，部署的 cert-manager 则会根据 `Issuer` 和 `Certificate` 生成 TLS 证书，并将证书保存进 Kubernetes 的 Secret 资源中，然后在 Ingress 资源中就可以引用到这些生成的 Secret 资源作为 TLS 证书使用，对于已经生成的证书，还会定期检查证书的有效期，如即将超过有效期，还会自动续期。

要在 Kubernetes 集群上安装 cert-manager 也非常简单，官方提供了一个单一的资源清单文件，包含了所有的资源对象，所以直接安装即可：

```sh
# Kubernetes 1.16+
$ kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.1.0/cert-manager.yaml

# Kubernetes <1.16
$ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.1.0/cert-manager-legacy.yaml
```

上面的命令会创建一个名为 cert-manager 的命名空间，安装大量的 CRD 以及 AdmissionWebhook 对象，可以通过如下命令来查看是否安装成功：

```sh
$ kubectl get pods -n cert-manager
NAME                                      READY   STATUS    RESTARTS   AGE
cert-manager-5597cff495-q6rzh             1/1     Running   0          5m31s
cert-manager-cainjector-bd5f9c764-5sc7d   1/1     Running   0          5m31s
cert-manager-webhook-5f57f59fbc-mvcq4     1/1     Running   0          5m30s
```

正常情况下可以看到 `cert-manager`、`cert-manager-cainjector` 以及 `cert-manager-webhook` 这几个 Pod 处于 Running 状态。我们可以通过下面的测试来验证下是否可以签发基本的证书类型，创建一个 Issuer 资源对象来测试 webhook 工作是否正常(在开始签发证书之前，必须在群集中至少配置一个 Issuer 或 ClusterIssuer 资源)：

```yaml
$ cat <<EOF > test-selfsigned.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: cert-manager-test
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: test-selfsigned
  namespace: cert-manager-test
spec:
  selfSigned: {}  # 配置自签名的证书机构类型
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: selfsigned-cert
  namespace: cert-manager-test
spec:
  dnsNames:
  - example.com
  secretName: selfsigned-cert-tls
  issuerRef:
    name: test-selfsigned
EOF
```

这里我们创建了一个名为 `cert-manager-test` 的命名空间，创建了一个自签名的 Issuer 证书颁发机构，然后使用这个 Issuer 来创建一个证书请求的 Certificate 对象，直接创建上面的资源清单即可：

```sh
$ kubectl apply -f test-selfsigned.yaml
namespace/cert-manager-test created
issuer.cert-manager.io/test-selfsigned created
certificate.cert-manager.io/selfsigned-cert created
```

创建完成后可以检查新创建的证书状态，在 cert-manager 处理证书请求之前，可能需要稍微等几秒：

```sh
$ kubectl describe certificate -n cert-manager-test
Name:         selfsigned-cert
Namespace:    cert-manager-test
......
Spec:
  Dns Names:
    example.com
  Issuer Ref:
    Name:       test-selfsigned
  Secret Name:  selfsigned-cert-tls
Status:
  Conditions:
    Last Transition Time:  2020-12-12T03:29:07Z
    Message:               Certificate is up to date and has not expired
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2021-03-12T03:29:06Z
  Not Before:              2020-12-12T03:29:06Z
  Renewal Time:            2021-02-10T03:29:06Z
  Revision:                1
Events:
  Type    Reason     Age   From          Message
  ----    ------     ----  ----          -------
  Normal  Issuing    6s    cert-manager  Issuing certificate as Secret does not exist
  Normal  Generated  6s    cert-manager  Stored new private key in temporary Secret resource "selfsigned-cert-sppz7"
  Normal  Requested  6s    cert-manager  Created new CertificateRequest resource "selfsigned-cert-z4nvl"
  Normal  Issuing    5s    cert-manager  The certificate has been successfully issued
```

从上面的 Events 事件中我们可以证书已经成功签发了，生成的证书存放在一个名为 `selfsigned-cert-tls` 的 Secret 对象下面：

```sh
$ kubectl get secret -n cert-manager-test                            
NAME                  TYPE                                  DATA   AGE
default-token-t928x   kubernetes.io/service-account-token   3      64s
selfsigned-cert-tls   kubernetes.io/tls                     3      63s

$ kubectl get secret -n cert-manager-test selfsigned-cert-tls -o yaml
apiVersion: v1
data:
  ca.crt: ......
  tls.crt: ......
  tls.key: ......
kind: Secret
......
  name: selfsigned-cert-tls
  namespace: cert-manager-test
  resourceVersion: "13461084"
  selfLink: /api/v1/namespaces/cert-manager-test/secrets/selfsigned-cert-tls
  uid: 42e456dc-6d34-4269-b207-f1f3bd50db8b
type: kubernetes.io/tls
```

到这里证明我们的 cert-manager 已经安装成功了。我们需要注意的是 cert-manager 的功能非常强大，不只是可以支持 ACME 类型的证书签发，还支持其他众多的类型，比如 SelfSigned(自签名)、CA、Vault、Venafi、External、ACME，只是我们一般主要是使用 ACME 来帮我们生成自动化的证书。

下面我们就来使用 cert-manager 结合 ingress-nginx 为 Kubernetes 应用自动签发 `Let's Encrypt` 类型的 HTTPS 证书。



#### 自动化 HTTPS

`Let's Encrypt` 使用 `ACME` 协议来校验域名是否真的属于你，校验成功后就可以自动颁发免费证书，证书有效期只有 90 天，在到期前需要再校验一次来实现续期，而 cert-manager 是可以自动续期的，所以事实上并不用担心证书过期的问题。目前主要有 HTTP 和 DNS 两种校验方式。



**HTTP-01 校验**

`HTTP-01` 的校验是通过给你域名指向的 HTTP 服务增加一个临时 location，在校验的时候 `Let's Encrypt` 会发送 http 请求到 `http://<YOUR_DOMAIN>/.well-known/acme-challenge/<TOKEN>`，其中 `YOUR_DOMAIN` 就是被校验的域名，`TOKEN` 是 cert-manager 生成的一个路径，它通过修改 Ingress 规则来增加这个临时校验路径并指向提供 TOKEN 的服务。`Let's Encrypt` 会对比 TOKEN 是否符合预期，校验成功后就会颁发证书了，不过这种方法不支持泛域名证书。

使用 HTTP 校验这种方式，首先需要将域名解析配置好，也就是需要保证 ACME 服务端可以正常访问到你的 HTTP 服务。这里我们以上面的 TODO 应用为例，我们已经将 `todo.qikqiak.com` 域名做好了正确的解析。

由于 Let's Encrypt 的生产环境有着严格的接口调用限制，所以一般我们需要先在 staging 环境测试通过后，再切换到生产环境。首先我们创建一个全局范围 staging 环境使用的 HTTP-01 校验方式的证书颁发机构：

```yaml
$ cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging-http01
spec:
  acme:
    # ACME 服务端地址
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    # 用于 ACME 注册的邮箱
    email: icnych@gmail.com
    # 用于存放 ACME 帐号 private key 的 secret
    privateKeySecretRef:
      name: letsencrypt-staging-http01
    solvers:
    - http01: # ACME HTTP-01 类型
        ingress:
          class: nginx  # 指定ingress的名称
EOF
```

同样再创建一个用于生产环境使用的 ClusterIssuer 对象：

```yaml
$ cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-http01
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: icnych@gmail.com
    privateKeySecretRef:
      name: letsencrypt-http01
    solvers:
    - http01: 
        ingress:
          class: nginx
EOF
```

创建完成后可以看到两个 ClusterIssuer 对象：

```sh
$ kubectl get clusterissuer
NAME                         READY   AGE
letsencrypt-http01           True    17s
letsencrypt-staging-http01   True    3m12s
```

有了 `Issuer/ClusterIssuer` 证书颁发机构，接下来我们就可以生成免费证书了，cert-manager 给我们提供了 `Certificate` 这个用于生成证书的自定义资源对象，不过这个对象需要在一个具体的命名空间下使用，证书最终会在这个命名空间下以 Secret 的资源对象存储。我们这里是要结合 ingress-nginx 一起使用，实际上我们只需要修改 Ingress 对象，添加上 cert-manager 的相关注解即可，不需要手动创建 Certificate 对象了，修改上面的 todo 应用的 Ingress 资源对象，如下所示：

```yaml
➜ cat <<EOF | kubectl apply -f -
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: todo
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-staging-http01"  # 使用哪个issuer
spec:
  tls:
  - hosts:
    - todo.qikqiak.com     # TLS 域名
    secretName: todo-tls   # 用于存储证书的 Secret 对象名字 
  rules:
  - host: todo.qikqiak.com
    http:
      paths:
      - path: /
        backend:
          serviceName: todo
          servicePort: 3000
EOF
```

更新了上面的 Ingress 对象后，正常就会自动为我们创建一个 Certificate 对象：

```sh
$ kubectl get certificate                              
NAME       READY   SECRET     AGE
todo-tls   False   todo-tls   34s
```

在校验过程中会自动创建一个 Ingress 对象用于 ACME 服务端访问：

```sh
$ kubectl get ingress     
NAME                        CLASS    HOSTS                ADDRESS        PORTS     AGE
cm-acme-http-solver-tgwlb   <none>   todo.qikqiak.com     10.151.30.11   80        25s
my-nginx                    <none>   ngdemo.qikqiak.com   10.151.30.11   80        23h
todo                        <none>   todo.qikqiak.com     10.151.30.11   80, 443   33s
```

校验成功后会将证书保存到 todo-tls 的 Secret 对象中：

```sh
$ kubectl get certificate                         
NAME       READY   SECRET     AGE
todo-tls   True    todo-tls   21m

$ kubectl get secret                                                 
NAME                          TYPE                                  DATA   AGE
default-token-hpd7s           kubernetes.io/service-account-token   3      55d
todo-tls                      kubernetes.io/tls                     2      20m

$  kubectl describe certificate todo-tls 
Name:         todo-tls
Namespace:    default
......
Events:
  Type    Reason     Age   From          Message
  ----    ------     ----  ----          -------
  Normal  Issuing    22m   cert-manager  Issuing certificate as Secret does not exist
  Normal  Generated  22m   cert-manager  Stored new private key in temporary Secret resource "todo-tls-tr4pq"
  Normal  Requested  22m   cert-manager  Created new CertificateRequest resource "todo-tls-2gchg"
  Normal  Issuing    21m   cert-manager  The certificate has been successfully issued
```

证书自动获取成功后，现在就可以讲 ClusterIssuer 替换成生产环境的了：

```yaml
$ cat <<EOF | kubectl apply -f -
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: todo
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-http01"  # 使用生产环境的issuer
spec:
  tls:
  - hosts:
    - todo.qikqiak.com     # TLS 域名
    secretName: todo-tls   # 用于存储证书的 Secret 对象名字 
  rules:
  - host: todo.qikqiak.com
    http:
      paths:
      - path: /
        backend:
          serviceName: todo
          servicePort: 3000
EOF

$ kubectl get certificate                         
NAME       READY   SECRET     AGE
todo-tls   True    todo-tls   25m
```

校验成功后就可以自动获取真正的 HTTPS 证书了，现在在浏览器中访问 https://todo.qikqiak.com 就可以看到证书是有效的了。

![image-20210105175914994](D:\学习资料\笔记\k8s\k8s图\image-20210105175914994.png)

**DNS-01 校验**

`DNS-01` 的校验是通过 DNS 提供商的 API 拿到你的 DNS 控制权限， 在 `Let's Encrypt` 为 cert-manager 提供 TOKEN 后，cert-manager 将创建从该 TOKEN 和你的帐户密钥派生的 `TXT` 记录，并将该记录放在 `_acme-challenge.<YOUR_DOMAIN>`。然后 `Let's Encrypt` 将向 DNS 系统查询该记录，如果找到匹配项，就可以颁发证书，这种方法是支持泛域名证书的。



### 参考链接

- https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/
- https://v2-1.docs.kubesphere.io/docs/zh-CN/quick-start/ingress-canary/
- https://cert-manager.io/docs/tutorials/acme/ingress/





## Nginx Ingress on TKE 部署最佳实践



### 概述

开源的 Ingress Controller 的实现使用量最大的莫过于 Nginx Ingress 了，功能强大且性能极高。Nginx Ingress 有多种部署方式，本文将介绍 Nginx Ingress 在 TKE 上的一些部署方案，这几种方案的原理、各自优缺点以及一些选型和使用上的建议。



### 什么是 Nginx Ingress ?

在介绍如何部署 Nginx Ingress 之前，我们先简单了解下什么是 Nginx Ingress。

Nginx Ingress 是 Kubernetes Ingress 的一种实现，它通过 watch Kubernetes 集群的 Ingress 资源，将 Ingress 规则转换成 Nginx 的配置，然后让 Nginx 来进行 7 层的流量转发:

![image-20210106092313629](D:\学习资料\笔记\k8s\k8s图\image-20210106092313629.png)

实际 Nginx Ingress 有两种实现：

1. https://github.com/kubernetes/ingress-nginx
2. https://github.com/nginxinc/kubernetes-ingress

第一种是 Kubernetes 开源社区的实现，第二种是 Nginx 官方的实现，我们通常用的是 Kubernetes 社区的实现，这也是本文所关注的重点。



### 有哪些部署方案 ?

那么如何在 TKE 上部署 Nginx Ingress 呢？主要有三种方案，下面分别介绍下这几种方案及其部署方法。



#### **方案一：Deployment + LB**

在 TKE 上部署 Nginx Ingress 最简单的方式就是将 Nginx Ingress Controller 以 Deployment 的方式部署，并且为其创建 LoadBalancer 类型的 Service(可以是自动创建 CLB 也可以是绑定已有 CLB)，这样就可以让 CLB 接收外部流量，然后转发到 Nginx Ingress 内部：

![image-20210106165546101](D:\学习资料\笔记\k8s\k8s图\image-20210106165546101.png)

当前 TKE 上 LoadBalancer 类型的 Service 默认实现是基于 NodePort，CLB 会绑定各节点的 NodePort 作为后端 rs，将流量转发到节点的 NodePort，然后节点上再通过 Iptables 或 IPVS 将请求路由到 Service 对应的后端 Pod，这里的 Pod 就是 Nginx Ingress Controller 的 Pod。后续如果有节点的增删，CLB 也会自动更新节点 NodePort 的绑定。

这是最简单的一种方式，可以直接通过下面命令安装:

```sh
$ kubectl create ns nginx-ingress
$ kubectl apply -f https://raw.githubusercontent.com/TencentCloudContainerTeam/manifest/master/nginx-ingress/nginx-ingress-deployment.yaml -n nginx-ingress
```



#### **方案二：Daemonset + HostNetwork + LB**

方案一虽然简单，但是流量会经过一层 NodePort，会多一层转发。这种方式有一些缺点：

1. 转发路径较长，流量到了 NodePort 还会再经过 Kubernetes 内部负载均衡，通过 Iptables 或 IPVS 转发到 Nginx，会增加一点网络耗时。
2. 经过 NodePort 必然发生 SNAT，如果流量过于集中容易导致源端口耗尽或者 conntrack 插入冲突导致丢包，引发部分流量异常。
3. 每个节点的 NodePort 也充当一个负载均衡器，CLB 如果绑定大量节点的 NodePort，负载均衡的状态就分散在每个节点上，容易导致全局负载不均。
4. CLB 会对 NodePort 进行健康探测，探测包最终会被转发到 Nginx Ingress 的 Pod，如果 CLB 绑定的节点多，Nginx Ingress 的 Pod 少，会导致探测包对 Nginx Ingress 造成较大的压力。

我们可以让 Nginx Ingress 使用 hostNetwork，CLB 直接绑节点 IP + 端口(80,443)， 这样就不用走 NodePort；由于使用 hostNetwork，Nginx Ingress 的 pod 就不能被调度到同一节点避免端口监听冲突。通常做法是提前规划好，选取部分节点作为边缘节点，专门用于部署 Nginx Ingress，为这些节点打上 label，然后 Nginx Ingress 以 DaemonSet 方式部署在这些节点上。下面是架构图：

![image-20210106165927129](D:\学习资料\笔记\k8s\k8s图\image-20210106165927129.png)

安装步骤：

1. 将规划好的用于部署 Nginx Ingress 的节点打上 label: `kubectl label node 10.0.0.3 nginx-ingress=true`(注意替换节点名称)。
2. 将 Nginx Ingress 部署在这些节点上:

```sh
$ kubectl create ns nginx-ingress
$ kubectl apply -f https://raw.githubusercontent.com/TencentCloudContainerTeam/manifest/master/nginx-ingress/nginx-ingress-daemonset-hostnetwork.yaml -n nginx-ingress
```

3. 手动创建 CLB，创建 80 和 443 端口的 TCP 监听器，分别绑定部署了 Nginx Ingress 的这些节点的 80 和 443 端口。



#### **方案三：Deployment + LB 直通 Pod**

方案二虽然相比方案一有一些优势，但同时也引入了手动维护 CLB 和 Nginx Ingress 节点的运维成本，需要提前规划好 Nginx Ingress 的节点，增删 Nginx Ingress 节点时需要手动在 CLB 控制台绑定和解绑节点，也无法支持自动扩缩容。如果你的网络模式是 VPC-CNI，那么所有的 Pod 都使用的弹性网卡，弹性网卡的 Pod 是支持 CLB 直接绑 Pod 的，可以绕过 NodePort，并且不用手动管理 CLB，支持自动扩缩容:

![image-20210106170342508](D:\学习资料\笔记\k8s\k8s图\image-20210106170342508.png)

如果你的网络模式是 Global Router(大多集群都是这种模式)，你可以为集群开启 VPC-CNI 的支持，即两种网络模式混用，在集群信息页可打开：

![image-20210106170530094](D:\学习资料\笔记\k8s\k8s图\image-20210106170530094.png)

确保集群支持 VPC-CNI 之后，可以使用下面命令安装 Nginx Ingress:

```sh
$ kubectl create ns nginx-ingress
$ kubectl apply -f https://raw.githubusercontent.com/TencentCloudContainerTeam/manifest/master/nginx-ingress/nginx-ingress-deployment-eni.yaml -n nginx-ingress
```



### 如何选型？

前面介绍了 Nginx Ingress 在 TKE 上部署的三种方案，也说了各种方案的优缺点，这里做一个简单汇总下，给出一些选型建议：

1. 方案一较为简单通用，但在大规模和高并发场景可能有一点性能问题。如果对性能要求不那么严格，可以考虑使用这种方案。
2. 方案二使用 hostNetwork 性能好，但需要手动维护 CLB 和 Nginx Ingress 节点，也无法实现自动扩缩容，通常不太建议用这种方案。
3. 方案三性能好，而且不需要手动维护 CLB，是最理想的方案。它需要集群支持 VPC-CNI，如果你的集群本身用的 VPC-CNI 网络插件，或者用的 Global Router 网络插件并开启了 VPC-CNI 的支持(两种模式混用)，那么建议直接使用这种方案。



### 如何支持内网 Ingress ?

方案二由于是手动管理 CLB，自行创建 CLB 时可以选择用公网还是内网；方案一和方案三默认会创建公网 CLB，如果要用内网，可以改下部署 YAML，给 `nginx-ingress-controller` 这个 Service 加一个 key 为 `service.kubernetes.io/qcloud-loadbalancer-internal-subnetid`，value 为内网 CLB 所被创建的子网 id 的 annotation，示例:

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.kubernetes.io/qcloud-loadbalancer-internal-subnetid: subnet-xxxxxx # value 替换为集群所在 vpc 的其中一个子网 id
  labels:
    app: nginx-ingress
    component: controller
  name: nginx-ingress-controller
```



### 如何复用已有 LB ?

方案一和方案三默认会自动创建新的 CLB，Ingress 的流量入口地址取决于新创建出来的 CLB 的 IP 地址。如果业务对入口地址有依赖，比如配置了 DNS 解析到之前的 CLB IP，不希望切换 IP；或者想使用包年包月的 CLB (默认创建是按量计费)，那么也可以让 Nginx Ingress 绑定已有的 CLB。

操作方法同样也是修改下部署 yaml，给 `nginx-ingress-controller` 这个 Service 加一个 key 为 `service.kubernetes.io/tke-existed-lbid`，value 为 CLB ID 的 annotation，示例:

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.kubernetes.io/tke-existed-lbid: lb-6swtxxxx # value 替换为 CLB 的 ID
  labels:
    app: nginx-ingress
    component: controller
  name: nginx-ingress-controller
```



### Nginx Ingress 公网带宽有多大？

有同学可能会问：我的 Nginx Ingress 的公网带宽到底有多大？能否支撑住我服务的并发量？

这里需要普及一下，腾讯云账号有带宽上移和非带宽上移两种类型：

1. 非带宽上移，是指带宽在云主机(CVM)上管理。
2. 带宽上移，是指带宽上移到了 CLB 或 IP 上管理。

具体来讲，如果你的账号是非带宽上移类型，Nginx Ingress 使用公网 CLB，那么 Nginx Ingress 的公网带宽是 CLB 所绑定的 TKE 节点的带宽之和；如果使用方案三，CLB 直通 Pod，也就是 CLB 不是直接绑的 TKE 节点，而是弹性网卡，那么此时 Nginx Ingress 的公网带宽是所有 Nginx Ingress Controller Pod 被调度到的节点上的带宽之和。

如果你的账号是带宽上移类型就简单了，Nginx Ingress 的带宽就等于你所购买的 CLB 的带宽，默认是 10Mbps (按量计费)，你可以按需调整下。



### 如何创建 Ingress ?

目前还没有完成对 Nginx Ingress 的产品化支持，所以如果是在 TKE 上自行了部署 Nginx Ingress，想要使用 Nginx Ingress 来管理 Ingress，目前是无法通过在 TKE 控制台(网页) 上进行操作的，只有通过 YAML 的方式来创建，并且需要给每个 Ingress 都指定 Ingress Class 的 annotation，示例:

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: test-ingress
  annotations:
    kubernetes.io/ingress.class: nginx # 这里是重点
spec:
  rules:
  - host: *
    http:
      paths:
      - path: /
        backend:
          serviceName: nginx-v1
          servicePort: 80
```



### 如何监控？

通过上面的方法安装的 Nginx Ingress，已经暴露了 metrics 端口，可以被 Prometheus 采集。如果集群内安装了 prometheus-operator，可以使用下面的 ServiceMonitor 来采集 Nginx Ingress 的监控数据:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: nginx-ingress-controller
  namespace: nginx-ingress
  labels:
    app: nginx-ingress
    component: controller
spec:
  endpoints:
  - port: metrics
    interval: 10s
  namespaceSelector:
    matchNames:
    - nginx-ingress
  selector:
    matchLabels:
      app: nginx-ingress
      component: controller
```

这里也给个原生 Prometheus 配置的示例:

```yaml
 - job_name: nginx-ingress
      scrape_interval: 5s
      kubernetes_sd_configs:
      - role: endpoints
        namespaces:
          names:
          - nginx-ingress
      relabel_configs:
      - action: keep
        source_labels:
        - __meta_kubernetes_service_label_app
        - __meta_kubernetes_service_label_component
        regex: nginx-ingress;controller
      - action: keep
        source_labels:
        - __meta_kubernetes_endpoint_port_name
        regex: metrics
```

有了数据后，我们再给 grafana 配置一下面板来展示数据，Nginx Ingress 社区提供了面板：https://github.com/kubernetes/ingress-nginx/tree/master/deploy/grafana/dashboards

我们直接复制 json 导入到 grafana 即可导入面板。其中，`nginx.json` 是展示 Nginx Ingress 各种常规监控的面板：

![image-20210106171440716](D:\学习资料\笔记\k8s\k8s图\image-20210106171440716.png)

`request-handling-performance.json` 是展示 Nginx Ingress 性能方面的监控面板：

![image-20210106171518303](D:\学习资料\笔记\k8s\k8s图\image-20210106171518303.png)



### 总结

本文梳理了 Nginx Ingress 在 TKE 上部署的三种方案以及许多实用的建议，对于想要在 TKE 上使用 Nginx Ingress 的同学是一个很好的参考。由于 Nginx Ingress 的使用需求量较大，我们也正在做 Nginx Ingress 的产品化支持， 可以实现一键部署，集成日志和监控能力，并且会对其进行性能优化。相信在不久的将来，我们就能够在 TKE 上更简单高效的使用 Nginx Ingress 了，敬请期待吧！



### 参考资料

1. TKE Service YAML 示例: https://cloud.tencent.com/document/product/457/45489#yaml-.E7.A4.BA.E4.BE.8B
2. TKE Service 使用已有 CLB: https://cloud.tencent.com/document/product/457/45491
3. 区分腾讯云账户类型: https://cloud.tencent.com/document/product/684/39903

































































































































































## Nginx Ingress 高并发实践



### 概述

Nginx Ingress Controller 基于 Nginx 实现了 Kubernetes Ingress API，Nginx 是公认的高性能网关，但如果不对其进行一些参数调优，就不能充分发挥出高性能的优势。之前我们在 [Nginx Ingress on TKE 部署最佳实践](http://mp.weixin.qq.com/s?__biz=MzU5Mzc0NDUyNg==&mid=2247483870&idx=1&sn=ee161e1cf40c07be1cb728626e6df6d4&chksm=fe0a863fc97d0f295d2cd0660a3df8de34eba964106a5c4ac965df48dcff2fee23adb7db042e&scene=21#wechat_redirect) 一文中讲了 Nginx Ingress 在 TKE 上部署最佳实践，涉及的部署 YAML 其实已经包含了一些性能方面的参数优化，只是没有提及，本文将继续展开介绍针对 Nginx Ingress 的一些全局配置与内核参数调优的建议，可用于支撑我们的高并发业务。



### 内核参数调优

我们先看下如何对 Nginx Ingress 进行内核参数调优，设置内核参数的方法可以用 initContainers 的方式，后面会有示例。



#### **调大连接队列的大小**

进程监听的 socket 的连接队列最大的大小受限于内核参数 `net.core.somaxconn`，在高并发环境下，如果队列过小，可能导致队列溢出，使得连接部分连接无法建立。要调大 Nginx Ingress 的连接队列，只需要调整 somaxconn 内核参数的值即可，但我想跟你分享下这背后的相关原理。

进程调用 listen 系统调用来监听端口的时候，还会传入一个 backlog 的参数，这个参数决定 socket 的连接队列大小，其值不得大于 somaxconn 的取值。Go 程序标准库在 listen 时，默认直接读取 somaxconn 作为队列大小，但 Nginx 监听 socket 时没有读取 somaxconn，而是有自己单独的参数配置。在 `nginx.conf` 中 listen 端口的位置，还有个叫 backlog 参数可以设置，它会决定 nginx listen 的端口的连接队列大小。

```nginx
server {
    listen  80  backlog=1024;
    ...
```

如果不设置，backlog 在 linux 上默认为 511:

```nginx
backlog=number
   sets the backlog parameter in the listen() call that limits the maximum length for the queue of pending connections. By default, backlog is set to -1 on FreeBSD, DragonFly BSD, and macOS, and to 511 on other platforms.
```

也就是说，即便你的 somaxconn 配的很高，nginx 所监听端口的连接队列最大却也只有 511，高并发场景下可能导致连接队列溢出。

不过这个在 Nginx Ingress 这里情况又不太一样，因为 Nginx Ingress Controller 会自动读取 somaxconn 的值作为 backlog 参数写到生成的 nginx.conf 中：https://github.com/kubernetes/ingress-nginx/blob/controller-v0.34.1/internal/ingress/controller/nginx.go#L592

也就是说，Nginx Ingress 的连接队列大小只取决于 somaxconn 的大小，这个值在 TKE 默认为 4096，建议给 Nginx Ingress 设为 65535: `sysctl -w net.core.somaxconn=65535`。



#### **扩大源端口范围**

高并发场景会导致 Nginx Ingress 使用大量源端口与 upstream 建立连接，源端口范围从 `net.ipv4.ip_local_port_range` 这个内核参数中定义的区间随机选取，在高并发环境下，端口范围小容易导致源端口耗尽，使得部分连接异常。TKE 环境创建的 Pod 源端口范围默认是 32768-60999，建议将其扩大，调整为 1024-65535: `sysctl -w net.ipv4.ip_local_port_range="1024 65535"`。



#### **TIME_WAIT 复用**

如果短连接并发量较高，它所在 netns 中 TIME_WAIT 状态的连接就比较多，而 TIME_WAIT 连接默认要等 2MSL 时长才释放，长时间占用源端口，当这种状态连接数量累积到超过一定量之后可能会导致无法新建连接。

所以建议给 Nginx Ingress 开启 TIME_WAIT 重用，即允许将 TIME_WAIT 连接重新用于新的 TCP 连接：`sysctl -w net.ipv4.tcp_tw_reuse=1`



#### **调大最大文件句柄数**

Nginx 作为反向代理，对于每个请求，它会与 client 和 upstream server 分别建立一个连接，即占据两个文件句柄，所以理论上来说 Nginx 能同时处理的连接数最多是系统最大文件句柄数限制的一半。

系统最大文件句柄数由 `fs.file-max` 这个内核参数来控制，TKE 默认值为 838860，建议调大：`sysctl -w fs.file-max=1048576`。



#### **配置示例**

给 Nginx Ingress Controller 的 Pod 添加 initContainers 来设置内核参数：

```yaml
      initContainers:
      - name: setsysctl
        image: busybox
        securityContext:
          privileged: true
        command:
        - sh
        - -c
        - |
          sysctl -w net.core.somaxconn=65535
          sysctl -w net.ipv4.ip_local_port_range="1024 65535"
          sysctl -w net.ipv4.tcp_tw_reuse=1
          sysctl -w fs.file-max=1048576
```



### 全局配置调优

除了内核参数需要调优，Nginx 本身的一些配置也需要进行调优，下面我们来详细看下。



#### **调高 keepalive 连接最大请求数**

Nginx 针对 client 和 upstream 的 keepalive 连接，均有 keepalive_requests 这个参数来控制单个 keepalive 连接的最大请求数，且默认值均为 100。当一个 keepalive 连接中请求次数超过这个值时，就会断开并重新建立连接。

如果是内网 Ingress，单个 client 的 QPS 可能较大，比如达到 10000 QPS，Nginx 就可能频繁断开跟 client 建立的 keepalive 连接，然后就会产生大量 TIME_WAIT 状态连接。我们应该尽量避免产生大量 TIME_WAIT 连接，所以，建议这种高并发场景应该增大 Nginx 与 client 的 keepalive 连接的最大请求数量，在 Nginx Ingress 的配置对应 `keep-alive-requests`，可以设置为 10000，参考：https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#keep-alive-requests

同样的，Nginx 与 upstream 的 keepalive 连接的请求数量的配置是 `upstream-keepalive-requests`，参考：https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#upstream-keepalive-requests

但是，一般情况应该不必配此参数，如果将其调高，可能导致负载不均，因为 Nginx 与 upstream 保持的 keepalive 连接过久，导致连接发生调度的次数就少了，连接就过于 "固化"，使得流量的负载不均衡。



#### **调高 keepalive 最大空闲连接数**

Nginx 针对 upstream 有个叫 keepalive 的配置，它不是 keepalive 超时时间，也不是 keepalive 最大连接数，而是 keepalive 最大空闲连接数。

它的默认值为 32，在高并发下场景下会产生大量请求和连接，而现实世界中请求并不是完全均匀的，有些建立的连接可能会短暂空闲，而空闲连接数多了之后关闭空闲连接，就可能导致 Nginx 与 upstream 频繁断连和建连，引发 TIME_WAIT 飙升。在高并发场景下可以调到 1000，参考：https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#upstream-keepalive-connections



#### **调高单个 worker 最大连接数**

`max-worker-connections` 控制每个 worker 进程可以打开的最大连接数，TKE 环境默认 16384，在高并发环境建议调高，比如设置到 65536，这样可以让 nginx 拥有处理更多连接的能力，参考：https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#max-worker-connections



#### **配置示例**

Nginx 全局配置通过 configmap 配置（Nginx Ingress Controller 会 watch 并自动 reload 配置）：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-ingress-controller
data:
  keep-alive-requests: "10000"   #nginx 与 client 保持的一个长连接能处理的请求数量，默认 100，高并发场景建议调高。
  upstream-keepalive-connections: "200"   #nginx 与 upstream 保持长连接的最大空闲连接数 (不是最大连接数)，默认 32，在高并发下场景下调大，避免频繁建联导致 TIME_WAIT 飙升。
  max-worker-connections: "65536"    #每个 worker 进程可以打开的最大连接数，默认 16384。
```



### 总结

本文分享了对 Nginx Ingress 进行性能调优的方法及其原理的解释，包括内核参数与 Nginx 本身的配置调优，更好的适配高并发的业务场景。



### 参考:

- nginx ingress 性能优化: https://www.nginx.com/blog/tuning-nginx/

- Nginx Ingress 配置参考：https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/
- Tuning NGINX for Performance：https://www.nginx.com/blog/tuning-nginx/
- ngx_http_upstream_module 官方文档：http://nginx.org/en/docs/http/ngx_http_upstream_module.html
- https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#max-worker-connections
- https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#upstream-keepalive-connections
- https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#keep-alive-requests





## 高性能 Nginx HTTPS 调优



### 为什么要优化 Ngin HTTPS 延迟

Nginx 常作为最常见的服务器，常被用作负载均衡 (Load Balancer)、反向代理 (Reverse Proxy)，以及网关 (Gateway) 等等。一个配置得当的 Nginx 服务器单机应该可以**期望承受住 50K 到 80K 左右**[1]每秒的请求，同时将 CPU 负载在可控范围内。

但在很多时候，负载并不是需要首要优化的重点。比如对于卡拉搜索来说，我们希望用户在每次击键的时候，可以体验即时搜索的感觉，也就是说，**每个搜索请求必须在 100ms - 200ms 的时间**内端对端地返回给用户，才能让用户搜索时没有“卡顿”和“加载”。因此，对于我们来说，优化请求延迟才是最重要的优化方向。

这篇文章中，我们先介绍 Nginx 中的 TLS 设置有哪些与请求延迟可能相关，如何调整才能最大化加速。然后我们用优化**卡拉搜索**[2] Nginx 服务器的实例来分享如何调整 Nginx TLS/SSL 设置，为首次搜索的用户提速 30% 左右。我们会详细讨论每一步我们做了一些什么优化，优化的动机和效果。希望可以对其它遇到类似问题的同学提供帮助。

照例，本文的 Nginx 设置文件放置于 github，欢迎直接使用: **高性能 Nginx HTTPS 调优**[3]



### TLS 握手和延迟

很多时候开发者会认为：如果不是绝对在意性能，那么了解底层和更细节的优化没有必要。这句话在很多时候是恰当的，因为很多时候复杂的底层逻辑必须包起来，才能让更高层的应用开发复杂度可控。比如说，如果你就只需要开发一个 APP 或者网站，可能并没有必要关注汇编细节，关注编译器如何优化你的代码——毕竟在苹果或者安卓上很多优化在底层就做好了。

那么，了解底层的 TLS 和应用层的 Nginx 延迟优化有什么关系呢？

答案是多数情况下，优化网络延迟其实是在尝试减少用户和服务器之间的数据传输次数，也就是所谓的 roundtrip。由于物理限制，北京到云南的光速传播差不多就是要跑 20 来毫秒，如果你不小心让数据必须多次往返于北京和云南之间，那么必然延迟就上去了。

因此如果你需要优化请求延迟，那么了解一点底层网络的上下文则会大有裨益，很多时候甚至是你是否可以轻松理解一个优化的关键。本文中我们不深入讨论太多 TCP 或者 TLS 机制的细节，如果有兴趣的话请参考 **High Performance Browser Networking**[4] 一书，可以免费阅读。

举个例子，下图中展示了如果你的服务启用了 HTTPS，在开始传输任何数据之前的数据传输情况。

![image-20210111213348498](D:\学习资料\笔记\k8s\k8s图\image-20210111213348498.png)



可以看到，在你的用户拿到他需要的数据前，底层的数据包就已经在用户和你的服务器之间跑了 3 个来回。

假设每次来回需要 28 毫秒的话，用户已经等了 224 毫秒之后才开始接收数据。

同时这个 28 毫秒其实是非常乐观的假设，在国内电信、联通和移动以及各种复杂的网络状况下，用户与服务器之间的延迟更不可控。另一方面，通常一个网页需要数十个请求，这些请求不一定可以全部并行，因此几十乘以 224 毫秒，页面打开可能就是数秒之后了。

所以，原则上如果可能的话，我们需要尽量减少用户和服务器之间的往返程 (roundtrip)，在下文的设置中，对于每个设置我们会讨论为什么这个设置有可能帮助减少往返程。



### Nginx 中的 TLS 设置

那么在 Nginx 设置中，怎样调整参数会减少延迟呢？

#### 开启 HTTP/2

HTTP/2 标准是从 Google 的 SPDY 上进行的改进，比起 HTTP 1.1 提升了不少性能，尤其是需要并行多个请求的时候可以显着减少延迟。在现在的网络上，一个网页平均需要请求几十次，而在 HTTP 1.1 时代浏览器能做的就是多开几个连接（通常是 6 个）进行并行请求，而 HTTP 2 中可以在一个连接中进行并行请求。HTTP 2 原生支持多个并行请求，因此大大减少了顺序执行的请求的往返程，可以首要考虑开启。

如果你想自己看一下 HTTP 1.1 和 HTTP 2.0 的速度差异，可以试一下：https://www.httpvshttps.com/。我的网络测试下来 HTTP/2 比 HTTP 1.1 快了 66%。

![image-20210111213514529](D:\学习资料\笔记\k8s\k8s图\image-20210111213514529.png)



在 Nginx 中开启 HTTP 2.0 非常简单，只需要增加一个 http2 标志即可

```
listen 443 ssl;

# 改为
listen 443 ssl http2;
```

如果你担心你的用户用的是旧的客户端，比如 Python 的 requests，暂时还不支持 HTTP 2 的话，那么其实不用担心。如果用户的客户端不支持 HTTP 2，那么连接会自动降级为 HTTP 1.1，保持了后向兼容。因此，所有使用旧 Client 的用户，仍然不受影响，而新的客户端则可以享受 HTTP/2 的新特性。



#### 如何确认你的网站或者 API 开启了 HTTP 2

在 Chrome 中打开开发者工具，点开 `Protocol` 之后在所有的请求中都可以看到请求用的协议了。如果 `protocol` 这列的值是 `h2` 的话，那么用的就是 HTTP 2 了

![image-20210111213626097](D:\学习资料\笔记\k8s\k8s图\image-20210111213626097.png)



当然另一个办法是直接用 `curl` 如果返回的 status 前有 `HTTP/2` 的话自然也就是 HTTP/2 开启了。

```sh
$ curl --http2 -I https://kalasearch.cn
HTTP/2 403
server: Tengine
content-type: application/xml
content-length: 264
date: Tue, 22 Dec 2020 18:38:46 GMT
x-oss-request-id: 5FE23D363ADDB93430197043
x-oss-cdn-auth: success
x-oss-server-time: 0
x-alicdn-da-ups-status: endOs,0,403
via: cache13.l2et2[148,0], cache10.l2ot7[291,0], cache4.us13[360,0]
timing-allow-origin: *
eagleid: 2ff6169816086623266688093e
```



#### 调整 Cipher 优先级

尽量挑选更新更快的 Cipher，有助于**减少延迟**[5]:

```sh
# 手动启用 cipher 列表
ssl_prefer_server_ciphers on;  # prefer a list of ciphers to prevent old and slow ciphers
ssl_ciphers 'EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH';
```



#### 启用 OCSP Stapling

在国内这可能是对使用 Let's Encrypt 证书的服务或网站影响最大的延迟优化了。如果不启用 OCSP Stapling 的话，在用户连接你的服务器的时候，有时候需要去验证证书。而因为一些不可知的原因（这个就不说穿了）**Let's Encrypt 的验证服务器并不是非常通畅**[6]，因此可以造成有时候**数秒甚至十几秒延迟的问题**[7]，这个问题在 iOS 设备上特别严重

解决这个问题的方法有两个：

1. 不使用 Let's Encrypt，可以尝试替换为阿里云提供的免费 DV 证书
2. 开启 OCSP Stapling

开启了 OCSP Stapling 的话，跑到证书验证这一步可以省略掉。省掉一个 roundtrip，特别是网络状况不可控的 roundtrip，可能可以将你的延迟大大减少。

在 Nginx 中启用 OCSP Stapling 也非常简单，只需要设置：

```nginx
ssl_stapling on;
ssl_stapling_verify on;
ssl_trusted_certificate /path/to/full_chain.pem;
```



#### 如何检测 OCSP Stapling 是否已经开启？

可以通过以下命令

```sh
openssl s_client -connect test.kalasearch.cn:443 -servername kalasearch.cn -status -tlsextdebug < /dev/null 2>&1 | grep -i "OCSP response"
```

来测试。如果结果为

```sh
OCSP response:
OCSP Response Data:
    OCSP Response Status: successful (0x0)
    Response Type: Basic OCSP Response
```

则表明已经开启。参考 **HTTPS 在 iPhone 上慢的问题**[8] 一文。



#### 调整 `ssl_buffer_size`

`ssl_buffer_size` 控制在发送数据时的 buffer 大小，默认设置是 16k。这个值越小，则延迟越小，而添加的报头之类会使 overhead 会变大，反之则延迟越大，overhead 越小。

因此如果你的服务是 **REST API**[9] 或者网站的话，将这个值调小可以减小延迟和 TTFB，但如果你的服务器是用来传输大文件的，那么可以维持 16k。关于这个值的讨论和更通用的 TLS Record Size 的讨论，可以参考：**Best value for nginx's ssl\*buffer\*size option**[10]

如果是网站或者 REST API，建议值为 4k，但是这个值的最佳取值显然会因为数据的不同而不一样，因此请尝试 2 - 16k 间不同的值。在 Nginx 中调整这个值也非常容易

```nginx
ssl_buffer_size 4k;
```



#### 启用 SSL Session 缓存

启用 SSL Session 缓存可以大大减少 TLS 的反复验证，减少 TLS 握手的 roundtrip。虽然 session 缓存会占用一定内存，但是用 1M 的内存就可以缓存 4000 个连接，可以说是非常非常划算的。同时，对于绝大多数网站和服务，要达到 4000 个同时连接本身就需要非常非常大的用户基数，因此可以放心开启。

这里 `ssl_session_cache` 设置为使用 50M 内存，以及 4 小时的连接超时关闭时间 `ssl_session_timeout`

```nginx
# Enable SSL cache to speed up for return visitors
ssl_session_cache   shared:SSL:50m; # speed up first time. 1m ~= 4000 connections
ssl_session_timeout 4h;
```



### 卡拉搜索如何减少 30% 的请求延迟

卡拉搜索是国内的 **Algolia**[11]，致力于帮助开发者快速搭建即时搜索功能(instant search)，做国内最快最易用的搜索即服务。

开发者接入后，所有搜索请求通过卡拉 API 即可直接返回给终端用户。为了让用户有即时搜索的体验，我们需要在用户每次击键后极短的时间内（通常是 100ms 到 200ms）将结果返回给用户。因此每次搜索需要可以达到 50 毫秒以内的引擎处理时间和 200 毫秒以内的端对端时间。

> 我们用豆瓣电影的数据做了一个电影搜索的 Demo，如果感兴趣的话欢迎体验一下即时搜索，尝试一下搜索“无间道”或者“大话西游”体验一下速度和相关度：https://movies-demo.kalasearch.cn/

对于每个请求只有 100 到 200 毫秒的延迟预算，我们必须把每一步的延迟都考虑在内。

简化一下，每个搜索请求需要经历的延迟有

![image-20210111214202218](D:\学习资料\笔记\k8s\k8s图\image-20210111214202218.png)

总延迟 = 用户请求到达服务器(T1) + 反代处理(Nginx T2) + 数据中心延迟(T3) + 服务器处理 (卡拉引擎 T4) + 用户请求返回(T3+T1)

在上述延迟中，T1 只与用户与服务器的物理距离相关，而 T3 非常小（参考**Jeff Dean Number**[12])可以忽略不计。

所以我们能控制的大致只有 T2 和 T4，即 Nginx 服务器的处理时间和卡拉的引擎处理时间。

Nginx 在这里作为反向代理，处理一些安全、流量控制和 TLS 的逻辑，而卡拉的引擎则是一个在 Lucene 基础上的倒排引擎。

我们首先考虑的第一个可能性是：延迟是不是来自卡拉引擎呢？

在下图展示的 **Grafana 仪表盘**[13]中，我们看到除了几个时不时的慢查询，搜索的 95% 服务器处理延迟小于 20 毫秒。对比同样的数据集上 benchmark 的 Elastic Search 引擎的 P95 搜索延迟则在 200 毫秒左右，所以排除了引擎速度慢的可能。

![image-20210111214318794](D:\学习资料\笔记\k8s\k8s图\image-20210111214318794.png)



而在阿里云监控中，我们设置了从全国各地向卡拉服务器发送搜索请求。我们终于发现 SSL 处理时间时常会超过 300 毫秒，也就是说在 T2 这一步，光处理 TLS 握手之类的事情，Nginx 已经用掉了我们所有的请求时间预算。

同时检查之后我们发现，在苹果设备上搜索速度格外慢，特别是第一次访问的设备。因此我们大致判断应该是因为我们使用的 Let's Encrypt 证书的问题。

我们按照上文中的步骤对 Nginx 设置进行了调整，并将步骤总结出来写了这篇文章。在调整了 Nginx TLS 的设置后，SSL 时间从平均的 140ms 降低到了 110ms 左右（全国所有省份联通和移动测试点），同时苹果设备上首次访问慢的问题也消失了。

![image-20210111214447366](D:\学习资料\笔记\k8s\k8s图\image-20210111214447366.png)



在调整过后，全国范围内测试的搜索延迟降低到了 150 毫秒左右。



### 总结

调整 Nginx 中的 TLS 设置对于使用 HTTPS 的服务和网站延迟有非常大的影响。本文中总结了 Nginx 中与 TLS 相关的设置，详细讨论各个设置可能对延迟的影响，并给出了调整建议。之后我们会继续讨论 HTTP/2 对比 HTTP 1.x 有哪些具体改进，以及在 REST API 使用 HTTP/2 有哪些优缺点，请继续关注。



### 参考资料

[1]期望承受住 50K 到 80K 左右: *https://github.com/denji/nginx-tuning*

[2]卡拉搜索: *https://kalasearch.cn/*

[3]高性能 Nginx HTTPS 调优: *https://github.com/Kalasearch/high-performance-nginx-tls-tuning*

[4]High Performance Browser Networking: *https://hpbn.co/*

[5]减少延迟: *https://stackoverflow.com/questions/36672261/how-to-reduce-ssl-time-of-website*

[6]Let's Encrypt 的验证服务器并不是非常通畅: *https://juejin.cn/post/6844904135653851150*

[7]数秒甚至十几秒延迟的问题: *https://jhuo.ca/post/ocsp-stapling-letsencrypt/*

[8]HTTPS 在 iPhone 上慢的问题: *https://www.jianshu.com/p/2fb12a80c33f*

[9]REST API: *https://kalasearch.cn/blog/rest-api-best-practices/*

[10]Best value for nginx's ssl*buffer*size option: *https://github.com/igrigorik/istlsfastyet.com/issues/63*

[11]Algolia: *https://kalasearch.cn/blog/algolia-alternative/*

[12]Jeff Dean Number: *https://gist.github.com/jboner/2841832*[

13]Grafana 仪表盘: *https://kalasearch.cn/blog/grafana-with-prometheus-tutorial*





## Traefik2.3.x 使用大全



Traefik 是一个开源的可以使服务发布变得轻松有趣的边缘路由器。它负责接收你系统的请求，然后使用合适的组件来对这些请求进行处理。

![image-20210112172601723](D:\学习资料\笔记\k8s\k8s图\image-20210112172601723.png)

除了众多的功能之外，Traefik 的与众不同之处还在于它会自动发现适合你服务的配置。当 Traefik 在检查你的服务时，会找到服务的相关信息并找到合适的服务来满足对应的请求。

Traefik 兼容所有主流的集群技术，比如 Kubernetes，Docker，Docker Swarm，AWS，Mesos，Marathon，等等；并且可以同时处理多种方式。（甚至可以用于在裸机上运行的比较旧的软件。）

使用 Traefik，不需要维护或者同步一个独立的配置文件：因为一切都会自动配置，实时操作的（无需重新启动，不会中断连接）。使用 Traefik，你可以花更多的时间在系统的开发和新功能上面，而不是在配置和维护工作状态上面花费大量时间。



### 核心概念

Traefik 是一个边缘路由器，是你整个平台的大门，拦截并路由每个传入的请求：它知道所有的逻辑和规则，这些规则确定哪些服务处理哪些请求；传统的反向代理需要一个配置文件，其中包含路由到你服务的所有可能路由，而 Traefik 会实时检测服务并自动更新路由规则，可以自动服务发现。



![image-20210112172729147](D:\学习资料\笔记\k8s\k8s图\image-20210112172729147.png)



首先，当启动 Traefik 时，需要定义 `entrypoints`（入口点），然后，根据连接到这些 entrypoints 的**路由**来分析传入的请求，来查看他们是否与一组**规则**相匹配，如果匹配，则路由可能会将请求通过一系列**中间件**转换过后再转发到你的**服务**上去。在了解 Traefik 之前有几个核心概念我们必须要了解：

- `Providers` 用来自动发现平台上的服务，可以是编排工具、容器引擎或者 key-value 存储等，比如 Docker、Kubernetes、File
- `Entrypoints` 监听传入的流量（端口等…），是网络入口点，它们定义了接收请求的端口（HTTP 或者 TCP）。
- `Routers` 分析请求（host, path, headers, SSL, …），负责将传入请求连接到可以处理这些请求的服务上去。
- `Services` 将请求转发给你的应用（load balancing, …），负责配置如何获取最终将处理传入请求的实际服务。
- `Middlewares` 中间件，用来修改请求或者根据请求来做出一些判断（authentication, rate limiting, headers, ...），中间件被附件到路由上，是一种在请求发送到你的**服务**之前（或者在服务的响应发送到客户端之前）调整请求的一种方法。



### 安装

-----

由于 Traefik 2.X 版本和之前的 1.X 版本不兼容，我们这里选择功能更加强大的 2.X 版本来和大家进行讲解，我们这里使用的是最新的镜像 `traefik:2.3.6`。

在 Traefik 中的配置可以使用两种不同的方式：

- 动态配置：完全动态的路由配置
- 静态配置：启动配置

`静态配置`中的元素（这些元素不会经常更改）连接到 providers 并定义 Treafik 将要监听的 entrypoints。

> 在 Traefik 中有三种方式定义静态配置：在配置文件中、在命令行参数中、通过环境变量传递

`动态配置`包含定义系统如何处理请求的所有配置内容，这些配置是可以改变的，而且是无缝热更新的，没有任何请求中断或连接损耗。

这里我们还是使用 Helm 来快速安装 traefik，首先获取 Helm Chart 包：

```sh
$ git clone https://github.com/traefik/traefik-helm-chart
```

创建一个定制的 values 配置文件：

```yaml
# values-prod.yaml
# Create an IngressRoute for the dashboard
ingressRoute:
  dashboard:
    enabled: false  # 禁用helm中渲染的dashboard，我们自己手动创建

# Configure ports
ports:
  web:
    port: 8000
    hostPort: 80  # 使用 hostport 模式
    # Use nodeport if set. This is useful if you have configured Traefik in a
    # LoadBalancer
    # nodePort: 32080
    # Port Redirections
    # Added in 2.2, you can make permanent redirects via entrypoints.
    # https://docs.traefik.io/routing/entrypoints/#redirection
    # redirectTo: websecure
  websecure:
    port: 8443
    hostPort: 443  # 使用 hostport 模式

# Options for the main traefik service, where the entrypoints traffic comes
# from.
service:  # 使用 hostport 模式就不需要Service了
  enabled: false

# Logs
# https://docs.traefik.io/observability/logs/
logs:
  general:
    level: DEBUG
    
tolerations:   # kubeadm 安装的集群默认情况下master是有污点，需要容忍这个污点才可以部署
- key: "node-role.kubernetes.io/master"
  operator: "Equal"
  effect: "NoSchedule"

nodeSelector:   # 固定到master1节点（该节点才可以访问外网）
  kubernetes.io/hostname: "master1"
```

这里我们使用 hostport 模式将 Traefik 固定到 master1 节点上，因为只有这个节点有外网 IP，所以我们这里 master1 是作为流量的入口点。直接使用上面的 values 文件安装 traefik：

```sh
$ helm install --namespace kube-system traefik ./traefik -f ./values-prod.yaml
NAME: traefik
LAST DEPLOYED: Thu Dec 24 11:23:51 2020
NAMESPACE: kube-system
STATUS: deployed
REVISION: 1
TEST SUITE: None

$ kubectl get pods -n kube-system -l app.kubernetes.io/name=traefik
NAME                       READY   STATUS    RESTARTS   AGE
traefik-78ff486794-64jbd   1/1     Running   0          3m15s
```

安装完成后我们可以通过查看 Pod 的资源清单来了解 Traefik 的运行方式：

```yaml
$ kubectl get pods traefik-78ff486794-64jbd -n kube-system -o yaml
apiVersion: v1
kind: Pod
metadata:
......
spec:
  containers:
  - args:
    - --global.checknewversion
    - --global.sendanonymoususage
    - --entryPoints.traefik.address=:9000/tcp
    - --entryPoints.web.address=:8000/tcp
    - --entryPoints.websecure.address=:8443/tcp
    - --api.dashboard=true
    - --ping=true
    - --providers.kubernetescrd
    - --providers.kubernetesingress
    - --accesslog=true
    - --accesslog.fields.defaultmode=keep
    - --accesslog.fields.headers.defaultmode=drop
...
```

其中 `entryPoints` 属性定义了 `web` 和 `websecure` 这两个入口点的，并开启 `kubernetesingress` 和 `kubernetescrd` 这两个 provider，也就是我们可以使用 Kubernetes 原本的 Ingress 资源对象，也可以使用 Traefik 自己扩展的 IngressRoute 这样的 CRD 资源对象。

我们可以首先创建一个用于 Dashboard 访问的 IngressRoute 资源清单：

```yaml
$ cat <<EOF | kubectl apply -f -
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-dashboard
  namespace: kube-system
spec:
  entryPoints:
  - web
  routes:
  - match: Host(`traefik.qikqiak.com`)  # 指定域名
    kind: Rule
    services:
    - name: api@internal
      kind: TraefikService  # 引用另外的 Traefik Service
EOF

$ kubectl get ingressroute -n kube-system
NAME                AGE
traefik-dashboard   19m
```

其中的 `TraefikService` 是 `Traefik Service` 的一个 CRD 实现，这里我们使用的 `api@internal` 这个 `TraefikService`，表示我们访问的是 Traefik 内置的应用服务。

部署完成后我们可以通过在本地 `/etc/hosts` 中添加上域名 `traefik.qikqiak.com` 的映射即可访问 Traefik 的 Dashboard 页面了：

![image-20210112174201429](D:\学习资料\笔记\k8s\k8s图\image-20210112174201429.png)





### ACME

Traefik 通过扩展 CRD 的方式来扩展 Ingress 的功能，除了默认的用 Secret 的方式可以支持应用的 HTTPS 之外，还支持自动生成 HTTPS 证书。

比如现在我们有一个如下所示的 `whoami` 应用：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: whoami
spec:
  ports:
    - protocol: TCP
      name: web
      port: 80
  selector:
    app: whoami
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: whoami
  labels:
    app: whoami
spec:
  replicas: 2
  selector:
    matchLabels:
      app: whoami
  template:
    metadata:
      labels:
        app: whoami
    spec:
      containers:
        - name: whoami
          image: containous/whoami
          ports:
            - name: web
              containerPort: 80
```

然后定义一个 IngressRoute 对象：

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: simpleingressroute
spec:
  entryPoints:
  - web
  routes:
  - match: Host(`who.qikqiak.com`) && PathPrefix(`/notls`)
    kind: Rule
    services:
    - name: whoami
      port: 80
```

通过 `entryPoints` 指定了我们这个应用的入口点是 `web`，也就是通过 80 端口访问，然后访问的规则就是要匹配 `who.qikqiak.com` 这个域名，并且具有 `/notls` 的路径前缀的请求才会被 `whoami` 这个 Service 所匹配。我们可以直接创建上面的几个资源对象，然后对域名做对应的解析后，就可以访问应用了：

![image-20210112174902047](D:\学习资料\笔记\k8s\k8s图\image-20210112174902047.png)



在 `IngressRoute` 对象中我们定义了一些匹配规则，这些规则在 Traefik 中有如下定义方式：

![image-20210112174930764](D:\学习资料\笔记\k8s\k8s图\image-20210112174930764.png)

如果我们需要用 HTTPS 来访问我们这个应用的话，就需要监听 `websecure` 这个入口点，也就是通过 443 端口来访问，同样用 HTTPS 访问应用必然就需要证书，这里我们用 `openssl` 来创建一个自签名的证书：

```sh
$ openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=who.qikqiak.com"
```

然后通过 Secret 对象来引用证书文件：

```sh
# 要注意证书文件名称必须是 tls.crt 和 tls.key
$ kubectl create secret tls who-tls --cert=tls.crt --key=tls.key
secret/who-tls created
```

这个时候我们就可以创建一个 HTTPS 访问应用的 IngressRoute 对象了：

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: ingressroutetls
spec:
  entryPoints:
    - websecure
  routes:
  - match: Host(`who.qikqiak.com`) && PathPrefix(`/tls`)
    kind: Rule
    services:
    - name: whoami
      port: 80
  tls:
    secretName: who-tls
```

创建完成后就可以通过 HTTPS 来访问应用了，由于我们是自签名的证书，所以证书是不受信任的：

![image-20210112180153381](D:\学习资料\笔记\k8s\k8s图\image-20210112180153381.png)

除了手动提供证书的方式之外 Traefik 同样也支持使用 `Let’s Encrypt` 自动生成证书，要使用 `Let’s Encrypt` 来进行自动化 HTTPS，就需要首先开启 `ACME`，开启 `ACME` 需要通过静态配置的方式，也就是说可以通过环境变量、启动参数等方式来提供。

ACME 有多种校验方式 `tlsChallenge`、`httpChallenge` 和 `dnsChallenge` 三种验证方式，之前更常用的是 http 这种验证方式，关于这几种验证方式的使用可以查看文档：https://www.qikqiak.com/traefik-book/https/acme/ 了解他们之间的区别。要使用 tls 校验方式的话需要保证 Traefik 的 443 端口是可达的，dns 校验方式可以生成通配符的证书，只需要配置上 DNS 解析服务商的 API 访问密钥即可校验。我们这里用 DNS 校验的方式来为大家说明如何配置 ACME。

我们可以重新修改 Helm 安装的 values 配置文件，添加如下所示的定制参数：

```yaml
# values-prod.yaml
additionalArguments:
# 使用 dns 验证方式
- --certificatesResolvers.ali.acme.dnsChallenge.provider=alidns
# 先使用staging环境进行验证，验证成功后再使用移除下面一行的配置
# - --certificatesResolvers.ali.acme.caServer=https://acme-staging-v02.api.letsencrypt.org/directory
# 邮箱配置
- --certificatesResolvers.ali.acme.email=ych_1024@163.com
# 保存 ACME 证书的位置
- --certificatesResolvers.ali.acme.storage=/data/acme.json

envFrom: 
- secretRef:
    name: traefik-alidns-secret
    # ALICLOUD_ACCESS_KEY
    # ALICLOUD_SECRET_KEY
    # ALICLOUD_REGION_ID

persistence:
  enabled: true  # 开启持久化
  accessMode: ReadWriteOnce
  size: 128Mi
  path: /data

# 由于上面持久化了ACME的数据，需要重新配置下面的安全上下文
securityContext:
  readOnlyRootFilesystem: false
  runAsGroup: 0
  runAsUser: 0
  runAsNonRoot: false
```

这样我们可以通过设置 `--certificatesresolvers.ali.acme.dnschallenge.provider=alidns` 参数来指定指定阿里云的 DNS 校验，要使用阿里云的 DNS 校验我们还需要配置3个环境变量：`ALICLOUD_ACCESS_KEY`、`ALICLOUD_SECRET_KEY`、`ALICLOUD_REGION_ID`，分别对应我们平时开发阿里云应用的时候的密钥，可以登录阿里云后台获取，由于这是比较私密的信息，所以我们用 Secret 对象来创建：

```sh
$ kubectl create secret generic traefik-alidns-secret --from-literal=ALICLOUD_ACCESS_KEY=<aliyun ak> --from-literal=ALICLOUD_SECRET_KEY=<aliyun sk> --from-literal=ALICLOUD_REGION_ID=cn-beijing -n kube-system
```

创建完成后将这个 Secret 通过环境变量配置到 Traefik 的应用中，还有一个值得注意的是验证通过的证书我们这里存到 `/data/acme.json` 文件中，我们一定要将这个文件持久化，否则每次 Traefik 重建后就需要重新认证，而 `Let’s Encrypt` 本身校验次数是有限制的。所以我们在 values 中重新开启了数据持久化，不过开启过后需要我们提供一个可用的 PV 存储，由于我们将 Traefik 固定到 master1 节点上的，所以我们可以创建一个 hostpath 类型的 PV（后面会详细讲解）：

```yaml
$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: traefik
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 128Mi
  hostPath:
    path: /data/k8s/traefik
EOF
```

然后使用如下所示的命令更新 Traefik：

```sh
$ helm upgrade --install traefik --namespace=kube-system ./traefik -f ./values-prod.yaml 
Release "traefik" has been upgraded. Happy Helming!
NAME: traefik
LAST DEPLOYED: Thu Dec 24 14:32:04 2020
NAMESPACE: kube-system
STATUS: deployed
REVISION: 2
TEST SUITE: None 
```

更新完成后现在我们来修改上面我们的 `whoami` 应用：

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: ingressroutetls
spec:
  entryPoints:
    - websecure
  routes:
  - match: Host(`who.qikqiak.com`) && PathPrefix(`/tls`)
    kind: Rule
    services:
    - name: whoami
      port: 80
  tls:
    certResolver: ali
    domains:
    - main: "*.qikqiak.com"
```

其他的都不变，只需要将 tls 部分改成我们定义的 `ali` 这个证书解析器，如果我们想要生成一个通配符的域名证书的话可以定义 `domains` 参数来指定，然后更新 IngressRoute 对象，这个时候我们再去用 HTTPS 访问我们的应用（当然需要将域名在阿里云 DNS 上做解析）：

![image-20210112180605758](D:\学习资料\笔记\k8s\k8s图\image-20210112180605758.png)

我们可以看到访问应用已经是受浏览器信任的证书了，查看证书我们还可以发现该证书是一个通配符的证书



### 中间件

中间件是 Traefik2.x 中一个非常有特色的功能，我们可以根据自己的各种需求去选择不同的中间件来满足服务，Traefik 官方已经内置了许多不同功能的中间件，其中一些可以修改请求，头信息，一些负责重定向，一些添加身份验证等等，而且中间件还可以通过链式组合的方式来适用各种情况。

![image-20210112180715194](D:\学习资料\笔记\k8s\k8s图\image-20210112180715194.png)

同样比如上面我们定义的 whoami 这个应用，我们可以通过 `https://who.qikqiak.com/tls` 来访问到应用，但是如果我们用 `http` 来访问的话呢就不行了，就会404了，因为我们根本就没有简单80端口这个入口点，所以要想通过 `http` 来访问应用的话自然我们需要监听下 `web` 这个入口点：

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: ingressroutetls-http
spec:
  entryPoints:
    - web
  routes:
  - match: Host(`who.qikqiak.com`) && PathPrefix(`/tls`)
    kind: Rule
    services:
    - name: whoami
      port: 80
```

注意这里我们创建的 IngressRoute 的 entryPoints 是 `web`，然后创建这个对象，这个时候我们就可以通过 http 访问到这个应用了。

但是我们如果只希望用户通过 https 来访问应用的话呢？按照以前的知识，我们是不是可以让 http 强制跳转到 https 服务去，对的，在 Traefik 中也是可以配置强制跳转的，只是这个功能现在是通过中间件来提供的了。如下所示，我们使用 `redirectScheme` 中间件来创建提供强制跳转服务：

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: redirect-https
spec:
  redirectScheme:
    scheme: https
```

然后将这个中间件附加到 http 的服务上面去，因为 https 的不需要跳转：

```yaml
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: ingressroutetls-http
spec:
  entryPoints:
    - web
  routes:
  - match: Host(`who.qikqiak.com`) && PathPrefix(`/tls`)
    kind: Rule
    services:
    - name: whoami
      port: 80
    middlewares: 
    - name: redirect-https
```

这个时候我们再去访问 http 服务可以发现就会自动跳转到 https 去了。关于更多中间件的用法可以查看文档 Traefik Docs。



### Traefik Pilot

虽然 Traefik 已经默认实现了很多中间件，可以满足大部分我们日常的需求，但是在实际工作中，用户仍然还是有自定义中间件的需求，这就 Traefik Pilot 的功能了。

Traefik Pilot 是一个 SaaS 平台，和 Traefik 进行链接来扩展其功能，它提供了很多功能，通过一个全局控制面板和 Dashboard 来增强对 Traefik 的观测和控制：

- Traefik 代理和代理组的网络活动的指标
- 服务健康问题和安全漏洞警报
- 扩展 Traefik 功能的插件

在 Traefik 可以使用 `Traefik Pilot` 的功能之前，必须先连接它们，我们只需要对 Traefik 的静态配置进行少量更改即可。

> Traefik 代理必须要能访问互联网才能连接到 `Traefik Pilot`，通过 HTTPS 在 443 端口上建立连接。

首先我们需要在 `Traefik Pilot` 主页上(https://pilot.traefik.io/)创建一个帐户，注册新的 `Traefik` 实例并开始使用 `Traefik Pilot`。登录后，可以通过选择 `Register New Traefik Instance`来创建新实例。

![image-20210112181030084](D:\学习资料\笔记\k8s\k8s图\image-20210112181030084.png)

另外，当我们的 Traefik 尚未连接到 `Traefik Pilot` 时，Traefik Web UI 中将出现一个响铃图标，我们可以选择 `Connect with Traefik Pilot` 导航到 Traefik Pilot UI 进行操作。

![image-20210112181048976](D:\学习资料\笔记\k8s\k8s图\image-20210112181048976.png)

登录完成后，`Traefik Pilot` 会生成一个新实例的令牌，我们需要将这个 Token 令牌添加到 Traefik 静态配置中。

![image-20210112181105263](D:\学习资料\笔记\k8s\k8s图\image-20210112181105263.png)

我们这里就是在 `values-prod.yaml` 文件中启用 Pilot 的配置：

```yaml
# values-prod.yaml
# Activate Pilot integration
pilot:
  enabled: true
  token: "e079ea6e-536a-48c6-b3e3-f7cfaf94f477"
```

然后重新更新 Traefik：

```sh
$ helm upgrade --install traefik --namespace=kube-system ./traefik -f ./values-prod.yaml
```

更新完成后，我们在 Traefik 的 Web UI 中就可以看到 Traefik Pilot UI 相关的信息了。

![image-20210112181142514](D:\学习资料\笔记\k8s\k8s图\image-20210112181142514.png)

接下来我们就可以在 Traefik Pilot 的插件页面选择我们想要使用的插件，比如我们这里使用 Demo Plugin 这个插件。

![image-20210112181156943](D:\学习资料\笔记\k8s\k8s图\image-20210112181156943.png)

点击右上角的 `Install Plugin` 按钮安装插件会弹出一个对话框提示我们如何安装。

![image-20210112181212769](D:\学习资料\笔记\k8s\k8s图\image-20210112181212769.png)

首先我们需要将当前 Traefik 注册到 Traefik Pilot（已完成），然后需要以静态配置的方式添加这个插件到 Traefik 中，这里我们同样更新 `values-prod.yaml` 文件中的 Values 值即可：

```yaml
# values-prod.yaml
# Activate Pilot integration
pilot:
  enabled: true
  token: "e079ea6e-536a-48c6-b3e3-f7cfaf94f477"

additionalArguments:
# 添加 demo plugin 的支持
- --experimental.plugins.plugindemo.modulename=github.com/traefik/plugindemo
- --experimental.plugins.plugindemo.version=v0.2.1
# 其他配置
```

同样重新更新 Traefik：

```sh
$ helm upgrade --install traefik --namespace=kube-system ./traefik -f ./values-prod.yaml
```

更新完成后创建一个如下所示的 Middleware 对象：

```yaml
$ cat <<EOF | kubectl apply -f -
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: myplugin
spec:
  plugin:
    plugindemo:  # 插件名
      Headers:
        X-Demo: test
        Foo: bar
EOF
```

然后添加到上面的 whoami 应用的 IngressRoute 对象中去：

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: simpleingressroute
  namespace: default
spec:
  entryPoints:
  - web
  routes:
  - match: Host(`who.qikqiak.com`) && PathPrefix(`/notls`)
    kind: Rule
    services:
    - name: whoami  # K8s Service
      port: 80
    middlewares: 
    - name: myplugin  # 使用上面新建的 middleware
```

更新完成后，当我们去访问 `http://who.qikqiak.com/notls` 的时候就可以看到新增了两个上面插件中定义的两个 Header。

![image-20210112181319935](D:\学习资料\笔记\k8s\k8s图\image-20210112181319935.png)

当然除了使用 Traefik Pilot 上开发者提供的插件之外，我们也可以根据自己的需求自行开发自己的插件，可以自行参考文档：https://doc.traefik.io/traefik-pilot/plugins/plugin-dev/。



### 灰度发布

Traefik2.0 的一个更强大的功能就是灰度发布，灰度发布我们有时候也会称为金丝雀发布（Canary），主要就是让一部分测试的服务也参与到线上去，经过测试观察看是否符号上线要求。

![image-20210112181353021](D:\学习资料\笔记\k8s\k8s图\image-20210112181353021.png)

比如现在我们有两个名为 `appv1` 和 `appv2` 的服务，我们希望通过 Traefik 来控制我们的流量，将 3⁄4 的流量路由到 appv1，1/4 的流量路由到 appv2 去，这个时候就可以利用 Traefik2.0 中提供的**带权重的轮询（WRR）**来实现该功能，首先在 Kubernetes 集群中部署上面的两个服务。为了对比结果我们这里提供的两个服务一个是 whoami，一个是 nginx，方便测试。

appv1 服务的资源清单如下所示：（appv1.yaml）

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: appv1
spec:
  selector:
    matchLabels:
      app: appv1
  template:
    metadata:
      labels:
        use: test
        app: appv1
    spec:
      containers:
      - name: whoami
        image: containous/whoami
        ports:
        - containerPort: 80
          name: portv1
---
apiVersion: v1
kind: Service
metadata:
  name: appv1
spec:
  selector:
    app: appv1
  ports:
  - name: http
    port: 80
    targetPort: portv1
```

appv2 服务的资源清单如下所示：（appv2.yaml）

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: appv2
spec:
  selector:
    matchLabels:
      app: appv2
  template:
    metadata:
      labels:
        use: test
        app: appv2
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
          name: portv2
---
apiVersion: v1
kind: Service
metadata:
  name: appv2
spec:
  selector:
    app: appv2
  ports:
  - name: http
    port: 80
    targetPort: portv2
```

直接创建上面两个服务：

```sh
$ kubectl apply -f appv1.yaml
$ kubectl apply -f appv2.yaml
# 通过下面的命令可以查看服务是否运行成功
$ kubectl get pods -l use=test
NAME                     READY   STATUS    RESTARTS   AGE
appv1-58f856c665-shm9j   1/1     Running   0          12s
appv2-ff5db55cf-qjtrf    1/1     Running   0          12s
```

在 Traefik2.1 中新增了一个 `TraefikService` 的 CRD 资源，我们可以直接利用这个对象来配置 WRR，之前的版本需要通过 File Provider，比较麻烦，新建一个描述 WRR 的资源清单：(wrr.yaml)

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: TraefikService
metadata:
  name: app-wrr
spec:
  weighted:
    services:
      - name: appv1
        weight: 3  # 定义权重
        port: 80
        kind: Service  # 可选，默认就是 Service
      - name: appv2
        weight: 1
        port: 80
```

然后为我们的灰度发布的服务创建一个 IngressRoute 资源对象：(ingressroute.yaml)

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: wrringressroute
  namespace: default
spec:
  entryPoints:
    - web
  routes:
  - match: Host(`wrr.qikqiak.com`)
    kind: Rule
    services:
    - name: app-wrr
      kind: TraefikService
```

不过需要注意的是现在我们配置的 Service 不再是直接的 Kubernetes 对象了，而是上面我们定义的 TraefikService 对象，直接创建上面的两个资源对象，这个时候我们对域名 `wrr.qikqiak.com` 做上解析，去浏览器中连续访问 4 次，我们可以观察到 appv1 这应用会收到 3 次请求，而 appv2 这个应用只收到 1 次请求，符合上面我们的 `3:1` 的权重配置。

![image-20210112181705130](D:\学习资料\笔记\k8s\k8s图\image-20210112181705130.png)



### 流量复制

除了灰度发布之外，Traefik 2.0 还引入了流量镜像服务，是一种可以将流入流量复制并同时将其发送给其他服务的方法，镜像服务可以获得给定百分比的请求同时也会忽略这部分请求的响应。

![image-20210112181735031](D:\学习资料\笔记\k8s\k8s图\image-20210112181735031.png)



现在我们部署两个 whoami 的服务，资源清单文件如下所示：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: v1
spec:
  ports:
    - protocol: TCP
      name: web
      port: 80
  selector:
    app: v1
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: v1
  labels:
    app: v1
spec:
  selector:
    matchLabels:
      app: v1
  template:
    metadata:
      labels:
        app: v1
    spec:
      containers:
        - name: v1
          image: nginx
          ports:
            - name: web
              containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: v2
spec:
  ports:
    - protocol: TCP
      name: web
      port: 80
  selector:
    app: v2
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: v2
  labels:
    app: v2
spec:
  selector:
    matchLabels:
      app: v2
  template:
    metadata:
      labels:
        app: v2
    spec:
      containers:
        - name: v2
          image: nginx
          ports:
            - name: web
              containerPort: 80
```

直接创建上面的资源对象：

```sh
$ kubectl get pods
NAME                                      READY   STATUS    RESTARTS   AGE
v1-77cfb86999-wfbl2                       1/1     Running   0          94s
v2-6f45d498b7-g6qjt                       1/1     Running   0          91s
$ kubectl get svc 
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
v1              ClusterIP   10.96.218.173   <none>        80/TCP      99s
v2              ClusterIP   10.99.98.48     <none>        80/TCP      96s
```

现在我们创建一个 IngressRoute 对象，将服务 v1 的流量复制 50% 到服务 v2，如下资源对象所示：(mirror-ingress-route.yaml)

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: TraefikService
metadata:
  name: app-mirror
spec:
  mirroring:
    name: v1 # 发送 100% 的请求到 K8S 的 Service "v1"
    port: 80
    mirrors:
    - name: v2 # 然后复制 50% 的请求到 v2
      percent: 50
      port: 80
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: mirror-ingress-route
  namespace: default
spec:
  entryPoints:
  - web
  routes:   
  - match: Host(`mirror.qikqiak.com`)
    kind: Rule
    services:
    - name: app-mirror
      kind: TraefikService # 使用声明的 TraefikService 服务，而不是 K8S 的 Service
```

然后直接创建这个资源对象即可：

```sh
$ kubectl apply -f mirror-ingress-route.yaml 
ingressroute.traefik.containo.us/mirror-ingress-route created
traefikservice.traefik.containo.us/mirroring-example created
```

这个时候我们在浏览器中去连续访问4次 `mirror.qikqiak.com` 可以发现有一半的请求也出现在了 `v2` 这个服务中：![图片](https://mmbiz.qpic.cn/mmbiz_png/z9BgVMEm7Yv1FB6IS2KtN2dWt33vs0aBkxR2lK2pHtKuxsxcTlOXkPGvsqDfb8E1M9bgicHqD3c40fCa2ibt45Ew/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



### TCP

另外 Traefik2.x 已经支持了 TCP 服务的，下面我们以 mongo 为例来了解下 Traefik 是如何支持 TCP 服务得。

#### 简单 TCP 服务

首先部署一个普通的 mongo 服务，资源清单文件如下所示：（mongo.yaml）

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo-traefik
  labels:
    app: mongo-traefik
spec:
  selector:
    matchLabels:
      app: mongo-traefik
  template:
    metadata:
      labels:
        app: mongo-traefik
    spec:
      containers:
      - name: mongo
        image: mongo:4.0
        ports:
        - containerPort: 27017
---
apiVersion: v1
kind: Service
metadata:
  name: mongo-traefik
spec:
  selector:
    app: mongo-traefik
  ports:
  - port: 27017
```

直接创建 mongo 应用：

```sh
$ kubectl apply -f mongo.yaml
deployment.apps/mongo-traefik created
service/mongo-traefik created
```

创建成功后就可以来为 mongo 服务配置一个路由了。由于 Traefik 中使用 TCP 路由配置需要 `SNI`，而 `SNI` 又是依赖 `TLS` 的，所以我们需要配置证书才行，如果没有证书的话，我们可以使用通配符 `*` 进行配置，我们这里创建一个 `IngressRouteTCP` 类型的 CRD 对象（前面我们就已经安装了对应的 CRD 资源）：(mongo-ingressroute-tcp.yaml)

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRouteTCP
metadata:
  name: mongo-traefik-tcp
spec:
  entryPoints:
    - mongo
  routes:
  - match: HostSNI(`*`)
    services:
    - name: mongo-traefik
      port: 27107
```

要注意的是这里的 `entryPoints` 部分，是根据我们启动的 Traefik 的静态配置中的 entryPoints 来决定的，我们当然可以使用前面我们定义得 80 和 443 这两个入口点，但是也可以可以自己添加一个用于 mongo 服务的专门入口点，更新 `values-prod.yaml` 文件，新增 mongo 这个入口点：

```yaml
# values-prod.yaml
# Configure ports
ports:
  web:
    port: 8000
    hostPort: 80
  websecure:
    port: 8443
    hostPort: 443
  mongo:
    port: 27017
    hostPort: 27017
```

然后更新 Traefik 即可：

```sh
$ helm upgrade --install traefik --namespace=kube-system ./traefik -f ./values-prod.yaml 
```

这里给入口点添加 `hostPort` 是为了能够通过节点的端口访问到服务，关于 entryPoints 入口点的更多信息，可以查看文档 entrypoints 了解更多信息。

然后更新 Traefik 后我们就可以直接创建上面的资源对象：

```sh
$ mongo-ingressroute-tcp.yaml
ingressroutetcp.traefik.containo.us/mongo-traefik-tcp created
```

创建完成后，同样我们可以去 Traefik 的 Dashboard 页面上查看是否生效：

![image-20210112182331752](D:\学习资料\笔记\k8s\k8s图\image-20210112182331752.png)



然后我们配置一个域名 `mongo.local` 解析到 Traefik 所在的节点，然后通过 27017 端口来连接 mongo 服务：

```sh
$ mongo --host mongo.local --port 27017
mongo(75243,0x1075295c0) malloc: *** malloc_zone_unregister() failed for 0x7fffa56f4000
MongoDB shell version: 2.6.1
connecting to: mongo.local:27017/test
> show dbs
admin   0.000GB
config  0.000GB
local   0.000GB
```

到这里我们就完成了将 mongo（TCP）服务暴露给外部用户了。



### 带 TLS 证书的 TCP

上面我们部署的 mongo 是一个普通的服务，然后用 Traefik 代理的，但是有时候为了安全 mongo 服务本身还会使用 TLS 证书的形式提供服务，下面是用来生成 mongo tls 证书的脚本文件：（generate-certificates.sh）

```sh
#!/bin/bash
#
# From https://medium.com/@rajanmaharjan/secure-your-mongodb-connections-ssl-tls-92e2addb3c89

set -eu -o pipefail

DOMAINS="${1}"
CERTS_DIR="${2}"
[ -d "${CERTS_DIR}" ]
CURRENT_DIR="$(cd "$(dirname "${0}")" && pwd -P)"

GENERATION_DIRNAME="$(echo "${DOMAINS}" | cut -d, -f1)"

rm -rf "${CERTS_DIR}/${GENERATION_DIRNAME:?}" "${CERTS_DIR}/certs"

echo "== Checking Requirements..."
command -v go >/dev/null 2>&1 || echo "Golang is required"
command -v minica >/dev/null 2>&1 || go get github.com/jsha/minica >/dev/null

echo "== Generating Certificates for the following domains: ${DOMAINS}..."
cd "${CERTS_DIR}"
minica --ca-cert "${CURRENT_DIR}/minica.pem" --ca-key="${CURRENT_DIR}/minica-key.pem" --domains="${DOMAINS}"
mv "${GENERATION_DIRNAME}" "certs"
cat certs/key.pem certs/cert.pem > certs/mongo.pem

echo "== Certificates Generated in the directory ${CERTS_DIR}/certs"
```

将上面证书放置到 certs 目录下面，然后我们新建一个 `02-tls-mongo` 的目录，在该目录下面执行如下命令来生成证书：

```sh
$ bash ../certs/generate-certificates.sh mongo.local .
== Checking Requirements...
== Generating Certificates for the following domains: mongo.local...
```

最后的目录如下所示，在 `02-tls-mongo` 目录下面会生成包含证书的 certs 目录：

```sh
➜ tree .
.
├── 01-mongo
│   ├── mongo-ingressroute-tcp.yaml
│   └── mongo.yaml
├── 02-tls-mongo
│   └── certs
│       ├── cert.pem
│       ├── key.pem
│       └── mongo.pem
└── certs
    ├── generate-certificates.sh
    ├── minica-key.pem
    └── minica.pem
```

在 `02-tls-mongo/certs` 目录下面执行如下命令通过 Secret 来包含证书内容：

```sh
$ kubectl create secret tls traefik-mongo-certs --cert=cert.pem --key=key.pem
secret/traefik-mongo-certs created
```

然后重新更新 `IngressRouteTCP` 对象，增加 TLS 配置：

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRouteTCP
metadata:
  name: mongo-traefik-tcp
spec:
  entryPoints:
    - mongo
  routes:
  - match: HostSNI(`mongo.local`)
    services:
    - name: mongo-traefik
      port: 27017
  tls: 
    secretName: traefik-mongo-certs
```

同样更新后，现在我们直接去访问应用就会被 hang 住，因为我们没有提供证书：

```sh
$ mongo --host mongo.local --port 27017
MongoDB shell version: 2.6.1
connecting to: mongo1.local:27017/test
```

这个时候我们可以带上证书来进行连接：

```sh
$ mongo --host mongo.local --port 27017 --ssl --sslCAFile=../certs/minica.pem --sslPEMKeyFile=./certs/mongo.pem
MongoDB shell version v4.0.3
connecting to: mongodb://mongo.local:27017/
Implicit session: session { "id" : UUID("e7409ef6-8ebe-4c5a-9642-42059bdb477b") }
MongoDB server version: 4.0.14
......
> show dbs;
admin   0.000GB
config  0.000GB
local   0.000GB
```

可以看到现在就可以连接成功了，这样就完成了一个使用 TLS 证书代理 TCP 服务的功能，这个时候如果我们使用其他的域名去进行连接就会报错了，因为现在我们指定的是特定的 HostSNI：

```sh
$ mongo --host mongo.k8s.local --port 27017 --ssl --sslCAFile=../certs/minica.pem --sslPEMKeyFile=./certs/mongo.pem
MongoDB shell version v4.0.3
connecting to: mongodb://mongo.k8s.local:27017/
2019-12-29T15:03:52.424+0800 E NETWORK  [js] SSL peer certificate validation failed: Certificate trust failure: CSSMERR_TP_NOT_TRUSTED; connection rejected
2019-12-29T15:03:52.429+0800 E QUERY    [js] Error: couldn't connect to server mongo.qikqiak.com:27017, connection attempt failed: SSLHandshakeFailed: SSL peer certificate validation failed: Certificate trust failure: CSSMERR_TP_NOT_TRUSTED; connection rejected :
connect@src/mongo/shell/mongo.js:257:13
@(connect):1:6
exception: connect failed
```

Traefik 还有很多功能，比如 UDP 也支持了，特别是强大的中间件和自定义插件的功能，为我们提供了不断扩展其功能的能力，我们完成可以根据自己的需求进行二次开发。





## Kubernetes 部署 Ingress 控制器 Traefik v2.3



**系统环境：**

- Traefik 版本：v2.3.6
- Kubernetes 版本：1.20.1

**参考地址：**

- [Traefik 2.3 官方文档](https://docs.traefik.io/v2.3/)
- [Traefik RCD 路由规则参考地址](https://docs.traefik.io/v2.3/routing/providers/kubernetes-crd/)
- [Traefik Ingress 路由规则参考地址](https://docs.traefik.io/v2.3/routing/providers/kubernetes-ingress/)

**示例部署文件：**

- 示例部署文件 Github 地址：https://github.com/my-dlq/blog-example/tree/master/kubernetes/traefik-v2.3-deploy



### 一、Traefik 简介

​    Traefik 是一款开源的边缘路由器，它可以让发布服务变得轻松有趣。它代表您的系统接收请求，并找出负责处理这些请求的组件。与众不同之处在于，除了它的许多特性之外，它还可以自动为您的服务发现正确的配置。当 Traefik 检查您的基础设施时，它会发现相关信息，并发现哪个服务为哪个请求提供服务。

​    Traefik 与每个主要的集群技术都是原生兼容的，比如 Kubernetes、Docker、Docker Swarm、AWS、Mesos、Marathon 等等;并且可以同时处理多个。(它甚至适用于运行在裸机上的遗留软件。) 使用 Traefik，不需要维护和同步单独的配置文件:所有事情都是实时自动发生的(没有重启，没有连接中断)。使用 Traefik，只需要花费时间开发和部署新功能到您的系统，而不是配置和维护其工作状态。



### 二、Kubernetes 部署 Traefik

​    Traefik 最新推出了 v2.3 版本，这里将 Traefik 升级到最新版本，简单的介绍了下如何在 Kubernetes 环境下安装 Traefik v2.3，下面将介绍如何在 Kubernetes 环境下部署并配置 Traefik v2.3。

​    当部署完 Traefik 后还需要创建外部访问 Kubernetes 内部应用的路由规则，Traefik 支持两种方式创建路由规则，一种是创建 Traefik 自定义 `Kubernetes CRD` 资源方式，还有一种是创建 `Kubernetes Ingress` 资源方式。

> 注意：这里 Traefik 是部署在 Kube-system Namespace 下，如果不想部署到配置的 Namespace，需要修改下面部署文件中的 Namespace 参数。



#### 1、创建 CRD 资源

在 `Traefik v2.0` 版本后，开始使用 `CRD`（Custom Resource Definition）来完成路由配置等，所以需要提前创建 `CRD` 资源。

**创建 traefik-crd.yaml 文件**

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ingressroutes.traefik.containo.us
spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: IngressRoute
    plural: ingressroutes
    singular: ingressroute
  scope: Namespaced
---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: middlewares.traefik.containo.us
spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: Middleware
    plural: middlewares
    singular: middleware
  scope: Namespaced
---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ingressroutetcps.traefik.containo.us
spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: IngressRouteTCP
    plural: ingressroutetcps
    singular: ingressroutetcp
  scope: Namespaced
---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ingressrouteudps.traefik.containo.us
spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: IngressRouteUDP
    plural: ingressrouteudps
    singular: ingressrouteudp
  scope: Namespaced
---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: tlsoptions.traefik.containo.us
spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: TLSOption
    plural: tlsoptions
    singular: tlsoption
  scope: Namespaced
---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: tlsstores.traefik.containo.us
spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: TLSStore
    plural: tlsstores
    singular: tlsstore
  scope: Namespaced
---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: traefikservices.traefik.containo.us
spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: TraefikService
    plural: traefikservices
    singular: traefikservice
  scope: Namespaced
```

**创建 Traefik CRD 资源**

```bash
$ kubectl apply -f traefik-crd.yaml
```



#### 2、创建 RBAC 权限

Kubernetes 在 1.6 版本中引入了基于角色的访问控制（RBAC）策略，方便对 `Kubernetes` 资源和 `API` 进行细粒度控制。`Traefik` 需要一定的权限，所以，这里提前创建好 `Traefik ServiceAccount` 并分配一定的权限。

**# 创建 traefik-rbac.yaml 文件:**

```yaml
## ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: kube-system
  name: traefik-ingress-controller
---
## ClusterRole
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
rules:
  - apiGroups: [""]
    resources: ["services","endpoints","secrets"]
    verbs: ["get","list","watch"]
  - apiGroups: ["extensions","networking.k8s.io"]
    resources: ["ingresses","ingressclasses"]
    verbs: ["get","list","watch"]
  - apiGroups: ["extensions"]
    resources: ["ingresses/status"]
    verbs: ["update"]
  - apiGroups: ["traefik.containo.us"]
    resources: ["middlewares","ingressroutes","ingressroutetcps","tlsoptions","ingressrouteudps","traefikservices","tlsstores"]
    verbs: ["get","list","watch"]
---
## ClusterRoleBinding
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
  - kind: ServiceAccount
    name: traefik-ingress-controller
    namespace: kube-system
```

**# 创建 Traefik RBAC 资源**

- -n：指定部署的 Namespace

```bash
$ kubectl apply -f traefik-rbac.yaml -n kube-system
```



#### 3、创建 Traefik 配置文件

由于 Traefik 配置很多，通过 `CLI` 定义不是很方便，一般时候都会通过配置文件配置 `Traefik` 参数，然后存入 `ConfigMap`，将其挂入 `Traefik` 中。

**# 创建 traefik-config.yaml 文件**

> 下面配置中可以通过配置 kubernetesCRD 与 kubernetesIngress 两项参数，让 Traefik 支持 CRD 与 Ingress 两种路由方式。

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: traefik-config
data:
  traefik.yaml: |-
    ping: ""                    ## 启用 Ping
    serversTransport:
      insecureSkipVerify: true  ## Traefik 忽略验证代理服务的 TLS 证书
    api:
      insecure: true            ## 允许 HTTP 方式访问 API
      dashboard: true           ## 启用 Dashboard
      debug: false              ## 启用 Debug 调试模式
    metrics:
      prometheus: ""            ## 配置 Prometheus 监控指标数据，并使用默认配置
    entryPoints:
      web:
        address: ":80"          ## 配置 80 端口，并设置入口名称为 web
      websecure:
        address: ":443"         ## 配置 443 端口，并设置入口名称为 websecure
    providers:
      kubernetesCRD: ""         ## 启用 Kubernetes CRD 方式来配置路由规则
      kubernetesIngress: ""     ## 启动 Kubernetes Ingress 方式来配置路由规则
    log:
      filePath: ""              ## 设置调试日志文件存储路径，如果为空则输出到控制台
      level: error              ## 设置调试日志级别
      format: json              ## 设置调试日志格式
    accessLog:
      filePath: ""              ## 设置访问日志文件存储路径，如果为空则输出到控制台
      format: json              ## 设置访问调试日志格式
      bufferingSize: 0          ## 设置访问日志缓存行数
      filters:
        #statusCodes: ["200"]   ## 设置只保留指定状态码范围内的访问日志
        retryAttempts: true     ## 设置代理访问重试失败时，保留访问日志
        minDuration: 20         ## 设置保留请求时间超过指定持续时间的访问日志
      fields:                   ## 设置访问日志中的字段是否保留（keep 保留、drop 不保留）
        defaultMode: keep       ## 设置默认保留访问日志字段
        names:                  ## 针对访问日志特别字段特别配置保留模式
          ClientUsername: drop  
        headers:                ## 设置 Header 中字段是否保留
          defaultMode: keep     ## 设置默认保留 Header 中字段
          names:                ## 针对 Header 中特别字段特别配置保留模式
            User-Agent: redact
            Authorization: drop
            Content-Type: keep
    #tracing:                     ## 链路追踪配置,支持 zipkin、datadog、jaeger、instana、haystack 等 
    #  serviceName:               ## 设置服务名称（在链路追踪端收集后显示的服务名）
    #  zipkin:                    ## zipkin配置
    #    sameSpan: true           ## 是否启用 Zipkin SameSpan RPC 类型追踪方式
    #    id128Bit: true           ## 是否启用 Zipkin 128bit 的跟踪 ID
    #    sampleRate: 0.1          ## 设置链路日志采样率（可以配置0.0到1.0之间的值）
    #    httpEndpoint: http://localhost:9411/api/v2/spans     ## 配置 Zipkin Server 端点
```

**# 创建 Traefik ConfigMap 资源**

- -n： 指定程序启动的 Namespace

```bash
$ kubectl apply -f traefik-config.yaml -n kube-system
```



#### 4、节点设置 Label 标签

由于是 `Kubernetes DeamonSet` 这种方式部署 `Traefik`，所以需要提前给节点设置 `Label`，这样当程序部署时会自动调度到设置 `Label` 的节点上。

**# 节点设置 Label 标签**

- 格式：kubectl label nodes [节点名] [key=value]

```bash
$ kubectl label nodes k8s-node-2-12 IngressProxy=true
```

**# 查看节点是否设置 Label 成功**

```bash
$ kubectl get nodes --show-labels

NAME            STATUS ROLES  VERSION  LABELS
k8s-master-2-11 Ready  master  v1.20.1  kubernetes.io/hostname=k8s-master-2-11,node-role.kubernetes.io/master=
k8s-node-2-12   Ready  <none> v1.20.1  kubernetes.io/hostname=k8s-node-2-12,IngressProxy=true
k8s-node-2-13   Ready  <none> v1.20.1  kubernetes.io/hostname=k8s-node-2-13
k8s-node-2-14   Ready  <none> v1.20.1  kubernetes.io/hostname=k8s-node-2-14
```

> 如果想删除标签，可以使用 “`kubectl label nodes k8s-node-2-12 IngressProxy-`” 命令



#### 5、Kubernetes 部署 Traefik

下面将用 `DaemonSet` 方式部署 `Traefik`，便于在多服务器间扩展，用 `hostport` 方式绑定服务器 `80`、`443` 端口，方便流量通过物理机进入 `Kubernetes` 内部。

**# 创建 traefik 部署 traefik-deploy.yaml 文件**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: traefik
spec:
  ports:
    - name: web
      port: 80
    - name: websecure
      port: 443
    - name: admin
      port: 8080
  selector:
    app: traefik
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: traefik-ingress-controller
  labels:
    app: traefik
spec:
  selector:
    matchLabels:
      app: traefik
  template:
    metadata:
      name: traefik
      labels:
        app: traefik
    spec:
      serviceAccountName: traefik-ingress-controller
      terminationGracePeriodSeconds: 1
      containers:
        - image: traefik:v2.3.6
          name: traefik-ingress-lb
          ports:
            - name: web
              containerPort: 80
              hostPort: 80         ## 将容器端口绑定所在服务器的 80 端口
            - name: websecure
              containerPort: 443
              hostPort: 443        ## 将容器端口绑定所在服务器的 443 端口
            - name: admin
              containerPort: 8080  ## Traefik Dashboard 端口
          resources:
            limits:
              cpu: 2000m
              memory: 1024Mi
            requests:
              cpu: 1000m
              memory: 1024Mi
          securityContext:
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
          args:
            - --configfile=/config/traefik.yaml
          volumeMounts:
            - mountPath: "/config"
              name: "config"
          readinessProbe:
            httpGet:
              path: /ping
              port: 8080
            failureThreshold: 3
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 5
          livenessProbe:
            httpGet:
              path: /ping
              port: 8080
            failureThreshold: 3
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 5    
      volumes:
        - name: config
          configMap:
            name: traefik-config 
      tolerations:              ## 设置容忍所有污点，防止节点被设置污点
        - operator: "Exists"
      nodeSelector:             ## 设置node筛选器，在特定label的节点上启动
        IngressProxy: "true"
```

**# Kubernetes 部署 Traefik**

```bash
$ kubectl apply -f traefik-deploy.yaml -n kube-system
```

到此 Traefik v2.3 应用已经部署完成。



### 三、配置路由规则

​    Traefik 应用已经部署完成，但是想让外部访问 `Kubernetes` 内部服务，还需要配置路由规则，上面部署 `Traefik` 时开启了 `Traefik Dashboard`，这是 `Traefik` 提供的视图看板，所以，首先配置基于 `HTTP` 的 `Traefik Dashboard` 路由规则，使外部能够访问 `Traefik Dashboard`。然后，再配置基于 `HTTPS` 的 `Kubernetes Dashboard` 的路由规则，这里分别使用 `CRD` 和 `Ingress` 两种方式进行演示。



#### 1、方式一：使用 CRD 方式配置 Traefik 路由规则

> 使用 CRD 方式创建路由规则可言参考 Traefik 文档 [Kubernetes IngressRoute](https://docs.traefik.io/v2.3/routing/providers/kubernetes-crd/)

#### (1)、配置 HTTP 路由规则 （Traefik Dashboard 为例）

**# 创建 Traefik Dashboard 路由规则 traefik-dashboard-route.yaml 文件**

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-dashboard-route
spec:
  entryPoints:
  - web
  routes:
  - match: Host(`traefik.mydlq.club`)
    kind: Rule
    services:
      - name: traefik
        port: 8080
```

**# 创建 Traefik Dashboard 路由规则对象**

```bash
$ kubectl apply -f traefik-dashboard-route.yaml -n kube-system
```



#### (2)、配置 HTTPS 路由规则（Kubernetes Dashboard 为例）

这里我们创建 `Kubernetes` 的 `Dashboard` 看板创建路由规则，它是 `Https` 协议方式，由于它是需要使用 Https 请求，所以我们配置基于 Https 的路由规则并指定证书。

**# 创建私有证书 tls.key、tls.crt 文件**

```bash
# 创建自签名证书
$ openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=cloud.mydlq.club"

# 将证书存储到 Kubernetes Secret 中
$ kubectl create secret generic cloud-mydlq-tls --from-file=tls.crt --from-file=tls.key -n kube-system
```

**# 创建 Traefik Dashboard CRD 路由规则 kubernetes-dashboard-route.yaml 文件**

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata
  name: kubernetes-dashboard-route
spec:
  entryPoints:
  - websecure
  tls:
    secretName: cloud-mydlq-tls
  routes:
  - match: Host(`cloud.mydlq.club`) 
    kind: Rule
    services:
      - name: kubernetes-dashboard
        port: 443
```

**# 创建 Kubernetes Dashboard 路由规则对象**

```bash
$ kubectl apply -f kubernetes-dashboard-route.yaml -n kube-system
```



#### 2、方式二：使用 Ingress 方式配置 Traefik 路由规则

> 使用 Ingress 方式创建路由规则可言参考 Traefik 文档 [Kubernetes Ingress](https://docs.traefik.io/v2.3/routing/providers/kubernetes-ingress/)

##### (1)、配置 HTTP 路由规则 （Traefik Dashboard 为例）

**# 创建 Traefik Ingress 路由规则 traefik-dashboard-ingress.yaml 文件**

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: traefik-dashboard-ingress
  namespace: kube-system
  annotations:
    kubernetes.io/ingress.class: traefik  
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  rules:
  - host: traefik.mydlq.club 
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          serviceName: traefik
          servicePort: 8080
```

**# 创建 Traefik Dashboard Ingress 路由规则对象**

```bash
$ kubectl apply -f traefik-dashboard-ingress.yaml -n kube-system
```



##### (2)、配置 HTTPS 路由规则（Kubernetes Dashboard 为例）

跟上面以 `CRD` 方式创建路由规则一样，也需要创建使用证书，然后再以 `Ingress` 方式创建路由规则。

**# 创建私有证书 tls.key、tls.crt 文件**

```bash
# 创建自签名证书
$ openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=cloud.mydlq.club"

# 将证书存储到 Kubernetes Secret 中
$ kubectl create secret generic cloud-mydlq-tls --from-file=tls.crt --from-file=tls.key -n kube-system
```

**# 创建 Traefik Dashboard Ingress 路由规则 kubernetes-dashboard-ingress.yaml 文件**

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: kubernetes-dashboard-ingress
  namespace: kube-system
  annotations:
    kubernetes.io/ingress.class: traefik                  
    traefik.ingress.kubernetes.io/router.tls: "true"
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
spec:
  tls:
  - hosts:
      - cloud.mydlq.club
    secretName: cloud-mydlq-tls
  rules:
  - host: cloud.mydlq.club 
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
            serviceName: kubernetes-dashboard
            servicePort: 443
```

**# 创建 Traefik Dashboard 路由规则对象**

```bash
$ kubectl apply -f kubernetes-dashboard-ingress.yaml -n kube-system
```



### 四、方式创建路由规则后的应用

#### 1、配置 Host 文件

客户端想通过域名访问服务，必须要进行 `DNS` 解析，由于这里没有 `DNS` 服务器进行域名解析，所以修改 `hosts` 文件将 `Traefik` 所在节点服务器的 `IP` 和自定义 `Host` 绑定。打开电脑的 `Hosts` 配置文件，往其加入下面配置：

```
10.210.3.235  traefik.mydlq.club
10.210.3.235  cloud.mydlq.club
```



#### 2、访问对应应用

**访问 Traefik Dashboard**

打开浏览器输入地址：http://traefik.mydlq.club 打开 Traefik Dashboard。

![image-20210113161301710](D:\学习资料\笔记\k8s\k8s图\image-20210113161301710.png)



**访问 Traefik Dashboard**

打开浏览器输入地址：https://cloud.mydlq.club 打开 Dashboard Dashboard。

![image-20210113161409211](D:\学习资料\笔记\k8s\k8s图\image-20210113161409211.png)





## Traefik 路由规则及中间件 Traefik Middlewares 的配置



**系统环境：**

- Traefik 版本：v2.2.0
- Kubernetes 版本：1.18.2
- 操作系统版本：CentOS 7.8

**参考地址：**

- [Traefik 官方手册](https://docs.traefik.io/)



### 一、什么是 Traefik

​    Traefik 是一款开源的边缘路由器，现在本人主要要作用于 kubernetes 中对外的网关，即 Ingress 路由器，可以很轻松的配置其路由规则，让 Kubernetes 外部流量涌入。并且，还支持如 Zipkin、Jaeger 等链路追踪功能，简单好用。



### 二、什么是 Traefik 路由规则

![image-20210113164226515](D:\学习资料\笔记\k8s\k8s图\image-20210113164226515.png)

​    首先，当部署完后启动 Traefik 时，定义了入口点（端口号和对应的端口名称），然后 Kubernetes 集群外部就可以通过访问 Traefik 服务器地址和配置的入口点对 Traefik 服务进行访问，在访问时一般会带上 “域名” + “入口点端口”，然后 Traefik 会根据域名和入口点端口在 Traefik 路由规则表中进行匹配，如果匹配成功，则将流量发送到 Kubernetes 内部应用中与外界进行交互。这里面的域名与入口点与对应后台服务关联的规则，即是 Traefik 路由规则。

> 如何部署 Traefik 请参考地址：http://www.mydlq.club/article/72/



### 三、什么是 Traefik Middlewares 中间件

![image-20210113164327400](D:\学习资料\笔记\k8s\k8s图\image-20210113164327400.png)

​    Traefik Middlewares 是一个处于路由和后端服务之前的中间件，在外部流量进入 Traefik，且路由规则匹配成功后，将流量发送到对应的后端服务前，先将其发给中间件进行一些列处理（类似于过滤器链 Filter，进行一系列处理），例如，添加 Header 头信息、鉴权、流量转发、处理访问路径前缀、IP 白名单等等，经过一个或者多个中间件处理完成后，再发送给后端服务，这个就是中间件的作用。

> 了解什么是 Traefik Middlewares 之前，最好先了解上面提到的 Traefik Ingress 路由规则。



### 四、如何配置 Traefik 路由规则

在 Kubernetes 中 Traefik 支持 CRD 和 Ingress 两种方式进行路由规则配置，这里配置 Http 和 Https 两种方式为例进行演示。

- (1)、配置 Traefik Dashboard 的 HTTP 路由规则为例，使用域名 **traefik**.mydlq.club 与其后端服务关联。
- (2)、配置 Kubernetes Dashboard 的 HTTPS 路由规则为例，使用域名 **cloud**.mydlq.club 与其后端服务关联。

> 注意：本人部署 Traefik 时候指定了 80 端口的进入点名称是 “web”，而 443 端口的进入点名称为 “websecure”，部署 traefik 时可能这个进入点名称不同，下面的路由规则中设置进入点也不一样，这点需要注意下。



#### 1、方式一：使用 Traefik CRD 配置路由规则

**基于 HTTP 路由路由规则：**

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-dashboard-route
spec:
  entryPoints:
  - web
  routes:
  - match: Host(`traefik.mydlq.club`)
    kind: Rule
    services:
      - name: traefik
        port: 8080
```

**基于 HTTPS 路由路由规则**

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: kubernetes-dashboard-route
spec:
  entryPoints:
  - websecure
  tls:
    secretName: cloud-mydlq-tls
  routes:
  - match: Host(`cloud.mydlq.club`) 
    kind: Rule
    services:
      - name: kubernetes-dashboard
        port: 443
```



#### 2、方式二：使用 Kubernetes Ingress 配置路由规则

> 注意：使用该方式需要 Traefik 版本 ≥ 2.2

**基于 HTTP 路由路由规则**

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: traefik-dashboard-ingress
  namespace: kube-system
  annotations:
    kubernetes.io/ingress.class: traefik            
    traefik.ingress.kubernetes.io/router.entrypoints: web         ##指定 Traefik 中进入点
spec:
  rules:
  - host: traefik.mydlq.club                                 
    http:
      paths:
      - path: /              
        backend:
          serviceName: traefik
          servicePort: 8080
```

**基于 HTTPS 路由路由规则**

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kubernetes-dashboard-ingress
  namespace: kube-system
  annotations:
    kubernetes.io/ingress.class: traefik                  
    traefik.ingress.kubernetes.io/router.tls: "true"              ##指定使用 tls 证书方式
    traefik.ingress.kubernetes.io/router.entrypoints: websecure   ##指定 Traefik 中进入点
spec:
  tls:
  - secretName: cloud-mydlq-tls
  rules:
  - host: cloud.mydlq.club                               
    http:
      paths:
      - path: /                                     
        backend:
          serviceName: kubernetes-dashboard
          servicePort: 443
```



### 五、如何配置 Traefik Middlewares

Traefik Middlewares 中间件是用于流量进入 Traefik 且通过定义的路由规则后，转发到对应后端服务前，在这期间对该流量进行加工的操作，它支持：

- 重试、压缩、缓冲、断路器
- header 管理、错误页、中间件链
- 服务限流、同一主机并发请求限制
- 基本认证、IP 白名单、摘要认证、转发鉴权验证
- regex 请求重定向、scheme 请求重定向、请求 URL 替换、regex 请求 URL 替换、删除 URL 前缀、regex 删除 URL 前缀、添加 URL 前缀

**这里使用常用 Middleware 来进行演示配置中间件，示例如下：**



#### 1、去除请求路径前缀中间件

例如，有一个路由规则中配置的域名路径为 “{host}/one”，traefik 进行路由规则进行流量转发时，也会带上这个前缀作为相对路径发送到后端服务中，而对后端服务来说，一般都是以 “/” 根路径作为相对路径的，如果带上这个路径到后端服务中，那么后端服务则变成以 “/one” 作为相对路径，显然，这样会导致状态码 404 错误。所以，很多时候我们需要去掉这个前缀。

使用去除前缀中间件去除前缀，方便配置一个域名多个二级路径这种需求，中间件写法如下：

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: test-stripprefix
spec:
  stripPrefix:
    prefixes:
      - /one
```

- prefixes：设置去除的 url 前缀。



#### 2、正则表达式匹配去除请求路径前缀中间件

上面说了为什么去除前缀，也写了个去除前缀中间件，不过很多时候我们有这样一个需求，就是通过一个域名访问多个服务，这样易于统一服务的入口，而 traefik 中也提供了这种批量匹配去除前缀的功能，就是通过正则表达式方式批量匹配对应路径的前缀，然后去除，即 stripPrefixRegex 中间件。

使用正则表达式，只要正则表达式匹配成功就能将前缀去除，这样的好处就是可以多个路由规则使用该中间件进行去除前缀，方便配置通用的一个域名多个的二级路径这种需求，中间件写法如下：

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: test-stripprefixregex
spec:
  stripPrefixRegex:
    regex:
      - "/foo/[a-z0-9]+/[0-9]+/"
```

- prefixes：使用正则表达式匹配需要去除的 url 前缀。



#### 3、转发鉴权验证中间件

在 traefik 中支持简单的第三方鉴权，当访问某个服务路由规则匹配后，则可以使用该转发鉴权验证中间件将请求转发到第三方服务进行鉴权。当第三方鉴权服务返回 Http 200 状态码 traeifk 则认为有权限访问后续服务，不拦截。如果鉴权服务返回非 200 状态码 traefik 认为无权访问，进行拦截。这种中间件的写法如下：

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: test-auth
spec:
  forwardAuth:
    address: https://example.com/auth
    trustForwardHeader: true
```

- address：将权限鉴定转发到第三方进行验证，设置转发的地址。
- trustForwardHeader：信任全部 X-Forwarded-* headers 信息。

更多中间件的使用方法可也查看 Traefik 官方文档 Middlewares 部分：https://docs.traefik.io/middlewares/overview/



### 六、Traefik 路由规则中如何使用 Traefik Middleware

上面演示了如何简单的配置 Traefik Middlewares，这里我们在路由规则中使用这些中间件，同样也是基于 CRD 和 Ingress 两种路由规则进行演示。

#### 1、使用去除前缀中间件

![image-20210113165317226](D:\学习资料\笔记\k8s\k8s图\image-20210113165317226.png)

##### (1)、创建去除前缀中间件

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: stripprefix-middleware               ##设置中间件名称,要和路由规则中的名称一致 
  namespace: mydlqcloud                      ##指定 Namespace
spec:
  stripPrefix:
    prefixes:
      - /foo                                 ##设置要去除的前缀
```



#### (2)、创建路由规则并使用中间件

**CRD 方式：**

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: test-route
  namespace: mydlqcloud                      ##指定 Namespace
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`www.mydlq.club`) && PathPrefix(`/foo`)
      kind: Rule
      services:
        - name: test-service
          port: 80
      middlewares:
        - name: stripprefix-middleware       ##指定使用的中间件
```

**Ingress 方式：**

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test-ingress
  namespace: mydlqcloud                      ##指定 Namespace
  annotations:
    kubernetes.io/ingress.class: traefik
    traefik.ingress.kubernetes.io/router.entrypoints: web
    ##指定使用的 Middleware，规则是 {namespace名称}-{middleware名称}@{资源类型}
    traefik.ingress.kubernetes.io/router.middlewares: mydlqcloud-stripprefix-middleware@kubernetescrd
spec:
  rules:
  - host: www.mydlq.club                                 
    http:
      paths:
      - path: /foo            
        backend:
          serviceName: test-service
          servicePort: 80
```

#### 2、正则表达式去除前缀中间件

![image-20210113165525902](D:\学习资料\笔记\k8s\k8s图\image-20210113165525902.png)

##### (1)、创建正则表达式去除中间件

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: regex-stripprefix-middleware
  namespace: mydlqcloud
spec:
  stripPrefixRegex:
    regex:
      - ^/[a-zA-Z0-9-]+                       ##设置正则表达式
```

##### (2)、创建路由规则并使用中间件

**CRD 方式：**

```yaml
## IngressRoute
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: test-route1
  namespace: mydlqcloud
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`www.mydlq.club`) && PathPrefix(`/one`)   ##路由规则1
      kind: Rule
      services:
        - name: test-service1
          port: 80
      middlewares:
        - name: regex-stripprefix-middleware
    - match: Host(`www.mydlq.club`) && PathPrefix(`/two`)   ##路由规则2
      kind: Rule
      services:
        - name: test-service2
          port: 80
      middlewares:
        - name: regex-stripprefix-middleware
```

**Ingress 方式：**

```yaml
## Ingress
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test-ingress1
  namespace: mydlqcloud
  annotations:
    kubernetes.io/ingress.class: traefik
    traefik.ingress.kubernetes.io/router.entrypoints: web
    traefik.ingress.kubernetes.io/router.middlewares: mydlqcloud-regex-stripprefix-middleware@kubernetescrd
spec:
  rules:
  - host: www.mydlq.club                                 
    http:
      paths:
      - path: /one              ##路由规则1  
        backend:
          serviceName: test-service1
          servicePort: 80
      - path: /two              ##路由规则2
        backend:
          serviceName: test-service2
          servicePort: 80
```



#### 3、转发鉴权验证中间件

![image-20210113165717619](D:\学习资料\笔记\k8s\k8s图\image-20210113165717619.png)



##### (1)、创建转发鉴权中间件

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: authentication-middleware            ##设置中间件名称
  namespace: mydlqcloud                      ##指定 Namespace
spec:
  forwardAuth:
    address: "http://auth-service/auth"
    trustForwardHeader: true
```



##### (2)、创建路由规则并使用中间件

**CRD 方式：**

```yaml
## IngressRoute
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: test-route
  namespace: mydlqcloud
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`www.mydlq.club`) && PathPrefix(`/foo`)
      kind: Rule
      services:
        - name: test-service
          port: 80
      middlewares:
        - name: authentication-middleware
```

**Ingress 方式：**

```yaml
## IngressRoute
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test-ingress
  namespace: mydlqcloud
  annotations:
    kubernetes.io/ingress.class: traefik
    traefik.ingress.kubernetes.io/router.entrypoints: web
    traefik.ingress.kubernetes.io/router.middlewares: mydlqcloud-authentication-middleware@kubernetescrd
spec:
  rules:
  - host: www.mydlq.club                                 
    http:
      paths:
      - path: /foo            
        backend:
          serviceName: test-service
          servicePort: 80
```



### 七、Traefik 路由规则中使用多个 Traefik Middlewares

这里再演示一下使用一个路由中如何使用两个中间件，使用去除路径前缀与鉴权配合：

![image-20210113165819963](D:\学习资料\笔记\k8s\k8s图\image-20210113165819963.png)



#### 1、创建转发鉴权与去除前缀中间件

```yaml
## Middleware1
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: authentication-middleware            ##设置中间件名称
  namespace: mydlqcloud                      ##指定 Namespace
spec:
  forwardAuth:
    address: "http://auth-service/auth"
    trustForwardHeader: true
---
## Middleware2
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: regex-stripprefix-middleware
  namespace: mydlqcloud
spec:
  stripPrefixRegex:
    regex:
      - ^/[a-zA-Z0-9-]+                       ##设置正则表达式
```



#### 2、创建路由规则并使用中间件

**CRD 方式：**

```yaml
## IngressRoute
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: test-route
  namespace: mydlqcloud                      ##指定 Namespace
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`www.mydlq.club`) && PathPrefix(`/foo`)
      kind: Rule
      services:
        - name: test-service
          port: 80
      middlewares:
        - name: authentication-middleware    ##指定使用的中间件1
        - name: regex-stripprefix-middleware ##指定使用的中间件2
```

**Ingress 方式：**

```yaml
## IngressRoute
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test-ingress
  namespace: mydlqcloud                      ##指定 Namespace
  annotations:
    kubernetes.io/ingress.class: traefik
    traefik.ingress.kubernetes.io/router.entrypoints: web
    ##指定使用的 Middleware，规则是 {namespace名称}-{middleware名称}@{资源类型}，如果使用多个中间件，则逗号隔开
    traefik.ingress.kubernetes.io/router.middlewares: mydlqcloud-authentication-middleware@kubernetescrd,mydlqcloud-regex-stripprefix-middleware@kubernetescrd
spec:
  rules:
  - host: www.mydlq.club                                 
    http:
      paths:
      - path: /foo            
        backend:
          serviceName: test-service
          servicePort: 80
```

到此 Traefik 路由规则和中间件配置介绍完毕，可以访问 [Traefik 官方文档](https://docs.traefik.io/) 了解更多相关知识 。













# Kubernetes 最佳实践



## Kubernetes 服务部署最佳实践(一)



### 实现容器资源调度均衡分配，Request 与 Limit 怎么设置更方便？

如何为容器配置 Request 与 Limit，资源调度分配更方便? 这是一个既常见又棘手的问题，这个根据服务类型，需求与场景的不同而不同，没有固定的答案。



#### **所有容器都应该设置 request**

request 的值并不是指给容器实际分配的资源大小，它仅仅是给调度器看的，调度器会 "观察" 每个节点可以用于分配的资源有多少，也知道每个节点已经被分配了多少资源。被分配资源的大小就是节点上所有 Pod 中定义的容器 request 之和，它可以计算出节点剩余多少资源可以被分配(可分配资源减去已分配的 request 之和)。如果发现节点剩余可分配资源大小比当前要被调度的 Pod 的 reuqest 还小，那么就不会考虑调度到这个节点，反之，才可能调度。所以，如果不配置 request，那么调度器就不能知道节点大概被分配了多少资源出去，调度器得不到准确信息，也就无法做出合理的调度决策，很容易造成调度不合理，有些节点可能很闲，而有些节点可能很忙，甚至 NotReady。

所以，建议是**给所有容器都设置 request，让调度器感知节点有多少资源被分配了，以便做出合理的调度决策**，让集群节点的资源能够被合理的分配使用，避免陷入资源分配不均导致一些意外发生。



#### **老是忘记设置怎么办？**

有时候我们会忘记给部分容器设置 request 与 limit，其实我们可以**使用 LimitRange 来设置 namespace 的默认 request 与 limit 值，同时它也可以用来限制最小和最大的 request 与 limit。**示例:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-limit-range
  namespace: test
spec:
  limits:
  - default:
      memory: 512Mi
      cpu: 500m
    defaultRequest:
      memory: 256Mi
      cpu: 100m
    type: Container
```



#### **重要的线上应用该如何设置？**

节点资源不足时，会触发自动驱逐，将一些低优先级的 Pod 删除掉以释放资源让节点自愈。没有设置 request，limit 的 Pod 优先级最低，容易被驱逐；request 不等于 limit 的其次；request 等于 limit 的 Pod 优先级较高，不容易被驱逐。所以如果是重要的线上应用，不希望在节点故障时被驱逐，导致线上业务受影响，那么**建议将 request 和 limit 设成一致**。



#### **怎样设置才能提高资源利用率？**

如果给你的应用设置较高的 request 值，而实际占用资源长期远小于它的 request 值，会导致节点整体的资源利用率较低。当然这里对时延非常敏感的业务除外，因为敏感的业务本身不期望节点利用率过高，从而影响网络包收发速度。所以对一些**非核心，并且资源不长期占用的应用，可以适当减少 request 以提高资源利用率**。

如果你的**服务支持水平扩容，单副本的 request 值一般可以设置到不大于 1 核，CPU 密集型应用除外**。比如 coredns，设置到 0.1 核就可以，即 100m。



#### **尽量避免使用过大的 request 与 limit**

如果你的服务使用单副本或者少量副本，给很大的 request 与 limit，让它分配到足够多的资源来支撑业务，那么某个副本故障对业务带来的影响可能就比较大，并且由于 request 较大，当集群内资源分配比较碎片化，如果这个 Pod 所在节点挂了，其它节点又没有一个有足够的剩余可分配资源能够满足这个 Pod 的 request 时，这个 Pod 就无法实现漂移，也就不能自愈，反而加重对业务的影响。

相反，**建议尽量减小 request 与 limit，通过增加副本的方式来对你的服务支撑能力进行水平扩容，让你的系统更加灵活可靠**。



#### **避免测试 namespace 消耗过多资源，影响生产业务**

若生产集群有用于测试的 namespace，如果不加以限制，可能导致集群负载过高，从而影响生产业务。可以使用 ResourceQuota 来限制测试 namespace 的 request 与 limit 的总大小。示例:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota-test
  namespace: test
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
```



### 如何让资源得到更合理的分配？

设置 Request 能够解决让 Pod 调度到有足够资源的节点上，但无法做到更细致的控制。如何进一步让资源得到合理的使用？我们可以结合亲和性、污点与容忍等高级调度技巧，让 Pod 能够被合理调度到合适的节点上，让资源得到充分的利用。



#### **使用亲和性**

对节点有特殊要求的服务可以用节点亲和性 (Node Affinity) 部署，以便调度到符合要求的节点，比如让 MySQL 调度到高 IO 的机型以提升数据读写效率。可以将需要离得比较近的有关联的服务用 Pod 亲和性 (Pod Affinity) 部署，比如让 Web 服务跟它的 Redis 缓存服务都部署在同一可用区，实现低延时。也可使用 Pod 反亲和 (Pod AntiAffinity) 将 Pod 进行打散调度，避免单点故障或者流量过于集中导致的一些问题。



#### **使用污点与容忍**

使用污点 (Taint) 与容忍 (Toleration) 可优化集群资源调度:

通过给节点打污点来给某些应用预留资源，避免其它 Pod 调度上来。需要使用这些资源的 Pod 加上容忍，结合节点亲和性让它调度到预留节点，即可使用预留的资源。



### 如何实现业务的弹性伸缩？

#### **支持流量突发型业务，如何应对？**

通常业务都会有高峰和低谷，为了更合理的利用资源，我们为服务定义 HPA，实现根据 Pod 的资源实际使用情况来对服务进行自动扩缩容，在业务高峰时自动扩容 Pod 数量来支撑服务，在业务低谷时，自动缩容 Pod 释放资源，以供其它服务使用（比如在夜间，线上业务低峰，自动缩容释放资源以供大数据之类的离线任务运行) 。

使用 HPA 前提是让 K8S 知道你服务的实际资源占用情况(指标数据)，需要安装 resource metrics (metrics.k8s.io) 或 custom metrics (custom.metrics.k8s.io) 的实现，好让 hpa controller 查询这些 API 来获取到服务的资源占用情况。早期 HPA 用 resource metrics 获取指标数据，后来推出 custom metrics，可以实现更灵活的指标来控制扩缩容。官方有个叫 metrics-server 的实现，通常社区使用更多的是基于 prometheus 实现 prometheus-adapter，而云厂商托管的 K8S 集群通常集成了自己的实现，比如 TKE，实现了 CPU、内存、硬盘、网络等维度的指标，可以在网页控制台可视化创建 HPA，但最终都会转成 K8S 的 yaml，示例:

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx
spec:
  scaleTargetRef:
    apiVersion: apps/v1beta2
    kind: Deployment
    name: nginx
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metric:
        name: k8s_pod_rate_cpu_core_used_request
      target:
        averageValue: "100"
        type: AverageValue
```



#### **如何节约成本？**

HPA 能实现 Pod 水平扩缩容，但如果节点资源不够用了，Pod 扩容出来还是会 Pending。如果我们提前准备好大量节点，做好资源冗余，提前准备好大量节点，通常不会有 Pod Pending 的问题，但也意味着需要付出更高的成本。通常云厂商托管的 K8S 集群都会实现 cluster-autoscaler，即根据资源使用情况，动态增删节点，让计算资源能够被最大化的弹性使用，按量付费，以节约成本。在 TKE 上的实现叫做伸缩组，以及一个包含伸缩功能组但更高级的特性：节点池(正在灰度)



#### **无法水平扩容的服务怎么办？**

对于无法适配水平伸缩的单体应用，或者不确定最佳 request 与 limit 超卖比的应用，可以尝用 VPA 来进行垂直伸缩，即自动更新 request 与 limit，然后重启 pod。不过这个特性容易导致你的服务出现短暂的不可用，不建议在生产环境中大规模使用。



### 参考资料

- Understanding Kubernetes limits and requests by example: https://sysdig.com/blog/kubernetes-limits-requests/
- Understanding resource limits in kubernetes: cpu time: https://medium.com/@betz.mark/understanding-resource-limits-in-kubernetes-cpu-time-9eff74d3161b
- Understanding resource limits in kubernetes: memory: https://medium.com/@betz.mark/understanding-resource-limits-in-kubernetes-memory-6b41e9a955f9
- Kubernetes best practices: Resource requests and limits: https://cloud.google.com/blog/products/gcp/kubernetes-best-practices-resource-requests-and-limits
- Kubernetes 资源分配之 Request 和 Limit 解析: https://cloud.tencent.com/developer/article/1004976
- Assign Pods to Nodes using Node Affinity: https://kubernetes.io/docs/tasks/configure-pod-container/assign-pods-nodes-using-node-affinity/
- Taints and Tolerations: https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/
- metrics-server: https://github.com/kubernetes-sigs/metrics-server
- prometheus-adapter: https://github.com/DirectXMan12/k8s-prometheus-adapter
- cluster-autoscaler: https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler
- VPA: https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscalerand-limits



## Kubernetes 服务部署最佳实践(二) ——如何提高服务可用性



上一篇文章我们围绕**如何合理利用资源**的主题做了一些**最佳实践的分享**，这一次我们就如何提高服务可用性的主题来展开探讨。

**怎样提高我们部署服务的可用性呢？**

K8S 设计本身就考虑到了各种故障的可能性，并提供了一些自愈机制以提高系统的容错性，但有些情况还是可能导致较长时间不可用，拉低服务可用性的指标。**本文将结合生产实践经验，为大家提供一些最佳实践来最大化的提高服务可用性。**



### 如何避免单点故障？

K8S 的设计就是假设节点是不可靠的。节点越多，发生软硬件故障导致节点不可用的几率就越高，所以我们通常需要给服务部署多个副本，根据实际情况调整 replicas 的值，如果值为 1 就必然存在单点故障，如果大于 1 但所有副本都调度到同一个节点了，那还是有单点故障，有时候还要考虑到灾难，比如整个机房不可用。

所以我们不仅要有合理的副本数量，还需要让这些不同副本调度到不同的拓扑域(节点、可用区)，打散调度以避免单点故障，这个可以利用 Pod 反亲和性来做到，反亲和主要分强反亲和与弱反亲和两种。更多亲和与反亲和信息可参考官方文档**Affinity and anti-affinity**[1]。

先来看个强反亲和的示例，将 DNS 服务强制打散调度到不同节点上:

```yaml
affinity:
 podAntiAffinity:
   requiredDuringSchedulingIgnoredDuringExecution:
   - labelSelector:
       matchExpressions:
       - key: k8s-app
         operator: In
         values:
         - kube-dns
     topologyKey: kubernetes.io/hostname
```

- `labelSelector.matchExpressions` 写该服务对应 pod 中 labels 的 key 与 value，因为 Pod 反亲和性是通过判断 replicas 的 pod label 来实现的。
- `topologyKey` 指定反亲和的拓扑域，即节点 label 的 key。这里用的 `kubernetes.io/hostname` 表示避免 pod 调度到同一节点，如果你有更高的要求，比如避免调度到同一个可用区，实现异地多活，可以用 `failure-domain.beta.kubernetes.io/zone`。通常不会去避免调度到同一个地域，因为一般同一个集群的节点都在一个地域，如果跨地域，即使用专线时延也会很大，所以 `topologyKey` 一般不至于用 `failure-domain.beta.kubernetes.io/region`。
- `requiredDuringSchedulingIgnoredDuringExecution` 调度时必须满足该反亲和性条件，如果没有节点满足条件就不调度到任何节点 (Pending)。

如果不用这种硬性条件可以使用 `preferredDuringSchedulingIgnoredDuringExecution` 来指示调度器尽量满足反亲和性条件，即弱反亲和性，如果实在没有满足条件的，只要节点有足够资源，还是可以让其调度到某个节点，至少不会 Pending。

我们再来看个弱反亲和的示例:

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchExpressions:
          - key: k8s-app
            operator: In
            values:
            - kube-dns
      topologyKey: kubernetes.io/hostname
```

注意到了吗？相比强反亲和有些不同哦，多了一个 `weight`，表示此匹配条件的权重，而匹配条件被挪到了 `podAffinityTerm` 下面。



### 如何避免节点维护或升级时导致服务不可用？

有时候我们需要对节点进行维护或进行版本升级等操作，操作之前需要对节点执行驱逐 (kubectl drain)，驱逐时会将节点上的 Pod 进行删除，以便它们漂移到其它节点上，当驱逐完毕之后，节点上的 Pod 都漂移到其它节点了，这时我们就可以放心的对节点进行操作了。

有一个问题就是，驱逐节点是一种有损操作，驱逐的原理:

1. 封锁节点 (设为不可调度，避免新的 Pod 调度上来)。
2. 将该节点上的 Pod 删除。
3. ReplicaSet 控制器检测到 Pod 减少，会重新创建一个 Pod，调度到新的节点上。

这个过程是先删除，再创建，并非是滚动更新，因此更新过程中，如果一个服务的所有副本都在被驱逐的节点上，则可能导致该服务不可用。

我们再来下什么情况下驱逐会导致服务不可用:

1. 服务存在单点故障，所有副本都在同一个节点，驱逐该节点时，就可能造成服务不可用。
2. 服务没有单点故障，但刚好这个服务涉及的 Pod 全部都部署在这一批被驱逐的节点上，所以这个服务的所有 Pod 同时被删，也会造成服务不可用。
3. 服务没有单点故障，也没有全部部署到这一批被驱逐的节点上，但驱逐时造成这个服务的一部分 Pod 被删，短时间内服务的处理能力下降导致服务过载，部分请求无法处理，也就降低了服务可用性。

针对第一点，我们可以使用前面讲的反亲和性来避免单点故障。

针对第二和第三点，我们可以通过配置 PDB (PodDisruptionBudget) 来避免所有副本同时被删除，驱逐时 K8S 会 "观察" nginx 的当前可用与期望的副本数，根据定义的 PDB 来控制 Pod 删除速率，达到阀值时会等待 Pod 在其它节点上启动并就绪后再继续删除，以避免同时删除太多的 Pod 导致服务不可用或可用性降低，下面给出两个示例。

示例一 (保证驱逐时 nginx 至少有 90% 的副本可用):

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchExpressions:
          - key: k8s-app
            operator: In
            values:
            - kube-dns
      topologyKey: kubernetes.io/hostname
```

示例二 (保证驱逐时 zookeeper 最多有一个副本不可用，相当于逐个删除并等待在其它节点完成重建):

```yaml
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: zk-pdb
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app: zookeeper
```



### 如何让服务进行平滑更新？

解决了服务单点故障和驱逐节点时导致的可用性降低问题后，我们还需要考虑一种可能导致可用性降低的场景，那就是滚动更新。为什么服务正常滚动更新也可能影响服务的可用性呢？别急，下面我来解释下原因。

假如集群内存在服务间调用:

![image-20210108094142662](D:\学习资料\笔记\k8s\k8s图\image-20210108094142662.png)



当 server 端发生滚动更新时:

![image-20210108094226094](D:\学习资料\笔记\k8s\k8s图\image-20210108094226094.png)

发生两种尴尬的情况:

1. 旧的副本很快销毁，而 client 所在节点 kube-proxy 还没更新完转发规则，仍然将新连接调度给旧副本，造成连接异常，可能会报 "connection refused" (进程停止过程中，不再接受新请求) 或 "no route to host" (容器已经完全销毁，网卡和 IP 已不存在)。
2. 新副本启动，client 所在节点 kube-proxy 很快 watch 到了新副本，更新了转发规则，并将新连接调度给新副本，但容器内的进程启动很慢 (比如 Tomcat 这种 java 进程)，还在启动过程中，端口还未监听，无法处理连接，也造成连接异常，通常会报 "connection refused" 的错误。

针对第一种情况，可以给 container 加 preStop，让 Pod 真正销毁前先 sleep 等待一段时间，等待 client 所在节点 kube-proxy 更新转发规则，然后再真正去销毁容器。这样能保证在 Pod Terminating 后还能继续正常运行一段时间，这段时间如果因为 client 侧的转发规则更新不及时导致还有新请求转发过来，Pod 还是可以正常处理请求，避免了连接异常的发生。听起来感觉有点不优雅，但实际效果还是比较好的，分布式的世界没有银弹，我们只能尽量在当前设计现状下找到并实践能够解决问题的最优解。

针对第二种情况，可以给 container 加 ReadinessProbe (就绪检查)，让容器内进程真正启动完成后才更新 Service 的 Endpoint，然后 client 所在节点 kube-proxy 再更新转发规则，让流量进来。这样能够保证等 Pod 完全就绪了才会被转发流量，也就避免了链接异常的发生。

最佳实践 yaml 示例:

```yaml
      readinessProbe:
          httpGet:
            path: /healthz
            port: 80
            httpHeaders:
            - name: X-Custom-Header
              value: Awesome
          initialDelaySeconds: 10
          timeoutSeconds: 1
        lifecycle:
          preStop:
            exec:
              command: ["/bin/bash", "-c", "sleep 10"]
```



### 健康检查怎么配才好？

我们都知道，给 Pod 配置健康检查也是提高服务可用性的一种手段，配置 ReadinessProbe (就绪检查) 可以避免将流量转发给还没启动完全或出现异常的 Pod；配置 LivenessProbe (存活检查) 可以让存在 bug 导致死锁或 hang 住的应用重启来恢复。但是，如果配置配置不好，也可能引发其它问题，这里根据一些踩坑经验总结了一些指导性的建议：

- 不要轻易使用 LivenessProbe，除非你了解后果并且明白为什么你需要它，参考 **Liveness Probes are Dangerous**[3]
- 如果使用 LivenessProbe，不要和 ReadinessProbe 设置成一样 (failureThreshold 更大)
- 探测逻辑里不要有外部依赖 (db, 其它 pod 等)，避免抖动导致级联故障
- 业务程序应尽量暴露 HTTP 探测接口来适配健康检查，避免使用 TCP 探测，因为程序 hang 死时， TCP 探测仍然能通过 (TCP 的 SYN 包探测端口是否存活在内核态完成，应用层不感知)



### 参考资料

[1]Affinity and anti-affinity: *https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinit*

[2]Specifying a Disruption Budget for your Application : *https://kubernetes.io/docs/tasks/run-application/configure-pdb/*

[3]Liveness Probes are Dangerous: *https://srcco.de/posts/kubernetes-liveness-probes-are-dangerous.html*







## 大型Kubernetes集群的资源编排优化



### 背景

云原生这个词想必大家应该不陌生了，容器是云原生的重要基石，而Kubernetes经过这几年的快速迭代发展已经成为容器编排的事实标准了。越来越多的公司不论是大公司还是中小公司已经在他们的生产环境中开始使用Kubernetes, 原生Kubernetes虽然已经提供了一套非常完整的资源调度及管理方案，但是在实际使用过程中还是会碰到很多问题：

1. 集群节点负载不均衡的问题
2. 业务创建Pod资源申请不合理的问题
3. 业务如何更快速的扩容问题
4. 多租户资源抢占问题

这些问题可能是大家在使用Kubernetes的过程中应该会经常遇到的几个比较典型的资源问题，接下来我们将分别介绍在腾讯内部是如何解决和优化这些问题的。



### 集群节点负载不均衡的问题

我们知道Kubernetes原生的调度器多是基于Pod Request的资源来进行调度的，没有根据Node当前和过去一段时间的真实负载情况进行相关调度的决策。这样就会导致一个问题在集群内有些节点的剩余可调度资源比较多但是真实负载却比较高，而另一些节点的剩余可调度资源比较少但是真实负载却比较低, 但是这时候Kube-scheduler会优先将Pod调度到剩余资源比较多的节点上（根据LeastRequestedPriority策略）。

![image-20210108205105300](D:\学习资料\笔记\k8s\k8s图\image-20210108205105300.png)

如上图，很显然调度到Node1是一个更优的选择。为了将Node的真实负载情况加到调度策略里，避免将Pod调度到高负载的Node上，同时保障集群中各Node的真实负载尽量均衡，我们扩展了Kube-scheduler实现了一个基于Node真实负载进行预选和优选的动态调度器（Dynamic-scheduler)。

![image-20210108205221198](D:\学习资料\笔记\k8s\k8s图\image-20210108205221198.png)

Dynamic-scheduler在调度的时候需要各Node上的负载数据，为了不阻塞动态调度器的调度这些负载数据，需要有模块定期去收集和记录。如下图所示node-annotator会定期收集各节点过去5分钟，1小时，24小时等相关负载数据并记录到Node的annotation里，这样Dynamic-scheduler在调度的时候只需要查看Node的annotation，便能很快获取该节点的历史负载数据。

![image-20210108205330171](D:\学习资料\笔记\k8s\k8s图\image-20210108205330171.png)

为了避免Pod调度到高负载的Node上，需要先通过预选把一些高负载的Node过滤掉，如下图所示（其中的过滤策略和比例是可以动态配置的，可以根据集群的实际情况进行调整）Node2过去5分钟的负载，Node3过去1小时的负载多超过了对应的域值，所以不会参与接下来的优选阶段。

![image-20210108205432923](D:\学习资料\笔记\k8s\k8s图\image-20210108205432923.png)

同时为了使集群各节点的负载尽量均衡，Dynamic-scheduler会根据Node负载数据进行打分, 负载越低打分越高。如下图所示Node1的打分最高将会被优先调度（这些打分策略和权重也是可以动态配置的）。

![image-20210108205512935](D:\学习资料\笔记\k8s\k8s图\image-20210108205512935.png)

Dynamic-scheduler只能保证在调度的那个时刻会将Pod调度到低负载的Node上，但是随着业务的高峰期不同Pod在调度之后，这个Node可能会出现高负载。为了避免由于Node的高负载对业务产生影响，我们在Dynamic-scheduler之外还实现了一个Descheduler，它会监控Node的高负载情况，将一些配置了高负载迁移的Pod迁移到负载比较低的Node上。

![image-20210108205611877](D:\学习资料\笔记\k8s\k8s图\image-20210108205611877.png)



### 业务如何更快速的扩容问题

业务上云的一个重要优势就是在云上能实现更好的弹性伸缩，Kubernetes原生也提供了这种横向扩容的弹性伸缩能力（HPA)。但是官方的这个HPA Controller在实现的时候用的是一个Gorountine来处理整个集群的所有HPA的计算和同步问题，在集群配置的HPA比较多的时候可能会导致业务扩容不及时的问题，其次官方HPA Controller不支持为每个HPA进行单独的个性化配置。

![image-20210108205713357](D:\学习资料\笔记\k8s\k8s图\image-20210108205713357.png)

为了优化HPA Controller的性能和个性化配置问题，我们把HPA Controller单独抽离出来单独部署。同时为每一个HPA单独配置一个Gorountine，并且每一个HPA多可以根据业务的需要进行单独的配置。

![image-20210108205815989](D:\学习资料\笔记\k8s\k8s图\image-20210108205815989.png)

其实仅仅优化HPA Controller还是不能满足一些业务在业务高峰时候的一些需求，比如在业务做活动的时候，希望在流量高峰期之前就能够把业务扩容好。这个时候我们就需要一个定时HPA的功能，为此我们定义了一个CronHPA的CRD和CronHPA Operator。CronHPA会在业务定义的时间进行扩容和缩容，同时还能和HPA一起配合工作。

![image-20210108205922036](D:\学习资料\笔记\k8s\k8s图\image-20210108205922036.png)



### 业务创建Pod资源申请不合理的问题

通过Dynamic-scheduler和Descheduler来保障集群各节点的负载均衡问题。但是我可能会面临另一个比较头疼的问题，就是集群的整体负载比较低但是可调度资源已经没有了，从而导致Pod Pending。这个往往是由于Pod 资源申请不合理或者业务高峰时段不同所导致的，那我们是否可以根据Node负载情况对Node资源进行一定比例的超卖呢？于是我们通过Kubernetes的MutatingWebhook来截获并修改Node的可调度资源量的方式，来对Node资源进行超卖。

![image-20210108210116048](D:\学习资料\笔记\k8s\k8s图\image-20210108210116048.png)

这里需要注意的是节点的超卖控制需要比较灵活，不能一概而论，比如负载高的Node超卖比例应该要设置比较小或者不能设置超卖。



### 多租户资源抢占问题

当平台用户增多的时候，如果对资源不做任何控制，那么各租户之间资源抢占是不可避免的。Kubernetes原生提供的ResourceQuota，可以提供Namespace级别对资源配额限制。但是腾讯内部资源预算管理通常是按产品维度，而一个产品可能会包括很多Namespace的，显然ResourceQuota不太适合这种场景。其次ResourceQuota只有资源限制功能，不能做资源预留，当业务要做活动的时候不能保证活动期间有足够的资源可以使用。

![image-20210108210244087](D:\学习资料\笔记\k8s\k8s图\image-20210108210244087.png)

为了实现一个产品维度且有资源预留功能的配额管理功能，我们设计了一个DynamicQuota的CRD来管理产品在各集群的配额。如下图所示产品2在各集群所占的配额资源其它产品多无法使用。

![image-20210108210336974](D:\学习资料\笔记\k8s\k8s图\image-20210108210336974.png)

如果一个产品占用配额一直不使用就可能会导致平台资源的浪费，因此我们在产品配额预留的基础上提供了在不同产品间配额借调的功能。如下图所示产品1暂时不用的配额可以借调给产品2临时使用。

![image-20210108210419474](D:\学习资料\笔记\k8s\k8s图\image-20210108210419474.png)

当平台有多集群的时候，产品配额需要如何分配。为了简化配下发操作，如下图所示管理员在下发产品配额的时候只需配置一个该产品的配额总量，配额下发模块会根据产品目前在各集群的使用情况按比例分配到各个集群。

![image-20210108210532636](D:\学习资料\笔记\k8s\k8s图\image-20210108210532636.png)

产品在各集群的资源使用情况是会时刻变化的，所以产品在各集群配额也需要根据业务的使用情况进行动态的调整。如下图所示产品在集群2中的配额已经快用完的时候，配额调整模块会动态的把配额从使用不多的集群1和集群3调到集群2。

![image-20210108210655451](D:\学习资料\笔记\k8s\k8s图\image-20210108210655451.png)

在线业务和离线业务混合部署是云平台的发展趋势，所以我们在设计配额管理的时候也把在离线配额控制考虑进去了。如下图所示我们在集群维度加了一个离线配额控制，一个集群的离线业务资源使用不能超过该集群总资源的30%（这个比例可以根据实际情况进行调整）。其次一个产品的在线业务和离线业务配额之和又不能超过产品在该集群的总配额。

![image-20210108210746251](D:\学习资料\笔记\k8s\k8s图\image-20210108210746251.png)



### 总结

上面提到的方案只是简单说了一下我们的一些解决问题的思路，其实在真正运作的过程中还有很多细节需要考虑和优化。比如：上面提到的产品配额管理，如果一个产品的配额不足了，这时候业务有高峰需要进行HPA扩容，配额管理模块需要对这种扩容优化并放行。





## 为什么已经用了滚动更新服务还会中断



滚动更新作为一个最佳实践，是每个服务在变更时都会采纳的方案。但在 Kubernetes 实践中，即便使用了滚动更新，也并不一定能够保证服务在更新和维护时总是可用的。



### **滚动更新的原理**

在 Kubernetes 中，我们一般通过 Deployment、Daemonset 等控制器管理 Pod，并且把他们放到 Service 后面，使用 Service 的虚拟 IP 或者负载均衡器 IP 去访问。在 Pod 配置变更（如更新镜像）时，这些控制器默认就会采用滚动更新的方式逐步用新 Pod 替换已有的 Pod。下图所示就是一个典型的**滚动更新**[1]过程：

![image-20210119083559246](D:\学习资料\笔记\k8s\k8s图\image-20210119083559246.png)

由于新的 Pod Ready 之后才会去删除旧的 Pod，在滚动更新中新的连接过来会自动路由到健康的 Pod 上，所以一般来说，新连接不会出问题，容易出问题的是旧连接。

这儿最容易想到的就是长连接。由于旧 Pod 最终会被删除，已有的长连接总是需要关闭。对这种长连接问题，想要解决，最好的方法是客户端在连接断开后重新建立连接。

而对短连接来说，是不是说就一定没问题呢？其实并不一定。



### **哪些问题会导致滚动更新时的服务中断**

#### 已有Pod过早终止

如果 Pod2 在终止的时候还有未处理完成的连接，那这些连接势必会失败。所以，在终止 Pod2 的时候，需要采用优雅关闭的方式，等待已有连接处理完成之后再终止。

比如，对 Nginx Pod 来说，可以这么做

```YAML
lifecycle:
  preStop:
    exec:
      command: [
        # Gracefully shutdown nginx
        "/usr/sbin/nginx", "-s", "quit"
      ]
```



#### 新Pod未初始化完成就收到外部请求

很多容器启动时都有一个初始化的过程，虽然 Pod 处于 Running 状态了，但实际上进程还在初始化过程中，不能处理外部过来的请求。所以，在 Pod 启动过程中，需要一种机制，等着容器进程初始化完成之后再接收外部过来的请求。

这个问题比较好解决，Kubernetes 已经提供了 Readiness 探针，只需要应用提供一个探针接口即可。比如

```yaml
        readinessProbe:
          failureThreshold: 3
          initialDelaySeconds: 5
          periodSeconds: 10
          httpGet:
            port: 80
            path: /
```



#### 异步操作延迟导致iptables中没有健康Endpoint

滚动更新涉及到 kube-apiserver、kubelet、kube-controller-manager（包括 endpoint controller、service controller 和 cloud provider）以及 CNI 插件等。假设新建Pod的名字为Pod2，而旧的Pod名字为Pod1，这些组件在滚动更新过程中的典型过程如下图所示

![image-20210119083909316](D:\学习资料\笔记\k8s\k8s图\image-20210119083909316.png)

注意 Endpoints 更新（加入新 Pod2 IP 和删除旧 Pod1 IP）以及以后的步骤都是异步的。如果 Pod1 的 IP 摘除时间过早，Pod2 的 IP 还没有更新到 iptables 中，那么新的连接进来就会因为没有健康 Pod 而无法连接。

要解决这个问题不容易，但有一个简单的方法可以绕过去，即在 **Zero Downtime Server Updates For Your Kubernetes Cluster**[2] 中提到的利用 PreStop Hook 主动等一段时间之后再执行优雅关闭。

```yaml
 lifecycle:
          preStop:
            exec:
              command: [
                "sh", "-c",
                # Introduce a delay to the shutdown sequence to wait for the
                # pod eviction event to propagate. Then, gracefully shutdown
                # nginx.
                "sleep 15 && /usr/sbin/nginx -s quit",
              ]
```



#### 集群维护导致所有Pod同时删除

在集群常规或者异常维护过程中，管理员经常需要驱逐一个或多个异常节点，把这些节点之上的 Pod 迁移到其他节点上面去。如果一个应用的所有 Pod 刚好在这些节点上，那就有可能所有 Pod 都被同时驱逐了，导致一段时间内没有任何健康的容器在运行。

Kubernetes 也为这个问题提供了一种很好的解决方法，即使用 **PodDisruptionBudget**[3] 给应用设置中断预算，避免所有 Pod 被同时重启。

```yaml
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: nginx
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: nginx
```



#### 负载均衡器健康检测延迟

使用负载均衡器访问 Service 并且设置了 externalTrafficPolicy 为 Local（为了保留请求原始地址）时，除了上述提到的这些因素，负载均衡器本身提供的健康检测机制也可能会导致新连接短时间内的超时问题。

比如，在执行 kubectl drain node 的同时，对服务进行压力测试，就会发现部分连接断开（下面的例子成功率只有 97.27%）：

```sh
Requests      [total, rate, throughput]         299988, 4999.56, 4856.10
Duration      [total, attack, wait]             1m0s, 1m0s, 87.815ms
Latencies     [min, mean, 50, 90, 95, 99, max]  65.523ms, 866.673ms, 80.412ms, 2.409s, 5.066s, 10.003s, 10.367s
Bytes In      [total, mean]                     178585272, 595.31
Bytes Out     [total, mean]                     0, 0.00
Success       [ratio]                           97.27%
Status Codes  [code:count]                      0:8182  200:291806
Error Set:
context deadline exceeded (Client.Timeout or context cancellation while reading body)
```

这是为什么呢？

- 通常，负载均衡器后端放置的是所有的 Node，利用每个 Service 的 NodePort 来访问 Service。
- 当一个 Pod 被标记为 Terminating 状态时，Pod IP 会被 kube-controller-manager 立刻从 Endpoints 中摘除。
- 这之后，kube-proxy 就会把相应的 IP 从 iptables 中摘除掉，而负载均衡器此时还会继续把新请求发送到该 Pod 所在节点上。
- 由于 Pod IP 已经从 iptables 中清除了，新转发过来的请求就会失败。

对这个问题，一个最简单的方法就是把新的 Pod 调度到已有 Pod 所在节点上，确保 iptables 之后总是有健康的 Pod。

但这个方法不适用于节点驱逐的场景，毕竟节点驱逐之后不允许任何 Pod 继续运行了。所以，在节点驱逐的场景中，应该先从负载均衡器中把节点摘除，确保没有任何请求转发到节点之后，再去执行驱逐操作。



### **最佳实践**

- 所有应用都使用控制器管理，并且必须多副本运行，尽量将副本分散到不同节点上。
- 为所有 Pod 添加 livenessProbe 和 readinessProbe。
- 容器进程在收到 SIGTERM 信号后优雅终止，比如持久化数据、清理网络连接等。
- 终止之前利用 preStop 稍等一会，等待各个组件异步操作完成。
- 必要时才设置 externalTrafficPolicy 为 Local，保留请求原始 IP。
- 使用 PodDiscruptionBudget 为应用设置中断预算，并总是使用 Eviction API（比如 kubectl drain）来清理 Pod，以确保遵循 PodDiscruptionBudget 的配置。

基于这些最佳实践，一个简单的 Nginx 配置就如下所示：

```yaml
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: nginx
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  ...
  template:
    spec:
      terminationGracePeriodSeconds: 30
      containers:
      - image: nginx
        name: nginx
        readinessProbe:
          failureThreshold: 3
          initialDelaySeconds: 5
          periodSeconds: 10
          httpGet:
            port: 80
            path: /
        lifecycle:
          preStop:
            exec:
              command: [
                "sh", "-c",
                # Introduce a delay to the shutdown sequence to wait for the
                # pod eviction event to propagate. Then, gracefully shutdown
                # nginx.
                "sleep 15 && /usr/sbin/nginx -s quit",
              ]
```

完整的 Nginx 示例以及压力测试步骤请参考 **Kubernetes Handbook**[4]。





## 大规模集群优化



Kubernetes 自 v1.6 以来，官方就宣称单集群最大支持 5000 个节点。不过这只是理论上，在具体实践中从 0 到 5000，还是有很长的路要走，需要见招拆招。

官方标准如下：

- 不超过 5000 个节点
- 不超过 150000 个 pod
- 不超过 300000 个容器
- 每个节点不超过 100 个 pod



### 内核参数调优

----



```sh
# max-file 表示系统级别的能够打开的文件句柄的数量， 一般如果遇到文件句柄达到上限时，会碰到
# "Too many open files" 或者 Socket/File: Can’t open so many files 等错误
fs.file-max=1000000
# 配置 arp cache 大小
# 存在于 ARP 高速缓存中的最少层数，如果少于这个数，垃圾收集器将不会运行。缺省值是 128
net.ipv4.neigh.default.gc_thresh1=1024
# 保存在 ARP 高速缓存中的最多的记录软限制。垃圾收集器在开始收集前，允许记录数超过这个数字 5 秒。缺省值是 512
net.ipv4.neigh.default.gc_thresh2=4096
# 保存在 ARP 高速缓存中的最多记录的硬限制，一旦高速缓存中的数目高于此，垃圾收集器将马上运行。缺省值是 1024
net.ipv4.neigh.default.gc_thresh3=8192
# 以上三个参数，当内核维护的 arp 表过于庞大时候，可以考虑优化
# 允许的最大跟踪连接条目，是在内核内存中 netfilter 可以同时处理的“任务”（连接跟踪条目）
net.netfilter.nf_conntrack_max=10485760
net.netfilter.nf_conntrack_tcp_timeout_established=300
# 哈希表大小（只读）（64位系统、8G内存默认 65536，16G翻倍，如此类推）
net.netfilter.nf_conntrack_buckets=655360
# 每个网络接口接收数据包的速率比内核处理这些包的速率快时，允许送到队列的数据包的最大数目
net.core.netdev_max_backlog=10000
# 默认值: 128 指定了每一个 real user ID 可创建的 inotify instatnces 的数量上限
fs.inotify.max_user_instances=524288
# 默认值: 8192 指定了每个inotify instance相关联的watches的上限
fs.inotify.max_user_watches=524288
```



### ETCD 优化

#### 高可用部署

部署一个高可用ETCD集群可以参考官方文档: https://github.com/etcd-io/etcd/blob/master/Documentation/op-guide/clustering.md

> 如果是 self-host 方式部署的集群，可以用 etcd-operator 部署 etcd 集群；也可以使用另一个小集群专门部署 etcd (使用 etcd-operator)



#### 提高磁盘 IO 性能

ETCD 对磁盘写入延迟非常敏感，对于负载较重的集群建议磁盘使用 SSD 固态硬盘。可以使用 diskbench 或 fio 测量磁盘实际顺序 IOPS。



#### 提高 ETCD 的磁盘 IO 优先级

由于 ETCD 必须将数据持久保存到磁盘日志文件中，因此来自其他进程的磁盘活动可能会导致增加写入时间，结果导致 ETCD 请求超时和临时 leader 丢失。当给定高磁盘优先级时，ETCD 服务可以稳定地与这些进程一起运行:

```sh
sudo ionice -c2 -n0 -p $(pgrep etcd)
```



#### 提高存储配额

默认 ETCD 空间配额大小为 2G，超过 2G 将不再写入数据。通过给 ETCD 配置 `--quota-backend-bytes` 参数增大空间配额，最大支持 8G。



#### 分离 events 存储

集群规模大的情况下，集群中包含大量节点和服务，会产生大量的 event，这些 event 将会对 etcd 造成巨大压力并占用大量 etcd 存储空间，为了在大规模集群下提高性能，可以将 events 存储在单独的 ETCD 集群中。

配置 kube-apiserver：

```sh
--etcd-servers="http://etcd1:2379,http://etcd2:2379,http://etcd3:2379" --etcd-servers-overrides="/events#http://etcd4:2379,http://etcd5:2379,http://etcd6:2379"
```



#### 减小网络延迟

如果有大量并发客户端请求 ETCD leader 服务，则可能由于网络拥塞而延迟处理 follower 对等请求。在 follower 节点上的发送缓冲区错误消息：

```sh
dropped MsgProp to 247ae21ff9436b2d since streamMsg's sending buffer is full
dropped MsgAppResp to 247ae21ff9436b2d since streamMsg's sending buffer is full
```

可以通过在客户端提高 ETCD 对等网络流量优先级来解决这些错误。在 Linux 上，可以使用 tc 对对等流量进行优先级排序：

```sh
$ tc qdisc add dev eth0 root handle 1: prio bands 3
$ tc filter add dev eth0 parent 1: protocol ip prio 1 u32 match ip sport 2380 0xffff flowid 1:1
$ tc filter add dev eth0 parent 1: protocol ip prio 1 u32 match ip dport 2380 0xffff flowid 1:1
$ tc filter add dev eth0 parent 1: protocol ip prio 2 u32 match ip sport 2379 0xffff flowid 1:1
$ tc filter add dev eth0 parent 1: protocol ip prio 2 u32 match ip dport 2379 0xffff flowid 1:1
```



### Master 节点配置优化

GCE 推荐配置：

- 1-5 节点: n1-standard-1
- 6-10 节点: n1-standard-2
- 11-100 节点: n1-standard-4
- 101-250 节点: n1-standard-8
- 251-500 节点: n1-standard-16
- 超过 500 节点: n1-standard-32

AWS 推荐配置：

- 1-5 节点: m3.medium
- 6-10 节点: m3.large
- 11-100 节点: m3.xlarge
- 101-250 节点: m3.2xlarge
- 251-500 节点: c4.4xlarge
- 超过 500 节点: c4.8xlarge

对应 CPU 和内存为：

- 1-5 节点: 1vCPU 3.75G内存
- 6-10 节点: 2vCPU 7.5G内存
- 11-100 节点: 4vCPU 15G内存
- 101-250 节点: 8vCPU 30G内存
- 251-500 节点: 16vCPU 60G内存
- 超过 500 节点: 32vCPU 120G内存



### kube-apiserver 优化

#### 高可用

- 方式一: 启动多个 kube-apiserver 实例通过外部 LB 做负载均衡。
- 方式二: 设置 `--apiserver-count` 和 `--endpoint-reconciler-type`，可使得多个 kube-apiserver 实例加入到 Kubernetes Service 的 endpoints 中，从而实现高可用。

不过由于 TLS 会复用连接，所以上述两种方式都无法做到真正的负载均衡。为了解决这个问题，可以在服务端实现限流器，在请求达到阀值时告知客户端退避或拒绝连接，客户端则配合实现相应负载切换机制。



#### 控制连接数

kube-apiserver 以下两个参数可以控制连接数:

```sh
--max-mutating-requests-inflight int           The maximum number of mutating requests in flight at a given time. When the server exceeds this, it rejects requests. Zero for no limit. (default 200)
--max-requests-inflight int                    The maximum number of non-mutating requests in flight at a given time. When the server exceeds this, it rejects requests. Zero for no limit. (default 400)
```

节点数量在 1000 - 3000 之间时，推荐：

```sh
--max-requests-inflight=1500
--max-mutating-requests-inflight=500
```

节点数量大于 3000 时，推荐：

```sh
--max-requests-inflight=3000
--max-mutating-requests-inflight=1000
```



### kube-scheduler 与 kube-controller-manager 优化

#### 高可用

kube-controller-manager 和 kube-scheduler 是通过 leader election 实现高可用，启用时需要添加以下参数:

```sh
--leader-elect=true
--leader-elect-lease-duration=15s
--leader-elect-renew-deadline=10s
--leader-elect-resource-lock=endpoints
--leader-elect-retry-period=2s
```



#### 控制 QPS

与 kube-apiserver 通信的 qps 限制，推荐为：

```sh
--kube-api-qps=100
```



### 集群 DNS 高可用

设置反亲和，让集群 DNS (kube-dns 或 coredns) 分散在不同节点，避免单点故障:

```yaml
affinity:
 podAntiAffinity:
   requiredDuringSchedulingIgnoredDuringExecution:
   - weight: 100
     labelSelector:
       matchExpressions:
       - key: k8s-app
         operator: In
         values:
         - kube-dns
     topologyKey: kubernetes.io/hostname
```



























































## Kubernetes 最佳安全实践指南



对于大部分 Kubernetes 用户来说，安全是无关紧要的，或者说没那么紧要，就算考虑到了，也只是敷衍一下，草草了事。实际上 Kubernetes 提供了非常多的选项可以大大提高应用的安全性，只要用好了这些选项，就可以将绝大部分的攻击抵挡在门外。为了更容易上手，我将它们总结成了几个最佳实践配置，大家看完了就可以开干了。当然，本文所述的最佳安全实践仅限于 Pod 层面，也就是容器层面，于容器的生命周期相关，至于容器之外的安全配置（比如操作系统啦、k8s 组件啦），以后有机会再唠。



### **1. 为容器配置 Security Context**

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
+   securityContext:
```



### **2. 禁用 allowPrivilegeEscalation**

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
  +   allowPrivilegeEscalation: false
```



### **3. 不要使用 root 用户**

为了防止来自容器内的提权攻击，最好不要使用 root 用户运行容器内的应用。UID 设置大一点，尽量大于 `3000`。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: <name>
spec:
  securityContext:
+   runAsUser: <UID higher than 1000>
+   runAsGroup: <UID higher than 3000>
```



### **4. 限制 CPU 和内存资源**

这个就不用多说了吧，requests 和 limits 都加上。



### **5. 不必挂载 Service Account Token**

ServiceAccount 为 Pod 中运行的进程提供身份标识，怎么标识呢？当然是通过 Token 啦，有了 Token，就防止假冒伪劣进程。如果你的应用不需要这个身份标识，可以不必挂载：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: <name>
spec:
+ automountServiceAccountToken: false
```



### **6. 确保 seccomp 设置正确**

对于 Linux 来说，用户层一切资源相关操作都需要通过系统调用来完成，那么只要对系统调用进行某种操作，用户层的程序就翻不起什么风浪，即使是恶意程序也就只能在自己进程内存空间那一分田地晃悠，进程一终止它也如风消散了。seccomp（secure computing mode）就是一种限制系统调用的安全机制，可以可以指定允许那些系统调用。

对于 Kubernetes 来说，大多数容器运行时都提供一组允许或不允许的默认系统调用。通过使用 `runtime/default` 注释或将 Pod 或容器的安全上下文中的 seccomp 类型设置为 `RuntimeDefault`，可以轻松地在 Kubernetes 中应用默认值。

默认的 seccomp 配置文件应该为大多数工作负载提供足够的权限。如果你有更多的需求，可以自定义配置文件。



### **7. 限制容器的 capabilities**

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
  + runAsNonRoot: true
  + runAsUser: <specific user>
  capabilities:
  drop:
  + -NET_RAW
  + -ALL
```

如果你对 Linux capabilities 这个词一脸懵逼，建议去看看我的脑残入门系列：

- [👉Linux Capabilities 入门教程：概念篇](http://mp.weixin.qq.com/s?__biz=MzU1MzY4NzQ1OA==&mid=2247484610&idx=1&sn=0f75f48b1651f03163bef421280c25f8&chksm=fbee440fcc99cd19c786acd3de00fee7914664171395013bb3fb4e1dfb1a84f618fc1a9b042e&scene=21#wechat_redirect)
- [👉Linux Capabilities 入门教程：基础实战篇](http://mp.weixin.qq.com/s?__biz=MzU1MzY4NzQ1OA==&mid=2247484631&idx=1&sn=6d65cfd09e61f0b56967867a190ca48e&chksm=fbee441acc99cd0cfcd0042021e4d1bac0eb67a6797d7a73234dc746cca6fc3494e38b6c0c4c&scene=21#wechat_redirect)
- [👉Linux Capabilities 入门教程：进阶实战篇](http://mp.weixin.qq.com/s?__biz=MzU1MzY4NzQ1OA==&mid=2247488377&idx=1&sn=430ba2812e467ccfcf12b29f88e0215a&chksm=fbee53b4cc99daa22d9c67bb1f09d93da08d02010752e204b49aca4b35858993d4bab82e8629&scene=21#wechat_redirect)



### **8. 只读**

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
  + readOnlyRootFilesystem: true
```



### **9 总结**

总之，Kubernetes 提供了非常多的选项来增强集群的安全性，没有一个放之四海而皆准的解决方案，所以需要对这些选项非常熟悉，以及了解它们是如何增强应用程序的安全性，才能使集群更加稳定安全。





# Kubernetes 认证



## Kubernetes 准入控制器详解



**Kubernetes 控制平面由几个组件组成。其中一个组件是 kube-apiserver，简单的 API server。**它公开了一个 REST 端点，用户、集群组件以及客户端应用程序可以通过该端点与集群进行通信。总的来说，它会进行以下操作：

1. 从客户端应用程序（如 kubectl）接收标准 HTTP 请求。
2. 验证传入请求并应用授权策略。
3. 在成功的身份验证中，它能根据端点对象（Pod、Deployments、Namespace 等）和 http 动作（Create、Put、Get、Delete 等）执行操作。
4. 对 etcd 数据存储进行更改以保存数据。
5. 操作完成，它就向客户端发送响应。

![image-20210203223654302](D:\学习资料\笔记\k8s\k8s图\image-20210203223654302.png)

现在让我们考虑这样一种情况：**在请求经过身份验证后，但在对 etcd 数据存储进行任何更改之前，我们需要拦截该请求。**例如：

1. 拦截客户端发送的请求。
2. 解析请求并执行操作。
3. 根据请求的结果，决定对 etcd 进行更改还是拒绝对 etcd 进行更改。

Kubernetes 准入控制器就是用于这种情况的插件。在代码层面，准入控制器逻辑与 API server 逻辑解耦，这样用户就可以开发自定义拦截器（custom interceptor），无论何时对象被创建、更新或从 etcd 中删除，都可以调用该拦截器。

有了准入控制器，从任意来源到 API server 的请求流将如下所示：

![image-20210203223901020](D:\学习资料\笔记\k8s\k8s图\image-20210203223901020.png)

根据准入控制器执行的操作类型，它可以分为 3 种类型：

- Mutating（变更）
- Validating（验证）
- Both（两者都有）

**Mutating：**这种控制器可以解析请求，并在请求向下发送之前对请求进行更改（变更请求）。

示例：AlwaysPullImages

**Validating：**这种控制器可以解析请求并根据特定数据进行验证。

示例：NamespaceExists

**Both：**这种控制器可以执行变更和验证两种操作。

示例：CertificateSigning

> 有关这些控制器更多信息，查看官方文档：https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#what-does-each-admission-controller-do

准入控制器过程包括按顺序执行的2个阶段：

1. **Mutating（变更）阶段**（先执行）
2. **Validation （验证）阶段**（变更阶段后执行）

Kubernetes 集群已经在使用准入控制器来执行许多任务。

> Kubernetes 附带的准入控制器列表：https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#what-does-each-admission-controller-do

通过该列表，我们可以发现大多数操作，如 AlwaysPullImages、DefaultStorageClass、PodSecurityPolicy 等，实际上都是由不同的准入控制器执行的。



### **如何启用或禁用准入控制器？**

要启用准入控制器，我们必须在启动 kube-apiserver 时，将以逗号分隔的准入控制器插件名称列表传递给 `--enable-ading-plugins`。对于默认插件，命令如下所示：

![图片](https://mmbiz.qpic.cn/mmbiz_png/1e9ia4YcKpMMwGS47HH9hsGCpEnzJ3wOZ6B3zz9o8PIoVIia5PZD3ZPApWduA8xpChL2ZQcpIhYzNHYzb2uia1lCQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

要禁用准入控制器插件，可以将插件名称列表传递给 `--disable-admission-plugins`。它将覆盖默认启用的插件列表。

![图片](https://mmbiz.qpic.cn/mmbiz_png/1e9ia4YcKpMMwGS47HH9hsGCpEnzJ3wOZy9cwXjZ3g66xibrG72oJGBvgyK8WnzCuYvPzrhHZC76kZbUfVZ0S9lA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



### **默认准入控制器**

- NamespaceLifecycle
- LimitRanger
- ServiceAccount
- TaintNodesByCondition
- Priority
- DefaultTolerationSeconds
- DefaultStorageClass
- StorageObjectInUseProtection
- PersistentVolumeClaimResize
- RuntimeClass
- CertificateApproval
- CertificateSigning
- CertificateSubjectRestriction
- DefaultIngressClass
- MutatingAdmissionWebhook
- ValidatingAdmissionWebhook
- ResourceQuota



### **为什么要使用准入控制器？**

准入控制器能提供额外的安全和治理层，以帮助 Kubernetes 集群的用户使用。

**执行策略**：通过使用自定义准入控制器，我们可以验证请求并检查它是否包含特定的所需信息。例如，我们可以检查 Pod 是否设置了正确的标签。如果没有，那可以一起拒绝该请求。某些情况下，如果请求中缺少一些字段，我们也可以更改这些字段。例如，如果 Pod 没有设置资源限制，我们可以为 Pod 添加特定的资源限制。通过这样的方式，除非明确指定，集群中的所有 Pod 都将根据我们的要求设置资源限制。Limit Range 就是这种实现。

**安全性**：我们可以拒绝不遵循特定规范的请求。例如，没有一个 Pod 请求可以将安全网关设置为以 root 用户身份运行。

**统一工作负载**：通过更改请求并为用户未设置的规范设置默认值，我们可以确保集群上运行的工作负载是统一的，并遵循集群管理员定义的特定标准。这些就是我们开始使用 Kubernetes 准入控制器需要知道的所有理论。



### Example: Writing and Deploying an Admission Controller Webhook

To illustrate how admission controller webhooks can be leveraged to establish custom security policies, let’s consider an example that addresses one of the shortcomings of Kubernetes: a lot of its defaults are optimized for ease of use and reducing friction, sometimes at the expense of security. One of these settings is that containers are by default allowed to run as root (and, without further configuration and no `USER` directive in the Dockerfile, will also do so). Even though containers are isolated from the underlying host to a certain extent, running containers as root does increase the risk profile of your deployment— and should be avoided as one of many [security best practices](https://www.stackrox.com/post/2018/12/6-container-security-best-practices-you-should-be-following/). The [recently exposed runC vulnerability](https://www.stackrox.com/post/2019/02/the-runc-vulnerability-a-deep-dive-on-protecting-yourself/) ([CVE-2019-5736](https://nvd.nist.gov/vuln/detail/CVE-2019-5736)), for example, could be exploited only if the container ran as root.

You can use a custom mutating admission controller webhook to apply more secure defaults: unless explicitly requested, our webhook will ensure that pods run as a non-root user (we assign the user ID 1234 if no explicit assignment has been made). Note that this setup does not prevent you from deploying any workloads in your cluster, including those that legitimately require running as root. It only requires you to explicitly enable this riskier mode of operation in the deployment configuration, while defaulting to non-root mode for all other workloads.

The full code along with deployment instructions can be found in our accompanying [GitHub repository](https://github.com/stackrox/admission-controller-webhook-demo). Here, we will highlight a few of the more subtle aspects about how webhooks work.



### Mutating Webhook Configuration

A mutating admission controller webhook is defined by creating a `MutatingWebhookConfiguration` object in Kubernetes. In our example, we use the following configuration:

```yaml
apiVersion: admissionregistration.k8s.io/v1beta1
kind: MutatingWebhookConfiguration
metadata:
  name: demo-webhook
webhooks:
  - name: webhook-server.webhook-demo.svc
    clientConfig:
      service:
        name: webhook-server
        namespace: webhook-demo
        path: "/mutate"
      caBundle: ${CA_PEM_B64}
    rules:
      - operations: [ "CREATE" ]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
```

This configuration defines a `webhook webhook-server.webhook-demo.svc`, and instructs the Kubernetes API server to consult the service `webhook-server` in `namespace webhook-demo` whenever a pod is created by making a HTTP POST request to the `/mutate` URL. For this configuration to work, several prerequisites have to be met.



### Webhook REST API

The Kubernetes API server makes an HTTPS POST request to the given service and URL path, with a JSON-encoded [`AdmissionReview`](https://github.com/kubernetes/api/blob/master/admission/v1beta1/types.go#L29) (with the `Request` field set) in the request body. The response should in turn be a JSON-encoded `AdmissionReview`, this time with the Response field set.

Our demo repository contains a [function](https://github.com/stackrox/admission-controller-webhook-demo/blob/master/cmd/webhook-server/admission_controller.go#L132) that takes care of the serialization/deserialization boilerplate code and allows you to focus on implementing the logic operating on Kubernetes API objects. In our example, the function implementing the admission controller logic is called `applySecurityDefaults`, and an HTTPS server serving this function under the /mutate URL can be set up as follows:

```yaml
mux := http.NewServeMux()
mux.Handle("/mutate", admitFuncHandler(applySecurityDefaults))
server := &http.Server{
  Addr:    ":8443",
  Handler: mux,
}
log.Fatal(server.ListenAndServeTLS(certPath, keyPath))
```

Note that for the server to run without elevated privileges, we have the HTTP server listen on port 8443. Kubernetes does not allow specifying a port in the webhook configuration; it always assumes the HTTPS port 443. However, since a service object is required anyway, we can easily map port 443 of the service to port 8443 on the container:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webhook-server
  namespace: webhook-demo
spec:
  selector:
    app: webhook-server  # specified by the deployment/pod
  ports:
    - port: 443
      targetPort: webhook-api  # name of port 8443 of the container
```



### Object Modification Logic

In a mutating admission controller webhook, mutations are performed via [JSON patches](https://tools.ietf.org/html/rfc6902). While the JSON patch standard includes a lot of intricacies that go well beyond the scope of this discussion, the Go data structure in our example as well as its usage should give the user a good initial overview of how JSON patches work:

```yaml
type patchOperation struct {
  Op    string      `json:"op"`
  Path  string      `json:"path"`
  Value interface{} `json:"value,omitempty"`
}
```

For setting the field `.spec.securityContext.runAsNonRoot` of a pod to true, we construct the following `patchOperation` object:

```yaml
patches = append(patches, patchOperation{
  Op:    "add",
  Path:  "/spec/securityContext/runAsNonRoot",
  Value: true,
})
```



### TLS Certificates

Since a webhook must be served via HTTPS, we need proper certificates for the server. These certificates can be self-signed (rather: signed by a self-signed CA), but we need Kubernetes to instruct the respective CA certificate when talking to the webhook server. In addition, the common name (CN) of the certificate must match the server name used by the Kubernetes API server, which for internal services is `<service-name>`.`<namespace>.svc`, i.e., `webhook-server.webhook-demo.svc` in our case. Since the generation of self-signed TLS certificates is well documented across the Internet, we simply refer to the respective [shell script](https://github.com/stackrox/admission-controller-webhook-demo/blob/master/deployment/generate-keys.sh) in our example.

The webhook configuration shown previously contains a placeholder `${CA_PEM_B64}`. Before we can create this configuration, we need to replace this portion with the Base64-encoded PEM certificate of the CA. The `openssl base64 -A` command can be used for this purpose.



### Testing the Webhook

After deploying the webhook server and configuring it, which can be done by invoking the ./deploy.sh script from the repository, it is time to test and verify that the webhook indeed does its job. The repository contains [three examples](https://github.com/stackrox/admission-controller-webhook-demo/tree/master/examples):

- A pod that does not specify a security context (`pod-with-defaults`). We expect this pod to be run as non-root with user id 1234.
- A pod that does specify a security context, explicitly allowing it to run as root (`pod-with-override`).
- A pod with a conflicting configuration, specifying it must run as non-root but with a user id of 0 (`pod-with-conflict`). To showcase the rejection of object creation requests, we have augmented our admission controller logic to reject such obvious misconfigurations.

Create one of these pods by running `kubectl create -f examples/<name>.yaml`. In the first two examples, you can verify the user id under which the pod ran by inspecting the logs, for example:

```sh
$ kubectl create -f examples/pod-with-defaults.yaml
$ kubectl logs pod-with-defaults
I am running as user 1234
```

In the third example, the object creation should be rejected with an appropriate error message:

```sh
$ kubectl create -f examples/pod-with-conflict.yaml
Error from server (InternalError): error when creating "examples/pod-with-conflict.yaml": Internal error occurred: admission webhook "webhook-server.webhook-demo.svc" denied the request: runAsNonRoot specified, but runAsUser set to 0 (the root user)
```

Feel free to test this with your own workloads as well. Of course, you can also experiment a little bit further by changing the logic of the webhook and see how the changes affect object creation. More information on how to do experiment with such changes can be found in the [repository’s readme](https://github.com/stackrox/admission-controller-webhook-demo/blob/master/README.md).



### Summary

Kubernetes admission controllers offer significant advantages for security. Digging into two powerful examples, with accompanying available code, will help you get started on leveraging these powerful capabilities.



### References:

- https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/
- https://docs.okd.io/latest/architecture/additional_concepts/dynamic_admission_controllers.html
- https://kubernetes.io/blog/2018/01/extensible-admission-is-beta/
- https://medium.com/ibm-cloud/diving-into-kubernetes-mutatingadmissionwebhook-6ef3c5695f74
- https://github.com/kubernetes/kubernetes/blob/v1.10.0-beta.1/test/images/webhook/main.go
- https://github.com/istio/istio
- https://www.stackrox.com/post/2019/02/the-runc-vulnerability-a-deep-dive-on-protecting-yourself/





## 使用动态准入控制器



### 原理概述

动态准入控制器 Webhook 在访问鉴权过程中可以更改请求对象或完全拒绝该请求，其调用 Webhook 服务的方式使其独立于集群组件，具有非常大的灵活性，可以方便的做很多自定义准入控制，下图为动态准入控制在 API 请求调用链的位置：

![image-20210203223901020](file://D:/%E5%AD%A6%E4%B9%A0%E8%B5%84%E6%96%99/%E7%AC%94%E8%AE%B0/k8s/k8s%E5%9B%BE/image-20210203223901020.png?lastModify=1612186149)



从上图可以看出，动态准入控制过程分为两个阶段：首先执行 Mutating 阶段，可以对到达请求进行修改，然后执行 Validating 阶段来验证到达的请求是否被允许，两个阶段可以单独使用也可以组合使用，本文将在 TKE 中实现一个简单的动态准入控制调用示例。



### 查看验证插件

在 TKE 现有集群版本中（1.10.5 及以上）已经默认开启了 **validating admission webhook**[2] 和 **mutating admission webhook**[3] API，如果是更低版本的集群，可以在 Apiserver Pod 中执行 `kube-apiserver -h | grep enable-admission-plugins` 验证当前集群是否开启，输出插件列表中如果有 `MutatingAdmissionWebhook` 和 `ValidatingAdmissionWebhook` 就说明当前集群开启了动态准入的控制器插件



### 签发证书

为了确保动态准入控制器调用的是可信任的 Webhook 服务端，必须通过 HTTPS 来调用 Webhook 服务（TLS认证）， 所以需要为 Webhook 服务端颁发证书，并且在注册动态准入控制 Webhook 时为 `caBundle` 字段（ `ValidatingWebhookConfiguration` 和 `MutatingAdmissionWebhook` 资源清单中的 `caBundle` 字段）绑定受信任的颁发机构证书（CA）来核验 Webhook 服务端的证书是否可信任， 这里分别介绍两种推荐的颁发证书方法：

> 注意：当`ValidatingWebhookConfiguration` 和 `MutatingAdmissionWebhook` 使用 `clientConfig.service` 配置时（Webhook 服务在集群内），为服务器端颁发的证书域名必须为 `<svc_name>.<svc_namespace>.svc`。



#### **方法一：制作自签证书**

制作自签证书的方法比较独立，不依赖于 K8s 集群，类似于为一个网站做一个自签证书，有很多工具可以制作自签证书，本示例使用 Openssl 制作自签证书，操作步骤如下所示：

1. 生成密钥位数为 2048 的 ca.key：

   ```sh
   openssl genrsa -out ca.key 2048
   ```

2. 依据 ca.key 生成 ca.crt，"webserver.default.svc" 为 Webhook 服务端在集群中的域名，使用 `-days` 参数来设置证书有效时间：

   ```sh
   openssl req -x509 -new -nodes -key ca.key -subj "/CN=webserver.default.svc" -days 10000 -out ca.crt
   ```

3. 生成密钥位数为 2048 的 server.key：

   ```sh
   openssl genrsa -out server.key 2048
   ```

   i. 创建用于生成证书签名请求（CSR）的配置文件 csr.conf 示例如下：

   ```sh
   [ req ]
   default_bits = 2048
   prompt = no
   default_md = sha256
   distinguished_name = dn
   [ dn ]
   C = cn
   ST = shaanxi
   L = xi'an
   O = default
   OU = websever
   CN = webserver.default.svc
   [ v3_ext ]
   authorityKeyIdentifier=keyid,issuer:always
   basicConstraints=CA:FALSE
   keyUsage=keyEncipherment,dataEncipherment
   extendedKeyUsage=serverAuth,clientAuth
   ```

4. 基于配置文件 csr.conf 生成证书签名请求：

   ```sh
   openssl req -new -key server.key -out server.csr -config csr.conf
   ```

5. 使用 ca.key、ca.crt 和 server.csr 颁发生成服务器证书（x509签名）：

   ```sh
   openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key \
     -CAcreateserial -out server.crt -days 10000 \
     -extensions v3_ext -extfile csr.conf
   ```

6. 查看 Webhook server 端证书：

   ```sh
   openssl x509  -noout -text -in ./server.crt
   ```

其中，生成的证书、密钥文件说明如下：

ca.crt 为颁发机构证书，ca.key 为颁发机构证书密钥，用于服务端证书颁发。

server.crt 为 颁发的服务端证书，server.key 为颁发的服务端证书密钥。



#### **方法二：使用 K8s CSR API 签发**

除了使用方案一加密工具制作自签证书，还可以使用 K8s 的证书颁发机构系统来下发证书，执行下面脚本可使用 K8s 集群根证书和根密钥签发一个可信任的证书用户，需要注意的是用户名应该为 Webhook 服务在集群中的域名：

```sh
USERNAME='webserver.default.svc' # 设置需要创建的用户名为 Webhook 服务在集群中的域名
# 使用 Openssl 生成自签证书 key
openssl genrsa -out ${USERNAME}.key 2048
# 使用 Openssl 生成自签证书 CSR 文件, CN 代表用户名，O 代表组名
openssl req -new -key ${USERNAME}.key -out ${USERNAME}.csr -subj "/CN=${USERNAME}/O=${USERNAME}" 
# 创建 Kubernetes 证书签名请求（CSR）
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: ${USERNAME}
spec:
  request: $(cat ${USERNAME}.csr | base64 | tr -d '\n')
  usages:
  - digital signature
  - key encipherment
  - server auth
EOF
# 证书审批允许信任
kubectl certificate approve ${USERNAME}
# 获取自签证书 CRT
kubectl get csr ${USERNAME} -o jsonpath={.status.certificate} > ${USERNAME}.crt
```

其中， `${USERNAME}`.crt 为服务端证书， `${USERNAME}`.key 为 Webhook 服务端证书密钥。



### 操作示例

下面将使用 `ValidatingWebhookConfiguration` 资源在 TKE 中实现一个动态准入 Webhook 调用示例，本示例代码可在 **示例代码**[4] 中获取（为了确保可访问性，示例代码 Fork 自 **原代码库**[5]，作者实现了一个简单的动态准入 Webhook 请求和响应的接口，具体接口格式请参考 **Webhook 请求和响应**[6] 。为了方便，我将使用它作为我们的 Webhook 服务端代码。

1. 准备 `caBundle` 内容

2. - 若颁发证书方法是方案一, 使用 `base64` 编码 ca.crt 生成 `caBundle` 字段内容：

     ```sh
      cat ca.crt | base64 --wrap=0
     ```

   - 若颁发证书方法是方案二，集群的根证书即为 `caBundle` 字段内容，可以通过 TKE 集群控制台【基本信息】-> 【集群APIServer信息】Kubeconfig 内容中的`clusters.cluster[].certificate-authority-data` 字段获取，该字段已经 `base64` 编码过了，无需再做处理。

3. 复制生成的 ca.crt （颁发机构证书），server.crt（HTTPS 证书)）, server.key（HTTPS 密钥） 到项目主目录：

   ![image-20210204100016276](D:\学习资料\笔记\k8s\k8s图\image-20210204100016276.png)

4. 修改项目中的 Dockerfile ，添加三个证书文件到容器工作目录：

![image-20210204100245061](D:\学习资料\笔记\k8s\k8s图\image-20210204100245061.png)

然后使用 docker 命令构建 Webhook 服务端镜像：

```sh
$ docker build -t webserver .
```

4. 部署一个域名为 "weserver.default.svc" 的 Webhook 后端服务，修改适配后的 controller.yaml 如下：

![image-20210204100348711](D:\学习资料\笔记\k8s\k8s图\image-20210204100348711.png)

5. 注册创建类型为 `ValidatingWebhookConfiguration` 的资源，本示例配置的 Webhook 触发规则是当创建 `pods`类型，API 版本 "v1" 时触发调用，`clientConfig` 配置对应上述在集群中创建的的 Webhook 后端服务， `caBundle` 字段内容为证书颁发方法一获取的ca.crt 内容，修改适配项目中的 admission.yaml 文件如下图：

   ![image-20210204100813415](D:\学习资料\笔记\k8s\k8s图\image-20210204100813415.png)

6. 注册好后创建一个 Pod 类型， API 版本为 "v1" 的测试资源如下：

   ![image-20210204100845687](D:\学习资料\笔记\k8s\k8s图\image-20210204100845687.png)

7. 测试代码有打印请求日志， 查看 Webhook 服务端日志可以看到动态准入控制器触发了 webhook 调用，如下图：

   ![image-20210204104230676](D:\学习资料\笔记\k8s\k8s图\image-20210204104230676.png)

8. 此时查看创建的测试pod 是成功创建的，是因为测试 Webhook 服务端代码写死的 `allowed: true`，所以是可以创建成功的，如下图：

   ![image-20210204104320985](D:\学习资料\笔记\k8s\k8s图\image-20210204104320985.png)

9. 为了进一步验证，我们把 "allowed" 改成 "false" ，然后重复上述步骤重新打 Webserver 服务端镜像，并重新部署 controller.yaml 和 admission.yaml 资源，当再次尝试创建 "pods" 资源时请求被动态准入拦截，说明配置的动态准入策略是生效的，如下图所示：
10. ![image-20210204104353955](D:\学习资料\笔记\k8s\k8s图\image-20210204104353955.png)



### 总结

本文主要介绍了动态准入控制器 Webhook 的概念和作用、如何在 TKE 集群中签发动态准入控制器所需的证书，并使用简单示例演示如何配置和使用动态准入 Webhook 功能。



### 参考资料

[1]**Kubernetes 官网**: *https://kubernetes.io/blog/2019/03/21/a-guide-to-kubernetes-admission-controllers/*

[2]**validating admission webhook**: *https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#validatingadmissionwebhook*

[3]**mutating admission webhook**: *https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#mutatingadmissionwebhook*

[4]**示例代码**: *https://github.com/imjokey/hello-dynamic-admission-control*

[5]**原代码库**: *https://github.com/larkintuckerllc/hello-dynamic-admission-control.git*

[6]**Webhook 请求和响应**: *https://kubernetes.io/zh/docs/reference/access-authn-authz/extensible-admission-controllers/#request*

[7]**Kubernetes Dynamic Admission Control by Example**: *https://codeburst.io/kubernetes-dynamic-admission-control-by-example-d8cc2912027c*

[8]**Dynamic Admission Control（官网）**: *https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/*













# kubernetes 原理



## kubectl 创建 Pod 背后到底发生了什么？



想象一下，如果我想将 nginx 部署到 Kubernetes 集群，我可能会在终端中输入类似这样的命令：

```shell
$ kubectl run --image=nginx --replicas=3
```

然后回车。几秒钟后，你就会看到三个 nginx pod 分布在所有的工作节点上。这一切就像变魔术一样，但你并不知道这一切的背后究竟发生了什么事情。

Kubernetes 的神奇之处在于：它可以通过用户友好的 API 来处理跨基础架构的 `deployments`，而背后的复杂性被隐藏在简单的抽象中。但为了充分理解它为我们提供的价值，我们需要理解它的内部原理。

本指南将引导您理解从 client 到 `Kubelet` 的请求的完整生命周期，必要时会通过源代码来说明背后发生了什么。

这是一份可以在线修改的文档，如果你发现有什么可以改进或重写的，欢迎提供帮助！



### 1. kubectl

------

#### 验证和生成器

当敲下回车键以后，`kubectl` 首先会执行一些客户端验证操作，以确保不合法的请求（例如，创建不支持的资源或使用格式错误的镜像名称）将会快速失败，也不会发送给 `kube-apiserver`。通过减少不必要的负载来提高系统性能。

验证通过之后， kubectl 开始将发送给 kube-apiserver 的 HTTP 请求进行封装。`kube-apiserver` 与 etcd 进行通信，所有尝试访问或更改 Kubernetes 系统状态的请求都会通过 kube-apiserver 进行，kubectl 也不例外。kubectl 使用生成器（[generators](https://kubernetes.io/docs/user-guide/kubectl-conventions/#generators)）来构造 HTTP 请求。生成器是一个用来处理序列化的抽象概念。

通过 `kubectl run` 不仅可以运行 `deployment`，还可以通过指定参数 `--generator` 来部署其他多种资源类型。如果没有指定 `--generator` 参数的值，kubectl 将会自动判断资源的类型。

例如，带有参数 `--restart-policy=Always` 的资源将被部署为 Deployment，而带有参数 `--restart-policy=Never` 的资源将被部署为 Pod。同时 kubectl 也会检查是否需要触发其他操作，例如记录命令（用来进行回滚或审计）。

在 kubectl 判断出要创建一个 Deployment 后，它将使用 `DeploymentV1Beta1` 生成器从我们提供的参数中生成一个[运行时对象](https://github.com/kubernetes/kubernetes/blob/7650665059e65b4b22375d1e28da5306536a12fb/pkg/kubectl/run.go#L59)。



#### API 版本协商与 API 组

为了更容易地消除字段或者重新组织资源结构，Kubernetes 支持多个 API 版本，每个版本都在不同的 API 路径下，例如 `/api/v1` 或者 `/apis/extensions/v1beta1`。不同的 API 版本表明不同的稳定性和支持级别，更详细的描述可以参考 [Kubernetes API 概述](https://k8smeetup.github.io/docs/reference/api-overview/)。

API 组旨在对类似资源进行分类，以便使得 Kubernetes API 更容易扩展。API 的组名在 REST 路径或者序列化对象的 `apiVersion` 字段中指定。例如，Deployment 的 API 组名是 `apps`，最新的 API 版本是 `v1beta2`，这就是为什么你要在 Deployment manifests 顶部输入 `apiVersion: apps/v1beta2`。

kubectl 在生成运行时对象后，开始为它[找到适当的 API 组和 API 版本](https://github.com/kubernetes/kubernetes/blob/7650665059e65b4b22375d1e28da5306536a12fb/pkg/kubectl/cmd/run.go#L580-L597)，然后[组装成一个版本化客户端](https://github.com/kubernetes/kubernetes/blob/7650665059e65b4b22375d1e28da5306536a12fb/pkg/kubectl/cmd/run.go#L598)，该客户端知道资源的各种 REST 语义。该阶段被称为版本协商，kubectl 会扫描 `remote API` 上的 `/apis` 路径来检索所有可能的 API 组。由于 kube-apiserver 在 `/apis` 路径上公开了 OpenAPI 格式的规范文档， 因此客户端很容易找到合适的 API。

为了提高性能，kubectl [将 OpenAPI 规范缓存](https://github.com/kubernetes/kubernetes/blob/7650665059e65b4b22375d1e28da5306536a12fb/pkg/kubectl/cmd/util/factory_client_access.go#L117)到了 `~/.kube/cache` 目录。如果你想了解 API 发现的过程，请尝试删除该目录并在运行 kubectl 命令时将 `-v` 参数的值设为最大值，然后你将会看到所有试图找到这些 API 版本的HTTP 请求。参考 [kubectl 备忘单](https://k8smeetup.github.io/docs/reference/kubectl/cheatsheet/)。

最后一步才是真正地发送 HTTP 请求。一旦请求发送之后获得成功的响应，kubectl 将会根据所需的输出格式打印 success message。



#### 客户端身份认证

在发送 HTTP 请求之前还要进行客户端认证，这是之前没有提到的，现在可以来看一下。

为了能够成功发送请求，kubectl 需要先进行身份认证。用户凭证保存在 `kubeconfig` 文件中，kubectl 通过以下顺序来找到 kubeconfig 文件：

- 如果提供了 `--kubeconfig` 参数， kubectl 就使用 –kubeconfig 参数提供的 kubeconfig 文件。
- 如果没有提供 –kubeconfig 参数，但设置了环境变量 `$KUBECONFIG`，则使用该环境变量提供的 kubeconfig 文件。
- 如果 –kubeconfig 参数和环境变量 `$KUBECONFIG` 都没有提供，kubectl 就使用默认的 kubeconfig 文件 `$HOME/.kube/config`。

解析完 kubeconfig 文件后，kubectl 会确定当前要使用的上下文、当前指向的群集以及与当前用户关联的任何认证信息。如果用户提供了额外的参数（例如 –username），则优先使用这些参数覆盖 kubeconfig 中指定的值。一旦拿到这些信息之后， kubectl 就会把这些信息填充到将要发送的 HTTP 请求头中：

- x509 证书使用 [tls.TLSConfig](https://github.com/kubernetes/client-go/blob/82aa063804cf055e16e8911250f888bc216e8b61/rest/transport.go#L80-L89) 发送（包括 CA 证书）。
- `bearer tokens` 在 HTTP 请求头 `Authorization` 中[发送](https://github.com/kubernetes/client-go/blob/c6f8cf2c47d21d55fa0df928291b2580544886c8/transport/round_trippers.go#L314)。
- 用户名和密码通过 HTTP 基本认证[发送](https://github.com/kubernetes/client-go/blob/c6f8cf2c47d21d55fa0df928291b2580544886c8/transport/round_trippers.go#L223)。
- `OpenID` 认证过程是由用户事先手动处理的，产生一个像 bearer token 一样被发送的 token。



### 2. kube-apiserver

------

#### [认证](https://k8smeetup.github.io/docs/admin/authentication/)

现在我们的请求已经发送成功了，接下来将会发生什么？这时候就该 `kube-apiserver` 闪亮登场了！kube-apiserver 是客户端和系统组件用来保存和检索集群状态的主要接口。为了执行相应的功能，kube-apiserver 需要能够验证请求者是合法的，这个过程被称为认证。

那么 apiserver 如何对请求进行认证呢？当 kube-apiserver 第一次启动时，它会查看用户提供的所有 [CLI 参数](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/)，并组合成一个合适的令牌列表。

**举个例子 :** 如果提供了 `--client-ca-file` 参数，则会将 x509 客户端证书认证添加到令牌列表中；如果提供了 `--token-auth-file` 参数，则会将 breaer token 添加到令牌列表中。

每次收到请求时，apiserver 都会[通过令牌链进行认证，直到某一个认证成功为止](https://github.com/kubernetes/apiserver/blob/51bebaffa01be9dc28195140da276c2f39a10cd4/pkg/authentication/request/union/union.go#L54)：

- [x509 处理程序](https://github.com/kubernetes/apiserver/blob/51bebaffa01be9dc28195140da276c2f39a10cd4/pkg/authentication/request/x509/x509.go#L60)将验证 HTTP 请求是否是由 CA 根证书签名的 TLS 密钥进行编码的。
- [bearer token 处理程序](https://github.com/kubernetes/apiserver/blob/51bebaffa01be9dc28195140da276c2f39a10cd4/pkg/authentication/request/bearertoken/bearertoken.go#L38)将验证 `--token-auth-file` 参数提供的 token 文件是否存在。
- [基本认证处理程序](https://github.com/kubernetes/apiserver/blob/51bebaffa01be9dc28195140da276c2f39a10cd4/plugin/pkg/authenticator/request/basicauth/basicauth.go#L37)确保 HTTP 请求的基本认证凭证与本地的状态匹配。

如果[认证失败](https://github.com/kubernetes/apiserver/blob/20bfbdf738a0643fe77ffd527b88034dcde1b8e3/pkg/authentication/request/union/union.go#L71)，则请求失败并返回相应的错误信息；如果验证成功，则将请求中的 `Authorization` 请求头删除，并[将用户信息添加到](https://github.com/kubernetes/apiserver/blob/e30df5e70ef9127ea69d607207c894251025e55b/pkg/endpoints/filters/authentication.go#L71-L75)其上下文中。这给后续的授权和准入控制器提供了访问之前建立的用户身份的能力。



#### [授权](https://k8smeetup.github.io/docs/admin/authorization/)

OK，现在请求已经发送，并且 kube-apiserver 已经成功验证我们是谁，终于解脱了！

然而事情并没有结束，虽然我们已经证明了**我们是合法的**，但我们有权执行此操作吗？毕竟身份和权限不是一回事。为了进行后续的操作，kube-apiserver 还要对用户进行授权。

kube-apiserver 处理授权的方式与处理身份验证的方式相似：通过 kube-apiserver 的启动参数 `--authorization_mode` 参数设置。它将组合一系列授权者，这些授权者将针对每个传入的请求进行授权。如果所有授权者都拒绝该请求，则该请求会被禁止响应并且[不会再继续响应](https://github.com/kubernetes/apiserver/blob/e30df5e70ef9127ea69d607207c894251025e55b/pkg/endpoints/filters/authorization.go#L60)。如果某个授权者批准了该请求，则请求继续。

kube-apiserver 目前支持以下几种授权方法：

- [webhook](https://k8smeetup.github.io/docs/admin/authorization/webhook/): 它与集群外的 HTTP(S) 服务交互。
- [ABAC](https://k8smeetup.github.io/docs/admin/authorization/abac/): 它执行静态文件中定义的策略。
- [RBAC](https://k8smeetup.github.io/docs/admin/authorization/rbac/): 它使用 `rbac.authorization.k8s.io` API Group实现授权决策，允许管理员通过 Kubernetes API 动态配置策略。
- [Node](https://k8smeetup.github.io/docs/admin/authorization/node/): 它确保 kubelet 只能访问自己节点上的资源。



#### [准入控制](https://k8smeetup.github.io/docs/admin/admission-controllers/)

突破了之前所说的认证和授权两道关口之后，客户端的调用请求就能够得到 API Server 的真正响应了吗？答案是：不能！

从 kube-apiserver 的角度来看，它已经验证了我们的身份并且赋予了相应的权限允许我们继续，但对于 Kubernetes 而言，其他组件对于应不应该允许发生的事情还是很有意见的。所以这个请求还需要通过 `Admission Controller` 所控制的一个 `准入控制链` 的层层考验，官方标准的 “关卡” 有近十个之多，而且还能自定义扩展！

虽然授权的重点是回答用户是否有权限，但准入控制器会拦截请求以确保它符合集群的更广泛的期望和规则。它们是资源对象保存到 `etcd` 之前的最后一个堡垒，封装了一系列额外的检查以确保操作不会产生意外或负面结果。不同于授权和认证只关心请求的用户和操作，准入控制还处理请求的内容，并且仅对创建、更新、删除或连接（如代理）等有效，而对读操作无效。

准入控制器的工作方式与授权者和验证者的工作方式类似，但有一点区别：与验证链和授权链不同，如果某个准入控制器检查不通过，则整个链会中断，整个请求将立即被拒绝并且返回一个错误给终端用户。

准入控制器设计的重点在于提高可扩展性，某个控制器都作为一个插件存储在 `plugin/pkg/admission` 目录中，并且与某一个接口相匹配，最后被编译到 kube-apiserver 二进制文件中。

大部分准入控制器都比较容易理解，接下来着重介绍 `SecurityContextDeny`、`ResourceQuota` 及 `LimitRanger` 这三个准入控制器。

- **SecurityContextDeny** 该插件将禁止创建设置了 Security Context 的 Pod。
- **ResourceQuota** 不仅能限制某个 Namespace 中创建资源的数量，而且能限制某个 Namespace 中被 Pod 所请求的资源总量。该准入控制器和资源对象 `ResourceQuota` 一起实现了资源配额管理。
- **LimitRanger** 作用类似于上面的 ResourceQuota 控制器，针对 Namespace 资源的每个个体（Pod 与 Container 等）的资源配额。该插件和资源对象 `LimitRange` 一起实现资源配额管理。



### 3. etcd

------

到现在为止，Kubernetes 已经对该客户端的调用请求进行了全面彻底地审查，并且已经验证通过，运行它进入下一个环节。下一步 kube-apiserver 将对 HTTP 请求进行反序列化，然后利用得到的结果构建运行时对象（有点像 kubectl 生成器的逆过程），并保存到 `etcd` 中。下面我们将这个过程分解一下。

当收到请求时，kube-apiserver 是如何知道它该怎么做的呢？事实上，在客户端发送调用请求之前就已经产生了一系列非常复杂的流程。我们就从 kube-apiserver 二进制文件首次运行开始分析吧：

1. 当运行 kube-apiserver 二进制文件时，它会[创建一个允许 apiserver 聚合的服务链](https://github.com/kubernetes/kubernetes/blob/7650665059e65b4b22375d1e28da5306536a12fb/cmd/kube-apiserver/app/server.go#L119)。这是一种对 `Kubernetes API` 进行扩展的方式。
2. 同时会创建一个 `generic apiserver` 作为默认的 apiserver。
3. 然后利用[生成的 OpenAPI 规范](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/server/config.go#L149)来填充 apiserver 的配置。
4. 然后 kube-apiserver 遍历数据结构中指定的所有 API 组，并将每一个 API 组作为通用的存储抽象保存到 etcd 中。当你访问或变更资源状态时，kube-apiserver 就会调用这些 API 组。
5. 每个 API 组都会遍历它的所有组版本，并且将每个 HTTP 路由[映射到 REST 路径中](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/groupversion.go#L92)。
6. 当请求的 METHOD 是 `POST` 时，kube-apiserver 就会将请求转交给 [资源创建处理器](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L37)。

现在 kube-apiserver 已经知道了所有的路由及其对应的 REST 路径，以便在请求匹配时知道调用哪些处理器和键值存储。多么机智的设计！现在假设客户端的 HTTP 请求已经被 kube-apiserver 收到了：

1. 如果处理链可以将请求与已经注册的路由进行匹配，就会将该请求交给注册到该路由的[专用处理器](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/server/handler.go#L143)来处理；如果没有任何一个路由可以匹配该请求，就会将请求转交给[基于路径的处理器](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/server/mux/pathrecorder.go#L248)（比如当调用 `/apis` 时）；如果没有任何一个基于路径的处理器注册到该路径，请求就会被转交给 [not found 处理器](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/server/mux/pathrecorder.go#L254)，最后返回 `404`。
2. 幸运的是，我们有一个名为 `createHandler` 的注册路由！它有什么作用呢？首先它会解码 HTTP 请求并进行基本的验证，例如确保请求提供的 json 与 API 资源的版本相匹配。
3. 接下来进入[审计和准入控制](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L93-L104)阶段。
4. 然后资源将会通过 [storage provider](https://github.com/kubernetes/apiserver/blob/19667a1afc13cc13930c40a20f2c12bbdcaaa246/pkg/registry/generic/registry/store.go#L327) 保存[到 etcd](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L111) 中。默认情况下保存到 etcd 中的键的格式为 `<namespace>/<name>`，你也可以自定义。
5. 资源创建过程中出现的任何错误都会被捕获，最后 `storage provider` 会执行 `get` 调用来确认该资源是否被成功创建。如果需要额外的清理工作，就会调用后期创建的处理器和装饰器。
6. 最后构造 HTTP 响应并返回给客户端。

原来 apiserver 做了这么多的工作，以前竟然没有发现呢！到目前为止，我们创建的 `Deployment` 资源已经保存到了 etcd 中，但 apiserver 仍然看不到它。



### 4. 初始化

------

在一个资源对象被持久化到数据存储之后，apiserver 还无法完全看到或调度它，在此之前还要执行一系列[Initializers](https://v1-13.docs.kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#initializers)。Initializers是一种与资源类型相关联的控制器，它会在资源对外可用之前执行某些逻辑。如果某个资源类型没有Initializers，就会跳过此初始化步骤立即使资源对外可见。

正如[大佬的博客](https://ahmet.im/blog/initializers/)指出的那样，Initializers是一个强大的功能，因为它允许我们执行通用引导操作。例如：

- 将代理边车容器注入到暴露 80 端口的 Pod 中，或者加上特定的 `annotation`。
- 将保存着测试证书的 `volume` 注入到特定命名空间的所有 Pod 中。
- 如果 `Secret` 中的密码小于 20 个字符，就组织其创建。

`initializerConfiguration` 资源对象允许你声明某些资源类型应该运行哪些Initializers。如果你想每创建一个 Pod 时就运行一个自定义Initializers，你可以这样做：

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

通过该配置创建资源对象 `InitializerConfiguration` 之后，就会在每个 Pod 的 `metadata.initializers.pending` 字段中添加 `custom-pod-initializer` 字段。该初始化控制器会定期扫描新的 Pod，一旦在 Pod 的 `pending` 字段中检测到自己的名称，就会执行其逻辑，执行完逻辑之后就会将 `pending` 字段下的自己的名称删除。

只有在 `pending` 字段下的列表中的第一个Initializers可以对资源进行操作，当所有的Initializers执行完成，并且 `pending` 字段为空时，该对象就会被认为初始化成功。

**你可能会注意到一个问题：如果 kube-apiserver 不能显示这些资源，那么用户级控制器是如何处理资源的呢？**

为了解决这个问题，kube-apiserver 暴露了一个 `?includeUninitialized` 查询参数，它会返回所有的资源对象（包括未初始化的）。



### 5. 控制循环

------

#### Deployments controller

到了这个阶段，我们的 Deployment 记录已经保存在 etcd 中，并且所有的初始化逻辑都执行完成，接下来的阶段将会涉及到该资源所依赖的拓扑结构。在 Kubernetes 中，Deployment 实际上只是一系列 `Replicaset` 的集合，而 Replicaset 是一系列 `Pod` 的集合。那么 Kubernetes 是如何从一个 HTTP 请求按照层级结构依次创建这些资源的呢？其实这些工作都是由 Kubernetes 内置的 `Controller`(控制器) 来完成的。

Kubernetes 在整个系统中使用了大量的 Controller，Controller 是一个用于将系统状态从“当前状态”修正到“期望状态”的异步脚本。所有 Controller 都通过 `kube-controller-manager` 组件并行运行，每种 Controller 都负责一种具体的控制流程。首先介绍一下 `Deployment Controller`：

将 Deployment 记录存储到 etcd 并初始化后，就可以通过 kube-apiserver 使其可见，然后 `Deployment Controller` 就会检测到它（它的工作就是负责监听 Deployment 记录的更改）。在我们的例子中，控制器通过一个 `Informer` [注册一个创建事件的特定回调函数](https://github.com/kubernetes/kubernetes/blob/7650665059e65b4b22375d1e28da5306536a12fb/pkg/controller/deployment/deployment_controller.go#L122)（更多信息参加下文）。

当 Deployment 第一次对外可见时，该 Controller 就会[将该资源对象添加到内部工作队列](https://github.com/kubernetes/kubernetes/blob/7650665059e65b4b22375d1e28da5306536a12fb/pkg/controller/deployment/deployment_controller.go#L170)，然后开始处理这个资源对象：

> 通过使用标签选择器查询 kube-apiserver 来[检查](https://github.com/kubernetes/kubernetes/blob/7650665059e65b4b22375d1e28da5306536a12fb/pkg/controller/deployment/deployment_controller.go#L633)该 Deployment 是否有与其关联的 `ReplicaSet` 或 `Pod` 记录。

有趣的是，这个同步过程是状态不可知的，它核对新记录与核对已经存在的记录采用的是相同的方式。

在意识到没有与其关联的 `ReplicaSet` 或 `Pod` 记录后，Deployment Controller 就会开始执行[弹性伸缩流程](https://github.com/kubernetes/kubernetes/blob/7650665059e65b4b22375d1e28da5306536a12fb/pkg/controller/deployment/sync.go#L385)：

> 创建 ReplicaSet 资源，为其分配一个标签选择器并将其版本号设置为 1。

ReplicaSet 的 `PodSpec` 字段从 Deployment 的 manifest 以及其他相关元数据中复制而来。有时 Deployment 记录在此之后也需要更新（例如，如果设置了 `process deadline`）。

当完成以上步骤之后，该 Deployment 的 `status` 就会被更新，然后重新进入与之前相同的循环，等待 Deployment 与期望的状态相匹配。由于 Deployment Controller 只关心 ReplicaSet，因此需要通过 `ReplicaSet Controller` 来继续协调。



#### ReplicaSets controller

在前面的步骤中，Deployment Controller 创建了第一个 ReplicaSet，但仍然还是没有 Pod，这时候就该 `ReplicaSet Controller` 登场了！ReplicaSet Controller 的工作是监视 ReplicaSets 及其相关资源（Pod）的生命周期。和大多数其他 Controller 一样，它通过触发某些事件的处理器来实现此目的。

当创建 ReplicaSet 时（由 Deployment Controller 创建），RS Controller [检查新 ReplicaSet 的状态](https://github.com/kubernetes/kubernetes/blob/7650665059e65b4b22375d1e28da5306536a12fb/pkg/controller/replicaset/replica_set.go#L583)，并检查当前状态与期望状态之间存在的偏差，然后通过[调整 Pod 的副本数](https://github.com/kubernetes/kubernetes/blob/7650665059e65b4b22375d1e28da5306536a12fb/pkg/controller/replicaset/replica_set.go#L460)来达到期望的状态。

Pod 的创建也是批量进行的，从 `SlowStartInitialBatchSize` 开始，然后在每次成功的迭代中以一种 `slow start` 操作加倍。这样做的目的是在大量 Pod 启动失败时（例如，由于资源配额），可以减轻 kube-apiserver 被大量不必要的 HTTP 请求吞没的风险。如果创建失败，最好能够优雅地失败，并且对其他的系统组件造成的影响最小！

Kubernetes 通过 `Owner References`（在子级资源的某个字段中引用其父级资源的 ID） 来构造严格的资源对象层级结构。这确保了一旦 Controller 管理的资源被删除（级联删除），子资源就会被垃圾收集器删除，同时还为父级资源提供了一种有效的方式来避免他们竞争同一个子级资源（想象两对父母都认为他们拥有同一个孩子的场景）。

Owner References 的另一个好处是：它是有状态的。如果有任何 Controller 重启了，那么由于资源对象的拓扑关系与 Controller 无关，该操作不会影响到系统的稳定运行。这种对资源隔离的重视也体现在 Controller 本身的设计中：Controller 不能对自己没有明确拥有的资源进行操作，它们应该选择对资源的所有权，互不干涉，互不共享。

有时系统中也会出现孤儿（orphaned）资源，通常由以下两种途径产生：

- 父级资源被删除，但子级资源没有被删除
- 垃圾收集策略禁止删除子级资源

当发生这种情况时，Controller 将会确保孤儿资源拥有新的 `Owner`。多个父级资源可以相互竞争同一个孤儿资源，但只有一个会成功（其他父级资源会收到验证错误）。



#### Informers

你可能已经注意到，某些 Controller（例如 RBAC 授权器或 Deployment Controller）需要先检索集群状态然后才能正常运行。拿 RBAC 授权器举例，当请求进入时，授权器会将用户的初始状态缓存下来，然后用它来检索与 etcd 中的用户关联的所有 角色（`Role`）和 角色绑定（`RoleBinding`）。那么问题来了，Controller 是如何访问和修改这些资源对象的呢？事实上 Kubernetes 是通过 `Informer` 机制来解决这个问题的。

Infomer 是一种模式，它允许 Controller 查找缓存在本地内存中的数据(这份数据由 Informer 自己维护)并列出它们感兴趣的资源。

虽然 Informer 的设计很抽象，但它在内部实现了大量的对细节的处理逻辑（例如缓存），缓存很重要，因为它不但可以减少对 Kubenetes API 的直接调用，同时也能减少 Server 和 Controller 的大量重复性工作。通过使用 Informer，不同的 Controller 之间以线程安全（Thread safety）的方式进行交互，而不必担心多个线程访问相同的资源时会产生冲突。

有关 Informer 的更多详细解析，请参考这篇文章：[Kubernetes: Controllers, Informers, Reflectors and Stores](https://borismattijssen.github.io/articles/kubernetes-informers-controllers-reflectors-stores)



#### Scheduler

当所有的 Controller 正常运行后，etcd 中就会保存一个 Deployment、一个 ReplicaSet 和 三个 Pod 资源记录，并且可以通过 kube-apiserver 查看。然而，这些 Pod 资源现在还处于 `Pending` 状态，因为它们还没有被调度到集群中合适的 Node 上运行。这个问题最终要靠调度器（Scheduler）来解决。

`Scheduler` 作为一个独立的组件运行在集群控制平面上，工作方式与其他 Controller 相同：监听实际并将系统状态调整到期望的状态。具体来说，Scheduler 的作用是将待调度的 Pod 按照特定的算法和调度策略绑定（Binding）到集群中某个合适的 Node 上，并将绑定信息写入 etcd 中（它会过滤其 PodSpec 中 `NodeName` 字段为空的 Pod），默认的调度算法的工作方式如下：

1. 当 Scheduler 启动时，会[注册一个默认的预选策略链](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/plugin/pkg/scheduler/algorithmprovider/defaults/defaults.go#L65-L81)，这些 `预选策略` 会对备选节点进行评估，判断备选节点是否[满足备选 Pod 的需求](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/plugin/pkg/scheduler/core/generic_scheduler.go#L117)。例如，如果 PodSpec 字段限制了 CPU 和内存资源，那么当备选节点的资源容量不满足备选 Pod 的需求时，备选 Pod 就不会被调度到该节点上（**资源容量=备选节点资源总量-节点中已存在 Pod 的所有容器的需求资源（CPU 和内存）的总和**）
2. 一旦筛选出符合要求的候选节点，就会采用 `优选策略` 计算出每个候选节点的积分，然后对这些候选节点进行排序，积分最高者胜出。例如，为了在整个系统中分摊工作负载，这些优选策略会从备选节点列表中选出资源消耗最小的节点。每个节点通过优选策略时都会算出一个得分，计算各项得分，最终选出分值大的节点作为优选的结果。

一旦找到了合适的节点，Scheduler 就会创建一个 `Binding` 对象，该对象的 `Name` 和 `Uid` 与 Pod 相匹配，并且其 `ObjectReference` 字段包含所选节点的名称，然后通过 `POST` 请求[发送给 apiserver](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/plugin/pkg/scheduler/factory/factory.go#L1095)。

当 kube-apiserver 接收到此 Binding 对象时，注册吧会将该对象**反序列化**并更新 Pod 资源中的以下字段：

- 将 `NodeName` 的值设置为 ObjectReference 中的 NodeName。
- 添加相关的注释。
- 将 `PodScheduled` 的 `status` 值设置为 True。可以通过 kubectl 来查看：

```bash
$ kubectl get <PODNAME> -o go-template='{{range .status.conditions}}{{if eq .type "PodScheduled"}}{{.status}}{{end}}{{end}}'
```

一旦 Scheduler 将 Pod 调度到某个节点上，该节点的 `Kubelet` 就会接管该 Pod 并开始部署。

预选策略和优选策略都可以通过 `–policy-config-file` 参数来扩展，如果默认的调度器不满足要求，还可以部署自定义的调度器。如果 `podSpec.schedulerName` 的值设置为其他的调度器，则 Kubernetes 会将该 Pod 的调度转交给那个调度器。



### 6. Kubelet

------

#### Pod 同步

现在，所有的 Controller 都完成了工作，我们来总结一下：

- HTTP 请求通过了认证、授权和准入控制阶段。
- 一个 Deployment、ReplicaSet 和三个 Pod 资源被持久化到 etcd 存储中。
- 然后运行了一系列Initializers。
- 最后每个 Pod 都被调度到合适的节点。

然而到目前为止，所有的状态变化仅仅只是针对保存在 etcd 中的资源记录，接下来的步骤涉及到运行在工作节点之间的 Pod 的分布状况，这是分布式系统（比如 Kubernetes）的关键因素。这些任务都是由 `Kubelet` 组件完成的，让我们开始吧！

在 Kubernetes 集群中，每个 Node 节点上都会启动一个 Kubelet 服务进程，该进程用于处理 Scheduler 下发到本节点的任务，管理 Pod 的生命周期，包括挂载卷、容器日志记录、垃圾回收以及其他与 Pod 相关的事件。

如果换一种思维模式，你可以把 Kubelet 当成一种特殊的 Controller，它每隔 20 秒（可以自定义）向 kube-apiserver 通过 `NodeName` 获取自身 Node 上所要运行的 Pod 清单。一旦获取到了这个清单，它就会通过与自己的内部缓存进行比较来检测新增加的 Pod，如果有差异，就开始同步 Pod 列表。我们来详细分析一下同步过程：

1. 如果 Pod 正在创建， Kubelet 就会[记录一些在 `Prometheus` 中用于追踪 Pod 启动延时的指标](https://github.com/kubernetes/kubernetes/blob/fc8bfe2d8929e11a898c4557f9323c482b5e8842/pkg/kubelet/kubelet.go#L1519)。
2. 然后生成一个 `PodStatus` 对象，它表示 Pod 当前阶段的状态。Pod 的状态(`Phase`) 是 Pod 在其生命周期中的最精简的概要，包括 `Pending`，`Running`，`Succeeded`，`Failed` 和 `Unkown` 这几个值。状态的产生过程非常复杂，所以很有必要深入了解一下背后的原理：

- 首先串行执行一系列 Pod 同步处理器（`PodSyncHandlers`），每个处理器检查 Pod 是否应该运行在该节点上。当所有的处理器都认为该 Pod 不应该运行在该节点上，则 Pod 的 `Phase` 值就会变成 `PodFailed`，并且将该 Pod 从该节点上驱逐出去。例如当你创建一个 `Job` 时，如果 Pod 失败重试的时间超过了 `spec.activeDeadlineSeconds` 设置的值，就会将 Pod 从该节点驱逐出去。

- 接下来，Pod 的 Phase 值由 `init 容器` 和应用容器的状态共同来决定。因为目前容器还没有启动，容器被视为[处于等待阶段](https://github.com/kubernetes/kubernetes/blob/fc8bfe2d8929e11a898c4557f9323c482b5e8842/pkg/kubelet/kubelet_pods.go#L1244)，如果 Pod 中至少有一个容器处于等待阶段，则其 `Phase` 值为 [Pending](https://github.com/kubernetes/kubernetes/blob/fc8bfe2d8929e11a898c4557f9323c482b5e8842/pkg/kubelet/kubelet_pods.go#L1258-L1261)。

- 最后，Pod 的 `Condition` 字段由 Pod 内所有容器的状态决定。现在我们的容器还没有被容器运行时创建，所以 [`PodReady` 的状态被设置为 `False`](https://github.com/kubernetes/kubernetes/blob/fc8bfe2d8929e11a898c4557f9323c482b5e8842/pkg/kubelet/status/generate.go#L70-L81)。可以通过 kubectl 查看：

  ```bash
  $ kubectl get <PODNAME> -o go-template='{{range .status.conditions}}{{if eq .type "Ready"}}{{.status}}{{end}}{{end}}'
  ```


1. 生成 PodStatus 之后（Pod 中的 `status` 字段），Kubelet 就会将它发送到 Pod 的状态管理器，该管理器的任务是通过 apiserver 异步更新 etcd 中的记录。
2. 接下来运行一系列**准入处理器**来确保该 Pod 是否具有相应的权限（包括强制执行 [`AppArmor` 配置文件和 `NO_NEW_PRIVS`](https://github.com/kubernetes/kubernetes/blob/fc8bfe2d8929e11a898c4557f9323c482b5e8842/pkg/kubelet/kubelet.go#L883-L884)），被准入控制器拒绝的 Pod 将一直保持 `Pending` 状态。
3. 如果 Kubelet 启动时指定了 `cgroups-per-qos` 参数，Kubelet 就会为该 Pod 创建 `cgroup` 并进行相应的资源限制。这是为了更方便地对 Pod 进行服务质量（QoS）管理。
4. 然后为 Pod 创建相应的目录，包括 Pod 的目录（`/var/run/kubelet/pods/<podID>`），该 Pod 的卷目录（`<podDir>/volumes`）和该 Pod 的插件目录（`<podDir>/plugins`）。
5. **卷管理器**会[挂载 `Spec.Volumes` 中定义的相关数据卷，然后等待是否挂载成功](https://github.com/kubernetes/kubernetes/blob/2723e06a251a4ec3ef241397217e73fa782b0b98/pkg/kubelet/volumemanager/volume_manager.go#L330)。根据挂载卷类型的不同，某些 Pod 可能需要等待更长的时间（比如 NFS 卷）。
6. [从 apiserver 中检索](https://github.com/kubernetes/kubernetes/blob/dd9981d038012c120525c9e6df98b3beb3ef19e1/pkg/kubelet/kubelet_pods.go#L788) `Spec.ImagePullSecrets` 中定义的所有 `Secret`，然后将其注入到容器中。
7. 最后通过容器运行时接口（`Container Runtime Interface（CRI）`）开始启动容器（下面会详细描述）。



#### CRI 与 pause 容器

到了这个阶段，大量的初始化工作都已经完成，容器已经准备好开始启动了，而容器是由**容器运行时**（例如 `Docker` 和 `Rkt`）启动的。

为了更容易扩展，Kubelet 从 1.5.0 开始通过**容器运行时接口**与容器运行时（Container Runtime）交互。简而言之，CRI 提供了 Kubelet 和特定的运行时之间的抽象接口，它们之间通过[协议缓冲区](https://github.com/google/protobuf)（它像一个更快的 JSON）和 [gRPC API](https://grpc.io/)（一种非常适合执行 Kubernetes 操作的 API）。这是一个非常酷的想法，通过使用 Kubelet 和运行时之间定义的契约关系，容器如何编排的具体实现细节已经变得无关紧要。由于不需要修改 Kubernetes 的核心代码，开发者可以以最小的开销添加新的运行时。

第一次启动 Pod 时，Kubelet 会通过 `Remote Procedure Command`(RPC) 协议调用 [RunPodSandbox](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/pkg/kubelet/kuberuntime/kuberuntime_sandbox.go#L51)。`sandbox` 用于描述一组容器，例如在 Kubernetes 中它表示的是 Pod。`sandbox` 是一个很宽泛的概念，所以对于其他没有使用容器的运行时仍然是有意义的（比如在一个基于 `hypervisor` 的运行时中，sandbox 可能指的就是虚拟机）。

我们的例子中使用的容器运行时是 Docker，创建 sandbox 时首先创建的是 `pause` 容器。pause 容器作为同一个 Pod 中所有其他容器的基础容器，它为 Pod 中的每个业务容器提供了大量的 Pod 级别资源，这些资源都是 Linux 命名空间（包括网络命名空间，IPC 命名空间和 PID 命名空间）。

pause 容器提供了一种方法来管理所有这些命名空间并允许业务容器共享它们，在同一个网络命名空间中的好处是：同一个 Pod 中的容器可以使用 `localhost` 来相互通信。pause 容器的第二个功能与 PID 命名空间的工作方式相关，在 PID 命名空间中，进程之间形成一个树状结构，一旦某个子进程由于父进程的错误而变成了“孤儿进程”，其便会被 `init` 进程进行收养并最终回收资源。关于 pause 工作方式的详细信息可以参考：[The Almighty Pause Container](https://www.ianlewis.org/en/almighty-pause-container)。

一旦创建好了 pause 容器，下面就会开始检查磁盘状态然后开始启动业务容器。



#### CNI 和 Pod 网络

现在我们的 Pod 已经有了基本的骨架：一个共享所有命名空间以允许业务容器在同一个 Pod 里进行通信的 pause 容器。但现在还有一个问题，那就是容器的网络是如何建立的？

当 Kubelet 为 Pod 创建网络时，它会将创建网络的任务交给 `CNI` 插件。CNI 表示容器网络接口（Container Network Interface），和容器运行时的运行方式类似，它也是一种抽象，允许不同的网络提供商为容器提供不同的网络实现。通过将 json 配置文件（默认在 `/etc/cni/net.d` 路径下）中的数据传送到相关的 CNI 二进制文件（默认在 `/opt/cni/bin` 路径下）中，cni 插件可以给 pause 容器配置相关的网络，然后 Pod 中其他的容器都使用 pause 容器的网络。下面是一个简单的示例配置文件：

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

CNI 插件还会通过 `CNI_ARGS` 环境变量为 Pod 指定其他的元数据，包括 Pod 名称和命名空间。

下面的步骤因 CNI 插件而异，我们以 `bridge` 插件举例：

- 该插件首先会在根网络命名空间（也就是宿主机的网络命名空间）中设置本地 Linux 网桥，以便为该主机上的所有容器提供网络服务。
- 然后它会将一个网络接口（`veth` 设备对的一端）插入到 pause 容器的网络命名空间中，并将另一端连接到网桥上。你可以这样来理解 veth 设备对：它就像一根很长的管道，一端连接到容器，一端连接到根网络命名空间中，数据包就在管道中进行传播。
- 接下来 json 文件中指定的 `IPAM` Plugin 会为 pause 容器的网络接口分配一个 IP 并设置相应的路由，现在 Pod 就有了自己的 IP。
  - IPAM Plugin 的工作方式和 CNI Plugin 类似：通过二进制文件调用并具有标准化的接口，每一个 IPAM Plugin 都必须要确定容器网络接口的 IP、子网以及网关和路由，并将信息返回给 CNI 插件。最常见的 IPAM Plugin 是 `host-local`，它从预定义的一组地址池中为容器分配 IP 地址。它将地址池的信息以及分配信息保存在主机的文件系统中，从而确保了同一主机上每个容器的 IP 地址的唯一性。
- 最后 Kubelet 会将集群内部的 `DNS` 服务器的 `Cluster IP` 地址传给 CNI 插件，然后 CNI 插件将它们写到容器的 `/etc/resolv.conf` 文件中。

一旦完成了上面的步骤，CNI 插件就会将操作的结果以 json 的格式返回给 Kubelet。



#### 跨主机容器网络

到目前为止，我们已经描述了容器如何与宿主机进行通信，但跨主机之间的容器如何通信呢？

通常情况下使用 `overlay` 网络来进行跨主机容器通信，这是一种动态同步多个主机间路由的方法。 其中最常用的 overlay 网络插件是 `flannel`，flannel 具体的工作方式可以参考 [CoreOS 的文档](https://github.com/coreos/flannel)。



#### 容器启动

所有网络都配置完成后，接下来就开始真正启动业务容器了！

一旦 sanbox 完成初始化并处于 `active` 状态，Kubelet 就可以开始为其创建容器了。首先[启动 PodSpec 中定义的 init 容器](https://github.com/kubernetes/kubernetes/blob/5adfb24f8f25a0d57eb9a7b158db46f9f46f0d80/pkg/kubelet/kuberuntime/kuberuntime_manager.go#L690)，然后再启动业务容器。具体过程如下：

1. 首先拉取容器的镜像。如果是私有仓库的镜像，就会利用 PodSpec 中指定的 Secret 来拉取该镜像。
2. 然后通过 CRI 接口创建容器。Kubelet 向 PodSpec 中填充了一个 `ContainerConfig` 数据结构（在其中定义了命令，镜像，标签，挂载卷，设备，环境变量等待），然后通过 `protobufs` 发送给 CRI 接口。对于 Docker 来说，它会将这些信息反序列化并填充到自己的配置信息中，然后再发送给 `Dockerd` 守护进程。在这个过程中，它会将一些元数据标签（例如容器类型，日志路径，sandbox ID 等待）添加到容器中。
3. 接下来会使用 CPU 管理器来约束容器，这是 Kubelet 1.8 中新添加的 alpha 特性，它使用 `UpdateContainerResources` CRI 方法将容器分配给本节点上的 CPU 资源池。
4. 最后容器开始真正[启动](https://github.com/kubernetes/kubernetes/blob/5f9f4a1c5939436fa320e9bc5973a55d6446e59f/pkg/kubelet/kuberuntime/kuberuntime_container.go#L135)。
5. 如果 Pod 中配置了容器生命周期钩子（Hook），容器启动之后就会运行这些 `Hook`。Hook 的类型包括两种：`Exec`（执行一段命令） 和 `HTTP`（发送HTTP请求）。如果 PostStart Hook 启动的时间过长、挂起或者失败，容器将永远不会变成 `running` 状态。



### 7. 总结

------

如果上面一切顺利，现在你的集群上应该会运行三个容器，所有的网络，数据卷和秘钥都被通过 CRI 接口添加到容器中并配置成功。

上文所述的创建 Pod 整个过程的流程图如下所示：

![img](D:\学习资料\笔记\k8s\k8s图\what-happens-when-k8s.svg)





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







































































































































































































































































