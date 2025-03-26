## 安装MySQL服务

安装MySQL前，先在linux环境下使用centos7系统安装docker。

如果yum报错，就执行一下yum换源

```Bash
cd /etc/yum.repos.d

mv CentOS-Base.repo CentOS-Base.repo.bak

wget -O CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo

yum clean all

yum makecache

yum -y install update
```



安装docker

```Go
yum -y remove docker*

yum install -y yum-utils

yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo 

yum install docker-ce docker-ce-cli containerd.io -y
// 启动docker
systemctl start docker
// 开机启动
systemctl enable docker

```

### docker配置代理

因为众所周知的原因，我们国内使用docker下载镜像没办法下载了



有三种办法下载镜像

1. 一些云服务器可以使用内部docker源下载镜像，比如腾讯云

2. 在能下载镜像的地方下载好，然后在不能下载镜像的地方导入

    docker save -o mysql_5_7.tar mysql:5.7

    docker load < mysql_5_7.tar

3. docker配置代理

```bash
mkdir /etc/systemd/system/docker.service.d
创建 /etc/systemd/system/docker.service.d/http-proxy.conf 配置文件

mkdir -p /etc/systemd/system/docker.service.d
vi /etc/systemd/system/docker.service.d/http-proxy.conf

[Service]
Environment="HTTP_PROXY=http://本机ip地址:代理端口号"
Environment="HTTPS_PROXY=http://本机ip地址:代理端口号"

例如:
Environment="HTTPS_PROXY=http://10.129.231.64:7890"

然后重新加载配置并重启服务:
systemctl daemon-reload
systemctl restart docker

然后检查加载的配置:
systemctl show docker --property Environment
```

### docker安装MySQL服务

```mysql
docker pull mysql:5.7

mkdir -p ./mysql/conf ./mysql/datadir

# 直接运行
docker run -itd --name mysql -p 3307:3306 -e MYSQL_ROOT_PASSWORD=root mysql:5.7

# 映射目录
docker run -itd --name mysql --restart=always -p 3307:3306 -v /opt/db/mysql/conf:/etc/mysql/conf.d -v /opt/db/mysql/datadir:/var/lib/mysql -e  MYSQL_ROOT_PASSWORD=root -e MYSQL_DATABASE=db mysql:5.7
```
`docker run -itd --name mysql --restart=always` 
`-p 3307:3306` 
`-v /opt/db/mysql/conf:/etc/mysql/conf.d` 
`-v /opt/db/mysql/datadir:/var/lib/mysql` 
`-e  MYSQL_ROOT_PASSWORD=root`
`-e MYSQL_DATABASE=db` 
`mysql:5.7`
创建一个名为db 的数据库，用户root的密码是root，端口映射为本机的3307映射到容器的3306，
-v是映射目录，主要是mysql的配置和数据，这样重启容器，数据还会保持。
如果你已经有mysql服务了，那你换一个端口就行了。

```bash
# 进入docker容器里的mysql
docker exec -it mysql bash
mysql -uroot -proot
```



用docker的好处就是，版本可以随便换。

---

## MySQL基础使用
Windows下连接MySQL
```bash
// 连接本机的3306端口上的mysql服务
mysql -uroot -proot

// 连接本机的其他端口
mysql -uroot -proot -P 3307

//连接其他主机的其他端口
mysql -uroot -proot -P 3307 -h 192.168.179.135
```



## MySQL数据库

- 关系型数据库 (RDBMS)

  概念：建立在关系模型基础上，由多张相互连接的二维表组成的数据库。

----



## SQL的分类 

### 1.DDL

1. **DDL(Data Definition Language) 数据定义语言，用来操作数据库、表、列等； 常用语句：CREATE、 ALTER、DROP**

   - DDL-数据库操作

   ```mysql
   # 查询所有数据库
   show databases;
   
   # 查询当前数据库
   select database();
   
   # 创建
   create database [if not exists]数据库名 [default charset 字符集] [collate 排序规则];
   
   # 删除
   drop database [if exists]数据库名;
   
   # 使用
   use 数据库名;
   ```

   - DDL-表操作-查询  

   ```mysql
   # 查询当前数据库所有表
   show tables;
   
   # 查询表结构
   desc 表名;
   
   # 查询指定表的建表语句
   show create table 表名;
   ```

   - DDL-表操作-创建

   ```mysql
   create table 表名(
       字段1 字段1类型[comment 字段1注释],
       字段2 字段2类型[comment 字段2注释],
       字段3 字段3类型[comment 字段3注释],
       ...
       字段n 字段n类型[comment 字段n注释],
   )[comment 标注释];
   
   create table tb_user(
   	id 		int 			comment '编号',
       name 	varchar(50) 	comment '姓名',
       age 	int 			comment '年龄',
       gender  varchar(1)		comment '性别'
   ) comment '用户表';
   ```

   - DDL-表操作-数据类型

   1. 整数类型  

   根据数值取值范围的不同MySQL 中的整数类型可分为5种，分别是TINYINT、SMALUNT、MEDIUMINT、INT和 BIGINT。下图列举了 MySQL不同整数类型所对应的字节大小和取值范围而最常用的为INT类型。

   | 数据类型  | 字节数 | 无符号数的取值范围     | 有符号数的取值范围                       |
   | --------- | ------ | ---------------------- | ---------------------------------------- |
   | TINYINT   | 1      | 0~255                  | -128~127                                 |
   | SMALLINT  | 2      | 0~65535                | -32768~32767                             |
   | MEDIUMINT | 3      | 0~16777215             | -8388608~8388607                         |
   | INT       | 4      | 0~4294967295           | -2147483648~2147483647                   |
   | BIGINT    | 8      | 0~18446744073709551615 | -9223372036854775808~9223372036854775807 |

   2. 浮点数类型和定点数类型  

   在MySQL数据库中使用浮点数和定点数来存储小数。浮点数的类型有两种：单精度浮点数类型（FLOAT)和双精度浮点数类型（DOUBLE)。而定点数类型只有一种即DECIMAL类型。下图列举了 MySQL中浮点数和定点数类型所对应的字节大小及其取值范围：

   | 数据类型      | 字节数 | 有符号的取值范围                                 | 无符号的取值范围          |
   | ------------- | ------ | ------------------------------------------------ | ------------------------- |
   | FLOAT         | 4      | -3.402823466E+38~1.175494351E-38                 | 0~3.402823466E+38         |
   | DOUBLE        | 8      | -1.7976931348623157E+308~2.2250738585072014E-308 | 0~1.7976931348623157E+308 |
   | DECIMAL (M,D) | M+2    | -1.7976931348623157E+308~2.2250738585072014E-308 | 0~1.7976931348623157E+308 |

   3. 字符串类型 

   文本类型用于表示大文本数据，例如，文章内容、评论、详情等，它的类型分为如下4种：

   | 数据类型   | 储存范围         |
   | ---------- | ---------------- |
   | TINYTEXT   | 0~255字节        |
   | TEXT       | 0~65535字节      |
   | MEDIUMTEXT | 0~16777215字节   |
   | LONGTEXT   | 0~4294967295字节 |

   4. 字符串类型

   在MySQL中常用CHAR 和 VARCHAR 表示字符串。两者不同的是：VARCHAR存储可变长度的字符串。**当数据为CHAR(M)类型时，不管插入值的长度是实际是多少它所占用的存储空间都是M个字节；而VARCHAR(M)所对应的数据所占用的字节数为实际长度加1**。

   | 插入值 | CHAR(3) | 存储需求 | VARCHAR(3) | 存储需求 |
   | ------ | ------- | -------- | ---------- | -------- |
   | ""     | ""      | 3个字节  | ""         | 1个字节  |
   | 'a'    | 'a'     | 3个字节  | 'a'        | 2个字节  |
   | 'ab'   | 'ab'    | 3个字节  | 'ab'       | 3个字节  |
   | 'abc'  | 'abc'   | 3个字节  | 'abc'      | 4个字节  |
   | 'abcd' | 'abc'   | 3个字节  | 'abc'      | 4个字节  |

   5. 日期与时间类型  

   MySQL提供的表示日期和时间的数据类型分别是 ：YEAR、DATE、TIME、DATETIME 和 TIMESTAMP。下图列举了日期和时间数据类型所对应的字节数、取值范围、日期格式以及零值：

   | 数据类型  | 字节数 | 取值范围                                   | 日期格式            | 零值                |
   | --------- | ------ | ------------------------------------------ | ------------------- | ------------------- |
   | YEAR      | 1      | 1901-2155                                  | YYYY                | 0000                |
   | DATE      | 4      | 1000-01-01 到 9999-12-31                   | YYYY-MM-DD          | 0000-00-00          |
   | TIME      | 3      | -838:59:59 到 838:59:59                    | HH:MM:SS            | 00:00:00            |
   | DATETIME  | 8      | 1000-01-01 00:00:00 到 9999-12-31 23:59:59 | YYYY-MM-DD HH:MM:SS | 0000-00-00 00:00:00 |
   | TIMESTAMP | 4      | 1970-01-01 00:00:01 到 2038-01-19 03:14:07 | YYYY-MM-DD HH:MM:SS | 0000-00-00 00:00:00 |

   6. 二进制类型  

   在MySQL中常用BLOB存储二进制类型的数据，例如：图片、PDF文档等。BLOB类型分为如下四种：

   | 数据类型   | 储存范围         |
   | ---------- | ---------------- |
   | TINYBLOB   | 0~255字节        |
   | BLOB       | 0~65535字节      |
   | MEDIUMBLOB | 0~16777215字节   |
   | LONGBLOB   | 0~4294967295字节 |
   | TINYTEXT   | 0~255字节        |
   | TEXT       | 0~65535字节      |
   | MEDIUMTEXT | 0~16777215字节   |
   | LONGTEXT   | 0~4294967295字节 |

   - DDL-表操作-修改 

   ```mysql
   # 添加字段
   alter table 表名 add 字段名 类型(长度) [comment 注释] [约束]; 
   alter table emp add nickname varchar(20); # 为emp表增加一个新的字段“昵称”为nickname，类型为varchar(20)
   
   # 修改数据类型
   alter table 表名 modify 字段名 新数据类型(长度);
   
   # 修改字段名和字段类型
   alter table 表名 change 旧字段名 新字段名 类型(长度) [comment 注释] [约束];
   alter table emp change nickname username varchar(30) comment '用户名'# 将emp表的nickname字段修改为username，类型为varchar(30)
   
   # 修改表名
   alter table 表名 rename to 新表名;
   ```

   - DDL-表操作-删除 

   ```mysql
   # 删除表
   drop table [if exists] 表名;
   ```

### 2.DML 

2. **DML(Data Manipulation Language) 数据操作语言，用来操作数据库中表里的数据；常用语句：INSERT、 UPDATE、 DELETE**   

   - DML-添加数据 

   ```mysql
   # 给指定字段添加数据
   insert into 表名 (字段名1, 字段名2, ...) values (值1, 值2, ...);
   
   # 给全部字段添加数据
   insert into 表名 values(值1, 值2, ...);
   
   # 批量添加数据
   insert into 表名 (字段名1, 字段名2, ...) values(值1, 值2, ...), (值1, 值2, ...);
   insert into 表名 values(值1, 值2, ...), (值1, 值2, ...);
   ```

   - DML-修改数据 

   ```mysql
   update 表名 set 字段名1=值1, 字段名2=值2, ...[where 条件];
   update employee set name = 'xiaoyao' where id = 1;
   ```

   - DML-删除数据 

   ```mysql
   delete from 表名 [where 条件];
   ```

### 3.DCL

3. **DCL(Data Control Language) 数据控制语言，用来操作访问权限和安全级别； 常用语句：GRANT、DENY** 

   - DCL-管理用户

   ```mysql
   # 查询用户
   use mysql;
   select * from user;
   
   # 创建用户
   create user '用户名@主机名' identified by '密码';
   
   # 修改用户密码
   alter user '用户名@主机名' identified with mysql_native_password by '新密码';
   
   # 删除用户
   drop user '用户名@主机名';
   ```

### 4.DQL

4. **DQL(Data Query Language) 数据查询语言，用来查询数据 常用语句：SELECT** 

   - DQL-基本查询 

   ```mysql
   # 查询多个字段
   select 字段1, 字段2, 字段3, ... from 表名;
   
   # 查询所有字段
   select * from 表名;
   
   # 设置别名
   select 字段1[as 别名1], 字段2[as 别名2] ... from 表名; # as 可以省略
   select workaddress as '工作地址' from emp;
   select workaddress '工作地址' from emp;
   
   # 去除重复记录
   select distinct 字段列表 from 表名;
   select distinct workaddress '工作地址' from emp;
   ```

   - DQL-条件查询 

   ```mysql 
   select 字段列表 from 表名 where 条件列表;
   
   select * from emp where age = 28;
   
   select * from emp where age <= 20;
   
   select * from emp where idcard is null;
   
   select * form emp where age != 88;
   
   select * from emp where age <> 88;
   
   select * from emp where age >= 15 and age <=20;
   
   select * from emp where age between 15 and 20;
   
   select * from emp in(18, 20, 40);
   
   select * from emp where name like '__';
   
   select * from emp where idcard like '%X';
   ```

   | 运算符              | 说明                                     |
   | ------------------- | ---------------------------------------- |
   | =                   | 等于                                     |
   | <>                  | 不等于                                   |
   | !=                  | 不等于                                   |
   | <                   | 小于                                     |
   | <=                  | 小于等于                                 |
   | >                   | 大于                                     |
   | >=                  | 大于等于                                 |
   | BETWEEN ... AND ... | 在某个范围之内                           |
   | IN(...)             | 在IN之后的列表中的值，多选一             |
   | LIKE 占位符         | 模糊匹配(_匹配单个字符，%匹配任意个字符) |
   | IS NULL             | 是NULL                                   |
   | AND 或 &&           | 并且(多个条件同时成立)                   |
   | OR 或 \|\|          | 或者(多个条件任意一个成立)               |
   | NOT 或 ！           | 非，不是                                 |

   - DQL-聚合函数 

   | 函数  | 功能     |
   | ----- | -------- |
   | count | 统计数量 |
   | max   | 最大值   |
   | min   | 最小值   |
   | avg   | 平均值   |
   | sum   | 求和     |

   ```mysql
   select 聚合函数(字段列表) from 表名;
   
   select count(*) from emp;
   
   select avg(age) from emp;
   
   select max(age) from emp;
   ```

   - 分组查询 

   ```mysql
   select 字段列表 from 表名 [where 条件] group by 分组字段名 [having 分组后过滤条件];
   
   # where 与 having区别
   # - 执行时机不同: where是分组之前进行过滤，不满足where条件，不参与分组；而having是分组之后对结果进行过滤。
   # - 判断条件不同: where不能对聚合函数进行判断，而having可以。
   
   select gender, count(*) from emp group by gender;
   
   # 查询年龄小于45的员工，并根据工作地址分组，获取员工数量大于等于3的工作地址
   select workaddress, count(*) from emp where age < 45 group by workaddress having count(*) >= 3;
   
   # 或者给count(*)加上别名address_count
   select workaddress, count(*) address_count from emp where age < 45 group by workaddress having address_count >= 3;
   
   # 分组之后，查询的字段一般为聚合函数和分组字段，查询其他字段无任何意义
   ```

   - 排序查询 

   ```mysql
   select 字段列表 from 表名 group by 字段1 排序方式1, 字段2 排序方式2;
   
   # 排序方式
   # - ASC: 升序
   # - DESC: 降序
   
   # 注意：如果是多字段排序，当第一个字段值相同时，才会根据第二个字段进行排序。
   
   # 根据年龄对公司的员工进行升序排序
   select * from emp order by aeg asc;
   
   # 根据入职时间，对员工进行降序排序
   select * from emp order by entrydate desc;
   
   # 根据年龄对公司的员工进行升序排序，年龄相同，再按照入职的时间进行排序
   select * from emp order by age asc, entrydate desc;
   ```

   -  分页查询 

   ```mysql
   select 字段列表 from 表名 limit 起始索引，查询记录数;
   
   # 注意
   # - 起始索引从0开始，起始索引 = (查询页码 - 1) * 每页显示记录数
   # - 分页查询是数据库的方言, 不同的数据库有着不同的实现, MySQL中式limit
   # - 如果查询的是第一页数据, 起始索引可以省略，直接简写为limit 10。
   
   # 查询第一页员工数据, 每页展示10条记录
   select * from emp limit 0, 10;
   
   # 查询第二页员工数据, 每页展示10条记录
   select * from emp limit 10, 10;
   ```

   练习

   ```mysql
   # 查询年龄为20, 21, 22, 23岁的女性员工信息
   select * from emp where gender = '女' and age in(20,21,22,23);
   
   # 查询性别为男，并且年龄在20-40岁(含)以内的姓名为三个字的员工
   select * from emp where gender = '男' and (age between 20 and 40) and name like '___';
   
   # 统计员工表中，年龄小于60岁的，男性员工和女性员工的人数
   select gender, count(*) from emp where age < 60 group by gender;
   
   # 查询所有年龄小于等于35岁员工的姓名和年龄, 并对查询结果升序排序, 如果年龄相同按入职时间降序排序
   select name, age from emp where age <= 35 order by age asc, entrydate desc;
   
   # 查询性别为男，且年龄在20-40岁(含)以内的前5个员工信息，对查询的结果按年龄升序排序，年龄相同按入职时间升序排序
   select * from emp where gender='男' and age between 20 and 40 order by age asc, entrydate asc limit 5;
   ```

   - DQL-执行顺序 

   ```mysql
   from 
   	 表名列表
   where
   	 条件列表
   group by 
   	 分组字段列表
   having 
   	 分组后条件列表
   select
   	 字段列表
   order by 
   	 排序字段列表
   limit
   	 分页参数
   ```

## 函数

### 1. 字符串函数

| 函数                     | 功能                                                      |
| ------------------------ | --------------------------------------------------------- |
| CONCAT(S1,S2,...Sn)      | 字符串拼接，将S1，S2，... Sn拼接成一个字符串              |
| LOWER(str)               | 将字符串str全部转为小写                                   |
| UPPER(str)               | 将字符串str全部转为大写                                   |
| LPAD(str,n,pad)          | 左填充，用字符串pad对str的左边进行填充，达到n个字符串长度 |
| RPAD(str,n,pad)          | 右填充，用字符串pad对str的右边进行填充，达到n个字符串长度 |
| TRIM(str)                | 去掉字符串头部和尾部的空格                                |
| SUBSTRING(str,start,len) | 返回从字符串str从start位置起的len个长度的字符串           |

### 2. 数值函数

| 函数       | 功能                               |
| ---------- | ---------------------------------- |
| CEIL(x)    | 向上取整                           |
| FLOOR(x)   | 向下取整                           |
| MOD(x,y)   | 返回x/y的模                        |
| RAND()     | 返回0~1内的随机数                  |
| ROUND(x,y) | 求参数x的四舍五入的值，保留y位小数 |

### 3. 日期函数

| 函数                               | 功能                                              |
| ---------------------------------- | ------------------------------------------------- |
| CURDATE()                          | 返回当前日期                                      |
| CURTIME()                          | 返回当前时间                                      |
| NOW()                              | 返回当前日期和时间                                |
| YEAR(date)                         | 获取指定date的年份                                |
| MONTH(date)                        | 获取指定date的月份                                |
| DAY(date)                          | 获取指定date的日期                                |
| DATE_ADD(date, INTERVAL expr type) | 返回一个日期/时间值加上一个时间间隔expr后的时间值 |
| DATEDIFF(date1, date2)             | 返回起始时间date1和结束时间date2之间的天数        |

### 4. 流程函数

| 函数                                                       | 功能                                                      |
| ---------------------------------------------------------- | --------------------------------------------------------- |
| IF(value, t, f)                                            | 如果value为true，则返回t，否则返回f                       |
| IFNULL(value1, value2)                                     | 如果value1不为空，返回value1，否则返回value2              |
| CASE WHEN [val1] THEN [res1] ... ELSE [default] END        | 如果val1为true，返回res1，... 否则返回default默认值       |
| CASE [expr] WHEN [val1] THEN [res1] ... ELSE [default] END | 如果expr的值等于val1，返回res1，... 否则返回default默认值 |

## 约束

### 概述

1. **概念**：约束是作用于表中字段上的规则，用于限制存储在表中的数据。

2. **目的**：保证数据库中数据的正确、有效性和完整性。

3. **分类**：

| 约束     | 描述                                                     | 关键字      |
| -------- | -------------------------------------------------------- | ----------- |
| 非空约束 | 限制该字段的数据不能为null                               | NOT NULL    |
| 唯一约束 | 保证该字段的所有数据都是唯一、不重复的                   | UNIQUE      |
| 主键约束 | 主键是一行数据的唯一标识，要求非空且唯一                 | PRIMARY KEY |
| 默认约束 | 保存数据时，如果未指定该字段的值，则采用默认值           | DEFAULT     |
| 检查约束 | 保证字段值满足某一个条件（8.0.16版本之后）               | CHECK       |
| 外键约束 | 用来让两张表的数据之间建立连接，保证数据的一致性和完整性 | FOREIGN KEY |

### 演示

| 字段名 | 字段含义   | 字段类型    | 约束条件                  | 约束关键字                  |
| ------ | ---------- | ----------- | ------------------------- | --------------------------- |
| id     | ID唯一标识 | int         | 主键，并且自动增长        | PRIMARY KEY, AUTO_INCREMENT |
| name   | 姓名       | varchar(10) | 不为空，并且唯一          | NOT NULL, UNIQUE            |
| age    | 年龄       | int         | 大于0，并且小于等于120    | CHECK                       |
| status | 状态       | char(1)     | 如果没有指定该值，默认为1 | DEFAULT                     |
| gender | 性别       | char(1)     | 无                        |                             |

```mysql
create table user(
	id int primary key auto_increment comment '主键',
    name varchar(10) not null unique comment '姓名',
    age int check (age > 0 && age <= 120) comment '年龄',
    status char(1) default '1' comment '状态',
    gender char(1) comment '性别'
) comment '用户表';
```

### 外键约束

- 外键用来让两张表的数据之间建立联系，从而保证数据的一致性和完整性。

![image-20250319045924349](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20250319045924349.png)

```mysql
# 添加外键
create table 表名(
	字段名 数据类型,
    ...
    [constraint][外键名称] foreign key(外键字段名) references 主表(主表列名)
);

alter table 表名 add constraint 外键名称 foreign key(外键字段名) references 主表(主表列名);

# 添加外键
alter table emp add constraint fk_emp_dept_id foreign key (dept_id) references dept(id);

# 删除外键
alter table emp drop foreign key fk_emp_dept_id;
```

| 行为        | 说明                                                         |
| ----------- | ------------------------------------------------------------ |
| NO ACTION   | 当在父表中删除/更新对应记录时，首先检查该记录是否有对应外键，如果有则不允许删除/更新。（与 RESTRICT 一致） |
| RESTRICT    | 当在父表中删除/更新对应记录时，首先检查该记录是否有对应外键，如果有则不允许删除/更新。（与 NO ACTION 一致） |
| CASCADE     | 当在父表中删除/更新对应记录时，首先检查该记录是否有对应外键，如果有，则删除/更新外键在子表中的记录。 |
| SET NULL    | 当在父表中删除对应记录时，首先检查该记录是否有对应外键，如果有则设置子表中该外键值为null（这就要求该外键允许取null）。 |
| SET DEFAULT | 父表有变更时，子表将外键列设置成一个默认的值 (Innodb 不支持) |

### 外键约束语法

```sql
ALTER TABLE 表名 ADD CONSTRAINT 外键名称 FOREIGN KEY (外键字段) REFERENCES 主表名 (主表字段名) ON UPDATE CASCADE ON DELETE CASCADE;

# 外键的删除和更新行为
alter table emp add constraint fk_emp_dept_id foreign key (dept_id) references dept(id) on update cascade on delete cascade;

alter table emp add constraint fk_emp_dept_id foreign_key (dept_id) references dept(id) on update set null on delete set null;
```

## 多表查询 

### 多表关系

​	项目开发中，在进行数据库表结构设计时，会根据业务需求及业务模块之间的关系，分析并设计表结构，由于业务之间相互关联，所以各个表之间也存在着各种联系，基本上分为三种:

##### 一对多(多对一)

- 案例：部门与员工的关系
- 关系：一个部门对应多个员工，一个员工对应一个部门
- 实现：<span style="color: darkred;">在多的一方建立外键，指向一的一方的主键</span>

![image-20250319131717394](https://raw.githubusercontent.com/LiXiaoYaoCareFree/images/refs/heads/master/docker%E5%85%A8%E8%A7%A3/img_1.png)

##### 多对多

- 案例：学生与课程的关系
- 关系：一个学生可以选修多门课程，一门课程也可以供多个学生选择
- 实现：<span style="color: darkred;">建立第三张中间表，中间表至少包含两个外键，分别关联两方的主键</span> 

![image-20250319132643399](https://raw.githubusercontent.com/LiXiaoYaoCareFree/images/refs/heads/master/docker%E5%85%A8%E8%A7%A3/img_2.png)

##### 一对一

- 案例：用户与详情的关系
- 关系：一对一关系，多用于单表拆分，将一张表的基础字段放在第一张表中，其他详情字段放在另一张表中，以提升操作效率
- 实现：<span style="color: darkred;">在任意一方加入外键，关联另外一方的主键，并且设置外键为唯一的(UNIQUE)</span> 

![](https://raw.githubusercontent.com/LiXiaoYaoCareFree/images/refs/heads/master/docker%E5%85%A8%E8%A7%A3/img_3.png)

### 多表查询分类

- 连接查询 
  - 内连接：相当于查询A、B交集部分数据
  
    ```mysql
    # 查询每一个员工的姓名，及关联的部门的名称(隐式内连接实现)
    select emp.name, dept.name from emp, dept where emp.dept_id = dept.id;
    
    # 查询每一个员工的姓名，及关联的部门的名称(显式内连接实现)
    select e.name, d.name from emp e inner join dept d on e.dept_id = d.id;
    ```
  
    
  
  - 外连接：
    - 左外连接：查询左表所有数据，以及两张表交集部分数据
    - 右外连接：查询右表所有数据，以及两张表交集部分数据 
  
    ```mysql
    # 查询emp表的所有数据，和对应的部门信息(左外连接)
    select e.*,d.name from emp e left outer join dept d on e.dept_id = d.id;
    
    # 查询dept表的所有数据，和对应的员工信息(右外连接)
    select d.*, e.* from emp e right outer join dept d on e.dept_id = d.id;
    ```
  
    
  
  - 自连接：当前表与自身的连接查询，自连接必须使用表别名
  
    ```mysql
    # 查询员工及其所属领导的名字
    select a.name, b.name from emp a, emp b where a.managerid = b.id;
    
    # 查询所有员工emp及其领导的名字emp, 如果员工没有领导，也要查询出来
    select a.name '员工', b.name '领导' from emp a left join emp b on a.managerid = b.id;
    ```
  
- 联合查询-union，union all

  - 对于union查询，就是把多次查询的结果合并起来，形成一个新的查询结果集。

    ```mysql
    SELECT 字段列表 FROM 表A ...
    UNION[ALL]
    SELECT 字段列表 FROM 表B ...;
    
    # 将薪资低于5000的员工，和年龄大于50岁的员工全部查询出来
    select * from emp where salary < 5000
    union
    select * from emp where age > 50;
    ```

    <span style="color: darkred;">对于联合查询的多张表的列数必须保持一致，字段类型也需要保持一致。</span>

- 子查询

  - 概念：SQL语句中嵌套select语句，称为<span style="color: red">嵌套查询</span>，又称<span style="color: red;">子查询</span> 

    ```mysql 
    select * from t1 where column1 = (select column1 from t2);
    ```

    <span style="color: darkred;">子查询外部的语句可以是insert / update / delete / select 的任何一个。</span>

    根据子查询的结果不同，分为：

    - 标量子查询(子查询结果为单个值)

      ```mysql
      # 查看“销售部“部门ID
      select id from dept where name = '销售部';
      
      # 根据销售部门ID, 查询员工信息
      select * from emp where dept_id = (select id from dept where name = '销售部');
      
      # 查询指定入职日期之后入职的员工信息
      select * from emp where entrydate > (select entrydate from emp where name = '东方白');
      ```

      

    - 列子查询(子查询结果为一列) 

      ```mysql
      # 1.查询“销售部”和“市场部”的所有员工信息
      -- a.查询“销售部”和“市场部”的部门ID
      select id from dept where name = '销售部' or name = '市场部';
      
      -- b.根据部门ID,查询员工信息
      select * from emp dept_id in (select id from dept where name = '销售部' or name = '市场部');
      
      # 2.查询比财务部所有人工资都高的员工信息
      -- a. 查询所有财务部人员工资
      select id from dept where name = '财务部';
      
      select salary from emp where dept_id = (select id from dept where name = '财务部');
      
      -- b. 比财务部所有人工资都高的员工信息
      select * emp where salary > all(select salary from emp where dept_id = (select id from dept where name = '财务部'));
      
      # 3.查询比研发部其中任意一人工资高的员工信息
      -- a.查询研发部所有人工资
      select salary from emp where dept_id = (select id from dept where name = '研发部');
      
      -- b.比研发部其中任意一人工资高的员工信息
      select * from emp where salary > any(select salary from emp where dept_id = (select id from dept where name = '研发部'));
      ```

    - 行子查询(子查询结果为一行)

      ```mysql
      # 查询与“张无忌”的薪资及直属领导相同的员工信息
      -- a.查询“张无忌”的薪资及直属领导
      select salary, managerid from emp where name = '张无忌';
      
      -- b.查询与“张无忌”的薪资及直属领导相同的员工信息
      select * from emp where (salary, managerid) = (select salary, managerid from emp where name = '张无忌');
      ```

    - 表子查询(多行多列)

      ```mysql
      # 1. 查询与”常遇春“，“宋远桥”的职位和薪资相同的员工信息
      -- a.查询”常遇春“，“宋远桥”的职位和薪资
      select job, salary from emp where name = '常遇春' or name = '宋远桥';
      
      -- b.查询与”常遇春“，“宋远桥”的职位和薪资相同的员工信息
      select * from emp where (job,salary) in (select job, salary from emp where name = '常遇春' or name = '宋远桥');
      
      # 2.查询入职日期是‘2006-01-01’之后的员工信息，及其部门信息
      -- a.入职日期是‘2006-01-01’的员工信息
      select * from emp where entrydate > '2006-01-01';
      
      -- b.查询这部分员工，对应的部门信息
      select e.*, d.* from (select * from emp where entrydate > '2006-01-01') e left join dept d on e.dept_id = d.id;
      ```

    根据子查询位置，分为：where之后，from之后，select之后。

### 多表查询案例

![](https://raw.githubusercontent.com/LiXiaoYaoCareFree/images/refs/heads/master/docker%E5%85%A8%E8%A7%A3/img_4.png)

![](https://raw.githubusercontent.com/LiXiaoYaoCareFree/images/refs/heads/master/docker%E5%85%A8%E8%A7%A3/img_5.png)

```mysql
-- 1.查询员工的姓名、年龄、职位、部门信息(隐式内连接)
-- 表: emp, dept
-- 连接条件: emp.dept_id = dept.id
select e.name, e.age, e.job, d.name from emp e, dept d where e.dept_id = d.id;


-- 2.查询年龄小于30岁的员工的姓名、年龄、职位、部门信息(显示内连接)
-- 表: emp, dept
-- 连接条件: emp.dept_id = dept.id
select e.name, e.age, e.job, d.name from emp e inner join dept d on e.dept_id = d.id where e.age < 30;


-- 3.查询拥有员工的部门ID、部门名称
-- 表: emp, dept
-- 连接条件: emp.dept_id = dept.id
select distinct d.id, d.name from emp e, dept d where e.dept_id = d.id;


-- 4.查询所有年龄大于40岁的员工，及其归属的部门名称；如果员工没有分配部门，也需要展示出来
-- 表: emp, dept
-- 连接条件: emp.dept_id = dept.id
select e.*, d.name from emp e left join dept d on e.dept_id = d.id where e.age > 40;


-- 5.查询所有员工的工资等级
-- 表: emp, dept
-- 连接条件: emp.salary >= salgrade.losal and emp.salary <= salgrade.hisal
select e.*, s.grade from emp e, salgrade s where e.salary >= s.losal and e.salary <= s.hisal;


-- 6.查询“研发部”所有员工的信息及工资等级
-- 表: emp, dept， salgrade
-- 连接条件: emp.salary between salgrade.losal and salgrade.hisal, emp.dept_id = dept.id
-- 查询条件: dept.name = '研发部'

select e.*, s.grade from emp e, dept d, salgrade s where e.dept_id = d.id and (e.salary between s.losal and s.hisal) and d.name = '研发部';


-- 7.查询‘研发部’员工的平均工资
-- 表: emp, dept
-- 连接条件: emp.dept_id = dept.id
select avg(e.salary) from emp e, dept d where e.dept_id = d.id and d.name = '研发部';

-- 8.查询工资比“常遇春”高的员工信息
-- a.查询“常遇春”的薪资
select salary from emp where name = '常遇春';

-- b.查询比他工资高的员工
select * from emp where salary > (select salary from emp where name = '常遇春');

-- 9.查询比平均薪资高的员工信息
-- a.查询员工的平均薪资
select avg(salary) from emp;

-- b.查询比平均薪资高的员工信息
select * from emp where salary > (select avg(salary) from emp);


-- 10.查询低于本部门平均工资的员工信息
-- a.查询指定部门的平均薪资
select avg(e1.salary) from emp e1 where e1.dept_id = 1;
select avg(e1.salary) from emp e1 where e1.dept_id = 2;

-- b.查询低于本部门平均工资的员工信息
select * from emp e2 where e2.salary < (select avg(e1.salary) from emp e1 where e1.dept_id = e2.dept_id);


-- 11.查询所有部门的信息，并统计部门的员工数量
select d.id, d.name, (select count(*) from emp e where e.dept_id = d.id) '人数' from dept d;

select count(*) from emp where dept_id = 1;


-- 12.查询所有学生的选课情况，展示出学生名称，学号，课程名称
-- 表: student, course, student_course
-- 连接条件: student.id = student_course.studentid, course.id = student_course.courseid

select s.name, s.no, c.name from student s, student_course sc, course c where s.id = sc.studentid and sc.courseid = c.id;
```



## 事务

<span style="color: darkred;">事务</span>是一组操作的集合，它是一个不可分割的工作单位，事务会把所有的操作作为一个整体一起向系统提交或撤销操作请求，即这些操作要么同时成功，要么同时失败。

<span style="color: darkred;">默认MySQL的事务是自动提交的，也就是说，当执行一条DML语句时，MySQL会立即隐式的提交事务。</span>

#### 事务操作

- 查看/设置事务提交方式

  ```mysql
  select @@autocommit;
  set @@autocommit=0;
  ```

- 或者开启事务

  ```mysql
  start transaction 或 begin;
  ```

  

- 提交事务

  ```mysql
  commit;
  ```

- 回滚事务

  ```mysql
  rollback;
  ```

  事务四大特性

  - **原子性（Atomicity）：** 事务是不可分割的最小操作单元，要么全部成功，要么全部失败。
  - **一致性（Consistency）：** 事务完成时，必须使所有的数据都保持一致状态。
  - **隔离性（Isolation）：** 数据库系统提供的隔离机制，保证事务在不受外部并发操作影响的独立环境下运行。
  - **持久性（Durability）：** 事务一旦提交或回滚，它对数据库中的数据的改变就是永久的。

#### 事务隔离级别

| 隔离级别               | 脏读 | 不可重复读 | 幻读 |
| ---------------------- | ---- | ---------- | ---- |
| Read uncommitted       | √    | √          | √    |
| Read committed         | ×    | √          | √    |
| Repeatable Read (默认) | ×    | ×          | √    |
| Serializable           | ×    | ×          | ×    |

---

- 查看事务隔离级别  
  ```sql
  SELECT @@TRANSACTION_ISOLATION;

- 设置事务隔离级别

  ```mysql
  SET {SESSION | GLOBAL} TRANSACTION ISOLATION LEVEL {READ UNCOMMITTED | READ COMMITTED | REPEATABLE READ | SERIALIZABLE};
  ```

  