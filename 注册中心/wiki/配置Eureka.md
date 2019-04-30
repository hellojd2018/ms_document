# Configuring Eureka

阅读[Eureka-at-a-glance页面](https://github.com/Netflix/eureka/wiki/Eureka-at-a-glance)以更好地理解设置的概念。

Eureka有两个组件**--Eureka Client**和**Eureka Server**。使用Eureka的架构通常有两个应用程序

- **应用程序客户端**使用Eureka客户端向应用程序服务发出请求。
- **应用程序服务**，它接收来自Application Client的请求并发回响应。

设置涉及以下内容

- 尤里卡服务器
- 应用程序客户端的Eureka客户端
- Eureka客户端的应用程序服务

Eureka可以在AWS和非AWS环境中运行。

**如果您在云环境中运行，则需要传入java命令行属性-Deureka.datacenter = cloud，以便Eureka客户端/服务器知道初始化特定于AWS云的信息。**

# 配置Eureka客户端

## 先决条件

- JDK 1.8或更高版本

您有以下选择来获取Eureka客户端二进制文件。总是尝试获得最新版本，因为往往有更多的修复。

- 您可以使用此URL“ [http://search.maven.org/#search%7Cga%7C1%7Ceureka-client](http://search.maven.org/#search|ga|1|eureka-client) ” 下载Eureka Client二进制文件
- 您可以将eureka客户端添加为maven依赖项

```
< 依赖 >
  < groupId > com.netflix.eureka </ groupId >
  < artifactId > eureka-client </ artifactId >
  < version > 1.1.16 </ version >
 </ dependency >
```

- 您可以按[此处](https://github.com/Netflix/eureka/wiki/Building-Eureka-Client-and-Server)指定构建客户端。

## 组态

配置Eureka客户端的最简单方法是使用属性文件。默认情况下，Eureka客户端在*类路径中*搜索属性文件*eureka-client.properties*。它还在特定于环境的属性文件中搜索特定于环境的覆盖。环境通常是*测试*或*产品*，由*-Deureka.environment* java命令行切换提供给*eureka*客户端（不带*.properties*后缀）。因此，客户端还搜索*eureka-client- {test，prod} .properties。*

您可以在[此处](https://github.com/Netflix/eureka/tree/master/eureka-examples/conf) 查看默认配置的示例。您可以复制这些配置并根据需要进行编辑，并将它们放在类路径中。如果由于某种原因想要更改属性文件的名称，可以通过 在java命令行开关中指定*-Deureka.client.props =*（不带后缀）来实现。 是要搜索的属性文件的名称。

文件中的属性说明了它们的用途。至少需要配置以下内容：

```
Application Name (eureka.name)
Application Port (eureka.port)
Virtual HostName (eureka.vipAddress)
Eureka Service Urls (eureka.serviceUrls)
```

有关更高级的配置，请查看以下链接中提供的选项。[https://github.com/Netflix/eureka/blob/master/eureka-client/src/main/java/com/netflix/appinfo/EurekaInstanceConfig.java ](https://github.com/Netflix/eureka/blob/master/eureka-client/src/main/java/com/netflix/appinfo/EurekaInstanceConfig.java)[https://github.com/Netflix/eureka/blob/master /eureka-client/src/main/java/com/netflix/discovery/EurekaClientConfig.java](https://github.com/Netflix/eureka/blob/master/eureka-client/src/main/java/com/netflix/discovery/EurekaClientConfig.java)

# 配置Eureka服务器

## 先决条件

- JDK 1.8或更高版本
- Tomcat 6.0.10或更高版本

使用Eureka服务器，您可以通过以下选项获取二进制文件

- 您可以从[此处](https://github.com/Netflix/eureka/wiki/Building-Eureka-Client-and-Server)指定的源构建WAR归档文件。
- 您可以使用此URL 
  “ [http://search.maven.org/#search%7Cga%7C1%7Ceureka-server](http://search.maven.org/#search|ga|1|eureka-server) ” 从mavencentral下载WAR存档

## 组态

Eureka Server有两组配置

- Eureka客户端配置如上所述。
- Eureka Server配置。

配置Eureka Server的最简单方法是使用类似于上面的Eureka Client的属性文件。首先，配置与上面指定的服务器一起运行的Eureka客户端。Eureka服务器本身启动了一个Eureka客户端，用于查找其他Eureka服务器。因此，您需要首先为Eureka Server配置Eureka客户端，就像连接到Eureka服务的任何其他客户端一样。Eureka Server将使用其Eureka Client配置来识别具有相同名称的对等eureka服务器（即）eureka.name

配置Eureka客户端后，如果您在AWS中运行，则可能需要配置Eureka Server。默认情况下，Eureka服务器在*类路径中*搜索属性文件*eureka-server.properties*。它还在特定于环境的属性文件中搜索特定于环境的覆盖。环境通常是*测试*或*产品*，由*-Deureka.environment* java命令行切换提供给*eureka*服务器（不带*.properties*后缀）。因此，服务器还搜索*eureka-server- {test，prod} .properties。*

**配置本地开发**
当运行eureka服务器进行本地开发时，通常需要等待大约3分钟才能完成启动。这是由于在找不到可用对等体时搜索对等体以进行同步和重试的默认服务器行为。通过设置属性可以减少此等待时间`eureka.numberRegistrySyncRetries=0`。

**配置AWS**
如果您在AWS上运行的解释额外的配置都需要[在这里](https://github.com/Netflix/eureka/wiki/Deploying-Eureka-Servers-in-EC2)。有关更高级的服务器配置，请参阅[此处](https://github.com/Netflix/eureka/blob/master/eureka-core/src/main/java/com/netflix/eureka/EurekaServerConfig.java)提供的选项。

如果要[构建](https://github.com/Netflix/eureka/wiki/Building-Eureka-Client-and-Server) WAR归档文件，则可以在*eureka-server / conf*下编辑文件，并且构建在创建归档文件之前将属性文件放在WEB-INF / classes下。

如果从maven 下载存档，则可以自己在WEB-INF / classes下合并已编辑的属性文件。

[运行](https://github.com/Netflix/eureka/wiki/Running-the-Demo-Application)演示应用程序可以帮助您更好地理解配置。

# 客户/服务器版本兼容性

我们对eureka使用语义版本控制，并将在次要版本升级之间保持客户端/服务器协议兼容性（即1.x版本应该在客户端和服务器部署之间具有兼容的协议）。通常，让服务器使用比客户端更新的版本总是更安全。