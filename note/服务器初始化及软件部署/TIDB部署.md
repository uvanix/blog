使用 Docker Compose 快速构建 TiDB 集群

参考官方文档：https://pingcap.com/docs-cn/dev/how-to/get-started/deploy-tidb-from-docker-compose/

注意docker需禁用iptables防火墙，防止iptables重启后影响docker

centos7中修改 

vim /usr/lib/systemd/system/docker.service

ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --graph /home/docker --iptables=false

主要是添加最后一句 --iptables=false

然后重启docker

配置iptables规则

```
# Firewall configuration written by system-config-firewall
# Manual customization of this file is not recommended.
*nat
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
 
#em2 = eth0
-A POSTROUTING -o em2 -j MASQUERADE
COMMIT
 
*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
# Allow localhost
-A INPUT -i lo -j ACCEPT
-A OUTPUT -o lo -j ACCEPT
-A INPUT -p icmp -j ACCEPT
 
#Docker default em2 = eth0
-A FORWARD -i docker0 -o em2 -j ACCEPT
-A FORWARD -i em2 -o docker0 -j ACCEPT
-A FORWARD -i docker0 -o docker0 -j ACCEPT
 
# Docker user define network
-A FORWARD -i br-15b9332b5c3f -o em2 -j ACCEPT
-A FORWARD -i em2 -o br-15b9332b5c3f -j ACCEPT
-A FORWARD -i br-15b9332b5c3f -o br-15b9332b5c3f -j ACCEPT
 
-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
-A INPUT -s 192.168.1.0/24 -j ACCEPT
#-A INPUT -j REJECT --reject-with icmp-host-prohibited
#-A FORWARD -j REJECT --reject-with icmp-host-prohibited
-A INPUT -j DROP
-A OUTPUT -j ACCEPT
-A FORWARD -j DROP
COMMIT
```

部署好后设置root密码，修改时区及group by问题
```
select version();
 
SET PASSWORD FOR 'root'@'%' = 'xxx';
FLUSH PRIVILEGES;
 
SELECT @@global.time_zone, @@session.time_zone;
set @@global.time_zone = '+8:00';
set @@session.time_zone = '+8:00';
select now();
 
select @@global.sql_mode;
set @@global.sql_mode='STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION';
```

