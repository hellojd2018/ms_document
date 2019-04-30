# Running the Demo Application

该演示应用程序包含了配置，构建和运行运行Eureka所需的所有组件的能力

- Eureka Server
- Application Service
- Application Client

有关了解配置的更多详细信息，请访问[此](https://github.com/Netflix/eureka/wiki/Configuring-Eureka)页面。

# 关于演示

该演示将帮助您设置一个在您选择的端口中监听的Eureka服务器。它还将帮助您设置将为请求提供服务的应用程序服务以及将向服务发送请求的应用程序客户端。

应用程序服务向Eureka Server注册，应用程序客户端可以查找并将请求发送到应用程序服务。交易消息后，客户端和服务器正常退出。

# 关于演示服务器的注意事项

演示服务器配置为易于设置演示用例，并且未配置为正确的生产用途。服务器配置，以了解如何正确配置eureka服务器。

# Eureka服务器配置

- 导航到eureka-server / conf /并根据需要编辑*eureka-client.properties*和*eureka-client-test.properties*。（除非您要设置高级服务器配置，否则不必编辑演示的eureka-server.properties）

- [构建](https://github.com/Netflix/eureka/wiki/Building-Eureka-Client-and-Server)应用程序。

- 上面的版本还设置了运行演示服务和演示客户端所需的所有库。

- 将WAR工件复制到_ $ TOMCAT_HOME / webapps /下的tomcat部署目录

  ```
    cp ./eureka-server/build/libs/eureka-server-XXX-SNAPSHOT.war $TOMCAT_HOME/webapps/eureka.war
    
  ```

- 启动tomcat服务器。访问***http：// localhost：8080 / eureka***以验证其中的信息。您的服务器的eureka客户端应该在30秒内注册，您应该在那里看到该信息。

# 应用程序服务的Eureka客户端配置

- 导航到eureka-examples / conf并根据需要编辑*sample-eureka-service.properties*。

# 应用程序客户端的Eureka客户端配置

- 导航到eureka-examples / conf并根据需要编辑*sample-eureka-client.properties*。

# 运行演示

请参阅示例自述文件<https://github.com/Netflix/eureka/tree/master/eureka-examples>