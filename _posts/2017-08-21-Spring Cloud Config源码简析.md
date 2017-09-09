---
title: Spring Cloud Config源码简析
updated: 2017-08-21 01:10
---
## Spring Cloud Config源码简析

简单来说Spring Cloud Config服务端做的事情有以下几项：

* 从远程仓库克隆配置文件到本地临时目录（目录可配置）
* 从本地目录读取配置文件，生成Environment对象
* 将Environment对象通过服务的方式提供出去，让客户端得以访问

当然，也可以不使用远程仓库，也可以指定本地仓库的目录，有很多可定制化的功能。本篇文章将覆盖Spring Cloud Config服务端处理Git远程仓库的相关代码。使用SVN远程仓库的代码大同小异，将不做特别说明。



### 主要流程

#### 时序图

![1503285576(1)](/assets/1503285576(1).png)

#### 服务端入口

在Spring Cloud Config源码中有个`EnvironmentController类`， 我们可以发现它被`@RestController`标注了。其中`@RequestMapping`映射的方法，正是服务端的入口。我们通过它们来读取配置文件信息。

```java
@RestController
@RequestMapping(method = RequestMethod.GET, path = "${spring.cloud.config.server.prefix:}")
public class EnvironmentController {

	@RequestMapping("/{name}/{profiles}/{label:.*}")
	public Environment labelled(@PathVariable String name, @PathVariable String profiles,
			@PathVariable String label) {
		if (label != null && label.contains("(_)")) {
			// "(_)" is uncommon in a git branch name, but "/" cannot be matched
			// by Spring MVC
			label = label.replace("(_)", "/");
		}
		Environment environment = this.repository.findOne(name, profiles, label);
		return environment;
	}
}
```

上面的`labelled()`方法接收URL中的`name`、`profiles`、`label`作为参数，调用`findOne()`返回`Environment`对象。这个`findOne()`方法就是我们要重点关注的，它包含了服务端的大部分逻辑。

#### 包装

一如spring的其他代码，spring cloud config的源码也有着层层的包装。在debug时进入`findOne()`方法，首先进入的是`EnvironmentEncryptorEnvironmentRepository`类的`findOne()`。这个类使用了装饰器模式，它主要提供了解密配置文件的功能。我们只需要配置一个`EnvironmentEncryptor类`，就可以使用了。有了它，我们就有了将配置文件加密后存放在不安全地方的能力，比如Github。spring cloud config服务端会在读取后解密。

```java
@Override
public Environment findOne(String name, String profiles, String label) {
	Environment environment = this.delegate.findOne(name, profiles, label);
	if (this.environmentEncryptor != null) {
		environment = this.environmentEncryptor.decrypt(environment);
	}
	if (!this.overrides.isEmpty()) {
		environment.addFirst(new PropertySource("overrides", this.overrides));
	}
	return environment;
}
```



继续，接下来进入的是`CompositeEnvironmentRepository`类的`findOne()`方法。看它的源码很容易发现，它是一个`Repository`集合。它可以从多个地方读取配置文件，然后将他们合并后返回。

```java
@Override
public Environment findOne(String application, String profile, String label) {
	Environment env = new Environment(application, new String[]{profile}, label, null, null);
	if(environmentRepositories.size() == 1) {
		Environment envRepo = environmentRepositories.get(0).findOne(application, profile, label);
		env.addAll(envRepo.getPropertySources());
		env.setVersion(envRepo.getVersion());
		env.setState(envRepo.getState());
	} else {
		for (EnvironmentRepository repo : environmentRepositories) {
			env.addAll(repo.findOne(application, profile, label).getPropertySources());
		}
	}
	return env;
}
```



然后进入的是`MultipleJGitEnvironmentRepository`类的`findOne()`方法。如果我们配置了多个远程仓库的话，它可以分别从多个远程仓库读取配置文件。

```java
@Override
public Environment findOne(String application, String profile, String label) {
	for (PatternMatchingJGitEnvironmentRepository repository : this.repos.values()) {
		if (repository.matches(application, profile, label)) {
			for (JGitEnvironmentRepository candidate : getRepositories(repository,
					application, profile, label)) {
				try {
					if (label == null) {
						label = candidate.getDefaultLabel();
					}
					Environment source = candidate.findOne(application, profile,
							label);
					if (source != null) {
						return source;
					}
				}
				catch (Exception e) {
					if (logger.isDebugEnabled()) {
						this.logger.debug(
								"Cannot load configuration from " + candidate.getUri()
										+ ", cause: (" + e.getClass().getSimpleName()
										+ ") " + e.getMessage(),
								e);
					}
					continue;
				}
			}
		}
	}
	JGitEnvironmentRepository candidate = getRepository(this, application, profile,
			label);
	if (label == null) {
		label = candidate.getDefaultLabel();
	}
	if (candidate == this) {
		return super.findOne(application, profile, label);
	}
	return candidate.findOne(application, profile, label);
}
```



然后进入的是`JGitEnvironmentRepository类`的`findOne()`。因为`JGitEnvironmentRepository`继承自`AbstractScmEnvironmentRepository`而没有覆盖`findOne()`。所以阅读源码时会跳转到`AbstractScmEnvironmentRepository类`。这个读者需要注意一下。

这儿有个特别的地方，它用了`synchronized`关键字。这儿涉及到了一些线程安全的点。同时，这个函数里的一些操作本身就是比较耗时的，比如从远程仓库克隆。所以执行起来会比较慢。

这部分代码就是spring cloud config的核心，按照下面的标注，我们一个个看。

1. 创建一个`NativeEnvironmentRepository`实例。
2. 从远程仓库同步数据到本地，然后返回本地仓库的地址
3. 读取本地git仓库的配置，返回`Environment`对象
4. 一些后置处理

```java
@Override
public synchronized Environment findOne(String application, String profile, String label) {
  	//1
	NativeEnvironmentRepository delegate = new NativeEnvironmentRepository(
			getEnvironment());
	//2 从远程仓库同步数据到本地，然后返回本地仓库的地址
	Locations locations = getLocations(application, profile, label);
	delegate.setSearchLocations(locations.getLocations());
	//3 从本地仓库获取Environment对象
	Environment result = delegate.findOne(application, profile, "");
	result.setVersion(locations.getVersion());
	result.setLabel(label);
  	//4
	return this.cleaner.clean(result, getWorkingDirectory().toURI().toString(),
			getUri());
}
```

我们可以进入`getLocations()`方法看看，它是怎样从远程仓库克隆数据到本地的。经过几个包装后到达了`JGitEnvironmentRepository`的`getLocations()`。而关键的代码都在`refresh()`中。`refresh()`会从远程仓库`fetch`代码、`checkout`到`label`指定的分支、`merge`数据、有必要的话还会执行`git reset --hard`命令。使用的时候需要注意。这一部分就是从远程仓库读取数据，保存到本地的关键代码。

```java
//NOTE：为了提高阅读舒适度，这儿删减了部分源码
@Override
public synchronized Locations getLocations(String application, String profile,
		String label) {
	// 克隆或更新本地仓库。 
	String version = refresh(label);
	return new Locations(application, profile, label, version,
			getSearchLocations(getWorkingDirectory(), application, profile, label));
}

public String refresh(String label) {
	Git git = createGitClient();
	if (shouldPull(git)) {
		fetch(git, label);
		checkout(git, label);
		if (isBranch(git, label)) {
			merge(git, label);
			if (!isClean(git)) {
				resetHard(git, label, "refs/remotes/origin/" + label);
			}
		}
	}
	else {
		// 没有东西需要更新，所以只切换一下分支
		checkout(git, label);
	}
	// 返回HEAD指针的指向的版本号
	return git.getRepository().findRef("HEAD").getObjectId().getName();
}
```

然后我们来看，从本地仓库读取配置文件的代码。进入`NativeEnvironmentRepository`的`findOne()`。这边的代码很特别，一开始看到我是懵逼的。Spring Cloud Config在这儿会创建一个临时的SpringApplication，将配置文件地址设置为之前返回的本地仓库目录。然后把这个临时SpringApplication生成的Environment对象返回。这样的好处在于复用了代码，实现特别简单。但同时，这个执行起来还是比较慢的。我不知道这种方式和只加载配置文件生成Environment对象效率有多大区别，如果你知道，请告诉我。

```java
public Environment findOne(String config, String profile, String label) {
	SpringApplicationBuilder builder = new SpringApplicationBuilder(
			PropertyPlaceholderAutoConfiguration.class);
	ConfigurableEnvironment environment = getEnvironment(profile);
	builder.environment(environment);
	builder.web(false).bannerMode(Mode.OFF);
	if (!logger.isDebugEnabled()) {
		// Make the mini-application startup less verbose
		builder.logStartupInfo(false);
	}
	String[] args = getArgs(config, profile, label);
	// Explicitly set the listeners (to exclude logging listener which would change
	// log levels in the caller)
	builder.application()
			.setListeners(Arrays.asList(new ConfigFileApplicationListener()));
	ConfigurableApplicationContext context = builder.run(args);
	environment.getPropertySources().remove("profiles");
	try {
		return clean(new PassthruEnvironmentRepository(environment).findOne(config,
				profile, label));
	}
	finally {
		context.close();
	}
}
```

因为临时SpringBootApplication启动后的`Enviroment`会有很多个不需要的`PropertySource`，所以这边有一个`clean()`操作。它会将`Enviroment`对象中不需要的一些数据去掉，并且把一些数据转换一下形式提供给客户端，这些都比较细枝末节，不详细说明。

有一个需要注意的地方是，`PropertySource`是有先后顺序的，前面的`PropertySource`优先级大于后面的。临时SpringBootApplication会帮我们加入当前`profile`对应的配置文件，**以及默认配置文件**。只不过默认配置文件的优先级是低于当前`profile`对应的配置文件的。



以上就是Spring Cloud Config 服务端的主要流程。
