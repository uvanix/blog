## 设置静态IP
参考修改配置文件：vi /etc/sysconfig/network-scripts/ifcfg-ens160
```
TYPE=Ethernet
BOOTPROTO=static
DEFROUTE=yes
IPV4_FAILURE_FATAL=yes
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens160
UUID=0c557558-aceb-4646-b2a7-24a96014e55b
DEVICE=ens160
ONBOOT=yes
IPADDR=172.16.10.27
PREFIX=24
GATEWAY=172.16.10.1
DNS1=8.8.8.8
IPV6_PEERDNS=yes
IPV6_PEERROUTES=yes
IPV6_PRIVACY=no
```

## 设置hostname
```
hostnamectl --static set-hostname {hostname}
```

## 磁盘分区挂载（扩容/根目录）
### 小于2T可以使用fdisk命令分区，现对新加的磁盘进行分区（只创建一个主分区）

使用命令 df -h 查看磁盘容量 lsblk 查看磁盘分区情况

查看磁盘：fdisk -l 可以看到新加的磁盘 /dev/sdb

开始分区：fdisk /dev/sdb

新建分区：输入n

创建主分区：输入p

分区号默认1：回车

起始扇区：回车

结束扇区：回车

 查看分区：输入p

修改扇区类型：输入t

选择分区（默认已选择分区一，因为只有一个分区）

输入Hex代码：8e (lvm)

保存退出：输入w回车

强制系统刷新磁盘分区命令：partprobe /dev/sdb

### 大于2T只能使用parted命令分区

开始分区：parted /dev/sdb

设置gpt格式：mklabel gpt

设置扇区格式及大小：mkpart primary 0 -1 （-1表示结束位置）

设置分区类型为lvm：set 1 lvm on

查看分区：print

保存退出：quit

强制系统刷新磁盘分区命令：partprobe /dev/sdb

输入 lsblk 或者 fdisk -l 或者 cat /proc/partitions 查看磁盘分区信息

格式化分区：mkfs.xfs /dev/sdb1 （centos7默认使用xfs此步骤可省略）

接下来创建物理卷pv：pvcreate /dev/sdb1 -y  查看物理卷：pvs

查看卷组信息：vgs 将刚创建的物理卷加入 cl 卷组（cl卷组挂载为根目录）命令：vgextend cl /dev/sdb1 再次 vgs 查看卷组信息发现空间已变大

查看逻辑卷信息：lvs 发现有连个lv一个root 一个swap 我们需要对root逻辑卷进行扩容 参考命令：lvextend -l +100%FREE /dev/cl/root 再次 lvs 查看逻辑卷 root 空间已变大

文件系统刷新：ext3文件系统命令 resize2fs /dev/mapper/cl-root    xfs文件系统使用下面命令 xfs_growfs /dev/mapper/cl-root （centos7默认使用xfs）

再次使用 df -Th 命令查看/根目录容量已增加


### 增加交换分区

首先查看当前系统swap是否存在swap分区，以下命令会显示swap 分区大小，为0表示没有分区。

一般设置为内存*2，比如32g内存那么交换分区=64g = 65536m

free -h


1.添加swap分区
dd if=/dev/zero of=/root/.swapfile bs=1M count=8192

if(即输入文件,input file)，dev/zero 是Linux的一种特殊字符输入设备，用来创建一个指定长度用于初始化的空文件。
of(即输出文件,output file)。 /data/swapfile 是 swap 文件地址。
bs=1M ：单位数据块同时读写块字节大小为1M(默认单位为字节)。
count=8192K ：数据块数量为8192*1024。
计算出swap分区的容量为：1KB*8192*1024=4G。

转换为swap分区：
mkswap /root/.swapfile
设置权限为root可操作
chmod -R 0600 /root/.swapfile
挂载并激活分区：
swapon /root/.swapfile
设置开机自动挂载该分区：
vi /etc/fstab 

/root/.swapfile swap swap defaults 0 0

或者
UUDI=swapfile的UUID swap swap defaults 0 0

2.删除某swap分区

先停止正在使用swap分区：
swapoff /root/.swapfile
删除swap分区文件
rm -rf /root/.swapfile
删除 /etc/fstab 中的配置
UUDI=swapfile的UUID swap swap defaults 0 0

3.更改Swap配置，swappiness值越高系统对swap分区的使用优先级越高，默认为30.

查看当前的swappiness数值：
cat /proc/sys/vm/swappiness
修改swappiness值，这里以10为例。
sysctl vm.swappiness=10
永久生效
echo "vm.swappiness = 10" >> /etc/sysctl.conf

## 关闭SELINUX

查看selinux：getenforce

临时关闭：setenforce 0

永久关闭：vi /etc/selinux/config （将SELINUX=enforcing改为SELINUX=disabled 设置后需要重启才能生效）

## 修改安全限制

修改 vi /etc/sysctl.conf 文件，末尾增加配置 vm.max_map_count=262144 然后执行命令生效 sysctl -p

修改 vi /etc/security/limits.conf 文件在最后添加如下内容

```
*           soft  core   unlimited
*           hard  core   unlimited
*           soft  fsize  unlimited
*           hard  fsize  unlimited
*           soft  data   unlimited
*           hard  data   unlimited
*           soft  nproc  524288
*           hard  nproc  524288
*           soft  stack  unlimited
*           hard  stack  unlimited
*           soft  nofile  1048576
*           hard  nofile  1048576
```

```
cat >>/etc/security/limits.conf<<EOF
*           soft  core   unlimited
*           hard  core   unlimited
*           soft  fsize  unlimited
*           hard  fsize  unlimited
*           soft  data   unlimited
*           hard  data   unlimited
*           soft  nproc  524288
*           hard  nproc  524288
*           soft  stack  unlimited
*           hard  stack  unlimited
*           soft  nofile  1048576
*           hard  nofile  1048576
EOF
```

修改后需重启系统生效

ulimit调优：https://www.cnblogs.com/sunsky303/p/8359592.html


## 关闭防火墙

```
systemctl stop firewalld && systemctl disable firewalld

yum -y install iptables-services
```

## 升级内核安装常用工具

yum -y –enablerepo=elrepo-kernel install kernel-ml

centos7内核升级完毕后，还需要我们修改内核的启动顺序，使用如下命令：

vim /etc/default/grub

GRUB_DEFAULT=0

接下来还需要运行grub2-mkconfig 命令来重新创建内核配置，如下：

grub2-mkconfig -o /boot/grub2/grub.cfg

重启系统，查看内核版本

shutdown -r now

uname -r



yum -y update upgrade

yum clean all
yum makecache

yum -y install vim wget psmisc net-tools nc


## bash常用配置

history 添加时间：vi /etc/profile 最后添加如下内容

export HISTTIMEFORMAT='%F %T '

然后刷新环境变量 source /etc/profile


修改默认的shell为zsh并安装常用插件（参考：https://segmentfault.com/a/1190000013612471）

查看系统有哪些shell： cat /etc/shells   

当前使用的shell：echo $SHELL

安装 zsh ： yum -y install zsh

设置zsh为默认shell：chsh -s /bin/zsh

注销重新登录后生效

安装oh-my-zsh：
```
# 安装oh-my-zsh前需要安装git：
yum -y install git
sh -c "$(curl -fsSL https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh)" （命令来自官网：https://ohmyz.sh/）
```

安装zsh-autosuggestions和zsh-syntax-highlighting语法高亮插件：
```
git clone git://github.com/zsh-users/zsh-autosuggestions $ZSH_CUSTOM/plugins/zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
```

插件名称加入 oh-my-zsh 插件列表
```
# 打开 zsh 配置文件
vim ~/.zshrc
 
# 把插件名称加入插件列表
plugins=(
  git
  zsh-autosuggestions
  zsh-syntax-highlighting
)
# 刷新配置
source ~/.zshrc
```

修改主题为 candy：

编辑~/.zshrc文件，将ZSH_THEME="candy",即将主题修改为candy



注意使用zsh后会出现通配符问题：比如安装 php时 yum install php70w*

zsh: no matches found: php70w*

这时需要修改 ~/.zshrc 文件在最后加入 setopt no_nomatch, 然后进行source .zshrc命令


centos 设置 ssh 代理上网

```
ssh -qTfnN -D 7070 dev29

vim /etc/profile

export http_proxy=socks5://127.0.0.1:7070

export https_proxy=socks5://127.0.0.1:7070

export all_proxy=socks5://127.0.0.1:7070

export no_proxy=*.mhteam.com,172.16.*,192.168.*,localhost,127.0.0.1
```

jvm 设置socks代理 -DsocksProxyHost=localhost -DsocksProxyPort=7070



Linux释放操作系统缓存：echo 3 > /proc/sys/vm/drop_caches

## DNS修改

在CentOS 7下，手工设置 /etc/resolv.conf 里的DNS，过了一会，发现被系统重新覆盖或者清除了。和CentOS 6下
的设置DNS方法不同，有几种方式：
1、使用全新的命令行工具 nmcli 来设置

#显示当前网络连接
```
#nmcli connection show
NAME UUID TYPE DEVICE

ens160  0c557558-aceb-4646-b2a7-24a96014e55b  802-3-ethernet  ens160
```

#修改当前网络连接对应的DNS服务器，这里的网络连接可以用名称或者UUID来标识
```
#

nmcli con mod ens160 ipv4.dns "114.114.114.114 8.8.8.8"

# 

nmcli con mod ens160 ipv4.dns "172.16.10.25" && nmcli con up ens160

```

#将dns配置生效
```
#

nmcli con up ens160
```

2、使用传统方法，手工修改 /etc/resolv.conf
修改 /etc/NetworkManager/NetworkManager.conf 文件，在main部分添加 “dns=none” 选项：
```
[main]
plugins=ifcfg­rh
dns=none
NetworkManager重新装载上面修改的配置
# systemctl restart NetworkManager.service
手工修改 /etc/resolv.conf
nameserver 114.114.114.114
nameserver 8.8.8.8
详细参见：
# man NetworkManager.conf
# man nmcli
```
