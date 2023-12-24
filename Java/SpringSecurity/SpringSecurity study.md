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
- 登录
	1. 自定义登录接口
	调用 ProviderManager 的方法进行认证，如果认证通过生成 JWT，把用户信息存入 Redis 中。
	2. 自定义 UserDetailsService 在这个实现类中去查询数据库
- 校验
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
import org.springframework.data.redis.serializer.StringRedisSerializer;  
  
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
5. 实体类（已使用 MybatisX 快速生成）
### 核心代码实现
创建一个类实现 UserDetailService 接口，重写其中的方法，用于从数据库中查询用户信息。
```java
@Service  
public class UserDetailsServiceImpl implements UserDetailsService {  
    @Autowired  
    private UserMapper userMapper;  
  
    @Override  
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {  
  
        //查询用户信息  
        LambdaQueryWrapper<User> queryWrapper = new LambdaQueryWrapper<>();  
        queryWrapper.eq(User::getUserName,username);  
        User user = userMapper.selectOne(queryWrapper);  
  
        //如果没有查询到用户，抛出异常  
        Optional<User> userOptional = Optional.ofNullable(user);  
        try {  
            userOptional.orElseThrow((Supplier<Throwable>) () -> new RuntimeException("用户名或密码错误"));  
        } catch (Throwable e) {  
            throw new RuntimeException(e);  
        }  
        //TODO:查询对应的权限信息  
  
        //把数据封装成UserDetails返回  
        return new LoginUser(user);  
    }  
}
```
UserDetailService 方法的返回值为 UserDetail 类型，需要定义一个类来实现该接口并将用户信息封装到其中。
```java
package com.xinzhou.domain;  
  
import lombok.AllArgsConstructor;  
import lombok.Data;  
import lombok.NoArgsConstructor;  
import org.springframework.security.core.GrantedAuthority;  
import org.springframework.security.core.userdetails.UserDetails;  
  
import java.util.Collection;  
  
@Data  
@AllArgsConstructor  
@NoArgsConstructor  
public class LoginUser implements UserDetails {  
  
    private User user;  
    @Override  
    public Collection<? extends GrantedAuthority> getAuthorities() {  
        return null;  
    }  
  
    @Override  
    public String getPassword() {  
        return user.getPassword();  
    }  
  
    @Override  
    public String getUsername() {  
        return user.getUserName();  
    }  
  
    @Override  
    public boolean isAccountNonExpired() {  
        return true;  
    }  
  
    @Override  
    public boolean isAccountNonLocked() {  
        return true;  
    }  
  
    @Override  
    public boolean isCredentialsNonExpired() {  
        return true;  
    }  
  
    @Override  
    public boolean isEnabled() {  
        return true;  
    }  
}
```
### 测试
需要在表中加入数据，如密码为明文，则需在密码前加入{noop}
![[Pasted image 20231223112232.png]]
![[Pasted image 20231223112341.png]]
### 密码加密存储
项目中我们不会对密码进行明文存储，默认使用 PasswordEncoder 要求数据库中密码格式为{id}password，我们一般不采用该方式
一般使用 SpringSecurity 中提供的 BCryptPasswordEncoder。
我们只需要把 BCryptPasswordEncoder 对象注入到 Spring 容器中，SpringSecurity 就会使用该 PasswordEncoder 来进行密码校验，我们可以定义一个 SpringSecurity 配置类，要求这个配置类继承 WebSecurityConfigurerAdapter。
```java
@Configuration  
public class SecurityConfig extends WebSecurityConfigurerAdapter {  
  
    //创建BCryptPasswordEncoder注入容器  
    @Bean  
    public PasswordEncoder passwordEncoder(){  
        return new BCryptPasswordEncoder();  
    }  
}
```
- 测试
```java
@Autowired  
private PasswordEncoder passwordEncoder;  
  
  
@Test  
public void testPasswordEncoder(){  
    //对密码进行加密  
    String encode = passwordEncoder.encode("1234");  
    System.out.println(encode);  
    //$2a$10$BbaUhmUweyGkexagqQykouGzGMqR7RzmP6AlREaooxuFdOt8aQgmO  
    //将加密后的密文与密码对比  
    boolean flag = passwordEncoder.matches("1234", "$2a$10$BbaUhmUweyGkexagqQykouGzGMqR7RzmP6AlREaooxuFdOt8aQgmO");  
    System.out.println(flag);  
    //true  
}
```
### 登录接口
我们接下来自定义一个登录接口，让 SpringSecurity 对这个接口进行放行。在接口中我们通过 AuthenticationManager 的 authenticate 方法来进行用户认证，所以需要在 SecurityConfig 中配置把 AuthenticationManager 注入容器。
如果认证成功，生成 jwt，放入响应中返回，并且为了让用户下次请求时能通过 jwt 识别具体是哪个用户，我们需要把用户信息存入 redis。
- 自定义登录接口 
```java
@RestController  
public class LoginController {  
  
    @Autowired  
    private LoginService loginService;  
  
    @PostMapping("/user/login")  
    public ResponseResult login(@RequestBody User user){  
        //登录  
        return loginService.login(user);  
    }  
}
```
- 放行配置及注入 AuthentictionManager
```java
@Configuration  
public class SecurityConfig extends WebSecurityConfigurerAdapter {  
  
    //创建BCryptPasswordEncoder注入容器  
    @Bean  
    public PasswordEncoder passwordEncoder(){  
        return new BCryptPasswordEncoder();  
    }  
  
    //在 SecurityConfig 中配置把 AuthenticationManager 注入容器  
    @Bean  
    @Override    public AuthenticationManager authenticationManagerBean() throws Exception {  
        return super.authenticationManagerBean();  
    }  
  
    //让 SpringSecurity 对这个登录接口进行放行  
    @Override  
    protected void configure(HttpSecurity http) throws Exception {  
        http  
                //关闭csrf  
                .csrf().disable()  
                //不通过Session获取SecurityContext  
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)  
                .and()  
                .authorizeRequests()  
                //对于登录接口，允许匿名访问  
                .antMatchers("/user/login").anonymous()  
                //除上面外所有的授权都需要授权认证  
                .anyRequest().authenticated();  
    }  
}
```
- 实现类
```java
package com.xinzhou.service.impl;  
  
import com.xinzhou.domain.LoginUser;  
import com.xinzhou.domain.ResponseResult;  
import com.xinzhou.domain.User;  
import com.xinzhou.service.LoginService;  
import com.xinzhou.utils.JWTUtil;  
import com.xinzhou.utils.RedisCache;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.security.authentication.AuthenticationManager;  
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;  
import org.springframework.security.core.Authentication;  
import org.springframework.stereotype.Service;  
  
import javax.swing.text.html.Option;  
import java.util.HashMap;  
import java.util.Map;  
import java.util.Optional;  
import java.util.function.Supplier;  
  
@Service  
public class LoginServiceImpl implements LoginService {  
    @Autowired  
    private AuthenticationManager authenticationManager;  
  
    @Autowired  
    private RedisCache redisCache;  
    /**  
     * 登录  
     * @return  
     */  
    @Override  
    public ResponseResult login(User user) {  
  
        //创建一个Authentication的实现类对象，用于存储用户名和密码，并作为参数传入authenticationManager的authenticate方法中  
        UsernamePasswordAuthenticationToken authenticationToken = new UsernamePasswordAuthenticationToken(user.getUserName(),user.getPassword());  
        //通过 AuthenticationManager 的 authenticate 方法来进行用户认证  
        Authentication authenticate = authenticationManager.authenticate(authenticationToken);  
  
        //如认证没通过，给出对应提示  
        try {  
            Optional.ofNullable(authenticate)  
                    .orElseThrow((Supplier<Throwable>) () -> new RuntimeException("登录失败"));  
        } catch (Throwable e) {  
            throw new RuntimeException(e);  
        }  
  
        //如果认证成功，生成 jwt,并返回  
        LoginUser loginUser = (LoginUser) authenticate.getPrincipal();  
        String userId = loginUser.getUser().getId().toString();  
        String jwt = JWTUtil.createJWT(userId);  
  
        Map<String,String> map = new HashMap<>();  
        map.put("token",jwt);  
        //把完整的用户信息存入redis,userId作为key  
        redisCache.setCacheObject("login:"+userId,loginUser);  
  
        return new ResponseResult(200,"登录成功",map);  
    }  
}
```
### 认证过滤器 
在 filter 中定义一个 JwtAuthenticationTokenFilter 过滤器
```java
package com.xinzhou.filter;  
  
import com.xinzhou.domain.LoginUser;  
import com.xinzhou.utils.JWTUtil;  
import com.xinzhou.utils.RedisCache;  
import io.jsonwebtoken.Claims;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;  
import org.springframework.security.core.context.SecurityContextHolder;  
import org.springframework.stereotype.Component;  
import org.springframework.util.StringUtils;  
import org.springframework.web.filter.OncePerRequestFilter;  
  
import javax.servlet.FilterChain;  
import javax.servlet.ServletException;  
import javax.servlet.http.HttpServletRequest;  
import javax.servlet.http.HttpServletResponse;  
import java.io.IOException;  
import java.util.Objects;  
  
@Component  
public class JwtAuthenticationTokenFilter extends OncePerRequestFilter {  
    @Autowired  
    private RedisCache redisCache;  
  
    @Override  
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {  
        //获取token  
        String token = request.getHeader("token");  
        //如果没有token则放行  
        if (!StringUtils.hasText(token)) {  
            filterChain.doFilter(request, response);  
            return;  
        }  
        //解析token  
        String userId;  
        try {  
            Claims claims = JWTUtil.parseJwt(token);  
            //获取userId  
            userId = claims.getSubject();  
        } catch (Exception e) {  
            throw new RuntimeException("token非法");  
        }  
  
        //从redis中获取用户信息  
        String redisKey = "login:" + userId;  
        //判断获取到的用户是否为空  
        LoginUser loginUser = redisCache.getCacheObject(redisKey);  
        if(Objects.isNull(loginUser)){  
            throw new RuntimeException("用户未登录");  
        }  
        //存入SecurityContextHolder中，-->Security过滤器链都是从SecurityContextHolder中获取用户信息  
        //使用UsernamePasswordAuthenticationToken三个参数的构造，其中会setAuthenticated(true);  
        //TODO 获取权限信息封装到Authentication中  
        UsernamePasswordAuthenticationToken authenticationToken = new UsernamePasswordAuthenticationToken(loginUser, null, null);  
        SecurityContextHolder.getContext().setAuthentication(authenticationToken);  
        //放行  
        filterChain.doFilter(request,response);  
    }  
}
```
### 配置过滤器
配置认证过来器在 Security 过滤器链中的位置。
```java
package com.xinzhou.config;  
  
import com.xinzhou.filter.JwtAuthenticationTokenFilter;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.context.annotation.Bean;  
import org.springframework.context.annotation.Configuration;  
import org.springframework.security.authentication.AuthenticationManager;  
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;  
import org.springframework.security.config.annotation.web.builders.HttpSecurity;  
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;  
import org.springframework.security.config.http.SessionCreationPolicy;  
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;  
import org.springframework.security.crypto.password.PasswordEncoder;  
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;  
  
@Configuration  
public class SecurityConfig extends WebSecurityConfigurerAdapter {  
  
    //创建BCryptPasswordEncoder注入容器  
    @Bean  
    public PasswordEncoder passwordEncoder(){  
        return new BCryptPasswordEncoder();  
    }  
  
    @Autowired  
    private JwtAuthenticationTokenFilter jwtAuthenticationTokenFilter;  
  
    //在 SecurityConfig 中配置把 AuthenticationManager 注入容器  
    @Bean  
    @Override    public AuthenticationManager authenticationManagerBean() throws Exception {  
        return super.authenticationManagerBean();  
    }  
  
    //让 SpringSecurity 对这个登录接口进行放行  
    @Override  
    protected void configure(HttpSecurity http) throws Exception {  
        http  
                //关闭csrf  
                .csrf().disable()  
                //不通过Session获取SecurityContext  
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)  
                .and()  
                .authorizeRequests()  
                //对于登录接口，允许匿名访问  
                .antMatchers("/user/login").anonymous()  
                //除上面外所有的授权都需要授权认证  
                .anyRequest().authenticated();  
        //配置认证过滤器  
        http.addFilterBefore(jwtAuthenticationTokenFilter, UsernamePasswordAuthenticationFilter.class);  
    }  
}
```
- 测试
![[Pasted image 20231223141801.png]]
### 退出登录
我们只需要定义一个退出登录接口，然后获取 SecurityContextHolder 中的认证信息，再删除 redis 中对应的数据即可。
```java
/**  
 * 退出登录  
 * @return  
 */  
@Override  
public ResponseResult logout() {  
    //获取SecurityContextHolder中的用户id  
    UsernamePasswordAuthenticationToken authentication = (UsernamePasswordAuthenticationToken) SecurityContextHolder.getContext().getAuthentication();  
    LoginUser loginUser = (LoginUser) authentication.getPrincipal();  
  
    Long userId = loginUser.getUser().getId();  
    //删除redis中的值  
    redisCache.deleteObject("login:"+userId);  
    return new ResponseResult(200,"退出成功",null);  
}
```
# 授权
## 基本流程
在 SpringSecurity 中，会默认使用 FilterSecurityInterceptor：负责权限校验的过滤器。在 FilterSecurityInterceptor 中会从 SecurityContextHolder 中获取其中的 Authentication，然后获取其中的权限信息。
所以我们只需当前登录用户的权限信息也存入 Authentication 中，然后设置我们的资源所需权限即可。
## 授权实现
### 限制访问资源所需权限
SpringSecurity 为我们提供了基于注解的权限控制方案，我们可以使用注解去指定访问对应资源所需的权限。
- 在 SecurityConfig 类中开启相关配置
`@EnableGlobalMethodSecurity(prePostEnabled=true)`
- @PreAuthorize
```java
@RestController  
public class HelloController {  
  
    @GetMapping("/hello")  
    @PreAuthorize("hasAuthority('test')")  
    public String hello(){  
        return "hello";  
    }  
}
```
### 封装权限信息
在写 UserDetailsServiceImpl 时，在查询用户后还需获取对应的权限信息，封装到 UserDetails 中返回。
我们之前定义了 UserDetails 的实现了 LoginUser，想让其返回封装的权限信息，需要对其进行改造
```java
package com.xinzhou.domain;  
  
import com.alibaba.fastjson.annotation.JSONField;  
import lombok.Data;  
import lombok.NoArgsConstructor;  
import org.springframework.security.core.GrantedAuthority;  
import org.springframework.security.core.authority.SimpleGrantedAuthority;  
import org.springframework.security.core.userdetails.UserDetails;  
  
import java.util.Collection;  
import java.util.List;  
import java.util.stream.Collectors;  
  
@Data  
@NoArgsConstructor  
public class LoginUser implements UserDetails {  
  
    private User user;  
  
    //存储权限信息  
    private List<String> permissions;  
  
    public LoginUser(User user, List<String> permissions) {  
        this.user = user;  
        this.permissions = permissions;  
    }  
  
    //存储Spring security所需权限集合  
    @JSONField(serialize = false)//不将该属性转成json存入redis  
    private List<SimpleGrantedAuthority> authorities;  
  
    @Override  
    public Collection<? extends GrantedAuthority> getAuthorities() {  
        if(authorities != null){  
            return authorities;  
        }  
        //把permissions中String类型的权限信息封装成SimpleGrantedAuthority对象返回  
        authorities = permissions.stream()  
                .map(SimpleGrantedAuthority::new)  
                .collect(Collectors.toList());  
        return authorities;  
    }  
  
    @Override  
    public String getPassword() {  
        return user.getPassword();  
    }  
  
    @Override  
    public String getUsername() {  
        return user.getUserName();  
    }  
  
    @Override  
    public boolean isAccountNonExpired() {  
        return true;  
    }  
  
    @Override  
    public boolean isAccountNonLocked() {  
        return true;  
    }  
  
    @Override  
    public boolean isCredentialsNonExpired() {  
        return true;  
    }  
  
    @Override  
    public boolean isEnabled() {  
        return true;  
    }  
}
```
- 实现类 UserDetailsServiceImpl 查询对应的权限信息
```java
@Service  
public class UserDetailsServiceImpl implements UserDetailsService {  
    @Autowired  
    private UserMapper userMapper;  
  
    @Override  
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {  
  
        //查询用户信息  
        LambdaQueryWrapper<User> queryWrapper = new LambdaQueryWrapper<>();  
        queryWrapper.eq(User::getUserName,username);  
        User user = userMapper.selectOne(queryWrapper);  
  
        //如果没有查询到用户，抛出异常  
        Optional<User> userOptional = Optional.ofNullable(user);  
        try {  
            userOptional.orElseThrow((Supplier<Throwable>) () -> new RuntimeException("用户名或密码错误"));  
        } catch (Throwable e) {  
            throw new RuntimeException(e);  
        }  
        //TODO:查询对应的权限信息  
        List<String> list = new ArrayList<>(Arrays.asList("test","admin"));  
  
  
        //把数据封装成UserDetails返回  
        return new LoginUser(user,list);  
    }  
}
```
- 在认证拦截器中获取权限信息封装到 Authentication中
```java
@Component  
public class JwtAuthenticationTokenFilter extends OncePerRequestFilter {  
    @Autowired  
    private RedisCache redisCache;  
  
    @Override  
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {  
        //获取token  
        String token = request.getHeader("token");  
        //如果没有token则放行  
        if (!StringUtils.hasText(token)) {  
            filterChain.doFilter(request, response);  
            return;  
        }  
        //解析token  
        String userId;  
        try {  
            Claims claims = JWTUtil.parseJwt(token);  
            //获取userId  
            userId = claims.getSubject();  
        } catch (Exception e) {  
            throw new RuntimeException("token非法");  
        }  
  
        //从redis中获取用户信息  
        String redisKey = "login:" + userId;  
        //判断获取到的用户是否为空  
        LoginUser loginUser = redisCache.getCacheObject(redisKey);  
        if(Objects.isNull(loginUser)){  
            throw new RuntimeException("用户未登录");  
        }  
        //存入SecurityContextHolder中，-->Security过滤器链都是从SecurityContextHolder中获取用户信息  
        //使用UsernamePasswordAuthenticationToken三个参数的构造，其中会setAuthenticated(true);  
        //TODO 获取权限信息封装到Authentication中  
  
        UsernamePasswordAuthenticationToken authenticationToken = new UsernamePasswordAuthenticationToken(loginUser, null, loginUser.getAuthorities());  
        SecurityContextHolder.getContext().setAuthentication(authenticationToken);  
        //放行  
        filterChain.doFilter(request,response);  
    }  
}
```
### 从数据库查询权限信息
#### RBAC 权限模型
Role-Based Access Control：即基于角色的权限控制
![[Pasted image 20231223160659.png]]
#### 准备工作
- 创建数据库表
```sql
DROP TABLE IF EXISTS sys_menu;
CREATE TABLE sys_menu (
    id bigint(20) NOT NULL AUTO_INCREMENT,
    menu_name varchar(64) NOT NULL DEFAULT 'NULL' COMMENT '菜单名',
    path varchar(200) DEFAULT 'NULL' COMMENT '路由地址',
    component varchar(255) DEFAULT 'NULL' COMMENT '组件路径',
    visible char(1) DEFAULT '0' COMMENT '菜单状态 (0显示 隐藏)',
    status char(1) DEFAULT '0' COMMENT '菜单状态 (0正常 1停用)',
    perms varchar(100) DEFAULT 'NULL' COMMENT '权限标识',
    icon varchar(100) DEFAULT '#' COMMENT '菜单图标',
    create_by bigint(20),
    create_time datetime,
    update_by bigint(20),
    update_time datetime,
    del_flag int(1) DEFAULT '0' COMMENT '是否删除 (0未删除 1已删除)',
    remark varchar(500) DEFAULT 'NULL' COMMENT '备注',
    PRIMARY KEY (id)
) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8mb4 COMMENT='菜单表';
```
```sql
DROP TABLE IF EXISTS sys_role;
CREATE TABLE sys_role (
	id bigint(20) NOT NULL AUTO_INCREMENT,
	name varchar(128) DEFAULT NULL,
	role_key varchar(100) DEFAULT NULL COMMENT '角色权限字符串',
	status char(1) DEFAULT '0' COMMENT '角色状态 (0正常 1停用)',
	del_flag int(1) DEFAULT '0' COMMENT '是否删除 (0未删除 1已删除)',
	create_by bigint(200),
	create_time datetime,
	update_by bigint(200),
	update_time datetime,
	remark varchar(500) DEFAULT 'NULL' COMMENT '备注',
	PRIMARY KEY (id)
)ENGINE=InnoDB AUTO_INCREMENT=3 DEFAULT CHARSET=utf8mb4 COMMENT='角色表';
```
```sql
DROP TABLE IF EXISTS sys_role_menu;
CREATE TABLE sys_role_menu (
	role_id bigint(200) NOT NULL AUTO_INCREMENT COMMENT '角色ID',
	menu_id bigint(200) NOT NULL DEFAULT '0' COMMENT '菜单id',
	PRIMARY KEY (role_id ,menu_id)
)ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8mb4;
```
```sql
DROP TABLE IF EXISTS sys_user_role;
CREATE TABLE sys_user_role(
	user_id bigint(200) NOT NULL AUTO_INCREMENT COMMENT'用户id',
	role_id bigint(200) NOT NULL DEFAULT '0' COMMENT '角色id',
	PRIMARY KEY (user_id,role_id)
)ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```
- 查询用户权限信息的 SQL 编写
	根据 userid 查询 perms 对应的 role 和 menu 都必须为正常状态
```sql
SELECT 
			DISTINCT m.perms
FROM `sys_user_role` ur
	LEFT JOIN sys_role r on ur.role_id = r.id
	LEFT JOIN sys_role_menu rm on ur.role_id = rm.role_id
	LEFT JOIN sys_menu m on m.id = rm.menu_id
WHERE
		user_id = 2
		AND r.status = 0 AND m.status = 0
```
- 创建 menu 表实体类（MybaitsX 生成）
#### 代码实现
我们需要根据用户 id 查询到其所对应的权限信息
```java
public interface MenuMapper extends BaseMapper<Menu> {  
  
    List<String> selectPermsByUserId(Long userId);  
  
}
```
```xml
<select id="selectPermsByUserId" resultType="java.lang.String">  
    SELECT  
        DISTINCT m.perms    FROM `sys_user_role` ur             LEFT JOIN sys_role r on ur.role_id = r.id             LEFT JOIN sys_role_menu rm on ur.role_id = rm.role_id             LEFT JOIN sys_menu m on m.id = rm.menu_id    WHERE        user_id = #{userId}      AND r.status = 0 AND m.status = 0
</select>
```
- 修改 UserDetailsServiceImpl 中的 TODO: 查询对应的权限信息
```java
List<String> list = menuMapper.selectPermsByUserId(user.getId());
```
# 自定义失败处理
我们希望在认证失败或授权失败的情况下也能和我们的接口一样返回相同结构的 json，这样可以让前端进行统一处理。
在 SpringSecurity 中，认证和授权过程中的异常会被 ExceptionTranslationFilter 捕获。
如果是认证过程中的异常会被封装成 AuthenticationException 然后调用 AuthenticationEntryPoint 对象的方法进行异常处理。
如果是授权过程中出现的异常会被封装从 AccessDeniedException 然后调用 AccessDeniedHandler 对象方法进行异常处理。
所以我们需要自定义异常处理，只需自定义 AuthenticationEntryPoint 和 AccessDeniedHandler 然后配置给 SpringSecurity 即可。
- 自定义异常类
```java
@Component  
public class AuthenticationEntryPointImpl implements AuthenticationEntryPoint {  
    @Override  
    public void commence(HttpServletRequest request, HttpServletResponse response, AuthenticationException authException) throws IOException, ServletException {  
        ResponseResult result = new ResponseResult<>(HttpStatus.UNAUTHORIZED.value(), "用户认证失败，请重新登录");  
        String json = JSON.toJSONString(result);  
        //将json写入到响应体中  
        WebUtils.renderString(response,json);  
    }  
}
```
```java
@Component  
public class AccessDeniedHandlerImpl implements AccessDeniedHandler {  
    @Override  
    public void handle(HttpServletRequest request, HttpServletResponse response, AccessDeniedException accessDeniedException) throws IOException, ServletException {  
        ResponseResult result = new ResponseResult<>(HttpStatus.UNAUTHORIZED.value(), "您的权限不足");  
        String json = JSON.toJSONString(result);  
        //将json写入到响应体中  
        WebUtils.renderString(response,json);  
    }  
}
```
- 对 SecurityConfig 中配置异常处理器
```java
@Configuration  
@EnableGlobalMethodSecurity(prePostEnabled = true)  
public class SecurityConfig extends WebSecurityConfigurerAdapter {  
  
    //创建BCryptPasswordEncoder注入容器  
    @Bean  
    public PasswordEncoder passwordEncoder(){  
        return new BCryptPasswordEncoder();  
    }  
  
    @Autowired  
    private JwtAuthenticationTokenFilter jwtAuthenticationTokenFilter;  
  
    @Autowired  
    private AuthenticationEntryPoint authenticationEntryPoint;  
    @Autowired  
    private AccessDeniedHandler accessDeniedHandler;  
  
    //在 SecurityConfig 中配置把 AuthenticationManager 注入容器  
    @Bean  
    @Override    public AuthenticationManager authenticationManagerBean() throws Exception {  
        return super.authenticationManagerBean();  
    }  
  
    //让 SpringSecurity 对这个登录接口进行放行  
    @Override  
    protected void configure(HttpSecurity http) throws Exception {  
        http  
                //关闭csrf  
                .csrf().disable()  
                //不通过Session获取SecurityContext  
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)  
                .and()  
                .authorizeRequests()  
                //对于登录接口，允许匿名访问  
                .antMatchers("/user/login").anonymous()  
                //除上面外所有的授权都需要授权认证  
                .anyRequest().authenticated();  
        //配置认证过滤器  
        http.addFilterBefore(jwtAuthenticationTokenFilter, UsernamePasswordAuthenticationFilter.class);  
  
        //配置异常处理器  
        http.exceptionHandling()  
                //配置认证失败处理器  
                .authenticationEntryPoint(authenticationEntryPoint)  
                //配置授权失败处理器  
                .accessDeniedHandler(accessDeniedHandler);  
    }  
}
```
# 跨域
浏览器出于安全考虑，使用 XMLHttpRequest 对象发起 HTTP 请求时必须遵循同源策略，否则就是跨域的 HTTP 请求，默认情况下是被禁止的。同源策略要求源相同才能正常进行通信，即协议、域名、端口号都完全一致。
- 对 SpringBoot 配置，允许跨域请求
```java
package com.xinzhou.config;  
  
import org.springframework.context.annotation.Configuration;  
import org.springframework.web.servlet.config.annotation.CorsRegistry;  
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;  
@Configuration  
public class CorsConfig implements WebMvcConfigurer {  
    public void addCorsMapping(CorsRegistry registry){  
        //设置允许跨域的路径  
        registry.addMapping("/**")  
                //设置允许跨域请求的域名  
                .allowedOriginPatterns("*")  
                //是否允许cookie  
                .allowCredentials(true)  
                //设置允许的请求方式  
                .allowedMethods("GET","POST","DELETE","PUT")  
                //设置允许的header属性  
                .allowedHeaders("*")  
                //跨域允许时间  
                .maxAge(3600);  
    }  
}
```
- 在 SpringSecurity 的配置文件SecurityConfig 中配置允许跨域
```java
//配置允许跨域  
http.cors();
```
# 遗留问题解决
## 其他权限校验方法
我们前面都是使用@PreAuthorize 注解，然后在其中使用的是 hasAuthority 方法进行校验。SpringSecurity 还提供了如 hasAnyAuthority、hasRole、hasAnyRole...
**hasAuthority** 原理：该方法实际执行到了 SecurityExpressionRoot 的 hasAuthority，它内部其实调用了 authentication 的 getAuthorities 方法互殴去用户的权限列表。然后判断我们存入的方法参数是否存在权限列表中。
**hasAnyAuthority** 方法可以传入多个权限，用户只要用其中一个权限都可以访问对应资源。
hasRole 要求有对应角色权限才可以访问，但他内部会把我们传入的参数拼接 ROLE_再去比较，所以这种情况需要用户对应的权限也有 ROLE_这个前缀才可以。
hasAnyRole 有任意角色就可以访问，他内部也会把我们传入的参数拼接 ROLE_再去比较。
## 自定义权限校验方法
我们可以自定义自己的权限方法，再@PreAuthorize 注解中使用我们的方法
```java
package com.xinzhou.expression;  
  
import com.xinzhou.domain.LoginUser;  
import org.springframework.security.core.Authentication;  
import org.springframework.security.core.context.SecurityContextHolder;  
import org.springframework.stereotype.Component;  
  
import java.util.List;  
  
@Component("ex")  
public class MyExpressionRoot {  
    public boolean hasAuthority(String authority){  
        //获取当前用户的权限  
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();  
        LoginUser loginUser = (LoginUser) authentication.getPrincipal();  
        List<String> permissions = loginUser.getPermissions();  
        //判断用户权限集合中是否存在authority  
        return permissions.contains(authority);  
    }  
}
```
- 指定使用自定义的权限校验方法 (再 SPEL 表达式中使用@ex 相当于获取容器中 bean 的名字为 ex 的对象。然后再调用这个对象的方法)
```java
@RestController  
public class HelloController {  
  
    @GetMapping("/hello")  
    @PreAuthorize("@ex.hasAuthority('system:dept:list')")  
    public String hello(){  
        return "hello";  
    }  
}
```
## 基于配置的权限控制
可以再 SecurityConfig 中配置权限控制
```java
        http   
                .authorizeRequests()  
                //对于登录接口，允许匿名访问  
                .antMatchers("/user/login").anonymous()
                //配置权限控制
                        .antMatchers("/user/hello").hasAuthority("system:dept:list")
```

## CSRF
CSRF 是指跨站请求伪造（Cross-site request forgery），是 web 常见的攻击之一。
SpringSecurity 去防止 CSRF 攻击的方式是通过后端生成一个 csrf_token，前端发起请求时需要携带该 token，后端会进行校验，没有携带就是伪造的不会允许访问。
前后端分离的项目的认证信息就是 token，而 token 不需要存储到 cookie 中，并且前端代码去把 token 设置到请求头中才可以，所以 CSRF 攻击就不用考虑了，我们可以在SpringSecurity配置中将 CSRF 关闭。
## 认证成功处理器
UsernamePasswordAuthenticationFilter 进行登录认证时，如果认证成功了会调用 AuthenticationSucceesHandler 的方法进行认证失败后的处理。AuthenticationSucceesHandler 就是认证失败处理器

## 认证失败处理器
UsernamePasswordAuthenticationFilter 进行登录认证时，如果认证失败了会调用 AuthenticationFailureHandler 的方法进行认证失败后的处理。AuthenticationFailureHandler 就是认证失败处理器
## 其他认证方案
# 源码解读
