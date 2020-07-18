php 7.0 +

1. 若之前安装过其他版本PHP，先删除
# yum remove php*

2. rpm安装PHP7相应的yum源
CentOS/RHEL 7.x:
# rpm -Uvh https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
# rpm -Uvh https://mirror.webtatic.com/yum/el7/webtatic-release.rpm
CentOS/RHEL 6.x:
# rpm -Uvh https://mirror.webtatic.com/yum/el6/latest.rpm

3. yum安装PHP7
yum -y install php70w* --skip-broken //意为安装全部插件

4. vi /etc/php-fpm.d/www.conf
修改 user 和 group 为nginx

5. 添加nginx 用户组
groupadd nginx
useradd nginx -g nginx -s /sbin/nologin -M

6. 配置nginx.conf

7. chkconfig php-fpm on

或者php-fpm -R

8. service php-fpm restart

9. nginx -s reload



需要设置:用户组

chown nginx:nginx dir
安装Composer
# 下载composer.phar

curl -sS https://getcomposer.org/installer | php

# 把composer.phar移动到环境下让其变成可执行

mv composer.phar /usr/local/bin/composer

# 授权

chmod a+x /usr/local/bin/composer

# 测试

composer -V

# 更新

composer self-update


