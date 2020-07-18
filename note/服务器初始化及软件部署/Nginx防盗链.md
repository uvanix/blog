使用ngx_http_secure_link_module 防盗链，参考：http://nginx.org/en/docs/http/ngx_http_secure_link_module.html
```
server {
    listen       80;
    server_name  mhteam.io;
 
    charset utf-8;
 
    access_log  logs/mhteam.io.access.log  main;
 
    location / {
        root   html;
        index  index.html index.htm;
 
        limit_rate_after 3048k;
        limit_rate       120k;
 
        location ~* \.(gif|jpg|png|jpeg|bmp)$ {
            valid_referers none blocked *.test.io;
            if ($invalid_referer) {
                return 403;
            }
                 
            expires        7d;
            log_not_found  off;
            access_log     off;
        }
 
        location ~ \.(mp4|m3u8)$ {
            rewrite ^(.*)/(.*)/(.*)/video/(.+)\.mp4$ $1/video/$4.mp4?s=$2&e=$3 last;
            secure_link      $arg_s,$arg_e;
            secure_link_md5  "customSecretKey$uri$arg_e";
            #add_header X-debug-message "customSecretKey$uri$arg_e" always;
            if ($secure_link = "") {
                return 403;
            }  
            if ($secure_link = "0") {
                return 410;
            }  
            #if ($request_filename ~* ^.*?\.(mp4|m3u8)$){
            #    add_header Content-Disposition 'attachment;';
            #}  
        }  
    }
 
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   html;
    }
}
```

shell生成md5：
```
currtime=$(date +%s)
echo $currtime
expires=$[$currtime+600]
echo $expires
data="customSecretKey/video/test.mp4$expires"
echo -n $data
echo -n $data | openssl md5 -binary | openssl base64 | tr +/ -_ | tr -d =
```

java代码生成md5:
```
public static void main(String[] args) {
    // +600代表600秒后地址失效
    String time = String.valueOf(System.currentTimeMillis() / 1000 + 600);
    String md5 = Base64.encodeBase64URLSafeString(DigestUtils.md5("customSecretKey" + "/video/test.mp4" + time));
    String url = "http://172.16.10.25/video/test.mp4?s=" + md5 + "&e=" + time;
    System.out.println(url);
}
```

