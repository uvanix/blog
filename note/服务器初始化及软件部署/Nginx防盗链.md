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

