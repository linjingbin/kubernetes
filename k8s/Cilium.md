[toc]



# Cilium



## Kubernetes网络方案Cilium入门



![service map ex001 1 1024x496](D:\学习资料\笔记\k8s\k8s图\service-map-ex001-1-1024x496.png)

> *最近业界使用范围最广的K8S CNI网络方案*[Calico宣布支持eBPF](https://www.projectcalico.org/introducing-the-calico-ebpf-dataplane/)*，而作为第一个通过eBPF实现了kube-proxy所有功能的K8S网络方案——Cilium，它的先见之名是否能转成优势，继而成为CNI新的头牌呢？今天我们一起来入门最Cool Kubernetes网络方案Cilium。*



### Cilium介绍

> *以下基于*[Cilium官网文档](https://cilium.readthedocs.io/en/stable/)*翻译整理。*



### 当前趋势

现代数据中心的应用系统已经逐渐转向基于微服务架构的开发体系，一个微服务架构的应用系统是由多个小的独立的服务组成，它们之间通过轻量通信协议如HTTP、gRPC、Kafka等进行通信。微服务架构下的服务天然具有动态变化的特点，结合容器化部署，时常会引起大规模的容器实例启动或重启。要确保这种向高度动态化的微服务应用之间的安全可达，既是挑战，也是机遇。



### 现有问题

传统的Linux网络访问安全控制机制（如iptables）是基于静态环境的IP地址和端口配置网络转发、过滤等规则，但是IP地址在微服务架构下是不断变化的，非固定的；出于安全目的，协议端口(例如HTTP传输的TCP端口80)也不再固定用来区分应用系统。为了匹配大规模容器实例快速变化的生命周期，传统网络技术需要维护成千上万的负载均衡规则和访问控制规则，并且需要以不断增长的频率更新这些规则，而如果没有准确的可视化功能，要维护这些规则也是十分困难，这些对传统网络技术的可用性和性能都是极大的挑战。比如经常会有人对kube-proxy基于iptables的服务负载均衡功能在大规模容器场景下具有严重的性能瓶颈，同时由于容器的创建和销毁非常频繁，基于IP做身份关联的故障排除和安全审计等也很难实现。



### 解决方案

Cilium作为一款Kubernetes CNI插件，从一开始就是为大规模和高度动态的容器环境而设计，并且带来了API级别感知的网络安全管理功能，通过使用基于Linux内核特性的新技术——[BPF](https://docs.cilium.io/en/stable/bpf/)，提供了基于service/pod/container作为标识，而非传统的IP地址，来定义和加强容器和Pod之间网络层、应用层的安全策略。因此，Cilium不仅将安全控制与寻址解耦来简化在高度动态环境中应用安全性策略，而且提供传统网络第3层、4层隔离功能，以及基于http层上隔离控制，来提供更强的安全性隔离。

另外，由于BPF可以动态地插入控制Linux系统的程序，实现了强大的安全可视化功能，而且这些变化是不需要更新应用代码或重启应用服务本身就可以生效，因为BPF是运行在系统内核中的。

以上这些特性，使Cilium能够在大规模容器环境中也具有高度可伸缩性、可视化以及安全性



### 部署Cilium

部署Cilium非常简单，可以通过单独的yaml文件部署全部组件（目前我使用了这个方式部署了1.7.1版本），也可以通过helm chart一键完成。重要的是部署环境和时机：

1. 官方建议所有部署节点都使用Linux最新稳定内核版本，这样所有的功能都能启用，具体部署环境建议可以参照[这里](https://cilium.readthedocs.io/en/stable/install/system_requirements/)。
2. 作为一个Kubernetes网络组件，它应该在部署Kubernetes其他基础组件之后，才进行部署。这里，我自己遇到的问题是，因为还没有CNI插件，coredns组件的状态一直是pending的，直到部署完Cilium后，coredns完成了重置变成running状态。 下图是Cilium的整体部署组件图：

![img](D:\学习资料\笔记\k8s\k8s图\cilium-provision.png)



### 测试安装效果

官方提供了一个[connectivity检查工具](https://github.com/cilium/cilium/blob/master/examples/kubernetes/connectivity-check/connectivity-check.yaml)，以检测部署好的Cilium是否工作正常。如果你的网络环境有些限制，我作了一些简单修改，可以参照[这里](https://github.com/nevermosby/K8S-CNI-Cilium-Tutorial/blob/master/cilium/connectivity-check.yaml)。部署起来很简单，请确保至少有两个可用的节点，否则有几个deployment会无法成功运行：

```sh
$ kubectl apply -f connectivity-check.yaml

NAME                                    READY   UP-TO-DATE   AVAILABLE   AGE
echo-a                                  1/1     1            1           16d
echo-b                                  1/1     1            1           16d
host-to-b-multi-node-clusterip          1/1     1            1           16d
host-to-b-multi-node-headless           1/1     1            1           16d
pod-to-a                                1/1     1            1           16d
pod-to-a-allowed-cnp                    1/1     1            1           16d
pod-to-a-external-1111                  1/1     1            1           16d
pod-to-a-l3-denied-cnp                  1/1     1            1           16d
pod-to-b-intra-node                     1/1     1            1           16d
pod-to-b-multi-node-clusterip           1/1     1            1           16d
pod-to-b-multi-node-headless            1/1     1            1           16d
pod-to-external-fqdn-allow-google-cnp   1/1     1            1           16d
```

如果所有的deployment都能成功运行起来，说明Cilium已经成功部署并工作正常。

![img](D:\学习资料\笔记\k8s\k8s图\draggedimage-13.png)



### 网络可视化神器Hubble

上文提到了Cilium强大之处就是提供了简单高效的网络可视化功能，它是通过[Hubble](https://github.com/cilium/hubble)组件完成的。[Cilium在1.7版本后推出并开源了Hubble](https://cilium.io/blog/2019/11/19/announcing-hubble)，它是专门为网络可视化设计，能够利用Cilium提供的eBPF数据路径，获得对Kubernetes应用和服务的网络流量的深度可见性。这些网络流量信息可以对接Hubble CLI、UI工具，可以通过交互式的方式快速诊断如与DNS相关的问题。除了Hubble自身的监控工具，还可以对接主流的云原生监控体系——Prometheus和Grafana，实现可扩展的监控策略。 



#### 部署Hubble 和 Hubble UI

官方提供了基于Helm Chart部署方式，这样可以灵活控制部署变量，实现不同监控策略。出于想要试用hubble UI和对接Grafana，我是这样的部署的：

```sh
$ helm template hubble \
    --namespace kube-system \
    --set metrics.enabled="{dns:query;ignoreAAAA;destinationContext=pod-short,drop:sourceContext=pod;destinationContext=pod,tcp,flow,port-distribution,icmp,http}" \
    --set ui.enabled=true \
    > hubble.yaml
$ kubectl apply -f hubble.yaml
# 包含两个组件
# - daemonset hubble
# - deployment hubble UI
> kubectl get pod -n kube-system |grep hubble
hubble-67ldp                       1/1     Running   0          21h
hubble-f287p                       1/1     Running   0          21h
hubble-fxzms                       1/1     Running   0          21h
hubble-tlq64                       1/1     Running   1          21h
hubble-ui-5f9fc85849-hkzkr         1/1     Running   0          15h
hubble-vpxcb                       1/1     Running   0          21h 
```



#### 运行效果

由于默认的Hubble UI只提供了ClusterIP类似的service，无法通过外部访问。因此需要创建一个NodePort类型的service，如下所示：

```yaml
# hubble-ui-nodeport-svc.yaml
kind: Service
apiVersion: v1
metadata:
  namespace: kube-system
  name: hubble-ui-np
spec:
  selector:
    k8s-app: hubble-ui
  ports:
    - name: http
      port: 12000
      nodePort: 32321
  type: NodePort
```

执行`kubectl apply -f hubble-ui-nodeport-svc.yaml`，就可以通过任意集群节点IP地址加上32321端口访问Hubble UI的web服务了。打开效果如下所示：

![img](D:\学习资料\笔记\k8s\k8s图\hubble-ui-000.png)

页面上半部分是之前部署的一整套conectivity-check组件的数据流向图，官方叫做`Service Map`，默认情况下可以自动发现基于网络3层和4层的访问依赖路径，看上去非常cool，也有点分布式链路追踪图的感觉。点击某个服务，还能看到更为详细的关系图：

![service map ex001](D:\学习资料\笔记\k8s\k8s图\service-map-ex001.png)

下图是kube-system命名空间下的数据流图，能看到Hubble-UI组件和Hubble组件是通过GRPC进行通信的，非常有趣。但令人感到的好奇的是，为何没有显示Kubernetes核心组件之间的调用关系图。

![DraggedImage](D:\学习资料\笔记\k8s\k8s图\DraggedImage.png)

页面的下半部分默认显示的是对于每条数据流路径的详细描述，包括发起请求的pod名称、发起请求的service名称、请求目标的pod名称、请求目标的service名称、目标IP、目标端口、目标7层信息、请求状态、最后一次查看时间等，如下图所示：

![hubble ui flow 000](https://cilium.io/static/3c324c3ed50e2c7ae25924b8ce66ec36/29007/hubble-ui-flow-000.png)

点击任意一条flow，可以查看到更多详细信息：

![image-20210107182026753](D:\学习资料\笔记\k8s\k8s图\image-20210107182026753.png)

页面的下半部分可以通过点击切换成显示network policy模式，列出了当前命名空间下所有的网络策略：

![image-20210107182054368](D:\学习资料\笔记\k8s\k8s图\image-20210107182054368.png)

如果想开启网络7层的可视化观察，就需要对目标pod进行annotations ，感兴趣可以看[这里](http://docs.cilium.io/en/stable/policy/visibility/)，就不在入门篇详述了。

这样的网络可视化是不是你梦寐以求的，绝对能在排查请求调用问题的时候帮上大忙。



#### 对接Grafana+ Prometheus

如果你跟一样是Grafana+ Prometheus的忠实粉丝，那么使Hubble对接它们就是必然操作了。仔细的同学已经发现之前helm template的玄机了：

```sh
--set metrics.enabled="{dns:query;ignoreAAAA;destinationContext=pod-short,drop:sourceContext=pod;destinationContext=pod,tcp,flow,port-distribution,icmp,http}"
# 上面的设置，表示开启了hubble的metrics输出模式，并输出以上这些信息。
# 默认情况下，Hubble daemonset会自动暴露metrics API给Prometheus。
```

你可以对接现有的Grafana+Prometheus服务，也可以部署一个简单的：

```sh
# 下面的命令会在命名空间cilium-monitoring下部署一个Grafana服务和Prometheus服务
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/v1.6/examples/kubernetes/addons/prometheus/monitoring-example.yaml

# 创建对应NodePort Service，方便外部访问web服务
kubectl expose deployment/grafana --type=NodePort --port=3000 --name=gnp -n cilium-monitoring 

kubectl expose deployment/prometheus --type=NodePort --port=9090 --name=pnp -n cilium-monitoring
```

完成部署后，打开Grafana网页，导入官方制作的[dashboard](https://github.com/cilium/hubble/blob/v0.5/tutorials/deploy-hubble-and-grafana/grafana.json)，可以快速创建基于Hubble的metrics监控。等待一段时间，就能在Grafana上看到数据了：

![image-20210107182228498](D:\学习资料\笔记\k8s\k8s图\image-20210107182228498.png)

![image-20210107182243387](D:\学习资料\笔记\k8s\k8s图\image-20210107182243387.png)

![image-20210107182258872](D:\学习资料\笔记\k8s\k8s图\image-20210107182258872.png)

Cilium配合Hubble，的确非常好用！



### 取代kube-proxy组件

Cilium另外一个很大的宣传点是宣称已经全面实现kube-proxy的功能，包括`ClusterIP`, `NodePort`, `ExternalIPs` 和 `LoadBalancer`，可以完全取代它的位置，同时提供更好的性能、可靠性以及可调试性。当然，这些都要归功于eBPF的能力。

官方文档中提到，如果你是在先有kube-proxy后部署的Cilium，那么他们是一个“共存”状态，Cilium会根据节点操作系统的内核版本来决定是否还需要依赖kube-proxy实现某些功能，可以通过以下手段验证是否能停止kube-proxy组件：

```sh
# 检查Cilium对于取代kube-proxy的状态
$ kubectl exec -it -n kube-system [Cilium-agent-pod] -- cilium status | grep KubeProxyReplacement

# 默认是Probe状态
# 当Cilium agent启动并运行，它将探测节点内核版本，判断BPF内核特性的可用性，
# 如果不满足，则通过依赖kube-proxy来补充剩余的Kubernetess，
# 并禁用BPF中的一部分功能
KubeProxyReplacement:   Probe   [NodePort (SNAT, 30000-32767), ExternalIPs, HostReachableServices (TCP, UDP)]

# 查看Cilium保存的应用服务访问列表
# 有了这些信息，就不需要kube-proxy进行中转了
$ kubectl exec -it -n kube-system [Cilium-agent-pod] -- cilium service list
ID   Frontend              Service Type   Backend
1    10.96.0.10:53         ClusterIP      1 => 100.64.0.98:53
                                          2 => 100.64.3.65:53
2    10.96.0.10:9153       ClusterIP      1 => 100.64.0.98:9153
                                          2 => 100.64.3.65:9153
3    10.96.143.131:9090    ClusterIP      1 => 100.64.4.100:9090
4    10.96.90.39:9090      ClusterIP      1 => 100.64.4.100:9090
5    0.0.0.0:32447         NodePort       1 => 100.64.4.100:9090
6    10.1.1.179:32447      NodePort       1 => 100.64.4.100:9090
7    100.64.0.74:32447     NodePort       1 => 100.64.4.100:9090
8    10.96.190.1:80        ClusterIP
9    10.96.201.51:80       ClusterIP
10   10.96.0.1:443         ClusterIP      1 => 10.1.1.171:6443
                                          2 => 10.1.1.179:6443
                                          3 => 10.1.1.188:6443
11   10.96.129.193:12000   ClusterIP      1 => 100.64.4.221:12000
12   0.0.0.0:32321         NodePort       1 => 100.64.4.221:12000
13   10.1.1.179:32321      NodePort       1 => 100.64.4.221:12000
14   100.64.0.74:32321     NodePort       1 => 100.64.4.221:12000
15   10.96.0.30:3000       ClusterIP
16   10.96.156.253:3000    ClusterIP
17   100.64.0.74:31332     NodePort
18   0.0.0.0:31332         NodePort
19   10.1.1.179:31332      NodePort
20   10.96.131.215:12000   ClusterIP      1 => 100.64.4.221:12000

# 查看iptables是否有kube-proxy维护的规则
$ iptables-save | grep KUBE-SVC
<Empty> # 说明kube-proxy没有维护任何应用服务跳转，即可以停止它了。
```



### 小结

Cilium作为当下最Cool的Kubernetes CNI网络插件，还有很多特性，如高阶network policy、7层流量控制等，这款基于BPF/eBPF打造出的简单、高效、易用的网络管理体验，有机会大家都来试用吧。





## 基于 BPF/XDP 实现 Kubernetes Service 负载均衡



### **Kubernetes 网络基础：访问集群内服务的几种方式**

Kubernetes 是一个分布式容器调度器，最小调度单位是 Pod。从网络的角度来说，可以认为 一个 Pod 就是网络命名空间的一个实例（an instance of network namespace）。一个 Pod 内可能会有多个容器，因此，多个容器可以共存于同一个网络命名空间。

需要注意的是：Kubernetes 只定义了网络模型，具体实现则是交给所谓的 CNI 插件 ，后者完成 Pod 网络的创建和销毁。本文接下来将以 Cilium CNI 插件作为例子。

Kubernetes 规定了每个 Pod 的 IP 在集群内要能访问，这是通过 CNI 来完成的：CNI 插件负责为 Pod 分配 IP 地址，然后为其创建和打通网络。除此之外，Kubernetes 没有对 CNI 插件做任何限制。尤其是，Kubernetes 没有对从集群外访问 Pod 的行为做任何规定。

接下来我们就来看看如何访问 Kubernetes 集群里的一个服务（通常会对应多个 backend pods）。



#### **PodIP（直连容器 IP）**

第一种方式是通过 PodIP 直接访问，这是最简单的方式。

![image-20210107160232053](D:\学习资料\笔记\k8s\k8s图\image-20210107160232053.png)

如上图所示，这个服务的 3 个 backend pods 分别位于两个 Node 上。当集群外的客户端 访问这个服务时，它会直接通过某个具体的 PodIP 来访问。

假设客户端和 Pod 之间的网络是可达的，那这种访问是没问题的。

但这种方式有几个缺点：

- Pod 会因为某些原因重建，而 Kubernetes 无法保证它每次都会分到同一个 IP 地址 。例如，如果 Node 重启了，Pod 很可能就会分到不同的 IP 地址，这对客户端来说个 大麻烦。
- 没有内置的负载均衡。即，客户端选择一个 PodIP 后，所有的请求都会发送到这个 Pod，而不是分散到不同的后端 Pod。



#### **HostPort（宿主机端口映射）**

第二种方式是使用所谓的 HostPort。

![image-20210107160602163](D:\学习资料\笔记\k8s\k8s图\image-20210107160602163.png)

如上图所示，在宿主机的 netns 分配一个端口，并将这个端口的所有流量转发到 后端 Pod。

这种情况下，客户端通过 Pod 所在的宿主机的 HostIP:HostPort 访问服务，例如上图中访问 10.0.0.1:10000；宿主机先对流量进行 DNAT，然后转发给 Pod。

这种方式的缺点：

- 宿主机的端口资源是所有 Pod 共享的，任何一个端口只能被一个 Pod 使用 ，因此在每台 Node 上，任何一个服务最多只能有一个 Pod（每个 backend 都是一 致的，因此需要使用相同的 HostPort）。对用户非常不友好。
- 和 PodIP 方式一样，没有内置的负载均衡。



#### **NodePort Service**

NodePort 和上面的 HostPort 有点像（可以认为是 HostPort 的增强版），也是将 Pod 暴 露到宿主机 netns 的某个端口，但此时，集群内的每个 Node 上都会为这个服务的 pods 预留这个端口，并且将流量负载均衡到这些 pods。

如下图所示，假设这里的 NodePort 是 30001。当客户端请求到达任意一台 node 的 30001 端口时，它可以对请求做 DNAT 然后转发给本节点内的 Pod，如下图所示：

![image-20210107160822681](D:\学习资料\笔记\k8s\k8s图\image-20210107160822681.png)

也可以 DNAT 之后将请求转发给其他节点上的 Pod，如下图所示：

![image-20210107160851198](D:\学习资料\笔记\k8s\k8s图\image-20210107160851198.png)

注意在后面跨宿主机转发的情况下，除了做 DNAT 还需要做 SNAT。

优点：

- 已经有了服务（Service）的概念，多个 Pod 属于同一个 Service，挂掉一个时其 他 Pod 还能继续提供服务。

- 客户端不用关心 Pod 在哪个 Node 上，因为集群内的所有 Node 上都开了这个端口并监听在那里，它们对全局的 backend 有一致的视图。

- 已经有了负载均衡，每个 node 都是 LB。

  在宿主机 netns 内访问这些服务时，通过 localhost:NodePort 就行了，无需 DNS 解析。

缺点：

- 大部分实现都是基于 SNAT，当 Pod 不在本节点时，导致 packet 中的真实客户端 IP 地址信息丢失，监控、排障等不方便。
- Node 做转发使得转发路径多了一跳，延时变大。



#### **ExternalIPs Service**

第四种从集群外访问 Service 的方式是 external IP。

如果有外部可达的 IP ，即集群外能通过这个 IP 访问到集群内特定的 nodes，那我们就可以通过这些 nodes 将流量转发到 Service 的后端 pods，并提供负载均衡。

如下图所示，1.1.1.1 是一个 external IP，所有目的 IP 地址是 1.1.1.1 的流量会 被底层的网络（Kubernetes 控制之外）转发到 node1。1.1.1.1:8080 在 Kubernetes 里定义了一个 Service，如果它将流量转发到本机内的 backend pod，需要做一次 DNAT：

![image-20210107161127724](D:\学习资料\笔记\k8s\k8s图\image-20210107161127724.png)

同样，这里的后端 Pod 也可以在其他 Node 上，这时除了做 DNAT 还需要做一次 SNAT， 如下图所示：

![image-20210107161222089](D:\学习资料\笔记\k8s\k8s图\image-20210107161222089.png)

优点：可以使用任何外部可达的 IP 地址来定义 Service 入口，只要用这个 IP 地址能访问集群内的至少一台机器即可。

缺点：

- External IP 在 Kubernetes 的控制范围之外，是由底层的网络平台提供的。例如，底层网络通过 BGP 宣告，使得 IP 能到达某些 nodes。
- 由于这个 IP 是在 Kubernetes 的控制之外，对 Kubernetes 来说就是黑盒，因此从集群内访问 external IP 是存在安全隐患的，例如 external IP 上可能运行了 恶意服务，能够进行中间人攻击。因此，Cilium 目前不支持在集群内通过 external IP 访问 Service。



#### **LoadBalancer Service**

第五种访问方式是所谓的 LoadBalancer 模式。针对公有云还是私有云，LoadBalancer 又分为两种。



##### **私有云**

如果是私有云，可以考虑实现一个自己的 cloud provider，或者直接使用 MetalLB。

如下图所示，这种模式和 externalIPs 模式非常相似，local 转发：

![image-20210107161416902](D:\学习资料\笔记\k8s\k8s图\image-20210107161416902.png)

remote 转发：

![image-20210107161459301](D:\学习资料\笔记\k8s\k8s图\image-20210107161459301.png)

但是，二者有重要区别：

- externalIPs 在 Kubernetes 的控制之外，使用方式是从某个地方申请一个 external IP， 然后填到 Service 的 Spec 里；这个 external IP 是存在安全隐患的，因为并不是 Kubernetes 分配和控制的；
- LoadBalancer 在 Kubernetes 的控制之内，只需要声明 这是一个 LoadBalancer 类型的 Service，Kubernetes 的 cloud-provider 组件 就会自动给这个 Service 分配一个外部可达的 IP，本质上 cloud-provider 做的事情就是从某个 LB 分配一个受信任的 VIP 然后填到 Service 的 Spec 里。

优点：LoadBalancer 分配的 IP 是归 Kubernetes 管的，用户无法直接配置这些 IP，因 此也就避免了前面 external IP 的流量欺骗（traffic spoofing）风险。

但注意这些 IP 不是由 CNI 分配的，而是由 LoadBalancer 实现分配。

MetalLB 能完成 LoadBalancer IP 的分配，然后基于 ARP/NDP 或 BGP 宣告 IP 的可达性。此外，MetalLB 本身并不在 critical fast path 上（可以认为它只是控制平面，完成 LoadBalancer IP 的生效，接下来的请求和响应流量，即数据平面，都不经过它），因此不 影响 XDP 的使用。



##### **公有云**

主流的云厂商都实现了 LoadBalancer，在它们提供的托管 Kubernetes 内可以直接使用。

特点：

- 有专门的 LB 节点作为统一入口。
- LB 节点再将流量转发到 NodePort。
- NodePort 再将流量转发到 backend pods。

如下图所示，local 转发：

![image-20210107161813991](D:\学习资料\笔记\k8s\k8s图\image-20210107161813991.png)

remote 转发：

![image-20210107161922708](D:\学习资料\笔记\k8s\k8s图\image-20210107161922708.png)

优点：

- LoadBalancer 由云厂商实现，无需用户安装 BGP 软件、配置 BGP 协议等来宣告 VIP 可达性。
- 开箱即用，主流云厂商都针对它们的托管 Kubernetes 集群实现了这样的功能。
- 在这种情况下，Cloud LB 负责检测后端 Node（注意不是后端 Pod）的健康状态。

缺点：

- 存在两层 LB：LB 节点转发和 node 转发。
- 使用方式因厂商而已，例如各厂商的 annotations 并没有标准化到 Kubernetes 中，跨云使用会有一些麻烦。
- Cloud API 非常慢，调用厂商的 API 来做拉入拉出非常受影响。



#### **ClusterIP Service**

最后一种是集群内访问 Service 的方式：ClusterIP 方式。

![image-20210107162100347](D:\学习资料\笔记\k8s\k8s图\image-20210107162100347.png)

ClusterIP 也是 Service 的一种 VIP，但这种方式只适用于从集群内访问 Service，例如 从一个 Pod 访问相同集群内的一个 Service。

ClusterIP 的特点：

- ClusterIP 使用的 IP 地址段是在创建 Kubernetes 集群之前就预留好的；
- ClusterIP 不可路由（会在出宿主机之前被拦截，然后 DNAT 成具体的 PodIP）；
- 只能在集群内访问（For in-cluster access only）。

实际上，当创建一个 LoadBalancer 类型的 Service 时，Kubernetes 会为我们自动创建三种类 型的 Service：

- LoadBalancer
- NodePort
- ClusterIP

这三种类型的 Service 对应着同一组 backend pods。



### **Kubernetes Service 负载均衡：Cilium 基于 BPF/XDP 的实现**

Cilium 基于 eBPF/XDP 实现了前面提到的所有类型的 Kubernetes Service。实现方式是：

- 在每个 node 上运行一个 cilium-agent；
- cilium-agent 监听 Kubernetes apiserver，因此能够感知到 Kubernetes 里 Service 的变化；
- 根据 Service 的变化动态更新 BPF 配置。

![image-20210107162408524](D:\学习资料\笔记\k8s\k8s图\image-20210107162408524.png)

如上图所示，Service 的实现由两个主要部分组成：

- 运行在 socket 层的 BPF 程序
- 运行在 tc/XDP 层的 BPF 程序

以上两者共享 service map 等资源，其中存储了 service 及其 backend pods 的映射关系。



#### **Socket 层负载均衡（东西向流量）**

Socket 层 BPF 负载均衡负责处理集群内的东西向流量。



**实现**

实现方式是：将 BPF 程序 attach 到 socket 的系统调用 hooks，使客户端直接和后端 Pod 建连和通信，如下图所示，这里能 hook 的系统调用包括 connect()、sendmsg()、 recvmsg()、getpeername()、bind() 等。

![image-20210107162805668](D:\学习资料\笔记\k8s\k8s图\image-20210107162805668.png)

这里的一个问题是，Kubernetes 使用的还是 cgroup v1，但这个功能需要使用 v2，而由于兼容性问题，v2 完全替换 v1 还需要很长时间。所以我们目前所能做的就是 支持 v1 和 v2 的混合模式。这也是为什么 Cilium 会 mount 自己的 cgroup v2 instance 的原因。

Cilium mounts cgroup v2, attaches BPF to root cgroup. Hybrid use works well for root v2.

具体到实现上：

- connect + sendmsg 做正向变换（translation）
- recvmsg + getpeername 做反向变换

这个变换或转换是基于 socket structure 的，此时还没有创建 packet，因此 不存在 packet 级别的 NAT！目前已经支持 TCP/UDP v4/v6, v4-in-v6。应用对此是无感知的，它以为自己连接到的还是 Service IP，但其实是 PodIP。



**查找后端 pods**

Service lookup 不一定能选到所有的 backend pods（scoped lookup），我们将 backend pods 拆成不同的集合。

这样设计的好处：可以根据流量类型，例如是来自集群内还是集群外（ internal/external），来选择不同的 backends。例如，如果是到达 node 的 external traffic，我们可以限制它只能选择本机上的 backend pods，这样相比于转发到其他 node 上的 backend 就少了一跳。

另外，还支持通配符（wildcard）匹配，这样就能将 Service 暴露到 localhost 或者 loopback 地址，能在宿主机 netns 访问 Service。但这种方式不会将 Service 暴露到宿主机外面。



**好处**

显然，这种 socket 级别的转换是非常高效和实用的，它可以直接将客户端 Pod 连接到某个 backend pod，与 kube-proxy 这样的实现相比，转发路径少了好几跳。

此外，bind BPF 程序在 NodePort 冲突时会直接拒绝应用的请求，因此相比产生流量（packet）然后在后面的协议栈中被拒绝，bind 这里要更加高效，因为此时流量（packet）都还没有产生。

对这一功能至关重要的两个函数：

- bpf_get_socket_cookie()，主要用于 UDP sockets，我们希望每个 UDP flow 都能选中相同的 backend pods。

- bpf_get_netns_cookie()，用在两个地方：

- - 用于区分 host netns 和 pod netns，例如检测到在 host netns 执行 bind 时，直接拒绝（reject）；
  - 用于 serviceSessionAffinity，实现在某段时间内永远选择相同的 backend pods。

由于 cgroup v2 不感知 netns，因此在这个 context 中我们没用 Pod 源 IP 信 息，通过这个 helper 能让它感知到源 IP，并以此作为它的 source identifier。



#### **TC & XDP 层负载均衡（南北向流量）**

第二种是进出集群的流量，称为南北向流量，在宿主机 tc 或 XDP hook 里处理。

![image-20210107163513549](D:\学习资料\笔记\k8s\k8s图\image-20210107163513549.png)

BPF 做的事情，将入向流量转发到后端 Pod，如果 Pod 在本节点，做 DNAT；如果在其他节点，还需要做 SNAT 或者 DSR，这些都是 packet 级别的操作。



#### **XDP 相关优化**

在引入 XDP 支持时，为了使 context 的抽象更加通用，我们做了很多事情。下面就其中的 一些展开讨论。



**BPF/XDP context 通用化**

DNAT/SNAT engine，DSR，conntrack 等等都是在 tc BPF 里实现的。BPF 代码中用 context 结构体传递数据包信息。

支持 XDP 时遇到的一个问题是：到底是将 context 抽象地更通用一些，还是直接实现一个 支持 XDP 的最小子集。我们最后是花大力气重构了以前几乎所有的 BPF 代码，来使得它更加通用。好处是共用一套代码，这样对代码的优化同时适用于 TC 和 XDP 逻辑。

下面是一个具体例子：

ctx 是一个通用抽象，具体是什么类型和 include 的头文件有关，基于 cxt 可以同时处理 tc BPF 和 XDP BPF 逻辑。

![image-20210107164030152](D:\学习资料\笔记\k8s\k8s图\image-20210107164030152.png)

例如对于 XDP 场景，编译时这些宏会被相应的 XDP 实现替换掉：

![image-20210107164058625](D:\学习资料\笔记\k8s\k8s图\image-20210107164058625.png)



**内联汇编：绕过编译器自动优化**

我们遇到的另一个问题是：tc BPF 中已经为 skb 实现了很多的 helper 函数，由于共用一 套抽象，因此现在需要为 XDP 实现对应的一套函数集。这些 helpers 都是 inline 函数， 而 LLVM 会对 inline 函数的自动优化会导致接下来校验器（BPF verifier）失败。

我们的解决方式是用 inline asm（内联汇编）来绕过这个问题。

下面是一个具体例子：xdp_load_bytes()，使用下面这段等价的汇编代码，才能 让 verifier 认出来：

![image-20210107164151520](D:\学习资料\笔记\k8s\k8s图\image-20210107164151520.png)



**避免在用户侧使用 generic XDP**

5.6 内核对 XDP 来说是一个里程碑式的版本（但可能不会是一个 LTS 版本），这个版本使得 XDP 在公有云上大规模可用了，例如 AWS ENA 和 Azure hv_netvsc 驱动。但如果想跨平台使用 XDP，那你只应该使用最基本的一些 API，例如 XDP_PASS/DROP/TX 等等。

Cilium 在用户侧只使用 native XDP（only supports native XDP on user side）， 我们也用 Generic XDP，但目前只限于 CI 等场景。

为什么我们避免在用户侧使用 generic XDP 呢？因为这套 LB 逻辑会运行在集群内的每个 node 上，目前 linearize skb 以及 bypass GRO 会增加太大的 overhead。



**自定义内存操作函数**

现在回到加载和存储字节相关的辅助函数（load and store bytes helpers）。

查看 BPF 反汇编代码时，发现内置函数会执行字节级别（byte-wise）的一些操作，因此我 们实现了自己优化过的 mem{cpy,zero,cmp,move}() 函数。这一点做起来还是比较容 易的，因为 LLVM 对栈外数据（non-stack data）没有上下文信息，例如 packet data 、map data，因而它无法准确地知道底层的架构是否支持高效的非对齐访问（unaligned access）。

另外，在基准测试中我们发现，大流量的场景下，bpf_ktime_get_ns() 在 XDP 中的开 销非常大，因此我们将 clock source 变成可选的，Cilium 启动时会执行检查，如果内 核支持，就自动切换到 bpf_jiffies64()（精度更低，但 conntrack 不需要那么高的 精度），这使得转发性能增加了大约 1.1Mpps。



**cb（control buffer）**

tc BPF 中大量使用 skb->cb[] 来传递数据，显然，XDP 中也是没有这个东西的。

为了在 XDP 中传递数据，我们最开始使用的是 xdp_adjust_meta()，但有两个缺点：

- missing driver support
- high rate of cache-misses

后来换成 per-CPU scratch map（每个 CPU 独立的、内容可随意修改的 map）, 增加了大约 1.2Mpps。



**bpf_map_update_elem()**

在 fast path 中有很多 bpf_map_update_elem() 调用，触发了 bucket spinlock。

如果流量来自多个 CPU，这里可以优化的是：先检查一下是否需要更新（这一步不需要加锁 ），如果原来已经存在，并且需要更新的值并没有变，那就直接返回。

![image-20210107164701682](D:\学习资料\笔记\k8s\k8s图\image-20210107164701682.png)



**bpf_fib_lookup()**

bpf_fib_lookup() 开销非常大，但在 XDP 中，例如 hairpin LB 场景，是不需要这个函数的，可以在编译时去掉。我们在测试环境的结果显示可以提高 1.5Mpps。



**静态 key**

作为这次分享的最后一个例子，不要对不确定的 LLVM 行为做任何假设。

我们在 BPF map 的基础上有大量的尾调用，它们有静态的 keys，能够在编译期间确定 key 的大小。我们还实现了一个内联汇编来做静态的尾递归调用，保证 LLVM 不会出现 尾调用相关的问题。

![image-20210107165026493](D:\学习资料\笔记\k8s\k8s图\image-20210107165026493.png)



#### **XDP 转发性能**

我们在 Kubernetes 集群测试了 XDP 对 Kubernetes Service 的转发。用 pktgen 生成 10Mpps 的 入向处理流量，然后让 Node 转发到位于其他节点的 backend pods。来看下几种不同的 负载均衡实现分别能处理多少。

![image-20210107165313577](D:\学习资料\笔记\k8s\k8s图\image-20210107165313577.png)

由上图可以看出：

- Cilium XDP 模式：能够处理全部的 10Mpps 入向流量，将它们转发到其他节点上的 backend pods。
- Cilium TC 模式：可以处理大约 2.8Mpps，虽然它的处理逻辑和 Cilium XDP 是类似的（除了 BPF helpers）。
- kube-proxy iptables 模式：能处理 2.4Mpps，这是 Kubernetes 的默认 Service 负载均衡实现。
- kube-proxy IPVS 模式：性能更差一些，因为它的 per-packet overhead 更大一 些，这里测试的 Service 只对应一个 backend pod。当 Service 数量更多时， IPVS 的可扩展性更好，相比 iptables 模式的 kube-proxy 性能会更好，但仍然没法跟我们基于 TC BPF 和 XDP 的实现相比（no comparison at all）。



softirq 开销也是类似的，如下图所示，流量从 1Mpps 到 2Mpps 再到 4Mpps 时， XDP 模式下的 softirq 开销都远小于其他几种模式。

![image-20210107165446233](D:\学习资料\笔记\k8s\k8s图\image-20210107165446233.png)

特别是 pps 到达某个临界点时，TC 和 Netfilter 实现中 softirq 开销会大到饱和 —— 占用几乎全部 CPU。



### 新的 BPF 内核扩展

下面介绍几个新的 BPF 内核扩展，主要是 Cilium 相关的场景。



#### **避免穿越内核协议栈**

主机收到的包，当其 backend 是本机上的 Pod 时，或者包是本机产生的，目的端是一个本 机端口，这个包需要跨越不同的 netns，例如从宿主机的 netns 进入到容 器的 netns，现在 Cilium 的做法是，将包送到内核协议栈，如下图所示：

![image-20210107165635795](D:\学习资料\笔记\k8s\k8s图\image-20210107165635795.png)

将包送到内核协议栈有两个原因（需要）：

- TPROXY 需要由内核协议栈完成：我们目前的 L7 proxy 功能会用到这个功能。
- Kubernetes 默认安装了一些 iptables rule，用来检测从连接跟踪的角度看是非法的连接 （‘invalid’ connections on asymmetric paths），然后 netfilter 会 drop 这些连接 的包。我们最开始时曾尝试将包从宿主机 tc 层直接 redirect 到 veth，但应答包却要 经过协议栈，因此形成了非对称路径，流量被 drop。因此目前进和出都都要经过协议栈。

但这样带来两个问题，如下图所示：

![image-20210107165833347](D:\学习资料\笔记\k8s\k8s图\image-20210107165833347.png)

- Pod 的出向流量在进入协议栈后，在 socket buffer 层会丢掉 socket 信息 （skb->sk gets orphaned at ip_rcv_core()），这导致包从主机设备发出去时， 我们无法在 FQ leaf 获得 TCP 反压（TCP back-pressure）。
- 转发和处理都是 packet 级别的，因此有 per-packet overhead。

不久之前，BPF TPROXY 已经合并到内核，因此最后一个真正依赖 Netfilter 的东西已经 解决了。因此我们现在可以在 TC 层做全部逻辑处理了，无需进入内核协议栈，如下图所示：

![image-20210107170435467](D:\学习资料\笔记\k8s\k8s图\image-20210107170435467.png)



**Redirection helpers**

两个用于 redirection 的 TC BPF helpers：

- bpf_redirect_neigh()
- bpf_redirect_peer()

从 IPVLAN driver 中借鉴了一些理念，实现到了 veth 驱动中。



**Pod Egress：bpf_redirect_neigh()**

![image-20210107170833009](D:\学习资料\笔记\k8s\k8s图\image-20210107170833009.png)

对于 Pod Egress 流量，我们会填充 src 和 dst mac 地址，这和原来 neighbor subsystem 做的事情相同；此外，我们还可以保留 skb 的 socket。这些都是由 bpf_redirect_neigh() 来完成的：

![image-20210107171330429](D:\学习资料\笔记\k8s\k8s图\image-20210107171330429.png)

整个过程大致实现如下，在 veth 主机端的 Ingress（对应 Pod 的 egress）调用这个方法的时候：

- 首先会查找路由，ip_route_output_flow()

- 将 skb 和匹配的路由条目（dst entry）关联起来，skb_dst_set()

- 然后调用到 neighbor 子系统，ip_finish_output2()

- - 填充 neighbor 信息，即 src/dst MAC 地址
  - 保留 skb->sk 信息，因此物理网卡上的 qdisc 都能访问到这个字段
  - 这就是 Pod 出向的处理过程。



**Pod Ingress：bpf_redirect_peer()**

入向流量，会有快速 netns 切换，从宿主机 netns 直接进入容器的 netns。

![image-20210107175812067](D:\学习资料\笔记\k8s\k8s图\image-20210107175812067.png)

这是由 bpf_redirect_peer() 完成的。

![image-20210107175839650](D:\学习资料\笔记\k8s\k8s图\image-20210107175839650.png)

在主机设备的 Ingress 执行这个 helper 的时候：

- 首先会获取对应的 veth pair，dev = ops->ndo_get_peer_dev(dev)，然后获取 veth 的对端（在另一个 netns）

- 然后，skb_scrub_packet()

- 设置包的 dev 为容器内的 dev，skb->dev = dev

- 重新调度一次，sch_handle_ingress()，这不会进入 CPU 的 backlog queue：

- - goto another_round
  - no CPU backlog queue



**veth to veth**

同宿主机上的两个 Pod 之间通信时，这两个 helper 也非常有用。因为我们已经在主机 netns 的 TC ingress 层了，因此能直接将其 redirect 到另一个容器的 Ingress 路径。

![image-20210107175936821](D:\学习资料\笔记\k8s\k8s图\image-20210107175936821.png)

这里比较好的一点是，需要针对老版本内核所做的兼容性非常少；因此，我们只需要在启动的 时候检测内核是否有相应的 helper，如果有，就用 redirection 功能；如果没有，就直接返回 TC_OK，走传统的内核协议栈方式，经过内核邻居子系统。支持这些功能无需对原有的 BPF datapath 进行大规模重构。

![image-20210107180004160](D:\学习资料\笔记\k8s\k8s图\image-20210107180004160.png)



**BPF redirection 性能**

下面看下性能。

TCP stream 场景，相比 Cilium baseline，转发带宽增加了 1.3Gbps，接近线速：

![image-20210107180028489](D:\学习资料\笔记\k8s\k8s图\image-20210107180028489.png)

更有趣的是 TCP_RR 的场景，以 transactions/second 衡量，提升了 2.9 倍，接近最 大性能：

![image-20210107180053080](D:\学习资料\笔记\k8s\k8s图\image-20210107180053080.png)







## 深入理解 Cilium 的 eBPF 收发包路径



### 1. 为什么要关注 eBPF？

#### 1.1 网络成为瓶颈

大家已经知道网络成为瓶颈，但我是从下面这个角度考虑的：近些年业界使用网络的方式 ，使其成为瓶颈（it is the bottleneck in a way that is actually pretty recent） 。

- 网络一直都是 I/O 密集型的，但直到最近，这件事情才变得尤其重要。
- 分布式任务（workloads）业界一直都在用，但直到近些年，这种模型才成为主流。虽然何时成为主流众说纷纭，但我认为最早不会早于 90 年代晚期。
- 公有云的崛起，我认为可能是网络成为瓶颈的最主要原因。



这种情况下，用于管理依赖和解决瓶颈的工具都已经过时了。

但像 eBPF 这样的技术使得网络调优和整流（tune and shape this traffic）变得简单很多。 eBPF 提供的许多能力是其他工具无法提供的，或者即使提供了，其代价也要比 eBPF 大 的多。



#### 1.2 eBPF 无处不在

eBPF 正在变得无处不在，我们可能会争论这到底是一件好事还是坏事（eBPF 也确实带了一 些安全问题），但当前无法忽视的事实是：Linux 内核的网络开发者们正在将 eBPF 应用 于各种地方（putting it everywhere）。其结果是，eBPF 与内核的默认收发包路径（ datapath）耦合得越来越紧（more and more tightly coupled with the default datapath）。



#### 1.3 性能就是金钱

“Metrics are money”， 这是今年 Paris Kernel Recipes 峰会上，来自 Synthesio 的 Aurelian Rougemont 的 精彩分享。

他展示了一些史诗级的调试（debugging）案例，感兴趣的可以去看看；但更重要的是，他 从更高层次提出了这样一个观点：理解这些东西是如何工作的，最终会产生资本收益（ understanding how this stuff works translates to money）。为客户节省金钱，为 自己带来收入。

如果你能从更少的资源中榨取出更高的性能，使软件运行更快，那显然你对公司的贡献就更大。Cilium 就是这样一个能让你带来更大价值的工具。

在进一步讨论之前，我先简要介绍一下 eBPF 是什么，以及为什么它如此强大。



### 2. eBPF 是什么？

BPF 程序有多种类型，图 2.1 是其中一种，称为 XDP BPF 程序。

- XDP 是 eXpress DataPath（特快数据路径）。
- XDP 程序可以直接加载到网络设备上。
- XDP 程序在数据包收发路径上很前面的位置就开始执行，下面会看到例子。

BPF 程序开发方式：

1. 编写一段 BPF 程序
2. 编译这段 BPF 程序
3. 用一个特殊的系统调用将编译后的代码加载到内核

实际上就是编写了一段内核代码，并动态插入到了内核（written kernel code and dynamically inserted it into the kernel）。

![image-20201031225041751](D:\学习资料\笔记\linux 性能\linux 性能笔记图片\image-20201031225041751.png)



图 2.1 中的程序使用了一种称为 map 的东西，这是一种特殊的数据结构，可用于 在内核和用户态之间传递数据，例如通过一个特殊的系统从用户态向 map 里插入数据。

这段程序的功能：丢弃所有源 IP 命中黑名单的 ARP 包。右侧四个框内的代码功能：

1. 初始化以太帧结构体（ethernet packet）。
2. 如果不是 ARP 包，直接退出，将包交给内核继续处理。
3. 至此已确定是 ARP，因此初始化一个 ARP 数据结构，对包进行下一步处理。例如，提取出 ARP 中的源 IP，去之前创建好的黑名单中查询该 IP 是否存在。
4. 如果存在，返回丢弃判决（`XDP_DROP`）；否则，返回允许通行判决（ `XDP_PASS`），内核会进行后续处理。

你可能不会相信，就这样一段简单的程序，会让服务器性能产生质的飞跃，因为它此时已经拥有了一条极为高效的网络路径（an extremely efficient network path）。



### 3. 为什么 eBPF 如此强大？

三方面原因：

1. 快速（fast）
2. 灵活（flexible）
3. 数据与功能分离（separates data from functionality）



#### 3.1 快速

eBPF 几乎总是比 iptables 快，这是有技术原因的。

- eBPF 程序本身并不比 iptables 快，但 eBPF 程序更短。
- iptables 基于一个非常庞大的内核框架（Netfilter），这个框架出现在内核 datapath 的多个地方，有很大冗余。

因此，同样是实现 ARP drop 这样的功能，基于 iptables 做冗余就会非常大，导致性能很低。



#### 3.2 灵活

这可能是最主要的原因。你可以用 eBPF 做几乎任何事情。

eBPF 基于内核提供的一组接口，运行 JIT 编译的字节码，并将计算结果返回给内核。例如内核只关心 XDP 程序的返回是 PASS, DROP 还是 REDIRECT。至于在 XDP 程序里做什么， 完全看你自己。



#### 3.3 数据与功能分离

eBPF separates data from functionality.

`nftables` 和 `iptables` 也能干这个事情，但功能没有 eBPF 强大。例如，eBPF 可以使 用 per-cpu 的数据结构，因此能取得更极致的性能。

eBPF 真正的优势是将“数据与功能分离”这件事情做地非常干净（clean separation）：可以在 eBPF 程序不中断的情况下修改它的运行方式。具体方式是修改它访问的配置数据或应用数据，例如黑名单里规定的 IP 列表和域名

![image-20201031225709937](D:\学习资料\笔记\linux 性能\linux 性能笔记图片\image-20201031225709937.png)



### 4. eBPF 简史

这里是简单介绍几句，后面 datapath 才是重点。

两篇论文，可读性还是比较好的，感兴趣的自行阅读：

- Steven McCanne, et al, in 1993 - The BSD Packet Filter
- Jeffrey C. Mogul, et al, in 1987 - first open source implementation of a packet filter.



### 5. Cilium 是什么，为什么要关注它？

我认为理解 eBPF 代码还比较简单，多看看内核代码就行了，但配置和编写 eBPF 就要难多了。

Cilium 是一个很好的 eBPF 之上的通用抽象，覆盖了分布式系统的绝大多数场景。

Cilium 封装了 eBPF，提供一个更上层的 API。如果你使用的是 Kubernetes，那你至少应该听说过 Cilium。

Cilium 提供了 CNI 和 kube-proxy replacement 功能，相比 iptables 性能要好很多。

接下来开始进入本文重点。



### 6. 内核默认 datapath

本节将介绍数据包是如何穿过 network datapath（网络数据路径）的：包括从硬件到内核，再到用户空间。

这里将只介绍 Cilium 所使用的 eBPF 程序，其中有 Cilium logo 的地方，都是 datapath 上 Cilium 重度使用 BPF 程序的地方。



#### 6.1 L1 -> L2（物理层 -> 数据链路层）

**网卡收包简要流程**：

![image-20201031230135461](D:\学习资料\笔记\linux 性能\linux 性能笔记图片\image-20201031230135461.png)



1.  网卡驱动初始化。
2.  网卡获得一块物理内存，作用收发包的缓冲区（ring-buffer）。这种方式称为 DMA（直接内存访问）。
3.  驱动向内核 NAPI（New API）注册一个轮询（poll ）方法。
4.  网卡从云上收到一个包，将包放到 ring-buffer。
5.  如果此时 NAPI 没有在执行，网卡就会触发一个硬件中断（HW IRQ），告诉处理器 DMA 区域中有包等待处理。
6.  收到硬中断信号后，处理器开始执行 NAPI。
7.  NAPI 执行网卡注册的 poll 方法开始收包。



**关于 NAPI poll 机制**：

- 这是 Linux 内核中的一种通用抽象，任何等待不可抢占状态发生（wait for a preemptible state to occur）的模块，都可以使用这种注册回调函数的方式。
- 驱动注册的这个 poll 是一个主动式 poll（active poll），一旦执行就会持续处理 ，直到没有数据可供处理，然后进入 idle 状态。
- 在这里，执行 poll 方法的是运行在某个或者所有 CPU 上的内核线程（kernel thread）。虽然这个线程没有数据可处理时会进入 idle 状态，但如前面讨论的，在当前大部分分布 式系统中，这个线程大部分时间内都是在运行的，不断从驱动的 DMA 区域内接收数据包。
- poll 会告诉网卡不要再触发硬件中断，使用软件中断（softirq）就行了。此后这些内核线程会轮询网卡的 DMA 区域来收包。之所以会有这种机制，是因为硬件中断代价太 高了，因为它们比系统上几乎所有东西的优先级都要高。



我们接下来还将多次看到这个广义的 NAPI 抽象，因为它不仅仅处理驱动，还能处理许多 其他场景。内核用 NAPI 抽象来做驱动读取（driver reads）、epoll 等等。

NAPI 驱动的 poll 机制将数据从 DMA 区域读取出来，对数据做一些准备工作，然后交给比它更上一层的内核协议栈。



#### 6.2 L2 续（数据链路层 - 续）

同样，这里不会深入展开驱动层做的事情，而主要关注内核所做的一些更上层的事情，例如

- 分配 socket buffers（skb）
- BPF
- iptables
- 将包送到网络栈（network stack）和用户空间



##### Step 1：NAPI poll

![image-20201031230915708](D:\学习资料\笔记\linux 性能\linux 性能笔记图片\image-20201031230915708.png)



首先，NAPI poll 机制不断调用驱动实现的 poll 方法，后者处理 RX 队列内的包，并最终 将包送到正确的程序。这就到了我们前面的 XDP 类型程序。



##### Step 2：XDP 程序处理

![image-20201031231156696](D:\学习资料\笔记\linux 性能\linux 性能笔记图片\image-20201031231156696.png)



如果驱动支持 XDP，那 XDP 程序将在 poll 机制内执行。如果不支持，那 XDP 程序将只能在更后面执行（run significantly upstack，见 **Step 6**），性能会变差， 因此确定你使用的网卡是否支持 XDP 非常重要。

XDP 程序返回一个判决结果给驱动，可以是 PASS, TRANSMIT, 或 DROP。

- Transmit 非常有用，有了这个功能，就可以用 XDP 实现一个 TCP/IP 负载均衡器。XDP只适合对包进行较小修改，如果是大动作修改，那这样的 XDP 程序的性能 可能并不会很高，因为这些操作会降低 poll 函数处理 DMA ring-buffer 的能力。
- 更有趣的是 DROP 方法，因为一旦判决为 DROP，这个包就可以直接原地丢弃了，而 无需再穿越后面复杂的协议栈然后再在某个地方被丢弃，从而节省了大量资源。如果本次 分享我只能给大家一个建议，那这个建议就是：在 datapath 越前面做 tuning 和 dropping 越好，这会显著增加系统的网络吞吐。
- 如果返回是 PASS，内核会继续沿着默认路径处理包，到达 `clean_rx()` 方法。



##### Step 3：`clean_rx()`：创建skb

![image-20201031231554892](D:\学习资料\笔记\linux 性能\linux 性能笔记图片\image-20201031231554892.png)



如果返回是 PASS，内核会继续沿着默认路径处理包，到达 `clean_rx()` 方法。

这个方法创建一个 socket buffer（skb）对象，可能还会更新一些统计信息，对 skb 进行硬件校验和检查，然后将其交给 `gro_receive()` 方法。



##### Step 4：`gro_receive()`

![image-20201031231706661](D:\学习资料\笔记\linux 性能\linux 性能笔记图片\image-20201031231706661.png)



GRO 是一种较老的硬件特性（LRO）的软件实现，功能是对分片的包进行重组然后交给更 上层，以提高吞吐。

GRO 给协议栈提供了一次将包交给网络协议栈之前，对其检查校验和 、修改协议头和发送应答包（ACK packets）的机会。

1. 如果 GRO 的 buffer 相比于包太小了，它可能会选择什么都不做。
2. 如果当前包属于某个更大包的一个分片，调用 `enqueue_backlog` 将这个分片放到某个 CPU 的包队列。当包重组完成后，会交给 `receive_skb()` 方法处理。
3. 如果当前包不是分片包，直接调用 `receive_skb()`，进行一些网络栈最底层的处理。



##### Step 5：`receive_skb()`

![image-20201102113725027](D:\学习资料\笔记\linux 性能\linux 性能笔记图片\image-20201102113725027.png)

`receive_skb()` 之后会再次进入 XDP 程序点。



#### 6.3 L2 -> L3（数据链路层 -> 网络层）

##### Step 6：通用 XDP 处理（gXDP）

![image-20201031232316003](D:\学习资料\笔记\linux 性能\linux 性能笔记图片\image-20201031232316003.png)



`receive_skb()` 之后，我们又来到了另一个 XDP 程序执行点。这里可以通过 `receive_xdp()` 做一些通用（generic）的事情，因此我在图中将其标注为 `(g)XDP`

Step 2 中提到，如果网卡驱动不支持 XDP，那 XDP 程序将延迟到更后面执行，这个 “更后面”的位置指的就是这里的 `(g)XDP`。



##### Step 7：Tap 设备处理

![image-20201031232515903](D:\学习资料\笔记\linux 性能\linux 性能笔记图片\image-20201031232515903.png)



图中有个 `*check_taps` 框，但其实并没有这个方法：`receive_skb()` 会轮询所有的 socket tap，将包放到正确的 tap 设备的缓冲区。

tap 设备监听的是三层协议（L3 protocols），例如 IPv4、ARP、IPv6 等等。如果 tap 设 备存在，它就可以操作这个 skb 了。



##### Step 8：`tc`（traffic classifier）处理

接下来我们遇到了第二种 eBPF 程序：tc eBPF。

![image-20201031232631804](D:\学习资料\笔记\linux 性能\linux 性能笔记图片\image-20201031232631804.png)



tc（traffic classifier，流量分类器）是 Cilium 依赖的最基础的东西，它提供了多种功 能，例如修改包（mangle，给 skb 打标记）、重路由（reroute）、丢弃包（drop），这 些操作都会影响到内核的流量统计，因此也影响着包的排队规则（queueing discipline ）。

Cilium 控制的网络设备，至少被加载了一个 tc eBPF 程序。



##### Step 9：Netfilter 处理

如果 tc BPF 返回 OK，包会再次进入 Netfilter。

![image-20201031232746102](D:\学习资料\笔记\linux 性能\linux 性能笔记图片\image-20201031232746102.png)



Netfilter 也会对入向的包进行处理，这里包括 `nftables` 和 `iptables` 模块。

有一点需要记住的是：Netfilter 是网络栈的下半部分（the “bottom half” of the network stack），因此 iptables 规则越多，给网络栈下半部分造成的瓶颈就越大。

`*def_dev_protocol` 框是二层过滤器（L2 net filter），由于 Cilium 没有用到任何 L2 filter，因此这里我就不展开了。



##### Step 10：L3 协议层处理：`ip_rcv()`

最后，如果包没有被前面丢弃，就会通过网络设备的 `ip_rcv()` 方法进入协议栈的三层（ L3）—— 即 IP 层 —— 进行处理。

![image-20201102102344000](D:\学习资料\笔记\linux 性能\linux 性能笔记图片\image-20201102102344000.png)

接下来我们将主要关注这个函数，但这里需要提醒大家的是，Linux 内核也支持除了 IP 之 外的其他三层协议，它们的 datapath 会与此有些不同。



#### 6.4 L3 -> L4（网络层 -> 传输层）

##### Step 11：Netfilter L4 处理

![image-20201102102712906](D:\学习资料\笔记\linux 性能\linux 性能笔记图片\image-20201102102712906.png)

`ip_rcv()` 做的第一件事情是再次执行 Netfilter 过滤，因为我们现在是从四层（L4）的 视角来处理 socket buffer。因此，这里会执行 Netfilter 中的任何四层规则（L4 rules ）。



##### Step 12：`ip_rcv_finish()` 处理

Netfilter 执行完成后，调用回调函数 `ip_rcv_finish()`。

![image-20201102102837895](D:\学习资料\笔记\linux 性能\linux 性能笔记图片\image-20201102102837895.png)

`ip_rcv_finish()` 立即调用 `ip_routing()` 对包进行路由判断。



##### Step 13：`ip_routing()` 处理

`ip_routing()` 对包进行路由判断，例如看它是否是在 lookback 设备上，是否能 路由出去（could egress），或者能否被路由，能否被 unmangle 到其他设备等等。

![image-20201102102947080](D:\学习资料\笔记\linux 性能\linux 性能笔记图片\image-20201102102947080.png)

在 Cilium 中，如果没有使用隧道模式（tunneling），那就会用到这里的路由功能。相比 隧道模式，路由模式会的 datapath 路径更短，因此性能更高。



##### Step 14：目的是本机：`ip_local_deliver()` 处理

根据路由判断的结果，如果包的目的端是本机，会调用 `ip_local_deliver()` 方法。

![image-20201102103102927](D:\学习资料\笔记\linux 性能\linux 性能笔记图片\image-20201102103102927.png)

`ip_local_deliver()` 会调用 `xfrm4_policy()`。



##### Step 15：`xfrm4_policy()` 处理

`xfrm4_policy()` 完成对包的封装、解封装、加解密等工作。例如，IPSec 就是在这里完成的。

![image-20201102103206347](D:\学习资料\笔记\linux 性能\linux 性能笔记图片\image-20201102103206347.png)

最后，根据四层协议的不同，`ip_local_deliver()` 会将最终的包送到 TCP 或 UDP 协议 栈。这里必须是这两种协议之一，否则设备会给源 IP 地址回一个 `ICMP destination unreachable` 消息。

接下来我将拿 UDP 协议作为例子，因为 TCP 状态机太复杂了，不适合这里用于理解 datapath 和数据流。但不是说 TCP 不重要，Linux TCP 状态机还是非常值得好好学习的。



#### 6.5 L4（传输层，以 UDP 为例）

##### Step 16：`udp_rcv()` 处理

![image-20201102103313582](D:\学习资料\笔记\linux 性能\linux 性能笔记图片\image-20201102103313582.png)

`udp_rcv()` 对包的合法性进行验证，检查 UDP 校验和。然后，再次将包送到 `xfrm4_policy()` 进行处理。



##### Step 17：`xfrm4_policy()` 再次处理

![image-20201102103610696](D:\学习资料\笔记\linux 性能\linux 性能笔记图片\image-20201102103610696.png)

这里再次对包执行 transform policies 是因为，某些规则能指定具体的四层协议，所以只 有到了协议层之后才能执行这些策略。



##### Step 18：将包放入 `socket_receive_queue`

这一步会拿端口（port）查找相应的 socket，然后将 skb 放到一个名为 `socket_receive_queue` 的链表。

![image-20201102115651350](D:\学习资料\笔记\linux 性能\linux 性能笔记图片\image-20201102115651350.png)



##### Step 19：通知 socket 收数据：`sk_data_ready()`

最后，`udp_rcv()` 调用 `sk_data_ready()` 方法，标记这个 socket 有数据待收。

![image-20201102103928537](D:\学习资料\笔记\linux 性能\linux 性能笔记图片\image-20201102103928537.png)

本质上，一个 socket 就是 Linux 中的一个文件描述符，这个描述符有一组相关的文件操作抽象，例如 `read`、`write` 等等。



#### 6.6 L4 - User Space

下图左边是一段 socket listening 程序，这里省略了错误检查，而且 `epoll` 本质上也 是不需要的，因为 UDP 的 recv 方法以及在帮我们 poll 了。

![image-20201102104130526](D:\学习资料\笔记\linux 性能\linux 性能笔记图片\image-20201102104130526.png)



由于大家还是对 TCP 熟悉一些，因此在这里我假设这是一段 TCP 代码。事实上当我们调用 `recvmsg()` 方法时，内核所做的事情就和上面这段代码差不多。对照右边的图：

1. 首先初始化一个 epoll 实例和一个 UDP socket，然后告诉 epoll 实例我们想监听这个 socket 上的 receive 事件，然后等着事件到来。

2. 当 socket buffer 收到数据时，其 wait queue 会被上一节的 `sk_data_ready()` 方法置位（标记）。

3. epoll 监听在 wait queue，因此 epoll 收到事件通知后，提取事件内容，返回给用户空间。

4. 用户空间程序调用 `recv` 方法，它接着调用 `udp_recv_msg` 方法，后者又会调用 cgroup eBPF 程序 —— 这是本文出现的第三种 BPF 程序。Cilium 利用 cgroup eBPF 实现 socket level 负载均衡，这非常酷：

5. - 一般的客户端负载均衡对客户端并不是透明的，即，客户端应用必须将负载均衡逻辑内置到应用里。
   - 有了 cgroup BPF，客户端根本感知不到负载均衡的存在。

6. 本文介绍的最后一种 BPF 程序是 sock_ops BPF，用于 socket level 整流（traffic shaping ），这对某些功能至关重要，例如客户端级别的限速（rate limiting）。

7. 最后，我们有一个用户空间缓冲区，存放收到的数据。

以上就是 Cilium 基于 eBPF 的内核收包之旅（traversing the kernel’s datapath）。



### 7 Kubernets、Cilium 和 Kernel：原子对象对应关系

| Kubernetes                             | Cilium                | Kernel                                                       |
| :------------------------------------- | :-------------------- | :----------------------------------------------------------- |
| Endpoint (includes Pods)               | Endpoint              | tc, cgroup socket BPF, sock_ops BPF, XDP                     |
| Network Policy                         | Cilium Network Policy | XDP, tc, sock-ops                                            |
| Service (node ports, cluster ips, etc) | Service               | XDP, tc                                                      |
| Node                                   | Node                  | ip-xfrm (for encryption), ip tables for initial decapsulation routing (if vxlan), veth-pair, ipvlan |

以上就是 Kubernetes 的所有网络对象（the only artificial network objects）。什么意思？这就是 k8s CNI 所依赖的全部网络原语（network primitives）。例如，LoadBalancer 对象只是 ClusterIP 和 NodePort 的组合，而后二者都属于 Service 对象，所以他们并不 是一等对象。

这张图非常有价值，但不幸的是，实际情况要比这里列出的更加复杂，因为 Cilium 本身的 实现是很复杂的。这有两个主要原因，我觉得值得拿出来讨论和体会：

首先，内核 datapath 要远比我这里讲的复杂。

1. 前面只是非常简单地介绍了协议栈每个位置（Netfilter、iptables、eBPF、XDP）能执行的动作。

2. 这些位置提供的处理能力是不同的。例如

3. 1. XDP 可能是能力最受限的，因为它只是设计用来做快速丢包（fast dropping）和 非本地重定向（non-local redirecting）；但另一方面，它又是最快的程序，因为 它在整个 datapath 的最前面，具备对整个 datapath 进行短路处理（short circuit the entire datapath）的能力。
   2. tc 和 iptables 程序能方便地 mangle 数据包，而不会对原来的转发流程产生显著影响。

理解这些东西非常重要，因为这是 Cilium 乃至广义 datapath 里非常核心的东西。如 果遇到底层网络问题，或者需要做 Cilium/kernel 调优，那你必须要理解包的收发/转发 路径，有时你会发现包的某些路径非常反直觉。

第二个原因是，eBPF 还非常新，某些最新特性只有在 5.x 内核中才有。尤其是 XDP BPF， 可能一个节点的内核版本支持，调度到另一台节点时，可能就不支持。









## 连接跟踪（conntrack）：原理、应用及 Linux 内核实现



### 1 引言

连接跟踪是许多网络应用的基础。例如，Kubernetes Service、ServiceMesh sidecar、 软件四层负载均衡器 LVS/IPVS、Docker network、OVS、iptables 主机防火墙等等，都依赖 连接跟踪功能。



#### 1.1 概念

##### 连接跟踪（conntrack）

![image-20201214084923310](D:\学习资料\笔记\linux 性能\linux 性能笔记图片\image-20201214084923310.png)

​                                                          *图 1.1. 连接跟踪及其内核位置*

连接跟踪，顾名思义，就是**跟踪（并记录）连接的状态**。

例如，图 1.1 是一台 IP 地址为 `10.1.1.2` 的 Linux 机器，我们能看到这台机器上有三条 连接：

1. 机器访问外部 HTTP 服务的连接（目的端口 80）
2. 外部访问机器内 FTP 服务的连接（目的端口 21）
3. 机器访问外部 DNS 服务的连接（目的端口 53）

连接跟踪所做的事情就是发现并跟踪这些连接的状态，具体包括：

- 从数据包中提取**元组**（tuple）信息，辨别**数据流**（flow）和对应的**连接**（connection）
- 为所有连接维护一个**状态数据库**（conntrack table），例如连接的创建时间、发送 包数、发送字节数等等
- 回收过期的连接（GC）
- 为更上层的功能（例如 NAT）提供服务

需要注意的是，**连接跟踪中所说的“连接”，概念和 TCP/IP 协议中“面向连接”（ connection oriented）的“连接”并不完全相同**，简单来说：

- TCP/IP 协议中，连接是一个四层（Layer 4）的概念。

- - TCP 是有连接的，或称面向连接的（connection oriented），发送出去的包都要求对端应答（ACK），并且有重传机制
  - UDP 是无连接的，发送的包无需对端应答，也没有重传机制

- CT 中，一个元组（tuple）定义的一条数据流（flow ）就表示一条连接（connection）。

- - 后面会看到 UDP 甚至是 **ICMP 这种三层协议在 CT 中也都是有连接记录的**
  - 但**不是所有协议都会被连接跟踪**

文中用到“连接”一词时，大部分情况下指的都是后者，即“连接跟踪”中的“连接”。



##### 网络地址转换（NAT）

![image-20201214090209044](D:\学习资料\笔记\linux 性能\linux 性能笔记图片\image-20201214090209044.png)

​                                                        *图 1.2. NAT 及其内核位置*

网络地址转换（NAT），意思也比较清楚：对（数据包的）网络地址（`IP + Port`）进行转换。

例如，图 1.2 中，机器自己的 IP `10.1.1.2` 是能与外部正常通信的，但 `192.168` 网段是私有 IP 段，外界无法访问，也就是说源 IP 地址是 `192.168` 的包，其**应答包是无 法回来的**。

因此当源地址为 `192.168` 网段的包要出去时，机器会先将源 IP 换成机器自己的 `10.1.1.2` 再发送出去；收到应答包时，再进行相反的转换。这就是 NAT 的基本过程。

Docker 默认的 `bridge` 网络模式就是这个原理 [4]。每个容器会分一个私有网段的 IP 地址，这个 IP 地址可以在宿主机内的不同容器之间通信，但容器流量出宿主机时要进行 NAT。

NAT 又可以细分为几类：

- SNAT：对源地址（source）进行转换
- DNAT：对目的地址（destination）进行转换
- Full NAT：同时对源地址和目的地址进行转换

以上场景属于 SNAT，将不同私有 IP 都映射成同一个“公有 IP”，以使其能访问外部网络服 务。这种场景也属于正向代理。

NAT 依赖连接跟踪的结果。连接跟踪**最重要的使用场景**就是 NAT。



##### 四层负载均衡（L4 LB）

![image-20201214090446067](D:\学习资料\笔记\linux 性能\linux 性能笔记图片\image-20201214090446067.png)

​                                            *图 1.3. L4LB: Traffic path in NAT mode [3]*

再将范围稍微延伸一点，讨论一下 NAT 模式的四层负载均衡。

四层负载均衡是根据包的四层信息（例如 `src/dst ip, src/dst port, proto`）做流量分发。

VIP（Virtual IP）是四层负载均衡的一种实现方式：

- 多个后端真实 IP（Real IP）挂到同一个虚拟 IP（VIP）上
- 客户端过来的流量先到达 VIP，再经负载均衡算法转发给某个特定的后端 IP

如果在 VIP 和 Real IP 节点之间使用的 NAT 技术（也可以使用其他技术），那客户端访 问服务端时，L4LB 节点将做双向 NAT（Full NAT），数据流如图 1.3。



#### 1.2 原理

了解以上概念之后，我们来思考下连接跟踪的技术原理。

要跟踪一台机器的所有连接状态，就需要

1. **拦截（或称过滤）流经这台机器的每一个数据包，并进行分析**。
2. 根据这些信息**建立**起这台机器上的**连接信息数据库**（conntrack table）。
3. 根据拦截到的包信息，不断更新数据库

例如，

1. 拦截到一个 TCP `SYNC` 包时，说明正在尝试建立 TCP 连接，需要创建一条新 conntrack entry 来记录这条连接
2. 拦截到一个属于已有 conntrack entry 的包时，需要更新这条 conntrack entry 的收发包数等统计信息

除了以上两点功能需求，还要考虑**性能问题**，因为连接跟踪要对每个包进行过滤和分析 。性能问题非常重要，但不是本文重点，后面介绍实现时会进一步提及。

之外，这些功能最好还有配套的管理工具来更方便地使用。



#### 1.3 设计：Netfilter

![image-20201214090800068](D:\学习资料\笔记\linux 性能\linux 性能笔记图片\image-20201214090800068.png)

​                                              *图 1.4. Netfilter architecture inside Linux kernel*

**Linux 的连接跟踪是在 Netfilter 中实现的。**

Netfilter 是 Linux 内核中一个对数据 包进行**控制、修改和过滤**（manipulation and filtering）的框架。它在内核协议 栈中设置了若干hook 点，以此对数据包进行拦截、过滤或其他处理。

> 说地更直白一些，hook 机制就是在数据包的必经之路上设置若干检测点，所有到达这些检测点的包都必须接受检测，根据检测的结果决定：
>
> 1. 放行：不对包进行任何修改，退出检测逻辑，继续后面正常的包处理
> 2. 修改：例如修改 IP 地址进行 NAT，然后将包放回正常的包处理逻辑
> 3. 丢弃：安全策略或防火墙功能
>
> 连接跟踪模块只是完成连接信息的采集和录入功能，并不会修改或丢弃数据包，后者是其 他模块（例如 NAT）基于 Netfilter hook 完成的。

Netfilter 是最古老的内核框架之一，1998 年开始开发，2000 年合并到 `2.4.x` 内 核主线版本 [5]。



#### 1.4 设计：进一步思考

现在提到连接跟踪（conntrack），可能首先都会想到 Netfilter。但由 1.2 节的讨论可知， 连接跟踪概念是独立于 Netfilter 的，**Netfilter 只是 Linux 内核中的一种连接跟踪实现**。

换句话说，**只要具备了 hook 能力，能拦截到进出主机的每个包，完全可以在此基础上自 己实现一套连接跟踪**。

![image-20201214091120196](D:\学习资料\笔记\linux 性能\linux 性能笔记图片\image-20201214091120196.png)

​                                         *图 1.5. Cilium's conntrack and NAT architectrue*

云原生网络方案 Cilium 在 `1.7.4+` 版本就实现了这样一套独立的连接跟踪和 NAT 机制 （完备功能需要 Kernel `4.19+`）。其基本原理是：

1. 基于 BPF hook 实现数据包的拦截功能（等价于 netfilter 里面的 hook 机制）
2. 在 BPF hook 的基础上，实现一套全新的 conntrack 和 NAT

因此，即便卸载掉 Netfilter ，也不会影响 Cilium 对 Kubernetes ClusterIP、NodePort、ExternalIPs 和 LoadBalancer 等功能的支持 [2]。

由于这套连接跟踪机制是独立于 Netfilter 的，因此它的 conntrack 和 NAT 信息也没有存储在内核的（也就是 Netfilter 的）conntrack table 和 NAT table。所以常规的 `conntrack/netstats/ss/lsof` 等工具是看不到的，要使用 Cilium 的命令，例如：

```sh
$ cilium bpf nat list
$ cilium bpf ct list global
```

配置也是独立的，需要在 Cilium 里面配置，例如命令行选项 `--bpf-ct-tcp-max`。

另外，本文会多次提到连接跟踪模块和 NAT 模块独立，但出于性能考虑，具体实现中 二者代码可能是有耦合的。例如 Cilium 做 conntrack 的垃圾回收（GC）时就会顺便把 NAT 里相应的 entry 回收掉，而非为 NAT 做单独的 GC。

以上是理论篇，接下来看一下内核实现。



### 2 Netfilter hook 机制实现

Netfilter 由几个模块构成，其中最主要的是**连接跟踪**（CT） 模块和**网络地址转换**（NAT）模块。

CT 模块的主要职责是识别出可进行连接跟踪的包。CT 模块独立于 NAT 模块，但主要目的是服务于后者。



#### 2.1 Netfilter 框架

##### 5 个 hook 点                         

![image-20201214091838806](D:\学习资料\笔记\k8s\k8s图\image-20201214091838806.png)

​                                             *图 2.1. The 5 hook points in netfilter framework*

如上图所示，Netfilter 在内核协议栈的包处理路径上提供了 5 个 hook 点，分别是：

```c
// include/uapi/linux/netfilter_ipv4.h

#define NF_IP_PRE_ROUTING    0 /* After promisc drops, checksum checks. */
#define NF_IP_LOCAL_IN       1 /* If the packet is destined for this box. */
#define NF_IP_FORWARD        2 /* If the packet is destined for another interface. */
#define NF_IP_LOCAL_OUT      3 /* Packets coming from a local process. */
#define NF_IP_POST_ROUTING   4 /* Packets about to hit the wire. */
#define NF_IP_NUMHOOKS       5
```

用户可以在这些 hook 点注册自己的处理函数（handlers）。当有数据包经过 hook 点时， 就会调用相应的 handlers。

> 另外还有一套 `NF_INET_` 开头的定义，`include/uapi/linux/netfilter.h`。这两套是等价的，从注释看，`NF_IP_` 开头的定义可能是为了保持兼容性。

```c
enum nf_inet_hooks {
    NF_INET_PRE_ROUTING,
    NF_INET_LOCAL_IN,
    NF_INET_FORWARD,
    NF_INET_LOCAL_OUT,
    NF_INET_POST_ROUTING,
    NF_INET_NUMHOOKS
};
```



##### hook 返回值类型

hook 函数对包进行判断或处理之后，需要返回一个判断结果，指导接下来要对这个包做什 么。可能的结果有：

```c
// include/uapi/linux/netfilter.h

#define NF_DROP   0  // 已丢弃这个包
#define NF_ACCEPT 1  // 接受这个包，继续下一步处理
#define NF_STOLEN 2  // 当前处理函数已经消费了这个包，后面的处理函数不用处理了
#define NF_QUEUE  3  // 应当将包放到队列
#define NF_REPEAT 4  // 当前处理函数应当被再次调用
```



##### hook 优先级

每个 hook 点可以注册多个处理函数（handler）。在注册时必须指定这些 handlers 的**优先级**，这样触发 hook 时能够根据优先级依次调用处理函数。



#### 2.2 过滤规则的组织

`iptables` 是配置 Netfilter 过滤功能的用户空间工具。为便于管理， 过滤规则按功能分为若干 table：

- raw
- filter
- nat
- mangle

这不是本文重点。更多信息可参考 (译) 深入理解 iptables 和 netfilter 架构



### 3 Netfilter conntrack 实现

连接跟踪模块用于维护**可跟踪协议**（trackable protocols）的连接状态。也就是说， 连接跟踪**针对的是特定协议的包，而不是所有协议的包**。稍后会看到它支持哪些协议。



#### 3.1 重要结构体和函数

重要结构体：

```c
struct nf_conntrack_tuple {}
```

- : 定义一个 tuple。

- - `struct nf_conntrack_man_proto {}`：manipulable part 中协议相关的部分。

    ```c
    struct nf_conntrack_man {}
    ```

- - ：tuple 的 manipulable part。

- `struct nf_conntrack_l4proto {}`: 支持连接跟踪的**协议需要实现的方法集**（以及其他协议相关字段）。

- `struct nf_conntrack_tuple_hash {}`：哈希表（conntrack table）中的表项（entry）。

- `struct nf_conn {}`：定义一个 flow。

重要函数：

- `hash_conntrack_raw()`：根据 tuple 计算出一个 32 位的哈希值（hash key）。
- `nf_conntrack_in()`：**连接跟踪模块的核心，包进入连接跟踪的地方**。
- `resolve_normal_ct() -> init_conntrack() -> l4proto->new()`：创建一个新的连接记录（conntrack entry）。
- `nf_conntrack_confirm()`：确认前面通过 `nf_conntrack_in()` 创建的新连接。



#### 3.2 `struct nf_conntrack_tuple {}`：元组（Tuple）

Tuple 是连接跟踪中最重要的概念之一。

**一个 tuple 定义一个单向（unidirectional）flow**。内核代码中有如下注释：

> //include/net/netfilter/nf_conntrack_tuple.h
>
> A `tuple` is a structure containing the information to uniquely identify a connection. ie. if two packets have the same tuple, they are in the same connection; if not, they are not.



##### 结构体定义

```c
//include/net/netfilter/nf_conntrack_tuple.h

// 为方便 NAT 的实现，内核将 tuple 结构体拆分为 "manipulatable" 和 "non-manipulatable" 两部分
// 下面结构体中的 _man 是 manipulatable 的缩写
                                               // ude/uapi/linux/netfilter.h
                                               union nf_inet_addr {
                                                   __u32            all[4];
                                                   __be32           ip;
                                                   __be32           ip6[4];
                                                   struct in_addr   in;
                                                   struct in6_addr  in6;
/* manipulable part of the tuple */         /  };
struct nf_conntrack_man {                  /
    union nf_inet_addr           u3; -->--/
    union nf_conntrack_man_proto u;  -->--\
                                           \   // include/uapi/linux/netfilter/nf_conntrack_tuple_common.h
    u_int16_t l3num; // L3 proto            \  // 协议相关的部分
};                                            union nf_conntrack_man_proto {
                                                  __be16 all;/* Add other protocols here. */

                                                  struct { __be16 port; } tcp;
                                                  struct { __be16 port; } udp;
                                                  struct { __be16 id;   } icmp;
                                                  struct { __be16 port; } dccp;
                                                  struct { __be16 port; } sctp;
                                                  struct { __be16 key;  } gre;
                                              };

struct nf_conntrack_tuple { /* This contains the information to distinguish a connection. */
    struct nf_conntrack_man src;  // 源地址信息，manipulable part
    struct {
        union nf_inet_addr u3;
        union {
            __be16 all; /* Add other protocols here. */

            struct { __be16 port;         } tcp;
            struct { __be16 port;         } udp;
            struct { u_int8_t type, code; } icmp;
            struct { __be16 port;         } dccp;
            struct { __be16 port;         } sctp;
            struct { __be16 key;          } gre;
        } u;
        u_int8_t protonum; /* The protocol. */
        u_int8_t dir;      /* The direction (for tuplehash) */
    } dst;                       // 目的地址信息
};
```

Tuple 结构体中只有两个字段 `src` 和 `dst`，分别保存源和目的信息。`src` 和 `dst` 自身也是结构体，能保存不同类型协议的数据。以 IPv4 UDP 为例，五元组分别保存在如下字段：

- `dst.protonum`：协议类型
- `src.u3.ip`：源 IP 地址
- `dst.u3.ip`：目的 IP 地址
- `src.u.udp.port`：源端口号
- `dst.u.udp.port`：目的端口号



##### CT 支持的协议

从以上定义可以看到，连接跟踪模块**目前只支持以下六种协议**：TCP、UDP、ICMP、DCCP、SCTP、GRE。

**注意其中的 ICMP 协议**。大家可能会认为，连接跟踪模块依据包的三层和四层信息做 哈希，而 ICMP 是三层协议，没有四层信息，因此 ICMP 肯定不会被 CT 记录。但**实际上 是会的**，上面代码可以看到，ICMP 使用了其头信息中的 ICMP `type`和 `code` 字段来 定义 tuple。



#### 3.3 `struct nf_conntrack_l4proto {}`：协议需要实现的方法集合

支持连接跟踪的协议都需要实现 `struct nf_conntrack_l4proto {}` 结构体 中定义的方法，例如 `pkt_to_tuple()`。

```c
// include/net/netfilter/nf_conntrack_l4proto.h

struct nf_conntrack_l4proto {
    u_int16_t l3proto; /* L3 Protocol number. */
    u_int8_t  l4proto; /* L4 Protocol number. */

    // 从包（skb）中提取 tuple
    bool (*pkt_to_tuple)(struct sk_buff *skb, ... struct nf_conntrack_tuple *tuple);

    // 对包进行判决，返回判决结果（returns verdict for packet）
    int (*packet)(struct nf_conn *ct, const struct sk_buff *skb ...);

    // 创建一个新连接。如果成功返回 TRUE；如果返回的是 TRUE，接下来会调用 packet() 方法
    bool (*new)(struct nf_conn *ct, const struct sk_buff *skb, unsigned int dataoff);

    // 判断当前数据包能否被连接跟踪。如果返回成功，接下来会调用 packet() 方法
    int (*error)(struct net *net, struct nf_conn *tmpl, struct sk_buff *skb, ...);

    ...
```



#### 3.4 `struct nf_conntrack_tuple_hash {}`：哈希表项

conntrack 将活动连接的状态存储在一张哈希表中（`key: value`）。

`hash_conntrack_raw()` 根据 tuple 计算出一个 32 位的哈希值（key）：

```c
// net/netfilter/nf_conntrack_core.c

static u32 hash_conntrack_raw(struct nf_conntrack_tuple *tuple, struct net *net)
{
    get_random_once(&nf_conntrack_hash_rnd, sizeof(nf_conntrack_hash_rnd));

    /* The direction must be ignored, so we hash everything up to the
     * destination ports (which is a multiple of 4) and treat the last three bytes manually.  */
    u32 seed = nf_conntrack_hash_rnd ^ net_hash_mix(net);
    unsigned int n = (sizeof(tuple->src) + sizeof(tuple->dst.u3)) / sizeof(u32);

    return jhash2((u32 *)tuple, n, seed ^ ((tuple->dst.u.all << 16) | tuple->dst.protonum));
}
```

注意其中是如何利用 tuple 的不同字段来计算哈希的。

`nf_conntrack_tuple_hash` 是哈希表中的表项（value）:

```c
// include/net/netfilter/nf_conntrack_tuple.h

// 每条连接在哈希表中都对应两项，分别对应两个方向（egress/ingress）
// Connections have two entries in the hash table: one for each way
struct nf_conntrack_tuple_hash {
    struct hlist_nulls_node   hnnode;   // 指向该哈希对应的连接 struct nf_conn，采用 list 形式是为了解决哈希冲突
    struct nf_conntrack_tuple tuple;    // N 元组，前面详细介绍过了
};
```



#### 3.5 `struct nf_conn {}`：连接（connection）

Netfilter 中每个 flow 都称为一个 connection，即使是对那些非面向连接的协议（例 如 UDP）。每个 connection 用 `struct nf_conn {}` 表示，主要字段如下：

```c
// include/net/netfilter/nf_conntrack.h

                                                  // include/linux/skbuff.h
                                        ------>   struct nf_conntrack {
                                        |             atomic_t use;  // 连接引用计数？
                                        |         };
struct nf_conn {                        |
    struct nf_conntrack            ct_general;

    struct nf_conntrack_tuple_hash tuplehash[IP_CT_DIR_MAX]; // 哈希表项，数组是因为要记录两个方向的 flow

    unsigned long status; // 连接状态，见下文
    u32 timeout;          // 连接状态的定时器

    possible_net_t ct_net;

    struct hlist_node    nat_bysource;
                                                        // per conntrack: protocol private data
    struct nf_conn *master;                             union nf_conntrack_proto {
                                                            /* insert conntrack proto private data here */
    u_int32_t mark;    /* 对 skb 进行特殊标记 */            struct nf_ct_dccp dccp;
    u_int32_t secmark;                                      struct ip_ct_sctp sctp;
                                                            struct ip_ct_tcp tcp;
    union nf_conntrack_proto proto; ---------->----->       struct nf_ct_gre gre;
};                                                          unsigned int tmpl_padto;
                                                        };
```

连接的状态集合 `enum ip_conntrack_status`：

```c
// include/uapi/linux/netfilter/nf_conntrack_common.h

enum ip_conntrack_status {
    IPS_EXPECTED      = (1 << IPS_EXPECTED_BIT),
    IPS_SEEN_REPLY    = (1 << IPS_SEEN_REPLY_BIT),
    IPS_ASSURED       = (1 << IPS_ASSURED_BIT),
    IPS_CONFIRMED     = (1 << IPS_CONFIRMED_BIT),
    IPS_SRC_NAT       = (1 << IPS_SRC_NAT_BIT),
    IPS_DST_NAT       = (1 << IPS_DST_NAT_BIT),
    IPS_NAT_MASK      = (IPS_DST_NAT | IPS_SRC_NAT),
    IPS_SEQ_ADJUST    = (1 << IPS_SEQ_ADJUST_BIT),
    IPS_SRC_NAT_DONE  = (1 << IPS_SRC_NAT_DONE_BIT),
    IPS_DST_NAT_DONE  = (1 << IPS_DST_NAT_DONE_BIT),
    IPS_NAT_DONE_MASK = (IPS_DST_NAT_DONE | IPS_SRC_NAT_DONE),
    IPS_DYING         = (1 << IPS_DYING_BIT),
    IPS_FIXED_TIMEOUT = (1 << IPS_FIXED_TIMEOUT_BIT),
    IPS_TEMPLATE      = (1 << IPS_TEMPLATE_BIT),
    IPS_UNTRACKED     = (1 << IPS_UNTRACKED_BIT),
    IPS_HELPER        = (1 << IPS_HELPER_BIT),
    IPS_OFFLOAD       = (1 << IPS_OFFLOAD_BIT),

    IPS_UNCHANGEABLE_MASK = (IPS_NAT_DONE_MASK | IPS_NAT_MASK |
                 IPS_EXPECTED | IPS_CONFIRMED | IPS_DYING |
                 IPS_SEQ_ADJUST | IPS_TEMPLATE | IPS_OFFLOAD),
};
```



#### 3.6 `nf_conntrack_in()`：进入连接跟踪

![image-20201214113943434](D:\学习资料\笔记\k8s\k8s图\image-20201214113943434.png)

如上图所示，Netfilter 在四个 Hook 点对包进行跟踪：

1. `PRE_ROUTING` 和 `LOCAL_OUT`：调用 `nf_conntrack_in()` 开始连接跟踪，正常情况 下会创建一条新连接记录，然后将 conntrack entry 放到 **unconfirmed list**。

   为什么是这两个 hook 点呢？因为它们都是**新连接的第一个包最先达到的地方**，

2. - `PRE_ROUTING` 是**外部主动和本机建连**时包最先到达的地方
   - `LOCAL_OUT` 是**本机主动和外部建连**时包最先到达的地方

3. `POST_ROUTING` 和 `LOCAL_IN`：调用 `nf_conntrack_confirm()` 将 `nf_conntrack_in()` 创建的连接移到 **confirmed list**。

   同样要问，为什么在这两个 hook 点呢？因为如果新连接的第一个包没有被丢弃，那这是它们**离开 netfilter 之前的最后 hook 点**：

4. - **外部主动和本机建连**的包，如果在中间处理中没有被丢弃，`LOCAL_IN` 是其被送到应用（例如 nginx 服务）之前的最后 hook 点
   - **本机主动和外部建连**的包，如果在中间处理中没有被丢弃，`POST_ROUTING` 是其离开主机时的最后 hook 点

下面的代码可以看到这些 handler 是如何注册的：

```c
// net/netfilter/nf_conntrack_proto.c

/* Connection tracking may drop packets, but never alters them, so make it the first hook.  */
static const struct nf_hook_ops ipv4_conntrack_ops[] = {
 {
  .hook  = ipv4_conntrack_in,       // 调用 nf_conntrack_in() 进入连接跟踪
  .pf  = NFPROTO_IPV4,
  .hooknum = NF_INET_PRE_ROUTING,     // PRE_ROUTING hook 点
  .priority = NF_IP_PRI_CONNTRACK,
 },
 {
  .hook  = ipv4_conntrack_local,    // 调用 nf_conntrack_in() 进入连接跟踪
  .pf  = NFPROTO_IPV4,
  .hooknum = NF_INET_LOCAL_OUT,       // LOCAL_OUT hook 点
  .priority = NF_IP_PRI_CONNTRACK,
 },
 {
  .hook  = ipv4_confirm,            // 调用 nf_conntrack_confirm()
  .pf  = NFPROTO_IPV4,
  .hooknum = NF_INET_POST_ROUTING,    // POST_ROUTING hook 点
  .priority = NF_IP_PRI_CONNTRACK_CONFIRM,
 },
 {
  .hook  = ipv4_confirm,            // 调用 nf_conntrack_confirm()
  .pf  = NFPROTO_IPV4,
  .hooknum = NF_INET_LOCAL_IN,        // LOCAL_IN hook 点
  .priority = NF_IP_PRI_CONNTRACK_CONFIRM,
 },
};
```

**nf_conntrack_in 函数是连接跟踪模块的核心**。

```c
// net/netfilter/nf_conntrack_core.c

unsigned int
nf_conntrack_in(struct net *net, u_int8_t pf, unsigned int hooknum, struct sk_buff *skb)
{
  struct nf_conn *tmpl = nf_ct_get(skb, &ctinfo); // 获取 skb 对应的 conntrack_info 和连接记录
  if (tmpl || ctinfo == IP_CT_UNTRACKED) {        // 如果记录存在，或者是不需要跟踪的类型
      if ((tmpl && !nf_ct_is_template(tmpl)) || ctinfo == IP_CT_UNTRACKED) {
          NF_CT_STAT_INC_ATOMIC(net, ignore);     // 无需跟踪的类型，增加 ignore 计数
          return NF_ACCEPT;                       // 返回 NF_ACCEPT，继续后面的处理
      }
      skb->_nfct = 0;                             // 不属于 ignore 类型，计数器置零，准备后续处理
  }

  struct nf_conntrack_l4proto *l4proto = __nf_ct_l4proto_find(...);    // 提取协议相关的 L4 头信息

  if (l4proto->error != NULL) {                   // skb 的完整性和合法性验证
      if (l4proto->error(net, tmpl, skb, dataoff, pf, hooknum) <= 0) {
          NF_CT_STAT_INC_ATOMIC(net, error);
          NF_CT_STAT_INC_ATOMIC(net, invalid);
          goto out;
      }
  }

repeat:
  // 开始连接跟踪：提取 tuple；创建新连接记录，或者更新已有连接的状态
  resolve_normal_ct(net, tmpl, skb, ... l4proto);

  l4proto->packet(ct, skb, dataoff, ctinfo); // 进行一些协议相关的处理，例如 UDP 会更新 timeout

  if (ctinfo == IP_CT_ESTABLISHED_REPLY && !test_and_set_bit(IPS_SEEN_REPLY_BIT, &ct->status))
      nf_conntrack_event_cache(IPCT_REPLY, ct);
out:
  if (tmpl)
      nf_ct_put(tmpl); // 解除对连接记录 tmpl 的引用
}
```

大致流程：

1. 尝试获取这个 skb 对应的连接跟踪记录
2. 判断是否需要对这个包做连接跟踪，如果不需要，更新 ignore 计数，返回 `NF_ACCEPT`；如果需要，就**初始化这个 skb 的引用计数**。
3. 从包的 L4 header 中提取信息，初始化协议相关的 `struct nf_conntrack_l4proto {}` 变量，其中包含了该协议的**连接跟踪相关的回调方法**。
4. 调用该协议的 `error()` 方法检查包的完整性、校验和等信息。
5. 调用 `resolve_normal_ct()` **开始连接跟踪**，它会创建新 tuple，新 conntrack entry，或者更新已有连接的状态。
6. 调用该协议的 `packet()` 方法进行一些协议相关的处理，例如对于 UDP，如果 status bit 里面设置了 `IPS_SEEN_REPLY` 位，就会更新 timeout。timeout 大小和协 议相关，越小越越可以防止 DoS 攻击（DoS 的基本原理就是将机器的可用连接耗尽）





#### 3.7 `init_conntrack()`：创建新连接记录

如果连接不存在（flow 的第一个包），`resolve_normal_ct()` 会调用 `init_conntrack` ，后者进而会调用 `new()` 方法创建一个新的 conntrack entry。

```c
// include/net/netfilter/nf_conntrack_core.c

// Allocate a new conntrack
static noinline struct nf_conntrack_tuple_hash *
init_conntrack(struct net *net, struct nf_conn *tmpl,
        const struct nf_conntrack_tuple *tuple,
        const struct nf_conntrack_l4proto *l4proto,
        struct sk_buff *skb, unsigned int dataoff, u32 hash)
{
 struct nf_conn *ct;
 ct = __nf_conntrack_alloc(net, zone, tuple, &repl_tuple, GFP_ATOMIC, hash);

 l4proto->new(ct, skb, dataoff); // 协议相关的方法

 local_bh_disable();             // 关闭软中断

 if (net->ct.expect_count) {
  exp = nf_ct_find_expectation(net, zone, tuple);
  if (exp) {
   /* Welcome, Mr. Bond.  We've been expecting you... */
   __set_bit(IPS_EXPECTED_BIT, &ct->status);

   /* exp->master safe, refcnt bumped in nf_ct_find_expectation */
   ct->master = exp->master;

   ct->mark = exp->master->mark;
   ct->secmark = exp->master->secmark;
   NF_CT_STAT_INC(net, expect_new);
  }
 }

 /* Now it is inserted into the unconfirmed list, bump refcount */
 nf_conntrack_get(&ct->ct_general);
 nf_ct_add_to_unconfirmed_list(ct);

 local_bh_enable();              // 重新打开软中断

 if (exp) {
  if (exp->expectfn)
   exp->expectfn(ct, exp);
  nf_ct_expect_put(exp);
 }

 return &ct->tuplehash[IP_CT_DIR_ORIGINAL];
}
```

每种协议需要实现自己的 `l4proto->new()` 方法，代码见：`net/netfilter/nf_conntrack_proto_*.c`。

如果当前包会影响后面包的状态判断，`init_conntrack()` 会设置 `struct nf_conn` 的 `master` 字段。面向连接的协议会用到这个特性，例如 TCP。



#### 3.8 `nf_conntrack_confirm()`：确认包没有被丢弃

`nf_conntrack_in()` 创建的新 conntrack entry 会插入到一个 **未确认连接**（ unconfirmed connection）列表。

如果这个包之后没有被丢弃，那它在经过 `POST_ROUTING` 时会被 `nf_conntrack_confirm()` 方法处理，原理我们在分析过了 3.6 节的开头分析过了。`nf_conntrack_confirm()` 完成之后，状态就变为了 `IPS_CONFIRMED`，并且连接记录从 **未确认列表**移到**正常**的列表。

之所以要将创建一个合法的新 entry 的过程分为创建（new）和确认（confirm）两个阶段 ，是因为**包在经过 nf_conntrack_in() 之后，到达 nf_conntrack_confirm() 之前 ，可能会被内核丢弃**。这样会导致系统残留大量的半连接状态记录，在性能和安全性上都 是很大问题。分为两步之后，可以加快半连接状态 conntrack entry 的 GC。

```c
// include/net/netfilter/nf_conntrack_core.h

/* Confirm a connection: returns NF_DROP if packet must be dropped. */
static inline int nf_conntrack_confirm(struct sk_buff *skb)
{
 struct nf_conn *ct = (struct nf_conn *)skb_nfct(skb);
 int ret = NF_ACCEPT;

 if (ct) {
  if (!nf_ct_is_confirmed(ct))
   ret = __nf_conntrack_confirm(skb);
  if (likely(ret == NF_ACCEPT))
   nf_ct_deliver_cached_events(ct);
 }
 return ret;
}
```

confirm 逻辑，省略了各种错误处理逻辑：

```c
// net/netfilter/nf_conntrack_core.c

/* Confirm a connection given skb; places it in hash table */
int
__nf_conntrack_confirm(struct sk_buff *skb)
{
 struct nf_conn *ct;
 ct = nf_ct_get(skb, &ctinfo);

 local_bh_disable();               // 关闭软中断

 hash = *(unsigned long *)&ct->tuplehash[IP_CT_DIR_REPLY].hnnode.pprev;
 reply_hash = hash_conntrack(net, &ct->tuplehash[IP_CT_DIR_REPLY].tuple);

 ct->timeout += nfct_time_stamp;   // 更新连接超时时间，超时后会被 GC
 atomic_inc(&ct->ct_general.use);  // 设置连接引用计数？
 ct->status |= IPS_CONFIRMED;      // 设置连接状态为 confirmed

 __nf_conntrack_hash_insert(ct, hash, reply_hash);  // 插入到连接跟踪哈希表

 local_bh_enable();                // 重新打开软中断

 nf_conntrack_event_cache(master_ct(ct) ? IPCT_RELATED : IPCT_NEW, ct);
 return NF_ACCEPT;
}
```

可以看到，连接跟踪的处理逻辑中需要频繁关闭和打开软中断，此外还有各种锁， 这是短连高并发场景下连接跟踪性能损耗的主要原因？。



### 4 Netfilter NAT 实现

NAT 是与连接跟踪独立的模块。



#### 4.1 重要数据结构和函数

**重要数据结构：**

支持 NAT 的协议需要实现其中的方法：

- `struct nf_nat_l3proto {}`
- `struct nf_nat_l4proto {}`

**重要函数：**

- `nf_nat_inet_fn()`：NAT 的核心函数是，在**除 NF_INET_FORWARD 之外的其他 hook 点都会被调用**。



#### 4.2 NAT 模块初始化

```c
// net/netfilter/nf_nat_core.c

static struct nf_nat_hook nat_hook = {
 .parse_nat_setup = nfnetlink_parse_nat_setup,
 .decode_session  = __nf_nat_decode_session,
 .manip_pkt  = nf_nat_manip_pkt,
};

static int __init nf_nat_init(void)
{
 nf_nat_bysource = nf_ct_alloc_hashtable(&nf_nat_htable_size, 0);

 nf_ct_helper_expectfn_register(&follow_master_nat);

 RCU_INIT_POINTER(nf_nat_hook, &nat_hook);
}

MODULE_LICENSE("GPL");

module_init(nf_nat_init);
```



#### 4.3 `struct nf_nat_l3proto {}`：协议相关的 NAT 方法集

```c
// include/net/netfilter/nf_nat_l3proto.h

struct nf_nat_l3proto {
    u8    l3proto; // 例如，AF_INET

    u32     (*secure_port    )(const struct nf_conntrack_tuple *t, __be16);
    bool    (*manip_pkt      )(struct sk_buff *skb, ...);
    void    (*csum_update    )(struct sk_buff *skb, ...);
    void    (*csum_recalc    )(struct sk_buff *skb, u8 proto, ...);
    void    (*decode_session )(struct sk_buff *skb, ...);
    int     (*nlattr_to_range)(struct nlattr *tb[], struct nf_nat_range2 *range);
};
```



#### 4.4 `struct nf_nat_l4proto {}`：协议相关的 NAT 方法集

```c
// include/net/netfilter/nf_nat_l4proto.h

struct nf_nat_l4proto {
    u8 l4proto; // Protocol number，例如 IPPROTO_UDP, IPPROTO_TCP

    // 根据传入的 tuple 和 NAT 类型（SNAT/DNAT）修改包的 L3/L4 头
    bool (*manip_pkt)(struct sk_buff *skb, *l3proto, *tuple, maniptype);

    // 创建一个唯一的 tuple
    // 例如对于 UDP，会根据 src_ip, dst_ip, src_port 加一个随机数生成一个 16bit 的 dst_port
    void (*unique_tuple)(*l3proto, tuple, struct nf_nat_range2 *range, maniptype, struct nf_conn *ct);

    // If the address range is exhausted the NAT modules will begin to drop packets.
    int (*nlattr_to_range)(struct nlattr *tb[], struct nf_nat_range2 *range);
};
```

各协议实现的方法，见：`net/netfilter/nf_nat_proto_*.c`。例如 TCP 的实现：

```c
// net/netfilter/nf_nat_proto_tcp.c

const struct nf_nat_l4proto nf_nat_l4proto_tcp = {
 .l4proto  = IPPROTO_TCP,
 .manip_pkt  = tcp_manip_pkt,
 .in_range  = nf_nat_l4proto_in_range,
 .unique_tuple  = tcp_unique_tuple,
 .nlattr_to_range = nf_nat_l4proto_nlattr_to_range,
};
```



#### 4.5`nf_nat_inet_fn()`：进入 NAT

NAT 的核心函数是 `nf_nat_inet_fn()`，它会在以下 hook 点被调用：

- `NF_INET_PRE_ROUTING`
- `NF_INET_POST_ROUTING`
- `NF_INET_LOCAL_OUT`
- `NF_INET_LOCAL_IN`

也就是除了 `NF_INET_FORWARD` 之外其他 hook 点都会被调用。

在这些 hook 点的优先级：**Conntrack > NAT > Packet Filtering**。**连接跟踪的优先 级高于 NAT 是因为 NAT 依赖连接跟踪的结果**。

![image-20201214114616355](D:\学习资料\笔记\k8s\k8s图\image-20201214114616355.png)



```c
unsigned int
nf_nat_inet_fn(void *priv, struct sk_buff *skb, const struct nf_hook_state *state)
{
    ct = nf_ct_get(skb, &ctinfo);
    if (!ct)    // conntrack 不存在就做不了 NAT，直接返回，这也是为什么说 NAT 依赖 conntrack 的结果
        return NF_ACCEPT;

    nat = nfct_nat(ct);

    switch (ctinfo) {
    case IP_CT_RELATED:
    case IP_CT_RELATED_REPLY: /* Only ICMPs can be IP_CT_IS_REPLY.  Fallthrough */
    case IP_CT_NEW: /* Seen it before? This can happen for loopback, retrans, or local packets. */
        if (!nf_nat_initialized(ct, maniptype)) {
            struct nf_hook_entries *e = rcu_dereference(lpriv->entries); // 获取所有 NAT 规则
            if (!e)
                goto null_bind;

            for (i = 0; i < e->num_hook_entries; i++) { // 依次执行 NAT 规则
                if (e->hooks[i].hook(e->hooks[i].priv, skb, state) != NF_ACCEPT )
                    return ret;                         // 任何规则返回非 NF_ACCEPT，就停止当前处理

                if (nf_nat_initialized(ct, maniptype))
                    goto do_nat;
            }
null_bind:
            nf_nat_alloc_null_binding(ct, state->hook);
        } else { // Already setup manip
            if (nf_nat_oif_changed(state->hook, ctinfo, nat, state->out))
                goto oif_changed;
        }
        break;
    default: /* ESTABLISHED */
        if (nf_nat_oif_changed(state->hook, ctinfo, nat, state->out))
            goto oif_changed;
    }
do_nat:
    return nf_nat_packet(ct, ctinfo, state->hook, skb);
oif_changed:
    nf_ct_kill_acct(ct, ctinfo, skb);
    return NF_DROP;
}
```

首先查询 conntrack 记录，如果不存在，就意味着无法跟踪这个连接，那就更不可能做 NAT 了，因此直接返回。

如果找到了 conntrack 记录，并且是 `IP_CT_RELATED`、`IP_CT_RELATED_REPLY` 或 `IP_CT_NEW` 状态，就去获取 NAT 规则。如果没有规则，直接返回 `NF_ACCEPT`，对包不 做任何改动；如果有规则，最后执行 `nf_nat_packet`，这个函数会进一步调用 `manip_pkt` 完成对包的修改，如果失败，包将被丢弃。



**Masquerade**

NAT 模块一般配置方式：`Change IP1 to IP2 if matching XXX`。

此次还支持一种更灵活的 NAT 配置，称为 Masquerade：`Change IP1 to dev1's IP if matching XXX`。与前面的区别在于，当设备（网卡）的 IP 地址发生变化时，这种方式无 需做任何修改。缺点是性能比第一种方式要差。



#### 4.6 `nf_nat_packet()`：执行 NAT

```c
// net/netfilter/nf_nat_core.c

/* Do packet manipulations according to nf_nat_setup_info. */
unsigned int nf_nat_packet(struct nf_conn *ct, enum ip_conntrack_info ctinfo,
      unsigned int hooknum, struct sk_buff *skb)
{
 enum nf_nat_manip_type mtype = HOOK2MANIP(hooknum);
 enum ip_conntrack_dir dir = CTINFO2DIR(ctinfo);
 unsigned int verdict = NF_ACCEPT;

 statusbit = (mtype == NF_NAT_MANIP_SRC? IPS_SRC_NAT : IPS_DST_NAT)

 if (dir == IP_CT_DIR_REPLY)     // Invert if this is reply dir
  statusbit ^= IPS_NAT_MASK;

 if (ct->status & statusbit)     // Non-atomic: these bits don't change. */
  verdict = nf_nat_manip_pkt(skb, ct, mtype, dir);

 return verdict;
}
static unsigned int nf_nat_manip_pkt(struct sk_buff *skb, struct nf_conn *ct,
         enum nf_nat_manip_type mtype, enum ip_conntrack_dir dir)
{
 struct nf_conntrack_tuple target;

 /* We are aiming to look like inverse of other direction. */
 nf_ct_invert_tuplepr(&target, &ct->tuplehash[!dir].tuple);

 l3proto = __nf_nat_l3proto_find(target.src.l3num);
 l4proto = __nf_nat_l4proto_find(target.src.l3num, target.dst.protonum);
 if (!l3proto->manip_pkt(skb, 0, l4proto, &target, mtype)) // 协议相关处理
  return NF_DROP;

 return NF_ACCEPT;
}
```



### 5. 总结

连接跟踪是一个非常基础且重要的网络模块，但只有在少数场景下才会引起普通开发者的注意。

例如，L4LB 短时高并发场景下，LB 节点每秒接受大量并发短连接，可能导致 conntrack table 被打爆。此时的现象是：

- 客户端和 L4LB 建连失败，失败可能是随机的，也可能是集中在某些时间点。
- 客户端重试可能会成功，也可能会失败。
- 在 L4LB 节点抓包看，客户端过来的 TCP SYNC 包 L4LB 收到了，但没有回 ACK。即，包被静默丢弃了（silently dropped）。

**此时的原因可能是 conntrack table 太小，也可能是 GC 不够及 时，甚至是 GC 有bug。**

> 原文链接：http://arthurchiao.art/blog/conntrack-design-and-implementation-zh/





## 深入理解 Kubernetes 网络模型：自己实现 Kube Proxy 的功能

本文译自 [Cracking kubernetes node proxy (aka kube-proxy)](https://arthurchiao.art/blog/cracking-k8s-node-proxy/)。

Kubernetes 中有几种类型的代理。其中有 **node proxier** 或 [kube-proxy](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/)，它在每个节点上反映 Kubernetes API 中定义的服务，可以跨一组后端执行简单的 TCP/UDP/SCTP 流转发 [1]。

为了更好地理解节点代理模型，在这篇文章中，我们将用不同的方法设计和实现我们自己版本的 `kube-proxy`; 尽管这些只是 `toy-proxy`，但从**透明流量拦截、转发、负载均衡**等方面来说，它们的工作方式与 K8S 集群中运行的普通 `kube-proxy` 基本相同。

通过我们的 `toy-proxy` 程序，非 K8S 节点（不在 K8S 集群中）上的应用程序（无论是宿主本地应用程序，还是在 VM/容器中运行的应用程序）也可以通过 **ClusterIP** 访问 K8S 服务 – **注意，在 kubernetes 的设计中，ClusterIP 只能在 K8S 集群节点中访问（在某种意义上，我们的 `toy-proxy` 程序将非 K8S 节点变成了 K8S 节点）。**



### 背景知识

了解 Linux 内核中的流量拦截和代理需要具备以下背景知识。

#### Netfilter

Netfilter 是 Linux 内核内部的**包过滤和处理框架**。如果你不熟悉 Iptables 和 Netfilter 体系结构，请参阅 [A Deep Dive into Iptables and Netfilter Architecture](https://www.digitalocean.com/community/tutorials/a-deep-dive-into-iptables-and-netfilter-architecture)

一些要点：

- 主机上的**所有数据包**都将通过 netfilter 框架
- 在 netfilter 框架中有 **5 个钩子**点：`PRE_ROUTING`, `INPUT`, `FORWARD`, `OUTPUT`, `POST_ROUTING`
- 命令行工具 `iptables` 可用于**动态地将规则插入到钩子点中**
- 可以通过组合各种 `iptables` 规则来操作数据包（接受/重定向/删除/修改，等等）

![The 5 hook points in netfilter framework](D:\学习资料\笔记\k8s\k8s图\proxy_hooks.png)

此外，这 5 个钩子点还可以与内核的其他网络设施，如内核路由子系统进行协同工作。

此外，在每个钩子点中，规则被组织到具有预定义优先级的不同链中。为了按目的管理链，链被进一步组织到表中。现在有 5 个表：

- `filter`：做正常的过滤，如接受，拒绝/删，跳
- `nat`：网络地址转换，包括 SNAT（源 nat) 和 DNAT（目的 nat)
- `mangle`：修改包属性，例如 TTL
- `raw`：最早的处理点，连接跟踪前的特殊处理 (conntrack 或 CT，也包含在上图中，但这不是链）
- `security`：本文未涉及

将表/链添加到上图中，我们可以得到更详细的视图：

![iptables table/chains inside hook points](D:\学习资料\笔记\k8s\k8s图\proxy_hooks-and-tables.png)

#### VIP 与负载均衡 (LB)

虚拟 IP (IP) 将所有后端 IP 隐藏给客户端/用户，因此客户端/用户总是与 VIP 的后端服务通信，而不需要关心 VIP 后面有多少实例。

VIP 总是伴随着负载均衡，因为它需要在不同的后端之间分配流量。

![VIP and load balancing](D:\学习资料\笔记\k8s\k8s图\proxy_vip-and-lb.png)



#### Cross-host 网络模型

主机 A 上的实例（容器、VM 等）如何与主机 B 上的另一个实例通信？有很多解决方案：

- 直接路由：BGP 等
- 隧道：VxLAN, IPIP, GRE 等
- NAT：例如 docker 的桥接网络模式
- 其它方式



### 节点代理模型

在 kubernetes 中，你可以将应用程序定义为 `Service`。`Service` 是一种抽象，它定义了一组 pod 的逻辑集和访问它们的策略。

#### Service 类型

K8S 中定义了 4 种 `Service` 类型：

- `ClusterIP`：通过 VIP 访问 Service，但该 VIP 只能在此集群内访问
- `NodePort`：通过 NodeIP:NodePort 访问 Service，这意味着该端口将暴露在集群内的所有节点上
- `ExternalIP`：与 `ClusterIP` 相同，但是这个 VIP 可以从这个集群之外访问
- `LoadBalancer`

这篇文章将关注 `ClusterIP`，但是其他三种类型在流量拦截和转发方面的底层实现非常相似。

#### 节点代理

一个 Service 有一个 VIP（本文中的 `ClusterIP`）和多个端点（后端 pod）。每个 pod 或节点都可以通过 VIP 直接访问应用程序。要做到这一点，节点代理程序需要在每个节点上运行，它应该能够透明地拦截到任何 `ClusterIP:Port`[注解 1] 的流量，并将它们重定向到一个或多个后端 pod。

![Kubernetes proxier model](D:\学习资料\笔记\k8s\k8s图\proxy_k8s-proxier-model.png)



> 注解 1：
>
> 对 `ClusterIP` 的一个常见误解是，`ClusterIP` 是可访问的——它们不是通过定义访问的。如果 ping 一个 `ClusterIP`，可能会发现它不可访问。
>
> 根据定义，**<Protocol,ClusterIP,Port>** 元组独特地定义了一个服务（因此也定义了一个拦截规则）。例如，如果一个服务被定义为 `<tcp,10.7.0.100,80>`，那么代理只处理 `tcp:10.7.0.100:80` 的流量，其他流量，例如。`tcp:10.7.0.100:8080`, `udp:10.7.0.100:80` 将不会被代理。因此，也无法访问 ClusterIP（ICMP 流量）。
>
> 但是，如果你使用的是带有 IPVS 模式的 `kube-proxy`，那么确实可以通过 ping 访问 `ClusterIP`。这是因为 IPVS 模式实现比定义所需要的做得更多。你将在下面几节中看到不同之处。



#### 节点代理的角色：反向代理

想想节点代理的作用，在 K8S 网络模型中，它实际上是一个反向代理，也就是说，在每个节点上，它将：

- 将所有后端 pod 隐藏到客户端
- 过滤所有出口流量（对后端的请求）

对于 ingress traffic，它什么也不做。



#### 性能问题

如果我们在主机上有一个应用程序，并且在 K8S 集群中有 1K 个服务，那么我们永远无法猜测该应用程序在下一时刻将访问哪个服务（这里忽略网络策略）。因此，为了让应用程序能够访问所有服务，我们必须为节点上的所有服务应用所有代理规则。将这个想法推广到整个集群，这意味着：

**所有服务的代理规则应该应用于整个集群中的所有节点。**

在某种意义上，这是一个完全分布式的代理模型，因为任何节点都拥有集群的所有规则。

当集群变大时，这会导致严重的性能问题，因为每个节点上可能有数十万条规则 [6,7]。



### 测试环境

#### 集群拓扑和测试环境

我们将使用以下环境进行测试：

- 一个 k8s 集群
  - 一个 master 节点
  - 一个 node 节点
  - 网络解决方案：直接路由（PodIP 可直接路由）
- 一个非 k8s 节点，但是它可以到达工作节点和 Pod（得益于直接路由网络方案）

![test env](D:\学习资料\笔记\k8s\k8s图\proxy_test-env.png)

我们将在工作节点上部署 pod，并从 test 节点通过 `ClusterIP` 访问 pod 中的应用程序。



#### 创建一个 Service

创建一个简单的 `Statefulset`，其中包括一个 `Service`，该 `Service` 将有一个或多个后端 pod:

```sh
# see appendix for webapp.yaml
$ kubectl create -f webapp.yaml

$ kubectl get svc -o wide webapp
NAME     TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE     SELECTOR
webapp   ClusterIP   10.7.111.132   <none>        80/TCP    2m11s   app=webapp

$ kubectl get pod -o wide | grep webapp
webapp-0    2/2     Running   0    2m12s 10.5.41.204    node1    <none>  <none>
```

应用程序在带有 tcp 协议的 80 端口上运行。



### 可达性测试

首先访问 PodIP+Port:

```bash
$ curl 10.5.41.204:80
<!DOCTYPE html>
...
</html>
```

成功的！然后用 `ClusterIP` 替换 PodIP 再试一次：

```bash
$ curl 10.7.111.132:80
^C
```

正如所料，它是不可访问的！

在下一节中，我们将研究如何使用不同的方法使 `ClusterIP` 可访问。



### 实现：通过 userspace socket 实现 proxy

#### 中间人模型

最容易理解的实现是在此主机上的通信路径中插入我们的 `toy-proxy` 作为中间人：对于从本地客户端到 ClusterIP:Port 的每个连接，**我们拦截该连接并将其分割为两个单独的连接**:

- 本地客户端和 `toy-proxy` 之间的连接
- 连接 `toy-proxy` 和后端 pod

实现此目的的最简单方法是在用户空间中实现它：

- `监听资源`：启动一个守护进程，监听 K8S apiserver、监视服务 (ClusterIP) 和端点 (Pod) 的变化
- `代理通信`：对于从本地客户端到服务 (ClusterIP) 的每个连接请求，通过充当中间人来拦截请求
- `动态应用代理规则`：对于任何 Service/Endpoint 更新，相应地更改 `toy-proxy` 连接设置

对于我们上面的测试应用 `webapp`，数据流程如下图：

![userspace-proxier](D:\学习资料\笔记\k8s\k8s图\proxy_userspace-proxier.png)



#### POC 实现

让我们来看看上图的概念验证实现。

##### 代码

以下代码省略了一些错误处理代码，便于阅读：

```go
func main() {
	clusterIP := "10.7.111.132"
	podIP := "10.5.41.204"
	port := 80
	proto := "tcp"

	addRedirectRules(clusterIP, port, proto)
	createProxy(podIP, port, proto)
}

func addRedirectRules(clusterIP string, port int, proto string) error {
	p := strconv.Itoa(port)
	cmd := exec.Command("iptables", "-t", "nat", "-A", "OUTPUT", "-p", "tcp",
		"-d", clusterIP, "--dport", p, "-j", "REDIRECT", "--to-port", p)
	return cmd.Run()
}

func createProxy(podIP string, port int, proto string) {
	host := ""
	listener, err := net.Listen(proto, net.JoinHostPort(host, strconv.Itoa(port)))

	for {
		inConn, err := listener.Accept()
		outConn, err := net.Dial(proto, net.JoinHostPort(podIP, strconv.Itoa(port)))

		go func(in, out *net.TCPConn) {
			var wg sync.WaitGroup
			wg.Add(2)
			fmt.Printf("Proxying %v <-> %v <-> %v <-> %v\n",
				in.RemoteAddr(), in.LocalAddr(), out.LocalAddr(), out.RemoteAddr())
			go copyBytes(in, out, &wg)
			go copyBytes(out, in, &wg)
			wg.Wait()
		}(inConn.(*net.TCPConn), outConn.(*net.TCPConn))
	}

	listener.Close()
}

func copyBytes(dst, src *net.TCPConn, wg *sync.WaitGroup) {
	defer wg.Done()
	if _, err := io.Copy(dst, src); err != nil {
		if !strings.HasSuffix(err.Error(), "use of closed network connection") {
			fmt.Printf("io.Copy error: %v", err)
		}
	}
	dst.Close()
	src.Close()
}
```



##### 一些解释

**traffic 拦截**

我们想拦截所有发往 `ClusterIP:Port` 的流量，但是在这个节点上任何设备都没有配置`ClusterIP`，因此我们无法执行诸如 listen（ClusterIP，Port）之类的操作，那么我们如何才能拦截呢？答案是：使用`iptables/netfilter` 提供的 `REDIRECT` 能力。

以下命令会将所有发往 `ClusterIP:Port` 的流量定向到 `localhost:Port`：

```bash
$ sudo iptables -t nat -A OUTPUT -p tcp -d $CLUSTER_IP --dport $PORT -j REDIRECT --to-port $PORT
```

如果你现在不能理解这一点，不要害怕。稍后我们将讨论这个问题。

通过下面命令的输出来验证这一点：

```bash
$ iptables -t nat -L -n
...
Chain OUTPUT (policy ACCEPT)
target     prot opt source      destination
REDIRECT   tcp  --  0.0.0.0/0   10.7.111.132         tcp dpt:80 redir ports 80
```

在代码中，函数 `addRedirectRules()` 包装了上述过程。



##### 创建 proxy

函数 `createProxy()` 创建用户空间代理，并执行双向转发。



#### 可达性测试

编译代码并执行二进制文件：

```bash
$ go build toy-proxy-userspace.go
$ sudo ./toy-proxy-userspace
```

现在测试访问：

```bash
$ curl $CLUSTER_IP:$PORT
<!DOCTYPE html>
...
</html>
```

成功！我们的代理传达的信息是：

```bash
$ sudo ./toy-proxy-userspace
Creating proxy between <host ip>:53912 <-> 127.0.0.1:80 <-> <host ip>:40194 <-> 10.5.41.204:80
```

表示，对于原 `<host ip>:53912 <-> 10.7.111.132:80` 的连接请求，将其拆分为两个连接：

1. `<host ip>:53912 <-> 127.0.0.1:80`
2. `<host ip>:40194 <-> 10.5.41.204:80`

删除这条规则：

```bash
$ iptables -t nat -L -n --line-numbers
...
Chain OUTPUT (policy ACCEPT)
num  target     prot opt source               destination
2    REDIRECT   tcp  --  0.0.0.0/0   10.7.111.132         tcp dpt:80 redir ports 80

# iptables -t nat -D OUTPUT <num>
$ iptables -t nat -D OUTPUT 2
```

或者删除（刷新）所有规则，如果你把 iptabels 弄的一团糟的情况下：

```bash
$ iptables -t nat -F # delete all rules
$ iptables -t nat -X # delete all custom chains
```



##### 改进

在这个 `toy-proxy` 实现中，我们拦截了 `ClusterIP:80` 到 `localhost:80`，但是如果该主机上的本机应用程序也想使用 `localhost:80` 怎么办？此外，如果多个服务都公开 80 端口会怎样？显然，我们需要区分这些应用程序或服务。解决这个问题的正确方法是：为每个代理分配一个未使用的临时端口 TmpPort，拦截 `ClusterIP:Port` 到 `local:TmpPort`。例如，app1 使用 10001, app2 使用 10002。

其次，上面的代码只处理一个后端，如果有多个后端 pod 怎么办？因此，我们需要通过负载均衡算法将请求分发到不同的后端 pod。

![userspace-proxier-2](D:\学习资料\笔记\k8s\k8s图\proxy_userspace-proxier-2.png)



##### 优缺点

这种方法非常容易理解和实现，但是，它的性能会很差，因为它必须在两端以及内核和用户空间内存之间复制字节。

我们没有在这上面花太多时间，如果你感兴趣，可以在这里查看用户空间 `kube-proxy` 的简单实现。

接下来，让我们看看实现这个任务的另一种方法。



### 实现：通过 iptables 实现 proxy

用户空间代理程序的主要瓶颈来自内核-用户空间切换和数据复制。**如果我们可以完全在内核空间中实现代理**，它将在性能上大大提高，从而击败用户空间的代理。`iptables` 可用于实现这一目标。

在开始之前，让我们首先弄清楚在执行 `curl ClusterIP:Port` 时的流量路径，然后研究如何使用 `iptables` 规则使其可访问。



#### Host -> ClusterIP（单一后端）

`ClusterIP` 不存在于任何网络设备上，所以为了让我们的数据包最终到达后端 Pod，我们需要将 `ClusterIP` 转换为 PodIP（可路由），即：

- 条件：匹配 `dst=ClusterIP,proto=tcp,dport=80` 的数据包
- 操作：将数据包的 IP 报头中的 `dst=ClusterIP` 替换为 `dst=PodIP`

用网络术语来说，这是一个网络地址转换 (NAT) 过程。



##### 在哪里做 DNAT

通过 curl 查看出口数据包路径（下图展示了数据流向过程）：

![host-to-clusterip-dnat](D:\学习资料\笔记\k8s\k8s图\proxy_host-to-clusterip-dnat.png)

```bash
<curl process> -> raw -> CT -> mangle -> dnat -> filter -> security -> snat -> <ROUTING> -> mangle -> snat -> NIC
```

很明显，在 OUTPUT 钩中只有一个 dnat（链），我们可以在其中进行 DNAT。

让我们看看我们将如何进行黑客入侵。



##### 检查当前的 NAT 规则

`NAT` 规则被组织到 `nat` 表中。检查 `nat` 表中的当前规则：

```bash
# -t <table>
# -L list rules
# -n numeric output
$ iptables -t nat -L -n
Chain PREROUTING (policy ACCEPT)

Chain INPUT (policy ACCEPT)

Chain OUTPUT (policy ACCEPT)
DOCKER     all  --  0.0.0.0/0    !127.0.0.0/8   ADDRTYPE match dst-type LOCAL

Chain POSTROUTING (policy ACCEPT)
```

输出显示除了与 DOCKER 相关的规则外，没有其他规则。这些 DOCKER 规则是 DOCKER 在安装时插入的，但它们不会影响我们在这篇文章中的实验。所以我们忽略它们。



##### 增加 DNAT 规则

为了便于查看，我们不会用 go 代码包装 `iptables` 命令，而是直接显示命令本身。

> 注意：在继续之前，请确保删除了在上一节中添加的所有规则。

确认目前无法访问 ClusterIP：

```bash
$ curl $CLUSTER_IP:$PORT
^C
```

现在添加我们的出口 NAT 规则：

```bash
$ cat ENV
CLUSTER_IP=10.7.111.132
POD_IP=10.5.41.204
PORT=80
PROTO=tcp

# -p               <protocol>
# -A               add rule
# --dport          <dst port>
# -d               <dst ip>
# -j               jump to
# --to-destination <ip>:<port>
$ iptables -t nat -A OUTPUT -p $PROTO --dport $PORT -d $CLUSTER_IP -j DNAT --to-destination $POD_IP:$PORT
```

再次检查规则表：

```bash
$ iptables -t nat -L -n

Chain OUTPUT (policy ACCEPT)
target     prot opt source      destination
DNAT       tcp  --  0.0.0.0/0   10.7.111.132   tcp dpt:80 to:10.5.41.204:80
```

我们可以看到规则已经被添加。

##### 测试可达性

现在再一次访问：

```bash
$ curl $CLUSTER_IP:$PORT
<!DOCTYPE html>
...
</html>
```

就是这样！访问成功。

但是等等！我们期望出口的交通应该是正确的，但我们没有添加任何 NAT 规则的入口路径，怎么可能交通是正常的两个方向？事实证明，当你为一个方向添加一个 NAT 规则时，Linux 内核会自动为另一个方向添加保留规则！这与 conntrack (CT，连接跟踪）模块协同工作。

![host-to-clusterip-dnat-ct](D:\学习资料\笔记\k8s\k8s图\proxy_host-to-clusterip-dnat-ct.png)

##### 清理

删除这些规则：

```bash
$ iptables -t nat -L -n --line-numbers
...
Chain OUTPUT (policy ACCEPT)
num  target     prot opt source               destination
2    DNAT       tcp  --  0.0.0.0/0   10.7.111.132   tcp dpt:80 to:10.5.41.204:80

# iptables -t <table> -D <chain> <num>
$ iptables -t nat -D OUTPUT 2
```



#### Host -> ClusterIP （多个后端）

在上一节中，我们展示了如何使用一个后端 Pod 执行 NAT。现在让我们看看多后端情况。

> 注意：在继续之前，请确保删除了在上一节中添加的所有规则。

##### 伸缩 webapp

首先扩大我们的服务到 2 个后端 pod:

```bash
$ kubectl scale sts webapp --replicas=2
statefulset.apps/webapp scaled

$ kubectl get pod -o wide | grep webapp
webapp-0   2/2     Running   0   1h24m   10.5.41.204    node1    <none> <none>
webapp-1   2/2     Running   0   11s     10.5.41.5      node1    <none> <none>
```

##### 通过负载平衡添加 DNAT 规则

我们需要 `iptables` 中的 `statistic` 模块以概率的方式将请求分发到后端 pod，这样才能达到负载均衡的效果：

```bash
# -m <module>
$ iptables -t nat -A OUTPUT -p $PROTO --dport $PORT -d $CLUSTER_IP \
    -m statistic --mode random --probability 0.5  \
    -j DNAT --to-destination $POD1_IP:$PORT
$ iptables -t nat -A OUTPUT -p $PROTO --dport $PORT -d $CLUSTER_IP \
    -m statistic --mode random --probability 1.0  \
    -j DNAT --to-destination $POD2_IP:$PORT
```

上面的命令指定在两个 pod 之间随机分配请求，每个都有 50% 的概率。

现在检查这些规则：

```bash
$ iptables -t nat -L -n
...
Chain OUTPUT (policy ACCEPT)
target  prot opt source      destination
DNAT    tcp  --  0.0.0.0/0   10.7.111.132  tcp dpt:80 statistic mode random probability 0.50000000000 to:10.5.41.204:80
DNAT    tcp  --  0.0.0.0/0   10.7.111.132  tcp dpt:80 statistic mode random probability 1.00000000000 to:10.5.41.5:80
```

![host-to-clusterip-lb-ct](D:\学习资料\笔记\k8s\k8s图\proxy_host-to-clusterip-lb-ct.png)

##### 验证

现在，我们来验证下负载均衡是否生效。我们发出 8 个 请求，并捕获到这个主机通信的真实 PodIPs:

在测试节点上打开一个 shell:

```bash
$ for i in {1..8}; do curl $CLUSTER_IP:$PORT 2>&1 >/dev/null; sleep 1; done
```

测试节点上的另一个 shell 窗口：

```bash
$ tcpdump -nn -i eth0 port $PORT | grep "GET /"
10.21.0.7.48306 > 10.5.41.5.80:   ... HTTP: GET / HTTP/1.1
10.21.0.7.48308 > 10.5.41.204.80: ... HTTP: GET / HTTP/1.1
10.21.0.7.48310 > 10.5.41.204.80: ... HTTP: GET / HTTP/1.1
10.21.0.7.48312 > 10.5.41.5.80:   ... HTTP: GET / HTTP/1.1
10.21.0.7.48314 > 10.5.41.5.80:   ... HTTP: GET / HTTP/1.1
10.21.0.7.48316 > 10.5.41.204.80: ... HTTP: GET / HTTP/1.1
10.21.0.7.48318 > 10.5.41.5.80:   ... HTTP: GET / HTTP/1.1
10.21.0.7.48320 > 10.5.41.204.80: ... HTTP: GET / HTTP/1.1
```

在 Pod1 中有 4 次，在 Pod2 中有 4 次，每个 pod 有 50%，这正是我们所期望的。

##### 清理

```bash
$ iptables -t nat -L -n --line-numbers
...
Chain OUTPUT (policy ACCEPT)
num  target     prot opt source               destination
2    DNAT    tcp  --  0.0.0.0/0   10.7.111.132  tcp dpt:80 statistic mode random probability 0.50000000000 to:10.5.41.204:80
3    DNAT    tcp  --  0.0.0.0/0   10.7.111.132  tcp dpt:80 statistic mode random probability 1.00000000000 to:10.5.41.5:80

$ iptables -t nat -D OUTPUT 2
$ iptables -t nat -D OUTPUT 3
```



#### Pod (app A) -> ClusterIP (app B)

如果想通过 hostA 上的 `Pod A` 通过 `ClusterIP` 访问 `Pod B`，B 的 Pod 驻留在 hostB 上，我们应该做什么？

实际上，这与 `Host -> ClusterIP` 情况非常相似，但是有一点需要注意：在执行 NAT 之后，源节点 (hostA) 需要将包发送到目的地 Pod 所在的正确目的地节点 (hostB)。根据不同的跨主机网络解决方案，这有很大不同：

1. 对于直接路由的情况下，主机只是发送数据包。对应的有这些解决方案：
   - calico + bird
   - cilium + kube-router（Cilium BGP 的默认解决方案）
   - cilium + bird（实际上这只是我们的测试环境网络解决方案）
2. 对于隧道的情况，每个主机上必须有一个代理，它在 DNAT 之后执行 encap，在 SNAT 之前执行 decap。这些解决方案包括：
   - calico + VxLAN 模式
   - flannel + IPIP 模式
   - flannel + VxLAN 模式
   - cilium + VxLAN 模式
3. 像 aws 的 ENI 模式：类似于直接路由，但不需要 BGP 代理
   - cilium + ENI 模式

下图展示了隧道的情况：

![tunneling](D:\学习资料\笔记\k8s\k8s图\proxy_tunneling.png)

代理与隧道相关的职责包括：

- **同步所有节点之间的隧道信息**，例如描述哪个实例在哪个节点上的信息
- **在 DNAT 之后对 pod 流量执行封装**：对于所有的出口流量，例如来自 hostA 的 `dst=<PodIP>`，其中 PodIP 在 hostB 上，通过添加另一个头来封装数据包，例如 VxLAN 头，其中封装头有 `src=hostA_IP,dst=hostB_IP`
- **在 SNAT 之前对 Pod 流量执行解封装**：解封装每个入口封装的数据包：删除外层（例如 VxLAN 标头）

同时，主机需要决定：

- 哪些数据包应该交给解码器（pod 流量），哪些不应该（例如主机流量）
- 哪些包应该封装（pod 流量），哪些不应该（例如主机流量）



#### 重新构造 iptables 规则

> 注意：在继续之前，请确保删除了在上一节中添加的所有规则。

当你有大量的 Service 时，每个节点上的 iptables 规则将相当复杂，因此你需要进行一些结构化工作来组织这些规则。

在本节中，我们将在 nat 表中创建几个专用的 iptables 链，具体如下：

- 链 `KUBE-SERVICES`：拦截 nat 表的输出链中所有到此链的出口流量，如果它们被指定为 ClusterIP，则执行 DNAT
- 链 `KUBE-SVC-WEBAPP`：如果 `dst`、`proto` 和 `port` 匹配，则拦截该链 `KUBE-SERVICES` 中的所有流量
- 链 `KUBE-SEP-WEBAPP1`：拦截 50% 的流量在 `KUBE-SVC-WEBAPP` 到这里
- 链 `KUBE-SEP-WEBAPP2`：拦截 50% 的流量在 `KUBE-SVC-WEBAPP` 到这里

DNAT 路径现在为：

```bash
OUTPUT -> KUBE-SERVICES -> KUBE-SVC-WEBAPP --> KUBE-SEP-WEBAPP1
                                         \
                                          \--> KUBE-SEP-WEBAPP2
```

如果你有多个 Service，DNAT 路径如下：

```bash
OUTPUT -> KUBE-SERVICES -> KUBE-SVC-A --> KUBE-SEP-A1
                      |              \--> KUBE-SEP-A2
                      |
                      |--> KUBE-SVC-B --> KUBE-SEP-B1
                      |              \--> KUBE-SEP-B2
                      |
                      |--> KUBE-SVC-C --> KUBE-SEP-C1
                                     \--> KUBE-SEP-C2
```

iptables 命令：

```bash
$ cat add-dnat-structured.sh
source ../ENV

set -x

KUBE_SVCS="KUBE-SERVICES"        # chain that serves as kubernetes service portal
SVC_WEBAPP="KUBE-SVC-WEBAPP"     # chain that serves as DNAT entrypoint for webapp
WEBAPP_EP1="KUBE-SEP-WEBAPP1"    # chain that performs dnat to pod1
WEBAPP_EP2="KUBE-SEP-WEBAPP2"    # chain that performs dnat to pod2

# OUTPUT -> KUBE-SERVICES
sudo iptables -t nat -N $KUBE_SVCS
sudo iptables -t nat -A OUTPUT -p all -s 0.0.0.0/0 -d 0.0.0.0/0 -j $KUBE_SVCS

# KUBE-SERVICES -> KUBE-SVC-WEBAPP
sudo iptables -t nat -N $SVC_WEBAPP
sudo iptables -t nat -A $KUBE_SVCS -p $PROTO -s 0.0.0.0/0 -d $CLUSTER_IP --dport $PORT -j $SVC_WEBAPP

# KUBE-SVC-WEBAPP -> KUBE-SEP-WEBAPP*
sudo iptables -t nat -N $WEBAPP_EP1
sudo iptables -t nat -N $WEBAPP_EP2
sudo iptables -t nat -A $WEBAPP_EP1 -p $PROTO -s 0.0.0.0/0 -d 0.0.0.0/0 --dport $PORT -j DNAT --to-destination $POD1_IP:$PORT
sudo iptables -t nat -A $WEBAPP_EP2 -p $PROTO -s 0.0.0.0/0 -d 0.0.0.0/0 --dport $PORT -j DNAT --to-destination $POD2_IP:$PORT
sudo iptables -t nat -A $SVC_WEBAPP -p $PROTO -s 0.0.0.0/0 -d 0.0.0.0/0 -m statistic --mode random --probability 0.5  -j $WEBAPP_EP1
sudo iptables -t nat -A $SVC_WEBAPP -p $PROTO -s 0.0.0.0/0 -d 0.0.0.0/0 -m statistic --mode random --probability 1.0  -j $WEBAPP_EP2
```

现在测试我们设计：

```bash
$ ./add-dnat-structured.sh
++ KUBE_SVCS=KUBE-SERVICES
++ SVC_WEBAPP=KUBE-SVC-WEBAPP
++ WEBAPP_EP1=KUBE-SEP-WEBAPP1
++ WEBAPP_EP2=KUBE-SEP-WEBAPP2
++ sudo iptables -t nat -N KUBE-SERVICES
++ sudo iptables -t nat -A OUTPUT -p all -s 0.0.0.0/0 -d 0.0.0.0/0 -j KUBE-SERVICES
++ sudo iptables -t nat -N KUBE-SVC-WEBAPP
++ sudo iptables -t nat -A KUBE-SERVICES -p tcp -s 0.0.0.0/0 -d 10.7.111.132 --dport 80 -j KUBE-SVC-WEBAPP
++ sudo iptables -t nat -N KUBE-SEP-WEBAPP1
++ sudo iptables -t nat -N KUBE-SEP-WEBAPP2
++ sudo iptables -t nat -A KUBE-SEP-WEBAPP1 -p tcp -s 0.0.0.0/0 -d 0.0.0.0/0 --dport 80 -j DNAT --to-destination 10.5.41.204:80
++ sudo iptables -t nat -A KUBE-SEP-WEBAPP2 -p tcp -s 0.0.0.0/0 -d 0.0.0.0/0 --dport 80 -j DNAT --to-destination 10.5.41.5:80
++ sudo iptables -t nat -A KUBE-SVC-WEBAPP -p tcp -s 0.0.0.0/0 -d 0.0.0.0/0 -m statistic --mode random --probability 0.5 -j KUBE-SEP-WEBAPP1
++ sudo iptables -t nat -A KUBE-SVC-WEBAPP -p tcp -s 0.0.0.0/0 -d 0.0.0.0/0 -m statistic --mode random --probability 1.0 -j KUBE-SEP-WEBAPP2
```

检查这些规则：

```bash
$ sudo iptables -t nat -L -n
...
Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
KUBE-SERVICES  all  --  0.0.0.0/0            0.0.0.0/0

Chain KUBE-SEP-WEBAPP1 (1 references)
target     prot opt source               destination
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:80 to:10.5.41.204:80

Chain KUBE-SEP-WEBAPP2 (1 references)
target     prot opt source               destination
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:80 to:10.5.41.5:80

Chain KUBE-SERVICES (1 references)
target     prot opt source               destination
KUBE-SVC-WEBAPP  tcp  --  0.0.0.0/0            10.7.111.132         tcp dpt:80

Chain KUBE-SVC-WEBAPP (1 references)
target     prot opt source               destination
KUBE-SEP-WEBAPP1  tcp  --  0.0.0.0/0            0.0.0.0/0            statistic mode random probability 0.50000000000
KUBE-SEP-WEBAPP2  tcp  --  0.0.0.0/0            0.0.0.0/0            statistic mode random probability 1.00000000000
$ curl $CLUSTER_IP:$PORT
<!DOCTYPE html>
...
</html>
```

成功！

如果你将上面的输出与普通的 `kube-proxy` 规则进行比较，这两个规则是非常相似的，下面是从启用 `kube-proxy` 的节点提取的：

```bash
Chain OUTPUT (policy ACCEPT)
target         prot opt source               destination
KUBE-SERVICES  all  --  0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */

Chain KUBE-SERVICES (2 references)
target                     prot opt source               destination
KUBE-SVC-YK2SNH4V42VSDWIJ  tcp  --  0.0.0.0/0            10.7.22.18           /* default/nginx:web cluster IP */ tcp dpt:80

Chain KUBE-SVC-YK2SNH4V42VSDWIJ (1 references)
target                     prot opt source               destination
KUBE-SEP-GL2BLSI2B4ICU6WH  all  --  0.0.0.0/0            0.0.0.0/0            /* default/nginx:web */ statistic mode random probability 0.33332999982
KUBE-SEP-AIRRSG3CIF42U3PX  all  --  0.0.0.0/0            0.0.0.0/0            /* default/nginx:web */

Chain KUBE-SEP-GL2BLSI2B4ICU6WH (1 references)
target          prot opt source               destination
DNAT            tcp  --  0.0.0.0/0            0.0.0.0/0            /* default/nginx:web */ tcp to:10.244.3.181:80

Chain KUBE-SEP-AIRRSG3CIF42U3PX (1 references)
target          prot opt source               destination
DNAT            tcp  --  0.0.0.0/0            0.0.0.0/0            /* default/nginx:web */ tcp to:10.244.3.182:80
```



#### 进一步重新构造 iptables 规则

TODO：为来自集群外部的流量添加规则。



### 实现：通过 ipvs 实现 proxy

虽然基于 iptables 的代理在性能上优于基于用户空间的代理，但在集群服务过多的情况下也会导致性能严重下降 [6,7]。

本质上，这是因为 iptables 判决是基于链的，它是一个复杂度为 O(n) 的线性算法。iptables 的一个好的替代方案是 IPVS——内核中的 L4 负载均衡器，它在底层使用 ipset（哈希实现），因此复杂度为 O(1)。

让我们看看如何使用 ipvs 实现相同的目标。

> 注意：在继续之前，请确保删除了在上一节中添加的所有规则。



#### 安装 IPVS

```bash
$ yum install -y ipvsadm

# -l  list load balancing status
# -n  numeric output
$ ipvsadm -ln
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
```

默认无规则

##### 增加虚拟/真正的 services

使用 ipvs 实现负载均衡：

```bash
# -A/--add-service           add service
# -t/--tcp-service <address> VIP + Port
# -s <method>                scheduling-method
# -r/--real-server <address> real backend IP + Port
# -m                         masquerading (NAT)
$ ipvsadm -A -t $CLUSTER_IP:$PORT -s rr
$ ipvsadm -a -t $CLUSTER_IP:$PORT -r $POD1_IP -m
$ ipvsadm -a -t $CLUSTER_IP:$PORT -r $POD2_IP -m
```

或者使用我的脚本：

```bash
$ ./ipvs-add-server.sh
Adding virtual server CLUSTER_IP:PORT=10.7.111.132:80 ...
Adding real servers ...
10.7.111.132:80 -> 10.5.41.204
10.7.111.132:80 -> 10.5.41.5
Done
```

再次检查状态：

```bash
$ ipvsadm -ln
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.7.111.132:80 rr
  -> 10.5.41.5:80                 Masq    1      0          0
  -> 10.5.41.204:80               Masq    1      0          0
```

一些解释：

- 对于所有发往 `10.7.111.132:80` 的流量，将负载均衡到 `10.5.41.5:80` 和 `10.5.41.204:80`
- 使用轮询 (rr) 算法实现负载均衡
- 两个后端，每个后端的权重为 1（各 50％）
- 使用 MASQ（增强型 SNAT）在 VIP 和 RealIP 之间进行流量转发



#### 验证

```bash
$ for i in {1..8}; do curl $CLUSTER_IP:$PORT 2>&1 >/dev/null; sleep 1; done

$ tcpdump -nn -i eth0 port $PORT | grep "HTTP: GET"
IP 10.21.0.7.49556 > 10.5.41.204.80: ... HTTP: GET / HTTP/1.1
IP 10.21.0.7.49558 > 10.5.41.5.80  : ... HTTP: GET / HTTP/1.1
IP 10.21.0.7.49560 > 10.5.41.204.80: ... HTTP: GET / HTTP/1.1
IP 10.21.0.7.49562 > 10.5.41.5.80  : ... HTTP: GET / HTTP/1.1
IP 10.21.0.7.49566 > 10.5.41.204.80: ... HTTP: GET / HTTP/1.1
IP 10.21.0.7.49568 > 10.5.41.5.80  : ... HTTP: GET / HTTP/1.1
IP 10.21.0.7.49570 > 10.5.41.204.80: ... HTTP: GET / HTTP/1.1
IP 10.21.0.7.49572 > 10.5.41.5.80  : ... HTTP: GET / HTTP/1.1
```



#### 清理

```bash
$ ./ipvs-del-server.sh
Deleting real servers ...
10.7.111.132:80 -> 10.5.41.204
10.7.111.132:80 -> 10.5.41.5
Deleting virtual server CLUSTER_IP:PORT=10.7.111.132:80 ...
Done
```



### 实现：通过 bpf 实现 proxy

这也是一个 `O(1)` 代理，但是与 IPVS 相比具有更高的性能。

让我们看看如何在不到 100 行 C 代码中使用 eBPF 实现代理功能。



#### 先决条件

如果你有足够的时间和兴趣来阅读 eBPF/BPF，可以考虑阅读 [Cilium: BPF and XDP Reference Guide](https://docs.cilium.io/en/v1.6/bpf/)，它对开发人员来说是一个完美的 BPF 文档。



#### 实现

让我们看看出口部分的基本概念：

1. 对于所有流量，匹配 `dst=CLUSTER_IP && proto==TCP && dport==80`
2. 更改目标 IP：`CLUSTER_IP -> POD_IP`
3. 更新 IP 和 TCP 报头中的校验和文件（否则我们的数据包将被丢弃）

```c
__section("egress")
int tc_egress(struct __sk_buff *skb)
{
    const __be32 cluster_ip = 0x846F070A; // 10.7.111.132
    const __be32 pod_ip = 0x0529050A;     // 10.5.41.5

    const int l3_off = ETH_HLEN;    // IP header offset
    const int l4_off = l3_off + 20; // TCP header offset: l3_off + sizeof(struct iphdr)
    __be32 sum;                     // IP checksum

    void *data = (void *)(long)skb->data;
    void *data_end = (void *)(long)skb->data_end;
    if (data_end < data + l4_off) { // not our packet
        return TC_ACT_OK;
    }

    struct iphdr *ip4 = (struct iphdr *)(data + l3_off);
    if (ip4->daddr != cluster_ip || ip4->protocol != IPPROTO_TCP /* || tcp->dport == 80 */) {
        return TC_ACT_OK;
    }

    // DNAT: cluster_ip -> pod_ip, then update L3 and L4 checksum
    sum = csum_diff((void *)&ip4->daddr, 4, (void *)&pod_ip, 4, 0);
    skb_store_bytes(skb, l3_off + offsetof(struct iphdr, daddr), (void *)&pod_ip, 4, 0);
    l3_csum_replace(skb, l3_off + offsetof(struct iphdr, check), 0, sum, 0);
	l4_csum_replace(skb, l4_off + offsetof(struct tcphdr, check), 0, sum, BPF_F_PSEUDO_HDR);

    return TC_ACT_OK;
}
```

对于入口部分，非常类似于出口代码：

```c
__section("ingress")
int tc_ingress(struct __sk_buff *skb)
{
    const __be32 cluster_ip = 0x846F070A; // 10.7.111.132
    const __be32 pod_ip = 0x0529050A;     // 10.5.41.5

    const int l3_off = ETH_HLEN;    // IP header offset
    const int l4_off = l3_off + 20; // TCP header offset: l3_off + sizeof(struct iphdr)
    __be32 sum;                     // IP checksum

    void *data = (void *)(long)skb->data;
    void *data_end = (void *)(long)skb->data_end;
    if (data_end < data + l4_off) { // not our packet
        return TC_ACT_OK;
    }

    struct iphdr *ip4 = (struct iphdr *)(data + l3_off);
    if (ip4->saddr != pod_ip || ip4->protocol != IPPROTO_TCP /* || tcp->dport == 80 */) {
        return TC_ACT_OK;
    }

    // SNAT: pod_ip -> cluster_ip, then update L3 and L4 header
    sum = csum_diff((void *)&ip4->saddr, 4, (void *)&cluster_ip, 4, 0);
    skb_store_bytes(skb, l3_off + offsetof(struct iphdr, saddr), (void *)&cluster_ip, 4, 0);
    l3_csum_replace(skb, l3_off + offsetof(struct iphdr, check), 0, sum, 0);
	l4_csum_replace(skb, l4_off + offsetof(struct tcphdr, check), 0, sum, BPF_F_PSEUDO_HDR);

    return TC_ACT_OK;
}

char __license[] __section("license") = "GPL";
```



#### 编译并加载到内核中

现在使用我的小脚本编译和加载到内核：

```bash
$ ./compile-and-load.sh
...
++ sudo tc filter show dev eth0 egress
filter protocol all pref 49152 bpf chain 0
filter protocol all pref 49152 bpf chain 0 handle 0x1 toy-proxy-bpf.o:[egress] direct-action not_in_hw id 18 tag f5f39a21730006aa jited

++ sudo tc filter show dev eth0 ingress
filter protocol all pref 49152 bpf chain 0
filter protocol all pref 49152 bpf chain 0 handle 0x1 toy-proxy-bpf.o:[ingress] direct-action not_in_hw id 19 tag b41159c5873bcbc9 jited
```

脚本是这样的：

```bash
$ cat compile-and-load.sh
set -x

NIC=eth0

# compile c code into bpf code
clang -O2 -Wall -c toy-proxy-bpf.c -target bpf -o toy-proxy-bpf.o

# add tc queuing discipline (egress and ingress buffer)
sudo tc qdisc del dev $NIC clsact 2>&1 >/dev/null
sudo tc qdisc add dev $NIC clsact

# load bpf code into the tc egress and ingress hook respectively
sudo tc filter add dev $NIC egress bpf da obj toy-proxy-bpf.o sec egress
sudo tc filter add dev $NIC ingress bpf da obj toy-proxy-bpf.o sec ingress

# show info
sudo tc filter show dev $NIC egress
sudo tc filter show dev $NIC ingress
```

#### 验证

```bash
$ curl $CLUSTER_IP:$PORT
<!DOCTYPE html>
...
</html>
```

完美！

#### 清理

```bash
$ sudo tc qdisc del dev $NIC clsact 2>&1 >/dev/null
```



### 总结

在这篇文章中，我们用不同的方法手工实现了 `kube-proxy` 的核心功能。希望你现在对 kubernetes 节点代理有了更好的理解，以及关于网络的其他一些配置。

在这篇文章中使用的代码和脚本：[这里](https://github.com/icyxp/icyxp.github.io/tree/mastercode)。



#### 参考文献

1. [Kubernetes Doc: CLI - kube-proxy](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/)
2. [kubernetes/enhancements: enhancements/0011-ipvs-proxier.md](https://github.com/kubernetes/enhancements/blob/master/keps/sig-network/0011-ipvs-proxier.md)
3. [Kubernetes Doc: Service types](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types)
4. [Proxies in Kubernetes - Kubernetes](https://kubernetes.io/docs/concepts/cluster-administration/proxies/)
5. [A minimal IPVS Load Balancer demo](https://medium.com/@benmeier_/a-quick-minimal-ipvs-load-balancer-demo-d5cc42d0deb4)
6. [Scaling Kubernetes to Support 50,000 Services](https://docs.google.com/presentation/d/1BaIAywY2qqeHtyGZtlyAp89JIZs59MZLKcFLxKE6LyM/edit#slide=id.p3)
7. [华为云在 K8S 大规模场景下的 Service 性能优化实践](https://zhuanlan.zhihu.com/p/37230013)



### 附录

webapp.yaml:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp
  labels:
    app: webapp
spec:
  ports:
  - port: 80
    name: web
  selector:
    app: webapp
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: webapp
spec:
  serviceName: "webapp"
  replicas: 1
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      # affinity:
      #   nodeAffinity:
      #     requiredDuringSchedulingIgnoredDuringExecution:
      #       nodeSelectorTerms:
      #       - matchExpressions:
      #         - key: kubernetes.io/hostname
      #           operator: In
      #           values:
      #           - node1
      tolerations:
      - effect: NoSchedule
        key: smoke
        operator: Equal
        value: test
      containers:
      - name: webapp
        image: nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
```













































## 利用 eBPF 支撑大规模 K8S Service



K8S 当前重度依赖 iptables 来实现 Service 的抽象，对于每个 Service 及其 backend pods，在 K8s 里会生成很多 iptables 规则。**例如 5K 个 Service 时，iptables 规则将达到 25K 条**，导致的后果：

- **较高、并且不可预测的转发延迟**（packet latency），因为每个包都要遍历这些规则 ，直到匹配到某条规则；
- **更新规则的操作非常慢**：无法单独更新某条 iptables 规则，只能将全部规则读出来 ，更新整个集合，再将新的规则集合下发到宿主机。在动态环境中这一问题尤其明显，因为每 小时可能都有几千次的 backend pods 创建和销毁。
- **可靠性问题**：iptables 依赖 Netfilter 和系统的连接跟踪模块（conntrack），在 大流量场景下会出现一些竞争问题（race conditions）；**UDP 场景尤其明显**，会导 致丢包、应用的负载升高等问题。

本文将介绍如何基于 Cilium/BPF 来解决这些问题，实现 K8s Service 的大规模扩展。



### 1 K8s Service 类型及默认基于 kube-proxy 的实现

K8s 提供了 Service 抽象，可以将多个 backend pods 组织为一个**逻辑单元**（logical unit）。K8s 会为这个逻辑单元分配 **虚拟 IP 地址**（VIP），客户端通过该 VIP 就 能访问到这些 pods 提供的服务。

下图是一个具体的例子，

![image-20210121205235546](D:\学习资料\笔记\k8s\k8s图\image-20210121205235546.png)

1. 右边的 yaml 定义了一个名为 `nginx` 的 Service，它在 TCP 80 端口提供服务；

2. - 创建：`kubectl -f nginx-svc.yaml`

3. K8s 会给每个 Service 分配一个虚拟 IP，这里给 `nginx` 分的是 `3.3.3.3`；

4. - 查看：`kubectl get service nginx`

5. 左边是 `nginx` Service 的两个 backend pods（在 K8s 对应两个 endpoint），这里 位于同一台节点，每个 Pod 有独立的 IP 地址；

6. - 查看：`kubectl get endpoints nginx`

上面看到的是所谓的 `ClusterIP` 类型的 Service。实际上，**在 K8s 里有几种不同类型 的 Service**：

- ClusterIP
- NodePort
- LoadBalancer
- ExternalName



本文将主要关注前两种类型。

**K8S 里实现 Service 的组件是 kube-proxy**，实现的主要功能就是**将访问 VIP 的请 求转发（及负载均衡）到相应的后端 pods**。前面提到的那些 iptables 规则就是它创建 和管理的。

另外，kube-proxy 是 K8s 的可选组件，如果不需要 Service 功能，可以不启用它。



#### 1.1 ClusterIP Service

这是 **K8s 的默认 Service 类型**，使得**宿主机或 pod 可以通过 VIP 访问一个 Service**。

- Virtual IP to any endpoint (pod)
- Only in-cluster access

kube-proxy 是通过如下的 iptables 规则来实现这个功能的：

```sh
-t nat -A {PREROUTING, OUTPUT} -m conntrack --ctstate NEW -j KUBE-SERVICES

# 宿主机访问 nginx Service 的流量，同时满足 4 个条件：
# 1. src_ip 不是 Pod 网段
# 2. dst_ip=3.3.3.3/32 (ClusterIP)
# 3. proto=TCP
# 4. dport=80
# 如果匹配成功，直接跳转到 KUBE-MARK-MASQ；否则，继续匹配下面一条（iptables 是链式规则，高优先级在前）
# 跳转到 KUBE-MARK-MASQ 是为了保证这些包出宿主机时，src_ip 用的是宿主机 IP。
-A KUBE-SERVICES ! -s 1.1.0.0/16 -d 3.3.3.3/32 -p tcp -m tcp --dport 80 -j KUBE-MARK-MASQ
# Pod 访问 nginx Service 的流量：同时满足 4 个条件：
# 1. 没有匹配到前一条的，（说明 src_ip 是 Pod 网段）
# 2. dst_ip=3.3.3.3/32 (ClusterIP)
# 3. proto=TCP
# 4. dport=80
-A KUBE-SERVICES -d 3.3.3.3/32 -p tcp -m tcp --dport 80 -j KUBE-SVC-NGINX

# 以 50% 的概率跳转到 KUBE-SEP-NGINX1
-A KUBE-SVC-NGINX -m statistic --mode random --probability 0.50 -j KUBE-SEP-NGINX1
# 如果没有命中上面一条，则以 100% 的概率跳转到 KUBE-SEP-NGINX2
-A KUBE-SVC-NGINX -j KUBE-SEP-NGINX2

# 如果 src_ip=1.1.1.1/32，说明是 Service->client 流量，则
# 需要做 SNAT（MASQ 是动态版的 SNAT），替换 src_ip -> svc_ip，这样客户端收到包时，
# 看到就是从 svc_ip 回的包，跟它期望的是一致的。
-A KUBE-SEP-NGINX1 -s 1.1.1.1/32 -j KUBE-MARK-MASQ
# 如果没有命令上面一条，说明 src_ip != 1.1.1.1/32，则说明是 client-> Service 流量，
# 需要做 DNAT，将 svc_ip -> pod1_ip，
-A KUBE-SEP-NGINX1 -p tcp -m tcp -j DNAT --to-destination 1.1.1.1:80
# 同理，见上面两条的注释
-A KUBE-SEP-NGINX2 -s 1.1.1.2/32 -j KUBE-MARK-MASQ
-A KUBE-SEP-NGINX2 -p tcp -m tcp -j DNAT --to-destination 1.1.1.2:80
```

1. Service 既要能被宿主机访问，又要能被 pod 访问（**二者位于不同的 netns**）， 因此需要在 `PREROUTING` 和 `OUTPUT` 两个 hook 点拦截请求，然后跳转到自定义的 `KUBE-SERVICES` chain；
2. `KUBE-SERVICES` chain **执行真正的 Service 匹配**，依据协议类型、目的 IP 和目的端口号。当匹配到某个 Service 后，就会跳转到专门针对这个 Service 创建的 chain，命名格式为 `KUBE-SVC-<Service>`。
3. `KUBE-SVC-<Service>` chain **根据概率选择某个后端 pod** 然后将请 求转发过去。这其实是一种**穷人的负载均衡器** —— 基于 iptables。选中某个 pod 后，会跳转到这个 pod 相关的一条 iptables chain `KUBE-SEP-<POD>`。
4. `KUBE-SEP-<POD>` chain 会**执行 DNAT**，将 VIP 换成 PodIP。

> 译注：以上解释并不是非常详细和直观，因为这不是本文重点。想更深入地理解基于 iptables 的实现，可参考网上其他一些文章，例如下面这张图所出自的博客 Kubernetes Networking Demystified: A Brief Guide，

![image-20210121211030395](D:\学习资料\笔记\k8s\k8s图\image-20210121211030395.png)



#### 1.2 NodePort Service

这种类型的 Service 也能被宿主机和 pod 访问，但与 ClusterIP 不同的是，**它还能被集群外的服务访问**。

- External node IP + port in NodePort range to any endpoint (pod), e.g. 10.0.0.1:31000
- Enables access from outside

实现上，kube-apiserver 会**从预留的端口范围内分配一个端口给 Service**，然后 **每个宿主机上的 kube-proxy 都会创建以下规则**：

```sh
-t nat -A {PREROUTING, OUTPUT} -m conntrack --ctstate NEW -j KUBE-SERVICES

-A KUBE-SERVICES ! -s 1.1.0.0/16 -d 3.3.3.3/32 -p tcp -m tcp --dport 80 -j KUBE-MARK-MASQ
-A KUBE-SERVICES -d 3.3.3.3/32 -p tcp -m tcp --dport 80 -j KUBE-SVC-NGINX
# 如果前面两条都没匹配到（说明不是 ClusterIP service 流量），并且 dst 是 LOCAL，跳转到 KUBE-NODEPORTS
-A KUBE-SERVICES -m addrtype --dst-type LOCAL -j KUBE-NODEPORTS

-A KUBE-NODEPORTS -p tcp -m tcp --dport 31000 -j KUBE-MARK-MASQ
-A KUBE-NODEPORTS -p tcp -m tcp --dport 31000 -j KUBE-SVC-NGINX

-A KUBE-SVC-NGINX -m statistic --mode random --probability 0.50 -j KUBE-SEP-NGINX1
-A KUBE-SVC-NGINX -j KUBE-SEP-NGINX2
```

1. 前面几步和 ClusterIP Service 一样；如果没匹配到 ClusterIP 规则，则跳转到 `KUBE-NODEPORTS` chain。
2. `KUBE-NODEPORTS` chain 里做 Service 匹配，但**这次只匹配协议类型和目的端口号**。
3. 匹配成功后，转到对应的 `KUBE-SVC-<Service>` chain，后面的过程跟 ClusterIP 是一样的。



#### 1.3 小结

以上可以看到，每个 Service 会对应多条 iptables 规则。

Service 数量不断增长时，**iptables 规则的数量增长会更快**。而且，**每个包都需要 遍历这些规则**，直到最终匹配到一条相应的规则。如果不幸匹配到最后一条规则才命中， 那相比其他流量，这些包就会有**很高的延迟**。

有了这些背景知识，我们来看如何用 BPF/Cilium 来替换掉 kube-proxy，也可以说是 重新实现 kube-proxy 的逻辑。



### 2 用 Cilium/BPF 替换 kube-proxy

我们从 Cilium 早起版本开始，已经逐步用 BPF 实现 Service 功能，但其中仍然有些 地方需要用到 iptables。在这一时期，每台 node 上会同时运行 cilium-agent 和 kube-proxy。

到了 Cilium 1.6，我们已经能**完全基于 BPF 实现，不再依赖 iptables，也不再需要 kube-proxy**。

![image-20210122085937252](D:\学习资料\笔记\k8s\k8s图\image-20210122085937252.png)



#### 2.1 ClusterIP Service

对于 ClusterIP，我们在 BPF 里**拦截 socket 的 connect 和 send 系统调用**；这些 BPF 执行时，**协议层还没开始执行**（这些系统调用 handlers）。

- Attach on the cgroupv2 root mount `BPF_PROG_TYPE_CGROUP_SOCK_ADDR`
- `BPF_CGROUP_INET{4,6}_CONNECT` - TCP, connected UDP



##### TCP & connected UDP

对于 TCP 和 connected UDP 场景，执行的是下面一段逻辑，

```c
int sock4_xlate(struct bpf_sock_addr *ctx) {
 struct lb4_svc_key key = { .dip = ctx->user_ip4, .dport = ctx->user_port };
 svc = lb4_lookup_svc(&key)
  if (svc) {
   ctx->user_ip4 = svc->endpoint_addr;
   ctx->user_port = svc->endpoint_port;
  }
 return 1;
}
```

所做的事情：在 BPF map 中查找 Service，然后做地址转换。但这里的重点是（相比于 TC ingress BPF 实现）：

1. **不经过连接跟踪（conntrack）模块，也不需要修改包头**（实际上这时候还没有包 ），也不再 mangle 包。这也意味着，**不需要重新计算包的 checksum**。
2. 对于 TCP 和 connected UDP，**负载均衡的开销是一次性的**，只需要在 socket 建立 时做一次转换，后面都不需要了，**不存在包级别的转换**。
3. 这种方式是对宿主机 netns 上的 socket 和 pod netns 内的 socket 都是适用的。



##### 某些 UDP 应用：存在的问题及解决方式

但这种方式**对某些 UDP 应用是不适用的**，因为这些 UDP 应用会检查包的源地址，以及 会调用 `recvmsg` 系统调用。

针对这个问题，我们引入了新的 BPF attach 类型：

- `BPF_CGROUP_UDP4_RECVMSG`
- `BPF_CGROUP_UDP6_RECVMSG`

另外还引入了用于 NAT 的 UDP map、rev-NAT map：

```c
    BPF rev NAT map
Cookie   EndpointIP  Port => ServiceID  IP       Port
-----------------------------------------------------
42       1.1.1.1     80   => 1          3.3.3.30 80
```

- 通过 `bpf_get_socket_cookie()` 创建 socket cookie。

  除了 Service 访问方式，还会有一些**客户端通过 PodIP 直连的方式建立 UDP 连接， cookie 就是为了防止对这些类型的流量做 rev-NAT**。

- 在 `connect(2)` 和 `sendmsg(2)` 时更新 map。

- 在 `recvmsg(2)` 时做 rev-NAT。



#### 2.2 NodePort Service

NodePort 会更复杂一些，我们先从最简单的场景看起。



##### 2.2.1 后端 pod 在本节点

![image-20210122090520645](D:\学习资料\笔记\k8s\k8s图\image-20210122090520645.png)

后端 pod 在本节点时，只需要**在宿主机的网络设备上 attach 一段 tc ingress bpf 程序**，这段程序做的事情：

1. Service 查找
2. DNAT
3. redirect 到容器的 lxc0。

对于应答包，lxc0 负责 rev-NAT，FIB 查找（因为我们需要设置 L2 地址，否则会被 drop）， 然后将其 redirect 回客户端。



##### 2.2.2 后端 pod 在其他节点

后端 pod 在其他节点时，会复杂一些，因为要转发到其他节点。这种情况下，**需要在 BPF 做 SNAT**，否则 pod 会直接回包给客户端，而由于不同 node 之间没有做连接跟踪（ conntrack）同步，因此直接回给客户端的包出 pod 后就会被 drop 掉。

所以需要**在当前节点做一次 SNAT**（`src_ip` 从原来的 ClientIP 替换为 NodeIP），让回包也经过 当前节点，然后在这里再做 rev-SNAT（`dst_ip` 从原来的 NodeIP 替换为 ClientIP）。

具体来说，在 **TC ingress** 插入一段 BPF 代码，然后依次执行：Service 查找、DNAT、 选择合适的 egress interface、SNAT、FIB lookup，最后发送给相应的 node，

![image-20210122090930108](D:\学习资料\笔记\k8s\k8s图\image-20210122090930108.png)

现在跨宿主机转发是 SNAT 模式，但将来我们打算支持 **DSR 模式**（译注，Cilium 1.8+ 已经支持了）。DSR 的好处是 **backend pods 直接将包回给客户端**，回包不再经过当前 节点转发。

另外，现在 Service 的处理是在 TC ingress 做的，**这些逻辑其实也能够在 XDP 层实现**， 那将会是另一件激动人心的事情（译注，Cilium 1.8+ 已经支持了，性能大幅提升）。



###### SNAT

当前基于 BPF 的 SNAT 实现中，用一个 LRU BPF map 存放 Service 和 backend pods 的映射信息。

需要说明的是，SNAT **除了替换 src_ip，还可能会替换 src_port**：不同客户端的 src_port 可能是相同的，如果只替换 `src_ip`，不同客户端的应答包在反向转换时就会失 败。因此这种情况下需要做 src_port 转换。现在的做法是，先进行哈希，如果哈希失败， 就调用 `prandom()` 随机选择一个端口。

此外，我们还需要跟踪宿主机上的流（local flows）信息，因此在 Cilium 里**基于 BPF 实现了一个连接跟踪器**（connection tracker），它会监听宿主机的主物理网络设备（ main physical device）；我们也会对宿主机上的应用执行 NAT，pod 流量 NAT 之后使用的 是宿主机的 src_port，而宿主机上的应用使用的也是同一个 src_port 空间，它们可能会 有冲突，因此需要在这里处理。

这就是 NodePort Service 类型的流量到达一台节点后，我们在 BPF 所做的事情。



##### 2.2.3 Client pods 和 backend pods 在同一节点

另外一种情况是：本机上的 pod 访问某个 NodePort Service，而且 backend pods 也在本机。

这种情况下，流量会从 loopback 口转发到 backend pods，中间会经历路由和转发过程， 整个过程对应用是透明的 —— 我们可以**在应用无感知的情况下，修改二者之间的通信方式**， 只要流量能被双方正确地接受就行。因此，我们在这里**使用了 ClusterIP，并对其进 行了一点扩展**，只要连接的 Service 是 loopback 地址或者其他 local 地址，它都能正 确地转发到本机 pods。

另外，比较好的一点是，这种实现方式是基于 cgroups 的，因此独立于 netns。这意味着 我们不需要进入到每个 pod 的 netns 来做这种转换。

![image-20210122091948420](D:\学习资料\笔记\k8s\k8s图\image-20210122091948420.png)



#### 2.3 Service 规则的规模及请求延迟对比

有了以上功能，基本上就可以避免 kube-proxy 那样 per-service 的 iptables 规则了， 每个节点上只留下了少数几条由 Kubernetes 自己创建的 iptables 规则：

```sh
$ iptables-save | grep ‘\-A KUBE’ | wc -l:
```

- With kube-proxy: 25401
- With BPF: 4

在将来，我们有希望连这几条规则也不需要，完全绕开 Netfilter 框架（译注：新版本已经做到了）。

此外，我们做了一些初步的基准测试，如下图所示，

![image-20210122093235215](D:\学习资料\笔记\k8s\k8s图\image-20210122093235215.png)

可以看到，随着 Service 数量从 1 增加到 2000+，**kube-proxy/iptables 的请求延 迟增加了将近一倍**，而 Cilium/eBPF 的延迟几乎没有任何增加。



### 3 相关的 Cilium/BPF 优化

接下来介绍一些我们在实现 Service 过程中的优化工作，以及一些未来可能会做的事情。



#### 3.1 BPF UDP `recvmsg()` hook

实现 socket 层 UDP Service 转换时，我们发现如果只对 UDP `sendmsg` 做 hook ，会导致 **DNS 等应用无法正常工作**，会出现下面这种错误：

![image-20210122093341395](D:\学习资料\笔记\k8s\k8s图\image-20210122093341395.png)

深入分析发现，`nslookup` 及其他一些工具会检查 **connect() 时用的 IP 地址**和 **recvmsg() 读到的 reply message 里的 IP 地址**是否一致。如果不一致，就会报上面的错误。

原因清楚之后，解决就比较简单了：我们引入了一个做反向映射的 BPF hook，对 `recvmsg()` 做额外处理，这个问题就解决了：

![image-20210122093624471](D:\学习资料\笔记\k8s\k8s图\image-20210122093624471.png)

> 983695fa6765 bpf: fix unconnected udp hooks。
>
> 这个 patch 能在不重写包（without packet rewrite）的前提下，会对 BPF ClusterIP 做反向映射（reverse mapping）。



#### 3.2 全局唯一 socket cookie

BPF ClusterIP Service 为 UDP 维护了一个 LRU 反向映射表（reverse mapping table）。

**Socket cookie 是这个映射表的 key 的一部分**，但**这个 cookie 只在每个 netns 内唯一**， 其背后的实现比较简单：每次调用 BPF cookie helper，它都会增加计数器，然后将 cookie 存储到 socket。因此不同 netns 内分配出来的 cookie 值可能会一样，导致冲突。

为解决这个问题，我们将 cookie generator 改成了全局的，见下面的 commit。

> cd48bdda4fb8 sock: make cookie generation global instead of per netns。



#### 3.3 维护邻居表

Cilium agent 从 K8s apiserver 收到 Service 事件时， 会将 backend entry 更新到 datapath 中的 Service backend 列表。

前面已经看到，当 Service 是 NodePort 类型并且 backend 是 remote 时，需要转发到其他节点（TC ingress BPF `redirect()`）。

我们发现**在某些直接路由（direct routing）的场景下，会出现 fib 查找失败的问题** （`fib_lookup()`），原因是系统中没有对应 backend 的 neighbor entry（IP->MAC 映射 信息），并且接下来**不会主动做 ARP 探测**（ARP probe）。

> “
>
> Tunneling 模式下这个问题可以忽略，因为本来发送端的 BPF 程 序就会将 src/dst mac 清零，另一台节点对收到的包做处理时， VxLAN 设备上的另一段 BPF 程序会能够正确的转发这个包，因此这种方式更像是 L3 方式。
>
> ”

我们目前 workaround 了这个问题，解决方式有点丑陋：Cilium 解析 backend，然后直接 将 neighbor entry 永久性地（`NUD_PERMANENT`）插入邻居表中。

目前这样做是没问题的，因为邻居的数量是固定或者可控的（fixed/controlled number of entries）。但后面我们想尝试的是让内核来做这些事情，因为它能以最好的方式处理这个 问题。实现方式就是引入一些新的 `NUD_*` 类型，只需要传 L3 地址，然后内核自己将解 析 L2 地址，并负责这个地址的维护。这样 Cilium 就不需要再处理 L2 地址的事情了。但到今天为止，我并没有看到这种方式的可能性。

对于从集群外来的访问 NodePort Service 的请求，也存在类似的问题， 因为最后将响应流量回给客户端也需要邻居表。由于这些流量都是在 pre-routing，因此我 们现在的处理方式是：自己维护了一个小的 BPF LRU map（L3->L2 mapping in BPF LRU map）；由于这是主处理逻辑（转发路径），流量可能很高，因此将这种映射放到 BPF LRU 是更合适的，不会导致邻居表的 overflow。



#### 3.4 LRU BPF callback on entry eviction

我们想讨论的另一件事情是：在每个 LRU entry 被 eviction（驱逐）时，能有一个 callback 将会更好。为什么呢？

Cilium 中现在有一个 BPF conntrack table，我们支持到了一些非常老的内核版本 ，例如 4.9。Cilium 在启动时会检查内核版本，优先选择使用 LRU，没有 LRU 再 fallback 到普通的哈希表（Hash Table）。**对于哈希表，就需要一个不断 GC 的过程**。

我们**有意将 NAT map 与 CT map 独立开来**，这是因 为我们要求在 **cilium-agent 升级或降级过程中，现有的连接/流量不能受影响**。如果二者是耦合在一起的，假如 CT 相关的东西有很大改动，那升级时那要么 是将当前的连接状态全部删掉重新开始；要么就是服务中断，临时不可用，升级完成后再将 老状态迁移到新状态表，但我认为，要轻松、正确地实现这件事情非常困难。这就是为什么将它们分开的原因。但实际上，GC 在回收 CT entry 的同时， 也会顺便回收 NAT entry。

另外一个问题：**每次从 userspace 操作 conntrack entry 都会破坏 LRU 的正常工作流程**（因为不恰当地更新了所有 entry 的时间戳）。我们通过下面的 commit 解决了这个问题，但要彻底避免这个问题，**最好有一个 GC 以 callback 的方式在第一时 间清理掉这些被 evicted entry**，例如在 CT entry 被 evict 之后，顺便也清理掉 NAT 映射。这是我们正在做的事情（译注，Cilium 1.9+ 已经实现了）。

> 50b045a8c0cc (“bpf, lru: avoid messing with eviction heuristics upon syscall lookup”) fixed map walking from user space



#### 3.5 LRU BPF eviction zones

另一件跟 CT map 相关的比较有意思的探讨：未来**是否能根据流量类型，将 LRU eviction 分割为不同的 zone**？例如，

- 东西向流量分到 zone1：处理 ClusterIP service 流量，都是 pod-{pod,host} 流量， 比较大；
- 南北向流量分到 zone2：处理 NodePort 和 ExternalName service 流量，相对比较小。

这样的好处是：当**对南北向流量 CT 进行操作时，占大头的东西向流量不会受影响**。

理想的情况是这种隔离是有保障的，例如：可以安全地假设，如果正在清理 zone1 内的 entries， 那预期不会对 zone2 内的 entry 有任何影响。不过，虽然分为了多个 zones，但在全局， 只有一个 map。



#### 3.6 BPF 原子操作

另一个要讨论的内容是原子操作。

使用场景之一是**过期 NAT entry 的快速重复利用**（fast recycling）。例如，结合前面的 GC 过程，如果一个连接断开时， 不是直接删除对应的 entry，而是更 新一个标记，表明这条 entry 过期了；接下来如果有新的连接刚好命中了这个 entry，就 直接将其标记为正常（非过期），重复利用（循环）这个 entry，而不是像之前一样从新创 建。

现在基于 BPF spinlock 可以实现做这个功能，但并不是最优的方式，因为如果有合适的原 子操作，我们就能节省两次辅助函数调用，然后将 spinlock 移到 map 里。将 spinlock 放到 map 结构体的额外好处是，每个结构体都有自己独立的结构（互相解耦），因此更能够避免升级/降低导致的问题。

当前内核只有 `BPF_XADD` 指令，我认为它主要适用于计数（counting），因为它并不像原子递增（inc）函数一样返回一个值。此外内核中还有的就是针对 maps 的 spinlock。

我觉得如果有 `READ_ONCE/WRITE_ONCE` 语义将会带来很大便利，现在的 BPF 代码中其实已经有了一些这样功能的、自己实现的代码。此外，我们还需要 `BPF_XCHG, BPF_CMPXCHG` 指 令，这也将带来很大帮助。



#### 3.7 BPF `getpeername` hook

还有一个 hook —— `getpeername()` —— 没有讨论到，它**用在 TCP 和 connected UDP 场 景**，对应用是透明的。

这里的想法是：永远返回 Service IP 而不是 backend pod IP，这样对应用来说，它看到 就是和 Service IP 建立的连接，而不是和某个具体的 backend pod。

现在返回的是 backend IP 而不是 service IP。从应用的角度看，它连接到的对端并不是它期望的。



#### 3.8 绕过内核最大 BPF 指令数的限制

最后再讨论几个非内核的改动（non-kernel changes）。

内核对 **BPF 最大指令数有 4K 条**的限制，现在这个限制已经放大到 **1M**（一百万） 条（但需要 5.1+ 内核，或者稍低版本的内核 + 相应 patch）。

我们的 BPF 程序中包含了 NAT 引擎，因此肯定是超过这个限制的。但 Cilium 这边，我们目前还并未用到这个新的最大限制，而是通过“外包”的方式将 BPF 切分成了子 BPF 程序，然后通过尾调用（tail call）跳转过去，以此来绕过这个 4K 的限 制。

另外，我们当前使用的是 BPF tail call，而不是 BPF-to-BPF call，因为**二者不能同时使用**。更好的方式是，Cilium agent 在启动时进行检查，如果内核支持 1M BPF insns/complexity limit + bounded loops（我们用于 NAT mappings 查询优化），就用这 些新特性；否则回退到尾调用的方式。



### 4 Cilium 上手：用 kubeadm 搭建体验环境

有兴趣尝试 Cilium，可以参考下面的快速安装命令：

```sh
$ kubeadm init --pod-network-cidr=10.217.0.0/16 --skip-phases=addon/kube-proxy
$ kubeadm join [...]
$ helm template cilium \
   --namespace kube-system --set global.nodePort.enabled=true \
   --set global.k8sServiceHost=$API_SERVER_IP \
   --set global.k8sServicePort=$API_SERVER_PORT \
   --set global.tag=v1.6.1 > cilium.yaml
   kubectl apply -f cilium.yaml
```

> 翻译链接：http://arthurchiao.art/blog/cilium-k8s-service-lb-zh/





## 基于 eBPF 实现容器运行时安全



### 1 前言

随着容器技术的发展，越来越多业务甚至核心业务开始采用这一轻量级虚拟化方案。作为一项依然处于发展阶段的新技术，容器的安全性在不断提高，也在不断地受到挑战。天翼云云容器引擎于去年11月底上线，目前已经在22个自研资源池部署上线。天翼云云容器引擎使用 ebpf 技术实现了细粒度容器安全，对主机和容器异常行为进行检测，对有问题的节点和容器进行自动隔离，保证了多租户容器平台容器运行时安全。

BPF 是一项革命性的技术，可在无需编译内核或加载内核模块的情况下，安全地高效地附加到内核的各种事件上，对内核事件进行监控、跟踪和可观测性。BPF 可用于多种用途，如：开发性能分析工具、软件定义网络和安全等。我很荣幸获得今年 openEuler Summit大会的演讲资格，做 BPF 技术知识和实践经验的分享。本文将作为技术分享，从 BPF 技术由来、架构演变、BPF 跟踪、以及容器安全面对新挑战，如何基于 BPF 技术实现容器运行时安全等方面进行介绍。



### 2 初出茅庐：BPF 只是一种数据包过滤技术

BPF 全称是「Berkeley Packet Filter」，中文翻译为「伯克利包过滤器」。它源于 1992 年伯克利实验室，Steven McCanne 和 Van Jacobson 写得一篇名为《The BSD Packet Filter: A New Architecture for User-level Packet Capture》的论文。该论文描述是在 BSD 系统上设计了一种新的用户级的数据包过滤架构。在性能上，新的架构比当时基于栈过滤器的 CSPF 快20倍，比之前 Unix 的数据包过滤器，例如：SunOS 的 NIT（The Network Interface Tap ）快100倍。

BPF 在数据包过滤上引入了两大革新来提高性能：

● BPF 是基于寄存器的过滤器，可以有效地工作在基于寄存器结构的 CPU 之上。

● BPF 使用简单无共享的缓存模型。数据包会先经过 BPF 过滤再拷贝到缓存，缓存不会拷贝所有数据包数据，这样可以最大程度地减少了处理的数据量从而提高性能。

![img](D:\学习资料\笔记\k8s\k8s图\bpf_overview.png)



### 3 Linux 超能力终于到来了：eBPF 架构演变

#### 3.1 eBPF 介绍

2013 年 BPF 技术沉默了 20 年之后，Alexei Starovoitov 提出了对 BPF 进行重大改写。2013 年 9月 Alexei 发布了补丁，名为「extended BPF」。eBPF 实现的最初目标是针对现代硬件进行优化。eBPF 增加了寄存器数量，将原有的 2 个 32 位寄存器增加到 10 个 64 位寄存器。由于寄存器数量和宽度的增加，函数参数可以交换更多的信息，编写更复杂的程序。eBPF 生成的指令集比旧的 BPF 解释器生成的机器码执行速度提高了 4 倍。

当时 BPF 程序仍然限于内核空间使用，只有少数用户空间程序可以编写内核的 BPF 过滤器，例如：tcpdump 和  seccomp 。2014 年 3 月， 经过 Alexei Starovoitov 和 Daniel Borkmann 的进一步开发， Daniel 将 eBPF 提交到 Linux 内核中。2014年 6月 BPF JIT 组件提交到 Linux 3.15 中。2014 年 12 月 系统调用bpf 提交到 Linux 3.18 中。随后，Linux 4.x 加入了 BPF 对 kprobes、uprobe、tracepoints 和 perf_evnets 支持。至此，eBPF 完成了架构演变，eBPF 扩展到用户空间成为了 BPF 技术的转折点。 正如 Alexei 在提交补丁的注释中写到：“这个补丁展示了 eBPF 的潜力”。当前eBPF 不再局限于网络栈，成为内核顶级的子系统。

后来，Alexei将 eBPF改为 BPF。原来的 BPF 就被称为cBPF「classic BPF」。现在 cBPF 已经基本废弃，Linux 内核只运行 eBPF，内核会将加载的 cBPF 字节码透明地转换成 eBPF 再执行。

下面是cBPF和eBPF的对比：

| 纬度           | cBPF                      | eBPF                                                         |
| :------------- | :------------------------ | :----------------------------------------------------------- |
| 内核版本       | Linux 2.1.75（1997年）    | Linux 3.18（2014年）[4.x for kprobe/uprobe/tracepoint/perf-event]（注：虽然eBPF 在 Linux 3.18 版本以后引入，并不代表只能在内核 3.18+ 版本上运行，低版本的内核升级到最新也可以使用 eBPF 能力，只是可能部分功能受限。） |
| 寄存器数目     | 2个：A, X                 | 10个： R0–R9, 另外 R10 是一个只读的帧指针* R0 - eBPF 中内核函数的返回值和退出值* R1 - R5 - eBF 程序在内核中的参数值* R6 - R9 - 内核函数将保存的被调用者callee保存的寄存器* R10 -一个只读的堆栈帧指针 |
| 寄存器宽度     | 32位                      | 64位                                                         |
| 存储           | 16 个内存位: M[0–15]      | 512 字节堆栈，无限制大小的 “map” 存储                        |
| 限制的内核调用 | 非常有限，仅限于 JIT 特定 | 有限，通过 bpf_call 指令调用                                 |
| 目标事件       | 数据包、 seccomp-BPF      | 数据包、内核函数、用户函数、跟踪点 PMCs 等                   |

接下来，让我们来看看演变后的BPF架构。



#### 3.2 eBPF 架构演变

BPF 是一个通用执行引擎，能够高效地安全地执行基于系统事件的特定代码。BPF 内部由字节码指令，存储对象和帮助函数组成。从某种意义上看，BPF和Java虚拟机功能类似。对于Java 开发人员而言，可以使用 javac 将高级编程语言编译成机器代码，Java虚拟机是运行该机器代码的专用程序。相应地，BPF 开发人员可以使用编译器 LLVM 将 C 代码编译成 BPF 字节码，字节码指令在内核执行前必须通过 BPF 验证器进行验证，同时使用内核中的 BPF JIT 模块，将字节码指令直接转成内核可执行的本地指令。编译后的程序附加到内核的各种事件上，以便在 Linux 内核中运行该 BPF 程序。下图是 BPF 架构图：

![img](D:\学习资料\笔记\k8s\k8s图\bpf_architecture.png)

BPF 使内核具有可编程性。BPF 程序是运行在各种内核事件上的小型程序。这与JavaScript 程序有一些相似之处：JavaScript 是允许在浏览器事件，例如：鼠标单击上运行的微型 Web 程序。BPF 是允许内核在系统和应用程序事件，例如：磁盘 I/O 上运行的微型程序。内核运行 BPF 程序之前，需要知道程序附加的执行点。程序执行点是由BPF程序类型确定。通过查看/kernel-src/sample/bpf/bpf_load.c 可以查看 BPF 程序类型。下面是定义在 bpf 头文件中的 bpf 程序类型：

![img](D:\学习资料\笔记\k8s\k8s图\bpf_prog_type.png)

BPF 映射提供了内核和用户空间双向数据共享，允许用户从内核和用户空间读取和写入数据。BPF 映射的数据结构类型可以从简单数组、哈希映射到自定义类型映射。下面是定义在 bpf 头文件中的 bpf 映射类型：



#### 3.3.BPF 与传统 Linux 内核模块的对比

BPF 看上去更像内核模块，所以总是会拿来与 Linux 内核模块方式进行对比，但 BPF 与内核模块不同。BPF 在安全性、入门门槛上及高性能上比内核模块都有优势。

传统 Linux 内核模块开发，内核开发工程师通过直接修改内核代码，每次功能的更新都需要重新编译打包内核代码。内核工程师可以开发即时加载的内核模块，在运行时加载到 Linux 内核中，从而实现扩展内核功能的目的。然而每次内核版本的官方更新，可能会引起内核 API 的变化，因此你编写的内核模块可能会随着每一个内核版本的发布而不可用，这样就必须得为每次的内核版本更新调整你的模块代码，并且，错误的代码会造成内核直接崩溃。

BPF 具有强安全性。BPF 程序不需要重新编译内核，并且 BPF 验证器会保证每个程序能够安全运行，确保内核本身不会崩溃。BPF 虚拟机会使用 BPF JIT 编译器将 BPF 字节码生成本地机器字节码，从而能获得本地编译后的程序运行速度。

下面是 BPF 与 Linux 内核模块的对比：

| 维度                 | Linux 内核模块                       | BPF                                            |
| :------------------- | :----------------------------------- | :--------------------------------------------- |
| kprobes、tracepoints | 支持                                 | 支持                                           |
| 安全性               | 可能引入安全漏洞或导致内核 Panic     | 通过验证器进行检查，可以保障内核安全           |
| 内核函数             | 可以调用内核函数                     | 只能通过 BPF Helper 函数调用                   |
| 编译性               | 需要编译内核                         | 不需要编译内核，引入头文件即可                 |
| 运行                 | 基于相同内核运行                     | 基于稳定 ABI 的 BPF 程序可以编译一次，各处运行 |
| 与应用程序交互       | 打印日志或文件                       | 通过 perf_event 或 map 结构                    |
| 数据结构丰富性       | 一般                                 | 丰富                                           |
| 入门门槛             | 高                                   | 低                                             |
| 升级                 | 需要卸载和加载，可能导致处理流程中断 | 原子替换升级，不会造成处理流程中断             |
| 内核内置             | 视情况而定                           | 内核内置支持                                   |



### 4 BPF 实践中的第一公民：BPF 跟踪

BPF跟踪是 Linux 可观测性的新方法。在BPF 技术的众多应用场景中，BPF 跟踪是应用最广泛的。2013 年 12 月 Alexei 已将 eBPF 用于跟踪。BPF 跟踪支持的各种内核事件包括：kprobes、uprobes、tracepoint 、USDT 和 perf_events：

- kprobes：实现内核动态跟踪。 kprobes 可以跟踪到 Linux 内核中的函数入口或返回点，但是不是稳定 ABI 接口，可能会因为内核版本变化导致，导致跟踪失效。

- uprobes：用户级别的动态跟踪。与 kprobes 类似，只是跟踪的函数为用户程序中的函数。

- tracepoints：内核静态跟踪。tracepoints 是内核开发人员维护的跟踪点，能够提供稳定的 ABI 接口，但是由于是研发人员维护，数量和场景可能受限。

- USDT：为用户空间的应用程序提供了静态跟踪点。

- perf_events：定时采样和 PMC。

![img](D:\学习资料\笔记\k8s\k8s图\bpf_tracing.png)



### 5 容器安全

#### 5.1 容器生态链带来新挑战

虚拟机（VM）是一个物理硬件层抽象，用于将一台服务器变成多台服务器。管理程序允许多个VM在一台机器上运行。每个VM都包含一整套操作系统、一个或多个应用、必要的二进制文件和库资源，因此占用大量空间。VM启动也较慢。

容器是一种应用层抽象，用于将代码和依赖资源打包在一起。多个容器可以在同一台机器上运行，共享操作系统内核，但各自作为独立的进程在用户空间中运行。与虚拟机相比，容器占用的空间比较少（容器镜像大小通常只有几十兆），瞬间就能完成启动。

容器技术面临的新挑战：

- 容器共享宿主机内核，隔离性相对较弱！

- 有 root 权限的用户可以访问所有容器资源！某容器提权后可能影响全局！

- 容器在主机网络之上构建了一层Overlay 网络，使容器间的互访避开了传统网络安全的防护！

- 容器的弹性伸缩性，使有些容器只是短暂运行，短暂运行的容器行为异常不容易被发现！

- 容器和容器编排给系统增加了新的元素,带来新的风险!

![img](D:\学习资料\笔记\k8s\k8s图\container.png)



#### 5.2 容器安全事故：容器逃逸

在容器安全问题中，容器逃逸是最为严重，它直接影响到了承载容器的底层基础设施的保密性、完整性和可用性。下面的情况会导致容器逃逸：

- 危险配置导致容器逃逸。在这些年的迭代中，容器社区一直在努力将「纵深防御」、「最小权限」等理念和原则落地。例如，Docker已经将容器运行时的Capabilities黑名单机制改为如今的默认禁止所有Capabilities，再以白名单方式赋予容器运行所需的最小权限。如果容器启动，配置危险能力，或特权模式容器，或容器以root用户权限运行都会导致容器逃逸。下面是容器运行时默认的最小权限。
 ![img](D:\学习资料\笔记\k8s\k8s图\docker_default_capability.png)

- 危险挂载导致容器逃逸。Docker Socket 是 Docker 守护进程监听的 Unix 域套接字，用来与守护进程通信——查询信息或下发命令。如果在攻击者可控的容器内挂载了该套接字文件（/var/run/docker.sock），容器逃逸就相当容易了，除非有进一步的权限限制。

![img](D:\学习资料\笔记\k8s\k8s图\container_escape.png)

下面通过一个小实验来展示这种逃逸可能性：

1. 准备dockertest镜像，该镜像是基于ubuntu镜像安装docker，通过docker commit生成

2. 创建一个容器并挂载/var/run/docker.sock

```sh
[root@bpftest ~]$ docker run -itd --name with_docker_sock -v /var/run/docker.sock:/var/run/docker.sock dockertest
```

3. 接着使用该客户端通过Docker Socket与Docker守护进程通信，发送命令创建并运行一个新的容器，将宿主机的根目录挂载到新创建的容器内部；

```sh
[root@bpftest ~]$ docker exec -it <CONTAINER_ID> /bin/bash
[root@bpftest ~]$ docker ps
[root@bpftest ~]$ docker run -it -v /:/host dockertest /bin/bash
```

![img](D:\学习资料\笔记\k8s\k8s图\container_escape_1.png)

-  相关程序漏洞导致容器逃逸，例如：runc漏洞CVE-2019-5736 导致容器逃逸。

- 内核漏洞导致容器逃逸，例如：Copy_on_Write脏牛漏洞，向vDSO内写入shellcode并劫持正常函数的调用过程，导致容器逃逸。

下面是2019年排名最高的容器安全事故列表：

![img](D:\学习资料\笔记\k8s\k8s图\container_runtime_violation.png)



### 6 容器安全主控引擎

#### 6.1 主机和容器异常活动的检测

确保容器运行时安全的关键点：

- 降低容器的攻击面，每个容器以最小权限运行，包括容器挂载的文件系统、网络控制、运行命令等。

- 确保每个用户对不同的容器拥有合适的权限。

- 使用安全容器控制容器之间的访问。当前，Linux的Docker容器技术隔离技术采用Namespace 和 Cgroups 无法阻止提权攻击或打破沙箱的攻击。

- 使用ebpf跟踪技术自动生成容器访问控制权限。包括：容器对文件的可疑访问，容器对系统的可疑调用，容器之间的可疑互访，检测容器的异常进程，对可疑行为进行取证。例如：
  - 检测容器运行时是否创建其他进程。
  - 检测容器运行时是否存在文件系统读取和写入的异常行为，例如在运行的容器中安装了新软件包或者更新配置。
  - 检测容器运行时是否打开了新的监听端口或者建立意外连接的异常网络活动。
  - 检测容器中用户操作及可疑的shell脚本的执行。

![img](D:\学习资料\笔记\k8s\k8s图\ebpf_container_security.png)



#### 6.2 隔离有问题的容器和节点

如果检测到有问题的容器或节点，可以将节点设置为维护状态，或使用网络策略隔离有问题的容器，或将deployment的副本数设置为0，删除有问题的容器。同时，使用sidecar WAF，进行虚拟补丁等进行容器应用安全防范。

![img](D:\学习资料\笔记\k8s\k8s图\container_isolation.png)



#### 6.3 大型公有云容器平台安全主控引擎

下面是天翼云云容器引擎产品为了保证容器运行时安全实现的安全主控引擎：

- Pod 通过 sidecar 注入 WAF组件对容器进行深度攻击防御。

- 容器安全代理 Sage 组件以Daemonset形式部署在各个节点上，用来收集容器和主机异常行为，并通过自己的 sidecar 推送到消息队列中。

- 安全主控引擎组件 jasmine 从消息队列中拉取事件，对数据进行分析，对有故障的容器和主机进行隔离。并将事件推送给SIEM安全信息事件管理平台进行管理。

![img](https://www.ebpf.top/post/ebpf_container_security/imgs/secuirty_engine.png)



### 参考资料：

1. [【云原生攻防研究】容器逃逸技术概览](https://mp.weixin.qq.com/s/_GwGS0cVRmuWEetwMesauQ)
2. [GKE Security using Falco, Pub/Sub, Cloud Functions | Sysdig](https://sysdig.com/blog/gke-security-using-falco/)
3. [Kubernetes Security monitoring at scale with Sysdig Falco](https://medium.com/@SkyscannerEng/kubernetes-security-monitoring-at-scale-with-sysdig-falco-a60cfdb0f67a)
4. [Enterprise Falco Open-Source Cloud-Native Security Project | Sysdig](https://sysdig.com/opensource/falco/)
5. [容器逃逸成真：从CTF解题到CVE-2019-5736漏洞挖掘分析](https://mp.weixin.qq.com/s/UZ7VdGSUGSvoo-6GVl53qg)





## Linux网络新技术基石 |eBPF and XDP

今天给大家分享的是Linux 网络新技术，当前正流行网络技是什么？那就是eBPF和XDP技术，Cilium+eBPF超级火热，Google GCP也刚刚全面转过来。



### 新技术出现的历史原因

**iptables/netfilter**

iptables/netfilter 是上个时代Linux网络提供的优秀的防火墙技术，扩展性强，能够满足当时大部分网络应用需求，如果不知道iptables/netfilter是什么**，**请参考之前文章：[一个奇葩的网络问题，把技术砖家"搞蒙了"](http://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247506496&idx=1&sn=c629e22f0de944c0940ffb3a665b726f&chksm=c1842d11f6f3a407e2200d28da9033c23a411bdc64f85ddb756c0ff36d660eed38338e611d1f&scene=21#wechat_redirect) ，里面对iptables/netfilter技术有详细介绍。

但该框架也存在很多明显问题：

- 路径太长

**netfilter** 框架在IP层，报文需要经过链路层，IP层才能被处理，如果是需要丢弃报文，会白白浪费很多CPU资源，影响整体性能；

- O(N)匹配

![图片](D:\学习资料\笔记\k8s\k8s图\da)

如上图所示，极端情况下，报文需要依次遍历所有规则，才能匹配中，极大影响报文处理性能；

- 规则太多

netfilter 框架类似一套可以自由添加策略规则专家系统，并没有对添加规则进行合并优化，这些都严重依赖操作人员技术水平，随着规模的增大，规则数量n成指数级增长，而报文处理又是0（n）复杂度，最终性能会直线下降。

**内核协议栈**

![图片](D:\学习资料\笔记\k8s\k8s图\内核协议栈)

随着互联网流量越来愈大, 网卡性能越来越强，Linux内核协议栈在10Mbps/100Mbps网卡的慢速时代是没有任何问题的，那个时候应用程序大部分时间在等网卡送上来数据。

现在到了1000Mbps/10Gbps/40Gbps网卡的时代，数据被很快地收入，协议栈复杂处理逻辑，效率捉襟见肘，把大量报文堵在内核里。

- 各类链表在多CPU环境下的同步开销。

- 不可睡眠的软中断路径过长。

- sk_buff的分配和释放。

- 内存拷贝的开销。

- 上下文切换造成的cache miss。

于是，内核协议栈各种优化措施应着需求而来：

- 网卡RSS，多队列。

- 中断线程化。

- 分割锁粒度。

- Busypoll。

但却都是见招拆招，治标不治本。问题的根源不是这些机制需要优化，而是这些机制需要推倒重构。蒸汽机车刚出来的时候，马车夫为了保持竞争优势，不是去换一匹昂贵的快马，而是卖掉马去买一台蒸汽机装上。基本就是这个意思。

重构的思路很显然有两个：

**upload方法**：别让应用程序等内核了，让应用程序自己去网卡直接拉数据。

**offload方法**：别让内核处理网络逻辑了，让网卡自己处理。

总之，绕过内核就对了，内核协议栈背负太多历史包袱。

**DPDK**让用户态程序直接处理网络流，bypass掉内核，使用独立的CPU专门干这个事。

**XDP**让灌入网卡的eBPF程序直接处理网络流，bypass掉内核，使用网卡NPU专门干这个事**。**

如此一来，内核协议栈就不再参与数据平面的事了，留下来专门处理诸如路由协议，远程登录等控制平面和管理平面的数据流。

改善iptables/netfilter的规模瓶颈，提高Linux内核协议栈IO性能，内核需要提供新解决方案，那就是eBPF/XDP框架，让我们来看一看，这套框架是如何解决问题的。



### eBPF到底是什么?

#### eBPF的历史

BPF 是 Linux 内核中高度灵活和高效的类似虚拟机的技术，允许以安全的方式在各个挂钩点执行字节码。它用于许多 Linux 内核子系统，最突出的是网络、跟踪和安全（例如沙箱）。

![图片](D:\学习资料\笔记\k8s\k8s图\ebpf)

BPF 是一个通用目的 RISC 指令集，其最初的设计目标是：用 C 语言的一个子集编 写程序，然后用一个编译器后端（例如 LLVM）将其编译成 BPF 指令，稍后内核再通 过一个位于内核中的（in-kernel）即时编译器（JIT Compiler）将 BPF 指令映射成处理器的原生指令（opcode ），以取得在内核中的最佳执行性能。



![图片](D:\学习资料\笔记\k8s\k8s图\tcpdump1)

**BPF指令**

尽管 BPF 自 1992 年就存在，扩展的 Berkeley Packet Filter (eBPF) 版本首次出现在 Kernel3.18中，如今被称为“经典”BPF (cBPF) 的版本已过时。许多人都知道 cBPF是tcpdump使用的数据包过滤语言。现在Linux内核只运行 eBPF，并且加载的 cBPF 字节码在程序执行之前被透明地转换为内核中的eBPF表示。除非指出 eBPF 和 cBPF 之间的明确区别，一般现在说的BPF就是指eBPF。



### eBPF总体设计

![图片](D:\学习资料\笔记\k8s\k8s图\eBPF总体设计)

- BPF 不仅通过提供其指令集来定义自己，而且还通过提供围绕它的进一步基础设施，例如充当高效键/值存储的映射、与内核功能交互并利用内核功能的辅助函数、调用其他 BPF 程序的尾调用、安全加固原语、用于固定对象（地图、程序）的伪文件系统，以及允许将 BPF 卸载到网卡的基础设施。
- LLVM 提供了一个 BPF后端，因此可以使用像 clang 这样的工具将 C 编译成 BPF 目标文件，然后可以将其加载到内核中。BPF与Linux 内核紧密相连，允许在不牺牲本机内核性能的情况下实现完全可编程。

eBPF总体设计包括以下几个部分：



#### eBPF Runtime

![图片](D:\学习资料\笔记\k8s\k8s图\eBPF Runtime)

- 安全保障 ： eBPF的verifier 将拒绝任何不安全的程序并提供沙箱运行环境
- 持续交付： 程序可以更新在不中断工作负载的情况下
- 高性能：JIT编译器可以保证运行性能



#### eBPF Hooks

![图片](D:\学习资料\笔记\k8s\k8s图\eBPF Hooks)

**内核函数 (kprobes)、用户空间函数 (uprobes)、系统调用、fentry/fexit、跟踪点、网络设备 (tc/xdp)、网络路由、TCP 拥塞算法、套接字（数据面）**



#### eBPF Maps

![图片](D:\学习资料\笔记\k8s\k8s图\eBPF Maps)

Map 类型

- Hash tables, Arrays

- LRU (Least Recently Used)

- Ring Buffer

- Stack Trace

- LPM (Longest Prefix match)



作用

- 程序状态
- 程序配置
- 程序间共享数据
- 和用户空间共享状态、指标和统计



#### eBPF Helpers

![图片](D:\学习资料\笔记\k8s\k8s图\eBPF Helpers)

有哪些Helpers？

- 随机数
- 获取当前时间
- map访问
- 获取进程/cgroup 上下文
- 处理网络数据包和转发
- 访问套接字数据
- 执行尾调用
- 访问进程栈
- 访问系统调用参数
- ...



#### eBPF Tail and Function Calls

![图片](D:\学习资料\笔记\k8s\k8s图\eBPF Tail and Function Calls)

尾调用有什么用？

● 将程序链接在一起

● 将程序拆分为独立的逻辑组件

● 使 BPF 程序可组合



函数调用有什么用？

● 重用内部的功能程序

● 减少程序大小（避免内联）



#### eBPF JIT Compiler

![图片](D:\学习资料\笔记\k8s\k8s图\eBPF JIT Compiler)

- 确保本地执行性能而不需要了解CPU
- 将 BPF字节码编译到CPU架构特定指令集



#### eBPF可以做什么？

![图片](D:\学习资料\笔记\k8s\k8s图\eBPF可以做什么)



#### eBPF 开源 Projects

![图片](D:\学习资料\笔记\k8s\k8s图\eBPF 开源 Projects)



### Cilium

![图片](D:\学习资料\笔记\k8s\k8s图\Cilium)



- Cilium 是开源软件，用于Linux容器管理平台（如 Docker 和 Kubernetes）部署的服务之间的透明通信和提供安全隔离保护。
- Cilium基于微服务的应用，使用HTTP、gRPC、Kafka等轻量级协议API相互通信。

![图片](D:\学习资料\笔记\k8s\k8s图\cilium networking)

- Cilium 的基于 eBPF 的新 Linux 内核技术，它能够在 Linux 本身中动态插入强大的安全可见性和控制逻辑。由于 eBPF 在 Linux 内核中运行，因此可以在不更改应用程序代码或容器配置的情况下应用和更新 Cilium 安全策略。



#### Cilium在它的 datapath 中重度使用了 BPF 技术

![图片](D:\学习资料\笔记\k8s\k8s图\Cilium datapath)

- Cilium 是位于 Linux kernel 与容器编排系统的中间层。向上可以为容器配置网络，向下可以向 Linux 内核生成 BPF 程序来控制容器的安全性和转发行为。
- 利用 Linux BPF，Cilium 保留了透明地插入安全可视性 + 强制执行的能力，但这种方式基于服务 /pod/ 容器标识（与传统系统中的 IP 地址识别相反），并且可以根据应用层进行过滤 （例如 HTTP）。因此，通过将安全性与寻址分离，Cilium 不仅可以在高度动态的环境中应用安全策略，而且除了提供传统的第 3 层和第 4 层分割之外，还可以通过在 HTTP 层运行来提供更强的安全隔离。
- BPF 的使用使得 Cilium 能够以高度可扩展的方式实现以上功能，即使对于大规模环境也不例外。



对比传统容器网络（采用iptables/netfilter）：

![图片](D:\学习资料\笔记\k8s\k8s图\netfilter)

- eBPF主机路由允许绕过主机命名空间中所有的 iptables 和上层网络栈，以及穿过Veth对时的一些上下文切换，以节省资源开销。网络数据包到达网络接口设备时就被尽早捕获，并直接传送到Kubernetes Pod的网络命名空间中。在流量出口侧，数据包同样穿过Veth对，被eBPF捕获后，直接被传送到外部网络接口上。eBPF直接查询路由表，因此这种优化完全透明。
- 基于eBPF中的kube-proxy网络技术正在替换基于iptables的kube-proxy技术，与Kubernetes中的原始kube-proxy相比，eBPF中的kuber-proxy替代方案具有一系列重要优势，例如更出色的性能、可靠性以及可调试性等等。



### BCC(BPF Compiler Collection)

BCC 是一个框架，它使用户能够编写嵌入其中的 eBPF 程序的 Python 程序。该框架主要针对涉及应用程序和系统分析/跟踪的用例，其中 eBPF 程序用于收集统计信息或生成事件，用户空间中的对应部分收集数据并以人类可读的形式显示。运行 python 程序将生成 eBPF 字节码并将其加载到内核中。

![图片](D:\学习资料\笔记\k8s\k8s图\BPF Compiler Collection)



#### bpftrace

bpftrace 是一种用于 Linux eBPF 的高级跟踪语言，可在最近的 Linux 内核 (4.x) 中使用。bpftrace 使用 LLVM 作为后端将脚本编译为 eBPF 字节码，并利用 BCC 与 Linux eBPF 子系统以及现有的 Linux 跟踪功能进行交互：内核动态跟踪 (kprobes)、用户级动态跟踪 (uprobes) 和跟踪点. bpftrace 语言的灵感来自 awk、C 和前身跟踪器，例如 DTrace 和 SystemTap。

![图片](D:\学习资料\笔记\k8s\k8s图\bpftrace)



#### eBPF Go 库

eBPF Go 库提供了一个通用的 eBPF 库，它将获取 eBPF 字节码的过程与 eBPF 程序的加载和管理解耦。eBPF 程序通常是通过编写高级语言创建的，然后使用 clang/LLVM 编译器编译为 eBPF 字节码。

![图片](D:\学习资料\笔记\k8s\k8s图\eBPF Go)



#### libbpf C/C++ 库

libbpf 库是一个基于 C/C++ 的通用 eBPF 库，它有助于解耦从 clang/LLVM 编译器生成的 eBPF 目标文件加载到内核中，并通过提供易于使用的库 API 来抽象与 BPF 系统调用的交互应用程序。

![图片](D:\学习资料\笔记\k8s\k8s图\libbpf C C++ 库)



### 那XDP又是什么?

XDP的全称是： **eXpress Data Path**

XDP 是Linux 内核中提供高性能、可编程的网络数据包处理框架。

![图片](D:\学习资料\笔记\k8s\k8s图\XDP整体框架)

- 直接接管网卡的RX数据包（类似DPDK用户态驱动）处理；
- 通过运行BPF指令快速处理报文；
- 和Linux协议栈无缝对接；



### XDP总体设计

![图片](D:\学习资料\笔记\k8s\k8s图\XDP总体设计)

XDP总体设计包括以下几个部分：

**XDP驱动**

网卡驱动中XDP程序的一个挂载点，每当网卡接收到一个数据包就会执行这个XDP程序；XDP程序可以对数据包进行逐层解析、按规则进行过滤，或者对数据包进行封装或者解封装，修改字段对数据包进行转发等；

**BPF虚拟机**

并没有在图里画出来，一个XDP程序首先是由用户编写用受限制的C语言编写的，然后通过clang前端编译生成BPF字节码，字节码加载到内核之后运行在eBPF虚拟机上，虚拟机通过即时编译将XDP字节码编译成底层二进制指令；eBPF虚拟机支持XDP程序的动态加载和卸载；

**BPF maps**

存储键值对，作为用户态程序和内核态XDP程序、内核态XDP程序之间的通信媒介，类似于进程间通信的共享内存访问；用户态程序可以在BPF映射中预定义规则，XDP程序匹配映射中的规则对数据包进行过滤等；XDP程序将数据包统计信息存入BPF映射，用户态程序可访问BPF映射获取数据包统计信息；

**BPF程序校验器**

XDP程序肯定是我们自己编写的，那么如何确保XDP程序加载到内核之后不会导致内核崩溃或者带来其他的安全问题呢？程序校验器就是在将XDP字节码加载到内核之前对字节码进行安全检查，比如判断是否有循环，程序长度是否超过限制，程序内存访问是否越界，程序是否包含不可达的指令；



### XDP Action

XDP用于报文的处理，支持如下action：

```c
enum xdp_action {
    XDP_ABORTED = 0,
    XDP_DROP,
    XDP_PASS,
    XDP_TX,
    XDP_REDIRECT,
};
```

- **XDP_DROP**：在驱动层丢弃报文，通常用于实现DDos或防火墙

- **XDP_PASS**：允许报文上送到内核网络栈，同时处理该报文的CPU会分配并填充一个skb，将其传递到GRO引擎。之后的处理与没有XDP程序的过程相同。

- **XDP_TX**：从当前网卡发送出去。

- **XDP_REDIRECT**：从其他网卡发送出去。

- **XDP_ABORTED**：表示程序产生了异常，其行为和 XDP_DROP相同，但 XDP_ABORTED 会经过 trace_xdp_exception tracepoint，因此可以通过 tracing 工具来监控这种非正常行为。

  



### AF_XDP

AF_XDP 是为高性能数据包处理而优化的地址族，AF_XDP 套接字使 XDP 程序可以将帧重定向到用户空间应用程序中的内存缓冲区。

**XDP设计原则**

- XDP 专为高性能而设计。它使用已知技术并应用选择性约束来实现性能目标

- XDP 还具有可编程性。无需修改内核即可即时实现新功能

- XDP 不是内核旁路。它是内核协议栈的快速路径

- XDP 不替代TCP/IP 协议栈。与协议栈协同工作

- XDP 不需要任何专门的硬件。它支持网络硬件的少即是多原则

  

**XDP技术优势**

**及时处理**

- 在网络协议栈前处理，由于 XDP 位于整个 Linux 内核网络软件栈的底部，能够非常早地识别并丢弃攻击报文，具有很高的性能。可以改善 iptables 协议栈丢包的性能瓶颈
- DDIO
- Packeting steering
- 轮询式

**高性能优化**

- 无锁设计
- 批量I/O操作
- 不需要分配skbuff
- 支持网络卸载
- 支持网卡RSS

**指令虚拟机**

- 规则优化，编译成精简指令，快速执行
- 支持热更新，可以动态扩展内核功能
- 易编程-高级语言也可以间接在内核运行
- 安全可靠，BPF程序先校验后执行，XDP程序没有循环

**可扩展模型**

- 支持应用处理（如应用层协议GRO）
- 支持将BPF程序卸载到网卡
- BPF程序可以移植到用户空间或其他操作系统

**可编程性**

- 包检测，BPF程序发现的动作
- 灵活（无循环）协议头解析
- 可能由于流查找而有状态
- 简单的包字段重写（encap/decap）



### XDP 工作模式

XDP 有三种工作模式，默认是 `native`（原生）模式，当讨论 XDP 时通常隐含的都是指这 种模式。

- Native XDP

  默认模式，在这种模式中，XDP BPF 程序直接运行在网络驱动的早期接收路径上（ early receive path）。

- Offloaded XDP

  在这种模式中，XDP BPF程序直接 offload 到网卡。

- Generic XDP

  对于还没有实现 native 或 offloaded XDP 的驱动，内核提供了一个 generic XDP 选 项，这种设置主要面向的是用内核的 XDP API 来编写和测试程序的开发者，对于在生产环境使用XDP，推荐要么选择native要么选择offloaded模式。



### XDP vs DPDK

![图片](D:\学习资料\笔记\k8s\k8s图\XDP vs DPDK)



相对于DPDK，XDP：



**优点**

- 无需第三方代码库和许可

- 同时支持轮询式和中断式网络

- 无需分配大页

- 无需专用的CPU

- 无需定义新的安全网络模型

  

**缺点**

注意XDP的性能提升是有代价的，它牺牲了通用型和公平性

- XDP不提供缓存队列（qdisc），TX设备太慢时直接丢包，因而不要在RX比TX快的设备上使用XDP

- XDP程序是专用的，不具备网络协议栈的通用性

  

**如何选择？**

- 内核延伸项目，不想bypass内核的下一代高性能方案；

- 想直接重用内核代码；

- 不支持DPDK程序环境；

  

**XDP适合场景**

- DDoS防御
- 防火墙
- 基于XDP_TX的负载均衡
- 网络统计
- 流量监控
- 栈前过滤/处理
- ...



**XDP例子**

下面是一个最小的完整 XDP 程序，实现丢弃包的功能（`xdp-example.c`）：

```C
#include <linux/bpf.h>

#ifndef __section
# define __section(NAME)                  \
   __attribute__((section(NAME), used))
#endif

__section("prog")
int xdp_drop(struct xdp_md *ctx)
{
    return XDP_DROP;
}

char __license[] __section("license") = "GPL";
```



用下面的命令编译并加载到内核：

```sh
$ clang -O2 -Wall -target bpf -c xdp-example.c -o xdp-example.o
$ ip link set dev em1 xdp obj xdp-example.o
```

> 以上命令将一个 XDP 程序 attach 到一个网络设备，需要是 Linux 4.11 内核中支持 XDP 的设备，或者 4.12+ 版本的内核。



### 最后

eBPF/XDP 作为Linux网络革新技术正在悄悄改变着Linux网络发展模式。

eBPF正在将Linux内核转变为微内核，越来越多的新内核功能采用eBPF实现，让新增内核功能更加快捷高效。

总体而言，基于业界基准测试结果，eBPF 显然是解决具有挑战性的云原生需求的最佳技术。



### 参考&延伸阅读

The eXpress Data Path: 

Fast Programmable Packet Processing in the Operating System Kernel

https://docs.cilium.io/en/v1.6/bpf/bpf-rethinkingthelinuxkernel-200303183208

https://ebpf.io/what-is-ebpf/

https://www.kernel.org/doc/html/latest/networking/af_xdp.html

https://cilium.io/blog/2021/05/11/cni-benchmark

https://blog.csdn.net/dog250/article/details/107243696





## 大规模微服务利器：eBPF + Kubernetes



微服务，云原生近来大热，在企业积极进行数字化转型，全面提升效率的今天，几乎无人否认云原生代表着云计算的“下一个时代”，IT大厂们都不约而同的将其视为未来云应用的发展方向，从网络方向来看，eBPF可能会成为“为云而生”下一代网络技术。



### 译者序

本文翻译自 2020 年 Daniel Borkmann 在 KubeCon 的一篇分享: eBPF and Kubernetes: Little Helper Minions for Scaling Microservices， 视频见油管。翻译已获得 Daniel 授权。

Daniel 是 eBPF 两位 maintainer 之一，目前在 eBPF commits 榜单上排名第一，也是 Cilium 的核心开发者之一。

本文内容的时间跨度有 8 年，覆盖了 eBPF 发展的整个历史，非常值得一读。时间限制， Daniel 很多地方只是点到，没有展开。译文中加了一些延展阅读，有需要的同学可以参考。

由于译者水平有限，本文不免存在遗漏或错误之处。如有疑问，请查阅原文。

http://arthurchiao.art/blog/ebpf-and-k8s-zh



### 1 eBPF 正在吞噬世界

#### 1.1 Kubernetes 已经是云操作系统

Kubernetes 正在吞噬世界（eating the world）。越来越多的企业开始迁移到容器平台上 ，而 Kubernetes 已经是公认的云操作系统（Cloud OS）。从技术层面来说，

1. Linux 内核是一切的坚实基础，例如，内核提供了 cgroup、namespace 等特性。
2. Kubernetes CNI 插件串联起了关键路径（critical path）上的组件。例如，从网络的 视角看，包括，
   - 广义的 Pod 连通性：一个容器创建之后，CNI 插件会给它创建网络设备，移动到容器的网络命名空间。
   - IPAM：CNI 向 IPAM 发送请求，为容器分配 IP 地址，然后配置路由。
   - Kubernetes 的 Service 处理和负载均衡功能。
   - 网络策略的生效（network policy enforcement）。
   - 监控和排障。



#### 1.2 两个清晰的容器技术趋势

今天我们能清晰地看到两个技术发展趋势：

1. 容器的部署密度越来越高（increasing Pod density）。
2. 容器的生命周期越来越短（decreasing Pod lifespan）。甚至短到秒级或毫秒级。

大家有兴趣的可以查阅相关调查：

- CNCF’19 survey report
- sysdig’19 container usage report



### 2 内核面临的挑战

从操作系统内核的角度看，我们面临很多挑战。



#### 2.1 复杂度性不断增长，性能和可扩展性新需求

内核，或者说通用内核（general kernel），必须在子系统复杂度不断增长（ increasing complexity of kernel subsystems）的前提下，满足这些性能和可扩展性需求（performance & scalability requirements）。



#### 2.2 永远保持后向兼容

Linus torvalds 的名言大家都知道：never break user space。

这对用户来说是好事，但对内核开发者来说意味着：我们必须保证在引入新代码时，五年前甚至十几年前的老代码仍然能正常工作。

显然，这会使原本就复杂的内核变得更加复杂，对于网络来说，这意味着快速收发包路径 （fast path）将受到影响。



#### 2.3 Feature creeping normality

开发者和用户不断往内核加入新功能，导致**内核非常复杂，现在已经没有一个人能理解所有东西了。**

Wikipedia 对 creeping normality 的定义：

> Def. creeping normality: … is a process by which a major change can be accepted as normal and acceptable if it happens slowly through small, often unnoticeable, increments of change. The change could otherwise be regarded as objectionable if it took place in a single step or short period.

应用到这里，意思就是：内核不允许一次性引入非常大的改动，只能将它们拆分成数量众多的小 patch，每次合并的 patch 保证系统后向兼容，并且对系统的影响非常小。

![image-20210615151414894](D:\学习资料\笔记\k8s\k8s图\image-20210615151414894.png)

来看 Linus torvalds 的原话：

> Linus Torvalds on crazy new kernel features:
>
> So I can work with crazy people, that’s not the problem. They just need to *sell* their crazy stuff to me using non-crazy arguments, and in small and well-defined pieces. When I ask for killer features, I want them to lull me into a safe and cozy world where the stuff they are pushing is actually useful to mainline people *first*.
>
> In other words, every new crazy feature should be hidden in a nice solid “Trojan Horse” gift: something that looks *obviously* good at first sight.
>
> Linus Torvalds,
>
> https://lore.kernel.org/lkml/alpine.LFD.2.00.1001251002430.3574@localhost.localdomain/



### 3 eBPF 降世：重新定义数据平面（datapth）

这就是我们最开始想将 eBPF 合并到内核时遇到的问题：改动太大，功能太新（a crazy new kernel feature）。

但是，eBPF 带来的好处也是无与伦比的。

首先，从长期看，eBPF 这项新功能会减少未来的 feature creeping normality。因为用户或开发者希望内核实现的功能，以后不需要再通过改内核的方式来实现了。只需要一段 eBPF 代码，实时动态加载到内核就行了。

其次，因为 eBPF，内核也不会再引入那些影响 fast path 的蹩脚甚至 hardcode 代码 ，从而也避免了性能的下降。

第三，eBPF 还使得内核**完全可编程，安全地可编程**（fully and safely programmable ），用户编写的 eBPF 程序不会导致内核 crash。另外，eBPF 设计用来解决真实世界 中的线上问题，而且我们现在仍然在坚守这个初衷。



### 4 eBPF 长什么样，怎么用？Cilium eBPF networking 案例研究

eBPF 程序长什么样？如下图所示，和 C 语言差不多，一般由用户空间 application 或 agent 来生成。



#### 4.1 Cilium eBPF 流程

下面我们将看看 Cilium 是如何用 eBPF 实现容器网络方案的。

![image-20210615153141381](D:\学习资料\笔记\k8s\k8s图\image-20210615153141381.png)

如上图所示，几个步骤：

1. Cilium agent 生成 eBPF 程序。
2. 用 LLVM 编译 eBPF 程序，生成 eBPF 对象文件（object file，`*.o`）。
3. 用 eBPF loader 将对象文件加载到 Linux 内核。
4. 校验器（verifier）对 eBPF 指令会进行合法性验证，以确保程序是安全的，例如 ，无非法内存访问、不会 crash 内核、不会有无限循环等。
5. 对象文件被即时编译（JIT）为能直接在底层平台（例如 x86）运行的 native code。
6. 如果要在内核和用户态之间共享状态，BPF 程序可以使用 BPF map，这是一种共享存储 ，BPF 侧和用户侧都可以访问。
7. BPF 程序就绪，等待事件触发其执行。对于这个例子，就是有数据包到达网络设备时，触发 BPF 程序的执行。
8. BPF 程序对收到的包进行处理，例如 mangle。最后返回一个裁决（verdict）结果。
9. 根据裁决结果，如果是 DROP，这个包将被丢弃；如果是 PASS，包会被送到更网络栈的 更上层继续处理；如果是重定向，就发送给其他设备。



#### 4.2 eBPF 特点

1. 最重要的一点：不能 crash 内核。
2. 执行起来，与内核模块（kernel module）一样快。
3. 提供稳定的 API。

![image-20210615153719417](D:\学习资料\笔记\k8s\k8s图\image-20210615153719417.png)

这意味着什么？简单来说，如果一段 BPF 程序能在老内核上执行，那它一定也能继续在新内核上执行，而无需做任何修改。

这就像是内核空间与用户空间的契约，内核保证对用户空间应用的兼容性，类似地，内核也 会保证 eBPF 程序的兼容性。



### 5 温故：kube-proxy 包转发路径

从网络角度看，使用传统的 kube-proxy 处理 Kubernetes Service 时，包在内核中的 转发路径是怎样的？如下图所示：

![image-20210615153831451](D:\学习资料\笔记\k8s\k8s图\image-20210615153831451.png)

步骤：

1. 网卡收到一个包（通过 DMA 放到 ring-buffer）。
2. 包经过 XDP hook 点。
3. **内核给包分配内存**，此时才有了大家熟悉的 `skb`（包的内核结构体表示），然后 送到内核协议栈。
4. 包经过 GRO 处理，对分片包进行重组。
5. 包进入 tc（traffic control）的 ingress hook。接下来，**所有橙色的框都是 Netfilter 处理点。**
6. Netfilter：在 `PREROUTING` hook 点处理 `raw` table 里的 iptables 规则。
7. 包经过内核的**连接跟踪（conntrack）模块**。
8. Netfilter：在 `PREROUTING` hook 点处理 `mangle` table 的 iptables 规则。
9. Netfilter：在 `PREROUTING` hook 点处理 `nat` table 的 iptables 规则。
10. 进行**路由判断**（FIB：Forwarding Information Base，路由条目的内核表示，译者注） 。接下来又是四个 Netfilter 处理点。
11. Netfilter：在 `FORWARD` hook 点处理 `mangle` table 里的 iptables 规则。
12. Netfilter：在 `FORWARD` hook 点处理 `filter` table 里的 iptables 规则。
13. Netfilter：在 `POSTROUTING` hook 点处理 `mangle` table 里的 iptables 规则。
14. Netfilter：在 `POSTROUTING` hook 点处理 `nat` table 里的 iptables 规则。
15. 包到达 TC egress hook 点，会进行出方向（egress）的判断，例如判断这个包是到本 地设备，还是到主机外。
16. 对大包进行分片。根据 step 15 判断的结果，这个包接下来可能会：
17. 发送到一个本机 veth 设备，或者一个本机 service endpoint，
18. 或者，如果目的 IP 是主机外，就通过网卡发出去。

相关阅读，有助于理解以上过程：

1. Cracking Kubernetes Node Proxy (aka kube-proxy)
2. (译) 深入理解 iptables 和 netfilter 架构
3. 连接跟踪（conntrack）：原理、应用及 Linux 内核实现
4. (译) 深入理解 Cilium 的 eBPF 收发包路径（datapath）



### 6 知新：Cilium eBPF 包转发路径

作为对比，再来看下 Cilium eBPF 中的包转发路径：

> 建议和 (译) 深入理解 Cilium 的 eBPF 收发包路径（datapath） 对照看。

![image-20210615154733461](D:\学习资料\笔记\k8s\k8s图\image-20210615154733461.png)



对比可以看出，Cilium eBPF datapath 做了短路处理：从 tc ingress 直接 shortcut 到 tc egress，节省了 9 个中间步骤（总共 17 个）。更重要的是：这个 datapath 绕过了 整个 Netfilter 框架（橘黄色的框们），Netfilter 在大流量情况下性能是很差的。

去掉那些不用的框之后，Cilium eBPF datapath 长这样：

![image-20210615154817682](D:\学习资料\笔记\k8s\k8s图\image-20210615154817682.png)

Cilium/eBPF 还能走的更远。例如，如果包的目的端是另一台主机上的 service endpoint，那你可以直接在 XDP 框中完成包的重定向（收包 `1->2`，在步骤 `2` 中对包 进行修改，再通过 `2->1` 发送出去），将其发送出去，如下图所示：

![image-20210615154914667](D:\学习资料\笔记\k8s\k8s图\image-20210615154914667.png)

可以看到，这种情况下包都没有进入内核协议栈（准确地说，都没有创建 skb）就被转 发出去了，性能可想而知。

> XDP 是 eXpress DataPath 的缩写，支持在网卡驱动中运行 eBPF 代码，而无需将包送到复杂的协议栈进行处理，因此处理代价很小，速度极快。



### 7 eBPF 年鉴

eBPF 是如何诞生的呢？我最初开始讲起。这里“最初”我指的是 2013 年之前。

#### 2013

##### 前浪工具和子系统

回顾一下当时的 “SDN” 蓝图。

1. 当时有 OpenvSwitch（OVS）、`tc`（Traffic control），以及内核中的 Netfilter 子系统（包括 `iptables`、`ipvs`、`nftalbes` 工具），可以用这些工具对 datapath 进行“ 编程”：。
2. BPF 当时用于 `tcpdump`，在内核中尽量前面的位置抓包，它不会 crash 内核；此 外，它还用于 seccomp，对系统调用进行过滤（system call filtering），但当时使用的非常受限，远不是今天我们已经在用的样子。
3. 此外就是前面提到的 feature creeping 问题，以及 tc 和 netfilter 的代码重复问题，因为这两个子系统是竞争关系。
4. OVS 当时被认为是内核中最先进的数据平面，但它最大的问题是：与内核中其他网络模块的集成不好【译者注 1】。此外，很多核心的内核开发者也比较抵触 OVS，觉得它很怪。

> 【译者注 1】
>
> 例如，OVS 的 internal port、patch port 用 tcpdump 都是 抓不到包的，排障非常不方便。



##### eBPF 与前浪的区别

对比 eBPF 和这些已经存在很多年的工具：

1. tc、OVS、netfilter 可以对 datapath 进行“编程”：但前提是 datapath 知道你想做什 么（but only if the datapath knows what you want to do）。

2. - 只能利用这些工具或模块提供的既有功能。

3. eBPF 能够让你创建新的 datapath（eBPF lets you create the datapath instead）。

> - eBPF 就是内核本身的代码，想象空间无限，并且热加载到内核；换句话说，一旦加 载到内核，内核的行为就变了。
> - 在 eBPF 之前，改变内核行为这件事情，只能通过修改内核再重新编译，或者开发内核模块才能实现。



##### eBPF：第一个（巨型）patch

- 描述 eBPF 的 RFC 引起了广泛讨论，但普遍认为侵入性太强了（改动太大）。
- 另外，当时 nftables (inspired by BPF) 正在上升期，它是一个与 eBPF 有点类似的 BPF 解释器，大家不想同时维护两个解释器。

最终这个 patch 被拒绝了。

被拒的另外一个原因是前面提到的，没有遵循“大改动小提交”原则，全部代码放到了一个 patch。Linus 会疯的。



#### 2014

##### 第一个 eBPF patch 合并到内核

- 用一个扩展（extended）指令集逐步、全面替换原来老的 BPF 解释器。
- 自动新老 BPF 转换：in-kernel translation。
- 后续 patch 将 eBPF 暴露给 UAPI，并添加了 verifier 代码和 JIT 代码。
- 更多后续 patch，从核心代码中移除老的 BPF。

![image-20210615155438331](D:\学习资料\笔记\k8s\k8s图\image-20210615155438331.png)

我们也从那时开始，顺理成章地成为了 eBPF 的 maintainer。



##### Kubernetes 提交第一个 commit

巧合的是，对后来影响深远的 Kubernetes，也在这一年提交了第一个 commit：

![image-20210615155533757](D:\学习资料\笔记\k8s\k8s图\image-20210615155533757.png)



#### 2015

##### eBPF 分成两个方向：networking & tracing

到了 2015 年，eBPF 开发分成了两个方向：

- networking
- tracing



##### eBPF backend 合并到 LLVM 3.7

这一年的一个重要里程碑是 eBPF backend 合并到了 upstream LLVM 编译器套件，因此你现在才能用 clang 编译 eBPF 代码。



##### 支持将 eBPF attach 到 kprobes

这是 tracing 的第一个使用案例。

Alexei 主要负责 tracing 部分，他添加了一个 patch，支持加载 eBPF 用来做 tracing， 能获取系统的观测数据。



##### 通过 cls_bpf，tc 变得完全可编程

我主要负责 networking 部分，使 tc 子系统可编程，这样我们就能用 eBPF 来灵活的对 datapath 进行编程，获得一个高性能 datapath。



##### 为 tc 添加了一个 lockless ingress & egress hook 点

> 译注：可参考：
>
> - 深入理解 tc ebpf 的 direct-action (da) 模式（2020）
> - 为容器时代设计的高级 eBPF 内核特性（FOSDEM, 2021）



##### 添加了很多 verifer 和 eBPF 辅助代码（helper）

使用更方便。



##### bcc 项目发布

作为 tracing frontend for eBPF。



#### 2016

##### eBPF 添加了一个新 fast path：XDP

- XDP 合并到内核，支持在驱动的 ingress 层 attach BPF 程序。
- nfp 为第一家网卡及驱动，支持将 eBPF 程序 offload 到 cls_bpf & XDP hook 点。



##### Cilium 项目发布

Cilium 最开始的目标是 docker 网络解决方案。

- 通过 eBPF 实现高效的 label-based policy、NAT64、tunnel mesh、容器连通性。
- 整个 datapath & forwarding 逻辑全用 eBPF 实现，不再需要 Docker 或 OVS 桥接设备。



#### 2017

##### eBPF 开始大规模应用于生产环境

2016 ~ 2017 年，eBPF 开始应用于生产环境：

1. Netflix on eBPF for tracing: ‘Linux BPF superpowers’

2. Facebook 公布了生产环境 XDP+eBPF 使用案例（DDoS & LB）

3. - 用 XDP/eBPF 重写了原来基于 IPVS 的 L4LB，性能 `10x`。
   - eBPF 经受住了严苛的考验：从 2017 开始，每个进入 facebook.com 的包，都是经过了 XDP & eBPF 处理的。

4. Cloudflare 将 XDP+BPF 集成到了它们的 DDoS mitigation 产品。

5. - 成功将其组件从基于 Netfilter 迁移到基于 eBPF。
   - 到 2018 年，它们的 XDP L4LB 完全接管生产环境。
   - 扩展阅读：(译) Cloudflare 边缘网络架构：无处不在的 BPF（2019）

> 译者注：基于 XDP/eBPF 的 L4LB 原理都是类似的，简单来说，
>
> 1. 通过 BGP 宣告 VIP
> 2. 通过 ECMP 做物理链路高可用
> 3. 通过 XDP/eBPF 代码做重定向，将请求转发到后端（VIP -> Backend）
>
> 对此感兴趣可参考入门级介绍：L4LB for Kubernetes: Theory and Practice with Cilium+BGP+ECMP



#### 2017 ~ 2018

##### eBPF 成为内核独立子系统

随着 eBPF 社区的发展，feature 和 patch 越来越多，为了管理这些 patch，Alexei、我和 networking 的一位 maintainer David Miller 经过讨论，决定将 eBPF 作为独立的内核子系统。

- eBPF patch 合并到 `bpf` & `bpf-next` kernel trees on git.kernel.org
- 拆分 eBPF 邮件列表：`bpf@vger.kernel.org` (archive at: `lore.kernel.org/bpf/`)
- eBPF PR 经内核网络部分的 maintainer David S. Miller 提交给 Linus Torvalds

##### kTLS & eBPF

> kTLS & eBPF for introspection and ability for in-kernel TLS policy enforcement

kTLS 是将 TLS 处理 offload 到内核，例如，将加解密过程从 openssl 下放到内核进 行，以使得内核具备更强的可观测性（gain visibility）。

有了 kTLS，就可以用 eBPF 查看数据和状态，在内核应用安全策略。 目前 openssl 已经完全原生支持这个功能。

##### bpftool & libbpf

为了检查内核内 eBPF 的状态（introspection）、查看内核加载了哪些 BPF 程序等， 我们添加了一个新工具 bpftool。现在这个工具已经功能非常强大了。

同样，为了方便用户空间应用使用 eBPF，我们提供了用户空间 API（user space API for applications） `libbpf`。这是一个 C 库，接管了所有加载工作，这样用户就不需要 自己处理复杂的加载过程了。

##### BPF to BPF function calls

增加了一个 BPF 函数调用另一个 BPF 函数的支持，使得 BPF 程序的编写更加灵活。



#### 2018

##### Cilium 1.0 发布

这标志着 BPF 革命之火燃烧到了 Kubernetes networking & security 领域。

Cilium 此时支持的功能：

- K8s CNI
- Identity-based L3-L7 policy
- ClusterIP Services

##### BTF（Byte Type Format）

内核添加了一个称为 BTF 的组件。这是一种元数据格式，和 DWARF 这样的 debugging data 类似。但 BTF 的 size 要小的多，而更重要的是，有史以来，内核第一次变得可自 描述了（self-descriptive）。什么意思？

想象一下当前正在运行中的内核，它内置了自己的数据格式（its own data format） 和内部数据结构（internal structures），你能用工具来查看这些东西（you can introspect them）。还是不太懂？这么说吧，BTF 是后来的 “一次编译、到处运行”、 热补丁（live-patching）、BPF global data 处理等等所有这些 BPF 特性的基础。

新的特性不断加入，它们都依赖 BTF 提供富元数据（rich metadata）这个基础。

> 更多 BTF 内容，可参考 (译) Cilium：BPF 和 XDP 参考指南（2019）
>
> 译者注

##### Linux Plumbers 会议开辟 BPF/XDP 主题

这一年，Linux Plumbers 会议第一次开辟了专门讨论 BPF/XDP 的微型分会，我们 一起组织这场会议。其中，Networking Track 一半以上的议题都涉及 BPF 和 XDP 主题，因为这是一个非常振奋人心的特性，越来越多的人用它来解决实际问题。

##### 新 socket 类型：AF_XDP

内核添加了一个新 socket 类型 `AF_XDP`。它提供的能力是：在零拷贝（ zero-copy）的前提下将包从网卡驱动送到用户空间。

> 回忆前面的内容，数据包到达网卡后，先经过 XDP，然后才为这个包分配内存。因此在 XDP 层直接将包送到用户态是无需拷贝的。
>
> 译者注

`AF_XDP` 提供的能力与 DPDK 有点类似，不过

- DPDK 需要重写网卡驱动，需要额外维护用户空间的驱动代码。
- `AF_XDP` 在复用内核网卡驱动的情况下，能达到与 DPDK 一样的性能。

而且由于复用了内核基础设施，所有的网络管理工具还都是可以用的，因此非常方便， 而 DPDK 这种 bypass 内核的方案导致绝大大部分现有工具都用不了了。

由于所有这些操作都是发生在 XDP 层的，因此它称为 `AF_XDP`。插入到这里的 BPF 代码 能直接将包送到 socket。

##### bpffilter

开始了 bpffilter prototype，作用是通过用户空间驱动（userspace driver），将 iptables 规则转换成 eBPF 代码。

这是将 iptables 转换成 eBPF 的第一次尝试，整个过程对用户都是无感知的，其中的某些组件现在还在用，用于在其他方面扩展内核的功能。



#### 2018 ~ 2019

### bpftrace

Brendan 发布了 bpftrace 工具，作为 DTrace 2.0 for Linux。

##### BPF 专著《BPF Performance Tools》

Berendan 写了一本 800 多页的 BPF 书。

##### Cilium 1.6 发布

第一次支持完全干掉基于 iptables 的 kube-proxy，全部功能基于 eBPF。

> 这个版本其实是有问题的，例如 1.6 发布之后我们发现 externalIPs 的实现是有问题 ，社区在后面的版本修复了这个问题。在修复之前，还是得用 kube-proxy：https://github.com/cilium/cilium/issues/9285
>
> 译者注

##### BPF live-patching

添加了一些内核新特性，例如尾调用（tail call），这使得 eBPF 核心基础 设施第一次实现了热加载。这个功能帮我们极大地优化了 datapath。

另一个重要功能是 BPF trampolines，这里就不展开了，感兴趣的可以搜索相关资料，我只 能说这是另一个振奋人心的技术。

##### 第一次 bpfconf：受邀请才能参加的 BPF 内核专家会议

如题，这是 BPF 内核专家交换想法和讨论问题的会议。与 Linux Plumbers 会议互补。

##### BPF backend 合并到 GCC

前面提到，BPF backend 很早就合并到 LLVM/Clang，现在，它终于合并到 GCC 了。至此，GCC 和 LLVM 这两个最主要的编译器套件都支持了 BPF backend。

此外，BPF 开始支持有限循环（bounded loops），在此之前，是不支持循环的，以防止程序无限执行。



#### 2019 ~ 2020

##### 不知疲倦的增长和 eBPF 的第三个方向：Linux security modules

- Google 贡献了 BPF LSM（安全），部署在了他们的数据中心服务器上。
- BPF verifier 防护 Spectre 漏洞（2018 年轰动世界的 CPU bug）：even verifying safety on speculative program paths。
- 主流云厂商开始通过 SRIOV 支持 XDP：AWS (ena driver), Azure (hv_netvsc driver), …
- Cilium 1.8 支持基于 XDP 的 Service 负载均衡和 host network policies。
- Facebook 开发了基于 BPF 的 TCP 拥塞控制模块。
- Microsoft 基于 BPF 重写了将他们的 Windows monitoring 工具。



### 8 eBPF：过去 50 年操作系统最大的变革

![image-20210615161029711](D:\学习资料\笔记\k8s\k8s图\image-20210615161029711.png)

Brendan 说在他的职业生涯中，eBPF 是他见过的操作系统中最大的变革之一（one of the biggest operating system changes），他为能身处其中而感到非常兴奋。

我接下来只能用数字证明：Brendan 的兴奋是没错的。



### 9 eBPF 数字榜单（截至 2020.07）

eBPF 内核社区截至 7 月份的一些数据：

- 347 个贡献者，贡献了 4,935 个 patch 到 BPF 子系统。

- BPF 内核邮件列表日均 50 封邮件 (高峰经常超过日均 100)。

- - 23,395 mails since mailing list git archive was added in Feb 2019

- 每天合并 4 个新 patch。patch 接受率 30% 左右。

- 30 个程序（different program），27 种 BPF map 类型，141 个 BPF helpers，超过 3,500 个测试。

- 2 个 BPF kernel maintainers & team of 6 senior core reviewers。

- - 主要贡献来自：Isovalent（Cilium）, Facebook and Google

毫无疑问，这是内核里发展最快的子系统！



### 10 业界趋势

![image-20210615161157603](D:\学习资料\笔记\k8s\k8s图\image-20210615161157603.png)



列举几个生产环境大规模使用 BPF 的大厂：

- Facebook：L4LB、DDoS、tracing。
- Netflix：BPF 重度用户，例如生产环境 tracing。
- Google：Android、服务器安全以及其他很多方面。
- Cloudflare：L4LB、DDoS。
- Cilium

上图中，右下角是前 Netfilter 维护者 Rusty Russel 说的一句，业界对 eBPF 的受认可程度可窥一斑。



### 11 eBPF 革命：燃烧到 Kubernetes 社区

![image-20210615161257191](D:\学习资料\笔记\k8s\k8s图\image-20210615161257191.png)

eBPF 已无处不在，而你还在使用 iptables？



#### 11.1 干掉 kube-proxy/iptables

不用再忍受 iptables 复杂冗长的规则和差劲的性能了，以前没得选，现在你可以做个好人：

```sh
$ kubectl -n kube-system delete ds kube-proxy
```

作为例子，我们来看看 Cilium 是怎么做 Service 的负载均衡的。

Service 细节实现可参考 Cracking Kubernetes Node Proxy (aka kube-proxy)。



#### 11.2 Cilium 的 Service load balancing 设计

![image-20210615161424306](D:\学习资料\笔记\k8s\k8s图\image-20210615161424306.png)

如上图所示，主要涉及两部分：

1. 在 socket 层运行的 BPF 程序
2. 在 XDP 和 tc 层运行的 BPF 程序



##### 东西向流量

我们先来看 socker 层。

![image-20210615161518350](D:\学习资料\笔记\k8s\k8s图\image-20210615161518350.png)

如上图所示，

**Socket 层的 BPF 程序主要处理 Cilium 节点的东西向流量（E-W）。**

- 将 Service 的 `IP:Port` 映射到具体的 backend pods，并做负载均衡。
- 当应用发起 **connect、sendmsg、recvmsg 等请求（系统调用）时，拦截这些请求，** 并根据请求的 `IP:Port` 映射到后端 pod，直接发送过去。反向进行相反的变换。

这里实现的好处：性能更高。

- **不需要包级别（packet leve）的地址转换（NAT）。在系统调用时，还没有创建包**，因此性能更高。
- 省去了 kube-proxy 路径中的很多中间节点（intermediate node hops）

可以看出，应用对这种拦截和重定向是无感知的（符合 k8s Service 的设计）。



##### 南北向流量

再来看从 k8s 集群外进入节点，或者从节点出 k8s 集群的流量（external traffic）， 即南北向流量（N-S）：

> 区分集群外流量的一个原因是：Pod IP 很多情况下都是不可路由的（与跨主机选用的网 络方案有关），只在集群内有效，即，集群外访问 Pod IP 是不通的。
>
> 因此，如果 Pod 流量直接从 node 出宿主机，必须确保它能正常回来。而 node IP 一般都是全局可达的，集群外也可以访问，所以常见的解决方案就是：在 Pod 通过 node 出集群时，对其进行 SNAT，将源 IP 地址换成 node IP 地址；应答包回来时，再进行相 反的 DNAT，这样包就能回到 Pod 了。

![image-20210615161720832](D:\学习资料\笔记\k8s\k8s图\image-20210615161720832.png)

如上图所示，集群外来的流量到达 node 时，由 **XDP 和 tc 层的 BPF 程序进行处理**， 它们做的事情与 socket 层的差不多，将 Service 的 `IP:Port` 映射到后端的 `PodIP:Port`，如果 backend pod 不在本 node，就通过网络再发出去。发出去的流程我们 在前面 `Cilium eBPF 包转发路径` 讲过了。

这里 BPF 做的事情：执行 DNAT。这个功能可以在 XDP 层做，也可以在 TC 层做，但 在 XDP 层代价更小，性能也更高。

总结起来，这里的核心理念就是：

1. 将**东西向流量**放在**离 socket 层尽量近**的地方做。
2. 将**南北向流量**放在**离驱动（XDP 和 tc）层尽量近**的地方做。



#### 11.3 XDP/eBPF vs kube-proxy 性能对比

##### 网络吞吐

测试环境：两台物理节点，一个发包，一个收包，收到的包做 Service loadbalancing 转发给后端 Pods。

![image-20210615161958878](D:\学习资料\笔记\k8s\k8s图\image-20210615161958878.png)

可以看出：

1. Cilium XDP eBPF 模式能处理接收到的全部 10Mpps（packets per second）。
2. Cilium tc eBPF 模式能处理 3.5Mpps。
3. kube-proxy iptables 只能处理 2.3Mpps，因为它的 hook 点在收发包路径上更后面的位置。
4. kube-proxy ipvs 模式这里表现更差，它**相比 iptables 的优势要在 backend 数量很多的时候才能体现出来。**



##### CPU 利用率

我们生成了 1Mpps、2Mpps 和 4Mpps 流量，空闲 CPU 占比（可以被应用使用的 CPU）结果如下：

![image-20210615162144583](D:\学习资料\笔记\k8s\k8s图\image-20210615162144583.png)

结论与上面吞吐类似。

- XDP 性能最好，是因为 **XDP BPF 在驱动层执行，不需要将包 push 到内核协议栈**。
- kube-proxy 不管是 iptables 还是 ipvs 模式，都在**处理软中断（softirq）上消耗了大量 CPU**。



### 12 eBPF 和 Kubernetes：未来展望

> “The Linux kernel continues its march towards becoming BPF runtime-powered microkernel.”

“Linux 内核继续朝着成为 BPF runtime-powered microkernel 而前进”。这是一个非 常有趣的思考角度。

- 设想在将来，Linux 只会保留一个非常小的核心内核（tiny core kernel），其他所有 内核功能都由用户定义，并用 BPF 实现（而不再是开发内核模块的方式）。
- 这样可以减少受攻击面，因为此时的核心内核非常小；另外，所有 BPF 代码都会经过 verifer 校验。
- 极大减少 ‘static’ feature creep，资源（例如 CPU）可以用在更有意义的地方。
- 设想一下，未来 Kubernetes 可能会内置 custom BPF-tailored extensions，能根据用户的应用自动适配（optimize needs for user workloads）；例如，判断 pod 是跑在数据中心，还是在嵌入式系统上。

![image-20210615162401812](D:\学习资料\笔记\k8s\k8s图\image-20210615162401812.png)

我们的目标是星辰大海，与此相比，kube-proxy replacement 只是最微不足道的开端。



### 13 结束语

![image-20210615162431338](D:\学习资料\笔记\k8s\k8s图\image-20210615162431338.png)

- Try it out：https://cilium.link/kubeproxy-free
- Contribute：https://github.com/cilium/cilium



## 网易轻舟对 CIlium 容器网络的探索和实践

2019 年网易轻舟使用 sockmap+sk redirect 来优化轻舟 Service Mesh 的延迟，2020 年开始，我们逐步对 eBPF Service、Cilium NetworkPolicy，Cilium 容器网络进行实践，到 2021 年中旬，网易轻舟对外部客户提供了 Cilium 整体解决方案。

本文会深入介绍 Cilium，并澄清一些认知误区，然后给出网易轻舟是如何使用 Cilium 的。目前国内这方面深入解析材料较少，如果您也正在探究，希望这篇文章能给您带来帮助。



### eBPF 容器网络探索背景

目前容器网络方案仍然呈现着百花齐放的态势，这主要是由于不同的 CNI 各有优势。开源的原生 CNI 具备较广的使用场景，即插即用。其中支持 overlay 方式的几个方案（openshift-sdn、flannel-vxlan、calico-ipip），能更好地适配 L2/L3 的网络背景，而 underlay 方式的几个方案（calico-bgp、kube-router、flannel-hostgw），则在性能上更趋近于物理网络，在小包场景中性能明显好过 overlay 类型方案。此外，许多主流云厂商基于还会自建 VPC 能力，实现 VPC-based CNI ，这类 CNI 与集群交付场景深度耦合，但也赋予了容器网络 VPC 的属性，具备更强的安全特性。

此外，业界还有大量针对 CNI 或四层负载均衡的优化措施，例如：

- 优化封装协议，基于 vxlan 有更好的兼容性，基于 ipip 协议则能提升一定的带宽能力（部分云厂商如 AWS 并不支持 ipip 协议的包）
- 使用 IPVS 替换 kube-proxy 解决大规模环境里，Service 带宽降低和数据路径下发时间变长
- 使用 eBPF 技术来加速 IPVS，进一步降低延迟，提升 Service 数据路径性能
- 利用 multus 组合 CNI 多网络平面，并利用 SR-IOV 技术做定向硬件加速
- 虚拟网卡时使用 ipvlan 替代 vethpair，提升性能

总的来说，通过方案整合和调优的方式进一步满足了业务发展的需求，但是当前流行的 CNI 支持场景要么是不够通用，要么在高 IO 业务下存在性能问题，要么不具备安全能力，要么因 Kubernetes service 性能问题，集群规模上不去。因此我们想使用 eBPF 技术实现一套无硬件依赖的高性能、高兼容性的容器网络解决方案，这个方案解决的问题如下：

- 解决 kube-proxy 性能问题
- 优化数据路径降低业务端到端的延时解决高 IO 业务的性能瓶颈
- 降低 Service Mesh 路径延迟
- 支持链路加密和高性能安全策略
- 支持无成本集成到主流的容器网络方案

基于 eBPF 技术做的比较好的是 Cilium，所以 2019 年末开始，我们就开始了相关的探索。



### Cilium 功能解析

Cilium 功能解析，从功能列表，亮点功能，功能限制，三个角度进行阐述。

#### （1）功能列表

![图片](D:\学习资料\笔记\k8s\k8s图\145)

#### （2）亮点功能

Cilium 支持的功能更像一个容器网络功能的复合体，其具有 Underlay 路由模式、Overlay 模式，也具有链路加密能力，此外其独特能力如下：

1. 带宽管理
2. 集群联邦
3. IPVLAN 模式
4. 七层安全策略
5. DSR 模式保留源地址，节省带宽，避免 SNAT 端口耗尽问题
6. Kubernetes Service 性能调优，支持 Maglev 算法
7. XDP 性能调优，高性能南北向 Service
8. Local Redirect Policy
9. 流量可见性，细粒度流量控制，用于审计
10. 高性能更能适配高带宽的网卡（eg: 50G、100G）

从功能范畴看，其亮点功能有安全、性能、功能三个方面，其有一部分亮点能力是 Cilium 社区对容器网络最佳实践的功能支持，有一部分能力是来源于 eBPF 能力的创新，所以 Cilium 是一个复合品；社区思路可以一句话概括为 “在容器网络领域，大家有的功能我也有，而且我性能更好，此外我借助 eBPF 能力从内核侧创新了大部分功能”。

#### （3）功能限制

Cilium 带来的技术红利让不少国内外公司去实践，国内公司来说，除网易轻舟外，我们看到腾讯、阿里、爱奇艺、携程等同行在生产环境中落地 Cilium。虽然 Cilium 通过 eBPF 给容器网络提供了创新机会，但是也带来了两点问题，这两点问题让绝大部分用户持观望态度。

 **问题 1: eBPF 相比 iptables 调试难度更大，技术要求较高**

iptables 存在内核已经存在几十年了，相关技术栈的开发者多，已经被广泛的接受，而 2014 年 eBPF 概念才被提出，2018 年依托于 eBPF 的容器网络 Cilium 诞生

eBPF 技术研发难度会大些（报错调试日志不明显、整体技术把控的人才少）

从灵活性角度来看，Iptables 配合策略路由只需要几条命令就可以编排数据路径，更简单些，eBPF 需要编写代码支持新的数据链路需求复杂度会高些

面临上述问题，但是我们想法是：

Iptables 技术栈的确是更加通用的，但也搞不定某些业务场景，比如：

- 安全性较高的 Kubernetes 集群，需要强流量日志审计的功能的业务
- 存在业务高 IO，低延时业务，或者 Service 规模大，且业务短链接较多，集群规模较大，需要节省南北向网络带宽等业务

我们认为 eBPF 技术栈处于发展期、周边工具在逐步完善中，高复杂度会带来额外的人力成本，如果业务需求强于克服 eBPF 的技术的人力成本，外加上有技术厂商支撑，某些对性能和安全要求较高的场景选型 eBPF 技术问题也不大。

 **问题 2: 内核版本要求高**

根据 CIlium 社区，基于 eBPF Kubernetes 容器网络需要内核允许最小版本 4.9，Calico 支持的 eBPF 数据面要求内核至少 5.3 版本。

面临上述问题，我们经历是，前两年内核版本的限制属于很大的阻力，当前于网易轻舟支撑容器云平台来说，5.4 内核逐步在铺开使用了，因为我们需要更强的 eBPF 能力进行监控和更完整干净的 CGROUPv2 特性，我们希望对业务更细粒度的分资源优先级，更强的资源隔离能力（IO 隔离），防止业务之间互相影响，增强容器平台的可用性。而 5.4 内核作为基础内核跑 Cilium eBPF 功能够用。



### Cilium 数据面

看完 CIlium 功能后，我们看下这些功能的数据面是如何实现的，因此后文，会先深入解析 Cilium 数据面的实现，并揭露其高性能的原因，最后再澄清一些常常被误解的说法。

#### （1）数据面路径

首先我们知道：

- Cilium 数据面和 Kernel 版本有一定关系，不同内核支持的数据面路径不同，上层的数据面实现也有一定差异。
- Cilium 数据面和功能开关有一定关系，不同的功能开关的开启和关闭，会影响数据路径的转发行为。

因此，选定 Kernel5.4，Kernel5.10 来进行探讨，同时打开 kube-proxy 卸载，目前只开启 IPv4 协议，其它数据面采用 CIlium 的默认值，选择 Cilium1.10 版本，下文主要以图示和说明的方式进行表述， 因篇幅限制，选择路由模式、VETH 接口类型进行探讨。

![图片](D:\学习资料\笔记\k8s\k8s图\146)

**1. 跨节点 Pod -> Pod**

Pod 发出流量后，经过宿主机的 veth lxc 接口的 tc ingress hook 的 eBPF 程序处理后，送给 kernel stack 进行路由查找，然后经过物理口 eth0 的 tc egress 发出到达 Node2 的 eth0

Node2 接收流量后，先经过 Node2 物理口 eth0 的 tc ingress hook 的 eBPF 处理，然后送给 kernel stack 进行路由查找，发给 cilium_host 接口，流量经过 cilium_host 的 tc egress hook 点后，最后 redirect 到目的 Pod 的宿主机 lxc 接口上，最终送给目的 pod。

**2. 同节点 pod->pod**

pod 发出流量后，流量会发给本 Pod 宿主机 lxc 接口的 tc ingress hook 的 eBPF 程序处理，eBPF 最终会查找目的 Pod，确定位于同一个节点，直接通过 redirect 方法将流量直接越过 kernel stack，送给目的 Pod 的 lxc 口，最终将流量送给目的 Pod。

**3. 访问 NodePort Local Endpoint**

Client->NodePort，流量从 Node2 的 eth0 口进入，经过 tc ingress hook eBPF Dnat 处理后，继续将流量发给 kernel，kernel 查路由转发给 cilium_host 接口，cilium_host 接口的 tc ingress 收到流量后直接 redirect 流量到目的 pod 的宿主机 lxc 接口上，最终送给目的 pod。

回程流量从 Pod 出来，经过 veth 宿主机侧接口口 lxc 的 tc ingress hook eBPF 反 Dnat 处理，该 eBPF 程序直接将流量从定向到物理口，该过程不经过 kernel，最终在经过物理口的 tc egress hook 点后发给 Client。

**4. 访问 NodePort Remote EndPoint**

Client 访问 NodePort，流量发给 Node1 接口 eth0 ，经过 tc ingress hook eBPF 先进行 Dnat 将目的地址变成目的 Pod 的 IP 和 Port，后进行 Snat 将源地址变成 Node1 物理口的 IP 和 Port，最后将流量 redirect 到 Node1 的 eth0 物理口上，流量会经过 tc egress hook 点后发出。

**5.Pod 访问外网**

Pod 发出流量后，经过宿主机的 veth 的 lxc 口的 tc ingress hook 的 eBPF 程序处理后，送给 kernel stack 进行路由查找，确定需要从 eth0 发出，因此流量回发给物理口 eth0，而经过物理口 eth0 的 tc egress eBPF 程序做个 Snat，将源地址转换成 Node 节点的物理接口的 IP 和 Port 发出。

从外网回来的反向流量，经过 Node 节点物理口 eth0 tc ingress eBPF 程序进行 Snat 还原，然后将流量发给 kernel 查路由，流量流到 cilium_host 接口后，经过 tc egress eBPF 程序，直接识别出是发给 Pod 的流量，将流量直接 redirect 到目的 Pod 的宿主机 lxc 接口上，最终反向流量回到 Pod。

**6. 主机访问 Pod**

主机访问 Pod 流量使用 cilium_host 口发出，所以在 tc egress hook 点 eBPF 程序直接 redirect 到目的 Pod 的宿主机 lxc 接口上，最终发给 Pod。

反向流量，从 Pod 发出到宿主机的接口 lxc，经过 tc ingress hook eBPF 识别送给 kernel stack，回到宿主机上。

针对于 XDP 情况没有标注到上面流程中，其实 XDP 过程和 tc ingress 处理方式一致，XDP Hook 要比 tc ingress 更靠近收包流程，这个时候 skb 还没有形成，因此性能更高，使用 xdp 特别适合 NodePort、HostPort、External IP 等这种南北向 Service 流量加速，从社区测试结果看，大概 3-4 倍 PPS 提升。

上面介绍是基于 5.4 内核的，在 5.10 内核以后，Cilium 新增了 eBPF Host-Routing 功能，该功能更加速了 eBPF 数据面性能，新增了 bpf_redirect_peer 和 bpf_redirect_neigh 两个 redirect 方式，bpf_redirect_peer 可以理解成 bpf_redirect 的升级版，其将数据包直接送到 veth pair Pod 里面接口 eth0 上，而不经过宿主机的 lxc 接口，这样实现的好处是少进入一次 cpu backlog queue 队列，该特性引入后，路由模式下，Pod -> Pod 性能接近 Node -> Node 性能，同时 Cilium 数据面路径发生了较大的变化，如下图，具体变化直接对比看图即可，篇幅有限，重点说明两点变化：

![图片](D:\学习资料\笔记\k8s\k8s图\147)

1.除了 host->Pod 外，所有路径都是经过接口跳着走的，最大变化是不再经过 Kernel 转发处理，也意味着来回流量路径不再经过 kernel 的 Netfilter 框架，kernel tc 等模块，大大提升了转发性能；

2.redirect_peer 特性专用于 veth pair 类型接口，因为流量被直接重定向到 Pod 里面的接口，所以在宿主机 lxc 口上无法抓到进入 Pod 的流量，因 tcpdump 抓包点在 tc ingress hook 之前，所以可以抓到出 Pod 未经过 eBPF 处理的流量。

#### （2）高性能的原因

高性能可分为如下两个方面

**数据路径延迟低，带宽高**

数据路径低延迟，高带宽原因是，其采用技术创新短路传统了 Kernel 框架，减少了数据包转发所需的指令。这些技术包括，eBPF 夹持下的 redirect、redirect_peer、redirect_neigh、XDP、sockmap、sk redirect 等技术。

**Kubernetes service 数据面性能高，Service 规模扩大，控制面下发时间变化不大。**

Kubernetes service 数据面性能高，分为集群内访问的 socket LB 技术，该技术使得不需要做数据包 skb 更改，直接将容器平台内部访问 service 流量转变成直接访问 Pod 效果，访问 Pod 路径也是经过加速的。南北向 Service（NodePort、external IP、HostPort）性能高的原因是其支持 Native XDP 加速，外加上路径跳转优化，对于 Remote Pod 的 Service 场景，还支持了 DR 模式，从逻辑上减少一跳路径。

Kubernetes service 控制面下发性能，这块主要采用了 eBPF map 技术，该技术实质会采用 Hash 方法，摆脱了变动 Service 就要全量下发 iptables 的问题，因此控制面下发时间随着 Service 规模的变大，策略更新延时变化不大。

#### （3）概念澄清

 IPtables 实现 Service 的性能高低是要看场景的

基于 Iptables 实现的 Service，KubeCon Talk “Scale Kubernetes to Support 50,000 Services” 有一个特定的测量结果：

> 这是 Kubernetes 服务转发的瓶颈，部署 5,000 个 Services（40k rules）时吞吐量降低了约 30％，而部署 10,000 个服务则降低了 80％（性能相差 6 倍）。同样 5,000 个 Services（40k rules）的规则更新花费了 11 分钟。

但是 Iptables 是线性规则匹配和规则跳查的，但是这个慢速过程只发生在新建阶段，一旦会话建立好，后续流量走 CT 即可，所以 Service 到达一定规模，对于短链接业务延迟会有较大的影响，从容忍度角度看，这个性能降低的影响，对于某些业务是可容忍的。

此处不讨论 5000 个 Service 是否在生产环境真的需要 11 分钟（这个和环境，endpoint 规模有关），只讨论业务容忍度问题，业务对于 Service 规则更新分钟级别生效的容忍度会差些，因为 Service 规则更新慢会导致经过 Service 流量仍然发给异常的 Pod，导致业务失败。

Kubernetes Service 所带来的性能问题，也不是说一定会成为集群的性能瓶颈，因为如果选择不使用 Kubernetes Service 数据路径，或者甚至不使用 CNI，也就没有这个问题。

 Cilium XDP 是有限制的

首先通用的 XDP 性能提升有限，Native XDP 性能较符合预期，但 XDP 使用是有限制的，当前其还无法支持 Bond 口，这块网易轻舟的实践方法是物理机器不做 Bond，直接将两个接口通过 ecmp 等价路由方向做 HA，使用 XDP 对物理口加速南北向的 Service，并选定若干台 Node 节点，直接通过 BGP 将 Service IP 直接发布出去，直接省去了南北向集中式的四层 LB。



### Cilium 控制面

Cilium 控制面，从控制面框架，数据面下发过程来展开，控制面框架能对 CIlium 控制面有一个全局意识，数据面构建过程可以明确数据面如何搭建的。

![图片](D:\学习资料\笔记\k8s\k8s图\148)

首先明确事情是，CIlium agent 负责所有数据路径 eBPF 加载，网络连通的配置管理，负责将数据面搭建起来，是控制面的核心组件。而 Cilium agent 工作可按照加载时机分为两部分，一部分在于启动时候加载，一部分在于创建 Pod 后或者说读取到 Pod 所对应 endpoint 对象后加载。

启动时候加载的对象一般是数据面骨架，这部分和数据面框架有关，如下

- cilium_host /cilium_net 接口建立，以及 eBPF 程序加载
- 全局 socket LB、vxlan 口、物理口的 eBPF 规则的编译下发，少量的 IPtables 规则的下发
- MTU 设置，物理口识别，开启接口的 Forward 转发、Kernel eBPF 能力探测，并更具探测能力以及预配置选项进行 eBPF 功能开启等

创建 Pod 加载，用于给创建 Pod 编译加载 eBPF 程序，一般用于加载位于宿主机侧 veth 口的 tc ingress

为了控制 eBPF 程序的编译和加载，Cilium 使用自己的 Cilium endpoint 进行管理，Pod 建立过程中，Kubelet 会调用 Cilium cni 创建 Pod veth 口，其中 Cilium cni 会通过 unix socket 调用 cilium agent 来创建 endpoint 对象，这些对象含有丰富相关信息（eg mac 地址、接口名称、加载策略名称、policy 规则等），这些对象会通过 API Server 被下发到 etcd 中记录起来，Cilium agent 根据这些对象编译加载 eBPF 程序。

为了适配支持不同内核，Cilium agent 会自己探测是否开启某功能，如果默认内核不支持则起自动回退到能支持的内核特性，为了更大的功能灵活性，Cilium eBPF 有设计了许多数据面的配置开关，非常零碎，这些配置选项通过 cilium-config map 管理，Cilium agent 会将此 map 挂载成路径，同时根据配置的开关，动态形成 eBPF 程序的编译宏，通过该编译宏来编译 eBPF 程序，进而生效和关闭某些数据面功能。



### 网易轻舟 Cilium 的实践

阅读上文后心中会有一个整体框架，再看网易轻舟对 CIlium 实践思路，以及实践过程中遇到的一些问题和解决办法，就比较清晰了。

#### （1）网易轻舟基于 CIlium 的实践思路

![图片](D:\学习资料\笔记\k8s\k8s图\149)

简单概括我们的实践思路就是 “3+1”，其中“3”指 3 个插件，“1” 指 1 个 Cilium eBPF 容器网络 CNI，我们的思考是：

- Sockops 主要是加速轻舟服务网格产品、轻舟 Serverless 产品的 Sidecar 场景，降低服务网格延时 9%，QPS 提升 20%，提升轻舟 Serverless 产品 Knative QueueProxy 和业务容器之间的访问 QPS 20%
- NetworkPolicy 插件，支持插入主流的 CNI 和 Netease 容器网络，该插件主要是使用 eBPF 实现安全策略，性能相比较与 IPtables 更好，同时配合轻舟 Hubble 组件来做集群的流量审计，主要用于金融业务。
- eBPF Service 插件，该插件用于替换 kube-proxy，适用于有 kube-proxy 性能需求的业务场景，当前该组件我们优先支持了 Cluster IP，南北向的 Service 和容器网络类型关联度较大，我们只支持了 5.10 内核的路由场景。

一开始我们想直接推动老的集群使用使用 Cilium 容器网络，但是发现这块难度较大，因为并不是所有的业务都需要高性能的容器网络，从 0-1 的替换也不合适，但是我们的业务方对不同的功能特性是有需求的，所以我们将相关功能抠出来插入到当前容器网络场景，满足业务需求。

对于新的 Kubernetes 集群，我们推动使用 CIlium 容器网络，2021 年上半年，我们将 Cilium CNI 落地了到了大规模使用 Kafka 和 ES 的高 IO 场景的新业务集群，我们使用 kernel5.4，并且将 eBPF Host-Routing 所需要的 bpf_redirect_peer 和 bpf_redirect_neigh eBPF helper 移植到 Kernel5.4。



#### （2）实践过程中遇到的问题和解决办法举例

##### Cilium Chaining 的适配问题

CIlium Chaining 方式做 kube-proxy 卸载，低于 5.10 的 Kernel 内核情况下，对于南北向类型的 Service 来说，NodePort 流量不通，这个问题是由于 Node ->Pod 经过 kernel 转发，Pod -> Node 直接走 eBPF redirect 到物理口，导致了来回路径不一致，导致三次握手的 ACK 数据包被内核的 iptables 状态检查（ --ctstate INVALID -j DROP）丢弃，而 5.10 内核没有该问题， 因为 5.10 内核是从接口之间来回跳转的，不会经过 kernel 转发路径，我们的做法是通过高版本内核来解决此问题。

#####  数据链路还不够灵活

在私有云场景，其数据链路的需求是多种多样的，业务 Pod 某些流量有定制的数据路径要求，但是 eBPF 程序往往需要做额外的开发才能支持，这对业务来说都是时间成本，因此我们抽象了此种场景，支持将目标流量直接打上 mark 送给 Kernel，Kernel 协议栈通过 IPtables 和策略路由配合起来快速实现业务定制路径，通过此种方法满足业务快速迭代的需求，而后我们再根据通用性程度选择是否完全通过 eBPF 技术实现。

#####  网络问题定位困难

CIlium 场景下，特别是使用 eBPF 后，其数据包流量是来回跳转的，针对于非常特殊场景，有时候走内核，有时候不走内核，非常容易导致网络不通，所以我们通过 eBPF 做了一个 tcp packet 抓取工具，该工具通过 kprobe 到内核关键收发包函数和 iptables ，来进行流量抓取，同时丰富了 eBPF 数据包 trace 点，配合 Hubble 来快速定位网络问题。

#####  生产环境内核版本低

**Sockops 的支持**

我们在 2019 年开始使用 socket 短路方式降低服务网格延迟，但是当时内核 4.19.87 还没有完全支持 sockmap+sk redirect，所以我们将内核 patch 回合了一些，达到我们的使用需求，这里额外说明的是，针对于 Sockops 功能，特别是用于 Service Mesh 场景，该场景下我们采用 127.0.0.1 进行业务容器和 Sidecar 的互通，Cilium 的实现的 local redirect policy 默认情况下是没有做 NS 隔离的，这样做会导致数据面冲突，所以对于 sock_key 我们额外加入了 NS 信息，这样同的 NS 内部，sock 重定向业务的五元组相同也不会出现冲突。

**高版本内核特性和合入**

为了使用高性能数据路径，我们将 bpf_redirect_peer 和 bpf_redirect_neigh patch 移植到 kernel5.4，开启 eBPF Host-Routing 功能。































































