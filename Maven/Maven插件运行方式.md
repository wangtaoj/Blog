### 执行方式

maven插件执行的最小单元为goal，一个插件可以有多个goal。执行的方式主要为以下两种

* 由maven生命周期驱动，需要插件相应的goals绑定到具体的生命周期阶段中。

  **需要插件和要执行的goal显示声明到pom文件中。**

  ```bash
  # 可通过-D传递参数给插件的goal
  mvn package -Dkey=value
  ```

* 通过mvn命令显示执行，**插件可以不用声明到pom文件中，从仓库中查找。**

  语法形如`mvn groupId:artifactId:version:goal -Dkey=value`

  groupId:artifactId:version为插件的坐标，其中version可以省略，若项目pom没有声明这个插件，则使用仓库最新的版本。

  特别的，maven还支持短名称写法，不过需要插件命名(artifactId)遵循一定规则，形如xxx-maven-plugin，官方的插件为maven-xxx-plugin。这样命令可以变成`mvn xxx:goal -Dkey=value`。**使用短名称命令时非官方插件需要在项目pom文件中声明插件，否则需要配置插件前缀才能使用。**

  ```bash
  # 执行插件的hello这个goal
  mvn com.wangtao:demo-maven-plugin:1.0.0:hello
  # 短名称形式, 由于插件遵循了maven的命名机制
  mvn demo:hello
  ```

  短名称形式需要在pom声明插件，不需要声明goal，goal由命令明确指定。

  ```xml
  <build>
    <plugins>
      <plugin>
        <groupId>com.wangtao</groupId>
        <artifactId>demo-maven-plugin</artifactId>
        <version>1.0-SNAPSHOT</version>
      </plugin>
    </plugins>
  </build>
  ```


### 插件命名

官方插件命名形如maven-xxx-plugin，比如maven-jar-plugin、maven-clean-plugin。

非官方插件命名形如xxx-maven-plugin，比如spring-boot-maven-plugin。

遵循这样的命名规则可以使用短名称执行插件。

### 插件的goal绑定生命周期

两种方式

* 可以在编写goal时使用`@Mojo`注解的`defaultPhase`属性绑定一个生命周期，要想goal执行，还是需要在pom文件中声明这个goal才行，官方的一些插件由于在super pom已经声明，所以可以不用写。

  ```java
  @Mojo(name = "bindPhase", defaultPhase = LifecyclePhase.PACKAGE)
  public class BindPhaseMojo extends AbstractMojo {
  }
  ```

  pom使用如下

  ```xml
  <build>
    <plugins>
      <plugin>
        <groupId>com.wangtao</groupId>
        <artifactId>demo-maven-plugin</artifactId>
        <version>1.0-SNAPSHOT</version>
        <executions>
          <execution>
            <id>default-bindPhase</id>
            <!-- 声明要执行的goal，可以不用指定phase标签 -->
            <goals>
              <goal>bindPhase</goal>
            </goals>
            <configuration>
              <greeting>hello maven plugin</greeting>
            </configuration>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
  ```

  

* 在pom文件中指定，可以覆盖掉默认的绑定值

  ```xml
  <build>
    <plugins>
      <plugin>
        <groupId>com.wangtao</groupId>
        <artifactId>demo-maven-plugin</artifactId>
        <version>1.0-SNAPSHOT</version>
        <executions>
          <execution>
            <id>default-bindPhase</id>
            <!-- 使用phase标签明确绑定生命周期 -->
            <phase>package</phase>
            <goals>
              <goal>bindPhase</goal>
            </goals>
            <configuration>
              <greeting>hello maven plugin</greeting>
            </configuration>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
  ```

### 使用插件名的命令方式执行插件中的goal

有时执行插件时需要前置任务，怎么触发呢，比如spring-boot-maven-plugin:run这个goal，它需要运行程序，势必要触发编译、打包。

可以通过`@Execute`注解来实现这个效果

```java
@Mojo(name = "hello")
@Execute(phase = LifecyclePhase.PACKAGE)
public class HelloMojo extends AbstractMojo {
}
```

假设插件名为demo-maven-plugin。

当执行`mvn demo:hello`时，先build到package阶段(这会触发中间各个阶段绑定的goal), 然后执行hello这个goal。

### 参数

参数可以配置在项目pom文件中，也可以在执行时通过-D动态传参。

参数由`@Parameter`注解标记，有如下几个属性

name：参数名字，不写就是java字段名称

property: jvm系统参数对应的key，可以通过-D动态传参

defaultValue：默认值

required：参数是否必输

比如下面插件的一个goal

```java
@Mojo(name = "bindPhase", defaultPhase = LifecyclePhase.PACKAGE)
public class BindPhaseMojo extends AbstractMojo {

    @Parameter(property = "sayhi.greeting", defaultValue = "Hello World!" )
    private String greeting;

    public void execute() {
        getLog().info("====================BindPhaseMojo execute===================");
        getLog().info(greeting);
    }

    public void setGreeting(String greeting) {
        this.greeting = greeting;
    }
}
```

pom文件配置

```xml
<build>
  <plugins>
    <plugin>
      <groupId>com.wangtao</groupId>
      <artifactId>demo-maven-plugin</artifactId>
      <version>1.0-SNAPSHOT</version>
      <executions>
        <execution>
          <id>default-bindPhase</id>
          <goals>
            <goal>bindPhase</goal>
          </goals>
          <configuration>
            <!-- 指定参数值，所有的参数在configuration标签下面 -->
            <greeting>hello maven plugin</greeting>
          </configuration>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>
```

-D动态传参形式

对应的pom文件

```xml
<build>
  <plugins>
    <plugin>
      <groupId>com.wangtao</groupId>
      <artifactId>demo-maven-plugin</artifactId>
      <version>1.0-SNAPSHOT</version>
      <executions>
        <execution>
          <id>default-bindPhase</id>
          <goals>
            <goal>bindPhase</goal>
          </goals>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>
```

```bash
# 通过-D参数动态传参(该插件默认绑定到package生命周期)
mvn package -Dsayhi.greeting=hello
```

**注意：pom文件若明确指定了参数，再通过-D动态传参，是不会覆盖掉pom文件明确指定的值的。**

### FAQ

1. 当`<execution>下`标签中的`<id>`没有指定值时，默认值为default。

2. 每一个`<execution>`标签下可以声明多个`<goal>`，他们共享同一个`<configuration>`、`<phase>`，因此如果需要不一样的参数，将`<goal>`放到不同的`<execution>`下，一个插件可以配置多个`<execution>`。

3. `<plugin>`标签下的`<configuration>`参数配置是针对声明的所有的goal参数配置，不过`<execution>`标签下的`<configuration>`优先级更高。

   ```xml
   <build>
     <plugins>
       <plugin>
         <groupId>com.wangtao</groupId>
         <artifactId>demo-maven-plugin</artifactId>
         <version>1.0-SNAPSHOT</version>
         <executions>
           <execution>
             <id>default-bindPhase</id>
             <phase>package</phase>
             <goals>
               <goal>bindPhase</goal>
             </goals>
             <!-- 参数局部配置，优先级更高 -->
             <configuration>
               <greeting1>hello maven plugin</greeting1>
             </configuration>
           </execution>
         </executions>
         <!-- 插件参数全局配置 -->
         <configuration>
           <greeting1>hello1</greeting1>
           <greeting2>hello2</greeting2>
         </configuration>
       </plugin>
     </plugins>
   </build>
   ```

4. 命令行-D动态传参不会覆盖掉pom文件中配置的参数值。

5. 默认绑定生命周期的goal，也要在pom文件中`execution`显示声明要运行的goal，只不过是`<phase>`标签可以省略。官方的插件之所以可以不用写，是因为super pom已经做了这个事情。
