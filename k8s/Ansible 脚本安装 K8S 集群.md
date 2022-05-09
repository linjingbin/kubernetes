[TOC]

# 简介

使用Ansible脚本安装K8S集群，方便直接，不受国内网络环境影响 [参考](<https://github.com/easzlab/kubeasz>)

项目致力于提供快速部署高可用`k8s`集群的工具, 同时也努力成为`k8s`实践、使用的参考书；基于二进制方式部署和利用`ansible-playbook`实现自动化；既提供一键安装脚本, 也可以根据`安装指南`分步执行安装各个组件。

- **集群特性** `TLS`双向认证、`RBAC`授权、[多Master高可用](https://github.com/easzlab/kubeasz/blob/master/docs/setup/00-planning_and_overall_intro.md#ha-architecture)、支持`Network Policy`、备份恢复、[离线安装](https://github.com/easzlab/kubeasz/blob/master/docs/setup/offline_install.md)
- **集群版本** kubernetes v1.15, v1.16, v1.17, v1.18
- **操作系统** CentOS/RedHat 7, Debian 9/10, Ubuntu 1604/1804
- **运行时** docker 18.06.x-ce, 18.09.x, 19.03.x [containerd](https://github.com/easzlab/kubeasz/blob/master/docs/guide/containerd.md) 1.2.6
- **网络** [calico](https://github.com/easzlab/kubeasz/blob/master/docs/setup/network-plugin/calico.md), [cilium](https://github.com/easzlab/kubeasz/blob/master/docs/setup/network-plugin/cilium.md), [flannel](https://github.com/easzlab/kubeasz/blob/master/docs/setup/network-plugin/flannel.md), [kube-ovn](https://github.com/easzlab/kubeasz/blob/master/docs/setup/network-plugin/kube-ovn.md), [kube-router](https://github.com/easzlab/kubeasz/blob/master/docs/setup/network-plugin/kube-router.md)

# 安装指南

| [00-规划集群和配置介绍](https://github.com/easzlab/kubeasz/blob/master/docs/setup/00-planning_and_overall_intro.md) | [02-安装etcd集群](https://github.com/easzlab/kubeasz/blob/master/docs/setup/02-install_etcd.md) | [04-安装master节点](https://github.com/easzlab/kubeasz/blob/master/docs/setup/04-install_kube_master.md) | [06-安装集群网络](https://github.com/easzlab/kubeasz/blob/master/docs/setup/06-install_network_plugin.md) |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| [01-创建证书和安装准备](https://github.com/easzlab/kubeasz/blob/master/docs/setup/01-CA_and_prerequisite.md) | [03-安装docker服务](https://github.com/easzlab/kubeasz/blob/master/docs/setup/03-install_docker.md) | [05-安装node节点](https://github.com/easzlab/kubeasz/blob/master/docs/setup/05-install_kube_node.md) | [07-安装集群插件](https://github.com/easzlab/kubeasz/blob/master/docs/setup/07-install_cluster_addon.md) |

- 命令行工具 [easzctl介绍](https://github.com/easzlab/kubeasz/blob/master/docs/setup/easzctl_cmd.md)
- 公有云自建集群 [部署指南](https://github.com/easzlab/kubeasz/blob/master/docs/setup/kubeasz_on_public_cloud.md)

## 00-集群规划和基础参数设定

**HA architecture**

![1591672165960](D:\学习资料\笔记\linux\assets\1591672165960.png)

- 注意1：确保各节点时区设置一致、时间同步。推荐集成安装[chrony](https://github.com/easzlab/kubeasz/blob/master/docs/guide/chrony.md)

- 注意2：确保在干净的系统上开始安装，不要使用曾经装过kubeadm或其他k8s发行版的环境

- 注意3：建议操作系统升级到新的稳定内核，请结合阅读[内核升级文档](https://github.com/easzlab/kubeasz/blob/master/docs/guide/kernel_upgrade.md)

  >### Linux Kernel 升级
  >
  >k8s,docker,cilium等很多功能、特性需要较新的linux内核支持，所以有必要在集群部署前对内核进行升级；CentOS7 和 Ubuntu16.04可以很方便的完成内核升级。
  >
  >#### CentOS7
  >
  >红帽企业版 Linux 仓库网站 [https://www.elrepo.org，主要提供各种硬件驱动（显卡、网卡、声卡等）和内核升级相关资源；兼容](https://www.elrepo.org%EF%BC%8C%E4%B8%BB%E8%A6%81%E6%8F%90%E4%BE%9B%E5%90%84%E7%A7%8D%E7%A1%AC%E4%BB%B6%E9%A9%B1%E5%8A%A8%EF%BC%88%E6%98%BE%E5%8D%A1%E3%80%81%E7%BD%91%E5%8D%A1%E3%80%81%E5%A3%B0%E5%8D%A1%E7%AD%89%EF%BC%89%E5%92%8C%E5%86%85%E6%A0%B8%E5%8D%87%E7%BA%A7%E7%9B%B8%E5%85%B3%E8%B5%84%E6%BA%90%EF%BC%9B%E5%85%BC%E5%AE%B9/) CentOS7 内核升级。如下按照网站提示载入elrepo公钥及最新elrepo版本，然后按步骤升级内核（以安装长期支持版本 kernel-lt 为例）
  >
  >```bash
  ># 载入公钥
  >rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
  ># 安装ELRepo
  >rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
  ># 载入elrepo-kernel元数据
  >yum --disablerepo=\* --enablerepo=elrepo-kernel repolist
  ># 查看可用的rpm包
  >yum --disablerepo=\* --enablerepo=elrepo-kernel list kernel*
  ># 安装长期支持版本的kernel
  >yum --disablerepo=\* --enablerepo=elrepo-kernel install -y kernel-lt.x86_64
  ># 删除旧版本工具包
  >yum remove kernel-tools-libs.x86_64 kernel-tools.x86_64 -y
  ># 安装新版本工具包
  >yum --disablerepo=\* --enablerepo=elrepo-kernel install -y kernel-lt-tools.x86_64
  >
  >#查看默认启动顺序
  >awk -F\' '$1=="menuentry " {print $2}' /etc/grub2.cfg  
  >CentOS Linux (4.4.183-1.el7.elrepo.x86_64) 7 (Core)  
  >CentOS Linux (3.10.0-327.10.1.el7.x86_64) 7 (Core)  
  >CentOS Linux (0-rescue-c52097a1078c403da03b8eddeac5080b) 7 (Core)
  >#默认启动的顺序是从0开始，新内核是从头插入（目前位置在0，而4.4.4的是在1），所以需要选择0。
  >grub2-set-default 0  
  >#重启并检查
  >reboot
  >```
  >
  >
  >
  >#### Ubuntu16.04
  >
  >```bash
  >打开 http://kernel.ubuntu.com/~kernel-ppa/mainline/ 并选择列表中选择你需要的版本（以4.16.3为例）。
  >接下来，根据你的系统架构下载 如下.deb 文件：
  >Build for amd64 succeeded (see BUILD.LOG.amd64):
  >  linux-headers-4.16.3-041603_4.16.3-041603.201804190730_all.deb
  >  linux-headers-4.16.3-041603-generic_4.16.3-041603.201804190730_amd64.deb
  >  linux-image-4.16.3-041603-generic_4.16.3-041603.201804190730_amd64.deb
  >#安装后重启即可
  >$ sudo dpkg -i *.deb
  >```

- 注意4：在公有云上创建多主集群，请结合阅读[在公有云上部署 kubeasz](https://github.com/easzlab/kubeasz/blob/master/docs/setup/kubeasz_on_public_cloud.md)



### 高可用集群所需节点配置如下

| 角色       | 数量 | 描述                                                     |
| ---------- | ---- | -------------------------------------------------------- |
| 管理节点   | 1    | 运行ansible/easzctl脚本，一般复用master节点              |
| etcd节点   | 3    | 注意etcd集群需要1,3,5,7...奇数个节点，一般复用master节点 |
| master节点 | 2    | 高可用集群至少2个master节点                              |
| node节点   | 3    | 运行应用负载的节点，可根据需要提升机器配置/增加节点数    |

在 kubeasz 2x 版本，多节点高可用集群安装可以使用2种方式

- 1.先部署单节点集群 [AllinOne部署](https://github.com/easzlab/kubeasz/blob/master/docs/setup/quickStart.md)，然后通过 [节点添加](https://github.com/easzlab/kubeasz/blob/master/docs/op/op-index.md) 扩容成高可用集群
- 2.按照如下步骤先规划准备，直接安装多节点高可用集群



### 部署步骤

按照`example/hosts.multi-node`示例的节点配置，准备4台虚机，搭建一个多主高可用集群。



#### 1.基础系统配置

- 推荐内存2G/硬盘30G以上
- 最小化安装`Ubuntu 16.04 server`或者`CentOS 7 Minimal`
- 配置基础网络、更新源、SSH登录等



#### 2.在每个节点安装依赖工具

Ubuntu 16.04 请执行以下脚本:

```bash
# 文档中脚本默认均以root用户执行
$ apt-get update && apt-get upgrade -y && apt-get dist-upgrade -y
# 安装python2
$ apt-get install python2.7
# Ubuntu16.04可能需要配置以下软连接
$ ln -s /usr/bin/python2.7 /usr/bin/python
```

CentOS 7 请执行以下脚本：

```bash
# 文档中脚本默认均以root用户执行
$ yum update
# 安装python
$ yum install python -y
```



#### 3.在ansible控制端安装及准备ansible

- 3.1 pip 安装 ansible（如果 Ubuntu pip报错)，请看[附录](https://github.com/easzlab/kubeasz/blob/master/docs/setup/00-planning_and_overall_intro.md#Appendix)

```bash
# Ubuntu 16.04 
apt-get install git python-pip -y
# CentOS 7
$ yum install git python-pip -y
# pip安装ansible(国内如果安装太慢可以直接用pip阿里云加速)
$ pip install pip --upgrade -i https://mirrors.aliyun.com/pypi/simple/
$ pip install ansible==2.6.18 netaddr==0.7.19 -i https://mirrors.aliyun.com/pypi/simple/
```

- 3.2 在ansible控制端配置免密码登录

```bash
# 更安全 Ed25519 算法
$ ssh-keygen -t ed25519 -N '' -f ~/.ssh/id_ed25519
# 或者传统 RSA 算法
$ ssh-keygen -t rsa -b 2048 -N '' -f ~/.ssh/id_rsa

$ ssh-copy-id $IPs #$IPs为所有节点地址包括自身，按照提示输入yes 和root密码
```



#### 4.在ansible控制端编排k8s安装

- 4.0 下载项目源码
- 4.1 下载二进制文件
- 4.2 下载离线docker镜像

推荐使用 easzup 脚本下载 4.0/4.1/4.2 所需文件；运行成功后，所有文件（kubeasz代码、二进制、离线镜像）均已整理好放入目录`/etc/ansible`

```bash
# 下载工具脚本easzup，举例使用kubeasz版本2.0.2
export release=2.2.1
curl -C- -fLO --retry 3 https://github.com/easzlab/kubeasz/releases/download/${release}/easzup
chmod +x ./easzup
# 使用工具脚本下载
./easzup -D
```

- 4.3 配置集群参数
  - 4.3.1 必要配置：`cd /etc/ansible && cp example/hosts.multi-node hosts`, 然后实际情况修改此hosts文件
  - 4.3.2 可选配置，初次使用可以不做修改，详见[配置指南](https://github.com/easzlab/kubeasz/blob/master/docs/setup/config_guide.md)
  - 4.3.3 验证 ansible 安装：`ansible all -m ping` 正常能看到节点返回 SUCCESS

- 4.4 通过ansible-playbook 一键式部署安装

  ```bash
  # 分步安装
  ansible-playbook 01.prepare.yml
  ansible-playbook 02.etcd.yml
  ansible-playbook 03.docker.yml
  ansible-playbook 04.kube-master.yml
  ansible-playbook 05.kube-node.yml
  ansible-playbook 06.network.yml
  ansible-playbook 07.cluster-addon.yml
  
  # 一步安装
  ansible-playbook 90.setup.yml
  ```

- [可选]对集群所有节点进行操作系统层面的安全加固 `ansible-playbook roles/os`

- `harden/os-harden.yml`，详情请参考[os-harden项目](https://github.com/dev-sec/ansible-os-hardening)



## 01-创建证书和环境准备

本步骤[01.prepare.yml](https://github.com/easzlab/kubeasz/blob/master/01.prepare.yml)主要完成:

- [chrony role](https://github.com/easzlab/kubeasz/blob/master/docs/guide/chrony.md): 集群节点时间同步[可选]
- deploy role: 创建CA证书、集群组件访问apiserver所需的各种kubeconfig
- prepare role: 系统基础环境配置、分发CA证书、kubectl客户端安装

#### deploy 角色

[roles/deploy/tasks/main.yml](https://github.com/easzlab/kubeasz/blob/master/roles/deploy/tasks/main.yml) 文件

```json
vim /etc/ansible/roles/deploy/tasks/main.yml

- name: prepare some dirs
  file: name={{ item }} state=directory
  with_items:
  - "{{ base_dir }}/.cluster/ssl"
  - "{{ base_dir }}/.cluster/backup"

- name: 本地设置 bin 目录权限
  file: path={{ base_dir }}/bin state=directory mode=0755 recurse=yes

# 注册变量p，根据p的stat信息判断是否已经生成过ca证书，如果没有，下一步生成证书
# 如果已经有ca证书，为了保证整个安装的幂等性，跳过证书生成的步骤
- name: 读取ca证书stat信息
  stat: path="{{ base_dir }}/.cluster/ssl/ca.pem"
  register: p

- name: 准备CA配置文件和签名请求
  template: src={{ item }}.j2 dest={{ base_dir }}/.cluster/ssl/{{ item }}
  with_items:
  - "ca-config.json"
  - "ca-csr.json"
  when: p.stat.isreg is not defined

- name: 生成 CA 证书和私钥
  when: p.stat.isreg is not defined
  shell: "cd {{ base_dir }}/.cluster/ssl && \
	 {{ base_dir }}/bin/cfssl gencert -initca ca-csr.json | {{ base_dir }}/bin/cfssljson -bare ca" 

#----------- 创建kubectl kubeconfig文件: /root/.kube/config
- block:
    - name: 删除原有kubeconfig
      file: path=/root/.kube/config state=absent
      ignore_errors: true
    
    - name: 下载 group:read rbac 文件
      copy: src=read-group-rbac.yaml dest=/tmp/read-group-rbac.yaml
      when: USER_NAME == "read"
    
    - name: 创建group:read rbac 绑定
      shell: "{{ base_dir }}/bin/kubectl apply -f /tmp/read-group-rbac.yaml"
      when: USER_NAME == "read"
    
    - name: 准备kubectl使用的{{ USER_NAME }}证书签名请求
      template: src={{ USER_NAME }}-csr.json.j2 dest={{ base_dir }}/.cluster/ssl/{{ USER_NAME }}-csr.json
    
    - name: 创建{{ USER_NAME }}证书与私钥
      shell: "cd {{ base_dir }}/.cluster/ssl && {{ base_dir }}/bin/cfssl gencert \
            -ca=ca.pem \
            -ca-key=ca-key.pem \
            -config=ca-config.json \
            -profile=kubernetes {{ USER_NAME }}-csr.json | {{ base_dir }}/bin/cfssljson -bare {{ USER_NAME }}"
    
    - name: 设置集群参数
      shell: "{{ base_dir }}/bin/kubectl config set-cluster {{ CLUSTER_NAME }} \
            --certificate-authority={{ base_dir }}/.cluster/ssl/ca.pem \
            --embed-certs=true \
            --server={{ KUBE_APISERVER }}"
    
    - name: 设置客户端认证参数
      shell: "{{ base_dir }}/bin/kubectl config set-credentials {{ USER_NAME }} \
            --client-certificate={{ base_dir }}/.cluster/ssl/{{ USER_NAME }}.pem \
            --embed-certs=true \
            --client-key={{ base_dir }}/.cluster/ssl/{{ USER_NAME }}-key.pem"
    
    - name: 设置上下文参数
      shell: "{{ base_dir }}/bin/kubectl config set-context {{ CONTEXT_NAME }} \
            --cluster={{ CLUSTER_NAME }} --user={{ USER_NAME }}"
    
    - name: 选择默认上下文
      shell: "{{ base_dir }}/bin/kubectl config use-context {{ CONTEXT_NAME }}"
  tags: create_kctl_cfg

#------------创建kube-proxy配置文件: kube-proxy.kubeconfig
- name: 准备kube-proxy 证书签名请求
  template: src=kube-proxy-csr.json.j2 dest={{ base_dir }}/.cluster/ssl/kube-proxy-csr.json

- name: 创建 kube-proxy证书与私钥
  shell: "cd {{ base_dir }}/.cluster/ssl && {{ base_dir }}/bin/cfssl gencert \
        -ca=ca.pem \
        -ca-key=ca-key.pem \
        -config=ca-config.json \
        -profile=kubernetes kube-proxy-csr.json | {{ base_dir }}/bin/cfssljson -bare kube-proxy"

- name: 设置集群参数
  shell: "{{ base_dir }}/bin/kubectl config set-cluster kubernetes \
        --certificate-authority={{ base_dir }}/.cluster/ssl/ca.pem \
        --embed-certs=true \
        --server={{ KUBE_APISERVER }} \
        --kubeconfig={{ base_dir }}/.cluster/kube-proxy.kubeconfig"
- name: 设置客户端认证参数
  shell: "{{ base_dir }}/bin/kubectl config set-credentials kube-proxy \
        --client-certificate={{ base_dir }}/.cluster/ssl/kube-proxy.pem \
        --client-key={{ base_dir }}/.cluster/ssl/kube-proxy-key.pem \
        --embed-certs=true \
        --kubeconfig={{ base_dir }}/.cluster/kube-proxy.kubeconfig"
- name: 设置上下文参数
  shell: "{{ base_dir }}/bin/kubectl config set-context default \
        --cluster=kubernetes \
        --user=kube-proxy \
        --kubeconfig={{ base_dir }}/.cluster/kube-proxy.kubeconfig"
- name: 选择默认上下文
  shell: "{{ base_dir }}/bin/kubectl config use-context default \
	--kubeconfig={{ base_dir }}/.cluster/kube-proxy.kubeconfig"

- name: 本地创建 easzctl 工具的软连接
  file: src={{ base_dir }}/tools/easzctl dest=/usr/bin/easzctl state=link

# ansible 控制端一些易用性配置
# 注册变量以判断是否容器化运行ansible控制端，如果容器化运行那么进程数小于20
- name: 注册变量以判断是否容器化运行ansible控制端
  shell: "ps aux|wc -l"
  register: procs

- name: ansible 控制端写入环境变量$PATH
  lineinfile:
    dest: ~/.bashrc
    state: present
    regexp: 'kubeasz'
    line: 'export PATH={{ base_dir }}/bin/:$PATH # generated by kubeasz'
  when: "procs.stdout|int > 50"
  ignore_errors: true

- name: ansible 控制端添加 kubectl 自动补全
  lineinfile:
    dest: ~/.bashrc
    state: present
    regexp: 'kubectl completion'
    line: 'source <(kubectl completion bash)'
  when: "procs.stdout|int > 50"
  ignore_errors: true

- name: ansible 控制端创建 kubectl 软链接
  file: src={{ base_dir }}/bin/kubectl dest=/usr/bin/kubectl state=link
  ignore_errors: true
```



##### 创建 CA 证书

```bash
$ tree roles/deploy/
├── defaults
│   └── main.yml		# 配置文件：证书有效期，kubeconfig 相关配置
├── files
│   └── read-group-rbac.yaml	# 只读用户的 rbac 权限配置
├── tasks
│   └── main.yml		# 主任务脚本
└── templates
    ├── admin-csr.json.j2	# kubectl客户端使用的admin证书请求模板
    ├── ca-config.json.j2	# ca 配置文件模板
    ├── ca-csr.json.j2		# ca 证书签名请求模板
    ├── kube-proxy-csr.json.j2  # kube-proxy使用的证书请求模板
    └── read-csr.json.j2        # kubectl客户端使用的只读证书请求模板
```

kubernetes 系统各组件需要使用 TLS 证书对通信进行加密，使用 CloudFlare 的 PKI 工具集生成自签名的 CA 证书，用来签名后续创建的其它 TLS 证书。[参考阅读](https://coreos.com/os/docs/latest/generate-self-signed-certificates.html)

根据认证对象可以将证书分成三类：服务器证书`server cert`，客户端证书`client cert`，对等证书`peer cert`(表示既是`server cert`又是`client cert`)，在kubernetes 集群中需要的证书种类如下：

- `etcd` 节点需要标识自己服务的`server cert`，也需要`client cert`与`etcd`集群其他节点交互，当然可以分别指定2个证书，为方便这里使用一个对等证书
- `master` 节点需要标识 apiserver服务的`server cert`，也需要`client cert`连接`etcd`集群，这里也使用一个对等证书
- `kubectl` `calico` `kube-proxy` 只需要`client cert`，因此证书请求中 `hosts` 字段可以为空
- `kubelet` 需要标识自己服务的`server cert`，也需要`client cert`请求`apiserver`，也使用一个对等证书

整个集群要使用统一的CA 证书，只需要在ansible控制端创建，然后分发给其他节点；为了保证安装的幂等性，如果已经存在CA 证书，就跳过创建CA 步骤

##### 创建 CA 配置文件 [ca-config.json.j2](https://github.com/easzlab/kubeasz/blob/master/roles/deploy/templates/ca-config.json.j2)

```json
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

- `signing`：表示该证书可用于签名其它证书；生成的 ca.pem 证书中 `CA=TRUE`；
- `server auth`：表示可以用该 CA 对 server 提供的证书进行验证；
- `client auth`：表示可以用该 CA 对 client 提供的证书进行验证；
- `profile kubernetes` 包含了`server auth`和`client auth`，所以可以签发三种不同类型证书



##### 创建 CA 证书签名请求 [ca-csr.json.j2](https://github.com/easzlab/kubeasz/blob/master/roles/deploy/templates/ca-csr.json.j2)

```json
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "HangZhou",
      "L": "XS",
      "O": "k8s",
      "OU": "System"
    }
  ],
  "ca": {
    "expiry": "876000h"
  }
}
```



##### 生成CA 证书和私钥

```bash
$ cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```



#### 生成 kubeconfig 配置文件

kubectl使用`~/.kube/config` 配置文件与kube-apiserver进行交互，且拥有管理 K8S集群的完全权限，准备kubectl使用的admin 证书签名请求 [admin-csr.json.j2](https://github.com/easzlab/kubeasz/blob/master/roles/deploy/templates/admin-csr.json.j2)

```json
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
      "ST": "HangZhou",
      "L": "XS",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
```

- kubectl 使用客户端证书可以不指定hosts 字段
- 证书请求中 `O` 指定该证书的 Group 为 `system:masters`，而 `RBAC` 预定义的 `ClusterRoleBinding` 将 Group `system:masters` 与 ClusterRole `cluster-admin` 绑定，这就赋予了kubectl**所有集群权限**

```bash
$ kubectl describe clusterrolebinding cluster-admin
Name:         cluster-admin
Labels:       kubernetes.io/bootstrapping=rbac-defaults
Annotations:  rbac.authorization.kubernetes.io/autoupdate=true
Role:
  Kind:  ClusterRole
  Name:  cluster-admin
Subjects:
  Kind   Name            Namespace
  ----   ----            ---------
  Group  system:masters  
```



##### 生成 admin 用户证书

```bash
$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin
```



##### 生成 ~/.kube/config 配置文件

使用`kubectl config` 生成kubeconfig 自动保存到 `~/.kube/config`，生成后 `cat ~/.kube/config`可以验证配置文件包含 kube-apiserver 地址、证书、用户名等信息

```bash
$ kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=127.0.0.1:8443
$ kubectl config set-credentials admin --client-certificate=admin.pem --embed-certs=true --client-key=admin-key.pem
$ kubectl config set-context kubernetes --cluster=kubernetes --user=admin
$ kubectl config use-context kubernetes
```



#### 生成 kube-proxy.kubeconfig 配置文件

创建 kube-proxy 证书请求

```json
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
      "ST": "HangZhou",
      "L": "XS",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```

- kube-proxy 使用客户端证书可以不指定hosts 字段
- CN 指定该证书的 User 为 system:kube-proxy，预定义的 ClusterRoleBinding system:node-proxier 将User system:kube-proxy 与 Role system:node-proxier 绑定，授予了调用 kube-apiserver Proxy 相关 API 的权限；

```bash
$ kubectl describe clusterrolebinding system:node-proxier
Name:         system:node-proxier
Labels:       kubernetes.io/bootstrapping=rbac-defaults
Annotations:  rbac.authorization.kubernetes.io/autoupdate=true
Role:
  Kind:  ClusterRole
  Name:  system:node-proxier
Subjects:
  Kind  Name               Namespace
  ----  ----               ---------
  User  system:kube-proxy  
```

##### 生成 system:kube-proxy 用户证书

```bash
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy
```

##### 生成 kube-proxy.kubeconfig

使用`kubectl config` 生成kubeconfig 自动保存到 kube-proxy.kubeconfig

```bash
$ kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=127.0.0.1:8443 --kubeconfig=kube-proxy.kubeconfig
$ kubectl config set-credentials kube-proxy --client-certificate=kube-proxy.pem --embed-certs=true --client-key=kube-proxy-key.pem --kubeconfig=kube-proxy.kubeconfig
$ kubectl config set-context default --cluster=kubernetes --user=kube-proxy --kubeconfig=kube-proxy.kubeconfig
$ kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```



#### prepare 角色

[roles/prepare/tasks/main.yml](https://github.com/easzlab/kubeasz/blob/master/roles/prepare/tasks/main.yml) 文件，比较简单直观

1. 首先设置基础操作系统软件和系统参数，请阅读脚本中的注释内容
2. 首先创建一些基础文件目录
3. 把证书工具 CFSSL 下发到指定节点，并下发kubeconfig配置文件
4. 把CA 证书相关下发到指定节点的 {{ ca_dir }} 目录

```json
cat roles/prepare/tasks/main.yml 
# 系统基础软件环境
- name: apt更新缓存刷新
  apt: update_cache=yes cache_valid_time=72000
  ignore_errors: true
  when:
  - 'ansible_distribution in ["Ubuntu","Debian"]'
  - 'INSTALL_SOURCE != "offline"'

- import_tasks: ubuntu.yml
  when: 'ansible_distribution in ["Ubuntu","Debian"]'

- import_tasks: centos.yml
  when: 'ansible_distribution in ["CentOS","RedHat","Amazon"]' 

# 公共系统参数设置
- import_tasks: common.yml

- name: prepare some dirs
  file: name={{ item }} state=directory
  with_items:
  - "{{ bin_dir }}"
  - "{{ ca_dir }}"
  - /root/.kube

- name: 分发证书工具 CFSSL
  copy: src={{ base_dir }}/bin/{{ item }} dest={{ bin_dir }}/{{ item }} mode=0755
  with_items:
  - cfssl
  - cfssl-certinfo
  - cfssljson

- name: 写入环境变量$PATH 
  lineinfile:
    dest: ~/.bashrc
    state: present
    regexp: 'kubeasz'
    line: 'export PATH={{ bin_dir }}:$PATH # generated by kubeasz'

- block:
    - name: 分发证书相关
      copy: src={{ base_dir }}/.cluster/ssl/{{ item }} dest={{ ca_dir }}/{{ item }}
      with_items:
      - admin.pem
      - admin-key.pem
      - ca.pem
      - ca-key.pem
      - ca-config.json

    - name: 添加 kubectl 命令自动补全
      lineinfile:
        dest: ~/.bashrc
        state: present
        regexp: 'kubectl completion'
        line: 'source <(kubectl completion bash)'

    - name: 分发 kubeconfig配置文件
      copy: src=/root/.kube/config dest=/root/.kube/config

    - name: 分发 kube-proxy.kubeconfig配置文件
      copy: src={{ base_dir }}/.cluster/kube-proxy.kubeconfig dest=/etc/kubernetes/kube-proxy.kubeconfig
  when: "inventory_hostname in groups['kube-master'] or inventory_hostname in groups['kube-node']"
```



## 02-安装etcd集群

kuberntes 集群使用 etcd 存储所有数据，是最重要的组件之一，注意 etcd集群需要奇数个节点(1,3,5...)，本文档使用3个节点做集群。

[roles/etcd/tasks/main.yml](https://github.com/easzlab/kubeasz/blob/master/roles/etcd/tasks/main.yml) 

```bash
[root@k8s-1 ansible]# cat roles/etcd/tasks/main.yml 
- name: prepare some dirs
  file: name={{ item }} state=directory
  with_items:
  - "{{ bin_dir }}"
  - "{{ ca_dir }}"
  - "/etc/etcd/ssl"    # etcd 证书目录
  - "/var/lib/etcd"    # etcd 工作目录

- name: 下载etcd二进制文件
  copy: src={{ base_dir }}/bin/{{ item }} dest={{ bin_dir }}/{{ item }} mode=0755
  with_items:
  - etcd
  - etcdctl
  tags: upgrade_etcd

- name: 分发证书相关
  copy: src={{ base_dir }}/.cluster/ssl/{{ item }} dest={{ ca_dir }}/{{ item }}
  with_items:
  - ca.pem
  - ca-key.pem
  - ca-config.json

- name: 创建etcd证书请求
  template: src=etcd-csr.json.j2 dest=/etc/etcd/ssl/etcd-csr.json

- name: 创建 etcd证书和私钥
  shell: "cd /etc/etcd/ssl && {{ bin_dir }}/cfssl gencert \
        -ca={{ ca_dir }}/ca.pem \
        -ca-key={{ ca_dir }}/ca-key.pem \
        -config={{ ca_dir }}/ca-config.json \
        -profile=kubernetes etcd-csr.json | {{ bin_dir }}/cfssljson -bare etcd"

- name: 创建etcd的systemd unit文件
  template: src=etcd.service.j2 dest=/etc/systemd/system/etcd.service
  tags: upgrade_etcd

- name: 开机启用etcd服务
  shell: systemctl enable etcd
  ignore_errors: true

- name: 开启etcd服务
  shell: systemctl daemon-reload && systemctl restart etcd
  ignore_errors: true
  tags: upgrade_etcd

- name: 以轮询的方式等待服务同步完成
  shell: "systemctl status etcd.service|grep Active"
  register: etcd_status
  until: '"running" in etcd_status.stdout'
  retries: 8
  delay: 8
  tags: upgrade_etcd
```



#### 下载etcd/etcdctl 二进制文件、创建证书目录

<https://github.com/etcd-io/etcd/releases>

#### 创建etcd证书请求 [etcd-csr.json.j2](https://github.com/easzlab/kubeasz/blob/master/roles/etcd/templates/etcd-csr.json.j2)

```json
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "{{ inventory_hostname }}"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "HangZhou",
      "L": "XS",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```

etcd使用对等证书，hosts 字段必须指定授权使用该证书的 etcd 节点 IP



#### 创建证书和私钥

```bash
$ cd /etc/etcd/ssl && {{ bin_dir }}/cfssl gencert \
        -ca={{ ca_dir }}/ca.pem \
        -ca-key={{ ca_dir }}/ca-key.pem \
        -config={{ ca_dir }}/ca-config.json \
        -profile=kubernetes etcd-csr.json | {{ bin_dir }}/cfssljson -bare etcd
```



#### 创建etcd 服务文件 [etcd.service.j2](https://github.com/easzlab/kubeasz/blob/master/roles/etcd/templates/etcd.service.j2)

先创建工作目录 /var/lib/etcd/

```bash
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
ExecStart={{ bin_dir }}/etcd \
  --name={{ NODE_NAME }} \
  --cert-file=/etc/etcd/ssl/etcd.pem \
  --key-file=/etc/etcd/ssl/etcd-key.pem \
  --peer-cert-file=/etc/etcd/ssl/etcd.pem \
  --peer-key-file=/etc/etcd/ssl/etcd-key.pem \
  --trusted-ca-file={{ ca_dir }}/ca.pem \
  --peer-trusted-ca-file={{ ca_dir }}/ca.pem \
  --initial-advertise-peer-urls=https://{{ inventory_hostname }}:2380 \
  --listen-peer-urls=https://{{ inventory_hostname }}:2380 \
  --listen-client-urls=https://{{ inventory_hostname }}:2379,http://127.0.0.1:2379 \
  --advertise-client-urls=https://{{ inventory_hostname }}:2379 \
  --initial-cluster-token=etcd-cluster-0 \
  --initial-cluster={{ ETCD_NODES }} \
  --initial-cluster-state=new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

- 完整参数列表请使用 `etcd --help` 查询。
- 注意etcd 即需要服务器证书也需要客户端证书，这里为方便使用一个peer 证书代替两个证书。
- `--initial-cluster-state` 值为 `new` 时，`--name` 的参数值必须位于 `--initial-cluster` 列表中。



#### 启动etcd服务

``` bash
systemctl daemon-reload && systemctl enable etcd && systemctl start etcd
```



#### 验证etcd集群状态

- systemctl status etcd 查看服务状态
- journalctl -u etcd 查看运行日志
- 在任一 etcd 集群节点上执行如下命令

```bash
# 根据hosts中配置设置shell变量 $NODE_IPS
export NODE_IPS="192.168.1.1 192.168.1.2 192.168.1.3"
for ip in ${NODE_IPS}; do
  ETCDCTL_API=3 etcdctl \
  --endpoints=https://${ip}:2379  \
  --cacert=/etc/kubernetes/ssl/ca.pem \
  --cert=/etc/etcd/ssl/etcd.pem \
  --key=/etc/etcd/ssl/etcd-key.pem \
  endpoint health; done
```

预期结果：

```bash
https://192.168.1.1:2379 is healthy: successfully committed proposal: took = 2.210885ms
https://192.168.1.2:2379 is healthy: successfully committed proposal: took = 2.784043ms
https://192.168.1.3:2379 is healthy: successfully committed proposal: took = 3.275709ms
```

三台 etcd 的输出均为 healthy 时表示集群服务正常。



## 03-安装docker服务

```bash
roles/docker/
├── defaults
│   └── main.yml		# 变量配置文件
├── files
│   ├── docker			# bash 自动补全
│   └── docker-tag		# 查询镜像tag的小工具
├── tasks
│   └── main.yml		# 主执行文件
└── templates	
    ├── daemon.json.j2		# docker daemon 配置文件
    └── docker.service.j2	# service 服务模板
```

打开[roles/docker/tasks/main.yml](https://github.com/easzlab/kubeasz/blob/master/roles/docker/tasks/main.yml) 文件，对照看以下讲解内容。

```bash
[root@k8s-1 ansible]# cat roles/docker/tasks/main.yml 
- name: 获取是否已经安装containerd
  shell: 'systemctl status containerd|grep Active || echo "NOT FOUND"'
  register: containerd_status

- name: fail info1
  fail: msg="Containerd already installed!"
  when: '"running" in containerd_status.stdout'

- name: 获取是否运行名为'kubeasz'的容器
  shell: 'systemctl status docker|grep Active && docker ps|grep kubeasz || echo "NOT FOUND"'
  register: install_info
  tags: upgrade_docker, download_docker

# 18.09.x 版本二进制名字有变化，需要做判断
- name: 获取docker版本信息
  shell: "{{ base_dir }}/bin/dockerd --version|cut -d' ' -f3"
  register: docker_ver
  connection: local
  run_once: true
  tags: upgrade_docker, download_docker

- name: 转换docker版本信息为浮点数
  set_fact: 
    DOCKER_VER: "{{ docker_ver.stdout.split('.')[0]|int + docker_ver.stdout.split('.')[1]|int/100 }}"
  tags: upgrade_docker, download_docker

- name: debug info
  debug: var="DOCKER_VER"
  tags: upgrade_docker, download_docker

- block:
    - name: 准备docker相关目录
      file: name={{ item }} state=directory
      with_items:
      - "{{ bin_dir }}"
      - /etc/docker
    
    - name: 下载 docker 二进制文件
      copy: src={{ base_dir }}/bin/{{ item }} dest={{ bin_dir }}/{{ item }} mode=0755
      with_items:
      - docker-containerd
      - docker-containerd-shim
      - docker-init
      - docker-runc
      - docker
      - docker-containerd-ctr
      - dockerd
      - docker-proxy
      tags: upgrade_docker, download_docker
      when: "DOCKER_VER|float < 18.09"
    
    - name: 下载 docker 二进制文件(>= 18.09.x)
      copy: src={{ base_dir }}/bin/{{ item }} dest={{ bin_dir }}/{{ item }} mode=0755
      with_items:
      - containerd
      - containerd-shim
      - docker-init
      - runc
      - docker
      - ctr
      - dockerd
      - docker-proxy
      tags: upgrade_docker, download_docker
      when: "DOCKER_VER|float >= 18.09"
    
    - name: docker命令自动补全
      copy: src=docker dest=/etc/bash_completion.d/docker mode=0644
    
    - name: docker国内镜像加速
      template: src=daemon.json.j2 dest=/etc/docker/daemon.json
    
    - name: flush-iptables
      shell: "iptables -P INPUT ACCEPT \
            && iptables -F && iptables -X \
            && iptables -F -t nat && iptables -X -t nat \
            && iptables -F -t raw && iptables -X -t raw \
            && iptables -F -t mangle && iptables -X -t mangle"
    
    - name: 创建docker的systemd unit文件
      template: src=docker.service.j2 dest=/etc/systemd/system/docker.service
      tags: upgrade_docker, download_docker
    
    - name: 开机启用docker 服务
      shell: systemctl enable docker
      ignore_errors: true
    
    - name: 开启docker 服务
      shell: systemctl daemon-reload && systemctl restart docker
      tags: upgrade_docker
    
    ## 可选 ------安装docker查询镜像 tag的小工具----
    # 先要安装轻量JSON处理程序‘jq’，已在 prepare 节点安装
    - name: 下载 docker-tag
      copy: src=docker-tag dest={{ bin_dir }}/docker-tag mode=0755
    
    - name: 轮询等待docker服务运行
      shell: "systemctl status docker.service|grep Active"
      register: docker_status
      until: '"running" in docker_status.stdout'
      retries: 8
      delay: 2
      tags: upgrade_docker
    
    # 配置 docker 命令软链接，方便单独安装 docker
    - name: 配置 docker 命令软链接
      file: src={{ bin_dir }}/docker dest=/usr/bin/docker state=link
      ignore_errors: true
  when: "'kubeasz' not in install_info.stdout"
```



#### 创建docker的systemd unit文件

```bash
[Unit]
Description=Docker Application Container Engine
Documentation=http://docs.docker.io

[Service]
Environment="PATH={{ bin_dir }}:/bin:/sbin:/usr/bin:/usr/sbin"
ExecStart={{ bin_dir }}/dockerd
ExecStartPost=/sbin/iptables -I FORWARD -s 0.0.0.0/0 -j ACCEPT
ExecReload=/bin/kill -s HUP $MAINPID
Restart=on-failure
RestartSec=5
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
Delegate=yes
KillMode=process

[Install]
WantedBy=multi-user.target
```

- dockerd 运行时会调用其它 docker 命令，如 docker-proxy，所以需要将 docker 命令所在的目录加到 PATH 环境变量中；
- docker 从 1.13 版本开始，将`iptables` 的`filter` 表的`FORWARD` 链的默认策略设置为`DROP`，从而导致 ping 其它 Node 上的 Pod IP 失败，因此必须在 `filter` 表的`FORWARD` 链增加一条默认允许规则 `iptables -I FORWARD -s 0.0.0.0/0 -j ACCEPT`
- 运行`dockerd --help` 查看所有可配置参数，确保默认开启 `--iptables` 和 `--ip-masq` 选项



#### 配置国内镜像加速

从国内下载docker官方仓库镜像非常缓慢，所以对于k8s集群来说配置镜像加速非常重要，配置 `/etc/docker/daemon.json`

```json
{
  "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn"],
  "max-concurrent-downloads": 10,
  "log-driver": "json-file",
  "log-level": "warn",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
    }
}
```

这将在后续部署calico下载 calico/node镜像和kubedns/heapster/dashboard镜像时起到加速效果。

由于K8S的官方镜像存放在`gcr.io`仓库，因此这个镜像加速对K8S的官方镜像没有效果；好在`Docker Hub`上有很多K8S镜像的转存，而`Docker Hub`上的镜像可以加速。这里推荐两个K8S镜像的`Docker Hub`项目。

当然对于企业内部应用的docker镜像，想要在K8S平台运行的话，特别是结合开发`CI/CD` 流程，肯定是需要部署私有镜像仓库的，参阅[harbor文档](https://github.com/easzlab/kubeasz/blob/master/docs/guide/harbor.md)。

- [mirrorgooglecontainers](https://hub.docker.com/u/mirrorgooglecontainers/)
- [anjia0532](https://hub.docker.com/u/anjia0532/), [项目github地址](https://github.com/anjia0532/gcr.io_mirror)

另外，daemon.json配置中也配置了docker 容器日志相关参数，设置单个容器日志超过10M则进行回卷，回卷的副本数超过3个就进行清理。



#### 清理 iptables

因为后续`calico`网络、`kube-proxy`等将大量使用 iptables规则，安装前清空所有`iptables`策略规则；常见发行版`Ubuntu`的 `ufw` 和 `CentOS`的 `firewalld`等基于`iptables`的防火墙最好直接卸载，避免不必要的冲突。

```bash
iptables -F && iptables -X \
        && iptables -F -t nat && iptables -X -t nat \
        && iptables -F -t raw && iptables -X -t raw \
        && iptables -F -t mangle && iptables -X -t mangle
```

calico 网络支持 `network-policy`，使用的`calico-kube-controllers` 会使用到`iptables` 所有的四个表 `filter` `nat` `raw` `mangle`，所以一并清理



#### 启动 docker

```bash
systemctl daemon-reload && systemctl enable docker && systemctl start docker
```



#### 可选-安装docker查询镜像 tag的小工具

docker官方目前没有提供在命令行直接查询某个镜像的tag信息的方式，网上找来一个脚本工具，使用很方便。

```bash
$ docker-tag library/ubuntu
"14.04"
"16.04"
"17.04"
"latest"
"trusty"
"trusty-20171117"
"xenial"
"xenial-20171114"
"zesty"
"zesty-20171114"
$ docker-tag mirrorgooglecontainers/kubernetes-dashboard-amd64
"v0.1.0"
"v1.0.0"
"v1.0.0-beta1"
"v1.0.1"
"v1.1.0-beta1"
"v1.1.0-beta2"
"v1.1.0-beta3"
"v1.7.0"
"v1.7.1"
"v1.8.0"
```

- 需要先apt安装轻量JSON处理程序 `jq`
- 然后下载脚本即可使用
- 脚本很简单，就一行命令如下

```bash
#!/bin/bash
curl -s -S "https://registry.hub.docker.com/v2/repositories/$@/tags/" | jq '."results"[]["name"]' |sort
```

- CentOS7 安装 `jq` 稍微费力一点，需要启用 `EPEL` 源

```bash
$ wget http://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
$ rpm -ivh epel-release-latest-7.noarch.rpm
$ yum install jq
```



#### 验证

运行`ansible-playbook 03.docker.yml` 成功后可以验证

```bash
$ systemctl status docker 	# 服务状态
$ journalctl -u docker 		# 运行日志
$ docker version
$ docker info
```

`iptables-save|grep FORWARD` 查看 iptables filter表 FORWARD链，最后要有一个 `-A FORWARD -j ACCEPT` 保底允许规则

```bash
iptables-save|grep FORWARD
:FORWARD ACCEPT [0:0]
:FORWARD DROP [0:0]
-A FORWARD -j DOCKER-USER
-A FORWARD -j DOCKER-ISOLATION
-A FORWARD -o docker0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -o docker0 -j DOCKER
-A FORWARD -i docker0 ! -o docker0 -j ACCEPT
-A FORWARD -i docker0 -o docker0 -j ACCEPT
-A FORWARD -j ACCEPT
```





## 04-安装kube-master节点

部署master节点主要包含三个组件`apiserver` `scheduler` `controller-manager`，其中：

- apiserver 提供集群管理的REST API 接口，包括认证授权、数据校验以及集群状态变更等
  - 只有API Server 才直接操作 etcd
  - 其他模块通过API Server 查询或修改数据
  - 提供其他模块之间的数据交互和通信的枢纽
- scheduler负责分配调度Pod到集群内的node 节点
  - 监听kube-apiserver，查询还未分配Node 的Pod
  - 根据调度策略为这些Pod分配节点
- controller-manager 由一系列的控制器组成，它通过apiserver监控整个集群的状态，并确保集群处于预期的工作状态

master节点的高可用主要就是实现apiserver组件的高可用，后面我们会看到每个 node 节点会启用 4 层负载均衡来请求 apiserver 服务。

```bash
roles/kube-master/

├── defaults
│   └── main.yml
├── tasks
│   └── main.yml
└── templates
    ├── aggregator-proxy-csr.json.j2
    ├── basic-auth.csv.j2
    ├── basic-auth-rbac.yaml.j2
    ├── kube-apiserver.service.j2
    ├── kube-apiserver-v1.8.service.j2
    ├── kube-controller-manager.service.j2
    ├── kubernetes-csr.json.j2
    └── kube-scheduler.service.j2
```

[roles/kube-master/tasks/main.yml](https://github.com/easzlab/kubeasz/blob/master/roles/kube-master/tasks/main.yml)

```bash
[root@k8s-1 ansible]# cat roles/kube-master/tasks/main.yml 
- name: 下载 kube-master 二进制
  copy: src={{ base_dir }}/bin/{{ item }} dest={{ bin_dir }}/{{ item }} mode=0755
  with_items:
  - kube-apiserver
  - kube-controller-manager
  - kube-scheduler
  - kubectl
  tags: upgrade_k8s

# 设置 kubernetes svc ip (一般是 SERVICE_CIDR 中第一个IP)
- name: 注册变量 KUBERNETES_SVC_IP
  shell: echo {{ SERVICE_CIDR }}|cut -d/ -f1|awk -F. '{print $1"."$2"."$3"."$4+1}'
  register: KUBERNETES_SVC_IP
  tags: change_cert

- name: 设置变量 CLUSTER_KUBERNETES_SVC_IP
  set_fact: CLUSTER_KUBERNETES_SVC_IP={{ KUBERNETES_SVC_IP.stdout }}
  tags: change_cert

- name: 创建 kubernetes 证书签名请求
  template: src=kubernetes-csr.json.j2 dest={{ ca_dir }}/kubernetes-csr.json
  tags: change_cert

- name: 创建 kubernetes 证书和私钥
  shell: "cd {{ ca_dir }} && {{ bin_dir }}/cfssl gencert \
        -ca={{ ca_dir }}/ca.pem \
        -ca-key={{ ca_dir }}/ca-key.pem \
        -config={{ ca_dir }}/ca-config.json \
        -profile=kubernetes kubernetes-csr.json | {{ bin_dir }}/cfssljson -bare kubernetes"
  tags: change_cert

# 创建aggregator proxy相关证书
- name: 创建 aggregator proxy证书签名请求
  template: src=aggregator-proxy-csr.json.j2 dest={{ ca_dir }}/aggregator-proxy-csr.json

- name: 创建 aggregator-proxy证书和私钥
  shell: "cd {{ ca_dir }} && {{ bin_dir }}/cfssl gencert \
        -ca={{ ca_dir }}/ca.pem \
        -ca-key={{ ca_dir }}/ca-key.pem \
        -config={{ ca_dir }}/ca-config.json \
        -profile=kubernetes aggregator-proxy-csr.json | {{ bin_dir }}/cfssljson -bare aggregator-proxy"

- block:
  - name: 生成 basic-auth 随机密码
    shell: 'PWD=`date +%s%N|md5sum|head -c16`; \
	sed -i "s/_pwd_/$PWD/g" {{ base_dir }}/roles/kube-master/defaults/main.yml; \
	echo $PWD;'
    connection: local
    register: TMP_PASS
    run_once: true
    
  - name: 设置 basic-auth 随机密码
    set_fact: BASIC_AUTH_PASS={{ TMP_PASS.stdout }}
  when: 'BASIC_AUTH_ENABLE == "yes" and BASIC_AUTH_PASS == "_pwd_"'
  tags: restart_master
  
- name: 创建 basic-auth.csv
  template: src=basic-auth.csv.j2 dest={{ ca_dir }}/basic-auth.csv
  when: 'BASIC_AUTH_ENABLE == "yes"'
  tags: restart_master

- name: 创建 master 服务的 systemd unit 文件
  template: src={{ item }}.j2 dest=/etc/systemd/system/{{ item }}
  with_items:
  - kube-apiserver.service
  - kube-controller-manager.service
  - kube-scheduler.service
  tags: restart_master, upgrade_k8s

# 为兼容v1.8版本，配置不同 kube-apiserver的systemd unit文件
- name: 获取 k8s 版本信息
  shell: "{{ bin_dir }}/kube-apiserver --version"
  register: k8s_ver
  tags: restart_master, upgrade_k8s

- name: 创建kube-apiserver v1.8的systemd unit文件
  template: src=kube-apiserver-v1.8.service.j2 dest=/etc/systemd/system/kube-apiserver.service
  tags: restart_master, upgrade_k8s
  when: "'v1.8' in k8s_ver.stdout"

- name: enable master 服务
  shell: systemctl enable kube-apiserver kube-controller-manager kube-scheduler
  ignore_errors: true

- name: 启动 master 服务
  shell: "systemctl daemon-reload && systemctl restart kube-apiserver && \
	systemctl restart kube-controller-manager && systemctl restart kube-scheduler"
  tags: upgrade_k8s, restart_master

- name: 替换 kubeconfig 的 apiserver 地址
  lineinfile:
    dest: /root/.kube/config
    regexp: "^    server"
    line: "    server: https://{{ inventory_hostname }}:6443"
  tags: upgrade_k8s, restart_master

- name: 以轮询的方式等待master服务启动完成
  command: "{{ bin_dir }}/kubectl get node"
  register: result
  until:    result.rc == 0
  retries:  5
  delay: 6
  tags: upgrade_k8s, restart_master

- name: 配置{{ BASIC_AUTH_USER }}用户rbac权限
  template: src=basic-auth-rbac.yaml.j2 dest=/opt/kube/basic-auth-rbac.yaml
  when: 'BASIC_AUTH_ENABLE == "yes"'
  tags: restart_master

- name: 创建{{ BASIC_AUTH_USER }}用户rbac权限
  shell: "{{ bin_dir }}/kubectl apply -f /opt/kube/basic-auth-rbac.yaml"
  when: 'BASIC_AUTH_ENABLE == "yes"'
  run_once: true
  tags: restart_master
```



#### 创建 kubernetes 证书签名请求

```json
{
  "CN": "kubernetes",
  "hosts": [
    "127.0.0.1",
{% if groups['ex-lb']|length > 0 %}
    "{{ hostvars[groups['ex-lb'][0]]['EX_APISERVER_VIP'] }}",
{% endif %}
    "{{ inventory_hostname }}",
    "{{ CLUSTER_KUBERNETES_SVC_IP }}",
{% for host in MASTER_CERT_HOSTS %}
    "{{ host }}",
{% endfor %}
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
      "ST": "HangZhou",
      "L": "XS",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```

kubernetes 证书既是服务器证书，同时apiserver又作为客户端证书去访问etcd 集群；作为服务器证书需要设置hosts 指定使用该证书的IP 或域名列表，需要注意的是：

- 如果配置 ex-lb，需要把 EX_APISERVER_VIP 也配置进去
- 如果需要外部访问 apiserver，需要在 defaults/main.yml 配置 MASTER_CERT_HOSTS
- `kubectl get svc` 将看到集群中由 api-server 创建的默认服务 `kubernetes`，因此也要把 `kubernetes` 服务名和各个服务域名也添加进去



#### 【可选】创建基础用户名/密码认证配置

若未创建任何基础认证配置，K8S集群部署完毕后访问dashboard将会提示`401`错误。

当前如需创建基础认证，需单独修改`roles/kube-master/defaults/main.yml`文件，将`BASIC_AUTH_ENABLE`改为`yes`，并相应配置用户名`BASIC_AUTH_USER`（默认用户名为`admin`）及密码`BASIC_AUTH_PASS`。



#### 创建apiserver的服务配置文件

```bash
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
ExecStart={{ bin_dir }}/kube-apiserver \
  --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,NodeRestriction,MutatingAdmissionWebhook,ValidatingAdmissionWebhook \
  --bind-address={{ inventory_hostname }} \
  --insecure-bind-address=127.0.0.1 \
  --authorization-mode=Node,RBAC \
  --kubelet-https=true \
  --kubelet-client-certificate={{ ca_dir }}/admin.pem \
  --kubelet-client-key={{ ca_dir }}/admin-key.pem \
  --anonymous-auth=false \
  --basic-auth-file={{ ca_dir }}/basic-auth.csv \
  --service-cluster-ip-range={{ SERVICE_CIDR }} \
  --service-node-port-range={{ NODE_PORT_RANGE }} \
  --tls-cert-file={{ ca_dir }}/kubernetes.pem \
  --tls-private-key-file={{ ca_dir }}/kubernetes-key.pem \
  --client-ca-file={{ ca_dir }}/ca.pem \
  --service-account-key-file={{ ca_dir }}/ca-key.pem \
  --etcd-cafile={{ ca_dir }}/ca.pem \
  --etcd-certfile={{ ca_dir }}/kubernetes.pem \
  --etcd-keyfile={{ ca_dir }}/kubernetes-key.pem \
  --etcd-servers={{ ETCD_ENDPOINTS }} \
  --enable-swagger-ui=true \
  --endpoint-reconciler-type=lease \
  --allow-privileged=true \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=3 \
  --audit-log-maxsize=100 \
  --audit-log-path=/var/lib/audit.log \
  --event-ttl=1h \
  --requestheader-client-ca-file={{ ca_dir }}/ca.pem \
  --requestheader-allowed-names= \
  --requestheader-extra-headers-prefix=X-Remote-Extra- \
  --requestheader-group-headers=X-Remote-Group \
  --requestheader-username-headers=X-Remote-User \
  --proxy-client-cert-file={{ ca_dir }}/aggregator-proxy.pem \
  --proxy-client-key-file={{ ca_dir }}/aggregator-proxy-key.pem \
  --enable-aggregator-routing=true \
  --runtime-config=batch/v2alpha1=true \
  --v=2
Restart=on-failure
RestartSec=5
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

- Kubernetes 对 API 访问需要依次经过认证、授权和准入控制(admission controll)，认证解决用户是谁的问题，授权解决用户能做什么的问题，Admission Control则是资源管理方面的作用。
- 支持同时提供https（默认监听在6443端口）和http API（默认监听在127.0.0.1的8080端口），其中http API是非安全接口，不做任何认证授权机制，kube-scheduler、kube-controller-manager 一般和 kube-apiserver 部署在同一台机器上，它们使用非安全端口和 kube-apiserver通信; 其他集群外部就使用HTTPS访问 apiserver
- 关于authorization-mode=Node,RBAC v1.7+支持Node授权，配合NodeRestriction准入控制来限制kubelet仅可访问node、endpoint、pod、service以及secret、configmap、PV和PVC等相关的资源；需要注意的是v1.7中Node 授权是默认开启的，v1.8中需要显式配置开启，否则 Node无法正常工作
- 缺省情况下 kubernetes 对象保存在 etcd /registry 路径下，可以通过 --etcd-prefix 参数进行调整
- 详细参数配置请参考`kube-apiserver --help`，关于认证、授权和准入控制请[阅读](https://github.com/feiskyer/kubernetes-handbook/blob/master/components/apiserver.md)
- 增加了访问kubelet使用的证书配置，防止匿名访问kubelet的安全漏洞，详见[漏洞说明](https://github.com/easzlab/kubeasz/blob/master/docs/mixes/01.fix_kubelet_annoymous_access.md)



#### 创建controller-manager 的服务文件

```bash
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart={{ bin_dir }}/kube-controller-manager \
  --address=127.0.0.1 \
  --master=http://127.0.0.1:8080 \
  --allocate-node-cidrs=true \
  --service-cluster-ip-range={{ SERVICE_CIDR }} \
  --cluster-cidr={{ CLUSTER_CIDR }} \
  --cluster-name=kubernetes \
  --cluster-signing-cert-file={{ ca_dir }}/ca.pem \
  --cluster-signing-key-file={{ ca_dir }}/ca-key.pem \
  --service-account-private-key-file={{ ca_dir }}/ca-key.pem \
  --root-ca-file={{ ca_dir }}/ca.pem \
  --horizontal-pod-autoscaler-use-rest-clients=true \
  --leader-elect=true \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

- --address 值必须为 127.0.0.1，因为当前 kube-apiserver 期望 scheduler 和 controller-manager 在同一台机器
- --master=[http://127.0.0.1:8080](http://127.0.0.1:8080/) 使用非安全 8080 端口与 kube-apiserver 通信
- --cluster-cidr 指定 Cluster 中 Pod 的 CIDR 范围，该网段在各 Node 间必须路由可达(calico 实现)
- --service-cluster-ip-range 参数指定 Cluster 中 Service 的CIDR范围，必须和 kube-apiserver 中的参数一致
- --cluster-signing-* 指定的证书和私钥文件用来签名为 TLS BootStrap 创建的证书和私钥
- --root-ca-file 用来对 kube-apiserver 证书进行校验，指定该参数后，才会在Pod 容器的 ServiceAccount 中放置该 CA 证书文件
- --leader-elect=true 使用多节点选主的方式选择主节点。只有主节点才会启动所有控制器，而其他从节点则仅执行选主算法



#### 创建scheduler 的服务文件

```bash
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart={{ bin_dir }}/kube-scheduler \
  --address=127.0.0.1 \
  --master=http://127.0.0.1:8080 \
  --leader-elect=true \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

- --address 同样值必须为 127.0.0.1
- --master=[http://127.0.0.1:8080](http://127.0.0.1:8080/) 使用非安全 8080 端口与 kube-apiserver 通信
- --leader-elect=true 部署多台机器组成的 master 集群时选举产生一个处于工作状态的 kube-controller-manager 进程



#### 在master 节点安装 node 服务: kubelet kube-proxy

项目master 分支使用 DaemonSet 方式安装网络插件，如果master 节点不安装 kubelet 服务是无法安装网络插件的，如果 master 节点不安装网络插件，那么通过`apiserver` 方式无法访问 `dashboard` `kibana`等管理界面，[ISSUES #130](https://github.com/easzlab/kubeasz/issues/130)

```bash
# vi 04.kube-master.yml
- hosts: kube-master
  roles:
  - kube-master
  - kube-node
  # 禁止业务 pod调度到 master节点
  tasks:
  - name: 禁止业务 pod调度到 master节点
    shell: "{{ bin_dir }}/kubectl cordon {{ inventory_hostname }} "
    when: DEPLOY_MODE != "allinone"
    ignore_errors: true
```

在master 节点也同时成为 node 节点后，默认业务 POD也会调度到 master节点，多主模式下这显然增加了 master节点的负载，因此可以使用 `kubectl cordon`命令禁止业务 POD调度到 master节点



#### master 集群的验证

运行 `ansible-playbook 04.kube-master.yml` 成功后，验证 master节点的主要组件：

```bash
# 查看进程状态
systemctl status kube-apiserver
systemctl status kube-controller-manager
systemctl status kube-scheduler
# 查看进程运行日志
journalctl -u kube-apiserver
journalctl -u kube-controller-manager
journalctl -u kube-scheduler
```

执行 `kubectl get componentstatus` 可以看到

```bash
NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok                   
controller-manager   Healthy   ok                   
etcd-0               Healthy   {"health": "true"}   
etcd-2               Healthy   {"health": "true"}   
etcd-1               Healthy   {"health": "true"} 
```





## 05-安装kube-node节点

`kube-node` 是集群中运行工作负载的节点，前置条件需要先部署好`kube-master`节点，它需要部署如下组件：

- docker：运行容器
- kubelet： kube-node上最主要的组件
- kube-proxy： 发布应用服务与负载均衡
- haproxy：用于请求转发到多个 apiserver，详见[HA-2x 架构](https://github.com/easzlab/kubeasz/blob/master/docs/setup/00-planning_and_overall_intro.md#ha-architecture)
- calico： 配置容器网络 (或者其他网络组件)

```bash
roles/kube-node/
├── defaults
│   └── main.yml		# 变量配置文件
├── tasks
│   ├── main.yml		# 主执行文件
│   ├── node_lb.yml		# haproxy 安装文件
│   └── offline.yml             # 离线安装 haproxy
└── templates
    ├── cni-default.conf.j2	# 默认cni插件配置模板
    ├── haproxy.cfg.j2		# haproxy 配置模板
    ├── haproxy.service.j2	# haproxy 服务模板
    ├── kubelet-config.yaml.j2  # kubelet 独立配置文件
    ├── kubelet-csr.json.j2	# 证书请求模板
    ├── kubelet.service.j2	# kubelet 服务模板
    └── kube-proxy.service.j2	# kube-proxy 服务模板
```

`roles/kube-node/tasks/main.yml`

```json
[root@k8s-1 ansible]# cat roles/kube-node/tasks/main.yml 
- name: 创建kube-node 相关目录
  file: name={{ item }} state=directory
  with_items:
  - /var/lib/kubelet
  - /var/lib/kube-proxy
  - /etc/cni/net.d

- name: 下载 kubelet,kube-proxy 二进制和基础 cni plugins
  copy: src={{ base_dir }}/bin/{{ item }} dest={{ bin_dir }}/{{ item }} mode=0755
  with_items:
  - kubectl
  - kubelet
  - kube-proxy
  - bridge
  - host-local
  - loopback
  tags: upgrade_k8s

# 每个 node 节点运行 haproxy 连接到多个 apiserver
- import_tasks: node_lb.yml
  when: "inventory_hostname not in groups['kube-master']"

- name: 替换 kubeconfig 的 apiserver 地址
  lineinfile:
    dest: /root/.kube/config
    regexp: "^    server"
    line: "    server: {{ KUBE_APISERVER }}"

##----------kubelet 配置部分--------------

- name: 准备kubelet 证书签名请求
  template: src=kubelet-csr.json.j2 dest={{ ca_dir }}/kubelet-csr.json

- name: 创建 kubelet 证书与私钥
  shell: "cd {{ ca_dir }} && {{ bin_dir }}/cfssl gencert \
        -ca={{ ca_dir }}/ca.pem \
        -ca-key={{ ca_dir }}/ca-key.pem \
        -config={{ ca_dir }}/ca-config.json \
        -profile=kubernetes kubelet-csr.json | {{ bin_dir }}/cfssljson -bare kubelet"

# 创建kubelet.kubeconfig 
- name: 设置集群参数
  shell: "{{ bin_dir }}/kubectl config set-cluster kubernetes \
        --certificate-authority={{ ca_dir }}/ca.pem \
        --embed-certs=true \
        --server={{ KUBE_APISERVER }} \
	--kubeconfig=/etc/kubernetes/kubelet.kubeconfig"

- name: 设置客户端认证参数
  shell: "{{ bin_dir }}/kubectl config set-credentials system:node:{{ inventory_hostname }} \
        --client-certificate={{ ca_dir }}/kubelet.pem \
        --embed-certs=true \
        --client-key={{ ca_dir }}/kubelet-key.pem \
	--kubeconfig=/etc/kubernetes/kubelet.kubeconfig"

- name: 设置上下文参数
  shell: "{{ bin_dir }}/kubectl config set-context default \
        --cluster=kubernetes \
	--user=system:node:{{ inventory_hostname }} \
	--kubeconfig=/etc/kubernetes/kubelet.kubeconfig"

- name: 选择默认上下文
  shell: "{{ bin_dir }}/kubectl config use-context default \
	--kubeconfig=/etc/kubernetes/kubelet.kubeconfig"

- name: 准备 cni配置文件
  template: src=cni-default.conf.j2 dest=/etc/cni/net.d/10-default.conf

# 设置 dns svc ip (这里选用 SERVICE_CIDR 中第2个IP)
- name: 注册变量 DNS_SVC_IP
  shell: echo {{ SERVICE_CIDR }}|cut -d/ -f1|awk -F. '{print $1"."$2"."$3"."$4+2}'
  register: DNS_SVC_IP
  tags: upgrade_k8s, restart_node

- name: 设置变量 CLUSTER_DNS_SVC_IP
  set_fact: CLUSTER_DNS_SVC_IP={{ DNS_SVC_IP.stdout }}
  tags: upgrade_k8s, restart_node

# 判断 kubernetes 版本
- name: 注册变量 TMP_VER
  shell: "{{ base_dir }}/bin/kube-apiserver --version|cut -d' ' -f2|cut -d'v' -f2"
  register: TMP_VER
  connection: local
  tags: upgrade_k8s, restart_node

- name: 获取 kubernetes 主版本号
  set_fact:
    KUBE_VER: "{{ TMP_VER.stdout.split('.')[0]|int + TMP_VER.stdout.split('.')[1]|int/100 }}"
  tags: upgrade_k8s, restart_node

- name: debug info
  debug: var="KUBE_VER"

- name: 创建kubelet的配置文件
  template: src=kubelet-config.yaml.j2 dest=/var/lib/kubelet/config.yaml
  tags: upgrade_k8s, restart_node

- name: 创建kubelet的systemd unit文件
  template: src=kubelet.service.j2 dest=/etc/systemd/system/kubelet.service
  tags: upgrade_k8s, restart_node

- name: 开机启用kubelet 服务
  shell: systemctl enable kubelet
  ignore_errors: true

- name: 开启kubelet 服务
  shell: systemctl daemon-reload && systemctl restart kubelet
  tags: upgrade_k8s, restart_node

##-------kube-proxy部分----------------

- name: 替换 kube-proxy.kubeconfig 的 apiserver 地址
  lineinfile:
    dest: /etc/kubernetes/kube-proxy.kubeconfig
    regexp: "^    server"
    line: "    server: {{ KUBE_APISERVER }}"

- name: 创建kube-proxy 服务文件
  template: src=kube-proxy.service.j2 dest=/etc/systemd/system/kube-proxy.service
  tags: reload-kube-proxy, restart_node, upgrade_k8s

- name: 开机启用kube-proxy 服务
  shell: systemctl enable kube-proxy
  ignore_errors: true

- name: 开启kube-proxy 服务
  shell: systemctl daemon-reload && systemctl restart kube-proxy
  tags: reload-kube-proxy, upgrade_k8s, restart_node

# 轮询等待kubelet启动完成
- name: 轮询等待kubelet启动
  shell: "systemctl status kubelet.service|grep Active"
  register: kubelet_status
  until: '"running" in kubelet_status.stdout'
  retries: 8
  delay: 2
  tags: reload-kube-proxy, upgrade_k8s, restart_node

- name: 轮询等待node达到Ready状态
  shell: "{{ bin_dir }}/kubectl get node {{ inventory_hostname }}|awk 'NR>1{print $2}'"
  register: node_status
  until: node_status.stdout == "Ready" or node_status.stdout == "Ready,SchedulingDisabled"
  retries: 8 
  delay: 8
  tags: upgrade_k8s, restart_node

- name: 设置node节点role
  shell: "{{ bin_dir }}/kubectl label node {{ inventory_hostname }} kubernetes.io/role=node --overwrite"
  ignore_errors: true
```



#### 变量配置文件

详见 roles/kube-node/defaults/main.yml，举例以下3个变量配置说明

```bash
cat roles/kube-node/defaults/main.yml 
# 默认使用kube-proxy的 'iptables' 模式，可选 'ipvs' 模式(experimental)
PROXY_MODE: "iptables"

# 基础容器镜像
SANDBOX_IMAGE: "mirrorgooglecontainers/pause-amd64:3.1"
#SANDBOX_IMAGE: "registry.access.redhat.com/rhel7/pod-infrastructure:latest"

# Kubelet 根目录
KUBELET_ROOT_DIR: "/var/lib/kubelet"

# node节点最大pod 数
MAX_PODS: 110

# 配置为kube组件（kubelet,kube-proxy,dockerd等）预留的资源量
KUBE_RESERVED_ENABLED: "yes"
KUBE_RESERVED: "{'cpu':'200m','memory':'500Mi','ephemeral-storage':'1Gi'}"
# k8s 官方不建议草率开启 system-reserved, 除非你基于长期监控，了解系统的资源占用状况；并且随着系统运行时间，需要适当增加资源预留
SYS_RESERVED_ENABLED: "no"
# 以下系统预留设置基于 4c/8g 虚机，最小化安装系统服务，如果使用高性能物理机请适当增加数值
SYS_RESERVED: "{'cpu':'200m','memory':'500Mi','ephemeral-storage':'1Gi'}"

# node 请求 apiserver 负载均衡算法，常见如下：
# "roundrobin": 基于服务器权重的轮询
# "leastconn": 基于服务器最小连接数
# "source": 基于请求源IP地址
# "uri": 基于请求的URI
BALANCE_ALG: "roundrobin"

# 设置 APISERVER 地址
KUBE_APISERVER: "{%- if inventory_hostname in groups['kube-master'] -%} \
                     https://{{ inventory_hostname }}:6443 \
                 {%- else -%} \
                     {%- if groups['kube-master']|length > 1 -%} \
                         https://127.0.0.1:6443 \
                     {%- else -%} \
                         https://{{ groups['kube-master'][0] }}:6443 \
                     {%- endif -%} \
                 {%- endif -%}"

# 增加/删除 master 节点时，node 节点需要重新配置 haproxy 等
MASTER_CHG: "no"

# 离线安装 haproxy (offline|online)
INSTALL_SOURCE: "online"
```

- 变量`KUBE_APISERVER`，根据不同的节点情况，它有三种取值方式
- 变量`MASTER_CHG`，变更 master 节点时会根据它来重新配置 haproxy



#### 创建cni 基础网络插件配置文件

因为后续需要用 `DaemonSet Pod`方式运行k8s网络插件，所以kubelet.server服务必须开启cni相关参数，并且提供cni网络配置文件



#### 创建 kubelet 的服务文件

- 根据官方建议独立使用 kubelet 配置文件，详见roles/kube-node/templates/kubelet-config.yaml.j2
- 必须先创建工作目录 `/var/lib/kubelet`

```bash
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
WorkingDirectory=/var/lib/kubelet
{% if KUBE_RESERVED_ENABLED == "yes" or SYS_RESERVED_ENABLED == "yes" %}
ExecStartPre=/bin/mount -o remount,rw '/sys/fs/cgroup'
ExecStartPre=/bin/mkdir -p /sys/fs/cgroup/cpuset/system.slice/kubelet.service
ExecStartPre=/bin/mkdir -p /sys/fs/cgroup/hugetlb/system.slice/kubelet.service
ExecStartPre=/bin/mkdir -p /sys/fs/cgroup/memory/system.slice/kubelet.service
ExecStartPre=/bin/mkdir -p /sys/fs/cgroup/pids/system.slice/kubelet.service
{% endif %}
ExecStart={{ bin_dir }}/kubelet \
  --config=/var/lib/kubelet/config.yaml \
{% if KUBE_VER|float < 1.13 %}
  --allow-privileged=true \
{% endif %}
  --cni-bin-dir={{ bin_dir }} \
  --cni-conf-dir=/etc/cni/net.d \
{% if CONTAINER_RUNTIME == "containerd" %}
  --container-runtime=remote \
  --container-runtime-endpoint=unix:///run/containerd/containerd.sock \
{% endif %}
  --hostname-override={{ inventory_hostname }} \
  --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \
  --network-plugin=cni \
  --pod-infra-container-image={{ SANDBOX_IMAGE }} \
  --root-dir={{ KUBELET_ROOT_DIR }} \
  --v=2
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

- --pod-infra-container-image 指定`基础容器`（负责创建Pod 内部共享的网络、文件系统等）镜像，**K8S每一个运行的 POD里面必然包含这个基础容器**，如果它没有运行起来那么你的POD 肯定创建不了，kubelet日志里面会看到类似 `FailedCreatePodSandBox` 错误，可用`docker images` 查看节点是否已经下载到该镜像
- --cluster-dns 指定 kubedns 的 Service IP(可以先分配，后续创建 kubedns 服务时指定该 IP)，--cluster-domain 指定域名后缀，这两个参数同时指定后才会生效；
- --network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir={{ bin_dir }} 为使用cni 网络，并调用calico管理网络所需的配置
- --fail-swap-on=false K8S 1.8+需显示禁用这个，否则服务不能启动
- --client-ca-file={{ ca_dir }}/ca.pem 和 --anonymous-auth=false 关闭kubelet的匿名访问，详见[匿名访问漏洞说明](https://github.com/easzlab/kubeasz/blob/master/docs/setup/mixes/01.fix_kubelet_annoymous_access.md)
- --ExecStartPre=/bin/mkdir -p xxx 对于某些系统（centos7）cpuset和hugetlb 是默认没有初始化system.slice 的，需要手动创建，否则在启用--kube-reserved-cgroup 时会报错Failed to start ContainerManager Failed to enforce System Reserved Cgroup Limits
- 关于kubelet资源预留相关配置请参考 <https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/>



#### 创建 kube-proxy kubeconfig 文件

该步骤已经在 deploy节点完成，[roles/deploy/tasks/main.yml](https://github.com/easzlab/kubeasz/blob/master/roles/deploy/tasks/main.yml)

- 生成的kube-proxy.kubeconfig 配置文件需要移动到/etc/kubernetes/目录，后续kube-proxy服务启动参数里面需要指定

#### 创建 kube-proxy服务文件

```bash
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
WorkingDirectory=/var/lib/kube-proxy
ExecStart={{ bin_dir }}/kube-proxy \
  --bind-address={{ inventory_hostname }} \
  --cluster-cidr={{ CLUSTER_CIDR }} \
  --hostname-override={{ inventory_hostname }} \
  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig \
  --logtostderr=true \
  --v=2
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

- --hostname-override 参数值必须与 kubelet 的值一致，否则 kube-proxy 启动后会找不到该 Node，从而不会创建任何 iptables 规则
- 特别注意：kube-proxy 根据 --cluster-cidr 判断集群内部和外部流量，指定 --cluster-cidr 或 --masquerade-all 选项后 kube-proxy 才会对访问 Service IP 的请求做 SNAT；



#### 验证 node 状态

```bash
$ systemctl status kubelet	# 查看状态
$ systemctl status kube-proxy
$ journalctl -u kubelet		# 查看日志
$ journalctl -u kube-proxy 
```

运行 `kubectl get node` 可以看到类似

```bash
NAME           STATUS    ROLES     AGE       VERSION
192.168.1.42   Ready     <none>    2d        v1.9.0
192.168.1.43   Ready     <none>    2d        v1.9.0
192.168.1.44   Ready     <none>    2d        v1.9.0
```





## 06-安装网络组件



首先回顾下K8S网络设计原则，在配置集群网络插件或者实践K8S 应用/服务部署请时刻想到这些原则：

- 1.每个Pod都拥有一个独立IP地址，Pod内所有容器共享一个网络命名空间
- 2.集群内所有Pod都在一个直接连通的扁平网络中，可通过IP直接访问
  - 所有容器之间无需NAT就可以直接互相访问
  - 所有Node和所有容器之间无需NAT就可以直接互相访问
  - 容器自己看到的IP跟其他容器看到的一样
- 3.Service cluster IP尽可在集群内部访问，外部请求需要通过NodePort、LoadBalance或者Ingress来访问

`Container Network Interface (CNI)`是目前CNCF主推的网络模型，它由两部分组成：

- CNI Plugin负责给容器配置网络，它包括两个基本的接口
  - 配置网络: AddNetwork(net *NetworkConfig, rt *RuntimeConf) (types.Result, error)
  - 清理网络: DelNetwork(net *NetworkConfig, rt *RuntimeConf) error
- IPAM Plugin负责给容器分配IP地址

Kubernetes Pod的网络是这样创建的：

- 0.每个Pod除了创建时指定的容器外，都有一个kubelet启动时指定的`基础容器`，比如：`easzlab/pause-amd64` `registry.access.redhat.com/rhel7/pod-infrastructure`
- 1.首先 kubelet创建`基础容器`生成network namespace
- 2.然后 kubelet调用网络CNI driver，由它根据配置调用具体的CNI 插件
- 3.然后 CNI 插件给`基础容器`配置网络
- 4.最后 Pod 中其他的容器共享使用`基础容器`的网络

本项目基于CNI driver 调用各种网络插件来配置kubernetes的网络，常用CNI插件有 `flannel` `calico` `weave`等等，这些插件各有优势，也在互相借鉴学习优点，比如：在所有node节点都在一个二层网络时候，flannel提供hostgw实现，避免vxlan实现的udp封装开销，估计是目前最高效的；calico也针对L3 Fabric，推出了IPinIP的选项，利用了GRE隧道封装；因此这些插件都能适合很多实际应用场景。

项目当前内置支持的网络插件有：`calico` `cilium` `flannel` `kube-ovn` `kube-router`

**安装讲解**

- [安装calico](https://github.com/easzlab/kubeasz/blob/master/docs/setup/network-plugin/calico.md)
- [安装cilium](https://github.com/easzlab/kubeasz/blob/master/docs/setup/network-plugin/cilium.md)
- [安装flannel](https://github.com/easzlab/kubeasz/blob/master/docs/setup/network-plugin/flannel.md)
- [安装kube-ovn](https://github.com/easzlab/kubeasz/blob/master/docs/setup/network-plugin/kube-ovn.md)
- [安装kube-router](https://github.com/easzlab/kubeasz/blob/master/docs/setup/network-plugin/kube-router.md)

**参考**

- [kubernetes.io networking docs](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
- [feiskyer-kubernetes指南网络章节](https://github.com/feiskyer/kubernetes-handbook/blob/master/zh/network/network.md)



### 安装calico网络组件

推荐阅读[calico kubernetes guide](https://docs.projectcalico.org/v3.4/getting-started/kubernetes/)

本项目提供多种网络插件可选，如果需要安装calico，请在/etc/ansible/hosts文件中设置变量 `CLUSTER_NETWORK="calico"`，更多的calico设置在`roles/calico/defaults/main.yml`文件定义。

- calico-node需要在所有master节点和node节点安装

```bash
roles/calico/
├── defaults
│   └── main.yml
├── tasks
│   └── main.yml
└── templates
    ├── calico-csr.json.j2
    ├── calicoctl.cfg.j2
    └── calico-vx.y.yaml.j2
```

`roles/calico/tasks/main.yml`

```bash
cat roles/calico/tasks/main.yml 
- name: 在节点创建相关目录
  file: name={{ item }} state=directory
  with_items:
  - /etc/calico/ssl
  - /etc/cni/net.d
  - /opt/kube/images 
  - /opt/kube/kube-system

- name: 创建calico 证书请求
  template: src=calico-csr.json.j2 dest=/etc/calico/ssl/calico-csr.json

- name: 创建 calico证书和私钥
  shell: "cd /etc/calico/ssl && {{ bin_dir }}/cfssl gencert \
        -ca={{ ca_dir }}/ca.pem \
        -ca-key={{ ca_dir }}/ca-key.pem \
        -config={{ ca_dir }}/ca-config.json \
        -profile=kubernetes calico-csr.json | {{ bin_dir }}/cfssljson -bare calico"

- name: get calico-etcd-secrets info
  shell: "{{ bin_dir }}/kubectl get secrets -n kube-system"
  register: secrets_info
  run_once: true

- name: 创建 calico-etcd-secrets
  shell: "cd /etc/calico/ssl && \
        {{ bin_dir }}/kubectl create secret generic -n kube-system calico-etcd-secrets \
        --from-file=etcd-ca={{ ca_dir }}/ca.pem \
        --from-file=etcd-key=calico-key.pem \
        --from-file=etcd-cert=calico.pem"
  when: '"calico-etcd-secrets" not in secrets_info.stdout'
  run_once: true

- name: 配置 calico DaemonSet yaml文件
  template: src=calico-{{ calico_ver_main }}.yaml.j2 dest=/opt/kube/kube-system/calico.yaml
    
# 【可选】推送离线docker 镜像，可以忽略执行错误
- block:
    - name: 检查是否已下载离线calico镜像
      command: "ls {{ base_dir }}/down"
      register: download_info
      connection: local
      run_once: true
    
    - name: 尝试推送离线docker 镜像（若执行失败，可忽略）
      copy: src={{ base_dir }}/down/{{ item }} dest=/opt/kube/images/{{ item }}
      when: 'item in download_info.stdout'
      with_items:
      - "pause_3.1.tar"
      - "{{ calico_offline }}"
      ignore_errors: true
    
    - name: 获取calico离线镜像推送情况
      command: "ls /opt/kube/images"
      register: image_info
    
    # 如果目录下有离线镜像，就把它导入到node节点上
    - name: 导入 calico的离线镜像（若执行失败，可忽略）
      shell: "{{ bin_dir }}/docker load -i /opt/kube/images/{{ item }}"
      with_items:
      - "pause_3.1.tar"
      - "{{ calico_offline }}"
      ignore_errors: true
      when: "item in image_info.stdout and CONTAINER_RUNTIME == 'docker'" 

    - name: 导入 calico的离线镜像（若执行失败，可忽略）
      shell: "{{ bin_dir }}/ctr -n=k8s.io images import /opt/kube/images/{{ item }}"
      with_items:
      - "pause_3.1.tar"
      - "{{ calico_offline }}"
      ignore_errors: true
      when: "item in image_info.stdout and CONTAINER_RUNTIME == 'containerd'"

# 只需单节点执行一次
- name: 运行 calico网络
  shell: "{{ bin_dir }}/kubectl apply -f /opt/kube/kube-system/calico.yaml"
  run_once: true

# 删除原有cni配置
- name: 删除默认cni配置
  file: path=/etc/cni/net.d/10-default.conf state=absent

# [可选]cni calico plugins 已经在calico.yaml完成自动安装
- name: 下载calicoctl 客户端
  copy: src={{ base_dir }}/bin/{{ item }} dest={{ bin_dir }}/{{ item }} mode=0755
  with_items:
  #- calico
  #- calico-ipam
  #- loopback
  - calicoctl
  ignore_errors: true

- name: 准备 calicoctl配置文件
  template: src=calicoctl.cfg.j2 dest=/etc/calico/calicoctl.cfg

# 等待网络插件部署成功，视下载镜像速度而定
- name: 轮询等待calico-node 运行，视下载镜像速度而定
  shell: "{{ bin_dir }}/kubectl get pod -n kube-system -o wide|grep 'calico-node'|grep ' {{ inventory_hostname }} '|awk '{print $3}'"
  register: pod_status
  until: pod_status.stdout == "Running"
  retries: 15
  delay: 15
  ignore_errors: true
```



##### 创建calico 证书申请

```json
{
  "CN": "calico",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "HangZhou",
      "L": "XS",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```

- calico 使用客户端证书，所以hosts字段可以为空；后续可以看到calico证书用在四个地方：
  - calico/node 这个docker 容器运行时访问 etcd 使用证书
  - cni 配置文件中，cni 插件需要访问 etcd 使用证书
  - calicoctl 操作集群网络时访问 etcd 使用证书
  - calico/kube-controllers 同步集群网络策略时访问 etcd 使用证书



##### 创建 calico DaemonSet yaml文件和rbac 文件

请对照 roles/calico/templates/calico.yaml.j2文件注释和以下注意内容

- 详细配置参数请参考[calico官方文档](https://docs.projectcalico.org/v2.6/reference/node/configuration)
- 配置ETCD_ENDPOINTS 、CA、证书等，所有{{ }}变量与ansible hosts文件中设置对应
- 配置集群POD网络 CALICO_IPV4POOL_CIDR={{ CLUSTER_CIDR }}
- **重要**本K8S集群运行在同网段kvm虚机上，虚机间没有网络ACL限制，因此可以设置`CALICO_IPV4POOL_IPIP=off`，如果你的主机位于不同网段，或者运行在公有云上需要打开这个选项 `CALICO_IPV4POOL_IPIP=always`
- 配置FELIX_DEFAULTENDPOINTTOHOSTACTION=ACCEPT 默认允许Pod到Node的网络流量，更多[felix配置选项](https://docs.projectcalico.org/v2.6/reference/felix/configuration)

##### 安装calico 网络

- 安装前检查主机名不能有大写字母，只能由`小写字母` `-` `.` 组成 (name must consist of lower case alphanumeric characters, '-' or '.' (regex: [a-z0-9](https://github.com/easzlab/kubeasz/blob/master/docs/setup/network-plugin/%5B-a-z0-9%5D*%5Ba-z0-9%5D)?(.[a-z0-9](https://github.com/easzlab/kubeasz/blob/master/docs/setup/network-plugin/%5B-a-z0-9%5D*%5Ba-z0-9%5D)?)*))(calico-node v3.0.6以上已经解决主机大写字母问题)
- **安装前必须确保各节点主机名不重复** ，calico node name 由节点主机名决定，如果重复，那么重复节点在etcd中只存储一份配置，BGP 邻居也不会建立。
- 安装之前必须确保`kube-master`和`kube-node`节点已经成功部署
- 只需要在任意装有kubectl客户端的节点运行 `kubectl apply -f`安装即可
- 等待15s后(视网络拉取calico相关镜像速度)，calico 网络插件安装完成，删除之前kube-node安装时默认cni网络配置



##### [可选]配置calicoctl工具 [calicoctl.cfg.j2](https://github.com/easzlab/kubeasz/blob/master/docs/setup/network-plugin/roles/calico/templates/calicoctl.cfg.j2)

```yaml
apiVersion: v1
kind: calicoApiConfig
metadata:
spec:
  datastoreType: "etcdv2"
  etcdEndpoints: {{ ETCD_ENDPOINTS }}
  etcdKeyFile: /etc/calico/ssl/calico-key.pem
  etcdCertFile: /etc/calico/ssl/calico.pem
  etcdCACertFile: /etc/calico/ssl/ca.pem
```



##### 验证calico网络

执行calico安装成功后可以验证如下：(需要等待镜像下载完成，有时候即便上一步已经配置了docker国内加速，还是可能比较慢，请确认以下容器运行起来以后，再执行后续验证步骤)

```bash
kubectl get pod --all-namespaces
NAMESPACE     NAME                                       READY     STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-5c6b98d9df-xj2n4   1/1       Running   0          1m
kube-system   calico-node-4hr52                          2/2       Running   0          1m
kube-system   calico-node-8ctc2                          2/2       Running   0          1m
kube-system   calico-node-9t8md                          2/2       Running   0          1m
```

**查看网卡和路由信息**

先在集群创建几个测试pod: ` kubectl run test --image=busybox --replicas=3 sleep 30000`

```bash
# 查看网卡信息
ip a
```

- 可以看到包含类似cali1cxxx的网卡，是calico为测试pod生成的
- tunl0网卡现在不用管，是默认生成的，当开启IPIP 特性时使用的隧道

```bash
# 查看路由
route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.1.1     0.0.0.0         UG    0      0        0 ens3
192.168.1.0     0.0.0.0         255.255.255.0   U     0      0        0 ens3
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
172.20.3.64     192.168.1.34    255.255.255.192 UG    0      0        0 ens3
172.20.33.128   0.0.0.0         255.255.255.192 U     0      0        0 *
172.20.33.129   0.0.0.0         255.255.255.255 UH    0      0        0 caliccc295a6d4f
172.20.104.0    192.168.1.35    255.255.255.192 UG    0      0        0 ens3
172.20.166.128  192.168.1.63    255.255.255.192 UG    0      0        0 ens3
```

**查看所有calico节点状态**

```bash
calicoctl node status
Calico process is running.

IPv4 BGP status
+--------------+-------------------+-------+----------+-------------+
| PEER ADDRESS |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+--------------+-------------------+-------+----------+-------------+
| 192.168.1.34 | node-to-node mesh | up    | 12:34:00 | Established |
| 192.168.1.35 | node-to-node mesh | up    | 12:34:00 | Established |
| 192.168.1.63 | node-to-node mesh | up    | 12:34:01 | Established |
+--------------+-------------------+-------+----------+-------------+
```

**BGP 协议是通过TCP 连接来建立邻居的，因此可以用netstat 命令验证 BGP Peer**

```bash
netstat -antlp|grep ESTABLISHED|grep 179
tcp        0      0 192.168.1.66:179        192.168.1.35:41316      ESTABLISHED 28479/bird      
tcp        0      0 192.168.1.66:179        192.168.1.34:40243      ESTABLISHED 28479/bird      
tcp        0      0 192.168.1.66:179        192.168.1.63:48979      ESTABLISHED 28479/bird
```

**查看etcd中calico相关信息**

因为这里calico网络使用etcd存储数据，所以可以在etcd集群中查看数据

- calico 3.x 版本默认使用 etcd v3存储，**登录集群的一个etcd 节点**，查看命令：

```
# 查看所有calico相关数据
ETCDCTL_API=3 etcdctl --endpoints="http://127.0.0.1:2379" get --prefix /calico
# 查看 calico网络为各节点分配的网段
ETCDCTL_API=3 etcdctl --endpoints="http://127.0.0.1:2379" get --prefix /calico/ipam/v2/host
```

- calico 2.x 版本默认使用 etcd v2存储，**登录集群的一个etcd 节点**，查看命令：

```bash
# 查看所有calico相关数据
etcdctl --endpoints=http://127.0.0.1:2379 --ca-file=/etc/kubernetes/ssl/ca.pem ls /calico
```



##### calico 配置 BGP Route Reflectors

`Calico`作为`k8s`的一个流行网络插件，它依赖`BGP`路由协议实现集群节点上的`POD`路由互通；而路由互通的前提是节点间建立 BGP Peer 连接。BGP 路由反射器（Route Reflectors，简称 RR）可以简化集群BGP Peer的连接方式，它是解决BGP扩展性问题的有效方式；具体来说：

- 没有 RR 时，所有节点之间需要两两建立连接（IBGP全互联），节点数量增加将导致连接数剧增、资源占用剧增
- 引入 RR 后，其他 BGP 路由器只需要与它建立连接并交换路由信息，节点数量增加连接数只是线性增加，节省系统资源

calico-node 版本 v3.3 开始支持内建路由反射器，非常方便，因此使用 calico 作为网络插件可以支持大规模节点数的`K8S`集群。

本文档主要讲解配置 BGP Route Reflectors，建议首先阅读[基础calico文档](https://github.com/easzlab/kubeasz/blob/master/docs/setup/network-plugin/calico.md)。

##### 前提条件

实验环境为按照kubeasz安装的2主2从集群，calico 版本 v3.3.2

```bash
$ kubectl get node
NAME           STATUS                     ROLES    AGE    VERSION
192.168.1.1   Ready,SchedulingDisabled   master   178m   v1.13.1
192.168.1.2   Ready,SchedulingDisabled   master   178m   v1.13.1
192.168.1.3   Ready                      node     178m   v1.13.1
192.168.1.4   Ready                      node     178m   v1.13.1

$ kubectl get pod -n kube-system -o wide | grep calico
calico-kube-controllers-77487546bd-jqrlc   1/1     Running   0          179m   192.168.1.3   192.168.1.3   <none>           <none>
calico-node-67t5m                          2/2     Running   0          179m   192.168.1.1   192.168.1.1   <none>           <none>
calico-node-drmhq                          2/2     Running   0          179m   192.168.1.2   192.168.1.2   <none>           <none>
calico-node-rjtkv                          2/2     Running   0          179m   192.168.1.4   192.168.1.4   <none>           <none>
calico-node-xtspl                          2/2     Running   0          179m   192.168.1.3   192.168.1.3 
```

查看当前集群中BGP连接情况：可以看到集群中4个节点两两建立了 BGP 连接

```bash
$ ansible all -m shell -a '/opt/kube/bin/calicoctl node status'
192.168.1.3 | SUCCESS | rc=0 >>
Calico process is running.

IPv4 BGP status
+--------------+-------------------+-------+----------+-------------+
| PEER ADDRESS |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+--------------+-------------------+-------+----------+-------------+
| 192.168.1.1 | node-to-node mesh | up    | 03:08:20 | Established |
| 192.168.1.2 | node-to-node mesh | up    | 03:08:18 | Established |
| 192.168.1.4 | node-to-node mesh | up    | 03:08:19 | Established |
+--------------+-------------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.

192.168.1.2 | SUCCESS | rc=0 >>
Calico process is running.

IPv4 BGP status
+--------------+-------------------+-------+----------+-------------+
| PEER ADDRESS |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+--------------+-------------------+-------+----------+-------------+
| 192.168.1.4 | node-to-node mesh | up    | 03:08:17 | Established |
| 192.168.1.3 | node-to-node mesh | up    | 03:08:18 | Established |
| 192.168.1.1 | node-to-node mesh | up    | 03:08:20 | Established |
+--------------+-------------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.

192.168.1.1 | SUCCESS | rc=0 >>
Calico process is running.

IPv4 BGP status
+--------------+-------------------+-------+----------+-------------+
| PEER ADDRESS |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+--------------+-------------------+-------+----------+-------------+
| 192.168.1.2 | node-to-node mesh | up    | 03:08:21 | Established |
| 192.168.1.3 | node-to-node mesh | up    | 03:08:21 | Established |
| 192.168.1.4 | node-to-node mesh | up    | 03:08:21 | Established |
+--------------+-------------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.

192.168.1.4 | SUCCESS | rc=0 >>
Calico process is running.

IPv4 BGP status
+--------------+-------------------+-------+----------+-------------+
| PEER ADDRESS |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+--------------+-------------------+-------+----------+-------------+
| 192.168.1.2 | node-to-node mesh | up    | 03:08:17 | Established |
| 192.168.1.3 | node-to-node mesh | up    | 03:08:19 | Established |
| 192.168.1.1 | node-to-node mesh | up    | 03:08:20 | Established |
+--------------+-------------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.
```

##### 配置全局禁用全连接（BGP full mesh）

```bash
$ cat << EOF | calicoctl create -f -
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  logSeverityScreen: Info
  nodeToNodeMeshEnabled: false
  asNumber: 64512
EOF
```

上述命令配置完成后，再次使用命令`ansible all -m shell -a '/opt/kube/bin/calicoctl node status'`查看，可以看到之前所有的bgp连接都消失了。



#### 配置 BGP node 与 Route Reflector 的连接建立规则

```bash
$ cat << EOF | calicoctl create -f -
kind: BGPPeer
apiVersion: projectcalico.org/v3
metadata:
  name: peer-to-rrs
spec:
  # 规则1：普通 bgp node 与 rr 建立连接
  nodeSelector: "!has(i-am-a-route-reflector)"
  peerSelector: has(i-am-a-route-reflector)

---
kind: BGPPeer
apiVersion: projectcalico.org/v3
metadata:
  name: rr-mesh
spec:
  # 规则2：route reflectors 之间也建立连接
  nodeSelector: has(i-am-a-route-reflector)
  peerSelector: has(i-am-a-route-reflector)
EOF
```

上述命令配置完成后，使用命令：`calicoctl get bgppeer` `calicoctl get bgppeer rr-mesh -o yaml` 检查配置是否正确。

##### 选择并配置 Route Reflector 节点

首先查看当前集群中的节点：

```bash
$ calicoctl get node -o wide
NAME     ASN       IPV4              IPV6   
k8s401   (64512)   192.168.1.1/24          
k8s402   (64512)   192.168.1.2/24          
k8s403   (64512)   192.168.1.3/24          
k8s404   (64512)   192.168.1.4/24
```

可以在集群中选择1个或多个节点作为 rr 节点，这里先选择节点：k8s401

```bash
# 1.先导出 node k8s401 的配置，准备修改
$ calicoctl get node k8s-4 --export -o yaml |tee rr01.yml
apiVersion: projectcalico.org/v3
kind: Node
metadata:
  creationTimestamp: null
  name: k8s401
spec:
  bgp:
    ipv4Address: 192.168.1.1/24
    ipv4IPIPTunnelAddr: 172.20.7.128
  orchRefs:
  - nodeName: 192.168.1.1
    orchestrator: k8s

# 2.修改上述 rr01.yml 的配置如下
apiVersion: projectcalico.org/v3
kind: Node
metadata:
  creationTimestamp: null
  name: k8s-04
  labels:
    # 设置标签
    i-am-a-route-reflector: true
spec:
  bgp:
    ipv4Address: 192.168.1.1/24
    ipv4IPIPTunnelAddr: 172.20.7.128
    # 设置集群ID
    routeReflectorClusterID: 224.0.0.1
  orchRefs:
  - nodeName: 192.168.1.1
    orchestrator: k8s

# 3.应用修改后的 rr node 配置
$ calicoctl apply -f rr01.yml
```

##### 查看增加 rr 之后的bgp 连接情况

```bash
$ ansible all -m shell -a '/opt/kube/bin/calicoctl node status'
192.168.1.4 | SUCCESS | rc=0 >>
Calico process is running.

IPv4 BGP status
+--------------+-----------+-------+----------+-------------+
| PEER ADDRESS |   PEER TYPE   | STATE |  SINCE   |    INFO     |
+--------------+-----------+-------+----------+-------------+
| 192.168.1.1 | node specific | up    | 11:02:55 | Established |
+--------------+-----------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.

192.168.1.3 | SUCCESS | rc=0 >>
Calico process is running.

IPv4 BGP status
+--------------+-----------+-------+----------+-------------+
| PEER ADDRESS | PEER TYPE | STATE |  SINCE   |    INFO     |
+--------------+-----------+-------+----------+-------------+
| 192.168.1.1 | node specific | up    | 11:02:55 | Established |
+--------------+-----------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.

192.168.1.1 | SUCCESS | rc=0 >>
Calico process is running.

IPv4 BGP status
+--------------+---------------+-------+----------+-------------+
| PEER ADDRESS |   PEER TYPE   | STATE |  SINCE   |    INFO     |
+--------------+---------------+-------+----------+-------------+
| 192.168.1.2 | node specific | up    | 11:02:55 | Established |
| 192.168.1.3 | node specific | up    | 11:02:55 | Established |
| 192.168.1.4 | node specific | up    | 11:02:55 | Established |
+--------------+---------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.

192.168.1.2 | SUCCESS | rc=0 >>
Calico process is running.

IPv4 BGP status
+--------------+-----------+-------+----------+-------------+
| PEER ADDRESS | PEER TYPE | STATE |  SINCE   |    INFO     |
+--------------+-----------+-------+----------+-------------+
| 192.168.1.1 | node specific | up    | 11:02:55 | Established |
+--------------+-----------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.
```

可以看到所有其他节点都与所选rr节点建立bgp连接。



##### 再增加一个 rr 节点

步骤同上述选择第1个 rr 节点，这里省略；添加成功后可以看到所有其他节点都与两个rr节点建立bgp连接，两个rr节点之间也建立bgp连接。

- 对于节点数较多的`K8S`集群建议配置3-4个 RR 节点



##### 参考文档

- 1.[Calico 使用指南：Route Reflectors](https://docs.projectcalico.org/v3.3/usage/routereflector)
- 2.[BGP路由反射器基础](https://www.sohu.com/a/140033025_761420)

更多 BGP 路由协议相关知识请查阅思科/华为相关网络文档。





### 安装cilium网络组件

`cilium` 是一个革新的网络与安全组件；基于 linux 内核新技术--`BPF`，它可以透明、零侵入地实现服务间安全策略与可视化，主要优势如下：

- 支持L3/L4, L7(如：HTTP/gRPC/Kafka)的安全策略
- 支持基于安全ID而不是地址+端口的传统防火墙策略
- 支持基于Overlay或Native Routing的扁平多节点pod网络
  - Overlay VXLAN 方式类似于 flannel 的VXLAN后端
- 高性能负载均衡，支持DSR
- 支持事件、策略跟踪和监控集成

##### 开始使用 cilium

以下为简要翻译 `cilium doc`上的一个应用示例[原文](http://docs.cilium.io/en/stable/gettingstarted/minikube/#step-2-deploy-the-demo-application)，部署在单节点k8s 环境的实践。

##### 0.升级内核并重启

- Linux kernel >= 4.9.17，请阅读文档[升级内核](https://github.com/easzlab/kubeasz/blob/master/docs/setup/network-plugin/guide/kernel_upgrade.md)
- etcd >= 3.1.0 or consul >= 0.6.4

##### 1.选择cilium网络后安装k8s(allinone)

- 参考[快速指南](https://github.com/easzlab/kubeasz/blob/master/docs/setup/network-plugin/quickStart.md)，设置 ansible hosts 文件中变量 `CLUSTER_NETWORK="cilium"`

##### 2.部署示例应用

官方文档用几个`pod/svc` 抽象一个有趣的应用场景（星战迷）：星战中帝国方建造了被称为“终极武器”的“死星”，它是一个卫星大小的战斗空间站，它的核心是使用凯伯晶体（Kyber Crystal）的超级激光炮，剧中它的首秀就以完全火力摧毁了“杰达圣城”（Jedha）。下面将用运行于 k8s上的 pod/svc/cilium 等模拟“死星“的一个“飞船登陆”系统安全策略设计。

- deploy/deathstar：作为控制整个“死星”的飞船登陆管理系统，它暴露一个SVC，提供HTTP REST 接口给飞船请求登陆使用；
- pod/tiefighter：作为“帝国”方的常规战斗飞船，它会调用上述 HTTP 接口，请求登陆“死星”；
- pod/xwing：作为“盟军”方的飞行舰，它也尝试调用 HTTP 接口，请求登陆“死星”；

![1591875408919](D:\学习资料\笔记\linux\assets\1591875408919.png)

根据文件[http-sw-app.yaml](https://github.com/easzlab/kubeasz/blob/master/roles/cilium/files/star_war_example/http-sw-app.yaml) 创建 `$ kubectl create -f http-sw-app.yaml` 后，验证如下：

```bash
$ kubectl get pods,svc
NAME                             READY     STATUS    RESTARTS   AGE
pod/deathstar-5fc7c7795d-djf2q   1/1       Running   0          4h
pod/deathstar-5fc7c7795d-hrgst   1/1       Running   0          4h
pod/tiefighter                   1/1       Running   0          4h
pod/xwing                        1/1       Running   0          4h

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/deathstar    ClusterIP   10.68.242.130   <none>        80/TCP    4h
service/kubernetes   ClusterIP   10.68.0.1       <none>        443/TCP   5h
```

每个 POD 在 `cilium` 中都表示为 `Endpoint`，初始每个 `Endpoint` 的”进出安全策略“状态均为 `Disabled`，如下：(已省略部分无关 POD 信息)

```bash
$ kubectl exec -n kube-system cilium-6t5vx -- cilium endpoint list
ENDPOINT   POLICY (ingress)   POLICY (egress)   IDENTITY   LABELS (source:key[=value])                                    IPv6                  IPv4           STATUS   
           ENFORCEMENT        ENFORCEMENT                                                                                                                      
643        Disabled           Disabled          31371      k8s:class=deathstar                                            f00d::ac14:0:0:283    172.20.0.246   ready   
                                                           k8s:io.cilium.k8s.policy.serviceaccount=default                                                             
                                                           k8s:io.kubernetes.pod.namespace=default                                                                     
                                                           k8s:org=empire                                                                                              
1011       Disabled           Disabled          31371      k8s:class=deathstar                                            f00d::ac14:0:0:3f3    172.20.0.63    ready   
                                                           k8s:io.cilium.k8s.policy.serviceaccount=default                                                             
                                                           k8s:io.kubernetes.pod.namespace=default                                                                     
                                                           k8s:org=empire                                                                                              
32030      Disabled           Disabled          5350       k8s:class=tiefighter                                           f00d::ac14:0:0:7d1e   172.20.0.201   ready   
                                                           k8s:io.cilium.k8s.policy.serviceaccount=default                                                             
                                                           k8s:io.kubernetes.pod.namespace=default                                                                     
                                                           k8s:org=empire                                                                                              
45943      Disabled           Disabled          14309      k8s:class=xwing                                                f00d::ac14:0:0:b377   172.20.0.189   ready   
                                                           k8s:io.cilium.k8s.policy.serviceaccount=default                                                             
                                                           k8s:io.kubernetes.pod.namespace=default                                                                     
                                                           k8s:org=alliance                                                                                            
52035      Disabled           Disabled          4          reserved:health                         
```



##### 3.检查初始状态

当然“死星”应该只允许“帝国”的飞船着陆，因为没有应用任何策略，所以初始状态下“帝国”和“联盟”的飞船都可以登陆，如下测试：

```bash
$ kubectl exec xwing -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
Ship landed # 成功着陆

$ kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
Ship landed # 成功着陆
```



##### 4.应用 L3/L4 策略

现在我们应用策略，仅让带有标签 `org=empire`的飞船登陆“死星”；那么带有标签 `org=alliance`的“联盟”飞船将禁止登陆；这个就是我们熟悉的传统L3/L4 防火墙策略，并跟踪连接（会话）状态；

![1591875360464](D:\学习资料\笔记\linux\assets\1591875360464.png)

根据文件[sw_l3_l4_policy.yaml](https://github.com/easzlab/kubeasz/blob/master/roles/cilium/files/star_war_example/sw_l3_l4_policy.yaml) 创建 `$ kubectl apply -f sw_l3_l4_policy.yaml` 后，验证如下：

```bash
$ kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
Ship landed # 成功着陆

$ kubectl exec xwing -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
# 失败超时
```



##### 5.查看安全策略

再次执行 `cilium endpoint list`，可以看到标签带`deathstar`的 POD 已经应用了 `Ingress`方向的策略：

```bash
# kubectl exec -n kube-system cilium-6t5vx -- cilium endpoint list
ENDPOINT   POLICY (ingress)   POLICY (egress)   IDENTITY   LABELS (source:key[=value])                                    IPv6                  IPv4           STATUS   
           ENFORCEMENT        ENFORCEMENT                                                                                                                      
643        Enabled            Disabled          31371      k8s:class=deathstar                                            f00d::ac14:0:0:283    172.20.0.246   ready   
                                                           k8s:io.cilium.k8s.policy.serviceaccount=default                                                             
                                                           k8s:io.kubernetes.pod.namespace=default                                                                     
                                                           k8s:org=empire                                                                                              
1011       Enabled            Disabled          31371      k8s:class=deathstar                                            f00d::ac14:0:0:3f3    172.20.0.63    ready   
                                                           k8s:io.cilium.k8s.policy.serviceaccount=default                                                             
                                                           k8s:io.kubernetes.pod.namespace=default                                                                     
                                                           k8s:org=empire                                                                                              
32030      Disabled           Disabled          5350       k8s:class=tiefighter                                           f00d::ac14:0:0:7d1e   172.20.0.201   ready   
                                                           k8s:io.cilium.k8s.policy.serviceaccount=default                                                             
                                                           k8s:io.kubernetes.pod.namespace=default                                                                     
                                                           k8s:org=empire                                                                                              
45943      Disabled           Disabled          14309      k8s:class=xwing                                                f00d::ac14:0:0:b377   172.20.0.189   ready   
                                                           k8s:io.cilium.k8s.policy.serviceaccount=default                                                             
                                                           k8s:io.kubernetes.pod.namespace=default                                                                     
                                                           k8s:org=alliance                                                                                            
52035      Disabled           Disabled          4          reserved:health                                                f00d::ac14:0:0:cb43   172.20.0.92    ready   
```

查看具体策略内容 `kubectl describe cnp rule1`



##### 6. L7 安全策略

上述的策略可以进行简单的安全防护了，但是“死星”的这个系统还有很多复杂的功能；比如它还提供了一个内部维护接口，如果被不合理调用将带来严重灾难性后果，也许“联盟”勇士劫持了一架“帝国”飞船正在进行这个任务（虽然我们内心希望他能够成功摧毁“死星”）。不幸的是“死星”系统设计者考虑到这个风险，它有办法严格限制每架飞船能够请求的权限。

没有限制飞船请求权限时，如下运行：

```bash
$ kubectl exec tiefighter -- curl -s -XPUT deathstar.default.svc.cluster.local/v1/exhaust-port
Panic: deathstar exploded

goroutine 1 [running]:
main.HandleGarbage(0x2080c3f50, 0x2, 0x4, 0x425c0, 0x5, 0xa)
        /code/src/github.com/empire/deathstar/
        temp/main.go:9 +0x64
main.main()
        /code/src/github.com/empire/deathstar/
        temp/main.go:5 +0x85
```

限制L7 的安全策略，根据文件[sw_l3_l4_l7_policy.yaml](https://github.com/easzlab/kubeasz/blob/master/roles/cilium/files/star_war_example/sw_l3_l4_l7_policy.yaml) 创建 `$ kubectl apply -f sw_l3_l4_l7_policy.yaml` 后，验证如下：

```sh
$ kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
Ship landed
$ kubectl exec tiefighter -- curl -s -XPUT deathstar.default.svc.cluster.local/v1/exhaust-port
Access denied
```

我们同样可以使用 `kubectl desribe cnp`检查更新的策略，或者使用 `cilium` 命令行：

```json
$ kubectl exec -n kube-system cilium-6t5vx -- cilium policy get
[
  {
    "endpointSelector": {
      "matchLabels": {
        "any:class": "deathstar",
        "any:org": "empire",
        "k8s:io.kubernetes.pod.namespace": "default"
      }
    },
    "ingress": [
      {
        "fromEndpoints": [
          {
            "matchLabels": {
              "any:org": "empire",
              "k8s:io.kubernetes.pod.namespace": "default"
            }
          }
        ],
        "toPorts": [
          {
            "ports": [
              {
                "port": "80",
                "protocol": "TCP"
              }
            ],
            "rules": {
              "http": [
                {
                  "path": "/v1/request-landing",
                  "method": "POST"
                }
              ]
            }
          }
        ]
      }
    ],
    "labels": [
      {
        "key": "io.cilium.k8s.policy.name",
        "value": "rule1",
        "source": "k8s"
      },
      {
        "key": "io.cilium.k8s.policy.namespace",
        "value": "default",
        "source": "k8s"
      }
    ]
  }
]
Revision: 267
```

我们看到 `cilium` 可以实现 `7层 HTTP `协议的请求方法（GET/PUT/POST等）、路径（/v1/request-landing）等等安全策略；另外，它还可以防护其他应用（如：Kafka, gRPC, Elasticsearch），可以去官网文档示例学习！

##### 参考资料

- [cilium github](https://github.com/cilium/cilium)
- [cilium doc](http://docs.cilium.io/)





## 07-安装集群主要插件

目前挑选一些常用、必要的插件自动集成到安装脚本之中:

- [自动脚本](https://github.com/easzlab/kubeasz/blob/master/roles/cluster-addon/tasks/main.yml)
- 配置开关
  - 参照[配置指南](https://github.com/easzlab/kubeasz/blob/master/docs/setup/config_guide.md)，生成后在`roles/cluster-addon/defaults/main.yml`配置

```bash
cat roles/cluster-addon/defaults/main.yml 
# dns 自动安装，'dns_backend'可选"coredns"和“kubedns”
dns_install: "yes"
dns_backend: "coredns"
kubednsVer: "1.14.13"
corednsVer: "1.5.0"
kubedns_offline: "kubedns_{{ kubednsVer }}.tar"
coredns_offline: "coredns_{{ corednsVer }}.tar"
dns_offline: "{%- if dns_backend == 'coredns' -%} \
                {{ coredns_offline }} \
              {%- else -%} \
                {{ kubedns_offline }} \
              {%- endif -%}"

# metric server 自动安装
metricsserver_install: "yes"
metricsVer: "v0.3.3"
metricsserver_offline: "metrics-server_{{ metricsVer }}.tar"

# dashboard 自动安装
# 现阶段 dashboard 获取metrics仍旧依赖于heapster，因此需连带安装heapster
dashboard_install: "yes"
dashboardVer: "v1.10.1"
dashboard_offline: "dashboard_{{ dashboardVer }}.tar"
heapsterVer: "v1.5.4"
heapster_offline: "heapster_{{ heapsterVer }}.tar"

# ingress 自动安装，可选 "traefik" 和 "nginx-ingress"
ingress_install: "yes"
ingress_backend: "traefik"
traefikVer: "v1.7.12"
nginxingVer: "0.21.0"
traefik_offline: "traefik_{{ traefikVer }}.tar"
nginx_ingress_offline: "nginx_ingress_{{ nginxingVer }}.tar"

# metallb 自动安装
metallb_install: "no"
metallbVer: "v0.7.3"
# 模式选择: 二层 "layer2" 或者三层 "bgp"
metallb_protocol: "layer2"
metallb_offline: "metallb_{{ metallbVer }}.tar"
metallb_vip_pool: "192.168.1.240/29"

# efk 自动安装
#efk_install: "no"

# prometheus 自动安装
#prometheus_install: "no"
```



**脚本介绍**

- 1.根据hosts文件中配置的`CLUSTER_DNS_SVC_IP` `CLUSTER_DNS_DOMAIN`等参数生成kubedns.yaml和coredns.yaml文件
- 2.注册变量pod_info，pod_info用来判断现有集群是否已经运行各种插件
- 3.根据pod_info和`配置开关`逐个进行/跳过插件安装

**下一步**

- [创建ex-lb节点组](https://github.com/easzlab/kubeasz/blob/master/docs/setup/ex-lb.md), 向集群外提供高可用apiserver
- [创建集群持久化存储](https://github.com/easzlab/kubeasz/blob/master/docs/setup/08-cluster-storage.md)



### EX-LB 负载均衡部署

根据[HA 2x架构](https://github.com/easzlab/kubeasz/blob/master/docs/setup/00-planning_and_overall_intro.md)，k8s集群自身高可用已经不依赖于外部 lb 服务；但是有时我们要从外部访问 apiserver（比如 CI 流程），就需要 ex-lb 来请求多个 apiserver；

还有一种情况是需要[负载转发到ingress服务](https://github.com/easzlab/kubeasz/blob/master/docs/op/loadballance_ingress_nodeport.md)，也需要部署ex-lb；

**注意：当遇到公有云环境无法自建 ex-lb 服务时，可以配置对应的云负载均衡服务**

#### ex-lb 服务组件

ex-lb 服务由 keepalived 和 haproxy 组成：

- haproxy：高效代理（四层模式）转发到多个 apiserver
- keepalived：利用主备节点vrrp协议通信和虚拟地址，消除haproxy的单点故障

```bash
roles/ex-lb/
├── clean-ex-lb.yml
├── defaults
│   └── main.yml
├── ex-lb.yml
├── tasks
│   └── main.yml
└── templates
    ├── haproxy.cfg.j2
    ├── haproxy.service.j2
    ├── keepalived-backup.conf.j2
    └── keepalived-master.conf.j2
```

Haproxy支持四层和七层负载，稳定性好，根据官方文档，HAProxy可以跑满10Gbps-New benchmark of HAProxy at 10 Gbps using Myricom's 10GbE NICs (Myri-10G PCI-Express)；另外，openstack高可用也有用haproxy的。

keepalived观其名可知，保持存活，它是基于VRRP协议保证所谓的高可用或热备的，这里用来预防haproxy的单点故障。

keepalived与haproxy配合，实现master的高可用过程如下：

- 1.keepalived利用vrrp协议生成一个虚拟地址(VIP)，正常情况下VIP存活在keepalive的主节点，当主节点故障时，VIP能够漂移到keepalived的备节点，保障VIP地址高可用性。
- 2.在keepalived的主备节点都配置相同haproxy负载配置，并且监听客户端请求在VIP的地址上，保障随时都有一个haproxy负载均衡在正常工作。并且keepalived启用对haproxy进程的存活检测，一旦主节点haproxy进程故障，VIP也能切换到备节点，从而让备节点的haproxy进行负载工作。
- 3.在haproxy的配置中配置多个后端真实kube-apiserver的endpoints，并启用存活监测后端kube-apiserver，如果一个kube-apiserver故障，haproxy会将其剔除负载池。

#### 安装haproxy

- 使用apt源安装

**配置haproxy (roles/ex-lb/templates/haproxy.cfg.j2)**

```json
cat roles/ex-lb/templates/haproxy.cfg.j2 
global
        log /dev/log    local1 warning
        chroot /var/lib/haproxy
        user haproxy
        group haproxy
        daemon
        maxconn 50000
        nbproc 1

defaults
        log     global
        timeout connect 5s
        timeout client  10m
        timeout server  10m

listen kube-master
        bind 0.0.0.0:{{ EX_APISERVER_PORT }}
        mode tcp
        option tcplog
        option dontlognull
        option dontlog-normal
        balance {{ BALANCE_ALG }}
{% for host in groups['kube-master'] %}
        server {{ host }} {{ host }}:6443 check inter 5s fall 2 rise 2 weight 1
{% endfor %}

{% if INGRESS_NODEPORT_LB == "yes" %}
listen ingress-node
	bind 0.0.0.0:80
	mode tcp
        option tcplog
        option dontlognull
        option dontlog-normal
        balance {{ BALANCE_ALG }}
{% if groups['kube-node']|length > 3 %}
        server {{ groups['kube-node'][0] }} {{ groups['kube-node'][0] }}:23456 check inter 5s fall 2 rise 2 weight 1
        server {{ groups['kube-node'][1] }} {{ groups['kube-node'][1] }}:23456 check inter 5s fall 2 rise 2 weight 1
        server {{ groups['kube-node'][2] }} {{ groups['kube-node'][2] }}:23456 check inter 5s fall 2 rise 2 weight 1
{% else %}
{% for host in groups['kube-node'] %}
        server {{ host }} {{ host }}:23456 check inter 5s fall 2 rise 2 weight 1
{% endfor %}
{% endif %}
{% endif %}

{% if INGRESS_TLS_NODEPORT_LB == "yes" %}
listen ingress-node-tls
	bind 0.0.0.0:443
	mode tcp
        option tcplog
        option dontlognull
        option dontlog-normal
        balance {{ BALANCE_ALG }}
{% if groups['kube-node']|length > 3 %}
        server {{ groups['kube-node'][0] }} {{ groups['kube-node'][0] }}:23457 check inter 5s fall 2 rise 2 weight 1
        server {{ groups['kube-node'][1] }} {{ groups['kube-node'][1] }}:23457 check inter 5s fall 2 rise 2 weight 1
        server {{ groups['kube-node'][2] }} {{ groups['kube-node'][2] }}:23457 check inter 5s fall 2 rise 2 weight 1
{% else %}
{% for host in groups['kube-node'] %}
        server {{ host }} {{ host }}:23457 check inter 5s fall 2 rise 2 weight 1
{% endfor %}
{% endif %}
{% endif %}
```

配置由全局配置和三个listen配置组成：

- listen kube-master 用于转发至多个apiserver
- listen ingress-node 用于转发至node节点的ingress http服务，[参阅](https://github.com/easzlab/kubeasz/blob/master/docs/op/loadballance_ingress_nodeport.md)
- listen ingress-node-tls 用于转发至node节点的ingress https服务

如果用apt安装的话，可以在/usr/share/doc/haproxy目录下找到配置指南configuration.txt.gz，全局和默认配置这里不展开，关注`listen` 代理设置模块，各项配置说明：

- 名称 kube-master
- bind 监听客户端请求的地址/端口，保证监听master的VIP地址和端口
- mode 选择四层负载模式 (当然你也可以选择七层负载，请查阅指南，适当调整)
- balance 选择负载算法 (负载算法也有很多供选择)



#### 安装keepalived

- 使用apt源安装

**配置keepalived主节点 [keepalived-master.conf.j2](https://github.com/easzlab/kubeasz/blob/master/roles/ex-lb/templates/keepalived-master.conf.j2)**

```bash
global_defs {
    router_id lb-master-{{ inventory_hostname }}
}

vrrp_script check-haproxy {
    script "killall -0 haproxy"
    interval 5
    weight -60
}

vrrp_instance VI-kube-master {
    state MASTER
    priority 120
    unicast_src_ip {{ inventory_hostname }}
    unicast_peer {
{% for h in groups['ex-lb'] %}{% if h != inventory_hostname %}
        {{ h }}
{% endif %}{% endfor %}
    }
    dont_track_primary
    interface {{ LB_IF }}
    virtual_router_id {{ ROUTER_ID }}
    advert_int 3
    track_script {
        check-haproxy
    }
    virtual_ipaddress {
        {{ EX_APISERVER_VIP }}
    }
}
```

- vrrp_script 定义了监测haproxy进程的脚本，利用shell 脚本`killall -0 haproxy` 进行检测进程是否存活，如果进程不存在，根据`weight -30`设置将主节点优先级降低30，这样原先备节点将变成主节点。
- vrrp_instance 定义了vrrp组，包括优先级、使用端口、router_id、心跳频率、检测脚本、虚拟地址VIP等
- 特别注意 `virtual_router_id` 标识了一个 VRRP组，在同网段下必须唯一，否则出现 `Keepalived_vrrp: bogus VRRP packet received on eth0 !!!`类似报错
- 配置 vrrp 协议通过单播发送

**配置keepalived备节点 [keepalived-backup.conf.j2](https://github.com/easzlab/kubeasz/blob/master/roles/ex-lb/templates/keepalived-backup.conf.j2)**

- 备节点的配置类似主节点，除了优先级和检测脚本，其他如 `virtual_router_id` `advert_int` `virtual_ipaddress`必须与主节点一致

**启动 keepalived 和 haproxy 后验证**

- lb 节点验证

```bash
systemctl status haproxy 	# 检查进程状态
journalctl -u haproxy		# 检查进程日志是否有报错信息
systemctl status keepalived 	# 检查进程状态
journalctl -u keepalived	# 检查进程日志是否有报错信息
```

- 在 keepalived 主节点

```bash
ip a				# 检查 master的 VIP地址是否存在
```

#### keepalived 主备切换演练

1. 尝试关闭 keepalived主节点上的 haproxy进程，然后在keepalived 备节点上查看 master的 VIP地址是否能够漂移过来，并依次检查上一步中的验证项。
2. 尝试直接关闭 keepalived 主节点系统，检查各验证项。





### K8S 集群存储

#### 前言

在kubernetes(k8s)中对于存储的资源抽象了两个概念，分别是PersistentVolume(PV)、PersistentVolumeClaim(PVC)。

- PV是集群中的资源
- PVC是对这些资源的请求。

如上面所说PV和PVC都只是抽象的概念，在k8s中是通过插件的方式提供具体的存储实现。目前包含有NFS、iSCSI和云提供商指定的存储系统，更多的存储实现[参考官方文档](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes)。

这里PV又有两种提供方式: 静态或者动态。 本篇以介绍 **NFS存储** 为例，讲解k8s 众多存储方案中的一个实现。

#### 静态 PV

首先我们需要一个NFS服务器，用于提供底层存储。通过文档[nfs-server](https://github.com/easzlab/kubeasz/blob/master/docs/guide/nfs-server.md)，我们可以创建一个NFS服务器。

> ##### 创建 NFS 服务器
>
> NFS 允许系统将其目录和文件共享给网络上的其他系统。通过 NFS，用户和应用程序可以访问远程系统上的文件，就象它们是本地文件一样。
>
> ##### 安装
>
> Ubuntu 16.04 键入以下命令安装 NFS 服务器：
>
> ```
> apt install nfs-kernel-server
> ```
>
> ##### 配置
>
> 编辑`/etc/exports`文件添加需要共享目录，每个目录的设置独占一行，编写格式如下：
>
> ```
> NFS共享目录路径 客户机IP或者名称(参数1,参数2,...,参数n)
> ```
>
> 例如：
>
> ```
> /home *(ro,sync,insecure,no_root_squash)
> /share 192.168.1.0/24(rw,sync,insecure,no_subtree_check,no_root_squash)
> ```
>
> | 参数             | 说明                                                         |
> | ---------------- | ------------------------------------------------------------ |
> | ro               | 只读访问                                                     |
> | rw               | 读写访问                                                     |
> | sync             | 所有数据在请求时写入共享                                     |
> | async            | nfs在写入数据前可以响应请求                                  |
> | secure           | nfs通过1024以下的安全TCP/IP端口发送                          |
> | insecure         | nfs通过1024以上的端口发送                                    |
> | wdelay           | 如果多个用户要写入nfs目录，则归组写入（默认）                |
> | no_wdelay        | 如果多个用户要写入nfs目录，则立即写入，当使用async时，无需此设置 |
> | hide             | 在nfs共享目录中不共享其子目录                                |
> | no_hide          | 共享nfs目录的子目录                                          |
> | subtree_check    | 如果共享/usr/bin之类的子目录时，强制nfs检查父目录的权限（默认） |
> | no_subtree_check | 不检查父目录权限                                             |
> | all_squash       | 共享文件的UID和GID映射匿名用户anonymous，适合公用目录        |
> | no_all_squash    | 保留共享文件的UID和GID（默认）                               |
> | root_squash      | root用户的所有请求映射成如anonymous用户一样的权限（默认）    |
> | no_root_squash   | root用户具有根目录的完全管理访问权限                         |
> | anonuid=xxx      | 指定nfs服务器/etc/passwd文件中匿名用户的UID                  |
> | anongid=xxx      | 指定nfs服务器/etc/passwd文件中匿名用户的GID                  |
>
> - 注1：尽量指定主机名或IP或IP段最小化授权可以访问NFS 挂载的资源的客户端
> - 注2：经测试参数insecure必须要加，否则客户端挂载出错mount.nfs: access denied by server while mounting
>
> ##### 启动
>
> 配置完成后，您可以在终端提示符后运行以下命令来启动 NFS 服务器：
>
> ```
> systemctl start nfs-kernel-server.service
> ```
>
> ##### 客户端挂载
>
> Ubuntu 16.04，首先需要安装 `nfs-common` 包
>
> ```
> apt install nfs-common
> ```
>
> CentOS 7, 需要安装 `nfs-utils` 包
>
> ```
> yum install nfs-utils
> ```
>
> 使用 mount 命令来挂载其他机器共享的 NFS 目录。可以在终端提示符后输入以下类似的命令：
>
> ```
> mount example.hostname.com:/ubuntu /local/ubuntu
> ```
>
> 挂载点 /local/ubuntu 目录必须已经存在。而且在 /local/ubuntu 目录中没有文件或子目录。
>
> 另一个挂载NFS 共享的方式就是在 /etc/fstab 文件中添加一行。该行必须指明 NFS 服务器的主机名、服务器输出的目录名以及挂载 NFS 共享的本机目录。
>
> 以下是在 /etc/fstab 中的常用语法：
>
> ```sh
> example.hostname.com:/ubuntu /local/ubuntu nfs rsize=8192,wsize=8192,timeo=14,intr
> ```

- 创建静态 pv，指定容量，访问模式，回收策略，存储类等；参考[这里](https://github.com/feiskyer/kubernetes-handbook/blob/master/zh/concepts/persistent-volume.md)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-es-0
spec:
  capacity:
    storage: 4Gi
  accessModes:
    - ReadWriteMany
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: "es-storage-class"
  nfs:
    # 根据实际共享目录修改
    path: /share/es0
    # 根据实际 nfs服务器地址修改
    server: 192.168.1.208
```

- 创建 pvc即可绑定使用上述 pv了，具体请看后文 test pod例子



#### 创建动态PV

在一个工作k8s 集群中，`PVC`请求会很多，如果每次都需要管理员手动去创建对应的 `PV`资源，那就很不方便；因此 K8S还提供了多种 `provisioner`来动态创建 `PV`，不仅节省了管理员的时间，还可以根据`StorageClasses`封装不同类型的存储供 PVC 选用。

项目中的 `role: cluster-storage`目前支持自建nfs 和aliyun_nas 的动态`provisioner`

- 1.编辑自定义配置文件：roles/cluster-storage/defaults/main.yml

```yaml
# 比如创建nfs provisioner
storage:
  nfs:
    enabled: "yes"
    server: "192.168.1.8"
    server_path: "/data/nfs"
    storage_class: "class-nfs-01"
    provisioner_name: "nfs-provisioner-01"
```

- 2.创建 nfs provisioner

```bash
$ ansible-playbook /etc/ansible/roles/cluster-storage/cluster-storage.yml
# 执行成功后验证
$ kubectl get pod --all-namespaces |grep nfs-prov
kube-system   nfs-provisioner-01-6b7fbbf9d4-bh8lh        1/1       Running   0          1d
```

**注意** k8s集群可以使用多个nfs provisioner，重复上述步骤1、2：修改使用不同的`nfs server` `nfs_storage_class` `nfs_provisioner_name`后执行创建即可。



#### 验证使用动态 PV

行以下命令进行创建：

```bash
$ kubectl apply -f test.yaml

# 验证测试pod
$ kubectl get pod --all-namespaces |grep test
default       test                                       1/1       Running   0          1m

# 验证自动创建的pv 资源，
$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM                STORAGECLASS           REASON    AGE
pvc-8f1b4ced-92d2-11e8-a41f-5254008ec7c0   1Mi        RWX            Delete           Bound     default/test-claim   nfs-dynamic-class-01             3m

# 验证PVC已经绑定成功：STATUS字段为 Bound
$ kubectl get pvc
NAME         STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS           AGE
test-claim   Bound     pvc-8f1b4ced-92d2-11e8-a41f-5254008ec7c0   1Mi        RWX            nfs-dynamic-class-01   3m
```

另外，Pod启动完成后，在挂载的目录中创建一个`SUCCESS`文件。我们可以到NFS服务器去看下：

```bash
.
└── default-test-claim-pvc-a877172b-5f49-11e8-b675-d8cb8ae6325a
    └── SUCCESS
```

如上，可以发现挂载的时候，nfs-client根据PVC自动创建了一个目录，我们Pod中挂载的`/mnt`，实际引用的就是该目录，而我们在`/mnt`下创建的`SUCCESS`文件，也自动写入到了这里。

**后续**

后面当我们需要为上层应用提供持久化存储时，只需要提供`StorageClass`即可。很多应用都会根据`StorageClass`来创建他们的所需的PVC, 最后再把PVC挂载到他们的Deployment或StatefulSet中使用，比如：efk、jenkins等



- 命令行工具 [easzctl介绍](https://github.com/easzlab/kubeasz/blob/master/docs/setup/easzctl_cmd.md)
- 公有云自建集群 [部署指南](https://github.com/easzlab/kubeasz/blob/master/docs/setup/kubeasz_on_public_cloud.md)



# 常用插件

|                                                              |                                                              |                                                              |                                                              |                                                              |                                                              |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **常用插件**[+](https://github.com/easzlab/kubeasz/blob/master/docs/guide/index.md) | [DNS](https://github.com/easzlab/kubeasz/blob/master/docs/guide/kubedns.md) | [dashboard](https://github.com/easzlab/kubeasz/blob/master/docs/guide/dashboard.md) | [metrics-server](https://github.com/easzlab/kubeasz/blob/master/docs/guide/metrics-server.md) | [prometheus](https://github.com/easzlab/kubeasz/blob/master/docs/guide/prometheus.md) | [efk](https://github.com/easzlab/kubeasz/blob/master/docs/guide/efk.md) |
| **集群管理**[+](https://github.com/easzlab/kubeasz/blob/master/docs/op/op-index.md) | [管理node节点](https://github.com/easzlab/kubeasz/blob/master/docs/op/op-node.md) | [管理master节点](https://github.com/easzlab/kubeasz/blob/master/docs/op/op-master.md) | [管理etcd节点](https://github.com/easzlab/kubeasz/blob/master/docs/op/op-etcd.md) | [升级集群](https://github.com/easzlab/kubeasz/blob/master/docs/op/upgrade.md) | [备份恢复](https://github.com/easzlab/kubeasz/blob/master/docs/op/cluster_restore.md) |
| **特性实验**                                                 | [NetworkPolicy](https://github.com/easzlab/kubeasz/blob/master/docs/guide/networkpolicy.md) | [RollingUpdate](https://github.com/easzlab/kubeasz/blob/master/docs/guide/rollingupdateWithZeroDowntime.md) | [HPA](https://github.com/easzlab/kubeasz/blob/master/docs/guide/hpa.md) |                                                              |                                                              |
| **周边生态**                                                 | [harbor](https://github.com/easzlab/kubeasz/blob/master/docs/guide/harbor.md) | [helm](https://github.com/easzlab/kubeasz/blob/master/docs/guide/helm.md) | [jenkins](https://github.com/easzlab/kubeasz/blob/master/docs/guide/jenkins.md) | [gitlab](https://github.com/easzlab/kubeasz/blob/master/docs/guide/gitlab/readme.md) |                                                              |
| **应用实践**                                                 | [go web应用部署](https://github.com/easzlab/kubeasz/blob/master/docs/practice/go_web_app) | [java应用部署](https://github.com/easzlab/kubeasz/blob/master/docs/practice/java_war_app.md) | [elasticsearch集群](https://github.com/easzlab/kubeasz/blob/master/docs/practice/es_cluster.md) | [mariadb集群](https://github.com/easzlab/kubeasz/blob/master/docs/practice/mariadb_cluster.md) |                                                              |
| **推荐工具**                                                 | [kuboard](https://github.com/easzlab/kubeasz/blob/master/docs/guide/kuboard.md) | [k9s](https://github.com/derailed/k9s)                       | [octant](https://github.com/vmware-tanzu/octant)             | [KubeSphere容器平台](https://github.com/easzlab/kubeasz/blob/master/docs/guide/kubesphere.md) |                                                              |





## 部署集群 DNS

DNS 是 k8s 集群首先需要部署的，集群中的其他 pods 使用它提供域名解析服务；主要可以解析 `集群服务名 SVC` 和 `Pod hostname`；目前 k8s v1.9+ 版本可以有两个选择：`kube-dns` 和 `coredns`（推荐），可以选择其中一个部署安装。



### 部署 dns

配置文件参考 `https://github.com/kubernetes/kubernetes` 项目目录 `kubernetes/cluster/addons/dns`

- 安装

目前 kubeasz 已经自动集成安装 dns 组件，配置模板位于`roles/cluster-addon/templates/`目录

- 集群 pod默认继承 node的dns 解析，修改 kubelet服务启动参数 --resolv-conf=""，可以更改这个特性，详见 kubelet 启动参数



### 验证 dns服务

新建一个测试nginx服务

```bash
kubectl run nginx --image=nginx --expose --port=80
```

确认nginx服务

```bash
kubectl get pod|grep nginx
nginx-7cbc4b4d9c-fl46v   1/1       Running   0          1m

kubectl get svc|grep nginx
nginx        ClusterIP   10.68.33.167   <none>        80/TCP    1m
```

测试pod alpine

```bash
kubectl run test --rm -it --image=alpine /bin/sh
If you don't see a command prompt, try pressing enter.
/ # cat /etc/resolv.conf
nameserver 10.68.0.2
search default.svc.cluster.local. svc.cluster.local. cluster.local.
options ndots:5
# 测试集群内部服务解析
/ # nslookup nginx.default.svc.cluster.local
Server:    10.68.0.2
Address 1: 10.68.0.2 kube-dns.kube-system.svc.cluster.local

Name:      nginx
Address 1: 10.68.33.167 nginx.default.svc.cluster.local
/ # nslookup kubernetes.default.svc.cluster.local
Server:    10.68.0.2
Address 1: 10.68.0.2 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.68.0.1 kubernetes.default.svc.cluster.local
# 测试外部域名的解析，默认集成node的dns解析
/ # nslookup www.baidu.com
Server:    10.68.0.2
Address 1: 10.68.0.2 kube-dns.kube-system.svc.cluster.local

Name:      www.baidu.com
Address 1: 180.97.33.108
Address 2: 180.97.33.107
/ #
```

- Note1: 如果你使用`calico`网络组件，通过命令`ansible-playbook 90.setup.yml`安装完集群后，直接安装dns组件，可能会出现如下BUG，分析是因为calico分配pod地址时候会从网段的第一个地址（网络地址）开始，详见提交的 [ISSUE #1710](https://github.com/projectcalico/calico/issues/1710)，临时解决办法为手动删除POD，重新创建后获取后面的IP地址

```bash
# BUG出现现象
$ kubectl get pod --all-namespaces -o wide
NAMESPACE     NAME                                       READY     STATUS             RESTARTS   AGE       IP              NODE
default       busy-5cc98488d4-s894w                      1/1       Running            0          28m       172.20.24.193   192.168.97.24
kube-system   calico-kube-controllers-6597d9c664-nq9hn   1/1       Running            0          1h        192.168.97.24   192.168.97.24
kube-system   calico-node-f8gnf                          2/2       Running            0          1h        192.168.97.24   192.168.97.24
kube-system   kube-dns-69bf9d5cc9-c68mw                  0/3       CrashLoopBackOff   27         31m       172.20.24.192   192.168.97.24

# 解决办法，删除pod，自动重建
$ kubectl delete pod -n kube-system kube-dns-69bf9d5cc9-c68mw
```

- Note2: 使用` kubectl run test -it --rm --image=busybox /bin/sh` 进行解析测试可能会失败, busybox内的nslookup程序有bug, 详见 https://github.com/kubernetes/dns/issues/109





## Dashboard

本文档基于 dashboard 1.10.1版本，k8s版本 1.13.x。因 dashboard 1.7 以后默认开启了自带的登录验证机制，因此不同版本登录有差异：

- 旧版（<= 1.6）建议通过apiserver访问，直接通过apiserver 认证授权机制去控制 dashboard权限，详见[旧版文档](https://github.com/easzlab/kubeasz/blob/master/docs/guide/dashboard.1.6.3.md)
- 新版（>= 1.7）可以使用自带的登录界面，使用不同Service Account Tokens 去控制访问 dashboard的权限



### 部署

如果之前已按照本项目部署dashboard1.6.3，先删除旧版本：`kubectl delete -f /etc/ansible/manifests/dashboard/1.6.3/`

新版配置文件参考 https://github.com/kubernetes/dashboard

- 增加了通过`api-server`方式访问dashboard
- 增加了`NodePort`方式暴露服务，这样集群外部可以使用 `https://NodeIP:NodePort` (注意是https不是http，区别于1.6.3版本) 直接访问 dashboard。

**安装部署**

```bash
# 部署dashboard 主yaml配置文件
$ kubectl apply -f /etc/ansible/manifests/dashboard/kubernetes-dashboard.yaml

# 创建可读可写 admin Service Account
$ kubectl apply -f /etc/ansible/manifests/dashboard/admin-user-sa-rbac.yaml

# 创建只读 read Service Account
$ kubectl apply -f /etc/ansible/manifests/dashboard/read-user-sa-rbac.yaml
```

**验证**

```bash
# 查看pod 运行状态
$ kubectl get pod -n kube-system | grep dashboard
kubernetes-dashboard-7c74685c48-9qdpn   1/1       Running   0          22s

# 查看dashboard service
$ kubectl get svc -n kube-system|grep dashboard
kubernetes-dashboard   NodePort    10.68.219.38   <none>        443:24108/TCP                   53s

# 查看集群服务
$ kubectl cluster-info|grep dashboard
$ kubernetes-dashboard is running at https://192.168.1.1:6443/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy

# 查看pod 运行日志
$ kubectl logs kubernetes-dashboard-7c74685c48-9qdpn -n kube-system
```

- 由于还未部署 Heapster 插件，当前 dashboard 不能展示 Pod、Nodes 的 CPU、内存等 metric 图形，后续部署 heapster后自然能够看到



### 访问控制

因为dashboard 作为k8s 原生UI，能够展示各种资源信息，甚至可以有修改、增加、删除权限，所以有必要对访问进行认证和控制，本项目部署的集群有以下安全设置：详见 [apiserver配置模板](https://github.com/easzlab/kubeasz/blob/master/roles/kube-master/templates/kube-apiserver.service.j2)

- 启用 `TLS认证` `RBAC授权`等安全特性
- 关闭 apiserver非安全端口8080的外部访问`--insecure-bind-address=127.0.0.1`
- 关闭匿名认证`--anonymous-auth=false`
- 可选启用基本密码认证 `--basic-auth-file=/etc/kubernetes/ssl/basic-auth.csv`，[密码文件模板](https://github.com/easzlab/kubeasz/blob/master/roles/kube-master/templates/basic-auth.csv.j2)中按照每行(密码,用户名,序号)的格式，可以定义多个用户；kubeasz 1.0.0 版本以后默认关闭 basic-auth，可以在 roles/kube-master/defaults/main.yml 选择开启

新版 dashboard可以有多层访问控制，首先与旧版一样可以使用apiserver 方式登录控制：

- 第一步通过api-server本身安全认证流程，与之前1.6.3版本相同，这里不再赘述
  - 如需（用户名/密码）认证，kubeasz 1.0.0以后使用 `easzctl basic-auth -s` 开启
- 第二步通过dashboard自带的登录流程，使用`Kubeconfig` `Token`等方式登录

**注意：** 如果集群已启用 ingress tls的话，可以[配置ingress规则访问dashboard](https://github.com/easzlab/kubeasz/blob/master/docs/guide/ingress-tls.md#配置-dashboard-ingress)



### 演示新登录方式

为演示方便这里使用 `https://NodeIP:NodePort` 方式访问 dashboard，支持两种登录方式：Kubeconfig、令牌(Token)

- 令牌登录（admin）

选择“令牌(Token)”方式登录，复制下面输出的admin token 字段到输入框

```bash
# 创建Service Account 和 ClusterRoleBinding
$ kubectl apply -f /etc/ansible/manifests/dashboard/admin-user-sa-rbac.yaml

# 获取 Bearer Token，找到输出中 ‘token:’ 开头那一行
$ kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```

- 令牌登录（只读）

选择“令牌(Token)”方式登录，复制下面输出的read token 字段到输入框

```bash
# 创建Service Account 和 ClusterRoleBinding
$ kubectl apply -f /etc/ansible/manifests/dashboard/read-user-sa-rbac.yaml

# 获取 Bearer Token，找到输出中 ‘token:’ 开头那一行
$ kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep read-user | awk '{print $1}')
```

- Kubeconfig登录（admin） Admin kubeconfig文件默认位置：`/root/.kube/config`，该文件中默认没有token字段，使用Kubeconfig方式登录，还需要将token追加到该文件中，完整的文件格式如下：

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdxxxxxxxxxxxxxx
    server: https://192.168.1.2:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: admin
  name: kubernetes
current-context: kubernetes
kind: Config
preferences: {}
users:
- name: admin
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRxxxxxxxxxxx
    client-key-data: LS0tLS1CRUdJTxxxxxxxxxxxxxx
    token: eyJhbGcixxxxxxxxxxxxxxxx
```

- Kubeconfig登录（只读） 首先[创建只读权限 kubeconfig文件](https://github.com/easzlab/kubeasz/blob/master/docs/op/readonly_kubectl.md)，然后类似追加只读token到该文件，略。



### 参考

- 1.[Dashboard Access control](https://github.com/kubernetes/dashboard/wiki/Access-control)
- 2.[a-read-only-kubernetes-dashboard](https://blog.cowger.us/2018/07/03/a-read-only-kubernetes-dashboard.html)





## Metrics Server

从 v1.8 开始，资源使用情况的度量（如容器的 CPU 和内存使用）可以通过 Metrics API 获取；前提是集群中要部署 Metrics Server，它从Kubelet 公开的Summary API采集指标信息，关于更多的背景介绍请参考如下文档：

- Metrics Server[设计提案](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/instrumentation/metrics-server.md)

大致是说它符合k8s的监控架构设计，受heapster项目启发，并且比heapster优势在于：访问不需要apiserver的代理机制，提供认证和授权等；很多集群内组件依赖它（HPA,scheduler,kubectl top），因此它应该在集群中默认运行；部分k8s集群的安装工具已经默认集成了Metrics Server的安装，以下概述下它的安装：

- 1.metric-server是扩展的apiserver，依赖于[kube-aggregator](https://github.com/kubernetes/kube-aggregator)，因此需要在apiserver中开启相关参数。
- 2.需要在集群中运行deployment处理请求

从kubeasz 0.1.0 开始，metrics-server已经默认集成在集群安装脚本中，请查看`roles/cluster-addon/defaults/main.yml`中的设置

#### 安装

默认已集成在90.setup.yml中，如果分步请执行`ansible-play /etc/ansible/07.cluster-addon.yml`

- 1.设置apiserver相关[参数](https://github.com/easzlab/kubeasz/blob/master/roles/kube-master/templates/kube-apiserver.service.j2)

```
... # 省略
  --requestheader-client-ca-file={{ ca_dir }}/ca.pem \
  --requestheader-allowed-names=aggregator \
  --requestheader-extra-headers-prefix=X-Remote-Extra- \
  --requestheader-group-headers=X-Remote-Group \
  --requestheader-username-headers=X-Remote-User \
  --proxy-client-cert-file={{ ca_dir }}/aggregator-proxy.pem \
  --proxy-client-key-file={{ ca_dir }}/aggregator-proxy-key.pem \
  --enable-aggregator-routing=true \
```

- 2.生成[aggregator proxy相关证书](https://github.com/easzlab/kubeasz/blob/master/roles/kube-master/tasks/main.yml)

参考1：https://kubernetes.io/docs/tasks/access-kubernetes-api/configure-aggregation-layer/
参考2：https://kubernetes.io/docs/tasks/access-kubernetes-api/setup-extension-api-server/

#### 验证

- 查看生成的新api：v1beta1.metrics.k8s.io

```bash
$ kubectl get apiservice|grep metrics
v1beta1.metrics.k8s.io                 1d
```

- 查看kubectl top命令（无需额外安装heapster）

```bash
$ kubectl top node
NAME           CPU(cores)   CPU%      MEMORY(bytes)   MEMORY%   
192.168.1.1   116m         2%        2342Mi          60%       
192.168.1.2   79m          1%        1824Mi          47%       
192.168.1.3   82m          2%        1897Mi          49%  

$ kubectl top pod --all-namespaces 	# 输出略
```

- 验证基于metrics-server实现的基础hpa自动缩放，请参考[hpa.md](https://github.com/easzlab/kubeasz/blob/master/docs/guide/hpa.md)

#### 补充

目前dashboard插件如果想在界面上显示资源使用率，它还依赖于`heapster`；另外，测试发现k8s 1.8版本的`kubectl top`也依赖`heapster`，因此建议补充安装`heapster`，无需安装`influxdb`和`grafana`。

```
$ kubectl apply -f /etc/ansible/manifests/heapster/heapster.yaml
```





## Prometheus

随着`heapster`项目停止更新并慢慢被`metrics-server`取代，集群监控这项任务也将最终转移。`prometheus`的监控理念、数据结构设计其实相当精简，包括其非常灵活的查询语言；但是对于初学者来说，想要在k8s集群中实践搭建一套相对可用的部署却比较麻烦，由此还产生了不少专门的项目（如：[prometheus-operator](https://github.com/coreos/prometheus-operator)），本文介绍使用`helm chart`部署集群的prometheus监控。

- `helm`已成为`CNCF`独立托管项目，预计会更加流行起来



### 前提

- 安装 helm：以本项目[安全安装helm](https://github.com/easzlab/kubeasz/blob/master/docs/guide/helm.md)为例
- 安装 [kube-dns](https://github.com/easzlab/kubeasz/blob/master/docs/guide/kubedns.md)



### 准备

安装目录概览 `ll /etc/ansible/manifests/prometheus`

```bash
drwx------  3 root root  4096 Jun  3 22:42 grafana/
-rw-r-----  1 root root 67875 Jun  4 22:47 grafana-dashboards.yaml
-rw-r-----  1 root root   690 Jun  4 09:34 grafana-settings.yaml
-rw-r-----  1 root root  1105 May 30 16:54 prom-alertrules.yaml
-rw-r-----  1 root root   474 Jun  5 10:04 prom-alertsmanager.yaml
drwx------  3 root root  4096 Jun  2 21:39 prometheus/
-rw-r-----  1 root root   294 May 30 18:09 prom-settings.yaml
```

- 目录`prometheus/`和`grafana/`即官方的helm charts，可以使用`helm fetch --untar stable/prometheus` 和 `helm fetch --untar stable/grafana`下载，本安装不会修改任何官方charts里面的内容，这样方便以后跟踪charts版本的更新
- `prom-settings.yaml`：个性化prometheus安装参数，比如禁用PV，禁用pushgateway，设置nodePort等
- `prom-alertrules.yaml`：配置告警规则
- `prom-alertsmanager.yaml`：配置告警邮箱设置等
- `grafana-settings.yaml`：个性化grafana安装参数，比如用户名密码，datasources，dashboardProviders等
- `grafana-dashboards.yaml`：预设置dashboard



### 安装

```bash
$ source ~/.bashrc
$ cd /etc/ansible/manifests/prometheus
# 安装 prometheus chart，如果你的helm安装没有启用tls证书，请忽略--tls参数
$ helm install --tls \
        --name monitor \
        --namespace monitoring \
        -f prom-settings.yaml \
        -f prom-alertsmanager.yaml \
        -f prom-alertrules.yaml \
        prometheus
# 安装 grafana chart
$ helm install --tls \
	--name grafana \
	--namespace monitoring \
	-f grafana-settings.yaml \
	-f grafana-dashboards.yaml \
	grafana
```



### 验证安装

```bash
# 查看相关pod和svc
$ kubectl get pod,svc -n monitoring 
NAME                                                     READY     STATUS    RESTARTS   AGE
grafana-54dc76d47d-2mk55                                 1/1       Running   0          1m
monitor-prometheus-alertmanager-6d9d9b5b96-w57bk         2/2       Running   0          2m
monitor-prometheus-kube-state-metrics-69f5d56f49-fh9z7   1/1       Running   0          2m
monitor-prometheus-node-exporter-55bwx                   1/1       Running   0          2m
monitor-prometheus-node-exporter-k8sb2                   1/1       Running   0          2m
monitor-prometheus-node-exporter-kxlr9                   1/1       Running   0          2m
monitor-prometheus-node-exporter-r5dx8                   1/1       Running   0          2m
monitor-prometheus-server-5ccfc77dff-8h9k6               2/2       Running   0          2m

NAME                                    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
grafana                                 NodePort    10.68.74.242   <none>        80:39002/TCP   1m
monitor-prometheus-alertmanager         NodePort    10.68.69.105   <none>        80:39001/TCP   2m
monitor-prometheus-kube-state-metrics   ClusterIP   None           <none>        80/TCP         2m
monitor-prometheus-node-exporter        ClusterIP   None           <none>        9100/TCP       2m
monitor-prometheus-server               NodePort    10.68.248.94   <none>        80:39000/TCP   2m
```

- 访问prometheus的web界面：`http://$NodeIP:39000`
- 访问alertmanager的web界面：`http://$NodeIP:39001`
- 访问grafana的web界面：`http://$NodeIP:39002` (默认用户密码 admin:admin，可在web界面修改)



### 管理操作

- 升级（修改配置）：修改配置请在`prom-settings.yaml` `prom-alertsmanager.yaml` 等文件中进行，保存后执行：

```bash
# 修改prometheus
$ helm upgrade --tls monitor -f prom-settings.yaml -f prom-alertsmanager.yaml -f prom-alertrules.yaml prometheus
# 修改grafana
$ helm upgrade --tls grafana -f grafana-settings.yaml -f grafana-dashboards.yaml grafana
```

- 回退：具体可以参考`helm help rollback`文档

```bash
$ helm rollback --tls monitor [REVISION]
```

- 删除

```bash
$ helm del --tls monitor --purge
$ helm del --tls grafana --purge
```



### 验证告警

- 修改`prom-alertsmanager.yaml`文件中邮件告警为有效的配置内容，并使用 helm upgrade更新安装
- 手动临时关闭 master 节点的 kubelet 服务，等待几分钟看是否有告警邮件发送

```bash
# 在 master 节点运行
$ systemctl stop kubelet
```



### [可选] 配置钉钉告警

- 创建钉钉群，获取群机器人 webhook 地址

使用钉钉创建群聊以后可以方便设置群机器人，【群设置】-【群机器人】-【添加】-【自定义】-【添加】，然后按提示操作即可，参考 https://open-doc.dingtalk.com/docs/doc.htm?spm=a219a.7629140.0.0.666d4a97eCG7XA&treeId=257&articleId=105735&docType=1

上述配置好群机器人，获得这个机器人对应的Webhook地址，记录下来，后续配置钉钉告警插件要用，格式如下

```
https://oapi.dingtalk.com/robot/send?access_token=xxxxxxxx
```

- 创建钉钉告警插件，参考 http://theo.im/blog/2017/10/16/release-prometheus-alertmanager-webhook-for-dingtalk/

```bash
# 编辑修改文件中 access_token=xxxxxx 为上一步你获得的机器人认证 token
$ vi /etc/ansible/manifests/prometheus/dingtalk-webhook.yaml
# 运行插件
$ kubectl apply -f /etc/ansible/manifests/prometheus/dingtalk-webhook.yaml
```

- 修改 alertsmanager 告警配置后，更新 helm prometheus 部署，成功后如上节测试告警发送

```bash
# 修改 alertsmanager 告警配置
$ cd /etc/ansible/manifests/prometheus
$ vi prom-alertsmanager.yaml
# 增加 receiver dingtalk，然后在 route 配置使用 receiver: dingtalk
    receivers:
    - name: dingtalk
      webhook_configs:
      - send_resolved: false
        url: http://webhook-dingtalk.monitoring.svc.cluster.local:8060/dingtalk/webhook1/send
# ...
# 更新 helm prometheus 部署
$ helm upgrade --tls monitor -f prom-settings.yaml -f prom-alertsmanager.yaml -f prom-alertrules.yaml prometheus
```

**下一步**

- 继续了解prometheus查询语言和配置文件
- 继续了解prometheus告警规则，编写适合业务应用的告警规则
- 继续了解grafana的dashboard编写，本项目参考了部分[feisky的模板](https://grafana.com/orgs/feisky/dashboards)
  如果对以上部分有心得总结，欢迎分享贡献在项目中。





## EFK

### 第一部分：EFK

`EFK` 插件是`k8s`项目的一个日志解决方案，它包括三个组件：[Elasticsearch](https://github.com/easzlab/kubeasz/blob/master/docs/guide), [Fluentd](https://github.com/easzlab/kubeasz/blob/master/docs/guide), [Kibana](https://github.com/easzlab/kubeasz/blob/master/docs/guide)；Elasticsearch 是日志存储和日志搜索引擎，Fluentd 负责把`k8s`集群的日志发送给 Elasticsearch, Kibana 则是可视化界面查看和检索存储在 ES 中的数据。

- 建议在熟悉本文档内容后使用[Log-Pilot + ES + Kibana 日志方案](https://github.com/easzlab/kubeasz/blob/master/docs/guide/log-pilot.md)

#### 准备

参考官方[部署文档](https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/fluentd-elasticsearch)的基础上使用本项目`manifests/efk/`部署，以下为几点主要的修改：

- 修改 fluentd-es-configmap.yaml 中的部分 journald 日志源（增加集群组件服务日志搜集）
- 修改官方docker镜像，方便国内下载加速
- 修改 es-statefulset.yaml 支持日志存储持久化等
- 增加自动清理日志，见后文`第四部分`

#### 安装

```bash
$ kubectl apply -f /etc/ansible/manifests/efk/
$ kubectl apply -f /etc/ansible/manifests/efk/es-without-pv/
```

#### 验证

```bash
kubectl get pods -n kube-system|grep -E 'elasticsearch|fluentd|kibana'
elasticsearch-logging-0                    1/1       Running   0          19h
elasticsearch-logging-1                    1/1       Running   0          19h
fluentd-es-v2.0.2-6c95c                    1/1       Running   0          17h
fluentd-es-v2.0.2-f2xh8                    1/1       Running   0          8h
fluentd-es-v2.0.2-pv5q5                    1/1       Running   0          8h
kibana-logging-d5cffd7c6-9lz2p             1/1       Running   0          1m
```

kibana Pod 第一次启动时会用较长时间(10-20分钟)来优化和 Cache 状态页面，可以查看 Pod 的日志观察进度，如下等待 `Ready` 状态

```bash
$ kubectl logs -n kube-system kibana-logging-d5cffd7c6-9lz2p -f
...
{"type":"log","@timestamp":"2018-03-13T07:33:00Z","tags":["listening","info"],"pid":1,"message":"Server running at http://0:5601"}
{"type":"log","@timestamp":"2018-03-13T07:33:00Z","tags":["status","ui settings","info"],"pid":1,"state":"green","message":"Status changed from uninitialized to green - Ready","prevState":"uninitialized","prevMsg":"uninitialized"}
```



#### 访问 Kibana

推荐使用`kube-apiserver`方式访问（可以使用basic-auth、证书和rbac等方式进行认证授权），获取访问 URL

- 开启 apiserver basic-auth(用户名/密码认证)：`easzctl basic-auth -s -u admin -p test1234`

```bash
$ kubectl cluster-info | grep Kibana
Kibana is running at https://192.168.1.10:8443/api/v1/namespaces/kube-system/services/kibana-logging/proxy
```

浏览器访问 URL：`https://192.168.1.10:8443/api/v1/namespaces/kube-system/services/kibana-logging/proxy`，然后使用`basic-auth`或者`证书` 的方式认证后即可，关于认证可以参考[dashboard文档](https://github.com/easzlab/kubeasz/blob/master/docs/guide/dashboard.md)

首次登录需要在`Management` - `Index Patterns` 创建 `index pattern`，可以使用默认的 logstash-* pattern，点击下一步；在 Time Filter field name 下拉框选择 @timestamp; 点击创建Index Pattern后，稍等几分钟就可以在 Discover 菜单看到 ElasticSearch logging 中汇聚的日志；



### 第二部分：日志持久化之静态PV

日志数据是存放于 `Elasticsearch POD`中，但是默认情况下它使用的是`emptyDir`存储类型，所以当 `POD`被删除或重新调度时，日志数据也就丢失了。以下讲解使用`NFS` 服务器手动（静态）创建`PV` 持久化保存日志数据的例子。

#### 配置 NFS

- 准备一个nfs服务器，如果没有可以参考[nfs-server](https://github.com/easzlab/kubeasz/blob/master/docs/guide/nfs-server.md)创建。
- 配置nfs服务器的共享目录，即修改`/etc/exports`（根据实际网段替换`192.168.1.*`），修改后重启`systemctl restart nfs-server`。

```bash
/share          192.168.1.*(rw,sync,insecure,no_subtree_check,no_root_squash)
/share/es0      192.168.1.*(rw,sync,insecure,no_subtree_check,no_root_squash)
/share/es1      192.168.1.*(rw,sync,insecure,no_subtree_check,no_root_squash)
/share/es2      192.168.1.*(rw,sync,insecure,no_subtree_check,no_root_squash)
```

#### 使用静态 PV安装 EFK

- 请按实际日志容量需求修改 `es-static-pv/es-statefulset.yaml` 文件中 volumeClaimTemplates 设置的 storage: 4Gi 大小
- 请根据实际nfs服务器地址、共享目录、容量大小修改 `es-static-pv/es-pv*.yaml` 文件中对应的设置

```bash
# 如果之前已经安装了默认的EFK，请用以下两个命令先删除它
$ kubectl delete -f /etc/ansible/manifests/efk/
$ kubectl delete -f /etc/ansible/manifests/efk/es-without-pv/

# 安装静态PV 的 EFK
$ kubectl apply -f /etc/ansible/manifests/efk/
$ kubectl apply -f /etc/ansible/manifests/efk/es-static-pv/
```

- 目录`es-static-pv` 下首先是利用 NFS服务预定义了三个 PV资源，然后在 `es-statefulset.yaml`定义中使用 `volumeClaimTemplates` 去匹配使用预定义的 PV资源；注意 PV参数：`accessModes` `storageClassName` `storage`容量大小必须两边匹配。

#### 验证安装

- 1.集群中查看 `pod` `pv` `pvc` 等资源

```bash
$ kubectl get pods -n kube-system|grep -E 'elasticsearch|fluentd|kibana'
elasticsearch-logging-0                    1/1       Running   0          10m
elasticsearch-logging-1                    1/1       Running   0          10m
fluentd-es-v2.0.2-6c95c                    1/1       Running   0          10m
fluentd-es-v2.0.2-f2xh8                    1/1       Running   0          10m
fluentd-es-v2.0.2-pv5q5                    1/1       Running   0          10m
kibana-logging-d5cffd7c6-9lz2p             1/1       Running   0          10m

$ kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                                                       STORAGECLASS       REASON    AGE
pv-es-0   4Gi        RWX            Recycle          Bound       kube-system/elasticsearch-logging-elasticsearch-logging-0   es-storage-class             1m
pv-es-1   4Gi        RWX            Recycle          Bound       kube-system/elasticsearch-logging-elasticsearch-logging-1   es-storage-class             1m
pv-es-2   4Gi        RWX            Recycle          Available                                                               es-storage-class             1m

$ kubectl get pvc --all-namespaces
NAMESPACE     NAME                                            STATUS    VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS       AGE
kube-system   elasticsearch-logging-elasticsearch-logging-0   Bound     pv-es-0   4Gi        RWX            es-storage-class   2m
kube-system   elasticsearch-logging-elasticsearch-logging-1   Bound     pv-es-1   4Gi        RWX            es-storage-class   1m
```

- 2.网页访问 `kibana`查看具体的日志，如上须等待（约15分钟） `kibana Pod`优化和 Cache 状态页面，达到 `Ready` 状态。
- 3.登录 NFS Server 查看对应目录和内部数据

```bash
$ ls /share
es0  es1  es2
```



### 第三部分：日志持久化之动态PV

`PV` 作为集群的存储资源，`StatefulSet` 依靠它实现 POD的状态数据持久化，但是当 `StatefulSet`动态伸缩时，它的 `PVC`请求也会变化，如果每次都需要管理员手动去创建对应的 `PV`资源，那就很不方便；因此 K8S还提供了 `provisioner`来动态创建 `PV`，不仅节省了管理员的时间，还可以根据不同的 `StorageClasses`封装不同类型的存储供 PVC 选用。

- 此功能需要 `API-SERVER` 参数 `--admission-control`字符串设置中包含 `DefaultStorageClass`，本项目中已经开启。
- `provisioner`指定 Volume 插件的类型，包括内置插件（如 kubernetes.io/glusterfs）和外部插件（如 external-storage 提供的 ceph.com/cephfs，nfs-client等），以下讲解使用 `nfs-client-provisioner`来动态创建 `PV`来持久化保存 `EFK`的日志数据。

#### **配置 NFS（同上）**

确保 `/etc/exports` 配置如下共享目录，并确保 `/share`目录可读可写权限，否则可能因为权限问题无法动态生成 PV的对应目录。（根据实际情况替换IP段`192.168.1.*`）

```
/share          192.168.1.*(rw,sync,insecure,no_subtree_check,no_root_squash)
```

#### 使用动态 PV安装 EFK

- 首先根据[集群存储](https://github.com/easzlab/kubeasz/blob/master/docs/setup/08-cluster-storage.md)创建nfs-client-provisioner
- 然后按实际需求修改 `es-dynamic-pv/es-statefulset.yaml` 文件中 volumeClaimTemplates 设置的 storage: 4Gi 大小

```bash
# 如果之前已经安装了默认的EFK或者静态PV EFK，请用以下命令先删除它
$ kubectl delete -f /etc/ansible/manifests/efk/
$ kubectl delete -f /etc/ansible/manifests/efk/es-without-pv/
$ kubectl delete -f /etc/ansible/manifests/efk/es-static-pv/

# 安装动态PV 的 EFK
$ kubectl apply -f /etc/ansible/manifests/efk/
$ kubectl apply -f /etc/ansible/manifests/efk/es-dynamic-pv/
```

- 首先 `nfs-client-provisioner.yaml` 创建一个工作 POD，它监听集群的 PVC请求，并当 PVC请求来到时调用 `nfs-client` 去请求 `nfs-server`的存储资源，成功后即动态生成对应的 PV资源。
- `nfs-dynamic-storageclass.yaml` 定义 NFS存储类型的类型名 `nfs-dynamic-class`，然后在 `es-statefulset.yaml`中必须使用这个类型名才能动态请求到资源。

#### **验证安装**

- 1.集群中查看 `pod` `pv` `pvc` 等资源

```bash
$ kubectl get pods -n kube-system|grep -E 'elasticsearch|fluentd|kibana'
elasticsearch-logging-0                    1/1       Running   0          10m
elasticsearch-logging-1                    1/1       Running   0          10m
fluentd-es-v2.0.2-6c95c                    1/1       Running   0          10m
fluentd-es-v2.0.2-f2xh8                    1/1       Running   0          10m
fluentd-es-v2.0.2-pv5q5                    1/1       Running   0          10m
kibana-logging-d5cffd7c6-9lz2p             1/1       Running   0          10m

$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM                                                       STORAGECLASS        REASON    AGE
pvc-50644f36-358b-11e8-9edd-525400cecc16   4Gi        RWX            Delete           Bound     kube-system/elasticsearch-logging-elasticsearch-logging-0   nfs-dynamic-class             10m
pvc-5b105ee6-358b-11e8-9edd-525400cecc16   4Gi        RWX            Delete           Bound     kube-system/elasticsearch-logging-elasticsearch-logging-1   nfs-dynamic-class             10m

$ kubectl get pvc --all-namespaces
NAMESPACE     NAME                                            STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS        AGE
kube-system   elasticsearch-logging-elasticsearch-logging-0   Bound     pvc-50644f36-358b-11e8-9edd-525400cecc16   4Gi        RWX            nfs-dynamic-class   10m
kube-system   elasticsearch-logging-elasticsearch-logging-1   Bound     pvc-5b105ee6-358b-11e8-9edd-525400cecc16   4Gi        RWX            nfs-dynamic-class   10m
```

- 2.网页访问 `kibana`查看具体的日志，如上须等待（约15分钟） `kibana Pod`优化和 Cache 状态页面，达到 `Ready` 状态。
- 3.登录 NFS Server 查看对应目录和内部数据

```bash
$ ls /share # 可以看到类似如下的目录生成
kube-system-elasticsearch-logging-elasticsearch-logging-0-pvc-50644f36-358b-11e8-9edd-525400cecc16
kube-system-elasticsearch-logging-elasticsearch-logging-1-pvc-5b105ee6-358b-11e8-9edd-525400cecc16
```



### 第四部分：日志自动清理

我们知道日志都存储在elastic集群中，且日志每天被分割成一个index，例如：

```bash
/ # curl elasticsearch-logging:9200/_cat/indices?v
health status index               uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   logstash-2019.04.29 ejMBlRcJQvqK76xIerenYg   5   1      69864            0     65.9mb         32.9mb
green  open   logstash-2019.04.28 hacNCuQVTQCUL62Sl8avOA   5   1      17558            0     21.3mb         10.6mb
green  open   .kibana_1           MVjF8lQeRDeKfoZcDhA93A   1   1          2            0     30.1kb           15kb
green  open   logstash-2019.05.05 m2aD8X9RQ3u48DvVq18x_Q   5   1      31218            0     34.4mb         17.2mb
green  open   logstash-2019.05.01 66OjwM5wT--DZaVfzUdXYQ   5   1      50610            0     54.6mb         27.1mb
green  open   logstash-2019.04.30 L3AH165jT6izjHHa5L5g0w   5   1      56401            0     55.5mb         27.8mb
...
```

因此 EFK 中的日志自动清理，只要定时去删除 es 中的 index 即可，如下命令

```bash
$ curl -X DELETE elasticsearch-logging:9200/logstash-xxxx.xx.xx
```

基于 alpine:3.8 创建镜像`es-index-rotator` [查看Dockerfile](https://github.com/easzlab/kubeasz/blob/master/dockerfiles/es-index-rotator/Dockerfile)，然后创建一个cronjob去完成清理任务

```bash
$ kubectl apply -f /etc/ansible/manifests/efk/es-index-rotator/
```

#### 验证日志清理

- 查看 cronjob

```
$ kubectl get cronjob -n kube-system 
NAME               SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
es-index-rotator   3 1 */1 * *   False     0        19h             20h
```

- 查看日志清理情况

```bash
$ kubectl get pod -n kube-system |grep es-index-rotator
es-index-rotator-1557507780-7xb89             0/1     Completed   0          19h

# 查看日志，可以了解日志清理情况
$ kubectl logs -n kube-system es-index-rotator-1557507780-7xb89 es-index-rotator 
```

HAVE FUN!



### 参考

1. [EFK 配置](https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/fluentd-elasticsearch)
2. [nfs-client-provisioner](https://github.com/kubernetes-incubator/external-storage/tree/master/nfs-client)
3. [persistent-volume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims)
4. [storage-classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)





### Log-Pilot Elasticsearch Kibana 日志解决方案

该方案是社区方案`EFK`的升级版，它支持两种搜集形式，对应容器标准输出日志和容器内的日志文件；个人使用了一把，在原有`EFK`经验的基础上非常简单、方便，值得推荐；更多的关于`log-pilot`的介绍详见链接：

- github 项目地址: https://github.com/AliyunContainerService/log-pilot
- 阿里云介绍文档: https://help.aliyun.com/document_detail/86552.html
- 介绍文档2: https://yq.aliyun.com/articles/674327

#### 安装步骤

- 1.安装 ES 集群，同[EFK](https://github.com/easzlab/kubeasz/blob/master/docs/guide/efk.md)文档
- 2.安装 Kibana，同[EFK](https://github.com/easzlab/kubeasz/blob/master/docs/guide/efk.md)文档
- 3.安装 Log-Pilot

```bash
$ kubectl apply -f /etc/ansible/manifests/efk/log-pilot/log-pilot-filebeat.yaml
```

- 4.创建示例应用，采集日志

```bash
$ cat > tomcat.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
  name: tomcat
spec:
  containers:
  - name: tomcat
    image: "tomcat:7.0"
    env:
    # 1、stdout为约定关键字，表示采集标准输出日志
    # 2、配置标准输出日志采集到ES的catalina索引下
    - name: aliyun_logs_catalina
      value: "stdout"
    # 1、配置采集容器内文件日志，支持通配符
    # 2、配置该日志采集到ES的access索引下
    - name: aliyun_logs_access
      value: "/usr/local/tomcat/logs/catalina.*.log"
    volumeMounts:
      - name: tomcat-log
        mountPath: /usr/local/tomcat/logs
  volumes:
    # 容器内文件日志路径需要配置emptyDir
    - name: tomcat-log
      emptyDir: {}
EOF

$ kubectl apply -f tomcat.yaml 
```

- 5.在 kibana 创建 Index Pattern，验证日志已搜集，如上示例应用，应创建如下 index pattern
  - catalina-*
  - access-*





## Ingress

### 简介

ingress就是从外部访问k8s集群的入口，将用户的URL请求转发到不同的service上。ingress相当于nginx反向代理服务器，它包括的规则定义就是URL的路由信息；它的实现需要部署`Ingress controller`(比如 [traefik](https://github.com/containous/traefik) [ingress-nginx](https://github.com/kubernetes/ingress-nginx) 等)，`Ingress controller`通过apiserver监听ingress和service的变化，并根据规则配置负载均衡并提供访问入口，达到服务发现的作用。

- 未配置ingress：

集群外部 -> NodePort -> K8S Service

- 配置ingress:

集群外部 -> Ingress -> K8S Service

- **注意：ingress 本身也需要部署`Ingress controller`时使用以下几种方式让外部访问**
  - 使用`NodePort`方式
  - 使用`hostPort`方式
  - 使用LoadBalancer地址方式
- 以下讲解基于`Traefik`，如果想要了解`ingress-nginx`的原理与实践，推荐阅读博客[烂泥行天下](https://www.ilanni.com/?p=14501)的相关文章



### 部署 Traefik

Traefik 提供了一个简单好用 `Ingress controller`，下文侧重讲解 ingress部署和测试例子。请查看yaml配置 [traefik-ingress.yaml](https://github.com/easzlab/kubeasz/blob/master/manifests/ingress/traefik/traefik-ingress.yaml)，参考[traefik 官方k8s例子](https://github.com/containous/traefik/tree/master/examples/k8s)

#### 安装 traefik ingress-controller

```bash
kubectl create -f /etc/ansible/manifests/ingress/traefik/traefik-ingress.yaml
```

- 注意需要配置 `RBAC`授权
- 注意`trafik pod`中 `80`端口为 traefik ingress-controller的服务端口，`8080`端口为 traefik 的管理WEB界面；为后续配置方便指定`80` 端口暴露`NodePort`端口为 `23456`(对应于在hosts配置中`NODE_PORT_RANGE`范围内可用端口)

#### 验证 traefik ingress-controller

```bash
$ kubectl get deploy -n kube-system traefik-ingress-controller
NAME                         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
traefik-ingress-controller   1         1         1            1           4m

$ kubectl get svc -n kube-system traefik-ingress-service
NAME                      TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                       AGE
traefik-ingress-service   NodePort   10.68.69.170   <none>        80:23456/TCP,8080:34815/TCP   4m
```

- 可以看到`traefik-ingress-service` 服务端口`80`暴露的nodePort确实为`23456`

#### 测试 ingress

- 首先创建测试用K8S应用，并且该应用服务不用nodePort暴露，而是用ingress方式让外部访问

```bash
$ kubectl run test-hello --image=nginx:alpine --expose --port=80

$ kubectl get deploy test-hello
NAME         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
test-hello   1         1         1            1           56s

$ kubectl get svc test-hello
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
test-hello   ClusterIP   10.68.124.115   <none>        80/TCP    1m
```

- 然后为这个应用创建 ingress，`kubectl create -f /etc/ansible/manifests/ingress/test-hello.ing.yaml`

```bash
$ vim test-hello.ing.yaml
apiVersion: networking.k8s.io/v1beta1 
kind: Ingress
metadata:
  name: test-hello
spec:
  rules:
  - host: hello.test.com
    http:
      paths:
      - path: /
        backend:
          serviceName: test-hello
          servicePort: 80
```

- 集群内部尝试访问: `curl -H Host:hello.test.com 10.68.69.170(traefik-ingress-service的服务地址)` 能够看到欢迎页面 `Welcome to nginx!`；
- 在集群外部尝试访问(假定集群一个NodeIP为 192.168.1.1): `curl -H Host:hello.test.com 192.168.1.1:23456`，也能够看到欢迎页面 `Welcome to nginx!`，说明ingress测试成功

#### 为 traefik WEB 管理页面创建 ingress 规则

```bash
kubectl create -f /etc/ansible/manifests/ingress/traefik/traefik-ui.ing.yaml
# traefik-ui.ing.yaml内容
---
apiVersion: networking.k8s.io/v1beta1 
kind: Ingress
metadata:
  name: traefik-web-ui
  namespace: kube-system
spec:
  rules:
  - host: traefik-ui.test.com
    http:
      paths:
      - path: /
        backend:
          serviceName: traefik-ingress-service
          servicePort: 8080
```

- 在集群外部可以使用 `curl -H Host:traefik-ui.test.com 192.168.1.1:23456` 尝试访问WEB管理页面，返回 `<a href="/dashboard/">Found</a>.`说明 traefik-ui的ingress配置生效了。
- 在客户端主机也可以通过修改本机 `hosts` 文件，如上例子，增加两条记录：

```
192.168.1.1	hello.test.com
192.168.1.1	traefik-ui.test.com
```

打开浏览器输入域名 `http://hello.test.com:23456` 和 `http://traefik-ui.test.com:23456` 就可以访问k8s的应用服务了。



#### 可选1: 使用`LoadBalancer`服务类型来暴露ingress，自有环境（非公有云）可以参考[metallb文档](https://github.com/easzlab/kubeasz/blob/master/docs/guide/metallb.md)

> #### metallb 网络负载均衡
>
> `Metallb`是在自有硬件上（非公有云）实现 `Kubernetes Load-balancer`的工具，由`google`团队开源，值得推荐！项目[github主页](https://github.com/google/metallb)。
>
> ##### metallb 简介
>
> 这里简单介绍下它的实现原理，具体可以参考[metallb官网](https://metallb.universe.tf/)，文档非常简洁、清晰。目前有如下的使用限制：
>
> - `Kubernetes v1.9.0`版本以上，暂不支持`ipvs`模式
> - 支持网络组件 (flannel/weave/romana), calico 部分支持
> - `layer2`和`bgp`两种模式，其中`bgp`模式需要外部网络设备支持`bgp`协议
>
> `metallb`主要实现了两个功能：地址分配和对外宣告
>
> - 地址分配：需要向网络管理员申请一段ip地址，如果是layer2模式需要这段地址与node节点地址同个网段（同一个二层）；如果是bgp模式没有这个限制。
> - 对外宣告：layer2模式使用arp协议，利用节点的mac额外宣告一个loadbalancer的ip（同mac多ip）；bgp模式下节点利用bgp协议与外部网络设备建立邻居，宣告loadbalancer的地址段给外部网络。
>
> ##### kubeasz 集成安装metallb
>
> 因bgp模式需要外部路由器的支持，这里主要选用layer2模式（如需选择bgp模式，相应修改roles/cluster-addon/templates/metallb/bgp.yaml.j2）。
>
> - 1.修改roles/cluster-addon/defaults/main.yml 配置文件相关
>
> ```
> # metallb 自动安装
> metallb_install: "yes"
> # 模式选择: 二层 "layer2" 或者三层 "bgp"
> metallb_protocol: "layer2"
> metallb_offline: "metallb_v0.7.3.tar"
> metallb_vip_pool: "192.168.1.240/29"  # 选一段与node节点相同网段的地址
> ```
>
> - 2.执行安装 `ansible-playbook 07.cluster-addon.yml`，其中controller 负责统一loadbalancer地址管理和服务监控，speaker 负责节点的loadbalancer地址的对外宣告（使用arp或者bgp网络协议），注意 **speaker是以DaemonSet 形式运行且只会调度到有node-role.kubernetes.io/metallb-speaker=true标签的节点**，所以你可以选择做speaker的节点（该节点网络性能要好），使用命令 `$ kubectl label nodes 192.168.1.43 node-role.kubernetes.io/metallb-speaker=true`
> - 3.验证metallb相关 pod
>
> ```sh
> $ kubectl get node
> NAME           STATUS                     ROLES                  AGE       VERSION
> 192.168.1.41   Ready,SchedulingDisabled   master                 4h        v1.11.3
> 192.168.1.42   Ready                      node                   4h        v1.11.3
> 192.168.1.43   Ready                      metallb-speaker,node   4h        v1.11.3
> 192.168.1.44   Ready                      metallb-speaker,node   4h        v1.11.3
> $ kubectl get pod -n metallb-system 
> NAME                        READY     STATUS    RESTARTS   AGE
> controller-9c57dbd4-798nb   1/1       Running   0          4h
> speaker-9rjmk               1/1       Running   0          4h
> speaker-n79l4               1/1       Running   0          4h
> ```
>
> - 3.创建测试应用验证 loadbalancer 地址分配
>
> ```sh
> # 创建测试应用
> $ cat > test-nginx.yaml << EOF
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: nginx3
> spec:
>   selector:
>     matchLabels:
>       app: nginx3
>   template:
>     metadata:
>       labels:
>         app: nginx3
>     spec:
>       containers:
>       - name: nginx3
>         image: nginx:1
>         ports:
>         - name: http
>           containerPort: 80
> 
> ---
> apiVersion: v1
> kind: Service
> metadata:
>   name: nginx3
> spec:
>   ports:
>   - name: http
>     port: 80
>     protocol: TCP
>     targetPort: 80
>   selector:
>     app: nginx3
>   type: LoadBalancer
> EOF
> $ kubectl apply -f test-nginx.yaml
> 
> # 查看生成的loadbalancer 地址，如下验证成功
> $ kubectl get svc
> NAME         TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)        AGE
> kubernetes   ClusterIP      10.68.0.1      <none>          443/TCP        5h
> nginx3       LoadBalancer   10.68.82.227   192.168.1.240   80:38702/TCP   1m
> ```
>
> - 4.验证使用loadbalacer 来暴露ingress的服务地址，之前在[ingress文档](https://github.com/easzlab/kubeasz/blob/master/docs/guide/ingress.md)中我们是使用nodeport方式服务类型，现在我们可以方便的使用loadbalancer类型了，使用loadbalancer地址(192.168.1.241)方便的绑定你要的域名进行访问。
>
> ```sh
> # 修改traefik-ingress 使用 LoadBalancer服务
> $ sed -i 's/NodePort$/LoadBalancer/g' /etc/ansible/manifests/ingress/traefik/traefik-ingress.yaml
> # 创建traefik-ingress
> $ kubectl apply -f /etc/ansible/manifests/ingress/traefik/traefik-ingress.yaml
> # 验证
> $ kubectl get svc --all-namespaces |grep traefik
> kube-system   traefik-ingress-service   LoadBalancer   10.68.163.243   192.168.1.241   80:23456/TCP,8080:37088/TCP   1m
> ```

```bash
# 修改traefik-ingress 使用 LoadBalancer服务
$ sed -i 's/NodePort$/LoadBalancer/g' /etc/ansible/manifests/ingress/traefik/traefik-ingress.yaml

# 创建traefik-ingress
$ kubectl apply -f /etc/ansible/manifests/ingress/traefik/traefik-ingress.yaml

# 验证
$ kubectl get svc --all-namespaces |grep traefik
kube-system   traefik-ingress-service   LoadBalancer   10.68.163.243   192.168.1.241   80:23456/TCP,8080:37088/TCP   1m
```

这时可以修改客户端本机 `hosts`文件：(如上例192.168.1.241)

```bash
192.168.1.241     hello.test.com
192.168.1.241     traefik-ui.test.com
```

打开浏览器输入域名 `http://hello.test.com` 和 `http://traefik-ui.test.com`可以正常访问。



#### 可选2: 部署`ingress-service`的负载均衡

- 利用 nginx/haproxy 等集群，可以做代理转发以去掉 `23456`这个端口。如果你的集群根据本项目部署了高可用方案，那么可以利用`LB` 节点haproxy 来做，当然如果生产环境K8S应用已经部署非常多，建议还是使用独立的 `nginx/haproxy`集群。

具体参考[配置转发 ingress nodePort](https://github.com/easzlab/kubeasz/blob/master/docs/op/loadballance_ingress_nodeport.md)，如上配置访问集群`MASTER_IP`的`80`端口时，由haproxy代理转发到实际的node节点暴露的nodePort端口上了。这时可以修改客户端本机 `hosts`文件如下：(假定 MASTER_IP=192.168.1.10)

```
192.168.1.10     hello.test.com
192.168.1.10    traefik-ui.test.com
```

打开浏览器输入域名 `http://hello.test.com` 和 `http://traefik-ui.test.com`可以正常访问。



### 使用 traefik 配置 https ingress

本文档基于 traefik 配置 https ingress 规则，请先阅读[配置基本 ingress](https://github.com/easzlab/kubeasz/blob/master/docs/guide/ingress.md)。与基本 ingress-controller 相比，需要额外配置 https tls 证书，主要步骤如下：

#### 1.准备 tls 证书

可以使用 Let's Encrypt 签发的免费证书，这里为了测试方便使用自签证书 (tls.key/tls.crt)，注意CN 配置为 ingress 的域名：

```bash
$ openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=hello.test.com"
```

#### 2.在 kube-system 命名空间创建 secret: traefik-cert，以便后面 traefik-controller 挂载该证书

```bash
$ kubectl -n kube-system create secret tls traefik-cert --key=tls.key --cert=tls.crt
```

#### 3.创建 traefik-controller，增加 traefik.toml 配置文件及https 端口暴露等，详见该 yaml 文件

```bash
$ kubectl apply -f /etc/ansible/manifests/ingress/traefik/tls/traefik-controller.yaml
```

#### 4.创建 https ingress 例子

```bash
# 创建示例应用
$ kubectl run test-hello --image=nginx:alpine --port=80 --expose
# hello-tls-ingress 示例
apiVersion: networking.k8s.io/v1beta1 
kind: Ingress
metadata:
  name: hello-tls-ingress
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: hello.test.com
    http:
      paths:
      - backend:
          serviceName: test-hello
          servicePort: 80
  tls:
  - secretName: traefik-cert

# 创建https ingress
$ kubectl apply -f /etc/ansible/manifests/ingress/traefik/tls/hello-tls.ing.yaml

# 注意根据hello示例，需要在default命名空间创建对应的secret: traefik-cert
$ kubectl create secret tls traefik-cert --key=tls.key --cert=tls.crt
```

#### 5.验证 https 访问

验证 traefik-ingress svc

```bash
$ kubectl get svc -n kube-system traefik-ingress-service 
NAME                      TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                                     AGE
traefik-ingress-service   NodePort   10.68.250.253   <none>        80:23456/TCP,443:23457/TCP,8080:35941/TCP   66m
```

可以看到项目默认使用nodePort 23456暴露traefik 80端口，nodePort 23457暴露 traefik 443端口，因此在客户端 hosts 增加记录 `$Node_IP hello.test.com`之后，可以在浏览器验证访问如下：

```bash
https://hello.test.com:23457
```

如果你已经配置了[转发 ingress nodePort](https://github.com/easzlab/kubeasz/blob/master/docs/op/loadballance_ingress_nodeport.md)，那么增加对应 hosts记录后，可以验证访问 `https://hello.test.com`

#### 配置 dashboard ingress

前提1：k8s 集群的dashboard 已安装

```bash
$ kubectl get svc -n kube-system | grep dashboard
kubernetes-dashboard      NodePort    10.68.211.168   <none>        443:39308/TCP	3d11h
```

前提2：`/etc/ansible/manifests/ingress/traefik/tls/traefik-controller.yaml`的配置文件`traefik.toml`开启了`insecureSkipVerify = true`

配置 dashboard ingress：`kubectl apply -f /etc/ansible/manifests/ingress/traefik/tls/k8s-dashboard.ing.yaml` 内容如下：

```bash
apiVersion: networking.k8s.io/v1beta1 
kind: Ingress
metadata:
  name:  kubernetes-dashboard
  namespace: kube-system
  annotations:
    traefik.ingress.kubernetes.io/redirect-entry-point: https
spec:
  rules:
  - host: dashboard.test.com
    http:
      paths:
      - path: /
        backend:
          serviceName: kubernetes-dashboard
          servicePort: 443
```

- 注意annotations 配置了 http 跳转 https 功能
- 注意后端服务是443端口

#### 参考

- [Add a TLS Certificate to the Ingress](https://docs.traefik.io/user-guide/kubernetes/#add-a-tls-certificate-to-the-ingress)







# 集群管理



## 管理 node 节点

目录

- 1.增加 kube-node 节点
- 2.增加非标准ssh端口节点
- 3.删除 kube-node 节点



### 1.增加 kube-node 节点

新增`kube-node`节点大致流程为：tools/02.addnode.yml

- [可选]新节点安装 chrony 时间同步
- 新节点预处理 prepare
- 新节点安装 docker 服务
- 新节点安装 kube-node 服务
- 新节点安装网络插件相关

#### 操作步骤

首先配置 ssh 免密码登录新增节点，然后执行 (假设待增加节点为 192.168.1.11)：

```bash
$ easzctl add-node 192.168.1.11
```

#### 验证

```bash
# 验证新节点状态
$ kubectl get node

# 验证新节点的网络插件calico 或flannel 的Pod 状态
$ kubectl get pod -n kube-system

# 验证新建pod能否调度到新节点，略
```



### 2.增加非标准ssh端口节点

目前 easzctl 暂不支持自动添加非标准 ssh 端口的节点，可以手动操作如下：

- 假设待添加节点192.168.2.1，ssh 端口 10022；配置免密登录 ssh-copy-id -p 10022 192.168.2.1，按提示输入密码
- 在 /etc/ansible/hosts文件 [kube-node] 组下添加一行：

```
192.168.2.1 ansible_ssh_port=10022
```

- 最后执行 `ansible-playbook /etc/ansible/tools/02.addnode.yml -e NODE_TO_ADD=192.168.2.1`



### 3.删除 kube-node 节点

删除 node 节点流程：tools/12.delnode.yml

- 检测是否可以删除
- 迁移节点上的 pod
- 删除 node 相关服务及文件
- 从集群删除 node

#### 操作步骤

```bash
$ easzctl del-node 192.168.1.11 # 假设待删除节点为 192.168.1.11
```



## 管理 kube-master 节点

### 1.增加 kube-master 节点

新增`kube-master`节点大致流程为：tools/03.addmaster.yml

- [可选]新节点安装 chrony 时间同步
- 新节点预处理 prepare
- 新节点安装 docker 服务
- 新节点安装 kube-master 服务
- 新节点安装 kube-node 服务
- 新节点安装网络插件相关
- 禁止业务 pod调度到新master节点
- 更新 node 节点 haproxy 负载均衡并重启



#### 操作步骤

首先配置 ssh 免密码登录新增节点，然后执行 (假设待增加节点为 192.168.1.11)：

```bash
$ easzctl add-master 192.168.1.11
```



#### 验证

```bash
# 在新节点master 服务状态
$ systemctl status kube-apiserver 
$ systemctl status kube-controller-manager
$ systemctl status kube-scheduler

# 查看新master的服务日志
$ journalctl -u kube-apiserver -f

# 查看集群节点，可以看到新 master节点 Ready, 并且禁止了POD 调度功能
$ kubectl get node
NAME           STATUS                     ROLES     AGE       VERSION
192.168.1.1    Ready,SchedulingDisabled   <none>    3h        v1.9.3
192.168.1.2    Ready,SchedulingDisabled   <none>    3h        v1.9.3
192.168.1.3    Ready                      <none>    3h        v1.9.3
192.168.1.4    Ready                      <none>    3h        v1.9.3
192.168.1.11   Ready,SchedulingDisabled   <none>    2h        v1.9.3	# 新增 master节点
```



### 2.删除 kube-master 节点

删除`kube-master`节点大致流程为：tools/13.delmaster.yml

- 检测是否可以删除
- 迁移节点 pod
- 删除 master 相关服务及文件
- 删除 node 相关服务及文件
- 从集群删除 node 节点
- 从 ansible hosts 移除节点
- 在 ansible 控制端更新 kubeconfig
- 更新 node 节点 haproxy 配置

#### 操作步骤

```bash
$ easzctl del-master 192.168.1.11  # 假设待删除节点 192.168.1.11
```



## 管理 etcd 集群

Etcd 集群支持在线改变集群成员节点，可以增加、修改、删除成员节点；不过改变成员数量仍旧需要满足集群成员多数同意原则（quorum），另外请记住集群成员数量变化的影响：

- 注意：如果etcd 集群有故障节点，务必先删除故障节点，然后添加新节点，[参考FAQ](https://etcd.io/docs/v3.4.0/faq/)
- 增加 etcd 集群节点, 提高集群稳定性
- 增加 etcd 集群节点, 提高集群读性能（所有节点数据一致，客户端可以从任意节点读取数据）
- 增加 etcd 集群节点, 降低集群写性能（所有节点数据一致，每一次写入会需要所有节点数据同步）



### 备份 etcd 数据

可以根据需要进行定期备份（使用 crontab），或者手动在任意正常 etcd 节点上执行备份：

```bash
# snapshot备份
$ ETCDCTL_API=3 etcdctl snapshot save backup.db

# 查看备份
$ ETCDCTL_API=3 etcdctl --write-out=table snapshot status backup.db
```



### etcd 集群节点操作

首先确认配置 ssh 免密码登录，然后执行 (假设待操作节点为 192.168.1.11)：

- 增加 etcd 节点：`$ easzctl add-etcd 192.168.1.11` (注意：增加 etcd 还需要根据提示输入集群内唯一的 NODE_NAME)
- 删除 etcd 节点：`$ easzctl del-etcd 192.168.1.11`



### 验证 etcd 集群

```bash
# 登录任意etcd节点验证etcd集群状态
$ export ETCDCTL_API=3 
$ etcdctl member list

# 验证所有etcd节点服务状态和日志
$ systemctl status etcd
$ journalctl -u etcd -f
```



### 重置 k8s 连接 etcd 参数

上述步骤验证成功，确认新etcd集群工作正常后，可以重新配置运行apiserver，以让 k8s 集群能够识别新的etcd集群：

```bash
# 重启 master 节点服务
$ ansible-playbook /etc/ansible/04.kube-master.yml -t restart_master

# 验证 k8s 能够识别新 etcd 集群
$ kubectl get cs
```



### 参考

- 官方文档 https://github.com/etcd-io/etcd/blob/master/Documentation/op-guide/runtime-configuration.md



## k8s 集群升级

集群升级存在一定风险，请谨慎操作。

- 项目分支`master`安装的集群可以在当前支持k8s大版本基础上升级任意小版本，比如当前安装集群为1.16.0，你可以方便的升级到任何1.16.x版本
- 项目分支`closed`（已停止更新）安装的集群目前只能进行小版本1.8.x的升级



### 备份etcd数据

- 升级前对 etcd数据做备份，在任意 etcd节点上执行：

```bash
# snapshot备份
$ ETCDCTL_API=3 etcdctl snapshot save backup.db

# 查看备份
$ ETCDCTL_API=3 etcdctl --write-out=table snapshot status backup.db
```

- `kubeasz`项目也可以方便执行 `ansible-playbook /etc/ansible/23.backup.yml`，详情阅读文档[备份恢复](https://github.com/easzlab/kubeasz/blob/master/docs/op/cluster_restore.md)



### 快速k8s版本升级

快速升级是指只升级`k8s`版本，比较常见如`Bug修复` `重要特性发布`时使用。

- 首先去官网release下载待升级的k8s版本，例如`https://dl.k8s.io/v1.11.5/kubernetes-server-linux-amd64.tar.gz`

- 解压下载的tar.gz文件，找到如下

  ```bash
  kube*
  ```

  开头的二进制，复制替换ansible控制端目录

  ```bash
  /etc/ansible/bin
  ```

  对应文件

  - kube-apiserver
  - kube-controller-manager
  - kubectl
  - kubelet
  - kube-proxy
  - kube-scheduler

- 在ansible控制端执行`ansible-playbook -t upgrade_k8s 22.upgrade.yml`即可完成k8s 升级，不会中断业务应用

如果使用 easzctl 命令行，可按如下执行：

- 首先确认待升级的集群（如果有多集群的话） `easzctl checkout <cluster_name>`
- 执行升级 `easzctl upgrade`

### 其他升级说明

其他升级是指升级k8s组件包括：`etcd版本` `docker版本`，一般不需要用到，以下仅作说明。

- 1.下载所有组件相关新的二进制解压并替换 `/etc/ansible/bin/` 目录下文件
- 2.升级 etcd: `ansible-playbook -t upgrade_etcd 02.etcd.yml`，**注意：etcd 版本只能升级不能降低！**
- 3.升级 docker （建议使用k8s官方支持的docker稳定版本）
  - 如果可以接受短暂业务中断，执行 `ansible-playbook -t upgrade_docker 03.docker.yml`
  - 如果要求零中断升级，执行 `ansible-playbook -t download_docker 03.docker.yml`，然后手动执行如下
    - 待升级节点，先应用`kubectl cordon`和`kubectl drain`命令迁移业务pod
    - 待升级节点执行 `systemctl restart docker`
    - 恢复节点可调度 `kubectl uncordon`

## K8S 集群备份与恢复

虽然 K8S 集群可以配置成多主多节点的高可用的部署，还是有必要了解下集群的备份和容灾恢复能力；在高可用k8s集群中 etcd集群保存了整个集群的状态，因此这里的备份与恢复重点就是：

- 从运行的etcd集群备份数据到磁盘文件
- 从etcd备份文件恢复数据，从而使集群恢复到备份时状态

### 备份与恢复操作说明

- 1.首先搭建一个测试集群，部署几个测试deployment，验证集群各项正常后，进行一次备份：

```bash
$ ansible-playbook /etc/ansible/23.backup.yml
```

执行完毕可以在备份目录下检查备份情况，示例如下：

```bash
/etc/ansible/.cluster/backup/
├── hosts
├── hosts-201907030954
├── snapshot-201907030954.db
├── snapshot-201907031048.db
└── snapshot.db
```

- 2.模拟误删除操作（略）
- 3.恢复集群及验证

可以在 `roles/cluster-restore/defaults/main.yml` 文件中配置需要恢复的 etcd备份版本（从上述备份目录中选取），默认使用最近一次备份；执行恢复后，需要一定时间等待 pod/svc 等资源恢复重建。

```bash
$ ansible-playbook /etc/ansible/24.restore.yml
```

如果集群主要组件（master/etcd/node）等出现不可恢复问题，可以尝试使用如下步骤 [清理](https://github.com/easzlab/kubeasz/blob/master/docs/op) --> [创建](https://github.com/easzlab/kubeasz/blob/master/docs/op) --> [恢复](https://github.com/easzlab/kubeasz/blob/master/docs/op)

```bash
$ ansible-playbook /etc/ansible/99.clean.yml
$ ansible-playbook /etc/ansible/90.setup.yml
$ ansible-playbook /etc/ansible/24.restore.yml
```

### 参考

- https://github.com/coreos/etcd/blob/master/Documentation/op-guide/recovery.md



# 特性实验 

## Network Policy

`Network Policy`提供了基于策略的网络控制，用于隔离应用并减少攻击面。它使用标签选择器模拟传统的分段网络，并通过策略控制它们之间的流量以及来自外部的流量；目前基于`linux iptables`实现，使用类似`nf_conntrack`检查记录网络流量`session`从而决定流量是否阻断；因此它是`状态检测防火墙`。

- 网络插件要支持 Network Policy，如 Calico、Romana、Weave Net

### 简单示例

实验环境：k8s v1.9, calico 2.6.5

首先部署测试用nginx服务

```bash
$ kubectl run nginx --image=nginx --replicas=3 --port=80 --expose

# 验证测试nginx服务
$ kubectl get pod -o wide 
NAME     READY     STATUS    RESTARTS   AGE   IP   NODE
nginx-7587c6fdb6-p2fpz  1/1  Running  0  55m  172.20.125.2     10.0.96.7
nginx-7587c6fdb6-pbw7c   1/1       Running   0          55m       172.20.124.2     10.0.96.6
nginx-7587c6fdb6-v48db   1/1       Running   0          55m       172.20.121.195   10.0.96.4

$ kubectl get svc nginx
NAME      TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
nginx     ClusterIP   10.68.7.183   <none>        80/TCP    1h
```

默认情况下，其他pod可以访问nginx服务

```bash
$ kubectl run busy1 --rm -it --image=busybox /bin/sh
If you don't see a command prompt, try pressing enter.
/ # wget --spider --timeout=1 nginx
Connecting to nginx (10.68.7.183:80)
```

创建`DefaultDeny Network Policy`后，其他Pod（包括namespace外部）不能访问nginx

```bash
$ cat > default-deny.yaml << EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Ingress
EOF

$ kubectl create -f default-deny.yaml
networkpolicy "default-deny" created
$ kubectl run busy1 --rm -it --image=busybox /bin/sh
If you don't see a command prompt, try pressing enter.
/ # wget --spider --timeout=1 nginx
Connecting to nginx (10.68.7.183:80)
wget: download timed out
```

创建一个允许带有access=true的Pod访问nginx的网络策略

```bash
$ cat > nginx-policy.yaml << EOF
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: access-nginx
spec:
  podSelector:
    matchLabels:
      run: nginx
  ingress:
  - from:
    - podSelector:
        matchLabels:
          access: "true"
EOF
$ kubectl create -f nginx-policy.yaml
networkpolicy "access-nginx" created

# 不带access=true标签的Pod还是无法访问nginx服务
$ kubectl run busy1 --rm -it --image=busybox /bin/sh
If you don't see a command prompt, try pressing enter.
/ # wget --spider --timeout=1 nginx
Connecting to nginx (10.68.7.183:80)
wget: download timed out

# 而带有access=true标签的Pod可以访问nginx服务
$ kubectl run busy2 --rm -it --labels="access=true" --image=busybox /bin/sh
If you don't see a command prompt, try pressing enter.
/ # wget --spider --timeout=1 nginx
Connecting to nginx (10.68.7.183:80)
```

### 示例策略解读

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 172.17.0.0/16
        except:
        - 172.17.1.0/24
    - namespaceSelector:
        matchLabels:
          project: myproject
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 6379
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 5978
```

策略作用的对象Pods：default命名空间下带有`role=db`标签的Pod

- 内向流量策略
  - 允许属于`172.17.0.0/16`网段但不属于`172.17.1.0/24`的源地址访问该对象Pods的TCP 6379端口
  - 允许带有project=myprojects标签的namespace中所有Pod访问该对象Pods的TCP 6379端口
  - 允许default命名空间下带有role=frontend标签的Pod访问该对象Pods的TCP 6379端口
  - 拒绝其他所有主动访问该对象Pods的网络流量
- 外向流量策略
  - 允许该对象Pods主动访问目的地址属于`10.0.0.0/24`网段且目的端口为TCP 5978的流量
  - 拒绝该对象Pods其他所有主动外向网络流量

### 使用场景

参考阅读[ahmetb/kubernetes-network-policy-recipes](https://github.com/ahmetb/kubernetes-network-policy-recipes) 该项目举例一些使用NetworkPolicy的场景，并有形象的配图

#### 拒绝其他namespaces访问服务

![image-20200612173606768](D:\学习资料\笔记\linux\assets\image-20200612173606768.png)

- 场景1：你的k8s集群应用按照namespaces区分生产、测试环境，你要确保生产环境不会受到测试环境错误访问影响
- 场景2：你的k8s集群有多租户应用采用namespaces区分的，你要确保多租户之间的应用隔离

在你需要隔离的命名空间创建如下策略:

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  namespace: your-ns
  name: deny-other-namespaces
spec:
  podSelector:
    matchLabels:
  ingress:
  - from:
    - podSelector: {}
```

#### 允许外部访问服务

- 场景：暴露特定Pod的特定端口给外部访问

![image-20200612173643559](D:\学习资料\笔记\linux\assets\image-20200612173643559.png)

```yaml
# 创建示例应用待暴露服务
$ kubectl run web --image=nginx --labels=app=web --port 80 --expose

# 创建网络策略
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: web-allow-external
spec:
  podSelector:
    matchLabels:
      app: web
  ingress:
  - from: []
    ports:
    - protocol: TCP
      port: 80
```





## RollingUpdate

### 1、前言

在当下微服务架构盛行的时代，用户希望应用程序时时刻刻都是可用，为了满足不断变化的新业务，需要不断升级更新应用程序，有时可能需要频繁的发布版本。实现"零停机"、“零感知”的持续集成(Continuous Integration)和持续交付/部署(Continuous Delivery)应用程序，一直都是软件升级换代不得不面对的一个难题和痛点，也是一种追求的理想方式，也是DevOps诞生的目的。

### 2、滚动发布

把一次完整的发布过程，合理地分成多个批次，每次发布一个批次，**成功后**，再发布下一个批次，最终完成所有批次的发布。在整个滚动过程期间，保证始终有可用的副本在运行，从而平滑的发布新版本，实现**零停机(without an outage)**、用户**零感知**，是一种非常主流的发布方式。由于其自动化程度比较高，通常需要复杂的发布工具支撑，而k8s可以完美的胜任这个任务。

### 3、k8s滚动更新机制

**k8s 创建副本应用程序的最佳方法就是部署(Deployment)，部署自动创建副本集(ReplicaSet)，副本集可以精确地控制每次替换的 Pod 数量，从而可以很好的实现滚动更新**。具体来说，k8s 每次使用一个新的副本控制器(replication controller)来替换已存在的副本控制器，从而始终使用一个新的Pod模板来替换旧的pod模板。

> 大致步骤如下：
>
> 1. 创建一个新的replication controller。
> 2. 增加或减少pod副本数量，直到满足当前批次期望的数量。
> 3. 删除旧的replication controller。

### 4、演示

> 使用kubectl更新一个已部署的应用程序，并模拟回滚。为了方便分析，将应用程序的 pod 副本数量设置为10。

```bash
$ kubectl run busy --image=busybox:1.28.4 sleep 36000000 --replicas=10
```

#### 4.1. 发布微服务

- 当前服务状态查看

```bash
# 查看部署列表
root@kube-aio:~$ kubectl get deploy busy
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
busy      10        10        10           10          5m

# 查看正在运行的pod
root@kube-aio:~$ kubectl get pod | grep busy
busy-794c95f5d7-56b6w        1/1       Running   0          5m
busy-794c95f5d7-8ddjr        1/1       Running   0          5m
busy-794c95f5d7-8zm8r        1/1       Running   0          5m
busy-794c95f5d7-9hjhp        1/1       Running   0          5m
busy-794c95f5d7-df2r2        1/1       Running   0          5m
busy-794c95f5d7-fsn94        1/1       Running   0          5m
busy-794c95f5d7-k4w8r        1/1       Running   0          5m
busy-794c95f5d7-lsmgb        1/1       Running   0          5m
busy-794c95f5d7-rg8kw        1/1       Running   0          5m
busy-794c95f5d7-xpxxt        1/1       Running   0          5m

# 通过pod描述，查看应用程序的当前映像版本
root@kube-aio:~$ kubectl describe pod busy-794c95f5d7-56b6w |grep Image
    Image:         busybox:1.28.4
    Image ID:      docker-pullable://busybox@sha256:141c253bc4c3fd0a201d32dc1f493bcf3fff003b6df416dea4f41046e0f37d47
```

- 升级镜像版本到1.29
  - 为了更清晰看到更新过程，可另开一个窗口使用`$ watch kubectl get deployment busy`实时查看变化

```bash
$ kubectl set image deployments/busy busy=busybox:1.29
```

#### 4.2. 验证发布

```bash
# 检查rollout状态
root@kube-aio:~$ kubectl rollout status deployments/busy
deployment "busy" successfully rolled out

# 检查pod详情
root@kube-aio:~$ kubectl describe pod busy-665cdb7b-44jnt |grep Image
    Image:         busybox:1.29
    Image ID:      docker-pullable://busybox@sha256:cb63aa0641a885f54de20f61d152187419e8f6b159ed11a251a09d115fdff9bd
```

从上面可以看到，镜像已经升级到1.29版本

### 4.3. 回滚发布

```bash
# 回滚发布
root@kube-aio:~$ kubectl rollout undo deployments/busy
deployment.apps "busy" 

# 回滚完成
root@kube-aio:~$ kubectl rollout status deployments/busy
deployment "busy" successfully rolled out

# 镜像又回退到1.28.4 版本
root@kube-aio:~$ kubectl describe pod busy-794c95f5d7-4x9bn |grep Image
    Image:         busybox:1.28.4
    Image ID:      docker-pullable://busybox@sha256:141c253bc4c3fd0a201d32dc1f493bcf3fff003b6df416dea4f41046e0f37d47
```

到目前为止，整个滚动发布工作就圆满完成了！！！ **那么如果我们想回滚到指定版本呢？答案是k8s完美支持，并且还可以通过资源文件进行配置保留的历史版次量**。由于篇幅有限，感兴趣的朋友，可以自己下去实战，回滚命令如下：

```bash
kubectl rollout undo deployment/busy --to-revision=<版次>
```



### 5、原理

k8s 精确地控制着整个发布过程，分批次有序地进行着滚动更新，直到把所有旧的副本全部更新到新版本。实际上，k8s是通过两个参数来精确地控制着每次滚动的pod数量：

> - **`maxSurge` 滚动更新过程中运行操作期望副本数的最大pod数，可以为绝对数值(eg：5)，但不能为0；也可以为百分数(eg：10%)。**
> - **`maxUnavailable` 滚动更新过程中不可用的最大pod数，可以为绝对数值(eg：5)，但不能为0；也可以为百分数(eg：10%)。**

如果未指定这两个可选参数，则k8s会使用默认配置：

```bash
root@kube-aio:~$ kubectl get deploy busy -o yaml
apiVersion: apps/v1 
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "3"
  creationTimestamp: 2018-08-19T02:42:56Z
  generation: 3
  labels:
    run: busy
  name: busy
  namespace: default
  resourceVersion: "199461"
  uid: 93fde307-a359-11e8-a93b-525400c61543
spec:
  progressDeadlineSeconds: 600
  replicas: 10
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      run: busy
  strategy:
    rollingUpdate:
      maxSurge: 1	# 滚动更新中最多超过预期值的 pod数
      maxUnavailable: 1	# 滚动更新中最多不可用的 pod数
    type: RollingUpdate
...
```



### 5.1. 浅析部署概况

```bash
# 初始状态
root@kube-aio:~$ kubectl get deploy busy
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
busy      10        10        10           10          1h

# 再做一遍回退
root@kube-aio:~$ kubectl rollout undo deploy busy
deployment.apps "busy" 

# 更新过程1
root@kube-aio:~$ kubectl get deploy busy
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
busy      10        11        2            9           1h

# 更新过程2
root@kube-aio:~$ kubectl get deploy busy
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
busy      10        11        4            9           1h

# 更新过程3
root@kube-aio:~$ kubectl get deploy busy
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
busy      10        11        6            9           1h

# 更新结束
root@kube-aio:~$ kubectl get deploy busy
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
busy      10        10        10           10          1h
```

> - `DESIRED`  最终期望处于READY状态的副本数  
> - `CURRENT` 当前的副本总数
> - `UP-TO-DATE` 当前完成更新的副本数
> - `AVAILABLE` 当前可用的副本数

当前的副本总数：10(DESIRED) + 1(maxSurge) = 11，所以CURRENT为11。 当前可用的副本数：10(DESIRED) - 1(maxUnavailable) = 9，所以AVAILABLE为9。



### 5.2. 浅析部署详情

```bash
root@kube-aio:~$ kubectl describe deploy busy
Name:                   busy
Namespace:              default
CreationTimestamp:      Sun, 19 Aug 2018 12:27:19 +0800
Labels:                 run=busy
Annotations:            deployment.kubernetes.io/revision=2
Selector:               run=busy
Replicas:               10 desired | 10 updated | 10 total | 10 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  1 max unavailable, 1 max surge
Pod Template:
  Labels:  run=busy
  Containers:
   busy:
    Image:      busybox:1.29
    Port:       <none>
    Host Port:  <none>
    Args:
      sleep
      3600000
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   busy-84cb46955d (10/10 replicas created)
Events:
  Type    Reason             Age                 From                   Message
  ----    ------             ----                ----                   -------
  Normal  ScalingReplicaSet  1m                  deployment-controller  Scaled up replica set busy-9669c8599 to 10
  Normal  ScalingReplicaSet  46s                 deployment-controller  Scaled up replica set busy-84cb46955d to 1
  Normal  ScalingReplicaSet  46s                 deployment-controller  Scaled down replica set busy-9669c8599 to 9
  Normal  ScalingReplicaSet  46s                 deployment-controller  Scaled up replica set busy-84cb46955d to 2
  Normal  ScalingReplicaSet  43s                 deployment-controller  Scaled down replica set busy-9669c8599 to 8
  Normal  ScalingReplicaSet  43s                 deployment-controller  Scaled up replica set busy-84cb46955d to 3
  Normal  ScalingReplicaSet  43s                 deployment-controller  Scaled down replica set busy-9669c8599 to 7
  Normal  ScalingReplicaSet  43s                 deployment-controller  Scaled up replica set busy-84cb46955d to 4
  Normal  ScalingReplicaSet  40s                 deployment-controller  Scaled down replica set busy-9669c8599 to 6
  Normal  ScalingReplicaSet  28s (x12 over 40s)  deployment-controller  (combined from similar events): Scaled down replica set busy-9669c8599 to 0
```

整个滚动过程是通过控制两个副本集来完成的，新的副本集：busy-84cb46955d；旧的副本集：busy-9669c8599 。 理想状态下的滚动过程：

> 1. 创建新副本集，并为其分配1个新版本的pod。
> 2. 通知旧副本集，销毁1个旧版本的pod。
> 3. 当旧副本销毁成功后，通知新副本集，再新增1个新版本的pod；当新副本创建成功后，通知旧副本再减少1个pod。 只要销毁成功，新副本集就会创造新的pod，一直循环，直到旧的副本集pod数量为0。



### 5.4 总结

**`无论理想还是不理想，k8s最终都会使应用程序全部更新到期望状态，都会始终保持最大的副本总数和可用副本总数的不变性！！！`**





## HPA

### Horizontal Pod Autoscaling

自动水平伸缩，是指运行在k8s上的应用负载(POD)，可以根据资源使用率进行自动扩容、缩容；我们知道应用的资源使用率通常都有高峰和低谷，所以k8s的`HPA`特性应运而生；它也是最能体现区别于传统运维的优势之一，不仅能够弹性伸缩，而且完全自动化！

根据 CPU 使用率或自定义 metrics 自动扩展 Pod 数量（支持 replication controller、deployment）；k8s1.6版本之前是通过kubelet来获取监控指标，1.6版本之后是通过api server、heapster或者kube-aggregator来获取监控指标。

### Metrics支持

根据不同版本的API中，HPA autoscale时靠以下指标来判断资源使用率：

- autoscaling/v1: CPU
- autoscaling/v2alpha1
  - 内存
  - 自定义metrics
  - 多metrics组合: 根据每个metric的值计算出scale的值，并将最大的那个值作为扩容的最终结果

### 基础示例

本实验环境基于k8s 1.8 和 1.9，仅使用`autoscaling/v1` 版本API，**注意确保**`k8s` 集群插件`kubedns` 和 `heapster` 工作正常。

```bash
# 创建deploy和service
$ kubectl run php-apache --image=pilchard/hpa-example --requests=cpu=200m --expose --port=80

# 创建autoscaler
$ kubectl autoscale deploy php-apache --cpu-percent=50 --min=1 --max=10

# 等待3~5分钟查看hpa状态
$ kubectl get hpa php-apache
NAME         REFERENCE               TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   0% / 50%   1         10        1          3m

# 增加负载
$ kubectl run --rm -it load-generator --image=busybox /bin/sh
Hit enter for command prompt
$ while true; do wget -q -O- http://php-apache; done;

# 等待约5分钟查看hpa显示负载增加，且副本数目增加为4
$ kubectl get hpa php-apache
NAME         REFERENCE               TARGETS      MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   430% / 50%   1         10        4          4m

# 注意k8s为了避免频繁增删pod，对副本的增加速度有限制
# 实验过程可以看到副本数目从1到4到8到10，大概都需要4~5分钟的缓冲期
$ kubectl get hpa php-apache
NAME         REFERENCE               TARGETS     MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   86% / 50%   1         10        8          9m
$ kubectl get hpa php-apache
NAME         REFERENCE               TARGETS     MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   52% / 50%   1         10        10         12m

# 清除负载，CTRL+C 结束上述循环程序，稍后副本数目变回1
$ kubectl get hpa php-apache
NAME         REFERENCE               TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   0% / 50%   1         10        1          17m
```





# 周边生态



## Harbor

### harbor 镜像仓库

Habor是由VMWare中国团队开源的容器镜像仓库。事实上，Habor是在Docker Registry上进行了相应的企业级扩展，从而获得了更加广泛的应用，这些新的企业级特性包括：管理用户界面，基于角色的访问控制 ，水平扩展，同步，AD/LDAP集成以及审计日志等。本文档仅说明部署单个基础harbor服务的步骤。

- 目录
  - 安装步骤
  - 安装讲解
  - 配置docker/containerd信任harbor证书
  - 在k8s集群使用harbor
  - 管理维护

#### **安装步骤**

1. 在ansible控制端下载最新的 [docker-compose](https://github.com/docker/compose/releases) 二进制文件，改名后把它放到项目 `/etc/ansible/bin`目录（已包含）
2. 在ansible控制端下载最新的 [harbor](https://github.com/vmware/harbor/releases) 离线安装包，把它放到项目 `/etc/ansible/down` 目录
3. 在ansible控制端编辑/etc/ansible/hosts文件，可以参考 `example`目录下的模板，修改部分举例如下

```bash
# 参数 NEW_INSTALL=(yes/no)：yes表示新建 harbor，并配置k8s节点的docker可以使用harbor仓库
# no 表示仅配置k8s节点的docker使用已有的harbor仓库
# 参数 SELF_SIGNED_CERT=(yes/no): yes表示使用自签名证书，即安装程序帮你做一个自己签名的证书（当然这样的证书是得不到浏览器直接认可的）
# no 表示使用已有的证书，如 letsencrypt 或者其他证书颁发机构，如使用此参数，需把证书提前放在 down 目录下，文件名称分别为：harbor.pem 和 harbor-key.pem
# 如果不需要设置域名访问 harbor，可以配置参数 HARBOR_DOMAIN=""
[harbor]
192.168.1.8 HARBOR_DOMAIN="harbor.yourdomain.com" NEW_INSTALL=yes SELF_SIGNED_CERT=yes
```

1. 在ansible控制端执行 `ansible-playbook /etc/ansible/11.harbor.yml`，完成harbor安装和docker 客户端配置

- 安装验证

1. 在harbor节点使用`docker ps -a` 查看harbor容器组件运行情况
2. 浏览器访问harbor节点的IP地址 `https://$NodeIP`，管理员账号是 admin ，密码见 harbor.cfg(v1.5-v1.7) 或 harbor.yml(v1.8+) 文件 harbor_admin_password 对应值（默认密码 Harbor12345 已被随机生成的16位随机密码替换，不然存在安全隐患)

#### **安装讲解**

根据`11.harbor.yml`文件，harbor节点需要以下步骤：

- role `prepare` 基础系统环境准备
- role `docker` 安装docker
- role `harbor` 安装harbor
- 注意：`kube-node`节点在harbor部署完之后，需要配置harbor的证书（详见下节配置docker/containerd信任harbor证书），并可以在hosts里面添加harbor的域名解析，如果你的环境中有dns服务器，可以跳过hosts文件设置

请在另外窗口打开 [roles/harbor/tasks/main.yml](https://github.com/easzlab/kubeasz/blob/master/roles/harbor/tasks/main.yml)，对照以下讲解

1. 下载docker-compose可执行文件到$PATH目录
2. 自注册变量result判断是否已经安装harbor，避免重复安装问题
3. 解压harbor离线安装包到指定目录
4. 导入harbor所需 docker images
5. 创建harbor证书和私钥(复用集群的CA证书)
6. 修改harbor.cfg配置文件
7. 启动harbor安装脚本



#### 配置docker/containerd信任harbor证书

因为我们创建的harbor仓库使用了自签证书，所以当docker/containerd客户端拉取自建harbor仓库镜像前必须配置信任harbor证书，否则出现如下错误：

```bash
# docker
$ docker pull harbor.test.lo/pub/hello:v0.1.4
Error response from daemon: Get https://harbor.test.lo/v1/_ping: x509: certificate signed by unknown authority

# containerd
$ crictl pull harbor.test.lo/pub/hello:v0.1.4
FATA[0000] pulling image failed: rpc error: code = Unknown desc = failed to resolve image "harbor.test.lo/pub/hello:v0.1.4": no available registry endpoint: failed to do request: Head https://harbor.test.lo/v2/pub/hello/manifests/v0.1.4: x509: certificate signed by unknown authority
```

项目脚本`11.harbor.yml`中已经自动为k8s集群的每个node节点配置 docker/containerd 信任自建 harbor 证书；如果你无法运行此脚本，可以参考下述手工配置（使用受信任的正式证书 SELF_SIGNED_CERT=no 可忽略）



#### docker配置信任harbor证书

在集群每个 node 节点进行如下配置

- 创建目录 /etc/docker/certs.d/harbor.test.lo/ (harbor.test.lo为你的harbor域名)
- 复制 harbor 安装时的 CA 证书到上述目录，并改名 ca.crt 即可

**containerd配置信任harbor证书**

在集群每个 node 节点进行如下配置（假设ca.pem为自建harbor的CA证书）

- ubuntu 1604:
  - cp ca.pem /usr/share/ca-certificates/harbor-ca.crt
  - echo harbor-ca.crt >> /etc/ca-certificates.conf
  - update-ca-certificates
- CentOS 7:
  - cp ca.pem /etc/pki/ca-trust/source/anchors/harbor-ca.crt
  - update-ca-trust

上述配置完成后，重启 containerd 即可 `systemctl restart containerd`



### 在k8s集群使用harbor

admin用户web登录后可以方便的创建项目，并指定项目属性(公开或者私有)；然后创建用户，并在项目`成员`选项中选择用户和权限；



#### 镜像上传

在node上使用harbor私有镜像仓库首先需要在指定目录配置harbor的CA证书，详见 `11.harbor.yml`文件。

使用docker客户端登录`harbor.test.com`，然后把镜像tag成 `harbor.test.com/$项目名/$镜像名:$TAG` 之后，即可使用docker push 上传

```bash
docker login harbor.test.com
Username: 
Password:
Login Succeeded
docker tag busybox:latest harbor.test.com/library/busybox:latest
docker push harbor.test.com/library/busybox:latest
The push refers to a repository [harbor.test.com/library/busybox]
0271b8eebde3: Pushed 
latest: digest: sha256:91ef6c1c52b166be02645b8efee30d1ee65362024f7da41c404681561734c465 size: 527
```



#### k8s中使用harbor

1. 如果镜像保存在harbor中的公开项目中，那么只需要在yaml文件中简单指定harbor私有镜像即可，例如

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-busybox
spec:
  containers:
  - name: test-busybox
    image: harbor.test.com/xxx/busybox:latest
    imagePullPolicy: Always
```

1. 如果镜像保存在harbor中的私有项目中，那么yaml文件中使用该私有项目的镜像需要指定`imagePullSecrets`，例如

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-busybox
spec:
  containers:
  - name: test-busybox
    image: harbor.test.com/xxx/busybox:latest
    imagePullPolicy: Always
  imagePullSecrets:
  - name: harborkey1
```

其中 `harborKey1`可以用以下两种方式生成：

- 1.使用 `kubectl create secret docker-registry harborkey1 --docker-server=harbor.test.com --docker-username=admin --docker-password=Harbor12345 --docker-email=team@test.com`
- 2.使用yaml配置文件生成

```yaml
//harborkey1.yaml
apiVersion: v1
kind: Secret
metadata:
  name: harborkey1
  namespace: default
data:
    .dockerconfigjson: {base64 -w 0 ~/.docker/config.json}
type: kubernetes.io/dockerconfigjson
```

前面docker login会在~/.docker下面创建一个config.json文件保存鉴权串，这里secret yaml的.dockerconfigjson后面的数据就是那个json文件的base64编码输出（-w 0让base64输出在单行上，避免折行



### 管理维护

- 日志目录 `/var/log/harbor`
- 数据目录 `/data` ，其中最主要是 `/data/database` 和 `/data/registry` 目录，如果你要彻底重新安装harbor，删除这两个目录即可

先进入harbor安装目录 `cd /data/harbor`，常规操作如下：

1. 暂停harbor `docker-compose stop` : docker容器stop，并不删除容器
2. 恢复harbor `docker-compose start` : 恢复docker容器运行
3. 停止harbor `docker-compose down -v` : 停止并删除docker容器
4. 启动harbor `docker-compose up -d` : 启动所有docker容器

修改harbor的运行配置，需要如下步骤：

```bash
# 停止 harbor
 docker-compose down -v
# 修改配置
 vim harbor.cfg
# 执行./prepare已更新配置到docker-compose.yml文件
 ./prepare
# 启动 harbor
 docker-compose up -d
```

#### **harbor 升级**

以下步骤基于harbor 1.1.2 版本升级到 1.2.2版本

```bash
# 进入harbor解压缩后的目录，停止harbor
cd /data/harbor
docker-compose down

# 备份这个目录
cd ..
mkdir -p /backup && mv harbor /backup/harbor

# 下载更新的离线安装包，并解压
tar xvf harbor-offline-installer-v1.2.2.tgz  -C /data

# 使用官方数据库迁移工具，备份数据库，修改数据库连接用户和密码，创建数据库备份目录
# 迁移工具使用docker镜像，镜像tag由待升级到目标harbor版本决定，这里由 1.1.2升级到1.2.2，所以使用 tag 1.2
docker pull vmware/harbor-db-migrator:1.2
mkdir -p /backup/db-1.1.2
docker run -it --rm -e DB_USR=root -e DB_PWD=xxxx -v /data/database:/var/lib/mysql -v /backup/db-1.1.2:/harbor-migration/backup vmware/harbor-db-migrator:1.2 backup

# 因为新老版本数据库结构不一样，需要数据库migration
docker run -it --rm -e DB_USR=root -e DB_PWD=xxxx -v /data/database:/var/lib/mysql vmware/harbor-db-migrator:1.2 up head

# 修改新版本 harbor.cfg(v1.5-v1.7) 或 harbor.yml(v1.8+) 配置，需要保持与老版本相关配置项保持一致，然后执行安装即可
cd /data/harbor
vi harbor.cfg
./install.sh
```

## Helm

`Helm`致力于成为k8s集群的应用包管理工具，希望像linux 系统的`RPM` `DPKG`那样成功；确实在k8s上部署复杂一点的应用很麻烦，需要管理很多yaml文件（configmap,controller,service,rbac,pv,pvc等等），而helm能够整齐管理这些文档：版本控制，参数化安装，方便的打包与分享等。

- 建议积累一定k8s经验以后再去使用helm；对于初学者来说手工去配置那些yaml文件对于快速学习k8s的设计理念和运行原理非常有帮助，而不是直接去使用helm，面对又一层封装与复杂度。
- 本文基于helm 3（建议版本），helm 2 文档[请看这里](https://github.com/easzlab/kubeasz/blob/master/docs/guide/helm2.md)

### 安装 helm

在官方repo下载[release版本](https://github.com/helm/helm/releases)中自带的二进制文件即可（以Linux amd64为例）

```bash
wget https://get.helm.sh/helm-v3.2.1-linux-amd64.tar.gz
mv ./linux-amd64/helm /usr/bin
```

- 启用官方 charts 仓库

```bash
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
```

### 使用 helm 安装应用

helm3 安装命令与 helm2 稍有变化，个人习惯先下载对应charts到本地然后按照固定目录格式安装，以创建一个redis集群举例：

- 创建 redis-cluster 目录

```bash
mkdir -p /opt/charts/redis-cluster
cd /opt/charts/redis-cluster
```

- 下载最新stalbe/redis-ha

```bash
helm repo update
helm pull stable/redis-ha
```

- 解压 charts，复制 values.yaml设置

```bash
tar zxvf redis-ha-*.tgz
cp redis-ha/values.yaml .
```

- 创建 start.sh 脚本记录启动命令

```bash
cat > start.sh << EOF
#!/bin/sh
set -x

ROOT=$(cd `dirname $0`; pwd)
cd $ROOT

helm install redis \
	--create-namespace \
	--namespace dependency \
	-f ./values.yaml \
	./redis-ha
EOF
```

- 查看当前目录结构如下

```bash
tree .
.
├── redis-ha		# redis-ha 原始charts目录
├── start.sh		# 启动命名脚本
└── values.yaml		# 个性化参数配置
```

- 修改当前目录的 values.yaml 为你的个性化配置

```bash
#举例values.yaml 配置如下，没有启用PV
#cat values.yaml
image:
  repository: redis
  tag: 5.0.6-alpine

replicas: 2

## Redis specific configuration options
redis:
  port: 6379
  masterGroupName: "mymaster"       # must match ^[\\w-\\.]+$) and can be templated
  config:
    ## For all available options see http://download.redis.io/redis-stable/redis.conf
    min-replicas-to-write: 1
    min-replicas-max-lag: 5   # Value in seconds
    maxmemory: "4g"       # Max memory to use for each redis instance. Default is unlimited.
    maxmemory-policy: "allkeys-lru"  # Max memory policy to use for each redis instance. Default is volatile-lru.
    repl-diskless-sync: "yes"
    rdbcompression: "yes"
    rdbchecksum: "yes"

  resources:
    requests:
      memory: 200Mi
      cpu: 100m
    limits:
      memory: 4000Mi

## Sentinel specific configuration options
sentinel:
  port: 26379
  quorum: 1

  resources:
    requests:
      memory: 200Mi
      cpu: 100m
    limits:
      memory: 200Mi

hardAntiAffinity: true

## Configures redis with AUTH (requirepass & masterauth conf params)
auth: false

persistentVolume:
  enabled: false

hostPath:
  path: "/data/mcs-redis/{{ .Release.Name }}"
```

- 执行安装

```
bash ./start.sh
```

- 查看安装

```bash
helm ls -A
NAME 	NAMESPACE 	REVISION	UPDATED                                	STATUS  	CHART         	APP VERSION
redis	dependency	1       	2020-05-28 20:57:31.166002853 +0800 CST	deployed	redis-ha-4.4.4	5.0.6

# 查看k8s上资源
kubectl get pod,svc -n dependency
NAME                          READY   STATUS    RESTARTS   AGE
pod/redis-redis-ha-server-0   2/2     Running   0          119s
pod/redis-redis-ha-server-1   2/2     Running   0          104s

NAME                                TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)              AGE
service/redis-redis-ha              ClusterIP   None          <none>        6379/TCP,26379/TCP   119s
service/redis-redis-ha-announce-0   ClusterIP   10.68.41.65   <none>        6379/TCP,26379/TCP   119s
service/redis-redis-ha-announce-1   ClusterIP   10.68.64.49   <none>        6379/TCP,26379/TCP   119s
```

## jenkins

本文档介绍如何快速通过K8s集群实现Jenkins 动态Slave CI/CD流程。

### 开始之前

在开始之前需要准备以下环境：

- k8s dns组件
  参考文档：[kubedns](https://github.com/easzlab/kubeasz/blob/master/docs/guide/kubedns.md)
- helm
  为了简化部署，通过helm来安装Jenkins，可参考文档：[helm](https://github.com/easzlab/kubeasz/blob/master/docs/guide/helm.md)
- 持久化存储
  这里使用**NFS**演示，参考文档：[cluster-storage](https://github.com/easzlab/kubeasz/blob/master/docs/setup/08-cluster-storage.md)。 如果k8s集群是部署在公有云，也可使用厂商的NAS等存储方案，项目中已集成支持阿里云NAS，其他的方案参考相关厂商文档
- Ingress Controller(nginx-ingress/traefik)
  默认是通过Ingress访问Jenkins，因此需要安装一种`Ingress Controller`。参考文档：[ingress](https://github.com/easzlab/kubeasz/blob/master/docs/guide/ingress.md)
- Gitlab 代码管理仓库
  用于提交代码后自动触发CI, 目前项目中还没有相关内容，可[参考官网](https://about.gitlab.com/installation/)进行安装。

### 安装Jenkins

执行以下命令快速安装：

```bash
helm install manifests/jenkins/ --name jenkins
```

如果通过/etc/ansible/roles/helm/helm.yml安装的helm，安装过程会出现如下错误

```bash
E0703 08:40:22.376225   19888 portforward.go:331] an error occurred forwarding 41655 -> 44134: error forwarding port 44134 to pod 5098414beaaa07140a4ba3240690b1ce989ece01e5db33db65eec83bd64bdedf, uid : exit status 1: 2018/07/03 08:40:22 socat[19991] E write(5, 0x1aec120, 3424): Connection reset by peer
Error: transport is closing
```

请执行以下命令快速安装进行修复：

```bash
helm install --tls manifests/jenkins/ --name jenkins
```

由于初始化过程中，默认安装指定的插件，所以启动较慢，大概5-10分钟左右就可以启动完成了。

部分默认配置说明： **注**：以下配置都定义在`manifests/jenkins/values.yaml`文件中。

| **字段**       | **说明**         | **默认值**                                                   |
| -------------- | ---------------- | ------------------------------------------------------------ |
| InstallPlugins | 初始化安装的插件 | kubernetes:1.6.3workflow-aggregator:2.5workflow-job:2.21credentials-binding:1.16git:3.9.0gitlab:1.5.6 |
| HostName       | Ingress访问入口  | jenkins.local.com                                            |
| AdminPassword  | admin登录密码    | admin                                                        |
| UpdateCenter   | 插件下载镜像地址 | https://mirrors.tuna.tsinghua.edu.cn/jenkins                 |
| StorageClass   | 持久化存储SC     | nfs-dynamic-class                                            |

### 配置Kubernetes plugin

登录Jenkins，点击左边导航`系统管理`——>`系统设置`，拖动到最下面可以看到`云——>Kubernetes`配置，默认配置有以下字段：

- Name：配置名称，后面运行测试的时候会用到，用于区别多个Kubernetes配置，默认为：kubernetes
- Kubernetes URL：集群访问url，可通过`kubectl cluster-info`查看，如果集群有部署**DNS**插件, 也可以直接填服务名称(自动解析)，默认使用服务名称：[https://kubernetes](https://kubernetes/)
- Jenkins URL：Jenkins访问地址，默认使用服务名称+端口号

在Jenkins初始化时，默认都已经配置好了，可以直接新建项目测试了。

### 简单测试

点击左边：新建任务——>流水线(Pipeline) 任务名称可以随便起，这里为：k8s-test 配置——>流水线，选择`Pipeline script` 以下为测试脚本内容：

```bash
podTemplate(label: 'jenkins-slave', cloud: 'kubernetes')
{
    node ('jenkins-slave') {
        stage('test') {
            echo "hello, world"
            sleep 60
        }
    }
}
```

- cloud：插件配置中的Name
- label：插件配置中的Images——>Kubernetes Pod Tempalte——>Labels
- node：与label一致即可

保存配置，点击立即构建，查看控制台输出，出现以下内容就表示运行成功了：

```bash
Agent default-lsths is provisioned from template Kubernetes Pod Template
Agent specification [Kubernetes Pod Template] (jenkins-slave): 
* [jnlp] jenkins/jnlp-slave:alpine(resourceRequestCpu: 200m, resourceRequestMemory: 256Mi, resourceLimitCpu: 200m, resourceLimitMemory: 256Mi)

Running on default-lsths in /home/jenkins/workspace/k8s-test
[Pipeline] {
[Pipeline] stage
[Pipeline] { (test)
[Pipeline] echo
hello, world
[Pipeline] sleep
Sleeping for 1 min 0 sec
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // node
[Pipeline] }
[Pipeline] // podTemplate
[Pipeline] End of Pipeline
Finished: SUCCESS
```

### 配置自动触发CI

- 配置Gitlab项目
  在`Gitlab`中创建一个测试项目，将上面测试的脚本内容写入到一个`Jenkinsfile`文件中，然后上传到该测试项目根路径下。
- 配置Jenkins项目
  点击项目`配置`——>`构建触发器`——>勾选`Build when a change is pushed to GitLab. GitLab webhook URL:http://jenkins.local.com/project/k8s-test`——>保存配置
- 配置Webhook
  进入Gitlab测试项目的`Settings——>Integrations`，一般只需要填写`URL`即可，其他的可根据需求环境配置 默认Jenkins配置不允许匿名用户触发构建，因此还需要添加用户和token。
  URL的格式为：
  `http://[UserID]:[API Token]@jenkins.local.com/project/[ProjectName]`

Jenkins 用户ID Token查看： 点击右上角的`用户名——>设置——>API Token(点击Show API Token...)`

最终Webhook中的URL类似： http://admin:a910b1492e39e9dd1ea48ea7f7638aaf@jenkins.local.com/project/k8s-test

后面只需要我们一提交代码到Git仓库，就会自动触发Jenkins进行构建了。

### 项目应用

这里我们以一个简单的Java项目为例，实战演示如何进行CI/CD。 基本环境配置上面已经说过了，这里就不多介绍。
示例项目：https://github.com/lusyoe/springboot-k8s-example

结构说明：

- 镜像构建文件：`Dockerfile`
- k8s应用配置：`k8s-example.yaml`
- 项目源码：`src`
- Jenkins构建文件：`jenkins/Jenkinsfile`

构建流程说明：

- 通过Jenkins kubernetes插件，定义构建过程中所需的3个docker容器：maven、docker、kubectl (这3个容器都在一个pod中)
- 挂载docker.sock和kubeconfig文件
- 首先使用`maven`容器，检出代码，执行项目构建
- 使用`docker`容器，构建镜像，推送到镜像参考
- 使用`kubectl`容器，部署`k8s-example`应用(这里后面也可以使用helm)

访问：
项目通过Ingress访问`k8s-example.com`，出现`hello, world`,就表示服务部署成功了。

## Gitlab

### 安装 gitlab

gitlab 是深受企业用户喜爱的基于 git 的代码管理系统。安装 gitlab 最理想的方式是利用 gitlab charts 部署到 k8s 集群上，但此方式还未成熟，期待后续推出更成熟稳定版本；本文使用 Docker 方式安装 gitlab:

- 环境：Ubuntu 16.04，虚机内存/CPU/存储请根据实际使用情况配置，一般`4C/8G/200G`足够
- 安装 docker: 18.06.1-ce

#### 准备启动脚本

```bash
$ cat > gitlab-setup.sh << EOF
#!/bin/bash
# 注意：设置 gitlab_shell_ssh_port 是为了后续可以使用 SSH 方式访问你的项目
docker run --detach \\
    --hostname gitlab.test.com \\
    --env GITLAB_OMNIBUS_CONFIG="external_url 'http://gitlab.test.com/'; gitlab_rails['gitlab_shell_ssh_port'] = 6022;" \\
    --publish 443:443 --publish 80:80 --publish 6022:22 \\
    --name gitlab \\
    --restart always \\
    --volume /srv/gitlab/config:/etc/gitlab \\
    --volume /srv/gitlab/logs:/var/log/gitlab \\
    --volume /srv/gitlab/data:/var/opt/gitlab \\
    docker.mirrors.ustc.edu.cn/gitlab/gitlab-ce:11.2.2-ce.0
EOF
```

执行启动脚本：`sh gitlab-setup.sh` 执行成功后，等待数分钟可以看到

```bash
$ docker ps -a
CONTAINER ID        IMAGE                                                 COMMAND             CREATED             STATUS                   PORTS                                                            NAMES
4f9d5f97f494        docker.mirrors.ustc.edu.cn/gitlab/gitlab-ce:11.2.2-ce.0   "/assets/wrapper"   9 minutes ago       Up 9 minutes (healthy)   0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp, 0.0.0.0:6022->22/tcp   gitlab
```

#### 配置 gitlab

```bash
$ docker exec -it gitlab vi /etc/gitlab/gitlab.rb
```

请阅读后修改（因为前面docker run 已经指定了必要参数，可以不修改，后续有需要再修改），修改保存以后需要重启容器

```bash
$ docker restart gitlab
```

#### 首次访问 gitlab

使用域名`gitlab.test.com`或者该主机 IP 首次登录时会要求设置 root 用户的密码，完成后就可以用 root 和新设密码登录；然后按需创建 Group, User, Projects等，还有相关配置。

#### 备份数据

无论是企业、组织、个人都十分重视代码资产，之前我们的 gitlab 安装是单机版的，虽然可以有硬盘 raid 等保护，还有是丢失 gitlab 数据和配置的风险，因此我们有必要再做一些备份操作。这里利用 crontab 定期执行 rsync 命令备份到其他服务器。

```bash
# 创建备份脚本
cat > /root/gitlab-backup.sh << EOF
#!/bin/bash
# 请事先配置 gitlab 服务器到备份服务器的免密码 ssh 登录
rsync -av --delete /srv/gitlab/config '-e ssh -l root' 192.168.1.xx:/backup_gitlab/config
rsync -av --delete /srv/gitlab/data '-e ssh -l root' 192.168.1.xx:/backup_gitlab/data
EOF

# 创建并应用 crontab
cat > /etc/cron.d/gitlab-backup << EOF
## 每3个小时同步备份一次，具体根据需要修改
11 */3 * * * root bash /root/gitlab-backup.sh > /root/gitlab/sync.log 2>&1
EOF
```

如果 gitlab 服务器真的出现不可恢复的故障，丢失数据，那么至少保留有3小时前的备份，利用备份的文件，同样再用 docker 挂载 volume的方式运行，这样就可以恢复原 gitlab 服务运行。

#### 升级 gitlab

因为前面使用了 docker 方式安装，因此 gitlab 升级很方便。

- 升级前停止/删除容器：`$ docker stop gitlab && docker rm gitlab`
- 如上节执行备份数据
- 修改 gitlab-setup.sh 指定新的版本，执行该脚本

#### 参考

- 1.[Install GitLab with Docker](https://docs.gitlab.com/omnibus/docker/)

### Gitlab CI/CD 基础

Gitlab-ci 兼容 travis ci 格式，也是最流行的 CI 工具之一；本文讲解利用 gitlab, gitlab-runner, docker, harbor, kubernetes 等流行开源工具搭建一个自动化CI/CD流水线；示例配置以简单实用为原则，暂时没有选用 dind（docker in dockers）打包、gitlab Auto DevOps 等方式。一个最简单的流水线如下：

- 代码提交 --> 镜像构建 --> 部署测试 --> 部署生产

### 0.前提条件

- 正常运行的 gitlab，[安装 gitlab 文档](https://github.com/easzlab/kubeasz/blob/master/docs/guide/gitlab/gitlab-install.md)
- 正常运行的容器仓库，[安装 Harbor 文档](https://github.com/easzlab/kubeasz/blob/master/docs/guide/harbor.md)
- 正常运行的 k8s，可以本地自建 k8s 集群，也可以使用公有云 k8s 集群
- 若干虚机运行 gitlab-runner: 运行自动化流水线任务 pipeline job
- 了解代码管理流程 gitflow 等

### 1.准备测试项目代码

假设你要开发一个 spring boot 项目；先登录你的 gitlab 账号，创建项目，上传你的代码；项目根目录看起来如下：

```bash
-rw-r--r-- 1 root root    44 Jan  2 16:38 eclipse.bat
drwxr-xr-x 8 root root  4096 Jan  7 15:29 .git/
-rw-r--r-- 1 root root   276 Jan  7 08:44 .gitignore
drwxr-xr-x 3 root root  4096 Jan  7 08:44 example-api/
drwxr-xr-x 3 root root  4096 Jan  7 08:44 example-biz/
drwxr-xr-x 3 root root  4096 Jan  2 16:38 example-dal/
drwxr-xr-x 3 root root  4096 Jan  2 16:38 example-web/
-rw-r--r-- 1 root root    54 Jan  2 16:38 install.bat
-rw-r--r-- 1 root root 10419 Jan  2 16:38 pom.xml
```

传统做法是在本地配置好相关环境后使用 mvn 编译生成jar包，然后测试运行jar；这里我们要把应用打包成 docker 镜像，并创建 CI/CD 流水线：如下示例，在项目根目录新增创建2个文件夹及相关文件

```bash
dockerfiles        ### 新增文件夹用来 docker 镜像打包
└── Dockerfile     # 定义 docker 镜像
.ci                ### 新增文件夹用来存放 CI/CD 相关内容
├── app.yaml       # k8s 平台的应用部署文件
├── config.sh      # 配置替换脚本
└── gitlab-ci.yml  # gitlab-ci 的主配置文件
```

### 2.准备 docker 镜像描述文件 Dockerfile

我们把 Dockerfile 放在独立目录下，java spring boot 应用可以这样写：

```bash
cat > dockerfiles/Dockerfile << EOF
FROM openjdk:8-jdk-alpine
VOLUME /tmp
COPY *.jar app.jar         # 这里 *.jar 包就是后续在cicd pipeline 过程中 mvn 生成的jar包移动到此目录
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
EOF
```

### 3.准备 CI/CD 相关脚本和文件

装完 gitlab 后使用浏览器登录gitlab，很容易找到帮助文档，里面有介绍gitlab-ci的内容（文档权威、详细！请多多阅读~ 随着CI/CD流程的深入，部分内容也可以回来查阅），先看如下文档（假设你本地gitlab使用域名`gitlab.test.com`）

- 文档首页 http://gitlab.test.com/help
- gitlab-ci 基本概念 http://gitlab.test.com/help/ci/README.md
- variables 变量 http://gitlab.test.com/help/ci/variables/README.md

目录`.ci`下面的三个文件`app.yaml`, `config.sh`, `gitlab-ci.yml`是互相关联的；gitlab-ci.yml 文件中会调用到另外两个文件；文件之间又通过一些变量定义联系，流程中用到的变量大致可以分为三种：

- 第一种是gitlab自身预定义变量（比如项目名: CI_PROJECT_NAME，流水线ID: CI_PIPELINE_ID）；无需更改；
- 第二种是在gitlab-ci.yml文件中定义的变量，一般是少量的自定义变量；按需少量改动；
- 第三种是用户可以在项目web界面配置的变量：“Settings”>"CI/CD">"Variables"，本示例项目用到该类型变量举例：

| 变量            | 值           | 注解                       |
| --------------- | ------------ | -------------------------- |
| BETA_APP_REP    | 1            | beta环境应用副本数         |
| BETA_DB_HOST    | 1.1.1.1:3306 | beta环境应用连接数据库主机 |
| BETA_DB_PWD     | xxxx         | beta环境数据库连接密码     |
| BETA_DB_USR     | xxxx         | beta环境数据库连接用户     |
| BETA_REDIS_HOST | 1.1.1.2      | beta环境redis主机          |
| BETA_REDIS_PORT | 6379         | beta环境redis端口          |
| BETA_REDIS_PWD  | xxxx         | beta环境redis密码          |
| BETA_HARBOR     | 1.1.1.3      | beta环境镜像仓库地址       |
| BETA_HARBOR_PWD | xxxx         | beta环境镜像仓库密码       |
| BETA_HARBOR_USR | xxxx         | beta环境镜像仓库用户       |
| PROD_APP_REP    | 2            | prod环境应用副本数         |
| PROD_DB_HOST    | 2.2.2.1:3306 | prod环境应用连接数据库主机 |
| PROD_DB_PWD     | xxxx         | prod环境数据库连接密码     |
| PROD_DB_USR     | xxxx         | prod环境数据库连接用户     |
| PROD_REDIS_HOST | 2.2.2.2      | prod环境redis主机          |
| PROD_REDIS_PORT | 6379         | prod环境redis端口          |
| PROD_REDIS_PWD  | xxxx         | prod环境redis密码          |
| PROD_HARBOR     | 2.2.2.3      | prod环境镜像仓库地址       |
| PROD_HARBOR_PWD | xxxx         | prod环境镜像仓库密码       |
| PROD_HARBOR_USR | xxxx         | prod环境镜像仓库用户       |
| ...             | ...          | 根据项目需要自行添加设置   |

掌握了以上基础知识，可以开始以下三个任务：

- 3.1[配置 gitlab-ci.yml](https://github.com/easzlab/kubeasz/blob/master/docs/guide/gitlab/gitlab-ci.yml.md), 整个CI/CD的主配置文件，定义所有的CI/CD阶段和每个阶段的任务
- 3.2[配置 config.sh](https://github.com/easzlab/kubeasz/blob/master/docs/guide/gitlab/config.sh.md)，根据不同分支/环境替换不同的应用程序变量（对应上述第三种变量）
- 3.3[配置 app.yaml](https://github.com/easzlab/kubeasz/blob/master/docs/guide/gitlab/app.yaml.md)，K8S应用部署简单模板，替换完成后可以部署到测试/生产的K8S平台上

### 4.为项目配置 CI/CD 及创建 RUNNER

使用浏览器访问gitlab，登录后，在项目页面进行配置，如图：

![image-20200617171302511](D:\学习资料\笔记\linux\assets\image-20200617171302511.png)

- 在 General pipelines 中配置 Custom CI config path 为 .ci/gitlab-ci.yml
- 在 Variables 中配置需要用到的变量
- 在 Runners 中配置注册 gitlab-runner 实例（runner 就是用来自动执行ci job的），点进去后如图：

![image-20200617171324932](D:\学习资料\笔记\linux\assets\image-20200617171324932.png)

- 作为入门，先来手动创建 specific Runner，后续同样可以创建 Group Runners/Shared Runners，使用起来更方便；本文档暂不涉及在 kubernetes 自动创建 Runner
  - 按照官网文档安装 Gitlab Runner，参考
  - 记下 gitlab URL, 项目 token，注册 Runner 时要用到
  - 在 Gitlab Runner 注册本项目

### 6.gitlab-ci 安全实践

现在为止 CICD Pipelines 已经可以跑通了，甚至稍微修改下 gitlab-ci.yml 配置，项目代码每一次提交后可以自动执行`编译`、`打包`、`部署测试`、`部署生产`等等工作；也许你还没来得及慢慢体会这顺畅的感觉，赶紧先踩个刹车，控制下车速；因为现在你需要考虑 gitlab-ci 的安全配置了，这很重要！

首先 gitlab 项目的基本安全就是项目成员控制，访问项目的权限分为：所有者（Owner），维护者（Maintainer），开发者（Developer），报告者（Reporter），访客（Guest）；详细的权限介绍请查阅官方文档，这里简单地介绍两类权限：所有者和维护者属于`特权用户`，开发者属于`普通用户`，他们应该具有如下权限区分：

- 特权用户对整个项目负责，包括项目代码开发、配置管理、CI流程、测试环境、生产环境等
- 特权用户可以提交代码到所有分支包括 master/release 分支，执行所有 ci job
- 普通用户只负责对应项目模块代码开发、不接触程序配置、只能访问测试环境
- 普通用户只能提交代码到 develop/feature 分支，只能执行这两个分支的 ci job

以下的安全实践配置作为个人经验分享，仅作参考；如果你的项目需要更高的安全性，请阅读 gitlab-ci 官方相关文档，尝试找到属于自己的最佳实践。

- 正确设置项目成员（Settings > Members），严格限制项目维护者（Maintainer）人数，大部分应该作为开发者（Developer）提交代码
- 配置项目受保护分支/受保护标签，一般把master/release分支设置成受保护分支，限制只有维护者才能在保护分支commit和merge，从而限制只有维护者才能执行部署生产的 ci job，http://gitlab.test.com/help/user/project/protected_branches.md
- 配置受保护的变量，受保护的变量只在受保护分支和受保护tag的pipeline中可见，防止生产环境配置参数泄露，http://gitlab.test.com/help/ci/variables/README#protected-variables
- 配置受保护的Runner，只能执行受保护分支上的 ci jobs
- CICD Pipelines 中发布生产的任务请设置手动执行，同样生产的回退任务设置手动执行

# **应用实践**

## go web 应用部署

### 容器化 GO 应用

Golang 作为服务器端新兴热门语言同时也是容器技术的主要编写语言备受关注；它简洁、有趣、并行、安全等特点让 GO 应用容器化相对省心；一般来说做下时间本地化、安装信任根证书，然后把编译生成的二进制拷贝进去即可。

### 一个演示 GO WEB 应用

[hellogo 代码](https://github.com/easzlab/kubeasz/blob/master/docs/practice/go_web_app/hellogo.go)

### Dockerfile

作为演示项目的Dockerfile比较简单，请看 [Dockerfile 文件](https://github.com/easzlab/kubeasz/blob/master/docs/practice/go_web_app/Dockerfile)

```bash
# a demon for containerize golang web apps
#
# @author:
# @repo:    
# @ref:     

# stage 1: build src code to binary
FROM golang:1.13-alpine3.10 as builder

COPY *.go /app/

RUN cd /app && go build -o hellogo .

# stage 2: use alpine as base image
FROM alpine:3.10

RUN apk update && \
    apk --no-cache add tzdata ca-certificates && \
    cp -f /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    apk del tzdata && \
    rm -rf /var/cache/apk/*

COPY --from=builder /app/hellogo /hellogo

CMD ["/hellogo"] 
```

- 采用 docker 多阶段编译，使生成的目标镜像最小
- 使用 alpine 基础镜像
- 安装 tzdata 做时间本地化
- 安装信任根证书

一个真实复杂go项目的Dockerfile可能如这个例子：[复杂 Dockerfile](https://github.com/easzlab/kubeasz/blob/master/docs/practice/go_web_app/Dockerfile-more)

### 制作镜像

在 Dockerfile 文件所在目录，执行

```bash
docker build -t hellogo:v1.0 .
```

### 本地测试应用

- 1.单机运行 hellogo 容器应用

```bash
docker run -d --name hello -p3000:3000 hellogo:v1.0
```

- 2.验证测试

```bash
# 查看本地监听端口
$ ss -ntl|grep 3000
LISTEN   0         128                       *:3000                   *:*

# 查看应用状态
$ curl localhost:3000
Hello, Go! I'm instance 987 running version 1.2 at 13109-10-13 08:39:11

$ curl localhost:3000/health -i
HTTP/1.1 200 OK
Date: Sun, 13 Oct 2019 00:39:15 GMT
Content-Length: 0

$ curl localhost:3000/version
1.2
```

### 在 k8s 上运行演示应用

- 可以参考项目`github.com/easzlab/kubeasz` 快速搭建一个本地 k8s 测试环境
- 1.编写基于k8s的应用编排文件 [hellogo.yaml](https://github.com/easzlab/kubeasz/blob/master/docs/practice/go_web_app/hellogo.yaml)
  - 设置应用副本数`replicas: 3`
  - 预设新副本启动延迟5秒`minReadySeconds: 5`
  - 设置滚动更新策略
  - 设置资源使用限制，安装实际情况修改
  - 设置服务对外暴露方式 NodePort，根据实际情况修改端口，或者使用 ingress 方式
- 2.在 k8s 上运行应用

```bash
# 运行
$ kubectl apply -f hellogo.yaml

# 验证
$ kubectl get pod
NAME                             READY   STATUS    RESTARTS   AGE
hellogo-deploy-854dcd85c-2zm9l   1/1     Running   0          12m
hellogo-deploy-854dcd85c-7nfk5   1/1     Running   0          12m
hellogo-deploy-854dcd85c-ns7fp   1/1     Running   0          12m

$kubectl get deploy
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
hellogo-deploy   3/3     3            3           13m

$kubectl get svc
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
hellogo-svc   NodePort    10.68.194.109   <none>        80:30000/TCP   13m

# 使用curl测试应用三副本状态（用curl多次访问看到三个不同`instance id`）
$ curl http://192.168.111.3:30000
Hello, Go! I'm instance 629 running version 1.2 at 13109-10-13 09:06:25

$ curl http://192.168.111.3:30000
Hello, Go! I'm instance 722 running version 1.2 at 13109-10-13 09:06:27

$curl http://192.168.111.3:30000
Hello, Go! I'm instance 799 running version 1.2 at 13109-10-13 09:06:28
```

## java 应用部署

### JAVA WAR 应用迁移 K8S 实践

初步思路是这样：应用代码与应用配置分离，应用代码打包成 docker 镜像存于内部 harbor 仓库，应用配置使用 configmap 挂载，这样不同的环境只需要修改 configmap 即可部署。

- 使用 maven 把 java 应用代码打包成 xxx.war
- 基于 tomcat 镜像和 xxx.war 做成应用 docker 镜像
- 编写 k8s deployment 文件，在 pod 指定上述应用镜像，同时把应用配置做成 configmap 挂载到 pod 里

经过多次尝试部署发现问题：configmap配置是可以挂载上去，但是会把目录下其他的文件删掉，而且tomcat 目录 webapps/xxxxx/下其他目录也消失了。原来是因为 tomcat 容器完全启动完成后才会解压 war包，而 configmap 配置文件是一开始就挂载上去了，导致失败。

- 调整应用镜像打包过程：xxx.war 先解压后再进行应用镜像打包

### 应用 gitlab CI/CD 集成

- 在内部gitlab创建项目，上传应用java代码，同时在项目根目录下新加如下目录和文件，配置相应的 gitlab-runner 和 环境变量参数

```bash
├── .app.yaml		# k8s deployment 部署模板文件 
├── config.yaml		# k8s configmap 配置模板文件
├── dockerfiles
│   └── Dockerfile	# Dockerfile 文件
├── .gitlab-ci.yml	# gitlab ci 配置文件
└── .ns.yaml		# k8s namespace 和 imagePullSecrets的配置文件
```

#### gitlab-ci 文件摘要

```bash
variables:
  PROJECT_NS: '$CI_PROJECT_NAMESPACE-$CI_JOB_STAGE'
  APP_NAME: '$CI_PROJECT_NAME-$CI_COMMIT_REF_SLUG'

stages:
  - package
  - beta

job_package:
  stage: package
  tags:
    - package-shell
  only:
    - master
    - /^feature-.*$/
  script:
  - mvn clean install -Dmaven.test.skip=true
  - unzip target/xxxx.war -d dockerfiles/project
  - cd dockerfiles && docker build -t harbor.test.lo/project/$CI_PROJECT_NAME:$CI_PIPELINE_ID .
  - docker login -u $HARBOR_USR -p $HARBOR_PWD harbor.test.lo
  - docker push harbor.test.lo/project/$CI_PROJECT_NAME:$CI_PIPELINE_ID
  - docker logout harbor.test.lo

job_push_beta:
  stage: beta
  tags:
    - beta-shell
  only:
    - master
    - /^feature-.*$/
  when: manual
  script:
  # 替换beta环境的参数配置
  - sed -i "s/PROJECT_NS/$PROJECT_NS/g" config.yaml .app.yaml .ns.yaml
  - sed -i "s/TemplateProject/$APP_NAME/g" config.yaml .app.yaml
  - sed -i "s/DB_HOST/$BETA_DB_HOST/g" config.yaml
  - sed -i "s/DB_PWD/$BETA_DB_PWD/g" config.yaml
  - sed -i "s/APP_REP/$BETA_APP_REP/g" .app.yaml
  - sed -i "s/ProjectImage/$CI_PROJECT_NAME:$CI_PIPELINE_ID/g" .app.yaml
  #
  - mkdir -p /opt/kube/$PROJECT_NS/$APP_NAME
  - cp -f .ns.yaml config.yaml .app.yaml /opt/kube/$PROJECT_NS/$APP_NAME
  - kubectl --kubeconfig=/etc/.beta/config apply -f .ns.yaml
  - kubectl --kubeconfig=/etc/.beta/config apply -f config.yaml
  - kubectl --kubeconfig=/etc/.beta/config apply -f .app.yaml

# 生产部署与beta环境类同，这里省略
```

#### Dockerfile 编写

```bash
FROM tomcat:8.5.33-jre8-alpine

COPY . /usr/local/tomcat/webapps/

# 设置tomcat日志使用的时区
RUN sed -i 's/^JAVA_OPTS=.*webresources\"$/JAVA_OPTS=\"$JAVA_OPTS -Djava.protocol.handler.pkgs=org.apache.catalina.webresources -Duser.timezone=GMT+08\"/g' /usr/local/tomcat/bin/catalina.sh
```

#### k8s deployment 配置举例

```bash
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: TemplateProject
  namespace: PROJECT_NS
spec:
  replicas: APP_REP
  template:
    metadata:
      labels:
        run: TemplateProject
    spec:
      containers:
      - name: TemplateProject
        image: harbor.test.lo/project/ProjectImage
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: db-config
          mountPath: "/usr/local/tomcat/webapps/project/xxxx/yyyy/config/datasource.properties"
          subPath: datasource.properties
      imagePullSecrets:
      - name: projectkey1
      volumes:
      - name: db-config
        configMap:
          name: TemplateProject-config
          defaultMode: 0640
          items:
          - path: datasource.properties
            key: datasource.properties

---
apiVersion: v1
kind: Service
metadata:
  labels:
    run: TemplateProject
  name: TemplateProject
  namespace: PROJECT_NS
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    run: TemplateProject
  sessionAffinity: None
```

#### k8s configmap 配置举例

```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: TemplateProject-config
  namespace: PROJECT_NS
data:
  datasource.properties: |
    dataSource.maxIdle = 5
    dataSource.maxActive = 41
    dataSource.driverClassName = com.mysql.jdbc.Driver
    dataSource.url = jdbc:mysql://DB_HOST:8066/project?useUnicode=true&characterEncoding=utf-8
    dataSource.username = username
    dataSource.password = DB_PWD
```

## Elasticsearch 部署实践

`Elasticsearch`是目前全文搜索引擎的首选，它可以快速地储存、搜索和分析海量数据；也可以看成是真正分布式的高效数据库集群；`Elastic`的底层是开源库`Lucene`；封装并提供了`REST API`的操作接口。

### 单节点 docker 测试安装

```bash
cat > es-start.sh << EOF
#!/bin/bash

sysctl -w vm.max_map_count=262144

docker run --detach \
   --name es01 \
   -p 9200:9200 -p 9300:9300 \
   -e "discovery.type=single-node" \
   -e "bootstrap.memory_lock=true" --ulimit memlock=-1:-1 \
   --ulimit nofile=65536:65536 \
   --volume /srv/elasticsearch/data:/usr/share/elasticsearch/data \
   --volume /srv/elasticsearch/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml \
   jmgao1983/elasticsearch:6.4.0
EOF
```

执行`sh es-start.sh`后，就在本地运行了。

- 验证 docker 镜像运行情况

```bash
root@docker-ts:~$ docker ps -a
CONTAINER ID        IMAGE                           COMMAND                  CREATED             STATUS              PORTS                                            NAMES
171f3fecb596        jmgao1983/elasticsearch:6.4.0   "/usr/local/bin/do..."   2 hours ago         Up 2 hours          0.0.0.0:9200->9200/tcp, 0.0.0.0:9300->9300/tcp   es01
```

- 验证 es 健康检查

```bash
root@docker-ts:~$ curl http://127.0.0.1:9200/_cat/health
epoch      timestamp cluster       status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1535523956 06:25:56  docker-es     green           1         1      0   0    0    0        0             0                  -                100.0%
```

### 在 k8s 上部署 Elasticsearch 集群

在生产环境下，Elasticsearch 集群由不同的角色节点组成：

- master 节点：参与主节点选举，不存储数据；建议3个以上，维护整个集群的稳定可靠状态
- data 节点：不参与选主，负责存储数据；主要消耗磁盘，内存
- client 节点：不参与选主，不存储数据；负责处理用户请求，实现请求转发，负载均衡等功能

这里使用`helm chart`来部署 (https://github.com/helm/charts/tree/master/incubator/elasticsearch)

- 1.安装 helm: 以本项目[安全安装helm](https://github.com/easzlab/kubeasz/blob/master/docs/guide/helm.md)为例
- 2.准备 PV: 以本项目[K8S 集群存储](https://github.com/easzlab/kubeasz/blob/master/docs/setup/08-cluster-storage.md)创建`nfs`动态 PV 为例
  - 编辑配置文件：roles/cluster-storage/defaults/main.yml

```bash
storage:
  nfs:
    enabled: "yes"
    server: "192.168.1.8"
    server_path: "/share"
    storage_class: "nfs-es"
    provisioner_name: "nfs-provisioner-01"
```

- 创建 nfs provisioner

```bash
$ ansible-playbook /etc/ansible/roles/cluster-storage/cluster-storage.yml
# 执行成功后验证
$ kubectl get pod --all-namespaces |grep nfs-prov
kube-system   nfs-provisioner-01-6b7fbbf9d4-bh8lh        1/1       Running   0          1d
```

- 3.安装 elasticsearch chart

```bash
$ cd /etc/ansible/manifests/es-cluster

# 如果你的helm安装没有启用tls证书，请忽略以下--tls参数
$ helm install --tls --name es-cluster --namespace elastic -f es-values.yaml elasticsearch
```

- 4.验证 es 集群

```bash
# 验证k8s上 es集群状态
$ kubectl get pod,svc -n elastic 
NAME                                                   READY   STATUS    RESTARTS   AGE
pod/es-cluster-elasticsearch-client-778df74c8f-7fj4k   1/1     Running   0          2m17s
pod/es-cluster-elasticsearch-client-778df74c8f-skh8l   1/1     Running   0          2m3s
pod/es-cluster-elasticsearch-data-0                    1/1     Running   0          25m
pod/es-cluster-elasticsearch-data-1                    1/1     Running   0          11m
pod/es-cluster-elasticsearch-master-0                  1/1     Running   0          25m
pod/es-cluster-elasticsearch-master-1                  1/1     Running   0          12m
pod/es-cluster-elasticsearch-master-2                  1/1     Running   0          10m

NAME                                         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                         AGE
service/es-cluster-elasticsearch-client      NodePort    10.68.157.105   <none>        9200:29200/TCP,9300:29300/TCP   25m
service/es-cluster-elasticsearch-discovery   ClusterIP   None            <none>        9300/TCP                        25m

# 验证 es集群本身状态
$ curl $NODE_IP:29200/_cat/health
1539335131 09:05:31 es-on-k8s green 7 2 0 0 0 0 0 0 - 100.0%

$ curl $NODE_IP:29200/_cat/indices?v
health status index uuid pri rep docs.count docs.deleted store.size pri.store.size

root@k8s401:/etc/ansible$ curl 10.100.97.41:29200/_cat/nodes?
172.31.2.4 27 80 5 0.09 0.11 0.21 mi - es-cluster-elasticsearch-master-0
172.31.1.7 30 97 3 0.39 0.29 0.27 i  - es-cluster-elasticsearch-client-778df74c8f-skh8l
172.31.3.7 20 97 3 0.11 0.17 0.18 i  - es-cluster-elasticsearch-client-778df74c8f-7fj4k
172.31.1.5  8 97 5 0.39 0.29 0.27 di - es-cluster-elasticsearch-data-0
172.31.2.5  8 80 3 0.09 0.11 0.21 di - es-cluster-elasticsearch-data-1
172.31.1.6 18 97 4 0.39 0.29 0.27 mi - es-cluster-elasticsearch-master-2
172.31.3.6 20 97 4 0.11 0.17 0.18 mi * es-cluster-elasticsearch-master-1
```

### es 性能压测

如上已使用 chart 在 k8s上部署了 **7** 节点的 elasticsearch 集群；各位应该十分好奇性能怎么样；官方提供了压测工具[esrally](https://github.com/elastic/rally)可以方便的进行性能压测，这里省略安装和测试过程；压测机上执行：
`esrally --track=http_logs --target-hosts="$NODE_IP:29200" --pipeline=benchmark-only --report-file=report.md`
压测过程需要1-2个小时，部分压测结果如下：

```bash
-----------------------------------------------------
    _______             __   _____
   / ____(_)___  ____ _/ /  / ___/_________  ________
  / /_  / / __ \/ __ `/ /   \__ \/ ___/ __ \/ ___/ _ \
 / __/ / / / / / /_/ / /   ___/ / /__/ /_/ / /  /  __/
/_/   /_/_/ /_/\__,_/_/   /____/\___/\____/_/   \___/
------------------------------------------------------

|   Lap |                               Metric |         Task |       Value |    Unit |
|------:|-------------------------------------:|-------------:|------------:|--------:|
...
|   All |                       Min Throughput | index-append |     16903.2 |  docs/s |
|   All |                    Median Throughput | index-append |     17624.4 |  docs/s |
|   All |                       Max Throughput | index-append |     19382.8 |  docs/s |
|   All |              50th percentile latency | index-append |     1865.74 |      ms |
|   All |              90th percentile latency | index-append |     3708.04 |      ms |
|   All |              99th percentile latency | index-append |     6379.49 |      ms |
|   All |            99.9th percentile latency | index-append |     8389.74 |      ms |
|   All |           99.99th percentile latency | index-append |     9612.84 |      ms |
|   All |             100th percentile latency | index-append |     9861.02 |      ms |
|   All |         50th percentile service time | index-append |     1865.74 |      ms |
|   All |         90th percentile service time | index-append |     3708.04 |      ms |
|   All |         99th percentile service time | index-append |     6379.49 |      ms |
|   All |       99.9th percentile service time | index-append |     8389.74 |      ms |
|   All |      99.99th percentile service time | index-append |     9612.84 |      ms |
|   All |        100th percentile service time | index-append |     9861.02 |      ms |
|   All |                           error rate | index-append |           0 |       % |
|   All |                       Min Throughput |      default |        0.66 |   ops/s |
|   All |                    Median Throughput |      default |        0.66 |   ops/s |
|   All |                       Max Throughput |      default |        0.66 |   ops/s |
|   All |              50th percentile latency |      default |      770131 |      ms |
|   All |              90th percentile latency |      default |      825511 |      ms |
|   All |              99th percentile latency |      default |      838030 |      ms |
|   All |             100th percentile latency |      default |      839382 |      ms |
|   All |         50th percentile service time |      default |      1539.4 |      ms |
|   All |         90th percentile service time |      default |     1635.39 |      ms |
|   All |         99th percentile service time |      default |     1728.02 |      ms |
|   All |        100th percentile service time |      default |      1736.2 |      ms |
|   All |                           error rate |      default |           0 |       % |
...
```

从测试结果看：集群的吞吐可以（k8s es-client pod还可以扩展）；延迟略高一些（因为使用了nfs共享存储）；整体效果不错。

### 中文分词安装

安装 ik 插件即可，可以自定义已安装ik插件的es docker镜像：创建如下 Dockerfile

```
FROM jmgao1983/elasticsearch:6.4.0

RUN /usr/share/elasticsearch/bin/elasticsearch-plugin install \
  --batch https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v6.4.0/elasticsearch-analysis-ik-6.4.0.zip \
  && cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

### 参考阅读

1. [Elasticsearch 入门教程](http://www.ruanyifeng.com/blog/2017/08/elasticsearch.html)
2. [Elasticsearch 压测方案之 esrally 简介](https://segmentfault.com/a/1190000011174694)

## Mariadb 数据库集群

Mariadb 是从 MySQL 衍生出来的开源关系型数据库，目前兼容 mysql 5.7 版本；它也非常流行，拥有 Google Facebook 等重要企业用户。本文档介绍使用 helm charts 方式安装 mariadb cluster，仅供实践交流使用。

### 前提条件

- ### 已部署 k8s 集群，参考[这里](https://github.com/easzlab/kubeasz/blob/master/docs/setup/quickStart.md)

- 已部署 helm，参考[这里](https://github.com/easzlab/kubeasz/blob/master/docs/guide/helm.md)

- 集群提供持久性存储，参考[这里](https://github.com/easzlab/kubeasz/blob/master/docs/setup/08-cluster-storage.md)

这里演示使用 nfs 动态存储，编辑修改 nfs 存储部分参数

```bash
$ vi roles/cluster-storage/defaults/main.yml
storage:
  # nfs server 参数
  nfs:
    enabled: "yes"              # 启用 nfs
    server: "172.16.3.86"       # 设置 nfs 服务器地址
    server_path: "/data/nfs"    # 设置共享目录
    storage_class: "nfs-db"     # 定义 storage_class，后面pvc要调用这个 
    provisioner_name: "nfs-provisioner-01"  # 任意命名

# 配置完成，保存退出，运行下面命令
$ ansible-playbook /etc/ansible/roles/cluster-storage/cluster-storage.yml

# 确认nfs provisioner pod
$ kubectl get pod --all-namespaces |grep nfs
kube-system   nfs-provisioner-01-88694d78c-mrn7f            1/1     Running   0          6m
```

### mariadb charts 配置修改

按照惯例，直接把 chart 下载到本地，然后把配置复制 values.yaml 出来进行修改，这样方便以后整体更新 chart，安装实际使用需要修改配置文件

```bash
$ cd /etc/ansible/manifests/mariadb-cluster
# 编辑 my-values.yaml 修改以下部分

service:
  type: NodePort     # 方便集群外部访问
  port: 3306
  nodePort:
    master: 33306    # 设置主库的nodePort
    slave: 33307     # 设置从库的nodePort

rootUser:            # 设置 root 密码
  password: test.c0m
  forcePassword: true

db:                  # 设置初始测试数据库
  user: hello
  password: hello
  name: hello
  forcePassword: true

replication:         # 设置主从复制
  enabled: true
  user: replicator
  password: R4%forep11CAT0r
  forcePassword: true

master:
  affinity: {}
  antiAffinity: soft
  tolerations: []
  persistence:
    enabled: true    # 启用持久化存储
    mountPath: /bitnami/mariadb
    storageClass: "nfs-db"  # 设置使用 nfs-db 存储类
    annotations: {}
    accessModes:
    - ReadWriteOnce
    size: 5Gi        # 设置存储容量 

slave:
  replicas: 1
  affinity: {}
  antiAffinity: soft
  tolerations: []
  persistence:
    enabled: false   # 从库这里没有启用持久性存储
```

### 安装

使用 helm 安装

```bash
$ cd /etc/ansible/manifests/mariadb-cluster
$ helm install --name mariadb --namespace default -f my-values.yaml ./mariadb
```

### 验证

```bash
$ kubectl get pod,svc | grep mariadb
pod/mariadb-mariadb-master-0      1/1     Running   0          27m
pod/mariadb-mariadb-slave-0       1/1     Running   0          29m

service/mariadb                       NodePort    10.68.170.168   <none>        3306:33306/TCP       29m
service/mariadb-mariadb-slave         NodePort    10.68.151.95    <none>        3306:33307/TCP       29m
```

# 推荐工具

##  kuboard

### Kuboard 介绍

Kuboard 官方文档请参考 [https://kuboard.cn](https://kuboard.cn/)

Kuboard 是一款免费的 Kubernetes 管理工具，提供了丰富的功能：

- Kubernetes 基本管理功能
  - 节点管理
  - 名称空间管理
  - 存储类/存储卷管理
  - 控制器（Deployment/StatefulSet/DaemonSet/CronJob/Job/ReplicaSet）管理
  - Service/Ingress 管理
  - ConfigMap/Secret 管理
  - CustomerResourceDefinition 管理
- Kubernetes 问题诊断
  - Top Nodes / Top Pods
  - 事件列表及通知
  - 容器日志及终端
  - KuboardProxy (kubectl proxy 的在线版本)
  - PortForward (kubectl port-forward 的快捷版本)
  - 复制文件 （kubectl cp 的在线版本）
- 认证与授权
  - Github/GitLab 单点登录
  - KeyCloak 认证
  - LDAP 认证
  - 完整的 RBAC 权限管理
- Kuboard 特色功能
  - Kuboard 官方套件
    - Grafana+Prometheus 资源监控
    - Grafana+Loki+Promtail 日志聚合
  - Kuboard 自定义名称空间布局
  - Kuboard 中英文语言包

### 安装前提

Kuboard 只依赖于 Kubernetes API，您可以在多种情况下使用 Kuboard：

- 使用 kubeadm 安装的 Kubernetes 集群
- 使用二进制方式安装的 Kubernetes 集群
- 阿里云/腾讯云等云供应商托管的 Kubernetes 集群

Kuboard 对 Kubernetes 的版本兼容性，如下表所示

| Kubernetes 版本 | Kuboard 版本    | 兼容性 | 说明                                                         |
| --------------- | --------------- | ------ | ------------------------------------------------------------ |
| v1.18           | v1.0.x， v2.0.x | 😄      | 已验证                                                       |
| v1.17           | v1.0.x， v2.0.x | 😄      | 已验证                                                       |
| v1.16           | v1.0.x， v2.0.x | 😄      | 已验证                                                       |
| v1.15           | v1.0.x， v2.0.x | 😄      | 已验证                                                       |
| v1.14           | v1.0.x， v2.0.x | 😄      | 已验证                                                       |
| v1.13           | v1.0.x， v2.0.x | 😄      | 已验证                                                       |
| v1.12           | v1.0.x， v2.0.x | 😐      | Kubernetes Api v1.12 不支持 dryRun， Kuboard 不支持 Kubernetes v1.12 |
| v1.11           | v1.0.x， v2.0.x | 😐      | Kuboard 不支持 Kubernetes v1.11                              |

### 安装

#### 安装 Kuboard

```bash
kubectl apply -f https://kuboard.cn/install-script/kuboard.yaml
kubectl apply -f https://addons.kuboard.cn/metrics-server/0.3.6/metrics-server.yaml
```

#### 卸载 Kuboard

```bash
kubectl delete -f https://kuboard.cn/install-script/kuboard.yaml
kubectl delete -f https://addons.kuboard.cn/metrics-server/0.3.6/metrics-server.yaml
```

### 获取 Token

您可以获得管理员用户、只读用户的Token

#### 管理员用户

**拥有的权限**

- 此Token拥有 ClusterAdmin 的权限，可以执行所有操作

**执行命令**

```bash
echo $(kubectl -n kube-system get secret $(kubectl -n kube-system get secret | grep kuboard-user | awk '{print $1}') -o go-template='{{.data.token}}' | base64 -d)  
```

**输出**

取输出信息中 token 字段

```bash
eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLWc4aHhiIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI5NDhiYjVlNi04Y2RjLTExZTktYjY3ZS1mYTE2M2U1ZjdhMGYiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.DZ6dMTr8GExo5IH_vCWdB_MDfQaNognjfZKl0E5VW8vUFMVvALwo0BS-6Qsqpfxrlz87oE9yGVCpBYV0D00811bLhHIg-IR_MiBneadcqdQ_TGm_a0Pz0RbIzqJlRPiyMSxk1eXhmayfPn01upPdVCQj6D3vAY77dpcGplu3p5wE6vsNWAvrQ2d_V1KhR03IB1jJZkYwrI8FHCq_5YuzkPfHsgZ9MBQgH-jqqNXs6r8aoUZIbLsYcMHkin2vzRsMy_tjMCI9yXGiOqI-E5efTb-_KbDVwV5cbdqEIegdtYZ2J3mlrFQlmPGYTwFI8Ba9LleSYbCi4o0k74568KcN_w
```

### 访问 Kuboard

您可以通过NodePort、port-forward 两种方式当中的任意一种访问 Kuboard

#### 通过NodePort访问

Kuboard Service 使用了 NodePort 的方式暴露服务，NodePort 为 32567；您可以按如下方式访问 Kuboard。

```bash
http://任意一个Worker节点的IP地址:32567/
```

输入前一步骤中获得的 token，可进入 **Kubernetes 集群概览**，界面如下所示：

![image-20200617185652036](D:\学习资料\笔记\linux\assets\image-20200617185652036.png)

## KubeSphere 容器平台

### 什么是 KubeSphere

[KubeSphere](https://github.com/kubesphere/kubesphere) 是在 [Kubernetes](https://kubernetes.io/) 之上构建的**开源企业级容器平台**，提供全栈的 IT 自动化运维的能力，简化企业的 DevOps 工作流。KubeSphere 作为一个**全栈的容器平台**，不仅支持**安装和纳管原生 Kubernetes**，还设计了一套完整的管理界面方便开发者与运维人员在一个**统一的平台** 中安装与管理最常用的云原生工具，**从业务视角提供一致的用户体验来降低复杂性**。目前版本提供以下功能：

| 功能                                   | 介绍                                                         |
| -------------------------------------- | ------------------------------------------------------------ |
| Kubernetes 集群搭建与运维              | 支持在线 & 离线安装、升级与扩容 K8s 集群，支持安装 “云原生全家桶” |
| Kubernetes 资源可视化管理              | 比 K8s 原生 Dashboard 功能更丰富的控制面板，支持向导式创建与管理 K8s 资源 |
| 基于 Jenkins 的 DevOps 系统            | 支持图形化与脚本两种方式构建 CI/CD 流水线，内置 Source/Binary to Image 等 CD 工具 |
| 应用商店与应用生命周期管理             | 内置 Redis、MySQL 等十个常用应用，基于 Helm 提供应用上传、审核、发布、部署、下架等操作 |
| 基于 Istio 的微服务治理 (Service Mesh) | 提供可视化无代码侵入的 **灰度发布、熔断、流量治理与流量拓扑、分布式 Tracing** |
| 多租户管理                             | 提供基于角色的细粒度多租户统一认证，支持 **对接企业 LDAP/AD**，提供多层级的权限管理 |
| 丰富的可观察性功能                     | UI 提供集群/工作负载/Pod/容器等多维度的监控、日志、告警与通知 |
| 存储管理                               | 支持对接 Ceph、GlusterFS、NFS，支持可视化管理 PVC、PV、StorageClass |
| 网络管理                               | 支持 Calico、Flannel，提供 Porter LB 帮助暴露物理环境 K8s 集群的 LoadBalancer 服务 |
| GPU support                            | 集群支持添加 GPU 与 vGPU，可运行 TensorFlow 等 ML 框架       |

### 在 Kubernetes 与 Kubeasz 之上安装 KubeSphere

KubeSphere 可以安装在任何私有或托管的 Kubernetes、私有云、公有云、VM 或物理环境之上，并且所有功能组件都是可插拔的。当使用 Kubeasz 完成 K8s 集群的安装后，可参考以下步骤在 Kubernetes 上安装 KubeSphere v2.1.0。

**前提条件**

> - `Kubernetes 版本` ： `1.13.0 ≤ K8s version < 1.16`；
> - `Helm 版本`: `2.10.0 ≤ Helm ＜ 3.0.0`，且已安装了 Tiller（v3.0 支持 Helm v3）；参考 [如何安装与配置 Helm](https://devopscube.com/install-configure-helm-kubernetes/)；
> - 集群的可用 CPU > 1 C，可用内存 > 2 G；且集群能够访问外网
> - 集群已有默认的存储类型（StorageClass）；
>
> 以上四项可参考 [前提条件](https://kubesphere.io/docs/v2.1/zh-CN/installation/prerequisites/) 进行验证。

1. 若待安装的环境满足以上条件则可以通过一条命令部署 KubeSphere。

```bash
$ kubectl apply -f https://raw.githubusercontent.com/kubesphere/ks-installer/master/kubesphere-minimal.yaml
```

2. 查看 ks-installer 安装过程中产生的动态日志，等待安装成功（约 10 min 左右）：

```bash
$ kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l app=ks-install -o jsonpath='{.items[0].metadata.name}') -f
```

![image-20200617190741524](D:\学习资料\笔记\linux\assets\image-20200617190741524.png)

3. 当 KubeSphere 的所有 Pod 都为 Running 则说明安装成功。使用 `http://IP:30880` 访问 Dashboard，默认账号为 `admin/P@88w0rd`。

**Tips**：KubeSphere 在 K8s 默认仅开启 **最小化安装**，执行以下命令开启可插拔功能组件的安装，开启安装前确认您的机器资源已符合 [资源最低要求](https://kubesphere.io/docs/v2.1/zh-CN/installation/intro/#可插拔功能组件列表)。

```bash
$ kubectl edit cm -n kubesphere-system ks-installer
```



### 延伸阅读

- [安装 Kubeasz 与 KubeSphere](https://kubesphere.com.cn/forum/d/716-play-with-kubesphere-and-kubeasz)
- [在 Linux 完整安装 KubeSphere 与 Kubernetes](https://kubesphere.com.cn/docs/v2.1/zh-CN/installation/intro/)
- [KubeSphere 官网](https://kubesphere.com.cn/zh-CN/)





# 推荐阅读

- [kubernetes-the-hard-way](https://github.com/kelseyhightower/kubernetes-the-hard-way)
- [feisky-Kubernetes 指南](https://github.com/feiskyer/kubernetes-handbook/blob/master/SUMMARY.md)
- [rootsongjc-Kubernetes 指南](https://github.com/rootsongjc/kubernetes-handbook)
- [opsnull 安装教程](https://github.com/opsnull/follow-me-install-kubernetes-cluster)

