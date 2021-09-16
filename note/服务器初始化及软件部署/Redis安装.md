版本：5.x
下载redis5.x：cd /opt && curl -o redis-5.0.13.tar.gz http://download.redis.io/releases/redis-5.0.13.tar.gz   (wget http://download.redis.io/releases/redis-5.0.13.tar.gz)
创建redis安装目录并解压：mkdir /opt/redis && cd /opt && tar zxvf redis-5.0.13.tar.gz
编译安装redis：cd /opt/redis-5.0.13 && make && make install PREFIX=/opt/redis

添加环境变量：vim /etc/profile 添加如下内容
```
# redis env
export REDIS_HOME=/opt/redis
export PATH=$PATH:${REDIS_HOME}/bin
```

创建单例6379实例

mkdir -p /opt/redis/6379

创建配置文件：
```
cat >> /opt/redis/6379/redis.conf << EOF
bind 0.0.0.0
protected-mode no
port 6379
daemonize yes
pidfile /opt/redis/6379/redis.pid
logfile /opt/redis/6379/redis.log
save ""
dbfilename dump.rdb
dir /opt/redis/6379
appendonly yes
appendfilename "appendonly.aof"
# auth
#masterauth 654321
#requirepass 654321
EOF
```

启动redis-server：redis-server /opt/redis/6379/redis.conf



创建集群3主3从的实例（端口号7001～7006）

mkdir -p /opt/redis/{7001,7002,7003,7004,7005,7006}

给每个端口都创建配置文件：
```
cat >> /opt/redis/7001/redis.conf << EOF
bind 0.0.0.0
protected-mode no
port 7001
daemonize yes
pidfile /opt/redis/7001/redis.pid
logfile /opt/redis/7001/redis.log
save ""
dbfilename dump.rdb
dir /opt/redis/7001
appendonly yes
appendfilename "appendonly.aof"
cluster-enabled yes
cluster-node-timeout 5000
cluster-config-file /opt/redis/7001/nodes.conf
# auth
#masterauth 654321
#requirepass 654321
EOF
 
cd /opt/redis
cp -rf 7001/redis.conf 7002/redis.conf && sed -i 's/7001/7002/g' 7002/redis.conf
cp -rf 7001/redis.conf 7003/redis.conf && sed -i 's/7001/7003/g' 7003/redis.conf
cp -rf 7001/redis.conf 7004/redis.conf && sed -i 's/7001/7004/g' 7004/redis.conf
cp -rf 7001/redis.conf 7005/redis.conf && sed -i 's/7001/7005/g' 7005/redis.conf
cp -rf 7001/redis.conf 7006/redis.conf && sed -i 's/7001/7006/g' 7006/redis.conf
```

启动各个redis-server并加入集群（注意创建集群的时候ip不能使用127.0.0.1 否则java应用启动时候会报错127.0.0.1:7001连不上）：
```
# start redis cluster
cat >> start-redis-cluster.sh << EOF
#!/bin/sh
redis-server /opt/redis/7001/redis.conf
redis-server /opt/redis/7002/redis.conf
redis-server /opt/redis/7003/redis.conf
redis-server /opt/redis/7004/redis.conf
redis-server /opt/redis/7005/redis.conf
redis-server /opt/redis/7006/redis.conf
EOF
chmod u+x start-redis-cluster.sh
./start-redis-cluster.sh
ps -ef|grep redis
 
# create redis cluster
cat >> create-redis-cluster.sh << EOF
#!/bin/sh
redis-cli --cluster create 127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 127.0.0.1:7006 --cluster-replicas 1
EOF
chmod u+x create-redis-cluster.sh
./create-redis-cluster.sh
```

检查集群状态：
```
# cluster nodes
redis-cli --cluster check localhost:7001 -a 654321
redis-cli -c -p 7001 -a 654321
cluster info
cluster nodes
```

批量删除redis key 命令

redis-cli -c -h localhost -p 7001 keys zset* | xargs -t echo "redis-cli -c -h localhost -p 7001 del {}"

redis-cli -c -h localhost -p 7001 keys zset* | xargs -i echo "redis-cli -c -h localhost -p 7001 del {}" > del_redis_keys.sh

chmod u+x del_redis_keys.sh

./del_redis_keys.sh

