## MAC下brew安装
```
brew install go

安装好后查看版本：go version

从安装提示中可以看出需要设置GOPATH和GOROOT的环境变量，以及设置PATH

查看环境变量：go env

配置修改：vim ~/.bash_profile 或者 vim ~/.zshrc （使用zsh）添加以下配置

export GOROOT=/usr/local/opt/go/libexec

export GOPATH=$HOME/go

export PATH=$PATH:$GOROOT/bin:$GOPATH/bin

保存修改刷新：source /.zshrc ( ~/.bash_profile)

在使用命令 go env 查看环境变量是否生效（注意最好关闭所有终端重新打开防止未刷新终端，此时gopath默认目录为$HOME/go）

说明：

GOROOT：golang的安装目录

GOPATH：go的工作空间，在GOPATH下会有三个目录：src，bin，pkg。

src：存放源码（在不设置go mod的情况下，go get的包放至这里，你自己的项目源码也可以放在这里）；
bin：编译后的可执行文件放置的位置，如果在你项目go install，生成的可执行文件会放在此处；
pkg：存放编译时产生的文件（.a文件）（非main函数的文件在go install后生成）
并根据平台进行归档，比如，windows64下产生的文件放在windows_amd64目录下，通常情况下，我们不会太关心这些文件。不过在go module包管理方式下，pkg目录变得受人关注了， 依赖的第三方包被下载到了$GOPATH/pkg/mod路径下，而不是src文件夹。
PATH：环境变量，需要$GOROOT/bin目录加入到path路径下，生成可执行文件就可以直接运行了。
```

## Linux下安装

### 1. install-go.sh
```
#!/bin/bash
# for linux
set -x
set -e
 
# default version: 1.10.3
VERSION=$1
VERSION=${VERSION:-1.10.3}
GOROOT="/usr/local/go"
GOPATH=$HOME/gopath
 
# download and install
wget https://dl.google.com/go/go${VERSION}.linux-amd64.tar.gz
tar -C /usr/local -xzf go${VERSION}.linux-amd64.tar.gz
 
# set golang env
cat >> $HOME/.bashrc << EOF
# Golang env
export GOROOT=/usr/local/go
export GOPATH=\$HOME/go
export PATH=\$PATH:\$GOROOT/bin:\$GOPATH/bin
EOF
 
source $HOME/.bashrc
 
# mkdir gopath
mkdir -p $GOPATH/src $GOPATH/pkg $GOPATH/bin
```

### 2. 安装
```
chmod +x install-go.sh
./install-go.sh 1.10.3
```

更多版本号可参考：https://golang.org/dl/
参考：https://www.huweihuang.com/golang-notes/introduction/install.html


