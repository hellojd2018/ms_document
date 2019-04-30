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

  ```sh
  cd eureka
  ./gradlew clean build
  ```

- 您可以找到以下工件

  - Eureka Server WAR archive (./eureka-server/build/libs/eureka-server-XXX.war )
  - Eureka Client (./eureka-client/build/libs/eureka-client-XXX.jar )
  - Dependencies (./eureka-server/testlibs/) (If you do not want to use maven to download dependencies you can use these archives)