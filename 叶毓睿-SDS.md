# 软件定义存储（SDS） By 叶毓睿(Peter Ye) #

2013年10月Gartner发布2014年十大战略技术中，重要的组成部分就有：软件定义一切。Gartner认为：软件定义一切囊括了在基础设施可编程性标准提升下不断增长的市场势头、由云计算内在自动化驱动的数据中心互通性、DevOps和快速的基础设施提供等。软件定义一切还包括各种举措，如OpenStack、OpenFlow、Open Compute Project和Open Rack，共享相同的愿景。开放性将成为供应商的目标，SDN（网络）、SDDC（数据中心）、SDS（存储）和SDI（基础架构）技术的供应商都力图成为所在领域的领导，但在恪守开放性和标准方面却可能各有各的打算

个人感觉，未来软件定义存储(SDS)的发展，到后面与DevOps会有很紧密的联系

我们先来看下什么是软件定义存储？

大家心目中认为的软件定义存储（SDS）是什么？

最早的软件定义 是SDN，SDN起源于2006年斯坦福大学的Clean Slate研究课题。2009年，Mckeown教授正式提出了SDN概念。通过将网络设备的控制平面与数据平面分离开来，并实现可编程化控制，实现了网络流量的灵活控制，为核心网络及应用的创新提供了良好的平台。

SDS的最早提出，在2012年

2012年 VMworld大会上，VMware在全球首次提出软件定义数据中心(SDDC)的概念。作为VMware软件定义数据中心五大组成部分(计算、存储、网络、管理和安全)之一，软件定义存储(SDS)的概念也首次被提出。

VMware对SDS定义如下：
软件定义的存储产品是一个将硬件抽象化的解决方案，它使你可以轻松地将所有资源池化并通过一个友好的用户界面（UI）或API来提供给消费者。一个软件定义的存储的解决方案使得你可以在不增加任何工作量的情况下进行纵向扩展（Scale-Up）或横向扩展（Scale-Out）。

软件定义存储是VMware软件定义数据中心的五大组成部分之一。VMware认为，软件定义的数据中心，是 IT 演变的下一个阶段，是迄今为止最有效、恢复能力最强和最经济高效的云计算基础架构方法。SDDC方法论将对存储、网络连接、安全和可用性应用池化、抽象化和自动化，整个数据中心由软件自动控制。 基础架构提供的服务将聚合起来，并与基于策略的智能调配、自动化和监控功能结合在一起使用。 应用编程接口和其他连接器支持无缝延展到私有云、混合云和公有云平台。

![](http://i.imgur.com/ePOR073.jpg)

这个当中，比较容易理解的是其软件定义的三阶段论：抽象 池化 自动化

    抽象其实就是软硬件解耦的过程
    
    池化就是存储虚拟化和存储标准化
    
    存储虚拟化包括阵列内的各种存储资源的虚拟化

当我们讨论异构存储之间的管理的时候，其实也同时在讨论存储标准化，只有当大家开放的接口遵循共同的标准（也即规范）的时候，也就是用相同的“语言”对话时，才有可能被调用、被管理

**最高的阶段是自动化**

SDS能够根据业务应用的SLA、合规性、高可用性、工作负载等要求，通过存储API，自动地按需部署存储资源，也即虚拟存储设备（例如，块存储逻辑单元甚至RAID子设备，文件系统共享，存储对象等），提供业务应用所需的数据服务级别

我们再来看一下其他人对SDS的定义

IBM SDS:•	SDS is enterprise class storage that uses standard hardware(标准硬件) with all the important storage and management functions performed in intelligent software.
•	SDS delivers automated (自动化), policy-driven (策略驱动）, application-aware(应用感知) storage services through orchestration of the underlining storage infrastructure in support of an overall software defined environment.

SNIA的定义：SNIA认为，SDS需要满足的是：提供自助的服务接口，用于分配和管理虚拟存储空间；
SDS应该包括如下功能:
·自动化  Automation– Simplified management that reduces the cost of maintaining the storage infrastructure.
· 标准接口 Standard Interfaces – APIs for the management, provisioning and maintenance of storage devices and services.
· 虚拟数据路径 Virtualized Data Path – Block, File and Object interfaces that support applications written to these interfaces.
· 扩展性 Scalability – Seamless ability to scale the storage infrastructure without disruption to availability or performance.
· 透明性Transparency – The ability for storage consumers to monitor and manage their own storage consumption against available resources and costs.

还有许多，如EMC, IDC, Gartner等，我就不一一列举了

无论哪家，都把自动化当作其中最重要的一环，也是最高阶段

借助抽象池化自动化 最好要实现的就是Storage as a Service (STaaS)

软件定义存储中很重要的两个概念是控制平面与数据平面

这个与SDN类似

原来的传统存储，它的数据平面与控制平面是紧耦合的，所以应用的数据请求不得不通过应用人员手动告诉存储管理员。再由存储管理员来实施空间分配、数据保护、确保数据性能及安全等数据的部署和服务

数据平面(Data Plane)负责数据的处理

控制平面(Control Plane)负责数据的调度

例如：空间配置、自动分级、快照和克隆、压缩和去重、加密、高可用等功能组成了数据平面(Data Plane)的部分以及向上与数据中心或者云管理软件，甚至与应用软件的配合、联动的API管理，则组成了控制平面(Control Plane)的部分

控制平面(Control Plane)和数据平面(Data Plane)的逐渐分离使得SDS能够逐渐呈现出更丰富的API供Hypervisor/OS/Cloud去调用，实现更高程度的自动化

现在涌现出许多初创SDS公司，主要以数据平面为主，如Server SAN，也有少量做控制平面的，必须佩服，因为投资大，收效慢，如ProphetStor （希智）的Federator

![](http://i.imgur.com/E3gR3Ol.jpg)

这张图是我尝试做的SDS分类，不一定对 只是个人理解，可以参考一下

在SDS Control Plane这一层，比较著名的有：
1）	VMware SPBM (Storage Policy Base Management, 基于存储策略的管理)；

2）	OpenStack Cinder 。Cinder是OpenStack云平台的一个组件，用来提供块存储服务；

3）	EMC ViPR。目标是实现EMC存储、异构存储、商用硬件本地存储资源的存储虚拟化（包括互操作性）；备注：对互操作性不了解的朋友，可以查看历史文章 SDS之三）；

4）	ProphetStor （希智）的Federator；

5）	FalconStor（飞康）的 Freestor；


在整个SDS框架中，难度最大，但也最有价值的，是这一部分的公司，从长远来看，也许十年后，我们回过头会发现，得控制平面者，得SDS之天下。

数据平面的初创公司如雨后春笋般出现，国内最近一、两年内也出现很多，初步估计不少于20家，例如：国外有 Nutanix、VMware VSAN或EVO:RAIL、EMC ScaleIO、SimpliVity、Scale Computing、Pivot3、Maxta；国内有华为FusionStorage、大道运行TaoCloud、达沃时代Dawoo、志凌海纳SmartX等；开源的有Open vStorage（类似Nutanix架构）

**以上是软件定义存储（SDS）的定义及分类**


----------
软件定义是一个渐进的过程，目前软件定义存储主要专注在Data Plane。主要的形态是Server SAN或者超融合架构。不过这两个概念不完全一样。详细区分可以看一下我微信公众号的 SDS之四。也就是分类那张。

Server SAN把服务器盘资源池化 往外提供，内部多数用对象方式组织，聂风提到的块存储 是指对外的接口

应用使用存储空间 在传统外置磁盘阵列占主流的时候，
主要由SAN(FC, iSCSI)、NAS和对象这几种方式，大家最常见的是FC SAN

用在数据库等场景，NAS用在文件共享等场景，但外置磁盘阵列是时代的产物

二、三十年前CPU处理能力不够，所以数据处理搬到外面，再通过不同的接口协议去访问存储空间

二、三十年前CPU处理能力不够，所以数据处理搬到外面，再通过不同的接口协议去访问存储空间，它的兴起，其实是利用了现在强劲的cpu，加上高速网络和SSD的出现

分布式存储在未来5、6年会成为主流
有更高维度的关系型数据这样的？
这个可能是与更远期的SDS相关
也就是应用定义
不过，我个人仍然把应用定义的存储，看成为软件定义存储的一部分

### 现在谈下：SDS的发展：过去、现状与未来 ###
前面的其实已经部分提到了SDS出现的原因

![](http://i.imgur.com/hzdhHDI.jpg)

二三十年前，CPU的处理能力较弱，内存较小，单块磁盘的性能和容量都较小。为了不抢占宝贵的CPU和内存资源，也为了提高数据的性能、可靠性（如RAID保护）、可用性（如快照，容灾，双活等）、扩展性，以及提供方便易用的集中管理，诞生了外置磁盘阵列（也叫集中存储），阵列本身自带智能控制器，能够组织管理数据，并提供快照、容灾等高级的软件功能。有些高端存储甚至能在一个单一阵列里提供1000乃至数千块盘，如EMC  VMAX，HDS  VSP和华为OceanStor等

CPU迅猛发展，以及多核技术的出现；高性能SSD的出现；高速网络技术的出现；大容量盘的出现；奠定了分布式存储逐渐普及的基础

这块与Data Plan关联