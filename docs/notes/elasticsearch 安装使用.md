# elasticsearch 安装使用

- docker 安装 elasticsearch

> 拉取镜像

```xml
docker pull elasticsearch:7.4.2   # 存储和检索数据

docker pull kibana:7.4.2      # 可视化检索数据
```

> 创建实例

```linux
mkdir -p /mydata/elasticsearch/config # 外部配置

mkdir -p /mydata/elasticsearch/data  # 存放数据

echo "http.host: 0.0.0.0" >> /mydata/elasticsearch/config/elasticsearch.yml # 访问权限

chmod -R 777 /mydata/elasticsearch/  # 保证权限
```

> 启动命令

```linux

docker run --name elasticsearch -p 9200:9200 -p 9300:9300 \ 
-e "discovery.type=single-node" \
-e ES_JAVA_OPTS="-Xms64m -Xmx512m" \ 
-v /mydata/elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml \ 
-v /mydata/elasticsearch/data:/usr/share/elasticsearch/data \ 
-v /mydata/elasticsearch/plugins:/usr/share/elasticsearch/plugins \ 
-d elasticsearch:7.4.2
```

- 注意:-   e ES_JAVA_OPTS="-Xms64m -Xmx256m"  测试环境下，设置 ES 的初始内存和最大内存，否则导 

  致过大启动不了 ES 

> 启动kibana

```linux
docker run --name kibana -e ELASTICSEARCH_HOSTS=http://192.168.56.10:9200 -p 5601:5601 \ -d kibana:7.4.2 


http://192.168.56.10:9200 一定改为自己虚拟机的地址
```

