安装参考：https://pkg.jenkins.io/redhat-stable/  https://ken.io/note/centos7-jenkins-install-tutorial



安装前确认已有java环境

添加jenkins yum 源

```
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
```

安装jenkins
```
yum install jenkins
```

配置java可选路径

因为Jenkins默认的java可选路径不包含我们部署的jdk路径，所以这里要配置一下，不然Jenkins服务会启动失败

#修改jenkins启动脚本
sudo vi /etc/init.d/jenkins

#修改candidates增加java可选路径：/opt/jdk1.8.0_131/bin/java
```
candidates="

/etc/alternatives/java

/usr/lib/jvm/java-1.8.0/bin/java

/usr/lib/jvm/jre-1.8.0/bin/java

/usr/lib/jvm/java-1.7.0/bin/java

/usr/lib/jvm/jre-1.7.0/bin/java

/usr/bin/java

/opt/jdk1.8.0_131/bin/java

"
```

启动Jenkins并设置Jenkins开机启动
#重载服务（由于前面修改了Jenkins启动脚本）
sudo systemctl daemon-reload

#启动Jenkins服务
sudo systemctl start jenkins

#将Jenkins服务设置为开机启动
#由于Jenkins不是Native Service，所以需要用chkconfig命令而不是systemctl命令
sudo /sbin/chkconfig jenkins on

PS:
查找jenkins安装路径：rpm -ql jenkins

jenkins相关目录释义：
- (1) /usr/lib/jenkins/：jenkins安装目录，war包会放在这里。
- (2) /etc/sysconfig/jenkins：jenkins配置文件，“端口”，“JENKINS_HOME”等都可以在这里配置。
- (3) /var/lib/jenkins/：默认的JENKINS_HOME。
- (4) /var/log/jenkins/jenkins.log：jenkins日志文件。



jenkins可能的权限问题可以修改为root启动

vim /etc/sysconfig/jenkins #修改配置 $JENKINS_USER="root"


