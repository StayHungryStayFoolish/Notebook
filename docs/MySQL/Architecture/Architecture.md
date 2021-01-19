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

**连接池工作原理**

大多数应用程序在主动处理事务时，只需要一个线程即可访问 JDBC 连接 ，通常只需几毫秒即可完成。当不处理事务时，连接处于空闲状态。连接池使空闲连接可以被其他一些线程用来完成有用的工作。

实际上，当线程需要使用 JDBC 对 MySQL 或其他数据库进行处理时，它会请求池中的连接。使用连接完成线程后，它将把它返回到池中，以便其他任何线程都可以使用它。

当从池中借出连接时，该连接仅由请求该连接的线程使用。从编程的角度来看，这就像线程在 `DriverManager.getConnection()` 每次需要 JDBC 连接时都调用它一样。使用连接池，线程可能最终会使用新连接或已经存在的连接。

#### 1.2.1 Authentication & Security(授权 & 安全)

身份验证和授权是管理和保护 MySQL 服务器的基本考虑因素。 **身份验证**（有时缩写为 "authn"）是指用来验证允许客户端作为特定用户连接的策略和机制的类别。 **授权**（有时缩写为 "authz"）是在验证后发生的过程，用于确定允许帐户执行哪些操作。

在 MySQL 服务器运行初时，会初始化一个 `mysql` 的数据库，在 `user` 表中存储相关信息。

##### 1.2.1.1 身份验证 & 权限优先级

查看 `mysql.user` 表中行信息，为了显示方便，省略了部分列、字段描述信息，可以看到 `Host` 和 `User` 是联合主键、权限、密码、账户是否锁定等比较重要的行字段。 

```mysql
mysql> use mysql;
mysql> show full columns from user;

+-----------------------+----------------------+------+-----+-----------------------+---------------------------------+
| Field                 | Type                 | Null | Key | Default               | Privileges                      |
+-----------------------+----------------------+------+-----+-----------------------+---------------------------------+
| Host                  | char(60)             | NO   | PRI |                       | select,insert,update,references |
| User                  | char(32)             | NO   | PRI |                       | select,insert,update,references |
| Select_priv           | enum('N','Y')        | NO   |     | N                     | select,insert,update,references |
| Insert_priv           | enum('N','Y')        | NO   |     | N                     | select,insert,update,references |
| Update_priv           | enum('N','Y')        | NO   |     | N                     | select,insert,update,references |
| Delete_priv           | enum('N','Y')        | NO   |     | N                     | select,insert,update,references |
| Create_priv           | enum('N','Y')        | NO   |     | N                     | select,insert,update,references |
| Drop_priv             | enum('N','Y')        | NO   |     | N                     | select,insert,update,references |
| ... ...               | ... ...              | ...  |     | ... ...               | ... ...                         |
| plugin                | char(64)             | NO   |     | mysql_native_password | select,insert,update,references |
| authentication_string | text                 | YES  |     | NULL                  | select,insert,update,references |
| password_expired      | enum('N','Y')        | NO   |     | N                     | select,insert,update,references |
| password_last_changed | timestamp            | YES  |     | NULL                  | select,insert,update,references |
| password_lifetime     | smallint(5) unsigned | YES  |     | NULL                  | select,insert,update,references |
| account_locked        | enum('N','Y')        | NO   |     | N                     | select,insert,update,references |
+-----------------------+----------------------+------+-----+-----------------------+---------------------------------+
45 rows in set (0.00 sec)
```

**连接请求相关 5 个列的基本信息：**

>   **1. User & Host**

`User` 是用户名，与 `Host` 共同组成**唯一用户身份**。

`Host` 是客户端所在的主机地址**（`Host` 由完整域名或 IP 地址组成）**，与 `User` 共同组成**唯一用户身份**。

当连接发出请求后，MySQL 服务器会根据当前用户身份验证当前请求，根据以下因素确定是否接收请求：

1.  是否可以正确验证为您所请求的用户帐户
2.  该用户帐户是否在系统内被锁定

如果验证失败或当前账户被锁定，MySQL 会拒绝连接请求。

>   例如以下两个用户身份信息会被认定为两个不同的用户，因为一个来自 Host 为 example.com 的主机，一个来自 Host 为 example.org 的主机。
>
>   user@example.com
>
>   user@example.org

>   **2. plugin**

`plugin` 使用 `User` 和 `Host` 方式的身份验证插件。`plugin` 字段决定如何对客户端进行身份验证。

MySQL **4.1** 版本前，默认使用 `mysql_old_password` 为身份验证插件，该 `mysql_old_password` 插件基于较旧的（4.1之前的）密码哈希方法实现本机身份验证（在 MySQL 5.7.5 中已弃用并删除）。

MySQL **5.7** 版本中，默认使用 `mysql_native_password` 为身份验证插件，该 `mysql_native_password` 插件基于本机密码哈希方法实现身份验证。

MySQL **8.0** 版本中，默认使用 `caching_sha2_password` 为身份验证插件，该 `caching_sha2_password` 插件需要安全的连接（使用 TLS 凭据，Unix 套接字文件或共享内存来建立）或未加密的连接，该连接支持使用 RSA 密钥对进行密码交换。但是，插件的缓存功能可以减轻与安全连接相关的性能成本。一旦将哈希密码存储在内存中，便可以使用基于 SHA256 的质询-响应机制在未加密的通道上执行身份验证，这意味着可以更快地对以前连接的用户进行身份验证。

具体可以参考： [MySQL 8.0 可插拔身份验证官方文档](https://dev.mysql.com/doc/mysql-secure-deployment-guide/8.0/en/secure-deployment-configure-authentication.html)、[MySQL 5.7 可插拔身份验证官方文档](https://dev.mysql.com/doc/refman/5.7/en/pluggable-authentication.html)

>   **3. authentication_string**

对于 `native` 身份验证插件（那些仅使用 `mysql.user` 表中的信息对用户进行身份验证的插件），该 `authentication_string` 行包含用于检查用户密码的字符串。大多数情况下，这是密码的哈希版本。如果 `authentication_string` 为空，则客户端不指定密码才能成功进行身份验证。

对于使用外部系统进行身份验证的插件，`authentication_string` 通常会使用来指定外部系统正确验证用户身份所需的其他信息（例如服务名称，其他标识信息等）。

>   **4. account_locked**

`account_locked` 确定用户账户是否在系统内被锁定。帐户可以由数据库管理员手动锁定。

**权限优先级**

MySQL 根据以上 5 个列的字段信息确定是否接受连接请求。

用户账户的权限优先级表现为：`Global Privileges > Database Privileges > Table Privileges`

`mysql` 初始数据库的表的优先级表现为：`mysql.user > mysql.db > mysql.tables_priv > mysql.columns_priv`

**行的排序优先级**

当多个连接请求同时到达 MySQL 服务器时，服务器会按照特定性对其排序。

在 `Host` 行表现为：`myhost.example.com > %.example.com > %.com > %`

在 `Name` 行表现为：`非空用户名 > 空白用户名`，即 `非匿名用户 > 匿名用户`

根据以上权限，不同的权限信息存储在不同的数据库表中：

-   `GRANT SELECT ON Demo.table TO abc@123;` 存储在 `mysql.tables_priv`
-   `GRANT ALL PRIVILEGES ON Demo.* TO abc@123;` 存储在 `mysql.db`

具体可以参考：[MySQL 5.7 权限官方文档](https://dev.mysql.com/doc/refman/5.7/en/privileges-provided.html)

##### 1.2.1.2 授权

验证客户端的以上凭据后，MySQL 建立连接，然后进入访问控制系统的第二部分以确定授权。MySQL 授权是一个`持续`的过程，它会根据用户帐户的特定权限检查每个命令。如果命令属于用户权限范围，则允许执行该操作。如果不是，则服务器拒绝该请求。

**权限范围的授权**

>   **Global Privileges**

`全局权限`不与特定数据库进行绑定的特权。在该 MySQL 整个服务器均有效。许多`全局权限`与系统管理有关，并且与系统管理职责有关，而不是直接与数据库管理相关。

>   **Database Privileges**

`数据库权限`与 MySQL 服务器的特定数据库进行绑定。

>   **Table Privileges**

`数据表权限`可以控制数据库中指定的表中的数据和索引等。

以上授权可以使用 `show grants for '<root>@<host>'` 查询。

```mysql
mysql> show grants for root@localhost;
+---------------------------------------------------------------------+
| Grants for root@localhost                                           |
+---------------------------------------------------------------------+
| GRANT ALL PRIVILEGES ON *.* TO 'root'@'localhost' WITH GRANT OPTION |
| GRANT PROXY ON ''@'' TO 'root'@'localhost' WITH GRANT OPTION        |
+---------------------------------------------------------------------+
2 rows in set (0.00 sec)
```

**内部的静态与动态权限**

`静态权限`内置在服务器中，不可取消、禁用。静态权限始终可用，属于基本权限（例如：读写权限）。

`动态权限`取决于组件或插件的可用性。动态权限可以在 MySQL 服务器运行时进行注册或注销，以影响其权限的可用性。动态权限在范围上属于全局的，适用于 MySQL 整个服务器。

**MySQL 权限相关的表**

查看 `mysql` 数据库的表，可以看到以下和权限相关的表。

>   **user**

`user` 表定义了用于连接 MySQL 服务器上用户身份验证的账户，还定义了每个账户的**静态全局权限**。

>   **global_grants**

`global_grants` 定义了每个账户的**动态全局权限**。组件或插件定义的权限都在该表中注册。

**注：**该表只存在于 MySQL 8.0，具体文档可以参考：[MySQL Grants Tables 8.0 官方文档](https://dev.mysql.com/doc/refman/8.0/en/grant-tables.html)

>   **db**

`db` 定义了数据库级别权限。`db` 表与 `user` 表类似，定义了账户的**静态全局权限**，只是多了一列 `db` 用来定义该账户指定可操作的数据库。

>   **tables_priv、columns_priv**

`tables_priv` 和 `columns_priv` 类似，根本的区别只有权限的粒度不同，其中 `columns_priv` 多了一个列 `tables_priv` 用来指定数据表和列的关系。

>   **procs_priv**

`procs_priv` 定义了执行过程和功能的权限。其中 `Routine_name` 表示可以执行的过程或功能的名字，`Routine_type` 表示可以执行的过程或功能。

>   **proxies_priv**

`proxies_priv` 定义了账户的代理特权，允许一个账户**代理/继承**另一个账户的特权。`Host` 和 `User` 列表示当前账户（代理者），`Proxied_host` 和 `Proxied_user` 列表示可以被前者继承的账户（被代理者），`With_grant` 表示代理者是否将 **PROXY** 权限授予其他账户。

>   **MySQL 8.0 增加的部分权限相关的表**
>
>      
>
>   **default_roles** 定义了默认的账户角色
>
>   **role_edges** 定义了账户与角色关系描述
>
>   **password_history** 定义了有关账户密码更改的信息

具体参考：[MySQL Grants Tables 5.7 官方文档](https://dev.mysql.com/doc/refman/5.7/en/grant-tables.html)、[MySQL Grants Tables 8.0 官方文档](https://dev.mysql.com/doc/refman/8.0/en/grant-tables.html)

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

![ConnectionAcquireQuestStates](https://gitee.com/bonismo/notebook-img/raw/master/img/MySQL/connectionacquirerequeststates.gif)

**使用连接池最直观的优点：**

1.  减少用于创建/销毁 TCP 连接的应用程序和数据库管理系统 I/O 开销
2.  减少 JVM 对象垃圾

##### 1.2.2.3 线程缓存与线程池
