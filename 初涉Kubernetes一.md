---
title: 初涉 Kubernetes(一)
date: 2017-06-23 15:56:15
tags: [Docker,应用编排,Kubernetes,调度算法]
categories: 云平台
---


## Overview

![官方给出的架构图](kube-architecture.png)

上一篇[《Kubeadm 搭建 Kubernetes 集群》](https://leonlibraries.github.io/2017/06/15/Kubeadm搭建Kubernetes集群/) 简单交代了 Kubernetes 的集群搭建方法，这一篇算是基于安装好的集群，做了一个简单的入门总结。


##  API Server

API Server 是 Kubernetes 非常核心的一个组件，主要的作用就是提供了 Kubernetes 各类资源对象（Service、Pod、RC、Deployment 等） 的增删改查以及 Watch 等 HTTP Rest 接口，成为集群内各个功能模块之间数据交互和通信的中心枢纽。是整个系统的数据总线和数据中心。它同样是集群管理的入口、资源配额控制的入口。既然这么重要，安全性肯定要保障，因此它也提供了较为完备的安全机制。

如何对他进行直接的访问呢？

* ``kubectl``：一个客户端工具，可以搭建在集群之外，安装和配置的过程可以参考上一篇文章；
* ``kubectl proxy``：一个代理出口，也可以搭建在集群之外，有自己的权限控制体系（RBAC），可以在里边部署可视化界面直接看到集群内部重要的元数据。

![Kubernetes 组件架构图](Kubernetes.001.png)

整个 Kubernetes 组件大致就是这样的，可以见得 API Server 起着枢纽的作用，元数据都会通过 API Server 存到 Etcd 集群之中。无论是 Controller Manager 还是 Scheduler，抑或是 Kubelet 都要直接与 API Server 进行数据对接。其中 Kubelet 的作用是将自身资源情况上报给 API Server 以及 Watch Pod 的创建以及删除，如果这个 Pod 与自己的 Node 有关，那么就要进行下一步的拉取镜像、创建容器或删除容器操作。

## Controller Manager

Controller Manager 是一个集群内部的管理控制中心，有一组控制器构成，这组控制器负责集群内部的 Node、Pod、Endpoint、Namespace、ServiceAccount、ResourceQuota 等等资源的管理。与之对应就有如下的控制器：

![Kubernetes Controller Manager](Kubernetes.002.png)

其中重点介绍几个比较重要的控制器：

* ``Replication Controller``：顾名思义，副本控制器，其作用是针对 RC 来说的（这里的 RC 可以是 ReplicationController、Deployment 以及 ReplicaSet，此处的 ReplicationController 为资源对象，并非指控制器），目的是保证 RC 指定的副本数量预期值与现实保持一致，意味着当副本数与预期不一致的时候，会发起重新调度的功能。并且滚动升级、弹性扩缩等功能都是交由它来实现。
> **tips**：这里的滚动升级是在版本更新的过程中保持旧版本 Pod 不会立马被删除，只有当新版本完全部署完毕后旧版本的 Pod 才会慢慢被删掉；弹性扩缩是根据其本身的资源监控（Heapster），默认是以 CPU 的使用率，超过指定的阀值后实现动态增加节点，反之低于阀值将会释放资源，缩减节点。

* ``Node Controller``：Kubelet 启动时，通过 API Server 注册自身的节点信息，并定时向 API Server 汇报，API Server 接收这些信息，将其更新到 etcd 之中。而 Node Controller 就是通过 API Server 获取到这些信息。
![Node 信息](nodestatus.jpeg)
当 Controller Manager 启动时指定了 ``--cluster-cidr`` 这个参数，那么每个节点都会自动分配一个与其他节点不一样的 Pod CIDR，也就意味着规定了该节点的网段，且不会与其他节点产生冲突。值得一提的是，当一个节点在很长一段时间内没有上报状态信息，Node Controller会认为这个节点已经死了，并删除节点将其踢出。

* ``ResourceQuota Controller``：资源配额控制器会将容器、Pod以及 Namespace 三个维度统计资源占用情况，并将统计结果汇总到 etcd 数据库中，每次向 API Server 创建资源的时候，会经过 Admission Control ，这里 Admission Control 有两种策略，LimitRanger 和 ResourceQuota，分别对应 Pod 与容器的资源 和 Namespaces 的资源限制。确保调度不会超过集群的资源规划以及物理资源限制。如果超出资源限制，就会报错。

* ``Service Controller`` 和 ``Endpoint Controller``：Service Controller 负责将 Pod 集合暴露成一个 Service ，并通过 Endpoint Controller 分配其一个虚拟的 Cluster IP（Endpoint 对象的一个属性）。Endpoint Controller 负责维护所有的 Endpoint 对象，其监听 Service 的创建和删除事件，并对应分配和删除与之对应的 Endpoint 对象。

> 值得一提的是， Cluster IP 指向的是一个微服务，其中可能包含了多个节点，节点访问的负载均衡是通过 kube-proxy 进程来实现的。后续会有详细的介绍。


## Scheduler

![调度器示意](Kubernetes.003.png)

Scheduler 也免不了要和 API Server 打交道，整个调度流程大致分为两个步骤：
* 预选调度过程，遍历所有 Node，筛选出符合要求的候选节点，这里给用户提供了许多预选策略；
* 优选节点，采用优先算法计算出每个候选节点的几分，积分高出者胜出。

这两种策略是每次调度都要执行的，具体的策略不做过多描述，可以参考这篇文档：
https://github.com/kubernetes/community/blob/master/contributors/devel/scheduler_algorithm.md

## Kubelet
集群中的每个 Node 都有 Kubelet 进程，该进程用于处理 Master 节点下发到本节点的任务，管理 Pod 以及 Pod 中的容器。

* 节点管理：kubelet 启动时向 API Server 注册节点信息，并定时向 API Server 汇报节点状况；
* Pod管理：创建/删除 Pod，下载容器镜像，用 Pause 创建容器，运行容器，校验容器是否正确等；
* 容器健康检查：通过``cat /tmp/health`` 以及访问容器的 HTTP 接口（HTTP 状态码作为判断依据）来判断容器是否健康；
* cAdvisor 资源监控：cAdvisor 集成到 kubelet 程序的代码之中，负责查找当前节点的容器，自动采集容器级别的 CPU、内存、文件系统和网络使用的统计信息。 Heapster 通过 kubelet 的 cAdvisor 拿到这些参数，推到后端的 InfluxDB（with Grafana）之中。

> cAdvisor 也是一个不错的容器级监控工具，用于排障有一定的意义

## kube-proxy：从 Service 说开去

上边有简单交代过 kube-proxy 的作用，这里展开说。

Pod 是用来挂载一个或一组容器的，应用的滚动升级以及弹性扩缩都是基于 Pod 作为最小单元来实现的，Pod 有自己的 IP 和端口，理论上来说，已经可以正常访问到容器内部程序了，但依旧存在不小的问题。

简单来说，Pods 确乎有自己的 IP 资源，然而是瞬时的，其资源以及 IP 都是不稳定的，也是容易产生变化的。以 Pod IP 做服务的基础 IP 的话显然会出现问题，毕竟服务调用方无法准确而及时的知道 Pod 是否发生了变化（这个变化包括滚动升级、弹性扩缩以及重新调度）。

为了解决上面说的问题，Kubernetes 抽象出了 Service 概念，而 Service 本身也就是对一组服务的抽象。
![Service 详情](kube-proxy.png)
Kubernetes 创建服务的时候，通常会给这个服务一个 Cluster IP，这个 IP 对后端一组服务的 Pod IP 实现了映射并且负载均衡。嗯，理解的没错，这个 Cluster IP 就是一个反向代理的 IP 地址。这个反向代理的关键，就是 kube-proxy 组件。

每个节点都会运行一个 kube-proxy 的进程，这个进程本身就是一个反向代理服务进程，其核心功能就是将 Service Endpoint 映射到后端的一组 Pod 里。这里默认采用 Round Robin 负载均衡算法，将流量轮流分发给后端的 Pod。当然这里也有一个可选项，对于某些状态未抽离的服务来说，会话保持是靠内存来实现的，Kubernetes 提供了通过修改 Service 的``service.spec.sessionAffinity``参数的值来实现会话保持特性的定向转发，假设设置为``ClientIP``，则将来自同一个 ClientIP的请求分发到同一个后端 Pod 上。

此外，Service 的 Cluster IP 与 NodePort 是 kube-proxy 服务通过 Iptables 的 DNAT 转换实现的。kube-proxy 在运行过程中动态创建与 Service 相关的 Iptables 规则，这些规则实现了 Cluster IP及 NodePort 的请求流量重定向到 kube-proxy 进程上对应服务的代理端口的功能。

综上，无论是 Cluster IP + target Port 还是 Node IP + Node Port的方式，都会被节点机的 Iptables 规则重定向到 kube-proxy 监听的 Service 服务代理端口中去。

![kube-proxy 分发逻辑](Kubernetes.004.png)


kube-proxy 负责 Iptables 的 DNAT 规则的维护，这个维护与 Service 的生命周期是息息相关的。kube-proxy 监听了 Service 的创建和删除的事件，实时维护这一套路由规则，当 Service 请求来临的时候，根据指定的规则转发到被改写后的 Pod IP 和 port。因此我们可以看到，Cluster IP 之所以称之为虚拟的 IP，是因为其仅仅是在 Iptables 路由表里维护，并非实际意义的 IP。这个做法相当巧妙！

我们来看下 sock-shop 微服务下 payment 的 Iptables 路由规则

```txt
[root@node2 ~]# iptables-save |grep payment
-A KUBE-SEP-OO2TWKZ7ZV6AUJIZ -s 10.244.1.13/32 -m comment --comment "sock-shop/payment:" -j KUBE-MARK-MASQ
-A KUBE-SEP-OO2TWKZ7ZV6AUJIZ -p tcp -m comment --comment "sock-shop/payment:" -m tcp -j DNAT --to-destination 10.244.1.13:80
-A KUBE-SEP-UV4LMV4IYDXHR55Y -s 10.244.3.2/32 -m comment --comment "sock-shop/payment:" -j KUBE-MARK-MASQ
-A KUBE-SEP-UV4LMV4IYDXHR55Y -p tcp -m comment --comment "sock-shop/payment:" -m tcp -j DNAT --to-destination 10.244.3.2:80
-A KUBE-SERVICES ! -s 10.244.0.0/16 -d 10.109.104.9/32 -p tcp -m comment --comment "sock-shop/payment: cluster IP" -m tcp --dport 80 -j KUBE-MARK-MASQ
-A KUBE-SERVICES -d 10.109.104.9/32 -p tcp -m comment --comment "sock-shop/payment: cluster IP" -m tcp --dport 80 -j KUBE-SVC-T7TCZ5NWHRXELJYN
-A KUBE-SVC-T7TCZ5NWHRXELJYN -m comment --comment "sock-shop/payment:" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-OO2TWKZ7ZV6AUJIZ
-A KUBE-SVC-T7TCZ5NWHRXELJYN -m comment --comment "sock-shop/payment:" -j KUBE-SEP-UV4LMV4IYDXHR55Y
```
其中 ``-j DNAT --to-destination 10.244.1.13:80``改写了 Packet 数据包的目的地址和端口，将其映射到后端的 Pod IP

至于改写之后的 IP 地址为什么就能路由到对应的 Pod 上呢？这个下一篇文章讨论。

```txt
[root@kafka4 ~]# iptables --table nat --list |grep payment
KUBE-MARK-MASQ  all  --  10.244.1.13          anywhere             /* sock-shop/payment: */
DNAT       tcp  --  anywhere             anywhere             /* sock-shop/payment: */ tcp to:10.244.1.13:80
KUBE-MARK-MASQ  all  --  10.244.3.2           anywhere             /* sock-shop/payment: */
DNAT       tcp  --  anywhere             anywhere             /* sock-shop/payment: */ tcp to:10.244.3.2:80
KUBE-MARK-MASQ  tcp  -- !10.244.0.0/16        10.109.104.9         /* sock-shop/payment: cluster IP */ tcp dpt:http
KUBE-SVC-T7TCZ5NWHRXELJYN  tcp  --  anywhere             10.109.104.9         /* sock-shop/payment: cluster IP */ tcp dpt:http
KUBE-SEP-OO2TWKZ7ZV6AUJIZ  all  --  anywhere             anywhere             /* sock-shop/payment: */ statistic mode random probability 0.50000000000
KUBE-SEP-UV4LMV4IYDXHR55Y  all  --  anywhere             anywhere             /* sock-shop/payment: */
```


对于使用 NodePort 的方式绑定 Node 上的端口，可以看这个前端容器的例子：
```txt
[root@kafka4 ~]# iptables-save |grep front-end
-A KUBE-NODEPORTS -p tcp -m comment --comment "sock-shop/front-end:" -m tcp --dport 30001 -j KUBE-MARK-MASQ
-A KUBE-NODEPORTS -p tcp -m comment --comment "sock-shop/front-end:" -m tcp --dport 30001 -j KUBE-SVC-LFMD53S3EZEAOUSJ
...省略后面内容...
```



## 小结

本篇对 Kubernetes 的介绍暂时就到这里，下一篇将具体研究 Kubernetes 的网络解决方案。


Further Reading
https://github.com/kubernetes/community/blob/master/contributors/devel/scheduler_algorithm.md
https://github.com/kubernetes/community/tree/master/contributors/design-proposals
https://github.com/kubernetes/community/blob/master/contributors/design-proposals/architecture.md
https://kubernetes.io/docs/concepts/services-networking/service/
《Kubernetes 权威指南：从 Docker 到 Kubernetes 实践全接触》
