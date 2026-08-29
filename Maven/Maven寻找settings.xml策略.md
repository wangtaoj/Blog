Maven 查找 `settings.xml` 时，可以理解为 **两层配置：全局（Global）+ 用户级（User）**，其中用户级优先。

**注：相同的按照优先级生效，不同的会进行合并，不是单存的文件级覆盖。全局文件和用户级文件都会进行加载(如果存在的话)**

### 1. 默认位置

| 级别   | 默认路径                          | 优先级 |
| ------ | --------------------------------- | ------ |
| 用户级 | `${user.home}/.m2/settings.xml`   | **高** |
| 全局级 | `${maven.home}/conf/settings.xml` | 低     |

例如 macOS：

```
~/.m2/settings.xml
```

以及 Maven 安装目录：

```
/opt/maven/apache-maven-3.9.x/conf/settings.xml
```

### 2. Maven 实际加载顺序

Maven 默认会加载：

```
${maven.home}/conf/settings.xml
        ↓
${user.home}/.m2/settings.xml
```

可以把它理解成：

```
Global settings
      +
User settings
      ↓
Effective settings
```

**用户级配置覆盖全局级配置。**

例如全局：

```
<settings>
    <localRepository>/data/maven/repository</localRepository>
</settings>
```

用户：

```
<settings>
    <localRepository>/Users/xxx/.m2/repository</localRepository>
</settings>
```

最终使用用户级的：

```
/Users/xxx/.m2/repository
```

------

### 3. `-s` 可以指定用户 settings

执行：

```
mvn -s /tmp/settings.xml clean install
```

此时指定的 `settings.xml` 会作为 **User Settings**。

也就是说：

```
mvn -s xxx/settings.xml
```

相当于告诉 Maven：

> 不使用默认的 `~/.m2/settings.xml`，使用我指定的这个文件作为用户级 settings。

### 4. `-gs` 可以指定全局 settings

还有一个容易混淆的参数：

```
mvn -gs /tmp/global-settings.xml clean install
```

它指定的是：

```
Global Settings
```

所以：

```
mvn -s user.xml -gs global.xml
```

可以同时指定两套配置：

```
global.xml
    ↓
user.xml
    ↓
最终 Effective Settings
```

### 5. 一个比较完整的优先级关系

可以记成：

```
                    Maven 参数
                       │
              ┌────────┴────────┐
              │                 │
             -gs               -s
              │                 │
              ▼                 ▼
        Global Settings     User Settings
              │                 │
              │                 │
              └────────┬────────┘
                       ↓
              Effective Settings
```

默认情况下则是：

```
${maven.home}/conf/settings.xml
              │
              ▼
${user.home}/.m2/settings.xml
              │
              ▼
      Effective Settings
```

### 6. 怎么确认 Maven 到底用了哪个？

最实用的是：

```
mvn help:effective-settings
```

它会把 Maven 最终合并后的配置打印出来。

如果想看 Maven 的环境信息：

```
mvn -version
```

可以看到类似：

```
Apache Maven 3.9.x
Maven home: /xxx/apache-maven-3.9.x
Java version: 21
```

那么全局 settings 就可以推断为：

```
/xxx/apache-maven-3.9.x/conf/settings.xml
```

### 7. 一个很容易踩坑的地方

**IDEA / Eclipse / Jenkins / CI 使用的 Maven，不一定是你命令行的 Maven。**

例如命令行：

```
mvn -version
```

显示：

```
Maven home: /opt/maven
```

那么它找：

```
/opt/maven/conf/settings.xml
```

但 IDEA 可能配置了自己的 Maven：

```
Settings
  → Build, Execution, Deployment
  → Build Tools
  → Maven
```

甚至 IDEA 可以直接指定：

```
User settings file:
    /xxx/settings.xml
```

这时你会发现：

```
mvn clean package
```

和 IDEA 点击 Maven 构建，使用的 `settings.xml` 不一样。