# Understanding Eureka Peer to Peer Communication

确保您已访问[此](https://github.com/Netflix/eureka/wiki/Configuring-Eureka-in-AWS-Cloud)页面以了解Eureka群集配置。

Eureka客户试图在同一区域与Eureka Server交谈。如果与服务器通信时出现问题或者同一区域中不存在服务器，则客户端将故障转移到其他区域中的服务器。

一旦服务器开始接收流量，服务器上执行的所有[操作](https://github.com/Netflix/eureka/wiki/Understanding-eureka-client-server-communication)都将复制到服务器知道的所有对等节点。如果某个操作由于某种原因而失败，则会在下一个也会在服务器之间复制的心跳上协调该信息。

当Eureka服务器启动时，它会尝试从相邻节点获取所有实例注册表信息。如果从节点获取信息时出现问题，服务器会在放弃之前尝试所有对等体。如果服务器能够成功获取所有实例，则会根据该信息设置应接收的续订阈值。如果有任何时间，续订低于为该值配置的百分比（15分钟内低于85％），服务器将停止使实例过期以保护当前实例注册表信息。

在Netflix中，上述安全措施称为**自我保护**模式，主要用作在一组客户端和Eureka服务器之间存在网络分区的情况下的保护。在这些情况下，服务器会尝试保护已有的信息。在大规模中断的情况下可能存在这样的情况，这可能导致客户端获得不再存在的实例。客户端必须确保它们对eureka服务器具有弹性，可以返回不存在或无响应的实例。这些方案中的最佳保护是快速超时并尝试其他服务器。

在服务器无法从相邻节点获取注册表信息的情况下，它等待几分钟（5分钟），以便客户端可以注册其信息。服务器努力不通过仅将流量偏移到一组实例并导致容量问题来向那里的客户端提供部分信息。

Eureka服务器彼此使用尤里卡客户端和服务器所描述之间使用相同的机制进行通信[此处](https://github.com/Netflix/eureka/wiki/Understanding-eureka-client-server-communication)。

另外值得注意的是，有几种[配置](https://github.com/Netflix/eureka/blob/master/eureka-core/src/main/java/com/netflix/eureka/EurekaServerConfig.java)可以在服务器上进行调整，包括服务器之间的通信（如果需要）。

## 在Peers之间的网络中断期间会发生什么？

在Peer之间的网络中断的情况下，可能发生以下事情

- 对等体之间的心跳复制可能会失败，服务器会检测到这种情况并进入保护当前状态的自我保护模式。
- 注册可能发生在孤立的服务器中，一些客户可能会反映新的注册，而其他客户可能不会。
- 在网络连接恢复到稳定状态后，情况会自动更新。当对等方能够正常通信时，注册信息将自动传输到没有它们的服务器。

最重要的是，在网络中断期间，服务器尝试尽可能具有弹性，但是在此期间客户端可能具有不同的服务器视图。