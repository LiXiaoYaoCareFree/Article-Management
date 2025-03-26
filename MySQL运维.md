# MySQL运维

## 1.日志

### 1.1错误日志

错误日志是 MySQL 中最重要的日志之一，它记录了当 `mysqld` 启动和停止时，以及服务器在运行过程中发生任何严重错误时的相关信息。当数据库出现任何故障导致无法正常使用时，建议首先查看此日志。

该日志是默认开启的，默认存放目录为 `/var/log/`，默认的日志文件名为 `mysqld.log`。查看日志位置：

```sql
SHOW VARIABLES LIKE '%log_error%';
```

![image-20250320145102425](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20250320145102425.png)

### 1.2二进制日志

二进制日志（BINLOG）记录了所有的 DDL（数据定义语言）语句和 DML（数据操纵语言）语句，但不包括数据查询（SELECT、SHOW）语句。

作用：(1)灾难时的数据恢复 (2) MySQL的主从复制。在MySQL8版本中，默认二进制日志是开启的，涉及到的参数如下：

```mysql
show variables like '%log_bin%';
```

##### 二进制日志查看

由于日志是以二进制方式存储的，不能直接读取，需要通过二进制日志查询工具 `mysqlbinlog` 来查看，具体语法：

```bash
mysqlbinlog [参数选项] logfilename
```

**参数选项：**

- `-d`：指定数据库名称，只列出指定的数据库相关操作。
- `-o`：忽略掉日志中的前 n 行命令。
- `-v`：将行事件（数据变更）重构为 SQL 语句。
- `-w`：将行事件（数据变更）重构为 SQL 语句，并输出注释信息。

##### 二进制日志删除

对于比较繁忙的业务系统，每天生成的 binlog 数据巨大，如果长时间不清除，将会占用大量磁盘空间。可以通过以下几种方式清理日志：

| 指令                                               | 含义                                                         |
| -------------------------------------------------- | ------------------------------------------------------------ |
| `reset master`                                     | 删除全部 binlog 日志，删除之后，日志编号将从 `binlog.000001` 重新开始。 |
| `purge master logs to 'binlog.******'`             | 删除 `******` 编号之前的所有日志。                           |
| `purge master logs before 'yyyy-mm-dd hh24:mi:ss'` | 删除在 `yyyy-mm-dd hh24:mi:ss` 之前产生的所有日志。          |

也可以在mysql的配置文件中配置二进制日志的过期时间，设置了之后，二进制日志过期会自动删除。

```mysql
show variables like '%binlog_expire_logs_seconds%';
```

### 1.3查询日志

查询日志中记录了客户端的所有操作语句，而二进制日志不包含查询数据的 SQL 语句。默认情况下，查询日志是未开启的。如果需要开启查询日志，可以设置以下配置：

```mysql
show variables like '%general%';
```

修改MySQL的配置文件 /etc/my.cnf文件，添加如下内容:

```mysql
# 该选项用来开启查询日志，可选值：0或者1；0代表关闭，1代表开启
general_log = 1;
# 设置日志的文件名，如果没有指定，默认的文件名为host_name.log
general_log_file=mysql_query.log
```

### 1.4慢查询日志

慢查询日志记录了所有执行时间超过参数 `long_query_time` 设置值并且扫描记录数不小于 `min_examined_row_limit` 的所有 SQL 语句的日志，默认未开启。`long_query_time` 默认为 10 秒，最小为 0，精度可以到微秒。

```bash
# 慢查询日志
slow_query_log=1
# 设置慢查询时间阈值
long_query_time=2
```

默认情况下，不会记录管理语句，也不会记录不使用索引进行查找的查询。可以使用 `log_slow_admin_statements` 和 `log_queries_not_using_indexes` 来更改此行为，如下所述。

```mysql
# 记录执行较慢的管理语句
log_slow_admin_statements=1
# 记录执行较慢的未使用索引的查询
log_queries_not_using_indexes=1
```

## 2.主从复制

![image-20250320152548258](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20250320152548258.png)

### 概述

主从复制是指将主数据库的 DDL 和 DML 操作通过二进制日志传到从库服务器中，然后在从库上对这些日志重新执行（也叫重做），从而使得从库和主库的数据保持同步。

MySQL 支持一台主库同时向多台从库进行复制，从库同时也可以作为其他从服务器的主库，实现链状复制。

**MySQL 复制的主要优点包含以下三个方面：**

1. **高可用性**：主库出现问题，可以快速切换到从库提供服务。
2. **读写分离**：实现读写分离，降低主库的访问压力。
3. **备份优化**：可以在从库中执行备份，以避免备份期间影响主库服务。

### 原理

![image-20250320152944759](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20250320152944759.png)

从上图来看，复制分成三步：

1. **主库记录日志**：Master 主库在事务提交时，会把数据变更记录在二进制日志文件 Binlog 中。

2. **从库读取日志**：从库读取主库的二进制日志文件 Binlog，写入到从库的中继日志 Relay Log。

3. **从库重做日志**：Slave 重做中继日志中的事件，将改变反映到它自己的数据。

### 主从配置

先写docker-compose.yml配置文件

```YAML
version: '3.2'
services:
  mysql-master:
    image: "mysql:5.7"
    container_name: mysql-master
    restart: always
    privileged: true
    environment:
      MYSQL_ROOT_PASSWORD: root  #主库root用户的密码
    command:
      --default-authentication-plugin=mysql_native_password
      --character-set-server=utf8mb4
      --collation-server=utf8mb4_general_ci
      --explicit_defaults_for_timestamp=true
      --lower_case_table_names=1
      --max_allowed_packet=128M;
    ports:
      - 3306:3306  #映射宿主机端口：容器端口
    volumes:  #宿主机配置目录或文件挂载到容器
      - ./master/my.cnf:/etc/mysql/my.cnf
      - ./master/mysql-files:/var/lib/mysql-files
      - ./master/data:/var/lib/mysql
      - ./master/log:/var/log/
    networks:
      blogx_network:
        ipv4_address: 10.2.0.2
  mysql-slave:
    image: "mysql:5.7"
    container_name: mysql-slave
    restart: always
    privileged: true
    environment:
      MYSQL_ROOT_PASSWORD: root  #从库root用户的密码
    command:
      --default-authentication-plugin=mysql_native_password
      --character-set-server=utf8mb4
      --collation-server=utf8mb4_general_ci
      --explicit_defaults_for_timestamp=true
      --lower_case_table_names=1
      --max_allowed_packet=128M;
    ports:
      - 3307:3306  #映射宿主机端口：容器端口
    volumes:  #宿主机配置目录或文件挂载到容器
      - ./slave/my.cnf:/etc/mysql/my.cnf
      - ./slave/mysql-files:/var/lib/mysql-files
      - ./slave/data:/var/lib/mysql
      - ./slave/log:/var/log/
    networks:
      blogx_network:
        ipv4_address: 10.2.0.3
networks:  #定义容器连接网络
  blogx_network:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: 10.2.0.0/24
```



配置主库文件master/my.cnf

```Ini
[client]
port        = 3306
socket      = /var/run/mysqld/mysqld.sock

[mysqld_safe]
pid-file    = /var/run/mysqld/mysqld.pid
socket      = /var/run/mysqld/mysqld.sock
nice        = 0

[mysqld]
user        = mysql
pid-file    = /var/run/mysqld/mysqld.pid
socket      = /var/run/mysqld/mysqld.sock
port        = 3306
basedir     = /usr
datadir     = /var/lib/mysql
tmpdir      = /tmp
secure-file-priv=/var/lib/mysql-files
lc-messages-dir = /usr/share/mysql
explicit_defaults_for_timestamp

log-bin = mysql-bin
server-id = 1
```



配置从库文件slave/my.cnf

```Ini
[client]
port        = 3306
socket      = /var/run/mysqld/mysqld.sock

[mysqld_safe]
pid-file    = /var/run/mysqld/mysqld.pid
socket      = /var/run/mysqld/mysqld.sock
nice        = 0

[mysqld]
user        = mysql
pid-file    = /var/run/mysqld/mysqld.pid
socket      = /var/run/mysqld/mysqld.sock
port        = 3306
basedir     = /usr
datadir     = /var/lib/mysql
tmpdir      = /tmp
secure-file-priv=/var/lib/mysql-files
lc-messages-dir = /usr/share/mysql
explicit_defaults_for_timestamp

log-bin = mysql-bin
server-id = 2

```

然后使用docker compose up -d 跑起来

进主节点的mysql

```Bash
docker exec -it mysql-master bash
mysql -uroot -proot

GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%' identified by '123456';
flush privileges;
show master status \G;
```

进从节点的mysql

```SQL
docker exec -it mysql-slave bash
mysql -uroot -proot

CHANGE MASTER TO MASTER_HOST='10.2.0.2', MASTER_USER='repl', MASTER_PASSWORD='123456', MASTER_LOG_FILE='mysql-bin.000003', MASTER_LOG_POS=589;
start slave;
show slave status \G;
```

MASTER_HOST是主节点的ip，

MASTER_LOG_FILE和MASTER_LOG_POS是主节点查询show master status \G;之后填上去的

![img](http://www.xiaoyaocloud.com/uploads/images001/912bb5755a94a78fddef6d60f19703b0.png)

这两个一定得是Yes才行

## 3.分库分表

#### 单数据库性能瓶颈

随着互联网及移动互联网的发展，应用系统的数据量也是成指数式增长，若采用单数据库进行数据存储，存在以下性能瓶颈：

1. **IO 瓶颈**：热点数据太多，数据库缓存不足，产生大量磁盘 IO，效率较低。请求数据太多，带宽不够，网络 IO 瓶颈。

2. **CPU 瓶颈**：排序、分组、连接查询、聚合统计等 SQL 会耗费大量的 CPU 资源，请求数太多，CPU 出现瓶颈。

<span style="color:darkred;">分库分表的中心思想都是将数据分散存储，使得单一数据库/表的数据量变小来缓解单一数据库的性能问题，从而达到提升数据库性能的目的。</span>

#### 介绍

![image-20250320165145297](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20250320165145297.png)

![image-20250320183754272](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20250320183754272.png)

<span style="color: red;">垂直分库</span>：以表为依据，根据业务将不同表拆分到不同库中。

特点：

1. 每个库的表结构都不一样。
2. 每个库的数据也不一样。
3. 所有库的并集是全量数据。

![image-20250320184618015](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20250320184618015.png)

<span style="color: red;">垂直分表</span>：以字段为依据，根据字段属性将不同字段拆分到不同表中。

特点：

1. 每个表的结构都不一样。
2. 每个表的数据也不一样，一般通过一列（主键/外键）关联。
3. 所有表的并集是全量数据。

![image-20250320185031951](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20250320185031951.png)

<span style="color: red;">水平分库</span>：以字段为依据，按照一定策略，将一个库的数据拆分到多个库中。

特点：

 1.每个库的表结构都一样。

 2.每个库的数据都不一样。

3.所有库的并集是全量数据。

![image-20250320185409735](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20250320185409735.png)

<span style="color: red;">水平分表</span>：以字段为依据，按照一定策略，将一个表的数据拆分到多个表中。

特点：

 1.每个表的表结构都不一样。

 2.每个表的数据都不一样。‘

 3.所有表的并集是全量数据。

#### docker安装Mycat和MySQL水平分库分表

因为Mycat和MySQL都是使用docker安装的，容器和容器之间是相互隔离的，这时候需要用到docker网络帮助我们进行两个容器之间的通信！

```bash
docker network create -d bridge --ip-range=192.168.1.0/24 --gateway=192.168.1.1 --subnet=192.168.1.0/24 bridge2
```

- **`docker network create`**: 创建一个新的 Docker 网络。
- **`-d bridge`**: 指定网络驱动为 `bridge`（桥接模式）。
- **`--ip-range=192.168.1.0/24`**: 设置 IP 地址范围为 `192.168.1.0/24`，即从 `192.168.1.1` 到 `192.168.1.254`。
- **`--gateway=192.168.1.1`**: 设置网关为 `192.168.1.1`。
- **`--subnet=192.168.1.0/24`**: 设置子网为 `192.168.1.0/24`。
- **`bridge2`**: 新创建的网络名称。

##### 安装Mycat

先下载Mycat压缩包[Mycat-server-1.6.7.5-release-20200422133810-linux.tar.gz](https://github.com/MyCATApache/Mycat-Server/releases/download/Mycat-server-1675-release/Mycat-server-1.6.7.5-release-20200422133810-linux.tar.gz)到/usr/local/目录下，再解压

```bash
tar -zxvf Mycat-server-1.6.7.5-release-20200422133810-linux.tar.gz
```

会生成mycat文件夹

在/usr/local/目录下准备Dockerfile

```bash
touch Dockerfile
```

Dockerfile内容如下：

```dockerfile
# 因为mycat运行需要jdk的支持，所以基于jdk的镜像来构建mycat的镜像。
FROM openjdk:8-jdk-stretch

# 把本机当前目录下面的mycat安装包，拷贝到容器中，并且解压到容器的指定目录。
ADD ./Mycat-server-1.6.7.5-release-20200422133810-linux.tar.gz /usr/local

# 声明卷目录，该目录可以被挂载到宿主机本地某个目录下。
VOLUME /usr/local/mycat/conf
VOLUME /usr/local/mycat/logs

# 暴露mycat的连接端口和管理端口。
EXPOSE 8066 9066

# 启动mycat服务。
CMD ["/usr/local/mycat/bin/mycat", "console"]
```

构建镜像

docker build命令构建镜像

```bash
docker build -t mycat:v1.6.7.5 .
```

先创建/usr/local/mycat/logs/mycat.pid文件

```bash
cd /usr/local/mycat

mkdir logs

chmod 0777 logs

touch mycat.pid

# 控制台直接启动的命令
/usr/local/mycat/bin/mycat console
```

后台启动mycat的命令如下：

- 启动的命令就是：`/usr/local/mycat/bin/mycat start`
- 停止的命令就是：`/usr/local/mycat/bin/mycat stop`
- 重启的命令就是：`/usr/local/mycat/bin/mycat restart`
- 查看mycat的运行状态的命令就是：`/usr/local/mycat/bin/mycat status`

##### MyCat配置文件介绍
mycat的配置文件有很多，我们经常使用到的是：server.xml、schema.xml、rule.xml等配置文件。

- **server.xml**
  定义mycat系统的配置和mycat的登录用信息。

  有两个标签

  - system：配置mycat系统的相关属性，包括：管理端口、连接端口、连接池的配置、支出的事务配置等。

  - user：配置mycat的用户相关的信息，包括登录mycat的时候使用的用户名、密码、权限信息等。
    配置一个用户mycat，密码也是mycat。这个用户不是正在的MySQL数据库中的登录用户，这个是连接到mycat服务的时候使用的用户和密码。

    在前端应用在连接到mycat的时候，配置的数据库连接信息就是配置的这里的账号和密码，而不是配置真正的MySQL数据库的账号和密码。

    ```html
    <user name="mycat">
    		<property name="password">mycat</property>
    		<property name="schemas">testdb</property><!-- 逻辑库名称，并不是真正的MySQL中的schema名称。这里定义为什么，在schema.xml文件中的schema标签就需要与这里对应上。 -->
    		<!-- <property name="readOnly">true</property> --><!-- 只读用户的配置 -->
    		<property name="benchmark">1000</property><!-- 限制该用户的连接数，如果设置为0表示不限制。默认不配置，也是不限制 -->
    	</user>
    ```

- **schema.xml**

定义数据库、连接信息

schema配置文件中主要包括逻辑库、表和表分片的规则、分片节点、mycat后面代理的真实的数据库的连接地址、账号和密码相关信息

简单示例:

```html
	<schema name="testdb" checkSQLschema="true" sqlMaxLimit="100" dataNode="dn1" />
    
	<dataNode name="dn1" dataHost="localhost1" database="testdb" />
    
<dataHost name="localhost1" maxCon="1000" minCon="1" balance="3" writeType="1" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">
		<heartbeat>select user()</heartbeat>
		
		<!-- 首选的写节点，如果首选的写节点宕机，那么该节点中的读节点也会认为宕机，处于不可用状态。自动切换为备用的写和备用的读。 -->
		<!-- can have multi write hosts -->
		<writeHost host="hostM1" url="localhost:3301" user="root" password="root">
			<!-- can have multi read hosts -->
			<readHost host="hostS1" url="localhost:3303" user="root" password="root" />
		</writeHost>
		
		<!-- 备选的写节点 -->
		<!-- can have multi write hosts -->
		<writeHost host="hostM2" url="localhost:3302" user="root" password="root">
			<!-- can have multi read hosts -->
			<readHost host="hostS2" url="localhost:3304" user="root" password="root" />
		</writeHost>

	</dataHost>
```

- role.xml

  水平分表的时候会用到该配置文件，涉及到把一个表中的数据如何水平拆分到各个数据库实例中。这里暂时不做水平拆分的验证，先不配置。

## 4.读写分离

读写分离是一种数据库优化技术，通过将对数据库的读取和写操作分开，分配到不同的数据库服务器上。主数据库负责处理写操作，而从数据库则负责处理读操作。这种方式可以有效减轻单台数据库服务器的压力。

### 实现方式

通过使用MyCat，可以轻松实现读写分离功能。MyCat不仅支持MySQL，还支持Oracle和SQL Server等多种数据库系统。

优点

- **减轻主数据库负载**：将读操作分散到从数据库，减少主数据库的压力。
- **提高系统性能**：通过并行处理读操作，提升整体系统的响应速度。
- **扩展性强**：可以根据需要增加从数据库的数量，以应对更高的读取需求。

### 使用场景

- 高并发的读操作场景。
- 需要频繁读取数据的应用程序。
- 数据库写操作相对较少，但读操作较多的系统。

![image-20250321113822417](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20250321113822417.png)