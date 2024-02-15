## Spring Data Redis 的使用

### Spring Cache简介

当SpringBoot结合Redis来作为缓存使用时，最简单的方式就是使用Spring Cache了，使用它我们无需知道Spring中对Redis的各种操作，仅仅通过它提供的[@Cacheable](/Cacheable) 、[@CachePut](/CachePut) 、[@CacheEvict](/CacheEvict) 、@EnableCaching等注解就可以实现缓存功能。
### Spring Cache常用注解

#### [@EnableCaching](/EnableCaching)
开启缓存功能，一般放在启动类上。

#### [@Cacheable](/Cacheable)
使用该注解的方法当缓存存在时，会从缓存中获取数据而不执行方法，当缓存不存在时，会执行方法并把返回结果存入缓存中。`一般使用在查询方法上`，可以设置如下属性：
- value：缓存名称（必填），指定缓存的命名空间；
- key：用于设置在命名空间中的缓存key值，可以使用SpEL表达式定义；
- unless：条件符合则不缓存；
- condition：条件符合则缓存。
#### [@CachePut](/CachePut)

使用该注解的方法每次执行时都会把返回结果存入缓存中。`一般使用在新增方法上`，可以设置如下属性：
- value：缓存名称（必填），指定缓存的命名空间；
- key：用于设置在命名空间中的缓存key值，可以使用SpEL表达式定义；
- unless：条件符合则不缓存；
- condition：条件符合则缓存。
#### [@CacheEvict](/CacheEvict)
使用该注解的方法执行时会清空指定的缓存。`一般使用在更新或删除方法上`，可以设置如下属性：
- value：缓存名称（必填），指定缓存的命名空间；
- key：用于设置在命名空间中的缓存key值，可以使用SpEL表达式定义；
- condition：条件符合则清空缓存。
### Spring Cache使用步骤

- 在 `pom.xml` 中添加项目依赖；
```xml
<!--redis依赖配置-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```
- 修改配置文件 application. Yml，添加 Redis 的连接配置；
```yaml
spring:
  redis:
    host: localhost # Redis服务器地址
    database: 0 # Redis数据库索引（默认为0）
    port: 6379 # Redis服务器连接端口
    password: # Redis服务器连接密码（默认为空）
    timeout: 1000ms # 连接超时时间
```
- 在启动类上添加@EnableCaching 注解启动缓存功能；
```java
@EnableCaching
@SpringBootApplication
public class MallTinyApplication {

    public static void main(String[] args) {
        SpringApplication.run(MallTinyApplication.class, args);
    }

}
```
- 接下来在 PmsBrandServiceImpl 类中使用相关注解来实现缓存功能，可以发现我们获取品牌详情的方法中使用了@Cacheable 注解，在修改和删除品牌的方法上使用了@CacheEvict 注解；
```java
/**
 * @auther macrozheng
 * @description PmsBrandService实现类
 * @date 2019/4/19
 * @github https://github.com/macrozheng
 */
@Service
public class PmsBrandServiceImpl implements PmsBrandService {
    @Autowired
    private PmsBrandMapper brandMapper;

    @CacheEvict(value = RedisConfig.REDIS_KEY_DATABASE, key = "'pms:brand:'+#id")
    @Override
    public int update(Long id, PmsBrand brand) {
        brand.setId(id);
        return brandMapper.updateByPrimaryKeySelective(brand);
    }

    @CacheEvict(value = RedisConfig.REDIS_KEY_DATABASE, key = "'pms:brand:'+#id")
    @Override
    public int delete(Long id) {
        return brandMapper.deleteByPrimaryKey(id);
    }

    @Cacheable(value = RedisConfig.REDIS_KEY_DATABASE, key = "'pms:brand:'+#id", unless = "#result==null")
    @Override
    public PmsBrand getItem(Long id) {
        return brandMapper.selectByPrimaryKey(id);
    }

}
```
### Redis 存储 JSON 格式数据
让Redis存储的标准的JSON格式数据有利于我们接下来的数据处理，由于不设置过期时间容易产生很多不必要的缓存数据，所以我们还需要设置一定的过期时间。
- 我们可以通过给 RedisTemplate 设置 JSON 格式的序列化器，并通过配置 RedisCacheConfiguration 设置超时时间，此时别忘了去除启动类上的@EnableCaching 注解，具体配置类 RedisConfig 代码如下；
```java
/**
 * @auther macrozheng
 * @description Redis配置类
 * @date 2020/3/2
 * @github https://github.com/macrozheng
 */
@EnableCaching
@Configuration
public class RedisConfig {

    /**
     * redis数据库自定义key
     */
    public  static final String REDIS_KEY_DATABASE="mall";

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
        RedisSerializer<Object> serializer = redisSerializer();
        RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
        redisTemplate.setConnectionFactory(redisConnectionFactory);
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setValueSerializer(serializer);
        redisTemplate.setHashKeySerializer(new StringRedisSerializer());
        redisTemplate.setHashValueSerializer(serializer);
        redisTemplate.afterPropertiesSet();
        return redisTemplate;
    }

    @Bean
    public RedisSerializer<Object> redisSerializer() {
        //创建JSON序列化器
        Jackson2JsonRedisSerializer<Object> serializer = new Jackson2JsonRedisSerializer<>(Object.class);
        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        //必须设置，否则无法将JSON转化为对象，会转化成Map类型
        objectMapper.activateDefaultTyping(LaissezFaireSubTypeValidator.instance,ObjectMapper.DefaultTyping.NON_FINAL);
        serializer.setObjectMapper(objectMapper);
        return serializer;
    }

    @Bean
    public RedisCacheManager redisCacheManager(RedisConnectionFactory redisConnectionFactory) {
        RedisCacheWriter redisCacheWriter = RedisCacheWriter.nonLockingRedisCacheWriter(redisConnectionFactory);
        //设置Redis缓存有效期为1天
        RedisCacheConfiguration redisCacheConfiguration = RedisCacheConfiguration.defaultCacheConfig()
                .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(redisSerializer())).entryTtl(Duration.ofDays(1));
        return new RedisCacheManager(redisCacheWriter, redisCacheConfiguration);
    }

}
```
- 此时我们调用获取商品详情的接口进行测试，会发现 Redis 中已经缓存了标准的 JSON 格式数据，并且超时时间被设置为了1天。
### 使用 Redis 连接池

SpringBoot 1.5.x版本Redis客户端默认是Jedis实现的，SpringBoot 2.x版本中默认客户端是用Lettuce实现的，我们先来了解下Jedis和Lettuce客户端。
#### Jedis vs Lettuce
Jedis在实现上是直连Redis服务，多线程环境下非线程安全，除非使用连接池，为每个 RedisConnection 实例增加物理连接。
Lettuce是一种可伸缩，线程安全，完全非阻塞的Redis客户端，多个线程可以共享一个RedisConnection，它利用Netty NIO框架来高效地管理多个连接，从而提供了异步和同步数据访问方式，用于构建非阻塞的反应性应用程序。
#### 修改配置
- 修改 `application.yml` 添加 Lettuce 连接池配置，用于配置线程数量和阻塞等待时间。
```yaml
spring:
  redis:
    lettuce:
      pool:
        max-active: 8 # 连接池最大连接数
        max-idle: 8 # 连接池最大空闲连接数
        min-idle: 0 # 连接池最小空闲连接数
        max-wait: -1ms # 连接池最大阻塞等待时间，负值表示没有限制

```

## 整合 Redis
- 修改配置文件 `application.yml`，添加 Redis 中自定义 key 的连接配置； 
```yaml
# 自定义redis key
redis:
  key:
    prefix:
      authCode: "portal:authCode:"
    expire:
      authCode: 120 # 验证码超期时间
```
## 实现验证码逻辑
	这里以短信验证码的存储验证为例，具体逻辑如下：生成验证码时，将自定义的Redis键值加上手机号生成一个Redis的key，以验证码为value存入到Redis中，并设置过期时间为自己配置的时间（这里为120s）。校验验证码时根据手机号码来获取Redis里面存储的验证码，并与传入的验证码进行比对。
- 添加 UmsMemberController 类，添加根据电话号码获取验证码的接口和校验验证码的接口；
```java
/**
 * @auther macrozheng
 * @description 会员登录注册管理Controller
 * @date 2018/8/3
 * @github https://github.com/macrozheng
 */
@Controller
@Api(tags = "UmsMemberController")
@Tag(name = "UmsMemberController", description = "会员登录注册管理")
@RequestMapping("/sso")
public class UmsMemberController {
    @Autowired
    private UmsMemberService memberService;

    @ApiOperation("获取验证码")
    @RequestMapping(value = "/getAuthCode", method = RequestMethod.GET)
    @ResponseBody
    public CommonResult getAuthCode(@RequestParam String telephone) {
        return memberService.generateAuthCode(telephone);
    }

    @ApiOperation("判断验证码是否正确")
    @RequestMapping(value = "/verifyAuthCode", method = RequestMethod.POST)
    @ResponseBody
    public CommonResult updatePassword(@RequestParam String telephone,
                                       @RequestParam String authCode) {
        return memberService.verifyAuthCode(telephone,authCode);
    }
}
```
- 添加 UmsMemberService 接口；
```java
/**
 * @auther macrozheng
 * @description 会员管理Service
 * @date 2018/8/3
 * @github https://github.com/macrozheng
 */
public interface UmsMemberService {

    /**
     * 生成验证码
     */
    CommonResult generateAuthCode(String telephone);

    /**
     * 判断验证码和手机号码是否匹配
     */
    CommonResult verifyAuthCode(String telephone, String authCode);

}
```
- 添加UmsMemberService接口的实现类UmsMemberServiceImpl。
```java
/**
 * @auther macrozheng
 * @description 会员管理Service实现类
 * @date 2018/8/3
 * @github https://github.com/macrozheng
 */
@Service
public class UmsMemberServiceImpl implements UmsMemberService {
    @Autowired
    private RedisService redisService;
    @Value("${redis.key.prefix.authCode}")
    private String REDIS_KEY_PREFIX_AUTH_CODE;
    @Value("${redis.key.expire.authCode}")
    private Long AUTH_CODE_EXPIRE_SECONDS;

    @Override
    public CommonResult generateAuthCode(String telephone) {
        StringBuilder sb = new StringBuilder();
        Random random = new Random();
        for (int i = 0; i < 6; i++) {
            sb.append(random.nextInt(10));
        }
        //验证码绑定手机号并存储到redis
        redisService.set(REDIS_KEY_PREFIX_AUTH_CODE + telephone, sb.toString());
        redisService.expire(REDIS_KEY_PREFIX_AUTH_CODE + telephone, AUTH_CODE_EXPIRE_SECONDS);
        return CommonResult.success(sb.toString(), "获取验证码成功");
    }


    //对输入的验证码进行校验
    @Override
    public CommonResult verifyAuthCode(String telephone, String authCode) {
        if (StrUtil.isEmpty(authCode)) {
            return CommonResult.failed("请输入验证码");
        }
        String realAuthCode = (String) redisService.get(REDIS_KEY_PREFIX_AUTH_CODE + telephone);
        boolean result = authCode.equals(realAuthCode);
        if (result) {
            return CommonResult.success(null, "验证码校验成功");
        } else {
            return CommonResult.failed("验证码不正确");
        }
    }

}
```