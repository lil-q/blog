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

对于 java，最开始是使用 JDBC 来进行数据库管理的。**java数据库连接**（**J**ava **D**atabase **C**onnectivity，**JDBC**）是 java 提供的一个操作数据库的 API。它是由各种数据库厂商提供类和接口组成的数据库驱动，为多种数据库提供统一访问，使用数据库时只需要调用 JDBC 接口就行了，大致的步骤如下：

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
        try{
            //STEP 1: Register JDBC driver
            Class.forName("com.mysql.cj.jdbc.Driver");

            //STEP 2: Open a connection
            conn = DriverManager.getConnection(DB_URL,USER,PASS);

            //STEP 3: Execute a query
            stmt = conn.createStatement();
            String sql = "SELECT id, name FROM Employees";
            ResultSet rs = stmt.executeQuery(sql);

            //STEP 4: Extract data from result set
            while(rs.next()){
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
        }catch(SQLException se){
            //Handle errors for JDBC
            se.printStackTrace();
        }catch(Exception e){
            //Handle errors for Class.forName
            e.printStackTrace();
        }finally{
            //finally block used to close resources
            try{
                if(stmt!=null)
                    stmt.close();
            }catch(SQLException se2){
            }// nothing we can do
            try{
                if(conn!=null)
                    conn.close();
            }catch(SQLException se){
                se.printStackTrace();
            }
        }
    }
}
```

可大致总结为四个步骤：

* 通过类加载注册特定数据库的 driver；
* 通过 URL、用户名和密码建立连接；
* 新建 SQL 语句进行查询；
* 获取结果。

### 2.1 SPI

那么类加载是如何注册 driver 的呢？可以在 MySQL 的 jar 包 com.mysql.cj.jdbc 下找到 MySQL 的 Driver，其中包含静态代码块：

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

所以`Class.forName("com.mysql.cj.jdbc.Driver")`加载时，就会创建相应的 Driver 并自动加入到 DriverManager 下的 *registeredDrivers* 这个 list 中。

```java
public class DriverManager {
    
    private final static CopyOnWriteArrayList<DriverInfo> registeredDrivers = new CopyOnWriteArrayList<>();
}
```

假如我们同时可能会用到两个数据库，则需要分别加载这两个数据库的 driver。那么当我们需要特定一个数据库时，JDBC 是如何找到特定的那个数据库的呢？

其实在第二步建立连接时 JDBC 会去 *registeredDrivers* 这个 list 中按照 URL 找到相对应的 driver：

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

其实上面 DriverManager 管理多个数据库的 driver 就是用到了 SPI 的思想，管理者提供一个接口，服务提供商们按照接口的规范来实现功能。

> **Service Provider Interface** (**SPI**) is an API intended to be implemented or extended by a third party. It can be used to enable framework extension and replaceable components.

如果我们使用的 JDBC 版本是 4.0 及以后的版本，那么第一步的手动类加载其实可以删去，因为 JDBC 采用新的规范让 SPI 帮我们完成了这一步。我们可以在 MySQL 的 jar 包 com.mysql.cj.jdbc 下找到 META-INF.services 中的 java.sql.Driver 文件，其中记录了 MySQL 所使用的 Driver 的全限定名，DriverManager 中的 ServiceLoader 会根据这个全限定名来完成注册。

这里还涉及到一个与类加载相关的问题：JDBC 属于 java 提供的服务，DriverManager 的类加载器应该是 Bootstrap Class Loarder，但是 MySQL 等提供的第三方服务不应该使用这个加载器，而应该用 Application Class Loader。 为了解决这个问题，只能打破双亲委派机制了，比如使用线程上下文类加载器（Thread Context ClassLoader）。

### 2.2 JDBC 存在的问题

* 频繁地开启和关闭数据库连接，严重地影响了数据库的性能；
* 代码中存在硬编码。每当需求变更时，都需要修改源代码然后重新编译，系统不易维护；
* 设置查询参数的过程繁琐；
* 获取查询结果的过程繁琐。

## 三、ORM

> **ORM**(**O**bject-**R**elational **M**apping) in computer science is a programming technique for converting data between incompatible type systems using object-oriented programming languages. This creates, in effect, a "virtual object database" that can be used from within the programming language. 

java 是面向对象的语言，ORM 就是建立对象与数据库表之间的关系，从而达到操作对象就相当于操作数据库表的目的，ORM 能够大大减少重复性代码。

### 3.1 JPA

**JPA**（**J**ava **P**ersistence **A**PI）规范本质上就是一种 ORM 规范，JPA 并未提供 ORM 实现，仅仅提供了一些编程的 API 接口，但具体实现则由服务厂商来提供实现。常见的有：

* Hibernate（JBoos开源）；
* Open JPA（apache开源）；
* Toplink。

## 四、MyBatis

MyBatis 和 Hibernate 都属于持久层框架，但是 MyBatis 并不是完整的 ORM 框架。Hibernate 完全可以通过对象关系模型实现对数据库的操作，拥有完整的 JavaBean 对象与数据库的映射结构来自动生成 SQL 语句。而 MyBatis 仅有基本的字段映射，对象数据的管理以及对象实际关系仍然需要通过手写 SQL 语句来实现。

Hibernate 数据库移植性远大于 MyBatis。 Hibernate 通过它强大的映射结构和 HQL 语言，大大降低了对象与数据库的耦合性，而Mybatis 由于需要手写 SQL 语句，因此与数据库的耦合性直接取决于程序员写 SQL 语句的方法，移植成本高。



## 参考

1. [JDBC中的知识点](https://www.bilibili.com/video/BV1st4y1Q7zP)
2. [Java 持久层——JDBC，MyBatis，ORM与JPA](https://www.jianshu.com/p/ae3debd55486)
3. [MyBatis 和 Hibernate 的区别](https://www.zhihu.com/question/39454008/answer/253817938)
4. 

