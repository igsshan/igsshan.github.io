# Nginx实战

## linux下以jar包安装nginx

```sh
# 安装之前检查是否有安装过
nginx find-name nginx 1
```

```sh
## 下载安装包(最好在自己自定义的文件夹下下载压缩包)
wget http://nginx.org/download/nginx-1.20.2.tar.gz
```

```sh
## 安装nginx所有依赖
yum -y install gcc pcre-devel zlib-devel openssl openssl-devel
```

```sh
## 安装nginx
#解压
tar -zxvf nginx-1.20.2.tar.gz

#进入NG目录
cd ./nginx-1.20.2

#配置
./configure --prefix=/usr/local/nginx

#编译
make
make install
```

```sh
#启动
/usr/local/nginx/sbin/nginx

#重新加载配置
/usr/local/nginx/sbin/nginx -s reload

#停止
/usr/local/nginx/sbin/nginx -s stop

```

## linux下以docker方式安装nginx

```sh
## 下载镜像
docker pull redis #下载最新 redis => redis:latest
docker pull redis:6.2.4 # 下载指定版本
```

```sh
## 启动镜像
docker run --name=mynginx \
-d --restart=always \
-p 88:80 \
-v /data/html:/usr/share/nginx/html:ro \
nginx

# -v 挂载外部数据,修改页面只需要在主机的 /data/html下即可
# ro :只读
# rw :读写
```

> 实战部署->复制nginx容器配置到本地文件夹

```sh
## 实战部署

# 启动一个nginx服务
docker run -p 80:80 --name nginx -d nginx:1.10  # 如果没有会自动下载

# 复制nginx容器配置到本地文件夹/mydata/nginx
docker container cp nginx:/etc/nginx .  # 注意后面有一个点 (.) 不要遗忘

#修改文件名称：
mv nginx conf   # 把这个 conf 移动到/mydata/nginx 下 

# 终止原容器：
docker stop nginx 

# 执行命令删除原容器：
docker rm $ContainerId

# 挂载外部配置文件启动nginx服务
docker run -p 80:80 --name nginx -v /mydata/nginx/html:/usr/share/nginx/html -v /mydata/nginx/logs:/var/log/nginx -v /mydata/nginx/conf:/etc/nginx -d nginx:1.10

# 开放防火墙端口后测试访问
http:localhost:80
```





## Nginx搭建域名访问(反向代理配置)

> nginx官网
>
> https://nginx.org/en/docs/

- 配置本地hosts映射地址

  > windows本机hosts文件(C:\Windows\System32\drivers\etc\hosts)

  ```sh
  192.168.56.101(ip)  igsshan.com(域名)
  ```

- 修改nginx文件

  > nginx.conf文件

  ```conf
  listen 80;
  server_name igsshan.com;
  
  location / {
  	proxy_pass http://192.168.56.101:8080;
  }
  ```

- docker启动nginx

  ```sh
  docker restart nginx
  ```

让nginx为我们代理,所有来之 igsshan.com 的请求,都代理转发到 192.168.56.101:8080 这个服务地址



## Nginx搭建域名访问(负载均衡到网关)

> 由nginx反向代理单独的一个服务,在微服务架构下,这个服务配置了集群,反向代理就会很复杂了.
>
> 这次,配置网关负载均衡到各个服务

- 修改nginx文件

  > nginx.conf

  ```sh
  # 加入upstream栏,配置上游服务器
    upstream myapp1 {
          least_conn;
          server srv1.example.com;
          server srv2.example.com;
          server srv3.example.com;
      }
      
  # 以上是案例
  # 结合案例配置一个自己的
    upstream igsshan {
          server 192.168.56.1:88;
          server srv2.example.com;
          server srv3.example.com;
      }
  ```

  > 修改location块

  ```conf
  location / {
  	proxy_set_header Host $host
  	proxy_pass http://igsshan
  }
  # 在进行网关转发的时候,会丢失很多东西,这里都丢失请求host,需要我们自己加上,带到网关中
  ```

  > 配置网关路由

  ```yml
  - id: mall_host_route
    uri: lb://product
    predicates:
    - Host=**.igsshan.mall.com,igsshan.mall.com
  ```

- docker启动nginx

  ```sh
  docker restart nginx
  ```



## Nginx实现动静分离(Nginx加载静态资源包括js,css等)

> nginx配置加载静态资源 css/js 样式

- 首先,将静态资源等文件,在外部挂载的/nginx/html 文件夹下创建一个static文件夹,存储静态资源文件

- 修改nginx配置

  ```
  location /static/{
      root /user/share/nginx/html;
  }
  ```

- 配置首页index.html资源请求路径

- 重启 nginx 服务

