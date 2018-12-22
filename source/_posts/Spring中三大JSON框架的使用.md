---
layout: post
title: Spring中三大JSON框架的使用
date: '2018-12-16'
tags:
  - Java
  - Spring
  - 后台
  - Service
  - Json序列化
categories:
  - 技术
---

在SpringBoot中，常常会需要把请求中的参数进行反序列化，得到我们需要的实体对象，在进行处理之后，再把实体序列化返回给请求方（这里不提什么DTO、VO、BO的概念，其实很多公司对这些领域模型的区分都不怎么严格，毕竟搬砖的靠技术，大佬们才谈规范和标准，在[阿里巴巴的Java开发手册](https://github.com/alibaba/p3c/blob/master/%E9%98%BF%E9%87%8C%E5%B7%B4%E5%B7%B4Java%E5%BC%80%E5%8F%91%E6%89%8B%E5%86%8C%EF%BC%88%E8%AF%A6%E5%B0%BD%E7%89%88%EF%BC%89.pdf)中对这些有比较详细的规约，建议参考）。大部分情况下，开放API的数据协议都是用的JSON（也就是请求的**`content-type:application/json`，参数以json格式放在请求body中的**，本文只讨论这种请求类型），常见的JSON序列化工具有Jackson（SpringBoot默认使用）、Gson、FastJson等等。下面看一下怎么使用它们。

<!-- more -->

# 一、Jackson

## 1.1 基本用法

Jackson是SpringBoot默认的JSON序列化/反序列化框架，对于`content-type:application/json`的请求参数，只需要在Controller的方法中使用`@RequestBody`注解到参数上，即可实现反序列化。下面是基本的示例代码，`User.java`（必须有getter、setter方法和默认的无参构造函数）、`UserController.java`（方法的参数必须有`@RequestBody`注解）。

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

## 1.2 字段命名映射的处理

实际业务上，有些API要求参数会以**小写字母+下划线**的方式命名，有些又要求以**驼峰**方式命名，有时可能又为了某种兼容，参数的名字可能和代码中实体的属性名称完全不同。我们在上面的`User.java`中增加一个`private String personalPage`属性（相应地也添加了setter/getter方法），看看如何解析这个参数。

然后**构造一个新的请求，在这个请求中添加一个带有了小写字母加下划线的参数`personal_page`**，我们发现反序列化之后的`User`对象的`personalPage`属性为`null`。这很好理解，框架没有这么智能，它并不会知道`personalPage`属性在序列化的数据中是叫`personal_page`的。为了指定序列化时这个属性的名称，我们可以给它使用`@JsonProperty("personal_page")`注解，这个注解也是Jackson这个框架提供的。

通过`@JsonProperty`注解可以指定序列化的属性名称，但是每一个属性如果都要这么写的话，就太费劲了，我们可以统一指定这种命名的解析规则。具体做法有两种，一种是配置的方式，一种是编程的方式。

- `application.properties`配置文件中设置：`spring.jackson.property-naming-strategy=CAMEL_CASE_TO_LOWER_CASE_WITH_UNDERSCORES`（新版本好像已经用`SNAKE_CASE`替代了）。关于更多的配置参数，可以查看一下`org.springframework.boot.autoconfigure.jackson.JacksonProperties`这个类。配置文件里面有哪些Jackson的配置可以写，可以参考[这里](https://docs.spring.io/spring-boot/docs/current/reference/html/common-application-properties.html)。
- 编程[自定义`ObjectMapper`](https://docs.spring.io/spring-boot/docs/current/reference/html/howto-spring-mvc.html#howto-customize-the-jackson-objectmapper)：ObjectMapper是Jackson的核心，它可以控制SpringBoot中如何使用Jackson的功能，下面通过配置的方式指定了自定义的`ObjectMapper`。这种方式除了可以配置参数名称的解析规则外，还可以指定更多的Jackson序列化和反序列化的规则。

具体使用`ObjectMapper`的时候有两种方式：**一种是配置一个单例的`ObjectMapper`对象**，如下所示。

```java
// CustomObjectMapper.java
...
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

**另一种方式是以在`SpringBootApplication`类中指定`HttpMessageConverter`来实现同样的功能。**

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

配置文件的方式不赘述，`spring.jackson.date-format=`即可。

在1.2节提到了采用`ObjectMapper`可以设置参数解析的规则，时间类型的字段处理也是一种。具体方法是：

```java
// CustomObjectMapper.java
mapper.setDateFormat(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));

// User.java
public class User {
    // 省略其他字段
    private LocalDate birthDay;

    public LocalDate getBirthDay() {
        return birthDay;
    }

    public void setBirthDay(LocalDate birthDay) {
        this.birthDay = birthDay;
    }
}
```

这样，在反序列化接收参数时候，都只能接收形如`yyyy-MM-dd HH:mm:ss`的字符串了。并且在序列化返回结果的时候也是返回这个格式。当然你也可以使用`@JsonFormat(timezone = "GMT+8", pattern = "yyyy-MM-dd HH:mm:ss")`注解。不过有时为了让接口可以更灵活，可能我们的VO/DTO里面还是会把`XXXTime`定义成`String`，等到反序列化完成之后，再手动校验它们。

## 1.4 枚举类型的字段处理

这个更简单，啥也不用干，定义一个枚举`Gender`，Jackson框架自动会帮你反序列化解析成对象的属性。

```
// Gender.java
public enum Gender {
    MALE, FEMALE
}

// User.java
public class User2 {
    // 省略其他字段
    private Gender gender;
    // 省略getter/setter
}
```

当传入的参数不在枚举值范围的时候，框架会抛出异常，我们可以在全局的异常Handler里面处理它，然后向用户返回友好的信息。

如果不希望Jackson框架抛出异常的话，可以对自定义的`ObjectMapper`进行设置，`ObjectMapper.configure(DeserializationFeature.READ_UNKNOWN_ENUM_VALUES_AS_NULL, true);`。

## 1.5 Jackson序列化小结

总结下来，就是两种方式，一种靠注解，一种靠`CustomObjectMapper`，注解适合灵活用在某个属性上（灵活的意思是比如，某个属性名称完全没有任何规则，参数叫`a`，而属性叫`b`），如果整个工程都是统一的风格，那么最好用`CustomObjectMapper`来节省体力（个人也更推荐这种做法，节省体力，API风格规范统一，便于团队协作）。

两种方式在能力上其实都是对应的，最灵活的方式就是可以自己定义`serializer`和`deserializer`，具体如何实现，可以参考文档。不过其实Jackson已经提供了足够多的反序列化配置选项来给我们用，在[`DeserializationFeature.java`](https://fasterxml.github.io/jackson-databind/javadoc/2.8/index.html?com/fasterxml/jackson/databind/DeserializationFeature.html)里面我们可以找到诸如`READ_UNKNOWN_ENUM_VALUES_AS_NULL`这样的配置。

反序列化还可以和前一篇提到的[参数校验](http://unclechen.github.io/2018/12/15/SpringBoot%E8%87%AA%E5%AE%9A%E4%B9%89%E8%AF%B7%E6%B1%82%E5%8F%82%E6%95%B0%E6%A0%A1%E9%AA%8C/)结合起来进行使用，此时只需要将`@RequetsBody`注解前面加上一个`@Validated`注解即可。

# 二、Gson

除了默认的Jackson，还可以使用Gson来对参数进行反序列化，具体实现是通过`GsonHttpMessageConverter`。我们先看一下Spring里面的`HttpMessageConverter`继承关系，如下图所示。

<img src="https://ws1.sinaimg.cn/large/006tNbRwly1fydmth2md0j316y0peajm.jpg" width="480"/>

了解这个之后，看一下如何将SpringBoot的Json框架替换成`Gson`。

## 2.1 基本用法

第一步，引入`Gson`库。

```groovy
dependencies {
    implementation('org.springframework.boot:spring-boot-starter-web')
    implementation('com.google.code.gson:gson:2.8.4')
}

configurations {
    compile.exclude group: 'com.fasterxml.jackson.core'
}
```

这里强制剔除掉Jackson是为了防止SpringBoot的`AutoConfig`机制因为Jackson的类存在导致Gson的失效。Spring-AutoConfig使用的是`@ConditionalOnClass(Gson.class)`自动配置Gson，使用的是`@ConditionalOnClass(ObjectMapper.class)`自动配置的Jackson，由于`ObjectMapper.class`在`com.fasterxml.jackson.core`中，所以把它所在的库exclude掉。

第二步，回到`SpringBootApplication类`中，添加`GsonHttpMessageConverter`到Coverters。同时去掉Jackson的Converter和ObjectMapper（如果有的话）。

```
@SpringBootApplication
public class DemoApplication extends WebMvcConfigurationSupport {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }

    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        // Gson
        GsonHttpMessageConverter gsonHttpMessageConverter = new GsonHttpMessageConverter();
        converters.add(gsonHttpMessageConverter);
    }
}
```

除了代码方式，还可以写到配置文件。具体可以配置的属性有哪些，可以参考源代码`com.google.gson.FieldNamingPolicy.java`。

```
// application.properties
# Naming policy that should be applied to an object's field during serialization and deserialization.
spring.gson.field-naming-policy=LOWER_CASE_WITH_UNDERSCORES
```

> 配置这里，我始终都没有实践成功，虽然我确定已经排除掉了Jackson库，但是`spring.gson.field-naming-policy= LOWER_CASE_WITH_UNDERSCORES`不知道为什么一直都没有配置成功。希望熟悉的朋友可以指点一下。

## 2.2 字段命名映射的处理


上面只是很简单的添加了`Gson`作为Json框架，使用`GsonBuilder`创建Gson对象，可以对序列化的映射规则进行配置，代码如下：

```
    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        // Gson
        GsonHttpMessageConverter gsonHttpMessageConverter = new GsonHttpMessageConverter();
        // 自定义Gson，配置参数解析规则
        gsonHttpMessageConverter.setGson(new GsonBuilder()
                .setFieldNamingPolicy(FieldNamingPolicy.LOWER_CASE_WITH_UNDERSCORES)
                .create());
        converters.add(gsonHttpMessageConverter);
    }
```

除了这种方式，还可以在SpringBoot的配置文件中配置Gson的Mapper规则。

```
// application.properties
# Preferred JSON mapper to use for HTTP message conversion.
spring.http.converters.preferred-json-mapper=gson

```

## 2.3 时间参数的处理

既然都是用`GsonBuilder`，那么很简单加一行`.setDateFormat("yyyy-MM-dd HH:mm:ss")`即可。

## 2.4 枚举参数的处理

和Jackson一样，Gson也不需要我们干什么就会自动将字符串形式的参数转换成枚举对象。


# 三、FastJson

## 3.1 基本用法

阿里出产的轮子，官方有wiki-[《在 Spring 中集成 Fastjson》](https://github.com/alibaba/fastjson/wiki/%E5%9C%A8-Spring-%E4%B8%AD%E9%9B%86%E6%88%90-Fastjson)，这里我就不搬运了。本质上也是一样的，替换SpringBoot自带的`HttpMessageConverter`为`FastJsonHttpMessageConverter`即可。

在官方文档中提到一句：

> SpringBoot 2.0.1版本中加载WebMvcConfigurer的顺序发生了变动，故需使用converters.add(0, converter);指定FastJsonHttpMessageConverter在converters内的顺序，否则在SpringBoot 2.0.1及之后的版本中将优先使用Jackson处理。详情：[WebMvcConfigurer is overridden by WebMvcAutoConfiguration #12389](https://github.com/spring-projects/spring-boot/issues/12389)

## 3.2 配置映射规则

```
@SpringBootApplication
public class DemoApplication extends WebMvcConfigurationSupport {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }

    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        FastJsonHttpMessageConverter converter = new FastJsonHttpMessageConverter();
        //自定义配置...
        FastJsonConfig config = new FastJsonConfig(); // 按照阿里的文档说，生产环境要尽可能单例配置对象
        config.setDateFormat("yyyy-MM-dd HH:mm:ss"); // 设置时间格式
        // 字段名称映射规则，详见：https://github.com/alibaba/fastjson/wiki/PropertyNamingStrategy_cn
        config.getParserConfig().propertyNamingStrategy = PropertyNamingStrategy.SnakeCase; // 设置反序列化名称规则
        config.getSerializeConfig().propertyNamingStrategy = PropertyNamingStrategy.SnakeCase; // 设置序列化名称规则
        converter.setFastJsonConfig(config);
        // 添加下面这一段是因为高版本的Fastjson的MediaType默认为*/*，而高版本的Spring不允许这样，需要明确指定MediaType。详见：https://github.com/alibaba/fastjson/issues/1554
        List<MediaType> supportedMediaTypes = new ArrayList<>();
        supportedMediaTypes.add(MediaType.APPLICATION_JSON);
        supportedMediaTypes.add(MediaType.APPLICATION_JSON_UTF8);
        supportedMediaTypes.add(MediaType.APPLICATION_ATOM_XML);
        supportedMediaTypes.add(MediaType.APPLICATION_FORM_URLENCODED);
        supportedMediaTypes.add(MediaType.APPLICATION_OCTET_STREAM);
        supportedMediaTypes.add(MediaType.APPLICATION_PDF);
        supportedMediaTypes.add(MediaType.APPLICATION_RSS_XML);
        supportedMediaTypes.add(MediaType.APPLICATION_XHTML_XML);
        supportedMediaTypes.add(MediaType.APPLICATION_XML);
        supportedMediaTypes.add(MediaType.IMAGE_GIF);
        supportedMediaTypes.add(MediaType.IMAGE_JPEG);
        supportedMediaTypes.add(MediaType.IMAGE_PNG);
        supportedMediaTypes.add(MediaType.TEXT_EVENT_STREAM);
        supportedMediaTypes.add(MediaType.TEXT_HTML);
        supportedMediaTypes.add(MediaType.TEXT_MARKDOWN);
        supportedMediaTypes.add(MediaType.TEXT_PLAIN);
        supportedMediaTypes.add(MediaType.TEXT_XML);
        converter.setSupportedMediaTypes(supportedMediaTypes);
        converters.add(0, converter); // 加入到HttpMessageConverterList的最前面
    }
}
```


# 四、性能

这里我也没有做过真正的测试，网上随便一搜可以找到大量讨论，我个人的态度比较开放，觉得在功能、性能上，三个框架都能满足基本使用，喜欢使用最简洁的方式。不过比较早的时候我开发Android App用的都是Gson，猜测可能是Gson这个库比较精简，没有太多复杂的功能，不过后来fastjson也针对Android出了专门的版本。到现在，Kotlin出来了，大家可能又有适配Kotlin的需求，具体还是看各自的需要来选择吧。

# 五、总结

三大框架各自有自己的默认配置（实际上默认配置可能是随着框架的版本而变化的），使用框架的时候要熟悉它们，才可以避免一些麻烦。

- 比如Jackson，默认情况下，对请求中多余的属性，会反序列化失败，需要把`DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES`设置为`false`才可以反序列化请求参数。
- 比如Gson，默认会忽略多余的属性。但是对java8里面的LocalDate类型的属性，反序列化需要手动编写`deserialize`方法，因为Gson默认只能处理`java.util.Date`类型的属性。

最后对于怎么使用框架，如果写配置能满足，那是最简单的、最方便管理的。[common-application-properties](https://docs.spring.io/spring-boot/docs/current/reference/html/common-application-properties.html)里面预制了大量的配置，但是有时配置可能不够，这时还是编程方式来的更好一些。总之，我建议统一使用一种方式，保证整个项目风格的统一最好。







