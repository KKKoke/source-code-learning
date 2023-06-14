# MyBatis 源码小结

## 一、JDBC 详解

### 1.JDBC 的重要性

MyBatis 框架对 JDBC 做了轻量级的封装，所以要看懂 MyBatis 的源码就必须熟练掌握 JDBC API 的使用。

### 2.什么是 JDBC

JDBC 是 Java 提供的访问关系型数据的接口。JDBC 提供了一种自然的、易于使用的 Java 语言与数据库交互的接口。

使用 JDBC 操作数据源大致需要以下几个步骤：

1）与数据源建立连接

2）执行 SQL 语句

3）检索 SQL 执行结果

4）关闭连接

示例代码：

```java
//1.加载驱动程序
Class.forName("com.mysql.jdbc.Driver");
//2. 获得数据库连接
Connection conn = DriverManager.getConnection(URL, USER, PASSWORD);
//3. 操作数据库，实现增删改查
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery("SELECT user_name, age FROM user");
//4. 关闭连接
conn.close();
```

### 3.步骤详解

#### 3.1.建立数据源连接

##### DriverManager

在第一次尝试通过 URL 连接数据源时，DriverManager 会自动加载 CLASSPATH 下所有的 JDBC 驱动。DriverManager 提供了一系列重载的 getConnection() 方法来获取 Connection 对象。

```java
Connection conn = DriverManager.getConnection(URL, USER, PASSWORD);
```

##### DataSource

它相比于 DriverManager 更受欢迎，因为它提供了更多底层数据源相关的实现，而且对于应用来说，不需要关注 JDBC 驱动的实现。一个 DataSource 对象的属性被设置后，它就代表一个特定的数据源。需要注意的是，JDBC API 中只提供了DataSource 接口，没有提供 DataSource 的具体实现，DataSource 具体的实现由 JDBC 驱动程序提供。

```java
Connection connection = dataSource.getConnection();
```

#### 3.2.执行 SQL 语句

获取到 JDBC 中的 Connection 对象之后，我们可以通过 Connection 对象设置事务属性，并且可以通过 Connection 接口中提供的方法创建 Statement、PreparedStatement 或者 CallableStatement 对象。

Statement 接口可以理解为 JDBC API 中提供的 SQL 语句的执行器，我们可以调用 Statement 接口中定义的executeQuery() 方法执行查询操作，调用 executeUpdate() 方法执行更新操作，另外还可以调用 executeBatch() 方法执行批量处理。当不知道 SQL 类型的时候也可以直接调用 execute() 方法进行统一的操作。最后通过 Statement 接口提供的getResultSet() 方法来获取查询结果集，或者通过 getUpdateCount() 方法来获取更新操作影响的行数。

#### 3.3.处理 SQL 执行结果

JDBC API 中提供了 ResultSet 接口，该接口的实现类封装 SQL 查询的结果，我们可以对 ResultSet 对象进行遍历，然后通过 ResultSet 提供的一系列 getXXX() 方法（例如 getString）获取查询结果集。

### 4.组件详解

#### 4.1.各组件之间的关系

![image-20230613160553068](./imgs/image-20230613160553068.png)

#### 4.2.Connection

一个 Connection 对象表示通过 JDBC 驱动与数据源建立的连接，这里的数据源可以是关系型数据库管理系统（DBMS）、文件系统或者其他通过 JDBC 驱动访问的数据。使用 JDBC API 的应用程序可能需要维护多个 Connection 对象，一个Connection 对象可能访问多个数据源，也可能访问单个数据源。

从 JDBC 驱动的角度来看，Connection 对象表示客户端会话，因此它需要一些相关的状态信息，例如用户 Id、一组 SQL 语句和会话中使用的结果集以及事务隔离级别等信息。

我们可以通过两种方式获取 JDBC 中的 Connection 对象：（1）通过 JDBC API 中提供的 DriverManager 类获取。（2）通过 DataSource 接口的实现类获取。

使用 DataSource 的具体实现获取 Connection 对象是比较推荐的一种方式，因为它增强了应用程序的可移植性，使代码维护更加容易，并且使应用程序能够透明地使用连接池和处理分布式事务。

#### 4.3.Statement

Statement 接口中定义了执行 SQL 语句的方法，这些方法不支持参数输入，Statement 的主要作用是与数据库进行交互，该接口中定义了一些数据库操作以及检索SQL执行结果相关的方法。

PreparedStatement 继承自 Statement 接口中增加了一些方法，可以为占位符设置值。PreparedStatement 的实例表示可以被预编译的SQL语句，执行一次后，后续多次执行时效率会比较高。使用 PreparedStatement 实例执行 SQL 语句时，可以使用“?”作为参数占位符，然后使用 PreparedStatement 接口中提供的方法为占位符设置参数值。

CallableStatement 接口继承自 PreparedStatement，在此基础上增加了调用存储过程以及检索存储过程调用结果的方法。CallableStatement 接口中新增了一些额外的方法允许参数通过名称注册和检索。

#### 4.4.ResultSet

ResultSet 接口是 JDBC API 中另一个比较重要的组件，提供了检索和操作 SQL 执行结果相关的方法。

ResultSet 对象的类型主要体现在两个方面：（1）游标可操作的方式。（2）ResultSet 对象的修改对数据库的影响。

#### 4.5.DatabaseMetaData

DatabaseMetaData 接口是由 JDBC 驱动程序实现的，用于提供底层数据源相关的信息。该接口主要用于为应用程序或工具确定如何与底层数据源交互。应用程序也可以使用 DatabaseMetaData 接口提供的方法获取数据源信息。

DatabaseMetaData 接口中包含超过150个方法，根据这些方法的类型可以分为以下几类：

1）获取数据源信息。

2）确定数据源是否支持某一特性或功能。

3）获取数据源的限制。

4）确定数据源包含哪些SQL对象以及这些对象的属性。

5）获取数据源对事务的支持。

DatabaseMetaData 对象的创建比较简单，需要依赖 Connection 对象。Connection 对象中提供了一个 getMetadata() 方法，用于创建 DatabaseMetaData 对象。一旦创建了 DatabaseMetaData 对象，我们就可以通过该对象动态地获取数据源相关的信息了。

### 5.JDBC 事务

事务用于提供数据完整性、正确的应用程序语义和并发访问的数据一致性。所有遵循JDBC规范的驱动程序都需要提供事务支持。JDBC 事务主要涉及以下概念：自动提交模式、事务隔离级别、保存点。

#### 5.1.自动提交模式

Connection 对象的 autoCommit 属性决定什么时候结束一个事务。启用自动提交后，会在每个 SQL 语句执行完毕后自动提交事务。当 Connection 对象创建时，默认情况下，事务自动提交是开启的。Connection 接口中提供了一个setAutoCommit() 方法，可以禁用事务自动提交。此时，需要显式地调用 Connection 接口提供 commit() 方法提交事务，或者调用 rollback() 方法回滚事务。禁用事务自动提交适用于需要将多个 SQL 语句作为一个事务提交或者事务由应用服务器管理。

#### 5.2.事务隔离级别

主要的事务隔离级别如下：

1）TRANSACTION_NONE：表示驱动不支持事务，这意味着它是不兼容 JDBC 规范的驱动程序。

2）TRANSACTION_READ_UNCOMMITTED：允许事务读取未提交更改的数据，这意味着可能会出现脏读、不可重复读、幻读等现象。

3）TRANSACTION_READ_COMMITTED：表示在事务中进行的任何数据更改，在提交之前对其他事务是不可见的。这样可以防止脏读，但是不能解决不可重复读和幻读的问题。

4）TRANSACTION_REPEATABLE_READ：该事务隔离级别能够解决脏读和不可重复读问题，但是不能解决幻读问题。

5）TRANSACTION_SERIALIZABLE：该事务隔离级别下，所有事务串行执行，能够有效解决脏读、不可重复读和幻读问题，但是并发效率较低。

Connection 对象的默认事务级别由 JDBC 驱动程序指定。通常它是底层数据源支持的默认事务隔离级别。Connection 接口中提供了一个 setTransactionIsolation() 方法，允许 JDBC 客户端设置 Connection 对象的事务隔离级别。新设置的事务隔离级别会在之后的会话中生效。

#### 5.3.保存点

保存点通过在事务中标记一个中间的点来对事务进行更细粒度的控制，一旦设置保存点，事务就可以回滚到保存点，而不影响保存点之前的操作。DatabaseMetaData 接口提供了 supportsSavepoints() 方法，用于判断 JDBC 驱动是否支持保存点。Connection 接口中提供了 setSavepoint() 方法用于在当前事务中设置保存点，如果 setSavepoint() 方法在事务外调用，则调用该方法后会在 setSavepoint() 方法调用处开启一个新的事务。setSavepoint() 方法的返回值是一个 Savepoint 对象，该对象可作为 Connection 对象 rollback() 方法的参数，用于回滚到对应的保存点。

## 二、MyBatis 常用工具类

## 三、MyBatis 核心组件

### 1.MyBatis 操作数据库示例

```java
public void testSqlSessionFactory() throws IOException {
    // 1. 从SqlSessionFactory中获取SqlSession
    SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(Resources.getResourceAsReader("mybatis-config-datasource.xml"));
    SqlSession sqlSession = sqlSessionFactory.openSession();

    // 2. 获取映射器对象
    IUserDao userDao = sqlSession.getMapper(IUserDao.class);

    // 3. 测试验证
    User user = userDao.queryUserInfoById(1L);
    logger.info("测试结果：{}", JSON.toJSONString(user));
}
```

### 2.各组件之间的关系

![image-20230613165039411](./imgs/image-20230613165039411.png)

### 3.SqlSession

SqlSession 是 MyBatis 中提供的与数据库交互的接口，SqlSession 实例通过工厂模式创建。而 SqlSession 的创建又依赖于 SqlSessionFactory，SqlSessionFactory 的创建又依赖于 SqlSessionFactoryBuilder 类。SqlSession 是 MyBatis 提供的面向用户的操作数据库 API。

### 4.Configuration

Configuration 用于描述 MyBatis 的主配置信息，其他组件需要获取配置信息时，直接通过 Configuration 对象获取。除此之外，MyBatis 在应用启动时，将 Mapper 配置信息、类型别名、TypeHandler 等注册到 Configuration 组件中，其他组件需要这些信息时，也可以从 Configuration 对象中获取。也就是说 Configuration 是 MyBatis 中各组件之间相互连接的纽带。

MyBatis 框架的配置信息有两种，一种是配置 MyBatis 框架属性的主配置文件；另一种是配置执行 SQL 语句的 Mapper 配置文件。Configuration 的作用是描述MyBatis主配置文件的信息。

Configuration 存储了各种属性控制 MyBatis 的行为，如：cacheEnabled、lazyLoadingEnabled、useGeneratedKeys 等，同时还作为容器存放 TypeHandler（类型处理器）、TypeAlias（类型别名）、Mapper 接口及 Mapper SQL 配置信息。这些信息在 MyBatis 框架启动时注册到 Configuration 组件中。

### 5.Executor

SqlSession 是 MyBatis 提供给用户的操作数据库的 API，而 Executor 才是 MyBatis 真正的 SQL 执行器，MyBatis 中对数据库所有的增删改查操作都是由 Executor 组件完成的。

MyBatis 提供了3种不同的 Executor，分别为 SimpleExecutor、ResueExecutor、BatchExecutor，这些 Executor 都继承至BaseExecutor，BaseExecutor 中定义的方法的执行流程及通用的处理逻辑，具体的方法由子类来实现，是典型的模板方法模式的应用。

SimpleExecutor 是基础的 Executor，能够完成基本的增删改查操作，ResueExecutor 对 JDBC 中的Statement 对象做了缓存，当执行相同的 SQL 语句时，直接从缓存中取出 Statement 对象进行复用，避免了频繁创建和销毁 Statement 对象，从而提升系统性能，这是享元思想的应用。

BatchExecutor 则会对调用同一个 Mapper 执行的 update、insert 和 delete 操作，调用 Statement 对象的批量操作功能。

当 MyBatis 开启了二级缓存功能时，CachingExecutor 会使用对 SimpleExecutor、ResueExecutor、BatchExecutor 进行装饰，为查询操作增加二级缓存功能，这是装饰器模式的应用。

![image-20230613173606314](./imgs/image-20230613173606314.png)

### 6.MappedStatement

MappedStatement 用于描述 Mapper 中的 SQL 配置信息，是对 Mapper XML 配置文件中<select|update|delete|insert>等标签或者@Select/@Update 等注解配置信息的封装。

### 7.StatementHandler

StatementHandler 组件封装了对 JDBC Statement 的操作，例如设置 Statement 对象的 fetchSize 属性、设置查询超时时间、调用 JDBC Statement 与数据库交互等。

BaseStatementHandler 是一个抽象类，封装了通用的处理逻辑及方法执行流程，具体方法的实现由子类完成，这里使用到了设计模式中的模板方法模式。SimpleStatementHandler 继承至 BaseStatementHandler，封装了对 JDBC Statement对象的操作，PreparedStatementHandler 封装了对 JDBC PreparedStatement 对象的操作，而 CallableStatementHandler则封装了对 JDBC CallableStatement 对象的操作。RoutingStatementHandler 会根据 Mapper 配置中的 statementType 属性（取值为 STATEMENT、PREPARED 或 CALLABLE）创建对应的 StatementHandler 实现。

![image-20230613173632271](./imgs/image-20230613173632271.png)

### 8.TypeHandler

TypeHandler 的作用是解决 JDBC 类型与 Java 类型之间的转换。MyBatis 通过 TypeHandlerRegistry 建立 JDBC 类型、Java 类型与 TypeHandler 之间的映射关系。在 TypeHandlerRegistry 的构造方法中就注册了许多 TypeHandler。

### 9.ParameterHandler

当使用 PreparedStatement 或者 CallableStatement 对象时，如果 SQL 语句中有参数占位符，在执行 SQL 语句之前，就需要为参数占位符设置值。ParameterHandler 的作用是在 PreparedStatementHandler 和 CallableStatementHandler 操作对应的 Statement 执行数据库交互之前为参数占位符设置值。

### 10.ResultSetHandler

ResultSetHandler 用于在 StatementHandler 对象执行完查询操作或存储过程后，对结果集或存储过程的执行结果进行处理。

## 四、SqlSession 创建过程

### 1.Configuration 实例创建过程

MyBatis 通过 XMLConfigBuilder 类完成 Configuration 对象的构建工作。

示例代码：

```java
public void testParseConfiguration() throws IOException {
    Reader reader = Resources.getResourceAsReader("mybatis-config-datasource.xml");
    XMLConfigBuilder builder = new XMLConfigBuilder(reader);
    Configuration configuration = builder.parse();
}
```

首先使用 Resources 工具加载配置文件，再将加载好的配置文件作为参数传递到 XMLConfigBuilder 中进行解析。XMLConfigBuilder 采用的是 Xpath 去解析 XML 配置文件，使用的类是 XPathParser。

```java
public Configuration parse() {
    if (parsed) {
        throw new BuilderException("Each XMLConfigBuilder can only be used once.");
    }
    parsed = true;
    // 找到配置文件中的 <configuration> 标签进行解析
    parseConfiguration(parser.evalNode("/configuration"));
    return configuration;
}
```

```java
private void parseConfiguration(XNode root) {
    try {
        //issue #117 read properties first
        propertiesElement(root.evalNode("properties"));
        Properties settings = settingsAsProperties(root.evalNode("settings"));
        loadCustomVfs(settings);
        typeAliasesElement(root.evalNode("typeAliases"));
        pluginElement(root.evalNode("plugins"));
        objectFactoryElement(root.evalNode("objectFactory"));
        objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
        reflectorFactoryElement(root.evalNode("reflectorFactory"));
        settingsElement(settings);
        // read it after objectFactory and objectWrapperFactory issue #631
        environmentsElement(root.evalNode("environments"));
        databaseIdProviderElement(root.evalNode("databaseIdProvider"));
        typeHandlerElement(root.evalNode("typeHandlers"));
        mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
      	throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
}
```

这里便是根据不同的标签去解析对应的配置信息，并将所有的配置信息存在 Configuration 对象中。

### 2.SqlSession 示例创建过程

代码示例：

```java
public void testSqlSession() throws IOException {
    Reader reader = Resources.getResourceAsReader("mybatis-config-datasource.xml");
    SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(reader);
    SqlSession sqlSession = sqlSessionFactory.openSession();
}
```

```java
public SqlSessionFactory build(Reader reader, String environment, Properties properties) {
    try {
        XMLConfigBuilder parser = new XMLConfigBuilder(reader, environment, properties);
        return build(parser.parse());
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

public SqlSessionFactory build(Configuration config) {
    return new DefaultSqlSessionFactory(config);
}
```

```java
public class DefaultSqlSessionFactory implements SqlSessionFactory {

  private final Configuration configuration;

    public DefaultSqlSessionFactory(Configuration configuration) {
    	this.configuration = configuration;
    }
}
```

这里先通过 XMLConfigBuilder 解析出配置信息，生成 Configuration，再作为参数传递给 SqlSessionFactoryBuilder#build() 方法构建 SqlSessionFactory。

```java
public SqlSession openSession() {
    return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, false);
}
```

```java
private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
Transaction tx = null;
    try {
        // 获取 MyBatis 主配置文件配置的环境信息
        final Environment environment = configuration.getEnvironment();
        // 创建事务管理工厂
        final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
        tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
        // 根据主配置文件中指定的 Executor 类型创建对应的 Executor
        final Executor executor = configuration.newExecutor(tx, execType);
        // 创建 DefaultSqlSession
        return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
    	closeTransaction(tx); // may have fetched a connection so lets call close()
    	throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
    	ErrorContext.instance().reset();
    }
}
```

事务管理器对象创建完毕后，接着调用 Configuration#newExecutor() 方法，根据 MyBatis 主配置文件中指定的 Executor 类型创建对应的 Executor 对象，最后以 Executor 对象和 Configuration 对象作为参数，通过 Java 中的 new 关键字创建一个 DefaultSqlSession 对象。DefaultSqlSession 对象中持有 Executor 对象的引用，真正执行 SQL 操作的是 Executor 对象。

## 五、SqlSession 执行 Mapper 过程

### 1.Mapper 接口的注册过程

```java
public static void main(String[] args) throws IOException {
    Reader reader = Resources.getResourceAsReader("mybatis-config-datasource.xml");
    SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(reader);
    SqlSession sqlSession = sqlSessionFactory.openSession();
    IUserDao userDao = sqlSession.getMapper(IUserDao.class);
    User user = userDao.queryUserInfoById(1L);
}
```

IUserDao 是一个接口，调用 SqlSession#getMapper() 获得的是一个动态代理对象。

#### 1.1.MapperProxy

MyBatis 通过 MapperProxy 类实现动态代理。

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
            if (Object.class.equals(method.getDeclaringClass())) {
            	return method.invoke(this, args);
            } else if (isDefaultMethod(method)) {
            	return invokeDefaultMethod(proxy, method, args);
            }
        } catch (Throwable t) {
        	throw ExceptionUtil.unwrapThrowable(t);
        }
        // 对 Mapper 接口中定义的方法进行封装，生成 MapperMethod 对象
        final MapperMethod mapperMethod = cachedMapperMethod(method);
        return mapperMethod.execute(sqlSession, args);
    }
    //......
}
```

MapperProxy 使用的是 JDK 内置的动态代理，MapperProxy 通过实现 InvocationHandler 接口，定义方法执行拦截逻辑后，还需要调用 Proxy 类的 newProxyInstance() 方法创建代理对象。

#### 1.2.MapperProxyFactory

MyBatis 对这一过程做了封装，使用 MapperProxyFactory 创建 Mapper 动态代理对象。

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

    // 工厂方法
    public T newInstance(SqlSession sqlSession) {
    	final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
    	return newInstance(mapperProxy);
    }
}
```

MapperProxyFactory 类的工厂方法 newInstance() 是非静态的。也就是说，使用 MapperProxyFactory 创建 Mapper 动态代理对象首先需要创建 MapperProxyFactory 实例。那么 MapperProxyFactory 在哪里创建的呢？

#### 1.3.MapperRegistry

在 Configuration 中有一个属性 mapperRegistry。

```java
protected final MapperRegistry mapperRegistry = new MapperRegistry(this);
```

MyBatis 通过 mapperRegistry 属性注册 Mapper 接口与 MapperProxyFactory 对象之间的对应关系。

```java
public class MapperRegistry {

    // Configuration 的引用
    private final Configuration config;
    // 用于注册 Mapper 接口对应的 Class 对象和 MapperProxyFactory 对象对应关系
    private final Map<Class<?>, MapperProxyFactory<?>> knownMappers = new HashMap<Class<?>, MapperProxyFactory<?>>();

    public MapperRegistry(Configuration config) {
        this.config = config;
    }

    // 根据 Mapper 接口 Class 对象获取 Mapper 动态代理对象
    @SuppressWarnings("unchecked")
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

    public <T> boolean hasMapper(Class<T> type) {
        return knownMappers.containsKey(type);
    }

    // 根据 Mapper 接口 Class 对象创建 MapperProxyFactory 对象，并注册到 knownMappers 属性中
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
}
```

MyBatis 框架在应用启动时会解析所有的 Mapper 接口，然后调用 MapperRegistry 对象的 addMapper() 方法将 Mapper 接口信息和对应的 MapperProxyFactory 对象注册到 MapperRegistry 对象中。

### 2.MappedStatement 注册过程

MyBatis 通过 MappedStatement 类描述 Mapper 的 SQL 配置信息。SQL 配置有两种方式：一种是通过 XML 文件配置；另一种是通过 Java 注解，而 Java 注解的本质就是一种轻量级的配置信息。

同样在 Configuration 类中有一个 mappedStatements 属性，该属性用于注册 MyBatis 中所有的 MappedStatement 对象。

```java
protected final Map<String, MappedStatement> mappedStatements = new StrictMap<MappedStatement>("Mapped Statements collection");
```

mappedStatements 属性是一个 Map 对象，它的 Key 为 Mapper SQL 配置的 Id，如果 SQL 是通过 XML 配置的，则 Id 为命名空间加上 <select|update|delete|insert> 标签的 Id，如果 SQL 通过 Java 注解配置，则 Id 为 Mapper 接口的完全限定名（包括包名）加上方法名称。

#### 2.1.XMLConfigBuilder

MyBatis 主配置文件的解析是通过 XMLConfigBuilder 对象来完成的。MappedStatement 就需要解析配置文件中的 \<mappers\> 标签。

```java
private void parseConfiguration(XNode root) {
    try {
        //issue #117 read properties first
        propertiesElement(root.evalNode("properties"));
        Properties settings = settingsAsProperties(root.evalNode("settings"));
        loadCustomVfs(settings);
        typeAliasesElement(root.evalNode("typeAliases"));
        pluginElement(root.evalNode("plugins"));
        objectFactoryElement(root.evalNode("objectFactory"));
        objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
        reflectorFactoryElement(root.evalNode("reflectorFactory"));
        settingsElement(settings);
        // read it after objectFactory and objectWrapperFactory issue #631
        environmentsElement(root.evalNode("environments"));
        databaseIdProviderElement(root.evalNode("databaseIdProvider"));
        typeHandlerElement(root.evalNode("typeHandlers"));
        mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
        throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
}
```

```java
private void mapperElement(XNode parent) throws Exception {
    if (parent != null) {
        for (XNode child : parent.getChildren()) {
            // 通过 <package> 标签指定包名
            if ("package".equals(child.getName())) {
                String mapperPackage = child.getStringAttribute("name");
                configuration.addMappers(mapperPackage);
            } else {
                String resource = child.getStringAttribute("resource");
                String url = child.getStringAttribute("url");
                String mapperClass = child.getStringAttribute("class");
                // 通过 resource 属性指定 XML 文件路径
                if (resource != null && url == null && mapperClass == null) {
                    ErrorContext.instance().resource(resource);
                    InputStream inputStream = Resources.getResourceAsStream(resource);
                    XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
                    mapperParser.parse();
                    // 通过 url 属性指定 XML 文件路径
                } else if (resource == null && url != null && mapperClass == null) {
                    ErrorContext.instance().resource(url);
                    InputStream inputStream = Resources.getUrlAsStream(url);
                    XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());
                    mapperParser.parse();
                    // 通过 class 属性指定 XML 文件路径
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

#### 2.2.XMLMapperBuilder

由上面的代码可以知道，Mapper SQL 配置文件的解析需要借助 XMLMapperBuilder 对象。在 mapperElement() 方法中，首先创建一个 XMLMapperBuilder 对象，然后调用 XMLMapperBuilder 对象的 parse() 方法完成解析。

```java
public void parse() {
    if (!configuration.isResourceLoaded(resource)) {
        // 调用 XpathParser#evalNode()，获取根节点对应的 XNode 对象
        configurationElement(parser.evalNode("/mapper"));
        // 将资源路径添加到 Configuration 对象中
        configuration.addLoadedResource(resource);
        bindMapperForNamespace();
    }

    // 继续解析之前解析出现异常的 ResultMap 对象
    parsePendingResultMaps();
    // 继续解析之前解析出现异常的 CacheRef 对象
    parsePendingCacheRefs();
    // 继续解析之前解析出现异常的 <select|update|delete|insert> 标签
    parsePendingStatements();
}
```

```java
private void configurationElement(XNode context) {
    try {
        // 获取命名空间
        String namespace = context.getStringAttribute("namespace");
        if (namespace == null || namespace.equals("")) {
            throw new BuilderException("Mapper's namespace cannot be empty");
        }
        // 设置当前正在解析的 Mapper 配置的命名空间
        builderAssistant.setCurrentNamespace(namespace);
        cacheRefElement(context.evalNode("cache-ref"));
        cacheElement(context.evalNode("cache"));
        parameterMapElement(context.evalNodes("/mapper/parameterMap"));
        resultMapElements(context.evalNodes("/mapper/resultMap"));
        sqlElement(context.evalNodes("/mapper/sql"));
        buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
    } catch (Exception e) {
        throw new BuilderException("Error parsing Mapper XML. Cause: " + e, e);
    }
}
```

这里查看 <select|update|delete|insert> 标签的解析。

```java
 private void buildStatementFromContext(List<XNode> list) {
    if (configuration.getDatabaseId() != null) {
        buildStatementFromContext(list, configuration.getDatabaseId());
    }
    buildStatementFromContext(list, null);
}

private void buildStatementFromContext(List<XNode> list, String requiredDatabaseId) {
    for (XNode context : list) {
        // 通过 XMLStatementBuilder 来解析  <select|update|delete|insert> 标签
        final XMLStatementBuilder statementParser = new XMLStatementBuilder(configuration, builderAssistant, context, requiredDatabaseId);
        try {
            statementParser.parseStatementNode();
        } catch (IncompleteElementException e) {
            configuration.addIncompleteStatement(statementParser);
        }
    }
}
```
#### 2.3.XMLStatementBuilder

```java
public class XMLStatementBuilder extends BaseBuilder {

    private final MapperBuilderAssistant builderAssistant;
    private final XNode context;
    private final String requiredDatabaseId;

    public void parseStatementNode() {
        String id = context.getStringAttribute("id");
        String databaseId = context.getStringAttribute("databaseId");

        if (!databaseIdMatchesCurrent(id, databaseId, this.requiredDatabaseId)) {
            return;
        }

        // 获取 <select|insert|delete|update> 标签的所有属性信息
        Integer fetchSize = context.getIntAttribute("fetchSize");
        Integer timeout = context.getIntAttribute("timeout");
        String parameterMap = context.getStringAttribute("parameterMap");
        String parameterType = context.getStringAttribute("parameterType");
        Class<?> parameterTypeClass = resolveClass(parameterType);
        String resultMap = context.getStringAttribute("resultMap");
        String resultType = context.getStringAttribute("resultType");
        String lang = context.getStringAttribute("lang");
       	// 获取 LanguageDriver
        LanguageDriver langDriver = getLanguageDriver(lang);

        Class<?> resultTypeClass = resolveClass(resultType);
        String resultSetType = context.getStringAttribute("resultSetType");
        // 获取 StatementType，默认为 PREPARED，用于之后创建对应的 Statement
        StatementType statementType = StatementType.valueOf(context.getStringAttribute("statementType", StatementType.PREPARED.toString()));
        ResultSetType resultSetTypeEnum = resolveResultSetType(resultSetType);

        String nodeName = context.getNode().getNodeName();
        SqlCommandType sqlCommandType = SqlCommandType.valueOf(nodeName.toUpperCase(Locale.ENGLISH));
        boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
        boolean flushCache = context.getBooleanAttribute("flushCache", !isSelect);
        boolean useCache = context.getBooleanAttribute("useCache", isSelect);
        boolean resultOrdered = context.getBooleanAttribute("resultOrdered", false);

        // Include Fragments before parsing
        // 将 <include> 标签引用的 SQL 片段替换为对应的 <sql> 标签中定义的内容。
        XMLIncludeTransformer includeParser = new XMLIncludeTransformer(configuration, builderAssistant);
        includeParser.applyIncludes(context.getNode());

        // Parse selectKey after includes and remove them.
        processSelectKeyNodes(id, parameterTypeClass, langDriver);

        // Parse the SQL (pre: <selectKey> and <include> were parsed and removed)
        // 通过 LanguageDriver 对象创建 SqlSource
        SqlSource sqlSource = langDriver.createSqlSource(configuration, context, parameterTypeClass);
        String resultSets = context.getStringAttribute("resultSets");
        String keyProperty = context.getStringAttribute("keyProperty");
        String keyColumn = context.getStringAttribute("keyColumn");
        KeyGenerator keyGenerator;
        String keyStatementId = id + SelectKeyGenerator.SELECT_KEY_SUFFIX;
        keyStatementId = builderAssistant.applyCurrentNamespace(keyStatementId, true);
        // 获取主键生成策略
        if (configuration.hasKeyGenerator(keyStatementId)) {
            keyGenerator = configuration.getKeyGenerator(keyStatementId);
        } else {
            keyGenerator = context.getBooleanAttribute("useGeneratedKeys",
                    configuration.isUseGeneratedKeys() && SqlCommandType.INSERT.equals(sqlCommandType))
                    ? Jdbc3KeyGenerator.INSTANCE : NoKeyGenerator.INSTANCE;
        }

        builderAssistant.addMappedStatement(id, sqlSource, statementType, sqlCommandType,
                fetchSize, timeout, parameterMap, parameterTypeClass, resultMap, resultTypeClass,
                resultSetTypeEnum, flushCache, useCache, resultOrdered,
                keyGenerator, keyProperty, keyColumn, databaseId, langDriver, resultSets);
    }
}
```

如上面的代码所示，XMLStatementBuilder 类的 parseStatementNode() 方法的内容相对较多，但是逻辑非常清晰，主要做了以下几件事情：

（1）获取 <select|insert|delete|update> 标签的所有属性信息。

（2）将 \<include\> 标签引用的SQL片段替换为对应的 \<sql\> 标签中定义的内容。

（3）获取 lang 属性指定的 LanguageDriver，通过 LanguageDriver 创建 SqlSource。MyBatis 中的 SqlSource 表示一个SQL 资源。

（4）获取 KeyGenerator 对象。KeyGenerator 的不同实例代表不同的主键生成策略。

（5）所有解析工作完成后，使用 MapperBuilderAssistant 对象的 addMappedStatement() 方法创建 MappedStatement 对象。创建完成后，调用 Configuration 对象的 addMappedStatement() 方法将 MappedStatement 对象注册到 Configuration 对象中。

MyBatis 中的 MapperBuilderAssistant 是一个辅助工具类，用于构建 Mapper 相关的对象，例如 Cache、ParameterMap、ResultMap 等。

### 3.Mapper 方法调用过程详解

#### 3.1.MapperProxy 

MyBatis 中的 MapperProxy 实现了 InvocationHandler 接口，用于实现动态代理相关逻辑。当调用动态代理对象方法的时候，会执行 MapperProxy 类的 invoke() 方法。

```java
public class MapperProxy<T> implements InvocationHandler, Serializable {

    private static final long serialVersionUID = -6424540398559729838L;
    private final SqlSession sqlSession;
    private final Class<T> mapperInterface;
    private final Map<Method, MapperMethod> methodCache;

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        try {
            if (Object.class.equals(method.getDeclaringClass())) {
                return method.invoke(this, args);
            } else if (isDefaultMethod(method)) {
                return invokeDefaultMethod(proxy, method, args);
            }
        } catch (Throwable t) {
            throw ExceptionUtil.unwrapThrowable(t);
        }
        final MapperMethod mapperMethod = cachedMapperMethod(method);
        return mapperMethod.execute(sqlSession, args);
    }
}
```

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

#### 3.2.MapperMethod

在 MapperProxy 类的 invoke() 方法中，对从 Object 类继承的方法不做任何处理，调用 cachedMapperMethod() 方法获取一个 MapperMethod 对象。MapperMethod 类是对 Mapper 方法相关信息的封装，通过 MapperMethod 能够很方便地获取 SQL 语句的类型、方法的签名信息等。

```java
public class MapperMethod {

    private final SqlCommand command;
    private final MethodSignature method;

	public MapperMethod(Class<?> mapperInterface, Method method, Configuration config) {
    	this.command = new SqlCommand(config, mapperInterface, method);
    	this.method = new MethodSignature(config, mapperInterface, method);
    }
    
    //......
    
    public static class SqlCommand {

        // Mapper Id
        private final String name;
        // SQL 类型
        private final SqlCommandType type;

        public SqlCommand(Configuration configuration, Class<?> mapperInterface, Method method) {
            final String methodName = method.getName();
            final Class<?> declaringClass = method.getDeclaringClass();
            // 获取描述 <select|insert|delete|update> 标签的 MappedStatement 对象
            MappedStatement ms = resolveMappedStatement(mapperInterface, methodName, declaringClass, configuration);
            if (ms == null) {
                if (method.getAnnotation(Flush.class) != null) {
                    name = null;
                    type = SqlCommandType.FLUSH;
                } else {
                    throw new BindingException("Invalid bound statement (not found): "
                            + mapperInterface.getName() + "." + methodName);
                }
            } else {
                name = ms.getId();
                type = ms.getSqlCommandType();
                if (type == SqlCommandType.UNKNOWN) {
                    throw new BindingException("Unknown execution method for: " + name);
                }
            }
        }
        
        private MappedStatement resolveMappedStatement(Class<?> mapperInterface, String methodName,
                                   Class<?> declaringClass, Configuration configuration) {
            // 获取 Mapper 的 Id
            String statementId = mapperInterface.getName() + "." + methodName;
            // 如果 Concentration 中已经注册了 MappedStatement 对象，就直接获取
            if (configuration.hasStatement(statementId)) {
                return configuration.getMappedStatement(statementId);
            } else if (mapperInterface.equals(declaringClass)) {
                return null;
            }
            // 如果方法是在 Mapper 父接口中定义的，则根据父接口获取对应的 MappedStatement 对象
            for (Class<?> superInterface : mapperInterface.getInterfaces()) {
                if (declaringClass.isAssignableFrom(superInterface)) {
                    MappedStatement ms = resolveMappedStatement(superInterface, methodName,
                            declaringClass, configuration);
                    if (ms != null) {
                        return ms;
                    }
                }
            }
            return null;
        }
    }
    
    public static class MethodSignature {

        private final boolean returnsMany;
        private final boolean returnsMap;
        private final boolean returnsVoid;
        private final boolean returnsCursor;
        private final Class<?> returnType;
        private final String mapKey;
        private final Integer resultHandlerIndex;
        private final Integer rowBoundsIndex;
        private final ParamNameResolver paramNameResolver;

        public MethodSignature(Configuration configuration, Class<?> mapperInterface, Method method) {
            // 获取方法返回值类型
            Type resolvedReturnType = TypeParameterResolver.resolveReturnType(method, mapperInterface);
            if (resolvedReturnType instanceof Class<?>) {
                this.returnType = (Class<?>) resolvedReturnType;
            } else if (resolvedReturnType instanceof ParameterizedType) {
                this.returnType = (Class<?>) ((ParameterizedType) resolvedReturnType).getRawType();
            } else {
                this.returnType = method.getReturnType();
            }
            // 返回值为 void
            this.returnsVoid = void.class.equals(this.returnType);
            // 返回值类型为集合
            this.returnsMany = (configuration.getObjectFactory().isCollection(this.returnType) || this.returnType.isArray());
            // 返回值类型为 Cursor
            this.returnsCursor = Cursor.class.equals(this.returnType);
            this.mapKey = getMapKey(method);
            // 返回类型为 Map
            this.returnsMap = (this.mapKey != null);
            // RowBounds 参数位置索引
            this.rowBoundsIndex = getUniqueParamIndex(method, RowBounds.class);
            // ResultHandler 参数位置索引
            this.resultHandlerIndex = getUniqueParamIndex(method, ResultHandler.class);
            // ParamNameResolver 用于解析 Mapper 方法参数
            this.paramNameResolver = new ParamNameResolver(configuration, method);
        }
    }
}
```

如上面的代码所示，MethodSignature 构造方法中只做了3件事情：

1）获取 Mapper 方法的返回值类型，具体是哪种类型，通过 boolean 类型的属性进行标记。

2）记录 RowBounds 参数位置，用于处理后续的分页查询，同时记录 ResultHandler 参数位置，用于处理从数据库中检索的每一行数据。

3）创建 ParamNameResolver 对象。ParamNameResolver 对象用于解析 Mapper 方法中的参数名称及参数注解信息。

#### 3.3.ParamNameResolver

ParamNameResolver 构造方法中完成了 Mapper 方法参数的解析过程。

```java
public ParamNameResolver(Configuration config, Method method) {
    final Class<?>[] paramTypes = method.getParameterTypes();
    // 获取所有参数注解
    final Annotation[][] paramAnnotations = method.getParameterAnnotations();
    final SortedMap<Integer, String> map = new TreeMap<Integer, String>();
    int paramCount = paramAnnotations.length;
    // get names from @Param annotations
    // 从 @Param 注解中获取参数名称
    for (int paramIndex = 0; paramIndex < paramCount; paramIndex++) {
        if (isSpecialParameter(paramTypes[paramIndex])) {
            // skip special parameters
            continue;
        }
        String name = null;
        for (Annotation annotation : paramAnnotations[paramIndex]) {
            // 方法参数中是否有 Param 注解
            if (annotation instanceof Param) {
                hasParamAnnotation = true;
                // 参数名称
                name = ((Param) annotation).value();
                break;
            }
        }
        if (name == null) {
            // @Param was not specified.
            // 未指定 @Param 注解，用于判断是否使用实际的参数名称
            if (config.isUseActualParamName()) {
                // 获取参数名
                name = getActualParamName(method, paramIndex);
            }
            if (name == null) {
                // use the parameter index as the name ("0", "1", ...)
                // gcode issue #71
                name = String.valueOf(map.size());
            }
        }
        // 将参数信息存放在 map 中，key 为参数位置索引，value 为参数名称
        map.put(paramIndex, name);
    }
    // 将参数信息保存在 names 属性中
    names = Collections.unmodifiableSortedMap(map);
}
```

到此为止，整个 MapperMethod 对象的创建过程已经完成，接下来便是 Mapper 方法的执行。

```java
public class MapperProxy<T> implements InvocationHandler, Serializable {

    private static final long serialVersionUID = -6424540398559729838L;
    private final SqlSession sqlSession;
    private final Class<T> mapperInterface;
    private final Map<Method, MapperMethod> methodCache;

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        try {
            if (Object.class.equals(method.getDeclaringClass())) {
                return method.invoke(this, args);
            } else if (isDefaultMethod(method)) {
                return invokeDefaultMethod(proxy, method, args);
            }
        } catch (Throwable t) {
            throw ExceptionUtil.unwrapThrowable(t);
        }
        final MapperMethod mapperMethod = cachedMapperMethod(method);
        return mapperMethod.execute(sqlSession, args);
    }
}
```

MapperProxy#invoke()，最终还是调用的 MapperMethod#execute()。

```java
public class MapperMethod {

    private final SqlCommand command;
    private final MethodSignature method;

    public Object execute(SqlSession sqlSession, Object[] args) {
        Object result;
        // 其中 command 为 MapperMethod 构造时创建的 SqlCommand 对象
        // 获取 SQL 语句的类型
        switch (command.getType()) {
            case INSERT: {
                // 获取参数信息
                Object param = method.convertArgsToSqlCommandParam(args);
                // 调用 SqlSession 的 insert() 方法，然后调用 rowCountResult() 方法统计行数
                result = rowCountResult(sqlSession.insert(command.getName(), param));
                break;
            }
            case UPDATE: {
                Object param = method.convertArgsToSqlCommandParam(args);
                // 调用 SqlSession 的 update() 方法
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
}
```

在 execute() 方法中，首先根据 SqlCommand 对象获取 SQL 语句的类型，然后根据 SQL 语句的类型调用 SqlSession 对象对应的方法。也就是说 MyBatis 通过动态代理将 Mapper 方法的调用转换成通过 SqlSession 提供的 API 方法完成数据库的增删改查操作。

#### 3.4.SqlSession 执行 Mapper 过程

接下来以 SELECT 语句为例展示 SqlSession 执行 Mapper 的过程，SqlSession 接口只有一个默认的实现，即 DefaultSqlSession。

DefaultSqlSession#selectList() 代码如下：

```java
public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
    try {
        // 根据 Mapper Id 获取对应的 MappedStatement 对象
        MappedStatement ms = configuration.getMappedStatement(statement);
        // 以 MappedStatement 对象作为参数，调用 Executor 的 query() 方法
        return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
    } catch (Exception e) {
        throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
        ErrorContext.instance().reset();
    }
}
```

由上述代码可知，DefaultSqlSession#selectList() 实际上调用的是 Executor#query()。

BaseExecutor#query() 代码如下：

```java
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    // 获取 BoundSql，BoundSql 是对动态 SQL 解析生成的 SQL 语句和参数映射信息的封装
    BoundSql boundSql = ms.getBoundSql(parameter);
    // 创建 CacheKey，用于缓存 Key
    CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
    return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
}
```

```java
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
        clearLocalCache();
    }
    List<E> list;
    try {
        queryStack++;
        // 从缓存中获取结果，这里的缓存便是一级缓存
        list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
        if (list != null) {
            handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
        } else {
            // 若缓存中获取不到，则调用 queryFromDatabase() 从数据库中查询
            list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
        }
    } finally {
        queryStack--;
    }
    if (queryStack == 0) {
        for (DeferredLoad deferredLoad : deferredLoads) {
            deferredLoad.load();
        }
        // issue #601
        deferredLoads.clear();
        if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
            // issue #482
            clearLocalCache();
        }
    }
    return list;
}
```

```java
private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
        // 调用 doQuery() 方法查询
        list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
        localCache.removeObject(key);
    }
    
    // 查询缓存结果
    localCache.putObject(key, list);
    if (ms.getStatementType() == StatementType.CALLABLE) {
        localOutputParameterCache.putObject(key, parameter);
    }
    return list;
}
```

doQuery() 是一个模板方法，由 BaseExecutor 子类实现。Executor有三种不同的实现，分别为 BatchExecutor、SimpleExecutor 和 ReuseExecutor。

这里查看一下 SimpleExecutor#doQuery()。

```java
public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
        Configuration configuration = ms.getConfiguration();
        // 获取 StatementHandler 对象
        StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
        stmt = prepareStatement(handler, ms.getStatementLog());
        return handler.<E>query(stmt, resultHandler);
    } finally {
        closeStatement(stmt);
    }
}
```

在调用 Configuration 对象的 newStatementHandler() 方法创建 StatementHandler 对象时，newStatementHandler() 方法返回的是 RoutingStatementHandler 的实例。在 RoutingStatementHandler 类中，会根据配置 Mapper时statementType 属性指定的 StatementHandler 类型创建对应的 StatementHandler 实例进行处理。

```java
private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    // 获取 JDBC 中的 Connection 对象
    Connection connection = getConnection(statementLog);
    // 获取对应的 Statement 对象
    stmt = handler.prepare(connection, transaction.getTimeout());
    // 调用 StatementHandler#parameterize() 为 Statement 设置参数
    handler.parameterize(stmt);
    return stmt;
}
```

StatementHandler 接口有几个不同的实现类，分别为 SimpleStatementHandler、PreparedStatementHandler 和 CallableStatementHandler。默认情况下一般使用的是 PreparedStatementHandler。

那么查看一下 PreparedStatementHandler#query()。

```java
public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    PreparedStatement ps = (PreparedStatement) statement;
    // 调用 PreparedStatement 对象的 execute() 方法执行 SQL 语句
    ps.execute();
    // 使用 ResultHandler 来处理结果集
    return resultSetHandler.<E> handleResultSets(ps);
}
```

那么以上就是 MyBatis 如何通过调用 Mapper 接口定义的方法执行注解或者 XML 文件中配置的 SQL 语句的过程。









