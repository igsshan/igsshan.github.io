# SpringBoot整合RabbitMQ

> 官网地址 : https://www.rabbitmq.com/

## rabbitmq安装

> docker 安装

- https://hub.docker.com/_/rabbitmq
- docker pull rabbitmq:management

> docker 启动命令

```sh
## 启动rabbitMQ
docker run -d --name rabbitmq -p 5671:5671 -p 5672:5672 -p 4369:4369 -p 25672:25672 -p 15671:15671 -p 15672:15672 rabbitmq:management

## 自启动
docker update rabbitmq --restart=always

#### 防火墙开启,注意端口放行
## 查询放行的端口
firewall-cmd --list-ports
## 添加放行端口
firewall-cmd --zone=public --permanent --add-port=5671/tcp
## 刷新端口
firewall-cmd --reload
```

> 4369, 25672 (Erlang发现&集群端口) 
>
> 5672, 5671 (AMQP端口) 
>
> 15672 (web管理后台端口) 
>
> 61613, 61614 (STOMP协议端口) 
>
> 1883, 8883 (MQTT协议端口) 
>
> 官方协议文档: https://www.rabbitmq.com/networking.html
>
> http访问: http://192.168.56.101:15672

## springBoot整合rabbitmq

> 引入 spring-boot-starter-amqp

```pom
		<!-- 引入 RabbitMQ 依赖-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-amqp</artifactId>
        </dependency>
```

- 引入 rabbitmq 场景启动器, RabbitAutoConfiguration 配置类会自动生效

  - 它给容器中配置了一系列的组件

    ```java
    		/**
    		* 连接工厂
    		*/
    		@Bean
    		public CachingConnectionFactory rabbitConnectionFactory(RabbitProperties properties,
    				ObjectProvider<ConnectionNameStrategy> connectionNameStrategy) throws Exception {
    			PropertyMapper map = PropertyMapper.get();
    			CachingConnectionFactory factory = new CachingConnectionFactory(
    					getRabbitConnectionFactoryBean(properties).getObject());
    			map.from(properties::determineAddresses).to(factory::setAddresses);
    			map.from(properties::isPublisherReturns).to(factory::setPublisherReturns);
    			map.from(properties::getPublisherConfirmType).whenNonNull().to(factory::setPublisherConfirmType);
    			RabbitProperties.Cache.Channel channel = properties.getCache().getChannel();
    			map.from(channel::getSize).whenNonNull().to(factory::setChannelCacheSize);
    			map.from(channel::getCheckoutTimeout).whenNonNull().as(Duration::toMillis)
    					.to(factory::setChannelCheckoutTimeout);
    			RabbitProperties.Cache.Connection connection = properties.getCache().getConnection();
    			map.from(connection::getMode).whenNonNull().to(factory::setCacheMode);
    			map.from(connection::getSize).whenNonNull().to(factory::setConnectionCacheSize);
    			map.from(connectionNameStrategy::getIfUnique).whenNonNull().to(factory::setConnectionNameStrategy);
    			return factory;
    		}
    
    
    		/**
    		* RabbitTemplate 消息发送组件
    		*/
    		@Bean
    		@ConditionalOnSingleCandidate(ConnectionFactory.class)
    		@ConditionalOnMissingBean(RabbitOperations.class)
    		public RabbitTemplate rabbitTemplate(RabbitTemplateConfigurer configurer, ConnectionFactory connectionFactory) {
    			RabbitTemplate template = new RabbitTemplate();
    			configurer.configure(template, connectionFactory);
    			return template;
    		}
     		
    
    		/**
    		* amqpAdmin 高级消息队列管理组件
    		* 使用 amqpAdmin 创建 exchanges, queue, binding
    		*/
    		@Bean
    		@ConditionalOnSingleCandidate(ConnectionFactory.class)
    		@ConditionalOnProperty(prefix = "spring.rabbitmq", name = "dynamic", matchIfMissing = true)
    		@ConditionalOnMissingBean
    		public AmqpAdmin amqpAdmin(ConnectionFactory connectionFactory) {
    			return new RabbitAdmin(connectionFactory);
    		}
    
    
    		/**
    		* RabbitMessagingTemplate
    		*/
    		@Bean
    		@ConditionalOnSingleCandidate(RabbitTemplate.class)
    		public RabbitMessagingTemplate rabbitMessagingTemplate(RabbitTemplate rabbitTemplate) {
    			return new RabbitMessagingTemplate(rabbitTemplate);
    		}
    		.....
    ```

> 开启 RabbitMQ : 如果只是创建组件,发送消息可以不开启;但是如果需要接收消息,必须开启

```java
/**
* @EnableXXX 开启某项服务
*/
@EnableRabbit
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

}
```

> 配置 配置属性

```yml
spring:
  rabbitmq:
    host: 192.168.56.101 ## 主机地址
    port: 5672 	## 端口
    virtual-host: / ## 虚拟主机
```

> Demo 测试使用 mq 创建queue , exchanges , binding

```java
public class Test {
	@Autowired
    AmqpAdmin amqpAdmin;

    @Test
    void createExchanges() {

        DirectExchange exchange = new DirectExchange("hello-java-exchanges", true, false);
        amqpAdmin.declareExchange(exchange);
        log.info("-----------创建[{}]交换机成功-------", "hello-java-exchanges");
    }

    @Test
    void createQueue() {

        Queue queue = new Queue("hello-java-queue", true, false, false);
        amqpAdmin.declareQueue(queue);
        log.info("-----------创建[{}]队列 成功-------", "hello-java-queue");
    }

    @Test
    void createBinding() {

        // String destination, 目的地
        // DestinationType destinationType, 目的地类型
        // String exchange, 交换机
        // String routingKey, 路由键
        // @Nullable Map<String, Object> arguments 参数
        Binding binding = new Binding("hello-java-queue", Binding.DestinationType.QUEUE,"hello-java-exchanges","hello",null);
        amqpAdmin.declareBinding(binding);
        log.info("-----------为[{}]队列 绑定交换机成功-------", "hello-java-queue");
    }
}
```

> Demo 测试使用 mq 发送消息

```java
// 可以直接发送一条字符串作为消息
// 也可以发送一个对象,发送对象时,要指定序列化规则

// 容易会默认注入一个消息转换器 (MessageConverter),我们要使用最好自己在指定自己的消息转换器

/**
 * 配置消息转换规则-->使用json格式
 */
@Configuration
public class MyRabbitConfig {
    
    @Bean
    public MessageConverter messageConverter () {
        return new Jackson2JsonMessageConverter ();
    }
}

public class Test {
    
    @Autowired
    RabbitTemplate rabbitTemplate ;
    
    @Test
    void sendMessage() {

        Student student = new Student("张三", 18);
        rabbitTemplate.convertAndSend("hello-java-exchanges", "hello", student);
        log.info("-----------给[{}]交换机发送消息成功-------", "hello-java-exchanges");
    }
}
```

> Mq 接收消息

- ```java
  @RabbitListener(queues = {"hello-java-queue"})
  // 标注在 类 + 方法上(监听多个队列)
  ```

- ```java
  @RabbitHandler
  // 标注在 方法上(重载区分不同的消息)
  ```

- ```java
  @RabbitListener(queues = {"hello-java-queue"})
  public class RabbitService {
  
      @RabbitHandler
      public void message1(MessageEntity1 entity){
          log.info("-----------接收到 MessageEntity1 类型的消息 ----------- {}",entity);
      }
  
      @RabbitHandler
      public void message2(MessageEntity2 entity){
          log.info("===========接收到 MessageEntity2 类型的消息 =========== {}",entity);
      }
  }
  ```

## rabbitMQ消息确认机制--可靠抵达

- 保证消息不丢失,可靠抵达,可以使用事务消息,但是性能下降250倍,为此引入确认机制
- **publisher** confirmCallback  消息确认模式
- **publisher** returnCallback 未投递到 queue 退回模式
- **consumer** ack模式

> 可靠抵达配置开启

```properties
## 开始发送端确认
spring.rabbitmq.publisher-confirms=true  ## 可以不使用,只搭配使用下面两个配置

## 开启发送端消息未抵达队列确认
spring.rabbitmq.publisher-returns=true

## 只要抵达队列,以异步发送优先回调 return-confirm
spring.rabbitmq.template.mandatory=true
```

> demo实例

```java
@Configuration
public class MyRabbitConfig {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    @Bean
    public MessageConverter messageConverter() {
        return new Jackson2JsonMessageConverter();
    }


    /**
     * 被@PostConstruct修饰的方法会在服务器加载Servlet的时候运行，并且只会被服务器执行一次。PostConstruct在构造函数之后执行，init（）方法之前执行。
     * <p>
     * 通常我们会是在Spring框架中使用到@PostConstruct注解 该注解的方法在整个Bean初始化中的执行顺序：
     * <p>
     * Constructor(构造方法) -> @Autowired(依赖注入) -> @PostConstruct(注释的方法)
     * <p>
     * 应用：在静态方法中调用依赖注入的Bean中的方法。
     */
    @PostConstruct
    public void initRabbitTemplate() {
        /*
          消息发送确认回调
         */
        rabbitTemplate.setConfirmCallback(new RabbitTemplate.ConfirmCallback() {
            /**
             *
             * @param correlationData 当前消息的唯一关联数据，每一个消息发送的时候可以给一个唯一id
             * @param ack     消息是否成功收到
             * @param cause   失败原因
             */
            @Override
            public void confirm(CorrelationData correlationData, boolean ack, String cause) {
                System.out.println("--------消息发送成功------- correlationData=>[" + correlationData + "]==>ack=>[" + ack + "]==>cause=>[" + cause + "]");
            }
        });

        /*
          消息抵达回调
         */
        rabbitTemplate.setReturnCallback(new RabbitTemplate.ReturnCallback() {
            /**
             *
             * @param message 哪个消息投递失败了，消息的具体内容
             * @param replyCode 回复的状态码
             * @param replyText 回复的文本内容
             * @param exchange 前这个消息发给哪个交换机
             * @param routingKey 当前这个消息发送的时候，指定的哪个路由键
             */
            @Override
            public void returnedMessage(Message message, int replyCode, String replyText, String exchange, String routingKey) {
                //只要服务器收到消息，发送错误了就相当于修改数据库当前消息的错误状态，改为错误了，没有收到消息，就定期重新发送消息
                System.out.println("Fail Message[" + message + "]==>replyCode[" + replyCode + "]==>replyText[" + replyText + "]==>exchange[" + exchange + "]==>routingKey[" + routingKey + "]");
            }
        });
    }
}
```

### 可靠抵达-Ack消息确认机制

- 消费者获取到消息,成功处理,可以回复ack给Broker.此时才可以删除这条消息

  ```properties
  ## 配置签收规则(默认 auto 自动签收)
  spring.rabbitmq.listener.simple.acknowledge-mode=manual   ## 手动签收
  ```

  - 系统默认是自动确认的,只要消息接收到,客户端会自动确认,服务端就会移除这个消息
  - 消费者设置为手动确认模式,只要我们没有明确告诉MQ签收消息,就不会ack,这样消息一直都是unacked状态,即使consumer宕机,消息也不会丢失,会重新变为ready,下一次有新的consumer连接进来就会发给他

  -basic.ack 用于肯定确认;broker将移除消息

  -basic.nack 用于否定确认;可以指定broker是否丢弃此消息,支持批量操作

  -basic.reject 用于否定确认;同上,但是不能批量

- 消费者收到消息,默认会自动ack,但是无法确定此消息是否被处理完成,因此最好开启手动ack模式

  - 消息处理成功,ack(),接收下一个消息,此消息就会被broker移除掉
  - 消息处理失败,nack()/reject(),重新会重新发给其他人进行处理,或者容错处理后ack
  - 消息一直没有调用ack()/nack();broker会认为消息正在被处理,不会投递给别人,此时客户端断开,消息不会被broker移除,但是会投递给别人

> demo演示

```java
	@RabbitHandler
    public void message1(Message message, Entity entity, Channel channel) {
       // log.info("-----------接收到 entity 类型的消息 ----------- {}", entity);
        long deliveryTag = message.getMessageProperties().getDeliveryTag();
        try {
            if (deliveryTag % 2 == 0) {
                /**
                 * @param deliveryTag id channel内按顺序自增的。
                 * @param multiple 是否批量 true=是/false=否
                 */
                channel.basicAck(deliveryTag, false);
                System.out.println("---------签收了消息-------" + deliveryTag);
            } else {
                /**
                 * @param deliveryTag id channel内按顺序自增的。
                 * @param multiple 是否批量 true=是/false=否
                 * @param requeue 是否回收 truer=是(重新入队)/false=否(直接丢弃)
                 */
                channel.basicNack(deliveryTag, false, true);
                System.out.println("=========丢弃了消息=======" + deliveryTag);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```

## RabbitMQ延时队列

### 消息的TTL (time to live)

- 消息的TTL 就是消息的存活时间

- RabbitMQ 可以对队列和消息分别设置 TTL

  - 对队列设置就是队列没有消费者连着的保留时间,也可以对每一个单独的消息做单独设置,超过了这个时间,我们就认为这个消息死了,称之为`死信`
  - 如果队列设置了 TTL,消息也设置了,那么会`取小的`.所以一个消息如果被路由到不同的队列中,这个消息死亡的时间有可能不一样(不同的队列设置).这里单讲单个消息的TTL,因为他才是实现延时任务的关键.可以通过`设置消息的expiration字段或者x-message-ttl属性来设置时间`两者都是一样的效果


### 死信 Dead Letter Exchanges (DLX)

- 一个消息在满足如下条件下,会进`死信路由`,这里是`路由`而不是`队列`,一个路由可以对应很多队列
  - 一个消息被consumer拒收了,并且reject方法的参数里requeue是false,也就是说不会再次放进队列里,被其他消费者使用.(basic.reject / basic.nack) requeue = false
  - 上面的消息的TTL 到期了,消息过期了
  - 队列的长度限制满了,排在前面的消息会被丢弃或者扔到死信路由上
- Dead Letter Exchange 其实就是一种普通exchange,和创建其他exchange没有两样.只是在某一个设置 dead letter exchange队列中有消息过期了,会自动触发消息的转发,发送到dead letter exchange中去
- 我们既可以控制消息在一段时间后变成死信,又可以控制变成死信的消息被路由到某一个指定的交换机,结合二者,其实就可以实现一个延时队列

### 延时队列的实现-1

> 设置队列过期时间实现延时队列

- 生产者 --- > (routing - key deal message) --> 交换机 (exchange) -- >发送到队列
- 队列设置了死信路由交换机,设置了过期时间,路由的键 (x-dead-letter-exchange delay exchange  ,x-dead-letter-routing-key  delay message , x-message-ttl  60000) ---> 到期的消息被转发到死信交换机---->(delay exchange) --->再根据路由键发送到死信队列 ---( routing -key  delay message) ---> test queue ---> consumer 消费

### 延时队列的实现-2

> 设置消息过期时间实现延时队列 ---->不太推荐,由于消息队列是懒加载的.不太适合,推荐方式1



### 延时队列的实现 demo

> 创建队列

```java
// 项目启动队列创建好了之后,即使属性发生了变化,queue也不会改变,只有重新删除在重新启动就可以

@Configuration
public class MyRabbitConfig {

    /**
     * 使用 Bean注入的方式 创建 Queue , Exchange ,Banding
     * 没有就会创建
     */
    @Bean
    public Queue orderDelayQueue() {

        // String name, boolean durable, boolean exclusive, boolean autoDelete,@Nullable Map<String, Object> arguments

        Map<String, Object> arguments = new HashMap<>(0);
        // 死信路由
        arguments.put("x-dead-letter-exchange", "order-event-exchange");
        // 路由键
        arguments.put("x-dead-letter-routing-key", "order.release.order");
        // 过期时间(毫秒)
        arguments.put("x-message-ttl", 120000);

        return new Queue("order.delay.queue", true, false, false, arguments);
    }

    @Bean
    public Queue orderReleaseOrderQueue() {

        return new Queue("order.release.order.queue", true, false, false);
    }


    @Bean
    public Exchange orderEventExchange() {

        return new TopicExchange("order-event-exchange", true, false);
    }

    @Bean
    public Binding orderCreateOrderBinding() {

        /**
         *                  目的地               目的地的类型                     哪个交换机跟目的地绑定  路由键            是否有其他参数属性
         *  public Binding(String destination, DestinationType destinationType, String exchange, String routingKey, @Nullable Map<String, Object> arguments) {
         */
        return new Binding("order.delay.queue", Binding.DestinationType.QUEUE, "order-event-exchange", "order.create.order", null);
    }

    @Bean
    public Binding orderReleaseOrderBinding() {

        return new Binding("order.release.order.queue", Binding.DestinationType.QUEUE, "order-event-exchange", "order.release.order", null);
    }
}
```

> 消费者监听队列,接收消息

```java
@slf4j
@Service
public class Test{
    
    @RabbitListener(queues = {"order.release.order.queue"})
    public void receiveMessage(Entity entity, Channel channel, Message message) throws IOException {
        log.info("---------- 接收到Entity的消息------------ {}", JSON.toJSONString(entity));
        channel.basicAck(message.getMessageProperties().getDeliveryTag(),false);
    }
}
```

> 测试发送消息

```java
@Test
void testSendMessage() {
        Entity entity = new Entity();
        entity.setId(UUID.randomUUID().toString().replace("-", ""));
        entity.setCreateTime(new Date());
        log.info("---------- 发送的消息内容是------------ {}", JSON.toJSONString(entity));
        rabbitTemplate.convertAndSend("test-event-exchange", "test.create.message", entity);
    }
```

## 消息的积压,重复,丢失等解决方案

### 消息丢失

- 消息发送出去,由于网络问题,没有抵达服务器

  - 做好容错方法(try-catch),发送消息可能会网络失败,失败后要有重试机制,可记录到数据库,采用定时扫描失败消息重新发送
  - 做好日志记录,每个消息的状态是否被服务器收到都要做好记录
  - 做好定期重发,如果消息没有发送成功,定期去数据库查询发送失败的消息进行重发

  > ```sql
  > DROP TABLE IF EXISTS `mq_message`;
  > CREATE TABLE `mq_message`  (
  >   `message_id` char(32) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL COMMENT '消息id',
  >   `content` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL COMMENT '消息的内容',
  >   `to_exchange` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL COMMENT '交换机',
  >   `routing_key` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL COMMENT '路由键',
  >   `class_type` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL COMMENT '数据类型',
  >   `message_status` int(2) NOT NULL COMMENT '0-新建/1-已发送/2-错误抵达/3-已抵达',
  >   `create_time` datetime(0) NULL DEFAULT NULL,
  >   `update_time` datetime(0) NULL DEFAULT NULL,
  >   PRIMARY KEY (`message_id`) USING BTREE
  > ) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic;
  > 
  > SET FOREIGN_KEY_CHECKS = 1;
  > ```

- 消息抵达Broker,Broker要将消息写入磁盘(持久化)才算成功,此时Broker尚未完成持久化,宕机

  - publisher也必须加入确认回调机制,确认消息成功,修改数据库状态

  ```java
  @Configuration
  public class MyRabbirConfig{
      /**
       * 被@PostConstruct修饰的方法会在服务器加载Servlet的时候运行，并且只会被服务器执行一次。PostConstruct在构造函数之后执行，init（）方法之前执行。
       * <p>
       * 通常我们会是在Spring框架中使用到@PostConstruct注解 该注解的方法在整个Bean初始化中的执行顺序：
       * <p>
       * Constructor(构造方法) -> @Autowired(依赖注入) -> @PostConstruct(注释的方法)
       * <p>
       * 应用：在静态方法中调用依赖注入的Bean中的方法。
       */
      @PostConstruct
      public void initRabbitTemplate() {
          /*
            消息发送确认回调
           */
          rabbitTemplate.setConfirmCallback(new RabbitTemplate.ConfirmCallback() {
              /**
               *
               * @param correlationData 当前消息的唯一关联数据，每一个消息发送的时候可以给一个唯一id
               * @param ack     消息是否成功收到
               * @param cause   失败原因
               */
              @Override
              public void confirm(CorrelationData correlationData, boolean ack, String cause) {
                  // TODO 服务器收到了,记录到数据库,保存好状态
                  System.out.println("--------消息发送成功------- correlationData=>[" + correlationData + "]==>ack=>[" + ack + "]==>cause=>[" + cause + "]");
              }
          });
  
          /*
            消息抵达回调
           */
          rabbitTemplate.setReturnCallback(new RabbitTemplate.ReturnCallback() {
              /**
               *
               * @param message 哪个消息投递失败了，消息的具体内容
               * @param replyCode 回复的状态码
               * @param replyText 回复的文本内容
               * @param exchange 前这个消息发给哪个交换机
               * @param routingKey 当前这个消息发送的时候，指定的哪个路由键
               */
              @Override
              public void returnedMessage(Message message, int replyCode, String replyText, String exchange, String routingKey) {
                  // TODO 报错误了,修改数据库当前消息的状态-->错误.在定期扫描失败消息,重新发送
                  System.out.println("Fail Message[" + message + "]==>replyCode[" + replyCode + "]==>replyText[" + replyText + "]==>exchange[" + exchange + "]==>routingKey[" + routingKey + "]");
              }
          });
      }
  }
  ```

- 自动ACK的状态下,消费者收到消息,还没来得及消费消息就宕机

  - 一定要开启手动ACK,消费成功才移除,失败或者没来得及处理的消息就noACK,并重新入队

  ```properties
  spring:
  	rabbitmq:
          listener:
            simple:
              acknowledge-mode: manual  ## 手动确认消息
  ```

  ```java
  @Slf4j
  @Service
  @RabbitListener(queues = {"order.release.order.queue"})
  public class StockClosureListener {
      
      @RabbitHandler
      public void receiveMessage(OrderEntity orderEntity, Channel channel, Message message) throws IOException {
          
          // 手动消费消息
          channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
  
          // 消息重新入队
           channel.basicReject(message.getMessageProperties().getDeliveryTag(), true);
      }
  }
  ```

### 消息重复

- 消息消费成功,事务已经提交,ack时候,机器宕机,导致没有ack成功,broker消息重新由unACK变成ready,并发送给其他消费者
- 消息消费失败,由于重试机制,自动又将消息发送出去
- 成功消费，ack时宕机，消息由unack变为ready，Broker又重新发送

解决方法:

- 消费者的业务消费接口应该设计成幂等性的.比如扣减库存有工作单的状态标志
- 使用防重表(redis/mysql),发送消息每一个都有业务的唯一标志,处理过的就不用处理
- rabbitMQ的每一个消息都有redelivered字段,可以获取是否是重新投递过来的,而不是第一次过来的消息,做一下处理

### 消息积压

- 消费者宕机消息积压
- 消费者消费能力不足积压
- 发送者发送流量太大积压

解决方法:

- 上线更多的消费者,进行正常消费
- 上线专门的队列消费服务,将消息先批量取出来,记录到数据库,在慢慢进行处理

