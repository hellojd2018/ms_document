# Deploying Eureka Servers in EC2

## Peer 配置概述

需要配置Eureka服务器，以便每个服务器都知道所有其他对等体。有关如何为DNS方法和基于属性的方法执行此操作的示例，请参阅下文。

此外，可以对eureka服务器复制进行批处理以提高效率。要启用批量复制，请设置`eureka. shouldBatchReplication=true`。有关更多详细信息，请参见DefaultEurekaServerConfig.java。

## AWS配置概述

在AWS Cloud中，实例来去匆匆。这意味着您无法使用标准主机名或IP地址识别Eureka服务器。但是，由于Eureka服务器可以帮助您识别更改主机名的其他服务，因此您需要为Eureka服务器提供一组标准的可识别地址。

这就是AWS EC2弹性IP地址派上用场的地方。如果您之前没有听说过弹性IP地址，那么[这](http://aws.amazon.com/articles/1346)是必读的。

第一步是为每个Eureka服务器获取一个弹性IP，您将使用您的设置运行该服务器。理想情况下，您可以使用[此处](https://github.com/Netflix/eureka/wiki/Eureka-at-a-glance)指定的体系结构运行Eureka 。因此，群集中的每个服务器都需要一个弹性IP。一旦您使用弹性IP地址列表配置您的Eureka服务器，Eureka服务器就会处理找到未使用的弹性IP并在启动期间将其绑定到自身的麻烦。

您通常在AWS区域内拥有一个针对Eureka群集的[ASG](http://aws.amazon.com/autoscaling/)，并且实例启动配置为在所有区域中均匀分布。这意味着，您至少会为每个区域启动一个Eureka Server，以实现冗余和区域故障。每当一个实例被杀死时，ASG将发布一个新的Eureka服务器，最有可能在空出的区域内。新服务器将从该区域中选择一个免费的弹性IP并在那里绑定自己。对于正在访问Eureka服务器的客户端，这是透明的，并且像往常一样，eureka客户端会自动故障转移到其他服务器，然后在服务器恢复时重新连接。

您可以通过两种方式配置弹性IP，具体取决于您需要的灵活性级别。在Netflix，我们希望透明地添加新区域或新的Eureka服务器，因此我们使用DNS模型来分配新的EIP，并且客户端和服务器都可以在运行中找到它们。一个更简单的模型是在Eureka配置文件中定义它们，但缺点是，它们被融入[AMI](https://aws.amazon.com/amis/)并分发到大约1000个需要相互通信的实例，并且添加或删除区域可能非常麻烦，因为您需要将具有已更改配置的新AMI部署到所有客户端。

## 使用属性文件配置EIP

您将在Eureka Servers中配置的第一个信息是区域的可用区域。这可以在*eureka-client*属性中完成（或者如果你想要环境覆盖，可以在*eureka-client-test.properties*或*eureka-client-prod.properties中完成*）

在以下示例中，区域*us-east-1*被指定为具有3个可用区域*us-east-1c，us-east-1d，us-east-1e*。

```
eureka.us-east-1.availabilityZones=us-east-1c,us-east-1d,us-east-1e
```

接下来，配置是Eureka正在侦听请求的每个区域的服务URL。可以通过提供以逗号分隔的列表来配置区域的多个eureka服务器。

```
eureka.serviceUrl.us-east-1c=http://ec2-552-627-568-165.compute-1.amazonaws.com:7001/eureka/v2/,http://ec2-368-101-182-134.compute-1.amazonaws.com:7001/eureka/v2/
eureka.serviceUrl.us-east-1d=http://ec2-552-627-568-170.compute-1.amazonaws.com:7001/eureka/v2/
eureka.serviceUrl.us-east-1e=http://ec2-500-179-285-592.compute-1.amazonaws.com:7001/eureka/v2/
```

然后，相同的配置包括在Eureka客户注册的Eureka客户端以及想要从Eureka Server查找服务的Eureka客户端。

## 使用DNS配置EIP

如果您正在寻求灵活性，则应使用DNS配置Eureka服务URL。

首先为可用于查找可用区域列表的区域配置DNS名称。因为，使用DNS，您只能找到一个DNS名称的CNAME，我们使用TXT记录查找列表DNS名称。

例如，以下是在DNS服务器中创建的DNS TXT记录，其中列出了区域的可用DNS名称集。

```
txt.us-east-1.mydomaintest.netflix.net="us-east-1c.mydomaintest.netflix.net" 
"us-east-1d.mydomaintest.netflix.net" "us-east-1e.mydomaintest.netflix.net"
```

然后，您可以递归地为每个区域定义TXT记录，类似于以下内容（如果每个区域有多个主机名，则空格分隔）

```
txt.us-east-1c.mydomaintest.netflix.net="ec2-552-627-568-165.compute-1.amazonaws.com" 
"ec2-368-101-182-134.compute-1.amazonaws.com"
txt.us-east-1d.mydomaintest.netflix.net="ec2-552-627-568-170.compute-1.amazonaws.com"
txt.us-east-1e.mydomaintest.netflix.net="ec2-500-179-285-592.compute-1.amazonaws.com"
```

然后在Eureka Server *（eureka-client.properties*）和所有Eureka客户端中指定以下属性，以便他们能够查找DNS并查找通信所需的信息。

```
eureka.shouldUseDns=true
eureka.eurekaServer.domainName=mydomaintest.netflix.net
eureka.eurekaServer.port=7001
eureka.eurekaServer.context=eureka/v2
```

在Netflix，我们使用此模型动态添加/删除新的Eureka服务器，从而在几分钟内将信息传播给数千个客户端。

## 使用服务URL分配EIP

那么，当我们应该将EIP分配给服务器时，为什么要定义URL？任何希望彼此通信的2个实例通常使用公共主机名，以便AWS安全组遵守安全限制。Eureka服务器使用这些URL相互通信，每个URL包含一个公共主机名（*ec2-552-627-568-170.compute-1.amazonaws.com*），该主机名源自弹性IP（552.627.568.170）。

Eureka服务器根据启动的区域找到EIP。然后它尝试从该区域中找到未使用的EIP，然后在启动期间将该EIP绑定到自身。

Eureka如何找到未使用的EIP？它使用Eureka客户端查找对等实例列表，并查看它们绑定的EIPS，并选择未绑定的EIPS。它更喜欢找到分配给其区域的EIP，以便区域中所有其他实例的Eureka客户端可以与位于同一区域中的Eureka服务器通信。如果Eureka服务器无法为其区域找到任何EIPS，则会尝试在其他区域中分配的EIP。如果所有这些都被绑定，那么Eureka服务器启动并等待EIP变为空闲并且每5分钟尝试绑定EIP。

Eureka客户同样试图找到位于同一区域的Eureka服务器，如果他们找不到，他们会故障转移到其他区域的Eureka服务器。

## Eureka故障转移

当为客户端提供eureka服务器列表时，eureka客户端会自动故障转移到群集中的其他节点。让我们考虑下面的配置来理解它是如何工作的

```
eureka.us-east-1.availabilityZones=us-east-1c,us-east-1d,us-east-1e
eureka.serviceUrl.us-east-1c=http://ec2-552-627-568-165.compute-1.amazonaws.com:7001/eureka/v2/,http://ec2-368-101-182-134.compute-1.amazonaws.com:7001/eureka/v2/
eureka.serviceUrl.us-east-1d=http://ec2-552-627-568-170.compute-1.amazonaws.com:7001/eureka/v2/
eureka.serviceUrl.us-east-1e=http://ec2-500-179-285-592.compute-1.amazonaws.com:7001/eureka/v2/
```

我们已经定义了3个区域（us-east-1c，us-east-1d和us-east1e），每个区域只有一个EIP。比方说，例如，在位于[http://ec2-552-627-568-165.compute-1.amazonaws.com:7001/eureka/v2/的](http://ec2-552-627-568-165.compute-1.amazonaws.com:7001/eureka/v2/) us-east-1c运行的eureka服务器失败，所有eureka客户端自动与位于<http://ec2-552-627-568-170.compute-1.amazonaws.com:7001/eureka/v2/>的区域中的下一个服务器通信。如果失败，客户端将尝试列表中的下一个，依此类推。

当故障服务器恢复时，eureka客户端会自动重新连接回其区域中的服务器。

## Eureka AWS特定属性

在AWS云环境中，传入java命令行属性**-Deureka.datacenter = cloud，**以便Eureka客户端/服务器知道初始化特定于AWS云的信息。

Eureka服务器将要求AWS访问在EC2中运行。有两种选择。使用默认（no）配置，服务器将使用EC2实例上的InstanceProfileCredentials，或者您可以通过配置显式设置AWS accessId和secretKey：

```
   eureka.awsAccessId =
   eureka.awsSecretKey =
```

## AWS访问策略

Eureka尝试查询ASG相关信息，以便它可以确保启动的实例自动为**OUT_OF_SERVICE**或**UP，**具体取决于Autoscaling组属性中“addToLoadbalancer”标志的值。用于确定您所属的ASG的属性是通过在[配置](https://github.com/Netflix/eureka/wiki/Configuring-Eureka)Eureka客户端时指定此属性来[配置](https://github.com/Netflix/eureka/wiki/Configuring-Eureka)的。

```
eureka.asgName
```

Eureka服务器需要访问以查询ASG信息以及绑定/取消绑定云中的IP。因此，AWS策略应配置为允许上述访问。以下是具有所需访问权限的示例策略。

```
{
  "Statement": [
    {
      "Sid": "Stmt1358974336152",
      "Action": [
        "ec2:DescribeAddresses",
        "ec2:AssociateAddress",
        "ec2:DisassociateAddress"
      ],
      "Effect": "Allow",
      "Resource": "*"
    },
    {
      "Sid": "Stmt1358974395291",
      "Action": [
        "autoscaling:DescribeAutoScalingGroups"
      ],
      "Effect": "Allow",
      "Resource": "*"
    }
  ]
}
```