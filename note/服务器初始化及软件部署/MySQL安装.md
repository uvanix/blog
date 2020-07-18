卸载
```
# yum remove mysql* mariadb* -y
 
# rm /etc/my.cnf
 
# rm -rf /var/lib/mysql
 
# rm -rf /usr/share/mysql
 
# rm -f /var/log/mysqld.log
 
# rm -rf /var/run/mysql
 
查询mysql服务
 
# systemctl list-unit-files | grep mysql
 
# systemctl disable mysqld.service
 
# systemctl disable mysql.service
```

MySQL版本：5.7.23

rpm安装参考：https://blog.csdn.net/smiles13/article/details/81460617

解压：mkdir mysql-rpm && tar -xvf mysql-5.7.23-1.el7.x86_64.rpm-bundle.tar -C mysql-rpm

卸载centos7自带的mariadb-lib：rpm -qa|grep mariadb && rpm -e mariadb-libs-5.5.52-1.el7.x86_64 --nodeps

安装：cd mysql-rpm

rpm -ivh mysql-community-common-5.7.23-1.el7.x86_64.rpm

rpm -ivh mysql-community-libs-5.7.23-1.el7.x86_64.rpm

rpm -ivh mysql-community-client-5.7.23-1.el7.x86_64.rpm

rpm -ivh mysql-community-server-5.7.23-1.el7.x86_64.rpm

修改配置：vim /etc/my.cnf （参考后续的配置）

记得修改 setenforce 0 和 chown -R mysql.mysql mysqldatadir

初始化数据库：mysqld --initialize #初始化后会在/var/log/mysqld.log生成随机密码

（修改mysql数据库目录的所属用户及其所属组：chown mysql:mysql /var/lib/mysql -R）



启动数据库：systemctl start mysqld.service

加入开机启动：systemctl enable mysqld.service

登录mysql：mysql -uroot -p'123456'  

修改root密码：set password=password('123456');

set password for 'root'@'%'=password('123456');

修改root账号host：update mysql.user set host='%' where user='root';

更改账号host：RENAME USER 'root'@'%' TO 'root'@'localhost';

创建slave账号：GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'slave'@'localhost' IDENTIFIED BY '123456';

刷新权限：flush privileges;

创建数据库与账号：

CREATE DATABASE IF NOT EXISTS test DEFAULT CHARACTER SET utf8mb4 collate utf8mb4_general_ci;
GRANT ALL ON test.* TO 'test'@'%' IDENTIFIED BY 'test';


单例配置参考：/etc/my.conf  （注意创建mysql目录并更改用户及用户组：mkdir /var/log/mysql && chown -R mysql.mysql /var/log/mysql )
```
[mysqld]
server-id                         = 28
pid-file                          = /var/run/mysqld/mysqld.pid
socket                            = /var/lib/mysql/mysql.sock
datadir                           = /var/lib/mysql
user                              = mysql
tmpdir                            = /tmp
port                              = 3306
character_set_server              = utf8mb4
collation-server                  = utf8mb4_unicode_ci
init_connect                      = 'SET NAMES utf8mb4'
sql_mode                          = "NO_ENGINE_SUBSTITUTION,NO_AUTO_CREATE_USER,STRICT_TRANS_TABLES,NO_ZERO_DATE,NO_ZERO_IN_DATE,ERROR_FOR_DIVISION_BY_ZERO"
transaction-isolation             = READ-COMMITTED
secure_file_priv                  = ''
explicit_defaults_for_timestamp   = 0
 
lower_case_table_names            = 1
explicit_defaults_for_timestamp   = true
default-storage-engine            = INNODB
innodb_buffer_pool_size           = 2G
innodb_log_file_size              = 2G
innodb_flush_log_at_trx_commit    = 1
innodb_flush_method               = O_DIRECT
join_buffer_size                  = 128M
sort_buffer_size                  = 8M
read_rnd_buffer_size              = 8M
 
max_allowed_packet                = 256M
max_connections                   = 2048
open_files_limit                  = 65535
 
log-output                        = FILE
log-error                         = /var/log/mysql/mysqld.log
# 慢查询
slow-query-log                    = 1
slow_query_log_file               = /var/log/mysql/slow_query.log
# 慢查的时长单位为秒，可以精确到小数点后6位(微秒)
long_query_time                   = 1
# 二进制日志
binlog-ignore-db                  = mysql,information_schema,performance_schema,sys
log-bin                           = /var/log/mysql/mysql-bin
binlog_cache_size                 = 1M
binlog-format                     = Row
expire_logs_days                  = 7
slave_skip_errors                 = 1062
 
symbolic-links=0
 
[mysql]
default-character-set=utf8mb4
 
[client]
port                              = 3306
default-character-set             = utf8mb4
```

创建从库复制账号

CREATE USER slave IDENTIFIED BY 'slave'; GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'slave'@'%'; FLUSH PRIVILEGES; show grants for 'slave' 
从库配置：/etc/my.conf
```
[mysqld]
server-id                         = 28
pid-file                          = /var/run/mysqld/mysqld.pid
socket                            = /var/lib/mysql/mysql.sock
datadir                           = /var/lib/mysql
user                              = mysql
tmpdir                            = /tmp
port                              = 3306
character_set_server              = utf8mb4
collation-server                  = utf8mb4_unicode_ci
init_connect                      = 'SET NAMES utf8mb4'
sql_mode                          = "NO_ENGINE_SUBSTITUTION,NO_AUTO_CREATE_USER,STRICT_TRANS_TABLES,NO_ZERO_DATE,NO_ZERO_IN_DATE,ERROR_FOR_DIVISION_BY_ZERO"
transaction-isolation             = READ-COMMITTED
secure_file_priv                  = ''
explicit_defaults_for_timestamp   = 0
 
lower_case_table_names            = 1
explicit_defaults_for_timestamp   = true
default-storage-engine            = INNODB
innodb_buffer_pool_size           = 2G
innodb_log_file_size              = 2G
innodb_flush_log_at_trx_commit    = 1
innodb_flush_method               = O_DIRECT
join_buffer_size                  = 128M
sort_buffer_size                  = 8M
read_rnd_buffer_size              = 8M
 
max_allowed_packet                = 256M
max_connections                   = 2048
open_files_limit                  = 65535
 
log-output                        = FILE
log-error                         = /var/log/mysql/mysqld.log
# 慢查询
slow-query-log                    = 1
slow_query_log_file               = /var/log/mysql/slow_query.log
# 慢查的时长单位为秒，可以精确到小数点后6位(微秒)
long_query_time                   = 1
# 二进制日志
binlog-ignore-db                  = mysql,information_schema,performance_schema,sys
log-bin                           = /var/log/mysql/mysql-bin
binlog_cache_size                 = 1M
binlog-format                     = Row
expire_logs_days                  = 7
slave_skip_errors                 = 1062
# relay_log配置中继日志
relay_log                         = mysql-relay-bin
# log_slave_updates表示slave将复制事件写进自己的二进制日志
log_slave_updates                 = 1
# 防止改变数据(除了特殊的线程)
read_only                         = 1
symbolic-links=0
 
[mysql]
default-character-set=utf8mb4
 
[client]
port                              = 3306
default-character-set             = utf8mb4
```

在线添加从库参考：https://linux.cn/article-5782-1.html

```
MySQL主从是基于binlog日志，所以在安装好数据库后就要开启binlog。这样好处是，一方面可以用binlog恢复数据库，另一方面可以为主从做准备。
 
原有主库配置参数如下：
 
# vi my.cnf
server-id = 1             #id要唯一
log-bin = mysql-bin         #开启binlog日志
auto-increment-increment = 1   #在Ubuntu系统中MySQL5.5以后已经默认是1
auto-increment-offset = 1
slave-skip-errors = all      #跳过主从复制出现的错误
1. 主库创建同步账号
mysql> grant all on *.* to 'sync'@'192.168.18.%' identified by 'sync';
2. 从库配置MySQL
# vi my.cnf
server-id = 3             #这个设置3
log-bin = mysql-bin         #开启binlog日志
auto-increment-increment = 1   #这两个参数在Ubuntu系统中MySQL5.5以后都已经默认是1
auto-increment-offset = 1
slave-skip-errors = all      #跳过主从复制出现的错误
3. 备份主库
# mysqldump -uroot -p123 --routines --single_transaction --master-data=2 --databases weibo > weibo.sql
参数说明：
 
--routines：导出存储过程和函数
 
--single_transaction：导出开始时设置事务隔离状态，并使用一致性快照开始事务，然后unlock tables;而lock-tables是锁住一张表不能写操作，直到dump完毕。
 
--master-data：默认等于1，将dump起始（change master to）binlog点和pos值写到结果中，等于2是将change master to写到结果中并注释。
 
4. 把备份库拷贝到从库
# scp weibo.sql root@192.168.18.214:/home/root
5. 在主库创建test_tb表，模拟数据库新增数据，weibo.sql是没有的
mysql> create table test_tb(id int,name varchar(30));
6. 从库导入备份库
# mysql -uroot -p123 -e 'create database weibo;'
# mysql -uroot -p123 weibo < weibo.sql
7. 在备份文件weibo.sql查看binlog和pos值
# head -25 weibo.sql
-- CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=107;   #大概22行
8. 从库设置从这个日志点同步，并启动
mysql> change master to master_host='192.168.18.212',
    -> master_user='sync',
    -> master_password='sync',
    -> master_log_file='mysql-bin.000001',
    -> master_log_pos=107;
mysql> start slave;
mysql> show slave status\G;
ERROR 2006 (HY000): MySQL server has gone away
No connection. Trying to reconnect...
Connection id:    90
Current database: *** NONE ***
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.18.212
                  Master_User: sync
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 358
               Relay_Log_File: mysqld-relay-bin.000003
                Relay_Log_Pos: 504
        Relay_Master_Log_File: mysql-bin.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
......
9. 从库查看weibo库里面的表
可以看到IO和SQL线程均为YES，说明主从配置成功。
 
mysql> show tables;
+---------------------------+
| Tables_in_weibo           |
+---------------------------+
| test_tb                   |
发现刚才模拟创建的test_tb表已经同步过来！
```

备份全库：

mysqldump -uroot -p'root' --routines --all-databases  | gzip > mysql63.sql.tar.gz

备份指定数据库：

mysqldump -uroot -p'root' --routines --databases database_name mysql | gzip > mysql63.sql.tar.gz

导入：

mysql -uroot -p'root' < mysql63.sql

binlog 命令查看恢复：

登录数据库刷新binlog
show master status;
flush logs;
show master status;

查询二进制日志位置
show variables like'log_bin%';

从二进制日志中获取表被删除的时间
mysqlbinlog --no-defaults -d fc_video mysql-bin.000007 | grep --ignore-case DROP -A3 -B4

drop语句的前两行表名drop语句的执行时间是在 16:00:41

SELECT from_unixtime('1559289641');

```
从binlog中获取指定数据库的改变数据
获取所有binlog的数据
mysqlbinlog --no-defaults -d fc_video mysql-bin.000004 > restore.sql
mysqlbinlog --no-defaults -d fc_video mysql-bin.000005 >> restore.sql
mysqlbinlog --no-defaults -d fc_video mysql-bin.000006 >> restore.sql
mysqlbinlog --no-defaults -d fc_video --stop-datetime="2019-05-31 16:00:00" mysql-bin.000007 >> restore.sql


more restore.sql | grep -A1 -B3 -i -E '^insert|^update|^delete|^replace|^alter' | grep -A1 -B3 fc_column

将指定的表操作内容过滤到指定文件

grep -B3 -w tb_name data.sql |grep -v  '^--$' >tb_name.sql


```

