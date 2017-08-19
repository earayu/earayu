---
title: 性感的 Spring Cloud Config
updated: 2017-08-18 01:15
---

## 性感的 Spring Cloud Config

>本文覆盖spring cloud config的基本使用、高可用方案；以及一些我遇到过的小坑。
>
>配合[这个](https://github.com/earayu/configserver)食用，风味更佳。

#### 前言

我大三的时候曾在一家提供支付服务的公司实习过，开发一个分布式的转账系统。当时我们3个实习生就负责了整个后台的编码，受限于当时的经验和眼界，没有使用到任何的分布式或微服务框架。对应到配置文件这一块就是：我们的程序部署N各实例，每个实例的包是一样的，部署到服务器后会有脚本自动执行，修改配置文件的相关部分，达到一个半自动化的目的。但本质上还是，每个项目实例使用自己本地的配置文件。这样的弊端很多，最明显的就是，如果要修改配置文件，必须手动去服务器上修改每个实例的配置文件。Spring Cloud作为一个分布式框架，提供了它的配置文件解决方案：Spring Cloud Config。

#### 概要

Spring Cloud Config分为服务端和客户端。服务端读取物理配置文件，然后通过服务的方式提供出去。它可以和Git或SVN配合使用，以达到用版本控制来管理配置文件的目的。而客户端则主要是消费服务端提供的服务。为了达到高可用的目的，服务端可以和Eureka或consul配合使用。

#### 这是一份使用说明书

实际开发中，我们通常都会有多套环境： 如果用版本控制来管理配置文件就非常合适。spring有一个概念叫`profile`。 像[这个项目](https://github.com/earayu/configserver)一样，假如我们有4套环境，则他们的`profile`分别为`默认(default)`、 `开发(dev)`、`测试(test)`、`生产(prd)`。每个环境都可能发生各种变动，版本控制工具可以记录这些变动，而spring也有对应的概念叫`label`。 举个例子：

```
configserver-prd.yml
```

这个配置文件的`profile`就是`prd`， 因为它处于git的master分支下，所以它的`label`就是`master`。当然，也可以用git的SHA码来表示，如`67769093f63f7a9f4abad4599ce6dff6b2245308`。

> 然而，`configserver-prd.yml`这个文件的`profile`为什么就是`prd`呢？它的`label`为什么就是`master`呢？因为它实际上，就仅仅是一个文本文件，被保存在github上而已。唯一相关的东西，好像就是它有一个`prd`后缀了。

 Spring Cloud Config的服务端在读取物理配置文件的时候，会使用一些约定好的方式，将它们解释成对应的`application(也就是name)`、`profile`和`label`。 要记住喔，每一个配置文件，都有这3个属性！

```python
# 蛤蛤，这个值得好好说一说
# 配置文件以以下方式解释
{application}-{profile}.yml
# 然后可以通过以下URL来提供服务
/{application}/{profile}[/{label}]
/{label}/{application}-{profile}.yml
/{application}-{profile}.properties
/{label}/{application}-{profile}.properties

# 为什么我要强调这个值得好好说一说？
# 考虑刚才那个文件 configserver-prd.yml
# 它可以被解释为：application=configserver; profile=prd; label=master
# 通过上述URL可以访问到它，如http://[host]/configserver/prd/master

# 同时！！！
# 它也可以被解释为：application=configserver-prd; profile=default; laber=master
# 要访问的话，URL为：http://[host]/configserver-prd/default/master
# yeah，这就是我遇到的第一个坑。同一个配置文件，其实是可以有多种解释的。
# 横看成林侧成峰，远近高低各不同。
# PS:上面的案例中，因为匹配不到profile，所以profile=default。
```



关于`profile=default`的配置文件，值得一提的是：它默认会被继承！

蛤？这是啥意思？配置文件还可以继承？

```python
# yeah，spring cloud config就是这么任性。
# 它默认会将default环境的Key-Value都加入别的环境，如果Key相同，则优先使用当前环境的Value
# 举个栗子，如果我们有2个配置文件：
# configserver.yml或configserver-default.yml
# configserver-prd.yml
# configserver-prd.yml会继承configserver.yml的所有Key-Value。
# 我们厂里的profile都是default，我猜测就是为了应对这个问题。
# 这是我遇到的第二个坑。
```

#### 使用

食用方式特别简单，加半勺盐、葱花。。。额。。。

服务端加入相应的依赖、启动类加上注解、配置文件写上物理配置文件地址。我纠结了一下，决定不贴上代码，可以从[configserver](https://github.com/earayu/configserver)看到详情。

客户端也是加入相应的依赖、不用加注解、配置文件写上服务端地址，就可以了。可以从[configclient](https://github.com/earayu/configserver)看到详情。

#### 动态配置

最近开发项目的时候，最先是有改动就重新去测试环境部署一下，每次5分钟，后来受不了了。把环境迁移到本地，每次有改动部署一下1分钟，但有时候还是感觉太漫长了。那就试试这个吧！

Spring Boot提供的`DevTools`可以实现每次只部署有改动的类功能，所以每次代码修改后几秒钟就完成了重新部署。这个在本文不展开讲解。

那么配置怎么办？

Spring Cloud Config其实是可以完成动态配置功能的。它配合Spring Boot Actuator的`/refresh`接口，每次触发对`/refresh`接口的POST请求，被`@RefreshScope`注解的类中的配置都会刷新。

```java
@SpringBootApplication
@RestController
@RefreshScope
public class ConfigclientApplication {
	//这个会被/refresh触发后刷新
	@Value("${name}")
	private String name;
}
```



Spring Boot Actuator的/refresh接口默认是不提供出来的，你可以这样来打开它：

```yaml
management:
  security:
    enabled: false

endpoints:
  refresh:
    enabled: true
```



TODO HOOK



#### 高可用

TODO





```

```