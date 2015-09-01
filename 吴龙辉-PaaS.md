# 分享人@吴龙辉@厦门-网宿 PaaS #

本次分享跟大家聊聊PaaS，PaaS(平台即服务)是云计算中的一个火热的话题，特别是PaaS的三个典型代表：CloudFoundry、Docker和Kubernetes，包括一些技术性和非技术性话题，希望更加全面地帮助大家了解PaaS。


## 第一部分：PaaS的前世今生 ##

我们所熟悉的云计算分为3个层面：IaaS(基础架构即服务)、PaaS(平台即服务)，SaaS(软件即服务)。与SaaS相比,PaaS和SaaS的概念相对较新，也是最近几年云计算领域的集中发力点，像Amozon, 微软，谷歌，阿里云, 还有很多初创公司都推出了IaaS和PaaS产品。

下面一张图比较经典的比较传统IT、IaaS和PaaS:

![](http://i.imgur.com/IKHXYvm.jpg)

目前市面上有很多PaaS平台，我自己认为可以分为3个阶段

1. 第一代PaaS，比如Google App Engine, SAE.这是最早期的PaaS，当时并没有PaaS这个概念，但是他们做的事情，现在看来是包含在PaaS范围内的。
2.  第二代PaaS,比如CloudFoundry, Openshift.这是各大IaaS流行之后，顺势推出的PaaS, 并且迅速发展。 
3.  第三代Paas，比如Kubernetes。这是在Docker火爆之后，利用Docker的特性构建出许多PaaS，这些PaaS更加灵活，更加适应企业，逐渐成为PaaS的主力，我想也是在座各位很多人正在做的方向。

接下来给大家说明下刚才提到的几个典型的项目,CloudFoundry、Docker和Kubernetes，从中就可以几代PaaS进化的技术驱动力。

## 第二部分 CloudFoundry ##

Cloud Foundry是VMware于2011年推出的业界第一个开源PaaS云平台，后来分拆出Pivotal公司进行接管，2014创立Cloud Foundry基金会，希望吸引更多的公司参与，而不是Pivotal一家独大。

现在说的Cloud Foundry是V2架构，它的结构图如下：
![](http://i.imgur.com/Fo5N1rq.png)

1. Router：路由模块，所有的数据面和管理面请求都通过Router进行分发。
2. UAA/Login Server: 鉴权模块。
3. NATS: 消息总线，CloudFoundry内部组件的通信主要通过NATS进行通信。
4. Cloud Controller(CC):管理中心，负责应用的生命周期管理等等。
5. Health Manager: 应用的健康监控状态。
6. DEA: 应用的运行时节点，应用都是运行在DEA上。
7. Warden: 容器管理模块，类似Docker，提供给应用容器运行环境。
8. Service Brokers：用于适配对接各类的第三方服务，可以是各种关系数据库、中间件、缓存、云存储、内存数据库等各种服务。
9. Metrics Collector/App Log Aggregator: 平台应用的日志和监控数据收集。

CloudFoundry推出以后逐渐得到了各大厂商的支持， 华为云，IBM BlueMix, HP Cloud和Dell云服务都采用了CloudFoundry作为基础，一时间成为PaaS的代表。那么CloudFoudry的优势有哪些呢？

### 第一，开源、开发的架构 ###

开源是趋势，CloudFoundry顺应了趋势，自然可以吸引大批的开发者和公司参与其中。同时CloudFoundry是一个开发的架构，定义了一套标准，可以扩展多种框架、语言、运行时环境及应用服务，支持运行在云平台IaaS上。

### 第二，运维智能化 ###

运维能力是PaaS最最最最最重要的能力，这决定了PaaS的成功与否，如果PaaS无法提供强大的运维支持，为什么我要把应用托管在PaaS，我需要看日志和监控，我需要经常升级应用等等，IaaS可是提供了相当灵活的处理机制。CloudFoudry在这方面做出了很多努力，提供应用的容错容灾，弹性伸缩，负载均衡，安全控制，监控日志的收集汇总等等。

### 第三，容器 ###

这里不得不提容器技术，容器的快速，隔离的特性是非常适合PaaS的需求的，CloudFoundry中开发了Warden组件来实现容器管理，实际上Warden和Docker类似，只不过CloudFoundry当时并没有专注于容器这一块，Warden在易用性和设计上都不如Docker，自然被Docker抢了风头。但是CloudFoundry也有一些问题。CloudFoundry最大的问题是它只能支持简单的Web类应用: 应用只能暴露一个HTTP端口，应用之间通信无法直接通向，必须只能通过Router走HTTP等等。这是因为CloudFoundry一开始的设计模型是比较简单的，很多复杂场景是不支持的，目前V3架构Diego正在开发中，应该有这方面的考虑。总之CloudFoundry是第二代PaaS的代表，它在运作，架构和技术上相比第一代PaaS都有一定的提高，在云计算大潮中引领了PaaS的发展。
但是实际上PaaS一直处在雷声大雨点小的尴尬处境，使用PaaS绝大多数是开发者个人的玩票性质，真正使用的企业用户其实很少。
那接下来就是Docker登场了，可以说它的火爆再次激活了PaaS。

### 第四部分 Docker ###

Docker 是 PaaS 提供商 dotCloud 开源的高级容器引擎，Docker自2013年以来非常火热，2014年就已经火得没有朋友了。

这里有篇文章讲Docker的由来的,蛮有趣的http://www.oschina.net/news/57838/docker-dotcloud，有点无心插柳柳成荫的意思。

关于Docker的好处，现在有许多文章都会说明，这里我讲讲将个人的理解：

像很多文章提到的Docker快速敏捷（启动，停止都是以秒或毫秒为单位的），隔离轻量级（不添加额外的操作系统），这些实际上是Linux内核提供的能力，Docker是沿用了这些特性，像CloudFoundry的Warden也有这些能力。

Docker最大的创新点在于Docker Image的设计，下面是张Docker Image的层次图，分层的文件系统，一层层地搭建出一个完整的容器运行环境：
![](http://i.imgur.com/HAwVGMI.png)

这样一来，我们可以像乐高玩具一样搭建各式各样的镜像，同时Docker提供了一整套镜像存储方案（Docker Registry），可以非常方便地获取想要的镜像，然后快速地启动运行，而不必像以前那样安装各种麻烦的依赖软件。所以Docker更像个微创新者，但是解决了最大痛点，像Warden只解决了应用运行的问题，而Docker解决了应用的发布，构建和运行，创建了很多想象空间，紧接着基于Docker衍生出了很多新的解决方案，特别是PaaS，即第三代的PaaS, Kubernetes是其中最具代表性的一员，最后讲一下Kubernetes。 

### 第五部分 Kubernetes ###

Kubernetes是Google开源的容器集群管理系统。它构建Docker技术之上，为容器化的应用提供资源调度、部署运行、服务发现、扩容缩容等整一套功能，本质上可看作是基于容器技术的micro-PaaS平台。

关于Kubernetes的技术分析可以参考我的系列文章：http://dockone.io/people/wulonghui，这里不做展开。

因为Docker解决了应用编译构建的问题，所以Kubernetes架构上就可以专注在容器编排，服务发现等等运维相关特性上，像CloudFoundry就是花了很大精力在应用编译构建上，集成了BuildPack，但效果其实不是特别好。PaaS的价值不是在帮你编译打包上，而是在于节省运维成本。

Kubernetes的设计模型我认为有几个好的地方：

1. Label的设计： Pod、Service、Replication Controller之间的是通过Label进行关联，这是一种轻绑定的方式，可以方便地组合业务，做灰度升级等等
2. 网络模型:  Kubernetes采用扁平化的网络模型，每个Pod都有一个全局唯一的IP(IP-per-pod),Pod之间可以跨主机通信，相比于Docker原生的NAT方式来说，这样使得容器在网络层面更像虚拟机或者物理机，复杂度整体降低，更加容易实现服务发现，迁移，负载均衡等功能。
3. 最小单位处理单位Pod而非容器: Pod包含多个容器，可以进行组合，可以实现很多复杂的场景。

4. 简单合适的结构：Kubernetes组件的不多，但是合理。Master和Node的结构，以etcd作为存储和同步处理

当然，任何技术都不是银弹，需要按照各自的需求进行选型。
今天的分享就到了，大家有问题可以提出来，一起探讨探讨！！

# 问题 #

1.您说PaaS的三个典型代表：CloudFoundry、Docker和Kubernetes，最近的DaoCloud、灵雀云、时速云等以Docker为核心提出了新的CaaS概念。CaaS和PaaS都使用了Docker，Kubernetes等技术，您是怎么理解PaaS和CaaS的？

答：CaaS容器即服务，我的理解就是PaaS,主不过主打这Docker容器，所以叫CaaS

2.开源势必引入安全问题。Cloud Foundry的安全保护机制？
Cloud Foundry对于应用有个安全组的特性，可以控制应用容器的访问。但是实际上能力不强，容器安全这块一直是被担心的地方。

3.K8s可以实现应用的自主漂移吗？

答：目前没有Pod迁移的API

4.Cloud Foundry的DEA,即应用的运行时节点，是虚拟的操作系统么？

答： DEA是个服务，其实还是linux系统

5.Warden对底层的操作系统有和要求？Warden如何实现的隔离性？

答：内核有要求，实现原理和Docker类似

6.java/spring/web 应用很容易在Docker/Kubernetes环境运行，如果要迁移到CF，如何做 工作量大吗？

答：迁移到CF V2工作量应该挺大，V3的话因为支持DOcker，所以还好

7.关于K8S中flannel和weave两者哪个更适合生产环境？

答：这个我最近正在分析，感觉flannel和weave都还不成熟，推荐ovs

8.cloud foundary中监控部分有插件机制吗？

答：支持导入到各种第三方监控系统

9.Pod直接通信需要开通隧道么？换句话说他们之间是如何实现通信的？

答：容器跨主机的通信，是对docker的增强，像flannel,ovs等

10.docker集群如何保证应用的高可用？

答:高可用是个通用的课题，这个具体问题具体考虑了

11.对于管理Docker这件事情，除了Kubernetes, openstack (IaaS阵营), YARN(Hadoop阵营)也开始发力，怎么看待这几者间的竞争？

答：没有任何技术是银弹，不同需求不同设计。当然大家都想占据制高点了，这就是运作了

12.Pod直接通信需要开通隧道么？换句话说他们之间是如何实现通信的？

答：容器跨主机的通信，是对docker的增强，像flannel,ovs等

13.flannel是建立在kubernates上的吧，具体通信原理我不是很清楚哎，如果是手动的，好像是通过桥接的吧，那么kubernates具体是怎么玩的呢？？？

答：这个参考上一期的分享http://dockone.io/article/618

14.如果将flannel替换成ovs是不是也可以替换掉kube-proxy，如果是那会不会影响集群中SVC的使用？
flannel/ovs和kube-proxy的作用是不一样的，可以看下http://dockone.io/article/545