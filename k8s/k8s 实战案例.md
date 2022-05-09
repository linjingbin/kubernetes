# 1 基础集群环境搭建

k8s 基础集群环境主要是运行 kubernetes 管理端服务以及 node 节点上的服务部署及使用。

**Kubernetes 设计架构：**

https://feisky.gitbooks.io/kubernetes/architecture/architecture.html

**CNCF 云原生容器生态系统概要：**

http://dockone.io/article/3006



## 1.1 k8s 高可用集群环境规划信息

安装实际需求，进行规划与部署相应的单 master 或多 master 的高可用 k8s 运行环境。



### 1.1.1 kubeadm 安装 k8s 集群

#### 1.1.1.1 k8s 组件介绍

**kube-apiserver：**

Kubernetes API server 为 api 对象验证并配置数据，包括 pods、services、replicationcontrollers 和其它 api 对象,API Server 提供 REST 操作和到集群共享状态的前端，所有其他组件通过它进行交互。

**Kubernetes scheduler：**

Kubernetes scheduler 是一个拥有丰富策略、能够感知拓扑变化、支持特定负载的功能组件，它对集群的可用性、性能表现以及容量都影响巨大。scheduler 需要考虑独立的和集体的资源需求、服务质量需求、硬件/软件/策略限制、亲和与反亲和规范、数据位置、内部负载接口、截止时间等等。如有必要，特定的负载需求可以通过 API 暴露出来。

**kube-controller-manager：**

Controller Manager 作为集群内部的管理控制中心，负责集群内的 Node、Pod 副本、服务端点（Endpoint）、命名空间（Namespace）、服务账号（ServiceAccount）、资源定额（ResourceQuota）的管理，当某个 Node
意外宕机时，Controller Manager 会及时发现并执行自动化修复流程，确保集群始终处于预期的工作状态。

**kube-proxy：**

Kubernetes 网络代理运行在 node 上，它反映了 node 上 Kubernetes API 中定义的服务，并可以通过一组后端进行简单的 TCP、UDP 流转发或循环模式（round robin)）的 TCP、UDP 转发，用户必须使用 apiserver API 创建一个服务来配置代理，其实就是 kube-proxy 通过在主机上维护网络规则并执行连接转发来实现 Kubernetes 服务访问。

**kubelet ：**

节点代理，监视已分配给节点的 pod，具体功能如下：

- 向 master 汇报 node 节点的状态信息
- 接受指令并在 Pod 中创建 docker 容器
- 准备 Pod 所需的数据卷
- 返回 pod 的运行状态
- 在 node 节点执行容器健康检查

**etcd：**

etcd 是 Kubernetes 提供默认的存储系统，保存所有集群数据，使用时需要为 etcd 数据提供备份计划。



#### 1.1.1.2 部署规划图

![1588214526174](D:\学习资料\笔记\linux\assets\1588214526174.png)



#### 1.1.1.3 kubeadm

使用 k8s 官方提供的部署工具 kubeadm 自动安装，需要在 master 和 node 节点上安装 docker 等组件，然后初始化，把管理端的控制服务和 node 上的服务都以 pod 的方式运行。

**安装注意事项：禁用 swap, seliunx, iptables**

```bash
#关闭防火墙：
$ systemctl stop firewalld iptables
$ systemctl disable firewalld
$ systemctl disable iptables

#关闭selinux：
$ sed -i 's/SELINUX=.*/SELINUX=disabled/' /etc/selinux/config 
$ setenforce 0

#关闭swap：
$ swapoff -a $ 临时
$ vim /etc/fstab $ 永久

#设置主机名：
$ hostnamectl set-hostname <hostname>

#在master添加hosts：
$ cat >> /etc/hosts << EOF
192.168.31.61 k8s-master
192.168.31.62 k8s-node1
192.168.31.63 k8s-node2
EOF

#将桥接的IPv4流量传递到iptables的链：
$ cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
$ sysctl --system

#时间同步：
$ yum install ntpdate -y
$ ntpdate time.windows.com
```

**启用 ipvs 内核模块（可选步骤）**

```bash
[root@ansible_ modules]$ cat /etc/sysconfig/modules/ipvs.modules
#!/bin/bash
ipvs_mods_dir="/usr/lib/modules/$(uname -r)/kernel/net/netfilter/ipvs"
for i in $(ls $ipvs_mods_dir | grep -o "^[^.]*"); do
  /sbin/modinfo -F filename $i &> /dev/null
  if [ $? -eq 0 ]; then
    /sbin/modprobe $i
  fi
done

[root@k8s-1 modules]$ chmod +x /etc/sysconfig/modules/ipvs.modules 
[root@k8s-1 modules]$ /etc/sysconfig/modules/ipvs.modules 
```



#### 1.1.1.4 安装步骤

具体步骤：

1. master 和 node 所有节点安装 Kubelet/Docker，Kubeadm
2. maser 节点运行 kubeadm init 初始化命令
3. 验证 master
4. Node 节点使用 kubeadm 加入 k8s master
5. 验证 node
6. 启动容器测试访问

**1.1.1.4.1 安装 docker**

```bash
wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
$ yum -y install docker-ce-18.06.1.ce-3.el7
$ systemctl enable docker && systemctl start docker
$ docker --version
Docker version 18.06.1-ce, build e68fc7a
```

> 另外，Docker 自 1.13 版起会自动设置 iptables 的 FORWARD 默认策略为 DROP，这可能会影响 kubernetes 集群依赖的报文转发功能。因此，需要在 docker 服务启动后，重新将 FORWARD 链的默认策略设置为 ACCEPT，方式是修改 /usr/lib/systemd/system/docker.service 文件，在 "ExecStart=/usr/bin/dockerd" 一行之后新增一行如下内容：
>
> ExecStartPost=/usr/sbin/iptables -P FORWARD ACCEPT

```bash
[root@k8s-6 system]# cat docker.service
[Service]
Type=notify
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
ExecStartPost=/usr/sbin/iptables -P FORWARD ACCEPT
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutSec=0
RestartSec=2
Restart=always

[root@k8s-6 system]# systemctl daemon-reload
[root@k8s-6 system]# systemctl restart docker.service 
```



##### 1.1.1.4.2 配置 docker 加速器

```bash
sudo mkdir -p /etc/docker
cat /etc/docker/daemon.json << EOF
{
  "registry-mirrors": ["https://70enc47m.mirror.aliyuncs.com"],
  "storage-driver": "overlay2"
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```



##### 1.1.1.4.3 配置阿里云仓库地址

配置阿里云镜像的 kubernetes 源(用于安装 kubelet kubeadm kubectl 命令)

```bash
cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

##### 1.1.1.4.4 安装 kubeadm,kubelet 和 kubectl

```bash
$ yum install -y kubelet-1.13.5 kubeadm-1.13.5 kubectl-1.13.5
$ systemctl enable kubelet
```

##### 1.1.1.4.6 kubeadm 命令使用

**命令使用：**
https://kubernetes.io/zh/docs/reference/setup-tools/kubeadm/kubeadm/

**集群初始化名：**
https://kubernetes.io/zh/docs/reference/setup-tools/kubeadm/kubeadm-init/

```bash
root@docker-node1:~# kubeadm --help
--apiserver-advertise-address string #API Server 将要监听的监听地址
--apiserver-bind-port int32 #API Server 绑定的端口,默认为 6443,
--apiserver-cert-extra-sans stringSlice #可选的证书额外信息，用于指定 API Server的服务器证书。可以是 IP 地址也可以是 DNS 名称。
--cert-dir string #证书的存储路径，缺省路径为 /etc/kubernetes/pki
--config string #kubeadm 配置文件的路径
--ignore-preflight-errors strings #可以忽略检查过程中出现的错误信息，比如忽略swap，如果为 all 就忽略所有
--image-repository string #设置一个镜像仓库，默认为 k8s.gcr.io
--kubernetes-version string #选择 k8s 版本，默认为 stable-1
--node-name string #指定 node 名称
--pod-network-cidr #设置 pod ip 地址范围
--service-cidr #设置 service 网络地址范围
--service-dns-domain string #设置域名，默认为 cluster.local
--skip-certificate-key-print #不打印用于加密的 key 信息
--skip-phases strings #要跳过哪些阶段
--skip-token-print #跳过打印 token 信息
--token #指定 token
--token-ttl #指定 token 过期时间，默认为 24 小时，0 为永不过期
--upload-certs #更新证书

#全局选项
--log-file string #日志路径
--log-file-max-size uint #设置日志文件的最大大小，单位为兆，默认为 1800，0 为没有限制 
--rootfs #宿主机的根路径，也就是使用绝对路径
--skip-headers #为 true，在 log 日志里面不显示消息的头部信息
--skip-log-headers #为 true 在日志文件里面不记录头部信息
```

**验证版本：**

```bash
kubeadm version #查看当前 kubeadm 版本
kubeadm config images list --kubernetes-version v1.13.5 #查看安装指定版本 k8s
 
```



##### 1.1.1.4.7 需要的镜像

```
k8s.gcr.io/kube-apiserver:v1.13.5
k8s.gcr.io/kube-controller-manager:v1.13.5
k8s.gcr.io/kube-scheduler:v1.13.5
k8s.gcr.io/kube-proxy:v1.13.5
k8s.gcr.io/pause:3.1
k8s.gcr.io/etcd:3.2.24
k8s.gcr.io/coredns:1.3.1
```



##### 1.1.1.4.8 初始化 master

```bash
kubeadm  init  \
--apiserver-advertise-address=192.168.7.101 \
--image-repository registry.aliyuncs.com/google_containers \ #由于默认拉取镜像地址k8s.gcr.io国内无法访问，这里指定阿里云镜像仓库地址。
--apiserver-bind-port=6443 \
--kubernetes-version=v1.13.5 \
--pod-network-cidr=10.10.0.0/16  \
--service-cidr=10.20.0.0/16  \
--service-dns-domain=linux36.local  \
# --ignore-preflight-errors=swap 如果没有禁用swap，可加此选项
```

![1588231235623](D:\学习资料\笔记\linux\assets\1588231235623.png)



##### 1.1.1.4.9 master 配置 kube 证书

证书中包含 kube-apiserver 地址及相关认证信息

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
cat /root/.kube/config
```



##### 1.1.1.4.10 验证 k8s 状态

```bash
root@docker-node1:~# kubectl get cs
NAME STATUS MESSAGE ERROR
controller-manager Healthy ok
scheduler Healthy ok
etcd-0 Healthy {"health":"true"}
```

![1588232019999](D:\学习资料\笔记\linux\assets\1588232019999.png)



##### 1.1.1.4.11 安装 Pod 网络插件 CNI (flannel)

```bash
root@docker-node1:~#  kubectl  apply  -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

![1588232378089](D:\学习资料\笔记\linux\assets\1588232378089.png)



##### 1.1.1.4.12 添加两个 node 节点

各 node 节点都要安装 docker kubeadm kubelet，因此都要执行之前步骤，即配置 apt 仓库、配置 docker 加速器、安装命令、启动 kubelet 服务。

```bash
root@docker-node2:~# systemctl start docker kubelet
root@docker-node2:~# systemctl enable docker kubelet

root@docker-node2:~#  kubeadm  join  192.168.7.101:6443  --token dzh911.z4sa1rk48xznlu31  --discovery-token-ca-cert-hash sha256:0faae813959a0f45d41be2ae5c4658300a2e4771101cfd2010c745e6bed149dc
```

![1588232568019](D:\学习资料\笔记\linux\assets\1588232568019.png)

 注：Node 节点会自动加入到 master 节点，下载镜像并启动 flannel，直到在 master 看到 node 处于 Ready 状态。

![1588774944041](D:\学习资料\笔记\linux\assets\1588774944041.png)



##### 1.1.1.4.13 k8s 创建容器并测试

创建测试容器，测试网络连接

```bash
kubectl run net-test1 --image=alpine --replicas=2 sleep 360000
```

![1588775052182](D:\学习资料\笔记\linux\assets\1588775052182.png)



##### 1.1.1.4.14 kubeadm init 流程

<https://k8smeetup.github.io/docs/reference/setup-tools/kubeadm/kubeadm-init/#init-workflow>



#### 1.1.1.5 kubeadm 升级 k8s 集群

升级 k8s 集群必须 先升级 kubeadm 版本到目的 k8s 版本

- **验证当前 k8s 版本**

```bash
root@docker-node1:~# kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"13",
GitVersion:"v1.13.5",GitCommit:"2166946f41b36dea2c4626f90a77706f4
26cdea2", GitTreeState:"clean", BuildDate:"2019-03-25T15:24:33Z",
GoVersion:"go1.11.5", Compiler:"gc", Platform:"linux/amd64"}
```

- **安装指定版本 kubeadm**

```bash
root@docker-node1:~# apt-cache madison kubeadm #查看具体版本
root@docker-node1:~# apt-get install kubeadm=1.13.6-00 #安装具体版本
root@docker-node1:~# kubeadm version #验证版本
kubeadm  version:  &version.Info{Major:"1",  Minor:"13",  GitVersion:"v1.13.6",
GitCommit:"abdda3f9fefa29172298a2e42f5102e777a8ec25", GitTreeState:"clean", BuildDate:"2019-05-08T13:51:38Z", GoVersion:"go1.11.5", Compiler:"gc", Platform:"linux/amd64"}
```

- **kubeadm 升级命令使用帮助**

```bash
root@docker-node1:~# kubeadm upgrade --help
```

- **升级计划**

```bash
root@docker-node1:~# kubeadm upgrade plan #查看升级计划
```

![1588775684228](D:\学习资料\笔记\linux\assets\1588775684228.png)

- **开始升级**

```bash
root@docker-node1:~# kubeadm upgrade apply v1.13.6
```

![1588775734914](D:\学习资料\笔记\linux\assets\1588775734914.png)

![1588775743029](D:\学习资料\笔记\linux\assets\1588775743029.png)

**升级 kubelet**

否则 node 节点还是 1.13.5 的旧版本，但是 server 已经是 1.13.6

![1588775836296](D:\学习资料\笔记\linux\assets\1588775836296.png)

```bash
root@docker-node1:~# kubectl get nodes
NAME STATUS ROLES AGE VERSION
docker-node1.magedu.net Ready master 18h v1.13.5
docker-node2.magedu.net Ready <none> 18h v1.13.5
docker-node3.magedu.net Ready <none> 18h v1.13.5

root@docker-node1:~# kubeadm upgrade node config --kubelet-version 1.13.6
```

- **升级 node 节点配置文件**

```bash
root@docker-node1:~# kubeadm upgrade node config --kubelet-version 1.13.6
[kubelet] Downloading configuration for the kubelet from the "kubelet-config-1.13"
Conf
igMap in the kube-system namespace[kubelet-start] Writing kubelet configuration to
file "/var/lib/kubelet/config.yaml"
[upgrade] The configuration for this node was successfully updated!
[upgrade] Now you should go ahead and upgrade the kubelet package using your
package manager.
```

 **各 Node  节点升级 kubelet  二进制包**

```bash
root@docker-node2:~# apt-get install kubelet=1.13.6-00
```

![1588777157143](D:\学习资料\笔记\linux\assets\1588777157143.png)



#### 1.1.1.6 从集群中移除节点

**在 Master 上执行如下命令：**

```bash
kubectl drain NODE_ID --delete-local-data --force --ignore-daemonsets
kubectl delete node NODE_ID
```

在要删除的 Node 上执行如下命令重置系统状态便可完成移除操作：

```bash
kubeadm
```



### 1.1.2 多 master

![1588777309991](D:\学习资料\笔记\linux\assets\1588777309991.png)



### 1.1.3 服务器统计

![1588777409208](D:\学习资料\笔记\linux\assets\1588777409208.png)



## 1.2 主机名设置

![1588777455797](D:\学习资料\笔记\linux\assets\1588777455797.png)



## 1.3 软件清单

![1588813565448](D:\学习资料\笔记\linux\assets\1588813565448.png)

 [calico-release-v3.3.6.tgz](..\..\马哥2019\马哥教育杰哥视频\第20天\calico-release-v3.3.6.tgz) 

 [ansible-k8s_1.13.5-all_image-and_yaml-linux36.tar.gz](..\..\马哥2019\马哥教育杰哥视频\第20天\ansible-k8s_1.13.5-all_image-and_yaml-linux36.tar.gz) 

 [kubeasz-0.6.0.zip](..\..\马哥2019\马哥教育杰哥视频\第20天\kubeasz-0.6.0.zip) 

**API 端口：**

```bash
端口：192.168.7.248：6443 #需要配置在负载均衡上实现反向代理，dashboard的端口为8443
操作系统：ubuntu server 1804
k8s版本： 1.13.5
calico：3.4.4
```



## 1.4 基础环境准备

系统主机名配置、IP配置、系统参数优化，以及依赖的负载均衡和 Harbor 部署

### 1.4.1 系统配置

主机名等系统配置

### 1.4.2 高可用负载均衡

k8s 高可用反向代理

#### 1.4.2.1 keepalived 安装及配置

**编译安装 keepalived**

```bash
[root@linux-node137 ~]$ cd /usr/local/src/
[root@linux-node137 src]$ wget http://www.keepalived.org/software/keepalived-1.3.4.tar.gz
[root@linux-node137 src]$ tar xvf keepalived-1.3.4.tar.gz
[root@linux-node137 src]$ cd keepalived-1.3.4/
[root@linux-node137 keepalived-1.3.4]$ yum install libnfnetlink-devel libnfnetlink ipvsadm  libnl libnl-devel  libnl3 libnl3-devel   lm_sensors-libs net-snmp-agent-libs net-snmp-libs  openssh-server openssh-clients  openssl openssl-devel automake iproute 

[root@localhost keepalived-1.3.4]$  ./configure --prefix=/usr/local/keepalived --disable-fwmark

[root@linux-node137 keepalived-1.3.4]$ make && amke install
```

**复制相关配置文件及启动脚本**

```bash
[root@linux-node137 keepalived-1.3.4]$ cp /usr/local/src/keepalived-1.3.4/keepalived/etc/init.d/keepalived.rh.init /etc/sysconfig/keepalived.sysconfig
[root@linux-node137 keepalived-1.3.4]$ cp /usr/local/src/keepalived-1.3.4/keepalived/keepalived.service  /usr/lib/systemd/system/
[root@linux-node137 keepalived-1.3.4]$ cp  /usr/local/src/keepalived-1.3.4/bin/keepalived  /usr/sbin/
```

**准备一个简单的配置文件**

```bash
root@k8s-ha1:~$ cat /etc/keepalived/keepalived.conf
vrrp_instance VI_1 {
 state MASTER
 interface eth0
 virtual_router_id 1
 priority 100
 advert_int 3
 unicast_src_ip 192.168.7.108
 unicast_peer {
   192.168.7.109
 }
 authentication {
   auth_type PASS
   auth_pass 123abc
 }
 virtual_ipaddress {
   192.168.7.248 dev eth0 label eth0:1
 }
}
```



#### 1.4.2.2 安装haproxy

**编译安装 haproxy**

 [haproxy-1.8.25.tar.gz](..\..\软件\haproxy-1.8.25.tar.gz) 

```bash
[root@linux-node137 ~]$ cd /usr/local/src/
[root@linux-node137 src]$ http://www.haproxy.org/download/1.8/src/haproxy-1.8.25.tar.gz
[root@linux-node137 src]$ tar xvf haproxy-1.8.25.tar.gz
[root@linux-node137 src]$ cd haproxy-1.8.25/
[root@linux-node137 haproxy-1.8.25]$  yum install gcc pcre pcre-devel openssl  openssl-devel -y
[root@linux-node137 haproxy-1.8.25]$ vim README #安装文档及相关帮助信息
[root@linux-node137 haproxy-1.8.25]$ make ARCH=x86_64 TARGET=linux2628 USE_PCRE=1 USE_OPENSSL=1 USE_ZLIB=1 USE_SYSTEMD=1USE_CPU_AFFINITY=1 PREFIX=/usr/local/haproxy
[root@linux-node137 haproxy-1.8.25]$ make install PREFIX=/usr/local/haproxy
[root@linux-node137 haproxy-1.8.25]$ cp haproxy /usr/sbin/
```

**准备启动脚本文件**

```bash
[root@linux-node137 haproxy-1.8.25]$ vim /usr/lib/systemd/system/haproxy.service
[Unit]
Description=HAProxy Load Balancer
After=syslog.target network.target

[Service]
EnvironmentFile=/etc/sysconfig/haproxy
ExecStartPre=/usr/sbin/haproxy -f /etc/haproxy/haproxy.cfg -c -q
ExecStart=/usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid
ExecReload=/bin/kill -USR2 $MAINPID

[Install]
WantedBy=multi-user.target
```

**创建目录和用户**

```bash
$ mkdir /etc/haproxy
$ useradd haproxy -s /sbin/nologin
$ mkdir /var/lib/haproxy
$ chown haproxy.haproxy /var/lib/haproxy/ -R
```

**主备配置文件，简单配置，后续完善**

```bash
[root@linux-node137 haproxy-1.8.25]$  vim /etc/haproxy/haproxy.cfg
global
	maxconn 100000
	chroot /usr/local/haproxy
	#stats socket /var/lib/haproxy/haproxy.sock mode 600 level admin
	#uid 99
	#gid 99
	uid haproxy
	gid haproxy
	daemon
	nbproc 4
	cpu-map 1 0
	cpu-map 2 1
	cpu-map 3 2
	cpu-map 4 3
	pidfile /usr/local/haproxy/run/haproxy.pid
	log 127.0.0.1 local3 info

defaults
	option http-keep-alive
	option  forwardfor
	maxconn 100000
	mode http
	timeout connect 300000ms
	timeout client  300000ms
	timeout server  300000ms

listen stats
	mode http
 	bind 0.0.0.0:9999
	stats enable
 	log global
 	stats uri     /haproxy-status
 	stats auth    haadmin:q1w2e3r4ys

listen k8s_api_nodes_6443
	bind 192.168.7.248:6443
 	mode tcp
 	#balance leastconn
 	server 192.168.7.101 192.168.7.101:6443 check inter 2000 fall 3 rise 5
 	#server 192.168.7.202 192.168.7.202:6443 check inter 2000 fall 3 rise 5

#haproxy.cfg文件中定义了chroot、pidfile、user、group等参数，如果系统没有相应的资源会导致haproxy无法启动，具体参考日志文件/var/log/messages

#启动HAProxy：
systemctl enable haproxy
systemctl restart haproxy
```

**开启 haproxy 日志**

```bash
[root@linux-node137 ~]# vim /etc/rsyslog.conf
 15 $ModLoad imudp
 16 $UDPServerRun 514
 92 local3.*         /var/log/haproxy.log #保存后的日志目录
 
 #重启 rsyslog 服务
 [root@linux-node137 ~]# systemctl  restart  rsyslog
```

**配置 haproxy 调用 rsyslog**

```bash
[root@linux-node137 ~]$  vim /etc/haproxy/haproxy.cfg
 9 log 127.0.0.1 local3 info
[root@linux-node137 ~]$  systemctl  restart haproxy
```

**访问 web 界面并验证 haproxy 日志目录**

```bash
[root@linux-node137 ~]$ tail /var/log/haproxy.log 
Mar  9 16:04:40 localhost haproxy[55688]: Proxy stats started.
Mar  9 16:04:40 localhost haproxy[55688]: Proxy web_port started.
Mar  9 16:06:45 localhost haproxy[55689]: Connect from 192.168.10.1:2623 to 192.168.10.137:80 (web_port/TCP)
```



### 1.4.3 Harbor 之 https

**内部镜像将统一保存在内部 Harbor 服务器**

```bash
root@k8s-harbor1:/usr/local/src/harbor$ pwd
/usr/local/src/harbor
root@k8s-harbor1:/usr/local/src/harbor$  mkdir certs/
$  openssl genrsa -out /usr/local/src/harbor/certs/harbor-ca.key #生成私有key
$  openssl req -x509 -new -nodes -key /usr/local/src/harbor/certs/harbor-ca.key -subj "/CN=harbor.magedu.net" -days 7120 -out /usr/local/src/harbor/certs/harbor-ca.crt #签证

$ vim harbor.cfg
hostname = harbor.magedu.net
ui_url_protocol = https
ssl_cert =  /usr/local/src/harbor/certs/harbor-ca.crt
ssl_cert_key = /usr/local/src/harbor/certs/harbor-ca.key
harbor_admin_password = 123456

$ ./install.sh
```

**client 同步在 crt 证书**

```bash
master1:~$ mkdir  /etc/docker/certs.d/harbor.magedu.net -p
harbor1:~$  scp /usr/local/src/harbor/certs/harbor-ca.crt
192.168.7.101:/etc/docker/certs.d/harbor.magedu.net
master1:~$ vim /etc/hosts #添加host文件解析
192.168.7.103 harbor.magedu.net
master1:~$ systemctl restart docker #重启docker
```

**测试登录 harbor**

```bash
root@k8s-master1:~$ docker login harbor.magedu.net
Username: admin
Password:
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store
Login Succeeded
```

**测试 push 镜像到 harbor**

```bash
master1:~$ docker pull alpine
root@k8s-master1:~$ docker tag alpine harbor.magedu.net/library/alpine:linux36
root@k8s-master1:~$ docker push harbor.magedu.net/library/alpine:linux36
The push refers to repository [harbor.magedu.net/library/alpine]
256a7af3acb1: Pushed
linux36: digest:
sha256:97a042bf09f1bf78c8cf3dcebef94614f2b95fa2f988a5c07314031bc2570c7a size: 528
```



## 1.5 手动二进制部署

**基于 ubuntu 1804 安装 k8s**

### 1.5.1 准备工作

**安装常用命令**

```bash
$ apt-get update
$ apt-get purge ufw lxd lxd-client lxcfs lxc-common #卸载不用的包
$ apt-get  install iproute2  ntpdate  tcpdump telnet traceroute nfs-kernel-server nfs-common lrzsz tree  openssl libssl-dev libpcre3 libpcre3-dev zlib1g-dev ntpdate tcpdump telnet traceroute  gcc openssh-server lrzsz tree  openssl libssl-dev libpcre3 libpcre3-dev zlib1g-dev ntpdate tcpdump telnet traceroute iotop unzip zip
```

**安装 docker**

```bash
root@k8s-node1:~$ apt-get update
root@k8s-node1:~$ apt-get -y install apt-transport-https ca-certificates curl software-properties-common
root@k8s-node1:~$ curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
root@k8s-node1:~$ add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
root@k8s-node1:~$ apt-get -y update && apt-get -y install docker-ce
root@k8s-node1:~$ docker info
```

**系统优化配置**

```bash
root@k8s-node1:~$ grep "^[a-Z]" /etc/sysctl.conf 
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
vm.swappiness=0
net.ipv4.ip_forward = 1
```



### 1.5.2 服务器初始化及证书制作

```bash
$ yum install -y https://mirrors.aliyun.com/docker-ce/linux/centos/7/x86_64/stable/Packages/docker-ce-selinux-17.03.2.ce-1.el7.centos.noarch.rpm

$ yum install -y https://mirrors.aliyun.com/docker-ce/linux/centos/7/x86_64/stable/Packages/docker-ce-17.03.2.ce-1.el7.centos.x86_64.rpm
```

**配置主机名和 host 文件：同步各服务器时间**

```bash
192.168.100.101 k8s-master1.example.com   k8s-master1
192.168.100.102 k8s-master2.example.com   k8s-master2
192.168.100.103 k8s-harbor1.example.com   k8s-harbor1
192.168.100.104 k8s-harbor2.example.com   k8s-harbor2
192.168.100.105 k8s-etcd1.example.com       k8s-etcd1
192.168.100.106 k8s-etcd2.example.com       k8s-etcd2
192.168.100.107 k8s-etcd3.example.com       k8s-etcd3
192.168.100.108 k8s-node1.example.com      k8s-node1
192.168.100.109 k8s-node2.example.com      k8s-node2
192.168.100.110 k8s-haproxy1.example.com k8s-haproxy1
192.168.100.111 k8s-haproxy2.example.com k8s-haproxy2
VIP:192.168.100.112
```

```bash
[root@k8s-master1 ~]$ yum install sshpass -y
ssh-keygen
```



### 1.5.3 安装 harbor 服务器

```bash
hostname = k8s-harbor1.example.com
ui_url_protocol = https

ssl_cert = /usr/local/src/harbor/cert/server.crt
ssl_cert_key = /usr/local/src/harbor/cert/server.key
harbor_admin_password = 123456

$ mkdir  /usr/local/src/harbor/cert
$ openssl genrsa -out /usr/local/src/harbor/cert/server.key 2048  #生成私有key
$ openssl req -x509 -new -nodes -key /usr/local/src/harbor/cert/server.key  -subj "/CN=k8s-harbor1.example.com" -days 7120 -out /usr/local/src/harbor/cert/server.crt   #创建有效期时间的自签名证书

$ openssl req -x509 -new -nodes -key /usr/local/src/harbor/cert/server.key -subj "/CN=k8s-harbor2.example.com" -days 7120 -out /usr/local/src/harbor/cert/server.crt   #创建有效期时间的自签名证书

$ yum install python-pip -y
$ pip install docker-compose
```

**配置客户端使用 harbor**

```bash
$ mkdir /etc/docker/certs.d/k8s-harbor1.example.com -pv
$ mkdir /etc/docker/certs.d/k8s-harbor2.example.com -pv

[root@k8s-harbor1 harbor]$ scp cert/server.crt  192.168.100.101:/etc/docker/certs.d/k8s-harbor1.example.com/
[root@k8s-harbor2 harbor]$ scp cert/server.crt  192.168.100.101:/etc/docker/certs.d/k8s-harbor2.example.com/
```

**测试登录**

```bash
[root@k8s-master1 ~]$ docker login k8s-harbor1.example.com
Username (admin):  
Password: 
Login Succeeded

[root@k8s-master1 ~]# docker login k8s-harbor2.example.com
Username (admin): 
Password: 
Login Succeeded
```



### 1.5.4 准备证书环境

使用 cfssl 来生成自签证书，先下载 cfssl 工具

```bash
#每个机器
mkdir -p /opt/kubernetes/{cfg,bin,ssl,log}  

#准备证书制作工具：
$ cd /usr/local/src
$ wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
$ wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
$ wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
$ chmod a+x cfssl_linux-amd64 cfssljson_linux-amd64 cfssl-certinfo_linux-amd64
[root@k8s-master1 src]$ mv cfssl-certinfo_linux-amd64  /usr/bin/cfssl-certinfo
[root@k8s-master1 src]$ mv cfssljson_linux-amd64  /usr/bin/cfssljson
[root@k8s-master1 src]$ mv cfssl_linux-amd64  /usr/bin/cfssl

[root@k8s-master1 ~]$ cd /usr/local/src/ #初始化cfssl
[root@k8s-master1 src]$ cfssl print-defaults config > config.json
[root@k8s-master1 src]$ cfssl print-defaults csr > csr.json
```

**创建生成 CA 的 json 文件**

```bash
[root@k8s-master1 src]$ vim  ca-config.json
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "87600h"
      }
    }
  }
}
```

**创建生成CA签名证书 csr 文件的 json 文件**

> CN是证书拥有者名字，一般为网站名或IP+端口，如www.baidu.com，OU组织机构名 O组织名 L城市 ST州或省 C国家代码

```json
[root@k8s-master1 src]$ cat ca-csr.json
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```

**生成 CA 证书 (ca.pem) 和这密钥 (ca-key.pem)**

```bash
[root@k8s-master1 src]$ cfssl gencert -initca ca-csr.json | cfssljson -bare ca
[root@k8s-master1 src]$ ll *.pem
-rw------- 1 root root 1675 Jul 11 21:27 ca-key.pem
-rw-r--r-- 1 root root 1359 Jul 11 21:27 ca.pem

[root@k8s-master1 src]$ cp ca.csr ca.pem ca-key.pem ca.csr.json ca-config.json /opt/kubernetes/ssl
[root@k8s-master1 src]$ ll /opt/kubernetes/ssl/
total 16
-rw-r--r-- 1 root root  290 Jul 11 21:29 ca-config.json
-rw-r--r-- 1 root root 1001 Jul 11 21:29 ca.csr.json
-rw------- 1 root root 1675 Jul 11 21:29 ca-key.pem
-rw-r--r-- 1 root root 1359 Jul 11 21:29 ca.pem

[root@k8s-master1 src]$ cat /root/ssh.sh
#!/bin/bash
IP="
192.168.100.102
192.168.100.103
192.168.100.104
192.168.100.105
192.168.100.106
192.168.100.107
192.168.100.108
192.168.100.109
192.168.100.110
192.168.100.111
"

for node in ${IP};do
  sshpass -p 123456 ssh-copy-id  -p22 ${node}  -o StrictHostKeyChecking=no
    if [ $? -eq 0 ];then
    echo "${node} 秘钥copy完成,准备环境初始化....."
$ ssh  -p22   ${node}  "test ! -d /etc/docker/certs.d/k8s-harbor1.example.com && mkdir /etc/docker/certs.d/k8s-harbor1.example.com -pv"
#      ssh  -p22   ${node}  "test ! -d /etc/docker/certs.d/k8s-harbor2.example.com && mkdir /etc/docker/certs.d/k8s-harbor2.example.com -pv"
#      echo "${node} Harbor 证书目录创建成功!"
#      scp -P22 /etc/docker/certs.d/k8s-harbor1.example.com/server.crt ${node}:/etc/docker/certs.d/k8s-harbor1.example.com/server.crt
#      scp -P22 /etc/docker/certs.d/k8s-harbor2.example.com/server.crt ${node}:/etc/docker/certs.d/k8s-harbor2.example.com/server.crt
#      echo "${node} Harbor 证书拷贝成功!"
#      scp -P22 /etc/hosts ${node}:/etc/hosts
#      echo "${node} host 文件拷贝完成"
#      scp -P22 /etc/sysctl.conf  ${node}:/etc/sysctl.conf
#      echo "${node} sysctl.conf 文件拷贝完成"
#      scp -P22 /etc/security/limits.conf  ${node}:/etc/security/limits.conf
#      echo "${node} limits.conf 文件拷贝完成"
#      scp -r -P22  /root/.docker  ${node}:/root/
#      echo "${node} Harbor 认证文件拷贝完成!"
#      scp -r -P22  /etc/resolv.conf  ${node}:/etc/
#      sleep 2
#      ssh  -p22   ${node}  "reboot"
#      sleep 2
        scp -r -P22 /opt/kubernetes/ssl/*  ${node}:/opt/kubernetes/ssl 
    else
    echo "${node} ssh-key copy error!"
    fi
done
```



### 1.5.5 etcd  集群部署

**各 etcd 服务器下载 etcd 安装包**

二进制包下载地址：<https://github.com/coreos/etcd/releases/tag/v3.2.18>

```bash
[root@k8s-etcd1 src]$ tar zxf etcd-v3.2.18-linux-amd64.tar.gz
[root@k8s-etcd1 src]$ cd etcd-v3.2.18-linux-amd64
[root@k8s-etcd1 etcd-v3.2.18-linux-amd64]$ cp etcdctl  etcd /opt/kubernetes/bin/
[root@k8s-etcd1 etcd-v3.2.18-linux-amd64]$ scp  /opt/kubernetes/bin/etcd* 192.168.100.106:/opt/kubernetes/bin/
[root@k8s-etcd1 etcd-v3.2.18-linux-amd64]$ scp  /opt/kubernetes/bin/etcd* 192.168.100.107:/opt/kubernetes/bin/
```

**在 master 创建 etcd 证书签名请求**

```json
root@k8s-master1:/usr/local/src/ssl/etcd$ cat etcd-csr.json
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "192.168.100.105",
    "192.168.100.106",
    "192.168.100.107"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```

**生成 etcd 证书和私钥**

```bash
root@k8s-master1:/usr/local/src/ssl/etcd$ pwd
/usr/local/src/ssl/etcd
root@k8s-master1:/usr/local/src/ssl/etcd$ cfssl gencert -ca=/opt/kubernetes/ssl/ca.pem \
   -ca-key=/opt/kubernetes/ssl/ca-key.pem \
   -config=/opt/kubernetes/ssl/ca-config.json \
   -profile=kubernetes etcd-csr.json | cfssljson -bare etcd
   
root@k8s-master1:/usr/local/src/ssl/etcd# ll
total 24
drwxr-xr-x 2 root root 4096 Jul 14 08:59 ./
drwxr-xr-x 4 root root 4096 Jul 14 08:56 ../
-rw-r--r-- 1 root root 1062 Jul 14 08:59 etcd.csr
-rw-r--r-- 1 root root  293 Jul 14 08:58 etcd-csr.json
-rw------- 1 root root 1679 Jul 14 08:59 etcd-key.pem
-rw-r--r-- 1 root root 1436 Jul 14 08:59 etcd.pem

#证书移动到/opt/kubernetes/ssl目录下
root@k8s-master1:/usr/local/src/ssl/etcd$ cp etcd*.pem /opt/kubernetes/ssl


#将证书复制到各etcd 服务器节点
root@k8s-master1:/usr/local/src/ssl/etcd$ bash /root/ssh-dir/ssh.sh 
```

**启动脚本**

```bash
vim /etc/systemd/system/etcd.service
[Unit]
Description=Etcd Server
After=network.target

[Service]
Type=simple
WorkingDirectory=/var/lib/etcd
EnvironmentFile=-/opt/kubernetes/cfg/etcd.conf
# set GOMAXPROCS to number of processors
ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /opt/kubernetes/bin/etcd"
Type=notify

[Install]
WantedBy=multi-user.target
```

**配置文件**

```bash
root@k8s-etcd1:/usr/local/src/etcd-v3.2.18-linux-amd64$ vi /opt/kubernetes/cfg/etcd.conf

#[member]
ETCD_NAME="etcd-node1"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
#ETCD_SNAPSHOT_COUNTER="10000"
#ETCD_HEARTBEAT_INTERVAL="100"
#ETCD_ELECTION_TIMEOUT="1000"
ETCD_LISTEN_PEER_URLS="https://192.168.100.105:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.100.105:2379,https://127.0.0.1:2379"
#ETCD_MAX_SNAPSHOTS="5"
#ETCD_MAX_WALS="5"
#ETCD_CORS=""
#[cluster]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.100.105:2380"
# if you use different ETCD_NAME (e.g. test),
# set ETCD_INITIAL_CLUSTER value for this name, i.e. "test=http://..."
ETCD_INITIAL_CLUSTER="etcd-node1=https://192.168.100.105:2380,etcd-node2=https://192.168.100.106:2380,etcd-node3=https://192.168.100.107:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="k8s-etcd-cluster"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.100.105:2379"
#[security]
CLIENT_CERT_AUTH="true"
ETCD_CA_FILE="/opt/kubernetes/ssl/ca.pem"
ETCD_CERT_FILE="/opt/kubernetes/ssl/etcd.pem"
ETCD_KEY_FILE="/opt/kubernetes/ssl/etcd-key.pem"
PEER_CLIENT_CERT_AUTH="true"
ETCD_PEER_CA_FILE="/opt/kubernetes/ssl/ca.pem"
ETCD_PEER_CERT_FILE="/opt/kubernetes/ssl/etcd.pem"
ETCD_PEER_KEY_FILE="/opt/kubernetes/ssl/etcd-key.pem"

[root@k8s-etcd1 src]$ scp /opt/kubernetes/cfg/etcd.conf  192.168.100.106:/opt/kubernetes/cfg/
[root@k8s-etcd1 src]$ scp /opt/kubernetes/cfg/etcd.conf  192.168.100.107:/opt/kubernetes/cfg/
```

**各服务器创建数据目录**

```bash
mkdir /var/lib/etcd
root@k8s-etcd1:/usr/local/src/etcd-v3.2.18-linux-amd64$ systemctl start etcd &&  systemctl enable etcd && systemctl  status etcd

#etcdctl命令软连接
root@k8s-etcd1:/usr/local/src/etcd-v3.2.18-linux-amd64$ ln -sv /opt/kubernetes/bin/etcdctl  /usr/bin/
```

**验证集群状态**

```bash
etcdctl --endpoints=https://192.168.100.105:2379 \
  --ca-file=/opt/kubernetes/ssl/ca.pem \
  --cert-file=/opt/kubernetes/ssl/etcd.pem \
  --key-file=/opt/kubernetes/ssl/etcd-key.pem cluster-health

etcdctl --endpoints=https://192.168.100.105:2379   --ca-file=/opt/kubernetes/ssl/ca.pem   --cert-file=/opt/kubernetes/ssl/etcd.pem   --key-file=/opt/kubernetes/ssl/etcd-key.pem member list
```



### 1.5.6 master 部署

```bash
[root@k8s-master1 ~]$ cd /usr/local/src/
[root@k8s-master1 src]$ tar xvf kubernetes-1.11.0-client-linux-amd64.tar.gz
[root@k8s-master1 src]$ tar xvf  kubernetes-1.11.0-node-linux-amd64.tar.gz
[root@k8s-master1 src]$ tar xvf  kubernetes-1.11.0-server-linux-amd64.tar.gz
[root@k8s-master1 src]$ tar xvf kubernetes-1.11.0.tar.gz

[root@k8s-master1 src]$ cp kubernetes/server/bin/kube-apiserver /opt/kubernetes/bin/
[root@k8s-master1 src]$ cp kubernetes/server/bin/kube-scheduler /usr/bin/
[root@k8s-master1 src]$ cp kubernetes/server/bin/kube-controller-manager  /usr/bin/
```

**创建生成 CSR 文件的 json 文件**

```json
[root@k8s-master1 src]# vim kubernetes-csr.json
{
  "CN": "kubernetes",
  "hosts": [
    "127.0.0.1",
    "10.1.0.1",
    "192.168.100.101",
    "192.168.100.102",
	"192.168.100.112",
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```

**生成证书**

```bash
[root@k8s-master1 src]$ cfssl gencert -ca=/opt/kubernetes/ssl/ca.pem -ca-key=/opt/kubernetes/ssl/ca-key.pem -config=/opt/kubernetes/ssl/ca-config.json  -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes

[root@k8s-master1 src]$ ll *.pem
-rw------- 1 root root 1675 Jul 11 22:24 kubernetes-key.pem
-rw-r--r-- 1 root root 1619 Jul 11 22:24 kubernetes.pem
```

**将证书复制到各服务器**

```bash
root@k8s-master1:/usr/local/src/ssl/master$ cp kubernetes*.pem /opt/kubernetes/ssl/
[root@k8s-master1 src]$ bash /root/ssh.sh
```

**创建 apiserver 使用的客户端 token 文件**

```bash
root@k8s-master1:/usr/local/src/ssl/master$ head -c 16 /dev/urandom | od -An -t x | tr -d ' '
9077bdc74eaffb83f672fe4c530af0d6
[root@k8s-master2 ~]# vim /opt/kubernetes/ssl/bootstrap-token.csv #各master服务器
9077bdc74eaffb83f672fe4c530af0d6,kubelet-bootstrap,10001,"system:kubelet-bootstrap"
```

**配置认证用户密码**

```bash
vim /opt/kubernetes/ssl/basic-auth.csv
admin,admin,1
readonly,readonly,2
```

**部署 api-server 启动脚本**

```bash
[root@k8s-master1 src]$ cat  /usr/lib/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
ExecStart=/opt/kubernetes/bin/kube-apiserver \
  --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,NodeRestriction \
  --bind-address=192.168.100.101 \
  --insecure-bind-address=127.0.0.1 \
  --authorization-mode=Node,RBAC \
  --runtime-config=rbac.authorization.k8s.io/v1 \
  --kubelet-https=true \
  --anonymous-auth=false \
  --basic-auth-file=/opt/kubernetes/ssl/basic-auth.csv \
  --enable-bootstrap-token-auth \
  --token-auth-file=/opt/kubernetes/ssl/bootstrap-token.csv \
  --service-cluster-ip-range=10.1.0.0/16 \
  --service-node-port-range=20000-40000 \
  --tls-cert-file=/opt/kubernetes/ssl/kubernetes.pem \
  --tls-private-key-file=/opt/kubernetes/ssl/kubernetes-key.pem \
  --client-ca-file=/opt/kubernetes/ssl/ca.pem \
  --service-account-key-file=/opt/kubernetes/ssl/ca-key.pem \
  --etcd-cafile=/opt/kubernetes/ssl/ca.pem \
  --etcd-certfile=/opt/kubernetes/ssl/kubernetes.pem \
  --etcd-keyfile=/opt/kubernetes/ssl/kubernetes-key.pem \
  --etcd-servers=https://192.168.100.105:2379,https://192.168.100.106:2379,https://192.168.100.107:2379 \
  --enable-swagger-ui=true \
  --allow-privileged=true \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=3 \
  --audit-log-maxsize=100 \
  --audit-log-path=/opt/kubernetes/log/api-audit.log \
  --event-ttl=1h \
  --v=2 \
  --logtostderr=false \
  --log-dir=/opt/kubernetes/log
Restart=on-failure
RestartSec=5
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
#将启动脚本复制到server，更改--bind-address 为server2 IP地址，然后重启server 2的api-server并验证
```

**启动并验证 api-server**

```bash
[root@k8s-master1 src]$ systemctl daemon-reload && systemctl enable kube-apiserver && systemctl start kube-apiserver && systemctl status  kube-apiserver
```

**配置 Controller Manager 服务**

```bash
[root@k8s-master1 src]$ cat /usr/lib/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/opt/kubernetes/bin/kube-controller-manager \
  --address=127.0.0.1 \
  --master=http://127.0.0.1:8080 \
  --allocate-node-cidrs=true \
  --service-cluster-ip-range=10.1.0.0/16 \
  --cluster-cidr=10.2.0.0/16 \
  --cluster-name=kubernetes \
  --cluster-signing-cert-file=/opt/kubernetes/ssl/ca.pem \
  --cluster-signing-key-file=/opt/kubernetes/ssl/ca-key.pem \
  --service-account-private-key-file=/opt/kubernetes/ssl/ca-key.pem \
  --root-ca-file=/opt/kubernetes/ssl/ca.pem \
  --leader-elect=true \
  --v=2 \
  --logtostderr=false \
  --log-dir=/opt/kubernetes/log

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

**复制并启动二进制文件**

```bash
[root@k8s-master1 src]$ cp kubernetes/server/bin/kube-controller-manager /opt/kubernetes/bin/
```

**将启动文件和启动脚本 scp 到 master2 并启动服务和验证**

```bash
[root@k8s-master1 src]$ systemctl restart kube-controller-manager &&  systemctl status  kube-controller-manager
```

**部署 Kubernetes Scheduler**

```bash
[root@k8s-master1 src]$ vim /lib/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/opt/kubernetes/bin/kube-scheduler \
  --address=127.0.0.1 \
  --master=http://127.0.0.1:8080 \
  --leader-elect=true \
  --v=2 \
  --logtostderr=false \
  --log-dir=/opt/kubernetes/log

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

**准备启动二进制 Kubernetes Scheduler 文件**

```bash
[root@k8s-master1 src]$ cp kubernetes/server/bin/kube-scheduler  /opt/kubernetes/bin/
[root@k8s-master1 src]$ scp /opt/kubernetes/bin/kube-scheduler  192.168.100.102:/opt/kubernetes/bin/
#启动并验证服务
```

**部署 kubectl 命令行工具**

```bash
[root@k8s-master1 src]$ cp kubernetes/client/bin/kubectl  /opt/kubernetes/bin/
[root@k8s-master1 src]$ scp /opt/kubernetes/bin/kubectl  192.168.100.102:/opt/kubernetes/bin/
[root@k8s-master1 src]$ ln -sv /opt/kubernetes/bin/kubectl  /usr/bin/
```

**创建 admin 证书签名请求**

```json
vim admin-csr.json
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
```

**生成 admin 证书和私钥**

```bash
[root@k8s-master1 src]$ cfssl gencert -ca=/opt/kubernetes/ssl/ca.pem -ca-key=/opt/kubernetes/ssl/ca-key.pem -config=/opt/kubernetes/ssl/ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin

[root@k8s-master1 src]$ ll admin*
-rw-r--r-- 1 root root 1009 Jul 11 22:51 admin.csr
-rw-r--r-- 1 root root  229 Jul 11 22:50 admin-csr.json
-rw------- 1 root root 1679 Jul 11 22:51 admin-key.pem
-rw-r--r-- 1 root root 1399 Jul 11 22:51 admin.pem


[root@k8s-master1 src]$ cp admin*.pem /opt/kubernetes/ssl/
```

**设置集群参数**

```bash
master1：
kubectl config set-cluster kubernetes \
   --certificate-authority=/opt/kubernetes/ssl/ca.pem \
   --embed-certs=true \
   --server=https://192.168.100.101:6443

master2：
kubectl config set-cluster kubernetes \
   --certificate-authority=/opt/kubernetes/ssl/ca.pem \
   --embed-certs=true \
   --server=https://192.168.100.112:6443
```

**设置客户端认证参数**

```bash
[root@k8s-master1 src]$ kubectl config set-credentials admin \
    --client-certificate=/opt/kubernetes/ssl/admin.pem \
    --embed-certs=true \
    --client-key=/opt/kubernetes/ssl/admin-key.pem
User "admin" set.

[root@k8s-master2 src]$ kubectl config set-credentials admin \
    --client-certificate=/opt/kubernetes/ssl/admin.pem \
    --embed-certs=true \
    --client-key=/opt/kubernetes/ssl/admin-key.pem
User "admin" set.
```

**设置上下文参数**

```bash
[root@k8s-master1 src]$ kubectl config set-context kubernetes \
    --cluster=kubernetes  --user=admin
Context "kubernetes" created.

[root@k8s-master2 src]$ kubectl config set-context kubernetes --cluster=kubernetes --user=admin
Context "kubernetes" created.
```

**设置默认上下文**

```bash
[root@k8s-master2 src]$ kubectl config use-context kubernetes
Switched to context "kubernetes".

[root@k8s-master1 src]$ kubectl config use-context kubernetes
Switched to context "kubernetes".
```



### 1.5.7 node 部署

**各 node 节点安装基础命令**

```bash
$ apt-get install ipvsadm ipset conntrack
```

**准备二进制包**

```bash
[root@k8s-master1 src]$ scp kubernetes/server/bin/kube-proxy  kubernetes/server/bin/kubelet  192.168.100.107:/opt/kubernetes/bin/
[root@k8s-master1 src]$ scp kubernetes/server/bin/kube-proxy kubernetes/server/bin/kubelet  192.168.100.108:/opt/kubernetes/bin/
```

**角色绑定**

```bash
[root@k8s-master1 src]$ kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap
```

**创建 kubelet bootstrapping kubeconfig 文件并设置集群参数**

```bash
[root@k8s-master1 src]$ kubectl config set-cluster kubernetes \
   --certificate-authority=/opt/kubernetes/ssl/ca.pem \
   --embed-certs=true \
   --server=https://192.168.100.112:6443 \
   --kubeconfig=bootstrap.kubeconfig
Cluster "kubernetes" set.
```

**设置客户端认证参数**

```bash
 [root@k8s-master1 src]$ kubectl config set-credentials kubelet-bootstrap \
   --token=9077bdc74eaffb83f672fe4c530af0d6 \
   --kubeconfig=bootstrap.kubeconfig   
```

**设置上下文**

```bash
[root@k8s-master1 src]$ kubectl config set-context default \
   --cluster=kubernetes \
   --user=kubelet-bootstrap \
   --kubeconfig=bootstrap.kubeconfig
```

**选择默认上下文**

```bash
[root@k8s-master1 src]$ kubectl config use-context default --kubeconfig=bootstrap.kubeconfig
Switched to context "default"
[root@k8s-master1 src]$ scp bootstrap.kubeconfig  192.168.100.108:/opt/kubernetes/cfg/ #自动生成文件
[root@k8s-master1 src]$ scp bootstrap.kubeconfig  192.168.100.109:/opt/kubernetes/cfg/
```

**部署 kubelet**

设置 CNI 支持

```bash
#node节点：
 mkdir -p /etc/cni/net.d
 [root@k8s-node1 ~]# cat  /etc/cni/net.d/10-default.conf
{
        "name": "flannel",
        "type": "flannel",
        "delegate": {
            "bridge": "docker0",
            "isDefaultGateway": true,
            "mtu": 1400
        }
}

mkdir /var/lib/kubelet
```

**创建 kubelet 服务配置，每个 node**

```bash
[root@k8s-node2 ~]$ vim /lib/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
ExecStart=/opt/kubernetes/bin/kubelet \
  --address=192.168.100.108 \
  --hostname-override=192.168.100.108 \
  --pod-infra-container-image=mirrorgooglecontainers/pause-amd64:3.0 \
  --experimental-bootstrap-kubeconfig=/opt/kubernetes/cfg/bootstrap.kubeconfig \
  --kubeconfig=/opt/kubernetes/cfg/kubelet.kubeconfig \
  --cert-dir=/opt/kubernetes/ssl \
  --network-plugin=cni \
  --cni-conf-dir=/etc/cni/net.d \
  --cni-bin-dir=/opt/kubernetes/bin/cni \
  --cluster-dns=10.1.0.1 \
  --cluster-domain=cluster.local. \
  --hairpin-mode hairpin-veth \
  --allow-privileged=true \
  --fail-swap-on=false \
  --logtostderr=true \
  --v=2 \
  --logtostderr=false \
  --log-dir=/opt/kubernetes/log
Restart=on-failure
RestartSec=5

systemctl daemon-reload && systemctl enable kubelet && systemctl start kubelet && systemctl  status kubelet
```

**master 查看 csr 请求**

```bash
[root@k8s-master2 src]$ kubectl get csr
NAME                                                   AGE       REQUESTOR           CONDITION
node-csr-dD84BwWWLl43SeiB5G1PnUaee5Sv60RsoVZFkuuePg0   1m        kubelet-bootstrap   Pending
node-csr-vW4eBZb98z-DvAeG9q8hb9mOAUg0U9HSML9YRBscP8A   2m        kubelet-bootstrap   Pending
```

**master 批准 TLS 请求**

```bash
#执行完毕后，查看节点状态已经是Ready的状态了
[root@k8s-master2 src]# 
certificatesigningrequest.certificates.k8s.io/node-csr-dD84BwWWLl43SeiB5G1PnUaee5Sv60RsoVZFkuuePg0 approved
certificatesigningrequest.certificates.k8s.io/node-csr-vW4eBZb98z-DvAeG9q8hb9mOAUg0U9HSML9YRBscP8A approved

[root@k8s-master2 src]$ kubectl get nodes
NAME              STATUS    ROLES     AGE       VERSION
192.168.100.108   Ready     <none>    32s       v1.11.0
192.168.100.109   Ready     <none>    32s       v1.11.0
```

**node 节点部署 Kubernetes Proxy**

```bash
[root@k8s-node1 ~]$ yum install -y ipvsadm ipset conntrack*
[root@k8s-node2 ~]$ yum install -y ipvsadm ipset conntrack*
```

**创建 kube-proxy 证书请求**

```json
[root@k8s-master1 src]$ vim kube-proxy-csr.json
{
  "CN": "system:kube-proxy",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```

**生成证书**

```bash
[root@k8s-master1 src]$ cfssl gencert -ca=/opt/kubernetes/ssl/ca.pem -ca-key=/opt/kubernetes/ssl/ca-key.pem    -config=/opt/kubernetes/ssl/ca-config.json -profile=kubernetes  kube-proxy-csr.json | cfssljson -bare kube-proxy
```

**复制证书到各 node 节点**

```bash
[root@k8s-master1 src]$ cp kube-proxy*.pem /opt/kubernetes/ssl/
[root@k8s-master1 src]$ bash /root/ssh.sh
```

**创建 kube-proxy 配置文件**

```bash
[root@k8s-master1 src]$ kubectl config set-cluster kubernetes \
    --certificate-authority=/opt/kubernetes/ssl/ca.pem \
   --embed-certs=true \
   --server=https://192.168.100.101:6443 \
   --kubeconfig=kube-proxy.kubeconfig
Cluster "kubernetes" set.

[root@k8s-master1 src]$ kubectl config set-credentials kube-proxy \
    --client-certificate=/opt/kubernetes/ssl/kube-proxy.pem \
    --client-key=/opt/kubernetes/ssl/kube-proxy-key.pem \
   --embed-certs=true \
   --kubeconfig=kube-proxy.kubeconfig
User "kube-proxy" set.

[root@k8s-master1 src]$ kubectl config set-context default \
    --cluster=kubernetes \
    --user=kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig
    
[root@k8s-master1 src]$ kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
Switched to context "default".
```

**分发 kubeconfig 配置文件**

```bash
[root@k8s-master1 src]$ scp kube-proxy.kubeconfig  192.168.100.108:/opt/kubernetes/cfg/
[root@k8s-master1 src]$ scp kube-proxy.kubeconfig  192.168.100.109:/opt/kubernetes/cfg/
```

**创建 kube-proxy 服务配置**

```bash
[root@k8s-node1 ~]$ mkdir /var/lib/kube-proxy
[root@k8s-node2 ~]$ mkdir /var/lib/kube-proxy

[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/opt/kubernetes/bin/kube-proxy \
  --bind-address=192.168.100.109 \
  --hostname-override=192.168.100.109 \
  --kubeconfig=/opt/kubernetes/cfg/kube-proxy.kubeconfig \
   --masquerade-all \
  --feature-gates=SupportIPVSProxyMode=true \
  --proxy-mode=ipvs \
  --ipvs-min-sync-period=5s \
  --ipvs-sync-period=5s \
  --ipvs-scheduler=rr \
  --logtostderr=true \
  --v=2 \
  --logtostderr=false \
  --log-dir=/opt/kubernetes/log

Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

**重启并验证服务**

``` bash
ipvsadm ipset conntrack
[root@k8s-node1 ~]$ systemctl daemon-reload && systemctl enable kube-proxy && systemctl start kube-proxy && systemctl status kube-proxy
[root@k8s-node2 ~]$ systemctl daemon-reload && systemctl enable kube-proxy && systemctl start kube-proxy && systemctl status kube-proxy

root@k8s-node1:/opt/kubernetes/cfg$ ipvsadm -L -n
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.1.0.1:443 rr
  -> 192.168.100.101:6443         Masq    1      0          0         
  -> 192.168.100.102:6443         Masq    1      0          0 
```



### 1.5.8 部署 flannel

在 master 各节点和各 node 节点部署 flannel 服务

**生成证书申请 csr 文件**

```json
[root@k8s-master1 src]$ vim flanneld-csr.json
{
  "CN": "flanneld",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```

**生成证书**

```bash
cfssl gencert -ca=/opt/kubernetes/ssl/ca.pem  -ca-key=/opt/kubernetes/ssl/ca-key.pem -config=/opt/kubernetes/ssl/ca-config.json -profile=kubernetes flanneld-csr.json | cfssljson -bare flanneld

cp  flanneld*.pem /opt/kubernetes/ssl/
scp flanneld*.pem 192.168.100.108:/opt/kubernetes/ssl/
scp flanneld*.pem 192.168.100.109:/opt/kubernetes/ssl/

root@k8s-master1:/usr/local/src$ tar xvf flannel-v0.10.0-linux-amd64.tar.gz
root@k8s-master1:/usr/local/src$ scp -P22 flanneld  192.168.100.108:/opt/kubernetes/bin/
root@k8s-master1:/usr/local/src$ scp -P22 flanneld  192.168.100.109:/opt/kubernetes/bin/
```

**复制对应脚本到 /opt/kubernetes/bin 目录**

```bash
[root@k8s-master1 src]$ cd /usr/local/src/kubernetes/cluster/centos/node/bin/
[root@k8s-master1 bin]$ cp remove-docker0.sh /opt/kubernetes/bin/

[root@k8s-master1:/usr/local/src/kubernetes/cluster/centos/node/bin]$ scp remove-docker0.sh  mk-docker-opts.sh  192.168.100.108:/opt/kubernetes/bin/
[root@k8s-master1:/usr/local/src/kubernetes/cluster/centos/node/bin]$ scp remove-docker0.sh  mk-docker-opts.sh  192.168.100.109:/opt/kubernetes/bin/
```

**配置 Flannel**

```bash
[root@k8s-master1 bin]$ vim /opt/kubernetes/cfg/flannel
FLANNEL_ETCD="-etcd-endpoints=https://192.168.100.105:2379,https://192.168.100.106:2379,https://192.168.100.107:2379"
FLANNEL_ETCD_KEY="-etcd-prefix=/kubernetes/network"
FLANNEL_ETCD_CAFILE="--etcd-cafile=/opt/kubernetes/ssl/ca.pem"
FLANNEL_ETCD_CERTFILE="--etcd-certfile=/opt/kubernetes/ssl/flanneld.pem"
FLANNEL_ETCD_KEYFILE="--etcd-keyfile=/opt/kubernetes/ssl/flanneld-key.pem"
```

**复制配置到其它节点上**

```bash
[root@k8s-master1:/usr/local/src/ssl/flannel]$ vim /lib/systemd/system/flannel.service
[Unit]
Description=Flanneld overlay address etcd agent
After=network.target
Before=docker.service

[Service]
EnvironmentFile=-/opt/kubernetes/cfg/flannel
ExecStartPre=/opt/kubernetes/bin/remove-docker0.sh
ExecStart=/opt/kubernetes/bin/flanneld ${FLANNEL_ETCD} ${FLANNEL_ETCD_KEY} ${FLANNEL_ETCD_CAFILE} ${FLANNEL_ETCD_CERTFILE} ${FLANNEL_ETCD_KEYFILE}
ExecStartPost=/opt/kubernetes/bin/mk-docker-opts.sh -d /run/flannel/docker

Type=notify

[Install]
WantedBy=multi-user.target
RequiredBy=docker.service
```

**复制系统服务脚本到其它节点**

```bash
$ scp /lib/systemd/system/flannel.service 192.168.100.108:/lib/systemd/system/flannel.service
$ scp /lib/systemd/system/flannel.service 192.168.100.109:/lib/systemd/system/flannel.service
```

**Flannel CNI 集成**

```bash
[root@k8s-master1 src]$ mkdir /opt/kubernetes/bin/cni
[root@k8s-master1 src]$ tar zxf cni-plugins-amd64-v0.7.1.tgz -C /opt/kubernetes/bin/cni
[root@k8s-master1:/usr/local/src]$ scp -r  /opt/kubernetes/bin/cni 192.168.100.108:/opt/kubernetes/bin/
[root@k8s-master1:/usr/local/src]$ scp -r  /opt/kubernetes/bin/cni 192.168.100.109:/opt/kubernetes/bin/
```

**在 etcd 创建网络**

```bash
#提前将证书复制到etcd或在node节点操作：
[root@k8s-master1:/usr/local/src/ssl/flannel]$ /opt/kubernetes/bin/etcdctl --ca-file /opt/kubernetes/ssl/ca.pem --cert-file /opt/kubernetes/ssl/flanneld.pem --key-file    /opt/kubernetes/ssl/flanneld-key.pem  --no-sync -C https://192.168.100.105:2379,https://192.168.100.106:2379,https://192.168.100.107:2379  mk /kubernetes/network/config  '{ "Network": "10.2.0.0/16", "Backend": { "Type": "vxlan", "VNI": 1 }}' 

#验证网段：
[root@k8s-etcd1 ~]$ /opt/kubernetes/bin/etcdctl --ca-file /opt/kubernetes/ssl/ca.pem --cert-file /opt/kubernetes/ssl/flanneld.pem --key-file /opt/kubernetes/ssl/flanneld-key.pem --no-sync -C  https://192.168.100.107:2379 get  /kubernetes/network/config   
#以下是返回值
{ "Network": "10.2.0.0/16", "Backend": { "Type": "vxlan", "VNI": 1 }}

#启动flannel：
[root@k8s-node2 ~]$ systemctl daemon-reload && systemctl enable flannel && chmod +x /opt/kubernetes/bin/* &&  systemctl start flannel &&  systemctl status  flannel

#验证网段：
[root@k8s-node2~]$ ifconfig
flannel.1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        inet 10.2.32.0  netmask 255.255.255.255  broadcast 0.0.0.0
        inet6 fe80::accd:61ff:fe52:8805  prefixlen 64  scopeid 0x20<link>
        ether ae:cd:61:52:88:05  txqueuelen 0  (Ethernet)
        RX packets 7  bytes 588 (588.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 7  bytes 588 (588.0 B)
        TX errors 0  dropped 12 overruns 0  carrier 0  collisions 0
```

**配置 Docker 服务使用 Flannel**

```bash
[root@k8s-node1 ~]$ vim /usr/lib/systemd/system/docker.service
[Unit]
After=network-online.target firewalld.service flannel.service
Wants=network-online.target
Requires=flannel.service

[Service]
Type=notify
EnvironmentFile=-/run/flannel/docker
ExecStart=/usr/bin/dockerd $DOCKER_OPTS
```

**启动脚本内容**

```bash
root@k8s-node2:~$ cat /lib/systemd/system/docker.service 
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service flannel.service
Wants=network-online.target
Requires=flannel.service

[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
EnvironmentFile=-/run/flannel/docker
ExecStart=/usr/bin/dockerd  $DOCKER_OPTS  -H fd://
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=1048576
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity
# Uncomment TasksMax if your systemd version supports it.
# Only systemd 226 and above support this version.
TasksMax=infinity
TimeoutStartSec=0
# set delegate yes so that systemd does not reset the cgroups of docker containers
Delegate=yes
# kill only the docker process, not all processes in the cgroup
KillMode=process
# restart the docker process if it exits prematurely
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s

[Install]
WantedBy=multi-user.target
```

**分发至各个 node 节点**

```bash
[root@k8s-node1 ~]$ scp /lib/systemd/system/docker.service   192.168.100.108:/lib/systemd/system/docker.service 
[root@k8s-node1 ~]$ scp /lib/systemd/system/docker.service   192.168.100.109:/lib/systemd/system/docker.service 
```

**测试创建应用**

```bash
root@k8s-master1:~$ kubectl run net-test --image=alpine --replicas=2 sleep 360000

#验证容器启动成功：
root@k8s-master1:~$ kubectl get pods
NAME                      READY     STATUS    RESTARTS   AGE
net-test-ff75d6c6-5tj88   1/1       Running   0          10m
net-test-ff75d6c6-fsftw   1/1       Running   0          10m

#在master节点进入容器：
root@k8s-master1:~$ kubectl exec -it net-test-ff75d6c6-fsftw sh
/ #

#在node节点验证进入容器：
 root@k8s-node1:~$ docker exec -it 5418e1f1ae17 sh
/ # 
```



### 1.5.9 部署 coreDNS

```bash
[root@k8s-master1:/usr/local/src/kubernetes/cluster/addons/dns/coredns]$ docker tag gcr.io/google-containers/coredns:1.1.3  k8s-harbor1.example.com/baseimages/coredns:1.1.3
[root@k8s-master1:/usr/local/src/kubernetes/cluster/addons/dns/coredns]$ docker push k8s-harbor1.example.com/baseimages/coredns:1.1.3

cp coredns.yaml.base  coredns.yaml

vim coredns.yaml #更改镜像地址和ClusterIP

kubectl create -f coredns.yaml

kubectl get pods --all-namespaces
kubectl get services -n kube-system
   
#DNS测试
[root@k8s-master1:/usr/local/src/kubernetes/cluster/addons/dns/coredns]$ docker tag gcr.io/google-containers/busybox  k8s-harbor1.example.com/baseimages/busybox
[root@k8s-master1:/usr/local/src/kubernetes/cluster/addons/dns/coredns]$ docker push k8s-harbor1.example.com/baseimages/busybox
```

```yaml
#yaml文件
[root@k8s-master1]$ cat busybox.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: default
spec:
  containers:
  - image: k8s-harbor1.example.com/baseimages/busybox 
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
    name: busybox
  restartPolicy: Always
```



**创建并验证 pod**

```bash
[root@k8s-master1:/usr/local/src/kubernetes/cluster/addons/dns/coredns]$ kubectl create -f busybox.yaml
[root@k8s-master1:/usr/local/src/kubernetes/cluster/addons/dns/coredns]$ kubectl  get pods

#验证CorsDNS解析域名： 不在一个namespace的域名要写域名的全称，在一个namespace的域名只要写service名称即可解析
[root@k8s-master1:/usr/local/src/kubernetes/cluster/addons/dns/coredns]$ kubectl  get  services --all-namespaces
NAMESPACE     NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
default       kubernetes   ClusterIP   10.1.0.1     <none>        443/TCP         4d
kube-system   kube-dns     ClusterIP   10.1.0.254   <none>        53/UDP,53/TCP   11m

#域名格式： service名称 +  namespace名称.后缀
[root@k8s-master1:/usr/local/src/kubernetes/cluster/addons/dns/coredns]$ kubectl exec busybox nslookup kubernetes
Server:    10.1.0.254
Address 1: 10.1.0.254 kube-dns.kube-system.svc.cluster.local

nslookup: can't resolve 'kubernetes'
command terminated with exit code 1

[root@k8s-master1:/usr/local/src/kubernetes/cluster/addons/dns/coredns]$ kubectl exec busybox nslookup kubernetes.default.svc.cluster.local
Server:    10.1.0.254
Address 1: 10.1.0.254

Name:      kubernetes.default.svc.cluster.local
Address 1: 10.1.0.1 kubernetes.default.svc.cluster.local

https://192.168.100.102:6443/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy
```



### 1.5.10 部署 kubernetes 图形管理 dashboard

将访问账号名 admin 与 cluster-admin 关联以获得访问权限

````bash
$ kubectl create clusterrolebinding login-dashboard-admin --clusterrole=cluster-admin --user=admin
````

**更改 image 镜像地址**

```bash
[root@k8s-master1:/usr/local/src/kubernetes/cluster/addons/dashboard]$ pwd
/usr/local/src/kubernetes/cluster/addons/dashboard-1.8.2
[root@k8s-master1:/usr/local/src/kubernetes/cluster/addons/dashboard]$ kubectl  create  -f  . 
serviceaccount/kubernetes-dashboard created
clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
deployment.extensions/kubernetes-dashboard created
service/kubernetes-dashboard created
clusterrole.rbac.authorization.k8s.io/ui-admin created
rolebinding.rbac.authorization.k8s.io/ui-admin-binding created
clusterrole.rbac.authorization.k8s.io/ui-read created
rolebinding.rbac.authorization.k8s.io/ui-read-binding created

#浏览器访问测试：
https://192.168.100.101:6443/api/v1/namespaces/kube-system/services/http:kubernetes-dashboard:/proxy/
```



### 1.5.11 heapster

```bash
[root@k8s-master1:/usr/local/src/kubernetes/heapster]$ docker load -i heapster-amd64_v1.5.1.tar
[root@k8s-master1:/usr/local/src/kubernetes/heapster]$ docker tag gcr.io/google-containers/heapster-amd64:v1.5.1  k8s-harbor1.example.com/baseimages/eapster-amd64:v1.5.1
[root@k8s-master1:/usr/local/src/kubernetes/heapster]$ ocker push k8s-harbor1.example.com/baseimages/heapster-amd64:v1.5.1
root@k8s-master1:/usr/local/src/kubernetes/heapster$ vim heapster.yaml #更改image地址
 
[root@k8s-master1:/usr/local/src/kubernetes/heapster]$ docker load -i heapster-grafana-amd64-v4.4.3.tar
[root@k8s-master1:/usr/local/src/kubernetes/heapster]$ docker tag 8cb3de219af7 k8s-harbor1.example.com/baseimages/heapster-grafana-amd64:v4.4.3
[root@k8s-master1:/usr/local/src/kubernetes/heapster]$ docker push k8s-harbor1.example.com/baseimages/heapster-grafana-amd64:v4.4.3
[root@k8s-master1:/usr/local/src/kubernetes/heapster]$ vim heapster.yaml  
 
 
[root@k8s-master1:/usr/local/src/kubernetes/heapster]$ docker load -i heapster-influxdb-amd64_v1.3.3.tar 
[root@k8s-master1:/usr/local/src/kubernetes/heapster]$ docker tag gcr.io/google-containers/heapster-influxdb-amd64:v1.3.3 k8s-harbor1.example.com/baseimages/heapster-influxdb-amd64:v1.3.3
[root@k8s-master1:/usr/local/src/kubernetes/heapster]$ vim influxdb.yaml

[root@k8s-master1:/usr/local/src/kubernetes/heapster]$ kubectl  create -f .
deployment.extensions/monitoring-grafana created
service/monitoring-grafana created
serviceaccount/heapster created
clusterrolebinding.rbac.authorization.k8s.io/heapster created
deployment.extensions/heapster created
service/heapster created
deployment.extensions/monitoring-influxdb created
service/monitoring-influxdb created
```





## 1.6 ansbible 部署

### 1.6.1 基础环境准备

```bash
apt-get install python2.7
ln -s /usr/bin/python2.7 /usr/bin/python
apt-get install git ansible -y
ssh-keygen #生成密钥对
apt-get install sshpass #ssh同步公钥到各k8s服务器
#分发公钥脚本：
root@k8s-master1:~$ cat scp.sh
#!/bin/bash
#目标主机列表
IP="
192.168.7.101
192.168.7.102
192.168.7.103
192.168.7.104
192.168.7.105
192.168.7.106
192.168.7.107
192.168.7.108
192.168.7.109
192.168.7.110
192.168.7.111
"
for node in ${IP};do
  sshpass -p 123456 ssh-copy-id ${node} -o StrictHostKeyChecking=no
  if [ $? -eq 0 ];then
    echo "${node} 秘钥copy完成"
  else
    echo "${node} 秘钥copy失败"
  fi
done

#同步docker证书脚本：
#!/bin/bash
#目标主机列表
IP="
192.168.7.101
192.168.7.102
192.168.7.103
192.168.7.104
192.168.7.105
192.168.7.106
192.168.7.107
192.168.7.108
192.168.7.109
192.168.7.110
192.168.7.111
"
for node in ${IP};do
  sshpass -p 123456 ssh-copy-id ${node} -o StrictHostKeyChecking=no
  if [ $? -eq 0 ];then
 	echo "${node} 秘钥copy完成"
 	echo "${node} 秘钥copy完成,准备环境初始化....."
  	ssh ${node} "mkdir /etc/docker/certs.d/harbor.magedu.net -p"
  	echo "Harbor 证书目录创建成功!"
  	scp /etc/docker/certs.d/harbor.magedu.net/harbor-ca.crt
${node}:/etc/docker/certs.d/harbor.magedu.net/harbor-ca.crt
  echo "Harbor 证书拷贝成功!"
  scp  /etc/hosts ${node}:/etc/hosts
  echo "host 文件拷贝完成"
  scp -r /root/.docker ${node}:/root/
  echo "Harbor 认证文件拷贝完成!"
  scp -r /etc/resolv.conf ${node}:/etc/
else
 echo "${node} 秘钥copy失败"
fi
done
#执行脚本同步：
k8s-master1:~# bash scp.sh
root@s2:~# vim ~/.vimrc #取消vim 自动缩进功能
set paste
```



### 1.6.2 clone 项目

```bash
$ git clone -b 0.6.1 https://github.com/easzlab/kubeasz.git
root@k8s-master1:~$ mv /etc/ansible/* /opt/
root@k8s-master1:~$ mv kubeasz/* /etc/ansible/
root@k8s-master1:~$ cd /etc/ansible/
root@k8s-master1:/etc/ansible$ cp example/hosts.m-masters.example ./hosts #复制hosts模板文件
```



### 1.6.3 准备 hosts 文件

```bash
root@k8s-master1:/etc/ansible$ pwd
/etc/ansible
root@k8s-master1:/etc/ansible$ cp example/hosts.m-masters.example ./hosts
root@k8s-master1:/etc/ansible$ cat hosts
# 集群部署节点：一般为运行ansible 脚本的节点
# 变量 NTP_ENABLED (=yes/no) 设置集群是否安装 chrony 时间同步
[deploy]
192.168.7.101 NTP_ENABLED=no
# etcd集群请提供如下NODE_NAME，注意etcd集群必须是1,3,5,7...奇数个节点
[etcd]
192.168.7.105 NODE_NAME=etcd1
192.168.7.106 NODE_NAME=etcd2
192.168.7.107 NODE_NAME=etcd3
[new-etcd] # 预留组，后续添加etcd节点使用
#192.168.7.x NODE_NAME=etcdx
[kube-master]
192.168.7.101
[new-master] # 预留组，后续添加master节点使用
#192.168.7.5
[kube-node]
192.168.7.110
[new-node] # 预留组，后续添加node节点使用
#192.168.7.xx
# 参数 NEW_INSTALL：yes表示新建，no表示使用已有harbor服务器
# 如果不使用域名，可以设置 HARBOR_DOMAIN=""
[harbor]
#192.168.7.8 HARBOR_DOMAIN="harbor.yourdomain.com" NEW_INSTALL=no
# 负载均衡(目前已支持多于2节点，一般2节点就够了) 安装 haproxy+keepalived
[lb]
192.168.7.1 LB_ROLE=backup
192.168.7.2 LB_ROLE=master
#【可选】外部负载均衡，用于自有环境负载转发 NodePort 暴露的服务等
[ex-lb]
#192.168.7.6 LB_ROLE=backup EX_VIP=192.168.7.250
#192.168.7.7 LB_ROLE=master EX_VIP=192.168.7.250
[all:vars]
# ---------集群主要参数---------------
#集群部署模式：allinone, single-master, multi-master
DEPLOY_MODE=multi-master
#集群主版本号，目前支持: v1.8, v1.9, v1.10，v1.11, v1.12, v1.13
K8S_VER="v1.13"
# 集群 MASTER IP即 LB节点VIP地址，为区别与默认apiserver端口，设置VIP监听的服务端口8443
# 公有云上请使用云负载均衡内网地址和监听端口
MASTER_IP="192.168.7.248"
KUBE_APISERVER="https://{{ MASTER_IP }}:6443"
# 集群网络插件，目前支持calico, flannel, kube-router, cilium
CLUSTER_NETWORK="calico"
# 服务网段 (Service CIDR），注意不要与内网已有网段冲突
SERVICE_CIDR="10.20.0.0/16"
# POD 网段 (Cluster CIDR），注意不要与内网已有网段冲突
CLUSTER_CIDR="172.31.0.0/16"
# 服务端口范围 (NodePort Range)
NODE_PORT_RANGE="20000-60000"
# kubernetes 服务 IP (预分配，一般是 SERVICE_CIDR 中第一个IP)
CLUSTER_KUBERNETES_SVC_IP="10.20.0.1"
# 集群 DNS 服务 IP (从 SERVICE_CIDR 中预分配)
CLUSTER_DNS_SVC_IP="10.20.254.254"
# 集群 DNS 域名
CLUSTER_DNS_DOMAIN="linux36.local."
# 集群basic auth 使用的用户名和密码
BASIC_AUTH_USER="admin"
BASIC_AUTH_PASS="123456"
# ---------附加参数--------------------
#默认二进制文件目录
bin_dir="/usr/bin"
#证书目录
ca_dir="/etc/kubernetes/ssl"
#部署目录，即 ansible 工作目录，建议不要修改
base_dir="/etc/ansible"
```



### 1.6.4 准备二进制文件

```bash
k8s-master1:/etc/ansible/bin$ pwd
/etc/ansible/bin
k8s-master1:/etc/ansible/bin$ tar xvf k8s.1-13-5.tar.gz
k8s-master1:/etc/ansible/bin$ mv bin/* .
```



### 1.6.5 开始按步骤部署

通过 ansible 脚本初始化环境及部署 k8s 高可用集群



#### 1.6.5.1 环境初始化

```bash
root@k8s-master1:/etc/ansible$ pwd
/etc/ansible
root@k8s-master1:/etc/ansible$ ansible-playbook 01.prepare.yml
```



#### 1.6.5.2 部署 etcd 集群

可选更改启动脚本路径

```bash
root@k8s-master1:/etc/ansible$ ansible-playbook 02.etcd.yml
```

各 etcd 服务器验证服务如下：

```bash
root@k8s-etcd1:~$ export NODE_IPS="192.168.7.105 192.168.7.106 192.168.7.107"
root@k8s-etcd1:~$ for ip in ${NODE_IPS}; do  ETCDCTL_API=3 /usr/bin/etcdctl  --
endpoints=https://${ip}:2379  --cacert=/etc/kubernetes/ssl/ca.pem  --
cert=/etc/etcd/ssl/etcd.pem  --key=/etc/etcd/ssl/etcd-key.pem  endpoint health;
done
https://192.168.7.105:2379 is healthy: successfully committed proposal: took =
2.198515ms
https://192.168.7.106:2379 is healthy: successfully committed proposal: took =
2.457971ms
https://192.168.7.107:2379 is healthy: successfully committed proposal: took =
1.859514ms
```



#### 1.6.5.3 部署 docker

可选更改启动脚本路径，但是 docker 已经提前安装，因此不需要重新执行

```bash
root@k8s-master1:/etc/ansible$ ansible-playbook 03.docker.yml
```



#### 1.6.5.4  部署 master

可选更改启动脚本路径

```bash
root@k8s-master1:/etc/ansible$ ansible-playbook 04.kube-master.yml
```



#### 1.6.5.5 部署 node

node 节点必须安装 docker

```bash
root@k8s-master1:/etc/ansible$ vim roles/kube-node/defaults/main.yml
# 基础容器镜像
SANDBOX_IMAGE: "harbor.magedu.net/baseimages/pause-amd64:3.1"
root@k8s-master1:/etc/ansible$ ansible-playbook 05.kube-node.yml
```



#### 1.6.5.6 部署网络服务 calico

可选更改 calico 服务启动脚本路径， csr 证书信息

```bash
$ docker load -i calico-cni.tar
$ docker tag calico/cni:v3.4.4 harbor.magedu.net/baseimages/cni:v3.4.4
$ docker push harbor.magedu.net/baseimages/cni:v3.4.4
$ docker load -i calico-kube-controllers.tar
$ docker tag calico/kube-controllers:v3.4.4  harbor.magedu.net/baseimages/kube-
controllers:v3.4.4
$ docker push harbor.magedu.net/baseimages/kube-controllers:v3.4.4

#执行部署网络：
root@k8s-master1:/etc/ansible$ ansible-playbook 06.network.yml
```

验证 calico

```bash
root@k8s-master1:/etc/ansible$ calicoctl node status
Calico process is running.
IPv4 BGP status
+---------------+-------------------+-------+----------+-------------+
| PEER ADDRESS |   PEER TYPE   | STATE | SINCE  |  INFO   |
+---------------+-------------------+-------+----------+-------------+
| 192.168.7.110 | node-to-node mesh | up  | 14:22:44 | Established |
+---------------+-------------------+-------+----------+-------------+
IPv6 BGP status
No IPv6 peers found.

kubectl run net-test1 --image=alpine --replicas=4 sleep 360000 #创建pod测试夸主机网络通信是否正常
```



#### 1.6.5.7 添加 node 节点

```bash
[kube-node]
192.168.7.110
[new-node] # 预留组，后续添加node节点使用
192.168.7.111
root@k8s-master1:/etc/ansible$ ansible-playbook 20.addnode.yml
```



#### 1.6.5.8 添加 master 节点

注释掉 lb，否则无法下一步

```bash
[kube-master]
192.168.7.101
[new-master] # 预留组，后续添加master节点使用
192.168.7.102
root@k8s-master1:/etc/ansible$ ansible-playbook 21.addmaster.yml
```



#### 1.6.5.9 验证当前状态

```bash
root@k8s-master1:/etc/ansible$ calicoctl node status
calico process is running.
IPv4 BGP status
+---------------+-------------------+-------+----------+-------------+
| PEER ADDRESS |   PEER TYPE   | STATE | SINCE  |  INFO   |
+---------------+-------------------+-------+----------+-------------+
| 192.168.7.110 | node-to-node mesh | up  | 14:22:45 | Established |
| 192.168.7.111 | node-to-node mesh | up  | 14:33:24 | Established |
| 192.168.7.102 | node-to-node mesh | up  | 14:42:21 | Established |
+---------------+-------------------+-------+----------+-------------+
IPv6 BGP status
No IPv6 peers found.

root@k8s-master1:/etc/ansible$ kubectl get nodes
NAME      STATUS           ROLES  AGE  VERSION
192.168.7.101  Ready,SchedulingDisabled  master  37m  v1.13.5
192.168.7.102  Ready,SchedulingDisabled  master  41s  v1.13.5
192.168.7.110  Ready           node   33m  v1.13.5
192.168.7.111  Ready           node   15m  v1.13.5
```



## 1.7 k8s 应用环境

### 1.7.1 dashboard 

部署 kubernetes  的 web 管理界面 dashboard

1.7.1.1 具体步骤

```bash
1.导入dashboard镜像并上传至本地harbor服务器
$ tar xvf dashboard-yaml_image-1.10.1.tar.gz
./admin-user-sa-rbac.yaml
./kubernetes-dashboard-amd64-v1.10.1.tar.gz
./kubernetes-dashboard.yaml
./read-user-sa-rbac.yaml
./ui-admin-rbac.yaml
./ui-read-rbac.yaml

$ docker load -i kubernetes-dashboard-amd64-v1.10.1.tar.gz

$ docker tag gcr.io/google-containers/kubernetes-dashboard-amd64:v1.10.1 harbor.magedu.net/baseimages/kubernetes-dashboard-amd64:v1.10.1

$ docker push harbor.magedu.net/baseimages/kubernetes-dashboard-amd64:v1.10.1

2.修改 yaml 文件中的 dashboard 镜像地址为本地 harbor 地址
image: harbor.magedu.net/baseimages/kubernetes-dashboard-amd64:v1.10.1

3.创建服务
$ kubectl apply -f .
serviceaccount/admin-user created
clusterrolebinding.rbac.authorization.k8s.io/admin-user created
secret/kubernetes-dashboard-certs created
serviceaccount/kubernetes-dashboard created
role.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
deployment.apps/kubernetes-dashboard created
service/kubernetes-dashboard created
serviceaccount/dashboard-read-user created
clusterrolebinding.rbac.authorization.k8s.io/dashboard-read-binding created
clusterrole.rbac.authorization.k8s.io/dashboard-read-clusterrole created
clusterrole.rbac.authorization.k8s.io/ui-admin created
rolebinding.rbac.authorization.k8s.io/ui-admin-binding created
clusterrole.rbac.authorization.k8s.io/ui-read created
rolebinding.rbac.authorization.k8s.io/ui-read-binding created

4.验证dashboard启动完成：
$ kubectl get pods -n kube-system
NAME                    READY  STATUS  RESTARTS  AGE
calico-kube-controllers-77d9f69cdd-rbml2  1/1   Running  0     25m
calico-node-rk6jk             1/1   Running  1     20m
calico-node-vjn65             1/1   Running  0     25m
calico-node-wmj48             1/1   Running  0     25m
calico-node-wvsxb             1/1   Running  0     5m14s
kubernetes-dashboard-cb96d4fd8-th6nw    1/1   Running  0     37s

$ kubectl get svc -n kube-system
NAME          TYPE    CLUSTER-IP   EXTERNAL-IP  PORT(S)     AGE
kubernetes-dashboard  NodePort  10.20.32.158  <none>    443:27775/TCP  79s

$ kubectl cluster-info #查看集群信息
Kubernetes master is running at https://192.168.7.248:6443
kubernetes-dashboard is running at
https://192.168.7.248:6443/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy
```



#### 1.7.1.2 token  登录 dashboard

```bash
$ kubectl -n kube-system get secret | grep admin-user
$ kubectl -n kube-system describe secret admin-user-token-2wm96
```



#### 1.7.1.3 kubeconfig 登录

制作 Kubeconfig 文件



#### 1.7.1.4 修改 iptables 为 ipvs 及调度算法

```bash
root@s6:~$ vim /etc/systemd/system/kube-proxy.service --proxy-mode=ipvs \ --ipvs-scheduler=sh
```



#### 1.7.1.5 设置 token 登录会话保持时间

```bash
$ vim dashboard/kubernetes-dashboard.yaml
   image: 192.168.200.110/baseimages/kubernetes-dashboard-amd64:v1.10.1
   ports:
   - containerPort: 8443
    protocol: TCP
   args:
    - --auto-generate-certificates
    - --token-ttl=43200
```



#### 1.7.1.6 session 保持

```bash
sessionAffinity: ClientIP
sessionAffinityConfig:
 clientIP:
  timeoutSeconds: 10800
```





## 1.8 DNS 服务

> 目前常用的 dns 组件有 kube-dns 和 coredns 两个；推荐使用 coredns



### 1.8.1 部署 coredns

```bash
$ docker tag gcr.io/google-containers/coredns:1.2.6 harbor.magedu.net/baseimages/coredns:1.2.6
$ docker push harbor.magedu.net/baseimages/coredns:1.2.6
```



### 1.8.2 dns 测试

```bash
$ vim coredns.yaml
$ kubectl apply -f coredns.yaml

$ kubectl exec busybox nslookup kubernetes
Server:  10.20.254.254
Address 1: 10.20.254.254 kube-dns.kube-system.svc.linux36.local
Name:   kubernetes
Address 1: 10.20.0.1 kubernetes.default.svc.linux36.local

$ kubectl exec busybox nslookup kubernetes.default.svc.linux36.local
Server:  10.20.254.254
Address 1: 10.20.254.254 kube-dns.kube-system.svc.linux36.local
Name:   kubernetes.default.svc.linux36.local
Address 1: 10.20.0.1 kubernetes.default.svc.linux36.local
```



### 1.8.3 监控组件 heapster

heapster：数据采集
influxdb：数据存储
grafana：web展示

```bash
导入相应的镜像
更改yaml中的镜像地址
创建服务
```



















































