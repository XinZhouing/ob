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
