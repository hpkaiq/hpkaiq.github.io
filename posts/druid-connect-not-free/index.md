# 阿里Druid连接池连接不释放、连接泄漏排查


&lt;!--more--&gt;

配置好下面三个属性。
```xml
&lt;!-- 超过时间限制是否回收 --&gt;  
&lt;property name=&#34;removeAbandoned&#34; value=&#34;true&#34; /&gt;  
&lt;!-- 超时时间；单位为秒。180秒=3分钟   --&gt;
&lt;property name=&#34;removeAbandonedTimeout&#34; value=&#34;180&#34; /&gt;  
&lt;!-- 关闭abanded连接时输出错误日志   --&gt;
&lt;property name=&#34;logAbandoned&#34; value=&#34;true&#34; /&gt;
```
查看日志文件。
```xml
2019-04-17 17:10:05:140 - ERROR [Druid-ConnectionPool-Destroy-1559154670] - com.alibaba.druid.pool.DruidDataSource com.alibaba.druid.pool.DruidDataSource.removeAbandoned:2666 abandon connection, owner thread: http-nio-8085-exec-1, connected at : 1555491605100, open stackTrace
	at java.lang.Thread.getStackTrace(Thread.java:1559)
	at com.alibaba.druid.pool.DruidDataSource.getConnectionDirect(DruidDataSource.java:1313)
	at com.alibaba.druid.pool.DruidDataSource.getConnection(DruidDataSource.java:1235)
	at com.alibaba.druid.pool.DruidDataSource.getConnection(DruidDataSource.java:1225)
	at com.xxxx.xx.common.util.JdbcUtil.linkTrace(JdbcUtil.java:39)
	at com.xxxx.xx.xxxxx.service.impl.DashboardServiceImpl.getExpenTrend(DashboardServiceImpl.java:263)
	at com.xxxx.xx.xxxxx.service.impl.DashboardServiceImpl.getBossData(DashboardServiceImpl.java:58)
	at com.xxxx.xx.xxxxx.controller.DashboardController.getBossData(DashboardController.java:43)
```
日志里查找```removeAbandoned```关键字，找到自己哪个方法获取连接后没有关闭资源，排查原因。


---

> 作者:   
> URL: https://hpk.me/posts/druid-connect-not-free/  

