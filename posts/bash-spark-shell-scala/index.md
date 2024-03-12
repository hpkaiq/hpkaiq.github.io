# 脚本执行spark-shell scala文件退出


&lt;!--more--&gt;

脚本
```bash
#! /bin/bash
source /etc/profile

set &#43;o posix  # to enable process substitution when not running on bash 

scala_file=$1

shift 1

arguments=$@
 
##### scala 文件后加  sys.exit

spark-shell --master yarn \
--executor-cores 5 \
--num-executors 4 \
--executor-memory 8g \
--driver-memory 3g \
--conf spark.serializer=org.apache.spark.serializer.KryoSerializer \
--conf spark.default.parallelism=400 \
--conf spark.sql.shuffle.partitions=400 \
--conf spark.shuffle.io.maxRetries=10 \
--conf spark.shuffle.io.retryWait=20s \
--conf spark.yarn.executor.memoryOverhead=4096 \
--conf spark.network.timeout=300s \
--name sparkshell_scala \
-i &lt;(echo &#39;val args = &#34;&#39;$arguments&#39;&#34;.split(&#34;\\s&#43;&#34;)&#39; ; cat $scala_file)

```
简单scala文件示例
```java
val path = args(0)
spark.read.parquet(path).where(&#34;app_id = 77701&#34;).repartition(1).write.parquet(s&#34;${path}_new&#34;)
sys.exit
```

如果不需要传参，可简单使用
```bash
spark-shell &lt; test.scala
```


---

> 作者: hpkaiq  
> URL: https://hpk.me/posts/bash-spark-shell-scala/  

