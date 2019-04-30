# Overriding Default Configurations

Eureka带有内置默认设置，适用于大多数场景。如果您要覆盖默认配置，则可能需要查看3种类型的配置。

您可以使用您想要的任何机制扩展以下默认配置类以提供您自己的配置值。

**重要提示：请不要使用这些接口，它们仅用于文档目的，将来可能会更改。**

- [云](https://github.com/Netflix/eureka/blob/master/eureka-client/src/main/java/com/netflix/appinfo/CloudInstanceConfig.java)或[数据中心](https://github.com/Netflix/eureka/blob/master/eureka-client/src/main/java/com/netflix/appinfo/MyDataCenterInstanceConfig.java)实例配置都扩展了[PropertiesInstanceConfig](https://github.com/Netflix/eureka/blob/master/eureka-client/src/main/java/com/netflix/appinfo/PropertiesInstanceConfig.java)
- [Eureka客户端](https://github.com/Netflix/eureka/blob/master/eureka-client/src/main/java/com/netflix/discovery/DefaultEurekaClientConfig.java)配置
- [Eureka服务器](https://github.com/Netflix/eureka/blob/master/eureka-core/src/main/java/com/netflix/eureka/DefaultEurekaServerConfig.java)配置

请注意，上述内容还可作为Eureka提供的所有上述配置的默认配置值的文档。

## 向注册信息添加自定义元数据

有时，您可能希望添加特定于部署环境的自定义元数据以进行注册和查询。例如，您可能希望传播应该可用于特定部署环境的自定义环境标识。Eureka提供了将自定义元数据添加为标准注册数据之外的关键：值对的功能。您可以通过两种方式执行此操作：

### 访问元数据

```
String myValue = instanceInfo 。getMetadata（）。得到（“ myKey ”）;
```

### 通过配置静态设置：

表单的任何配置 `eureka.metadata.mykey=myvalue`都会将k：v对`mykey:myvalue`添加到eureka的元数据映射中。

### 通过代码动态设置：

要动态执行此操作，您需要首先提供自己的[EurekaInstanceConfig](https://github.com/Netflix/eureka/blob/master/eureka-client/src/main/java/com/netflix/appinfo/EurekaInstanceConfig.java)接口的自定义实现。然后，您可以重载该`public Map<String, String> getMetadataMap()`方法以返回包含所需元数据值的元数据映射。有关提供上述基于配置的系统的示例实现，请参见[PropertiesInstanceConfig](https://github.com/Netflix/eureka/blob/master/eureka-client/src/main/java/com/netflix/appinfo/PropertiesInstanceConfig.java)。