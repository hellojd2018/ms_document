# 微服务注册中心 Eureka 架构深入解读

微服务架构中最核心的部分是服务治理，服务治理最基础的组件是注册中心。随着微服务架构的发展，出现了很多微服务架构的解决方案，其中包括我们熟知的 Dubbo 和 Spring Cloud。

关于注册中心的解决方案，dubbo 支持了 Zookeeper、Redis、Multicast 和 Simple，官方推荐 Zookeeper。Spring Cloud 支持了 Zookeeper、Consul 和 Eureka，官方推荐 Eureka。

两者之所以推荐不同的实现方式，原因在于组件的特点以及适用场景不同。简单来说：

- ZK 的设计原则是 CP，即强一致性和分区容错性。他保证数据的强一致性，但舍弃了可用性，**如果出现网络问题可能会影响 ZK 的选举，导致 ZK 注册中心的不可用**。
- Eureka 的设计原则是 AP，即可用性和分区容错性。他保证了注册中心的可用性，但舍弃了数据一致性，**各节点上的数据有可能是不一致的（会最终一致）**。
   
  Eureka 采用纯 Java 实现，除实现了注册中心基本的服务注册和发现之外，极大的满足注册中心的可用性，即使只有一台服务可用，也可以保证注册中心的可用性。

本文将聚焦到 Eureka 的内部实现原理，先从微服务架构的部署图介绍 Eureka 的总体架构，然后剖析服务信息的存储结构，最后探究跟服务生命周期相关的服务注册机制、服务续约机制、服务注销机制、服务剔除机制、服务获取机制、和服务同步机制。

## Eureka 总体架构

下面是 Eureka 注册中心部署在多个机房的架构图，这正是他高可用性的优势（Zookeeper 千万别这么部署）。

![img](https://static001.infoq.cn/resource/image/8b/8b/8bf6e27c60dbfd717b6830263890368b.png)

从组件功能看：

- 黄色注册中心集群，分别部署在北京、天津、青岛机房；
- 红色服务提供者，分别部署北京和青岛机房；
- 淡绿色服务消费者，分别部署在北京和天津机房；

从机房分布看：

- 北京机房部署了注册中心、服务提供者和服务消费者；
- 天津机房部署了注册中心和服务消费者；
- 青岛机房部署了注册中心和服务提供者；

### 组件调用关系

**服务提供者**

1. 启动后，向注册中心发起 register 请求，注册服务
2. 在运行过程中，定时向注册中心发送 renew 心跳，证明“我还活着”。
3. 停止服务提供者，向注册中心发起 cancel 请求，清空当前服务注册信息。

**服务消费者**

1. 启动后，从注册中心拉取服务注册信息
2. 在运行过程中，定时更新服务注册信息。
3. 服务消费者发起远程调用：
   a> 服务消费者（北京）会从服务注册信息中选择同机房的服务提供者（北京），发起远程调用。只有同机房的服务提供者挂了才会选择其他机房的服务提供者（青岛）。
   b> 服务消费者（天津）因为同机房内没有服务提供者，则会按负载均衡算法选择北京或青岛的服务提供者，发起远程调用。

**注册中心**

1. 启动后，从其他节点拉取服务注册信息。
2. 运行过程中，定时运行 evict 任务，剔除没有按时 renew 的服务（包括非正常停止和网络故障的服务）。
3. 运行过程中，接收到的 register、renew、cancel 请求，都会同步至其他注册中心节点。
    
   本文将详细说明上图中的 registry、register、renew、cancel、getRegistry、evict 的内部机制。

## 数据存储结构

既然是服务注册中心，必然要存储服务的信息，我们知道 ZK 是将服务信息保存在树形节点上。而下面是 Eureka 的数据存储结构：

![img](https://static001.infoq.cn/resource/image/61/2a/61a26fffaeaf63d5dd5c0c2c5dd7852a.png)

Eureka 的数据存储分了两层：数据存储层和缓存层。

Eureka Client 在拉取服务信息时，先从缓存层获取（相当于 Redis），如果获取不到，先把数据存储层的数据加载到缓存中（相当于 Mysql），再从缓存中获取。值得注意的是，数据存储层的数据结构是服务信息，而缓存中保存的是经过处理加工过的、可以直接传输到 Eureka Client 的数据结构。
 
Eureka 这样的数据结构设计是把内部的数据存储结构与对外的数据结构隔离开了，就像是我们平时在进行接口设计一样，对外输出的数据结构和数据库中的数据结构往往都是不一样的。

**数据存储层**

这里为什么说是存储层而不是持久层？因为 rigistry 本质上是一个双层的 ConcurrentHashMap，存储在内存中的。

- 第一层的 key 是`spring.application.name`，value 是第二层 ConcurrentHashMap；
- 第二层 ConcurrentHashMap 的 key 是服务的 InstanceId，value 是 Lease 对象；
- Lease 对象包含了服务详情和服务治理相关的属性。
   
  **二级缓存层**

Eureka 实现了二级缓存来保存即将要对外传输的服务信息，数据结构完全相同。

- 一级缓存：`ConcurrentHashMap<Key,Value> readOnlyCacheMap`，本质上是 HashMap，无过期时间，保存服务信息的对外输出数据结构。
- 二级缓存：`Loading<Key,Value> readWriteCacheMap`，本质上是 guava 的缓存，包含失效机制，保存服务信息的对外输出数据结构。

既然是缓存，那必然要有更新机制，来保证数据的一致性。下面是缓存的更新机制：

![img](https://static001.infoq.cn/resource/image/2d/c3/2dcf60f2fa8ec888f2b3262e6e9de5c3.png)

更新机制包含删除和加载两个部分，上图黑色箭头表示删除缓存的动作，绿色表示加载或触发加载的动作。

**删除二级缓存：**

1. Eureka Client 发送 register、renew 和 cancel 请求并更新 registry 注册表之后，删除二级缓存；
2. Eureka Server 自身的 Evict Task 剔除服务后，删除二级缓存；
3. 二级缓存本身设置了 guava 的失效机制，隔一段时间后自己自动失效；

**加载二级缓存：**

1. Eureka Client 发送 getRegistry 请求后，如果二级缓存中没有，就触发 guava 的 load，即从 registry 中获取原始服务信息后进行处理加工，再加载到二级缓存中。
2. Eureka Server 更新一级缓存的时候，如果二级缓存没有数据，也会触发 guava 的 load。

**更新一级缓存：**

1. Eureka Server 内置了一个 TimerTask，定时将二级缓存中的数据同步到一级缓存（这个动作包括了删除和加载）。
    
   关于缓存的实现参考 ResponseCacheImpl

### 服务注册机制

服务提供者、服务消费者、以及服务注册中心自己，启动后都会向注册中心注册服务（如果配置了注册）。下图是介绍如何完成服务注册的：

![img](https://static001.infoq.cn/resource/image/11/18/1120a85e1270610c0ac1eeba9e1d5718.png)

注册中心服务接收到 register 请求后：

1. 保存服务信息，将服务信息保存到 registry 中；
2. 更新队列，将此事件添加到更新队列中，供 Eureka Client 增量同步服务信息使用。
3. 清空二级缓存，即 readWriteCacheMap，用于保证数据的一致性。
4. 更新阈值，供剔除服务使用。
5. 同步服务信息，将此事件同步至其他的 Eureka Server 节点。

### 服务续约机制

服务注册后，要定时（默认 30S，可自己配置）向注册中心发送续约请求，告诉注册中心“我还活着”。

![img](https://static001.infoq.cn/resource/image/d5/44/d54cd75f968b71a37cbf3139ecfa6a44.png)

注册中心收到续约请求后：

1. 更新服务对象的最近续约时间，即 Lease 对象的 lastUpdateTimestamp;
2. 同步服务信息，将此事件同步至其他的 Eureka Server 节点。
    
   剔除服务之前会先判断服务是否已经过期，判断服务是否过期的条件之一是续约时间和当前时间的差值是不是大于阈值。

### 服务注销机制

服务**正常停止**之前会向注册中心发送注销请求，告诉注册中心“我要下线了”。

![img](https://static001.infoq.cn/resource/image/81/8e/8112c5bf591b4a32519f142220c2c28e.png)

注册中心服务接收到 cancel 请求后：

1. 删除服务信息，将服务信息从 registry 中删除；
2. 更新队列，将此事件添加到更新队列中，供 Eureka Client 增量同步服务信息使用。
3. 清空二级缓存，即 readWriteCacheMap，用于保证数据的一致性。
4. 更新阈值，供剔除服务使用。
5. 同步服务信息，将此事件同步至其他的 Eureka Server 节点。
    
   服务正常停止才会发送 Cancel，如果是非正常停止，则不会发送，此服务由 Eureka Server 主动剔除。

### 服务剔除机制

Eureka Server 提供了服务剔除的机制，用于剔除没有正常下线的服务。

![img](https://static001.infoq.cn/resource/image/bd/60/bd003c6b9ddc4f13840443ca04920560.png)

服务的剔除包括三个步骤，首先判断是否满足服务剔除的条件，然后找出过期的服务，最后执行剔除。
 
**判断是否满足服务剔除的条件**

有两种情况可以满足服务剔除的条件：

1. 关闭了自我保护
2. 如果开启了自我保护，需要进一步判断是 Eureka Server 出了问题，还是 Eureka Client 出了问题，如果是 Eureka Client 出了问题则进行剔除。

这里比较核心的条件是自我保护机制，Eureka 自我保护机制是为了防止误杀服务而提供的一个机制。Eureka 的自我保护机制“谦虚”的认为如果大量服务都续约失败，则认为是自己出问题了（如自己断网了），也就不剔除了；反之，则是 Eureka Client 的问题，需要进行剔除。而**自我保护阈值是区分 Eureka Client 还是 Eureka Server 出问题的临界值：如果超出阈值就表示大量服务可用，少量服务不可用，则判定是 Eureka Client 出了问题。如果未超出阈值就表示大量服务不可用，则判定是 Eureka Server 出了问题**。

条件 1 中如果关闭了自我保护，则统统认为是 Eureka Client 的问题，把没按时续约的服务都剔除掉（这里有剔除的最大值限制）。
 
这里比较难理解的是阈值的计算：

- 自我保护阈值 = 服务总数 * 每分钟续约数 * 自我保护阈值因子。
- 每分钟续约数 =（60S/ 客户端续约间隔）

最后自我保护阈值的计算公式为：

自我保护阈值 = 服务总数 * （60S/ 客户端续约间隔） * 自我保护阈值因子。
 
**举例**：如果有 100 个服务，续约间隔是 30S，自我保护阈值 0.85。

自我保护阈值 =100 * 60 / 30 * 0.85 = 170。

如果上一分钟的续约数 =180>170，则说明大量服务可用，是服务问题，进入剔除流程；

如果上一分钟的续约数 =150<170，则说明大量服务不可用，是注册中心自己的问题，进入自我保护模式，不进入剔除流程。
 
**找出过期的服务**

遍历所有的服务，判断上次续约时间距离当前时间大于阈值就标记为过期。并将这些过期的服务保存到集合中。
 
**剔除服务**

在剔除服务之前先计算剔除的数量，然后遍历过期服务，通过洗牌算法确保每次都公平的选择出要剔除的任务，最后进行剔除。

执行剔除服务后：

1. 删除服务信息，从 registry 中删除服务。
2. 更新队列，将当前剔除事件保存到更新队列中。
3. 清空二级缓存，保证数据的一致性。
    
   实现过程参考 AbstractInstanceRegistry.evict() 方法。

### 服务获取机制

Eureka Client 获取服务有两种方式，全量同步和增量同步。获取流程是根据 Eureka Server 的多层数据结构进行的：

![img](https://static001.infoq.cn/resource/image/7a/b6/7a28cef8df4ee96193ae1fa36aca34b6.png)

无论是全量同步还是增量同步，都是先从缓存中获取，如果缓存中没有，则**先加载到缓存中，再从缓存中获取。（registry 只保存数据结构，缓存中保存 ready 的服务信息。）**

1. 先从一级缓存中获取
   a> 先判断是否开启了一级缓存
   b> 如果开启了则从一级缓存中获取，如果存在则返回，如果没有，则从二级缓存中获取
   d> 如果未开启，则跳过一级缓存，从二级缓存中获取
2. 再从二级缓存中获取
   a> 如果二级缓存中存在，则直接返回；
   b> 如果二级缓存中不存在，则先将数据加载到二级缓存中，再从二级缓存中获取。注意加载时需要判断是增量同步还是全量同步，增量同步从 recentlyChangedQueue 中 load，全量同步从 registry 中 load。

### 服务同步机制

服务同步机制是用来同步 Eureka Server 节点之间服务信息的。它包括 Eureka Server 启动时的同步，和运行过程中的同步。

**启动时同步**

![img](https://static001.infoq.cn/resource/image/04/68/0485b748af4f3bf0061672b9c13e0b68.png)

Eureka Server 启动后，遍历 eurekaClient.getApplications 获取服务信息，并将服务信息注册到自己的 registry 中。

注意这里是两层循环，第一层循环是为了保证已经拉取到服务信息，第二层循环是遍历拉取到的服务信息。
 
**运行过程中同步**

![img](https://static001.infoq.cn/resource/image/b6/64/b612ee78ac0703851b59bac9ca3b3564.png)

当 Eureka Server 节点有 register、renew、cancel 请求进来时，会将这个请求封装成 TaskHolder 放到 acceptorQueue 队列中，然后经过一系列的处理，放到 batchWorkQueue 中。

`TaskExecutor.BatchWorkerRunnable`是个线程池，不断的从 batchWorkQueue 队列中 poll 出 TaskHolder，然后向其他 Eureka Server 节点发送同步请求。
 
这里省略了两个部分：

- 一个是在 acceptorQueue 向 batchWorkQueue 转化时，省略了中间的 processingOrder 和 pendingTasks 过程。
- 另一个是当同步失败时，会将失败的 TaskHolder 保存到 reprocessQueue 中，重试处理。

## 写在最后

对微服务解决方案 Dubbo 和 Spring Cloud 的对比非常多，这里对注册中心做个简单对比。

![img](https://static001.infoq.cn/resource/image/ef/c4/ef69164a4afb2198cf4468c82f4a6cc4.png)

|          | Zookeeper                                        | Eureka                                                       |
| :------- | :----------------------------------------------- | :----------------------------------------------------------- |
| 设计原则 | CP                                               | AP                                                           |
| 优点     | 数据强一致                                       | 服务高可用                                                   |
| 缺点     | 网络分区会影响 Leader 选举，超过阈值后集群不可用 | 服务节点间的数据可能不一致； Client-Server 间的数据可能不一致； |
| 适用场景 | 单机房集群，对数据一致性要求较高                 | 云机房集群，跨越多机房部署；对注册中心服务可用性要求较高     |