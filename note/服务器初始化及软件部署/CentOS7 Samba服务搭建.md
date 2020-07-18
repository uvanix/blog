```
yum install samba

###创建用户pd和家目录，同时默认创建了用户组pd
# useradd -d /home/ftp/public  -m -s /sbin/nologin public
# useradd -d /home/ftp/pd  -m -s /sbin/nologin  pd
# useradd -d /home/ftp/ui  -m -s /sbin/nologin ui
###创建smb用户，smb用户必须是系统已经存在的用户,执行命令后输入两次密码即创建成功
# pdbedit  -a -u public
# pdbedit  -a -u pd
# pdbedit  -a -u ui

###查看创建的用户
# pdbedit -L


修改目录权限



chown nobody:nobody public



修改配置文件添加用户资源控制

vim /etc/samba/smb.conf


[public]

        comment = public

        path = /data/share/public

        public = yes

        writable = yes

        browseable = yes

        printable = no



[ui]

        comment = ui

        path = /data/share/ui

        public = no

        writable = yes

        browseable = yes

        valid users = ui

        write list = ui

        printable = no

        create mask = 07774

        directory mask = 0777



[pd]

        comment = pd

        path = /data/share/pd

        public = no

        writable = yes

        browseable = yes

        valid users = pd

        write list = pd

        printable = no

        create mask = 07774

        directory mask = 0777


```


