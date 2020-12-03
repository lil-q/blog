---
title: "持久层与持久层框架"
date: 2020-09-14T12:00:08+08:00
slug: ""
description: "持久层框架 MyBatis"
keywords: [MyBatis, 持久层, SPI, 打破双亲委派]
tags: [framework]
math: false
toc: true
---

为了保存数据，需要持久层；为了实现持久层，需要数据库；为了管理数据库，需要数据库管理系统；为了操作数据库管理系统，需要持久层框架。

## 一、持久层

储存在内存之上的数据在断电后就会清除，与之相对的，储存在硬盘中的数据可以长时间、安全的保存，称之为持久层。

企业应用中的数据（各种订单数据、客户数据、库存数据等）十分重要，所以需要把数据持久化。持久化可以通过很多方式，写文件和数据库都可以，一般会把数据持久化到数据库中，因为可以很方便的查询、统计和分析，而且数据库 ACID 特性中的持久性（durability）就是针对的持久层。

## 二、JDBC

对于 Java，最开始是使用 JDBC 来进行数据库管理的。**Java 数据库连接**（**J**ava **D**atabase **C**onnectivity，**JDBC**）是 Java 提供的一个操作数据库的 API。它是由各种数据库厂商提供类和接口组成的数据库驱动，为多种数据库提供统一访问，使用数据库时只需要调用 JDBC 接口就行了，大致的步骤如下：

```java
public class JDBCTest {
    // JDBC driver name and database URL
    static final String JDBC_DRIVER = "com.mysql.cj.jdbc.Driver";
    static final String DB_URL = "jdbc:mysql://localhost:port/DB_name?...";

    // Database credentials
    static final String USER = "root";
    static final String PASS = "password";

    public static void main(String[] args) {
        Connection conn = null;
        Statement stmt = null;
        try {
            //STEP 1: Register JDBC driver
            Class.forName("com.mysql.cj.jdbc.Driver");

            //STEP 2: Open a connection
            conn = DriverManager.getConnection(DB_URL,USER,PASS);

            //STEP 3: Execute a query
            stmt = conn.createStatement();
            String sql = "SELECT id, name FROM Employees";
            ResultSet rs = stmt.executeQuery(sql);

            //STEP 4: Extract data from result set
            while (rs.next()) {
                //Retrieve by column name
                int id  = rs.getInt("id");
                String first = rs.getString("name");

                //Display values
                System.out.print("ID: " + id);
                System.out.println(", name: " + name);
            }
            //STEP 5: Clean-up environment
            rs.close();
            stmt.close();
            conn.close();
        } catch (SQLException se) {
            //Handle errors for JDBC
            se.printStackTrace();
        } catch (Exception e) {
            //Handle errors for Class.forName
            e.printStackTrace();
        } finally {
            //finally block used to close resources
            try {
                if (stmt!=null)
                    stmt.close();
            } catch (SQLException se2) {
            // nothing we can do
            }
            try {
                if (conn!=null)
                    conn.close();
            } catch (SQLException se) {
                se.printStackTrace();
            }
        }
    }
}
```

可大致总结为五个步骤：

1. 通过类加载注册特定数据库的 *Driver*；
2. 通过 URL、用户名和密码建立连接 *Connection*；
3. 创建 JDBC *Statements* 对象并进行 SQL 语句查询；
5. 获取结果 *ResultSet*；
6. 释放相关资源（关闭 *Connection*，关闭 *Statement*，关闭 *ResultSet*）。

### 2.1 SPI

那么类加载是如何注册 *Driver* 的呢？我们可以在 MySQL 的 jar 包 com.mysql.cj.jdbc 下找到 MySQL 的 *Driver*，其中包含静态代码块：

```java
public class Driver extends NonRegisteringDriver implements java.sql.Driver {
    public Driver() throws SQLException {
    }

    static {
        try {
            // 这个函数会把创建的 driver 对象加入到 registerDrivers 这个 list 中
            DriverManager.registerDriver(new Driver());
        } catch (SQLException var1) {
            throw new RuntimeException("Can't register driver!");
        }
    }
}
```

 `Class.forName("com.mysql.cj.jdbc.Driver")` 加载时，就会创建相应的 *Driver* 对象并自动加入到 *DriverManager* 下的 *registeredDrivers* 这个 list 中。

```java
public class DriverManager {
    
    private final static CopyOnWriteArrayList<DriverInfo> registeredDrivers = new CopyOnWriteArrayList<>();
}
```

假如我们同时可能会用到两个数据库，则需要分别加载这两个数据库的 *Driver*。那么当我们需要特定一个数据库时，JDBC 是如何找到特定的那个数据库的呢？

其实在第二步 `DriverManager.getConnection(DB_URL,USER,PASS)` 建立连接时传入了 URL，JDBC 会根据这个 URL 去 *registeredDrivers* 中找到相对应的 *Driver*：

```java
for (DriverInfo aDriver : registeredDrivers) {
    // If the caller does not have permission to load the driver then
    // skip it.
    if (isDriverAllowed(aDriver.driver, callerCL)) {
        try {
            println("    trying " + aDriver.driver.getClass().getName());
            Connection con = aDriver.driver.connect(url, info);
            if (con != null) {
                // Success!
                println("getConnection returning " + aDriver.driver.getClass().getName());
                return (con);
            }
        } catch (SQLException ex) {
            if (reason == null) {
                reason = ex;
            }
        }
    } else {
        println("    skipping: " + aDriver.getClass().getName());
    }
}
```

上面 *DriverManager* 管理多个数据库的 *Driver* 就是用到了 SPI 的思想，管理者提供一个接口，服务提供商们按照接口的规范来实现功能。

> **Service Provider Interface** (**SPI**) is an API intended to be implemented or extended by a third party. It can be used to enable framework extension and replaceable components.

如果我们使用的 JDBC 版本是 4.0 及以后的版本，那么第一步的手动类加载其实可以删去，因为 JDBC 采用新的规范让 SPI 帮我们完成了这一步。我们可以在 MySQL 的 jar 包 com.mysql.cj.jdbc 下找到 META-INF.services 中的 java.sql.Driver 文件，其中记录了 MySQL 所使用的 *Driver* 的全限定名，*DriverManager* 中的 *ServiceLoader* 会根据这个全限定名来完成注册。

这里还涉及到一个与类加载相关的问题：JDBC 属于 Java 提供的服务，*DriverManager* 的类加载器应该是 Bootstrap Class Loarder，但是 MySQL 等提供的第三方服务不应该使用这个加载器，而应该用 Application Class Loader。 为了解决这个问题，只能打破双亲委派机制了，比如使用线程上下文类加载器（Thread Context ClassLoader）。

### 2.2 JDBC 存在的问题

* 频繁地开启和关闭数据库连接，严重地影响了数据库的性能；
* 代码中存在硬编码。每当需求变更时都需要修改源代码然后重新编译，系统不易维护；
* 设置查询参数的过程繁琐；
* 获取查询结果的过程繁琐。

## 三、ORM

> **ORM**(**O**bject-**R**elational **M**apping) in computer science is a programming technique for converting data between incompatible type systems using object-oriented programming languages. This creates, in effect, a "virtual object database" that can be used from within the programming language. 

Java 是面向对象的语言，ORM 就是建立对象与数据库表之间的关系，从而达到操作对象就相当于操作数据库表的目的，ORM 能够大大减少重复性代码。

### 3.1 JPA

**JPA**（**J**ava **P**ersistence **A**PI）规范本质上就是一种 ORM 规范，JPA 并未提供 ORM 实现，仅仅提供了一些编程的 API 接口，但具体实现则由服务厂商来提供实现。常见的有：

* Hibernate（JBoos开源）；
* Open JPA（apache开源）；
* Toplink。

## 四、MyBatis

MyBatis 和 Hibernate 都属于持久层框架，但是 MyBatis 并不是完整的 ORM 框架。Hibernate 完全可以通过对象关系模型实现对数据库的操作，拥有完整的 JavaBean 对象与数据库的映射结构来自动生成 SQL 语句。而 MyBatis 仅有基本的字段映射，对象数据的管理以及对象实际关系仍然需要通过手写 SQL 语句来实现。

Hibernate 数据库移植性远大于 MyBatis。 Hibernate 通过它强大的映射结构和 HQL 语言，大大降低了对象与数据库的耦合性，而 Mybatis 由于需要手写 SQL 语句，因此与数据库的耦合性直接取决于程序员写 SQL 语句的方法，移植成本高。

MyBatis 的使用大致可以分为以下五个步骤：

1. 读取核心配置文件并创建 *SqlSessionFactory* 单例；
2. 从 *SqlSessionFactory* 工厂中创建 *SqlSession*；
3. SQL 预编译并执行；
4. 对执行结果进行二次封装；
5. 根据事务情况选择提交或回滚。

### 4.1 建造工厂

从 XML 文件中构建 *SqlSessionFactory* 的实例非常简单，建议使用类路径下的资源文件进行配置。 但也可以使用任意的输入流（*InputStream*）实例，比如用文件路径字符串或 file:// URL 构造的输入流。MyBatis 包含一个名叫 *Resources* 的工具类，它包含一些实用方法，使得从类路径或其它位置加载资源文件更加容易。

```java
String resource = "org/mybatis/example/mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
```

在获得配置的输入流后，就可以使用 *SqlSessionFactoryBuilder* 读取这个输入流并创建 *SqlSessionFactory*。但是需要注意两者的作用域和什么周期：

***SqlSessionFactoryBuilder*** 实例化后，一旦创建了 *SqlSessionFactory*，就不再需要它了。 因此 *SqlSessionFactoryBuilder* 实例的最佳作用域是方法作用域（也就是局部方法变量）。 

***SqlSessionFactory*** 一旦被创建就应该在应用的运行期间一直存在，没有任何理由丢弃它或重新创建另一个实例。 使用 *SqlSessionFactory* 的最佳实践是在应用运行期间不要重复创建多次，多次重建 *SqlSessionFactory* 被视为一种代码 “坏习惯”。因此 *SqlSessionFactory* 的最佳作用域是应用作用域。 

有很多方法可以做到，最简单的就是使用单例模式或者静态单例模式，比如下面这个简单的单例模式：

```java
public class SessionFactory {
    private static SqlSessionFactory sqlSessionFactory;
    
    public static synchronized SqlSessionFactory getInstance(InputStream inputStream){
        if (sqlSessionFactory == null) {
            sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
        }
        
        return sqlSessionFactory;
    }
}
```

注意到 *SqlSessionFactoryBuilder* 只会在第一次创建 *SqlSessionFactory* 时新建一次，并且作为局部变量会在方法出栈时销毁，以保证所有的 XML 解析资源可以被释放给更重要的事情。而 *SqlSessionFactory* 作为单例在整个应用运行时期只会存在唯一一个对象。

### 4.2 创建 sqlSession

*SqlSession* 是 Mybatis 工作的最顶层 API 会话接口，所有的数据库操作都经由它来实现，由于它就是一个会话，即一个 *SqlSession* 应该仅存活于一个业务请求中，也可以说一个 *SqlSession* 对应这一次数据库会话，它不是永久存活的，每次访问数据库时都需要创建它。

每个线程都应该有它自己的 *SqlSession* 实例。*SqlSession* 的实例不是线程安全的，因此是不能被共享的，所以它的最佳的作用域是请求或方法作用域。每次收到 HTTP 请求，就可以打开一个 *SqlSession*，返回一个响应后，就关闭它。 这个关闭操作很重要，为了确保每次都能执行关闭操作，你应该把这个关闭操作放到 *finally* 块中。 下面的示例就是一个确保 *SqlSession* 关闭的标准模式：

```java
try (SqlSession session = sqlSessionFactory.openSession()) {
    // 你的应用逻辑代码
} finally {
    // 关闭操作
}
```

特殊的，如果多个请求属于同一个事务，那么多个请求都在共用一个 *SqlSession* [[6]](http://objcoding.com/2019/03/20/mybatis-sqlsession/)。

### 4.3 执行 SQL 语句

在拿到 *SqlSession* 对象后，我们调用它的各种方法来进行相应的操作。在加载 XML 配置的时候，mapping.xml 的 namespace 和 id 信息就会存放为 *mappedStatements* 的 *key*，对应的，SQL 语句就是对应的 *value*。MyBatis 先通过 *mappedStatements* 找到相应的 *Statement*，使用 `prepareStatement()` 方法进行预编译，预编译能够提高效率并防止 SQL 注入。

### 4.4 事务

当使用 Spring 整合的 MyBatis 时，Spring 封装了很多处理细节，这样我们就不用写大量的冗余代码，可以专注于业务开发。MyBatis 的 *SqlSession* 提供了几个方法来处理事务，但是当使用 MyBatis-Spring 时，所有的 *Bean* 将会注入由 Spring 管理的 *SqlSession* 或映射器。也就是说，Spring 将负责处理事务。

## 五、MyBatis 所做的改进

对比 JDBC 来说，MyBatis 进行了很多改进，具体表现在以下四个方面：

#### 1. 连接获取和释放

MyBatis 不用像 JDBC 一样注册相关 *Driver*，而是通过 *DataSource* 进行隔离解耦，统一从 *DataSource* 里面获取数据库连接，*DataSource* 具体由 DBCP 实现还是由容器的 JNDI 实现都可以，*DataSource* 的具体设置通过用户配置来实现。

#### 2. SQL 统一管理

SQL 语句统一在 XML 文件中编辑保存，通过键值对的形式储存在 *mappedStatements* 中。

#### 3. 传入参数映射和动态 SQL

Mybatis 在处理 #{} 时，会将 SQL 语句中的 #{} 替换为 ? 号，调用 *PreparedStatement* 的 `set()` 方法来赋值；Mybatis 在处理 ${} 时，就是把 ${} 替换成变量的值。

MyBatis 实现了动态 SQL，这样在 Java 代码中就不需要考虑 SQL 语句相关的业务逻辑了，包含以下标签：`if` / `choose` / `when` / `otherwise` / `trim` / `where` / `set` / `foreach`。

#### 4. 结果映射和结果缓存

结果映射有两种方式，第一种是使用 `<resultMap>` 标签，逐一定义列名和对象属性名之间的映射关系。第二种是使用 SQL 列的别名功能，将列别名书写为对象属性名，不区分大小写。

MyBatis 提供两个级别的缓存：

* 一级缓存是 *SqlSession* 级别的缓存。在操作数据库时需要构造 *SqlSession* 对象，这个对象中（内存区域）有一个数据结构（*HashMap*）用于存储缓存数据。不同的 *SqlSession* 之间的缓存数据区域（*HashMap*）是互相不影响的。
* 二级缓存是 *mapper* 级别的缓存。多个 *SqlSession* 去操作同一个 *Mapper* 的 SQL 语句可以共用二级缓存，二级缓存是跨 *SqlSession* 的。

## 参考

1. [JDBC中的知识点](https://www.bilibili.com/video/BV1st4y1Q7zP)
2. [Java 持久层——JDBC，MyBatis，ORM与JPA](https://www.jianshu.com/p/ae3debd55486)
3. [MyBatis 和 Hibernate 的区别](https://www.zhihu.com/question/39454008/answer/253817938)
4. [MyBatis 官网教程](https://mybatis.org/mybatis-3/zh/getting-started.html)
5. [MyBatis 源码分析其工作原理](https://zhuanlan.zhihu.com/p/59792101)
6. [从源码的角度解析Mybatis的会话机制](http://objcoding.com/2019/03/20/mybatis-sqlsession/)
7. [终结篇：MyBatis原理深入解析（一）](https://www.jianshu.com/p/ec40a82cae28)
8. [JDBC 存在的问题](https://www.cnblogs.com/diffx/p/10611082.html)

