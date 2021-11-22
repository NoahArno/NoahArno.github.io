# 1、SpringMVC细节

1、对于一些常用的页面跳转，比如

```java
@GetMapping("/login.html") 
public String loginPage() {
    return "login";
}
```

类似于这些，只包含页面渲染，而不包含任何其余的逻辑的代码，都可以通过一个配置类来代替

```java
@Configuration
public class GulimallWebConfig implements WebMvcConfigurer {
    // 视图映射，默认都是get请求
    public void addViewControllers(ViewControllerRegistry registry) {
        registry.addViewController("/login.html").setViewName("login");
    }
}
```

2、**重定向携带数据**

使用**RedirectAttributes**来模拟重定向携带数据。

原理：利用session原理，将数据放在session中，然后重定向到页面之后，再从session中取出来，跳到下一个页面取出数据之后，就删除session中的数据

`redirectAttributes.addFlashAttribute("errors", errors);`：添加数据到重定向中

也可以使用`redirectAttributes.addAttribute("skuId", skuId);`将数据拼接到重定向的url后面

但是在分布式下，session会存在相应的问题，这时候就可以用SpringSession来解决。

# 2、SpringSession

1. **session原理**：以登录为案例，在进行第一次访问服务器的时候，登录成功后将用户信息保存到session中，然后就会命令浏览器保存一个jsessionid = 123的cookie，以后访问就会带上这个cookie，当浏览器关闭的时候就会清除会话cookie，下次访问的时候没有jsessionid，就会再创建一个
2. 值得注意的是，session不能跨不同域名共享；同一个服务，复制多份，session不同步问题（重定向携带数据就会有这个问题）；不同服务之间session不能共享问题；
3. **解决方案--session复制**：将所有的服务器的session都进行同步，但是session的同步需要数据传输，占用大量网络带宽，降低了服务器群的业务处理能力，方案不可取
4. **解决方案--客户端存储**：服务器不存储session，用户保存自己的session信息到cookie中，节省服务器资源，但是session数据放在cookie中，存在泄漏、篡改、窃取等安全隐患。
5. **解决方案--hash一致性**
6. **解决方案--统一存储**：将所有的session都存储在中间件redis中



1、导入相关依赖

```xml
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>
```

2、添加redis相关的配置，并且可以配置springsession相关配置

```properties
spring.session.store-type=redis
#使用redis作为springsession的存储中间件
#设置过期时间
server.servlet.session.timeout=30m
```

3、添加注解`@EnableRedisHttpSession`

4、添加配置类解决子域session不共享问题以及配置redis序列化

```java
@Configuration
public class GulimallSessionConfig {
    @Bean
    public CookieSerializer cookieSerializer(){
        DefaultCookieSerializer cookieSerializer = new DefaultCookieSerializer();
        cookieSerializer.setDomainName("gulimall.com");//放大整个系统域的作用域
        cookieSerializer.setCookieName("GULISESSION");
        return cookieSerializer;
    }

    /**
	     * 序列化机制
	     * @return
	     */
    @Bean
    public RedisSerializer<Object> springSessionDefaultRedisSerializer() {
        return new GenericJackson2JsonRedisSerializer();
    }
}
```

5、值得注意的是，springsession具有自动续期的功能。