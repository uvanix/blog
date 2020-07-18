参考：https://docs.phpmyadmin.net/zh_CN/latest/require.html#web-server
```
官网下载最新版本：
wget https://files.phpmyadmin.net/phpMyAdmin/4.9.4/phpMyAdmin-4.9.4-all-languages.zip
解压
unzip phpMyAdmin-4.9.4-all-languages.zip && mv phpMyAdmin-4.9.4-all-languages phpMyAdmin
 
修改配置文件
cd phpMyAdmin
cp config.sample.inc.php config.inc.php
vim config.inc.php
使用cookie认证默认中文显示
$cfg['blowfish_secret'] = '0zT9Zk3q0zT9Zk3q0zT9Zk3q0zT9Zk3q';
$cfg['Servers'][$i]['host'] = '127.0.0.1';
$cfg['DefaultLang'] = 'zh';
配置连接多实例mysql
使用 sed -i '22,34s#^#//#g' config.inc.php 命令注释掉之前相关行并编辑这个文件，添加一个$hosts数组和一个for循环
$hosts = array(
'1'=>array('host'=>'127.0.0.1','user'=>'root','password'=>'123456','port'=>3306),
'2'=>array('host'=>'127.0.0.1','user'=>'root','password'=>'123456','port'=>4000)                    
);                                                                                                           
                                                                                                               
for($i=1,$j=count($hosts);$i<=$j;$i++){                                                                      
    /* Authentication type */                                                                                
    $cfg['Servers'][$i]['auth_type'] = 'cookie';                                                             
    /* Server parameters */                                                                                  
    $cfg['Servers'][$i]['host'] = $hosts[$i]['host'];   //修改host                                           
    $cfg['Servers'][$i]['port'] = $hosts[$i]['port'];                                                        
    $cfg['Servers'][$i]['connect_type'] = 'tcp';                                                             
    $cfg['Servers'][$i]['compress'] = false;                                                                 
    /* Select mysqli if your server has it */                                                                
    $cfg['Servers'][$i]['extension'] = 'mysqli';                                                             
    $cfg['Servers'][$i]['AllowNoPassword'] = true;                                                           
    $cfg['Servers'][$i]['user'] = $hosts[$i]['user'];  //修改用户名                                          
    $cfg['Servers'][$i]['password'] = $hosts[$i]['password']; //密码                                         
    /* rajk - for blobstreaming */                                                                           
    $cfg['Servers'][$i]['bs_garbage_threshold'] = 50;                                                        
    $cfg['Servers'][$i]['bs_repository_threshold'] = '32M';                                                  
    $cfg['Servers'][$i]['bs_temp_blob_timeout'] = 600;                                                       
    $cfg['Servers'][$i]['bs_temp_log_threshold'] = '32M';                                                    
}
更改完保存后刷新页面就可以选择不通服务器了
 
设置临时目录加快运行
cd phpMyAdmin
mkdir tmp
chmod 777 tmp
 
 
修改session.save_path约1264行
vim /etc/php.ini
session.save_path = "/var/lib/php/session"
 
查看session目录权限：ll -d /var/lib/php/session/
修改session目录用户为nginx
chown root.nginx /var/lib/php/session
 
 
nginx配置
server {
    listen               *:6666;
    server_name          localhost;
    access_log           logs/localhost.log;
    error_log            logs/localhost.error.log;
 
    charset               utf-8;
 
    location / {
        root   html;
        index  index.html index.htm;
    }
 
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   html;
    }
    location /phpMyAdmin/ {
        index index.php;
        root /home/software;
        location ~ \.php$ {
            fastcgi_pass   127.0.0.1:9000;
            fastcgi_index  index.php;
            fastcgi_param  SCRIPT_FILENAME  /home/software/$fastcgi_script_name;
            include        fastcgi_params;
        }
    }
}
```