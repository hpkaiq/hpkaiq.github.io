# Hive 日期维度表


hive日期维度表，常用的字段基本都有。

&lt;!--more--&gt;

### 1. 建表语句

hive执行以下语句建表。库名表名，存储格式、位置根据需要修改。

```sql
create table if not exists datasystem.dim_date (

date_id string comment &#39;日期(yyyymmdd)&#39;

,datestr string comment &#39;日期(yyyy-mm-dd)&#39;

,date_name string comment &#39;日期名称中文&#39;

,weekid int comment &#39;周(0-6,周日~周六)&#39;

,week_cn_name string comment &#39;周_名称_中文&#39;

,week_en_name string comment &#39;周_名称_英文&#39;

,week_en_nm string comment &#39;周_名称_英文缩写&#39;

,yearmonthid string comment &#39;月份id(yyyymm)&#39;

,yearmonthstr string comment &#39;月份(yyyy-mm)&#39;

,monthid int comment &#39;月份id（1-12）&#39;

,monthstr string comment &#39;月份&#39;

,month_cn_name string comment &#39;月份名称_中文&#39;

,month_en_name string comment &#39;月份名称_英文&#39;

,month_en_nm string comment &#39;月份名称_简写_英文&#39;

,quarterid int comment &#39;季度id（1-4）&#39;

,quarterstr string comment &#39;季度名称&#39;

,quarter_cn_name string comment &#39;季度名称_中文&#39;

,quarter_en_name string comment &#39;季度名称_英文&#39;

,quarter_cn_nm string comment &#39;季度名称_简写中文&#39;

,quarter_en_nm string comment &#39;季度名称_简写英文&#39;

,yearid int comment &#39;年份id&#39;

,year_cn_name string comment &#39;年份名称_中文&#39;

,year_en_name string comment &#39;年份名称_英文&#39;

,month_start_date string comment &#39;当月1号(yyyy-mm-dd)&#39;

,month_end_date string comment &#39;当月最后日期(yyyy-mm-dd)&#39;

,month_timespan int comment &#39;月跨天数&#39;

,week_with_year string comment &#39;年-当年第几周&#39;

,week_of_year int comment &#39;当年第几周&#39;

,day_of_year int comment &#39;当年第几天&#39;

,workday_flag string comment &#39;是否工作日(周一至周五Y，否则:N)&#39;

,weekend_flag string comment &#39;是否周末(周六和周日Y,否则:N)&#39;

)comment &#39;日期维度表&#39;

STORED AS PARQUET;
```

### 2. 插入数据

hive执行以下语句插入数据，注意修改对应的库名表名。

```sql
insert overwrite table datasystem.dim_date

select date_id

,datestr

,concat(yearid,&#39;年&#39;,monthid,&#39;月&#39;,substr(datestr,9,2),&#39;日&#39;) as date_name

,weekid

,case weekid when 0 then &#39;星期日&#39;

when 1 then &#39;星期一&#39;

when 2 then &#39;星期二&#39;

when 3 then &#39;星期三&#39;

when 4 then &#39;星期四&#39;

when 5 then &#39;星期五&#39;

when 6 then &#39;星期六&#39;

end as week_cn_name

,case weekid when 0 then &#39;Sunday&#39;

when 1 then &#39;Monday&#39;

when 2 then &#39;Tuesday&#39;

when 3 then &#39;Wednesday&#39;

when 4 then &#39;Thurday&#39;

when 5 then &#39;Friday&#39;

when 6 then &#39;Saturday&#39;

end as week_en_name

,case weekid when 0 then &#39;Sun&#39;

when 1 then &#39;Mon&#39;

when 2 then &#39;Tues&#39;

when 3 then &#39;Wed&#39;

when 4 then &#39;Thur&#39;

when 5 then &#39;Fri&#39;

when 6 then &#39;Sat&#39;

end as week_en_nm

,substr(date_id,1,6) as yearmonthid

,substr(datestr,1,7) as yearmonthstr

,monthid

,concat(yearid,&#39;年&#39;,monthid,&#39;月&#39;) as monthstr

,concat(monthid,&#39;月&#39;) as month_cn_name

,case monthid when 1 then &#39;January&#39;

when 2 then &#39;February&#39;

when 3 then &#39;March&#39;

when 4 then &#39;April&#39;

when 5 then &#39;May&#39;

when 6 then &#39;June&#39;

when 7 then &#39;July&#39;

when 8 then &#39;August&#39;

when 9 then &#39;September&#39;

when 10 then &#39;October&#39;

when 11 then &#39;November&#39;

when 12 then &#39;December&#39;

end as month_en_name

,case monthid when 1 then &#39;Jan&#39;

when 2 then &#39;Feb&#39;

when 3 then &#39;Mar&#39;

when 4 then &#39;Apr&#39;

when 5 then &#39;May&#39;

when 6 then &#39;Jun&#39;

when 7 then &#39;Jul&#39;

when 8 then &#39;Aug&#39;

when 9 then &#39;Sept&#39;

when 10 then &#39;Oct&#39;

when 11 then &#39;Nov&#39;

when 12 then &#39;Dec&#39;

end as month_en_nm

,quarterid

,concat(yearid,quarterid) as quarterstr

,concat(yearid,&#39;年第&#39;,quarterid,&#39;季度&#39;) as quarter_cn_name

,concat(yearid,&#39;Q&#39;,quarterid) as quarter_en_name

,case quarterid when 1 then &#39;第一季度&#39;

when 2 then &#39;第二季度&#39;

when 3 then &#39;第三季度&#39;

when 4 then &#39;第四季度&#39;

end as quarter_cn_nm

,concat(&#39;Q&#39;,quarterid) as quarter_en_nm

,yearid

,concat(yearid,&#39;年&#39;) as year_cn_name

,yearid as year_en_name

,month_start_date

,month_end_date

,datediff(month_end_date,month_start_date) &#43; 1 as month_timespan

,week_with_year

,week_of_year

,day_of_year

,case when weekid in (1,2,3,4,5) then &#39;Y&#39; else &#39;N&#39; end as workday_flag

,case when weekid in (0,6) then &#39;Y&#39; else &#39;N&#39; end as weekend_flag

from

(

select from_unixtime(unix_timestamp(datestr,&#39;yyyy-MM-dd&#39;),&#39;yyyyMMdd&#39;) as date_id

,datestr as datestr

,pmod(datediff(datestr, &#39;2012-01-01&#39;), 7) as weekid

,concat(substr(datestr,1,4),substr(datestr,6,2)) as yearmonthid

,substr(datestr,1,7) as yearmonthstr

,cast(substr(datestr,6,2) as int) as monthid

,case when cast(substr(datestr,6,2) as int) &lt;= 3 then 1

when cast(substr(datestr,6,2) as int) &lt;= 6 then 2

when cast(substr(datestr,6,2) as int) &lt;= 9 then 3

when cast(substr(datestr,6,2) as int) &lt;= 12 then 4

end as quarterid

,substr(datestr,1,4) as yearid

,date_sub(datestr,dayofmonth(datestr)-1) as month_start_date --当月第一天

,last_day(date_sub(datestr,dayofmonth(datestr)-1)) month_end_date --当月最后一天

,concat(case when month(datestr) = 1 and weekofyear(datestr) &gt; 10 then cast((year(datestr) -1) as string)
 when month(datestr) = 12 and weekofyear(datestr) &lt; 10 then cast((year(datestr) &#43;1) as string)
 else cast(year(datestr) as string) end,&#39;-&#39;,cast(weekofyear(datestr) as string)) as week_with_year

,weekofyear(datestr) as week_of_year

,cast(date_format(datestr, &#39;D&#39;) as int) as day_of_year

from

(

select date_add(&#39;2000-01-01&#39;,t0.pos) as datestr

from

(

select posexplode(

split(

repeat(&#39;o&#39;, datediff(from_unixtime(unix_timestamp(&#39;20991231&#39;,&#39;yyyymmdd&#39;),&#39;yyyy-mm-dd&#39;), &#39;2000-01-01&#39;)), &#39;o&#39;

)

)

) t0

) t
) A
;
```



---

> 作者: hpkaiq  
> URL: https://hpk.me/posts/hive-dim-date/  

