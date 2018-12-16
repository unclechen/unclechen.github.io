---
layout: post
title: SpringBoot中JSON的序列化和反序列化
date: '2018-12-16'
tags:
  - Java
  - SpringBoot
  - 后台
  - Service
categories:
  - 技术
---

在SpringBoot中，常常会需要把请求中的参数进行反序列化，得到我们需要的实体对象，在进行处理之后，再把实体序列化返回给请求方（这里不提什么DTO、VO、BO的概念，其实很多公司对这些领域模型的区分都不怎么严格，毕竟搬砖的靠技术，大佬们才谈规范和标准。在[阿里巴巴的Java开发手册](https://github.com/alibaba/p3c/blob/master/%E9%98%BF%E9%87%8C%E5%B7%B4%E5%B7%B4Java%E5%BC%80%E5%8F%91%E6%89%8B%E5%86%8C%EF%BC%88%E8%AF%A6%E5%B0%BD%E7%89%88%EF%BC%89.pdf)中对这些有比较详细的规约，建议参考）。大部分情况下，开放API的数据协议都是用的JSON（也就是请求的**`content-type:application/json`**，这篇文章只讨论这种请求类型），常见的JSON序列化工具有Jackson（SpringBoot默认使用）、Gson、FastJson等等。下面看一下怎么使用它们。

<!-- more -->

# 一、Jackson

## 1.1 基本用法

Jackson是SpringBoot默认的JSON序列化/反序列化框架，对于`content-type:application/json`的请求参数，需要在Controller的方法中使用`@RequestBody`来接收它。下面是基本的示例代码，`User.java`（必须有getter、setter方法和默认的无参构造函数）、`UserController.java`（方法的参数必须有`@RequestBody`注解）。

```
// User.java
public class User2 {
    private Integer id;
    private String name;

    public Integer getId() {
        return id;
    }

    public void setId(Integer id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}

// UserController.java
@RestController
public class UserController {

    private static final Logger LOGGER = LoggerFactory.getLogger(UserController.class);

    @RequestMapping(value = "/user/post", method = RequestMethod.POST)
    public ServiceResponse getUserByPost(@RequestBody User user) {
        LOGGER.info(user.getId() + ": " + user.getName());
        ServiceResponse serviceResponse = new ServiceResponse();
        serviceResponse.setCode(0);
        serviceResponse.setMessage("test");
        return serviceResponse;
    }
}

// ServiceResponse.java，作为getUserByPost的返回值，自动被序列化返回。
public class ServiceResponse {

    private int code;
    private String message;

    public int getCode() {
        return code;
    }

    public void setCode(int code) {
        this.code = code;
    }

    public String getMessage() {
        return message;
    }

    public void setMessage(String message) {
        this.message = message;
    }
}
```

用postman发出一个请求试试。

<img src="https://ws2.sinaimg.cn/large/006tNbRwly1fy8x6ahy8hj30z40scq5q.jpg" width="400">

喜欢用`curl`的话，也可以用下面的命令发请求。

```
curl -X POST \
  http://127.0.0.1:8080/user/post \
  -H 'Content-Type: application/json' \
  -H 'cache-control: no-cache' \
  -d '{
	"id": 1,
	"name": "unclechen"
}'
```

返回结果如下，可以看到`ServiceResponse`对象，自动被框架序列化返回。

```
{
    "code": 0,
    "message": "test"
}
```

由此看到，作为SpringBoot中默认使用的JSON框架，基本上不用做任何事情，就可以实现json请求的参数反序列化以及处理结果的序列化。

> 注意：虽然不用做什么事情，但是也不能做多余的事。比如对于User.java类，一定要保证它会有一个无参数的构造函数。按照Java的规则，如果你没有指定任何构造函数，那么编译时会自动添加一个无参构造函数，但是如果你指定了含有参数的构造函数，那么就不会再添加一个无参构造函数了，此时反序列化就会失败，抛出`com.fasterxml.jackson.databind.exc.InvalidDefinitionException`。

## 1.2 字段命名方式的处理

实际业务上，有些API要求参数会以**小写字母+下划线**的方式命名，有些又要求以**驼峰**方式命名，有时可能又为了某种兼容，参数的名字可能和代码中实体的属性名称完全不同。我们在上面的`User.java`中增加一个`private String personalPage`属性（相应地也添加了setter/getter方法），看看如何解析这个参数。

**还是使用原来的请求，没有带有新的参数`personalPage`**。我们在代码中用Log打印反序列化之后的结果（直接用debug更方便），发现反序列化之后的`User`对象的`personalPage`属性为`null`。

接着**使用新的请求，请求中带有了小写字母加下划线的参数`personal_page`**，我们发现反序列化之后的`User`对象的`personalPage`属性仍然为`null`。其实这很好理解，框架没有这么智能，它不会知道`personalPage`属性在序列化的数据中是叫`personal_page`的。为了指定序列化时这个属性的名称，我们可以给它使用`@JsonProperty("personal_page")`注解，这个注解也是Jackson这个框架提供的。

通过`@JsonProperty`注解可以指定序列化的属性名称，但是每一个属性如果都要这么写的话，就太费劲了，我们可以统一指定这种命名的解析规则。具体做法有两种：

- `application.properties`中配置一个：`spring.jackson.property-naming-strategy=CAMEL_CASE_TO_LOWER_CASE_WITH_UNDERSCORES`（新版本好像已经用`SNAKE_CASE`替代了）。关于更多的配置参数，可以查看一下`org.springframework.boot.autoconfigure.jackson.JacksonProperties`这个类。
- [自定义`ObjectMapper`](https://docs.spring.io/spring-boot/docs/current/reference/html/howto-spring-mvc.html#howto-customize-the-jackson-objectmapper)：ObjectMapper是Jackson的核心，它可以控制SpringBoot中如何使用Jackson的功能，下面通过配置的方式指定了自定义的`ObjectMapper`。通过这种方式，我们还可以指定更多的Jackson序列化和反序列化行为，后面会提到。

```java
@Configuration
public class CustomObjectMapper {

    @Bean
    public ObjectMapper getObjectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        // 指定PropertyNamingStrategy，甚至可以自定义PropertyNamingStrategy
        mapper.setPropertyNamingStrategy(PropertyNamingStrategy.SNAKE_CASE);
        return mapper;
    }
}
```

除了配置一个单例`Bean`的方法，也可以在`SpringBootApplication`类中指定`HttpMessageConverter`来实现同样的功能。

```java
@SpringBootApplication
public class DemoApplication extends WebMvcConfigurationSupport {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }

    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        Jackson2ObjectMapperBuilder builder = new Jackson2ObjectMapperBuilder();
        builder.propertyNamingStrategy(PropertyNamingStrategy.SNAKE_CASE);
        converters.add(new MappingJackson2HttpMessageConverter(builder.build()));
    }
}

```


## 1.3 时间类型的字段处理

## 1.4 枚举类型的字段处理

## 1.5 多个对象

有的时候，我们会遵循一些标准或者规范，因此API的参数命名

# 二、Gson

# 三、FastJson

# 四、特定用法

## 4.1 指定序列化的名字

使用Jackson的注解@JsonProperty可以设置序列化和反序列化时的JSON名

## 时间类型的参数

## 枚举类型的参数

## 4.2 多个对象

## 4.3 增加校验

# 五、性能







