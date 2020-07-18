HOSTNAME=dnsmasq
hostnamectl set-hostname "$HOSTNAME"
echo "$HOSTNAME">/etc/hostname
echo "$(grep -E '127|::1' /etc/hosts)">/etc/hosts
echo "$(ip a|grep "inet "|grep -v 127|awk -F'[ /]' '{print $6}') $HOSTNAME">>/etc/hosts

yum -y install dnsmasq

cat >>/etc/dnsmasq.conf<<EOF
resolv-file=/etc/resolv.conf
strict-order
listen-address=127.0.0.1,$(hostname -i)
cache-size=150
log-queries
log-facility=/var/log/dnsmasq.log
address=/vincent.com/192.168.77.190
server=/google.com/8.8.8.8
EOF

echo '192.168.77.100 gitlab.com'>>/etc/hosts
systemctl restart dnsmasq
systemctl enable dnsmasq



参考：

https://blog.51cto.com/longlei/2065967

https://www.hi-linux.com/posts/30947.html


