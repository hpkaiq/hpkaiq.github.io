# spark 读取 hive date 分区表 奇怪的报错


<!--more-->

当 hive 表的分区字段 是 date 类型时，用如下方式读取会发生报错。
```scala
val targetDay = "2020-08-20"
spark.read.table(tableName)
.where(s"targetday in (" +
      s"date_sub('$targetDay',59)," +
      s"date_sub('$targetDay',49)," +
      s"date_sub('$targetDay',39)," +
      s"date_sub('$targetDay',29)," +
      s"date_sub('$targetDay',14)," +
      s"date_sub('$targetDay',6)," +
      s"date_sub('$targetDay',5)," +
      s"date_sub('$targetDay',4)," +
      s"date_sub('$targetDay',3)," +
      s"date_sub('$targetDay',2)," +
      s"date_sub('$targetDay',1)," +
      s"date_sub('$targetDay',0))" ).show
```
targetday 为 表的分区字段，date类型。
当 in 后面 的日期个数大于 10 时就会报错，小于等于 10 时都不会报错。奇怪的现象。。。
targetday 为 string 类型不会有此错误。


---

> 作者: <no value>  
> URL: https://hpk.me/posts/spark-hive-partition-issue/  

