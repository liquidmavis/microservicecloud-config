springcloud

## 1、微服务

### 1、特点：

1、拆分成小服务

2、各自独立进程。通信两种方式：http和tcp

3、拥有自己独立的数据库



## 2. 微服务技术栈

服务开发   springboot、spring、springmvc

服务配置与管理 	Diamond

服务注册与发现  	Eureka、zookeeper

服务调用	Feign

负载均衡	Ribbon、Nginx、Feign

熔断器		Hystrix

路由			Zuul

服务监控    Zabbix

配置中心	springcloudConfig

服务部署	docker

事件总线    springcloud bus

数据流操作  springcloud stream



### 3、什么是springcloud

微服务结构解决的一站式框架



### 4、springcloud与springboot区别

springboot可以快速部署一个微服务

springcloud可以对每个微服务提供服务治理等功能



springcloud依赖于springboot

### 5、dubbo与springcloud优缺点

dubbo在通信性能上较为优势，通信采用tcp

springcloud微服务一站式解决，通信采用http



### 6、微服务案例

spring提供的RestTemplate，有多种便捷访问远程Http服务的方法

是一种简单的restful服务模板



使用

(url、requestMap、requestBean.class

rest请求地址、请求参数、http响应转换被转换成的对象类型



## 3、Springcloud各部件

### 1、Eureka服务注册与发现

#### 1、是什么

类似于dubbo注册中心，zookeeper

遵循AP（可用性、分区容错性）

#### 2、原理

采用了C-S架构

![1566880575694](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\1566880575694.png)



三大角色

- Eureka Server 提供服务注册与发现
- service provider 服务提供方
- service consumer 服务消费方

#### 3、自我保护

在自我保护模式下，Eureka Server会保护服务注册表中的信息，不再注销任务服务实例。当收到的心跳数重新恢复到阀值以上，就退出自我保护。它的涉及哲学是宁可保留错误的注册信息，也不盲目注销任何可能健康的服务实例。



#### 4、eureka和zookeeper

zookeeper遵守CP，eureka遵守AP

zookeeper保证强制性，但如果节点leader节点死亡，那么就要选举，短时间内瘫痪

eureka保证高可用，集群状态下，一个或多个节点死亡，不会影响其他节点工作，但不能保证节点注册信息的实时性



### 2、Ribbo负载均衡

#### 1、是什么

客户端的负载均衡

客户端自己根据负载均衡算法来选择服务器

#### 

ribbon和eureka整合后，consumer直接调用服务不用在关心地址和端口号

#### 2、配置

配置集群url

```yml
eureka:
  client:
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka/,http://eureka7003.com:7003/eureka/,http://eureka7002.com:7002/eureka/
    register-with-eureka: false
```

给restTemplate配置负载均衡

```java
 @LoadBalanced  //负载均衡
 @Bean
    public RestTemplate restTemplate(){
        return new RestTemplate();
    }
```

在消费者主类上添加@EnableEurekaClient



#### 3、负载均衡算法

 默认使用轮询算法

选择哪个负载均衡通过Rule接口实现的rule类

注册到容器，可以自己实现，也可以用ribbo提供的 

```java
@Bean
    public IRule myRule(){
        return new RandomRule();
    }
```

##### 自定义IRule

注意在自定义的时候，自己的写的Rule不要放在有@componetScan扫描的包下，也就是说不要放在主启动类的包下，需要重新建一个包放这个类

```java
@Configuration
public class MyRule implements IRule {
    @Bean
    public IRule myRule(){
        return new RandomRule();
    }
```

并在启动类配置如下注解，第一个参数是服务名，第二是配置文件类名

```java
@RibbonClient(name = "MICROSERVICECLOUD-DEPT",configuration = MyRule.class)
```



### 3、feign负载均衡

#### 1、是什么？

声明式web服务客户端，只需要创建一个接口，使用注解即可。

默认也是使用轮询

#### 2、步骤

在公共api层创建接口

```java

@FeignClient("MICROSERVICECLOUD-DEPT")
public interface DetpClientService {
    @GetMapping(value = "dept/get/{id}")
    public Dept get(@PathVariable long id);

    @GetMapping("dept/list")
    public List<Dept> list();

    @PostMapping("dept/add")
    public boolean add(Dept dept);
}
```

在消费者那编写controller

```java
@RestController
public class DeptController_Consumer {
    //feign定义的接口
    @Autowired
    private DetpClientService detpClientService;


    @RequestMapping("consumer/dept/add")
    public boolean add(Dept dept){
        return detpClientService.add(dept);
    }
```

开启feign接口使用

```java
@EnableFeignClients(basePackages = {"com.liquid"})
@ComponentScan("com.liquid")
```

#### 3、与ribbon区别

ribbon可以自定义rule算法，并且面向微服务编写

feign默认轮询，面向接口编写

### 4、Hystrix断路器

#### 1、是什么

类似于断路器，保护系统不会因为单个服务调用不同，长时间占用资源而导致系统雪崩。如果出现上面的情形，就会直接回调给调用方一个预期中结果。

#### 2、能干什么

##### 1、服务熔断

快速返回错误信息给调用方，**服务器**层面产生的



在服务提供方启动类开启hystrix

```
@EnableHystrix
```

在服务类的controller上编写如下

```java
@RestController
public class DeptController {
    @Autowired
    private DeptService deptService;

    @GetMapping("dept/get/{id}")
    @HystrixCommand(fallbackMethod = "fallHystrixGetMethod")
    public Dept get(@PathVariable Long id){
        Dept dept = deptService.get(id);
        if(dept == null){
            throw new RuntimeException("id="+id+"没有对应信息");
        }
        return dept;
    }

    public Dept fallHystrixGetMethod(@PathVariable Long id){
        return new Dept().setDeptno(id).setDname("该id"+id+"不存在").setDb_source("no this database in mysql");
    }
```



##### 2、服务降级

整体资源不够用，某个服务暂时停止，**客户端**自己准备本地fallback信息来回调，保证了一定的可用性



使用：

在公共api中编写一个类实现FallFactory，泛型为消费者使用的DetpClientService

```java
@Component
public class FallBackDeptClientFactory implements FallbackFactory<DetpClientService> {
    @Override
    public DetpClientService create(Throwable throwable) {
        return new DetpClientService() {
            @Override
            public Dept get(long id) {
                return new Dept().setDname("id:"+id+"当前服务提供已经down掉，请稍后尝试").setDb_source("没有该数据库");
            }

```

并且在DetpClientService添加如下，指定降级fallbackFactory

```
@FeignClient(value = "microservicecloud-dept",fallbackFactory = FallBackDeptClientFactory.class)
```

并且在消费者yml文件中配置

```yml
feign:
  hystrix:
    enabled: true
```



##### 3、服务监控

hystrixDashBoard 提供了实时的调用监控，可视化界面

### 5、zuul路由网关 

#### 1、是什么

对请求的路由和过滤

路由：将外部请求转发到具体的微服务实例，实现外部访问统一入口

过滤：负责对请求的处理过程进行干预相当于过滤器

Zuul与eureka整合，将Zuul注册到eureka下，同时从eureka中获得其他微服务信息，以后访问都是从Zuul然后到eureka最后到微服务实例



### 6、config分布式配置中心

#### 1、是什么

解决分布式中出现的大量配置问题

是一种集中式、动态配置管理





