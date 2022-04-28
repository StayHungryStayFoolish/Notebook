# SQL Tutorial

## Prerequisites

1. Install Navicat(MySQL Client)
Navicat Download: https://www.navicat.com/en/download/navicat-premium

2. Sign up for AWS 
   
    2.1 Open https://portal.aws.amazon.com/billing/signup.

    2.2 Follow the online instructions. Part of the sign-up procedure involves receiving a phone call and entering a verification code on the phone keypad.

3. Creating a MySQL DB instance

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