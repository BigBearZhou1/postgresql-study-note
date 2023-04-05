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



# 排序

1. 在pg中null值默认是最大值，升序排在最后面
2. 可以使用nulls first 或者nulls last来指定null值的排序

# 控制查询输出的条数

1. TOP-N
   - fetch first 10 rows only
   - limit
   - 如果最后一个数值有多个相同的rows，**FETCH FIRST 10 ROWS WITH TIES**来抓取出来

```sql
select * from employees order by salary desc fetch first 10 rows only ;
select * from employees order by salary limit 10 ;
```

2. 

# 集合并集

1. 默认union 全称是 distinct
2. union all对重复的行也会再次展示

# 集合交集

1. intersect

```sql
select emp_id from excellent_emp where year = 2018
intersect
select emp_id from excellent_emp where year = 2019;
```

2. 和union一样intersect也有默认distinct和all

# 集合差集

1. except

```sql
-- 2019年获得优秀员工称号新进的员工
select * from excellent_emp where year = 2019 
except 
select * from excellent_emp where year < 2019;
```

2. except和 union以及intersect一样，也有默认distinct和all之间的配置

# 分组

## group by rollup(col1,col2)

```sql
select item,year,sum(sales)
from employee
group by rollup (item,year);

-- rollup 相当于 group by item+year 和group by item 二合一
```

## coalesce

- 对于为null的情况做的特殊处理

```sql
select coalesce(item,'所有产品') as '产品',coalesce(year,'所有年份') as '年份',sum(sales)
from employee
group by rollup(item,year);
```

## group by cube(col1,col2)

- 字段的所有排列组合，汇总

```sql
select coalesce(item,'所有产品') as '产品',coalesce(year,'所有年份') as '年份',sum(sales)
from employee
group by cube(item,year);
```

## group by grouping sets 

- group by rollup 和group by cube是group by grouping sets的特殊场景

```sql
group by rollup (col1,col2)相当于group by grouping sets ((col1,col2),(col1),())
group by cube (col1,col2) 相当于group by grouping sets((col1,col2),(col2),(col1),())
```



# 连表

## cross join 笛卡尔积

- 九九乘法表

```sql
select concat(t1,' * ',t2,' = ',t1*t2)
from generate_series(1,9) t1
cross join generate_series(1,9)t2
```

## 自连接

- 找出员工对应的经理

```sql
select t1.first_name ,t1.last_name ,t2.first_name ,t2.last_name 
from employees t1
left join employees t2
on t1.manager_id =t2.employee_id 
```



# 数据字典

## 查看表大小

```sql
select pg_size_pretty(pg_table_size('table_name'));
```

```shell
pg_size_pretty|
--------------|
40 kB         |
```

## 查看索引大小

```sql
select pg_size_pretty(pg_relation_size('index_name'));
```



# 函数

## 数字函数

1. 随机选取10个员工
   - 使用order by random() 函数

```sql
select first_name 
from employees
order by random()
limit 10;
```

## 字符函数

### 连接字符串

1. concat

```sql
select concat('s','q','l');
```

2. concat_ws
   - 用分隔符连接

```sql
select concat_ws('-','s','q','l');
```

3. ”||“连接多个字符串

```SQL
select 's'||'q'||'l';
```

### 字符串截取

1. substring(string from startIndex for length)

```sql
select substring('Thomas' from 2 for 3);

substring|
---------|
hom      |
```

2. substr(string,FROM,count)

```sql
select substr('Thomas',2,3);

substr|
------|
hom   |
```

### 字符串替换

```sql
select replace('abcbcd','bc','xx');

replace|
-------|
axxxxd |
```

### trim

- ([leading|trailing|both] [characters] from string )

- 开头|结尾|开头+结尾 去掉多余字符（默认是空格）

### format

- 类似java的String.format

### MD5

```sql
select md5('abc');

md5                             |
--------------------------------|
900150983cd24fb0d6963f7d28e17f72|
```

## 时间函数

1. 换取当前时间

   - current_date 年月日
   - current_time 时间
   - current_timestamp 时间戳

   ```sql
   select current_date,current_time,current_timestamp;
   ```

2. 时间运算

   - 加时间段interval

   ```sql 
   select current_date + interval '7 month';
   ```

   - 时间间隔

     1. age(timestamp,timestamp)

        返回的是interval

        ```sql
        select age(timestamp '2020-12-3',timestamp '2020-01-01')
        ```

     2. 直接相减

        ```sql
        select current_date - date '2023-01-01'
        ```

3. 获取时间段中的信息

   - date_part(text,timestamp)

     ```sql
     select date_part('year',current_date);
     
     select date_part('hour',timestamp '2023-02-11 03:01:00')
     ```

   - extract(field from timestamp)

     extrect函数实际上也是调用date_part函数，只是语法上有不同

4. date_trunc

   - 截断日期/时间

     date_trunc(field,source) 将timestamp、date、time或者interval数据截取到指定的精度

     最后返回source的类型去掉精度的变量

   ```sql
   select date_trunc('year',timestamp '2023-03-03 11:00:00')
   
   select date_trunc('year',current_timestamp)
   ```

5. 创建时间make_date

# 类型转换

1. cast函数

   CAST( expr AS date_type)

2. expr :: date_type

3. to_date(string, format) 

4. 隐式类型转换

   ```sql
   select 1 + '2','todo: '|| current_timestamp;
   ```

   

# 子查询

## 关联子查询

1. 查询分为内外两层，内层的查询条件借用外层查询的结果作为入参
2. 查询每个部门的工资总和

```sql
select d.department_name ,(select sum(salary ) from employees where department_id = d.department_id )
from departments d
```

3. 查询每个部门中超过平均薪酬的员工

```sql
select * 
from employees e
where salary >(select avg(salary ) from employees where department_id = e.department_id )
```

## 横向子查询 - join lateral

1. 查询每个部门的工资总和

```sql
select d.department_name ,t.sum_sal
from departments d 
join lateral (select sum(salary) sum_sal from employees where department_id =d.department_id ) t
on true;
```

2. 查询每个部门工资前三的员工

```sql
select d.department_name ,e.first_name, e.salary
from departments d 
join lateral (select first_name,salary from employees where department_id =d.department_id order by salary desc limit 3) e
on true ;
```





# 时序数据存储引擎 MARS2

## 有序存储

1. 数据按照排序键存储（稀疏索引）：如列device、ts
2. 非排序键，采用**minmax过滤**：如列：metric_1、metric_2、comment

## 列存

1. 列裁剪
2. 压缩
3. 向量化

## 数据布局

1. 类似LSM Tree结构（LSM是什么？）

​		磁盘顺序读写

## 使用

### 创建表

```SQL
create table metrics(
	device_id varchar(32) collate "C" ENCODING(minmax), --minmax过滤
    ts TIMESTAMP ENCODING(minmax),
    metric_1 FLOAT4(16),
    metric_2 INT4,
    comment varchar(32),
)using mars2                                            --使用mars2存储引擎
distributed by (device_id)
partition by range(ts);

create index on metrix using mars2_btree(device_id,ts); --排序键：定义物理存储的顺序，建议将最常用的查询条件定义为排序键
```

### UniqueMode

1. 解决update修改问题
2. 语法

```sql
create index on table_name using mars2_btree(column_name) with (uniquemode=true);

非json类型
k=1 a=1   + k1 a=2 => k=1 a=2
k=1 a=1   + k1 a=null => k=1 a=1
```

### 数据插入后analyz表

### kafka可以直接如matricDB

### count(*)调整为持续聚集

- 持续聚集是什么？