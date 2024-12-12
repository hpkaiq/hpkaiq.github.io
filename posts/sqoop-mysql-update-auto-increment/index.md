# Sqoop Mysql Update AUTO_INCREMENT 自增主键重复增长问题


&lt;!--more--&gt;

```bash
sqoop export \
--update-key unique_index_columns \
--update-mode allowinsert
```
问题描述：
用上述模式 sqoop 导入数据更新 mysql 数据，无论导入的数据与mysql里数据相比有没有更新，mysql表的 AUTO_INCREMENT 的PRIMARY KEY 例如 （`id` int(11) NOT NULL AUTO_INCREMENT COMMENT &#39;自增主键&#39;） 实际都是增加了的，当有新唯一主键数据插入时，id会是累加后的开始，会变的很大。

解决：
sqoop源码层级暂时没看原因。。。
可以从业务流程上解决下这个问题，保证导入的数据都是新增或者更新的，不会有与mysql一样的数据，可以一定程度上减少 id 的增长。


---

> 作者: hpkaiq  
> URL: https://hpk.me/posts/sqoop-mysql-update-auto-increment/  

