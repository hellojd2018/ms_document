# 在Spring Cloud中实现降级之权重路由和标签路由

**前言** 限流、降级、灰度是服务治理的一个很重要的功能。本文参考[Spring Cloud中国社区的VIP会员-何鹰的博客-整理](http://www.jianshu.com/p/37ee1e84900a)
Dubbo自带服务降级、限流功能，spring cloud并没有提供此功能，只能由我们自行实现。这里的限流、降级、灰度都是针对服务实例级别，并不是整个服务级别，整个服务级别可以通过实例部署数量来实现。

## 限流降级设计

### 场景

服务A，部署了3个实例A1、A2、A3。spring cloud默认客户端负载均衡策略是采用轮询方式，A1、A2、A3三个实例流量均分，各1/3。如果这个时候需要将服务A由1.0版升级至2.0版，我们需要做的步骤是：将A1的流量降为0，柔性下线，关闭A1实例并升级到2.0，将A1流量提升为10%观察2.0线上运行情况，如果情况稳定，则逐步开放流量至不限制及1/3。依次在A2，A3上执行上述操作。
在上述步骤中，我们想让特别的人使用2.0，其他人还是使用1.0版，稳定后再全员开放。

### 思路

分析，服务A的流量产生有两个方面，一个是外部流量，外网通过zuul过来的流量，一个是内部流量，服务间调用，服务B调用服务A的这类流量。不管是zuul还是内部服务来的，都是要通过ribbon做客户端负载均衡，我们可以修改ribbon负载均衡策略来实现上述限流、降级、灰度功能。

要实现这些想法，我们需要对spring-cloud的各个组件、数据流非常熟悉，这样才能知道该在哪里做扩展。一个典型的调用：外网-》Zuul网关-》服务A-》服务B。。。

spring-cloud跟dubbo一样都是客户端负载均衡，所有调用均由Ribbon来做负载均衡选择服务器，所有调用前后会套一层hystrix做隔离、熔断。服务间调用均用带LoadBalanced注解的RestTemplate发出。RestTemplate-》Ribbon-》hystrix

通过上述分析我们可以看到，我们的扩展点就在Ribbon，Ribbon根据我们的规则，选择正确的服务器即可。

我们先来一个dubbo自带的功能：基于权重的流量控制。dubbo自带的控制台可以设置服务实例粒度的半权，倍权。其实就是在客户端负载均衡时，选择服务器带上权重即可，spring-cloud默认是ZoneAvoidanceRule，优先选择相同Zone下的实例，实例间采用轮询方式做负载均衡。我们的想把基于轮询改为基于权重即可。接下来的问题是，每个实例的权重信息保存在哪里？从哪里取？dubbo放在zookeeper中，spring-cloud放在eureka中。我们只需从eureka拿每个实例的权重信息，然后根据权重来选择服务器即可。具体代码LabelAndWeightMetadataRule（先忽略里面的优先匹配label相关代码）。

## 工程案例演示

[![工程目录](http://xujin.org/images/sc-study/sc-r-d02.png)](http://xujin.org/images/sc-study/sc-r-d02.png)工程目录

> <https://github.com/SoftwareKing/spring-cloud-study/tree/master/sc-ribbon-demoted>

### 项目结构

1. config 配置中心
   端口：8888，方便起见直接读取配置文件，生产环境可以读取git。application-dev.properties为全局配置。先启动配置中心，所有服务的配置（包括注册中心的地址）均从配置中心读取。
2. consumer 服务消费者
   端口：18090，调用服务提供者，为了演示header传递。
3. core 框架核心包
   核心jar包，所有微服务均引用该包，使用AutoConfig实现免配置，模拟生产环境下spring-cloud的使用。
4. eureka 注册中心
   端口：8761，/metadata端点实现metadata信息配置。
5. provider 服务提供者
   端口：18090，服务提供者，无特殊逻辑。
6. zuul 网关
   端口：8080，演示解析token获得label并放入header往后传递

## 案例具体实现

### 基于权重的实现思路

LabelAndWeightMetadataRule写好了，那么我们如何使用它，使之生效呢？有3种方式。

1）写个AutoConfig将LabelAndWeightMetadataRule声明成@Bean，用来替换默认的ZoneAvoidanceRule。这种方式在技术验证、开发测试阶段使用短平快。但是这种方式是强制全局设置，无法个性化。

2）由于spring-cloud的Ribbon并没有实现netflix Ribbon的所有配置项。netflix配置全局rule方式为：ribbon.NFLoadBalancerRuleClassName=package.YourRule，spring-cloud并不支持，spring-cloud直接到服务粒度，只支持SERVICE_ID.ribbon.NFLoadBalancerRuleClassName=package.YourRule。

> 我们可以扩展org.springframework.cloud.netflix.ribbon.PropertiesFactory修正spring cloud ribbon未能完全支持netflix ribbon配置的问题。这样我们可以将全局配置写到配置中心的application-dev.properties全局配置中，然后各个微服务还可以根据自身情况做个性化定制。但是PropertiesFactory属性均为私有，应该是spring cloud不建议在此扩展。参见<https://github.com/spring-cloud/spring-cloud-netflix/issues/1741。>

3）使用spring cloud官方建议的@RibbonClient方式。该方式仅存在于spring-cloud单元测试中（在我提问后，现在还存在于spring-cloud issue list）。具体代码参见DefaultRibbonConfiguration.java、CoreAutoConfiguration.java。

> 目前采用第三种方式处理

### 基于权重的路由测试

依次开启 config eureka provide（开两个实例，通过启动参数server.port指定不同端口区分） consumer zuul
访问 <http://localhost:8761/metadata.html> 这是我手写的一个简单的metadata管理界面，分别设置两个provider实例的weight值（设置完需要一段2分钟才能生效），然后访问 <http://localhost:8080/provider/user>多刷几次来测试zuul是否按权重发送请求，也可以访问 <http://localhost:8080/consumer/test> 多刷几次来测试consumer是否按权重来调用provide服务。

### 基于标签的路由处理

基于权重的搞定之后，接下来才是重头戏：基于标签的路由。入口请求含有各种标签，然后我们可以根据标签幻化出各种各样的路由规则。例如只有标注为粉丝的用户才使用新版本（灰度、AB、金丝雀），例如标注为中国的用户请求必须发送到中国的服务器（全球部署），例如标注为写的请求必须发送到专门的写服务实例（读写分离），等等等等，唯一限制你的就是你的想象力。

#### 基于标签的路由实现思路

根据标签的控制，我们当然放到之前写的Ribbon的rule中，每个实例配置的不同规则也是跟之前一样放到注册中心的metadata中。需要解决以下几个问题:

**Q:关键是标签数据如何传过来?**

> A:权重随机的实现思路里面有答案，请求都通过zuul进来，因此我们可以在zuul里面给请求打标签，基于用户，IP或其他看你的需求，然后将标签信息放入ThreadLocal中，然后在Ribbon Rule中从ThreadLocal拿出来使用就可以了。
>
> 然而，按照这个方式去实验时，发现有问题，拿不到ThreadLocal。原因是有hystrix这个东西，回忆下hystrix的原理，为了做到故障隔离，hystrix启用了自己的线程，不在同一个线程ThreadLocal失效。

那么还有什么办法能够将标签信息一传到底呢，想想之前有没有人实现过类似的东西，没错sleuth，它的链路跟踪就能够将span传递下去，翻翻sleuth源码，找找其他资料，发现可以使用HystrixRequestVariableDefault，这里不建议直接使用HystrixConcurrencyStrategy，会和sleuth的strategy冲突。代码参见CoreHeaderInterceptor.java。现在可以测试zuul里面的rule，看能否拿到标签内容了。

> 标签传到HystrixRequestVariableDefault这里的，如果项目中没有使用Hystrix就用不了了,这个时候需要做一个判断在restTemple里面做个判断，没有hystrix就直接threadlocal取。

**Q:这里还不是终点，解决了zuul的路由，服务A调服务B这里的路由怎么处理呢？zuul算出来的标签如何往后面依次传递下去呢?**

我们还是抄sleuth：把标签放入header，服务A调服务B时，将服务A header里面的标签放到服务B的header里，依次传递下去。这里的关键点就是：内部的微服务在接收到发来的请求时(zuul->A，A->B）我们将请求放入ThreadLocal，哦，不对，是HystrixRequestVariableDefault，还记得上面说的原因么：）。
这个容易处理，写一个spring mvc拦截器即可，代码参见CoreHeaderInterceptor。然后发送请求时自动带上这个里面保存的标签信息，参见RestTemplate的拦截器CoreHttpRequestInterceptor。到此为止，技术上全部走通实现。

> 总结一下：zuul依据用户或IP等计算标签，并将标签放入header里向后传递，后续的微服务通过拦截器，将header里的标签放入RestTemplate请求的header里继续向后接力传递。标签的内容通过放入类似于ThreadLocal的全局变量（HystrixRequestVariableDefault），使Ribbon Rule可以使用。

### 基于标签路由的测试

参见PreFilter源码，模拟了几个用户的标签，参见LabelAndWeightMetadataRule源码，模拟了OR AND两种标签处理策略。依次开启 config eureka provide（开两个实例，通过启动参数server.port指定不同端口区分） consumer zuul.

[![测试](http://xujin.org/images/sc-study/sc-r-d03.png)](http://xujin.org/images/sc-study/sc-r-d03.png)测试

------

[![测试](http://xujin.org/images/sc-study/sc-r-d01.png)](http://xujin.org/images/sc-study/sc-r-d01.png)测试
访问 <http://localhost:8761/metadata.html> 设置第一个provide 实例 orLabel为 CN,Test 发送请求头带入Authorization: emt 访问<http://localhost:8080/provider/user> 多刷几次，可以看到zuul所有请求均路由给了第一个实例。访问<http://localhost:8080/consumer/test> 多刷几次，可以看到，consumer调用均路由给了第一个实例。

设置第二个provide 实例 andLabel为 EN,Male 发送请求头带入Authorization: em 访问<http://localhost:8080/provider/user> 多刷几次，可以看到zuul所有请求均路由给了第二个实例。访问<http://localhost:8080/consumer/test> 多刷几次，可以看到，consumer调用均路由给了第二个实例。

Authorization头还可以设置为PreFilter里面的模拟token来做测试，至此所有内容讲解完毕，技术路线拉通，剩下的就是根据需求来完善你自己的路由策略啦。

## 伪代码分析实现流程

### 伪代码示例

Ribbon默认采用ZoneAvoidanceRule，优先选择同zone下的实例。我们继承这个rule并扩展我们自己的限流功能，仔细阅读ZoneAvoidanceRule及其父类源码。

```
public class WeightedMetadataRule extends ZoneAvoidanceRule {
public static final String META_DATA_KEY_WEIGHT = "weight";

@Override
public Server choose(Object key) {
    List<Server> serverList = this.getPredicate().getEligibleServers(this.getLoadBalancer().getAllServers(), key);
    if (CollectionUtils.isEmpty(serverList)) {
        return null;
    }

    // 计算总值并剔除0权重节点
    int sum = 0;
    Map<Server, Integer> serverWeightMap = new HashMap<>();
    for (Server server : serverList) {
        String strWeight = ((DiscoveryEnabledServer) server).getInstanceInfo().getMetadata().get(META_DATA_KEY_WEIGHT);

        int weight = 100;
        try {
            weight = Integer.parseInt(strWeight);
        } catch (Exception e) {
            // 无需处理
        }

        if (weight <= 0) {
            continue;
        }

        serverWeightMap.put(server, weight);
        sum += weight;
    }

    // 权重随机
    int random = (int) (Math.random() * sum);
    int current = 0;
    for (Map.Entry<Server, Integer> entry : serverWeightMap.entrySet()) {
        current += entry.getValue();
        if (random < current) {
            return entry.getKey();
        }
    }

    return null;
}
}
```

使上述代码生效，在zuul网关中加入

```
@Bean
public IRule weightedMetadataRule(){
   return new WeightedMetadataRule();
}
```

### 代码示例测试

打断点测试是否进入WeightedMetadataRule，开启多个服务A实例，通过zuul访问服务A。
成功进入断点，代码生效后，我们再来看如何指定metadata。
访问eureka restful API （我的eureka服务器端口为8100，修改为你自己的eureka端口）
Get <http://localhost:8100/eureka/apps>
这个api可以看到所有服务
Get <http://localhost:8100/eureka/apps/YOUR_SERVICE_NAME>
这个api可以看到你的服务信息，包括部署了哪些实例
Get <http://localhost:8100/eureka/apps/YOUR_SERVICE_NAME/INSTANCE_ID>
这个api可以看到服务实例的信息，注意其中的metadata节点，目前为empty
Put <http://localhost:8100/eureka/apps/YOUR_SERVICE_NAME/INSTANCE_ID/metadata?weight=10>
通过put方式可以修改metadata的内容，放入weight，设为10

然后稍等两分钟，让zuul更新注册中心中的信息，接着重新访问，调试就可以看到metadata的内容了，并且也是按照权重随机来进行流量限制的，至此hello world搞定。

### 生产上使用WeightedMetadataRule

接下来，在生产环境中，我们如何应用这个WeightedMetadataRule呢，有如下几种方式：

### 手动指定服务策略，

spring cloud ribbon并没有完整实现netflix ribbon的所有配置功能，负载策略默认只能配置微服务级别，无法配置全局默认值。
例如：只能配置 SOME_SERVICE_ID.ribbon.NFLoadBalancerRuleClassName=package.WeightedMetadataRule
而不支持配置全局默认值 ribbon.NFLoadBalancerRuleClassName=package.WeightedMetadataRule
这种方案明显不符合我们的要求。

### 通过声明Irule spring bean配置全局负载策略

```
@Bean
public IRule weightedMetadataRule(){
   return new WeightedMetadataRule();
}
```

> 这种方式也就是我们上面用的hello world方式，配置后强制所有微服务使用该策略，没有例外，微服务无法个性化定制策略，符合目前需求，但不适于长期规划。

### 继承重写PropertiesFactory

继承重写org.springframework.cloud.netflix.ribbon.PropertiesFactory类，修正spring cloud ribbon未能完全支持netflix ribbon的问题。但是PropertiesFactory属性均为私有，应该是spring cloud不建议在此扩展。参见<https://github.com/spring-cloud/spring-cloud-netflix/issues/1741>

### 使用spring cloud官方建议的@RibbonClient方式

```
@Configuration
@RibbonClients(defaultConfiguration = DefaultRibbonConfiguration.class)
public class DefaultRibbonConfiguration {

    @Value("${ribbon.client.name:#{null}}")
    private String name;

    @Autowired(required = false)
    private IClientConfig config;

    @Autowired
    private PropertiesFactory propertiesFactory;

    @Bean
    public IRule ribbonRule() {
        if (StringUtils.isEmpty(name)) {
            return null;
        }

        if (this.propertiesFactory.isSet(IRule.class, name)) {
            return this.propertiesFactory.get(IRule.class, config, name);
        }

        // 默认配置
        WeightedMetadataRule rule = new WeightedMetadataRule();
        rule.initWithNiwsConfig(config);
        return rule;
    }
}
```

## 总结

关于权重随机的性能，上述代码用的数组分段查找法，还可以采用TreeMap二分查找法。可以将权重数组或权重TreeMap缓存起来。
根据测试，在实例数量为50个时 缓存权重数组和权重TreeMap，数组分段查找百万次耗时78-125ms，TreeMap二分耗时50-80ms。

这篇文章只是把技术打通，至于如何根据服务器负载情况，自动降级，限流等需求，只需要监控服务器状况，调用eureka接口设置metadata即可（其实我个人建议这方面需求通过docker的自动扩容缩容完成，只是有朋友问到如何通过spring cloud实现）。

下一篇会写基于标签的流量控制。如何控制部分用户使用服务A2.0，其他用户使用服务A1.0。