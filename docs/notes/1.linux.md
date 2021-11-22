# Linux使用

## 1.配置网络

```sh
# 修改网卡配置
vi /etc/sysconfig/network-scripts/ifcfg-ens33
```

```sh
# 修改文件内容
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static
IPADDR=192.168.23.129
NETMASK=255.255.255.0
GATEWAY=192.168.23.2
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens33
UUID=2c2371f1-ef29-4514-a568-c4904bd11c82
DEVICE=ens33
ONBOOT=true
```

```sh
# 保存退出
:wq!

# 重启网络
service network restart

# 测试网络是否正常
ping www.baidu.com

# 查看IP地址
ip addr
```

>解决centos7在Windows下ping不通的原因

```sh
vi /etc/sysconfig/network-scripts/ifcfg-ens33
```

```sh
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static
IPADDR=192.168.23.129   -->对应子网ip地址
NETMASK=255.255.255.0	-->子网掩码地址
GATEWAY=192.168.73.2    -->网关对应子网ip
DNS1=192.168.73.2       -->对应网关ip
DNS2=114.14.114.114		-->固定ip
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens33
UUID=2c2371f1-ef29-4514-a568-c4904bd11c82
DEVICE=ens33
ONBOOT=true
```

```sh
# 重启网络
service network restart

# ping百度
ping www.baidu.com

# 查询ip
ifconfig
```

## 2.安装JDK

>一般在安装oracle的JDK之前,会先移除openJDK

```sh
# 查询jdk版本信息
java -version

# 查询所有包含java的文件
rpm -qa | grep java
-qa		使用询问模式，查询所有套件
grep	查找文件里符合条件的字符串
java	查找包含java字符串的文件
注意:noarch文件可以不用删除

# 移除openJDK
rpm -e --nodeps java-....
-e			删除指定的套件
--nodeps	不验证套件档的相互关联性
# 删除之后,再用 java -version 命令验证
# 如果删除失败,使用yum -y remove去删除他们
```

### 1.下载jdk

```xml
在oracle官网上下载jdk linux版本 64位安装包
```

#### 2.上传

```xml
上传安装包到/opt/software目录下
```

#### 3.解压

```xml
解压安装包并移动到/opt/source/jdk目录下
```

#### 4.配置环境变量

```sh
vi /etc/profile

export JAVA_HOME=/opt/app/jdk/jdk1.8.0_291
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib:$CLASSPATH
export JAVA_PATH=${JAVA_HOME}/bin:${JRE_HOME}/bin
export PATH=$PATH:${JAVA_PATH}
```

#### 5.查看

```sh
# 更新配置
source /etc/profile

# 查看JAVA_HOME
echo $JAVA_HOME

# 查看PATH
echo $path

# 查看版本信息
java -version
```

## 3.yum相关命令

```sh
# yum install package -y
默认是安装来自仓库里的软件，指定的是软件名字。多个包空格隔开；-y （取消交互）

# yum install ./xxx.rpm
或者
# yum localinstall ./xxx.rpm
安装来自本地指定路径下的rpm包，而不是来自仓库

# yum remove 或者 erase package
卸载软件包

# yum update
更新仓库里所有比本机已经安装过的软件要的软件	

# yum update package
指定升级的软件

# yum search mysql
搜索出所有软件名字“mysql”关键字的软件

# yum provides  "libaudiofile.so.0"
找出模块由哪些软件包提供

# yum clean all
清空之前的yum列表缓存

# yum makecache
创建新的缓存

# yum list
列出仓库里的所有软件包

# yum repolist
列出已配置的软件仓库

# yum list |grep 关键字
代表已经安装成功

# yum list installed
查看已安装的包

# yum grouplist
查看包组

# yum groupinstall  "包组"
安装包组

# yum groupremove "包组"
移除包组

# 查看pcre包是否重复
rpm -qa | grep pcre 

# 移除一个
yum remove pcre-8.32-15.el7_2.1.i686
```

## 4.防火墙相关

```sh
# 防火墙命令

# 查看防火墙状态:
systemctl status firewalld

# 关闭防火墙命令：
systemctl stop firewalld.service

# 开启防火墙：
systemctl start firewalld.service

# 关闭开机自启动：
systemctl disable firewalld.service

# 开启开机启动：
systemctl enable firewalld.service
```

> 解决:Unit firewalld.service could not be found 问题

```sh
# 如果提示: Unit firewalld.service could not be found .说明没有安装防火墙,需要安装

yum install firewalld firewall-config
```

注意: centos7下默认的防火墙是firewall,替代了之前的iptables,firewall有图形化管理界面和命令行管理两种方式.

```sh
# 重启,关闭,开启firewall.service服务

service firewalld restart
service firewalld start
service firewalld stop
```

```sh
# 添加自定义端口
# 添加8080端口
firewall-cmd --zone=public --permanent --add-port=8080/tcp

注意:一定要刷新
# 刷新生效
firewall-cmd --reload

# 查看firewall服务状态
systemctl status firewalld.service

[root@localhost ~]# systemctl status firewalld.service
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)
     
# 查看firewall状态
firewall-cmd --state

[root@localhost ~]# firewall-cmd --state
not running
```

```sh
# 查看防火墙放行规则
firewall-cmd --list-all

# 添加服务
firewall-cmd --permanent --zone=public --add-service=http

firewall-cmd --state                           #查看防火墙状态，是否是running
 
firewall-cmd --reload                          #重新载入配置，比如添加规则之后，需要执行此命令
 
firewall-cmd --get-zones                       #列出支持的zone
 
firewall-cmd --get-services                    #列出支持的服务，在列表中的服务是放行的
 
firewall-cmd --query-service ftp               #查看ftp服务是否支持，返回yes或者no
 
firewall-cmd --add-service=ftp                 #临时开放ftp服务
 
firewall-cmd --add-service=ftp --permanent     #永久开放ftp服务
 
firewall-cmd --remove-service=ftp --permanent  #永久移除ftp服务
 
firewall-cmd --add-port=80/tcp --permanent     #永久添加80端口 
 
firewall-cmd --remove-port=80/tcp --permanent  #永久移除80端口
 
firewall-cmd --list-ports                      #查看已经开放的端口
 
iptables -L -n                                 #查看规则，这个命令是和iptables的相同的
 
man firewall-cmd                               #查看帮助
```

