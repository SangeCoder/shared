# 如何在裸机中自动安装部署CoreOS和K8S #

先介绍一下背景

随着光音业务规模的上升，线上业务产品的数量及服务器的采购量也越来越大。当达到一定数量级后，就不能使用常规的维护方法来解决这些问题。

以前，一旦业务量上去，我们就不得不停下手头的开发工作，部署业务所需要的环境及线上调试，到最后，只有特别熟悉业务和代码的同事才能胜任此工作。为了解决这些问题，我们从前年开始就关注了LXC，并试着小规模地使用了一段时间，但是由于LXC本身存在一系列的问题，比如内核版本的限制及二次开发困难，没能大规模地推广。我们也接触到了CoreOS，它的AB分区升级特性特别吸引人。使用后才发现，跟宣传写的不一样，还是需要重启服务器才能升级内核的，但是总的来说，结合Fleet的使用，可以动态地把业务迁到其它服务器，来达到平滑升级的目的，因此还是非常不错的。

我是从开发转到基础平台维护岗位的，所以希望用开发的方式来解决运维的问题，并想通过改变运维的方式的来加快业务研发的速度。但是真正专职做平台开发后，才意识到搭建整个运维体系是多么困难的事。

所以，为了更快地构建我们的平台，业务的选型首选开源框架，然后再它的基础上，根据业务的需要做二次开发。借着Google的名气以及相对完善的生态圈，我们最终选择了Kubernetes + CoreOS + Docker，做为整个平台编排调度的基础。

我们每次采购机器，一般都是几百个节点，所以整个平台的部署就非常头疼，特别是CoreOS和Kubernetes，必须借助梯子才能安装和更新，让人非常不爽。前期的大部分时间都浪费在这上面了！

于是，写了一个简单的Yoo-Installer工具来解决这些问题，现在分享给大家。

由于想呈现最简单便捷的安装方式，结果没把握好时间，这次的分享准备不足，所以你们可以多提问，我多分享一些采过的坑。

项目代码我放到GitHub（https://github.com/Goyoo/yoo-installer）上了，代码还在不断完善中，如果有问题，可以直接在上面提Issue。另外，我们组的一个同事赵文来也贡献Kubernetes-client 的Nodejs版本，一并分享给大家：https://github.com/Goyoo/node-Kubernetes-client，希望能跟大家一起打造美好的Docker生态圈。

下面介绍Yoo-Installer是如何工作的
它共分为DHCP Service、TFTP Service、HTTP Service，其中HTTP Service 里包括了CoreOS的引导，安装到硬盘，Kubernetes的安装及其它服务脚本的初始化。

![](http://i.imgur.com/HXvHE8T.png)


PXE引导

收到DHCP广播，获取IP
使用TFTP进行通信，传输CoreOS的基础IMGAGE
在内存启动CoreOS系统
系统启动成功后，下载脚本，执行安装Kubernetes及其它相关服务。

在这里需要分享几个点：

服务器的IP。在安装服务器前，应该确定好服务器的IP，因为IP在后面的安装是一个非常重要的变量。比如etcd的service IP，Kubernetes的Master IP都需要写入到配置里的。

我们是这样做的：

我们的服务器是高密度刀片服务器，都配有管理模块。通过一些简单的API调用，就可以得到所有的网卡mac信息，与KVM的IP保持一定的逻辑关系，这样就可以保证服务器的IP有序，便于日后的管理与维护。（可以参考Yoo-installer项目的app/utils/IPMI/dhcpMacList.js，通过这段代码得到dhcpd所需要的配置格式，直接使用即可）。

## CoreOS集群的安装方式 ##

我们专门做了一个菜单，以满足各种场景部署的需要。

![](http://i.imgur.com/0hOZHhw.png)

在Yoo-Installer里，我分了6个菜单，适合四种使用场景。

分别对应于官方的CoreOS集群架构

	* Docker Dev Environment on Laptop

	* Small Cluster

	* Easy Development/Testing Cluster

	* Production Cluster with Central Services


参考：CoreOS Cluster Architectures（https://coreos.com/os/docs/latest/cluster-architectures.html）

![](http://i.imgur.com/CVuwCue.png)

第一种，就是开发环境的方式。我们在开发时，可以使用这种方式来开发。

![](http://i.imgur.com/IQkM26G.png)

小规模集群的方式。每个节点都安装了etcd服务，任何一个节点故障都不会影响整体服务。

![](http://i.imgur.com/fetQyLE.png)

开发测试集群。这种方式的优点是不用额外部署etcd服务，需要几个节点来测试，就加入几个，非常灵活。但是不是高可用的架构。因为如果etcd死了，整体集群就不能工作了。所以，非常适合开发测试使用。

![](http://i.imgur.com/jrESHBs.png)

线上生产环境集群架构。这种基本上就可以做到高可用了。不同服务资源之间可以使用Meta信息进行标识。

这些代码是如何实现的呢？

![](http://i.imgur.com/1dd1dUc.png)

主要是通过cloud-config-url参数的设置来改变不同的启动脚本，如cloud-config-url=http://192.168.1.10/config/develop-etcd/pxe.yml。

注意，这里面有一个技巧。默认的话，CoreOS是没有密码的。有时安装时会出现一些问题，我会在启动参数里加上coreos.autologin标记，这样引导后，就可以直接进入系统，然后查找问题了。

详见代码：https://github.com/Goyoo/yoo-installer/blob/master/app/tftp/pxelinux.cfg/default。

这样，我们就能顺利启动系统了。

启动后，我们需要安装系统到硬盘，下载并安装Kubernetes，并初始化一些系统环境等等。
这样是怎么做的呢?

先看代码

![](http://i.imgur.com/NobBOPZ.png)

这里我们在cloud-config-url里自定义了一个服务，叫setup.service。等系统启动后，会下载相应的脚本：pxe.sh。
详见代码（https://github.com/Goyoo/yoo-installer/blob/master/app/views/script/pxe.ejs）。

脚本分为几个部分：

	1. 同步系统时间，新机器可能时钟有问题，这样会导致CoreOS不能正常安装，

	2. 硬盘分区，这个要根据自己的机器情况进行处理。

	3. 系统的安装。

	4. 离线下载已经准备好的Kubernetes，并放到相应的系统目录里。

	5. 其它脚本。

	6. 通知Yoo-Installer已经安装完成并重启。


那么，我们对于Kubernetes是如何自动处理的呢？

在CoreOS里，有一个Cloud-init文件，在每次系统启动时，它便会自动执行。我们利用了这个文件，自动搭建了Kubernetes的服务。

比如这个Central的YAML文件。

在这个配置文件里，有以下几个部分

hostname: 配置后，方便识别。

ssh_authorized_keys: 可以设置一个跳板机的key，方便管理。

update: CoreOS版本更新策略及更新版本设置。

Fleet: Fleet的服务。

units: 具体的systemd units文件配置，这里面包括了etcd、Fleet、Flannel、Docker、kube api server、kube controller manage、kube scheduler、kube-register等服务的配置。

这些服务的配置会自动创建相应的service文件。

我们也可以直接写文件，比如DNS的配置，『search localhost』 ，我在coreos-init里就没找到对应的功能配置key，只有写文件了。

这里有几个需要注意的地方服务器不建议用DHCP动态分配IP，就算是MAC绑死的也不好。我们出过一次故障，DHCP服务异常，导致业务网络中断。

关于cloud-init文件， CoreOS 提供了coreos-cloudinit的方法，可以做validate 。如果自己修改了init文件，最好检验一下，不然只有重启后才能发现问题。
coreos-cloudinit不加validate参数，还可以执行，像写文件之类的操作，直接就可以看到结果。

关于CoreOS自动下载更新。像我们这样自建机房的，最郁闷了，不像一些国外的公有云那么方便，直接可以下载更新。怎么做呢？首先，你要有个梯子，然后通过专门设置update 的service，加上ALL_PROXY，就可以自动更新了！

为了更便捷的使用Yoo-Installer，我计划做二个版本，一个是VM版本的，下载后，直接可以用，另一个是Docker版本的。由时准备不足，没有做完，但是因为我自己也需用，所以我会继续完善！

这里要吐个嘈。我在封装DHCPD时，因为桥接的问题，Docker里面的广播根本发不出来。如果您有高招，请告诉我，我会请你吃饭哦~
对于Yoo-Installer，为了便于使用，我们专门做了一个UI，功能还不丰富，希望大家能积极提意见，欢迎PR。

![](http://i.imgur.com/5PpHTBW.png)


# 问题： #
1.Q:节点=服务器？梯子？
A: 我们有几个管理服务器，做为节点进行proxy。如果需要的话，就在特定的程序变量里加ALL_PROXY. 这样做，也不会影响正常的网络

2。Q:网络模型怎么选?桥接，同网段大网？OpenvSwitch？
A:我们目前用的是Flannel，这个是CoreOS自带的

3.Q: cores的pxe是什么？
A: PXE功能是由主板提供的吧

4.Q:我有一个问题，刚好刚才问了一个Kubernetes创建pod，要从gcr.io下载image，被墙了，这个你们怎么解决呢？
A: 这个有二个方法。
1). 使用国外的服务器下载好一个tar包，然后下过来，load进来。
2). 修改tag, 在hub.docker.com里google好像有这个镜像，下载好后，修改成gc.io的tag
我在项目里放了一个tar包。

5.Q:最小集群的话一般需要几台VM？
A:这个要看你的业务场景。 我不知道你是不是指Small Cluster, 上面有图，需要3~5个即可

6.Q:从Docker容器,pod.到service,的网络是如何走的，有用dns吗
A:首先我们自己有一个内网的DNS，k8s也提供了DNS的功能

7.Q:我用虚拟机，里面跑了CoreOS , 3台机器组成一个小集群可以吗？
A:当然可以啊。集群组建有三种方式，如果你是自己玩玩，建议使用DiscoveryID的方法

8.Q:今天说容器动态IP分配不好，那你们实际生产中如何做的呢？
A: 直接写Static IP. 在初始化安装时，已经通过mac绑定上IP，然后通过IP这个变量写到文件里，做成static IP

9.Q:yoo installer做为开发开发人员的开发环境 有什么可以分享的吗？
A: 我觉得如果仅仅开发使用，可能不太适合。因为这个是为批量安装部署设计的。如果是开发后的压测，倒是可以考虑使用啊，因为想安装的话，放到安装网络环境里就可以了

10.Q: coreos 本身也是操作系统吗？跟虚机有啥区别呢，是安装更方便，还是本身比较简单？
A:CoreOS比较轻，而且默认没有包管理器，也就是不能随便安装东西。这样最大的好处是底层没有任何依赖。而且可以自动升级。当然还有其它优点，你可以看一下官方的介绍

11.Q: 请问有对K8S的API调用做限制类吗
A: 你是指限制资源吗？ 在配置文件里做就可以了啊

12.Q: 关于K8S的使用，RC，Services是否做了权限分配
A: 因为我们是自己的私有云，没有权限的分配

13.Q:coreos的系统和常用的linux系统的区别是什么呢！
A:
1).A/B系统分区
2).系统只读
3).通过配置cloud-config.yml来安装coreos系统（需要把coreos的安装包其他下载回来，更改一下install脚本）
4).进行安装coreos，第一次启动coreos是启动在内存中，需要运行安装脚本，安装到磁盘中
5).默认是没有密码，使用ssh-key进行登录，ssh-key在cloud-config.yml文件中进行配置（cloud-config.yml文件格式比较严格写的时候注意）

14.Q:开场前@王磊@北京—TenxCloud 似乎说道可以用dashboard.daocloud.io/mirror 里提供的docker加速器解决，但是我刚才试了，不行。不知道有人尝试过吗？
A: 这个你可以找DaoCloud的客服，他们服务很好。 我们是自己做的mirror cache。因为我们的服务器数量大。在内网做cache，一来可以节约带宽，二来可以提升某些机器无cache的开启速度

15.Q:请问整个集群中网络模式使用的是什么vxlan？网络服务用的flannel还是pipework
A:flannel

16.Q:你之前提到的有使用dhcp.server，想问下你们部署完后的测试环境，IP地址还能更改吗？
A:: 可以改啊，只要改IP的配置文件就可以了。一般安装后就不改了。  我们自己也做了指修改工具，如果好用的话，也可以分享出来给大家

17.Q:环境中存在其他dhcp服务，你们检测么？
A: 因为是自建机房，都是我们自己维护的，自己对拓扑很清楚。目前没有检测

18.Q:引导coreOS 时使用HTTP service作用是？
A:用http server充当软件包服务器

19.Q: 我用docker+etcd+confd+nginx搭建过集群，没有用于生产
kubernates这种集群，不是很懂，CoreOS现在很流行么？我也没有玩过
A: 我也只是抱着试试用的态度，有些特性很吸引人。我们公司的文化是这样的，鼓励试错，不怕出错。所以有什么问题，只有用了才知道。不过，确实有很多坑。

20.Q:如果环境做成IPv6,现在这套软件可以直接用吗?
A:可以

21.Q: 问题：这个coreos 系统   是怎么实现的
A: 我直接放代码了啊。有一个菜单库，我直接拿来用了

22.Q: 你们应该是实现了一个裸机安装os,然后部署k8s点东西吧,为啥没有用puppet或者installer这些来做k8
A: 因为整合成一套了啊，插上网线，上电，系统就安装好了。

23.Q: 硬件故障可能是coreos硬件驱动问题吧，你们是重启解决的还是重新部署？
A: 因为机器多了，故障也会显得多一些。 节点的挂机，不会影响正常的业务运转，这就是使用docker的好处啊。有些故障解决不了，我就直接关机

24.Q: 整个安装过程是自动化的 进度怎么掌握？
A: 在安装完成后，会有一个成功的请求到Yoo-Cloud，通过这种方式来统计

25.Q: 整个安装过程是自动化的 进度怎么掌握？
A: 在安装完成后，会有一个成功的请求到Yoo-Cloud，通过这种方式来统计

26.Q: 已经部署的改成静态IP不是得一个容器一个容器的改啊？
A: 有一套自动的脚本，只需要取出当前机器应该分配的IP即可。

27.Q: k8s自带ui用到了吗？如何？
A: 不好用，功能简单

28.Q: 请问这个能部署在云主机里不
A: 云主机提供商，大部分都支持COREOS了吧。具体的情况，我还真没测试过

29.Q: 他们没有把os安装和服务部署分开！现在应该很少自己公司产品用
A: 我们做的是私有混合云，服务的部署是另一套编排工具

30. Q：网络规划的时候没有吧ip和端口绑定？还需要用yoo_install分配ip给物理机？
A: 没有。那样做网络更加复杂，在我们的应用场景下，偏向于大二层

31.Q: flannel 容器到 host 能不能通信？
A: 可以通信啊，你需要用来做什么？
因为数据层，不会采用 docker 那么这个网段和 contain 不同，在 第三层 就是不通的，即使使用动态路由协议也不行，应为中间可能还有路由器
这样就只能使用，一个 host 的 ip ，现在有两个方案，1：nat 2：tunnel

32.Q: 这套里面你们都跑哪些应用呢？
A: 我们公司里的大部分业务，目前正在努力推进Micro-services

33.Q: 问，k8s对外发布服务时  像端口规划如何处理
A: 我们会规定一些port，因为我们业务还算单一，主要就是http和部分tcp

34.Q: 之前应用分布多台云主机上，端口比较统一，放k8s集群，如何处理
A: 他们在flannel里，都是不同的ip了，所以不会出现端口重复的问题。

35.Q: 请问碰到过手机服务 TCP 长连接被中断问题么
A: 我们也有长连接，不过目前没放到coreos里。对于移动端，中间很常见的吧

36.Q:应用部署用的是？Chef/Puppet？
A: 目前尽量使用docker, 除了一些编排类的工具。 因为coreos本身安装东西就不方便，而且为了减少底层依赖。还有一个想法，热备放到公有云上，这样未来就不用考虑这些依赖了。一些简单的，使用Fleet Global的形式，全局跑一个脚本的体验也很爽。

37.Q: 问：有现成来源的方案没？关于lb
A: 没有，如果你有好的方案，可以跟我分享一下，请吃饭哦






