# 1.什么是ark-jetcache？
&emsp;&emsp;ark-jetcache是ark系列框架中的缓存组件，基于阿里jetcache组件开发，jetcache的基础上做了封装和功能的增强。
# 2.ark-jetcache解决了什么问题？
&emsp;&emsp; 在并发高的服务中，需要做两级缓存，而公版的jetcache有一些限制，不能满足我们自己的需求。基于公司自己业务的需求及内部一些中间件串联的需要，因此我们基于公版做了一些定制化改造，如trace-id支持，公共日志打印等。
# 3.功能列表
- 开启fastjson2序列化方式
- 加入redisTemplate自动装配
- 适配springdata-redis配置
- 分布式锁支持
- trace-id,压测标等flag传递

# 4.接入ark-jetcache
```java
1.POM文件引入：
<!-- 版本属性 -->
<properties>
  <jetcache.version>2.7.0</jetcache.version>
</properties>

<!-- jetcache starter-->
<dependency>
    <groupId>com.ark.cache</groupId>
    <artifactId>ark-jetcache</artifactId>
    <version>${jetcache.version}</version>
</dependency>
<!-- #注解方式#id占位支持： -->
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <configuration>
                <source>8</source>
                <target>8</target>
                <compilerArgument>-parameters</compilerArgument>
            </configuration>
        </plugin>
    </plugins>
</build>


2.Spring Boot 项目开启jetcache
    Application类加入注解如下：
    @SpringBootApplication
    @Configuration
    @EnableAutoConfiguration
    @ComponentScan({"com.alicp.jetcache.anno"})
    @EnableMethodCache(basePackages = {"com.alicp.jetcache.anno"})
    public class RedisTestApplication {
    }


3.在spring boot环境下使用springdata-redis
    application.yml文件如下（这里省去了local相关的配置）：

jetcache:
    statIntervalMinutes: 15
    areaInCacheName: false
    penetrationProtect: false
    #本地缓存配置
    local:
        default:
            type: caffeine
            keyConvertor: fastjson2
            limit: 200
            defaultExpireInMillis: 100000
    #分布式缓存配置
    remote:
        default:
            type: redis.springdata
            keyConvertor: fastjson2
            valueEncoder: fastjson2?useIdentityNumber=false
            valueDecoder: fastjson2?useIdentityNumber=false
            broadcastChannel: projectA
            defaultExpireInMillis: 100000
            keyPrefix: spring-data-redis

    #redis单机模式
spring:
    redis:
        port: 31186
        timeout: 10000
        host: 127.0.0.1
        port: 7001
        password:
        timeout: 3000ms
        database: 0

    #springdata-redis lettuce配置
        lettuce:
            pool:
            max-idle: 8
            max-active: 15
            max-wait: 5s
            min-idle: 5
    
    如果需要直接使用springdata-redis的RedisTemplate,已有默认配置(RedisAutoConfig类)。
    代码中注入redisTemplate：
  
    @Autowired
  private RedisTemplate redisTemplate;


4.jetcache API使用
    如果不通过@CreateCache和@Cached注解，可以通过下面的方式创建Cache。通过注解创建的缓存会自动设置keyPrefix，这里是手工创建缓存，对于远程缓存需要设置keyPrefix属性，以免不同Cache实例的key发生冲突。

    LettuceConnectionFactory connectionFactory =  new LettuceConnectionFactory(
    new RedisStandaloneConfiguration("127.0.0.1", 6379));
    connectionFactory.afterPropertiesSet();

    Cache<Long,OrderDO> cache = RedisSpringDataCacheBuilder.createBuilder()
    .keyConvertor(Fastjson2KeyConvertor.INSTANCE)
    .valueEncoder(Fastjson2ValueEncoder.INSTANCE)
    .valueDecoder(Fastjson2ValueDecoder.INSTANCE)
    .connectionFactory(connectionFactory)
    .keyPrefix("orderCache")
    .expireAfterWrite(500, TimeUnit.MILLISECONDS)
    .buildCache();


5.jetcache注解方式实现缓存
JetCache方法缓存和SpringCache比较类似，它原生提供了TTL支持，以保证最终一致，并且支持二级缓存。JetCache2.4以后支持基于注解的缓存更新和删除。
在spring环境下，使用@Cached注解可以为一个方法添加缓存，@CacheUpdate用于更新缓存，@CacheInvalidate用于移除缓存元素。注解可以加在接口上也可以加在类上，加注解的类必须是一个spring bean，例如：

public interface UserService {
  @Cached(name="userCache.", key="#userId", expire = 3600)
  User getUserById(long userId);

  @CacheUpdate(name="userCache.", key="#user.userId", value="#user")
  void updateUser(User user);

  @CacheInvalidate(name="userCache.", key="#userId")
  void deleteUser(long userId);
}
key使用Spring的SpEL脚本来指定。如果要使用参数名（比如这里的key="#userId"），项目编译设置target必须为1.8格式，并且指定javac的-parameters参数，否则就要使用key="args[0]"这样按下标访问的形式。


6.分布式锁使用
    1.启用自动装配
        @EnableAutoConfiguration
    2.对象注入
        @Resource
        private SimpleRedisLock simpleRedisLock;
    3.使用代码

        方式1
        try (AutoReleaseLock autoReleaseLock = lock.tryLock("MyKey", 10, TimeUnit.SECONDS)) {
            //加锁成功，执行业务逻辑
            if (autoReleaseLock != null) {
                System.out.println("success lock! Does anything.");
            }
        }

    方式2
        lock.tryLockAndRun("MyKey", 10, TimeUnit.SECONDS,() -> {
          //加锁成功，执行业务逻辑
          System.out.println("success lock! Does anything.");
        });

```

