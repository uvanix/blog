客户端连接为mac时mac添加如下配置：
```
sudo vi /etc/ppp/options
plugin L2TP.ppp
l2tpnoipsec
```

客户端连接为ikuai路由器时使用以下脚本安装：
```
#!/bin/sh
 
# Script for automatic setup of an IPsec VPN server on CentOS/RHEL 6, 7 and 8.
# execute as root
# reference https://www.wenjinyu.me/zh/centos-7-build-l2tp-vpn https://github.com/hwdsl2/setup-ipsec-vpn/blob/master/vpnsetup_centos.sh
 
# =====================================================
 
# Define your own values for these variables
# - IPsec pre-shared key, VPN username and password
# - All values MUST be placed inside 'single quotes'
# - DO NOT use these special characters within values: \ " '
 
VPN_IPSEC_PSK=''
VPN_USER=''
VPN_PASSWORD=''
VPN_NET_IFACE=''
VPN_NET_IP=''
 
# =====================================================
 
exiterr() { echo "Error: $1" >&2; exit 1; }
bigecho() { echo; echo "## $1"; echo; }
 
if [ "$(id -u)" != 0 ]; then
  exiterr "Script must be run as root. Try 'sudo sh $0'"
fi
 
 
bigecho "Installing packages required for setup..."
 
# 查看主机是否支持pptp，返回结果为yes就表示通过
modprobe ppp-compress-18 && echo yes
 
# 查看是否开启了TUN
# 有的虚拟机主机需要开启，返回结果为**cat: /dev/net/tun: File descriptor in bad state**。就表示通过。
cat /dev/net/tun
 
# 更新yum
yum install update -y
yum update -y
 
# 安装EPEL源，因为CentOS7官方源中已经去掉了xl2tpd
yum install -y epel-release
 
# 安装xl2tpd和libreswan（libreswan用以实现IPSec，原先的openswan已经停止维护）
yum install -y xl2tpd libreswan lsof net-tools
 
# 安装iptables防火墙，一般都有自带
yum install -y iptables-services
 
 
bigecho "VPN setup in progress... Please be patient."
 
# 获取默认网卡
if [ -z "$VPN_NET_IFACE" ]; then
  def_iface=$(route 2>/dev/null | grep -m 1 '^default' | grep -o '[^ ]*$')
  [ -z "$def_iface" ] && def_iface=$(ip -4 route list 0/0 2>/dev/null | grep -m 1 -Po '(?<=dev )(\S+)')
  def_state=$(cat "/sys/class/net/$def_iface/operstate" 2>/dev/null)
  if [ -n "$def_state" ] && [ "$def_state" != "down" ]; then
    case "$def_iface" in
      wl*)
        exiterr "Wireless interface '$def_iface' detected. DO NOT run this script on your PC or Mac!"
        ;;
    esac
    VPN_NET_IFACE="$def_iface"
  else
    eth0_state=$(cat "/sys/class/net/eth0/operstate" 2>/dev/null)
    if [ -z "$eth0_state" ] || [ "$eth0_state" = "down" ]; then
      exiterr "Could not detect the default network interface."
    fi
    VPN_NET_IFACE=eth0
  fi
fi
 
# 获取网卡绑定ip
if [ -z "$VPN_NET_IP" ]; then
  def_ip=$(ip a|grep inet|grep -v 127.0.0.1|grep -v inet6|grep $VPN_NET_IFACE |awk 'NR==1{print $2}'| awk -F '/' '{print $1}')
  if [ -z "$def_ip" ]; then
    exiterr "Could not detect the default network interface ip."
  fi
  VPN_NET_IP="$def_ip"
fi
 
# 获取默认VPN账号
if [ -z "$VPN_IPSEC_PSK" ] && [ -z "$VPN_USER" ] && [ -z "$VPN_PASSWORD" ]; then
  bigecho "VPN credentials not set by user. Generating random PSK and password..."
  VPN_IPSEC_PSK=$(LC_CTYPE=C tr -dc 'A-HJ-NPR-Za-km-z2-9' < /dev/urandom | head -c 20)
  VPN_USER=vpnuser
  VPN_PASSWORD=$(LC_CTYPE=C tr -dc 'A-HJ-NPR-Za-km-z2-9' < /dev/urandom | head -c 16)
fi
if [ -z "$VPN_IPSEC_PSK" ] || [ -z "$VPN_USER" ] || [ -z "$VPN_PASSWORD" ]; then
  exiterr "All VPN credentials must be specified. Edit the script and re-enter them."
fi
if printf '%s' "$VPN_IPSEC_PSK $VPN_USER $VPN_PASSWORD" | LC_ALL=C grep -q '[^ -~]\+'; then
  exiterr "VPN credentials must not contain non-ASCII characters."
fi
 
case "$VPN_IPSEC_PSK $VPN_USER $VPN_PASSWORD" in
  *[\\\"\']*)
    exiterr "VPN credentials must not contain these special characters: \\ \" '"
    ;;
esac
 
 
# 配置xl2tpd
cat>/etc/xl2tpd/xl2tpd.conf<<EOF
[global]
listen-addr = $VPN_NET_IP
auth file = /etc/ppp/chap-secrets
port = 1701
[lns default]
ip range = 172.168.1.128-172.168.1.254 
local ip = 172.168.1.99
require chap = yes
refuse pap = yes
require authentication = yes
name = LinuxVPNserver
ppp debug = yes
pppoptfile = /etc/ppp/options.xl2tpd
length bit = yes
EOF
 
# 配置ppp
cat>/etc/ppp/options.xl2tpd<<EOF
ipcp-accept-local
ipcp-accept-remote
ms-dns  8.8.8.8
ms-dns  8.8.4.4
# ms-dns  192.168.1.1
# ms-dns  192.168.1.3
# ms-wins 192.168.1.2
# ms-wins 192.168.1.4
name xl2tpd
noccp
auth
#crtscts
idle 1800
mtu 1400
mru 1400
nodefaultroute
debug
#lock
proxyarp
connect-delay 5000
refuse-pap
refuse-chap
refuse-mschap
require-mschap-v2
persist
#logfile /var/log/xl2tpd.log
EOF
 
# 配置IPSec
cat>/etc/ipsec.conf<<EOF
config setup
    protostack=netkey
    dumpdir=/var/run/pluto/
    virtual_private=%v4:10.0.0.0/8,%v4:172.100.0.0/12,%v4:25.0.0.0/8,%v4:100.64.0.0/10,%v6:fd00::/8,%v6:fe80::/10
#include /etc/ipsec.d/*.conf
EOF
 
# 设置预共享密钥PSK
cat>/etc/ipsec.secrets<<EOF
include /etc/ipsec.d/*.secrets
$VPN_NET_IP %any: PSK "$VPN_IPSEC_PSK"
EOF
 
# 配置服务器
cat>/etc/ipsec.d/l2tp_psk.conf<<EOF
conn L2TP-PSK-NAT
        rightsubnet=vhost:%priv
        also=L2TP-PSK-noNAT
conn L2TP-PSK-noNAT
        authby=secret
        pfs=no
        auto=add
        keyingtries=3
        dpddelay=30
        dpdtimeout=120
        dpdaction=clear
        rekey=no
        ikelifetime=8h
        keylife=1h
        type=transport
        left=$VPN_NET_IP
        leftprotoport=17/1701
        right=%any
        rightprotoport=17/%any
EOF
 
# 添加账号密码
cat>/etc/ppp/chap-secrets<<EOF
# Secrets for authentication using CHAP
# client        server  secret                  IP addresses
$VPN_USER * $VPN_PASSWORD *
EOF
 
# 开启内核转发
cat>/etc/sysctl.conf<<EOF
# Kernel sysctl configuration file for Red Hat Linux
#
# For binary values, 0 is disabled, 1 is enabled.  See sysctl(8) and
# sysctl.conf(5) for more details.
#
# Use '/sbin/sysctl -a' to list all possible parameters.
 
# Controls IP packet forwarding
net.ipv4.ip_forward = 1
 
# Controls source route verification
 
# Do not accept source routing
net.ipv4.conf.default.accept_source_route = 0
 
# Controls the System Request debugging functionality of the kernel
kernel.sysrq = 0
 
# Controls whether core dumps will append the PID to the core filename.
# Useful for debugging multi-threaded applications.
kernel.core_uses_pid = 1
 
# Controls the use of TCP syncookies
 
# Controls the default maxmimum size of a mesage queue
kernel.msgmnb = 65536
 
# Controls the maximum size of a message, in bytes
kernel.msgmax = 65536
 
# Controls the maximum shared segment size, in bytes
kernel.shmmax = 68719476736
 
# Controls the maximum number of shared memory segments, in pages
kernel.shmall = 4294967296
 
vm.swappiness = 0
net.ipv4.neigh.default.gc_stale_time = 120
 
# see details in https://help.aliyun.com/knowledge_detail/39428.html
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.default.arp_announce = 2
net.ipv4.conf.lo.arp_announce=2
net.ipv4.conf.all.arp_announce=2
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.default.log_martians = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.icmp_ignore_bogus_error_responses = 1
 
# see details in https://help.aliyun.com/knowledge_detail/41334.html
net.ipv4.tcp_max_tw_buckets = 5000
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 1024
net.ipv4.tcp_synack_retries = 2
 
#net.ipv6.conf.all.disable_ipv6 = 1
#net.ipv6.conf.default.disable_ipv6 = 0
#net.ipv6.conf.lo.disable_ipv6 = 1
EOF
 
# 重新加载内核配置项，如果输出结果有显示 net.ipv4.ip_forward = 1 ，则说明更改生效
sysctl -p
 
# 开启IPSec
ipsec setup start
ipsec verify
 
# 开启xl2tpd
systemctl start xl2tpd
 
# 此时关闭iptables防火墙进行测试，已经可以连上VPN，可以上外网
 
# 防火墙iptables设置
systemctl stop firewalld
iptables -F # 清空默认所有规则
iptables -X # 清空自定义所有规则
iptables -Z # 计数器置0
iptables -F -t nat
iptables -X -t nat
iptables -Z -t nat
iptables -F -t mangle
iptables -X -t mangle
iptables -Z -t mangle
 
#service iptables stop
iptables -P INPUT ACCEPT
iptables -F
iptables -X
iptables -Z
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -m state --state ESTABLISHED -j ACCEPT
iptables -A INPUT -p icmp -m icmp --icmp-type 8 -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -t nat -A POSTROUTING -o $VPN_NET_IFACE -j MASQUERADE
service iptables save
systemctl restart iptables
 
# 添加开机自启
chkconfig iptables on
chkconfig xl2tpd on
chkconfig ipsec on
 
 
#systemctl restart network
#systemctl enable ipsec
#systemctl restart ipsec
#ipsec verify
#systemctl enable xl2tpd
#systemctl restart xl2tpd
 
echo "================================================"
echo "IPsec VPN server is now ready for use!"
echo "Connect to your new VPN with these details:"
echo "Server IP: $VPN_NET_IP"
echo "IPsec PSK: $VPN_IPSEC_PSK"
echo "Username: $VPN_USER"
echo "Password: $VPN_PASSWORD"
echo "Write these down. You'll need them to connect!"
echo "To see connection log at /var/log/secure"
echo "================================================"

```

搭建ip隧道连接至阿里云ECS，本端机房选择1安装输入对端阿里云公网ip，对端阿里云选择2安装输入机房公网ip
```
#!/bin/sh
 
clear
echo -e "\033[32;1m############################################################################"
echo "                                                                           "
echo -e "                               IP遂道配置脚本                              "
echo "                                                                           "
echo " 说明："
echo " 1、正常网卡是外网IP时，入口和出口端使用正常IP即可"
echo " 2、网卡只有内网IP对外时，本端使用内网IP，对端使用外网IP，入口和出口都是一样"
echo " IP遂道是双向连接，所以两边连接的IP是对调的"
echo ""
echo -e "############################################################################\033[0m"
echo ""
 
exiterr() { echo "Error: $1" >&2; exit 1; }
bigecho() { echo; echo "## $1"; echo; }
 
# 获取默认网卡
DEFAULT_NET_IFACE=''
DEFAULT_NET_IP=''
if [ -z "$DEFAULT_NET_IFACE" ]; then
  def_iface=$(route 2>/dev/null | grep -m 1 '^default' | grep -o '[^ ]*$')
  [ -z "$def_iface" ] && def_iface=$(ip -4 route list 0/0 2>/dev/null | grep -m 1 -Po '(?<=dev )(\S+)')
  def_state=$(cat "/sys/class/net/$def_iface/operstate" 2>/dev/null)
  if [ -n "$def_state" ] && [ "$def_state" != "down" ]; then
    case "$def_iface" in
      wl*)
        exiterr "Wireless interface '$def_iface' detected. DO NOT run this script on your PC or Mac!"
        ;;
    esac
    DEFAULT_NET_IFACE="$def_iface"
  else
    eth0_state=$(cat "/sys/class/net/eth0/operstate" 2>/dev/null)
    if [ -z "$eth0_state" ] || [ "$eth0_state" = "down" ]; then
      exiterr "Could not detect the default network interface."
    fi
    DEFAULT_NET_IFACE=eth0
  fi
fi
  
# 获取网卡绑定ip
if [ -z "$DEFAULT_NET_IP" ]; then
  def_ip=$(ip a|grep inet|grep -v 127.0.0.1|grep -v inet6|grep $DEFAULT_NET_IFACE |awk 'NR==1{print $2}'| awk -F '/' '{print $1}')
  if [ -z "$def_ip" ]; then
    exiterr "Could not detect the default network interface ip."
  fi
  DEFAULT_NET_IP="$def_ip"
fi
 
# 获取公网ip和对应的网卡
PUBLIC_IP=$(curl -s icanhazip.com | tr -d "\n")
if [ -n "$PUBLIC_IP" ]; then
  PUBLIC_IFACE=$(ip a |grep inet | grep $PUBLIC_IP | awk '{print $8}')
  if [ -z "$PUBLIC_IFACE" ]; then
    # 出口ip没有对应的网卡使用默认网卡ip
    PUBLIC_IP="$DEFAULT_NET_IP"
    PUBLIC_IFACE="$DEFAULT_NET_IFACE"
  fi
else
  PUBLIC_IP="$DEFAULT_NET_IP"
  PUBLIC_IFACE="$DEFAULT_NET_IFACE"
fi
 
#PPP_IP=`ip a | grep 'global ppp' | awk '{print $4}' | sed -r 's/(.*\.).*/\10/g'`
PPP_IP=172.168.1.0
 
# 设置出口IP
function baseSet()
{
  echo ""
  read -p "本地出口IP [$PUBLIC_IP]：" t_export_ip
  read -p "本地出口网卡名 [$PUBLIC_IFACE]：" t_inet_intface
 
  if [ -z "$t_export_ip" ] || [ -z "$t_inet_intface" ];then
    echo ""
  else
    PUBLIC_IP=$t_export_ip
    PUBLIC_IFACE=$t_inet_intface
  fi
}
 
 
PORT=0
REMOTE_IP=""
 
 
# 本地安装ipip
function local_install()
{
yum -y install iptables-services
systemctl disable firewalld
systemctl stop firewalld
#iptables -F
#iptables -X
#iptables -Z
#iptables -F -t nat
#iptables -X -t nat
#iptables -Z -t nat
 
sysctl -w net.ipv4.ip_forward=1
 
cat >/bin/ipip.sh<<EOF
#! /bin/bash
 
isipip_tb=\`find "/etc/iproute2/rt_tables"|xargs grep -ri "ipip_tb"\`
if ! [ -n "\${isipip_tb}" ]; then
echo "201 ipip_tb" >> /etc/iproute2/rt_tables
fi
 
ip tunnel del tun0
modprobe ipip
ip tunnel add tun0 mode ipip local $PUBLIC_IP remote $REMOTE_IP ttl 255
ip link set tun0 up
ip addr add 10.138.0.1 peer 10.138.0.2 dev tun0
#route add -net 10.1.1.0/24 dev tun0
route add -net $PPP_IP/24 dev tun0
ip route add default via 10.138.0.2 dev tun0 table ipip_tb
#ip rule add from 10.1.1.0/24 table ipip_tb
ip rule add from $PPP_IP/24 table ipip_tb
 
#防火墙设置
#isiptable=\`find "/etc/sysconfig/iptables"|xargs grep -ri "\-A FORWARD \-i ppp+ \-p tcp \-m tcp \-\-tcp\-flags FIN,SYN,RST,ACK SYN \-j TCPMSS \-\-set\-mss 1356"\`
#if ! [ -n "\${isiptable}" ]; then
##iptables -t nat -A POSTROUTING -s 10.1.1.0/24 -o $PUBLIC_IFACE -j SNAT --to-source $PUBLIC_IP
##iptables -t nat -A POSTROUTING -s $PPP_IP/24 -o $PUBLIC_IFACE -j SNAT --to-source $PUBLIC_IP
##iptables -I FORWARD -i ppp+ -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -j TCPMSS --set-mss 1356
##service iptables save
##service iptables restart
#fi
 
#iptables -A FORWARD -i ppp+ -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -j TCPMSS --set-mss 1356
iptables -t nat -A POSTROUTING -d $REMOTE_IP -o $PUBLIC_IFACE -j MASQUERADE
 
ping 10.138.0.2 -c 10
EOF
 
#vpnInnerIPSet
chkconfig --level 2345 iptables on
isping=`find "/etc/crontab"|xargs grep -ri "ping 10.138.0.2"`
if ! [ -n "${isping}" ]; then
echo "*/5 * * * * root ping 10.138.0.2 -c 5" >> /etc/crontab
service crond restart
fi
}
 
# 对端安装
function remote_install()
{
yum -y install iptables
yum -y install iptables-services
systemctl disable firewalld
systemctl stop firewalld
#iptables -F
#iptables -X
#iptables -Z
#iptables -F -t nat
#iptables -X -t nat
#iptables -Z -t nat
 
sysctl -w net.ipv4.ip_forward=1
 
cat >/bin/ipip.sh<<EOF
#! /bin/bash
 
ip tunnel del tun0
modprobe ipip
ip tunnel add tun0 mode ipip local $PUBLIC_IP remote $REMOTE_IP ttl 255
ip link set tun0 up
ip addr add 10.138.0.2 peer 10.138.0.1 dev tun0
#route add -net 10.1.1.0/24 dev tun0
route add -net $PPP_IP/24 dev tun0
 
##防火墙设置
#isiptable=\`find "/etc/sysconfig/iptables"|xargs grep -ri "\-A FORWARD \-i ppp+ \-p tcp \-m tcp \-\-tcp\-flags FIN,SYN,RST,ACK SYN \-j TCPMSS \-\-set\-mss 1356"\`
#if ! [ -n "\${isiptable}" ]; then
##iptables -t nat -A POSTROUTING -s 10.1.1.0/24 -j SNAT --to-source $PUBLIC_IP
##iptables -t nat -A POSTROUTING -s $PPP_IP/24 -j SNAT --to-source $PUBLIC_IP
##iptables -I FORWARD -i ppp+ -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -j TCPMSS --set-mss 1356
##service iptables save
##service iptables restart
#fi
 
#iptables -A FORWARD -i ppp+ -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -j TCPMSS --set-mss 1356
iptables -t nat -A POSTROUTING -o $PUBLIC_IFACE -j MASQUERADE
 
ping 10.138.0.1 -c 10
EOF
 
chkconfig --level 2345 iptables on
isping=`find "/etc/crontab"|xargs grep -ri "ping 10.138.0.1"`
if ! [ -n "${isping}" ]; then
echo "*/5 * * * * root ping 10.138.0.1 -c 5" >> /etc/crontab
service crond restart
fi
}
 
# 设置对端ip
function remote_ip_set()
{
  echo -e "\033[33;1m  请输入对端外网IP :\033[0m \c"
  read REMOTE_IP
   
  if [ -z "$REMOTE_IP" ]; then
    remote_ip_set
  else
    if [ "$PORT" == 1 ]; then
      local_install
    else
      remote_install
    fi
  fi
}
 
function portSet(){
echo ""
echo -e "请选择 IP遂道 配置的端"
echo -e "  1 配置入口端 "
echo -e "  2 配置出口端"
#echo -e "  3 修改本地vpn内网IP"
echo -e "  e 退出安装"
echo ""
echo -e "\033[33;1m  请输入安装服务序号 :\033[0m \c"
read -n1 port_val
echo ""
case "$port_val" in
  1)
    PORT=1
  ;;
  2)
    PORT=2
  ;;
  e)
    exit
  ;;
  *)
  portSet
esac
remote_ip_set
 
chmod +x /etc/rc.d/rc.local
chmod +x /bin/ipip.sh
if [ -n `find "/etc/rc.d/rc.local"|xargs grep -ri "\/bin\/ipip.sh"` ]; then
echo "/bin/ipip.sh" >> /etc/rc.d/rc.local
fi
/bin/ipip.sh
 
isinstall=`ifconfig | grep -Po 'tun0'`
if [ -n "$isinstall" ]; then
echo -e "\033[32;1m安装成功！\033[0m"
else
echo -e "\033[31;1m 安装失败，请重新安装或检查是否有配置错误\033[0m "
fi
}
 
# 基本信息显示
echo -e "\n\033[32;1m本地网卡信息\n网卡名  IP地址"
#ifconfig | grep -v "lo" | grep -v "ppp" | grep -v "tun" | grep -A1 'flags='
ip a|grep inet|grep -v 127.0.0.1|grep -v inet6|grep -v ppp|grep -v tun
echo -e "\033[0m"
echo -e "出口网卡 \033[33;1m $PUBLIC_IFACE \033[0m"
echo -e "出口网卡IP \033[33;1m $PUBLIC_IP \033[0m"
echo -e "默认网卡   \033[33;1m $DEFAULT_NET_IFACE \033[0m"
echo -e "默认网卡IP   \033[33;1m $DEFAULT_NET_IP \033[0m"
echo -e "将使用的出口IP \033[33;1m $PUBLIC_IP \033[0m"
echo -e "将使用的ppp协议IP \033[33;1m $PPP_IP \033[0m"
 
read -p "是否需要修改配置信息？(y/n default:n) " isSet
if [ "$isSet" == "y" ] || [ -z "$PUBLIC_IP" ] || [ -z "$PUBLIC_IFACE" ]; then
  baseSet
fi
 
portSet


```


