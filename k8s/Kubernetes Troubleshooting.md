Kubernetes Troubleshooting

[toc]

# 排错概览



在排错过程中，`kubectl` 是最重要的工具，通常也是定位错误的起点。这里也列出一些常用的命令，在后续的各种排错过程中都会经常用到。



### 查看 Pod 状态以及运行节点

```sh
kubectl get pods -o wide
kubectl -n kube-system get pods -o wide
```



### 查看 Pod 事件

```sh
kubectl describe pod <pod-name>
```



### 查看 Node 状态

```sh
kubectl get nodes
kubectl describe node <node-name>
```



### kube-apiserver 日志

```sh
PODNAME=$(kubectl -n kube-system get pod -l component=kube-apiserver -o jsonpath='{.items[0].metadata.name}')
kubectl -n kube-system logs $PODNAME --tail 100
```

以上命令操作假设控制平面以 Kubernetes 静态 Pod 的形式来运行。如果 kube-apiserver 是用 systemd 管理的，则需要登录到 master 节点上，然后使用 journalctl -u kube-apiserver 查看其日志。



### kube-controller-manager 日志

```sh
PODNAME=$(kubectl -n kube-system get pod -l component=kube-controller-manager -o jsonpath='{.items[0].metadata.name}')
kubectl -n kube-system logs $PODNAME --tail 100
```

以上命令操作假设控制平面以 Kubernetes 静态 Pod 的形式来运行。如果 kube-controller-manager 是用 systemd 管理的，则需要登录到 master 节点上，然后使用 journalctl -u kube-controller-manager 查看其日志。



### kube-scheduler 日志

```sh
PODNAME=$(kubectl -n kube-system get pod -l component=kube-scheduler -o jsonpath='{.items[0].metadata.name}')
kubectl -n kube-system logs $PODNAME --tail 100
```

以上命令操作假设控制平面以 Kubernetes 静态 Pod 的形式来运行。如果 kube-scheduler 是用 systemd 管理的，则需要登录到 master 节点上，然后使用 journalctl -u kube-scheduler 查看其日志。



### kube-dns 日志

kube-dns 通常以 Addon 的方式部署，每个 Pod 包含三个容器，最关键的是 kubedns 容器的日志：

```sh
PODNAME=$(kubectl -n kube-system get pod -l k8s-app=kube-dns -o jsonpath='{.items[0].metadata.name}')
kubectl -n kube-system logs $PODNAME -c kubedns
```



### Kubelet 日志

Kubelet 通常以 systemd 管理。查看 Kubelet 日志需要首先 SSH 登录到 Node 上，推荐使用 [kubectl-node-shell](https://github.com/kvaps/kubectl-node-shell) 插件而不是为每个节点分配公网 IP 地址。比如：

```sh
curl -LO https://github.com/kvaps/kubectl-node-shell/raw/master/kubectl-node_shellchmod +x ./kubectl-node_shellsudo mv ./kubectl-node_shell /usr/local/bin/kubectl-node_shell
kubectl node-shell <node>journalctl -l -u kubelet
```



### Kube-proxy 日志

Kube-proxy 通常以 DaemonSet 的方式部署，可以直接用 kubectl 查询其日志

```sh
$ kubectl -n kube-system get pod -l component=kube-proxy
NAME               READY     STATUS    RESTARTS   AGE
kube-proxy-42zpn   1/1       Running   0          1d
kube-proxy-7gd4p   1/1       Running   0          3d
kube-proxy-87dbs   1/1       Running   0          4d
$ kubectl -n kube-system logs kube-proxy-42zpn
```



### 参考文档

- [hjacobs/kubernetes-failure-stories](https://github.com/hjacobs/kubernetes-failure-stories) 整理了一些公开的 Kubernetes 异常案例。
- https://docs.microsoft.com/en-us/azure/aks/troubleshooting 包含了 AKS 中排错的一般思路
- https://cloud.google.com/kubernetes-engine/docs/troubleshooting 包含了 GKE 中问题排查的一般思路
- https://www.oreilly.com/ideas/kubernetes-recipes-maintenance-and-troubleshooting





# troubleshooting



## troubleshooting-deployments



![img](D:\学习资料\笔记\k8s\k8s图\排错)

When you wish to deploy an application in Kubernetes, you usually define three components:

- A **Deployment** — which is a recipe for creating copies of your application called Pods.
- A **Service** — an internal load balancer that routes the traffic to Pods.
- An **Ingress** — a description of how the traffic should flow from outside the cluster to your Service.

  

Here's a quick visual recap.

![In Kubernetes your applications are exposed through two layers of load balancers: internal and external.](D:\学习资料\笔记\k8s\k8s图\92543837cbecdd1189ee0a6d68fa9434.svg)

![The internal load balancer is called Service, whereas the external one is called Ingress.](D:\学习资料\笔记\k8s\k8s图\202ec241073585d03480f5d58560ccb2.svg)

![Pods are not deployed directly. Instead, the Deployment creates the Pods and whatches over them.](D:\学习资料\笔记\k8s\k8s图\c0a7fd878b9da1b42ba1d84597a45235.svg)

- In Kubernetes your applications are exposed through two layers of load balancers: internal and external.

- The internal load balancer is called Service, whereas the external one is called Ingress.

- Pods are not deployed directly. Instead, the Deployment creates the Pods and whatches over them.



Assuming you wish to deploy a simple *Hello World* application, the YAML for such application should look similar to this:

**hello-world.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
  labels:
    track: canary
spec:
  selector:
    matchLabels:
      any-name: my-app
  template:
    metadata:
      labels:
        any-name: my-app
    spec:
      containers:
        - name: cont1
          image: learnk8s/app:1.0.0
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  ports:
    - port: 80
      targetPort: 8080
  selector:
    name: app
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - http:
      paths:
      - backend:
          service:
            name: my-service
            port:
              number: 80
        path: /
        pathType: Prefix
```

The definition is quite long, and it's easy to overlook how the components relate to each other.

For example:

- *When should you use port 80 and when port 8080?*
- *Should you create a new port for every Service so that they don't clash?*
- *Do label names matter? Should it be the same everywhere?*

Before focusing on the debugging, let's recap how the three components link to each other.

Let's start with Deployment and Service.



### Connecting Deployment and Service

The surprising news is that Service and Deployment aren't connected at all.

Instead, the Service points to the Pods directly and skips the Deployment altogether.

So what you should pay attention to is how the Pods and the Services are related to each other.

You should remember three things:

1. The Service selector should match at least one Pod's label.
2. The Service's `targetPort` should match the `containerPort` of the Pod.
3. The Service `port` can be any number. Multiple Services can use the same port because they have different IP addresses assigned.

The following diagram summarises how to connect the ports:

![Consider the following Pod exposed by a Service.](D:\学习资料\笔记\k8s\k8s图\2adc624ed44f39ac5577c42ed9c621bc.svg)

![When you create a Pod, you should define the port containerPort for each container in your Pods.](D:\学习资料\笔记\k8s\k8s图\441d2ca9f4ce838178e207c20bb17a24.svg)

![When you create a Service, you can define a port and a targetPort. But which one should you connect to the container?](D:\学习资料\笔记\k8s\k8s图\8454ae73acd4e2bf2ea874e77e768cfa.svg)

![targetPort and containerPort should always match.](D:\学习资料\笔记\k8s\k8s图\d8e5ddb44443d7b78e536aa7809b36f5.svg)

![If your container exposes port 3000, then the targetPort should match that number.](D:\学习资料\笔记\k8s\k8s图\79612e74f9a7d386c5558503068171b9.svg)

- Consider the following Pod exposed by a Service.
- When you create a Pod, you should define the port `containerPort` for each container in your Pods.
- When you create a Service, you can define a `port` and a `targetPort`. *But which one should you connect to the container?*
- `targetPort` and `containerPort` should always match.
- If your container exposes port 3000, then the `targetPort` should match that number.

If you look at the YAML, the labels and `ports`/`targetPort` should match:
**hello-world.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
  labels:
    track: canary
spec:
  selector:
    matchLabels:
      any-name: my-app
  template:
    metadata:
      labels:
        any-name: my-app
    spec:
      containers:
        - name: cont1
          image: learnk8s/app:1.0.0
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  ports:
    - port: 80
      targetPort: 8080
  selector:
    any-name: my-app
```

*What about the `track: canary` label at the top of the Deployment?*

*Should that match too?*

That label belongs to the deployment, and it's not used by the Service's selector to route traffic.

In other words, you can safely remove it or assign it a different value.

*And what about the `matchLabels` selector*?

[**It always has to match the Pod's labels**](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#resources-that-support-set-based-requirements) and it's used by the Deployment to track the Pods.

*Assuming that you made the correct change, how do you test it?*

You can check if the Pods have the right label with the following command:

```sh
$ kubectl get pods --show-labels
NAME                  READY   STATUS    LABELS
my-deployment-pv6pd   1/1     Running   any-name=my-app,pod-template-hash=7d6979fb54
my-deployment-f36rt   1/1     Running   any-name=my-app,pod-template-hash=7d6979fb54
```

Or if you have Pods belonging to several applications:

```sh
kubectl get pods --selector any-name=my-app --show-labels
```

Where `any-name=my-app` is the label `any-name: my-app`.

*Still having issues?*

You can also connect to the Pod!

You can use the `port-forward` command in kubectl to connect to the Service and test the connection.

```sh
kubectl port-forward service/<service name> 3000:8080
Forwarding from 127.0.0.1:3000 -> 8080
Forwarding from [::1]:3000 -> 8080
```

Where:

- `service/<service name>` is the name of the service — in the current YAML is "my-service".
- 3000 is the port that you wish to open on your computer.
- 8080 is the port exposed by the Service in the `port` field.

If you can connect, the setup is correct.

If you can't, you most likely misplaced a label or the port doesn't match.

### Connecting Service and Ingress

The next step in exposing your app is to configure the Ingress.

The Ingress has to know how to retrieve the Service to then connect the Pods and route traffic.

The Ingress retrieves the right Service by name and port exposed.

Two things should match in the Ingress and Service:

1. The `service.port` of the Ingress should match the `port` of the Service
2. The `service.name` of the Ingress should match the `name` of the Service

The following diagram summarises how to connect the ports:

![+](D:\学习资料\笔记\k8s\k8s图\f44dd6eea6391537dd84dee5344b7b83.svg)

![The Ingress has a field called servicePort.](D:\学习资料\笔记\k8s\k8s图\212eb8d3988e4b45ed7021f3546e4a57.svg)

![The Service port and the Ingress servicePort should always match.](D:\学习资料\笔记\k8s\k8s图\523463c584ac57b4fa7a34f4d2a8316b.svg)

![If you decide to assign port 80 to the service, you should change servicePort to 80 too.](D:\学习资料\笔记\k8s\k8s图\70fca3015bab436f3bcef5fc7c8a6c96.svg)

- You already know that the Service expose a `port`.
- The Ingress has a field called `servicePort`.
- The Service `port` and the Ingress `servicePort` should always match.
- If you decide to assign port 80 to the service, you should change `servicePort` to 80 too.

In practice, you should look at these lines:

**hello-world.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  ports:
    - port: 80
      targetPort: 8080
  selector:
    any-name: my-app
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - http:
      paths:
      - backend:
          service:
            name: my-service
            port:
              number: 80
        path: /
        pathType: Prefix
```

*How do you test that the Ingress works?*

You can use the same strategy as before with `kubectl port-forward`, but instead of connecting to a Service, you should connect to the Ingress controller.

First, retrieve the Pod name for the Ingress controller with:

```sh
kubectl get pods --all-namespaces
NAMESPACE   NAME                              READY STATUS
kube-system coredns-5644d7b6d9-jn7cq          1/1   Running
kube-system etcd-minikube                     1/1   Running
kube-system kube-apiserver-minikube           1/1   Running
kube-system kube-controller-manager-minikube  1/1   Running
kube-system kube-proxy-zvf2h                  1/1   Running
kube-system kube-scheduler-minikube           1/1   Running
kube-system nginx-ingress-controller-6fc5bcc  1/1   Running
```

Identify the Ingress Pod (which might be in a different Namespace) and describe it to retrieve the port:

```sh
kubectl describe pod nginx-ingress-controller-6fc5bcc \
 --namespace kube-system \
 | grep Ports
Ports:         80/TCP, 443/TCP, 18080/TCP
```

Finally, connect to the Pod:

```sh
kubectl port-forward nginx-ingress-controller-6fc5bcc 3000:80 --namespace kube-system
Forwarding from 127.0.0.1:3000 -> 80
Forwarding from [::1]:3000 -> 80
```

At this point, every time you visit port 3000 on your computer, the request is forwarded to port 80 on the Ingress controller Pod.

If you visit [http://localhost:3000](http://localhost:3000/), you should find the app serving a web page.

### Recap on ports

Here's a quick recap on what ports and labels should match:

1. The Service selector should match the label of the Pod
2. The Service `targetPort` should match the `containerPort` of the container inside the Pod
3. The Service port can be any number. Multiple Services can use the same port because they have different IP addresses assigned.
4. The `service.port` of the Ingress should match the `port` in the Service
5. The name of the Service should match the field `service.name` in the Ingress

Knowing how to structure your YAML definition is only part of the story.

*What happens when something goes wrong?*

Perhaps the Pod doesn't start, or it's crashing.

### 3 steps to troubleshoot Kubernetes deployments

It's essential to have a well defined mental model of how Kubernetes works before diving into debugging a broken deployment.

Since there are three components in every deployment, you should debug all of them in order, starting from the bottom.

1. You should make sure that your Pods are running, then
2. Focus on getting the Service to route traffic to the Pods and then
3. Check that the Ingress is correctly configured.

![You should start troubleshooting your deployments from the bottom. First, check that the Pod is Ready and Running.](D:\学习资料\笔记\k8s\k8s图\6aa1ab46349a22202f6cfe73022e9a12.svg)

![If the Pods is Ready, you should investigate if the Service can distribute traffic to the Pods.](D:\学习资料\笔记\k8s\k8s图\140d05f49bf4347e37f664e370441486.svg)

![Finally, you should examine the connection between the Service and the Ingress.](D:\学习资料\笔记\k8s\k8s图\c3f870a281346c4bdc64de1ca81d21a0.svg)

- You should start troubleshooting your deployments from the bottom. First, check that the Pod is *Ready* and *Running*.
- If the Pods is *Ready*, you should investigate if the Service can distribute traffic to the Pods.
- Finally, you should examine the connection between the Service and the Ingress.

#### 1. Troubleshooting Pods

Most of the time, the issue is in the Pod itself.

You should make sure that your Pods are *Running* and *Ready*.

*How do you check that?*

```sh
$ kubectl get pods
NAME                    READY STATUS            RESTARTS  AGE
app1                    0/1   ImagePullBackOff  0         47h
app2                    0/1   Error             0         47h
app3-76f9fcd46b-xbv4k   1/1   Running           1         47h
```

In the output above, the last Pod is *Running* and *Ready* — however, the first two Pods are neither *Running* nor *Ready*.

*How do you investigate on what went wrong?*

There are four useful commands to troubleshoot Pods:

1. `kubectl logs <pod name>` is helpful to retrieve the logs of the containers of the Pod.
2. `kubectl describe pod <pod name>` is useful to retrieve a list of events associated with the Pod.
3. `kubectl get pod <pod name>` is useful to extract the YAML definition of the Pod as stored in Kubernetes.
4. `kubectl exec -it <pod name> bash` is useful to run an interactive command within one of the containers of the Pod.

*Which one should you use?*

There isn't a one-size-fits-all.

Instead, you should use a combination of them.

##### Common Pods errors

Pods can have startup and runtime errors.

Startup errors include:

- ImagePullBackoff
- ImageInspectError
- ErrImagePull
- ErrImageNeverPull
- RegistryUnavailable
- InvalidImageName

Runtime errors include:

- CrashLoopBackOff
- RunContainerError
- KillContainerError
- VerifyNonRootError
- RunInitContainerError
- CreatePodSandboxError
- ConfigPodSandboxError
- KillPodSandboxError
- SetupNetworkError
- TeardownNetworkError

Some errors are more common than others.

The following is a list of the most common error and how you can fix them.

##### ImagePullBackOff

This error appears when Kubernetes isn't able to retrieve the image for one of the containers of the Pod.

There are three common culprits:

1. The image name is invalid — as an example, you misspelt the name, or the image does not exist.
2. You specified a non-existing tag for the image.
3. The image that you're trying to retrieve belongs to a private registry, and Kubernetes doesn't have credentials to access it.

The first two cases can be solved by correcting the image name and tag.

For the last, you should add the credentials to your private registry in a Secret and reference it in your Pods.

[The official documentation has an example about how you could to that.](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/)

##### CrashLoopBackOff

If the container can't start, then Kubernetes shows the CrashLoopBackOff message as a status.

Usually, a container can't start when:

1. There's an error in the application that prevents it from starting.
2. You [misconfigured the container](https://stackoverflow.com/questions/41604499/my-kubernetes-pods-keep-crashing-with-crashloopbackoff-but-i-cant-find-any-lo).
3. The Liveness probe failed too many times.

You should try and retrieve the logs from that container to investigate why it failed.

If you can't see the logs because your container is restarting too quickly, you can use the following command:

```sh
$ kubectl logs <pod-name> --previous
```

Which prints the error messages from the previous container.

##### RunContainerError

The error appears when the container is unable to start.

That's even before the application inside the container starts.

The issue is usually due to misconfiguration such as:

- Mounting a not-existent volume such as ConfigMap or Secrets.
- Mounting a read-only volume as read-write.

You should use `kubectl describe pod <pod-name>` to inspect and analyse the errors.

##### Pods in a *Pending* state

When you create a Pod, the Pod stays in the *Pending* state.

*Why?*

Assuming that your scheduler component is running fine, here are the causes:

1. The cluster doesn't have enough resources such as CPU and memory to run the Pod.
2. The current Namespace has a ResourceQuota object and creating the Pod will make the Namespace go over the quota.
3. The Pod is bound to a *Pending* PersistentVolumeClaim.

Your best option is to inspect the *Events* section in the `kubectl describe` command:

```sh
$ kubectl describe pod <pod name>
```

For errors that are created as a result of ResourceQuotas, you can inspect the logs of the cluster with:

```sh
$ kubectl get events --sort-by=.metadata.creationTimestamp
```

##### Pods in a not *Ready* state

If a Pod is *Running* but not *Ready* it means that the Readiness probe is failing.

When the Readiness probe is failing, the Pod isn't attached to the Service, and no traffic is forwarded to that instance.

A failing Readiness probe is an application-specific error, so you should inspect the *Events* section in `kubectl describe` to identify the error.

#### 2. Troubleshooting Services

If your Pods are *Running* and *Ready*, but you're still unable to receive a response from your app, you should check if the Service is configured correctly.

Services are designed to route the traffic to Pods based on their labels.

So the first thing that you should check is how many Pods are targeted by the Service.

You can do so by checking the Endpoints in the Service:

```sh
$ kubectl describe service my-service
Name:                     my-service
Namespace:                default
Selector:                 app=my-app
IP:                       10.100.194.137
Port:                     <unset>  80/TCP
TargetPort:               8080/TCP
Endpoints:                172.17.0.5:8080
```

An endpoint is a pair of `<ip address:port>`, and there should be at least one — when the Service targets (at least) a Pod.

If the "Endpoints" section is empty, there are two explanations:

1. You don't have any Pod running with the correct label (hint: you should check if you are in the right namespace).
2. You have a typo in the `selector` labels of the Service.

If you see a list of endpoints, but still can't access your application, then the `targetPort` in your service is the likely culprit.

*How do you test the Service?*

Regardless of the type of Service, you can use `kubectl port-forward` to connect to it:

```sh
$ kubectl port-forward service/<service-name> 3000:80
```

Where:

- `<service-name>` is the name of the Service.
- `3000` is the port that you wish to open on your computer.
- `80` is the port exposed by the Service.

#### 3. Troubleshooting Ingress

If you've reached this section, then:

- The Pods are *Running* and *Ready*.
- The Service distributes the traffic to the Pod.

But you still can't see a response from your app.

It means that most likely, the Ingress is misconfigured.

Since the Ingress controller is a third-party component in the cluster, there are different debugging techniques depending on the type of Ingress controller.

But before diving into Ingress specific tools, there's something straightforward that you could check.

The Ingress uses the `service.name` and `service.port` to connect to the Service.

You should check that those are correctly configured.

You can inspect that the Ingress is correctly configured with:

```sh
$ kubectl describe ingress my-ingress
Name:             my-ingress
Namespace:        default
Rules:
  Host        Path  Backends
  ----        ----  --------
  *
              /   my-service:80 (<error: endpoints "my-service" not found>)
```

If the *Backend* column is empty, then there must be an error in the configuration.

If you can see the endpoints in the *Backend* column, but still can't access the application, the issue is likely to be:

- How you exposed your Ingress to the public internet.
- How you exposed your cluster to the public internet.

You can isolate infrastructure issues from Ingress by connecting to the Ingress Pod directly.

First, retrieve the Pod for your Ingress controller (which could be located in a different namespace):

```sh
$ kubectl get pods --all-namespaces
NAMESPACE   NAME                              READY STATUS
kube-system coredns-5644d7b6d9-jn7cq          1/1   Running
kube-system etcd-minikube                     1/1   Running
kube-system kube-apiserver-minikube           1/1   Running
kube-system kube-controller-manager-minikube  1/1   Running
kube-system kube-proxy-zvf2h                  1/1   Running
kube-system kube-scheduler-minikube           1/1   Running
kube-system nginx-ingress-controller-6fc5bcc  1/1   Running
```

Describe it to retrieve the port:

```sh
kubectl describe pod nginx-ingress-controller-6fc5bcc
 --namespace kube-system \
 | grep Ports
    Ports:         80/TCP, 443/TCP, 8443/TCP
    Host Ports:    80/TCP, 443/TCP, 0/TCP
```

Finally, connect to the Pod:

```sh
$ kubectl port-forward nginx-ingress-controller-6fc5bcc 3000:80 --namespace kube-system
Forwarding from 127.0.0.1:3000 -> 80
Forwarding from [::1]:3000 -> 80
```

At this point, every time you visit port 3000 on your computer, the request is forwarded to port 80 on the Pod.

*Does it work now?*

- If it does, the issue is in the infrastructure. You should investigate how the traffic is routed to your cluster.
- If it doesn't work, the problem is in the Ingress controller. You should debug the Ingress.

If you still can't get the Ingress controller to work, you should start debugging it.

There are many different versions of Ingress controllers.

Popular options include Nginx, HAProxy, Traefik, etc.

You should consult the documentation of your Ingress controller to find a troubleshooting guide.

Since [Ingress Nginx](https://github.com/kubernetes/ingress-nginx) is the most popular Ingress controller, we included a few tips for it in the next section.

#### Debugging Ingress Nginx

The Ingress-nginx project has an [official plugin for Kubectl](https://kubernetes.github.io/ingress-nginx/kubectl-plugin/).

You can use `kubectl ingress-nginx` to:

- Inspect logs, backends, certs, etc.
- Connect to the Ingress.
- Examine the current configuration.

The three commands that you should try are:

- `kubectl ingress-nginx lint`, which checks the `nginx.conf`.
- `kubectl ingress-nginx backend`, to inspect the backend (similar to `kubectl describe ingress <ingress-name>`).
- `kubectl ingress-nginx logs`, to check the logs.

> Please notice that you might need to specify the correct namespace for your Ingress controller with `--namespace <name>`.

### Summary

Troubleshooting in Kubernetes can be a daunting task if you don't know where to start.

You should always remember to approach the problem bottom-up: start with the Pods and move up the stack with Service and Ingress.

The same debugging techniques that you learnt in this article can be applied to other objects such as:

- failing Jobs and CronJobs.
- StatefulSets and DaemonSets.

Many thanks to [Gergely Risko](https://github.com/errge), [Daniel Weibel](https://medium.com/@weibeld) and [Charles Christyraj](https://www.linkedin.com/in/charles-christyraj-0bab8a36/) for offering some invaluable suggestions.

Don't miss the next article!

Be the first to be notified when a new article or Kubernetes experiment is published.





## Load balancing and scaling long-lived connections in Kubernetes

> **TL;DR:** Kubernetes doesn't load balance long-lived connections, and some Pods might receive more requests than others. If you're using HTTP/2, gRPC, RSockets, AMQP or any other long-lived connection such as a database connection, you might want to consider client-side load balancing.

Kubernetes offers two convenient abstractions to deploy apps: Services and Deployments.

Deployments describe a recipe for what kind and how many copies of your app should run at any given time.

Each app is deployed as a Pod, and an IP address is assigned to it.

Services, on the other hand, are similar to load balancers.

They are designed to distribute the traffic to a set of Pods.

- In this diagram you have three instances of a single app and a load balancer.

![image-20210208212635040](D:\学习资料\笔记\k8s\k8s图\image-20210208212635040.png)

- The load balancer is called Service and has an IP address. Any incoming request is distributed to one of the Pods.

![image-20210208212720195](D:\学习资料\笔记\k8s\k8s图\image-20210208212720195.png)

- The Deployment defines a recipe to create more instances of the same Pod. You almost never deploy Pod individually.

![image-20210208212744741](D:\学习资料\笔记\k8s\k8s图\image-20210208212744741.png)

- Pods have IP address assigned to them.

![image-20210208212803472](D:\学习资料\笔记\k8s\k8s图\image-20210208212803472.png)



It is often useful to think about Services as a collection of IP address.

Every time you make a request to a Service, one of the IP addresses from that list is selected and used as the destination.

- Imagine issuing a request such as `curl 10.96.45.152` to the Service.

![image-20210208212900462](D:\学习资料\笔记\k8s\k8s图\image-20210208212900462.png)

- The Service picks one of the three Pods as the destination.

![image-20210208212932049](D:\学习资料\笔记\k8s\k8s图\image-20210208212932049.png)

- The traffic is forwarded to that instance.

![image-20210208212957321](D:\学习资料\笔记\k8s\k8s图\image-20210208212957321.png)

If you have two apps such as a front-end and a backend, you can use a Deployment and a Service for each and deploy them in the cluster.

When the front-end app makes a request, it doesn't need to know how many Pods are connected to the backend Service.

It could be one Pod, tens or hundreds.

The front-end app isn't aware of the individual IP addresses of the backend app either.

When it wants to make a request, that request is sent to the backend Service which has an IP address that doesn't change.

- The red Pod issues a request to an internal (beige) component. Instead of choosing one of the Pod as the destination, the red Pod issues the request to the Service.

![image-20210208213029198](D:\学习资料\笔记\k8s\k8s图\image-20210208213029198.png)

- The Service selects one of the ready Pods as the destination.

![image-20210208213046535](D:\学习资料\笔记\k8s\k8s图\image-20210208213046535.png)

- The traffic flows from the red Pod to the light brown Pod.

![image-20210208213110426](D:\学习资料\笔记\k8s\k8s图\image-20210208213110426.png)



- Notice how the red Pod doesn't know how many Pods are hidden behind the Service.

![image-20210208213138100](D:\学习资料\笔记\k8s\k8s图\image-20210208213138100.png)

*But what's the load balancing strategy for the Service?*

*It is round-robin, right?*

Sort of.



### Load balancing in Kubernetes Services

Kubernetes Services don't exist.

There's no process listening on the IP address and port of the Service.

> You can check that this is the case by accessing any node in your Kubernetes cluster and executing `netstat -ntlp`.

Even the IP address can't be found anywhere.

The IP address for a Service is allocated by the control plane in the controller manager and stored in the database — etcd.

That same IP address is then used by another component: kube-proxy.

Kube-proxy reads the list of IP addresses for all Services and writes a collection of iptables rules in every node.

The rules are meant to say: "if you see this Service IP address, instead rewrite the request and pick one of the Pod as the destination".

The Service IP address is used only as a placeholder — that's why there is no process listening on the IP address or port.

- Consider a cluster with three Nodes. Each Node has a Pod deployed.

![image-20210208213231366](D:\学习资料\笔记\k8s\k8s图\image-20210208213231366.png)

- The beige Pods are part of a Service. Services don't exist, so the diagram has the component grayed out.

![image-20210208213312118](D:\学习资料\笔记\k8s\k8s图\image-20210208213312118.png)

- The red Pod wants to issue a request to the Service and eventually reaches one of the beige Pods.

![image-20210208213334353](D:\学习资料\笔记\k8s\k8s图\image-20210208213334353.png)

- But Service don't exist. There's no process listening on the Service IP address. *How does it work?*

![image-20210208213357060](D:\学习资料\笔记\k8s\k8s图\image-20210208213357060.png)

- Before the request is dispatched from the Node, it is intercepted by iptables rules.

![image-20210208213421907](D:\学习资料\笔记\k8s\k8s图\image-20210208213421907.png)

- The iptables rules know that the Service doesn't exist and proceed to replace the IP address of the Service with one of the IP addresses of the Pods beloning to that Service.

![image-20210208213449344](D:\学习资料\笔记\k8s\k8s图\image-20210208213449344.png)

- The request has a real IP address as the destination and it can proceed normally.

![image-20210208213510587](D:\学习资料\笔记\k8s\k8s图\image-20210208213510587.png)

- Depening on your particular network implementation, the request finally reaches the Pod.

![image-20210208213530282](D:\学习资料\笔记\k8s\k8s图\image-20210208213530282.png)



*Does iptables use round-robin?*

No, iptables is primarily used for firewalls, and it is not designed to do load balancing.

However, you could [craft a smart set of rules that could make iptables behave like a load balancer](https://scalingo.com/blog/iptables#load-balancing).

And this is precisely what happens in Kubernetes.

If you have three Pods, kube-proxy writes the following rules:

1. select Pod 1 as the destination with a likelihood of 33%. Otherwise, move to the next rule
2. choose Pod 2 as the destination with a probability of 50%. Otherwise, move to the following rule
3. select Pod 3 as the destination (no probability)

The compound probability is that Pod 1, Pod 2 and Pod 3 have all have a one-third chance (33%) to be selected.

![image-20210208213556510](D:\学习资料\笔记\k8s\k8s图\image-20210208213556510.png)

Also, there's no guarantee that Pod 2 is selected after Pod 1 as the destination.

> Iptables use the [statistic module](http://ipset.netfilter.org/iptables-extensions.man.html#lbCD) with `random` mode. So the load balancing algorithm is random.

Now that you're familiar with how Services work let's have a look at more exciting scenarios.



### Long-lived connections don't scale out of the box in Kubernetes

With every HTTP request started from the front-end to the backend, a new TCP connection is opened and closed.

If the front-end makes 100 HTTP requests per second to the backend, 100 different TCP connections are opened and closed in that second.

You can improve the latency and save resources if you open a TCP connection and reuse it for any subsequent HTTP requests.

The HTTP protocol has a feature called HTTP keep-alive, or HTTP connection reuse that uses a single TCP connection to send and receive multiple HTTP requests and responses.

![Opening and closing connections VS HTTP connection reuse](D:\学习资料\笔记\k8s\k8s图\1b3b1c24bf7ba7baa98eb16330943792.svg)

It doesn't work out of the box; your server and client should be configured to use it.

The change itself is straightforward, and it's available in most languages and frameworks.

Here a few examples on how to implement keep-alive in different languages:

- [Keep-alive in Node.js](https://medium.com/@onufrienkos/keep-alive-connection-on-inter-service-http-requests-3f2de73ffa1)
- [Keep-alive in Spring boot](https://www.baeldung.com/httpclient-connection-management)
- [Keep-alive in Python](https://blog.insightdatascience.com/learning-about-the-http-connection-keep-alive-header-7ebe0efa209d)
- [Keep-alive in .NET](https://docs.microsoft.com/en-us/dotnet/api/system.net.httpwebrequest.keepalive?view=netframework-4.8)

*What happens when you use keep-alive with a Kubernetes Service?*

Let's imagine that front-end and backend support keep-alive.

You have a single instance of the front-end and three replicas for the backend.

The front-end makes the first request to the backend and opens the TCP connection.

The request reaches the Service, and one of the Pod is selected as the destination.

The backend Pod replies and the front-end receives the response.

But instead of closing the TCP connection, it is kept open for subsequent HTTP requests.

*What happens when the front-end issues more requests?*

They are sent to the same Pod.

*Isn't iptables supposed to distribute the traffic?*

It is.

There is a single TCP connection open, and iptables rule were invocated the first time.

One of the three Pods was selected as the destination.

Since all subsequent requests are channelled through the same TCP connection, [iptables isn't invoked anymore.](https://scalingo.com/blog/iptables#load-balancing)



- The red Pod issues a request to the Service.

![image-20210208213713912](D:\学习资料\笔记\k8s\k8s图\image-20210208213713912.png)

- You already know what happens next. Services don't exist, but iptables rules intercept the requests.

![image-20210208213732840](D:\学习资料\笔记\k8s\k8s图\image-20210208213732840.png)

- One of the Pods that belong the Service is selected as the destination.

![image-20210208213801232](D:\学习资料\笔记\k8s\k8s图\image-20210208213801232.png)

- Finally the request reaches the Pod. At this point, a persistent connection between the two Pods is established.

![image-20210208213822941](D:\学习资料\笔记\k8s\k8s图\image-20210208213822941.png)

- Any subsequet request from the red Pod reuses the existing open connection.

![Any subsequet request from the red Pod reuses the existing open connection.](D:\学习资料\笔记\k8s\k8s图\12ea238c567fff0cb231bd15e40f0f79.svg)

So you have now achieved better latency and throughput, but you lost the ability to scale your backend.

Even if you have two backend Pods that can receive requests from the frontend Pod, only one is actively used.

*Is it fixable?*

Since Kubernetes doesn't know how to load balance persistent connections, you could step in and fix it yourself.

Services are a collection of IP addresses and ports — usually called endpoints.

Your app could retrieve the list of endpoints from the Service and decide how to distribute the requests.

As a first try, you could open a persistent connection to every Pod and round-robin requests to them.

Or you could [implement more sophisticated load balancing algorithms](https://blog.twitter.com/engineering/en_us/topics/infrastructure/2019/daperture-load-balancer.html).

The client-side code that executes the load balancing should follow the logic below:

1. retrieve a list of endpoints from the Service
2. for each of them, open a connection and keep it open
3. when you need to make a request, pick one of the open connections
4. on a regular interval refresh the list of endpoints and remove or add new connections



- Instead of having the red Pod issuing a request to your Service, you could load balance the request client-side.

![image-20210208213920174](D:\学习资料\笔记\k8s\k8s图\image-20210208213920174.png)

- You could write some code that asks what Pods are part of the Service.

![image-20210208213938696](D:\学习资料\笔记\k8s\k8s图\image-20210208213938696.png)

- Once you have that list, you could store it locally and use it to connect to the Pods.

![image-20210208213958453](D:\学习资料\笔记\k8s\k8s图\image-20210208213958453.png)

- You are in charge of the load balancing algorithm.

![You are in charge of the load balancing algorithm.](D:\学习资料\笔记\k8s\k8s图\56193238b42336fccbf0375465b155f1.svg)

*Does this problem apply only to HTTP keep-alive?*



### Client-side load balancing

HTTP isn't the only protocol that can benefit from long-lived TCP connections.

If your app uses a database, the connection isn't opened and closed every time you wish to retrieve a record or a document.

Instead, the TCP connection is established once and kept open.

If your database is deployed in Kubernetes using a Service, you might experience the same issues as the previous example.

There's one replica in your database that is utilised more than the others.

Kube-proxy and Kubernetes don't help to balance persistent connections.

Instead, you should take care of load balancing the requests to your database.

Depending on the library that you use to connect to the database, you might have different options.

The following example is from a clustered MySQL database called from Node.js:

**index.js**

```sh
var mysql = require('mysql');
var poolCluster = mysql.createPoolCluster();

var endpoints = /* retrieve endpoints from the Service */

for (var [index, endpoint] of endpoints) {
  poolCluster.add(`mysql-replica-${index}`, endpoint);
}

// Make queries to the clustered MySQL database
```

As you can imagine, several other protocols work over long-lived TCP connections.

Here you can read a few examples:

- Websockets and secured WebSockets
- HTTP/2
- gRPC
- RSockets
- AMQP

You might recognise most of the protocols above.

*So if these protocols are so popular, why isn't there a standard answer to load balancing?*

*Why does the logic have to be moved into the client?*

*Is there a native solution in Kubernetes?*

Kube-proxy and iptables are designed to cover the most popular use cases of deployments in a Kubernetes cluster.

But they are mostly there for convenience.

If you're using a web service that exposes a REST API, then you're in luck — this use case usually doesn't reuse TCP connections, and you can use any Kubernetes Service.

But as soon as you start using persistent TCP connections, you should look into how you can evenly distribute the load to your backends.

Kubernetes doesn't cover that specific use case out of the box.

However, there's something that could help.



### Load balancing long-lived connections in Kubernetes

Kubernetes has four different kinds of Services:

1. ClusterIP
2. NodePort
3. LoadBalancer
4. Headless

The first three Services have a virtual IP address that is used by kube-proxy to create iptables rules.

But the fundamental building block of all kinds of the Services is the Headless Service.

The headless Service doesn't have an assigned IP address and is only a mechanism to collect a list of Pod IP addresses and ports (also called endpoints).

Every other Service is built on top of the Headless Service.

The ClusterIP Service is a Headless Service with some extra features:

- the control plane assigns it an IP address
- kube-proxy iterates through all the IP addresses and creates iptables rules

So you could ignore kube-proxy all together and always use the list of endpoints collected by the Headless Service to load balance requests client-side.

*But can you imagine adding that logic to all apps deployed in the cluster?*

If you have an existing fleet of applications, this might sound like an impossible task.

But there's an alternative.



### Service meshes to the rescue

You probably already noticed that the client-side load balancing strategy is quite standard.

When the app starts, it should

- retrieve a list of IP addresses from the Service
- open and maintain a pool of connections
- periodically refresh the pool by adding and removing endpoints

As soon as it wishes to make a request, it should:

- pick one of the available connections using a predefined logic such as round-robin
- issue the request

The steps above are valid for WebSockets connections as well as gRPC and AMQP.

You could extract that logic in a separate library and share it with all apps.

Instead of writing a library from scratch, you could use a Service mesh such as Istio or Linkerd.

Service meshes augment your app with a new process that:

- automatically discovers IP addresses Services
- inspects connections such as WebSockets and gRPC
- load-balances requests using the right protocol

Service meshes can help you to manage the traffic inside your cluster, but they aren't exactly lightweight.

Other options include using a library such as Netflix Ribbon, a programmable proxy such as Envoy or just ignore it.

*What happens if you ignore it?*

You can ignore the load balancing and still don't notice any change.

There are a couple of scenarios that you should consider.

If you have more clients than servers, there should be limited issues.

Imagine you have five clients opening persistent connections to two servers.

Even if there's no load balancing, both servers likely utilised.

![More clients than servers](D:\学习资料\笔记\k8s\k8s图\c15fdd00f8bfae23e2f7ace4d81ea53f.svg)

The connections might not be distributed evenly (perhaps four ended up connecting to the same server), but overall there's a good chance that both servers are utilised.

What's more problematic is the opposite scenario.

If you have fewer clients and more servers, you might have some underutilised resources and a potential bottleneck.

Imagine having two clients and five servers.

At best, two persistent connections to two servers are opened.

The remaining servers are not used at all.

![More servers than clients](D:\学习资料\笔记\k8s\k8s图\8d35ec17be524303a3511b2288e36227.svg)

If the two servers can't handle the traffic generated by the clients, horizontal scaling won't help.



### Summary

Kubernetes Services are designed to cover most common uses for web applications.

However, as soon as you start working with application protocols that use persistent TCP connections, such as databases, gRPC, or WebSockets, they fall apart.

Kubernetes doesn't offer any built-in mechanism to load balance long-lived TCP connections.

Instead, you should code your application so that it can retrieve and load balance upstreams client-side.

Many thanks to [Daniel Weibel](https://medium.com/@weibeld), [Gergely Risko](https://github.com/errge) and [Salman Iqbal](https://twitter.com/soulmaniqbal) for offering some invaluable suggestions.

And to [Chris Hanson](https://twitter.com/CloudNativChris) who suggested to include the detailed explanation (and flow chart) on how iptables rules work in practice.





## Allocatable memory and CPU in Kubernetes Nodes



![image-20210209105705642](D:\学习资料\笔记\k8s\k8s图\image-20210209105705642.png)



**TL;DR:** Not all CPU and memory in your Kubernetes nodes can be used to run Pods.

*The infographic below summarises how memory and CPU are allocated in Google Kubernetes Engine (GKE), Elastic Kubernetes Service (EKS) and Azure Kubernetes Service (AKS).*

![Allocatable resources in Kubernetes](D:\学习资料\笔记\k8s\k8s图\7c56d074a37a452bbc8ae0132b84cf55.png)



### How resources are allocated in cluster nodes

Pods deployed in your Kubernetes cluster consume resources such as memory, CPU and storage.

**However, not all resources in a Node can be used to run Pods.**

The operating system and the kubelet require memory and CPU too, and you should cater for those extra resources.

If you look closely at a single Node, you can divide the available resources in:

1. Resources needed to run the operating system and system daemons such as SSH, systemd, etc.
2. Resources necessary to run Kubernetes agents such as the Kubelet, the container runtime, [node problem detector](https://github.com/kubernetes/node-problem-detector), etc.
3. Resources available to Pods
4. Resources reserved to the [eviction threshold](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/#eviction-thresholds)

![The amount of compute resources that are available to Pods](D:\学习资料\笔记\k8s\k8s图\d9e4cd38d833eed0da570d13b04078ad.svg)

As you can guess, [all of those quotas are customisable.](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/#eviction-thresholds)

But please notice that reserving 100MB of memory for the operating system doesn't mean that the OS is limited to use only that amount.

It could use more (or less) resources — you're just allocating and estimating memory and CPU usage at best of your abilities.

*But how do you decide how to assign resources?*

Unfortunately, there isn't a *fixed* answer as it depends on your cluster.

However, there's consensus in the major managed Kubernetes services [Google Kubernetes Engine (GKE)](https://cloud.google.com/kubernetes-engine), [Azure Kubernetes Service (AKS)](https://docs.microsoft.com/en-us/azure/aks/intro-kubernetes), and [Elastic Kubernetes Service (EKS)](https://aws.amazon.com/eks/), and it's worth discussing how they partition the available resources.



### Google Kubernetes Engine (GKE)

Google Kubernetes Engine (GKE) has [a well-defined list of rules to assign memory and CPU to a Node](https://cloud.google.com/kubernetes-engine/docs/concepts/cluster-architecture#memory_cpu).

For memory resources, GKE reserves the following:

- 255 MiB of memory for machines with less than 1 GB of memory
- 25% of the first 4GB of memory
- 20% of the next 4GB of memory (up to 8GB)
- 10% of the next 8GB of memory (up to 16GB)
- 6% of the next 112GB of memory (up to 128GB)
- 2% of any memory above 128GB

For CPU resources, GKE reserves the following:

- 6% of the first core
- 1% of the next core (up to 2 cores)
- 0.5% of the next 2 cores (up to 4 cores)
- 0.25% of any cores above 4 cores

*Let's look at an example.*

A virtual machine of type `n1-standard-2` has 2 vCPU and 7.5GB of memory.

According to the above rules the CPU reserved is:

```
Allocatable CPU = 0.06 * 1 (first core) + 0.01 * 1 (second core)
```

**That totals to 70 millicores or 3.5% — a modest amount.**

The allocatable memory is more interesting:

```
Allocatable memory = 0.25 * 4 (first 4GB) + 0.2 * 3.5 (remaining 3.5GB)
```

**The total is 1.7GB of memory reserved to the kubelet.**

At this point, you might think that the remaining memory `7.5GB - 1.7GB = 5.8GB` is something that you can use for your Pods.

*Not really.*

**The kubelet reserves an extra 100M of CPU and 100MB of memory for the Operating System and 100MB for the eviction threshold.**

The total CPU reserved is 170 millicores (or about 8%).

However, you started with 7.5GB of memory, but you can only use 5.6GB for your Pods.

**That's close to ~75% of the overall capacity.**

![Allocatable CPU and memory in Google Kubernetes Engine (GKE)](D:\学习资料\笔记\k8s\k8s图\8f4fee7a10bdcb84afc7666e50193215.svg)

You can be more efficient if you decide to use larger instances.

The instance type `n1-standard-96` has 96 vCPU and 360GB of memory.

If you do the maths that amounts to:

- 405 millicores are reserved for Kubelet and operating system
- 14.16GB of memory are reserved to Operating System, kubernetes agent and eviction threshold.

**In this extreme case, only 4% of memory is not allocatable.**



### Elastic Kubernetes Service (EKS)

Let's explore Elastic Kubernetes Service (EKS) allocations.

> Unfortunately, Elastic Kubernetes Service (EKS) doesn't offer documentation for allocatable resources. You can [rely on their code implementation](https://github.com/awslabs/amazon-eks-ami/blob/master/files/bootstrap.sh#L278) to extract the values.

EKS reserves the following memory for each Node:

```
Reserved memory = 255MiB + 11MiB * MAX_POD_PER_INSTANCE
```

*What's `MAX_POD_PER_INSTANCE`?*

**In Amazon Web Service, each instance type has a different upper limit on how many Pods it can run.**

For example, an `m5.large` instance can only run 29 Pods, but an `m5.4xlarge` can run up to 234.

[You can view the full list here.](https://github.com/awslabs/amazon-eks-ami/blob/master/files/eni-max-pods.txt)

If you were to select an `m5.large`, the memory reserved for the kubelet and agents is:

```
Reserved memory = 255Mi + 11MiB * 29 = 574MiB
```

For CPU resources, [EKS copies the GKE implementation](https://github.com/awslabs/amazon-eks-ami/blob/master/files/bootstrap.sh#L171) and reserves:

- 6% of the first core
- 1% of the next core (up to 2 cores)
- 0.5% of the next 2 cores (up to 4 cores)
- 0.25% of any cores above 4 cores

*Let's have a look at an example.*

An `m5.large` instance has 2 vCPU and 8GiB of memory:

1. You know already from the calculation above that 574MiB of memory is reserved to the kubelet.
2. An extra 100M of CPU and 100MB of memory is reserved to the Operating System and 100MB for the eviction threshold.
3. The reserved allocation for the CPU is the same 70 millicores (same as the `n1-standard-2` since they both 2 vCPU and the quota is calculated similarly).

![Allocatable CPU and memory in Elastic Kubernetes Service (EKS)](D:\学习资料\笔记\k8s\k8s图\5c8847e5dc29983899cd1601ae0b3db8.svg)

It's interesting to note that the memory allocatable to Pods is almost 90% in this case.



### Azure Kubernetes Service

Azure offers [a detailed explanation of their resource allocations](https://docs.microsoft.com/en-us/azure/aks/concepts-clusters-workloads#resource-reservations).

The memory reserved for the Kubelet is:

- 255 MiB of memory for machines with less than 1 GB of memory
- 25% of the first 4GB of memory
- 20% of the next 4GB of memory (up to 8GB)
- 10% of the next 8GB of memory (up to 16GB)
- 6% of the next 112GB of memory (up to 128GB)
- 2% of any memory above 128GB

Notice how the allocation is the same as Google Kubernetes Engine (GKE).

The CPU reserved for the Kubelet follows the following table:

| CPU CORES | CPU Reserved (in millicores) |
| :-------: | :--------------------------: |
|     1     |              60              |
|     2     |             100              |
|     4     |             140              |
|     8     |             180              |
|    16     |             260              |
|    32     |             420              |
|    64     |             740              |

The values are slightly higher than their counterparts but still modest.

Overall, CPU and memory reserved for AKS are remarkably similar to Google Kubernetes Engine (GKE).

*There's one departure, though.*

**The hard eviction threshold in Google and Amazon's offering is 100MB, but a staggering 750MiB in AKS.**

Let's have a look at a D3 v2 instance that has 8GiB of memory and 2 vCPU.

![Allocatable CPU and memory in Azure Kubernetes Service (AKS)](D:\学习资料\笔记\k8s\k8s图\2a9b23604bf2dd221d07dfb7f00ad544.svg)

**Only 55% of the available memory is allocatable to Pods, in this scenario.**



### Summary

You might be tempted to conclude that larger instances are the way to go as you maximise the allocable memory and CPU.

**Unfortunately, cost is only one factor when designing your cluster.**

If you're running large nodes you should also consider:

1. The **overhead on the Kubernetes agents that run on the node** — such as the container runtime (e.g. Docker), the kubelet, and cAdvisor.
2. **Your high-availability (HA) strategy.** Pods can be deployed to a selected number of Nodes
3. **Blast radius.** If you have only a few nodes, then the impact of a failing node is bigger than if you have many nodes.
4. **Autoscaling is less cost-effective** as the next increment is a (very) large Node.

*Smaller nodes aren't a silver bullet either.*

So you should architect your cluster for the type of workloads that you run rather than following the most common option.

If you wish to explore the pros and cons of different instance types, you should check out this sister blog post [Architecting Kubernetes clusters — choosing a worker node size ](https://learnk8s.io/kubernetes-node-size).





## Architecting Kubernetes clusters — choosing a worker node size



**Welcome to Bite-sized Kubernetes learning** — a regular column on the most interesting questions that we see online and during our workshops answered by a Kubernetes expert.

> Today's answers are curated by [Daniel Weibel](https://medium.com/@weibeld). Daniel is a software engineer and instructor at Learnk8s.

*If you wish to have your question featured on the next episode, [please get in touch via email](mailto:hello@learnk8s.io) or [you can tweet us at @learnk8s](https://twitter.com/learnk8s).*

Did you miss the previous episodes? [You can find them here.](https://learnk8s.io/bite-sized)

**When you create a Kubernetes cluster, one of the first questions that pops up is: "what type of worker nodes should I use, and how many of them?".**

If you're building an on-premises cluster, should you order some last-generation power servers, or use the dozen or so old machines that are lying around in your data centre?

Or if you're using a managed Kubernetes service like [Google Kubernetes Engine (GKE)](https://cloud.google.com/kubernetes-engine/), should you use eight `n1-standard-1` or two `n1-standard-4` instances to achieve your desired computing capacity?



### Cluster capacity

In general, a Kubernetes cluster can be seen as abstracting a set of individual nodes as a big "super node".

The total compute capacity (in terms of CPU and memory) of this super node is the sum of all the constituent nodes' capacities.

There are multiple ways to achieve a desired target capacity of a cluster.

For example, imagine that you need a cluster with a total capacity of 8 CPU cores and 32 GB of RAM.

> For example, because the set of applications that you want to run on the cluster require this amount of resources.

Here are just two of the possible ways to design your cluster:

![Small vs. large nodes](D:\学习资料\笔记\k8s\k8s图\3df0bc0c1da6077acc91967cf0f809c4.svg)

Both options result in a cluster with the same capacity — but the left option uses 4 smaller nodes, whereas the right one uses 2 larger nodes.

*Which is better?*

To approach this question, let's look at the pros and cons of the two opposing directions of "few large nodes" and "many small nodes".

> Note that "nodes" in this article always refers to worker nodes. The choice of number and size of master nodes is an entirely different topic.



### Few large nodes

The most extreme case in this direction would be to have a single worker node that provides the entire desired cluster capacity.

In the above example, this would be a single worker node with 16 CPU cores and 16 GB of RAM.

*Let's look at the advantages such an approach could have.*



#### 1. Less management overhead

Simply said, having to manage a small number of machines is less laborious than having to manage a large number of machines.

Updates and patches can be applied more quickly, the machines can be kept in sync more easily.

Furthermore, the absolute number of expected failures is smaller with few machines than with many machines.

*However, note that this applies primarily to bare metal servers and not to cloud instances.*

If you use cloud instances (as part of a managed Kubernetes service or your own Kubernetes installation on cloud infrastructure) you outsource the management of the underlying machines to the cloud provider.

Thus managing, 10 nodes in the cloud is not much more work than managing a single node in the cloud.



#### 2. Lower costs per node

While a more powerful machine is more expensive than a low-end machine, the price increase is not necessarily linear.

In other words, a single machine with 10 CPU cores and 10 GB of RAM might be cheaper than 10 machines with 1 CPU core and 1 GB of RAM.

*However, note that this likely doesn't apply if you use cloud instances.*

In the current pricing schemes of the major cloud providers [Amazon Web Services](https://aws.amazon.com/ec2/pricing/on-demand/), [Google Cloud Platform](https://cloud.google.com/compute/vm-instance-pricing), and [Microsoft Azure](https://azure.microsoft.com/en-us/pricing/calculator/#virtual-machines), the instance prices increase linearly with the capacity.

For example, on Google Cloud Platform, 64 `n1-standard-1` instances cost you exactly the same as a single `n1-standard-64` instance — and both options provide you 64 CPU cores and 240 GB of memory.

So, in the cloud, you typically can't save any money by using larger machines.



#### 3. Allows running resource-hungry applications

Having large nodes might be simply a requirement for the type of application that you want to run in the cluster.

For example, if you have a machine learning application that requires 8 GB of memory, you can't run it on a cluster that has only nodes with 1 GB of memory.

But you can run it on a cluster that has nodes with 10 GB of memory.

*Having seen the pros, let's see what the cons are.*



#### 1. Large number of pods per node

Running the same workload on fewer nodes naturally means that more pods run on each node.

*This could become an issue.*

The reason is that each pod introduces some overhead on the Kubernetes agents that run on the node — such as the container runtime (e.g. Docker), the kubelet, and cAdvisor.

For example, the kubelet executes regular liveness and readiness probes against each container on the node — more containers means more work for the kubelet in each iteration.

The cAdvisor collects resource usage statistics of all containers on the node, and the kubelet regularly queries this information and exposes it on its API — again, this means more work for both the cAdvisor and the kubelet in each iteration.

If the number of pods becomes large, these things might start to slow down the system and even make it unreliable.

![Many pods per node](D:\学习资料\笔记\k8s\k8s图\7572c4bb976fabbefb69afa0481c6831.svg)

There are reports of [nodes being reported as non-ready](https://github.com/kubernetes/kubernetes/issues/45419) because the regular kubelet health checks took too long for iterating through all the containers on the node.

*For these reasons, Kubernetes [recommends a maximum number of 110 pods per node](https://kubernetes.io/docs/setup/best-practices/cluster-large/).*

Up to this number, Kubernetes has been tested to work reliably on common node types.

Depending on the performance of the node, you might be able to successfully run more pods per node — but it's hard to predict whether things will run smoothly or you will run into issues.

*Most managed Kubernetes services even impose hard limits on the number of pods per node:*

- On [Amazon Elastic Kubernetes Service (EKS)](https://github.com/awslabs/amazon-eks-ami/blob/master/files/eni-max-pods.txt), the maximum number of pods per node depends on the node type and ranges from 4 to 737.
- On [Google Kubernetes Engine (GKE)](https://cloud.google.com/kubernetes-engine/quotas), the limit is 100 pods per node, regardless of the type of node.
- On [Azure Kubernetes Service (AKS)](https://docs.microsoft.com/bs-latn-ba/azure/aks/configure-azure-cni#maximum-pods-per-node), the default limit is 30 pods per node but it can be increased up to 250.

So, if you plan to run a large number of pods per node, you should probably test beforehand if things work as expected.



#### 2. Limited replication

A small number of nodes may limit the effective degree of replication for your applications.

For example, if you have a high-availability application consisting of 5 replicas, but you have only 2 nodes, then the effective degree of replication of the app is reduced to 2.

This is because the 5 replicas can be distributed only across 2 nodes, and if one of them fails, it may take down multiple replicas at once.

On the other hand, if you have at least 5 nodes, each replica can run on a separate node, and a failure of a single node takes down at most one replica.

Thus, if you have high-availability requirements, you might require a certain minimum number of nodes in your cluster.



#### 3. Higher blast radius

If you have only a few nodes, then the impact of a failing node is bigger than if you have many nodes.

For example, if you have only two nodes, and one of them fails, then about half of your pods disappear.

Kubernetes can reschedule workloads of failed nodes to other nodes.

However, if you have only a few nodes, the risk is higher that there is not enough spare capacity on the remaining node to accommodate all the workloads of the failed node.

The effect is that parts of your applications will be permanently down until you bring up the failed node again.

So, if you want to reduce the impact of hardware failures, you might want to choose a larger number of nodes.



#### 4. Large scaling increments

Kubernetes provides a [Cluster Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler) for cloud infrastructure that allows to automatically add or remove nodes based on the current demand.

If you use large nodes, then you have a large scaling increment, which makes scaling more clunky.

For example, if you only have 2 nodes, then adding an additional node means increasing the capacity of the cluster by 50%.

This might be much more than you actually need, which means that you pay for unused resources.

So, if you plan to use cluster autoscaling, then smaller nodes allow a more fluid and cost-efficient scaling behaviour.

*Having discussed the pros and cons of few large nodes, let's turn to the scenario of many small nodes.*



### Many small nodes

This approach consists of forming your cluster out of many small nodes instead of few large nodes.

*What are the pros and cons of this approach?*

The pros of using many small nodes correspond mainly to the cons of using few large nodes.



#### 1. Reduced blast radius

If you have more nodes, you naturally have fewer pods on each node.

For example, if you have 100 pods and 10 nodes, then each node contains on average only 10 pods.

Thus, if one of the nodes fails, the impact is limited to a smaller proportion of your total workload.

Chances are that only some of your apps are affected, and potentially only a small number of replicas so that the apps as a whole stay up.

Furthermore, there are most likely enough spare resources on the remaining nodes to accommodate the workload of the failed node, so that Kubernetes can reschedule all the pods, and your apps return to a fully functional state relatively quickly.



#### 2. Allows high replication

If you have replicated high-availability apps, and enough available nodes, the Kubernetes scheduler can assign each replica to a different node.

> You can influence scheduler's the placement of pods with [node affinites](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#node-affinity), [pod affinities/anti-affinities](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#inter-pod-affinity-and-anti-affinity), and [taints and tolerations](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/).

This means that if a node fails, there is at most one replica affected and your app stays available.

*Having seen the pros of using many small nodes, what are the cons?*



#### 1. Large number of nodes

If you use smaller nodes, you naturally need more of them to achieve a given cluster capacity.

*But large numbers of nodes can be a challenge for the Kubernetes control plane.*

For example, every node needs to be able to communicate with every other node, which makes the number of possible communication paths grow by square of the number of nodes — all of which has to be managed by the control plane.

The node controller in the Kubernetes controller manager regularly iterates through all the nodes in the cluster to run health checks — more nodes mean thus more load for the node controller.

More nodes mean also more load on the etcd database — each kubelet and kube-proxy results in a [watcher](https://etcd.io/docs/v3.3.12/dev-guide/interacting_v3/#watch-key-changes) client of etcd (through the API server) that etcd must broadcast object updates to.

In general, each worker node imposes some overhead on the system components on the master nodes.

![Many worker nodes per cluster](D:\学习资料\笔记\k8s\k8s图\b26e876ca8626d3b0bf6752ffe192944.svg)

Officially, Kubernetes claims to support clusters with [up to 5000 nodes](https://kubernetes.io/docs/setup/best-practices/cluster-large/).

However, in practice, 500 nodes may already [pose non-trivial challenges](https://www.lfasiallc.com/wp-content/uploads/2017/11/BoF_-Not-One-Size-Fits-All-How-to-Size-Kubernetes-Clusters_Guang-Ya-Liu-_-Sahdev-Zala.pdf).

The effects of large numbers of worker nodes can be alleviated by using more performant master nodes.

That's what's done in practice — here are the [master node sizes](https://kubernetes.io/docs/setup/best-practices/cluster-large/#size-of-master-and-master-components) used by `kube-up` on cloud infrastructure:

- Google Cloud Platform
  - 5 worker nodes → `n1-standard-1` master nodes
  - 500 worker nodes → `n1-standard-32` master nodes
- Amazon Web Services
  - 5 worker nodes → `m3.medium` master nodes
  - 500 worker nodes → `c4.8xlarge` master nodes

As you can see, for 500 worker nodes, the used master nodes have 32 and 36 CPU cores and 120 GB and 60 GB of memory, respectively.

*These are pretty large machines!*

So, if you intend to use a large number of small nodes, there are two things you need to keep in mind:

- The more worker nodes you have, the more performant master nodes you need
- If you plan to use more than 500 nodes, you can expect to hit some performance bottlenecks that require some effort to solve

> New developments like the [Virtual Kubelet](https://www.youtube.com/watch?v=v9cwYvuzROs) allow to bypass these limitations and allow for clusters with huge numbers of worker nodes.



#### 2. More system overhead

Kubernetes runs a set of system daemons on every worker node — these include the container runtime (e.g. Docker), kube-proxy, and the kubelet including cAdvisor.

> cAdvisor is incorporated in the kubelet binary.

All of these daemons together consume a fixed amount of resources.

If you use many small nodes, then the portion of resources used by these system components is bigger.

For example, imagine that all system daemons of a single node together use 0.1 CPU cores and 0.1 GB of memory.

If you have a single node of 10 CPU cores and 10 GB of memory, then the daemons consume 1% of your cluster's capacity.

On the other hand, if you have 10 nodes of 1 CPU core and 1 GB of memory, then the daemons consume 10% of your cluster's capacity.

Thus, in the second case, 10% of your bill is for running the system, whereas in the first case, it's only 1%.

So, if you want to maximise the return on your infrastructure spendings, then you might prefer fewer nodes.



#### 3. Lower resource utilisation

If you use smaller nodes, then you might end up with a larger number of resource fragments that are too small to be assigned to any workload and thus remain unused.

For example, assume that all your pods require 0.75 GB of memory.

If you have 10 nodes with 1 GB memory, then you can run 10 of these pods — and you end up with a chunk of 0.25 GB memory on each node that you can't use anymore.

That means, 25% of the total memory of your cluster is wasted.

On the other hand, if you use a single node with 10 GB of memory, then you can run 13 of these pods — and you end up only with a single chunk of 0.25 GB that you can't use.

In this case, you waste only 2.5% of your memory.

So, if you want to minimise resource waste, using larger nodes might provide better results.



#### 4. Pod limits on small nodes

*On some cloud infrastructure, the maximum number of pods allowed on small nodes is more restricted than you might expect.*

This is the case on [Amazon Elastic Kubernetes Service (EKS)](https://aws.amazon.com/eks/) where the [maximum number of pods per node](https://github.com/awslabs/amazon-eks-ami/blob/master/files/eni-max-pods.txt) depends on the instance type.

For example, for a `t2.medium` instance, the maximum number of pods is 17, for `t2.small` it's 11, and for `t2.micro` it's 4.

*These are very small numbers!*

Any pods that exceed these limits, fail to be scheduled by the Kubernetes scheduler and remain in the *Pending* state indefinitely.

If you are not aware of these limits, this can lead to hard-to-find bugs.

Thus, if you plan to use small nodes on Amazon EKS, [check the corresponding pods-per-node limits](https://github.com/awslabs/amazon-eks-ami/blob/master/files/eni-max-pods.txt) and count twice whether the nodes can accommodate all your pods.



### Conclusion

*So, should you use few large nodes or many small nodes in your cluster?*

As always, there is no definite answer.

The type of applications that you want to deploy to the cluster may guide your decision.

For example, if your application requires 10 GB of memory, you probably shouldn't use small nodes — the nodes in your cluster should have at least 10 GB of memory.

Or if your application requires 10-fold replication for high-availability, then you probably shouldn't use just 2 nodes — your cluster should have at least 10 nodes.

For all the scenarios in-between it depends on your specific requirements.

*Which of the above pros and cons are relevant for you? Which are not?*

That being said, there is no rule that all your nodes must have the same size.

Nothing stops you from using a mix of different node sizes in your cluster.

The worker nodes of a Kubernetes cluster can be totally heterogeneous.

This might allow you to trade off the pros and cons of both approaches.

*In the end, the proof of the pudding is in the eating — the best way to go is to experiment and find the combination that works best for you!*









## Graceful shutdown and zero downtime deployments in Kubernetes



**TL;DR:** *In this article, you will learn how to prevent broken connections when a Pod starts up or shuts down. You will also learn how to shut down long-running tasks gracefully.*

![Graceful shutdown and zero downtime deployments in Kubernetes](D:\学习资料\笔记\k8s\k8s图\55d21503055aaf9ef8a04d5e595ed505.png)

> You can [download this handy diagram as a PDF here](https://learnk8s.io/a/graceful-shutdown-and-zero-downtime-deployments-in-kubernetes/graceful-shutdown.pdf).

In Kubernetes, creating and deleting Pods is one of the most common tasks.

Pods are created when you execute a rolling update, scale deployments, for every new release, for every job and cron job, etc.

But Pods are also deleted and recreated after evictions — when you mark a node as unschedulable for example.

*If the nature of those Pods is so ephemeral, what happens when a Pod is in the middle of responding to a request but it's told to shut down?*

*Is the request completed before shutdown?*

*What about subsequent requests, are those redirected somewhere else?*

Before discussing what happens when a Pod is deleted, it's necessary to talk to about what happens when a Pod is created.

Let's assume you want to create the following Pod in your cluster:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
    - name: web
      image: nginx
      ports:
        - name: web
          containerPort: 80
```

You can submit the YAML definition to the cluster with:

```sh
$ kubectl apply -f pod.yaml
```

As soon as you enter the command, kubectl submits the Pod definition to the Kubernetes API.

*This is where the journey begins.*



### Saving the state of the cluster in the database

The Pod definition is received and inspected by the API and subsequently stored in the database — etcd.

The Pod is also added to [the Scheduler's queue.](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/#scheduling-cycle-binding-cycle)

The Scheduler:

1. inspects the definition
2. collects details about the workload such as CPU and memory requests and then
3. decides which Node is best suited to run it [(through a process called Filters and Predicates).](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/#extension-points)

At the end of the process:

- The Pod is marked as *Scheduled* in etcd.
- The Pod has a Node assigned to it.
- The state of the Pod is stored in etcd.

**But the Pod still does not exist.**

- When you submit a Pod with `kubectl apply -f` the YAML is sent to the Kubernetes API

![When you submit a Pod with kubectl apply -f the YAML is sent to the Kubernetes API.](D:\学习资料\笔记\k8s\k8s图\1225bea0e28347d94d3d10aa3947c9d1.svg)

- The API saves the Pod in the database — etcd.

![The API saves the Pod in the database — etcd.](D:\学习资料\笔记\k8s\k8s图\5808dba3226281703590f748c66331b5.svg)

- The scheduler assigns the best node for that Pod and the Pod's status changes to *Pending*. The Pod exists only in etcd.

![The scheduler assigns the best node for that Pod and the Pod's status changes to Pending. The Pod exists only in etcd.](D:\学习资料\笔记\k8s\k8s图\c14babd1b8a071673dfb2d916e6f02ec.svg)



The previous tasks happened in the control plane, and the state is stored in the database.

*So who is creating the Pod in your Nodes?*



### The kubelet — the Kubernetes agent

**It's the kubelet's job to poll the control plane for updates.**

You can imagine the kubelet relentlessly asking to the master node: *"I look after the worker Node 1, is there any new Pod for me?".*

When there is a Pod, the kubelet creates it.

*Sort of.*

The kubelet doesn't create the Pod by itself. Instead, it delegates the work to three other components:

1. **The Container Runtime Interface (CRI)** — the component that creates the containers for the Pod.
2. **The Container Network Interface (CNI)** — the component that connects the containers to the cluster network and assigns IP addresses.
3. **The Container Storage Interface (CSI)** — the component that mounts volumes in your containers.

In most cases, the Container Runtime Interface (CRI) is doing a similar job to:

```sh
$ docker run -d <my-container-image>
```

The Container Networking Interface (CNI) is a bit more interesting because it is in charge of:

1. Generating a valid IP address for the Pod.
2. Connecting the container to the rest of the network.

As you can imagine, there are several ways to connect the container to the network and assign a valid IP address (you could choose between IPv4 or IPv6 or maybe assign multiple IP addresses).

As an example, [Docker creates virtual ethernet pairs and attaches it to a bridge](https://archive.shivam.dev/docker-networking-explained/), whereas [the AWS-CNI connects the Pods directly to the rest of the Virtual Private Cloud (VPC).](https://itnext.io/kubernetes-is-hard-why-eks-makes-it-easier-for-network-and-security-architects-ea6d8b2ca965)

When the Container Network Interface finishes its job, the Pod is connected to the rest of the network and has a valid IP address assigned.

*There's only one issue.*

**The kubelet knows about the IP address (because it invoked the Container Network Interface), but the control plane does not.**

No one told the master node that the Pod has an IP address assigned and it's ready to receive traffic.

As far the control plane is concerned, the Pod is still being created.

It's the job of **the kubelet to collect all the details of the Pod such as the IP address and report them back to the control plane.**

You can imagine that inspecting etcd would reveal not just where the Pod is running, but also its IP address.

- The Kubelet polls the control plane for updates.

![The Kubelet polls the control plane for updates.](D:\学习资料\笔记\k8s\k8s图\07419b4c3e26617ba358a774f424d4ac.svg)

- When a new Pod is assigned to its Node, the kubelet retrieves the details.

![When a new Pod is assigned to its Node, the kubelet retrieves the details.](D:\学习资料\笔记\k8s\k8s图\916894bb5c0b53a2f6f92ffba2470c6e.svg)

- The kubelet doesn't create the Pod itself. It relies on three components: the Container Runtime Interface, Container Network Interface and Container Storage Interface.

![The kubelet doesn't create the Pod itself. It relies on three components: the Container Runtime Interface, Container Network Interface and Container Storage Interface.](D:\学习资料\笔记\k8s\k8s图\4c0ce00df1327e51a046d2f1176b0046.svg)

- Once all three components have successfully completed, the Pod is *Running* in your Node and has an IP address assigned.

![Once all three components have successfully completed, the Pod is Running in your Node and has an IP address assigned.](D:\学习资料\笔记\k8s\k8s图\0c2a6abab68960b2c608d509ca09c81e.svg)

- The kubelet reports the IP address back to the control plane.

![The kubelet reports the IP address back to the control plane.](D:\学习资料\笔记\k8s\k8s图\5e3833138a7052fd3dc7543795afd0c7.svg)

If the Pod isn't part of any Service, this is the end of the journey.

The Pod is created and ready to use.

*When the Pod is part of the Service, there are a few more steps needed.*



### Pods and Services

When you create a Service, there are usually two pieces of information that you should pay attention to:

1. The selector, which is used to specify the Pods that will receive the traffic.
2. The `targetPort` — the port used by the Pods to receive traffic.

A typical YAML definition for the Service looks like this:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  ports:
  - port: 80
    targetPort: 3000
  selector:
    name: app
```

When you submit the Service to the cluster with `kubectl apply`, Kubernetes finds all the Pods that have the same label as the selector (`name: app`) and collects their IP addresses — but only if they passed the [Readiness probe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-a-tcp-liveness-probe).

Then, for each IP address, it concatenates the IP address and the port.

If the IP address is `10.0.0.3` and the `targetPort` is `3000`, Kubernetes concatenates the two values and calls them an endpoint.

```
IP address + port = endpoint
---------------------------------
10.0.0.3   + 3000 = 10.0.0.3:3000
```

The endpoints are stored in etcd in another object called Endpoint.

*Confused?*

Kubernetes refers to:

- endpoint (in this article and the Learnk8s material this is referred to as a lowercase `e` endpoint) is the IP address + port pair (`10.0.0.3:3000`).
- Endpoint (in this article and the Learnk8s material this is referred to as an uppercase `E` Endpoint) is a collection of endpoints.

The Endpoint object is a real object in Kubernetes and for every Service Kubernetes automatically creates an Endpoint object.

You can verify that with:

```sh
$ kubectl get services,endpoints
NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)
service/my-service-1   ClusterIP   10.105.17.65   <none>        80/TCP
service/my-service-2   ClusterIP   10.96.0.1      <none>        443/TCP

NAME                     ENDPOINTS
endpoints/my-service-1   172.17.0.6:80,172.17.0.7:80
endpoints/my-service-2   192.168.99.100:8443
```

The Endpoint collects all the IP addresses and ports from the Pods.

*But not just one time.*

The Endpoint object is refreshed with a new list of endpoints when:

1. A Pod is created.
2. A Pod is deleted.
3. A label is modified on the Pod.

So you can imagine that every time you create a Pod and after the kubelet posts its IP address to the master Node, Kubernetes updates all the endpoints to reflect the change:

```sh
$ kubectl get services,endpoints
NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)
service/my-service-1   ClusterIP   10.105.17.65   <none>        80/TCP
service/my-service-2   ClusterIP   10.96.0.1      <none>        443/TCP

NAME                     ENDPOINTS
endpoints/my-service-1   172.17.0.6:80,172.17.0.7:80,172.17.0.8:80
endpoints/my-service-2   192.168.99.100:8443
```

Great, the endpoint is stored in the control plane, and the Endpoint object was updated.

- In this picture, there's a single Pod deployed in your cluster. The Pod belongs to a Service. If you were to inspect etcd, you would find the Pod's details as well as Service.

![In this picture, there's a single Pod deployed in your cluster. The Pod belongs to a Service. If you were to inspect etcd, you would find the Pod's details as well as Service.](D:\学习资料\笔记\k8s\k8s图\b884492311e14caabab8e77a4ad15736.svg)

- *What happens when a new Pod is deployed?*

![What happens when a new Pod is deployed?](D:\学习资料\笔记\k8s\k8s图\45a13740f9904346ddd3ccecc3095eb6.svg)

- Kubernetes has to keep track of the Pod and its IP address. The Service should route traffic to the new endpoint, so the IP address and port should be propagated.

![Kubernetes has to keep track of the Pod and its IP address. The Service should route traffic to the new endpoint, so the IP address and port should be propagated.](D:\学习资料\笔记\k8s\k8s图\cebaf2f0aae8170d579a8955755d4922.svg)

- What happens when *another* Pod is deployed?

![What happens when another Pod is deployed?](D:\学习资料\笔记\k8s\k8s图\9c8cd44a83e0d91991e6f14f9d06e5a4.svg)

- The exact same process. A new "row" for the Pod is created in the database, and the endpoint is propagated.

![The exact same process. A new "row" for the Pod is created in the database, and the endpoint is propagated.](D:\学习资料\笔记\k8s\k8s图\68afeba680a87d867c38dff9ed4ab23d.svg)

- *What happens when a Pod is deleted, though?*

![What happens when a Pod is deleted, though?](D:\学习资料\笔记\k8s\k8s图\737e8fabd7b3b0800a2566c7eafb4e61.svg)

- The Service immediately removes the endpoint, and, eventually, the Pod is removed from the database too.

![The Service immediately removes the endpoint, and, eventually, the Pod is removed from the database too.](D:\学习资料\笔记\k8s\k8s图\736f5a1722a2c74cc8a332d028ede4f3.svg)

- Kubernetes reacts to every small change in your cluster.

![Kubernetes reacts to every small change in your cluster.](D:\学习资料\笔记\k8s\k8s图\7fc2c39cce7044de1fef171158774225.svg)

*Are you ready to start using your Pod?*

**There's more.**

A lot more!



### Consuming endpoints in Kubernetes

**Endpoints are used by several components in Kubernetes.**

Kube-proxy uses the endpoints to set up iptables rules on the Nodes.

So every time there is a change to an Endpoint (the object), kube-proxy retrieves the new list of IP addresses and ports and write the new iptables rules.

- Let's consider this three-node cluster with two Pods and no Services. The state of the Pods is stored in etcd.

![Let's consider this three-node cluster with two Pods and no Services. The state of the Pods is stored in etcd.](D:\学习资料\笔记\k8s\k8s图\424be4bf0bb5a98a8c8eca9d290bec15.svg)

- *What happens when you create a Service?*

![What happens when you create a Service?](D:\学习资料\笔记\k8s\k8s图\5983a21ca12d1149170d5da23f8a290a.svg)

- Kubernetes created an Endpoint object and collects all the endpoints (IP address and port pairs) from the Pods.

![Kubernetes created an Endpoint object and collects all the endpoints (IP address and port pairs) from the Pods.](D:\学习资料\笔记\k8s\k8s图\73eb9fd5886132caa0bd3a8cc8c32511.svg)

- Kube-proxy daemon is subscribed to changes to Endpoints.

![Kube-proxy daemon is subscribed to changes to Endpoints.](D:\学习资料\笔记\k8s\k8s图\13f70aca3561e5b3ddd125eb92ec58ae.svg)

- When an Endpoint is added, removed or updated, kube-proxy retrieves the new list of endpoints.

![When an Endpoint is added, removed or updated, kube-proxy retrieves the new list of endpoints.](D:\学习资料\笔记\k8s\k8s图\5dbc43d11f4126c918730a7a07f7a69e.svg)

- Kube-proxy uses the endpoints to creating iptables rules on each Node of your cluster.

![Kube-proxy uses the endpoints to creating iptables rules on each Node of your cluster.](D:\学习资料\笔记\k8s\k8s图\57c616735f28c9ad5ce8cdd6716bfd05.svg)

The Ingress controller uses the same list of endpoints.

The Ingress controller is that component in the cluster that routes external traffic into the cluster.

When you set up an Ingress manifest you usually specify the Service as the destination:

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - http:
      paths:
      - backend:
          serviceName: my-service
          servicePort: 80
        path: /
```

*In reality, the traffic is not routed to the Service.*

Instead, the Ingress controller sets up a subscription to be notified every time the endpoints for that Service change.

**The Ingress routes the traffic directly to the Pods skipping the Service.**

As you can imagine, every time there is a change to an Endpoint (the object), the Ingress retrieves the new list of IP addresses and ports and reconfigures the controller to include the new Pods.

- In this picture, there's an Ingress controller with a Deployment with two replicas and a Service.

![In this picture, there's an Ingress controller with a Deployment with two replicas and a Service.](D:\学习资料\笔记\k8s\k8s图\f2947f298293c6e7ad2dadcb659ec4e6.svg)



- If you want to route external traffic to the Pods through the Ingress, you should create an Ingress manifest (a YAML file).

![If you want to route external traffic to the Pods through the Ingress, you should create an Ingress manifest (a YAML file).](D:\学习资料\笔记\k8s\k8s图\7375a6b537808b06a64bd131e9b67a71.svg)

- As soon as you `kubectl apply -f ingress.yaml`, the Ingress controller retrieves the file from the control plane.

![As soon as you kubectl apply -f ingress.yaml, the Ingress controller retrieves the file from the control plane.](D:\学习资料\笔记\k8s\k8s图\5fe59ddd01d020fdf4a0734c4b1157b6.svg)

- The Ingress YAML has a `serviceName` property that describes which Service it should use.

![The Ingress YAML has a serviceName property that describes which Service it should use.](D:\学习资料\笔记\k8s\k8s图\a71a294e027bc013466591d9637f051c.svg)

- The Ingress controller retrieves the list of endpoints from the Service and skips it. The traffic flows directly to the endpoints (Pods).

![The Ingress controller retrieves the list of endpoints from the Service and skips it. The traffic flows directly to the endpoints (Pods).](D:\学习资料\笔记\k8s\k8s图\a402b8795d18ca28bb8355e3085fdee0.svg)

- *What happens when a new Pod is created?*

![What happens when a new Pod is created?](D:\学习资料\笔记\k8s\k8s图\c6aeb8c7a44e0c43b730afb1536fd455.svg)

- You know already how Kubernetes creates the Pod and propagates the endpoint.

![You know already how Kubernetes creates the Pod and propagates the endpoint.](D:\学习资料\笔记\k8s\k8s图\aae508483bd909f1e9d2fe1cf34c739e.svg)

- The Ingress controller is subscribing to changes to the endpoints. Since there's an incoming change, it retrieves the new list of endpoints.

![The Ingress controller is subscribing to changes to the endpoints. Since there's an incoming change, it retrieves the new list of endpoints.](D:\学习资料\笔记\k8s\k8s图\5fd347b4acb5d36e7a20b69b63a11804.svg)

- The Ingress controller routes the traffic to the new Pod.

![The Ingress controller routes the traffic to the new Pod.](D:\学习资料\笔记\k8s\k8s图\c00976d70513af74446a2b35870c3524.svg)

There are more examples of Kubernetes components that subscribe to changes to endpoints.

CoreDNS, the DNS component in the cluster, is another example.

If you use [Services of type Headless](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services), CoreDNS will have to subscribe to changes to the endpoints and reconfigure itself every time an endpoint is added or removed.

The same endpoints are consumed by service meshes such as Istio or Linkerd, [by cloud providers to create Services of `type:LoadBalancer`](https://thebsdbox.co.uk/2020/03/18/Creating-a-Kubernetes-cloud-doesn-t-required-boiling-the-ocean/) and countless operators.

You must remember that several components subscribe to change to endpoints and they might receive notifications about endpoint updates at different times.

*Is it enough, or is there something happening after you create a Pod?*

**This time you're done!**

A quick recap on what happens when you create a Pod:

1. The Pod is stored in etcd.
2. The scheduler assigns a Node. It writes the node in etcd.
3. The kubelet is notified of a new and scheduled Pod.
4. The kubelet delegates creating the container to the Container Runtime Interface (CRI).
5. The kubelet delegates attaching the container to the Container Network Interface (CNI).
6. The kubelet delegates mounting volumes in the container to the Container Storage Interface (CSI).
7. The Container Network Interface assigns an IP address.
8. The kubelet reports the IP address to the control plane.
9. The IP address is stored in etcd.

And if your Pod belongs to a Service:

1. The kubelet waits for a successful Readiness probe.
2. All relevant Endpoints (objects) are notified of the change.
3. The Endpoints add a new endpoint (IP address + port pair) to their list.
4. Kube-proxy is notified of the Endpoint change. Kube-proxy updates the iptables rules on every node.
5. The Ingress controller is notified of the Endpoint change. The controller routes traffic to the new IP addresses.
6. CoreDNS is notified of the Endpoint change. If the Service is of type Headless, the DNS entry is updated.
7. The cloud provider is notified of the Endpoint change. If the Service is of `type: LoadBalancer`, the new endpoint are configured as part of the load balancer pool.
8. Any service mesh installed in the cluster is notified of the Endpoint change.
9. Any other operator subscribed to Endpoint changes is notified too.

*Such a long list for what is surprisingly a common task — creating a Pod.*

The Pod is *Running*. It is time to discuss what happens when you delete it.



### Deleting a Pod

You might have guessed it already, but when the Pod is deleted, you have to follow the same steps but in reverse.

First, the endpoint should be removed from the Endpoint (object).

This time the Readiness probe is ignored, and the endpoint is removed immediately from the control plane.

That, in turn, fires off all the events to kube-proxy, Ingress controller, DNS, service mesh, etc.

Those components will update their internal state and stop routing traffic to the IP address.

Since the components might be busy doing something else, **there is no guarantee on how long it will take to remove the IP address from their internal state.**

For some, it could take less than a second; for others, it could take more.

- If you're deleting a Pod with `kubectl delete pod`, the command reaches the Kubernetes API first.

![If you're deleting a Pod with kubectl delete pod, the command reaches the Kubernetes API first.](D:\学习资料\笔记\k8s\k8s图\52bfaf15e838a12cec28ed656479d515.svg)

- The message is intercepted by a specific controller in the control plane: the Endpoint controller.

![The message is intercepted by a specific controller in the control plane: the Endpoint controller.](D:\学习资料\笔记\k8s\k8s图\e7e784358791597480b1e90efa5a180c.svg)

- The Endpoint controller issue a command to the API to remove the IP address and port from the Endpoint object.

![The Endpoint controller issue a command to the API to remove the IP address and port from the Endpoint object.](D:\学习资料\笔记\k8s\k8s图\ee1385cf9cdf974d741733be04f7d5ab.svg)

- *Who listens for Endpoint changes?* Kube-proxy, the Ingress controller, CoreDNS, etc. are notified of the change.

![Who listens for Endpoint changes? Kube-proxy, the Ingress controller, CoreDNS, etc. are notified of the change.](D:\学习资料\笔记\k8s\k8s图\ebc4293a1a6713162af536f40ecfac6b.svg)

- A few components such as kube-proxy might need some extra time to propagate the changes further.

![A few components such as kube-proxy might need some extra time to propagate the changes further.](D:\学习资料\笔记\k8s\k8s图\d48e4005dc26411e5aaf9fd58dbf0f00.svg)

At the same time, the status of the Pod in etcd is changed to *Terminating*.

The kubelet is notified of the change and delegates:

1. Unmounting any volumes from the container to the Container Storage Interface (CSI).
2. Detaching the container from the network and releasing the IP address to the Container Network Interface (CNI).
3. Destroying the container to the Container Runtime Interface (CRI).

In other words, Kubernetes follows precisely the same steps to create a Pod but in reverse.

- If you're deleting a Pod with `kubectl delete pod`, the command reaches the Kubernetes API first.

![If you're deleting a Pod with kubectl delete pod, the command reaches the Kubernetes API first.](D:\学习资料\笔记\k8s\k8s图\6b4c471a91b57589260f8be6eccf5c3d.svg)

- When the kubelet polls the control plane for updates, it notices that the Pod was deleted.

![When the kubelet polls the control plane for updates, it notices that the Pod was deleted.](D:\学习资料\笔记\k8s\k8s图\88ff9f8f44b0530226430da3ca8c892e.svg)

- The kubelet delegates destroying the Pod to the Container Runtime Interface, Container Network Interface and Container Storage Interface.

![The kubelet delegates destroying the Pod to the Container Runtime Interface, Container Network Interface and Container Storage Interface.](D:\学习资料\笔记\k8s\k8s图\a0d938dcf2644aecbde29116145df766.svg)

However, there is a subtle but essential difference.

**When you terminate a Pod, removing the endpoint and the signal to the kubelet are issued at the same time.**

When you create a Pod for the first time, Kubernetes waits for the kubelet to report the IP address and then kicks off the endpoint propagation.

**However, when you delete a Pod, the events start in parallel.**

And this could cause quite a few race conditions.

*What if the Pod is deleted before the endpoint is propagated?*

- Deleting the endpoint and deleting the Pod happen at the same time.

![Deleting the endpoint and deleting the Pod happen at the same time.](D:\学习资料\笔记\k8s\k8s图\4e92253a939a7e8bb97c6eb37f7a3d7e.svg)



- So you could end up deleting the endpoint before kube-proxy updates the iptables rules.

![So you could end up deleting the endpoint before kube-proxy updates the iptables rules.](D:\学习资料\笔记\k8s\k8s图\900616530908d5f4d5e883cc5812404e.svg)



- Or you might luckier, and the Pod is deleted only after the endpoints are fully propagated.

![Or you might luckier, and the Pod is deleted only after the endpoints are fully propagated.](D:\学习资料\笔记\k8s\k8s图\ac8dd428b56f684d16dd922625f52bd6.svg)



### Graceful shutdown

When a Pod is terminated before the endpoint is removed from kube-proxy or the Ingress controller, you might experience downtime.

And, if you think about it, it makes sense.

Kubernetes is still routing traffic to the IP address, but the Pod is no longer there.

The Ingress controller, kube-proxy, CoreDNS, etc. didn't have enough time to remove the IP address from their internal state.

Ideally, Kubernetes should wait for all components in the cluster to have an updated list of endpoints before the Pod is deleted.

*But Kubernetes doesn't work like that.*

Kubernetes offers robust primitives to distribute the endpoints (i.e. the Endpoint object and more advanced abstractions such as [Endpoint Slices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)).

However, Kubernetes does not verify that the components that subscribe to endpoints changes are up-to-date with the state of the cluster.

*So what can you do avoid this race conditions and make sure that the Pod is deleted after the endpoint is propagated?*

**You should wait.**

**When the Pod is about to be deleted, it receives a SIGTERM signal.**

Your application can capture that signal and start shutting down.

Since it's unlikely that the endpoint is immediately deleted from all components in Kubernetes, you could:

1. Wait a bit longer before exiting.
2. Still process incoming traffic, despite the SIGTERM.
3. Finally, close existing long-lived connections (perhaps a database connection or WebSockets).
4. Close the process.

*How long should you wait?*

**By default, Kubernetes will send the SIGTERM signal and wait 30 seconds before force killing the process.**

So you could use the first 15 seconds to continue operating as nothing happened.

Hopefully, the interval should be enough to propagate the endpoint removal to kube-proxy, Ingress controller, CoreDNS, etc.

And, as a consequence, less and less traffic will reach your Pod until it stops.

After the 15 seconds, it's safe to close your connection with the database (or any persistent connections) and terminate the process.

If you think you need more time, you can stop the process at 20 or 25 seconds.

However, you should remember that Kubernetes will forcefully kill the process after 30 seconds [(unless you change the `terminationGracePeriodSeconds` in your Pod definition).](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/#hook-handler-execution)

*What if you can't change the code to wait longer?*

You could invoke a script to wait for a fixed amount of time and then let the app exit.

Before the SIGTERM is invoked, Kubernetes exposes a `preStop` hook in the Pod.

You could set the `preStop` to hook to wait for 15 seconds.

Let's have a look at an example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
    - name: web
      image: nginx
      ports:
        - name: web
          containerPort: 80
      lifecycle:
        preStop:
          exec:
            command: ["sleep", "15"]
```

The `preStop` hook is one of the [Pod LifeCycle hooks](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/).

*Is a 15 seconds delay the recommended amount?*

It depends, but it could be a sensible way to start testing.

Here's a recap of what options you have:

- You already know that, when a Pod is deleted, the kubelet is notified of the change.

![You already know that, when a Pod is deleted, the kubelet is notified of the change.](D:\学习资料\笔记\k8s\k8s图\2ddb2f96d586a7eb508f652728535922.svg)

- If the Pod has a `preStop` hook, it is invoked first.

![If the Pod has a preStop hook, it is invoked first.](D:\学习资料\笔记\k8s\k8s图\50e256378e2a73a86a43eae7c1cf143d.svg)

- When the `preStop` completes, the kubelet sends the SIGTERM signal to the container. From that point, the container should close all long-lived connections and prepare to terminate.

![When the preStop completes, the kubelet sends the SIGTERM signal to the container. From that point, the container should close all long-lived connections and prepare to terminate.](D:\学习资料\笔记\k8s\k8s图\c537202f15000748135f98fdfcd6c31f.svg)

- By default, the process has 30 seconds to exit, and that includes the `preStop` hook. If the process isn't exited by then, the kubelet sends the SIGKILL signal and force killing the process.

![By default, the process has 30 seconds to exit, and that includes the preStop hook. If the process isn't exited by then, the kubelet sends the SIGKILL signal and force killing the process.](D:\学习资料\笔记\k8s\k8s图\f7985c9e68bd8cf65dca13edefd96f81.svg)

- The kubelet notifies the control plane that the Pod was deleted successfully.

![The kubelet notifies the control plane that the Pod was deleted successfully.](D:\学习资料\笔记\k8s\k8s图\2281bf7553ed051cf4ead780661c6b42.svg)

### Grace periods and rolling updates

Graceful shutdown applies to Pods being deleted.

*But what if you don't delete the Pods?*

Even if you don't, Kubernetes deletes Pods all the times.

In particular, Kubernetes creates and deletes Pods every time you deploy a newer version of your application.

When you change the image in your Deployment, Kubernetes rolls out the change incrementally.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 3
  selector:
    matchLabels:
      name: app
  template:
    metadata:
      labels:
        name: app
    spec:
      containers:
      - name: app
        # image: nginx:1.18 OLD
        image: nginx:1.19
        ports:
          - containerPort: 3000
```

If you have three replicas and as soon as you submit the new YAML resources Kubernetes:

- Creates a Pod with the new container image.
- Destroys an existing Pod.
- Waits for the Pod to be ready.

And it repeats the steps above until all the Pods are migrated to the newer version.

Kubernetes repeats each cycle only after the new Pod is ready to receive traffic (in other words, it passes the Readiness check).

*Does Kubernetes wait for the Pod to be deleted before moving to the next one?*

**No.**

If you have 10 Pods and the Pod takes 2 seconds to be ready and 20 to shut down this is what happens:

1. The first Pod is created, and a previous Pod is terminated.
2. The new Pod takes 2 seconds to be ready after that Kubernetes creates a new one.
3. In the meantime, the Pod being terminated stays terminating for 20 seconds

After 20 seconds, all new Pods are live (10 Pods, *Ready* after 2 seconds) and all 10 the previous Pods are terminating (the first *Terminated* Pod is about to exit).

In total, you have double the amount of Pods for a short amount of time (10 *Running*, 10 *Terminating*).

![Rolling update and graceful shutdown](D:\学习资料\笔记\k8s\k8s图\ed8e17bf72f5d5424d1f58c9602e2e81.svg)

The longer the graceful period compared to the Readiness probe, the more Pods you will have *Running* (and *Terminating*) at the same time.

*Is it bad?*

Not necessarily since you're careful not dropping connections.



### Terminating long-running tasks

*And what about long-running jobs?*

*If you are transcoding a large video, is there any way to delay stopping the Pod?*

Imagine you have a Deployment with three replicas.

Each replica is assigned a video to transcode, and the task could take several hours to complete.

When you trigger a rolling update, the Pod has 30 seconds to complete the task before it's killed.

*How can you avoid delaying shutting down the Pod?*

You could increase the `terminationGracePeriodSeconds` to a couple of hours.

**However, the endpoint of the Pod is unreachable at that point.**

![Unreachable Pod](D:\学习资料\笔记\k8s\k8s图\3ef3d002d72dfad05f538ea00f5cbd01.svg)

If you expose metrics to monitor your Pod, your instrumentation won't be able to reach your Pod.

*Why?*

**Tools such as Prometheus rely on Endpoints to scrape Pods in your cluster.**

However, as soon as you delete the Pod, the endpoint deletion is propagated in the cluster — even to Prometheus!

**Instead of increasing the grace period, you should consider creating a new Deployment for every new release.**

When you create a brand new Deployment, the existing Deployment is left untouched.

The long-running jobs can continue processing the video as usual.

Once they are done, you can delete them manually.

If you wish to delete them automatically, you might want to set up an autoscaler that can scale your deployment to zero replicas when they run out of tasks.

An example of such Pod autoscaler is [Osiris — a general-purpose, scale-to-zero component for Kubernetes.](https://github.com/deislabs/osiris)

The technique is sometimes referred to as **Rainbow Deployments** and is useful every time you have to keep the previous Pods *Running* for longer than the grace period.

*Another excellent example is WebSockets.*

If you are streaming real-time updates to your users, you might not want to terminate the WebSockets every time there is a release.

If you are frequently releasing during the day, that could lead to several interruptions to real-time feeds.

**Creating a new Deployment for every release is a less obvious but better choice.**

Existing users can continue streaming updates while the most recent Deployment serves the new users.

As a user disconnects from old Pods, you can gradually decrease the replicas and retire past Deployments.



### Summary

You should pay attention to Pods being deleted from your cluster since their IP address might be still used to route traffic.

Instead of immediately shutting down your Pods, you should consider waiting a little bit longer in your application or set up a `preStop` hook.

The Pod should be removed only after all the endpoints in the cluster are propagated and removed from kube-proxy, Ingress controllers, CoreDNS, etc.

If your Pods run long-lived tasks such as transcoding videos or serving real-time updates with WebSockets, you should consider using rainbow deployments.

In rainbow deployments, you create a new Deployment for every release and delete the previous one when the connection (or the tasks) drained.

You can manually remove the older deployments as soon as the long-running task is completed.

Or you could automatically scale your deployment to zero replicas to automate the process.





## Extending applications on Kubernetes with multi-container pods



*TL;DR: In this article you will learn how you can use the ambassador, adapter, sidecar and init containers to extend yours apps in Kubernetes without changing their code.*

Kubernetes offers an immense amount of flexibility and the ability to run a wide variety of applications.

If your applications are cloud-native microservices or [12-factor apps](https://12factor.net/), chances are that running them in Kubernetes will be relatively straightforward.

*But what about running applications that weren't explicitly designed to be run in a containerized environment?*

Kubernetes can handle these as well, although it may be a bit more work to set up.

One of the most powerful tools that Kubernetes offers to help is the **multi-container pod** (although multi-container pods are also useful for cloud-native apps in a variety of cases, as you'll see).

*Why would you want to run multiple containers in a pod?*

**Multi-container pods allow you to change the behaviour of an application without changing its code.**

This can be useful in all sorts of situations, but it's convenient for applications that weren't originally designed to be run in containers.

Let's start with an example.



### Securing an HTTP service

Elasticsearch was designed before containers became popular (although it's pretty straightforward to run in Kubernetes nowadays) and can be seen as a stand-in for, say, a legacy Java application designed to run in a virtual machine.

*Let's use Elasticsearch as an example application that you'd like to enhance using multi-container pods.*

The following is a very basic (not at all production-ready) Elasticsearch Deployment and Service:

es-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: elasticsearch
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: elasticsearch
  template:
    metadata:
      labels:
        app.kubernetes.io/name: elasticsearch
    spec:
      containers:
        - name: elasticsearch
          image: elasticsearch:7.9.3
          env:
            - name: discovery.type
              value: single-node
          ports:
            - name: http
              containerPort: 9200
---
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch
spec:
  selector:
    app.kubernetes.io/name: elasticsearch
  ports:
    - port: 9200
      targetPort: 9200
```

> The `discovery.type` environment variable is necessary to get it running with a single replica.

Elasticsearch will listen on port 9200 over HTTP by default.

You can confirm that the pod works by running another pod in the cluster and `curl`ing to the `elasticsearch` service:

```sh
$ kubectl run -it --rm --image=curlimages/curl curl \
  -- curl http://elasticsearch:9200
{
  "name" : "elasticsearch-77d857c8cf-mk2dv",
  "cluster_name" : "docker-cluster",
  "cluster_uuid" : "z98oL-w-SLKJBhh5KVG4kg",
  "version" : {
    "number" : "7.9.3",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "c4138e51121ef06a6404866cddc601906fe5c868",
    "build_date" : "2020-10-16T10:36:16.141335Z",
    "build_snapshot" : false,
    "lucene_version" : "8.6.2",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

Now let's say that you're moving towards a [zero-trust security model](https://en.wikipedia.org/wiki/Zero_Trust_Networks) and you'd like to encrypt all traffic on the network.

*How would you go about this if the application doesn't have native TLS support?*

> Recent versions of Elasticsearch support TLS, but it was a paid extra feature for a long time.

Our first thought might be to do TLS termination with an [nginx ingress](https://kubernetes.github.io/ingress-nginx/), since the ingress is the component routing the external traffic in the cluster.

But that won't meet the requirements, since traffic between the ingress pod and the Elasticsearch pod could go over the network unencrypted.

- The external traffic is routed to the Ingress and then to Pods.

![The external traffic is routed to the Ingress and then to Pods.](D:\学习资料\笔记\k8s\k8s图\e447fb7257dea4279ccb9d624c5c176a.svg)

- If you terminate TLS at the ingress, the rest of the traffic is unencrypted.

![If you terminate TLS at the ingress, the rest of the traffic is unencrypted.](D:\学习资料\笔记\k8s\k8s图\4b0217c5505ad0e50edeee2708694a7b.svg)

A solution that *will* meet the requirements is to tack an nginx proxy container onto the pod that will listen over TLS.

The traffic flows encrypted all the the way from the user to the Pod.

- If you include a proxy container in the pod, you can terminate TLS in the Nginx pod.

![If you include a proxy container in the pod, you can terminate TLS in the Nginx pod.](D:\学习资料\笔记\k8s\k8s图\dd82c8011456b5f4f5822fc8af57d8d9.svg)

- When you compare the current setup, you can notice that the traffic is encrypted all the way until the Elasticsearch container.

![When you compare the current setup, you can notice that the traffic is encrypted all the way until the Elasticsearch container.](D:\学习资料\笔记\k8s\k8s图\a423cb1bf7b03ed57360fe266ea25615.svg)

Here's what the deployment might look like:

es-secure-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: elasticsearch
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: elasticsearch
  template:
    metadata:
      labels:
        app.kubernetes.io/name: elasticsearch
    spec:
      containers:
        - name: elasticsearch
          image: elasticsearch:7.9.3
          env:
            - name: discovery.type
              value: single-node
            - name: network.host
              value: 127.0.0.1
            - name: http.port
              value: '9201'
        - name: nginx-proxy
          image: nginx:1.19.5
          volumeMounts:
            - name: nginx-config
              mountPath: /etc/nginx/conf.d
              readOnly: true
            - name: certs
              mountPath: /certs
              readOnly: true
          ports:
            - name: https
              containerPort: 9200
      volumes:
        - name: nginx-config
          configMap:
            name: elasticsearch-nginx
        - name: certs
          secret:
            secretName: elasticsearch-tls
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: elasticsearch-nginx
data:
  elasticsearch.conf: |
    server {
        listen 9200 ssl;
        server_name elasticsearch;
        ssl_certificate /certs/tls.crt;
        ssl_certificate_key /certs/tls.key;

        location / {
            proxy_pass http://localhost:9201;
        }
    }
```

Let's unpack that a little bit:

- Elasticsearch is listening on `localhost` on port `9201` instead of the default `0.0.0.0:9200` (that's what the `network.host` and `http.port` environment variables are for).
- The new `nginx-proxy` container listens on port `9200` over HTTPS and proxies requests to Elasticsearch on port `9201`. (The `elasticsearch-tls` secret contains the TLS cert and key, which could be generated with [cert-manager](https://cert-manager.io/), for example.)

So requests from outside the pod will go to Nginx on port 9200 over HTTPS and then forwarded to Elasticsearch on port 920

![The request is proxies by Nginx on port 9220 and forwarded to port 9201 on Elastisearch](D:\学习资料\笔记\k8s\k8s图\808d9f2059e8c0c8e4e0cde84ed67d37.svg)

You can confirm it's working by making an HTTPS request from within the cluster.

```sh
$ kubectl run -it --rm --image=curlimages/curl curl \
  -- curl -k https://elasticsearch:9200
{
  "name" : "elasticsearch-5469857795-nddbn",
  "cluster_name" : "docker-cluster",
  "cluster_uuid" : "XPW9Z8XGTxa7snoUYzeqgg",
  "version" : {
    "number" : "7.9.3",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "c4138e51121ef06a6404866cddc601906fe5c868",
    "build_date" : "2020-10-16T10:36:16.141335Z",
    "build_snapshot" : false,
    "lucene_version" : "8.6.2",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

> The `-k` version is necessary for self-signed TLS certificates. In a production environment, you'd want to use a trusted certificate.

A quick look at the logs shows that the request went through the Nginx proxy:

bash

```sh
$ kubectl logs elasticsearch-5469857795-nddbn nginx-proxy | grep curl
10.88.4.127 - - [26/Nov/2020:02:37:07 +0000] "GET / HTTP/1.1" 200 559 "-" "curl/7.73.0-DEV" "-"
```

You can also check that you're unable to connect to Elasticsearch over unencrypted connections:

```sh
$ kubectl run -it --rm --image=curlimages/curl curl \
  -- curl http://elasticsearch:9200
<html>
<head><title>400 The plain HTTP request was sent to HTTPS port</title></head>
<body>
<center><h1>400 Bad Request</h1></center>
<center>The plain HTTP request was sent to HTTPS port</center>
<hr><center>nginx/1.19.5</center>
</body>
</html>
```

**You've enforced TLS without having to touch the Elasticsearch code or the container image!**



**Proxy containers are a common pattern**

The practice of adding a proxy container to a pod is common enough that it has a name: the **Ambassador Pattern**.

> All of the patterns in this post are described in detail in a [excellent paper](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/45406.pdf) from Google.

Adding basic TLS support is only the beginning.

Here are a few other things you can do with the Ambassador Pattern:

- If you want all the traffic in the cluster to be encrypted with TLS certificate, you might decide to install an nginx (or other) proxy in every pod in the cluster. You can even go a step farther and use [mutual TLS](https://medium.com/sitewards/the-magic-of-tls-x509-and-mutual-authentication-explained-b2162dec4401) to ensure that all requests are authenticated as well as encrypted. (This is the primary approach used service meshes such as [Istio](https://istio.io/) and [Linkerd](https://linkderd.io/).)
- You can use a proxy to ensure that a centralized OAuth authority authenticates all requests by verifying [jwt](https://jwt.io/)s. One example of this is [gcp-iap-auth](https://github.com/Roblox/gcp-iap-auth), which verifies that requests are authenticated by [GCP Identity-Aware Proxy](https://cloud.google.com/iap/).
- You can connect over a secure tunnel to an external database. This is especially handy for databases that don't have built-in TLS support (like older versions of Redis). Another example is the [Google Cloud SQL proxy](https://github.com/GoogleCloudPlatform/cloudsql-proxy).



### How do multi-container pods work?

Let's take a step back and tease apart the difference between pods and containers on Kubernetes to get a better picture of what's happening under the hood.

A "traditional" container (e.g. one started by `docker run`) provides several forms of isolation:

- Resource isolation (for example, memory limits).
- Process isolation.
- Filesystem and mount isolation.
- Network isolation.

> There are a few other things that Docker sets up, but those are the most significant.

The tools that are used under the hood are [Linux namespaces](https://en.wikipedia.org/wiki/Linux_namespaces) and [control groups (cgroups)](https://en.wikipedia.org/wiki/Cgroups).

**Control groups are a convenient way to limit resources such as CPU or memory that a particular process can use.**

As an example, you could say that your process should use only 2GB of memory and one of your four CPU cores.

**Namespaces, on the other hand, are in charge of isolating the process and limiting what it can see.**

As an example, the process can only see the network packets that are directly related to it.

It won't be able to see all of the network packets flowing through the network adapter.

Or you could isolate the filesystem and let the process believe that it has access to all of it.

- Since kernel version 5.6, there are eight kinds of namespaces and the mount namespace is one of them.

![Since kernel version 5.6, there are eight kinds of namespaces and the mount namespace is one of them.](D:\学习资料\笔记\k8s\k8s图\837a954167f1706b621c6db79a2d8f30.svg)

- With the mount namespace, you can let the process believe that it has access to all directories on the host when in fact it has not.

![With the mount namespace, you can let the process believe that it has access to all directories on the host when in fact it has not.](D:\学习资料\笔记\k8s\k8s图\cb49a19918ef745a4ebf0e55136aacef.svg)

- The mount namespace is designed to isolate resources — in this case, the filesystem.

![The mount namespace is designed to isolate resources — in this case, the filesystem.](D:\学习资料\笔记\k8s\k8s图\f9a389eb22faee5b3a4486584ca23586.svg)

- Each process can see the same file system, while still being isolated from the others.

![Each process can see the same file system, while still being isolated from the others.](D:\学习资料\笔记\k8s\k8s图\1484bdd9816d892f21bde5d3b5ddd43b.svg)

> If you need a refresher on cgroups and namespaces, [here's an excellent blog post diving into some of the technical details.](https://jvns.ca/blog/2016/10/10/what-even-is-a-container/)

On Kubernetes, a container provides all of those forms of isolation *except* network isolation.

Instead, **network isolation happens at the pod level**.

In other words, each container in a pod will have its filesystem, process table, etc., but all of them will share the same network namespace.

Let's play around with a straightforward multi-pod container to get a better idea of how it works.

pod-multiple-containers.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: podtest
spec:
  containers:
    - name: c1
      image: busybox
      command: ['sleep', '5000']
      volumeMounts:
        - name: shared
          mountPath: /shared
    - name: c2
      image: busybox
      command: ['sleep', '5000']
      volumeMounts:
        - name: shared
          mountPath: /shared
  volumes:
    - name: shared
      emptyDir: {}
```

Breaking that down a bit:

- There are two containers, both of which just `sleep` for a while.
- There is an [`emptyDir` volume](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir), which is essentially a temporary local volume that lasts for the lifetime of the pod.
- The `emptyDir` volume is mounted in each pod at the `/shared` directory.

You can see that the volume is mounted on the first container by using `kubectl exec`:

```sh
$ kubectl exec -it podtest --container c1 -- sh
```

The command attached a terminal session to the container `c1` in the `podtest` pod.

> The `--container` option for `kubectl exec` is often abbreviated `-c`.

You can inspect the volumes attached to `c1` with:

```sh
$ mount | grep shared
/dev/vda1 on /shared type ext4 (rw,relatime)
```

As you can see, a volume is mounted on `/shared` — it's the `shared` volume we created earlier.

Now let's create some files:

```sh
$ echo "foo" > /tmp/foo
$ echo "bar" > /shared/bar
```

Let's check the same files from the second container.

First connect to it with:

```sh
$ kubectl exec -it podtest --container c2 -- sh
```

```sh
$ cat /shared/bar
bar
$ cat /tmp/foo
$ cat: can't open '/tmp/foo': No such file or directory
```

As you can see, the file created in the `shared` directory is available on both containers, but the file in `/tmp` isn't.

This is because other than volume, the containers' filesystems are entirely isolated from each other.

Now let's take a look at networking and process isolation.

A good way of seeing how the network is set up is to use the command `ip link`, which shows the Linux system's network devices.

Let's execute the command in the first container:

```sh
$ kubectl exec -it podtest -c c1 -- ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
178: eth0@if179: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1450 qdisc noqueue
    link/ether 46:4c:58:6c:da:37 brd ff:ff:ff:ff:ff:ff
```

And now the same command in the other:

```sh
$ kubectl exec -it podtest -c c2 -- ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
178: eth0@if179: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1450 qdisc noqueue
    link/ether 46:4c:58:6c:da:37 brd ff:ff:ff:ff:ff:ff
```

You can see that both containers have:

- The same device `eth0`.
- The same MAC addresses `46:4c:58:6c:da:37`.

Since MAC addresses are supposed to be globally unique, this is a clear indication that the pods share the same device.

Now let's see network sharing in action!

Let's connect to the first container with:

```sh
$ kubectl exec -it podtest -c c1 -- sh
```

Start a very simple network listener with `nc`:

c1@podtest

```sh
$ nc -lk -p 5000 127.0.0.1 -e 'date'
```

The command starts a listener on localhost on port 5000 and prints the `date` command to any connected TCP client.

*Can the second container connect to it?*

Open a terminal in the second container with:

bash

```sh
$ kubectl exec -it podtest -c c2 -- sh
```

Now you can verify that the second container *can* connect to the network listener, but *cannot* see the `nc` process:

```sh
$ telnet localhost 5000
Connected to localhost
Sun Nov 29 00:57:37 UTC 2020
Connection closed by foreign host

$ ps aux
PID   USER     TIME  COMMAND
    1 root      0:00 sleep 5000
   73 root      0:00 sh
   81 root      0:00 ps aux
```

Connecting over `telnet`, you can see the output of `date`, which proves that the `nc` listener is working, but `ps aux` (which shows all processes on the container) doesn't show `nc` at all.

This is because containers within a pod have process isolation but not network isolation.

This explains how the Ambassador Pattern works:

1. Since all containers share the same network namespace, a single container can listen to all connections — even external ones.
2. The rest of the containers only accept connections from localhost — rejecting any external connection.

The container that receives external traffic is the *Ambassador*, hence the name of the pattern.

![The ambassador pattern in a Pod](D:\学习资料\笔记\k8s\k8s图\32e814422af5af45f0a7566bd5596ba5.svg)

> One crucial thing to remember, though: because the network namespace is shared, multiple containers in a pod can't listen on the same port!

Let's have a look at some other use cases for multi-container pods.



### Exposing metrics with a standard interface

Let's say you've standardized on using [Prometheus](http://prometheus.io/) for monitoring all of the services in your Kubernetes cluster, but you're using some applications that don't natively export Prometheus metrics *(for example, Elasticsearch).*

*Can you add Prometheus metrics to your pods without altering your application code?*

Indeed you can, using the **Adapter Pattern**.

For the Elasticsearch example, let's add an "exporter" container to the pod that exposes various Elasticsearch metrics in the Prometheus format.

This will be easy, because there's an [open-source exporter for Elasticsearch](https://github.com/justwatchcom/elasticsearch_exporter) (you'll also need to add the relevant port to the Service):

es-prometheus.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: elasticsearch
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: elasticsearch
  template:
    metadata:
      labels:
        app.kubernetes.io/name: elasticsearch
    spec:
      containers:
        - name: elasticsearch
          image: elasticsearch:7.9.3
          env:
            - name: discovery.type
              value: single-node
          ports:
            - name: http
              containerPort: 9200
        - name: prometheus-exporter
          image: justwatch/elasticsearch_exporter:1.1.0
          args:
            - '--es.uri=http://localhost:9200'
          ports:
            - name: http-prometheus
              containerPort: 9114
---
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch
spec:
  selector:
    app.kubernetes.io/name: elasticsearch
  ports:
    - name: http
      port: 9200
      targetPort: http
    - name: http-prometheus
      port: 9114
      targetPort: http-prometheus
```

Once this has been applied, you can find the metrics exposed on port 9114:

```sh
$ kubectl run -it --rm --image=curlimages/curl curl \
  -- curl -s elasticsearch:9114/metrics | head
# HELP elasticsearch_breakers_estimated_size_bytes Estimated size in bytes of breaker
# TYPE elasticsearch_breakers_estimated_size_bytes gauge
elasticsearch_breakers_estimated_size_bytes{breaker="accounting",name="elasticsearch-ss86j"} 0
elasticsearch_breakers_estimated_size_bytes{breaker="fielddata",name="elasticsearch-ss86j"} 0
elasticsearch_breakers_estimated_size_bytes{breaker="in_flight_requests",name="elasticsearch-ss86j"} 0
elasticsearch_breakers_estimated_size_bytes{breaker="model_inference",name="elasticsearch-ss86j"} 0
elasticsearch_breakers_estimated_size_bytes{breaker="parent",name="elasticsearch-ss86j"} 1.61106136e+08
elasticsearch_breakers_estimated_size_bytes{breaker="request",name="elasticsearch-ss86j"} 16440
# HELP elasticsearch_breakers_limit_size_bytes Limit size in bytes for breaker
# TYPE elasticsearch_breakers_limit_size_bytes gauge
```

Once again, you've been able to alter your application's behaviour without actually changing your code or your container images.

You've exposed standardized Prometheus metrics that can be consumed by cluster-wide tools (like the [Prometheus Operator](https://github.com/prometheus-operator/prometheus-operator)), and have thus achieved a good separation of concerns between the application and the underlying infrastructure.



### Tailing logs

Next, let's take a look at the **Sidecar Pattern**, where you add a container to a pod that enhances an application in some way.

The Sidecar Pattern is pretty general and can apply to all sorts of different use cases (and you'll often hear any containers in a pod past the first referred to as "sidecars").

Let's first explore one of the classic sidecar use cases: a log tailing sidecar.

In a containerized environment, the best practice is to always log to standard out so that logs can be collected and aggregated in a centralized manner.

**But many older applications were designed to log to files, and changing that can sometimes be non-trivial.**

Adding a log tailing sidecar means you might not have to!

Let's return to Elasticsearch as an example, which is a bit contrived since the Elasticsearch container logs to standard out by default (and it's non-trivial to get it to log to a file).

Here's what the deployment looks like:

sidecar-example.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: elasticsearch
  labels:
    app.kubernetes.io/name: elasticsearch
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: elasticsearch
  template:
    metadata:
      labels:
        app.kubernetes.io/name: elasticsearch
    spec:
      containers:
        - name: elasticsearch
          image: elasticsearch:7.9.3
          env:
            - name: discovery.type
              value: single-node
            - name: path.logs
              value: /var/log/elasticsearch
          volumeMounts:
            - name: logs
              mountPath: /var/log/elasticsearch
            - name: logging-config
              mountPath: /usr/share/elasticsearch/config/log4j2.properties
              subPath: log4j2.properties
              readOnly: true
          ports:
            - name: http
              containerPort: 9200
        - name: logs
          image: alpine:3.12
          command:
            - tail
            - -f
            - /logs/docker-cluster_server.json
          volumeMounts:
            - name: logs
              mountPath: /logs
              readOnly: true
      volumes:
        - name: logging-config
          configMap:
            name: elasticsearch-logging
        - name: logs
          emptyDir: {}
```

> The logging configuration file is a separate ConfigMap that's too long to include here.

Both containers share a common volume named `logs`.

The Elasticsearch container writes logs to that volume, while the `logs` container just reads from the appropriate file and outputs it to standard out.

You can retrieve the log stream by specifying the appropriate container with `kubectl logs`:

bash

```sh
$ kubectl logs elasticsearch-6f88d74475-jxdhl logs | head
{
  "type": "server",
  "timestamp": "2020-11-29T23:01:42,849Z",
  "level": "INFO",
  "component": "o.e.n.Node",
  "cluster.name": "docker-cluster",
  "node.name": "elasticsearch-6f88d74475-jxdhl",
  "message": "version[7.9.3], pid[7], OS[Linux/5.4.0-52-generic/amd64], JVM"
}
{
  "type": "server",
  "timestamp": "2020-11-29T23:01:42,855Z",
  "level": "INFO",
  "component": "o.e.n.Node",
  "cluster.name": "docker-cluster",
  "node.name": "elasticsearch-6f88d74475-jxdhl",
  "message": "JVM home [/usr/share/elasticsearch/jdk]"
}
{
  "type": "server",
  "timestamp": "2020-11-29T23:01:42,856Z",
  "level": "INFO",
  "component": "o.e.n.Node",
  "cluster.name": "docker-cluster",
  "node.name": "elasticsearch-6f88d74475-jxdhl",
  "message": "JVM arguments […]"
}
```

The great thing about using a sidecar is that streaming to standard out isn't the only option.

If you needed to switch to a customized log aggregation service, you could just change the sidecar container without altering anything else about your application.



**Other examples of sidecars**

There are many use cases for sidecars; a logging container is only one (straightforward) example.

Here are some other use cases you might encounter in the wild:

- [Reloading ConfigMaps](https://github.com/jimmidyson/configmap-reload) in realtime without requiring pod restarts.
- [Injecting secrets from Hashicorp Vault](https://www.vaultproject.io/docs/platform/k8s/injector) into your application.
- [Adding a local Redis instance](https://thenewstack.io/tutorial-apply-the-sidecar-pattern-to-deploy-redis-in-kubernetes/) to your application for low-latency in-memory caching.



### Preparing for a pod to run

All of the examples of multi-container pods this post has gone over so far involve several containers running simultaneously.

Kubernetes also provides the ability to run [**Init Containers**](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/), which are containers that run to completion before the "normal" containers start.

This allows you to run an initialization script before your pod starts in earnest.

*Why would you want your preparation to run in a separate container, instead of (for instance) adding some initialization to your container's [entrypoint](https://docs.docker.com/engine/reference/builder/#entrypoint) script?*

Let's look to Elasticsearch for a real-world example.

The [Elasticsearch docs](https://www.elastic.co/guide/en/elasticsearch/reference/current/vm-max-map-count.html) recommending setting the `vm.max_map_count` sysctl setting in production-ready deployments.

**This is problematic in containerized environments since there's no container-level isolation for sysctls and any changes have to happen on the node level.**

*How can you handle this in cases where you can't customize the Kubernetes nodes?*

One way would be to run Elasticsearch in a [privileged container](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/), which would give Elasticsearch the ability to change system settings on its host node, and alter the entrypoint script to add the sysctls.

**But this would be extremely dangerous from a security perspective!**

If the Elasticsearch service were ever compromised, an attacker would have root access to its host node.

You can use an init container to mitigate this risk somewhat:

init-es.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: elasticsearch
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: elasticsearch
  template:
    metadata:
      labels:
        app.kubernetes.io/name: elasticsearch
    spec:
      initContainers:
        - name: update-sysctl
          image: alpine:3.12
          command: ['/bin/sh']
          args:
            - -c
            - |
              sysctl -w vm.max_map_count=262144
          securityContext:
            privileged: true
      containers:
        - name: elasticsearch
          image: elasticsearch:7.9.3
          env:
            - name: discovery.type
              value: single-node
          ports:
            - name: http
              containerPort: 9200
```

The pod sets the sysctl in a privileged init container, after which the Elasticsearch container starts as expected.

You're still using a privileged container, which isn't ideal, but at least it's extremely minimal and short-lived, so the attack surface is much lower.

> This is the approach recommended by the [Elastic Cloud Operator](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-virtual-memory.html).

Using a privileged init container to prepare a node for running a pod is a fairly common pattern.

For instance, Istio uses init containers to set up iptables rules every time a pod runs.

Another reason to use an init container is to prepare the pod's filesystem in some way.

One common use case is secrets management.



### Another init container use case

If you're using something like [HashicCorp Vault](https://www.hashicorp.com/) for secrets management instead of Kubernetes secrets, you can retrieve secrets in an init container and persist them to a shared `emptyDir` volume.

It might look something like this:

init-secrets.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app.kubernetes.io/name: myapp
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: myapp
  template:
    metadata:
      labels:
        app.kubernetes.io/name: myapp
    spec:
      initContainers:
        - name: get-secret
          image: vault
          volumeMounts:
            - name: secrets
              mountPath: /secrets
          command: ['/bin/sh']
          args:
            - -c
            - |
              vault read secret/my-secret > /secrets/my-secret
      containers:
        - name: myapp
          image: myapp
          volumeMounts:
            - name: secrets
              mountPath: /secrets
      volumes:
        - name: secrets
          emptyDir: {}
```

Now the `secret/my-secret` secret will be available on the filesystem for the `myapp` container.

This is the basic idea of how systems like the [Vault Agent Sidecar Injector](https://www.vaultproject.io/docs/platform/k8s/injector) work. However, they're quite a bit more sophisticated in practice (combining [mutating webhooks](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#mutatingadmissionwebhook), init containers, and sidecars to hide most of the complexity).



**Even more init container use cases**

Here are some other reasons you might want to use an init container:

- You want a database migration script to run before your application (this can generally be accomplished in an entrypoint script, but is sometimes easier to do so with a dedicated container).
- You want to retrieve a large file from S3 or GCS that your application depends on (using an init container for this helps to avoid bloat in your application container).



### Summary

This post covered quite a lot of ground, so here's a table of some multi-container patterns and when you might want to use them:

| Use Case                                                  | Ambassador Pattern | Adapter Pattern | Sidecar Pattern | Init Pattern |
| :-------------------------------------------------------- | :----------------: | :-------------: | :-------------: | :----------: |
| Encrypt and/or authenticate incoming requests             |         ✅          |                 |                 |              |
| Connect to external resources over a secure tunnel        |         ✅          |                 |                 |              |
| Expose metrics in a standardized format (e.g. Prometheus) |                    |        ✅        |                 |              |
| Stream logs from a file to a log aggregator               |                    |                 |        ✅        |              |
| Add a local Redis cache to your pod                       |                    |                 |        ✅        |              |
| Monitor and live-reload ConfigMaps                        |                    |                 |        ✅        |              |
| Inject secrets from Vault into your application           |                    |                 |        ✅        |      ✅       |
| Change node-level settings with a privileged container    |                    |                 |                 |      ✅       |
| Retrieve files from S3 before your application starts     |                    |                 |                 |      ✅       |

Be sure to read the [official documentation](https://kubernetes.io/docs/concepts/workloads/pods/) and the original [container design pattern paper](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/45406.pdf) if you want to dig deeper into this subject.





















# 集群排错



本章介绍集群状态异常的排错方法，包括 Kubernetes 主要组件以及必备扩展（如 kube-dns）等，而有关网络的异常排错请参考[网络异常排错方法]()。



### 概述

排查集群状态异常问题通常从 Node 和 Kubernetes 服务 的状态出发，定位出具体的异常服务，再进而寻找解决方法。集群状态异常可能的原因比较多，常见的有

- 虚拟机或物理机宕机
- 网络分区
- Kubernetes 服务未正常启动
- 数据丢失或持久化存储不可用（一般在公有云或私有云平台中）
- 操作失误（如配置错误）

按照不同的组件来说，具体的原因可能包括

- kube-apiserver 无法启动会导致
  - 集群不可访问
  - 已有的 Pod 和服务正常运行（依赖于 Kubernetes API 的除外）
- etcd 集群异常会导致
  - kube-apiserver 无法正常读写集群状态，进而导致 Kubernetes API 访问出错
  - kubelet 无法周期性更新状态
- kube-controller-manager/kube-scheduler 异常会导致
  - 复制控制器、节点控制器、云服务控制器等无法工作，从而导致 Deployment、Service 等无法工作，也无法注册新的 Node 到集群中来
  - 新创建的 Pod 无法调度（总是 Pending 状态）
- Node 本身宕机或者 Kubelet 无法启动会导致
  - Node 上面的 Pod 无法正常运行
  - 已在运行的 Pod 无法正常终止
- 网络分区会导致 Kubelet 等与控制平面通信异常以及 Pod 之间通信异常

为了维持集群的健康状态，推荐在部署集群时就考虑以下

- 在云平台上开启 VM 的自动重启功能
- 为 Etcd 配置多节点高可用集群，使用持久化存储（如 AWS EBS 等），定期备份数据
- 为控制平面配置高可用，比如多 kube-apiserver 负载均衡以及多节点运行 kube-controller-manager、kube-scheduler 以及 kube-dns 等
- 尽量使用复制控制器和 Service，而不是直接管理 Pod
- 跨地域的多 Kubernetes 集群



### 查看 Node 状态

一般来说，可以首先查看 Node 的状态，确认 Node 本身是不是 Ready 状态

```sh
kubectl get nodes
kubectl describe node <node-name>
```

如果是 NotReady 状态，则可以执行 `kubectl describe node <node-name>` 命令来查看当前 Node 的事件。这些事件通常都会有助于排查 Node 发生的问题。



### SSH 登录 Node

在排查 Kubernetes 问题时，通常需要 SSH 登录到具体的 Node 上面查看 kubelet、docker、iptables 等的状态和日志。在使用云平台时，可以给相应的 VM 绑定一个公网 IP；而在物理机部署时，可以通过路由器上的端口映射来访问。但更简单的方法是使用 SSH Pod （不要忘记替换成你自己的 nodeName）：

```yaml
# cat ssh.yaml
apiVersion: v1
kind: Service
metadata:
  name: ssh
spec:
  selector:
    app: ssh
  type: LoadBalancer
  ports:
  - protocol: TCP
    port: 22
    targetPort: 22
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: ssh
  labels:
    app: ssh
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ssh
  template:
    metadata:
      labels:
        app: ssh
    spec:
      containers:
      - name: alpine
        image: alpine
        ports:
        - containerPort: 22
        stdin: true
        tty: true
      hostNetwork: true
      nodeName: <node-name>
```

```sh
$ kubectl create -f ssh.yaml
$ kubectl get svc ssh
NAME      TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)        AGE
ssh       LoadBalancer   10.0.99.149   52.52.52.52   22:32008/TCP   5m
```

接着，就可以通过 ssh 服务的外网 IP 来登录 Node，如 `ssh user@52.52.52.52`。

在使用完后， 不要忘记删除 SSH 服务 `kubectl delete -f ssh.yaml`。



### 查看日志

一般来说，Kubernetes 的主要组件有两种部署方法

- 直接使用 systemd 等启动控制节点的各个服务
- 使用 Static Pod 来管理和启动控制节点的各个服务

使用 systemd 等管理控制节点服务时，查看日志必须要首先 SSH 登录到机器上，然后查看具体的日志文件。如

```sh
journalctl -l -u kube-apiserver
journalctl -l -u kube-controller-manager
journalctl -l -u kube-scheduler
journalctl -l -u kubelet
journalctl -l -u kube-proxy
```

或者直接查看日志文件

- /var/log/kube-apiserver.log
- /var/log/kube-scheduler.log
- /var/log/kube-controller-manager.log
- /var/log/kubelet.log
- /var/log/kube-proxy.log

而对于使用 Static Pod 部署集群控制平面服务的场景，可以参考下面这些查看日志的方法。



### kube-apiserver 日志

```sh
PODNAME=$(kubectl -n kube-system get pod -l component=kube-apiserver -o jsonpath='{.items[0].metadata.name}')
kubectl -n kube-system logs $PODNAME --tail 100
```



### kube-controller-manager 日志

```sh
PODNAME=$(kubectl -n kube-system get pod -l component=kube-controller-manager -o jsonpath='{.items[0].metadata.name}')
kubectl -n kube-system logs $PODNAME --tail 100
```



### kube-scheduler 日志

```sh
PODNAME=$(kubectl -n kube-system get pod -l component=kube-scheduler -o jsonpath='{.items[0].metadata.name}')
kubectl -n kube-system logs $PODNAME --tail 100
```



### kube-dns 日志

```sh
PODNAME=$(kubectl -n kube-system get pod -l k8s-app=kube-dns -o jsonpath='{.items[0].metadata.name}')
kubectl -n kube-system logs $PODNAME -c kubedns
```



### Kubelet 日志

查看 Kubelet 日志需要首先 SSH 登录到 Node 上。

```sh
journalctl -l -u kubelet
```



### Kube-proxy 日志

Kube-proxy 通常以 DaemonSet 的方式部署

```sh
$ kubectl -n kube-system get pod -l component=kube-proxy
NAME               READY     STATUS    RESTARTS   AGE
kube-proxy-42zpn   1/1       Running   0          1d
kube-proxy-7gd4p   1/1       Running   0          3d
kube-proxy-87dbs   1/1       Running   0          4d
$ kubectl -n kube-system logs kube-proxy-42zpn
```



### Kube-dns/Dashboard CrashLoopBackOff

由于 Dashboard 依赖于 kube-dns，所以这个问题一般是由于 kube-dns 无法正常启动导致的。查看 kube-dns 的日志

```sh
$ kubectl logs --namespace=kube-system $(kubectl get pods --namespace=kube-system -l k8s-app=kube-dns -o name) -c kubedns
$ kubectl logs --namespace=kube-system $(kubectl get pods --namespace=kube-system -l k8s-app=kube-dns -o name) -c dnsmasq
$ kubectl logs --namespace=kube-system $(kubectl get pods --namespace=kube-system -l k8s-app=kube-dns -o name) -c sidecar
```

可以发现如下的错误日志

```sh
Waiting for services and endpoints to be initialized from apiserver...
skydns: failure to forward request "read udp 10.240.0.18:47848->168.63.129.16:53: i/o timeout"
Timeout waiting for initialization
```

这说明 kube-dns pod 无法转发 DNS 请求到上游 DNS 服务器。解决方法为

- 如果使用的 Docker 版本大于 1.12，则在每个 Node 上面运行 `iptables -P FORWARD ACCEPT` 开启 Docker 容器的 IP 转发
- 等待一段时间，如果还未恢复，则检查 Node 网络是否正确配置，比如是否可以正常请求上游DNS服务器、是否开启了 IP 转发（包括 Node 内部和公有云上虚拟网卡等）、是否有安全组禁止了 DNS 请求等

如果错误日志中不是转发 DNS 请求超时，而是访问 kube-apiserver 超时，比如

```sh
E0122 06:56:04.774977       1 reflector.go:199] k8s.io/dns/vendor/k8s.io/client-go/tools/cache/reflector.go:94: Failed to list *v1.Endpoints: Get https://10.0.0.1:443/api/v1/endpoints?resourceVersion=0: dial tcp 10.0.0.1:443: i/o timeout
I0122 06:56:04.775358       1 dns.go:174] Waiting for services and endpoints to be initialized from apiserver...
E0122 06:56:04.775574       1 reflector.go:199] k8s.io/dns/vendor/k8s.io/client-go/tools/cache/reflector.go:94: Failed to list *v1.Service: Get https://10.0.0.1:443/api/v1/services?resourceVersion=0: dial tcp 10.0.0.1:443: i/o timeout
I0122 06:56:05.275295       1 dns.go:174] Waiting for services and endpoints to be initialized from apiserver...
I0122 06:56:05.775182       1 dns.go:174] Waiting for services and endpoints to be initialized from apiserver...
I0122 06:56:06.275288       1 dns.go:174] Waiting for services and endpoints to be initialized from apiserver...
```

这说明 Pod 网络（一般是多主机之间）访问异常，包括 Pod->Node、Node->Pod 以及 Node-Node 等之间的往来通信异常。可能的原因比较多，具体的排错方法可以参考[网络异常排错指南]()。



### Node NotReady

Node 处于 NotReady 状态，大部分是由于 PLEG（Pod Lifecycle Event Generator）问题导致的。社区 issue [#45419](https://github.com/kubernetes/kubernetes/issues/45419) 目前还处于未解决状态。

NotReady 的原因比较多，在排查时最重要的就是执行 `kubectl describe node <node name>` 并查看 Kubelet 日志中的错误信息。常见的问题及修复方法为：

- Kubelet 未启动或者异常挂起：重新启动 Kubelet。
- CNI 网络插件未部署：部署 CNI 插件。
- Docker 僵死（API 不响应）：重启 Docker。
- 磁盘空间不足：清理磁盘空间，比如镜像、临时文件等。

> Kubernetes node 有可能会出现各种硬件、内核或者运行时等问题，这些问题有可能导致服务异常。而 Node Problem Detector（NPD）就是用来监测这些异常的服务。NPD 以 DaemonSet 的方式运行在每台 Node 上面，并在异常发生时更新 NodeCondition（比如 KernelDaedlock、DockerHung、BadDisk 等）或者 Node Event（比如 OOM Kill 等）。
>
> 可以参考 [kubernetes/node-problem-detector](https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/node-problem-detector) 来部署 NPD，以便更快发现 Node 上的问题。



### Kubelet: failed to initialize top level QOS containers

重启 kubelet 时报错 `Failed to start ContainerManager failed to initialise top level QOS containers`（参考 [#43856](https://github.com/kubernetes/kubernetes/issues/43856)），临时解决方法是：

1. 在 docker.service 配置中增加 `--exec-opt native.cgroupdriver=systemd` 选项。
2. 重启主机

该问题已于2017年4月27日修复（v1.7.0+， [#44940](https://github.com/kubernetes/kubernetes/pull/44940)）。更新集群到新版本即可解决这个问题。



### Kubelet 一直报 FailedNodeAllocatableEnforcement 事件

当 NodeAllocatable 特性未开启时（即 kubelet 设置了 `--cgroups-per-qos=false` ），查看 node 的事件会发现每分钟都会有 `Failed to update Node Allocatable Limits` 的警告信息：

```sh
$ kubectl describe node node1
Events:
  Type     Reason                            Age                  From                               Message
  ----     ------                            ----                 ----                               -------
  Warning  FailedNodeAllocatableEnforcement  2m (x1001 over 16h)  kubelet, aks-agentpool-22604214-0  Failed to update Node Allocatable Limits "": failed to set supported cgroup subsystems for cgroup : Failed to set config for supported subsystems : failed to write 7285047296 to memory.limit_in_bytes: write /var/lib/docker/overlay2/5650a1aadf9c758946073fefa1558446ab582148ddd3ee7e7cb9d269fab20f72/merged/sys/fs/cgroup/memory/memory.limit_in_bytes: invalid argument
```

如果 NodeAllocatable 特性确实不需要，那么该警告事件可以忽略。但根据 Kubernetes 文档 [Reserve Compute Resources for System Daemons](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/)，最好开启该特性：

> Kubernetes nodes can be scheduled to `Capacity`. Pods can consume all the available capacity on a node by default. This is an issue because nodes typically run quite a few system daemons that power the OS and Kubernetes itself. Unless resources are set aside for these system daemons, pods and system daemons compete for resources and lead to resource starvation issues on the node.
>
> The `kubelet` exposes a feature named `Node Allocatable` that helps to reserve compute resources for system daemons. Kubernetes recommends cluster administrators to configure `Node Allocatable` based on their workload density on each node.
>
> ```
>       Node Capacity
> ---------------------------
> |     kube-reserved       |
> |-------------------------|
> |     system-reserved     |
> |-------------------------|
> |    eviction-threshold   |
> |-------------------------|
> |                         |
> |      allocatable        |
> |   (available for pods)  |
> |                         |
> |                         |
> ---------------------------
> ```

开启方法为：

```sh
kubelet --cgroups-per-qos=true --enforce-node-allocatable=pods ...
```



### Kube-proxy: error looking for path of conntrack

kube-proxy 报错，并且 service 的 DNS 解析异常

```sh
kube-proxy[2241]: E0502 15:55:13.889842    2241 conntrack.go:42] conntrack returned error: error looking for path of conntrack: exec: "conntrack": executable file not found in $PATH
```

解决方式是安装 `conntrack-tools` 包后重启 kube-proxy 即可。



### Dashboard 中无资源使用图表

正常情况下，Dashboard 首页应该会显示资源使用情况的图表，如

![image-20210207213052305](D:\学习资料\笔记\k8s\k8s图\image-20210207213052305.png)

如果没有这些图表，则需要首先检查 Heapster 是否正在运行（因为Dashboard 需要访问 Heapster 来查询资源使用情况）：

```sh
kubectl -n kube-system get pods -l k8s-app=heapster
NAME                        READY     STATUS    RESTARTS   AGE
heapster-86b59f68f6-h4vt6   2/2       Running   0          5d
```

如果查询结果为空，说明 Heapster 还未部署，可以参考 https://github.com/kubernetes/heapster 来部署。

但如果 Heapster 处于正常状态，那么需要查看 dashboard 的日志，确认是否还有其他问题

```sh
$ kubectl -n kube-system get pods -l k8s-app=kubernetes-dashboardNAME                                   READY     STATUS    RESTARTS   AGEkubernetes-dashboard-665b4f7df-dsjpn   1/1       Running   0          5d
$ kubectl -n kube-system logs kubernetes-dashboard-665b4f7df-dsjpn
```

> 注意：Heapster 已被社区弃用，推荐部署 metrics-server 来获取这些指标。支持 metrics-server 的 dashboard 可以参考[这里](https://github.com/kubernetes/dashboard/blob/master/aio/deploy/recommended/kubernetes-dashboard-head.yaml)。



### HPA 不自动扩展 Pod

查看 HPA 的事件，发现

```sh
$ kubectl describe hpa php-apache
Name:                                                  php-apache
Namespace:                                             default
Labels:                                                <none>
Annotations:                                           <none>
CreationTimestamp:                                     Wed, 27 Dec 2017 14:36:38 +0800
Reference:                                             Deployment/php-apache
Metrics:                                               ( current / target )
  resource cpu on pods  (as a percentage of request):  <unknown> / 50%
Min replicas:                                          1
Max replicas:                                          10
Conditions:
  Type           Status  Reason                   Message
  ----           ------  ------                   -------
  AbleToScale    True    SucceededGetScale        the HPA controller was able to get the target's current scale
  ScalingActive  False   FailedGetResourceMetric  the HPA was unable to compute the replica count: unable to get metrics for resource cpu: unable to fetch metrics from API: the server could not find the requested resource (get pods.metrics.k8s.io)
Events:
  Type     Reason                   Age                  From                       Message
  ----     ------                   ----                 ----                       -------
  Warning  FailedGetResourceMetric  3m (x2231 over 18h)  horizontal-pod-autoscaler  unable to get metrics for resource cpu: unable to fetch metrics from API: the server could not find the requested resource (get pods.metrics.k8s.io)
```

这说明 [metrics-server]() 未部署，可以参考 [这里]() 部署。



### Node 存储空间不足

Node 存储空间不足一般是容器镜像未及时清理导致的，比如短时间内运行了很多使用较大镜像的容器等。Kubelet 会自动清理未使用的镜像，但如果想要立即清理，可以使用 [spotify/docker-gc](https://github.com/spotify/docker-gc)：

```sh
sudo docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v /etc:/etc:ro spotify/docker-gc
```

你也可以 SSH 到 Node 上，执行下面的命令来查看占用空间最多的镜像：

```sh
# 按镜像大小由大到小排序
sudo docker images --format '{{.Size}}\t{{.Repository}}:{{.Tag}}\t{{.ID}}' | sort -h -r | column -t

# 统计总值
$ docker images --format '{{.Size}}\t{{.Repository}}:{{.Tag}}\t{{.ID}}' | sort -h -r | column -t | awk '{print $1}'  | awk '{sum+=$0}END{print sum}'
```



### /sys/fs/cgroup 空间不足

很多发行版默认的 fs.inotify.max_user_watches 太小，只有 8192，可以通过增大该配置解决。比如

```sh
$ sudo sysctl fs.inotify.max_user_watches=524288
```

除此之外，社区也存在 [no space left on /sys/fs/cgroup](https://github.com/kubernetes/kubernetes/issues/70324) 以及 [Kubelet CPU/Memory Usage linearly increases using CronJob](https://github.com/kubernetes/kubernetes/issues/64137) 的问题。临时解决方法有两种：

- 参考 [这里的 Gist](https://gist.github.com/reaperes/34ed7b07344ccc61b9570c46a3b4e564) 通过定时任务定期清理 systemd cgroup
- 或者，参考 [这里](https://github.com/derekrprice/k8s-hacks/blob/master/systemd-cgroup-gc.yaml) 通过 Daemonset 定期清理 systemd cgroup



### 大量 ConfigMap/Secret 导致Kubernetes缓慢

这是从 Kubernetes 1.12 开始才有的问题，Kubernetes issue: [#74412](https://github.com/kubernetes/kubernetes/issues/74412)。

> This worked well on version 1.11 of Kubernetes. After upgrading to 1.12 or 1.13, I've noticed that doing this will cause the cluster to significantly slow down; up to the point where nodes are being marked as NotReady and no new work is being scheduled.
>
> For example, consider a scenario in which I schedule 400 jobs, each with its own ConfigMap, which print "Hello World" on a single-node cluster would.
>
> - On v1.11, it takes about 10 minutes for the cluster to process all jobs. New jobs can be scheduled.
> - On v1.12 and v1.13, it takes about 60 minutes for the cluster to process all jobs. After this, no new jobs can be scheduled.
>
> This is related to max concurrent http2 streams and the change of configmap manager of kubelet. By default, max concurrent http2 stream of http2 server in kube-apiserver is 250, and every configmap will consume one stream to watch in kubelet at least from version 1.13.x. Kubelet will stuck to communicate to kube-apiserver and then become NotReady if too many pods with configmap scheduled to it. A work around is to change the config http2-max-streams-per-connection of kube-apiserver to a bigger value.

临时解决方法：为 Kubelet 设置 `configMapAndSecretChangeDetectionStrategy: Cache` （参考 [这里](https://github.com/kubernetes/kubernetes/pull/74755) ）。

修复方法：升级 Go 版本到 1.12 后重新构建 Kubernetes（社区正在进行中）。修复后，Kubelet 可以 watch 的 configmap 可以从之前的 236 提高到至少 10000。



### Kubelet 内存泄漏

这是从 1.12 版本开始有的问题（只在使用 hyperkube 启动 kubelet 时才有问题），社区 issue 为 [#73587](https://github.com/kubernetes/kubernetes/issues/73587)。

```sh
(pprof) root@ip-172-31-10-50:~# go tool pprof  http://localhost:10248/debug/pprof/heap
Fetching profile from http://localhost:10248/debug/pprof/heap
Saved profile in /root/pprof/pprof.hyperkube.localhost:10248.alloc_objects.alloc_space.inuse_objects.inuse_space.002.pb.gz
Entering interactive mode (type "help" for commands)
(pprof) top
2406.93MB of 2451.55MB total (98.18%)
Dropped 2863 nodes (cum <= 12.26MB)
Showing top 10 nodes out of 34 (cum >= 2411.39MB)
      flat  flat%   sum%        cum   cum%
 2082.07MB 84.93% 84.93%  2082.07MB 84.93%  k8s.io/kubernetes/vendor/github.com/beorn7/perks/quantile.newStream (inline)
  311.65MB 12.71% 97.64%  2398.72MB 97.84%  k8s.io/kubernetes/vendor/github.com/prometheus/client_golang/prometheus.newSummary
   10.71MB  0.44% 98.08%  2414.43MB 98.49%  k8s.io/kubernetes/vendor/github.com/prometheus/client_golang/prometheus.(*MetricVec).getOrCreateMetricWithLabelValues
    2.50MB   0.1% 98.18%  2084.57MB 85.03%  k8s.io/kubernetes/vendor/github.com/beorn7/perks/quantile.NewTargeted
         0     0% 98.18%  2412.06MB 98.39%  k8s.io/kubernetes/cmd/kubelet/app.startKubelet.func1
         0     0% 98.18%  2412.06MB 98.39%  k8s.io/kubernetes/pkg/kubelet.(*Kubelet).HandlePodAdditions
         0     0% 98.18%  2412.06MB 98.39%  k8s.io/kubernetes/pkg/kubelet.(*Kubelet).Run
```

```sh
curl -s localhost:10255/metrics | sed 's/{.*//' | sort | uniq -c | sort -nr
  25749 reflector_watch_duration_seconds
  25749 reflector_list_duration_seconds
  25749 reflector_items_per_watch
  25749 reflector_items_per_list
   8583 reflector_watches_total
   8583 reflector_watch_duration_seconds_sum
   8583 reflector_watch_duration_seconds_count
   8583 reflector_short_watches_total
   8583 reflector_lists_total
   8583 reflector_list_duration_seconds_sum
   8583 reflector_list_duration_seconds_count
   8583 reflector_last_resource_version
   8583 reflector_items_per_watch_sum
   8583 reflector_items_per_watch_count
   8583 reflector_items_per_list_sum
   8583 reflector_items_per_list_count
    165 storage_operation_duration_seconds_bucket
     51 kubelet_runtime_operations_latency_microseconds
     44 rest_client_request_latency_seconds_bucket
     33 kubelet_docker_operations_latency_microseconds
     17 kubelet_runtime_operations_latency_microseconds_sum
     17 kubelet_runtime_operations_latency_microseconds_count
     17 kubelet_runtime_operations
```

修复方法：禁止 [Reflector metrics](https://github.com/kubernetes/kubernetes/issues/73587)。



### kube-controller-manager 无法更新 Object

参考[kubernetes#95958](https://github.com/kubernetes/kubernetes/issues/95958)，kube-controller-manager 报错：

```sh
Event(v1.ObjectReference{Kind:"HorizontalPodAutoscaler", Namespace:"cig-prod-apps", Name:"<omitted>", UID:"4593f854-b824-4a9e-8e10-c16d558797b9", APIVersion:"autoscaling/v2beta2", ResourceVersion:"71905040", FieldPath:""}): type: 'Warning' reason: 'FailedUpdateStatus' Operation cannot be fulfilled on horizontalpodautoscalers.autoscaling "<omitted>": the object has been modified; please apply your changes to the latest version and try again
```

这是由于 etcd restore 之后，在重启 kube-apiserver 之前，控制平面各个组件缓存中的 Object 版本跟 etcd 备份中不一致。

解决方法是是在 etcd restore 之后，重启控制平面所有组件。



### 其他已知问题

- [Kubernetes is vulnerable to stale reads, violating critical pod safety guarantees](https://github.com/kubernetes/kubernetes/issues/59848)



### 参考文档

- [Troubleshoot Clusters](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-cluster/)
- [SSH into Azure Container Service (AKS) cluster nodes](https://docs.microsoft.com/en-us/azure/aks/aks-ssh#configure-ssh-access)





# Pod 排错



本章介绍 Pod 运行异常的排错方法。

一般来说，无论 Pod 处于什么异常状态，都可以执行以下命令来查看 Pod 的状态

- `kubectl get pod <pod-name> -o yaml` 查看 Pod 的配置是否正确
- `kubectl describe pod <pod-name>` 查看 Pod 的事件
- `kubectl logs <pod-name> [-c <container-name>]` 查看容器日志

这些事件和日志通常都会有助于排查 Pod 发生的问题。



### Pod 一直处于 Pending 状态

Pending 说明 Pod 还没有调度到某个 Node 上面。可以通过 `kubectl describe pod <pod-name>` 命令查看到当前 Pod 的事件，进而判断为什么没有调度。如

```sh
$ kubectl describe pod mypod
...
Events:
  Type     Reason            Age                From               Message
  ----     ------            ----               ----               -------
  Warning  FailedScheduling  12s (x6 over 27s)  default-scheduler  0/4 nodes are available: 2 Insufficient cpu.
```

可能的原因包括

- 资源不足，集群内所有的 Node 都不满足该 Pod 请求的 CPU、内存、GPU 或者临时存储空间等资源。解决方法是删除集群内不用的 Pod 或者增加新的 Node。
- HostPort 端口已被占用，通常推荐使用 Service 对外开放服务端口



### Pod 一直处于 Waiting 或 ContainerCreating 状态

首先还是通过 `kubectl describe pod <pod-name>` 命令查看到当前 Pod 的事件

```sh
$ kubectl -n kube-system describe pod nginx-pod
Events:
  Type     Reason                 Age               From               Message
  ----     ------                 ----              ----               -------
  Normal   Scheduled              1m                default-scheduler  Successfully assigned nginx-pod to node1
  Normal   SuccessfulMountVolume  1m                kubelet, gpu13     MountVolume.SetUp succeeded for volume "config-volume"
  Normal   SuccessfulMountVolume  1m                kubelet, gpu13     MountVolume.SetUp succeeded for volume "coredns-token-sxdmc"
  Warning  FailedSync             2s (x4 over 46s)  kubelet, gpu13     Error syncing pod
  Normal   SandboxChanged         1s (x4 over 46s)  kubelet, gpu13     Pod sandbox changed, it will be killed and re-created.
```

可以发现，该 Pod 的 Sandbox 容器无法正常启动，具体原因需要查看 Kubelet 日志：

```sh
$ journalctl -u kubelet
...
Mar 14 04:22:04 node1 kubelet[29801]: E0314 04:22:04.649912   29801 cni.go:294] Error adding network: failed to set bridge addr: "cni0" already has an IP address different from 10.244.4.1/24
Mar 14 04:22:04 node1 kubelet[29801]: E0314 04:22:04.649941   29801 cni.go:243] Error while adding to cni network: failed to set bridge addr: "cni0" already has an IP address different from 10.244.4.1/24
Mar 14 04:22:04 node1 kubelet[29801]: W0314 04:22:04.891337   29801 cni.go:258] CNI failed to retrieve network namespace path: Cannot find network namespace for the terminated container "c4fd616cde0e7052c240173541b8543f746e75c17744872aa04fe06f52b5141c"
Mar 14 04:22:05 node1 kubelet[29801]: E0314 04:22:05.965801   29801 remote_runtime.go:91] RunPodSandbox from runtime service failed: rpc error: code = 2 desc = NetworkPlugin cni failed to set up pod "nginx-pod" network: failed to set bridge addr: "cni0" already has an IP address different from 10.244.4.1/24
```

发现是 cni0 网桥配置了一个不同网段的 IP 地址导致，删除该网桥（网络插件会自动重新创建）即可修复

```sh
$ ip link set cni0 down
$ brctl delbr cni0
```

除了以上错误，其他可能的原因还有

- 镜像拉取失败，比如
  - 配置了错误的镜像
  - Kubelet 无法访问镜像（国内环境访问 `gcr.io` 需要特殊处理）
  - 私有镜像的密钥配置错误
  - 镜像太大，拉取超时（可以适当调整 kubelet 的 `--image-pull-progress-deadline` 和 `--runtime-request-timeout` 选项）
- CNI 网络错误，一般需要检查 CNI 网络插件的配置，比如
  - 无法配置 Pod 网络
  - 无法分配 IP 地址
- 容器无法启动，需要检查是否打包了正确的镜像或者是否配置了正确的容器参数



### Pod 处于 ImagePullBackOff 状态

这通常是镜像名称配置错误或者私有镜像的密钥配置错误导致。这种情况可以使用 `docker pull <image>` 来验证镜像是否可以正常拉取。

```sh
$ kubectl describe pod mypod
...
Events:
  Type     Reason                 Age                From                                Message
  ----     ------                 ----               ----                                -------
  Normal   Scheduled              36s                default-scheduler                   Successfully assigned sh to k8s-agentpool1-38622806-0
  Normal   SuccessfulMountVolume  35s                kubelet, k8s-agentpool1-38622806-0  MountVolume.SetUp succeeded for volume "default-token-n4pn6"
  Normal   Pulling                17s (x2 over 33s)  kubelet, k8s-agentpool1-38622806-0  pulling image "a1pine"
  Warning  Failed                 14s (x2 over 29s)  kubelet, k8s-agentpool1-38622806-0  Failed to pull image "a1pine": rpc error: code = Unknown desc = Error response from daemon: repository a1pine not found: does not exist or no pull access
  Warning  Failed                 14s (x2 over 29s)  kubelet, k8s-agentpool1-38622806-0  Error: ErrImagePull
  Normal   SandboxChanged         4s (x7 over 28s)   kubelet, k8s-agentpool1-38622806-0  Pod sandbox changed, it will be killed and re-created.
  Normal   BackOff                4s (x5 over 25s)   kubelet, k8s-agentpool1-38622806-0  Back-off pulling image "a1pine"
  Warning  Failed                 1s (x6 over 25s)   kubelet, k8s-agentpool1-38622806-0  Error: ImagePullBackOff
```

如果是私有镜像，需要首先创建一个 docker-registry 类型的 Secret

```sh
$ kubectl create secret docker-registry my-secret --docker-server=DOCKER_REGISTRY_SERVER --docker-username=DOCKER_USER --docker-password=DOCKER_PASSWORD --docker-email=DOCKER_EMAIL
```

然后在容器中引用这个 Secret

```yaml
spec:
  containers:
  - name: private-reg-container
    image: <your-private-image>
  imagePullSecrets:
  - name: my-secret
```



### Pod 一直处于 CrashLoopBackOff 状态

CrashLoopBackOff 状态说明容器曾经启动了，但又异常退出了。此时 Pod 的 RestartCounts 通常是大于 0 的，可以先查看一下容器的日志

```sh
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl logs --previous <pod-name>
```

这里可以发现一些容器退出的原因，比如

- 容器进程退出
- 健康检查失败退出
- OOMKilled

```sh
$ kubectl describe pod mypod
...
Containers:
  sh:
    Container ID:  docker://3f7a2ee0e7e0e16c22090a25f9b6e42b5c06ec049405bc34d3aa183060eb4906
    Image:         alpine
    Image ID:      docker-pullable://alpine@sha256:7b848083f93822dd21b0a2f14a110bd99f6efb4b838d499df6d04a49d0debf8b
    Port:          <none>
    Host Port:     <none>
    State:          Terminated
      Reason:       OOMKilled
      Exit Code:    2
    Last State:     Terminated
      Reason:       OOMKilled
      Exit Code:    2
    Ready:          False
    Restart Count:  3
    Limits:
      cpu:     1
      memory:  1G
    Requests:
      cpu:        100m
      memory:     500M
...
```

如果此时如果还未发现线索，还可以到容器内执行命令来进一步查看退出原因

```sh
kubectl exec cassandra -- cat /var/log/cassandra/system.log
```

如果还是没有线索，那就需要 SSH 登录该 Pod 所在的 Node 上，查看 Kubelet 或者 Docker 的日志进一步排查了

```sh
# Query Node
kubectl get pod <pod-name> -o wide

# SSH to Node
ssh <username>@<node-name>
```



### Pod 处于 Error 状态

通常处于 Error 状态说明 Pod 启动过程中发生了错误。常见的原因包括

- 依赖的 ConfigMap、Secret 或者 PV 等不存在
- 请求的资源超过了管理员设置的限制，比如超过了 LimitRange 等
- 违反集群的安全策略，比如违反了 PodSecurityPolicy 等
- 容器无权操作集群内的资源，比如开启 RBAC 后，需要为 ServiceAccount 配置角色绑定



### Pod 处于 Terminating 或 Unknown 状态

从 v1.5 开始，Kubernetes 不会因为 Node 失联而删除其上正在运行的 Pod，而是将其标记为 Terminating 或 Unknown 状态。想要删除这些状态的 Pod 有三种方法：

- 从集群中删除该 Node。使用公有云时，kube-controller-manager 会在 VM 删除后自动删除对应的 Node。而在物理机部署的集群中，需要管理员手动删除 Node（如 `kubectl delete node <node-name>`。
- Node 恢复正常。Kubelet 会重新跟 kube-apiserver 通信确认这些 Pod 的期待状态，进而再决定删除或者继续运行这些 Pod。
- 用户强制删除。用户可以执行 `kubectl delete pods <pod> --grace-period=0 --force` 强制删除 Pod。除非明确知道 Pod 的确处于停止状态（比如 Node 所在 VM 或物理机已经关机），否则不建议使用该方法。特别是 StatefulSet 管理的 Pod，强制删除容易导致脑裂或者数据丢失等问题。

如果 Kubelet 是以 Docker 容器的形式运行的，此时 kubelet 日志中可能会发现[如下的错误](https://github.com/kubernetes/kubernetes/issues/51835)：

```sh
{"log":"I0926 19:59:07.162477   54420 kubelet.go:1894] SyncLoop (DELETE, \"api\"): \"billcenter-737844550-26z3w_meipu(30f3ffec-a29f-11e7-b693-246e9607517c)\"\n","stream":"stderr","time":"2017-09-26T11:59:07.162748656Z"}
{"log":"I0926 19:59:39.977126   54420 reconciler.go:186] operationExecutor.UnmountVolume started for volume \"default-token-6tpnm\" (UniqueName: \"kubernetes.io/secret/30f3ffec-a29f-11e7-b693-246e9607517c-default-token-6tpnm\") pod \"30f3ffec-a29f-11e7-b693-246e9607517c\" (UID: \"30f3ffec-a29f-11e7-b693-246e9607517c\") \n","stream":"stderr","time":"2017-09-26T11:59:39.977438174Z"}
{"log":"E0926 19:59:39.977461   54420 nestedpendingoperations.go:262] Operation for \"\\\"kubernetes.io/secret/30f3ffec-a29f-11e7-b693-246e9607517c-default-token-6tpnm\\\" (\\\"30f3ffec-a29f-11e7-b693-246e9607517c\\\")\" failed. No retries permitted until 2017-09-26 19:59:41.977419403 +0800 CST (durationBeforeRetry 2s). Error: UnmountVolume.TearDown failed for volume \"default-token-6tpnm\" (UniqueName: \"kubernetes.io/secret/30f3ffec-a29f-11e7-b693-246e9607517c-default-token-6tpnm\") pod \"30f3ffec-a29f-11e7-b693-246e9607517c\" (UID: \"30f3ffec-a29f-11e7-b693-246e9607517c\") : remove /var/lib/kubelet/pods/30f3ffec-a29f-11e7-b693-246e9607517c/volumes/kubernetes.io~secret/default-token-6tpnm: device or resource busy\n","stream":"stderr","time":"2017-09-26T11:59:39.977728079Z"}
```

如果是这种情况，则需要给 kubelet 容器设置 `--containerized` 参数并传入以下的存储卷

```sh
# 以使用 calico 网络插件为例
      -v /:/rootfs:ro,shared \
      -v /sys:/sys:ro \
      -v /dev:/dev:rw \
      -v /var/log:/var/log:rw \
      -v /run/calico/:/run/calico/:rw \
      -v /run/docker/:/run/docker/:rw \
      -v /run/docker.sock:/run/docker.sock:rw \
      -v /usr/lib/os-release:/etc/os-release \
      -v /usr/share/ca-certificates/:/etc/ssl/certs \
      -v /var/lib/docker/:/var/lib/docker:rw,shared \
      -v /var/lib/kubelet/:/var/lib/kubelet:rw,shared \
      -v /etc/kubernetes/ssl/:/etc/kubernetes/ssl/ \
      -v /etc/kubernetes/config/:/etc/kubernetes/config/ \
      -v /etc/cni/net.d/:/etc/cni/net.d/ \
      -v /opt/cni/bin/:/opt/cni/bin/ \
```

处于 `Terminating` 状态的 Pod 在 Kubelet 恢复正常运行后一般会自动删除。但有时也会出现无法删除的情况，并且通过 `kubectl delete pods <pod> --grace-period=0 --force` 也无法强制删除。此时一般是由于 `finalizers` 导致的，通过 `kubectl edit` 将 finalizers 删除即可解决。

```sh
"finalizers": [
  "foregroundDeletion"
]
```



### Pod 行为异常

这里所说的行为异常是指 Pod 没有按预期的行为执行，比如没有运行 podSpec 里面设置的命令行参数。这一般是 podSpec yaml 文件内容有误，可以尝试使用 `--validate` 参数重建容器，比如

```sh
kubectl delete pod mypod
kubectl create --validate -f mypod.yaml
```

也可以查看创建后的 podSpec 是否是对的，比如

```
kubectl get pod mypod -o yaml
```





### 修改静态 Pod 的 Manifest 后未自动重建

Kubelet 使用 inotify 机制检测 `/etc/kubernetes/manifests` 目录（可通过 Kubelet 的 `--pod-manifest-path` 选项指定）中静态 Pod 的变化，并在文件发生变化后重新创建相应的 Pod。但有时也会发生修改静态 Pod 的 Manifest 后未自动创建新 Pod 的情景，此时一个简单的修复方法是重启 Kubelet。



### Nginx 启动失败

Nginx 启动失败，错误消息是 `nginx: [emerg] socket() [::]:8000 failed (97: Address family not supported by protocol)`。这是由于服务器未开启 IPv6 导致的，解决方法有两种：

- 第一种方法，服务器开启 IPv6；
- 或者，第二种方法，删除或者注释掉 `/etc/nginx/conf.d/default.conf` 文件中的 `listen [::]:80 default_server;`。



### Namespace 一直处于 terminating 状态

Namespace 一直处于 terminating 状态，一般有两种原因：

- Namespace 中还有资源正在删除中
- Namespace 的 Finalizer 未正常清理

对第一个问题，可以执行下面的命令来查询所有的资源

```sh
kubectl api-resources --verbs=list --namespaced -o name | xargs -n 1 kubectl get --show-kind --ignore-not-found -n $NAMESPACE
```

而第二个问题则需要手动清理 Namespace 的 Finalizer 列表：

1) 使用 kubectl proxy：

```sh
kubectl proxy &

NAMESPACE="<ns-to-delete>"
kubectl get namespaces $NAMESPACE -o json | jq '.metadata.finalizers=[]' | jq '.spec.finalizers=[]' > /tmp/ns.json
curl -k -H "Content-Type: application/json" -X PUT --data-binary @/tmp/ns.json http://127.0.0.1:8001/api/v1/namespaces/$NAMESPACE/finalize
```

2) 使用 kubectl --raw：

```sh
NAMESPACE="<ns-to-delete>"
kubectl get namespaces $NAMESPACE -o json | jq '.metadata.finalizers=[]' | jq '.spec.finalizers=[]' | kubectl replace --raw /api/v1/namespaces/$NAMESPACE/finalize -f -
```





### Pod 排错图解

![img](D:\学习资料\笔记\k8s\k8s图\排错)





# 网络排错



本章主要介绍各种常见的网络问题以及排错方法，包括 Pod 访问异常、Service 访问异常以及网络安全策略异常等。

说到 Kubernetes 的网络，其实无非就是以下三种情况之一

- Pod 访问容器外部网络
- 从容器外部访问 Pod 网络
- Pod 之间相互访问

当然，以上每种情况还都分别包括本地访问和跨主机访问两种场景，并且一般情况下都是通过 Service 间接访问 Pod。

排查网络问题基本上也是从这几种情况出发，定位出具体的网络异常点，再进而寻找解决方法。网络异常可能的原因比较多，常见的有

- CNI 网络插件配置错误，导致多主机网络不通，比如
  - IP 网段与现有网络冲突
  - 插件使用了底层网络不支持的协议
  - 忘记开启 IP 转发等
    - `sysctl net.ipv4.ip_forward`
    - `sysctl net.bridge.bridge-nf-call-iptables`
- Pod 网络路由丢失，比如
  - kubenetes 要求网络中有 podCIDR 到主机 IP 地址的路由，这些路由如果没有正确配置会导致 Pod 网络通信等问题
  - 在公有云平台上，kube-controller-manager 会自动为所有 Node 配置路由，但如果配置不当（如认证授权失败、超出配额等），也有可能导致无法配置路由
- Service NodePort 和 health probe 端口冲突
  - 在 1.10.4 版本之前的集群中，多个不同的 Service 之间的 NodePort 和 health probe 端口有可能会有重合 （已经在 [kubernetes#64468](https://github.com/kubernetes/kubernetes/pull/64468) 修复）
- 主机内或者云平台的安全组、防火墙或者安全策略等阻止了 Pod 网络，比如
  - 非 Kubernetes 管理的 iptables 规则禁止了 Pod 网络
  - 公有云平台的安全组禁止了 Pod 网络（注意 Pod 网络有可能与 Node 网络不在同一个网段）
  - 交换机或者路由器的 ACL 禁止了 Pod 网络



### Flannel Pods 一直处于 Init:CrashLoopBackOff 状态

Flannel 网络插件非常容易部署，只要一条命令即可

```sh
$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

然而，部署完成后，Flannel Pod 有可能会碰到初始化失败的错误

```sh
$ kubectl -n kube-system get pod
NAME                            READY     STATUS                  RESTARTS   AGE
kube-flannel-ds-ckfdc           0/1       Init:CrashLoopBackOff   4          2m
kube-flannel-ds-jpp96           0/1       Init:CrashLoopBackOff   4          2m
```

查看日志会发现

```sh
$ kubectl -n kube-system logs kube-flannel-ds-jpp96 -c install-cni
cp: can't create '/etc/cni/net.d/10-flannel.conflist': Permission denied
```

这一般是由于 SELinux 开启导致的，关闭 SELinux 既可解决。有两种方法：

- 修改 `/etc/selinux/config` 文件方法：`SELINUX=disabled`
- 通过命令临时修改（重启会丢失）：`setenforce 0`



### Pod 无法分配 IP

Pod 一直处于 ContainerCreating 状态，查看事件发现网络插件无法为其分配 IP：

```sh
  Normal   SandboxChanged          5m (x74 over 8m)    kubelet, k8s-agentpool-66825246-0  Pod sandbox changed, it will be killed and re-created.
  Warning  FailedCreatePodSandBox  21s (x204 over 8m)  kubelet, k8s-agentpool-66825246-0  Failed create pod sandbox: rpc error: code = Unknown desc = NetworkPlugin cni failed to set up pod "deployment-azuredisk6-56d8dcb746-487td_default" network: Failed to allocate address: Failed to delegate: Failed to allocate address: No available addresses
```

查看网络插件的 IP 分配情况，进一步发现 IP 地址确实已经全部分配完，但真正处于 Running 状态的 Pod 数却很少：

```sh
# 详细路径取决于具体的网络插件，当使用 host-local IPAM 插件时，路径位于 /var/lib/cni/networks 下面
$ cd /var/lib/cni/networks/kubenet
$ ls -al|wc -l
258
$ docker ps | grep POD | wc -l
7
```

这有两种可能的原因

- 网络插件本身的问题，Pod 停止后其 IP 未释放
- Pod 重新创建的速度比 Kubelet 调用 CNI 插件回收网络（垃圾回收时删除已停止 Pod 前会先调用 CNI 清理网络）的速度快

对第一个问题，最好联系插件开发者询问修复方法或者临时性的解决方法。当然，如果对网络插件的工作原理很熟悉的话，也可以考虑手动释放未使用的 IP 地址，比如：

- 停止 Kubelet
- 找到 IPAM 插件保存已分配 IP 地址的文件，比如 `/var/lib/cni/networks/cbr0`（flannel）或者 `/var/run/azure-vnet-ipam.json`（Azure CNI）等
- 查询容器已用的 IP 地址，比如 `kubectl get pod -o wide --all-namespaces | grep <node-name>`
- 对比两个列表，从 IPAM 文件中删除未使用的 IP 地址，并手动删除相关的虚拟网卡和网络命名空间（如果有的话）
- 重启启动 Kubelet

```sh
# Take kubenet for example to delete the unused IPs
$ for hash in $(tail -n +1 * | grep '^[A-Za-z0-9]*$' | cut -c 1-8); do if [ -z $(docker ps -a | grep $hash | awk '{print $1}') ]; then grep -ilr $hash ./; fi; done | xargs rm
```

而第二个问题则可以给 Kubelet 配置更快的垃圾回收，如

```sh
--minimum-container-ttl-duration=15s
--maximum-dead-containers-per-container=1
--maximum-dead-containers=100
```



### Pod 无法解析 DNS

如果 Node 上安装的 Docker 版本大于 1.12，那么 Docker 会把默认的 iptables FORWARD 策略改为 DROP。这会引发 Pod 网络访问的问题。解决方法则在每个 Node 上面运行 `iptables -P FORWARD ACCEPT`，比如

```sh
echo "ExecStartPost=/sbin/iptables -P FORWARD ACCEPT" >> /etc/systemd/system/docker.service.d/exec_start.conf
systemctl daemon-reload
systemctl restart docker
```

如果使用了 flannel/weave 网络插件，更新为最新版本也可以解决这个问题。

除此之外，还有很多其他原因导致 DNS 无法解析：

（1）DNS 无法解析也有可能是 kube-dns 服务异常导致的，可以通过下面的命令来检查 kube-dns 是否处于正常运行状态

```sh
$ kubectl get pods --namespace=kube-system -l k8s-app=kube-dns
NAME                    READY     STATUS    RESTARTS   AGE
...
kube-dns-v19-ezo1y      3/3       Running   0           1h
...
```

如果 kube-dns 处于 CrashLoopBackOff 状态，那么可以参考 [Kube-dns/Dashboard CrashLoopBackOff 排错]() 来查看具体排错方法。

（2）如果 kube-dns Pod 处于正常 Running 状态，则需要进一步检查是否正确配置了 kube-dns 服务：

```sh
$ kubectl get svc kube-dns --namespace=kube-system
NAME          CLUSTER-IP     EXTERNAL-IP   PORT(S)             AGE
kube-dns      10.0.0.10      <none>        53/UDP,53/TCP        1h
$ kubectl get ep kube-dns --namespace=kube-system
NAME       ENDPOINTS                       AGE
kube-dns   10.180.3.17:53,10.180.3.17:53    1h
```

如果 kube-dns service 不存在，或者 endpoints 列表为空，则说明 kube-dns service 配置错误，可以重新创建 [kube-dns service](https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/dns)，比如

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: "KubeDNS"
spec:
  selector:
    k8s-app: kube-dns
  clusterIP: 10.0.0.10
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
```

（3）如果最近升级了 CoreDNS，并且使用了 CoreDNS 的 proxy 插件，那么需要注意 [1.5.0 以上](https://coredns.io/2019/04/06/coredns-1.5.0-release/)的版本需要将 proxy 插件替换为 forward 插件。比如：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-custom
  namespace: kube-system
data:
  test.server: |
    abc.com:53 {
        errors
        cache 30
        forward . 1.2.3.4
    }
    my.cluster.local:53 {
        errors
        cache 30
        forward . 2.3.4.5
    }
    azurestack.local:53 {
        forward . tls://1.1.1.1 tls://1.0.0.1 {
          tls_servername cloudflare-dns.com
          health_check 5s
        }
    }
```

（4）如果 kube-dns Pod 和 Service 都正常，那么就需要检查 kube-proxy 是否正确为 kube-dns 配置了负载均衡的 iptables 规则。具体排查方法可以参考下面的 Service 无法访问部分。



### DNS解析缓慢

由于内核的一个 [BUG](https://www.weave.works/blog/racy-conntrack-and-dns-lookup-timeouts)，连接跟踪模块会发生竞争，导致 DNS 解析缓慢。社区跟踪 Issue 为 https://github.com/kubernetes/kubernetes/issues/56903。

临时[解决方法](https://github.com/kubernetes/kubernetes/issues/56903)：为容器配置 `options single-request-reopen`，避免相同五元组的并发 DNS 请求：

```yaml
        lifecycle:
          postStart:
            exec:
              command:
              - /bin/sh
              - -c 
              - "/bin/echo 'options single-request-reopen' >> /etc/resolv.conf"
```

或者为 Pod 配置 dnsConfig：

```yaml
template:
  spec:
    dnsConfig:
      options:
        - name: single-request-reopen
```

> 注意：`single-request-reopen` 选项对 Alpine 无效，请使用 Debian 等其他的基础镜像，或者参考下面的修复方法。

修复方法：升级内核并保证包含以下两个补丁

1. [netfilter: nf_nat: skip nat clash resolution for same-origin entries](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=4e35c1cb9460240e983a01745b5f29fe3a4d8e39) (included since kernel v5.0)
2. [netfilter: nf_conntrack: resolve clash for matching conntracks](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ed07d9a021df6da53456663a76999189badc432a) (included since kernel v4.19)

> 对于 Azure 来说，这个问题已经在 [v4.15.0-1030.31/v4.18.0-1006.6](https://bugs.launchpad.net/ubuntu/+source/linux-azure/+bug/1795493) 中修复了（[patch1](https://git.launchpad.net/~canonical-kernel/ubuntu/+source/linux-azure/commit/?id=6f4fe585e573d7edd4122e45f58ca3da5b478265), [patch2](https://git.launchpad.net/~canonical-kernel/ubuntu/+source/linux-azure/commit/?id=4c7917876cf9560492d6bc2732365cbbfecfe623)）。

其他可能的原因和修复方法还有：

- Kube-dns 和 CoreDNS 同时存在时也会有问题，只保留一个即可。
- kube-dns 或者 CoreDNS 的资源限制太小时会导致 DNS 解析缓慢，这时候需要增大资源限制。
- 配置 DNS 选项 `use-vc`，强制使用 TCP 协议发送 DNS 查询。
- 在每个节点上运行一个 DNS 缓存服务，然后把所有容器的 DNS nameserver 都指向该缓存。

推荐部署 Nodelocal DNS Cache 扩展来解决这个问题，同时也提升 DNS 解析的性能。 Nodelocal DNS Cache 的部署步骤请参考 https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/dns/nodelocaldns。

> 更多 DNS 配置的方法可以参考 [Customizing DNS Service](https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/)。



### Service 无法访问

访问 Service ClusterIP 失败时，可以首先确认是否有对应的 Endpoints

```sh
$ kubectl get endpoints <service-name>
```

如果该列表为空，则有可能是该 Service 的 LabelSelector 配置错误，可以用下面的方法确认一下

```sh
# 查询 Service 的 LabelSelector
$ kubectl get svc <service-name> -o jsonpath='{.spec.selector}'
# 查询匹配 LabelSelector 的 Pod
$ kubectl get pods -l key1=value1,key2=value2
```

如果 Endpoints 正常，可以进一步检查

- Pod 的 containerPort 与 Service 的 containerPort 是否对应
- 直接访问 `podIP:containerPort` 是否正常

再进一步，即使上述配置都正确无误，还有其他的原因会导致 Service 无法访问，比如

- Pod 内的容器有可能未正常运行或者没有监听在指定的 containerPort 上
- CNI 网络或主机路由异常也会导致类似的问题
- kube-proxy 服务有可能未启动或者未正确配置相应的 iptables 规则，比如正常情况下名为 `hostnames` 的服务会配置以下 iptables 规则

```sh
$ iptables-save | grep hostnames
-A KUBE-SEP-57KPRZ3JQVENLNBR -s 10.244.3.6/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
-A KUBE-SEP-57KPRZ3JQVENLNBR -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.3.6:9376
-A KUBE-SEP-WNBA2IHDGP2BOBGZ -s 10.244.1.7/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
-A KUBE-SEP-WNBA2IHDGP2BOBGZ -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.1.7:9376
-A KUBE-SEP-X3P2623AGDH6CDF3 -s 10.244.2.3/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
-A KUBE-SEP-X3P2623AGDH6CDF3 -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.2.3:9376
-A KUBE-SERVICES -d 10.0.1.175/32 -p tcp -m comment --comment "default/hostnames: cluster IP" -m tcp --dport 80 -j KUBE-SVC-NWV5X2332I4OT4T3
-A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-WNBA2IHDGP2BOBGZ
-A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-X3P2623AGDH6CDF3
-A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -j KUBE-SEP-57KPRZ3JQVENLNBR
```



### Pod 无法通过 Service 访问自己

这通常是 hairpin 配置错误导致的，可以通过 Kubelet 的 `--hairpin-mode` 选项配置，可选参数包括 "promiscuous-bridge"、"hairpin-veth" 和 "none"（默认为"promiscuous-bridge"）。

对于 hairpin-veth 模式，可以通过以下命令来确认是否生效

```sh
$ for intf in /sys/devices/virtual/net/cbr0/brif/*; do cat $intf/hairpin_mode; done
1
1
1
1
```

而对于 promiscuous-bridge 模式，可以通过以下命令来确认是否生效

```sh
$ ifconfig cbr0 |grep PROMISC
UP BROADCAST RUNNING PROMISC MULTICAST  MTU:1460  Metric:1
```



### 无法访问 Kubernetes API

很多扩展服务需要访问 Kubernetes API 查询需要的数据（比如 kube-dns、Operator 等）。通常在 Kubernetes API 无法访问时，可以首先通过下面的命令验证 Kubernetes API 是正常的：

```sh
$ kubectl run curl  --image=appropriate/curl -i -t  --restart=Never --command -- sh
If you don't see a command prompt, try pressing enter.
/ #
/ # KUBE_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
/ # curl -sSk -H "Authorization: Bearer $KUBE_TOKEN" https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT/api/v1/namespaces/default/pods
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "selfLink": "/api/v1/namespaces/default/pods",
    "resourceVersion": "2285"
  },
  "items": [
   ...
  ]
 }
```

如果出现超时错误，则需要进一步确认名为 `kubernetes` 的服务以及 endpoints 列表是正常的：

```sh
$ kubectl get service kubernetes
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   25m
$ kubectl get endpoints kubernetes
NAME         ENDPOINTS          AGE
kubernetes   172.17.0.62:6443   25m
```

然后可以直接访问 endpoints 查看 kube-apiserver 是否可以正常访问。无法访问时通常说明 kube-apiserver 未正常启动，或者有防火墙规则阻止了访问。

但如果出现了 `403 - Forbidden` 错误，则说明 Kubernetes 集群开启了访问授权控制（如 RBAC），此时就需要给 Pod 所用的 ServiceAccount 创建角色和角色绑定授权访问所需要的资源。比如 CoreDNS 就需要创建以下 ServiceAccount 以及角色绑定：

```yaml
# 1. service account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: coredns
  namespace: kube-system
  labels:
      kubernetes.io/cluster-service: "true"
      addonmanager.kubernetes.io/mode: Reconcile
---
# 2. cluster role
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
    addonmanager.kubernetes.io/mode: Reconcile
  name: system:coredns
rules:
- apiGroups:
  - ""
  resources:
  - endpoints
  - services
  - pods
  - namespaces
  verbs:
  - list
  - watch
---
# 3. cluster role binding
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
    addonmanager.kubernetes.io/mode: EnsureExists
  name: system:coredns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:coredns
subjects:
- kind: ServiceAccount
  name: coredns
  namespace: kube-system
---
# 4. use created service account
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: coredns
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/name: "CoreDNS"
spec:
  replicas: 2
  selector:
    matchLabels:
      k8s-app: coredns
  template:
    metadata:
      labels:
        k8s-app: coredns
    spec:
      serviceAccountName: coredns
      ...
```



### 内核导致的问题

除了以上问题，还有可能碰到因内核问题导致的服务无法访问或者服务访问超时的错误，比如

- [未设置 `--random-fully` 导致无法为 SNAT 分配端口，进而会导致服务访问超时](https://tech.xing.com/a-reason-for-unexplained-connection-timeouts-on-kubernetes-docker-abd041cf7e02)。注意， Kubernetes 暂时没有为 SNAT 设置 `--random-fully` 选项，如果碰到这个问题可以参考[这里](https://gist.github.com/maxlaverse/1fb3bfdd2509e317194280f530158c98) 配置。



### 参考文档

- [Troubleshoot Applications](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-application/)
- [Debug Services](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-service/)





# PV 排错



本章介绍持久化存储异常（PV、PVC、StorageClass等）的排错方法。

一般来说，无论 PV 处于什么异常状态，都可以执行 `kubectl describe pv/pvc <pod-name>` 命令来查看当前 PV 的事件。这些事件通常都会有助于排查 PV 或 PVC 发生的问题。

```sh
$ kubectl get pv
$ kubectl get pvc
$ kubectl get sc
$ kubectl describe pv <pv-name>
$ kubectl describe pvc <pvc-name>
$ kubectl describe sc <storage-class-name>
```





## AzureDisk



[AzureDisk](https://docs.microsoft.com/zh-cn/azure/virtual-machines/windows/about-disks-and-vhds) 为 Azure 上面运行的虚拟机提供了弹性块存储服务，它以 VHD 的形式挂载到虚拟机中，并可以在 Kubernetes 容器中使用。AzureDisk 优点是性能高，特别是 Premium Storage 提供了非常好的[性能](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/premium-storage)；其缺点是不支持共享，只可以用在单个 Pod 内。

根据配置的不同，Kubernetes 支持的 AzureDisk 可以分为以下几类

- Managed Disks: 由 Azure 自动管理磁盘和存储账户
- Blob Disks:
  - Dedicated (默认)：为每个 AzureDisk 创建单独的存储账户，当删除 PVC 的时候删除该存储账户
  - Shared：AzureDisk 共享 ResourceGroup 内的同一个存储账户，这时删除 PVC 不会删除该存储账户

> 注意：
>
> - AzureDisk 的类型必须跟 VM OS Disk 类型一致，即要么都是 Manged Disks，要么都是 Blob Disks。当两者不一致时，AzureDisk PV 会报无法挂载的错误。
> - 由于 Managed Disks 需要创建和管理存储账户，其创建过程会比 Blob Disks 慢（3 分钟 vs 1-2 分钟）。
> - 但节点最大支持同时挂载 16 个 AzureDisk。

使用 AzureDisk 推荐的版本：

| Kubernetes version | Recommended version |
| ------------------ | ------------------- |
| 1.12               | 1.12.9 或更高版本   |
| 1.13               | 1.13.6 或更高版本   |
| 1.14               | 1.14.2 或更高版本   |
| >=1.15             | >=1.15              |

使用 [aks-engine](https://github.com/Azure/aks-engine) 部署的 Kubernetes 集群，会自动创建两个 StorageClass，默认为managed-standard（即HDD）：

```sh
$ kubectl get storageclass
NAME                PROVISIONER                AGE
default (default)   kubernetes.io/azure-disk   45d
managed-premium     kubernetes.io/azure-disk   53d
managed-standard    kubernetes.io/azure-disk   53d
```



### AzureDisk 挂载失败

在 AzureDisk 从一个 Pod 迁移到另一 Node 上面的 Pod 时或者同一台 Node 上面使用了多块 AzureDisk 时有可能会碰到这个问题。这是由于 kube-controller-manager 未对 AttachDisk 和 DetachDisk 操作加锁从而引发了竞争问题（[kubernetes#60101](https://github.com/kubernetes/kubernetes/issues/60101) [acs-engine#2002](https://github.com/Azure/acs-engine/issues/2002) [ACS#12](https://github.com/Azure/ACS/issues/12)）。

通过 kube-controller-manager 的日志，可以查看具体的错误原因。常见的错误日志为

```sh
Cannot attach data disk 'cdb-dynamic-pvc-92972088-11b9-11e8-888f-000d3a018174' to VM 'kn-edge-0' because the disk is currently being detached or the last detach operation failed. Please wait until the disk is completely detached and then try again or delete/detach the disk explicitly again.
```

临时性解决方法为

（1）更新所有受影响的虚拟机状态

使用powershell：

```sh
$vm = Get-AzureRMVM -ResourceGroupName $rg -Name $vmname
Update-AzureRmVM -ResourceGroupName $rg -VM $vm -verbose -debug
```

使用 Azure CLI：

```sh
# For VM:
az vm update -n <VM_NAME> -g <RESOURCE_GROUP_NAME>
# For VMSS:
az vmss update-instances -g <RESOURCE_GROUP_NAME> --name <VMSS_NAME> --instance-id <ID>
```

（2）重启虚拟机

- `kubectl cordon NODE`
- 如果 Node 上运行有 StatefulSet，需要手动删除相应的 Pod
- `kubectl drain NODE`
- `Get-AzureRMVM -ResourceGroupName $rg -Name $vmname | Restart-AzureVM`
- `kubectl uncordon NODE`

该问题的修复 [#60183](https://github.com/kubernetes/kubernetes/pull/60183) 已包含在 v1.10 中。



### 挂载新的 AzureDisk 后，该 Node 中其他 Pod 已挂载的 AzureDisk 不可用

在 Kubernetes v1.7 中，AzureDisk 默认的缓存策略修改为 `ReadWrite`，这会导致在同一个 Node 中挂载超过 5 块 AzureDisk 时，已有 AzureDisk 的盘符会随机改变（[kubernetes#60344](https://github.com/kubernetes/kubernetes/issues/60344) [kubernetes#57444](https://github.com/kubernetes/kubernetes/issues/57444) [AKS#201](https://github.com/Azure/AKS/issues/201) [acs-engine#1918](https://github.com/Azure/acs-engine/issues/1918)）。比如，当挂载第六块 AzureDisk 后，原来 lun0 磁盘的挂载盘符有可能从 `sdc` 变成 `sdk`：

```sh
$ tree /dev/disk/azure
...
â””â”€â”€ scsi1
    â”œâ”€â”€ lun0 -> ../../../sdk
    â”œâ”€â”€ lun1 -> ../../../sdj
    â”œâ”€â”€ lun2 -> ../../../sde
    â”œâ”€â”€ lun3 -> ../../../sdf
    â”œâ”€â”€ lun4 -> ../../../sdg
    â”œâ”€â”€ lun5 -> ../../../sdh
    â””â”€â”€ lun6 -> ../../../sdi
```

这样，原来使用 lun0 磁盘的 Pod 就无法访问 AzureDisk 了

```sh
[root@admin-0 /]# ls /datadisk
ls: reading directory .: Input/output error
```

临时性解决方法是设置 AzureDisk StorageClass 的 `cachingmode: None`，如

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: managed-standard
provisioner: kubernetes.io/azure-disk
parameters:
  skuname: Standard_LRS
  kind: Managed
  cachingmode: None
```

该问题的修复 [#60346](https://github.com/kubernetes/kubernetes/pull/60346) 将包含在 v1.10 中。



### AzureDisk 挂载慢

AzureDisk PVC 的挂载过程一般需要 1 分钟的时间，这些时间主要消耗在 Azure ARM API 的调用上（查询 VM 以及挂载 Disk）。[#57432](https://github.com/kubernetes/kubernetes/pull/57432) 为 Azure VM 增加了一个缓存，消除了 VM 的查询时间，将整个挂载过程缩短到大约 30 秒。该修复包含在v1.9.2+ 和 v1.10 中。

另外，如果 Node 使用了 `Standard_B1s` 类型的虚拟机，那么 AzureDisk 的第一次挂载一般会超时，等再次重复时才会挂载成功。这是因为在 `Standard_B1s` 虚拟机中格式化 AzureDisk 就需要很长时间（如超过 70 秒）。

```sh
$ kubectl describe pod <pod-name>
...
Events:
  FirstSeen     LastSeen        Count   From                                    SubObjectPath                           Type            Reason                  Message
  ---------     --------        -----   ----                                    -------------                           --------        ------                  -------
  8m            8m              1       default-scheduler                                                               Normal          Scheduled               Successfully assigned nginx-azuredisk to aks-nodepool1-15012548-0
  7m            7m              1       kubelet, aks-nodepool1-15012548-0                                               Normal          SuccessfulMountVolume   MountVolume.SetUp succeeded for volume "default-token-mrw8h"
  5m            5m              1       kubelet, aks-nodepool1-15012548-0                                               Warning         FailedMount             Unable to mount volumes for pod "nginx-azuredisk_default(4eb22bb2-0bb5-11e8-8
d9e-0a58ac1f0a2e)": timeout expired waiting for volumes to attach/mount for pod "default"/"nginx-azuredisk". list of unattached/unmounted volumes=[disk01]
  5m            5m              1       kubelet, aks-nodepool1-15012548-0                                               Warning         FailedSync              Error syncing pod
  4m            4m              1       kubelet, aks-nodepool1-15012548-0                                               Normal          SuccessfulMountVolume   MountVolume.SetUp succeeded for volume "pvc-20240841-0bb5-11e8-8d9e-0a58ac1f0
a2e"
  4m            4m              1       kubelet, aks-nodepool1-15012548-0       spec.containers{nginx-azuredisk}        Normal          Pulling                 pulling image "nginx"
  3m            3m              1       kubelet, aks-nodepool1-15012548-0       spec.containers{nginx-azuredisk}        Normal          Pulled                  Successfully pulled image "nginx"
  3m            3m              1       kubelet, aks-nodepool1-15012548-0       spec.containers{nginx-azuredisk}        Normal          Created                 Created container
  2m            2m              1       kubelet, aks-nodepool1-15012548-0       spec.containers{nginx-azuredisk}        Normal          Started                 Started container
```



### Azure German Cloud 无法使用 AzureDisk

Azure German Cloud 仅在 v1.7.9+、v1.8.3+ 以及更新版本中支持（[#50673](https://github.com/kubernetes/kubernetes/pull/50673)），升级 Kubernetes 版本即可解决。



### MountVolume.WaitForAttach failed

```sh
MountVolume.WaitForAttach failed for volume "pvc-f1562ecb-3e5f-11e8-ab6b-000d3af9f967" : azureDisk - Wait for attach expect device path as a lun number, instead got: /dev/disk/azure/scsi1/lun1 (strconv.Atoi: parsing "/dev/disk/azure/scsi1/lun1": invalid syntax)
```

[该问题](https://github.com/kubernetes/kubernetes/issues/62540) 仅在 Kubernetes v1.10.0 和 v1.10.1 中存在，将在 v1.10.2 中修复。



### 在 mountOptions 中设置 uid 和 gid 时失败

默认情况下，Azure 磁盘使用 ext4、xfs filesystem 和 mountOptions，如 uid = x，gid = x 无法在装入时设置。 例如，如果你尝试设置 mountOptions uid = 999，gid = 999，将看到类似于以下内容的错误:

```sh
Warning  FailedMount             63s                  kubelet, aks-nodepool1-29460110-0  MountVolume.MountDevice failed for volume "pvc-d783d0e4-85a1-11e9-8a90-369885447933" : azureDisk - mountDevice:FormatAndMount failed with mount failed: exit status 32
Mounting command: systemd-run
Mounting arguments: --description=Kubernetes transient mount for /var/lib/kubelet/plugins/kubernetes.io/azure-disk/mounts/m436970985 --scope -- mount -t xfs -o dir_mode=0777,file_mode=0777,uid=1000,gid=1000,defaults /dev/disk/azure/scsi1/lun2 /var/lib/kubelet/plugins/kubernetes.io/azure-disk/mounts/m436970985
Output: Running scope as unit run-rb21966413ab449b3a242ae9b0fbc9398.scope.
mount: wrong fs type, bad option, bad superblock on /dev/sde,
       missing codepage or helper program, or other error
```

可以通过执行以下操作之一来缓解此问题

- 通过在 fsGroup 中的 runAsUser 和 gid 中设置 uid 来[配置 pod 的安全上下文](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)。例如，以下设置会将 pod 设置为 root，使其可供任何文件访问：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  securityContext:
    runAsUser: 0
    fsGroup: 0
```

> 备注: 因为 gid 和 uid 默认装载为 root 或0。如果 gid 或 uid 设置为非根（例如1000），则 Kubernetes 将使用 `chown` 更改该磁盘下的所有目录和文件。此操作可能非常耗时，并且可能会导致装载磁盘的速度非常慢。

- 使用 initContainers 中的 `chown` 设置 gid 和 uid。例如：

```yaml
initContainers:
- name: volume-mount
  image: busybox
  command: ["sh", "-c", "chown -R 100:100 /data"]
  volumeMounts:
  - name: <your data volume>
    mountPath: /data
```



#### 删除 pod 使用的 Azure 磁盘 PersistentVolumeClaim 时出错

如果尝试删除 pod 使用的 Azure 磁盘 PersistentVolumeClaim，可能会看到错误。例如：

```sh
$ kubectl describe pv pvc-d8eebc1d-74d3-11e8-902b-e22b71bb1c06
...
Message:         disk.DisksClient#Delete: Failure responding to request: StatusCode=409 -- Original Error: autorest/azure: Service returned an error. Status=409 Code="OperationNotAllowed" Message="Disk kubernetes-dynamic-pvc-d8eebc1d-74d3-11e8-902b-e22b71bb1c06 is attached to VM /subscriptions/{subs-id}/resourceGroups/MC_markito-aks-pvc_markito-aks-pvc_westus/providers/Microsoft.Compute/virtualMachines/aks-agentpool-25259074-0."
```

在 Kubernetes 版本1.10 及更高版本中，默认情况下已启用 PersistentVolumeClaim protection 功能以防止此错误。如果你使用的 Kubernetes 版本不能解决此问题，则可以通过在删除 PersistentVolumeClaim 前使用 PersistentVolumeClaim 删除 pod 来缓解此问题。



### 参考文档

- [Known kubernetes issues on Azure](https://github.com/andyzhangx/demo/tree/master/issues)
- [Introduction of AzureDisk](https://docs.microsoft.com/zh-cn/azure/virtual-machines/windows/about-disks-and-vhds)
- [AzureDisk volume examples](https://github.com/kubernetes/examples/tree/master/staging/volumes/azure_disk)
- [High-performance Premium Storage and managed disks for VMs](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/premium-storage)





## AzureFile



[AzureFile](https://docs.microsoft.com/zh-cn/azure/storage/files/storage-files-introduction) 提供了基于 SMB 协议（也称 CIFS）托管文件共享服务。它支持 Windows 和 Linux 容器，并支持跨主机的共享，可用于多个 Pod 之间的共享存储。AzureFile 的缺点是性能[较差](https://docs.microsoft.com/en-us/azure/storage/files/storage-files-scale-targets)（[AKS#223](https://github.com/Azure/AKS/issues/223)），并且不提供 Premium 存储。

推荐基于 StorageClass 来使用 AzureFile，即

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: azurefile
provisioner: kubernetes.io/azure-file
mountOptions:
  - dir_mode=0777
  - file_mode=0777
  - uid=1000
  - gid=1000
parameters:
  skuName: Standard_LRS
```

使用 AzureFile 推荐的版本：

| Kubernetes version | Recommended version |
| ------------------ | ------------------- |
| 1.12               | 1.12.6 或更高版本   |
| 1.13               | 1.13.4 或更高版本   |
| 1.14               | 1.14.0 或更高版本   |
| >=1.15             | >=1.15              |



### 访问权限

AzureFile 使用 [mount.cifs](https://linux.die.net/man/8/mount.cifs) 将其远端存储挂载到 Node 上，而`fileMode` 和 `dirMode` 控制了挂载后文件和目录的访问权限。不同的 Kubernetes 版本，`fileMode` 和 `dirMode` 的默认选项是不同的

| Kubernetes 版本 | fileMode和dirMode |
| --------------- | ----------------- |
| v1.6.x, v1.7.x  | 0777              |
| v1.8.0-v1.8.5   | 0700              |
| v1.8.6 or above | 0755              |
| v1.9.0          | 0700              |
| v1.9.1-v1.12.1  | 0755              |
| >=v1.12.2       | 0777              |

按照默认的权限会导致非跟用户无法在目录中创建新的文件，解决方法为

- v1.8.0-v1.8.5：设置容器以 root 用户运行，如设置 `spec.securityContext.runAsUser: 0`
- v1.8.6 以及更新版本：在 AzureFile StorageClass 通过 mountOptions 设置默认权限，比如设置为 `0777` 的方法为

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: azurefile
provisioner: kubernetes.io/azure-file
mountOptions:
  - dir_mode=0777
  - file_mode=0777
  - uid=1000
  - gid=1000
  - mfsymlinks
  - nobrl
  - cache=none
parameters:
  skuName: Standard_LRS
```



### Windows Node 重启后无法访问 AzureFile

Windows Node 重启后，挂载 AzureFile 的 Pod 可以看到如下错误（[#60624](https://github.com/kubernetes/kubernetes/issues/60624)）：

```sh
Warning  Failed                 1m (x7 over 1m)  kubelet, 77890k8s9010  Error: Error response from daemon: invalid bind mount spec "c:\\var\\lib\\kubelet\\pods\\07251c5c-1cfc-11e8-8f70-000d3afd4b43\\volumes\\kubernetes.io~azure-file\\pvc-fb6159f6-1cfb-11e8-8f70-000d3afd4b43:c:/mnt/azure": invalid volume specification: 'c:\var\lib\kubelet\pods\07251c5c-1cfc-11e8-8f70-000d3afd4b43\volumes\kubernetes.io~azure-file\pvc-fb6159f6-1cfb-11e8-8f70-000d3afd4b43:c:/mnt/azure': invalid mount config for type "bind": bind source path does not exist
  Normal   SandboxChanged         1m (x8 over 1m)  kubelet, 77890k8s9010  Pod sandbox changed, it will be killed and re-created.
```

临时性解决方法为删除并重新创建使用了 AzureFile 的 Pod。当 Pod 使用控制器（如 Deployment、StatefulSet等）时，删除 Pod 后控制器会自动创建一个新的 Pod。

该问题的修复 [#60625](https://github.com/kubernetes/kubernetes/pull/60625) 包含在 v1.10 中。



### AzureFile ProvisioningFailed

Azure 文件共享的名字最大只允许 63 个字节，因而在集群名字较长的集群（Kubernetes v1.7.10 或者更老的集群）里面有可能会碰到 AzureFile 名字长度超限的情况，导致 AzureFile ProvisioningFailed：

```
persistentvolume-controller    Warning    ProvisioningFailed Failed to provision volume with StorageClass "azurefile": failed to find a matching storage account
```

碰到该问题时可以通过升级集群解决，其修复 [#48326](https://github.com/kubernetes/kubernetes/pull/48326) 已经包含在 v1.7.11、v1.8 以及更新版本中。

在开启 RBAC 的集群中，由于 AzureFile 需要访问 Secret，而 kube-controller-manager 中并未为 AzureFile 自动授权，从而也会导致 ProvisioningFailed：

```
Events:
  Type     Reason              Age   From                         Message
  ----     ------              ----  ----                         -------
  Warning  ProvisioningFailed  8s    persistentvolume-controller  Failed to provision volume with StorageClass "azurefile": Couldn't create secret secrets is forbidden: User "system:serviceaccount:kube-syste
m:persistent-volume-binder" cannot create secrets in the namespace "default"
  Warning  ProvisioningFailed  8s    persistentvolume-controller  Failed to provision volume with StorageClass "azurefile": failed to find a matching storage account
```

解决方法是为 ServiceAccount `persistent-volume-binder` 授予 Secret 的访问权限：

```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: system:azure-cloud-provider
rules:
- apiGroups: ['']
  resources: ['secrets']
  verbs:     ['get','create']
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: system:azure-cloud-provider
roleRef:
  kind: ClusterRole
  apiGroup: rbac.authorization.k8s.io
  name: system:azure-cloud-provider
subjects:
- kind: ServiceAccount
  name: persistent-volume-binder
  namespace: kube-system
```



### Azure German Cloud 无法使用 AzureFile

Azure German Cloud 仅在 v1.7.11+、v1.8+ 以及更新版本中支持（[#48460](https://github.com/kubernetes/kubernetes/pull/48460)），升级 Kubernetes 版本即可解决。



### "could not change permissions" 错误

在 Azure Files 插件上运行 PostgreSQL 时，可能会看到类似于以下内容的错误：

```
initdb: could not change permissions of directory "/var/lib/postgresql/data": Operation not permitted
fixing permissions on existing directory /var/lib/postgresql/data
```

此错误是由使用 cifs/SMB 协议的 Azure 文件插件导致的。 使用 cifs/SMB 协议时，无法在装载后更改文件和目录权限。 若要解决此问题，请将子路径与 Azure 磁盘插件结合使用。



### 参考文档

- [Known kubernetes issues on Azure](https://github.com/andyzhangx/demo/tree/master/issues)
- [Introduction of Azure File Storage](https://docs.microsoft.com/zh-cn/azure/storage/files/storage-files-introduction)
- [AzureFile volume examples](https://github.com/kubernetes/examples/tree/master/staging/volumes/azure_file)
- [Persistent volumes with Azure files](https://docs.microsoft.com/en-us/azure/aks/azure-files-dynamic-pv)
- [Azure Files scalability and performance targets](https://docs.microsoft.com/en-us/azure/storage/files/storage-files-scale-targets)





# 云平台排错



本章主要介绍在公有云中运行 Kubernetes 时可能会碰到的问题以及解决方法。

在公有云平台上运行 Kubernetes，一般可以使用云平台提供的托管 Kubernetes 服务（比如 Google 的 GKE、微软 Azure 的 AKS 或者 AWS 的 Amazon EKS 等）。当然，为了更自由的灵活性，也可以直接在这些公有云平台的虚拟机中部署 Kubernetes。无论哪种方法，一般都需要给 Kubernetes 配置 Cloud Provider 选项，以方便直接利用云平台提供的高级网络、持久化存储以及安全控制等功能。

而在云平台中运行 Kubernetes 的常见问题有

- 认证授权问题：比如 Kubernetes Cloud Provider 中配置的认证方式无权操作虚拟机所在的网络或持久化存储。这一般从 kube-controller-manager 的日志中很容易发现。
- 网络路由配置失败：正常情况下，Cloud Provider 会为每个 Node 配置一条 PodCIDR 至 NodeIP 的路由规则，如果这些规则有问题就会导致多主机 Pod 相互访问的问题。
- 公网 IP 分配失败：比如 LoadBalancer 类型的 Service 无法分配公网 IP 或者指定的公网 IP 无法使用。这一版也是配置错误导致的。
- 安全组配置失败：比如无法为 Service 创建安全组（如超出配额等）或与已有的安全组冲突等。
- 持久化存储分配或者挂载问题：比如分配 PV 失败（如超出配额、配置错误等）或挂载到虚拟机失败（比如 PV 正被其他异常 Pod 引用而导致无法从旧的虚拟机中卸载）。
- 网络插件使用不当：比如网络插件使用了云平台不支持的网络协议等。



### Node 未注册到集群中

通常，在 Kubelet 启动时会自动将自己注册到 kubernetes API 中，然后通过 `kubectl get nodes` 就可以查询到该节点。 如果新的 Node 没有自动注册到 Kubernetes 集群中，那说明这个注册过程有错误发生，需要检查 kubelet 和 kube-controller-manager 的日志，进而再根据日志查找具体的错误原因。



### Kubelet 日志

查看 Kubelet 日志需要首先 SSH 登录到 Node 上，然后运行 `journalctl` 命令查看 kubelet 的日志：

```sh
$ journalctl -l -u kubelet
```



### kube-controller-manager 日志

kube-controller-manager 会自动在云平台中给 Node 创建路由，如果路由创建创建失败也有可能导致 Node 注册失败。

```sh
$ PODNAME=$(kubectl -n kube-system get pod -l component=kube-controller-manager -o jsonpath='{.items[0].metadata.name}')
$ kubectl -n kube-system logs $PODNAME --tail 100
```







# 排错工具



本章主要介绍在 Kubernetes 排错中常用的工具。



### 必备工具

- `kubectl`：用于查看 Kubernetes 集群以及容器的状态，如 `kubectl describe pod <pod-name>`
- `journalctl`：用于查看 Kubernetes 组件日志，如 `journalctl -u kubelet -l`
- `iptables`和`ebtables`：用于排查 Service 是否工作，如 `iptables -t nat -nL` 查看 kube-proxy 配置的 iptables 规则是否正常
- `tcpdump`：用于排查容器网络问题，如 `tcpdump -nn host 10.240.0.8`
- `perf`：Linux 内核自带的性能分析工具，常用来排查性能问题，如 [Container Isolation Gone Wrong](https://dzone.com/articles/container-isolation-gone-wrong) 问题的排查



### kubectl-node-shell

查看 Kubelet、CNI、kernel 等系统组件的日志需要首先 SSH 登录到 Node 上，推荐使用 [kubectl-node-shell](https://github.com/kvaps/kubectl-node-shell) 插件而不是为每个节点分配公网 IP 地址。比如：

```sh
$ curl -LO https://github.com/kvaps/kubectl-node-shell/raw/master/kubectl-node_shell
$ chmod +x ./kubectl-node_shell
$ sudo mv ./kubectl-node_shell /usr/local/bin/kubectl-node_shell
$ kubectl node-shell <node>
$ journalctl -l -u kubelet
```



### sysdig

sysdig 是一个容器排错工具，提供了开源和商业版本。对于常规排错来说，使用开源版本即可。

除了 sysdig，还可以使用其他两个辅助工具

- csysdig：与 sysdig 一起自动安装，提供了一个命令行界面
- [sysdig-inspect](https://github.com/draios/sysdig-inspect)：为 sysdig 保存的跟踪文件（如 `sudo sysdig -w filename.scap`）提供了一个图形界面（非实时）



#### 安装

```sh
# on Ubuntu
$ curl -s https://s3.amazonaws.com/download.draios.com/DRAIOS-GPG-KEY.public | apt-key add -
$ curl -s -o /etc/apt/sources.list.d/draios.list http://download.draios.com/stable/deb/draios.list
$ apt-get update
$ apt-get -y install linux-headers-$(uname -r)apt-get -y install sysdig

# on REHL
$ rpm --import https://s3.amazonaws.com/download.draios.com/DRAIOS-GPG-KEY.public
$ curl -s -o /etc/yum.repos.d/draios.repo http://download.draios.com/stable/rpm/draios.repo
$ rpm -i http://mirror.us.leaseweb.net/epel/6/i386/epel-release-6-8.noarch.rpm
$ yum -y install kernel-devel-$(uname -r)
$ yum -y install sysdig

# on MacOS
$ brew install sysdig
```



#### 示例

```sh
# Refer https://www.sysdig.org/wiki/sysdig-examples/.
# View the top network connections
$ sudo sysdig -pc -c topconns

# View the top network connections inside the wordpress1 container
$ sudo sysdig -pc -c topconns container.name=wordpress1

# Show the network data exchanged with the host 192.168.0.1
$ sudo sysdig fd.ip=192.168.0.1
$ sudo sysdig -s2000 -A -c echo_fds fd.cip=192.168.0.1

# List all the incoming connections that are not served by apache.
$ sudo sysdig -p"%proc.name %fd.name" "evt.type=accept and proc.name!=httpd"

# View the CPU/Network/IO usage of the processes running inside the container.
$ sudo sysdig -pc -c topprocs_cpu container.id=2e854c4525b8
$ sudo sysdig -pc -c topprocs_net container.id=2e854c4525b8
$ sudo sysdig -pc -c topfiles_bytes container.id=2e854c4525b8

# See the files where apache spends the most time doing I/O
$ sudo sysdig -c topfiles_time proc.name=httpd

# Show all the interactive commands executed inside a given container.
$ sudo sysdig -pc -c spy_users 

# Show every time a file is opened under /etc.
$ sudo sysdig evt.type=open and fd.name

# View the list of processes with container context
$ sudo csysdig -pc
```

更多示例和使用方法可以参考 [Sysdig User Guide](https://github.com/draios/sysdig/wiki/Sysdig-User-Guide)。



### Weave Scope

Weave Scope 是另外一款可视化容器监控和排错工具。与 sysdig 相比，它没有强大的命令行工具，但提供了一个简单易用的交互界面，自动描绘了整个集群的拓扑，并可以通过插件扩展其功能。从其官网的介绍来看，其提供的功能包括

- [交互式拓扑界面](https://www.weave.works/docs/scope/latest/features/#topology-mapping)
- [图形模式和表格模式](https://www.weave.works/docs/scope/latest/features/#mode)
- [过滤功能](https://www.weave.works/docs/scope/latest/features/#flexible-filtering)
- [搜索功能](https://www.weave.works/docs/scope/latest/features/#powerful-search)
- [实时度量](https://www.weave.works/docs/scope/latest/features/#real-time-app-and-container-metrics)
- [容器排错](https://www.weave.works/docs/scope/latest/features/#interact-with-and-manage-containers)
- [插件扩展](https://www.weave.works/docs/scope/latest/features/#custom-plugins)



Weave Scope 由 [App 和 Probe 两部分](https://www.weave.works/docs/scope/latest/how-it-works)组成，它们

- Probe 负责收集容器和宿主的信息，并发送给 App
- App 负责处理这些信息，并生成相应的报告，并以交互界面的形式展示

```sh
                    +--Docker host----------+      +--Docker host----------+
.---------------.   |  +--Container------+  |      |  +--Container------+  |
| Browser       |   |  |                 |  |      |  |                 |  |
|---------------|   |  |  +-----------+  |  |      |  |  +-----------+  |  |
|               |----->|  | scope-app |<-----.    .----->| scope-app |  |  |
|               |   |  |  +-----------+  |  | \  / |  |  +-----------+  |  |
|               |   |  |        ^        |  |  \/  |  |        ^        |  |
'---------------'   |  |        |        |  |  /\  |  |        |        |  |
                    |  | +-------------+ |  | /  \ |  | +-------------+ |  |
                    |  | | scope-probe |-----'    '-----| scope-probe | |  |
                    |  | +-------------+ |  |      |  | +-------------+ |  |
                    |  |                 |  |      |  |                 |  |
                    |  +-----------------+  |      |  +-----------------+  |
                    +-----------------------+      +-----------------------+
```



#### 安装

```sh
$ kubectl apply -f "https://cloud.weave.works/k8s/scope.yaml?k8s-version=$(kubectl version | base64 | tr -d '\n')&k8s-service-type=LoadBalancer"
```



#### 查看界面

安装完成后，可以通过 weave-scope-app 来访问交互界面

```sh
$ kubectl -n weave get service weave-scope-app
$ kubectl -n weave port-forward service/weave-scope-app :80
```

![img](D:\学习资料\笔记\k8s\k8s图\assets%2)



点击 Pod，还可以查看该 Pod 所有容器的实时状态和度量数据：

![img](D:\学习资料\笔记\k8s\k8s图\daeeqr)



#### 已知问题

在 Ubuntu 内核 4.4.0 上面开启 `--probe.ebpf.connections` 时（默认开启），Node 有可能会因为[内核问题而不停重启](https://github.com/weaveworks/scope/issues/3131)：

```sh
[ 263.736006] CPU: 0 PID: 6309 Comm: scope Not tainted 4.4.0-119-generic #143-Ubuntu
[ 263.736006] Hardware name: Microsoft Corporation Virtual Machine/Virtual Machine, BIOS 090007 06/02/2017
[ 263.736006] task: ffff88011cef5400 ti: ffff88000a0e4000 task.ti: ffff88000a0e4000
[ 263.736006] RIP: 0010:[] [] bpf_map_lookup_elem+0x6/0x20
[ 263.736006] RSP: 0018:ffff88000a0e7a70 EFLAGS: 00010082
[ 263.736006] RAX: ffffffff8117cd70 RBX: ffffc90000762068 RCX: 0000000000000000
[ 263.736006] RDX: 0000000000000000 RSI: ffff88000a0e7cd8 RDI: 000000001cdee380
[ 263.736006] RBP: ffff88000a0e7cf8 R08: 0000000005080021 R09: 0000000000000000
[ 263.736006] R10: 0000000000000020 R11: ffff880159e1c700 R12: 0000000000000000
[ 263.736006] R13: ffff88011cfaf400 R14: ffff88000a0e7e38 R15: ffff88000a0f8800
[ 263.736006] FS: 00007f5b0cd79700(0000) GS:ffff88015b600000(0000) knlGS:0000000000000000
[ 263.736006] CS: 0010 DS: 0000 ES: 0000 CR0: 0000000080050033
[ 263.736006] CR2: 000000001cdee3a8 CR3: 000000011ce04000 CR4: 0000000000040670
[ 263.736006] Stack:
[ 263.736006] ffff88000a0e7cf8 ffffffff81177411 0000000000000000 00001887000018a5
[ 263.736006] 000000001cdee380 ffff88000a0e7cd8 0000000000000000 0000000000000000
[ 263.736006] 0000000005080021 ffff88000a0e7e38 0000000000000000 0000000000000046
[ 263.736006] Call Trace:
[ 263.736006] [] ? __bpf_prog_run+0x7a1/0x1360
[ 263.736006] [] ? update_curr+0x79/0x170
[ 263.736006] [] ? update_cfs_shares+0xbc/0x100
[ 263.736006] [] ? update_curr+0x79/0x170
[ 263.736006] [] ? dput+0xb8/0x230
[ 263.736006] [] ? follow_managed+0x265/0x300
[ 263.736006] [] ? kmem_cache_alloc_trace+0x1d4/0x1f0
[ 263.736006] [] ? seq_open+0x5a/0xa0
[ 263.736006] [] ? probes_open+0x33/0x100
[ 263.736006] [] ? dput+0x34/0x230
[ 263.736006] [] ? mntput+0x24/0x40
[ 263.736006] [] trace_call_bpf+0x37/0x50
[ 263.736006] [] kretprobe_perf_func+0x3d/0x250
[ 263.736006] [] ? pre_handler_kretprobe+0x135/0x1b0
[ 263.736006] [] kretprobe_dispatcher+0x3d/0x60
[ 263.736006] [] ? do_sys_open+0x1b2/0x2a0
[ 263.736006] [] ? kretprobe_trampoline_holder+0x9/0x9
[ 263.736006] [] trampoline_handler+0x133/0x210
[ 263.736006] [] ? do_sys_open+0x1b2/0x2a0
[ 263.736006] [] kretprobe_trampoline+0x25/0x57
[ 263.736006] [] ? kretprobe_trampoline_holder+0x9/0x9
[ 263.736006] [] SyS_openat+0x14/0x20
[ 263.736006] [] entry_SYSCALL_64_fastpath+0x1c/0xbb
```



解决方法有两种

- 禁止 eBPF 探测，如 `--probe.ebpf.connections=false`
- 升级内核，如升级到 4.13.0



### 参考文档

- [Overview of kubectl](https://kubernetes.io/docs/reference/kubectl/overview/)
- [Monitoring Kuberietes with sysdig](https://sysdig.com/blog/kubernetes-service-discovery-docker/)







# 排错指南



## Pod 排错

本文是本书排查指南板块下问题排查章节的 Pod排错 一节，介绍 Pod 各种异常现象，可能的原因以及解决方法。



### 常用命令

排查过程常用的命名如下:

- 查看 Pod 状态: `kubectl get pod <pod-name> -o wide`
- 查看 Pod 的 yaml 配置: `kubectl get pod <pod-name> -o yaml`
- 查看 Pod 事件: `kubectl describe pod <pod-name>`
- 查看容器日志: `kubectl logs <pod-name> [-c <container-name>]`



### Pod 状态

Pod 有多种状态，这里罗列一下:

- `Error`: Pod 启动过程中发生错误
- `NodeLost`: Pod 所在节点失联
- `Unkown`: Pod 所在节点失联或其它未知异常
- `Waiting`: Pod 等待启动
- `Pending`: Pod 等待被调度
- `ContainerCreating`: Pod 容器正在被创建
- `Terminating`: Pod 正在被销毁
- `CrashLoopBackOff`： 容器退出，kubelet 正在将它重启
- `InvalidImageName`： 无法解析镜像名称
- `ImageInspectError`： 无法校验镜像
- `ErrImageNeverPull`： 策略禁止拉取镜像
- `ImagePullBackOff`： 正在重试拉取
- `RegistryUnavailable`： 连接不到镜像中心
- `ErrImagePull`： 通用的拉取镜像出错
- `CreateContainerConfigError`： 不能创建 kubelet 使用的容器配置
- `CreateContainerError`： 创建容器失败
- `RunContainerError`： 启动容器失败
- `PreStartHookError`: 执行 preStart hook 报错
- `PostStartHookError`： 执行 postStart hook 报错
- `ContainersNotInitialized`： 容器没有初始化完毕
- `ContainersNotReady`： 容器没有准备完毕
- `ContainerCreating`：容器创建中
- `PodInitializing`：pod 初始化中
- `DockerDaemonNotReady`：docker还没有完全启动
- `NetworkPluginNotReady`： 网络插件还没有完全启动



### Pod 一直处于 Pending 状态

Pending 状态说明 Pod 还没有被调度到某个节点上，需要看下 Pod 事件进一步判断原因，比如:

```sh
$ kubectl describe pod tikv-0...Events:  Type     Reason            Age                 From               Message  ----     ------            ----                ----               -------  Warning  FailedScheduling  3m (x106 over 33m)  default-scheduler  0/4 nodes are available: 1 node(s) had no available volume zone, 2 Insufficient cpu, 3 Insufficient memory.
```

下面列举下可能原因和解决方法。



#### 节点资源不够

节点资源不够有以下几种情况:

- CPU 负载过高
- 剩余可以被分配的内存不够
- 剩余可用 GPU 数量不够 (通常在机器学习场景，GPU 集群环境)

如果判断某个 Node 资源是否足够？ 通过 `kubectl describe node <node-name>` 查看 node 资源情况，关注以下信息：

- `Allocatable`: 表示此节点能够申请的资源总和
- `Allocated resources`: 表示此节点已分配的资源 (Allocatable 减去节点上所有 Pod 总的 Request)

前者与后者相减，可得出剩余可申请的资源。如果这个值小于 Pod 的 request，就不满足 Pod 的资源要求，Scheduler 在 Predicates (预选) 阶段就会剔除掉这个 Node，也就不会调度上去。



#### 不满足 nodeSelector 与 affinity

如果 Pod 包含 nodeSelector 指定了节点需要包含的 label，调度器将只会考虑将 Pod 调度到包含这些 label 的 Node 上，如果没有 Node 有这些 label 或者有这些 label 的 Node 其它条件不满足也将会无法调度。参考官方文档：https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#nodeselector

如果 Pod 包含 affinity（亲和性）的配置，调度器根据调度算法也可能算出没有满足条件的 Node，从而无法调度。affinity 有以下几类:

- nodeAffinity: 节点亲和性，可以看成是增强版的 nodeSelector，用于限制 Pod 只允许被调度到某一部分 Node。
- podAffinity: Pod 亲和性，用于将一些有关联的 Pod 调度到同一个地方，同一个地方可以是指同一个节点或同一个可用区的节点等。
- podAntiAffinity: Pod 反亲和性，用于避免将某一类 Pod 调度到同一个地方避免单点故障，比如将集群 DNS 服务的 Pod 副本都调度到不同节点，避免一个节点挂了造成整个集群 DNS 解析失败，使得业务中断。



#### Node 存在 Pod 没有容忍的污点

如果节点上存在污点 (Taints)，而 Pod 没有响应的容忍 (Tolerations)，Pod 也将不会调度上去。通过 describe node 可以看下 Node 有哪些 Taints:

```sh
$ kubectl describe nodes host1
...
Taints:             special=true:NoSchedule
...
```

污点既可以是手动添加也可以是被自动添加，下面来深入分析一下。



##### 手动添加的污点

通过类似以下方式可以给节点添加污点:

```sh
$ kubectl taint node host1 special=true:NoSchedule
node "host1" tainted
```

另外，有些场景下希望新加的节点默认不调度 Pod，直到调整完节点上某些配置才允许调度，就给新加的节点都加上 `node.kubernetes.io/unschedulable` 这个污点。



##### 自动添加的污点

如果节点运行状态不正常，污点也可以被自动添加，从 v1.12 开始，`TaintNodesByCondition` 特性进入 Beta 默认开启，controller manager 会检查 Node 的 Condition，如果命中条件就自动为 Node 加上相应的污点，这些 Condition 与 Taints 的对应关系如下:

```sh
Conditon               Value       Taints
--------               -----       ------
OutOfDisk              True        node.kubernetes.io/out-of-disk
Ready                  False       node.kubernetes.io/not-ready
Ready                  Unknown     node.kubernetes.io/unreachable
MemoryPressure         True        node.kubernetes.io/memory-pressure
PIDPressure            True        node.kubernetes.io/pid-pressure
DiskPressure           True        node.kubernetes.io/disk-pressure
NetworkUnavailable     True        node.kubernetes.io/network-unavailable
```

解释下上面各种条件的意思:

- OutOfDisk 为 True 表示节点磁盘空间不够了
- Ready 为 False 表示节点不健康
- Ready 为 Unknown 表示节点失联，在 `node-monitor-grace-period` 这么长的时间内没有上报状态 controller-manager 就会将 Node 状态置为 Unknown (默认 40s)
- MemoryPressure 为 True 表示节点内存压力大，实际可用内存很少
- PIDPressure 为 True 表示节点上运行了太多进程，PID 数量不够用了
- DiskPressure 为 True 表示节点上的磁盘可用空间太少了
- NetworkUnavailable 为 True 表示节点上的网络没有正确配置，无法跟其它 Pod 正常通信

另外，在云环境下，比如腾讯云 TKE，添加新节点会先给这个 Node 加上 `node.cloudprovider.kubernetes.io/uninitialized` 的污点，等 Node 初始化成功后才自动移除这个污点，避免 Pod 被调度到没初始化好的 Node 上。



#### 低版本 kube-scheduler 的 bug

可能是低版本 `kube-scheduler` 的 bug, 可以升级下调度器版本。



#### kube-scheduler 没有正常运行

检查 maser 上的 `kube-scheduler` 是否运行正常，异常的话可以尝试重启临时恢复。



#### 驱逐后其它可用节点与当前节点有状态应用不在同一个可用区

有时候服务部署成功运行过，但在某个时候节点突然挂了，此时就会触发驱逐，创建新的副本调度到其它节点上，对于已经挂载了磁盘的 Pod，它通常需要被调度到跟当前节点和磁盘在同一个可用区，如果集群中同一个可用区的节点不满足调度条件，即使其它可用区节点各种条件都满足，但不跟当前节点在同一个可用区，也是不会调度的。为什么需要限制挂载了磁盘的 Pod 不能漂移到其它可用区的节点？试想一下，云上的磁盘虽然可以被动态挂载到不同机器，但也只是相对同一个数据中心，通常不允许跨数据中心挂载磁盘设备，因为网络时延会极大的降低 IO 速率。





### Pod 一直处于 ContainerCreating 或 Waiting 状态

#### Pod 配置错误

- 检查是否打包了正确的镜像
- 检查配置了正确的容器参数



#### 挂载 Volume 失败

Volume 挂载失败也分许多种情况，先列下我这里目前已知的。



##### Pod 漂移没有正常解挂之前的磁盘

在云尝试托管的 K8S 服务环境下，默认挂载的 Volume 一般是块存储类型的云硬盘，如果某个节点挂了，kubelet 无法正常运行或与 apiserver 通信，到达时间阀值后会触发驱逐，自动在其它节点上启动相同的副本 (Pod 漂移)，但是由于被驱逐的 Node 无法正常运行并不知道自己被驱逐了，也就没有正常执行解挂，cloud-controller-manager 也在等解挂成功后再调用云厂商的接口将磁盘真正从节点上解挂，通常会等到一个时间阀值后 cloud-controller-manager 会强制解挂云盘，然后再将其挂载到 Pod 最新所在节点上，这种情况下 ContainerCreating 的时间相对长一点，但一般最终是可以启动成功的，除非云厂商的 cloud-controller-manager 逻辑有 bug。



##### 命中 K8S 挂载 configmap/secret 的 subpath 的 bug

最近发现如果 Pod 挂载了 configmap 或 secret， 如果后面修改了 configmap 或 secret 的内容，Pod 里的容器又原地重启了(比如存活检查失败被 kill 然后重启拉起)，就会触发 K8S 的这个 bug，团队的小伙伴已提 PR: https://github.com/kubernetes/kubernetes/pull/82784

如果是这种情况，容器会一直启动不成功，可以看到类似以下的报错:

```sh
$ kubectl -n prod get pod -o yaml manage-5bd487cf9d-bqmvm
...
lastState: terminated
containerID: containerd://e6746201faa1dfe7f3251b8c30d59ebf613d99715f3b800740e587e681d2a903
exitCode: 128
finishedAt: 2019-09-15T00:47:22Z
message: 'failed to create containerd task: OCI runtime create failed: container_linux.go:345:
starting container process caused "process_linux.go:424: container init
caused \"rootfs_linux.go:58: mounting \\\"/var/lib/kubelet/pods/211d53f4-d08c-11e9-b0a7-b6655eaf02a6/volume-subpaths/manage-config-volume/manage/0\\\"
to rootfs \\\"/run/containerd/io.containerd.runtime.v1.linux/k8s.io/e6746201faa1dfe7f3251b8c30d59ebf613d99715f3b800740e587e681d2a903/rootfs\\\"
at \\\"/run/containerd/io.containerd.runtime.v1.linux/k8s.io/e6746201faa1dfe7f3251b8c30d59ebf613d99715f3b800740e587e681d2a903/rootfs/app/resources/application.properties\\\"
caused \\\"no such file or directory\\\"\"": unknown'
```



#### 磁盘爆满

启动 Pod 会调 CRI 接口创建容器，容器运行时创建容器时通常会在数据目录下为新建的容器创建一些目录和文件，如果数据目录所在的磁盘空间满了就会创建失败并报错:

```sh
Events:
  Type     Reason                  Age                  From                   Message
  ----     ------                  ----                 ----                   -------
  Warning  FailedCreatePodSandBox  2m (x4307 over 16h)  kubelet, 10.179.80.31  (combined from similar events): Failed create pod sandbox: rpc error: code = Unknown desc = failed to create a sandbox for pod "apigateway-6dc48bf8b6-l8xrw": Error response from daemon: mkdir /var/lib/docker/aufs/mnt/1f09d6c1c9f24e8daaea5bf33a4230de7dbc758e3b22785e8ee21e3e3d921214-init: no space left on device
```

解决方法参考本书 [处理实践：磁盘爆满](https://www.bookstack.cn/read/kubernetes-practice-guide/$troubleshooting-problems-pod-troubleshooting-handle-disk-full.md)



#### 节点内存碎片化

如果节点上内存碎片化严重，缺少大页内存，会导致即使总的剩余内存较多，但还是会申请内存失败，参考 [处理实践: 内存碎片化](https://www.bookstack.cn/read/kubernetes-practice-guide/$troubleshooting-problems-pod-troubleshooting-handle-memory-fragmentation.md)



#### limit 设置太小或者单位不对

如果 limit 设置过小以至于不足以成功运行 Sandbox 也会造成这种状态，常见的是因为 memory limit 单位设置不对造成的 limit 过小，比如误将 memory 的 limit 单位像 request 一样设置为小 `m`，这个单位在 memory 不适用，会被 k8s 识别成 byte， 应该用 `Mi` 或 `M`。，

举个例子: 如果 memory limit 设为 1024m 表示限制 1.024 Byte，这么小的内存， pause 容器一起来就会被 cgroup-oom kill 掉，导致 pod 状态一直处于 ContainerCreating。

这种情况通常会报下面的 event:

```sh
Pod sandbox changed, it will be killed and re-created。
```

kubelet 报错:

```sh
to start sandbox container for pod ... Error response from daemon: OCI runtime create failed: container_linux.go:348: starting container process caused "process_linux.go:301: running exec setns process for init caused \"signal: killed\"": unknown
```



#### 拉取镜像失败

镜像拉取失败也分很多情况，这里列举下:

- 配置了错误的镜像
- Kubelet 无法访问镜像仓库（比如默认 pause 镜像在 gcr.io 上，国内环境访问需要特殊处理）
- 拉取私有镜像的 imagePullSecret 没有配置或配置有误
- 镜像太大，拉取超时（可以适当调整 kubelet 的 —image-pull-progress-deadline 和 —runtime-request-timeout 选项）



#### CNI 网络错误

如果发生 CNI 网络错误通常需要检查下网络插件的配置和运行状态，如果没有正确配置或正常运行通常表现为:

- 无法配置 Pod 网络
- 无法分配 Pod IP



#### controller-manager 异常

查看 master 上 kube-controller-manager 状态，异常的话尝试重启。



#### 安装 docker 没删干净旧版本

如果节点上本身有 docker 或者没删干净，然后又安装 docker，比如在 centos 上用 yum 安装:

```sh
yum install -y docker
```

这样可能会导致 dockerd 创建容器一直不成功，从而 Pod 状态一直 ContainerCreating，查看 event 报错:

```sh
  Type     Reason                  Age                     From                  Message
  ----     ------                  ----                    ----                  -------
  Warning  FailedCreatePodSandBox  18m (x3583 over 83m)    kubelet, 192.168.4.5  (combined from similar events): Failed create pod sandbox: rpc error: code = Unknown desc = failed to start sandbox container for pod "nginx-7db9fccd9b-2j6dh": Error response from daemon: ttrpc: client shutting down: read unix @->@/containerd-shim/moby/de2bfeefc999af42783115acca62745e6798981dff75f4148fae8c086668f667/shim.sock: read: connection reset by peer: unknown
  Normal   SandboxChanged          3m12s (x4420 over 83m)  kubelet, 192.168.4.5  Pod sandbox changed, it will be killed and re-created.
```

可能是因为重复安装 docker 版本不一致导致一些组件之间不兼容，从而导致 dockerd 无法正常创建容器。



#### 存在同名容器

如果节点上已有同名容器，创建 sandbox 就会失败，event:

```sh
  Warning  FailedCreatePodSandBox  2m                kubelet, 10.205.8.91  Failed create pod sandbox: rpc error: code = Unknown desc = failed to create a sandbox for pod "lomp-ext-d8c8b8c46-4v8tl": operation timeout: context deadline exceeded
  Warning  FailedCreatePodSandBox  3s (x12 over 2m)  kubelet, 10.205.8.91  Failed create pod sandbox: rpc error: code = Unknown desc = failed to create a sandbox for pod "lomp-ext-d8c8b8c46-4v8tl": Error response from daemon: Conflict. The container name "/k8s_POD_lomp-ext-d8c8b8c46-4v8tl_default_65046a06-f795-11e9-9bb6-b67fb7a70bad_0" is already in use by container "30aa3f5847e0ce89e9d411e76783ba14accba7eb7743e605a10a9a862a72c1e2". You have to remove (or rename) that container to be able to reuse that name.
```

关于什么情况下会产生同名容器，这个有待研究。



### Pod 处于 CrashLoopBackOff 状态

Pod 如果处于 `CrashLoopBackOff` 状态说明之前是启动了，只是又异常退出了，只要 Pod 的 [restartPolicy](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy) 不是 Never 就可能被重启拉起，此时 Pod 的 `RestartCounts` 通常是大于 0 的，可以先看下容器进程的退出状态码来缩小问题范围，参考本书 [排错技巧: 分析 ExitCode 定位 Pod 异常退出原因](https://www.bookstack.cn/read/kubernetes-practice-guide/$troubleshooting-problems-pod-troubleshooting-trick-analysis-exitcode.md)



#### 容器进程主动退出

如果是容器进程主动退出，退出状态码一般在 0-128 之间，除了可能是业务程序 BUG，还有其它许多可能原因，参考: [容器进程主动退出](https://www.bookstack.cn/read/kubernetes-practice-guide/troubleshooting-problems-pod-container-proccess-exit-by-itself.md)



#### 系统 OOM

如果发生系统 OOM，可以看到 Pod 中容器退出状态码是 137，表示被 `SIGKILL` 信号杀死，同时内核会报错: `Out of memory: Kill process ...`。大概率是节点上部署了其它非 K8S 管理的进程消耗了比较多的内存，或者 kubelet 的 `--kube-reserved` 和 `--system-reserved` 配的比较小，没有预留足够的空间给其它非容器进程，节点上所有 Pod 的实际内存占用总量不会超过 `/sys/fs/cgroup/memory/kubepods` 这里 cgroup 的限制，这个限制等于 `capacity - "kube-reserved" - "system-reserved"`，如果预留空间设置合理，节点上其它非容器进程（kubelet, dockerd, kube-proxy, sshd 等) 内存占用没有超过 kubelet 配置的预留空间是不会发生系统 OOM 的，可以根据实际需求做合理的调整。



#### cgroup OOM

如果是 cgroup OOM 杀掉的进程，从 Pod 事件的下 `Reason` 可以看到是 `OOMKilled`，说明容器实际占用的内存超过 limit 了，同时内核日志会报: ``。 可以根据需求调整下 limit。



#### 节点内存碎片化

如果节点上内存碎片化严重，缺少大页内存，会导致即使总的剩余内存较多，但还是会申请内存失败，参考 [处理实践: 内存碎片化](https://www.bookstack.cn/read/kubernetes-practice-guide/$troubleshooting-problems-pod-troubleshooting-handle-memory-fragmentation.md)



#### 健康检查失败

参考 [Pod 健康检查失败](https://www.bookstack.cn/read/kubernetes-practice-guide/troubleshooting-problems-pod-healthcheck-failed.md) 进一步定位。





### Pod 一直处于 Terminating 状态

#### 磁盘爆满

如果 docker 的数据目录所在磁盘被写满，docker 无法正常运行，无法进行删除和创建操作，所以 kubelet 调用 docker 删除容器没反应，看 event 类似这样：

```sh
Normal  Killing  39s (x735 over 15h)  kubelet, 10.179.80.31  Killing container with id docker://apigateway:Need to kill Pod
```

处理建议是参考本书 [处理实践：磁盘爆满](https://www.bookstack.cn/read/kubernetes-practice-guide/$troubleshooting-problems-pod-troubleshooting-handle-disk-full.md)



#### 存在 “i” 文件属性

如果容器的镜像本身或者容器启动后写入的文件存在 “i” 文件属性，此文件就无法被修改删除，而删除 Pod 时会清理容器目录，但里面包含有不可删除的文件，就一直删不了，Pod 状态也将一直保持 Terminating，kubelet 报错:

```sh
Sep 27 14:37:21 VM_0_7_centos kubelet[14109]: E0927 14:37:21.922965   14109 remote_runtime.go:250] RemoveContainer "19d837c77a3c294052a99ff9347c520bc8acb7b8b9a9dc9fab281fc09df38257" from runtime service failed: rpc error: code = Unknown desc = failed to remove container "19d837c77a3c294052a99ff9347c520bc8acb7b8b9a9dc9fab281fc09df38257": Error response from daemon: container 19d837c77a3c294052a99ff9347c520bc8acb7b8b9a9dc9fab281fc09df38257: driver "overlay2" failed to remove root filesystem: remove /data/docker/overlay2/b1aea29c590aa9abda79f7cf3976422073fb3652757f0391db88534027546868/diff/usr/bin/bash: operation not permitted
Sep 27 14:37:21 VM_0_7_centos kubelet[14109]: E0927 14:37:21.923027   14109 kuberuntime_gc.go:126] Failed to remove container "19d837c77a3c294052a99ff9347c520bc8acb7b8b9a9dc9fab281fc09df38257": rpc error: code = Unknown desc = failed to remove container "19d837c77a3c294052a99ff9347c520bc8acb7b8b9a9dc9fab281fc09df38257": Error response from daemon: container 19d837c77a3c294052a99ff9347c520bc8acb7b8b9a9dc9fab281fc09df38257: driver "overlay2" failed to remove root filesystem: remove /data/docker/overlay2/b1aea29c590aa9abda79f7cf3976422073fb3652757f0391db88534027546868/diff/usr/bin/bash: operation not permitted
```

通过 `man chattr` 查看 “i” 文件属性描述:

```sh
       A file with the 'i' attribute cannot be modified: it cannot be deleted or renamed, no
link can be created to this file and no data can be written to the file.  Only the superuser
or a process possessing the CAP_LINUX_IMMUTABLE capability can set or clear this attribute.
```

彻底解决当然是不要在容器镜像中或启动后的容器设置 “i” 文件属性，临时恢复方法： 复制 kubelet 日志报错提示的文件路径，然后执行 `chattr -i <file>`:

```sh
chattr -i /data/docker/overlay2/b1aea29c590aa9abda79f7cf3976422073fb3652757f0391db88534027546868/diff/usr/bin/bash
```

执行完后等待 kubelet 自动重试，Pod 就可以被自动删除了。



#### docker 17 的 bug

docker hang 住，没有任何响应，看 event:

```sh
Warning FailedSync 3m (x408 over 1h) kubelet, 10.179.80.31 error determining status: rpc error: code = DeadlineExceeded desc = context deadline exceeded
```

怀疑是17版本dockerd的BUG。可通过 `kubectl -n cn-staging delete pod apigateway-6dc48bf8b6-clcwk --force --grace-period=0` 强制删除pod，但 `docker ps` 仍看得到这个容器

处置建议：

- 升级到docker 18. 该版本使用了新的 containerd，针对很多bug进行了修复。
- 如果出现terminating状态的话，可以提供让容器专家进行排查，不建议直接强行删除，会可能导致一些业务上问题。



#### 存在 Finalizers

k8s 资源的 metadata 里如果存在 `finalizers`，那么该资源一般是由某程序创建的，并且在其创建的资源的 metadata 里的 `finalizers` 加了一个它的标识，这意味着这个资源被删除时需要由创建资源的程序来做删除前的清理，清理完了它需要将标识从该资源的 `finalizers` 中移除，然后才会最终彻底删除资源。比如 Rancher 创建的一些资源就会写入 `finalizers` 标识。

处理建议：`kubectl edit` 手动编辑资源定义，删掉 `finalizers`，这时再看下资源，就会发现已经删掉了



#### 低版本 kubelet list-watch 的 bug

之前遇到过使用 v1.8.13 版本的 k8s，kubelet 有时 list-watch 出问题，删除 pod 后 kubelet 没收到事件，导致 kubelet 一直没做删除操作，所以 pod 状态一直是 Terminating



#### dockerd 与 containerd 的状态不同步

判断 dockerd 与 containerd 某个容器的状态不同步的方法：

- describe pod 拿到容器 id

- docker ps 查看的容器状态是 dockerd 中保存的状态

- 通过 docker-container-ctr 查看容器在 containerd 中的状态，比如:

  ```sh
  $ docker-container-ctr --namespace moby --address /var/run/docker/containerd/docker-containerd.sock task ls |grep a9a1785b81343c3ad2093ad973f4f8e52dbf54823b8bb089886c8356d4036fe0a9a1785b81343c3
  ad2093ad973f4f8e52dbf54823b8bb089886c8356d4036fe0    30639    STOPPED
  ```

containerd 看容器状态是 stopped 或者已经没有记录，而 docker 看容器状态却是 runing，说明 dockerd 与 containerd 之间容器状态同步有问题，目前发现了 docker 在 aufs 存储驱动下如果磁盘爆满可能发生内核 panic :

```sh
aufs au_opts_verify:1597:dockerd[5347]: dirperm1 breaks the protection by the permission bits on the lower branch
```

如果磁盘爆满过，dockerd 一般会有下面类似的日志:

```sh
Sep 18 10:19:49 VM-1-33-ubuntu dockerd[4822]: time="2019-09-18T10:19:49.903943652+08:00" level=error msg="Failed to log msg \"\" for logger json-file: write /opt/docker/containers/54922ec8b1863bcc504f6dac41e40139047f7a84ff09175d2800100aaccbad1f/54922ec8b1863bcc504f6dac41e40139047f7a84ff09175d2800100aaccbad1f-json.log: no space left on device"
```

随后可能发生状态不同步，已提issue: https://github.com/docker/for-linux/issues/779

- 临时恢复: 执行 `docker prune` 或重启 dockerd
- 长期方案: 运行时推荐直接使用 containerd，绕过 dockerd 避免 docker 本身的各种 BUG



#### Daemonset Controller 的 BUG

有个 k8s 的 bug 会导致 daemonset pod 无限 terminating，1.10 和 1.11 版本受影响，原因是 daemonset controller 复用 scheduler 的 predicates 逻辑，里面将 nodeAffinity 的 nodeSelector 数组做了排序（传的指针），spec 就会跟 apiserver 中的不一致，daemonset controller 又会为 rollingUpdate类型计算 hash (会用到spec)，用于版本控制，造成不一致从而无限启动和停止的循环。

- issue: https://github.com/kubernetes/kubernetes/issues/66298
- 修复的PR: https://github.com/kubernetes/kubernetes/pull/66480

升级集群版本可以彻底解决，临时规避可以给 rollingUpdate 类型 daemonset 不使用 nodeAffinity，改用 nodeSelector。





### Pod 一直处于 Unknown 状态

通常是节点失联，没有上报状态给 apiserver，到达阀值后 controller-manager 认为节点失联并将其状态置为 `Unknown`。

可能原因:

- 节点高负载导致无法上报
- 节点宕机
- 节点被关机
- 网络不通



### Pod 一直处于 Error 状态

通常处于 Error 状态说明 Pod 启动过程中发生了错误。常见的原因包括：

- 依赖的 ConfigMap、Secret 或者 PV 等不存在
- 请求的资源超过了管理员设置的限制，比如超过了 LimitRange 等
- 违反集群的安全策略，比如违反了 PodSecurityPolicy 等
- 容器无权操作集群内的资源，比如开启 RBAC 后，需要为 ServiceAccount 配置角色绑定



### Pod 一直处于 ImagePullBackOff 状态

#### http 类型 registry，地址未加入到 insecure-registry

dockerd 默认从 https 类型的 registry 拉取镜像，如果使用 https 类型的 registry，则必须将它添加到 insecure-registry 参数中，然后重启或 reload dockerd 生效。



#### https 自签发类型 resitry，没有给节点添加 ca 证书

如果 registry 是 https 类型，但证书是自签发的，dockerd 会校验 registry 的证书，校验成功才能正常使用镜像仓库，要想校验成功就需要将 registry 的 ca 证书放置到 `/etc/docker/certs.d/<registry:port>/ca.crt` 位置。



#### 私有镜像仓库认证失败

如果 registry 需要认证，但是 Pod 没有配置 imagePullSecret，配置的 Secret 不存在或者有误都会认证失败。



#### 镜像文件损坏

如果 push 的镜像文件损坏了，下载下来也用不了，需要重新 push 镜像文件。



#### 镜像拉取超时

如果节点上新起的 Pod 太多就会有许多可能会造成容器镜像下载排队，如果前面有许多大镜像需要下载很长时间，后面排队的 Pod 就会报拉取超时。

kubelet 默认串行下载镜像:

```sh
--serialize-image-pulls   Pull images one at a time. We recommend *not* changing the default value on nodes that run docker daemon with version < 1.9 or an Aufs storage backend. Issue #10959 has more details. (default true)
```

也可以开启并行下载并控制并发:

```sh
--registry-qps int32   If > 0, limit registry pull QPS to this value.  If 0, unlimited. (default 5)
--registry-burst int32   Maximum size of a bursty pulls, temporarily allows pulls to burst to this number, while still not exceeding registry-qps. Only used if --registry-qps > 0 (default 10)
```



#### 镜像不存在

kubelet 日志:

```sh
PullImage "imroc/test:v0.2" from image service failed: rpc error: code = Unknown desc = Error response from 
```





### Pod 健康检查失败

- Kubernetes 健康检查包含就绪检查(readinessProbe)和存活检查(livenessProbe)
- pod 如果就绪检查失败会将此 pod ip 从 service 中摘除，通过 service 访问，流量将不会被转发给就绪检查失败的 pod
- pod 如果存活检查失败，kubelet 将会杀死容器并尝试重启

健康检查失败的可能原因有多种，除了业务程序BUG导致不能响应健康检查导致 unhealthy，还能有有其它原因，下面我们来逐个排查。



#### 健康检查配置不合理

`initialDelaySeconds` 太短，容器启动慢，导致容器还没完全启动就开始探测，如果 successThreshold 是默认值 1，检查失败一次就会被 kill，然后 pod 一直这样被 kill 重启。



#### 节点负载过高

cpu 占用高（比如跑满）会导致进程无法正常发包收包，通常会 timeout，导致 kubelet 认为 pod 不健康。参考本书 [处理实践: 高负载](https://www.bookstack.cn/read/kubernetes-practice-guide/troubleshooting-handle-high-load.md) 一节。



#### 容器进程被木马进程杀死

参考本书 [处理实践: 使用 systemtap 定位疑难杂症](https://www.bookstack.cn/read/kubernetes-practice-guide/$troubleshooting-problems-pod-troubleshooting-trick-use-systemtap-to-locate-problems.md) 进一步定位。



#### 容器内进程端口监听挂掉

使用 `netstat -tunlp` 检查端口监听是否还在，如果不在了，抓包可以看到会直接 reset 掉健康检查探测的连接:

```sh
20:15:17.890996 IP 172.16.2.1.38074 > 172.16.2.23.8888: Flags [S], seq 96880261, win 14600, options [mss 1424,nop,nop,sackOK,nop,wscale 7], length 0
20:15:17.891021 IP 172.16.2.23.8888 > 172.16.2.1.38074: Flags [R.], seq 0, ack 96880262, win 0, length 0
20:15:17.906744 IP 10.0.0.16.54132 > 172.16.2.23.8888: Flags [S], seq 1207014342, win 14600, options [mss 1424,nop,nop,sackOK,nop,wscale 7], length 0
20:15:17.906766 IP 172.16.2.23.8888 > 10.0.0.16.54132: Flags [R.], seq 0, ack 1207014343, win 0, length 0
```

连接异常，从而健康检查失败。发生这种情况的原因可能在一个节点上启动了多个使用 `hostNetwork` 监听相同宿主机端口的 Pod，只会有一个 Pod 监听成功，但监听失败的 Pod 的业务逻辑允许了监听失败，并没有退出，Pod 又配了健康检查，kubelet 就会给 Pod 发送健康检查探测报文，但 Pod 由于没有监听所以就会健康检查失败。



#### SYN backlog 设置过小

SYN backlog 大小即 SYN 队列大小，如果短时间内新建连接比较多，而 SYN backlog 设置太小，就会导致新建连接失败，通过 `netstat -s | grep TCPBacklogDrop` 可以看到有多少是因为 backlog 满了导致丢弃的新连接。

如果确认是 backlog 满了导致的丢包，建议调高 backlog 的值，内核参数为 `net.ipv4.tcp_max_syn_backlog`。



### 容器进程主动退出

容器进程如果是自己主动退出(不是被外界中断杀死)，退出状态码一般在 0-128 之间，根据约定，正常退出时状态码为 0，1-127 说明是程序发生异常，主动退出了，比如检测到启动的参数和条件不满足要求，或者运行过程中发生 panic 但没有捕获处理导致程序退出。除了可能是业务程序 BUG，还有其它许多可能原因，这里我们一一列举下。



#### DNS 无法解析

可能程序依赖 集群 DNS 服务，比如启动时连接数据库，数据库使用 service 名称或外部域名都需要 DNS 解析，如果解析失败程序将报错并主动退出。解析失败的可能原因:

- 集群网络有问题，Pod 连不上集群 DNS 服务
- 集群 DNS 服务挂了，无法响应解析请求
- Service 或域名地址配置有误，本身是无法解析的地址



#### 程序配置有误

- 配置文件格式错误，程序启动解析配置失败报错退出
- 配置内容不符合规范，比如配置中某个字段是必选但没有填写，配置校验不通过，程序报错主动退出





## 网络排错



### LB 健康检查失败

可能原因:

- 节点防火墙规则没放开 nodeport 区间端口 (默认 30000-32768) 检查iptables和云主机安全组
- LB IP 绑到 `kube-ipvs0` 导致丢源 IP为 LB IP 的包: https://github.com/kubernetes/kubernetes/issues/79783



### DNS 解析异常

#### 5 秒延时

如果DNS查询经常延时5秒才返回，通常是遇到内核 conntrack 冲突导致的丢包，详见 [案例分享: DNS 5秒延时](https://www.bookstack.cn/read/kubernetes-practice-guide/$troubleshooting-problems-network-troubleshooting-cases-dns-lookup-5s-delay.md)



#### 解析超时

如果容器内报 DNS 解析超时，先检查下集群 DNS 服务 (`kube-dns`/`coredns`) 的 Pod 是否 Ready，如果不是，请参考本章其它小节定位原因。如果运行正常，再具体看下超时现象。



#### 解析外部域名超时

可能原因:

- 上游 DNS 故障
- 上游 DNS 的 ACL 或防火墙拦截了报文



#### 所有解析都超时

如果集群内某个 Pod 不管解析 Service 还是外部域名都失败，通常是 Pod 与集群 DNS 之间通信有问题。

可能原因:

- 节点防火墙没放开集群网段，导致如果 Pod 跟集群 DNS 的 Pod 不在同一个节点就无法通信，DNS 请求也就无法被收到



### Service 无法解析

#### 集群 DNS 没有正常运行(kube-dns或CoreDNS)

检查集群 DNS 是否运行正常:

- kubelet 启动参数 `--cluster-dns` 可以看到 dns 服务的 cluster ip:

```sh
$ ps -ef | grep kubelet
... /usr/bin/kubelet --cluster-dns=172.16.14.217 ...
```

- 找到 dns 的 service:

```sh
$ kubectl get svc -n kube-system | grep 172.16.14.217
kube-dns      ClusterIP   172.16.14.217   <none>      53/TCP,53/UDP     47d
```

- 看是否存在 endpoint:

```sh
$ kubectl -n kube-system describe svc kube-dns | grep -i endpoints
Endpoints:         172.16.0.156:53,172.16.0.167:53
Endpoints:         172.16.0.156:53,172.16.0.167:53
```

- 检查 endpoint 的 对应 pod 是否正常:

```sh
$ kubectl -n kube-system get pod -o wide | grep 172.16.0.156
kube-dns-898dbbfc6-hvwlr    3/3       Running   0   8d   172.16.0.156   10.0.0.3
```



#### Pod 与 DNS 服务之间网络不通

检查下 pod 是否连不上 dns 服务，可以在 pod 里 telnet 一下 dns 的 53 端口:

```sh
# 连 dns service 的 cluster ip
$ telnet 172.16.14.217 53
```

如果检查到是网络不通，就需要排查下网络设置:

- 检查节点的安全组设置，需要放开集群的容器网段
- 检查是否还有防火墙规则，检查 iptables



### 网络性能差

IPVS 模式吞吐性能低

内核参数关闭 `conn_reuse_mode`:

```sh
sysctl net.ipv4.vs.conn_reuse_mode=0
```

参考 issue: [https://github.com/kubernetes/kubernetes/issues/7074](https://github.com/kubernetes/kubernetes/issues/70747)





## 集群排错



### Node 全部消失

#### Rancher 清除 Node 导致集群异常

##### 现象

安装了 rancher 的用户，在卸载 rancher 的时候，可能会手动执行 `kubectl delete ns local` 来删除这个 rancher 创建的 namespace，但直接这样做会导致所有 node 被清除，通过 `kubectl get node` 获取不到 node。



##### 原因

看了下 rancher 源码，rancher 通过 `nodes.management.cattle.io` 这个 CRD 存储和管理 node，会给所有 node 创建对应的这个 CRD 资源，metadata 中加入了两个 finalizer，其中 `user-node-remove_local` 对应的 finalizer 处理逻辑就是删除对应的 k8s node 资源，也就是 `delete ns local` 时，会尝试删除 `nodes.management.cattle.io` 这些 CRD 资源，进而触发 rancher 的 finalizer 逻辑去删除对应的 k8s node 资源，从而清空了 node，所以 `kubectl get node` 就看不到 node 了，集群里的服务就无法被调度。



##### 规避方案

不要在 rancher 组件卸载完之前手动 `delete ns local`。



### Daemonset 没有被调度

Daemonset 的期望实例为 0，可能原因:

- controller-manager 的 bug，重启 controller-manager 可以恢复
- controller-manager 挂了





## 经典报错



### no space left on device

- 有时候节点 NotReady， kubelet 日志报 `no space left on device`
- 有时候创建 Pod 失败，`describe pod` 看 event 报 `no space left on device`

出现这种错误有很多中可能原因，下面我们来根据现象找对应原因。



#### inotify watch 耗尽

节点 NotReady，kubelet 启动失败，看 kubelet 日志:

```sh
Jul 18 15:20:58 VM_16_16_centos kubelet[11519]: E0718 15:20:58.280275   11519 raw.go:140] Failed to watch directory "/sys/fs/cgroup/memory/kubepods": inotify_add_watch /sys/fs/cgroup/memory/kubepods/burstable/pod926b7ff4-7bff-11e8-945b-52540048533c/6e85761a30707b43ed874e0140f58839618285fc90717153b3cbe7f91629ef5a: no space left on device
```

系统调用 `inotify_add_watch` 失败，提示 `no space left on device`， 这是因为系统上进程 watch 文件目录的总数超出了最大限制，可以修改内核参数调高限制，详细请参考本书 [处理实践: inotify watch 耗尽](https://www.bookstack.cn/read/kubernetes-practice-guide/$troubleshooting-problems-errors-troubleshooting-handle-runnig-out-of-inotify-watches.md)



#### cgroup 泄露

查看当前 cgroup 数量:

```sh
$ cat /proc/cgroups | column -t
#subsys_name  hierarchy  num_cgroups  enabled
cpuset        5          29           1
cpu           7          126          1
cpuacct       7          126          1
memory        9          127          1
devices       4          126          1
freezer       2          29           1
net_cls       6          29           1
blkio         10         126          1
perf_event    3          29           1
hugetlb       11         29           1
pids          8          126          1
net_prio      6          29           1
```

cgroup 子系统目录下面所有每个目录及其子目录都认为是一个独立的 cgroup，所以也可以在文件系统中统计目录数来获取实际 cgroup 数量，通常跟 `/proc/cgroups` 里面看到的应该一致:

```sh
$ find -L /sys/fs/cgroup/memory -type d | wc -l
127
```

当 cgroup 泄露发生时，这里的数量就不是真实的了，低版本内核限制最大 65535 个 cgroup，并且开启 kmem 删除 cgroup 时会泄露，大量创建删除容器后泄露了许多 cgroup，最终总数达到 65535，新建容器创建 cgroup 将会失败，报 `no space left on device`

详细请参考本书 [案例分享: cgroup 泄露](https://www.bookstack.cn/read/kubernetes-practice-guide/$troubleshooting-problems-errors-troubleshooting-summary-cgroup-leaking.md)



#### 磁盘被写满

Pod 启动失败，状态 `CreateContainerError`:

```sh
csi-cephfsplugin-27znb       0/2     CreateContainerError   167        17h
```

Pod 事件报错:

```sh
Warning  Failed   5m1s (x3397 over 17h)  kubelet, ip-10-0-151-35.us-west-2.compute.internal  (combined from similar events): Error: container create failed: container_linux.go:336: starting container process caused "process_linux.go:399: container init caused \"rootfs_linux.go:58: mounting \\\"/sys\\\" to rootfs \\\"/var/lib/containers/storage/overlay/051e985771cc69f3f699895a1dada9ef6483e912b46a99e004af7bb4852183eb/merged\\\" at \\\"/var/lib/containers/storage/overlay/051e985771cc69f3f699895a1dada9ef6483e912b46a99e004af7bb4852183eb/merged/sys\\\" caused \\\"no space left on device\\\"\""
```





### arp_cache: neighbor table overflow!

节点内核报这个错说明当前节点 arp 缓存满了。

查看当前 arp 记录数:

```sh
$ arp -an | wc -l
1335
```

查看 gc 阀值:

```sh
$ sysctl -a | grep net.ipv4.neigh.default.gc_thresh
net.ipv4.neigh.default.gc_thresh1 = 128
net.ipv4.neigh.default.gc_thresh2 = 512
net.ipv4.neigh.default.gc_thresh3 = 1024
```

当前 arp 记录数接近 gc_thresh3 比较容易 overflow，因为当 arp 记录达到 gc_thresh3 时会强制触发 gc 清理，当这时又有数据包要发送，并且根据目的 IP 在 arp cache 中没找到 mac 地址，这时会判断当前 arp cache 记录数加 1 是否大于 gc_thresh3，如果没有大于就会时就会报错: `neighbor table overflow!`



#### 什么场景下会发生

集群规模大，node 和 pod 数量超多，参考本书避坑宝典的 [案例分享: ARP 缓存爆满导致健康检查失败](https://www.bookstack.cn/read/kubernetes-practice-guide/$troubleshooting-problems-errors-troubleshooting-cases-arp-cache-overflow-causes-healthcheck-failed.md)



#### 解决方案

调整部分节点内核参数，将 arp cache 的 gc 阀值调高 (`/etc/sysctl.conf`):

```sh
net.ipv4.neigh.default.gc_thresh1 = 80000
net.ipv4.neigh.default.gc_thresh2 = 90000
net.ipv4.neigh.default.gc_thresh3 = 100000
```

并给 node 打下label，修改 pod spec，加下 nodeSelector 或者 nodeAffnity，让 pod 只调度到这部分改过内核参数的节点



#### 参考资料

- Scaling Kubernetes to 2,500 Nodes: https://openai.com/blog/scaling-kubernetes-to-2500-nodes/



### Cannot allocate memory

容器启动失败，报错 `Cannot allocate memory`。

PID 耗尽

如果登录 ssh 困难，并且登录成功后执行任意命名经常报 `Cannot allocate memory`，多半是 PID 耗尽了。

处理方法参考本书 [处理实践: PID 耗尽](https://www.bookstack.cn/read/kubernetes-practice-guide/$troubleshooting-problems-errors-troubleshooting-handle-pid-full.md)





## 其它排错

### Job 无法被删除

#### 原因

- 可能是 k8s 的一个bug: https://github.com/kubernetes/kubernetes/issues/43168
- 本质上是脏数据问题，Running+Succeed != 期望Completions 数量，低版本 kubectl 不容忍，delete job 的时候打开debug(加-v=8)，会看到kubectl不断在重试，直到达到timeout时间。新版kubectl会容忍这些，删除job时会删除关联的pod



#### 解决方法

1. 升级 kubectl 版本，1.12 以上
2. 低版本 kubectl 删除 job 时带 `--cascade=false` 参数(如果job关联的pod没删完，加这个参数不会删除关联的pod)

```sh
$ kubectl delete job --cascade=false  <job name>
```



### kubectl 执行 exec 或 logs 失败

通常是 apiserver —> kubelet:10250 之间的网络不通，10250 是 kubelet 提供接口的端口，`kubectl exec` 和 `kubectl logs` 的原理就是 apiserver 调 kubelet，kubelet 再调运行时 (比如 dockerd) 来实现的，所以要保证 kubelet 10250 端口对 apiserver 放通。检查防火墙、iptables 规则是否对 10250 端口或某些 IP 进行了拦截。



### 内核软死锁

#### 内核报错

```sh
Oct 14 15:13:05 VM_1_6_centos kernel: NMI watchdog: BUG: soft lockup - CPU#5 stuck for 22s! [runc:[1:CHILD]:2274]
```



#### 原因

发生这个报错通常是内核繁忙 (扫描、释放或分配大量对象)，分不出时间片给用户态进程导致的，也伴随着高负载，如果负载降低报错则会消失。



#### 什么情况下会导致内核繁忙

- 短时间内创建大量进程 (可能是业务需要，也可能是业务bug或用法不正确导致创建大量进程)



#### 参考资料

- What are all these “Bug: soft lockup” messages about : https://www.suse.com/support/kb/doc/?id=7017652





# 处理实践



## 高负载



节点高负载会导致进程无法获得足够的 cpu 时间片来运行，通常表现为网络 timeout，健康检查失败，服务不可用。



### 过多 IO 等待

有时候即便 cpu ‘us’ (user) 不高但 cpu ‘id’ (idle) 很高的情况节点负载也很高，这是为什么呢？通常是文件 IO 性能达到瓶颈导致 IO WAIT 过多，从而使得节点整体负载升高，影响其它进程的性能。

使用 `top` 命令看下当前负载：

```sh
top - 19:42:06 up 23:59,  2 users,  load average: 34.64, 35.80, 35.76
Tasks: 679 total,   1 running, 678 sleeping,   0 stopped,   0 zombie
Cpu(s): 15.6%us,  1.7%sy,  0.0%ni, 74.7%id,  7.9%wa,  0.0%hi,  0.1%si,  0.0%st
Mem:  32865032k total, 30989168k used,  1875864k free,   370748k buffers
Swap:  8388604k total,     5440k used,  8383164k free,  7982424k cached
  PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND
 9783 mysql     20   0 17.3g  16g 8104 S 186.9 52.3   3752:33 mysqld
 5700 nginx     20   0 1330m  66m 9496 S  8.9  0.2   0:20.82 php-fpm
 6424 nginx     20   0 1330m  65m 8372 S  8.3  0.2   0:04.97 php-fpm
 6573 nginx     20   0 1330m  64m 7368 S  8.3  0.2   0:01.49 php-fpm
 5927 nginx     20   0 1320m  56m 9272 S  7.6  0.2   0:12.54 php-fpm
 5956 nginx     20   0 1330m  65m 8500 S  7.6  0.2   0:12.70 php-fpm
 6126 nginx     20   0 1321m  57m 8964 S  7.3  0.2   0:09.72 php-fpm
 6127 nginx     20   0 1319m  54m 9520 S  6.6  0.2   0:08.73 php-fpm
 6131 nginx     20   0 1320m  56m 9404 S  6.6  0.2   0:09.43 php-fpm
 6174 nginx     20   0 1321m  56m 8444 S  6.3  0.2   0:08.92 php-fpm
 5790 nginx     20   0 1319m  54m 9468 S  5.6  0.2   0:17.33 php-fpm
 6575 nginx     20   0 1320m  55m 8212 S  5.6  0.2   0:02.11 php-fpm
 6160 nginx     20   0 1310m  44m 8296 S  4.0  0.1   0:10.05 php-fpm
 5597 nginx     20   0 1310m  46m 9556 S  3.6  0.1   0:21.03 php-fpm
 5786 nginx     20   0 1310m  45m 8528 S  3.6  0.1   0:15.53 php-fpm
 5797 nginx     20   0 1310m  46m 9444 S  3.6  0.1   0:14.02 php-fpm
 6158 nginx     20   0 1310m  45m 8324 S  3.6  0.1   0:10.20 php-fpm
 5698 nginx     20   0 1310m  46m 9184 S  3.3  0.1   0:20.62 php-fpm
 5779 nginx     20   0 1309m  44m 8336 S  3.3  0.1   0:15.34 php-fpm
 6540 nginx     20   0 1306m  40m 7884 S  3.3  0.1   0:02.46 php-fpm
 5553 nginx     20   0 1300m  36m 9568 S  3.0  0.1   0:21.58 php-fpm
 5722 nginx     20   0 1310m  45m 8552 S  3.0  0.1   0:17.25 php-fpm
 5920 nginx     20   0 1302m  36m 8208 S  3.0  0.1   0:14.23 php-fpm
 6432 nginx     20   0 1310m  45m 8420 S  3.0  0.1   0:05.86 php-fpm
 5285 nginx     20   0 1302m  38m 9696 S  2.7  0.1   0:23.41 php-fpm
```

`wa` (wait) 表示 IO WAIT 的 cpu 占用，默认看到的是所有核的平均值，要看每个核的 `wa` 值需要按下 “1”:

```sh
top - 19:42:08 up 23:59,  2 users,  load average: 34.64, 35.80, 35.76
Tasks: 679 total,   1 running, 678 sleeping,   0 stopped,   0 zombie
Cpu0  : 29.5%us,  3.7%sy,  0.0%ni, 48.7%id, 17.9%wa,  0.0%hi,  0.1%si,  0.0%st
Cpu1  : 29.3%us,  3.7%sy,  0.0%ni, 48.9%id, 17.9%wa,  0.0%hi,  0.1%si,  0.0%st
Cpu2  : 26.1%us,  3.1%sy,  0.0%ni, 64.4%id,  6.0%wa,  0.0%hi,  0.3%si,  0.0%st
Cpu3  : 25.9%us,  3.1%sy,  0.0%ni, 65.5%id,  5.4%wa,  0.0%hi,  0.1%si,  0.0%st
Cpu4  : 24.9%us,  3.0%sy,  0.0%ni, 66.8%id,  5.0%wa,  0.0%hi,  0.3%si,  0.0%st
Cpu5  : 24.9%us,  2.9%sy,  0.0%ni, 67.0%id,  4.8%wa,  0.0%hi,  0.3%si,  0.0%st
Cpu6  : 24.2%us,  2.7%sy,  0.0%ni, 68.3%id,  4.5%wa,  0.0%hi,  0.3%si,  0.0%st
Cpu7  : 24.3%us,  2.6%sy,  0.0%ni, 68.5%id,  4.2%wa,  0.0%hi,  0.3%si,  0.0%st
Cpu8  : 23.8%us,  2.6%sy,  0.0%ni, 69.2%id,  4.1%wa,  0.0%hi,  0.3%si,  0.0%st
Cpu9  : 23.9%us,  2.5%sy,  0.0%ni, 69.3%id,  4.0%wa,  0.0%hi,  0.3%si,  0.0%st
Cpu10 : 23.3%us,  2.4%sy,  0.0%ni, 68.7%id,  5.6%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu11 : 23.3%us,  2.4%sy,  0.0%ni, 69.2%id,  5.1%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu12 : 21.8%us,  2.4%sy,  0.0%ni, 60.2%id, 15.5%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu13 : 21.9%us,  2.4%sy,  0.0%ni, 60.6%id, 15.2%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu14 : 21.4%us,  2.3%sy,  0.0%ni, 72.6%id,  3.7%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu15 : 21.5%us,  2.2%sy,  0.0%ni, 73.2%id,  3.1%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu16 : 21.2%us,  2.2%sy,  0.0%ni, 73.6%id,  3.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu17 : 21.2%us,  2.1%sy,  0.0%ni, 73.8%id,  2.8%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu18 : 20.9%us,  2.1%sy,  0.0%ni, 74.1%id,  2.9%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu19 : 21.0%us,  2.1%sy,  0.0%ni, 74.4%id,  2.5%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu20 : 20.7%us,  2.0%sy,  0.0%ni, 73.8%id,  3.4%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu21 : 20.8%us,  2.0%sy,  0.0%ni, 73.9%id,  3.2%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu22 : 20.8%us,  2.0%sy,  0.0%ni, 74.4%id,  2.8%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu23 : 20.8%us,  1.9%sy,  0.0%ni, 74.4%id,  2.8%wa,  0.0%hi,  0.0%si,  0.0%st
Mem:  32865032k total, 30209248k used,  2655784k free,   370748k buffers
Swap:  8388604k total,     5440k used,  8383164k free,  7986552k cached
```

`wa` 通常是 0%，如果经常在 1 之上，说明存储设备的速度已经太慢，无法跟上 cpu 的处理速度。

使用 `atop` 看下当前磁盘 IO 状态:

```sh
ATOP - lemp              2017/01/23  19:42:32              ---------                10s elapsed
PRC | sys    3.18s | user  33.24s | #proc    679 | #tslpu    28 | #zombie    0 | #exit      0 |
CPU | sys      29% | user    330% | irq       1% | idle   1857% | wait    182% | curscal  69% |
CPL | avg1   33.00 | avg5   35.29 | avg15  35.59 | csw    62610 | intr   76926 | numcpu    24 |
MEM | tot    31.3G | free    2.1G | cache   7.6G | dirty  41.0M | buff  362.1M | slab    1.2G |
SWP | tot     8.0G | free    8.0G |              |              | vmcom  23.9G | vmlim  23.7G |
DSK |          sda | busy    100% | read       4 | write   1789 | MBw/s   2.84 | avio 5.58 ms |
NET | transport    | tcpi   10357 | tcpo    9065 | udpi       0 | udpo       0 | tcpao    174 |
NET | network      | ipi    10360 | ipo     9065 | ipfrw      0 | deliv  10359 | icmpo      0 |
NET | eth0      4% | pcki    6649 | pcko    6136 | si 1478 Kbps | so 4115 Kbps | erro       0 |
NET | lo      ---- | pcki    4082 | pcko    4082 | si 8967 Kbps | so 8967 Kbps | erro       0 |
  PID   TID  THR  SYSCPU  USRCPU  VGROW  RGROW  RDDSK  WRDSK ST EXC S CPUNR  CPU CMD       1/12
 9783     -  156   0.21s  19.44s     0K  -788K     4K  1344K --   - S     4 197% mysqld
 5596     -    1   0.10s   0.62s 47204K 47004K     0K   220K --   - S    18   7% php-fpm
 6429     -    1   0.06s   0.34s 19840K 19968K     0K     0K --   - S    21   4% php-fpm
 6210     -    1   0.03s   0.30s -5216K -5204K     0K     0K --   - S    19   3% php-fpm
 5757     -    1   0.05s   0.27s 26072K 26012K     0K     4K --   - S    13   3% php-fpm
 6433     -    1   0.04s   0.28s -2816K -2816K     0K     0K --   - S    11   3% php-fpm
 5846     -    1   0.06s   0.22s -2560K -2660K     0K     0K --   - S     7   3% php-fpm
 5791     -    1   0.05s   0.21s  5764K  5692K     0K     0K --   - S    22   3% php-fpm
 5860     -    1   0.04s   0.21s 48088K 47724K     0K     0K --   - S     1   3% php-fpm
 6231     -    1   0.04s   0.20s  -256K    -4K     0K     0K --   - S     1   2% php-fpm
 6154     -    1   0.03s   0.21s -3004K -3184K     0K     0K --   - S    21   2% php-fpm
 6573     -    1   0.04s   0.20s  -512K  -168K     0K     0K --   - S     4   2% php-fpm
 6435     -    1   0.04s   0.19s -3216K -2980K     0K     0K --   - S    15   2% php-fpm
 5954     -    1   0.03s   0.20s     0K   164K     0K     4K --   - S     0   2% php-fpm
 6133     -    1   0.03s   0.19s 41056K 40432K     0K     0K --   - S    18   2% php-fpm
 6132     -    1   0.02s   0.20s 37836K 37440K     0K     0K --   - S    11   2% php-fpm
 6242     -    1   0.03s   0.19s -12.2M -12.3M     0K     4K --   - S    12   2% php-fpm
 6285     -    1   0.02s   0.19s 39516K 39420K     0K     0K --   - S     3   2% php-fpm
 6455     -    1   0.05s   0.16s 29008K 28560K     0K     0K --   - S    14   2% php-fpm
```

在本例中磁盘 `sda` 已经 100% busy，已经严重达到性能瓶颈。按 ‘d’ 看下是哪些进程在使用磁盘IO:

```sh
ATOP - lemp               2017/01/23  19:42:46               ---------               2s elapsed
PRC | sys    0.24s | user   1.99s | #proc    679 | #tslpu    54 | #zombie    0 | #exit      0 |
CPU | sys      11% | user    101% | irq       1% | idle   2089% | wait    208% | curscal  63% |
CPL | avg1   38.49 | avg5   36.48 | avg15  35.98 | csw     4654 | intr    6876 | numcpu    24 |
MEM | tot    31.3G | free    2.2G | cache   7.6G | dirty  48.7M | buff  362.1M | slab    1.2G |
SWP | tot     8.0G | free    8.0G |              |              | vmcom  23.9G | vmlim  23.7G |
DSK |          sda | busy    100% | read       2 | write    362 | MBw/s   2.28 | avio 5.49 ms |
NET | transport    | tcpi    1031 | tcpo     968 | udpi       0 | udpo       0 | tcpao     45 |
NET | network      | ipi     1031 | ipo      968 | ipfrw      0 | deliv   1031 | icmpo      0 |
NET | eth0      1% | pcki     558 | pcko     508 | si  762 Kbps | so 1077 Kbps | erro       0 |
NET | lo      ---- | pcki     406 | pcko     406 | si 2273 Kbps | so 2273 Kbps | erro       0 |
  PID          TID         RDDSK         WRDSK        WCANCL         DSK        CMD         1/5
 9783            -            0K          468K           16K         40%        mysqld
 1930            -            0K          212K            0K         18%        flush-8:0
 5896            -            0K          152K            0K         13%        nginx
  880            -            0K          148K            0K         13%        jbd2/sda5-8
 5909            -            0K           60K            0K          5%        nginx
 5906            -            0K           36K            0K          3%        nginx
 5907            -           16K            8K            0K          2%        nginx
 5903            -           20K            0K            0K          2%        nginx
 5901            -            0K           12K            0K          1%        nginx
 5908            -            0K            8K            0K          1%        nginx
 5894            -            0K            8K            0K          1%        nginx
 5911            -            0K            8K            0K          1%        nginx
 5900            -            0K            4K            4K          0%        nginx
 5551            -            0K            4K            0K          0%        php-fpm
 5913            -            0K            4K            0K          0%        nginx
 5895            -            0K            4K            0K          0%        nginx
 6133            -            0K            0K            0K          0%        php-fpm
 5780            -            0K            0K            0K          0%        php-fpm
 6675            -            0K            0K            0K          0%        atop
```

也可以使用 `iotop -oPa` 查看哪些进程占用磁盘 IO:

```sh
Total DISK READ: 15.02 K/s | Total DISK WRITE: 3.82 M/s
  PID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN     IO>    COMMAND
 1930 be/4 root          0.00 B   1956.00 K  0.00 % 83.34 % [flush-8:0]
 5914 be/4 nginx         0.00 B      0.00 B  0.00 % 36.56 % nginx: cache manager process
  880 be/3 root          0.00 B     21.27 M  0.00 % 35.03 % [jbd2/sda5-8]
 5913 be/2 nginx        36.00 K   1000.00 K  0.00 %  8.94 % nginx: worker process
 5910 be/2 nginx         0.00 B   1048.00 K  0.00 %  8.43 % nginx: worker process
 5896 be/2 nginx        56.00 K    452.00 K  0.00 %  6.91 % nginx: worker process
 5909 be/2 nginx        20.00 K   1144.00 K  0.00 %  6.24 % nginx: worker process
 5890 be/2 nginx        48.00 K    692.00 K  0.00 %  6.07 % nginx: worker process
 5892 be/2 nginx        84.00 K    736.00 K  0.00 %  5.71 % nginx: worker process
 5901 be/2 nginx        20.00 K    504.00 K  0.00 %  5.46 % nginx: worker process
 5899 be/2 nginx         0.00 B    596.00 K  0.00 %  5.14 % nginx: worker process
 5897 be/2 nginx        28.00 K   1388.00 K  0.00 %  4.90 % nginx: worker process
 5908 be/2 nginx        48.00 K    700.00 K  0.00 %  4.43 % nginx: worker process
 5905 be/2 nginx        32.00 K   1140.00 K  0.00 %  4.36 % nginx: worker process
 5900 be/2 nginx         0.00 B   1208.00 K  0.00 %  4.31 % nginx: worker process
 5904 be/2 nginx        36.00 K   1244.00 K  0.00 %  2.80 % nginx: worker process
 5895 be/2 nginx        16.00 K    780.00 K  0.00 %  2.50 % nginx: worker process
 5907 be/2 nginx         0.00 B   1548.00 K  0.00 %  2.43 % nginx: worker process
 5903 be/2 nginx        36.00 K   1032.00 K  0.00 %  2.34 % nginx: worker process
 6130 be/4 nginx         0.00 B     72.00 K  0.00 %  2.18 % php-fpm: pool www
 5906 be/2 nginx        12.00 K    844.00 K  0.00 %  2.10 % nginx: worker process
 5889 be/2 nginx        40.00 K   1164.00 K  0.00 %  2.00 % nginx: worker process
 5894 be/2 nginx        44.00 K    760.00 K  0.00 %  1.61 % nginx: worker process
 5902 be/2 nginx        52.00 K    992.00 K  0.00 %  1.55 % nginx: worker process
 5893 be/2 nginx        64.00 K    972.00 K  0.00 %  1.22 % nginx: worker process
 5814 be/4 nginx        36.00 K     44.00 K  0.00 %  1.06 % php-fpm: pool www
 6159 be/4 nginx         4.00 K      4.00 K  0.00 %  1.00 % php-fpm: pool www
 5693 be/4 nginx         0.00 B      4.00 K  0.00 %  0.86 % php-fpm: pool www
 5912 be/2 nginx        68.00 K    300.00 K  0.00 %  0.72 % nginx: worker process
 5911 be/2 nginx        20.00 K    788.00 K  0.00 %  0.72 % nginx: worker process
```

通过 `man iotop` 可以看下这几个参数的含义：

```sh
-o, --only
       Only show processes or threads actually doing I/O, instead of showing all processes or threads. This can be dynamically toggled by pressing o.
-P, --processes
       Only show processes. Normally iotop shows all threads.
-a, --accumulated
       Show accumulated I/O instead of bandwidth. In this mode, iotop shows the amount of I/O processes have done since iotop started.
```



### 节点上部署了其它非 K8S 管理的服务

比如在节点上装了数据库，但不被 K8S 所管理，这是用法不正确，不建议在 K8S 节点上部署其它进程。



### 参考资料

- Linux server performance: Is disk I/O slowing your application: https://haydenjames.io/linux-server-performance-disk-io-slowing-application/



## 内存碎片化

### 判断是否内存碎片化严重

内存页分配失败，内核日志报类似下面的错：

```sh
mysqld: page allocation failure. order:4, mode:0x10c0d0
```

- `mysqld` 是被分配的内存的程序
- `order` 表示需要分配连续页的数量(2^order)，这里 4 表示 2^4=16 个连续的页
- `mode` 是内存分配模式的标识，定义在内核源码文件 `include/linux/gfp.h` 中，通常是多个标识相与运算的结果，不同版本内核可能不一样，比如在新版内核中 `GFP_KERNEL` 是 `__GFP_RECLAIM | __GFP_IO | __GFP_FS` 的运算结果，而 `__GFP_RECLAIM` 又是 `___GFP_DIRECT_RECLAIM|___GFP_KSWAPD_RECLAIM` 的运算结果

当 order 为 0 时，说明系统以及完全没有可用内存了，order 值比较大时，才说明内存碎片化了，无法分配连续的大页内存。



### 内存碎片化造成的问题

#### 容器启动失败

K8S 会为每个 pod 创建 netns 来隔离 network namespace，内核初始化 netns 时会为其创建 nf_conntrack 表的 cache，需要申请大页内存，如果此时系统内存已经碎片化，无法分配到足够的大页内存内核就会报错(`v2.6.33 - v4.6`):

```sh
runc:[1:CHILD]: page allocation failure: order:6, mode:0x10c0d0
```

Pod 状态将会一直在 ContainerCreating，dockerd 启动容器失败，日志报错:

```sh
Jan 23 14:15:31 dc05 dockerd: time="2019-01-23T14:15:31.288446233+08:00" level=error msg="containerd: start container" error="oci runtime error: container_linux.go:247: starting container process caused \"process_linux.go:245: running exec setns process for init caused \\\"exit status 6\\\"\"\n" id=5b9be8c5bb121264899fac8d9d36b02150269d41ce96ba6ad36d70b8640cb01c
Jan 23 14:15:31 dc05 dockerd: time="2019-01-23T14:15:31.317965799+08:00" level=error msg="Create container failed with error: invalid header field value \"oci runtime error: container_linux.go:247: starting container process caused \\\"process_linux.go:245: running exec setns process for init caused \\\\\\\"exit status 6\\\\\\\"\\\"\\n\""
```

kubelet 日志报错:

```sh
Jan 23 14:15:31 dc05 kubelet: E0123 14:15:31.352386   26037 remote_runtime.go:91] RunPodSandbox from runtime service failed: rpc error: code = 2 desc = failed to start sandbox container for pod "matchdataserver-1255064836-t4b2w": Error response from daemon: {"message":"invalid header field value \"oci runtime error: container_linux.go:247: starting container process caused \\\"process_linux.go:245: running exec setns process for init caused \\\\\\\"exit status 6\\\\\\\"\\\"\\n\""}
Jan 23 14:15:31 dc05 kubelet: E0123 14:15:31.352496   26037 kuberuntime_sandbox.go:54] CreatePodSandbox for pod "matchdataserver-1255064836-t4b2w_basic(485fd485-1ed6-11e9-8661-0a587f8021ea)" failed: rpc error: code = 2 desc = failed to start sandbox container for pod "matchdataserver-1255064836-t4b2w": Error response from daemon: {"message":"invalid header field value \"oci runtime error: container_linux.go:247: starting container process caused \\\"process_linux.go:245: running exec setns process for init caused \\\\\\\"exit status 6\\\\\\\"\\\"\\n\""}
Jan 23 14:15:31 dc05 kubelet: E0123 14:15:31.352518   26037 kuberuntime_manager.go:618] createPodSandbox for pod "matchdataserver-1255064836-t4b2w_basic(485fd485-1ed6-11e9-8661-0a587f8021ea)" failed: rpc error: code = 2 desc = failed to start sandbox container for pod "matchdataserver-1255064836-t4b2w": Error response from daemon: {"message":"invalid header field value \"oci runtime error: container_linux.go:247: starting container process caused \\\"process_linux.go:245: running exec setns process for init caused \\\\\\\"exit status 6\\\\\\\"\\\"\\n\""}
Jan 23 14:15:31 dc05 kubelet: E0123 14:15:31.352580   26037 pod_workers.go:182] Error syncing pod 485fd485-1ed6-11e9-8661-0a587f8021ea ("matchdataserver-1255064836-t4b2w_basic(485fd485-1ed6-11e9-8661-0a587f8021ea)"), skipping: failed to "CreatePodSandbox" for "matchdataserver-1255064836-t4b2w_basic(485fd485-1ed6-11e9-8661-0a587f8021ea)" with CreatePodSandboxError: "CreatePodSandbox for pod \"matchdataserver-1255064836-t4b2w_basic(485fd485-1ed6-11e9-8661-0a587f8021ea)\" failed: rpc error: code = 2 desc = failed to start sandbox container for pod \"matchdataserver-1255064836-t4b2w\": Error response from daemon: {\"message\":\"invalid header field value \\\"oci runtime error: container_linux.go:247: starting container process caused \\\\\\\"process_linux.go:245: running exec setns process for init caused \\\\\\\\\\\\\\\"exit status 6\\\\\\\\\\\\\\\"\\\\\\\"\\\\n\\\"\"}"
Jan 23 14:15:31 dc05 kubelet: I0123 14:15:31.372181   26037 kubelet.go:1916] SyncLoop (PLEG): "matchdataserver-1255064836-t4b2w_basic(485fd485-1ed6-11e9-8661-0a587f8021ea)", event: &pleg.PodLifecycleEvent{ID:"485fd485-1ed6-11e9-8661-0a587f8021ea", Type:"ContainerDied", Data:"5b9be8c5bb121264899fac8d9d36b02150269d41ce96ba6ad36d70b8640cb01c"}
Jan 23 14:15:31 dc05 kubelet: W0123 14:15:31.372225   26037 pod_container_deletor.go:77] Container "5b9be8c5bb121264899fac8d9d36b02150269d41ce96ba6ad36d70b8640cb01c" not found in pod's containers
Jan 23 14:15:31 dc05 kubelet: I0123 14:15:31.678211   26037 kuberuntime_manager.go:383] No ready sandbox for pod "matchdataserver-1255064836-t4b2w_basic(485fd485-1ed6-11e9-8661-0a587f8021ea)" can be found. Need to start a new one
```

查看slab (后面的0多表示伙伴系统没有大块内存了)：

```sh
$ cat /proc/buddyinfo
Node 0, zone      DMA      1      0      1      0      2      1      1      0      1      1      3
Node 0, zone    DMA32   2725    624    489    178      0      0      0      0      0      0      0
Node 0, zone   Normal   1163   1101    932    222      0      0      0      0      0      0      0
```



#### 系统 OOM

内存碎片化会导致即使当前系统总内存比较多，但由于无法分配足够的大页内存导致给进程分配内存失败，就认为系统内存不够用，需要杀掉一些进程来释放内存，从而导致系统 OOM



### 解决方法

- 周期性地或者在发现大块内存不足时，先进行drop_cache操作:

```sh
echo 3 > /proc/sys/vm/drop_caches
```

- 必要时候进行内存整理，开销会比较大，会造成业务卡住一段时间(慎用):

```sh
echo 1 > /proc/sys/vm/compact_memory
```



### 附录

相关链接：

- https://huataihuang.gitbooks.io/cloud-atlas/content/os/linux/kernel/memory/drop_caches_and_compact_memory.html





## 磁盘爆满

### 什么情况下磁盘可能会爆满 ？

kubelet 有 gc 和驱逐机制，通过 `--image-gc-high-threshold`, `--image-gc-low-threshold`, `--eviction-hard`, `--eviction-soft`, `--eviction-minimum-reclaim` 等参数控制 kubelet 的 gc 和驱逐策略来释放磁盘空间，如果配置正确的情况下，磁盘一般不会爆满。

通常导致爆满的原因可能是配置不正确或者节点上有其它非 K8S 管理的进程在不断写数据到磁盘占用大量空间导致磁盘爆满。



### 磁盘爆满会有什么影响 ？

影响 K8S 运行我们主要关注 kubelet 和容器运行时这两个最关键的组件，它们所使用的目录通常不一样，kubelet 一般不会单独挂盘，直接使用系统磁盘，因为通常占用空间不会很大，容器运行时单独挂盘的场景比较多，当磁盘爆满的时候我们也要看 kubelet 和 容器运行时使用的目录是否在这个磁盘，通过 `df` 命令可以查看磁盘挂载点。



#### 容器运行时使用的目录所在磁盘爆满

如果容器运行时使用的目录所在磁盘空间爆满，可能会造成容器运行时无响应，比如 docker，执行 docker 相关的命令一直 hang 住， kubelet 日志也可以看到 PLEG unhealthy，因为 CRI 调用 timeout，当然也就无法创建或销毁容器，通常表现是 Pod 一直 ContainerCreating 或 一直 Terminating。

docker 默认使用的目录主要有:

- `/var/run/docker`: 用于存储容器运行状态，通过 dockerd 的 `--exec-root` 参数指定。
- `/var/lib/docker`: 用于持久化容器相关的数据，比如容器镜像、容器可写层数据、容器标准日志输出、通过 docker 创建的 volume 等

Pod 启动可能报类似下面的事件:

```sh
  Warning  FailedCreatePodSandBox    53m                 kubelet, 172.22.0.44  Failed create pod sandbox: rpc error: code = DeadlineExceeded desc = context deadline exceeded
  Warning  FailedCreatePodSandBox  2m (x4307 over 16h)  kubelet, 10.179.80.31  (combined from similar events): Failed create pod sandbox: rpc error: code = Unknown desc = failed to create a sandbox for pod "apigateway-6dc48bf8b6-l8xrw": Error response from daemon: mkdir /var/lib/docker/aufs/mnt/1f09d6c1c9f24e8daaea5bf33a4230de7dbc758e3b22785e8ee21e3e3d921214-init: no space left on device
  Warning  Failed   5m1s (x3397 over 17h)  kubelet, ip-10-0-151-35.us-west-2.compute.internal  (combined from similar events): Error: container create failed: container_linux.go:336: starting container process caused "process_linux.go:399: container init caused \"rootfs_linux.go:58: mounting \\\"/sys\\\" to rootfs \\\"/var/lib/dockerd/storage/overlay/051e985771cc69f3f699895a1dada9ef6483e912b46a99e004af7bb4852183eb/merged\\\" at \\\"/var/lib/dockerd/storage/overlay/051e985771cc69f3f699895a1dada9ef6483e912b46a99e004af7bb4852183eb/merged/sys\\\" caused \\\"no space left on device\\\"\""
```

Pod 删除可能报类似下面的事件:

```sh
Normal  Killing  39s (x735 over 15h)  kubelet, 10.179.80.31  Killing container with id docker://apigateway:Need to kill Pod
```



#### kubelet 使用的目录所在磁盘爆满

如果 kubelet 使用的目录所在磁盘空间爆满(通常是系统盘)，新建 Pod 时连 Sandbox 都无法创建成功，因为 mkdir 将会失败，通常会有类似这样的 Pod 事件:

```sh
  Warning  UnexpectedAdmissionError  44m                 kubelet, 172.22.0.44  Update plugin resources failed due to failed to write checkpoint file "kubelet_internal_checkpoint": write /var/lib/kubelet/device-plugins/.728425055: no space left on device, which is unexpected.
```

kubelet 默认使用的目录是 `/var/lib/kubelet`， 用于存储插件信息、Pod 相关的状态以及挂载的 volume (比如 `emptyDir`, `ConfigMap`, `Secret`)，通过 kubelet 的 `--root-dir` 参数指定。



### 如何分析磁盘占用 ?

- 如果运行时使用的是 Docker，请参考本书 排错技巧: 分析 Docker 磁盘占用



### 如何恢复 ？

如果容器运行时使用的 Docker，我们无法直接重启 dockerd 来释放一些空间，因为磁盘爆满后 dockerd 无法正常响应，停止的时候也会卡住。我们需要先手动清理一点文件腾出空间好让 dockerd 能够停止并重启。

可以手动删除一些 docker 的 log 文件或可写层文件，通常删除 log:

```sh
$ cd /var/lib/docker/containers
$ du -sh * # 找到比较大的目录
$ cd dda02c9a7491fa797ab730c1568ba06cba74cecd4e4a82e9d90d00fa11de743c
$ cat /dev/null > dda02c9a7491fa797ab730c1568ba06cba74cecd4e4a82e9d90d00fa11de743c-json.log.9 # 删除log文件
```

- **注意:** 使用 `cat /dev/null >` 方式删除而不用 `rm`，因为用 rm 删除的文件，docker 进程可能不会释放文件，空间也就不会释放；log 的后缀数字越大表示越久远，先删除旧日志。

然后将该 node 标记不可调度，并将其已有的 pod 驱逐到其它节点，这样重启 dockerd 就会让该节点的 pod 对应的容器删掉，容器相关的日志(标准输出)与容器内产生的数据文件(没有挂载 volume, 可写层)也会被清理：

```sh
$ kubectl drain <node-name>
```

重启 dockerd:

```sh
systemctl restart dockerd
# or systemctl restart docker
```

等重启恢复，pod 调度到其它节点，排查磁盘爆满原因并清理和规避，然后取消节点不可调度标记:

```sh
$ kubectl uncordon <node-name>
```



### 如何规避 ？

正确配置 kubelet gc 和 驱逐相关的参数，即便到达爆满地步，此时节点上的 pod 也都早就自动驱逐到其它节点了，不会存在 Pod 一直 ContainerCreating 或 Terminating 的问题。





## inotify watch 耗尽

每个 linux 进程可以持有多个 fd，每个 inotify 类型的 fd 可以 watch 多个目录，每个用户下所有进程 inotify 类型的 fd 可以 watch 的总目录数有个最大限制，这个限制可以通过内核参数配置: `fs.inotify.max_user_watches`

查看最大 inotify watch 数:

```sh
$ cat /proc/sys/fs/inotify/max_user_watches
8192
```

使用下面的脚本查看当前有 inotify watch 类型 fd 的进程以及每个 fd watch 的目录数量，降序输出，带总数统计:

```sh
#!/usr/bin/env bash
#
# Copyright 2019 (c) roc
#
# This script shows processes holding the inotify fd, alone with HOW MANY directories each inotify fd watches(0 will be ignored).
total=0
result="EXE PID FD-INFO INOTIFY-WATCHES\n"
while read pid fd; do \
  exe="$(readlink -f /proc/$pid/exe || echo n/a)"; \
  fdinfo="/proc/$pid/fdinfo/$fd" ; \
  count="$(grep -c inotify "$fdinfo" || true)"; \
  if [ $((count)) != 0 ]; then
    total=$((total+count)); \
    result+="$exe $pid $fdinfo $count\n"; \
  fi
done <<< "$(lsof +c 0 -n -P -u root|awk '/inotify$/ { gsub(/[urw]$/,"",$4); print $2" "$4 }')" && echo "total $total inotify watches" && result="$(echo -e $result|column -t)\n" && echo -e "$result" | head -1 && echo -e "$result" | sed "1d" | sort -k 4rn;
```

示例输出:

```sh
total 7882 inotify watches
EXE                                         PID    FD-INFO                INOTIFY-WATCHES
/usr/local/qcloud/YunJing/YDEyes/YDService  25813  /proc/25813/fdinfo/8   7077
/usr/bin/kubelet                            1173   /proc/1173/fdinfo/22   665
/usr/bin/ruby2.3                            13381  /proc/13381/fdinfo/14  54
/usr/lib/policykit-1/polkitd                1458   /proc/1458/fdinfo/9    14
/lib/systemd/systemd-udevd                  450    /proc/450/fdinfo/9     13
/usr/sbin/nscd                              7935   /proc/7935/fdinfo/3    6
/usr/bin/kubelet                            1173   /proc/1173/fdinfo/28   5
/lib/systemd/systemd                        1      /proc/1/fdinfo/17      4
/lib/systemd/systemd                        1      /proc/1/fdinfo/18      4
/lib/systemd/systemd                        1      /proc/1/fdinfo/26      4
/lib/systemd/systemd                        1      /proc/1/fdinfo/28      4
/usr/lib/policykit-1/polkitd                1458   /proc/1458/fdinfo/8    4
/usr/local/bin/sidecar-injector             4751   /proc/4751/fdinfo/3    3
/usr/lib/accountsservice/accounts-daemon    1178   /proc/1178/fdinfo/7    2
/usr/local/bin/galley                       8228   /proc/8228/fdinfo/10   2
/usr/local/bin/galley                       8228   /proc/8228/fdinfo/9    2
/lib/systemd/systemd                        1      /proc/1/fdinfo/11      1
/sbin/agetty                                1437   /proc/1437/fdinfo/4    1
/sbin/agetty                                1440   /proc/1440/fdinfo/4    1
/usr/bin/kubelet                            1173   /proc/1173/fdinfo/10   1
/usr/local/bin/envoy                        4859   /proc/4859/fdinfo/5    1
/usr/local/bin/envoy                        5427   /proc/5427/fdinfo/5    1
/usr/local/bin/envoy                        6058   /proc/6058/fdinfo/3    1
/usr/local/bin/envoy                        6893   /proc/6893/fdinfo/3    1
/usr/local/bin/envoy                        6950   /proc/6950/fdinfo/3    1
/usr/local/bin/galley                       8228   /proc/8228/fdinfo/3    1
/usr/local/bin/pilot-agent                  3819   /proc/3819/fdinfo/5    1
/usr/local/bin/pilot-agent                  4244   /proc/4244/fdinfo/5    1
/usr/local/bin/pilot-agent                  5901   /proc/5901/fdinfo/3    1
/usr/local/bin/pilot-agent                  6789   /proc/6789/fdinfo/3    1
/usr/local/bin/pilot-agent                  6808   /proc/6808/fdinfo/3    1
/usr/local/bin/pilot-discovery              6231   /proc/6231/fdinfo/3    1
/usr/local/bin/sidecar-injector             4751   /proc/4751/fdinfo/5    1
/usr/sbin/acpid                             1166   /proc/1166/fdinfo/6    1
/usr/sbin/dnsmasq                           7572   /proc/7572/fdinfo/8    1
```

如果看到总 watch 数比较大，接近最大限制，可以修改内核参数调高下这个限制。

临时调整:

```sh
sudo sysctl fs.inotify.max_user_watches=524288
```

永久生效:

```sh
echo "fs.inotify.max_user_watches=524288" >> /etc/sysctl.conf && sysctl -p
```

打开 inotify_add_watch 跟踪，进一步 debug inotify watch 耗尽的原因:

```sh
echo 1 >> /sys/kernel/debug/tracing/events/syscalls/sys_exit_inotify_add_watch/enable
```





## PID 耗尽

### 如何判断 PID 耗尽

首先要确认当前的 PID 限制，检查全局 PID 最大限制:

```sh
cat /proc/sys/kernel/pid_max
```

也检查下当前用户是否还有 `ulimit` 限制最大进程数。

然后要确认当前实际 PID 数量，检查当前用户的 PID 数量:

```sh
ps -eLf | wc -l
```

如果发现实际 PID 数量接近最大限制说明 PID 就可能会爆满导致经常有进程无法启动，低版本内核可能报错: `Cannot allocate memory`，这个报错信息不准确，在内核 4.1 以后改进了: https://github.com/torvalds/linux/commit/35f71bc0a09a45924bed268d8ccd0d3407bc476f



### 如何解决

临时调大：

```sh
echo 65535 > /proc/sys/kernel/pid_max
```

永久调大:

```sh
echo "kernel.pid_max=65535 " >> /etc/sysctl.conf && sysctl -p
```

k8s 1.14 支持了限制 Pod 的进程数量:

 https://kubernetes.io/blog/2019/04/15/process-id-limiting-for-stability-improvements-in-kubernetes-1.14/





## arp_cache 溢出

### 如何判断 arp_cache 溢出？

内核日志会有有下面的报错:

```sh
arp_cache: neighbor table overflow!
```

查看当前 arp 记录数:

```sh
$ arp -an | wc -l
1335
```

查看 arp gc 阀值:

```sh
$ systectl -a | grep gc_thresh
net.ipv4.neigh.default.gc_thresh1 = 128
net.ipv4.neigh.default.gc_thresh2 = 512
net.ipv4.neigh.default.gc_thresh3 = 1024
net.ipv6.neigh.default.gc_thresh1 = 128
net.ipv6.neigh.default.gc_thresh2 = 512
net.ipv6.neigh.default.gc_thresh3 = 1024
```

当前 arp 记录数接近 `gc_thresh3` 比较容易 overflow，因为当 arp 记录达到 `gc_thresh3` 时会强制触发 gc 清理，当这时又有数据包要发送，并且根据目的 IP 在 arp cache 中没找到 mac 地址，这时会判断当前 arp cache 记录数加 1 是否大于 `gc_thresh3`，如果没有大于就会 时就会报错: `arp_cache: neighbor table overflow!`



### 解决方案

调整节点内核参数，将 arp cache 的 gc 阀值调高 (`/etc/sysctl.conf`):

```sh
net.ipv4.neigh.default.gc_thresh1 = 80000
net.ipv4.neigh.default.gc_thresh2 = 90000
net.ipv4.neigh.default.gc_thresh3 = 100000
```

分析是否只是部分业务的 Pod 的使用场景需要节点有比较大的 arp 缓存空间。

如果不是，就需要调整所有节点内核参数。

如果是，可以将部分 Node 打上标签，比如:

```sh
$ kubectl label node host1 arp_cache=large
```

然后用 nodeSelector 或 nodeAffnity 让这部分需要内核有大 arp_cache 容量的 Pod 只调度到这部分节点，推荐使用 nodeAffnity，yaml 示例:

```yaml
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: arp_cache
                operator: In
                values:
                - large
```





# 踩坑总结



## cgroup 泄露

### 内核 Bug

`memcg` 是 Linux 内核中用于管理 cgroup 内存的模块，整个生命周期应该是跟随 cgroup 的，但是在低版本内核中(已知3.10)，一旦给某个 memory cgroup 开启 kmem accounting 中的 `memory.kmem.limit_in_bytes` 就可能会导致不能彻底删除 memcg 和对应的 cssid，也就是说应用即使已经删除了 cgroup (`/sys/fs/cgroup/memory` 下对应的 cgroup 目录已经删除), 但在内核中没有释放 cssid，导致内核认为的 cgroup 的数量实际数量不一致，我们也无法得知内核认为的 cgroup 数量是多少。

关于 cgroup kernel memory，在 [kernel.org](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v1/memory.html#kernel-memory-extension-config-memcg-kmem) 中有如下描述：

```c
2.7 Kernel Memory Extension (CONFIG_MEMCG_KMEM)
-----------------------------------------------
With the Kernel memory extension, the Memory Controller is able to limit
the amount of kernel memory used by the system. Kernel memory is fundamentally
different than user memory, since it can't be swapped out, which makes it
possible to DoS the system by consuming too much of this precious resource.
Kernel memory accounting is enabled for all memory cgroups by default. But
it can be disabled system-wide by passing cgroup.memory=nokmem to the kernel
at boot time. In this case, kernel memory will not be accounted at all.
Kernel memory limits are not imposed for the root cgroup. Usage for the root
cgroup may or may not be accounted. The memory used is accumulated into
memory.kmem.usage_in_bytes, or in a separate counter when it makes sense.
(currently only for tcp).
The main "kmem" counter is fed into the main counter, so kmem charges will
also be visible from the user counter.
Currently no soft limit is implemented for kernel memory. It is future work
to trigger slab reclaim when those limits are reached.
```

这是一个 cgroup memory 的扩展，用于限制对 kernel memory 的使用，但该特性在老于 4.0 版本中是个实验特性，存在泄露问题，在 4.x 较低的版本也还有泄露问题，应该是造成泄露的代码路径没有完全修复，推荐 4.3 以上的内核。



### 造成容器创建失败

这个问题可能会导致创建容器失败，因为创建容器为其需要创建 cgroup 来做隔离，而低版本内核有个限制：允许创建的 cgroup 最大数量写死为 65535 ([点我跳转到 commit](https://github.com/torvalds/linux/commit/38460b48d06440de46b34cb778bd6c4855030754#diff-c04090c51d3c6700c7128e84c58b1291R3384))，如果节点上经常创建和销毁大量容器导致创建很多 cgroup，删除容器但没有彻底删除 cgroup 造成泄露(真实数量我们无法得知)，到达 65535 后再创建容器就会报创建 cgroup 失败并报错 `no space left on device`，使用 kubernetes 最直观的感受就是 pod 创建之后无法启动成功。

pod 启动失败，报 event 示例:

```sh
Events:
  Type     Reason                    Age                 From                   Message
  ----     ------                    ----                ----                   -------
  Normal   Scheduled                 15m                 default-scheduler      Successfully assigned jenkins/jenkins-7845b9b665-nrvks to 10.10.252.4
  Warning  FailedCreatePodContainer  25s (x70 over 15m)  kubelet, 10.10.252.4  unable to ensure pod container exists: failed to create container for [kubepods besteffort podc6eeec88-8664-11e9-9524-5254007057ba] : mkdir /sys/fs/cgroup/memory/kubepods/besteffort/podc6eeec88-8664-11e9-9524-5254007057ba: no space left on device
```

dockerd 日志报错示例:

```sh
Dec 24 11:54:31 VM_16_11_centos dockerd[11419]: time="2018-12-24T11:54:31.195900301+08:00" level=error msg="Handler for POST /v1.31/containers/b98d4aea818bf9d1d1aa84079e1688cd9b4218e008c58a8ef6d6c3c106403e7b/start returned error: OCI runtime create failed: container_linux.go:348: starting container process caused \"process_linux.go:279: applying cgroup configuration for process caused \\\"mkdir /sys/fs/cgroup/memory/kubepods/burstable/pod79fe803c-072f-11e9-90ca-525400090c71/b98d4aea818bf9d1d1aa84079e1688cd9b4218e008c58a8ef6d6c3c106403e7b: no space left on device\\\"\": unknown"
```

kubelet 日志报错示例:

```sh
Sep 09 18:09:09 VM-0-39-ubuntu kubelet[18902]: I0909 18:09:09.449722   18902 remote_runtime.go:92] RunPodSandbox from runtime service failed: rpc error: code = Unknown desc = failed to start sandbox container for pod "osp-xxx-com-ljqm19-54bf7678b8-bvz9s": Error response from daemon: oci runtime error: container_linux.go:247: starting container process caused "process_linux.go:258: applying cgroup configuration for process caused \"mkdir /sys/fs/cgroup/memory/kubepods/burstable/podf1bd9e87-1ef2-11e8-afd3-fa163ecf2dce/8710c146b3c8b52f5da62e222273703b1e3d54a6a6270a0ea7ce1b194f1b5053: no space left on device\""
```

新版的内核限制为 `2^31` (可以看成几乎不限制，[点我跳转到代码](https://github.com/torvalds/linux/blob/3120b9a6a3f7487f96af7bd634ec49c87ef712ab/kernel/cgroup/cgroup.c#L5233)): `cgroup_idr_alloc()` 传入 end 为 0 到 `idr_alloc()`， 再传给 `idr_alloc_u32()`, end 的值最终被三元运算符 `end>0 ? end-1 : INT_MAX` 转成了 `INT_MAX` 常量，即 `2^31`。所以如果新版内核有泄露问题会更难定位，表现形式会是内存消耗严重，幸运的是新版内核已经修复，推荐 4.3 以上。



#### 规避方案

如果你用的低版本内核(比如 CentOS 7 v3.10 的内核)并且不方便升级内核，可以通过不开启 kmem accounting 来实现规避，但会比较麻烦。

kubelet 和 runc 都会给 memory cgroup 开启 kmem accounting，所以要规避这个问题，就要保证kubelet 和 runc 都别开启 kmem accounting，下面分别进行说明:



##### runc

runc 在合并 [这个PR](https://github.com/opencontainers/runc/pull/1350/files) (2017-02-27) 之后创建的容器都默认开启了 kmem accounting，后来社区也注意到这个问题，并做了比较灵活的修复， [PR 1921](https://github.com/opencontainers/runc/pull/1921) 给 runc 增加了 “nokmem” 编译选项，缺省的 release 版本没有使用这个选项， 自己使用 nokmem 选项编译 runc 的方法:

```sh
cd $GO_PATH/src/github.com/opencontainers/runc/
make BUILDTAGS="seccomp nokmem"
```

docker-ce v18.09.1 之后的 runc 默认关闭了 kmem accounting，所以也可以直接升级 docker 到这个版本之后。



##### kubelet

如果是 1.14 版本及其以上，可以在编译的时候通过 build tag 来关闭 kmem accounting:

```sh
KUBE_GIT_VERSION=v1.14.1 ./build/run.sh make kubelet GOFLAGS="-tags=nokmem"
```

如果是低版本需要修改代码重新编译。kubelet 在创建 pod 对应的 cgroup 目录时，也会调用 libcontianer 中的代码对 cgroup 做设置，在 `pkg/kubelet/cm/cgroup_manager_linux.go` 的 `Create` 方法中，会调用 `Manager.Apply` 方法，最终调用 `vendor/github.com/opencontainers/runc/libcontainer/cgroups/fs/memory.go` 中的 `MemoryGroup.Apply` 方法，开启 kmem accounting。这里也需要进行处理，可以将这部分代码注释掉然后重新编译 kubelet。



### 参考资料

- 一行 kubernetes 1.9 代码引发的血案（与 CentOS 7.x 内核兼容性问题）: http://dockone.io/article/4797
- Cgroup泄漏—潜藏在你的集群中: https://tencentcloudcontainerteam.github.io/2018/12/29/cgroup-leaking/





## tcp_tw_recycle 引发丢包

`tcp_tw_recycle` 这个内核参数用来快速回收 `TIME_WAIT` 连接，不过如果在 NAT 环境下会引发问题。

RFC1323 中有如下一段描述：

```
An additional mechanism could be added to the TCP, a per-host cache of the last timestamp received from any connection. This value could then be used in the PAWS mechanism to reject old duplicate segments from earlier incarnations of the connection, if the timestamp clock can be guaranteed to have ticked at least once since the old connection was open. This would require that the TIME-WAIT delay plus the RTT together must be at least one tick of the sender’s timestamp clock. Such an extension is not part of the proposal of this RFC.
```

- 大概意思是说TCP有一种行为，可以缓存每个连接最新的时间戳，后续请求中如果时间戳小于缓存的时间戳，即视为无效，相应的数据包会被丢弃。

- Linux是否启用这种行为取决于tcp_timestamps和tcp_tw_recycle，因为tcp_timestamps缺省就是开启的，所以当tcp_tw_recycle被开启后，实际上这种行为就被激活了，当客户端或服务端以NAT方式构建的时候就可能出现问题，下面以客户端NAT为例来说明：

- 当多个客户端通过NAT方式联网并与服务端交互时，服务端看到的是同一个IP，也就是说对服务端而言这些客户端实际上等同于一个，可惜由于这些客户端的时间戳可能存在差异，于是乎从服务端的视角看，便可能出现时间戳错乱的现象，进而直接导致时间戳小的数据包被丢弃。如果发生了此类问题，具体的表现通常是是客户端明明发送的SYN，但服务端就是不响应ACK。

- 在4.12之后的内核已移除tcp_tw_recycle内核参数:

   https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=4396e46187ca5070219b81773c4e65088dac50cc https://github.com/torvalds/linux/commit/4396e46187ca5070219b81773c4e65088dac50cc





## 使用 oom-guard 在用户态处理 cgroup OOM

### 背景

由于 linux 内核对 cgroup OOM 的处理，存在很多 bug，经常有由于频繁 cgroup OOM 导致节点故障(卡死， 重启， 进程异常但无法杀死)，于是 TKE 团队开发了 `oom-guard`，在用户态处理 cgroup OOM 规避了内核 bug。



### 原理

核心思想是在发生内核 cgroup OOM kill 之前，在用户空间杀掉超限的容器， 减少走到内核 cgroup 内存回收失败后的代码分支从而触发各种内核故障的机会。



#### threshold notify

参考文档: https://lwn.net/Articles/529927/

`oom-guard` 会给 memory cgroup 设置 threshold notify， 接受内核的通知。

以一个例子来说明阀值计算通知原理: 一个 pod 设置的 memory limit 是 1000M， `oom-guard` 会根据配置参数计算出 margin:

```c
margin = 1000M * margin_ratio = 20M // 缺省margin_ratio是0.02
```

margin 最小不小于 mim_margin(缺省1M)， 最大不大于 max_margin(缺省为30M)。如果超出范围，则取 mim_margin 或 max_margin。计算 threshold = limit - margin ，也就是 1000M - 20M = 980M，把 980M 作为阈值设置给内核。当这个 pod 的内存使用量达到 980M 时， `oom-guard` 会收到内核的通知。

在触发阈值之前，`oom-gurad` 会先通过 `memory.force_empty` 触发相关 cgroup 的内存回收。 另外，如果触发阈值时，相关 cgroup 的 memory.stat 显示还有较多 cache， 则不会触发后续处理策略，这样当 cgroup 内存达到 limit 时，会内核会触发内存回收。 这个策略也会造成部分容器内存增长太快时，还是会触发内核 cgroup OOM



#### 达到阈值后的处理策略

通过 `--policy` 参数来控制处理策略。目前有三个策略， 缺省策略是 process。

- `process`: 采用跟内核cgroup OOM killer相同的策略，在该cgroup内部，选择一个 oom_score 得分最高的进程杀掉。 通过 oom-guard 发送 SIGKILL 来杀掉进程
- `container`: 在该cgroup下选择一个 docker 容器，杀掉整个容器
- `noop`: 只记录日志，并不采取任何措施



#### 事件上报

通过 webhook reporter 上报 k8s event，便于分析统计，使用`kubectl get event` 可以看到:

```sh
LAST SEEN   FIRST SEEN   COUNT     NAME                            KIND      SUBOBJECT                  TYPE      REASON                   SOURCE                    MESSAGE
14s         14s          1         172.21.16.23.158b732d352bcc31   Node                                 Warning   OomGuardKillContainer    oom-guard, 172.21.16.23   {"hostname":"172.21.16.23","timestamp":"2019-03-13T07:12:14.561650646Z","oomcgroup":"/sys/fs/cgroup/memory/kubepods/burstable/pod3d6329e5-455f-11e9-a7e5-06925242d7ea/223d4795cc3b33e28e702f72e0497e1153c4a809de6b4363f27acc12a6781cdb","proccgroup":"/sys/fs/cgroup/memory/kubepods/burstable/pod3d6329e5-455f-11e9-a7e5-06925242d7ea/223d4795cc3b33e28e702f72e0497e1153c4a809de6b4363f27acc12a6781cdb","threshold":205520896,"usage":206483456,"killed":"16481(fakeOOM) ","stats":"cache 20480|rss 205938688|rss_huge 199229440|mapped_file 0|dirty 0|writeback 0|pgpgin 1842|pgpgout 104|pgfault 2059|pgmajfault 0|inactive_anon 8192|active_anon 203816960|inactive_file 0|active_file 0|unevictable 0|hierarchical_memory_limit 209715200|total_cache 20480|total_rss 205938688|total_rss_huge 199229440|total_mapped_file 0|total_dirty 0|total_writeback 0|total_pgpgin 1842|total_pgpgout 104|total_pgfault 2059|total_pgmajfault 0|total_inactive_anon 8192|total_active_anon 203816960|total_inactive_file 0|total_active_file 0|total_unevictable 0|","policy":"Container"}
```



### 使用方法

保存部署 yaml: `oom-guard.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: oomguard
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:oomguard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: oomguard
    namespace: kube-system
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: oom-guard
  namespace: kube-system
  labels:
    app: oom-guard
spec:
  selector:
    matchLabels:
      app: oom-guard
  template:
    metadata:
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ""
      labels:
        app: oom-guard
    spec:
      serviceAccountName: oomguard
      hostPID: true
      hostNetwork: true
      dnsPolicy: ClusterFirst
      containers:
      - name: k8s-event-writer
        image: ccr.ccs.tencentyun.com/paas/k8s-event-writer:v1.6
        resources:
          limits:
            cpu: 10m
            memory: 60Mi
          requests:
            cpu: 10m
            memory: 30Mi
        args:
        - --logtostderr
        - --unix-socket=true
        env:
          - name: NODE_NAME
            valueFrom:
              fieldRef:
                fieldPath: status.hostIP
        volumeMounts:
        - name: unix
          mountPath: /unix
      - name: oomguard
        image: ccr.ccs.tencentyun.com/paas/oomguard:nosoft-v2
        imagePullPolicy: Always
        securityContext:
          privileged: true
        resources:
          limits:
            cpu: 10m
            memory: 60Mi
          requests:
            cpu: 10m
            memory: 30Mi
        volumeMounts:
        - name: cgroupdir
          mountPath: /sys/fs/cgroup/memory
        - name: unix
          mountPath: /unix
        - name: kmsg
          mountPath: /dev/kmsg
          readOnly: true
        command: ["/oom-guard"]
        args: 
        - --v=2
        - --logtostderr
        - --root=/sys/fs/cgroup/memory
        - --walkIntervalSeconds=277
        - --inotifyResetSeconds=701
        - --port=0
        - --margin-ratio=0.02
        - --min-margin=1
        - --max-margin=30
        - --guard-ms=50
        - --policy=container
        - --openSoftLimit=false
        - --webhook-url=http://localhost/message
        env:
          - name: NODE_NAME
            valueFrom:
              fieldRef:
                fieldPath: status.hostIP
      volumes:
      - name: cgroupdir
        hostPath:
          path: /sys/fs/cgroup/memory
      - name: unix
        emptyDir: {}
      - name: kmsg
        hostPath:
          path: /dev/kmsg
```

一键部署:

```sh
kubectl apply -f oom-guard.yaml
```

检查是否部署成功：

```sh
$ kubectl -n kube-system get ds oom-guard
NAME     DESIRED   CURRENT   READY     UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
oom-guard   2         2         2         2            2        <none>          6m
```

其中 **AVAILABLE** 数量跟节点数一致，说明所有节点都已经成功运行了 `oom-guard`。



#### 查看 oom-guard 日志

```sh
kubectl -n kube-system logs oom-guard-xxxxx oomguard
```



#### 查看 oom 相关事件

```sh
$ kubectl get events |grep CgroupOOM
$ kubectl get events |grep SystemOOM
$ kubectl get events |grep OomGuardKillContainer
$ kubectl get events |grep OomGuardKillProcess
```



#### 卸载

```sh
$ kubectl delete -f oom-guard.yaml
```

这个操作可能有点慢，如果一直不返回 (有节点 NotReady 时可能会卡住)，`ctrl+C` 终止，然后执行下面的脚本:

```sh
for pod in `kubectl get pod -n kube-system | grep oom-guard | awk '{print $1}'`
do
 kubectl delete pod $pod -n kube-system --grace-period=0 --force
done
```

检查删除操作是否成功

```sh
$ kubectl -n kube-system get ds oom-guard
```

提示 `...not found` 就说明删除成功了



### 关于开源

当前 `oom-gaurd` 暂未开源，正在做大量生产试验，后面大量反馈效果统计比较好的时候会考虑开源出来。



## no space left on device

- 有时候节点 NotReady， kubelet 日志报 `no space left on device`
- 有时候创建 Pod 失败，`describe pod` 看 event 报 `no space left on device`

出现这种错误有很多中可能原因，下面我们来根据现象找对应原因。



### inotify watch 耗尽

节点 NotReady，kubelet 启动失败，看 kubelet 日志:

```sh
Jul 18 15:20:58 VM_16_16_centos kubelet[11519]: E0718 15:20:58.280275   11519 raw.go:140] Failed to watch directory "/sys/fs/cgroup/memory/kubepods": inotify_add_watch /sys/fs/cgroup/memory/kubepods/burstable/pod926b7ff4-7bff-11e8-945b-52540048533c/6e85761a30707b43ed874e0140f58839618285fc90717153b3cbe7f91629ef5a: no space left on device
```

系统调用 `inotify_add_watch` 失败，提示 `no space left on device`， 这是因为系统上进程 watch 文件目录的总数超出了最大限制，可以修改内核参数调高限制，详细请参考本书内核相关章节的 [inotify watch 耗尽](https://www.bookstack.cn/read/kubernetes-practice-guide/$troubleshooting-linux-faq-runnig-out-of-inotify-watches.html)



### cgroup 泄露

查看当前 cgroup 数量:

```sh
$ cat /proc/cgroups | column -t
#subsys_name  hierarchy  num_cgroups  enabled
cpuset        5          29           1
cpu           7          126          1
cpuacct       7          126          1
memory        9          127          1
devices       4          126          1
freezer       2          29           1
net_cls       6          29           1
blkio         10         126          1
perf_event    3          29           1
hugetlb       11         29           1
pids          8          126          1
net_prio      6          29           1
```

cgroup 子系统目录下面所有每个目录及其子目录都认为是一个独立的 cgroup，所以也可以在文件系统中统计目录数来获取实际 cgroup 数量，通常跟 `/proc/cgroups` 里面看到的应该一致:

```sh
$ find -L /sys/fs/cgroup/memory -type d | wc -l
127
```

当 cgroup 泄露发生时，这里的数量就不是真实的了，低版本内核限制最大 65535 个 cgroup，并且开启 kmem 删除 cgroup 时会泄露，大量创建删除容器后泄露了许多 cgroup，最终总数达到 65535，新建容器创建 cgroup 将会失败，报 `no space left on device`

详细请参考本书内核相关章节的 [cgroup 泄露](https://www.bookstack.cn/read/kubernetes-practice-guide/$troubleshooting-kernel-cgroup-leaking.html)



### 磁盘被写满

- pod 启动失败 (状态 `CreateContainerError`)

```sh
csi-cephfsplugin-27znb                        0/2     CreateContainerError   167        17h
Warning  Failed   5m1s (x3397 over 17h)  kubelet, ip-10-0-151-35.us-west-2.compute.internal  (combined from similar events): Error: container create failed: container_linux.go:336: starting container process caused "process_linux.go:399: container init caused \"rootfs_linux.go:58: mounting \\\"/sys\\\" to rootfs \\\"/var/lib/containers/storage/overlay/051e985771cc69f3f699895a1dada9ef6483e912b46a99e004af7bb4852183eb/merged\\\" at \\\"/var/lib/containers/storage/overlay/051e985771cc69f3f699895a1dada9ef6483e912b46a99e004af7bb4852183eb/merged/sys\\\" caused \\\"no space left on device\\\"\""
```







# 案例分享



## 驱逐导致服务中断



### 案例

TKE 一客户的某个节点有问题，无法挂载nfs，通过新加节点，驱逐故障节点的 pod 来规避，但导致了业务 10min 服务不可用，排查发现其它节点 pod 很多集体出现了重启，主要是连不上 kube-dns 无法解析 service，业务调用不成功，从而对外表现为服务不可用。

为什么会中断？驱逐的原理是先封锁节点，然后将旧的 node 上的 pod 删除，replicaset 控制器检测到 pod 减少，会重新创建一个 pod，调度到新的 node上，这个过程是先删除，再创建，并非是滚动更新，因此更新过程中，如果一个deployment的所有 pod 都在被驱逐的节点上，则可能导致该服务不可用。

那为什么会影响其它 pod？分析kubelet日志，kube-dns 有两个副本，都在这个被驱逐的节点上，所以驱逐的时候 kube-dns 不通，影响了其它 pod 解析 service，导致服务集体不可用。

那为什么会中断这么久？通常在新的节点应该很快才是，通过进一步分析新节点的 kubelet 日志，发现 kube-dns 从拉镜像到容器启动之间花了很长时间，检查节点上的镜像发现有很多大镜像(1~2GB)，猜测是拉取镜像有并发限制，kube-dns 的镜像虽小，但在排队等大镜像下载完，检查 kubelet 启动参数，确实有 `--registry-burst` 这个参数控制镜像下载并发数限制。但最终发现其实应该是 `--serialize-image-pulls` 这个参数导致的，kubelet 启动参数没有指定该参数，而该参数默认值为 true，即默认串行下载镜像，不并发下载，所以导致镜像下载排队，是的 kube-dns 延迟了很长时间才启动。



### 解决方案

- 避免服务单点故障，多副本，并加反亲和性
- 设置 preStop hook 与 readinessProbe，更新路由规则





## DNS 5 秒延时



### 延时现象

客户反馈从 pod 中访问服务时，总是有些请求的响应时延会达到5秒。正常的响应只需要毫秒级别的时延。



### 抓包

- 通过 `nsenter` 进入 pod netns，使用节点上的 tcpdump 抓 pod 中的包 (抓包方法参考[这里](https://www.bookstack.cn/read/kubernetes-practice-guide/$troubleshooting-cases-troubleshooting-trick-capture-packets-in-container.md))，发现是有的 DNS 请求没有收到响应，超时 5 秒后，再次发送 DNS 请求才成功收到响应。
- 在 kube-dns pod 抓包，发现是有 DNS 请求没有到达 kube-dns pod，在中途被丢弃了。

为什么是 5 秒？ `man resolv.conf` 可以看到 glibc 的 resolver 的缺省超时时间是 5s:

```c
timeout:n
       Sets the amount of time the resolver will wait for a response from a remote name server before retrying the query via a different name server.  Measured in seconds, the default is RES_TIMEOUT (currently  5,  see
       <resolv.h>).  The value for this option is silently capped to 30.
```



### 丢包原因

经过搜索发现这是一个普遍问题。

根本原因是内核 conntrack 模块的 bug，netfilter 做 NAT 时可能发生资源竞争导致部分报文丢弃。

Weave works的工程师 [Martynas Pumputis](https://www.bookstack.cn/read/kubernetes-practice-guide/$troubleshooting-cases-martynas@weave.works) 对这个问题做了很详细的分析：[Racy conntrack and DNS lookup timeouts](https://www.weave.works/blog/racy-conntrack-and-dns-lookup-timeouts)

相关结论：

- 只有多个线程或进程，并发从同一个 socket 发送相同五元组的 UDP 报文时，才有一定概率会发生
- glibc, musl(alpine linux的libc库)都使用 “parallel query”, 就是并发发出多个查询请求，因此很容易碰到这样的冲突，造成查询请求被丢弃
- 由于 ipvs 也使用了 conntrack, 使用 kube-proxy 的 ipvs 模式，并不能避免这个问题



### 问题的根本解决

Martynas 向内核提交了两个 patch 来 fix 这个问题，不过他说如果集群中有多个DNS server的情况下，问题并没有完全解决。

其中一个 patch 已经在 2018-7-18 被合并到 linux 内核主线中: [netfilter: nf_conntrack: resolve clash for matching conntracks](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ed07d9a021df6da53456663a76999189badc432a)

目前只有4.19.rc 版本包含这个patch。



### 规避办法

#### 规避方案一：使用TCP发送DNS请求

由于TCP没有这个问题，有人提出可以在容器的resolv.conf中增加`options use-vc`, 强制glibc使用TCP协议发送DNS query。下面是这个man resolv.conf中关于这个选项的说明：

```c
use-vc (since glibc 2.14)
                     Sets RES_USEVC in _res.options.  This option forces the
                     use of TCP for DNS resolutions.
```

笔者使用镜像”busybox:1.29.3-glibc” (libc 2.24) 做了试验，并没有见到这样的效果，容器仍然是通过UDP发送DNS请求。



#### 规避方案二：避免相同五元组DNS请求的并发

resolv.conf还有另外两个相关的参数：

- single-request-reopen (since glibc 2.9)
- single-request (since glibc 2.10)

man resolv.conf中解释如下：

```c
single-request-reopen (since glibc 2.9)
                     Sets RES_SNGLKUPREOP in _res.options.  The resolver
                     uses the same socket for the A and AAAA requests.  Some
                     hardware mistakenly sends back only one reply.  When
                     that happens the client system will sit and wait for
                     the second reply.  Turning this option on changes this
                     behavior so that if two requests from the same port are
                     not handled correctly it will close the socket and open
                     a new one before sending the second request.
single-request (since glibc 2.10)
                     Sets RES_SNGLKUP in _res.options.  By default, glibc
                     performs IPv4 and IPv6 lookups in parallel since
                     version 2.9.  Some appliance DNS servers cannot handle
                     these queries properly and make the requests time out.
                     This option disables the behavior and makes glibc
                     perform the IPv6 and IPv4 requests sequentially (at the
                     cost of some slowdown of the resolving process).
```

用自己的话解释下：

- `single-request-reopen`: 发送 A 类型请求和 AAAA 类型请求使用不同的源端口，这样两个请求在 conntrack 表中不占用同一个表项，从而避免冲突
- `single-request`: 避免并发，改为串行发送 A 类型和 AAAA 类型请求，没有了并发，从而也避免了冲突

要给容器的 `resolv.conf` 加上 options 参数，有几个办法：

**1) 在容器的 “ENTRYPOINT” 或者 “CMD” 脚本中，执行 /bin/echo ‘options single-request-reopen’ >> /etc/resolv.conf**

**2) 在 pod 的 postStart hook 中:**

```yaml
        lifecycle:
          postStart:
            exec:
              command:
              - /bin/sh
              - -c 
              - "/bin/echo 'options single-request-reopen' >> /etc/resolv.conf"
```

**3) 使用 template.spec.dnsConfig (k8s v1.9 及以上才支持):**

```yaml
  template:
    spec:
      dnsConfig:
        options:
          - name: single-request-reopen
```

**4) 使用 ConfigMap 覆盖 pod 里面的 /etc/resolv.conf:**

configmap:

```yaml
apiVersion: v1
data:
  resolv.conf: |
    nameserver 1.2.3.4
    search default.svc.cluster.local svc.cluster.local cluster.local ec2.internal
    options ndots:5 single-request-reopen timeout:1
kind: ConfigMap
metadata:
  name: resolvconf
```

pod spec:

```yaml
        volumeMounts:
        - name: resolv-conf
          mountPath: /etc/resolv.conf
          subPath: resolv.conf
...
      volumes:
      - name: resolv-conf
        configMap:
          name: resolvconf
          items:
          - key: resolv.conf
            path: resolv.conf
```

**5) 使用 MutatingAdmissionWebhook**

[MutatingAdmissionWebhook](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#mutatingadmissionwebhook-beta-in-1-9) 是 1.9 引入的 Controller，用于对一个指定的 Resource 的操作之前，对这个 resource 进行变更。 istio 的自动 sidecar注入就是用这个功能来实现的。 我们也可以通过 MutatingAdmissionWebhook，来自动给所有POD，注入以上3)或者4)所需要的相关内容。

以上方法中， 1)和2)都需要修改镜像， 3)和4)则只需要修改POD的spec， 能适用于所有镜像。不过还是有不方便的地方：

- 每个工作负载的yaml都要做修改，比较麻烦
- 对于通过helm创建的工作负载，需要修改helm charts

方法5)对集群使用者最省事，照常提交工作负载即可。不过初期需要一定的开发工作量。



#### 规避方案三：使用本地DNS缓存

容器的DNS请求都发往本地的DNS缓存服务(dnsmasq, nscd等)，不需要走DNAT，也不会发生conntrack冲突。另外还有个好处，就是避免DNS服务成为性能瓶颈。

使用本地DNS缓存有两种方式：

- 每个容器自带一个DNS缓存服务
- 每个节点运行一个DNS缓存服务，所有容器都把本节点的DNS缓存作为自己的 nameserver

从资源效率的角度来考虑的话，推荐后一种方式。官方也意识到了这个问题比较常见，给出了 coredns 以 cache 模式作为 daemonset 部署的解决方案: https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/



#### 实施办法

条条大路通罗马，不管怎么做，最终到达上面描述的效果即可。

POD中要访问节点上的DNS缓存服务，可以使用节点的IP。 如果节点上的容器都连在一个虚拟bridge上， 也可以使用这个bridge的三层接口的IP(在TKE中，这个三层接口叫cbr0)。 要确保DNS缓存服务监听这个地址。

**如何把POD的/etc/resolv.conf中的nameserver设置为节点IP呢？**

一个办法，是设置 POD.spec.dnsPolicy 为 “Default”， 意思是POD里面的 /etc/resolv.conf， 使用节点上的文件。缺省使用节点上的 /etc/resolv.conf(如果kubelet通过参数—resolv-conf指定了其他文件，则使用—resolv-conf所指定的文件)。

另一个办法，是给每个节点的kubelet指定不同的—cluster-dns参数，设置为节点的IP，POD.spec.dnsPolicy仍然使用缺省值”ClusterFirst”。 kops项目甚至有个issue在讨论如何在部署集群时设置好—cluster-dns指向节点IP: https://github.com/kubernetes/kops/issues/5584



### 参考资料

- Racy conntrack and DNS lookup timeouts: https://www.weave.works/blog/racy-conntrack-and-dns-lookup-timeouts
- 5 – 15s DNS lookups on Kubernetes? : https://blog.quentin-machu.fr/2018/06/24/5-15s-dns-lookups-on-kubernetes/
- DNS intermittent delays of 5s: https://github.com/kubernetes/kubernetes/issues/56903
- 记一次Docker/Kubernetes上无法解释的连接超时原因探寻之旅: https://mp.weixin.qq.com/s/VYBs8iqf0HsNg9WAxktzYQ





## ARP 缓存爆满导致健康检查失败



### 案例

TKE 一用户某集群节点数 1200+，用户监控方案是 daemonset 部署 node-exporter 暴露节点监控指标，使用 hostNework 方式，statefulset 部署 promethues 且仅有一个实例，落在了一个节点上，promethues 请求所有节点 node-exporter 获取节点监控指标，也就是或扫描所有节点，导致 arp cache 需要存所有 node 的记录，而节点数 1200+，大于了 `net.ipv4.neigh.default.gc_thresh3` 的默认值 1024，这个值是个硬限制，arp cache记录数大于这个就会强制触发 gc，所以会造成频繁gc，当有数据包发送会查本地 arp，如果本地没找到 arp 记录就会判断当前 arp cache 记录数+1是否大于 gc_thresh3，如果没有就会广播 arp 查询 mac 地址，如果大于了就直接报 `arp_cache: neighbor table overflow!`，并且放弃 arp 请求，无法获取 mac 地址也就无法知道探测报文该往哪儿发(即便就在本机某个 veth pair)，kubelet 对本机 pod 做存活检查发 arp 查 mac 地址，在 arp cahce 找不到，由于这时 arp cache已经满了，刚要 gc 但还没做所以就只有报错丢包，导致存活检查失败重启 pod



### 解决方案

调整部分节点内核参数，将 arp cache 的 gc 阀值调高 (`/etc/sysctl.conf`):

```sh
net.ipv4.neigh.default.gc_thresh1 = 80000
net.ipv4.neigh.default.gc_thresh2 = 90000
net.ipv4.neigh.default.gc_thresh3 = 100000
```

并给 node 打下label，修改 pod spec，加下 nodeSelector 或者 nodeAffnity，让 pod 只调度到这部分改过内核参数的节点，更多请参考本书 [处理实践: arp_cache 溢出](https://www.bookstack.cn/read/kubernetes-practice-guide/$troubleshooting-cases-troubleshooting-handle-arp_cache-overflow.md)





## 跨 VPC 访问 NodePort 经常超时



现象: 从 VPC a 访问 VPC b 的 TKE 集群的某个节点的 NodePort，有时候正常，有时候会卡住直到超时。

原因怎么查？

当然是先抓包看看啦，抓 server 端 NodePort 的包，发现异常时 server 能收到 SYN，但没响应 ACK:

![跨 VPC 访问 NodePort 经常超时 - 图1](D:\学习资料\笔记\k8s\k8s图\f37ae87b48a5b13ce58f6e7f12ddb1df.png)

反复执行 `netstat -s | grep LISTEN` 发现 SYN 被丢弃数量不断增加:

![跨 VPC 访问 NodePort 经常超时 - 图2](D:\学习资料\笔记\k8s\k8s图\bad0777df555f871dce1dd386f7a2d48.png)

分析：

- 两个VPC之间使用对等连接打通的，CVM 之间通信应该就跟在一个内网一样可以互通。
- 为什么同一 VPC 下访问没问题，跨 VPC 有问题? 两者访问的区别是什么?

再仔细看下 client 所在环境，发现 client 是 VPC a 的 TKE 集群节点，捋一下:

- client 在 VPC a 的 TKE 集群的节点
- server 在 VPC b 的 TKE 集群的节点

因为 TKE 集群中有个叫 `ip-masq-agent` 的 daemonset，它会给 node 写 iptables 规则，默认 SNAT 目的 IP 是 VPC 之外的报文，所以 client 访问 server 会做 SNAT，也就是这里跨 VPC 相比同 VPC 访问 NodePort 多了一次 SNAT，如果是因为多了一次 SNAT 导致的这个问题，直觉告诉我这个应该跟内核参数有关，因为是 server 收到包没回包，所以应该是 server 所在 node 的内核参数问题，对比这个 node 和 普通 TKE node 的默认内核参数，发现这个 node `net.ipv4.tcp_tw_recycle = 1`，这个参数默认是关闭的，跟用户沟通后发现这个内核参数确实在做压测的时候调整过。

解释一下，TCP 主动关闭连接的一方在发送最后一个 ACK 会进入 `TIME_AWAIT` 状态，再等待 2 个 MSL 时间后才会关闭(因为如果 server 没收到 client 第四次挥手确认报文，server 会重发第三次挥手 FIN 报文，所以 client 需要停留 2 MSL的时长来处理可能会重复收到的报文段；同时等待 2 MSL 也可以让由于网络不通畅产生的滞留报文失效，避免新建立的连接收到之前旧连接的报文)，了解更详细的过程请参考 TCP 四次挥手。

参数 `tcp_tw_recycle` 用于快速回收 `TIME_AWAIT` 连接，通常在增加连接并发能力的场景会开启，比如发起大量短连接，快速回收可避免 `tw_buckets` 资源耗尽导致无法建立新连接 (`time wait bucket table overflow`)

查得 `tcp_tw_recycle` 有个坑，在 RFC1323 有段描述:

```
An additional mechanism could be added to the TCP, a per-host cache of the last timestamp received from any connection. This value could then be used in the PAWS mechanism to reject old duplicate segments from earlier incarnations of the connection, if the timestamp clock can be guaranteed to have ticked at least once since the old connection was open. This would require that the TIME-WAIT delay plus the RTT together must be at least one tick of the sender’s timestamp clock. Such an extension is not part of the proposal of this RFC.
```

大概意思是说 TCP 有一种行为，可以缓存每个连接最新的时间戳，后续请求中如果时间戳小于缓存的时间戳，即视为无效，相应的数据包会被丢弃。

Linux 是否启用这种行为取决于 `tcp_timestamps` 和 `tcp_tw_recycle`，因为 `tcp_timestamps` 缺省开启，所以当 `tcp_tw_recycle` 被开启后，实际上这种行为就被激活了，当客户端或服务端以 `NAT` 方式构建的时候就可能出现问题。

当多个客户端通过 NAT 方式联网并与服务端交互时，服务端看到的是同一个 IP，也就是说对服务端而言这些客户端实际上等同于一个，可惜由于这些客户端的时间戳可能存在差异，于是乎从服务端的视角看，便可能出现时间戳错乱的现象，进而直接导致时间戳小的数据包被丢弃。如果发生了此类问题，具体的表现通常是是客户端明明发送的 SYN，但服务端就是不响应 ACK。

回到我们的问题上，client 所在节点上可能也会有其它 pod 访问到 server 所在节点，而它们都被 SNAT 成了 client 所在节点的 NODE IP，但时间戳存在差异，server 就会看到时间戳错乱，因为开启了 `tcp_tw_recycle` 和 `tcp_timestamps` 激活了上述行为，就丢掉了比缓存时间戳小的报文，导致部分 SYN 被丢弃，这也解释了为什么之前我们抓包发现异常时 server 收到了 SYN，但没有响应 ACK，进而说明为什么 client 的请求部分会卡住直到超时。

由于 `tcp_tw_recycle` 坑太多，在内核 4.12 之后已移除: [remove tcp_tw_recycle](https://github.com/torvalds/linux/commit/4396e46187ca5070219b81773c4e65088dac50cc)





## 访问 externalTrafficPolicy 为 Local 的 Service 对应 LB 有时超时



现象：用户在 TKE 创建了公网 LoadBalancer 类型的 Service，externalTrafficPolicy 设为了 Local，访问这个 Service 对应的公网 LB 有时会超时。

externalTrafficPolicy 为 Local 的 Service 用于在四层获取客户端真实源 IP，官方参考文档：[Source IP for Services with Type=LoadBalancer](https://kubernetes.io/docs/tutorials/services/source-ip/#source-ip-for-services-with-type-loadbalancer)

TKE 的 LoadBalancer 类型 Service 实现是使用 CLB 绑定所有节点对应 Service 的 NodePort，CLB 不做 SNAT，报文转发到 NodePort 时源 IP 还是真实的客户端 IP，如果 NodePort 对应 Service 的 externalTrafficPolicy 不是 Local 的就会做 SNAT，到 pod 时就看不到客户端真实源 IP 了，但如果是 Local 的话就不做 SNAT，如果本机 node 有这个 Service 的 endpoint 就转到对应 pod，如果没有就直接丢掉，因为如果转到其它 node 上的 pod 就必须要做 SNAT，不然无法回包，而 SNAT 之后就无法获取真实源 IP 了。

LB 会对绑定节点的 NodePort 做健康检查探测，检查 LB 的健康检查状态: 发现这个 NodePort 的所有节点都不健康 !!!

那么问题来了:

1. 为什么会全不健康，这个 Service 有对应的 pod 实例，有些节点上是有 endpoint 的，为什么它们也不健康?
2. LB 健康检查全不健康，但是为什么有时还是可以访问后端服务?

跟 LB 的同学确认: 如果后端 rs 全不健康会激活 LB 的全死全活逻辑，也就是所有后端 rs 都可以转发。

那么有 endpoint 的 node 也是不健康这个怎么解释?

在有 endpoint 的 node 上抓 NodePort 的包: 发现很多来自 LB 的 SYN，但是没有响应 ACK。

看起来报文在哪被丢了，继续抓下 cbr0 看下: 发现没有来自 LB 的包，说明报文在 cbr0 之前被丢了。

再观察用户集群环境信息:

1. k8s 版本1.12
2. 启用了 ipvs
3. 只有 local 的 service 才有异常

尝试新建一个 1.12 启用 ipvs 和一个没启用 ipvs 的测试集群。也都创建 Local 的 LoadBalancer Service，发现启用 ipvs 的测试集群复现了那个问题，没启用 ipvs 的集群没这个问题。

再尝试创建 1.10 的集群，也启用 ipvs，发现没这个问题。

看起来跟集群版本和是否启用 ipvs 有关。

1.12 对比 1.10 启用 ipvs 的集群: 1.12 的会将 LB 的 `EXTERNAL-IP` 绑到 `kube-ipvs0` 上，而 1.10 的不会:

```sh
$ ip a show kube-ipvs0 | grep -A2 170.106.134.124
    inet 170.106.134.124/32 brd 170.106.134.124 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
```

- 170.106.134.124 是 LB 的公网 IP
- 1.12 启用 ipvs 的集群将 LB 的公网 IP 绑到了 `kube-ipvs0` 网卡上

`kube-ipvs0` 是一个 dummy interface，实际不会接收报文，可以看到它的网卡状态是 DOWN，主要用于绑 ipvs 规则的 VIP，因为 ipvs 主要工作在 netfilter 的 INPUT 链，报文通过 PREROUTING 链之后需要决定下一步该进入 INPUT 还是 FORWARD 链，如果是本机 IP 就会进入 INPUT，如果不是就会进入 FORWARD 转发到其它机器。所以 k8s 利用 `kube-ipvs0` 这个网卡将 service 相关的 VIP 绑在上面以便让报文进入 INPUT 进而被 ipvs 转发。

当 IP 被绑到 `kube-ipvs0` 上，内核会自动将上面的 IP 写入 local 路由:

```sh
$ ip route show table local | grep 170.106.134.124
local 170.106.134.124 dev kube-ipvs0  proto kernel  scope host  src 170.106.134.124
```

内核认为在 local 路由里的 IP 是本机 IP，而 linux 默认有个行为: 忽略任何来自非回环网卡并且源 IP 是本机 IP 的报文。而 LB 的探测报文源 IP 就是 LB IP，也就是 Service 的 `EXTERNAL-IP` 猜想就是因为这个 IP 被绑到 `kube-ipvs0`，自动加进 local 路由导致内核直接忽略了 LB 的探测报文。

带着猜想做实现， 试一下将 LB IP 从 local 路由中删除:

```sh
$ ip route del table local local 170.106.134.124 dev kube-ipvs0  proto kernel  scope host  src 170.106.134.124
```

发现这个 node 的在 LB 的健康检查的状态变成健康了! 看来就是因为这个 LB IP 被绑到 `kube-ipvs0` 导致内核忽略了来自 LB 的探测报文，然后 LB 收不到回包认为不健康。

那为什么其它厂商没反馈这个问题？应该是 LB 的实现问题，腾讯云的公网 CLB 的健康探测报文源 IP 就是 LB 的公网 IP，而大多数厂商的 LB 探测报文源 IP 是保留 IP 并非 LB 自身的 VIP。

如何解决呢? 发现一个内核参数: [accept_local](https://github.com/torvalds/linux/commit/8153a10c08f1312af563bb92532002e46d3f504a) 可以让 linux 接收源 IP 是本机 IP 的报文。

试了开启这个参数，确实在 cbr0 收到来自 LB 的探测报文了，说明报文能被 pod 收到，但抓 eth0 还是没有给 LB 回包。

为什么没有回包? 分析下五元组，要给 LB 回包，那么 `目的IP:目的Port` 必须是探测报文的 `源IP:源Port`，所以目的 IP 就是 LB IP，由于容器不在主 netns，发包经过 veth pair 到 cbr0 之后需要再经过 netfilter 处理，报文进入 PREROUTING 链然后发现目的 IP 是本机 IP，进入 INPUT 链，所以报文就出不去了。再分析下进入 INPUT 后会怎样，因为目的 Port 跟 LB 探测报文源 Port 相同，是一个随机端口，不在 Service 的端口列表，所以没有对应的 IPVS 规则，IPVS 也就不会转发它，而 `kube-ipvs0` 上虽然绑了这个 IP，但它是一个 dummy interface，不会收包，所以报文最后又被忽略了。

再看看为什么 1.12 启用 ipvs 会绑 `EXTERNAL-IP` 到 `kube-ipvs0`，翻翻 k8s 的 kube-proxy 支持 ipvs 的 [proposal](https://github.com/kubernetes/enhancements/blob/baca87088480254b26d0fdeb26303d7c51a20fbd/keps/sig-network/0011-ipvs-proxier.md#support-loadbalancer-service)，发现有个地方说法有点漏洞:

![访问 externalTrafficPolicy 为 Local 的 Service 对应 LB 有时超时 - 图1](D:\学习资料\笔记\k8s\k8s图\a8304eeaff8f322d145e9da8ace45b78.png)

LB 类型 Service 的 status 里有 ingress IP，实际就是 `kubectl get service` 看到的 `EXTERNAL-IP`，这里说不会绑定这个 IP 到 kube-ipvs0，但后面又说会给它创建 ipvs 规则，既然没有绑到 `kube-ipvs0`，那么这个 IP 的报文根本不会进入 INPUT 被 ipvs 模块转发，创建的 ipvs 规则也是没用的。

后来找到作者私聊，思考了下，发现设计上确实有这个问题。

看了下 1.10 确实也是这么实现的，但是为什么 1.12 又绑了这个 IP 呢? 调研后发现是因为 [#59976](https://github.com/kubernetes/kubernetes/issues/59976) 这个 issue 发现一个问题，后来引入 [#63066](https://github.com/kubernetes/kubernetes/pull/63066) 这个 PR 修复的，而这个 PR 的行为就是让 LB IP 绑到 `kube-ipvs0`，这个提交影响 1.11 及其之后的版本。

[#59976](https://github.com/kubernetes/kubernetes/issues/59976) 的问题是因为没绑 LB IP到 `kube-ipvs0` 上，在自建集群使用 `MetalLB` 来实现 LoadBalancer 类型的 Service，而有些网络环境下，pod 是无法直接访问 LB 的，导致 pod 访问 LB IP 时访问不了，而如果将 LB IP 绑到 `kube-ipvs0` 上就可以通过 ipvs 转发到 LB 类型 Service 对应的 pod 去， 而不需要真正经过 LB，所以引入了 [#63066](https://github.com/kubernetes/kubernetes/pull/63066) 这个PR。

临时方案: 将 [#63066](https://github.com/kubernetes/kubernetes/pull/63066) 这个 PR 的更改回滚下，重新编译 kube-proxy，提供升级脚本升级存量 kube-proxy。

如果是让 LB 健康检查探测支持用保留 IP 而不是自身的公网 IP ，也是可以解决，但需要跨团队合作，而且如果多个厂商都遇到这个问题，每家都需要为解决这个问题而做开发调整，代价较高，所以长期方案需要跟社区沟通一起推进，所以我提了 issue，将问题描述的很清楚: [#79783](https://github.com/kubernetes/kubernetes/issues/79783)

小思考: 为什么 CLB 可以不做 SNAT ? 回包目的 IP 就是真实客户端 IP，但客户端是直接跟 LB IP 建立的连接，如果回包不经过 LB 是不可能发送成功的呀。

是因为 CLB 的实现是在母机上通过隧道跟 CVM 互联的，多了一层封装，回包始终会经过 LB。

就是因为 CLB 不做 SNAT，正常来自客户端的报文是可以发送到 nodeport，但健康检查探测报文由于源 IP 是 LB IP 被绑到 `kube-ipvs0` 导致被忽略，也就解释了为什么健康检查失败，但通过LB能访问后端服务，只是有时会超时。那么如果要做 SNAT 的 LB 岂不是更糟糕，所有报文都变成 LB IP，所有报文都会被忽略?

我提的 issue 有回复指出，AWS 的 LB 会做 SNAT，但它们不将 LB 的 IP 写到 Service 的 Status 里，只写了 hostname，所以也不会绑 LB IP 到 `kube-ipvs0`:

![访问 externalTrafficPolicy 为 Local 的 Service 对应 LB 有时超时 - 图2](D:\学习资料\笔记\k8s\k8s图\8165040b9ee8ed4f8aab94dcd7edd08c.png)

但是只写 hostname 也得 LB 支持自动绑域名解析，并且个人觉得只写 hostname 很别扭，通过 `kubectl get svc` 或者其它 k8s 管理系统无法直接获取 LB IP，这不是一个好的解决方法。

我提了 [#79976](https://github.com/kubernetes/kubernetes/pull/79976) 这个 PR 可以解决问题: 给 kube-proxy 加 `--exclude-external-ip` 这个 flag 控制是否为 LB IP创建 ipvs 规则和绑定 `kube-ipvs0`。

但有人担心增加 kube-proxy flag 会增加 kube-proxy 的调试复杂度，看能否在 iptables 层面解决:

![访问 externalTrafficPolicy 为 Local 的 Service 对应 LB 有时超时 - 图3](D:\学习资料\笔记\k8s\k8s图\e472530cf8bf8f768cb7161911cb19ec.png)

仔细一想，确实可行，打算有空实现下，重新提个 PR:

![访问 externalTrafficPolicy 为 Local 的 Service 对应 LB 有时超时 - 图4](D:\学习资料\笔记\k8s\k8s图\664ee65ca70238d0090324cc7916e5a3.png)





## Pod 偶尔存活检查失败



现象: Pod 偶尔会存活检查失败，导致 Pod 重启，业务偶尔连接异常。

之前从未遇到这种情况，在自己测试环境尝试复现也没有成功，只有在用户这个环境才可以复现。这个用户环境流量较大，感觉跟连接数或并发量有关。

用户反馈说在友商的环境里没这个问题。

对比友商的内核参数发现有些区别，尝试将节点内核参数改成跟友商的一样，发现问题没有复现了。

再对比分析下内核参数差异，最后发现是 backlog 太小导致的，节点的 `net.ipv4.tcp_max_syn_backlog` 默认是 1024，如果短时间内并发新建 TCP 连接太多，SYN 队列就可能溢出，导致部分新连接无法建立。

解释一下:

![Pod 偶尔存活检查失败 - 图1](D:\学习资料\笔记\k8s\k8s图\1e0beee0c572c27a9289333365d12fbf.png)



TCP 连接建立会经过三次握手，server 收到 SYN 后会将连接加入 SYN 队列，当收到最后一个 ACK 后连接建立，这时会将连接从 SYN 队列中移动到 ACCEPT 队列。在 SYN 队列中的连接都是没有建立完全的连接，处于半连接状态。如果 SYN 队列比较小，而短时间内并发新建的连接比较多，同时处于半连接状态的连接就多，SYN 队列就可能溢出，`tcp_max_syn_backlog` 可以控制 SYN 队列大小，用户节点的 backlog 大小默认是 1024，改成 8096 后就可以解决问题。





## DNS 解析异常

现象: 有个用户反馈域名解析有时有问题，看报错是解析超时。

第一反应当然是看 coredns 的 log:

```sh
[ERROR] 2 loginspub.xxxxmobile-inc.net.
A: unreachable backend: read udp 172.16.0.230:43742->10.225.30.181:53: i/o timeout
```

这是上游 DNS 解析异常了，因为解析外部域名 coredns 默认会请求上游 DNS 来查询，这里的上游 DNS 默认是 coredns pod 所在宿主机的 `resolv.conf` 里面的 nameserver (coredns pod 的 dnsPolicy 为 “Default”，也就是会将宿主机里的 `resolv.conf` 里的 nameserver 加到容器里的 `resolv.conf`, coredns 默认配置 `proxy . /etc/resolv.conf`, 意思是非 service 域名会使用 coredns 容器中 `resolv.conf` 文件里的 nameserver 来解析)

确认了下，超时的上游 DNS 10.225.30.181 并不是期望的 nameserver，VPC 默认 DNS 应该是 180 开头的。看了 coredns 所在节点的 `resolv.conf`，发现确实多出了这个非期望的 nameserver，跟用户确认了下，这个 DNS 不是用户自己加上去的，添加节点时这个 nameserver 本身就在 `resolv.conf` 中。

根据内部同学反馈， 10.225.30.181 是广州一台年久失修将被撤裁的 DNS，物理网络，没有 VIP，撤掉就没有了，所以如果 coredns 用到了这台 DNS 解析时就可能 timeout。后面我们自己测试，某些 VPC 的集群确实会有这个 nameserver，奇了怪了，哪里冒出来的？

又试了下直接创建 CVM，不加进 TKE 节点发现没有这个 nameserver，只要一加进 TKE 节点就有了 !!!

看起来是 TKE 的问题，将 CVM 添加到 TKE 集群会自动重装系统，初始化并加进集群成为 K8S 的 node，确认了初始化过程并不会写 `resolv.conf`，会不会是 TKE 的 OS 镜像问题？尝试搜一下除了 `/etc/resolv.conf` 之外哪里还有这个 nameserver 的 IP，最后发现 `/etc/resolvconf/resolv.conf.d/base` 这里面有。

看下 `/etc/resolvconf/resolv.conf.d/base` 的作用：Ubuntu 的 `/etc/resolv.conf` 是动态生成的，每次重启都会将 `/etc/resolvconf/resolv.conf.d/base` 里面的内容加到 `/etc/resolv.conf` 里。

经确认: 这个文件确实是 TKE 的 Ubuntu OS 镜像里自带的，可能发布 OS 镜像时不小心加进去的。

那为什么有些 VPC 的集群的节点 `/etc/resolv.conf` 里面没那个 IP 呢？它们的 OS 镜像里也都有那个文件那个 IP 呀。

请教其它部门同学发现:

- 非 dhcp 子机，cvm 的 cloud-init 会覆盖 `/etc/resolv.conf` 来设置 dns
- dhcp 子机，cloud-init 不会设置，而是通过 dhcp 动态下发
- 2018 年 4 月 之后创建的 VPC 就都是 dhcp 类型了的，比较新的 VPC 都是 dhcp 类型的

真相大白：`/etc/resolv.conf` 一开始内容都包含 `/etc/resolvconf/resolv.conf.d/base` 的内容，也就是都有那个不期望的 nameserver，但老的 VPC 由于不是 dhcp 类型，所以 cloud-init 会覆盖 `/etc/resolv.conf`，抹掉了不被期望的 nameserver，而新创建的 VPC 都是 dhcp 类型，cloud-init 不会覆盖 `/etc/resolv.conf`，导致不被期望的 nameserver 残留在了 `/etc/resolv.conf`，而 coredns pod 的 dnsPolicy 为 “Default”，也就是会将宿主机的 `/etc/resolv.conf` 中的 nameserver 加到容器里，coredns 解析集群外的域名默认使用这些 nameserver 来解析，当用到那个将被撤裁的 nameserver 就可能 timeout。

临时解决: 删掉 `/etc/resolvconf/resolv.conf.d/base` 重启

长期解决: 我们重新制作 TKE Ubuntu OS 镜像然后发布更新

这下应该没问题了吧，But, 用户反馈还是会偶尔解析有问题，但现象不一样了，这次并不是 dns timeout。

用脚本跑测试仔细分析现象:

- 请求 `loginspub.xxxxmobile-inc.net` 时，偶尔提示域名无法解析
- 请求 `accounts.google.com` 时，偶尔提示连接失败

进入 dns 解析偶尔异常的容器的 netns 抓包:

- dns 请求会并发请求 A 和 AAAA 记录
- 测试脚本发请求打印序号，抓包然后 wireshark 分析对比异常时请求序号偏移量，找到异常时的 dns 请求报文，发现异常时 A 和 AAAA 记录的请求 id 冲突，并且 AAAA 响应先返回

![DNS 解析异常 - 图1](https://static.sitestack.cn/projects/kubernetes-practice-guide/48f1cfecc409b9dc0aa1d081aa6bab44.png)

正常情况下id不会冲突，这里冲突了也就能解释这个 dns 解析异常的现象了:

- `loginspub.xxxxmobile-inc.net` 没有 AAAA (ipv6) 记录，它的响应先返回告知 client 不存在此记录，由于请求 id 跟 A 记录请求冲突，后面 A 记录响应返回了 client 发现 id 重复就忽略了，然后认为这个域名无法解析
- `accounts.google.com` 有 AAAA 记录，响应先返回了，client 就拿这个记录去尝试请求，但当前容器环境不支持 ipv6，所以会连接失败

那为什么 dns 请求 id 会冲突?

继续观察发现: 其它节点上的 pod 不会复现这个问题，有问题这个节点上也不是所有 pod 都有这个问题，只有基于 alpine 镜像的容器才有这个问题，在此节点新起一个测试的 `alpine:latest` 的容器也一样有这个问题。

为什么 alpine 镜像的容器在这个节点上有问题在其它节点上没问题？ 为什么其他镜像的容器都没问题？它们跟 alpine 的区别是什么？

发现一点区别: alpine 使用的底层 c 库是 musl libc，其它镜像基本都是 glibc

翻 musl libc 源码, 构造 dns 请求时，请求 id 的生成没加锁，而且跟当前时间戳有关 (`network/res_mkquery.c`):

```c
/* Make a reasonably unpredictable id */
clock_gettime(CLOCK_REALTIME, &ts);
id = ts.tv_nsec + ts.tv_nsec/65536UL & 0xffff;
```

看注释，作者应该认为这样id基本不会冲突，事实证明，绝大多数情况确实不会冲突，我在网上搜了很久没有搜到任何关于 musl libc 的 dns 请求 id 冲突的情况。这个看起来取决于硬件，可能在某种类型硬件的机器上运行，短时间内生成的 id 就可能冲突。我尝试跟用户在相同地域的集群，添加相同配置相同机型的节点，也复现了这个问题，但后来删除再添加时又不能复现了，看起来后面新建的 cvm 又跑在了另一种硬件的母机上了。

OK，能解释通了，再底层的细节就不清楚了，我们来看下解决方案:

- 换基础镜像 (不用alpine)
- 完全静态编译业务程序(不依赖底层c库)，比如go语言程序编译时可以关闭 cgo (CGO_ENABLED=0)，并告诉链接器要静态链接 (`go build` 后面加 `-ldflags '-d'`)，但这需要语言和编译工具支持才可以

最终建议用户基础镜像换成另一个比较小的镜像: `debian:stretch-slim`。





## Pod 访问另一个集群的 apiserver 有延时



现象：集群 a 的 Pod 内通过 kubectl 访问集群 b 的内网地址，偶尔出现延时的情况，但直接在宿主机上用同样的方法却没有这个问题。

提炼环境和现象精髓:

1. 在 pod 内将另一个集群 apiserver 的 ip 写到了 hosts，因为 TKE apiserver 开启内网集群外内网访问创建的内网 LB 暂时没有支持自动绑内网 DNS 域名解析，所以集群外的内网访问 apiserver 需要加 hosts
2. pod 内执行 kubectl 访问另一个集群偶尔延迟 5s，有时甚至10s

观察到 5s 延时，感觉跟之前 conntrack 的丢包导致 [DNS 解析 5S 延时](https://www.bookstack.cn/read/kubernetes-practice-guide/$troubleshooting-cases-troubleshooting-cases-dns-lookup-5s-delay.md) 有关，但是加了 hosts 呀，怎么还去解析域名？

进入 pod netns 抓包: 执行 kubectl 时确实有 dns 解析，并且发生延时的时候 dns 请求没有响应然后做了重试。

看起来延时应该就是之前已知 conntrack 丢包导致 dns 5s 超时重试导致的。但是为什么会去解析域名? 明明配了 hosts 啊，正常情况应该是优先查找 hosts，没找到才去请求 dns 呀，有什么配置可以控制查找顺序?

搜了一下发现: `/etc/nsswitch.conf` 可以控制，但看有问题的 pod 里没有这个文件。然后观察到有问题的 pod 用的 alpine 镜像，试试其它镜像后发现只有基于 alpine 的镜像才会有这个问题。

再一搜发现: musl libc 并不会使用 `/etc/nsswitch.conf` ，也就是说 alpine 镜像并没有实现用这个文件控制域名查找优先顺序，瞥了一眼 musl libc 的 `gethostbyname` 和 `getaddrinfo` 的实现，看起来也没有读这个文件来控制查找顺序，写死了先查 hosts，没找到再查 dns。

这么说，那还是该先查 hosts 再查 dns 呀，为什么这里抓包看到是先查的 dns? (如果是先查 hosts 就能命中查询，不会再发起dns请求)

访问 apiserver 的 client 是 kubectl，用 go 写的，会不会是 go 程序解析域名时压根没调底层 c 库的 `gethostbyname` 或 `getaddrinfo`?

搜一下发现果然是这样: go runtime 用 go 实现了 glibc 的 `getaddrinfo` 的行为来解析域名，减少了 c 库调用 (应该是考虑到减少 cgo 调用带来的的性能损耗)

issue: [net: replicate DNS resolution behaviour of getaddrinfo(glibc) in the go dns resolver](https://github.com/golang/go/issues/18518)

翻源码验证下:

Unix 系的 OS 下，除了 openbsd， go runtime 会读取 `/etc/nsswitch.conf` (`net/conf.go`):

```c
if runtime.GOOS != "openbsd" {
    confVal.nss = parseNSSConfFile("/etc/nsswitch.conf")
}
```

`hostLookupOrder` 函数决定域名解析顺序的策略，Linux 下，如果没有 `nsswitch.conf` 文件就 dns 比 hosts 文件优先 (`net/conf.go`):

```c
// hostLookupOrder determines which strategy to use to resolve hostname.
// The provided Resolver is optional. nil means to not consider its options.
func (c *conf) hostLookupOrder(r *Resolver, hostname string) (ret hostLookupOrder) {
    ......
    // If /etc/nsswitch.conf doesn't exist or doesn't specify any
    // sources for "hosts", assume Go's DNS will work fine.
    if os.IsNotExist(nss.err) || (nss.err == nil && len(srcs) == 0) {
        ......
        if c.goos == "linux" {
            // glibc says the default is "dns [!UNAVAIL=return] files"
            // https://www.gnu.org/software/libc/manual/html_node/Notes-on-NSS-Configuration-File.html.
            return hostLookupDNSFiles
        }
        return hostLookupFilesDNS
    }
```

可以看到 `hostLookupDNSFiles` 的意思是 dns first (`net/dnsclient_unix.go`):

```c
// hostLookupOrder specifies the order of LookupHost lookup strategies.
// It is basically a simplified representation of nsswitch.conf.
// "files" means /etc/hosts.
type hostLookupOrder int
const (
    // hostLookupCgo means defer to cgo.
    hostLookupCgo      hostLookupOrder = iota
    hostLookupFilesDNS                 // files first
    hostLookupDNSFiles                 // dns first
    hostLookupFiles                    // only files
    hostLookupDNS                      // only DNS
)
var lookupOrderName = map[hostLookupOrder]string{
    hostLookupCgo:      "cgo",
    hostLookupFilesDNS: "files,dns",
    hostLookupDNSFiles: "dns,files",
    hostLookupFiles:    "files",
    hostLookupDNS:      "dns",
}
```

所以虽然 alpine 用的 musl libc 不是 glibc，但 go 程序解析域名还是一样走的 glibc 的逻辑，而 alpine 没有 `/etc/nsswitch.conf` 文件，也就解释了为什么 kubectl 访问 apiserver 先做 dns 解析，没解析到再查的 hosts，导致每次访问都去请求 dns，恰好又碰到 conntrack 那个丢包问题导致 dns 5s 延时，在用户这里表现就是 pod 内用 kubectl 访问 apiserver 偶尔出现 5s 延时，有时出现 10s 是因为重试的那次 dns 请求刚好也遇到 conntrack 丢包导致延时又叠加了 5s 。

解决方案:

1. 换基础镜像，不用 alpine
2. 挂载 `nsswitch.conf` 文件 (可以用 hostPath)





## LB 压测 NodePort CPS 低



现象: LoadBalancer 类型的 Service，直接压测 NodePort CPS 比较高，但如果压测 LB CPS 就很低。

环境说明: 用户使用的黑石TKE，不是公有云TKE，黑石的机器是物理机，LB的实现也跟公有云不一样，但 LoadBalancer 类型的 Service 的实现同样也是 LB 绑定各节点的 NodePort，报文发到 LB 后转到节点的 NodePort， 然后再路由到对应 pod，而测试在公有云 TKE 环境下没有这个问题。

- client 抓包: 大量SYN重传。
- server 抓包: 抓 NodePort 的包，发现当 client SYN 重传时 server 能收到 SYN 包但没有响应。

又是 SYN 收到但没响应，难道又是开启 `tcp_tw_recycle` 导致的？检查节点的内核参数发现并没有开启，除了这个原因，还会有什么情况能导致被丢弃？

`conntrack -S` 看到 `insert_failed` 数量在不断增加，也就是 conntrack 在插入很多新连接的时候失败了，为什么会插入失败？什么情况下会插入失败？

挖内核源码: netfilter conntrack 模块为每个连接创建 conntrack 表项时，表项的创建和最终插入之间还有一段逻辑，没有加锁，是一种乐观锁的过程。conntrack 表项并发刚创建时五元组不冲突的话可以创建成功，但中间经过 NAT 转换之后五元组就可能变成相同，第一个可以插入成功，后面的就会插入失败，因为已经有相同的表项存在。比如一个 SYN 已经做了 NAT 但是还没到最终插入的时候，另一个 SYN 也在做 NAT，因为之前那个 SYN 还没插入，这个 SYN 做 NAT 的时候就认为这个五元组没有被占用，那么它 NAT 之后的五元组就可能跟那个还没插入的包相同。

在我们这个问题里实际就是 netfilter 做 SNAT 时源端口选举冲突了，黑石 LB 会做 SNAT，SNAT 时使用了 16 个不同 IP 做源，但是短时间内源 Port 却是集中一致的，并发两个 SYN a 和SYN b，被 LB SNAT 后源 IP 不同但源 Port 很可能相同，这里就假设两个报文被 LB SNAT 之后它们源 IP 不同源 Port 相同，报文同时到了节点的 NodePort 会再次做 SNAT 再转发到对应的 Pod，当报文到了 NodePort 时，这时它们五元组不冲突，netfilter 为它们分别创建了 conntrack 表项，SYN a 被节点 SNAT 时默认行为是 从 port_range 范围的当前源 Port 作为起始位置开始循环遍历，选举出没有被占用的作为源 Port，因为这两个 SYN 源 Port 相同，所以它们源 Port 选举的起始位置相同，当 SYN a 选出源 Port 但还没将 conntrack 表项插入时，netfilter 认为这个 Port 没被占用就很可能给 SYN b 也选了相同的源 Port，这时他们五元组就相同了，当 SYN a 的 conntrack 表项插入后再插入 SYN b 的 conntrack 表项时，发现已经有相同的记录就将 SYN b 的 conntrack 表项丢弃了。

解决方法探索: 不使用源端口选举，在 iptables 的 MASQUERADE 规则如果加 `--random-fully` 这个 flag 可以让端口选举完全随机，基本上能避免绝大多数的冲突，但也无法完全杜绝。最终决定开发 LB 直接绑 Pod IP，不基于 NodePort，从而避免 netfilter 的 SNAT 源端口冲突问题。





## kubectl edit 或者 apply 报 SchemaError



### 问题现象

kubectl edit 或 apply 资源报如下错误:

```
error: SchemaError(io.k8s.apimachinery.pkg.apis.meta.v1.APIGroup): invalid object doesn't have additional properties
```

集群版本：v1.10



### 排查过程

1. 使用 `kubectl apply -f tmp.yaml --dry-run -v8` 发现请求 `/openapi/v2` 这个 api 之后，kubectl在 validate 过程报错。
2. 换成 kubectl 1.12 之后没有再报错。
3. `kubectl get --raw '/openapi/v2'` 发现返回的 json 内容与正常集群有差异，刚开始返回的 json title 为 `Kubernetes metrics-server`，正常的是 Kubernetes。
4. 怀疑是 `metrics-server` 的问题，发现集群内确实安装了 k8s 官方的 `metrics-server`，询问得知之前是 0.3.1，后面升级为了 0.3.5。
5. 将 metrics-server 回滚之后恢复正常。



### 原因分析

初步怀疑，新版本的 metrics-server 使用了新的 openapi-generator，生成的 openapi 格式和之前 k8s 版本生成的有差异。导致旧版本的 kubectl 在解析 openapi 的 schema 时发生异常，查看代码发现1.10 和 1.12 版本在解析 openapi 的 schema 时，实现确实有差异。







# 排错技巧

 

## 分析 ExitCode 定位 Pod 异常退出原因



使用 `kubectl describe pod <pod name>` 查看异常 pod 的状态:

```sh
Containers:
  kubedns:
    Container ID:  docker://5fb8adf9ee62afc6d3f6f3d9590041818750b392dff015d7091eaaf99cf1c945
    Image:         ccr.ccs.tencentyun.com/library/kubedns-amd64:1.14.4
    Image ID:      docker-pullable://ccr.ccs.tencentyun.com/library/kubedns-amd64@sha256:40790881bbe9ef4ae4ff7fe8b892498eecb7fe6dcc22661402f271e03f7de344
    Ports:         10053/UDP, 10053/TCP, 10055/TCP
    Host Ports:    0/UDP, 0/TCP, 0/TCP
    Args:
      --domain=cluster.local.
      --dns-port=10053
      --config-dir=/kube-dns-config
      --v=2
    State:          Running
      Started:      Tue, 27 Aug 2019 10:58:49 +0800
    Last State:     Terminated
      Reason:       Error
      Exit Code:    255
      Started:      Tue, 27 Aug 2019 10:40:42 +0800
      Finished:     Tue, 27 Aug 2019 10:58:27 +0800
    Ready:          True
    Restart Count:  1
```

在容器列表里看 `Last State` 字段，其中 `ExitCode` 即程序上次退出时的状态码，如果不为 0，表示异常退出，我们可以分析下原因。



### 退出状态码的区间

- 必须在 0-255 之间
- 0 表示正常退出
- 外界中断将程序退出的时候状态码区间在 129-255，(操作系统给程序发送中断信号，比如 `kill -9` 是 `SIGKILL`，`ctrl+c` 是 `SIGINT`)
- 一般程序自身原因导致的异常退出状态区间在 1-128 (这只是一般约定，程序如果一定要用129-255的状态码也是可以的)

假如写代码指定的退出状态码时不在 0-255 之间，例如: `exit(-1)`，这时会自动做一个转换，最终呈现的状态码还是会在 0-255 之间。我们把状态码记为 `code`

- 当指定的退出时状态码为负数，那么转换公式如下:

```c
256 - (|code| % 256)
```

- 当指定的退出时状态码为正数，那么转换公式如下:

```c
code % 256
```



### 常见异常状态码

- 137 (被 `SIGKILL` 中断信号杀死)

  - 此状态码一般是因为 pod 中容器内存达到了它的资源限制(`resources.limits`)，一般是内存溢出(OOM)，CPU达到限制只需要不分时间片给程序就可以。因为限制资源是通过 linux 的 cgroup 实现的，所以 cgroup 会将此容器强制杀掉，类似于 `kill -9`，此时在 `describe pod` 中可以看到 Reason 是 `OOMKilled`

  - 还可能是宿主机本身资源不够用了(OOM)，内核会选取一些进程杀掉来释放内存

  - 不管是 cgroup 限制杀掉进程还是因为节点机器本身资源不够导致进程死掉，都可以从系统日志中找到记录:

    > ubuntu 的系统日志在 `/var/log/syslog`，centos 的系统日志在 `/var/log/messages`，都可以用 `journalctl -k` 来查看系统日志

  - 也可能是 livenessProbe (存活检查) 失败，kubelet 杀死的 pod

  - 还可能是被恶意木马进程杀死

- 1 和 255

  - 这种可能是一般错误，具体错误原因只能看容器日志，因为很多程序员写异常退出时习惯用 `exit(1)` 或 `exit(-1)`，-1 会根据转换规则转成 255



### 状态码参考

这里罗列了一些状态码的含义：[Appendix E. Exit Codes With Special Meanings](http://tldp.org/LDP/abs/html/exitcodes.html)



### Linux 标准中断信号

Linux 程序被外界中断时会发送中断信号，程序退出时的状态码就是中断信号值加上 128 得到的，比如 `SIGKILL` 的中断信号值为 9，那么程序退出状态码就为 9+128=137。以下是标准信号值参考：

```c
Signal     Value     Action   Comment
──────────────────────────────────────────────────────────────────────
SIGHUP        1       Term    Hangup detected on controlling terminal
                                     or death of controlling process
SIGINT        2       Term    Interrupt from keyboard
SIGQUIT       3       Core    Quit from keyboard
SIGILL        4       Core    Illegal Instruction
SIGABRT       6       Core    Abort signal from abort(3)
SIGFPE        8       Core    Floating-point exception
SIGKILL       9       Term    Kill signal
SIGSEGV      11       Core    Invalid memory reference
SIGPIPE      13       Term    Broken pipe: write to pipe with no
                                     readers; see pipe(7)
SIGALRM      14       Term    Timer signal from alarm(2)
SIGTERM      15       Term    Termination signal
SIGUSR1   30,10,16    Term    User-defined signal 1
SIGUSR2   31,12,17    Term    User-defined signal 2
SIGCHLD   20,17,18    Ign     Child stopped or terminated
SIGCONT   19,18,25    Cont    Continue if stopped
SIGSTOP   17,19,23    Stop    Stop process
SIGTSTP   18,20,24    Stop    Stop typed at terminal
SIGTTIN   21,21,26    Stop    Terminal input for background process
SIGTTOU   22,22,27    Stop    Terminal output for background process
```



### C/C++ 退出状态码

`/usr/include/sysexits.h` 试图将退出状态码标准化(仅限 C/C++):

```c
#define EX_OK           0       /* successful termination */
#define EX__BASE        64      /* base value for error messages */
#define EX_USAGE        64      /* command line usage error */
#define EX_DATAERR      65      /* data format error */
#define EX_NOINPUT      66      /* cannot open input */
#define EX_NOUSER       67      /* addressee unknown */
#define EX_NOHOST       68      /* host name unknown */
#define EX_UNAVAILABLE  69      /* service unavailable */
#define EX_SOFTWARE     70      /* internal software error */
#define EX_OSERR        71      /* system error (e.g., can't fork) */
#define EX_OSFILE       72      /* critical OS file missing */
#define EX_CANTCREAT    73      /* can't create (user) output file */
#define EX_IOERR        74      /* input/output error */
#define EX_TEMPFAIL     75      /* temp failure; user is invited to retry */
#define EX_PROTOCOL     76      /* remote error in protocol */
#define EX_NOPERM       77      /* permission denied */
#define EX_CONFIG       78      /* configuration error */
#define EX__MAX 78      /* maximum listed value */
```





## 容器内抓包定位网络问题



在使用 kubernetes 跑应用的时候，可能会遇到一些网络问题，比较常见的是服务端无响应(超时)或回包内容不正常，如果没找出各种配置上有问题，这时我们需要确认数据包到底有没有最终被路由到容器里，或者报文到达容器的内容和出容器的内容符不符合预期，通过分析报文可以进一步缩小问题范围。那么如何在容器内抓包呢？本文提供实用的脚本一键进入容器网络命名空间(netns)，使用宿主机上的tcpdump进行抓包。



### 使用脚本一键进入 pod netns 抓包

- 发现某个服务不通，最好将其副本数调为1，并找到这个副本 pod 所在节点和 pod 名称

```sh
kubectl get pod -o wide
```

- 登录 pod 所在节点，将如下脚本粘贴到 shell (注册函数到当前登录的 shell，我们后面用)

```c
function e() {
    set -eu
    ns=${2-"default"}
    pod=`kubectl -n $ns describe pod $1 | grep -A10 "^Containers:" | grep -Eo 'docker://.*$' | head -n 1 | sed 's/docker:\/\/\(.*\)$/\1/'`
    pid=`docker inspect -f {{.State.Pid}} $pod`
    echo "entering pod netns for $ns/$1"
    cmd="nsenter -n --target $pid"
    echo $cmd
    $cmd
}
```

- 一键进入 pod 所在的 netns，格式：`e POD_NAME NAMESPACE`，示例：

```c
e istio-galley-58c7c7c646-m6568 istio-system
e proxy-5546768954-9rxg6 # 省略 NAMESPACE 默认为 default
```

- 这时已经进入 pod 的 netns，可以执行宿主机上的 `ip a` 或 `ifconfig` 来查看容器的网卡，执行 `netstat -tunlp` 查看当前容器监听了哪些端口，再通过 `tcpdump` 抓包：

```sh
$ tcpdump -i eth0 -w test.pcap port 80
```

- `ctrl-c` 停止抓包，再用 `scp` 或 `sz` 将抓下来的包下载到本地使用 `wireshark` 分析，提供一些常用的 `wireshark` 过滤语法：

```sh
# 使用 telnet 连上并发送一些测试文本，比如 "lbtest"，
# 用下面语句可以看发送的测试报文有没有到容器
tcp contains "lbtest"
# 如果容器提供的是http服务，可以使用 curl 发送一些测试路径的请求，
# 通过下面语句过滤 uri 看报文有没有都容器
http.request.uri=="/mytest"
```



### 脚本原理

我们解释下步骤二中用到的脚本的原理

- 查看指定 pod 运行的容器 ID

```
kubectl describe pod <pod> -n mservice
```

- 获得容器进程的 pid

```
docker inspect -f {{.State.Pid}} <container>
```

- 进入该容器的 network namespace

```
nsenter -n --target <PID>
```

依赖宿主机的命名：`kubectl`, `docker`, `nsenter`, `grep`, `head`, `sed`































































































































































