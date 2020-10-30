# 访问控制 & 账号管理
ClickHouse 支持基于 RBAC 方法的访问控制管理。

https://clickhouse.tech/docs/en/operations/access-rights/

## 可授权的实体包括
- User account
- Role
- Row Policy
- Settings Profile
- Quota

两种设置访问实体的方式：
- <em><b>SQL-driven 驱动的方式</b></em>（手动 enable: 1. access_control_path 指定访问实体配置的存储路径；2. enable SQL-driven -> access_management 设置为 1）
- config.xml 和 users.xml 服务器的配置文件

## 特性
- 可以授予权限给不存在的库表
- 表被删除不会回收该表对应的权限，重新创建相同名称表后拥有权限；回收已删除表对应的权限需要执行 `REVOKE ALL PRIVILEGES ON db.table FROM ALL` 
- 权限无生命周期设置


## User Account
账号是一个可授予权限的实体，一个账号包含：
- 认证信息
- 定义用户可执行的查询范围的 privileges
- 允许连接ClickHouse服务器的hosts
- Assigned and default roles.
- 用户登录时默认应用设置及其约束
- 分配的 profiles 设置.

`GRANT {privilege} ON db.table TO {user} [WITH GRANT OPTION] ` 查询语句授予账号privileges

`REVOKE {privilege} ON db.table FROM {user}` 查询语句收回账号的privileges

`REVOKE [ADMIN OPTION FOR] {role} FROM {user}` query 收回账号的角色（ROLE ADMIN特权允许用户分配和撤消任何角色，包括未使用admin选项分配给该用户的角色）

`SHOW GRANTS [FOR user]` list一个用户所有privileges.


管理的查询语句：
- CREATE USER
- ALTER USER
- DROP USER
- SHOW CREATE USER
- SHOW USERS


## Settings Applying 应用配置
3种配置方式：为 user account 配置，为用户授予角色，或在 profiles 配置。
不同方式配置的优先级从高到低如下：
1. user account 配置
2. 账号的 default roles 配置，多个角色应用配置的顺序随机
3. 配置文件profiles中的配置，多个profiles应用的顺序随机
4. 应用于所有服务器的 default 配置或者 default profiles 中的配置

## Role
Role是一个可以被授予账号的访问实体的容器，role包括：
- Privileges
- 配置和约束：Settings and constraints
- List of assigned roles

管理的查询语句：
- CREATE ROLE
- ALTER ROLE
- DROP ROLE
- SET ROLE
- SET DEFAULT ROLE
- SHOW CREATE ROLE
- SHOW ROLES

`GRANT ON db.table TO {user|role} [WITH GRANT OPTION]` 授予privileges, `REVOKE` 收回privileges.

WITH GRANT OPTION子句向用户或角色授予执行GRANT查询的权限

## Row Policy
Row policy 是一个过滤器，定义了用户或者角色可使用的行。 Row policy包含一个特定表的过滤器和应使用该策略的角色或用户列表。

Management queries:
- CREATE ROW POLICY
- ALTER ROW POLICY
- DROP ROW POLICY
- SHOW CREATE ROW POLICY
- SHOW POLICIES

## Settings Profile
settings profile 是一个settings集合，包含配置和约束以及应用的roles/users列表.

Management queries:
- CREATE SETTINGS PROFILE
- ALTER SETTINGS PROFILE
- DROP SETTINGS PROFILE
- SHOW CREATE SETTINGS PROFILE
- SHOW PROFILES

## Quota 限额/指标
限制资源的使用。

Quota包含一段时间的一系列限制，以及使用该 Quota 的 roles/users 列表。

Management queries:
- CREATE QUOTA
- ALTER QUOTA
- DROP QUOTA
- SHOW CREATE QUOTA
- SHOW QUOTA
- SHOW QUOTAS