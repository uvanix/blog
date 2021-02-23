官网下载源码包：https://openresty.org/cn/download.html


cd /opt

curl -o openresty-1.13.6.2.tar.gz  https://openresty.org/download/openresty-1.13.6.2.tar.gz

开始安装

解压：tar -zxvf openresty-1.13.6.2.tar.gz

创建安装目录：mkdir /opt/openresty

安装编译openresty需要的开发库：yum install pcre-devel openssl-devel gcc curl -y

进入源码目录开始编译：

```
cd openresty-1.13.6.2

./configure --prefix=/opt/openresty --with-luajit \
--with-select_module \
--with-poll_module \
--with-file-aio \
--with-http_ssl_module \
--with-http_realip_module \
--with-http_gzip_static_module \
--with-http_secure_link_module \
--with-http_stub_status_module \
--with-http_sub_module \
--with-http_mp4_module \
--with-http_flv_module \
--with-threads \
--with-pcre \
--with-stream \
--with-stream_ssl_module \
以下参数用于nginx监控，防盗链和mp4在线切片
--add-module=../nginx-module-vts-0.1.18 \
--add-module=../nginx-accesskey-2.0.5 \
--add-module=../nginx-vod-module-1.25 \
--with-cc-opt="-O3" \
gmake && gmake install

```

添加开机启动

```
cat >>/usr/lib/systemd/system/nginx.service<<EOF
# OpenResty 安装在 /opt/openresty
# 该文件应该放在 /usr/lib/systemd/system/nginx.service
# 放好后请运行：
# - systemctl enable nginx.service
# - systemctl start nginx.service
# 然后之后就可以用 service nginx [start/reload/stop] 等命令了
 
[Unit]
Description=The nginx HTTP and reverse proxy server
After=syslog.target network.target remote-fs.target nss-lookup.target
 
[Service]
Type=forking
PIDFile=/opt/openresty/nginx/logs/nginx.pid
ExecStartPre=/opt/openresty/nginx/sbin/nginx -t
ExecStart=/opt/openresty/nginx/sbin/nginx
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true
 
[Install]
WantedBy=multi-user.target
EOF
```


添加环境变量（此方法拷贝时$符号将被替换）

```
cat >>/etc/profile<<EOF
 
# openresty env
export OPENRESTY_HOME=/opt/openresty
export PATH=$PATH:${OPENRESTY_HOME}/bin
EOF
 
source /etc/profile
```

