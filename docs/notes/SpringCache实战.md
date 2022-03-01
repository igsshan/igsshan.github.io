# SpringCache实战

## 简介

> 官方文档：
>
> https://docs.spring.io/spring-framework/docs/5.2.19.RELEASE/spring-framework-reference/integration.html#cache

- Spring 从 3.1 开始定义了 org.springframework.cache.Cache 和 org.springframework.cache.CacheManager 接口来统一不同的缓存技术； 并支持使用 JCache（JSR-107）注解简化我们开发；

- Cache 接口为缓存的组件规范定义，包含缓存的各种操作集合； 

  Cache 接 口 下 Spring 提 供 了 各 种 xxxCache 的 实 现 ； 如 RedisCache ， EhCacheCache，ConcurrentMapCache 等；

- 每次调用需要缓存功能的方法时，Spring 会检查检查指定参数的指定的目标方法是否已 

  经被调用过；如果有就直接从缓存中获取方法调用后的结果，如果没有就调用方法并缓 

  存结果后返回给用户。下次调用直接从缓存中获取。

> 使用 Spring 缓存抽象时我们需要关注以下两点
>
> - 1，确定方法需要被缓存以及他们的缓存策略
> - 2，从缓存中读取之前缓存存储的数据



## 整合使用SpringCache

> 引入依赖

```xml
		<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-cache</artifactId>
        </dependency>

<!-- 想要使用redis做为缓存,还需要引入redis依赖 -->
		<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
```

> 配置cache文件

- 缓存的自动配置

  ```java
  // CacheAutoConfiguration配置类,会根据CacheConfigurationImportSelector自动装配已经引入的redis的配置进行映射;
  // CacheAutoConfiguration会导入RedisCacheConfiguration配置类
  // RedisCacheConfiguration 会注入一个 Bean 是 RedisCacheManager;
  // RedisCacheConfiguration 会定义很多的缓存规则
  ```

- 我们需要配置哪些？

  ```properties
  spring:
    cache:
      type: redis # 指定缓存类型
      cache-names: # 指定缓存使用的名称;注意,这里指定缓存名称之后,不能再业务中实时创建,所以最好不指定名称
  ```

- 测试使用缓存

  ```java
  @Cacheable: Triggers cache population. // 触发将数据保存到缓存的操作
  
  @CacheEvict: Triggers cache eviction. // 触发将数据从缓存中删除的操作
  
  @CachePut: Updates the cache without interfering with the method execution. // 不影响方法执行更新缓存
  
  @Caching: Regroups multiple cache operations to be applied on a method. // 重新组合多个缓存操作以应用于一个方法
  
  @CacheConfig: Shares some common cache-related settings at class-level. // 在类级别共享一些常用的缓存相关配置
  ```

- **`开始缓存功能`**

  ```java
  // @EnableCaching
  
  // 使用注解开发,引用上面的注解@Cachable.....
  // 每一个需要缓存的数据,都要指定一个缓存的名称(缓存数据名称);按照业务类型分区
  ```


### @Cacheable自定义使用

> @Cacheable的默认行为

- 1.  如果缓存中存在直接返回,方法不用调用
  2.  key默认生成: 缓存的名字: : SimpleKey[] (自主生成的key值)
  3.  缓存的value值.默认使用jdk序列化机制,将序列化后的数据存到redis
  4.  默认TTL时间 -1 : 永不过期

> 我们需要自定义

- 1.  指定生成的缓存使用的key;  使用key属性指定,默认规则使用 SpEL表达式

  2.  指定缓存的数据存活时间;  使用配置文件属性, spring.cache.redis.time-to-live= 100 默认是毫秒

  3. 将数据保存为JSON格式;  需要自定义配置


> SpringCache配置原理

- **`源码解析`**

```java
// CacheAutoConfiguration 
// -> 会给系统自动转配一个系统匹配的缓存类型(如:redis):RedisCacheConfiguration 
// -> RedisCacheConfiguration 又会自动配置了:RedisCacheManager 
// -> 初始化所有的缓存
// -> 每个缓存决定使用什么配置
// -> 如果RedisCacheConfiguration有就用已有的,没有就用默认的
// -> 想改缓存的配置,只需要给容器中放一个RedisCacheConfiguration即可
// -> 就会应用到当前RedisCacheManager管理的所有缓存分区中

// 源码

class RedisCacheConfiguration {
    // 注入的RedisCacheManager
    @Bean
	RedisCacheManager cacheManager(CacheProperties cacheProperties, CacheManagerCustomizers cacheManagerCustomizers,
			ObjectProvider<org.springframework.data.redis.cache.RedisCacheConfiguration> redisCacheConfiguration,
			ObjectProvider<RedisCacheManagerBuilderCustomizer> redisCacheManagerBuilderCustomizers,
			RedisConnectionFactory redisConnectionFactory, ResourceLoader resourceLoader) {
		RedisCacheManagerBuilder builder = RedisCacheManager.builder(redisConnectionFactory).cacheDefaults(
				determineConfiguration(cacheProperties, redisCacheConfiguration, resourceLoader.getClassLoader()));
        // 获取所有的缓存
		List<String> cacheNames = cacheProperties.getCacheNames();
		if (!cacheNames.isEmpty()) {
            // 初始化所有缓存
			builder.initialCacheNames(new LinkedHashSet<>(cacheNames));
		}
		redisCacheManagerBuilderCustomizers.orderedStream().forEach((customizer) -> customizer.customize(builder));
		return cacheManagerCustomizers.customize(builder.build());
	}
    
    // 方法在被注入时调用
    private org.springframework.data.redis.cache.RedisCacheConfiguration determineConfiguration(
			CacheProperties cacheProperties,
			ObjectProvider<org.springframework.data.redis.cache.RedisCacheConfiguration> redisCacheConfiguration,
			ClassLoader classLoader) {
        // redisCacheConfiguration.getIfAvailable():如果配置中没有,就调用createConfiguration()方法
		return redisCacheConfiguration.getIfAvailable(() -> createConfiguration(cacheProperties, classLoader));
	}

	private org.springframework.data.redis.cache.RedisCacheConfiguration createConfiguration(
			CacheProperties cacheProperties, ClassLoader classLoader) {
        // 加载配置文件
		Redis redisProperties = cacheProperties.getRedis();
        // 没有手动配置,就使用系统的默认配置
		org.springframework.data.redis.cache.RedisCacheConfiguration config = org.springframework.data.redis.cache.RedisCacheConfiguration
				.defaultCacheConfig();
		config = config.serializeValuesWith(
				SerializationPair.fromSerializer(new JdkSerializationRedisSerializer(classLoader)));
        // 过期时间
		if (redisProperties.getTimeToLive() != null) {
			config = config.entryTtl(redisProperties.getTimeToLive());
		}
        // key前缀
		if (redisProperties.getKeyPrefix() != null) {
			config = config.prefixCacheNameWith(redisProperties.getKeyPrefix());
		}
        // 是否缓存null值
		if (!redisProperties.isCacheNullValues()) {
			config = config.disableCachingNullValues();
		}
        // 是否使用key前缀
		if (!redisProperties.isUseKeyPrefix()) {
			config = config.disableKeyPrefix();
		}
		return config;
	}
}
```

> 自定义配置RedisCacheConfiguration (org.springframework.data.redis.cache.RedisCacheConfiguration)

```java
@EnableConfigurationProperties(CacheProperties.class)
@EnableCaching
@Configuration
public class MyCacheRedisConfiguration {

    /**
     *
     * 原来系统的默认配置,没有加载到bean容器中
     *          @ConfigurationProperties(prefix = "spring.cache")
     *          public class CacheProperties
     * 所以我们还需要注入到配置文件管理器中
     *          @EnableConfigurationProperties(CacheProperties.class)
     * @return
     */
    @Bean
    RedisCacheConfiguration redisCacheConfiguration(CacheProperties cacheProperties) {

        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig();

        // 配置key和value的序列化方式
        config = config.serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()));
        config = config.serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericFastJsonRedisSerializer()));

        // 因为系统如果我们自己自定义配置了,就不会继续加载默认的defaultCacheConfig,所以我们还需要加载默认配置
        CacheProperties.Redis redisProperties = cacheProperties.getRedis();
        if (redisProperties.getTimeToLive() != null) {
            config = config.entryTtl(redisProperties.getTimeToLive());
        }
        if (redisProperties.getKeyPrefix() != null) {
            config = config.prefixCacheNameWith(redisProperties.getKeyPrefix());
        }
        if (!redisProperties.isCacheNullValues()) {
            config = config.disableCachingNullValues();
        }
        if (!redisProperties.isUseKeyPrefix()) {
            config = config.disableKeyPrefix();
        }

        return config;
    }
}

```

> 配置文件配置

```properties
spring:
  cache:
    type: redis # 指定使用缓存类型
    redis:
      time-to-live: 3600000  # 指定过期时间,单位/毫秒
      key-prefix: CACHE_  # 指定key前缀,如果没有指定就默认使用缓存的名称做为前缀
      use-key-prefix: true  # 是否使用key前缀,默认true
      cache-null-values: true  # 是否缓存空值,防止缓存穿透
```

### @CacheEvict使用

> 触发将数据从缓存中删除的操作

再修改了接口之后,删除缓存 -->保证数据一致性的.失效模式

```java
// 可以这样写
@Caching(evict = {
            @CacheEvict(value = {"category"}, key = "'getCategoryLevelOne'"),
            @CacheEvict(value = {"category"}, key = "'getCategoryTwoAndThree'")
})

// 也可以这样写
@CacheEvict(value = {"category"}, allEntries = true)

// 两种都可以删除.第一种按照key值组合删除;第二种直接分区删除
```

### @CachePut使用

> 不影响方法执行更新缓存

```txt
可以与@CacheEvict一起使用,在更新之后的返回值还是需要的值,达到双写模式
```

## SpringCache的不足

### 读模式

- 缓存穿透：查询一个null的数据。解决：缓存空数据：cache-null-values: true
- 缓存击穿：大量并发进来同时查询一个正好过期的数据。解决：加锁：默认是无加锁的;sync=true(手动加锁,解决击穿)
- 缓存雪崩：大量的key同时过期。解决：加随机的时间：加上过期时间，time-to-live: 3600000

### 写模式（保证缓存与数据的一致性）

- 读写加锁
- 引入Canal，感知到Mysql的更新去更新缓存
- 读多写多的情况，建议直接查询数据库

**`原理`**

```java
//CacheManage（RedisCacheManage）->Cache（RedisCache）->Cache负责缓存的读写

// 源码

public class RedisCache extends AbstractValueAdaptingCache {
    // 常规的查询方法,不会加锁
    @Override
	protected Object lookup(Object key) {
		// 查询缓存
		byte[] value = cacheWriter.get(name, createAndConvertCacheKey(key));
		// 没有缓存直接返回null
		if (value == null) {
			return null;
		}

		return deserializeCacheValue(value);
	}
    
    
    // 使用@Cacheable(sync=true)属性的,会走这个方法
    // 被synchronized标识的方法
    @Override
	@SuppressWarnings("unchecked")
	public synchronized <T> T get(Object key, Callable<T> valueLoader) {
		// 先调用查询的方法
		ValueWrapper result = get(key);
		// 有就直接返回
		if (result != null) {
			return (T) result.get();
		}
		// 没有就查询数据库
		T value = valueFromLoader(key, valueLoader);
		put(key, value);
		return value;
	}
}

// 具体详细可以debug断点,可以更清楚查看方法调用规则
```

**`总结`**

​	1、常规数据（读多写少，即时性，一致性要求不高的数据）：完全可以使用SpringCache 

​		写模式（只要缓存中的数据设置了过期时间就没太大问题）

​	2、特殊数据：需要特殊去设计