# SpringBoot之@Validated注解

## 为什么使用@Validation来验证参数

> 在后端编写接口时,对于部分接口的参数需要进行非空或者格式校验来避免程序出错,普遍使用 if - else 逐个对参数进行校验,可以达到预期效果,但是对于代码的可读性和美观程度来说,效果不好.那么我们就可以使用@Validated注解进行优雅校验参数 

## 什么是@Validated

> @Validation是一套帮助我们继续对外传输的参数进行数据校验的注解,通过配置Validation可以很轻松的完成对数据的约束

```java
@Target({ElementType.TYPE, ElementType.METHOD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Validated {
    Class<?>[] value() default {};
}
```

## 如何使用@Validated进行参数校验

> - 为实体类中的参数或者对象添加相应的注解
> - 在控制器层(controller)进行注解声明,或者手动调用校验方法进行校验
> - 对异常进行处理



### 空和非空检查: @Null、@NotNull、@NotBlank、@NotEmpty

| 注解      | 支持Java类型                           | 备注                                                         |
| --------- | -------------------------------------- | ------------------------------------------------------------ |
| @Null     | Object                                 | 验证元素值为null                                             |
| @NotNull  | Object                                 | 验证元素值不能为 null                                        |
| @NotBlank | CharSequence                           | 验证元素值不为null且移除两边空格后长度大于0                  |
| @NotEmpty | CharSequence,Collection,Map and Arrays | 验证元素值不为null且不为空（字符串长度不为0、集合大小不为0） |

### Boolean值检查: @AssertTrue、@AssertFalse

| 注解         | 支持Java类型     | 备注                             |
| ------------ | ---------------- | -------------------------------- |
| @AssertTrue  | Boolean, boolean | 验证元素值必须为true，否则抛异常 |
| @AssertFalse | Boolean, boolean | 验证元素值必须为flase            |

### 长度检查: @Size、@Length

| 注解    | 支持Java类型                              | 备注                       |
| ------- | ----------------------------------------- | -------------------------- |
| @Size   | String,Collection,Map,arrays,CharSequence | 验证元素个数包含在一个区间 |
| @Length | CharSequence                              | 验证元素值包含在一个区间   |

### 日期检查: @Future、@FutureOrPresent、@Past、@PastOrPresent

| 注解             | 支持Java类型                       | 备注                             |
| ---------------- | ---------------------------------- | -------------------------------- |
| @Future          | java.util.Date, java.util.Calendar | 验证日期为当前时间之后           |
| @FutureOrPresent | java.util.Date, java.util.Calendar | 验证日期为当前时间或之后一个时间 |
| @Past            | java.util.Date, java.util.Calendar | 验证日期为当前时间之前           |
| @PastOrPresent   | java.util.Date, java.util.Calendar | 验证日期为当前时间或之前         |

### 其它检查: @Email、@CreditCardNumber、@URL、@Pattern、@ScriptAssert、@UniqueElements

| 注解              | 支持Java类型 | 备注                                  |
| ----------------- | ------------ | ------------------------------------- |
| @Email            | CharSequence | 验证日期为当前时间之后                |
| @CreditCardNumber | CharSequence | 验证日期为当前时间或之后一个时间      |
| @URL              | CharSequence | 验证日期为当前时间之前                |
| @Pattern          | CharSequence | 验证日期为当前时间或之前              |
| @Valid            | Object       | 验证关联对象元素进行递归校验检查      |
| @UniqueElements   | Collection   | 校验集合中的元素必须保持唯一 否则异常 |

### 数值检查: @Min、@Max、@Range、@DecimalMin、@DecimalMax、@Digits

| 注解                               | 支持Java类型                                          | 备注                                     |
| ---------------------------------- | ----------------------------------------------------- | ---------------------------------------- |
| @Min                               | BigDecimal, BigInteger, byte, short,int, long,Number. | 检验当前数值大于等于指定值               |
| @Max                               | CharSequence                                          | 检验当前数值小于等于指定值               |
| @Range                             | CharSequence                                          | 验证数值为指定值区间范围内               |
| @DecimalMin                        | CharSequence                                          | 验证数值是否大于等于指定值               |
| @DecimalMax                        | Object                                                | 验证数值是否小于等于指定值               |
| @Digits(integer = 3, fraction = 2) |                                                       | 验证注解的元素值的整数位数和小数位数上限 |

## @Valid和@Validated 区别

> @Valid

```java
@Target({ElementType.METHOD, ElementType.FIELD, ElementType.CONSTRUCTOR, ElementType.PARAMETER, ElementType.TYPE_USE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Valid {
}
```

- @Valid：没有分组的功能。
- @Valid：可以用在方法、构造函数、方法参数和成员属性（字段）上
- @Validated：提供了一个分组功能，可以在入参验证时，根据不同的分组采用不同的验证机制
- @Validated：可以用在类型、方法和方法参数上。但是不能用在成员属性（字段）上

两者是否能用于成员属性（字段）上直接影响能否提供嵌套验证的功能

## @Validation使用

> 实体类

```java
@Data
public class BasePageDto implements Serializable{

    @NotNull(message = "pageSize不能为空", groups = {QueryGroup.class})
    private Integer pageSize;

    @NotNull(message = "pageNum不能为空", groups = {QueryGroup.class})
    private Integer pageNum;
}
```

> 分组类

```java
public interface QueryGroup {
}
```

> 控制器层

```java
@Slf4j
@RestController
@RequestMapping("/test")
public class AlarmTaskController {

    @RequestMapping(value = "/list")
    public Result list(@Validated({ QueryGroup.class }) BasePageDto pageParams) {
        return result;
    }
}
```

