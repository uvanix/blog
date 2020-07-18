## ES备份还原

使用工具：elasticsearch-dump


列出所有索引：

curl 'localhost:9200/_cat/indices?v'


## 下载kibana解压并修改配置并后台启动
```
wget https://artifacts.elastic.co/downloads/kibana/kibana-5.5.3-linux-x86_64.tar.gz
 
tar -zxvf kibana-5.5.3-linux-x86_64.tar.gz
mv kibana-5.5.3-linux-x86_64 kibana
 
vi kibana/config/kibana.yml
server.port: 5601
server.host: "172.20.30.162"
server.basePath: "/kibana"
elasticsearch.url: "http://127.0.0.1:9200"
logging.dest: /home/software/kibana/logs/kibana.log
 
 
cd kibana && nohup bin/kibana &
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
 
    location /kibana/ {
        proxy_http_version   1.1;
        proxy_set_header     Upgrade         $http_upgrade;
        proxy_set_header     Connection      'upgrade';
        proxy_set_header     Host            $host;
        proxy_set_header     X-Real-IP       $remote_addr;
        proxy_set_header     X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_cache_bypass   $http_upgrade;
  
        proxy_redirect       off;
        proxy_pass           http://172.20.30.162:5601/;
        rewrite ^/kibana/(.*)$ /$1 break;
  
        auth_basic           "Restricted";
        auth_basic_user_file vhost.d/passwd;
    }
}
```

