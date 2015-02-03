---
layout: post
category: yarn
title: Yarn公平调度器设计
tagline: by 袁野
tags: [yarn,scheduler]
---

<!--more-->


##FairScheduler主要类类图

![Main Classes in FS](/images/fs/classes.jpg "Main Classes in FS")

YARN中调度器均采用队列方式进行资源的细化管理，FS亦不例外。FS采用三层调度模型：队列 => 应用 => Container。可以通过配置文件对FS队列结构以及各个队列的属性进行限制。配置示例如下:

{% highlight xml %}
<?xml version="1.0"?>
<allocations>
  <queue name="sample_queue">
    <minResources>10000 mb,0vcores</minResources>
    <maxResources>90000 mb,0vcores</maxResources>
    <maxRunningApps>50</maxRunningApps>
    <weight>2.0</weight>
    <schedulingPolicy>fair</schedulingPolicy>
    <queue name="sample_sub_queue">
      <aclSubmitApps>charlie</aclSubmitApps>
      <minResources>5000 mb,0vcores</minResources>
    </queue>
  </queue>
  
  <user name="sample_user">
    <maxRunningApps>30</maxRunningApps>
  </user>
  <userMaxAppsDefault>5</userMaxAppsDefault>
  
  <queuePlacementPolicy>
    <rule name="specified" />
    <rule name="primaryGroup" create="false" />
    <rule name="default" />
  </queuePlacementPolicy>
</allocations>

{% endhighlight %}


对于各队列，可以设置队列名称、最大最小资源限制、最大允许App数量、权重、调度策略以及ACLs（访问控制表）。

##资源公平性

FS中，公平性定义为：

R= ResourceAllocation/Weight

对于所有的Schedulable，R相等。

该公式可以看出，默认情况下（所有的权重为1），各个Schedulable获得相等的资源量，当设置权重之后，权重则表示该Schedulable获得资源量的倍数。

但为了满足Schedulable的MinShare以及MaxShare，实际中ResourceAllocation需要做如下处理：

* 对于S.minShare > R * S.weight，分配S.minShare
* 对于S.maxShare < R * S.weight，分配S.maxShare
* 其他的，分配R * S.weight

故现在需要解决的是，对于给定的资源总量C，R取多大值使得能最大利用资源，并不超过资源总量。

计算实现在：*org.apache.hadoop.yarn.server.resourcemanager.scheduler.fair.policies.ComputeFairShares*


YARN的实现为采用二分近似法。

首先设R最小为0（即所有Schedulable都分配minShare)，然后再找一个合适的上限。

上限的方法为：

- 1. 设R=1
- 2. 计算当前资源使用量U
- 3. 若U < C，则 R = R * 2,跳转1
- 4. 结束计算

得到上限和下限之后，即可食用二分近似法，求解近似值。

设上限为Right，下限为Left，求解方法为：

- 1. R = (Right + Left )/2
- 2. 计算当前资源使用量U
- 3. 若U < C 则Left = R
- 4. 否则 Right = R
- 5. 若迭代次数 < 指定次数，转1,否则取Right值为最终值

从如上算法可知，得出的R将会使的资源总使用量稍大于集群的总资源量。

如上算法的复杂度为O(n) n为Schedulable的数量。

##资源分配策略

当前YARN中有三种分配策略：1）FIFO；2）FairShare；3)DFS

所有调度策略继承自SchedulingPolicy，该类为抽象类，提供了一些工厂方法，帮助产生实际的调度策略类。并由于调度策略类为无状态类，所以SchedulingPolicy还将会对这些调度策略具体的类进行缓存，实现单例模式。

SchedulingPolicy规定了调度策略需要处理的问题，主要有如下几个：

{% highlight java %}
void computeShares(Collection<? extends Schedulable> schedulables, Resource totalResources);

java.util.Comparator<Schedulable> getComparator();
{% endhighlight %}

computeShare将计算各Schedulable的FairShare，Comparator用于在资源分配过程中，根据各Schedulable的情况，决定各自优先级。

### FIFO 
即First IN,First Out,(FCFS或许更好)。

资源比较器比较步骤：

- 1. 首先比较优先级
- 2. 若优先级相等，比较开始时间
- 3. 若开始时间相等，按字典顺序比较名称

FairShare计算：

- 1. 选择提交时间最早的Schedulable eariestSc
- 2. eariestSc.fairShare = totalResource

FIFO仅仅考虑了优先级以及启动时间,在公平资源的分配时，将所有资源分配给最早提交的那个。

### FairShare

即公平调度器，FS调度目前仅仅考虑了内存。

资源比较器：
	
- 1. 资源未得到满足的优先级高于资源得到满足的（满足指已达到资源 >= 最小资源保障量）
- 2. 对于资源均未得到满足的，满足率越低，优先级越高（满足率=已使用资源量/最小资源保障量）
- 3. 对于资源均已得到满足的，资源使用权重比越低，优先级越高（资源使用权重比=已使用的内存量/权重）
- 4. 若上述均相等，则采用FIFO的比较方式比较

FairShare计算：采用‘资源公平性’中介绍的算法，计算各Schedulable的内存公平量
	
### DFS
即统治资源调度器，统治资源指在多种资源中，已使用的资源占总资源量比率最高的资源。
目前YARN资源只有CPU和内存两种，即非统治资源只有一种。

资源比较器：
	1. 统治资源未得到满足的优先级高于已得到满足的
	2. 若统治资源均已得到满足，统治资源集群占用率越低，则优先级越高（资源集群占用率=资源使用量/（集群资源总量*资源权重））
	3. 若统治资源集群占用率相等，非统治资源的集群占用率越低，则优先级越高
	4. 若资源集群占用率均相等，统治资源最小占用率越低，则优先级越高(资源最小占用率=资源使用量/最小保证资源量
	5. 若统治资源最小占用率相等，非统治资源最小占用率越低，则优先级越高
	6. 若均相等，开始时间越早，优先级越高。

FairShare计算：采用‘资源公平性’中的算法，分别计算CPU和内存的公平量。


## 队列分配流程
关注与将资源分配到某个叶子队列的某个App中。

父队列首先检查能否继续接受资源分配，然后选择一个子队列分配该节点资源。
子队列亦需要检查是否可以接受资源分配，通过检查之后，将资源分配给应用。
每次分配，仅分配一个Container。

检查条件为：

- 1. 要分配的节点上没有预留的Container
- 2. 当前队列已使用资源 <= 最大允许资源
	
子队列选择策略：

- 1. 子队列使用该队列调度策略的资源比较器，计算优先级，并按优先级从高到低排序
- 2. 队列按优先级从高到底，向子队列分配资源，直到分配成功或则所有子队列都分配失败
- 3. 子队列亦按此顺序进行分配，直到LeafQueue中。若分配成功，返回分配到的资源量，若失败，返回为0的资源量

应用选择策略：

- 1. 使用队列调度策略的资源比较器，对应用进行排序
- 2. 跳过当前状态不是RUNNABLE的App
- 3. 依次尝试将资源分配给应用，知道分配成功一个Container或所有应用均为分配资源
- 4. 成功返回分配的资源量；失败返回为0的资源量
	
>注：FS的队列在调度时，仅仅会更新使用资源等量，而不会想CS一样直接将所有调度器使用的导出量就行计算


## 应用分配流程

关注于将资源分配到该应用的某个Container中。

应用中，不需要检查是否能够分配资源。因为调度器仅仅会对队列和用户的资源使用量进行限制，但不会对应用能够使用的资源进行限制；或者说调度器对应用资源使用的限制是通过对队列和用户的限制，间接进行的。

Container分配需要考虑节点位置的问题，YARN调度器提供了三种不同的本地化级别：LOCAL，RACK_LOCAL以及OFF_SWITCH。LOCAL指Container分配的节点即为应用请求的节点，RACK_LOCAL指分配Container所在的节点与应用请求的节点在同一个机架内；OFF_SWITCH则是分配的和请求的不在同一个机架内。应用可以设置节点位置的松弛度以限制分配的节点位置，若例如禁止RACK_LOCAL，则表示只能再要求的节点上分配Container，其他节点资源不接受，当然这也降低了改应用能够得到资源的可能性。

同样，一次分配，最多分配一个Container。

首先需要找到合适的资源请求：

- 1. 根据应用中Container的优先级，从高到低进行寻找
- 2. 尝试在该节点上分配LOCAL类型的Container（需要该应用在该优先级上，有对该节点的资源请求）
- 3. 若不存在LOCAL，根据请求松弛度，依次尝试RACK_LOCAL以及OFF_SWITCH。

然后，进行Container分配：

- 1. 检查节点剩余资源是否满足请求
- 2. 满足则进行分配，并通知相应的应用以及节点分配的Container
- 3. 若不满足，则在该节点上进行资源预留

## 动态配置以及资源抢占

由于FS的FairShare计算代价较大，不能每次调度都重新计算。故，FS使用一个单独的现场周期性的计算并更新各个FairShare值。

该线程不仅仅只更新FairShare，还需要做如下事情：

* 动态更新FS配置
* 检查并处理队列和用户的使用上限
* 更新叶子队列资源满足情况
* 资源抢占处理

### 动态更新FS配置

FS会定期检查配置文件是否被修改，以确定是否需要重新加载配置文件。在选择加载时机时，若配置文件修改的时间大于FS上一次成功加载配置的时间，FS就会等待一个时间段并开始重新加载配置文件。等待一个时间段是为了降低当前配置文件尚未完全修改完成的可能性。

更新配置，首先读取allocateFile中的配置项，包括队列名称、资源和App限制、队列调度策略、权重以及ACLs。然后将读取到的新配置信息用于更新QueueManager中的参数，并创建当前不存在的叶子队列。



### 检查并处理各队列和用户的使用上限

FS中，运行状态RUNNABLE为False的App将不会参与资源调度，该处即使用这一特性进行上限处理。

步骤基本如下：

- 1. 将所有App状态设置为False，禁止所有调度
- 2. 将所有App使用FIFO策略进行排序
- 3. 遍历App列表
- 4. 若该App所属用户以及所属队列均为达到上限，则设置App Runnable为True，并更新相应用户和队列App数量
- 5. 直到所有App遍历结束

如上处理之后，将使得各个队列以及用户满足App数量限制，并且也照顾了App的优先级以及启动时间。


### 更新叶子队列资源满足情况

该处很简单，仅仅设置两个值：上一次达到MinShare的时间(LastTimeAtMinShare)，上一次达到FairShare一半的时间(LastTimeAtHalfFairShare)。这两个参数主要用于抢占判断。

遍历所有叶子队列，检查当前资源满足情况，若满足MinShare，则设置LastTimeAtMinShare = currentTime;若当前使用资源达到fairShare的一半，则设置LastTimeAtHalfFairShare = currentTime。

### 资源抢占处理

抢占默认是关闭的。

抢占采用周期方式，在每个周期中，首先计算需要抢占的资源量，然后进行抢占。

资源的抢占较为的“友好”，首先调度器会将需要抢占的Container告知AM，使得其尽快结束这些Container，若在指定的时间内，这些Container未退出，调度器将强制对其杀死并释放资源。

抢占资源量即为所有叶子队列中，需要抢占的资源量之和。对于各叶子队列，需要抢占的资源由两个因素造成：1）长期未得到miniShare；2）长期未达到fairShare的一半。是否为“长期”，判断依据即为上面设置的两个时间值与当前时间的差值是否超过了规定时间。若需要抢占，则该叶子队列需要抢占的资源量为当前得到资源量与minShare之间的差距和当前得到资源量与fairShare一半的差距中的最大值。

调度器首先将检查是否存在已告知抢占但未退出的Container，若存在则强制杀死以释放资源。此时若释放的资源小于需要抢占的资源时，就需要启动新一轮的抢占。

抢占步骤如下：

- 1. 获取所有占用资源大于FairShare的叶子队列中的Container
- 2. 对这些Container按优先级由低到高排序
- 3. 依次抢占这些Container，直到抢占的资源满足需要抢占的资源或这些Container全部抢走
