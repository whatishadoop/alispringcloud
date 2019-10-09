# alispringcloud

4.RestTemplate 使用介绍 ，spring boot提供，参看ShareService类，学会post,get两种类型请求 get Object/Entity区别 , getEntity能获取对应的响应码
在ContentCenterApplication 中进行RestTemplate 对象Ioc容器注入，在使用的地方注入即可

5.DTO和domain对象区别在于DOMAIN多了很多注解用于持久化或表映射，而dto只作为单纯的对象使用很少加持久化相关注释

6.版本定义
<!--语义化的版本控制-->
<!--2：主版本，第几代-->
<!--1：次版本，一些功能的增加，但是架构没有太大的变化，是兼容的-->
<!--5：增量版本，bug修复-->
<!--release：里程碑，SNAPSHOT：开发版 M：里程碑 RELEASE：正式版-->
<version>2.1.5.RELEASE</version>

GREENWICH-SR1 (SERVICE-REALESE)  表示第一个bug修复版本，建议使用SR2版本间隔时间长，大部分问题得到解决
GREENWICH RELEASE 表示第一个正式发布版本

一般不用不维护版本以及M1,M2里程碑版本

7.spring cloud以及spring alibaba 整合到spring boot中，在pom.xml中加入如下代码即可，下面是整好的pom
<dependencyManagement>
    <dependencies>
        <!--整合spring cloud-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>Greenwich.SR1</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <!--整合spring cloud alibaba-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-alibaba-dependencies</artifactId>
            <version>0.9.0.RELEASE</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

使用上面包中的工具依赖时，直接添加如下工具依赖即可，无需版本号
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>

8.参看nacos官网
https://nacos.io/zh-cn/docs/what-is-nacos.html
每个微服务实例都注册一个nacos client用于和nacos server进行通信，启动注册，停止关闭注册信息
启动nacos
Linux/Unix/Mac
启动命令(standalone代表着单机模式运行，非集群模式):
sh startup.sh -m standalone

如果您使用的是ubuntu系统，或者运行脚本报错提示[[符号找不到，可尝试如下运行：
bash startup.sh -m standalone

Windows
启动命令：
cmd startup.cmd
或者双击startup.cmd运行文件。

添加nacos依赖，启动注解不要了，自动添加
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>

添加配置nacos服务端配置,另外需要添加服务名称
cloud:
    nacos:
      discovery:
        # 指定nacos server的地址
        server-addr: localhost:8848
application:
    # 服务名称尽量用-，不要用_，不要用特殊字符
    name: user-center
    
9.spring cloud 子项目依赖命名
spring-cloud-starter-alibaba{子项目名}-sentinel{模块名}

10.TestController 测试服务发现接口DiscoveryClient, 若有个服务实例下线了，再次查询就差不到了
 @GetMapping("test2")
    public List<ServiceInstance> getInstances() {
        // 查询指定服务的所有实例的信息
        // consul/eureka/zookeeper...
        return this.discoveryClient.getInstances("user-center");
    }

11.了解stream 以及lamda表达式使用，functional函数式编程

12.nacos的领域划分， namespace > group > service > cluster > instance , namespace用于划分dev,test,prod实现隔离, group用于划分多个微服务为一组， service用于区分微服务业务领域, cluster 作为微服务集群,cluster可以建设两个集群用于容灾，比如一个南京机房，一个北京机房，服务可以指定访问哪个机房集群服务，instance为单个微服务实例

13.在微服务中配置如上领域，在配置文件中进行元数据设置，也可以在nacos页面进行元数据设置
cloud:
    nacos:
      discovery:
        # namespace: 56116141-d837-4d15-8842-94e153bb6cfb // 需要再nacos上新建再配置此处uuid
        # NJ
        # 指定集群名称
        cluster-name: BJ  // 设置该微服务属于南京机房的集群，在ribbon时设置调用
        metadata:  // 只能设置实例元数据，另外也可以通过nacos界面进行设置
          instance: c
          haha: hehe
          version: v1
          
14.客户端负载均衡 ribbon，它会自动从nacos上获取注册的负载均衡列表实例信息 ,而不是服务端nginx负载均衡,添加ribbon依赖添加
spring-cloud-starter-alibaba-nacos-discovery已经带了ribbon包

添加注释@LoadBalanced 支持客户端负载均衡ribbon功能
@LoadBalanced
public RestTemplate restTemplate() {
    RestTemplate template = new RestTemplate();
    template.setInterceptors(
        Collections.singletonList(
            new TestRestTemplateTokenRelayInterceptor()
        )
    );
    return template;
}
请求代码如下，会自动将user-center转化为对应的负载均衡分配的实例ip地址以及端口号
 ResponseEntity<String> forEntity = restTemplate.getForEntity(
            "http://user-center/users/{id}",
            String.class, 2
        );
        
15.ribbon 默认规则配置，通过接口实现自定义配置，其中负载均衡策略默认使用zoneavoidanceRule，在没有
zone时使用的是roundrobinrule规则，另外还有其他规则可以选择使用

方式一: 
// 先定义ribbon规则，另外该类必须放在主类之外，避免被主启动程序扫描到因为它也是@Component，若放在主类中被扫描到，会出现该规则将作为全局的ribbon规则使用，其它Iping等接口也可以在如下类中进行注册实现
@Configuration
public class RibbonConfiguration {
    @Bean
    public IRule ribbonRule() {
        return new RandomRule();
    }
}

// 再将该规则设置到ribbonclient上， 可以通过设置name="user-center"指定访问对应的微服务使用对应的负载均衡方式
@Configuration
@RibbonClient(name="user-center", configuration = RibbonConfiguration.class)
public class UserCenterRibbonConfiguration {
}

方式二: 配置文件方式配置负载均衡 ，使用application.yml ,属性配置方式优先级比上面代码方式高，先用属性配置，实现不了再用代码配置
user-center:  // 这里可以配置对应的微服务名使用如下负载均衡方法
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule
    
 16.ribbon的全局配置方式设置@RibbonClients 而不是上面@RibbonClient
 @Configuration
 @RibbonClients(defaultConfiguration = RibbonConfiguration.class)
 public class UserCenterRibbonConfiguration {
 }
 
17. ribbon的饥饿加载方式配置，默认为懒加载造成首次ribbon访问很慢，若开启饥饿加载在程序启动时就会加载ribbon信息，初次访问url就很快
 ribbon:
   eager-load:
     enabled: true
     clients: user-center
     
18.自定义nacos的负载均衡权重，需要再微服务实例上设置对应的实例权重值，值越大被访问的概率越高
，下面配置好后，通过修改nacos服务实例对应的权重值进行测试，若将weight设置为0，则该服务实例将
不会再访问

首先自定义NacosWeightedRule规则
最后将该规则配置到文件或者如下java代码中
@Configuration
@RibbonClients(defaultConfiguration = RibbonConfiguration.class)
public class UserCenterRibbonConfiguration {
}

19.实现同机房优先调用，content-center属于bj机房会优先访问BJ集群下的user-center实例，若bj的user-center没有可用的实例，则会访问nj机房content-center实例

首先创建一个基于相同集群优先访问的规则类
NacosSameClusterWeightedRule

最后两个微服务实例配置如下
content-center配置BJ机房集群
  cloud:
    nacos:
      discovery:
        cluster-name: BJ  // 设置cluster名称 
        
user-center 分别配置BJ,NJ机房集群，


20 ，获取元数据信息，在元数据中设置对应微服务实例只能访问对应版本的元数据实例，在规则中进行判断
 List<Instance> instances = namingService.selectInstances(name, true);
            // instances.get(0).getMetadata(); 获取元数据信息
            
21.namespace用于资源隔离，服务只能访问同一个namespace下服务，namespace配置如下uuid,先在nacos界面进行配置
cloud:
    nacos:
      discovery:
        namespace: 56116141-d837-4d15-8842-94e153bb6cfb

