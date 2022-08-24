### 背景

有时候使用`insert into xxx values (),()`语句插入大量数据时，会使得SQL语句超长，为了解决这个问题，在Mybatis中编写一个分批次插入的插件。

### 实现

```java
package com.wangtao.plugin.interceptor;

import org.apache.ibatis.executor.Executor;
import org.apache.ibatis.mapping.MappedStatement;
import org.apache.ibatis.plugin.*;
import org.apache.ibatis.reflection.MetaObject;
import org.apache.ibatis.session.Configuration;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.sql.SQLException;
import java.util.ArrayList;
import java.util.List;
import java.util.Properties;

/**
 * 针对SQL insert into values(), ()语句, 为了避免大量数据使得SQL语句超长, 分批次插入
 * 使用方式: 插入集合的name为_batchList, 最大的数目name为_maxCount
 * <pre> {@code
 * int batchInsert(@Param("_batchList")List<User> users, @Param("_maxCount")int maxCount);
 * }</pre>
 * Created at 2022/8/24 21:24
 */
@Intercepts({@Signature(type = Executor.class, method = "update", args = {MappedStatement.class, Object.class})})
public class BatchInsertInterceptor implements Interceptor {

    /**
     * 参数名字
     */
    public static final String DEFAULT_DATA_NAME = "_batchList";

    /**
     * 一次插入的最大条数参数名
     */
    public static final String DEFAULT_MAX_COUNT_NAME = "_maxCount";

    private static final int DEFAULT_MAX_COUNT = 100;

    private final Logger logger = LoggerFactory.getLogger(BatchInsertInterceptor.class);

    private String dataName = DEFAULT_DATA_NAME;

    private String maxCountName = DEFAULT_MAX_COUNT_NAME;

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        Object[] args = invocation.getArgs();
        MappedStatement ms = (MappedStatement) args[0];
        Object parameter = args[1];
        Executor executor = (Executor) invocation.getTarget();
        Configuration configuration = ms.getConfiguration();
        MetaObject metaObject = configuration.newMetaObject(parameter);
        if (metaObject.hasGetter(dataName)) {
            Object dataList = metaObject.getValue(dataName);
            int maxCount = DEFAULT_MAX_COUNT;
            if (metaObject.hasGetter(maxCountName)) {
                maxCount = (Integer) metaObject.getValue(maxCountName);
            }
            if (dataList instanceof List) {
                List<?> list = (List<?>) dataList;
                if (list.size() > maxCount) {
                    logger.info("================proxy executor.update method.========");
                    return batchInsertData(executor, ms, metaObject, list, maxCount);
                }
            }
        }
        // not proxy
        return invocation.proceed();
    }

    private Object batchInsertData(Executor executor, MappedStatement ms, MetaObject metaObject,
                                   List<?> dataList, int maxCount) throws SQLException {
        int updateCount = 0;
        List<Object> temp = new ArrayList<>();
        for (int i = 0; i < dataList.size(); i++) {
            if (i != 0 && i % maxCount == 0) {
                metaObject.setValue(dataName, temp);
                updateCount += executor.update(ms, metaObject.getOriginalObject());
                temp.clear();
            }
            temp.add(dataList.get(i));
        }
        if (!temp.isEmpty()) {
            updateCount += executor.update(ms, metaObject.getOriginalObject());
        }
        // 还原参数
        metaObject.setValue(dataName, dataList);
        return updateCount;
    }

    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    @Override
    public void setProperties(Properties properties) {
        if (properties != null) {
            this.dataName = properties.getProperty("dataName", DEFAULT_DATA_NAME);
            this.maxCountName = properties.getProperty("maxCountName", DEFAULT_MAX_COUNT_NAME);
        }
    }
}

```

### 使用

在Spring Boot项目中，如果引用了以下依赖

```xml
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
</dependency>
```

将该插件注入到容器中即可

```java
package com.wangtao.plugin.config;

import com.wangtao.plugin.interceptor.BatchInsertInterceptor;
import org.apache.ibatis.plugin.Interceptor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MybatisConfig {

    @Bean
    public Interceptor batchInsertInterceptor() {
        return new BatchInsertInterceptor();
    }
}

```

接口层

```java
@Mapper
public interface ExampleMapper {

    /**
     * 批量插入
     * @param exampleList 记录列表
     * @param maxCount 一次最大的插入条数
     * @return 成功的条数
     */
    int batchInsert(@Param(BatchInsertInterceptor.DEFAULT_DATA_NAME) List<Example> exampleList,
                    @Param(BatchInsertInterceptor.DEFAULT_MAX_COUNT_NAME) int maxCount);
}
```

