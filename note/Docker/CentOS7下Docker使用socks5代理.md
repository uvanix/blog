1.创建docker服务插件目录

sudo mkdir -p /etc/systemd/system/docker.service.d

2.创建一个名为http-proxy.conf的文件

sudo touch /etc/systemd/system/docker.service.d/http-proxy.conf

3.编辑http-proxy.conf的文件

sudo vim /etc/systemd/system/docker.service.d/http-proxy.conf

4.写入内容(将代理ip和代理端口修改成你自己的)

[Service]

Environment="HTTP_PROXY=socks5://127.0.0.1:7070"

Environment="NO_PROXY=localhost,127.0.0.0/8,192.168.1.0/24"


5.重新加载服务程序的配置文件

sudo systemctl daemon-reload

6.重启docker

sudo systemctl restart docker

7.验证是否配置成功

systemctl show --property=Environment docker
