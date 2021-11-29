# SpringBoot发送HTTP请求

## HttpClient

### 什么是HttpClient

>  HttpClient 是Apache HttpComponents 下的子项目，用来提供高效的、最新的、功能丰富的支持 HTTP 协议的客户端编程工具包，并且它支持 HTTP 协议最新的版本和建议。

### HttpClient使用流程

- 创建HttpClient对象
- 创建请求方法的实例,并指定请求URL.如果发送GET请求,创建HttpGet对象;如果发送POST请求,创建HttpPost对象
- 调用HttpClient对象的execute(HttpUriRequest request)发送请求,该方法返回一个HttpResponse对象
- 调用HttpResponse的getAllHeaders()、getHeaders(String name)等方法可获取服务器的响应头;调用HttpResponse的getEntity()方法可获取HttpEntity对象,该对象包装了服务器的响应内容.程序可通过该对象获取服务器的响应内容
- 释放连接.无论执行方法是否成功,都必须释放连接

### HttpClient使用

> 引入依赖

```java
<dependency>
	<groupId>org.apache.httpcomponents</groupId>
    <artifactId>httpmime</artifactId>
</dependency>
```

> httpClient工具类

```java
/**
 * @author igsshan
 * @data 2021-10-14
 */
@Component
@Slf4j
public class HttpClientUtil {

    private CloseableHttpClient httpClient;

    public HttpClientUtil() {
        this.httpClient = HttpClients.createDefault();
    }

	
    /**
     * 发送get请求；带请求头和请求参数
     *
     * @param url    url
     * @param params params
     * @return 返回
     */
    public HttpResult doGet(String url, String params) {
        HttpResult httpResult = null;
        URIBuilder uriBuilder = null;
        try {
            uriBuilder = new URIBuilder(url);
            if (params != null) {
                Map<String, String> map = JsonUtil.toMap(params);
                Set<Map.Entry<String, String>> entitySet = map.entrySet();
                for (Map.Entry<String, String> entry : entitySet) {
                    uriBuilder.setParameter(entry.getKey(), entry.getValue());
                }
            }
            HttpGet httpGet = new HttpGet(uriBuilder.build());
            CloseableHttpResponse response = this.httpClient.execute(httpGet);
            if (response.getEntity() != null) {
                httpResult = new HttpResult(response.getStatusLine().getStatusCode(),
                        EntityUtils.toString(response.getEntity(), "UTF-8"));
            } else {
                httpResult = new HttpResult(response.getStatusLine().getStatusCode(), "");
            }
            response.close();
        } catch (Exception e) {
            log.error("调用接口出错：{}", e.getMessage());
        }
        return httpResult;
    }


    /**
     * 发送post请求
     *
     * @param url    url
     * @param params params
     * @return 返回结果
     */
    public HttpResult doPost(String url, String params) {
        HttpResult httpResult = null;
        try {
            HttpPost httpPost = new HttpPost(url);
            if (params != null) {
                Map<Object, Object> map = JsonUtil.toMap(params);
                List<NameValuePair> param = new ArrayList<NameValuePair>();
                for (Map.Entry<Object, Object> entry : map.entrySet()) {
                    param.add(new BasicNameValuePair(entry.getKey().toString(), entry.getValue().toString()));
                }
                UrlEncodedFormEntity formEntity = new UrlEncodedFormEntity(param, "UTF-8");
                httpPost.setHeader("Connection", "keep-alive");
                httpPost.setHeader("Accept", "application/json");
                httpPost.setHeader("User-Agent", "Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/65.0.3325.181 Safari/537.36");
                httpPost.setEntity(formEntity);
            }
            CloseableHttpResponse response = this.httpClient.execute(httpPost);
            if (response.getEntity() != null) {
                httpResult = new HttpResult(response.getStatusLine().getStatusCode(),
                        EntityUtils.toString(response.getEntity(), "UTF-8"));
            } else {
                httpResult = new HttpResult(response.getStatusLine().getStatusCode(), "");
            }
        } catch (IOException e) {
            log.error("调用接口出错：{}", e.getMessage());
        }
        return httpResult;
    }

}
```

> Json转换工具类

```java
/**
 * @author igsshan
 * @data 2021-10-14
 */
public class JsonUtil {
    private static final ObjectMapper MAPPER = new ObjectMapper();

    public JsonUtil() {
    }

    public static String toJSONString(Object o) {
        return toJSONString(o, false);
    }

    public static String toJSONString(Object o, boolean format) {
        try {
            if (o == null) {
                return "";
            } else if (o instanceof Number) {
                return o.toString();
            } else if (o instanceof String) {
                return (String)o;
            } else {
                return format ? MAPPER.writerWithDefaultPrettyPrinter().writeValueAsString(o) : MAPPER.writeValueAsString(o);
            }
        } catch (JsonProcessingException var3) {
            throw new RuntimeException(var3);
        }
    }

    public static <T> T toObject(String json, Class<T> cls) {
        if (!StringUtils.isBlank(json) && cls != null) {
            try {
                return MAPPER.readValue(json, cls);
            } catch (IOException var3) {
                throw new RuntimeException(var3);
            }
        } else {
            return null;
        }
    }

    public static <T> T toObject(String json, Class<?> parametrized, Class<?>... parameterClasses) {
        if (!StringUtils.isBlank(json) && parametrized != null) {
            try {
                JavaType javaType = MAPPER.getTypeFactory().constructParametricType(parametrized, parameterClasses);
                return MAPPER.readValue(json, javaType);
            } catch (IOException var4) {
                throw new RuntimeException(var4);
            }
        } else {
            return null;
        }
    }

    public static <T> T toObject(String json, TypeReference<T> typeReference) {
        if (!StringUtils.isBlank(json) && typeReference != null) {
            try {
                return MAPPER.readValue(json, typeReference);
            } catch (IOException var3) {
                throw new RuntimeException(var3);
            }
        } else {
            return null;
        }
    }

    public static JsonNode parse(String json) {
        if (StringUtils.isBlank(json)) {
            return null;
        } else {
            try {
                return MAPPER.readTree(json);
            } catch (IOException var2) {
                throw new RuntimeException(var2);
            }
        }
    }

    public static <K, V> Map<K, V> toMap(Object o) {
        if (o == null) {
            return null;
        } else {
            return o instanceof String ? (Map)toObject((String)o, Map.class) : (Map)MAPPER.convertValue(o, Map.class);
        }
    }

    public static <T> List<T> toList(String json) {
        if (StringUtils.isBlank(json)) {
            return null;
        } else {
            try {
                return (List)MAPPER.readValue(json, List.class);
            } catch (JsonProcessingException var2) {
                throw new RuntimeException(var2);
            }
        }
    }

    public static <T> List<T> toList(String json, Class<T> cls) {
        if (StringUtils.isBlank(json)) {
            return null;
        } else {
            try {
                JavaType javaType = MAPPER.getTypeFactory().constructParametricType(List.class, new Class[]{cls});
                return (List)MAPPER.readValue(json, javaType);
            } catch (JsonProcessingException var3) {
                throw new RuntimeException(var3);
            }
        }
    }

    static {
        MAPPER.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
        MAPPER.configure(SerializationFeature.FAIL_ON_EMPTY_BEANS, false);
        MAPPER.configure(JsonReadFeature.ALLOW_UNQUOTED_FIELD_NAMES.mappedFeature(), true);
        MAPPER.configure(JsonReadFeature.ALLOW_SINGLE_QUOTES.mappedFeature(), true);
        MAPPER.configure(JsonReadFeature.ALLOW_LEADING_ZEROS_FOR_NUMBERS.mappedFeature(), true);
        MAPPER.configure(JsonReadFeature.ALLOW_UNESCAPED_CONTROL_CHARS.mappedFeature(), true);
        MAPPER.setSerializationInclusion(Include.NON_NULL);
        MAPPER.setPropertyNamingStrategy(PropertyNamingStrategy.LOWER_CAMEL_CASE);
        MAPPER.enable(new MapperFeature[]{MapperFeature.USE_STD_BEAN_NAMING});
        MAPPER.setDateFormat(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));
        MAPPER.setTimeZone(TimeZone.getTimeZone("GMT+8"));
    }
}
```

> 响应类

```java
/**
 * @author igsshan
 */
public class HttpResult {
    /**
     * 响应的状态码
     */
    private int code;

    /**
     * 响应的响应体
     */
    private String body;

    public HttpResult() {
    }

    public HttpResult(int code, String body) {
        this.code = code;
        this.body = body;
    }

    public int getCode() {
        return code;
    }

    public void setCode(int code) {
        this.code = code;
    }

    public String getBody() {
        return body;
    }

    public void setBody(String body) {
        this.body = body;
    }

    @Override
    public String toString() {
        return "HttpResult{" +
                "code=" + code +
                ", body='" + body + '\'' +
                '}';
    }
}
```

> 测试

```java
public class Demo{
    @Autowired
    HttpClientUtil httpCilentUtil;
}
```

## RestTemplate

### 简述RestTemplate

> RestTemplate是Spring用于同步client端的核心类,简化了与HTTP服务的通信,并且满足了RestFu原则,默认情况下，RestTemplate默认依赖jdk的HTTP连接工具。当然你也可以 通过setRequestFactory属性切换到不同的HTTP源，比如Apache HttpComponents、Netty和OkHttp

- 该类的入口主要是根据HTTP的六个方法制定：

| HTTP method | RestTemplate methods | 解析                                                         |
| ----------- | -------------------- | ------------------------------------------------------------ |
| DELETE      | delete               | 在特定的URL上对资源执行HTTP DELETE操作                       |
| GET         | getForObject         | 发送一个HTTP GET请求，返回的请求体将映射为一个对象           |
|             | getForEntity         | 发送一个HTTP GET请求，返回的ResponseEntity包含了响应体所映射成的对象 |
| HEAD        | headForHeaders       | 发送HTTP HEAD请求，返回包含特定资源URL的HTTP头               |
| OPTIONS     | optionsForAllow      | 发送HTTP OPTIONS请求，返回对特定URL的Allow头信息             |
| POST        | postForLocation      | POST 数据到一个URL，返回新创建资源的URL                      |
|             | postForEntity        | POST 数据到一个URL，返回包含一个对象的ResponseEntity         |
|             | postForObject        | POST 数据到一个URL，返回根据响应体匹配形成的对象             |
| PUT         | put                  | PUT 资源到特定的URL                                          |
| any         | exchange             | 在URL上执行特定的HTTP方法，返回包含对象的ResponseEntity      |
|             | execute              | 在URL上执行特定的HTTP方法，返回一个从响应体映射得到的对象    |

### RestTemplate使用

> 引入依赖 ( RestTemplate 集成在 Web Start 中)

```java
<!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-web 
-->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
  <version>2.2.0.RELEASE</version>
</dependency>
```

> 调用工具类

```java
/**
 * @author igsshan
 * @data 2021-11-29
 */
@Component
@Slf4j
public class HttpRestfulUtil {
    @Autowired
    private RestTemplate restTemplate;

    public <T> T doGet (HttpCallDto dto, Class<T> tClass) {
        if (null == dto || null == dto.getUrl()) {
            return null;
        }
        ResponseEntity<T> response;
        if (null == dto.getHeaders()) {
            response = restTemplate.getForEntity(handleUrl(dto), tClass, dto.getParams());
        } else {
            HttpEntity<Object> request = new HttpEntity<>(dto.getBody(), handleHeader(dto));
            response = restTemplate.exchange(handleUrl(dto), HttpMethod.GET, request, tClass, dto.getParams());
        }
        return response.getBody();
    }

    public <T> T doPost (HttpCallDto dto, Class<T> tClass) {
        if (null == dto || null == dto.getUrl()) {
            return null;
        }
        HttpEntity<Object> request = new HttpEntity<>(dto.getBody(), handleHeader(dto));
        ResponseEntity<T> response = restTemplate.postForEntity(handleUrl(dto), request, tClass);
        return response.getBody();
    }

    private HttpHeaders handleHeader (HttpCallDto dto) {
        if (null == dto) {
            return new HttpHeaders();
        }
        HttpHeaders header = dto.getHeaders();
        if (null == header) {
            header = new HttpHeaders();
        }
        List<MediaType> accept = new ArrayList<>();
        accept.add(MediaType.APPLICATION_JSON_UTF8);
        header.setAccept(accept);
        header.setContentType(MediaType.APPLICATION_JSON_UTF8);
        return header;
    }

    private String handleUrl (HttpCallDto dto) {
        if (null == dto || null == dto.getUrl()) {
            return StringUtils.EMPTY;
        }
        StringBuilder urlTemplate = new StringBuilder();
        urlTemplate.append(dto.getUrl());

        Map<String, String> params = dto.getParams();
        if (null == params || params.isEmpty()) {
            return urlTemplate.toString();
        } else {
            if (!dto.getUrl().contains("?")) {
                urlTemplate.append('?');
            }
            for (String key : params.keySet()) {
                urlTemplate.append(key);
                urlTemplate.append('=');
                urlTemplate.append(params.get(key));
                urlTemplate.append('&');
            }
            return urlTemplate.substring(0, urlTemplate.length() - 1);
        }
    }

    public static JSONObject postUrlRetJson(String url, Map<String, String> headerMap, String jsonString) {
        CloseableHttpClient httpclient = HttpClients.createDefault();
        HttpPost httpPost = new HttpPost(url);
        JSONObject retJson = null;
        log.info("发起POST url：{},参数：{}", url, jsonString);
        try {
            for (String key : headerMap.keySet()) {
                httpPost.setHeader(key,headerMap.get(key));
            }
            StringEntity reqEntity = new StringEntity(jsonString, "utf-8");
            httpPost.setEntity(reqEntity);
            CloseableHttpResponse response = httpclient.execute(httpPost);
            retJson = JSONObject.parseObject(EntityUtils.toString(response.getEntity()));
        } catch (Exception ex) {
            log.error(ex.getMessage());
        }finally {
            try {
                httpclient.close();
            } catch (IOException e) {
                log.error(e.getMessage());
            }
        }
        return retJson;
    }
}
```

> 封装参数Dto

```java
/**
 * @author igsshan
 * @data 2021-11-29
 */
@Data
public class HttpCallDto {

    private String url;

    private HttpHeaders headers;

    private Map<String, String> params;

    private Object body;

}
```

## Forest

> https://forest.dtflyx.com/
>
> Forest能够帮助您使用更简单的方式编写Java的HTTP客户端，快速接入第三方RESTful接口

### 什么是Forest

> Forest 是一个开源的 Java HTTP 客户端框架，它能够将 HTTP 的所有请求信息（包括 URL、Header 以及 Body 等信息）绑定到您自定义的 Interface 方法上，能够通过调用本地接口方法的方式发送 HTTP 请求。

### 为什么使用Forest

> 使用 Forest 就像使用类似 Dubbo 那样的 RPC 框架一样，只需要定义接口，调用接口即可，不必关心具体发送 HTTP 请求的细节。同时将 HTTP 请求信息与业务代码解耦，方便您统一管理大量 HTTP 的 URL、Header 等信息。而请求的调用方完全不必在意 HTTP 的具体内容，即使该 HTTP 请求信息发生变更，大多数情况也不需要修改调用发送请求的代码。

### 依赖环境

> Forest 1.0.x 和 Forest 1.1.x 基于 JDK 1.7, Forest 1.2.x及以上版本基于 JDK 1.8

### Forest使用

> 引入依赖

```java
<dependency>
    <groupId>com.dtflys.forest</groupId>
    <artifactId>forest-spring-boot-starter</artifactId>
    <version>1.5.13</version>
</dependency>

<!-- json依赖 -->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>fastjson</artifactId>
    <version>1.2.48</version>
</dependency>
```

> Hello World

- 创建一个 interface

  ```java
  
  public interface MyClient {
  
      @Request("http://localhost:8080/hello")
      String helloForest();
  
  }
  ```

- SpringBoot启动类

  ```java
  /**
   * 只要在Spring Boot的配置类或者启动类上加上@ForestScan注解，并在basePackages属性里填上远程接口的所在的包名
   */
  @SpringBootApplication
  @Configuration
  @ForestScan(basePackages = "com.yoursite.client")
  public class MyApp {
   ...
  }
  ```

- 接口调用

  ```java
  @Component
  public class MyService {
      @Autowired
      private MyClient myClient;
  
      public void testClient() {
          String result = myClient.helloForest();
          System.out.println(result);
      }
  
  }
  ```