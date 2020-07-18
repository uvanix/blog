上传jdk压缩包到服务器

scp jdk1.8.0_131.tar.gz root@xxx.xxx.xxx:/opt

解压jdk

tar -zxvf jdk1.8.0_131.tar.gz

配置环境变量

vim /etc/profile

添加

# java env

export JAVA_HOME=/opt/jdk1.8.0_131

export CLASSPATH=.:${JAVA_HOME}/jre/lib/rt.jar:${JAVA_HOME}/lib/dt.jar:${JAVA_HOME}/lib/tools.jar

export PATH=$PATH:${JAVA_HOME}/bin

