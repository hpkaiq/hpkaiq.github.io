# 脚本执行spark-shell scala文件退出


<!--more-->

脚本
```bash
#! /bin/bash
source /etc/profile

set +o posix  # to enable process substitution when not running on bash 

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
-i <(echo 'val args = "'$arguments'".split("\\s+")' ; cat $scala_file)

```
简单scala文件示例
```java
val path = args(0)
spark.read.parquet(path).where("app_id = 77701").repartition(1).write.parquet(s"${path}_new")
sys.exit
```

如果不需要传参，可简单使用
```bash
spark-shell < test.scala
```


---

> 作者: [hpkaiq](https://hpk.me)  
> URL: https://hpk.me/posts/bash-spark-shell-scala/  

