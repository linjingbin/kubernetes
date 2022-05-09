[TOC]

### Kubernetes 组件的认证方式

Kubernetes 中的组件：在 kubernetes 中包含多个以独立进程形式运行的组件，这些组件之间通过 HTTP/GRPC 相互通信，以协同完成集群中应用的部署和管理工作。

kubernetes 组件示图：

![1590141086673](D:\学习资料\笔记\linux\assets\1590141086673.png)

从图中可以看到，Kubernetes 控制平面中包含了 etcd,kube-api-server,kube-scheduler,kube-controller-manager 等组件，这些组件会相互进行远程调用，例如 kube-api-server 会调用 etcd 接口存储数据，kube-controller-manager 会调用 kube-api-server 接口查询集群中的对象状态；同时，kube-api-server 也会和工作节点上的 kubelet 和 kube-proxy 进行通信，以在工作节点上部署和管理应用.

以上这些组件之间的相互调用都是通过网络进行的。在进行网络通信时，通信双方需要验证对方的身份，以避免恶意第三方伪造身份窃取信息或者对系统进行攻击。为了相互验证对方的身体，通信双方中的任何一方都需要做下面两件事情：

- 向对方提供标明自己身体的一个证书
- 验证对方提供的身体证书是否合法，是否伪造的？

在 kubernetes 中使用了数字证书来提供身份证明，采用一个通信双方都信任的 CA 来颁发证书。

![1590142731036](D:\学习资料\笔记\linux\assets\1590142731036.png)

在 Kubernetes 的组件之间进行通信时，数字证书的验证是在协议层面通过 TLS 完成的，除了需要在建立通信时提供相关的证书和密钥外，在应用层面并不需要进行特殊处理。采用 TLS 进行验证有两种方式：

- 服务器单向认证：只需要服务器端提供证书，客户端通过服务器端证书验证服务的身份，但服务器并不要求验证客户端的身体。这种情况一般适用于对 Internet 开放的服务，例如搜索引擎网站，任何客户端都可以连接到服务器上进行访问，但客户端需要验证服务器的身份，以避免连接到伪造的恶意站点。
- 双向 TLS 认证：除了客户端需要验证服务器的证书，服务器也要通过客户端的证书来验证客户端的身份。这种情况下服务器提供的敏感信息，只允许特定身体的客户端访问。

在 kubernetes 中，各个组件之间的通信就是采用双向 TLS 认证机制。在两个组件进行双向认证时，会涉及到下面这些证书相关的文件：

- 服务器端证书：服务器用于证明自身身份的数字证书，里面主要包含了服务器端的公钥以及服务器的身份信息。
- 服务器端私钥：服务器端证书中包含的公钥所对应的私钥。公钥和私钥是成对使用的，在进行 TLS 验证时，服务器使用该私钥来向客户端证明自己是服务器证书的拥有者。
- 客户端证书：客户端用于证明自身身份的数字证书，里面主要包含了客户端的公钥以及客户端的身份信息。
- 客户端私钥：客户端用于证明自身身份的数字证书，同理，客户端使用该私钥来向服务器端证明自己是客户端证书的拥有者。
- 服务器端 CA 根证书：签发服务器端证书的 CA 根证书，客户端使用该 CA 根证书来验证服务器端证书的合法性。
- 客户端 CA 根证书：签发客户端证书的 CA 根证书，服务器端使用该 CA 根证书来验证客户端证书的合法性。

![1590143947598](D:\学习资料\笔记\linux\assets\1590143947598.png)

### Kubernetes 中使用到的主要证书

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

> etcd 3套、kubenetes 5套，共8套证书。另同一个套内的证书必须是用同一个 CA 签署的。
>
> 原因在验证这些证书的一端。因为在要验证这些证书的一端，通常只能指定一个 **Root CA**。这样一来，被验证的证书自然都需要是被这同一个 **Root CA** 对应的私钥签署，不然不能通过认证。
>
> 其实实际上，使用一套证书（都使用一套CA来签署）一样可以搭建出K8S，一样可以上生产，但是理清这些证书的关系，在遇到因为证书错误，请求被拒绝的现象的时候，不至于无从下手，而且如果没有搞清证书之间的关系，在维护或者解决问题的时候，贸然更换了证书，弄不好会把整个系统搞瘫。



#### TLS bootstrapping 简化kubelet证书制作

Kubernetes1.4版本引入了一组签署证书用的API。这组API的引入，使我们可以不用提前准备kubelet用到的证书。

每个kubelet用到的证书都是独一无二的，因为它要绑定各自的IP地址，于是需要给每个kubelet单独制作证书，如果业务量很大的情况下，node节点会很多，这样一来kubelet的数量也随之增加，而且还会经常变动（增减Node）kubelet的证书制作就成为一件很麻烦的事情。使用TLS bootstrapping就可以省事儿很多。

**工作原理：**Kubelet第一次启动的时候，先用同一个 bootstrap token 作为凭证。这个 token 已经被提前设置为隶属于用户组 system：bootstrappers，并且这个用户组的权限也被限定为只能用来申请证书。 用这个bootstrap token通过认证后，kubelet申请到属于自己的两套证书（kubelet server、kube-apiserver client for kubelet），申请成功后，再用属于自己的证书做认证，从而拥有了kubelet应有的权限。这样一来，就去掉了手动为每个kubelet准备证书的过程，并且kubelet的证书还可以自动轮替更新

**官方文档参考**：<https://kubernetes.io/docs/tasks/tls/certificate-rotation/>

**kubelet 证书为何不同**

这样做是一个为了审计，另一个为了安全。 每个kubelet既是服务端（kube-apiserver 需要访问kubelet ），也是客户端（kubelet需要访问kube-apiserver），所以要有服务端和客户端两组证书。

服务端证书需要与服务器地址绑定，每个 kubelet 的地址都不相同，即使绑定域名也是绑定不同的域名，故服务端地址不同。

客户端证书也不应相同，每个 kubelet 的认证证书与所在机器的IP绑定后，可以防止一个 kubelet 的认证证书泄露以后，使从另外的机器上伪造的请求通过验证。

安全方面，如果每个 node 上保留了用于签署证书的 bootstrap token，那么 bootstrap token 泄漏以后，是不是可以随意签署证书了？安全隐患非常大。所以，kubelet启动成功以后，本地的 bootstrap token 需要被删除。

**需要准备的证书:**

- admin-key.pem
- admin.pem
- ca-key.pem
- ca.pem
- kube-proxy-key.pem
- kube-proxy.pem
- kubernetes-key.pem
- kubernetes.pem

**使用证书的组件如下：**

- etcd：使用 ca.pem、kubernetes-key.pem、kubernetes.pem
- kube-apiserver：使用 ca.pem、kubernetes-key.pem、kubernetes.pem
- kubelet：使用 ca.pem
- kube-proxy：使用 ca.pem、kube-proxy-key.pem、kube-proxy.pem
- kubectl：使用 ca.pem、admin-key.pem、admin.pem
- kube-controller-manager：使用 ca-key.pem、ca.pem

使用 CFSSL 来制作证书，它是 cloudflare 开发的一个开源的 PKI 工具，是一个完备的 CA 服务系统，可以签署、撤销证书等，覆盖了一个证书的整个生命周期。

> 注：一般情况下， k8s 中证书只需要创建一次，以后在向集群中添加新节点时只要将 /etc/kubernetes/ssl 目录下的证书拷贝到新节点即可。

**下载安装 cfssl 命令行工具**

```bash
[root@dong bin]$ mkdir -p /usr/local/bin/cfssl
[root@dong bin]$ wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 
[root@dong bin]$ wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 
[root@dong bin]$ wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64 
[root@dong bin]$ chmod +x cfssl*
[root@dong bin]$ mv cfssl_linux-amd64 /usr/local/bin/cfssl
[root@dong bin]$ mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
[root@dong bin]$ mv cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo
[root@dong bin]$ export PATH=/usr/local/bin:$PATH
```



#### 创建 CA 证书

**创建存放证书目录**

```bash
[root@dong bin]$ mkdir -p /opt/kubernetes/ssl/
[root@dong bin]$ cd /opt/kubernetes/ssl/
```

**创建证书配置文件**

```json
[root@dong ssl]$ vim ca-config.json
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

> 创建用来生成 CA 文件的 JSON 配置文件，这个文件后面会被各组件使用，包括了证书过期时间的配置。
>
> 字段说明：
>
> ca-config.json：可以定义多个 profiles，分别指定不同的过期时间、使用场景等参数；后续在签名证书时使用某个 profile；
>
> signing：表示该证书可以签名其他证书；生成的ca.pem证书中 CA=TRUE；
>
> server auth：表示client可以用该 CA 对 server 提供的证书进行验证；
>
> client auth：表示server可以用该 CA 对 client 提供的证书进行验证；
>
> expiry：过期时间

**创建CA证书签名请求（CSR）文件**

```json
[root@dong ssl]$ vim ca-csr.json
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

> “CN”：Common Name，kube-apiserver 从证书中提取该字段作为请求的用户名 (User Name)；浏览器使用该字段验证网站是否合法；
>
> “O”：Organization，kube-apiserver 从证书中提取该字段作为请求用户所属的组 (Group)；

**生成 CA 证书和私钥**

```bash
[root@dong ssl]$ cfssl gencert -initca ca-csr.json | cfssljson -bare ca
[root@dong ssl]$ ls | grep ca
ca-config.json
ca.csr
ca-csr.json
ca-key.pem
ca.pem
```

> 其中 ca-key.pem 是 ca 的私钥，ca.csr 是一个签署请求，ca.pem 是 CA 证书，是后面 kubernetes 组件会用到的 RootCA。



#### 创建 Kubernetes 证书

**创建 Kubernetes 证书签名请求文件 kubernetes-csr.json**

```bash
[root@dong ssl]$ vim kubernetes-csr.json
{
    "CN": "kubernetes",
    "hosts": [
      "127.0.0.1",
      "192.168.214.88",
      "192.168.214.89",
      "192.168.214.90",
      "192.168.214.200",
      "192.168.214.201",
      "192.168.214.202",
      "10.254.0.1",
      "192.168.214.210",
      "192.168.214.1/24",
      "kubernetes",
      "kube-api.wangdong.com",
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

> 如果 hosts 字段不为空则需要指定授权使用该证书的 IP 或域名列表。
>
> 由于该证书后续被 etcd 集群和 kubernetes master使用，将etcd、master节点的IP都填上，同时还有 service 网络的首IP。(一般是 kube-apiserver 指定的 service-cluster-ip-range 网段的第一个IP，如 10.254.0.1)
>
> 这里的设置包括一个私有镜像仓库，三个etcd，三个master，以上物理节点的IP也可以更换为主机名。

**生成 kubernetes 证书和私钥**

```bash
[root@dong ssl]$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes
[root@dong ssl]$ ls |grep kubernetes
kubernetes.csr
kubernetes-csr.json
kubernetes-key.pem
kubernetes.pem
```



#### 创建 admin 证书

**创建admin证书签名请求文件admin-csr.json**

```bash
[root@dong ssl]$ admin-csr.json
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

> 说明：
>
> 后续 kube-apiserver 使用 RBAC 对客户端(如 kubelet、kube-proxy、Pod)请求进行授权；
>
> kube-apiserver 预定义了一些 RBAC 使用的 RoleBindings，如 cluster-admin 将 Group system:masters 与 Role cluster-admin 绑定，该 Role 授予了调用kube-apiserver 的所有 API的权限；
>
> O指定该证书的 Group 为 system:masters，kubelet 使用该证书访问 kube-apiserver 时 ，由于证书被 CA 签名，所以认证通过，同时由于证书用户组为经过预授权的 system:masters，所以被授予访问所有 API 的权限；
>
> 注：这个admin 证书，是将来生成管理员用的kube config 配置文件用的，现在我们一般建议使用RBAC 来对kubernetes 进行角色权限控制， kubernetes 将证书中的CN 字段 作为User， O 字段作为 Group

**生成 admin 证书和私钥**

```bash
[root@dong ssl]$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin
[root@dong ssl]$ ls | grep admin
admin.csr
admin-csr.json
admin-key.pem
admin.pem
```

#### 创建 kube-proxy 证书

**创建 kube-proxy 证书签名请求文件 kube-proxy-csr.json**

```bash
root@dong ssl]$ vim kube-proxy-csr.json
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

> 说明：
>
> CN 指定该证书的 User 为 system:kube-proxy；
>
> kube-apiserver 预定义的 RoleBinding system:node-proxier 将User system:kube-proxy 与 Role system:node-proxier 绑定，该 Role 授予了调用 kube-apiserver Proxy 相关 API 的权限；

**生成kube-proxy证书和私钥**

```bash
[root@dong ssl]$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes  kube-proxy-csr.json | cfssljson -bare kube-proxy
[root@dong ssl]$ ls |grep kube-proxy
kube-proxy.csr
kube-proxy-csr.json
kube-proxy-key.pem
kube-proxy.pem
```

`经过上述操作，我们会用到如下文件：`

```bash
[root@dong ssl]# ls | grep pem
admin-key.pem
admin.pem
ca-key.pem
ca.pem
kube-proxy-key.pem
kube-proxy.pem
kubernetes-key.pem
kubernetes.pem
```

**查看证书信息**

```bash
[root@master1 ssl]$ cfssl-certinfo -cert kubernetes.pem
{
  "subject": {
    "common_name": "kubernetes",
    "country": "CN",
    "organization": "k8s",
    "organizational_unit": "System",
    "locality": "BeiJing",
    "province": "BeiJing",
    "names": [
      "CN",
      "BeiJing",
      "BeiJing",
      "k8s",
      "System",
      "kubernetes"
    ]
  },
  "issuer": {
    "common_name": "kubernetes",
    "country": "CN",
    "organization": "k8s",
    "organizational_unit": "System",
    "locality": "BeiJing",
    "province": "BeiJing",
    "names": [
      "CN",
      "BeiJing",
      "BeiJing",
      "k8s",
      "System",
      "kubernetes"
    ]
  },
  "serial_number": "321233745860282370502438768971300435157761820875",
  "sans": [
    "192.168.214.1/24",
    "kubernetes",
    "kube-api.wangdong.com",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local",
    "127.0.0.1",
    "192.168.214.88",
    "192.168.214.89",
    "192.168.214.90",
    "192.168.214.200",
    "192.168.214.201",
    "192.168.214.202",
    "10.254.0.1",
    "192.168.214.210"
  ],
  "not_before": "2019-03-12T11:26:00Z",
  "not_after": "2029-03-09T11:26:00Z",
  "sigalg": "SHA256WithRSA",
  "authority_key_id": "CB:34:54:33:1F:F4:37:E:E5:94:B7:F5:8A:3D:F4:A4:43:43:E2:7F",
  "subject_key_id": "EC:31:D8:5F:4:E3:6F:C2:7F:DA:A8:F0:BD:A:B9:1F:56:7B:9A:DF",
  "pem": "-----BEGIN CERTIFICATE-----\nMIIExjCCA66gAwIBAgIUOESejeFvtUe1qwPcXOQdC9a6iMswDQYJKoZIhvcNAQEL\nBQAwZTELMAkGA1UEBhMCQ04xEDAOBgNVBAgTB0JlaUppbmcxEDAOBgNVBAcTB0Jl\naUppbmcxDDAKBgNVBAoTA2s4czEPMA0GA1UECxMGU3lzdGVtMRMwEQYDVQQDEwpr\ndWJlcm5ldGVzMB4XDTE5MDMxMjExMjYwMFoXDTI5MDMwOTExMjYwMFowZTELMAkG\nA1UEBhMCQ04xEDAOBgNVBAgTB0JlaUppbmcxEDAOBgNVBAcTB0JlaUppbmcxDDAK\nBgNVBAoTA2s4czEPMA0GA1UECxMGU3lzdGVtMRMwEQYDVQQDEwprdWJlcm5ldGVz\nMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAteZIJbL5G2ZHEKajyVe7\nv4E1F9K9RzLTxghStRo808QOpVclOkFRHCi2qplrFrQmW4d/5AhJmofdoBuwIe/T\n3UgrhlPj1rWC5DhaG8J7+wOIp62yURslnXE+A+EsXQLXxeKxrbrodNwTmGJHXdGl\nv2pi0lyAgewdnhJHcYTvQbrDvbxpqYOHqKzJ3sqm1TSjnWSI9C1Hk/iF9xmjA4CG\nLDHocnxzNv+T/qSofv0yyGgA/HovlNxP+jSIwaWJu3QHhOxV3k2Bj7i0jSJoq3n9\nDl4co22Ge4SLiI2zPZayt9whzSyUoc5eloYJ1w7INcmfz2gOYl7L3godLg/gI5Eh\nNQIDAQABo4IBbDCCAWgwDgYDVR0PAQH/BAQDAgWgMB0GA1UdJQQWMBQGCCsGAQUF\nBwMBBggrBgEFBQcDAjAMBgNVHRMBAf8EAjAAMB0GA1UdDgQWBBTsMdhfBONvwn/a\nqPC9CrkfVnua3zAfBgNVHSMEGDAWgBTLNFQzH/Q3DuWUt/WKPfSkQ0PifzCB6AYD\nVR0RBIHgMIHdghAxOTIuMTY4LjIxNC4xLzI0ggprdWJlcm5ldGVzghVrdWJlLWFw\naS53YW5nZG9uZy5jb22CEmt1YmVybmV0ZXMuZGVmYXVsdIIWa3ViZXJuZXRlcy5k\nZWZhdWx0LnN2Y4Iea3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVygiRrdWJl\ncm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWyHBH8AAAGHBMCo1liHBMCo\n1lmHBMCo1lqHBMCo1siHBMCo1smHBMCo1sqHBAr+AAGHBMCo1tIwDQYJKoZIhvcN\nAQELBQADggEBADtxejgpSQy913K9YyVLcS2k9bs41mZZbuokyrZiJeJ/CKZKzCo+\ngnF7P9/35IjkNlnYhlpUTTIJbnlQY8mDyKx1AaZOkzr+2djYRpg2vL3E7+CdRldQ\nUpNANSITolInKqboXev8SlLF9Mc/dWqgZzoifezuEkZ+c5KM6MY6MpMDVjVKNBxy\nJTd3bZNaPopop8IWxLAel5IQbzUhooswtzUxUslwKnYYC9tsKc5AgiXdehCnbGNf\nIr6wkK2OJBJStNPqarnpH6FZ6JxJ+qt59SdNhLixOT84HBR7ews/ZCYhQuPaJTy2\nwIb0XOtxILF3JMBNW/n21IyhF0vXsMdLg+o=\n-----END CERTIFICATE-----\n"
}
```





#### Service Account 证书

Kubernetes 中有两类用户，一类为 user account，一类为 service account。service account 主要被 pod 用于访问 kube-apiserver。在为一个 pod 指定了 service  account 后，kubenetes 会为该 service account 生成一个 JWT token，并使用 secret 将该 service account token 加载到 pod 上。pod 中的应用可以使用 service account token 来访问 apiserver。service account 证书被用于生成和验证 service account token。该证书的用法和前面介绍的其他证书不同，因为实际上的是其公钥和私钥。而并不需要对证书进行验证。

service account 证书的公钥和私钥分别被配置到了 kube-apiserver 和 kube-controller-manager 命令行参数中，如下所示：

```bash
usr/local/bin/kube-apiserver \\ 
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\          # 用于验证 service account token 的公钥
  ...
  
 /usr/local/bin/kube-controller-manager \\
 --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem  # 用于对 service account token 进行签名的私钥
 ...
```

下图展示了 kubernetes 中生成、使用和验证 service account token 的过程。

![1590245092641](D:\学习资料\笔记\linux\assets\1590245092641.png)

认证方法： 客户端证书采用是 token 方式

Kubernetes 提供了两种客户端认证的方法：

- 控制面组件采用的是客户端数字证书
- 而在集群中部署的应用则采用了 service account token 的方式。













