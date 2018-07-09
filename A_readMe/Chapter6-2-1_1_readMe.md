# 在传统Spring应用中使用spring-boot-actuator模块提供监控端点

在之前发布的《Spring Boot Actuator监控端点小结》一文中，我们介绍了Spring Boot Actuator模块为应用提供的强大监控能力。
在Spring Boot应用中，我们只需要简单的引入spring-boot-starter-actuator依赖就能为应用添加各种有用的监控端点。
其中，/health端点能够全面检查应用的健康状态，该端点也被Spring Cloud中的服务治理（Eureka、Consul）用来检查应用的健康状态。
所以，在使用Spring Cloud构建微服务架构的时候，如果还存在一些遗留的传统Spring应用时，我们就需要为这些应用也加入/health端点。
那么在传统的Spring应用中我们是否也能引入该模块来提供这些有用的监控端点呢？下面我们就来介绍整合的详细步骤：

第一步：引入相关依赖

由于在传统Spring应用中，我们不能直接使用Starter POMs。所以，我们需要拆解了来引入到传统Spring应用的pom.xml中，主要有如下两个依赖：
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-actuator</artifactId>
    <version>1.4.3.RELEASE</version>
    <type>jar</type>
</dependency>
<dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-validator</artifactId>
    <version>4.3.2.Final</version>
</dependency>
```

第二部：手工引入配置

由于在传统Spring应用中没有自动化配置功能，所以我们需要手工的来创建配置并启用Spring Boot Actuator的监控端点。
比如，我们先来创建一个实现/health端点的配置，具体如下：

```
@Configuration
@Import({ EndpointAutoConfiguration.class,
        HealthIndicatorAutoConfiguration.class})
public class MyAppSpringConfig {

    @Bean
    public EndpointHandlerMapping endpointHandlerMapping(
            Collection<? extends MvcEndpoint> endpoints) {
        return new EndpointHandlerMapping(endpoints);
    }

    @Bean
    public HealthMvcEndpoint healthMvcEndpoint(HealthEndpoint delegate) {
        return new HealthMvcEndpoint(delegate, false);
    }

}
```

其中，@Import中引入的org.springframework.boot.actuate.autoconfigure.EndpointAutoConfiguration类是Spring Boot Actuator的基础配置类。
org.springframework.boot.actuate.autoconfigure.HealthIndicatorAutoConfiguration类是/health端点的基础配置，具体内容本文不做详细展开，
读者可自行查看。而在该配置类中，还创建了两个Bean，其中EndpointHandlerMapping是org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping的子类，
它用来加载所有的监控端点；而HealthMvcEndpoint是具体的/health端点实现。

在完成上面配置之后，我们就可以启动Spring应用，此时就可以看控制台中看到打印出了/health端点，我们可以尝试访问该端点来获取当前实例的健康状况。

除了在传统应用中可以加载/health端点之外，我们也可以如法炮制地创建其他端点，比如：获取各个度量指标的/metrics端点，可以通过如下配置实现：

```
@Configuration
@Import({ EndpointAutoConfiguration.class,
        PublicMetricsAutoConfiguration.class,
        HealthIndicatorAutoConfiguration.class})
public class MyAppSpringConfig {

    @Bean
    public EndpointHandlerMapping endpointHandlerMapping(
            Collection<? extends MvcEndpoint> endpoints) {
        return new EndpointHandlerMapping(endpoints);
    }

    @Bean
    public HealthMvcEndpoint healthMvcEndpoint(HealthEndpoint delegate) {
        return new HealthMvcEndpoint(delegate, false);
    }

    @Bean
    public EndpointMvcAdapter metricsEndPoint(MetricsEndpoint delegate) {
        return new EndpointMvcAdapter(delegate);
    }
  
}
```
这里，我们主要增加了两个内容：

@Import中增加引入PublicMetricsAutoConfiguration配置类
创建/metrics端点的实现Bean

到这里，本文的内容就介绍完了，更多关于传统Spring应用与Spring Boot/Cloud的配合使用。敬请关注我的博客和公众号，获取持续分享。

