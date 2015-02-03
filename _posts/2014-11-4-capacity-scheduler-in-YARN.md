---
layout: post
category: yarn
title: Yarn容器调度器设计
tagline: by 袁野
tags: [yarn,scheduler]
---

简要介绍Capacity Scheduler结构以及配置。

## 简介

随着Hadoop愈加流行，公司内部的Hadoop集群规模也不断增加，使得Hadoop资源成为一个共享资源。CS(Capacity Scheduler)既要使得hadoop资源能够以一种友好的方式为多用户共享，又能最大化集群的资源使用和吞吐量。

传统上，各个部门拥有各自能够满足自己性能需求的集群，通常将导致集群的平均使用率较低。通过使用一个大规模的共享Hadoop集群，可以减少维护成本以及提高整体资源使用率，从而减少成本。

CS需要在保证各个部门资源情况下，共享集群。其主要思想是Hadoop集群在多个部门之间共享，各部门根据自己需要使用集群资源。并且CS还给各个部门提供了资源弹性，使得可以使用其他部门空闲的资源。

CS支持如下特性:

- 层级队列 - 支持层级队列，提供对资源更细粒度的控制。
- 容量保证 - 管理员可以配置各队列设置资源的最低保证和最高使用上限，提交到该队列的所用应用将共享这些资源。
- 安全保证 - 各队列有严格的访问控制，控制各个用户对各个队列应用的提交权限、查看权限和修改权限。
- 弹性 - 空闲的队列的资源可以被其它资源紧缺的队列使用，提高整体资源使用率。而一旦该队列中有应用提交，其它资源将释放资源归还给该队列。
- 多重租约 - 提供多种限制，防止单个应用、单个用户或单个队列使用过多资源。
- 操作性
	- 运行时配置 - 队列定义以及容量、ACL等属性支持运行时配置。
	- 应用清空 - 管理员可以动态的停止队列，使得已经提交的应用正常完成，并拒绝新应用的提交。

## 队列控制

YARN调度的基本单位为队列。各队列有如下属性：

- 简称
- 完整名称（包括完整路径）
- 相关的子队列和应用
- 最小容量保证
- 最大可用容量
- 对应用户以及各用户的资源分配限制
- 队列状态（RUNNING or STOPPED)
- ACLs(访问控制列表）

### 启用CS

配置`etc/hadoop/conf/yarn-site.xml`中的如下项：

{% highlight xml %}
<property>
  <name>yarn.resourcemanager.scheduler.class</name>  <value>org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.CapacityScheduler</value>
</property>
{% endhighlight %}

### 设置队列

YARN调度的基本单位为队列。各队列的容量规定了提交到该队列的应用可以使用整个集群资源量的百分比。根据使用集群资源的的部门、组织以及用户分布情况，设置相应的队列结构和资源分配。

例如，假设某一公司有三个部门：工程部，支持部以及市场部。工程部有开发和QA两个团队。支持部有培训和服务两个团队。市场部分为销售和广告两个团队。下图反应了改层级关系：

![capacity scheduler queues](/images/cs/capacity_scheduler_queues.png "capacity_scheduler_queue")

各子队列通过`capacity-scheduler.xml`中的`yarn.scheduler.capacity.<queue-path>.queues`属性与父队列建立关联。顶级队列“支持”，“工程”，以及“市场”队列可以通过如下配置与root队列建立关联（root队列为整个集群的顶级队列，不属于任何组织）:
{% highlight xml %}
<property>
  <name>yarn.scheduler.capacity.root.queues</name>
  <value>support,engineering,marketing</value>
  <description>The top-level queues below root.</description>
</property>
{% endhighlight %}
通过如上方式定义剩余队列。

**层级队列特性**

- 两种队列类型：父队列和叶子队列
- 父队列用于管理各部门以及子部门资源，可以包含其他父队列以及叶子队列。不能直接将应用提交到父队列中。
- 叶子队列存在父队列下，不包含子队列，直接接受应用提交。
- 最顶级队列root不属于任何部门，代表这集群。
- 通过父队列和叶子队列，管理员可以为不同的组织和部门分配资源。

**队列间调度**
 	
层级队列保证资源首先在组织的个子部门间空闲，然后在各个组织间共享。这使得各个组织对其资源可以有如下控制：

- 在队列的各个层次，各父队列以子队列当前资源使用情况将其有序的排列。
- root队列知道如何在一级父队列间分配资源以及调度。
- 父队列约束对其所有子队列有效
- 叶子队列维护该队列中的应用，以FIFO方式进行调度，并保证对各用户资源限制。

### 配置队列ACLs

ACLs可以用于对用户和管理员对队列的访问进行控制。只能将应用提交到叶子队列，但父队列上的ACL约束将应用于其所有后继队列。

在CS中各，通过`acl_submit_applications`属性配置，将队列的访问权限授权给用户和组列表。列表格式为"user1,user2 group1,group"。该属性也可设置为'*'，表示允许所有用户和组允许访问；设置为" "（空格符）表示禁止任何用户和组的访问。

如下配置将"support"队列的访问权限授权给用户"sherlock"、"pacioli"以及"cfo-group"组中的所有成员：

{% highlight xml %}
<property>
	<name>yarn.scheduler.capacity.root.support.acl_submit_applications</name>
	<value>sherlock,pacioli cfo-group</value>
</property>
{% endhighlight %}

还可以设置单独的ACL，控制队列的管理员。队列管理员可以向该队列提交应用，杀死队列中的应用以及获取队列中任何应用的信息。

管理员ACLs通过*acl_administer_queue*属性配置。如下示例将授权"cfo-group"对"support"的管理员权限：
{% highlight xml %}
<property>
	<name>yarn.scheduler.capacity.root.support.acl_administer_queue</name>
	<value>cfo-group</value>
</property>
{% endhighlight %}

### 配置队列资源

管理员使用容量属性给队列分配资源百分比。如下配置将集群资源按6:1:3的比率分配给工程部以及市场部。

{% highlight xml %}
<property>
	<name>yarn.scheduler.capacity.root.engineering.capacity</name>
	<value>60</value>
	<description>分配给工程部60%的资源</description>
</property>
<property>
	<name>yarn.scheduler.capacity.root.support.capacity</name>
	<value>10</value>
</property>
<property>
	<name>yarn.scheduler.capacity.root.marketing.capacity</name>
	<value>30</value>
</property>
{% endhighlight %}

假设工程部需要将资源以1:4的比率分配给开发部和QA部，则可以如下设置：

{% highlight xml %}
<property>
	<name>yarn.scheduler.capacity.root.engineering.development.capacity</name>
	<value>20</value>
</property>
<property>
	<name>yarn.scheduler.capacity.root.engineering.qa.capacity</name>
	<value>80</value>
</property>
{% endhighlight %}

`[Note]`
任何层次所有容量之和为100%、并且，任何层次中的任何队列容量不小于1%。

下图展示了集群容量配置:

![capacity_scheduler_queue_percents](/images/cs/capacity_scheduler_queue_percents.png "capacity_scheduler_queue_percents")

**资源分配工作流**

调度过程中，层级队列中任何一级的队列都将以当前使用资源量进行排序。对于资源调度有如下工作流：

- 越缺乏资源的队列，资源分配的优先级越高。最缺乏资源的队列是指集群中，资源使用比率最低的队列。
	- 任何父队列已使用资源定义为其所有后继队列使用资源的总和。

	- 叶子队列已使用资源定义为该队列中当前正在运行的应用所使用的所有Container资源总和。
 
- 一旦某个父队列获得了资源，该资源就将根据上面描述的分配方法，递归的计算并调度给某一个子队列。
- 之后，各叶子队列以FIFO的方式将资源分配各其中的应用。

	- 该步骤与本地化、用户级别限制以及应用限制独立。

- 为了保证分配的弹性，其他队列为使用的资源将自动被分配给需要资源的队列。

**资源分布工作流示例**：

假设集群有100个节点，每个节点10GB内存可用于分配YARN Container，共有1TB内存可用。根据之前的分配测量，工程部将获得600GB的内存，支持部将获得100GB，而市场部将获得300GB。

在工程部中，开发团队和QA团队奖分配获得120GB和480GB。

考虑如下事件时间线：

开始时，整个过程部队列没有任何应用运行，而支持部和市场部资源以及完全使用。

用户Sid和Hitesh首先往开发叶子队列提交了应用。。它们的应用根据配置情况，可以使用整个集群的资源，整个集群的部分资源。尽管开发部队列只分配了120GB，Sid和Hitesh却各得到了120GB，总共240GB。

尽管开发部配置只分配120GB，但是为了更好的使用集群资源，CS允许弹性的共享集群资源。因为工程部队列中没有其他用户使用，Sid和Hitesh允许使用其他的空闲资源。

接着，用户Jian和Zhijie以及Xuan向开发部叶子队列提交了更多的应用。尽管各限制为120GB，但资源中分配却是600GB -- 基本使用完分配给QA队列的所有资源。

用户Gupta又提交了一个应用到QA队列中。因为集群中没用空余的资源，他的应用必须等待。

假设开发部队列使用了集群所有可用的资源，Gupta可能可以要回QA队列保证的资源 -- 根据是否启用了抢占机制。

随着Sid，Hitesh，Jian和Zhijie以及Xuan的应用运行完成，出现了剩余资源，然后可用的Container将分配给Gupta的应用。该过程将持续，知道开发部和QA部资源使用比率得到配置的1:4。

从该例中可以看出，存在可能使得一个用户滥用资源，不断的提交应用，并锁住其他队列的资源分配，知道Container完成或被抢占。为了防止这种情况，CS至此后限制队列的弹性成长。例如，为了限制开发队列独占工程部队列的资源，管理员可以设置如下属性：

{% highlight xml %}
<property>
	<name>yarn.scheduler.capacity.root.engineering.development.maximum-capacity</name>
	<value>40</value>
</property>
{% endhighlight %}

设置了该属性之后，开发部队列的用户依然能够使用比其容量120GB更多的资源，但是他们不会使用超过；工程部父队列总容量的40%，也就是240GB。

Capacity和Maximun-capacity属性可以用于控制整个部门以及子部门的YARN集群资源使用。管理员可以平衡这些属性，并避免过于严格的限制而导致资源使用率的降低，也要避免过多的跨部门资源共享。并且这两个属性可以动态的通过 `yarn rmadmin -refreshQueues`配置。


## 设置用户管理

`minimum-user-limit-percent`用于设置各个叶子队列用户最小资源占有比率。例如，为了使得五个用户共享服务叶子节点，可以将该属性设置为20%。:  

{% highlight xml %}
<property>
	<name>yarn.scheduler.capacity.root.support.services.minimum-user-limit-percent</name>
	<value>20</value>
</property>
{% endhighlight %}

下图展示了随着提交Job的用户增多，队列资源是如何调整的：

![min_user_limit.png](/images/cs/min_user_limit.png "min_user_limit") 

同队列控制一样，可以控制每个用户最多能使用其队列资源的比率。

{% highlight xml %}
<property>
	<name>yarn.scheduler.capacity.root.support.user-limit-factor</name>
	<value>0.5</value>
</property>
{% endhighlight %}

默认值"1"表示队列中任何用户最多可以使用该队列的配置容量。这可以防止单个队列中的用户独占整个集群的资源。该值设置为"2"表示限制最大资源使用量为当前队列配置容量的2倍。设置为"0.5"表示任何用户最多使用队列资源的一半。

## 应用预留
CS负载根据资源请求，匹配相应的可用资源。而存在如下情况，尽管某节点上存在空闲资源，但资源大小不足以满足队列首部的应用。通常在高内存需求的应用情况下会出现这种请款。这种资源不匹配情况,将导致资源密集型应用饥饿。

CS预留机制如下处理该问题：

- 当某节点报告已完成Container时，CS将选择一个合适的队列去使用该资源。
- 在选择的队列中，CS以FIFO顺序查找应用。一旦找到合适的应用，CS将判断该节点的空闲资源能够满足该应用的需求。
- 若出现大小不匹配，CS将立即在该节点上为该应用要求的资源创建一个资源预留。
- 一旦某个节点上创建了预留，这些预留资源将不会被其它队列、应用或者Container使用，知道该应用的预留得到满足。
- 预留资源的节点在出现足够资源时通过心跳报告。此后，CS将该预留标记为已满足，并移除，接着在该节点上为该应用分配资源。
- 某些情况下，其他节点出现了满足要求的资源，所以该应用不再需要等待，而该预留也将直接被取消。

为了防止预留资源无节制的增加，并避免任何潜在的调度死锁，CS对一个节点最多维护一个预留。

## 队列启动停止
 
YARN中队列存在两个状态：RUNNING和STOPPED。RUNNING表示队列能够接受应用提交，STOPPED队列将不再接受应用提交。队列默认状态为RUNNING。

CS中，父队列、叶子队列以及root队列都能够停止。一个应用要成功的提交到某一个叶子队列中，该队列以及其所有层级队列都必须为RUNNING状态。这就意味着某个队列为STOPPED，其所有后继队列也将停止，而无论其状态如何。

管理员可以使用该功能停止或清理某队列中的所有应用。

## 应用控制

Setting Application Limits

为了防止由于过大负载造成系统抖动 -- 由于太多用户或意外 -- CS可以对应用并发数进行一些静态的，可配置的限制。`

{% highlight xml %}
<property>
	<name>yarn.scheduler.capacity.maximum-applications</name>
	<value>1000</value>
</property>
{% endhighlight %}

任何队列中同时运行应用限制都是该限制的一部分。该限制为硬限制，意味着达到或超过该限制后，任何新应用都将被拒绝。该限制可以通过如下方式为各子队列单独设置：

{% highlight xml %}
<property>
	<name>yarn.scheduler.capacity.<queue-path>.maximum-app****lications</name>
	<value>absolute-capacity * yarn.scheduler.capacity.maximum-applications</value>
</property>
{% endhighlight %}

如下配置为设置ApplicationMaster最大使用的资源量，默认为10%。

{% highlight xml %}
<property>
	<name>yarn.scheduler.capacity.maximum-am-resource-percent</name>
	<value>0.1</value>
</property>
{% endhighlight %}
同样，该属性也能单独为各队列进行设置。
{% highlight xml %}
<property>
	<name>yarn.scheduler.capacity.<queue-path>.maximum-am-resource-percent</name>
	<value>0.1</value>
</property>
{% endhighlight %}



## 抢占

Update: 当前抢占存在bug.

**抢占工作流**：
![preemption_workflow](/images/cs/preemption_workflow.png "preemption_workflow")


**抢占配置**

在`etc/hadoop/conf/yarn-site.xml`中如下配置以启动抢占。

{% highlight xml %}
<property>
	<name>yarn.resourcemanager.scheduler.monitor.enable</name>
	<value>true</value>
</property>
{% endhighlight %}

将该值设置为true以启动抢占机制。浙江启动一系列周期性监控。
{% highlight xml %}
<property>
	<name>yarn.resourcemanager.scheduler.monitor.policies</name>
	<value>org.apache.hadoop.yarn.server.resourcemanager.monitor.capacity.ProportionalCapacityPreemptionPolicy</value>
</property>
{% endhighlight %}

SchedulingEditPolicy为当前唯一可用的抢占机制。
{% highlight xml %}
<property>
	<name>yarn.resourcemanager.monitor.capacity.preemption.monitoring_interval</name>
	<value>3000</value>
</property>
{% endhighlight %}

以毫秒为单位。
{% highlight xml %}
<property>
	<name>yarn.resourcemanager.monitor.capacity.preemption.max_wait_before_kill</name>
	<value>15000</value>
</property>
{% endhighlight %}

以毫秒为单位，表示从请求抢占到杀死container之间最大时间。将该值设置为较大值将给应用更多时间对抢占请求作出反应。
{% highlight xml %}
<property>
	<name>yarn.resourcemanager.monitor.capacity.preemption.total_preemption_per_round</name>
	<value>0.1</value>
</property>
{% endhighlight %}

一次抢占中，最多抢占资源的最大百分比。

**EOF**