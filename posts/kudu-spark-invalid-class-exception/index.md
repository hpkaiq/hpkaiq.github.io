# kudu-spark KuduContext  java.io.InvalidClassException 解决


背景：
线上kudu 集群版本为1.11.0版本, spark 使用 kudu-spark2_2.11-1.7.0.jar，
为了使用新版本中的
`val wo = new KuduWriteOptions(ignoreNull = true)`
特性，升级至 kudu-spark2_2.11-1.11.0.jar 版本，但是报错。

<!--more-->



```java
java.io.InvalidClassException: org.apache.kudu.spark.kudu.KuduContext; local class incompatible: stream classdesc serialVersionUID = xxxxxxxx, local class serialVersionUID = xxxxxxx111
```

在https://issues.apache.org/jira/browse/KUDU-2898 中说这个已经修复，但是修复的版本是kudu 1.12.0，升级线上kudu版本这个暂时不考虑，只好自己修改源码，重新编译 kudu-spark2_2.11-1.11.0.jar

- 1.解压 kudu-spark2_2.11-1.11.0.jar ，将解压出的所有文件夹移动至一个空文件夹，例如tmp

- 2.在idea中新建maven项目，配置好 scala 等环境

- 3.解压 kudu-spark2_2.11-1.11.0-sources.jar，将 其中 org 文件夹 复制 至 maven中刚刚建好的项目中（org/apache/kudu/spark/kudu）

- 4.修改文件 KuduContext.scala 、 KuduRDD.scala， 增加或修改 @SerialVersionUID(xxxxxxxxxxL) ，xxxxxxxxxx改为报错信息中的local class serialVersionUID<br>
- ![请输入图片描述][1]<br>
- ![请输入图片描述][2]<br>
- ![请输入图片描述][3]<br>
-    5：OperationType.scala文件也需修改，和 上面的serialVersionUID不同，走完整个流程运行会报这个文件的serialVersionUID错误，改成错误信息中的local class serialVersionUID

- 6.将 kudu-spark2_2.11-1.11.0.pom 中的 dependencies 粘贴至 maven 中项目的pom文件中，还需额外增加以下依赖：

```xml
        <dependency>
            <groupId>org.apache.kudu</groupId>
            <artifactId>kudu-client</artifactId>
            <version>1.11.0</version>
        </dependency>

        <dependency>
            <groupId>org.apache.yetus</groupId>
            <artifactId>audience-annotations</artifactId>
            <version>0.13.0</version>
        </dependency>
        <dependency>
            <groupId>org.hdrhistogram</groupId>
            <artifactId>HdrHistogram</artifactId>
            <version>2.1.12</version>
        </dependency>
```
- 7.idea中将maven项目打包

- 8.将idea中打好的包解压，将其中 org/apache/kudu/spark/kudu  下的文件覆盖至 tmp 中 org/apache/kudu/spark/kudu

- 9.将 tmp 文件夹下的所有目录打包成新的jar包





参考链接：

1. https://community.cloudera.com/t5/Support-Questions/KUDU-need-rebuild-post-upgrade-of-CDH-cluster/td-p/92885
2. https://issues.apache.org/jira/browse/KUDU-2898


[1]: https://i3.wp.com/telegra.ph/file/62c78529eb3073b0727e7.png
[2]: https://i0.wp.com/telegra.ph/file/c5e7c401b84230d7a9bc3.png
[3]: https://i0.wp.com/telegra.ph/file/9513fccdb6c8b328ad76a.png


---

> 作者: [hpkaiq](https://hpk.me)  
> URL: https://hpkaiq.github.io/posts/kudu-spark-invalid-class-exception/  

