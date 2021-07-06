# Spring Cloud组件

- Spring Cloud Netflix：核心组件，可以对多个Netflix OSS开源套件进行整合，包括以下几个组件：
  - Eureka：服务治理组件，包含服务注册与发现
  - Hystrix：容错管理组件，实现了熔断器
  - Ribbon：客户端负载均衡的服务调用组件
  - Feign：基于Ribbon和Hystrix的声明式服务调用组件
  - Zuul：网关组件，提供智能路由、访问过滤等功能
  - Archaius：外部化配置组件
- Spring Cloud Config：配置管理工具，实现应用配置的外部化存储，支持客户端配置信息刷新、加密/解密配置内容等。
- Spring Cloud Bus：事件、消息总线，用于传播集群中的状态变化或事件，以及触发后续的处理
- Spring Cloud Security：基于spring security的安全工具包，为我们的应用程序添加安全控制
- Spring Cloud Consul : 封装了Consul操作，Consul是一个服务发现与配置工具（与Eureka作用类似），与Docker容器可以无缝集成



**注册入eureka的微服务down了后 在eureka上还是显示up**

## **openFeign**

@FeignClient(value = "cloud-provider-hystrix-payment") 添加在service上 controller调用的时候就会通过value 找到对应的微服务，并调用其中的方法。

如果一个service中需要调用两个不同的微服务的接口openFeign如何配置

**使用openFeign时服务端的HystrixProperty 不管配置多少秒 消费端调用都会超时**

解决方法：openFeign中的超时是由ribbon控制的，而ribbon的超时时间是小于Hystrix的，所有不管怎么调用Hystrix都会报超时错误，需要设置这两个的超时时间

 

```yaml
#hystrix的超时时间
hystrix:
  command:
    default:
      execution:
        timeout:
          enabled: true
        isolation:
          thread:
            #设置请求超时时间，默认1秒，超过指定的时间后，触发服务熔断
            timeoutInMilliseconds: 10000
#ribbon的超时时间
ribbon:
  ReadTimeout: 5000 #设置请求处理的超时时间
  ConnectTimeout: 5000  #设置请求链接的超时时间
```



## **Hystrix服务的降级**

**说明：**

当服务出现调用超时或者运行时错误的时候，系统不会直接报错（前端就不会出现报错页面），而是会调用设置好的fallback方法，在这个方法中处理这个错误，并返回。不会造成服务的卡顿。

**使用：**

首先在启动类上启用Hystrix 即加上 @EnableCircuitBreaker

在方法上加上注解 设置服务的降级，当调用方法时出现超时或者运行时错误的时候就会调用fallbackMethod中设置的方法。

 

```java
@HystrixCommand(fallbackMethod = "paymentTimeOutFallbackMethod",commandProperties = {
@HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "5000")
})
```



## **Hystrix服务的熔断**

**说明：**

防止服务雪崩的保护机制。当服务中某个节点出错或者超时时，会进行服务的降级（执行fallback方法），进而熔断该服务节点的调用，当检测到该节点的调用正常后会恢复此节点。

 

```java
@HystrixCommand(fallbackMethod = "paymentCircuitBreaker_fallback",commandProperties = {
        @HystrixProperty(name = "circuitBreaker.enabled",value = "true"),
        @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold",value = "10"),
        @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds",value = "10000"),
        @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage",value = "60")
})
```



**参数说明：**

| 参数                                     | 描述                                                   | 默认值 |
| ---------------------------------------- | ------------------------------------------------------ | ------ |
| circuitBreaker.enabled                   | 是否开启熔断机制                                       | true   |
| circuitBreaker.requestVolumeThreshold    | 触发熔断的请求次数                                     | 20     |
| circuitBreaker.sleepWindowInMilliseconds | 熔断后多少秒尝试请求重连                               | 50000  |
| circuitBreaker.errorThresholdPercentage  | 在触发熔断的请求次数中失败率（百分比）达到此值就会熔断 | 50     |

在10次请求中如果失败了达到60%，就会触发熔断（open），所有对该节点的请求（无论是否是正常的运行的请求）都会执行fallback方法。持续请求这个节点，当检测到请求的成功率超过60%(half open)，就会熔断就会关闭（close），正常执行的请求就不会执行fallback方法。