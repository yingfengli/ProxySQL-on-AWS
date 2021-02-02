
本文采用开源软件ProxySQL作为数据库防火墙，以代理中间件方式部署在RDS MySQL数据库之前，配置和运维都非常简单。
ProxySQL的Firewall配置
在系列blog（一）中的ProxySQL基本配置完成后，测试环境已经准备好，可以开始做Firewall的配置和测试。

在2.0.9之前的ProxySQL，可以在mysql_query_rules表中配置多个查询规则来实现防火墙的功能，但在实际环境中，如果有几百上千个sql查询需要进行配置的话，对规则的配置实现起来就会非常复杂。
Firewall特性是ProxySQL在2.0.9中推出的新功能，为数据库防火墙提供了简化配置。其中主要通过两个表来配置：
mysql_firewall_whitelist_rules 
mysql_firewall_whitelist_users

登陆ProxySQL管理界面。先启用两个参数，以便把查询记录到stats.stats_mysql_query_digest内存表，再把这个表中数据自动保存到磁盘上的history_mysql_query_digest表：
Admin> set mysql-query_digests = 1;
Admin> set admin-stats_mysql_query_digest_to_disk = 1;
Admin> LOAD MYSQL VARIABLES TO RUNTIME;
Admin> SAVE MYSQL VARIABLES TO DISK;

1.创建测试用的数据库和表
通过ProxySQL代理后访问RDS MySQL数据库，创建两个数据库，每个数据库一个表，每个表两条记录。
$mysql -u user01 -pWelcome1 -h 127.0.0.1 -P6033 --prompt='RDS> '
RDS>create database proxy_test1;	
RDS>create database proxy_test2;	
RDS>use proxy_test1	
RDS>create table table1 (id int auto_increment, col1 varchar(100), col2 varchar(100), primary key(id));	
RDS>insert into table1 values (1, 'shanghai', '2020');	
RDS>insert into table1 values (2, 'Beijing', '2020');	
RDS>select * from table1;	
+----+----------+------+
| id | col1     | col2 |
+----+----------+------+
|  1 | shanghai | 2020 |
|  2 | Beijing  | 2020 |
+----+----------+------+
2 rows in set (0.00 sec)

RDS>use proxy_test2	
RDS>create table table2 (id int auto_increment, col1 varchar(100), col2 varchar(100), primary key(id));	
RDS>insert into table2 values (1, 'Guangzhou', '2019');	
RDS>insert into table2 values (2, 'Shenzhen', '2019');	
RDS>select * from table2;	
+----+-----------+------+
| id | col1      | col2 |
+----+-----------+------+
|  1 | Guangzhou | 2019 |
|  2 | Shenzhen  | 2019 |
+----+-----------+------+
2 rows in set (0.00 sec)

2.查看stats和history表的记录，注意记录digest。
Admin> select distinct(digest), username, digest_text from stats_mysql_query_digest where username = 'user01';
Admin> select distinct(digest), username, digest_text from stats_history.history_mysql_query_digest where username = 'user01';
 

如果history_mysql_query_digest没有记录，执行以下命令再查记录：
Admin> SAVE MYSQL DIGEST TO DISK;
Admin> select distinct(digest), username, digest_text from stats_history.history_mysql_query_digest where username = 'user01';
 

1.	启用Firewall Whitelist功能
这个功能不启用的话，任何查询都被允许通过。
Admin> SET mysql-firewall_whitelist_enabled = 1;
Admin> SET mysql-firewall_whitelist_errormsg = 'The ProxySQL Firewall blocked this query';
Admin> LOAD MYSQL VARIABLES TO RUNTIME;
Admin> SAVE MYSQL VARIABLES TO DISK;

4.把用户user01加入whitelist的users表

Admin> INSERT INTO mysql_firewall_whitelist_users (active, username, client_address, mode, comment) VALUES (1, 'user01', '', 'OFF', '');
Admin> LOAD MYSQL FIREWALL TO RUNTIME;
Admin> SAVE MYSQL FIREWALL TO DISK;

用户的状态有三个模式：
	OFF : 允许任何查询
	DETECTING :	允许任何查询，但没有被在mysql_firewall_whitelist_rules被enable的查询会记录到error log
	PROTECTING :只允许mysql_firewall_whitelist_rules里enable的查询，其它查询都会被block
			
5.将用户状态改为Detecting模式
Admin> UPDATE mysql_firewall_whitelist_users SET mode='DETECTING' WHERE username = 'user01';
Admin> LOAD MYSQL FIREWALL TO RUNTIME;
Admin> SAVE MYSQL FIREWALL TO DISK;

通过ProxySQL对proxy_test1的table做几个查询
RDS> use proxy_test1;		
RDS> show tables;		
RDS> select * from table1;	
查看/var/lib/ProxySQL/ProxySQL.log，有unkown query的告警。
$sudo cat /var/lib/ProxySQL/ProxySQL.log
2020-04-10 09:50:43 MySQL_Session.cpp:134:kill_query_thread(): [WARNING] KILL CONNECTION 11981 on mytest57-r2.cs8yfqtavsti.rds.cn-northwest-1.amazonaws.com.cn:3306
2020-04-10 09:50:43 Query_Processor.cpp:1905:process_mysql_query(): [WARNING] Firewall detected unknown query with digest 0x02033E45904D3DF0 from user user01@127.0.0.1
2020-04-10 09:50:43 Query_Processor.cpp:1905:process_mysql_query(): [WARNING] Firewall detected unknown query with digest 0x99531AEFF718C501 from user user01@127.0.0.1
2020-04-10 09:50:43 Query_Processor.cpp:1905:process_mysql_query(): [WARNING] Firewall detected unknown query with digest 0xD82EC15EB253E71B from user user01@127.0.0.1
2020-04-10 09:50:46 Query_Processor.cpp:1905:process_mysql_query(): [WARNING] Firewall detected unknown query with digest 0x99531AEFF718C501 from user user01@127.0.0.1
2020-04-10 09:50:51 Query_Processor.cpp:1905:process_mysql_query(): [WARNING] Firewall detected unknown query with digest 0xA6D96F2525BD0179 from user user01@127.0.0.1
6.将用户状态改为Protect模式
如果改为Protect保护模式，但还没有配置whitelist 规则的情况下，所有查询都被block。
Admin> UPDATE mysql_firewall_whitelist_users SET mode='PROTECTING' WHERE username = 'user01';
Admin> LOAD MYSQL FIREWALL TO RUNTIME;
Admin> SAVE MYSQL FIREWALL TO DISK;

通过ProxySQL代理登陆RDS MySQL数据库

RDS> show databases;
ERROR 1148 (42000): ProxySQL Firewall blocked this query
RDS> use proxy_test1;
Database changed
RDS> show tables;
ERROR 1148 (42000): ProxySQL Firewall blocked this query
RDS> select * from table1;
ERROR 1148 (42000): ProxySQL Firewall blocked this query

查看/var/lib/ProxySQL/ProxySQL.log，有查询被block的信息
2020-04-10 10:04:35 Query_Processor.cpp:1905:process_mysql_query(): [WARNING] Firewall blocked query with digest 0x02033E45904D3DF0 from user user01@127.0.0.1
2020-04-10 10:04:55 Query_Processor.cpp:1905:process_mysql_query(): [WARNING] Firewall blocked query with digest 0x620B328FE9D6D71A from user user01@127.0.0.1
2020-04-10 10:04:55 Query_Processor.cpp:1905:process_mysql_query(): [WARNING] Firewall blocked query with digest 0x02033E45904D3DF0 from user user01@127.0.0.1
2020-04-10 10:04:55 Query_Processor.cpp:1905:process_mysql_query(): [WARNING] Firewall blocked query with digest 0x99531AEFF718C501 from user user01@127.0.0.1
2020-04-10 10:05:04 Query_Processor.cpp:1905:process_mysql_query(): [WARNING] Firewall blocked query with digest 0x99531AEFF718C501 from user user01@127.0.0.1
2020-04-10 10:05:09 Query_Processor.cpp:1905:process_mysql_query(): [WARNING] Firewall blocked query with digest 0xA6D96F2525BD0179 from user user01@127.0.0.1

7.配置whitelist规则
可以根据stats_history.history_mysql_query_digest中的记录，按照需要逐条添加rule。但如果查询很多，建议采用批量方式，将stats_history.history_mysql_query_digest表的记录导入到whitelist表中，再根据需要进行调整。
Admin> INSERT INTO mysql_firewall_whitelist_rules (active, username, client_address, schemaname, flagIN, digest, comment) SELECT DISTINCT 1, username, client_address, schemaname, 0, digest, '' FROM stats_history.history_mysql_query_digest;
Admin> LOAD MYSQL FIREWALL TO RUNTIME;
Admin> SAVE MYSQL FIREWALL TO DISK;

查看规则
Admin> select * from mysql_firewall_whitelist_rules;
+--------+----------+----------------+-------------+--------+--------------------+---------+
| active | username | client_address | schemaname  | flagIN | digest             | comment |
+--------+----------+----------------+-------------+--------+--------------------+---------+
| 1      | user01   |                | proxy_test2 | 0      | 0x907932FE528CCC74 |         |
| 1      | user01   |                | proxy_test2 | 0      | 0xA61313D2974A4080 |         |
| 1      | user01   |                | proxy_test2 | 0      | 0xA74FAF80A4A93A29 |         |
| 1      | user01   |                | proxy_test2 | 0      | 0x99531AEFF718C501 |         |
| 1      | user01   |                | test01      | 0      | 0x226CD90D52A2BA0B |         |
| 1      | user01   |                | test01      | 0      | 0x7668E0FB7134207D |         |
| 1      | user01   |                | test01      | 0      | 0x566A3FD4ABB3B92F |         |
| 1      | user01   |                | test01      | 0      | 0x620B328FE9D6D71A |         |
| 1      | user01   |                | proxy_test1 | 0      | 0x02033E45904D3DF0 |         |
| 1      | user01   |                | proxy_test1 | 0      | 0x99531AEFF718C501 |         |
| 1      | user01   |                | proxy_test1 | 0      | 0xBA3A41A6FF5DFE4E |         |
| 1      | user01   |                | proxy_test1 | 0      | 0x236BBB7F360888D6 |         |
| 1      | user01   |                | proxy_test1 | 0      | 0xA6D96F2525BD0179 |         |
| 1      | user01   |                | proxy_test2 | 0      | 0x02033E45904D3DF0 |         |
| 1      | user01   |                | proxy_test1 | 0      | 0x620B328FE9D6D71A |         |
+--------+----------+----------------+-------------+--------+--------------------+---------+
15 rows in set (0.00 sec)

查看每个digest对应查询语句，可以查看stats_history.history_mysql_query_digest表。
Admin> SELECT username,schemaname, digest,digest_text FROM stats_history.history_mysql_query_digest;
 

可以根据业务需要，找到需要的查询语句，确定是否需要从mysql_firewall_whitelist_rules中删除。或者，也可以直接把需要允许通过的查询语句，直接增加一条记录，其中的digest在stats_history.history_mysql_query_digest表中可以查到：
Admin> INSERT INTO mysql_firewall_whitelist_rules VALUES (1, 'user01', '', 'proxy_test1', 0, '0x1**************', '');
Admin> LOAD MYSQL FIREWALL TO RUNTIME;
Admin> SAVE MYSQL FIREWALL TO DISK;

接下来可以测试一下Firewall是否正常工作：
通过ProxySQL登陆RDS MySQL，然后执行多个查询语句。

Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MySQL connection id is 24
Server version: 5.7.22 (ProxySQL)
Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
RDS> show databases;
ERROR 1148 (42000): ProxySQL Firewall blocked this query
RDS> use proxy_test1;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A
Database changed
RDS> show tables;
+-----------------------+
| Tables_in_proxy_test1 |
+-----------------------+
| table1                |
+-----------------------+
1 row in set (0.01 sec)

RDS> select * from table1;
+----+----------+------+
| id | col1     | col2 |
+----+----------+------+
|  1 | shanghai | 2020 |
|  2 | Beijing  | 2020 |
+----+----------+------+
2 rows in set (0.00 sec)

RDS> select id from test1;
ERROR 1148 (42000): ProxySQL Firewall blocked this query
RDS> use proxy_test2;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
RDS> show tables;
+-----------------------+
| Tables_in_proxy_test2 |
+-----------------------+
| table2                |
+-----------------------+
1 row in set (0.00 sec)

RDS> select * from table2;
+----+-----------+------+
| id | col1      | col2 |
+----+-----------+------+
|  1 | Guangzhou | 2019 |
|  2 | Shenzhen  | 2019 |
+----+-----------+------+
2 rows in set (0.00 sec)

RDS> drop table table1;
ERROR 1148 (42000): ProxySQL Firewall blocked this query
RDS> drop database proxy_test1;
ERROR 1148 (42000): ProxySQL Firewall blocked this query
RDS>

可以看到，在whitelist中的查询语句都可以正常执行，而其它语句比如： select id from table1; drop table table1; drop database proxy_test1; 都会被block，从而很好的保护了数据库。

细心的同学可能会发现，配置好Whitelist的规则后，其中第一个执行的语句 show databases在 whitelist_rules表中是有的，为什么也被block呢？这是因为在第一次执行的时候，我是先执行了use proxy_test1，然后才运行了 show databases，所以这个语句有个对应schema是proxy_test1。那么，在把这个语句导入到规则时，show databases语句也带了这个shcema，在图中可以看到。
 

所以大家可以测试一下，先执行use proxy_test1，再执行show databases就可以正常显示结果了。
另外需要提示一下，有些在我们看来是一样的语句，对Firewall的whitelist规则来说是不同的，比如 select * from table1和 select * from proxy_test1.table1就是两条不同的语句。

小结

对于Firewall的使用，通常会先配置为“Detecting”模式，经过一段时间的业务运行后，把所有执行的SQL语句都记录下来，然后根据安全的要求，把正常的语句放入白名单规则，可以允许执行，其它语句都将被block。当然，也可以先在业务系统的开发测试环境进行部署，也会发现很多语句。

