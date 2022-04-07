# 分布式session解决方案

> session共享问题-分布式下session共享问题

- 同一服务,复制多份,session不同步问题
- 不同服务,session不能共享问题

### 方案一

#### session复制

- 优点: web-sever(tomcat) 原生支持,只需要修改配置文件
- 缺点
  - session同步需要数据传输,占用大量网络宽带,降低了服务器群的业务处理能力
  - 任意一台web-server 保存的数据都是所有web-server 的session总和,受到内存限制无法水平扩展更多的web-server
  - 大型分布式集群情况下,由于所有web-server都全量保存数据,所以此方案不可取

### 方案二

#### 客户端存储

- 优点: 服务器不需存储session,用户保存自己的session信息到cookie中,节省服务端资源
- 缺点
  - 每次http请求,携带用户在cookie中的完整信息,浪费网络宽度
  - session数据放在cookie中,cookie有长度限制,不能保存大量信息
  - session数据保存在cookie中,存在泄漏,篡改,窃取等安全隐患
- 所有说,这种只是一种思路,不会使用

### 方案三

#### hash一致性

- 优点
  - 只需要修改nginx配置,不需要修改应用代码
  - 负载均衡,只要hash属性的值分布是均匀的,多台web-server的负载是均衡的
  - 可以支持web-server水平扩展(session同步法是不行的.受内存限制)
- 缺点
  - session还是存在web-server中的,所以web-server重启可能会导致部分session丢失,影响业务,如部分用户需要重新登录
  - 如果web-server水平扩展,rehash后session重新分布,也会有一部分用户路由不到正确的session
- 但是以上缺点问题也不是很大,因为session本身就是有时效的,也会过期

### 方案四

#### 统一存储

> 将用户的所有session信息,后端统一存储在db/redis中

- 优点
  - 没有安全隐患
  - 可以水平扩展,数据库/缓存水平切分即可
  - web-server重启或者扩容都不会有session丢失
- 缺点
  - 增加了一次网络调用,并且需要修改应用代码,如将所有的getSession方法替换为从redis中查询的方式,redis获取数据比内存慢很多
  - 上面的缺点可以使用SpringSession框架解决



## SpringBoot整合SpringSession

> 引入依赖

```pom

<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>
```

> 启用类

```java
// 使用注解开始session
@EnableRedisHttpSession
public class MyApplication {

    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }

}
```

> 配置文件

```yml
## 使用redis作为session存储
spring:
	session:
		store-type: redis
```

> 编写配置类

```java
@Configuration
public class MySessionConfig {

    /**
     *cookie配置类
     */
    @Bean
    public CookieSerializer cookieSerializer() {
        DefaultCookieSerializer cookieSerializer = new DefaultCookieSerializer();
        // 放大整个系统域的作用域
        cookieSerializer.setDomainName("mall.com");
        cookieSerializer.setCookieName("MALL_SESSION");
        
        // 设置cookie时长
        cookieSerializer.setCookieMaxAge();
        return cookieSerializer;
    }

    /**
     * 序列化机制
     *
     * @return
     */
    @Bean
    public RedisSerializer<Object> springSessionDefaultRedisSerializer() {
        return new GenericJackson2JsonRedisSerializer();
    }
}
```

## SpringSession原理

