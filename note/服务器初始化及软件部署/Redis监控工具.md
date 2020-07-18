redmon安装，参考地址：https://github.com/steelThread/redmon

此工具只能监控4.0以下的（需先安装ruby环境）
```
安装redmon
 
gem install redmon
 
查看redmo帮助信息
redmon -h
 
启动redmon
nohup redmon -a 172.20.30.162 -b /redmon -p 4567 -r redis://172.20.30.162:7001 &
```

phpRedisAdmin安装参考：https://github.com/erikdubbelboer/phpRedisAdmin（需先安装php环境）

git clone https://github.com/ErikDubbelboer/phpRedisAdmin.git
cd phpRedisAdmin
git clone https://github.com/nrk/predis.git vendor
复制一份config.sample.inc.php修改配置文件
cd includes && cp config.sample.inc.php config.inc.php
vim config.inc.php
添加redis-server配置及basic认证login账号密码
配置nginx 代理配置
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
    location /phpRedisAdmin/ {
        index index.php;
        root /home/software;
        location ~ \.php$ {
            fastcgi_pass   127.0.0.1:9000;
            fastcgi_index  index.php;
            fastcgi_param  SCRIPT_FILENAME  /home/software/$fastcgi_script_name;
            include        fastcgi_params;
        }
    }
}

