版本：3.6.12

下载mongo：cd /opt && curl -o mongodb-linux-x86_64-rhel70-3.6.12.tgz  https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-rhel70-3.6.12.tgz

解压：tar -zxvf mongodb-linux-x86_64-rhel70-3.6.12.tgz

重命名为mongo：mv mongodb-linux-x86_64-rhel70-3.6.12 mongodb

添加环境变量：vim /etc/profile

```
# mongodb env
export MONGODB_HOME=/opt/mongodb
export PATH=$PATH:${MONGODB_HOME}/bin
```

在单机上安装mongo伪集群（3配置服务+3分片（每个分片3副本）+3mongos）

分三个目录，每个目录代表一个server，每个server下有一个config+一个mongos+3个分片
```
cd /opt/mongodb
# 创建目录
mkdir ./{server1,server2,server3}
mkdir ./server1/{conf,config,shard1,shard2,shard3,mongos}
mkdir ./server1/config/{data,logs}
mkdir ./server1/shard1/{data,logs}
mkdir ./server1/shard2/{data,logs}
mkdir ./server1/shard3/{data,logs}
mkdir ./server1/mongos/logs
 
mkdir ./server2/{conf,config,shard1,shard2,shard3,mongos}
mkdir ./server2/config/{data,logs}
mkdir ./server2/shard1/{data,logs}
mkdir ./server2/shard2/{data,logs}
mkdir ./server2/shard3/{data,logs}
mkdir ./server2/mongos/logs
 
mkdir ./server3/{conf,config,shard1,shard2,shard3,mongos}
mkdir ./server3/config/{data,logs}
mkdir ./server3/shard1/{data,logs}
mkdir ./server3/shard2/{data,logs}
mkdir ./server3/shard3/{data,logs}
mkdir ./server3/mongos/logs
 
# 创建config配置文件
cat >> server1/conf/config.conf << EOF
systemLog:
    verbosity: 0  #日志等级，0-5，默认0
    quiet: false  #限制日志输出，
    traceAllExceptions: true  #详细错误日志
    # syslogFacility: user #记录到操作系统的日志级别，指定的值必须是操作系统支持的，并且要以--syslog启动
    path: /opt/mongodb/server1/config/logs/mongod.log
    logAppend: true #启动时，日志追加在已有日志文件内还是备份旧日志后，创建新文件记录日志, 默认false
    # logRotate: rename #rename/reopen。rename，重命名旧日志文件，创建新文件记录；reopen，重新打开旧日志记录，需logAppend为true
    destination: file #日志输出方式。file/syslog,如果是file，需指定path，默认是输出到标准输出流中
    timeStampFormat: iso8601-local #日志日期格式。ctime/iso8601-utc/iso8601-local, 默认iso8601-local
    # component: #各组件的日志级别
    #    accessControl:
    #       verbosity: <int>
    #    command:
    #       verbosity: <int>
processManagement: 
    fork: true 
    pidFilePath: /opt/mongodb/server1/config/mongod.pid
net:
    bindIp: mongodb.test.com
    port: 27014
    maxIncomingConnections: 65536
    wireObjectCheck: true
    ipv6: false
storage:
    dbPath: /opt/mongodb/server1/config/data
    indexBuildRetry: true
    journal:
        enabled: true
        commitIntervalMs: 100
#security:
#    authorization: enabled
#    clusterAuthMode: keyFile
#    keyFile: /opt/mongodb/keyfile
setParameter:
   enableLocalhostAuthBypass: false
replication:
    oplogSizeMB: 10240
    replSetName: configsvr
    secondaryIndexPrefetch: all
sharding:
    clusterRole: configsvr
    archiveMovedChunks: true
EOF
cp server1/conf/config.conf server2/conf/config.conf && sed -i 's/server1/server2/g' server2/conf/config.conf && sed -i 's/27014/27024/g' server2/conf/config.conf
cp server1/conf/config.conf server3/conf/config.conf && sed -i 's/server1/server3/g' server3/conf/config.conf && sed -i 's/27014/27034/g' server3/conf/config.conf
 
# 创建分片配置文件
# 分片1及三个副本
cat >> server1/conf/shard1.conf << EOF
systemLog:
    verbosity: 0  #日志等级，0-5，默认0
    quiet: false  #限制日志输出，
    traceAllExceptions: true  #详细错误日志
    # syslogFacility: user #记录到操作系统的日志级别，指定的值必须是操作系统支持的，并且要以--syslog启动
    path: /opt/mongodb/server1/shard1/logs/mongod.log
    logAppend: true #启动时，日志追加在已有日志文件内还是备份旧日志后，创建新文件记录日志, 默认false
    # logRotate: rename #rename/reopen。rename，重命名旧日志文件，创建新文件记录；reopen，重新打开旧日志记录，需logAppend为true
    destination: file #日志输出方式。file/syslog,如果是file，需指定path，默认是输出到标准输出流中
    timeStampFormat: iso8601-local #日志日期格式。ctime/iso8601-utc/iso8601-local, 默认iso8601-local
    # component: #各组件的日志级别
    #    accessControl:
    #       verbosity: <int>
    #    command:
    #       verbosity: <int>
processManagement: 
    fork: true 
    pidFilePath: /opt/mongodb/server1/shard1/mongod.pid
net:
    bindIp: mongodb.test.com
    port: 27011
    maxIncomingConnections: 65536
    wireObjectCheck: true
    ipv6: false
storage:
    dbPath: /opt/mongodb/server1/shard1/data
    indexBuildRetry: true
    journal:
        enabled: true
        commitIntervalMs: 100
operationProfiling:
    slowOpThresholdMs: 100
    mode: slowOp
#security:
#    authorization: enabled
#    clusterAuthMode: keyFile
#    keyFile: /opt/mongodb/keyfile
setParameter:
   enableLocalhostAuthBypass: false
replication:
    oplogSizeMB: 10240
    replSetName: shard1
    secondaryIndexPrefetch: all
sharding:
    clusterRole: shardsvr
    archiveMovedChunks: true
EOF
cp server1/conf/shard1.conf server2/conf/shard1.conf && sed -i 's/server1/server2/g' server2/conf/shard1.conf && sed -i 's/27011/27021/g' server2/conf/shard1.conf
cp server1/conf/shard1.conf server3/conf/shard1.conf && sed -i 's/server1/server3/g' server3/conf/shard1.conf && sed -i 's/27011/27031/g' server3/conf/shard1.conf
# 分片2及三个副本
cp server1/conf/shard1.conf server1/conf/shard2.conf && sed -i 's/shard1/shard2/g' server1/conf/shard2.conf && sed -i 's/27011/27012/g' server1/conf/shard2.conf
cp server1/conf/shard2.conf server2/conf/shard2.conf && sed -i 's/server1/server2/g' server2/conf/shard2.conf && sed -i 's/27012/27022/g' server2/conf/shard2.conf
cp server1/conf/shard2.conf server3/conf/shard2.conf && sed -i 's/server1/server3/g' server3/conf/shard2.conf && sed -i 's/27012/27032/g' server3/conf/shard2.conf
# 分片3及三个副本
cp server1/conf/shard1.conf server1/conf/shard3.conf && sed -i 's/shard1/shard3/g' server1/conf/shard3.conf && sed -i 's/27011/27013/g' server1/conf/shard3.conf
cp server1/conf/shard3.conf server2/conf/shard3.conf && sed -i 's/server1/server2/g' server2/conf/shard3.conf && sed -i 's/27013/27023/g' server2/conf/shard3.conf
cp server1/conf/shard3.conf server3/conf/shard3.conf && sed -i 's/server1/server3/g' server3/conf/shard3.conf && sed -i 's/27013/27033/g' server3/conf/shard3.conf
 
# 创建mongos配置文件
cat >> server1/conf/mongos.conf << EOF
systemLog:
    verbosity: 0  #日志等级，0-5，默认0
    quiet: false  #限制日志输出，
    traceAllExceptions: true  #详细错误日志
    # syslogFacility: user #记录到操作系统的日志级别，指定的值必须是操作系统支持的，并且要以--syslog启动
    path: /opt/mongodb/server1/mongos/logs/mongos.log
    logAppend: true #启动时，日志追加在已有日志文件内还是备份旧日志后，创建新文件记录日志, 默认false
    # logRotate: rename #rename/reopen。rename，重命名旧日志文件，创建新文件记录；reopen，重新打开旧日志记录，需logAppend为true
    destination: file #日志输出方式。file/syslog,如果是file，需指定path，默认是输出到标准输出流中
    timeStampFormat: iso8601-local #日志日期格式。ctime/iso8601-utc/iso8601-local, 默认iso8601-local
    # component: #各组件的日志级别
    #    accessControl:
    #       verbosity: <int>
    #    command:
    #       verbosity: <int>
processManagement: 
    fork: true
    pidFilePath: /opt/mongodb/server1/mongos/mongos.pid
net:
    bindIp: mongodb.test.com
    port: 27015
    maxIncomingConnections: 65536
    wireObjectCheck: true
    ipv6: false
#security:
#    clusterAuthMode: keyFile
#    keyFile: /opt/mongodb/keyfile
setParameter:
   enableLocalhostAuthBypass: true
replication:
    localPingThresholdMs: 15
sharding:
    configDB: configsvr/mongodb.test.com:27014,mongodb.test.com:27024,mongodb.test.com:27034
EOF
cp server1/conf/mongos.conf server2/conf/mongos.conf && sed -i 's/server1/server2/g' server2/conf/mongos.conf && sed -i 's/27015/27025/g' server2/conf/mongos.conf
cp server1/conf/mongos.conf server3/conf/mongos.conf && sed -i 's/server1/server3/g' server3/conf/mongos.conf && sed -i 's/27015/27035/g' server3/conf/mongos.conf
```

启动实例前务必配置host：vi /etc/hosts 添加 

127.0.0.1 mongodb.test.com

启动config并初始化副本集
```
cd /opt/mongodb
# 创建启动config脚本
cat >> start-mongodb-cluster-config.sh << EOF
#!/bin/sh
mongod -f /opt/mongodb/server1/conf/config.conf
mongod -f /opt/mongodb/server2/conf/config.conf
mongod -f /opt/mongodb/server3/conf/config.conf
echo "config server started..."
EOF
chmod u+x start-mongodb-cluster-config.sh
./start-mongodb-cluster-config.sh
 
# 登录任意一台服务器，初始化配置副本集
mongo localhost:27014/admin
rs.initiate({_id: "configsvr",members:[{_id: 0,host: "127.0.0.1:27014"},{_id: 1,host: "127.0.0.1:27024"},{_id: 2,host: "127.0.0.1:27034"}]})
```

启动分片并初始化副本集
```
cd /opt/mongodb
# 创建启动shard脚本
cat >> start-mongodb-cluster-shard.sh << EOF
#!/bin/sh
mongod -f /opt/mongodb/server1/conf/shard1.conf
mongod -f /opt/mongodb/server2/conf/shard1.conf
mongod -f /opt/mongodb/server3/conf/shard1.conf
echo "shard1 server started..."
sleep 5
mongod -f /opt/mongodb/server1/conf/shard2.conf
mongod -f /opt/mongodb/server2/conf/shard2.conf
mongod -f /opt/mongodb/server3/conf/shard2.conf
echo "shard2 server started..."
sleep 5
mongod -f /opt/mongodb/server1/conf/shard3.conf
mongod -f /opt/mongodb/server2/conf/shard3.conf
mongod -f /opt/mongodb/server3/conf/shard3.conf
echo "shard3 server started..."
EOF
chmod u+x start-mongodb-cluster-shard.sh
./start-mongodb-cluster-shard.sh
 
# 分别连接三个分片其中一个副本，初始化副本集
mongo localhost:27011/admin
rs.initiate({_id: "shard1",members:[{_id: 0,host: "127.0.0.1:27011"},{_id: 1,host: "127.0.0.1:27021"},{_id: 2,host: "127.0.0.1:27031"}]})
 
mongo localhost:27012/admin
rs.initiate({_id: "shard2",members:[{_id: 0,host: "127.0.0.1:27012"},{_id: 1,host: "127.0.0.1:27022"},{_id: 2,host: "127.0.0.1:27032"}]})
 
mongo localhost:27013/admin
rs.initiate({_id: "shard3",members:[{_id: 0,host: "127.0.0.1:27013"},{_id: 1,host: "127.0.0.1:27023"},{_id: 2,host: "127.0.0.1:27033"}]})
```

启动mongos
```
cd /opt/mongodb
# 创建启动脚本
cat >> start-mongodb-cluster-mongos.sh << EOF
#!/bin/sh
mongos -f /opt/mongodb/server1/conf/mongos.conf
mongos -f /opt/mongodb/server2/conf/mongos.conf
mongos -f /opt/mongodb/server3/conf/mongos.conf
echo "mongos server started..."
EOF
chmod u+x start-mongodb-cluster-mongos.sh
./start-mongodb-cluster-mongos.sh
```

启用分片
```
cd /opt/mongodb
mongo localhost:27015/admin
sh.addShard("shard1/127.0.0.1:27011,127.0.0.1:27021,127.0.0.1:27031")
sh.addShard("shard2/127.0.0.1:27012,127.0.0.1:27022,127.0.0.1:27032")
sh.addShard("shard3/127.0.0.1:27013,127.0.0.1:27023,127.0.0.1:27033")
sh.status()
```

测试分片
```
mongo localhost:27015/admin
db.runCommand({enablesharding: "testdb"})
db.runCommand({shardcollection: "testdb.table1",key: {id: "hashed"}})
use testdb
for (var i=1; i<=5000; i++){ db.table1.insert({id: i,text: "hello world"}) }
db.table1.stats()
```

创建root和管理员账号
```
mongo localhost:27015/admin
# 创建一个超级用户root
db.createUser({user: "root",pwd: "root",roles: ["root"]})
# 创建一个管理用户admin可对所有数据库读写和用户管理
db.createUser({user: "admin",pwd: "admin",roles: ["userAdminAnyDatabase", "readWriteAnyDatabase", "dbAdminAnyDatabase", 'clusterAdmin']})
# 创建一个测试用户testdb可对testdb数据库进行读写其他数据库只读
db.createUser({user: "testdb",pwd: "testdb",roles: [{role: "readWrite",db: "testdb"},"read"]})
# 创建一个demo用户作为demo数据库的所有者，然后可以使用次账号登录 mongodb://demo:demo@localhost:27015/openinstallonline
use demo
db.createUser({user:"demo",pwd:"demo",roles:[{role:"dbOwner",db:"demo"}]});
use admin
db.runCommand({enablesharding:"demo"});
use demo;
 
# 用户认证
db.auth("admin", "admin")
# 查看创建的用户
show users 或 db.system.users.find() 或 db.runCommand({usersInfo:"userName"})
# 修改密码
use admin
db.changeUserPassword("username", "xxx")
# 删除用户
use admin
db.dropUser('xxx')
```

开启认证
```
# 生成认证文件
cd /opt/mongodb
openssl rand -base64 756 > keyfile
chmod 400 keyfile
 
# 修改所有config/shard/mongos配置文件开启认证
sed -i 's/#//g' server1/conf/config.conf
sed -i 's/#//g' server2/conf/config.conf
sed -i 's/#//g' server3/conf/config.conf
 
sed -i 's/#//g' server1/conf/shard1.conf
sed -i 's/#//g' server2/conf/shard1.conf
sed -i 's/#//g' server3/conf/shard1.conf
 
sed -i 's/#//g' server1/conf/shard2.conf
sed -i 's/#//g' server2/conf/shard2.conf
sed -i 's/#//g' server3/conf/shard2.conf
 
sed -i 's/#//g' server1/conf/shard3.conf
sed -i 's/#//g' server2/conf/shard3.conf
sed -i 's/#//g' server3/conf/shard3.conf
 
sed -i 's/#//g' server1/conf/mongos.conf
sed -i 's/#//g' server2/conf/mongos.conf
sed -i 's/#//g' server3/conf/mongos.conf
 
关闭mongod和mongos重新启动
killall mongod && killall mongos
./start-mongodb-cluster-config.sh
./start-mongodb-cluster-shard.sh
./start-mongodb-cluster-mongos.sh
 
验证账号登录
mongo localhost:27015/admin
db.auth("admin", "admin")
```


备注：mongodb启动顺序为先启动config配置服务，在启动shard分片服务，最后启动mongos

停止mongodb可以直接 killall mongod && killall mongos （记住不能直接kill -5 和 -9 得用 -2 或者 -4），当然最好的方式是登录mongos执行命令：use admin 然后 db.shutdownServer()

用户角色关系表

- 数据库用户角色：read、readWrite；
- 数据库管理角色：dbAdmin、dbOwner、userAdmin;
- 集群管理角色：clusterAdmin、clusterManager、4. clusterMonitor、hostManage；
- 备份恢复角色：backup、restore；
- 所有数据库角色：readAnyDatabase、readWriteAnyDatabase、userAdminAnyDatabase、dbAdminAnyDatabase
- 超级用户角色：root
- 内部角色：__system
- Read：允许用户读取指定数据库
- readWrite：允许用户读写指定数据库
- dbAdmin：允许用户在指定数据库中执行管理函数，如索引创建、删除，查看统计或访问system.profile
- userAdmin：允许用户向system.users集合写入，可以在指定数据库里创建、删除和管理用户
- clusterAdmin：只在admin数据库中可用，赋予用户所有分片和复制集相关函数的管理权限。
- readAnyDatabase：只在admin数据库中可用，赋予用户所有数据库的读权限
- readWriteAnyDatabase：只在admin数据库中可用，赋予用户所有数据库的读写权限
- userAdminAnyDatabase：只在admin数据库中可用，赋予用户所有数据库的userAdmin权限
- dbAdminAnyDatabase：只在admin数据库中可用，赋予用户所有数据库的dbAdmin权限。
- root：只在admin数据库中可用。超级账号，超级权限


注意事项
mongdb启动出现如下警告：
```
2019-05-11T17:29:08.696+0800 I CONTROL  [initandlisten] ** WARNING: /sys/kernel/mm/transparent_hugepage/enabled is 'always'.
2019-05-11T17:29:08.696+0800 I CONTROL  [initandlisten] **        We suggest setting it to 'never'
2019-05-11T17:29:08.696+0800 I CONTROL  [initandlisten]
2019-05-11T17:29:08.696+0800 I CONTROL  [initandlisten] ** WARNING: /sys/kernel/mm/transparent_hugepage/defrag is 'always'.
2019-05-11T17:29:08.696+0800 I CONTROL  [initandlisten] **        We suggest setting it to 'never'
2019-05-11T17:29:08.696+0800 I CONTROL  [initandlisten]
2019-05-11T17:29:08.696+0800 I CONTROL  [initandlisten] ** WARNING: soft rlimits too low. rlimits set to 63535 processes, 409600 files. Number of processes should be at least 204800 : 0.5 times number of files.
```

解决方法（针对警告1/2）：

临时解决：
```
sudo echo "never" > /sys/kernel/mm/transparent_hugepage/enabled
sudo echo "never" >  /sys/kernel/mm/transparent_hugepage/defrag
```

永久解决：
```
# 第一种：在/etc/rc.local中加入如下两行
if test -f /sys/kernel/mm/transparent_hugepage/enabled; then
 echo never > /sys/kernel/mm/transparent_hugepage/enabled
fi
if test -f /sys/kernel/mm/transparent_hugepage/defrag; then
 echo never > /sys/kernel/mm/transparent_hugepage/defrag
fi
然后确保该文件会在boot时执行
chmod +x /etc/rc.d/rc.local
 
# 第二种：
1 .编辑 /etc/default/grub，在GRUB_CMDLINE_LINUX加入选项 transparent_hugepage=never
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="rd.lvm.lv=fedora/swap rd.lvm.lv=fedora/root rhgb quiet transparent_hugepage=never"
GRUB_DISABLE_RECOVERY="true"
2.重新生成grub配置文件
On BIOS-based machines, issue the following command as root:
# grub2-mkconfig -o /boot/grub2/grub.cfg
On UEFI-based machines, issue the following command as root:
# grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg
step3 重启你的系统 reboot
```

警告3 rlimits 的解决方法：
```
vim /etc/security/limits.conf
# 添加一下几行
mongod  soft  nofile  1048576
mongod  hard  nofile  1048576
mongod  soft  nproc  524288
mongod  hard  nproc  524288
# 重启系统
reboot
```

参考：

https://blog.csdn.net/u013075468/article/details/51471033

https://blog.csdn.net/ffwar/article/details/77853498

