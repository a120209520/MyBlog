---
layout: post
title: Spring-IOC&AOP总结——JavaConfig方式
date:   2018-10-16 7:32
comments: true
description: 根据《Spring实战第4版》整理JavaConfig方式配置Spring的IOC&AOP
tags:
- java
- spring
---


# 1. 装配Bean

## 1.1 装配方案

### (1) 对于自己的类
&emsp;&emsp;建议使用`@Component`定义Bean，`@Autowired`及其他注解引用Bean

### (2) 对于第三方库
&emsp;&emsp;只能采用`JavaConfig`创建Bean，`@Autowired`及其他注解引用Bean

<hr>

## 1.2 单元测试
>Test.java
{:.filename}
{% highlight java linenos%}
@RunWith(SrpingJUnit4ClassRunner.class)
@ContextConfiguration(classes=JavaConfigNamexxx.class)
public class Tester {
    @Test
    public void test01() {...}
}
{% endhighlight %}

<hr>

## 1.3 自动装配 ———— 用于自己的类
### (1) @Component
{% highlight java linenos%}
//@Component代表我是一个Bean，扫描到我的时候请创建我
@Component              //默认id是player
@Component("player01")  //指定一个id
public class Player {...} 
}
{% endhighlight %}
### (2) 定义JavaConfig  (代表从哪些包中装配Bean)
{% highlight java linenos%}
@Configuration
@ComponentScan  //这里默认扫描和JavaConfig同一个包下的
@ComponentScan(basePackages="xxx.xxx.xxx")
{% endhighlight %}
### (3) 或者定义XML (JavaConfig的另一种写法)
{% highlight xml linenos%}
<context:component-scan base-package="xxx.xxx.xxx">
{% endhighlight %}
### (4) @Autowired

- `@Autowired`的两个作用范围

>作用范围————域
{:.filename}
{% highlight java linenos%}
@Autowired
private User user
{% endhighlight %}

>作用范围————任何方法的入参
{:.filename}
{% highlight java linenos%}
@Autowired
public void fund(User user) {...}
{% endhighlight %}

- 注解参数

>注解参数————required
{:.filename}
{% highlight java linenos%}
@Autowired(required=false) //如果装配失败，则传入null
{% endhighlight %}

<hr>

## 1.4 手动装配 ———— 用于第三方库
### (1) @Bean
&emsp;&emsp;`@Bean`是`JavaConfig`中直接定义Bean的注解，相当于XML中的<bean .../>
- 作用范围

&emsp;&emsp;`@Bean`作用在任何`JavaConfig`的方法上，创建其返回值类型的Bean<br>
注意，仅在系统启动时调用一次方法，后续使用时将直接使用Context中的对象

- BeanID

>BeanID
{:.filename}
{% highlight java linenos%}
@Bean  //默认id是方法名
@Bean(name="createBean") //指定一个id
{% endhighlight %}

- Bean之间的引用

>Bean之间的引用————入参引用
{:.filename}
{% highlight java linenos%}
    //例如构造一个CDPlayer对象，想通过入参嵌入id为cd2的CD对象(相当于XML中的ref='cd2')
    @Bean(name="playerB")
    public CDPlayer getCDPlayerB(CD cd2) {
        return new CDPlayerB(cd2);
    }
{% endhighlight %}

>Bean之间的引用————调用JavaConfig的方法引用
{:.filename}
{% highlight java linenos%}
    //例如构造一个CDPlayer对象，想通过入参嵌入id为cd2的CD对象(相当于XML中的ref='cd2')
    @Bean(name="playerB")
    public CDPlayer getCDPlayerB() {
        return new CDPlayerB(cd2());
    }
{% endhighlight %}

- 初始化/摧毁对象方法

{% highlight java linenos%}
@Bean(destroyMethod="close")
@PostConstruct  //对应init-method
@PreDestroy     //对应destroy-method
{% endhighlight %}

<hr>

## 1.6 JavaConfig和XML的组合
{% highlight java linenos%}
@Import({CDConfig.class, CDPlayerConfig.class})      //JavaConfig中导入JavaConfig
@ImportResource("classpath:application.xml")         //JavaConfig中导入xml
<import resource="classpath:application-redis.xml"/> //xml中导入xml
<bean class="org.ppl.xxx.PlayerConfig"/>             //xml中导入JavaConfig
{% endhighlight %}

<hr>

# 2. 高级装配Bean

## 2.1 profile

### (1) JavaConfig的bean定义profile
{% highlight java linenos%}
@Profile("dev")
{% endhighlight %}
### (2) XML的bean定义profile
{% highlight xml linenos%}
<beans ... ... profile="dev"/>
{% endhighlight %}
### (3) web.xml中 激活profile (Spring实战——Page73)
>两个参数
{:.filename}
{% highlight java linenos%}
spring.profiles.active //优先级最高
spring.profile.default  //如果没有定义active则使用default
{% endhighlight %}
>web.xml中激活方法
{:.filename}
{% highlight xml linenos%}
<context-param>
    <param-name>spring.profiles.active</param-name>
    <param-value>dev</param-value>
</context-param>
{% endhighlight %}
### (4) 单元测试中 激活profile
{% highlight java linenos%}
@ActiveProfiles("dev")
{% endhighlight %}

<hr>

## 2.2 条件化Bean

### (1) @Conditional注解 (决定是否创建Bean)
{% highlight java linenos%}
//@Conditional放在@Bean下面:
@Bean
@Conditional(MagicCondition.class)
public MagicBean magicBean(){return new MagicBean();}
{% endhighlight %}
### (2) Condition接口

&emsp;&emsp;需要实现
{% highlight java linenos%}
//能够获取bean的定义，bean是否存在，bean的属性，环境变量，ResourceLoder，ClassLoader，检查有没有其他注解
matches(ConditionContext context, AnnotatedTypeMetadata data)
{% endhighlight %}  
&emsp;&emsp;详见 (Spring实战————Page 77)<br>
&emsp;&emsp;Spring4.0以后使用@Conditional和Condition来实现@Profile

<hr>

## 2.3 解决@Autowired的歧义性

### (1) 标记首选bean

&emsp;&emsp;`@Primary`可以放到`@Component`或者`@Bean`注解下面
针对XML方式，使用<bean .... primary="true"/>
### (2) 指定@Autowired的限定: @Qualifier("beanID")

&emsp;&emsp;`@Qualifier`放到`@Autowired`下面，表示引用指定限定名的bean，默认是beanID<br>
&emsp;&emsp;`@Qualifier`放到`@Component`或者`@Bean`下面，表示指定限定名，此时可以和beanID无关
### (3) 还可以自定义@Qualifier注解进行组合使用，但是一般用得很少

<hr>

## 2.4 Bean的作用域

### (1) @Scope
&emsp;&emsp;放在`@Bean`或者`@Component`下面

### (2) Bean的四种作用域
>JavaConfig方式
{:.filename}
{% highlight java linenos%}
@Scope(ConfigurableBeanFactory.SCOPE_SINGLETON) //单例
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE) //多例，每次@Autowired或者context.get()时会重新创建一个
@Scope(WebApplicationContext.SCOPE_SESSION)   //每个Session创建一个
@Scope(WebApplicationContext.SCOPE_REQUEST)   //每个Request创建一个
{% endhighlight %}
>xml方式
{:.filename}
{% highlight xml linenos%}
<bean .... scope="prototype"/>
{% endhighlight %}

### (3) 关于BeanID和作用域
&emsp;&emsp;注意！作用域仅针对同一`BeanID`而言！<br>
&emsp;&emsp;对于不同`BeanID`或者`Qualifier`，属于不同对象。<br>
&emsp;&emsp;而针对同一`BeanID`或者`Qualifier`，才分`Scope`:<br>
- 对于SINGLETON，`@Autowired`的是同一个对象
- 对于PROTOTYPE，每次`@Autowired`的是不同对象

### (4) 关于SCOPE_SESSION和SCOPE_REQUEST
&emsp;&emsp;例如购物车系统，需要在单例的`Service`对象中针对不同用户注入独立的`Cart`对象，那么需要这样实现:
{% highlight java linenos%}
@Component
@Scope(
    value=WebApplicationContext.SCOPE_SESSION,
    proxyMode=ScopedProxy.INTERFACES)
public Cart cart() {...}
{% endhighlight %}
&emsp;&emsp;其中`proxyMode`指的是采用代理模式，此时`@Autowired`其实是个代理对象，并且`ScopedProxy.INTERFACES`表示使用的是`JDK代理` <br>
&emsp;&emsp;PS: xml配置方法见 Spring实战————Page 88

<hr>

## 2.5 运行时注入

### (1) 读取properties文件————Enviroment
{% highlight java linenos%}
@Configuration
@PropertySource("classpath:user.properties")
public class UserConfig {

    @Autowired
    private Environment env;

    @Bean
    public User getUser() {
        return new User(
                env.getProperty("user.name"),
                env.getProperty("user.pass")
        );
    }
}

//getProperty(): 4种重载，可以提供任意类型返回值以及默认值
//getRequiredProperty(): 功能一样，不同在于获取不到值时会抛异常
//containsProperty(): 检查是否存在值
{% endhighlight %}

### (2) 读取properties文件————属性占位符和@Value
{% highlight java linenos%}
@Configuration
@PropertySource("classpath:user.properties")
public class UserConfig {

    @Bean
    public static PropertySourcesPlaceholderConfigurer placeholderConfigurer() {
        return new PropertySourcesPlaceholderConfigurer();
    }

    @Bean
    public User getUser(
            @Value("${user.name}") String name,
            @Value("${user.pass}") String pass ) {
        return new User(name, pass);
    }
}
{% endhighlight %}

### (3) 读取properties文件————SpEL表达式
- SpEL表达式
{% highlight text linenos%}
(1) #{1} 或者 #{'Hello'}  //直接取值
(2) #{T(System).currentTimeMillis()}   //调用static方法，只会调用一次
(3) #{beanID.name}  //调用其他Bean的域，如果是public直接调用域，如果是private则调用getter
(4) #{beanID?.name}  //安全调用；如果不是null才调用，否则返回null
(5) #{ 2*T(java.lang.Math).PI^2 } //2π²
(6) #{ beanID.name + 'hello' } //字符串拼接
(7) #{ beanID.name == 'ppl' }  //逻辑判断
(8) #{ beanID.age > 20 ? "old" : "young" } //三目运算
(9) #{ beanID.name ?: 'DefaultName' } //如果beanID.name是null，就输出DefaultName
(10) #{ admin.email mathches '\w*@\w*.com' } //判断是否符合正则表达式
(11) #{ beanID.list[2].name } //操作集合
(12) #{ 'Hello'[2] } //取字符串中的字符
(13) #{ beanID.list.?[age > 10] }  //筛选子集合
(14) #{ beanID.list.?[age > 10].![name] } //把age>10的元素的name抽取成一个String集合
还有其他功能，比如筛选集合中第一个或最后一个某种元素，不详述
{% endhighlight %}

- 调用SpEL表达式

>Java方式
{:.filename}
{% highlight java linenos%}
@Value("#{beanID.name}")
{% endhighlight %}
>xml方式
{:.filename}
{% highlight xml linenos%}
<!--XML中调用方法比较麻烦，见Spring实战————Page 94 -->
{% endhighlight %}

<hr>

# 3. Spring的AOP

## 3.1 AOP术语

{% highlight text linenos%}
(1) 通知 advice: 需要增强的功能，比如给某个类的某个方法添加日志，其中添加日志的操作称作advice
5种advice类型: Before, After, After-returning, After-throwing, Around,
(2) 连接点 join point: 要增强的类的所有可以增强的地方
(3) 切点 pointcut: 要增强的类实际被增强的地方(join point的子集)
(4) 切面 aspect: 整个增强的逻辑(不太严谨，简单理解)
总结: pointcut就是定义了aspect中哪些join point会得到advice
{% endhighlight %}

<hr>

## 3.2 切点指示器和切点表达式

### (1) 常用切点指示器
{% highlight text linenos%}
(1) execution()  //切某些方法
(2) within()     //切包内所有方法
(3) bean()       //切某些beanID (Spring自定义的指示器)
{% endhighlight %}
### (2) 切点表达式例子
{% highlight text linenos%}
(1) execution(* org.ppl.MyService.apply(..))  //切MyService类的apply方法，任意返回值，任意参数
(2) execution(* *.*.MyService.apply(..)) and within(org.ppl.*) 
    //只切org.ppl包下的apply方法
(3) execution(* org.ppl.MyService.apply(..)) and bean('myservice01')
    //只切beanID为myservice01的Myservice对象的apply方法
其中*表示任意类型的参数，而..表示任意类型参数且参数个数不限
{% endhighlight %}
### (3) 定义aspect
{% highlight java linenos%}
@Aspect
public class Audience {
    @Pointcut("execution(* org.ppl.frame.aop.Audience.perform(..))")
    public void perform(){}

    @Before("perform()")
    public void silenceCellPhone() {
        System.out.println("Silencing cell phones");
    }
    @Before("perform()")
    public void takeSeats() {
        System.out.println("Taking seats");
    }
    @AfterReturning("perform()")
    public void applause() {
        System.out.println("CLAP CLAP CLAP!!");
    }
    @AfterReturning("perform()")
    public void demandRefund() {
        System.out.println("Demanding a refund");
    }
    //Around非常重要，可以自定义处理模式，比如失败后多次重试
    @Around("perform()")
    public void around(ProceedingJoinPoint jp) {
        try {
            System.out.println("Silencing cell phones");
            System.out.println("Taking seats");
            jp.proceed();
            System.out.println("CLAP CLAP CLAP!!");
        } catch (Throwable e) {
            System.out.println("Demanding a refund");
        }
    }

    //还可以嵌入原始类的调用参数
    @Pointcut("execution(* org.ppl.frame.aop.Audience.perform(String)) && args(pName)")
    public void perform(String pName){}  //定义带参数的切点

    @Before("perform(pName)") 
    public silenceCellPhone(String pName) {
        System.out.println("Silencing cell phones");
        System.out.println("Ready to watch perform:" + pName);
    }
}
{% endhighlight %}
### (4) 使用aspect
{% highlight java linenos%}
@Configuration
@EnableAspectJAutoProxy
public class PerformanceConfig {
    @Bean
    public Audience audience() {
        return new Audience();
    }
}
{% endhighlight %}

### (5) 给对象新增方法

&emsp;&emsp;@DeclareParents注解(参考Spring实战————Page 120)<br>
&emsp;&emsp;参考这篇博客[spring-AOP通过注解@DeclareParents引入新的方法](https://blog.csdn.net/u010502101/article/details/76944753)

### (6) 基于XML方式的切面配置
&emsp;&emsp;参考Spring实战————Page 120

<hr>

# 4. Spring事务

## 4.1 配置事务
>TransactionConfig基本配置
{:.filename}
{% highlight java linenos%}
@Configuration
@EnableTransactionManagement
public class TransactionConfig {

    @Autowired
    private DataSource dataSource;

    @Bean
    public DataSourceTransactionManager transactionManager(DataSource dataSource) {
        DataSourceTransactionManager manager = new DataSourceTransactionManager();
        manager.setDataSource(dataSource);
        return manager;
    }
}
{% endhighlight %}

<hr>

## 4.2 切入事务
>@Transactional注解
{:.filename}
{% highlight java linenos%}
//建议加在Service实现类或者其方法上
@Transactional(propagation=Propagation.REQUIRED)                 //一般用于增删改方法
@Transactional                          //一般用于增删改方法(简写，默认配置就是REQUIRED)
@Transactional(propagation=Propagation.SUPPORTS, readOnly=true)  //一般用于查询类方法
{% endhighlight %}

>@Transactional参数解析
{:.filename}
{% highlight text linenos%}
propagation——传播行为
(1) REQUIRED        //共享当前事务，也就是多个事务迭代时会事务共享，并且会主动创建
(2) SUPPORT         //共享当前事务，但是不会主动创建，只能被传播
(3) MANDATORY       //共享当前事务，只能被传播，如果没有被传播则会抛出异常
(4) REQUIRES_NEW    //挂起当前事务，并且创建新事务
(5) NOT_SUPPORT     //挂起当前事务，并且自己以非事务方式执行
(6) NEVER           //不使用事务，如果有当前事务，抛异常
(7) NESTED          //事务嵌套，如果出现异常，会自己回滚自己的，互不影响
{% endhighlight %}