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

cdn-url鉴权m3u8：
```
local chunk, eof = ngx.arg[1], ngx.arg[2]
local buffered = ngx.ctx.buffered
if not buffered then
    buffered = {}
    ngx.ctx.buffered = buffered
end
if chunk ~= "" and not ngx.is_subrequest then
    buffered[#buffered + 1] = chunk
    ngx.arg[1] = nil
end
if eof then
    local whole = table.concat(buffered)
    ngx.ctx.buffered = nil
    local auth_err = function(err)
        ngx.log(ngx.ERR, "rewrite m3u8 file error: ", err)
        return ngx.exit(403)
    end
    local mp4, err = ngx.re.sub(ngx.var.uri, "index.m3u8", "", "jo")
    -- ngx.log(ngx.ERR, "######## "..mp4.." ########")
    local auth_uri = function(str)
        local uri = mp4..str[0]
        local timestamp = ngx.time() + ngx.var.auth_expires
        local auth_key = timestamp.."-0-0-"..ngx.md5(uri.."-"..timestamp.."-0-0-"..ngx.var.auth_secret)
        return str[0].."?auth_key="..auth_key
    end
    local m3u8, err = ngx.re.gsub(whole, "^seg-[a-z0-9-]+.ts$", auth_uri, "mjo")
    if not m3u8 then
        auth_err(err)
    else
        m3u8, err = ngx.re.sub(m3u8, "encryption.key", auth_uri, "jo")
        if not m3u8 then
            auth_err(err)
        end
    end
    ngx.arg[1] = m3u8
end
---
local auth_err = function(msg)
    local timestamp = ngx.time() + ngx.var.auth_expires
    local auth_key = timestamp.."-0-0-"..ngx.md5(ngx.var.uri.."-"..timestamp.."-0-0-"..ngx.var.auth_secret)
    ngx.log(ngx.ERR, "######## "..msg..", test auth_key is: "..auth_key.." ########")
    return ngx.exit(403)
end
local auth_key = ngx.var.arg_auth_key
if not auth_key then
    auth_err("no uri arg auth_key")
end
local timestamp = string.sub(ngx.var.arg_auth_key, 1, 10)
if ngx.time() > tonumber(timestamp) then
    auth_err("timestamp expired")
end
local auth_key = timestamp.."-0-0-"..ngx.md5(ngx.var.uri.."-"..timestamp.."-0-0-"..ngx.var.auth_secret)
if auth_key ~= ngx.var.arg_auth_key then
    auth_err("auth_key error")
end
```

cloudflare workers
```
const encoder = new TextEncoder()

async function verifyAndFetch(request) {
  const url = new URL(request.url)

  if (!url.searchParams.has("auth_key")) {
    return new Response("Missing query parameter. ", { status: 403 })
  }

  const authKey = url.searchParams.get("auth_key").split("-")
  if(authKey.length != 4) {
    return new Response("Invalid Sign. ", { status: 403 })
  }

  const expiry = Number(authKey[0])
  const now = Date.now() + 28800
  if (now > expiry * 1000) {
    const body = `URL expired at ${expiry}`
    return new Response(body, { status: 403 })
  }

  const signStr = url.pathname + "-" + expiry + "-" + authKey[1] + "-" + authKey[2] + "-rwwkz8oV132t"
  const dataBuf = encoder.encode(signStr)
  const signBuf = await crypto.subtle.digest("MD5", dataBuf);
  const signArr = Array.from(new Uint8Array(signBuf));
  const signHex = signArr.map(b => b.toString(16).padStart(2, '0')).join('');
  if(signHex === authKey[3]) {
    return fetch(request)
    // return new Response("Sign Right.", { status: 403 })
  }
  
  return new Response("Invalid Sign. " + signHex + " " + signStr, { status: 403 })
}

addEventListener("fetch", event => {
  event.respondWith(verifyAndFetch(event.request))
})
```
http.conf
```
server_tokens off;

proxy_http_version   1.1;
proxy_set_header     Upgrade            $http_upgrade;
proxy_set_header     Connection         'upgrade';
proxy_set_header     User-Agent         $http_user_agent;
proxy_set_header     Host               $host;
proxy_set_header     Referer            $http_referer;
proxy_set_header     Cookie             $http_cookie;
proxy_set_header     X-Real-IP          $remote_addr;
proxy_set_header     X-Forwarded-For    $proxy_add_x_forwarded_for;
proxy_set_header     X-Forwarded-Host   $server_name;
proxy_set_header     X-Forwarded-Proto  $scheme;
proxy_hide_header    X-Powered-By;
proxy_cache_bypass   $http_upgrade;

gzip on;
gzip_min_length 1k;
gzip_buffers 4 16k;
#gzip_http_version 1.0;
gzip_comp_level 4;
gzip_types  text/plain text/css text/javascript application/javascript application/x-javascript application/xml application/json image/jpeg image/gif image/png application/font-woff video/mp2t video/mp4;
gzip_vary on;
gzip_disable "MSIE [1-6]\.";

```
m3u8.access.lua
```
local auth_err = function(msg)
    local timestamp = ngx.time() + ngx.var.auth_expires
    local auth_key = timestamp.."-0-0-"..ngx.md5(ngx.var.uri.."-"..timestamp.."-0-0-"..ngx.var.auth_secret)
    ngx.log(ngx.ERR, "######## "..msg..", test auth_key is: "..auth_key.." ########")
    return ngx.exit(403)
end
local auth_key = ngx.var.arg_auth_key
if not auth_key then
    auth_err("no uri arg auth_key")
end
local timestamp = string.sub(ngx.var.arg_auth_key, 1, 10)
if ngx.time() > tonumber(timestamp) then
    auth_err("timestamp expired")
end
local auth_key = timestamp.."-0-0-"..ngx.md5(ngx.var.uri.."-"..timestamp.."-0-0-"..ngx.var.auth_secret)
if auth_key ~= ngx.var.arg_auth_key then
    auth_err("auth_key error")
end
```
m3u8.auth.conf
```
variables_hash_max_size 2048;
variables_hash_bucket_size 1024;

log_format json '{"@timestamp":"$time_iso8601",'
    '"server_name": "$server_name",'
    '"request_uri": "$request_uri",'
    '"request_method": "$request_method",'
    '"request_time": $request_time,'
    '"remote_addr": "$remote_addr",'
    '"remote_port": "$remote_port",'
    '"http_user_agent": "$http_user_agent",'
    '"http_referer": "$http_referer",'
    '"http_x_forwarded_for": "$http_x_forwarded_for",'
    '"body_bytes_sent": "$body_bytes_sent",'
    '"upstream_status": "$upstream_status",'
    '"upstream_addr": "$upstream_addr",'
    '"upstream_response_time": "$upstream_response_time",'
    '"status": "$status"}';

upstream titan_vod {
    server aes.nnnpx.com;
}

map $subdomain $titan_img_domain {
    default "img1.nnnpx.com";
    ~^ssl-img "img2.nnnpx.com";
}
map $subdomain $titan_vod_domain {
#    default "video1.nnnpx.com";
    default "aes.nnnpx.com";
    ~^ssl-vod "video2.nnnpx.com";
    ~^vod "aes.nnnpx.com";
    ~^json "json.nnnpx.com";
}

server {
    listen       80 ;
    server_name  ~^(?<subdomain>.+\.)?shgongz\.com$;
    error_log    logs/8.129.111.126.error.log;
    access_log   logs/8.129.111.126.log json;

    resolver 8.8.8.8;
    proxy_set_header Host $proxy_host;

    types {
        application/vnd.apple.mpegurl m3u8;
        video/mp2t ts;
    }

    location = /favicon.ico {
        log_not_found off;
        log_subrequest off;
    }

    location ~.*\.(jpg|jpeg|png|gif|bmp|webp|tif|ico|svg|svga)$ {
        proxy_pass http://$titan_img_domain;
    }

    location ~.*\.m3u8$ {
        set $auth_secret "rwwkz8oV132t";
        set $auth_expires 25200;
        rewrite ^/[a-z0-9/]+(/dockbucket/.*) $1 break;
        proxy_set_header Accept-Encoding "";
        proxy_pass http://$titan_vod_domain;
        header_filter_by_lua 'ngx.header.content_length = nil';
        body_filter_by_lua_file /home/software/openresty/nginx/conf/conf.d/m3u8.body.filter2.lua;
    }

    location ~.*\.(key|ts)$ {
        rewrite ^/[a-z0-9/]+(/dockbucket/.*) $1 break;
        proxy_pass http://$titan_vod_domain;
    }

    location / {
        proxy_pass http://origin.nnnpx.com;
    }
}

server {
    listen       80 ;
    server_name  8.129.111.126 test.shgongz.com auth-vod.shgongz.com;
    error_log    logs/test.error.log;
    access_log   logs/test.log json;

    resolver 8.8.8.8;
    proxy_set_header Host $proxy_host;

    location = /favicon.ico {
        log_not_found off;
        log_subrequest off;
    }

    location ~.*\.m3u8$ {
        #accesskey off;
        #accesskey_hashmethod md5;
        #accesskey_arg "auth_key";
        #accesskey_signature "123456$uri$arg_expires";
        #sub_filter encryption.key "encryption.key?key=$arg_auth_key&expires=$arg_expires";
        #sub_filter .ts ".ts?key=$arg_auth_key&expires=$arg_expires";
        #sub_filter_once off;
        #sub_filter_types application/vnd.apple.mpegurl;

        set $auth_secret "rwwkz8oV132t";
        set $auth_expires 25200;
        rewrite ^/[a-z0-9/]+(/dockbucket/.*) $1 break;
        proxy_set_header Accept-Encoding "";
        proxy_pass http://$titan_vod_domain;
#        access_by_lua_file /home/software/openresty/nginx/conf/conf.d/m3u8.access.lua;
#        proxy_pass_request_headers off;
#        content_by_lua_file /home/software/openresty/nginx/conf/conf.d/m3u8.content.lua;
        header_filter_by_lua 'ngx.header.content_length = nil';
        body_filter_by_lua_file /home/software/openresty/nginx/conf/conf.d/m3u8.body.filter2.lua;
    }

    location ~ /titan-m3u8 {
        internal;
        proxy_pass http://titan_vod$request_uri;
        proxy_redirect off;
    }

    location ~.*\.(key|ts)$ {
#        rewrite_by_lua_block {
#            local uri = ngx.re.sub(ngx.var.uri, "^/[a-z0-9/]+(/dockbucket/)", "/dockbucket/", "jo")
#            ngx.req.set_uri(uri)
#        }
        rewrite ^/[a-z0-9/]+(/dockbucket/.*) $1 break;
        proxy_pass http://$titan_vod_domain;
    }

    location / {
        proxy_pass http://$titan_vod_domain;
    }
}
```
m3u8.body.filter
```
local chunk, eof = ngx.arg[1], ngx.arg[2]
local buffered = ngx.ctx.buffered
if not buffered then
    buffered = {}
    ngx.ctx.buffered = buffered
end
if chunk ~= "" and not ngx.is_subrequest then
    buffered[#buffered + 1] = chunk
    ngx.arg[1] = nil
end
if eof then
    local whole = table.concat(buffered)
    ngx.ctx.buffered = nil
    local auth_err = function(err)
        ngx.log(ngx.ERR, "rewrite m3u8 file error: ", err)
        return ngx.exit(403)
    end
    local mp4, err = ngx.re.sub(ngx.var.uri, "index.m3u8", "", "jo")
    -- ngx.log(ngx.ERR, "######## "..mp4.." ########")
    local auth_uri = function(str)
        local uri = mp4..str[0]
        local timestamp = ngx.time() + ngx.var.auth_expires
        local auth_key = timestamp.."-0-0-"..ngx.md5(uri.."-"..timestamp.."-0-0-"..ngx.var.auth_secret)
        return str[0].."?auth_key="..auth_key
    end
    local m3u8, err = ngx.re.gsub(whole, "^seg-[a-z0-9-]+.ts$", auth_uri, "mjo")
    if not m3u8 then
        auth_err(err)
    else
        m3u8, err = ngx.re.sub(m3u8, "encryption.key", auth_uri, "jo")
        if not m3u8 then
            auth_err(err)
        end
    end
    ngx.arg[1] = m3u8
end

```
m3u8.body.filter2
```
local chunk, eof = ngx.arg[1], ngx.arg[2]
local buffered = ngx.ctx.buffered
if not buffered then
    buffered = {}
    ngx.ctx.buffered = buffered
end
if chunk ~= "" and not ngx.is_subrequest then
    buffered[#buffered + 1] = chunk
    ngx.arg[1] = nil
end
if eof then
    local whole = table.concat(buffered)
    ngx.ctx.buffered = nil
    local auth_err = function(err)
        ngx.log(ngx.ERR, "rewrite m3u8 file error: ", err)
        return ngx.exit(403)
    end
    local host = ngx.var.scheme.."://"..ngx.var.host..":"..ngx.var.server_port
    local mp4, err = ngx.re.sub(ngx.var.uri, "index.m3u8", "", "jo")
    local timestamp = os.date("%Y%m%d%H%M", ngx.time() + ngx.var.auth_expires)
    -- ngx.log(ngx.ERR, "######## "..mp4)
    local auth_uri = function(str)
        -- ngx.log(ngx.ERR, "######## "..str[0].." ########")
        local uri = mp4..str[0]
        local md5 = ngx.md5(ngx.var.auth_secret..timestamp..uri)
        return host.."/"..timestamp.."/"..md5..uri
    end
    local m3u8, err = ngx.re.sub(whole, "encryption.key", auth_uri, "jo")
    if not m3u8 then
        auth_err(err)
    else
        m3u8, err = ngx.re.gsub(m3u8, "^seg-[a-z0-9-]+.ts$", auth_uri, "mjo")
        if not m3u8 then
            auth_err(err)
        end
        ngx.arg[1] = m3u8
    end
end
```

m3u8.content.lua
```
ngx.req.set_header("Accept-Encoding", "")
local res = ngx.location.capture("/titan-m3u8")
if not res then
    ngx.log(ngx.ERR, tostring(res.status))
    ngx.say("request error :", err)
    return
end
ngx.status = res.status
for k, v in pairs(res.header) do
    if k ~= "Transfer-Encoding" and k ~= "Connection" then
        ngx.header[k] = v
    end
end

if not res.body then
    return
end

local auth_uri = function(str)
    ngx.log(ngx.ERR, "######## "..str[0].." ########")
    local uri = ngx.re.sub(ngx.var.uri, "index.m3u8", "", "jo")..str[0]
    local timestamp = ngx.time() + ngx.var.auth_expires
    local auth_key = timestamp.."-0-0-"..ngx.md5(uri.."-"..timestamp.."-0-0-"..ngx.var.auth_secret)
    return str[0].."?auth_key="..auth_key
end
local m3u8, err = ngx.re.gsub(res.body, "^(seg-)[a-z0-9-]+(.ts)$", auth_uri, "mjo")
if not m3u8 then
    auth_err(err)
else
    m3u8, err = ngx.re.sub(m3u8, "encryption.key", auth_uri, "jo")
    if not m3u8 then
        auth_err(err)
    end
end
ngx.say(m3u8)
```
status.conf
```
vhost_traffic_status_zone;
vhost_traffic_status_filter_by_host on;

server {
    listen 80;
    server_name  status.testout.guangtou666.com;
    access_log   logs/status.testout.guangtou666.com.log  main;

    location / {
        vhost_traffic_status_display;
        vhost_traffic_status_display_format html;
    }
}
```
# Nginx配置Nacos反向代理
```
server {
    listen       80;
    server_name  nacos.xxx.com;
    access_log   logs/nacos.log  main;

    proxy_http_version   1.1;
    proxy_set_header     Upgrade            $http_upgrade;
    proxy_set_header     Connection         'upgrade';
    proxy_set_header     Host               $host;
    proxy_set_header     X-Real-IP          $remote_addr;
    proxy_set_header     X-Forwarded-For    $proxy_add_x_forwarded_for;
    proxy_set_header     X-Forwarded-Host   $server_name;
    proxy_set_header     X-Forwarded-Proto  $scheme;
    proxy_cache_bypass   $http_upgrade;

    location / {
        proxy_pass http://172.17.140.232:8848;
    }

    location =/ {
        rewrite / /nacos redirect;
    }
}
```
