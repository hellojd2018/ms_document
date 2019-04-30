# Server Self Preservation Mode

# 自我保护模式

如果Eureka服务器检测到大于预期数量的已注册客户端以非正常方式终止其连接，并且同时正在等待驱逐，则它们将进入自我保护模式。这样做是为了确保灾难性网络事件不会消除eureka注册表数据，并将其传播到下游所有客户端。

为了更好地理解自我保护，首先要了解尤里卡客户如何“结束”他们的注册生命周期。eureka协议要求客户端在永久离开时执行显式取消注册操作。例如，在提供的java客户端中，这是在shutdown（）方法中完成的。任何连续三次心跳续订失败的客户都被认为有不干净的终止，并将被背景驱逐程序逐出。当当前注册表的大于15％处于此后状态时，将启用自我保留。

在自我保护模式下，eureka服务器将停止驱逐所有实例，直到：

1. 它看到的心跳续订数量已超过预期阈值，或
2. 自我保护被禁用（见下文）

默认情况下启用自我保留，启用自我保留的默认阈值大于当前注册表大小的15％。

## 如何配置自我保护阈值

自保护配置[在此处](https://github.com/Netflix/eureka/blob/master/eureka-core/src/main/java/com/netflix/eureka/DefaultEurekaServerConfig.java#L221)定义。要在示例中更改自我保留阈值，请设置属性：`eureka.renewalPercentThreshold=[0.0, 1.0]`。

## 如何禁用自我保护

自保护配置[在此处](https://github.com/Netflix/eureka/blob/master/eureka-core/src/main/java/com/netflix/eureka/DefaultEurekaServerConfig.java#L195)定义。要在示例中禁用自我保留，请设置属性：`eureka.enableSelfPreservation=false`。

在生产环境中，如果由于某种原因您的服务器由于正当理由而进入了自我保留状态，您可以通过暂时禁用通过配置进行自我保存来强制服务器退出自我保护模式。我们希望在这些情况下，人类行动正在评估情况并采取适当的行动。