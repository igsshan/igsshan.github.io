# SpringBoot 2.0核心技术

## 1.SpringBoot 详解

### 01.为什么用SpringBoot

> Spring Boot makes it easy to create stand-alone, production-grade Spring based Applications that you can "just run".
>
> 
>
> 能快速创建出生产级别的Spring应用

#### SpringBoot优点

- Create stand-alone Spring applications

  - 创建独立Spring应用

- Embed Tomcat, Jetty or Undertow directly (no need to deploy WAR files)

  - 内嵌web服务器

- Provide opinionated 'starter' dependencies to simplify your build configuration

  - 自动starter依赖，简化构建配置

- Automatically configure Spring and 3rd party libraries whenever possible

  - 自动配置Spring以及第三方功能

- Provide production-ready features such as metrics, health checks, and externalized configuration

  - 提供生产级别的监控、健康检查及外部化配置

- Absolutely no code generation and no requirement for XML configuration

  - 无代码生成、无需编写XML

  >  SpringBoot是整合Spring技术栈的一站式框架
  >
  > SpringBoot是简化Spring技术栈的快速开发脚手架


#### SpringBoot缺点

- 人称版本帝，迭代快，需要时刻关注变化
- 封装太深，内部原理复杂，不容易精通

### 02.微服务

[James Lewis and Martin Fowler (2014)](https://martinfowler.com/articles/microservices.html)  提出微服务完整概念。<https://martinfowler.com/microservices/>

> In short, the **microservice architectural style** is an approach to developing a single application as a **suite of small services**, each **running in its own process** and communicating with **lightweight** mechanisms, often an **HTTP** resource API. These services are **built around business capabilities** and **independently deployable** by fully **automated deployment** machinery. There is a **bare minimum of centralized management** of these services, which may be **written in different programming languages** and use different data storage technologies.-- [James Lewis and Martin Fowler (2014)](https://martinfowler.com/articles/microservices.html)

- 微服务是一种架构风格
- 一个应用拆分为一组小型服务

- 每个服务运行在自己的进程内，也就是可独立部署和升级
- 服务之间使用轻量级HTTP交互

- 服务围绕业务功能拆分
- 可以由全自动部署机制独立部署

- 去中心化，服务自治。服务可以使用不同的语言、不同的存储技术

### 03.分布式

> 微服务时代下,将一个大型软件,拆分成很多个单独服务部署之后,就会产生分布式

#### 分布式的困难

- 远程调用
- 服务发现

- 负载均衡
- 服务容错

- 配置管理
- 服务监控

- 链路追踪
- 日志管理

- 任务调度
- ......

#### 分布式的解决

> SpringBoot + SpringColud

### 04.云原生

原生应用如何上云,Colud Native

#### 上云的困难

- 服务自愈
- 弹性伸缩

- 服务隔离
- 自动化部署

- 灰度发布
- 流量治理

- ......

#### 上云的解决

> Docker + Kubernetes

## 2.SpringBoot 入门Demo

> SpringBoot要求Java8以上版本
>
> Maven 3.3+

```xml
<-- Maven设置 /-->
    
  <mirrors>
      <mirror>
        <id>nexus-aliyun</id>
        <mirrorOf>central</mirrorOf>
        <name>Nexus aliyun</name>
        <url>http://maven.aliyun.com/nexus/content/groups/public</url>
      </mirror>
  </mirrors>
 
  <profiles>
         <profile>
              <id>jdk-1.8</id>
              <activation>
                <activeByDefault>true</activeByDefault>
                <jdk>1.8</jdk>
              </activation>
              <properties>
                <maven.compiler.source>1.8</maven.compiler.source>
                <maven.compiler.target>1.8</maven.compiler.target>
            <maven.compiler.compilerVersion>1.8</maven.compiler.compilerVersion>
              </properties>
         </profile>
  </profiles>
```

```xml
<-- 创建SpringBoot项目 /-->
<-- 引入依赖 /-->

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.3.4.RELEASE</version>
    </parent>


    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

    </dependencies>
```

```java
// 编写主程序类
@SpringBootApplication
public class MainApplication {
    public static void main(String[] args) {
        SpringApplication.run(MainApplication.class,args);
    }
}
```

```java
// 业务请求
@RestController
public class HelloController {
    @RequestMapping("/hello")
    public String handle01(){
        return "Hello, Spring Boot 2!";
    }
}
```

```properties
server.port=8001
```

```xml
<-- 简化部署 /-->
<-- 把项目打成jar包，直接在目标服务器执行即可 /-->
 <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
```

## 3.SpringBoot 依赖管理

> 每一个SpringBoot项目,都要引入父项目做依赖管理

```xml
<-- 依赖管理 /-->  
<parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.3.4.RELEASE</version>
</parent>

<-- 他的父项目 /-->
 <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-dependencies</artifactId>
    <version>2.3.4.RELEASE</version>
  </parent>

<-- 在 spring-boot-dependencies中,几乎声明了所有开发中常用的依赖的版本号,自动版本仲裁机制 /-->
```

> 无需关注版本号,自动版本仲裁,默认依赖可以不写版本,非版本仲裁的jar,需要写版本号
>
> 需要对默认版本号进行修改,可以在当前项目里重写配置

- 开发导入starter场景启动器

```xml
1、见到很多 spring-boot-starter-* ： *就某种场景
2、只要引入starter，这个场景的所有常规需要的依赖我们都自动引入
3、SpringBoot所有支持的场景
https://docs.spring.io/spring-boot/docs/current/reference/html/using-spring-boot.html#using-boot-starter
4、见到的  *-spring-boot-starter： 第三方为我们提供的简化开发的场景启动器。
5、所有场景启动器最底层的依赖
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter</artifactId>
  <version>2.3.4.RELEASE</version>
  <scope>compile</scope>
</dependency>
```

## 4.SpringBoot 自动配置

- 配置Tomcat
  - 引入tomcat依赖
  - 配置tomcat

```xml
<dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-tomcat</artifactId>
      <version>2.3.4.RELEASE</version>
      <scope>compile</scope>
    </dependency>
```

- 自动配置SpringMVC

  - 自动引入SpringMVC全套组件

  - 自动配置SpringMVC常用组件

    ```java
    @SpringBootApplication
    public class MainApplication {
        public static void main(String[] args) {
            ConfigurableApplicationContext run = SpringApplication.run(MainApplication.class, args);
        }
    }
    
    // ConfigurableApplicationContext 类中自动配置好所有Web开发常见场景
    ```

- 默认包结构

  - 主程序所在包及其下面的所有子包里面的组件都会被默认扫描出来

  - 无需添加包扫描配置

  - 想要改变默认包扫描规则,使用 @SpringBootApplication(scanBasePackages="com.demo")

    - 或者使用 @ComponentScan 指定扫描路径

      ```java
      @SpringBootApplication
      等同于
      @SpringBootConfiguration
      @EnableAutoConfiguration
      @ComponentScan("com.demo.boot")
      ```

- 各种配置拥有默认值

- - 默认配置最终都是映射到某个类上，如：MultipartProperties
  - 配置文件的值最终会绑定每个类上，这个类会在容器中创建对象

- 按需加载所有自动配置项

- - 非常多的starter,引入了哪些场景这个场景的自动配置才会开启

- - SpringBoot所有的自动配置功能都在 spring-boot-autoconfigure 包里面

### Spring注解

#### 组件添加

@Configuration

- 基本使用
  - 