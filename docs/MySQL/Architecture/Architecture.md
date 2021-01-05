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

**Client Connectors** 为客户端程序提供与 **MySQL RDMS** 的连接。API 使用传统的 MySQL协议或 X 协议提供对 MySQL资源的低级访问。连接器和 API 均能够连接和执行来自另一种语言或环境的 MySQL 语句，包括ODBC，Java（JDBC），C ++，Python，Node.js，PHP，Perl，Ruby和C。

