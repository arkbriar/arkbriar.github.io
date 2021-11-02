---
title: "Docker and Kubernetes"
description: 
date: 2021-11-01T16:39:22+08:00
slug: docker-and-kubernetes
image: img/docker-vs-k8s.png
draft: true
categories: ["Container", "Cloud Native"]
tags: ["docker", "kubernetes", "cloud native", "container"]
---

过去十年间，云计算的技术得到了长足的发展，越来越多的人开始了解“云原生”技术。以著名的云原生计算基金会 CNCF（Cloud Native Computing Foundation）为首，各大企业和社区都开始发展云原生软件技术。关于云原生的定义各方都有着不同的见解，但云原生软件技术中绕不开的一个话题就是容器技术。而说到容器，想必各位都能想到两个的软件 -- Docker 和 Kubernetes。

由于 Docker 和 Kubernetes 在使用方式上的一些相似性，初次接触这两个技术的同学常常会搞混，但实际上它们并不是解决同一个问题的技术，也没有办法用其中一个替代另一个。通俗一点来说，Docker 解决的是容器技术直接相关的问题，而 Kubernetes 更多地关注在集群上调度和部署容器应用。在过去几年 Docker 公司也推出了 Docker Swarm 来和 Kubernetes 竞争，当然本文不会对比 Docker Swarm 和 Kubernetes，感兴趣的同学可以自己了解。

本文主要将简单介绍下这两种软件，帮助大家建立对云原生容器技术的一些基础认识。

## Docker 是什么？

Docker 是 Docker 公司 (Docker, Inc.) 在 2013 年发行的一款提供统一、易用的容器技术的软件，它使开发者能够轻松地打包、上传、下载、运行和调试容器应用。Docker 并不是历史上第一个提出或是使用容器技术的软件，但它极大地降低了容器技术的使用成本，因此一经发布就受到了很多人的喜爱，并且目前已经发展成了容器技术的事实标准。

### 容器是什么？

既然 Docker 是容器技术的代表，那容器又是个什么东西？Docker 自己对容器有一个解释：

> A container is a standard unit of software that packages up code and all its dependencies so the application runs quickly and reliably from one computing environment to another.

容器是一个打包了代码和它所有依赖的一个标准的软件单元，它可以使应用能够在不同的计算环境中快速且可靠地运行。

![What is container?[9]](img/container-what-is-container.png)

Docker 对于容器的定义有那么一点宽泛。相较于虚拟化技术，容器技术实际上是在**操作系统之上**能够提供的资源隔离能力，因此目前所谓容器都是针对**进程**（组）的。从技术角度出发可以想见，容器的资源隔离的能力必须包括计算（CPU）、内存、存储（磁盘）、网络等其他一系列资源。

大家都听说过 Linux 中的 cgroups（Control Groups），是今天容器技术使用的最典型的资源隔离技术。Cgroups 的前身是 Google 公司在 2006 年开始研发的 Process Containers 进程容器，它不仅能够在进程级别隔离资源，还提供了对资源使用的限制和审计功能。从 Linux 2.6.24 起，cgroups 正式合并进入 Linux 主干并成为其中的一个重要的子系统。

另外，Linux 还提供了命名空间（namespace）、OverlayFS 等技术用于提供独立的用户、进程、网络空间等和隔离文件系统等，从而使容器内的进程可以（近似）完全运行在一组独立的资源之上。

早些年 Windows 还是不支持原生的（进程级别的）容器运行时的，即在 Windows 上运行 Docker 容器化的 Windows 应用还是无法实现的。在 2015 年 Docker 联手一些大型公司开始制定平台无关的容器技术标准，而在次年 Docker 宣布可以在 Windows Server 2016 上运行原生的容器应用，Windows 继 Linux 之后成为第二个原生支持 Docker 的平台。

截止到当前，Docker 已经支持将容器化应用部署到 Linux、Windows 和 macOS 之上，并支持 x86-64、ARM、ppc64le 等不同的指令架构。

> 注1：容器和虚拟机有着本质的区别，一个是硬件级的虚拟化，另一个仅仅是操作系统级的。
>
> 注2：在 macOS 上 Docker 运行了一个 Linux 的虚拟机，实际部署的是 Linux 应用。

### libcontainer 和 runc

早期 Docker 使用了 LXC（LinuX Containers）来管理容器的运行，这个 LXC 是第一个在 Linux 系统上能够直接使用的容器管理器。自 2008 年 LXC 实现以来，不断有人尝试构建容器的技术栈，其中知名的包括 CloudFoundry 在 2011 年启动的 Warden 和 2013 年 Google 开源的 LMCTFY（Let Me Contain That For You），以及同样发布的 Docker。

后来 Docker 废弃了 LXC 并开发了一套自己的容器管理库，并命名为 libcontainer。群雄逐鹿之下，Google 也放弃了 LMCTFY 并将核心概念合并进入了 libcontainer。

现在 libcontainer 更名为 [runc](https://github.com/opencontainers/runc)，并由开放容器基金会（Open Container Foundation）负责运营和维护。

runc 是一个容器的入口和守护进程，负责容器的创建（设置 cgroups 和 namespace 等资源隔离，准备文件系统）、启动（运行容器内的第一个进程）和资源回收。因此在 Linux 上，当你使用 Docker 运行容器应用后，你总能在进程树中发现一些 runc 进程。

### 镜像

Docker 的成功不仅源自于它对容器使用的效率和规范化，它还提供了一个很重要的能力 -- 将软件打包成标准化的容器应用单元，也就是镜像。这使得人们可以轻易地制作、获取和运行一个标准化的应用，解放了无数人肉运维机器的 SRE 的日日夜夜和无数高喊着 “你的环境有问题吧” 的程序员的担忧。

Docker 提供了 Dockerfile 这样可重复的方式来标准化镜像的制作流程，并提供了 Docker Hub 这样的镜像发布和分发平台，使得镜像成为了容器世界软件流通的标准方式。

```Dockerfile
FROM busybox
RUN echo 'This is an echo from busybox.'
```

现在，我们使用 `docker run` 第一次运行一个容器应用时，它首先会从 Docker Hub（或是其他镜像仓库）上拉取镜像，然后再交给容器运行时运行。

## Kubernetes 是什么？

看到这里，相信各位对 Docker 应该有个大概的认知了，那我们可以来谈谈 Kubernetes 是什么了。在 Docker 掀起了容器的热潮之后，各式各样关于容器应用的需求和问题开始被提出。其中最热门的一类是容器应用的部署问题。

想象这样一个场景，我们希望能够部署一套服务，它包含了好几个组件，且组件之间有一定的依赖关系。比如现在大家喜闻乐见的千万并发互联网系统啥的，它们的架构里至少包含 web 服务、缓存、数据库等基本组件。Docker 解决了这些组件的镜像化、标准化的运行问题，它能够让你简单通过一两条指令就将他们运行起来，并设置好资源的限制。但是这些都仅限于当前的机器，且组件之间的关联关系和跨多台机器的部署需要开发者自己去考虑。当应用的规模开始逐渐增大，使用 Docker 容器来部署应用的难度也随之上升了。Kubernetes 也就是为此而生的。

Kubernetes，别名也叫 K8s（因为 K 和 s 中有 8 个字符），是一个旨在自动化容器化应用的部署、扩展和管理的开源软件系统。2014 年 Google 公司发起了 Kubernetes 项目，至今已经发布了数十个版本（目前最新为 v1.22.3），并成为了大规模容器应用调度和管理技术的事实标准，也成为了“云原生”技术的代名词。

Kubernetes 将容器化的概念又拔高了一寸，它试图用系统化的方法来辅助我们在机器集群上部署和管理复数的容器应用。Kubernetes 在容器的基础上还提供了许多能力：

+ 宿主机资源监测和管理
+ 应用自动水平/垂直扩缩容以及回滚
+ 应用健康检查和自愈
+ 服务发现
+ 集群内/外负载均衡
+ ...

### Kubernetes 系统组件

为了提供上述功能，Kubernetes 需要至少这些组件：

+ 全局容器调度器
+ 宿主机上的容器管理
+ 完善的 API 服务
+ 高可用的元数据存储

同时，Kubernetes 在设计中将它管理的容器都组织到一张集群统一的虚拟网络之下，从而方便集群内容器应用的通信。因此 Kubernetes 至少还需要在每个节点上有一个虚拟网络控制器。

![Components of Kubernetes[11]](img/components-of-kubernetes.png)

有了这些概念后，上图所示的 Kubernetes 系统组件图就很容易理解了：

+ etcd，一个基于 raft 的 KV 数据库，作为元数据存储；
+ api (API server)，统一的 API 服务接口；
+ sched (Scheduler)，全局容器调度器；
+ kubelet，部署在每个宿主机（节点）上的 agent，负责宿主机状态观测和宿主机上容器应用的管理和观测；
+ k-proxy (kube-proxy)，部署在每个宿主机（节点）上的 agent，负责在宿主机上维护 Kubernetes 集群内的虚拟网络的部分功能；
+ c-m (Controller Manager)，控制器管理器，负责各种资源的控制，比如节点（Node）、服务（Service）和 Pod 等；
+ c-c-m (Cloud Controller Manager) 云服务控制器管理器，负责将云资源集成进 Kubernetes，典型的例子比如 LoadBalancer 类型的服务经由它在云上创建一个负载均衡器；

自然，有虚拟网络怎们能缺少 DNS 服务。虽然上图没有展示，但基本上所有 Kubernetes 集群都会在系统组件中部署 coredns 来提供集群内网络的 DNS 服务。

### API -- 微小又巨大的差异

在使用 Kubernetes 创建容器应用时，通常需要我们编写一个 yaml 文件来描述应用：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```

接下来的事情和使用 Docker 十分相似 -- 同样使用 `kubectl create -f nginx.yaml` 来创建一个 nginx 容器应用，然后再使用 `kubectl get pod nginx` 来查看容器应用的运行状态。

当想要进入容器查看容器内的进程状态时，Kubernetes 贴心的提供了 `exec` 子命令让我们进入容器：

```bash
kubectl exec -it nginx -- bash
```

除了名称不同，简直和 Docker 提供的一模一样！

|   操作   |                  Docker 指令                  |                Kubernetes 指令                 |
| :------: | :-------------------------------------------: | :--------------------------------------------: |
|   创建   |     docker run --name nginx nginx:1.14.2      |          kubectl create -f nginx.yaml          |
|   查看   |                   docker ps                   |                kubectl get pods                |
|   删除   |                docker rm nginx                |            kubectl delete pod nginx            |
|   exec   |          docker exec -it nginx bash           |         kubectl exec -it nginx -- bash         |
|   日志   |               docker logs nginx               |               kubectl logs nginx               |
| 文件拷贝 | docker cp /path/to/file nginx:/path/to/target | kubectl cp /path/to/file nginx:/path/to/target |

这么相似的两组命令一度让我产生了 Kubernetes 和 Docker 其实是差不多的错觉。

但是，这里需要一个但是，Kubernetes 的 API 运作方式和 Docker 截然不同。在 Docker 中，`docker` 指令是以命令式的（imperative）方式执行的。换句话说，当 `docker run` 执行完成时，已经有一个容器被创建并运行起来了。这种方式符合人们一贯的直觉。而 Kubernetes 则是所谓的声明式（declarative）的，当 `kubectl create -f nginx.yaml` 执行完成时，仅仅只是声明了一个名为 nginx 的容器应用，翻译一下就是刚把这个应用的 yaml 写进 etcd 里，容器啥的还没个影呢。接下来 Kubernetes 系统才开始真正地运行，并自动进行的一系列工作，包括挑选节点、拉取镜像、启动容器、汇报状态等等。

声明式的 API 和自动化的设计理念非常相符：你只需要声明一个期望的状态，剩下的事情由系统自动完成。当然，声明式 API 的复杂程度远非命令式可比，也不是所有 API 都适合设计成声明式的，因此也有了现在所谓的“云原生 yaml 工程师”的戏称。

### 与云的结合

在之前的介绍中我们提到，Kubernetes 中可以部署一个云服务控制器管理器（c-c-m）来让 Kubernetes 尽情地使用云上的资源，这项特性使得 Kubernetes 成为了当之无愧的云原生基石。

在 Kubernetes 的基础资源中，天然支持使用云上资源的就有 Service、Volume 和 Persistent Volume 等，从它们的[定义](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)上你就可能看到和各个云的紧密联系，各个云服务厂商也可以通过实现标准的扩展接口来扩展 Kubernetes 的能力：

+ CRI (Container Runtime Interface)，容器运行时接口
+ CNI (Container Network Interface)，容器网络接口
+ CSI (Container Storage Interface)，容器存储接口

![Triton's Multi-Cloud Kubernetes [12]](img/multi-cloud-k8s.png)

通过与云的结合，Kubernetes 可以源源不断地获取计算、存储、网络等资源，甚至可以通过组合多个线上、线下的云资源，实现真正的多云互通。

## Docker 和 Kubernetes 的关系？

回到最初的话题，Docker 和 Kubernetes 是什么关系？相信各位已经能理解，两个技术其实解决的是不同的问题，但它们又是相辅相成、可以有机结合的。

+ Docker 使得我们便捷地开发、打包、传播、运行容器应用；
+ 而 Kubernetes 则让我们在大规模集群上部署和管理容器应用。

使用 Docker 和 Kubernetes 可以让你真正在生产级别实现应用的容器化，让你的软件交付更加的鲁棒，让管理更加的方便。

过往几年，关于 Docker 和 Kubernetes 也出过不少新闻，像 Kubernetes 将停用 Docker 作为运行时，实质上改变不了两个软件在各自领域的地位。

## 写在最后

介绍 Docker 和 Kubernetes 的文章数不胜数，本文所写不过覆盖其中万一。只希望大家在阅读之后能对 Docker 和 Kubernetes 以及背后的容器化思想有一个基本的了解，那文章的目的也就达到了。

最后，笔者水平有限，错漏之处还望各位不吝指出。

## 参考文献

[1] <https://www.sumologic.com/blog/kubernetes-vs-docker/>

[2] <https://www.whitesourcesoftware.com/resources/blog/docker-vs-kubernetes/>

[3] <https://www.docker.com/>

[4] <https://kubernetes.io/>

[5] <https://blog.aquasec.com/>

[6] <https://www.docker.com/get-started>

[7] <https://www.infoq.cn/article/docker-container-management-libcontainer-depth-analysis>

[8] <https://www.docker.com/resources/what-container>

[9] <https://www.docker.com/sites/default/files/d8/styles/large/public/2018-11/container-what-is-container.png>

[10] <https://en.wikipedia.org/wiki/Docker_(software)>

[11] <https://d33wubrfki0l68.cloudfront.net/2475489eaf20163ec0f54ddc1d92aa8d4c87c96b/e7c81/images/docs/components-of-kubernetes.svg>

[11] <https://kubernetes.io/docs/concepts/extend-kubernetes/operator/>

[12] <https://www.joyent.com/blog/triton-kubernetes-multicloud>

[13] <https://acloudguru.com/blog/engineering/kubernetes-is-deprecating-docker-what-you-need-to-know>
