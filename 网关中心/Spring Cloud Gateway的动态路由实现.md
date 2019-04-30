# Spring Cloud Gateway的动态路由实现

## 1.前言

网关中有两个重要的概念，那就是路由配置和路由规则，路由配置是指配置某请求路径路由到指定的目的地址。而路由规则是指匹配到路由配置之后，再根据路由规则进行转发处理。
Spring Cloud Gateway作为所有请求流量的入口，在实际生产环境中为了保证高可靠和高可用，尽量避免重启,需要实现Spring Cloud Gateway动态路由配置。前面章节介绍了Spring Cloud Gateway提供的两种方法去配置路由规则，但都是在Spring Cloud Gateway启动时候，就将路由配置和规则加载到内存里，无法做到不重启网关就可以动态的对应路由的配置和规则进行增加，修改和删除。`本篇文章简单介绍如何实现Spring Cloud Gateway的动态路由。`

## 2. Spring Cloud Gateway简单的动态路由实现

Spring Cloud Gateway的官方文档并没有讲如何动态配置，查看 Spring Cloud Gateway的源码，发现`在org.springframework.cloud.gateway.actuate.GatewayControllerEndpoint`类中提供了动态配置的Rest接口，但是`需要开启Gateway的端点`，而且提供的功能不是很强大。通过参考和GatewayControllerEndpoint相关的代码，可以自己编码实际动态路由配置。
下面通过案例的方式去讲解怎么实现Gateway的动态路由配置。案例工程如ch18-7-gateway所示。

> 代码地址:<https://github.com/SpringCloud/spring-cloud-code/blob/master/ch18-7/ch18-7-gateway>

## 3. 简单动态路由的实现

### 3.1 新建Maven工程ch18-7-gateway

配置主要的核心依赖如代码清单18-33所示：
代码清单: ch18-7/ch18-7-gateway/pom.xml

```
<dependencies>
	<dependency>
		<groupId>org.springframework.cloud</groupId>
		<artifactId>spring-cloud-starter-gateway</artifactId>
	</dependency>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-webflux</artifactId>
	</dependency>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-actuator</artifactId>
	</dependency>
</dependencies>
```



### 3.2 根据Spring Cloud Gateway的路由模型定义数据传输模型

分别创建GatewayRouteDefinition.java, GatewayPredicateDefinition.java, GatewayFilterDefinition.java这三个类。
(1) `创建路由定义模型`如下代码清单18-34所示：
代码清单 18-34: ch18-7/ch18-7-gateway/src/main/java/cn/springcloud/book/gateway/model/GatewayRouteDefinition.java

```
public class GatewayRouteDefinition {
    //路由的Id
    private String id;
    //路由断言集合配置
    private List<GatewayPredicateDefinition> predicates = new ArrayList<>();
    //路由过滤器集合配置
    private List<GatewayFilterDefinition> filters = new ArrayList<>();
    //路由规则转发的目标uri
    private String uri;
    //路由执行的顺序
    private int order = 0;
    //此处省略get和set方法
}
```

(2)`创建过滤器定义模型`,代码如代码清单18-35所示：
代码清单18-35: ch18-7/ch18-7-gateway/src/main/java/cn/springcloud/book/gateway/model/GatewayFilterDefinition.java

```
public class GatewayFilterDefinition {
    //Filter Name
    private String name;
    //对应的路由规则
    private Map<String, String> args = new LinkedHashMap<>();
    //此处省略Get和Set方法
}
```

(3)`路由断言定义模型`，代码如代码清单18-36所示:
代码清单18-36: ch18-7/ch18-7-gateway/src/main/java/cn/springcloud/book/gateway/model/GatewayPredicateDefinition.java

```
public class GatewayPredicateDefinition {
    //断言对应的Name
    private String name;
    //配置的断言规则
    private Map<String, String> args = new LinkedHashMap<>();
    //此处省略Get和Set方法
}
```

### 3.3 编写动态路由实现类

编写DynamicRouteServiceImpl并实现ApplicationEventPublisherAware接口，代码如代码清单18-37所示: ch18-37/ch18-7-gateway/src/main/java/cn/springcloud/book/gateway/route/DynamicRouteServiceImpl.java

```
@Service
public class DynamicRouteServiceImpl implements ApplicationEventPublisherAware {
    @Autowired
    private RouteDefinitionWriter routeDefinitionWriter;
    private ApplicationEventPublisher publisher;
    //增加路由
    public String add(RouteDefinition definition) {
        routeDefinitionWriter.save(Mono.just(definition)).subscribe();
        this.publisher.publishEvent(new RefreshRoutesEvent(this));
        return "success";
    }
    //更新路由
    public String update(RouteDefinition definition) {
        try {
            this.routeDefinitionWriter.delete(Mono.just(definition.getId()));
        } catch (Exception e) {
            return "update fail,not find route  routeId: "+definition.getId();
        }
        try {
            routeDefinitionWriter.save(Mono.just(definition)).subscribe();
            this.publisher.publishEvent(new RefreshRoutesEvent(this));
            return "success";
        } catch (Exception e) {
            return "update route  fail";
        }
    }
    //删除路由
    public Mono<ResponseEntity<Object>> delete(String id) {
        return this.routeDefinitionWriter.delete(Mono.just(id))
                .then(Mono.defer(() -> Mono.just(ResponseEntity.ok().build())))
                .onErrorResume(t -> t instanceof NotFoundException, t -> Mono.just(ResponseEntity.notFound().build()));
    }
    @Override
    public void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher) {
        this.publisher = applicationEventPublisher;
    }
}
```

### 3.4 编写Rest接口

编写RouteController类的提供Rest接口，用于动态路由配置。代码如代码清单18-38所示:
代码清单 18-38: ch18-7/ch18-7-gateway/src/main/java/cn/springcloud/book/gateway/controller/RouteController.java

```
@RestController
@RequestMapping("/route")
public class RouteController {
    @Autowired
    private DynamicRouteServiceImpl dynamicRouteService;
    //增加路由
    @PostMapping("/add")
    public String add(@RequestBody GatewayRouteDefinition gwdefinition) {
        try {
            RouteDefinition definition = assembleRouteDefinition(gwdefinition);
            return this.dynamicRouteService.add(definition);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return "succss";
    }
    
    //删除路由
    @DeleteMapping("/routes/{id}")
    public Mono<ResponseEntity<Object>> delete(@PathVariable String id) {
        return this.dynamicRouteService.delete(id);
    }
    //更新路由
    @PostMapping("/update")
    public String update(@RequestBody GatewayRouteDefinition gwdefinition) {
        RouteDefinition definition = assembleRouteDefinition(gwdefinition);
        return this.dynamicRouteService.update(definition);
    }
}
```

### 3.5 配置application.yml文件

在application.yml文件配置应用的配置信息，并开启Spring Cloud Gateway对外提供的端点Rest接口。代码如代码清单18-39所示:
代码清单 18-39: ch18-7/ch18-7-gateway/src/main/resources/application.yml
配置输出日志如下所示:

```
# 配置输出日志
logging:
  level:
    org.springframework.cloud.gateway: TRACE
    org.springframework.http.server.reactive: DEBUG
    org.springframework.web.reactive: DEBUG
    reactor.ipc.netty: DEBUG
#开启端点
management:
  endpoints:
    web:
      exposure:
        include: '*'
  security:
    enabled: false
```



### 3.6 启动ch18-7-gateway应用测试

(1) 启动ch18-7-gateway应用之后，由于开启了端点，首先打开浏览器访问端点URL:
<http://localhost:8080/actuator/gateway/routes> ,查看路由信息返回为空，如下图所示:

[![空的路由信息](http://xujin.org/images/sc-g-route/1.png)](http://xujin.org/images/sc-g-route/1.png)空的路由信息

(2)打开PostMan，访问<http://localhost:8080/route/add>, 发起Post请求，如下图所示,返回success说明向Gateway增加路由配置成功。
[![动态添加路由成功](http://xujin.org/images/sc-g-route/2.png)](http://xujin.org/images/sc-g-route/2.png)动态添加路由成功

然后再打开PostMan访问端点URL:<http://localhost:8080/actuator/gateway/routes> ,
查看路由信息返回如下图所示，可以看到已经添加的路由配置。
[![路由端点返回结果](http://xujin.org/images/sc-g-route/3.png)](http://xujin.org/images/sc-g-route/3.png)路由端点返回结果

(3) 打开浏览器访问<http://localhost:8080/jd>, 可以正常转发<https://www.jd.com/对应的京东商城首页。>
(4) 通过访问<http://localhost:8080/route/update>, 对id为jd_route的路由更新配置，如下图所示：
[![更新路由配置](http://xujin.org/images/sc-g-route/4.png)](http://xujin.org/images/sc-g-route/4.png)更新路由配置

然后再访问路由端点URL,发现路由配置已经被更新，如下图所示:
[![查看路由端点](http://xujin.org/images/sc-g-route/5.png)](http://xujin.org/images/sc-g-route/5.png)查看路由端点

然后通过浏览器访问<http://localhost:8080/taobao> ,可以成功转发到淘宝网。
(5) 通过访问http: //localhost:8080/route/delete/jd_route,其中的id为路由对应的id，删除路由结果如下图所示:
[![删除路由成功](http://xujin.org/images/sc-g-route/6.png)](http://xujin.org/images/sc-g-route/6.png)删除路由成功