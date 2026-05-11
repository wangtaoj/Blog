### 安装

```xml
<dependency>
  <groupId>com.baomidou</groupId>
  <artifactId>mybatis-plus-boot-starter</artifactId>
</dependency>
<!-- jdk8, 分页插件分离到这个依赖了 -->
<dependency>
  <groupId>com.baomidou</groupId>
  <artifactId>mybatis-plus-jsqlparser-4.9</artifactId>
</dependency>

<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>com.baomidou</groupId>
      <artifactId>mybatis-plus-bom</artifactId>
      <version>3.5.15</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

新版本分页插件挪到mybatis-plus-jsqlparser依赖中了，并且这个依赖分成两个名称，因为jsqlparser 5.0不再支持jdk8了。

可参考[官方文档-安装章节](https://baomidou.com/getting-started/install/)

### 分页

首先需要注册分页插件

```java
@Configuration(proxyBeanMethods = false)
public class MybatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        PaginationInnerInterceptor paginationInnerInterceptor = new PaginationInnerInterceptor();
        paginationInnerInterceptor.setDbType(DbType.MYSQL);
        interceptor.addInnerInterceptor(paginationInnerInterceptor);
        return interceptor;
    }

    @Bean
    public MetaObjectHandler metaObjectHandler() {
        return new FillMetaObjectHandler();
    }
}
```

#### 分页方法定义

方法的第一个参数必须是IPage对象

自定义方法

```java
IPage<User> list(IPage<User> page, User condition);
```

BaseMapper内置方法

```java
default <P extends IPage<T>> P selectPage(P page, @Param(Constants.WRAPPER) Wrapper<T> queryWrapper) {
    page.setRecords(selectList(page, queryWrapper));
    return page;
}
```

无论是内置方法还是自定义方法

如果返回类型是 IPage，则入参的 IPage 不能为 null，因为实际返回的IPage对象就是传入的参数IPage对象，会自动调用setRecords方法设置结果

如果返回类型是 List，则需要手动设置入参的 IPage.setRecords(返回的 List)。

#### 临时不分页方案

* 若返回类型是List，则入参IPage对象传null即可。分页插件会判断如果为空则不拦截。

* 通用方案，初始化IPage对象时size参数传负数即可。

  ```java
  IPage<User> page = Page.of(1, -1);
  ```

#### 只查数据不查count(但是需要分页)

将searchCount属性设置为false

```java
IPage<User> page = Page.of(1, 10, false);
```

#### 指定count方法

若不想要分页插件自动生成count方法，则指定Page对象的countId属性即可，countId可以不带名称空间，分页插件自己会拼接当前查询方法的名称空间。

适合复杂sql优化场景

### 自动填充字段

```java
public class FillMetaObjectHandler implements MetaObjectHandler {

    @Override
    public void insertFill(MetaObject metaObject) {
        /*
         * 字段名称、类型需要和实体中的字段名称、类型相匹配
         * 实体中对应字段存在@TableField(fill = FieldFill.INSERT)或@TableField(fill = FieldFill.INSERT_UPDATE)
         * 若实体中对应字段本身有值, 则不会去覆盖它
         * 使用lambda表达式可以避免填充值提前创建
         */
        this.strictInsertFill(metaObject, "createTime", LocalDateTime::now, LocalDateTime.class);
        this.strictInsertFill(metaObject, "updateTime", LocalDateTime::now, LocalDateTime.class);
    }

    @Override
    public void updateFill(MetaObject metaObject) {
        /*
         * 字段名称、类型需要和实体中的字段名称、类型相匹配
         * 实体中对应字段存在@TableField(fill = FieldFill.UPDATE)或@TableField(fill = FieldFill.INSERT_UPDATE)
         * 若实体中对应字段本身有值, 则不会去覆盖它
         * 使用lambda表达式可以避免填充值提前创建
         */
        this.strictUpdateFill(metaObject, "updateTime", LocalDateTime::now, LocalDateTime.class);
    }
}
```

实体

```java
/**
 * 创建时间
 * updateStrategy = FieldStrategy.NEVER: 更新方法会忽略createTime
 */
@TableField(fill = FieldFill.INSERT, updateStrategy = FieldStrategy.NEVER)
private LocalDateTime createTime;

/**
 * 修改时间
 */
@TableField(fill = FieldFill.INSERT_UPDATE)
private LocalDateTime updateTime;
```

FillMetaObjectHandler需要注入到Spring容器中，可以通过 `@Component` 或 `@Bean` 注解来实现。

注意事项

* 如果实体中属性本身有值不会被覆盖，如果填充值为 `null` 也不会去调用set方法进行填充赋值。
* 字段必须声明 `@TableField` 注解，并设置 `fill` 属性来选择填充策略。
* 在 `update(T entity, Wrapper<T> updateWrapper)` 时，`entity` 不能为空，否则自动填充失效。

- 在 `update(Wrapper<T> updateWrapper)` 时不会自动填充，需要手动赋值字段条件。

* `delete`操作在符合逻辑删除时(变成了update)也会自动填充。

* 对于自定义方法，多参数时参数名必须是et

  ```java
  // 单参数
  @Update("update user set age = #{age}, update_time = #{updateTime} where id = #{id}")
  int updateAgeById(User user);
  
  /*
   * 单参数带@Param注解或者多参数，参数名称为et的才会被填充
   * 因为带@Param，底层真实的参数对象变成了map，需要一个标记来确定到底要填充map中的哪一个参数
   */
  @Update("update user set age = #{et.age}, update_time = #{et.updateTime} where id = #{et.id}")
  int updateAgeByIdWithParamName(@Param(Constants.ENTITY) User user);
  ```

### 逻辑删除

全局配置

```yaml
mybatis-plus:
  global-config:
    db-config:
      logicDeleteField: delFlg
      logic-delete-value: 1
      logic-not-delete-value: 0
```

全局配置时必须实体类中有这个字段才会生效，没有则无效。

实体局部配置

```java
@TableLogic(value = "0", delval = "1")
private Integer delFlg;
```

简要工作原理

- **插入**：逻辑删除字段的值不受限制。
- **查找**：自动添加条件，过滤掉标记为已删除的记录。
- **更新**：防止更新已删除的记录。
- **删除**：将删除操作转换为更新操作，标记记录为已删除。

例如：

- **删除**：`update user set deleted=1 where id = 1 and del_flg = 0`
- **查找**：`select id,name,deleted from user where del_flg = 0`
- **修改**: `update user set name = 'xx' where del_flg = 0`

对于插入

- **方法一**：在数据库中为逻辑删除字段设置默认值。
- **方法二**：在插入数据前手动设置逻辑删除字段的值。
- **方法三**：使用 MyBatis-Plus 的自动填充功能。

**注意：逻辑删除对自定义Mapper方法无效。**

### insert或者update方法默认拼接字段规则

当实体中字段值不为null时，就会包含在sql中。

可通过`@TableField`注解的`insertStrategy`以及`updateStrategy`指定。默认值为DEFAULT，因为全局配置的默认值也是DEFAULT，所以效果等同于NOT_NULL

```java
public enum FieldStrategy {
    /**
     * 任何时候都加入 SQL
     */
    ALWAYS,
    /**
     * 非NULL判断
     */
    NOT_NULL,
    /**
     * 非空判断(只对字符串类型字段,其他类型字段依然为非NULL判断)
     */
    NOT_EMPTY,
    /**
     * 默认的,一般只用于注解里
     * <p>1. 在全局里代表 NOT_NULL</p>
     * <p>2. 在注解里代表 跟随全局</p>
     */
    DEFAULT,
    /**
     * 不加入 SQL
     */
    NEVER
}
```

因此如果想要修改字段值为null，最好使用wrapper

```java
/*
 * UPDATE user SET age=null WHERE del_flg=0 AND (id = 28)
 */
ChainWrappers.lambdaUpdateChain(userMapper)
            .set(User::getAge, null)
            .eq(User::getId, 28)
            .update();
```

### 批量新增修改方法

```java
default List<BatchResult> insert(Collection<T> entityList) {
    return insert(entityList, Constants.DEFAULT_BATCH_SIZE);
}

default List<BatchResult> updateById(Collection<T> entityList) {
    return updateById(entityList, Constants.DEFAULT_BATCH_SIZE);
}
```

可以和别的方法在一个事务中共存，批量方法是新开了一个ExecutorType=BATCH的DefaultSqlSession，注意不能是SqlSessionTemplate，如果是SqlSessionTemplate，底层执行时会去拿绑定在ThreadLocal中的DefaultSqlSession，其中会校验ExecutorType，不能进行修改。而新开一个DefaultSqlSession，由于Mybatis中的Transaction对象为SpringManagedTransaction，因此底层的Connection是共享的，由Spring事务管理器绑定在ThreadLocal中，因此能共享同一个事物，且这个新开的DefaultSqlSession调用commit时是没有任何效果的，由Spring事务管理器统一提交和回滚事务。