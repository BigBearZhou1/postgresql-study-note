# postgresql 实例

- 用来访问postgresql数据库
- 一个实例对应一个数据库集簇
- 由内存和后台进程组成
- 实例类似word程序，数据库类似word文档

## 连接到一个postgresql实例

- 建立一个用户连接
- 创建一个会话

![image-20230222202327869](D:\learnNotes\postgresql.assets\image-20230222202327869.png)

## 初始化参数文件

- 静态参数文件：postgresql.conf

- 动态参数文件：postgresql.auto.conf 

  实例中就可以修改

- 可选参数文件：postgresql.conf.user

  需要在postgresql.conf中指定

  ![image-20230222203405535](D:\learnNotes\postgresql.assets\image-20230222203405535.png)

- 读取顺序：postgresql.conf>postgresql.auto.conf>postgresql.conf.user

### pg_settings数据字典记录了所有的配置项

```shell
select name,setting,context from pg_settings;

# name 配置名称
# setting 配置值
# context 上下文
## sighup 超管修改，reload生效
## superuser 超管修改
## postmaster 超管修改，重启生效
## user 普通用户修改即可，立即生效
```

## 内存模型

### local memory area

- 后端进程分配给自己使用：**temp_buffers、work_mem、maintenance_work_mem**
- work_mem用在查询，join
- maintenance_work_mem用在VACUUM,REINDEX
- temp_buffers用在临时表中

### shared memory area

- pg服务器给所有的进程使用

- shared buffer pool 持久化表和索引的缓存
- wal buffer
- commit log 事务相关



## 进程结构-Process Architecture

![image-20230222210125312](D:\learnNotes\postgresql.assets\image-20230222210125312.png)

![image-20230222210451301](D:\learnNotes\postgresql.assets\image-20230222210451301.png)

- 父进程postgres

  ```shell
  postgres 17305     1  0 Feb19 ?        00:00:00 /usr/local/postgres12.2/bin/postgres
  ```

- backgroud writer

  ![image-20230222211233813](D:\learnNotes\postgresql.assets\image-20230222211233813.png)

- wal writer

  ![image-20230222211333773](D:\learnNotes\postgresql.assets\image-20230222211333773.png)

- checkpoint

![image-20230222211550510](D:\learnNotes\postgresql.assets\image-20230222211550510.png)









# 数据字典

- pg_proc存储过程和函数

# 表空间-Tablespaces

> 表空间把数据从逻辑或者物理上分割存放

# 用户管理

![image-20230226115638952](D:\learnNotes\postgresql.assets\image-20230226115638952.png)

- pg新建的用户，默认就会有登录数据库、创建对象的权限（表、索引）

![image-20230226133230987](D:\learnNotes\postgresql.assets\image-20230226133230987.png)

# Schema

## Schema概念

- 用户对象的集合叫做模式。可以理解为用户更低一层的分级管理
- 比如一个用户可以有schema1到scheman
- 不同的schema下面可以有同名的表

## 用户与Schema对应关系

- 一个用户可以拥有多个模式
- 一个模式只能属于一个用户
- 普通用户创建模式时需要授权在指定的数据库下创建模式的权限

### 使用schema

1. 授权

   ```SHELL
   GRANT CREATE ON DATABASE testdb TO u1;
   ```

2. 创建模式

   ```shell
   CREATE SCHEMA sport;
   ```

3. 查看

   ```shell
   #查看当前有哪些模式
   \dn
   ```

   ```shell
   u2=> grant create on database u2 to u2;
   GRANT
   u2=> create schema s1u2;
   CREATE SCHEMA
   u2=> \dn
     List of schemas
     Name  |  Owner
   --------+----------
    public | postgres
    s1u2   | u2
   
   ```

4. 创建模式下的表对象

   ```shell
   create table u1.tablename();
   ```

5. 授权别的用户访问schema和schema下面的对象

   单独授权schema下面的对象给别的用户是没有效果的，必须将schema也授权给这个用户
   
   ```shell
   GRANT USAGE ON SCHEMA sch_name TO role_name;
   GRANT SELECT ON sch_name.tab_name TO role_name;
   
   
   u2=> select * from s1u2.test;
   ERROR:  permission denied for schema s1u2
   LINE 1: select * from s1u2.test;
   
   u2=> grant usage on schema s1u2 to u3;
   GRANT
   
   u2=> grant select on s1u2.test to u3;
   GRANT
   ```
   
   

## Public schema

- 数据库起来就自动创建的Public模式，共享给所有用户使用
- public模式时可以被删除的

## Schema 管理

- 删除schema

```shell
DROP SCHEMA schema_name;

DROP SCHEMA schema_name cascade;
```

- 搜索路径

  默认会搜索和当前用户名（$user）相同的schema(不管有没有)和public这两个

  **所以推荐建一个和用户相同名称的schema**

- 搜索规则

  \d如果找到了一个表名，就不会再去搜索路径中后面的schema中找同名的表了，并不代表table被删除了

```shell
show search_path;

set search_path="$user",public,s1u2,sport,art;
```

# Objectives对象权限

### 概述

- 每个数据库对象都有一个所有者，所有者拥有该对象的所有权限
- 数据库中所有的权限都和角色挂钩
- 除了超级用户postgresql，其他用户都要走ACL

### 对象权限列表

![image-20230311121913069](D:\learnNotes\postgresql.assets\image-20230311121913069.png)

### 对象权限授权

1. 语法

![image-20230311122400166](D:\learnNotes\postgresql.assets\image-20230311122400166.png)

2. 实操

   - **注意：只有当用户没赋予了表所在的schema的usage权限才能正常使用这些权限，否则就算赋予了对象权限也没有办法使用**

     ```shell
     grant usage on schema scheme_name to role_name
     ```

     

   ![image-20230311125453015](D:\learnNotes\postgresql.assets\image-20230311125453015.png)

3. 数据字典

   - \z

   ```shell
   u4=> set search_path = "$user", public,u4;
   SET
   u4=> \d
          List of relations
    Schema | Name | Type  | Owner
   --------+------+-------+-------
    u4     | t1   | table | u4
   (1 row)
   
   u4=> \z
                               Access privileges
    Schema | Name | Type  | Access privileges | Column privileges | Policies
   --------+------+-------+-------------------+-------------------+----------
    u4     | t1   | table | u4=arwdDxt/u4    +|                   |
           |      |       | u3=r/u4           |                   |
   (1 row)
   
   ```

   - \dp

   ```shell
   u4=> \dp
                               Access privileges
    Schema | Name | Type  | Access privileges | Column privileges | Policies
   --------+------+-------+-------------------+-------------------+----------
    u4     | t1   | table | u4=arwdDxt/u4    +|                   |
           |      |       | u3=r/u4           |                   |
   (1 row)
   ```

   - information_schema.table_privileges

   ```shell
   select grantor,grantee,privilege_type,is_grantable from information_schema.table_privileges where table_name='t1'
   ```

   ```shell
   u4=> select grantor,grantee,privilege_type,is_grantable from information_schema.table_privileges where table_name='t1';
   
    grantor | grantee | privilege_type | is_grantable
   ---------+---------+----------------+--------------
    u4      | u3      | SELECT         | NO
   (1 row)
   ```

   

   4. 回收权限

   ![image-20230311130841223](D:\learnNotes\postgresql.assets\image-20230311130841223.png)

   5. 对象易主管理

   ![image-20230311131035558](D:\learnNotes\postgresql.assets\image-20230311131035558.png)

#  PG实例连接访问控制

### psql的默认连接方式

```shell
os user: postgres
psql 命令等于： psql -U postgres -d postgres
```

### pg_hba.conf文件

- 客户端连接数据库的配置文件
- 这个文件存放在${PGDATA}路径下面

### 配置项内容

- type: 指定连接类型

  1. TYPE ： 指定连接类型

     local、host、hostssl

     本地连接不需要指定-h -p

  2. DATABASE ：指定连接的数据库

     all: 表示所有的数据库

     db_name: 表示指定的数据库

     replication: 表示主备复制时的连接

  3. USER：指定连接用户

     all :

     user_name :

     +group_name :

     @file_name : 文件中包含的用户列表

  4. ADDRESS：指定连接的客户端

     127.0.0.1/32 本机

     0.0.0.0/0 所有客户端主机

     host_name 在host文件中指定的

     ip_addr/mask 

  5. METHOD 验证方式

     trust 无需密码

     password 明文密码

     md5 加密

     scram-sha-256

  ### 常见配置

  1. 配置某个ip可以访问数据库

  ```shell
  host	all		all 	192.168.16.7	md5
  ```

  2. 所有0.0.0.0/0

  ```shell
  host	all		all 	0.0.0.0/0	md5p
  ```

  ### 使修改生效pg_ctl reload

  ```shell
  [postgres@localhost data]$ pg_ctl reload
  server signaled
  ```

  ### 生效规则

  1. 在最前面的优先生效

  ### 还需要修改postgresql.conf文件的listen_address = '*' 才能使别的主机访问成功

  1. postgresql.conf文件也再${PGDATA}下面
  2. 修改listen_address='*'，这个配置项原先是被注释掉的，先接触注释
  3. 重启pg

  ```shell
  pg_ctl re
  ```

  

# PG数据库管理

 ## 数据库属主

- pg中的数据库属主属于创建者，只要有createdb权限的用户就可以创建数据库

- 数据库属主不一定拥有存放在该数据库中其他用户创建的对象的访问权限

- 数据库在创建后，允许public角色连接，即允许任何人连接

- 数据库创建后，任何用户都可以创建表

  原因是：数据库创建后会自动创建public的schema，这个schema的all权限已经赋予了public角色（即全体用户）

- 数据库创建后，不允许除了超级用户和属主之外的任何人在数据库中创建schema

## 数据库权限

- CREATE：可以在指定数据库创建schema的权限 ,简写 C
- CONNECT：可以连接到指定数据库的权限，简写c
- TEMPORARY： 可以创建临时表的权限，简写T
- ALL：指定数据库所有的权限

![image-20230314231458508](D:\learnNotes\postgresql.assets\image-20230314231458508.png)

## 取消某个用户的数据库访问权限，需要先取消public权限

```shell
revoke connect on database db_name from public;

revoke connect on database db_name from user_name;
```

## 数据字典

- 查看哪些用户有某个数据库的connect权限

```shell
select datname , datacl from pg_database where datname='bd_name';
```



# PostgreSQL的使用

## 注释

1. --
2. /**/

## 转义字符-ESCAPE

1. 场景：模糊匹配字符like和%配合使用时，想要查询文本内容有%的该如何处理呢？
2. 通过关键字ESCAPE来制定强制转义字符

```shell
select * from table_name where column_name like '%25#%%' ESCAPE '#';
```

3. 默认的转义字符是"\"

```sql
select * from table_name where column_name like '%25\%%';
```

## 忽略大小写的字符模糊匹配 - ILIKE

```sql
select column_name from table_name where table_name ILIKE '%x%';
```

## IS NULL or ISNULL and not Null

```sql
select * from table_name where column_name is null;
```

## 优先级NOT>AND>OR，使用括号“（）”可以改变优先级



## 集合并集

1. 默认union 全称是 distinct
2. union all对重复的行也会再次展示

## 集合交集

1. intersect

```sql
select emp_id from excellent_emp where year = 2018
intersect
select emp_id from excellent_emp where year = 2019;
```

2. 和union一样intersect也有默认distinct和all

## 集合差集

1. except

```sql
-- 2019年获得优秀员工称号新进的员工
select * from excellent_emp where year = 2019 
except 
select * from excellent_emp where year < 2019;
```

2. except和 union以及intersect一样，也有默认distinct和all之间的配置

## 连表

