[TOC]



本文的HA是vip,生产和云上可以用LB和SLB,不过阿里的SLB四层有问题(不支持回源),可以每个node上代理127.0.0.1的某个port分摊在所有apiserver的port上,aws的SLB正常

master节点一定要kube-proxy和calico或者flannel,kube-proxy是维持svc的ip到pod的ip的负载均衡，而你流量想到pod的ip需要calico或者flannel组件的overlay网络下才可以，后续学到APIService和CRD的时候，APIService如果选中了svc，kube-apiserver会把这个APISerivce的请求代理到选中的svc上，最后流量流到svc选中的pod，该pod要处理请求然后回应，这个时候就是kube-apiserver解析svc的名字得到svc的ip，然后kube-proxy定向到pod的ip，calico或者flannel把包发到目标机器上，这个时候如果kube-proxy和calico或者flannel没有那你创建的APISerivce就没用了。apiserver的路由聚合没试过不知道可行不可行，本文中的metrics-server就是这样的工作流程，所以建议master也跑kubelet和cni，pod可以设置污点不让一般的pod调度上来，不然后续某些CRD用不了。

安装的版本：

> - Kubernetes v1.13.12
> - CNI v0.8.1
> - Flannel v0.11.0 或者 Calico v3.4
> - Docker CE 18.06.03，18.09.09

部署的网络信息：

> - Cluster IP CIDR: 10.244.0.0/16
> - Service Cluster IP CIDR: 10.96.0.0/12
> - Service DNS IP: 10.96.0.10
> - DNS DN: cluster.local
> - Kubernetes API VIP: 10.0.6.155
> - Kubernetes Ingress: 10.0.6.156

### 节点信息

|                                                              |          |      |        |
| ------------------------------------------------------------ | -------- | ---- | ------ |
| IP                                                           | Hostname | CPU  | Memory |
| 10.0.6.166                                                   | K8S-M1   | 4    | 8G     |
| 10.0.6.167                                                   | K8S-M2   | 4    | 8G     |
| 10.0.6.168                                                   | K8S-M3   | 4    | 8G     |
| 10.0.6.169                                                   | K8S-N1   | 2    | 4G     |
| 10.0.6.170                                                   | K8S-N2   | 2    | 4G     |
| 另外VIP为`10.0.6.155`,由所有`master节点`的`keepalived+haproxy`来选择VIP的归属保持高可用 |          |      |        |

### 事前准备

**安装注意事项：禁用 swap, seliunx, iptables**

- 关闭防火墙与 SELinux

```bash
$ systemctl stop firewalld iptables
$ systemctl disable firewalld
$ systemctl disable iptables
$ sed -i 's/SELINUX=.*/SELINUX=disabled/' /etc/selinux/config 
$ setenforce 0
```

- 关闭 dnsmasq (可选)
  linux 系统开启了 dnsmasq 后(如 GUI 环境)，将系统 DNS Server 设置为 127.0.0.1，这会导致 docker 容器无法解析域名，需要关闭它

```bash
systemctl disable --now dnsmasq
```

- Kubernetes v1.8+ 要求关闭系统Swap,若不关闭则需要修改 kubelet 设定参数( –fail-swap-on 设置为 false 来忽略 swap on),在所有机器使用以下指令关闭 swap 并注释掉/etc/fstab中swap的行：

```bash
$ swapoff -a && sysctl -w vm.swappiness=0
$ sed -ri '/^[^#]*swap/s@^@#@' /etc/fstab
```

- 设置主机名，在master添加hosts

```bash
$ hostnamectl set-hostname K8S-M1

$ cat >> /etc/hosts << EOF
10.0.6.166 K8S-M1
10.0.6.167 K8S-M2
10.0.6.168 K8S-M3
10.0.6.169 K8S-N1
10.0.6.170 K8S-N2
EOF
```

- 如果是centos的话不想升级后面的最新内核可以此时升级到保守内核去掉update的exclude即可

```bash
yum install epel-release -y
yum install wget git  jq psmisc socat -y
yum update -y --exclude=kernel*
```

- 如果上面`yum update`没有加`--exclude=kernel*`就重启下加载保守内核

```bash
reboot
```

> #因为目前市面上包管理下内核版本会很低,安装docker后无论centos还是ubuntu会有如下 bug,4.15的内核依然存在(所以建议先升级内核)

```bash
kernel:unregister_netdevice: waiting for lo to become free. Usage count = 1
```

**perl是内核的依赖包,如果没有就安装下**

```bash
[ ! -f /usr/bin/perl ] && yum install perl -y
```

- 升级内核需要使用 elrepo 的yum 源,首先我们导入 elrepo 的 key并安装 elrepo 源

```bash
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
```

- 查看可用的内核

```bash
yum --disablerepo="*" --enablerepo="elrepo-kernel" list available  --showduplicates
```

> ipvs依赖于`nf_conntrack_ipv4`内核模块,`4.19`包括之后内核里改名为`nf_conntrack`,1.13.1之前的kube-proxy的代码里没有加判断一直用的`nf_conntrack_ipv4`,1.13.1后的kube-proxy代码里增加了判断,会去load `nf_conntrack`使用ipvs正常
>
> 下面链接可以下载到其他归档版本的
>
> - ubuntu<http://kernel.ubuntu.com/~kernel-ppa/mainline/>
> - RHEL<http://mirror.rc.usf.edu/compute_lock/elrepo/kernel/el7/x86_64/RPMS/>

- 升级 linux 内核

```bash
yum update -y
# 导入公钥
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org 
# 安装7.x版本的ELRepo
rpm -Uvh https://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
# 安装新版本内核
yum --enablerepo=elrepo-kernel install kernel-lt -y
```

- 修改内核启动顺序,默认启动的顺序应该为1,升级以后内核是往前面插入,为0

```bash
grub2-set-default  0 && grub2-mkconfig -o /etc/grub2.cfg
```

- 使用下面命令看看确认下是否启动默认内核指向上面安装的内核

```bash
grubby --default-kernel
```

- 重启加载新内核

```bast
reboot
```

> ### 关于两个内核版本的说明
>
> *ELRepo有两种类型的Linux内核包，kernel-lt和kernel-ml。 他们之间有什么区别？*
> *kernel-ml软件包是根据Linux Kernel Archives的主线稳定分支提供的源构建的。 内核配置基于默认的RHEL-7配置，并根据需要启用了添加的功能。 这些软件包有意命名为kernel-ml，以免与RHEL-7内核发生冲突，因此，它们可以与常规内核一起安装和更新。*
> *kernel-lt包是从Linux Kernel Archives提供的源代码构建的，就像kernel-ml软件包一样。 不同之处在于kernel-lt基于长期支持分支，而kernel-ml基于主线稳定分支*

- **所有机器** 安装 ipvs

  在每台机器上安装依赖包：

  CentOS:

  ```bash
  yum install ipvsadm ipset sysstat conntrack libseccomp -y
  ```

  Ubuntu:

  ```bash
  sudo apt-get install -y wget git conntrack ipvsadm ipset jq sysstat curl iptables libseccomp
  ```

- **所有机器** 选择需要开机加载的内核模块

```bash
vi /etc/sysconfig/modules/ipvs.modules 
#!/bin/bash
ipvs_mods_dir="/usr/lib/modules/$(uname -r)/kernel/net/netfilter/ipvs"
for i in $(ls $ipvs_mods_dir | grep -o "^[^.]*"); do
  /sbin/modinfo -F filename $i &> /dev/null
  if [ $? -eq 0 ]; then
    /sbin/modprobe $i
  fi
done
```

- **所有机器** 需要设定系统参数

```bash
cat <<EOF > /etc/sysctl.d/k8s.conf
# https://github.com/moby/moby/issues/31208 
# ipvsadm -l --timout
# 修复ipvs模式下长连接timeout问题 小于900即可
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 10
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
net.ipv4.neigh.default.gc_stale_time = 120
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.default.arp_announce = 2
net.ipv4.conf.lo.arp_announce = 2
net.ipv4.conf.all.arp_announce = 2
net.ipv4.ip_forward = 1
net.ipv4.tcp_max_tw_buckets = 5000
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 1024
net.ipv4.tcp_synack_retries = 2
# 要求iptables不对bridge的数据进行处理
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-arptables = 1
net.netfilter.nf_conntrack_max = 2310720
fs.inotify.max_user_watches=89100
fs.may_detach_mounts = 1
fs.file-max = 52706963
fs.nr_open = 52706963
vm.swappiness = 0
vm.overcommit_memory=1
vm.panic_on_oom=0
EOF

sysctl --system
```

- 检查系统内核和模块是否适合运行 docker

```bash
curl https://raw.githubusercontent.com/docker/docker/master/contrib/check-config.sh > check-config.sh
bash ./check-config.sh
```

- `所有机器`需要安装Docker CE 版本的容器引擎
- 在官方查看K8s支持的docker版本 <https://github.com/kubernetes/kubernetes> 里进对应版本的changelog里搜`The list of validated docker versions remain`
- 这里利用docker的官方安装脚本来安装,可以使用`yum list --showduplicates docker-ce`查询可用的docker版本,选择你要安装的k8s版本支持的docker版本即可,这里我使用的是`18.06.03`

```bash
wget https://mirrors.aliyun.com/docker-ce/linux/centos/7/x86_64/stable/Packages/docker-ce-18.09.9-3.el7.x86_64.rpm
wget https://mirrors.aliyun.com/docker-ce/linux/centos/7/x86_64/stable/Packages/docker-ce-cli-19.03.9-3.el7.x86_64.rpm

export VERSION=18.06
curl -fsSL "https://get.docker.com/" | bash -s -- --mirror Aliyun
```

- `所有机器`配置加速源并配置docker的启动参数使用systemd,使用systemd是官方的建议,详见 <https://kubernetes.io/docs/setup/cri/>

```json
mkdir -p /etc/docker/
cat>/etc/docker/daemon.json<<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors": ["https://fz5yth0r.mirror.aliyuncs.com"],
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
EOF
```

- 设置 docker 开机启动, CentOS 安装完成后 docker 需要手动设置 docker 命令补全：

```bash
$ yum install -y epel-release bash-completion && cp /usr/share/bash-completion/completions/docker /etc/bash_completion.d/
$ systemctl enable --now docker
```

- 在`k8s-m1`上声明集群信息

> 根据自己环境声明用于的变量，后续操作依赖于环境变量。
>
> 默认 kubelet 向集群注册是带 `--hostname-override` 去设置注册的名字。

```bash
# 声明集群成员信息
declare -A MasterArray otherMaster NodeArray AllNode Other
MasterArray=(['k8s-m1']=10.0.6.166 ['k8s-m2']=10.0.6.167 ['k8s-m3']=10.0.6.168)
otherMaster=(['k8s-m2']=10.0.6.167 ['k8s-m3']=10.0.6.168)
NodeArray=(['k8s-n1']=10.0.6.169 ['k8s-n2']=10.0.6.170)
AllNode=(['k8s-m1']=10.0.6.166 ['k8s-m2']=10.0.6.167 ['k8s-m3']=10.0.6.168 ['k8s-n1']=10.0.6.169 ['k8s-n2']=10.0.6.170)
Other=(['k8s-m2']=10.0.6.167 ['k8s-m3']=10.0.6.168 ['k8s-n1']=10.0.6.169 ['k8s-n2']=10.0.6.170)

export VIP=10.0.6.155

[ "${#MasterArray[@]}" -eq 1 ]  && export VIP=${MasterArray[@]} || export API_PORT=8443
export KUBE_APISERVER=https://${VIP}:${API_PORT:=6443}

# 声明大规模安装的k8s版本
export KUBE_VERSION=v1.13.12

# 声明网卡名
export interface=eth0

# 声明cni
export CNI_URL="https://github.com/containernetworking/plugins/releases/download"
export CNI_VERSION=v0.8.1

# 声明etcd
export ETCD_version=v3.3.17
```

`k8s-m1`登陆其他机器要免密在`k8s-m1`安装`sshpass`后使用别名来让ssh和scp不输入密码,`zhangguanzhang`为所有机器密码

```bash
$ yum install sshpass -y
$ alias ssh='sshpass -p zhangguanzhang ssh -o StrictHostKeyChecking=no'
$ alias scp='sshpass -p zhangguanzhang scp -o StrictHostKeyChecking=no'
```

- 设置所有机器的`hostname`

```bash
for name in ${!AllNode[@]};do 
      echo "--- $name ${AllNode[$name]} ---"
  ssh ${AllNode[$name]} "hostnamectl set-hostname $name"
done
```

- 在`k8s-m1`通过git获取部署要用到的二进制配置文件和yml

```bash
$ git clone https://github.com/zhangguanzhang/k8s-manual-files.git ~/k8s-manual-files -b v1.13.4
$ cd ~/k8s-manual-files/
```

- 在`k8s-m1`下载Kubernetes二进制文件后分发到其他机器

```bash
$ curl https://storage.googleapis.com/kubernetes-release/release/v1.13.4/kubernetes-server-linux-amd64.tar.gz > kubernetes-server-linux-amd64.tar.gz
$ tar -zxvf kubernetes-server-linux-amd64.tar.gz  --strip-components=3 -C /usr/local/bin kubernetes/server/bin/kube{let,ctl,-apiserver,-controller-manager,-scheduler,-proxy}
```

分发 master 相关组件二进制文件到其他master上

```bash
for NODE in "${!otherMaster[@]}"; do
    echo "--- $NODE ${otherMaster[$NODE]} ---"
    scp /usr/local/bin/kube{let,ctl,-apiserver,-controller-manager,-scheduler,-proxy} ${otherMaster[$NODE]}:/usr/local/bin/ 
done
```

分发 node 的 kubernetes 二进制文件

```bash
for NODE in "${!NodeArray[@]}"; do
    echo "--- $NODE ${NodeArray[$NODE]} ---"
    scp /usr/local/bin/kube{let,-proxy} ${NodeArray[$NODE]}:/usr/local/bin/ 
done
```

- 在 `k8-m1` 下载 kubernetes CNI 二进制文件并分发

```bash
$ mkdir -p /opt/cni/bin
$ wget  "${CNI_URL}/${CNI_VERSION}/cni-plugins-linux-amd64-${CNI_VERSION}.tgz" 
$ tar -zxf cni-plugins-amd64-${CNI_VERSION}.tgz -C /opt/cni/bin

# 分发cni文件
for NODE in "${!Other[@]}"; do
    echo "--- $NODE ${Other[$NODE]} ---"
    ssh ${Other[$NODE]} 'mkdir -p /opt/cni/bin'
    scp /opt/cni/bin/* ${Other[$NODE]}:/opt/cni/bin/
done
```



### 建立集群 CA keys 与 Certificates

#### Kubernetes 中使用到的主要证书

![1590156654983](D:\学习资料\笔记\linux\assets\1590156654983.png) 

上图中使用序号对证书进行了标注。图中的箭头表明了组件的调用方向，箭头所指方向为服务提供方，另一头为服务调用方。为了实现 TLS 双向认证，服务提供方需要使用一个服务器证书，服务调用方则需要提供一个客户端证书，并且双方都需要使用一个 CA 证书来验证对方提供的证书。

1. etcd 对外提供服务，要有一套 etc server 证书。
2. etcd 各节点之间进行通信，要有一套 etcd peer 证书。
3. kube-apiserver 访问 Etcd，要有一套 etcd client 证书。
4. kube-apiserver 对外提供服务；要有一套 kube-apiserver 证书。
5. kube-controller-manager、kube-scheduler、kube-proxy、kubelet、kubectl，需要访问 kube-apiserver ，要有一套 kube-apiserver  client 证书。
6. kubelet 对外提供服务，要有一套 kubelet server 证书。
7. kube-apiserver 访问 kubelet ，要有一套 kubelet client 证书。
8. kube-controller-manager 要生成服务的 service account，要有一对用来签署 service account 的证书（CA 证书）

#### 准备 cfssl 证书生成工具

使用 CFSSL 来制作证书，它是 cloudflare 开发的一个开源的 PKI 工具，是一个完备的 CA 服务系统，可以签署、撤销证书等，覆盖了一个证书的整个生命周期。

```bash
 mkdir -p /usr/local/bin/cfssl
 wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 
 wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 
 wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64 
 chmod +x cfssl*
 mv cfssl_linux-amd64 /usr/local/bin/cfssl
 mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
 mv cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo
 export PATH=/usr/local/bin:$PATH
```

#### 创建 CA 证书

创建存放证书目录

```bash
 mkdir -p /opt/kubernetes/{cfg,bin,ssl,log}
 cd /usr/local/src
```

**创建证书配置文件**

```bash
 vim ca-config.json
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

**创建CA证书签名请求（CSR）文件**

```bash
[root@k8s-m1 ssl]$ vim ca-csr.json
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
  ],
    "ca": {
       "expiry": "87600h"
    }
}
```

**生成CA证书（ca.pem）和密钥（ca-key.pem）**

```bash
[root@k8s-m1 src]$ cfssl gencert -initca ca-csr.json | cfssljson -bare ca
[root@k8s-m1 src]$ ls | grep ca
ca-config.json
ca.csr
ca-csr.json
ca-key.pem
ca.pem

[root@k8s-m1 src]$ scp ca.csr ca.pem ca-key.pem ca-config.json /opt/kubernetes/ssl
[root@k8s-m1 src]$ ls /opt/kubernetes/ssl/
ca-config.json
ca.csr
ca-key.pem
ca.pem
```

> 其中 ca-key.pem 是 ca 的私钥，ca.csr 是一个签署请求，ca.pem 是 CA 证书，是后面 kubernetes 组件会用到的 RootCA。

**分发证书**

```bash
for NODE in "${!Other[@]}"; do
    echo "--- $NODE ${Other[$NODE]} ---"
    ssh ${Other[$NODE]} 'mkdir -p /opt/kubernetes/{cfg,bin,ssl,log}'
    scp /opt/kubernetes/ssl/* ${Other[$NODE]}:/opt/kubernetes/ssl
done
```



### 部署 etcd 集群

下载 etcd 安装包

```bash
$ wget https://github.com/etcd-io/etcd/releases/download/v3.2.18/etcd-v3.2.18-linux-amd64.tar.gz

$ tar zxf etcd-v3.2.18-linux-amd64.tar.gz
$ cd etcd-v3.2.18-linux-amd64

$ cp etcdctl  etcd /opt/kubernetes/bin/

$ scp  /opt/kubernetes/bin/etcd* k8s-m2:/opt/kubernetes/bin/
$ scp  /opt/kubernetes/bin/etcd* k8s-m3:/opt/kubernetes/bin/
```



#### 自签 CA 签发 Etcd HTTPS 证书

**创建证书申请文件**

```json
vim etcd-csr.json
{
  "CN": "etcd",
  "hosts": [
  "10.0.6.166",
  "10.0.6.167",
  "10.0.6.168"
 ],
  "key": {
    "algo": "rsa",
    "size": 2048
 },
  "names": [
   {
      "C": "CN",
      "L": "BeiJing",
      "ST": "BeiJing"
   }
 ]
}
```

**生成 etcd 证书和私钥**

```bash
$ cd /usr/local/src/ssl/etcd

$ cfssl gencert -ca=/opt/kubernetes/ssl/ca.pem -ca-key=/opt/kubernetes/ssl/ca-key.pem -config=/opt/kubernetes/ssl/ca-config.json -profile=kubernetes etcd-csr.json | cfssljson -bare etcd

$ ls /usr/local/src/ssl/etcd
etcd.csr  etcd-csr.json etcd-key.pem etcd.pem

# 将证书移动到/opt/kubernetes/ssl目录下将证书移动到/opt/kubernetes/ssl目录下
$ cp etcd*.pem /opt/kubernetes/ssl

#将证书复制到各etcd 服务器节点
$ scp  /opt/kubernetes/ssl/etcd* k8s-m2:/opt/kubernetes/ssl/
$ scp  /opt/kubernetes/ssl/etcd* k8s-m3:/opt/kubernetes/ssl/
```



#### 创建 etcd 配置文件

```bash
vim /opt/kubernetes/cfg/etcd.conf
#[Member]
ETCD_NAME="k8m-1"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://10.0.6.166:2380"
ETCD_LISTEN_CLIENT_URLS="https://10.0.6.166:2379"
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://10.0.6.166:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://10.0.6.166:2379"
ETCD_INITIAL_CLUSTER="k8m-1=https://10.0.6.166:2380,etcd-2=https://10.0.6.167:2380,etcd-3=https://10.0.6.168:2380"
ETCD_INITIAL_CLUSTER_TOKEN="k8s-etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
#[security]
CLIENT_CERT_AUTH="true"
ETCD_CA_FILE="/opt/kubernetes/ssl/ca.pem"
ETCD_CERT_FILE="/opt/kubernetes/ssl/etcd.pem"
ETCD_KEY_FILE="/opt/kubernetes/ssl/etcd-key.pem"
PEER_CLIENT_CERT_AUTH="true"
ETCD_PEER_CA_FILE="/opt/kubernetes/ssl/ca.pem"
ETCD_PEER_CERT_FILE="/opt/kubernetes/ssl/etcd.pem"
ETCD_PEER_KEY_FILE="/opt/kubernetes/ssl/etcd-key.pem"

# 同步到其他节点后,修改对应的NAME和监听IP地址
[root@k8s-etcd1 src]$ scp /opt/kubernetes/cfg/etcd.conf  10.0.6.167:/opt/kubernetes/cfg/
[root@k8s-etcd1 src]$ scp /opt/kubernetes/cfg/etcd.conf  10.0.6.168:/opt/kubernetes/cfg/

# 节点2参考如例
vim /opt/kubernetes/cfg/etcd.conf
ETCD_NAME="k8m-2"  # 修改此处，节点2改为k8m-2，节点3改为k8m-3
ETCD_LISTEN_PEER_URLS="https://10.0.6.167:2380"  # 修改此处为当前服务器IP
ETCD_LISTEN_CLIENT_URLS="https://10.0.6.167:2379"  # 修改此处为当前服务器IP
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://10.0.6.167:2380"  # 修改此处为当前服务器IP
ETCD_ADVERTISE_CLIENT_URLS="https://10.0.6.167:2379"   # 修改此处为当前服务器IP
```

> ETCD_NAME：节点名称，集群中唯一
> ETCD_DATA_DIR：数据目录
> ETCD_LISTEN_PEER_URLS：集群通信监听地址
> ETCD_LISTEN_CLIENT_URLS：客户端访问监听地址
> ETCD_INITIAL_ADVERTISE_PEER_URLS：集群通告地址
> ETCD_ADVERTISE_CLIENT_URLS：客户端通告地址
> ETCD_INITIAL_CLUSTER：集群节点地址
> ETCD_INITIAL_CLUSTER_TOKEN：集群Token
> ETCD_INITIAL_CLUSTER_STATE：加入集群的当前状态，new是新集群，existing表示加入已有集群



#### systemd 管理 etcd

```bash
vim /usr/lib/systemd/system/etcd.service
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
[Service]
Type=notify
WorkingDirectory=/var/lib/etcd
EnvironmentFile=-/opt/kubernetes/cfg/etcd.conf
ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /opt/kubernetes/bin/etcd"
Restart=on-failure
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target

# 各服务器创建数据目录：
 mkdir /var/lib/etcd

systemctl start etcd &&  systemctl enable etcd && systemctl  status etcd

# 同步到其他节点后
[root@k8s-etcd1 src]$ scp /usr/lib/systemd/system/etcd.service 10.0.6.167:/usr/lib/systemd/system/
[root@k8s-etcd1 src]$ scp /usr/lib/systemd/system/etcd.service 10.0.6.168:/usr/lib/systemd/system/
```



#### 创建 etcdctl 命令软连接

```bash
ln -sv /opt/kubernetes/bin/etcdctl  /usr/bin/
```



#### 验证集群状态

```bash
etcdctl --endpoints=https://10.0.6.166:2379 \
  --ca-file=/opt/kubernetes/ssl/ca.pem \
  --cert-file=/opt/kubernetes/ssl/etcd.pem \
  --key-file=/opt/kubernetes/ssl/etcd-key.pem cluster-health
  
etcdctl --endpoints=https://10.0.6.166:2379 \
  --ca-file=/opt/kubernetes/ssl/ca.pem \
  --cert-file=/opt/kubernetes/ssl/etcd.pem \
  --key-file=/opt/kubernetes/ssl/etcd-key.pem member list

# 使用3的api去查询集群的键值
ETCDCTL_API=3 \
   etcdctl   \
   --cert=/opt/kubernetes/ssl/etcd.pem \
   --key=/opt/kubernetes/ssl/etcd-key.pem \
   --cacert=/opt/kubernetes/ssl/ca.pem \
   --endpoints=https://10.0.6.166:2379 get / --prefix --keys-only
```

> 如果有问题第一步先看日志：`/var/log/message` 或 `journalctl -u etcd`



### 部署Master Node

**kubelet**:

> - 负责管理容器的生命周期,定期从`API Server`获取节点上的预期状态(如网络、存储等等配置)资源,并让对应的容器插件(CRI、CNI 等)来达成这个状态。任何 Kubernetes 节点(node)都会拥有这个
> - 关闭只读端口，在安全端口 10250 接收 https 请求，对请求进行认证和授权，拒绝匿名访问和非授权访问
> - 使用 kubeconfig 访问 apiserver 的安全端口

**kube-apiserver**:

> - 以 REST APIs 提供 Kubernetes 资源的 CRUD,如授权、认证、存取控制与 API 注册等机制。
> - 关闭非安全端口,在安全端口 6443 接收 https 请求
> - 严格的认证和授权策略 (x509、token、RBAC)
> - 开启 bootstrap token 认证，支持 kubelet TLS bootstrapping
> - 使用 https 访问 kubelet、etcd，加密通信

**kube-controller-manager**:

> - 通过核心控制循环(Core Control Loop)监听 Kubernetes API 的资源来维护集群的状态,这些资源会被不同的控制器所管理,如 Replication Controller、Namespace Controller 等等。而这些控制器会处理着自动扩展、滚动更新等等功能。
> - 关闭非安全端口，在安全端口 10252 接收 https 请求
> - 使用 kubeconfig 访问 apiserver 的安全端口

**kube-scheduler**:

> - 负责将一個(或多个)容器依据调度策略分配到对应节点上让容器引擎(如 Docker)执行。而调度受到 QoS 要求、软硬性约束、亲和性(Affinity)等等因素影响。



#### 生成kube-apiserver证书

**创建生成 CSR 文件的 json 文件**

```json
[root@k8s-master1 src]$ vim kubernetes-csr.json
{
  "CN": "kubernetes",
  "hosts": [
    "127.0.0.1",
    "10.1.0.1",
    "10.0.6.155",
    "10.0.6.166",
    "10.0.6.167",
	"10.0.6.168",
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

> 注：上述文件hosts字段中IP为所有Master/LB/VIP IP，一个都不能少！为了方便后期扩容可以多写几个预留的IP。

**生成证书**

```bash
$ cfssl gencert -ca=/opt/kubernetes/ssl/ca.pem -ca-key=/opt/kubernetes/ssl/ca-key.pem -config=/opt/kubernetes/ssl/ca-config.json  -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes

$ ls *.pem
kubernetes-key.pem kubernetes.pem
```

**将证书复制到各服务器**

```bash
cp kubernetes*.pem /opt/kubernetes/ssl/

for NODE in "${!Other[@]}"; do
    echo "--- $NODE ${Other[$NODE]} ---"
    scp /opt/kubernetes/ssl/kubernetes*.pem ${Other[$NODE]}:/opt/kubernetes/ssl/
done
```



#### 部署 kube-apiserver

**创建配置文件**

```bash
$ vim /opt/kubernetes/cfg/kube-apiserver.conf
KUBE_APISERVER_OPTS="--logtostderr=false \
--v=2 \
--log-dir=/opt/kubernetes/logs \
--etcd-servers=https://10.0.6.166:2379,https://10.0.6.167:2379,https://10.0.6.168:2379 \
--bind-address=10.0.6.166 \
--secure-port=6443 \
--advertise-address=10.0.6.166 \
--allow-privileged=true \
--service-cluster-ip-range=10.254.0.0/16 \
--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota,NodeRestriction \
--authorization-mode=RBAC,Node \
--enable-bootstrap-token-auth=true \
--token-auth-file=/opt/kubernetes/ssl/bootstrap-token.csv \
--service-node-port-range=30000-50000 \
--kubelet-client-certificate=/opt/kubernetes/ssl/kubernetes.pem \
--kubelet-client-key=/opt/kubernetes/ssl/kubernetes-key.pem \
--tls-cert-file=/opt/kubernetes/ssl/kubernetes.pem \
--tls-private-key-file=/opt/kubernetes/ssl/kubernetes-key.pem \
--client-ca-file=/opt/kubernetes/ssl/ca.pem \
--service-account-key-file=/opt/kubernetes/ssl/ca-key.pem \
--etcd-cafile=/opt/kubernetes/ssl/ca.pem \
--etcd-certfile=/opt/kubernetes/ssl/kubernetes.pem \
--etcd-keyfile=/opt/kubernetes/ssl/kubernetes-key.pem \
--audit-log-maxage=30 \
--audit-log-maxbackup=3 \
--audit-log-maxsize=100 \
--audit-log-path=/opt/kubernetes/log/api-audit.log"
```

> - --logtostderr：启用日志
> - ---v：日志等级
> - --log-dir：日志目录
> - --etcd-servers：etcd集群地址
> - --bind-address：监听地址
> - --secure-port：https安全端口
> - --advertise-address：集群通告地址
> - --allow-privileged：启用授权
> - --service-cluster-ip-range：Service虚拟IP地址段
> - --enable-admission-plugins：准入控制模块
> - --authorization-mode：认证授权，启用RBAC授权和节点自管理
> - --enable-bootstrap-token-auth：启用TLS bootstrap机制
> - --token-auth-file：bootstrap token文件
> - --service-node-port-range：Service nodeport类型默认分配端口范围
> - --kubelet-client-xxx：apiserver访问kubelet客户端证书
> - --tls-xxx-file：apiserver https证书
> - --etcd-xxxfile：连接Etcd集群证书
> - --audit-log-xxx：审计日志



**启用 TLS Bootstrapping 机制**

TLS Bootstraping：Master apiserver启用TLS认证后，Node节点kubelet和kube-proxy要与kube-apiserver进行通信，必须使用CA签发的有效证书才可以，当Node节点很多时，这种客户端证书颁发需要大量工作，同样也会增加集群扩展复杂度。为了简化流程，Kubernetes引入了 TLS bootstraping 机制来自动颁发客户端证书，kubelet 会以一个低权限用户自动向apiserver 申请证书，kubelet 的证书由 apiserver 动态签署。所以强烈建议在Node上使用这种方式，目前主要用于kubelet，kube-proxy还是由我们统一颁发一个证书。

TLS bootstraping 工作流程：

![1591149565353](D:\学习资料\笔记\linux\assets\1591149565353.png)

**创建上述配置文件中token文件**

```bash
head -c 16 /dev/urandom | od -An -t x | tr -d ' '
9077bdc74eaffb83f672fe4c530af0d6

# 格式：token，用户名，UID，用户组
vim /opt/kubernetes/ssl/bootstrap-token.csv #各master服务器
9077bdc74eaffb83f672fe4c530af0d6,kubelet-bootstrap,10001,"system:kubelet-bootstrap"
```

**配置认证用户密码：**

```bash
vim /opt/kubernetes/ssl/basic-auth.csv
admin,admin,1
readonly,readonly,2
```

**systemd管理apiserver**

```bash
vim /usr/lib/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-apiserver.conf
ExecStart=/opt/kubernetes/bin/kube-apiserver \$KUBE_APISERVER_OPTS
Restart=on-failure
[Install]
WantedBy=multi-user.target
```

**启动并设置开机启动**

```bash
systemctl daemon-reload && systemctl enable kube-apiserver && systemctl start kube-apiserver && systemctl status  kube-apiserver
```

**授权kubelet-bootstrap用户允许请求证书**

```bash
kubectl create clusterrolebinding kubelet-bootstrap \
--clusterrole=system:node-bootstrapper \
--user=kubelet-bootstrap
```



#### 部署kube-controller-manager

**创建配置文件**

```bash
vim /opt/kubernetes/cfg/kube-controller-manager.conf
KUBE_CONTROLLER_MANAGER_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--leader-elect=true \\
--master=http://127.0.0.1:8080 \\
--bind-address=127.0.0.1 \\
--allocate-node-cidrs=true \\
--cluster-cidr=10.2.0.0/16 \\
--service-cluster-ip-range=10.1.0.0/16 \\
--cluster-signing-cert-file=/opt/kubernetes/ssl/ca.pem \\
--cluster-signing-key-file=/opt/kubernetes/ssl/ca-key.pem \\
--root-ca-file=/opt/kubernetes/ssl/ca.pem \\
--service-account-private-key-file=/opt/kubernetes/ssl/ca-key.pem \\
--experimental-cluster-signing-duration=87600h0m0s"
EOF
```

**systemd管理controller-manager**

```bash
cat > /usr/lib/systemd/system/kube-controller-manager.service << EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes
[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-controller-manager.conf
ExecStart=/opt/kubernetes/bin/kube-controller-manager
\$KUBE_CONTROLLER_MANAGER_OPTS
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF
```

> 将启动文件和启动脚本scp到其他 master 并启动服务和验证

**启动并设置开机启动**

```bash
systemctl daemon-reload && systemctl enable kube-controller-manager && systemctl start kube-controller-manager && systemctl status  kube-controller-manager
```



#### 部署 kube-scheduler

**创建配置文件**

```bash
vim /opt/kubernetes/cfg/kube-scheduler.conf
KUBE_SCHEDULER_OPTS="--logtostderr=false \
--v=2 \
--log-dir=/opt/kubernetes/logs \
--leader-elect \
--master=127.0.0.1:8080 \
--bind-address=127.0.0.1"
```

**systemd管理scheduler**

```bash
cat > /usr/lib/systemd/system/kube-scheduler.service << EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes
[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-scheduler.conf
ExecStart=/opt/kubernetes/bin/kube-scheduler \$KUBE_SCHEDULER_OPTS
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF
```

> 将启动文件和启动脚本scp到其他 master 并启动服务和验证

**启动并设置开机启动**

```bash
systemctl daemon-reload && systemctl enable kube-scheduler && systemctl start kube-scheduler && systemctl status  kube-scheduler
```



#### 部署 kubectl

```bash
$ cp kubernetes/client/bin/kubectl  /opt/kubernetes/bin/
$ scp /opt/kubernetes/bin/kubectl  192.168.100.102:/opt/kubernetes/bin/
$ ln -sv /opt/kubernetes/bin/kubectl  /usr/bin/
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
$ cfssl gencert -ca=/opt/kubernetes/ssl/ca.pem -ca-key=/opt/kubernetes/ssl/ca-key.pem -config=/opt/kubernetes/ssl/ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin

$ ls admin*
admin.csr admin-csr.json admin-key.pem admin.pem
$ cp admin*.pem /opt/kubernetes/ssl/
```

**设置集群参数**

```bash
# master1：
kubectl config set-cluster kubernetes \
   --certificate-authority=/opt/kubernetes/ssl/ca.pem \
   --embed-certs=true \
   --server=https://192.168.100.101:6443

# master2：
kubectl config set-cluster kubernetes \
   --certificate-authority=/opt/kubernetes/ssl/ca.pem \
   --embed-certs=true \
   --server=https://192.168.100.112:6443
```

**设置客户端认证参数**

```bash
# master1：
kubectl config set-credentials admin \
    --client-certificate=/opt/kubernetes/ssl/admin.pem \
    --embed-certs=true \
    --client-key=/opt/kubernetes/ssl/admin-key.pem

# master2：
kubectl config set-credentials admin \
    --client-certificate=/opt/kubernetes/ssl/admin.pem \
    --embed-certs=true \
    --client-key=/opt/kubernetes/ssl/admin-key.pem
```

**设置上下文参数**

```bash
[root@k8s-master1 src]$ kubectl config set-context kubernetes \
--cluster=kubernetes  --user=admin 

[root@k8s-master2 src]$ kubectl config set-context kubernetes --cluster=kubernetes  --user=admin 
```

**设置默认上下文**

```bash
[root@k8s-master1 src]$ kubectl config use-context kubernetes

[root@k8s-master2 src]$ kubectl config use-context kubernetes
```



### 部署 Worker Node

#### 环境准备

**准备二进制包**

```bash
[root@k8s-master1 src]$ scp kubernetes/server/bin/kube-proxy  kubernetes/server/bin/kubelet  192.168.100.107:/opt/kubernetes/bin/
[root@k8s-master1 src]$ scp kubernetes/server/bin/kube-proxy kubernetes/server/bin/kubelet  192.168.100.108:/opt/kubernetes/bin/
```

**角色绑定**

```bash
[root@k8s-master1 src]$  kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap
```

**设置集群参数**

```bash
[root@k8s-master1 src]$ kubectl config set-cluster kubernetes \
   --certificate-authority=/opt/kubernetes/ssl/ca.pem \
   --embed-certs=true \
   --server=${KUBE_APISERVER}  \
   --kubeconfig=bootstrap.kubeconfig
```

**设置客户端认证参数**

```bash
 [root@k8s-master1 src]$ kubectl config set-credentials kubelet-bootstrap \
 --token=9077bdc74eaffb83f672fe4c530af0d6 \
 --kubeconfig=bootstrap.kubeconfig  
```

 **设置上下文参数**

```bash
[root@k8s-master1 src]$ kubectl config set-context default \
   --cluster=kubernetes \
   --user=kubelet-bootstrap \
   --kubeconfig=bootstrap.kubeconfig
```

**选择默认上下文**

```bash
$ kubectl config use-context default --kubeconfig=bootstrap.kubeconfig
```

**同步配置至 node 节点**

```bash
[root@k8s-master1 src]$ scp bootstrap.kubeconfig  192.168.100.108:/opt/kubernetes/cfg/ #自动生成文件
[root@k8s-master1 src]$ scp bootstrap.kubeconfig  192.168.100.109:/opt/kubernetes/cfg/
```



#### 部署 kubelet

**创建配置文件**

```bash
# 每个 node
vim /lib/systemd/system/kubelet.service
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
root@k8s-master2 src]$  kubectl get csr
NAME                            AGE       REQUESTOR           CONDITION
node-csr-dD84BwWWLl43SeiB5G1PnUaee5Sv60RsoVZFkuuePg0   1m        kubelet-bootstrap   Pending
node-csr-vW4eBZb98z-DvAeG9q8hb9mOAUg0U9HSML9YRBscP8A   2m        kubelet-bootstrap   Pending
```

**master 批准TLS 请求**

```bash
[root@k8s-master2 src]$ kubectl certificate approve node-csr-dD84BwWWLl43SeiB5G1PnUaee5Sv60RsoVZFkuuePg0

[root@k8s-master2 src]$ kubectl get nodes
NAME              STATUS    ROLES     AGE       VERSION
192.168.100.108   Ready     <none>    32s       v1.11.0
192.168.100.109   Ready     <none>    32s       v1.11.0
```



#### 部署 kube-proxy

**创建 kube-proxy 证书请求**

```json
[root@k8s-master1 src]$  vim kube-proxy-csr.json
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
$ cfssl gencert -ca=/opt/kubernetes/ssl/ca.pem -ca-key=/opt/kubernetes/ssl/ca-key.pem -config=/opt/kubernetes/ssl/ca-config.json -profile=kubernetes  kube-proxy-csr.json | cfssljson -bare kube-proxy
```

**复制证书到各node节点**

```bash
[root@k8s-master1 src]$ cp kube-proxy*.pem /opt/kubernetes/ssl/
[root@k8s-master1 src]$ bash /root/ssh.sh
```

**创建kube-proxy配置文件**

```bash
kubectl config set-cluster kubernetes \
   --certificate-authority=/opt/kubernetes/ssl/ca.pem \
   --embed-certs=true \
   --server=https://192.168.100.101:6443 \
   --kubeconfig=kube-proxy.kubeconfig
   
kubectl config set-credentials kube-proxy \
    --client-certificate=/opt/kubernetes/ssl/kube-proxy.pem \
    --client-key=/opt/kubernetes/ssl/kube-proxy-key.pem \
   --embed-certs=true \
   --kubeconfig=kube-proxy.kubeconfig

kubectl config set-context default \
    --cluster=kubernetes \
    --user=kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

**分发kubeconfig配置文件至 node 节点**

```bash
scp kube-proxy.kubeconfig  192.168.100.108:/opt/kubernetes/cfg/
scp kube-proxy.kubeconfig  192.168.100.109:/opt/kubernetes/cfg/
```

**创建kube-proxy服务配置**

```bash
mkdir /var/lib/kube-proxy

vim /usr/lib/systemd/system/kube-proxy.service
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

```bash
ipvsadm ipset conntrack
[root@k8s-node1 ~]$ systemctl daemon-reload && systemctl enable kube-proxy && systemctl start kube-proxy && systemctl status kube-proxy
[root@k8s-node2 ~]$ systemctl daemon-reload && systemctl enable kube-proxy && systemctl start kube-proxy && systemctl status kube-proxy

[root@k8s-node1 ~]$ ipvsadm -L -n
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.1.0.1:443 rr
  -> 192.168.100.101:6443         Masq    1      0          0         
  -> 192.168.100.102:6443         Masq    1      0          0 
```



### 集群网络

Kubernetes 在默认情况下与 Docker 的网络有所不同。在 Kubernetes 中有四个问题是需要被解决的，分别为：

- 高耦合的容器到容器通信：通过 Pods 内  localhost 来解决。
- Pod 到 Pod 的通信：通过实现网络模型来解决。
- Pod 到 Service 通信：由 Service objects 结合 kube-proxy 解决。
- 外部到 Service 通信：一样由 Service objects 结合 kube-proxy 解决。

而 Kubernetes 对于任何网络的实现都需要满足以下基本要求(除非是有意调整的网络分段策略)：

- 所有容器能够在没有 NAT 的情况下与其他容器通信。
- 所有节点能够在没有 NAT 的情况下与所有容器通信（反之亦然）。
- 容器看到的 IP 与其他人看到的 IP 是一样的。

目前 Kubernetes 已经有非常多种的[网络模型](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-implement-the-kubernetes-networking-model)作为[网络插件(Network Plugins)](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)方式被实现,因此可以选用满足自己需求的网络功能来使用。另外 Kubernetes 中的网络插件有以下两种形式：

- CNI plugins：以 appc/CNI 标准规范所实现的网络，详细可以阅读 [CNI Specification](https://github.com/containernetworking/cni/blob/master/SPEC.md)。
- Kubenet plugin：使用 CNI plugins 的 bridge 与  host-local 来实现基本的 cbr0。这通常被用在公有云服务上的 Kubernetes 集群网络。

> 如果想了解如何选择可以如阅读 Chris Love 的 [Choosing a CNI Network Provider for Kubernetes](https://chrislovecnm.com/kubernetes/cni/choosing-a-cni-provider/) 文章。



#### flannel

flannel 使用 vxlan 技术为各节点创建一个可以互通的 Pod 网络，使用的端口为 UDP 8472，需要开放该端口（如公有云 AWS 等）。

flannel 第一次启动时，从 etcd 获取 Pod 网段信息，为本节点分配一个未使用的 /24 段地址，然后创建 flannel.1（也可能是其它名称，如 flannel1 等） 接口。

> 在mster各节点和各node节点部署flannel服务。

**生成证书申请csr文件**

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
$ cfssl gencert -ca=/opt/kubernetes/ssl/ca.pem -ca-key=/opt/kubernetes/ssl/ca-key.pem -config=/opt/kubernetes/ssl/ca-config.json -profile=kubernetes flanneld-csr.json | cfssljson -bare flanneld

$ cp  flanneld*.pem /opt/kubernetes/ssl/
$ scp flanneld*.pem 192.168.100.108:/opt/kubernetes/ssl/
$ scp flanneld*.pem 192.168.100.109:/opt/kubernetes/ssl/

$ tar xvf flannel-v0.10.0-linux-amd64.tar.gz
$ scp -P22 flanneld  192.168.100.108:/opt/kubernetes/bin/
$ scp -P22 flanneld  192.168.100.109:/opt/kubernetes/bin/
```

**复制对应脚本到 /opt/kubernetes/bin 目录**

```bash
$ cd /usr/local/src/kubernetes/cluster/centos/node/bin/
$ cp remove-docker0.sh /opt/kubernetes/bin/

$ scp remove-docker0.sh  mk-docker-opts.sh 192.168.100.108:/opt/kubernetes/bin/
$ scp remove-docker0.sh  mk-docker-opts.sh 192.168.100.109:/opt/kubernetes/bin/
```

**配置 Flannel**

```bash
$ vim /opt/kubernetes/cfg/flannel
FLANNEL_ETCD="-etcd-endpoints=https://192.168.100.105:2379,https://192.168.100.106:2379,https://192.168.100.107:2379"
FLANNEL_ETCD_KEY="-etcd-prefix=/kubernetes/network"
FLANNEL_ETCD_CAFILE="--etcd-cafile=/opt/kubernetes/ssl/ca.pem"
FLANNEL_ETCD_CERTFILE="--etcd-certfile=/opt/kubernetes/ssl/flanneld.pem"
FLANNEL_ETCD_KEYFILE="--etcd-keyfile=/opt/kubernetes/ssl/flanneld-key.pem"
```

**配置 Flannel 系统启动文件**

```bash
$ vim /lib/systemd/system/flannel.service

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

**复制配置到其它节点上**

```bash
$ scp /opt/kubernetes/cfg/flannel 192.168.100.108:/opt/kubernetes/cfg/
$ scp /lib/systemd/system/flannel.service 192.168.100.108:/lib/systemd/system/flannel.service
```

**Flannel CNI集成**

```bash
$ mkdir /opt/kubernetes/bin/cni
$ tar zxf cni-plugins-amd64-v0.7.1.tgz -C /opt/kubernetes/bin/cni
$ scp -r  /opt/kubernetes/bin/cni 192.168.100.108:/opt/kubernetes/bin/
```

**在 etcd 创建网络**

提前将证书复制到etcd或在node节点操作

```bash
$ /opt/kubernetes/bin/etcdctl --ca-file /opt/kubernetes/ssl/ca.pem --cert-file /opt/kubernetes/ssl/flanneld.pem --key-file    /opt/kubernetes/ssl/flanneld-key.pem  --no-sync -C https://192.168.100.105:2379,https://192.168.100.106:2379,https://192.168.100.107:2379  mk /kubernetes/network/config  '{ "Network": "10.2.0.0/16", "Backend": { "Type": "vxlan", "VNI": 1 }}' 
```

**验证网段**

```bash
$ /opt/kubernetes/bin/etcdctl --ca-file /opt/kubernetes/ssl/ca.pem --cert-file /opt/kubernetes/ssl/flanneld.pem --key-file /opt/kubernetes/ssl/flanneld-key.pem     --no-sync -C  https://192.168.100.107:2379 get  /kubernetes/network/config   
```

**启动 flannel**

```bash
$ systemctl daemon-reload && systemctl enable flannel && chmod +x /opt/kubernetes/bin/* &&  systemctl start flannel &&  systemctl status  flannel
```

**node 节点验证获取到 IP 地址**

```bash
[root@k8s-node2]$ cat /var/run/flannel/docker 
DOCKER_OPT_BIP="--bip=10.2.32.1/24"
DOCKER_OPT_MTU="--mtu=1450"
DOCKER_OPTS=" --bip=10.2.32.1/24 --mtu=1450 "

$ cat /var/run/flannel/subnet.env 
FLANNEL_NETWORK=10.2.0.0/16
FLANNEL_SUBNET=10.2.32.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=false
```

**验证网段**

```bash
[root@k8s-node2]$ ip a
flannel.1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        inet 10.2.32.0  netmask 255.255.255.255  broadcast 0.0.0.0
```

**配置 Docker 服务使用 Flannel**

```bash
$ vim /usr/lib/systemd/system/docker.service
[Unit] #在Unit下面修改After和增加Requires
After=network-online.target firewalld.service flannel.service
Wants=network-online.target
Requires=flannel.service

[Service] #增加EnvironmentFile=-/run/flannel/docker
Type=notify
EnvironmentFile=-/run/flannel/docker
ExecStart=/usr/bin/dockerd $DOCKER_OPTS
```

**测试创建应用**

```bash
$ kubectl run net-test --image=alpine --replicas=2 sleep 360000

# 会自动下载镜像，然后验证容器启动成功：
kubectl get pods
```



#### Calico

Calico 是一款纯 Layer 3 的网络，其好处是它整合了各种云原生平台(Docker、Mesos 与 OpenStack 等)，且 Calico 不采用 vSwitch，而是在每个 Kubernetes 节点使用 vRouter 功能，并通过 Linux Kernel 既有的 L3 forwarding 功能，而当集群中心复杂度增加时，Calico 也可以利用 BGP route reflector 來达成。

> - 想了解 Calico 与传统 overlay networks 的差异，可以阅读 [Difficulties with traditional overlay networks](https://www.projectcalico.org/learn/) 文章。
>
> - 另外当节点超过 50 台，可以使用 Calico 的 Typha 模式来减少通过 Kubernetes datastore 造成 API Server 的负担。

由于 Calico 提供了 Kubernetes resources YAML 文件来快速以容器方式部署网络插件至所有节点上，因此只需要在 k8s-m1 使用 kubeclt 执行下面指令來建立：

```bash
$ curl https://docs.projectcalico.org/v3.1/getting-started/kubernetes/installation/hosted/calico.yaml -O
$ curl -s https://zhangguanzhang.github.io/bash/pull.sh | bash -s -- quay.io/calico/cni:v3.1.3
```

```bash
sed -ri "s#\{\{ interface \}\}#${interface}#" addons/calico/v3.1/calico.yml
kubectl apply -f addons/calico/v3.1
```

```bash
$ kubectl -n kube-system get pod --all-namespaces
NAMESPACE     NAME                              READY     STATUS              RESTARTS   AGE
kube-system   calico-node-2hdqf                 0/2       ContainerCreating   0          4m
kube-system   calico-node-456fh                 0/2       ContainerCreating   0          4m
kube-system   calico-node-jh6vd                 0/2       ContainerCreating   0          4m
kube-system   calico-node-sp6w9                 0/2       ContainerCreating   0          4m
kube-system   calicoctl-6dfc585667-24s9h        0/1       Pending             0          4m
kube-system   kube-proxy-46hr5                  1/1       Running             0          7m
kube-system   kube-proxy-l42sk                  1/1       Running             0          7m
kube-system   kube-proxy-p2nbf                  1/1       Running             0          7m
kube-system   kube-proxy-q6qn9                  1/1       Running             0          7m
```

calico正常是下面状态

```bash
$ kubectl get pod --all-namespaces
NAMESPACE     NAME                              READY     STATUS    RESTARTS   AGE
kube-system   calico-node-2hdqf                 2/2       Running   0          4m
kube-system   calico-node-456fh                 2/2       Running   2          4m
kube-system   calico-node-jh6vd                 2/2       Running   0          4m
kube-system   calico-node-sp6w9                 2/2       Running   0          4m
kube-system   calicoctl-6dfc585667-24s9h        1/1       Running   0          4m
kube-system   kube-proxy-46hr5                  1/1       Running   0          8m
kube-system   kube-proxy-l42sk                  1/1       Running   0          8m
kube-system   kube-proxy-p2nbf                  1/1       Running   0          8m
kube-system   kube-proxy-q6qn9                  1/1       Running   0          8m
```

部署后通过下面查看状态即使正常

```bash
$ kubectl -n kube-system get po -l k8s-app=calico-node
NAME                READY     STATUS    RESTARTS   AGE
calico-node-bv7r9   2/2       Running   4          5m
calico-node-cmh2w   2/2       Running   3          5m
calico-node-klzrz   2/2       Running   4          5m
calico-node-n4c9j   2/2       Running   4          5m
```

查找calicoctl的pod名字

```bash
$ kubectl -n kube-system get po -l k8s-app=calicoctl
NAME                         READY     STATUS    RESTARTS   AGE
calicoctl-6b5bf7cb74-d9gv8   1/1       Running   0          5m
```

通过 kubectl exec calicoctl pod 执行命令来检查功能是否正常

```bash
$ kubectl -n kube-system exec calicoctl-6b5bf7cb74-d9gv8 -- calicoctl get profiles -o wide
NAME              LABELS   
kns.default       map[]    
kns.kube-public   map[]    
kns.kube-system   map[]    

$ kubectl -n kube-system exec calicoctl-6b5bf7cb74-d9gv8 -- calicoctl get node -o wide
NAME     ASN         IPV4                 IPV6   
k8s-m1   (unknown)   10.0.6.166/24          
k8s-m2   (unknown)   10.0.6.167/24          
k8s-m3   (unknown)   10.0.6.168/24          
k8s-n1   (unknown)   10.244.3.1/24
```

完成后,通过检查节点是否不再是NotReady,以及 Pod 是否不再是Pending：



### 部署 CoreDNS

1.11 后 CoreDNS 已取代 [Kube DNS](https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/dns) 作为集群服务发现元件,由于 Kubernetes 需要让 Pod 与 Pod 之间能夠互相通信,然而要能够通信需要知道彼此的 IP 才行,而这种做法通常是通过 Kubernetes API 来获取,但是 Pod IP 会因为生命周期变化而改变,因此这种做法无法弹性使用,且还会增加 API Server 负担,基于此问题 Kubernetes 提供了 DNS 服务来作为查询,让 Pod 能夠以 Service 名称作为域名来查询 IP 位址,因此使用者就再不需要关心实际 Pod IP,而 DNS 也会根据 Pod 变化更新资源记录(Record resources)。

CoreDNS 是由 CNCF 维护的开源 DNS 方案,该方案前身是 SkyDNS,其采用了 Caddy 的一部分来开发伺服器框架,使其能够建立一套快速灵活的 DNS,而 CoreDNS 每个功能都可以被当作成一個插件的中介软体,如 Log、Cache、Kubernetes 等功能,甚至能够将源记录存储在 Redis、Etcd 中。

```bash
root@k8s-master1:/usr/local/src/kubernetes/cluster/addons/dns/coredns$ docker tag gcr.io/google-containers/coredns:1.1.3  k8s-harbor1.example.com/baseimages/coredns:1.1.3
root@k8s-master1:/usr/local/src/kubernetes/cluster/addons/dns/coredns$ docker push k8s-harbor1.example.com/baseimages/coredns:1.1.3

cp coredns.yaml.base  coredns.yaml

vim coredns.yaml  #更改镜像地址和ClusterIP

kubectl create -f coredns.yaml

kubectl get pods --all-namespaces
kubectl get services -n kube-system
```

**DNS 测试**

```bash
$ docker tag gcr.io/google-containers/busybox  k8s-harbor1.example.com/baseimages/busybox
$ docker push k8s-harbor1.example.com/baseimages/busybox
```

**yaml 文件**

```yaml
cat busybox.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: default  #default namespace的DNS
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

**创建并验证pod**

```bash
$ kubectl create -f busybox.yaml
$ kubectl  get pods
```

**验证 CorsDNS 解析域名**

```bash
kubectl  get  services --all-namespaces
NAMESPACE     NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
default       kubernetes   ClusterIP   10.1.0.1     <none>        443/TCP         4d
kube-system   kube-dns     ClusterIP   10.1.0.254   <none>        53/UDP,53/TCP   11m

kubectl exec busybox nslookup kubernetes

kubectl exec busybox nslookup kubernetes.default.svc.cluster.local
```















