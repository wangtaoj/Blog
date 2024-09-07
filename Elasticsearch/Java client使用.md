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

第二步，将`ElasticsearchClient`对象注册到Spring容器中

ElasticsearchConfig.java

```java
@Configuration
public class ElasticsearchConfig {

    private static final String STANDARD_PATTERN = "yyyy-MM-dd HH:mm:ss";

    @Bean
    public ElasticsearchClient elasticsearchClient() {
        RestClient restClient = RestClient.builder(
                new HttpHost("localhost", 9200)).build();

        /*
         * 7.x版本, JacksonJsonpMapper构造方法会修改传入的ObjectMapper属性
         * 切记不要使用Spring容器中的ObjectMapper，否则会影响SpringMVC JSON序列化行为
         */
        ElasticsearchTransport transport = new RestClientTransport(
                restClient, new JacksonJsonpMapper(createObjectMapper()));

        return new ElasticsearchClient(transport);
    }

    private ObjectMapper createObjectMapper() {
        ObjectMapper objectMapper = new ObjectMapper();

        // 设置全局的DateFormat
        objectMapper.setDateFormat(new SimpleDateFormat(STANDARD_PATTERN));

        // 设置全局的时区, Jackson默认值为UTC
        objectMapper.setTimeZone(TimeZone.getDefault());

        // 初始化JavaTimeModule
        JavaTimeModule javaTimeModule = JavaTimeModuleUtils.create();

        // 注册模块
        objectMapper.registerModule(javaTimeModule);

        // 注册JDK新增的一些类型, 比如Optional
        objectMapper.registerModule(new Jdk8Module());

        // 包含所有字段
        objectMapper.setSerializationInclusion(JsonInclude.Include.ALWAYS);

        // 在序列化一个空对象时时不抛出异常
        objectMapper.disable(SerializationFeature.FAIL_ON_EMPTY_BEANS);

        // 忽略反序列化时在json字符串中存在, 但在java对象中不存在的属性
        objectMapper.disable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES);

        // BigDecimal.toPlainString(), 这样不会有科学计数法
        objectMapper.enable(JsonGenerator.Feature.WRITE_BIGDECIMAL_AS_PLAIN);
        return objectMapper;
    }
}
```

JavaTimeModuleUtils.java

```java
public final class JavaTimeModuleUtils {

    public static final String STANDARD_PATTERN = "yyyy-MM-dd HH:mm:ss";

    public static final String DATE_PATTERN = "yyyy-MM-dd";

    public static final String TIME_PATTERN = "HH:mm:ss";

    private JavaTimeModuleUtils() {}

    public static JavaTimeModule create() {
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
        return javaTimeModule;
    }
}
```

### 客户端请求日志

如果想要查看客户端发送给服务端的具体请求信息，需要打开以下日志，这个对于开发调试时很有帮助，生产模式务必关闭，因为不仅会打印请求参数日志，还会把服务端的响应信息也打印出来。

具体可参见`RestClient.convertResponse()`、`RequestLogger.logResponse()`两个方法。

只打印请求行

开启日志`org.elasticsearch.client.RestClient=debug`

结果:

```text
request [POST http://localhost:9200/user/_search?typed_keys=true] returned [HTTP/1.1 200 OK]
```

打印完整请求信息与响应信息

开启日志`tracer=trace`，loggerName=tracer, level=trace

结果:

```text
curl -iX POST 'http://localhost:9200/user/_search?typed_keys=true' -d '{"query":{"match":{"name":{"query":"user_1"}}}}'
# HTTP/1.1 200 OK
# X-elastic-product: Elasticsearch
# Warning: 299 Elasticsearch-7.17.5-8d61b4f7ddf931f219e3745f295ed2bbc50c8e84 "Elasticsearch built-in security features are not enabled. Without authentication, your cluster could be accessible to anyone. See https://www.elastic.co/guide/en/elasticsearch/reference/7.17/security-minimal-setup.html to enable security."
# content-type: application/json; charset=UTF-8
# content-length: 333
#
# {"took":1,"timed_out":false,"_shards":{"total":1,"successful":1,"skipped":0,"failed":0},"hits":{"total":{"value":1,"relation":"eq"},"max_score":0.9808291,"hits":[{"_index":"user","_type":"_doc","_id":"1","_score":0.9808291,"_source":{"userId":1,"name":"user_1","age":30,"birthday":"1997-05-02","createTime":"2024-01-13 14:31:18"}}]}}
```

### API使用

现在就可以来感受下`Elasticsearch Java Client` API链式以及lambda风格

#### 索引操作

```java
@Slf4j
@SpringBootTest
public class IndexApiTest {

    private static final String INDEX = "user";

    @Autowired
    private ElasticsearchClient esClient;

    /**
     * 所有的API提供两种形式, Builder以及lambda
     */
    @Test
    public void testCreate() throws IOException {
        BooleanResponse existsResponse = esClient.indices()
                .exists(builder -> builder.index(INDEX));
        if (existsResponse.value()) {
            // 存在则先删除
            DeleteIndexResponse deleteResponse = esClient.indices()
                    .delete(builder -> builder.index(INDEX));
            if (deleteResponse.acknowledged()) {
                log.info("index '{}' is deleted!", INDEX);
            }
        }
        // 新增索引, 其中birthday、createTime不能作为搜索条件
        TypeMapping typeMapping = new TypeMapping.Builder()
                .dynamic(DynamicMapping.Strict)
                .properties("userId", p -> p.integer(i -> i))
                .properties("name", p -> p.text(t -> t))
                .properties("age", p -> p.integer(i -> i))
                .properties("birthday", p -> p.date(d -> d.index(false).format("yyyy-MM-dd")))
                .properties("createTime", p -> p.date(d -> d.index(false).format("yyyy-MM-dd HH:mm:ss")))
                .build();
        esClient.indices().create(new CreateIndexRequest.Builder()
                .index(INDEX)
                .mappings(typeMapping)
                .build()
        );
    }

    /**
     * 根据JSON内容创建索引
     */
    @Test
    public void testCreateByJson() throws IOException {
        BooleanResponse existsResponse = esClient.indices()
                .exists(builder -> builder.index(INDEX));
        if (existsResponse.value()) {
            // 存在则先删除
            DeleteIndexResponse deleteResponse = esClient.indices()
                    .delete(builder -> builder.index(INDEX));
            if (deleteResponse.acknowledged()) {
                log.info("index '{}' is deleted!", INDEX);
            }
        }
        String mappings = """
                {
                  "mappings" : {
                    "dynamic" : "strict",
                    "properties" : {
                      "age" : {
                        "type" : "integer"
                      },
                      "birthday" : {
                        "type" : "date",
                        "index" : false,
                        "format" : "yyyy-MM-dd"
                      },
                      "createTime" : {
                        "type" : "date",
                        "index" : false,
                        "format" : "yyyy-MM-dd HH:mm:ss"
                      },
                      "name" : {
                        "type" : "text"
                      },
                      "userId" : {
                        "type" : "integer"
                      }
                    }
                  }
                }
                """;
        esClient.indices().create(new CreateIndexRequest.Builder()
                .index(INDEX)
                .withJson(new StringReader(mappings))
                .build()
        );
    }

    /**
     * mapping操作
     * getMapping() 查询
     * putMapping() 添加
     */
    @Test
    public void testGetMapping() throws IOException {
        GetMappingResponse mappingResponse = esClient.indices().getMapping(builder -> builder.index(INDEX));
        IndexMappingRecord indexMappingRecord = mappingResponse.get(INDEX);
        TypeMapping typeMapping = indexMappingRecord.mappings();
        log.info("dynamic: {}", typeMapping.dynamic());
        log.info("mappings: {}", typeMapping);
    }

    /**
     * 分词
     */
    @Test
    public void testAnalyze() throws IOException {
        AnalyzeResponse analyzeResponse = esClient.indices().analyze(
                new AnalyzeRequest.Builder()
                        .index(INDEX)
                        .field("name")
                        .text("wang tao")
                        .build()
        );
        analyzeResponse.tokens().forEach(token -> log.info("{}", token));
    }
}
```

#### 文档操作

```java
@Slf4j
@SpringBootTest
public class DocApiTest {

    private static final String INDEX = "user";

    @Autowired
    private ElasticsearchClient esClient;

    /**
     * 新增或者修改
     * 修改时会修改所有字段
     * POST /user/_doc/1
     */
    @Test
    public void testCreateOrUpdate() throws IOException {
        User user = new User()
                .setUserId(1)
                .setName("zhang san")
                .setAge(30)
                .setBirthday(LocalDate.of(1997, 5, 2))
                .setCreateTime(LocalDateTime.now());
        esClient.index(new IndexRequest.Builder<User>()
                .index(INDEX)
                .id(String.valueOf(user.getUserId()))
                .document(user)
                .build()
        );
    }

    /**
     * 新增文档, 存在则报错
     * POST /user/_create/1
     */
    @Test
    public void testInsert() throws IOException {
        User user = new User()
                .setUserId(1)
                .setName("zhang san")
                .setAge(30)
                .setBirthday(LocalDate.of(1997, 5, 2))
                .setCreateTime(LocalDateTime.now());
        esClient.create(new CreateRequest.Builder<User>()
                .index(INDEX)
                .id(String.valueOf(user.getUserId()))
                .document(user)
                .build()
        );
    }

    /**
     * 修改文档, 不存在则报错
     * 支持只修改指定字段
     * POST /user/_update/1
     */
    @Test
    public void testUpdate() throws IOException {
        // 将年龄改成35
        Map<String, Object> updateDoc = Map.of("age", 35);
        esClient.update(
                new UpdateRequest.Builder<User, Map<String, Object>>()
                        .index(INDEX)
                        .id("1")
                        .doc(updateDoc)
                        .build(),
                User.class
        );
    }

    /**
     * 删除文档
     * DELETE /user/_doc/1
     */
    @Test
    public void testDelete() throws IOException {
        esClient.delete(new DeleteRequest.Builder()
                .index(INDEX)
                .id("1")
                .build()
        );
    }

    /**
     * 批量操作(_bulk api)
     * index: 新增或修改
     * create: 新增
     * update: 修改
     * delete: 删除
     * 注意: _bulk api批量操作每个操作互不影响, 报错不会影响别的记录, 会返回每个操作的结果是成功还是失败
     */
    @Test
    public void testBatch() throws IOException {
        List<User> userList = generateData();
        List<BulkOperation> bulkOperations = new ArrayList<>();
        for (User user : userList) {
            BulkOperation bulkOperation;
            // 当id=2时, 为update操作, 其余都为index操作
            if (user.getUserId().equals(2)) {
                UpdateAction<User, User> updateAction = new UpdateAction.Builder<User, User>()
                        .doc(user)
                        .build();
                UpdateOperation<User, User> updateOperation = new UpdateOperation.Builder<User, User>()
                        .id(String.valueOf(user.getUserId()))
                        .action(updateAction)
                        .build();
                bulkOperation = new BulkOperation.Builder()
                        .update(updateOperation)
                        .build();
            } else {
                bulkOperation = new BulkOperation.Builder()
                        .index(i -> i.id(String.valueOf(user.getUserId())).document(user))
                        .build();
            }
            bulkOperations.add(bulkOperation);
        }
        BulkRequest bulkRequest = new BulkRequest.Builder()
                .index(INDEX)
                .operations(bulkOperations)
                .build();
        BulkResponse response = esClient.bulk(bulkRequest);
        // 是否存在错误, 只要有一个操作出错则返回true
        if (response.errors()) {
            log.error("batch operations has error!");
        }
    }

    private List<User> generateData() {
        List<User> userList = new ArrayList<>();
        for (int i = 1; i <= 3; i++) {
            User user = new User()
                    .setUserId(i)
                    .setName("user_" + i)
                    .setAge(30)
                    .setBirthday(LocalDate.of(1997, 5, 2))
                    .setCreateTime(LocalDateTime.now());
            userList.add(user);
        }
        return userList;
    }

    /**
     * 根据文档id查询
     */
    @Test
    public void testGetByDocId() throws IOException {
        final String docId = "1";
        GetResponse<User> reponse = esClient.get(
                g -> g.index(INDEX).id(docId),
                User.class
        );
        if (reponse.found()) {
            User user = reponse.source();
            System.out.println(user);
        } else {
            throw new IllegalArgumentException("not found " + docId);
        }
    }

    /**
     * 根据文档id查询
     */
    @Test
    public void testGetByDocIdFluent() throws IOException {
        final String docId = "1";
        GetRequest getRequest = new GetRequest.Builder()
                .index(INDEX)
                .id(docId)
                .build();
        GetResponse<User> reponse = esClient.get(getRequest, User.class);
        if (reponse.found()) {
            User user = reponse.source();
            System.out.println(user);
        } else {
            throw new IllegalArgumentException("not found " + docId);
        }
    }

    /**
     * 复杂搜索
     */
    @Test
    public void testSearch() throws IOException {
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
                .index(INDEX)
                .query(query)
                .build();
        SearchResponse<User> response = esClient.search(searchRequest, User.class);
        List<Hit<User>> hits = response.hits().hits();
        List<User> users = hits.stream().map(Hit::source).toList();
        users.forEach(System.out::println);
    }

    /**
     * 通过JSON构造查询条件
     */
    @Test
    public void testSearchByJson() throws IOException {
        String queryJson = """
                {
                  "query": {
                    "match": {
                      "name": "user_1"
                    }
                  }
                }
                """;
        SearchRequest searchRequest = new SearchRequest.Builder()
                .index(INDEX)
                .withJson(new StringReader(queryJson))
                .build();
        SearchResponse<User> response = esClient.search(searchRequest, User.class);
        List<Hit<User>> hits = response.hits().hits();
        List<User> users = hits.stream().map(Hit::source).toList();
        users.forEach(System.out::println);
    }
}
```

