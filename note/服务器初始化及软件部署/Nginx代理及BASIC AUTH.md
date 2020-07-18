1. 生成用户密码文件
```
cd nginx配置目录
printf "admin:$(openssl passwd -crypt 123456)\n" > passwd
chmod 400 passwd
```

或者使用httpd-tools
```
yum install httpd-tools               #适用centos
sudo apt-get install apache2-utils    #适用ubuntu
生成用户密码文件
 
$ htpasswd -c /home/software/openresty/nginx/conf/vhost.d/.htpasswd admin  #回车会要求输入两遍密码，会清除所有用户！
$ htpasswd -bc /home/software/openresty/nginx/conf/vhost.d/.htpasswd admin admin  #不用回车，直接指定user1的密码为password
$ htpasswd -b /home/software/openresty/nginx/conf/vhost.d/.htpasswd admin admin   #添加一个用户，如果用户已存在，则是修改密码
$ htpasswd -D /home/software/openresty/nginx/conf/vhost.d/.htpasswd admin  #删除用户
```

2.nginx代理kibana/rabbitmq/mongodb/mysql/redis

```
server {
    listen               *:6666;
    server_name          localhost;
    charset              utf-8;
 
    access_log           logs/localhost.log;
    error_log            logs/localhost.error.log;
 
    location / {
        root   html;
        index  index.html index.htm;
    }
 
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   html;
    }
 
    client_max_body_size 4000m;
    client_body_buffer_size 256k;
 
    proxy_http_version   1.1;
    proxy_set_header     Upgrade            $http_upgrade;
    proxy_set_header     Connection         'upgrade';
    proxy_set_header     Host               $host;
    proxy_set_header     X-Real-IP          $remote_addr;
    proxy_set_header     X-Forwarded-For    $proxy_add_x_forwarded_for;
    proxy_set_header     X-Forwarded-Host   $server_name;
    proxy_set_header     X-Forwarded-Proto  $scheme;
    proxy_cache_bypass   $http_upgrade;
 
    proxy_connect_timeout 600s;
    proxy_send_timeout 120;
    proxy_read_timeout 120;
    proxy_buffer_size 1024m;
    proxy_buffers 8 1024m;
    proxy_next_upstream http_404 http_403 http_502;
    proxy_next_upstream_tries 5;
    proxy_busy_buffers_size 1024m;
    proxy_temp_file_write_size 1024m;
 
    location /kibana/ {
        proxy_redirect       off;
        proxy_pass           http://172.20.30.162:5601/;
        rewrite ^/kibana/(.*)$ /$1 break;
 
        auth_basic           "Restricted";
        auth_basic_user_file vhost.d/passwd;
 
        #access_log      logs/kibana.access.log;
        #error_log       logs/kibana.error.log;
    }
 
    location /rabbitmq/ {
        proxy_redirect       off;
        proxy_pass           http://172.20.30.162:15672;
        rewrite ^/rabbitmq/(.*)$ /$1 break;
    }
 
    location /mongo-express/ {
        proxy_redirect       off;
        proxy_pass           http://172.20.30.162:8081/mongo-express/;
    }
 
    location /api {
        proxy_redirect       off;
        proxy_pass           http://172.20.30.162:8081/api;
 
        add_header Access-Control-Allow-Origin *;                                                            
        add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS';                                        
        add_header Access-Control-Allow-Credentials true;                                                    
        add_header Access-Control-Allow-Headers 'DNT,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Authorization';
        add_header Access-Control-Max-Age 1728000;                                                           
                                                                                                               
        if ($request_method = 'OPTIONS') {                                                                   
            return 204;                                                                                      
        }
    }
}
```

PS：

kibana需要配置server-path：server.basePath: "/kibana"

mongodb webui使用mongo-express: https://github.com/mongo-express/mongo-express/

