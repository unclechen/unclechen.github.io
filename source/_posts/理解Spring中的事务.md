---
layout: post
title: 理解Spring中的事务
date: '2019-03-08'
tags:
  - Spring
  - 后台
  - Service
categories:
  - 技术
---



Spring为事务管理提供了丰富的支持，对于底层不同的事务（ 如Java Transaction API (JTA), JDBC, Hibernate, Java Persistence API (JPA), and Java Data Objects (JDO)）管理提供了统一的抽象编程模型，而且它的API也非常易于开发者理解和使用。



Spring事务管理分为编程式和声明式的两种。编程式事务指的是通过编码方式实现事务；声明式事务基于AOP，将具体业务逻辑与事务处理解耦。声明式事务对我们的业务代码无侵入，一般更倾向于选择它。而编程式的事务一般使用`TransactionTemplate`。本文主要介绍常用的声明式事务。

<!--more-->


## 1.Transactional注解基本用法



### 1.1 配置数据源和事务管理器



配置数据源可以采用xml配置，也可以在代码中。下面是一段代码配置示例。



```java

...

@Configuration
@MapperScan(basePackages = "com.xxx.dao",
    sqlSessionFactoryRef = DataSourceConfiguration.SQL_SESSION_FACTORY_NAME)
public class DataSourceConfiguration {
  public static final String DATASOURCE_NAME = "xxxDataSource";
  public static final String SQL_SESSION_FACTORY_NAME = "xxxProductSessionFactory";
  public static final String TRANSACTION_MANAGER_NAME = "xxxProductTransaction";


  @Bean(name = DataSourceConfiguration.DATASOURCE_NAME)

  public DataSource getDataSource() {
    DataSourceProperties dataSourceProperties = getDataSourceProperties();
    return dataSourceProperties.initializeDataSourceBuilder().build();
  }


  @Bean
  @ConfigurationProperties("spring.datasource.xxx") 

  // 数据源配置写在properties配置文件中，以spring.datasource开头，借助ConfigurationProperties注解来读取
  public DataSourceProperties getDataSourceProperties() {
    return new DataSourceProperties();
  }



  @Bean(name = DataSourceConfiguration.TRANSACTION_MANAGER_NAME)
  public PlatformTransactionManager transactionManager(
      @Named(DataSourceConfiguration.DATASOURCE_NAME) final DataSource dataSource) {
    return new DataSourceTransactionManager(dataSource);
  }
}

```



### 1.2 给需要使用事务的地方添加注解



```java

@Service
public class XXXServiceImpl implements XXXService {

  ...

  @Override
  @Transactional(transactionManager = DataSourceConfiguration.TRANSACTION_MANAGER_NAME)
  public void foo() {
}

```



如果项目中只有一个事务管理器，可以不用设置注解的`transactionManager`属性，关于注解属性的详细介绍在第3节。



虽然使用注解方式来实现事务管理看起来非常简单，但是如果对Spring注解实现事务的原理不够理解的话，在使用事务出现错误的时候，很难理解、排查问题。



## 2. Transactional注解实现的原理



- 对于使用了`Transactional`注解的方法的类，Spring AOP代理会在运行时生成这个类的代理对象。

- 当这个对象运行这个注解方法时，会读取`@Transactional`注解里面配置的信息，决定该方法是否要使用`TransactionInterceptor`拦截器来拦截。

- 当拦截器拦截该方法时，会在调用该方法之前，创建事务，并在这个事务中执行该方法，最后根据执行是否有异常使用**抽象事务管理器（AbstractPlatformTransactionManager）**操作数据源提交或者回滚这个事务。下面是一张常见的Spring注解事务实现机制图。



<img src="https://ws1.sinaimg.cn/large/006tKfTcly1g0qutyc3mnj30th0j20un.jpg" width="800" />



Spring AOP代理有两种实现，`Cglib`和`JDK Proxy`。对于实现了接口的类，默认走`JDK代理`，对于未实现接口的类，默认走`Cglib`。Spring中事务管理的框架由`AbstractPlatformTransactionManager`提供，具体Manager的实现取决于操作的数据源，比如数据源是数据库时，就会使用`DataSourceTransactionManager`管理JDBC的Connection。下图是`AbstractPlatformTransactionManager`具体实现类关系。



<img src="https://ws2.sinaimg.cn/large/006tKfTcly1g0quuh50khj30lo0c1dhf.jpg" width="600" />



## 3. Transactional注解提供的属性



除了基本用法之外，还应该知道它提供了哪些属性供开发者设置，以及这些属性如何在事务管理时发挥作用。



```java

@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Transactional {
    @AliasFor("transactionManager")
    String value() default "";
     
    @AliasFor("value")
    String transactionManager() default "";
     
    Propagation propagation() default Propagation.REQUIRED;
     
    Isolation isolation() default Isolation.DEFAULT;
     
    int timeout() default TransactionDefinition.TIMEOUT_DEFAULT;
     
    boolean readOnly() default false;
     
    Class<? extends Throwable>[] rollbackFor() default {};
     
    String[] rollbackForClassName() default {};
     
    Class<? extends Throwable>[] noRollbackFor() default {};
     
    String[] noRollbackForClassName() default {};
}

```



- `name`、`transactionManager`：当项目中有多个事务管理器时，选择使用哪一个

- `propagation`：事务的传播行为，默认为REQUIRED。这里需要开发者有更多了解，参考[官方API文档](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/transaction/annotation/Propagation.html)。

- `isolation`：事务的隔离级别，默认DEFAULT，即采用数据服务器设置的隔离级别。

- `timeout`：超时时间，默认-1

- `readOnly`：是否为只读事务，默认false；对于一些不需要事务的只读操作，可以设置为true

- `rollbackFor`、`rollbackForClassName`：指定触发回滚的异常类型

- `noRollbackFor`、`noRollbackForClassName`：指定不触发混滚的异常类型



`@Transactional`注解也可以添加到类级别上。当把`@Transactional`注解放在类级别时，表示所有该类的公共方法都配置相同的事务属性信息。此时如果方法级别也加了注解，会覆盖类级别的注解。



## 4. Spring事务失效的常见原因



了解Spring注解实现事务的基本原理，看下一些比较容易出现的事务失败的原因。



### 4.1 内部方法调用失效



Spring是通过AOP代理的方式来识别方法上面的注解，如果是同一个类中一个方法A调用另一个注解方法B，将无法被代理从而导致事务失效。因此方法A上面没有注解，代理对象在执行这个方法时，不会给A方法添加事务，所以被调用的B方法也不会运行在事务里面了。



### 4.2 `private`方法失效



这个意思是说只有给一个`public`方法加`Transactional`注解。Spring才会给它进行事务管理。原因是`TransactionInterceptor`拦截器时，会间接调用到一个`AbstractFallbackTransactionAttributeSource`里面的`computeTransactionAttribute`方法，通过这个方法来回去注解上的各个属性，同时会判断该方法是否为`public`，若不是public则不会读取注解属性，返回null。



```java

protected TransactionAttribute computeTransactionAttribute(Method method, Class<?> targetClass) {
  // Don't allow no-public methods as required.
  if (allowPublicMethodsOnly() && !Modifier.isPublic(method.getModifiers())) {

     return null;

  }

...

}

```


### 4.3 多线程切换导致事务失效



Spring会把很多一个事务中使用的很多变量放到很多`ThreadLocal`中，并保存在一个`org.springframework.transaction.support.TransactionSynchronizationManager`类中。如果在一个线程中启动了一个事务，Spring首先会初始化设置这些TheadLocal的值（`actualTransactionActive=true`），并把事务里这些`ThreadLocal`通过`TransactionSynchronizationManager`与这个线程绑定了。那么假设此时切换到另一个线程中，另一个线程将无法获得这些`ThreadLocal`，也无法知道当前是不是在一个事务里面，更无法知道这个事务做了哪些配置。所以切换线程之后，事务就无法管理了。



```java

public abstract class TransactionSynchronizationManager {
  private stati final Log logger = LogFactory.getLog(TransactionSynchronizationManager.class);
  private static final ThreadLocal<Map<Object, Object>> resources = new NamedThreadLocal("Transactional resources");
  private static final ThreadLocal<Set<TransactionSynchronization>> synchronizations = new NamedThreadLocal("Transaction synchronizations");
  private static final ThreadLocal<String> currentTransactionName = new NamedThreadLocal("Current transaction name");
  private static final ThreadLocal<Boolean> currentTransactionReadOnly = new NamedThreadLocal("Current transaction read-only status");
  private static final ThreadLocal<Integer> currentTransactionIsolationLevel = new NamedThreadLocal("Current transaction isolation level");
  private static final ThreadLocal<Boolean> actualTransactionActive = new NamedThreadLocal("Actual transaction active");
}

```



了解了切换线程导致的事务失效原因后，是不是觉得可以做点什么，让事务延续呢？当然是可以的，如果我们手动地给另一个线程也设置这些ThreadLocal，就可以将事务延续了。



### 4.4 单元测试中注解的方法自动回滚失效



有时候我们会给单元测试里面的某个testFunction加上`@Transactional`注解，希望这个方法运行完成之后，可以自动回滚，不污染数据源。但是有时回滚会失效。



先看下官方文档中，[测试章节](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-testing.html)提到的问题。



> If your test is `@Transactional`, it rolls back the transaction at the end of each test method by default. However, as using this arrangement with either RANDOM_PORT or DEFINED_PORT implicitly provides a real servlet environment, the HTTP client and server run in separate threads and, thus, in separate transactions. Any transaction initiated on the server does not roll back in this case.



通常情况下，`@SpringBootTest`注解里面的`webEnvironment`默认值是`MOCK`，它在运行单测时，不会真正启动一个Server来进行测试。但是如果我们手动将这个值改为`RANDOM_PORT/DEFINED_PORT`后，会启动一个真实的Web环境，内置的Server会启动起来。然后此时发出HTTP请求的Client和处理请求的Server试运行在不同的线程里面的，按照3.3的分析，这里的事务肯定就无法自动生效了，单元测试方法上面的`@Transactional`注解就是失效了。



所以如果你不需要启动真实的Server时，建议使用`@SpringBootTest`默认的`webEnvironment`，此时请求的client只能用`MockMvc`来发，而不是`TestRestTemplate`，这样就可以让单测跑完之后自动回滚了。



## 5. 参考资料


- [https://www.ibm.com/developerworks/cn/java/j-master-spring-transactional-use/index.html](https://www.ibm.com/developerworks/cn/java/j-master-spring-transactional-use/index.html)
- [https://dzone.com/articles/spring-transaction-management-over-multiple-thread-1](https://dzone.com/articles/spring-transaction-management-over-multiple-thread-1)
- [https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-testing.html](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-testing.html)













