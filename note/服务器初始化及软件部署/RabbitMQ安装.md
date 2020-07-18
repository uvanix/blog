CentOS7-Yum安装
参考：https://ken.io/note/centos7-rabbitmq-install-setup


Erlang安装

创建Yum源
```
# 创建yum源
sudo vi /etc/yum.repos.d/rabbitmq-erlang.repo
 
# 文件内容
[rabbitmq-erlang]
name=rabbitmq-erlang
baseurl=https://dl.bintray.com/rabbitmq/rpm/erlang/21/el/7
gpgcheck=1
gpgkey=https://dl.bintray.com/rabbitmq/Keys/rabbitmq-release-signing-key.asc
repo_gpgcheck=0
enabled=1
 
# 清理缓存yum
yum clean all
yum makecache
```

安装 socat
```
sudo yum install -y socat
```

安装 Erlang
```
sudo yum install -y erlang
# 提示找不到依赖包时 可以去github下载最新的erlang rpm安装
rpm -Uvh https://github.com/rabbitmq/erlang-rpm/releases/download/v21.3.7.1/erlang-21.3.7.1-1.el7.x86_64.rpm
```

验证
```
#进入erlang命令行表示成功
erl
 
按 ctrl+g 进入命令模式输入 q 退出
```

RabbitMQ安装
```
wget https://github.com/rabbitmq/rabbitmq-server/releases/download/v3.7.14/rabbitmq-server-3.7.14-1.el7.noarch.rpm
sudo rpm -Uvh https://dl.bintray.com/rabbitmq/all/rabbitmq-server/3.7.14/rabbitmq-server-3.7.14-1.el7.noarch.rpm

#启动服务
sudo systemctl start rabbitmq-server
 
#查看状态
sudo systemctl status rabbitmq-server
 
#设置为开机启动
sudo systemctl enable rabbitmq-server

#启用Web Console

rabbitmq-plugins enable rabbitmq_management
```


```
登录web console
User can only log in via localhost 
修改 /usr/lib/rabbitmq/lib/rabbitmq_server-3.7.14/ebin/rabbit.app
{loopback_users, [<<"guest">>]} 改为 {loopback_users, []}



修改guest默认密码 rabbitmqctl  change_password  guest  guest
重启：systemctl restart rabbitmq-server
访问：http://127.0.0.1:15672
```



配置nginx反向代理
```
server {
    listen               *:6666;
    server_name          localhost;
    access_log           logs/localhost.log;
    error_log            logs/localhost.error.log;
  
    charset               utf-8;
 
    location / {
        root   html;
        index  index.html index.htm;
    }
  
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   html;
    }
 
    location /rabbitmq/ {
        proxy_http_version   1.1;
        proxy_set_header     Upgrade         $http_upgrade;
        proxy_set_header     Connection      'upgrade';
        proxy_set_header     Host            $host;
        proxy_set_header     X-Real-IP       $remote_addr;
        proxy_set_header     X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_cache_bypass   $http_upgrade;
  
        proxy_redirect       off;
        proxy_pass           http://172.20.30.162:15672;
        rewrite ^/rabbitmq/(.*)$ /$1 break;
    }
}
```

Docker方式安装
```
docker run -d --name rabbitmq --publish 5671:5671 --publish 5672:5672 --publish 4369:4369 --publish 25672:25672 --publish 15671:15671 --publish 15672:15672 rabbitmq:management

```

