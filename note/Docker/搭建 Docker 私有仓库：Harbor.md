https://github.com/goharbor/harbor/blob/master/docs/installation_guide.md

安装 Harbor
# 1. 下载安装文件（可以在指定目录下载）
wget https://storage.googleapis.com/harbor-releases/harbor-online-installer-v1.5.2.tgz

# 2. 解压下载的文件
tar xvf harbor-online-installer-v1.5.2.tgz
配置 Harbor
1. 修改Harbor的配置文件
cd harbor
vim harbor.cfg
内容如下：
# hostname设置访问地址，可以使用ip、域名，不可以设置为127.0.0.1或localhost
hostname = 192.168.1.121

# 访问协议，默认是http，也可以设置https，如果设置https，则nginx ssl需要设置on
ui_url_protocol = http

# mysql数据库root用户默认密码root123，实际使用时修改下
db_password = root@1234
2. 配置 mirror proxy

cd harbor

vim ./common/config/registry/config.yml

在末尾添加proxy配置：

proxy:

  remoteurl: https://registry-1.docker.io

重新启动构建harbor

docker-compose down -v

docker-compose up -d

3. 配置docker proxy

vim /usr/lib/systemd/system/docker.service

修改如下：

ExecStart=/usr/bin/dockerd -g /data/docker --insecure-registry 192.168.1.121:8000 --registry-mirror=http://192.168.1.121:8000

重启docker daemon和docker service

systemctl daemon-reload

systemctl restart docker

启动 Harbor
# 1.在当前安装目录下
./install.sh
