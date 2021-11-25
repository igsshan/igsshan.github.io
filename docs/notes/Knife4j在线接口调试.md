# 在线接口调试

## SpringBoot使用knife4j进行在线接口调试

> knife4j是为Java MVC框架集成Swagger生成api文档的增强解决方案,前身是swagger-bootstrap-ui,具有小巧,轻量,并且功能强悍的优点
>
> `Knife4j`提供两大核心功能：文档说明 和 在线调试

- **文档说明**：根据`Swagger`的规范说明，详细列出接口文档的说明，包括接口地址、类型、请求示例、请求参数、响应示例、响应参数、响应码等信息，使用`swagger-bootstrap-ui`能根据该文档说明，对该接口的使用情况一目了然。
- **在线调试**：提供在线接口联调的强大功能，自动解析当前接口参数,同时包含表单验证，调用参数可返回接口响应内容、`headers`、`Curl`请求命令实例、响应时间、响应状态码等信息，帮助开发者在线调试，而不必通过其他测试工具测试接口是否正确,简洁、强大。

## 整合代码

### 注入依赖

```java
        <dependency>
            <groupId>com.github.xiaoymin</groupId>
            <artifactId>knife4j-spring-boot-starter</artifactId>
            <version>2.0.4</version>
        </dependency>

        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger-ui</artifactId>
            <version>2.9.2</version>
        </dependency>
```

### 配置类

```java
@Configuration
@EnableSwagger2
@EnableKnife4j
@Import(BeanValidatorPluginsConfiguration.class)
@ConditionalOnProperty(value = {"knife4j.enable"}, matchIfMissing = true)
public class SwaggerConfiguration {
    
    @Bean
    public Docket createRestApi() {
        return new Docket(DocumentationType.SWAGGER_2)
                .apiInfo(apiInfo())
                .select()
                .apis(RequestHandlerSelectors.basePackage("com.cmiot"))
                .paths(PathSelectors.any())
                .build();
    }

    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
                .title("在线接口文档")
                .description("swagger-bootstrap-ui")
                .termsOfServiceUrl("http://localhost:8081")
                .version("1.0")
                .build();
    }
}
```

项目启动后:访问 http://{ip}:{host}/doc.html

