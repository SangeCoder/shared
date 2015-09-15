
我们是在今年的5月份开始调研并尝试使用Mesos，第一个试点就是我们的日志平台，将日志分析全部托管在了Mesos平台上。这个平台面向业务线开发/测试/运营人员，方便定位/追溯线上问题和运营报表。

![](http://i.imgur.com/0kEceH7.jpg)

这个是我们平台的结构概览。

日志分析我们使用ELK（Elasticsearch，Logstash，Kibana），这三个应该说是目前非常常见的工具了。而且方案成熟，文档丰富，社区活跃（以上几点可以作为开源选型的重要参考点）。稍微定制了下Kibana和Logstash，主要是为了接入公司的监控和认证体系。

日志的入口有很多，如kernel,mail,cron,dmesg等日志通过rsyslog收集。业务日志通过flume收集，容器日志则使用mozilla的heka和fluentd收集。

这里稍稍给heka和fluentd打个广告，两者配合收集Mesos平台内的容器日志非常方便，可以直接利用MESOS_TASK_ID区分容器（此环境变量由Mesos启动容器时注入）。而且我们也有打算用heka替换logstash。

下面主要分享一下Mesos这块，我们使用了两个框架：Marathon和Chronos，另外自己开发了一个监控框架Universe。

先说Marathon，eventSubscriptions是个好功能，通过它的httpcallback可以有很多玩法，群里经常提到的bamboo就是利用这个功能做的。利用好这个功能，做容器监控就非常简单了。

另外，推荐设置一下minimumHealthCapacity，这样可以减少重启（更新）时的资源占用，防止同时PUT多个应用时耗光集群资源。

服务发现，Marathon提供了servicerouter.py导出haproxy配置，或者是bamboo，但是我们现在没有使用这两个。而是按协议分成了两部分，HTTP协议的服务是使用openresty开发了一个插件，动态加载Marathon（Mesos）内的应用信息，外部访问的时候proxy_pass到Mesos集群内的一个应用，支持upstream的配置。

非HTTP的应用，比如集群内部的statsd的UDP消息，我们就直接用Mesos DNS + 固定端口来做了。随即端口的应用依赖entrypoint拉取域名+端口动态替换。

带有UNIQUE attribute的应用，官方目前还无法做到自动扩容，从我们的使用情况来看，基于UNIQUE方式发布的应用全部是基础服务，比如statsd，heka（收集本机的docker日志），cadvisor（监控容器）等，集群新加机器的时候Marathon不会自动scale UNIQUE实例的数量，这块功能社区正在考虑加进去。我们自己写了一个daemon，专门用来监控UNIQUE的服务，发现有新机器就自动scale，省的自己上去点了。

另外一个问题，资源碎片化，Marathon只是个框架，关注点也不在这里。Mesos的UI里虽然有统计，但是很难反应真实的情况，于是我们就自己写了一个Mesos的框架，专门来计算资源碎片和真实的余量，模拟发布情况，这样我们发布新应用或者扩容的时候，就知道集群内真实的资源余量能否支持本次发布，这些数据会抄送一份给我们的监控/报警系统。Chronos我们主要是跑一些定时清理和监控的脚本。

![](http://i.imgur.com/ncNokvz.png)

docker这块，我们没有做什么改动，网络都使用host模式。Docker的监控和日志上面也提到了，我们用的是cadvisor和heka，很好很强大，美中不足的是cadvisor接入我们自己的监控系统要做定制。

我们也捣鼓了一个docker ssh proxy，可能是我们更习惯用虚拟机的缘故吧，有时候还是喜欢进入到容器里去干点啥的（其实业务线对这个需求更强烈），就是第一张图里的octopus，模拟docker exec -it的工作原理，对接Mesos和Marathon抓取容器信息。这样开发人员在自己机器上就能SSH到容器内部debug了，也省去了申请机器账号的时间了。

Mesos我们跑的是0.22版本，没有上最新的，主要是看到官方说新的功能还没有适用在生产。

集群上跑的全部是日志解析的服务，偏计算类与业务线整合度比较深，对接了一些内部系统，主要是为了考虑兼容性和业务线资源复用的问题，我尽量省略与内部系统关联的部分，毕竟这块不是通用性的。

前跑了有600+的容器，数量比较少。跟其他公司的业务线比起来不是一个体量的。网络是docker自带的host模式，每天给业务线处理51亿+日志，延时控制在60～100ms以内。

最先遇到的问题是镜像，是把镜像做成代码库，还是一个运行环境？或者更极端点，做一个通用的base image？结合Logstash，heka，statsd等应用特点后，我们发现运行环境更适合，这些应用变化最大的经常是配置文件。所以我们先剥离配置文件到gitlab，版本控制交给Gitlab，镜像启动后再根据tag拉取。

另外，Logstash的监控比较少，能用的也就一个metrics filter，写Ruby代码调试不太方便。索性就直接改了Logstash源码，加了一些监控项进去，主要是监控两个queue的状态，顺便也监控了下EPS和解析延时。

Kafka的partition lag统计跑在了Chronos上，配合我们每个机房专门用来引流的Logstash，监控业务线日志的流量变得轻松多了。

![](http://i.imgur.com/iWOP9iD.png)

容器监控最开始是自己开发的，从Mesos的接口里获取的数据，后来发现hostname：UNIQUE的应用Mesos经常取不到数据，就转而使用cadvisor了，对于Mesos/Marathon发布的应用，cadvisor需要通过libcontainer读取容器的config.json文件，获取ENV列表，拿到MESOS_TASK_ID和MARATHON_APP_ID，根据这两个值做聚合后再发到statsd里（上面提到的定制思路）。

发布这块我们围绕这Jenkins做了一个串接。业务线的开发同学写filter并提交到gitlab，打tag就发布了。发布的时候会根据业务线机房规划替换input和output，并验证配置，发布到线上。

本地也提供了一个sandbox，模拟线上的环境给开发人员debug自己的filter用。

同时发布过程中我们还会做一些小动作，比如Kibana索引的自动创建，dashboard的导入导出，尽最大可能减少业务线配置Kibana的时间。每个应用都会启动独立的Kibana实例，这样不同业务线间的ACL也省略了，简单粗暴，方便管理。没人使用的时候自动回收Kibana容器，有访问了再重新发一个（充分体验下容器的便捷）。


![](http://i.imgur.com/0pOPo3H.png)


今天的内容就到这里，感谢大家。


**问题：**

1）基础架构里面，数据持久化怎么做的？使用是么架构的存储？

答：持久化剥离在集群外部，Mesos0.23提供了持久化的支持，但是没敢上到生产环境中。

2）Storm on Mesos，快可用了吗？是否跟Spark on Mesos做过比较？

答：Storm on Mesos的代码在github可以找到，说实话只是基础功能，许多资源的控制和attributes的东西都没有做，而且我们的测试发现storm on mesos的消息ack特别慢。不建议直接拿来就跑。

3）发布用的业务镜像是怎么制作的，平台有相关功能支持吗？还是用户手工做好上传？

答：Jenkins build的镜像，可以用Jenkins on Mesos这个框架，安装docker和mesos插件就可以开始用了。

4）Docker网络你用Host模式，弹性扩展时不会出现端口冲突吗？

答：

5）我们最近贡献两个PR给cAdvisor，支持ELK，给提提意见

答：我看到了你的PR，因为我当时也在搜索cadvisor的一些issue。我个人认为cadvisor最好再过一层Queue再进入ELK，虽然cadvisor可以缓存部分数据，但是和Queue相比还是太弱。后端一旦出现503后可能会丢失部分时间的数据。

6）上周我做了一个本地容器，但还是对容器的理解有点问题。帮忙讲解一下本地容器的工作原理

答：

7）做这么个平台用多大规模的团队开发的？用了多久？

答：开始2个人，最多的时候5个人，现在保持在4个人。从5月份到现在一点一点测试，扩容慢慢堆出来的。

8）镜像存储单机应该是不行的，你们是怎么管理镜像的？

答：镜像放swift里了。

9）端口应该规划过？为什么选用了mesos？而不是直接使用分析框架

答：少量的规划，大部分还是随即端口。这个平台没有上下文的联动，Spark/Storm在最开始的时候没考虑接入，所以选了一个轻便的mesos了

10）mesos主要是做底层的么？它有木有安全方面的问题，可以做安全策略配置么？

答：做资源调度的，我了解到的情况是Mesos仅对slave和framework的接入做了认证，至于framework跑了什么它就不管了。

11）为什么没有考虑k8s，或者swarm

答：用它作为资源调度的平台，没用k8s的原因是我们有点不太敢绑死在docker上。mesos自己也有一个轻量级的容器，也算是个备选方案。

12） 是通过代理认证的方式么？

答：mesos的master会认证，--credentials就可以帮助。

13)docker采用host方式，是什么考虑或者限制，效果怎么样?

答：平台本身就是大吞吐量的，bridge模式性能测试都偏低，就选择了host模式跑了。

14）mesos的调度算法是怎样的？有没有做容器的高可用？

答：双层调度，底层offer一个资源列表给framework，framework根据cpu和mem去列表中过滤，选中合适的slave部署，framework层的调度可以随意实现。Marathon已经帮忙做了高可用。

15）如何在mesos栈的各级上实现容错和资源的分配？

答：没太明白问题，能在详细点描述么？

16）使用效果如何？600容器部在多少台机器上，cpu和内存使用率多少？有什么提升资源使用率的策略吗？

答：效果达到预想了，600台分部在了混合集群上，有VM和实体机，全部折算的话大概30台左右吧，目前资源还有富余，主要是预留buffer。不推荐跑虚拟机，Mesos的资源分配就是一刀切，即使没用也算的，所以虚拟机利用率会很低。

17）你们日志应该是不同业务的吧？不同业务日志怎么传递给你们的啊

答：业务线的日志我们公司有一个专门的服务治理系统，flume和这个系统是打通的，收集的时候其实只依赖一个对应的app code就行了。

18） kafka & ES 没有在mesos上吗？ 

答：是的，kafka和ES没放到mesos上，先跑稳定计算，存储在规划，单独跑计算的话磁盘都浪费了，不过我们第一步想把HDFS先放上来

19）既然使用了mesos ，对于docker 单个监控独立部署是不是有点重了？

答：cadvisor是一台宿主机一个，内存占用稳定在80M左右，CPU可以忽略。还是很值得的。

20）mesos在哪一层面实施了paxos投票？

答：leader的选举，mesos其实是两套选举，即使zk挂了mesos master也可以自己选举leader。

21）为什么选择heka，而不是fluend。然后logstash有什么问题

答：fluent和heka我们都在用，都是收容器日志，heka可以从容器的ENV里读更多东西出来。换logstash主要看重了heka的监控，资源占用和处理速度。

22）会涉及数据卷的迁移吗，有什么好解决方案？

答：目前平台里没有持久化的东西。这块不敢说用什么合适，我们也没开始尝试

23）镜像做成代码库和运行环境具体是什么意思，为什么运行环境方式合适

答：镜像具体做成什么样要根据自己的应用来判断，我们用的logstash，statsd什么的，都是配置文件描述流程的，所以我们选择了运行环境的方式。

24）如果使用UNIQUE模式，任务发布如何规划的？ 每个主机一种服务跑多task还是只跑一个task

答：全部是hostname:UNIQUE，部署基础服务，heka，statsd，cadvisor都是hostname:UNIQUE的，每个机器只有一个，HA交给Marathon

25）你们用fluentd收集容器日志是怎么区分业务的？把flume打到容器里面了吗？

答：业务日志走的flume，我们内部有一个服务治理的应用，你可以理解成公司内部的应用全部都在这里注册，注册后分配一个唯一的app code，flume收集的时候会自动同步app，同步app code，并发给kafka，后续的全部解析都是绑定到app code这个级别的；  不需要打，一个机器一个heka的容器，可以把这台机器上全部的docker日志都收了，heka配合MESOS_TASK_ID，容器就自动区分出来了，在Logstash里可以直接根据MESOS_TASK_ID分类，并写入不同的索引，整套过程不需要额外的工作。