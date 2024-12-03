**目录：**
一. Docker核心概念
二. Kubernetes是什么及架构
三. Kubernetes核心概念
# 一、Docker核心概念
![640](https://github.com/user-attachments/assets/bbba04d0-6aaa-49f4-91e0-2da197782903)
## 1、为什么是Docker
![2](https://github.com/user-attachments/assets/97aec6ce-7827-4ee8-b702-af601cbcadb4)
**虚拟机：**
基础设施（Infrastructure）。服务器，或者是云主机。
主操作系统（Host Operating System）。服务器上运行的操作系统
虚拟机管理系统（Hypervisor）。利用Hypervisor，可以在主操作系统之上运行多个不同的从操作系统。
从操作系统（Guest Operating System）。假设你需要运行3个相互隔离的应用，则需要使用Hypervisor启动3个从操作系统，也就是3个虚拟机。这些虚拟机都非常大，也许有700MB，这就意味着它们将占用2.1GB的磁盘空间。更糟糕的是，它们还会消耗很多CPU和内存。
各种依赖。每一个从操作系统都需要安装许多依赖。
应用。安装依赖之后，就可以在各个从操作系统分别运行应用了，这样各个应用就是相互隔离的。
**Docker：**
Docker守护进程（Docker Daemon）。Docker守护进程取代了Hypervisor，它是运行在操作系统之上的后台进程，负责管理Docker容器。
各种依赖。对于Docker，应用的所有依赖都打包在Docker镜像中，Docker容器是基于Docker镜像创建的。
应用。应用的源代码与它的依赖都打包在Docker镜像中，不同的应用需要不同的Docker镜像。不同的应用运行在不同的Docker容器中，它们是相互隔离的。
**对比虚拟机与Docker**
Docker守护进程可以直接与主操作系统进行通信，为各个Docker容器分配资源；它还可以将容器与主操作系统隔离，并将各个容器互相隔离。虚拟机启动需要数分钟，而Docker容器可以在数毫秒内启动。由于没有臃肿的从操作系统，Docker可以节省大量的磁盘空间以及其他系统资源。
![3](https://github.com/user-attachments/assets/4aa2e325-b951-474f-a91f-dd6f567c5454)
## 2、Docker 架构
![4](https://github.com/user-attachments/assets/208d538a-ddef-48d8-9676-5793ace57642)
![5](https://github.com/user-attachments/assets/3f44dcd8-5607-491b-b92d-9e0690f455ca)
Docker使用客户端 - 服务器（C/S）架构，使用远程API管理和创建Docker 容器。Docker 客户端与Docker 守护进程通信，后者负责构建、运行和分发Docker容器。
Docker客户端和守护进程可以在同一系统上运行，也可以将Docker客户端连接到远程Docker守护进程。Docker客户端和守护进程使用REST API，通过UNIX套接字或网络接口进行通信。
Client
客户端通过命令行或其他工具与守护进程通信，客户端会将这些命令发送给守护进程，然后执行这些命令。命令使用Docker API，Docker客户端可以与多个守护进程通信。
Docker daemon
Docker守护进程（docker daemon）监听Docker API请求并管理Docker对象，如镜像，容器，网络和卷。守护程序还可以与其他守护程序通信以管理Docker服务。
Docker Host
Docker Host是物理机或虚拟机，用于执行Docker守护进程的仓库。
Docker Registry
Docker仓库用于存储Docker镜像，可以是Docker Hub这种公共仓库，也可以是个人搭建的私有仓库。使用docker pull或docker run命令时，将从配置的仓库中提取所需的镜像。使用docker push命令时，镜像将被推送到配置的仓库。
Docker Image
Docker 镜像可以看作是一个特殊的文件系统，除了提供容器运行时所需的程序、库、资源、配置等文件外，还包含了一些为运行时准备的一些配置参数（如匿名卷、环境变量、用户等）。
镜像可以用来创建 Docker 容器，一个镜像可以创建很多容器。Docker 提供了一个很简单的机制来创建镜像或者更新现有的镜像，用户甚至可以直接从其他人那里下载一个已经做好的镜像来直接使用。
Docker Container
Docker 利用容器来运行应用。容器是从镜像创建的运行实例。它可以被启动、开始、停止、删除。每个容器都是相互隔离的、保证安全的平台。
可以把容器看做是一个简易版的 Linux 环境（包括root用户权限、进程空间、用户空间和网络空间等）和运行在其中的应用程序。
## 3、Docker CLI
![6](https://github.com/user-attachments/assets/535508ad-5895-44cb-8538-4eaa210271d7)
## 4、Dockerfile
![7](https://github.com/user-attachments/assets/840ed6cb-665a-4b1d-8c86-a01e6b7318a8)
## 5、Docker Compose
Compose 是用于定义和运行多容器 Docker 应用程序的工具。通过 Compose，您可以使用 YML 文件来配置应用程序需要的所有服务。然后，使用一个命令，就可以从 YML 文件配置中创建并启动所有服务。
![9](https://github.com/user-attachments/assets/2428a1a6-582d-4865-a98d-0a92e3a98df5)
## 6、Docker Swarm 集群管理
Swarm是Docker 引擎内置（原生）的集群管理和编排工具，它将 Docker 主机池转变为单个虚拟 Docker 主机。
![11](https://github.com/user-attachments/assets/735fd597-cdf1-40bf-89c1-6f9fc34ac1d3)
![12](https://github.com/user-attachments/assets/be3e2743-ac4e-41a3-b321-539c7522c09a)
Docker Swarm 适用于简单和快速开发至关重要的环境，而 Kubernetes 适合大中型集群运行复杂应用程序的环境。
两者不是竞争对手，各有利弊，因需选择。
# 二、Kubernetes是什么及架构
## 1、k8s是什么
先来一张Kubernetes官网的截图，可以看到，官方对Kubernetes的定义：Kubernetes（k8s）是一个自动化部署、扩展和管理容器化应用程序的开源系统。
![13](https://github.com/user-attachments/assets/68067279-c2dc-4186-95ba-3e2f51437e06)
Kubernetes 这个单词是希腊语，它的中文翻译是“舵手”或者“飞行员”。在一些常见的资料中也会看到“ks”这个词，也就是“k8s”，它是通过将8个字母“ubernete ”替换为“8”而导致的一个缩写。 Kubernetes 为什么要用“舵手”来命名呢？
![15](https://github.com/user-attachments/assets/85644976-9408-474e-a98e-7ac2c3371547)
这是一艘载着一堆集装箱的轮船，轮船在大海上运着集装箱奔波，把集装箱送到它们该去的地方。Container 这个英文单词也有另外的一个意思就是“集装箱”。Kubernetes 也就借着这个寓意，希望成为运送集装箱的一个轮船，来帮助我们管理这些集装箱，也就是管理这些容器。
这个就是为什么会选用 Kubernetes 这个词来代表这个项目的原因。更具体一点地来说：Kubernetes 是一个自动化的容器编排平台，它负责应用的部署、应用的弹性以及应用的管理。
## 2、k8s能做什么

- 服务的发现与负载的均衡 

- 容器的自动装箱，也会把它叫做 scheduling，就是“调度”，把一个容器放到一个集群的某一个机器上，Kubernetes会帮助我们去做存储的编排，让存储的声明周期与容器的生命周期建立连接

- 容器的自动化恢复。在一个集群中，经常会出现宿主机的问题，导致容器本身的不可用，Kubernetes会自动地对这些不可用的容器进行恢复

- 应用的自动发布与应用的回滚，以及与应用相关的配置密文的管理

- 对于 job 类型任务，Kubernetes可以去做批量的执行

- 为了让这个集群、这个应用更富有弹性，Kubernetes支持容器的水平伸缩
### 2.1调度
Kubernetes 可以把用户提交的容器放到 Kubernetes 管理的集群的某一台节点上去。Kubernetes 的调度器是执行这项能力的组件，它会观察正在被调度的这个容器的大小、规格。 
比如，容器所需要的CPU以及它所需要的内存，然后在集群中找一台相对比较空闲的机器来进行一次放置的操作。
![16](https://github.com/user-attachments/assets/aa6ac3ee-989e-4434-b6de-044a6d8deab9)
### 2.2 自动修复
Kubernetes 有节点健康检查的功能，它会监测这个集群中所有的宿主机，当宿主机本身出现故障，或者软件出现故障的时候，这个节点健康检查会自动对它进行发现。

接下来 Kubernetes 会把运行在这些失败节点上的容器进行自动迁移，迁移到一个正在健康运行的宿主机上，来完成集群内容器的自动恢复。
![17](https://github.com/user-attachments/assets/7bd1b01d-5e27-4913-bdc1-fdb344af3a8b)
### 2.3 水平伸缩
Kubernetes有业务负载检查的能力，它会监测业务上所承担的负载，如果这个业务本身的CPU利用率或内存占用过高，或者响应时间过长，它可以对这个业务进行一次扩容。

比如，下面的例子中，黄颜色的过度忙碌，Kubernetes就可以把黄颜色负载从一份变为三份。接下来，它就可以通过负载均衡把原来打到第一个黄颜色上的负载平均分到三个黄颜色的负载上去，以此来提高响应速度。
![18](https://github.com/user-attachments/assets/d8450b38-8dc2-4640-be41-14232caad4e5)
## 3、k8s的架构
Kubernetes 架构是一个比较典型的二层架构和server-client架构。Master作为中央管控节点，与Node建立连接。

所有 UI 的、clients、user侧的组件，只会和Master进行连接，把希望的状态或者想执行的命令下发给 Master，Master会把这些命令或者状态下发给相应的节点，进行最终的执行。
![19](https://github.com/user-attachments/assets/d4632209-033b-404e-b327-ea8cd8f253c4)

- Master
Kubernetes 的Master包含四个主要的组件：API Server、Controller、Scheduler以及etcd。
![20](https://github.com/user-attachments/assets/b0a0e9df-a9cf-435f-bdcb-019df2a87ec4)
**API Server**:提供了资源操作的唯一入口，并提供认证、授权、访问控制、API注册和发现等机制。
Kubernetes 中所有的组件都会和API Server进行连接，组件与组件之间一般不进行独立的连接，都依赖于API Server进行消息的传送；
**Controller**:控制器，它负责维护集群的状态，比如故障检测、自动扩展、滚动更新等。上面的2个例子，第1个自动对容器进行修复、第2个自动水平扩张，都是由Controller 完成的；
**Scheduler**:是调度器，负责资源的调度，按照预定的调度策略将Pod调度到相应的机器上。例如上面的例子，把用户提交的pod，依据它对CPU、memory请求的大小，找一台合适的节点，进行放置；
**etcd**:是一个分布式的存储系统，保存了整个集群的状态，比如Pod、Service等对象信息。API Server 中所需要的原信息都被放置在etcd中，etcd本身是一个高可用系统，通过etcd保证整个Kubernetes的Master组件的高可用性。

- Node
Kubernetes 的 Node 是真正运行业务负载的，每个业务负载会以 Pod 的形式运行。一个 Pod 中运行的一个或者多个容器。
**kubelet：**Master在Node节点上的Agent，是真正去运行 Pod 的组件，也是Node上最关键的组件，负责本Node节点上Pod的创建、修改、监控、删除等生命周期管理，同时Kubelet定时“上报”本Node的状态信息到API Server。
它通过 API Server 接收到所需要 Pod 运行的状态。然后提交到 Container Runtime 组件中。
![21](https://github.com/user-attachments/assets/af1c42f3-b18e-4387-964a-65a368e50db2)
**Container Runtime**:容器运行时。负责镜像管理以及Pod和容器的真正运行（CRI），可以理解为类似JVM
**Storage Plugin 或者 Network Plugin**:对存储跟网络进行管理

在 OS 上去创建容器所需要运行的环境，最终把容器或者 Pod 运行起来，也需要对存储跟网络进行管理。Kubernetes 并不会直接进行网络存储的操作，他们会靠 Storage Plugin 或者Network Plugin 来进行操作。用户自己或者云厂商都会去写相应的 Storage Plugin 或者 Network Plugin，去完成存储操作或网络操作。
**Kube-proxy**:负责为Service提供cluster内部的服务发现和负载均衡，完成 service 组网
在 Kubernetes 自己的环境中，也会有 Kubernetes 的 Network，它是为了提供 Service network 来进行搭网组网的。真正完成 service 组网的组件是 Kube-proxy，它是利用了 iptable 的能力来进行组建 Kubernetes 的 Network，就是 cluster network。
**组件间的通信** 
![22](https://github.com/user-attachments/assets/6b9254d6-ac92-4389-a1f5-380c26c95f47)
**步骤说明**:
1. 通过 UI 或者 CLI 提交1个 Pod 给 Kubernetes 进行部署，这个 Pod 请求首先会提交给API Server，下一步 API Server 会把这个信息写入到存储系统 etcd，之后 Scheduler 会通过 API Server 的 watch机制得到这个信息：有1个Pod 需要被调度。
2. Scheduler会根据node集群的内存状态进行1次调度决策，在完成这次调度之后，它会向 API Server 报告：“OK！这个 Pod 需要被调度到XX节点上。”
API Server 接收后，会把这次的操作结果再次写到 etcd 中。
3. API Server 通知相应的节点进行这个Pod真正的执行启动。相应节点的 kubelet 会得到通知，然后kubelet 会去调 Container runtime 来真正去启动配置这个容器和这个容器的运行环境，去调度 Storage Plugin 来去配置存储，network Plugin 去配置网络。
# 三、Kubernetes核心概念
**第一个概念: Pod**
Pod 是 Kubernetes 的最小调度以及资源单元。可以通过 Kubernetes 的 Pod API 生产一个 Pod，让 Kubernetes 对这个 Pod 进行调度，也就是把它放在某一个 Kubernetes 管理的节点上运行起来。**一个 Pod 简单来说是对一组容器的抽象，它里面会包含一个或多个容器。**
比如下图，它包含了两个容器，每个容器可以指定它所需要资源大小
当然，在这个 Pod 中也可以包含一些其他所需要的资源：比如说我们所看到的 Volume 卷这个存储资源。
![23](https://github.com/user-attachments/assets/f670b7af-910f-4e25-96bc-4492832f4efb)
**第二个概念：Volume**
管理 Kubernetes 存储，用来声明在 Pod 中的容器可以访问的文件目录，一个卷可以被挂载在 Pod 中一个或者多个容器的指定路径下面。
而 Volume 本身是一个抽象的概念，一个 Volume 可以去支持多种的后端的存储。Kubernetes 的 Volume 支持很多存储插件，可以支持本地的存储和分布式的存储，比如像 ceph，GlusterFS；也可以支持云存储，比如阿里云上的云盘、AWS 上的云盘、Google 上的云盘等等。
![24](https://github.com/user-attachments/assets/9fa4aa36-89de-4e00-b979-d1b9642c59ce)
**第三个概念：Deployment**
Deployment 是在Pod上更为上层的一个抽象，它可以定义一组Pod 的副本数目、以及Pod的版本。一般用Deployment来做应用的真正的管理，而Pod是组成Deployment最小的单元。
Kubernetes通过 Controller（控制器）维护Deployment中Pod 的数目，Controller也会去帮助Deployment自动恢复失败的Pod。
比如，可以定义一个Deployment，这个Deployment里面需要2个Pod，当1个Pod失败的时候，控制器就会监测到，再去新生成1个Pod，把Deployment中的Pod数目从1个恢复到2个。通过控制器，也可以完成发布策略，比如进行滚动升级、重新生成的升级或者进行版本回滚。
![25](https://github.com/user-attachments/assets/f286c8a7-a776-4acb-9010-11f43414b727)
**第四个概念：Service**
Service：**提供1个或者多个 Pod 实例的稳定访问地址**
比如，一个 Deployment 可能有2个甚至更多个完全相同的 Pod。对于外部的用户来讲，访问哪个 Pod 都是一样的，所以希望做一次负载均衡，在做负载均衡的同时，只需要访问某一个固定的 VIP，也就是 Virtual IP 地址，而不需要得知每一个具体的 Pod 的 IP 地址。
如果1个 Pod 失败了，可能会换成另外一个新的。提供了多个具体的 Pod 地址，对外部用户来说，要不停地去更新 Pod 地址。当这个 Pod 再失败重启之后，如果有一个抽象，把所有 Pod 的访问能力抽象成1个第三方的 IP 地址，实现这个的 Kubernetes 的抽象就叫 Service。
实现 Service 有多种入口方式： 

- ClusterIP：Service 在集群内的唯一 ip 地址，我们可以通过这个 ip，均衡的访问到后端的 Pod，而无须关心具体的 Pod。

- NodePort：Service 会在集群的每个 Node 上都启动一个端口，我们可以通过任意Node 的这个端口来访问到 Pod。

- LoadBalancer：在 NodePort 的基础上，借助公有云环境创建一个外部的负载均衡器，并将请求转发到 NodeIP:NodePort。

- ExternalName：将服务通过 DNS CNAME 记录方式转发到指定的域名（通过 spec.externlName 设定）。
![26](https://github.com/user-attachments/assets/94802be5-0038-42f6-ab28-48b8d3b80a52)
**第五个概念：Namespace**
Namespace：用来做一个集群内部的逻辑隔离，包括鉴权、资源管理等。Kubernetes 的每个资源，比如Pod、Deployment、Service 都属于一个 Namespace，同一个 Namespace 中的资源需要命名的唯一性，不同的 Namespace 中的资源可以重名。
![27](https://github.com/user-attachments/assets/a1512d9e-4635-4d40-8c77-23d7bdcdfd70)
**K8S的API**
Kubernetes API 是由 HTTP+JSON 组成的：用户访问的方式是HTTP，访问API 中 content 的内容是JSON格式的。
用Kubectl 命令、Kubernetes UI或者Curl，直接与Kubernetes交互都是使用 HTTP + JSON 的形式。
如下图，对于这个Pod类型的资源，它的**HTTP访问的路径就是 API**，apiVesion: V1, 之后是相应的Namespaces，以及Pods资源，最终是 Podname，也就是Pod的名字。
![28](https://github.com/user-attachments/assets/d1b7b7db-7e9a-443d-a0a2-b7d0a03135e3)
当提交一个 Pod，或者 get 一个 Pod 的时候，它的 content 内容都是用JSON 或者是YAML表达的。上图中YAML的例子，在这个YAML文件中，对Pod资源的描述分为几个部分。
第一个部分，一般是 API 的 version。比如在这个例子中是 V1，它也会描述我在操作哪个资源； kind 如果是 pod，在 Metadata 中，就写上这个 Pod 的名字；比如nginx。也会给pod打一些 label，在 Metadata 中，有时候也会去写 annotation，也就是对资源的额外的一些用户层次的描述。
比较重要的一个部分叫 Spec，Spec 也就是希望 Pod 达到的一个预期的状态。比如pod内部需要有哪些 container 被运行；这里是一个name为nginx 的 container，它的 image 是什么？它暴露的 port 是什么？
当从 Kubernetes API 中去获取这个资源的时候，一般在 Spec 下面会有一个status字段 ，它表达了这个资源当前的状态；比如一个 Pod 的状态可能是正在被调度、或者是已经 running、或者是已经被 terminates（被执行完毕）。
Label是一个比较有意思的 metadata，可以是一组KeyValue的集合。
如下图，第一个 pod 中，label 就可能是一个 color 等于 red，即它的颜色是红颜色。当然也可以加其他 label，比如说 size: big 就是大小，定义为大的，它可以是一组label。
这些 label 是可以被 selector（选择器）所查询的。就好比sql 类型的 select 语句。
![29](https://github.com/user-attachments/assets/7282d235-365f-48e2-84ba-b339b924de03)
通过label，kubernetes 的API层就可以对这些资源进行筛选。
例如，Deployment可能代表一组Pod，是一组Pod 的抽象，一组Pod就是通过label selector来表达的。当然Service对应的一组Pod来对它们进行统一的访问，这个描述也是通过label selector来选取的一组Pod。