# presto 自定义函数简述


presto自带unbase64函数某些时候会报错，所以想要自定义一个unbase64函数。

&lt;!--more--&gt;

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
    &lt;dependencies&gt;
        &lt;dependency&gt;
            &lt;groupId&gt;com.facebook.presto&lt;/groupId&gt;
            &lt;artifactId&gt;presto-spi&lt;/artifactId&gt;
            &lt;version&gt;0.272&lt;/version&gt;
        &lt;/dependency&gt;
        &lt;dependency&gt;
            &lt;groupId&gt;com.google.guava&lt;/groupId&gt;
            &lt;artifactId&gt;guava&lt;/artifactId&gt;
            &lt;version&gt;18.0&lt;/version&gt;
        &lt;/dependency&gt;
    &lt;/dependencies&gt;

    &lt;build&gt;
        &lt;plugins&gt;
            &lt;plugin&gt;
                &lt;groupId&gt;org.apache.maven.plugins&lt;/groupId&gt;
                &lt;artifactId&gt;maven-jar-plugin&lt;/artifactId&gt;
                &lt;configuration&gt;
                    &lt;classesDirectory&gt;target/classes/&lt;/classesDirectory&gt;
                    &lt;archive&gt;
                        &lt;manifest&gt;
                            &lt;mainClass&gt;com.alibaba.dubbo.container.Main&lt;/mainClass&gt;
                            &lt;!-- 打包时 MANIFEST.MF文件不记录的时间戳版本 --&gt;
                            &lt;useUniqueVersions&gt;false&lt;/useUniqueVersions&gt;
                            &lt;addClasspath&gt;true&lt;/addClasspath&gt;
                            &lt;classpathPrefix&gt;lib/&lt;/classpathPrefix&gt;
                        &lt;/manifest&gt;
                        &lt;manifestEntries&gt;
                            &lt;Class-Path&gt;.&lt;/Class-Path&gt;
                        &lt;/manifestEntries&gt;
                    &lt;/archive&gt;
                &lt;/configuration&gt;
            &lt;/plugin&gt;
            &lt;plugin&gt;
                &lt;groupId&gt;org.apache.maven.plugins&lt;/groupId&gt;
                &lt;artifactId&gt;maven-dependency-plugin&lt;/artifactId&gt;
                &lt;executions&gt;
                    &lt;execution&gt;
                        &lt;id&gt;copy-dependencies&lt;/id&gt;
                        &lt;phase&gt;package&lt;/phase&gt;
                        &lt;goals&gt;
                            &lt;goal&gt;copy-dependencies&lt;/goal&gt;
                        &lt;/goals&gt;
                        &lt;configuration&gt;
                            &lt;type&gt;jar&lt;/type&gt;
                            &lt;includeTypes&gt;jar&lt;/includeTypes&gt;
                            &lt;outputDirectory&gt;
                                ${project.build.directory}/lib
                            &lt;/outputDirectory&gt;
                        &lt;/configuration&gt;
                    &lt;/execution&gt;
                &lt;/executions&gt;
            &lt;/plugin&gt;
            &lt;plugin&gt;
                &lt;groupId&gt;com.facebook.presto&lt;/groupId&gt;
                &lt;artifactId&gt;presto-maven-plugin&lt;/artifactId&gt;
                &lt;version&gt;0.3&lt;/version&gt;
                &lt;extensions&gt;true&lt;/extensions&gt;
            &lt;/plugin&gt;


        &lt;/plugins&gt;
    &lt;/build&gt;
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
    @ScalarFunction(&#34;unBase64&#34;) // 固定参数，表示函数名的意思，也就我们在使用Presto的时候用的函数名
    @Description(&#34;base64解密&#34;) // 函数的注释
    @SqlType(StandardTypes.VARCHAR) // 表示数据类型
    public static Slice unBase64(@SqlType(StandardTypes.VARCHAR) Slice input) {
        // 解码
        String res = &#34;&#34;;
        try {
            byte[] decodeBytes = java.util.Base64.getDecoder().decode(input.toStringUtf8());
            res = new String(decodeBytes, &#34;utf-8&#34;);
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
    public Set&lt;Class&lt;?&gt;&gt; getFunctions() {
        return ImmutableSet.&lt;Class&lt;?&gt;&gt;builder()
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

![结构](https://m.360buyimg.com/babel/jfs/t1/2081/12/19574/60306/62d7d4afEe88e1e4b/e23c77b2aa3d6304.png &#34;结构&#34;)

参考：[【presto】方法二：presto实现自定义UDF函数](https://blog.csdn.net/Mrerlou/article/details/119997884 &#34;【presto】方法二：presto实现自定义UDF函数&#34;)


---

> 作者: hpkaiq  
> URL: https://hpk.me/posts/presto-udf-simple/  

