由浅入深吃透 Docker

[toc]



# 基础概念与操作



## 01 | Docker 安装：入门案例带你了解容器技术原理



### Docker 能做什么？

众所周知，Docker 是一个用于开发，发布和运行应用程序的开放平台。通俗地讲，Docker 类似于集装箱。在一艘大船上，各种货物要想被整齐摆放并且相互不受到影响，我们就需要把各种货物进行集装箱标准化。有了集装箱，我们就不需要专门运输水果或者化学用品的船了。我们可以把各种货品通过集装箱打包，然后统一放到一艘船上运输。Docker 要做的就是把各种软件打包成一个集装箱（镜像），然后分发，且在运行的时候可以相互隔离。



### CentOS 下安装 Docker

Docker 是跨平台的解决方案，它支持在当前主流的各大平台安装，包括 Ubuntu、RHEL、CentOS、Debian 等 Linux 发行版，同时也可以在 OSX 、Microsoft Windows 等非 Linux 平台下安装使用。

因为 Linux 是 Docker 的原生支持平台，所以推荐你在 Linux 上使用 Docker。由于生产环境中我们使用 CentOS 较多，下面主要针对在 CentOS 平台下安装和使用 Docker 展开介绍。

#### 操作系统要求

要安装 Docker，我们需要 CentOS 7 及以上的发行版本。建议使用 **overlay2** 存储驱动程序。

#### 卸载已有 Docker

如果你已经安装过旧版的 Docker，可以先执行以下命令卸载旧版 Docker。

```bash
$ sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```



#### 安装 Docker

首次安装 Docker 之前，需要添加 Docker 安装源。添加之后，我们就可以从已经配置好的源，安装和更新 Docker。添加 Docker 安装源的命令如下：

```bash
$ sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
```



正常情况下，直接安装最新版本的 Docker 即可，因为最新版本的 Docker 有着更好的稳定性和安全性。你可以使用以下命令安装最新版本的 Docker。

```bash
$ sudo yum install docker-ce docker-ce-cli containerd.io
```



如果你想要安装指定版本的 Docker，可以使用以下命令：

```bash
$ sudo yum list docker-ce --showduplicates | sort -r
docker-ce.x86_64            18.06.1.ce-3.el7                   docker-ce-stable
docker-ce.x86_64            18.06.0.ce-3.el7                   docker-ce-stable
docker-ce.x86_64            18.03.1.ce-1.el7.centos            docker-ce-stable
docker-ce.x86_64            18.03.0.ce-1.el7.centos            docker-ce-stable
docker-ce.x86_64            17.12.1.ce-1.el7.centos            docker-ce-stable
docker-ce.x86_64            17.12.0.ce-1.el7.centos            docker-ce-stable
docker-ce.x86_64            17.09.1.ce-1.el7.centos            docker-ce-stable
```



然后选取想要的版本执行以下命令：

```bash
$ sudo yum install docker-ce-<VERSION_STRING> docker-ce-cli-<VERSION_STRING> containerd.io
```



安装完成后，使用以下命令启动 Docker。

```bash
$ sudo systemctl start docker
```



这里有一个国际惯例，安装完成后，我们需要使用以下命令启动一个 hello world 的容器。

```bash
$ sudo docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
0e03bdcc26d7: Pull complete
Digest: sha256:7f0a9f93b4aa3022c3a4c147a449bf11e0941a1fd0bf4a8e6c9408b2600777c5
Status: Downloaded newer image for hello-world:latest
Hello from Docker!
```



运行上述命令，Docker 首先会检查本地是否有`hello-world`这个镜像，如果发现本地没有这个镜像，Docker 就会去 Docker Hub 官方仓库下载此镜像，然后运行它。最后我们看到该镜像输出 "Hello from Docker!" 并退出。

> 安装完成后默认 docker 命令只能以 root 用户执行，如果想允许普通用户执行 docker 命令，需要执行以下命令 `sudo groupadd docker && sudo gpasswd -a ${USER} docker && sudo systemctl restart docker` ，执行完命令后，退出当前命令行窗口并打开新的窗口即可。



### 容器技术原理

提起容器就不得不说 chroot，因为 chroot 是最早的容器雏形。chroot 意味着切换根目录，有了 chroot 就意味着我们可以把任何目录更改为当前进程的根目录，这与容器非常相似，下面我们通过一个实例了解下 chroot。



#### chroot

什么是 chroot 呢？下面是 chroot 维基百科定义：

> chroot 是在 Unix 和 Linux 系统的一个操作，针对正在运作的软件行程和它的子进程，改变它外显的根目录。一个运行在这个环境下，经由 chroot 设置根目录的程序，它不能够对这个指定根目录之外的文件进行访问动作，不能读取，也不能更改它的内容。

通俗地说 ，chroot 就是可以改变某进程的根目录，使这个程序不能访问该目录之外的其他目录，这个跟我们在一个容器中是很相似的。下面我们通过一个实例来演示下 chroot。

首先我们在当前目录下创建一个 rootfs 目录：

```bash
$ mkdir rootfs
```



这里为了方便演示，我使用现成的 busybox 镜像来创建一个系统，镜像的概念和组成后面我会详细讲解，如果你没有 Docker 基础可以把下面的操作命令理解成在 rootfs 下创建了一些目录和放置了一些二进制文件。

```bash
$ cd rootfs 
$ docker export $(docker create busybox) -o busybox.tar
$ tar -xf busybox.tar
```



执行完上面的命令后，在 rootfs 目录下，我们会得到一些目录和文件。下面我们使用 ls 命令查看一下 rootfs 目录下的内容。

```bash
$ ls
bin  busybox.tar  dev  etc  home  proc  root  sys  tmp  usr  var
```



可以看到我们在 rootfs 目录下初始化了一些目录，下面让我们通过一条命令来见证 chroot 的神奇之处。使用以下命令，可以启动一个 sh 进程，并且把 /home/centos/rootfs 作为 sh 进程的根目录。

```bash
$ chroot /home/centos/rootfs /bin/sh
```



此时，我们的命令行窗口已经处于上述命令启动的 sh 进程中。在当前 sh 命令行窗口下，我们使用 ls 命令查看一下当前进程，看是否真的与主机上的其他目录隔离开了。

```bash
/ # /bin/ls /
bin  busybox.tar  dev  etc  home  proc  root  sys  tmp  usr  var
```



这里可以看到当前进程的根目录已经变成了主机上的 /home/centos/rootfs 目录。这样就实现了当前进程与主机的隔离。到此为止，一个目录隔离的容器就完成了。
但是，此时还不能称之为一个容器，为什么呢？你可以在上一步（使用 chroot 启动命令行窗口）执行以下命令，查看如下路由信息：

```sh
/etc # /bin/ip route
default via 172.20.1.1 dev eth0
172.17.0.0/16 dev docker0 scope link  src 172.17.0.1
172.20.1.0/24 dev eth0 scope link  src 172.20.1.3
```



执行 ip route 命令后，你可以看到网络信息并没有隔离，实际上进程等信息此时也并未隔离。要想实现一个完整的容器，我们还需要 Linux 的其他三项技术： **Namespace、Cgroups 和联合文件系统**。

**Docker 是利用 Linux 的 Namespace 、Cgroups 和联合文件系统三大机制来保证实现的， 所以它的原理是使用 Namespace 做主机名、网络、PID 等资源的隔离，使用 Cgroups 对进程或者进程组做资源（例如：CPU、内存等）的限制，联合文件系统用于镜像构建和容器运行环境。**



#### Namespace

Namespace 是 Linux 内核的一项功能，该功能对内核资源进行隔离，使得容器中的进程都可以在单独的命名空间中运行，并且只可以访问当前容器命名空间的资源。Namespace 可以隔离**进程 ID、主机名、用户 ID、文件名、网络访问和进程间通信等相关资源。**

Docker 主要用到以下五种命名空间。

- **pid namespace**：用于隔离进程 ID。
- **net namespace**：隔离网络接口，在虚拟的 net namespace 内用户可以拥有自己独立的 IP、路由、端口等。
- **mnt namespace**：文件系统挂载点隔离。
- **ipc namespace**：信号量,消息队列和共享内存的隔离。
- **uts namespace**：主机名和域名的隔离。



#### Cgroups

Cgroups 是一种 Linux 内核功能，可以限制和隔离进程的资源使用情况（CPU、内存、磁盘 I/O、网络等）。在容器的实现中，Cgroups 通常用来限制容器的 CPU 和内存等资源的使用。



#### 联合文件系统

联合文件系统，又叫 UnionFS，是一种通过创建文件层进程操作的文件系统，因此，联合文件系统非常轻快。Docker 使用联合文件系统为容器提供构建层，使得容器可以实现写时复制以及镜像的分层构建和存储。常用的联合文件系统有 AUFS、Overlay 和 Devicemapper 等。



### 结语

容器技术从 1979 年 chroot 的首次问世便已崭露头角，但是到了 2013 年，Dokcer 的横空出世才使得容器技术迅速发展，可见 Docker 对于容器技术的推动力和影响力。

> 另外， Docker 还提供了工具和平台来管理容器的生命周期：
>
> 1. 使用容器开发应用程序及其支持组件。
> 2. 容器成为分发和测试你的应用程序的单元。
> 3. 可以将应用程序作为容器或协调服务部署到生产环境中。无论您的生产环境是本地数据中心，云提供商还是两者的混合，其工作原理都相同。



到此，相信你已经了解了实现容器的基本技术原理，并且对 Docker 的作用有了一定认知。那么你知道为什么容器技术在 Docker 出现之前一直没有爆发的根本原因吗？

Docker 爆发的原因是加入了镜像功能，并且封装了镜像仓库使得镜像分发更加方便



## 02 | 核心概念：镜像、容器、仓库，彻底掌握 Docker 架构核心设计理念



Docker 的操作围绕镜像、容器、仓库三大核心概念。



### Docker 核心概念



#### 镜像

镜像是什么呢？通俗地讲，它是一个只读的文件和文件夹组合。它包含了容器运行时所需要的所有基础文件和配置信息，是容器启动的基础。所以你想启动一个容器，那首先必须要有一个镜像。**镜像是 Docker 容器启动的先决条件。**

如果你想要使用一个镜像，你可以用这两种方式：

1. 自己创建镜像。通常情况下，一个镜像是基于一个基础镜像构建的，你可以在基础镜像上添加一些用户自定义的内容。例如你可以基于`centos`镜像制作你自己的业务镜像，首先安装`nginx`服务，然后部署你的应用程序，最后做一些自定义配置，这样一个业务镜像就做好了。
2. 从功能镜像仓库拉取别人制作好的镜像。一些常用的软件或者系统都会有官方已经制作好的镜像，例如`nginx`、`ubuntu`、`centos`、`mysql`等，你可以到 [Docker Hub](https://hub.docker.com/) 搜索并下载它们。



#### 容器

容器是什么呢？容器是 Docker 的另一个核心概念。通俗地讲，容器是镜像的运行实体。镜像是静态的只读文件，而容器带有运行时需要的可写文件层，并且容器中的进程属于运行状态。即**容器运行着真正的应用进程。容器有初建、运行、停止、暂停和删除五种状态。**

虽然容器的本质是主机上运行的一个进程，但是容器有自己独立的命名空间隔离和资源限制。也就是说，在容器内部，无法看到主机上的进程、环境变量、网络等信息，这是容器与直接运行在主机上进程的本质区别。



#### 仓库

Docker 的镜像仓库类似于代码仓库，用来存储和分发 Docker 镜像。镜像仓库分为公共镜像仓库和私有镜像仓库。

目前，[Docker Hub](https://hub.docker.com/) 是 Docker 官方的公开镜像仓库，它不仅有很多应用或者操作系统的官方镜像，还有很多组织或者个人开发的镜像供我们免费存放、下载、研究和使用。除了公开镜像仓库，你也可以构建自己的私有镜像仓库。



#### 镜像、容器、仓库，三者之间的联系

![Drawing 1.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F9PYryALHVmAABihjRzo4c527.png)



从图 1 可以看到，镜像是容器的基石，容器是由镜像创建的。一个镜像可以创建多个容器，容器是镜像运行的实体。仓库就非常好理解了，就是用来存放和分发镜像的。

了解了 Docker 的三大核心概念，接下来认识下 Docker 的核心架构和一些重要的组件。



### Docker 架构

在了解 Docker 架构前，我先说下相关的背景知识——容器的发展史。

容器技术随着 Docker 的出现变得炙手可热，所有公司都在积极拥抱容器技术。此时市场上除了有 Docker 容器，还有很多其他的容器技术，比如 CoreOS 的 rkt、lxc 等。容器技术百花齐放是好事，但也出现了很多问题。比如容器技术的标准到底是什么？容器标准应该由谁来制定？

也许你可能会说， Docker 已经成为了事实标准，把 Docker 作为容器技术的标准不就好了？事实并没有想象的那么简单。因为那时候不仅有容器标准之争，编排技术之争也十分激烈。当时的编排技术有三大主力，分别是 Docker Swarm、Kubernetes 和 Mesos 。Swarm 毋庸置疑，肯定愿意把 Docker 作为唯一的容器运行时，但是 Kubernetes 和 Mesos 就不同意了，因为它们不希望调度的形式过度单一。

在这样的背景下，最终爆发了容器大战，`OCI`也正是在这样的背景下应运而生。

`OCI`全称为开放容器标准（Open Container Initiative），它是一个轻量级、开放的治理结构。`OCI`组织在 Linux 基金会的大力支持下，于 2015 年 6 月份正式注册成立。基金会旨在为用户围绕工业化容器的格式和镜像运行时，制定一个开放的容器标准。目前主要有两个标准文档：**容器运行时标准 （runtime spec）\**和\**容器镜像标准（image spec）**。

正是由于容器的战争，才导致 Docker 不得不在战争中改变一些技术架构。最终形成了下图所示的技术架构。

![Drawing 2.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F9PYtCAC1GSAADIK4E6wrc368.png)

我们可以看到，Docker 整体架构采用 C/S（客户端 / 服务器）模式，主要由客户端和服务端两大部分组成。客户端负责发送操作指令，服务端负责接收和处理指令。客户端和服务端通信有多种方式，既可以在同一台机器上通过`UNIX`套接字通信，也可以通过网络连接远程通信。



#### Docker 客户端

Docker 客户端其实是一种泛称。其中 docker 命令是 Docker 用户与 Docker 服务端交互的主要方式。除了使用 docker 命令的方式，还可以使用直接请求 REST API 的方式与 Docker 服务端交互，甚至还可以使用各种语言的 SDK 与 Docker 服务端交互。目前社区维护着 Go、Java、Python、PHP 等数十种语言的 SDK，足以满足你的日常需求。



#### Docker 服务端

Docker 服务端是 Docker 所有后台服务的统称。其中 dockerd 是一个非常重要的后台管理进程，它负责响应和处理来自 Docker 客户端的请求，然后将客户端的请求转化为 Docker 的具体操作。例如镜像、容器、网络和挂载卷等具体对象的操作和管理。

Docker 从诞生到现在，服务端经历了多次架构重构。起初，服务端的组件是全部集成在 docker 二进制里。但是从 1.11 版本开始， dockerd 已经成了独立的二进制，此时的容器也不是直接由 dockerd 来启动了，而是集成了 containerd、runC 等多个组件。

虽然 Docker 的架构在不停重构，但是各个模块的基本功能和定位并没有变化。它和一般的 C/S 架构系统一样，Docker 服务端模块负责和 Docker 客户端交互，并管理 Docker 的容器、镜像、网络等资源。



#### Docker 重要组件

下面，我以 Docker 的 18.09.2 版本为例，看下 Docker 都有哪些工具和组件。在 Docker 安装路径下执行 ls 命令可以看到以下与 docker 有关的二进制文件。

```sh
-rwxr-xr-x 1 root root 27941976 Dec 12  2019 containerd
-rwxr-xr-x 1 root root  4964704 Dec 12  2019 containerd-shim
-rwxr-xr-x 1 root root 15678392 Dec 12  2019 ctr
-rwxr-xr-x 1 root root 50683148 Dec 12  2019 docker
-rwxr-xr-x 1 root root   764144 Dec 12  2019 docker-init
-rwxr-xr-x 1 root root  2837280 Dec 12  2019 docker-proxy
-rwxr-xr-x 1 root root 54320560 Dec 12  2019 dockerd
-rwxr-xr-x 1 root root  7522464 Dec 12  2019 runc
```



可以看到，Docker 目前已经有了非常多的组件和工具。这里我不对它们逐一介绍，因为在第 11 课时，我会带你深入剖析每一个组件和工具。
这里我先介绍一下 Docker 的两个至关重要的组件：`runC`和`containerd`。

- `runC`是 Docker 官方按照 OCI 容器运行时标准的一个实现。通俗地讲，runC 是一个用来运行容器的轻量级工具，是真正用来运行容器的。
- `containerd`是 Docker 服务端的一个核心组件，它是从`dockerd`中剥离出来的 ，它的诞生完全遵循 OCI 标准，是容器标准化后的产物。`containerd`通过 containerd-shim 启动并管理 runC，可以说`containerd`真正管理了容器的生命周期。



![Drawing 3.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F9PYuuAQINxAAA236heaL0459.png)



通过上图，可以看到，`dockerd`通过 gRPC 与`containerd`通信，由于`dockerd`与真正的容器运行时，`runC`中间有了`containerd`这一 OCI 标准层，使得`dockerd`可以确保接口向下兼容。

> [gRPC](https://grpc.io/) 是一种远程服务调用。想了解更多信息可以参考[https://grpc.io](https://grpc.io/)
> containerd-shim 的意思是垫片，类似于拧螺丝时夹在螺丝和螺母之间的垫片。containerd-shim 的主要作用是将 containerd 和真正的容器进程解耦，使用 containerd-shim 作为容器进程的父进程，从而实现重启 dockerd 不影响已经启动的容器进程。



了解了 dockerd、containerd 和 runC 之间的关系，下面可以通过启动一个 Docker 容器，来验证它们进程之间的关系。



#### Docker 各组件之间的关系

首先通过以下命令来启动一个 busybox 容器：

```sh
$ docker run -d busybox sleep 3600
```



容器启动后，通过以下命令查看一下 dockerd 的 PID：

```sh
$ sudo ps aux |grep dockerd
root      4147  0.3  0.2 1447892 83236 ?       Ssl  Jul09 245:59 /usr/bin/dockerd
```



通过上面的输出结果可以得知 dockerd 的 PID 为 4147。为了验证图 3 中 Docker 各组件之间的调用关系，下面使用 pstree 命令查看一下进程父子关系

```bash
$ sudo pstree -l -a -A 4147
dockerd
  |-containerd --config /var/run/docker/containerd/containerd.toml --log-level info
  |   |-containerd-shim -namespace moby -workdir /var/lib/docker/containerd/daemon/io.containerd.runtime.v1.linux/moby/d14d20507073e5743e607efd616571c834f1a914f903db6279b8de4b5ba3a45a -address /var/run/docker/containerd/containerd.sock -containerd-binary /usr/bin/containerd -runtime-root /var/run/docker/runtime-runc
  |   |   |-sleep 3600
```



事实上，dockerd 启动的时候， containerd 就随之启动了，dockerd 与 containerd 一直存在。当执行 docker run 命令（通过 busybox 镜像创建并启动容器）时，containerd 会创建 containerd-shim 充当 “垫片”进程，然后启动容器的真正进程 sleep 3600 。这个过程和架构图是完全一致的。





## 03 | 镜像使用：Docker 环境下如何配置你的镜像？



镜像是一个只读的 Docker 容器模板，包含启动容器所需要的所有文件系统结构和内容。简单来讲，镜像是一个特殊的文件系统，它提供了容器运行时所需的程序、软件库、资源、配置等静态数据。即**镜像不包含任何动态数据，镜像内容在构建后不会被改变**。



### 镜像操作

![Lark20200904-175130.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl9SDkWAaxh7AAFaMgWI7cI029.png)



从图中可知，镜像的操作可分为：

- 拉取镜像，使用`docker pull`命令拉取远程仓库的镜像到本地 ；
- 重命名镜像，使用`docker tag`命令“重命名”镜像 ；
- 查看镜像，使用`docker image ls`或`docker images`命令查看本地已经存在的镜像 ；
- 删除镜像，使用`docker rmi`命令删除无用镜像 ；
- 构建镜像，构建镜像有两种方式。第一种方式是使用`docker build`命令基于 Dockerfile 构建镜像，也是我比较推荐的镜像构建方式；第二种方式是使用`docker commit`命令基于已经运行的容器提交为镜像。



#### 拉取镜像

Docker 镜像的拉取使用`docker pull`命令， 命令格式一般为 `docker pull [Registry]/[Repository]/[Image]:[Tag]`。

- Registry 为注册服务器，Docker 默认会从 docker.io 拉取镜像，如果你有自己的镜像仓库，可以把 Registry 替换为自己的注册服务器。
- Repository 为镜像仓库，通常把一组相关联的镜像归为一个镜像仓库，`library`为 Docker 默认的镜像仓库。
- Image 为镜像名称。
- Tag 为镜像的标签，如果你不指定拉取镜像的标签，默认为`latest`。



例如，我们需要获取一个 busybox 镜像，可以执行以下命令：

> busybox 是一个集成了数百个 Linux 命令（例如 curl、grep、mount、telnet 等）的精简工具箱，只有几兆大小，被誉为 Linux 系统的瑞士军刀。我经常会使用 busybox 做调试来查找生产环境中遇到的问题。

```sh
$ docker pull busybox
Using default tag: latest
latest: Pulling from library/busybox
61c5ed1cbdf8: Pull complete
Digest: sha256:4f47c01fa91355af2865ac10fef5bf6ec9c7f42ad2321377c21e844427972977
Status: Downloaded newer image for busybox:latest
docker.io/library/busybox:latest
```



实际上执行`docker pull busybox`命令，都是先从本地搜索，如果本地搜索不到`busybox`镜像则从 Docker Hub 下载镜像。

拉取完镜像，如果你想查看镜像，应该怎么操作呢？



#### 查看镜像

Docker 镜像查看使用`docker images`或者`docker image ls`命令。

下面我们使用`docker images`命令列出本地所有的镜像。

```sh
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
nginx               latest              4bb46517cac3        9 days ago          133MB
nginx               1.15                53f3fd8007f7        15 months ago       109MB
busybox             latest              018c9d7b792b        3 weeks ago         1.22MB
```



如果我们想要查询指定的镜像，可以使用`docker image ls`命令来查询。

```sh
$ docker image ls busybox
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
busybox             latest              018c9d7b792b        3 weeks ago         1.22MB
```



当然你也可以使用`docker images`命令列出所有镜像，然后使用`grep`命令进行过滤。使用方法如下：

```sh
$ docker images |grep busybox
busybox             latest              018c9d7b792b        3 weeks ago         1.22MB
```



#### “重命名”镜像

如果你想要自定义镜像名称或者推送镜像到其他镜像仓库，你可以使用`docker tag`命令将镜像重命名。`docker tag`的命令格式为 `docker tag [SOURCE_IMAGE][:TAG] [TARGET_IMAGE][:TAG]`。

```sh
$ docker tag busybox:latest mybusybox:latest
```



执行完`docker tag`命令后，可以使用查询镜像命令查看一下镜像列表：

```sh
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
busybox             latest              018c9d7b792b        3 weeks ago         1.22MB
mybusybox           latest              018c9d7b792b        3 weeks ago         1.22MB
```

可以看到，镜像列表中多了一个`mybusybox`的镜像。但细心的同学可能已经发现，`busybox`和`mybusybox`这两个镜像的 IMAGE ID 是完全一样的。为什么呢？实际上它们指向了同一个镜像文件，只是别名不同而已。
如果我不需要`mybusybox`镜像了，想删除它，应该怎么操作呢？



#### 删除镜像

你可以使用`docker rmi`或者`docker image rm`命令删除镜像。

举例：你可以使用以下命令删除`mybusybox`镜像。

```sh
$ docker rmi mybusybox
Untagged: mybusybox:latest
```



此时，再次使用`docker images`命令查看一下我们机器上的镜像列表。

```sh
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
busybox             latest              018c9d7b792b        3 weeks ago         1.22MB
```



通过上面的输出，我们可以看到，`mybusybox`镜像已经被删除。
如果你想构建属于自己的镜像，应该怎么做呢？



#### 构建镜像

构建镜像主要有两种方式：

1. 使用`docker commit`命令从运行中的容器提交为镜像；
2. 使用`docker build`命令从 Dockerfile 构建镜像。

首先介绍下如何从运行中的容器提交为镜像。我依旧使用 busybox 镜像举例，使用以下命令创建一个名为 busybox 的容器并进入 busybox 容器。

```sh
$ docker run --rm --name=busybox -it busybox sh
/ #
```



执行完上面的命令后，当前窗口会启动一个 busybox 容器并且进入容器中。在容器中，执行以下命令创建一个文件并写入内容:

```sh
/ # touch hello.txt && echo "I love Docker. " > hello.txt
/ #
```



此时在容器的根目录下，已经创建了一个 hello.txt 文件，并写入了 "I love Docker. "。下面，我们新打开另一个命令行窗口，运行以下命令提交镜像：

```sh
$ docker commit busybox busybox:hello
sha256:cbc6406aaef080d1dd3087d4ea1e6c6c9915ee0ee0f5dd9e0a90b03e2215e81c
```



然后使用上面讲到的`docker image ls`命令查看镜像：

```sh
$ docker image ls busybox
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
busybox             hello               cbc6406aaef0        2 minutes ago       1.22MB
busybox             latest              018c9d7b792b        4 weeks ago         1.22MB
```



此时我们可以看到主机上新生成了 busybox:hello 这个镜像。

第二种方式是最重要也是最常用的镜像构建方式：Dockerfile。Dockerfile 是一个包含了用户所有构建命令的文本。通过`docker build`命令可以从 Dockerfile 生成镜像。

使用 Dockerfile 构建镜像具有以下特性：

- Dockerfile 的每一行命令都会生成一个独立的镜像层，并且拥有唯一的 ID；
- Dockerfile 的命令是完全透明的，通过查看 Dockerfile 的内容，就可以知道镜像是如何一步步构建的；
- Dockerfile 是纯文本的，方便跟随代码一起存放在代码仓库并做版本管理。



看到使用 Dockerfile 的方式构建镜像有这么多好的特性，你是不是已经迫不及待想知道如何使用了。别着急，我们先学习下 Dockerfile 常用的指令。

| Dockerfile 指令 | 指令简介                                                     |
| --------------- | ------------------------------------------------------------ |
| FROM            | Dockerfile 除了注释第一行必须是 FROM ，FROM 后面跟镜像名称，代表我们要基于哪个基础镜像构建我们的容器。 |
| RUN             | RUN 后面跟一个具体的命令，类似于 Linux 命令行执行命令。      |
| ADD             | 拷贝本机文件或者远程文件到镜像内                             |
| COPY            | 拷贝本机文件到镜像内                                         |
| USER            | 指定容器启动的用户                                           |
| ENTRYPOINT      | 容器的启动命令                                               |
| CMD             | CMD 为 ENTRYPOINT 指令提供默认参数，也可以单独使用 CMD 指定容器启动参数 |
| ENV             | 指定容器运行时的环境变量，格式为 key=value                   |
| ARG             | 定义外部变量，构建镜像时可以使用 build-arg = 的格式传递参数用于构建 |
| EXPOSE          | 指定容器监听的端口，格式为 [port]/tcp 或者 [port]/udp        |
| WORKDIR         | 为 Dockerfile 中跟在其后的所有 RUN、CMD、ENTRYPOINT、COPY 和 ADD 命令设置工作目录。 |



看了这么多指令，感觉有点懵？别担心，我通过一个实例让你来熟悉它们。这是一个 Dockerfile：

```sh
FROM centos:7
COPY nginx.repo /etc/yum.repos.d/nginx.repo
RUN yum install -y nginx
EXPOSE 80
ENV HOST=mynginx
CMD ["nginx","-g","daemon off"]
```



- 第一行表示我要基于 centos:7 这个镜像来构建自定义镜像。这里需要注意，每个 Dockerfile 的第一行除了注释都必须以 FROM 开头。
- 第二行表示拷贝本地文件 nginx.repo 文件到容器内的 /etc/yum.repos.d 目录下。这里拷贝 nginx.repo 文件是为了添加 nginx 的安装源。
- 第三行表示在容器内运行`yum install -y nginx`命令，安装 nginx 服务到容器内，执行完第三行命令，容器内的 nginx 已经安装完成。
- 第四行声明容器内业务（nginx）使用 80 端口对外提供服务。
- 第五行定义容器启动时的环境变量 HOST=mynginx，容器启动后可以获取到环境变量 HOST 的值为 mynginx。
- 第六行定义容器的启动命令，命令格式为 json 数组。这里设置了容器的启动命令为 nginx ，并且添加了 nginx 的启动参数 -g 'daemon off;' ，使得 nginx 以前台的方式启动。



上面这个 Dockerfile 的例子基本涵盖了常用的镜像构建指令，代码我已经放在 [GitHub](https://github.com/wilhelmguo/docker-demo/tree/master/dockerfiles)上，如果你感兴趣可以到 [GitHub 下载源码](https://github.com/wilhelmguo/docker-demo/tree/master/dockerfiles)并尝试构建这个镜像。



### 镜像的实现原理

其实 Docker 镜像是由一系列镜像层（layer）组成的，每一层代表了镜像构建过程中的一次提交。下面以一个镜像构建的 Dockerfile 来说明镜像是如何分层的。

```sh
FROM busybox
COPY test /tmp/test
RUN mkdir /tmp/testdir
```



上面的 Dockerfile 由三步组成：

第一行基于 busybox 创建一个镜像层；

第二行拷贝本机 test 文件到镜像内；

第三行在 /tmp 文件夹下创建一个目录 testdir。

为了验证镜像的存储结构，我们使用`docker build`命令在上面 Dockerfile 所在目录构建一个镜像：

```sh
$ docker build -t mybusybox .
```



这里我的 Docker 使用的是 overlay2 文件驱动，进入到`/var/lib/docker/overlay2`目录下使用`tree .`命令查看产生的镜像文件：

```sh
$ tree .
# 以下为 tree . 命令输出内容
|-- 3e89b959f921227acab94f5ab4524252ae0a829ff8a3687178e3aca56d605679
|   |-- diff  # 这一层为基础层，对应上述 Dockerfile 第一行，包含 busybox 镜像所有文件内容，例如 /etc,/bin,/var 等目录
... 此次省略部分原始镜像文件内容
|   `-- link 
|-- 6591d4e47eb2488e6297a0a07a2439f550cdb22845b6d2ddb1be2466ae7a9391
|   |-- diff   # 这一层对应上述 Dockerfile 第二行，拷贝 test 文件到 /tmp 文件夹下，因此 diff 文件夹下有了 /tmp/test 文件
|   |   `-- tmp
|   |       `-- test
|   |-- link
|   |-- lower
|   `-- work
|-- backingFsBlockDev
|-- bec6a018080f7b808565728dee8447b9e86b3093b16ad5e6a1ac3976528a8bb1
|   |-- diff  # 这一层对应上述 Dockerfile 第三行，在 /tmp 文件夹下创建 testdir 文件夹，因此 diff 文件夹下有了 /tmp/testdir 文件夹
|   |   `-- tmp
|   |       `-- testdir
|   |-- link
|   |-- lower
|   `-- work
...
```



通过上面的目录结构可以看到，Dockerfile 的每一行命令，都生成了一个镜像层，每一层的 diff 夹下只存放了增量数据，如图 2 所示。

![Lark20200904-175137.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl9SDmGACBEjAABkgtnn_hE625.png)



分层的结构使得 Docker 镜像非常轻量，每一层根据镜像的内容都有一个唯一的 ID 值，当不同的镜像之间有相同的镜像层时，便可以实现不同的镜像之间共享镜像层的效果。

总结一下， Docker 镜像是静态的分层管理的文件组合，镜像底层的实现依赖于联合文件系统（UnionFS）。充分掌握镜像的原理，可以帮助我们在生产实践中构建出最优的镜像，同时也可以帮助我们更好地理解容器和镜像的关系。



#### 总结

到此，相信你已经对 Docker 镜像这一核心概念有了较深的了解，并熟悉了 Docker 镜像的常用操作（拉取、查看、“重命名”、删除和构建自定义镜像）及底层实现原理。

镜像操作命令：

1. 拉取镜像，使用 `docker pull` 命令拉取远程仓库的镜像到本地 ；
2. 重命名镜像，使用 `docker tag` 命令“重命名”镜像 ；
3. 查看镜像，使用 `docker image ls` 或 `docker images` 命令查看本地已经存在的镜像；
4. 删除镜像，使用 `docker rmi` 命令删除无用镜像 ；
5. 构建镜像，构建镜像有两种方式。第一种方式是使用 `docker build` 命令基于 Dockerfile 构建镜像，也是我比较推荐的镜像构建方式；第二种方式是使用 `docker commit` 命令基于已经运行的容器提交为镜像。



镜像的实现原理：
镜像是由一系列的镜像层（layer ）组成，每一层代表了镜像构建过程中的一次提交，当我们需要修改镜像内的某个文件时，只需要在当前镜像层的基础上新建一个镜像层，并且只存放修改过的文件内容。分层结构使得镜像间共享镜像层变得非常简单和方便。



## 04 | 容器操作：得心应手掌握 Docker 容器基本操作



### 容器（Container）是什么？

容器是基于镜像创建的可运行实例，并且单独存在，一个镜像可以创建出多个容器。运行容器化环境时，实际上是在容器内部创建该文件系统的读写副本。 这将添加一个容器层，该层允许修改镜像的整个副本。如图 1 所示。

![image.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl9YmlSAGgF0AABXUH--rM4624.png)



了解完容器是什么，接下来我们聊一聊容器的生命周期。



### 容器的生命周期

容器的生命周期是容器可能处于的状态，容器的生命周期分为 5 种。

1. created：初建状态
2. running：运行状态
3. stopped：停止状态
4. paused： 暂停状态
5. deleted：删除状态



各生命周期之前的转换关系如图所示：

![Lark20200923-114857.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl9qxcuANmQGAADHS_nfwJE810.png)



通过`docker create`命令生成的容器状态为初建状态，初建状态通过`docker start`命令可以转化为运行状态，运行状态的容器可以通过`docker stop`命令转化为停止状态，处于停止状态的容器可以通过`docker start`转化为运行状态，运行状态的容器也可以通过`docker pause`命令转化为暂停状态，处于暂停状态的容器可以通过`docker unpause`转化为运行状态 。处于初建状态、运行状态、停止状态、暂停状态的容器都可以直接删除。

下面我通过实际操作和命令来讲解容器各生命周期间的转换关系。



### 容器的操作

容器的操作可以分为五个步骤：创建并启动容器、终止容器、进入容器、删除容器、导入和导出容器。下面我们逐一来看。



#### （1）创建并启动容器

容器十分轻量，用户可以随时创建和删除它。我们可以使用`docker create`命令来创建容器，例如：

```sh
$ docker create -it --name=busybox busybox
Unable to find image 'busybox:latest' locally
latest: Pulling from library/busybox
61c5ed1cbdf8: Pull complete
Digest: sha256:4f47c01fa91355af2865ac10fef5bf6ec9c7f42ad2321377c21e844427972977
Status: Downloaded newer image for busybox:latest
2c2e919c2d6dad1f1712c65b3b8425ea656050bd5a0b4722f8b01526d5959ec6
$ docker ps -a| grep busybox
2c2e919c2d6d        busybox             "sh"       
```



如果使用`docker create`命令创建的容器处于停止状态，我们可以使用`docker start`命令来启动它，如下所示。

```sh
$ docker start busybox
$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
d6f3d364fad3        busybox             "sh"                16 seconds ago      Up 8 seconds                            busybox
```



这时候我们可以看到容器已经处于启动状态了。
容器启动有两种方式：

1. 使用`docker start`命令基于已经创建好的容器直接启动 。
2. 使用`docker run`命令直接基于镜像新建一个容器并启动，相当于先执行`docker create`命令从镜像创建容器，然后再执行`docker start`命令启动容器。



使用`docker run`的命令如下:

```sh
$ docker run -it --name=busybox busybox
```



当使用`docker run`创建并启动容器时，Docker 后台执行的流程为：

- Docker 会检查本地是否存在 busybox 镜像，如果镜像不存在则从 Docker Hub 拉取 busybox 镜像；
- 使用 busybox 镜像创建并启动一个容器；
- 分配文件系统，并且在镜像只读层外创建一个读写层；
- 从 Docker IP 池中分配一个 IP 给容器；
- 执行用户的启动命令运行镜像。



上述命令中， -t 参数的作用是分配一个伪终端，-i 参数则可以终端的 STDIN 打开，同时使用 -it 参数可以让我们进入交互模式。 在交互模式下，用户可以通过所创建的终端来输入命令，例如：

```bash
$ ps aux
PID   USER     TIME  COMMAND
    1 root      0:00 sh
    6 root      0:00 ps aux
```



我们可以看到容器的 1 号进程为 sh 命令，在容器内部并不能看到主机上的进程信息，因为容器内部和主机是完全隔离的。同时由于 sh 是 1 号进程，意味着如果通过 exit 退出 sh，那么容器也会退出。所以对于容器来说，**杀死容器中的主进程，则容器也会被杀死。**



#### （2）终止容器

容器启动后，如果我们想停止运行中的容器，可以使用`docker stop`命令。命令格式为 docker stop [-t|--time[=10]]。该命令首先会向运行中的容器发送 SIGTERM 信号，如果容器内 1 号进程接受并能够处理 SIGTERM，则等待 1 号进程处理完毕后退出，如果等待一段时间后，容器仍然没有退出，则会发送 SIGKILL 强制终止容器。

```sh
$ docker stop busybox
busybox
```



如果你想查看停止状态的容器信息，你可以使用 docker ps -a 命令。

```sh
$ docker ps -a
CONTAINERID       IMAGE      COMMAND            CREATED             STATUS     PORTS         NAMES
28d477d3737a        busybox             "sh"                26 minutes ago      Exited (137) About a minute ago                       busybox
```



处于终止状态的容器也可以通过`docker start`命令来重新启动。

```sh
$ docker start busybox
busybox

$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
28d477d3737a        busybox             "sh"                30 minutes ago      Up 25 seconds                           busybox
```



此外，`docker restart`命令会将一个运行中的容器终止，并且重新启动它。

```sh
$ docker restart busybox
busybox

$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
28d477d3737a        busybox             "sh"                32 minutes ago      Up 3 seconds                            busybox
```



#### （3）进入容器

处于运行状态的容器可以通过`docker attach`、`docker exec`、`nsenter`等多种方式进入容器。

- **使用**`docker attach`命令**进入容器**



使用 docker attach ，进入我们上一步创建好的容器，如下所示。

```sh
$ docker attach busybox
/ # ps aux
PID   USER     TIME  COMMAND
    1 root      0:00 sh
    7 root      0:00 ps aux
/ #
```



注意：当我们同时使用`docker attach`命令同时在多个终端运行时，所有的终端窗口将同步显示相同内容，当某个命令行窗口的命令阻塞时，其他命令行窗口同样也无法操作。
由于`docker attach`命令不够灵活，因此我们一般不会使用`docker attach`进入容器。下面我介绍一个更加灵活的进入容器的方式`docker exec`

- **使用 docker exec 命令进入容器**



Docker 从 1.3 版本开始，提供了一个更加方便地进入容器的命令`docker exec`，我们可以通过`docker exec -it CONTAINER`的方式进入到一个已经运行中的容器，如下所示。

```sh
$ docker exec -it busybox sh
/ # ps aux
PID   USER     TIME  COMMAND
    1 root      0:00 sh
    7 root      0:00 sh
   12 root      0:00 ps aux
```



我们进入容器后，可以看到容器内有两个`sh`进程，这是因为以`exec`的方式进入容器，会单独启动一个 sh 进程，每个窗口都是独立且互不干扰的，也是使用最多的一种方式。



#### （4）删除容器

我们已经掌握了用 Docker 命令创建、启动和终止容器。那如何删除处于终止状态或者运行中的容器呢？删除容器命令的使用方式如下：`docker rm [OPTIONS] CONTAINER [CONTAINER...]`。

如果要删除一个停止状态的容器，可以使用`docker rm`命令删除。

```sh
$ docker rm busybox
```



如果要删除正在运行中的容器，必须添加 **-f (或 --force) 参数**， Docker 会发送 SIGKILL 信号强制终止正在运行的容器。

```sh
$ docker rm -f busybox
```



#### （5）导出导入容器

- **导出容器**

我们可以使用`docker export CONTAINER`命令导出一个容器到文件，不管此时该容器是否处于运行中的状态。导出容器前我们先进入容器，创建一个文件，过程如下。

首先进入容器创建文件

```sh
docker exec -it busybox sh
cd /tmp && touch test
```



然后执行导出命令

```sh
docker export busybox > busybox.tar
```



执行以上命令后会在当前文件夹下生成 busybox.tar 文件，我们可以将该文件拷贝到其他机器上，通过导入命令实现容器的迁移。

- **导入容器**

通过`docker export`命令导出的文件，可以使用`docker import`命令导入，执行完`docker import`后会变为本地镜像，最后再使用`docker run`命令启动该镜像，这样我们就实现了容器的迁移。 

导入容器的命令格式为 `docker import [OPTIONS] file|URL [REPOSITORY[:TAG]]`。接下来我们一步步将上一步导出的镜像文件导入到其他机器的 Docker 中并启动它。

首先，使用`docker import`命令导入上一步导出的容器

```sh
docker import busybox.tar busybox:test
```



此时，busybox.tar 被导入成为新的镜像，镜像名称为 busybox:test 。下面，我们使用`docker run`命令启动并进入容器，查看上一步创建的临时文件

```sh
docker run -it busybox:test sh
/ # ls /tmp/
test
```



可以看到我们之前在 /tmp 目录下创建的 test 文件也被迁移过来了。这样我们就通过`docker export`和`docker import`命令配合实现了容器的迁移。



### 结语

镜像包含了容器运行所需要的文件系统结构和内容，是静态的只读文件，而容器则是在镜像的只读层上创建了可写层，并且容器中的进程属于运行状态，容器是真正的应用载体。



那你知道为什么容器的文件系统要设计成写时复制(如图 1 所示)，而不是每一个容器都单独拷贝一份镜像文件吗？

每个容器单独拷贝一份镜像文件会占用较多的磁盘空间。假设我有3个程序都用到了jdk镜像，那该jdk镜像就得复制3份。如果改成写时复制的话，只需要一份jdk镜像，新的内容写在顶层即可。比如用docker images查看镜像的时候，把所有镜像的SIZE加起来可能有十几个G，但是到目录下查看，实际上只有几个G，这是因为很多镜像复用了同一个底层的镜像，起到了节省磁盘空间的作用。





## 05 | 仓库访问：怎样搭建属于你的私有仓库？



### 仓库是什么？

仓库（Repository）是存储和分发 Docker 镜像的地方。镜像仓库类似于代码仓库，Docker Hub 的命名来自 GitHub，Github 是我们常用的代码存储和分发的地方。同样 Docker Hub 是用来提供 Docker 镜像存储和分发的地方。

有的同学可能经常分不清注册服务器（Registry）和仓库（Repository）的概念。在这里我可以解释下这两个概念的区别：注册服务器是存放仓库的实际服务器，而仓库则可以被理解为一个具体的项目或者目录；注册服务器可以包含很多个仓库，每个仓库又可以包含多个镜像。例如我的镜像地址为 docker.io/centos，docker.io 是注册服务器，centos 是仓库名。 它们之间的关系如图 1 所示。

![Lark20200911-162223.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F9bM-uAI6MDAADk1noY7ic639.png)



按照类型，我们将镜像仓库分为公共镜像仓库和私有镜像仓库。



### 公共镜像仓库

公共镜像仓库一般是 Docker 官方或者其他第三方组织（阿里云，腾讯云，网易云等）提供的，允许所有人注册和使用的镜像仓库。

Docker Hub 是全球最大的镜像市场，目前已经有超过 10w 个容器镜像，这些容器镜像主要来自软件供应商、开源组织和社区。大部分的操作系统镜像和软件镜像都可以直接在 Docker Hub 下载并使用。

![Drawing 1.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F9bL9yAYd_LAAJW9Q4Ue2w855.png)



下面我以 Docker Hub 为例，教你如何使用公共镜像仓库分发和存储镜像。



#### 注册 Docker Hub 账号

我们首先访问[Docker Hub](https://hub.docker.com/)官网，点击注册按钮进入注册账号界面。

![Drawing 2.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl9bL-aABPLiAABcwVxClDY261.png)



注册完成后，我们可以点击创建仓库，新建一个仓库用于推送镜像。

![Drawing 3.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F9bL--AYVIKAADWoafHnho359.png)



这里我的账号为 lagoudocker，创建了一个名称为 busybox 的仓库，创建好仓库后我们就可以推送本地镜像到这个仓库里了。下面我通过一个实例来演示一下如何推送镜像到自己的仓库中。

首先我们使用以下命令拉取 busybox 镜像：

```sh
$ docker pull busybox
Using default tag: latest
latest: Pulling from library/busybox
Digest: sha256:4f47c01fa91355af2865ac10fef5bf6ec9c7f42ad2321377c21e844427972977
Status: Image is up to date for busybox:latest
docker.io/library/busybox:latest
```



在推送镜像仓库前，我们需要使用`docker login`命令先登录一下镜像服务器，因为只有已经登录的用户才可以推送镜像到仓库。

```sh
$ docker login
Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username: lagoudocker
Password:
Login Succeeded
```



使用`docker login`命令登录镜像服务器，这时 Docker 会要求我们输入用户名和密码，输入我们刚才注册的账号和密码，看到`Login Succeeded`表示登录成功。登录成功后就可以推送镜像到自己创建的仓库了。

> `docker login`命令默认会请求 Docker Hub，如果你想登录第三方镜像仓库或者自建的镜像仓库，在`docker login`后面加上注册服务器即可。例如我们想登录访问阿里云镜像服务器，则使用`docker login registry.cn-beijing.aliyuncs.com`，输入阿里云镜像服务的用户名密码即可。



在本地镜像推送到自定义仓库前，我们需要先把镜像“重命名”一下，才能正确推送到自己创建的镜像仓库中，使用`docker tag`命令将镜像“重命名”：

```sh
$ docker tag busybox lagoudocker/busybox
```



镜像“重命名”后使用`docker push`命令就可以推送镜像到自己创建的仓库中了。

```sh
$ docker push lagoudocker/busybox
The push refers to repository [docker.io/lagoudocker/busybox]
514c3a3e64d4: Mounted from library/busybox
latest: digest: sha256:400ee2ed939df769d4681023810d2e4fb9479b8401d97003c710d0e20f7c49c6 size: 527
```



此时，`busybox`这个镜像就被推送到自定义的镜像仓库了。这里我们也可以新建其他的镜像仓库，然后把自己构建的镜像推送到仓库中。
有时候，出于安全或保密的需求，你可能想要搭建一个自己的镜像仓库，下面我带你一步一步构建一个私有的镜像仓库。



### 搭建私有仓库

#### 启动本地仓库

Docker 官方提供了开源的镜像仓库 [Distribution](https://github.com/docker/distribution)，并且镜像存放在 Docker Hub 的 [Registry](https://hub.docker.com/_/registry) 仓库下供我们下载。

我们可以使用以下命令启动一个本地镜像仓库：

```sh
$ docker run -d -p 5000:5000 --name registry registry:2.7
Unable to find image 'registry:2.7' locally
2.7: Pulling from library/registry
cbdbe7a5bc2a: Pull complete
47112e65547d: Pull complete
46bcb632e506: Pull complete
c1cc712bcecd: Pull complete
3db6272dcbfa: Pull complete
Digest: sha256:8be26f81ffea54106bae012c6f349df70f4d5e7e2ec01b143c46e2c03b9e551d
Status: Downloaded newer image for registry:2.7
d7e449a8a93e71c9a7d99c67470bd7e7a723eee5ae97b3f7a2a8a1cf25982cc3
```



使用`docker ps`命令查看一下刚才启动的容器：

```sh
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
d7e449a8a93e        registry:2.7        "/entrypoint.sh /etc…"   50 seconds ago      Up 49 seconds       0.0.0.0:5000->5000/tcp   registry
```



此时我们就拥有了一个私有镜像仓库，访问地址为`localhost`，端口号为 5000。



#### 推送镜像到本地仓库

我们依旧使用 busybox 镜像举例。首先我们使用`docker tag`命令把 busybox 镜像"重命名"为`localhost:5000/busybox`

```sh
$ docker tag busybox localhost:5000/busybox
```



此时 Docker 为`busybox`镜像创建了一个别名`localhost:5000/busybox`，`localhost:5000`为主机名和端口，Docker 将会把镜像推送到这个地址。
使用`docker push`推送镜像到本地仓库：

```sh
$ docker push localhost:5000/busybox
The push refers to repository [localhost:5000/busybox]
514c3a3e64d4: Layer already exists
latest: digest: sha256:400ee2ed939df769d4681023810d2e4fb9479b8401d97003c710d0e20f7c49c6 size: 527
```



这里可以看到，我们已经可以把`busybox`推送到了本地镜像仓库。

此时，我们验证一下从本地镜像仓库拉取镜像。首先，我们删除本地的`busybox`和`localhost:5000/busybox`镜像。

```sh
$ docker rmi busybox localhost:5000/busybox
Untagged: busybox:latest
Untagged: busybox@sha256:4f47c01fa91355af2865ac10fef5bf6ec9c7f42ad2321377c21e844427972977
Untagged: localhost:5000/busybox:latest
Untagged: localhost:5000/busybox@sha256:400ee2ed939df769d4681023810d2e4fb9479b8401d97003c710d0e20f7c49c6
```



查看一下本地`busybox`镜像：

```sh
$ docker image ls busybox
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
```



可以看到此时本地已经没有`busybox`这个镜像了。下面，我们从本地镜像仓库拉取`busybox`镜像：

```sh
$ docker pull localhost:5000/busybox
Using default tag: latest
latest: Pulling from busybox
Digest: sha256:400ee2ed939df769d4681023810d2e4fb9479b8401d97003c710d0e20f7c49c6
Status: Downloaded newer image for localhost:5000/busybox:latest
localhost:5000/busybox:latest
```



然后再使用`docker image ls busybox`命令，这时可以看到我们已经成功从私有镜像仓库拉取`busybox`镜像到本地了



#### 持久化镜像存储

我们知道，容器是无状态的。上面私有仓库的启动方式可能会导致镜像丢失，因为我们并没有把仓库的数据信息持久化到主机磁盘上，这在生产环境中是无法接受的。下面我们使用以下命令将镜像持久化到主机目录：

```sh
$ docker run -v /var/lib/registry/data:/var/lib/registry -d -p 5000:5000 --name registry registry:2.7
```



我们在上面启动`registry`的命令中加入了`-v /var/lib/registry/data:/var/lib/registry`，`-v`的含义是把 Docker 容器的某个目录或文件挂载到主机上，保证容器被重建后数据不丢失。`-v`参数冒号前面为主机目录，冒号后面为容器内目录。

> 事实上，registry 的持久化存储除了支持本地文件系统还支持很多种类型，例如 S3、Google Cloud Platform、Microsoft Azure Blob Storage Service 等多种存储类型。



到这里我们的镜像仓库虽然可以本地访问和拉取，但是如果你在另外一台机器上是无法通过 Docker 访问到这个镜像仓库的，因为 Docker 要求非`localhost`访问的镜像仓库必须使用 HTTPS，这时候就需要构建外部可访问的镜像仓库。



#### 构建外部可访问的镜像仓库

要构建一个支持 HTTPS 访问的安全镜像仓库，需要满足以下两个条件：

- 拥有一个合法的域名，并且可以正确解析到镜像服务器；
- 从证书颁发机构（CA）获取一个证书。



在准备好域名和证书后，就可以部署我们的镜像服务器了。这里我以`regisry.lagoudocker.io`这个域名为例。首先准备存放证书的目录`/var/lib/registry/certs`，然后把申请到的证书私钥和公钥分别放到该目录下。 假设我们申请到的证书文件分别为`regisry.lagoudocker.io.crt`和`regisry.lagoudocker.io.key`。

如果上一步启动的仓库容器还在运行，我们需要先停止并删除它。

```sh
$ docker stop registry && docker rm registry
```



然后使用以下命令启动新的镜像仓库：

```sh
$ docker run -d \
  --name registry \
  -v "/var/lib/registry/data:/var/lib/registry \
  -v "/var/lib/registry/certs:/certs \
  -e REGISTRY_HTTP_ADDR=0.0.0.0:443 \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/regisry.lagoudocker.io.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/regisry.lagoudocker.io.key \
  -p 443:443 \
  registry:2.7
```



这里，我们使用 -v 参数把镜像数据持久化在`/var/lib/registry/data`目录中，同时把主机上的证书文件挂载到了容器的 /certs 目录下，同时通过 -e 参数设置 HTTPS 相关的环境变量参数，最后让仓库在主机上监听 443 端口。

仓库启动后，我们就可以远程推送镜像了。

```sh
$ docker tag busybox regisry.lagoudocker.io/busybox
$ docker push regisry.lagoudocker.io/busybox
```



#### 私有仓库进阶

Docker 官方开源的镜像仓库`Distribution`仅满足了镜像存储和管理的功能，用户权限管理相对较弱，并且没有管理界面。

如果你想要构建一个企业的镜像仓库，[Harbor](https://goharbor.io/) 是一个非常不错的解决方案。Harbor 是一个基于`Distribution`项目开发的一款企业级镜像管理软件，拥有 RBAC （基于角色的访问控制）、管理用户界面以及审计等非常完善的功能。目前已经从 CNCF 毕业，这代表它已经有了非常高的软件成熟度。

![Drawing 4.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl9bMHCAFgcMAABNmNOujV4312.png)



Harbor 的使命是成为 Kubernetes 信任的云原生镜像仓库。 Harbor 需要结合 Kubernetes 才能发挥其最大价值，因此，在这里我就不展开介绍 Harbor 了。如果你对 Harbor 构建企业级镜像仓库感兴趣，可以到它的[官网](https://goharbor.io/)了解更多。



### 结语

当你使用 Docker Hub 拉取镜像很慢的时候，你知道如何加快镜像的拉取速度吗？

加速通过国内三方镜像仓库通过私有镜像仓库缓存代理; `"registry-mirrors": [ "https://registry.docker-cn.com" ]`







## 06 | 最佳实践：如何在生产中编写最优 Dockerfile？



在介绍 Dockerfile 最佳实践前，这里再强调一下，**生产实践中一定优先使用 Dockerfile 的方式构建镜像。** 因为使用 Dockerfile 构建镜像可以带来很多好处：

- 易于版本化管理，Dockerfile 本身是一个文本文件，方便存放在代码仓库做版本管理，可以很方便地找到各个版本之间的变更历史；
- 过程可追溯，Dockerfile 的每一行指令代表一个镜像层，根据 Dockerfile 的内容即可很明确地查看镜像的完整构建过程；
- 屏蔽构建环境异构，使用 Dockerfile 构建镜像无须考虑构建环境，基于相同 Dockerfile 无论在哪里运行，构建结果都一致。



虽然有这么多好处，但是如果你 Dockerfile 使用不当也会引发很多问题。比如镜像构建时间过长，甚至镜像构建失败；镜像层数过多，导致镜像文件过大。所以，这一课时我就教你如何在生产环境中编写最优的 Dockerfile。

在介绍 Dockerfile 最佳实践前，我们再聊一下我们平时书写 Dockerfile 应该尽量遵循的原则。



### Dockerfile 书写原则

遵循以下 Dockerfile 书写原则，不仅可以使得我们的 Dockerfile 简洁明了，让协作者清楚地了解镜像的完整构建流程，还可以帮助我们减少镜像的体积，加快镜像构建的速度和分发速度。



#### （1）单一职责

由于容器的本质是进程，一个容器代表一个进程，因此不同功能的应用应该尽量拆分为不同的容器，每个容器只负责单一业务进程。



#### （2）提供注释信息

Dockerfile 也是一种代码，我们应该保持良好的代码编写习惯，晦涩难懂的代码尽量添加注释，让协作者可以一目了然地知道每一行代码的作用，并且方便扩展和使用。



#### （3）保持容器最小化

应该避免安装无用的软件包，比如在一个 nginx 镜像中，我并不需要安装 vim 、gcc 等开发编译工具。这样不仅可以加快容器构建速度，而且可以避免镜像体积过大。



#### （4）合理选择基础镜像

容器的核心是应用，因此只要基础镜像能够满足应用的运行环境即可。例如一个`Java`类型的应用运行时只需要`JRE`，并不需要`JDK`，因此我们的基础镜像只需要安装`JRE`环境即可。



#### （5）使用 .dockerignore 文件

在使用`git`时，我们可以使用`.gitignore`文件忽略一些不需要做版本管理的文件。同理，使用`.dockerignore`文件允许我们在构建时，忽略一些不需要参与构建的文件，从而提升构建效率。`.dockerignore`的定义类似于`.gitignore`。

`.dockerignore`的本质是文本文件，Docker 构建时可以使用换行符来解析文件定义，每一行可以忽略一些文件或者文件夹。具体使用方式如下：

| 规则       | 含义                                                         |
| ---------- | ------------------------------------------------------------ |
| #          | # 开头的表示注释，# 后面所有内容将会被忽略                   |
| */tmp*     | 匹配当前目录下任何以 tmp 开头的文件或者文件夹                |
| *.md       | 匹配以 .md 为后缀的任意文件                                  |
| tem?       | 匹配以 tem 开头并且以任意字符结尾的文件，？代表任意一个字符  |
| !README.md | ! 表示排除忽略。 例如 .dockerignore 定义如下：  *.md !README.md  表示除了 README.md 文件外所有以 .md 结尾的文件。 |



#### （6）尽量使用构建缓存

Docker 构建过程中，每一条 Dockerfile 指令都会提交为一个镜像层，下一条指令都是基于上一条指令构建的。如果构建时发现要构建的镜像层的父镜像层已经存在，并且下一条命令使用了相同的指令，即可命中构建缓存。

Docker 构建时判断是否需要使用缓存的规则如下：

- 从当前构建层开始，比较所有的子镜像，检查所有的构建指令是否与当前完全一致，如果不一致，则不使用缓存；
- 一般情况下，只需要比较构建指令即可判断是否需要使用缓存，但是有些指令除外（例如`ADD`和`COPY`）；
- 对于`ADD`和`COPY`指令不仅要校验命令是否一致，还要为即将拷贝到容器的文件计算校验和（根据文件内容计算出的一个数值，如果两个文件计算的数值一致，表示两个文件内容一致 ），命令和校验和完全一致，才认为命中缓存。



因此，基于 Docker 构建时的缓存特性，我们可以把不轻易改变的指令放到 Dockerfile 前面（例如安装软件包），而可能经常发生改变的指令放在 Dockerfile 末尾（例如编译应用程序）。

例如，我们想要定义一些环境变量并且安装一些软件包，可以按照如下顺序编写 Dockerfile：

```sh
FROM centos:7
# 设置环境变量指令放前面
ENV PATH /usr/local/bin:$PATH
# 安装软件指令放前面
RUN yum install -y make
# 把业务软件的配置,版本等经常变动的步骤放最后
...
```



按照上面原则编写的 Dockerfile 在构建镜像时，前面步骤命中缓存的概率会增加，可以大大缩短镜像构建时间。



#### （7）正确设置时区

我们从 Docker Hub 拉取的官方操作系统镜像大多数都是 UTC 时间（世界标准时间）。如果你想要在容器中使用中国区标准时间（东八区），请根据使用的操作系统修改相应的时区信息，下面我介绍几种常用操作系统的修改方式：

- **Ubuntu 和Debian 系统**

Ubuntu 和Debian 系统可以向 Dockerfile 中添加以下指令：

```sh
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
RUN echo "Asia/Shanghai" >> /etc/timezone
```



- **CentOS系统**

CentOS 系统则向 Dockerfile 中添加以下指令：

```sh
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```



#### （8）使用国内软件源加快镜像构建速度

由于我们常用的官方操作系统镜像基本都是国外的，软件服务器大部分也在国外，所以我们构建镜像的时候想要安装一些软件包可能会非常慢。

这里我以 CentOS 7 为例，介绍一下如何使用 163 软件源（国内有很多大厂，例如阿里、腾讯、网易等公司都免费提供的软件加速源）加快镜像构建。

首先在容器构建目录创建文件 CentOS7-Base-163.repo，文件内容如下：

```sh
# CentOS-Base.repo
#
# The mirror system uses the connecting IP address of the client and the
# update status of each mirror to pick mirrors that are updated to and
# geographically close to the client.  You should use this for CentOS updates
# unless you are manually picking other mirrors.
#
# If the mirrorlist= does not work for you, as a fall back you can try the 
# remarked out baseurl= line instead.
#
#
[base]
name=CentOS-$releasever - Base - 163.com
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=os
baseurl=http://mirrors.163.com/centos/$releasever/os/$basearch/
gpgcheck=1
gpgkey=http://mirrors.163.com/centos/RPM-GPG-KEY-CentOS-7
#released updates
[updates]
name=CentOS-$releasever - Updates - 163.com
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=updates
baseurl=http://mirrors.163.com/centos/$releasever/updates/$basearch/
gpgcheck=1
gpgkey=http://mirrors.163.com/centos/RPM-GPG-KEY-CentOS-7
#additional packages that may be useful
[extras]
name=CentOS-$releasever - Extras - 163.com
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=extras
baseurl=http://mirrors.163.com/centos/$releasever/extras/$basearch/
gpgcheck=1
gpgkey=http://mirrors.163.com/centos/RPM-GPG-KEY-CentOS-7
#additional packages that extend functionality of existing packages
[centosplus]
name=CentOS-$releasever - Plus - 163.com
baseurl=http://mirrors.163.com/centos/$releasever/centosplus/$basearch/
gpgcheck=1
enabled=0
gpgkey=http://mirrors.163.com/centos/RPM-GPG-KEY-CentOS-7
```



然后在 Dockerfile 中添加如下指令：

```sh
COPY CentOS7-Base-163.repo /etc/yum.repos.d/CentOS7-Base.repo
```



执行完上述步骤后，再使用`yum install`命令安装软件时就会默认从 163 获取软件包，这样可以大大提升构建速度。



#### （9）最小化镜像层数

在构建镜像时尽可能地减少 Dockerfile 指令行数。例如我们要在 CentOS 系统中安装`make`和`net-tools`两个软件包，应该在 Dockerfile 中使用以下指令：

```sh
RUN yum install -y make net-tools
```



而不应该写成这样：

```sh
RUN yum install -y make
RUN yum install -y net-tools
```



了解完 Dockerfile 的书写原则后，我们再来具体了解下这些原则落实到具体的 Dockerfile 指令应该如何书写。



### Dockerfile 指令书写建议

下面是我们常用的一些指令，这些指令对于刚接触 Docker 的人来说会非常容易出错，下面我对这些指令的书写建议详细讲解一下。



#### （1）RUN

`RUN`指令在构建时将会生成一个新的镜像层并且执行`RUN`指令后面的内容。

使用`RUN`指令时应该尽量遵循以下原则：

- 当`RUN`指令后面跟的内容比较复杂时，建议使用反斜杠（\） 结尾并且换行；
- `RUN`指令后面的内容尽量按照字母顺序排序，提高可读性。

例如，我想在官方的 CentOS 镜像下安装一些软件，一个建议的 Dockerfile 指令如下：

```sh
FROM centos:7
RUN yum install -y automake \
                   curl \
                   python \
                   vim
```



#### （2）CMD 和 ENTRYPOINT

`CMD`和`ENTRYPOINT`指令都是容器运行的命令入口，这两个指令使用中有很多相似的地方，但是也有一些区别。

这两个指令的相同之处，`CMD`和`ENTRYPOINT`的基本使用格式分为两种。

- 第一种为`CMD`/`ENTRYPOINT`["command" , "param"]。这种格式是使用 Linux 的`exec`实现的， 一般称为`exec`模式，这种书写格式为`CMD`/`ENTRYPOINT`后面跟 json 数组，也是Docker 推荐的使用格式。
- 另外一种格式为`CMD`/`ENTRYPOINT`command param ，这种格式是基于 shell 实现的， 通常称为`shell`模式。当使用`shell`模式时，Docker 会以 /bin/sh -c command 的方式执行命令。



> 使用 exec 模式启动容器时，容器的 1 号进程就是 CMD/ENTRYPOINT 中指定的命令，而使用 shell 模式启动容器时相当于我们把启动命令放在了 shell 进程中执行，等效于执行 /bin/sh -c "task command" 命令。因此 shell 模式启动的进程在容器中实际上并不是 1 号进程。



这两个指令的区别：

- Dockerfile 中如果使用了`ENTRYPOINT`指令，启动 Docker 容器时需要使用 --entrypoint 参数才能覆盖 Dockerfile 中的`ENTRYPOINT`指令 ，而使用`CMD`设置的命令则可以被`docker run`后面的参数直接覆盖。
- `ENTRYPOINT`指令可以结合`CMD`指令使用，也可以单独使用，而`CMD`指令只能单独使用。



看到这里你也许会问，我什么时候应该使用`ENTRYPOINT`,什么时候使用`CMD`呢？

如果你希望你的镜像足够灵活，推荐使用`CMD`指令。如果你的镜像只执行单一的具体程序，并且不希望用户在执行`docker run`时覆盖默认程序，建议使用`ENTRYPOINT`。

最后再强调一下，无论使用`CMD`还是`ENTRYPOINT`，都尽量使用`exec`模式。



#### （3）ADD 和 COPY

`ADD`和`COPY`指令功能类似，都是从外部往容器内添加文件。但是`COPY`指令只支持基本的文件和文件夹拷贝功能，`ADD`则支持更多文件来源类型，比如自动提取 tar 包，并且可以支持源文件为 URL 格式。

那么在日常应用中，我们应该使用哪个命令向容器里添加文件呢？你可能在想，既然`ADD`指令支持的功能更多，当然应该使用`ADD`指令了。然而事实恰恰相反，我更推荐你使用`COPY`指令，因为`COPY`指令更加透明，仅支持本地文件向容器拷贝，而且使用`COPY`指令可以更好地利用构建缓存，有效减小镜像体积。

当你想要使用`ADD`向容器中添加 URL 文件时，请尽量考虑使用其他方式替代。例如你想要在容器中安装 memtester（一种内存压测工具），你应该避免使用以下格式：

```sh
ADD http://pyropus.ca/software/memtester/old-versions/memtester-4.3.0.tar.gz /tmp/
RUN tar -xvf /tmp/memtester-4.3.0.tar.gz -C /tmp
RUN make -C /tmp/memtester-4.3.0 && make -C /tmp/memtester-4.3.0 install
```

下面是推荐写法：

```sh
RUN wget -O /tmp/memtester-4.3.0.tar.gz http://pyropus.ca/software/memtester/old-versions/memtester-4.3.0.tar.gz \
&& tar -xvf /tmp/memtester-4.3.0.tar.gz -C /tmp \
&& make -C /tmp/memtester-4.3.0 && make -C /tmp/memtester-4.3.0 install
```



#### （4）WORKDIR

为了使构建过程更加清晰明了，推荐使用 WORKDIR 来指定容器的工作路径，应该尽量避免使用 `RUN cd /work/path && do some work` 这样的指令。

最后给出几个常用软件的官方 Dockerfile 示例链接，希望可以对你有所帮助。

- [Go](https://github.com/docker-library/golang/blob/4d68c4dd8b51f83ce4fdce0f62484fdc1315bfa8/1.15/buster/Dockerfile)
- [Nginx](https://github.com/nginxinc/docker-nginx/blob/9774b522d4661effea57a1fbf64c883e699ac3ec/mainline/buster/Dockerfile)
- [Hy](https://github.com/hylang/docker-hylang/blob/f9c873b7f71f466e5af5ea666ed0f8f42835c688/dockerfiles-generated/Dockerfile.python3.8-buster)



### 结语

好了，到此为止，相信你已经对 Dockerfile 的书写原则和一些重要指令有了较深的认识。

当你需要编写编译型语言（例如 Golang、Java）的 Dockerfile 时，如何分离编译环境和运行环境，使得镜像体积尽可能小呢？

把编译环境单独打包镜像，只提供编译好的二进制（注意运行 CPU 架构）。运行环境单独做镜像，只考虑基础运行环境的配置。好处还是很明显，程序发布单独版本管理，运行环境也单独组合。还有好处就是，运行环境打补丁升级等等，不用影响程序发布。

分离编译环境和运行环境对于生产环境很重要.





## 07 | Docker 安全：基于内核的弱隔离系统如何保障安全性？



Docker 是基于 Linux 内核的 Namespace 技术实现资源隔离的，所有的容器都共享主机的内核。其实这与以虚拟机为代表的云计算时代还是有很多区别的，比如虚拟机有着更好的隔离性和安全性，而容器的隔离性和安全性则相对较弱。

在讨论容器的安全性之前，我们先了解下容器与虚拟机的区别，这样可以帮助我们更好地了解容器的安全隐患以及如何加固容器安全。



### Docker 与虚拟机区别

![image-20201107223748463](D:\学习资料\笔记\linux 性能\linux 性能笔记图片\image-20201107223748463.png)



从图 1 可以看出，虚拟机是通过管理系统(Hypervisor)模拟出 CPU、内存、网络等硬件，然后在这些模拟的硬件上创建客户内核和操作系统。这样做的好处就是虚拟机有自己的内核和操作系统，并且硬件都是通过虚拟机管理系统模拟出来的，用户程序无法直接使用到主机的操作系统和硬件资源，因此虚拟机也对隔离性和安全性有着更好的保证。

而 Docker 容器则是通过 Linux 内核的 Namespace 技术实现了文件系统、进程、设备以及网络的隔离，然后再通过 Cgroups 对 CPU、 内存等资源进行限制，最终实现了容器之间相互不受影响，由于容器的隔离性仅仅依靠内核来提供，因此容器的隔离性也远弱于虚拟机。

你可能会问，既然虚拟机安全性这么好，为什么我们还要用容器呢？这是因为容器与虚拟机相比，容器的性能损耗非常小，并且镜像也非常小，而且在业务快速开发和迭代的今天，容器秒级的启动等特性也非常匹配业务快速迭代的业务场景。

既然我们要利用容器的优点，那有没有什么办法可以尽量弥补容器弱隔离的安全性缺点呢？要了解如何解决容器的安全问题，我们首先需要了解下容器目前存在的安全问题。



### Docker 容器的安全问题

#### (1) Docker 自身安全

Docker 作为一款容器引擎，本身也会存在一些安全漏洞，CVE 目前已经记录了多项与 Docker 相关的安全漏洞，主要有权限提升、信息泄露等几类安全问题。具体 Docker 官方记录的安全问题可以参考[这里](https://docs.docker.com/engine/security/non-events/)。

> CVE 的维基百科定义：CVE 是公共漏洞和暴露（英语：CVE, Common Vulnerabilities and Exposures）又称常见漏洞与披露，是一个与信息安全有关的数据库，收集各种信息安全弱点及漏洞并给予编号以便于公众查阅。此数据库现由美国非营利组织 MITRE 所属的 National Cybersecurity FFRDC 所营运维护 。



#### (2) 镜像安全

由于 Docker 容器是基于镜像创建并启动，因此镜像的安全直接影响到容器的安全。具体影响镜像安全的总结如下。

- 镜像软件存在安全漏洞：由于容器需要安装基础的软件包，如果软件包存在漏洞，则可能会被不法分子利用并且侵入容器，影响其他容器或主机安全。

- 仓库漏洞：无论是 Docker 官方的镜像仓库还是我们私有的镜像仓库，都有可能被攻击，然后篡改镜像，当我们使用镜像时，就可能成为攻击者的目标对象。

- 用户程序漏洞：用户自己构建的软件包可能存在漏洞或者被植入恶意脚本，这样会导致运行时提权影响其他容器或主机安全。



#### (3) Linux 内核隔离性不够

尽管目前 Namespace 已经提供了非常多的资源隔离类型，但是仍有部分关键内容没有被完全隔离，其中包括一些系统的关键性目录（如 /sys、/proc 等），这些关键性的目录可能会泄露主机上一些关键性的信息，让攻击者利用这些信息对整个主机甚至云计算中心发起攻击。

而且仅仅依靠 Namespace 的隔离是远远不够的，因为一旦内核的 Namespace 被突破，使用者就有可能直接提权获取到主机的超级权限，从而影响主机安全。



#### (4) 所有容器共享主机内核

由于同一宿主机上所有容器共享主机内核，所以攻击者可以利用一些特殊手段导致内核崩溃，进而导致主机宕机影响主机上其他服务。

既然容器有这么多安全上的问题，那么我们应该如何做才能够既享受到容器的便利性同时也可以保障容器安全呢？下面我带你来逐步了解下如何解决容器的安全问题。



### 如何解决容器的安全问题？

#### (1) Docker 自身安全性改进

事实上，Docker 从 2013 年诞生到现在，在安全性上面已经做了非常多的努力。目前 Docker 在默认配置和默认行为下是足够安全的。

Docker 自身是基于 Linux 的多种 Namespace 实现的，其中有一个很重要的 Namespace 叫作 User Namespace，User Namespace 主要是用来做容器内用户和主机的用户隔离的。在过去容器里的 root 用户就是主机上的 root 用户，如果容器受到攻击，或者容器本身含有恶意程序，在容器内就可以直接获取到主机 root 权限。Docker 从 1.10 版本开始，使用 User Namespace 做用户隔离，实现了容器中的 root 用户映射到主机上的非 root 用户，从而大大减轻了容器被突破的风险。

因此，我们尽可能地使用 Docker 最新版本就可以得到更好的安全保障。



#### (2) 保障镜像安全

为保障镜像安全，我们可以在私有镜像仓库安装镜像安全扫描组件，对上传的镜像进行检查，通过与 CVE 数据库对比，一旦发现有漏洞的镜像及时通知用户或阻止非安全镜像继续构建和分发。同时为了确保我们使用的镜像足够安全，在拉取镜像时，要确保只从受信任的镜像仓库拉取，并且与镜像仓库通信一定要使用 HTTPS 协议。



#### (3) 加强内核安全和管理

由于仅仅依赖内核的隔离可能会引发安全问题，因此我们对于内核的安全应该更加重视。可以从以下几个方面进行加强。



##### 宿主机及时升级内核漏洞

宿主机内核应该尽量安装最新补丁，因为更新的内核补丁往往有着更好的安全性和稳定性。



##### 使用 Capabilities 划分权限

Capabilities 是 Linux 内核的概念，Linux 将系统权限分为了多个 Capabilities，它们都可以单独地开启或关闭，Capabilities 实现了系统更细粒度的访问控制。

容器和虚拟机在权限控制上还是有一些区别的，在虚拟机内我们可以赋予用户所有的权限，例如设置 cron 定时任务、操作内核模块、配置网络等权限。而容器则需要针对每一项 Capabilities 更细粒度的去控制权限，例如：

- cron 定时任务可以在容器内运行，设置定时任务的权限也仅限于容器内部；

- 由于容器是共享主机内核的，因此在容器内部一般不允许直接操作主机内核；

- 容器的网络管理在容器外部，这就意味着一般情况下，我们在容器内部是不需要执行ifconfig、route等命令的 。

由于容器可以按照需求逐项添加 Capabilities 权限，因此在大多数情况下，容器并不需要主机的 root 权限，Docker 默认情况下也是不开启额外特权的。

最后，在执行docker run命令启动容器时，如非特殊可控情况，--privileged 参数不允许设置为 true，其他特殊权限可以使用 --cap-add 参数，根据使用场景适当添加相应的权限。



##### 使用安全加固组件

Linux 的 SELinux、AppArmor、GRSecurity 组件都是 Docker 官方推荐的安全加固组件。下面我对这三个组件做简单介绍。

- SELinux (Secure Enhanced Linux): 是 Linux 的一个内核安全模块，提供了安全访问的策略机制，通过设置 SELinux 策略可以实现某些进程允许访问某些文件。

- AppArmor: 类似于 SELinux，也是一个 Linux 的内核安全模块，普通的访问控制仅能控制到用户的访问权限，而 AppArmor 可以控制到用户程序的访问权限。

- GRSecurity: 是一个对内核的安全扩展，可通过智能访问控制，提供内存破坏防御，文件系统增强等多种防御形式。

这三个组件可以限制一个容器对主机的内核或其他资源的访问控制。目前，容器报告的一些安全漏洞中，很多都是通过对内核进行加强访问和隔离来实现的。



##### 资源限制

在生产环境中，建议每个容器都添加相应的资源限制。下面给出一些执行docker run命令启动容器时可以传递的资源限制参数：

```dockerfile
  --cpus                          限制 CPU 配额
  -m, --memory                    限制内存配额
  --pids-limit                    限制容器的 PID 个数
```

例如我想要启动一个 1 核 2G 的容器，并且限制在容器内最多只能创建 1000 个 PID，启动命令如下：

```sh
$ docker run -it --cpus=1 -m=2048m --pids-limit=1000 busybox sh
```

推荐在生产环境中限制 CPU、内存、PID 等资源，这样即便应用程序有漏洞，也不会导致主机的资源完全耗尽，最大限度降低安全风险。



#### (4) 使用安全容器

容器有着轻便快速启动的优点，虚拟机有着安全隔离的优点，有没有一种技术可以兼顾两者的优点，做到既轻量又安全呢？

答案是有，那就是安全容器。安全容器是相较于普通容器的，安全容器与普通容器的主要区别在于，安全容器中的每个容器都运行在一个单独的微型虚拟机中，拥有独立的操作系统和内核，并且有虚拟化层的安全隔离。

安全容器目前推荐的技术方案是 Kata Containers，Kata Container 并不包含一个完整的操作系统，只有一个精简版的 Guest Kernel 运行着容器本身的应用，并且通过减少不必要的内存，尽量共享可以共享的内存来进一步减少内存的开销。另外，Kata Container 实现了 OCI 规范，可以直接使用 Docker 的镜像启动 Kata 容器，具有开销更小、秒级启动、安全隔离等许多优点。



### 结语

容器技术带来的技术革新是空前的，但是随之而来的容器安全问题也是我们必须要足够重视的。本课时解决 Docker 安全问题的精华我帮你总结如下：

![image-20201107230355552](D:\学习资料\笔记\linux 性能\linux 性能笔记图片\image-20201107230355552.png)



到此，相信你已经了解了 Docker 与虚拟机的本质区别，也知道了容器目前存在的一些安全隐患以及如何在生产环境中尽量避免这些安全隐患。

目前除了 Kata Container 外，你还知道其他的安全容器解决方案吗？

谷歌开源的gVisor是另一种安全容器技术，通过实现Linux的API，拦截容器对底层Linux接口的调用，实现容器安全的目标





# 底层实现原理及关键技术



## 08 | 容器监控：容器监控原理及 cAdvisor 的安装与使用



生产环境中监控容器的运行状况十分重要，通过监控我们可以随时掌握容器的运行状态，做到线上隐患和问题早发现，早解决。所以今天我就和你分享关于容器监控的知识（原理及工具 cAdvisor）。

虽然传统的物理机和虚拟机监控已经有了比较成熟的监控方案，但是容器的监控面临着更大的挑战，因为容器的行为和本质与传统的虚拟机是不一样的，总的来说，容器具有以下特性：

- 容器是短期存活的，并且可以动态调度；

- 容器的本质是进程，而不是一个完整操作系统；

- 由于容器非常轻量，容器的创建和销毁也会比传统虚拟机更加频繁。

Docker 容器的监控方案有很多，除了 Docker 自带的`docker stats`命令，还有很多开源的解决方案，例如 `sysdig`、`cAdvisor`、`Prometheus` 等，都是非常优秀的监控工具。

下面我们首先来看下，不借助任何外部工具，如何用 Docker 自带的`docker stats`命令实现容器的监控。



### 使用 docker stats 命令

使用 Docker 自带的docker stats命令可以很方便地看到主机上所有容器的 CPU、内存、网络 IO、磁盘 IO、PID 等资源的使用情况。下面我们可以具体操作看看。

首先在主机上使用以下命令启动一个资源限制为 1 核 2G 的 nginx 容器：

```sh
$ docker run --cpus=1 -m=2g --name=nginx  -d nginx
```

容器启动后，可以使用docker stats命令查看容器的资源使用状态:

```sh
$ docker stats nginx
```

通过docker stats命令可以看到容器的运行状态如下：

```sh
CONTAINER           CPU %               MEM USAGE / LIMIT   MEM %               NET I/O             BLOCK I/O           PIDS
f742a467b6d8        0.00%               1.387 MiB / 2 GiB   0.07%               656 B / 656 B       0 B / 9.22 kB       2
```

从容器的运行状态可以看出，docker stats命令确实可以获取并显示 Docker 容器运行状态。但是它的缺点也很明显，因为它只能获取本机数据，无法查看历史监控数据，没有可视化展示面板。

因此，生产环境中我们通常使用另一种容器监控解决方案 cAdvisor。



### cAdvisor

cAdvisor 是谷歌开源的一款通用的容器监控解决方案。cAdvisor 不仅可以采集机器上所有运行的容器信息，还提供了基础的查询界面和 HTTP 接口，更方便与外部系统结合。所以，cAdvisor很快成了容器指标监控最常用组件，并且 Kubernetes 也集成了 cAdvisor 作为容器监控指标的默认工具。



#### cAdvisor 的安装与使用

下面我们以 cAdvisor 0.37.0 版本为例，演示一下 cAdvisor 的安装与使用。

cAdvisor 官方提供了 Docker 镜像，我们只需要拉取镜像并且启动镜像即可。

> 由于 cAdvisor 镜像存放在谷歌的 gcr.io 镜像仓库中，国内无法访问到。这里我把打好的镜像放在了 Docker Hub。你可以使用 docker pull lagoudocker/cadvisor:v0.37.0 命令从 Docker Hub 拉取。

首先使用以下命令启动 cAdvisor：

```sh
$ docker run \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:ro \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --volume=/dev/disk/:/dev/disk:ro \
  --publish=8080:8080 \
  --detach=true \
  --name=cadvisor \
  --privileged \
  --device=/dev/kmsg \
  lagoudocker/cadvisor:v0.37.0
```

此时，cAdvisor 已经成功启动，我们可以通过访问 http://localhost:8080 访问到 cAdvisor 的 Web 界面。

![image-20201107231742077](D:\学习资料\笔记\linux 性能\linux 性能笔记图片\image-20201107231742077.png)



cAdvisor 不仅可以监控容器的资源使用情况，还可以监控主机的资源使用情况。下面我们就先看下它是如何查看主机资源使用情况的。



#### 使用 cAdvisor 查看主机监控

访问 http://localhost:8080/containers/ 地址，在首页可以看到主机的资源使用情况，包含 CPU、内存、文件系统、网络等资源，如下图所示。

![image-20201107231834771](D:\学习资料\笔记\linux 性能\linux 性能笔记图片\image-20201107231834771.png)



#### 使用 cAdvisor 查看容器监控

如果你想要查看主机上运行的容器资源使用情况，可以访问 http://localhost:8080/docker/，这个页面会列出 Docker 的基本信息和运行的容器情况，如下图所示。

![image-20201107231952759](D:\学习资料\笔记\linux 性能\linux 性能笔记图片\image-20201107231952759.png)



在上图中的 Subcontainers 下会列出当前主机上运行的所有容器，点击其中一个容器即可查看该容器的详细运行状态，如下图所示。

![image-20201107232121521](D:\学习资料\笔记\k8s\k8s图\image-20201107232121521.png)



总体来说，使用 cAdvisor 监控容器具有以下特点：

- 可以同时采集物理机和容器的状态；

- 可以展示监控历史数据。

了解 Docker 的监控工具，你是否想问，这些监控数据是怎么来的呢？下面我就带你了解一下容器监控的原理。



### 监控原理

我们知道 Docker 是基于 Namespace、Cgroups 和联合文件系统实现的。其中 Cgroups 不仅可以用于容器资源的限制，还可以提供容器的资源使用率。无论何种监控方案的实现，底层数据都来源于 Cgroups。

Cgroups 的工作目录为/sys/fs/cgroup，/sys/fs/cgroup目录下包含了 Cgroups 的所有内容。Cgroups包含很多子系统，可以用来对不同的资源进行限制。例如对CPU、内存、PID、磁盘 IO等资源进行限制和监控。

为了更详细的了解 Cgroups 的子系统，我们通过 ls -l 命令查看/sys/fs/cgroup文件夹，可以看到很多目录：

```sh
$ sudo ls -l /sys/fs/cgroup/
total 0
dr-xr-xr-x 5 root root  0 Jul  9 19:32 blkio
lrwxrwxrwx 1 root root 11 Jul  9 19:32 cpu -> cpu,cpuacct
dr-xr-xr-x 5 root root  0 Jul  9 19:32 cpu,cpuacct
lrwxrwxrwx 1 root root 11 Jul  9 19:32 cpuacct -> cpu,cpuacct
dr-xr-xr-x 3 root root  0 Jul  9 19:32 cpuset
dr-xr-xr-x 5 root root  0 Jul  9 19:32 devices
dr-xr-xr-x 3 root root  0 Jul  9 19:32 freezer
dr-xr-xr-x 3 root root  0 Jul  9 19:32 hugetlb
dr-xr-xr-x 5 root root  0 Jul  9 19:32 memory
lrwxrwxrwx 1 root root 16 Jul  9 19:32 net_cls -> net_cls,net_prio
dr-xr-xr-x 3 root root  0 Jul  9 19:32 net_cls,net_prio
lrwxrwxrwx 1 root root 16 Jul  9 19:32 net_prio -> net_cls,net_prio
dr-xr-xr-x 3 root root  0 Jul  9 19:32 perf_event
dr-xr-xr-x 5 root root  0 Jul  9 19:32 pids
dr-xr-xr-x 5 root root  0 Jul  9 19:32 systemd
```

这些目录代表了 Cgroups 的子系统，Docker 会在每一个 Cgroups 子系统下创建 docker 文件夹。这里如果你对 Cgroups 子系统不了解的话，不要着急，后续我会在第 10 课时对 Cgroups 子系统做详细讲解，这里你只需要明白容器监控数据来源于 Cgroups 即可。



#### 监控系统是如何获取容器的内存限制的？

下面我们以 memory 子系统（memory 子系统是Cgroups 众多子系统的一个，主要用来限制内存使用，Cgroups 会在第十课时详细讲解）为例，讲解一下监控组件是如何获取到容器的资源限制和使用状态的（即容器的内存限制）。

我们首先在主机上使用以下命令启动一个资源限制为 1 核 2G 的 nginx 容器：

```sh
$ docker run --name=nginx --cpus=1 -m=2g --name=nginx  -d nginx
## 这里输出的是容器 ID
51041a74070e9260e82876974762b8c61c5ed0a51832d74fba6711175f89ede1
```

> 注意：如果你已经创建过名称为 nginx 的容器，请先使用 `docker  rm -f nginx` 命令删除已经存在的 nginx 容器。

容器启动后，我们通过命令行的输出可以得到容器的 ID，同时 Docker 会在`/sys/fs/cgroup/memory/docker`目录下以容器 ID 为名称创建对应的文件夹。

下面我们查看一下`/sys/fs/cgroup/memory/docker`目录下的文件：

```sh
$ sudo ls -l /sys/fs/cgroup/memory/docker
total 0
drwxr-xr-x 2 root root 0 Sep  2 15:12 51041a74070e9260e82876974762b8c61c5ed0a51832d74fba6711175f89ede1
-rw-r--r-- 1 root root 0 Sep  2 14:57 cgroup.clone_children
--w--w--w- 1 root root 0 Sep  2 14:57 cgroup.event_control
-rw-r--r-- 1 root root 0 Sep  2 14:57 cgroup.procs
-rw-r--r-- 1 root root 0 Sep  2 14:57 memory.failcnt
--w------- 1 root root 0 Sep  2 14:57 memory.force_empty
-rw-r--r-- 1 root root 0 Sep  2 14:57 memory.kmem.failcnt
-rw-r--r-- 1 root root 0 Sep  2 14:57 memory.kmem.limit_in_bytes
-rw-r--r-- 1 root root 0 Sep  2 14:57 memory.kmem.max_usage_in_bytes
-r--r--r-- 1 root root 0 Sep  2 14:57 memory.kmem.slabinfo
-rw-r--r-- 1 root root 0 Sep  2 14:57 memory.kmem.tcp.failcnt
-rw-r--r-- 1 root root 0 Sep  2 14:57 memory.kmem.tcp.limit_in_bytes
-rw-r--r-- 1 root root 0 Sep  2 14:57 memory.kmem.tcp.max_usage_in_bytes
-r--r--r-- 1 root root 0 Sep  2 14:57 memory.kmem.tcp.usage_in_bytes
-r--r--r-- 1 root root 0 Sep  2 14:57 memory.kmem.usage_in_bytes
-rw-r--r-- 1 root root 0 Sep  2 14:57 memory.limit_in_bytes
-rw-r--r-- 1 root root 0 Sep  2 14:57 memory.max_usage_in_bytes
-rw-r--r-- 1 root root 0 Sep  2 14:57 memory.memsw.failcnt
-rw-r--r-- 1 root root 0 Sep  2 14:57 memory.memsw.limit_in_bytes
-rw-r--r-- 1 root root 0 Sep  2 14:57 memory.memsw.max_usage_in_bytes
-r--r--r-- 1 root root 0 Sep  2 14:57 memory.memsw.usage_in_bytes
-rw-r--r-- 1 root root 0 Sep  2 14:57 memory.move_charge_at_immigrate
-r--r--r-- 1 root root 0 Sep  2 14:57 memory.numa_stat
-rw-r--r-- 1 root root 0 Sep  2 14:57 memory.oom_control
---------- 1 root root 0 Sep  2 14:57 memory.pressure_level
-rw-r--r-- 1 root root 0 Sep  2 14:57 memory.soft_limit_in_bytes
-r--r--r-- 1 root root 0 Sep  2 14:57 memory.stat
-rw-r--r-- 1 root root 0 Sep  2 14:57 memory.swappiness
-r--r--r-- 1 root root 0 Sep  2 14:57 memory.usage_in_bytes
-rw-r--r-- 1 root root 0 Sep  2 14:57 memory.use_hierarchy
-rw-r--r-- 1 root root 0 Sep  2 14:57 notify_on_release
-rw-r--r-- 1 root root 0 Sep  2 14:57 tasks
```

可以看到 Docker 已经创建了以容器 ID 为名称的目录，我们再使用 ls 命令查看一下该目录的内容：

```sh
$ sudo ls -l /sys/fs/cgroup/memory/docker/51041a74070e9260e82876974762b8c61c5ed0a51832d74fba6711175f89ede1
total 0
-rw-r--r-- 1 root root 0 Sep  2 15:21 cgroup.clone_children
--w--w--w- 1 root root 0 Sep  2 15:13 cgroup.event_control
-rw-r--r-- 1 root root 0 Sep  2 15:12 cgroup.procs
-rw-r--r-- 1 root root 0 Sep  2 15:12 memory.failcnt
--w------- 1 root root 0 Sep  2 15:21 memory.force_empty
-rw-r--r-- 1 root root 0 Sep  2 15:21 memory.kmem.failcnt
-rw-r--r-- 1 root root 0 Sep  2 15:12 memory.kmem.limit_in_bytes
-rw-r--r-- 1 root root 0 Sep  2 15:21 memory.kmem.max_usage_in_bytes
-r--r--r-- 1 root root 0 Sep  2 15:21 memory.kmem.slabinfo
-rw-r--r-- 1 root root 0 Sep  2 15:21 memory.kmem.tcp.failcnt
-rw-r--r-- 1 root root 0 Sep  2 15:21 memory.kmem.tcp.limit_in_bytes
-rw-r--r-- 1 root root 0 Sep  2 15:21 memory.kmem.tcp.max_usage_in_bytes
-r--r--r-- 1 root root 0 Sep  2 15:21 memory.kmem.tcp.usage_in_bytes
-r--r--r-- 1 root root 0 Sep  2 15:21 memory.kmem.usage_in_bytes
-rw-r--r-- 1 root root 0 Sep  2 15:12 memory.limit_in_bytes
-rw-r--r-- 1 root root 0 Sep  2 15:12 memory.max_usage_in_bytes
-rw-r--r-- 1 root root 0 Sep  2 15:21 memory.memsw.failcnt
-rw-r--r-- 1 root root 0 Sep  2 15:12 memory.memsw.limit_in_bytes
-rw-r--r-- 1 root root 0 Sep  2 15:21 memory.memsw.max_usage_in_bytes
-r--r--r-- 1 root root 0 Sep  2 15:21 memory.memsw.usage_in_bytes
-rw-r--r-- 1 root root 0 Sep  2 15:21 memory.move_charge_at_immigrate
-r--r--r-- 1 root root 0 Sep  2 15:21 memory.numa_stat
-rw-r--r-- 1 root root 0 Sep  2 15:13 memory.oom_control
---------- 1 root root 0 Sep  2 15:21 memory.pressure_level
-rw-r--r-- 1 root root 0 Sep  2 15:21 memory.soft_limit_in_bytes
-r--r--r-- 1 root root 0 Sep  2 15:21 memory.stat
-rw-r--r-- 1 root root 0 Sep  2 15:21 memory.swappiness
-r--r--r-- 1 root root 0 Sep  2 15:12 memory.usage_in_bytes
-rw-r--r-- 1 root root 0 Sep  2 15:21 memory.use_hierarchy
-rw-r--r-- 1 root root 0 Sep  2 15:21 notify_on_release
-rw-r--r-- 1 root root 0 Sep  2 15:21 tasks
```

由上可以看到，容器 ID 的目录下有很多文件，其中 memory.limit_in_bytes 文件代表该容器内存限制大小，单位为 byte，我们使用 cat 命令（cat 命令可以查看文件内容）查看一下文件内容：

```sh
$ sudo cat /sys/fs/cgroup/memory/docker/51041a74070e9260e82876974762b8c61c5ed0a51832d74fba6711175f89ede1/memory.limit_in_bytes
2147483648
```

这里可以看到`memory.limit_in_bytes` 的值为2147483648，转换单位后正好为 2G，符合我们启动容器时的内存限制 2G。

通过 memory 子系统的例子，我们可以知道监控组件通过读取 `memory.limit_in_bytes` 文件即可获取到容器内存的限制值。了解完容器的内存限制我们来了解一下容器的内存使用情况。



#### 监控系统是如何获取容器的内存使用状态的？

内存使用情况存放在 memory.usage_in_bytes 文件里，同样我们也使用 cat 命令查看一下文件内容:

```sh
$ sudo /sys/fs/cgroup/memory/docker/51041a74070e9260e82876974762b8c61c5ed0a51832d74fba6711175f89ede1/memory.usage_in_bytes
4259840
```

可以看到当前内存的使用大小为 4259840 byte，约为 4 M。了解了内存的监控，下面我们来了解下网络的监控数据来源。

网络的监控数据来源是从 /proc/{PID}/net/dev 目录下读取的，其中 PID 为容器在主机上的进程 ID。下面我们首先使用 docker inspect 命令查看一下上面启动的 nginx 容器的 PID，命令如下：

```sh
$ docker inspect nginx |grep Pid
            "Pid": 27348,
            "PidMode": "",
            "PidsLimit": 0,
```

`/proc/27348/net/dev` 文件记录了该容器里每一个网卡的流量接收和发送情况，以及错误数、丢包数等信息。可见容器的网络监控数据都是定时从这里读取并展示的。

总结一下，**容器的监控原理其实就是定时读取 Linux 主机上相关的文件并展示给用户**。



### 结语

到此，相信你已经可以使用 docker stats 和 cAdvisor 监控并查看容器的状态了；也可以自己启动一个 cAdvisor 容器来监控主机和主机上的容器，并对监控系统的原理有了较深的了解。

试想下，cAdvisor 虽然可以临时存储一段历史监控数据，并且提供了一个简版的监控面板，在大规模的容器集群中，cAdvisor 有什么明显的不足吗？





## 09 | 资源隔离：为什么构建容器需要 Namespace ？



我们知道， Docker 是使用 Linux 的 Namespace 技术实现各种资源隔离的。那么究竟什么是 Namespace，各种 Namespace 都有什么作用，为什么 Docker 需要 Namespace呢？



### 什么是 Namespace？

下面是 Namespace 的维基百科定义：

> Namespace 是 Linux 内核的一项功能，该功能对内核资源进行分区，以使一组进程看到一组资源，而另一组进程看到另一组资源。Namespace 的工作方式通过为一组资源和进程设置相同的 Namespace 而起作用，但是这些 Namespace 引用了不同的资源。资源可能存在于多个 Namespace 中。这些资源可以是进程 ID、主机名、用户 ID、文件名、与网络访问相关的名称和进程间通信。

简单来说，Namespace 是 Linux 内核的一个特性，该特性可以实现在同一主机系统中，对进程 ID、主机名、用户 ID、文件名、网络和进程间通信等资源的隔离。Docker 利用 Linux 内核的 Namespace 特性，实现了每个容器的资源相互隔离，从而保证容器内部只能访问到自己 Namespace 的资源。

最新的 Linux 5.6 内核中提供了 8 种类型的 Namespace：

![image-20201111092805646](D:\学习资料\笔记\k8s\k8s图\image-20201111092805646.png)

虽然 Linux 内核提供了8种 Namespace，但是最新版本的 Docker 只使用了其中的前6 种，分别为Mount Namespace、PID Namespace、Net Namespace、IPC Namespace、UTS Namespace、User Namespace。

下面，我们详细了解下 Docker 使用的 6 种 Namespace的作用分别是什么。



### 各种 Namespace 的作用？

#### （1）Mount Namespace

Mount Namespace 是 Linux 内核实现的第一个 Namespace，从内核的 2.4.19 版本开始加入。它可以用来隔离不同的进程或进程组看到的挂载点。通俗地说，就是可以实现在不同的进程中看到不同的挂载目录。使用 Mount Namespace 可以实现容器内只能看到自己的挂载信息，在容器内的挂载操作不会影响主机的挂载目录。

下面我们通过一个实例来演示下 Mount Namespace。在演示之前，我们先来认识一个命令行工具 `unshare`。unshare 是 util-linux 工具包中的一个工具，CentOS 7 系统默认已经集成了该工具，使用 unshare 命令可以实现创建并访问不同类型的 Namespace。

首先我们使用以下命令创建一个 bash 进程并且新建一个 Mount Namespace：

```sh
$ sudo unshare --mount --fork /bin/bash
[root@centos7 centos]$
```

执行完上述命令后，这时我们已经在主机上创建了一个新的 Mount Namespace，并且当前命令行窗口加入了新创建的 Mount Namespace。下面我通过一个例子来验证下，在独立的 Mount Namespace 内创建挂载目录是不影响主机的挂载目录的。

首先在 /tmp 目录下创建一个目录。

```sh
[root@centos7 centos]$ mkdir /tmp/tmpfs
```

创建好目录后使用 mount 命令挂载一个 tmpfs 类型的目录。命令如下：

```sh
[root@centos7 centos]$ mount -t tmpfs -o size=20m tmpfs /tmp/tmpfs
```

然后使用 df 命令查看一下已经挂载的目录信息：

```sh
[root@centos7 centos]$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda1       500G  1.4G  499G   1% /
devtmpfs         16G     0   16G   0% /dev
tmpfs            16G     0   16G   0% /dev/shm
tmpfs            16G     0   16G   0% /sys/fs/cgroup
tmpfs            16G   57M   16G   1% /run
tmpfs           3.2G     0  3.2G   0% /run/user/1000
tmpfs            20M     0   20M   0% /tmp/tmpfs
```

可以看到 /tmp/tmpfs 目录已经被正确挂载。为了验证主机上并没有挂载此目录，我们新打开一个命令行窗口，同样执行 df 命令查看主机的挂载信息：

```sh
[centos@centos7 ~]$ df -h
Filesystem      Size  Used Avail Use% Mounted on
devtmpfs         16G     0   16G   0% /dev
tmpfs            16G     0   16G   0% /dev/shm
tmpfs            16G   57M   16G   1% /run
tmpfs            16G     0   16G   0% /sys/fs/cgroup
/dev/vda1       500G  1.4G  499G   1% /
tmpfs           3.2G     0  3.2G   0% /run/user/1000
```

通过上面输出可以看到主机上并没有挂载 /tmp/tmpfs，可见我们独立的 Mount Namespace 中执行 mount 操作并不会影响主机。

为了进一步验证我们的想法，我们继续在当前命令行窗口查看一下当前进程的 Namespace 信息，命令如下：

```sh
[root@centos7 centos]$ ls -l /proc/self/ns/
total 0
lrwxrwxrwx. 1 root root 0 Sep  4 08:20 ipc -> ipc:[4026531839]
lrwxrwxrwx. 1 root root 0 Sep  4 08:20 mnt -> mnt:[4026532239]
lrwxrwxrwx. 1 root root 0 Sep  4 08:20 net -> net:[4026531956]
lrwxrwxrwx. 1 root root 0 Sep  4 08:20 pid -> pid:[4026531836]
lrwxrwxrwx. 1 root root 0 Sep  4 08:20 user -> user:[4026531837]
lrwxrwxrwx. 1 root root 0 Sep  4 08:20 uts -> uts:[4026531838]
```

通过上面输出可以看到主机上并没有挂载 /tmp/tmpfs，可见我们独立的 Mount Namespace 中执行 mount 操作并不会影响主机。

为了进一步验证我们的想法，我们继续在当前命令行窗口查看一下当前进程的 Namespace 信息，命令如下：

```sh
[root@centos7 centos]$ ls -l /proc/self/ns/
total 0
lrwxrwxrwx. 1 root root 0 Sep  4 08:20 ipc -> ipc:[4026531839]
lrwxrwxrwx. 1 root root 0 Sep  4 08:20 mnt -> mnt:[4026532239]
lrwxrwxrwx. 1 root root 0 Sep  4 08:20 net -> net:[4026531956]
lrwxrwxrwx. 1 root root 0 Sep  4 08:20 pid -> pid:[4026531836]
lrwxrwxrwx. 1 root root 0 Sep  4 08:20 user -> user:[4026531837]
lrwxrwxrwx. 1 root root 0 Sep  4 08:20 uts -> uts:[4026531838]
```

然后新打开一个命令行窗口，使用相同的命令查看一下主机上的 Namespace 信息：

```sh
[centos@centos7 ~]$ ls -l /proc/self/ns/
total 0
lrwxrwxrwx. 1 centos centos 0 Sep  4 08:20 ipc -> ipc:[4026531839]
lrwxrwxrwx. 1 centos centos 0 Sep  4 08:20 mnt -> mnt:[4026531840]
lrwxrwxrwx. 1 centos centos 0 Sep  4 08:20 net -> net:[4026531956]
lrwxrwxrwx. 1 centos centos 0 Sep  4 08:20 pid -> pid:[4026531836]
lrwxrwxrwx. 1 centos centos 0 Sep  4 08:20 user -> user:[4026531837]
lrwxrwxrwx. 1 centos centos 0 Sep  4 08:20 uts -> uts:[4026531838]
```

通过对比两次命令的输出结果，我们可以看到，除了 Mount Namespace 的 ID 值不一样外，其他Namespace 的 ID 值均一致。

通过以上结果我们可以得出结论，**使用 unshare 命令可以新建 Mount Namespace，并且在新建的 Mount Namespace 内 mount 是和外部完全隔离的。**



#### （2）PID Namespace

PID Namespace 的作用是用来隔离进程。在不同的 PID Namespace 中，进程可以拥有相同的 PID 号，利用 PID Namespace 可以实现每个容器的主进程为 1 号进程，而容器内的进程在主机上却拥有不同的PID。例如一个进程在主机上 PID 为 122，使用 PID Namespace 可以实现该进程在容器内看到的 PID 为 1。

下面我们通过一个实例来演示下 PID Namespace的作用。首先我们使用以下命令创建一个 bash 进程，并且新建一个 PID Namespace：

```sh
$ sudo unshare --pid --fork --mount-proc /bin/bash
[root@centos7 centos]#
```

执行完上述命令后，我们在主机上创建了一个新的 PID Namespace，并且当前命令行窗口加入了新创建的 PID Namespace。在当前的命令行窗口使用 ps aux 命令查看一下进程信息：

```sh
[root@centos7 centos]# ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0 115544  2004 pts/0    S    10:57   0:00 bash
root        10  0.0  0.0 155444  1764 pts/0    R+   10:59   0:00 ps aux
```

通过上述命令输出结果可以看到当前 Namespace 下 bash 为 1 号进程，而且我们也看不到主机上的其他进程信息。



#### （3）UTS Namespace

UTS Namespace 主要是用来隔离主机名的，它允许每个 UTS Namespace 拥有一个独立的主机名。例如我们的主机名称为 docker，使用 UTS Namespace 可以实现在容器内的主机名称为 lagoudocker 或者其他任意自定义主机名。

同样我们通过一个实例来验证下 UTS Namespace 的作用，首先我们使用 unshare 命令来创建一个 UTS Namespace：

```sh
$ sudo unshare --uts --fork /bin/bash
[root@centos7 centos]#
```

创建好 UTS Namespace 后，当前命令行窗口已经处于一个独立的 UTS Namespace 中，下面我们使用 hostname 命令（hostname 可以用来查看主机名称）设置一下主机名：

```sh
root@centos7 centos]$ hostname -b lagoudocker
```

然后再查看一下主机名：

```sh
[root@centos7 centos]$ hostname
lagoudocker
```

通过上面命令的输出，我们可以看到当前UTS Namespace 内的主机名已经被修改为 lagoudocker。然后我们新打开一个命令行窗口，使用相同的命令查看一下主机的 hostname：

```sh
[centos@centos7 ~]$ hostname
centos7
```

可以看到主机的名称仍然为 centos7，并没有被修改。由此，可以验证 UTS Namespace 可以用来隔离主机名。



#### （4）IPC Namespace

IPC Namespace 主要是用来隔离进程间通信的。例如 PID Namespace 和 IPC Namespace 一起使用可以实现同一 IPC Namespace 内的进程彼此可以通信，不同 IPC Namespace 的进程却不能通信。

同样我们通过一个实例来验证下IPC Namespace的作用，首先我们使用 unshare 命令来创建一个 IPC Namespace：

```sh
$ sudo unshare --ipc --fork /bin/bash
[root@centos7 centos]#
```

下面我们需要借助两个命令来实现对 IPC Namespace 的验证。

ipcs -q 命令：用来查看系统间通信队列列表。

ipcmk -Q 命令：用来创建系统间通信队列。

我们首先使用 ipcs -q 命令查看一下当前 IPC Namespace 下的系统通信队列列表：

```sh
[centos@centos7 ~]$ ipcs -q
------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages
```

由上可以看到当前无任何系统通信队列，然后我们使用 ipcmk -Q 命令创建一个系统通信队列：

```sh
[root@centos7 centos]# ipcmk -Q
Message queue id: 0
```

再次使用 ipcs -q 命令查看当前 IPC Namespace 下的系统通信队列列表:

```sh
[root@centos7 centos]# ipcs -q
------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages
0x73682a32 0          root       644        0            0
```

可以看到我们已经成功创建了一个系统通信队列。然后我们新打开一个命令行窗口，使用ipcs -q 命令查看一下主机的系统通信队列：

```sh
[centos@centos7 ~]$ ipcs -q
------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages
```

通过上面的实验，可以发现，在单独的 IPC Namespace 内创建的系统通信队列在主机上无法看到。即 IPC Namespace 实现了系统通信队列的隔离。



#### （5）User Namespace

User Namespace 主要是用来隔离用户和用户组的。一个比较典型的应用场景就是在主机上以非 root 用户运行的进程可以在一个单独的 User Namespace 中映射成 root 用户。使用 User Namespace 可以实现进程在容器内拥有 root 权限，而在主机上却只是普通用户。

User Namesapce 的创建是可以不使用 root 权限的。下面我们以普通用户的身份创建一个 User Namespace，命令如下：

```sh
[centos@centos7 ~]$ unshare --user -r /bin/bash
[root@centos7 ~]#
```

> CentOS7 默认允许创建的 User Namespace 为 0，如果执行上述命令失败（ unshare 命令返回的错误为 unshare: unshare failed: Invalid argument ），需要使用以下命令修改系统允许创建的 User Namespace 数量，命令为：`echo 65535 > /proc/sys/user/max_user_namespaces`，然后再次尝试创建 User Namespace。

然后执行 id 命令查看一下当前的用户信息：

```sh
[root@centos7 ~]# id
uid=0(root) gid=0(root) groups=0(root),65534(nfsnobody) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
```

通过上面的输出可以看到我们在新的 User Namespace 内已经是 root 用户了。下面我们使用只有主机 root 用户才可以执行的 reboot 命令来验证一下，在当前命令行窗口执行 reboot 命令：

```sh
[root@centos7 ~]# reboot
Failed to open /dev/initctl: Permission denied
Failed to talk to init daemon.
```

可以看到，我们在新创建的 User Namespace 内虽然是 root 用户，但是并没有权限执行 reboot 命令。这说明在隔离的 User Namespace 中，并不能获取到主机的 root 权限，也就是说 User Namespace 实现了用户和用户组的隔离。



#### （6）Net Namespace

Net Namespace 是用来隔离网络设备、IP 地址和端口等信息的。Net Namespace 可以让每个进程拥有自己独立的 IP 地址，端口和网卡信息。例如主机 IP 地址为 172.16.4.1 ，容器内可以设置独立的 IP 地址为 192.168.1.1。

同样用实例验证，我们首先使用 ip a 命令查看一下主机上的网络信息：

```sh
$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 02:11:b0:14:01:0c brd ff:ff:ff:ff:ff:ff
    inet 172.20.1.11/24 brd 172.20.1.255 scope global dynamic eth0
       valid_lft 86063337sec preferred_lft 86063337sec
    inet6 fe80::11:b0ff:fe14:10c/64 scope link
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:82:8d:a0:df brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:82ff:fe8d:a0df/64 scope link
       valid_lft forever preferred_lft forever
```

然后我们使用以下命令创建一个 Net Namespace：

```sh
$ sudo unshare --net --fork /bin/bash
[root@centos7 centos]#
```

同样的我们使用 ip a 命令查看一下网络信息：

```sh
[root@centos7 centos]# ip a
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```

可以看到，宿主机上有 lo、eth0、docker0 等网络设备，而我们新建的 Net Namespace 内则与主机上的网络设备不同。



### 为什么 Docker 需要 Namespace？

Linux 内核从 2002 年 2.4.19 版本开始加入了 Mount Namespace，而直到内核 3.8 版本加入了 User Namespace 才为容器提供了足够的支持功能。

当 Docker 新建一个容器时， 它会创建这六种 Namespace，然后将容器中的进程加入这些 Namespace 之中，使得 Docker 容器中的进程只能看到当前 Namespace 中的系统资源。

正是由于 Docker 使用了 Linux 的这些 Namespace 技术，才实现了 Docker 容器的隔离，可以说没有 Namespace，就没有 Docker 容器。



### 小结

到此，相信你已经了解了什么是 Namespace。Namespace 是 Linux 内核的一个特性，该特性可以实现在同一主机系统中对进程 ID、主机名、用户 ID、文件名、网络和进程间通信等资源的隔离。Docker 正是结合了这六种 Namespace 的功能，才诞生了 Docker 容器。

最后，试想下，当我们使用 `docker run --net=host` 命令启动容器时，容器是否和主机共享同一个 Net Namespace？

> 这种方式启动容器，并不会创建Net Namespace，而是和宿主共享。所以，Docker不建议使用这种





## 10 | 资源限制：如何通过 Cgroups 机制实现资源限制？



上一课时，我们知道使用不同的 Namespace，可以实现容器中的进程看不到别的容器的资源，但是有一个问题你是否注意到？容器内的进程仍然可以任意地使用主机的 CPU 、内存等资源，如果某一个容器使用的主机资源过多，可能导致主机的资源竞争，进而影响业务。那如果我们想限制一个容器资源的使用（如 CPU、内存等）应该如何做呢？

这里就需要用到 Linux 内核的另一个核心技术cgroups。那么究竟什么是cgroups？我们应该如何使用cgroups？Docker 又是如何使用cgroups的？

### cgroups

cgroups（全称：control groups）是 Linux 内核的一个功能，它可以实现限制进程或者进程组的资源（如 CPU、内存、磁盘 IO 等）。

> 在 2006 年，Google 的工程师（ Rohit Seth 和 Paul Menage 为主要发起人） 发起了这个项目，起初项目名称并不是cgroups，而被称为进程容器（process containers）。在 2007 年cgroups代码计划合入Linux 内核，但是当时在 Linux 内核中，容器（container）这个词被广泛使用，并且拥有不同的含义。为了避免命名混乱和歧义，进程容器被重名为cgroups，并在 2008 年成功合入 Linux 2.6.24 版本中。cgroups目前已经成为 systemd、Docker、Linux Containers（LXC） 等技术的基础。



### cgroups 功能及核心概念

cgroups 主要提供了如下功能。

- 资源限制： 限制资源的使用量，例如我们可以通过限制某个业务的内存上限，从而保护主机其他业务的安全运行。

- 优先级控制：不同的组可以有不同的资源（ CPU 、磁盘 IO 等）使用优先级。

- 审计：计算控制组的资源使用情况。

- 控制：控制进程的挂起或恢复。



了解了 cgroups 可以为我们提供什么功能，下面我来看下 cgroups 是如何实现这些功能的。

cgroups功能的实现依赖于三个核心概念：子系统、控制组、层级树。

- 子系统（subsystem）：是一个内核的组件，一个子系统代表一类资源调度控制器。例如内存子系统可以限制内存的使用量，CPU 子系统可以限制 CPU 的使用时间。

- 控制组（cgroup）：表示一组进程和一组带有参数的子系统的关联关系。例如，一个进程使用了 CPU 子系统来限制 CPU 的使用时间，则这个进程和 CPU 子系统的关联关系称为控制组。

- 层级树（hierarchy）：是由一系列的控制组按照树状结构排列组成的。这种排列方式可以使得控制组拥有父子关系，子控制组默认拥有父控制组的属性，也就是子控制组会继承于父控制组。比如，系统中定义了一个控制组 c1，限制了 CPU 可以使用 1 核，然后另外一个控制组 c2 想实现既限制 CPU 使用 1 核，同时限制内存使用 2G，那么 c2 就可以直接继承 c1，无须重复定义 CPU 限制。

cgroups 的三个核心概念中，子系统是最核心的概念，因为子系统是真正实现某类资源的限制的基础。



### cgroups 子系统实例

下面我通过一个实例演示一下在 Linux 上默认都启动了哪些子系统。

我们先通过 mount 命令查看一下当前系统已经挂载的cgroups信息：

```sh
$ sudo mount -t cgroup
cgroup on /sys/fs/cgroup/systemd type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,xattr,release_agent=/usr/lib/systemd/systemd-cgroups-agent,name=systemd)
cgroup on /sys/fs/cgroup/net_cls,net_prio type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,net_prio,net_cls)
cgroup on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,blkio)
cgroup on /sys/fs/cgroup/pids type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,pids)
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,cpuacct,cpu)
cgroup on /sys/fs/cgroup/perf_event type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,perf_event)
cgroup on /sys/fs/cgroup/freezer type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,freezer)
cgroup on /sys/fs/cgroup/devices type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,devices)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,memory)
cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,cpuset)
cgroup on /sys/fs/cgroup/hugetlb type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,hugetlb)
```

> 我的操作系统版本为 CentOS7.8，内核为 3.10.0-1127.el7.x86_64 版本，不同内核版本cgroups子系统和使用方式可能略有差异。如果你对cgroups不是很熟悉，请尽量使用与我相同的内核环境操作。

通过输出，可以看到当前系统已经挂载了我们常用的cgroups子系统，例如 cpu、memory、pids 等我们常用的cgroups子系统。这些子系统中，cpu 和 memory 子系统是容器环境中使用最多的子系统，下面我对这两个子系统做详细介绍。



#### cpu 子系统

我首先以 cpu 子系统为例，演示一下cgroups如何限制进程的 cpu 使用时间。由于cgroups的操作很多需要用到 root 权限，我们在执行命令前要确保已经切换到了 root 用户，以下命令的执行默认都是使用 root 用户。

**第一步：在 cpu 子系统下创建 cgroup**

cgroups的创建很简单，只需要在相应的子系统下创建目录即可。下面我们到 cpu 子系统下创建测试文件夹：

```sh
$ mkdir /sys/fs/cgroup/cpu/mydocker
```

执行完上述命令后，我们查看一下我们新创建的目录下发生了什么？

```sh
$ ls -l /sys/fs/cgroup/cpu/mydocker
total 0
-rw-r--r--. 1 root root 0 Sep  5 09:19 cgroup.clone_children
--w--w--w-. 1 root root 0 Sep  5 09:19 cgroup.event_control
-rw-r--r--. 1 root root 0 Sep  5 09:19 cgroup.procs
-rw-r--r--. 1 root root 0 Sep  5 09:19 cpu.cfs_period_us
-rw-r--r--. 1 root root 0 Sep  5 09:19 cpu.cfs_quota_us
-rw-r--r--. 1 root root 0 Sep  5 09:19 cpu.rt_period_us
-rw-r--r--. 1 root root 0 Sep  5 09:19 cpu.rt_runtime_us
-rw-r--r--. 1 root root 0 Sep  5 09:19 cpu.shares
-r--r--r--. 1 root root 0 Sep  5 09:19 cpu.stat
-r--r--r--. 1 root root 0 Sep  5 09:19 cpuacct.stat
-rw-r--r--. 1 root root 0 Sep  5 09:19 cpuacct.usage
-r--r--r--. 1 root root 0 Sep  5 09:19 cpuacct.usage_percpu
-rw-r--r--. 1 root root 0 Sep  5 09:19 notify_on_release
-rw-r--r--. 1 root root 0 Sep  5 09:19 tasks
```

由上可以看到我们新建的目录下被自动创建了很多文件，其中 cpu.cfs_quota_us 文件代表在某一个阶段限制的 CPU 时间总量，单位为微秒。例如，我们想限制某个进程最多使用 1 核 CPU，就在这个文件里写入 100000（100000 代表限制 1 个核） ，tasks 文件中写入进程的 ID 即可（如果要限制多个进程 ID，在 tasks 文件中用换行符分隔即可）。

此时，我们所需要的 cgroup 就创建好了。对，就是这么简单。



**第二步：创建进程，加入 cgroup**

这里为了方便演示，我先把当前运行的 shell 进程加入 cgroup，然后在当前 shell 运行 cpu 耗时任务（这里利用到了继承，子进程会继承父进程的 cgroup）。

使用以下命令将 shell 进程加入 cgroup 中：

```sh
$ cd /sys/fs/cgroup/cpu/mydocker
$ echo $$ > tasks  
```

查看一下 tasks 文件内容：

```sh
$ cat tasks
3485
3543
```

其中第一个进程 ID 为当前 shell 的主进程，也就是说，当前 shell 主进程为 3485。



**第三步：执行 CPU 耗时任务，验证 cgroup 是否可以限制 cpu 使用时间**

下面，我们使用以下命令制造一个死循环，来提升 cpu 使用率：

```sh
$ while true;do echo;done;
```

执行完上述命令后，我们新打开一个 shell 窗口，使用 top -p 命令查看当前 cpu 使用率，-p 参数后面跟进程 ID，我这里是 3485。

```sh
$ top -p 3485
top - 09:51:35 up 3 days, 22:00,  4 users,  load average: 1.59, 0.58, 0.27
Tasks:   1 total,   0 running,   1 sleeping,   0 stopped,   0 zombie
%Cpu(s):  9.7 us,  2.8 sy,  0.0 ni, 87.4 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem : 32779616 total, 31009780 free,   495988 used,  1273848 buff/cache
KiB Swap:        0 total,        0 free,        0 used. 31852336 avail Mem
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
3485 root      20   0  116336   2852   1688 S  99.7  0.0   2:10.71 bash
```

通过上面输出可以看到 3485 这个进程被限制到了只能使用 100 % 的 cpu，也就是 1 个核。说明我们使用 cgroup 来限制 cpu 使用时间已经生效。此时，执行 while 循环的命令行窗口可以使用 Ctrl+c 退出循环。

为了进一步证实 cgroup 限制 cpu 的准确性，我们修改 cpu 限制时间为 0.5 核，命令如下：

```sh
$ cd /sys/fs/cgroup/cpu/mydocker
$ echo 50000 > cpu.cfs_quota_us
```

同样使用上面的命令来制造死循环：

```sh
$ while true;do echo;done;
```

保持当前窗口，新打开一个 shell 窗口，使用 top -p 参数查看 cpu 使用率：

```sh
$ top -p 3485
top - 10:05:25 up 3 days, 22:14,  3 users,  load average: 1.02, 0.43, 0.40
Tasks:   1 total,   1 running,   0 sleeping,   0 stopped,   0 zombie
%Cpu(s):  5.0 us,  1.3 sy,  0.0 ni, 93.7 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem : 32779616 total, 31055676 free,   450224 used,  1273716 buff/cache
KiB Swap:        0 total,        0 free,        0 used. 31898216 avail Mem
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 3485 root      20   0  115544   2116   1664 R  50.0  0.0   0:23.39 bash
```

通过上面输出可以看到，此时 cpu 使用率已经被限制到了 50%，即 0.5 个核。
验证完 cgroup 限制 cpu，我们使用相似的方法来验证 cgroup 对内存的限制。



#### memroy 子系统

**第一步：在 memory 子系统下创建 cgroup**

```sh
$ mkdir /sys/fs/cgroup/memory/mydocker
```

同样，我们查看一下新创建的目录下发生了什么？

```sh
total 0
-rw-r--r--. 1 root root 0 Sep  5 10:18 cgroup.clone_children
--w--w--w-. 1 root root 0 Sep  5 10:18 cgroup.event_control
-rw-r--r--. 1 root root 0 Sep  5 10:18 cgroup.procs
-rw-r--r--. 1 root root 0 Sep  5 10:18 memory.failcnt
--w-------. 1 root root 0 Sep  5 10:18 memory.force_empty
-rw-r--r--. 1 root root 0 Sep  5 10:18 memory.kmem.failcnt
-rw-r--r--. 1 root root 0 Sep  5 10:18 memory.kmem.limit_in_bytes
-rw-r--r--. 1 root root 0 Sep  5 10:18 memory.kmem.max_usage_in_bytes
-r--r--r--. 1 root root 0 Sep  5 10:18 memory.kmem.slabinfo
-rw-r--r--. 1 root root 0 Sep  5 10:18 memory.kmem.tcp.failcnt
-rw-r--r--. 1 root root 0 Sep  5 10:18 memory.kmem.tcp.limit_in_bytes
-rw-r--r--. 1 root root 0 Sep  5 10:18 memory.kmem.tcp.max_usage_in_bytes
-r--r--r--. 1 root root 0 Sep  5 10:18 memory.kmem.tcp.usage_in_bytes
-r--r--r--. 1 root root 0 Sep  5 10:18 memory.kmem.usage_in_bytes
-rw-r--r--. 1 root root 0 Sep  5 10:18 memory.limit_in_bytes
-rw-r--r--. 1 root root 0 Sep  5 10:18 memory.max_usage_in_bytes
-rw-r--r--. 1 root root 0 Sep  5 10:18 memory.memsw.failcnt
-rw-r--r--. 1 root root 0 Sep  5 10:18 memory.memsw.limit_in_bytes
-rw-r--r--. 1 root root 0 Sep  5 10:18 memory.memsw.max_usage_in_bytes
-r--r--r--. 1 root root 0 Sep  5 10:18 memory.memsw.usage_in_bytes
-rw-r--r--. 1 root root 0 Sep  5 10:18 memory.move_charge_at_immigrate
-r--r--r--. 1 root root 0 Sep  5 10:18 memory.numa_stat
-rw-r--r--. 1 root root 0 Sep  5 10:18 memory.oom_control
----------. 1 root root 0 Sep  5 10:18 memory.pressure_level
-rw-r--r--. 1 root root 0 Sep  5 10:18 memory.soft_limit_in_bytes
-r--r--r--. 1 root root 0 Sep  5 10:18 memory.stat
-rw-r--r--. 1 root root 0 Sep  5 10:18 memory.swappiness
-r--r--r--. 1 root root 0 Sep  5 10:18 memory.usage_in_bytes
-rw-r--r--. 1 root root 0 Sep  5 10:18 memory.use_hierarchy
-rw-r--r--. 1 root root 0 Sep  5 10:18 notify_on_release
-rw-r--r--. 1 root root 0 Sep  5 10:18 tasks
```

其中 `memory.limit_in_bytes` 文件代表内存使用总量，单位为 byte。

例如，这里我希望对内存使用限制为 1G，则向 memory.limit_in_bytes 文件写入 1073741824，命令如下：



**第二步：创建进程，加入 cgroup**
同样把当前 shell 进程 ID 写入 tasks 文件内：

```sh
$ cd /sys/fs/cgroup/memory/mydocker
$ echo $$ > tasks 
```



**第三步，执行内存测试工具，申请内存**
这里我们需要借助一下工具 memtester，memtester 的安装这里不再详细介绍了。具体安装方式可以参考[这里](https://wilhelmguo.cn/blog/post/william/CentOS7-%E5%AE%89%E8%A3%85%E5%86%85%E5%AD%98%E6%B5%8B%E8%AF%95%E5%B7%A5%E5%85%B7-memtester)。

安装好 memtester 后，我们执行以下命令：

```sh
$ memtester 1500M 1
memtester version 4.2.2 (64-bit)
Copyright (C) 2010 Charles Cazabon.
Licensed under the GNU General Public License version 2 (only).
pagesize is 4096
pagesizemask is 0xfffffffffffff000
want 1500MB (1572864000 bytes)
got  1500MB (1572864000 bytes), trying mlock ...Killed
```

该命令会申请 1500 M 内存，并且做内存测试。由于上面我们对当前 shell 进程内存限制为 1 G，当 memtester 使用的内存达到 1G 时，cgroup 便将 memtester 杀死。

上面最后一行的输出结果表示 memtester 想要 1500 M 内存，但是由于 cgroup 限制，达到了内存使用上限，被杀死了，与我们的预期一致。

我们可以使用以下命令，降低一下内存申请，将内存申请调整为 500M：

```sh
$ memtester 500M 1
memtester version 4.2.2 (64-bit)
Copyright (C) 2010 Charles Cazabon.
Licensed under the GNU General Public License version 2 (only).
pagesize is 4096
pagesizemask is 0xfffffffffffff000
want 500MB (524288000 bytes)
got  500MB (524288000 bytes), trying mlock ...locked.
Loop 1/1:
  Stuck Address       : ok
  Random Value        : ok
  Compare XOR         : ok
  Compare SUB         : ok
  Compare MUL         : ok
  Compare DIV         : ok
  Compare OR          : ok
  Compare AND         : ok
  Sequential Increment: ok
  Solid Bits          : ok
  Block Sequential    : ok
  Checkerboard        : ok
  Bit Spread          : ok
  Bit Flip            : ok
  Walking Ones        : ok
  Walking Zeroes      : ok
  8-bit Writes        : ok
  16-bit Writes       : ok
Done.
```

这里可以看到，此时 memtester 已经成功申请到 500M 内存并且正常完成了内存测试。
到此，我们讲解了cgroups的 cpu 和 memroy 子系统，如果你想了解更多的cgroups的知识和使用，可以参考 Red Hat [官网](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/resource_management_guide/chap-introduction_to_control_groups)。



#### 删除 cgroups

上面创建的cgroups如果不想使用了，直接删除创建的文件夹即可。

例如我想删除内存下的 mydocker 目录，使用以下命令即可：

```sh
$ rmdir /sys/fs/cgroup/memory/mydocker/
```

学习了cgroups的使用方式，下面了解一下 Docker 是如何使用cgroups的。



### Docker 是如何使用cgroups的？

首先，我们使用以下命令创建一个 nginx 容器：

```sh
$ docker run -it -m=1g nginx
```

上述命令创建并启动了一个 nginx 容器，并且限制内存为 1G。然后我们进入cgroups内存子系统的目录，使用 ls 命令查看一下该目录下的内容：

```sh
$ ls -l /sys/fs/cgroup/memory
total 0
-rw-r--r--.  1 root root 0 Sep  1 11:50 cgroup.clone_children
--w--w--w-.  1 root root 0 Sep  1 11:50 cgroup.event_control
-rw-r--r--.  1 root root 0 Sep  1 11:50 cgroup.procs
-r--r--r--.  1 root root 0 Sep  1 11:50 cgroup.sane_behavior
drwxr-xr-x.  3 root root 0 Sep  5 10:50 docker
... 省略部分输出
```

通过上面输出可以看到，该目录下有一个 docker 目录，该目录正是 Docker 在内存子系统下创建的。我们进入到 docker 目录下查看一下相关内容：

```sh
$ cd /sys/fs/cgroup/memory/docker
$ ls -l
total 0
drwxr-xr-x. 2 root root 0 Sep  5 10:49 cb5c5391177b44ad87636bf3840ecdda83529e51b76a6406d6742f56a2535d5e
-rw-r--r--. 1 root root 0 Sep  4 10:40 cgroup.clone_children
--w--w--w-. 1 root root 0 Sep  4 10:40 cgroup.event_control
-rw-r--r--. 1 root root 0 Sep  4 10:40 cgroup.procs
... 省略部分输出
-rw-r--r--. 1 root root 0 Sep  4 10:40 tasks
```

可以看到 docker 的目录下有一个一串随机 ID 的目录，该目录即为我们上面创建的 nginx 容器的 ID。然后我们进入该目录，查看一下该容器的 memory.limit_in_bytes 文件的内容。

```sh
$ cd cb5c5391177b44ad87636bf3840ecdda83529e51b76a6406d6742f56a2535d5e
$ cat memory.limit_in_bytes
1073741824
```

可以看到内存限制值正好为 1G。
事实上，Docker 创建容器时，Docker 会根据启动容器的参数，在对应的 cgroups 子系统下创建以容器 ID 为名称的目录, 然后根据容器启动时设置的资源限制参数, 修改对应的 cgroups 子系统资源限制文件, 从而达到资源限制的效果。



### 小结

本课时我们讲解了什么是 cgroups，以及 cgroups 可以为我们提供哪些核心功能。其实 cgroups 不仅可以实现资源的限制，还可以为我们统计资源的使用情况，容器监控系统的数据来源也是 cgroups 提供的。

另外，请注意 cgroups 虽然可以实现资源的限制，但是不能保证资源的使用。例如，cgroups 限制某个容器最多使用 1 核 CPU，但不保证总是能使用到 1 核 CPU，当 CPU 资源发生竞争时，可能会导致实际使用的 CPU 资源产生竞争。

那么，你知道cgroups还有哪些子系统吗？





## 11 | 组件组成：剖析 Docker 组件作用及其底层工作原理



### Docker 的组件构成

Docker 整体架构采用 C/S（客户端 / 服务器）模式，主要由客户端和服务端两大部分组成。客户端负责发送操作指令，服务端负责接收和处理指令。客户端和服务端通信有多种方式，即可以在同一台机器上通过UNIX套接字通信，也可以通过网络连接远程通信。

![img](D:\学习资料\笔记\k8s\k8s图\CgqCHl9rFtSAPGOeAADIK4E6wrc522.png)



从整体架构可知，Docker 组件大体分为 Docker 相关组件，containerd 相关组件和容器运行时相关组件。下面我们深入剖析下各个组件。



### Docker 组件剖析

Docker 到底有哪些组件呢？我们可以在 Docker 安装路径下执行 ls 命令，这样可以看到以下与 Docker 有关的组件。

```ls
-rwxr-xr-x 1 root root 27941976 Dec 12  2019 containerd
-rwxr-xr-x 1 root root  4964704 Dec 12  2019 containerd-shim
-rwxr-xr-x 1 root root 15678392 Dec 12  2019 ctr
-rwxr-xr-x 1 root root 50683148 Dec 12  2019 docker
-rwxr-xr-x 1 root root   764144 Dec 12  2019 docker-init
-rwxr-xr-x 1 root root  2837280 Dec 12  2019 docker-proxy
-rwxr-xr-x 1 root root 54320560 Dec 12  2019 dockerd
-rwxr-xr-x 1 root root  7522464 Dec 12  2019 runc
```

这些组件根据工作职责可以分为以下三大类。

1. Docker 相关的组件：docker、dockerd、docker-init 和 docker-proxy

2. containerd 相关的组件：containerd、containerd-shim 和 ctr

3. 容器运行时相关的组件：runc



#### Docker 相关的组件

**（1）docker**

docker 是 Docker 客户端的一个完整实现，它是一个二进制文件，对用户可见的操作形式为 docker 命令，通过 docker 命令可以完成所有的 Docker 客户端与服务端的通信（还可以通过 REST API、SDK 等多种形式与 Docker 服务端通信）。

Docker 客户端与服务端的交互过程是：docker 组件向服务端发送请求后，服务端根据请求执行具体的动作并将结果返回给 docker，docker 解析服务端的返回结果，并将结果通过命令行标准输出展示给用户。这样一次完整的客户端服务端请求就完成了。



**（2）dockerd**

dockerd 是 Docker 服务端的后台常驻进程，用来接收客户端发送的请求，执行具体的处理任务，处理完成后将结果返回给客户端。

Docker 客户端可以通过多种方式向 dockerd 发送请求，我们常用的 Docker 客户端与 dockerd 的交互方式有三种。

- 通过 UNIX 套接字与服务端通信：配置格式为unix://socket_path，默认 dockerd 生成的 socket 文件路径为 /var/run/docker.sock，该文件只有 root 用户或者 docker 用户组的用户才可以访问，这就是为什么 Docker 刚安装完成后只有 root 用户才能使用 docker 命令的原因。

- 通过 TCP 与服务端通信：配置格式为tcp://host:port，通过这种方式可以实现客户端远程连接服务端，但是在方便的同时也带有安全隐患，因此在生产环境中如果你要使用 TCP 的方式与 Docker 服务端通信，推荐使用 TLS 认证，可以通过设置 Docker 的 TLS 相关参数，来保证数据传输的安全。

- 通过文件描述符的方式与服务端通信：配置格式为：fd://这种格式一般用于 systemd 管理的系统中。

Docker 客户端和服务端的通信形式必须保持一致，否则将无法通信，只有当 dockerd 监听了 UNIX 套接字客户端才可以使用 UNIX 套接字的方式与服务端通信，UNIX 套接字也是 Docker 默认的通信方式，如果你想要通过远程的方式访问 dockerd，可以在 dockerd 启动的时候添加 -H 参数指定监听的 HOST 和 PORT。



**（3）docker-init**

如果你熟悉 Linux 系统，你应该知道在 Linux 系统中，1 号进程是 init 进程，是所有进程的父进程。主机上的进程出现问题时，init 进程可以帮我们回收这些问题进程。同样的，在容器内部，当我们自己的业务进程没有回收子进程的能力时，在执行 docker run 启动容器时可以添加 --init 参数，此时 Docker 会使用 docker-init 作为1号进程，帮你管理容器内子进程，例如回收僵尸进程等。

下面我们通过启动一个 busybox 容器来演示下：

```sh
$ docker run -it busybox sh
/ # ps aux
PID   USER     TIME  COMMAND
    1 root      0:00 sh
    6 root      0:00 ps aux
/ #
```

可以看到容器启动时如果没有添加 --init 参数，1 号进程就是 sh 进程。

我们使用 Crtl + D 退出当前容器，重新启动一个新的容器并添加 --init 参数，然后看下进程：

```sh
$ docker run -it --init busybox sh
/ # ps aux
PID   USER     TIME  COMMAND
    1 root      0:00 /sbin/docker-init -- sh
    6 root      0:00 sh
    7 root      0:00 ps aux
```

可以看到此时容器内的 1 号进程已经变为 /sbin/docker-init，而不再是 sh 了。



**（4）docker-proxy**

docker-proxy 主要是用来做端口映射的。当我们使用 docker run 命令启动容器时，如果使用了 -p 参数，docker-proxy 组件就会把容器内相应的端口映射到主机上来，底层是依赖于 iptables 实现的。

下面我们通过一个实例演示下。

使用以下命令启动一个 nginx 容器并把容器的 80 端口映射到主机的 8080 端口。

```sh
$ docker run --name=nginx -d -p 8080:80 nginx
```

然后通过以下命令查看一下启动的容器 IP：

```sh
$ docker inspect --format '{{ .NetworkSettings.IPAddress }}' nginx
172.17.0.2
```

可以看到，我们启动的 nginx 容器 IP 为 172.17.0.2。

此时，我们使用 ps 命令查看一下主机上是否有 docker-proxy 进程：

```sh
$ sudo ps aux |grep docker-proxy
root      9100  0.0  0.0 290772  9160 ?        Sl   07:48   0:00 /usr/bin/docker-proxy -proto tcp -host-ip 0.0.0.0 -host-port 8080 -container-ip 172.17.0.2 -container-port 80
root      9192  0.0  0.0 112784   992 pts/0    S+   07:51   0:00 grep --color=auto docker-proxy
```

可以看到当我们启动一个容器时需要端口映射时， Docker 为我们创建了一个 docker-proxy 进程，并且通过参数把我们的容器 IP 和端口传递给 docker-proxy 进程，然后 docker-proxy 通过 iptables 实现了 nat 转发。

我们通过以下命令查看一下主机上 iptables nat 表的规则：

```sh
$  sudo iptables -L -nv -t nat
Chain PREROUTING (policy ACCEPT 35 packets, 2214 bytes)
 pkts bytes target     prot opt in     out     source               destination
  398 21882 DOCKER     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL
Chain INPUT (policy ACCEPT 35 packets, 2214 bytes)
 pkts bytes target     prot opt in     out     source               destination
Chain OUTPUT (policy ACCEPT 1 packets, 76 bytes)
 pkts bytes target     prot opt in     out     source               destination
    0     0 DOCKER     all  --  *      *       0.0.0.0/0           !127.0.0.0/8          ADDRTYPE match dst-type LOCAL
Chain POSTROUTING (policy ACCEPT 1 packets, 76 bytes)
 pkts bytes target     prot opt in     out     source               destination
    0     0 MASQUERADE  all  --  *      !docker0  172.17.0.0/16        0.0.0.0/0
    0     0 MASQUERADE  tcp  --  *      *       172.17.0.2           172.17.0.2           tcp dpt:80
Chain DOCKER (2 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 RETURN     all  --  docker0 *       0.0.0.0/0            0.0.0.0/0
    0     0 DNAT       tcp  --  !docker0 *       0.0.0.0/0            0.0.0.0/0            tcp dpt:8080 to:172.17.0.2:80
```

通过最后一行规则我们可以得知，当我们访问主机的 8080 端口时，iptables 会把流量转发到 172.17.0.2 的 80 端口，从而实现了我们从主机上可以直接访问到容器内的业务。

我们通过 curl 命令访问一下 nginx 容器：

```sh
$ curl http://localhost:8080
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

通过上面的输出可以得知我们已经成功访问到了 nginx 容器。

总体来说，docker 是官方实现的标准客户端，dockerd 是 Docker 服务端的入口，负责接收客户端发送的指令并返回相应结果，而 docker-init 在业务主进程没有进程回收功能时则十分有用，docker-proxy 组件则是实现 Docker 网络访问的重要组件。

了解完 docker 相关的组件，下面我来介绍下 containerd 相关的组件。



#### containerd 相关的组件

**（1）containerd**

containerd 组件是从 Docker 1.11 版本正式从 dockerd 中剥离出来的，它的诞生完全遵循 OCI 标准，是容器标准化后的产物。containerd 完全遵循了 OCI 标准，并且是完全社区化运营的，因此被容器界广泛采用。

containerd 不仅负责容器生命周期的管理，同时还负责一些其他的功能：

- 镜像的管理，例如容器运行前从镜像仓库拉取镜像到本地；

- 接收 dockerd 的请求，通过适当的参数调用 runc 启动容器；

- 管理存储相关资源；

- 管理网络相关资源。

containerd 包含一个后台常驻进程，默认的 socket 路径为 /run/containerd/containerd.sock，dockerd 通过 UNIX 套接字向 containerd 发送请求，containerd 接收到请求后负责执行相关的动作并把执行结果返回给 dockerd。

如果你不想使用 dockerd，也可以直接使用 containerd 来管理容器，由于 containerd 更加简单和轻量，生产环境中越来越多的人开始直接使用 containerd 来管理容器。



**（2）containerd-shim**

containerd-shim 的意思是垫片，类似于拧螺丝时夹在螺丝和螺母之间的垫片。containerd-shim 的主要作用是将 containerd 和真正的容器进程解耦，使用 containerd-shim 作为容器进程的父进程，从而实现重启 containerd 不影响已经启动的容器进程。



**（3）ctr**

ctr 实际上是 containerd-ctr，它是 containerd 的客户端，主要用来开发和调试，在没有 dockerd 的环境中，ctr 可以充当 docker 客户端的部分角色，直接向 containerd 守护进程发送操作容器的请求。

了解完 containerd 相关的组件，我们来了解一下容器的真正运行时 runc。



#### 容器运行时组件runc

runc 是一个标准的 OCI 容器运行时的实现，它是一个命令行工具，可以直接用来创建和运行容器。

下面我们通过一个实例来演示一下 runc 的神奇之处。

第一步，准备容器运行时文件：进入 /home/centos 目录下，创建 runc 文件夹，并导入 busybox 镜像文件。

```sh
 $ cd /home/centos

## 创建 runc 运行根目录
 $ mkdir runc

## 导入 rootfs 镜像文件
 $ mkdir rootfs && docker export $(docker create busybox) | tar -C rootfs -xvf -
```



第二步，生成 runc config 文件。我们可以使用 runc spec 命令根据文件系统生成对应的 config.json 文件。命令如下：

```sh
$ runc spec
```

此时会在当前目录下生成 config.json 文件，我们可以使用 cat 命令查看一下 config.json 的内容：

```json
$ cat config.json
{
	"ociVersion": "1.0.1-dev",
	"process": {
		"terminal": true,
		"user": {
			"uid": 0,
			"gid": 0
		},
		"args": [
			"sh"
		],
		"env": [
			"PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
			"TERM=xterm"
		],
		"cwd": "/",
		"capabilities": {
			"bounding": [
				"CAP_AUDIT_WRITE",
				"CAP_KILL",
				"CAP_NET_BIND_SERVICE"
			],
			"effective": [
				"CAP_AUDIT_WRITE",
				"CAP_KILL",
				"CAP_NET_BIND_SERVICE"
			],
			"inheritable": [
				"CAP_AUDIT_WRITE",
				"CAP_KILL",
				"CAP_NET_BIND_SERVICE"
			],
			"permitted": [
				"CAP_AUDIT_WRITE",
				"CAP_KILL",
				"CAP_NET_BIND_SERVICE"
			],
			"ambient": [
				"CAP_AUDIT_WRITE",
				"CAP_KILL",
				"CAP_NET_BIND_SERVICE"
			]
		},
		"rlimits": [
			{
				"type": "RLIMIT_NOFILE",
				"hard": 1024,
				"soft": 1024
			}
		],
		"noNewPrivileges": true
	},
	"root": {
		"path": "rootfs",
		"readonly": true
	},
	"hostname": "runc",
	"mounts": [
		{
			"destination": "/proc",
			"type": "proc",
			"source": "proc"
		},
		{
			"destination": "/dev",
			"type": "tmpfs",
			"source": "tmpfs",
			"options": [
				"nosuid",
				"strictatime",
				"mode=755",
				"size=65536k"
			]
		},
		{
			"destination": "/dev/pts",
			"type": "devpts",
			"source": "devpts",
			"options": [
				"nosuid",
				"noexec",
				"newinstance",
				"ptmxmode=0666",
				"mode=0620",
				"gid=5"
			]
		},
		{
			"destination": "/dev/shm",
			"type": "tmpfs",
			"source": "shm",
			"options": [
				"nosuid",
				"noexec",
				"nodev",
				"mode=1777",
				"size=65536k"
			]
		},
		{
			"destination": "/dev/mqueue",
			"type": "mqueue",
			"source": "mqueue",
			"options": [
				"nosuid",
				"noexec",
				"nodev"
			]
		},
		{
			"destination": "/sys",
			"type": "sysfs",
			"source": "sysfs",
			"options": [
				"nosuid",
				"noexec",
				"nodev",
				"ro"
			]
		},
		{
			"destination": "/sys/fs/cgroup",
			"type": "cgroup",
			"source": "cgroup",
			"options": [
				"nosuid",
				"noexec",
				"nodev",
				"relatime",
				"ro"
			]
		}
	],
	"linux": {
		"resources": {
			"devices": [
				{
					"allow": false,
					"access": "rwm"
				}
			]
		},
		"namespaces": [
			{
				"type": "pid"
			},
			{
				"type": "network"
			},
			{
				"type": "ipc"
			},
			{
				"type": "uts"
			},
			{
				"type": "mount"
			}
		],
		"maskedPaths": [
			"/proc/acpi",
			"/proc/asound",
			"/proc/kcore",
			"/proc/keys",
			"/proc/latency_stats",
			"/proc/timer_list",
			"/proc/timer_stats",
			"/proc/sched_debug",
			"/sys/firmware",
			"/proc/scsi"
		],
		"readonlyPaths": [
			"/proc/bus",
			"/proc/fs",
			"/proc/irq",
			"/proc/sys",
			"/proc/sysrq-trigger"
		]
	}
}
```

config.json 文件定义了 runc 启动容器时的一些配置，如根目录的路径，文件挂载路径等配置。



第三步，使用 runc 启动容器。我们可以使用 runc run 命令直接启动 busybox 容器。

```sh
$ runc run busybox
/ #
```

此时，我们已经创建并启动了一个 busybox 容器。

我们新打开一个命令行窗口，可以使用 run list 命令看到刚才启动的容器。

```sh
$ cd /home/centos/runc/
$ runc list
D          PID         STATUS      BUNDLE              CREATED                          OWNER
busybox     9778        running     /home/centos/runc   2020-09-06T09:25:32.441957273Z   root
```

通过上面的输出，我们可以看到，当前已经有一个 busybox 容器处于运行状态。

总体来说，Docker 的组件虽然很多，但每个组件都有自己清晰的工作职责，Docker 相关的组件负责发送和接受 Docker 请求，contianerd 相关的组件负责管理容器的生命周期，而 runc 负责真正意义上创建和启动容器。这些组件相互配合，才使得 Docker 顺利完成了容器的管理工作。



### 总结

到此，相信你已经完全掌握了 Docker 的组件构成，各个组件的作用和工作原理。本节课时的重点总结如下。

![img](D:\学习资料\笔记\k8s\k8s图\Ciqc1F9y4vGAVzmAAADk1nlHpUA424.png)

Docker 当前的架构有什么弊端吗？

> 当前 docker 的问题就是调用链过长，出现问题需要分析很多组件才能定位问题。





## 12 | 网络模型：剖析 Docker 网络实现及 Libnetwork 底层原理



### 容器网络发展史

提起 Docker 网络，我们不得不从容器战争说起。Docker 从 2013 年诞生，到后来逐渐成为了容器的代名词，然而 Docker 的野心也不止于此，它还想在更多的领域独占鳌头，比如制定容器的网络和存储标准。

于是 Docker 从 1.7 版本开始，便把网络和存储从 Docker 中正式以插件的形式剥离开来，并且分别为其定义了标准，Docker 定义的网络模型标准称之为 CNM (Container Network Model) 。

> Docker 推出 CNM 的同时，CoreOS 推出了 CNI（Container Network Model）。起初，以 Kubernetes 为代表的容器编排阵营考虑过使用 CNM 作为容器的网络标准，但是后来由于很多技术和非技术原因（如果你对详细原因感兴趣，可以参考这篇博客），Kubernetes 决定支持 CoreOS 推出的容器网络标准 CNI。

从此，容器的网络标准便分为两大阵营，一个是以 Docker 公司为代表的 CNM，另一个便是以 Google、Kubernetes、CoreOS 为代表的 CNI 网络标准。



### CNM

CNM (Container Network Model) 是 Docker 发布的容器网络标准，意在规范和指定容器网络发展标准，CNM 抽象了容器的网络接口 ，使得只要满足 CNM 接口的网络方案都可以接入到 Docker 容器网络，更好地满足了用户网络模型多样化的需求。

CNM 只是定义了网络标准，对于底层的具体实现并不太关心，这样便解耦了容器和网络，使得容器的网络模型更加灵活。

CNM 定义的网络标准包含三个重要元素。

- **沙箱（Sandbox）**：沙箱代表了一系列网络堆栈的配置，其中包含路由信息、网络接口等网络资源的管理，沙箱的实现通常是 Linux 的 Net Namespace，但也可以通过其他技术来实现，比如 FreeBSD jail 等。
- **接入点（Endpoint）**：接入点将沙箱连接到网络中，代表容器的网络接口，接入点的实现通常是 Linux 的 veth 设备对。
- **网络（Network）**：网络是一组可以互相通信的接入点，它将多接入点组成一个子网，并且多个接入点之间可以相互通信。

CNM 的三个要素基本抽象了所有网络模型，使得网络模型的开发更加规范。

为了更好地构建容器网络标准，Docker 团队把网络功能从 Docker 中剥离出来，成为独立的项目 libnetwork，它通过插件的形式为 Docker 提供网络功能。Libnetwork 是开源的，使用 Golang 编写，它完全遵循 CNM 网络规范，是 CNM 的官方实现。Libnetwork 的工作流程也是完全围绕 CNM 的三个要素进行的，下面我们来详细了解一下 Libnetwork 是如何围绕 CNM 的三要素工作的。



### Libnetwork 的工作流程

Libnetwork 是 Docker 启动容器时，用来为 Docker 容器提供网络接入功能的插件，它可以让 Docker 容器顺利接入网络，实现主机和容器网络的互通。下面，我们来详细了解一下 Libnetwork 是如何为 Docker 容器提供网络的。

第一步：Docker 通过调用 libnetwork.New 函数来创建 NetworkController 实例。NetworkController 是一个接口类型，提供了各种接口，代码如下：

```sh
type NetworkController interface {
   // 创建一个新的网络。 options 参数用于指定特性类型的网络选项。
   NewNetwork(networkType, name string, id string, options ...NetworkOption) (Network, error)
   // ... 此次省略部分接口
}
```

第二步：通过调用 NewNetwork 函数创建指定名称和类型的 Network，其中 Network 也是接口类型，代码如下:

```
type Network interface {
   // 为该网络创建一个具有唯一指定名称的接入点（Endpoint）
   CreateEndpoint(name string, options ...EndpointOption) (Endpoint, error)
   // 删除网络
   Delete() error
// ... 此次省略部分接口
}
```

第三步：通过调用 CreateEndpoint 来创建接入点（Endpoint）。在 CreateEndpoint 函数中为容器分配了 IP 和网卡接口。其中 Endpoint 也是接口类型，代码如下：

```sh
// Endpoint 表示网络和沙箱之间的逻辑连接。
type Endpoint interface {
   // 将沙箱连接到接入点，并将为接入点分配的网络资源填充到沙箱中。
   // the network resources allocated for the endpoint.
   Join(sandbox Sandbox, options ...EndpointOption) error
   // 删除接入点
   Delete(force bool) error
   // ... 此次省略部分接口
}
```

第四步：调用 NewSandbox 来创建容器沙箱，主要是初始化 Namespace 相关的资源。

第五步：调用 Endpoint 的 Join 函数将沙箱和网络接入点关联起来，此时容器就加入了 Docker 网络并具备了网络访问能力。

Libnetwork 基于以上工作流程可以构建出多种网络模式，以满足我们的在不同场景下的需求，下面我们来详细了解一下 Libnetwork 提供的常见的四种网络模式。



### Libnetwork 常见网络模式

Libnetwork 比较典型的网络模式主要有四种，这四种网络模式基本满足了我们单机容器的所有场景。

1. **null 空网络模式**：可以帮助我们构建一个没有网络接入的容器环境，以保障数据安全。

2. **bridge 桥接模式**：可以打通容器与容器间网络通信的需求。

3. **host 主机网络模式**：可以让容器内的进程共享主机网络，从而监听或修改主机网络。

4. **container 网络模式**：可以将两个容器放在同一个网络命名空间内，让两个业务通过 localhost 即可实现访问。

下面我们对 libnetwork 的四种网络模式逐一讲解：

#### （1）null 空网络模式

有时候，我们需要处理一些保密数据，出于安全考虑，我们需要一个隔离的网络环境执行一些纯计算任务。这时候 null 网络模式就派上用场了，这时候我们的容器就像一个没有联网的电脑，处于一个相对较安全的环境，确保我们的数据不被他人从网络窃取。

使用 Docker 创建 null 空网络模式的容器时，容器拥有自己独立的 Net Namespace，但是此时的容器并没有任何网络配置。在这种模式下，Docker 除了为容器创建了 Net Namespace 外，没有创建任何网卡接口、IP 地址、路由等网络配置。我们可以一起来验证下。

我们使用 docker run 命令启动时，添加 --net=none 参数启动一个空网络模式的容器，命令如下：

```sh
$ docker run --net=none -it busybox
/ #
```

容器启动后，我们使用 ifconfig 命令查看一下容器内网络配置信息：

```sh
/ # ifconfig
lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

可以看到容器内除了 Net Namespace 自带的 lo 网卡并没有创建任何虚拟网卡，然后我们再使用 route -n 命令查看一下容器内的路由信息:

```sh
/ # route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
```

可以看到，容器内也并没有配置任何路由信息。



#### （2）bridge 桥接模式

**Docker 的 bridge 网络是启动容器时默认的网络模式**，使用 bridge 网络可以实现容器与容器的互通，可以从一个容器直接通过容器 IP 访问到另外一个容器。同时使用 bridge 网络可以实现主机与容器的互通，我们在容器内启动的业务，可以从主机直接请求。

在介绍 Docker 的 bridge 桥接模式前，我们需要先了解一下 Linux 的 veth 和 bridge 相关的技术，因为 Docker 的 bridge 模式正是由这两种技术实现的。

- Linux veth

veth 是 Linux 中的虚拟设备接口，veth 都是成对出现的，它在容器中，通常充当一个桥梁。veth 可以用来连接虚拟网络设备，例如 veth 可以用来连通两个 Net Namespace，从而使得两个 Net Namespace 之间可以互相访问。

- Linux bridge

Linux bridge 是一个虚拟设备，是用来连接网络的设备，相当于物理网络环境中的交换机。Linux bridge 可以用来转发两个 Net Namespace 内的流量。

- veth 与 bridge 的关系

![img](D:\学习资料\笔记\k8s\k8s图\Ciqc1F9y8IKAa-1NAABjDM-2kBk665.png)

通过图 1 ，我们可以看到，bridge 就像一台交换机，而 veth 就像一根网线，通过交换机和网线可以把两个不同 Net Namespace 的容器连通，使得它们可以互相通信。

Docker 的 bridge 模式也是这种原理。Docker 启动时，libnetwork 会在主机上创建 docker0 网桥，docker0 网桥就相当于图 1 中的交换机，而 Docker 创建出的 brige 模式的容器则都会连接 docker0 上，从而实现网络互通。

**bridge 桥接模式是 Docker 的默认网络模式，当我们创建容器时不指定任何网络模式，Docker 启动容器默认的网络模式为 bridge。**



#### （3）host 主机网络模式

容器内的网络并不是希望永远跟主机是隔离的，有些基础业务需要创建或更新主机的网络配置，我们的程序必须以主机网络模式运行才能够修改主机网络，这时候就需要用到 Docker 的 host 主机网络模式。

使用 host 主机网络模式时：

- libnetwork 不会为容器创建新的网络配置和 Net Namespace。

- Docker 容器中的进程直接共享主机的网络配置，可以直接使用主机的网络信息，此时，在容器内监听的端口，也将直接占用到主机的端口。

- 除了网络共享主机的网络外，其他的包括进程、文件系统、主机名等都是与主机隔离的。

host 主机网络模式通常适用于想要使用主机网络，但又不想把运行环境直接安装到主机上的场景中。例如我想在主机上运行一个 busybox 服务，但又不想直接把 busybox 安装到主机上污染主机环境，此时我可以使用以下命令启动一个主机网络模式的 busybox 镜像：

```sh
$ docker run -it --net=host busybox
/ #
```

然后我们使用ip a 命令查看一下容器内的网络环境：

```sh
/ # ip a
1: lo: <LOOPBACK,UP,LOWER\_UP> mtu 65536 qdisc noqueue qlen 1000
link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
inet 127.0.0.1/8 scope host lo
valid\_lft forever preferred\_lft forever
inet6 ::1/128 scope host
valid\_lft forever preferred\_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc pfifo\_fast qlen 1000
link/ether 02:11:b0:14:01:0c brd ff:ff:ff:ff:ff:ff
inet 172.20.1.11/24 brd 172.20.1.255 scope global dynamic eth0
valid\_lft 85785286sec preferred\_lft 85785286sec
inet6 fe80::11:b0ff:fe14:10c/64 scope link
valid\_lft forever preferred\_lft forever
3: docker0: \<NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue
link/ether 02:42:82:8d:a0:df brd ff:ff:ff:ff:ff:ff
inet 172.17.0.1/16 scope global docker0
valid\_lft forever preferred\_lft forever
inet6 fe80::42:82ff:fe8d:a0df/64 scope link
valid\_lft forever preferred\_lft forever
```

可以看到容器内的网络环境与主机完全一致。



#### （4）container 网络模式

container 网络模式允许一个容器共享另一个容器的网络命名空间。当两个容器需要共享网络，但其他资源仍然需要隔离时就可以使用 container 网络模式，例如我们开发了一个 http 服务，但又想使用 nginx 的一些特性，让 nginx 代理外部的请求然后转发给自己的业务，这时我们使用 container 网络模式将自己开发的服务和 nginx 服务部署到同一个网络命名空间中。

下面我举例说明。首先我们使用以下命令启动一个 busybox1 容器：

```sh
$ docker run -d --name=busybox1 busybox sleep 3600
```

然后我们使用 docker exec 命令进入到 centos 容器中查看一下网络配置：

```sh
$ docker exec -it busybox1 sh
/ # ifconfig
eth0 Link encap:Ethernet HWaddr 02:42:AC:11:00:02
inet addr:172.17.0.2 Bcast:172.17.255.255 Mask:255.255.0.0
UP BROADCAST RUNNING MULTICAST MTU:1500 Metric:1
RX packets:11 errors:0 dropped:0 overruns:0 frame:0
TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
collisions:0 txqueuelen:0
RX bytes:906 (906.0 B) TX bytes:0 (0.0 B)
lo Link encap:Local Loopback
inet addr:127.0.0.1 Mask:255.0.0.0
UP LOOPBACK RUNNING MTU:65536 Metric:1
RX packets:0 errors:0 dropped:0 overruns:0 frame:0
TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
collisions:0 txqueuelen:1000
RX bytes:0 (0.0 B) TX bytes:0 (0.0 B)
```

可以看到 busybox1 的 IP 地址为 172.17.0.2。

然后我们新打开一个命令行窗口，再启动一个 busybox2 容器，通过 container 网络模式连接到 busybox1 的网络，命令如下：

```sh
$ docker run -it --net=container:busybox1 --name=busybox2 busybox sh
/ #
```

在 busybox2 容器内同样使用 ifconfig 命令查看一下容器内的网络配置：

```sh
/ # ifconfig
eth0 Link encap:Ethernet HWaddr 02:42:AC:11:00:02
inet addr:172.17.0.2 Bcast:172.17.255.255 Mask:255.255.0.0
UP BROADCAST RUNNING MULTICAST MTU:1500 Metric:1
RX packets:14 errors:0 dropped:0 overruns:0 frame:0
TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
collisions:0 txqueuelen:0
RX bytes:1116 (1.0 KiB) TX bytes:0 (0.0 B)
lo Link encap:Local Loopback
inet addr:127.0.0.1 Mask:255.0.0.0
UP LOOPBACK RUNNING MTU:65536 Metric:1
RX packets:0 errors:0 dropped:0 overruns:0 frame:0
TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
collisions:0 txqueuelen:1000
RX bytes:0 (0.0 B) TX bytes:0 (0.0 B)
```

可以看到 busybox2 容器的网络 IP 也为 172.17.0.2，与 busybox1 的网络一致。

以上就是 Libnetwork 常见的四种网络模式，它们的作用及业务场景总结如下：

![img](D:\学习资料\笔记\k8s\k8s图\Ciqc1F9y8HGAaH1iAAClKDUq5FY736.png)



### 结语

我上面有说到 Libnetwork 的工作流程是完全围绕 CNM 的三个要素进行的，CNM 制定标准之初不仅仅是为了单台主机上的容器互通，更多的是为了定义跨主机之间的容器通信标准。但是后来由于 Kubernetes 逐渐成为了容器编排的标准，而 Kubernetes 最终选择了 CNI 作为容器网络的定义标准（具体原因可以参考 [这里](https://kubernetes.io/blog/2016/01/why-kubernetes-doesnt-use-libnetwork/)），很遗憾 CNM 最终没有成为跨主机容器通信的标准，但是CNM 却为推动容器网络标准做出了重大贡献，且 Libnetwork 也是 Docker 的默认网络实现，提供了单独使用 Docker 容器时的多种网络接入功能。

那你知道 libnetwork 除了我讲的四种网络模式外，还有什么网络模式吗？





## 13 | 数据存储：剖析 Docker 卷与持久化数据存储的底层原理



### 为什么容器需要持久化存储

容器按照业务类型，总体可以分为两类：

- 无状态的（数据不需要被持久化）

- 有状态的（数据需要被持久化）

显然，容器更擅长无状态应用。因为未持久化数据的容器根目录的生命周期与容器的生命周期一样，容器文件系统的本质是在镜像层上面创建的读写层，运行中的容器对任何文件的修改都存在于该读写层，当容器被删除时，容器中的读写层也会随之消失。

虽然容器希望所有的业务都尽量保持无状态，这样容器就可以开箱即用，并且可以任意调度，但实际业务总是有各种需要数据持久化的场景，比如 MySQL、Kafka 等有状态的业务。因此为了解决有状态业务的需求，Docker 提出了卷（Volume）的概念。

什么是卷？卷的本质是文件或者目录，它可以绕过默认的联合文件系统，直接以文件或目录的形式存在于宿主机上。卷的概念不仅解决了数据持久化的问题，还解决了容器间共享数据的问题。使用卷可以将容器内的目录或文件持久化，当容器重启后保证数据不丢失，例如我们可以使用卷将 MySQL 的目录持久化，实现容器重启数据库数据不丢失。

Docker 提供了卷（Volume）的功能，使用docker volume命令可以实现对卷的创建、查看和删除等操作。下面我们来详细了解一下这些命令。



### Docker 卷的操作

#### 创建数据卷

使用`docker volume create`命令可以创建一个数据卷。

我们使用以下命令创建一个名为 myvolume 的数据卷：

```sh
$ docker volume create myvolume
```

在这里要说明下，默认情况下 ，Docker 创建的数据卷为 local 模式，仅能提供本主机的容器访问。如果想要实现远程访问，需要借助网络存储来实现。Docker 的 local 存储模式并未提供配额管理，因此在生产环境中需要手动维护磁盘存储空间。

除了使用`docker volume create`的方式创建卷，我们还可以在 Docker 启动时使用 -v 的方式指定容器内需要被持久化的路径，Docker 会自动为我们创建卷，并且绑定到容器中，使用命令如下：

```sh
$ docker run -d --name=nginx-volume -v /usr/share/nginx/html nginx
```

使用以上命令，我们启动了一个 nginx 容器，-v参数使得 Docker 自动生成一个卷并且绑定到容器的 /usr/share/nginx/html 目录中。
我们可以使用`docker volume ls`命令来查看下主机上的卷：

```sh
$ docker volume ls
DRIVER              VOLUME NAME
local               eaa8a223eb61a2091bf5cd5247c1b28ac287450a086d6eee9632d9d1b9f69171
```

可以看到，Docker 自动为我们创建了一个名称为随机 ID 的卷。



#### 查看数据卷

已经创建的数据卷可以使用 `docker volume ls` 命令查看。

```sh
$ docker volume ls
DRIVER              VOLUME NAME
local               myvolume
```

通过输出可以看到 myvolume 卷已经创建成功。
如果想要查看某个数据卷的详细信息，可以使用`docker volume inspect`命令。例如，我想查看 myvolume 的详细信息，命令如下：

```sh
$ docker volume inspect myvolume
    {
        "CreatedAt": "2020-09-08T09:10:50Z",
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/myvolume/_data",
        "Name": "myvolume",
        "Options": {},
        "Scope": "local"
    }
]
```

通过`docker volume inspect`命令可以看到卷的创建日期、命令、挂载路径信息。



#### 使用数据卷

使用docker volume创建的卷在容器启动时，添加 --mount 参数指定卷的名称即可使用。

这里我们使用上一步创建的卷来启动一个 nginx 容器，并将 /usr/share/nginx/html 目录与卷关联，命令如下：

```sh
$ docker run -d --name=nginx --mount source=myvolume,target=/usr/share/nginx/html nginx
```

使用 Docker 的卷可以实现指定目录的文件持久化，下面我们进入容器中并且修改 index.html 文件内容，命令如下：

```sh
$ docker exec -it  nginx bash
## 使用以下内容直接替换 /usr/share/nginx/html/index.html 文件 
root@719d3c32e211:/# cat <<EOF >/usr/share/nginx/html/index.html
<!DOCTYPE html>
<html>
<head>
<title>Hello, Docker Volume!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Hello, Docker Volume!</h1>
</body>
</html>
EOF
```

此时我们使用docker rm命令将运行中的 nginx 容器彻底删除。

```sh
$ docker rm -f nginx
```

旧的 nginx 容器删除后，我们再使用docker run命令启动一个新的容器，并且挂载 myvolume 卷，命令如下。

```sh
$ docker run -d --name=nginx --mount source=myvolume,target=/usr/share/nginx/html nginx
```

新容器启动后，我们进入容器查看一下 index.html 文件内容：

```sh
$ docker exec -it nginx bash
root@7ffac645f431:/# cat /usr/share/nginx/html/index.html
<!DOCTYPE html>
<html>
<head>
<title>Hello, Docker Volume!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Hello, Docker Volume!</h1>
</body>
</html>
```

可以看到，此时 index.html 文件内容依旧为我们之前写入的内容。可见，使用 Docker 卷后我们的数据并没有随着容器的删除而消失。



#### 删除数据卷

容器的删除并不会自动删除已经创建的数据卷，因此不再使用的数据卷需要我们手动删除，删除的命令为 `docker volume rm` 。例如，我们想要删除上面创建 myvolume 数据卷，可以使用以下命令：

```sh
$ docker volume rm myvolume
```

> 这里需要注意，正在被使用中的数据卷无法删除，如果你想要删除正在使用中的数据卷，需要先删除所有关联的容器。

有时候，两个容器之间会有共享数据的需求，很典型的一个场景就是容器内产生的日志需要一个专门的日志采集程序去采集日志内容，例如我需要使用 Filebeat (一种日志采集工具)采集 nginx 容器内的日志，我就需要使用卷来共享一个日志目录，从而使得 Filebeat 和 nginx 容器都可以访问到这个目录，这时就需要用到容器之间共享数据卷的方式。



#### 容器与容器之间数据共享

那如何实现容器与容器之间数据共享呢？

首先使用`docker volume creat`e命令创建一个共享日志的数据卷。

```sh
$ docker volume create log-vol
```

启动一个生产日志的容器（下面用 producer 窗口来表示）：

```sh
$ docker run --mount source=log-vol,target=/tmp/log --name=log-producer -it busybox
```

然后新打开一个命令行窗口，启动一个消费者容器（下面用 consumer 窗口来表示）：

```sh
docker run -it --name consumer --volumes-from log-producer  busybox
```

使用**volumes-from**参数可以在启动新的容器时来挂载已经存在的容器的卷，volumes-from参数后面跟已经启动的容器名称。

下面我们切换到 producer 窗口，使用以下命令创建一个 mylog.log 文件并写入 "Hello，My log." 的内容：

```sh
/ # cat <<EOF >/tmp/log/mylog.log
Hello, My log.
EOF
```

然后我们切换到 consumer 窗口，查看一下相关内容：

```sh
/ # cat /tmp/log/mylog.log
Hello, My log.
```

可以看到我们从 producer 容器写入的文件内容会自动出现在 consumer 容器中，证明我们成功实现了两个容器间的数据共享。

总结一下，我们首先使用 `docker volume create` 命令创建了 log-vol 卷来作为共享目录，log-producer 容器向该卷写入数据，consumer 容器从该卷读取数据。这就像主机上的两个进程，一个向主机目录写数据，一个从主机目录读数据，利用主机的目录，实现了容器之间的数据共享。



#### 主机与容器之间数据共享

Docker 卷的目录默认在 /var/lib/docker 下，当我们想把主机的其他目录映射到容器内时，就需要用到主机与容器之间数据共享的方式了，例如我想把 MySQL 容器中的 /var/lib/mysql 目录映射到主机的 /var/lib/mysql 目录中，我们就可以使用主机与容器之间数据共享的方式来实现。

要实现主机与容器之间数据共享，其实很简单，只需要我们在启动容器的时候添加-v参数即可, 使用格式为：`-v HOST_PATH:CONTIANAER_PATH`。

例如，我想挂载主机的 /data 目录到容器中的 /usr/local/data 中，可以使用以下命令来启动容器：

```sh
$ docker run -v /data:/usr/local/data -it busybox
```

容器启动后，便可以在容器内的 /usr/local/data 访问到主机 /data 目录的内容了，并且容器重启后，/data 目录下的数据也不会丢失。

以上就是 Docker 卷的操作，关键命令总结如下：

![img](D:\学习资料\笔记\k8s\k8s图\Ciqc1F-BW1SAQEkaAACOwJuMTHI950.png)



### Docker 卷的实现原理

在了解 Docker 卷的原理之前，我们先来回顾一下镜像和容器的文件系统原理。

> 镜像和容器的文件系统原理： 镜像是由多层文件系统组成的，当我们想要启动一个容器时，Docker 会在镜像上层创建一个可读写层，容器中的文件都工作在这个读写层中，当容器删除时，与容器相关的工作文件将全部丢失。

Docker 容器的文件系统不是一个真正的文件系统，而是通过联合文件系统实现的一个伪文件系统，而 Docker 卷则是直接利用主机的某个文件或者目录，它可以绕过联合文件系统，直接挂载主机上的文件或目录到容器中，这就是它的工作原理。

下面，我们通过一个实例来说明卷的工作原理。首先，我们创建一个名称为 volume-data 的卷：

```sh
$ docker volume create volume-data
```

我们使用 ls 命令查看一下 /var/lib/docker/volumes 目录下的内容：

```sh
$ sudo ls -l /var/lib/docker/volumes
drwxr-xr-x. 3 root root    19 Sep  8 10:59 volume-data
```

然后再看下 volume-data 目录下有什么内容：

```sh
$ sudo ls -l /var/lib/docker/volumes/volume-data
total 0
drwxr-xr-x. 2 root root 6 Sep  8 10:59 _data
```

可以看到我们创建的卷出现在了 /var/lib/docker/volumes 目录下，并且 volume-data 目录下还创建了一个 _data 目录。

实际上，在我们创建 Docker 卷时，Docker 会把卷的数据全部放在 /var/lib/docker/volumes 目录下，并且在每个对应的卷的目录下创建一个 _data 目录，然后把 _data 目录绑定到容器中。因此我们在容器中挂载卷的目录下操作文件，实际上是在操作主机上的 _data 目录。为了证实我的说法，我们来实际演示下。

首先，我们启动一个容器，并且绑定 volume-data 卷到容器内的 /data 目录下：

```sh
$  docker run -it --mount source=volume-data,target=/data busybox
/ #
```

我们进入到容器的 /data 目录，创建一个 data.log 文件:

```sh
/ # cd data/
/data
# touch data.log
```

然后我们新打开一个命令行窗口，查看一下主机上的文件内容：

```sh
$  sudo ls -l /var/lib/docker/volumes/volume-data/_data
total 0
-rw-r--r--. 1 root root 0 Sep  8 11:15 data.log
```

可以看到主机上的 _data 目录下也出现了 data.log 文件。这说明，在容器内操作卷挂载的目录就是直接操作主机上的 _data 目录，符合我上面的说法。

综上，Docker 卷的实现原理是在主机的 /var/lib/docker/volumes 目录下，根据卷的名称创建相应的目录，然后在每个卷的目录下创建 _data 目录，在容器启动时如果使用 --mount 参数，Docker 会把主机上的目录直接映射到容器的指定目录下，实现数据持久化。



### 结语

到此，相信你已经了解了 Docker 使用卷做持久化存储的必要性，也了解 Docker 卷的常用操作，并且对卷的实现原理也有了较清晰的认识。

那么，你知道 Docker 如何使用卷来挂载 NFS 类型的持久化存储到容器内吗？





## 14 | 文件存储驱动：AUFS 文件系统原理及生产环境的最佳配置



我们知道，Docker 主要是基于 Namespace、cgroups 和联合文件系统这三大核心技术实现的。前面的课时我详细讲解了 Namespace 和 cgroups 的相关原理，那么你知道联合文件系统是什么吗？它的原理又是什么呢？

首先我们来了解一下什么是联合文件系统。



### 什么是联合文件系统

联合文件系统（Union File System，Unionfs）是一种分层的轻量级文件系统，它可以把多个目录内容联合挂载到同一目录下，从而形成一个单一的文件系统，这种特性可以让使用者像是使用一个目录一样使用联合文件系统。

那联合文件系统对于 Docker 是一个怎样的存在呢？它可以说是 Docker 镜像和容器的基础，因为它可以使 Docker 可以把镜像做成分层的结构，从而使得镜像的每一层可以被共享。例如两个业务镜像都是基于 CentOS 7 镜像构建的，那么这两个业务镜像在物理机上只需要存储一次 CentOS 7 这个基础镜像即可，从而节省大量存储空间。

说到这儿，你有没有发现，联合文件系统只是一个概念，真正实现联合文件系统才是关键，那如何实现呢？其实实现方案有很多，Docker 中最常用的联合文件系统有三种：AUFS、Devicemapper 和 OverlayFS。

今天我主要讲解 Docker 中最常用的联合文件系统里的 AUFS，为什么呢？因为 AUFS 是 Docker 最早使用的文件系统驱动，多用于 Ubuntu 和 Debian 系统中。在 Docker 早期，OverlayFS 和 Devicemapper 相对不够成熟，AUFS 是最早也是最稳定的文件系统驱动。 Devicemapper 和 OverlayFS 联合文件系统，我将在第 15 和 16 课时为你详细剖析 。

接下来，我们就看看如何配置 Docker 的 AUFS 模式。



### 如何配置 Docker 的 AUFS 模式

AUFS 目前并未被合并到 Linux 内核主线，因此只有 Ubuntu 和 Debian 等少数操作系统支持 AUFS。你可以使用以下命令查看你的系统是否支持 AUFS：

```sh
$ grep aufs /proc/filesystems
nodev   aufs
```

执行以上命令后，如果输出结果包含aufs，则代表当前操作系统支持 AUFS。AUFS 推荐在 Ubuntu 或 Debian 操作系统下使用，如果你想要在 CentOS 等操作系统下使用 AUFS，需要单独安装 AUFS 模块（生产环境不推荐在 CentOS 下使用 AUFS，如果你想在 CentOS 下安装 AUFS 用于研究和测试，可以参考这个链接），安装完成后使用上述命令输出结果中有aufs即可。
当确认完操作系统支持 AUFS 后，你就可以配置 Docker 的启动参数了。

先在 /etc/docker 下新建 daemon.json 文件，并写入以下内容：

```json
{
  "storage-driver": "aufs"
}
```

然后使用以下命令重启 Docker：

```sh
$ sudo systemctl restart docker
```

Docker 重启以后使用`docker info`命令即可查看配置是否生效：

```sh
$ sudo docker info
Client:
 Debug Mode: false
Server:
 Containers: 0
  Running: 0
  Paused: 0
  Stopped: 0
 Images: 1
 Server Version: 19.03.12
 Storage Driver: aufs
  Root Dir: /var/lib/docker/aufs
  Backing Filesystem: extfs
  Dirs: 1
  Dirperm1 Supported: true
```

可以看到 Storage Driver 已经变为 aufs，证明配置已经生效，配置生效后就可以使用 AUFS 为 Docker 提供联合文件系统了。



### AUFS 工作原理



#### AUFS 是如何存储文件的？

AUFS 是联合文件系统，意味着它在主机上使用多层目录存储，每一个目录在 AUFS 中都叫作分支，而在 Docker 中则称之为层（layer），但最终呈现给用户的则是一个普通单层的文件系统，我们把多层以单一层的方式呈现出来的过程叫作联合挂载。

![img](D:\学习资料\笔记\k8s\k8s图\CgqCHl-GwcCAOu4aAABzKSlpRlI180.png)



如图 1 所示，每一个镜像层和容器层都是 /var/lib/docker 下的一个子目录，镜像层和容器层都在 aufs/diff 目录下，每一层的目录名称是镜像或容器的 ID 值，联合挂载点在 aufs/mnt 目录下，mnt 目录是真正的容器工作目录。

下面我们针对 aufs 文件夹下的各目录结构，在创建容器前后的变化做详细讲述。

当一个镜像未生成容器时，AUFS 的存储结构如下。

- diff 文件夹：存储镜像内容，每一层都存储在以镜像层 ID 命名的子文件夹中。

- layers 文件夹：存储镜像层关系的元数据，在 diff 文件夹下的每个镜像层在这里都会有一个文件，文件的内容为该层镜像的父级镜像的 ID。

- mnt 文件夹：联合挂载点目录，未生成容器时，该目录为空。



当一个镜像已经生成容器时，AUFS 存储结构会发生如下变化。

- diff 文件夹：当容器运行时，会在 diff 目录下生成容器层。

- layers 文件夹：增加容器层相关的元数据。

- mnt 文件夹：容器的联合挂载点，这和容器中看到的文件内容一致。

以上便是 AUFS 的工作原理，那你知道容器的在工作过程中是如何使用 AUFS 的吗？



### AUFS 是如何工作的？

AUFS 的工作过程中对文件的操作分为读取文件和修改文件。下面我们分别来看下 AUFS 对于不同的文件操作是如何工作的。

**1. 读取文件**

当我们在容器中读取文件时，可能会有以下场景。

- 文件在容器层中存在时：当文件存在于容器层时，直接从容器层读取。

- 当文件在容器层中不存在时：当容器运行时需要读取某个文件，如果容器层中不存在时，则从镜像层查找该文件，然后读取文件内容。
- 文件既存在于镜像层，又存在于容器层：当我们读取的文件既存在于镜像层，又存在于容器层时，将会从容器层读取该文件。

**2. 修改文件或目录**

AUFS 对文件的修改采用的是写时复制的工作机制，这种工作机制可以最大程度节省存储空间。

具体的文件操作机制如下。

- 第一次修改文件：当我们第一次在容器中修改某个文件时，AUFS 会触发写时复制操作，AUFS 首先从镜像层复制文件到容器层，然后再执行对应的修改操作。

>  AUFS 写时复制的操作将会复制整个文件，如果文件过大，将会大大降低文件系统的性能，因此当我们有大量文件需要被修改时，AUFS 可能会出现明显的延迟。好在，写时复制操作只在第一次修改文件时触发，对日常使用没有太大影响。

- 删除文件或目录：当文件或目录被删除时，AUFS 并不会真正从镜像中删除它，因为镜像层是只读的，AUFS 会创建一个特殊的文件或文件夹，这种特殊的文件或文件夹会阻止容器的访问。



### AUFS 演示



#### 准备演示目录和文件

首先我们在 /tmp 目录下创建 aufs 目录：

```sh
$ cd /tmp
/tmp$ mkdir aufs
```

准备挂载点目录：

```sh
/tmp$ cd aufs
/tmp/aufs$ mkdir mnt
```

接下来准备容器层内容：

```sh
## 创建镜像层目录
/tmp/aufs$ mkdir container1
## 在镜像层目录下准备一个文件
/tmp/aufs$ echo Hello, Container layer! > container1/container1.txt
```

最后准备镜像层内容：

```sh
## 创建两个镜像层目录
/tmp/aufs$ mkdir image1 && mkdir image2
## 分别写入数据
/tmp/aufs$ echo Hello, Image layer1! > image1/image1.txt
/tmp/aufs$ echo Hello, Image layer2! > image2/image2.txt
```

准备好的目录和文件结构如下：

```sh
/tmp/aufs$ tree .
.
|-- container1
|   `-- container1.txt
|-- image1
|   `-- image1.txt
|-- image2
|   `-- image2.txt
`-- mnt
4 directories, 3 files
```



#### 创建 AUFS 联合文件系统

使用 mount 命令可以创建 AUFS 类型的文件系统，命令如下：

```sh
/tmp/aufs$ sudo mount -t aufs -o dirs=./container1:./image2:./image1  none ./mnt
```

mount 命令创建 AUFS 类型文件系统时，这里要注意，dirs 参数第一个冒号默认为读写权限，后面的目录均为只读权限，与 Docker 容器使用 AUFS 的模式一致。

执行完上述命令后，mnt 变成了 AUFS 的联合挂载目录，我们可以使用 mount 命令查看一下已经创建的 AUFS 文件系统：

```sh
/tmp/aufs$ mount -t aufs
none on /tmp/aufs/mnt type aufs (rw,relatime,si=4174b83d649ffb7c)
```

我们每创建一个 AUFS 文件系统，AUFS 都会为我们生成一个 ID，这个 ID 在 /sys/fs/aufs/ 会创建对应的目录，在这个 ID 的目录下可以查看文件挂载的权限。

```sh
tmp/aufs$ cat /sys/fs/aufs/si_4174b83d649ffb7c/*
/tmp/aufs/container1=rw
/tmp/aufs/image2=ro
/tmp/aufs/image1=ro
64
65
66
```

可以看到 container1 目录的权限为 rw（代表可读写），image1 和 image2 的权限为 ro（代表只读）。

为了验证 mnt 目录下可以看到 container1、image1 和 image2 目录下的所有内容，我们使用 ls 命令查看一下 mnt 目录：

```sh
/tmp/aufs$ ls -l mnt/
total 12
-rw-rw-r-- 1 ubuntu ubuntu 24 Sep  9 16:55 container1.txt
-rw-rw-r-- 1 ubuntu ubuntu 21 Sep  9 16:59 image1.txt
-rw-rw-r-- 1 ubuntu ubuntu 21 Sep  9 16:59 image2.txt
```

可以看到 mnt 目录下已经出现了我们准备的所有镜像层和容器层的文件。下面让我们来验证一下 AUFS 的写时复制。



#### 验证 AUFS 的写时复制

AUFS 的写时复制是指在容器中，只有需要修改某个文件时，才会把文件从镜像层复制到容器层，下面我们通过修改联合挂载目录 mnt 下的内容来验证下这个过程。

我们使用以下命令修改 mnt 目录下的 image1.txt 文件：

```sh
/tmp/aufs$ echo Hello, Image layer1 changed! > mnt/image1.txt
```

然后我们查看下 image1/image1.txt 文件内容：

```sh
/tmp/aufs$ cat image1/image1.txt
Hello, Image layer1!
```

发现“镜像层”的 image1.txt 文件并未被修改。

然后我们查看一下"容器层"对应的 image1.txt 文件内容：

```sh
/tmp/aufs$ ls -l container1/
total 8
-rw-rw-r-- 1 ubuntu ubuntu 24 Sep  9 16:55 container1.txt
-rw-rw-r-- 1 ubuntu ubuntu 29 Sep  9 17:21 image1.txt

## 查看文件内容
/tmp/aufs$ cat container1/image1.txt
Hello, Image layer1 changed!
```

发现 AUFS 在“容器层”自动创建了 image1.txt 文件，并且内容为我们刚才写入的内容。

至此，我们完成了 AUFS 写时复制的验证。我们在第一次修改镜像内某个文件时，AUFS 会复制这个文件到容器层，然后在容器层对该文件进行修改操作，这就是 AUFS 最典型的特性写时复制。



#### 结语

到此，相信你知道了联合文件系统是一种分层的轻量级文件系统，它可以把多个目录内容联合挂载到同一目录下，从而形成一个单一的文件系统。同时也学会了如何配置 Docker 使用 AUFS ，并且明白了 AUFS 的工作原理。

那么你知道 AUFS 为什么一直没能成功进入 Linux 内核主线吗？ 

> 没进入主线，是linus 认为他的代码写的太烂，后面有更好的取代品





## 15 | 文件存储驱动：Devicemapper 文件系统原理及生产环境的最佳配置



上一课时我带你学习了什么是联合文件系统，以及 AUFS 的工作原理和配置。我们知道 AUFS 并不在 Linux 内核主干中，所以如果你的操作系统是 CentOS，就不推荐使用 AUFS 作为 Docker 的联合文件系统了。

那在 CentOS 系统中，我们怎么实现镜像和容器的分层结构呢？我们通常使用 Devicemapper 作为 Docker 的联合文件系统。



### 什么是 Devicemapper ？

Devicemapper 是 Linux 内核提供的框架，从 Linux 内核 2.6.9 版本开始引入，Devicemapper 与 AUFS 不同，AUFS 是一种文件系统，而**Devicemapper 是一种映射块设备的技术框架**。

Devicemapper 提供了一种将物理块设备映射到虚拟块设备的机制，目前 Linux 下比较流行的 LVM （Logical Volume Manager 是 Linux 下对磁盘分区进行管理的一种机制）和软件磁盘阵列（将多个较小的磁盘整合成为一个较大的磁盘设备用于扩大磁盘存储和提供数据可用性）都是基于 Devicemapper 机制实现的。

那么 Devicemapper 究竟是如何实现的呢？下面我们首先来了解一下它的关键技术



### Devicemapper 的关键技术

Devicemapper 将主要的工作部分分为用户空间和内核空间。

- 用户空间负责配置具体的设备映射策略与相关的内核空间控制逻辑，例如逻辑设备 dm-a 如何与物理设备 sda 相关联，怎么建立逻辑设备和物理设备的映射关系等。

- 内核空间则负责用户空间配置的关联关系实现，例如当 IO 请求到达虚拟设备 dm-a 时，内核空间负责接管 IO 请求，然后处理和过滤这些 IO 请求并转发到具体的物理设备 sda 上。

这个架构类似于 C/S （客户端/服务区）架构的工作模式，客户端负责具体的规则定义和配置下发，服务端根据客户端配置的规则来执行具体的处理任务。



Devicemapper 的工作机制主要围绕三个核心概念。

- 映射设备（mapped device）：即对外提供的逻辑设备，它是由 Devicemapper 模拟的一个虚拟设备，并不是真正存在于宿主机上的物理设备。

- 目标设备（target device）：目标设备是映射设备对应的物理设备或者物理设备的某一个逻辑分段，是真正存在于物理机上的设备。

- 映射表（map table）：映射表记录了映射设备到目标设备的映射关系，它记录了映射设备在目标设备的起始地址、范围和目标设备的类型等变量。

![img](D:\学习资料\笔记\k8s\k8s图\CgqCHl-GyFOAG6TPAACE_8cMjoQ585.png)



Devicemapper 三个核心概念之间的关系如图 1，映射设备通过映射表关联到具体的物理目标设备。事实上，映射设备不仅可以通过映射表关联到物理目标设备，也可以关联到虚拟目标设备，然后虚拟目标设备再通过映射表关联到物理目标设备。

Devicemapper 在内核中通过很多模块化的映射驱动（target driver）插件实现了对真正 IO 请求的拦截、过滤和转发工作，比如 Raid、软件加密、瘦供给（Thin Provisioning）等。其中瘦供给模块是 Docker 使用 Devicemapper 技术框架中非常重要的模块，下面我们来详细了解下瘦供给（Thin Provisioning）。



#### 瘦供给（Thin Provisioning）

瘦供给的意思是动态分配，这跟传统的固定分配不一样。传统的固定分配是无论我们用多少都一次性分配一个较大的空间，这样可能导致空间浪费。而瘦供给是我们需要多少磁盘空间，存储驱动就帮我们分配多少磁盘空间。

这种分配机制就好比我们一群人围着一个大锅吃饭，负责分配食物的人每次都给你一点分量，当你感觉食物不够时再去申请食物，而当你吃饱了就不需要再去申请食物了，从而避免了食物的浪费，节约的食物可以分配给更多需要的人。

那么，你知道 Docker 是如何使用瘦供给来做到像 AUFS 那样分层存储文件的吗？答案就是： Docker 使用了瘦供给的快照（snapshot）技术。

什么是快照（snapshot）技术？这是全球网络存储工业协会 SNIA（StorageNetworking Industry Association）对快照（Snapshot）的定义：

> 关于指定数据集合的一个完全可用拷贝，该拷贝包括相应数据在某个时间点（拷贝开始的时间点）的映像。快照可以是其所表示的数据的一个副本，也可以是数据的一个复制品。

简单来说，快照是数据在某一个时间点的存储状态。快照的主要作用是对数据进行备份，当存储设备发生故障时，可以使用已经备份的快照将数据恢复到某一个时间点，而 Docker 中的数据分层存储也是基于快照实现的。

以上便是实现 Devicemapper 的关键技术，那 Docker 究竟是如何使用 Devicemapper 实现存储数据和镜像分层共享的呢？



### Devicemapper 是如何数据存储的？

当 Docker 使用 Devicemapper 作为文件存储驱动时，Docker 将镜像和容器的文件存储在瘦供给池（thinpool）中，并将这些内容挂载在 /var/lib/docker/devicemapper/ 目录下。

这些目录储存 Docker 的容器和镜像相关数据，目录的数据内容和功能说明如下。

devicemapper 目录（/var/lib/docker/devicemapper/devicemapper/）：存储镜像和容器实际内容，该目录由一个或多个块设备构成。

metadata 目录（/var/lib/docker/devicemapper/metadata/）： 包含 Devicemapper 本身配置的元数据信息, 以 json 的形式配置，这些元数据记录了镜像层和容器层之间的关联信息。

mnt 目录（ /var/lib/docker/devicemapper/mnt/）：是容器的联合挂载点目录，未生成容器时，该目录为空，而容器存在时，该目录下的内容跟容器中一致。



### Devicemapper 如何实现镜像分层与共享？

Devicemapper 使用专用的块设备实现镜像的存储，并且像 AUFS 一样使用了写时复制的技术来保障最大程度节省存储空间，所以 Devicemapper 的镜像分层也是依赖快照来是实现的。

Devicemapper 的每一镜像层都是其下一层的快照，最底层的镜像层是我们的瘦供给池，通过这种方式实现镜像分层有以下优点。

相同的镜像层，仅在磁盘上存储一次。例如，我有 10 个运行中的 busybox 容器，底层都使用了 busybox 镜像，那么 busybox 镜像只需要在磁盘上存储一次即可。

快照是写时复制策略的实现，也就是说，当我们需要对文件进行修改时，文件才会被复制到读写层。

相比对文件系统加锁的机制，Devicemapper 工作在块级别，因此可以实现同时修改和读写层中的多个块设备，比文件系统效率更高。

当我们需要读取数据时，如果数据存在底层快照中，则向底层快照查询数据并读取。当我们需要写数据时，则向瘦供给池动态申请存储空间生成读写层，然后把数据复制到读写层进行修改。Devicemapper 默认每次申请的大小是 64K 或者 64K 的倍数，因此每次新生成的读写层的大小都是 64K 或者 64K 的倍数。

以下是一个运行中的 Ubuntu 容器示意图。

![img](D:\学习资料\笔记\k8s\k8s图\CgqCHl-GyHeAX_zKAABoNW4U26c205.png)

这个 Ubuntu 镜像一共有四层，每一层镜像都是下一层的快照，镜像的最底层是基础设备的快照。当容器运行时，容器是基于镜像的快照。综上，Devicemapper 实现镜像分层的根本原理就是快照。

接下来，我们看下如何配置 Docker 的 Devicemapper 模式。



### 如何在 Docker 中配置 Devicemapper

Docker 的 Devicemapper 模式有两种：第一种是 loop-lvm 模式，该模式主要用来开发和测试使用；第二种是 direct-lvm 模式，该模式推荐在生产环境中使用。

下面我们逐一配置，首先来看下如何配置 loop-lvm 模式。



#### 配置 loop-lvm 模式

1. 使用以下命令停止已经运行的 Docker：

```sh
$ sudo systemctl stop docker
```

2. 编辑 /etc/docker/daemon.json 文件，如果该文件不存在，则创建该文件，并添加以下配置：

```json
{
  "storage-driver": "devicemapper"
}
```

3. 启动 Docker：

```sh
$ sudo systemctl start docker
```

4. 验证 Docker 的文件驱动模式：

```sh
$ docker info
Client:
 Debug Mode: false
Server:
 Containers: 1
  Running: 0
  Paused: 0
  Stopped: 1
 Images: 1
 Server Version: 19.03.12
 Storage Driver: devicemapper
  Pool Name: docker-253:1-423624832-pool
  Pool Blocksize: 65.54kB
  Base Device Size: 10.74GB
  Backing Filesystem: xfs
  Udev Sync Supported: true
  Data file: /dev/loop0
  Metadata file: /dev/loop1
  Data loop file: /var/lib/docker/devicemapper/devicemapper/data
  Metadata loop file: /var/lib/docker/devicemapper/devicemapper/metadata
  Data Space Used: 22.61MB
  Data Space Total: 107.4GB
  Data Space Available: 107.4GB
  Metadata Space Used: 17.37MB
  Metadata Space Total: 2.147GB
  Metadata Space Available: 2.13GB
  Thin Pool Minimum Free Space: 10.74GB
  Deferred Removal Enabled: true
  Deferred Deletion Enabled: true
  Deferred Deleted Device Count: 0
  Library Version: 1.02.164-RHEL7 (2019-08-27)
```

可以看到 **Storage Driver 为 devicemapper**，这表示 Docker 已经被配置为 Devicemapper 模式。

但是这里输出的 Data file 为 /dev/loop0，这表示我们目前在使用的模式为 loop-lvm。但是由于 loop-lvm 性能比较差，因此不推荐在生产环境中使用 loop-lvm 模式。下面我们看下生产环境中应该如何配置 Devicemapper 的 direct-lvm 模式。



#### 配置 direct-lvm 模式

1.使用以下命令停止已经运行的 Docker：

```sh
$ sudo systemctl stop docker
```

2.编辑 /etc/docker/daemon.json 文件，如果该文件不存在，则创建该文件，并添加以下配置：

```json
{
  "storage-driver": "devicemapper",
  "storage-opts": [
    "dm.directlvm_device=/dev/xdf",
    "dm.thinp_percent=95",
    "dm.thinp_metapercent=1",
    "dm.thinp_autoextend_threshold=80",
    "dm.thinp_autoextend_percent=20",
    "dm.directlvm_device_force=false"
  ]
}
```

其中 directlvm_device 指定需要用作 Docker 存储的磁盘路径，Docker 会动态为我们创建对应的存储池。例如这里我想把 /dev/xdf 设备作为我的 Docker 存储盘，directlvm_device 则配置为 /dev/xdf。

3. 启动 Docker：

```sh
$ sudo systemctl start docker
```

4.验证 Docker 的文件驱动模式：

```sh
$ docker info
Client:
 Debug Mode: false
Server:
 Containers: 1
  Running: 0
  Paused: 0
  Stopped: 1
 Images: 1
 Server Version: 19.03.12
 Storage Driver: devicemapper
  Pool Name: docker-thinpool
  Pool Blocksize: 65.54kB
  Base Device Size: 10.74GB
  Backing Filesystem: xfs
  Udev Sync Supported: true
  Data file:
  Metadata file:
  Data loop file: /var/lib/docker/devicemapper/devicemapper/data
  Metadata loop file: /var/lib/docker/devicemapper/devicemapper/metadata
  Data Space Used: 22.61MB
  Data Space Total: 107.4GB
  Data Space Available: 107.4GB
  Metadata Space Used: 17.37MB
  Metadata Space Total: 2.147GB
  Metadata Space Available: 2.13GB
  Thin Pool Minimum Free Space: 10.74GB
  Deferred Removal Enabled: true
  Deferred Deletion Enabled: true
  Deferred Deleted Device Count: 0
  Library Version: 1.02.164-RHEL7 (2019-08-27)
... 省略部分输出
```

当我们看到 Storage Driver 为 devicemapper，并且 Pool Name 为 docker-thinpool 时，这表示 Devicemapper 的 direct-lvm 模式已经配置成功。



### 结语

Devicemapper 使用块设备来存储文件，运行速度会比直接操作文件系统更快，因此很长一段时间内在 Red Hat 或 CentOS 系统中，Devicemapper 一直作为 Docker 默认的联合文件系统驱动，为 Docker 在 Red Hat 或 CentOS 稳定运行提供强有力的保障。

那么你知道使用 Devicemapper 作为 Docker 联合文件系统的一种解方案是哪家公司在推动吗？

> Devicemapper应该主要是RH公司推动的，当时RH内核不支持AUFS，早期，RH公司想将Docker的AUFS合并内核，其后不知具体原因，他们使用自己原有的后端存储技术，其后与Docker公司合作，贡献了这个第二个驱动。另：devicemapper出现的，当时的背景需求是：一个系统上只能连接有限的常用存储设备（e.g 只有一个硬盘），其后此出现了devicemapper技术，以此来解决当时的问题





## 16 | 文件存储驱动：OverlayFS 文件系统原理及生产环境的最佳配置



前面课时我分别介绍了 Docker 常见的联合文件系统解决方案： AUFS 和 Devicemapper。今天我给你介绍一个性能更好的联合文件系统解决方案—— OverlayFS。

OverlayFS 的发展分为两个阶段。2014 年，OverlayFS 第一个版本被合并到 Linux 内核 3.18 版本中，此时的 OverlayFS 在 Docker 中被称为overlay文件驱动。由于第一版的overlay文件系统存在很多弊端（例如运行一段时间后Docker 会报 "too many links problem" 的错误）， Linux 内核在 4.0 版本对overlay做了很多必要的改进，此时的 OverlayFS 被称之为overlay2。

因此，在 Docker 中 OverlayFS 文件驱动被分为了两种，一种是早期的overlay，不推荐在生产环境中使用，另一种是更新和更稳定的overlay2，推荐在生产环境中使用。下面的内容我们主要围绕overlay2展开。



### 使用 overlay2 的先决条件

overlay2虽然很好，但是它的使用是有一定条件限制的。

- 要想使用overlay2，Docker 版本必须高于 17.06.02。

- 如果你的操作系统是 RHEL 或 CentOS，Linux 内核版本必须使用 3.10.0-514 或者更高版本，其他 Linux 发行版的内核版本必须高于 4.0（例如 Ubuntu 或 Debian），你可以使用 `uname -a` 查看当前系统的内核版本。

- overlay2最好搭配 xfs 文件系统使用，并且使用 xfs 作为底层文件系统时，d_type 必须开启，可以使用以下命令验证 d_type 是否开启：

```sh
$ xfs_info /var/lib/docker | grep ftype
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
```

当输出结果中有 ftype=1 时，表示 d_type 已经开启。如果你的输出结果为 ftype=0，则需要重新格式化磁盘目录，命令如下：

```sh
$ sudo mkfs.xfs -f -n ftype=1 /path/to/disk
```

另外，在生产环境中，推荐挂载 /var/lib/docker 目录到单独的磁盘或者磁盘分区，这样可以避免该目录写满影响主机的文件写入，并且把挂载信息写入到 /etc/fstab，防止机器重启后挂载信息丢失。

挂载配置中推荐开启 **pquota**，这样可以防止某个容器写文件溢出导致整个容器目录空间被占满。写入到 /etc/fstab 中的内容如下：

```sh
$UUID /var/lib/docker xfs defaults,pquota 0 0
```

其中 UUID 为 /var/lib/docker 所在磁盘或者分区的 UUID 或者磁盘路径。

如果你的操作系统无法满足上面的任何一个条件，那我推荐你使用 AUFS 或者 Devicemapper 作为你的 Docker 文件系统驱动。

> 通常情况下， overlay2 会比 AUFS 和 Devicemapper 性能更好，而且更加稳定，因为 overlay2 在 inode 优化上更加高效。因此在生产环境中推荐使用 overlay2 作为 Docker 的文件驱动。

下面我通过实例来教你如何初始化 /var/lib/docker 目录，为后面配置 Docker 的overlay2文件驱动做准备。



### 准备 /var/lib/docker 目录

1.使用 lsblk（Linux 查看磁盘和块设备信息命令）命令查看本机磁盘信息：

```sh
$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
vda    253:0    0  500G  0 disk
`-vda1 253:1    0  500G  0 part /
vdb    253:16   0  500G  0 disk
`-vdb1 253:17   0    8G  0 part
```

可以看到，我的机器有两块磁盘，一块是 vda，一块是 vdb。其中 vda 已经被用来挂载系统根目录，这里我想把 /var/lib/docker 挂载到 vdb1 分区上。

2.使用 mkfs 命令格式化磁盘 vdb1：

```sh
$ sudo mkfs.xfs -f -n ftype=1 /dev/vdb1
```

3.将挂载信息写入到 /etc/fstab，保证机器重启挂载目录不丢失：

```shell
$ sudo echo "/dev/vdb1 /var/lib/docker xfs defaults,pquota 0 0" >> /etc/fstab
```

4.使用 mount 命令使得挂载目录生效：

```sh
$ sudo mount -a
```

5.查看挂载信息：

```sh
$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
vda    253:0    0  500G  0 disk
`-vda1 253:1    0  500G  0 part /
vdb    253:16   0  500G  0 disk
`-vdb1 253:17   0    8G  0 part /var/lib/docker
```

可以看到此时 /var/lib/docker 目录已经被挂载到了 vdb1 这个磁盘分区上。我们使用 xfs_info 命令验证下 d_type 是否已经成功开启：

```sh 
$ xfs_info /var/lib/docker | grep ftype
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
```

可以看到输出结果为 ftype=1，证明 d_type 已经被成功开启。

准备好 /var/lib/docker 目录后，我们就可以配置 Docker 的文件驱动为 overlay2，并且启动 Docker 了。



### 如何在 Docker 中配置 overlay2 ？

当你的系统满足上面的条件后，就可以配置你的 Docker 存储驱动为 overlay2 了，具体配置步骤如下。

1.停止已经运行的 Docker：

```sh
$ sudo systemctl stop docker
```

2.备份 /var/lib/docker 目录：

```sh
$ sudo cp -au /var/lib/docker /var/lib/docker.back
```

3.在 /etc/docker 目录下创建 daemon.json 文件，如果该文件已经存在，则修改配置为以下内容：

```json
{
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.size=20G",
    "overlay2.override_kernel_check=true"
  ]
}
```

其中 storage-driver 参数指定使用 overlay2 文件驱动，overlay2.size 参数表示限制每个容器根目录大小为 20G。限制每个容器的磁盘空间大小是通过 xfs 的 pquota 特性实现，overlay2.size 可以根据不同的生产环境来设置这个值的大小。我推荐你在生产环境中开启此参数，防止某个容器写入文件过大，导致整个 Docker 目录空间溢出。

4.启动 Docker：

```sh
$ sudo systemctl start docker
```

5.检查配置是否生效：

```sh
$ docker info
Client:
 Debug Mode: false
Server:
 Containers: 1
  Running: 0
  Paused: 0
  Stopped: 1
 Images: 1
 Server Version: 19.03.12
 Storage Driver: overlay2
  Backing Filesystem: xfs
  Supports d_type: true
  Native Overlay Diff: true
 Logging Driver: json-file
 Cgroup Driver: cgroupfs
 ... 省略部分无用输出
```

可以看到 Storage Driver 已经变为 overlay2，并且 d_type 也是 true。至此，你的 Docker 已经配置完成。下面我们看下 overlay2 是如何工作的。



### overlay2 工作原理



#### overlay2 是如何存储文件的？

overlay2 和 AUFS 类似，它将所有目录称之为层（layer），overlay2 的目录是镜像和容器分层的基础，而把这些层统一展现到同一的目录下的过程称为联合挂载（union mount）。overlay2 把目录的下一层叫作lowerdir，上一层叫作upperdir，联合挂载后的结果叫作merged。

>  overlay2 文件系统最多支持 128 个层数叠加，也就是说你的 Dockerfile 最多只能写 128 行，不过这在日常使用中足够了。

下面我们通过拉取一个 Ubuntu 操作系统的镜像来看下 overlay2 是如何存放镜像文件的。

首先，我们通过以下命令拉取 Ubuntu 镜像：

```sh
$ docker pull ubuntu:16.04
16.04: Pulling from library/ubuntu
8e097b52bfb8: Pull complete
a613a9b4553c: Pull complete
acc000f01536: Pull complete
73eef93b7466: Pull complete
Digest: sha256:3dd44f7ca10f07f86add9d0dc611998a1641f501833692a2651c96defe8db940
Status: Downloaded newer image for ubuntu:16.04
docker.io/library/ubuntu:16.04
```

可以看到镜像一共被分为四层拉取，拉取完镜像后我们查看一下 overlay2 的目录：

```sh
$ sudo ls -l /var/lib/docker/overlay2/
total 0
drwx------. 3 root root      47 Sep 13 08:16 01946de89606800dac8530e3480b32be9d7c66b493a1cdf558df52d7a1476d4a
drwx------. 4 root root      55 Sep 13 08:16 0849daa41598a333101f6a411755907d182a7fcef780c7f048f15d335b774deb
drwx------. 4 root root      72 Sep 13 08:16 94222a2fa3b2405cb00459285dd0d0ba7e6936d9b693ed18fbb0d08b93dc272f
drwx------. 4 root root      72 Sep 13 08:16 9d392cf38f245d37699bdd7672daaaa76a7d702083694fa8be380087bda5e396
brw-------. 1 root root 253, 17 Sep 13 08:14 backingFsBlockDev
drwx------. 2 root root     142 Sep 13 08:16 l
```

可以看到 overlay2 目录下出现了四个镜像层目录和一个l目录，我们首先来查看一下 l 目录的内容：

```sh
$ sudo ls -l /var/lib/docker/overlay2/l
total 0
lrwxrwxrwx. 1 root root 72 Sep 13 08:16 FWGSYEA56RNMS53EUCKEQIKVLQ -> ../9d392cf38f245d37699bdd7672daaaa76a7d702083694fa8be380087bda5e396/diff
lrwxrwxrwx. 1 root root 72 Sep 13 08:16 RNN2FM3YISKADNAZFRONVNWTIS -> ../0849daa41598a333101f6a411755907d182a7fcef780c7f048f15d335b774deb/diff
lrwxrwxrwx. 1 root root 72 Sep 13 08:16 SHAQ5GYA3UZLJJVEGXEZM34KEE -> ../01946de89606800dac8530e3480b32be9d7c66b493a1cdf558df52d7a1476d4a/diff
lrwxrwxrwx. 1 root root 72 Sep 13 08:16 VQSNH735KNX4YK2TCMBAJRFTGT -> ../94222a2fa3b2405cb00459285dd0d0ba7e6936d9b693ed18fbb0d08b93dc272f/diff
```

可以看到l目录是一堆软连接，把一些较短的随机串软连到镜像层的 diff 文件夹下，这样做是为了避免达到mount命令参数的长度限制。
下面我们查看任意一个镜像层下的文件内容：

```sh
$ sudo ls -l /var/lib/docker/overlay2/0849daa41598a333101f6a411755907d182a7fcef780c7f048f15d335b774deb/
total 8
drwxr-xr-x. 3 root root 17 Sep 13 08:16 diff
-rw-r--r--. 1 root root 26 Sep 13 08:16 link
-rw-r--r--. 1 root root 86 Sep 13 08:16 lower
drwx------. 2 root root  6 Sep 13 08:16 work
```

**镜像层的 link 文件内容为该镜像层的短 ID，diff 文件夹为该镜像层的改动内容，lower 文件为该层的所有父层镜像的短 ID。**

我们可以通过`docker image inspect`命令来查看某个镜像的层级关系，例如我想查看刚刚下载的 Ubuntu 镜像之间的层级关系，可以使用以下命令：

```sh
$ docker image inspect ubuntu:16.04
...省略部分输出
"GraphDriver": {
            "Data": {
                "LowerDir": "/var/lib/docker/overlay2/9d392cf38f245d37699bdd7672daaaa76a7d702083694fa8be380087bda5e396/diff:/var/lib/docker/overlay2/94222a2fa3b2405cb00459285dd0d0ba7e6936d9b693ed18fbb0d08b93dc272f/diff:/var/lib/docker/overlay2/01946de89606800dac8530e3480b32be9d7c66b493a1cdf558df52d7a1476d4a/diff",
                "MergedDir": "/var/lib/docker/overlay2/0849daa41598a333101f6a411755907d182a7fcef780c7f048f15d335b774deb/merged",
                "UpperDir": "/var/lib/docker/overlay2/0849daa41598a333101f6a411755907d182a7fcef780c7f048f15d335b774deb/diff",
                "WorkDir": "/var/lib/docker/overlay2/0849daa41598a333101f6a411755907d182a7fcef780c7f048f15d335b774deb/work"
            },
            "Name": "overlay2"
        },
...省略部分输出
```

其中 MergedDir 代表当前镜像层在 overlay2 存储下的目录，LowerDir 代表当前镜像的父层关系，使用冒号分隔，冒号最后代表该镜像的最底层。

下面我们将镜像运行起来成为容器：

```sh
$ docker run --name=ubuntu -d ubuntu:16.04 sleep 3600
```

我们使用docker inspect命令来查看一下容器的工作目录：

```sh
$ docker inspect ubuntu
...省略部分输出
 "GraphDriver": {
            "Data": {
                "LowerDir": "/var/lib/docker/overlay2/4753c2aa5bdb20c97cddd6978ee3b1d07ef149e3cc2bbdbd4d11da60685fe9b2-init/diff:/var/lib/docker/overlay2/0849daa41598a333101f6a411755907d182a7fcef780c7f048f15d335b774deb/diff:/var/lib/docker/overlay2/9d392cf38f245d37699bdd7672daaaa76a7d702083694fa8be380087bda5e396/diff:/var/lib/docker/overlay2/94222a2fa3b2405cb00459285dd0d0ba7e6936d9b693ed18fbb0d08b93dc272f/diff:/var/lib/docker/overlay2/01946de89606800dac8530e3480b32be9d7c66b493a1cdf558df52d7a1476d4a/diff",
                "MergedDir": "/var/lib/docker/overlay2/4753c2aa5bdb20c97cddd6978ee3b1d07ef149e3cc2bbdbd4d11da60685fe9b2/merged",
                "UpperDir": "/var/lib/docker/overlay2/4753c2aa5bdb20c97cddd6978ee3b1d07ef149e3cc2bbdbd4d11da60685fe9b2/diff",
                "WorkDir": "/var/lib/docker/overlay2/4753c2aa5bdb20c97cddd6978ee3b1d07ef149e3cc2bbdbd4d11da60685fe9b2/work"
            },
            "Name": "overlay2"
        },
...省略部分输出
```

MergedDir 后面的内容即为容器层的工作目录，LowerDir 为容器所依赖的镜像层目录。 然后我们查看下 overlay2 目录下的内容：

```sh
$ sudo ls -l /var/lib/docker/overlay2/
total 0
drwx------. 3 root root      47 Sep 13 08:16 01946de89606800dac8530e3480b32be9d7c66b493a1cdf558df52d7a1476d4a
drwx------. 4 root root      72 Sep 13 08:47 0849daa41598a333101f6a411755907d182a7fcef780c7f048f15d335b774deb
drwx------. 5 root root      69 Sep 13 08:47 4753c2aa5bdb20c97cddd6978ee3b1d07ef149e3cc2bbdbd4d11da60685fe9b2
drwx------. 4 root root      72 Sep 13 08:47 4753c2aa5bdb20c97cddd6978ee3b1d07ef149e3cc2bbdbd4d11da60685fe9b2-init
drwx------. 4 root root      72 Sep 13 08:16 94222a2fa3b2405cb00459285dd0d0ba7e6936d9b693ed18fbb0d08b93dc272f
drwx------. 4 root root      72 Sep 13 08:16 9d392cf38f245d37699bdd7672daaaa76a7d702083694fa8be380087bda5e396
brw-------. 1 root root 253, 17 Sep 13 08:14 backingFsBlockDev
drwx------. 2 root root     210 Sep 13 08:47 l
```

可以看到 overlay2 目录下增加了容器层相关的目录，我们再来查看一下容器层下的内容：

```sh
$ sudo ls -l /var/lib/docker/overlay2/4753c2aa5bdb20c97cddd6978ee3b1d07ef149e3cc2bbdbd4d11da60685fe9b2
total 8
drwxr-xr-x. 2 root root   6 Sep 13 08:47 diff
-rw-r--r--. 1 root root  26 Sep 13 08:47 link
-rw-r--r--. 1 root root 144 Sep 13 08:47 lower
drwxr-xr-x. 1 root root   6 Sep 13 08:47 merged
drwx------. 3 root root  18 Sep 13 08:47 work
```

link 和 lower 文件与镜像层的功能一致，link 文件内容为该容器层的短 ID，lower 文件为该层的所有父层镜像的短 ID 。diff 目录为容器的读写层，容器内修改的文件都会在 diff 中出现，merged 目录为分层文件联合挂载后的结果，也是容器内的工作目录。

总体来说，overlay2 是这样储存文件的：overlay2将镜像层和容器层都放在单独的目录，并且有唯一 ID，每一层仅存储发生变化的文件，最终使用联合挂载技术将容器层和镜像层的所有文件统一挂载到容器中，使得容器中看到完整的系统文件。



#### overlay2 如何读取、修改文件？

overlay2 的工作过程中对文件的操作分为读取文件和修改文件。

**读取文件**

容器内进程读取文件分为以下三种情况。

- 文件在容器层中存在：当文件存在于容器层并且不存在于镜像层时，直接从容器层读取文件；

- 当文件在容器层中不存在：当容器中的进程需要读取某个文件时，如果容器层中不存在该文件，则从镜像层查找该文件，然后读取文件内容；

- 文件既存在于镜像层，又存在于容器层：当我们读取的文件既存在于镜像层，又存在于容器层时，将会从容器层读取该文件。



**修改文件或目录**

overlay2 对文件的修改采用的是写时复制的工作机制，这种工作机制可以最大程度节省存储空间。具体的文件操作机制如下。

- 第一次修改文件：当我们第一次在容器中修改某个文件时，overlay2 会触发写时复制操作，overlay2 首先从镜像层复制文件到容器层，然后在容器层执行对应的文件修改操作。

> overlay2 写时复制的操作将会复制整个文件，如果文件过大，将会大大降低文件系统的性能，因此当我们有大量文件需要被修改时，overlay2 可能会出现明显的延迟。好在，写时复制操作只在第一次修改文件时触发，对日常使用没有太大影响。

- 删除文件或目录：当文件或目录被删除时，overlay2 并不会真正从镜像中删除它，因为镜像层是只读的，overlay2 会创建一个特殊的文件或目录，这种特殊的文件或目录会阻止容器的访问。



### 结语

overlay2 目前已经是 Docker 官方推荐的文件系统了，也是目前安装 Docker 时默认的文件系统，因为 overlay2 在生产环境中不仅有着较高的性能，它的稳定性也极其突出。但是 overlay2 的使用还是有一些限制条件的，例如要求 Docker 版本必须高于 17.06.02，内核版本必须高于 4.0 等。因此，在生产环境中，如果你的环境满足使用 overlay2 的条件，请尽量使用 overlay2 作为 Docker 的联合文件系统。

那么你知道除了我介绍的这三种联合文件系统外，Docker 还可以使用哪些联合文件系统吗？ 





## 17 | 原理实践：自己动手使用 Golang 开发 Docker（上）



第一模块，我们从 Docker 基础概念讲到 Docker 的基本操作。第二模块，我们详细剖析了 Docker 的三大关键技术（ Namespace、cgroups 和联合文件系统）的实现原理，并且讲解了 Docker 的网络模型等关键性技术。接下来动手使用 Golang 编写一个 Docker。



### Golang 是什么?

Golang 又称为 Go，是 Google 开源的一种静态编译型语言，Golang 自带内存管理机制，相比于 C 和 C++ 语言，我们不需要关心内存的分配和回收。

Golang 是新一代的互联网编程语言，在 Golang 诞生前，C 或 C++ 作为服务端高性能编程语言，使用 C 或 C++ 开发的业务具有非常高的执行效率，但是编译和开发效率却不尽人意，Java、.NET 等语言的诞生大大提高了软件开发速度，但是运行效率和资源占用却不如 C 和 C++。

这时 Golang 横空出世，由于 Golang 较高的开发效率和执行效率，很快便从众多编程语言中脱颖而出，成为众多互联网公司的新宠儿。滴滴、知乎、阿里等众多大型互联网公司都在大量使用 Golang。 同时，Docker 和 Kubernetes 等众多明星项目也都是使用 Golang 开发的。因此，熟练掌握 Golang 将会为你加分很多。



### Golang 安装

安装信息如下：

- CentOS 7系统

- Golang 版本 1.15.2

首先我们到 [Golang 官网](https://golang.org/)（由于国内无法访问 Golang 官网，推荐到  [Golang 中文网](https://studygolang.com/dl) 下载安装包）下载一个对应操作系统的安装包。

```sh
$ cd /tmp && wget https://studygolang.com/dl/golang/go1.15.2.linux-amd64.tar.gz
```

解压缩安装包：

```sh
$ sudo tar -C /usr/local -xzf go1.15.2.linux-amd64.tar.gz
```

在 `$HOME/.bashrc` 文件末尾添加以下内容，将 Golang 可执行文件目录添加到系统 PATH 中：

```sh
export PATH=$PATH:/usr/local/go/bin
```

将 go 的安装路径添加到系统 PATH 中后，就可以在命令行直接使用 go 命令了。配置好 go 命令后，我们还需要配置 GOPATH 才能正确存放和编译我们的 go 代码。



### 配置 GOPATH

GOPATH 是 Golang 的源码和相关编译文件的存放路径，GOPATH 路径下有三个文件夹 src、pkg 和 bin，它们的用途分别是：

![image-20201112161607494](D:\学习资料\笔记\k8s\k8s图\image-20201112161607494.png)

这里我们开始配置 GOPATH 路径为 /go。首先准备相关的目录：

```sh
$ sudo mkdir /go
$ sudo mkdir /go/src
$ sudo mkdir /go/pkg
$ sudo mkdir /go/bin
```

然后将 GOPATH 添加到 $HOME/.bashrc 文件末尾，并且把 GOPATH 下的 bin 目录也添加到系统的 PATH 中，这样方便程序编译后直接使用。添加的内容如下：

```sh
export GOPATH=/go
export PATH=$PATH:$GOPATH/bin
# 设置 Golang 的代理，方便我们顺利下载依赖包
export GOPROXY="https://goproxy.io,direct"
```

接下来，使用 `source $HOME/.bashrc` 命令生效一下我们的配置，然后我们再使用 `go env` 命令查看一下我们的配置结果：

```sh
$ go env
GO111MODULE=""
GOARCH="amd64"
GOBIN=""
GOCACHE="/root/.cache/go-build"
GOENV="/root/.config/go/env"
GOEXE=""
GOFLAGS=""
GOHOSTARCH="amd64"
GOHOSTOS="linux"
GOINSECURE=""
GOMODCACHE="/go/pkg/mod"
GONOPROXY=""
GONOSUMDB=""
GOOS="linux"
GOPATH="/go"
GOPRIVATE=""
GOPROXY="https://goproxy.io,direct"
GOROOT="/usr/local/go"
GOSUMDB="sum.golang.org"
GOTMPDIR=""
GOTOOLDIR="/usr/local/go/pkg/tool/linux_amd64"
GCCGO="gccgo"
AR="ar"
CC="gcc"
CXX="g++"
CGO_ENABLED="1"
GOMOD=""
CGO_CFLAGS="-g -O2"
CGO_CPPFLAGS=""
CGO_CXXFLAGS="-g -O2"
CGO_FFLAGS="-g -O2"
CGO_LDFLAGS="-g -O2"
PKG_CONFIG="pkg-config"
GOGCCFLAGS="-fPIC -m64 -pthread -fmessage-length=0 -fdebug-prefix-map=/tmp/go-build352828668=/tmp/go-build -gno-record-gcc-switches"
```

从 GOPATH 和 GOPROXY 两个变量的结果，可以看到 GOPATH 和 GOPROXY 均已经生效。到此，我们的 Golang 已经安装完毕。下面，我们就开始真正的 Docker 编写之旅吧。



### 编写 Docker

在开始编写 Docker 之前，我先介绍几个基础知识，如果你对这些基础知识已经很熟悉了，可以直接跳过这块的基础知识。



#### Linux Proc 文件系统

Linux 系统中，/proc 目录是一种“文件系统”，这里我用了引号，其实 /proc 目录并不是一个真正的文件系统。/proc 目录存放于内存中，是一个虚拟的文件系统，该目录存放了当前内核运行状态的一系列特殊的文件，你可以通过这些文件查看当前的进程信息。

下面，我们通过 ls 命令查看一下 /proc 目录下的内容：

```sh
$ sudo ls -l /proc
total 0
dr-xr-xr-x  9 root    root                  0 Sep 19 21:34 1
dr-xr-xr-x  9 root    root                  0 Sep 19 21:34 30097
...省略部分输出
dr-xr-xr-x  9 root    root                  0 Sep 19 21:34 8
dr-xr-xr-x  9 root    root                  0 Sep 19 21:34 9
dr-xr-xr-x  9 root    root                  0 Sep 19 21:34 97
dr-xr-xr-x  2 root    root                  0 Sep 19 22:27 acpi
-r--r--r--  1 root    root                  0 Sep 19 22:27 buddyinfo
dr-xr-xr-x  4 root    root                  0 Sep 19 22:27 bus
-r--r--r--  1 root    root                  0 Sep 19 22:27 cgroups
-r--r--r--  1 root    root                  0 Sep 19 22:27 cmdline
-r--r--r--  1 root    root                  0 Sep 19 22:27 consoles
-r--r--r--  1 root    root                  0 Sep 19 22:27 cpuinfo
-r--r--r--  1 root    root                  0 Sep 19 22:27 crypto
-r--r--r--  1 root    root                  0 Sep 19 22:27 devices
-r--r--r--  1 root    root                  0 Sep 19 21:34 diskstats
-r--r--r--  1 root    root                  0 Sep 19 22:27 dma
dr-xr-xr-x  2 root    root                  0 Sep 19 22:27 driver
-r--r--r--  1 root    root                  0 Sep 19 22:27 execdomains
-r--r--r--  1 root    root                  0 Sep 19 22:27 fb
-r--r--r--  1 root    root                  0 Sep 19 22:27 filesystems
dr-xr-xr-x  5 root    root                  0 Sep 19 22:27 fs
-r--r--r--  1 root    root                  0 Sep 19 22:27 interrupts
-r--r--r--  1 root    root                  0 Sep 19 22:27 iomem
-r--r--r--  1 root    root                  0 Sep 19 22:27 ioports
dr-xr-xr-x 27 root    root                  0 Sep 19 22:27 irq
-r--r--r--  1 root    root                  0 Sep 19 22:27 kallsyms
-r--------  1 root    root    140737486266368 Sep 19 22:27 kcore
-r--r--r--  1 root    root                  0 Sep 19 22:27 key-users
-r--r--r--  1 root    root                  0 Sep 19 22:27 keys
-r--------  1 root    root                  0 Sep 19 22:27 kmsg
-r--------  1 root    root                  0 Sep 19 22:27 kpagecount
-r--------  1 root    root                  0 Sep 19 22:27 kpageflags
-r--r--r--  1 root    root                  0 Sep 19 22:27 loadavg
-r--r--r--  1 root    root                  0 Sep 19 22:27 locks
-r--r--r--  1 root    root                  0 Sep 19 22:27 mdstat
-r--r--r--  1 root    root                  0 Sep 19 22:27 meminfo
-r--r--r--  1 root    root                  0 Sep 19 22:27 misc
-r--r--r--  1 root    root                  0 Sep 19 22:27 modules
lrwxrwxrwx  1 root    root                 11 Sep 19 22:27 mounts -> self/mounts
-rw-r--r--  1 root    root                  0 Sep 19 22:27 mtrr
lrwxrwxrwx  1 root    root                  8 Sep 19 22:27 net -> self/net
-r--r--r--  1 root    root                  0 Sep 19 22:27 pagetypeinfo
-r--r--r--  1 root    root                  0 Sep 19 22:27 partitions
-r--r--r--  1 root    root                  0 Sep 19 22:27 sched_debug
-r--r--r--  1 root    root                  0 Sep 19 22:27 schedstat
dr-xr-xr-x  2 root    root                  0 Sep 19 22:27 scsi
lrwxrwxrwx  1 root    root                  0 Sep 19 21:34 self -> 30097
-r--------  1 root    root                  0 Sep 19 22:27 slabinfo
-r--r--r--  1 root    root                  0 Sep 19 22:27 softirqs
-r--r--r--  1 root    root                  0 Sep 19 21:34 stat
-r--r--r--  1 root    root                  0 Sep 19 21:34 swaps
dr-xr-xr-x  1 root    root                  0 Sep 19 21:34 sys
--w-------  1 root    root                  0 Sep 19 22:27 sysrq-trigger
dr-xr-xr-x  2 root    root                  0 Sep 19 22:27 sysvipc
-r--r--r--  1 root    root                  0 Sep 19 22:27 timer_list
-rw-r--r--  1 root    root                  0 Sep 19 22:27 timer_stats
dr-xr-xr-x  4 root    root                  0 Sep 19 22:27 tty
-r--r--r--  1 root    root                  0 Sep 19 22:27 uptime
-r--r--r--  1 root    root                  0 Sep 19 22:27 version
-r--------  1 root    root                  0 Sep 19 22:27 vmallocinfo
-r--r--r--  1 root    root                  0 Sep 19 22:27 vmstat
-r--r--r--  1 root    root                  0 Sep 19 22:27 zoneinfo
```

可以看到，这个目录下有很多数字，这些数字目录实际上是以进程 ID 命名的。除了这些以进程 ID 命名的目录，还有一些特殊的目录，这里我讲解一下与我们编写 Docker 有关的文件和目录。

- **self 目录**：它是连接到当前正在运行的进程目录，比如我当前的进程 ID 为 30097，则 self 目录实际连接到 /proc/30097 这个目录。
- **/proc/{PID}/exe 文件**：exe 连接到进程执行的命令文件，例如 30097 这个进程的运行命令为 docker，则执行 /proc/30097/exe ps 等同于执行 docker ps。

好了，了解完这些基础知识后，我们就开始行动吧！因为我们的精简版 Docker 是使用 Golang 编写，这里就给我们编写的 Docker 命名为 gocker 吧。



#### 实现 gocker 的 run 命令

通过前面的章节，我们学习了要运行一个容器，必须先有镜像。这里我们首先准备一个 busybox 镜像，以便我们运行 gocker 容器。

```sh
$ mkdir /tmp/busybox && cd /tmp/busybox
$ docker export $(docker create busybox) -o busybox.tar
$ tar -xf busybox.tar
```

以上是我们在 /tmp/busybox 目录，使用 docker export 命令导出的一个 busybox 镜像文件，然后对镜像文件包进行解压，解压后 /tmp/busybox 目录内容如下：

```sh
$ ls -l /tmp/busybox/
total 1472
drwxr-xr-x 2 root      root        12288 Sep  9 02:09 bin
-rw------- 1 root      root      1455104 Sep 19 22:47 busybox.tar
drwxr-xr-x 4 root      root         4096 Sep 19 16:41 dev
drwxr-xr-x 3 root      root         4096 Sep 19 16:41 etc
drwxr-xr-x 2 nfsnobody nfsnobody    4096 Sep  9 02:09 home
drwxr-xr-x 2 root      root         4096 Sep 19 16:41 proc
drwx------ 2 root      root         4096 Sep 19 21:07 root
drwxr-xr-x 2 root      root         4096 Sep 19 16:41 sys
drwxrwxrwt 2 root      root         4096 Sep  9 02:09 tmp
drwxr-xr-x 3 root      root         4096 Sep  9 02:09 usr
drwxr-xr-x 4 root      root         4096 Sep  9 02:09 var
```

准备好镜像文件后，把我为你准备好的 gocker 代码下载下来吧，这里我使用手动下载源码的方式克隆代码：

```sh
$ mkdir -p /go/src/github.com/wilhelmguo
$ cd /go/src/github.com/wilhelmguo && git clone https://github.com/wilhelmguo/gocker.git
$ cd gocker
$ git checkout lesson-17
```

> 我的 GOPATH 在 /go 目录下，如果你的 GOPATH 跟我不一致，请根据 GOPATH 存放和编译源码。本课时的源码存放在 [这里](https://github.com/wilhelmguo/gocker/tree/lesson-17)，你也可以在线阅读。

代码下载完后，我们进入 gocker 的目录，查看下源码文件：

```sh
$ tree .
.
|-- go.mod
|-- go.sum
|-- main.go
|-- README.md
|-- runc
|   `-- run.go
`-- vendor
... 省略 vendor 目录结构
15 directories, 59 files
```

> 本项目使用 go mod 管理包依赖，go mod 是在 golang 1.11 版本加入的新的特性，是用来管理包的依赖的，也是目前官方的包依赖管理工具。如果你想学习更多个 go mod 使用方法，可以参考 [官网](https://golang.org/ref/mod)。

可以看到该源码下有两个主要文件：一个是 main.go 文件，这是 gocker 的主入口函数；另外一个是 run.go ，这个文件是 gocker run 命令的具体实现。

下面我们使用 go install 命令来编译一下我们的 gocker 项目：

```sh
$ go install
```

执行完 go install 后， Golang 会自动帮助我们编译当前项目下的代码，编译后的二进制文件存放在 `$GOPATH/bin` 目录下。由于我们之前在 `$HOME/.bashrc` 文件下把 ​`$GOPATH/bin` 放入了系统 PATH 中，所以此时你可以直接使用 gocker 命令了。

接下来我们使用 gocker 来启动一个容器：

```sh
# gocker run -it -rootfs=/tmp/busybox /bin/sh
2020/09/19 23:46:27 Current path is  /tmp/busybox
2020/09/19 23:46:27 CmdArray is  [/bin/sh]
/ #
```

> 如果出现 pivotRoot error pivot_root invalid argument 的报错，可以先执行 unshare -m 命令，然后使用 `rm -rf /tmp/busybox/.pivot_root` 命令删除临时文件，再次重试即可。

这里我们使用 it 参数指定以命令行交互的模式启动容器，rootfs 指定准备好的镜像目录。执行完上面的命令后 busybox 容器就成功启动了。

这时候，我们使用 ps 命令查看一下当前进程信息：

```sh
/ # /bin/ps -ef
PID   USER     TIME  COMMAND
    1 root      0:00 /bin/sh
    5 root      0:00 /bin/ps -ef
```

此时，容器内的进程已经与主机完全隔离。

我们再查看一下当前目录下的内容：

```sh
/ # pwd
/ # /bin/ls -l
total 1468
drwxr-xr-x    2 root     root         12288 Sep  8 18:09 bin
-rw-------    1 root     root       1455104 Sep 19 14:47 busybox.tar
drwxr-xr-x    4 root     root          4096 Sep 19 08:41 dev
drwxr-xr-x    3 root     root          4096 Sep 19 08:41 etc
drwxr-xr-x    2 nobody   nobody        4096 Sep  8 18:09 home
dr-xr-xr-x  122 root     root             0 Sep 19 15:46 proc
drwx------    2 root     root          4096 Sep 19 13:07 root
drwxr-xr-x    2 root     root          4096 Sep 19 08:41 sys
drwxrwxrwt    2 root     root          4096 Sep  8 18:09 tmp
drwxr-xr-x    3 root     root          4096 Sep  8 18:09 usr
drwxr-xr-x    4 root     root          4096 Sep  8 18:09 var
```

可以看到当前目录已经为根目录，并且根目录下的文件就是我们上面准备的 busybox 镜像文件。
到此，一个完全由我们自己编写的 gocker 已经可以启动容器了。



### 结语

本课时我们讲解了 Golang 是什么, 并且配置好了 Golang 环境，编译了 gocker，也了解了 Linux /proc 文件系统的一些重要功能，最后使用 gocker 成功启动了一个 busybox 容器。

那么你知道，为什么 Docker 会选择使用 Golang 来开发吗？

> 首先正如文中所说开发效率和执行效率选Go，其次无需像java和php或者python那样安装运行环境，只需一个二进制文件就能在Linux系统跑起来





## 18 | 原理实践：自己动手使用 Golang 开发 Docker（下）



上一课时我们安装了 Golang，学习了一些容器必备的基础知识，并且自己动手编译了一个 gocker，实现了 Namespace 的隔离。今天我将带你深入剖析 gocker 的源码和实现原理，并且带你实现 cgroups 的资源限制。



### gocker 源码剖析

打开 gocker 的源码，我们可以看到 gocker 的实现主要有两个 go 文件：一个是 main.go，一个是 run.go。这两个文件起了什么作用呢？

我们首先来看下 main.go 文件：

```sh
$ cat main.go
package main
import (
    "log"
    "os"
    "github.com/urfave/cli/v2"
    "github.com/wilhelmguo/gocker/runc"
)
func main() {
    app := cli.NewApp()
    app.Name = "gocker"
    app.Usage = "gocker 是 golang 编写的精简版 Docker，目的是学习 Docker 的运行原理。"
    app.Commands = []*cli.Command{
        runc.InitCommand,
        runc.RunCommand,
    }
    if err := app.Run(os.Args); err != nil {
        log.Fatal(err)
    }
}
```

main.go 文件中引用了一个第三方工具库 github.com/urfave/cli，该工具库提供了一个编写命令行的工具，可以帮助我们快速构建命令行应用程序，Docker 默认的容器运行时 runC 也引用了该工具库。
main 函数是 gocker 执行的入口文件，main 定义了 gocker 的名称和简单介绍，同时调用了 InitCommand 和 RunCommand 实现了gocker init和gocker run这两个命令的初始化。

下面我们查看一下 run.go 的文件内容，run.go 文件中定义了 InitCommand 和 RunCommand 的详细实现以及容器启动的过程，文件内容如下。

```sh
$ cat runc/run.go
package runc
import (
    "errors"
    "fmt"
    "io/ioutil"
    "log"
    "os"
    "os/exec"
    "path/filepath"
    "strings"
    "syscall"
    "github.com/urfave/cli/v2"
)
var RunCommand = &cli.Command{
    Name: "run",
    Usage: `启动一个隔离的容器
            gocker run -it [command]`,
    Flags: []cli.Flag{
        &cli.BoolFlag{
            Name:  "it",
            Usage: "是否启用命令行交互模式",
        },
        &cli.StringFlag{
            Name:  "rootfs",
            Usage: "容器根目录",
        },
    },
    Action: func(context *cli.Context) error {
        if context.Args().Len() < 1 {
            return errors.New("参数不全，请检查！")
        }
        read, write, err := os.Pipe()
        if err != nil {
            return err
        }
        tty := context.Bool("it")
        rootfs := context.String("rootfs")
        cmd := exec.Command("/proc/self/exe", "init")
        cmd.SysProcAttr = &syscall.SysProcAttr{
            Cloneflags: syscall.CLONE_NEWNS |
                syscall.CLONE_NEWUTS |
                syscall.CLONE_NEWIPC |
                syscall.CLONE_NEWPID |
                syscall.CLONE_NEWNET,
        }
        if tty {
            cmd.Stdin = os.Stdin
            cmd.Stdout = os.Stdout
            cmd.Stderr = os.Stderr
        }
        cmd.ExtraFiles = []*os.File{read}
        cmd.Dir = rootfs
        if err := cmd.Start(); err != nil {
            log.Println("command start error", err)
            return err
        }
        write.WriteString(strings.Join(context.Args().Slice(), " "))
        write.Close()
        cmd.Wait()
        return nil
    },
}
var InitCommand = &cli.Command{
    Name:  "init",
    Usage: "初始化容器进程，请勿直接调用！",
    Action: func(context *cli.Context) error {
        pwd, err := os.Getwd()
        if err != nil {
            log.Printf("Get current path error %v", err)
            return err
        }
        log.Println("Current path is ", pwd)
        cmdArray := readCommandArray()
        if cmdArray == nil || len(cmdArray) == 0 {
            return fmt.Errorf("Command is empty")
        }
        log.Println("CmdArray is ", cmdArray)
        err = pivotRoot(pwd)
        if err != nil {
            log.Printf("pivotRoot error %v", err)
            return err
        }
        //mount proc
        defaultMountFlags := syscall.MS_NOEXEC | syscall.MS_NOSUID | syscall.MS_NODEV
        syscall.Mount("proc", "/proc", "proc", uintptr(defaultMountFlags), "")
        // 配置hostname
        if err := syscall.Sethostname([]byte("lagoudocker")); err != nil {
            fmt.Printf("Error setting hostname - %s\n", err)
            return err
        }
        path, err := exec.LookPath(cmdArray[0])
        if err != nil {
            log.Printf("Exec loop path error %v", err)
            return err
        }
        // export PATH=$PATH:/bin
        if err := syscall.Exec(path, cmdArray[0:], os.Environ()); err != nil {
            log.Println(err.Error())
        }
        return nil
    },
}
func pivotRoot(root string) error {
    // 确保新 root 和老 root 不在同一目录
    // MS_BIND：执行bind挂载，使文件或者子目录树在文件系统内的另一个点上可视。
    // MS_REC： 创建递归绑定挂载，递归更改传播类型
    if err := syscall.Mount(root, root, "bind", syscall.MS_BIND|syscall.MS_REC, ""); err != nil {
        return fmt.Errorf("Mount rootfs to itself error: %v", err)
    }
    // 创建 .pivot_root 文件夹，用于存储 old_root
    pivotDir := filepath.Join(root, ".pivot_root")
    if err := os.Mkdir(pivotDir, 0777); err != nil {
        return err
    }
    // 调用 Golang 封装的 PivotRoot
    if err := syscall.PivotRoot(root, pivotDir); err != nil {
        return fmt.Errorf("pivot_root %v", err)
    }
    // 修改工作目录
    if err := syscall.Chdir("/"); err != nil {
        return fmt.Errorf("chdir / %v", err)
    }
    pivotDir = filepath.Join("/", ".pivot_root")
    // 卸载 .pivot_root
    if err := syscall.Unmount(pivotDir, syscall.MNT_DETACH); err != nil {
        return fmt.Errorf("unmount pivot_root dir %v", err)
    }
    // 删除临时文件夹 .pivot_root
    return os.Remove(pivotDir)
}
func readCommandArray() []string {
    pipe := os.NewFile(uintptr(3), "pipe")
    msg, err := ioutil.ReadAll(pipe)
    if err != nil {
        log.Printf("init read pipe error %v", err)
        return nil
    }
    msgStr := string(msg)
    return strings.Split(msgStr, " ")
}
```

看到这么多代码你是不是有点懵？别担心，我帮你一一解读。

上面文件中有两个比较重要的变量 InitCommand 和 RunCommand，它们的作用如下：

- RunCommand 是当我们执行 gocker run 命令时调用的函数，是实现 gocker run 的入口；

- InitCommand 是当我们执行 gocker run 时自动调用 gocker init 来初始化容器的一些环境。



#### RunCommand （容器启动的入口）

我们先从 RunCommand 来分析：

```go
var RunCommand = &cli.Command{
    // 定义一个启动命令，这里定义的是 run 命令，当执行 gocker run 时会调用该函数
    Name: "run",
    // 使用说明
    Usage: `启动一个隔离的容器
            gocker run -it [command]`,
    // 执行 gocker run 命令可以传递的参数
    Flags: []cli.Flag{
        &cli.BoolFlag{
            Name:  "it",
            Usage: "是否启用命令行交互模式",
        },
        &cli.StringFlag{
            Name:  "rootfs",
            Usage: "容器根目录",
        },
    },
    // gocker run 命令的执行函数
    Action: func(context *cli.Context) error {
        // 校验参数 
        if context.Args().Len() < 1 {
            return errors.New("参数不全，请检查！")
        }
        read, write, err := os.Pipe()
        if err != nil {
            return err
        }
        // 获取传入的参数的值
        tty := context.Bool("it")
        rootfs := context.String("rootfs")
        // 这里执行 /proc/self/exe init 相当于执行 gocker init
        cmd := exec.Command("/proc/self/exe", "init")
        // 定义新创建哪些命名空间
        cmd.SysProcAttr = &syscall.SysProcAttr{
            Cloneflags: syscall.CLONE_NEWNS |
                syscall.CLONE_NEWUTS |
                syscall.CLONE_NEWIPC |
                syscall.CLONE_NEWPID |
                syscall.CLONE_NEWNET,
        }
        // 把容器的标准输出重定向到主机的标准输出
        if tty {
            cmd.Stdin = os.Stdin
            cmd.Stdout = os.Stdout
            cmd.Stderr = os.Stderr
        }
        cmd.ExtraFiles = []*os.File{read}
        cmd.Dir = rootfs
        // 启动容器
        if err := cmd.Start(); err != nil {
            log.Println("command start error", err)
            return err
        }
        write.WriteString(strings.Join(context.Args().Slice(), " "))
        write.Close()
        // 等待容器退出
        cmd.Wait()
        return nil
        }
```

RunCommand 变量实际上是一个 Command 结构体，这个结构体包含了四个变量。

1. Name：定义一个启动命令，这里定义的是 run 命令，当执行 gocker run 时会调用该函数。

2. Usage：gocker run命令的使用说明。

3. Flags：执行gocker run命令可以传递的参数。

4. Action： 该变量是真正的 gocker run 命令的入口， 主要做了以下事情：
   - 校验 gocker run 传递的参数；
   - 构造一个 Pipe，把 gocker 的启动参数写入，方便在 init 进程中获取；
   - 定义 /proc/self/exe init 调用，相当于调用 gocker init ；
   - 创建五种命名空间用于资源隔离，分别为 Mount Namespace、UTS Namespace、IPC Namespace、PID Namespace 和 Net Namespace；
   - 调用 cmd.Start 函数，开始执行容器启动步骤，首先创建出来一个 namespace （上一步定义的五种namespace）隔离的进程，然后调用 /proc/self/exe，也就是调用 gocker init，执行 InitCommand 中定义的容器初始化步骤。

那么 InitCommand 究竟做了什么呢？



#### InitCommand（准备容器环境）

下面我们看下 InitCommand 中的内容：

```go
var InitCommand = &cli.Command{
    Name:  "init",
    Usage: "初始化容器进程，请勿直接调用！",
    Action: func(context *cli.Context) error {
        // 获取当前执行目录
        pwd, err := os.Getwd()
        if err != nil {
            log.Printf("Get current path error %v", err)
            return err
        }
        log.Println("Current path is ", pwd)
        // 获取用户传递的启动参数
        cmdArray := readCommandArray()
        if cmdArray == nil || len(cmdArray) == 0 {
            return fmt.Errorf("Command is empty")
        }
        log.Println("CmdArray is ", cmdArray)
        // pivotRoot 的作用类似于 chroot，可以把我们准备的镜像目录设置为容器的根目录。
        err = pivotRoot(pwd)
        if err != nil {
            log.Printf("pivotRoot error %v", err)
            return err
        }
        // 挂载容器自己的 proc 目录，实现 ps 只能看到容器自己的进程
        defaultMountFlags := syscall.MS_NOEXEC | syscall.MS_NOSUID | syscall.MS_NODEV
        syscall.Mount("proc", "/proc", "proc", uintptr(defaultMountFlags), "")
        // 配置主机名为 lagoudocker
        if err := syscall.Sethostname([]byte("lagoudocker")); err != nil {
            fmt.Printf("Error setting hostname - %s\n", err)
            return err
        }
        path, err := exec.LookPath(cmdArray[0])
        if err != nil {
            log.Printf("Exec loop path error %v", err)
            return err
        }
        // syscall.Exec 相当于 shell 中的 exec 实现，这里用 用户传递的主命令来替换 init 进程，从而实现容器的 1 号进程为用户传递的主进程
        if err := syscall.Exec(path, cmdArray[0:], os.Environ()); err != nil {
            log.Println(err.Error())
        }
        return nil
    },
}
```

通过代码你能看出 InitCommand 都做了哪些容器启动前的准备工作吗？

InitCommand 主要做了以下几件事情：

获取当前运行目录；

1. 从 RunCommand 中获取用户传递的容器启动参数；

2. 修改当前进程运行的根目录为用户传递的 rootfs 目录；

3. 挂载容器自己的 proc 目录，使得容器中执行 ps 命令只能看到自己命名空间下的进程；

4. 设置容器的主机名称为 lagoudocker；

5. 执行 syscall.Exec 实现使用用户传递的启动命令替换当前 init 进程。

这里有两个比较关键的技术点 pivotRoot 和 syscall.Exec。

- pivotRoot：pivotRoot 是一个系统调用，主要功能是改变当前进程的根目录，它可以把当前进程的根目录移动到我们传递的 rootfs 目录下，从而使得我们不仅能够看到指定目录，还可以看到它的子目录信息。

- syscall.Exec：syscall.Exec 是一个系统调用，这个系统调用可以实现执行指定的命令，但是并不创建新的进程，而是在当前的进程空间执行，替换掉正在执行的进程，复用同一个进程号。通过这种机制，才实现了我们在容器中看到的 1 号进程是我们传递的命令，而不是 init 进程。

最后，总结下容器的完整创建流程:

1. 使用以下命令创建容器

```sh
gocker run -it -rootfs=/tmp/busybox /bin/sh
```

2. RunCommand 解析请求的参数（-it -rootfs=/tmp/busybox）和主进程启动命令（/bin/sh）；

3. 创建 namespace 隔离的容器进程；

4. 启动容器进程；

5. 容器内的进程执行 /proc/self/exe 调用自己实现容器的初始化，修改当前进程运行的根目录，挂载 proc 文件系统，修改主机名，最后使用 sh 进程替换当前容器的进程，使得容器的主进程为 sh 进程。

目前我们的容器虽然实现了使用 Namespace 隔离各种资源，但是容器内的进程仍然可以任意地使用主机的 CPU 、内存等资源。而这可能导致主机的资源竞争，下面我们使用cgroups来实现对 CPU 和内存的限制。



### 为 gocker 添加 cgroups 限制

在第 10 讲中，我们手动操作 cgroups 实现了对容器资源的限制，下面我把这部分手动操作转化为代码。



#### 编写资源限制源码

首先我们定义 cgroups 的挂载目录和我们要创建的目录，定义如下：

```go
const gockerCgroupPath = "gocker"
const cgroupsRoot = "/sys/fs/cgroup"
```

然后定义Cgroups结构体，分别定义 CPU 和 Memory 字段，用于存储用户端传递的 CPU 和 Memory 限制值：

```go
type Cgroups struct {
    // 单位 核
    CPU int
    // 单位 兆
    Memory int
}
```

接着定义 Cgroups 对象的一些操作方法，这样方便我们对当前容器的 cgroups 进程操作。方法定义如下。

- Apply：把容器的 pid 写入到对应子系统下的 tasks 文件中，使得 cgroups 限制对容器进程生效。

- Destroy：容器退出时删除对应的 cgroups 文件。

- SetCPULimit：将 CPU 限制值写入到 cpu.cfs_quota_us 文件中。

- SetMemoryLimit：将内存限制值写入 memory.limit_in_bytes 文件中。

```go
func (c *Cgroups) Apply(pid int) error {
    if c.CPU != 0 {
        cpuCgroupPath, err := getCgroupPath("cpu", true)
        if err != nil {
            return err
        }
        err = ioutil.WriteFile(path.Join(cpuCgroupPath, "tasks"), []byte(strconv.Itoa(pid)), 0644)
        if err != nil {
            return fmt.Errorf("set cgroup cpu fail %v", err)
        }
    }
    if c.Memory != 0 {
        memoryCgroupPath, err := getCgroupPath("memory", true)
        if err != nil {
            return err
        }
        err = ioutil.WriteFile(path.Join(memoryCgroupPath, "tasks"), []byte(strconv.Itoa(pid)), 0644)
        if err != nil {
            return fmt.Errorf("set cgroup memory fail %v", err)
        }
    }
    return nil
}
// 释放cgroup
func (c *Cgroups) Destroy() error {
    if c.CPU != 0 {
        cpuCgroupPath, err := getCgroupPath("cpu", false)
        if err != nil {
            return err
        }
        return os.RemoveAll(cpuCgroupPath)
    }
    if c.Memory != 0 {
        memoryCgroupPath, err := getCgroupPath("memory", false)
        if err != nil {
            return err
        }
        return os.RemoveAll(memoryCgroupPath)
    }
    return nil
}
func (c *Cgroups) SetCPULimit(cpu int) error {
    cpuCgroupPath, err := getCgroupPath("cpu", true)
    if err != nil {
        return err
    }
    if err := ioutil.WriteFile(path.Join(cpuCgroupPath, "cpu.cfs_quota_us"), []byte(strconv.Itoa(cpu*100000)), 0644); err != nil {
        return fmt.Errorf("set cpu limit fail %v", err)
    }
    return nil
}
func (c *Cgroups) SetMemoryLimit(memory int) error {
    memoryCgroupPath, err := getCgroupPath("memory", true)
    if err != nil {
        return err
    }
    if err := ioutil.WriteFile(path.Join(memoryCgroupPath, "memory.limit_in_bytes"), []byte(strconv.Itoa(memory*1024*1024)), 0644); err != nil {
        return fmt.Errorf("set memory limit fail %v", err)
    }
    return nil
}
```

最后在 run 命令的 Action 函数中，添加 cgroups 初始化逻辑，将 CPU 和内存的限制值写入到 cgroups 文件中，并且将当前进程的 pid 也写入到 cgroups 的 tasks 文件中，使得 CPU 和内存的限制对于当前容器进程生效。

```go
        cgroup := cgroups.NewCgroups()
        defer cgroup.Destroy()
        cpus := context.Int("cpus")
        if cpus != 0 {
            cgroup.SetCPULimit(cpus)
        }
        m := context.Int("m")
        if m != 0 {
            cgroup.SetMemoryLimit(m)
        }
        cgroup.Apply(cmd.Process.Pid)
```

到此，我们成功实现了一个带有资源限制的 gocker 容器。下面进入 gocker 的目录，并且编译一下 gocker：

```sh
$ cd gocker
$ git checkout lesson-18
$ go install
```

执行完 go install 后， Golang 会自动帮助我们编译当前项目下的代码，编译后的二进制文件存放在 `$GOPATH/bin` 目录下，由于我们之前在 `$HOME/.bashrc` 文件下把 `$GOPATH/bin` 放入了系统 PATH 中，所以此时你可以直接使用 gocker 命令了。



#### 启动带有资源限制的容器

接下来我们使用 gocker 来启动一个带有 CPU 限制的容器：

```sh
# gocker run -it -cpus=1 -rootfs=/tmp/busybox /bin/sh
2020/09/19 23:46:27 Current path is  /tmp/busybox
2020/09/19 23:46:27 CmdArray is  [/bin/sh]
/ #
```

然后我们新打开一个命令行窗口，查看一下 cgroups 相关的文件是否被创建：

```sh
# cd /sys/fs/cgroup/cpu
# ls -l
总用量 0
-rw-r--r--  1 root root 0 9月  19 21:34 cgroup.clone_children
--w--w--w-  1 root root 0 9月  19 21:34 cgroup.event_control
-rw-r--r--  1 root root 0 9月  19 21:34 cgroup.procs
-r--r--r--  1 root root 0 9月  19 21:34 cgroup.sane_behavior
-r--r--r--  1 root root 0 9月  19 21:34 cpuacct.stat
-rw-r--r--  1 root root 0 9月  19 21:34 cpuacct.usage
-r--r--r--  1 root root 0 9月  19 21:34 cpuacct.usage_percpu
-rw-r--r--  1 root root 0 9月  19 21:34 cpu.cfs_period_us
-rw-r--r--  1 root root 0 9月  19 21:34 cpu.cfs_quota_us
-rw-r--r--  1 root root 0 9月  19 21:34 cpu.rt_period_us
-rw-r--r--  1 root root 0 9月  19 21:34 cpu.rt_runtime_us
-rw-r--r--  1 root root 0 9月  19 21:34 cpu.shares
-r--r--r--  1 root root 0 9月  19 21:34 cpu.stat
drwxr-xr-x  2 root root 0 9月  22 20:48 gocker
-rw-r--r--  1 root root 0 9月  19 21:34 notify_on_release
-rw-r--r--  1 root root 0 9月  19 21:34 release_agent
drwxr-xr-x 70 root root 0 9月  22 20:24 system.slice
-rw-r--r--  1 root root 0 9月  19 21:34 tasks
drwxr-xr-x  2 root root 0 9月  19 21:34 user.slice
```

可以看到我们启动容器后， gocker 在 cpu 子系统下，已经成功创建 gocker 目录。然后我们查看一下 gocker 目录下的内容：

```sh
# ls -l gocker/
总用量 0
-rw-r--r-- 1 root root 0 9月  22 20:48 cgroup.clone_children
--w--w--w- 1 root root 0 9月  22 20:48 cgroup.event_control
-rw-r--r-- 1 root root 0 9月  22 20:48 cgroup.procs
-r--r--r-- 1 root root 0 9月  22 20:48 cpuacct.stat
-rw-r--r-- 1 root root 0 9月  22 20:48 cpuacct.usage
-r--r--r-- 1 root root 0 9月  22 20:48 cpuacct.usage_percpu
-rw-r--r-- 1 root root 0 9月  22 20:48 cpu.cfs_period_us
-rw-r--r-- 1 root root 0 9月  22 20:48 cpu.cfs_quota_us
-rw-r--r-- 1 root root 0 9月  22 20:48 cpu.rt_period_us
-rw-r--r-- 1 root root 0 9月  22 20:48 cpu.rt_runtime_us
-rw-r--r-- 1 root root 0 9月  22 20:48 cpu.shares
-r--r--r-- 1 root root 0 9月  22 20:48 cpu.stat
-rw-r--r-- 1 root root 0 9月  22 20:48 notify_on_release
-rw-r--r-- 1 root root 0 9月  22 20:48 tasks
```

可以看到 cgroups 已经帮我们初始化好了 cpu 子系统的文件，然后我们查看一下 cpu.cfs_quota_us 的内容：

```sh
# cat gocker/cpu.cfs_quota_us
100000
```

可以看到我们容器的 CPU资源已经被限制为 1 核。下面我们来验证一下 CPU 限制是否生效。
首先我们在容器窗口使用以下命令制造一个死循环，来提升 cpu 使用率：

```sh
# while true;do echo;done;
```

然后在主机的窗口使用 top 查看一下cpu 使用率：

```sh
top - 20:57:50 up 2 days, 23:23,  2 users,  load average: 1.08, 0.27, 0.14
Tasks: 113 total,   4 running, 109 sleeping,   0 stopped,   0 zombie
%Cpu(s): 23.5 us, 26.9 sy,  0.0 ni, 49.2 id,  0.0 wa,  0.0 hi,  0.3 si,  0.0 st
KiB Mem :  3880512 total,  1573052 free,   408696 used,  1898764 buff/cache
KiB Swap:        0 total,        0 free,        0 used.  3141076 avail Mem
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
30766 root      20   0    1312    260    212 R  99.3  0.0   0:30.90 sh
```

通过 top 的输出可以看到我们的容器 cpu 使用率被限制到了 100% 以内，即 1 个核。

到此，我们的容器不仅有了 Namespace 隔离，同时也有了 cgroups 的资源限制。



### 结语

上一课时和本课时，我们一起安装了 golang，并且使用 golang 实现了一个精简版的 Docker，它具有基本的 namespace 隔离，并且还使用 cgroups 对容器进行了资源限制。

这两个课时的关键技术我帮你总结如下。

1. Linux 的 /proc 目录是一种“文件系统”，它存放于内存中，是一个虚拟的文件系统，/proc 目录存放了当前内核运行状态的一系列特殊的文件，你可以通过这些文件查看当前的进程信息。

2. /proc/self/exe 是一个特殊的连接，执行该文件等同于执行当前程序的二进制文件

3. pivotRoot 是一个系统调用，主要功能是改变当前进程的根目录，它可以把当前进程的根目录移动到我们传递的 rootfs 目录下

4. syscall.Exec 是一个系统调用，这个系统调用可以实现新的进程直接替换正在执行的老的进程，并且复用老进程的 ID。

另外，容器的实现当然离不开 Linux 的 namespace 和 cgroups 这两项关键技术，有了 Linux 的这些关键技术才使得我们的容器可以顺利实现，可以说 Linux 是容器技术的基石。而容器的编写，我们不仅可以使用 Go 语言，也可以使用其他编程语言，甚至只使用 shell 命令也可以实现一个容器。





# 容器编排三剑客





## 19 | 如何使用 Docker Compose 解决开发环境的依赖？



前两个模块，我们从 Docker 的基本操作到 Docker 的实现原理，为你一步一步揭开了 Docker 神秘的面纱。然而目前为止，我们所有的操作都是围绕单个容器进行的，但当我们的业务越来越复杂时，需要多个容器相互配合，甚至需要多个主机组成容器集群才能满足我们的业务需求，这个时候就需要用到容器的编排工具了。因为容器编排工具可以帮助我们批量地创建、调度和管理容器，帮助我们解决规模化容器的部署问题。

从这一课时开始，我将向你介绍 Docker 三种常用的编排工具：Docker Compose、Docker Swarm 和 Kubernetes。了解这些编排工具，可以让你在不同的环境中选择最优的编排框架。

本课时我们先来学习一个在开发时经常用到的编排工具——Docker Compose。合理地使用 Docker Compose 可以极大地帮助我们提升开发效率。那么 Docker Compose 究竟是什么呢？



### Docker Compose 的前世今生

Docker Compose 的前身是 Orchard 公司开发的 Fig，2014 年 Docker 收购了 Orchard 公司，然后将 Fig 重命名为 Docker Compose。现阶段 Docker Compose 是 Docker 官方的单机多容器管理系统，它本质是一个 Python 脚本，它通过解析用户编写的 yaml 文件，调用 Docker API 实现动态的创建和管理多个容器。

要想使用 Docker Compose，需要我们先安装一个 Docker Compose。



### 安装 Docker Compose

Docker Compose 可以安装在 macOS、 Windows 和 Linux 系统中，其中在 macOS 和 Windows 系统下 ，Docker Compose 都是随着 Docker 的安装一起安装好的，这里就不再详细介绍。 下面我重点介绍下如何在 Linux 系统下安装 Docker Compose。



### Linux 系统下安装 Docker Compose

在安装 Docker Compose 之前，请确保你的机器已经正确运行了 Docker，如果你的机器还没有安装 Docker，请参考 [官方网站](https://docs.docker.com/engine/install/) 安装 Docker。

要在 Linux 平台上安装 Docker Compose，我们需要到 Compose 的 Github 页面下载对应版本的安装包。这里我以 1.27.3 版本为例，带你安装一个 Docker Compose。

（1）使用 curl 命令（一种发送 http 请求的命令行工具）下载 Docker Compose 的安装包：

```sh
$ sudo curl -L "https://github.com/docker/compose/releases/download/1.27.3/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

> 如果你想要安装其他版本的 Docker Compose，将 1.27.3 替换为你想要安装的版本即可。

（2）修改 Docker Compose 执行权限：

```sh
$ sudo chmod +x /usr/local/bin/docker-compose
```

（3）检查 Docker Compose 是否安装成功：

```sh
$ docker-compose --version
docker-compose version 1.27.3, build 1110ad01
```

当我们执行完上述命令后，如果 Docker Compose 输出了当前版本号，就表示我们的 Docker Compose 已经安装成功。 Docker Compose 安装成功后，我们就可以很方便地使用它了。

在使用 Docker Compose 之前，我们首先需要先编写 Docker Compose 模板文件，因为 Docker Compose 运行的时候是根据 Docker Compose 模板文件中的定义来运行的。

下面我们首先来学习一下如何编写一个 Docker Compose 模板文件。



#### 编写 Docker Compose 模板文件

在使用 Docker Compose 启动容器时， Docker Compose 会默认使用 docker-compose.yml 文件， docker-compose.yml 文件的格式为 yaml（类似于 json，一种标记语言）。

Docker Compose 模板文件一共有三个版本： v1、v2 和 v3。目前最新的版本为 v3，也是功能最全面的一个版本，下面我主要围绕 v3 版本介绍一下如何编写 Docker Compose 文件。

Docker Compose 文件主要分为三部分： **services（服务）、networks（网络） 和 volumes（数据卷）。**

- services（服务）：服务定义了容器启动的各项配置，就像我们执行docker run命令时传递的容器启动的参数一样，指定了容器应该如何启动，例如容器的启动参数，容器的镜像和环境变量等。

- networks（网络）：网络定义了容器的网络配置，就像我们执行docker network create命令创建网络配置一样。

- volumes（数据卷）：数据卷定义了容器的卷配置，就像我们执行docker volume create命令创建数据卷一样。

一个典型的 Docker Compose 文件结构如下：

```yaml
version: "3"
services:
  nginx:
    ## ... 省略部分配置
networks:
  frontend:
  backend:
volumes:
  db-data:
```

下面我们首先来学习一下如何编写 services 部分的配置。



#### 编写 Service 配置

services 下，首先需要定义服务名称，例如你这个服务是 nginx 服务，你可以定义 service 名称为 nginx，格式如下：

```yaml
version: "3.8"
services:
  nginx: 
```

服务名称定义完毕后，我们需要在服务名称的下一级定义当前服务的各项配置，使得我们的服务可以按照配置正常启动。常用的 16 种 service 配置如下。如果你比较了解，可以直接跳过看 Volume 配置和后续实操即可。

**build：** 用于构建 Docker 镜像，类似于docker build命令，build 可以指定 Dockerfile 文件路径，然后根据 Dockerfile 命令来构建文件。使用方法如下：

```yaml
build:
  ## 构建执行的上下文目录
  context: .
  ## Dockerfile 名称
  dockerfile: Dockerfile-name
```

**cap_add、cap_drop**： 指定容器可以使用到哪些内核能力（capabilities）。使用格式如下：

```sh
cap_add:
  - NET_ADMIN
cap_drop:
  - SYS_ADMIN
```

**command：** 用于覆盖容器默认的启动命令，它和 Dockerfile 中的 CMD 用法类似，也有两种使用方式：

```yaml
command: sleep 3000
```

```yaml
command: ["sleep", "3000"]
```

**container_name：** 用于指定容器启动时容器的名称。使用格式如下：

```yaml
container_name: nginx
```

**depends_on：** 用于指定服务间的依赖关系，这样可以先启动被依赖的服务。例如，我们的服务依赖数据库服务 db，可以指定 depends_on 为 db。使用格式如下：

```yaml
version: "3.8"
services:
  my-web:
    build: .
    depends_on:
      - db
  db:
    image: mysql
```

**devices：** 挂载主机的设备到容器中。使用格式如下：

```yaml
devices:
  - "/dev/sba:/dev/sda"
```

**dns：** 自定义容器中的 dns 配置。

```yaml
dns:
  - 8.8.8.8
  - 114.114.114.114
```

**dns_search：** 配置 dns 的搜索域。

```yaml
dns_search:
  - svc.cluster.com
  - svc1.cluster.com
```

**entrypoint：** 覆盖容器的 entrypoint 命令。

```yaml
entrypoint: sleep 3000
```

或

```yaml
entrypoint: ["sleep", "3000"]
```

**env_file：** 指定容器的环境变量文件，启动时会把该文件中的环境变量值注入容器中。

```yaml
env_file:
  - ./dbs.env
```

env 文件的内容格式如下：

```yaml
KEY_ENV=values
```

**environment**： 指定容器启动时的环境变量。

```yaml
environment:
  - KEY_ENV=values
```

**image**： 指定容器镜像的地址。

```yaml
image: busybox:latest
```

**pid：** 共享主机的进程命名空间，像在主机上直接启动进程一样，可以看到主机的进程信息。

```yaml
pid: "host"
```

**ports：** 暴露端口信息，使用格式为 HOST:CONTAINER，前面填写要映射到主机上的端口，后面填写对应的容器内的端口。

```yaml
ports:
  - "1000"
  - "1000-1005"
  - "8080:8080"
  - "8888-8890:8888-8890"
  - "2222:22"
  - "127.0.0.1:9999:9999"
  - "127.0.0.1:3000-3005:3000-3005"
  - "6789:6789/udp"
```

**networks**： 这是服务要使用的网络名称，对应顶级的 networks 中的配置。

```yaml
services:
  my-service:
    networks:
     - hello-network
     - hello1-network
```

**volumes：** 不仅可以挂载主机数据卷到容器中，也可以直接挂载主机的目录到容器中，使用方式类似于使用docker run启动容器时添加 -v 参数。

```yaml
version: "3"
services:
  db:
    image: mysql:5.6
    volumes:
      - type: volume
        source: /var/lib/mysql
        target: /var/lib/mysql
```

volumes 除了上面介绍的长语法外，还支持短语法的书写方式，例如上面的写法可以精简为：

```yaml
version: "3"
services:
  db:
    image: mysql:5.6
    volumes:
      - /var/lib/mysql:/var/lib/mysql
```



#### 编写 Volume 配置

如果你想在多个容器间共享数据卷，则需要在外部声明数据卷，然后在容器里声明使用数据卷。例如我想在两个服务间共享日志目录，则使用以下配置：

```yaml
version: "3"
services:
  my-service1:
    image: service:v1
    volumes:
      - type: volume
        source: logdata
        target: /var/log/mylog
  my-service2:
    image: service:v2
    volumes:
      - type: volume
        source: logdata
        target: /var/log/mylog
volumes:
  logdata:
```



#### 编写 Network 配置

Docker Compose 文件顶级声明的 networks 允许你创建自定义的网络，类似于**docker network create**命令。

例如你想声明一个自定义 bridge 网络配置，并且在服务中使用它，使用格式如下：

```yaml
version: "3"
services:
  web:
    networks:
      mybridge: 
        ipv4_address: 172.16.1.11
networks:
  mybridge:
    driver: bridge
    ipam: 
      driver: default
      config:
        subnet: 172.16.1.0/24
```

编写完 Docker Compose 模板文件后，需要使用 docker-compose 命令来运行这些文件。下面我们来学习下 docker-compose 都有哪些操作命令。



### Docker Compose 操作命令

我们可以使用`docker-compose -h`命令来查看 docker-compose 的用法，docker-compose 的基本使用格式如下：

```sh
$ docker-compose [-f <arg>...] [options] [--] [COMMAND] [ARGS...]
```

其中 options 是 docker-compose 的参数，支持的参数和功能说明如下：

```sh
  -f, --file FILE             指定 docker-compose 文件，默认为 docker-compose.yml
  -p, --project-name NAME     指定项目名称，默认使用当前目录名称作为项目名称
  --verbose                   输出调试信息
  --log-level LEVEL           日志级别 (DEBUG, INFO, WARNING, ERROR, CRITICAL)
  -v, --version               输出当前版本并退出
  -H, --host HOST             指定要连接的 Docker 地址
  --tls                       启用 TLS 认证
  --tlscacert CA_PATH         TLS CA 证书路径
  --tlscert CLIENT_CERT_PATH  TLS 公钥证书问价
  --tlskey TLS_KEY_PATH       TLS 私钥证书文件
  --tlsverify                 使用 TLS 校验对端
  --skip-hostname-check       不校验主机名
  --project-directory PATH    指定工作目录，默认是 Compose 文件所在路径。
```

COMMAND 为 docker-compose 支持的命令。支持的命令如下：

```sh
  build              构建服务
  config             校验和查看 Compose 文件
  create             创建服务
  down               停止服务，并且删除相关资源
  events             实时监控容器的时间信息
  exec               在一个运行的容器中运行指定命令
  help               获取帮助
  images             列出镜像
  kill               杀死容器
  logs               查看容器输出
  pause              暂停容器
  port               打印容器端口所映射出的公共端口
  ps                 列出项目中的容器列表
  pull               拉取服务中的所有镜像
  push               推送服务中的所有镜像
  restart            重启服务
  rm                 删除项目中已经停止的容器
  run                在指定服务上运行一个命令
  scale              设置服务运行的容器个数
  start              启动服务
  stop               停止服务
  top                限制服务中正在运行中的进程信息
  unpause            恢复暂停的容器
  up                 创建并且启动服务
  version            打印版本信息并退出
```

学习完 Docker Compose 模板的编写和 docker-compose 命令的使用方法，下面我们编写一个 Docker Compose 模板文件，实现一键启动 WordPress 服务（一种博客系统），来搭建一个属于我们自己的博客系统。



### 使用 Docker Compose 管理 WordPress



#### 启动 WordPress

第一步，创建项目目录。首先我们在 /tmp 目录下创建一个 WordPress 的目录，这个目录将作为我们的工作目录。

```sh
$ mkdir /tmp/wordpress
```

第二步，进入工作目录。

```sh
$ cd /tmp/wordpress
```

第三步，创建 docker-compose.yml 文件。

```sh
$ touch docker-compose.yml
```

然后写入以下内容：

```yaml
version: '3'
services:
   mysql:
     image: mysql:5.7
     volumes:
       - mysql_data:/var/lib/mysql
     restart: always
     environment:
       MYSQL_ROOT_PASSWORD: root
       MYSQL_DATABASE: mywordpress
       MYSQL_USER: mywordpress
       MYSQL_PASSWORD: mywordpress
   wordpress:
     depends_on:
       - mysql
     image: wordpress:php7.4
     ports:
       - "8080:80"
     restart: always
     environment:
       WORDPRESS_DB_HOST: mysql:3306
       WORDPRESS_DB_USER: mywordpress
       WORDPRESS_DB_PASSWORD: mywordpress
       WORDPRESS_DB_NAME: mywordpress
volumes:
    mysql_data: {}
```

第四步，启动 MySQL 数据库和 WordPress 服务。

```sh
$ docker-compose up -d
Starting wordpress_mysql_1 ... done
Starting wordpress_wordpress_1 ... done
```

执行完以上命令后，Docker Compose 首先会为我们启动一个 MySQL 数据库，按照 MySQL 服务中声明的环境变量来设置 MySQL 数据库的用户名和密码。然后等待 MySQL 数据库启动后，再启动 WordPress 服务。WordPress 服务启动后，我们就可以通过 http://localhost:8080 访问它了，访问成功后，我们就可以看到以下界面，然后按照提示一步一步设置就可以拥有属于自己的专属博客系统了。

![img](D:\学习资料\笔记\k8s\k8s图\Ciqc1F-WuHqAMc6gAAGSco9Zyvc339.png)



#### 停止 WordPress

如果你不再需要 WordPress 服务了，可以使用docker-compose stop命令来停止已启动的服务。

```sh
$ docker-compose stop
Stopping wordpress_wordpress_1 ... done
Stopping wordpress_mysql_1     ... done
```



### 结语

Docker Compose 是一个用来定义复杂应用的单机编排工具，通常用于服务依赖关系复杂的开发和测试环境，如果你还在为配置复杂的开发环境而烦恼，Docker Compose 可以轻松帮你搞定复杂的开发环境。你只需要把复杂的开发环境使用 Docker Compose 模板文件描述出来，之后无论你在哪里可以轻松的一键启动开发和测试环境，极大地提高了我们的开发效率，同时也避免了污染我们开发机器的配置。





## 20 | 如何在生产环境中使用 Docker Swarm 调度容器？



Docker 的单节点引擎工具 Docker Compose，它能够在单一节点上管理和编排多个容器，当我们的服务和容器数量较小时可以使用 Docker Compose 来管理容器。

然而随着我们的业务规模越来越大，我们的容器规模也逐渐增大时，数量庞大的容器管理将给我们带来许多挑战。Docker 官方为了解决多容器管理的问题推出了 Docker Swarm ，我们可以用它来管理规模更大的容器集群。



### Swarm 的前生今世

2014 年 Docker 在容器界越来越火，这时容器的编排工具 Mesos 和 Kubernetes 也开始崭露头角。此时，Docker 公司也开始筹划容器的编排和集群管理工具，推出了自己的通信协议项目 Beam。后来，通过改进 Beam，Beam 成为一个允许使用 Docker API 来控制的一种分布式系统，之后项目被重命名为 libswarm。然而在 2014 年 11 月，Docker 公司又对 libswarm 进行了重新设计，支持了远程调用 API，并且被重新命名为 Swarm。到此我们称之为 Swarm V1。

在 2016 年，为了解决中央服务可扩展性的问题，Docker 团队重新设计了 Swarm，并称之为 Swarm V2。此时的 Docker Swarm 已经可以支持超过 1000 多个节点的集群规模，并且 Docker 团队在发布 Docker 1.12 版本时，将 Docker Swarm 默认集成到了 Docker 引擎中。

由于 Swarm 是 Docker 官方推出的容器集群管理工具，因此 Swarm 最大的优势之一就是原生支持 Docker API，给用户带来了极大的便利，原来的 Docker 用户可以很方便地将服务迁移到 Swarm 中来。

与此同时，Swarm 还内置了对 Docker 网络插件的支持，因此用户可以很方便地部署需要跨主机通信的容器集群。其实 Swarm 的优点远远不止这些，还有很多，例如以下优点。

- 分布式： Swarm 使用Raft（一种分布式一致性协议）协议来做集群间数据一致性保障，使用多个容器节点组成管理集群，从而避免单点故障。

- 安全： Swarm 使用 TLS 双向认证来确保节点之间通信的安全，它可以利用双向 TLS 进行节点之间的身份认证，角色授权和加密传输，并且可以自动执行证书的颁发和更换。

- 简单： Swarm 的操作非常简单，并且除 Docker 外基本无其他外部依赖，而且从 Docker 1.12 版本后， Swarm 直接被内置到了 Docker 中，可以说真正做到了开箱即用。

Swarm 的这些优点得益于它优美的架构设计，下面我们来了解一下 Swarm 的架构。



### Swarm 的架构

Swarm 的架构整体分为管理节点（Manager Nodes）和工作节点（Worker Nodes），整体架构如下图：

![img](D:\学习资料\笔记\k8s\k8s图\CgqCHl-iZxSAbYhzAABiA3_fQM8971.png)



管理节点： 管理节点负责接受用户的请求，用户的请求中包含用户定义的容器运行状态描述，然后 Swarm 负责调度和管理容器，并且努力达到用户所期望的状态。

工作节点： 工作节点运行执行器（Executor）负责执行具体的容器管理任务（Task），例如容器的启动、停止、删除等操作。

> 管理节点和工作节点的角色并不是一成不变的，你可以手动将工作节点转换为管理节点，也可以将管理节点转换为工作节点。



### Swarm 核心概念

在真正使用 Swarm 之前，我们需要了解几个 Swarm 的核心概念，这些核心概念可以帮助我们更好地学习和理解 Swarm 的设计理念。



#### Swarm 集群

Swarm 集群是一组被 Swarm 统一管理和调度的节点，被 Swarm纳管的节点可以是物理机或者虚拟机。其中一部分节点作为管理节点，负责集群状态的管理和协调，另一部分作为工作节点，负责执行具体的任务来管理容器，实现用户服务的启停等功能。



#### 节点

Swarm 集群中的每一台物理机或者虚拟机称为节点。节点按照工作职责分为管理节点和工作节点，管理节点由于需要使用 Raft 协议来协商节点状态，生产环境中通常建议将管理节点的数量设置为奇数个，一般为 3 个、5 个或 7 个。



#### 服务

服务是为了支持容器编排所提出的概念，它是一系列复杂容器环境互相协作的统称。一个服务的声明通常包含容器的启动方式、启动的副本数、环境变量、存储、配置、网络等一系列配置，用户通过声明一个服务，将它交给 Swarm，Swarm 负责将用户声明的服务实现。

服务分为全局服务（global services）和副本服务（replicated services）。

- 全局服务：每个工作节点上都会运行一个任务，类似于 Kubernetes 中的 Daemonset。

- 副本服务：按照指定的副本数在整个集群中调度运行。



#### 任务

任务是集群中的最小调度单位，它包含一个真正运行中的 Docker 容器。当管理节点根据服务中声明的副本数将任务调度到节点时，任务则开始在该节点启动和运行，当节点出现异常时，任务会运行失败。此时调度器会把失败的任务重新调度到其他正常的节点上正常运行，以确保运行中的容器副本数满足用户所期望的副本数。



#### 服务外部访问

由于容器的 IP 只能在集群内部访问到，而且容器又是用后马上销毁，这样容器的 IP 也会动态变化，因此容器集群内部的服务想要被集群外部的用户访问到，服务必须要映射到主机上的固定端口。Swarm 使用入口负载均衡（ingress load balancing）的模式将服务暴露在主机上，该模式下，每一个服务会被分配一个公开端口（PublishedPort），你可以指定使用某个未被占用的公开端口，也可以让 Swarm 自动分配一个。

Swarm 集群的公开端口可以从集群内的任意节点上访问到，当请求达到集群中的一个节点时，如果该节点没有要请求的服务，则会将请求转发到实际运行该服务的节点上，从而响应用户的请求。公有云的云负载均衡器（cloud load balancers）可以利用这一特性将流量导入到集群中的一个或多个节点，从而实现利用公有云的云负载均衡器将流量导入到集群中的服务。



### 搭建 Swarm 集群

要想使用 Swarm 集群有如下一些要求：

- Docker 版本大于 1.12，推荐使用最新稳定版 Docker；

- 主机需要开放一些端口（TCP：2377 UDP:4789 TCP 和 UDP:7946）。

下面我通过四台机器来搭建一个 Swarm 集群，演示的节点规划如下：

![img](D:\学习资料\笔记\k8s\k8s图\Ciqc1F-iZ0KAdrQoAABINXCXUv0846.png)

> 生产环境中推荐使用至少三个 manager 作为管理节点。



- 第一步：初始化集群

  Docker 1.12 版本后， Swarm 已经默认集成到了 Docker 中，因此我们可以直接使用 Docker 命令来初始化 Swarm，集群初始化的命令格式如下：

```sh
docker swarm init --advertise-addr <YOUR-IP>
```

> advertise-addr 一般用于主机有多块网卡的情况，如果你的主机只有一块网卡，可以忽略此参数。

在管理节点上，通过以下命令初始化集群：

```sh
$ docker swarm init
Swarm initialized: current node (1ehtnlcf3emncktgjzpoux5ga) is now a manager.
To add a worker to this swarm, run the following command:
    docker swarm join --token SWMTKN-1-1kal5b1iozbfmnnhx3kjfd3y6yqcjjjpcftrlg69pm2g8hw5vx-8j4l0t2is9ok9jwwc3tovtxbp 192.168.31.100:2377
To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

集群初始化后， Swarm 会提示我们当前节点已经作为一个管理节点了，并且提示了如何把一台主机加入集群成为工作节点。



- 第二步：加入工作节点

  按照第一步集群初始化后输出的提示，只需要复制其中的命令即可，然后在剩余的三台工作节点上分别执行如下命令：

```sh
$ docker swarm join --token SWMTKN-1-1kal5b1iozbfmnnhx3kjfd3y6yqcjjjpcftrlg69pm2g8hw5vx-8j4l0t2is9ok9jwwc3tovtxbp 192.168.31.100:2377
This node joined a swarm as a worker.
```

默认加入的节点为工作节点，如果是生产环境，我们可以使用docker swarm join-token manager命令来查看如何加入管理节点：

```sh
$ docker swarm join-to ken manager
To add a manager to this swarm, run the following command:
    docker swarm join --token SWMTKN-1-1kal5b1iozbfmnnhx3kjfd3y6yqcjjjpcftrlg69pm2g8hw5vx-8fq89jxo2axwggryvom5a337t 192.168.31.100:2377
```

复制 Swarm 输出的结果即可加入管理节点到集群中。

> 注意：管理节点的数量必须为奇数，生产环境推荐使用3个、5个或7个管理节点来管理 Swarm 集群。



- 第三步：节点查看

  节点添加完成后，我们使用以下命令可以查看当前节点的状态：

```sh
$ ]# docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
1ehtnlcf3emncktgjzpoux5ga *   swarm-manager       Ready               Active              Leader              19.03.12
pn7gdm847sfzydqhcv3vma97y *   swarm-node1         Ready               Active                                        19.03.12
4dtc9pw5quyjs5yf25ccgr8uh *   swarm-node2         Ready               Active                                        19.03.12
est7ww3gngna4u7td22g9m2k5 *   swarm-node3         Ready               Active                                        19.03.12
```

到此，一个包含 1 个管理节点，3 个工作节点的 Swarm 集群已经搭建完成。



### 使用 Swarm

集群搭建完成后，我们就可以在 Swarm 集群中创建服务了，Swarm 集群中常用的服务部署方式有以下两种。

#### （1）通过 docker service 命令创建服务

使用`docker service create`命令可以创建服务，创建服务的命令如下：

```sh
$ docker service create --replicas 1 --name hello-world nginx
24f9ng83m9sq4ml3e92k4g5by
overall progress: 1 out of 1 tasks
1/1: running   [==================================================>]
verify: Service converged
```

此时我们已经创建好了一个服务，使用docker service ls命令可以查看已经启动的服务：

```sh
$ docker service ls
ID                  NAME                  MODE                REPLICAS            IMAGE               PORTS
24f9ng83m9sq        hello-world           replicated          1/1                 nginx:latest
```

当我们不再需要这个服务了，可以使用docker service rm命令来删除服务：

```sh
$ docker service rm hello-world
hello-world
```

此时 hello-world 这个服务已经成功地从集群中删除。
想要了解更多的docker service命令的相关操作，可以参考这里。

生产环境中，我们推荐使用 docker-compose 模板文件来部署服务，这样服务的管理会更加方便并且可追踪，而且可以同时创建和管理多个服务，更加适合生产环境中依赖关系较复杂的部署模式。



#### （2）通过 docker stack 命令创建服务

我们在 19 课时中创建了 docker-compose 的模板文件，成功的使用该模板文件创建并启动了 MySQL 服务和 WordPress 两个服务。这里我们将 19 讲中的 docker-compose 模板文件略微改造一下：

```yaml
version: '3'
services:
   mysql:
     image: mysql:5.7
     volumes:
       - mysql_data:/var/lib/mysql
     restart: always
     environment:
       MYSQL_ROOT_PASSWORD: root
       MYSQL_DATABASE: mywordpress
       MYSQL_USER: mywordpress
       MYSQL_PASSWORD: mywordpress
   wordpress:
     depends_on:
       - mysql
     image: wordpress:php7.4
     deploy:
       mode: replicated
       replicas: 2
     ports:
       - "8080:80"
     restart: always
     environment:
       WORDPRESS_DB_HOST: mysql:3306
       WORDPRESS_DB_USER: mywordpress
       WORDPRESS_DB_PASSWORD: mywordpress
       WORDPRESS_DB_NAME: mywordpress
volumes:
    mysql_data: {}
```

我在服务模板文件中添加了 deploy 指令，并且指定使用副本服务（replicated）的方式启动两个 WordPress 实例。

准备好启动 WordPress 服务的配置后，我们在 /tmp 目下新建 docker-compose.yml 文件，并且写入以上的内容，然后我们使用以下命令启动服务：

```sh
$ docker stack deploy -c docker-compose.yml wordpress
Ignoring unsupported options: restart
Creating network wordpress_default
Creating service wordpress_mysql
Creating service wordpress_wordpress
```

执行完以上命令后，我们成功启动了两个服务：

1. MySQL 服务，默认启动了一个副本。

2. WordPress 服务，根据我们 docker-compose 模板的定义启动了两个副本。

下面我们用docker service ls命令查看一下当前启动的服务。

```sh
$ docker service ls
ID                  NAME                  MODE                REPLICAS            IMAGE               PORTS
v8i0pzb4e3tc        wordpress_mysql       replicated          1/1                 mysql:5.7
96m8xfyeqzr5        wordpress_wordpress   replicated          2/2                 wordpress:php7.4    *:8080->80/tcp
```

可以看到，Swarm 已经为我们成功启动了一个 MySQL 服务，并且启动了两个 WordPress 实例。WordPress 实例通过 8080 端口暴露在了主机上，我们通过访问集群中的任意节点的 IP 加 8080 端口即可访问到 WordPress 服务。例如，我们访问http://192.168.31.101:8080即可成功访问到我们搭建的 WordPress 服务。



### 结语

Docker Swarm 是一个用来定义复杂应用的集群编排工具，可以帮我们把多台主机组成一个 Swarm 集群，并且帮助我们管理和调度复杂的容器服务。由于 Swarm 已经被内置于 Docker 中，因此 Swarm 的安装和使用也变得非常简单，只要你有 Docker 的使用经验，就可以很快地将你的应用迁移到 Swarm 集群中。





## 21 | 如何使 Docker 和 Kubernetes 结合发挥容器的最大价值？



Docker 虽然在容器领域有着不可撼动的地位，然而在容器的编排领域，却有着另外一个事实标准，那就是 Kubernetes。本课时，我就带你一起来认识下 Kubernetes。



### Kubernetes 的前生今世

说起 Kubernetes，这一切还得从云计算这个词说起，云计算这个概念是 2006 年由 Google 提起的，近些年被提及的频率也越来越高。云计算从起初的概念演变为现在的 AWS、阿里云等实实在在的云产品（主要是虚拟机和相关的网络、存储服务），可见已经变得非常成熟和稳定。

正当大家以为云计算领域已经变成了以虚拟机为代表的云平台时，Docker 在 2013 年横空出世，Docker 提出了镜像、仓库等核心概念，规范了服务的交付标准，使得复杂服务的落地变得更加简单，之后 Docker 又定义了 OCI 标准，可以说在容器领域 Docker 已经成了事实的标准。

然而 Docker 诞生只是帮助我们定义了开发和交付标准，如果想要在生产环境中大批量的使用容器，还离不开的容器的编排技术。于是，在 2014 年 6 月 7 日，Kubernetes（Kubernetes 简称为 K8S，8 代表 ubernete 8个字母） 的第一个 commit（提交）拉开了容器编排标准定义的序幕。

Kubernetes 是舵手的意思，我们把 Docker 比喻成一个个集装箱，而 Kubernetes 正是运输这些集装箱的舵手。早期的 Kubernetes 主要参考 Google 内部的 Borg 系统，Kubernetes 刚刚诞生时，提出了 Pod、Sidecar 等概念，这些都是 Google 内部多年实战和沉淀所积累的精华。经过将近一年的沉淀和积累，Kubernetes 于 2015 年 7 月 21 日对外发布了第一个正式版本 v1.0，正式走入了大众的视线。

很荣幸，我也是在 2015 年下半年正式开始了 Kubernetes 和 Docker 的研发之路。时至今日，Kubernetes 经过 6 年的沉淀，已经成为了事实的编排技术标准。

接下来，我们就看来看看，究竟是什么样的架构使得 Kubernetes 在容器编排领域成为了王者？



### Kubernetes 架构

Kubernetes 采用声明式 API 来工作，所有组件的运行过程都是异步的，整个工作过程大致为用户声明想要的状态，然后 Kubernetes 各个组件相互配合并且努力达到用户想要的状态。

Kubernetes 采用典型的主从架构，分为 Master 和 Node 两个角色。

Mater 是 Kubernetes 集群的控制节点，负责整个集群的管理和控制功能。

Node 为工作节点，负责业务容器的生命周期管理。

整体架构如下图：

![img](D:\学习资料\笔记\k8s\k8s图\Ciqc1F-k_FqAdHbtAAFVTi8cyOE246.png)



#### Master 节点

Master 节点负责对集群中所有容器的调度，各种资源对象的控制，以及响应集群的所有请求。Master 节点包含三个重要的组件： kube-apiserver、kube-scheduler、kube-controller-manager。下面我对这三个组件逐一介绍。

- **kube-apiserver**

kube-apiserver 主要负责提供 Kubernetes 的 API 服务，所有的组件都需要与 kube-apiserver 交互获取或者更新资源信息，它是 Kubernetes Master 中最前端组件。

kube-apiserver 的所有数据都存储在 etcd 中，etcd 是一种采用 Go 语言编写的高可用 Key-Value 数据库，由 CoreOS 开发。etcd 虽然不是 Kubernetes 的组件，但是它在 Kubernetes 中却扮演着至关重要的角色，它是 Kubernetes 的数据大脑。可以说 etcd 的稳定性直接关系着 Kubernetes 集群的稳定性，因此生产环境中 etcd 一定要部署多个实例以确保集群的高可用。

- **kube-scheduler**

kube-scheduler 用于监听未被调度的 Pod，然后根据一定调度策略将 Pod 调度到合适的 Node 节点上运行。

- **kube-controller-manager**

kube-controller-manager 负责维护整个集群的状态和资源的管理。例如多个副本数量的保证，Pod 的滚动更新等。每种资源的控制器都是一个独立协程。kube-controller-manager 实际上是一系列资源控制器的总称。

> 为了保证 Kubernetes 集群的高可用，Master 组件需要部署在多个节点上，由于 Kubernetes 所有数据都存在于 etcd 中，Etcd 是基于 Raft 协议实现，因此生产环境中 Master 通常建议至少三个节点（如果你想要更高的可用性，可以使用 5 个或者 7 个节点）。



#### Node 节点

Node 节点是 Kubernetes 的工作节点，负责运行业务容器。Node 节点主要包含两个组件 ：kubelet 和 kube-proxy。

- **kubelet**

Kubelet 是在每个工作节点运行的代理，它负责管理容器的生命周期。Kubelet 通过监听分配到自己运行的主机上的 Pod 对象，确保这些 Pod 处于运行状态，并且负责定期检查 Pod 的运行状态，将 Pod 的运行状态更新到 Pod 对象中。

- **kube-proxy**

Kube-proxy 是在每个工作节点的网络插件，它实现了 Kubernetes 的 Service 的概念。Kube-proxy 通过维护集群上的网络规则，实现集群内部可以通过负载均衡的方式访问到后端的容器。

Kubernetes 的成功不仅得益于其优秀的架构设计，更加重要的是 Kubernetes 提出了很多核心的概念，这些核心概念构成了容器编排的主要模型。



### Kubernetes 核心概念

Kubernetes 这些概念是 Google 多年的技术沉淀和积累，理解 Kubernetes 的核心概念有助于我们更好的理解 Kubernetes 的设计理念。

**（1）集群**

集群是一组被 Kubernetes 统一管理和调度的节点，被 Kubernetes 纳管的节点可以是物理机或者虚拟机。集群其中一部分节点作为 Master 节点，负责集群状态的管理和协调，另一部分作为 Node 节点，负责执行具体的任务，实现用户服务的启停等功能。

**（2）标签（Label）**
Label 是一组键值对，每一个资源对象都会拥有此字段。Kubernetes 中使用 Label 对资源进行标记，然后根据 Label 对资源进行分类和筛选。

**（3）命名空间（Namespace）**
Kubernetes 中通过命名空间来实现资源的虚拟化隔离，将一组相关联的资源放到同一个命名空间内，避免不同租户的资源发生命名冲突，从逻辑上实现了多租户的资源隔离。

**（4）容器组（Pod）**
Pod 是 Kubernetes 中的最小调度单位，它由一个或多个容器组成，一个 Pod 内的容器共享相同的网络命名空间和存储卷。Pod 是真正的业务进程的载体，在 Pod 运行前，Kubernetes 会先启动一个 Pause 容器开辟一个网络命名空间，完成网络和存储相关资源的初始化，然后再运行业务容器。

**（5）部署（Deployment）**
Deployment 是一组 Pod 的抽象，通过 Deployment 控制器保障用户指定数量的容器副本正常运行，并且实现了滚动更新等高级功能，当我们需要更新业务版本时，Deployment 会按照我们指定策略自动的杀死旧版本的 Pod 并且启动新版本的 Pod。

**（6）状态副本集（StatefulSet）**
StatefulSet 和 Deployment 类似，也是一组 Pod 的抽象，但是 StatefulSet 主要用于有状态应用的管理，StatefulSet 生成的 Pod 名称是固定且有序的，确保每个 Pod 独一无二的身份标识。

**（7）守护进程集（DaemonSet）**
DaemonSet 确保每个 Node 节点上运行一个 Pod，当我们集群有新加入的 Node 节点时，Kubernetes 会自动帮助我们在新的节点上运行一个 Pod。一般用于日志采集，节点监控等场景。

**（8）任务（Job）**
Job 可以帮助我们创建一个 Pod 并且保证 Pod 的正常退出，如果 Pod 运行过程中出现了错误，Job 控制器可以帮助我们创建新的 Pod，直到 Pod 执行成功或者达到指定重试次数。

**（9）服务（Service）**
Service 是一组 Pod 访问配置的抽象。由于 Pod 的地址是动态变化的，我们不能直接通过 Pod 的 IP 去访问某个服务，Service 通过在主机上配置一定的网络规则，帮助我们实现通过一个固定的地址访问一组 Pod。

**（10）配置集（ConfigMap）**
ConfigMap 用于存放我们业务的配置信息，使用 Key-Value 的方式存放于 Kubernetes 中，使用 ConfigMap 可以帮助我们将配置数据和应用程序代码分开。

**（11）加密字典（Secret）**
Secret 用于存放我们业务的敏感配置信息，类似于 ConfigMap，使用 Key-Value 的方式存在于 Kubernetes 中，主要用于存放密码和证书等敏感信息。



### 安装 Kubernetes

Kubernetes 目前已经支持在多种环境下安装，我们可以在公有云，私有云，甚至裸金属中安装 Kubernetes。下面，我们通过 minikube 来演示一下如何快速安装和启动一个 Kubernetes 集群，minikube 是官方提供的一个快速搭建本地 Kubernetes 集群的工具，主要用于本地开发和调试。

下面，我以 Linux 平台为例，演示一下如何使用 minikube 安装一个 Kubernetes 集群。

> 如果你想要在其他平台使用 minikube 安装 Kubernetes，请参考官网 [安装教程](https://minikube.sigs.k8s.io/docs/start/)。
> 在使用 minikube 安装 Kubernetes 之前，请确保我们的机器已经正确安装并且启动 Docker。

第一步，安装 minikube 和 kubectl。首先执行以下命令安装 minikube。

```sh
$ curl -LO https://github.com/kubernetes/minikube/releases/download/v1.13.1/minikube-linux-amd64
$ sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

Kubectl 是 Kubernetes 官方的命令行工具，可以实现对 Kubernetes 集群的管理和控制。
我们使用以下命令来安装 kubectl：

```sh
$ curl -LO https://dl.k8s.io/v1.19.2/kubernetes-client-linux-amd64.tar.gz
$ tar -xvf kubernetes-client-linux-amd64.tar.gz
kubernetes/
kubernetes/client/
kubernetes/client/bin/
kubernetes/client/bin/kubectl
$ sudo install kubernetes/client/bin/kubectl /usr/local/bin/kubectl
```

第二步，安装 Kubernetes 集群。

执行以下命令使用 minikube 安装 Kubernetes 集群：

```sh
$ minikube start
```

执行完上述命令后，minikube 会自动帮助我们创建并启动一个 Kubernetes 集群。命令输出如下，当命令行输出 Done 时，代表集群已经部署完成。

![img](D:\学习资料\笔记\k8s\k8s图\CgqCHl-lL_WABqFRAAE7sPUop9w125.png)

第三步，检查集群状态。集群安装成功后，我们可以使用以下命令检查 Kubernetes 集群是否成功启动。

```sh
$ kubectl cluster-info
Kubernetes master is running at https://172.17.0.3:8443
KubeDNS is running at https://172.17.0.3:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

执行`kubectl cluster-info`命令后，输出 "Kubernetes master is running" 表示我们的集群已经成功运行。



### 创建第一个应用

集群搭建好后，下面我们来试着使用 Kubernetes 来创建我们的第一个应用。

这里我们使用 Deployment 来定义应用的部署信息，使用 Service 暴露我们的应用到集群外部，从而使得我们的应用可以从外部访问到。

第一步，创建 deployment.yaml 文件，并且定义启动的副本数（replicas）为 3。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: hello-world
        image: wilhelmguo/nginx-hello:v1
        ports:
        - containerPort: 80
```

第二步，发布部署文件到 Kubernetes 集群中。

```sh
$ kubectl create -f deployment.yaml
```

部署发布完成后，我们可以使用 kubectl 来查看一下 Pod 是否被成功启动。

```sh
$ kubectl get pod -o wide
NAME                           READY   STATUS    RESTARTS   AGE     IP           NODE       NOMINATED NODE   READINESS GATES
hello-world-57968f9979-xbmzt   1/1     Running   0          3m19s   172.18.0.7   minikube   <none>           <none>
hello-world-57968f9979-xq5w4   1/1     Running   0          3m18s   172.18.0.5   minikube   <none>           <none>
hello-world-57968f9979-zwvgg   1/1     Running   0          4m14s   172.18.0.6   minikube   <none>           <none>
```

这里可以看到 Kubernetes 帮助我们创建了 3 个 Pod 实例。

第三步，创建 service.yaml 文件，帮助我们将服务暴露出去，内容如下：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-world
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: hello-world
```

服务创建完成后，Kubernetes 会随机帮助我们分配一个外部访问端口，可以通过以下命令查看服务信息：

```sh
$ kubectl  get service -o wide
NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE   SELECTOR
hello-world   NodePort    10.101.83.18   <none>        80:32391/TCP   12s   app=hello-world
kubernetes    ClusterIP   10.96.0.1      <none>        443/TCP        40m   <none>
```

由于我们的集群使用 minikube 安装，要想集群中的服务可以通过外部访问，还需要执行以下命令：

```sh
$ minikube service hello-world
```

输出如下：

![img](D:\学习资料\笔记\k8s\k8s图\Ciqc1F-k_seAeN4RAACePALnr0Q662.png)

可以看到 minikube 将我们的服务暴露在了 32391 端口上，我们通过 http://{YOUR-IP}:32391 可以访问到我们启动的服务，如下图所示。

![img](D:\学习资料\笔记\k8s\k8s图\Ciqc1F-k_J-AWWQyAABkHB5NA0A837.png)

总结下，我们首先使用 Deployment 创建了三个 nginx-hello 的实例，然后使用 Service 的方式随机负载到后端的三个实例，并将服务通过 NodePort 的方式暴露在主机上，使得我们可以直接使用主机的端口访问到容器中的服务。



### 结语

Kubernetes 从诞生到现在已经经历了 6 个年头，起初由于它的超前理念被世人误认为设计过度复杂，使得 Kubernetes 的入门门槛非常高。然而 6 年后的今天， Kubernetes 已经拥有了非常完善的社区和工具集，它可以帮助我们一键搭建 Kubernetes 集群，并且围绕 Kubernetes 构建的各种应用也是越来越丰富。

Kubernetes 的目标一直很明确，那就是对标 Borg，可以支撑数亿容器的运行。目前来看，要达到这个目标，Kubernetes 还有很长的路要走，但是当我们谈及云原生，谈及容器云时都必然会提到 Kubernetes，显然它已经成为容器编排的标准和标杆，目前大多数公有云也有支持 Kubernetes。容器的未来一定是美好的，而使用 Kubernetes 来调度容器则更是未来云计算的一个重要风向标。







# 综合实战案例



## 22 | 多阶级构建：Docker 下如何实现镜像多阶级构建？



我们知道 Docker 镜像是分层的，并且每一层镜像都会额外占用存储空间，一个 Docker 镜像层数越多，这个镜像占用的存储空间则会越多。镜像构建最重要的一个原则就是要保持镜像体积尽可能小，要实现这个目标通常可以从两个方面入手：

- 基础镜像体积应该尽量小；
- 尽量减少 Dockerfile 的行数，因为 Dockerfile 的每一条指令都会生成一个镜像层。



在 Docker 的早期版本中，对于编译型语言（例如 C、Java、Go）的镜像构建，我们只能将应用的编译和运行环境的准备，全部都放到一个 Dockerfile 中，这就导致我们构建出来的镜像体积很大，从而增加了镜像的存储和分发成本，这显然与我们的镜像构建原则不符。

为了减小镜像体积，我们需要借助一个额外的脚本，将镜像的编译过程和运行过程分开。

- 编译阶段：负责将我们的代码编译成可执行对象。
- 运行时构建阶段：准备应用程序运行的依赖环境，然后将编译后的可执行对象拷贝到镜像中。



以 Go 语言开发的一个 HTTP 服务为例，代码如下：

```go
package main
import (
   "fmt"
   "net/http"
)
func hello(w http.ResponseWriter, req *http.Request) {
   fmt.Fprintf(w, "hello world!\n")
}
func main() {
   http.HandleFunc("/", hello)
   http.ListenAndServe(":8080", nil)
}
```



将这个 Go 服务构建成镜像分为两个阶段：代码的编译阶段和镜像构建阶段。

构建镜像时，镜像中需要包含 Go 语言编译环境，应用的编译阶段我们可以使用 Dockerfile.build 文件来构建镜像。Dockerfile.build 的内容如下：

```dockerfile
FROM golang:1.13
WORKDIR /go/src/github.com/wilhelmguo/multi-stage-demo/
COPY main.go .
RUN CGO_ENABLED=0 GOOS=linux go build -o http-server .
```



Dockerfile.build 可以帮助我们把代码编译成可以执行的二进制文件，使用以下 Dockerfile 构建一个运行环境：

```dockerfile
WORKDIR /root/
COPY http-server .
CMD ["./http-server"] 
```



然后，我们将应用的编译和运行环境的准备步骤，都放到一个 build.sh 脚本文件中，内容如下：

```sh
#!/bin/sh
echo Building http-server:build
docker build -t http-server:build . -f Dockerfile.build
docker create --name builder http-server:build  
docker cp builder:/go/src/github.com/wilhelmguo/multi-stage-demo/http-server ./http-server  
docker rm -f builder
echo Building http-server:latest
docker build -t http-server:latest .
rm ./http-server
```



下面来逐步分析下这个脚本。

第一步，声明 shell 文件，然后输出开始构建信息。

```sh
#!/bin/sh
echo Building http-server:build
```

第二步，使用 Dockerfile.build 文件来构建一个临时镜像 http-server:build。

```sh
docker build -t http-server:build . -f Dockerfile.build
```

第三步，使用 http-server:build 镜像创建一个名称为 builder 的容器，该容器包含编译后的 http-server 二进制文件。

```sh
docker create --name builder http-server:build  
```

第四步，使用`docker cp`命令从 builder 容器中拷贝 http-server 文件到当前构建目录下，并且删除名称为 builder 的临时容器。

```sh
docker cp builder:/go/src/github.com/wilhelmguo/multi-stage-demo/http-server ./http-server  
docker rm -f builder
```

第五步，输出开始构建镜像信息。

```sh
echo Building http-server:latest
```

第六步，构建运行时镜像，然后删除临时文件 http-server。

```sh
docker build -t http-server:latest .
rm ./http-server
```

这里总结一下，我们是使用 Dockerfile.build 文件来编译应用程序，使用 Dockerfile 文件来构建应用的运行环境。然后我们通过创建一个临时容器，把编译后的 http-server 文件拷贝到当前构建目录中，然后再把这个文件拷贝到运行环境的镜像中，最后指定容器的启动命令为 http-server。

使用这种方式虽然可以实现分离镜像的编译和运行环境，但是我们需要额外引入一个 build.sh 脚本文件，而且构建过程中，还需要创建临时容器 builder 拷贝编译后的 http-server 文件，这使得整个构建过程比较复杂，并且整个构建过程也不够透明。

为了解决这种问题， Docker 在 17.05 推出了多阶段构建（multistage-build）的解决方案。



### 使用多阶段构建

Docker 允许我们在 Dockerfile 中使用多个 FROM 语句，而每个 FROM 语句都可以使用不同基础镜像。最终生成的镜像，是以最后一条 FROM 为准，所以我们可以在一个 Dockerfile 中声明多个 FROM，然后选择性地将一个阶段生成的文件拷贝到另外一个阶段中，从而实现最终的镜像只保留我们需要的环境和文件。多阶段构建的主要使用场景是**分离编译环境和运行环境。**

接下来，我们使用多阶段构建的特性，将上述未使用多阶段构建的过程精简成如下 Dockerfile：

```dockerfile
FROM golang:1.13
WORKDIR /go/src/github.com/wilhelmguo/multi-stage-demo/
COPY main.go .
RUN CGO_ENABLED=0 GOOS=linux go build -o http-server .
FROM alpine:latest  
WORKDIR /root/
COPY --from=0 /go/src/github.com/wilhelmguo/multi-stage-demo/http-server .
CMD ["./http-server"] 
```



然后，我们将这个 Dockerfile 拆解成两步进行分析。

第一步，编译代码。

```dockerfile
FROM golang:1.13
WORKDIR /go/src/github.com/wilhelmguo/multi-stage-demo/
COPY main.go .
RUN CGO_ENABLED=0 GOOS=linux go build -o http-server .
```

将代码拷贝到 golang:1.13 镜像（已经安装好了 go）中，并且使用`go build`命令编译代码生成 http-server 文件。

第二步，构建运行时镜像。

```dockerfile
FROM alpine:latest  
WORKDIR /root/
COPY --from=0 /go/src/github.com/wilhelmguo/multi-stage-demo/http-server .
CMD ["./http-server"]
```

使用第二个 FROM 命令表示镜像构建的第二阶段，使用 COPY 指令拷贝编译后的文件到 alpine 镜像中，--from=0 表示从第一阶段构建结果中拷贝文件到当前构建阶段。

最后，我们只需要使用以下命令，即可实现整个镜像的构建：

```sh
docker build -t http-server:latest .
```

构建出来的镜像与未使用多阶段构建之前构建的镜像大小一致，为了验证这一结论，我们分别使用这两种方式来构建镜像，最后对比一下镜像构建的结果。



### 镜像构建对比

使用多阶段构建前后的代码我都已经放在了[Github](https://github.com/wilhelmguo/multi-stage-demo)，你只需要克隆代码到本地即可。

```sh
$ mkdir /go/src/github.com/wilhelmguo
$ cd /go/src/github.com/wilhelmguo
$ git clone https://github.com/wilhelmguo/multi-stage-demo.git
```

代码克隆完成后，我们首先切换到without-multi-stage分支：

```sh
$ cd without-multi-stage
$ git checkout without-multi-stage
```

这个分支是未使用多阶段构建技术构建镜像的代码，我们可以通过执行 build.sh 文件构建镜像：

```sh
$  chmod +x build.sh && ./build.sh
Building http-server:build
Sending build context to Docker daemon  96.26kB
Step 1/4 : FROM golang:1.13
1.13: Pulling from library/golang
d6ff36c9ec48: Pull complete 
c958d65b3090: Pull complete 
edaf0a6b092f: Pull complete 
80931cf68816: Pull complete 
813643441356: Pull complete 
799f41bb59c9: Pull complete 
16b5038bccc8: Pull complete 
Digest: sha256:8ebb6d5a48deef738381b56b1d4cd33d99a5d608e0d03c5fe8dfa3f68d41a1f8
Status: Downloaded newer image for golang:1.13
 ---> d6f3656320fe
Step 2/4 : WORKDIR /go/src/github.com/wilhelmguo/multi-stage-demo/
 ---> Running in fa3da5ffb0c0
Removing intermediate container fa3da5ffb0c0
 ---> 97245cbb773f
Step 3/4 : COPY main.go .
 ---> a021d2f2a5bb
Step 4/4 : RUN CGO_ENABLED=0 GOOS=linux go build -o http-server .
 ---> Running in b5c36bb67b9c
Removing intermediate container b5c36bb67b9c
 ---> 76c0c88a5cf7
Successfully built 76c0c88a5cf7
Successfully tagged http-server:build
4b0387b270bc4a4da570e1667fe6f9baac765f6b80c68f32007494c6255d9e5b
builder
Building http-server:latest
Sending build context to Docker daemon  7.496MB
Step 1/4 : FROM alpine:latest
latest: Pulling from library/alpine
df20fa9351a1: Already exists 
Digest: sha256:185518070891758909c9f839cf4ca393ee977ac378609f700f60a771a2dfe321
Status: Downloaded newer image for alpine:latest
 ---> a24bb4013296
Step 2/4 : WORKDIR /root/
 ---> Running in 0b25ffe603b8
Removing intermediate container 0b25ffe603b8
 ---> 80da40d3a0b4
Step 3/4 : COPY http-server .
 ---> 3f2300210b7b
Step 4/4 : CMD ["./http-server"]
 ---> Running in 045cea651dde
Removing intermediate container 045cea651dde
 ---> 5c73883177e7
Successfully built 5c73883177e7
Successfully tagged http-server:latest
```



经过一段时间的等待，我们的镜像就构建完成了。
镜像构建完成后，我们使用`docker image ls`命令查看一下刚才构建的镜像大小：

```sh
$ docker image ls http-server
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
http-server         latest              5c73883177e7        3 minutes ago       13MB
http-server         build               76c0c88a5cf7        3 minutes ago       819MB
```

可以看到，http-server:latest 镜像只有 13M，而我们的编译镜像 http-server:build 则为 819M，虽然我们编写了很复杂的脚本 build.sh，但是这个脚本确实帮助我们将镜像体积减小了很多。



下面，我们将代码切换到多阶段构建分支：

```sh
$ git checkout with-multi-stage
Switched to branch 'with-multi-stage'
```

为了避免镜像名称重复，我们将多阶段构建的镜像命名为 http-server-with-multi-stage:latest ，并且禁用缓存，避免缓存干扰构建结果，构建命令如下：

```sh
$ docker build --no-cache -t http-server-with-multi-stage:latest .
Sending build context to Docker daemon  96.77kB
Step 1/8 : FROM golang:1.13
 ---> d6f3656320fe
Step 2/8 : WORKDIR /go/src/github.com/wilhelmguo/multi-stage-demo/
 ---> Running in 640da7a92a62
Removing intermediate container 640da7a92a62
 ---> 9c27b4606da0
Step 3/8 : COPY main.go .
 ---> bd9ce4af24cb
Step 4/8 : RUN CGO_ENABLED=0 GOOS=linux go build -o http-server .
 ---> Running in 6b441b4cc6b7
Removing intermediate container 6b441b4cc6b7
 ---> 759acbf6c9a6
Step 5/8 : FROM alpine:latest
 ---> a24bb4013296
Step 6/8 : WORKDIR /root/
 ---> Running in c2aa2168acd8
Removing intermediate container c2aa2168acd8
 ---> f026884acda6
Step 7/8 : COPY --from=0 /go/src/github.com/wilhelmguo/multi-stage-demo/http-server .
 ---> 667503e6bc14
Step 8/8 : CMD ["./http-server"]
 ---> Running in 15c4cc359144
Removing intermediate container 15c4cc359144
 ---> b73cc4d99088
Successfully built b73cc4d99088
Successfully tagged http-server-with-multi-stage:latest
```

镜像构建完成后，我们同样使用`docker image ls`命令查看一下镜像构建结果：

```sh
$ docker image ls http-server-with-multi-stage:latest
REPOSITORY                     TAG                 IMAGE ID            CREATED             SIZE
http-server-with-multi-stage   latest              b73cc4d99088        2 minutes ago       13MB
```

可以看到，使用多阶段构建的镜像大小与上一步构建的镜像大小一致，都为 13M。但是使用多阶段构建后，却大大减少了我们的构建步骤，使得构建过程更加清晰可读。



### 多阶段构建的其他使用方式

多阶段构建除了我们上面讲解的使用方式，还有更多其他的使用方式，这些使用方式，可以使得多阶段构建实现更多的功能。



#### 为构建阶段命名

默认情况下，每一个构建阶段都没有被命名，你可以通过 FROM 指令出现的顺序来引用这些构建阶段，构建阶段的序号是从 0 开始的。然而，为了提高 Dockerfile 的可读性，我们需要为某些构建阶段起一个名称，这样即便后面我们对 Dockerfile 中的内容进程重新排序或者添加了新的构建阶段，其他构建过程中的 COPY 指令也不需要修改。

上面的 Dockerfile 我们可以优化成如下内容：

```dockerfile
FROM golang:1.13 AS builder
WORKDIR /go/src/github.com/wilhelmguo/multi-stage-demo/
COPY main.go .
RUN CGO_ENABLED=0 GOOS=linux go build -o http-server .
FROM alpine:latest  
WORKDIR /root/
COPY --from=builder /go/src/github.com/wilhelmguo/multi-stage-demo/http-server .
CMD ["./http-server"]
```

我们在第一个构建阶段，使用 AS 指令将这个阶段命名为 builder。然后在第二个构建阶段使用 --from=builder 指令，即可从第一个构建阶段中拷贝文件，使得 Dockerfile 更加清晰可读。



#### 停止在特定的构建阶段

有时候，我们的构建阶段非常复杂，我们想在代码编译阶段进行调试，但是多阶段构建默认构建 Dockerfile 的所有阶段，为了减少每次调试的构建时间，我们可以使用 target 参数来指定构建停止的阶段。

例如，我只想在编译阶段调试 Dockerfile 文件，可以使用如下命令：

```sh
$ docker build --target builder -t http-server:latest .
```

在执行`docker build`命令时添加 target 参数，可以将构建阶段停止在指定阶段，从而方便我们调试代码编译过程。



#### 使用现有镜像作为构建阶段

使用多阶段构建时，不仅可以从 Dockerfile 中已经定义的阶段中拷贝文件，还可以使用`COPY --from`指令从一个指定的镜像中拷贝文件，指定的镜像可以是本地已经存在的镜像，也可以是远程镜像仓库上的镜像。

例如，当我们想要拷贝 nginx 官方镜像的配置文件到我们自己的镜像中时，可以在 Dockerfile 中使用以下指令：

```dockerfile
COPY --from=nginx:latest /etc/nginx/nginx.conf /etc/local/nginx.conf
```

从现有镜像中拷贝文件还有一些其他的使用场景。例如，有些工具没有我们使用的操作系统的安装源，或者安装源太老，需要我们自己下载源码并编译这些工具，但是这些工具可能依赖的编译环境非常复杂，而网上又有别人已经编译好的镜像。这时我们就可以使用`COPY --from`指令从编译好的镜像中将工具拷贝到我们自己的镜像中，很方便地使用这些工具了。



### 结语

多阶段构建可以让我们通过一个 Dockerfile 很方便地构建出体积更小的镜像，并且我们只需要编写 Dockerfile 文件即可，无须借助外部脚本文件。这使得镜像构建过程更加简单透明，但要提醒一点：使用多阶段构建的唯一限制条件是我们使用的 Docker 版本必须高于 17.05 。







## 23 | DevOps：容器化后如何通过 DevOps 提高协作效能？



### DevOps 的前生今世

1964 年，世界上的第一台计算机诞生，那时的计算机主要用于军事领域。计算机的运行离不开程序，那时负责编程的人员被称之为“程序员”。由于那时的程序比较简单，很多工作可以一个人完成，所以早期的计算软件交付流程是这样的：设计—开发—自测—发布—部署—维护。如图 1 所示：

![img](D:\学习资料\笔记\k8s\k8s图\CgqCHl-uVieAfOxkAAAwBofBN_Y124.png)

然而，随着计算机的发展和普及，越来越多的人接触到了计算机，这时的计算机也开始逐渐应用于商业领域，市场上出现了越来越多的办公、游戏等“软件”，也有越来越多的人开始从事软件开发这个行业，而这些软件开发者也有了更加专业的称呼“软件开发工程师”。

后来，又随着计算机软件规模的增大，软件也越来越复杂，这时一个人已经无法完成一个软件完整的生命周期管理。一个软件的开发需要各个团队的分工配合，同时职能划分也需要更加细化，整个软件管理流程中除了软件工程师外又增加了测试工程师和运维工程师。

分工之后软件开发流程如下：研发工程师做代码设计和开发，测试工程师做专业的测试工作，运维工程师负责将软件部署并负责维护软件。如图 2 所示：

![Drawing 3.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F-uVjqANUjJAABEgJx4ezg502.png)

这种软件开发模式被称为瀑布模型，这种模式将软件生命周期划分为制定计划、需求分析、软件设计、程序编写、软件测试和运行维护等六个基本活动，并且规定了它们自上而下、相互衔接的固定次序，如瀑布流水一样，逐级的下降。

![Drawing 5.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F-uVkSAK6G-AABQOfQy504986.png)



瀑布模型的模式十分理想化，它假定用户需求十分明确，开发时间十分充足，且项目是单向迭代的。但随着互联网的出现，软件迭代速度越来越快，软件开发越来越“敏捷”，这时候大名鼎鼎的“敏捷开发”出现了，敏捷开发把大的时间点变成细小的时间点，快速迭代开发，软件更新速度也越来越快。

敏捷开发对传统的开发、测试、运维模式提出了新的挑战，要求更快的开发速度和更高的软件部署频率。而运维工程师信奉的则是稳定性压倒一切，不希望软件频繁变更而引发新的问题。于是乎，敏捷开发和运维工程师之间的矛盾便诞生了。

敏捷开发使得开发和运维工程师之间的矛盾变得越来越深，为了解决这个问题，DevOps 诞生了。DevOps 是研发工程师（Development）和运维工程师（Operations）的组合。下面是维基百科对 DevOps 的定义：

> DevOps（Development 和 Operations 的组合词）是一种重视“软件开发人员（Dev）”和“IT 运维技术人员（Ops）”之间沟通合作的文化、运动或惯例。透过自动化“软件交付”和“架构变更”的流程，来使得构建、测试、发布软件能够更加地快捷、频繁和可靠。

DevOps 的整体目标是**促进开发和运维人员之间的配合，并且通过自动化的手段缩短软件的整个交付周期，提高软件的可靠性。**



### 微服务、容器与 DevOps

软件开发早期，业务模型简单，很多功能都放在一个服务中，这时的服务称之为单体服务，然而随着业务功能越来越复杂，我们发现这种单体服务功能过于复杂，容易牵一发而动全身，导致开发维护成本很高，软件迭代成本也越来越高。

这时，软件开发者开始将单体服务拆分为多个小型服务，每一个小型服务独立负责一项任务，各个小型服务之间通过某种方式（RPC 或者 HTTP）相互调用，然后将不同的服务可以分发给不同的业务团队来开发，各个业务团队可以选择适合自己的编程语言来进行开发。

如果想要微服务实现更快的迭代和更加可靠的稳定性，一定是离不开一个一体化的 DevOps 平台，DevOps 的目标是构建一个稳定可靠的软件生命周期管理环境。所以它不仅可以帮助我们节省很多研发、测试和运维成本，还可以极大地提高我们的软件迭代速度，可以说微服务要想顺利实施，离不开 DevOps 的思想作为指导。

在 Docker 技术出现之前，人们通常更加关注如何做好 CI（Continuous Integration，持续集成）/CD（Continuous Delivery持续交付）以及 IAAS（基础设施即服务），这时我们称之为 DevOps 1.0 时代。

随着 Docker 技术的诞生，我们开始迎来了 DevOps 2.0 时代，DevOps 所有的这些需求都与 Docker 所提供的能力极其匹配。首先 Docker 足够轻量，可以帮助我们的微服务实现快速迭代。其次 Docker 可以很方便地帮助我们构建任何语言的运行环境，帮助我们顺利地使用多种语言来开发的我们的服务，最后 Docker 可以帮助我们更好地隔离开发环境和生产环境。

**可以说 Docker 几乎满足了微服务的所有需求，Docker 为 DevOps 提供了很好的基础支撑。**

这时的研发和运维都开始关注软件统一交付的格式和软件生命周期的管理，**而不像之前一样研发只关注“打包前”，而运维只关注“打包后”的模式**，DevOps 无论是研发环境还是生产环境都开始围绕 Docker 进行构建。

![Drawing 9.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F-uVmOASObQAAA7V7ib-l8145.png)



综上所述，微服务、Docker 与 DevOps 三者之间的关系，如上图所示。

1. 云平台作为底层基础，采用 Docker 技术将服务做容器化部署，并且使用资源管理和调度平台（例如 Kubernetes 或 Swarm）来自动化地管理容器。
2. DevOps 平台在云基础平台之上，通过流程自动化以及工具自动化的手段，为可持续集成和交付提供能力支持。
3. 有了云平台和 DevOps 的支撑，微服务才能够发挥更大的作用，使得我们的业务更加成熟和稳定。



### 容器如何助力 DevOps

Docker 可以在 DevOps 各个阶段发挥重要作用，例如 Docker 可以帮助我们在开发阶段提供统一的开发环境，在持续集成阶段帮助我们快速构建应用，在部署阶段帮助我们快速发布或更新生产环境的应用。

下面我们来详细认识一下 Docker 在整个 DevOps 阶段究竟发挥了哪些作用。



#### 开发流程

开发人员可以在本地或者开发机上快速安装一个 Docker 环境，然后使用 Docker 可以快速启动和部署一个复杂的开发环境。相比传统的配置开发环境的方式，不仅大大提升了开发环境部署的效率，同时也保证了不同开发人员的环境一致。



#### 集成流程

通过编写 Dockerfile 可以将我们的业务容器化，然后将我们的 Dockerfile 提交到代码仓库中，在做持续集成的过程中基于已有的 Dockerfile 来构建应用镜像，可以极大提升持续集成的构建速度。

这主要是因为 Docker 镜像使用了写时复制（Copy On Write）和联合文件系统（Union FileSystem）的机制。Docker 镜像分层存储，相同层仅会保存一份，不同镜像的相同层可以复用，比如 Golang 容器在一次构建停止后，镜像已经存在于构建机上了，当我们开始新一轮的测试时，可以直接复用已有的镜像层，大大提升了构建速度。



#### 部署流程

镜像仓库的存在使得 Docker 镜像分发变得十分简单，当我们的镜像构建完成后，无论在哪里只需要执行 docker pull 命令就可以快速地将镜像拉取到本地并且启动我们的应用，这使得应用的创建或更新更快、更高效。

另外，Docker 结合 Kubernetes 或者其他容器管理平台，可以轻松地实现蓝绿发布等流程，当我们升级应用观察到流量异常时，可以快速回滚到稳定版本。



### DevOps 工具介绍

工欲善其事，必先利其器，要想顺利落地 DevOps，工具的选择十分重要，下面我们来看下除了Docker 外还有哪些工具可以帮助我们顺利地构建 DevOps 平台。



#### Git

![Drawing 10.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F-uVomAACq6AAGSnXiZ7Xg745.png)

[Git](https://git-scm.com/) 是一种分布式的版本控制工具， 是目前使用最广泛的 DevOps 工具之一。Git 相比于其他版本控制工具，它可以**实现离线代码提交**，它允许我们提交代码时未连接到 Git 服务器，等到网络恢复再将我们的代码提交到远程服务器。

Git 非常容易上手，并且占用空间很小，相比于传统的版本控制工具（例如：Subversion、CVS 等）性能非常优秀，它可以帮助我们快速地创建分支，使得团队多人协作开发更加方便。

目前全球最大的在线 Git 代码托管服务是 GitHub，GitHub 提供了代码在线托管服务，可以帮助我们快速地将 DevOps 工作流集成起来。除了 GitHub 外，还有很多在线代码托管服务，例如 GitLab、Bitbucket 等。



#### Jenkins

![Drawing 11.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl-uVpaAF5u_AACv-5xaZ1E856.png)

[Jenkins](https://www.jenkins.io/) 是开源的 CI/CD 构建工具，Jenkins 采用插件化的方式来扩展它的功能，它拥有非常丰富的插件，这些插件可以帮助我们实现构建、部署、自动化等流程。它还拥有强大的生态系统，这些生态系统可以很方便地与 Docker 和 Kubernetes 集成。Jenkins 几乎可以和所有的 DevOps 工具进行集成。



#### Ansible

![Drawing 12.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F-uVqKAHHhCAANJIGhWQ_A950.png)

[Ansible](https://www.ansible.com/) 是一个配置管理工具。Ansible 可以帮助我们自动完成一些重复的 IT 配置管理，应用程序部署等任务，还可以帮助我们放弃编写繁杂的 shell 脚本，仅仅做一些 YAML 的配置，即可实现任务下发和管理工作。并且 Ansible 的每一行命令都是幂等的，它允许我们多次重复执行相同的脚本并且得到的结果都是一致的。



#### Kubernetes

![Drawing 13.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl-uVqmABoq8AAEeX_9ee0Y690.png)

[Kubernetes](https://kubernetes.io/) 是当前最流行的容器编排工具之一，Docker 帮助我们解决了容器打包和镜像分发的问题，而 Kubernetes 则帮助我们解决了大批量容器管理和调度的问题，它可以打通从研发到上线的整个流程，使得 DevOps 落地更加简单方便。



### 结语

DevOps 虽然已经被提及很多年，但是一直没有很好的落地，直到 2013 年 Docker 的诞生，才使得 DevOps 这个理念又重新火了起来，因为 Docker 为我们解决了应用的构建、分发和隔离的问题，才使得 DevOps 落地变得更加简单。

DevOps 提倡小规模和增量的服务发布方式，并且 DevOps 还指导我们尽量避免开发大单体（把所有的功能都集成到一个服务中）应用，这一切，都与 Docker 所能提供的能力十分匹配。因此，Docker 是非常重要的 DevOps 工具。





## 24 | CI/CD：容器化后如何实现持续集成与交付？（上）



### 什么是 CI/CD

#### CI 持续集成（Continuous Integration）

随着软件功能越来越复杂，一个大型项目要想在规定时间内顺利完成，就需要多位开发人员协同开发。但是，如果我们每个人都负责开发自己的代码，然后集中在某一天将代码合并在一起（称为“合并日”）。你会发现，代码可能会有很多冲突和编译问题，而这个处理过程十分烦琐、耗时，并且需要每一位工程师确认代码是否被覆盖，代码是否完整。这种情况显然不是我们想要看到的，这时持续集成（CI）就可以很好地帮助我们解决这个问题。

CI 持续集成要求开发人员频繁地（甚至是每天）将代码提交到共享分支中。一旦开发人员的代码被合并，将会自动触发构建流程来构建应用，并通过触发自动化测试（单元测试或者集成测试）来验证这些代码的提交，确保这些更改没有对应用造成影响。如果发现提交的代码在测试过程中或者构建过程中有问题，则会马上通知研发人员确认，修改代码并重新提交。通过将以往的定期合并代码的方式，改变为频繁提交代码并且自动构建和测试的方式，可以帮助我们**及早地发现问题和解决冲突，减少代码出错。**

传统 CI 流程的实现十分复杂，无法做到标准化交付，而当我们的应用容器化后，应用构建的结果就是 Docker 镜像。代码检查完毕没有缺陷后合并入主分支。此时启动构建流程，构建系统会自动将我们的应用打包成 Docker 镜像，并且推送到镜像仓库。



#### CD 持续交付（Continuous Delivery）

当我们每次完成代码的测试和构建后，我们需要将编译后的镜像快速发布到测试环境，这时我们的持续交付就登场了。持续交付要求我们实现自动化准备测试环境、自动化测试应用、自动化监控代码质量，并且自动化交付生产环境镜像。

在以前，测试环境的构建是非常耗时的，并且很难保证测试环境和研发环境的一致性。但现在，借助于容器技术，我们可以很方便地构建出一个测试环境，并且可以保证开发和测试环境的一致性，这样不仅可以提高测试效率，还可以提高敏捷性。

容器化后的应用交付过程是这样的，我们将测试的环境交由 QA 来维护，当我们确定好本次上线要发布的功能列表时，我们将不同开发人员开发的 feature 分支的代码合并到 release 分支。然后由 QA 来将构建镜像部署到测试环境，结合自动测试和人工测试、自动检测和人工记录，形成完整的测试报告，并且把测试过程中遇到的问题交由开发人员修改，开发修改无误后再次构建测试环境进行测试。测试没有问题后，自动交付生产环境的镜像到镜像仓库。



#### CD 持续部署（Continuous Deployment）

CD 不仅有持续交付的含义，还代表持续部署。经测试无误打包完生产环境的镜像后，我们需要把镜像部署到生产环境，持续部署是最后阶段，它作为持续交付的延伸，可以自动将生产环境的镜像发布到生产环境中。

部署业务首先需要我们有一个资源池，实现资源自动化供给，而且有的应用还希望有自动伸缩的能力，根据外部流量自动调整容器的副本数，而这一切在容器云中都将变得十分简单。

我们可以想象，如果有客户提出了反馈，我们通过持续部署在几分钟内，就能在更改完代码的情况下，将新的应用版本发布到生产环境中（假设通过了自动化测试），这时我们就可以实现快速迭代，持续接收和整合用户的反馈，将用户体验做到极致。

下面我们就动动手，利用一些工具搭建一个 DevOps 环境。

搭建 DevOps 环境的工具非常多，这里我选择的工具为 Jenkins、Docker 和 GitLab。Jenkins 和 Docker 都已经介绍过了，这里我再介绍一下 Gitlab。

Gitlab 是由 Gitlab Inc. 开发的一款基于 Git 的代码托管平台，它的功能和 GitHub 类似，可以帮助我们存储代码。除此之外，GitLab 还具有在线编辑 wiki、issue 跟踪等功能，另外最新版本的 GitLab 还集成了 CI/CD 功能，不过这里我们仅仅使用 GitLab 的代码存储功能， CI/CD 还是交给我们的老牌持续集成工具 Jenkins 来做。



### Docker+Jenkins+GitLab 搭建 CI/CD 系统

软件安装环境如下。

- 操作系统：CentOS 7
- Jenkins：tls 长期维护版
- Docker：18.06
- GitLab：13.3.8-ce.0



#### 第一步：安装 Docker

安装 Docker 的步骤可以在[第一讲](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=455#/detail/pc?id=4572)的内容中找到，这里就不再赘述。Docker 环境准备好后，我们就可以利用 Docker 来部署 GitLab 和 Jenkins 了。



#### 第二步：安装 GitLab

GitLab 官方提供了 GitLab 的 Docker 镜像，因此我们只需要执行以下命令就可以快速启动一个 GitLab 服务了。

```sh
$ docker run -d \
--hostname localhost \
-p 8080:80 -p 2222:22 \
--name gitlab \
--restart always \
--volume /tmp/gitlab/config:/etc/gitlab \
--volume /tmp/gitlab/logs:/var/log/gitlab \
--volume /tmp/gitlab/data:/var/opt/gitlab \
gitlab/gitlab-ce:13.3.8-ce.0
```

这个启动过程可能需要几分钟的时间。当服务启动后我们就可以通过 [http://localhost:8080](http://localhost:8080/) 访问到我们的 GitLab 服务了。

![Drawing 0.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl-05ceAOEtOAAC7xTjpRgo536.png)



第一次登陆，GitLab 会要求我们设置管理员密码，我们输入管理员密码后点击确认即可，之后 GitLab 会自动跳转到登录页面。

![Drawing 1.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl-05eKAftEiAACO_hts6R8497.png)

然后输入默认管理员用户名：admin@example.com，密码为我们上一步设置的密码。点击登录即可登录到系统中，至此，GitLab 已经安装成功。



#### 第三步：安装 Jenkins

Jenkins 官方提供了 Jenkins 的 Docker 镜像，我们使用 Jenkins 镜像就可以一键启动一个 Jenkins 服务。命令如下：

```sh
$ docker run -d --name=jenkins \
-p 8888:8080 \
-u root \
--restart always \
-v /var/run/docker.sock:/var/run/docker.sock \
-v /usr/bin/docker:/usr/bin/docker \
-v /tmp/jenkins_home:/var/jenkins_home \
jenkins/jenkins:lts
```

> 这里，我将 docker.sock 和 docker 二进制挂载到了 Jenkins 容器中，是为了让 Jenkins 可以直接调用 docker 命令来构建应用镜像。



Jenkins 的默认密码会在容器启动后打印在容器的日志中，我们可以通过以下命令找到 Jenkins 的默认密码，星号之间的类似于一串 UUID 的随机串就是我们的密码。

```sh
$ docker logs -f jenkins
unning from: /usr/share/jenkins/jenkins.war
webroot: EnvVars.masterEnvVars.get("JENKINS_HOME")
2020-10-31 16:13:06.472+0000 [id=1]	INFO	org.eclipse.jetty.util.log.Log#initialized: Logging initialized @292ms to org.eclipse.jetty.util.log.JavaUtilLog
2020-10-31 16:13:06.581+0000 [id=1]	INFO	winstone.Logger#logInternal: Beginning extraction from war file
2020-10-31 16:13:08.369+0000 [id=1]	WARNING	o.e.j.s.handler.ContextHandler#setContextPath: Empty contextPath
... 省略部分启动日志
Jenkins initial setup is required. An admin user has been created and a password generated.
Please use the following password to proceed to installation:
*************************************************************
*************************************************************
*************************************************************
Jenkins initial setup is required. An admin user has been created and a password generated.
Please use the following password to proceed to installation:
fb3499944e4845bba9d4b7d9eb4e3932
This may also be found at: /var/jenkins_home/secrets/initialAdminPassword
*************************************************************
*************************************************************
*************************************************************
This may also be found at: /var/jenkins_home/secrets/initialAdminPassword
2020-10-31 16:17:07.577+0000 [id=28]	INFO	jenkins.InitReactorRunner$1#onAttained: Completed initialization
2020-10-31 16:17:07.589+0000 [id=21]	INFO	hudson.WebAppMain$3#run: Jenkins is fully up and running
```



之后，我们通过访问 [http://localhost:8888](http://localhost:8888/) 就可以访问到 Jenkins 服务了。

![Drawing 2.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F-05giAJYDhAADCpDZzl2M065.png)

然后将日志中的密码粘贴到密码框即可，之后 Jenkins 会自动初始化，我们根据引导，安装推荐的插件即可。

![Drawing 3.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F-05hSAHd7VAAEpFeu4qfY218.png)



选择好安装推荐的插件后，Jenkins 会自动开始初始化一些常用插件。界面如下：

![Drawing 4.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F-05h-AUSiFAAEAxHFCt30058.png)



插件初始化完后，创建管理员账户和密码，输入用户名、密码和邮箱等信息，然后点击保存并完成即可。

![Drawing 5.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F-05iaANJXNAABXftlfsvQ115.png)

这里，确认 Jenkins 地址，我们就可以进入到 Jenkins 主页了。

![Drawing 6.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F-05i2AR2lRAADHULl7Ysk145.png)



然后在系统管理 -> 插件管理 -> 可选插件处，搜索 GitLab 和 Docker ，分别安装相关插件即可，以便我们的 Jenkins 服务和 GitLab 以及 Docker 可以更好地交互。

![Drawing 7.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F-05j2AfmTUAAGVJGSmG1g804.png)



等待插件安装完成， 重启 Jenkins ，我们的 Jenkins 环境就准备完成了。

现在，我们的 Docker+Jenkins+GitLab 环境已经准备完成，后面只需要把我们的代码推送到 GitLab 中，并做相关的配置即可实现推送代码自动构建镜像和发布。



### 结语

Docker 的出现解决了 CI/CD 流程中的各种问题，Docker 交付的镜像不仅包含应用程序，也包含了应用程序的运行环境，这很好地解决了开发和线上环境不一致问题。同时 Docker 的出现也极大地提升了 CI/CD 的构建效率，我们仅仅需要编写一个 Dockerfile 并将 Dockerfile 提交到我们的代码仓库即可快速构建出我们的应用，最后，当我们构建好 Docker 镜像后 Docker 可以帮助我们快速发布及更新应用。





## 25 | CI/CD：容器化后如何实现持续集成与交付？（下）



构建和部署一个应用的流程:

1. 我们首先需要配置 GitLab SSH 访问公钥，使得我们可以直接通过 SSH 拉取或推送代码到 GitLab。
2. 接着将代码通过 SSH 上传到 GitLab。
3. 再在 Jenkins 创建构建任务，使得 Jenkins 可以成功拉取 GitLab 的代码并进行构建。
4. 然后配置代码变更自动构建流程，使得代码变更可以触发自动构建 Docker 镜像。
5. 最后配置自动部署流程，镜像构建完成后自动将镜像发布到测试或生产环境。



### 1. 配置 GitLab SSH 访问公钥

为了能够让 Jenkins 顺利从 GitLab 拉取代码，我们需要先生成 ssh 密钥。我们可以使用 ssh-keygen 命令来生成 2048 位的 ras 密钥。在 Linux 上执行如下命令：

```sh
$ ssh-keygen -o -t rsa -b 2048 -C "email@example.com"
# 输入上面命令后系统会提示我们密钥保存的位置等信息，只需要按回车即可。
Generating public/private rsa key pair.
Enter file in which to save the key (/home/centos/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/centos/.ssh/id_rsa.
Your public key has been saved in /home/centos/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:A+d0NQQrjxV2h+zR3BQIJxT23puXoLi1RiTKJm16+rg email@example.com
The key's randomart image is:
+---[RSA 2048]----+
|          =XB=o+o|
|         ..=B+o .|
|      . + +. o   |
|       = B .o .  |
|      o S +  o . |
|     . * .... . +|
|      =  ..o   +.|
|     ...  o..   .|
|     E=. ...     |
+----[SHA256]-----+
```

执行完上述命令后 ，`$HOME/.ssh/` 目录下会自动生成两个文件：`id_rsa.pub` 文件为公钥文件，id_rsa 文件为私钥文件。我们可以通过 cat 命令来查看公钥文件内容：

```sh
$ cat $HOME/.ssh/id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDljSlDNHnUr4ursYISKXK5j2mWTYnt100mvYeJCLpr6tpeSarGyr7FnTc6sLM721plU2xq0bqlFEU5/0SSvFdLTht7bcfm/Hf31EdAuIqZuy/guP06ijpidfX6lVDxLWx/sO3Wbj3t7xgj4sfCFTiv+OOFP0NxKr5wy+emojm6KIaXkhjbPeJDgph5bvluFnKAtesMUkdhceAdN9grE3nkBOnwWw6G4dCtbrKt2o9wSyzgkDwPjj2qjFhcE9571/61/Nr8v9iqSHvcb/d7WZ0Qq7a2LYds6hQkpBg2RCDDJA16fFVs8Q5eNCpDQwGG3IbhHMUwvpKDf0OYrS9iftc5 email@example.com
```

然后将公钥文件拷贝到 GitLab 的个人设置 -> SSH Keys 中，点击添加按钮，将我们的公钥添加到 GitLab 中。

![Drawing 0.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl-2P_qAO6VIAAIAcpA55IY226.png)



### 2. 上传服务代码到 GitLab

这里，我使用 Golang 编写了一个 HTTP 服务，代码如下：

```go
package main
import (
    "fmt"
    "net/http"
)
func hello(w http.ResponseWriter, req *http.Request) {
    fmt.Fprintf(w, "hello\n")
}
func headers(w http.ResponseWriter, req *http.Request) {
    for name, headers := range req.Header {
        for _, h := range headers {
            fmt.Fprintf(w, "%v: %v\n", name, h)
        }
    }
}
func main() {
    http.HandleFunc("/hello", hello)
    http.HandleFunc("/headers", headers)
    http.ListenAndServe(":8090", nil)
}
```

然后编写一个 Dockerfile，利用多阶段构建将我们的 Go 编译，并将编译后的二进制文件拷贝到 scratch（scratch 是一个空镜像，用于构建其他镜像，体积非常小）的基础镜像中。Dockerfile 的内容如下：

```dockerfile
FROM golang:1.14 as builder
WORKDIR /go/src/github.com/wilhelmguo/devops-demo/
COPY main.go .
RUN CGO_ENABLED=0 GOOS=linux go build -o /tmp/http-server .
FROM scratch
WORKDIR /root/
COPY --from=builder /tmp/http-server .
CMD ["./http-server"]
```

编写完 Go HTTP 文件和 Dockerfile 文件后，代码目录内容如下：

```sh
$ ls -lh
total 24
-rw-r--r--  1 root  root   243B Nov  3 22:03 Dockerfile
-rw-r--r--  1 root  root    26B Nov  3 22:06 README.md
-rw-r--r--  1 root  root   441B Nov  3 22:03 main.go
```

> 源码详见[这里](https://github.com/wilhelmguo/devops-demo)



然后，我们在 GitLab 上创建一个 hello 项目，并将代码上传。

![Drawing 1.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl-2QA2ARz39AADE_fukgio780.png)

![Drawing 2.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl-2QQCAZUxWAAF7KHvN2DI582.png)



项目创建完成后，GitLab 会自动跳转到项目详情页面。

![Drawing 3.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F-2QQeAXsbVAAELrFGkphU008.png)



### 3. 创建 Jenkins 任务

在 Jenkins 中添加一个自由风格的任务。

![Drawing 4.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F-2QRGAIS83AAGKHDb05xE232.png)



点击确定，然后到源码管理选择 Git，填写 GitLab 项目的 URL。此时 Jenkins 会提示没有访问 GitLab 的相关权限，我们需要点击添加按钮将私钥添加到 Jenkins 中用以鉴权。

![Drawing 5.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F-2QSWAceMNAADnnjcKzCo548.png)

> 由于部署 GitLab 的宿主机 ssh 默认端口为 22，为了避免与宿主机的 ssh 端口冲突，我们的 GitLab ssh 端口配置为 2222，因此 Jenkins 连接 GitLab 的 URL 中需要包含端口号 2222， 配置格式为 `ssh://git@172.20.1.6:2222/root/hello.git`。



选择添加的密钥类型为 "SSH Username with private key"，Username 设置为 jenkins，然后将私钥粘贴到 Private Key 输入框中，点击添加即可。

![Drawing 6.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F-2QTSARpg5AAET_4BGb-0066.png)

添加完成后，认证名称选择 jenkins 后，红色报错提示就会消失。这证明此时 Jenkins 和 GitLab 已经认证成功，可以成功从 GitLab 拉取代码了。

![Drawing 7.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl-2QTqAQf8RAACXxIBN-Z8663.png)

下面我们使用 shell 脚本来构建我们的应用镜像，在构建中增加一个 Shell 类型的构建步骤，并且填入以下信息，将 USER 替换为目标镜像仓库的用户名，将 PASSWORD 替换为镜像仓库的密码。

```sh
# 第一步，登录镜像仓库
$ docker login -u {USER} -p  {PASSWORD}
# 第二步，使用 docker build 命令构建镜像
$ docker build -t lagoudocker/devops-demo . 
# 第三步, 使用 docker push 命令推送镜像
$ docker push lagoudocker/devops-demo 
```

![Drawing 8.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl-2QUKAJ-psAABwghmp76g949.png)

完成后点击保存，此时任务已经成功添加到 Jenkins 中。回到任务首页，点击构建按钮即可开始构建。第一次构建需要下载依赖的基础镜像，这个过程可能比较慢。构建过程中，我们也可以点击控制台查看构建输出的内容：

![Drawing 9.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F-2QUuAEXcXAAGe5l9e2h0928.png)



### 4. 配置自动构建

点击上一步创建的任务，点击配置进入任务配置界面，到构建触发器下勾选 GitLab 相关的选项，点击 Generate 按钮生成一个 GitLab 回调 Jenkins 的 token。记录下 Jenkins 的回调地址和生成的 token 信息。

![Drawing 10.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F-2QWCABHzrAAFQCgpFnLs787.png)

在 GitLab 项目设置中，选择 Webhooks，将 Jenkins 的回调地址和 token 信息添加到 Webhooks 的配置中，点击添加即可。

![Drawing 11.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F-2QWiAFOVBAAI93Lelr38996.png)

后面我们的每次提交都会触发自动构建。

为了实现根据 git 的 tag 自动构建相应版本的镜像，我们需要修改 Jenkins 构建步骤中的 shell 脚本为以下内容：

```sh
# 需要推送的镜像名称
IMAGE_NAME="lagoudocker/devops-demo" 
# 获取当前构建的版本号
GIT_VERSION=`git describe --always --tag`
# 生成完整的镜像 URL 变量，用于构建和推送镜像
REPOSITORY=docker.io/${IMAGE_NAME}:${GIT_VERSION} 
# 构建Docker镜像 
docker build -t $REPOSITORY -f Dockerfile . 
# 登录镜像仓库，username 跟 password 为目标镜像仓库的用户名和密码
docker login --username=xxxxx --password=xxxxxx docker.io
# 推送 Docker 镜像到目标镜像仓库
docker push $REPOSITORY 
```

到此我们已经完成了 GitLab -> Jenkins -> Docker 镜像仓库的自动构建和推送。当我们推送代码到 GitLab 中时，会自动触发 Webhooks，然后 GitLab 会根据配置的 Webhooks 调用 Jenkins 开始构建镜像，镜像构建完成后自动将镜像上传到我们的镜像仓库。



### 5. 配置自动部署

镜像构建完成后，我们还需要将镜像发布到测试或生产环境中将镜像运行起来。发布到环境的过程可以设置为自动发布，每当我们推送代码到 master 中时，即开始自动构建镜像，并将构建后的镜像发布到测试环境中。

在镜像构建过程中，实际上 Jenkins 是通过执行我们编写的 shell 脚本完成的，要想实现镜像构建完成后自动在远程服务器上运行最新的镜像，我们需要借助一个 Jenkins 插件 Publish Over SSH，这个插件可以帮助我们自动登录远程服务器，并执行一段脚本将我们的服务启动。

下面我们来实际操作下这个插件。

**第一步，在 Jenkins 中安装 Publish Over SSH 插件。** 在 Jenkins 系统管理，插件管理中，搜索 Publish Over SSH，然后点击安装并重启 Jenkins 服务。

![Drawing 12.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl-2QfmAc4iBAACDzvOoPWI585.png)



**第二步，配置 Publish Over SSH 插件。** 插件安装完成后，在 Jenkins 系统管理的系统设置下，找到 Publish Over SSH 功能模块，添加远程服务器节点，这里我使用密码验证的方式添加一台服务器。配置好后，我们可以使用测试按钮测试服务器是否可以正常连接，显示Success 代表服务器可以正常连接，测试连接成功后，点击保存按钮保存配置。

![Drawing 13.png](D:\学习资料\笔记\k8s\k8s图\CgqCHl-2QgSAVk0bAAC6abody2k836.png)



**第三步，修改之前 shell 任务中脚本，** 添加部署相关的内容：

```sh
# 需要推送的镜像名称
IMAGE_NAME="lagoudocker/devops-demo" 
# 获取当前构建的版本号
GIT_VERSION=`git describe --always --tag`
# 生成完整的镜像 URL 变量，用于构建和推送镜像
REPOSITORY=docker.io/${IMAGE_NAME}:${GIT_VERSION} 
# 构建Docker镜像 
docker build -t $REPOSITORY -f Dockerfile . 
# 登录镜像仓库，username 跟 password 为目标镜像仓库的用户名和密码
docker login --username={USER} --password={PASSWORD} docker.io
# 推送 Docker 镜像到目标镜像仓库
docker push $REPOSITORY 
mkdir -p ./shell && echo \ 
"docker login --username={USER} --password={PASSWORD} \n"\ 
"docker pull $REPOSITORY\n"\ 
"docker kill hello \n"\ 
"docker run --rm --name=hello -p 8090:8090 -d $REPOSITORY" >> ./shell/release
```

我们在 docker push 命令后，增加一个输出 shell 脚本到 release 文件的命令，这个脚本会发送到远端的服务器上并执行，通过执行这个脚本文件可以在远端服务器上，拉取最新镜像并且重新启动容器。



**第四步，配置远程执行。\**在 Jenkins 的 hello 项目中，点击配置，在执行步骤中点击添加\**Send files or execute commands over SSH**的步骤，选择之前添加的服务器，并且按照以下内容填写相关信息。

![Drawing 14.png](D:\学习资料\笔记\k8s\k8s图\Ciqc1F-2QhKAPblBAAC4Bp33K2Y632.png)



- Source file 就是我们要传递的 shell 脚本信息，这里填写我们上面生成的 shell 脚本文件即可。
- Remove prefix 是需要过滤的目录，这里我们填写 shell。
- Remote directory 为远程执行脚本的目录。

最后点击保存，保存我们的配置即可。配置完成后，我们就完成了推送代码到 GitLab，Jenkins 自动构建镜像，之后推送镜像到镜像仓库，最后自动在远程服务器上拉取并重新部署容器。

> 如果你是生产环境中使用的 Kubernetes 管理服务，可以在 Jenkins 中安装 Kubernetes 的插件，然后构建完成后直接发布镜像到 Kubernetes 集群中。



### 结语

本课时我们使用 Go 开发了一个简单的 HTTP 服务，并将代码托管在了 GitLab 中。然后通过配置 GitLab 和 Jenkins 的相互调用，实现了推送代码到 GitLab 代码仓库自动触发构建镜像并将镜像推送到远程镜像仓库中，最后将最新版本镜像发布到远程服务器上。

DevOps 是一个非常棒的指导思想，而 CI/CD 是整个 DevOps 流程中最重要的部分，目前 CI/CD 的市场已经非常成熟，CI/CD 的工具链也非常完善，因此，无论是小团队还是大团队，都有必要去学习和掌握 CI/CD，以便帮助我们改善团队的效能，一切可以自动化的流程，都应该尽量避免人工参与。

