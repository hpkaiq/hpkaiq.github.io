# python3 pandas 实现mysql upsert操作（唯一键更新）


&lt;!--more--&gt;

```python
from urllib import parse
import pandas as pd
from sqlalchemy import create_engine

db_info = {&#39;user&#39;: &#39;test&#39;,
           &#39;password&#39;: parse.quote_plus(&#39;test&#39;),
           &#39;host&#39;: &#39;xxxxxx.mysql.rds.aliyuncs.com&#39;,
           &#39;port&#39;: 3306,
           &#39;database&#39;: &#39;test&#39;
           }

engine = create_engine(
        &#39;mysql&#43;pymysql://%(user)s:%(password)s@%(host)s:%(port)d/%(database)s?charset=utf8&#39; % db_info,
        encoding=&#39;utf-8&#39;)

def mysql_replace_into(table, conn, keys, data_iter):
    from sqlalchemy.dialects.mysql import insert
    data = [dict(zip(keys, row)) for row in data_iter]
    stmt = insert(table.table).values(data)
    update_stmt = stmt.on_duplicate_key_update(**dict(zip(stmt.inserted.keys(), stmt.inserted.values())))
    conn.execute(update_stmt)


def write_database(df, table_name):
    df.to_sql(table_name, con=engine, index=False, if_exists=&#39;append&#39;, method=mysql_replace_into)


if __name__ == &#39;__main__&#39;:
    data = [
        {&#34;app_id&#34;: 1, &#34;app_secret&#34;: &#34;11111122&#34;, &#34;game_id&#34;: 11},
        {&#34;app_id&#34;: 2, &#34;app_secret&#34;: &#34;22222233&#34;, &#34;game_id&#34;: 22},
        {&#34;app_id&#34;: 3, &#34;app_secret&#34;: &#34;33333344&#34;, &#34;game_id&#34;: 33}
    ]

    detailDF = pd.DataFrame(data)
    write_database(detailDF, &#34;ad_app_ext&#34;)

```


---

> 作者: hpkaiq  
> URL: https://hpk.me/posts/python3-pandas-mysql-upsert/  

