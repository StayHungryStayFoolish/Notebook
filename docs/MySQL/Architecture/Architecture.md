# MySQL 逻辑架构

MySQL 逻辑架构可以分为三层（国内通常有另一种说法，但是包含的架构组件基本一样，使用 **/** 进行前后分割）：

-   Application Layer(应用层) **/** 连接层
-   Login Layer(逻辑层) **/** 解析层
-   Physical Layer(物理层) **/** 数据层

![MySQL-Architecture](https://gitee.com/bonismo/notebook-img/raw/master/img/MySQL/MySQL-Architecture.png)

## 1. Application Layer(应用层)

**应用层**主要负责用户或客户端与 **MySQL RDMS(Relation Database Management System)** 的交互，该层并非 MySQL 独有的服务，大多数基于网络的 **C/S 架构**都具备。该服务主要功能是：连接处理、身份验证、安全性等。

应用层在架构图中包含了以下组件：

### 1.1 Client Connectors(客户端连接器)

**Client Connectors** 为客户端程序提供与 **MySQL RDMS** 的连接。API 使用传统的 MySQL 协议或 X 协议提供对 MySQL 资源的低级访问。连接器和 API 均能够连接和执行来自另一种语言或环境的 MySQL 语句，包括 ODBC、Java（JDBC）、C ++、Python、Node.js、PHP、Perl、Ruby 和 C。

### 1.2 Connection Pool(连接池)

**Connection Pool** 是一种创建和管理可供任何需要它们的线程使用的连接池的技术 。连接池可以大大提高应用程序的性能，同时减少总体资源使用量。

在创建和管理过程中会是使用 **Authentication & Security** 进行授权和认证。

#### 1.2.1 连接池工作原理

大多数应用程序在主动处理事务时，只需要一个线程即可访问 JDBC 连接 ，通常只需几毫秒即可完成。当不处理事务时，连接处于空闲状态。连接池使空闲连接可以被其他一些线程用来完成有用的工作。

实际上，当线程需要使用 JDBC 对 MySQL 或其他数据库进行处理时，它会请求池中的连接。使用连接完成线程后，它将把它返回到池中，以便其他任何线程都可以使用它。

当从池中借出连接时，该连接仅由请求该连接的线程使用。从编程的角度来看，这就像线程在 `DriverManager.getConnection()` 每次需要 JDBC 连接时都调用 它一样。使用连接池，线程可能最终会使用新连接或已经存在的连接。

#### 1.2.1 Authentication & Security(授权 & 安全)

身份验证和授权是管理和保护 MySQL 服务器的基本考虑因素。 **身份验证**（有时缩写为“ authn”）是指用来验证允许客户端作为特定用户连接的策略和机制的类别。 **授权**（有时缩写为“ authz”）是在验证后发生的过程，用于确定允许帐户执行哪些操作。

#### 1.2.2 Connection Handling(连接处理)

##### 1.2.2.1 单次连接生命周期

![ConnectionLifecycle](https://gitee.com/bonismo/notebook-img/raw/master/img/MySQL/connectionlifecycle.gif)

**单次连接流程如下：**

1.  `Application` 向 `java.sql.Datasource` 请求数据库连接
2.  `java.sql.Datasource` 将使用 `java.sql.Driver` 驱动程序打开数据库连接
3.  创建数据库连接并打开 `TCP Socket`
4.  应用程序**读/写**数据库
5.  `Application` 不再需要时，`close` 数据库连接
6.  `Socket` 连接关闭

**从上述流程可以看出，打开/关闭数据库连接是一项非常昂贵的操作。**

##### 1.2.2.2 连接池生命周期

![PoolingConnectionLifecycle](https://gitee.com/bonismo/notebook-img/raw/master/img/MySQL/poolingconnectionlifecycle.gif)

**连接池流程如下：**

1.  `Application` 向 `java.sql.Datasource` 请求数据库连接
2.  `java.sql.Datasource` 向可用的 `Pool` 中获取可用的**空闲连接**并返回**（注：在没有可用连接且连接池尚未达到最大大小时，池才会创建新连接）**
3.  `Application` 不再需要时，`close` 数据库连接**（注：池连接的 `close()` 方法将返回到池的连接，而不是实际关闭）**

**关于步骤 2 创建新连接的流程图如下：**

![ConnectionAcquirerequestStates](https://gitee.com/bonismo/notebook-img/raw/master/img/MySQL/connectionacquirerequeststates.gif)

**使用连接池最直观的优点：**

1.  减少用于创建/销毁 TCP 连接的应用程序和数据库管理系统 I/O 开销
2.  减少 JVM 对象垃圾