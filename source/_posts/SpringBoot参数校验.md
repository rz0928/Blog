---
title: SpringBoot参数校验
date: 2024-08-13 23:56:00
tags:
   - SpringBoot
categories:
   - 后端开发
---

# 1. 参数校验

## 1.1 Spring参数校验快速开始



### 1.1.1 导入依赖

```xml
 <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-validation</artifactId>
  </dependency>
```

### 1.1.2 使用

参数校验实操分两步

- 在Controller层将要校验的参数上标注`@Vaild`注解
- 在要校验对象属性加上具体的约束

**例子**：

1. 我要校验登录的用户名和密码不能为空，那么我就在参数loginUserDTO前加上`@valid`注解。

   ```java
   @RestController
   public class testVaildController {
       @Resource
       public LoginService loginService;
       @PostMapping("/login")
       public ResponseResult login(@RequestBody @Valid LoginUserDTO loginUserDTO){
           return ResponseResult.success(loginService.login(loginUserDTO.getUserName(),loginUserDTO.getPassword()));
       }
   }
   ```

2. 然后在LoginUserDTO类中加上具体的约束

   ```java
   public class LoginUserDTO {
      @NotBlank(message = "用户名不能为空")
      private String userName = "";
      @NotBlank(message = "密码不能为空")
      private String password = "";
   
      
      public String getUserName() {
         return this.userName;
      }
   
      public void setUserName(String userName) {
         this.userName = userName;
      }
   
      public String getPassword() {
         return this.password;
      }
   
      public void setPassword(String password) {
         this.password = password;
      }
   }
   ```

   这样参数校验就可以生效了，测试接口结果如下：

   ```json
   {
       "timestamp": "2024-05-24T09:15:58.364+00:00",
       "status": 400,
       "error": "Bad Request",
       "path": "/login"
   }
   ```

3. 异常捕获

   为了**更有效的传递信息**以及进行**结果统一返回**、**日志记录**等操作，我们通常会定义异常处理器来统一处理参数校验的异常（BindException是什么见下文异常信息）

   ```java
   @RestControllerAdvice
   public class GlobalExceptionHandler {
       Logger logger = LoggerFactory.getLogger("GlobalExceptionHandler");
       //抛出的异常是MethodArgumentNotValidException,BindException是它的父类
       @ExceptionHandler(BindException.class)
       public ResponseResult bindExceptionHandler(BindException e) {
           //具体返回什么据情况而定，这里直接返回所有defalutMessage(写在约束注解里的信息)了
           List<String> list = new ArrayList<>();
           BindingResult bindingResult = e.getBindingResult();
           for (ObjectError objectError : bindingResult.getAllErrors()) {
               FieldError fieldError = (FieldError) objectError;
               //这里直接使用默认appender打印日志在控制台了，真实线上可以同步到本地、ELK等地方
               logger.error("参数 {} ,{} 校验错误：{}", fieldError.getField(), fieldError.getRejectedValue(), fieldError.getDefaultMessage());
               list.add(fieldError.getDefaultMessage());
           }
           return ResponseResult.error(AppHttpCodeEnum.SYSTEM_ERROR.getCode(),list.toString());
       }
   }
   ```

**测试数据与结果如下**:

```json
{
    "userName": "",
    "password": ""
}

{
    "data": null,
    "code": 500,
    "msg": "[用户名不能为空, 密码不能为空]"
}
```

## 1.2 常用的校验注解

1. **控制检查**

   | 注解        | 说明                                   |
   | ----------- | -------------------------------------- |
   | `@NotBlank` | 用于字符串，字符串不能为null也不能为空 |
   | `@NotEmpty` | 字符串同上，集合不能为空，必须有元素   |
   | `@NotNull`  | 不能为null                             |
   | `@Null`     | 必须为null                             |

2. **数值检查**

   | 注解                        | 说明                                                         |
   | --------------------------- | ------------------------------------------------------------ |
   | `@DecimalMax(value)`        | 被标注元素必须是数字，必须小于等于value                      |
   | `@DecimalMin(value)`        | 被标注元素必须是数字，必须大于等于value                      |
   | `@Digits(integer,fraction)` | 被标注的元素必须为数字，其值的整数部分精度为 `integer`，小数部分精度为 `fraction` |
   | `@Positive`                 | 被标注的元素必须为正数                                       |
   | `@PositiveOrZero`           | 被标注的元素必须为正数或 0                                   |
   | `@Max(value)`               | 被标注的元素必须小于等于指定的值                             |
   | `@Min(value)`               | 被标注的元素必须大于等于指定的值                             |
   | `@negative`                 | 被标注的元素必须为负数                                       |
   | `NegativeOrZero`            | 被标注的元素必须为负数或 0                                   |

3. **Boolean检查**（不太用的到的样子）

   | 注解         | 说明                         |
   | ------------ | ---------------------------- |
   | @AssertFalse | 被标注的元素必须值为 `false` |
   | @AssertTrue  | 被标注的元素必须值为 `true`  |

4. **长度检查**

   | 注解           | 说明                                                         |
   | -------------- | ------------------------------------------------------------ |
   | @Size(min,max) | 被标注的元素长度必须在 `min` 和 `max` 之间，可以是 String、Collection、Map、数组 |

5. **日期检查**

   | 注解               | 说明                                 |
   | ------------------ | ------------------------------------ |
   | `@Future`          | 被标注的元素必须是一个将来的日期     |
   | `@FutureOrPresent` | 被标注的元素必须是现在或者将来的日期 |
   | `@Past`            | 被标注的元素必须是一个过去的日期     |
   | `@PastOrPresent`   | 被标注的元素必须是现在或者过去的日期 |

6. **其他**

   | 注解             | 说明                           |
   | ---------------- | ------------------------------ |
   | @Email           | 被标注的元素必须是电子邮箱地址 |
   | @Pattern(regexp) | 被标注的元素必须符合正则表达式 |


## 1.3 @Vaild与@Vaildated

在第一步加注解的时候，可以明显的看到还有一个可能也是参数校验的注解`@Validated`，把`@Vaild`换成`@Validated`，我们惊奇的发现参数校验也能正常的工作，接下来我们就来看看这两注解之间的联系与区别。

### 1.3.1 来源

一个是**Spring**的，一个是**javax**，了解过`@Autowired`与`@Resource`区别的老哥可能很快就反应过来了，就像这两个注解一样。一个是JSR规范的，一个是Spring规范的

```java
import org.springframework.validation.annotation.Validated;
import javax.validation.Valid;
```

### 1.3.2 定义区别

```java
@Target({ElementType.TYPE, ElementType.METHOD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Validated {
    Class<?>[] value() default {};
}

@Target({ElementType.METHOD, ElementType.FIELD, ElementType.CONSTRUCTOR, ElementType.PARAMETER, ElementType.TYPE_USE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Valid {
}
```

**差异主要有两点**：

- 能标注的地方

  可以看到`@Valid`注解除了能标注在**类、方法、方法参数**上还能标注在**类里面属性、任何使用类型的地方**。

- 属性

  `Validated`注解里面比`Valid`多了一个value 

接下来我们来结合定义区别看看它们的功能差异

### 1.3.3 功能差异

#### 嵌套校验

上文校验的时候我们在类属性上加上相关的限制注解就可以了，但是如果属性是一个类的实例，我们想校验这个作为属性的实例里面的字段，我们就只能使用`@Vaild`加在属性上，标注这是需要校验的属性。**`@Validated`不能标注在属性上**，自然也就**不支持嵌套校验**

```java
@PostMapping("/buy")
    public ResponseResult buy(@RequestBody @Valid ShoppingCart shoppingCart){
        return ResponseResult.success(payService.pay(shoppingCart.getUserId(),shoppingCart.getItemList()));
    }
//购物车类
public class ShoppingCart {
    @Positive(message = "用户id必须大于0")
    private Long userId;
    @NotEmpty(message = "不能为空")
    @Valid
    private List<Item> itemList;

    public Long getUserId() {
        return userId;
    }

    public void setUserId(Long userId) {
        this.userId = userId;
    }

    public List<Item> getItemList() {
        return itemList;
    }

    public void setItemList(List<Item> itemList) {
        this.itemList = itemList;
    }
}
//物品类
public class Item {
    @Positive(message = "价格必须大于0")
    private BigDecimal price;
    @NotBlank(message = "物品名不能为空")
    private String name;
    @Positive(message = "数量必须大于0")
    private int number;

    public BigDecimal getPrice() {
        return price;
    }

    public void setPrice(BigDecimal price) {
        this.price = price;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getNumber() {
        return number;
    }

    public void setNumber(int number) {
        this.number = number;
    }
}
//...服务自行模拟
```

**结果**

```json
{
    "userId": "1",
    "itemList":[
        {
        "price": "142.0",
        "name": "小风车",
        "number": 1
        },
        {
        "price": "-2",
        "name": "",
        "number": -1
        }
    ]
}

{
    "data": null,
    "code": 500,
    "msg": "[数量必须大于0, 价格必须大于0, 物品名不能为空]"
}
```

可以看到校验成功了，当然这里的异常捕获逻辑比较简单，具体生产环境里可以将返回具体的校验信息，进行更清晰的信息提示。



**杂记**：

- `@Valid`作用域比较广还可以标注在许多意向不到的位置

  ```java
  //比如，(可以自行观察一下运行结果,都是可以正常校验的)
  private  List<@Valid Item> itemList;
  private @Valid List< Item> itemList;
  ```

- `@Valid`与`@Validated`可以混用

  ```java
  //controller里面注解改为@Validated依然可以生效 
  @PostMapping("/buy")
      public  ResponseResult  buy(@RequestBody @Validated ShoppingCart  shoppingCart){
          return ResponseResult.success(payService.pay(shoppingCart.getUserId(),shoppingCart.getItemList()));
      }
  ```

- 消息的顺序不固定

  ```java
  {
      "data": null,
      "code": 500,
      "msg": "[数量必须大于0, 价格必须大于0, 物品名不能为空]" //这三条消息打印的顺序是不固定的
  }
  ```



#### 分组校验

上面看注解代码的时候可以明显注意到`@Validated`注解里面有个`value`属性。**`@Valid`没有value属性**，自然也就**不支持分组校验**

```java
@Target({ElementType.TYPE, ElementType.METHOD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Validated {
    //传入分组,默认是Defalut.class
    Class<?>[] value() default {};
}
```

这就和`@Validated`的分组校验有关了，(`@Valid`没有定义这个属性，自然也就不支持分组校验）

```java
//Default是个接口，只起到标记作用
package javax.validation.groups;

public interface Default {
}
```

我们可以自定义分组来指定需要校验的时机。

```java
//定义Group类，里面两个接口用作校验(也可以用类，不过和原生的贴合一点比较好)
public class Group {
   public interface GroupTest1{}
   public interface GroupTest2{}
}
```

**测试**

```java
public class LoginUserDTO {
    //只有分组属于这两个才校验
   @NotBlank(message = "账户不能为空",groups = {Group.GroupTest1.class, Default.class})
   private String userName = "";
   @NotBlank(message = "密码不能为空")
   private String password = "";

   
   public String getUserName() {
      return this.userName;
   }

   public void setUserName(String userName) {
      this.userName = userName;
   }

   public String getPassword() {
      return this.password;
   }

   public void setPassword(String password) {
      this.password = password;
   }
}
//controller
	@PostMapping("/login")
    public ResponseResult login(@RequestBody @Validated(value = {Group.GroupTest2.class}) LoginUserDTO loginUserDTO){
        return ResponseResult.success(loginService.login(loginUserDTO.getUserName(),loginUserDTO.getPassword()));
    }

```

**结果**：

```json
//不属于分组，不校验userName（其他情况自行尝试
{
    "userName":"",
    "password":"123456"
}
{
    "data": "登录成功",
    "code": 200,
    "msg": "操作成功"
}
```



# 2. 深入理解@Valid与@Validated

因为`@Valid`与`@Validated`能混合使用，我们可以大胆猜测一下，一定有一个`Adapter`来承担两者的适配工作，搜搜Valid相关的Adapter，还真有一个。

## 2.1 SpringValidtorAdapter

```java
public class SpringValidatorAdapter implements SmartValidator, javax.validation.Validator {
    	@Nullable
	private javax.validation.Validator targetValidator;
    //没有返回值，重写Spring中Validator的方法
	@Override
	public void validate(Object target, Errors errors, Object... validationHints) {
		if (this.targetValidator != null) {
			processConstraintViolations(
					this.targetValidator.validate(target, asValidationGroups(validationHints)), errors);
		}
	}
    //有返回值，重写的javax中Validator的方法
    @Override
	public <T> Set<ConstraintViolation<T>> validate(T object, Class<?>... groups) {
		Assert.state(this.targetValidator != null, "No target Validator set");
		return this.targetValidator.validate(object, groups);
	}
    
}
```

**SpringValidtorAdapter的结构**

![image-20240602190311638](F:/Blog/source/_posts/SpringBoot参数校验/image-20240602190311638.png)

根据上面结构图和代码，可以看到`SpringValidatorAdapter`实现了两个不同的`validator`接口，针对其中核心方法`validate()`进行了返回值的适配（有点像`Runnable`和` Callable`之间的适配）

## 2.2 具体校验器的获取实现

### 2.2.1 ValidatorFactory

```java
public interface ValidatorFactory extends AutoCloseable {

	// 显然，这个接口是最为重要的
	Validator getValidator();
	// 定义一个新的ValidatorContext验证器上下文，并且和Validator关联上
	ValidatorContext usingContext();
    
	MessageInterpolator getMessageInterpolator();
	TraversableResolver getTraversableResolver();
	ConstraintValidatorFactory getConstraintValidatorFactory();
	ParameterNameProvider getParameterNameProvider();
	ClockProvider getClockProvider();

	public <T> T unwrap(Class<T> type);
	// 复写AutoCloseable的方法
	@Override
	public void close();

}
```

`ValidatorFactory`的实现可以分成两部分，`hibernate`和`Spring`实现。

#### LocalValidatorFactoryBean

`LocalValidatorFactoryBean`不仅是`ValidatorFactory`实现，还是`SpringValidAdapter`子类

```java
public class LocalValidatorFactoryBean extends SpringValidatorAdapter
		implements ValidatorFactory, ApplicationContextAware, InitializingBean, DisposableBean {
    
    //重写的InitializingBean中的方法，Bean初始化时会执行
    @Override
	@SuppressWarnings({"rawtypes", "unchecked"})
	public void afterPropertiesSet() {
		Configuration<?> configuration;
        
        ...
      
        //根据配置创建工厂，并从工厂里面拿到Validator
		try {
			this.validatorFactory = configuration.buildValidatorFactory();
			setTargetValidator(this.validatorFactory.getValidator());
		}
		finally {
			closeMappingStreams(mappingStreams);
		}
	}
}
```

#### ValidatorFactoryImpl

在最开始我们导入了校验starter，导入的就有**org.hibernate.validator**相关的类

```java
package org.hibernate.validator.internal.engine;

public class ValidatorFactoryImpl implements HibernateValidatorFactory {
    @Override
	public Validator getValidator() {
		return createValidator(
				constraintValidatorManager.getDefaultConstraintValidatorFactory(),
				valueExtractorManager,
				validatorFactoryScopedContext,
				methodValidationConfiguration
		);
	}
    
    Validator createValidator(ConstraintValidatorFactory constraintValidatorFactory, 
                              ValueExtractorManager valueExtractorManager,
			ValidatorFactoryScopedContext validatorFactoryScopedContext,
                              MethodValidationConfiguration methodValidationConfiguration) {
        
		BeanMetaDataManager beanMetaDataManager = beanMetaDataManagers.computeIfAbsent(
				new BeanMetaDataManagerKey( validatorFactoryScopedContext.getParameterNameProvider(), 
                                           valueExtractorManager, methodValidationConfiguration ),
				key -> new BeanMetaDataManager(
						constraintHelper,
						executableHelper,
						typeResolutionHelper,
						validatorFactoryScopedContext.getParameterNameProvider(),
						valueExtractorManager,
						validationOrderGenerator,
						buildDataProviders(),
						methodValidationConfiguration
				)
		 );

		return new ValidatorImpl(
				constraintValidatorFactory,
				beanMetaDataManager,
				valueExtractorManager,
				constraintValidatorManager,
				validationOrderGenerator,
				validatorFactoryScopedContext
		);
	}
}
```

### 2.2.2 ValidatorImpl

`ValidatorImpl`是**Hibernate Validator**提供的唯一校验器实现，校验过程很复杂，了解是哪个类就行，感兴趣可以深度剖析

```java
public class ValidatorImpl implements Validator, ExecutableValidator {
    private static final Collection<Class<?>> DEFAULT_GROUPS = Collections.<Class<?>>singletonList( Default.class );

	// 分组Group校验的顺序问题
	// 若依赖于校验顺序，可用使用@GroupSequence注解来控制Group顺序
	private final transient ValidationOrderGenerator validationOrderGenerator;
	private final ConstraintValidatorFactory constraintValidatorFactory;
	...


	@Override
	public final <T> Set<ConstraintViolation<T>> validate(T object, Class<?>... groups) {
		Contracts.assertNotNull( object, MESSAGES.validatedObjectMustNotBeNull() );
        // groups里面的内容不能有null
		sanityCheckGroups( groups );

		@SuppressWarnings("unchecked")
		Class<T> rootBeanClass = (Class<T>) object.getClass();
		BeanMetaData<T> rootBeanMetaData = beanMetaDataManager.getBeanMetaData( rootBeanClass );
        //没有约束直接返回
		if ( !rootBeanMetaData.hasConstraints() ) {
			return Collections.emptySet();
		}
        
		BaseBeanValidationContext<T> validationContext = getValidationContextBuilder().forValidate( rootBeanClass, rootBeanMetaData, object );

		ValidationOrder validationOrder = determineGroupValidationOrder( groups );
         // ValueContext一个实例用于收集所有相关信息，以验证单个类、属性或方法调用。
		BeanValueContext<?, Object> valueContext = ValueContexts.getLocalExecutionContextForBean(
				validatorScopedContext.getParameterNameProvider(),
				object,
				validationContext.getRootBeanMetaData(),
				PathImpl.createRootPath()
		);
        // 返回的是失败的消息对象：ConstraintViolation  它是被存储在ValidationContext里的
		return validateInContext( validationContext, valueContext, validationOrder );
        
}
```

# 3. 异常信息

## 3.1 说明

参数校验失败抛出的异常时`MethodArgumentNotValidException`，不过建议捕获`BindException`，因为`MethodArgumentNotValidException`是对`BindResult`的封装，只能通过`getMessage()`获取封装好的信息，不够灵活

```java
//看的出来，确实很简单
public class MethodArgumentNotValidException extends BindException {
    private final MethodParameter parameter;

    public MethodArgumentNotValidException(MethodParameter parameter, BindingResult bindingResult) {
        super(bindingResult);
        this.parameter = parameter;
    }

    public final MethodParameter getParameter() {
        return this.parameter;
    }

    public String getMessage() {
        StringBuilder sb = (new StringBuilder("Validation failed for argument [")).append(this.parameter.getParameterIndex()).append("] in ").append(this.parameter.getExecutable().toGenericString());
        BindingResult bindingResult = this.getBindingResult();
        if (bindingResult.getErrorCount() > 1) {
            sb.append(" with ").append(bindingResult.getErrorCount()).append(" errors");
        }

        sb.append(": ");
        Iterator var3 = bindingResult.getAllErrors().iterator();

        while(var3.hasNext()) {
            ObjectError error = (ObjectError)var3.next();
            sb.append('[').append(error).append("] ");
        }

        return sb.toString();
    }
}
```

## 3.2 BindException

```java
//BindException里面主要是封装了一个BindResult
public class BindException extends Exception implements BindingResult {
    private final BindingResult bindingResult;

    public BindException(BindingResult bindingResult) {
        Assert.notNull(bindingResult, "BindingResult must not be null");
        this.bindingResult = bindingResult;
    }

    public BindException(Object target, String objectName) {
        Assert.notNull(target, "Target object must not be null");
        this.bindingResult = new BeanPropertyBindingResult(target, objectName);
    }
}
```

### 3.2.1 BindResult

BindResult是一个接口，里面没有什么特别的东西。

```java
public interface BindingResult extends Errors {
    String MODEL_KEY_PREFIX = BindingResult.class.getName() + ".";

    @Nullable
    Object getTarget();

    Map<String, Object> getModel();

    @Nullable
    Object getRawFieldValue(String field);

    @Nullable
    PropertyEditor findEditor(@Nullable String field, @Nullable Class<?> valueType);

    @Nullable
    PropertyEditorRegistry getPropertyEditorRegistry();

    String[] resolveMessageCodes(String errorCode);

    String[] resolveMessageCodes(String errorCode, String field);

    void addError(ObjectError error);

    default void recordFieldValue(String field, Class<?> type, @Nullable Object value) {
    }

    default void recordSuppressedField(String field) {
    }

    default String[] getSuppressedFields() {
        return new String[0];
    }
}
```

### 3.2.2 AbstractBindingResult

一般来说抽象类都是接口的基本实现，BindingResult也不例外

```java
public abstract class AbstractBindingResult extends AbstractErrors implements BindingResult, Serializable {
    private final String objectName;
    private MessageCodesResolver messageCodesResolver = new DefaultMessageCodesResolver();
    //储存错误信息
    private final List<ObjectError> errors = new ArrayList();
    private final Map<String, Class<?>> fieldTypes = new HashMap();
    private final Map<String, Object> fieldValues = new HashMap();
    private final Set<String> suppressedFields = new HashSet();
    
    ...
        
}
```

#### ObjectError

1. 本体

   ```java
   public class ObjectError extends DefaultMessageSourceResolvable {
       //校验失败的对象名，比如shoppingCart
       private final String objectName;
       @Nullable
       private transient Object source;
       
   	...       
   }
   ```

2. 子类

   ```java
   //ObjectError的子类，获取ObjectError可以强转为该类 (具体实现是FieldError的子类ViolationFieldError）
   public class FieldError extends ObjectError {
       //校验失败的地方，比如itemList[1].name
       private final String field;
       //校验失败的值，比如-1
       @Nullable
       private final Object rejectedValue;
       private final boolean bindingFailure;
       ...
   }
   ```

3. 父类

   ```java
   //ObjectError的父类
   public class DefaultMessageSourceResolvable implements MessageSourceResolvable, Serializable {
       //存放详细的校验类型信息
       @Nullable
       private final String[] codes;
       @Nullable
       private final Object[] arguments;
       //注解里面传的Message到这了
       @Nullable
       private final String defaultMessage;
       
    	...   
           
   }
   ```

   

**Tips**：可以自行打印观察值，推测存放信息

```java
 @ExceptionHandler(BindException.class)
    public ResponseResult bindExceptionHandler(BindException e) {
        List<String> list = new ArrayList<>();
        BindingResult bindingResult = e.getBindingResult();
        for (ObjectError objectError : bindingResult.getAllErrors()) {
            FieldError fieldError = (FieldError) objectError;
            System.out.println("--------------");
            System.out.println("codes");
            Arrays.stream(fieldError.getCodes()).forEach(System.out::println);
            System.out.println("--------------");
            System.out.println("code");
            System.out.println(fieldError.getCode());
            System.out.println("--------------");
            System.out.println("ObjectName");
            System.out.println(fieldError.getObjectName());
            logger.error("参数 {} ,{} 校验错误：{}", fieldError.getField(), fieldError.getRejectedValue(), fieldError.getDefaultMessage());
            list.add(fieldError.getDefaultMessage());
        }
        return ResponseResult.error(AppHttpCodeEnum.SYSTEM_ERROR.getCode(),list.toString());
    }
```







