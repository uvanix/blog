## 1、安装依赖
```
yum -y install curl policycoreutils openssh-server openssh-clients postfix
yum install -y policycoreutils-python
```

## 2、下载rpm包
```
wget https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7/gitlab-ce-12.4.2-ce.0.el7.x86_64.rpm

可访问为此地址查看所需版本

https://mirror.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7/
```

## 3、安装
```
rpm -ivh gitlab-ce-12.4.2-ce.0.el7.x86_64.rpm
```

## 4、修改访问地址和默认端口
```
vim /etc/gitlab/gitlab.rb(此为访问git的地址或是域名）

external_url 'http://gitlab.example.com'
```

gitlab.rb 修改
配置文件在 /opt/gitlab/etc/gitlab.rb 。这个文件用于gitlab如何调用80和8080的服务等。
```
## Advanced settings
unicorn['listen'] = '127.0.0.1'
unicorn['port'] = 8082
nginx['listen_addresses'] = ['*']
nginx['listen_port'] = 82 # override only if you use a reverse proxy: https://gitlab.com/gitlab-org/omnibus-gitlab/blob/master/doc/settings/nginx.md#setting-the-nginx-listen-port
```

gitlab-rails 修改
配置文件 /var/opt/gitlab/gitlab-rails/etc/unicorn.rb
```
# What ports/sockets to listen on, and what options for them.
#listen "127.0.0.1:8080", :tcp_nopush => true
listen "127.0.0.1:8082", :tcp_nopush => true
listen "/var/opt/gitlab/gitlab-rails/sockets/gitlab.socket", :backlog => 1024
```

gitlab nginx 修改
配置文件 /var/opt/gitlab/nginx/conf/gitlab-http.conf。这个文件是gitlab内置的nginx的配置文件，里面可以影响到nginx真实监听端口号。
```
server {
  listen *:82;

  server_name gitlab.123.123.cn;
  server_tokens off; ## Don't show the nginx version number, a security best practice
}
```

## 5、安装
```
gitlab-ctl reconfigure
# 查看版本
cat /opt/gitlab/embedded/service/gitlab-rails/VERSION
```

## 6、启动
```
gitlab-ctl start
```

Nginx反向代理
如果还是想从80端口访问gitlab，我们可以用监听在80端口的nginx做一个反向代理。
```
server {
    listen 80;
    server_name gitlab.example.con;

    location / {
        #rewrite ^(.*) http://127.0.0.1:8082;
        proxy_pass http://127.0.0.1:8082;
    }
}
```

修改上传大小限制
```
git config --global http.postBuffer 524288000

git config --global https.postBuffer 524288000
```
在gitlab.rb中修改nginx配置
```
nginx['enable'] = true

nginx['client_max_body_size'] = '1024m'

nginx['redirect_http_to_https'] = false

nginx['redirect_http_to_https_port'] = 80
```
重载配置文件并重启gitlab
```
gitlab-ctl reconfigure

gitlab-ctl restart
```

### 创建备份
```
$ sudo gitlab-rake gitlab:backup:create
```

执行完备份命令后会在/var/opt/gitlab/backups目录下生成备份后的文件，如1581010075_2020_02_07_12.4.0_gitlab_backup.tar。

1500809139是一个时间戳，从1970年1月1日0时到当前时间的秒数；后面是具体日期及gitlab版本号。

这个压缩包包含Gitlab所有数据（例如：管理员、普通账户以及仓库等等）。

### 从备份恢复

将备份文件拷贝到/var/opt/gitlab/backups下（备份和恢复的GitLab版本尽量保持一致，后文描述了版本不匹配的处理方法）。

停止相关数据连接服务
```
sudo gitlab-ctl stop unicorn
sudo gitlab-ctl stop sidekiq
```

从备份恢复
从指定时间戳的备份恢复（backups目录下有多个备份文件时）：
```
sudo gitlab-rake gitlab:backup:restore BACKUP=1500809139
```

从默认备份恢复（backups目录下只有一个备份文件时）：
```
sudo gitlab-rake gitlab:backup:restore
```

重启gitlab服务并检查恢复数据情况
```
gitlab-ctl restart
gitlab-rake gitlab:check SANITIZE=true
```

### 修改默认备份目录【可选】

你也可以通过修改/etc/gitlab/gitlab.rb来修改默认存放备份文件的目录：
```
gitlab_rails['backup_path'] = '/home/backup'
```

/home/backup修改为你想存放备份的目录即可, 修改完成之后使用gitlab-ctl reconfigure命令重载配置文件即可。 

重装后访问页面出现500或502

在恢复数据时，提示版本不匹配，卸载、指定版本重装后出现500或502错误，网上搜索了很多方法，都不解决问题，最终发现是卸载不彻底引起，完整的卸载方法为：
```
sudo gitlab-ctl stop
sudo apt-get --purge remove gitlab-ce
sudo rm -r /var/opt/gitlab
sudo rm -r /opt/gitlab
sudo rm -r /etc/gitlab
```
