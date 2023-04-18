# presto 自定义函数简述


presto自带unbase64函数某些时候会报错，所以想要自定义一个unbase64函数。

<!--more-->

presto自带unbase64函数，用法如下。

```
FROM_UTF8(from_base64(nickname))
```

但是有些字符会报错。

```
Query failed (#20220720_091551_00087_mkhun): Illegal base64 character -1a
```

## idea新建maven项目
pom文件如下：
```xml
    <dependencies>
        <dependency>
            <groupId>com.facebook.presto</groupId>
            <artifactId>presto-spi</artifactId>
            <version>0.272</version>
        </dependency>
        <dependency>
            <groupId>com.google.guava</groupId>
            <artifactId>guava</artifactId>
            <version>18.0</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-jar-plugin</artifactId>
                <configuration>
                    <classesDirectory>target/classes/</classesDirectory>
                    <archive>
                        <manifest>
                            <mainClass>com.alibaba.dubbo.container.Main</mainClass>
                            <!-- 打包时 MANIFEST.MF文件不记录的时间戳版本 -->
                            <useUniqueVersions>false</useUniqueVersions>
                            <addClasspath>true</addClasspath>
                            <classpathPrefix>lib/</classpathPrefix>
                        </manifest>
                        <manifestEntries>
                            <Class-Path>.</Class-Path>
                        </manifestEntries>
                    </archive>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-dependency-plugin</artifactId>
                <executions>
                    <execution>
                        <id>copy-dependencies</id>
                        <phase>package</phase>
                        <goals>
                            <goal>copy-dependencies</goal>
                        </goals>
                        <configuration>
                            <type>jar</type>
                            <includeTypes>jar</includeTypes>
                            <outputDirectory>
                                ${project.build.directory}/lib
                            </outputDirectory>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>com.facebook.presto</groupId>
                <artifactId>presto-maven-plugin</artifactId>
                <version>0.3</version>
                <extensions>true</extensions>
            </plugin>


        </plugins>
    </build>
```
## function开发
```java
package base64;

import com.facebook.presto.common.type.StandardTypes;
import com.facebook.presto.spi.function.Description;
import com.facebook.presto.spi.function.ScalarFunction;
import com.facebook.presto.spi.function.SqlType;
import io.airlift.slice.Slice;
import io.airlift.slice.Slices;

public class UnBase64Function {
    @ScalarFunction("unBase64") // 固定参数，表示函数名的意思，也就我们在使用Presto的时候用的函数名
    @Description("base64解密") // 函数的注释
    @SqlType(StandardTypes.VARCHAR) // 表示数据类型
    public static Slice unBase64(@SqlType(StandardTypes.VARCHAR) Slice input) {
        // 解码
        String res = "";
        try {
            byte[] decodeBytes = java.util.Base64.getDecoder().decode(input.toStringUtf8());
            res = new String(decodeBytes, "utf-8");
        } catch (Exception e) {
        }
        return Slices.utf8Slice(res);
    }
}

```
## plugin开发
```java
import java.util.Set;

import base64.Base64Function;
import base64.UnBase64Function;
import com.facebook.presto.spi.Plugin;
import com.google.common.collect.ImmutableSet;

public class PrestoUdfPlugin implements Plugin {
    @Override
    public Set<Class<?>> getFunctions() {
        return ImmutableSet.<Class<?>>builder()
                // 添加插件class
                .add(UnBase64Function.class)
                .build();
    }
}

```
## 加载plugin
在src/main/resources下创建目录META-INF，再在META-INF目录下创建services目录，注意下图项目结构中META-INF是父目录 services是子目录，只是idea合并显示了，不是说文件夹名里面有点 “.” 。然后创建文件com.facebook.presto.spi.Plugin，文件内容为plugin的全类名。
例如本例：`PrestoUdfPlugin`
## 上线
1. maven打包。
2. 在所有presto节点服务器的presto安装目录${PRESTO_HOME}/plugin下新建一个my_plugin目录。
3. 上传jar包，及相关依赖包（在lib目录下）到所有服务器新建的my_plugin目录下。
4. 重启presto： ${PRESTO_HOME}/bin/launcher restart。

# 项目结构

![结构](https://m.360buyimg.com/babel/jfs/t1/2081/12/19574/60306/62d7d4afEe88e1e4b/e23c77b2aa3d6304.png "结构")

参考：[【presto】方法二：presto实现自定义UDF函数](https://blog.csdn.net/Mrerlou/article/details/119997884 "【presto】方法二：presto实现自定义UDF函数")


---

> 作者: [hpkaiq](https://hpk.me)  
> URL: https://hpk.me/posts/presto-udf-simple/  

