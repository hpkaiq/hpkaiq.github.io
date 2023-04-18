# Apache Kudu 写入数据定期出问题


<!--more-->

线上项目出现一个很奇怪的问题，数据经过Spark程序消费Kafka写入Kudu，出现Kudu Master连接超时，这个问题开始排查不出原因，有点头大，只能采用下下策，重启Spark程序，出现过几次后， 我记录了出现的时间，发现每次出现时间有个固定周期，一周，有规律就是最大的好消息，感觉离发现真相不远了，果然网上有这方面的问题讨论，虽说以前也去网上搜索过相关问题，毕竟Kudu相比于Hive、HBase还是小众了一点。。。是Kudu Java Client的问题，使用1.5以上版本就没问题了。

参考：[关于kudu使用的一些问题及解决办法][1]

[1]: https://blog.csdn.net/u011489205/article/details/79173537



---

> 作者: <no value>  
> URL: https://hpk.me/posts/kudu-periodic-issues/  

