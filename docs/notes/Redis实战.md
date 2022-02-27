# Redis实战

## Redis安装

那些数据适合使用缓存

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


## Redis处理异常问题

> 产生的堆外内存溢出: OutOfDirectMomoryError

- 原因是因为SpringBoot2.0以后版本默认使用 Lettuce 作为操作 Redis 的客户端 . Lettuce  使用 netty 进行网络通信
- Lettuce  的 bug 导致 netty 堆外内存溢出, -Hmx300m: netty如果没有指定堆外内存,会默认使用 -Hmx300m作为默认空间
  - 可以通过 -Dio.netty.maxDircetMemory 进行设置
- 解决方法
  - 不能使用 -Dio.netty.maxDircetMemory 只是去调大堆外内存
  - 1）、升级 lettuce 客户端
  - 2）、切换使用 jedis 

```sh
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

















