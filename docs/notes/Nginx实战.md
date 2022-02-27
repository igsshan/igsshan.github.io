# Nginx实战

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