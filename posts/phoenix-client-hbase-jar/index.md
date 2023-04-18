# phoenix-client-4.14.1-HBase-1.4.jar jar包冲突解决


项目用到phoenix，使用了这个jar包phoenix-client-4.14.1-HBase-1.4.jar，这个jar包导致的jar包冲突很多，一番摸索，解决了，解决如下。

<!--more-->



先jar命令解压jar包，然后删除以下内容。然后在jar命令打成jar包。

1. rm -r javax/
2. rm -r com/jayway/
3. rm -r org/apache/phoenix/shaded/org/apache/thrift
4. rm -r com/sun/jersey/


---

> 作者: <no value>  
> URL: https://hpk.me/posts/phoenix-client-hbase-jar/  

