---
layout: post
title: 记一次Spring-Mybatis事务失效的bug排查过程
date:   2018-10-14 21:59
comments: true
description: 重构过程遇到的一个坑，特此记录
tags:
- java
- spring
- mybatis
- 坑
---

>**001——bug现象**

&emsp;&emsp;在项目重构过程中，需要把所有`spring`的`xml`配置全部改成`JavaConfig`的形式，其他的配置都比较顺利，就是发现`数据库事务`无论如何都不生效，排查了大半天<br>
&emsp;&emsp;基本配置就不写了，记录一下排查思路:<br>

>**002——排查思路**

- 首先确认`@ComponentScan`扫到了所有需要添加事务的`Service实现类`，注意spring官方不建议在`Service接口`上添加事务`@Transactional`，原文如下：
{% highlight text %}
            Spring recommends that you only annotate concrete classes
        (and methods of concrete classes)with the @Transactional
        annotation, as opposed to annotating interfaces. You certainly 
        can place the @Transactional annotation on an interface 
        (or an interface method), but this works only as you would expect 
        it to if you are using interface-based proxies. 
            The fact that Java annotations are not inherited from 
        interfaces means that if you are using class-based proxies 
        (proxy-target-class="true") or the weaving-based aspect 
        (mode="aspectj"), then the transaction settings are not recognized 
        by the proxying and weaving infrastructure, and the object will 
        not be wrapped in a transactional proxy, which would be decidedly bad.

{% endhighlight text %}

&emsp;&emsp;意思是在接口或其方法添加`@Transactional`一定是`JDK动态代理`，无论是否配置proxy-target-class="true"，想要实现`CGLIB代理`需要在实现类或者实现类的方法上添加事务`@Transactional`

- 然后检查数据库引擎是否是`Innodb`，如果是`MyIsam`则不支持事务（这一点直接pass，重构前事务是正常的）

&emsp;&emsp;检查了以上都没有查出原因，接下来打开`log4j`的`debug`扫一遍：
![框架]({{ '/images/blog/2-1.png' | relative_url }})

&emsp;&emsp;发现事务实际上是回滚了的，但是数据库还是变化了，这就很诡异了= =<br>
&emsp;&emsp;猜测这种情况很可能是`TransactionManager`绑错了对象，根本没有对这个DataSource真正回滚。<br>
&emsp;&emsp;最终发现问题出在这里：

>DataSourceConfig.java
{:.filename}
{% highlight java linenos%}
//...
//数据库连接池
    @Bean
    public DataSource dataSource() {
        DruidDataSource dataSource = new DruidDataSource();
        dataSource.setUrl(env.getProperty("jdbc.url"));
        dataSource.setUsername(env.getProperty("jdbc.username"));
        dataSource.setPassword(env.getProperty("jdbc.password"));
        dataSource.setDriverClassName(env.getProperty("jdbc.driver"));
        dataSource.setMaxActive(env.getProperty("jdbc.max_active", Integer.class));
        dataSource.setMinIdle(env.getProperty("jdbc.min_idle", Integer.class));
        System.out.println("new a dataSource");  //测试发现这里打印了两次
        return dataSource;
    }
    @Bean
    public SqlSessionFactoryBean sqlSessionFactory() {
        SqlSessionFactoryBean sqlSessionFactory = new SqlSessionFactoryBean();
        sqlSessionFactory.setDataSource(dataSource()); //问题出在这里！！
        sqlSessionFactory.setConfigLocation(new ClassPathResource("mybatis/SqlMapConfig.xml"));
        return sqlSessionFactory;
    }
//...
{% endhighlight %}

&emsp;&emsp;原因在于`sqlSessionFactory`是mybatis最早创建的，此时spring还没有来得及拦截`dataSource()`方法，所以即使`dataSource()`是一个`@Bean`方法，在spring完成拦截前仍被看作是普通方法；<br>
&emsp;&emsp;每次调用`dataSource()`时，并不会直接返回单例，而是重新`new`一个新对象，所以sqlSessionFactory里面注入的`dataSource`和spring容器中的`dataSource`是两个不同的对象！
&emsp;&emsp;破案了。。把`sqlSessionFactory()`的注入方式改一下就ok了:
>DataSourceConfig.java
{:.filename}
{% highlight java linenos%}
//...
//数据库连接池
    @Bean
    public DataSource dataSource() {
        DruidDataSource dataSource = new DruidDataSource();
        dataSource.setUrl(env.getProperty("jdbc.url"));
        dataSource.setUsername(env.getProperty("jdbc.username"));
        dataSource.setPassword(env.getProperty("jdbc.password"));
        dataSource.setDriverClassName(env.getProperty("jdbc.driver"));
        dataSource.setMaxActive(env.getProperty("jdbc.max_active", Integer.class));
        dataSource.setMinIdle(env.getProperty("jdbc.min_idle", Integer.class));
        System.out.println("new a dataSource");  //这里只会在Spring初始化的时候调用一次
        return dataSource;
    }
    @Bean
    public SqlSessionFactoryBean sqlSessionFactory(DataSource dataSource) {
        SqlSessionFactoryBean sqlSessionFactory = new SqlSessionFactoryBean();
        sqlSessionFactory.setDataSource(dataSource); //这里保证调用的是Spring容器中的dataSource
        sqlSessionFactory.setConfigLocation(new ClassPathResource("mybatis/SqlMapConfig.xml"));
        return sqlSessionFactory;
    }
//...
{% endhighlight %}

>**003——完整配置**

>RootConfig.java
{:.filename}
{% highlight java linenos%}
/**
 * Spring根配置
 * @author PPL
 */
@Configuration
@ComponentScan(basePackages = {"org.ppl.mall"})
@Import({
        DataSourceConfig.class,
        RedisConfig.class,
        SolrConfig.class,
        DubboConfig.class,
        TransactionConfig.class
})
@ImportResource("classpath:spring/applicationContext*.xml")
public class RootConfig {
}
{% endhighlight %}


>DataSourceConfig.java
{:.filename}
{% highlight java linenos%}
/**
 * 持久层配置
 * 实现EnvironmentAware接口，是因为dataSource是最早配置的SpringConfig
 * 此时Spring还未加载Environment，需要手动加载，否则Environment为null
 * @author PPL
 */
@Configuration
@PropertySource("classpath:/conf/db.properties")
public class DataSourceConfig implements EnvironmentAware {

    private Environment env;

    @Override
    public void setEnvironment(Environment env) {
        this.env = env;
    }

    //数据库连接池
    @Bean
    public DataSource dataSource() {
        DruidDataSource dataSource = new DruidDataSource();
        dataSource.setUrl(env.getProperty("jdbc.url"));
        dataSource.setUsername(env.getProperty("jdbc.username"));
        dataSource.setPassword(env.getProperty("jdbc.password"));
        dataSource.setDriverClassName(env.getProperty("jdbc.driver"));
        dataSource.setMaxActive(env.getProperty("jdbc.max_active", Integer.class));
        dataSource.setMinIdle(env.getProperty("jdbc.min_idle", Integer.class));

        return dataSource;
    }

    //SqlSessionFactory
    @Bean
    public SqlSessionFactoryBean sqlSessionFactory(DataSource dataSource) {
        SqlSessionFactoryBean sqlSessionFactory = new SqlSessionFactoryBean();
        sqlSessionFactory.setDataSource(dataSource);
        sqlSessionFactory.setConfigLocation(new ClassPathResource("mybatis/SqlMapConfig.xml"));
        return sqlSessionFactory;
    }

    //Mapper扫描
    @Bean
    public MapperScannerConfigurer mapperScannerConfigurer() {
        MapperScannerConfigurer mapperScannerConfigurer = new MapperScannerConfigurer();
        mapperScannerConfigurer.setBasePackage("org.ppl.mall.mapper");
        mapperScannerConfigurer.setSqlSessionFactoryBeanName("sqlSessionFactory");
        return mapperScannerConfigurer;
    }
}
{% endhighlight %}

>TransactionConfig.java
{:.filename}
{% highlight java linenos%}
/**
 * 事务配置
 * @author PPL
 */
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