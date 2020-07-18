### 清空容器

```
docker kill $(docker ps -aq)
docker rm $(docker ps -aq)
docker rmi $(docker images -q)
docker volume rm $(docker volume ls -q)
```

Config Docker Library Path
修改docker.service文件，使用-g参数指定存储位置，关闭docker修改iptables权限

$ vim /usr/lib/systemd/system/docker.service
ExecStart=/usr/bin/dockerd --graph /new-path/docker --iptables=false

Docker Iptables 参考配置：https://my.oschina.net/myaniu/blog/1800088

// reload配置文件

$ systemctl daemon-reload
// 重启docker 

$ systemctl restart docker.service

