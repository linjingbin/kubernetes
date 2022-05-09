[TOC]

# 1 简介

## 1.1 docker 简介

### 1.1.1 Docker 是什么

   首先 Docker 是一个在 2003 年开源的应用程序并且是一个基于 go 语言编写，是一个开源的 PAAS 服务，go 语言是由 google 开发，docker 公司最早叫 dotCloud 后由于 Docker 开源后大受欢迎就将公司改名为 Docker lnc，总部位于美国加州的旧金山，Docker 是基于 linux 内核实现，Docker 最早采用 LXC 技术(LinuX Container 的简写，LXC 是 Linux 原生支持的容器技术，可以提供轻量级的虚拟化，可以说 docker 就是基于 LXC 发展起来的，提供 LXC 的高级封装，发展标准的配置方法)，而虚拟化技术 KVM(Kernel-based Virtual Machine) 基于模块实现，Docker 后改为自己研发并开源的 runc 技术运行容器。

   Docker 相比虚拟机的交付速度更快，资源消耗更低，Docker 采用客户端/服务端架构，使用远程 API 来管理和创建 Docker 容器，其可以轻松的创建一个轻量级的、可移植的、自给自足的容器，docker 的三大理念是 build(构建)、ship(运输)、 run(运行)，Docker 遵从 apache 2.0 协议，并通过（namespace 及cgroup 等）来提供容器的资源隔离与安全保障等，所以 Docke 容器在运行时不需要类似虚拟机（空运行的虚拟机占用物理机 6-8%性能）的额外资源开销，因此可以大幅提高资源利用率,总而言之 Docker 是一种用了新颖方式实现的轻量级虚拟机.类似于 VM 但是在原理和应用上和 VM 的差别还是很大的，并且 docker 的专业叫法是应用容器(Application Container)。



###  1.1.2 Docker 的组成

https://docs.docker.com/engine/docker-overview/
Docker 主机(Host)：一个物理机或虚拟机，用于运行 Docker 服务进程和容器。

Docker 服务端(Server)：Docker 守护进程，运行 docker 容器。

Docker 客户端(Client)：客户端使用 docker 命令或其他工具调用 docker API。

Docker 仓库(Registry): 保存镜像的仓库，类似于 git 或 svn 这样的版本控制系

Docker 镜像(Images)：镜像可以理解为创建实例使用的模板。

Docker 容器(Container): 容器是从镜像生成对外提供服务的一个或一组服务。

官方仓库: https://hub.docker.com/

![1587454222743](D:\学习资料\笔记\linux\assets\1587454222743.png)





### 1.1.3 Docker 对比虚拟机

资源利用率更高：一台物理机可以运行数百个容器，但是一般只能运行数十个虚拟机。

开销更小：不需要启动单独的虚拟机占用硬件资源。

启动速度更快：可以在数秒内完成启动。

![1587454336113](D:\学习资料\笔记\linux\assets\1587454336113.png)

![1587454366145](D:\学习资料\笔记\linux\assets\1587454366145.png)

使用虚拟机是为了更好的实现服务运行环境隔离，每个虚拟机都有独立的内核，虚拟化可以实现不同操作系统的虚拟机，但是通常一个虚拟机只运行一个服务，很明显资源利用率比较低且造成不必要的性能损耗，我们创建虚拟机的目的是为了运行应用程序，比如 Nginx、PHP、Tomcat 等 web 程序，使用虚拟机无疑带来了一些不必要的资源开销，但是容器技术则基于减少中间运行环节带来较大的性能提升。

![1587454431389](D:\学习资料\笔记\linux\assets\1587454431389.png)

但是，如上图一个宿主机运行了 N 个容器，多个容器带来的以下问题怎么解决:

1.怎么样保证每个容器都有不同的文件系统并且能互不影响

2.一个 docker 主进程内的各个容器都是其子进程，那么实现同一个主进程下不同类型的子进程？各个进程间通信能相互访问(内存数据)吗？

3.每个容器怎么解决 IP 及端口分配的问题？

4.多个容器的主机名能一样吗？

5.每个容器都要不要有 root 用户？怎么解决账户重名问题？

以上问题怎么解决？



### 1.1.4 Linux Namespace 技术

namespace 是 Linux 系统的底层概念，在内核层实现，即有一些不同类型的命名空间被部署在核内，各个 docker 容器运行在同一个 docker 主进程并且共用同一个宿主机系统内核，各 docker 容器运行在宿主机的用户空间，每个容器都要有类似于虚拟机一样的相互隔离的运行空间，但是容器技术是在一个进程内实现运行指定服务的运行环境，并且还可以保护宿主机内核不受其他进程的干扰和影响，如文件系统空间、网络空间、进程空间等，目前主要通过以下技术实现。
容器运行空间的相互隔离：

![1587454700985](D:\学习资料\笔记\linux\assets\1587454700985.png)

![1587454726223](D:\学习资料\笔记\linux\assets\1587454726223.png)



#### 1.1.4.1 MNT Namespace

每个容器都要有独立的根文件系统有独立的用户空间，以实现在容器里面启动服务并且使用容器的运行环境，即一个宿主机是 ubuntu 的服务器，可以在里面启动一个 centos 运行环境的容器并且在容器里面启动一个 Nginx 服务，此 Nginx 运行时使用的运行环境就是 centos 系统目录的运行环境，但是在容器里面是不能访问宿主机的资源，宿主机是使用了 chroot 技术把容器锁定到一个指定的运行目录里面。

例如：/var/lib/containerd/io.containerd.runtime.v1.linux/moby/ 容器 ID

启动三个容器用于以下验证过程：

```bash
Server: Docker Engine - Community
Engine:
Version: 18.09.7
API version: 1.39 (minimum version 1.12)
Go version: go1.10.8
Git commit: 2d0083d
Built: Thu Jun 27 17:26:28 2019
OS/Arch: linux/amd64
Experimental: false
# docker run -d --name nginx-1 -p 80:80 nginx
# docker run -d --name nginx-2 -p 81:80 nginx
# docker run -d --name nginx-3 -p 82:80 nginx
```

Debian 系统安装基础命令：

```bash
# apt update
# apt install procps (top 命令)
# apt install iputils-ping (ping 命令)
# apt install net-tools (网络工具)
```

验证容器的根文件系统：

![1587455016134](D:\学习资料\笔记\linux\assets\1587455016134.png)



#### 1.1.4.2 IPC Namespace

一个容器内的进程间通信，允许一个容器内的不同进程的(内存、缓存等)数据访问，但是不能夸容器访问其他容器的数据。



#### 1.1.4.3 UTS Namespace

UTS namespace（UNIX Timesharing System 包含了运行内核的名称、版本、底层体系结构类型等信息）用于系统标识，其中包含了 hostname 和域名domainname ，它使得一个容器拥有属于自己 hostname 标识，这个主机名标识独立于宿主机系统和其上的其他容器。

![1587455193839](D:\学习资料\笔记\linux\assets\1587455193839.png)



#### 1.1.4.4 PID Namespace

Linux 系统中，有一个 PID 为 1 的进程(init/systemd)是其他所有进程的父进程，那么在每个容器内也要有一个父进程来管理其下属的子进程，那么多个容器的进程通 PID namespace 进程隔离(比如 PID 编号重复、容器内的主进程生成与回收子进程等)。

例如：下图是在一个容器内使用 top 命令看到的 PID 为 1 的进程是 nginx ：

![1587457367890](D:\学习资料\笔记\linux\assets\1587457367890.png)

容器内的 Nginx 主进程与工作进程：

![1587457396020](D:\学习资料\笔记\linux\assets\1587457396020.png)

那么宿主机的 PID 究竟与容器内的 PID 是什么关系？

容器 PID 追踪：

##### 1.1.4.4.1 查看宿主机上的 PID 信息

![1587457481928](D:\学习资料\笔记\linux\assets\1587457481928.png)

##### 1.1.4.4.2 查看容器中的 PID 信息

![1587457624199](D:\学习资料\笔记\linux\assets\1587457624199.png)



#### 1.1.4.5 Net Namespace

每一个容器都类似于虚拟机一样有自己的网卡、监听端口、TCP/IP 协议栈等，Docker 使用 network namespace 启动一个 vethX 接口，这样你的容器将拥有它自己的桥接 ip 地址，通常是 docker0，而 docker0 实质就是 Linux 的虚拟网桥,网桥是在 OSI 七层模型的数据链路层的网络设备，通过 mac 地址对网络进行划分，并且在不同网络直接传递数据。

##### 1.1.4.5.1 查看宿主机的网卡信息

![1587457811665](D:\学习资料\笔记\linux\assets\1587457811665.png)

##### 1.1.4.5.2 查看宿主机桥接设备

通过 brctl show 命令查看桥接设备：

![1587457889052](D:\学习资料\笔记\linux\assets\1587457889052.png)

![1587457907539](D:\学习资料\笔记\linux\assets\1587457907539.png)

##### 1.1.4.5.3 逻辑网络图

![1587457978213](D:\学习资料\笔记\linux\assets\1587457978213.png)

##### 1.1.4.5.3 宿主机 iptables 规则

![1587458036518](D:\学习资料\笔记\linux\assets\1587458036518.png)

![1587458115147](D:\学习资料\笔记\linux\assets\1587458115147.png)

#### 1.1.4.6 User Namespace

各个容器内可能会出现重名的用户和用户组名称，或重复的用户 UID 或者GID，那么怎么隔离各个容器内的用户空间呢？

User Namespace 允许在各个宿主机的各个容器空间内创建相同的用户名以及相同的用户 UID 和 GID，只是会把用户的作用范围限制在每个容器内，即 A 容器和 B 容器可以有相同的用户名称和 ID 的账户，但是此用户的有效范围仅是当前容器内，不能访问另外一个容器内的文件系统，即相互隔离、互补影响、永不相见。

![1587458284615](D:\学习资料\笔记\linux\assets\1587458284615.png)



### 1.1.5 Linux control groups

在一个容器，如果不对其做任何资源限制，则宿主机会允许其占用无限大的内存空间，有时候会因为代码 bug 程序会一直申请内存，直到把宿主机内存占完，为了避免此类的问题出现，宿主机有必要对容器进行资源分配限制，比如
CPU、内存等，Linux Cgroups 的全称是 Linux Control Groups，它最主要的作用，就是限制一个进程组能够使用的资源上限，包括 CPU、内存、磁盘、网络带宽等等。此外，还能够对进程进行优先级设置，以及将进程挂起和恢复等操作。

#### 1.1.5.1 验证系统 cgroups

Cgroups 在内核层默认已经开启，从 centos 和 ubuntu 对比结果来看，显然内核较新的 ubuntu 支持的功能更多。

##### 1.1.5.1.1 Cenos 7.6 cgroups

![1587458481442](D:\学习资料\笔记\linux\assets\1587458481442.png)

##### 1.1.5.1.2 ubuntu cgroups

![1587458519382](D:\学习资料\笔记\linux\assets\1587458519382.png)

#### 1.1.5.2 cgroup 具体实现

```bash
blkio：块设备 IO 限制。
cpu：使用调度程序为 cgroup 任务提供 cpu 的访问。
cpuacct：产生 cgroup 任务的 cpu 资源报告。
cpuset：如果是多核心的 cpu，这个子系统会为 cgroup 任务分配单独的 cpu 和内存。
devices：允许或拒绝 cgroup 任务对设备的访问。
freezer：暂停和恢复 cgroup 任务。
memory：设置每个 cgroup 的内存限制以及产生内存资源报告。
net_cls：标记每个网络包以供 cgroup 方便使用。
ns：命名空间子系统。
perf_event：增加了对每 group 的监测跟踪的能力，可以监测属于某个特定的 group 的所
有线程以及运行在特定 CPU 上的线程。
```

#### 1.1.5.3 查看系统 cgroups

```bash
[root@node1 ~]$ ll /sys/fs/cgroup/
total 0
drwxr-xr-x 5 root root 0 Jun 30 18:12 blkio
lrwxrwxrwx 1 root root 11 Jun 30 18:12 cpu -> cpu,cpuacct
lrwxrwxrwx 1 root root 11 Jun 30 18:12 cpuacct -> cpu,cpuacct
drwxr-xr-x 5 root root 0 Jun 30 18:12 cpu,cpuacct
drwxr-xr-x 3 root root 0 Jun 30 18:12 cpuset
drwxr-xr-x 5 root root 0 Jun 30 18:12 devices
drwxr-xr-x 3 root root 0 Jun 30 18:12 freezer
drwxr-xr-x 3 root root 0 Jun 30 18:12 hugetlb
drwxr-xr-x 5 root root 0 Jun 30 18:12 memory
drwxr-xr-x 3 root root 0 Jun 30 18:12 net_cls
drwxr-xr-x 3 root root 0 Jun 30 18:12 perf_event
```

有了以上的 chroot、namespace、cgroups 就具备了基础的容器运行环境，但是还需要有相应的容器创建与删除的管理工具、以及怎么样把容器运行起来、容器数据怎么处理、怎么进行启动与关闭等问题需要解决，于是容器管理技术出现了。

### 1.1.6 容器管理工具

目前主要是使用 docker，早期有使用 lxc。

#### 1.1.6.1 LXC

LXC：LXC 为 Linux Container 的简写。可以提供轻量级的虚拟化，以便隔离进程和资源，官方网站：https://linuxcontainers.org/

Ubuntu 安装 lxc:

```bash
root@s1:~# apt install lxc lxd
Reading package lists... Done
Building dependency tree
Reading state information... Done
lxd is already the newest version (3.0.3-0ubuntu1~18.04.1).
lxc is already the newest version (3.0.3-0ubuntu1~18.04.1).
root@s1:~# lxc-checkconfig #检查内核对 lcx 的支持状况，必须全部为 lcx
root@s1:~# lxc-create -t 模板名称 -n lcx-test
root@s1:~# lxc-create -t download --name alpine12 -- --dist alpine --
release 3.9 --arch amd64
Setting up the GPG keyring
Downloading the image index
Downloading the rootfs
Downloading the metadata
The image cache is now ready
Unpacking the rootfs
---
You just created an Alpinelinux 3.9 x86_64 (20190630_13:00) container.
root@s1:~# lxc-start alpine12 #启动 lxc 容器
root@s1:~# lxc-attach alpine12 #进入 lxc 容器
~ # ifconfig
eth0 Link encap:Ethernet HWaddr 00:16:3E:DF:54:94
inet addr:10.0.3.115 Bcast:10.0.3.255 Mask:255.255.255.0
inet6 addr: fe80::216:3eff:fedf:5494/64 Scope:Link
UP BROADCAST RUNNING MULTICAST MTU:1500 Metric:1
RX packets:18 errors:0 dropped:0 overruns:0 frame:0
TX packets:13 errors:0 dropped:0 overruns:0 carrier:0
collisions:0 txqueuelen:1000
RX bytes:2102 (2.0 KiB) TX bytes:1796 (1.7 KiB)
lo Link encap:Local Loopback
inet addr:127.0.0.1 Mask:255.0.0.0
inet6 addr: ::1/128 Scope:Host
UP LOOPBACK RUNNING MTU:65536 Metric:1
RX packets:0 errors:0 dropped:0 overruns:0 frame:0
TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
collisions:0 txqueuelen:1000
RX bytes:0 (0.0 B) TX bytes:0 (0.0 B)
~ # uname -a
Linux alpine12 4.15.0-20-generic #21-Ubuntu SMP Tue Apr 24 06:16:15 UTC 2018
x86_64 Linux
~ # cat /etc/issue
Welcome to Alpine Linux 3.9
Kernel \r on an \m (\l)

命令备注：
-t 模板: -t 选项后面跟的是模板，模式可以认为是一个原型，用来说明我们需要一个什么
样的容器(比如容器里面需不需要有 vim, apache 等软件)．模板实际上就是一个脚本文件(位
于/usr/share/lxc/templates 目录)，我们这里指定 download 模板(lxc-create 会调用 lxc-
download 脚本，该脚本位于刚说的模板目录中)是说明我们目前没有自己模板，需要下载官方的模板
--name 容器名称： 为创建的容器命名
-- : --用来说明后面的参数是传递给 download 脚本的，告诉脚本需要下载什么样的模板
--dist 操作系统名称：指定操作系统
--release 操作系统: 指定操作系统，可以是各种 Linux 的变种
--arch 架构：指定架构，是 x86 还是 arm，是 32 位还是 64 位
```

lxc 启动容器依赖于模板，清华模板源：
https://mirrors.tuna.tsinghua.edu.cn/help/lxc-images/，但是做模板相对较难，需要手动一步步创构建文件系统、准备基础目录及可执行程序等，而且在大规模使用容器的场景很难横向扩展，另外后期代码升级也需要重新从头构建模板，基于以上种种原因便有了 docker。

#### 1.1.6.2 docker

Docker 启动一个容器也需要一个外部模板，但是较多 docke 的镜像可以保存在一个公共的地方共享使用，只要把镜像下载下来就可以使用，最主要的是可以在镜像基础之上做自定义配置并且可以再把其提交为一个镜像，一个镜像可以被启动为多个容器。

Docker 的镜像是分层的，镜像底层为库文件且只读层即不能写入也不能删除数据，从镜像加载启动为一个容器后会生成一个可写层，其写入的数据会复制到容器目录，但是容器内的数据在删除容器后也会被随之删除。

![1587462534159](D:\学习资料\笔记\linux\assets\1587462534159.png)



#### 1.1.6.3 pouth

https://www.infoq.cn/article/alibaba-pouch

https://github.com/alibaba/pouch



### 1.1.6 Dock 的优势

快速部署：短时间内可以部署成百上千个应用，更快速交付到线上。

高效虚拟化：不需要额外的 hypervisor 支持，直接基于 linux 实现应用虚拟化，相比虚拟机大幅提高性能和效率。

节省开支：提高服务器利用率，降低 IT 支出。

简化配置：将运行环境打包保存至容器，使用时直接启动即可。

快速迁移和扩展：可夸平台运行在物理机、虚拟机、公有云等环境，良好的兼容性可以方便将应用从 A 宿主机迁移到 B 宿主机，甚至是 A 平台迁移到 B 平台。



### 1.1.7 Docker 的缺点

隔离性：各应用之间的隔离不如虚拟机彻底。



### 1.1.8 docker (容器) 的核心技术

**容器规范：**

​      除了 docker 之外的 docker 技术，还有 coreOS 的 rkt，还有阿里的 Pouch，为了保证容器生态的标准性和健康可持续发展，包括 Linux 基金会、Docker、微软、红帽谷歌和、IBM、等公司在 2015 年 6 月共同成立了一个叫 open container （OCI）的组织，其目的就是制定开放的标准的容器规范，目前 OCI 一共发布了两个规范，分别是 runtime spec 和 和 image format spec，有了这两个规范，不同的容器公司开发的容器只要兼容这两个规范，就可以保证容器的可移植性和相互可操作性。

**容器 runtime ：**

runtime 是真正运行容器的地方，因此为了运行不同的容器 runtime 需要和操作系统内核紧密合作相互在支持，以便为容器提供相应的运行环境。

目前主流的三种 runtime：

Lxc：linux 上早期的 runtime，Docker 早期就是采用 lxc 作为 runtime。

runc：目前 Docker 默认的 runtime，runc 遵守 OCI 规范，因此可以兼容 lxc。

rkt：是 CoreOS 开发的容器 runtime，也符合 OCI 规范，所以使用 rktruntime 也可以运行 Docker 容器。

**容器管理工具：**

管理工具连接 runtime 与用户，对用户提供图形或命令方式操作，然后管理工具将用户操作传递给 runtime 执行。

lxc 是 lxd 的管理工具。

Runc 的管理工具是 docker engine，docker engine 包含后台 deamon 和 cli 两部分，大家经常提到的 Docker 就是指的 docker engine。

Rkt 的管理工具是 rkt cli。

**容器定义工具：**

容器定义工具允许用户定义容器的属性和内容，以方便容器能够被保存、共享和重建。

Docker image：是 docker 容器的模板，runtime 依据 docker image 创建容器。

Dockerfile：包含 N 个命令的文本文件，通过 dockerfile 创建出 docker image。

ACI(App container image)：与 docker image 类似，是 CoreOS 开发的 rkt 容器的镜像格式。

**Registry ：**

统一保存镜像而且是多个不同镜像版本的地方，叫做镜像仓库。

Image registry：docker 官方提供的私有仓库部署工具。

Docker hub：docker 官方的公共仓库，已经保存了大量的常用镜像，可以方便大家直接使用。

Harbor：vmware 提供的自带 web 界面自带认证功能的镜像仓库，目前有很多公司使用。

**编排工具 ：**

当多个容器在多个主机运行的时候，单独管理容器是相当复杂而且很容易出错，而且也无法实现某一台主机宕机后容器自动迁移到其他主机从而实现高可用的目的，也无法实现动态伸缩的功能，因此需要有一种工具可以实现统一管理、动态伸缩、故障自愈、批量执行等功能，这就是容器编排引擎。

容器编排通常包括容器管理、调度、集群定义和服务发现等功能。

Docker swarm：docker 开发的容器编排引擎。

Kubernetes：google 领导开发的容器编排引擎，内部项目为 Borg，且其同时支持docker 和 CoreOS。

Mesos+Marathon：通用的集群组员调度平台，mesos(资源分配)与 marathon(容器编排平台)一起提供容器编排引擎功能。



### 1.1.9 docker （容器）的依赖技术

**容器网络：**

docker 自带的网络 docker network 仅支持管理单机上的容器网络，当多主机运行的时候需要使用第三方开源网络，例如 calico、flannel 等。

**服务发现：**

容器的动态扩容特性决定了容器 IP 也会随之变化，因此需要有一种机制开源自动识别并将用户请求动态转发到新创建的容器上，kubernetes 自带服务发现功能，需要结合 kube-dns 服务解析内部域名。

**容器监控：**

可以通过原生命令 docker ps/top/stats 查看容器运行状态，另外也可以使heapster/ Prometheus 等第三方监控工具监控容器的运行状态。

**数据管理：**

容器的动态迁移会导致其在不同的 Host 之间迁移，因此如何保证与容器相关的数据也能随之迁移或随时访问，可以使用逻辑卷/存储挂载等方式解决。

**日志收集：**

docker 原生的日志查看工具 docker logs，但是容器内部的日志需要通过 ELK 等专门的日志收集分析和展示工具进行处理。



## 1.2 Docker 安装及基础命令介绍

官方网址：https://www.docker.com/

**系统版本选择：**

Docker 目前已经支持多种操作系统的安装运行，比如 Ubuntu、CentOS、Redhat、Debian、Fedora，甚至是还支持了 Mac 和 Windows，在 linux 系统上需要内核版本在 3.10 或以上，docker 版本号之前一直是 0.X 版本或 1.X 版本，但是从 2017 年 3 月 1 号开始改为每个季度发布一次稳版，其版本号规则也统一变更为 YY.MM，例如 17.09 表示是 2017 年 9 月份发布的，本次演示的操作系统使用 Centos 7.5 为例。

**Docker  版本选择 ：**
Docker 之前没有区分版本，但是 2017 年推出(将 docker 更名为)新的项目Moby，github 地址：https://github.com/moby/moby，Moby 项目属于 Docker 项目的全新上游，Docker 将是一个隶属于的 Moby 的子产品，而且之后的版本之后开始区分为 CE 版本（社区版本）和 EE（企业收费版），CE 社区版本和 EE 企业版本都是每个季度发布一个新版本，但是 EE 版本提供后期安全维护 1 年，而CE 版本是 4 个月，本次演示的 Docker 版本为 18.03，以下为官方原文：
https://blog.docker.com/2017/03/docker-enterprise-edition/

### 1.2.1 下载 rpm 包安装：

官方 rpm 包下载地址:
https://download.docker.com/linux/centos/7/x86_64/stable/Packages/

阿里镜像下载地址：
https://mirrors.aliyun.com/docker-ce/linux/centos/7/x86_64/stable/Packages/

![1587475386394](D:\学习资料\笔记\linux\assets\1587475386394.png)

### 1.2.2 通过修改 yum 源安装

```bash
[root@docker-server1 ~]$ rm -rf /etc/yum.repos.d/*
[root@docker-server1 ~]$ wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
[root@docker-server1 ~]$ wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
[root@docker-server1 ~]$ wget -O /etc/yum.repos.d/docker-ce.repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
[root@docker-server1 ~]$ yum install docker-ce
```

### 1.2.3 启动并验证 docker 服务

```bash
[root@docker-server1 ~]$ systemctl start docker
[root@docker-server1 ~]$ systemctl enable docker
```

### 1.2.4 验证 docker 版本

![1587476168779](D:\学习资料\笔记\linux\assets\1587476168779.png)

### 1.2.5 验证 docker0 网卡

在 docker 安装启动之后，默认会生成一个名称为 docker0 的网卡并且默认 IP 地址为 172.17.0.1 的网卡。

![1587476224082](D:\学习资料\笔记\linux\assets\1587476224082.png)

### 1.2.6 验证 docker 信息

![1587479793581](D:\学习资料\笔记\linux\assets\1587479793581.png)

### 1.2.7 docker 存储引擎

目前 docker 的默认存储引擎为 overlay2，需要磁盘分区支持 d-type 文件分层功能，因此需要系统磁盘的额外支持。

官方文档关于存储引擎的选择文档：
https://docs.docker.com/storage/storagedriver/select-storage-driver/

Docker 官方推荐首选存储引擎为 overlay2 其次为 devicemapper，但是 devicemapper 存在使用空间方面的一些限制，虽然可以通过后期配置解决，但是官方依然推荐使用 overlay2，以下是网上查到的部分资料：
https://www.cnblogs.com/youruncloud/p/5736718.html

![1587481046050](D:\学习资料\笔记\linux\assets\1587481046050.png)

如果 docker 数据目录是一块单独的磁盘分区而且是 xfs 格式的，那么需要在格式化的时候加上参数-n ftype=1，否则后期在启动容器的时候会报错不支持 d-type。

![1587481097016](D:\学习资料\笔记\linux\assets\1587481097016.png)

报错界面：

![1587481131085](D:\学习资料\笔记\linux\assets\1587481131085.png)



### 1.2.8 docker 服务进程

通过查看 docker 进程，了解 docker 的运行及工作方式

#### 1.2.8.1 查看宿主机进程树

```bash
[root@ansible_ ~]$ docker version
Client:
 Version:           18.09.9
 API version:       1.39
 Go version:        go1.11.13
 Git commit:        039a7df9ba
 Built:             Wed Sep  4 16:51:21 2019
 OS/Arch:           linux/amd64
 Experimental:      false

Server: Docker Engine - Community
 Engine:
  Version:          18.09.9
  API version:      1.39 (minimum version 1.12)
  Go version:       go1.11.13
  Git commit:       039a7df
  Built:            Wed Sep  4 16:22:32 2019
  OS/Arch:          linux/amd64
  Experimental:     false

[root@k8s-6 ~]$ pstree -p 1
systemd(1)─┬─NetworkManager(3823)─┬─{NetworkManager}(3839)
           │                      └─{NetworkManager}(3843)
           ├─VGAuthService(4160)
           ├─agetty(3834)
           ├─atd(3828)
           ├─auditd(27058)───{auditd}(27059)
           ├─containerd(18180)─┬─{containerd}(18184)
           │                   ├─{containerd}(18185)
           │                   ├─{containerd}(18186)
           │                   ├─{containerd}(18187)
           │                   ├─{containerd}(18188)
           │                   ├─{containerd}(18189)
           │                   ├─{containerd}(18190)
           │                   ├─{containerd}(18191)
           │                   ├─{containerd}(18192)
           │                   ├─{containerd}(18193)
           │                   ├─{containerd}(18204)
           │                   ├─{containerd}(18205)
           │                   ├─{containerd}(18206)
           │                   ├─{containerd}(18207)
           │                   ├─{containerd}(18208)
           │                   └─{containerd}(27757)
           ├─crond(27021)
           ├─dbus-daemon(3816)
           ├─dockerd(28428)─┬─{dockerd}(28429)
           │                ├─{dockerd}(28430)
           │                ├─{dockerd}(28431)
           │                ├─{dockerd}(28432)
           │                ├─{dockerd}(28433)
           │                ├─{dockerd}(28434)
           │                ├─{dockerd}(28435)
           │                ├─{dockerd}(28436)
           │                ├─{dockerd}(28437)
           │                ├─{dockerd}(28438)
           │                ├─{dockerd}(28440)
           │                ├─{dockerd}(28441)
           │                ├─{dockerd}(28442)
           │                ├─{dockerd}(28443)
           │                ├─{dockerd}(28445)
           │                ├─{dockerd}(28446)
           │                ├─{dockerd}(28609)
           │                └─{dockerd}(28711)
```

#### 1.2.8.2 查看 containerd 进程关系

**有四个进程：**

dockerd：被 client 直接访问，其父进程为宿主机的 systemd 守护进程。

docker-proxy：实现容器通信，其父进程为 dockerd

containerd：被 dockerd 进程调用以实现与 runc 交互。

containerd-shim：真正运行容器的载体，其父进程为 containerd。

![1587543497655](D:\学习资料\笔记\linux\assets\1587543497655.png)

#### 1.2.8.3 containerd-shim 命令使用

```bash
[root@ansible_ ~]$ containerd-shim -h
Usage of containerd-shim:
  -address string
        grpc address back to main containerd
  -containerd-binary containerd publish
        path to containerd binary (used for containerd publish) (default "containerd")
  -criu string
        path to criu binary
  -debug
        enable debug output in logs
  -namespace string
        namespace that owns the shim
  -runtime-root string
        root directory for the runtime (default "/run/containerd/runc")
  -socket string
        abstract socket path to serve
  -systemd-cgroup
        set runtime to use systemd-cgroup
  -workdir string
        path used to storge large temporary data
```

#### 1.2.8.4 容器的创建与管理过程

```bash
通信流程：
1. dockerd 通过 grpc 和 containerd 模块通信，dockerd 由 libcontainerd 负责和 containerd 进行交换，dockerd 和 containerd 通信 socket 文件：/run/containerd/containerd.sock。
2. containerd 在 dockerd 启动时被启动，然后 containerd 启动 grpc 请求监听，containerd 处理 grpc 请求，根据请求做相应动作。
3. 若是 start 或是 exec 容器，containerd 拉起一个 container-shim , 并进行相应的操作。
4. container-shim 被拉起后，start/exec/create 拉起 runC 进程，通过 exit、control 文件和containerd 通信，通过父子进程关系和 SIGCHLD 监控容器中进程状态。
5. 在整个容器生命周期中，containerd 通过 epoll 监控容器文件，监控容器事件。
```

![1587543724785](D:\学习资料\笔记\linux\assets\1587543724785.png)

#### 1.2.8.5 grpc 简介

gPRC 是 Google 开发的一款高性能、开源和通用的 RPC 框架，支持众多语言客户端。

![1587543905832](D:\学习资料\笔记\linux\assets\1587543905832.png)

## 1.3 docker 镜像加速配置

国内下载国外的镜像有时候会很慢，因此可以更改 docker 配置文件添加一个加速器，可以通过加速器达到加速下载镜像的目的。

### 1.3.1 获取加速地址

浏览器打开 http://cr.console.aliyun.com,注册或登录阿里云账号，点击左侧的镜像加速器，将会得到 一个专属的加速地址，而且下面有使用配置说明：

![1587544717291](D:\学习资料\笔记\linux\assets\1587544717291.png)

### 1.3.2 生成配置文件

```bash
[root@docker-server1 ~]$ mkdir -p /etc/docker
[root@docker-server1 ~]$ sudo tee /etc/docker/daemon.json <<-'EOF'
> {
> "registry-mirrors": ["https://9916w1ow.mirror.aliyuncs.com"]
> }
> EOF
{
"registry-mirrors": ["https://9916w1ow.mirror.aliyuncs.com"]
}
[root@docker-server1 ~]$ cat /etc/docker/daemon.json
{
"registry-mirrors": ["https://9916w1ow.mirror.aliyuncs.com"]
}
```

### 1.3.3 重启 docker 服务

```bash
[root@docker-server1 ~]$ systemctl daemon-reload
[root@docker-server1 ~]$ sudo systemctl restart docker
```



## 1.4 Docker 镜像管理

Docker 镜像含有启动容器所需要的文件系统及所需要的内容，因此镜像主要用于创建并启动 docker 容器。

Docker 镜像含里面是一层层文件系统,叫做 Union FS（联合文件系统）,联合文件系统，可以将几层目录挂载到一起，形成一个虚拟文件系统,虚拟文件系统的目录结构就像普通 linux 的目录结构一样，docker 通过这些文件再加上宿主机的内核提供了一个 linux 的虚拟环境,每一层文件系统我们叫做一层 layer，联合文件系统可以对每一层文件系统设置三种权限，只读（readonly）、读写（readwrite）和写出（whiteout-able），但是 docker 镜像中每一层文件系统都是只读的,构建镜像的时候,从一个最基本的操作系统开始,每个构建的操作都相当于做一层的修改,增加了一层文件系统,一层层往上叠加,上层的修改会覆盖底层该位置的可见性，这也很容易理解，就像上层把底层遮住了一样,当使用镜像的时候，我们只会看到一个完全的整体，不知道里面有几层也不需要知道里面有几层，结构如下：



![1587544956389](D:\学习资料\笔记\linux\assets\1587544956389.png)



一个典型的 Linux 文件系统由 bootfs 和 rootfs 两部分组成，bootfs(boot filesystem) 主要包含 bootloader 和 kernel，bootloader 主要用于引导加载 kernel，当 kernel 被加载到内存中后 bootfs 会被 umount 掉，rootfs (root file system) 包
含的就是典型 Linux 系统中的/dev，/proc，/bin，/etc 等标准目录和文件，下图就是 docker image 中最基础的两层结构，不同的 linux 发行版（如 ubuntu和 CentOS ) 在 rootfs 这一层会有所区别。

但是对于 docker 镜像通常都比较小，官方提供的 centos 基础镜像在 200MB 左右，一些其他版本的镜像甚至只有几 MB，docker 镜像直接调用宿主机的内核，镜像中只提供 rootfs，也就是 只需要包括最基本的命令、工具和程序库就可以了，比如 alpine 镜像，在 5M 左右。下图就是有两个不同的镜像在一个宿主机内核上实现不同的 rootfs。

![1587545069159](D:\学习资料\笔记\linux\assets\1587545069159.png)



容器、镜像父镜像：

![1587545094779](D:\学习资料\笔记\linux\assets\1587545094779.png)

docker 命令是最常使用的 docker 客户端命令，其后面可以加不同的参数以实现响应的功能，常用的命令如下：

### 1.4.1 搜索镜像

在官方的 docker 仓库中搜索指定名称的 docker 镜像，也会有很多三方镜像。

```bash
[root@docker-server1 ~]$ docker search centos:7.2.1511 #带指定版本号
[root@docker-server1 ~]$ docker search centos #不带版本号默认 latest

[root@ansible_ docker]$ docker search centos
NAME                               DESCRIPTION                                     STARS               OFFICIAL            AUTOMATED
centos                             The official build of CentOS.                   5953                [OK]         
ansible/centos7-ansible            Ansible on Centos7                              128                                     [OK]
jdeathe/centos-ssh                 OpenSSH / Supervisor / EPEL/IUS/SCL Repos - …   114                                     [OK]
consol/centos-xfce-vnc             Centos container with "headless" VNC session…   114                                     [OK]
centos/mysql-57-centos7            MySQL 5.7 SQL database server                   74                               
imagine10255/centos6-lnmp-php56    centos6-lnmp-php56                              58                                      [OK]
tutum/centos                       Simple CentOS docker image with SSH access      45                               
centos/postgresql-96-centos7       PostgreSQL is an advanced Object-Relational …   43                               
kinogmt/centos-ssh                 CentOS with SSH                                 29                                      [OK]
pivotaldata/centos-gpdb-dev        CentOS image for GPDB development. Tag names…   11                               
guyton/centos6                     From official centos6 container with full up…   10                                      [OK]
centos/tools                       Docker image that has systems administration…   6                                       [OK]
drecom/centos-ruby                 centos ruby                                     6                                       [OK]
pivotaldata/centos                 Base centos, freshened up a little with a Do…   4                                
pivotaldata/centos-gcc-toolchain   CentOS with a toolchain, but unaffiliated wi…   3                                
darksheer/centos                   Base Centos Image -- Updated hourly             3                                       [OK]
mamohr/centos-java                 Oracle Java 8 Docker image based on Centos 7    3                                       [OK]
pivotaldata/centos-mingw           Using the mingw toolchain to cross-compile t…   3                                
miko2u/centos6                     CentOS6 日本語環境                                   2                                       [OK]
blacklabelops/centos               CentOS Base Image! Built and Updates Daily!     1                                       [OK]
mcnaughton/centos-base             centos base image                               1                                       [OK]
indigo/centos-maven                Vanilla CentOS 7 with Oracle Java Developmen…   1                                       [OK]
pivotaldata/centos7-dev            CentosOS 7 image for GPDB development           0                                
pivotaldata/centos6.8-dev          CentosOS 6.8 image for GPDB development         0                                
smartentry/centos                  centos with smartentry                          0                                       [OK]

```

### 1.4.2 下载镜像

从 docker 仓库将镜像下载到本地，命令格式如下：

```bash
# docker pull 仓库服务器:端口/项目名称/镜像名称:tag(版本)号

[root@docker-server1 ~]$ docker pull alpine

[root@ansible_ docker]$ docker pull alpine
Using default tag: latest
latest: Pulling from library/alpine
aad63a933944: Pull complete
Digest: sha256:b276d875eeed9c7d3f1cfa7edb06b22ed22b14219a7d67c52c56612330348239
Status: Downloaded newer image for alpine:latest
```

### 1.4.3 查看本地镜像

下载完成的镜像比下载的大，因为下载完成后会解压缩

```bash
[root@k8s-6 ~]$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
alpine              latest              a187dde48cd2        4 weeks ago         5.6MB

REPOSITORY #镜像所属的仓库名称
TAG #镜像版本号（标识符），默认为 latest
IMAGE ID #镜像唯一 ID 标示
CREATED #镜像创建时间
VIRTUAL SIZE #镜像的大小
```

### 1.4.4 镜像导出

可以将镜像从本地导出为一个压缩文件，然后复制到其他服务器进行导入使用。

```bash
#导出方法 1：
[root@docker-server1 ~]$ docker save centos -o /opt/centos.tar.gz
[root@docker-server1 ~]$ ll /opt/centos.tar.gz
-rw------- 1 root root 205225472 Nov 1 03:52 /opt/centos.tar.gz

#导出方法 2：
[root@docker-server1 ~]$ docker save centos > /opt/centos-1.tar.gz
[root@docker-server1 ~]$ ll /opt/centos-1.tar.gz
-rw-r--r-- 1 root root 205225472 Nov 1 03:52 /opt/centos-1.tar.gz

#查看镜像内容：
[root@docker-server1 ~]$ cd /opt/
[root@docker-server1 opt]$ tar xvf centos.tar.gz
[root@docker-server1 opt]$ cat manifest.json #包含了镜像的相关配置，配置文件、分层
[{"Config":"196e0ce0c9fbb31da595b893dd39bc9fd4aa78a474bbdc21459a3ebe855b
7768.json","RepoTags":["docker.io/centos:latest"],"Layers":["892ebb5d1299cbf459f6
7aa070f29fdc6d83f40
25c58c090e9a69bd4f7af436b/layer.tar"]}]

#分层为了方便文件的共用，即相同的文件可以共用
[{"Config":" 配 置 文 件 .json","RepoTags":["docker.io/nginx:latest"],"Layers":[" 分层1/layer.tar","分层 2 /layer.tar","分层 3 /layer.tar"]}]
```

### 1.4.5 镜像导入

将镜像导入到 docker

```bash
[root@ansible_ opt]$ ansible 10.210.3.236 -m copy -a 'src=/opt/pause.tar.gz dest=/opt/'
[root@k8s-6 ~]$ docker load < /opt/pause.tar.gz
5f70bf18a086: Loading layer [==================================================>]  1.024kB/1.024kB
41ff149e94f2: Loading layer [==================================================>]  748.5kB/748.5kB
Loaded image: registry.cn-hangzhou.aliyuncs.com/google-containers/pause-amd64:3.0

[root@k8s-6 ~]$ docker images
REPOSITORY                                                        TAG                 IMAGE ID            CREATED             SIZE
alpine                                                            latest              a187dde48cd2        4 weeks ago         5.6MB
registry.cn-hangzhou.aliyuncs.com/google-containers/pause-amd64   3.0                 99e59f495ffa        3 years ago         747kB
```

### 1.4.6 docker 命令总结

```bash
docker rmi centos #删除镜像
docker load -i centos-latest.tar.xz #导入本地镜像
docker save > /opt/centos.tar #centos #导出镜像
docker rmi 镜像 ID/镜像名称 #删除指定 ID 的镜像，通过镜像启动容器的时候镜像不能被删除，除非将容器全部关闭
docker rm 容器 ID/容器名称 #删除容器
docker rm 容器 ID/容器名 -f #强制删除正在运行的容器
```



## 1.5 容器操作基础命令

```bash
#命令格式：
docker run [选项] [镜像名] [shell 命令] [参数]
docker run [参数选项] [镜像名称，必须在所有选项的后面] [/bin/echo 'hello wold'] #单次执行，没有自定义容器名称
docker run centos /bin/echo 'hello wold' #启动的容器在执行完 shel 命令就退出了
```

### 1.5.1 从镜像启动一个容器

```bash
#会直接进入到容器，并随机生成容器 ID 和名称
[root@docker-server1 ~]$ docker run -it docker.io/centos bash
[root@11445b3a84d3 /]$

#退出容器不注销
ctrl+p+q
```

### 1.5.2 显示正在运行的容器

```bash
[root@linux-docker ~]$ docker ps
```

![1587567710762](D:\学习资料\笔记\linux\assets\1587567710762.png)

### 1.5.3 显示所有容器

```bash
包括当前正在运行以及已经关闭的所有容器：
[root@linux-docker ~]$ docker ps -a
```

### 1.5.4 删除运行中的容器

```bash
即使容正在运行当中，也会被强制删除掉
[root@docker-server1 ~]$ docker rm -f 11445b3a84d3
```

![1587567804949](D:\学习资料\笔记\linux\assets\1587567804949.png)

### 1.5.5 随机映射端口

```bash
[root@docker-server1 ~]$ docker pull nginx #下载 nginx 镜像
[root@docker-server1 ~]$ docker run -P docker.io/nginx #前台启动并随机映射本地端口到容器的 80
#前台启动的会话窗口无法进行其他操作，除非退出，但是退出后容器也会退出

```

![1587567871620](D:\学习资料\笔记\linux\assets\1587567871620.png)

随机端口映射，其实是默认从 32768 开始

![1587567935599](D:\学习资料\笔记\linux\assets\1587567935599.png)

访问端口：

![1587567977362](D:\学习资料\笔记\linux\assets\1587567977362.png)

### 1.5.6 指定端口映射

```bash
#方式 1：本地端口 81 映射到容器 80 端口：
docker run -p 81:80 --name nginx-test-port1 nginx

#方式 2：本地 IP:本地端口:容器端口
docker run -p 192.168.10.205:82:80 --name nginx-test-port2 docker.io/nginx

#方式 3：本地 IP:本地随机端口:容器端口
docker run -p 192.168.10.205::80 --name nginx-test-port3 docker.io/nginx

#方式 4：本机 ip:本地端口:容器端口/协议，默认为 tcp 协议
docker run -p 192.168.10.205:83:80/udp --name nginx-test-port4 docker.io/nginx

#方式 5：一次性映射多个端口+协议：
docker run -p 86:80/tcp -p 443:443/tcp -p 53:53/udp --name nginx-test-port5 docker.io/nginx

#查看 Nginx 容器访问日志：
[root@docker-server1 ~]$ docker logs nginx-test-port3     #一次查看
[root@docker-server1 ~]$ docker logs -f nginx-test-port3  #持续查看
```

### 1.5.7 查看容器已经映射的端口

```bash
[root@docker-server1 ~]$ docker port nginx-test-port5
```

![1587568439910](D:\学习资料\笔记\linux\assets\1587568439910.png)

### 1.5.8 自定义容器名称

```bash
[root@docker-server1 ~]$ docker run -it --name nginx-test nginx
```

![1587568550534](D:\学习资料\笔记\linux\assets\1587568550534.png)

### 1.5.9 后台启动容器

```bash
[root@docker-server1 ~]$ docker run -d -P --name nginx-test1 docker.io/nginx 
9aaad776850bc06f516a770d42698e3b8f4ccce30d4b142f102ed3cb34399b31
```

![1587568658817](D:\学习资料\笔记\linux\assets\1587568658817.png)

### 1.5.10 创建并进入容器

```bash
[root@docker-server1 ~]$ docker run -t -i --name test-centos2 docker.io/centos /bin/bash
[root@a8fb69e71c73 /]$ #创建容器后直接进入，执行 exit 退出后容器关闭
```

### 1.5.11 单次运行

```bash
#容器退出后自动删除：
[root@linux-docker  opt]$  docker  run  -it  --rm  --name  nginx-delete-test docker.io/nginx
```

### 1.5.12 传递运行命令

容器需要有一个前台运行的进程才能保持容器的运行，通过传递运行参数是一种方式，另外也可以在构建镜像的时候指定容器启动时运行的前台命令。

```bash
[root@docker-server1 ~]$ docker run -d centos /usr/bin/tail -f '/etc/hosts'
```

![1587569005479](D:\学习资料\笔记\linux\assets\1587569005479.png)

### 1.5.13 容器的启动和关闭

```bash
[root@docker-server1 ~]$ docker stop f821d0cd5a99
[root@docker-server1 ~]$ docker start f821d0cd5a99
```

### 1.5.14 进入到正在运行的容器

#### 1.5.14.1 使用 attach 命令

使用方式为 docker attach 容器名，attach 类似于 vnc，操作会在各个容器界面显示，所有使用此方式进入容器的操作都是同步显示的且 exit 后容器将被关闭，且使用 exit 退出后容器关闭，不推荐使用，需要进入到有 shell 环境的容器。

```bash
[root@s1 ~]$ docker run -it centos bash
[root@63fbc2d5a3ec /]$
[root@s1 ~]$ docker attach 63fbc2d5a3ec
[root@63fbc2d5a3ec /]$
```

#### 1.5.14.2 使用 exec 命令

执行单次命令与进入容器，不是很推荐此方式，虽然 exit 退出容器还在运行

![1587569257318](D:\学习资料\笔记\linux\assets\1587569257318.png)

#### 1.5.14.3 使用 nsenter 命令

推荐使用此方式，nsenter 命令需要通过 PID 进入到容器内部，不过可以使用 docker inspect 获取到容器的 PID:

```bash
[root@docker-server1 ~]$ yum install util-linux  #安装 nsenter 命令
[root@docker-server1 ~]$ docker inspect -f "{{.NetworkSettings.IPAddress}}"
91fc190cb538
172.17.0.2
[root@docker-server1 ~]$ docker inspect -f "{{.State.Pid}}" mydocker #获取到某个docker 容器的 PID，可以通过 PID 进入到容器内
[root@docker-server1 ~]$ docker inspect -f "{{.State.Pid}}" centos-test3
5892
[root@docker-server1 ~]$ nsenter -t 5892 -m -u -i -n -p
[root@66f511bb15af /]$ ls
```

#### 1.5.14.4 脚本方式

```bash
#将 nsenter 命令写入到脚本进行调用，如下：
[root@docker-server1 ~]$ cat docker-in.sh
#!/bin/bash
docker_in(){
NAME_ID=$1
PID=$(docker inspect -f "{{.State.Pid}}" ${NAME_ID})
nsenter -t ${PID} -m -u -i -n -p
}
docker_in $1

#测试脚本是否可以正常进入到容器且退出后仍然正常运行：
[root@docker-server1 ~]$ chmod a+x docker-in.sh
[root@docker-server1 ~]$ ./docker-in.sh centos-test3
[root@66f511bb15af /]$ pwd
/
[root@66f511bb15af /]$ exit
logout
[root@docker-server1 ~]$ ./docker-in.sh centos-test3
[root@66f511bb15af /]$ exit
Logout
```

### 1.5.15 查看容器内部的 hosts 文件

```bash
[root@docker-server1 ~]$ docker run -i -t --name test-centos3 docker.io/centos
/bin/bash
[root@056bb4928b64 /]$ cat /etc/hosts
127.0.0.1  localhost
::1 localhost ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
172.17.0.4 056bb4928b64 #默认会将实例的 ID 添加到自己的 hosts 文件
```

ping 容器 ID：

![1587569792727](D:\学习资料\笔记\linux\assets\1587569792727.png)

### 1.5.16 批量关闭正在运行的容器

```bash
[root@docker-server1 ~]$ docker stop $(docker ps -a -q) #正常关闭所有运行中的容器
```

![1587569912432](D:\学习资料\笔记\linux\assets\1587569912432.png)

### 1.5.17 批量强制关闭正在运行的容器

```bash
[root@docker-server1 ~]$ docker kill $(docker ps -a -q) #强制关闭所有运行中的容器
```

![1587569885378](D:\学习资料\笔记\linux\assets\1587569885378.png)

### 1.5.18 批量删除已退出容器

```bash
[root@docker-server1 ~]$ docker rm -f `docker ps -aq -f status=exited`
```

![1587569996558](D:\学习资料\笔记\linux\assets\1587569996558.png)

### 1.5.19 批量删除所有容器

```bash
[root@docker-server1 ~]$ docker rm -f $(docker ps -a -q)
```

### 1.5.20 指定容器 DNS

Dns 服务，默认采用宿主机的 dns 地址

一是将 dns 地址配置在宿主机

二是将参数配置在 docker 启动脚本里面 –dns=1.1.1.1

```bash
[root@docker-server1 ~]$ docker run -it --rm --dns 223.6.6.6 centos bash
[root@afeb628bf074 /]$ cat /etc/resolv.conf
nameserver 223.6.6.6
```



# 2 Docker 镜像与制作

**Docker 镜像有没有内核？**

从镜像大小上面来说，一个比较小的镜像只有十几 MB，而内核文件需要一百多兆， 因此镜像里面是没有内核的，镜像在被启动为容器后将直接使用宿主机的内核，而镜像本身则只提供相应的 rootfs，即系统正常运行所必须的用户空间的文件系统，比如/dev/，/proc，/bin，/etc 等目录，所以容器当中基本是没有/boot目录的，而/boot 当中保存的就是与内核相关的文件和目录。

![1587628710449](D:\学习资料\笔记\linux\assets\1587628710449.png)

**为什么没有内核？**

由于容器启动和运行过程中是直接使用了宿主机的内核，所以没有直接调用过物理硬件，所以也不会涉及到硬件驱动，因此也用不上内核和驱动，另外有内核的那是虚拟机。



## 2.1 手动制作 yum 版 nginx 镜像



Docker 制作类似于虚拟机的镜像制作，即按照公司的实际业务务求将需要安装的软件、相关配置等基础环境配置完成，然后将其做成镜像，最后再批量从镜像批量生产实例，这样可以极大的简化相同环境的部署工作，Docker 的镜像制作分为手动制作和自动制作(基于 DockerFile)，其中手动制作镜像步骤具体如下：



### 2.1.1 下载镜像并初始化系统

基于某个基础镜像之上重新制作，因此需要先有一个基础镜像，本次使用官方提供的 centos 镜像为基础：

```bash
[root@docker-server1 ~]$ docker pull centos
[root@docker-server1 ~]$ docker run -it docker.io/centos /bin/bash
[root@37220e5c8410 /]$ yum install wget -y
[root@37220e5c8410 /]$ cd /etc/yum.repos.d/
[root@37220e5c8410 yum.repos.d]$ rm -rf ./* #更改 yum 源
[root@37220e5c8410 yum.repos.d]$ wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
[root@37220e5c8410  yum.repos.d]$  wget  -O  /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
```



### 2.1.2 yum 安装并配置 nginx

```bash
[root@37220e5c8410 yum.repos.d]$ yum install nginx –y #yum 安装 nginx
[root@37220e5c8410 yum.repos.d]$ yum install -y vim wget pcre pcre-devel zlib zlib-devel openssl openssl-devel iproute net-tools iotop #安装常用命令
```



### 2.1.3 关闭 nginx 后台运行

```bash
[root@37220e5c8410 yum.repos.d]$ vim /etc/nginx/nginx.conf #关闭 nginx 后台运行
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;
daemon off; #关闭后台运行
```



#### 2.1.4 自定义 web 页面

```bash
[root@37220e5c8410 yum.repos.d]$ vim /usr/share/nginx/html/index.html
[root@37220e5c8410 yum.repos.d]$ cat /usr/share/nginx/html/index.html
Docker Yum Nginx #自定义 web 界面
```



### 2.1.5 提交为镜像

在宿主机基于容器 ID 提交为镜像

```bash
[root@37220e5c8410 yum.repos.d]$ vim /usr/share/nginx/html/index.html
[root@37220e5c8410 yum.repos.d]$ cat /usr/share/nginx/html/index.html
Docker Yum Nginx #自定义 web 界面
```



### 2.1.6 带 tag 的镜像提交

提交的时候标记 tag 号：

```bash
#标记 tag 号，生产当中比较长用，后期可以根据 tag 标记启动不同版本启动 image 启动
[root@docker-server1 ~]$ docker commit -m "nginx image" f5f8c13d0f9f jack/centos-nginx:v1
sha256:ab9759679eb586f06e315971d28b88f0cd3e0895d2e162524ee21786b98b24e8
```



### 2.1.7 从自己镜像启动容器

```bash
[root@docker-server1 ~]$ docker run -d -p 80:80 --name my-centos-nginx jack/centos-nginx /usr/sbin/nginx
ce4ee8732a0c4c6a10b85f5463396b27ba3ed120b27f2f19670fdff3bf5cdb62
```

![1587633961273](D:\学习资料\笔记\linux\assets\1587633961273.png)



### 2.1.8 访问测试

![1587633983512](D:\学习资料\笔记\linux\assets\1587633983512.png)

![1587634044359](D:\学习资料\笔记\linux\assets\1587634044359.png)



## 2.2 DockerFile 制作 yum 版 nginx 镜像

DockerFile 可以说是一种可以被 Docker 程序解释的脚本，DockerFile 是由一条条的命令组成的，每条命令对应 linux 下面的一条命令，Docker 程序将这些DockerFile 指令再翻译成真正的 linux 命令，其有自己的书写方式和支持的命令，Docker 程序读取 DockerFile 并根据指令生成 Docker 镜像，相比手动制作镜像的方式，DockerFile 更能直观的展示镜像是怎么产生的，有了 DockerFile，当后期有额外的需求时，只要在之前的 DockerFile 添加或者修改响应的命令即可重新生成新的 Docke 镜像，避免了重复手动制作镜像的麻烦，具体如下：



### 2.1.1 下载镜像并初始化系统

```bash
[root@docker-server1 ~]$ docker pull centos
[root@docker-server1 ~]$ docker run -it docker.io/centos /bin/bash
[root@docker-server1 ~]$ cd /opt/ #创建目录环境
[root@docker-server1 opt]$ mkdir
dockerfile/{web/{nginx,tomcat,jdk,apache},system/{centos,ubuntu,redhat}} -pv # 目录结构按照业务类型 或系统类型等方式划分,方便后期镜像比较多的时候进行分类。

[root@docker-server1 opt]$ cd dockerfile/web/nginx/
[root@docker-server1 nginx]$ pwd
/opt/dockerfile/web/nginx
```



### 2.2.2 编写 Dockerfile

**Dockerfile 指令**

```bash
vim ./Dockerfile #生成的镜像的时候会在执行命令的当前目录查找 Dockerfile 文件，所以名称不可写错，而且 D 必须大写
FROM #构建的新镜像是基于哪个镜像 例如：FROM centos:7.6.1810
MAINTAINER #镜像维护者的信息
RUN #构建镜像时运行的Shell命令 例如：RUN yum install httpd
CMD #运行容器时执行的Shell命令 例如：CMD /usr/sbin/sshd –D
EXPOSE #声明容器运行的服务端口 例如：EXPOSE 80 443
ENV #设置容器内环境变量 例如：ENV MYSQL_ROOT_PASSWORD 123456
ADD #拷贝文件或目录到镜像，如果是URL或压缩包会自动下载或自动解压 ADD html.tar.gz /var/www/html
COPY #拷贝文件或目录到镜像，用法同上 例如：COPY ./start.sh /start.sh
ENTRYPOINT #运行容器时执行的Shell命令 例如：ENTRYPOINT /bin/bash -c ‘/start.sh’
VOLUME #指定容器挂载点到宿主机自动生成的目录或其他容器 例如：VOLUME ["/var/lib/mysql"]
USER #为RUN、CMD和ENTRYPOINT执行命令指定运行用户 例如：USER www
WORKDIR #为RUN、CMD、ENTRYPOINT、COPY和ADD设置工作目录 例如：WORKDIR /data
HEALTHCHECK #健康检查HEALTHCHECK --interval=5m --timeout=3s --retries=3 CMD curl -f http://localhost/ || exit 1
ARG #在构建镜像时指定一些参数 例如：docker build --build-arg user=www Dockerfile .
```

**nginx Dockerfile**

```bash
[root@docker-server1 nginx]$ vim ./Dockerfile 
From centos:7.6.1810
MAINTAINER Jack.Zhang 123456@qq.com 

RUN rpm -ivh http://mirrors.aliyun.com/epel/epel-release-latest-7.noarch.rpm
RUN yum install -y vim wget tree lrzsz gcc gcc-c++ automake pcre pcre-devel zlib zlib-devel openssl openssl-devel iproute net-tools iotop
ADD nginx-1.14.2.tar.gz /usr/local/src/ 
RUN cd /usr/local/src/nginx-1.14.2 && ./configure --prefix=/usr/local/nginx --with-http_sub_module && make && make install
RUN cd /usr/local/nginx/
ADD nginx.conf /usr/local/nginx/conf/nginx.conf
RUN useradd nginx -s /sbin/nologin
RUN ln -sv /usr/local/nginx/sbin/nginx /usr/sbin/nginx
RUN echo "test nginx page" > /usr/local/nginx/html/index.html
EXPOSE 80 443 
CMD ["nginx"] 

#如果在从该镜像启动容器的时候也指定了命令，那么指定的命令会覆盖Dockerfile 构建的镜像里面的 CMD 命令，即指定的命令优先级更高，Dockerfile 的优先级较低一些
```



### 2.2.3 准备源码包与配置文件

```bash
[root@docker-server1 nginx]$ cp /usr/local/nginx/conf/nginx.conf . #配置文件关闭后台运行
[root@docker-server1 nginx]$ cp /usr/local/src/nginx-1.14.2.tar.gz . #nginx 源码包
```



### 2.2.4 执行镜像构建

```bash
[root@docker-server1  nginx]$  docker  build  –t  jack/nginx-1.14.2:v1 /opt/dockerfile/web/nginx/
```

![1587636048257](D:\学习资料\笔记\linux\assets\1587636048257.png)



### 2.2.5 构建完成

可以清晰看到各个步骤执行的具体操作

![1587636096438](D:\学习资料\笔记\linux\assets\1587636096438.png)

### 2.2.6 查看是否生成本地镜像

![1587636132620](D:\学习资料\笔记\linux\assets\1587636132620.png)

### 2.2.7 从镜像启动容器

```bash
[root@docker-server1 nginx]$ docker run -d -p 80:80 --name yum-nginx jack/nginx-1.14.2:v1 /usr/sbin/nginx
```

![1587650231172](D:\学习资料\笔记\linux\assets\1587650231172.png)

### 2.2.8 访问 web 界面

![1587650316774](D:\学习资料\笔记\linux\assets\1587650316774.png)

### 2.2.9 编译过程中使用过的镜像

![1587650351249](D:\学习资料\笔记\linux\assets\1587650351249.png)



## 2.3 手动制作编译版本 nginx 镜像

过程为在 centos 基础镜像之上手动编译安装 nginx，然后再提交为镜像。



### 2.3.1 下载镜像并初始化系统

```bash
[root@docker-server1 ~]$ docker pull centos
[root@docker-server1 ~]$ docker run -it docker.io/centos /bin/bash
[root@86a48908bb97 /]$ yum install wget -y
[root@86a48908bb97 /]$ cd /etc/yum.repos.d/
[root@86a48908bb97 yum.repos.d]$ rm -rf ./* 
[root@86a48908bb97 yum.repos.d]$ wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
[root@86a48908bb97  yum.repos.d]$  wget  -O  /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
```

### 2.3.2 编译安装 nginx

```bash
[root@86a48908bb97 yum.repos.d]$ yum install -y vim wget tree lrzsz gcc gcc-c++
automake pcre pcre-devel zlib zlib-devel openssl openssl-devel iproute net-tools iotop
#安装基础包
[root@86a48908bb97 yum.repos.d]$ cd /usr/local/src/
[root@86a48908bb97 src]$ wget http://nginx.org/download/nginx-1.14.2.tar.gz
[root@86a48908bb97 src]$ tar xvf nginx-1.14.2.tar.gz
[root@86a48908bb97 src]$ cd nginx-1.14.2
[root@86a48908bb97 nginx-1.14.2]$ ./configure --prefix=/usr/local/nginx --with-http_sub_module
[root@86a48908bb97 nginx-1.14.2]$ make && make install
[root@86a48908bb97 nginx-1.14.2]$ cd /usr/local/nginx/
```

### 2.3.3 关闭 nginx 后台运行

```bash
[root@86a48908bb97 nginx]$ vim conf/nginx.conf
user nginx;
worker_processes auto;
daemon off;
[root@86a48908bb97 nginx]$ ln -sv /usr/local/nginx/sbin/nginx /usr/sbin/nginx #创建软连
```

### 2.3.4 创建用户及授权

```bash
[root@86a48908bb97 nginx]$ useradd nginx -s /sbin/nologin
[root@86a48908bb97 nginx]$ chown nginx.nginx /usr/local/nginx/ -R
```

### 2.3.5 自定义 web 界面

```bash
[root@86a48908bb97  nginx]$  echo  "My  Nginx  Test  Page"  > /usr/local/nginx/html/index.html
```

### 2.3.6 提交为镜像

```bash
[root@docker-server1 ~]$ docker commit -m "test nginx" 86a48908bb97 jack/nginx-test-image
sha256:fce6e69410e58b8e508c7ffd2c5ff91e59a1144847613f691fa5e80bb68efbfa
```

![1587655315342](D:\学习资料\笔记\linux\assets\1587655315342.png)

```bash
[root@docker-server1 ~]$ docker commit -m "test nginx" 86a48908bb97 jack/nginx-test-image:v1
sha256:474cad22f28b1e6b17898d87f040dc8d1f3882e2f4425c5f21599849a3d3c6a2
```

### 2.3.7：从自己的镜像启动容器：

```bash
[root@docker-server1 ~]$ docker run -d -p 80:80 --name my-centos-nginx jack/nginx-test-image:v1 /usr/sbin/nginx
8042aedec1d6412a79ac226c9289305087fc062b0087955a3a0a609c891e1122
```

![1587655412493](D:\学习资料\笔记\linux\assets\1587655412493.png)

```
备注： -name 是指定容器的名称，-d 是后台运行，-p 是端口映射，jack/nginx-test-image 是 xx 仓库下的 xx 镜像的 xx 版本，可以不加版本，不加版本默认是使用 latest，最后面的 nginx 是运行的命令，即镜像里面要运行一个 nginx 命令，所以才有了前面将/usr/local/nginx/sbin/nginx 软连接到/usr/sbin/nginx，目的就是为了让系统可以执行此命令。
```

### 2.3.8 访问测试

![1587655530411](D:\学习资料\笔记\linux\assets\1587655530411.png)

### 2.3.9 查看 Nginx 访问日志

![1587655567742](D:\学习资料\笔记\linux\assets\1587655567742.png)



## 2.4 自定义 tomcat 镜像

基于官方提供的 centos 7.6.1810 基础镜像构建 JDK 和 tomcat 镜像，先构建JDK 镜像，然后再基于 JDK 镜像构建 tomcat 镜像。



### 2.4.1 构建 JDK 镜像

#### 2.4.1.1 下载基础镜像 Centos

```bash
[root@docker-server1 ~]$ docker pull centos:7
```

#### 2.4.1.2 执行构建 JDK 镜像

```bash
[root@docker-server1 ~]$ mkdir /opt/dockerfile/{web/{nginx,tomcat,jdk,apache},system/{centos,ubuntu,redhat}} -pv
[root@docker-server1 ~]$ cd /opt/dockerfile/web/jdk/
[root@ansible_ 8u92]$ cat Dockerfile 
FROM  centos:7.6.1810

ADD jdk-8u192-linux-x64.tar.gz /usr/local/src  

RUN ln -sv /usr/local/src/jdk1.8.0_192 /usr/local/jdk
ADD profile /etc/profile
#&&  ln -sv /usr/local/src/jdk1.8.0_192  /usr/local/jdk 

ENV JAVA_HOME /usr/local/jdk
ENV JRE_HOME $JAVA_HOME/jre
ENV CLASSPATH $JAVA_HOME/lib/:$JRE_HOME/lib/
ENV PATH $PATH:$JAVA_HOME/bin
RUN rm -rf /etc/localtime && ln -snf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && echo "Asia/Shanghai" > /etc/timezone
```

#### 2.4.1.4 执行构建镜像

##### 2.4.1.4.1 通过脚本构建

```bash
[root@ansible_ 8u92]$ cat build-command.sh 
#!/bin/bash
docker build -t jdk-base:1.8.0.192 .

```

##### 2.4.1.4.2 执行构建

![1587656282892](D:\学习资料\笔记\linux\assets\1587656282892.png)

#### 2.4.1.5 镜像构建完成

![1587656333187](D:\学习资料\笔记\linux\assets\1587656333187.png)

#### 2.4.1.6 从镜像启动容器

```bash
[root@ansible_ 8u92]$ docker run -it jdk-base:1.8.0.192 bash
[root@17ebbebdf15b /]$ java -version
java version "1.8.0_192"
Java(TM) SE Runtime Environment (build 1.8.0_192-b12)
Java HotSpot(TM) 64-Bit Server VM (build 25.192-b12, mixed mode)
```

![1587656382159](D:\学习资料\笔记\linux\assets\1587656382159.png)

#### 2.4.1.7 将镜像上传到 harbor

```bash
[root@docker-server1 jdk]$ docker push 192.168.10.205/centos/jdk-base:1.8.0.192
```

#### 2.4.1.8 镜像仓库验证

![1587656506711](D:\学习资料\笔记\linux\assets\1587656506711.png)

#### 2.4.1.9 从其他 docker 客户端下载镜像并启动 JDK 容器

启动的时候本地没有镜像，会从仓库下载，然后从镜像启动容器

![1587656563886](D:\学习资料\笔记\linux\assets\1587656563886.png)

### 2.4.2 编辑 Dockerfile

```bash
[root@ansible_ tomcat-base]$ pwd
/opt/dockerfile/web/tomcat/tomcat-base

[root@ansible_ tomcat-base]$ cat Dockerfile 
FROM jdk-base:1.8.0.192

MAINTAINER 2973707860@qq.com

ADD  apache-tomcat-8.5.37.tar.gz /apps  
RUN ln -sv /apps/apache-tomcat-8.5.37 /apps/tomcat 
```

#### 2.4.2.2 上传 tomcat 压缩包

```bash
[root@ansible_ tomcat-base]$ ll
total 9436
-rw-r--r-- 1 root root 9653382 Apr 26 15:40 apache-tomcat-8.5.37.tar.gz
-rwxr-xr-x 1 root root      49 Apr 26 15:40 build-command.sh
-rw-r--r-- 1 root root     148 Apr 26 15:40 Dockerfile
```

#### 2.4.2.3 通过脚本构建 tomcat 基础镜像

```bash
[root@ansible_ tomcat-base]$ cat build-command.sh 
#!/bin/bash
docker build -t tomcat-base:8.5.37 .
```

执行构建：

![1587656864112](D:\学习资料\笔记\linux\assets\1587656864112.png)

#### 2.4.2.4 验证镜像构建完成

```bash
[root@ansible_ tomcat-base]$ docker images
REPOSITORY        TAG                 IMAGE ID            CREATED             SIZE
tomcat-base      8.5.37              b75fa3f30c84        3 minutes ago       612MB
jdk-base         1.8.0.192           570352219c32        30 minutes ago      598MB
centos           7.6.1810            f1cb7c7d58b7        13 months ago       202MB
```



### 2.4.3 构建业务镜像 1

创建 tomact-app1 和 tomact-app2 两个目录，代表不同的两个基于 tomact 的业务。

#### 2.4.3.1 准备 Dockerfile

```bash
[root@ansible_ tomcat]$ ll tomcat-app1
total 32
-rw-r--r-- 1 root root  292 Apr 26 15:40 \
-rwxr-xr-x 1 root root   45 Apr 26 15:40 build-command.sh
-rw-r--r-- 1 root root  141 Apr 26 15:40 code.tar.gz
-rw-r--r-- 1 root root  320 Apr 26 15:40 Dockerfile
-rw-r--r-- 1 root root   21 Apr 26 15:40 index.html
-rwxr-xr-x 1 root root  199 Apr 26 15:40 run_tomcat.sh
-rw-r--r-- 1 root root 7513 Apr 26 15:40 server.xml

[root@ansible_ tomcat]$ cat tomcat-app1/Dockerfile 
FROM tomcat-base:8.5.37

maintainer 2973707860@qq.com

ADD code.tar.gz /data/tomcat/webapps/app1
ADD run_tomcat.sh /apps/tomcat/bin/run_tomcat.sh
ADD server.xml  /apps/tomcat/conf
RUN useradd -s /sbin/nologin tomcat
RUN chown -R  tomcat.tomcat /apps/apache-tomcat-8.5.37 /apps/tomcat  /data/tomcat
EXPOSE 8080 8443

CMD ["/apps/tomcat/bin/run_tomcat.sh"]
```

#### 2.4.3.2 准备自定义 index.html 页面

```bash
[root@ansible_ tomcat-app1]$ cat index.html 
tomcat web app1 page
```

#### 2.4.3.3 准备容器启动执行脚本

```bash
[root@ansible_ tomcat-app1]$ cat run_tomcat.sh 
#!/bin/bash
source /etc/profile
echo "1.1.1.1 www.magedu.net" >> /etc/hosts
su - tomcat -c "/apps/tomcat/bin/catalina.sh start"
#su - tomcat -c "/apps/tomcat/bin/catalina.sh  run"
tail -f /etc/hosts
```

#### 2.4.3.4 准备构建脚本

```bash
[root@docker-server1 tomcat-app1]$ cat build-command.sh
#!/bin/bash
docker build -t tomcat-web:app1 .
```

#### 2.4.3.5 执行构建

![1587720607711](D:\学习资料\笔记\linux\assets\1587720607711.png)

#### 2.4.3.6 从镜像启动容器测试

```bash
[root@docker-server1 tomcat-app1]$ docker run -it -d -p 8888:8080 tomcat-web:app1
d03fd22cce891a29b590715bde9bd6fc314dd5ae3af178e8f48b215aeb878d9f
```

![1587720652086](D:\学习资料\笔记\linux\assets\1587720652086.png)

#### 2.4.3.7 访问测试

![1587720677901](D:\学习资料\笔记\linux\assets\1587720677901.png)

### 2.4.4 构建业务镜像2：

#### 2.4.4.1 准备 Dockerfile

```bash
[root@docker-server1 tomcat-app2]$ pwd
/opt/dockerfile/system/centos/tomcat-app2

[root@docker-server1 tomcat-app2]$ cat Dockerfile
#Tomcat Web2 Image
FROM tomcat-base:8.5.37

ADD run_tomcat.sh /apps/tomcat/bin/run_tomcat.sh
ADD myapp/* /apps/tomcat/webapps/myapp/
RUN useradd -s /sbin/nologin tomcat && chown www.www /apps/ -R

CMD ["/apps/tomcat/bin/run_tomcat.sh"]
EXPOSE 8080 8009
```

#### 2.4.4.2 准备自定义页面

```bash
[root@docker-server1 tomcat-app2]$ mkdir myapp
[root@docker-server1 tomcat-app2]$ echo "Tomcat Web Page2" > myapp/index.html
[root@docker-server1 tomcat-app2]$ cat myapp/index.html
Tomcat Web Page2
```

#### 2.4.4.3 准备容器启动脚本

```bash
[root@docker-server1 tomcat-app2]$ cat run_tomcat.sh
#!/bin/bash
echo "1.1.1.1 abc.test.com" >> /etc/hosts
echo "nameserver 223.5.5.5" > /etc/resolv.conf

su - www -c "/apps/tomcat/bin/catalina.sh start"
su - www -c "tail -f /etc/hosts"
```

#### 2.4.4.4 准备构建脚本

```bash
[root@docker-server1 tomcat-app2]$ cat build-command.sh
#!/bin/bash
docker build -t tomcat-web:app2 .
```

#### 2.4.4.5 执行构建

![1587720959220](D:\学习资料\笔记\linux\assets\1587720959220.png)

#### 2.4.4.6 从镜像启动容器

```bash
[root@docker-server1 tomcat-app2]$ docker run -it -d -p 8889:8080 tomcat-web:app2
4846126780f6bdb374f7d4a7a91a16c4af211a9a01b11db629751a597dc55e08
```

#### 2.4.4.7 访问测试

![1587721025217](D:\学习资料\笔记\linux\assets\1587721025217.png)

## 2.5 构建 haproxy 镜像

### 2.5.1 准备 Dockerfile

```bash
[root@docker-server1 haproxy]$ pwd
/opt/dockerfile/system/centos/haproxy
[root@docker-server1 haproxy]$ cat Dockerfile
#Haproxy Base Image
FROM centos:7.6.1810
MAINTAINER linjingbin "linjingbin@139.com"
RUN yum install gcc gcc-c++ glibc glibc-devel pcre pcre-devel openssl openssl-devel systemd-devel net-tools vim iotop bc zip unzip zlib-devel lrzsz tree screen lsof tcpdump wget ntpdate –y
ADD haproxy-1.8.12.tar.gz /usr/local/src/
RUN cd /usr/local/src/haproxy-1.8.12 && make ARCH=x86_64 TARGET=linux2628 USE_PCRE=1 USE_OPENSSL=1 USE_ZLIB=1 USE_SYSTEMD=1 USE_CPU_AFFINITY=1 PREFIX=/usr/local/haproxy && make install PREFIX=/usr/local/haproxy && cp haproxy /usr/sbin/ && mkdir /usr/local/haproxy/run
ADD haproxy.cfg /etc/haproxy/
ADD run_haproxy.sh /usr/bin
EXPOSE 80 9999
CMD ["/usr/bin/run_haproxy.sh"]
```

### 2.5.2 准备 haproxy 源码文件

```bash
[root@docker-server1 haproxy]$ ll haproxy-1.8.12.tar.gz
-rw-r--r-- 1 root root 2059925 Jun 30 23:32 haproxy-1.8.12.tar.gz
```

### 2.5.3 准备 haproxy 配置文件

```bash
[root@docker-server1 haproxy]$ cat haproxy.cfg
global
chroot /usr/local/haproxy
#stats socket /var/lib/haproxy/haproxy.sock mode 600 level admin
uid 99
gid 99
daemon
nbproc 1
pidfile /usr/local/haproxy/run/haproxy.pid
log 127.0.0.1 local3 info
defaults
option http-keep-alive
option forwardfor
mode http
timeout connect 300000ms
timeout client 300000ms
timeout server 300000ms
listen stats
mode http
bind 0.0.0.0:9999
stats enable
log global
stats uri /haproxy-status
stats auth haadmin:q1w2e3r4ys
listen web_port
bind 0.0.0.0:80
mode http
log global
balance roundrobin
server web1 192.168.100.101:8888 check inter 3000 fall 2 rise 5
server web2 192.168.100.101:8889 check inter 3000 fall 2 rise 5
```

### 2.5.4 准备构建脚本

```bash
[root@docker-server1 haproxy]$ cat build-command.sh
#!/bin/bash
docker build -t centos-haproxy-base:7.6.1810 .
```

### 2.5.5 执行构建 haproxy 镜像

![1587909113021](D:\学习资料\笔记\linux\assets\1587909113021.png)

### 2.5.6 从镜像启动容器

```bash
[root@docker-server1 haproxy]$ docker run -it -d -p80:80 -p9999:9999 centos-haproxy-base:7.5-1.8.12
0b9bfdf14beee6008b259efc2d9d4fe74edee957af67da055f1dd18c5a44b4fb
```

### 2.5.7 web 访问验证

![1587909183759](D:\学习资料\笔记\linux\assets\1587909183759.png)

### 2.5.8 访问 haproxy 控制端

![1587909216976](D:\学习资料\笔记\linux\assets\1587909216976.png)

## 2.6 本地镜像上传至官方 docker 仓库

将自制的镜像上传至 docker 仓库；https://hub.docker.com/

### 2.6.1 准备账户

登录到 docker hub 创建官网创建账户，登录后点击 settings 完善账户信息：

![1587909318566](D:\学习资料\笔记\linux\assets\1587909318566.png)

### 2.6.2 填写账户基本信息

![1587909341016](D:\学习资料\笔记\linux\assets\1587909341016.png)

### 2.6.3 在虚拟机使用自己的账号登录

```bash
#登录阿里云Docker Registry
[root@ansible_ haproxy]$ docker login --username=timke774911 registry.cn-hangzhou.aliyuncs.com
Password: 
```

#### 2.6.4 查看认证信息

登录成功之后会在当前目录生成一个隐藏文件用于保存登录认证信息

```
[root@ansible_ haproxy]$ cat /root/.docker/config.json 
{
	"auths": {
		"registry.cn-hangzhou.aliyuncs.com": {
			"auth": "dGlta2U3NzQ5MTE6TnVtZW4uMjAxOA=="
		}
	},
	"HttpHeaders": {
		"User-Agent": "Docker-Client/18.09.9 (linux)"
	}
}
```

### 2.6.5 给镜像做 tag 并开始上传

```bash
[root@linux-docker ~]$ docker images #查看镜像 ID

#为镜像做标记
[root@ansible_ haproxy]$docker tag 570352219c32 registry.cn-hangzhou.aliyuncs.com/linjingbin/jdk-base:1.8.0.192

#上传至仓库
[root@ansible_ haproxy]$docker push registry.cn-hangzhou.aliyuncs.com/linjingbin/jdk-base:1.8.0.192
```

### 2.6.6 更换到其他 docker 服务器下载镜像

```bash
[root@k8s-6 ~]$docker login --username=timke774911 registry.cn-hangzhou.aliyuncs.com
Password: 

[root@k8s-6 ~]$docker pull registry.cn-hangzhou.aliyuncs.com/linjingbin/jdk-base:1.8.0.192

[root@k8s-6 ~]$ docker images 
REPOSITORY                                            TAG       IMAGE ID      CREATED         SIZE
registry.cn-hangzhou.aliyuncs.com/linjingbin/jdk-base 1.8.0.192 570352219c32  7 hours ago     598MB
```



# 3 Docker 数据管理

如果运行中的容器生成了新的数据或者修改了现有的一个已经存在的文件内容，那么新产生的数据将会被复制到读写层进行持久化保存，这个读写层也就是容器的工作目录，此即“写时复制(COW)”机制

![1587911488588](D:\学习资料\笔记\linux\assets\1587911488588.png)

### 3.1 数据类型

Docker 的镜像是分层设计的，底层是只读的，通过镜像启动的容器添加了一层可读写的文件系统，用户写入的数据都保存在这一层当中，如果要将写入的数据永久生效，需要将其提交为一个镜像然后通过这个镜像再启动实例，然后就会给这个启动的实例添加一层可读写的文件系统，目前 Docker 的数据类型分为两种，一是数据卷，二是数据容器，数据卷类似于挂载的一块磁盘，数据容器是将数据保存在一个容器上

![1587912705421](D:\学习资料\笔记\linux\assets\1587912705421.png)

![1587912731169](D:\学习资料\笔记\linux\assets\1587912731169.png)

![1587912750007](D:\学习资料\笔记\linux\assets\1587912750007.png)

```bash
root@s1:~# docker exec -it f55c55544e05 bash
[root@f55c55544e05 /]$ dd if=/dev/zero of=file bs=1M count=100
100+0 records in
100+0 records out
104857600 bytes (105 MB) copied, 0.221863 s, 473 MB/s
[root@f55c55544e05 /]$ md5sum file
2f282b84e7e608d5852449ed940bfc51 file
[root@f55c55544e05 /]$ cp anaconda-post.log /opt/anaconda-post.log
```

数据在哪？

![1587912878507](D:\学习资料\笔记\linux\assets\1587912878507.png)

### 3.1.1 什么是数据卷(data volume)

数据卷实际上就是宿主机上的目录或者是文件，可以被直接 mount 到容器当中使用。

实际生成环境中，需要针对不同类型的服务、不同类型的数据存储要求做相应的规划，最终保证服务的可扩展性、稳定性以及数据的安全性。

![1587913128518](D:\学习资料\笔记\linux\assets\1587913128518.png)

#### 3.1.1.1 创建 APP 目录并生成 web 页面

```bash
[root@docker-server1 ~]$ mkdir testapp
[root@docker-server1 ~]$ echo "Test App Page" > testapp/index.html
[root@docker-server1 ~]$ cat testapp/index.html
Test App Page
```

#### 3.1.1.2 启动容器并验证数据

注意使用-v 参数，将宿主机目录映射到容器内部，web2 的 ro 标示在容器内对该目录只读，默认是可读写的：

```bash
[root@docker-server1 ~]$ docker run -d --name web1 -v /root/testapp/:/apps/tomcat/webapps/testapp -p 8811:8080 tomcat-web:app1
d982cbb2524680f20f6edef7347abf5fbc1ac587b6de1a37b99ddc91fc26be76

[root@docker-server1 ~]$ docker run -d --name web2 -v /root/testapp/:/apps/tomcat/webapps/testapp:ro -p 8812:8080 tomcat-web:app2
46698f391a1c2ebf6e4f959ee0f9277ff2a301afb4b5ec3b61dbdd5717f33364
```

#### 3.1.1.3 进入到容器内测试写入数据

![1587913369346](D:\学习资料\笔记\linux\assets\1587913369346.png)

#### 3.1.1.4 宿主机验证

![1587913517001](D:\学习资料\笔记\linux\assets\1587913517001.png)

#### 3.1.1.5  web 界面访问

![1587913453942](D:\学习资料\笔记\linux\assets\1587913453942.png)

#### 3.1.1.6 在宿主机修改数据

```bash
[root@docker-server1 ~]$ echo "1111" >> testapp/index.html
[root@docker-server1 ~]$ cat testapp/index.html
Test App Page
docker
1111
```

#### 3.1.1.7 web 端访问验证数据

![1587913615042](D:\学习资料\笔记\linux\assets\1587913615042.png)

#### 3.1.1.8 删除容器

创建容器的时候指定参数-v，可以删除/var/lib/docker/containers/的容器数据目录，默认不删除，但是不能删除数据卷的内容，如下：

![1587913668798](D:\学习资料\笔记\linux\assets\1587913668798.png)

#### 3.1.1.9 验证宿主机的数据

![1587914134730](D:\学习资料\笔记\linux\assets\1587914134730.png)

#### 3.1.1.10 数据卷的特点及使用

1. 数据卷是目录或者文件，并且可以在多个容器之间共同使用。
2. 对数据卷更改数据容器里面会立即更新。
3. 数据卷的数据可以持久保存，即使删除使用使用该容器卷的容器也不影响。
4. 在容器里面的写入数据不会影响到镜像本身。

### 3.1.2 文件挂载

文件挂载用于很少更改文件内容的场景，比如 nginx 的配置文件、tomcat 的配置文件等。

#### 3.1.2.1 创建容器并挂载文件

```bash
[root@docker-server1 ~]$ ll testapp/catalina.sh
-rwxr-xr-x 1 root root 23705 Jul 3 14:06 testapp/catalina.sh

#自定义 JAVA 选项：
JAVA_OPTS="-server -Xms4g -Xmx4g -Xss512k -Xmn1g -
XX:CMSInitiatingOccupancyFraction=65 -XX:+UseFastAccessorMethods -
XX:+AggressiveOpts -XX:+UseBiasedLocking -XX:+DisableExplicitGC -
XX:MaxTenuringThreshold=10 -XX:NewSize=2048M -XX:MaxNewSize=2048M -
XX:NewRatio=2 -XX:PermSize=128m -XX:MaxPermSize=512m -
XX:CMSFullGCsBeforeCompaction=5 -XX:+ExplicitGCInvokesConcurrent -
XX:+UseConcMarkSweepGC -XX:+UseParNewGC -XX:+CMSParallelRemarkEnabled -
XX:+UseCMSCompactAtFullCollection -XX:LargePageSizeInBytes=128m -
XX:+UseFastAccessorMethods"

[root@docker-server1 ~]$ docker run -d --name web1 -v /root/bin/catalina.sh:/apps/tomcat/bin/catalina.sh:ro -p 8811:8080 tomcat-web:app1
ef0de0f5628f2137eca7dfb69fe86947029586beb517cd6077086f8feda2c40d
```

#### 3.1.2.2 验证参数生效

![1587914566010](D:\学习资料\笔记\linux\assets\1587914566010.png)

#### 3.1.2.3 进入容器测试文件读写

![1587914685426](D:\学习资料\笔记\linux\assets\1587914685426.png)

#### 3.1.2.4 如何一次挂载多个目录

多个目录要位于不同的目录下

```bash
[root@docker-server1 ~]$ docker run -d --name web1 -v /root/bin/catalina.sh:/apps/tomcat/bin/catalina.sh:ro -v /root/testapp:/apps/tomcat/webapps/testapp -p 8811:8080 tomcat-web:app1
9c7a503888f07a880db7a1ffe39044176a3bf0a32286c28f1aee9c3bd9ca567e

[root@docker-server1 ~]$
[root@docker-server1 ~]$ docker run -d --name web2 -v /root/bin/catalina.sh:/apps/tomcat/bin/catalina.sh:ro -v /root/testapp:/apps/tomcat/webapps/testapp -p 8812:8080 tomcat-web:app2
f63effc2a4e9b4869f1c8e8267dd1717cf846e6d672b68f474bf4b569aed2cfb
```

![1587914989657](D:\学习资料\笔记\linux\assets\1587914989657.png)

#### 3.1.2.5 数据卷使用场景

1. 日志输出
2. 静态 web 页面
3. 应用配置文件
4. 多容器间目录或文件共享

### 3.1.3 数据卷容器

数据卷容器最大的功能是可以让数据在多个 docker 容器之间共享，即可以让 B 容器访问 A 容器的内容，而容器 C 也可以访问 A 容器的内容，即先要创建一个后台运行的容器作为 Server，用于卷提供，这个卷可以为其他容器提供数据
存储服务，其他使用此卷的容器作为 client 端：

#### 3.1.3.1 启动一个卷容器 Server

先启动一个容器，并挂载宿主机的数据目录：

```bash
[root@docker-server1 ~]$ docker run -d --name volume-docker -v /root/bin/catalina.sh:/apps/tomcat/bin/catalina.sh:ro -v /root/testapp:/apps/tomcat/webapps/testapp tomcat-web:app2
aa56f415c4d27bdd68f5f88d5dd2e8ac499177a606cdd1dccc09079b59ee9a72
```

#### 3.1.3.2 启动端容器 Client

```bash
[root@docker-server1 ~]$ docker run -d --name web1 -p 8801:8080 --volumes-from volume-docker tomcat-web:app1
241e9559f2bc7a8453a0917c1be25d6649c30bf2d6ce929fd7e03b0dc4215543

[root@docker-server1 ~]$ docker run -d --name web2 -p 8802:8080 --volumes-from volume-docker tomcat-web:app2
82b728184355dfb9f9be6dfad51ebfa9d3429ba1133e708cb56273fb1a9b9794
```

#### 3.1.3.3 进入容器测试读写

读写权限依赖于源数据卷 Server 容器

![1587915550186](D:\学习资料\笔记\linux\assets\1587915550186.png)

#### 3.1.3.4 关闭卷容器 Server 测试能否启动新容器

```bash
[root@docker-server1 ~]$ docker stop volume-docker
volume-docker

[root@docker-server1 ~]$ docker run -d --name web3 -p 8803:8080 --volumes-from volume-docker tomcat-web:app2
```

可以创建新容器

![1587915731195](D:\学习资料\笔记\linux\assets\1587915731195.png)

#### 3.1.3.5 测试删除源卷容器 Server 创建容器

```bash
[root@docker-server1 ~]$ docker rm -fv volume-docker
volume-docker
[root@docker-server1 ~]$ docker run -d --name web4 -p 8804:8080 --volumes-from volume-docker tomcat-web:app2
docker: Error response from daemon: No such container: volume-docker.
See 'docker run --help'.
```

![1587915849852](D:\学习资料\笔记\linux\assets\1587915849852.png)

#### 3.1.3.6 测试之前的容器是否正常

已经运行的容器不受任何影响

![1587916173554](D:\学习资料\笔记\linux\assets\1587916173554.png)

#### 3.1.3.7 重新创建容器卷 Server

```bash
[root@docker-server1 ~]$ docker run -d --name volume-docker -v /root/bin/catalina.sh:/apps/tomcat/bin/catalina.sh:ro -v /root/testapp:/apps/tomcat/webapps/testapp tomcat-web:app2
f0b578c60ad3a2550ad4ca937a8abd778e9dceab3f5af71739ef1e72051f2626

[root@docker-server1 ~]$ docker run -d --name web4 -p 8804:8080 --volumes-from volume-docker tomcat-web:app2
f9e118640f7baeb3bf6619af3f145955194594ce99eb6d7edfa93f5d5d652439
```

在当前环境下，即使把提供卷的容器 Server 删除，已经运行的容器 Client 依然可以使用挂载的卷，因为容器是通过挂载访问数据的，但是无法创建新的卷容器客户端，但是再把卷容器 Server 创建后即可正常创建卷容器 Client，此方式可以用于线上数据库、共享数据目录等环境，因为即使数据卷容器被删除了，其他已经运行的容器依然可以挂载使用。

数据卷容器可以作为共享的方式为其他容器提供文件共享，类似于 NFS 共享，可以在生产中启动一个实例挂载本地的目录，然后其他的容器分别挂载此容器的目录，即可保证各容器之间的数据一致性。



# 4 网络部分

## 4.1 docker 结合负载实现网站高可用

### 4.1.1 整体规划图

下图为一个小型的网络架构图，其中 nginx 使用 docker 运行。

![1587916481136](D:\学习资料\笔记\linux\assets\1587916481136.png)

### 4.1.2 安装并配置 keepalived

#### 4.1.2.1 Server1 安装并配置

```bash
[root@docker-server1 ~]$ yum install keepalived –y
[root@docker-server1 ~]$ cat /etc/keepalived/keepalived.conf
vrrp_instance MAKE_VIP_INT {
state MASTER
	interface eth0
	virtual_router_id 1
	priority 100
	advert_int 1
	unicast_src_ip 192.168.10.205
	unicast_peer {
		192.168.10.206
	}

	authentication {
		auth_type PASS
		auth_pass 1111
	}
	virtual_ipaddress {
		192.168.10.100/24 dev eth0 label eth0:1
	}
}

[root@docker-server1~]$ systemctl restart keepalived && systemctl enable keepalived
```

#### 4.1.2.2 Server2 安装并配置

```bash
[root@docker-server2 ~]$ yum install keepalived –y
[root@docker-server2 ~]$ cat /etc/keepalived/keepalived.conf
vrrp_instance MAKE_VIP_INT {
state BACKUP
	interface eth0
	virtual_router_id 1
	priority 50
	advert_int 1
	unicast_src_ip 192.168.10.206
	unicast_peer {
		192.168.10.205
	}

	authentication {
		auth_type PASS
		auth_pass 1111
	}
	virtual_ipaddress {
		192.168.10.100/24 dev eth0 label eth0:1
	}
}

[root@docker-server2~]$ systemctl restart keepalived && systemctl enable keepalived
```

### 4.1.3 安装并配置 haproxy

#### 4.1.3.1 各服务器配置内核参数

```bash
[root@docker-server1 ~]$ sysctl -w net.ipv4.ip_nonlocal_bind=1
[root@docker-server2 ~]$ sysctl -w net.ipv4.ip_nonlocal_bind=1
```

#### 4.1.3.2 Server1 安装并配置 haproxy

```bash
[root@docker-server1 ~]$ yum install haproxy –y
[root@docker-server1 ~]$ cat /etc/haproxy/haproxy.cfg
global
maxconn 100000
uid 99
gid 99
daemon
nbproc 1
log 127.0.0.1 local0 info

defaults
option http-keep-alive
#option forwardfor
maxconn 100000

mode tcp
timeout connect 500000ms
timeout client 500000ms
timeout server 500000ms

listen stats
  mode http
  bind 0.0.0.0:9999
  stats enable
  log global
  stats uri /haproxy-status
  stats auth haadmin:q1w2e3r4ys

#================================================================
frontend docker_nginx_web
  bind 192.168.10.100:80
  mode http
  default_backend docker_nginx_hosts

backend docker_nginx_hosts
  mode http
  #balance source
  balance roundrobin
  server 192.168.10.205 192.168.10.205:81 check inter 2000 fall 3 rise 5
  server 192.168.10.206 192.168.10.206:81 check inter 2000 fall 3 rise 5
```

#### 4.1.3.3  Server2 安装并配置 haproxy

```bash
[root@docker-server1 ~]$ yum install haproxy –y
[root@docker-server1 ~]$ cat /etc/haproxy/haproxy.cfg
global
maxconn 100000
uid 99
gid 99
daemon
nbproc 1
log 127.0.0.1 local0 info

defaults
option http-keep-alive
#option forwardfor
maxconn 100000

mode tcp
timeout connect 500000ms
timeout client 500000ms
timeout server 500000ms

listen stats
  mode http
  bind 0.0.0.0:9999
  stats enable
  log global
  stats uri /haproxy-status
  stats auth haadmin:q1w2e3r4ys

#================================================================
frontend docker_nginx_web
  bind 192.168.10.100:80
  mode http
  default_backend docker_nginx_hosts

backend docker_nginx_hosts
  mode http
  #balance source
  balance roundrobin
  server 192.168.10.205 192.168.10.205:81 check inter 2000 fall 3 rise 5
  server 192.168.10.206 192.168.10.206:81 check inter 2000 fall 3 rise 5
```

#### 4.1.3.4 各服务器分别启动 haproxy

```bash
[root@docker-server1 ~]$ systemctl enable haproxy
Created symlink from /etc/systemd/system/multi-user.target.wants/haproxy.service to /usr/lib/systemd/system/haproxy.service.
[root@docker-server1 ~]$ systemctl restart haproxy

[root@docker-server2 ~]$ systemctl enable haproxy
Created symlink from /etc/systemd/system/multi-user.target.wants/haproxy.service to /usr/lib/systemd/system/haproxy.service.
[root@docker-server2 ~]$ systemctl restart haproxy
```

### 4.1.4 服务器启动 nginx 容器并验证

#### 4.1.4.1 Server1 启动 Nginx 容器

从本地 Nginx 镜像启动一个容器，并指定端口，默认协议是 tcp 方式

```bash
[root@docker-server1 ~]$ docker rm -f `docker ps -a -q` #先删除之前所有的容器
[root@docker-server1 ~]$ docker run --name nginx-web1 -d -p 81:80 jack/nginx-1.14.2:v1 nginx
5410e4042f731d2abe100519269f9241a7db2b3a188c6747b28423b5a584d020
```

#### 4.1.4.2 验证端口

![1587917405687](D:\学习资料\笔记\linux\assets\1587917405687.png)

#### 4.1.4.3 验证 web 访问

![1587917440433](D:\学习资料\笔记\linux\assets\1587917440433.png)

#### 4.1.4.4 Server2 启动 nginx 容器

```bash
[root@docker-server2 ~]$ docker run --name nginx-web1 -d -p 81:80 jack/nginx-1.14.2:v1 nginx
84f2376242e38d7c8ba7fabf3134ac0610ab26358de0100b151df6a231a2b56a
```

#### 4.1.4.5 访问 VIP

![1587917537157](D:\学习资料\笔记\linux\assets\1587917537157.png)

#### 4.1.4.6 Server1 haproxy 状态页面

![1587917589018](D:\学习资料\笔记\linux\assets\1587917589018.png)

#### 4.1.4.7 Server2 haproxy 状态页面

![1587917633688](D:\学习资料\笔记\linux\assets\1587917633688.png)

日志可以在 nginx 里面通过 syslog 传递给 elk 收集

```bash
#指定 IP、协议和端口
[root@linux-docker ~]$ docker run --name nginx-web -d -p 192.168.10.22:80:80/tcp jack/centos-nginx nginx

[root@linux-docker ~]$ docker run --name nginx-web-udp -d -p 192.168.10.22:54:53/udp jack/centos-nginx nginx
```

## 4.2 容器之间的互联

### 4.2.1 通过容器名称互联

即在同一个宿主机上的容器之间可以通过自定义的容器名称相互访问，比如一个业务前端静态页面是使用 nginx，动态页面使用的是 tomcat，由于容器在启动的时候其内部 IP 地址是 DHCP 随机分配的，所以如果通过内部访问的话，自定义名称是相对比较固定的，因此比较适用于此场景。
此方式最少需要两个容器之间操作：

#### 4.2.1.1 先创建第一个容器，后续会使用到这个容器的名称

```bash
[root@docker-server1 ~]$ docker run --name nginx-1 -d -p 8801:80 jack/nginx-1.10.3:v1 nginx
c045c82e85bd620eb9444275b135634a9248760e2061505a1c8b4167e9b24b3d
```

#### 4.2.1.2 查看当着 hosts 文件内容

```bash
[root@c045c82e85bd /]$ cat /etc/hosts
127.0.0.1  localhost
::1 localhost ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
172.17.0.2 c045c82e85bd
```

#### 4.2.1.3 创建第二个容器

```bash
[root@docker-server1 ~]$ docker run -d --name nginx-2 --link nginx-1 -p 8802:80 jack/centos-nginx nginx
23813883ed977d4cc1e50355adaf37564832fc90b4b8b307866c1c99c8256c57
```

#### 4.2.1.4 查看第二个容器的 hosts 文件内容

```bash
[root@23813883ed97 /]$ cat /etc/hosts
127.0.0.1  localhost
::1 localhost ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
172.17.0.2 nginx-1 c045c82e85bd #第一个容器的名称和 ID,只会添加到本地不会添加到对方
172.17.0.3 23813883ed97
```

#### 4.2.1.5 检测通信

![1587918210695](D:\学习资料\笔记\linux\assets\1587918210695.png)

### 4.2.2 通过自定义容器别名互联

上一步骤中，自定义的容器名称可能后期会发生变化，那么一旦名称发生变化，程序之间也要随之发生变化，比如程序通过容器名称进行服务调用，但是容器名称发生变化之后再使用之前的名称肯定是无法成功调用，每次都进行更
改的话又比较麻烦，因此可以使用自定义别名的方式解决，即容器名称可以随意更，只要不更改别名即可，具体如下：

```bash
#命令格式：
docker run -d --name 新容器名称 --link 目标容器名称:自定义的名称 -p 本地端口:容器端口 镜像名称 shell 命令
```

#### 4.2.2.1 启动第三个容器

```bash
[root@docker-server1 ~]$ docker run -d --name nginx-3 --link nginx-1:custom_vm_name -p 8803:80 jack/centos-nginx /usr/sbin/nginx
2ed3016f90fd735c66ae79e039481c1576dd63f67a7861358f2825b234806b0c
```

#### 4.2.2.2 查看当前容器的 hosts 文件

```bash
[root@docker-server1 ~]$ ./docker-in.sh nginx-3
[root@2ed3016f90fd /]$ cat /etc/hosts
127.0.0.1 localhost
::1 localhost ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
172.17.0.2 custom_vm_name c045c82e85bd nginx-1
172.17.0.4 2ed3016f90fd
```

#### 4.2.2.3 检查自定义别名通信

![1587955490403](D:\学习资料\笔记\linux\assets\1587955490403.png)

**查看当前 docker 的网卡信息**

```bash
[root@ansible_ ~]$ docker network list
NETWORK ID          NAME                DRIVER              SCOPE
966e498845c6        bridge              bridge              local
467068f2e514        host                host                local
f7b47491feb2        none                null                local
```

**host 网络使用方式**

```bash
[root@linux-docker ~]$ docker run -it -d --name nginx-host-test --net=host jack/centos-nginx nginx
#验证端口
```

![1587955650397](D:\学习资料\笔记\linux\assets\1587955650397.png)

**web 访问验证**

![1587955673360](D:\学习资料\笔记\linux\assets\1587955673360.png)

### 4.2.3 通过网络跨宿主机互联

同一个宿主机之间的各容器之间是可以直接通信的，但是如果访问到另外一台宿主机的容器呢？

#### 4.2.3.1 docker 网络类型

Docker 的网络有四种类型，下面将介绍每一种类型的具体工作方式：

Bridge 模式，使用参数 –net=bridge 指定，不指定默认就是 bridge 模式。

##### 4.2.3.1.1 Host 模式

Host 模式，使用参数 –net=host 指定。

启动的容器如果指定了使用 host 模式，那么新创建的容器不会创建自己的虚拟网卡，而是直接使用宿主机的网卡和 IP 地址，因此在容器里面查看到的 IP 信息就是宿主机的信息，访问容器的时候直接使用宿主机 IP+容器端口即可，不过容器的其他资源们必须文件系统、系统进程等还是和宿主机保持隔离。

此模式的网络性能最高，但是各容器之间端口不能相同，适用于运行容器端口比较固定的业务。

**先确认宿主机端口没占用 80 端口：** 

![1587955882281](D:\学习资料\笔记\linux\assets\1587955882281.png)

**启动一个新容器，并指定网络模式为 host**

```bash
[root@docker-server1 ~]$ docker run -d --name net_host --net=host jack/centos-nginx nginx
abdc7c9c1e984278bc9344393edf493c3cb09929afb83e34bcd21179d226b61a
```

**验证网络信息**

![1587955980324](D:\学习资料\笔记\linux\assets\1587955980324.png)

**访问宿主机验证**

![1587956006228](D:\学习资料\笔记\linux\assets\1587956006228.png)

**Host 模式不支持端口映射，且容器无法启动**

```bash
[root@node1 ~]$ docker run -d --name net_host1 -p6380:6379 --net=host redis
WARNING: Published ports are discarded when using host network mode
28bae527845a4d70f342fab95148fc10e41481d8b36b4b11c86a5fb80947f901
```

##### 4.2.3.1.2 none 模式

None 模式，使用参数 –net=none 指定

在使用 none 模式后，Docker 容器不会进行任何网络配置，其没有网卡、没有 IP 也没有路由，因此默认无法与外界通信，需要手动添加网卡配置 IP 等，所以极少使用命令使用方式：

```bash
[root@docker-server1 ~]$ docker run -d --name net_none --net=none jack/centos-nginx nginx
143ce15733c1961bf6e23989cbd7d76fc9f9297dc7f11e610ae30c418c21297c
```

![1587956137763](D:\学习资料\笔记\linux\assets\1587956137763.png)

##### 4.2.3.1.3 Container 模式

Container 模式，使用参数 –net=container:名称或 ID 指定。

使用此模式创建的容器需指定和一个已经存在的容器共享一个网络，而不是和宿主机共享网，新创建的容器不会创建自己的网卡也不会配置自己的 IP，而是和一个已经存在的被指定的容器东西 IP 和端口范围，因此这个容器的端口不能和被指定的端口冲突，除了网络之外的文件系统、进程信息等仍然保持相互隔离，两个容器的进程可以通过 lo 网卡社保通信。

```bash
[root@docker-server1 ~]$ docker run -d --name nginx-web1 jack/nginx-1.10.3:v1 nginx
95d22a5d36c18544af47373cc227a1679f239e790f86907d310d13ef4eb85d5e

[root@docker-server1 ~]$ docker run -it --name net_container --
net=container:nginx-web1 jack/centos-nginx bash #直接使用对方的网络，较少使用
```

![1587956276025](D:\学习资料\笔记\linux\assets\1587956276025.png)

#### 4.2.3.1.4 bridge 模式

docker 的默认模式即不指定任何模式就是 bridge 模式，也是使用比较多的模式，此模式创建的容器会为每一个容器分配自己的网络 IP 等信息，并将容器连接到一个虚拟网桥与外界通信。

![1587956328540](D:\学习资料\笔记\linux\assets\1587956328540.png)

```bash
[root@ansible_ ~]$ docker network inspect bridge 
[
    {
        "Name": "bridge",
        "Id": "966e498845c62ad9446ce37214edc92f5285fd25c2f759f318138c4eefb45008",
        "Created": "2020-04-22T16:37:47.412963859+08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.87.0/24",
                    "Gateway": "172.17.87.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {},
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "1500"
        },
        "Labels": {}
    }
]
```

```bash
[root@docker-server1 ~]$ docker run -d --name net_bridge jack/nginx-1.10.3:v1 /usr/sbin/nginx
58e251cdb17fc309ee364c332549b434f4d51019b793702e81b5edb1ff701a7c
```

![1587956442559](D:\学习资料\笔记\linux\assets\1587956442559.png)

#### 4.2.3.2 docker 跨主机互联之简单实现

跨主机互联是说 A 宿主机的容器可以访问 B 主机上的容器，但是前提是保证各宿主机之间的网络是可以相互通信的，然后各容器才可以通过宿主机访问到对方的容器，实现原理是在宿主机做一个网络路由就可以实现 A 宿主机的容器访
问 B 主机的容器的目的，复杂的网络或者大型的网络可以使用 google 开源的k8s 进行互联。

##### 4.2.3.2.1 修改各宿主机网段

Docker 的默认网段是 172.17.0.x/24,而且每个宿主机都是一样的，因此要做路由的前提就是各个主机的网络不能一致，具体如下：
避免影响，先在各服务器删除之前运行的所有容器。

```bash
 docker rm -f `docker ps -a -q`
```

##### 4.2.3.2.2 服务器 A 更改网段

```bash
[root@linux-docker1 ~]$ vim /usr/lib/systemd/system/docker.service
18 ExecStart=/usr/bin/dockerd-current --bip=172.16.10.1/24 \
```

![1587956763882](D:\学习资料\笔记\linux\assets\1587956763882.png)

##### 4.2.3.2.3 重启 docker 服务并验证网卡

```bash
[root@linux-docker1 ~]$ systemctl daemon-reload
[root@linux-docker1 ~]$ systemctl restart docker
```

![1587956912838](D:\学习资料\笔记\linux\assets\1587956912838.png)

##### 4.2.3.2.4 服务器 B 更改网段

```bash
[root@linux-docker2 ~]$ vim /usr/lib/systemd/system/docker.service
18 ExecStart=/usr/bin/dockerd-current --bip=172.16.20.1/24 \
```

![1587956969657](D:\学习资料\笔记\linux\assets\1587956969657.png)

##### 4.2.3.2.5 验证网卡

```bash
[root@linux-docker2 ~]$ systemctl daemon-reload
[root@linux-docker2 ~]$ systemctl restart docker
```

![1587957023758](D:\学习资料\笔记\linux\assets\1587957023758.png)

#### 4.2.3.3 在两个宿主机分别启动一个实例

```bash
#Server1：
[root@docker-server1 ~]$ docker run -d --name test-net-vm jack/nginx-1.10.3:v1 nginx
5d625a868718656737d452d6e57cd23a8a9a1cc9d2e8f76057f1977da2b52a60
```

![1587957081361](D:\学习资料\笔记\linux\assets\1587957081361.png)

```bash
#Server2：
[root@docker-server2 ~]$ docker run -d --name test-net-vm jack/nginx-1.10.3:v1 nginx
ce8fbe4c1d6b5515136ac280f87e6c03884daa35996e996f124ac7056f609e0c
```

![1587957104064](D:\学习资料\笔记\linux\assets\1587957104064.png)

#### 4.2.3.4 添加静态路由

在各宿主机添加静态路由，网关指向对方的 IP

##### 4.2.3.4.1 Server1 添加静态路由

```bash
[root@docker-server1 ~]$ iptables -A FORWARD -s 192.168.10.0/24 -j ACCEPT
[root@docker-server1 ~]$ route add -net 172.16.20.0/24 gw 192.168.10.206
```

![1587957238079](D:\学习资料\笔记\linux\assets\1587957238079.png)

##### 4.2.3.4.2 server2 添加静态路由

```bash
[root@docker-server2 ~]$ iptables -A FORWARD -s 192.168.10.0/24 -j ACCEPT
[root@docker-server2 ~]$ route add -net 172.16.10.0/24 gw 192.168.10.205
```

![1587957298931](D:\学习资料\笔记\linux\assets\1587957298931.png)

##### 4.2.3.4.3 抓包分析

```bash
[root@linux-docker2 ~]$ tcpdump -i eth0 -vnn icmp
```

![1587957773684](D:\学习资料\笔记\linux\assets\1587957773684.png)

#### 4.2.3.5 测试容器间互联

##### 4.2.3.5.1  宿主机 A 到宿主机 B 容器测试

![1587957862831](D:\学习资料\笔记\linux\assets\1587957862831.png)

##### 4.2.3.5.2 宿主机 B 到宿主机 A 容器测试

![1587957911190](D:\学习资料\笔记\linux\assets\1587957911190.png)



# 5 Docker 仓库之单机 Docker Registry

Docker Registry 作为 Docker 的核心组件之一负责镜像内容的存储与分发，客户端的 docker pull 以及 push 命令都将直接与 registry 进行交互,最初版本的 registry 由 Python 实现,由于设计初期在安全性，性能以及 API 的设计上有着诸多的缺陷，该版本在 0.9 之后停止了开发，由新的项目 distribution（新的 docker register 被称为 Distribution）来重新设计并开发下一代 registry，新的项目由 go 语言开发，所有的 API，底层存储方式，系统架构都进行了全面的重新设计已解决上一代 registry 中存在的问题，2016 年 4 月份 rgistry 2.0 正式发布，docker 1.6 版本开始支持 registry 2.0，而八月份随着 docker 1.8 发布，docker hub 正式启用 2.1 版本 registry 全面替代之前版本 registry，新版 registry 对镜像存储格式进行了重新设计并和旧版不兼容，docker 1.5 和之前的版本无法读取 2.0 的镜像，另外，Registry 2.4 版本之后支持了回收站机制，也就是可以删除镜像了，在 2.4 版本之前是无法支持删除镜像的，所以如果你要使用最好是大于 Registry2.4 版本的。

本部分将介绍通过官方提供的 docker registry 镜像来简单搭建一套本地私有仓库环境。

## 5.1 下载 docker registry 镜像

```bash
[root@docker-server1 ~]$ docker pull registry
```

![1587958242481](D:\学习资料\笔记\linux\assets\1587958242481.png)

## 5.2 搭建单机仓库

### 5.2.1 创建授权使用目录

```bash
[root@docker-server1 ~]$ mkdir /docker/auth #创建一个授权使用目录
```

### 5.2.2 创建用户

```bash
[root@docker-server1 ~]$ cd /docker
[root@docker-server1 docker]$ docker run --entrypoint htpasswd registry -Bbn jack 123456 > auth/htpasswd #创建一个用户并生成密码
```

### 5.2.3 验证用户名密码

```bash
[root@docker-server1 docker]$ cat auth/htpasswd
jack:$2y$05$8W2aO/2RXMrMzw/0M5pig..QXwUh/m/XPoW5H/XxloLLRDTepVGP6
```

### 5.2.4 启动 docker registry

```bash
[root@docker-server1 docker]$ docker run -d -p 5000:5000 --restart=always --name registry1 -v /docker/auth:/auth -e "REGISTRY_AUTH=htpasswd" -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm"  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd
registryce659e85018bea3342045f839c43b66de1237ce5413c0b6b72c0887bece5325a
```

### 5.2.5 验证端口和容器

![1587958998011](D:\学习资料\笔记\linux\assets\1587958998011.png)

### 5.2.6 测试登录仓库

#### 5.2.6.1 报错如下

![1587959053382](D:\学习资料\笔记\linux\assets\1587959053382.png)

#### 5.2.6.2 解决办法

```bash
#编辑各 docker 服务器/etc/sysconfig/docker 配置文件如下：
[root@docker-server1 ~]$ vim /etc/sysconfig/docker
4 OPTIONS='--selinux-enabled --log-driver=journald'
9 ADD_REGISTRY='--add-registry 192.168.10.205:5000'
10 INSECURE_REGISTRY='--insecure-registry 192.168.10.205:5000'

[root@docker-server1 ~]$ systemctl restart docker

[root@docker-server2 ~]$ vim /etc/sysconfig/docker
4 OPTIONS='--selinux-enabled --log-driver=journald'
5 if [ -z "${DOCKER_CERT_PATH}" ]; then
6 DOCKER_CERT_PATH=/etc/docker
7 fi
8
9 ADD_REGISTRY='--add-registry 192.168.10.205:5000'
10 INSECURE_REGISTRY='--insecure-registry 192.168.10.205:5000'

[root@docker-server2 ~]$ systemctl restart docker
```

#### 5.2.6.3 验证各 docker 服务器登录

**server1**

![1587973492876](D:\学习资料\笔记\linux\assets\1587973492876.png)

**server2**

![1587973513210](D:\学习资料\笔记\linux\assets\1587973513210.png)

### 5.2.7 在 Server1 登录后上传镜像

#### 5.2.7.1 镜像打 tag

```bash
[root@docker-server1  ~]$  docker  tag  jack/nginx-1.10.3:v1 192.168.10.205:5000/jack/nginx-1.10.3:v1
```

![1587973593858](D:\学习资料\笔记\linux\assets\1587973593858.png)

#### 5.2.7.2 上传镜像

![1587973615060](D:\学习资料\笔记\linux\assets\1587973615060.png)

### 5.2.8 Server2 下载镜像并启动容器

#### 5.2.8.1 登录并从 docker registry 下载镜像

```bash
[root@docker-server2 ~]$ docker login 192.168.10.205:5000
Username (jack): jack
Password:
Login Succeeded
[root@docker-server2 ~]$ docker pull 192.168.10.205:5000/jack/nginx-1.10.3:v1
```

#### 5.2.8.2 从下载的镜像启动容器

```bash
[root@docker-server2 ~]$ docker run -d --name docker-registry -p 80:80 192.168.10.205:5000/jack/nginx-1.10.3:v1 nginx

2ba24f28362e1b039fbebda94a332111c2882aa06987463ae033c630f5c9927c
```

![1587973886997](D:\学习资料\笔记\linux\assets\1587973886997.png)



# 6 docker 仓库之分布式 Harbor

Harbor 是一个用于存储和分发 Docker 镜像的企业级 Registry 服务器，由 vmware 开源，其通过添加一些企业必需的功能特性，例如安全、标识和管理等，扩展了开源 Docker Distribution。作为一个企业级私有 Registry 服务器，
Harbor 提供了更好的性能和安全。提升用户使用 Registry 构建和运行环境传输镜像的效率。Harbor 支持安装在多个 Registry 节点的镜像资源复制，镜像全部保存在私有 Registry 中， 确保数据和知识产权在公司内部网络中管控，另外，Harbor 也提供了高级的安全特性，诸如用户管理，访问控制和活动审计等，官网地址：https://vmware.github.io/harbor/cn/，官方 github 地址：https://github.com/vmware/harbor

## 6.1 Harbor 功能官方介绍

基于角色的访问控制：用户与 Docker 镜像仓库通过“项目”进行组织管理，一个用户可以对多个镜像仓库在同一命名空间（project）里有不同的权限。

镜像复制：镜像可以在多个 Registry 实例中复制（同步）。尤其适合于负载均衡，高可用，混合云和多云的场景。
图形化用户界面：用户可以通过浏览器来浏览，检索当前 Docker 镜像仓库，管理项目和命名空间。

AD/LDAP 支持：Harbor 可以集成企业内部已有的 AD/LDAP，用于鉴权认证管理。

审计管理：所有针对镜像仓库的操作都可以被记录追溯，用于审计管理。

国际化：已拥有英文、中文、德文、日文和俄文的本地化版本。更多的语言将会添加进来。

RESTful API - RESTful API ：提供给管理员对于 Harbor 更多的操控, 使得与其它管理软件集成变得更容易。

部署简单：提供在线和离线两种安装工具， 也可以安装到 vSphere 平台(OVA 方式)虚拟设备。

## 6.2 安装 Harbor

下载地址：https://github.com/vmware/harbor/releases
安装文档：https://github.com/vmware/harbor/blob/master/docs/installation_guide.md

### 6.2.1 服务器 1 安装 docker

```bash
[root@docker-server1 ~]$ yum install docker -y
[root@docker-server1 ~]$ systemctl satrt docker
[root@docker-server1 ~]$ systemctl enable docker
Created symlink from /etc/systemd/system/multi-user.target.wants/docker.service to
/usr/lib/systemd/system/docker.service.
```

### 6.2.2 服务器 2 安装 docker

```bash
[root@docker-server2 ~]$ yum install docker -y
[root@docker-server2 ~]$ systemctl start docker
[root@docker-server2 ~]$ systemctl enable docker
Created symlink from /etc/systemd/system/multi-user.target.wants/docker.service to
/usr/lib/systemd/system/docker.service.
```

### 6.2.3  下载 Harbor 安装包

```bash
#下载离线完整安装包
root@docker-server2 ~]$ cd /usr/local/src/
[root@docker-server2 src]$ wget https://github.com/vmware/harbor/releases/download/v1.2.2/harbor-offline-installer-v1.7.5.tgz
```

## 6.3 配置 Harbor

### 6.3.1 解压并编辑 harbor.cfg

````bash
[root@docker-server1 src]$ tar xvf harbor-offline-installer-v1.7.5.tgz
[root@docker-server1 src]$ ln -sv /usr/local/src/harbor /usr/local/
‘/usr/local/harbor’ -> ‘/usr/local/src/harbor’
[root@docker-server1 harbor]$ cd /usr/local/harbor/
[root@docker-server1 harbor]$ yum install python-pip –y
[root@docker-server1 harbor]$ docker-compose start
[root@docker-server1 harbor]$ vim harbor.cfg
[root@docker-server1 harbor]$ grep "^[a-Z]" harbor.cfg
hostname = 192.168.10.205
ui_url_protocol = http
db_password = root123
max_job_workers = 3
customize_crt = on
ssl_cert = /data/cert/server.crt
ssl_cert_key = /data/cert/server.key
secretkey_path = /data
admiral_url = NA
clair_db_password = password
email_identity = harbor
email_server = smtp.163.com
email_server_port = 25
email_username = rooroot@163.com
email_password = zhang@123
email_from = admin <rooroot@163.com>
email_ssl = false
harbor_admin_password = zhang@123
auth_mode = db_auth
ldap_url = ldaps://ldap.mydomain.com
ldap_basedn = ou=people,dc=mydomain,dc=com
ldap_uid = uid
ldap_scope = 3
ldap_timeout = 5
self_registration = on
token_expiration = 30
project_creation_restriction = everyone
verify_remote_cert = on
````

### 6.3.2 更新 harbor 配置

#### 6.3.2.1 首次部署 harbor 更新

```bash
[root@docker-server1 harbor]$ pwd
/usr/local/harbor 
[root@docker-server1 harbor]$ ./prepare #更新配置
```

![1587982604314](D:\学习资料\笔记\linux\assets\1587982604314.png)

执行完毕会在当前目录生成一个 docker-compose.yml 文件，用于配置数据目录等配置信息

#### 6.3.2.2 后期修改配置

如果 harbor 运行一段时间之后需要更改配置，则步骤如下：

```
#停止 harbor
[root@docker-server1 harbor]$ pwd
/usr/local/harbor
[root@docker-server1 harbor]$ docker-compose stop

#编辑 harbor.cfg 进行相关配置
[root@docker-server1 harbor]$ vim harbor.cfg

#更新配置：
[root@docker-server1 harbor]$ ./prepare

#启动 harbor 服务
[root@docker-server1 harbor]$ docker-compose start
```

![1587983090559](D:\学习资料\笔记\linux\assets\1587983090559.png)

### 6.3.3 官方方式启动 Harbor

#### 6.3.3.1 官方方式安装并启动 harbor

```bash
[root@docker-server1 harbor]$ yum install python-pip
[root@docker-server1 harbor]$ pip install --upgrade pip
[root@docker-server1 harbor]$ pip install docker-compose
[root@docker-server1 harbor]$ ./install.sh #官方构建 harbor 和启动方式，推荐此方法，会下载官方的 docker 镜像
```

#### 6.3.3.2 部署过程中

![1587983274035](D:\学习资料\笔记\linux\assets\1587983274035.png)

#### 6.3.3.3 部署完成

![1587983293731](D:\学习资料\笔记\linux\assets\1587983293731.png)

#### 6.3.3.4 查看本地的镜像

![1587983316861](D:\学习资料\笔记\linux\assets\1587983316861.png)

#### 6.3.3.5 查看本地端口

![1587983378418](D:\学习资料\笔记\linux\assets\1587983378418.png)

#### 6.3.3.6 web 访问 Harbor 管理界面

![1587983492212](D:\学习资料\笔记\linux\assets\1587983492212.png)

#### 6.3.3.7 登录成功后的界面

![1587983524235](D:\学习资料\笔记\linux\assets\1587983524235.png)

### 6.3.4 非官方方式启动

#### 6.3.4.1 非官方方式启动 harbor

```bash
[root@docker-server2 harbor]$ ./prepare
[root@docker-server2 harbor]$ yum install python-pip -y
[root@docker-server2 harbor]$ pip install --upgrade pip #升级 pip 为最新版本
[root@docker-server2 harbor]$ pip install docker-compose #安装 docker-compose命令
```

![1587983593120](D:\学习资料\笔记\linux\assets\1587983593120.png)

#### 6.3.4.2 启动 harbor

```bash
[root@docker-server2 harbor]$ docker-compose up –d #非官方方式构建容器，此步骤会从官网下载镜像，需要相当长的时间
```

![1587983662878](D:\学习资料\笔记\linux\assets\1587983662878.png)

## 6.4 配置 docker 使用 harbor 仓库上传下载镜像

### 6.4.1 编辑 docker 配置文件

```bash
#注意：如果我们配置的是 https 的话，本地 docker 就不需要有任何操作就可以访问 harbor 了
[root@docker-server1 ~]$ vim /etc/sysconfig/docker
4 OPTIONS='--selinux-enabled --log-driver=journald --insecure-registry 192.168.10.205'
#其中 192.168.10.205 是我们部署 Harbor 的地址，即 hostname 配置项值。配置完后需要重启 docker 服务。
```

### 6.4.2 重启 docker 服务

```bash
[root@docker-server1 ~]$ systemctl stop docker
[root@docker-server1 ~]$ systemctl start docker
```

### 6.4.3 验证能否登录 harbor

```bash
[root@docker-server1 harbor]$ docker login 192.168.10.205
```

![1587983897798](D:\学习资料\笔记\linux\assets\1587983897798.png)

### 6.4.4 测试上传和下载镜像

将之前单机仓库构建的 Nginx 镜像上传到 harbor 服务器用于测试

#### 6.4.4.1 导入镜像

```bash
[root@docker-server1 harbor]$ docker load < /opt/nginx-1.10.3_docker.tar.gz
```

![1587983997141](D:\学习资料\笔记\linux\assets\1587983997141.png)

#### 6.4.4.2 镜像打 tag

```bash
#修改 images 的名称，不修改成指定格式无法将镜像上传到 harbor 仓库，格式为: HarborIP/项目名/image 名字:版本号：
[root@docker-server1 harbor]$ docker tag 192.168.10.205:5000/jack/nginx-1.10.3:v1 192.168.10.205/nginx/nginx_1.10.3:v1
[root@docker-server1 harbor]$ docker images
```

![1587984117749](D:\学习资料\笔记\linux\assets\1587984117749.png)

#### 6.4.4.3 在 harbor 管理界面创建项目

![1587984169751](D:\学习资料\笔记\linux\assets\1587984169751.png)

#### 6.4.4.4 将镜像 push 到 harbor

![1587984273799](D:\学习资料\笔记\linux\assets\1587984273799.png)

#### 6.4.4.5 harbor 界面验证镜像上传成功

![1587984320804](C:\Users\huawei\AppData\Roaming\Typora\typora-user-images\1587984320804.png)

#### 6.4.4.6 验证镜像信息

![1587984346939](D:\学习资料\笔记\linux\assets\1587984346939.png)

### 6.4.5 验证从 harbor 服务器下载镜像并启动容器

#### 6.4.5.1 更改 docker 配置文件

```bash
#从 harbor 镜像服务器下载 image 的 docker 服务都要更改，不更改的话无法下载：
[root@docker-server2 ~]$ vim /etc/sysconfig/docker
4  OPTIONS='--selinux-enabled  --log-driver=journald  --insecure-registry 192.168.10.205'

#重启 docker
[root@docker-server2 ~]$ systemctl stop docker
[root@docker-server2 ~]$ systemctl start docker
```

#### 6.4.5.2 查看下载命令

harbor 上的每个镜像里面自带 pull 命令

![1587984673020](D:\学习资料\笔记\linux\assets\1587984673020.png)

#### 6.4.5.3 执行下载

```bash
[root@docker-server2 ~]$ docker pull 192.168.10.205/nginx/nginx_1.10.3:v1
```

![1587984743220](D:\学习资料\笔记\linux\assets\1587984743220.png)

#### 6.4.5.4 启动容器

```bash
[root@docker-server2  ~]$  docker  run  -d  -p  80:80  -p  443:443 192.168.10.205/nginx/nginx_1.10.3:v1 nginx

89901f9badf74809f6abccc352fc7479f1490f0ebe6d6e3b36d689e73c3f9027
```

#### 6.4.5.5 验证端口

![1587984958312](D:\学习资料\笔记\linux\assets\1587984958312.png)

#### 6.4.5.6 验证 web访问

![1587985004672](D:\学习资料\笔记\linux\assets\1587985004672.png)

## 6.5 实现 harbor 高可用

Harbor 支持基于策略的 Docker 镜像复制功能，这类似于 MySQL 的主从同步，其可以实现不同的数据中心、不同的运行环境之间同步镜像，并提供友好的管理界面，大大简化了实际运维中的镜像管理工作，已经有很多互联网公司使用 harbor 搭建内网 docker 仓库的案例，并且还有实现了双向复制的案列，本文将实现单向复制的部署。

### 6.5.1 新部署一台 harbor 服务器

```bash
[root@docker-server2 ~]$ cd /usr/local/src/
[root@docker-server2 src]$ tar xf harbor-offline-installer-v1.2.2.tgz
[root@docker-server2 src]$ ln -sv /usr/local/src/harbor /usr/local/
[root@docker-server2 src]$ cd /usr/local/harbor/
[root@docker-server2 harbor]$ grep "^[a-Z]" harbor.cfg
hostname = 192.168.10.206
ui_url_protocol = http
db_password = root123
max_job_workers = 3
customize_crt = on
ssl_cert = /data/cert/server.crt
ssl_cert_key = /data/cert/server.key
secretkey_path = /data
admiral_url = NA
clair_db_password = password
email_identity = harbor-1.2.2
email_server = smtp.163.com
email_server_port = 25
email_username = rooroot@163.com
email_password = zhang@123
email_from = admin <rooroot@163.com>
email_ssl = false
harbor_admin_password = zhang@123
auth_mode = db_auth
ldap_url = ldaps://ldap.mydomain.com
ldap_basedn = ou=people,dc=mydomain,dc=com
ldap_uid = uid
ldap_scope = 3
ldap_timeout = 5
self_registration = on
token_expiration = 30
project_creation_restriction = everyone
verify_remote_cert = on

[root@docker-server2 harbor]$ yum install python-pip -y
[root@docker-server2 harbor]$ pip install --upgrade pip
[root@docker-server2 harbor]$ pip install docker-compose
[root@docker-server2 harbor]$ ./install.sh
```

### 6.5.2 验证从 harbor 登录

![1588060736449](D:\学习资料\笔记\linux\assets\1588060736449.png)

### 6.5.3 创建一个 nginx 项目

与主 harbor 项目名称保持一致

![1588060853392](D:\学习资料\笔记\linux\assets\1588060853392.png)

### 6.5.4 在主 harbor 服务器配置同步测试

![1588060928254](D:\学习资料\笔记\linux\assets\1588060928254.png)

### 6.5.5 点击复制规则

![1588060991153](D:\学习资料\笔记\linux\assets\1588060991153.png)

### 6.5.6 主 harbor 编辑同步策略

![1588086870259](D:\学习资料\笔记\linux\assets\1588086870259.png)

### 6.5.7 主 harbor 查看镜像同步状态

![1588086907386](D:\学习资料\笔记\linux\assets\1588086907386.png)

### 6.5.8 从 harbor 查看镜像

![1588086938847](D:\学习资料\笔记\linux\assets\1588086938847.png)

### 6.5.9 测试从 harbor 镜像下载和容器启动

#### 6.5.9.1 docker 客户端配置使用 harbor

本次新部署了一台 docker 客户端， IP 地址为 192.168.10.207

```bash
[root@docker-server3 ~]$ vim /etc/sysconfig/docker
4  OPTIONS='--selinux-enabled  --log-driver=journald  --insecure-registry 192.168.10.206'
```

#### 6.5.9.2 重启 docker 服务

```bash
[root@docker-server3 ~]$ systemctl restart docker
```

#### 6.5.9.3 从 harbor 项目设置为公开

![1588087103695](D:\学习资料\笔记\linux\assets\1588087103695.png)

#### 6.5.9.4 设置项目为公开访问

![1588087136858](D:\学习资料\笔记\linux\assets\1588087136858.png)

#### 6.5.9.5 docker 客户端下载镜像

![1588087166651](D:\学习资料\笔记\linux\assets\1588087166651.png)

#### 6.5.9.6 docker 客户端从镜像启动容器

```bash
[root@docker-server3  ~]$  docker run -d -p 80:80 -p 443:443 192.168.10.206/nginx/nginx_1.10.3:v1 nginx
0b496bc81035291b80062d1fba7d4065079ab911c2a550417cf9e593d353c20b
```

#### 6.5.9.7 验证 web 访问

![1588087248859](D:\学习资料\笔记\linux\assets\1588087248859.png)

至此，高可用模式的 harbor 仓库部署完毕

## 6.6 实现 harbor 双向同步

### 6.6.1 在 docer 客户端导入 centos 基础镜像

```bash
[root@docker-server3 ~]$ docker load -i /opt/centos.tar.gz
[root@docker-server3 ~]$ vim /etc/sysconfig/docker
4  OPTIONS='--selinux-enabled  --log-driver=journald  --insecure-registry 192.168.10.206'
```

### 6.6.2 镜像打 tag

```bash
[root@docker-server3  ~]$  docker  tag  docker.io/centos 192.168.10.206/nginx/centos_base
```

![1588087403615](D:\学习资料\笔记\linux\assets\1588087403615.png)

### 6.6.3 上传到从 harbor

```bash
[root@docker-server3 ~]$ docker push 192.168.10.206/nginx/centos_base
```

![1588087468325](D:\学习资料\笔记\linux\assets\1588087468325.png)

### 6.6.4 从 harbor 界面验证

![1588087500042](D:\学习资料\笔记\linux\assets\1588087500042.png)

### 6.6.5 从 harbor 创建同步规则

规则方式与主 harbor 相同，写对方的 IP+用户密码，然后点测试连接，确认可以测试连通过。

![1588087598342](D:\学习资料\笔记\linux\assets\1588087598342.png)

### 6.6.6 到主 harbor 验证镜像

![1588087627646](D:\学习资料\笔记\linux\assets\1588087627646.png)

### 6.6.7 docker 镜像端测试

#### 6.6.7.1 下载 centos 基础镜像

```bash
[root@docker-server1 harbor]$ docker pull 192.168.10.205/nginx/centos_base
Using default tag: latest
Trying to pull repository 192.168.10.205/nginx/centos_base ...
sha256:822de5245dc5b659df56dd32795b08ae42db4cc901f3462fc509e91e97132dc
0: Pulling from 192.168.10.205/nginx/centos_base

Digest:
sha256:822de5245dc5b659df56dd32795b08ae42db4cc901f3462fc509e91e97132dc0
```

![1588087714848](D:\学习资料\笔记\linux\assets\1588087714848.png)

#### 6.6.7.2 从镜像启动容器

```bash
[root@docker-server1  ~]$  docker  run  -it  --name  centos_base 192.168.10.205/nginx/centos_base bash
[root@771f5aa0d089 /]$
```

![1588087762158](D:\学习资料\笔记\linux\assets\1588087762158.png)

## 6.7 harbor https 配置

```bash
# openssl genrsa -out /usr/local/src/harbor/certs/harbor-ca.key 2048
# openssl req -x509 -new -nodes -key /usr/local/src/harbor/certs/harbor-ca.key -subj "/CN=harbor.magedu.net" -days 7120 -out /usr/local/src/harbor/certs/harbor-ca.crt

# vim harbor.cfg
	hostname = harbor.magedu.net
	ui_url_protocol = https
	ssl_cert = /usr/local/src/harbor/certs/harbor-ca.crt
	ssl_cert_key = /usr/local/src/harbor/certs/harbor-ca.key
	harbor_admin_password = 123456
# ./install.sh
# yum install docker-ce-18.06.3.ce-3.el7.x86_64.rpm
# yum install docker-compose
# mkdir /etc/docker/certs.d/harbor.magedu.net -p
# cp certs/harbor-ca.crt /etc/docker/certs.d/harbor.magedu.net/
# docker login harbor.magedu.net
```

# 7 单机编排之 Docker Compose

当在宿主机启动较多的容器时候，如果都是手动操作会觉得比较麻烦而且容器出错，这个时候推荐使用 docker 单机编排工具 docker compose，Docker Compose 是 docker 容器的一种编排服务，docker compose 是一个管理多个容器的工具，比如可以解决容器之间的依赖关系，就像启动一个 web 就必须得先把数据库服务先启动一样，docker compose 完全可以替代 docker run 启动容器。
#github 地址 https://github.com/docker/compose

## 7.1 基础环境准备

### 7.1.1 安装 ptyhon 环境及 pip 命令

```bash
[root@docker-server3 ~]$ yum install https://mirrors.aliyun.com/epel/epel-release-
latest-7.noarch.rpm -y
[root@docker-server3 ~]$ yum install python-pip -y
[root@docker-server3 ~]$ pip install --upgrade pip
```

![1588087978441](D:\学习资料\笔记\linux\assets\1588087978441.png)

### 7.1.2 安装 docker compose

```bash
[root@docker-server3 ~]$ pip install docker-compose
```

![1588088026848](D:\学习资料\笔记\linux\assets\1588088026848.png)

### 7.1.3 验证版本

```bash
[root@docker-server3 ~]$ docker-compose version
```

### 7.1.4 查看帮助

```bash
[root@docker-server3 ~]$ docker-compose --help
```

![1588088083889](D:\学习资料\笔记\linux\assets\1588088083889.png)

## 7.2 从 docker compose 启动单个容器

可以在任意目录，推荐话有意义的位置

```bash
[root@docker-server3 ~]$ mkdir docker-compose
[root@docker-server3 ~]$ cd docker-compose/
```

### 7.2.1 一个容器的 docker compose 文件

设置一个 yml 格式的配置文件，因此要注意前后的缩进。

```bash
[root@docker-server3 docker-compose]$ cat docker-compose.yml
web1:
  image: 192.168.10.206/nginx/nginx_1.10.3
  expose:
    - 80
    - 443
  ports:
    - "80:80"
    - "443:443"
```

### 7.7.2 启动容器

必须要在 docker compose 文件所在的目录执行

```bash
[root@docker-server3 docker-compose]$ docker-compose up #前台启动
```

![1588088306511](D:\学习资料\笔记\linux\assets\1588088306511.png)

### 7.2.3 启动完成

![1588088324665](D:\学习资料\笔记\linux\assets\1588088324665.png)

### 7.2.4 web 访问测试

![1588088345754](D:\学习资料\笔记\linux\assets\1588088345754.png)

### 7.2.5 后台启动服务

容器在启动的时候，会给容器自定义一个名称

```bash
[root@docker-server3 docker-compose]$ docker-compose up -d
```

![1588088408692](D:\学习资料\笔记\linux\assets\1588088408692.png)

### 7.2.6 自定义容器名称

```bash
[root@docker-server3 docker-compose]$ cat docker-compose.yml
  web1:
    image: 192.168.10.206/nginx/nginx_1.10.3
    expose:
      - 80
      - 443
    container_name: nginx-web1 #自定义容器名称
    ports:
      - "80:80"
     - "443:443
```

![1588088513665](D:\学习资料\笔记\linux\assets\1588088513665.png)

### 7.2.7 验证容器

![1588088532351](D:\学习资料\笔记\linux\assets\1588088532351.png)

### 7.2.8 查看容器进程

```bash
[root@docker-server3 docker-compose]$ docker-compose ps
```

![1588088566003](D:\学习资料\笔记\linux\assets\1588088566003.png)

## 7.3 从 docker compose 启动多个容器

### 7.3.1 编辑 docker-compose 文件

```bash
[root@docker-server3 docker-compose]$ cat docker-compose.yml
web1:
  image: 192.168.10.206/nginx/nginx_1.10.3
  expose:
    - 80
    - 443
  container_name: nginx-web1
  ports:
    - "80:80"
    - "443:443"

web2: #每一个容器一个 ID
  image: 192.168.10.206/nginx/nginx_1.10.3
  expose:
     - 80
     - 443
  container_name: nginx-web2
  ports:
    - "81:80"
    - "444:443"
```

### 7.3.2 重新启动容器

```bash
[root@docker-server3 docker-compose]$ docker-compose stop
[root@docker-server3 docker-compose]$ docker-compose up –d
```

![1588122613935](D:\学习资料\笔记\linux\assets\1588122613935.png)

### 7.3.3 web 访问测试

![1588122636907](D:\学习资料\笔记\linux\assets\1588122636907.png)

![1588122644861](D:\学习资料\笔记\linux\assets\1588122644861.png)

## 7.4 定义数据卷挂载

### 7.4.1 创建数据目录和文件

```bash
[root@docker-server3 ~]$ mkdir -p /data/nginx
[root@docker-server3 ~]$ echo "Test Nginx Volume" > /data/nginx/index.html
```

### 7.4.2 编辑 compose 配置文件

```bash
[root@docker-server3 docker-compose]$ vim docker-compose.yml
web1:
  image: 192.168.10.206/nginx/nginx_1.10.3
  expose:
    - 80
    - 443
  container_name: nginx-web1
  volumes:
    - /data/nginx:/usr/local/nginx/html
  ports:
    - "80:80"
    - "443:443"

web2:
  image: 192.168.10.206/nginx/nginx_1.10.3
  expose:
    - 80
    - 443
  container_name: nginx-web2
  ports:
    - "81:80"
    - "444:443"
```

### 7.4.3 重启容器

```bash
[root@docker-server3 docker-compose]$ docker-compose stop
[root@docker-server3 docker-compose]$ docker-compose up -d
```

![1588122843153](D:\学习资料\笔记\linux\assets\1588122843153.png)

### 7.4.4 验证 web 访问

![1588122869952](D:\学习资料\笔记\linux\assets\1588122869952.png)

可以发现，同一个文件，数据卷的优先级比镜像内的文件优先级高

![1588122892738](D:\学习资料\笔记\linux\assets\1588122892738.png)

### 7.4.5 其他常用命令

#### 7.4.5.1 重启单个指定容器

```bash
[root@docker-server3 docker-compose]$ docker-compose restart web1
Restarting nginx-web1 ... done
```

![1588122948032](D:\学习资料\笔记\linux\assets\1588122948032.png)

#### 7.4.5.2 重启容器

```bash
[root@docker-server3 docker-compose]$ docker-compose restart
```

![1588122992215](D:\学习资料\笔记\linux\assets\1588122992215.png)

#### 7.4.5.3 停止和启动

```bash
[root@docker-server3 docker-compose]$ docker-compose stop web1
[root@docker-server3 docker-compose]$ docker-compose start web1
```

![1588123556816](D:\学习资料\笔记\linux\assets\1588123556816.png)

#### 7.4.5.4 停止和启动所有容器

```bash
[root@docker-server3 docker-compose]$ docker-compose stop
[root@docker-server3 docker-compose]$ docker-compose start
```

![1588123603359](D:\学习资料\笔记\linux\assets\1588123603359.png)

## 7.5 实现单机版的 HA+Nginx+Tomcat

### 7.5.1 制作 Haproxy 镜像

```bash
[root@docker-server1 haproxy]$ pwd
/opt/dockerfile/web/haproxy
```

#### 7.5.1.1 编辑 Dockerfile 文件

```bash
[root@docker-server1 haproxy]$ cat Dockerfile
#My Dockerfile
From docker.io/centos:7.2.1511
MAINTAINER zhangshijie "zhangshijie@300.cn"

#Yum Setting
ADD epel.repo /etc/yum.repos.d/epel.repo
ADD CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo

RUN yum install gcc gcc-c++ pcre pcre-devel openssl openssl-devel -y

ADD haproxy-1.7.9.tar.gz /usr/local/src
RUN cd /usr/local/src/haproxy-1.7.9 && make TARGET=linux2628 USE_PCRE=1 USE_OPENSSL=1 USE_ZLIB=1 PREFIX=/usr/local/haproxy && make install
RUN cp /usr/local/src/haproxy-1.7.9/haproxy-systemd-wrapper /usr/sbin/haproxy-systemd-wrapper
RUN cp /usr/local/src/haproxy-1.7.9/haproxy /usr/sbin/haproxy
ADD haproxy.service /usr/lib/systemd/system/haproxy.service
ADD haproxy /etc/sysconfig/haproxy
ADD run_haproxy.sh /root/script/run_haproxy.sh
RUN chmod a+x /root/script/run_haproxy.sh

CMD ["/root/script/run_haproxy.sh"]
EXPOSE 80 9999
```

#### 7.5.1.2 准备服务启动脚本

```bash
[root@docker-server1 haproxy]$ cat haproxy.service
[Unit]
Description=HAProxy Load Balancer
After=syslog.target network.target

[Service]
EnvironmentFile=/etc/sysconfig/haproxy
ExecStart=/usr/sbin/haproxy-systemd-wrapper  -f  /etc/haproxy/haproxy.cfg  -p
/run/haproxy.pid $OPTIONS
ExecReload=/bin/kill -USR2 $MAINPID

[Install]
WantedBy=multi-user.target
```

#### 7.5.1.3 前台启动脚本

```bash
[root@docker-server1 haproxy]$ cat run_haproxy.sh
#!/bin/bash
/usr/sbin/haproxy-systemd-wrapper -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid
```

#### 7.5.1.4 haproxy 参数文件

```bash
[root@docker-server1 haproxy]$ cat haproxy
# Add extra options to the haproxy daemon here. This can be useful for
# specifying multiple configuration files with multiple -f options.
# See haproxy(1) for a complete list of options.
OPTIONS=""
```

#### 7.5.1.5 准备压缩包及其他文件

![1588124008783](D:\学习资料\笔记\linux\assets\1588124008783.png)

#### 7.5.1.6 执行构建镜像

```bash
[root@docker-server1  haproxy]$  docker build -t 192.168.10.205/centos/centos_7.2.1511_haproxy_1.7.9 /opt/dockerfile/web/haproxy/
```

#### 7.5.1.7 经镜像上传到 harbor 仓库

![1588124109013](D:\学习资料\笔记\linux\assets\1588124109013.png)

#### 7.5.1.8 harbor 仓库验证

![1588124135694](D:\学习资料\笔记\linux\assets\1588124135694.png)

### 7.5.2 编辑 docker compose 文件及环境准备

#### 7.5.2.1 编辑 docker compose 文件

```bash
[root@docker-server3 docker-compose]$ pwd
/root/docker-compose
[root@docker-server3 docker-compose]$ cat docker-compose.yml
nginx-web1:
  image: 192.168.10.205/nginx/nginx_1.10.3
  expose:
    - 80
    - 443
  container_name: nginx-web1
  volumes:
    - /data/nginx:/usr/local/nginx/html
    - /usr/local/nginx/conf/nginx.conf:/usr/local/nginx/conf/nginx.conf
  links:
    - tomcat-web1
    - tomcat-web2
  
  nginx-web2:
    image: 192.168.10.205/nginx/nginx_1.10.3
    volumes:
      - /usr/local/nginx/conf/nginx.conf:/usr/local/nginx/conf/nginx.conf
    expose:
      - 80
      - 443
    container_name: nginx-web2
    links:
      - tomcat-web1
      - tomcat-web2

   tomcat-web1:
     container_name: tomcat-web1
     image: 192.168.10.205/centos/jdk1.7.0.79_tomcat1.7.0.69
     user: www
     command: /apps/tomcat/bin/run_tomcat.sh
   volumes:
     - /apps/tomcat/webapps/SalesManager:/apps/tomcat/webapps/SalesManager
   expose:
     - 8080
     - 8443
   
   tomcat-web2:
     container_name: tomcat-web2
     image: 192.168.10.205/centos/jdk1.7.0.79_tomcat1.7.0.69
     user: www
     command: /apps/tomcat/bin/run_tomcat.sh
   
   volumes:
     - /apps/tomcat/webapps/SalesManager:/apps/tomcat/webapps/SalesManager
   expose:
     - 8080
     - 8443

  haproxy:
    container_name: haproxy-web1
    image: 192.168.10.205/centos/centos_7.2.1511_haproxy_1.7.9
    command: /root/script/run_haproxy.sh
    volumes:
      - /etc/haproxy/haproxy.cfg:/etc/haproxy/haproxy.cfg
    ports:
      - "9999:9999"
      - "80:80"
    links:
      - nginx-web1
      - nginx-web2
```

#### 7.5.2.2 准备 nginx 静态文件

```bash
[root@docker-server3 docker-compose]$ cat /data/nginx/index.html
Test Nginx Volume
```

#### 7.5.2.3 准备 nginx 配置文件

本地路径和 nginx 路径都是 /usr/local/nginx/conf/nginx.conf

```bash
[root@docker-server3 docker-compose]$ grep -v "#" /usr/local/nginx/conf/nginx.conf | grep -v "^$"
user nginx;
worker_processes auto;
daemon off;

events {
	worker_connections 1024;
}

http {
	include mime.types;
	default_type application/octet-stream;
	sendfile on;
	keepalive_timeout 65;

upstream tomcat_webserver {
		server tomcat-web1:8080;
		server tomcat-web2:8080;
}
	
	server {
		listen 80;
		server_name localhost;
		location / {
			root html;
			index index.html index.htm;
	}
	location /SalesManager {
		proxy_pass http://tomcat_webserver;
		proxy_set_header Host $host;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header X-Real-IP $remote_addr;
	}
	error_page 500 502 503 504 /50x.html;
	location = /50x.html {
		root html;
		}
	}
}
```

#### 7.5.2.4 准备 tomcat 页面文件

```bash
[root@docker-server3 docker-compose]$ ll /apps/tomcat/webapps/SalesManager
total 8
-rw-r--r-- 1 www www 15 Dec 21 05:01 index.html
-rw-r--r-- 1 www www 696 Dec 21 05:01 showhost.jsp

[root@docker-server3  docker-compose]$  cat
/apps/tomcat/webapps/SalesManager/showhost.jsp
<%@page import="java.util.Enumeration"%>
<br />
host:
<%try{out.println(""+java.net.InetAddress.getLocalHost().getHostName());}catch(Exc
eption e){}%>
<br />
remoteAddr: <%=request.getRemoteAddr()%>
<br />
remoteHost: <%=request.getRemoteHost()%>
<br />
sessionId: <%=request.getSession().getId()%>
<br />
serverName:
```



