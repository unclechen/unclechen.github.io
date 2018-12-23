---
layout: post
title: SpringBoot自定义请求参数校验
date: '2018-12-15'
tags:
  - Spring
  - SpringBoot
  - 后台
  - Service
categories:
  - 技术
---

最近在工作中遇到写一些API，这些API的请求参数非常多，嵌套也非常复杂，如果参数的校验代码全部都手动去实现，写起来真的非常痛苦。正好`Spring`轮子里面有一个`Validation`，这里记录一下怎么使用，以及怎么自定义它的返回结果。

<!-- more -->

# 一、Bean Validation基本概念

[Bean Validation](https://beanvalidation.org/)是Java中的一项标准，它通过一些**注解**表达了对实体的限制规则。通过提出了一些API和扩展性的规范，这个规范是没有提供具体实现的，希望能够`Constrain once, validate everywhere`。现在它已经发展到了2.0，兼容Java8。

[`hibernate validation`](http://hibernate.org/validator/)实现了Bean Validation标准，里面还增加了一些注解，在程序中引入它我们就可以直接使用。

`Spring MVC`也支持`Bean Validation`，它对`hibernate validation`进行了二次封装，添加了自动校验，并将校验信息封装进了特定的`BindingResult `类中，在SpringBoot中我们可以添加`    implementation('org.springframework.boot:spring-boot-starter-validation')
`引入这个库，实现对bean的校验功能。


# 二、基本用法

`gradle dependencies`如下：

```
dependencies {
    implementation('org.springframework.boot:spring-boot-starter-validation')
    implementation('org.springframework.boot:spring-boot-starter-web')
}
```

定义一个示例的Bean，例如下面的`User.java`。

```java
public class User {

    @NotBlank
    @Size(max=10)
    private String name;
    
    private String password;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }
}
```

在`name`属性上，添加`@NotBlank`和`@Size(max=10)`的注解，表示`User`对象的`name`属性不能为字符串且长度不能超过10个字符。

然后我们暂时不添加任何多余的代码，直接写一个`UserController`对外提供一个RESTful的`GET`接口，注意接口的参数用到了`@Validated注解`。

```java
// UserController.java，省略其他代码
@RestController
public class UserController {

    @RequestMapping(value = "/validation/get", method = RequestMethod.GET)
    public ServiceResponse validateGet(@Validated User user) {
        ServiceResponse serviceResponse = new ServiceResponse();
        serviceResponse.setCode(0);
        serviceResponse.setMessage("test");
        return serviceResponse;
    }
}

// ServiceResponse.java，简单包含了code、message字段返回结果。
public class ServiceResponse {

    private int code;
    private String message;
    ... 省略getter、setter ...
}
```

启动SpringBoot程序，发一个测试请求看一下：

`http://127.0.0.1:8080/validation/get?name=aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa&password=1`

返回的结果是，注意此时的`HTTP STATUS CODE = 400`：

![](https://ws3.sinaimg.cn/large/006tNbRwly1fy8m53m0sbj31fe0u0akp.jpg)

此时已经可以实现参数的校验了，但是返回的结果不太友好，下面看一下怎么定制返回的消息。在定制返回结果前，先看下一下内置的校验注解有哪些，在这里我不一个个去贴了，写代码的时候根据需要进入到源码里面去看即可。

<img src="https://ws3.sinaimg.cn/large/006tNbRwly1fy8n5hndwpj30mo104aed.jpg" width="360"/>

> 早期Spring版本中，都是在Controller的方法中添加Errors/BindingResult参数，由Spring注入Errors/BindingResult对象，再在Controller中手写校验逻辑实现校验。新版本提供注解的方式（Controller上面bean加一个@Validated注解），将校验逻辑和Controller分离。

# 三、自定义校验

## 3.1 自定义注解

显然除了自带的`NotNull`、`NotBlank`、`Size`等注解，实际业务上还会需要特定的校验规则。

假设我们有一个参数`address`，必须以`Beijing`开头，那我们可以定义一个注解和一个自定义的`Validator`。

```java
// StartWithValidator.java
public class StartWithValidator implements ConstraintValidator<StartWithValidation, String> {

    private String start;

    @Override
    public void initialize(StartWithValidation constraintAnnotation) {
        start = constraintAnnotation.start();
    }

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        if (!StringUtils.isEmpty(value)) {
            return value.startsWith(start);
        }
        return true;
    }
}


// StartWithValidation.java
@Documented
@Constraint(validatedBy = StartWithValidator.class)
@Target({METHOD, FIELD})
@Retention(RUNTIME)
public @interface StartWithValidation {

    String message() default "不是正确的性别取值范围";

    String start() default "_";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};

    @Target({METHOD, FIELD, ANNOTATION_TYPE, CONSTRUCTOR, PARAMETER})
    @Retention(RUNTIME)
    @Documented
    @interface List {
        GenderValidation[] value();
    }
}
```

然后在`User.java`中增加一个`address`属性，并给它加上上面这个自定义的注解，这里我们定义了一个可以传入`start`参数的注解，表示应该以什么开头。

```java
...
@StartWithValidation(message = "Param 'address' must be start with 'Beijing'.", start = "Beijing")
private String address;
...
```

**除了定义可以作用于属性的注解外，其实还可以定义作用于class的注解（@Target({TYPE})），用于校验class的实例。**

## 3.2 自定义Validator

第一步，实现一个`Validator`。(这种方法不需要我们的bean里面有任何注解之类的东西)

```
package com.example.validation.demo;

import org.springframework.stereotype.Component;
import org.springframework.validation.Errors;
import org.springframework.validation.ValidationUtils;
import org.springframework.validation.Validator;

@Component
public class UserValidator implements Validator {
    @Override
    public boolean supports(Class<?> clazz) {
        return User2.class.equals(clazz);
    }

    @Override
    public void validate(Object target, Errors errors) {
        ValidationUtils.rejectIfEmpty(errors, "name", "name.empty");
        User2 p = (User2) target;
        if (p.getId() == 0) {
            errors.rejectValue("id", "can not be zero");
        }
    }
}

```

第二步，修改Controller代码，注入上面的UserValidator实例，并给Controller的方法参数加上`@Validated`注解，即可完成和前面自定义注解一样的校验功能。

```
@RestController
public class UserController {

    @Autowired
    UserValidator validator;

    @InitBinder
    public void initBinder(WebDataBinder binder) {
        binder.setValidator(validator);
    }

    @RequestMapping(value = "/user/post", method = RequestMethod.POST)
    public ServiceResponse handValidatePost(@Validated @RequestBody User user) {
        ServiceResponse serviceResponse = new ServiceResponse();
        serviceResponse.setCode(0);
        serviceResponse.setMessage("test");
        return serviceResponse;
    }
}
```

这个方法和自定义注解的区别在于不需要在Bean里面添加注解，并且可以更加灵活的把一个Bean里面所有的Field的校验代码都搬到一起，而不是每一个属性都去加注解，**如果校验的属性非常多，且默认注解的能力又不够的话，**这种方式也是不错的，可以避免大量的自定义注解。

## 3.3 以编程的方式校验（手动）

这种方式可以算是原始的Hibernate-Validation的方式。直接看代码，这里有一个比较不同的是，可以使用[Hibernate-Validation的Fail fast mode](https://docs.jboss.org/hibernate/stable/validator/reference/en-US/html_single/#section-fail-fast)。因为前面的方式，都将所有的参数都验证完了，再把错误返回。有时我们希望遇到一个参数错误，就立即返回。设置`fast-fail`为`true`可以达到这个目的。不过貌似不能再用@Validated注解方法参数了，而是要用`ValidatorFactory`创建`Validator`。**在实际开发中，不必每次都编写代码创建Validator，可以采用`@Configuration`的方式创建，然后再`@Autowired`注入到每个需要使用Validator的Controller当中。**

```
@RestController
public class UserController {
    ...
    @RequestMapping(value = "/validation/postStudent", method = RequestMethod.POST)
    public ServiceResponse validatePostStudent(@RequestBody User user) {
        // User参数前面没有@Validated注解了，User类里面那些注解还是保留着即可。
        HibernateValidatorConfiguration configuration = Validation.byProvider(HibernateValidator.class).configure();
        ValidatorFactory factory = configuration.failFast(true).buildValidatorFactory(); // fastFail
        Validator validator = factory.getValidator();
        Set<ConstraintViolation<User>> set = validator.validate(user);
        // 根据set的size，大于0时，抛异常。由于设置了failFast，这里set最多就一个元素
        ServiceResponse serviceResponse = new ServiceResponse();
        serviceResponse.setCode(0);
        serviceResponse.setMessage("test");
        return serviceResponse;
    }
}
```

## 3.4 定义分组校验

有的时候，我们会有两个不同的接口，但是会使用到同一个`Bean`来作为`VO`（意思是两个接口的URI不同，但参数中都用到了同一个Bean）。而在不同的接口上，对Bean的校验需求可能不一样，比如接口2需要校验`studentId`，而接口1不需要。那么此时就可以用到校验注解的分组`groups`。

```java
// User.java
public class User {
    ... 省略其他属性
    // 指明在groups={Student.class}时才需要校验studentId
    @NotNull(groups = {Student.class}, message = "Param 'studentId' must not be null.")
    private Long studentId;

    // 增加Student interface
    public interface Student {
    }
}

// UserController.java，增加了一个/getStudent接口
@RestController
public class UserController {

    @RequestMapping(value = "/validation/get", method = RequestMethod.GET)
    public ServiceResponse validateGet(@Validated User user) {
        ServiceResponse serviceResponse = new ServiceResponse();
        serviceResponse.setCode(200);
        serviceResponse.setMessage("test");
        return serviceResponse;
    }

    @RequestMapping(value = "/validation/getStudent", method = RequestMethod.GET)
    public ServiceResponse validateGetStudent(@Validated({User.Student.class}) User user) {
        ServiceResponse serviceResponse = new ServiceResponse();
        serviceResponse.setCode(0);
        serviceResponse.setMessage("test");
        return serviceResponse;
    }
}
```

到这里，也可以带一嘴`Valid`和`Validated`注解的区别，其代码注释写着后者是对前者的一个扩展，支持了`group`分组的功能。


## 3.5 定制返回码和消息

第二节中定义了一个`ServiceResponse`，其实作为一个开放的API，不论用户传入任何参数，返回的结果都应该是预先定义好的格式，并且可以写明在接口文档中，即使发生了校验失败，应该返回一个包含错误码`code`（发生错误时一般大于0）和`message`字段。

```json
{
    "code": 51000,
    "message": "Param 'name' must be less than 10 characters."
}
```

的结果，而`HTTP STATUS CODE`一直都是`200`。

为了实现这个目的，我们加一个全局异常处理方法。

```java
// ServiceExceptionHandler.java
package com.example.validation.demo;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.util.CollectionUtils;
import org.springframework.validation.BindException;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.util.List;

@RestControllerAdvice
public class ServiceExceptionHandler {

    static final Logger LOG = LoggerFactory.getLogger(ServiceExceptionHandler.class);


    @ExceptionHandler(value = {Exception.class})
    public ServiceResponse handleBindException(Exception ex) {
        LOG.error("{}", ex);
        StringBuilder message = new StringBuilder();
        if (ex instanceof BindException) {
            List<FieldError> fieldErrorList = ((BindException) ex).getFieldErrors();
            if (!CollectionUtils.isEmpty(fieldErrorList)) {
                for (FieldError fieldError : fieldErrorList) {
                    if (fieldError != null && fieldError.getDefaultMessage() != null) {
                        message.append(fieldError.getDefaultMessage()).append(" ");
                    }
                }
            }
        } else if (ex instanceof MethodArgumentNotValidException) {
            List<FieldError> fieldErrorList = ((MethodArgumentNotValidException) ex).getBindingResult().getFieldErrors();
            if (!CollectionUtils.isEmpty(fieldErrorList)) {
                for (FieldError fieldError : fieldErrorList) {
                    if (fieldError != null && fieldError.getDefaultMessage() != null) {
                        message.append(fieldError.getDefaultMessage()).append(" ");
                    }
                }
            }
        }

        // 生成返回结果
        ServiceResponse errorResult = new ServiceResponse();
        errorResult.setCode(51000); // ErrorCode.PARAM_ERROR = 51000
        errorResult.setMessage(message.toString());
        return errorResult;
    }
}

// User.java，注解传入指定Message
public class User {

    @NotBlank(message = "Param 'name' can't be blank.")
    @Size(max=10, message = "Param 'name' must be less than 10 characters.")
    private String name;
    ...
}
```

在上面的方法中，我们处理了`BindException`（非请求body参数，例如@RequestParam接收的）和`MethodArgumentNotValidException`（请求body里面的参数，例如@RequestBody接收的），这两类Exception里面都有一个`BindingResult`对象，它里面有一个包装成`FieldError`的List，保存着Bean对象出现错误的Field等信息。取出它里面`defaultMessage`，放到统一的`ServiceResponse`返回即可实现返回码和消息的定制。由于消息内容是有注解默认的`DefaultMessage`决定的，为了按照自定义的描述返回，在Bean对象的注解上需要手动赋值为希望返回的消息内容。


```
...
@NotBlank(message = "Param 'name' can't be blank.")
@Size(max=10,message = "Param 'name' must be less than 10 characters.")
private String name;
...
```

这样当`name`参数长度超过10时，就会返回

```
{
    "code": 51000,
    "message": "Param 'name' must be less than 10 characters."
}
```

这里的`FieldError fieldError = ex.getFieldError();`只会随机返回一个出错的属性，如果Bean对象的多个属性都出错了，可以调用`ex.getFieldErrors()`来获得，**这里也可以看到Spring Validation在参数校验时不会在第一次碰到参数错误时就返回，而是会校验完成所有的参数。如果不想手动编程去校验，那么这里可以只读取一个随机的FieldError，返回它的错误消息即可。**

## 3.6 更加细致的返回码和消息

其实还有一种比较典型的自定义返回，就是错误码（`code`）和消息（`message`）是一一对应的，比如：

- 51001：字符串长度过长
- 51002：参数取值过大
- ...

这种情况比较特殊，一般当参数错误的时候，会返回一个整体的参数错误的错误码，然后携带参数的错误信息。但有时，业务上就要不同的参数错误，既要错误码不同，错误信息也要不同。我想了下，有两种思路。

- 第一种：通过message同时包含错误码和错误信息，在全局异常捕获方法中，再把它们拆开。
- 第二种：手动校验，抛出自定义的Exception（里面带有code、message）。手动校验这里，如果每一个Controller都去写一遍，确实比较费劲，可以结合AOP来实现，或者抽出一个基类BaseController的方式。

# 四、小结

其实在实际的工作中，肯定还有更复杂的校验逻辑，但是不一定非要都用框架去实现，框架里面的实现（比如注解）应该是一个比较简单通用的校验，能够达到复用，减少重复的劳动。而更加复杂的逻辑校验，一定是存在具体业务当中的，最好是在业务代码里面实现。还有一点需要注意，Spring Validation的`isValid`方法，如果返回false，那么`Controller`不再会被调用，而是直接返回。如果你在Controller上面加了AOP进行接口调用统计的话，可能会漏掉。这个时候，我们不应该让Controller不调用，建议这种情况在**AOP里面对Controller的参数切面进行校验后，抛出统一的业务异常。**


# 五、参考资料：

- [Spring Validation 深入浅出](http://benweizhu.github.io/blog/2014/07/19/spring-validation-by-example/)
- [Spring Frameworl#Validation, Data Binding, and Type Conversion](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#validation)
- [All You Need To Know About Bean Validation With Spring Boot](https://reflectoring.io/bean-validation-with-spring-boot/)








