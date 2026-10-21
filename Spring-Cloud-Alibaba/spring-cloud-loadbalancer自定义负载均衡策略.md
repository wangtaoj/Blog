### 背景

本地开发时调用远程服务想要稳定调用到本机的服务进行调试，便可以通过自定义一个负载均衡策略来达到这个目的，spring cloud loadbalancer默认注入的是`RoundRobinLoadBalancer`(轮训策略)

### 实现

NodeIdentity.java

```java
import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.client.HostInfoEnvironmentPostProcessor;
import org.springframework.stereotype.Component;

/**
 * 节点标识
 * @see HostInfoEnvironmentPostProcessor
 */
@Component
public class NodeIdentity {

    @Value("${spring.cloud.client.hostname:}")
    private String hostname;

    @Value("${spring.cloud.client.ip-address:}")
    private String ipAddress;

    public String getHostname() {
        return hostname;
    }

    public String getIpAddress() {
        return ipAddress;
    }
}
```

LocalFirstLoadBalancer.java

```java
/**
 * 本机服务优先负载均衡算法, 若本机没有该服务则降级为轮训策略
 */
@Slf4j
public class LocalFirstLoadBalancer implements ReactorServiceInstanceLoadBalancer {

    private final ObjectProvider<ServiceInstanceListSupplier> supplierProvider;

    /**
     * 服务id
     */
    private final String serviceId;

    /**
     * 本机ip
     */
    private final String localIp;

    private final AtomicInteger position;

    public LocalFirstLoadBalancer(ObjectProvider<ServiceInstanceListSupplier> supplierProvider,
                                  String serviceId,
                                  NodeIdentity nodeIdentity) {
        this.serviceId = serviceId;
        this.supplierProvider = supplierProvider;
        this.localIp = nodeIdentity.getIpAddress();
        this.position = new AtomicInteger(new Random().nextInt(1000));
    }

    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        ServiceInstanceListSupplier supplier = supplierProvider.getIfAvailable(NoopServiceInstanceListSupplier::new);
        return supplier.get(request).next().map(this::getInstanceResponse);
    }

    private Response<ServiceInstance> getInstanceResponse(List<ServiceInstance> instances) {
        ServiceInstance instance = chooseInstance(instances);
        if (Objects.isNull(instance)) {
            if (log.isWarnEnabled()) {
                log.warn("No servers available for service: {}", serviceId);
            }
            return new EmptyResponse();
        }
        return new DefaultResponse(instance);
    }

    private ServiceInstance chooseInstance(List<ServiceInstance> instances) {
        if (instances.isEmpty()) {
            return null;
        }
        for (ServiceInstance instance : instances) {
            if (Objects.equals(localIp, instance.getHost())) {
                return instance;
            }
        }
        // 降级为轮训策略
        if (instances.size() == 1) {
            return instances.get(0);
        }
        int pos = this.position.incrementAndGet() & Integer.MAX_VALUE;
        return instances.get(pos % instances.size());
    }
}
```

LocalFirstLoadBalancerConfiguration.java

```java
public class LocalFirstLoadBalancerConfiguration {

    /**
     * @see LoadBalancerClientConfiguration#reactorServiceInstanceLoadBalancer
     */
    @ConditionalOnMissingBean
    @Bean
    public ReactorServiceInstanceLoadBalancer localFirstLoadBalancer(Environment environment,
                                                                     LoadBalancerClientFactory loadBalancerClientFactory,
                                                                     NodeIdentity nodeIdentity) {
        // 服务名, NamedContextFactory.createContext方法创建子容器时会把服务名放到environment中
        String serviceId = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        ObjectProvider<ServiceInstanceListSupplier> lazyObjectProvider = loadBalancerClientFactory.getLazyProvider(serviceId, ServiceInstanceListSupplier.class);
        return new LocalFirstLoadBalancer(lazyObjectProvider, serviceId, nodeIdentity);
    }
}

```

LocalFirstLoadBalancerAutoConfiguration.java

```java
/**
 * 本地环境生效
 * {@code @LoadBalancerClients}注解中defaultConfiguration指定的配置类会被注册到每一个子容器中
 */
@Profile("local")
@LoadBalancerClients(defaultConfiguration = LocalFirstLoadBalancerConfiguration.class)
@Configuration(proxyBeanMethods = false)
public class LocalFirstLoadBalancerAutoConfiguration {
}
```

