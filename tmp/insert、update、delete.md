## insert、update、delete

### 综述

在JDBC中，`insert`、`update`、`delete`、和`DDL`语句都可以用`java.sql.Statement`的`executeUpdate()`或`execute()`方法来执行。在MyBatis中也一样，最终都是通过`update()`来执行这些数据库动作的。MyBatis执行增删改语句的逻辑可以分为几个层次：`SqlSession层`、`Executor层`、`StatementHandler层`、`Statement层`。每一层都调用它的下一层，最后的`Statement层`就是JDBC的代码了，不在mybatis源码范围内，所以我们主要讨论前面3层的语句执行逻辑。这篇文章介绍insert、update、delete的过程。不包含缓存，缓存在query的时候再讨论。



TODO：解析标签在哪一层做

SqlSession层几乎不做任何工作，面向用户。

Executor层引入了缓存、插件。

StatementHandler层创建Statement，设置参数，调用selectKey、keyGenerator为对象赋值。

Statement层执行数据库操作。

TODO:时序图，分层

### 正文

#### SqlSession层

在MyBatis的DefaultSqlSession中我们可以发现，不管是insert、update或是delete，最终都会调用一个update方法。该方法会执行我们所有的增、删、改语句。

```java
  @Override
  public int insert(String statement, Object parameter) {
    return update(statement, parameter);
  }
  
  @Override
  public int delete(String statement, Object parameter) {
    return update(statement, parameter);
  }
  
  @Override
  public int update(String statement, Object parameter) {
      //用statement定位到要执行的SQL语句（以MappedStatement数据结构的形式）
      MappedStatement ms = configuration.getMappedStatement(statement);
      return executor.update(ms, wrapCollection(parameter));
  }
```

MappedStatement是mapper.xml文件中insert、update、delete、select 4种标签的转成的数据结构。看一下MappedStatement中的字段和标签中的属性，就会知道他们的关系了。

```java
public final class MappedStatement {
  private String id;
  private Integer fetchSize;
  private Integer timeout;
  private Cache cache;
  private boolean flushCacheRequired;
  private boolean useCache;
  private boolean resultOrdered;
  private SqlSource sqlSource;
  //...其他略
}

<insert
  id=""
  parameterType=""
  flushCache=""
  statementType=""
  keyProperty=""
  keyColumn=""
  useGeneratedKeys=""
  timeout="">
```

TODO:Executor怎么生成的？是SqlSessionFactory中new 出来的，根据execType生成不同的类型，execType哪里来的？

#### Executor层

Executor是一个接口，它的update()方法由BaseExecutor类实现，而BaseExecutor又调用具体子类的doUpdate()

```java
@Override//BaseExecutor的update()
public int update(MappedStatement ms, Object parameter) throws SQLException {
    return doUpdate(ms, parameter);
}

@Override
//ReuseExecutor和BatchExecutor中的doUpdate()，略...
//SimpleExecutor中的doUpdate(), 我们来重点关注这个
public int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
  Statement stmt = null;
  try {
    Configuration configuration = ms.getConfiguration();
    //1 创建StatementHandler
    StatementHandler handler = configuration.newStatementHandler(this, ms, parameter, RowBounds.DEFAULT, null, null);
    //2 初始化java.sql.Statement、并将提供参数
    stmt = prepareStatement(handler, ms.getStatementLog());
    //3 执行数据库操作
    return handler.update(stmt);
  } finally {
    //4 每次都要关闭statementHandler，这是一个一次性用品
    closeStatement(stmt);
  }
}
```

然后来追踪一下doUpdate()中的4个步骤

```java
//1 创建StatementHandler
public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
  //RoutingStatementHandler为一个包装类，它将根据MappedStatement中的statementType字段来创建相应的StatementHandler，然后用它来执行各种操作。
  StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
  //注意：这儿有一个拦截器！返回statementHandler之前会按顺序执行一遍拦截器的plugin()方法。
  statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
  return statementHandler;
}

//2 初始化java.sql.Statement、并将提供参数
private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
  Connection connection = getConnection(statementLog);
  //prepare 执行BaseStatementHandler类的prepare()方法, 见StatementHandler层
  //这条语句对应JDBC中的创建java.sql.Statement的代码！
  Statement stmt = handler.prepare(connection, transaction.getTimeout());
  //parameterize 执行具体实现类的parameterize()方法，见StatementHandler层
  //这条语句对应JDBC中为java.sql.Statement设置参数的语句！
  handler.parameterize(stmt);
  return stmt;
}

//3 执行数据库操作，见StatementHandler层
//4 只是简单调用java.sql.Statement接口的close()方法，略。
```

#### StatementHandler层

```java
@Override//BaseStatementHandler类的prepare()方法
public Statement prepare(Connection connection, Integer transactionTimeout) throws SQLException {
  //执行具体实现类的instantiateStatement()方法
  return instantiateStatement(connection);
}

@Override//这个方法对应了JDBC中的创建statement的代码
protected Statement instantiateStatement(Connection connection) throws SQLException {
  //BoundSql是MyBatis解析mapper.xml中的标签后生成的SQL语句对象。
  String sql = boundSql.getSql();
  return connection.prepareStatement(sql);
}


@Override//PreparedStatementHandler中的parameterize()
public void parameterize(Statement statement) throws SQLException {
  parameterHandler.setParameters((PreparedStatement) statement);
}

@Override//这条语句对应JDBC中为java.sql.Statement设置参数的语句！
public void setParameters(PreparedStatement ps) {
  List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
  if (parameterMappings != null) {
    for (int i = 0; i < parameterMappings.size(); i++) {
      ParameterMapping parameterMapping = parameterMappings.get(i);
      if (parameterMapping.getMode() != ParameterMode.OUT) {
        Object value;
        String propertyName = parameterMapping.getProperty();
        if (boundSql.hasAdditionalParameter(propertyName)) { // issue #448 ask first for additional params
          value = boundSql.getAdditionalParameter(propertyName);
        } else if (parameterObject == null) {
          value = null;
        } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
          value = parameterObject;
        } else {
          MetaObject metaObject = configuration.newMetaObject(parameterObject);
          value = metaObject.getValue(propertyName);
        }
        TypeHandler typeHandler = parameterMapping.getTypeHandler();
        JdbcType jdbcType = parameterMapping.getJdbcType();
        if (value == null && jdbcType == null) {
          jdbcType = configuration.getJdbcTypeForNull();
        }
       typeHandler.setParameter(ps, i + 1, value, jdbcType);
      }
    }
  }
}

//StatementHandler中的update()方法，有3种不同的实现
@Override//PreparedStatementHandler中的update()
public int update(Statement statement) throws SQLException {
  PreparedStatement ps = (PreparedStatement) statement;
  //Statement层执行SQL语句
  ps.execute();
  //处理<selectKey order="AFTER">的情况
  int rows = ps.getUpdateCount();
  Object parameterObject = boundSql.getParameterObject();
  KeyGenerator keyGenerator = mappedStatement.getKeyGenerator();
  keyGenerator.processAfter(executor, mappedStatement, ps, parameterObject);
  return rows;
}


```

