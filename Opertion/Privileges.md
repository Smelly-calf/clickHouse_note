# Privileges
`SHOW PRIVILEGES;` 查看所有特权

PRIVILEGES 从低到高有7个等级：
-  COLUMN: COLUMN 特权可以被授予 column, table, database, or globally
-  TABLE
-  VIEW: view, database, or globally
-  DICTIONARY: dictionary, database, or globally.
-  DATABASE
-  GLOBAL: only globally
-  GROUP: 不同级别的特权会被分组，授予组级别的特权只会拥有该组下的特权 `GRANT SOURCES ON db.table TO user`  
    
`select * from system.privileges where parent_group='ACCESS MANAGEMENT'` 查看ACCESS MANAGEMENT组下面的特权：

- ACCESS MANAGEMENT （level: GLOBAL）
    - CREATE USER
    - ALTER USER
    - DROP USER
    - CREATE ROLE
    - ALTER ROLE
    - DROP ROLE
    - CREATE ROW POLICY
    - ALTER ROW POLICY
    - DROP ROW POLICY
    - CREATE QUOTA
    - ALTER QUOTA
    - DROP QUOTA
    - CREATE SETTINGS PROFILE
    - ALTER SETTINGS PROFILE
    - DROP SETTINGS PROFILE
    - SHOW ACCESS
        - SHOW_USERS
        - SHOW_ROLES
        - SHOW_ROW_POLICIES
        - SHOW_QUOTAS
        - SHOW_SETTINGS_PROFILES
    - ROLE ADMIN（ROLE ADMIN特权允许用户分配和撤消任何角色，包括未使用admin选项分配给该用户的角色）
    
- SOURCES （level: GLOBAL）
  - FILE
  - URL
  - REMOTE
  - MYSQL
  - ODBC
  - JDBC
  - HDFS
  - S3
 
连接 SOURCES： MySQL，MySQL引擎允许您对存储在远程MySQL服务器上的数据执行SELECT查询。

`show grants for default;` 查询default的权限，发现有 SOURCES on *.* 的权限。

`ENGINE = MySQL('host:port', 'database', 'table', 'user', 'password'[, replace_query, 'on_duplicate_clause'])`

1、进入 mysql client 终端：`mysql -uroot`

建表
```sql
CREATE TABLE `test`.`test` (
    `int_id` INT NOT NULL AUTO_INCREMENT,
    `int_nullable` INT NULL DEFAULT NULL,
    `float` FLOAT NOT NULL,
    `float_nullable` FLOAT NULL DEFAULT NULL,
    PRIMARY KEY (`int_id`));
```
插入数据
```sql
insert into test (`int_id`, `float`) VALUES (1,2);
```

2、进入 docker clickhouse client 终端：命令参考 `Introduction/docker_ck.md` 

建 clickhouse 表
```sql
CREATE TABLE mysql_table
(
    `float_nullable` Nullable(Float32),
    `int_id` Int32
)
ENGINE = MySQL('adminhost:3306', 'test', 'test', 'root', 'root')
```
查询 mysql 数据:
```
SELECT * FROM mysql_table
```
结果：

|float_nullable|int_id|
|----|----|
|   ᴺᵁᴸᴸ |      1 |

为了测试其他用户没有权限，让我们新建一个账号为 wq 的用户并用它登录：发现 default 账号没有SQL-Driven权限，需要开启，见 `Opertion/AccessControlAndAccountManagement.md`
```
> CREATE USER wq IDENTIFIED WITH plaintext_password BY '123';
> SELECT * FROM mysql_table;
```

没有权限：
```
Code: 497. DB::Exception: Received from clickhouse-server:9000. DB::Exception: wq: Not enough privileges. To execute this query it's necessary to have the grant SELECT(float_nullable, int_id) ON default.mysql_table.
```
切到 default 账户重新进入交互式终端执行 `GRANT SELECT ON default.mysql_table TO wq WITH GRANT OPTION`
切到 wq 账户进入终端：`select * from mysql_table` 成功。
    