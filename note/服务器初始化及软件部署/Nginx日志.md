通用日志配置
```
#user  nobody;
worker_processes  1;
 
error_log  logs/main_error.log;
 
pid        logs/nginx.pid;
 
events {
    worker_connections  1024;
}
 
http {
    include       mime.types;
    default_type  application/octet-stream;
 
    error_log     logs/http_error.log error;
    log_format    main escape=json '$server_name $remote_addr - $remote_user [$time_iso8601] "$request" '
                                   '$status $body_bytes_sent $request_time $bytes_sent $request_length '
                                   '"$http_referer" "$http_user_agent" "$http_x_forwarded_for" '
                                   '$upstream_addr $upstream_status $upstream_response_time';
    open_log_file_cache max=1000 inactive=20s valid=1m min_uses=2;
 
    map $time_iso8601 $logdate {
        '~^(?<ymd>\d{4}-\d{2}-\d{2})' $ymd;
        default                       'date-not-found';
    }
 
    access_log    logs/access-$logdate.log  main;
 
    sendfile        on;
    #tcp_nopush     on;
 
    #keepalive_timeout  0;
    keepalive_timeout  65;
 
    #gzip  on;
 
    split_clients $remote_addr $domain {
        10% www.localhost.io;
        * localhost:8080;
    }
 
    server {
        #server_name ~^(www\.)?(.+)$;
        server_name localhost.io;
        location / {
            return 301 $scheme://localhost:8080$request_uri;
            #rewrite ^/(.*)$ $scheme://$domain/$1 permanent;
        }
    }
 
    server {
        listen       8080;
        server_name  localhost;
        charset utf-8;
 
        location / {
            root   html;
            index  index.html index.htm;
 
            location ~ .*\.(gif|jpg|jpeg|png|bmp|swf)$ {
                expires        7d;
                log_not_found  off;
                access_log     off;
            }
 
            location ~ .*\.(js|css)?$ {
                expires      12h;
                log_not_found  off;
                access_log off;
            }
        }
 
        #error_page  404              /404.html;
 
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
 
        location = /(robots.txt|favicon.ico) {
            log_not_found  off;
            access_log off;
        }
    }
}
```

下面是log_format指令中常用的一些变量：

变量|含义
:--|:--
-|空白，变量没有值时用一个“-”占位符替代
$server_name|虚拟主机名称
$remote_addr|客户端IP
$remote_user|客户端用户名称，针对启用了用户认证的请求
$time_iso8601|标准格式的本地时间,形如“2017-05-24T18:31:27+08:00”
$time_local|通用日志格式下的本地时间，如"24/May/2017:18:31:27 +0800"
$request|完整的原始请求行，如 "GET / HTTP/1.1"
$request_uri|完整的请求地址，如 "https://daojia.com/"
$request_length|请求长度（包括请求行，请求头和请求体）
$request_time|请求处理时长，单位为秒，精度为毫秒，从读入客户端的第一个字节开始，直到把最后一个字符发送张客户端进行日志写入为止
$http_referer|请求的referer地址，记录从哪个页面链接访问过来的
$http_user_agent|客户端浏览器信息
$http_x_forwarded_for|当前端有代理服务器时，通过$remote_add拿到的IP地址是反向代理服务器的iP地址，反向代理服务器在转发请求的http头信息中，可以增加x_forwarded_for信息，用以记录原有客户端的IP地址和原来客户端的请求的服务器地址。
$status|响应状态码
$bytes_sent|发送给客户端的总字节数
$body_bytes_sent|发送给客户端的字节数，不包括响应头的大小，可以将日志每条记录中的这个值累加起来以粗略估计服务器吞吐量
$uptream_status|upstream状态，如200
$upstream_addr|upstream的地址，即真正提供服务的主机地址
$upstream_response_time|请求过程中，upstream的响应时间
$connection|连接序列号
$connection_requests|当前通过连接发出的请求数量
$msec|日志写入时间，单位为秒，精度是毫秒
$pipe|如果请求是通过http流水线发送，则其值为"p"，否则为“."
$ssl_protocol|SSL协议版本，比如TLSv1
$ssl_cipher|交换数据中的算法，比如RC4-SHA

### 参考：

https://juejin.im/post/59f94f626fb9a045023af34c

https://jingsam.github.io/2019/01/15/nginx-access-log.html



Nginx处理请求后把关于客户端请求的信息写到访问日志。默认，访问日志位于 logs/access.log，写到日志的信息是预定义的、组合的格式。要覆盖默认的配置，使用log_format指令来配置一个记录信息的格式，同样使用access_log 指令到设置日志和格式和位置。格式定义使用变量。

1、自定义一个日志格式
```
log_format  mylogformat '"$remote_addr" "[$time_local]" "$request_method" '
                      '"$uri" "$request_uri" "$request_time" "$status" "$body_bytes_sent"'
                       '"$http_referer" "$http_x_forwarded_for" "$http_user_agent" "$upstream_status"'
                      '"$upstream_addr" "$upstream_response_time"';
```

2、然后配置新的生成日志文件名：
```
access_log  logs/access1.log  mylogformat if=$loggable;
```

3、过滤掉需要过滤的日志，此处是过滤掉以2或者3开头的响应码
```
map $status $loggable {
                ~^[23]  0;
                     default 1;
  }

```

