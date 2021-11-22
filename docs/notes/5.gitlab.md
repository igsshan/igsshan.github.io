# 基于docker服务搭建gitlab服务端

### 下载GitLab镜像

```linux
docker pull gitlab/gitlab-ce
```

### 运行如下命令来启动Gitlab

```text
需要注意的是我们的Gitlab的http服务运行在宿主机的1080端口上，这里我们将Gitlab的配置，日志以及数据目录映射到了宿主机的指定文件夹下，防止我们在重新创建容器后丢失数据。
```

```linux
docker run --detach \
  --publish 10443:443 --publish 1080:80 --publish 1022:22 \
  --name gitlab \
  --restart always \
  --volume /mydata/gitlab/config:/etc/gitlab \
  --volume /mydata/gitlab/logs:/var/log/gitlab \
  --volume /mydata/gitlab/data:/var/opt/gitlab \
  gitlab/gitlab-ce:latest

```

### 开启防火墙的指定端口

```linux
# 开启1080端口
firewall-cmd --zone=public --add-port=1080/tcp --permanent 
# 重启防火墙才能生效
systemctl restart firewalld
# 查看已经开放的端口
firewall-cmd --list-ports
```

### 访问Gitlab

访问地址: localhost:1080/

### 通过docker命令动态查看容器启动日志来知道gitlab是否已经启动完成

```linux
docker logs gitlab -f
```

