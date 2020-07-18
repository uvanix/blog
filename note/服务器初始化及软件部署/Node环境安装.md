首先确保安装了git

yum -y install git

安装nvm

curl https://raw.githubusercontent.com/creationix/nvm/v0.30.2/install.sh | bash


参考：https://github.com/nvm-sh/nvm


*重启终端

nvm --version
v0.30.2
nvm help 

升级nvm,前往~/.nvm，从git服务器拉去最新的版本
```
[root@localhost .nvm]# nvm --version
0.30.2
[root@localhost .nvm]# git fetch -p
[root@localhost .nvm]# git rev-list --tags --max-count=1
0a95e77000515c1156be593642dd4e452f2f098e
[root@localhost .nvm]# git describe --tags 0a95e77000515c1156be593642dd4e452f2f098e
v0.33.2
[root@localhost .nvm]# git describe --abbrev=0 --tags
v0.33.2
[root@localhost .nvm]# git checkout $(git describe --tags `git rev-list --tags --max-count=1`)
之前的 HEAD 位置是 7f3145b... [New] add support for `$NVM_DIR/default-packages` file
HEAD 目前位于 0a95e77... v0.33.2
[root@localhost .nvm]# source ~/.nvm/nvm.sh
[root@localhost .nvm]# nvm --version
0.33.2
```

罗列下载相应版本的node
```
#不加node，无法罗列版本。。。。
nvm ls-remote node
 
......
         v6.9.5   (LTS: Boron)
->      v6.10.0   (LTS: Boron)
        v6.10.1   (LTS: Boron)
        v6.10.2   (LTS: Boron)
        v6.10.3   (Latest LTS: Boron)
   .....
   .....
        v7.10.0
 
nvm install v6.10.0
```



查看node版本

node -v

npm -v



安装pm2

npm install pm2 -g



