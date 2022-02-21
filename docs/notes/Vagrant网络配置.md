# Vagrant 网络配置

> 修改网络 连接外网

```sh
# 进入到文件配置目录
cd /etc/sysconfig/network-scripts

# 查看 ip 在那个网卡
ip addr 
结果: 
eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:10:b5:a2 brd ff:ff:ff:ff:ff:ff
    inet 192.168.56.101/24 brd 192.168.56.255 scope global noprefixroute eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe10:b5a2/64 scope link 
       valid_lft forever preferred_lft forever

# 修改eth1 网卡文件
vi ifcfg-eth1
修改内容:
GETWAY=192.168.56.1
DNS1=114.114.114.114
DNS2=8.8.8.8

# 保存重启
esc
:wq
service network restart

```

> 设置虚拟机 yum源

```sh
# 备份 yum源
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup

# 使用新 yum 源
curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.163.com/.help/CentOS7-Base-163.repo

# 生成缓存
yum makecache
```

> 使用 yum 源

```sh
# 安装 wget
yum install -y wget
# -y 就是全是确认
```

