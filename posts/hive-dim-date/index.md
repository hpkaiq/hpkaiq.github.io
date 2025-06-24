# Hive 日期维度表


hive日期维度表，常用的字段基本都有。

<!--more-->

### 1. 建表语句

hive执行以下语句建表。库名表名，存储格式、位置根据需要修改。

```sql
create table if not exists datasystem.dim_date (

date_id string comment '日期(yyyymmdd)'

,datestr string comment '日期(yyyy-mm-dd)'

,date_name string comment '日期名称中文'

,weekid int comment '周(0-6,周日~周六)'

,week_cn_name string comment '周_名称_中文'

,week_en_name string comment '周_名称_英文'

,week_en_nm string comment '周_名称_英文缩写'

,yearmonthid string comment '月份id(yyyymm)'

,yearmonthstr string comment '月份(yyyy-mm)'

,monthid int comment '月份id（1-12）'

,monthstr string comment '月份'

,month_cn_name string comment '月份名称_中文'

,month_en_name string comment '月份名称_英文'

,month_en_nm string comment '月份名称_简写_英文'

,quarterid int comment '季度id（1-4）'

,quarterstr string comment '季度名称'

,quarter_cn_name string comment '季度名称_中文'

,quarter_en_name string comment '季度名称_英文'

,quarter_cn_nm string comment '季度名称_简写中文'

,quarter_en_nm string comment '季度名称_简写英文'

,yearid int comment '年份id'

,year_cn_name string comment '年份名称_中文'

,year_en_name string comment '年份名称_英文'

,month_start_date string comment '当月1号(yyyy-mm-dd)'

,month_end_date string comment '当月最后日期(yyyy-mm-dd)'

,month_timespan int comment '月跨天数'

,week_with_year string comment '年-当年第几周'

,week_of_year int comment '当年第几周'

,day_of_year int comment '当年第几天'

,workday_flag string comment '是否工作日(周一至周五Y，否则:N)'

,weekend_flag string comment '是否周末(周六和周日Y,否则:N)'

)comment '日期维度表'

STORED AS PARQUET;
```

### 2. 插入数据

hive执行以下语句插入数据，注意修改对应的库名表名。

```sql
insert overwrite table datasystem.dim_date

select date_id

,datestr

,concat(yearid,'年',monthid,'月',substr(datestr,9,2),'日') as date_name

,weekid

,case weekid when 0 then '星期日'

when 1 then '星期一'

when 2 then '星期二'

when 3 then '星期三'

when 4 then '星期四'

when 5 then '星期五'

when 6 then '星期六'

end as week_cn_name

,case weekid when 0 then 'Sunday'

when 1 then 'Monday'

when 2 then 'Tuesday'

when 3 then 'Wednesday'

when 4 then 'Thurday'

when 5 then 'Friday'

when 6 then 'Saturday'

end as week_en_name

,case weekid when 0 then 'Sun'

when 1 then 'Mon'

when 2 then 'Tues'

when 3 then 'Wed'

when 4 then 'Thur'

when 5 then 'Fri'

when 6 then 'Sat'

end as week_en_nm

,substr(date_id,1,6) as yearmonthid

,substr(datestr,1,7) as yearmonthstr

,monthid

,concat(yearid,'年',monthid,'月') as monthstr

,concat(monthid,'月') as month_cn_name

,case monthid when 1 then 'January'

when 2 then 'February'

when 3 then 'March'

when 4 then 'April'

when 5 then 'May'

when 6 then 'June'

when 7 then 'July'

when 8 then 'August'

when 9 then 'September'

when 10 then 'October'

when 11 then 'November'

when 12 then 'December'

end as month_en_name

,case monthid when 1 then 'Jan'

when 2 then 'Feb'

when 3 then 'Mar'

when 4 then 'Apr'

when 5 then 'May'

when 6 then 'Jun'

when 7 then 'Jul'

when 8 then 'Aug'

when 9 then 'Sept'

when 10 then 'Oct'

when 11 then 'Nov'

when 12 then 'Dec'

end as month_en_nm

,quarterid

,concat(yearid,quarterid) as quarterstr

,concat(yearid,'年第',quarterid,'季度') as quarter_cn_name

,concat(yearid,'Q',quarterid) as quarter_en_name

,case quarterid when 1 then '第一季度'

when 2 then '第二季度'

when 3 then '第三季度'

when 4 then '第四季度'

end as quarter_cn_nm

,concat('Q',quarterid) as quarter_en_nm

,yearid

,concat(yearid,'年') as year_cn_name

,yearid as year_en_name

,month_start_date

,month_end_date

,datediff(month_end_date,month_start_date) + 1 as month_timespan

,week_with_year

,week_of_year

,day_of_year

,case when weekid in (1,2,3,4,5) then 'Y' else 'N' end as workday_flag

,case when weekid in (0,6) then 'Y' else 'N' end as weekend_flag

from

(

select from_unixtime(unix_timestamp(datestr,'yyyy-MM-dd'),'yyyyMMdd') as date_id

,datestr as datestr

,pmod(datediff(datestr, '2012-01-01'), 7) as weekid

,concat(substr(datestr,1,4),substr(datestr,6,2)) as yearmonthid

,substr(datestr,1,7) as yearmonthstr

,cast(substr(datestr,6,2) as int) as monthid

,case when cast(substr(datestr,6,2) as int) <= 3 then 1

when cast(substr(datestr,6,2) as int) <= 6 then 2

when cast(substr(datestr,6,2) as int) <= 9 then 3

when cast(substr(datestr,6,2) as int) <= 12 then 4

end as quarterid

,substr(datestr,1,4) as yearid

,date_sub(datestr,dayofmonth(datestr)-1) as month_start_date --当月第一天

,last_day(date_sub(datestr,dayofmonth(datestr)-1)) month_end_date --当月最后一天

,concat(case when month(datestr) = 1 and weekofyear(datestr) > 10 then cast((year(datestr) -1) as string)
 when month(datestr) = 12 and weekofyear(datestr) < 10 then cast((year(datestr) +1) as string)
 else cast(year(datestr) as string) end,'-',cast(weekofyear(datestr) as string)) as week_with_year

,weekofyear(datestr) as week_of_year

,cast(date_format(datestr, 'D') as int) as day_of_year

from

(

select date_add('2000-01-01',t0.pos) as datestr

from

(

select posexplode(

split(

repeat('o', datediff(from_unixtime(unix_timestamp('20991231','yyyymmdd'),'yyyy-mm-dd'), '2000-01-01')), 'o'

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

