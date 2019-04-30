# Understanding eureka client server communication

希望此时您已访问[此](https://github.com/Netflix/eureka/wiki/Configuring-Eureka)页面以了解如何设置Eureka服务器。

与Eureka Server交互的第一步是初始化Eureka客户端。如果您在AWS Cloud中运行，则可以通过以下方式初始化：

从1.1.153版开始，引入了EurekaModule类，允许使用eureka-client和governator / guice。请参阅此[受管制的示例](https://github.com/Netflix/eureka/blob/master/eureka-examples/src/main/java/com/netflix/eureka/ExampleEurekaGovernatedService.java)。

在1.1.153版之前，您可以通过以下方式初始化Eureka Client：

```
     DiscoveryManager.getInstance().initComponent(
                new CloudInstanceConfig(),
                new DefaultEurekaClientConfig());
```

如果您在其他数据中心中运行，则可以通过以下方式初始化：

```
     DiscoveryManager.getInstance().initComponent(
                new MyDataCenterInstanceConfig(),
                new DefaultEurekaClientConfig());
```

尤里卡客户端查找*eureka-client.properties*作为解释[在这里](https://github.com/Netflix/eureka/wiki/Configuring-Eureka#configuration)。

## About Instance Statuses

默认情况下，Eureka客户端在**STARTING**状态下启动，这使得实例有机会进行特定于应用程序的初始化，然后才能为流量提供服务。

然后，应用程序可以通过将实例状态转换为**UP**来明确地为流量设置实例。

```
     ApplicationInfoManager.getInstance().setInstanceStatus(InstanceStatus.UP)
```

应用程序还可以注册运行状况检查[回调](http://netflix.github.com/eureka/javadoc/eureka-client/index.html)，可以选择将实例状态更改为**DOWN**。

在Netflix，我们还使用**OUT_OF_SERVICE**状态，主要用于从流量中获取实例。它用于在出现问题时轻松回滚新版本的部署。大多数应用程序为新版本创建新的ASG，并且流量将路由到新的ASG。在出现问题的情况下，回滚修订版只需通过将ASG中的所有实例设置为**OUT_OF_SERVICE**来关闭流量。

## Eureka Client Operations

Eureka客户端首先尝试与AWS云中相同区域中的Eureka Server进行所有操作，如果找不到服务器，则会将其故障转移到其他区域。

应用程序客户端可以使用Eureka客户端返回的信息进行负载平衡。以下是使用Eureka客户端返回的信息对客户端进行负载平衡的示例应用程序示例。

```
 InstanceInfo nextServerInfo = DiscoveryManager.getInstance()
                .getDiscoveryClient()
                .getNextServerFromEureka(vipAddress, false);

        Socket s = new Socket();
        int serverPort = nextServerInfo.getPort();
        try {
            s.connect(new InetSocketAddress(nextServerInfo.getHostName(),
                    serverPort));
        } catch (IOException e) {
            System.err.println("Could not connect to the server :"
                    + nextServerInfo.getHostName() + " at port " + serverPort);
        }
```

如果基本的循环负载平衡不足以满足您的需求，您可以将负载均衡器包装在此处提供的[API /操作](https://github.com/Netflix/eureka/blob/master/eureka-client/src/main/java/com/netflix/discovery/EurekaClient.java)之上。在AWS云中，请确保重试故障并保持较低的超时，因为可能存在Eureka服务器可能返回在中断情况下不再存在的实例的情况。

值得注意的是，Eureka客户端会清除为服务器通信而创建的空闲超过30秒的HTTP连接。这是因为AWS防火墙限制不允许流量在几分钟的空闲时间后通过连接。

Eureka客户端通过以下方式与服务器交互。

## Register

Eureka客户端将有关正在运行的实例的信息注册到Eureka服务器。在AWS云中，可以通过访问URL *http://169.254.169.254/latest/metadata*获取有关实例的信息。注册发生在第一次心跳（30秒后）。

## Renew

Eureka客户需要每30秒发送一次心跳来续订租约。续订通知Eureka服务器该实例仍然存在。如果服务器没有看到续订90秒，它会将实例从其注册表中删除。建议不要更改续订间隔，因为服务器使用该信息来确定客户端到服务器通信是否存在广泛的问题。

## Fetch Registry

Eureka客户端从服务器获取注册表信息并在本地缓存它。之后，客户端使用该信息来查找其他服务。通过获取最后一个获取周期和当前获取周期之间的增量更新，定期（每30秒）更新此信息。增量信息在服务器中保持更长时间（约3分钟），因此增量提取可能会再次返回相同的实例。Eureka客户端自动处理重复信息。

获得增量后，Eureka客户端通过比较服务器返回的实例计数来协调与服务器的信息，如果信息由于某种原因不匹配，则再次获取整个注册表信息。Eureka服务器缓存增量的压缩有效负载，整个注册表以及每个应用程序以及相同的未压缩信息。有效负载还支持JSON / XML格式。Eureka客户端使用jersey apache客户端以压缩的JSON格式获取信息。

## Cancel

Eureka客户端在关机时向Eureka服务器发送取消请求。这将从服务器的实例注册表中删除实例，从而有效地将实例从流量中删除。

这是在Eureka客户端关闭时完成的，应用程序应确保在关闭期间调用以下内容。

```
     DiscoveryManager.getInstance().shutdownComponent()
```

## Time Lag

来自Eureka客户端的所有操作可能需要一些时间才能反映在Eureka服务器中以及随后的其他Eureka客户端中。这是因为eureka服务器上的有效负载缓存会定期刷新以反映新信息。Eureka客户还定期获取增量。因此，更改传播到所有Eureka客户端可能需要2分钟。

## Communication mechanism

默认情况下，Eureka客户端使用[Jersey](http://jersey.java.net/)和Jackson以及JSON有效负载与Eureka Server进行通信。您可以通过[覆盖](http://netflix.github.com/eureka/javadoc/eureka-client/index.html)默认机制来使用您选择的机制。请注意，对于某些遗留用例，[XStream](http://xstream.codehaus.org/)也是依赖关系图的一部分。