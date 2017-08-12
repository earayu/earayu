---
title: SpringBoot 启动过程
updated: 2017-08-12 22:25
---

## SpringBoot 启动过程

我在刚接触到spring的时候，它已经处于4.x版本了。记得在`spring in action` 这本书中，作者就表示`springboot` 是一项令人兴奋的技术。确实，现在接触到的项目无一例外都使用到了springboot。springboot以它的四大特性广受人民群众的喜爱：`自动配置`、`起步依赖`、`命令行界面`、`Actuator`。本文将介绍springboot的自动配置原理和启动过程。



### 小兄弟，你为何那么叼？

首先想想，单纯使用spring整合某个框架（如MyBatis），我们需要做些什么？

1. 添加各个第三方jar包的依赖
2. 加入各种配置文件：spring的、其他框架的、项目自身的等
3. 生成各种bean、加入spring容器

而我们发现，如果是springboot项目的话，只要加入了相应的`starter` 包，剩下的一切springboot都帮我们做好了。一旦启动springboot项目，容器中就有了相应的bean。那么，问题来了：小兄弟，你为何那么叼?



### 如何创建自己的starter

一句话：`springboot`项目在启动的时候会搜索`classpath`中的`META-INF\spring.factories`文件。然后根据文件的内容加载相应的配置类完成自动配置。

假如我们想提供一个**nothing-but-a-double-starter**，我们要做的只有简单3步：

1. 创建一个项目，然后创建一个配置类，如下：

   ```java
   @Configuration
   @ConditionalOnMissingBean(Double.class)
   public class MyAutoConfig
   {
       @Bean
       public Double createDouble()
       {
           return 3.14;
       }

   }
   ```

   `Configuration`注解表示这是一个spring的配置类。`bean`注解会将该方法返回的bean加入spring容器。`ConditionalOnMissingBean(Double.class)`表示只有当容器中不存在Double类的情况下，才使用这个配置类。***还有其他Conditional注解***

2. 创建一个`META-INF\spring.factories`文件，并在其中加入以下内容

   ```properties
   org.springframework.boot.autoconfigure.EnableAutoConfiguration = [MyAutoConfig的全类名]
   ```

3. 部署以上项目，在要使用这个`starter`的项目中加入对它的依赖。然后就会发现，容器中有一个Double类型的bean，并且它的值为3.14。



### SpringBoot源码分析

springboot的入口类通常如下。`@SpringBootApplication`是一个复合注解，由`@Configuration`、`@ComponentScan`、`@EnableAutoConfiguration` 这3个注解组成。
```java
@SpringBootApplication
public class MySpringBootAppication{
	public static void main(String[] args) {
		SpringApplication.run(MySpringBootAppication.class, args);
	}
}
```

`run()`方法会先new一个`SpringAppication对象`， 然后调用它的run方法。

```java
public static ConfigurableApplicationContext run(Object[] sources, String[] args) {
	return (new SpringApplication(sources)).run(args);
}
```




#### 构造方法源码

我们首先来看看`SpringAppication`的构造方法。它的内部主要是有一个`initialize()`方法。

```java
public SpringApplication(Object... sources) {
    this.initialize(sources);
  	//其他略
}

private void initialize(Object[] sources) {
    this.webEnvironment = this.deduceWebEnvironment();
    this.setInitializers(this.getSpringFactoriesInstances(ApplicationContextInitializer.class));
    this.setListeners(this.getSpringFactoriesInstances(ApplicationListener.class));
    this.mainApplicationClass = this.deduceMainApplicationClass();
}
```
我们来一行一行分析initialize()方法：
1. `deduceWebEnvironment()`判断当前项目是不是一个web项目（通过查找classpath中是否存在web相关的类）。

   ```java
   private boolean deduceWebEnvironment() {
       for (String className : WEB_ENVIRONMENT_CLASSES) {
           if (!ClassUtils.isPresent(className, null)) {
               return false;
           }
       }
       return true;
   }
   ```

2. 可以看到，关键是调用getSpringFactoriesInstances(ApplicationContextInitializer.class)，来获取ApplicationContextInitializer类型对象的列表。顾名思义，ApplicationContextInitializer是一个可以用来初始化ApplicationContext的接口。

   ```java
   private <T> Collection<? extends T> getSpringFactoriesInstances(Class<T> type) {
       Set<String> names = new LinkedHashSet<String>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
       List<T> instances = createSpringFactoriesInstances(type, parameterTypes,classLoader, args, names);
       return instances;
   }

   //在该方法中，首先通过调用SpringFactoriesLoader.loadFactoryNames(type, classLoader)来获取所有Spring Factories的名字，然后调用createSpringFactoriesInstances方法根据读取到的名字创建对象。最后会将创建好的对象列表排序并返回。
   public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";
   public static List<String> loadFactoryNames(Class<?> factoryClass, ClassLoader classLoader) {
       String factoryClassName = factoryClass.getName();
       Enumeration<URL> urls = (classLoader != null ? classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
                   ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
       List<String> result = new ArrayList<String>();
       while (urls.hasMoreElements()) {
           URL url = urls.nextElement();
           Properties properties = PropertiesLoaderUtils.loadProperties(new UrlResource(url));
           String factoryClassNames = properties.getProperty(factoryClassName);
           result.addAll(Arrays.asList(StringUtils.commaDelimitedListToStringArray(factoryClassNames)));
       }
       return result;
   }
   ```

   `springboot-starter`的jar包中的spring.factories文件有以下内容。这些就是上述代码会加载的类。`ApplicationContextInitializer类`会在spring的`refresh()`方法之前调用。

   ```properties
   # Application Context Initializers
   org.springframework.context.ApplicationContextInitializer=\
   org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer,\
   org.springframework.boot.context.ContextIdApplicationContextInitializer,\
   org.springframework.boot.context.config.DelegatingApplicationContextInitializer,\
   org.springframework.boot.context.web.ServerPortInfoApplicationContextInitializer
   ```

3. `setListeners()`方法做的事情跟第2点类似，略

4. 在`deduceMainApplicationClass()`方法中，通过获取当前调用栈，找到入口方法main所在的类，并将其复制给SpringApplication对象的成员变量mainApplicationClass。在我们的例子中mainApplicationClass即是我们自己编写的Application类。


#### run方法源码

> 分析run方法的主要内容，略去一些不重要内容。根据代码后部的注释，按点分析。

```java
public ConfigurableApplicationContext run(String... args) {
    ConfigurableApplicationContext context = null;
    SpringApplicationRunListeners listeners = this.getRunListeners(args);//1
    listeners.starting();//2
    ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
    ConfigurableEnvironment environment = this.prepareEnvironment(listeners, applicationArguments);
    Banner printedBanner = this.printBanner(environment);
    context = this.createApplicationContext();//3
    this.prepareContext(context, environment, listeners, applicationArguments, printedBanner);//3
    this.refreshContext(context);//3
    this.afterRefresh(context, applicationArguments);
    listeners.finished(context, (Throwable)null);//2
    return context;
}
```

1. 像之前分析的一样，从spring.factories中加载`SpringApplicationRunListener`

   ```properties
   # Run Listeners
   org.springframework.boot.SpringApplicationRunListener=\
   org.springframework.boot.context.event.EventPublishingRunListener
   ```

   `EventPublishingRunListener类`的源码比较冗杂，不贴出来。这里只说一下它的作用。`EventPublishingRunListener`实现了`SpringApplicationRunListener`。它会在`run()方法`执行到相应阶段的时候，将事件发送到容器的所有监听器中。

   ```java
   //以下5个方法对应run()的5个生命周期
   public interface SpringApplicationRunListener {
       void starting();
       void environmentPrepared(ConfigurableEnvironment var1);
       void contextPrepared(ConfigurableApplicationContext var1);
       void contextLoaded(ConfigurableApplicationContext var1);
       void finished(ConfigurableApplicationContext var1, Throwable var2);
   }
   ```

2. 如上述所说`listeners.starting()` 会将开始事件发给`ApplicationContext`的所有`Listener`。listeners.xxx()也是一样。

3. 创建并对ApplicationContext做一些处理

   ```java
   //判断是否是web项目，然后创建相应的AppicationContext。
   protected ConfigurableApplicationContext createApplicationContext() {
       Class<?> contextClass = this.applicationContextClass;
       if(contextClass == null) {
               contextClass = Class.forName(this.webEnvironment?"org.springframework.boot.context.embedded.AnnotationConfigEmbeddedWebApplicationContext":"org.springframework.context.annotation.AnnotationConfigApplicationContext");
       }
       return (ConfigurableApplicationContext)BeanUtils.instantiate(contextClass);
   }
   ```

   ​

#### 神奇的@EnableAutoConfiguration

> 想方设法找到加载自动配置的源码，没想到藏在注解里

`@EnableAutoConfiguration`中有一个`@Import注解`

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import({EnableAutoConfigurationImportSelector.class})
public @interface EnableAutoConfiguration {
    String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

    Class<?>[] exclude() default {};

    String[] excludeName() default {};
}
```

通过追踪`EnableAutoConfigurationImportSelector`,我们可以看到下面这个方法：它通过上文多次提到的`loadFactoryNames()`方法，从`spring.factories`文件获取需要自动配置的类，然后返回

```java
public String[] selectImports(AnnotationMetadata annotationMetadata) {
    //...
	List<String> configurations = this.getCandidateConfigurations(annotationMetadata, attributes);
    //...
}

protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
    List<String> configurations = SpringFactoriesLoader.loadFactoryNames(this.getSpringFactoriesLoaderFactoryClass(), this.getBeanClassLoader());
    return configurations;
}
protected Class<?> getSpringFactoriesLoaderFactoryClass() {
    return EnableAutoConfiguration.class;
}
```


### 其他

最近在看很多源码，包括工作项目、mybatis、spring。说两句自己的体会。看源码是一个剥丝抽茧的过程，首先要对目标代码的功能有一个比较详尽的了解，然后最好是能知道它的原理。最后是一层一层的去剥开源码，阅读源码的过程其实也是一个将源码和功能配对的过程。

在阅读开源项目源码的过程中，自然免不了去网上寻找优秀的博客。站在巨人的肩膀上，才能事倍功半。但也深刻体会到了语言描述的乏力和网页文本形式代码阅读的不便。所以一定要就着多篇优秀博客边看边捋代码。

推荐这篇, 写的比我更好：[[Spring Boot启动流程详解](http://www.cnblogs.com/xinzhao/p/5551828.html)](http://www.cnblogs.com/xinzhao/p/5551828.html)