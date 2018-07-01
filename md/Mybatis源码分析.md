# 
* [MyBatis使用](https://github.com/lucky-xin/Learning/blob/gh-pages/md/MybatisUse.md)

* 源码解析 
## 基于JDK动态代理实现，JDK代理必须有接口，而mapper就是接口。
#### 1.以Mapper接口方法名为XML节点名称配置映射文件xml。
#### 2.初始化时根据xml配置信息和Mapper接口定义为每个方法生成每个代理方法MapperMethod。
根据Mapper接口信息使用jdk代理生产代理对象MapperProxy。MapperProxy封装了所有的代理方法MapperMethod。然后注册Mapper.
#### 3.使用SqlSessionFactory创建SqlSession对象。通过SqlSession的getMapper方法获取Mapper时返回jdk代理生成的代理对象。调用Mapper对象方法时则传入参数SqlSession，调用代理方法MapperMethod。SqlSession封装了insert，update，delete方法，调用MapperMethod代理方法则调用SqlSession对象相应的数据库操作方法。源码实现如下：

* 创建SqlSessionFactory对象
### 配置xml文件创建SqlSessionFactory。根据SqlSessionFactory获取SqlSession对象。
```java
public class MySqlSessionFactory {

private static SqlSessionFactory sqlSessionFactory = null;

static void init() {
        // 1.获取配置文件
        String resource = "SqlMapConfig.xml";
        InputStream in;
        try {
            in = Resources.getResourceAsStream(resource);
            // 2.创建会话工厂
            sqlSessionFactory = new SqlSessionFactoryBuilder().build(in);
        } catch (IOException e) {
            throw new ExceptionInInitializerError(e);
        }
    }
}
```
### 调用SqlSessionFactoryBuilder解析xml配置文件，使用Configuration封装配置信息。根据Configuration构建SqlSessionFactory对象。并且注册Mapper并根据<transactionManager type="JDBC" />配置项确定事务工厂TransactionFactory,type="JDBC"则使用JdbcTransactionFactory事务工厂
```java
public class SqlSessionFactoryBuilder {

  public SqlSessionFactory build(Reader reader) {
    return build(reader, null, null);
  }

  public SqlSessionFactory build(Reader reader, String environment) {
    return build(reader, environment, null);
  }

  public SqlSessionFactory build(Reader reader, Properties properties) {
    return build(reader, null, properties);
  }

  public SqlSessionFactory build(Reader reader, String environment, Properties properties) {
    try {
      XMLConfigBuilder parser = new XMLConfigBuilder(reader, environment, properties);
      return build(parser.parse());//解析XML配置文件，注册Mapper
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error building SqlSession.", e);
    } finally {
      ErrorContext.instance().reset();
      try {
        reader.close();
      } catch (IOException e) {
        // Intentionally ignore. Prefer previous error.
      }
    }
  }

  public SqlSessionFactory build(InputStream inputStream) {
    return build(inputStream, null, null);
  }

  public SqlSessionFactory build(InputStream inputStream, String environment) {
    return build(inputStream, environment, null);
  }

  public SqlSessionFactory build(InputStream inputStream, Properties properties) {
    return build(inputStream, null, properties);
  }

  public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
    try {
      XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
      //解析XML配置文件，注册Mapper，根据<transactionManager type="JDBC" />配置项确定事务工厂TransactionFactory
      return build(parser.parse());
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error building SqlSession.", e);
    } finally {
      ErrorContext.instance().reset();
      try {
        inputStream.close();
      } catch (IOException e) {
        // Intentionally ignore. Prefer previous error.
      }
    }
  }
  //获取默认的SqlSessionFactory为DefaultSqlSessionFactory
  public SqlSessionFactory build(Configuration config) {
    return new DefaultSqlSessionFactory(config);
  }

}

```
# 注册Mapper过程。
### 调用XMLConfigBuilder的方法parse解析配置文件并注册Mapper。
XMLConfigBuilder的parse方法
```java
 public Configuration parse() {
    if (parsed) {
      throw new BuilderException("Each XMLConfigBuilder can only be used once.");
    }
    parsed = true;
    parseConfiguration(parser.evalNode("/configuration"));
    return configuration;
 }
private void parseConfiguration(XNode root) {
    try {
      //issue #117 read properties first
      propertiesElement(root.evalNode("properties"));//jdbc properties 配置
      Properties settings = settingsAsProperties(root.evalNode("settings"));//获取MyBatis配置如缓存设置等
      loadCustomVfs(settings);
      typeAliasesElement(root.evalNode("typeAliases"));//获取别名配置
      pluginElement(root.evalNode("plugins"));
      objectFactoryElement(root.evalNode("objectFactory"));
      objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
      reflectorFactoryElement(root.evalNode("reflectorFactory"));
      settingsElement(settings);
      // read it after objectFactory and objectWrapperFactory issue #631
      environmentsElement(root.evalNode("environments"));//jdbc环境设置
      databaseIdProviderElement(root.evalNode("databaseIdProvider"));
      typeHandlerElement(root.evalNode("typeHandlers"));
      mapperElement(root.evalNode("mappers"));//解析Mapper并注册
    } catch (Exception e) {
      throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
}

private void mapperElement(XNode parent) throws Exception {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {
          // 获取Class文件包名
        if ("package".equals(child.getName())) {
          String mapperPackage = child.getStringAttribute("name");
          // 注册包下的所有Mapper
          configuration.addMappers(mapperPackage);
        } else {
          String resource = child.getStringAttribute("resource");
          String url = child.getStringAttribute("url");
          String mapperClass = child.getStringAttribute("class");
          if (resource != null && url == null && mapperClass == null) {
            ErrorContext.instance().resource(resource);
            InputStream inputStream = Resources.getResourceAsStream(resource);
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
            mapperParser.parse();
          } else if (resource == null && url != null && mapperClass == null) {
            ErrorContext.instance().resource(url);
            InputStream inputStream = Resources.getUrlAsStream(url);
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());
            mapperParser.parse();
          } else if (resource == null && url == null && mapperClass != null) {
            Class<?> mapperInterface = Resources.classForName(mapperClass);
            configuration.addMapper(mapperInterface);
          } else {
            throw new BuilderException("A mapper element may only specify a url, resource or class, but not more than one.");
          }
        }
      }
    }
  }
```
### 最后委派到Configuration的addMappers方法注册Mapper
```java
 public void addMappers(String packageName) {
    mapperRegistry.addMappers(packageName);
  }
```
### 最后把Mapper注册到MapperRegistry之中
```java
  public void addMappers(String packageName) {
    addMappers(packageName, Object.class);
  }
  
   public void addMappers(String packageName, Class<?> superType) {
      ResolverUtil<Class<?>> resolverUtil = new ResolverUtil<Class<?>>();
      resolverUtil.find(new ResolverUtil.IsA(superType), packageName);
      Set<Class<? extends Class<?>>> mapperSet = resolverUtil.getClasses();
      for (Class<?> mapperClass : mapperSet) {
        addMapper(mapperClass);
      }
   }
    // 之后以Class->MapperProxyFactory存入Map之中
    public <T> void addMapper(Class<T> type) {
       if (type.isInterface()) {
         if (hasMapper(type)) {
           throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
         }
         boolean loadCompleted = false;
         try {
           knownMappers.put(type, new MapperProxyFactory<T>(type));
           // It's important that the type is added before the parser is run
           // otherwise the binding may automatically be attempted by the
           // mapper parser. If the type is already known, it won't try.
           MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
           parser.parse();
           loadCompleted = true;
         } finally {
           if (!loadCompleted) {
             knownMappers.remove(type);
           }
         }
       }
     }
```
### 这样所有的Mapper都注册到MapperRegistry之中。Configuration持有MapperRegistry对象。使用SqlSessionFactoryBuilder创建
SqlSessionFactory为DefaultSqlSessionFactory。通过DefaultSqlSessionFactory的构造方法创建并设置Configuration
`
public DefaultSqlSessionFactory(Configuration configuration) {
    this.configuration = configuration;
  }
`
# 获取Mapper
### 调用DefaultSqlSessionFactory方法openSession获取SqlSession。SqlSession封装了数据库操作方法。通过事务工厂对象创建事务Transaction（事务对象封装了javax.sql.DataSource,java.sql.Connection对象），执行器Executor持有事务对象Transaction。再通过DefaultSqlSession封装配置信息对象Configuration,执行器Executor。返回DefaultSqlSession对象
### 根据配置文件确定事务工厂TransactionFactory如果type="jdbc"则使用JdbcTransactionFactory，使用JdbcTransactionFactory创建事务对象JdbcTransaction。JdbcTransaction封装了java.sql.Connection对象，事务实现基于java.sql.Connection实现。看源码会知道（事务实现方法之一）java.sql.Connection有setSavepoint方法（返回值为Savepoint），rollback(Savepoint)方法。事务实现为java.sql.Connection执行commit方法之前先调用setSavepoint保存现场，如果commit失败则调用rollback(Savepoint)方法恢复现场。后面详细分析。
```java
public class DefaultSqlSessionFactory implements SqlSessionFactory {
    
 @Override
  public SqlSession openSession() {
    return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, false);
  }
  
  private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
      Transaction tx = null;
      try {
        final Environment environment = configuration.getEnvironment();
        // 根据配置文件确定事务工厂TransactionFactory如果type="jdbc"则使用JdbcTransactionFactory
        final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
        // 使用使用JdbcTransactionFactory创建事务对象JdbcTransaction
        tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
        
        final Executor executor = configuration.newExecutor(tx, execType);
        
        return new DefaultSqlSession(configuration, executor, autoCommit);
      } catch (Exception e) {
        closeTransaction(tx); // may have fetched a connection so lets call close()
        throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
      } finally {
        ErrorContext.instance().reset();
      }
    }
    
  //创建TransactionFactory，通过TransactionFactory获取Transaction
  private TransactionFactory getTransactionFactoryFromEnvironment(Environment environment) {
      if (environment == null || environment.getTransactionFactory() == null) {
        return new ManagedTransactionFactory();
      }
      return environment.getTransactionFactory();
    }
}

public enum TransactionIsolationLevel {
  //无
  NONE(Connection.TRANSACTION_NONE),
 //  (读已提交)：可避免脏读的发生READ_COMMITTED(Connection.TRANSACTION_READ_COMMITTED),
 // (读未提交)：最低级别，任何情况都无法保证  
READ_UNCOMMITTED(Connection.TRANSACTION_READ_UNCOMMITTED),
 //  (可重复读)：可避免脏读、不可重复读的发生REPEATABLE_READ(Connection.TRANSACTION_REPEATABLE_READ),
   (串行化)：可避免脏读、不可重复读、幻读的发生,但是性能低下
  SERIALIZABLE(Connection.TRANSACTION_SERIALIZABLE);

}
```
### JdbcTransactionFactory源码如下
```java
public class JdbcTransactionFactory implements TransactionFactory {

  @Override
  public void setProperties(Properties props) {
  }

  @Override
  public Transaction newTransaction(Connection conn) {
    return new JdbcTransaction(conn);
  }

  @Override
  public Transaction newTransaction(DataSource ds, TransactionIsolationLevel level, boolean autoCommit) {
    return new JdbcTransaction(ds, level, autoCommit);
  }
}
```
### newTransaction方法创建了JdbcTransaction对象，JdbcTransaction封装了javax.sql.DataSource,java.sql.Connection对象，事务基于java.sql.Connection对象实现。JdbcTransaction的commit方法直接调用java.sql.Connection对象的commit方法实现，回滚方法直接调用java.sql.Connection对象的rollback方法实现
```java
public class JdbcTransaction implements Transaction {

  private static final Log log = LogFactory.getLog(JdbcTransaction.class);

  protected Connection connection;
  protected DataSource dataSource;
  protected TransactionIsolationLevel level;
  // MEMO: We are aware of the typo. See #941
  protected boolean autoCommmit;

  public JdbcTransaction(DataSource ds, TransactionIsolationLevel desiredLevel, boolean desiredAutoCommit) {
    dataSource = ds;
    level = desiredLevel;
    autoCommmit = desiredAutoCommit;
  }

  public JdbcTransaction(Connection connection) {
    this.connection = connection;
  }

  @Override
  public Connection getConnection() throws SQLException {
    if (connection == null) {
      openConnection();
    }
    return connection;
  }

  @Override
  public void commit() throws SQLException {
    if (connection != null && !connection.getAutoCommit()) {
      if (log.isDebugEnabled()) {
        log.debug("Committing JDBC Connection [" + connection + "]");
      }
      connection.commit();
    }
  }

  @Override
  public void rollback() throws SQLException {
    if (connection != null && !connection.getAutoCommit()) {
      if (log.isDebugEnabled()) {
        log.debug("Rolling back JDBC Connection [" + connection + "]");
      }
      connection.rollback();
    }
  }

  @Override
  public void close() throws SQLException {
    if (connection != null) {
      resetAutoCommit();
      if (log.isDebugEnabled()) {
        log.debug("Closing JDBC Connection [" + connection + "]");
      }
      connection.close();
    }
  }

  protected void setDesiredAutoCommit(boolean desiredAutoCommit) {
    try {
      if (connection.getAutoCommit() != desiredAutoCommit) {
        if (log.isDebugEnabled()) {
          log.debug("Setting autocommit to " + desiredAutoCommit + " on JDBC Connection [" + connection + "]");
        }
        connection.setAutoCommit(desiredAutoCommit);
      }
    } catch (SQLException e) {
      // Only a very poorly implemented driver would fail here,
      // and there's not much we can do about that.
      throw new TransactionException("Error configuring AutoCommit.  "
          + "Your driver may not support getAutoCommit() or setAutoCommit(). "
          + "Requested setting: " + desiredAutoCommit + ".  Cause: " + e, e);
    }
  }

  protected void resetAutoCommit() {
    try {
      if (!connection.getAutoCommit()) {
        // MyBatis does not call commit/rollback on a connection if just selects were performed.
        // Some databases start transactions with select statements
        // and they mandate a commit/rollback before closing the connection.
        // A workaround is setting the autocommit to true before closing the connection.
        // Sybase throws an exception here.
        if (log.isDebugEnabled()) {
          log.debug("Resetting autocommit to true on JDBC Connection [" + connection + "]");
        }
        connection.setAutoCommit(true);
      }
    } catch (SQLException e) {
      if (log.isDebugEnabled()) {
        log.debug("Error resetting autocommit to true "
          + "before closing the connection.  Cause: " + e);
      }
    }
  }

  protected void openConnection() throws SQLException {
    if (log.isDebugEnabled()) {
      log.debug("Opening JDBC Connection");
    }
    connection = dataSource.getConnection();
    if (level != null) {
      connection.setTransactionIsolation(level.getLevel());
    }
    setDesiredAutoCommit(autoCommmit);
  }

  @Override
  public Integer getTimeout() throws SQLException {
    return null;
  }
}
```
### 创建了JdbcTransaction对象之后`Executor executor = configuration.newExecutor(tx, execType)`创建sql执行器，为SimpleExecutor
```java
public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    executorType = executorType == null ? defaultExecutorType : executorType;
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    Executor executor;
    if (ExecutorType.BATCH == executorType) {
      executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
      executor = new ReuseExecutor(this, transaction);
    } else {
      executor = new SimpleExecutor(this, transaction);
    }
    if (cacheEnabled) {
      executor = new CachingExecutor(executor);
    }
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
  }
```

### 执行器SimpleExecutor源码
```java
public class SimpleExecutor extends BaseExecutor {
    
      public SimpleExecutor(Configuration configuration, Transaction transaction) {
        super(configuration, transaction);
      }
    
      @Override
      public int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
        Statement stmt = null;
        try {
          Configuration configuration = ms.getConfiguration();
          StatementHandler handler = configuration.newStatementHandler(this, ms, parameter, RowBounds.DEFAULT, null, null);
          stmt = prepareStatement(handler, ms.getStatementLog());
          return handler.update(stmt);
        } finally {
          closeStatement(stmt);
        }
      }
    
      @Override
      public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
        Statement stmt = null;
        try {
          Configuration configuration = ms.getConfiguration();
          StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
          stmt = prepareStatement(handler, ms.getStatementLog());
          return handler.<E>query(stmt, resultHandler);
        } finally {
          closeStatement(stmt);
        }
      }
    
      @Override
      protected <E> Cursor<E> doQueryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds, BoundSql boundSql) throws SQLException {
        Configuration configuration = ms.getConfiguration();
        StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, null, boundSql);
        Statement stmt = prepareStatement(handler, ms.getStatementLog());
        return handler.<E>queryCursor(stmt);
      }
    
      @Override
      public List<BatchResult> doFlushStatements(boolean isRollback) throws SQLException {
        return Collections.emptyList();
      }
    
      private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
        Statement stmt;
        Connection connection = getConnection(statementLog);
        stmt = handler.prepare(connection, transaction.getTimeout());
        handler.parameterize(stmt);
        return stmt;
      }
}
```

### 返回的SqlSession为DefaultSqlSession，封装了查询方法selectOne，selectMap，selectList插入方法insert，更新方法update，删除方法delete源码如下
```java
public class DefaultSqlSession implements SqlSession {

  private final Configuration configuration;
  private final Executor executor;

  private final boolean autoCommit;
  private boolean dirty;
  private List<Cursor<?>> cursorList;

  public DefaultSqlSession(Configuration configuration, Executor executor, boolean autoCommit) {
    this.configuration = configuration;
    this.executor = executor;
    this.dirty = false;
    this.autoCommit = autoCommit;
  }

  public DefaultSqlSession(Configuration configuration, Executor executor) {
    this(configuration, executor, false);
  }

  @Override
  public <T> T selectOne(String statement) {
    return this.<T>selectOne(statement, null);
  }

  @Override
  public <T> T selectOne(String statement, Object parameter) {
    // Popular vote was to return null on 0 results and throw exception on too many.
    List<T> list = this.<T>selectList(statement, parameter);
    if (list.size() == 1) {
      return list.get(0);
    } else if (list.size() > 1) {
      throw new TooManyResultsException("Expected one result (or null) to be returned by selectOne(), but found: " + list.size());
    } else {
      return null;
    }
  }

  @Override
  public <K, V> Map<K, V> selectMap(String statement, String mapKey) {
    return this.selectMap(statement, null, mapKey, RowBounds.DEFAULT);
  }

  @Override
  public <K, V> Map<K, V> selectMap(String statement, Object parameter, String mapKey) {
    return this.selectMap(statement, parameter, mapKey, RowBounds.DEFAULT);
  }

  @Override
  public <K, V> Map<K, V> selectMap(String statement, Object parameter, String mapKey, RowBounds rowBounds) {
    final List<? extends V> list = selectList(statement, parameter, rowBounds);
    final DefaultMapResultHandler<K, V> mapResultHandler = new DefaultMapResultHandler<K, V>(mapKey,
        configuration.getObjectFactory(), configuration.getObjectWrapperFactory(), configuration.getReflectorFactory());
    final DefaultResultContext<V> context = new DefaultResultContext<V>();
    for (V o : list) {
      context.nextResultObject(o);
      mapResultHandler.handleResult(context);
    }
    return mapResultHandler.getMappedResults();
  }

  @Override
  public <T> Cursor<T> selectCursor(String statement) {
    return selectCursor(statement, null);
  }

  @Override
  public <T> Cursor<T> selectCursor(String statement, Object parameter) {
    return selectCursor(statement, parameter, RowBounds.DEFAULT);
  }

  @Override
  public <T> Cursor<T> selectCursor(String statement, Object parameter, RowBounds rowBounds) {
    try {
      MappedStatement ms = configuration.getMappedStatement(statement);
      Cursor<T> cursor = executor.queryCursor(ms, wrapCollection(parameter), rowBounds);
      registerCursor(cursor);
      return cursor;
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }

  @Override
  public <E> List<E> selectList(String statement) {
    return this.selectList(statement, null);
  }

  @Override
  public <E> List<E> selectList(String statement, Object parameter) {
    return this.selectList(statement, parameter, RowBounds.DEFAULT);
  }

  @Override
  public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
    try {
      MappedStatement ms = configuration.getMappedStatement(statement);
      return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }

  @Override
  public void select(String statement, Object parameter, ResultHandler handler) {
    select(statement, parameter, RowBounds.DEFAULT, handler);
  }

  @Override
  public void select(String statement, ResultHandler handler) {
    select(statement, null, RowBounds.DEFAULT, handler);
  }

  @Override
  public void select(String statement, Object parameter, RowBounds rowBounds, ResultHandler handler) {
    try {
      MappedStatement ms = configuration.getMappedStatement(statement);
      executor.query(ms, wrapCollection(parameter), rowBounds, handler);
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }

  @Override
  public int insert(String statement) {
    return insert(statement, null);
  }

  @Override
  public int insert(String statement, Object parameter) {
    return update(statement, parameter);
  }

  @Override
  public int update(String statement) {
    return update(statement, null);
  }

  @Override
  public int update(String statement, Object parameter) {
    try {
      dirty = true;
      MappedStatement ms = configuration.getMappedStatement(statement);
      return executor.update(ms, wrapCollection(parameter));
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error updating database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }

  @Override
  public int delete(String statement) {
    return update(statement, null);
  }

  @Override
  public int delete(String statement, Object parameter) {
    return update(statement, parameter);
  }

  @Override
  public void commit() {
    commit(false);
  }

  @Override
  public void commit(boolean force) {
    try {
      executor.commit(isCommitOrRollbackRequired(force));
      dirty = false;
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error committing transaction.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }

  @Override
  public void rollback() {
    rollback(false);
  }

  @Override
  public void rollback(boolean force) {
    try {
      executor.rollback(isCommitOrRollbackRequired(force));
      dirty = false;
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error rolling back transaction.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }

  @Override
  public List<BatchResult> flushStatements() {
    try {
      return executor.flushStatements();
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error flushing statements.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }

  @Override
  public void close() {
    try {
      executor.close(isCommitOrRollbackRequired(false));
      closeCursors();
      dirty = false;
    } finally {
      ErrorContext.instance().reset();
    }
  }

  private void closeCursors() {
    if (cursorList != null && cursorList.size() != 0) {
      for (Cursor<?> cursor : cursorList) {
        try {
          cursor.close();
        } catch (IOException e) {
          throw ExceptionFactory.wrapException("Error closing cursor.  Cause: " + e, e);
        }
      }
      cursorList.clear();
    }
  }

  @Override
  public Configuration getConfiguration() {
    return configuration;
  }

  @Override
  public <T> T getMapper(Class<T> type) {
    return configuration.<T>getMapper(type, this);
  }

  @Override
  public Connection getConnection() {
    try {
      return executor.getTransaction().getConnection();
    } catch (SQLException e) {
      throw ExceptionFactory.wrapException("Error getting a new connection.  Cause: " + e, e);
    }
  }

  @Override
  public void clearCache() {
    executor.clearLocalCache();
  }

  private <T> void registerCursor(Cursor<T> cursor) {
    if (cursorList == null) {
      cursorList = new ArrayList<Cursor<?>>();
    }
    cursorList.add(cursor);
  }

  private boolean isCommitOrRollbackRequired(boolean force) {
    return (!autoCommit && dirty) || force;
  }

  private Object wrapCollection(final Object object) {
    if (object instanceof Collection) {
      StrictMap<Object> map = new StrictMap<Object>();
      map.put("collection", object);
      if (object instanceof List) {
        map.put("list", object);
      }
      return map;
    } else if (object != null && object.getClass().isArray()) {
      StrictMap<Object> map = new StrictMap<Object>();
      map.put("array", object);
      return map;
    }
    return object;
  }

  public static class StrictMap<V> extends HashMap<String, V> {

    private static final long serialVersionUID = -5741767162221585340L;

    @Override
    public V get(Object key) {
      if (!super.containsKey(key)) {
        throw new BindingException("Parameter '" + key + "' not found. Available parameters are " + this.keySet());
      }
      return super.get(key);
    }

  }
}
```
#  通过SqlSession获取Mapper

### SqlSession为DefaultSqlSession。在DefaultSqlSession的getMapper方法。委派给configuration的getMapper方法
```java
  public <T> T getMapper(Class<T> type) {
    return configuration.<T>getMapper(type, this);
  }
```
### Configuration的getMapper方法
```java
 public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    return mapperRegistry.getMapper(type, sqlSession);
  }
```
### 最后委派给MapperRegistry的getMapper方法。从已注册的信息之中获取MapperProxyFactory，然后创建Mapper对象
```java
 public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
      return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
  }
```
### 使用Mapper代理工厂MapperProxyFactory创建Mapper对象
```java
public class MapperProxyFactory<T> {

  private final Class<T> mapperInterface;
  private final Map<Method, MapperMethod> methodCache = new ConcurrentHashMap<Method, MapperMethod>();

  public MapperProxyFactory(Class<T> mapperInterface) {
    this.mapperInterface = mapperInterface;
  }

  public Class<T> getMapperInterface() {
    return mapperInterface;
  }

  public Map<Method, MapperMethod> getMethodCache() {
    return methodCache;
  }

  @SuppressWarnings("unchecked")
  protected T newInstance(MapperProxy<T> mapperProxy) {
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }

  public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }

}
```
### 使用jdk代理方式创建代理对象MapperProxy，调用对象的方法实际调用代理对象的invoke方法实现代理服务。
```java
public class MapperProxy<T> implements InvocationHandler, Serializable {

  private static final long serialVersionUID = -6424540398559729838L;
  private final SqlSession sqlSession;
  private final Class<T> mapperInterface;
  private final Map<Method, MapperMethod> methodCache;

  public MapperProxy(SqlSession sqlSession, Class<T> mapperInterface, Map<Method, MapperMethod> methodCache) {
    this.sqlSession = sqlSession;
    this.mapperInterface = mapperInterface;
    this.methodCache = methodCache;
  }

  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
        // 如果是Object方法则直接执行方法并返回
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, args);
      } else if (isDefaultMethod(method)) {
          // 如果是接口默认方法则直接执行方法并返回
        return invokeDefaultMethod(proxy, method, args);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    return mapperMethod.execute(sqlSession, args);
  }

  private MapperMethod cachedMapperMethod(Method method) {
    MapperMethod mapperMethod = methodCache.get(method);
    if (mapperMethod == null) {
      mapperMethod = new MapperMethod(mapperInterface, method, sqlSession.getConfiguration());
      methodCache.put(method, mapperMethod);
    }
    return mapperMethod;
  }

  @UsesJava7
  private Object invokeDefaultMethod(Object proxy, Method method, Object[] args)
      throws Throwable {
    final Constructor<MethodHandles.Lookup> constructor = MethodHandles.Lookup.class
        .getDeclaredConstructor(Class.class, int.class);
    if (!constructor.isAccessible()) {
      constructor.setAccessible(true);
    }
    final Class<?> declaringClass = method.getDeclaringClass();
    return constructor
        .newInstance(declaringClass,
            MethodHandles.Lookup.PRIVATE | MethodHandles.Lookup.PROTECTED
                | MethodHandles.Lookup.PACKAGE | MethodHandles.Lookup.PUBLIC)
        .unreflectSpecial(method, declaringClass).bindTo(proxy).invokeWithArguments(args);
  }

  /**
   * Backport of java.lang.reflect.Method#isDefault()
   */
  private boolean isDefaultMethod(Method method) {
    return (method.getModifiers()
        & (Modifier.ABSTRACT | Modifier.PUBLIC | Modifier.STATIC)) == Modifier.PUBLIC
        && method.getDeclaringClass().isInterface();
  }
}

```
### 获取了Mapper之后，就可以使用Mapper了，接下来看看Mapper方法调用具体实现。
* Mapper方法调用具体实现
### 调用Mapper方法会调用代理对象的invoke方法，判断然后创建MapperMethod对象封装代理对象方法。然后处理分派到MapperMethod的
execute方法之中
```java
private MapperMethod cachedMapperMethod(Method method) {
    MapperMethod mapperMethod = methodCache.get(method);
    if (mapperMethod == null) {
      mapperMethod = new MapperMethod(mapperInterface, method, sqlSession.getConfiguration());
      methodCache.put(method, mapperMethod);
    }
    return mapperMethod;
  }
```
### MapperMethod的execute方法实现。通过MapperMethod不同的command分发调用SqlSession方法实现。
```java
public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    switch (command.getType()) {
      case INSERT: {
      Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.insert(command.getName(), param));
        break;
      }
      case UPDATE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.update(command.getName(), param));
        break;
      }
      case DELETE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.delete(command.getName(), param));
        break;
      }
      case SELECT:
        if (method.returnsVoid() && method.hasResultHandler()) {
          executeWithResultHandler(sqlSession, args);
          result = null;
        } else if (method.returnsMany()) {
          result = executeForMany(sqlSession, args);
        } else if (method.returnsMap()) {
          result = executeForMap(sqlSession, args);
        } else if (method.returnsCursor()) {
          result = executeForCursor(sqlSession, args);
        } else {
          Object param = method.convertArgsToSqlCommandParam(args);
          result = sqlSession.selectOne(command.getName(), param);
        }
        break;
      case FLUSH:
        result = sqlSession.flushStatements();
        break;
      default:
        throw new BindingException("Unknown execution method for: " + command.getName());
    }
    if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
      throw new BindingException("Mapper method '" + command.getName() 
          + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
    }
    return result;
  }
```

### MyBatis所有insert,delete方法最后都会分派到update(String,Object)方法
```java
 @Override
  public int update(String statement, Object parameter) {
    try {
      dirty = true;
      MappedStatement ms = configuration.getMappedStatement(statement);
      return executor.update(ms, wrapCollection(parameter));
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error updating database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```
### 最后分派给执行器.在BaseExecutor之中的update方法。
```java
 @Override
  public int update(MappedStatement ms, Object parameter) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing an update").object(ms.getId());
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    clearLocalCache();
    //doUpdate为抽象方法
    return doUpdate(ms, parameter);
  }
```
### SimpleExecutor的doUpdate方法实现
```java
@Override
  public int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();
      StatementHandler handler = configuration.newStatementHandler(this, ms, parameter, RowBounds.DEFAULT, null, null);
      stmt = prepareStatement(handler, ms.getStatementLog());
      return handler.update(stmt);
    } finally {
      closeStatement(stmt);
    }
  }
  
  // 分派到PreparedStatementHandler
  @Override
    public int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
      Statement stmt = null;
      try {
        Configuration configuration = ms.getConfiguration();
        StatementHandler handler = configuration.newStatementHandler(this, ms, parameter, RowBounds.DEFAULT, null, null);
        stmt = prepareStatement(handler, ms.getStatementLog());
        return handler.update(stmt);
      } finally {
        closeStatement(stmt);
      }
    }
```


```java
 // DefaultSqlSession之中的selectList方法
@Override
public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
    try {
      MappedStatement ms = configuration.getMappedStatement(statement);
      return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
}
  //分派到BaseExecutor的query方法
@Override
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
  BoundSql boundSql = ms.getBoundSql(parameter);
  CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
  return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
}
// 分派到子类SimpleExecutor
  @Override
  public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();
      StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
      stmt = prepareStatement(handler, ms.getStatementLog());
      return handler.<E>query(stmt, resultHandler);
    } finally {
      closeStatement(stmt);
    }
  }
```
  // 最后分派到PreparedStatementHandler
 ```java
@Override
public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
  PreparedStatement ps = (PreparedStatement) statement;
  ps.execute();
  return resultSetHandler.<E> handleResultSets(ps);
}
```



