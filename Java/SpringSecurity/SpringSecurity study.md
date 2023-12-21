# 快速入门
## 准备工作：创建一个 SpringBoot 工程
## 引入 SpringSecurity
```java
<dependency>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-starter-security</artifactId>  
</dependency>
```
- 引入依赖后我们在尝试访问之前的接口会自动跳转到 SpringSecurity 的默认登录页面，默认用户是 user，密码会输出在控制台。
# 认证
## 登录校验流程
![[Pasted image 20231216164106.png]]
## SpringSecurity 完整流程
- SpringSecurity 的原理其实就是一个过滤器链。
![[Pasted image 20231216170059.png]]
- UsernamePasswordAuthenticationFilter：负责处理我们在登录页填写了用户名密码后的登录请求
- ExceptionTranslationFilter：处理过滤器链中抛出的任何 AccessDeniedException 和 AuthenticationException。
- FilterSecurityInterceptor：负责权限校验的过滤器
	 我们可一通过 Debug 调试查看当前系统中 SpringSecurity 过滤器链中有哪些过滤器及他们的顺序
![[Pasted image 20231216194405.png]]
## 认证流程详解
![[Pasted image 20231216221225.png]]
- Authentication 接口：他的实现类，表示当前访问系统的用户，封装了用户相关信息。
- AuthenticationManager 接口：定义了认证 Authentication 的方法。
- UserDetailsService 接口：加载用户特定数据的核心接口。里面定义了一个根据用户名查询用户信息的方法。
- UserDetails 接口：提供核心用户信息。通过 UserDetailService 根据用户名获取处理的用户信息要封装成 UserDetails 对象返回。然后将这些信息封装到 Authentication 对象中。
## 解决问题
### 登录
1. 自定义登录接口
	调用 ProviderManager 的方法进行认证，如果认证通过生成 JWT，把用户信息存入 Redis 中。
1. 自定义 UserDetailsService 在这个实现类中去查询数据库
### 校验
1. 定义 JWT 认证过滤器
	获取 token
	解析 token 获取其中的 userid
	从 redis 中获取用户信息
	存入SecurityContextHolder

### 实现
- 数据库准备
```sql
CREATE TABLE sys_user ( id BIGINT(20) NOT NULL AUTO_INCREMENT COMMENT '主键', user_name VARCHAR(64) NOT NULL COMMENT '用户名', nick_name VARCHAR(64) NOT NULL COMMENT '昵称', password VARCHAR(64) NOT NULL COMMENT '密码', status CHAR(1) DEFAULT '0' COMMENT '账号状态（0正常，1停用）', email VARCHAR(64) COMMENT '邮箱', phonenumber VARCHAR(32) COMMENT '手机号', sex CHAR(1) COMMENT '用户性别（0男，1女，2未知）', avatar VARCHAR(128) COMMENT '头像', user_type CHAR(1) NOT NULL DEFAULT '1' COMMENT '用户类型（0管理员，1普通用户）', create_by BIGINT(20) COMMENT '创建人的用户id', create_time DATETIME COMMENT '创建时间', update_by BIGINT(20) COMMENT '更新人', update_time DATETIME COMMENT '更新时间', del_flag INT(11) DEFAULT '0' COMMENT '删除标志（0代表未删除，1代表已删除）', PRIMARY KEY (id) ) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8mb4 COMMENT='用户表';
```
- 引入 MybatisPlus 和 Mysql 驱动依赖
```xml
<dependency>  
    <groupId>com.baomidou</groupId>  
    <artifactId>mybatis-plus-boot-starter</artifactId>  
    <version>3.0.5</version>  
</dependency>  
<dependency>  
    <groupId>mysql</groupId>  
    <artifactId>mysql-connector-java</artifactId>  
</dependency>
```
- 配置数据库信息
```ymal
server:  
  port: 8081  
  
spring:  
  datasource:  
    url: jdbc:mysql://localhost:3306/securitydb?characterEncoding=utf-8&serverTimezone=UTC  
    username: root  
    password:  
    driver-class-name: com.mysql.cj.jdbc.Driver
```
- 利用 MybatisX 插件快速生成代码
### 准备工作
1. 添加依赖
```xml
<!--redis依赖-->  
<dependency>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-starter-data-redis</artifactId>  
</dependency>  
<!--fastjson依赖-->  
<dependency>  
    <groupId>com.alibaba</groupId>  
    <artifactId>fastjson</artifactId>  
    <version>1.2.76</version>  
</dependency>  
<!--jwt依赖-->  
<dependency>  
    <groupId>io.jsonwebtoken</groupId>  
    <artifactId>jjwt</artifactId>  
    <version>0.9.1</version>  
</dependency>
```
2. 添加 Redis 相关配置
- FastJsonRedisSerializer
```java
package org.study.utils;  
  
import com.alibaba.fastjson.JSON;  
import com.alibaba.fastjson.parser.ParserConfig;  
import com.alibaba.fastjson.serializer.SerializerFeature;  
import com.fasterxml.jackson.databind.JavaType;  
import com.fasterxml.jackson.databind.type.TypeFactory;  
import org.springframework.data.redis.serializer.RedisSerializer;  
import org.springframework.data.redis.serializer.SerializationException;  
  
import java.nio.charset.Charset;  
  
/**  
 * Redis使用FastJson序列化  
 * @param <T>  
 */  
public class FastJsonRedisSerializer<T> implements RedisSerializer<T> {  
    public static final Charset DEFAULT_CHARSET = Charset.forName("UTF-8");  
  
    private Class<T> clazz;  
  
    static  
    {  
        ParserConfig.getGlobalInstance().setAutoTypeSupport(true);  
    }  
  
    public FastJsonRedisSerializer(Class<T> clazz) {  
        super();  
        this.clazz = clazz;  
    }  
  
    public byte[] serialize(T t) throws SerializationException {  
        if (t == null) {  
            return new byte[0];  
        }  
        return JSON.toJSONString(t, SerializerFeature.WriteClassName).getBytes(DEFAULT_CHARSET);  
    }  
  
    public T deserialize(byte[] bytes) throws SerializationException {  
        if (bytes == null || bytes.length <= 0) {  
            return null;  
        }  
        String str = new String(bytes, DEFAULT_CHARSET);  
  
        return (T) JSON.parseObject(str, clazz);  
    }  
  
    protected JavaType getJavaType(Class<?> clazz)  
    {  
        return TypeFactory.defaultInstance().constructType(clazz);  
    }  
}
```
- RedisConfig
```java
package org.study.config;  
  
import org.springframework.context.annotation.Bean;  
import org.springframework.context.annotation.Configuration;  
import org.springframework.data.redis.connection.RedisConnectionFactory;  
import org.springframework.data.redis.core.RedisTemplate;  
import org.springframework.data.redis.serializer.StringRedisSerializer;  
import org.study.utils.FastJsonRedisSerializer;  
  
@Configuration  
public class RedisConfig {  
    @Bean  
    @SuppressWarnings(value = {"unchecked","rawtypes"})  
    public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory connectionFactory) {  
        RedisTemplate<Object, Object> template = new RedisTemplate<>();  
        template.setConnectionFactory(connectionFactory);  
  
        FastJsonRedisSerializer serializer = new FastJsonRedisSerializer(Object.class);  
  
        //使用StringRedisSerializer来序列化和反序列化redis的key值  
        template.setKeySerializer(new StringRedisSerializer());  
        template.setValueSerializer(serializer);  
  
        //Hash的key也采用StringRedisSerializer的序列化方式  
        template.setHashKeySerializer(new StringRedisSerializer());  
        template.setHashValueSerializer(serializer);  
  
        template.afterPropertiesSet();  
        return template;  
    }  
}
```
3. 响应类
```java
package org.study.domain;  
  
import lombok.Data;  
  
@Data  
public class ResponseResult <T> {  
  
    /**  
     * 状态码  
     */  
    private Integer code;  
  
    /**  
     * 提示信息  
     */  
    private String msg;  
  
    /**  
     * 查询到的结果数据  
     */  
    private T data;  
  
}
```
4. 工具类
- JWTUtil
```java
package org.study.utils;  
  
import io.jsonwebtoken.Claims;  
import io.jsonwebtoken.JwtBuilder;  
import io.jsonwebtoken.Jwts;  
import io.jsonwebtoken.SignatureAlgorithm;  
  
import javax.crypto.SecretKey;  
import javax.crypto.spec.SecretKeySpec;  
import java.util.Base64;  
import java.util.Date;  
import java.util.UUID;  
  
/**  
 * JWT工具类  
 */  
public class JWTUtil {  
  
    //有效期为  
    public static final Long JWT_TTL = 60 * 60 * 1000L;  
    //设置密钥明文  
    public static final String JWT_KEY = "study";  
  
    public static String getUUID(){  
        String token = UUID.randomUUID().toString().replaceAll("-","");  
        return token;  
    }  
  
    /**  
     * 生成JWT  
     * @param subject token中要存放的数据（json格式）  
     * @return  
     */  
    public static String createJWT(String subject){  
        JwtBuilder builder = getJwtBuilder(subject,null, getUUID());  
        return builder.compact();  
    }  
    /**  
     * 生成JWT  
     * @param subject token中要存放的数据（json格式）  
     * @param ttlMillis token超时时间  
     * @return  
     */  
    public static String createJWT(String subject ,Long ttlMillis){  
        JwtBuilder builder = getJwtBuilder(subject, ttlMillis, getUUID());  
        return builder.compact();  
    }  
    private static JwtBuilder getJwtBuilder(String subject,Long ttlMillis,String uuid){  
        SignatureAlgorithm signatureAlgorithm = SignatureAlgorithm.HS256;  
        SecretKey secretKey = generaKey();  
        long nowMillis = System.currentTimeMillis();  
        Date now = new Date(nowMillis);  
        if(ttlMillis == null){  
            ttlMillis = JWTUtil.JWT_TTL;  
        }  
        long expMillis = nowMillis + ttlMillis;  
        Date expDate = new Date(expMillis);  
        return Jwts.builder()  
                .setId(uuid)                            //唯一id  
                .setSubject(subject)                    //主题：可以是json数据  
                .setIssuer("xz")                        //签发者  
                .setIssuedAt(now)                       //签发时间  
                .signWith(signatureAlgorithm,secretKey) //使用HS256对称加密算法签名，第二个参数为密钥  
                .setExpiration(expDate);  
    }  
  
  
    /**  
     * 创建token  
     * @param id  
     * @param subject  
     * @param ttlMillis  
     * @return  
     */  
    public static String createJWT(String id,String subject ,Long ttlMillis){  
        JwtBuilder builder = getJwtBuilder(subject, ttlMillis, id);  
        return builder.compact();  
    }  
  
    public static void main(String[] args) throws Exception {  
        String token =  
                "eysfsahdfjkkkla.fdajfajfafa.fdalfdaljfaoifuweefw.fhhgioisaiidf.fhdaskhvfdjsfhas";  
        Claims claims = parseJwt(token);  
        System.out.println(claims);  
    }  
  
    /**  
     * 生成加密后的密钥 secretKey  
     * @return  
     */  
    private static SecretKey generaKey() {  
        byte[] encodeKey = Base64.getDecoder().decode(JWTUtil.JWT_KEY);  
        SecretKey key = new SecretKeySpec(encodeKey, 0, encodeKey.length, "AES");  
        return key;  
    }  
  
    /**  
     * 解析  
     * @param jwt  
     * @return  
     */  
    private static Claims parseJwt(String jwt) throws Exception {  
        SecretKey secretKey = generaKey();  
        return Jwts.parser()  
                .setSigningKey(secretKey)  
                .parseClaimsJws(jwt)  
                .getBody();  
    }  
}
```
- RedisCache
```java
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.data.redis.core.BoundSetOperations;  
import org.springframework.data.redis.core.HashOperations;  
import org.springframework.data.redis.core.RedisTemplate;  
import org.springframework.data.redis.core.ValueOperations;  
import org.springframework.stereotype.Component;  
  
import java.util.*;  
import java.util.concurrent.TimeUnit;  
  
@Component  
public class RedisCache {  
    @Autowired  
    public RedisTemplate redisTemplate;  
  
    /**  
     * 缓存基本的对象，Integer、String、实体类等  
     *  
     * @param key   缓存的键值  
     * @param value 缓存的值  
     */  
    public <T> void setCacheObject(final String key, final T value) {  
        redisTemplate.opsForValue().set(key, value);  
    }  
  
    /**  
     * 缓存基本的对象，Integer、String、实体类等  
     *  
     * @param key      缓存的键值  
     * @param value    缓存的值  
     * @param timeout  时间  
     * @param timeUnit 时间颗粒度  
     */  
    public <T> void setCacheObject(final String key, final T value, final Integer timeout, final TimeUnit timeUnit) {  
        redisTemplate.opsForValue().set(key, value, timeout, timeUnit);  
    }  
  
    /**  
     * 设置有效时间  
     *  
     * @param key     Redis键  
     * @param timeout 超时时间  
     * @return true=设置成功；false=设置失败  
     */  
    public boolean expire(final String key, final long timeout) {  
        return expire(key, timeout, TimeUnit.SECONDS);  
    }  
  
    /**  
     * 设置有效时间  
     *  
     * @param key     Redis键  
     * @param timeout 超时时间  
     * @param unit    时间单位  
     * @return true=设置成功；false=设置失败  
     */  
    public boolean expire(final String key, final long timeout, final TimeUnit unit) {  
        return redisTemplate.expire(key, timeout, unit);  
    }  
  
    /**  
     * 获得缓存的基本对象。  
     *  
     * @param key 缓存键值  
     * @return 缓存键值对应的数据  
     */  
    public <T> T getCacheObject(final String key) {  
        ValueOperations<String, T> operation = redisTemplate.opsForValue();  
        return operation.get(key);  
    }  
  
    /**  
     * 删除单个对象  
     *  
     * @param key  
     */  
    public boolean deleteObject(final String key) {  
        return redisTemplate.delete(key);  
    }  
  
    /**  
     * 删除集合对象  
     *  
     * @param collection 多个对象  
     * @return  
     */  
    public long deleteObject(final Collection collection) {  
        return redisTemplate.delete(collection);  
    }  
  
    /**  
     * 缓存List数据  
     *  
     * @param key      缓存的键值  
     * @param dataList 待缓存的List数据  
     * @return 缓存的对象  
     */  
    public <T> long setCacheList(final String key, final List<T> dataList) {  
        Long count = redisTemplate.opsForList().rightPushAll(key, dataList);  
        return count == null ? 0 : count;  
    }  
  
    /**  
     * 获得缓存的list对象  
     *  
     * @param key 缓存的键值  
     * @return 缓存键值对应的数据  
     */  
    public <T> List<T> getCacheList(final String key) {  
        return redisTemplate.opsForList().range(key, 0, -1);  
    }  
  
    /**  
     * 缓存Set  
     *     * @param key     缓存键值  
     * @param dataSet 缓存的数据  
     * @return 缓存数据的对象  
     */  
    public <T> BoundSetOperations<String, T> setCacheSet(final String key, final Set<T> dataSet) {  
        BoundSetOperations<String, T> setOperation = redisTemplate.boundSetOps(key);  
        Iterator<T> it = dataSet.iterator();  
        while (it.hasNext()) {  
            setOperation.add(it.next());  
        }  
        return setOperation;  
    }  
  
    /**  
     * 获得缓存的set  
     *     * @param key  
     * @return  
     */  
    public <T> Set<T> getCacheSet(final String key) {  
        return redisTemplate.opsForSet().members(key);  
    }  
  
    /**  
     * 缓存Map  
     *     * @param key  
     * @param dataMap  
     */  
    public <T> void setCacheMap(final String key, final Map<String, T> dataMap) {  
        if (dataMap != null) {  
            redisTemplate.opsForHash().putAll(key, dataMap);  
        }  
    }  
  
    /**  
     * 获得缓存的Map  
     *     * @param key  
     * @return  
     */  
    public <T> Map<String, T> getCacheMap(final String key) {  
        return redisTemplate.opsForHash().entries(key);  
    }  
  
    /**  
     * 往Hash中存入数据  
     *  
     * @param key   Redis键  
     * @param hKey  Hash键  
     * @param value 值  
     */  
    public <T> void setCacheMapValue(final String key, final String hKey, final T value) {  
        redisTemplate.opsForHash().put(key, hKey, value);  
    }  
  
    /**  
     * 获取Hash中的数据  
     *  
     * @param key  Redis键  
     * @param hKey Hash键  
     * @return Hash中的对象  
     */  
    public <T> T getCacheMapValue(final String key, final String hKey) {  
        HashOperations<String, String, T> opsForHash = redisTemplate.opsForHash();  
        return opsForHash.get(key, hKey);  
    }  
  
    /**  
     * 删除Hash中的数据  
     *  
     * @param key  
     * @param hkey  
     */  
    public void delCacheMapValue(final String key, final String hkey) {  
        HashOperations hashOperations = redisTemplate.opsForHash();  
        hashOperations.delete(key, hkey);  
    }  
  
    /**  
     * 获取多个Hash中的数据  
     *  
     * @param key   Redis键  
     * @param hKeys Hash键集合  
     * @return Hash对象集合  
     */  
    public <T> List<T> getMultiCacheMapValue(final String key, final Collection<Object> hKeys) {  
        return redisTemplate.opsForHash().multiGet(key, hKeys);  
    }  
  
    /**  
     * 获得缓存的基本对象列表  
     *  
     * @param pattern 字符串前缀  
     * @return 对象列表  
     */  
    public Collection<String> keys(final String pattern) {  
        return redisTemplate.keys(pattern);  
    }  
}
```
- WebUtils
```java
import javax.servlet.http.HttpServletResponse;  
import java.io.IOException;  
  
public class WebUtils {  
  
    /**  
     * 将字符串渲染到客户端  
     * @param response 渲染对象  
     * @param string 待渲染的字符串  
     * @return  
     */  
    public static String renderString(HttpServletResponse response,String string){  
        try {  
            response.setStatus(200);  
            response.setContentType("application/json");  
            response.setCharacterEncoding("utf-8");  
            response.getWriter().print(string);  
        }  
        catch (IOException e){  
            e.printStackTrace();  
        }  
        return null;  
    }  
}
```
5. 实体类（已快速生成）