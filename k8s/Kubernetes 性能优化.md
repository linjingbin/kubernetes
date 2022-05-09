[toc]



# Kubernetes 性能优化



## Kubernetes Controller高可用诡异的15mins超时



### 前言

节点宕机是生产环境无法规避的情况，生产环境必须适配这种情况，保障即便宕机后，服务依旧可用。而Kubernetes内部实现的Leader Election Mechanism很好的解决了controller高可用的问题。但是某一次生产环境的测试发现controller在节点宕机后并没有马上(在规定的分布式锁释放时间后)实现切换，本文对这个问题进行了详尽的描述，分析，并在最后给出了解决方案



### 问题

在生产环境中，controller架构图部署如下：

![image-20210108095448924](D:\学习资料\笔记\k8s\k8s图\image-20210108095448924.png)

可以看到controller利用反亲和特性在不同的母机上部署有两个副本，通过Kubernetes service访问AA(Aggregated APIServer)

集群采用iptables的代理模式，假设目前两个母机的Controller经过iptables DNAT访问的都是母机10.0.0.3(图中Node2)的AA；同时Controller采用Leader Election机制实现高可用，母机10.0.0.3上的Controller是Leader，而母机10.0.0.2(图中Node1)上的Controller是候选者

若此时母机10.0.0.3突然宕机，理论上母机10.0.0.2上的Controller会在分布式锁被释放后获取锁从而变成Leader，接管Controller任务，这也是Leader Election的原理

但是在手动关机母机10.0.0.3后(模拟宕机情景)，发现母机10.0.0.2上的Controller一直在报timeout错误，整个流程观察到现象如下：

Controller和AA部署详情如下：

```sh
$ kubectl get pods -o wide
controller-75f5547689-hjcmd               1/1     Running   0          47h     192.168.2.51    10.0.0.3   <none>           <none>
controller-75f5547689-m2l6g               1/1     Running   0          47h     192.168.1.108   10.0.0.2   <none>           <none>
aa-5ccf944d9f-9vnhx          1/1     Running   0          47h     192.168.2.52    10.0.0.3   <none>           <none>
aa-5ccf944d9f-zfldh          1/1     Running   0          47h     192.168.1.109   10.0.0.2   <none>           <none>

$ kubectl get svc 
NAME                     TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
aa         ClusterIP   192.168.255.220   <none>        443/TCP            51d
```

关闭母机10.0.0.3之前母机10.0.0.2上的Controller socket连接如下：

```sh
$ docker ps|grep controller-75f5547689-m2l6g
a10f0dcddddb ...
$ docker top a10f0dcddddb
UID                 PID                 PPID                C                   STIME               TTY                 TIME                CMD
root                16025               16007               0                   Aug01               ?                   00:03:18            controller xxx
$ nsenter -n -t 16025
$ netstat -tnope 
Active Internet connections (w/o servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       User       Inode      PID/Program name     Timer
tcp        0      0 192.168.1.108:45220      192.168.255.220:443     ESTABLISHED 0          6334873    16025/controller    keepalive (4.39/0/0)
```

母机10.0.0.2上的Controller日志如下：

```sh
2020-08-03 12:06:32.407 info    lock is held by controller-75f5547689-hjcmd_23b81f76-884a-498a-8549-63371441bf18 and has not yet expired
2020-08-03 12:06:32.407 info    failed to acquire lease /controller
2020-08-03 12:06:35.344 info    lock is held by controller-75f5547689-hjcmd_23b81f76-884a-498a-8549-63371441bf18 and has not yet expired
2020-08-03 12:06:35.344 info    failed to acquire lease /controller
...
```

母机10.0.0.2上的Controller(Candidate)每隔3s尝试获取分布式lock，由于已经被母机10.0.0.3上的Controller独占，所以会显示获取失败(正常逻辑)

另外，在母机10.0.0.2上抓包观察路由情况：

```sh
$ tcpdump -iany -nnvvXSs 0|grep 192.168.1.108|grep 443
...
192.168.2.52.443 > 192.168.1.108.45220: Flags [P.], cksum 0x6c09 (correct), seq 1740053674:1740053764, ack 3283065874, win 66, options [nop,nop,TS val 172345058 ecr 248885717], length 90
192.168.2.52.443 > 192.168.1.108.45220: Flags [P.], cksum 0x4f72 (correct), seq 1740053764:1740054232, ack 3283065874, win 66, options [nop,nop,TS val 172345058 ecr 248885717], length 468
192.168.2.52.443 > 192.168.1.108.45220: Flags [P.], cksum 0x6c09 (correct), seq 1740053674:1740053764, ack 3283065874, win 66, options [nop,nop,TS val 172345058 ecr 248885717], length 90
192.168.255.220.443 > 192.168.1.108.45220: Flags [P.], cksum 0x6e60 (correct), seq 1740053674:1740053764, ack 3283065874, win 66, options [nop,nop,TS val 172345058 ecr 248885717], length 90
192.168.2.52.443 > 192.168.1.108.45220: Flags [P.], cksum 0x4f72 (correct), seq 1740053764:1740054232, ack 3283065874, win 66, options [nop,nop,TS val 172345058 ecr 248885717], length 468
192.168.255.220.443 > 192.168.1.108.45220: Flags [P.], cksum 0x51c9 (correct), seq 1740053764:1740054232, ack 3283065874, win 66, options [nop,nop,TS val 172345058 ecr 248885717], length 468
192.168.1.108.45220 > 192.168.255.220.443: Flags [.], cksum 0x59a8 (incorrect -> 0xd003), seq 3283065874, ack 1740053764, win 349, options [nop,nop,TS val 248885720 ecr 172345058], length 0
192.168.1.108.45220 > 192.168.2.52.443: Flags [.], cksum 0x5bff (incorrect -> 0xcdac), seq 3283065874, ack 1740053764, win 349, options [nop,nop,TS val 248885720 ecr 172345058], length 0
192.168.1.108.45220 > 192.168.2.52.443: Flags [.], cksum 0xcdac (correct), seq 3283065874, ack 1740053764, win 349, options [nop,nop,TS val 248885720 ecr 172345058], length 0
192.168.1.108.45220 > 192.168.255.220.443: Flags [.], cksum 0x59a8 (incorrect -> 0xce2f), seq 3283065874, ack 1740054232, win 349, options [nop,nop,TS val 248885720 ecr 172345058], length 0
192.168.1.108.45220 > 192.168.2.52.443: Flags [.], cksum 0x5bff (incorrect -> 0xcbd8), seq 3283065874, ack 1740054232, win 349, options [nop,nop,TS val 248885720 ecr 172345058], length 0
192.168.1.108.45220 > 192.168.2.52.443: Flags [.], cksum 0xcbd8 (correct), seq 3283065874, ack 1740054232, win 349, options [nop,nop,TS val 248885720 ecr 172345058], length 0
192.168.1.108.45220 > 192.168.255.220.443: Flags [P.], cksum 0x59d5 (incorrect -> 0x0bcf), seq 3283065874:3283065919, ack 1740054232, win 349, options [nop,nop,TS val 248889253 ecr 172345058], length 45
192.168.1.108.45220 > 192.168.2.52.443: Flags [P.], cksum 0x5c2c (incorrect -> 0x0978), seq 3283065874:3283065919, ack 1740054232, win 349, options [nop,nop,TS val 248889253 ecr 172345058], length 45
192.168.1.108.45220 > 192.168.2.52.443: Flags [P.], cksum 0x0978 (correct), seq 3283065874:3283065919, ack 1740054232, win 349, options [nop,nop,TS val 248889253 ecr 172345058], length 45
192.168.2.52.443 > 192.168.1.108.45220: Flags [P.], cksum 0x52e7 (correct), seq 1740054232:1740054324, ack 3283065919, win 66, options [nop,nop,TS val 172348594 ecr 248889253], length 92
192.168.2.52.443 > 192.168.1.108.45220: Flags [P.], cksum 0x6dee (correct), seq 1740054324:1740054792, ack 3283065919, win 66, options [nop,nop,TS val 172348594 ecr 248889253], length 468
192.168.2.52.443 > 192.168.1.108.45220: Flags [P.], cksum 0x52e7 (correct), seq 1740054232:1740054324, ack 3283065919, win 66, options [nop,nop,TS val 172348594 ecr 248889253], length 92
192.168.255.220.443 > 192.168.1.108.45220: Flags [P.], cksum 0x553e (correct), seq 1740054232:1740054324, ack 3283065919, win 66, options [nop,nop,TS val 172348594 ecr 248889253], length 92
192.168.2.52.443 > 192.168.1.108.45220: Flags [P.], cksum 0x6dee (correct), seq 1740054324:1740054792, ack 3283065919, win 66, options [nop,nop,TS val 172348594 ecr 248889253], length 468
192.168.255.220.443 > 192.168.1.108.45220: Flags [P.], cksum 0x7045 (correct), seq 1740054324:1740054792, ack 3283065919, win 66, options [nop,nop,TS val 172348594 ecr 248889253], length 468
192.168.1.108.45220 > 192.168.255.220.443: Flags [.], cksum 0x59a8 (incorrect -> 0xb205), seq 3283065919, ack 1740054324, win 349, options [nop,nop,TS val 248889257 ecr 172348594], length 0
192.168.1.108.45220 > 192.168.2.52.443: Flags [.], cksum 0x5bff (incorrect -> 0xafae), seq 3283065919, ack 1740054324, win 349, options [nop,nop,TS val 248889257 ecr 172348594], length 0
192.168.1.108.45220 > 192.168.2.52.443: Flags [.], cksum 0xafae (correct), seq 3283065919, ack 1740054324, win 349, options [nop,nop,TS val 248889257 ecr 172348594], length 0
192.168.1.108.45220 > 192.168.255.220.443: Flags [.], cksum 0x59a8 (incorrect -> 0xb031), seq 3283065919, ack 1740054792, win 349, options [nop,nop,TS val 248889257 ecr 172348594], length 0
```

此时，关闭母机10.0.0.3电源(模拟宕机)。socket连接如下：

```sh
$ netstat -tnope 
Active Internet connections (w/o servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       User       Inode      PID/Program name     Timer
tcp        0   1263 192.168.1.108:45220      192.168.255.220:443     ESTABLISHED 0          6334873    16025/controller    on (21.57/9/0)
```

母机10.0.0.2上的Controller日志如下：

```sh
...
2020-08-03 12:17:34.137 info    lock is held by controller-75f5547689-hjcmd_23b81f76-884a-498a-8549-63371441bf18 and has not yet expired
2020-08-03 12:17:34.137 info    failed to acquire lease /controller
2020-08-03 12:17:37.674 info    lock is held by controller-75f5547689-hjcmd_23b81f76-884a-498a-8549-63371441bf18 and has not yet expired
2020-08-03 12:17:37.674 info    failed to acquire lease /controller
2020-08-03 12:17:52.040 error   error retrieving resource lock /controller: Get https://aa:443/api/v1/namespaces/default/configmaps/controller?timeout=10s: net/http: request canceled (Client.Timeout exceeded while awaiting headers)
2020-08-03 12:17:52.040 info    failed to acquire lease /controller
2020-08-03 12:18:04.937 error   error retrieving resource lock /controller: Get https://aa:443/api/v1/namespaces/default/configmaps/controller?timeout=10s: net/http: request canceled (Client.Timeout exceeded while awaiting headers)
2020-08-03 12:18:04.937 info    failed to acquire lease /controller
...
```

另外，在母机10.0.0.3宕机大约40s后，母机10.0.0.2上的iptables规则如下：

```sh
$ iptables-save |grep aa
-A KUBE-SERVICES -d 192.168.255.220/32 -p tcp -m comment --comment "default/aa: cluster IP" -m tcp --dport 443 -j KUBE-SVC-JIBCLJO3UBZXHPHV

$ iptables-save |grep KUBE-SVC-JIBCLJO3UBZXHPHV
:KUBE-SVC-JIBCLJO3UBZXHPHV - [0:0]
-A KUBE-SERVICES -d 192.168.255.220/32 -p tcp -m comment --comment "default/aa: cluster IP" -m tcp --dport 443 -j KUBE-SVC-JIBCLJO3UBZXHPHV
-A KUBE-SVC-JIBCLJO3UBZXHPHV -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-3B7I6NOW767V7PGC
-A KUBE-SVC-JIBCLJO3UBZXHPHV -j KUBE-SEP-ATGG5YPR5S4AT627

$ iptables-save |grep KUBE-SEP-3B7I6NOW767V7PGC
:KUBE-SEP-3B7I6NOW767V7PGC - [0:0]
-A KUBE-SEP-3B7I6NOW767V7PGC -s 192.168.0.119/32 -j KUBE-MARK-MASQ
-A KUBE-SEP-3B7I6NOW767V7PGC -p tcp -m tcp -j DNAT --to-destination 192.168.0.119:443
-A KUBE-SVC-JIBCLJO3UBZXHPHV -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-3B7I6NOW767V7PGC

$ iptables-save |grep KUBE-SEP-ATGG5YPR5S4AT627
:KUBE-SEP-ATGG5YPR5S4AT627 - [0:0]
-A KUBE-SEP-ATGG5YPR5S4AT627 -s 192.168.1.109/32 -j KUBE-MARK-MASQ
-A KUBE-SEP-ATGG5YPR5S4AT627 -p tcp -m tcp -j DNAT --to-destination 192.168.1.109:443
-A KUBE-SVC-JIBCLJO3UBZXHPHV -j KUBE-SEP-ATGG5YPR5S4AT627

$ kubectl get pods -o wide
aa-5ccf944d9f-9vnhx          1/1     Terminating   0          2d      192.168.2.52    10.0.0.3   <none>           <none>
aa-5ccf944d9f-zfldh          1/1     Running       0          47h     192.168.1.109   10.0.0.2   <none>           <none>
aa-5ccf944d9f-zzwr4          1/1     Running       0          3m3s    192.168.0.119   10.0.0.1   <none>           <none>
```

可以看到40s后iptables规则中已经剔除192.168.2.52(母机10.0.0.3上的AA)，但是socket一直存在，且一直访问的是192.168.2.52.443(母机10.0.0.3上的AA)，如下：

```sh
$ netstat -tnope 
Active Internet connections (w/o servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       User       Inode      PID/Program name     Timer
tcp        0   5874 192.168.1.108:45220      192.168.255.220:443     ESTABLISHED 0          6334873    16025/controller    on (37.31/15/0)

...
192.168.1.108.45220 > 192.168.255.220.443: Flags [P.], cksum 0x59d5 (incorrect -> 0x1889), seq 3283065919:3283065964, ack 1740054792, win 349, options [nop,nop,TS val 249584128 ecr 172348594], length 45
192.168.1.108.45220 > 192.168.2.52.443: Flags [P.], cksum 0x5c2c (incorrect -> 0x1632), seq 3283065919:3283065964, ack 1740054792, win 349, options [nop,nop,TS val 249584128 ecr 172348594], length 45
192.168.1.108.45220 > 192.168.2.52.443: Flags [P.], cksum 0x1632 (correct), seq 3283065919:3283065964, ack 1740054792, win 349, options [nop,nop,TS val 249584128 ecr 172348594], length 45
...
```

同时，母机10.0.0.2上的Controller一直在报timeout，如下：

```sh
...
2020-08-03 12:31:29.170 error   error retrieving resource lock /controller: Get https://aa:443/api/v1/namespaces/default/configmaps/controller?timeout=10s: net/http: request canceled (Client.Timeout exceeded while awaiting headers)
2020-08-03 12:31:29.170 info    failed to acquire lease /controller
2020-08-03 12:31:42.856 error   error retrieving resource lock /controller: Get https://aa:443/api/v1/namespaces/default/configmaps/controller?timeout=10s: context deadline exceeded (Client.Timeout exceeded while awaiting headers)
2020-08-03 12:31:42.856 info    failed to acquire lease /controller
...
```

最后，在15 mins的超时报错后，日志显示正常，如下：

```sh
2020-08-03 12:32:47.936 error   error retrieving resource lock /controller: Get https://aa:443/api/v1/namespaces/default/configmaps/controller?timeout=10s: net/http: request canceled (Client.Timeout exceeded while awaiting headers)
2020-08-03 12:32:47.936 info    failed to acquire lease /controller
2020-08-03 12:33:01.005 error   error retrieving resource lock /controller: Get https://aa:443/api/v1/namespaces/default/configmaps/controller?timeout=10s: context deadline exceeded
2020-08-03 12:33:01.005 info    failed to acquire lease /controller
2020-08-03 12:33:13.185 error   error retrieving resource lock /controller: Get https://aa:443/api/v1/namespaces/default/configmaps/controller?timeout=10s: read tcp 192.168.1.108:45220->192.168.255.220:443: read: connection timed out
2020-08-03 12:33:13.185 info    failed to acquire lease /controller
2020-08-03 12:33:17.003 info    lock is held by controller-75f5547689-br8n8_c920cdc5-4ec1-431a-8278-0d1038340241 and has not yet expired
2020-08-03 12:33:17.003 info    failed to acquire lease /controller
2020-08-03 12:33:19.396 info    lock is held by controller-75f5547689-br8n8_c920cdc5-4ec1-431a-8278-0d1038340241 and has not yet expired
2020-08-03 12:33:19.396 info    failed to acquire lease /controller
```

同时，192.168.1.108.45220 > 192.168.255.220.443 socket消失，如下：

```sh
$ netstat -tnope 
Active Internet connections (w/o servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       User       Inode      PID/Program name     Timer
tcp        0      0 192.168.1.108:44272      192.168.255.220:443     ESTABLISHED 0          17580619   16025/controller    keepalive (14.59/0/0)
```

而抓包显示此时没有192.168.1.108:45220 > 192.168.255.220.443连接的包；转换到192.168.1.108.44272 > 192.168.1.109.443，如下：

```sh
...
192.168.1.108.45220 > 192.168.255.220.443: Flags [P.], cksum 0x59d5 (incorrect -> 0x1889), seq 3283065919:3283065964, ack 1740054792, win 349, options [nop,nop,TS val 249584128 ecr 172348594], length 45
192.168.1.108.45220 > 192.168.2.52.443: Flags [P.], cksum 0x5c2c (incorrect -> 0x1632), seq 3283065919:3283065964, ack 1740054792, win 349, options [nop,nop,TS val 249584128 ecr 172348594], length 45
192.168.1.108.45220 > 192.168.2.52.443: Flags [P.], cksum 0x1632 (correct), seq 3283065919:3283065964, ack 1740054792, win 349, options [nop,nop,TS val 249584128 ecr 172348594], length 45
192.168.1.108.45220 > 192.168.255.220.443: Flags [P.], cksum 0x59d5 (incorrect -> 0x4287), seq 3283065919:3283065964, ack 1740054792, win 349, options [nop,nop,TS val 249704448 ecr 172348594], length 45
192.168.1.108.45220 > 192.168.2.52.443: Flags [P.], cksum 0x5c2c (incorrect -> 0x4030), seq 3283065919:3283065964, ack 1740054792, win 349, options [nop,nop,TS val 249704448 ecr 172348594], length 45
192.168.1.108.45220 > 192.168.2.52.443: Flags [P.], cksum 0x4030 (correct), seq 3283065919:3283065964, ack 1740054792, win 349, options [nop,nop,TS val 249704448 ecr 172348594], length 45

192.168.1.108.44272 > 192.168.255.220.443: Flags [S], cksum 0x59b0 (incorrect -> 0x42f9), seq 479406945, win 28200, options [mss 1410,sackOK,TS val 249828577 ecr 0,nop,wscale 9], length 0
192.168.1.108.44272 > 192.168.1.109.443: Flags [S], cksum 0x5b40 (incorrect -> 0x4169), seq 479406945, win 28200, options [mss 1410,sackOK,TS val 249828577 ecr 0,nop,wscale 9], length 0
192.168.1.109.443 > 192.168.1.108.44272: Flags [S.], cksum 0x5b40 (incorrect -> 0x107c), seq 847109001, ack 479406946, win 27960, options [mss 1410,sackOK,TS val 249828577 ecr 249828577,nop,wscale 9], length 0
192.168.255.220.443 > 192.168.1.108.44272: Flags [S.], cksum 0x59b0 (incorrect -> 0x120c), seq 847109001, ack 479406946, win 27960, options [mss 1410,sackOK,TS val 249828577 ecr 249828577,nop,wscale 9], length 0
192.168.1.108.44272 > 192.168.255.220.443: Flags [.], cksum 0x59a8 (incorrect -> 0xada8), seq 479406946, ack 847109002, win 56, options [nop,nop,TS val 249828577 ecr 249828577], length 0
192.168.1.108.44272 > 192.168.1.109.443: Flags [.], cksum 0x5b38 (incorrect -> 0xac18), seq 479406946, ack 847109002, win 56, options [nop,nop,TS val 249828577 ecr 249828577], length 0
192.168.1.108.44272 > 192.168.255.220.443: Flags [P.], cksum 0x5a92 (incorrect -> 0xdc04), seq 479406946:479407180, ack 847109002, win 56, options [nop,nop,TS val 249828577 ecr 249828577], length 234
192.168.1.108.44272 > 192.168.1.109.443: Flags [P.], cksum 0x5c22 (incorrect -> 0xda74), seq 479406946:479407180, ack 847109002, win 56, options [nop,nop,TS val 249828577 ecr 249828577], length 234
192.168.1.109.443 > 192.168.1.108.44272: Flags [.], cksum 0x5b38 (incorrect -> 0xab2d), seq 847109002, ack 479407180, win 57, options [nop,nop,TS val 249828577 ecr 249828577], length 0
192.168.255.220.443 > 192.168.1.108.44272: Flags [.], cksum 0x59a8 (incorrect -> 0xacbd), seq 847109002, ack 479407180, win 57, options [nop,nop,TS val 249828577 ecr 249828577], length 0
192.168.1.109.443 > 192.168.1.108.44272: Flags [P.], cksum 0x631b (incorrect -> 0xb6d9), seq 847109002:847111021, ack 479407180, win 57, options [nop,nop,TS val 249828580 ecr 249828577], length 2019
192.168.255.220.443 > 192.168.1.108.44272: Flags [P.], cksum 0x618b (incorrect -> 0xb869), seq 847109002:847111021, ack 479407180, win 57, options [nop,nop,TS val 249828580 ecr 249828577], length 2019
192.168.1.108.44272 > 192.168.255.220.443: Flags [.], cksum 0x59a8 (incorrect -> 0xa4ce), seq 479407180, ack 847111021, win 63, options [nop,nop,TS val 249828580 ecr 249828580], length 0
192.168.1.108.44272 > 192.168.1.109.443: Flags [.], cksum 0x5b38 (incorrect -> 0xa33e), seq 479407180, ack 847111021, win 63, options [nop,nop,TS val 249828580 ecr 249828580], length 0
192.168.1.108.44272 > 192.168.255.220.443: Flags [P.], cksum 0x5e41 (incorrect -> 0x7349), seq 479407180:479408357, ack 847111021, win 63, options [nop,nop,TS val 249828582 ecr 249828580], length 1177
192.168.1.108.44272 > 192.168.1.109.443: Flags [P.], cksum 0x5fd1 (incorrect -> 0x71b9), seq 479407180:479408357, ack 847111021, win 63, options [nop,nop,TS val 249828582 ecr 249828580], length 1177
192.168.1.109.443 > 192.168.1.108.44272: Flags [P.], cksum 0x5b6b (incorrect -> 0x3ee4), seq 847111021:847111072, ack 479408357, win 63, options [nop,nop,TS val 249828583 ecr 249828582], length 51
192.168.255.220.443 > 192.168.1.108.44272: Flags [P.], cksum 0x59db (incorrect -> 0x4074), seq 847111021:847111072, ack 479408357, win 63, options [nop,nop,TS val 249828583 ecr 249828582], length 51
192.168.1.109.443 > 192.168.1.108.44272: Flags [P.], cksum 0x5b76 (incorrect -> 0x3896), seq 847111072:847111134, ack 479408357, win 63, options [nop,nop,TS val 249828583 ecr 249828582], length 62
...
```



### 分析 - 禁用HTTP/2

从上面观察到的现象，我们开始分析问题出来哪里。一般分析问题的流程是自上而下，这里也不例外，我们从应用层开始进行分析

这里Controller每隔3s尝试获取一次分布式锁，超时时间设置为10s，如下为精简后的关键代码段(k8s.io/client-go/rest/config.go)：

```sh
...
// RESTClientFor returns a RESTClient that satisfies the requested attributes on a client Config
// object. Note that a RESTClient may require fields that are optional when initializing a Client.
// A RESTClient created by this method is generic - it expects to operate on an API that follows
// the Kubernetes conventions, but may not be the Kubernetes API.
func RESTClientFor(config *Config) (*RESTClient, error) {
  if config.GroupVersion == nil {
    return nil, fmt.Errorf("GroupVersion is required when initializing a RESTClient")
  }
  if config.NegotiatedSerializer == nil {
    return nil, fmt.Errorf("NegotiatedSerializer is required when initializing a RESTClient")
  }
  qps := config.QPS
  if config.QPS == 0.0 {
    qps = DefaultQPS
  }
  burst := config.Burst
  if config.Burst == 0 {
    burst = DefaultBurst
  }

  baseURL, versionedAPIPath, err := defaultServerUrlFor(config)
  if err != nil {
    return nil, err
  }

  transport, err := TransportFor(config)
  if err != nil {
    return nil, err
  }

  var httpClient *http.Client
  if transport != http.DefaultTransport {
    httpClient = &http.Client{Transport: transport}
    if config.Timeout > 0 {
      httpClient.Timeout = config.Timeout
    }
  }

  return NewRESTClient(baseURL, versionedAPIPath, config.ContentConfig, qps, burst, config.RateLimiter, httpClient)
}

// TransportFor returns an http.RoundTripper that will provide the authentication
// or transport level security defined by the provided Config. Will return the
// default http.DefaultTransport if no special case behavior is needed.
func TransportFor(config *Config) (http.RoundTripper, error) {
  cfg, err := config.TransportConfig()
  if err != nil {
    return nil, err
  }
  return transport.New(cfg)
}

// New returns an http.RoundTripper that will provide the authentication
// or transport level security defined by the provided Config.
func New(config *Config) (http.RoundTripper, error) {
  // Set transport level security
  if config.Transport != nil && (config.HasCA() || config.HasCertAuth() || config.HasCertCallback() || config.TLS.Insecure) {
    return nil, fmt.Errorf("using a custom transport with TLS certificate options or the insecure flag is not allowed")
  }

  var (
    rt  http.RoundTripper
    err error
  )

  if config.Transport != nil {
    rt = config.Transport
  } else {
    rt, err = tlsCache.get(config)
    if err != nil {
      return nil, err
    }
  }

  return HTTPWrappersForConfig(config, rt)
}

...
func (c *tlsTransportCache) get(config *Config) (http.RoundTripper, error) {
  key, err := tlsConfigKey(config)
  if err != nil {
    return nil, err
  }

  // Ensure we only create a single transport for the given TLS options
  c.mu.Lock()
  defer c.mu.Unlock()

  // See if we already have a custom transport for this config
  if t, ok := c.transports[key]; ok {
    return t, nil
  }

  // Get the TLS options for this client config
  tlsConfig, err := TLSConfigFor(config)
  if err != nil {
    return nil, err
  }
  // The options didn't require a custom TLS config
  if tlsConfig == nil && config.Dial == nil {
    return http.DefaultTransport, nil
  }

  dial := config.Dial
  if dial == nil {
    dial = (&net.Dialer{
      Timeout:   30 * time.Second,
      KeepAlive: 30 * time.Second,
    }).DialContext
  }
  // Cache a single transport for these options
  c.transports[key] = utilnet.SetTransportDefaults(&http.Transport{
    Proxy:               http.ProxyFromEnvironment,
    TLSHandshakeTimeout: 10 * time.Second,
    TLSClientConfig:     tlsConfig,
    MaxIdleConnsPerHost: idleConnsPerHost,
    DialContext:         dial,
  })
  return c.transports[key], nil
}

// SetTransportDefaults applies the defaults from http.DefaultTransport
// for the Proxy, Dial, and TLSHandshakeTimeout fields if unset
func SetTransportDefaults(t *http.Transport) *http.Transport {
  t = SetOldTransportDefaults(t)
  // Allow clients to disable http2 if needed.
  if s := os.Getenv("DISABLE_HTTP2"); len(s) > 0 {
    klog.Infof("HTTP2 has been explicitly disabled")
  } else {
    if err := http2.ConfigureTransport(t); err != nil {
      klog.Warningf("Transport failed http2 configuration: %v", err)
    }
  }
  return t
}
...
```

可以看出这里Controller默认通过SetTransportDefaults使用HTTP/2协议，且设置了如下http Timeout参数：

- http.Client.Timeout：10s(http.Client.Timeout涵盖了HTTP请求从连接建立，请求发送，接收回应，重定向，以及读取http.Response.Body的整个生命周期的超时时间)
- net.Dialer.Timeout：30s(底层连接建立的超时时间)
- net.Dialer.KeepAlive：30s(设置TCP的keepalive参数(syscall.TCP_KEEPINTVL以及syscall.TCP_KEEPIDLE))

在设置了上述http timeout参数后，理论上访问分布式锁10s超时后，http client会将底层的tcp socket给关闭，但是这里一直没有关闭(一直处于ESTABLISHED状态)，是什么原因呢？

这里分析源码，可以看出HTTP/2在请求超时后并不会关闭底层的socket，而是继续使用；而HTTP/1则会在请求超时后将使用的tcp socket关闭

那么这里的现象就可以解释如下：

- 由于Controller使用了HTTP/2协议，在10s的请求超时后并没有关闭tcp socket，而是继续使用
- 同时由于每隔3s重试一次请求(获取分布式lock)，导致TCP keepalive没办法触发(30s+30s*9=300s, 也即5mins)

```c
tcp_keepalive_time = 30s
tcp_keepalive_intvl = 30s
tcp_keepalive_probes = 9
```

**但是上面的两点并没有解释为什么15分钟后tcp socket会消失，timeout现象消失**

于是抱着先解决问题后查找原因的想法尝试先关闭HTTP/2协议，看看问题是否解决？

尝试设置Controller deployment环境变量，禁止HTTP/2，切换HTTP/1：

```yaml
spec:
  containers:
  - args:
    ...
    env:
    - name: DISABLE_HTTP2
      value: "true"
```

重新部署后，再次测试，现象如下：

```sh
2020-08-03 16:00:52.817 info    failed to acquire lease /controller
2020-08-03 16:00:55.041 info    lock is held by controller-7bb867f56f-x94m4_366f0ee0-fc9c-4bff-bb4f-48dc00e995bb and has not yet expired
2020-08-03 16:00:55.041 info    failed to acquire lease /controller
2020-08-03 16:01:08.583 error   error retrieving resource lock /controller: Get https://aa:443/api/v1/namespaces/default/configmaps/controller?timeout=10s: net/http: request canceled (Client.Timeout exceeded while awaiting headers)
2020-08-03 16:01:08.583 info    failed to acquire lease /controller
2020-08-03 16:01:22.665 error   error retrieving resource lock /controller: Get https://aa:443/api/v1/namespaces/default/configmaps/controller?timeout=10s: context deadline exceeded (Client.Timeout exceeded while awaiting headers)
2020-08-03 16:01:22.665 info    failed to acquire lease /controller
2020-08-03 16:01:36.472 error   error retrieving resource lock /controller: Get https://aa:443/api/v1/namespaces/default/configmaps/controller?timeout=10s: context deadline exceeded (Client.Timeout exceeded while awaiting headers)
2020-08-03 16:01:36.472 info    failed to acquire lease /controller
2020-08-03 16:01:39.980 info    lock is held by controller-7bb867f56f-x94m4_366f0ee0-fc9c-4bff-bb4f-48dc00e995bb and has not yet expired
2020-08-03 16:01:39.980 info    failed to acquire lease /controller
```

抓包如下：

```sh
192.168.2.148.443 > 192.168.1.204.50254: Flags [P.], cksum 0xc4a2 (correct), seq 136132035:136132691, ack 3350711668, win 319, options [nop,nop,TS val 645418 ecr 262286621], length 656
192.168.2.148.443 > 192.168.1.204.50254: Flags [P.], cksum 0xc4a2 (correct), seq 136132035:136132691, ack 3350711668, win 319, options [nop,nop,TS val 645418 ecr 262286621], length 656
192.168.255.220.443 > 192.168.1.204.50254: Flags [P.], cksum 0xc759 (correct), seq 136132035:136132691, ack 3350711668, win 319, options [nop,nop,TS val 645418 ecr 262286621], length 656
192.168.1.204.50254 > 192.168.255.220.443: Flags [.], cksum 0x5a08 (incorrect -> 0x4745), seq 3350711668, ack 136132691, win 210, options [nop,nop,TS val 262286624 ecr 645418], length 0
192.168.1.204.50254 > 192.168.2.148.443: Flags [.], cksum 0x5cbf (incorrect -> 0x448e), seq 3350711668, ack 136132691, win 210, options [nop,nop,TS val 262286624 ecr 645418], length 0
192.168.1.204.50254 > 192.168.2.148.443: Flags [.], cksum 0x448e (correct), seq 3350711668, ack 136132691, win 210, options [nop,nop,TS val 262286624 ecr 645418], length 0
192.168.1.204.50254 > 192.168.255.220.443: Flags [P.], cksum 0x5b37 (incorrect -> 0x790a), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262290166 ecr 645418], length 303
192.168.1.204.50254 > 192.168.2.148.443: Flags [P.], cksum 0x5dee (incorrect -> 0x7653), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262290166 ecr 645418], length 303
192.168.1.204.50254 > 192.168.2.148.443: Flags [P.], cksum 0x7653 (correct), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262290166 ecr 645418], length 303
192.168.1.204.50254 > 192.168.255.220.443: Flags [P.], cksum 0x5b37 (incorrect -> 0x783f), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262290369 ecr 645418], length 303
192.168.1.204.50254 > 192.168.2.148.443: Flags [P.], cksum 0x5dee (incorrect -> 0x7588), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262290369 ecr 645418], length 303
192.168.1.204.50254 > 192.168.2.148.443: Flags [P.], cksum 0x7588 (correct), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262290369 ecr 645418], length 303
192.168.1.204.50254 > 192.168.255.220.443: Flags [P.], cksum 0x5b37 (incorrect -> 0x7774), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262290572 ecr 645418], length 303
192.168.1.204.50254 > 192.168.2.148.443: Flags [P.], cksum 0x5dee (incorrect -> 0x74bd), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262290572 ecr 645418], length 303
192.168.1.204.50254 > 192.168.2.148.443: Flags [P.], cksum 0x74bd (correct), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262290572 ecr 645418], length 303
192.168.1.204.50254 > 192.168.255.220.443: Flags [P.], cksum 0x5b37 (incorrect -> 0x75dd), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262290979 ecr 645418], length 303
192.168.1.204.50254 > 192.168.2.148.443: Flags [P.], cksum 0x5dee (incorrect -> 0x7326), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262290979 ecr 645418], length 303
192.168.1.204.50254 > 192.168.2.148.443: Flags [P.], cksum 0x7326 (correct), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262290979 ecr 645418], length 303
192.168.1.204.50254 > 192.168.255.220.443: Flags [P.], cksum 0x5b37 (incorrect -> 0x72b0), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262291792 ecr 645418], length 303
192.168.1.204.50254 > 192.168.2.148.443: Flags [P.], cksum 0x5dee (incorrect -> 0x6ff9), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262291792 ecr 645418], length 303
192.168.1.204.50254 > 192.168.2.148.443: Flags [P.], cksum 0x6ff9 (correct), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262291792 ecr 645418], length 303
192.168.1.204.50254 > 192.168.255.220.443: Flags [P.], cksum 0x5b37 (incorrect -> 0x6c54), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262293420 ecr 645418], length 303
192.168.1.204.50254 > 192.168.2.148.443: Flags [P.], cksum 0x5dee (incorrect -> 0x699d), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262293420 ecr 645418], length 303
192.168.1.204.50254 > 192.168.2.148.443: Flags [P.], cksum 0x699d (correct), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262293420 ecr 645418], length 303
192.168.1.204.50254 > 192.168.255.220.443: Flags [P.], cksum 0x5b37 (incorrect -> 0x5fa0), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262296672 ecr 645418], length 303
192.168.1.204.50254 > 192.168.2.148.443: Flags [P.], cksum 0x5dee (incorrect -> 0x5ce9), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262296672 ecr 645418], length 303
192.168.1.204.50254 > 192.168.2.148.443: Flags [P.], cksum 0x5ce9 (correct), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262296672 ecr 645418], length 303
192.168.1.204.50254 > 192.168.255.220.443: Flags [FP.], cksum 0x5a27 (incorrect -> 0x1466), seq 3350711971:3350712002, ack 136132691, win 210, options [nop,nop,TS val 262300166 ecr 645418], length 31
192.168.1.204.50254 > 192.168.2.148.443: Flags [FP.], cksum 0x5cde (incorrect -> 0x11af), seq 3350711971:3350712002, ack 136132691, win 210, options [nop,nop,TS val 262300166 ecr 645418], length 31
192.168.1.204.50254 > 192.168.2.148.443: Flags [FP.], cksum 0x11af (correct), seq 3350711971:3350712002, ack 136132691, win 210, options [nop,nop,TS val 262300166 ecr 645418], length 31
192.168.1.204.50254 > 192.168.255.220.443: Flags [FP.], cksum 0x5b56 (incorrect -> 0xa413), seq 3350711668:3350712002, ack 136132691, win 210, options [nop,nop,TS val 262303184 ecr 645418], length 334
192.168.1.204.50254 > 192.168.2.148.443: Flags [FP.], cksum 0x5e0d (incorrect -> 0xa15c), seq 3350711668:3350712002, ack 136132691, win 210, options [nop,nop,TS val 262303184 ecr 645418], length 334
192.168.1.204.50254 > 192.168.2.148.443: Flags [FP.], cksum 0xa15c (correct), seq 3350711668:3350712002, ack 136132691, win 210, options [nop,nop,TS val 262303184 ecr 645418], length 334
192.168.1.204.51328 > 192.168.255.220.443: Flags [S], cksum 0x5a10 (incorrect -> 0xcf6c), seq 1911738311, win 28200, options [mss 1410,sackOK,TS val 262304249 ecr 0,nop,wscale 9], length 0
192.168.1.204.51328 > 192.168.2.148.443: Flags [S], cksum 0x5cc7 (incorrect -> 0xccb5), seq 1911738311, win 28200, options [mss 1410,sackOK,TS val 262304249 ecr 0,nop,wscale 9], length 0
192.168.1.204.51328 > 192.168.2.148.443: Flags [S], cksum 0xccb5 (correct), seq 1911738311, win 28200, options [mss 1410,sackOK,TS val 262304249 ecr 0,nop,wscale 9], length 0
192.168.1.204.51328 > 192.168.255.220.443: Flags [S], cksum 0x5a10 (incorrect -> 0xcb81), seq 1911738311, win 28200, options [mss 1410,sackOK,TS val 262305252 ecr 0,nop,wscale 9], length 0
192.168.1.204.51328 > 192.168.2.148.443: Flags [S], cksum 0x5cc7 (incorrect -> 0xc8ca), seq 1911738311, win 28200, options [mss 1410,sackOK,TS val 262305252 ecr 0,nop,wscale 9], length 0
192.168.1.204.51328 > 192.168.2.148.443: Flags [S], cksum 0xc8ca (correct), seq 1911738311, win 28200, options [mss 1410,sackOK,TS val 262305252 ecr 0,nop,wscale 9], length 0
192.168.1.204.51328 > 192.168.255.220.443: Flags [S], cksum 0x5a10 (incorrect -> 0xc3ad), seq 1911738311, win 28200, options [mss 1410,sackOK,TS val 262307256 ecr 0,nop,wscale 9], length 0
192.168.1.204.51328 > 192.168.2.148.443: Flags [S], cksum 0x5cc7 (incorrect -> 0xc0f6), seq 1911738311, win 28200, options [mss 1410,sackOK,TS val 262307256 ecr 0,nop,wscale 9], length 0
192.168.1.204.51328 > 192.168.2.148.443: Flags [S], cksum 0xc0f6 (correct), seq 1911738311, win 28200, options [mss 1410,sackOK,TS val 262307256 ecr 0,nop,wscale 9], length 0
192.168.1.204.51328 > 192.168.255.220.443: Flags [S], cksum 0x5a10 (incorrect -> 0xb405), seq 1911738311, win 28200, options [mss 1410,sackOK,TS val 262311264 ecr 0,nop,wscale 9], length 0
192.168.1.204.51328 > 192.168.2.148.443: Flags [S], cksum 0x5cc7 (incorrect -> 0xb14e), seq 1911738311, win 28200, options [mss 1410,sackOK,TS val 262311264 ecr 0,nop,wscale 9], length 0
192.168.1.204.51328 > 192.168.2.148.443: Flags [S], cksum 0xb14e (correct), seq 1911738311, win 28200, options [mss 1410,sackOK,TS val 262311264 ecr 0,nop,wscale 9], length 0
192.168.1.204.50254 > 192.168.255.220.443: Flags [FP.], cksum 0x5b56 (incorrect -> 0x7123), seq 3350711668:3350712002, ack 136132691, win 210, options [nop,nop,TS val 262316224 ecr 645418], length 334
192.168.1.204.50254 > 192.168.2.148.443: Flags [FP.], cksum 0x5e0d (incorrect -> 0x6e6c), seq 3350711668:3350712002, ack 136132691, win 210, options [nop,nop,TS val 262316224 ecr 645418], length 334
192.168.1.204.50254 > 192.168.2.148.443: Flags [FP.], cksum 0x6e6c (correct), seq 3350711668:3350712002, ack 136132691, win 210, options [nop,nop,TS val 262316224 ecr 645418], length 334
192.168.1.204.51482 > 192.168.255.220.443: Flags [S], cksum 0x5a10 (incorrect -> 0x1da9), seq 276343932, win 28200, options [mss 1410,sackOK,TS val 262318056 ecr 0,nop,wscale 9], length 0
192.168.1.204.51482 > 192.168.2.148.443: Flags [S], cksum 0x5cc7 (incorrect -> 0x1af2), seq 276343932, win 28200, options [mss 1410,sackOK,TS val 262318056 ecr 0,nop,wscale 9], length 0
192.168.1.204.51482 > 192.168.2.148.443: Flags [S], cksum 0x1af2 (correct), seq 276343932, win 28200, options [mss 1410,sackOK,TS val 262318056 ecr 0,nop,wscale 9], length 0
192.168.1.204.51482 > 192.168.255.220.443: Flags [S], cksum 0x5a10 (incorrect -> 0x19bf), seq 276343932, win 28200, options [mss 1410,sackOK,TS val 262319058 ecr 0,nop,wscale 9], length 0
192.168.1.204.51482 > 192.168.2.148.443: Flags [S], cksum 0x5cc7 (incorrect -> 0x1708), seq 276343932, win 28200, options [mss 1410,sackOK,TS val 262319058 ecr 0,nop,wscale 9], length 0
192.168.1.204.51482 > 192.168.2.148.443: Flags [S], cksum 0x1708 (correct), seq 276343932, win 28200, options [mss 1410,sackOK,TS val 262319058 ecr 0,nop,wscale 9], length 0
192.168.1.204.51482 > 192.168.255.220.443: Flags [S], cksum 0x5a10 (incorrect -> 0x11e9), seq 276343932, win 28200, options [mss 1410,sackOK,TS val 262321064 ecr 0,nop,wscale 9], length 0
192.168.1.204.51482 > 192.168.2.148.443: Flags [S], cksum 0x5cc7 (incorrect -> 0x0f32), seq 276343932, win 28200, options [mss 1410,sackOK,TS val 262321064 ecr 0,nop,wscale 9], length 0
192.168.1.204.51482 > 192.168.2.148.443: Flags [S], cksum 0x0f32 (correct), seq 276343932, win 28200, options [mss 1410,sackOK,TS val 262321064 ecr 0,nop,wscale 9], length 0
192.168.1.204.51482 > 192.168.255.220.443: Flags [S], cksum 0x5a10 (incorrect -> 0x0241), seq 276343932, win 28200, options [mss 1410,sackOK,TS val 262325072 ecr 0,nop,wscale 9], length 0
192.168.1.204.51482 > 192.168.2.148.443: Flags [S], cksum 0x5cc7 (incorrect -> 0xff89), seq 276343932, win 28200, options [mss 1410,sackOK,TS val 262325072 ecr 0,nop,wscale 9], length 0
192.168.1.204.51482 > 192.168.2.148.443: Flags [S], cksum 0xff89 (correct), seq 276343932, win 28200, options [mss 1410,sackOK,TS val 262325072 ecr 0,nop,wscale 9], length 0
192.168.1.204.51674 > 192.168.255.220.443: Flags [S], cksum 0x5a10 (incorrect -> 0xf7fc), seq 3929326320, win 28200, options [mss 1410,sackOK,TS val 262331555 ecr 0,nop,wscale 9], length 0
192.168.1.204.51674 > 192.168.1.203.443: Flags [S], cksum 0x5bfe (incorrect -> 0xf60e), seq 3929326320, win 28200, options [mss 1410,sackOK,TS val 262331555 ecr 0,nop,wscale 9], length 0
192.168.1.203.443 > 192.168.1.204.51674: Flags [S.], cksum 0x5bfe (incorrect -> 0x33cf), seq 1019320855, ack 3929326321, win 27960, options [mss 1410,sackOK,TS val 262331555 ecr 262331555,nop,wscale 9], length 0
192.168.255.220.443 > 192.168.1.204.51674: Flags [S.], cksum 0x5a10 (incorrect -> 0x35bd), seq 1019320855, ack 3929326321, win 27960, options [mss 1410,sackOK,TS val 262331555 ecr 262331555,nop,wscale 9], length 0
192.168.1.204.51674 > 192.168.255.220.443: Flags [.], cksum 0x5a08 (incorrect -> 0xd159), seq 3929326321, ack 1019320856, win 56, options [nop,nop,TS val 262331555 ecr 262331555], length 0
192.168.1.204.51674 > 192.168.1.203.443: Flags [.], cksum 0x5bf6 (incorrect -> 0xcf6b), seq 3929326321, ack 1019320856, win 56, options [nop,nop,TS val 262331555 ecr 262331555], length 0
192.168.1.204.51674 > 192.168.255.220.443: Flags [P.], cksum 0x5adc (incorrect -> 0x4fb3), seq 3929326321:3929326533, ack 1019320856, win 56, options [nop,nop,TS val 262331555 ecr 262331555], length 212
192.168.1.204.51674 > 192.168.1.203.443: Flags [P.], cksum 0x5cca (incorrect -> 0x4dc5), seq 3929326321:3929326533, ack 1019320856, win 56, options [nop,nop,TS val 262331555 ecr 262331555], length 212
192.168.1.203.443 > 192.168.1.204.51674: Flags [.], cksum 0x5bf6 (incorrect -> 0xce96), seq 1019320856, ack 3929326533, win 57, options [nop,nop,TS val 262331555 ecr 262331555], length 0
192.168.255.220.443 > 192.168.1.204.51674: Flags [.], cksum 0x5a08 (incorrect -> 0xd084), seq 1019320856, ack 3929326533, win 57, options [nop,nop,TS val 262331555 ecr 262331555], length 0
192.168.1.203.443 > 192.168.1.204.51674: Flags [P.], cksum 0x63d0 (incorrect -> 0x71e7), seq 1019320856:1019322866, ack 3929326533, win 57, options [nop,nop,TS val 262331557 ecr 262331555], length 2010
192.168.255.220.443 > 192.168.1.204.51674: Flags [P.], cksum 0x61e2 (incorrect -> 0x73d5), seq 1019320856:1019322866, ack 3929326533, win 57, options [nop,nop,TS val 262331557 ecr 262331555], length 2010
192.168.1.204.51674 > 192.168.255.220.443: Flags [.], cksum 0x5a08 (incorrect -> 0xc8a0), seq 3929326533, ack 1019322866, win 63, options [nop,nop,TS val 262331557 ecr 262331557], length 0
192.168.1.204.51674 > 192.168.1.203.443: Flags [.], cksum 0x5bf6 (incorrect -> 0xc6b2), seq 3929326533, ack 1019322866, win 63, options [nop,nop,TS val 262331557 ecr 262331557], length 0
192.168.1.204.51674 > 192.168.255.220.443: Flags [P.], cksum 0x5ea1 (incorrect -> 0xd7e0), seq 3929326533:3929327710, ack 1019322866, win 63, options [nop,nop,TS val 262331560 ecr 262331557], length 1177
192.168.1.204.51674 > 192.168.1.203.443: Flags [P.], cksum 0x608f (incorrect -> 0xd5f2), seq 3929326533:3929327710, ack 1019322866, win 63, options [nop,nop,TS val 262331560 ecr 262331557], length 1177
192.168.1.203.443 > 192.168.1.204.51674: Flags [P.], cksum 0x5c29 (incorrect -> 0xdc6f), seq 1019322866:1019322917, ack 3929327710, win 63, options [nop,nop,TS val 262331560 ecr 262331560], length 51
192.168.255.220.443 > 192.168.1.204.51674: Flags [P.], cksum 0x5a3b (incorrect -> 0xde5d), seq 1019322866:1019322917, ack 3929327710, win 63, options [nop,nop,TS val 262331560 ecr 262331560], length 51
192.168.1.204.51674 > 192.168.255.220.443: Flags [P.], cksum 0x5b37 (incorrect -> 0x9054), seq 3929327710:3929328013, ack 1019322917, win 63, options [nop,nop,TS val 262331561 ecr 262331560], length 303
192.168.1.204.51674 > 192.168.1.203.443: Flags [P.], cksum 0x5d25 (incorrect -> 0x8e66), seq 3929327710:3929328013, ack 1019322917, win 63, options [nop,nop,TS val 262331561 ecr 262331560], length 303
192.168.1.203.443 > 192.168.1.204.51674: Flags [P.], cksum 0x5e86 (incorrect -> 0x4750), seq 1019322917:1019323573, ack 3929328013, win 67, options [nop,nop,TS val 262331562 ecr 262331561], length 656
192.168.255.220.443 > 192.168.1.204.51674: Flags [P.], cksum 0x5c98 (incorrect -> 0x493e), seq 1019322917:1019323573, ack 3929328013, win 67, options [nop,nop,TS val 262331562 ecr 262331561], length 656
192.168.1.204.51674 > 192.168.255.220.443: Flags [.], cksum 0x5a08 (incorrect -> 0xbfdd), seq 3929328013, ack 1019323573, win 69, options [nop,nop,TS val 262331602 ecr 262331562], length 0
192.168.1.204.51674 > 192.168.1.203.443: Flags [.], cksum 0x5bf6 (incorrect -> 0xbdef), seq 3929328013, ack 1019323573, win 69, options [nop,nop,TS val 262331602 ecr 262331562], length 0
```

可以看到这里面一共有三个timeout报错，依次对应抓包socket如下：

```sh
192.168.1.204.50254 > 192.168.2.148.443(宕机前的socket)
192.168.1.204.51328 > 192.168.2.148.443(TCP尝试三次握手Flags [S]，没有成功)
192.168.1.204.51482 > 192.168.2.148.443(TCP尝试三次握手Flags [S]，没有成功)

# TCP handshake
    _____                                                     _____
   |     |                                                   |     |
   |  A  |                                                   |  B  |
   |_____|                                                   |_____|
      ^                                                         ^
      |--->--->--->-------------- SYN -------------->--->--->---|
      |---<---<---<------------ SYN/ACK ------------<---<---<---|
      |--->--->--->-------------- ACK -------------->--->--->---|
```

后面日志正常，对应socket为：`192.168.1.204.51674 > 192.168.1.203.443`。其中`192.168.1.203.443`是其它非宕机母机上的AA

结合上述日志和抓包看，从16:00:53开始宕机，Controller在16:00:58开始尝试获取分布式锁，到16:01:08(10s间隔)超时，于是关闭socket(192.168.1.204.50254 > 192.168.2.148.443)

之后3s(16:01:11)开始第二轮获取，尝试创建TCP连接，这个时候由于没有超过40s的service ep剔除时间，192.168.2.148.443对应的DNAT链(KUBE-SEP-XXX)还存在iptables规则中，iptables轮询机制将192.168.255.220.443(Kubernetes aa service)'随机'转化成了192.168.2.148.443

但是由于192.168.2.148.443已经挂掉，所以TCP三次握手并没有成功，因此也看到了`Client.Timeout exceeded while awaiting headers`的错误(注意：**http.Client.Timeout包括TCP连接建立的时间，虽然设置为30s，但是这里http.Client.Timeout=10s会将其覆盖**)

之后3s(16:01:25)开始第三轮获取，原理类似，不再展开

直到第四次尝试(16:01:39)获取，这个时候已经触发endpoint controller剔除逻辑，于是iptables会将service转化为其它正常母机上的pod(192.168.1.203.443)。访问也就正常了

```sh
$ netstat -tnope 
Active Internet connections (w/o servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       User       Inode      PID/Program name     Timer
tcp        0      0 192.168.1.204:51674      192.168.255.220:443     ESTABLISHED 0          18568190   26201/controller    keepalive (10.22/0/0)
```

从上面的分析可以看出似乎禁用HTTP/2可以解决问题

另外，社区提交了一个PR专门用于解决`HTTP/2存在的无法移除底层half-closed tcp连接的问题`，如下：

> http2: perform connection health check
>
> > After the connection has been idle for a while, periodic pings are sentover the connection to check its health. Unhealthy connection is closedand removed from the connection pool.

目前最新版本的golang 1.15已经合并该PR，而 Kubernetes 社区也对应存在着类似的client-go issue (https://github.com/kubernetes/client-go/issues/374)。



### 分析 - 禁用HTTP/2不生效

准备按照如上`禁用HTTP/2`的方法对集群中其它Controller进行测试，发现cluster-coredns-controller(https://github.com/duyanghao/cluster-coredns-controller)在禁用HTTP/2之后依旧会出现15mins的不可用状态

如下是cluster-coredns-controller架构图：

![image-20210108152115502](D:\学习资料\笔记\k8s\k8s图\image-20210108152115502.png)

如下是母机宕机前controller相关信息：

```sh
$ kubectl get pods -o wide
cluster-coredns-controller-j82s8           1/1     Running     0          33s     192.168.1.225   10.0.0.2   <none>           <none>
cluster-coredns-controller-nppjv           1/1     Running     0          44s     192.168.2.165   10.0.0.3   <none>           <none>
cluster-coredns-controller-xnxp8           1/1     Running     0          30s     192.168.0.122   10.0.0.1   <none>           <none>

$ netstat -tnope
Active Internet connections (w/o servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       User       Inode      PID/Program name     Timer
tcp        0      0 192.168.0.122:46306      192.168.255.220:443     ESTABLISHED 0          583867749  726/./cluster-cored  keepalive (7.89/0/0)
tcp        0      0 192.168.0.122:55286      192.168.255.220:443     ESTABLISHED 0          583730245  726/./cluster-cored  keepalive (3.16/0/0)
```

日志如下：

```sh
I0803 09:26:07.795505       1 reflector.go:243] forcing resync
I0803 09:26:07.795602       1 controller.go:133] Watch Cluster: global Updated ...
I0803 09:26:07.799698       1 controller.go:227] Successfully synced 'global'
```

这里将10.0.0.3母机关机(模拟宕机)，cluster-coredns-controller现象如下：

```sh
$ log
I0803 09:26:07.795505       1 reflector.go:243] forcing resync
I0803 09:26:07.795602       1 controller.go:133] Watch Cluster: global Updated ...
I0803 09:26:07.799698       1 controller.go:227] Successfully synced 'global'

I0803 09:26:37.795700       1 reflector.go:243] forcing resync
I0803 09:26:37.795768       1 controller.go:133] Watch Cluster: global Updated ...
I0803 09:27:07.795908       1 reflector.go:243] forcing resync
I0803 09:27:07.795966       1 controller.go:133] Watch Cluster: global Updated ...

$ netstat -tnope
Active Internet connections (w/o servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       User       Inode      PID/Program name     Timer
tcp        0    261 192.168.0.122:46306      192.168.255.220:443     ESTABLISHED 0          583867749  726/./cluster-cored  on (81.21/9/0)
tcp        0      0 192.168.0.122:55286      192.168.255.220:443     ESTABLISHED 0          583730245  726/./cluster-cored  keepalive (19.00/0/4)

192.168.0.122.46306 > 192.168.255.220.443: Flags [P.], cksum 0x59bb (incorrect -> 0x9fc3), seq 2494056844:2494057105, ack 2492532730, win 349, options [nop,nop,TS val 208032352 ecr 1530395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0x5c80 (incorrect -> 0x9cfe), seq 2494056844:2494057105, ack 2492532730, win 349, options [nop,nop,TS val 208032352 ecr 1530395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0x9cfe (correct), seq 2494056844:2494057105, ack 2492532730, win 349, options [nop,nop,TS val 208032352 ecr 1530395], length 261
192.168.2.162.443 > 192.168.0.122.46306: Flags [.], cksum 0x742d (correct), seq 2492532730:2492534128, ack 2494057105, win 122, options [nop,nop,TS val 1560395 ecr 208032352], length 1398
192.168.2.162.443 > 192.168.0.122.46306: Flags [.], cksum 0xb99d (correct), seq 2492536855:2492538253, ack 2494057105, win 122, options [nop,nop,TS val 1560395 ecr 208032352], length 1398
192.168.2.162.443 > 192.168.0.122.46306: Flags [.], cksum 0x449f (correct), seq 2492534128:2492535526, ack 2494057105, win 122, options [nop,nop,TS val 1560395 ecr 208032352], length 1398
192.168.2.162.443 > 192.168.0.122.46306: Flags [P.], cksum 0xffbc (correct), seq 2492538253:2492538869, ack 2494057105, win 122, options [nop,nop,TS val 1560395 ecr 208032352], length 616
192.168.2.162.443 > 192.168.0.122.46306: Flags [P.], cksum 0x344c (correct), seq 2492535526:2492536855, ack 2494057105, win 122, options [nop,nop,TS val 1560395 ecr 208032352], length 1329
192.168.2.162.443 > 192.168.0.122.46306: Flags [.], cksum 0x742d (correct), seq 2492532730:2492534128, ack 2494057105, win 122, options [nop,nop,TS val 1560395 ecr 208032352], length 1398
192.168.255.220.443 > 192.168.0.122.46306: Flags [.], cksum 0x76f2 (correct), seq 2492532730:2492534128, ack 2494057105, win 122, options [nop,nop,TS val 1560395 ecr 208032352], length 1398
192.168.2.162.443 > 192.168.0.122.46306: Flags [.], cksum 0xb99d (correct), seq 2492536855:2492538253, ack 2494057105, win 122, options [nop,nop,TS val 1560395 ecr 208032352], length 1398
192.168.255.220.443 > 192.168.0.122.46306: Flags [.], cksum 0xbc62 (correct), seq 2492536855:2492538253, ack 2494057105, win 122, options [nop,nop,TS val 1560395 ecr 208032352], length 1398
192.168.2.162.443 > 192.168.0.122.46306: Flags [.], cksum 0x449f (correct), seq 2492534128:2492535526, ack 2494057105, win 122, options [nop,nop,TS val 1560395 ecr 208032352], length 1398
192.168.255.220.443 > 192.168.0.122.46306: Flags [.], cksum 0x4764 (correct), seq 2492534128:2492535526, ack 2494057105, win 122, options [nop,nop,TS val 1560395 ecr 208032352], length 1398
192.168.2.162.443 > 192.168.0.122.46306: Flags [P.], cksum 0xffbc (correct), seq 2492538253:2492538869, ack 2494057105, win 122, options [nop,nop,TS val 1560395 ecr 208032352], length 616
192.168.255.220.443 > 192.168.0.122.46306: Flags [P.], cksum 0x0282 (correct), seq 2492538253:2492538869, ack 2494057105, win 122, options [nop,nop,TS val 1560395 ecr 208032352], length 616
192.168.2.162.443 > 192.168.0.122.46306: Flags [P.], cksum 0x344c (correct), seq 2492535526:2492536855, ack 2494057105, win 122, options [nop,nop,TS val 1560395 ecr 208032352], length 1329
192.168.255.220.443 > 192.168.0.122.46306: Flags [P.], cksum 0x3711 (correct), seq 2492535526:2492536855, ack 2494057105, win 122, options [nop,nop,TS val 1560395 ecr 208032352], length 1329
192.168.0.122.46306 > 192.168.255.220.443: Flags [.], cksum 0x58b6 (incorrect -> 0x93a4), seq 2494057105, ack 2492534128, win 347, options [nop,nop,TS val 208032356 ecr 1560395], length 0
192.168.0.122.46306 > 192.168.2.162.443: Flags [.], cksum 0x5b7b (incorrect -> 0x90df), seq 2494057105, ack 2492534128, win 347, options [nop,nop,TS val 208032356 ecr 1560395], length 0
192.168.0.122.46306 > 192.168.2.162.443: Flags [.], cksum 0x90df (correct), seq 2494057105, ack 2492534128, win 347, options [nop,nop,TS val 208032356 ecr 1560395], length 0
192.168.0.122.46306 > 192.168.255.220.443: Flags [.], cksum 0x58c2 (incorrect -> 0xfec5), seq 2494057105, ack 2492534128, win 347, options [nop,nop,TS val 208032356 ecr 1560395,nop,nop,sack 1 {2492536855:2492538253}], length 0
192.168.0.122.46306 > 192.168.2.162.443: Flags [.], cksum 0x5b87 (incorrect -> 0xfc00), seq 2494057105, ack 2492534128, win 347, options [nop,nop,TS val 208032356 ecr 1560395,nop,nop,sack 1 {2492536855:2492538253}], length 0
192.168.0.122.46306 > 192.168.2.162.443: Flags [.], cksum 0xfc00 (correct), seq 2494057105, ack 2492534128, win 347, options [nop,nop,TS val 208032356 ecr 1560395,nop,nop,sack 1 {2492536855:2492538253}], length 0
192.168.0.122.46306 > 192.168.255.220.443: Flags [.], cksum 0x58c2 (incorrect -> 0xf951), seq 2494057105, ack 2492535526, win 345, options [nop,nop,TS val 208032356 ecr 1560395,nop,nop,sack 1 {2492536855:2492538253}], length 0
192.168.0.122.46306 > 192.168.2.162.443: Flags [.], cksum 0x5b87 (incorrect -> 0xf68c), seq 2494057105, ack 2492535526, win 345, options [nop,nop,TS val 208032356 ecr 1560395,nop,nop,sack 1 {2492536855:2492538253}], length 0
192.168.0.122.46306 > 192.168.2.162.443: Flags [.], cksum 0xf68c (correct), seq 2494057105, ack 2492535526, win 345, options [nop,nop,TS val 208032356 ecr 1560395,nop,nop,sack 1 {2492536855:2492538253}], length 0
192.168.0.122.46306 > 192.168.255.220.443: Flags [.], cksum 0x58c2 (incorrect -> 0xf6e9), seq 2494057105, ack 2492535526, win 345, options [nop,nop,TS val 208032356 ecr 1560395,nop,nop,sack 1 {2492536855:2492538869}], length 0
192.168.0.122.46306 > 192.168.2.162.443: Flags [.], cksum 0x5b87 (incorrect -> 0xf424), seq 2494057105, ack 2492535526, win 345, options [nop,nop,TS val 208032356 ecr 1560395,nop,nop,sack 1 {2492536855:2492538869}], length 0
192.168.0.122.46306 > 192.168.2.162.443: Flags [.], cksum 0xf424 (correct), seq 2494057105, ack 2492535526, win 345, options [nop,nop,TS val 208032356 ecr 1560395,nop,nop,sack 1 {2492536855:2492538869}], length 0
192.168.0.122.46306 > 192.168.255.220.443: Flags [.], cksum 0x58b6 (incorrect -> 0x8127), seq 2494057105, ack 2492538869, win 339, options [nop,nop,TS val 208032356 ecr 1560395], length 0
192.168.0.122.46306 > 192.168.2.162.443: Flags [.], cksum 0x5b7b (incorrect -> 0x7e62), seq 2494057105, ack 2492538869, win 339, options [nop,nop,TS val 208032356 ecr 1560395], length 0
192.168.0.122.46306 > 192.168.2.162.443: Flags [.], cksum 0x7e62 (correct), seq 2494057105, ack 2492538869, win 339, options [nop,nop,TS val 208032356 ecr 1560395], length 0
192.168.0.122.55286 > 192.168.255.220.443: Flags [.], cksum 0x58b6 (incorrect -> 0x7c4d), seq 1519823272, ack 1455933629, win 349, options [nop,nop,TS val 208057856 ecr 1555815], length 0
192.168.0.122.55286 > 192.168.2.162.443: Flags [.], cksum 0x5b7b (incorrect -> 0x7988), seq 1519823272, ack 1455933629, win 349, options [nop,nop,TS val 208057856 ecr 1555815], length 0
192.168.0.122.55286 > 192.168.2.162.443: Flags [.], cksum 0x7988 (correct), seq 1519823272, ack 1455933629, win 349, options [nop,nop,TS val 208057856 ecr 1555815], length 0
192.168.2.162.443 > 192.168.0.122.55286: Flags [.], cksum 0x6580 (correct), seq 1455933629, ack 1519823273, win 204, options [nop,nop,TS val 1585896 ecr 207967512], length 0
192.168.2.162.443 > 192.168.0.122.55286: Flags [.], cksum 0x6580 (correct), seq 1455933629, ack 1519823273, win 204, options [nop,nop,TS val 1585896 ecr 207967512], length 0
192.168.255.220.443 > 192.168.0.122.55286: Flags [.], cksum 0x6845 (correct), seq 1455933629, ack 1519823273, win 204, options [nop,nop,TS val 1585896 ecr 207967512], length 0
...
192.168.0.122.46306 > 192.168.255.220.443: Flags [P.], cksum 0x59bb (incorrect -> 0xd5ec), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208062352 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0x5c80 (incorrect -> 0xd327), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208062352 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0xd327 (correct), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208062352 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.255.220.443: Flags [P.], cksum 0x59bb (incorrect -> 0xd521), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208062555 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0x5c80 (incorrect -> 0xd25c), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208062555 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0xd25c (correct), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208062555 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.255.220.443: Flags [P.], cksum 0x59bb (incorrect -> 0xd456), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208062758 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0x5c80 (incorrect -> 0xd191), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208062758 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0xd191 (correct), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208062758 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.255.220.443: Flags [P.], cksum 0x59bb (incorrect -> 0xd2bf), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208063165 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0x5c80 (incorrect -> 0xcffa), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208063165 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0xcffa (correct), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208063165 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.255.220.443: Flags [P.], cksum 0x59bb (incorrect -> 0xcf90), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208063980 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0x5c80 (incorrect -> 0xcccb), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208063980 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0xcccb (correct), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208063980 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.255.220.443: Flags [P.], cksum 0x59bb (incorrect -> 0xc934), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208065608 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0x5c80 (incorrect -> 0xc66f), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208065608 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0xc66f (correct), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208065608 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.255.220.443: Flags [P.], cksum 0x59bb (incorrect -> 0xbc7c), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208068864 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0x5c80 (incorrect -> 0xb9b7), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208068864 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0xb9b7 (correct), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208068864 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.255.220.443: Flags [P.], cksum 0x59bb (incorrect -> 0xa30c), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208075376 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0x5c80 (incorrect -> 0xa047), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208075376 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0xa047 (correct), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208075376 ecr 1560395], length 261
192.168.0.122.55286 > 192.168.255.220.443: Flags [.], cksum 0x58b6 (incorrect -> 0x914b), seq 1519823272, ack 1455933629, win 349, options [nop,nop,TS val 208087936 ecr 1585896], length 0
192.168.0.122.55286 > 192.168.2.162.443: Flags [.], cksum 0x5b7b (incorrect -> 0x8e86), seq 1519823272, ack 1455933629, win 349, options [nop,nop,TS val 208087936 ecr 1585896], length 0
192.168.0.122.55286 > 192.168.2.162.443: Flags [.], cksum 0x8e86 (correct), seq 1519823272, ack 1455933629, win 349, options [nop,nop,TS val 208087936 ecr 1585896], length 0
192.168.0.122.46306 > 192.168.255.220.443: Flags [P.], cksum 0x59bb (incorrect -> 0x703c), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208088384 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0x5c80 (incorrect -> 0x6d77), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208088384 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0x6d77 (correct), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208088384 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.255.220.443: Flags [P.], cksum 0x59bb (incorrect -> 0x0a7c), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208114432 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0x5c80 (incorrect -> 0x07b7), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208114432 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0x07b7 (correct), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208114432 ecr 1560395], length 261
```

5分钟后出现如下日志：

```sh
...
I0803 09:31:07.797244       1 reflector.go:243] forcing resync
I0803 09:31:07.797318       1 controller.go:133] Watch Cluster: global Updated ...
W0803 09:31:34.099349       1 reflector.go:302] watch of *v1.Cluster ended with: an error on the server ("unable to decode an event from the watch stream: read tcp 192.168.0.122:55286->192.168.255.220:443: read: connection timed out") has prevented the request from succeeding
...
```

另外，Controller的socket如下：

```sh
$ netstat -tnope
Active Internet connections (w/o servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       User       Inode      PID/Program name     Timer
tcp        0    261 192.168.0.122:46306      192.168.255.220:443     ESTABLISHED 0          583867749  726/./cluster-cored  on (98.33/14/0)
tcp        0      0 192.168.0.122:35258      192.168.255.220:443     ESTABLISHED 0          583982033  726/./cluster-cored  keepalive (10.97/0/0)
```

可以看到tcp socket：`192.168.0.122:55286->192.168.255.220:443`关闭了，并新产生了一个socket。但是Controller依旧异常

抓包如下：

```sh
192.168.0.122.35258 > 192.168.255.220.443: Flags [.], cksum 0x58b6 (incorrect -> 0x6c75), seq 2189163102, ack 1656563798, win 106, options [nop,nop,TS val 208844928 ecr 268182996], length 0
192.168.0.122.35258 > 192.168.1.226.443: Flags [.], cksum 0x5abb (incorrect -> 0x6a70), seq 2189163102, ack 1656563798, win 106, options [nop,nop,TS val 208844928 ecr 268182996], length 0
192.168.0.122.35258 > 192.168.1.226.443: Flags [.], cksum 0x6a70 (correct), seq 2189163102, ack 1656563798, win 106, options [nop,nop,TS val 208844928 ecr 268182996], length 0
192.168.1.226.443 > 192.168.0.122.35258: Flags [.], cksum 0xcaab (correct), seq 1656563798, ack 2189163103, win 76, options [nop,nop,TS val 268213076 ecr 208724707], length 0
192.168.1.226.443 > 192.168.0.122.35258: Flags [.], cksum 0xcaab (correct), seq 1656563798, ack 2189163103, win 76, options [nop,nop,TS val 268213076 ecr 208724707], length 0
192.168.255.220.443 > 192.168.0.122.35258: Flags [.], cksum 0xccb0 (correct), seq 1656563798, ack 2189163103, win 76, options [nop,nop,TS val 268213076 ecr 208724707], length 0
192.168.0.122.46306 > 192.168.255.220.443: Flags [P.], cksum 0x59bb (incorrect -> 0x7970), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208872448 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0x5c80 (incorrect -> 0x76ab), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208872448 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0x76ab (correct), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208872448 ecr 1560395], length 261
192.168.0.122.35258 > 192.168.255.220.443: Flags [.], cksum 0x58b6 (incorrect -> 0x8174), seq 2189163102, ack 1656563798, win 106, options [nop,nop,TS val 208875008 ecr 268213076], length 0
192.168.0.122.35258 > 192.168.1.226.443: Flags [.], cksum 0x5abb (incorrect -> 0x7f6f), seq 2189163102, ack 1656563798, win 106, options [nop,nop,TS val 208875008 ecr 268213076], length 0
192.168.0.122.35258 > 192.168.1.226.443: Flags [.], cksum 0x7f6f (correct), seq 2189163102, ack 1656563798, win 106, options [nop,nop,TS val 208875008 ecr 268213076], length 0
192.168.1.226.443 > 192.168.0.122.35258: Flags [.], cksum 0x552b (correct), seq 1656563798, ack 2189163103, win 76, options [nop,nop,TS val 268243156 ecr 208724707], length 0
192.168.1.226.443 > 192.168.0.122.35258: Flags [.], cksum 0x552b (correct), seq 1656563798, ack 2189163103, win 76, options [nop,nop,TS val 268243156 ecr 208724707], length 0
192.168.255.220.443 > 192.168.0.122.35258: Flags [.], cksum 0x5730 (correct), seq 1656563798, ack 2189163103, win 76, options [nop,nop,TS val 268243156 ecr 208724707], length 0
192.168.1.226.443 > 192.168.0.122.35258: Flags [.], cksum 0xe11f (correct), seq 1656563797, ack 2189163103, win 76, options [nop,nop,TS val 268272864 ecr 208724707], length 0
192.168.1.226.443 > 192.168.0.122.35258: Flags [.], cksum 0xe11f (correct), seq 1656563797, ack 2189163103, win 76, options [nop,nop,TS val 268272864 ecr 208724707], length 0
192.168.255.220.443 > 192.168.0.122.35258: Flags [.], cksum 0xe324 (correct), seq 1656563797, ack 2189163103, win 76, options [nop,nop,TS val 268272864 ecr 208724707], length 0
192.168.0.122.35258 > 192.168.255.220.443: Flags [.], cksum 0x58b6 (incorrect -> 0x97e7), seq 2189163103, ack 1656563798, win 106, options [nop,nop,TS val 208904715 ecr 268243156], length 0
192.168.0.122.35258 > 192.168.1.226.443: Flags [.], cksum 0x5abb (incorrect -> 0x95e2), seq 2189163103, ack 1656563798, win 106, options [nop,nop,TS val 208904715 ecr 268243156], length 0
192.168.0.122.35258 > 192.168.1.226.443: Flags [.], cksum 0x95e2 (correct), seq 2189163103, ack 1656563798, win 106, options [nop,nop,TS val 208904715 ecr 268243156], length
```

可以看到新产生的socket访问正常，但是旧的socket`192.168.0.122.46306 > 192.168.255.220.443`依旧存在访问。现象和上面的Controller如出一撤，区别是这里禁用了HTTP/2

过了15mins后(17:26:31开始宕机，17:42:05恢复)，cluster-coredns-controller日志正常：

```sh
I0803 09:40:35.112293       1 controller.go:133] Watch Cluster: global Updated ...
I0803 09:41:05.112424       1 reflector.go:243] forcing resync
I0803 09:41:05.112484       1 controller.go:133] Watch Cluster: global Updated ...
I0803 09:41:35.112637       1 reflector.go:243] forcing resync
I0803 09:41:35.112751       1 controller.go:133] Watch Cluster: global Updated ...

I0803 09:42:05.112807       1 reflector.go:243] forcing resync
I0803 09:42:05.112908       1 controller.go:133] Watch Cluster: global Updated ...
I0803 09:42:08.221876       1 controller.go:227] Successfully synced 'global'

$ netstat -tnope
Active Internet connections (w/o servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       User       Inode      PID/Program name     Timer
tcp        0      0 192.168.0.122:35258      192.168.255.220:443     ESTABLISHED 0          583982033  726/./cluster-cored  keepalive (0.84/0/0)
tcp        0      0 192.168.0.122:54190      192.168.255.220:443     ESTABLISHED 0          584112800  726/./cluster-cored  keepalive (25.03/0/0)
```

对应的socket也被删除掉了，并新创建了一个新的socket：`192.168.0.122.54190 > 192.168.255.220.443`

查cluster-coredns-controller代码，发现是http.Client.Timeout没有设置，加上之后，controller force rsync会存在问题，每隔http.Client.Timeout时间会出现request Timeout(没有宕机情况下)，同时关闭底层的tcp socket(也即需要调整timeout参数)

分析到这里就会有如下疑问：

- 其中一个socket 5mins后超时关闭，这个5mins哪里触发的？(并没有设置Timeout)
- 另外一个socket 15mins后超时关闭，这个15mins哪里来的？



### 分析 - TCP ARQ&keepalive

从上面的分析可以看出来：似乎不同的应用上层有不同的Timeout设置(包括no timeout)，但是都会存在15mins的超时，看来从上层分析是看不出来问题了，必须追到底层

于是写了一个http request demo例子(为了脱离业务逻辑，追踪底层)模拟访问AA，希望可以复现15 mins的超时：

```go
// httptest.go
import (
        "crypto/tls"
        "fmt"
        "io/ioutil"
        "net"
        "net/http"
        "time"
)

func doRequest(client *http.Client, url string) {
        fmt.Printf("time: %v, start to get ...\n", time.Now())

        req, err := http.NewRequest("GET", url, nil)
        if err != nil {
                fmt.Printf("time: %v, http NewRequest fail: %v\n", time.Now(), err)
                return
        }
        rsp, err := client.Do(req)
        if err != nil {
                fmt.Printf("time: %v, http Client Do request fail: %v\n", time.Now(), err)
                return
        }
        fmt.Printf("time: %v, http Client Do request successfully\n", time.Now())

        buf, err := ioutil.ReadAll(rsp.Body)
        if err != nil {
                fmt.Printf("time: %v, read http response body fail: %v\n", time.Now(), err)
                return
        }
        defer rsp.Body.Close()
        fmt.Printf("\n end-time: %v \n%s -- \n\n\n", time.Now(), string(buf))
}

func main() {
        tr := &http.Transport{
                TLSClientConfig:   &tls.Config{InsecureSkipVerify: true},
                DialContext: (&net.Dialer{
                        Timeout:   10 * time.Second,
                        KeepAlive: 5 * time.Second,
                        DualStack: true,
                }).DialContext,
        }

        client := &http.Client{Transport: tr}

        for i := 0; i < 500; i++ {
                doRequest(client, "https://10.0.0.3:6443")
                time.Sleep(5 * time.Second)
        }
}
```

在Kubernetes集群外母机10.0.0.126上运行httptest访问10.0.0.3:6443，查看socket如下：

```sh
$ netstat -tnope|grep httptest
tcp        0    127 10.0.0.126:42174       10.0.0.3:6443       ESTABLISHED 0          783573965  16334/./httptest     keepalive (0.48/0/0)
```

关闭母机10.0.0.3，日志如下：

```sh
time: 2020-08-04 15:26:44.476931372 +0800 CST m=+20.008447810, start to get ...
time: 2020-08-04 15:26:44.477686648 +0800 CST m=+20.009203129, http Client Do request successfully

 end-time: 2020-08-04 15:26:44.477726859 +0800 CST m=+20.009243329 
{"kind":"Status","apiVersion":"v1","metadata":{},"status":"Failure","message":"forbidden: User \"system:anonymous\" cannot get path \"/\"","reason":"Forbidden","details":{},"code":403}
 -- 


time: 2020-08-04 15:26:49.477844198 +0800 CST m=+25.009360665, start to get ...
```

抓包如下：

```sh
15:26:34.475194 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [P.], seq 3028317097:3028317224, ack 3343196009, win 292, options [nop,nop,TS val 431222239 ecr 2745903], length 127
15:26:34.475767 IP 10.0.0.3.sun-sr-https > 10.0.0.126.42174: Flags [P.], seq 1:364, ack 127, win 59, options [nop,nop,TS val 2750904 ecr 431222239], length 363
15:26:34.475798 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [.], ack 364, win 314, options [nop,nop,TS val 431222240 ecr 2750904], length 0
15:26:39.476197 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [P.], seq 127:254, ack 364, win 314, options [nop,nop,TS val 431227240 ecr 2750904], length 127
15:26:39.476718 IP 10.0.0.3.sun-sr-https > 10.0.0.126.42174: Flags [P.], seq 364:727, ack 254, win 59, options [nop,nop,TS val 2755905 ecr 431227240], length 363
15:26:39.476750 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [.], ack 727, win 335, options [nop,nop,TS val 431227241 ecr 2755905], length 0
15:26:44.477031 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [P.], seq 254:381, ack 727, win 335, options [nop,nop,TS val 431232241 ecr 2755905], length 127
15:26:44.477581 IP 10.0.0.3.sun-sr-https > 10.0.0.126.42174: Flags [P.], seq 727:1090, ack 381, win 59, options [nop,nop,TS val 2760906 ecr 431232241], length 363
15:26:44.477613 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [.], ack 1090, win 356, options [nop,nop,TS val 431232241 ecr 2760906], length 0


15:26:49.477981 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [P.], seq 381:508, ack 1090, win 356, options [nop,nop,TS val 431237242 ecr 2760906], length 127
15:26:49.678680 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [P.], seq 381:508, ack 1090, win 356, options [nop,nop,TS val 431237443 ecr 2760906], length 127
15:26:49.879674 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [P.], seq 381:508, ack 1090, win 356, options [nop,nop,TS val 431237644 ecr 2760906], length 127
15:26:50.282678 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [P.], seq 381:508, ack 1090, win 356, options [nop,nop,TS val 431238047 ecr 2760906], length 127
15:26:51.087680 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [P.], seq 381:508, ack 1090, win 356, options [nop,nop,TS val 431238852 ecr 2760906], length 127
15:26:52.699685 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [P.], seq 381:508, ack 1090, win 356, options [nop,nop,TS val 431240464 ecr 2760906], length 127
15:26:55.923668 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [P.], seq 381:508, ack 1090, win 356, options [nop,nop,TS val 431243688 ecr 2760906], length 127
15:27:02.379682 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [P.], seq 381:508, ack 1090, win 356, options [nop,nop,TS val 431250144 ecr 2760906], length 127
15:27:15.275684 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [P.], seq 381:508, ack 1090, win 356, options [nop,nop,TS val 431263040 ecr 2760906], length 127
15:27:41.067690 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [P.], seq 381:508, ack 1090, win 356, options [nop,nop,TS val 431288832 ecr 2760906], length 127
15:28:32.651690 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [P.], seq 381:508, ack 1090, win 356, options [nop,nop,TS val 431340416 ecr 2760906], length 127
15:30:15.691692 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [P.], seq 381:508, ack 1090, win 356, options [nop,nop,TS val 431443456 ecr 2760906], length 127

15:32:16.011697 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [P.], seq 381:508, ack 1090, win 356, options [nop,nop,TS val 431563776 ecr 2760906], length 127

15:34:16.331690 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [P.], seq 381:508, ack 1090, win 356, options [nop,nop,TS val 431684096 ecr 2760906], length 127

15:36:16.651678 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [P.], seq 381:508, ack 1090, win 356, options [nop,nop,TS val 431804416 ecr 2760906], length 127

15:38:16.971688 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [P.], seq 381:508, ack 1090, win 356, options [nop,nop,TS val 431924736 ecr 2760906], length 127

15:40:17.291684 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [P.], seq 381:508, ack 1090, win 356, options [nop,nop,TS val 432045056 ecr 2760906], length 127
```

socket如下：

```sh
$ netstat -tnope|grep httptest
tcp        0    127 10.0.0.126:42174       10.0.0.3:6443       ESTABLISHED 0          783573965  16334/./httptest     on (58.06/12/0)
```

经过15mins，程序报错如下：

```sh
...
time: 2020-08-04 15:26:49.477844198 +0800 CST m=+25.009360665, start to get ...
time: 2020-08-04 15:42:27.612016356 +0800 CST m=+963.143532890, http Client Do request fail: Get https://10.0.0.3:6443: dial tcp 10.0.0.3:6443: i/o timeout
time: 2020-08-04 15:42:32.612151633 +0800 CST m=+968.143668107, start to get ...
time: 2020-08-04 15:42:42.612363196 +0800 CST m=+978.143879643, http Client Do request fail: Get https://10.0.0.3:6443: dial tcp 10.0.0.3:6443: i/o timeout
```

同时，tcp socket：`10.0.0.126.42174 > 10.0.0.3.6443` 消失

可以看到在宕机后(15mins内)，httptest依旧在使用tcp socket：`10.0.0.126.42174 > 10.0.0.3.6443`。并不断地间隔发包，那么这些包是什么？？？

是tcp keepalive的包吗？显然不是，因为keepalive的包长度应该为0且是按照固定间隔发送(tcp_keepalive_intvl)；但是这里的包长度为127，且前面间隔短，后面间隔长(最长两分钟)

观察上述包内容，看起来是tcp ARQ(自动重传请求)的包，于是开始研究tcp ARQ与15mins的关系

TCP使用滑动窗口和ARQ机制保障可靠传输：TCP每发送一个报文段，就会对此报文段设置一个超时重传计时器。此计时器设置的超时重传时间RTO（Retransmission Time－Out）应当略大于TCP报文段的平均往返时延RTT(Round Trip Time)，一般可取RTO＝2RTT。当超过了规定的超时重传时间还未收到对此TCP报文段的预期确认信息，则必须重新传输此TCP报文段

在参考了RFC 6298后，了解到RTO按照双倍递增的算法进行计算。而Linux对RTO的设置为：RTO的最小值设为200ms（RFC建议1秒），最大值设置为120秒（RFC强制60秒以上）

再观察上面的抓包可以看出：从15:26:49.678680开始按照200ms间隔双倍递增，依次为15:26:49.879674(+0.2s)，15:26:50.282678(+0.4s)，一直到最后15:40:17.291684(+2mins)

tcp共计重试了15次(200ms为第一次)，总共重试时间为：924.6s，也即15mins 25s左右。可以跟上面httptest以及Controller恢复访问的时间对上，这也解释了15mins的来历

**最后验证发现：通过调整TCP ARQ重试次数 或者 设置http.Client.Timeout，发现httptest均会在短时间内关闭连接，服务重新建立连接**



### 解决

通过上面的分析，可以知道是TCP的ARQ机制导致了Controller在母机宕机后15mins内一直超时重试，超时重试失败后，tcp socket关闭，应用重新创建连接

这个问题本质上不是Kubernetes的问题，而是应用在复用tcp socket(长连接)时没有考虑设置超时，导致了母机宕机后，tcp socket没有及时关闭，服务依旧使用失效连接导致异常

要解决这个问题，可以从两方面考虑：

- 应用层使用超时设置或者健康检查机制，从上层保障连接的健康状态 - 作用于该应用
- 底层调整TCP ARQ设置(/proc/sys/net/ipv4/tcp_retries2)，缩小超时重试周期，作用于整个集群

由于应用层的超时或者健康检查机制无法使用统一的方案，这里只介绍如何采用系统配置的方式规避无效连接，如下：

```sh
# 0.2+0.4+0.8+1.6+3.2+6.4+12.8+25.6+51.2+102.4 = 222.6s
$ echo 9 > /proc/sys/net/ipv4/tcp_retries2
```

另外对于推送类的服务，比如Watch，在母机宕机后，可以通过tcp keepalive机制来关闭无效连接(这也是上面测试cluster-coredns-controller时其中一个连接5分钟(30+30*9=300s)断开的原因)：

```sh
# 30 + 30*5 = 180s
$ echo 30 >  /proc/sys/net/ipv4/tcp_keepalive_time
$ echo 30 > /proc/sys/net/ipv4/tcp_keepalive_intvl
$ echo 5 > /proc/sys/net/ipv4/tcp_keepalive_probes
```

通过上述的TCP keepalive以及TCP ARQ配置，我们可以将无效连接断开时间缩短到4分钟以内，一定程度上解决了母机宕机导致的连接异常问题。不过最好的解决方案是在应用层设置超时或者健康检查机制及时关闭底层无效连接。



### Refs

- Kubernetes Controller高可用诡异的15mins超时 (https://duyanghao.github.io/kubernetes-ha-http-keep-alive-bugs/)
- duyanghao kubernetes-reading-notes (https://github.com/duyanghao/kubernetes-reading-notes)





## 绕过conntrack，使用eBPF增强 IPVS优化K8s网络性能



**Kubernetes Service**[1] 用于实现集群中业务之间的互相调用和负载均衡，目前社区的实现主要有userspace，iptables和IPVS三种模式。IPVS模式的性能最好，但依然有优化的空间。该模式利用IPVS内核模块实现DNAT，利用nf_conntrack/iptables实现SNAT。nf_conntrack是为通用目的设计的，其内部的状态和流程都比较复杂，带来很大的性能损耗。

**腾讯TKE团队**[2] 开发了新的IPVS-BPF模式，完全绕过nf_conntrack的处理逻辑，使用eBPF完成SNAT功能。对最常用的Pod访问ClusterIP场景，短连接性能提升**40%**，p99时延降低**31%**；NodePort场景提升，详情见下表和`性能测量`章节。

![image-20210108160128156](D:\学习资料\笔记\k8s\k8s图\image-20210108160128156.png)



### 一、容器网络现状

#### **iptables模式**

存在的问题：

**1．可扩展性差。**随着service数据达到数千个，其控制面和数据面的性能都会急剧下降。原因在于iptables控制面的接口设计中，每添加一条规则，需要遍历和修改所有的规则，其控制面性能是*O(n²)*。在数据面，规则是用链表组织的，其性能是*O(n)*

**2．LB调度算法仅支持随机转发**



#### **IPVS模式**

IPVS 是专门为LB设计的。它用hash table管理service，对service的增删查找都是*O(1)*的时间复杂度。不过IPVS内核模块没有SNAT功能，因此借用了iptables的SNAT功能。IPVS 针对报文做DNAT后，将连接信息保存在nf_conntrack中，iptables据此接力做SNAT。该模式是目前Kubernetes网络性能最好的选择。但是由于nf_conntrack的复杂性，带来了很大的性能损耗。



### 二、IPVS-BPF方案介绍

#### **eBPF 介绍**

**eBPF**[3]是Linux内核中软件实现的虚拟机。用户把eBPF程序编译为eBPF指令，然后通过bpf()系统调用将eBPF指令加载到内核的特定挂载点，由特定的事件来触发eBPF指令的执行。在挂载eBPF指令时内核会进行充分验证，避免eBPF代码影响内核的安全和稳定性。另外内核也会进行JIT编译，把eBPF指令翻译为本地指令，减少性能开销。

内核在网络处理路径上中预置了很多eBPF的挂载点，例如xdp, qdisc, tcp-bpf, socket等。eBPF程序可以加载到这些挂载点，并调用内核提供的特定API来修改和控制网络报文。eBPF程序可以通过map数据结构来保存和交换数据。



#### **基于eBPF的IPVS-BPF优化方案**

针对nf_conntrack带来的性能问题，腾讯TKE团队设计实现了IPVS-BPF。核心思想是绕过nf_conntrack，减少处理每个报文的指令数目，从而节约CPU，提高性能。其主要逻辑如下:

1. 在IPVS内核模块中引入开关，支持原生IPVS逻辑和IPVS-BPF逻辑的切换
2. 在IPVS-BPF模式下，将IPVS hook点从LOCALIN前移到PREROUTING，使访问service的请求绕过nf_conntrack
3. 在IPVS新建连接和删除连接的代码中，相应的增删eBPF map中的session信息
4. 在qdisc挂载eBPF的SNAT代码，根据eBPF map中的session信息执行SNAT

此外，针对icmp, fragmentation均有专门处理，详细背景和细节，会在后续的**QCon在线会议**[4]上介绍, 欢迎一起探讨。

**优化前后报文处理流程的对比**

![image-20210108162114726](D:\学习资料\笔记\k8s\k8s图\image-20210108162114726.png)

可以看到，报文处理流程得到了极大简化。



#### **为什么不直接采用全eBPF方式**

那为什么还要用IPVS模块跟eBPF相结合，而不是直接使用eBPF把Service功能都实现了呢？

我们在设计之初也仔细研究了这个问题, 主要有以下几点考虑：

- nf_conntrack对CPU指令和时延的消耗，大于IPVS模块，是转发路径的头号性能杀手。而IPVS本身是为高性能而设计的，不是性能瓶颈所在
- IPVS有接近20年的历史，广泛应用于生产环境，性能和成熟度都有保障
- IPVS内部通过timer来维护session表的老化，而eBPF不支持timer, 只能通过用户空间代码来协同维护session表
- IPVS支持丰富的调度策略，用eBPF来重写这些调度策略，代码量大不说，很多调度策略需要的循环语句，eBPF也不支持

我们的目标是实现代码量可控，能落地的优化方案。基于以上考虑，我们选择了复用IPVS模块，绕过nf_conntrack，用eBPF完成SNAT的方案。最终数据面代码量为：500+行BPF代码， 1000+行IPVS模块改动（大部分为辅助SNAT map管理的新增代码）。



### 三、性能测量

本章节通过量化分析的方法，用perf工具读取CPU性能计数器，从微观的角度解释宏观的性能数据。本文采用的压测程序是wrk和iperf。



#### **测试环境**

复现该测试需要注意两点：

1. 不同的集群和机器，即使机型都一样，也可能因为各自母机和机架的拓扑不同，造成性能数据有背景差异。为了减少这类差异带来的误差，我们对比IPVS模式和IPVS-BPF模式时，是使用同一个集群，同样一组后端Pod, 并且使用同一个LB节点。先在IPVS模式下测出IPVS性能数据，然后把LB节点切换到IPVS-BPF模式, 再测出IPVS-BPF模式的性能数据。(注：切换模式是通过后台把控制面从kube-proxy切换为kube-proxy-bpf来实现的，产品功能上并不支持这样在线切换)
2. 本测试的目标是测量LB上软件模块优化对于访问service性能的影响，不能让客户端和RS目标服务器的带宽与CPU成为瓶颈。所以被压测的LB节点采用1核机型，不运行后端Pod实例；而运行后端服务的节点采用8核机型



##### NodePort

![image-20210108162607514](D:\学习资料\笔记\k8s\k8s图\image-20210108162607514.png)

为了采集CPI等指标，这里LB节点(红色部分)采用黑石裸金属机器，但通过hotplug只打开一个核，关闭其余核。



##### ClusterIP

![image-20210108162653607](D:\学习资料\笔记\k8s\k8s图\image-20210108162653607.png)

这里LB节点(左边的Node)采用SA2 1核1G机型。



#### **测量结果**

![image-20210108162905749](D:\学习资料\笔记\k8s\k8s图\image-20210108162905749.png)



![image-20210108162929410](D:\学习资料\笔记\k8s\k8s图\image-20210108162929410.png)

IPVS-BPF模式相对IPVS模式，NodePort短连接性能提高了64%，ClusterIP短连接性能提高了40%。

NodePort优化效果更明显，是因为NodePort需要SNAT，而我们的eBPF SNAT比iptables SNAT更高效，所以性能提升更多。

![image-20210108163049842](D:\学习资料\笔记\k8s\k8s图\image-20210108163049842.png)

如上图所示，iperf带宽测试中IPVS-BPF模式相对IPVS mode性能提升了22%。

![image-20210108163135194](D:\学习资料\笔记\k8s\k8s图\image-20210108163135194.png)

上图中，wrk测试表明NodePort 短连接p99延迟降低了47%。

![image-20210108163243762](D:\学习资料\笔记\k8s\k8s图\image-20210108163243762.png)

上图中，wrk测试表明ClusterIP短连接的p99延迟降低了31%。



**指令数和CPI**

![image-20210108163700337](D:\学习资料\笔记\k8s\k8s图\image-20210108163700337.png)

上图中从Perf工具看，平均每个请求耗费的CPU指令数，IPVS-BPF模式下降了38%。这也是性能提升的最主要原因。

![image-20210108163730932](D:\学习资料\笔记\k8s\k8s图\image-20210108163730932.png)

IPVS-BPF 模式下CPI略有增加，大概16%。



#### 测试总结

| Service类型 | 短连接cps | 短连接p99延迟 | 长连接吞吐 |
| :---------: | :-------: | :-----------: | :--------: |
|  clusterIP  |   +40%    |     -31%      | 无，见下文 |
|  nodePort   |   +64%    |     -47%      |    +22%    |

如上表，IPVS-BPF模式相对原生IPVS模式，Nodeport处理短连接性能提升了64%，p99延迟降低了47%，处理长连接带宽提升了22%；ClusterIP处理短连接吞吐量提升了40%, p99延迟降低了31%。

测试ClusterIP长连接吞吐时，iperf本身消耗了99% 的CPU，使得优化效果不容易直接测量。另外我们还发现IPVS-BPF模式下CPI有增加，值得进一步研究。



### 四、其他优化，特性限制和后续工作

#### **在开发IPVS-BPF方案过程中，顺便解决或优化了一些其他问题**

- `conn_reuse_mode = 1`时新建性能低 以及no route to host问题[6]

  这个问题是当client发起大量新建TCP连接时，新的连接被转发到terminating的Pod上，导致持续丢包。此问题在IPVS conn_reuse_mode=1的情况下不会有。但是conn_reuse_mode=1时，有另外的新建连接性能急剧下降的bug, 故一般都设置成了conn_reuse_mode=0。我们在TencentOS内核中彻底修复了该问题。

- **DNS解析偶尔5s延时 [10]**

  iptables SNAT分配lport到调用插入nf_conntrack，这中间是采用乐观锁机制。这中间如果发生竞争，相同的lport和五元组同时插入nf_conntrack会导致丢包。在IPVS-BPF模式下，SNAT选择lport的过程和插入hash table的过程在同一个循环中，循环次数最大为5次，从而减少了该问题的概率。

- **externalIP优化[11]**造成clb健康检查失败问题

  详情见：https://github.com/kubernetes/kubernetes/issues/79783#issuecomment-509007864



#### **特性限制**

- Pod访问自身所在的service，IPVS-BPF模式会把请求转发给其他pod，不会把请求转发给Pod自己



#### **后续工作**

- 借鉴Cilium提出的方法，利用eBPF进一步优化clusterIP性能
- 研究IPVS-BPF模式下CPI上升的原因，探索进一步提升性能的可能性



### 五、如何在TKE启用IPVS-BPF模式

如下图，在**腾讯云TKE控制台**[12]创建集群时，`高级设置`下的`Kube-proxy代理模式`选项，选择 `ipvs-bpf`即可。

![image-20210108164518517](D:\学习资料\笔记\k8s\k8s图\image-20210108164518517.png)

目前该特性需要申请白名单。请通过**申请页**[13]提交申请。



### 参考资料

[1]Kubernetes Service: *https://kubernetes.io/docs/concepts/services-networking/service/*

[2]腾讯TKE: *https://cloud.tencent.com/product/tke* 

[3]eBPF: *https://lwn.net/Articles/740157/* 

[4]QCon在线会议: *https://qcon.infoq.cn/2020/beijing/presentation/2321*

[5]conn_reuse_mode = 1时新建性能低: *https://github.com/kubernetes/kubernetes/issues/81775*

[6]no route to host问题: *https://tencentcloudcontainerteam.github.io/2019/12/15/no-route-to-host/*

[7]ef8004f8: *https://github.com/Tencent/TencentOS-kernel/commit/ef8004f8fe7d46e758955799a41cc9d66fa1ae34*

[8]8ec35911: *https://github.com/Tencent/TencentOS-kernel/commit/8ec35911f7c5e1cb29059c18d42cf3aec4fcc673*

[9]07a6e5ff63: *https://github.com/Tencent/TencentOS-kernel/commit/07a6e5ff63a74efcca67d496f6ac8126f1f114ff*

[10]DNS解析偶尔5s延时: *https://tencentcloudcontainerteam.github.io/2018/10/26/DNS-5-seconds-delay/*

[11]externalIP优化: *https://github.com/kubernetes/kubernetes/pull/63066*

[12]腾讯云TKE控制台: *https://console.cloud.tencent.com/tke2/cluster?rid=16*

[13]申请页: *https://cloud.tencent.com/apply/p/iwezhfpubqi*



## 【Pod Terminating原因追踪系列】之 containerd 中被漏掉的 runc 错误信息



前一段时间发现有一些containerd集群出现了Pod卡在Terminating的问题，经过一系列的排查发现是containerd对底层异常处理的问题。最后虽然通过一个短小的PR修复了这个bug，但是找到bug的过程和对问题的反思还是值得和大家分享的。

本文中会借由排查bug的过程来分析kubelet删除Pod的调用链，这样不仅仅可以了解containerd的bug，还可以借此了解更多Pod删除不掉的原因。在文章的最后会对问题进行反思，来探讨OCI出现的问题。



### 一个删除不掉的Pod

可能大家都会遇到这种问题，就是集群中有那么几个Pod无论如何也删除不掉，看起来和下图一样。当然可有很多可能导致Pod卡在Terminating的原因，比如mount目录被占用、dockerd卡死了或镜像中有“i”属性的文件。因为节点上复杂的组件（docker、containerd、cri、runc）和过长的调用链，导致很难瞬间定位出现问题的位置。所以一般遇到此类问题都会通过日志、Pod的信息和容器的状态来逐步缩小排查范围。

![image-20210108165249754](D:\学习资料\笔记\k8s\k8s图\image-20210108165249754.png)

当然首先看下集群的信息，发现没有使用docker而直接用的cri和containerd。直接使用containerd照比使用docker会有更短的调用链和更强的鲁棒性，照比使用docker应该更稳定才对（比如经常出现的docker和containerd数据不一致的问题在这里就不会出现）。接下来当然是查看kubelet日志，如下（只保留了核心部分），从这条日志中可以发现貌似是kubelet调用cri接口，最终调用runc去删除容器时报错导致删除失败。

```sh
$ journalctl -u kubelet
Feb 01 11:37:27 VM_74_45_centos kubelet[687]: E0201 11:37:27.241794     687 pod_workers.go:190] Error syncing pod 18c3d965-38cc-11ea-9c1d-6e3e7be2a462 ("advertise-api-bql7q_prod(18c3d965-38cc-11ea-9c1d-6e3e7be2a462)"), skipping: error killing pod: [failed to "KillContainer" for "memcache" with KillContainerError: "rpc error: code = Unknown desc = failed to kill container \"55d04f7c957e81fcf487b0dd71a4e50fe138165303cf6e96053751fd6770172c\": unknown error after kill: runc did not terminate sucessfully: container \"55d04f7c957e81fcf487b0dd71a4e50fe138165303cf6e96053751fd6770172c\" does not exist\n: unknown"
```

接下来我们打算分析下容器当前的状态，简单介绍下，**containerd**中用container来表示容器、用task来表示容器的运行状态，创建一个容器相当于创建container，而把容器运行起来相当于创建一个task并把task状态置为**Running**。当然停掉容器就相当于把task的状态设置为**Stopped**。通过ctr命令看下containerd中container和task的状态，容器**55d04f**对应的container和task都还在、task状态是STOPPED。接下来查看containerd日志，我们节选了一小部分，发现了如下现象，第一条日志是stop容器**55d04f**时做umount失败，接下来都是kill容器**55d04f**时发现container不存在。

```sh
error="failed to handle container TaskExit event: failed to stop container: failed rootfs umount: failed to unmount target /run/containerd/io.containerd.runtime.v1.linux/k8s.io/55d04f.../rootfs: device or resource busy: unknown"
error="failed to handle container TaskExit event: failed to stop container: unknown error after kill: runc did not terminate sucessfully: container "55d04f..." does not exist"
error="failed to handle container TaskExit event: failed to stop container: unknown error after kill: runc did not terminate sucessfully: container "55d04f..." does not exist"
error="failed to handle container TaskExit event: failed to stop container: unknown error after kill: runc did not terminate sucessfully: container "55d04f..." does not exist"
```

当然得到这些信息直觉会认为排查方向是：

- 为何rootfs会被占用，只要找出来是谁在占用rootfs就可以解决问题了
- 既然umount报错，我们是否可以使用lazy umount
- 反正之后containerd还会重试，再后来的重试中是否可以正确删除容器

第一个选项直接被排除了，看起来占用rootfs的进程并不是长期存在，等发现问题登录到节点上排查时进程已经不在了。如果不是常驻进程问题就变得麻烦了，可能是某个周期执行的监控组件，也可能是用户的某个日志收集容器某次收集时间较长在rootfs上多停留了一会。

处于懒惰的本能，我们先尝试下第二个方案。刚刚我们说过容器在containerd中被定义为**container**和**task**，查看容器信息时发现task并没有被删掉，于是我们直接在containerd的代码中找到了umount容器rootfs的代码，如下（为了阅读体验，已经简化）：

```go
func (p *Init) delete(ctx context.Context) error {
    err := p.runtime.Delete(ctx, p.id, nil)
  // ...
    if err2 := mount.UnmountAll(p.Rootfs, 0); err2 != nil {
        log.G(ctx).WithError(err2).Warn("failed to cleanup rootfs mount")
        if err == nil {
            err = errors.Wrap(err2, "failed rootfs umount")
        }
    }
    return err
}
func unmount(target string, flags int) error {
    for i := 0; i < 50; i++ {
        if err := unix.Unmount(target, flags); err != nil {
            switch err {
            case unix.EBUSY:
                time.Sleep(50 * time.Millisecond)
                continue
            default:
                return err
            }
        }
        return nil
    }
    return errors.Wrapf(unix.EBUSY, "failed to unmount target %s", target)
}
```

containerd创建容器时会创建一个containerd-shim进程来管理创建出来的容器，原本containerd对容器进程的操作就转化成了containerd对shim的RPC调用；而调用runc来操作容器的工作自然就会交给shim来做，这样最大的好处就是可以方便的实现live-restore能力，也就是即使containerd重启也不会影响到容器进程。

上面代码中的 delete函数就是由containerd-shim调用的，函数中主要工作有两个：调用`runc delete`删掉容器、调用umount卸载掉容器的rootfs。containerd日志中第一次device busy导致的umount失败就是在这里产生的。当然在umount函数中还是有个短暂的重试的，看来社区还是考虑到了偶尔可能会出现rootfs被占用的情况（怀疑是容器进程还没来的急被回收，但在某些场景下，可能这个重试的时间还有点短）。

这里要注意unmount的flags是0，查看docker代码，发现docker在umount时加了MNT_DETACH。在简单地修改了shim的代码后，在节点上测试，果然添加了MNT_DETACH以后就不会出现device busy了。于是自信的向社区提了PR，结果得到的回复却是：

> What typically happens in cases like this is you there is a mount marked as private that gets copied into a new mount namespace. A new mount namespace is created for every container, for systemd services that have MountPropagation or PrivateTmp defined, and these types of things. When those namespaces are created they get a copy of the root namespace, anything that has a private mount cannot be unmounted until all the namespaces are shut down. Mounts get marked private depending on the propagation defined on their root mount or if explicitly set.... so for example if you have /var/foo mounted and /var is mounted with mount private propagation, /var/foo will inherit the private propagation.
>
> In this case `MNT_DETACH` only detaches the mount and hides very real problems. Even if you remove the mountpoint the data will not be freed until (possibly?) a reboot or all other namespaces with copies of that mount in them are shut down.

大概意思就是如果你用了MNT_DETACH，会有一些真正的问题被藏起来。（这里有待测试，我觉得社区里这个人回复的思路有问题）。

看起来我们只能排查下为什么重试时还会失败了，节点上执行删除Pod的流程还是比较长的，很难简单通过几个举例直接说明问题，所以接下来分析下kubelet从cri到OCI删除容器的流程。



### kubelet如何删除Pod中的容器

对于kubelet的分析就要从大名鼎鼎的SyncPod开始分析了，在SyncPod开始时会计算podContainerChanges，接下来整个流程都是根据podContainerChanges的情况来执行对容器的操作。我们假设change就是KillPod，而kubelet执行KillPod会先通过创建多个goroutine并发执行StopContainers，等到所有Containers都删除成功后再删除Pod的Sandbox。具体调用流程如下：

![image-20210108171039684](D:\学习资料\笔记\k8s\k8s图\image-20210108171039684.png)

图中用红色标记的StopContainer其实就是最终调用了cri接口（container runtime interface），比如以下是两个和删除容器相关的两个cri接口，Kubernetes要求每种容器运行时都要实现cri接口。docker通过docker-shim实现了cri接口；而container通过cri插件实现了cri接口，两者并没区别。比如运行时是containerd时，对cri的调用就会通过containerd-shim最终在容器上产生影响。

```go
// StopContainer stops a running container with a grace period (i.e., timeout).
// This call is idempotent, and must not return an error if the container has
// already been stopped.
// TODO: what must the runtime do after the grace period is reached?
StopContainer(context.Context, *StopContainerRequest) (*StopContainerResponse, error)
// RemoveContainer removes the container. If the container is running, the
// container must be forcibly removed.
// This call is idempotent, and must not return an error if the container has
// already been removed.
RemoveContainer(context.Context, *RemoveContainerRequest) (*RemoveContainerResponse, error)
```

当请求到了cri后，剩下的任务就都交给了containerd和containerd-cri。cri以插件的方式运行在containerd中，本质和containerd是同一个进程，因此可以通过containerd提供的client直接通过函数调用containerd提供的service。正常情况下整个调用链如下图所示。

另外，cri插件中存在一个eventloop专门处理从containerd中获取的event。比如当容器删除后，会收到TaskExit事件，这时cri会做清理工作；比如当容器oom时，会收到OOMKill事件，cri除了清理还会更新Reason。接下来我们了解下整个删除流程：

1. 当kubelet调用cri的StopContainer接口后，cri会调用containerd的`task.Kill`接口（这里的task就是containerd中用来表示容器运行状态的模块），containerd收到请求后会调用containerd-shim的kill接口，而containerd-shim会通过命令行工具runc来kill掉容器进程。runc虽然不是守护进程，但是也有部分数据会被持久化到文件系统中，执行runc kill后，不只会给容器进程发送信号，同时还会修改runc的持久化数据。另外，当容器进程被干掉后，会被父进程shim回收掉。
2. shim成功干掉容器后，会给cri发送TaskExit的事件。当cri收到事件后会调用containerd的`task.Delete`接口，这个接口会先通过shim清理runc保留的容器持久化数据和容器运行时所用的rootfs。当两者都被清理后，shim留着也没用了，这时干脆直接发信号kill掉shim，并清理掉containerd保存的task信息。这时containerd中和容器状态相关的信息就都消失了，当然containerd中的container还完好无损。
3. 哪怕代码中不存在bug，这么长的调用链也可能会遇到系统问题。eventLoop调用`task.Delete`如果返回错误会把当前的event放到一个backoff队列，等过一段时间拿出来重试。这样就保证哪怕当前对一个容器的操作失败了，过段时间还可以重试。

![image-20210108171650816](D:\学习资料\笔记\k8s\k8s图\image-20210108171650816.png)

回到之前的问题上，可能有些聪明的同学通过上面的流程图和分析之前的日志就可以猜到答案了。没猜到也没关系，现在和大家一起分析下。还记的当时containerd的日志分成两部分么，首先是执行umount报错device busy，之后反复出现`unknown error after kill: runc did not terminate sucessfully: container "55d04f..." does not exist"`。这两部分和我们上面说的“**delete task**时清理rootfs，如果失败了会隔段时间进行重试”这个表述很接近。我们再把调用的流程图画的更细点，这下应该就可以在图中找到答案了。

当容器被kill掉之后还一切正常，cri收到了容器退出的信号，调用containerd的`task.Delete()`时，可以注意到，这里多了个withKill选项（上面的流程中其实也有，只不过被省略掉了）。添加这个选项会在调用shim的Delete接口之前再次调用Kill，这样可以防止Delete了正在运行的容器，算是“悲观”的决定。

在**第一次task Delete**的流程中，一切运行的都很顺畅，runc kill掉一个已经挂掉的容器也没什么问题。直到umount容器的rootfs，发现rootfs被占用了，而且在umount的50次重试中占用rootfs的进程并没有退出，shim只好通过containerd向cri返回一个错误。cri会在之后的一段时间里重新尝试处理刚刚的这个event。

在接下来**重试 task Delete**中，会和第一次执行一样，都会在**delete**之前执行**kill**。但由于第一次**runc delete**成功的删除了runc所持久化的容器信息，重试时执行**runc kill**会报错`container does not exist`。不巧的是shim和containerd并没有特别处理这个错误信息，而是直接返回给了cri。这就导致了cri删除容器会失败，并且再也无法umount容器的rootfs了。cri中的容器无法被删掉，自然发起删除流程的syncPod也会出现问题，这样最终就导致了Pod卡在了Terminating。

![image-20210108172137777](D:\学习资料\笔记\k8s\k8s图\image-20210108172137777.png)



### 最终修复与反思

当然这里的修复也很简单，只需要在调用runc kill后添加特殊判断就可以了，具体修复的pr见https://github.com/containerd/containerd/pull/4214，目前已经合并到主干，并且回溯到1.2的版本中了。很多时候发现问题远比修复问题要复杂的多，虽然最终修复bug的代码很简单，但是整个为了发现bug，我们用了好几天时间来分析梳理整个流程。简单看下错误处理的代码，这里的error就是调用runc出现错误的返回结果。

```go
if strings.Contains(err.Error(), "os: process already finished") ||
        strings.Contains(err.Error(), "container not running") ||        
        strings.Contains(strings.ToLower(err.Error()), "no such process") ||
        err == unix.ESRCH {
        return errors.Wrapf(errdefs.ErrNotFound, "process already finished")
    } else if strings.Contains(err.Error(), "does not exist") {
    // we add code here !
        return errors.Wrapf(errdefs.ErrNotFound, "no such container")
    }
    return errors.Wrapf(err, "unknown error after kill")
}
```

显而易见这坨代码存在问题：

containerd-shim原本目的就是支持各种OCI工具，但是却把runc的错误处理信息写死在调用OCI的路径上，这样最终可能导致shim只能为runc服务，而不好适配其他的OCI。比如完善containerd测试时就会发现这坨代码对crun并不work（crun是用纯c语言实现的OCI工具）。不可能在containerd中适配每一种OCI工具，所以问题还是出现在制定OCI规范时没考虑到错误处理的情况，同样我们也和OCI社区提了issue。





## 【Pod Terminating原因追踪系列之二】exec连接未关闭导致的事件阻塞



前一阵有客户docker18.06.3集群中出现Pod卡在terminating状态的问题，经过排查发现是containerd和dockerd之间事件流阻塞，导致后续事件得不到处理造成的。

定位问题的过程极其艰难，其中不乏大量工具的使用和大量的源码阅读。本文将梳理排查此问题的过程，并总结完整的dockerd和contaienrd之间事件传递流程，一步一步找到问题产生的原因，特写本文记录分享，希望大家在有类似问题发生时，能够迅速定位、解决。

对于本文中提到的问题，在docker19中已经得到解决，但docker18无法直接升级到docker19，因此本文在结尾参考docker19给出了一种简单的解决方案。



### 删除不掉Pod

相信大家在解决现网问题时，经常会遇到Pod卡在terminating不动的情况，产生这种情况的原因有很多，比如[【Pod Terminating原因追踪系列】之 containerd 中被漏掉的 runc 错误信息](http://mp.weixin.qq.com/s?__biz=Mzg5NjA1MjkxNw==&mid=2247487163&idx=1&sn=3b1ed7ca2b9ef1907de3e3a549b069b2&chksm=c007b561f7703c77a9b769ec609ffd7eb2d492e74853c44180746c324f1b821db9f6337e90d3&scene=21#wechat_redirect)中提到的containerd没有正确处理错误信息，当然更常见的比如umount失败、dockerd卡死等等。

遇到此类问题时，通常通过kubelet或dockerd日志、容器和Pod状态、堆栈信息等手段来排查问题。本问题也不例外，首先登录到Pod所在节点，使用以下两条指令查看容器状态：

```sh
#查看容器状态，看到容器状态为up
$ docker ps | grep <container-id>
#查看task状态，显示task的状态为STOPPED
$ docker-container-ctr --namespace moby --address var/run/docker/containerd/docker-containerd.sock task ls | grep <container-id>
```

可以看到在dockerd中容器状态为up，但在containerd中task状态为STOPPED，两者信息产生了不一致，也就是说由于某种原因containerd中的状态信息没有同步到dockerd中，为了探究为什么两者状态产生了不一致，首先需要了解从dockerd到containerd的整体调用链：

![image-20210108174222581](D:\学习资料\笔记\k8s\k8s图\image-20210108174222581.png)

当启动dockerd时，会通过NewClient方法创建一个client，该client维护一条到containerd的gRPC连接，同时起一个协程processEventStream订阅（subscribe）来自containerd的task事件，当某个容器的状态发生变化产生了事件，containerd会返回事件到client的eventQ队列中，并通过ProcessEvent方法进行处理，processEventStream协程在除优雅退出以外永远不会退出（但在有些情况下还是会退出）。

当容器进程退出时，containerd会通过上述gRPC连接返回一个exit的task事件给client，client接收到来自containerd的exit事件之后由ProcessEvent调用DeleteTask接口删除对应task，至此完成了一个容器的删除。

由于containerd一直处于STOPPED状态，因此通过上面的调用链猜测会不会是task exit事件因为某种原因而阻塞掉了？产生的结果就是在containerd侧由于发送了exit事件而进入STOPPED状态，但由于没有调用DeleteTask接口，因此本task还存在。



### 模拟task exit事件

通过发送task exit事件给一个卡住的Pod，来模拟容器结束的情况：

```go
/tasks/exit {"container_id":"23bd0b1118238852e9dec069f8a89c80e212c3d039ba030cfd33eb751fdac5a7","id":"23bd0b1118238852e9dec069f8a89c80e212c3d039ba030cfd33eb751fdac5a7","pid":17415,"exit_status":127,"exited_at":"2020-07-17T12:38:01.970418Z"}
```

我们可以手动将上述事件publish到containerd中，但是需要注意的一点的是，在publish之前需要将上述内容进行一下编码（参考containerd/cmd/containerd-shim/main_unix.go Publish方法）。得到的结果如下图，可以看到事件成功的被publish，也被dockerd捕获到，但容器的状态仍然没有变化。

```sh
#将file文件中的事件发送到containerd
docker-containerd --address var/run/docker/containerd/docker-containerd.sock publish --namespace moby --topic /tasks/exit < ~/file
```

当我们查看docker堆栈日志（向dockerd进程发送SIGUSR1信号），发现有大量的Goroutine卡在append方法，每次publish新的exit事件都会增加一个append方法的堆栈信息：

通过查看append方法的源码发现，append方法负责将收到的containerd事件放入eventQ，并执行回调函数，对收到的不同类型事件进行处理：

```go
func (q *queue) append(id string, f func()) {
    q.Lock()
    defer q.Unlock()
    if q.fns == nil {
        q.fns = make(map[string]chan struct{})
    }
    done := make(chan struct{})
    fn, ok := q.fns[id]
    q.fns[id] = done
    go func() {
        if ok {
            <-fn
        }
        f()
        close(done)
        q.Lock()
        if q.fns[id] == done {
            delete(q.fns, id)
        }
        q.Unlock()
    }()
}
```

形参中的id为container的id，因此对于同一个container它的事件是串行处理的，只有前一个事件处理结束才会处理下一个事件，且没有超时机制。

因此只要eventQ中有一个事件发生了阻塞，那么在它后面所有的事件都会被阻塞住。这也就解释了为什么每次publish新的对于同一个container的exit事件，都会在堆栈中增加一条append的堆栈信息，因为它们都被之前的一个事件阻塞住了。



### 深入源码定位问题原因

为了找到阻塞的原因，我们找到阻塞的第一个exit事件append的堆栈信息再详细的看一下：

![image-20210108174836465](D:\学习资料\笔记\k8s\k8s图\image-20210108174836465.png)

通过堆栈可以发现代码卡在了docker/daemon/monitor.go文件的123行（省略了不重要的代码）：

```go
func (daemon *Daemon) ProcessEvent(id string, e libcontainerd.EventType, ei libcontainerd.EventInfo) error {
    ......
    case libcontainerd.EventExit:
        ......
        if execConfig := c.ExecCommands.Get(ei.ProcessID); execConfig != nil {
            ......
123行        execConfig.StreamConfig.Wait()
            if err := execConfig.CloseStreams(); err != nil {
                logrus.Errorf("failed to cleanup exec %s streams: %s", c.ID, err)
            }
            ......
        } else {
            ......
        }
    ......
    return nil
}
```

可以看到收到的事件为exit事件，并在第123行streamConfig在等待一个wg，这里的streamconfig为一个内存队列，负责收集来自containerd的输出返回给客户端，具体是如何处理io的在后面会细讲，这里先顺藤摸瓜查一下wg在什么时候add的：

```go
func (c *Config) CopyToPipe(iop *cio.DirectIO) {
    copyFunc := func(w io.Writer, r io.ReadCloser) {
        c.Add(1)
        go func() {
            if _, err := pools.Copy(w, r); err != nil {
                logrus.Errorf("stream copy error: %v", err)
            }
            r.Close()
            c.Done()
        }()
    }
    if iop.Stdout != nil {
        copyFunc(c.Stdout(), iop.Stdout)
    }
    if iop.Stderr != nil {
        copyFunc(c.Stderr(), iop.Stderr)
    }
    .....
}
```

CopyToPipe是用来将containerd返回的输出copy到streamconfig的方法，可以看到当来自containerd的io流不为空，则会对wg add1，并开启协程进行copy，copy结束后才会done，因此一旦阻塞在copy，则对exit事件的处理会一直等待copy结束。我们再回到docker堆栈中进行查找，发现确实有一个IO wait，并阻塞在polls.Copy函数上：

![image-20210108174959881](D:\学习资料\笔记\k8s\k8s图\image-20210108174959881.png)

至此造成dockerd和containerd状态不一致的原因已经找到了！我们来梳理一下。

首先通过查看dockerd和containerd状态，发现两者状态不一致。由于containerd处于STOPPED状态因此判断在containerd发送task exit事件时可能发生阻塞，因此我们构造了task exit事件并publish到containerd，并查看docker堆栈发现有大量阻塞在append的堆栈信息，证实了我们的猜想。

最后我们通过分析代码和堆栈信息，最终定位在ProcessEvent由于pools.Copy的阻塞，也会被阻塞，直到copy结束，而事件又是串行处理的，因此只要有一个事件处理被阻塞，那么后面所有的事件都会被阻塞，最终表现出的现象就是dockerd和containerd状态不一致。



### 找出罪魁祸首

我们已经知道了阻塞的原因，但是究竟是什么操作阻塞了事件的处理？其实很简单，此exit事件是由exec退出产生的，我们通过查看堆栈信息，发现在堆栈有为数不多的ContainerExecStart方法，说明有exec正在执行，推测是客户行为：

![image-20210108180202440](D:\学习资料\笔记\k8s\k8s图\image-20210108180202440.png)

ContainerExecStart方法中第二个参数为exec的id值，因此可以使用gdb查找对应地址内容，查看其参数中的execId和terminating Pod中的容器的exexId（docker inspect可以查看execId，每个exec操作对应一个execId）是否一致，结果发现execId相同！因此可以断定是由于exec退出，产生的exit事件阻塞了ProcessEvent的处理逻辑，通过阅读源码总结出exec的处理逻辑：

![image-20210108180305823](D:\学习资料\笔记\k8s\k8s图\image-20210108180305823.png)

那么为什么exec的exit会导致Write阻塞呢？我们需要梳理一下exec的io处理流程看看究竟Write到了哪里。下图为io流的处理过程：

![image-20210108180343267](D:\学习资料\笔记\k8s\k8s图\image-20210108180343267.png)

首先在exec开始时会将socket的输出流attach到一个内存队列，并启动了⼀个goroutine用来把内存队列中的内容输出到socket中，除了内存队列外还有一个FIFO队列，通过CopyToPipe开启协程copy到内存队列。FIFO队列用来接收containerd-shim的输出，之后由内存队列写入socket，以response的方式返回给客户端。但我们的问题还没有解决，还是不清楚为什么Write会阻塞住。不过可以通过gdb来定位到Write函数打开的fd，查看一下socket的状态：

```go
n, err := syscall.Write(fd.Sysfd, p[nn:max])
type FD struct {
    // Lock sysfd and serialize access to Read and Write methods.
    fdmu fdMutex
    // System file descriptor. Immutable until Close.
    Sysfd int
    ......
｝
```

Write为系统调用，其参数中第一位即打开的fd号，但需要注意，Sysfd并非FD结构体的第一个参数，因此需要加上偏移量16字节（fdMutex占16字节）

![image-20210108180504595](D:\学习资料\笔记\k8s\k8s图\image-20210108180504595.png)



发现该fd为一个socket连接，使用ss查看一下socket的另一端是谁：

![image-20210108180536227](D:\学习资料\笔记\k8s\k8s图\image-20210108180536227.png)

发现该fd为来自kubelet的一个socket连接，且没有被关闭，因此可以判断Write阻塞的原因正是客户端exec退出以后，该socket没有正常的关闭，使Write不断地向socket中写数据，直到写满阻塞造成的。

通过询问客户是否使用过exec，发现客户自己写了一个客户端并通过kubelet exec来访问Pod，与上述排查结果相符，因此反馈客户可以排查下客户端代码，是否正确关闭了exec的socket连接。



### 修复与反思

其实docker的这个事件处理逻辑设计并不优雅，客户端的行为不应该影响到服务端的处理，更不应该造成服务端的阻塞，因此本打算提交pr修复此问题，发现在docker19中已经修复了此问题，而docker18的集群无法直接升级到docker19，因为docker会持久化数据到硬盘上，而docker19不支持docker18的持久化数据。

虽然不能直接升级到docker19，不过我们可以参考docker19的实现，在docker19中通过添加事件处理超时的逻辑避免事件一直阻塞，在docker18中同样可以添加一个超时的逻辑！

对exit事件添加超时处理：

```go
#/docker/daemon/monitor.go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
execConfig.StreamConfig.WaitWithTimeout(ctx)
cancel()
```

```go
#/docker/container/stream/streams.go
func (c *Config) WaitWithTimeout(ctx context.Context) {
    done := make(chan struct{}, 1)
    go func() {
        c.Wait()
        close(done)
    }()
    select {
    case <-done:
    case <-ctx.Done():
        if c.dio != nil {
            c.dio.Cancel()
            c.dio.Wait()
            c.dio.Close()
        }
    }
}
```

这里添加了一个2s超时时间，超时则优雅关闭来自containerd的事件流。

至此一个棘手的Pod terminating问题已经解决，后续也将推出小版本修复此问题，虽然修复起来比较简单，但问题分析的过程却无比艰辛，希望本篇文章能够对大家今后的问题定位打开思路。



## 【Pod Terminating原因追踪系列之三】让docker事件处理罢工的cancel状态码



本篇为Pod Terminating原因追踪系列的第三篇，前两篇[【Pod Terminating原因追踪系列】之 containerd 中被漏掉的 runc 错误信息](http://mp.weixin.qq.com/s?__biz=Mzg5NjA1MjkxNw==&mid=2247487163&idx=1&sn=3b1ed7ca2b9ef1907de3e3a549b069b2&chksm=c007b561f7703c77a9b769ec609ffd7eb2d492e74853c44180746c324f1b821db9f6337e90d3&scene=21#wechat_redirect)、[【Pod Terminating原因追踪系列之二】exec连接未关闭导致的事件阻塞](http://mp.weixin.qq.com/s?__biz=Mzg5NjA1MjkxNw==&mid=2247487524&idx=1&sn=f72fbb3c7f7a8116536791342458bdd6&chksm=c007abfef77022e8576381220d308f0a7160c1b50d35516a522db128f95ad94c1cf83b695763&scene=21#wechat_redirect)，分别介绍了两种可能导致Pod Terminating的原因。在处理现网问题时，Pod Terminating属于比较常见的问题，而本系列的初衷便是记录导致Pod Terminating问题的原因，希望能够帮助大家在遇到此类问题时，开拓排查思路。

本篇将再介绍一种造成Pod Terminating的原因，即**处理事件流的方法异常退出导致的Pod Terminating，**当docker版本在19以下且containerd进程由于各种原因（比如OOM）频繁重启时，会有概率导致此问题产生**。**对于本文中提到的问题，在docker19中已经得到解决，但由于docker18无法直接升级到docker19，且dockerd19修复的改动较大，难以cherry-pick到docker18，因此本文在结尾参考docker19的实现给出了一种简单的解决方案。



### Pod Terminating

前一阵有客户反馈使用docker18版本的节点上Pod一直处在Terminating状态，客户通过查看kubelet日志怀疑是Volume卸载失败导致的。现象如下图：

![image-20210108194951428](D:\学习资料\笔记\k8s\k8s图\image-20210108194951428.png)

```sh
Jul 31 09:53:52 VM_90_48_centos kubelet: E0731 09:53:52.860699     702 plugin_watcher.go:120] error could not find plugin for deleted file /var/lib/kubelet/plugins/kubernetes.io/qcloud-cbs/mounts/disk-o3yxvywa/WTEST.TMP when handling delete event: "/var/lib/kubelet/plugins/kubernetes.io/qcloud-cbs/mounts/disk-o3yxvywa/WTEST.TMP": REMOVE
Jul 31 09:53:52 VM_90_48_centos kubelet: E0731 09:53:52.860717     702 plugin_watcher.go:115] error stat file /var/lib/kubelet/plugins/kubernetes.io/qcloud-cbs/mounts/disk-o3yxvywa/WTEST.TMP failed: stat /var/lib/kubelet/plugins/kubernetes.io/qcloud-cbs/mounts/disk-o3yxvywa/WTEST.TMP: no such file or directory when handling create event: "/var/lib/kubelet/plugins/kubernetes.io/qcloud-cbs/mounts/disk-o3yxvywa/WTEST.TMP": CREATE
```

通过查看客户Pod的部署情况，发现客户同时使用了in-tree和out-tree的方式挂载cbs，kubelet中的报错是因为在in-tree中检测到了来自out-tree的旁路信息而报错，本质上并不会造成Pod Terminating不掉的问题，看来造成Pod Terminating的原因并非这么简单。



### 分析日志及源码

在排除了cbs卸载的问题后，我们首先想到会不会还是dockerd和containerd状态不一致的问题呢？通过下面两个指令查看了一下容器和task的状态，发现容器的状态是up而task的状态为STOPPED，果然又是状态不一致导致的问题。按照前两篇的经验来看应该是来自containerd的事件在dockerd中没有得到处理或处理的过程阻塞了。

```sh
#查看容器状态，看到容器状态为up
docker ps | grep <container-id>
#查看task状态，显示task的状态为STOPPED
docker-container-ctr --namespace moby --address var/run/docker/containerd/docker-containerd.sock task ls | grep <container-id>
```

这里提供一种简单验证方法来验证是否为task事件没有得到处理造成的Pod Terminating，随便起一个容器（例如CentOS），并通过exec进入容器并退出，这时去查看docker的堆栈（发送SIGUSR1信号给dockerd），如果发现如下有一条堆栈信息：

```go
goroutine 10717529 [select, 16 minutes]:
github.com/docker/docker/daemon.(*Daemon).ContainerExecStart(0xc4202b2000, 0x25df8e0, 0xc42010e040, 0xc4347904f1, 0x40, 0x7f7ea8250648, 0xc43240a5c0, 0x7f7ea82505e0, 0xc43240a5c0, 0x0, ...)
    /go/src/github.com/docker/docker/daemon/exec.go:264 +0xcb6
github.com/docker/docker/api/server/router/container.(*containerRouter).postContainerExecStart(0xc421069b00, 0x25df960, 0xc464e089f0, 0x25dde20, 0xc446f3a1c0, 0xc42b704000, 0xc464e08960, 0x0, 0x0)
    /go/src/github.com/docker/docker/api/server/router/container/exec.go:125 +0x34b
```

之后可以使用《[【Pod Terminating原因追踪系列之二】exec连接未关闭导致的事件阻塞](http://mp.weixin.qq.com/s?__biz=Mzg5NjA1MjkxNw==&mid=2247487524&idx=1&sn=f72fbb3c7f7a8116536791342458bdd6&chksm=c007abfef77022e8576381220d308f0a7160c1b50d35516a522db128f95ad94c1cf83b695763&scene=21#wechat_redirect)》中介绍的方法，确认一下该条堆栈信息是否是刚刚创建的CentOS容器产生的，当然从堆栈的时间上来看很容易看出来，也可以通过gdb判断ContainerExecStart参数（第二个参数的地址）中的execID是否和CentOS容器的execID相等的方式来确认，通过返回结果发现exexID相等，说明虽然我们的exec退出了，但是dockerd却没有正确处理来自containerd的exit事件。

在有了之前的排查经验后，我们很快猜到会不会是处理事件流的方法processEventStream在处理exit事件的时候发生了阻塞？验证方法很简单，只需要查看堆栈有没有goroutine卡在StreamConfig.Wait()即可，通过搜索processEventStream堆栈信息发现并没有goroutine卡在Wait方法上，甚至连processEventStream这个处理事件流的方法在堆栈都中也没有找到，说明事件处理的方法已经return了！自然也就无法处理来自containerd的所有事件了。

那么造成processEventStream方法return的具体原因是什么呢？通过查看源码发现，processEventStream中只有在一种情况下会return，即当gRPC连接返回的错误能够被解析（ok为true）且返回cancel状态码的时候proceEventStream会return，否则会另起协程递归调用proceEventStream：

```go
case err = <-errC:
    if err != nil {
        errStatus, ok := status.FromError(err)
        if !ok || errStatus.Code() != codes.Canceled {
            c.logger.WithError(err).Error("failed to get event")
            go c.processEventStream(ctx)
        } else {
            c.logger.WithError(ctx.Err()).Info("stopping event stream following graceful shutdown")
        }
    }
    return
```

那么为什么gRPC连接会返回cancel状态码呢？

在查看客户docker日志时发现containerd在之前不断的被kill并重启，持续了大概11分钟左右：

```sh
#日志省略了部分内容
Jul 29 19:23:09 VM_90_48_centos dockerd[11182]: time="2020-07-29T19:23:09.037480352+08:00" level=error msg="containerd did not exit successfully" error="signal: killed" module=libcontainerd
Jul 29 19:24:06 VM_90_48_centos dockerd[11182]: time="2020-07-29T19:24:06.972243079+08:00" level=info msg="starting containerd" revision=e6b3f5632f50dbc4e9cb6288d911bf4f5e95b18e version=v1.2.4
Jul 29 19:24:52 VM_90_48_centos dockerd[11182]: time="2020-07-29T19:24:52.643738767+08:00" level=error msg="containerd did not exit successfully" error="signal: killed" module=libcontainerd
Jul 29 19:25:02 VM_90_48_centos dockerd[11182]: time="2020-07-29T19:25:02.116798771+08:00" level=info msg="starting containerd" revision=e6b3f5632f50dbc4e9cb6288d911bf4f5e95b18e version=v1.2.4
```

查看系统日志文件（/var/log/messages）看下为什么containerd会被不断地重启：

```sh
#日志省略了部分内容
Jul 29 19:23:09 VM_90_48_centos kernel: Memory cgroup out of memory: Kill process 15069 (docker-containe) score 0 or sacrifice child
Jul 29 19:23:09 VM_90_48_centos kernel: Killed process 15069 (docker-containe) total-vm:51688kB, anon-rss:10068kB, file-rss:324kB
Jul 29 19:24:52 VM_90_48_centos kernel: Memory cgroup out of memory: Kill process 12411 (docker-containe) score 0 or sacrifice child
Jul 29 19:24:52 VM_90_48_centos kernel: Killed process 5836 (docker-containe) total-vm:1971688kB, anon-rss:22376kB, file-rss:0kB
```

可以发现containerd被kill是由于OOM导致的，那么会不会是因为containerd的不断重启导致gRPC返回cancel的状态码呢。先查看一下重启containerd这部分的逻辑：

在启动dockerd时，会创建一个独立的到containerd的gRPC连接，并启动一个monitor协程基于该gRPC连接对containerd的服务做健康检查，monitor每隔500ms会对到containerd的grpc连接做健康检查并记录失败的次数，如果发现gRPC连接返回状态码为UNKNOWN或者NOT_SERVING时对失败次数加1，当失败次数大于域值（域值为3）并且containerd进程已经down掉（通过向进程发送信号进行判断），则会重启containerd进程，并执行reconnect重置dockerd和containerd之间的gRPC连接，在reconnect的逻辑中，会先close旧的gRPC连接，之后新建一条新的gRPC连接：

```go
// containerd/containerd/client.go
func (c *Client) Reconnect() error {
    ....
    // close掉旧的连接
    c.conn.Close()
    // 建立新的连接
    conn, err := c.connector()
    ....
    c.conn = conn
    return nil
}
connector := func() (*grpc.ClientConn, error) {
    ctx, cancel := context.WithTimeout(context.Background(), 60*time.Second)
    defer cancel()
    conn, err := grpc.DialContext(ctx, dialer.DialAddress(address), gopts...)
    if err != nil {
        return nil, errors.Wrapf(err, "failed to dial %q", address)
    }
    return conn, nil
}
```

由于reconnect会先close旧连接，那么会不会是close造成的gRPC返回cancel呢？可以写一个简单的demo验证一下，服务端和客户端之间通过unix socket连接，客户端订阅服务端的消息，服务端不断地publish消息给客户端，客户端每隔一段时间close一次gRPC连接，得到的结果如下：

![image-20210108201531400](D:\学习资料\笔记\k8s\k8s图\image-20210108201531400.png)

从结果中发现在unix socket下客户端close连接是有概率导致grpc返回cancel状态码的，那么具体什么情况下会产生cancel状态码呢？通过查看gRPC源码发现，当服务端在发送事件过程中，客户端close了连接则会使服务端返回cancel状态码，若此时服务端没有发送事件，则会返回图中的transport is closing错误。

至此，问题已经基本定位了，很有可能是客户端close了gRPC连接导致服务端返回了cancel状态码，使processEventStream方法return，导致来自containerd的事件流无法得到处理，最终导致dockerd和containerd的状态不一致。但由于客户的日志级别较高，我们没法从中获得问题产生时的具体时序，因此希望通过调低日志级别复现问题来定位具体在什么情况下会产生这个问题。



### 问题复现

这个问题复现起来比较简单，只需要模仿客户产生问题时的情况，不断重启containerd进程即可。在docker18.06.3-ce版本集群下创建一个Pod，我们通过下面的脚本不断kill containerd进程：

```sh
#!/bin/bash  
for i in $(seq 1 1000)
do
process=`ps -elf | grep  "docker-containerd --config /var/run/docker/containerd/containerd.toml"| grep -v "grep"|awk '{print $4}'`
if [ ! -n "$process" ]; then
  echo "containerd not running"  
else
  echo $process;
  kill -9 $process;
fi
sleep 1;
done
```

运行上面的脚本便有几率复现该问题，之后删除Pod并查看Pod状态，发现Pod会一直卡在Terminating状态。

![image-20210108202203257](D:\学习资料\笔记\k8s\k8s图\image-20210108202203257.png)

查看容器状态和task状态，发现和客户问题的现象完全一致：

![image-20210108202219456](D:\学习资料\笔记\k8s\k8s图\image-20210108202219456.png)

由于我们调低了日志级别，查看日志发现下面这样一条日志，而这条日志只有processEventStream方法return时才会打印，且打印日志后processEventStream方法立即return，因此可以确定问题的根本原因就是processEventStream收到了gRPC返回的cancel状态码导致方法return，之后的来自containerd的事件无法得到处理，最终出现dockerd和containerd状态不一致的问题。

```sh
Aug 13 15:23:16 VM_1_6_centos dockerd[25650]: time="2020-08-13T15:23:16.686289836+08:00" level=info msg="stopping event stream following graceful shutdown" error="<nil>" module=libcontainerd namespace=moby
```



### 问题定位

通过分析docker日志，可以了解到docker18具体在什么情况下会产生processEventStream return的问题，下图是会导致processEventStream return的时序图：

![image-20210108202333612](D:\学习资料\笔记\k8s\k8s图\image-20210108202333612.png)

通过该时序图能够看出问题所在，首先当containerd进程被kill后，monitor通过健康检查，发现containerd进程已经停止，便会通过cmd重新启动containerd进程，并重新连接到contaienrd，如果processEventStream在reconnect之前使用旧的gRPC连接成功，订阅到containerd的事件，那么在reconnect时会close这条旧连接，而如果恰好在这时containerd在传输事件，那么该gRPC连接就会返回一个cancel的状态码给processEventStream方法，导致processEventStream方法return。



### 修复与反思

此问题产生的根本原因在于reconnect的逻辑，在重启时无法保证reconnect一定在processEventStream的subscribe之前发生。由于processEventStream会递归调用自动重连，因此实际上并不需要reconnect，在docker19中也已经修复了这个问题，且没有reconnect，但是docker19这部分改动较大，无法cherry-pick到docker18，因此我们可以参考docker19的实现修改docker18代码，只需要将reconnect的逻辑去除即可。

另外在修复时顺便修复了processEventStream方法不断递归导致瞬间产生大量日志的问题，由于subscribe失败以后会不断地启动协程递归调用，因此会在瞬间产生大量日志，在社区也有人已经提交过PR解决这个问题。（https://github.com/moby/moby/pull/39513）

![image-20210108202511892](D:\学习资料\笔记\k8s\k8s图\image-20210108202511892.png)

解决办法也很简单，在每次递归调用之前sleep 1秒即可，该改动也已经合进了docker19的代码中。

在后续我们将推出产品化运行时版本升级修复本篇中提到的bug，用户可以在控制台看到升级提醒并方便的进行一键升级。





## 容器网络防火墙状态异常导致丢包排查记录



### 导语

K8s容器网络涉及诸多内核子系统，IPVS，Iptable，3层路由，2层转发，TCP/IP协议栈，这些复杂的内核子系统在特定场景下可能会遇到设计者最初也想不到的问题。

本文分享了iptable防火墙状态异常导致丢包的排查记录，这个排查过程非常曲折，最后使用了在现在的作者看来非常落伍的工具：systemtap，才得以排查成功。其实依作者现有的经验，此问题现在仅需一条命令即可找到原因，这条命令就是作者之前分享过文章[使用 ebpf 深入分析容器网络 dup 包问题](http://mp.weixin.qq.com/s?__biz=Mzg5NjA1MjkxNw==&mid=2247483882&idx=1&sn=82f3cd1e5c6619d91310e50bbfe94ec7&chksm=c007ba30f7703326ef2fe42c660adaa9eb4729d8aa8db4a4b79e4d84935d01dd0f9702b43d0b&scene=21#wechat_redirect)中提到的skbtracker。时隔7个月，这个工具已经非常强大，能解决日常网络中的90%的网络问题。



### 1. 问题描述

腾讯内部某业务在容器场景上遇到了一个比较诡异的网络问题，在容器内使用GIT，SVN工具从内部代码仓库拉取代码偶发性卡顿失败，而在容器所在的Node节点使用同样版本的GIT，SVN工具却没有问题。用诡异这个词，是因为这个问题的分析持续时间比较久，经历了多个同学之手，最后都没有揪出问题根源。有挑战的问题排查对于本人来说是相当有吸引力的，于是在手头没有比较紧急任务的情况下，便开始了有趣的debug。

从客户描述看，问题复现概率极大，在Pod里面拉取10次GIT仓库，必然有一次出现卡死，对于必现的问题一般都不是问题，找到复现方法就找到了解决方法。从客户及其他同事反馈，卡死的时候，GIT Server不再继续往Client端发送数据，并且没有任何重传。



#### **1.1 网络拓扑**

业务方采用的是TKE单网卡多IP容器网络方案，node自身使用主网卡eth0，绑定一个ip，另一个弹性网卡eth1绑定多个ip地址，通过路由把这些地址与容器绑定，如图1-1.

![image-20210108213832439](D:\学习资料\笔记\k8s\k8s图\image-20210108213832439.png)



#### **1.2 复现方法**

![image-20210108214027781](D:\学习资料\笔记\k8s\k8s图\image-20210108214027781.png)



#### **1.3 抓包文件分析**

在如下三个网口eth1，veth_a1，veth_b1分别循环抓包，Server端持续向Client发送大包，卡顿发生时，Server端停止往Client发送数据包，没有任何重传报文。



### 2. 排查过程

#### **分析环境差异：Node和Pod环境差异**

- Node内存比Pod多，而Node和Pod的TCP 接收缓存大小配置一致，此处差异可能导致内存分配失败。
- 数据包进出Pod比Node多了一次路由判断，多经过两个网络设备：veth_a1和veth_b1，可能是veth的某种设备特性与TCP协议产生了冲突，或veth虚拟设备有bug，或veth设备上配置了限速规则导致。



#### **分析抓包文件: 有如下特征**

- 两个方向的数据包，在eth1，veth_a1设备上都有被buffer的现象：到达设备一段时间后被集中发送到下一跳
- 在卡住的情况下，Server端和Client端都没有重传，eth1处抓到的包总比veth_a1多很多，veth_a1处抓到的包与veth_b1处抓到的包总是能保持一致

分析：TCP是可靠传输协议，如果中间因为未知原因（比如限速）发生丢包，一定会重传。因此卡死一定是发包端收到了某种控制信号，主动停止了发包行为。



#### **猜测一：wscal协商不一致，Server端收到的wscal比较小**

在TCP握手协商阶段，Server端收到Client端wscal值比实际值小。传输过程中，因为Client端接收buffer还有充裕，Client端累计一段时间没及时回复ack报文，但实际上Server端认为Client端窗口满了（Server端通过比较小的wscal推断Client端接收buffer满了），Server端会直接停止报文发送。

如果Server端使用IPVS做接入层的时候，开启synproxy的情况下，确实会导致wscal协商不一致。

带着这个猜想进行了如下验证：

- 通过修改TCP 接收buffer（ipv4.tcp_rmem）大小，控制Client wscal值
- 通过修改Pod内存配置，保证Node和Pod的在内存配置上没有差异
- 在Server端的IPVS节点抓包，确认wscal协商结果

以上猜想经过验证一一被否决。并且找到业务方同学确认，虽然使用了IPVS模块，但是并没有开启synproxy功能，wscal协商不一致的猜想不成立。



#### **猜测二：设备buffer了报文**

设备开启了TSO，GSO特性，能极大提升数据包处理效率。猜测因为容器场景下，经过了两层设备，在每层设备都开启此特性，每层设备都buffer一段，再集中发送，导致数据包乱序或不能及时送到，TCP层流控算法判断错误导致报文停止发送。

带着这个猜想进行了如下验证：

1）关闭所有设备的高级功能（TSO，GSO，GRO，tx-nocache-copy，SG）

2）关闭容器内部delay ack功能（`net.ipv4.tcp_no_delay_ack`），让Client端积极回应Server端的数据包

以上猜想也都验证失败。



#### **终极方法：使用systamp脚本揪出罪魁祸首**

验证了常规思路都行不通。但唯一肯定的是，问题一定出在CVM内部。注意到eth1抓到的包总是比veth_a1多那么几个，之前猜想是被buffer了，但是buffer了总得发出来吧，可是持续保持抓包状态，并没有抓到这部分多余的包，那这部分包一定被丢了。这就非常好办了，只要监控这部分包的丢包点，问题就清楚了。使用systemtap监控skb的释放点并打印backtrace，即可快速找到引起丢包的内核函数。Systemtap脚本如图2-1，2-2所示。

![image-20210108215014606](D:\学习资料\笔记\k8s\k8s图\image-20210108215014606.png)



![image-20210108215035360](D:\学习资料\笔记\k8s\k8s图\image-20210108215035360.png)

首先通过图2-1脚本找到丢包点的具体函数，然后找到丢包具体的地址（交叉运行stap --all-modules dropwatch.stp -g和stap dropwatch.stp -g命令，结合/proc/kallsyms里面函数的具体地址），再把丢包地址作为判断条件，精确打印丢包点的backtrace（图2-2）。

运行脚本stap --all-modules dropwatch.stp -g，开始复现问题，脚本打印如图2-3：

![image-20210108215110082](D:\学习资料\笔记\k8s\k8s图\image-20210108215110082.png)

正常不卡顿的时候是没有nf_hook_slow的，当出现卡顿的时候，nf_hook_slow出现在屏幕中，基本确定丢包点在这个函数里面。但是有很多路径能到达这个函数，需要打印backtrace确定调用关系。再次运行脚本：stap dropwatch.stp -g，确认丢包地址列表，对比/proc/kallsyms符号表ffffffff8155c8b0 T nf_hook_slow，找到最接近0xffffffff8155c8b0 的那个值0xffffffff8155c9a3就是我们要的丢包点地址（具体内核版本和运行环境有差异）。加上丢包点的backtrace，再次复现问题，屏幕出现图2-4打印。

![image-20210108215151171](D:\学习资料\笔记\k8s\k8s图\image-20210108215151171.png)



可以看出ip_forward调用nf_hook_slow最终丢包。很明显数据包被iptable上的FORWARD 链规则丢了。查看FORWARD链上的规则，确实有丢包逻辑(-j REJECT --reject-with icmp-port-unreachable)，并且丢包的时候一定会发 icmp-port-unreachable类型的控制报文。到此基本确定原因了。因为是在Node上产生的icmp回馈信息，所以在抓包的时候无法通过Client和Server的地址过滤出这种报文（源地址是Node eth0地址，目的地址是Server的地址）。同时运行systamp脚本和tcpdump工具抓取icmp-port-unreachable报文，卡顿的时候两者都有体现。

接下来分析为什么好端端的连接传输了一段数据，后续的数据被规则丢了。仔细查看iptalbe规则发现客户配置的防火墙规则是依赖状态的：-m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT。只有ESTABLISHED连接状态的数据包才会被放行，如果在数据传输过程中，连接状态发生变化，后续入方向的报文都会被丢弃，并返回端口不可达。通过conntrack工具监测连接表状态，发现出问题时，对应连接的状态先变成了FIN_WAIT,最后变成了CLOSE_WAIT（图2-5）。通过抓包确认，GIT在下载数据的时候，会开启两个TCP连接，有一个连接在过一段时间后，Server端会主动发起fin包，而Client端因为还有数据等待传输，不会立即发送fin包，此后连接状态就会很快发生如下切换：

ESTABLISHED(Server fin)->FIN_WAIT(Client ack)->CLOSE_WAIT

所以后续的包就过不了防火墙规则了（猜测GIT协议有一个控制通道，一个数据通道，数据通道依赖控制通道，控制通道状态切换与防火墙规则冲突导致控制通道异常，数据通道也跟着异常。等有时间再研究下GIT数据传输相关协议）。这说明iptables的有状态的防火墙规则没有处理好这种半关闭状态的连接，只要一方（此场景的Server端）主动CLOSE连接以后，后续的连接状态都过不了规则（-m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT）。

明白其中原理以后，对应的解决方案也比较容易想到。因为客户是多租户容器场景，只放开了Pod主动访问的部分服务地址，这些服务地址不能主动连接Pod。了解到GIT以及SVN都是内部服务，安全性可控，让用户把这些服务地址的入方向放行，这样即使状态发生切换，因为满足入向放行规则，也能过防火墙规则。



### 3. 思考

在排查问题的过程中，发现其实容器的网络环境还有非常多值得优化和改进的地方的，比如：

- TCP接受发送buffer的大小一般是在内核启动的时候根据实际物理内存计算的一个合理值，但是到了容器场景，直接继承了Node上的默认值，明显是非常不合理的。其他系统资源配额也有类似问题。
- 网卡的TSO，GSO特性原本设计是为了优化终端协议栈处理性能的，但是在容器网络场景，Node的身份到底是属于网关还是终端？属于网关的话那做这个优化有没有其他副作用（数据包被多个设备buffer然后集中发出）。从Pod的角度看，Node在一定意义上属于网关的身份，如何扮演好终端和网关双重角色是一个比较有意思的问题
- Iptables相关问题

此场景中的防火墙状态问题

规则多了以后iptables规则同步慢问题

Service 负载均衡相关问题（规则加载慢，调度算法单一，无健康检查，无会话保持，CPS低问题）

SNAT源端口冲突问题，SNAT源端口耗尽问题

- IPVS相关问题

统计timer在配置量过大时导致CPU软中断收包延时问题

net.ipv4.vs.conn_reuse_mode导致的一系列问题 （参考：https://github.com/kubernetes/kubernetes/issues/81775 ）





## 使用 ebpf 深入分析容器网络 dup 包问题



###  **背 景**

云计算浪潮中，网络成为了跨越云端必不可少的一座桥梁，它给予人们便利，同时也带来了各种奇怪的困扰。这些困扰的奇怪之处，不仅仅在于你面对它时的束手无策，还在于当你直接或者间接解决了这些困扰时却又不知道为什么就解决了。究其本质的话，无外乎是我们不能够真正地去理清楚其中的门道儿。

但是，想要非常精通网络真的不是一件容易的事情。我们知道，内核技术门槛非常高，尤其是内核中最复杂的子系统——内核网络子系统，有了容器网络的加持使得该系统涉及到了 2 层转发，3 层转发，iptable，ipvs，NameSpace 交叉变化，面对如此复杂的环境，即使是专业的网络专家，解决问题的时候也是非常头大的。再加上由于业务的发展迅速使得网络问题变得更加频繁棘手，以及技术壁垒居高不下，开发运维人员在此花费重大人力也不一定能根治问题本身，更多时候只能绕道去尝试通过各种内核参数等等去解决问题。

既然把黑盒般的内核研究透彻是一件难于上青天的事情，那么我们是否可以尝试开发出一种工具旨在让 Linux 主机网络对一般开发运维人员来说成为一个彻底的白盒？答案是肯定的，本文正是利用这样的一款工具分析一个完全黑盒的容器网络问题。



### **1 问题描述**

用户在使用 TKE 的过程中，发现同一个节点上的 Pod1 通过 Service（ ClusterIP ）访问 Pod2， Pod1 通过 UDP push 的每一条消息会在 Pod2 上出现两次。当尝试在 Pod1  eth0 <—> veth1 <—> cbr0 <—> veth2 <—> Pod2 eth0 路径上的每个网络接口上分别抓包后，发现在 Pod1 eth0，veth1 上数据包都仅有一个，但在cbr0，veth2，Pod2 eth0 都能抓到两个完全相同的包。

进一步扩展场景发现，当满足如下条件时，就会出现 dup 包：

1. Pod1 与 Pod2在同一个 Node 。

2. Pod1 通过 Service 访问 Pod2 。

3. 容器网络为桥接模式且需要桥打开混杂模式。如 TKE 网络的 cbr0 需要打开混杂模式。

并且，这些条件都很容易满足，从『 一个 K8S 的 issue 』（1）可以看出，早在 1.2.4 版本，此问题就存在。



### 2 思路

实践发现，此问题百分之百复现，且不区分内核版本。从网页搜索结果看出，已经有人发现了这个问题并寻求帮助，但是遗憾的是还没有人能够分析出根本原因并给出解决方案：

```
Duplicate UDP packets between pods
https://github.com/kubernetes/kubernetes/issues/26255

Duplication of packets seen in bridge-promiscuous mode when accessing a pod on the same node through service ip
https://github.com/kubernetes/kubernetes/issues/25793

Resolve cbr0 hairpin/promiscuous mode duplicate packet issue
https://github.com/kubernetes/kubernetes/issues/28650
```

唯一找到一个似乎可以规避该问题的『方案』（2），也同样没有分析出原因，而是强行在 2 层通过 ebtable 丢包处理进行解决。

既然从当前掌握的情况中无法得知内核黑盒到底发生了什么，也就只能深入内核进行进一步分析了。当我们去分析内核数据包处理流程细节时，可以参考如下几种思路进行：

**1.修改内核代码，在数据包经过的路径上跟踪处理细节**

- 优点：实现比较直接。
- 缺点：不灵活，需要反复修改，重启内核；需要对协议栈非常熟悉，知道数据包处理的关键路径。

**2.插入内核模块，hook 关键函数**

- 优点：不用修改、重启内核。
- 缺点：不灵活。由于很多内部函数不能直接调用，通过该方式进行分析会导致代码量大，容易犯错。

**3. 通过 Linux 内核提供的 ebpf 去 hook 关键路径函数**

- 优点：轻量，安全，易于编写及调试。
- 缺点：ebpf 不支持循环，有些功能需要通过比较 trick 的方式实现；ebpf 当前属于比较新的技术，很多特性需要高版本内核的支持才更加完善，具体可参考『BPF Features by Linux Kernel Version』（3）。

考虑到以上分析思路，笔者已实现了基于『 bcc 』（4）的容器网络分析工具 skbtracer，旨在一键排查网络问题（主机网络，容器网络）。



### 3 工具说明

skbtracer 工具基于 bcc 框架，由 skbtracer.py 和 skbtracer.c 两个文件组成，C 代码是具体的 ebpf 代码，负责抓取过滤内核事件，Python 代码负责简单参数解析以及抓取事件的结果输出。该工具当前已支持跟踪路由信息、丢包信息以及 netfilter 处理过程，其中数据流跟踪功能正在完善，后续也将会在排查问题过程中不断完善强大，争取能够让一般用户也能把内核协议栈当作白盒观测。

本文提供的 skbtracer 需要 4.15 及以上内核才能正常运行，推荐使用 ubuntu18.04 或其他更高版本。你可以参照以下步骤进行 skbtracer 工具体验：

```sh
1. 运行镜像
docker run --privileged=true -it ccr.ccs.tencentyun.com/monkeyyang/bpftool:v0.0.1


2. 运行工具
root@14bd5f0406fd:~# ./skbtracer.py -h
usage: skbtracer.py [-h] [-H IPADDR] [--proto PROTO] [--icmpid ICMPID]
                   [-c CATCH_COUNT] [-P PORT] [-p PID] [-N NETNS]
                   [--dropstack] [--callstack] [--iptable] [--route] [--keep]
                   [-T] [-t]

Trace any packet through TCP/IP stack

optional arguments:
  -h, --help            show this help message and exit
  -H IPADDR, --ipaddr IPADDR
                        ip address
  --proto PROTO         tcp|udp|icmp|any
  --icmpid ICMPID       trace imcp id
  -c CATCH_COUNT, --catch-count CATCH_COUNT
                        catch and print count
  -P PORT, --port PORT  udp or tcp port
  -p PID, --pid PID     trace this PID only
  -N NETNS, --netns NETNS
                        trace this Network Namespace only
  --dropstack           output kernel stack trace when drop packet
  --callstack           output kernel stack trace
  --iptable             output iptable path
  --route               output route path
  --keep                keep trace packet all lifetime
  -T, --time            show HH:MM:SS timestamp
  -t, --timestamp       show timestamp in seconds at us resolution

examples:
      skbtracer.py                                      # trace all packets
      skbtracer.py --proto=icmp -H 1.2.3.4 --icmpid 22  # trace icmp packet with addr=1.2.3.4 and icmpid=22
      skbtracer.py --proto=tcp  -H 1.2.3.4 -P 22        # trace tcp  packet with addr=1.2.3.4:22
      skbtracer.py --proto=udp  -H 1.2.3.4 -P 22        # trace udp  packet wich addr=1.2.3.4:22
      skbtracer.py -t -T -p 1 --debug -P 80 -H 127.0.0.1 --proto=tcp --kernel-stack --icmpid=100 -N 10000




3.  抓取10个UDP报文
root@14bd5f0406fd:~# ./skbtracer.py  --proto=udp  -c10
time       NETWORK_NS   INTERFACE    DEST_MAC     PKT_INFO                                 TRACE_INFO
[12:51:01 ][4026531993] nil          00a8177f422e U:10.0.2.96:60479->183.60.83.19:53       ffff949a8df47700.0:ip_output
[12:51:01 ][4026531993] eth0         00a8177f422e U:10.0.2.96:60479->183.60.83.19:53       ffff949a8df47700.0:ip_finish_output
[12:51:01 ][4026531993] eth0         00a8177f422e U:10.0.2.96:60479->183.60.83.19:53       ffff949a8df47700.0:__dev_queue_xmit
[12:51:01 ][4026531993] nil          09c67eff9b77 U:10.0.2.96:56790->183.60.83.19:53       ffff949a8e655000.0:ip_output
[12:51:01 ][4026531993] eth0         09c67eff9b77 U:10.0.2.96:56790->183.60.83.19:53       ffff949a8e655000.0:ip_finish_output
[12:51:01 ][4026531993] eth0         09c67eff9b77 U:10.0.2.96:56790->183.60.83.19:53       ffff949a8e655000.0:__dev_queue_xmit
[12:51:01 ][4026531993] eth0         5254006c498f U:183.60.83.19:53->10.0.2.96:56790       ffff949a2151bd00.0:napi_gro_receive
[12:51:01 ][4026531993] eth0         5254006c498f U:183.60.83.19:53->10.0.2.96:56790       ffff949a2151bd00.0:__netif_receive_skb
[12:51:01 ][4026531993] eth0         5254006c498f U:183.60.83.19:53->10.0.2.96:56790       ffff949a2151bd00.0:ip_rcv
[12:51:01 ][4026531993] eth0         5254006c498f U:183.60.83.19:53->10.0.2.96:56790       ffff949a2151bd00.0:ip_rcv_finish

输出结果说明
第一列：ebpf抓取内核事件的时间
第二列：skb当前所在namespace的inode号
第三列：skb->dev 所指设备
第四列：抓取事件发生时，数据包目的mac地址
第五列：数据包信息，由4层协议+3层地址信息+4层端口信息组成（T代表TCP，U代表UDP，I代表ICMP，其他协议直接打印协议号）
第六列：数据包的跟踪信息，由skb内存地址+skb->pkt_type+抓取函数名（如果在netfilter抓取，则由pf号+表+链+执行结果构成）

第六列，skb->pkt_type含义如下（\include\uapi\linux\if_packet.h）：
/* Packet types */
#define PACKET_HOST		0		/* To us		*/
#define PACKET_BROADCAST	1		/* To all		*/
#define PACKET_MULTICAST	2		/* To group		*/
#define PACKET_OTHERHOST	3		/* To someone else 	*/
#define PACKET_OUTGOING		4		/* Outgoing of any type */
#define PACKET_LOOPBACK		5		/* MC/BRD frame looped back */
#define PACKET_USER		6		/* To user space	*/
#define PACKET_KERNEL		7		/* To kernel space	*/
/* Unused, PACKET_FASTROUTE and PACKET_LOOPBACK are invisible to user space */
#define PACKET_FASTROUTE	6		/* Fastrouted frame	*/

第六列，pf号含义如下（\include\uapi\linux\netfilter.h）：
enum {
	NFPROTO_UNSPEC =  0,
	NFPROTO_INET   =  1,
	NFPROTO_IPV4   =  2,
	NFPROTO_ARP    =  3,
	NFPROTO_NETDEV =  5,
	NFPROTO_BRIDGE =  7,
	NFPROTO_IPV6   = 10,
	NFPROTO_DECNET = 12,
	NFPROTO_NUMPROTO,
};
```



### **4 分析过程**

#### 4.1 环境说明

```sh
操作系统：ubuntu 18.01 4.15.0-54-generic
pod信息：
角色      nodeveth         eth0 mac地址          pod ip地址     node eth0地址
client    vethdfe06191     42:0f:47:45:67:ff     172.18.0.15    10.0.2.96
server    veth19678f6d     7a:47:5e:bb:26:b0     172.18.0.12    10.0.2.96
server2   vethe9f5bdcc     52:f5:ce:8f:62:f6     172.18.0.66    10.0.2.138

node信息：
node    eth0 ip地址    eth0 mac地址        cbr0 ip地址     crb0 mac地址
node1   10.0.2.96      52:54:00:6c:49:8f   172.18.0.1      2e:74:df:86:96:4b
node2   10.0.2.138     52:54:00:b8:05:ae   172.18.0.65     6e:41:27:22:69:0e

service信息：
ubuntu@VM-2-138-ubuntu:~$ kubectl get svc
NAME             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
kubernetes       ClusterIP   172.18.252.1     <none>        443/TCP    10d
ubuntu-server    ClusterIP   172.18.253.6     <none>        6666/UDP   10d
ubuntu-server2   ClusterIP   172.18.253.158   <none>        6667/UDP   5h41m

ubuntu-server 后端只有一个 pod，对应 pod 名为 server
ubuntu-server2 后端只有一个 pod，对应 pod 名为 server2
```

接下来用 skbtracer 工具分别输出如下场景数据包的处理细节：

1. cbr0 开启混杂模式：client 通过 Service 访问相同 Node 的 Pod

2. cbr0 开启混杂模式：client 通过 Service 访问不同 Node 的 Pod

3. cbr0 开启混杂模式：client 直接访问相同 Node 的 Pod

4. cbr0 关闭混杂模式：client 通过 Service 访问相同 Node的 Pod

5. cbr0 关闭混杂模式：client 通过 Service 访问不同 Node的 Pod

6. cbr0 关闭混杂模式：client 直接访问相同 Node 的 Pod



#### 4.2 分场景输出处理细节

##### 4.2.1. cbr0 开启混杂模式: client 通过 Service 访问相同 Node 的 Pod

```sh
root@14bd5f0406fd:~# ./skbtracer.py --route --iptable -H 172.18.0.15 --proto udp
time       NETWORK_NS   INTERFACE    DEST_MAC     PKT_INFO                                 TRACE_INFO
[02:05:54 ][4026532282] nil          000000000000 U:172.18.0.15:48655->172.18.253.6:6666   ffff949a8d639a00.0:ip_output
[02:05:54 ][4026532282] eth0         000000000000 U:172.18.0.15:48655->172.18.253.6:6666   ffff949a8d639a00.0:ip_finish_output
[02:05:54 ][4026532282] eth0         000000000000 U:172.18.0.15:48655->172.18.253.6:6666   ffff949a8d639a00.0:__dev_queue_xmit
[02:05:54 ][4026531993] vethdfe06191 2e74df86964b U:172.18.0.15:48655->172.18.253.6:6666   ffff949a8d639a00.3:__netif_receive_skb
[02:05:54 ][4026531993] vethdfe06191 2e74df86964b U:172.18.0.15:48655->172.18.253.6:6666   ffff949a8d639a00.3:br_handle_frame
[02:05:54 ][4026531993] vethdfe06191 2e74df86964b U:172.18.0.15:48655->172.18.253.6:6666   ffff949a8d639a00.0:br_nf_pre_routing
[02:05:54 ][4026531993] cbr0         2e74df86964b U:172.18.0.15:48655->172.18.253.6:6666   ffff949a8d639a00.0:2.nat.PREROUTING.ACCEPT
[02:05:54 ][4026531993] cbr0         2e74df86964b U:172.18.0.15:48655->172.18.0.12:6666    ffff949a8d639a00.0:br_nf_pre_routing_finish
[02:05:54 ][4026531993] vethdfe06191 7a475ebb26b0 U:172.18.0.15:48655->172.18.0.12:6666    ffff949a8d639a00.0:br_handle_frame_finish
[02:05:54 ][4026531993] vethdfe06191 7a475ebb26b0 U:172.18.0.15:48655->172.18.0.12:6666    ffff949a8d639a00.0:deliver_clone
[02:05:54 ][4026531993] vethdfe06191 7a475ebb26b0 U:172.18.0.15:48655->172.18.0.12:6666    ffff949a8d639400.0:__br_forward
[02:05:54 ][4026531993] veth19678f6d 7a475ebb26b0 U:172.18.0.15:48655->172.18.0.12:6666    ffff949a8d639400.0:br_nf_forward_ip
[02:05:54 ][4026531993] veth19678f6d 7a475ebb26b0 U:172.18.0.15:48655->172.18.0.12:6666    ffff949a8d639400.0:2.filter.FORWARD.ACCEPT
[02:05:54 ][4026531993] veth19678f6d 7a475ebb26b0 U:172.18.0.15:48655->172.18.0.12:6666    ffff949a8d639400.0:br_nf_forward_finish
[02:05:54 ][4026531993] veth19678f6d 7a475ebb26b0 U:172.18.0.15:48655->172.18.0.12:6666    ffff949a8d639400.0:br_forward_finish
[02:05:54 ][4026531993] veth19678f6d 7a475ebb26b0 U:172.18.0.15:48655->172.18.0.12:6666    ffff949a8d639400.0:br_nf_post_routing
[02:05:54 ][4026531993] veth19678f6d 7a475ebb26b0 U:172.18.0.15:48655->172.18.0.12:6666    ffff949a8d639400.0:2.nat.POSTROUTING.ACCEPT
[02:05:54 ][4026531993] veth19678f6d 7a475ebb26b0 U:172.18.0.15:48655->172.18.0.12:6666    ffff949a8d639400.0:br_nf_dev_queue_xmit
[02:05:54 ][4026531993] veth19678f6d 7a475ebb26b0 U:172.18.0.15:48655->172.18.0.12:6666    ffff949a8d639400.0:__dev_queue_xmit
[02:05:54 ][4026531993] vethdfe06191 7a475ebb26b0 U:172.18.0.15:48655->172.18.0.12:6666    ffff949a8d639a00.0:br_pass_frame_up
[02:05:54 ][4026531993] cbr0         7a475ebb26b0 U:172.18.0.15:48655->172.18.0.12:6666    ffff949a8d639a00.0:br_netif_receive_skb
[02:05:54 ][4026531993] cbr0         7a475ebb26b0 U:172.18.0.15:48655->172.18.0.12:6666    ffff949a8d639a00.0:__netif_receive_skb
[02:05:54 ][4026531993] cbr0         7a475ebb26b0 U:172.18.0.15:48655->172.18.0.12:6666    ffff949a8d639a00.0:ip_rcv
[02:05:54 ][4026531993] cbr0         7a475ebb26b0 U:172.18.0.15:48655->172.18.0.12:6666    ffff949a8d639a00.0:ip_rcv_finish
[02:05:54 ][4026531993] cbr0         7a475ebb26b0 U:172.18.0.15:48655->172.18.0.12:6666    ffff949a8d639a00.0:2.filter.FORWARD.ACCEPT
[02:05:54 ][4026531993] cbr0         7a475ebb26b0 U:172.18.0.15:48655->172.18.0.12:6666    ffff949a8d639a00.0:ip_output
[02:05:54 ][4026531993] cbr0         7a475ebb26b0 U:172.18.0.15:48655->172.18.0.12:6666    ffff949a8d639a00.0:ip_finish_output
[02:05:54 ][4026531993] cbr0         7a475ebb26b0 U:172.18.0.15:48655->172.18.0.12:6666    ffff949a8d639a00.0:__dev_queue_xmit
[02:05:54 ][4026531993] cbr0         7a475ebb26b0 U:172.18.0.15:48655->172.18.0.12:6666    ffff949a8d639a00.0:deliver_clone
[02:05:54 ][4026531993] cbr0         7a475ebb26b0 U:172.18.0.15:48655->172.18.0.12:6666    ffff949a8d639000.0:__br_forward
[02:05:54 ][4026531993] vethdfe06191 7a475ebb26b0 U:172.18.0.15:48655->172.18.0.12:6666    ffff949a8d639000.0:br_forward_finish
[02:05:54 ][4026531993] vethdfe06191 7a475ebb26b0 U:172.18.0.15:48655->172.18.0.12:6666    ffff949a8d639000.0:br_nf_post_routing
[02:05:54 ][4026531993] vethdfe06191 7a475ebb26b0 U:172.18.0.15:48655->172.18.0.12:6666    ffff949a8d639000.0:__dev_queue_xmit
[02:05:54 ][4026531993] cbr0         7a475ebb26b0 U:172.18.0.15:48655->172.18.0.12:6666    ffff949a8d639a00.0:__br_forward
[02:05:54 ][4026531993] veth19678f6d 7a475ebb26b0 U:172.18.0.15:48655->172.18.0.12:6666    ffff949a8d639a00.0:br_forward_finish
[02:05:54 ][4026531993] veth19678f6d 7a475ebb26b0 U:172.18.0.15:48655->172.18.0.12:6666    ffff949a8d639a00.0:br_nf_post_routing
[02:05:54 ][4026531993] veth19678f6d 7a475ebb26b0 U:172.18.0.15:48655->172.18.0.12:6666    ffff949a8d639a00.0:__dev_queue_xmit
[02:05:54 ][4026532360] eth0         7a475ebb26b0 U:172.18.0.15:48655->172.18.0.12:6666    ffff949a8d639400.0:__netif_receive_skb
[02:05:54 ][4026532360] eth0         7a475ebb26b0 U:172.18.0.15:48655->172.18.0.12:6666    ffff949a8d639400.0:ip_rcv
[02:05:54 ][4026532360] eth0         7a475ebb26b0 U:172.18.0.15:48655->172.18.0.12:6666    ffff949a8d639400.0:ip_rcv_finish
[02:05:54 ][4026532282] eth0         7a475ebb26b0 U:172.18.0.15:48655->172.18.0.12:6666    ffff949a8d639000.3:__netif_receive_skb
[02:05:54 ][4026532282] eth0         7a475ebb26b0 U:172.18.0.15:48655->172.18.0.12:6666    ffff949a8d639000.3:ip_rcv
[02:05:54 ][4026532360] eth0         7a475ebb26b0 U:172.18.0.15:48655->172.18.0.12:6666    ffff949a8d639a00.0:__netif_receive_skb
[02:05:54 ][4026532360] eth0         7a475ebb26b0 U:172.18.0.15:48655->172.18.0.12:6666    ffff949a8d639a00.0:ip_rcv
[02:05:54 ][4026532360] eth0         7a475ebb26b0 U:172.18.0.15:48655->172.18.0.12:6666    ffff949a8d639a00.0:ip_rcv_finish
```



##### 4.2.2. cbr0 开启混杂模式: client 通过 Service 访问不同 Node 的 Pod

```sh
root@14bd5f0406fd:~# ./skbtracer.py --route --iptable -H 172.18.0.15 --proto udp
time       NETWORK_NS   INTERFACE    DEST_MAC     PKT_INFO                                 TRACE_INFO
[02:06:41 ][4026532282] nil          005ea63e937c U:172.18.0.15:53947->172.18.253.158:6667 ffff949a8d639400.0:ip_output
[02:06:41 ][4026532282] eth0         005ea63e937c U:172.18.0.15:53947->172.18.253.158:6667 ffff949a8d639400.0:ip_finish_output
[02:06:41 ][4026532282] eth0         005ea63e937c U:172.18.0.15:53947->172.18.253.158:6667 ffff949a8d639400.0:__dev_queue_xmit
[02:06:41 ][4026531993] vethdfe06191 2e74df86964b U:172.18.0.15:53947->172.18.253.158:6667 ffff949a8d639400.3:__netif_receive_skb
[02:06:41 ][4026531993] vethdfe06191 2e74df86964b U:172.18.0.15:53947->172.18.253.158:6667 ffff949a8d639400.3:br_handle_frame
[02:06:41 ][4026531993] vethdfe06191 2e74df86964b U:172.18.0.15:53947->172.18.253.158:6667 ffff949a8d639400.0:br_nf_pre_routing
[02:06:41 ][4026531993] cbr0         2e74df86964b U:172.18.0.15:53947->172.18.253.158:6667 ffff949a8d639400.0:2.nat.PREROUTING.ACCEPT
[02:06:41 ][4026531993] cbr0         2e74df86964b U:172.18.0.15:53947->172.18.0.66:6667    ffff949a8d639400.0:br_nf_pre_routing_finish
[02:06:41 ][4026531993] vethdfe06191 2e74df86964b U:172.18.0.15:53947->172.18.0.66:6667    ffff949a8d639400.0:br_handle_frame_finish
[02:06:41 ][4026531993] vethdfe06191 2e74df86964b U:172.18.0.15:53947->172.18.0.66:6667    ffff949a8d639400.0:br_pass_frame_up
[02:06:41 ][4026531993] cbr0         2e74df86964b U:172.18.0.15:53947->172.18.0.66:6667    ffff949a8d639400.0:br_netif_receive_skb
[02:06:41 ][4026531993] cbr0         2e74df86964b U:172.18.0.15:53947->172.18.0.66:6667    ffff949a8d639400.0:__netif_receive_skb
[02:06:41 ][4026531993] cbr0         2e74df86964b U:172.18.0.15:53947->172.18.0.66:6667    ffff949a8d639400.0:ip_rcv
[02:06:41 ][4026531993] cbr0         2e74df86964b U:172.18.0.15:53947->172.18.0.66:6667    ffff949a8d639400.0:ip_rcv_finish
[02:06:41 ][4026531993] cbr0         2e74df86964b U:172.18.0.15:53947->172.18.0.66:6667    ffff949a8d639400.0:2.filter.FORWARD.ACCEPT
[02:06:41 ][4026531993] cbr0         2e74df86964b U:172.18.0.15:53947->172.18.0.66:6667    ffff949a8d639400.0:ip_output
[02:06:41 ][4026531993] eth0         2e74df86964b U:172.18.0.15:53947->172.18.0.66:6667    ffff949a8d639400.0:2.nat.POSTROUTING.ACCEPT
[02:06:41 ][4026531993] eth0         2e74df86964b U:172.18.0.15:53947->172.18.0.66:6667    ffff949a8d639400.0:ip_finish_output
[02:06:41 ][4026531993] eth0         feee1e1bcbe0 U:172.18.0.15:53947->172.18.0.66:6667    ffff949a8d639400.0:__dev_queue_xmit
```



##### 4.2.3. cbr0 开启混杂模式: client 直接访问相同 Node 的 Pod

```sh
root@14bd5f0406fd:~# ./skbtracer.py --route --iptable -H 172.18.0.15 --proto udp
time       NETWORK_NS   INTERFACE    DEST_MAC     PKT_INFO                                 TRACE_INFO
[02:07:21 ][4026532282] nil          000000000000 U:172.18.0.15:37729->172.18.0.12:6666    ffff949a8eb64b00.0:ip_output
[02:07:21 ][4026532282] eth0         000000000000 U:172.18.0.15:37729->172.18.0.12:6666    ffff949a8eb64b00.0:ip_finish_output
[02:07:21 ][4026532282] eth0         000000000000 U:172.18.0.15:37729->172.18.0.12:6666    ffff949a8eb64b00.0:__dev_queue_xmit
[02:07:21 ][4026531993] vethdfe06191 7a475ebb26b0 U:172.18.0.15:37729->172.18.0.12:6666    ffff949a8eb64b00.3:__netif_receive_skb
[02:07:21 ][4026531993] vethdfe06191 7a475ebb26b0 U:172.18.0.15:37729->172.18.0.12:6666    ffff949a8eb64b00.3:br_handle_frame
[02:07:21 ][4026531993] vethdfe06191 7a475ebb26b0 U:172.18.0.15:37729->172.18.0.12:6666    ffff949a8eb64b00.3:br_nf_pre_routing
[02:07:21 ][4026531993] cbr0         7a475ebb26b0 U:172.18.0.15:37729->172.18.0.12:6666    ffff949a8eb64b00.0:2.nat.PREROUTING.ACCEPT
[02:07:21 ][4026531993] cbr0         7a475ebb26b0 U:172.18.0.15:37729->172.18.0.12:6666    ffff949a8eb64b00.0:br_nf_pre_routing_finish
[02:07:21 ][4026531993] vethdfe06191 7a475ebb26b0 U:172.18.0.15:37729->172.18.0.12:6666    ffff949a8eb64b00.3:br_handle_frame_finish
[02:07:21 ][4026531993] vethdfe06191 7a475ebb26b0 U:172.18.0.15:37729->172.18.0.12:6666    ffff949a8eb64b00.3:br_forward
[02:07:21 ][4026531993] vethdfe06191 7a475ebb26b0 U:172.18.0.15:37729->172.18.0.12:6666    ffff949a8eb64b00.3:deliver_clone
[02:07:21 ][4026531993] vethdfe06191 7a475ebb26b0 U:172.18.0.15:37729->172.18.0.12:6666    ffff949a8eb64900.3:__br_forward
[02:07:21 ][4026531993] veth19678f6d 7a475ebb26b0 U:172.18.0.15:37729->172.18.0.12:6666    ffff949a8eb64900.3:br_nf_forward_ip
[02:07:21 ][4026531993] veth19678f6d 7a475ebb26b0 U:172.18.0.15:37729->172.18.0.12:6666    ffff949a8eb64900.0:2.filter.FORWARD.ACCEPT
[02:07:21 ][4026531993] veth19678f6d 7a475ebb26b0 U:172.18.0.15:37729->172.18.0.12:6666    ffff949a8eb64900.0:br_nf_forward_finish
[02:07:21 ][4026531993] veth19678f6d 7a475ebb26b0 U:172.18.0.15:37729->172.18.0.12:6666    ffff949a8eb64900.3:br_forward_finish
[02:07:21 ][4026531993] veth19678f6d 7a475ebb26b0 U:172.18.0.15:37729->172.18.0.12:6666    ffff949a8eb64900.3:br_nf_post_routing
[02:07:21 ][4026531993] veth19678f6d 7a475ebb26b0 U:172.18.0.15:37729->172.18.0.12:6666    ffff949a8eb64900.0:2.nat.POSTROUTING.ACCEPT
[02:07:21 ][4026531993] veth19678f6d 7a475ebb26b0 U:172.18.0.15:37729->172.18.0.12:6666    ffff949a8eb64900.0:br_nf_dev_queue_xmit
[02:07:21 ][4026531993] veth19678f6d 7a475ebb26b0 U:172.18.0.15:37729->172.18.0.12:6666    ffff949a8eb64900.0:__dev_queue_xmit
[02:07:21 ][4026531993] vethdfe06191 7a475ebb26b0 U:172.18.0.15:37729->172.18.0.12:6666    ffff949a8eb64b00.3:br_pass_frame_up
[02:07:21 ][4026531993] cbr0         7a475ebb26b0 U:172.18.0.15:37729->172.18.0.12:6666    ffff949a8eb64b00.3:br_netif_receive_skb
[02:07:21 ][4026531993] cbr0         7a475ebb26b0 U:172.18.0.15:37729->172.18.0.12:6666    ffff949a8eb64b00.3:__netif_receive_skb
[02:07:21 ][4026531993] cbr0         7a475ebb26b0 U:172.18.0.15:37729->172.18.0.12:6666    ffff949a8eb64b00.3:ip_rcv
[02:07:21 ][4026532360] eth0         7a475ebb26b0 U:172.18.0.15:37729->172.18.0.12:6666    ffff949a8eb64900.0:__netif_receive_skb
[02:07:21 ][4026532360] eth0         7a475ebb26b0 U:172.18.0.15:37729->172.18.0.12:6666    ffff949a8eb64900.0:ip_rcv
[02:07:21 ][4026532360] eth0         7a475ebb26b0 U:172.18.0.15:37729->172.18.0.12:6666    ffff949a8eb64900.0:ip_rcv_finish
```



##### 4.2.4. cbr0 关闭混杂模式: client 通过 Service 访问相同 Node 的 Pod

```sh
root@14bd5f0406fd:~# ./skbtracer.py --route --iptable -H 172.18.0.15 --proto udp
time       NETWORK_NS   INTERFACE    DEST_MAC     PKT_INFO                                 TRACE_INFO
[02:08:32 ][4026532282] nil          3dc486eebf45 U:172.18.0.15:53281->172.18.253.6:6666   ffff949a2151b100.0:ip_output
[02:08:32 ][4026532282] eth0         3dc486eebf45 U:172.18.0.15:53281->172.18.253.6:6666   ffff949a2151b100.0:ip_finish_output
[02:08:32 ][4026532282] eth0         3dc486eebf45 U:172.18.0.15:53281->172.18.253.6:6666   ffff949a2151b100.0:__dev_queue_xmit
[02:08:32 ][4026531993] vethdfe06191 2e74df86964b U:172.18.0.15:53281->172.18.253.6:6666   ffff949a2151b100.3:__netif_receive_skb
[02:08:32 ][4026531993] vethdfe06191 2e74df86964b U:172.18.0.15:53281->172.18.253.6:6666   ffff949a2151b100.3:br_handle_frame
[02:08:32 ][4026531993] vethdfe06191 2e74df86964b U:172.18.0.15:53281->172.18.253.6:6666   ffff949a2151b100.0:br_nf_pre_routing
[02:08:32 ][4026531993] cbr0         2e74df86964b U:172.18.0.15:53281->172.18.253.6:6666   ffff949a2151b100.0:2.nat.PREROUTING.ACCEPT
[02:08:32 ][4026531993] cbr0         2e74df86964b U:172.18.0.15:53281->172.18.0.12:6666    ffff949a2151b100.0:br_nf_pre_routing_finish
[02:08:32 ][4026531993] vethdfe06191 7a475ebb26b0 U:172.18.0.15:53281->172.18.0.12:6666    ffff949a2151b100.0:br_handle_frame_finish
[02:08:32 ][4026531993] vethdfe06191 7a475ebb26b0 U:172.18.0.15:53281->172.18.0.12:6666    ffff949a2151b100.0:br_forward
[02:08:32 ][4026531993] vethdfe06191 7a475ebb26b0 U:172.18.0.15:53281->172.18.0.12:6666    ffff949a2151b100.0:__br_forward
[02:08:32 ][4026531993] veth19678f6d 7a475ebb26b0 U:172.18.0.15:53281->172.18.0.12:6666    ffff949a2151b100.0:br_nf_forward_ip
[02:08:32 ][4026531993] veth19678f6d 7a475ebb26b0 U:172.18.0.15:53281->172.18.0.12:6666    ffff949a2151b100.0:2.filter.FORWARD.ACCEPT
[02:08:32 ][4026531993] veth19678f6d 7a475ebb26b0 U:172.18.0.15:53281->172.18.0.12:6666    ffff949a2151b100.0:br_nf_forward_finish
[02:08:32 ][4026531993] veth19678f6d 7a475ebb26b0 U:172.18.0.15:53281->172.18.0.12:6666    ffff949a2151b100.0:br_forward_finish
[02:08:32 ][4026531993] veth19678f6d 7a475ebb26b0 U:172.18.0.15:53281->172.18.0.12:6666    ffff949a2151b100.0:br_nf_post_routing
[02:08:32 ][4026531993] veth19678f6d 7a475ebb26b0 U:172.18.0.15:53281->172.18.0.12:6666    ffff949a2151b100.0:2.nat.POSTROUTING.ACCEPT
[02:08:32 ][4026531993] veth19678f6d 7a475ebb26b0 U:172.18.0.15:53281->172.18.0.12:6666    ffff949a2151b100.0:br_nf_dev_queue_xmit
[02:08:32 ][4026531993] veth19678f6d 7a475ebb26b0 U:172.18.0.15:53281->172.18.0.12:6666    ffff949a2151b100.0:__dev_queue_xmit
[02:08:32 ][4026532360] eth0         7a475ebb26b0 U:172.18.0.15:53281->172.18.0.12:6666    ffff949a2151b100.0:__netif_receive_skb
[02:08:32 ][4026532360] eth0         7a475ebb26b0 U:172.18.0.15:53281->172.18.0.12:6666    ffff949a2151b100.0:ip_rcv
[02:08:32 ][4026532360] eth0         7a475ebb26b0 U:172.18.0.15:53281->172.18.0.12:6666    ffff949a2151b100.0:ip_rcv_finish
```



##### 4.2.5. cbr0 关闭混杂模式: client 通过 Service 访问不同 Node 的 Pod

```sh
root@14bd5f0406fd:~# ./skbtracer.py --route --iptable -H 172.18.0.15 --proto udp
time       NETWORK_NS   INTERFACE    DEST_MAC     PKT_INFO                                 TRACE_INFO
[02:09:08 ][4026532282] nil          000000000000 U:172.18.0.15:44436->172.18.253.158:6667 ffff949a8e655b00.0:ip_output
[02:09:08 ][4026532282] eth0         000000000000 U:172.18.0.15:44436->172.18.253.158:6667 ffff949a8e655b00.0:ip_finish_output
[02:09:08 ][4026532282] eth0         000000000000 U:172.18.0.15:44436->172.18.253.158:6667 ffff949a8e655b00.0:__dev_queue_xmit
[02:09:08 ][4026531993] vethdfe06191 2e74df86964b U:172.18.0.15:44436->172.18.253.158:6667 ffff949a8e655b00.3:__netif_receive_skb
[02:09:08 ][4026531993] vethdfe06191 2e74df86964b U:172.18.0.15:44436->172.18.253.158:6667 ffff949a8e655b00.3:br_handle_frame
[02:09:08 ][4026531993] vethdfe06191 2e74df86964b U:172.18.0.15:44436->172.18.253.158:6667 ffff949a8e655b00.0:br_nf_pre_routing
[02:09:08 ][4026531993] cbr0         2e74df86964b U:172.18.0.15:44436->172.18.253.158:6667 ffff949a8e655b00.0:2.nat.PREROUTING.ACCEPT
[02:09:08 ][4026531993] cbr0         2e74df86964b U:172.18.0.15:44436->172.18.0.66:6667    ffff949a8e655b00.0:br_nf_pre_routing_finish
[02:09:08 ][4026531993] vethdfe06191 2e74df86964b U:172.18.0.15:44436->172.18.0.66:6667    ffff949a8e655b00.0:br_handle_frame_finish
[02:09:08 ][4026531993] vethdfe06191 2e74df86964b U:172.18.0.15:44436->172.18.0.66:6667    ffff949a8e655b00.0:br_pass_frame_up
[02:09:08 ][4026531993] cbr0         2e74df86964b U:172.18.0.15:44436->172.18.0.66:6667    ffff949a8e655b00.0:br_netif_receive_skb
[02:09:08 ][4026531993] cbr0         2e74df86964b U:172.18.0.15:44436->172.18.0.66:6667    ffff949a8e655b00.0:__netif_receive_skb
[02:09:08 ][4026531993] cbr0         2e74df86964b U:172.18.0.15:44436->172.18.0.66:6667    ffff949a8e655b00.0:ip_rcv
[02:09:08 ][4026531993] cbr0         2e74df86964b U:172.18.0.15:44436->172.18.0.66:6667    ffff949a8e655b00.0:ip_rcv_finish
[02:09:08 ][4026531993] cbr0         2e74df86964b U:172.18.0.15:44436->172.18.0.66:6667    ffff949a8e655b00.0:2.filter.FORWARD.ACCEPT
[02:09:08 ][4026531993] cbr0         2e74df86964b U:172.18.0.15:44436->172.18.0.66:6667    ffff949a8e655b00.0:ip_output
[02:09:08 ][4026531993] eth0         2e74df86964b U:172.18.0.15:44436->172.18.0.66:6667    ffff949a8e655b00.0:2.nat.POSTROUTING.ACCEPT
[02:09:08 ][4026531993] eth0         2e74df86964b U:172.18.0.15:44436->172.18.0.66:6667    ffff949a8e655b00.0:ip_finish_output
[02:09:08 ][4026531993] eth0         feee1e1bcbe0 U:172.18.0.15:44436->172.18.0.66:6667    ffff949a8e655b00.0:__dev_queue_xmit
```



##### 4.2.6. cbr0 关闭混杂模式: client 直接访问相同 Node 的 Pod

```sh
root@14bd5f0406fd:~# ./skbtracer.py --route --iptable -H 172.18.0.15 --proto udp
time       NETWORK_NS   INTERFACE    DEST_MAC     PKT_INFO                                 TRACE_INFO
[02:09:36 ][4026532282] nil          000000000000 U:172.18.0.15:41067->172.18.0.12:6666    ffff949a8d570300.0:ip_output
[02:09:36 ][4026532282] eth0         000000000000 U:172.18.0.15:41067->172.18.0.12:6666    ffff949a8d570300.0:ip_finish_output
[02:09:36 ][4026532282] eth0         000000000000 U:172.18.0.15:41067->172.18.0.12:6666    ffff949a8d570300.0:__dev_queue_xmit
[02:09:36 ][4026531993] vethdfe06191 7a475ebb26b0 U:172.18.0.15:41067->172.18.0.12:6666    ffff949a8d570300.3:__netif_receive_skb
[02:09:36 ][4026531993] vethdfe06191 7a475ebb26b0 U:172.18.0.15:41067->172.18.0.12:6666    ffff949a8d570300.3:br_handle_frame
[02:09:36 ][4026531993] vethdfe06191 7a475ebb26b0 U:172.18.0.15:41067->172.18.0.12:6666    ffff949a8d570300.3:br_nf_pre_routing
[02:09:36 ][4026531993] cbr0         7a475ebb26b0 U:172.18.0.15:41067->172.18.0.12:6666    ffff949a8d570300.0:2.nat.PREROUTING.ACCEPT
[02:09:36 ][4026531993] cbr0         7a475ebb26b0 U:172.18.0.15:41067->172.18.0.12:6666    ffff949a8d570300.0:br_nf_pre_routing_finish
[02:09:36 ][4026531993] vethdfe06191 7a475ebb26b0 U:172.18.0.15:41067->172.18.0.12:6666    ffff949a8d570300.3:br_handle_frame_finish
[02:09:36 ][4026531993] vethdfe06191 7a475ebb26b0 U:172.18.0.15:41067->172.18.0.12:6666    ffff949a8d570300.3:br_forward
[02:09:36 ][4026531993] vethdfe06191 7a475ebb26b0 U:172.18.0.15:41067->172.18.0.12:6666    ffff949a8d570300.3:__br_forward
[02:09:36 ][4026531993] veth19678f6d 7a475ebb26b0 U:172.18.0.15:41067->172.18.0.12:6666    ffff949a8d570300.3:br_nf_forward_ip
[02:09:36 ][4026531993] veth19678f6d 7a475ebb26b0 U:172.18.0.15:41067->172.18.0.12:6666    ffff949a8d570300.0:2.filter.FORWARD.ACCEPT
[02:09:36 ][4026531993] veth19678f6d 7a475ebb26b0 U:172.18.0.15:41067->172.18.0.12:6666    ffff949a8d570300.0:br_nf_forward_finish
[02:09:36 ][4026531993] veth19678f6d 7a475ebb26b0 U:172.18.0.15:41067->172.18.0.12:6666    ffff949a8d570300.3:br_forward_finish
[02:09:36 ][4026531993] veth19678f6d 7a475ebb26b0 U:172.18.0.15:41067->172.18.0.12:6666    ffff949a8d570300.3:br_nf_post_routing
[02:09:36 ][4026531993] veth19678f6d 7a475ebb26b0 U:172.18.0.15:41067->172.18.0.12:6666    ffff949a8d570300.0:2.nat.POSTROUTING.ACCEPT
[02:09:36 ][4026531993] veth19678f6d 7a475ebb26b0 U:172.18.0.15:41067->172.18.0.12:6666    ffff949a8d570300.0:br_nf_dev_queue_xmit
[02:09:36 ][4026531993] veth19678f6d 7a475ebb26b0 U:172.18.0.15:41067->172.18.0.12:6666    ffff949a8d570300.0:__dev_queue_xmit
[02:09:36 ][4026532360] eth0         7a475ebb26b0 U:172.18.0.15:41067->172.18.0.12:6666    ffff949a8d570300.0:__netif_receive_skb
[02:09:36 ][4026532360] eth0         7a475ebb26b0 U:172.18.0.15:41067->172.18.0.12:6666    ffff949a8d570300.0:ip_rcv
[02:09:36 ][4026532360] eth0         7a475ebb26b0 U:172.18.0.15:41067->172.18.0.12:6666    ffff949a8d570300.0:ip_rcv_finish
```



### **5 数据处理流程结合内核代码现象分析解释**

对比各种场景的处理流程，发现 br_handle_frame_finish 函数关键处理流程在不同场景有不同的处理结果，首先分析下这个函数：

```c
/* note: already called with rcu_read_lock */
int br_handle_frame_finish(struct net *net, struct sock *sk, struct sk_buff *skb)
{
	struct net_bridge_port *p = br_port_get_rcu(skb->dev);
	enum br_pkt_type pkt_type = BR_PKT_UNICAST;
	struct net_bridge_fdb_entry *dst = NULL;
	struct net_bridge_mdb_entry *mdst;
	bool local_rcv, mcast_hit = false;
	const unsigned char *dest;
	struct net_bridge *br;
	u16 vid = 0;

	if (!p || p->state == BR_STATE_DISABLED)
		goto drop;

	if (!br_allowed_ingress(p->br, nbp_vlan_group_rcu(p), skb, &vid))
		goto out;

	nbp_switchdev_frame_mark(p, skb);

	/* insert into forwarding database after filtering to avoid spoofing */
	br = p->br;
	if (p->flags & BR_LEARNING)
		br_fdb_update(br, p, eth_hdr(skb)->h_source, vid, false);
    //关键点，如果桥设备开启混杂模式，则local_rcv设置为true
	local_rcv = !!(br->dev->flags & IFF_PROMISC);
	dest = eth_hdr(skb)->h_dest;
	if (is_multicast_ether_addr(dest)) {
		/* by definition the broadcast is also a multicast address */
		if (is_broadcast_ether_addr(dest)) {
			pkt_type = BR_PKT_BROADCAST;
			local_rcv = true;
		} else {
			pkt_type = BR_PKT_MULTICAST;
			if (br_multicast_rcv(br, p, skb, vid))
				goto drop;
		}
	}

	if (p->state == BR_STATE_LEARNING)
		goto drop;

	BR_INPUT_SKB_CB(skb)->brdev = br->dev;

	if (IS_ENABLED(CONFIG_INET) &&
	    (skb->protocol == htons(ETH_P_ARP) ||
	     skb->protocol == htons(ETH_P_RARP))) {
		br_do_proxy_suppress_arp(skb, br, vid, p);
	} else if (IS_ENABLED(CONFIG_IPV6) &&
		   skb->protocol == htons(ETH_P_IPV6) &&
		   br->neigh_suppress_enabled &&
		   pskb_may_pull(skb, sizeof(struct ipv6hdr) +
				 sizeof(struct nd_msg)) &&
		   ipv6_hdr(skb)->nexthdr == IPPROTO_ICMPV6) {
			struct nd_msg *msg, _msg;

			msg = br_is_nd_neigh_msg(skb, &_msg);
			if (msg)
				br_do_suppress_nd(skb, br, vid, p, msg);
	}

	switch (pkt_type) {//当前的处理场景pkt_type只可能为BR_PKT_UNICAST
	case BR_PKT_MULTICAST:
		mdst = br_mdb_get(br, skb, vid);
		if ((mdst || BR_INPUT_SKB_CB_MROUTERS_ONLY(skb)) &&
		    br_multicast_querier_exists(br, eth_hdr(skb))) {
			if ((mdst && mdst->host_joined) ||
			    br_multicast_is_router(br)) {
				local_rcv = true;
				br->dev->stats.multicast++;
			}
			mcast_hit = true;
		} else {
			local_rcv = true;
			br->dev->stats.multicast++;
		}
		break;
	case BR_PKT_UNICAST: //根据数据包的目的mac地址查询该发往哪个设备
		dst = br_fdb_find_rcu(br, dest, vid);
	default:
		break;
	}

	if (dst) {
		unsigned long now = jiffies;

		if (dst->is_local) //目的mac地址是桥设备（cbr0），则把数据包往3层协议栈传递
			return br_pass_frame_up(skb);

		if (now != dst->used)
			dst->used = now;
		br_forward(dst->dst, skb, local_rcv, false); //目的mac地址是桥接的其他port，则在桥上直接转发出去
	} else {
		if (!mcast_hit)
			br_flood(br, skb, pkt_type, local_rcv, false);
		else
			br_multicast_flood(mdst, skb, local_rcv, false);
	}

	if (local_rcv) //关键点：开启混杂模式的情况下，此包再次被送入3层协议栈
		return br_pass_frame_up(skb);

out:
	return 0;
drop:
	kfree_skb(skb);
	goto out;
}
```

简单捋了下流程，简化如下图：

![image-20210109115031393](D:\学习资料\笔记\k8s\k8s图\image-20210109115031393.png)



![image-20210109115051699](D:\学习资料\笔记\k8s\k8s图\image-20210109115051699.png)



#### 5.1 为何 client 通过 Service 访问相同节点 Pod 有 dup 包？

对比内核函数 br_handle_frame_finish 和 br_forward 以及 4.2.1 中工具的输出结果，可知流程如下：

在 br_handle_frame_finish 函数内已经做完  DNAT 处理，目的 mac 地址是同 Pod 的 mac 地址，数据包有两条并行路线。

路线1：br_forward() —> deliver_clone() —> br_forward() —> 直接发给了 Service 后端的 Pod。

路线2：br_pass_frame_up()—>__netif_receive_skb(cbr0)—>ip_rcv()—>ip_output()—>发给 Service 后端的 Pod 。



#### 5.2 为何 client 直接访问相同节点 Pod 没有 dup 包？

仔细对比可以发现，4.2.3 这条数据的处理与 4.2.1（ 访问 Service ）类似，数据包也有两条路径，第一条路径完全一致，第二条路径基本一致，只不过是在 ip_rcv() 函数处的 skb->pkt_type 有区别：

- 4.2.1 的 skb->pkt_type == PACKET_HOST
- 4.2.3 的 skb->pkt_type == PACKET_OTHERHOST

而 ip_rcv() 函数入口处会直接丢弃 skb->pkt_type == PACKET_OTHERHOST 的包，因此导致第二条路径就断了。

从以下代码片段可以看出，内核函数 ip_rcv() 丢弃了 skb->pkt_type == PACKET_OTHERHOST 的数据包：

```c
/*
 * 	Main IP Receive routine.
 */
int ip_rcv(struct sk_buff *skb, struct net_device *dev, struct packet_type *pt, struct net_device *orig_dev)
{
	const struct iphdr *iph;
	struct net *net;
	u32 len;

	/* When the interface is in promisc. mode, drop all the crap
	 * that it receives, do not try to analyse it.
	 */
	if (skb->pkt_type == PACKET_OTHERHOST)
		goto drop;
```

那么，为什么 4.2.1 和 4.2.3 两种情况中 skb->pkt_type 会有差异呢？原因在于，函数 br_handle_frame 会根据数据包的目的 mac 地址进行如下判断并重新给 skb->pkt_type 赋值：

- 如果目的 mac 地址是 cbr0（ Pod 的默认网关），则 skb->pkt_type = PACKET_HOST；
- 否则skb->pkt_type = PACKET_OTHERHOST。



#### 5.3 开启混杂模式与否的哪些处理差异会导致出现  dup 包？

内核中有这样一处注释，如下：

```c
static int br_pass_frame_up(struct sk_buff *skb)
{
....
	/* Bridge is just like any other port.  Make sure the
	 * packet is allowed except in promisc modue when someone
	 * may be running packet capture.
	 */
.....
```

cbr0开启混杂模式后，内核处理逻辑认为当前正在进行 cbr0 抓包，需要让数据包经过 cbr0，因此调用 br_pass_frame_up 把包 clone 一份传给 cbr0 设备（原始包是直接转发给对应 port ），cbr0 收到包后，从 2 层协议栈转 3 层协议栈走到 ip_rcv 函数（如 5.2 所述，ip_rcv 在 skb->pkt_type == PACKET_HOST 时会继续往后处理），而后 3 层协议栈继续走 IP 转发，又再次把这个 clone 的包发给了目的 Pod 。



#### 5.4 为何在 netfilter 处理处，skb dev 会发生突变？

内核有一处注释如下：

```c
/* Direct IPv6 traffic to br_nf_pre_routing_ipv6.
 * Replicate the checks that IPv4 does on packet reception.
 * Set skb->dev to the bridge device (i.e. parent of the
 * receiving device) to make netfilter happy, the REDIRECT
 * target in particular.  Save the original destination IP
 * address to be able to detect DNAT afterwards. */
static unsigned int br_nf_pre_routing(unsigned int hook, struct sk_buff *skb,
				      const struct net_device *in,
				      const struct net_device *out,
				      int (*okfn)(struct sk_buff *))
```

该注释表明，在 netfilter 处理处 skb dev 发生突变，是由于桥模式下调用 iptable 规则，会让规则认为当前的处理接口就是 cbr0 。



#### 5.5 为何有时候能抓到两份包，有时候只能抓到一份？

关于抓包的问题，本次不做具体场景分析。抓包原理是应用层开启抓包后动态往协议栈注册 tpacket_rcv 回调函数，回调函数内部调用 bpf filter 判断满足抓包条件后会 clone 一份 skb 放到 ring buffer，此回调函数会在如下两点被调用：

- dev_hard_start_xmit （抓接口发出包的点）
- __netif_receive_skb_core（抓接口收包的点）

只要数据包经过这两个函数，就会被 tcpdump 抓取到。

以上即是所有可疑点的分析解释，如果读完本文还有其他质疑点，可以提出，一起分析讨论。



### **6 解决方案**

经过以上分析发现，当前问题是开启混杂模式后，目的端收到了两份 UDP 报文（其实 TCP 也是两份，只不过是 TCP 协议栈在接收端做了处理，并认为是重传报文）。出现该问题的关键点在于，为了使 cbr0 能够抓到数据包，br_handle_frame_finish 多 clone 出了一份 skb，往上走到了 Node 节点的 ip_rcv 。

因此，要解决这个问题，只需要把这个包提前丢掉，不让其进入目的 Pod 即可。当前 KBS 社区有人提出使用『 ebtable规则丢包』（5），通过在 veth 设置 ebtable 规则，把源地址是 Pod 网段，源 mac  是cbr0 的包丢掉：

```sh
ebtables -t filter -N CNI-DEDUP
ebtables -t filter -A OUTPUT -j CNI-DEDUP
ebtables -t filter -A CNI-DEDUP -p IPv4 -s 2e:74:df:86:96:4b -o veth+ --ip-src 172.18.0.1 -j ACCEPT
ebtables -t filter -A CNI-DEDUP -p IPv4 -s 2e:74:df:86:96:4b -o veth+ --ip-src 172.18.0.1/24 -j DROP
```

经测试，该方式能够有效防止目标 Pod 收到两份报文。也许有人会疑问：为何这个 patch 很早就被合并进了 Kubernetes 的 master 主分支，现在这个问题又冒出来了呢？原因是 Kubernetes 后来引入了网络 CNI 插件机制，把原本放到 Kubenet 中实现的网络功能移植到 CNI 插件实现，而对应的插件并没有把这个特性移植过去。



###  总 结

本文借助 ebpf 工具 skbtracer 分析了容器网桥模式下出现 dup 包问题的根本原因并讨论了解决方案， 整个分析过程工具的输出内容信息量较大，但是能够简化原本复杂的分析过程。本文仅涉及到了与问题相关的部分结论，并没有覆盖所有网络处理细节，但是工具的输出结果可以多多交叉对比，能从中收获很多惊喜，你会发现：哦~原来如此。



###  引 用

此处所展示的链接，为正文中带有『 』符号的内容所对应的扩展入口，你可以通过这些入口进一步深入阅读。

1. 这个 K8S 的 issue
   https://github.com/kubernetes/kubernetes/issues/25793

2. 方案
   https://github.com/kubernetes/kubernetes/pull/28717


3. BPF Features by Linux Kernel Version
   https://github.com/iovisor/bcc/blob/master/docs/kernel-versions.md


4. bcc
   https://github.com/iovisor/bcc

5. ebtable 规则丢包
   https://github.com/containernetworking/plugins/pull/42/commits/5e17bdf609ff76a5f7b382e3e58b561e4db1736b





## k8s api-server slb 异常流量定位分析



### 1. k8s master 的流量整体架构

![image-20210109120127762](D:\学习资料\笔记\k8s\k8s图\image-20210109120127762.png)

最近 k8s 集群新增不少服务，使得集群的 Pod 数量激增，某天接受到线上 3 台 Master 前置 SLB 流量丢失的告警（SLB 为阿里 ACK Master 默认的 SLB，默认 4 层 SLB，监听端口 6443），通过 SLB 控制台查看，在 8:00 - 9:00 的时刻，整个集群的出口流量带宽已经超过 5G，而阿里的 SLB 流量带宽最大为 5Gbps，而且带宽峰值仅是参考值，而不是业务承诺峰值；同时 SLB 是集群模式，总带宽/4 就是每台承受的最大带宽，单条流打到同一台上较高就会触发秒级分布式限速。通过与相关同事咨询，4 层 SLB 没有办法通过升级规格来抗住更高的流量，而且在集群中 Pod 2万多个情况下，也不应该出现如此大的流量，因此首先一个问题就是需要快速定位在 SLB 丢包的时候，出口的数据流量到底是流向何处，流量是否符合预期。

![k8s_master_slb_traffic.png](D:\学习资料\笔记\k8s\k8s图\k8s_master_slb_traffic.png)

排查的障碍点有两个：

- 四层 SLB 没有流量分析，阿里的 SLB 走的 Full-Nat 方式，可以在上面进行短时间抓包，但是我们的高峰点不确定，抓包分析和沟通成本都比较高，而且四层 SLB 直接将连接透传到了 RS（RealServer）上，所有的状态还得在后面的 API-Server 上进行定位；
- API-Server 是 TLS/SSL 加密，没有打开全部访问审计日志，线上流量高峰时候还是偏大，抓包分析也无法看到流量内容；

首先第一步我们需要确定丢包时刻出口流量的出去，这时候想到的第一个工具是 iftop，更多的网络流量监控工具参见这里：[Linux 网络监控工具总结](https://www.do1618.com/archives/665/linux-网络监控工具总结/)。 如果你想直接看到问题排查和结论，可以直接跳转到 [3-slb-异常流量分析](https://www.ebpf.top/post/ebpf_network_traffic_monitor/#3-slb-异常流量分析)



### 2. iftop 工具

[iftop](http://www.ex-parrot.com/~pdw/iftop/) 是类似于 top 的实时流量监控工具，底层基于 libpcap 的机制实现，可以用来监控网卡的实时流量，并且可通过指定 tcpdmp 过滤条件快速过滤想要的流量展示。代码仓库地址见[这里](https://code.blinkace.com/pdw/iftop.git)。



#### 2.1 安装

在 CentOS 系列系统中可以通过 Yum 快速安装。

```sh
$ sudo yum install iftop -y
```



#### 2.2 界面总览

![iftop](D:\学习资料\笔记\k8s\k8s图\iftop.png)



界面上方是流量的标尺，阴影的地方表示当前流量对在标尺上的位置。 <==> 表示是双向流量（在iftop 运行界面中连续输入两次 t）。

界面说明：

```
TX：发送流量
RX：接收流量
TOTAL：总流量
Cumm：运行 iftop 到目前时间的总流量
peak：流量峰值
rates：分别表示过去 2s 10s 40s 的平均流量
```



#### 2.3 Iftop 命令行参数

```sh
# iftop -h
iftop: display bandwidth usage on an interface by host

Synopsis: iftop -h | [-npblNBP] [-i interface] [-f filter code]
                               [-F net/mask] [-G net6/mask6]

   -h                  display this message
   -n                  don't do hostname lookups     # 常用 
   -N                  don't convert port numbers to services # 常用
   -p                  run in promiscuous mode (show traffic between other  # 混杂模式，类似于嗅探器
                       hosts on the same network segment)
   -b                  don't display a bar graph of traffic
   -B                  display bandwidth in bytes   # 常用
   -a                  display bandwidth in packets
   -i interface        listen on named interface    #  网卡
   -f filter code      use filter code to select packets to count
                       (default: none, but only IP packets are counted) # 支持 tcpdump 风格过滤
   -F net/mask         show traffic flows in/out of IPv4 network
   -G net6/mask6       show traffic flows in/out of IPv6 network
   -l                  display and count link-local IPv6 traffic (default: off)
   -P                  show ports as well as hosts
   -m limit            sets the upper limit for the bandwidth scale  # 设置带宽的最大刻度如  iftop -m 100M
   -c config file      specifies an alternative configuration file
   -t                  use text interface without ncurses

   Sorting orders:
   -o 2s                Sort by first column (2s traffic average)
   -o 10s               Sort by second column (10s traffic average) [default]
   -o 40s               Sort by third column (40s traffic average)
   -o source            Sort by source address
   -o destination       Sort by destination address

   The following options are only available in combination with -t
   -s num              print one single text output afer num seconds, then quit
   -L num              number of lines to print
```

如果需要把特定流量导出到日志文件供后续分析可以配合 `-t` 参数使用。

```sh
$ iftop -n -t -s 5 -L 5   // -s 5 5s 后退出， -L 5 只打印最大的 5 行
interface: eth0
IP address is: 172.16.18.161
MAC address is: 00:16:3e:12:32:fd
Listening on eth0
   # Host name (port/service if enabled)            last 2s   last 10s   last 40s cumulative
--------------------------------------------------------------------------------------------
   1 172.16.18.161                            =>     2.94Kb     1.83Kb     1.83Kb     1.38KB
     172.16.134.68                            <=     37.2Kb     23.0Kb     23.0Kb     17.3KB
   2 172.16.18.161                            =>     4.90Kb     16.1Kb     16.1Kb     12.1KB
     100.100.30.25                            <=       184b       245b       245b       184B
   3 172.16.18.161                            =>     37.5Kb     12.5Kb     12.5Kb     9.39KB
     100.100.120.55                           <=     4.41Kb     1.47Kb     1.47Kb     1.10KB
   4 172.16.18.161                            =>         0b     1.70Kb     1.70Kb     1.27KB
     100.100.120.58                           <=         0b       551b       551b       413B
   5 172.16.18.161                            =>     3.95Kb     1.32Kb     1.32Kb     0.99KB
     100.100.135.129                          <=     2.39Kb       816b       816b       612B
--------------------------------------------------------------------------------------------
Total send rate:                                     51.9Kb     35.3Kb     35.3Kb
Total receive rate:                                  47.9Kb     28.3Kb     28.3Kb
Total send and receive rate:                         99.8Kb     63.6Kb     63.6Kb
--------------------------------------------------------------------------------------------
Peak rate (sent/received/total):                     51.9Kb     47.9Kb     99.8Kb
Cumulative (sent/received/total):                    26.5KB     21.2KB     47.7KB
============================================================================================
```



#### 2.4 iftop 交互命令

在 iftop 运行主界面，可以进行交互体验，交互如下：

![iftop-interactive](D:\学习资料\笔记\k8s\k8s图\iftop-interactive.png)



常用的交互键有以下几个：

```
t  流量两行显示，还是单行显示（<==> 这种模式）
N  关闭服务端口解析
p  显示或者隐藏端口
l  设置屏幕过滤
T  显示累加值
```



#### 2.5 iftop 实现原理

![iftop_arch.png](D:\学习资料\笔记\k8s\k8s图\iftop_arch.png)



### 3. SLB 异常流量分析

由于我们的 SLB 流量抖动时间不固定，因此我们需要通过定时采集的方式来进行流量记录，然后结合 SLB 的流量时间点进行分析。

```sh
// -P 显示统计中的流量端口，-o 40s 按照近 40s 排序
$ iftop -nNB -P -f "tcp port 6443" -t -s 60 -L 100 -o 40s
iftop -nNB -P  -f "tcp port 6443" -t -s  5 -L 100
interface: eth0
IP address is: 172.16.18.161
MAC address is: 00:16:3e:12:32:fd
Listening on eth0
   # Host name (port/service if enabled)            last 2s   last 10s   last 40s（排序） cumulative
--------------------------------------------------------------------------------------------
   1 172.16.18.161:48118                      =>       177B       161B       161B       968B
     172.16.134.68:6443                       <=     2.25KB     2.07KB     2.07KB     12.4KB
   2 172.16.18.161:47894                      =>         0B        44B        44B       264B
     172.16.134.68:6443                       <=         0B        48B        48B       287B
--------------------------------------------------------------------------------------------
Total send rate:                                       177B       205B       205B
Total receive rate:                                  2.25KB     2.11KB     2.11KB
Total send and receive rate:                         2.42KB     2.32KB     2.32KB
--------------------------------------------------------------------------------------------
Peak rate (sent/received/total):                       262B     2.25KB     2.42KB
Cumulative (sent/received/total):                    1.20KB     12.7KB     13.9KB
============================================================================================
```

通过 `iftop` 的定时分析流量结果，我们发现在 SLB 流量高峰出现丢包的时候，数据都是发送到 `kube-proxy` 进程，由于集群规模大概 1000 台左右，在集群 Pod 出现集中调度的时候，会出现大量的同步事件，那么与 kube-proxy 同步的信息是什么呢？通过阅读 kube-proxy 的源码，得知 kube-proxy 进程只会从 kube-apiserver 上同步 service 和 endpoint 对象。

```go
// Run runs the specified ProxyServer.  This should never exit (unless CleanupAndExit is set).
func (s *ProxyServer) Run() error {
	// ...
  
  	
     // 监听 service 第一次同步结果，待 service 和 endpoint 都同步成功后，调用 proxier.syncProxyRules
	   go serviceConfig.Run(wait.NeverStop)
  
     // 监听 Endpoints 第一次同步结果，待 service 和 endpoint 都同步成功后，调用 proxier.syncProxyRules
	   go endpointsConfig.Run(wait.NeverStop)

		 // This has to start after the calls to NewServiceConfig and NewEndpointsConfig because those
	   // functions must configure their shared informer event handlers first.
     // 启动 informer 监听相关事件从 API Server的运行，包括 service 和 endpoint 事件的变化
	   go informerFactory.Start(wait.NeverStop)

	   // Birth Cry after the birth is successful
	   s.birthCry()

	    // Just loop forever for now...
	    s.Proxier.SyncLoop()  // iptables.NewProxier().SyncLoop()
	    return nil
}
```

在我们集群中的 Service 对象基本固定，那么在高峰流量期同步的必然是 endpoint 对象。

SLB 高峰流量时间点正好是我们一个离线服务 A 从混部集群中重新调度的时间，而该服务的副本大概有 1000 多个。同时通过分析生产环境的 service 的详情发现，由于监控的需要的，我们会在集群部署的每个服务上导出了一个额外的 Service 对象协助 Prometheus 来进行 Pod 发现（也包括在集群中**部署的 DaemonSet 服务**），结构如下：

![svc-pod](D:\学习资料\笔记\k8s\k8s图\svc_pod.png)

通过上述分析我们可以得知，因为集群中的多个 service 通过不同端口关联到了同一个 Pod，那么一个 Pod 销毁或创建触发的endpoint 对象更新会被放大（endpoint 是 service 与 pod 建立关联的中介对象）。

服务 A 副本在 1000 个左右，集群规模在 1000 台左右，那么服务 A 需要跨集群重新调度的时候（即使通过了平滑调度处理）由于批量 Pod 的频繁创建和销毁， endpoint 的状态更新流量会变得非常大，从而导致 SLB 的流量超过带宽限制，导致流量丢失。

同时由于 DaemonSet 对象也通过 Service 资源导出，用于监控服务发现，在 DaemonSet 服务发布的时候，也会面临 Endpoint 对象更新状态大量同步至 kube-proxy 的情况，导致 SLB 流量陡增。

问题排查清楚以后，那么解决的方式也比较简单：

- 对于副本大的服务 A 和 DeamonSet 服务取消用于**监控的 Service 对象**，同时对于副本较大的服务如果不需要 Ingress Service 也同步取消。

通过上述动作以后，集群 Master 节点前面的 SLB 的流量降低到最高 0.5 Gbps。

> 备注：在集群规模大的，Service 对象的过多，也可能导致 ipvs 的条目增多，从而因此 ipvs 模块的 estimation_timer 遍历时间过长，占用网络处理时间，导致网络延迟，可参见：https://www.infoq.cn/article/jmcbka0xx9nqrcx6loio 中的案例 2，当然这是我们排查的另外一个问题。如果服务设置了 NodePort，可以通过设置 kube-proxy 参数中的 –nodeport-addresses 参数设定网卡子网，避免 NodePort 在本地的所有网卡上绑定（比如 127.0.0.1 或者 docker0 169.xxx.xxx.xxx)



### 4. iftop 和 tcptop 性能对比

#### 4.1 iftop

考虑到 libpcap 的获取包的性能，决定采用 [iperf](https://iperf.fr/iperf-download.php) 工具进行压测，同时搜集一下 iftop 的性能数据，类似的网络测试工具还有 [netperf](https://github.com/HewlettPackard/netperf) 。

```sh
$ sudo yum install iperf3 -y

# 启动服务端，将 iperf3 服务端绑定在 15 核
$ taskset -c 15 iperf3 -s 
```

Client 测试，为了验证资源的情况我们把 iperf3 的客户端测试绑定在核 14 上。

```sh
$ taskset -c 14 iperf3 -c 127.0.0.1 -t 120

[ ID] Interval           Transfer     Bandwidth       Retr
[  4]   0.00-120.00 sec   288 GBytes  20.6 Gbits/sec   53             sender
[  4]   0.00-120.00 sec   288 GBytes  20.6 Gbits/sec         
```

通过 Top 命令分析 CPU 占用，占用 CPU 大概 70% 左右。

```sh
$ top -p `pidof iftop`
top - 14:50:49 up 12 days, 23:40,  4 users,  load average: 1.97, 1.19, 0.73
Tasks:   1 total,   0 running,   1 sleeping,   0 stopped,   0 zombie
%Cpu(s):  1.4 us,  9.6 sy,  0.0 ni, 86.8 id,  0.1 wa,  0.0 hi,  2.1 si,  0.0 st
KiB Mem : 65806516 total, 57069916 free,  1469748 used,  7266852 buff/cache
KiB Swap:        0 total,        0 free,        0 used. 63718576 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
19721 root      20   0  349092   5292   3436 S  66.8  0.0   0:59.21 iftop
```

pidstat 分析 iftop 程序的 CPU 的消耗，发现主要在 %system ，大概 50%，%user 大概 20%。 在 16C 的系统中，占用掉一个核，占用资源不到 5%，生产环境也是可以接受。

```sh
$ pidstat -p `pidof iftop` 1
Linux 3.10.0-1062.9.1.el7.x86_64 (bje-qtt-backend-paas-05) 	12/01/2020 	_x86_64_	(16 CPU)

02:51:21 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
02:51:22 PM     0     19721    0.00    0.00    0.00    0.00    10  iftop
02:51:23 PM     0     19721    0.00    0.00    0.00    0.00    10  iftop
02:51:24 PM     0     19721    0.00    0.00    0.00    0.00    14  iftop
02:51:25 PM     0     19721   21.00   51.00    0.00   72.00    14  iftop
02:51:26 PM     0     19721   20.00   51.00    0.00   71.00     2  iftop
02:51:27 PM     0     19721   12.00   60.00    0.00   72.00    10  iftop
02:51:28 PM     0     19721   16.00   54.00    0.00   70.00    14  iftop
02:51:29 PM     0     19721   17.00   55.00    0.00   72.00     0  iftop
02:51:30 PM     0     19721   13.00   56.00    0.00   69.00     2  iftop
02:51:31 PM     0     19721   14.00   56.00    0.00   70.00     2  iftop
02:51:32 PM     0     19721   18.00   53.00    0.00   71.00     2  iftop
02:51:33 PM     0     19721   15.00   53.00    0.00   68.00     4  iftop
02:51:34 PM     0     19721   14.00   57.00    0.00   71.00     6  iftop
02:51:35 PM     0     19721   19.00   50.00    0.00   69.00     8  iftop
```



#### 4.2 BPF BCC tcptop

在 BPF 开源项目 BCC 中也有一个基于 BPF 技术的 TCP 流量统计工具，称作 tcptop，在内核中进行数据汇总，定期同步汇总数据至用户空间，避免了每个数据包从内核传递到用户空间（iftop 中为 256个头部字节）。安装基于 eBPF 版本的 tcptop，进行相关性能测试：

```sh
$ yum install bcc -y
$ /usr/share/bcc/tools/tcptop 10   # 每 10s 刷新一次
```

在 iperf 压测过程中，观测 tcptop 的 CPU 使用情况, %user %system 基本上都为 0， 考虑到把数据统计和分析的功能转移到了内核空间的 BPF 程序中，用户空间只是定期负责收集汇总的数据，从整体性能上来讲会比使用 libpcap 库（底层采用 cBPF）采集协议头数据（256字节）通过 mmap 映射内存的方式传递到用户态分析性能更加高效。

```sh
$ pidstat -p 11264  1
Linux 3.10.0-1062.9.1.el7.x86_64 	12/01/2020 	_x86_64_	(16 CPU)

04:17:33 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
04:17:34 PM     0     11264    0.00    0.00    0.00    0.00    14  tcptop
04:17:35 PM     0     11264    0.00    0.00    0.00    0.00    14  tcptop
04:17:36 PM     0     11264    0.00    0.00    0.00    0.00    14  tcptop
04:17:37 PM     0     11264    0.00    0.00    0.00    0.00    14  tcptop
04:17:38 PM     0     11264    0.00    0.00    0.00    0.00    14  tcptop
04:17:39 PM     0     11264    0.00    0.00    0.00    0.00    14  tcptop
04:17:40 PM     0     11264    0.00    0.00    0.00    0.00    14  tcptop
04:17:41 PM     0     11264    0.00    0.00    0.00    0.00    14  tcptop
04:17:42 PM     0     11264    0.00    0.00    0.00    0.00    14  tcptop
```

bcc tcptop 的场景下，top 抽样：

```sh
top - 16:11:42 up 13 days,  1:01,  4 users,  load average: 1.01, 1.44, 1.40
Tasks: 263 total,   3 running, 260 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.7 us, 10.0 sy,  0.0 ni, 87.3 id,  0.1 wa,  0.0 hi,  1.9 si,  0.0 st
KiB Mem : 65806516 total, 56985872 free,  1542504 used,  7278140 buff/cache
KiB Swap:        0 total,        0 free,        0 used. 63645360 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
31356 root      20   0    9744    844    732 R 100.0  0.0   0:19.12 iperf3
11519 root      20   0    9744   1024    876 R  96.7  0.0  24:46.13 iperf3
```

iftop 的场景下，top 抽样：

```sh
top - 16:14:43 up 13 days,  1:04,  4 users,  load average: 0.93, 1.37, 1.39
Tasks: 266 total,   3 running, 263 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.8 us, 10.5 sy,  0.0 ni, 86.6 id,  0.1 wa,  0.0 hi,  2.0 si,  0.0 st
KiB Mem : 65806516 total, 57052332 free,  1470336 used,  7283848 buff/cache
KiB Swap:        0 total,        0 free,        0 used. 63717608 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 5809 root      20   0    9744    840    728 R 100.0  0.0   0:09.75 iperf3
11519 root      20   0    9744   1024    876 R  83.7  0.0  26:31.89 iperf3
 5782 root      20   0  349092   7420   3436 S  74.1  0.0   0:07.23 iftop
```

在下一篇文章我们会对基于 BPF 技术进行 tcp 流量分析的工具 tcptop 进行原理分析。



### 5. 参考

- [Linux 网络监控工具总结](https://www.do1618.com/archives/665/linux-网络监控工具总结/)
- [netperf 和 iperf 网络性能测试小结](https://wsgzao.github.io/post/netperf/)
- [Linux bcc tcptop](http://www.brendangregg.com/blog/2016-10-15/linux-bcc-tcptop.html)
- [使用 eBPF&BCC 提取内核网络流量信息](https://blog.csdn.net/qq_34258344/article/details/107605410)
- [使用 eBPF&bcc 提取内核网络流量信息（二）](https://blog.csdn.net/qq_34258344/article/details/107762867)
- [kube-proxy 源码解析](https://github.com/DavadDi/go_study/blob/master/kube-proxy.md)





## 容器网络延时之 ipvs 定时器篇



### 1. 前言

趣头条的容器化已经开展了一年有余，累计完成了近 1000 个服务的容器化工作，微服务集群的规模也达到了千台以上的规模。随着容器化服务数量和集群规模的不断增大，除了常规的 API Server 参数优化、Scheduler 优化等常规优化外，近期我们还碰到了 kubernetes 底层负载均衡 ipvs 模块导致的网络抖动问题，在此把整个问题的分析、排查和解决的思路进行总结，希望能为有类似问题场景解决提供一种思路。

涉及到的 k8s 集群和机器操作系统版本如下：

- k8s 阿里云 ACK 1.14.8 版本，网络模型为 CNI 插件 [terway](https://github.com/AliyunContainerService/terway) 中的 terway-eniip 模式；
- 操作系统为 CentOS 7.7.1908，内核版本为 3.10.0-1062.9.1.el7.x86_64；



### 2. 网络抖动问题

在容器集群中新部署的服务 A，在测试初期发现通过服务注册发现访问下游服务 B（在同一个容器集群） 调用延时 999 线偶发抖动，测试 QPS 比较小，从业务监控上看起来比较明显，最大的延时可以达到 200 ms。

![服务延时高](D:\学习资料\笔记\k8s\k8s图\service_latency_high.png)

服务间的访问通过 gRPC 接口访问，节点发现基于 consul 的服务注册发现。通过在服务 A 容器内的抓包分析和排查，经过了以下分析和排查：

- 服务 B 部分异常注册节点，排除异常节点后抖动情况依然存在；
- HTTP 接口延时测试， 抖动情况没有改善；
- 服务 A 在 VM（ECS）上部署测试，抖动情况没有改善；

经过上述的对比测试，我们逐步把范围缩小至服务 B 所在的主机上的底层网络抖动。

经过多次 ping 包测试，我们寻找到了某台主机 A 与 主机 B 两者之间的 ping 延时抖动与服务调用延时抖动规律比较一致，由于 ping 包 的分析比 gRPC 的分析更加简单直接，因此我们将目标转移至底层网络的 ping 包测试的轨道上。

能够稳定复现的主机环境如下图，通过主机 A ping 主机 B 中的容器实例 172.23.14.144 实例存在 ping 延时抖动。

```sh
# 主机 B 中的 Pod IP 地址
$ ip route|grep 172.23.14.144
172.23.14.144 dev cali95f3fd83a87 scope link
```

![image-20210109210301724](D:\学习资料\笔记\k8s\k8s图\image-20210109210301724.png)

基于主机B 网络 eth1 和容器网络 cali-xxx 进行 ping 的对比结果如图：

![image-20210109210359476](D:\学习资料\笔记\k8s\k8s图\image-20210109210359476.png)

通过多次测试我们发现至 Node 主机 B 主机网络的 ping 未有抖动，容器网络 cali-xx 存在比较大的抖动，最高达到 133 ms。

在 ping 测试过程中分别在主机 A 和主机 B 上使用 tcpdump 抓包分析，发现在主机 B 上的 eth1 与网卡 cali95f3fd83a87 之间的延时达 133 ms。

![ping_server_pcap.png](D:\学习资料\笔记\k8s\k8s图\ping_server_pcap.png)

主机 B 上的 ping 包延时

到此为止问题已经逐步明确，在主机 B 上接收到 ping 包在转发过程中有 100 多 ms 的延时，那么是什么原因导致的 ping 数据包在主机 B 转发的延时呢？



### 2. 问题分析

在分析 ping 数据包转发延时的情况之前，我们首先简单回顾一下网络数据包在内核中工作机制和数据流转路径。



#### 网络数据包在内核中的处理流程

在内核中，网络设备驱动是通过中断的方式来接受和处理数据包。当网卡设备上有数据到达的时候，会触发一个硬件中断来通知 CPU 来处理数据，此类处理中断的程序一般称作 ISR (Interrupt Service Routines)。ISR 程序不宜处理过多逻辑，否则会让设备的中断处理无法及时响应。因此 Linux 中将中断处理函数分为上半部和下半部。上半部是只进行最简单的工作，快速处理然后释放 CPU。剩下将绝大部分的工作都放到下半部中，下半部中逻辑由内核线程选择合适时机进行处理。

Linux 2.4 以后内核版本采用的下半部实现方式是软中断，由 `ksoftirqd` 内核线程全权处理， 正常情况下每个 CPU 核上都有自己的软中断处理数队列和 `ksoftirqd` 内核线程。软中断实现只是通过给内存中设置一个对应的二进制值来标识，软中断处理的时机主要为以下 2 种：

- 硬件中断 `irq_exit`退出时；
- 被唤醒 `ksoftirqd` 内核线程进行处理软中断；

常见的软中断类型如下：

```c
enum{
    HI_SOFTIRQ=0,
    TIMER_SOFTIRQ,
    NET_TX_SOFTIRQ,  // 网络数据包发送软中断
    NET_RX_SOFTIRQ,  // 网络数据包接受软中断
  //...
};
```

优先级自上而下，HI_SOFTIRQ 的优先级最高。其中 `NET_TX_SOFTIRQ` 对应于网络数据包的发送， `NET_RX_SOFTIRQ` 对应于网络数据包接受，两者共同完成网络数据包的发送和接收。

网络相关的中断程序在网络子系统初始化的时候进行注册， `NET_RX_SOFTIRQ` 的对应函数为 `net_rx_action()` ，在 `net_rx_action()` 函数中会调用网卡设备设置的 `poll` 函数，批量收取网络数据包并调用上层注册的协议函数进行处理，如果是为 ip 协议，则会调用 `ip_rcv`，上层协议为 icmp 的话，继续调用 `icmp_rcv` 函数进行后续的处理。

![image-20210109211325388](D:\学习资料\笔记\k8s\k8s图\image-20210109211325388.png)

```c
//net/core/dev.c

static int __init net_dev_init(void){

    ......
    for_each_possible_cpu(i) {
         struct softnet_data *sd = &per_cpu(softnet_data, i);

        memset(sd, 0, sizeof(*sd));
        skb_queue_head_init(&sd->input_pkt_queue);
        skb_queue_head_init(&sd->process_queue);
        sd->completion_queue = NULL;
        INIT_LIST_HEAD(&sd->poll_list);   // 软中断的处理中的 poll 函数列表
        // ......
    }
    ......
    open_softirq(NET_TX_SOFTIRQ, net_tx_action); // 注册网络数据包发送的软中断
    open_softirq(NET_RX_SOFTIRQ, net_rx_action); // 注册网络数据包接受的软中断
}

subsys_initcall(net_dev_init);
```

网络数据的收发的延时，多数场景下都会和系统软中断处理相关，这里我们将重点分析 ping 包抖动时的软中断情况。这里我们采用基于 **BCC** 的 ***\*traceicmpsoftirq.py\**** 来协助定位 ping 包处理的内核情况。

>BCC 为 Linux 内核 BPF 技术的前端程序，主要提供 Python 语言的绑定，`traceicmpsoftirq.py` 脚本依赖于 BCC 库，需要先安装 BCC 项目，各操作系统安装参见 **INSTALL.md**。
>
>`traceicmpsoftirq.py` 脚本在 Linux 3.10 内核与 Linux 4.x 内核上的读写方式有差异，需要根据内核略有调整。

使用 `traceicmpsoftirq.py` 在主机 B 上运行，我们发现出现抖动延时时的内核运行的内核线程都为 `ksoftirqd/0`。

```sh
# 主机 A
$ ping -c 150 -i 0.01  172.23.14.144 |grep -E "[0-9]{2,}[\.0-9]+ ms"

# 主机 B
$ ./traceicmpsoftirq.py
tgid    pid     comm            icmp_seq
...
0       0       swapper/0       128
6       6       ksoftirqd/0     129
6       6       ksoftirqd/0     130
...
```

`[ksoftirqd/0]` 这个给了我们两个重要的信息：

- 从主机 A ping 主机 B 中容器 IP 的地址，每次处理包的处理都会固定落到 CPU#0 上；
- 出现延时的时候该 CPU#0 都在运行软中断处理内核线程 `ksoftirqd/0`，即在处理软中断的过程中调用的数据包处理，软中断另外一种处理时机如上所述 `irq_exit` 硬中断退出时；

如果 ping 主机 B 中的容器 IP 地址落在 CPU#0 核上，那么按照我们的测试过程， ping 主机 B 的宿主机 IP 地址没有抖动，那么处理的 CPU 一定不在 #0 号上，才能符合测试场景，我们继续使用主机 B 主机 IP 地址进行测试：

```sh
# 主机 A
$ ping -c 150 -i 0.01  172.16.133.10 |grep -E "[0-9]{2,}[\.0-9]+ ms"

# 主机 B
$ ./traceicmpsoftirq.py
tgid    pid     comm            icmp_seq
...
0       0       swapper/19      55
0       0       swapper/19      56
0       0       swapper/19      57
...
```

通过实际的测试验证，ping 主机 B 宿主机 IP 地址时候，全部都落在了 CPU#19 上。问题排查至此处，我们可以断定是 CPU#0 与 CPU#19 在软中断处理的负载上存在差异，但是此处我们有带来另外一个疑问，为什么我们的 ping 包的处理总是固定落到同一个 CPU 核上呢？通过查询资料和主机配置确认，主机上默认启用了 RPS 的技术。RPS 全称是 Receive Packet Steering，这是 Google 工程师 Tom Herbert 提交的内核补丁, 在 2.6.35 进入 Linux 内核，采用软件模拟的方式，实现了多队列网卡所提供的功能，分散了在多 CPU 系统上数据接收时的负载，把软中断分到各个 CPU 处理，而不需要硬件支持，大大提高了网络性能。简单点讲，就是在软中断的处理函数 `net_rx_action()` 中依据 RPS 的配置，使用接收到的数据包头部（比如源 IP 地址端口等信息）信息进行作为 key 进行 Hash 到对应的 CPU 核上去处理，算法具体参见 **get_rps_cpu**[5] 函数。

Linux 环境下的 RPS 配置，可以通过下面的命令检查：

```sh
$ cat /sys/class/net/*/queues/rx-*/rps_cpus
```

通过对上述情况的综合分析，我们把问题定位在 CPU#0 在内核线程中对于软中断处理的问题上。



#### CPU 软中断处理排查

问题排查到这里，我们将重点开始排查 CPU#0 上的 CPU 内核态的性能指标，看看是否有运行的函数导致了软中断处理的延期。

首先我们使用 `perf` 命令对于 CPU#0 进行内核态使用情况进行分析。

```sh
$ perf top -C 0 -U
```

![image-20210109212357025](D:\学习资料\笔记\k8s\k8s图\image-20210109212357025.png)

通过 `perf top` 命令我们注意到 CPU#0 的内核态中，`estimation_timer` 这个函数的使用率一直占用比较高，同样我们通过对于 CPU#0 上的火焰图分析，也基本与 `perf top` 的结果一致。

![image-20210109212457323](D:\学习资料\笔记\k8s\k8s图\image-20210109212457323.png)

为了弄清楚 `estimation_timer` 的内核占用情况，我们继续使用 开源项目 **perf-tools**[6]（作者为 Brendan Gregg）中的 **funcgraph**[7] 工具分析函数 `estimation_timer` 在内核中的调用关系图和占用延时。

```sh
# -m 1最大堆栈为 1 层，-a 显示全部信息  -d 6 跟踪 6秒
$./funcgraph -m 1 -a -d 6 estimation_timer
```

![image-20210109212719066](D:\学习资料\笔记\k8s\k8s图\image-20210109212719066.png)

同时我们注意到 `estimation_timer` 函数在 CPU#0 内核中的遍历一次遍历时间为 119 ms，在内核处理软中断的情况中占用过长的时间，这一定会影响到其他软中断的处理。

为了进一步确认 CPU#0 上的软中断处理情况，我们基于 BCC 项目中的 **softirqs.py**[8] 脚本（本地略有修改），观察 CPU#0 上的软中断数量变化和整体耗时分布，发现 CPU#0 上的软中断数量增长并不是太快，但是 timer 的直方图却有异常点数据， 通过 timer 在持续 10s 内的 timer 数据分析，我们发现执行的时长分布在 [65 - 130] ms 区间的记录有 5 条。这个结论完全与通过 `funcgraph` 工具抓取到的 `estimation_timer` 在 CPU#0 上的延时一致。。

```sh
# -d 采用直方图  10 表示 10s 做一次聚合， 1 显示一次  -C 0 为我们自己修改的功能，用于过滤 CPU#0
$ /usr/share/bcc/tools/softirqs -d  10 1 -C 0
```

![image-20210109212929936](D:\学习资料\笔记\k8s\k8s图\image-20210109212929936.png)

通过上述分析我们得知 `estimation_timer` 来自于 ipvs 模块（参见图 3-4），kubernets 中 kube-proxy 组件负载均衡器正是基于 ipvs 模块，那么问题基本上出现在 kube-proxy 进程上。

我们在主机 B 上仅保留测试的容器实例，在停止 kubelet 服务后，手工停止 kube-proxy 容器进程，经过重新测试，ping 延时抖动的问题果然消失了。

到此问题的根源我们可以确定是 kube-proxy 中使用的 ipvs 内核模块中的 `estimation_timer` 函数执行时间过长，导致网络软中断处理延迟，从而使 ping 包的出现抖动，那么 `estimation_timer[ipvs]` 的作用是什么？什么情况下导致的该函数执行如此之长呢？



#### ipvs estimation_timer 定时器

谜底终将揭晓！

我们通过阅读 ipvs 相关的源码，发现 `estimation_timer()[ipvs]` 函数针对每个 Network Namespace 创建时候的通过 **ip_vs_core.c**[9] 中的 `__ip_vs_init` 初始化的：

```c
/*
 * Initialize IP Virtual Server netns mem.
 */
static int __net_init __ip_vs_init(struct net *net)
{
 struct netns_ipvs *ipvs;
  // ...
 if (ip_vs_estimator_net_init(ipvs) < 0)  // 初始化
  goto estimator_fail;
}
```

`ip_vs_estimator_net_init` 函数在文件 **ip_vs_est.c**[10] 中，定义如下：

```c
int __net_init ip_vs_estimator_net_init(struct netns_ipvs *ipvs)
{
 INIT_LIST_HEAD(&ipvs->est_list);
 spin_lock_init(&ipvs->est_lock);
 timer_setup(&ipvs->est_timer, estimation_timer, 0);  // 设置定时器函数 estimation_timer
 mod_timer(&ipvs->est_timer, jiffies + 2 * HZ);       // 启动第一次计时器，2秒启动
 return 0;
}
```

`estimation_timer` 也定义在 **ip_vs_est.c**[11] 文件中。

```c
static void estimation_timer(struct timer_list *t)
{
  // ...
 spin_lock(&ipvs->est_lock);
 list_for_each_entry(e, &ipvs->est_list, list) {
  s = container_of(e, struct ip_vs_stats, est);

  spin_lock(&s->lock);
  ip_vs_read_cpu_stats(&s->kstats, s->cpustats);

  /* scaled by 2^10, but divided 2 seconds */
  rate = (s->kstats.conns - e->last_conns) << 9;
  e->last_conns = s->kstats.conns;
  e->cps += ((s64)rate - (s64)e->cps) >> 2;

    // ...
 }
 spin_unlock(&ipvs->est_lock);
 mod_timer(&ipvs->est_timer, jiffies + 2*HZ);  // 2 秒后启动新的一轮统计
}
```

从 `estimation_timer` 的函数实现来看，会首先调用 spin_lock 进行锁的操作，然后遍历当前 Network Namespace 下的全部 ipvs 规则。由于我们集群的某些历史原因导致生产集群中的 Service 比较多，因此导致一次遍历的时候会占用比较长的时间。

该函数的统计最终体现在 `ipvsadm --stat` 的结果中（Conns InPkts OutPkts InBytes OutBytes）：

```sh
$ ipvsadm -Ln --stats
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port               Conns   InPkts  OutPkts  InBytes OutBytes  # 相关统计
  -> RemoteAddress:Port
TCP  10.85.0.10:9153                     0        0        0        0        0
  -> 172.22.34.187:9153                  0        0        0        0        0
```

对于我们集群中的 `ipvs` 规则进行统计，我们发现大概在 30000 左右。

```sh
$ ipvsadm -Ln --stats|wc -l
```

既然每个 Network Namespace 下都会有 `estimation_timer` 的遍历，为什么只有 CPU#0 上的规则如此多呢？

这是因为只有主机的 Host Network Namespace 中才会有全部的 ipvs 规则，这个我们也可以通过 `ipvsadm -Ln` (执行在 Host Network Namespace 下) 验证。从现象来看，CPU#0 是 ipvs 模块加载的时候用于处理宿主机 Host Network Namespace 中的 ipvs 规则，当然这个核的加载完全是随机的。



### **3. 问题解决**

#### 解决方案

到此，问题已经彻底定位，由于我们服务早期部署的历史原因，短期内调整 Service 的数目会导致大量的迁移工作，中间还有云厂商 SLB 产生的大量规则，也没有办法彻底根除，单从技术上解决的话，我们可以采用的方式有以下 3 种：

1. 动态感知到宿主机 Network Namespace 中 ipvs `estimation_timer` 函数的函数，在 RPS 中设置关闭该 CPU 映射；

   该方式需要调整 RPS 的配置，而且 ipvs 处理主机 Network Namespace 的核数不固定，需要识别并调整配置，还需要处理重启时候的 ipvs 主机 Network Namespace 的变动；

2. 由于我们不需要 ipvs 这种统计的功能，可以通过修改 ipvs 驱动的方式来规避该问题；

   修改 ipvs 的驱动模块，需要重新加载该内核模块，也会导致主机服务上的短暂中断；

3. ipvs 模块将内核遍历统计调整成一个独立的内核线程进行统计；

ipvs 规则在内核 timer 中遍历是 ipvs 移植到 k8s 上场景未适配的问题，社区应该需要把在 timer 中的遍历独立出去，但是这个方案需要社区的推动解决，远水解不了近渴。

通过上述 3 种方案的对比，解决我们当前抖动的问题都不太容易实施，为了保证生产环境的稳定和实施的难易程度，最终我们把眼光定位在 Linux Kernel 热修的 **kpatch**[12] 方案上， kpath 实现的 livepatch 功能可以实时为正在运行的内核提供功能增强，无需重新启动系统。



#### kpatch livepatch

Kpatch 是给 Linux 内核 livepatch 的工具，由 Redhat 公司出品。最早出现的打热补丁工具是 Ksplice。但是 Ksplice 被 Oracle 收购后，一些发行版生产商就不得不开发自己的热补丁工具，分别是 Redhat 的 Kpatch 和 Suse 的 KGraft。同时，在这两家厂商的推进下，kernel 4.0 开始，开始集成了 livepatch 技术。Kpatch 虽然是 Redhat 研发，但其也支持 Ubuntu、Debian、Oracle Linux 等的发行版。

这里我们简单同步一下实施的步骤，更多的文档可以从 kpath 项目中获取。



##### 获取 kpath 编译和安装

```sh
$ git clone https://github.com/dynup/kpatch.git
$ source test/integration/lib.sh
# 中间会使用 yum 安装相关的依赖包，安装时间视网络情况而定，在阿里云的环境下需要的时间比较长
$ sudo kpatch_dependencies
$ cd kpatch

# 进行编译
$ make

# 默认安装到 /usr/local，需要注意 kpatch-build 在目录 /usr/local/bin/ 下，而 kpatch 在 /usr/local/sbin/ 目录
$ sudo make install
```



##### 生成内核源码 patch

在 kpatch 的使用过程中，需要使用到内核的源码，源码拉取的方式可以参考这里[我需要内核的源代码](<https://wiki.centos.org/zh/HowTos/I_need_the_Kernel_Source?highlight=(kernel "我需要内核的源代码")|(src)>)。

```sh
$ rpm2cpio kernel-3.10.0-1062.9.1.el7.src.rpm |cpio -div
$ xz -d linux-3.10.0-1062.9.1.el7.tar.xz
$ tar -xvf linux-3.10.0-1062.9.1.el7.tar
$ cp -ra linux-3.10.0-1062.9.1.el7/ linux-3.10.0-1062.9.1.el7-patch
```

此处我们将 `estimation_timer` 函数的实现设置为空

```c
static void estimation_timer(unsigned long arg)
{
    printk("hotfix estimation_timer patched\n");
    return;
}
```

并生成对应的 patch 文件

```sh
$ diff -u linux-3.10.0-1062.9.1.el7/net/netfilter/ipvs/ip_vs_est.c linux-3.10.0-1062.9.1.el7-patch/net/netfilter/ipvs/ip_vs_est.c > ip_vs_timer_v1.patch
```



##### 生产内核补丁并 livepatch

然后生成相关的 patch ko 文件并应用到内核：

```sh
$ /usr/local/bin/kpatch-build ip_vs_timer_v1.patch --skip-gcc-check --skip-cleanup -r /root/kernel-3.10.0-1062.9.1.el7.src.rpm

# 编译成功后会在当前目录生成 livepatch-ip_vs_timer_v1.ko 文件
# 应用到内核中.

$ /usr/local/sbin/kpatch load livepatch-ip_vs_timer_v1.ko
```

通过内核日志查看确认

```sh
$ dmesg -T
[Thu Dec  3 19:50:50 2020] livepatch: enabling patch 'livepatch_ip_vs_timer_v1'
[Thu Dec  3 19:50:50 2020] livepatch: 'livepatch_ip_vs_timer_v1': starting patching transition
[Thu Dec  3 19:50:50 2020] hotfix estimation_timer patched
```

至此通过我们的 livepatch 成功修订了 `estimation_timer` 的调用，一切看起来很成功。然后通过 `funcgraph` 工具查看 `estimation_timer` 函数不再出现在调用关系中。

> 如果仅仅把函数设置为空的实现，等于是关闭了 `estimation_timer` 的调用，即使通过命令 unload 掉 livepatch，该函数也不会恢复，因此在生产环境中建议将函数的 2s 调用设置成个可以接受的时间范围内，比如 5 分钟，这样在 unload 以后，可以在 5 分钟以后恢复 `estimation_timer` 的继续调用。



##### 使用 kpatch 注意事项

- kpatch 是基于内核版本生成的 ko 内核模块，必须保证后续 livepatch 的内核版本与编译机器的内核完全一致。
- 通过手工 livepatch 的方式修复，如果保证机器在重启以后仍然生效需要通过 `install` 来启用 kpatch 服务进行保证。

```sh
$ /usr/local/sbin/kpatch install livepatch-ip_vs_timer_v1.ko
$ systemctl start kpatch
```

- 在其他的机器上进行 livepatch 需要文件`kpatch`、`livepatch-ip_vs_timer_v1.ko` 和 `kpatch.service`（用于 install 后重启生效） 3 个文件即可。



### 总结

网络抖动问题的排查，涉及应用层、网络协议栈和内核中运作机制等多方面的协调，排查过程中需要逐层排查、逐步缩小范围，在整个过程中，合适的工具至关重要，在我们本次问题的排查过程中， BPF 技术为我们排查的方向起到了至关重要的作用。BPF 技术的出现为我们观测和跟踪内核中的事件，提供了更加灵活的数据采集和数据分析的能力，在生产环境中我们已经将其广泛用于了监控网络底层的重传和抖动等维度，极大提升我们在偶发场景下的问题排查效率，希望更多的人能够从 BPF 技术中受益。



参考资料

[1]terway: *https://github.com/AliyunContainerService/terway*

[2]BCC: *https://github.com/iovisor/bcc*

[3]**traceicmpsoftirq.py**: *https://gist.github.com/DavadDi/62ee75228f03631c845c51af292c2b17*

[4]INSTALL.md: *https://github.com/iovisor/bcc/blob/master/INSTALL.md*

[5]get_rps_cpu: *https://elixir.bootlin.com/linux/v5.8/source/net/core/dev.c#L4305*

[6]perf-tools: *https://github.com/brendangregg/perf-tools*

[7]funcgraph: *https://github.com/brendangregg/perf-tools/blob/master/bin/funcgraph*

[8]softirqs.py: *https://github.com/iovisor/bcc/blob/master/tools/softirqs.py*

[9]ipvs]` 函数针对每个 Network Namespace 创建时候的通过 [ip_vs_core.c: *https://elixir.bootlin.com/linux/v5.8/source/net/netfilter/ipvs/ip_vs_core.c#L2469*

[10]ip_vs_est.c: *https://elixir.bootlin.com/linux/v5.8/source/net/netfilter/ipvs/ip_vs_est.c#L187*

[11]ip_vs_est.c: *https://elixir.bootlin.com/linux/v5.8/source/net/netfilter/ipvs/ip_vs_est.c#L96*

[12]kpatch: *https://github.com/dynup/kpatch*



## Kubernetes 网络疑难杂症排查分享



### 跨 VPC 访问 NodePort 经常超时

现象: 从 VPC a 访问 VPC b 的 TKE 集群的某个节点的 NodePort，有时候正常，有时候会卡住直到超时。

原因怎么查？

当然是先抓包看看啦，抓 server 端 NodePort 的包，发现异常时 server 能收到 SYN，但没响应 ACK:

反复执行 `netstat -s | grep LISTEN` 发现 SYN 被丢弃数量不断增加:

![image-20210109220218509](D:\学习资料\笔记\k8s\k8s图\image-20210109220218509.png)

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

由于 `tcp_tw_recycle` 坑太多，在内核 4.12 之后已移除: https://github.com/torvalds/linux/commit/4396e46187ca5070219b81773c4e65088dac50cc



### LB 压测 CPS 低

现象: LoadBalancer 类型的 Service，直接压测 NodePort CPS 比较高，但如果压测 LB CPS 就很低。

环境说明: 用户使用的黑石TKE，不是公有云TKE，黑石的机器是物理机，LB的实现也跟公有云不一样，但 LoadBalancer 类型的 Service 的实现同样也是 LB 绑定各节点的 NodePort，报文发到 LB 后转到节点的 NodePort， 然后再路由到对应 pod，而测试在公有云 TKE 环境下没有这个问题。

client 抓包: 大量SYN重传。

server 抓包: 抓 NodePort 的包，发现当 client SYN 重传时 server 能收到 SYN 包但没有响应。

又是 SYN 收到但没响应，难道又是开启 `tcp_tw_recycle` 导致的？检查节点的内核参数发现并没有开启，除了这个原因，还会有什么情况能导致被丢弃？

`conntrack -S` 看到 `insert_failed` 数量在不断增加，也就是 conntrack 在插入很多新连接的时候失败了，为什么会插入失败？什么情况下会插入失败？

挖内核源码: netfilter conntrack 模块为每个连接创建 conntrack 表项时，表项的创建和最终插入之间还有一段逻辑，没有加锁，是一种乐观锁的过程。conntrack 表项并发刚创建时五元组不冲突的话可以创建成功，但中间经过 NAT 转换之后五元组就可能变成相同，第一个可以插入成功，后面的就会插入失败，因为已经有相同的表项存在。比如一个 SYN 已经做了 NAT 但是还没到最终插入的时候，另一个 SYN 也在做 NAT，因为之前那个 SYN 还没插入，这个 SYN 做 NAT 的时候就认为这个五元组没有被占用，那么它 NAT 之后的五元组就可能跟那个还没插入的包相同。

在我们这个问题里实际就是 netfilter 做 SNAT 时源端口选举冲突了，黑石 LB 会做 SNAT，SNAT 时使用了 16 个不同 IP 做源，但是短时间内源 Port 却是集中一致的，并发两个 SYN a 和SYN b，被 LB SNAT 后源 IP 不同但源 Port 很可能相同，这里就假设两个报文被 LB SNAT 之后它们源 IP 不同源 Port 相同，报文同时到了节点的 NodePort 会再次做 SNAT 再转发到对应的 Pod，当报文到了 NodePort 时，这时它们五元组不冲突，netfilter 为它们分别创建了 conntrack 表项，SYN a 被节点 SNAT 时默认行为是 从 port_range 范围的当前源 Port 作为起始位置开始循环遍历，选举出没有被占用的作为源 Port，因为这两个 SYN 源 Port 相同，所以它们源 Port 选举的起始位置相同，当 SYN a 选出源 Port 但还没将 conntrack 表项插入时，netfilter 认为这个 Port 没被占用就很可能给 SYN b 也选了相同的源 Port，这时他们五元组就相同了，当 SYN a 的 conntrack 表项插入后再插入 SYN b 的 conntrack 表项时，发现已经有相同的记录就将 SYN b 的 conntrack 表项丢弃了。

解决方法探索: 不使用源端口选举，在 iptables 的 MASQUERADE 规则如果加 `--random-fully` 这个 flag 可以让端口选举完全随机，基本上能避免绝大多数的冲突，但也无法完全杜绝。最终决定开发 LB 直接绑 Pod IP，不基于 NodePort，从而避免 netfilter 的 SNAT 源端口冲突问题。



### **DNS 解析偶尔 5S 延时**

网上一搜，是已知问题，仔细分析，实际跟之前黑石 TKE 压测 LB CPS 低的根因是同一个，都是因为 netfilter conntrack 模块的设计问题，只不过之前发生在 SNAT，这个发生在 DNAT，这里用我的语言来总结下原因:

DNS client (glibc 或 musl libc) 会并发请求 A 和 AAAA 记录，跟 DNS Server 通信自然会先 connect (建立fd)，后面请求报文使用这个 fd 来发送，由于 UDP 是无状态协议， connect 时并不会创建 conntrack 表项, 而并发请求的 A 和 AAAA 记录默认使用同一个 fd 发包，这时它们源 Port 相同，当并发发包时，两个包都还没有被插入 conntrack 表项，所以 netfilter 会为它们分别创建 conntrack 表项，而集群内请求 kube-dns 或 coredns 都是访问的CLUSTER-IP，报文最终会被 DNAT 成一个 endpoint 的 POD IP，当两个包被 DNAT 成同一个 IP，最终它们的五元组就相同了，在最终插入的时候后面那个包就会被丢掉，如果 dns 的 pod 副本只有一个实例的情况就很容易发生，现象就是 dns 请求超时，client 默认策略是等待 5s 自动重试，如果重试成功，我们看到的现象就是 dns 请求有 5s 的延时。

参考 weave works 工程师总结的文章 Racy conntrack and DNS lookup timeouts: https://www.weave.works/blog/racy-conntrack-and-dns-lookup-timeouts



#### 解决方案一: 使用 TCP 发送 DNS 请求

如果使用 TCP 发 DNS 请求，connect 时就会插入 conntrack 表项，而并发的 A 和 AAAA 请求使用同一个 fd，所以只会有一次 connect，也就只会尝试创建一个 conntrack 表项，也就避免插入时冲突。

`resolv.conf` 可以加 `options use-vc` 强制 glibc 使用 TCP 协议发送 DNS query。下面是这个 `man resolv.conf`中关于这个选项的说明:

```sh
use-vc (since glibc 2.14)
                     Sets RES_USEVC in _res.options.  This option forces the
                     use of TCP for DNS resolutions.
```



#### 解决方案二: 避免相同五元组 DNS 请求的并发

`resolv.conf` 还有另外两个相关的参数：

- single-request-reopen (since glibc 2.9): A 和 AAAA 请求使用不同的 socket 来发送，这样它们的源 Port 就不同，五元组也就不同，避免了使用同一个 conntrack 表项。
- single-request (since glibc 2.10): A 和 AAAA 请求改成串行，没有并发，从而也避免了冲突。

`man resolv.conf` 中解释如下:

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

要给容器的 `resolv.conf` 加上 options 参数，最方便的是直接在 Pod Spec 里面的 dnsConfig 加 (k8s v1.9 及以上才支持)

```yaml
spec:
  dnsConfig:
    options:
      - name: single-request-reopen
```

加 options 还有其它一些方法:

- 在容器的 `ENTRYPOINT` 或者 `CMD` 脚本中，执行 `/bin/echo 'options single-request-reopen' >> /etc/resolv.conf`
- 在 postStart hook 里加:

```yaml
lifecycle:
  postStart:
    exec:
      command:
      - /bin/sh
      - -c
      - "/bin/echo 'options single-request-reopen' >> /etc/resolv.conf"
```

- 使用 MutatingAdmissionWebhook，这是 1.9 引入的 Controller，用于对一个指定的资源的操作之前，对这个资源进行变更。istio 的自动 sidecar 注入就是用这个功能来实现的，我们也可以通过 `MutatingAdmissionWebhook` 来自动给所有 Pod 注入 `resolv.conf` 文件，不过需要一定的开发量。



#### 解决方案三: 使用本地 DNS 缓存

仔细观察可以看到前面两种方案是 glibc 支持的，而基于 alpine 的镜像底层库是 musl libc 不是 glibc，所以即使加了这些 options 也没用，这种情况可以考虑使用本地 DNS 缓存来解决，容器的 DNS 请求都发往本地的 DNS 缓存服务(dnsmasq, nscd等)，不需要走 DNAT，也不会发生 conntrack 冲突。另外还有个好处，就是避免 DNS 服务成为性能瓶颈。

使用本地DNS缓存有两种方式：

- 每个容器自带一个 DNS 缓存服务
- 每个节点运行一个 DNS 缓存服务，所有容器都把本节点的 DNS 缓存作为自己的 nameserver

从资源效率的角度来考虑的话，推荐后一种方式。



### Pod 访问另一个集群的 apiserver 有延时

现象：集群 a 的 Pod 内通过 kubectl 访问集群 b 的内网地址，偶尔出现延时的情况，但直接在宿主机上用同样的方法却没有这个问题。

提炼环境和现象精髓:

1. 在 pod 内将另一个集群 apiserver 的 ip 写到了 hosts，因为 TKE apiserver 开启内网集群外内网访问创建的内网 LB 暂时没有支持自动绑内网 DNS 域名解析，所以集群外的内网访问 apiserver 需要加 hosts
2. pod 内执行 kubectl 访问另一个集群偶尔延迟 5s，有时甚至10s

观察到 5s 延时，感觉跟之前 conntrack 的丢包导致 dns 解析 5s 延时有关，但是加了 hosts 呀，怎么还去解析域名？

进入 pod netns 抓包: 执行 kubectl 时确实有 dns 解析，并且发生延时的时候 dns 请求没有响应然后做了重试。

看起来延时应该就是之前已知 conntrack 丢包导致 dns 5s 超时重试导致的。但是为什么会去解析域名? 明明配了 hosts 啊，正常情况应该是优先查找 hosts，没找到才去请求 dns 呀，有什么配置可以控制查找顺序?

搜了一下发现: `/etc/nsswitch.conf` 可以控制，但看有问题的 pod 里没有这个文件。然后观察到有问题的 pod 用的 alpine 镜像，试试其它镜像后发现只有基于 alpine 的镜像才会有这个问题。

再一搜发现: musl libc 并不会使用 `/etc/nsswitch.conf` ，也就是说 alpine 镜像并没有实现用这个文件控制域名查找优先顺序，瞥了一眼 musl libc 的 `gethostbyname` 和 `getaddrinfo` 的实现，看起来也没有读这个文件来控制查找顺序，写死了先查 hosts，没找到再查 dns。

这么说，那还是该先查 hosts 再查 dns 呀，为什么这里抓包看到是先查的 dns? (如果是先查 hosts 就能命中查询，不会再发起dns请求)

访问 apiserver 的 client 是 kubectl，用 go 写的，会不会是 go 程序解析域名时压根没调底层 c 库的 `gethostbyname` 或 `getaddrinfo`?

搜一下发现果然是这样: go runtime 用 go 实现了 glibc 的 `getaddrinfo` 的行为来解析域名，减少了 c 库调用 (应该是考虑到减少 cgo 调用带来的的性能损耗)

issue: net: replicate DNS resolution behaviour of getaddrinfo(glibc) in the go dns resolver

翻源码验证下:

Unix 系的 OS 下，除了 openbsd， go runtime 会读取 `/etc/nsswitch.conf` (`net/conf.go`):

![image-20210109224809381](D:\学习资料\笔记\k8s\k8s图\image-20210109224809381.png)

`hostLookupOrder` 函数决定域名解析顺序的策略，Linux 下，如果没有 `nsswitch.conf` 文件就 dns 比 hosts 文件优先 (`net/conf.go`):

![image-20210109224825205](D:\学习资料\笔记\k8s\k8s图\image-20210109224825205.png)

可以看到 `hostLookupDNSFiles` 的意思是 dns first (`net/dnsclient_unix.go`):

![image-20210109224848232](D:\学习资料\笔记\k8s\k8s图\image-20210109224848232.png)

所以虽然 alpine 用的 musl libc 不是 glibc，但 go 程序解析域名还是一样走的 glibc 的逻辑，而 alpine 没有 `/etc/nsswitch.conf` 文件，也就解释了为什么 kubectl 访问 apiserver 先做 dns 解析，没解析到再查的 hosts，导致每次访问都去请求 dns，恰好又碰到 conntrack 那个丢包问题导致 dns 5s 延时，在用户这里表现就是 pod 内用 kubectl 访问 apiserver 偶尔出现 5s 延时，有时出现 10s 是因为重试的那次 dns 请求刚好也遇到 conntrack 丢包导致延时又叠加了 5s 。

解决方案:

1. 换基础镜像，不用 alpine
2. 挂载 `nsswitch.conf` 文件 (可以用 hostPath)



### DNS 解析异常

现象: 有个用户反馈域名解析有时有问题，看报错是解析超时。

第一反应当然是看 coredns 的 log:

```sh
[ERROR] 2 loginspub.gaeamobile-inc.net.
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

- 请求 `loginspub.xxxx-inc.net` 时，偶尔提示域名无法解析
- 请求 `accounts.google.com` 时，偶尔提示连接失败

进入 dns 解析偶尔异常的容器的 netns 抓包:

- dns 请求会并发请求 A 和 AAAA 记录
- 测试脚本发请求打印序号，抓包然后 wireshark 分析对比异常时请求序号偏移量，找到异常时的 dns 请求报文，发现异常时 A 和 AAAA 记录的请求 id 冲突，并且 AAAA 响应先返回

正常情况下id不会冲突，这里冲突了也就能解释这个 dns 解析异常的现象了:

- `loginspub.xxxx-inc.net` 没有 AAAA (ipv6) 记录，它的响应先返回告知 client 不存在此记录，由于请求 id 跟 A 记录请求冲突，后面 A 记录响应返回了 client 发现 id 重复就忽略了，然后认为这个域名无法解析
- `accounts.google.com` 有 AAAA 记录，响应先返回了，client 就拿这个记录去尝试请求，但当前容器环境不支持 ipv6，所以会连接失败

那为什么 dns 请求 id 会冲突?

继续观察发现: 其它节点上的 pod 不会复现这个问题，有问题这个节点上也不是所有 pod 都有这个问题，只有基于 alpine 镜像的容器才有这个问题，在此节点新起一个测试的 `alpine:latest` 的容器也一样有这个问题。

为什么 alpine 镜像的容器在这个节点上有问题在其它节点上没问题？为什么其他镜像的容器都没问题？它们跟 alpine 的区别是什么？

发现一点区别: alpine 使用的底层 c 库是 musl libc，其它镜像基本都是 glibc

翻 musl libc 源码, 构造 dns 请求时，请求 id 的生成没加锁，而且跟当前时间戳有关:

![image-20210109225614105](D:\学习资料\笔记\k8s\k8s图\image-20210109225614105.png)

看注释，作者应该认为这样id基本不会冲突，事实证明，绝大多数情况确实不会冲突，我在网上搜了很久没有搜到任何关于 musl libc 的 dns 请求 id 冲突的情况。这个看起来取决于硬件，可能在某种类型硬件的机器上运行，短时间内生成的 id 就可能冲突。我尝试跟用户在相同地域的集群，添加相同配置相同机型的节点，也复现了这个问题，但后来删除再添加时又不能复现了，看起来后面新建的 cvm 又跑在了另一种硬件的母机上了。

OK，能解释通了，再底层的细节就不清楚了，我们来看下解决方案:

1. 换基础镜像 (不用alpine)
2. 完全静态编译业务程序(不依赖底层c库)，比如go语言程序编译时可以关闭 cgo (CGO_ENABLED=0)，并告诉链接器要静态链接 (`go build` 后面加 `-ldflags '-d'`)，但这需要语言和编译工具支持才可以

最终建议用户基础镜像换成另一个比较小的镜像: `debian:stretch-slim`。

问题解决，但用户后面觉得 `debian:stretch-slim` 做出来的镜像太大了，有 6MB 多，而之前基于 alpine 做出来只有 1MB 多，最后使用了一个非官方的修改过 musl libc 的 alpine 镜像作为基础镜像，里面禁止了 AAAA 请求从而避免这个问题。



### Pod 偶尔存活检查失败

现象: Pod 偶尔会存活检查失败，导致 Pod 重启，业务偶尔连接异常。

之前从未遇到这种情况，在自己测试环境尝试复现也没有成功，只有在用户这个环境才可以复现。这个用户环境流量较大，感觉跟连接数或并发量有关。

用户反馈说在友商的环境里没这个问题。

对比友商的内核参数发现有些区别，尝试将节点内核参数改成跟友商的一样，发现问题没有复现了。

再对比分析下内核参数差异，最后发现是 backlog 太小导致的，节点的 `net.ipv4.tcp_max_syn_backlog` 默认是 1024，如果短时间内并发新建 TCP 连接太多，SYN 队列就可能溢出，导致部分新连接无法建立。

解释一下:

![image-20210109225852143](D:\学习资料\笔记\k8s\k8s图\image-20210109225852143.png)

TCP 连接建立会经过三次握手，server 收到 SYN 后会将连接加入 SYN 队列，当收到最后一个 ACK 后连接建立，这时会将连接从 SYN 队列中移动到 ACCEPT 队列。在 SYN 队列中的连接都是没有建立完全的连接，处于半连接状态。如果 SYN 队列比较小，而短时间内并发新建的连接比较多，同时处于半连接状态的连接就多，SYN 队列就可能溢出，`tcp_max_syn_backlog` 可以控制 SYN 队列大小，用户节点的 backlog 大小默认是 1024，改成 8096 后就可以解决问题。



### 访问 externalTrafficPolicy 为 Local 的 Service 对应 LB 有时超时

现象：用户在 TKE 创建了公网 LoadBalancer 类型的 Service，externalTrafficPolicy 设为了 Local，访问这个 Service 对应的公网 LB 有时会超时。

externalTrafficPolicy 为 Local 的 Service 用于在四层获取客户端真实源 IP，官方参考文档：https://kubernetes.io/docs/tutorials/services/source-ip/#source-ip-for-services-with-type-loadbalancer

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

如何解决呢? 发现一个内核参数: accept_local 可以让 linux 接收源 IP 是本机 IP 的报文。

试了开启这个参数，确实在 cbr0 收到来自 LB 的探测报文了，说明报文能被 pod 收到，但抓 eth0 还是没有给 LB 回包。

为什么没有回包? 分析下五元组，要给 LB 回包，那么 `目的IP:目的Port` 必须是探测报文的 `源IP:源Port`，所以目的 IP 就是 LB IP，由于容器不在主 netns，发包经过 veth pair 到 cbr0 之后需要再经过 netfilter 处理，报文进入 PREROUTING 链然后发现目的 IP 是本机 IP，进入 INPUT 链，所以报文就出不去了。再分析下进入 INPUT 后会怎样，因为目的 Port 跟 LB 探测报文源 Port 相同，是一个随机端口，不在 Service 的端口列表，所以没有对应的 IPVS 规则，IPVS 也就不会转发它，而 `kube-ipvs0` 上虽然绑了这个 IP，但它是一个 dummy interface，不会收包，所以报文最后又被忽略了。

再看看为什么 1.12 启用 ipvs 会绑 `EXTERNAL-IP` 到 `kube-ipvs0`，翻翻 k8s 的 kube-proxy 支持 ipvs 的 proposal，发现有个地方说法有点漏洞:

![image-20210109230739403](D:\学习资料\笔记\k8s\k8s图\image-20210109230739403.png)

LB 类型 Service 的 status 里有 ingress IP，实际就是 `kubectl get service` 看到的 `EXTERNAL-IP`，这里说不会绑定这个 IP 到 kube-ipvs0，但后面又说会给它创建 ipvs 规则，既然没有绑到 `kube-ipvs0`，那么这个 IP 的报文根本不会进入 INPUT 被 ipvs 模块转发，创建的 ipvs 规则也是没用的。

后来找到作者私聊，思考了下，发现设计上确实有这个问题。

看了下 1.10 确实也是这么实现的，但是为什么 1.12 又绑了这个 IP 呢? 调研后发现是因为 #59976 这个 issue 发现一个问题，后来引入 #63066 这个 PR 修复的，而这个 PR 的行为就是让 LB IP 绑到 `kube-ipvs0`，这个提交影响 1.11 及其之后的版本。

\#59976 的问题是因为没绑 LB IP到 `kube-ipvs0` 上，在自建集群使用 `MetalLB` 来实现 LoadBalancer 类型的 Service，而有些网络环境下，pod 是无法直接访问 LB 的，导致 pod 访问 LB IP 时访问不了，而如果将 LB IP 绑到 `kube-ipvs0` 上就可以通过 ipvs 转发到 LB 类型 Service 对应的 pod 去， 而不需要真正经过 LB，所以引入了 #63066 这个PR。

临时方案: 将 #63066 这个 PR 的更改回滚下，重新编译 kube-proxy，提供升级脚本升级存量 kube-proxy。

如果是让 LB 健康检查探测支持用保留 IP 而不是自身的公网 IP ，也是可以解决，但需要跨团队合作，而且如果多个厂商都遇到这个问题，每家都需要为解决这个问题而做开发调整，代价较高，所以长期方案需要跟社区沟通一起推进，所以我提了 issue，将问题描述的很清楚: #79783

小思考: 为什么 CLB 可以不做 SNAT ? 回包目的 IP 就是真实客户端 IP，但客户端是直接跟 LB IP 建立的连接，如果回包不经过 LB 是不可能发送成功的呀。

是因为 CLB 的实现是在母机上通过隧道跟 CVM 互联的，多了一层封装，回包始终会经过 LB。

就是因为 CLB 不做 SNAT，正常来自客户端的报文是可以发送到 nodeport，但健康检查探测报文由于源 IP 是 LB IP 被绑到 `kube-ipvs0` 导致被忽略，也就解释了为什么健康检查失败，但通过LB能访问后端服务，只是有时会超时。那么如果要做 SNAT 的 LB 岂不是更糟糕，所有报文都变成 LB IP，所有报文都会被忽略?

我提的 issue 有回复指出，AWS 的 LB 会做 SNAT，但它们不将 LB 的 IP 写到 Service 的 Status 里，只写了 hostname，所以也不会绑 LB IP 到 `kube-ipvs0`:

![image-20210109231015387](D:\学习资料\笔记\k8s\k8s图\image-20210109231015387.png)

但是只写 hostname 也得 LB 支持自动绑域名解析，并且个人觉得只写 hostname 很别扭，通过 `kubectl get svc` 或者其它 k8s 管理系统无法直接获取 LB IP，这不是一个好的解决方法。

我提了 #79976 这个 PR 可以解决问题: 给 kube-proxy 加 `--exclude-external-ip` 这个 flag 控制是否为 LB IP
创建 ipvs 规则和绑定 `kube-ipvs0`。

但有人担心增加 kube-proxy flag 会增加 kube-proxy 的调试复杂度，看能否在 iptables 层面解决:

![image-20210109231115935](D:\学习资料\笔记\k8s\k8s图\image-20210109231115935.png)

仔细一想，确实可行，打算有空实现下，重新提个 PR

![image-20210109231141751](D:\学习资料\笔记\k8s\k8s图\image-20210109231141751.png)



### 结语

至此，我们一起完成了一段奇妙的问题排查之旅，信息量很大并且比较复杂，有些没看懂很正常，但我希望你可以收藏起来反复阅读，一起在技术的道路上打怪升级。





## Kubernetes 疑难杂症排查分享: 诡异的 No route to host



### **问题反馈**

有用户反馈 Deployment 滚动更新的时候，业务日志偶尔会报 “No route to host” 的错误。



### **问题分析**

之前没遇到滚动更新会报 “No route to host” 的问题，我们先看下滚动更新导致连接异常有哪些常见的报错:

- `Connection reset by peer`: 连接被重置。通常是连接建立过，但 server 端发现 client 发的包不对劲就返回 RST，应用层就报错连接被重置。比如在 server 滚动更新过程中，client 给 server 发的请求还没完全结束，或者本身是一个类似 grpc 的多路复用长连接，当 server 对应的旧 Pod 删除(没有做优雅结束，停止时没有关闭连接)，新 Pod 很快创建启动并且刚好有跟之前旧 Pod 一样的 IP，这时 kube-proxy 也没感知到这个 IP 其实已经被删除然后又被重建了，针对这个 IP 的规则就不会更新，旧的连接依然发往这个 IP，但旧 Pod 已经不在了，后面继续发包时依然转发给这个 Pod IP，最终会被转发到这个有相同 IP 的新 Pod 上，而新 Pod 收到此包时检查报文发现不对劲，就返回 RST 给 client 告知将连接重置。针对这种情况，建议应用自身处理好优雅结束：Pod 进入 Terminating 状态后会发送 SIGTERM 信号给业务进程，业务进程的代码需处理这个信号，在进程退出前关闭所有连接。
- `Connection refused`: 连接被拒绝。通常是连接还没建立，client 正在发 SYN 包请求建立连接，但到了 server 之后发现端口没监听，内核就返回 RST 包，然后应用层就报错连接被拒绝。比如在 server 滚动更新过程中，旧的 Pod 中的进程很快就停止了(网卡还未完全销毁)，但 client 所在节点的 iptables/ipvs 规则还没更新，包就可能会被转发到了这个停止的 Pod (由于 k8s 的 controller 模式，从 Pod 删除到 service 的 endpoint 更新，再到 kube-proxy watch 到更新并更新 节点上的 iptables/ipvs 规则，这个过程是异步的，中间存在一点时间差，所以有可能存在 Pod 中的进程已经没有监听，但 iptables/ipvs 规则还没更新的情况)。针对这种情况，建议给容器加一个 preStop，在真正销毁 Pod 之前等待一段时间，留时间给 kube-proxy 更新转发规则，更新完之后就不会再有新连接往这个旧 Pod 转发了，preStop 示例:

```yaml
lifecycle:
  preStop:
    exec:
      command:
      - /bin/bash
      - -c
      - sleep 30
```

- `Connection timed out`: 连接超时。通常是连接还没建立，client 发 SYN 请求建立连接一直等到超时时间都没有收到 ACK，然后就报错连接超时。这个可能场景跟前面 Connection refused 可能的场景类似，不同点在于端口有监听，但进程无法正常响应了: 转发规则还没更新，旧 Pod 的进程正在停止过程中，虽然端口有监听，但已经不响应了；或者转发规则更新了，新 Pod 端口也监听了，但还没有真正就绪，还没有能力处理新请求。针对这些情况的建议跟前面一样：加 preStop 和 readinessProbe。



下面我们来继续分析下滚动更新时发生 "No route to host" 的可能情况。

这个报错很明显，IP 无法路由，通常是将报文发到了一个已经彻底销毁的 Pod (网卡已经不在)。不可能发到一个网卡还没创建好的 Pod，因为即便不加存活检查，也是要等到 Pod 网络初始化完后才可能 Ready，然后 kube-proxy 才会更新转发规则。

什么情况下会转发到一个已经彻底销毁的 Pod？借鉴前面几种滚动更新的报错分析，我们推测应该是 Pod 很快销毁了但转发规则还没更新，从而新的请求被转发了这个已经销毁的 Pod，最终报文到达这个 Pod 所在 PodCIDR 的 Node 上时，Node 发现本机已经没有这个 IP 的容器，然后 Node 就返回 ICMP 包告知 client 这个 IP 不可达，client 收到 ICMP 后，应用层就会报错 “No route to host”。

所以根据我们的分析，关键点在于 Pod 销毁太快，转发规则还没来得及更新，导致后来的请求被转发到已销毁的 Pod。针对这种情况，我们可以给容器加一个 preStop，留时间给 kube-proxy 更新转发规则来解决，参考 《Kubernetes实践指南》中的部分章节: 

**https://k8s.imroc.io/best-practice/high-availability-deployment-of-applications#smooth-update-using-prestophook-and-readinessprobe**



### **问题没有解决**

我们自己没有复现用户的 “No route to host” 的问题，可能是复现条件比较苛刻，最后将我们上面理论上的分析结论作为解决方案给到了用户。

但用户尝试加了 preStop 之后，问题依然存在，服务滚动更新时偶尔还是会出现 “No route to host”。



### **深入分析**

为了弄清楚根本原因，我们请求用户协助搭建了一个可以复现问题的测试环境，最终这个问题在测试环境中可以稳定复现。

仔细观察，实际是部署两个服务：ServiceA 和 ServiceB。使用 ab 压测工具去压测 ServiceA （短连接），然后 ServiceA 会通过 RPC 调用 ServiceB (短连接)，滚动更新的是 ServiceB，报错发生在 ServiceA 调用 ServiceB 这条链路。

在 ServiceB 滚动更新期间，新的 Pod Ready 了之后会被添加到 IPVS 规则的 RS 列表，但旧的 Pod 不会立即被踢掉，而是将新的 Pod 权重置为1，旧的置为 0，通过在 client 所在节点查看 IPVS 规则可以看出来:

```sh
root@VM-0-3-ubuntu:~$ ipvsadm -ln -t 172.16.255.241:80
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  172.16.255.241:80 rr
  -> 172.16.8.106:80              Masq    0      5          14048
  -> 172.16.8.107:80              Masq    1      2          243
```

为什么不立即踢掉旧的 Pod 呢？因为要支持优雅结束，让存量的连接处理完，等存量连接全部结束了再踢掉它(ActiveConn+InactiveConn=0)，这个逻辑可以通过这里的代码确认：**https://github.com/kubernetes/kubernetes/blob/v1.17.0/pkg/proxy/ipvs/graceful_termination.go#L170**
然后再通过 `ipvsadm -lnc | grep 172.16.8.106` 发现旧 Pod 上的连接大多是`TIME_WAIT`状态，这个也容易理解：因为 ServiceA 作为 client 发起短连接请求调用 ServiceB，调用完成就会关闭连接，TCP 三次挥手后进入`TIME_WAIT`状态，等待 2*MSL (2 分钟) 的时长再清理连接。

经过上面的分析，看起来都是符合预期的，那为什么还会出现 “No route to host” 呢？难道权重被置为 0 之后还有新连接往这个旧 Pod 转发？我们来抓包看下：

```sh
root@VM-0-3-ubuntu:~$ tcpdump -i eth0 host 172.16.8.106 -n -tttt
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
2019-12-13 11:49:47.319093 IP 10.0.0.3.36708 > 172.16.8.106.80: Flags [S], seq 3988339656, win 29200, options [mss 1460,sackOK,TS val 3751111666 ecr 0,nop,wscale 9], length 0
2019-12-13 11:49:47.319133 IP 10.0.0.3.36706 > 172.16.8.106.80: Flags [S], seq 109196945, win 29200, options [mss 1460,sackOK,TS val 3751111666 ecr 0,nop,wscale 9], length 0
2019-12-13 11:49:47.319144 IP 10.0.0.3.36704 > 172.16.8.106.80: Flags [S], seq 1838682063, win 29200, options [mss 1460,sackOK,TS val 3751111666 ecr 0,nop,wscale 9], length 0
2019-12-13 11:49:47.319153 IP 10.0.0.3.36702 > 172.16.8.106.80: Flags [S], seq 1591982963, win 29200, options [mss 1460,sackOK,TS val 3751111666 ecr 0,nop,wscale 9], length 0
```

果然是！即使权重为 0，仍然会尝试发 SYN 包跟这个旧 Pod 建立连接，但永远无法收到 ACK，因为旧 Pod 已经销毁了。为什么会这样呢？难道是 IPVS 内核模块的调度算法有问题？尝试去看了下 linux 内核源码，并没有发现哪个调度策略的实现函数会将新连接调度到权重为 0 的 rs 上。

这就奇怪了，可能不是调度算法的问题？继续尝试看更多的代码，主要是`net/netfilter/ipvs/ip_vs_core.c`中的`ip_vs_in`函数，也就是 IPVS 模块处理报文的主要入口，发现它会先在本地连接转发表看这个包是否已经有对应的连接了（匹配五元组），如果有就说明它不是新连接也就不会调度，直接发给这个连接对应的之前已经调度过的 rs (也不会判断权重)；如果没匹配到说明这个包是新的连接，就会走到调度这里 (rr, wrr 等调度策略)，这个逻辑看起来也没问题。

那为什么会转发到权重为 0 的 rs ？难道是匹配连接这里出问题了？新的连接匹配到了旧的连接？我开始做实验验证这个猜想，修改一下这里的逻辑：检查匹配到的连接对应的 rs 如果权重为 0，则重新调度。然后重新编译和加载 IPVS 内核模块，再重新压测一下，发现问题解决了！没有报 “No route to host” 了。

虽然通过改内核源码解决了，但我知道这不是一个好的解决方案，它会导致 IPVS 不支持连接的优雅结束，因为不再转发包给权重为 0 的 rs，存量的连接就会立即中断。

继续陷入深思……

这个实验只是证明了猜想：新连接匹配到了旧连接。那为什么会这样呢？难道新连接报文的五元组跟旧连接的相同了？

经过一番思考，发现这个是有可能的。因为 ServiceA 作为 client 请求 ServiceB，不同请求的源 IP 始终是相同的，关键点在于源端口是否可能相同。由于 ServiceA 向 ServiceB 发起大量短连接，ServiceA 所在节点就会有大量`TIME_WAIT`状态的连接，需要等 2 分钟 (2*MSL) 才会清理，而由于连接量太大，每次发起的连接都会占用一个源端口，当源端口不够用了，就会重用`TIME_WAIT`状态连接的源端口，这个时候当报文进入 IPVS 模块，检测到它的五元组跟本地连接转发表中的某个连接一致(`TIME_WAIT` 状态)，就以为它是一个存量连接，然后直接将报文转发给这个连接之前对应的 rs 上，然而这个 rs 对应的 Pod 早已销毁，所以抓包看到的现象是将 SYN 发给了旧 Pod，并且无法收到 ACK，伴随着返回 ICMP 告知这个 IP 不可达，也被应用解释为 “No route to host”。

后来无意间又发现一个还在 open 状态的 issue，虽然还没提到 “No route to host” 关键字，但讨论的跟我们这个其实是同一个问题。我也参与了讨论，有兴趣的同学可以看下：**https://github.com/kubernetes/kubernetes/issues/81775**



### **总结**

这个问题通常发生的场景就是类似于我们测试环境这种：ServiceA 对外提供服务，当外部发起请求，ServiceA 会通过 RPC 或 HTTP 调用 ServiceB，如果外部请求量变大，ServiceA 调用 ServiceB 的量也会跟着变大，大到一定程度，ServiceA 所在节点源端口不够用，复用`TIME_WAIT`状态连接的源端口，导致五元组跟 IPVS 里连接转发表中的`TIME_WAIT`连接相同，IPVS 就认为这是一个存量连接的报文，就不判断权重直接转发给之前的 rs，导致转发到已销毁的 Pod，从而发生 “No route to host”。

如何规避？集群规模小可以使用 iptables 模式，如果需要使用 ipvs 模式，可以增加 ServiceA 的副本，并且配置反亲和性 (podAntiAffinity)，让 ServiceA 的 Pod 部署到不同节点，分摊流量，避免流量集中到某一个节点，导致调用 ServiceB 时源端口复用。

如何彻底解决？暂时还没有一个完美的方案。

Issue 85517 讨论让 kube-proxy 支持自定义配置几种连接状态的超时时间，但这对`TIME_WAIT`状态无效。

Issue 81308 讨论 IVPS 的优雅结束是否不考虑不活跃的连接 (包括`TIME_WAIT`状态的连接)，也就是只考虑活跃连接，当活跃连接数为 0 之后立即踢掉 rs。这个确实可以更快的踢掉 rs，但无法让优雅结束做到那么优雅了，并且有人测试了，即便是不考虑不活跃连接，当请求量很大，还是不能很快踢掉 rs，因为源端口复用还是会导致不断有新的连接占用旧的连接，在较新的内核版本，`SYN_RECV`状态也被视为活跃连接，所以活跃连接数还是不会很快降到 0。





## Kubernetes 疑难杂症排查分享：神秘的溢出与丢包



### **问题描述**

有用户反馈大量图片加载不出来。

图片下载走的 k8s ingress，这个 ingress 路径对应后端 service 是一个代理静态图片文件的 nginx deployment，这个 deployment 只有一个副本，静态文件存储在 nfs 上，nginx 通过挂载 nfs 来读取静态文件来提供图片下载服务，所以调用链是：

```
client –> k8s ingress –> nginx –> nfs
```



### **猜测**

猜测: ingress 图片下载路径对应的后端服务出问题了。

验证：在 k8s 集群直接 curl nginx 的 pod ip，发现不通，果然是后端服务的问题！



### **抓包**

继续抓包测试观察，登上 nginx pod 所在节点，进入容器的 netns 中：

```sh
# 拿到 pod 中 nginx 的容器 id
$ kubectl describe pod tcpbench-6484d4b457-847gl | grep -A10 "^Containers:" | grep -Eo 'docker://.*$' | head -n 1 | sed 's/docker:\/\/\(.*\)$/\1/'
49b4135534dae77ce5151c6c7db4d528f05b69b0c6f8b9dd037ec4e7043c113e

# 通过容器 id 拿到 nginx 进程 pid
$ docker inspect -f {{.State.Pid}} 49b4135534dae77ce5151c6c7db4d528f05b69b0c6f8b9dd037ec4e7043c113e
3985

# 进入 nginx 进程所在的 netns
$ nsenter -n -t 3985

# 查看容器 netns 中的网卡信息，确认下
$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
3: eth0@if11: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 56:04:c7:28:b0:3c brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.26.0.8/26 scope global eth0
       valid_lft forever preferred_lft foreve
```

使用 tcpdump 指定端口 24568 抓容器 netns 中 eth0 网卡的包:

```sh
$ tcpdump -i eth0 -nnnn -ttt port 24568
```

在其它节点准备使用 nc 指定源端口为 24568 向容器发包：

```sh
$ nc -u 24568 172.16.1.21 80
```

观察抓包结果：

```sh
00:00:00.000000 IP 10.0.0.3.24568 > 172.16.1.21.80: Flags [S], seq 416500297, win 29200, options [mss 1424,sackOK,TS val 3000206334 ecr 0,nop,wscale 9], length 0
00:00:01.032218 IP 10.0.0.3.24568 > 172.16.1.21.80: Flags [S], seq 416500297, win 29200, options [mss 1424,sackOK,TS val 3000207366 ecr 0,nop,wscale 9], length 0
00:00:02.011962 IP 10.0.0.3.24568 > 172.16.1.21.80: Flags [S], seq 416500297, win 29200, options [mss 1424,sackOK,TS val 3000209378 ecr 0,nop,wscale 9], length 0
00:00:04.127943 IP 10.0.0.3.24568 > 172.16.1.21.80: Flags [S], seq 416500297, win 29200, options [mss 1424,sackOK,TS val 3000213506 ecr 0,nop,wscale 9], length 0
00:00:08.192056 IP 10.0.0.3.24568 > 172.16.1.21.80: Flags [S], seq 416500297, win 29200, options [mss 1424,sackOK,TS val 3000221698 ecr 0,nop,wscale 9], length 0
00:00:16.127983 IP 10.0.0.3.24568 > 172.16.1.21.80: Flags [S], seq 416500297, win 29200, options [mss 1424,sackOK,TS val 3000237826 ecr 0,nop,wscale 9], length 0
00:00:33.791988 IP 10.0.0.3.24568 > 172.16.1.21.80: Flags [S], seq 416500297, win 29200, options [mss 1424,sackOK,TS val 3000271618 ecr 0,nop,wscale 9], length 
```

SYN 包到容器内网卡了，但容器没回 ACK，像是报文到达容器内的网卡后就被丢了。看样子跟防火墙应该也没什么关系，也检查了容器 netns 内的 iptables 规则，是空的，没问题。

排除是 iptables 规则问题，在容器 netns 中使用`netstat -s`检查下是否有丢包统计:

```sh
$ netstat -s | grep -E 'overflow|drop'
    12178939 times the listen queue of a socket overflowed
    12247395 SYNs to LISTEN sockets dropped
```

果然有丢包，为了理解这里的丢包统计数据，我深入研究了一下，下面插播一些相关知识。



### **半连接与全连接队列**

Linux 进程监听端口时，内核会给它对应的 socket 分配两个队列：

- `syn queue`: 半连接队列。server 收到 SYN 后，连接会先进入 SYN_RCVD 状态，并放入 syn queue，此队列的包对应还没有完全建立好的连接（TCP 三次握手还没完成）。
- `accept queue`: 全连接队列。当 TCP 三次握手完成之后，连接会进入 ESTABELISHED 状态并从 syn queue 移到 accept queue，等待被进程调用 `accept()` 系统调用 “拿走”。

> 注意：这两个队列的连接都还没有真正被应用层接收到，当进程调用 `accept()` 后，连接才会被应用层处理，具体到我们这个问题的场景就是 nginx 处理 HTTP 请求。

为了更好理解，可以看下这张 TCP 连接建立过程的示意图：

![image-20210111161009002](D:\学习资料\笔记\k8s\k8s图\image-20210111161009002.png)



### **listen 与 accept**

不管使用什么语言和框架，在写 server 端应用时，它们的底层在监听端口时最终都会调用`listen()`系统调用，处理新请求时都会先调用`accept()`系统调用来获取新的连接，然后再处理请求，只是有各自不同的封装而已，以 go 语言为例：

```go
// 调用 listen 监听端口
l, err := net.Listen("tcp", ":80")
if err != nil {
  panic(err)
}
for {
  // 不断调用 accept 获取新连接，如果 accept queue 为空就一直阻塞
  conn, err := l.Accept()
  if err != nil {
    log.Println("accept error:", err)
    continue
    }
  // 每来一个新连接意味着一个新请求，启动协程处理请求
  go handle(conn)
}
```



### **Linux 的 backlog**

内核既然给监听端口的 socket 分配了 syn queue 与 accept queue 两个队列，那它们有大小限制吗？可以无限往里面塞数据吗？当然不行！资源是有限的，尤其是在内核态，所以需要限制一下这两个队列的大小。

那么它们的大小是如何确定的呢？我们先来看下 listen 这个系统调用:

```c
int listen(int sockfd, int backlog)
```

可以看到，能够传入一个整数类型的 backlog 参数，我们再通过`man listen`看下解释：

> ```
> The behavior of the backlog argument on TCP sockets changed with Linux 2.2. Now it specifies the queue length for completely established sockets waiting to be accepted, instead of the number of incomplete connection requests. The maximum length of the queue for incomplete sockets can be set using /proc/sys/net/ipv4/tcp_max_syn_backlog. When syncookies are enabled there is no logical maximum length and this setting is ignored. See tcp(7) for more information.
> If the backlog argument is greater than the value in /proc/sys/net/core/somaxconn, then it is silently truncated to that value; the default value in this file is 128. In kernels before 2.4.25, this limit was a hard coded value, SOMAXCONN, with the value 128.
> ```



继续深挖了一下源码，结合这里的解释提炼一下：

- listen 的 backlog 参数同时指定了 socket 的 syn queue 与 accept queue 大小。
- accept queue 最大不能超过`net.core.somaxconn`的值，即:

```c
max accept queue size = min(backlog, net.core.somaxconn)
```

- 如果启用了 syncookies (`net.ipv4.tcp_syncookies=1`)，当 syn queue 满了，server 还是可以继续接收 SYN 包并回复 SYN+ACK 给 client，只是不会存入 syn queue 了。因为会利用一套巧妙的 syncookies 算法机制生成隐藏信息写入响应的 SYN+ACK 包中，等 client 回 ACK 时，server 再利用 syncookies 算法校验报文，校验通过后三次握手就顺利完成了。所以如果启用了 syncookies，syn queue 的逻辑大小是没有限制的，
- syncookies 通常都是启用了的，所以一般不用担心 syn queue 满了导致丢包。syncookies 是为了防止 SYN Flood 攻击 (一种常见的 DDoS 方式)，攻击原理就是 client 不断发 SYN 包但不回最后的 ACK，填满 server 的 syn queue 从而无法建立新连接，导致 server 拒绝服务。
- 如果 syncookies 没有启用，syn queue 的大小就有限制，除了跟 accept queue 一样受`net.core.somaxconn`大小限制之外，还会受到`net.ipv4.tcp_max_syn_backlog`的限制，即:

```c
max syn queue size = min(backlog, net.core.somaxconn, net.ipv4.tcp_max_syn_backlog)
```

4.3 及其之前版本的内核，syn queue 的大小计算方式跟现在新版内核这里还不一样，详细请参考 commit `ef547f2ac16b`



### **队列溢出**

毫无疑问，在队列大小有限制的情况下，如果队列满了，再有新连接过来肯定就有问题。

翻下 linux 源码，看下处理 SYN 包的部分，在`net/ipv4/tcp_input.c`的`tcp_conn_request`函数:

```c
if ((net->ipv4.sysctl_tcp_syncookies == 2 ||
     inet_csk_reqsk_queue_is_full(sk)) && !isn) {
  want_cookie = tcp_syn_flood_action(sk, rsk_ops->slab_name);
  if (!want_cookie)
    goto drop;
}

if (sk_acceptq_is_full(sk)) {
  NET_INC_STATS(sock_net(sk), LINUX_MIB_LISTENOVERFLOWS);
  goto drop;
}
```

`goto drop`最终会走到`tcp_listendrop`函数，实际上就是将 `ListenDrops` 计数器 +1:

 ```c
static inline void tcp_listendrop(const struct sock *sk)
{
  atomic_inc(&((struct sock *)sk)->sk_drops);
  __NET_INC_STATS(sock_net(sk), LINUX_MIB_LISTENDROPS);
}
 ```

大致可以看出来，对于 SYN 包：

- 如果 syn queue 满了并且没有开启 syncookies 就丢包，并将`ListenDrops`计数器 +1。
- 如果 accept queue 满了也会丢包，并将`ListenOverflows`和`ListenDrops`计数器 +1。

而我们前面排查问题通过`netstat -s`看到的丢包统计，其实就是对应的`ListenOverflows`和`ListenDrops`这两个计数器。

除了用`netstat -s`，还可以使用`nstat -az`直接看系统内各个计数器的值:

```sh
$ nstat -az | grep -E 'TcpExtListenOverflows|TcpExtListenDrops'
TcpExtListenOverflows           12178939              0.0
TcpExtListenDrops               12247395              0.0
```

另外，对于低版本内核，当 accept queue 满了，并不会完全丢弃 SYN 包，而是对 SYN 限速。把内核源码切到 3.10 版本，看`net/ipv4/tcp_ipv4.c`中`tcp_v4_conn_request`函数:

```c
/* Accept backlog is full. If we have already queued enough
 * of warm entries in syn queue, drop request. It is better than
 * clogging syn queue with openreqs with exponentially increasing
 * timeout.
 */
if (sk_acceptq_is_full(sk) && inet_csk_reqsk_queue_young(sk) > 1) {
        NET_INC_STATS_BH(sock_net(sk), LINUX_MIB_LISTENOVERFLOWS);
        goto drop;
}
```

其中`inet_csk_reqsk_queue_young(sk) > 1`的条件实际就是用于限速，仿佛在对 client 说: 哥们，你慢点！我的 accept queue 都满了，即便咱们握手成功，连接也可能放不进去呀。



### **回到问题上来**

总结之前观察到两个现象：

- 容器内抓包发现收到 client 的 SYN，但 nginx 没回包。
- 通过`netstat -s`发现有溢出和丢包的统计 (`ListenOverflows`与`ListenDrops`)。

根据之前的分析，我们可以推测是 syn queue 或 accept queue 满了。

先检查下 syncookies 配置:

```sh
$ cat /proc/sys/net/ipv4/tcp_syncookies
1
```

确认启用了 syncookies，所以 syn queue 大小没有限制，不会因为 syn queue 满而丢包，并且即便没开启 syncookies，syn queue 有大小限制，队列满了也不会使`ListenOverflows`计数器 +1。

从计数器结果来看，`ListenOverflows`和`ListenDrops`的值差别不大，所以推测很有可能是 accept queue 满了，因为当 accept queue 满了会丢 SYN 包，并且同时将 `ListenOverflows`与`ListenDrops`计数器分别 +1。

如何验证 accept queue 满了呢？可以在容器的 netns 中执行`ss -lnt`看下:

```sh
$ ss -lnt
State      Recv-Q Send-Q Local Address:Port                Peer Address:Port
LISTEN     129    128                *:80                             *:*
```

通过这条命令我们可以看到当前 netns 中监听 tcp 80 端口的 socket，`Send-Q` 为 128，`Recv-Q` 为 129。

什么意思呢？通过调研得知：

- 对于 `LISTEN` 状态，`Send-Q` 表示 accept queue 的最大限制大小，Recv-Q 表示其实际大小。
- 对于 `ESTABELISHED` 状态，`Send-Q` 和 `Recv-Q` 分别表示发送和接收数据包的 buffer。

所以，看这里输出结果可以得知 accept queue 满了，当 `Recv-Q` 的值比` Send-Q` 大 1 时表明 accept queue 溢出了，如果再收到 SYN 包就会丢弃掉。

导致 accept queue 满的原因一般都是因为进程调用 `accept()` 太慢了，导致大量连接不能被及时 “拿走”。

那么什么情况下进程调用 `accept()` 会很慢呢？猜测可能是进程连接负载高，处理不过来。

而负载高不仅可能是 CPU 繁忙导致，还可能是 IO 慢导致，当文件 IO 慢时就会有很多 `IO WAIT`，在 `IO WAIT` 时虽然 CPU 不怎么干活，但也会占据 CPU 时间片，影响 CPU 干其它活。

最终进一步定位发现是 nginx pod 挂载的 nfs 服务对应的 nfs server 负载较高，导致 IO 延时较大，从而使 nginx 调用 `accept()` 变慢，accept queue 溢出，使得大量代理静态图片文件的请求被丢弃，也就导致很多图片加载不出来。

虽然根因不是 k8s 导致的问题，但也从中挖出一些在高并发场景下值得优化的点，请继续往下看。



### **somaxconn 的默认值很小**

我们再看下之前 ss -lnt 的输出:

 ```sh
$ ss -lnt
State      Recv-Q Send-Q Local Address:Port                Peer Address:Port
LISTEN     129    128                *:80                             *:*
 ```

仔细一看，`Send-Q` 表示 accept queue 最大的大小，才 128 ？也太小了吧！

根据前面的介绍我们知道，accept queue 的最大大小会受 `net.core.somaxconn`内核参数的限制，我们看下 pod 所在节点上这个内核参数的大小:

```sh
$ cat /proc/sys/net/core/somaxconn
32768
```

是 32768，挺大的，为什么这里 accept queue 最大大小就只有 128 了呢？

`net.core.somaxconn` 这个内核参数是 namespace 隔离了的，我们在容器 netns 中再确认了下：

```sh
$ cat /proc/sys/net/core/somaxconn
128
```

为什么只有 128？看下 stackoverflow 这里的讨论:

```
The "net/core" subsys is registered per network namespace. And the initial value for somaxconn is set to 128.
```

原来新建的 netns 中 `somaxconn` 默认就为 128，在 `include/linux/socket.h` 中可以看到这个常量的定义:

```c
/* Maximum queue length specifiable by listen.  */
#define SOMAXCONN  128
```

很多人在使用 k8s 时都没太在意这个参数，为什么大家平常在较高并发下也没发现有问题呢？

因为通常进程 `accept()` 都是很快的，所以一般 accept queue 基本都没什么积压的数据，也就不会溢出导致丢包了。

对于并发量很高的应用，还是建议将 `somaxconn` 调高。虽然可以进入容器 netns 后使用 `sysctl -w net.core.somaxconn=1024` 或 `echo 1024 > /proc/sys/net/core/somaxconn` 临时调整，但调整的意义不大，因为容器内的进程一般在启动的时候才会调用 `listen()`，然后 accept queue 的大小就被决定了，并且不再改变。



### 下面介绍几种调整方式:

#### **方式一: 使用 k8s sysctls 特性直接给 pod 指定内核参数**

示例 yaml:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sysctl-example
spec:
  securityContext:
    sysctls:
    - name: net.core.somaxconn
      value: "8096"
```

有些参数是 unsafe 类型的，不同环境不一样，我的环境里是可以直接设置 pod 的 net.core.somaxconn 这个 sysctl 的。如果你的环境不行，请参考官方文档 **Using sysctls in a Kubernetes Cluster** 启用 unsafe 类型的 sysctl。

> 注：此特性在 k8s v1.12 beta，默认开启。



#### **方式二: 使用 initContainers 设置内核参数**

示例 yaml:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sysctl-example-init
spec:
  initContainers:
  - image: busybox
    command:
    - sh
    - -c
    - echo 1024 > /proc/sys/net/core/somaxconn
    imagePullPolicy: Always
    name: setsysctl
    securityContext:
      privileged: true
  Containers:
  ...
```

> 注: init container 需要 privileged 权限。



#### **方式三: 安装 tuning CNI 插件统一设置 sysctl**

tuning plugin 地址: **https://github.com/containernetworking/plugins/tree/master/plugins/meta/tuning**

CNI 配置示例:

```json
{
  "name": "mytuning",
  "type": "tuning",
  "sysctl": {
          "net.core.somaxconn": "1024"
  }
}
```



### **nginx 的 backlog**

我们使用方式一尝试给 nginx pod 的 `somaxconn` 调高到 8096 后观察:

```sh
$ ss -lnt
State      Recv-Q Send-Q Local Address:Port                Peer Address:Port
LISTEN     512    511                *:80                             *:*
```

WTF? 还是溢出了，而且调高了 `somaxconn` 之后虽然 accept queue 的最大大小 (`Send-Q`) 变大了，但跟 8096 还差很远呀！
在经过一番研究，发现 nginx 在 `listen()` 时并没有读取 `somaxconn` 作为 backlog 默认值传入，它有自己的默认值，也支持在配置里改。通过`ngx_http_core_module` 的官方文档我们可以看到它在 linux 下的默认值就是 511:

```nginx
backlog=number
   sets the backlog parameter in the listen() call that limits the maximum length for the queue of pending connections. By default, backlog is set to -1 on FreeBSD, DragonFly BSD, and macOS, and to 511 on other platforms.
```

配置示例:

 ```nginx
listen  80  default  backlog=1024;
 ```

所以，在容器中使用 nginx 来支撑高并发的业务时，记得要同时调整下`net.core.somaxconn`内核参数和 `nginx.conf` 中的 backlog 配置。





## K8S 中的 CPUThrottlingHigh 到底是个什么鬼？



### **1. 前言**

在使用 Kubernetes 的过程中，我们看到过这样一个告警信息：

> [K8S] 告警主题: CPUThrottlingHigh
> 告警级别: warning
> 告警类型: CPUThrottlingHigh
> 故障实例: 
> 告警详情: 27% throttling of CPU in namespace kube-system for container kube-proxy in pod kube-proxy-9pj9j.
> 触发时间: 2020-05-08 17:34:17

这个告警信息说明 kube-proxy 容器被 throttling 了，然而查看该容器的资源使用历史信息，发现该容器以及容器所在的节点的 CPU 资源使用率都不高：

![image-20210111215022090](D:\学习资料\笔记\k8s\k8s图\image-20210111215022090.png)

经过我们的分析，发现该告警实际上是和 Kubernetes 对于 CPU 资源的限制和管控机制有关。Kubernetes 依赖于容器的 runtime 进行 CPU 资源的调度，而容器 runtime 以 Docker 为例，是借助于 cgroup 和 CFS 调度机制进行资源管控。本文基于这个告警案例，首先分析了 CFS 的基本原理，然后对于 Kubernetes 借助 CFS 进行 CPU 资源的调度和管控方法进行了介绍，最后使用一个例子来分析 CFS 的一些调度特性来解释这个告警的 root cause 和解决方案。



### 2. CFS 基本原理

#### 基本原理

Linux 在 2.6.23 之后开始引入 CFS 逐步替代 O1 调度器作为新的进程调度器，正如它名字所描述的，**CFS(Completely Fair Scheduler) 调度器**[1]追求的是对所有进程的全面公平，实际上它的做法就是在一个特定的调度周期内，保证所有待调度的进程都能被执行一遍，主要和当前已经占用的 CPU 时间经权重除权之后的值 (vruntime，见下面公式) 来决定本轮调度周期内所能占用的 CPU 时间，vruntime 越少，本轮能占用的 CPU 时间越多；总体而言，CFS 就是通过保证各个进程 vruntime 的大小尽量一致来达到公平调度的效果：

> 进程的运行时间计算公式为:
> 进程运行时间  = 调度周期  * 进程权重  / 所有进程权重之和

> vruntime = 进程运行时间  * NICE_0_LOAD / 进程权重 = (调度周期  * 进程权重  / 所有进程总权重) * NICE_0_LOAD / 进程权重  = 调度周期  * NICE_0_LOAD / 所有进程总权重

通过上面两个公式，可以看到 vruntime 不是进程实际占用 CPU 的时间，而是剔除权重影响之后的 CPU 时间，这样所有进程在被调度决策的时候的依据是一致的，而实际占用 CPU 时间是经进程优先级权重放大的。这种方式使得系统的调度粒度更小来，更加适合高负载和多交互的场景。



#### Kernel 配置

在 kernel 文件系统中，可以通过调整如下几个参数来改变 CFS 的一些行为：

- `/proc/sys/kernel/sched_min_granularity_ns`，表示进程最少运行时间，防止频繁的切换，对于交互系统
- `/proc/sys/kernel/sched_nr_migrate`，在多 CPU 情况下进行负载均衡时，一次最多移动多少个进程到另一个 CPU 上
- `/proc/sys/kernel/sched_wakeup_granularity_ns`，表示进程被唤醒后至少应该运行的时间，这个数值越小，那么发生抢占的概率也就越高
- `/proc/sys/kernel/sched_latency_ns`，表示一个运行队列所有进程运行一次的时间长度 (正常情况下的队列调度周期，P)
- `sched_nr_latency`，这个参数是内核内部参数，无法直接设置，是通过 sched_latency_ns/sched_min_granularity_ns 这个公式计算出来的；在实际运行中，如果队列排队进程数  nr_running > sched_nr_latency，则调度周期就不是 sched_latency_ns，而是 P = sched_min_granularity_ns * nr_running，如果  nr_running <= sched_nr_latency，则  P = sched_latency_ns

在阿里云的 Kubernetes 节点上，这些参数配置如下：

```sh
$ cat /proc/sys/kernel/sched_min_granularity_ns
10000000
$ cat /proc/sys/kernel/sched_nr_migrate
32
$ cat /proc/sys/kernel/sched_wakeup_granularity_ns
15000000
$ cat /proc/sys/kernel/sched_latency_ns
24000000
```

可以算出来  sched_nr_latency = sched_latency_ns / sched_min_granularity_ns = 24000000 / 10000000 = 2.4

在阿里云普通的虚拟机上的参数如下：

```sh
$ cat /proc/sys/kernel/sched_min_granularity_ns
3000000
$ cat /proc/sys/kernel/sched_latency_ns
15000000
```

可以算出来  sched_nr_latency = sched_latency_ns / sched_min_granularity_ns = 15000000 / 3000000 = 5

而在普通的 CentOS Linux release 7.5.1804 (Core) 上的参数如下：

```sh
$ cat /proc/sys/kernel/sched_min_granularity_ns
3000000
$ cat /proc/sys/kernel/sched_nr_migrate
32
$ cat /proc/sys/kernel/sched_wakeup_granularity_ns
4000000
$ cat /proc/sys/kernel/sched_latency_ns
24000000
```

可以算出来  sched_nr_latency = sched_latency_ns / sched_min_granularity_ns = 24000000 / 3000000 = 8

可以看到，阿里云的 Kubernetes 节点设置了更长的最小执行时间，在进程队列稍有等待 (2.4) 的时候就开始保证每个进程的最小运行时间不少于 10 毫秒。



#### 运行和观察

部署这样一个 yaml POD：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  labels:
    app: busybox
spec:
  containers:
  - image: busybox
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
    command:
      - "/bin/sh"
      - "-c"
      - "while true; do sleep 10; done"
    imagePullPolicy: IfNotPresent
    name: busybox
  restartPolicy: Always
```

可以看到该容器内部的进程对应的 CPU 调度信息变化如下：

```sh
[root@k8s-node-04 ~]$ cat /proc/121133/sched
sh (121133, #threads: 1)
-------------------------------------------------------------------
se.exec_start                                :   20229360324.308323
se.vruntime                                  :             0.179610
se.sum_exec_runtime                          :            31.190620
se.nr_migrations                             :                   12
nr_switches                                  :                   79
nr_voluntary_switches                        :                   78
nr_involuntary_switches                      :                    1
se.load.weight                               :                 1024
policy                                       :                    0
prio                                         :                  120
clock-delta                                  :                   26
mm->numa_scan_seq                            :                    0
numa_migrations, 0
numa_faults_memory, 0, 0, 0, 0, -1
numa_faults_memory, 1, 0, 0, 0, -1
numa_faults_memory, 0, 1, 1, 0, -1
numa_faults_memory, 1, 1, 0, 0, -1


[root@k8s-node-04 ~]$ cat /proc/121133/sched
sh (121133, #threads: 1)
-------------------------------------------------------------------
se.exec_start                                :   20229480327.896307
se.vruntime                                  :             0.149504
se.sum_exec_runtime                          :            33.325310
se.nr_migrations                             :                   17
nr_switches                                  :                   91
nr_voluntary_switches                        :                   90
nr_involuntary_switches                      :                    1
se.load.weight                               :                 1024
policy                                       :                    0
prio                                         :                  120
clock-delta                                  :                   31
mm->numa_scan_seq                            :                    0
numa_migrations, 0
numa_faults_memory, 0, 0, 1, 0, -1
numa_faults_memory, 1, 0, 0, 0, -1
numa_faults_memory, 0, 1, 0, 0, -1
numa_faults_memory, 1, 1, 0, 0, -1


[root@k8s-node-04 ~]$ cat /proc/121133/sched
sh (121133, #threads: 1)
-------------------------------------------------------------------
se.exec_start                                :   20229520328.862396
se.vruntime                                  :             1.531536
se.sum_exec_runtime                          :            34.053116
se.nr_migrations                             :                   18
nr_switches                                  :                   95
nr_voluntary_switches                        :                   94
nr_involuntary_switches                      :                    1
se.load.weight                               :                 1024
policy                                       :                    0
prio                                         :                  120
clock-delta                                  :                   34
mm->numa_scan_seq                            :                    0
numa_migrations, 0
numa_faults_memory, 0, 0, 0, 0, -1
numa_faults_memory, 1, 0, 0, 0, -1
numa_faults_memory, 0, 1, 1, 0, -1
numa_faults_memory, 1, 1, 0, 0, -1
```

其中 sum_exec_runtime 表示实际运行的物理时间。



### 3. Kubernetes 借助 CFS 进行 CPU 管理

#### CFS 进行 CPU 资源限流 (throtting) 的原理

根据文章**《Kubernetes 生产实践系列之三十：Kubernetes 基础技术之集群计算资源管理》**[2]的描述，Kubernetes 的资源定义：

```yaml
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

比如里面的 CPU 需求，会被翻译成容器 runtime 的运行时参数，并最终变成 cgroups 和 CFS 的参数配置：

```sh
$ cat cpu.shares
256
$ cat cpu.cfs_quota_us
50000
$ cat cpu.cfs_period_us
100000
```

这里有一个默认的参数：

```sh
$ cat /proc/sys/kernel/sched_latency_ns
24000000
```

所以在这个节点上，正常压力下，系统的 CFS 调度周期是 24ms，CFS 重分配周期是 100ms，而该 POD 在一个重分配周期最多占用 50ms 的时间，在有压力的情况下，POD 可以占据的 CPU share 比例是 256。

下面一个例子可以说明不同资源需求的 POD 容器是如何在 CFS 的调度下占用 CPU 资源的：

![image-20210111220930224](D:\学习资料\笔记\k8s\k8s图\image-20210111220930224.png)

CPU 资源配置和 CFS 调度

在这个例子中，有如下系统配置情况：

- CFS 调度周期为 10ms，正常负载情况下，进程 ready 队列里面的进程在每 10ms 的间隔内都会保证被执行一次
- CFS 重分配周期为 100ms，用于保证一个进程的 limits 设置会被反映在每 100ms 的重分配周期内可以占用的 CPU 时间数，在多核系统中，limit 最大值可以是 CFS 重分配周期 * CPU 核数
- 该执行进程队列只有进程 A 和进程 B 两个进程
- 进程 A 和 B 定义的 CPU share 占用都一样，所以在系统资源紧张的时候可以保证 A 和 B 进程都可以占用可用 CPU 资源的一半
- 定义的 CFS 重分配周期都是 100ms
- 进程 A 在 100ms 内最多占用 50ms，进程 B 在 100ms 内最多占用 20ms

所以在一个 CFS 重分配周期 (相当于 10 个 CFS 调度周期) 内，进程队列的执行情况如下：

- 在前面的 4 个 CFS 调度周期内，进程 A 和 B 由于 share 值是一样的，所以每个 CFS 调度内 (10ms)，进程 A 和 B 都会占用 5ms
- 在第 4 个 CFS 调度周期结束的时候，在本 CFS 重分配周期内，进程 B 已经占用了 20ms，在剩下的 8 个 CFS 调度周期即 80ms 内，进程 B 都会被限流，一直到下一个 CFS 重分配周期内，进程 B 才可以继续占用 CPU
- 在第 5-7 这 3 个 CFS 调度周期内，由于进程 B 被限流，所以进程 A 可以完全拥有这 3 个 CFS 调度的 CPU 资源，占用 30ms 的执行时间，这样在本 CFS 重分配周期内，进程 A 已经占用了 50ms 的 CPU 时间，在后面剩下的 3 个 CFS 调度周期即后面的 30ms 内，进程 A 也会被限流，一直到下一个 CFS 重分配周期内，进程 A 才可以继续占用 CPU

如果进程被限流了，可以在如下的路径看到：

> cat /sys/fs/cgroup/cpu/kubepods/pod5326d6f4-789d-11ea-b093-fa163e23cb69/69336c973f9f414c3f9fdfbd90200b7083b35f4d54ce302a4f5fc330f2889846/cpu.stat
>
> nr_periods 14001693
>
> **nr_throttled 2160435**
>
> **throttled_time 570069950532853**



### 本文开头问题的原因分析

根据 3.1 描述的原理，很容易理解本文开通的告警信息的出现，是由于在某些特定的 CFS 重分配周期内，kube-proxy 的 CPU 占用率超过了给它分配的 limits，而参看 kube-proxy daemonset 的配置，确实它的 limits 配置只有 200ms，这就意味着在默认的 100ms 的 CFS 重调度周期内，它只能占用 20ms，所以在特定繁忙场景会有问题：

![image-20210111221308989](D:\学习资料\笔记\k8s\k8s图\image-20210111221308989.png)



```sh
$ cat cpu.shares
204
$ cat cpu.cfs_period_us
100000
$ cat cpu.cfs_quota_us
20000
```

注：这里 cpu.shares 的计算方法如下：200x1024/1000~=204

而这个问题的解决方案就是将 CPU limits 提高。

Zalando 公司有一个分享**《Optimizing Kubernetes Resource Requests/Limits for Cost-Efficiency and Latency / Henning Jacobs》**[3]很好的讲述了 CPU 资源管理的问题，可以参考，这个演讲的 **PPT 在这里**[4]可以找到。

更具体问题分析和讨论还可以参考如下文章：

- **CPUThrottlingHigh false positives #108**[5]
- **CFS quotas can lead to unnecessary throttling #67577**[6]
- **CFS Bandwidth Control**[7]
- **Overly aggressive CFS**[8]

其中《**Overly aggressive CFS**[9]》里面还有几个小实验可以帮助大家更好的认识到 CFS 进行 CPU 资源管控的特点：

![image-20210111221444240](D:\学习资料\笔记\k8s\k8s图\image-20210111221444240.png)



### 参考资料

[1]CFS(Completely Fair Scheduler) 调度器: *https://www.kernel.org/doc/Documentation/scheduler/sched-design-CFS.txt*

[2]《Kubernetes 生产实践系列之三十：Kubernetes 基础技术之集群计算资源管理》: *https://blog.csdn.net/cloudvtech/article/details/107634724*

[3]《Optimizing Kubernetes Resource Requests/Limits for Cost-Efficiency and Latency / Henning Jacobs》: *https://www.youtube.com/watch?v=eBChCFD9hfs*

[4]PPT 在这里: *https://www.slideshare.net/try_except_/optimizing-kubernetes-resource-requestslimits-for-costefficiency-and-latency-highload?from_action=save*

[5]CPUThrottlingHigh false positives #108: *https://github.com/kubernetes-monitoring/kubernetes-mixin/issues/108*

[6]CFS quotas can lead to unnecessary throttling #67577: *https://github.com/kubernetes/kubernetes/issues/67577*

[7]CFS Bandwidth Control: *https://www.kernel.org/doc/Documentation/scheduler/sched-bwc.txt*

[8]Overly aggressive CFS: *https://gist.github.com/bobrik/2030ff040fad360327a5fab7a09c4ff1*

[9]Overly aggressive CFS: *https://gist.github.com/bobrik/2030ff040fad360327a5fab7a09c4ff1*   

