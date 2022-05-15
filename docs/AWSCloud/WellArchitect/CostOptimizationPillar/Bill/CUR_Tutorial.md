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
      
   3.5 In the Settings section, set these values:

           DB instance identifier – cost-db-instance

           Master username – admin

           Auto generate a password – Clear the check box.

           Master password – Choose a password.

           Confirm password – Retype the password.
   
   **Use Navicat( other db tools) must use `Master username & Master password`, pls record info.**

   3.6 In the `Instance configuration` section, , choose the template that meets your needs.   

   3.7 In the `Storage` and `Availability & durability` sections, use the default values.

   3.8 In the `Connectivity` section, set these values:

            Network type - IPv4

            Virtual private cloud (VPC) - Choose an existing VPC with both public

            Subnet group - Choose default 

            Public access - Choose Yes

            VPC security group - Choose an exisiting security groups

   3.9 Other sections, use the default values.

   3.10 `Create database`

4. Use Navicat Connect MySQL DB

5. Create DB & Table

```sql
# Create DB
CREATE DATABASE IF NOT EXISTS bills_db DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE bills_db;

# Create Sample Table
CREATE TABLE cost
(
    ProductName            VARCHAR(255),
    UsageStartDate         TIMESTAMP,
    UsageEndDate           TIMESTAMP,
    CurrencyCode           VARCHAR(255),
    TotalCost              DECIMAL(20, 9)
    ) ENGINE = InnoDB
    DEFAULT CHARSET = utf8mb4;
```

7. Import Sample Data

```sql

```