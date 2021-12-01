# SpringBoot工厂

## 工厂模式和策略模式

> 自定义注解

```java
/**
 * @author igsshan
 * @data 2021-11-29
 * @Description：需要注入工厂容器的bean
 */
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ThirdFactoryBean {

    String beanName();
}
```

> 注册Bean实例

```java
/**
 * @author igsshan
 * @data 2021-11-29
 * @Description：将第三方工厂实现注入工厂容器
 */
@Component
public class InitThirdFactoryBeanProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        Class clz = bean.getClass();
        if (clz.isAnnotationPresent(ThirdFactoryBean.class)) {
            ThirdFactoryBean iotOpt = (ThirdFactoryBean)clz.getAnnotation(ThirdFactoryBean.class);
            String factoryBeanName = iotOpt.beanName();
         ThirdFactoryUtil.getFactoryMap().put(factoryBeanName,clz);
        }
        return bean;
    }
}
```

> service层

```java
/**
 * @author igsshan
 * @data 2021-11-29
 */
public interface ThirdFactoryService {
}
```

> 策略实现

```java
@Slf4j
@Service
@ThirdFactoryBean(beanName = "demo1")
public class Demo1Service implements ThirdFactoryService {
    // 功能实现
}
```

```java
@Slf4j
@Service
@ThirdFactoryBean(beanName = "demo2")
public class Demo2Service implements ThirdFactoryService {
    // 功能实现
}
```

