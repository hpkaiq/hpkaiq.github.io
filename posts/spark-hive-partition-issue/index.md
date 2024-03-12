# spark 读取 hive date 分区表 奇怪的报错


&lt;!--more--&gt;

当 hive 表的分区字段 是 date 类型时，用如下方式读取会发生报错。
```scala
val targetDay = &#34;2020-08-20&#34;
spark.read.table(tableName)
.where(s&#34;targetday in (&#34; &#43;
      s&#34;date_sub(&#39;$targetDay&#39;,59),&#34; &#43;
      s&#34;date_sub(&#39;$targetDay&#39;,49),&#34; &#43;
      s&#34;date_sub(&#39;$targetDay&#39;,39),&#34; &#43;
      s&#34;date_sub(&#39;$targetDay&#39;,29),&#34; &#43;
      s&#34;date_sub(&#39;$targetDay&#39;,14),&#34; &#43;
      s&#34;date_sub(&#39;$targetDay&#39;,6),&#34; &#43;
      s&#34;date_sub(&#39;$targetDay&#39;,5),&#34; &#43;
      s&#34;date_sub(&#39;$targetDay&#39;,4),&#34; &#43;
      s&#34;date_sub(&#39;$targetDay&#39;,3),&#34; &#43;
      s&#34;date_sub(&#39;$targetDay&#39;,2),&#34; &#43;
      s&#34;date_sub(&#39;$targetDay&#39;,1),&#34; &#43;
      s&#34;date_sub(&#39;$targetDay&#39;,0))&#34; ).show
```
targetday 为 表的分区字段，date类型。
当 in 后面 的日期个数大于 10 时就会报错，小于等于 10 时都不会报错。奇怪的现象。。。
targetday 为 string 类型不会有此错误。


---

> 作者:   
> URL: https://hpk.me/posts/spark-hive-partition-issue/  

