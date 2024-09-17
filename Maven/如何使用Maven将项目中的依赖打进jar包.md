### 需求

有时候写一些库，需要其它三方依赖，但是又不想这个依赖影响到使用方，可以将这些三方依赖打到自己的项目jar包，并且更换包名，避免冲突(**更换包名之后，项目中的类引用第三方依赖的类import语句也会跟着变化**)。如Mybatis就使用了Ognl库，在打包时把Ognl的所有类都打到了Mybatis自己的jar中。

### 实现

实现的目标主要为两点

* 核心目标：将所需依赖打进自己的项目jar包
* 次要目标：将所需依赖的源码也同步打进自己项目的源码中，方便用户浏览源码。

要实现这两个需求需要使用到`maven-source-plugin`、`maven-jar-plugin`、`maven-shade-plugin`。

例子如下:

项目依赖`javassist`库，需要把`javassist`库打进自己项目jar包中，pom配置如下所示。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.wangtao</groupId>
  <artifactId>enhanced-thread-local</artifactId>
  <version>1.0.0</version>

  <properties>
    <maven.compiler.source>21</maven.compiler.source>
    <maven.compiler.target>21</maven.compiler.target>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>

  <dependencies>
    <!-- 三方依赖 -->
    <dependency>
      <groupId>org.javassist</groupId>
      <artifactId>javassist</artifactId>
      <version>3.30.2-GA</version>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-source-plugin</artifactId>
        <version>3.3.1</version>
        <executions>
          <execution>
            <id>attach-sources</id>
            <!-- 必须绑定到package阶段, 因为shade插件也是package阶段执行 -->
            <phase>package</phase>
            <goals>
              <goal>jar</goal>
            </goals>
            <configuration>
              <!-- 不要将构建的源码jar附加到构建产物中，由maven-shade-plugin来做 -->
              <attach>false</attach>
            </configuration>
          </execution>
        </executions>
      </plugin>

      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-jar-plugin</artifactId>
        <version>3.4.2</version>
      </plugin>

      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-shade-plugin</artifactId>
        <version>3.6.0</version>
        <executions>
          <execution>
            <id>shade-when-package</id>
            <phase>package</phase>
            <goals>
              <goal>shade</goal>
            </goals>
            <configuration>
              <!-- 创建source jar -->
              <createSourcesJar>true</createSourcesJar>
              <shadeSourcesContent>true</shadeSourcesContent>
              <dependencyReducedPomLocation>
                ${project.build.directory}/dependency-reduced-pom.xml
              </dependencyReducedPomLocation>
              
              <!-- 需要将哪些依赖打进项目jar包中 -->
              <artifactSet>
                <includes>
                  <include>org.javassist:javassist</include>
                </includes>
              </artifactSet>
              <filters>
                <filter>
                  <artifact>org.javassist:javassist</artifact>
                  <excludes>
                    <exclude>META-INF/MANIFEST.MF</exclude>
                  </excludes>
                </filter>
              </filters>
              <relocations>
                <!-- 将javassist及其子包打到com.wangtao.etl.agent.javassist包中 -->
                <relocation>
                  <pattern>javassist</pattern>
                  <shadedPattern>com.wangtao.etl.agent.javassist</shadedPattern>
                </relocation>
              </relocations>
            </configuration>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>
```

执行`mvn install`命令的工作流程如下(`mvn package`同理，只是少了将jar、sources.jar安装到本地仓库这一步)

第一步：`maven-jar-plugin`先将本项目打成jar包

第二步：`maven-source-plugin`将本项目的源码打成sources.jar

第三步：`maven-shade-plugin`将artifactSet参数项指定的依赖打进项目jar包中，如果有relocations配置，则根据对应配置调整依赖在jar包中的位置，并且改写import语句。最后替换掉第一步打好的原始jar。

第四步：`maven-shade-plugin`将artifactSet参数项指定的依赖源码打进项目sources.jar包中，这一步默认是关闭的，需要配置createSourcesJar参数为true，源码位置也受relocations配置影响。其中shadeSourcesContent参数用于控制是否改写源码中的import语句，默认也是false。最后替换掉第二步打好的原始sources.jar。

第五步：将jar、sources.jar安装到本地仓库中。

### 注意的点

#### maven-source-plugin

* 绑定的生命周期需要为`package`，不要设置成`verify`。因为`maven-shade-plugin`要将`maven-source-plugin`生成的项目源码包和自己生成的第三方依赖源码包合成最终的源码包。
* `attach`参数要设置成false，默认为true，否则会报错。参数含义：不要将构建的源码jar附加到构建产物中。
* 如果没有`maven-shade-plugin`插件，只是把项目源码安装到本地仓库，则绑定生命周期为`verify`、`attach`设置成true，是比较好的工程实践，这样子`mvn package`命令不会触发生成源码包的任务，因为生命周期顺序为package->verify->install。install的目标是构建产物，因此需要把`attach`设置成true，才会把源码包也会安装到本地仓库中。

#### maven-shade-plugin

* `createSourcesJar`参数需要设置成true，该参数用于控制是否生成第三方依赖的源码包，且合并到项目中的源码包中。
* `shadeSourcesContent`参数需要设置成true，如果调整了第三方依赖的位置(即包名发生变化)，同步调整项目源码中引用的第三方依赖类的import语句。
* `dependencyReducedPomLocation`默认为`project.basedir/dependency-reduced-pom.xml`，将其调整到target目录，这样git提交时会被忽略，一般target目录肯定是设置了ignore的。
* 使用`filter`排除掉第三方依赖的MANIFEST.MF文件，不然和项目自己的MANIFEST.MF产生冲突。



