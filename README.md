# 什么是L2TPVPN

L2TP VPN概述

什么是L2TP？

L2TP代表第2层隧道协议，它本身不提供任何加密。 L2TP VPN通常使用身份验证协议IPSec（Internet协议安全性）进行强大的加密和身份验证，这使其在某些其他最常用的协议（如PPTP）上具有最终优势。 L2TP协议使用UDP端口1701。

L2TP如何工作？

通过L2TP / IPSec协议传输的数据通常会进行两次身份验证。经由隧道传输的每个数据包均包含L2TP报头。结果，数据被伺服器解复用。数据的双重身份验证会降低性能，但是确实提供了最高的安全性。

## 1.配置yum源

```
yum服务
cd /etc/yum.repos.d/
ls
rm -rf *
vi dvd.repo
[dvd]
name=dvd
baseurl=file:///dvd
gpgcheck=0
enabled=1

mkdir /dvd
mount /dev/cdrom /dvd

yum install -y 服务
```

## 2.修改成网络源

```
下载阿里云
wget -O /etc/yum.repos.d/CentOS-Base-epel.repo http://mirrors.aliyun.com/repo/Centos-7.repo
清理缓存
yum clean all
重新生成缓存
yum makecache

安装EPEL源（CentOS7官方源中已经去掉了xl2tpd）
yum install -y epel-release
```

## 3.安装依赖包

```
yum install -y make gcc gmp-devel xmlto bison flex xmlto libpcap-devel lsof vim-enhanced man

yum install -y xl2tpd


yum install -y libreswan
```

## 4、修改ipsec的配置文件

```
vim /etc/ipsec.conf（只添加一行nat_traversal=yes即可）
```

![image-20220411224156553](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20220411224156553.png)

## 5、建立ipsec 与 l2tp 服务关联的配置文件

```
vim /etc/ipsec.d/l2tp_psk.conf

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
    left=192.168.0.200   ###192.168.0.200 是自己的网卡Ip地址
    leftprotoport=17/1701
    right=%any
    rightprotoport=17/%any

```

## 6、当建立l2tp连接时，需要输入预共享密匙，以下为预共享密匙的配置文件。

```
#如果这个文件也没有也需要手动创建，访问的IP地址和密码
vim /etc/ipsec.d/ipsec.secrets

#include /etc/ipsec.d/*.secrets
192.168.0.200 %any: PSK "123456789"
```

## 7、修改内核支持，可以对照以下配置修改，修改完后运行sysctl -p 使配置生效

```
cat /etc/sysctl.conf

vim /etc/sysctl.conf

net.ipv4.ip_forward = 1
net.ipv4.conf.default.accept_redirects  = 0
net.ipv4.conf.default.send_redirects  = 0
net.ipv4.conf.eno16777736.rp_filter = 0
net.ipv4.conf.default.rp_filter  = 0

sysctl -p
```

```
net.ipv4.ip_forward = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.lo.accept_redirects = 0
net.ipv4.conf.lo.rp_filter = 0
net.ipv4.conf.lo.send_redirects = 0
```



## 8、检验ipsec服务配置



```
#重启ipsec
systemctl restart ipsec

#检验ipsec服务配置 
ipsec verify
systemctl status ipsec
```

![image-20220411225418461](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20220411225418461.png)

![image-20220411225429628](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20220411225429628.png)

## 9、启动服务



```
#启动ipsec 
systemctl start ipsec

#设置为开机自启 
systemctl enable ipsec
```

## 10、修改L2tp的配置文件

```
vim /etc/xl2tpd/xl2tpd.conf 

[global]
 listen-addr = 192.168.0.197    ###本机外网网卡IP
 ipsec saref = yes      ###取消注释
[lns default]
ip range = 192.168.0.128-192.168.0.254
local ip = 192.168.0.99
require chap = yes
refuse pap = yes
require authentication = yes
name = Linux×××server
ppp debug = yes
pppoptfile = /etc/ppp/options.xl2tpd
length bit = yes
```

## 11、修改xl2tpd属性配置文件 



```
vim /etc/ppp/options.xl2tpd

require-mschap-v2   ###添加此行
ipcp-accept-local
ipcp-accept-remote
#dns 写自己的网卡DNS ，写成8.8.8.8也行
ms-dns 192.168.0.2 
#ms-dns  8.8.8.8
noccp
auth
crtscts
idle 1800
mtu 1410
mru 1410
nodefaultroute
debug
lock
proxyarp
connect-delay 5000
```

## 12、添加用户名和密码（**登录的用户名和密码

```
vim /etc/ppp/chap-secrets

# Secrets for authentication using CHAP
# client        server  secret                  IP addresses
test      *  123 *
```

## 13、iptables安装配置

### 1.安装iptable iptable-service

```
yum install -y iptables
yum install iptables-services
```

### 2.禁用/停止自带的firewalld服务

```
#停止firewalld服务
systemctl stop firewalld

#冻结firewalld服务
systemctl mask firewalld
```

### 3设置现有规则

```
#查看iptables现有规则
iptables -L -n

#先允许所有,不然有可能会杯具
iptables -P INPUT ACCEPT

#清空所有默认规则
iptables -F

#清空所有自定义规则
iptables -X

#所有计数器归0
iptables -Z
```

### 4.开启**地址转换(eth0为外网网卡，\**根据实际情况替换\****

```
iptables -t nat -A POSTROUTING -s 192.168.0.0/24 -o eno16777736 -j MASQUERADE
iptables -I FORWARD -s 192.168.0.0/24 -j ACCEPT
iptables -I FORWARD -d 192.168.0.0/24 -j ACCEPT
 
iptables -A INPUT -p udp -m policy --dir in --pol ipsec -m udp --dport 1701 -j ACCEPT
iptables -A INPUT -p udp -m udp --dport 1701 -j ACCEPT
iptables -A INPUT -p udp -m udp --dport 500 -j ACCEPT
iptables -A INPUT -p udp -m udp --dport 4500 -j ACCEPT
iptables -A INPUT -p esp -j ACCEPT
iptables -A INPUT -m policy --dir in --pol ipsec -j ACCEPT
 
iptables -A FORWARD -i ppp+ -m state --state NEW,RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
 
service iptables save
/bin/systemctl restart iptables.service
```

## 14、完成服务配置

```
#启动xl2tp服务 
systemctl start xl2tpd
 
#设置开机自启 
systemctl enable xl2tpd
 
#查看状态 
systemctl status xl2tpd
```

![image-20220411230304243](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20220411230304243.png)

# 作者QQ：2565543706
