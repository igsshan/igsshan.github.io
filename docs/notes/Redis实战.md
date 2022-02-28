# Redis实战

## Redis安装

**那些数据适合使用缓存**

- 即时性,数据一致性要求不高的
- 访问量大且更新评率不高的数据(读多,写少)

> redis安装

- 下载镜像

  ```sh
  docker pull redis
  ```

- 创建实例并启动

  ```sh
  # 创建配置文件夹
  mkdir -p /mydata/redis/conf
  
  touch /mydata/redis/conf/redis.conf
  
  docker run -p 6379:6379 --name redis -v /mydata/redis/data:/data \ 
  -v /mydata/redis/conf/redis.conf:/etc/redis/redis.conf \ 
  -d redis redis-server /etc/redis/redis.conf
  ```

- 使用 redis 镜像执行 redis-cli 命令连接

  ```sh
  docker exec -it redis redis-cli
  ```


## Redis压测处理异常问题

> 产生的堆外内存溢出: OutOfDirectMomoryError

- 原因是因为SpringBoot2.0以后版本默认使用 Lettuce 作为操作 Redis 的客户端 . Lettuce  使用 netty 进行网络通信
- Lettuce  的 bug 导致 netty 堆外内存溢出, -Hmx300m: netty如果没有指定堆外内存,会默认使用 -Hmx300m作为默认空间
  - 可以通过 -Dio.netty.maxDircetMemory 进行设置
- 解决方法
  - 不能使用 -Dio.netty.maxDircetMemory 只是去调大堆外内存
  - 1）、升级 lettuce 客户端
  - 2）、切换使用 jedis 

```xml
<!--  引入redis依赖 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
    <exclusions>
        <exclusion>
            <groupId>io.lettuce</groupId>
            <artifactId>lettuce-core</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<!--  切换jedis-->
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
</dependency>
```



## 缓存失效问题之缓存穿透

> 缓存穿透

- 缓存穿透是指查询一个一定`不存在`的数据，由于缓存是不命中，将去查询数据库，但是数 

  据库也无此记录，我们没有将这次查询的 null 写入缓存，这将导致这个不存在的数据每次 

  请求都要到存储层去查询，失去了缓存的意义。

- 在流量大时，可能 DB 就挂掉了，要是有人利用不存在的 key 频繁攻击我们的应用，这就是 

  漏洞

- **`解决方案：`**

  - **接口层增加校验**，如用户鉴权校验，id做基础校验，id<=0的直接拦截

  - 即使查询了空结果也进行缓存，并且设置较短的失效时间



## 缓存失效问题之缓存雪崩

> 缓存雪崩

- 缓存雪崩是指在我们设置缓存时采用了相同的过期时间，导致缓存在某一时刻同时失 

  效，请求全部转发到 DB，DB 瞬时压力过重雪崩。

- **`解决方案：`**

  - 原有的失效时间基础上增加一个随机值，比如 1-5 分钟随机，这样每一个缓存的过期时间的 

    重复率就会降低，就很难引发集体失效的事件。

  - `redis高可用,`如果缓存数据库是分布式部署，将热点数据均匀分布在不同得缓存数据库中。

  - 设置热点数据永远不过期。



## 缓存失效问题之缓存击穿

> 缓存击穿

- 对于一些设置了过期时间的 key，如果这些 key 可能会在某些时间点被超高并发地访问,是一种非常“热点”的数据。

- 这个时候，需要考虑一个问题：如果这个 key 在大量请求同时进来前正好失效，那么所有对这个 key 的数据查询都落到 db，我们称为缓存击穿。 

- **`解决方案：`**

  - **设置热点数据永远不过期。**
  - **接口限流与熔断，降级。**重要的接口一定要做好限流策略，防止用户恶意刷接口，同时要降级准备，当接口中的某些 服务  不可用时候，进行熔断，失败快速返回机制。
  - **布隆过滤器**。bloomfilter就类似于一个hash set，用于快速判某个元素是否存在于集合中，其典型的应用场景就是快速判断一个key是否存在于某容器，不存在就直接返回。布隆过滤器的关键就在于hash算法和容器大小

  - **加锁**；高并发情况下，给单独需要查询数据的的线程加锁，其他线程等待。



## 分布式锁

### 分布式锁与本地锁

> 本地锁,只能锁住当前进程,在分布式架构下,集群部署中,效果不好,所以需要分布式锁
>
> `本地锁：` synchronize , JUC(Lock) 均是本地锁
>
> 在分布式情况下,必须使用分布式锁

### 分布式锁实现

> 我们可以同时去一个地方”占坑“，如果占到，就执行逻辑。否则就必须等待，知道释放锁。
>
> ”占坑“可以去redis，也可以去数据库，可以去任何大家都能访问的地方。
>
> 等待可以自选的方式。

#### 场景解决

- 使用Redis

  - 获取锁，setnx("lock",1111)

  - 设置成功返回ok->获取到锁 ->执行逻辑 ->删除锁 ->结束

  - 设置失败返回null ->没获取到锁 ->等待(自旋) ->重新设置setNX("lock",1111) -> 直到获取到锁

    ```java
    	// Boolean setIfAbsent(K key, V value);
    	String uuid = UUID.randomUUID().toString();
    	Boolean lock = redisTemplate.opsForValue().setIfAbsent("lock", uuid);
            // 获取到锁
            if (lock) {
                // 执行逻辑
                // 删除锁
                redisTemplate.delete("lock");
                return result;
            }else {
                // 没有获取到锁
                // 等待100ms重试
                try{Thread.sleep(200); }catch (Exception e){ }
                return testWithRedissonLock(); //自旋的方式
            }
    ```

  - **`问题`**

    - sernx占好了位置,业务代码异常或者程序在页面过程中宕机了,没有执行删除锁的逻辑,这就造成了死锁

  - `解决`

    - 设置锁的自动过期,即使没有删除,会自动删除

      ```java
      Boolean setIfAbsent(K key, V value);
      
      // 代码实现
      String uuid = UUID.randomUUID().toString();
      Boolean lock = redisTemplate.opsForValue().setIfAbsent("lock", uuid);
      ```

  - **`问题`**

    - 直接删除锁；如果由于业务时间很长，锁自己过期了，此时直接删除锁，有可能把别人正在持有的锁删除了

  - `解决`

    - 占锁的时候，值指定为uuid，每个人匹配自己的锁才删除

      ```java
      <T> T execute(RedisScript<T> script, List<K> keys, Object... args);
      
      // 代码实现
      String script = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
                
      Long lock1 = redisTemplate.execute(new DefaultRedisScript<Long>(script, Long.class), Arrays.asList("lock"), uuid);
      ```

  ```java
  		String uuid = UUID.randomUUID().toString();
          Boolean lock = redisTemplate.opsForValue().setIfAbsent("lock", uuid, 300, TimeUnit.SECONDS);
          if (lock) {
              System.out.println("获取分布式锁成功...");
              //加锁成功... 执行业务
              // 2、设置过期时间，必须和加锁是同步的，原子的;在setnx的时候就开始设置过期时间
              // redisTemplate.expire("lock",30,TimeUnit.SECONDS);
              try {
                 // 执行业务
              } finally {
                  String script = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
                  //删除锁
                  Long lock1 = redisTemplate.execute(new DefaultRedisScript<Long>(script, Long.class), Arrays.asList("lock"), uuid);
              }
              //获取值对比+对比成功删除=原子操作 lua 脚本解锁
              // String lockValue = redisTemplate.opsForValue().get("lock");
              // if(uuid.equals(lockValue)){
              // 删除我自己的锁
              // redisTemplate.delete("lock");//删除锁
              // }
              return result;
          } else {
              //加锁失败...重试。synchronized ()
              // 休眠 100ms 重试 
              System.out.println("获取分布式锁失败...等待重试");
              try {
                  Thread.sleep(200);
              } catch (Exception e) {
              }
              return testWithRedissonLock();//自旋的方式
          }
      }
  ```

  > 这种方式实现分布式锁，重点是原子获取锁和原子删除锁的逻辑实现

### Redisson实现分布式锁

#### redisson可重入锁

```txt
	Redisson是一个在Redis的基础上实现的Java驻内存数据网格（In-Memory Data Grid）。它不仅提供了一系列的分布式的Java常用对象，还提供了许多分布式服务。其中包括(BitSet, Set, Multimap, SortedSet, Map, List, Queue, BlockingQueue, Deque, BlockingDeque, Semaphore, Lock, AtomicLong, CountDownLatch, Publish / Subscribe, Bloom filter, Remote service, Spring cache, Executor service, Live Object service, Scheduler service) Redisson提供了使用Redis的最简单和最便捷的方法。Redisson的宗旨是促进使用者对Redis的关注分离（Separation of Concern），从而让使用者能够将精力更集中地放在处理业务逻辑上。
	充分的利用了 Redis 键值数据库提供的一系列优势，基于 Java 实用工具包中常用接口，为使用者 提供了一系列具有分布式特性的常用工具类。使得原本作为协调单机多线程并发程序的工 具包获得了协调分布式多机多线程并发系统的能力，大大降低了设计和研发大规模分布式 系统的难度。同时结合各富特色的分布式服务，更进一步简化了分布式环境中程序相互之间 的协作。
```

**整合Redisson做为分布式锁的框架**

> 官方文档 : https://github.com/redisson/redisson

> 引入依赖

```xml
<!-- https://mvnrepository.com/artifact/org.redisson/redisson -->
        <dependency>
            <groupId>org.redisson</groupId>
            <artifactId>redisson</artifactId>
            <version>3.12.0</version>
        </dependency>
```

> 配置redisson

```java
@Configuration
public class MyRedissonConfig {

    @Bean(destroyMethod = "shutdown")
    public RedissonClient redisson() {
        Config config = new Config();
        config.useSingleServer().setAddress("redis://192.168.136.141:6379");
        RedissonClient redissonClient = Redisson.create(config);
        return redissonClient;
    }

}
```

> 测试是否注入

```java
	@Test
    void testRedissonClient(){
        System.out.println(redissonClient);
    }

// 结果返回
org.redisson.Redisson@106802ea //说明注入成功
    
```

> demo演示

```java
	public String hello() {
        // 获取锁
        RLock lock = redisson.getLock("my-lock");
        // 加锁
        lock.lock();// 等待锁
        try {
            // 执行业务
            System.out.println("获取锁之后执行业务" + Thread.currentThread().getId());
            Thread.sleep(3000);
        } catch (Exception e) {
            // 获取异常
        } finally {
            // 释放锁
            System.out.println("释放锁" + Thread.currentThread().getId());
            lock.unlock();
        }
        return "hello redisson";
    }
```

- 模拟两台服务器集群,两个线程同时发请求,a服务先获取到锁之后,停止a服务器,人为制造a服务器不解锁的情况,b服务器一样也能完成请求
  - redisson 解决了两个事情
    - 1 . 锁的自动续期,如果业务时间超长,运行期间自动给锁续上新的30s(默认加的锁都是30s),不用担心业务时间长,锁自动过期被删掉的问题
    - 2 . 加锁的业务只要运行完成,就不会给当前锁续期,即使不手动解锁,锁默认在30s以后会自动删除



#### lock看门狗机制-redisson解决死锁

```java
// 加锁以后10秒钟自动解锁
// 无需调用unlock方法手动解锁
lock.lock(10, TimeUnit.SECONDS);
```

```java
// 最常见的使用方法
lock.lock();
```

- 两种加锁的方法,对比区别

  ```java
  lock.lock(10, TimeUnit.SECONDS);// 10自动解锁,自动解锁的时间一定要大于业务执行的时间；因为lock.lock(10, TimeUnit.SECONDS)在设置了10自动到期之后，不会自动续期
  
  //1、【lock.lock(10, TimeUnit.SECONDS)】如果我们传递了锁的超时时间，就发送给redis执行脚本，运行占锁，默认超时就是我们指定的时间
  //2、【lock.lock()】如果我们未指定锁的超时时间，就使用30*1000【LockWatchdogTimeout看门狗默认时间30s】
  //    只要占锁成功，就会启动一个定时任务【重新给锁设置过期时间，新的过期时间就是看门狗的默认时间(30s)】，每隔10s【internalLockLeaseTime【看门狗时间】/3】就会续期，续满看门狗的时间【30s】
     
  ```

- 最佳实战

  ```java
  1、lock.lock(30, TimeUnit.SECONDS);// 省掉了整个续期操作,业务完成后手动解锁
  ```


#### redisson读写锁

> 基于Redis的Redisson分布式可重入读写锁[`RReadWriteLock`](http://static.javadoc.io/org.redisson/redisson/3.4.3/org/redisson/api/RReadWriteLock.html) Java对象实现了`java.util.concurrent.locks.ReadWriteLock`接口。其中读锁和写锁都继承了[RLock](https://github.com/redisson/redisson/wiki/8.-%E5%88%86%E5%B8%83%E5%BC%8F%E9%94%81%E5%92%8C%E5%90%8C%E6%AD%A5%E5%99%A8#81-%E5%8F%AF%E9%87%8D%E5%85%A5%E9%94%81reentrant-lock)接口。
>
> 分布式可重入读写锁允许同时有多个读锁和一个写锁处于加锁状态。

```java
RReadWriteLock rwlock = redisson.getReadWriteLock("anyRWLock");
// 最常见的使用方法
rwlock.readLock().lock();
// 或
rwlock.writeLock().lock();
```

> 另外Redisson还通过加锁的方法提供了`leaseTime`的参数来指定加锁的时间。超过这个时间后锁便自动解开了。

```java
// 10秒钟以后自动解锁
// 无需调用unlock方法手动解锁
rwlock.readLock().lock(10, TimeUnit.SECONDS);
// 或
rwlock.writeLock().lock(10, TimeUnit.SECONDS);

// 尝试加锁，最多等待100秒，上锁以后10秒自动解锁
boolean res = rwlock.readLock().tryLock(100, 10, TimeUnit.SECONDS);
// 或
boolean res = rwlock.writeLock().tryLock(100, 10, TimeUnit.SECONDS);
...
lock.unlock();
```

- **`读写锁功能作用`**
  - 保证一定能读到最新的数据
    - 在修改期间。`写锁`是一个排他锁（互斥锁）；`读锁`是一个共享锁。
    - `写锁`没释放`读锁`就必须等待

> 补充细节

- 读锁 + 读锁：相当于无锁，并发读，只会在redis中记录好所有当前的读锁，他们都会自动加锁成功

- 写锁 + 读锁：等待写锁释放

- 写锁 + 写锁：阻塞方式

- 读锁 + 写锁： 

  ```java
  // 写一个demo,伪造在读的时候,也sleep30s,同步在写入一条数据
  ```

  有读锁，写锁也需要等待。

> 总结:
>
> ​	只要是写锁的存在，都必须等待写锁释放。

###  缓存数据一致性问题

#### 双写模式

#### 失效模式

#### 改进方法 1-分布式读写锁

> 分布式读写锁。读数据等待写数据整个操作完成

#### 改进方法 2-使用 cananl

