ProxySQL概述
ProxySQL 是一种基于 GNU 通用公共许可证 (GPL) ，并位于 MySQL 数据库与数据库客户端之间的数据库访问代理中间件。
ProxySQL的性能非常好，功能也很多，几乎能够满足中间件所需的绝大多数的功能，主要包括：
	跨多个数据库的智能负载均衡；
	连接池特性，在高并发连接下提高数据库访问性能；
	可以对查询语句进行mirror，实现日志审计功能；
	可以基于用户、基于 Schema 和基于 SQL 语句的规则对 SQL 语句进行路由转发，实现读/写分离和简单的数据库分片功能；
	Firewall白名单功能可以实现数据库防火墙，用白名单规则定义允许执行的查询语句；
 
本文主要介绍ProxySQL on AWS的整体架构和测试环境的基本配置，各个功能特性将在后续blog中陆续展开。

ProxySQL on AWS架构图
以下为本文的部署架构图：
见ProxySQL-on-AWS.png
 
因为是测试环境，所以只配置了一个ProxySQL实例，应用的访问也采用在ProxySQL上安装MySQL客户端进行模拟操作。如果部署生产环境，需要加上ProxySQL的cluster配置，然后配合ELB负载均衡服务器，对应用提供高可用和负载均衡。

RDS MySQL环境是标准的三个实例部署：一个Master实例，两个Replica只读副本。

先决条件
根据以上的架构图，搭建的环境需要以下：
1.	在AWS创建一个运行Amazon Linux2的实例，用于安装ProxySQL 2.0.10。步骤可以参考：https://docs.amazonaws.cn/AWSEC2/latest/UserGuide/EC2_GetStarted.html#ec2-launch-instance
创建完成后，记录下实例的公网IP，并保存好密钥文件。

2.	在EC2实例所在的VPC，创建一个AWS RDS MySQL的Master实例和两个replica实例，版本为5.7.22。对应的安全组需要配置ProxySQL实例所在网段，允许3306端口访问策略。具体操作步骤可以参考：https://docs.amazonaws.cn/AmazonRDS/latest/UserGuide/CHAP_GettingStarted.CreatingConnecting.MySQL.html

 
其中mytest57为Master实例，可以进行读写操作；mytest-r1和mytest-r2是replica，作为只读副本，可以进行读操作。
记录三个数据库实例的访问地址：
mytest57.cs8yfqtavsti.rds.cn-northwest-1.amazonaws.com.cn
mytest57-r1.cs8yfqtavsti.rds.cn-northwest-1.amazonaws.com.cn
mytest57-r2.cs8yfqtavsti.rds.cn-northwest-1.amazonaws.com.cn

记录下创建数据库实例时的用户名和密码：user01/Welcome1。
记录下为user01创建的数据库schema：test01。

过程列表
主要的步骤分为以下两个部分：
	在EC2实例下载并安装ProxySQL 2.0.10版本
	ProxySQL的基本配置

下载安装ProxySQL 2.0.10
EC2实例的操作系统为Amazon Linux 2，步骤如下：
1.	用保存的密钥文件登陆实例
$ssh -i mykey.pem ec2-user@instance-ip

2.	添加repo配置文件
$sudo vi /etc/yum.repos.d/ProxySQL.repo

[ProxySQL_repo]
name= ProxySQL YUM repository
baseurl=https://repo.ProxySQL.com/ProxySQL/ProxySQL-2.0.x/centos/latest
gpgcheck=1
gpgkey=https://repo.ProxySQL.com/ProxySQL/repo_pub_key

3.	安装和启动ProxySQL服务
$sudo yum install ProxySQL
$sudo systemctl start ProxySQL
$sudo systemctl enable ProxySQL

查看运行状态
$sudo systemctl status ProxySQL

正常启动后，服务将处于running状态。
● ProxySQL.service - High Performance Advanced Proxy for MySQL
   Loaded: loaded (/etc/systemd/system/ProxySQL.service; enabled; vendor preset: disabled)
   Active: active (running) since 五 2020-04-10 03:04:38 UTC; 3h 54min ago
  Process: 1499 ExecStart=/usr/bin/ProxySQL -c /etc/ProxySQL.cnf (code=exited, status=0/SUCCESS)
 Main PID: 1501 (ProxySQL)
   CGroup: /system.slice/ProxySQL.service
           ├─1501 /usr/bin/ProxySQL -c /etc/ProxySQL.cnf
           └─1502 /usr/bin/ProxySQL -c /etc/ProxySQL.cnf

4月 10 03:04:38 ip-172-31-32-82.cn-northwest-1.compute.internal systemd[1]: Starting High Performance Advanced Proxy for MySQL...
4月 10 03:04:38 ip-172-31-32-82.cn-northwest-1.compute.internal ProxySQL[1499]: 2020-04-10 03:04:38 [INFO] Using config file /etc/ProxySQL.cnf
4月 10 03:04:38 ip-172-31-32-82.cn-northwest-1.compute.internal ProxySQL[1499]: 2020-04-10 03:04:38 [INFO] Using OpenSSL version: OpenSSL 1.1.0h  27 Mar 2018
4月 10 03:04:38 ip-172-31-32-82.cn-northwest-1.compute.internal ProxySQL[1499]: 2020-04-10 03:04:38 [INFO] SSL keys/certificates found in datadir (/var/lib/ProxySQL): loading them.
4月 10 03:04:38 ip-172-31-32-82.cn-northwest-1.compute.internal systemd[1]: Started High Performance Advanced Proxy for MySQL.

4.	安装MySQL
由于ProxySQL要用mysql客户端进行管理，同时后端保护的数据库也需要进行一些测试的操作，所以在EC2实例上安装mysql。
$sudo yum install mysql

5.	登陆ProxySQL管理界面
管理端口为6032，数据库连接的端口为6033。
使用默认管理员和密码都为admin登陆。
$mysql -u admin -padmin -h 127.0.0.1 -P6032 --prompt='Admin> '
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MySQL connection id is 33
Server version: 5.7.22 (ProxySQL Admin Module)
Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
Admin> show databases;
+-----+---------------+-------------------------------------+
| seq | name          | file                                |
+-----+---------------+-------------------------------------+
| 0   | main          |                                     |
| 2   | disk          | /var/lib/ProxySQL/ProxySQL.db       |
| 3   | stats         |                                     |
| 4   | monitor       |                                     |
| 5   | stats_history | /var/lib/ProxySQL/ProxySQL_stats.db |
+-----+---------------+-------------------------------------+
5 rows in set (0.00 sec)
可以看到ProxySQL的默认5个数据库。

6.	用MySQL客户端直接访问RDS MySQL实例
为确认实例可以正常访问RDS MySQL实例，使用记录用户名密码和RDS Master实例的访问地址，登陆Master实例进行查看
$mysql -u user01 -pWelcome1 -h mytest57.cs8yfqtavsti.rds.cn-northwest-1.amazonaws.com.cn --prompt='RDS-Direct> '

Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MySQL connection id is 8418
Server version: 5.7.22-log Source distribution
Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
RDS-Direct> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| innodb             |
| mysql              |
| performance_schema |
| sys                |
| test01             |
+--------------------+
6 rows in set (0.00 sec)

创建ProxySQL用于监控后端数据库的monitor用户
RDS-Direct>CREATE USER 'monitor'@'%' IDENTIFIED  by 'monitor';
RDS-Direct>GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, RELOAD, PROCESS, REFERENCES, INDEX, ALTER, SHOW DATABASES, CREATE TEMPORARY TABLES, LOCK TABLES, EXECUTE, REPLICATION SLAVE, REPLICATION CLIENT, CREATE VIEW, SHOW VIEW, CREATE ROUTINE, ALTER ROUTINE, CREATE USER, EVENT, TRIGGER ON *.* TO 'monitor'@'%' WITH GRANT OPTION
RDS-Direct>FLUSH PRIVILEGES;

可以用monitor用户进行测试，登陆RDS实例。
$mysql -u monitor -pmonitor -h mytest57.cs8yfqtavsti.rds.cn-northwest-1.amazonaws.com.cn --prompt='RDS-Direct> '
RDS-Direct>

注意：之后的RDS MySQL数据库操作都将通过访问ProxySQL的6033端口进行操作。
Admin> 的提示符表示在ProxySQL的管理界面进行操作
登陆命令为：$mysql -u admin -padmin -h 127.0.0.1 -P6032 --prompt='Admin> '

RDS-Direct>的提示符表示直接访问RDS MySQL数据库进行操作
登陆命令为：$mysql -u user01 -pWelcome1 -h mytest57.cs8yfqtavsti.rds.cn-northwest-1.amazonaws.com.cn --prompt='RDS-Direct> '


RDS>的提示符表示通过ProxySQL代理访问RDS MySQL数据库进行操作
登陆命令为：$mysql -u user01 -pWelcome1 -h 127.0.0.1 -P6033 --prompt='RDS> '
ProxySQL的基本配置
为使用ProxySQL可以作为代理客户端连接，需要进行基本的配置，主要是加入后端的数据库服务器地址，配置主机组、数据库用户等，保证可以接受客户端连接，通过ProxSQL正常访问数据库。

1.	在ProxySQL配置加入RDS数据库三个实例的访问地址，作为后端的数据库服务器

mysql_servers 表中包括将由 ProxySQL 代理的所有后端数据库服务器；mysql_replication_hostgroups 表包含自己识别和划分只读终端节点的配置。
$mysql -u admin -padmin -h 127.0.0.1 -P6032  --prompt='Admin> '
Admin> delete from mysql_servers where hostgroup_id in (10,20);	
Admin> delete from mysql_replication_hostgroups where writer_hostgroup=10;	
Admin> INSERT INTO mysql_servers (hostname,hostgroup_id,port,weight,max_connections) VALUES ('mytest57.cs8yfqtavsti.rds.cn-northwest-1.amazonaws.com.cn',10,3306,1000,2000);	
Admin> INSERT INTO mysql_servers (hostname,hostgroup_id,port,weight,max_connections) VALUES ('mytest57-r1.cs8yfqtavsti.rds.cn-northwest-1.amazonaws.com.cn',20,3306,1000,2000);	
Admin> INSERT INTO mysql_servers (hostname,hostgroup_id,port,weight,max_connections) VALUES ('mytest57-r2.cs8yfqtavsti.rds.cn-northwest-1.amazonaws.com.cn',20,3306,1000,2000);	
Admin> INSERT INTO mysql_replication_hostgroups (writer_hostgroup,reader_hostgroup,comment,check_type) VALUES (10,20,'RDS-MySQL','read_only');	
Admin> LOAD MYSQL SERVERS TO RUNTIME;	
Admin> SAVE MYSQL SERVERS TO DISK;	

注：以上mysql_replication_hostgroups 的配置，是对于RDS MySQL的，如果后端数据库是Amazon Aurora ，则check_typ字段需要改为“innodb_read_only”来进行配置。	

2.	配置ProxySQL的后端用户和查询规则
在ProxySQL 配置中定义后端用户user01，其中test01是为user01创建的默认schema。
Admin> delete from mysql_users where username='user01';	
Admin> insert into mysql_users (username,password,active,default_hostgroup,default_schema,transaction_persistent) values ('user01','Welcome1',1,10,'test01',1);	
Admin> LOAD MYSQL USERS TO RUNTIME;	
Admin> SAVE MYSQL USERS TO DISK;	
	
以下的查询规则配置完成后，可以实现简单的读写分离。
Admin> delete from mysql_query_rules where rule_id in (50,51);	
Admin> INSERT INTO mysql_query_rules (rule_id,active,match_digest,destination_hostgroup,apply) VALUES (50,1,'^SELECT.*FOR UPDATE$',10,1), (51,1,'^SELECT',20,1);	
Admin> LOAD MYSQL QUERY RULES TO RUNTIME; 
Admin> SAVE MYSQL QUERY RULES TO DISK;	
	
3.	配置后端的监控用户monitor，配置监控间隔和后端数据库版本

Admin> UPDATE global_variables SET variable_value='monitor' WHERE variable_name='mysql-monitor_username';	
Admin> UPDATE global_variables SET variable_value='monitor' WHERE variable_name='mysql-monitor_password';	
Admin> UPDATE global_variables SET variable_value='2000' WHERE variable_name IN ('mysql-monitor_connect_interval','mysql-monitor_ping_interval','mysql-monitor_read_only_interval');
Admin> UPDATE global_variables SET variable_value='5.7.22' WHERE variable_name='mysql-server_version';	
Admin> LOAD MYSQL VARIABLES TO RUNTIME;	
Admin> SAVE MYSQL VARIABLES TO DISK;	

查看ProxySQL监控状态，确认后端数据库服务器是否配置成功
Admin> SELECT * FROM monitor.mysql_server_connect_log ORDER BY time_start_us DESC LIMIT 10;
 

Admin> SELECT * FROM monitor.mysql_server_ping_log ORDER BY time_start_us DESC LIMIT 10;
 

可以看到成功的监控到后端服务器的连接和ping的时间。如果配置有问题，会出现timeout的信息。

4.	使用MySQL客户端，通过ProxySQL代理访问后端RDS数据库
$mysql -u user01 -pWelcome1 -h 127.0.0.1 -P6033 -e "SELECT 1"
+---+
| 1 |
+---+
| 1 |
+---+
$mysql -u user01 -pWelcome1 -h 127.0.0.1 -P6033 -e "SELECT @@port"
+--------+
| @@port |
+--------+
|   3306 |
+--------+

通过显示的结果，可以看的MySQL客户端已经可以正常的通过ProxySQL代理，对后端RDS MySQL数据库进行各类查询操作。

小结
至此，ProxySQL on AWS的测试环境已经搭建好，可以开始进行各项功能测试了。

本文是对RDS MySQL数据库环境进行配置和测试，如果后端数据库是用AWS Aurora或在EC2自建MySQL，配置稍微有所不同，具体可参见ProxySQL文档：https://github.com/sysown/proxysql/wiki。

ProxySQL默认使用4个thread接受客户端的连接，考虑到资源使用情况，对单个节点的实例大小和配置也要根据实际情况进行调整。

如果将ProxySQL部署在生产环境，可以用ProxySQL的cluster功能，多个ProxySQL节点同时工作，配合ELB负载均衡服务，保证业务的高可用。或者也可以考虑使用AWS的自动扩展组功能，对ProxySQL节点配置ASG策略，保证多个节点可以同时工作。

