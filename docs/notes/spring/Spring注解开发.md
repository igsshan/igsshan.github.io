# Spring 注解开发

#### @Bean注解

> @Bean注解用于告诉方法，产生一个Bean对象，然后这个Bean对象交给Spring管理。
>
> 也可以在@Bean(value="name"),指定一个别名

#### @ComponentScan注解

> 包扫描,只要标注了@controller,@service,@repository,@component等注解会自动注册到容器中
>
> 使用 @ComponentScans 来添加多个 @ComponentScan，从而实现添加多个扫描规则,使用@ComponentScans需要加上@configuration注解,否则无效
>
>  @ComponentScan 的有一个 useDefaultFilters 属性，该属性默认值为 true，也就是说 spring 默认会自动发现被 @Component、@Repository、@Service 和 @Controller 标注的类，并注册进容器中,要达到只包含某些包的扫描效果,必须将这个行为给禁止(在 @ComponentScan 中将 useDefaultFilters 设为 false )

```java
@ComponentScan(value = "com.demo", 
        includeFilters = {@Filter(type = FilterType.ANNOTATION, classes = {Controller.class})},
        useDefaultFilters = false) // 扫描策略只包含带有@controller的注解类
public class BeanConfig {

}
```

- 使用 CUSTOM 类型，就可以实现自定义过滤规则

> 首先创建一个自定义类去实现TypeFilter接口,并重写其match方法

```java
/**
 * 实现对扫描到的类名进行判断,如果类包含"Co"的就符合条件,
 * 就会注入到容器中
 */
public class CustomTypeFilter implements TypeFilter {
    @Override
    public boolean match(MetadataReader metadataReader,
                         MetadataReaderFactory metadataReaderFactory) throws IOException {

        // 获取当前扫描到的类的注解元数据
        AnnotationMetadata annotationMetadata = metadataReader.getAnnotationMetadata();
        // 获取当前扫描到的类的元数据
        ClassMetadata classMetadata = metadataReader.getClassMetadata();
        // 获取当前扫描到的类的资源信息
        Resource resource = metadataReader.getResource();

        if (classMetadata.getClassName().contains("Co")) {
            return true;
        }
        return false;
    }
}


// 配置类
@ComponentScan(value = "com.demo",
        includeFilters = {@ComponentScan.Filter(type = FilterType.CUSTOM, value = {CustomTypeFilter.class})},
        useDefaultFilters = false)
public class BeanConfig {

}
```

#### @Scope

> @Scope注解是springIoc容器中的一个作用域

- 主要取值有:

  - singleton: 单实例的(默认值);ioc容器启动会调用方法创建对象放到ioc容器中,以后每次获取对象就是直接从ioc容器中获取

  - prototype:多实例的;ioc容器启动时并不会调用方法创建对象放入容器中,每次获取的时候才会调用方法创建对象
  - request: 同一次请求创建一个实例
  - session:同一次session创建一个实例

当然,singleton虽然保证了全局是一个实例,对性能有所提高,但是如果实例中有非静态实例变量时,会导致线程安全问题,共享资源的竞争; 当设置为prototype时,每次连接请求,都会生成一个bean实例,也会导致一个问题,就是请求数越多,性能会降低,频繁创建实例,导致GC频繁,GC时长增加

#### @Lazy

> 单实例bean,默认在容器启动的时候创建对象
>
> 如果想在容器启动不创建对象,第一次调用bean时创建对象并进行初始化,使用@Lazy注解(懒加载)

```xml
@Lazy 的value 取值有 true 和 false 两个 默认值为 true
true 表示使用 延迟加载， false 表示不使用，false 纯属多余，如果不使用，不标注该注解就可以了。
```

#### @Conditional

> @Conditional 是Spring4新提供的注解
>
> 按照一定的条件进行判断,满足条件给容器中注册bean

```java
package org.springframework.context.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Conditional {

	Class<? extends Condition>[] value();

}
```

- 属性是一个Class数组,并且需要继承Condition接口

```java
package org.springframework.context.annotation;

@FunctionalInterface
public interface Condition {
	/**
	 * ConditionContext: 能使用的上下文环境
	 * AnnotatedTypeMetadata 注释信息
	 */
	boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata);

}
```

- Condition是个接口，需要实现matches方法，返回true则注入bean，false则不注入。

> @Conditional注解
>
> 标注在方法上,只要满足该条件的bean就会生效
>
> 标注在类上,满足当前条件,这个类中配置的所有bean才会生效
>
> 多条件配置,当所有条件成立,返回true,被该注解标注的方法或者类才会生效

#### @Import

> @Import注解是用来导入配置类或者一些需要前置加载的类.

- @Import 导入配置的三种方式

  - 带有@Configuration注解的配置类,直接导入类

    ```java
    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    public @interface Import {
    
    	Class<?>[] value();
    
    }
    ```

    > 属性值是一个数组,可以一次导入多个类
    >
    > 普通的类，可以被@Autowired注释然后调用，就直接说明已经被Spring 注入并管理了，普通的类都是需要先实例化

  - ImportSeletor的实现

    ```java
    public interface ImportSelector {
    
        /**
         * AnnotationMetadata :当前标注@Import注解的类的所有注释信息
         */
    	String[] selectImports(AnnotationMetadata importingClassMetadata);
    
    	@Nullable
    	default Predicate<String> getExclusionFilter() {
    		return null;
    	}
    
    }
    ```

    > selectImports 方法返回值就是需要导入到容器中的组件全类名

  - ImportBeanDefinitionRegistrar的实现

    ```java
    public interface ImportBeanDefinitionRegistrar {
    	/**
    	 * AnnotationMetadata :当前类的注释信息
    	 * BeanDefinitionRegistry :BeanDefinition注册类
    	 */
    	default void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
            /**
             * 将需要注入的类,手动注入
             * String var1 :自定义bean名称
             * BeanDefinition var2 :指定bean的定义信息(类型,作用域...)
             */
            BeanDefinitionRegistry.registerBeanDefinition(String var1, BeanDefinition var2);
    	}
    
    }
    ```

    

