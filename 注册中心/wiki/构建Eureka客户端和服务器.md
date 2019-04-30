# 构建Eureka客户端和服务器

## 先决条件

- Git版本1.7.11.3或更高版本

## 构建步骤

- 安装最新的[git](http://git-scm.com/book/en/Getting-Started-Installing-Git)包。

- 从github获取Eureka源代码

  ```
  git clone https://github.com/Netflix/eureka.git
  ```

- 现在，通过在您提取源的目录中执行以下命令来构建Eureka Server。

  ```
  cd eureka
  ./gradlew清洁构建
  ```

- 您可以找到以下工件

  - Eureka Server WAR档案（./eureka-server/build/libs/eureka-server-XXX.war）
  - Eureka客户端（./eureka-client/build/libs/eureka-client-XXX.jar）
  - 依赖关系（./eureka-server/testlibs/）（如果您不想使用maven下载依赖关系，可以使用这些存档）