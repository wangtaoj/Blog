### 前言

`Elasticsearch`官方列出了好几个客户端，如下所示

* Java Client
* Java Rest Client（Java High Level REST Client）
* Java Transport Client

其中Java Rest Client在7.15.0被标记已过时，Java Transport Client暂时没找到在哪个版本被标记过时

注：

* [官方文档](https://www.elastic.co/guide/en/elasticsearch/client/index.html)
* Spring Boot 2.3.12.RELEASE
* Elasticsearch 7.17.5

### Java Client 集成

Java Client在构建对象时支持Build模式以及Lambda两种形式，暴露出来的API为`ElasticsearchClient`类，通过该类可进行对索引、文档的基本操作。

`ElasticsearchClient`对象初始化步骤

第一步，引入依赖

```xml
<dependency>
    <groupId>co.elastic.clients</groupId>
    <artifactId>elasticsearch-java</artifactId>
    <version>7.17.5</version>
</dependency>

<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.12.3</version>
</dependency>

<dependency>
    <groupId>jakarta.json</groupId>
    <artifactId>jakarta.json-api</artifactId>
    <version>2.0.1</version>
</dependency>
```

在Spring Boot项目中，如果引入了以下依赖，便不用单独引入`jackson-databind`依赖了

```xml
<!-- 已经包含了jackson-databind -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

第二步，配置`ObjectMapper`

如果不想自定义`ObjectMapper`的行为，可以省略，这里主要想要支持`LocalDate`、`LocalDateTime`类，不然文档中如果包含时间列，反序列成对象时会报错。

在Spring Boot项目中只需加入以下配置即可

```java
package com.wangtao.msgsearch.config;

import com.fasterxml.jackson.annotation.JsonInclude;
import com.fasterxml.jackson.core.JsonGenerator;
import com.fasterxml.jackson.datatype.jdk8.Jdk8Module;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import com.fasterxml.jackson.datatype.jsr310.deser.LocalDateDeserializer;
import com.fasterxml.jackson.datatype.jsr310.deser.LocalDateTimeDeserializer;
import com.fasterxml.jackson.datatype.jsr310.deser.LocalTimeDeserializer;
import com.fasterxml.jackson.datatype.jsr310.ser.LocalDateSerializer;
import com.fasterxml.jackson.datatype.jsr310.ser.LocalDateTimeSerializer;
import com.fasterxml.jackson.datatype.jsr310.ser.LocalTimeSerializer;
import org.springframework.boot.autoconfigure.jackson.Jackson2ObjectMapperBuilderCustomizer;
import org.springframework.http.converter.json.Jackson2ObjectMapperBuilder;
import org.springframework.stereotype.Component;

import java.time.LocalDate;
import java.time.LocalDateTime;
import java.time.LocalTime;
import java.time.format.DateTimeFormatter;

@Component
public class JacksonCustomizer implements Jackson2ObjectMapperBuilderCustomizer {

    private static final String STANDARD_PATTERN = "yyyy-MM-dd HH:mm:ss";

    private static final String DATE_PATTERN = "yyyy-MM-dd";

    private static final String TIME_PATTERN = "HH:mm:ss";

    @Override
    public void customize(Jackson2ObjectMapperBuilder builder) {
        // 初始化JavaTimeModule
        JavaTimeModule javaTimeModule = new JavaTimeModule();

        //处理LocalDateTime
        DateTimeFormatter dateTimeFormatter = DateTimeFormatter.ofPattern(STANDARD_PATTERN);
        javaTimeModule.addSerializer(LocalDateTime.class, new LocalDateTimeSerializer(dateTimeFormatter));
        javaTimeModule.addDeserializer(LocalDateTime.class, new LocalDateTimeDeserializer(dateTimeFormatter));

        //处理LocalDate
        DateTimeFormatter dateFormatter = DateTimeFormatter.ofPattern(DATE_PATTERN);
        javaTimeModule.addSerializer(LocalDate.class, new LocalDateSerializer(dateFormatter));
        javaTimeModule.addDeserializer(LocalDate.class, new LocalDateDeserializer(dateFormatter));

        //处理LocalTime
        DateTimeFormatter timeFormatter = DateTimeFormatter.ofPattern(TIME_PATTERN);
        javaTimeModule.addSerializer(LocalTime.class, new LocalTimeSerializer(timeFormatter));
        javaTimeModule.addDeserializer(LocalTime.class, new LocalTimeDeserializer(timeFormatter));

        /*
         * 1. java.util.Date yyyy-MM-dd HH:mm:ss
         * 2. 支持JDK8 LocalDateTime、LocalDate、 LocalTime
         * 3. Jdk8Module模块支持如Stream、Optional等类
         * 4. 序列化时包含所有字段
         * 5. 在序列化一个空对象时时不抛出异常
         * 6. 忽略反序列化时在json字符串中存在, 但在java对象中不存在的属性
         * 7. 数字序列化成字符穿且调用BigDecimal.toPlainString()方法
         */
        builder.simpleDateFormat(STANDARD_PATTERN)
                .modules(javaTimeModule, new Jdk8Module())
                .serializationInclusion(JsonInclude.Include.ALWAYS)
                .failOnEmptyBeans(false)
                .failOnUnknownProperties(false)
                .featuresToEnable(JsonGenerator.Feature.WRITE_BIGDECIMAL_AS_PLAIN);
    }
}

```

第三步，将`ElasticsearchClient`对象注册到Spring容器中

```java
package com.wangtao.msgsearch.config;

import co.elastic.clients.elasticsearch.ElasticsearchClient;
import co.elastic.clients.json.jackson.JacksonJsonpMapper;
import co.elastic.clients.transport.ElasticsearchTransport;
import co.elastic.clients.transport.rest_client.RestClientTransport;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.apache.http.HttpHost;
import org.elasticsearch.client.RestClient;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ElasticsearchConfig {

    @Bean
    public ElasticsearchClient elasticsearchClient(ObjectMapper objectMapper) {
        RestClient restClient = RestClient.builder(
                new HttpHost("localhost", 9200)).build();

        ElasticsearchTransport transport = new RestClientTransport(
                restClient, new JacksonJsonpMapper(objectMapper));

        return new ElasticsearchClient(transport);
    }
}

```

### API使用

现在就可以来感受下`Elasticsearch Java Client` API链式以及lambda风格

先准备测试数据

建立索引

```http
PUT /user
{
  "mappings": {
    "properties": {
      "userId": {
        "type": "integer",
        "index": true
      },
      "name": {
        "type": "text",
        "index": true
      },
      "age": {
        "type": "integer",
        "index": true
      },
      "birthday": {
        "type": "date",
        "index": true
      },
      "createTime": {
        "type": "date",
        "index": false,
        "format": "yyyy-MM-dd HH:mm:ss"
      }
    }
  }
}
```

插入数据

```http
POST /user/_bulk
{"index": {"_id": 1}}
{"userId": 1, "name" : "zhang san", "age" : 20, "birthday" : "2021-08-20", "createTime" : "2022-08-20 12:10:30"}
{"index": {"_id": 2}}
{"userId": 2, "name" : "li si", "age" : 20, "birthday" : "2022-08-20", "createTime" : "2022-08-20 12:10:30"}
{"index": {"_id": 3}}
{"userId": 3, "name" : "zhang san shuo", "age" : 25, "birthday" : "2022-08-20", "createTime" : "2022-08-20 12:10:30"}
{"index": {"_id": 4}}
{"userId": 4, "name" : "wang wu", "age" : 20, "birthday" : "2021-08-20", "createTime" : "2022-08-20 12:10:30"}
```

根据文档ID查询数据

lambda表达式

```java
 @GetMapping("/{docId}")
public User getByDocId(@PathVariable String docId) throws IOException {
    GetResponse<User> reponse = elasticsearchClient.get(
        g -> g.index("user").id(docId),
        User.class
    );
    if (reponse.found()) {
        return reponse.source();
    } else {
        throw new IllegalArgumentException("not found " + docId);
    }
}
```

Build模式

```java
@GetMapping("/getByDocIdFluent/{docId}")
public User getByDocIdFluent(@PathVariable String docId) throws IOException {
    GetRequest getRequest = new GetRequest.Builder()
        .index("user")
        .id(docId)
        .build();
    GetResponse<User> reponse = elasticsearchClient.get(getRequest, User.class);
    if (reponse.found()) {
        return reponse.source();
    } else {
        throw new IllegalArgumentException("not found " + docId);
    }
}
```

搜索操作

实现效果: `where name like '%zhang%' and age in (25, 20)`

```java
@GetMapping("/search")
public List<User> search() throws IOException {
    Query byName = new MatchQuery.Builder()
        .field("name")
        .query("zhang")
        .build()._toQuery();
    TermsQueryField termsQueryField = new TermsQueryField.Builder()
        .value(Arrays.asList(FieldValue.of(25), FieldValue.of(20)))
        .build();
    Query inAge = new TermsQuery.Builder()
        .field("age")
        .terms(termsQueryField)
        .build()._toQuery();
    Query query = new BoolQuery.Builder()
        .must(byName, inAge)
        .build()._toQuery();
    SearchRequest searchRequest = new SearchRequest.Builder()
        .index("user")
        .query(query)
        .build();
    SearchResponse<User> response = elasticsearchClient.search(
        searchRequest, User.class);
    List<Hit<User>> hits = response.hits().hits();
    return hits.stream().map(Hit::source).collect(Collectors.toList());
}
```

