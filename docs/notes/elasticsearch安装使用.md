# elasticsearch 安装使用

## docker 安装 elasticsearch

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

## elasticsearch 基本命令使用

>初步检索

```xml
_cat 

GET /_cat/nodes：#查看所有节点 

GET /_cat/health：#查看 es 健康状况 

GET /_cat/master：#查看主节点 

GET /_cat/indices：#查看所有索引

```

> 索引一个文档

- put . post 操作

- PUT 可以新增可以修改。PUT 必须指定 id；由于 PUT 需要指定 id，我们一般都用来做修改 

  操作，不指定 id 会报错。

  POST 新增。如果不指定 id，会自动生成 id。指定 id 就会修改这个数据，并新增版本号

```JAVA
PUT customer/external/1；在 customer 索引下的 external 类型下保存 1 号数据为

url: PUT http://ip:port/customer/external/1
body: 
	{ "name": "John Doe" }

```

```JAVA
POST customer/external/1；在 customer 索引下的 external 类型下保存 1 号数据为

url: POST http://ip:port/customer/external/1
body: 
	{ "name": "John Doe" }
```

> 查询文档

- GET

```xml
GET customer/external/1

结果： 
	{ 
		"_index": "customer", //在哪个索引 
		"_type": "external", //在哪个类型 
		"_id": "1", //记录 id 
		"_version": 2, //版本号 
		"_seq_no": 1, //并发控制字段，每次更新就会+1，用来做乐观锁 
		"_primary_term": 1, //同上，主分片重新分配，如重启，就会变化 
		"found": true, 
		"_source": 
			{ 
				//真正的内容 
				"name": "John Doe" 
			} 
	}
```

```url
更新携带 ?if_seq_no=1&if_primary_term=1

GET customer/external/1?if_seq_no=1&if_primary_term=1
```

> 更新文档 

- 不同：POST 操作会对比源文档数据，如果相同不会有什么操作，文档 version 不增加 

  PUT 操作总会将数据重新保存并增加 version 版本； 

  带_update 对比元数据如果一样就不进行任何操作。 

  看场景； 

  对于大并发更新，不带 update； 

  对于大并发查询偶尔更新，带 update；对比更新，重新计算分配规则。

```xml
POST customer/external/1/_update 

{ 
	"doc":
		{ 
			"name": "John Doew" 
		} 
}


or


POST customer/external/1 

{ "name": "John Doe2" }


or

PUT customer/external/1 

{ "name": "John Doe" }
```

```x
在更新时,同时增加属性
POST customer/external/1/_update 
{ 
	"doc": 
		{ 
			"name": "Jane Doe", 
			"age": 20 
		} 
}

PUT 和 POST 不带_update 也可以
```

> 删除文档&索引

- es中只有删除文档或者删除整个索引,没有删除类型

```
DELETE customer/external/1 
OR
DELETE customer
```

## elasticsearch 高级检索

> SearchAPI

ES 支持两种基本方式检索 : 

- 一个是通过使用 REST request URI 发送搜索参数（uri+检索参数 )
- 另一个是通过使用 REST request body 来发送它们（uri+请求体）

### 检索从_search 开始

- 检索信息

```sh
GET bank/_search
#检索 bank 下所有信息，包括 type 和 docs

GET bank/_search?q=*&sort=account_number:asc
#请求参数方式检索

GET bank/_search 
{ 
	"query": { 
		"match_all": {} 
		},
  	"sort": [ 
  		{ 
  			"account_number": { 
  				"order": "desc" 
  				} 
  		} 
  	]
}
# URI+请求体参数方式检索
注意:HTTP 客户端工具（POSTMAN），get 请求不能携带请求体，我们变为 post 也是一样的
```

- Query DSL

  - 基本语法格式

  ```XML
  Elasticsearch 提供了一个可以执行查询的 Json 风格的 DSL（domain-specific language 领域特 定语言）。这个被称为 Query DSL。该查询语言非常全面
  ```

  ```sh
  1. 一个查询语句 的典型结构如下：
  { 
  	QUERY_NAME: { 
  		ARGUMENT: VALUE, 
  		ARGUMENT: VALUE,... 
  		} 
  }
  
  2. 如果是针对某个字段，那么它的结构如下：
  { 
  	QUERY_NAME: { 
  		FIELD_NAME: { 
  			ARGUMENT: VALUE, 
  			ARGUMENT: VALUE,... 
  		} 
  	} 
  }
  ```

  ```sh
  # 查询案例
  GET bank/_search 
  { 
  	"query": { 
  		"match_all": {} 
  	},
  	"from": 0, 
  	"size": 5, 
  	"sort": [ 
  		{ "account_number": { 
  			"order": "desc" 
  			} 
  		} 
  	] 
  }
  
  # query 定义如何查询， 
  # match_all 查询类型【代表查询所有的所有】，es中可以在 query 中组合非常多的查 询类型完成复杂查询 
  # 除了 query 参数之外，我们也可以传递其它的参数以改变查询结果。如 sort，size 
  # from + size 限定，完成分页功能 
  # sort 排序，多字段排序，会在前序字段相等时后续字段内部排序，否则以前序为准
  ```

  - 返回部分字段

  ```sh
  # 查询案例
  GET bank/_search 
  { 
  	"query": {
  		"match_all": {} 
  	},
  	"from": 0, 
  	"size": 5, 
  	"_source": ["age","balance"] 
  }
  
  # from + size 限定，完成分页功能 
  # _source 只返回数组限定的元素
  ```

### match 匹配查询

- 基本类型（非字符串），精确匹配

```sh
GET bank/_search 
{ 
	"query": { 
		"match": { 
			"account_number": "20" 
		} 
	} 
}

# "account_number": "20"  account_number = 20 是基本类型,查询就是精确匹配
```

- 字符串，全文检索

```sh
GET bank/_search 
{ 
	"query": { 
		"match": { 
			"address": "mill road" 
		} 
	} 
}

# "address": "mill"  address = mill  是字符串类型,查询就是全文检索(相当于模糊查询)
# 最终查询出 address 中包含 mill 或者 road 或者 mill road 的所有记录，并给出相关性得分
```

