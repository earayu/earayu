---
title: MyBatis binding包源码
updated: 2017-07-30 16:32
---

## MyBatis binding包源码

SqlSession接口提供了我们操作数据库的所有功能，而我们事实上开发的时候却很少会去直接用SqlSession接口。因为MyBatis提供了一个更优雅的解决方案：`binding包`。我们只需要定义一个DAO接口类，就可以操作数据库了。为什么可以那么方便呢？代码都在binding包中！

```java
public interface StudentDao {
    void add(Student student);
    Student findAll();
    Student findById(int id);
}

StudentDao dao = sqlSession.getMapper(StudentDao.class);
Student stu = dao.findById(1);
```



#### 头脑风暴

分析源码之前，先来思考一下这个需求给我们自己的话，要怎么实现？

1. 接口和mapper.xml是一对一的关系，可以通过`namespace`来联系。
2. 我们需要一个接口和mapper.xml文件的映射。在解析xml文件的阶段可以生成一个`map` 保存这种映射。
3. 真正的操作还是由SqlSession来执行，我们可以使用动态代理来创建接口的实现类。


#### 映射

MyBatis源码阅读起来方便的一个地方在于，一切问题都可以从解析xml的源码处入手。

这是解析mapper.xml文件的`parse()` 方法，可以看到其中有一个`bindMapperForNamespace()` 方法，它建立了`DAO接口`和`mapper.xml` 文件的映射。

```java
public void parse() {
  if (!configuration.isResourceLoaded(resource)) {
    //解析mapper.xml文件的配置，如cache、resultMap
    configurationElement(parser.evalNode("/mapper"));
    //将resource标记为已加载，防止重复加载
    configuration.addLoadedResource(resource);
    bindMapperForNamespace();
  }
  parsePendingResultMaps();
  parsePendingCacheRefs();
  parsePendingStatements();
}
//获取当前mapper.xml文件的namespace，然后加入到Configuration类的mapperRegistry 属性中。
private void bindMapperForNamespace() {
  String namespace = getCurrentNamespace();
  Class<?> boundType = Resources.classForName(namespace);
  configuration.addMapper(boundType);
}
```

`MapperRegistry` 类存放了`接口` 和`MapperProxyFactory类` 的映射。`MapperProxyFactory类` 一看名字就知道是为了生成`MapperProxy类` 而存在的，而`MapperProxy类` 是binding包中最重要的类。它就是`接口`的`代理对象` 。

```java
//configuration类的addMapper()方法会直接调用这个addMapper()方法。
public <T> void addMapper(Class<T> type) {
  if (type.isInterface()) {
      knownMappers.put(type, new MapperProxyFactory<T>(type));
      MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
      parser.parse();
  }
}
```



#### MapperProxy

让我们来按照动态代理的思路来阅读源码。动态代理生成的代理类，运行的时候会先被`invoke()` 方法拦截。那我们就先来看看`MapperProxy类` 的`invoke()` 方法吧。

```java
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
  try {
    //如果是类，则直接执行方法。
    if (Object.class.equals(method.getDeclaringClass())) {
      return method.invoke(this, args);
    //如果是接口，但是正在被代理的方法是接口的default方法，也直接执行
    } else if (isDefaultMethod(method)) {
      return invokeDefaultMethod(proxy, method, args);
    }
  } catch (Throwable t) {
    throw ExceptionUtil.unwrapThrowable(t);
  }
  //通过被代理方法(method)->获取代理方法(mapperMethod)，然后执行
  final MapperMethod mapperMethod = cachedMapperMethod(method);
  return mapperMethod.execute(sqlSession, args);
}

private MapperMethod cachedMapperMethod(Method method) {
  //method的类型为Map<Method, MapperMethod>，提供被代理方法和代理方法的映射
  MapperMethod mapperMethod = methodCache.get(method);
  if (mapperMethod == null) {
    //注意这儿，代理方法是懒加载的
    mapperMethod = new MapperMethod(mapperInterface, method, sqlSession.getConfiguration());
    methodCache.put(method, mapperMethod);
  }
  return mapperMethod;
}
```

#### MapperMethod

接口的每个具体方法，都会有一个代理类：`MapperMethod` 。它对外只有一个`execute()`方法。

```java
public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    switch (command.getType()) {
      case INSERT: {
    	Object param = method.convertArgsToSqlCommandParam(args);
        //执行SqlSession的insert方法
        result = rowCountResult(sqlSession.insert(command.getName(), param));
        break;
      }
      case UPDATE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        //执行SqlSession的update方法
        result = rowCountResult(sqlSession.update(command.getName(), param));
        break;
      }
      //略case ...
    }
    return result;
  }
```



#### 总结

所以，整个结构还有流程还是很清晰的。mybatis在解析xml文件阶段，为`DAO接口`和`mapper.xml`建立联系。接口名对应namespace，方法名对应CRUD标签的id。然后在调用`SqlSession.getMapper()方法` 的时候为`DAO接口`生成代理类对象。`getMapper()方法`返回的其实是代理类。执行`DAO接口` 的方法的时候执行的也是代理类的方法。代理类的相应方法，内部是调用`SqlSession接口`来完成数据库操作的。