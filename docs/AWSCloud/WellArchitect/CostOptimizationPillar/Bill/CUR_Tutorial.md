# SQL Tutorial

## Prerequisites

1. Install Navicat(MySQL Client)

   Navicat Download: https://www.navicat.com/en/download/navicat-premium

2. Sign up for AWS 
   
    2.1 Open https://portal.aws.amazon.com/billing/signup.

    2.2 Follow the online instructions. Part of the sign-up procedure involves receiving a phone call and entering a verification code on the phone keypad.

3. Creating a MySQL DB instance

   3.1 In the navigation pane, choose `Databases`.
   
   3.2 Choose `Create database`.

   3.3 On the Create database page, shown following, make sure that the Standard create option is chosen, and then choose `MySQL` or `Aurora`.

   ![Create Database](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/images/MySQL-Launch01.png)

   3.4 In the `Templates` section, choose the template that meets your needs.
      
   3.5 In the `Settings` section, set these values:

    **Use Navicat must use `Master username & Master password`, pls record info.**

           DB instance identifier – cost-db-instance

           Master username – admin

           Auto generate a password – Clear the check box.

           Master password – Choose a password.

           Confirm password – Retype the password.
   


   3.6 In the `Instance configuration` section, choose the template that meets your needs.   

   3.7 In the `Storage` and `Availability & durability` sections, use the default values.

   3.8 In the `Connectivity` section, set these values:

            Network type - IPv4

            Virtual private cloud (VPC) - Choose an existing VPC with both public

            Subnet group - Choose default 

            Public access - Choose Yes

            VPC security group - Choose an exisiting security groups

   3.9 Other sections, use the default values.

   3.10 `Create database` and wait a moment.

4. Use Navicat Connect MySQL DB

   4.1 In the navigation pane, choose `Databases` to display a list of your DB instances.

   4.2 Choose the name of the DB instance to display its details. 

   4.3 On the `Connectivity & security` tab, copy the endpoint. Also, note the port number. You need both the endpoint and the port number to connect to the DB instance.

   ![DB Endpoint](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/images/endpoint-port.png)

   4.4 If you need to find the master user name, choose the `Configuration` tab and view the `Master username` value.

   4.5 In the `New Connection` section, set these values:
      
            Connection Name - AWS Cost Connection

            Endpoint - endpoint url
   
            Port - port number
   
            User Name - Master username value

            Password - password value

   4.6 Test Connection & Save
   
   ![Navicat Connection](https://github.com/StayHungryStayFoolish/notebook-img/blob/master/img/MySQL/Navicat%20Connect%20Configuration.png?raw=true)

5. Create DB & Table

   ![Create DB](https://github.com/StayHungryStayFoolish/notebook-img/blob/master/img/MySQL/Navicat-CreateDB.png?raw=true)

   5.1 Create DB & use

```sql
CREATE DATABASE IF NOT EXISTS aws_cost DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

USE aws_cost;
```

   5.2 Create table

```sql
CREATE TABLE IF NOT EXISTS Cost
(
    PayerAccountId         VARCHAR(255), 
    ProductName            VARCHAR(255),
    UsageStartDate         TIMESTAMP,
    UsageEndDate           TIMESTAMP,
    CurrencyCode           VARCHAR(255),
    TotalCost              DECIMAL(20, 9)
    ) ENGINE = InnoDB
    DEFAULT CHARSET = utf8mb4;
```

7. Import Sample Data
   
   7.1 Right-click on the new table name, choose `Import Wizard`

   ![Import Wizard](https://github.com/StayHungryStayFoolish/notebook-img/blob/master/img/MySQL/navicat-import-sample-data.png?raw=true)

   7.2 In `Import Type` choose `CSV file`

   ![Import Type](https://github.com/StayHungryStayFoolish/notebook-img/blob/master/img/MySQL/import-data-csv.png?raw=true)

   7.3 Add csv file, then next `two` steps use the **default** configurations.

   ![Add file](https://github.com/StayHungryStayFoolish/notebook-img/blob/master/img/MySQL/import-data-choose-file.png?raw=true)

   7.4 Fix date format, then next `three` steps use the **default** configurations.

   ![Fix date](https://github.com/StayHungryStayFoolish/notebook-img/blob/master/img/MySQL/import-repair-date.png?raw=true)

   7.5 Start Import

   ![Start](https://github.com/StayHungryStayFoolish/notebook-img/blob/master/img/MySQL/import-start.png?raw=true)

   7.6 Finish
   
   ![Finish](https://github.com/StayHungryStayFoolish/notebook-img/blob/master/img/MySQL/import-finish.png?raw=true)
   